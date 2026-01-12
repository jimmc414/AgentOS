# Agent OS Specification v1.0

## 1. Overview

### 1.1 Thesis

Agent OS treats LLMs as processes. Their I/O is file-like (stdin/stdout/stderr streams). Their memory is managed (context as address space). Their permissions are enforced (capabilities). Their lifecycle is controlled (signals). The kernel routes tokens; it does not interpret them.

### 1.2 Design Principles

1. **Pipes are dumb.** Transformation is a process, not a pipe property.
2. **Explicit over implicit.** If an LLM is invoked, the user specifies `--mode=llm`.
3. **Observable by default.** Every operation has stderr, every state has /proc.
4. **Capabilities intersect down.** A caller cannot escalate privileges through an agent.
5. **Compression is an event, not a daemon.** Agents know when memory is lossy.
6. **Cost is a first-class resource.** Budget limits are as fundamental as memory limits.
7. **Pointers over copies.** Context sharing via KV-cache, not serialization.
8. **Fail explicit.** Errors identify which component failed and why.

### 1.3 Two-Layer Architecture

Agent OS comprises two layers:

**Agent Unix (Userland):** Works on any backend--cloud APIs, local inference, any provider. Provides spawn/wait/kill, signals, exit codes, byte pipes, capability enforcement, session coloring, execution traces, and /proc introspection.

**Kernel Accelerators:** Optional optimizations on KV-cache-capable backends (vLLM, SGLang, llama.cpp). Enables O(1) context binding, shared prefixes, fast checkpoints, and efficient tree-of-thoughts branching.

When accelerators are unavailable, Agent Unix falls back to text serialization. Semantics are preserved; performance is degraded.

---

## 2. Architecture

### 2.1 Ring Model

| Ring | Name | Responsibilities | Interprets Text |
|------|------|------------------|-----------------|
| 0 | Kernel | Token routing, capability enforcement, signal delivery, resource accounting | No |
| 1 | Userland Tools | Sanitization, format conversion, policy decisions | Yes |
| 2 | Agent Processes | LLM inference, tool execution, reasoning | Yes |

Ring 0 enforces security labels; it does not decide which content deserves which label. Ring 1 tools (such as `sanitize`) present content to humans and set labels based on policy. Ring 2 processes execute within the constraints Ring 0 enforces.

### 2.2 Backend Requirements

| Capability | Required For | Fallback |
|------------|--------------|----------|
| Text generation | All operations | None (required) |
| Tool calling | Agent tool use | Text-based tool invocation |
| KV-cache access | O(1) context bind | Text serialization (~2s overhead) |
| KV-cache fork | Tree-of-Thoughts | Sequential execution |
| Prefix sharing | .sc shared contexts | Per-process compilation |

### 2.3 Backend Compatibility

| Backend | Agent Unix | KV-Cache Fork | Shared Prefixes | Context Bind |
|---------|------------|---------------|-----------------|--------------|
| OpenAI API | Full | No | Prompt caching | Text serialization |
| Anthropic API | Full | No | Prompt caching | Text serialization |
| Google Gemini | Full | Partial | Context caching | Text serialization |
| vLLM | Full | Yes | Yes | O(1) pointer |
| SGLang | Full | Yes | Yes | O(1) pointer |
| llama.cpp | Full | Yes | Yes | O(1) pointer |

---

## 3. Process Model

### 3.1 Process Control Block

Each agent process maintains:

```
ProcessControlBlock:
  pid: int                      # Process identifier
  ppid: int                     # Parent process ID
  state: READY | RUNNING | STOPPED | ZOMBIE
  color: BLUE | YELLOW | RED    # Security label

  context:
    handle: ContextHandle       # Reference to context state
    parent: ContextHandle?      # Inherited context (if bound)
    checkpoints: [CheckpointID]

  capabilities:
    granted: Set[Capability]    # From container spec
    effective: Set[Capability]  # After intersection with caller
    leases: Map[Capability, Lease]

  limits:
    budget_usd: float
    velocity_usd_min: float
    tokens_total: int
    context_tokens: int
    tool_calls: int

  accounting:
    spent_usd: float
    tokens_used: int
    tool_calls_made: int

  streams:
    stdin: Channel
    stdout: Channel
    stderr: Channel             # Execution trace
```

### 3.2 Lifecycle

**spawn(spec, stdin?, stdout?) -> pid**

Creates a new agent process from a container specification. Returns the process ID immediately. The process begins in READY state and transitions to RUNNING when scheduled.

```python
pid = spawn(
    spec=ContainerSpec(
        base="claude-sonnet-4-5-20250514",
        persona="You are a research assistant.",
        capabilities=["web.search", "fs.read"],
        limits=ProcessLimits(max_cost_usd=1.00),
    ),
    stdin=input_channel,
    stdout=output_channel,
)
```

**wait(pid, timeout?) -> ExitStatus**

Blocks until the process completes or timeout expires. Returns exit status including code, final accounting, and any error information.

```python
status = wait(pid, timeout=300)
# status.code: int
# status.accounting: ResourceUsage
# status.error: str?
```

**kill(pid, signal)**

Sends a signal to the process. SIGTERM requests graceful shutdown. SIGKILL forces immediate termination.

```python
kill(pid, SIGTERM)  # Graceful: "conclude and return"
kill(pid, SIGKILL)  # Immediate: context discarded
```

### 3.3 Signals

| Signal | Trigger | Default Handler |
|--------|---------|-----------------|
| SIGTERM | `kill PID` | Graceful shutdown, emit partial result |
| SIGKILL | `kill -9 PID` | Immediate termination |
| SIGSTOP | Debugger breakpoint | Pause generation |
| SIGCONT | Debugger continue | Resume generation |
| SIGPIPE | Downstream closed | Stop generating, exit 141 |
| SIGCTXPRESSURE | Context > 80% limit | Page out or summarize |
| SIGXCPU | Token budget exhausted | Exit 66 |
| SIGBRAKE | Spend velocity exceeded | Pause for authorization |
| SIGCAPDROP | Capability revoked | Degrade gracefully |
| SIGCHLD | Child completed | Reap zombie |

Processes register signal handlers:

```python
def handle_ctx_pressure(signal):
    old_turns = context.get_turns(age > 3600)
    summary = summarize(old_turns)
    context.replace(old_turns, summary, tag="COMPRESSED")

register_handler(SIGCTXPRESSURE, handle_ctx_pressure)
```

### 3.4 Exit Codes

| Code | Name | Meaning |
|------|------|---------|
| 0 | SUCCESS | Completed normally |
| 1 | FAILURE | General error |
| 2 | INVALID_INPUT | Malformed request |
| 64 | REFUSED | Capability violation attempted |
| 65 | CTX_OVERFLOW | Context limit exceeded |
| 66 | BUDGET_EXHAUSTED | Token or cost limit reached |
| 67 | UPSTREAM_FAILURE | Dependency failed |
| 68 | LOW_CONFIDENCE | Declined due to uncertainty |
| 69 | CONTAMINATED | Security color violation |
| 70 | LEASE_EXPIRED | Capability lease invalid |
| 71 | RATE_LIMITED | Provider QPS/TPM exceeded |
| 141 | SIGPIPE | Downstream closed (128 + 13) |

### 3.5 Determinism and Reproducibility

**Idempotency Keys**

Tool calls that produce side effects accept idempotency keys:

```python
net.post(
    url="https://api.example.com/create",
    body=data,
    idempotency_key=hash(ctx_id, intent, args)
)
```

If the same key is seen twice, the tool returns the cached result instead of re-executing. This prevents duplicate side effects on retry or replay.

**Dry-Run Planning**

The `--plan` flag emits proposed actions without executing:

```bash
spawn @coder --plan "refactor auth module"
# [PLAN] Proposed side effects:
#   1. fs.read: src/auth/*.py
#   2. fs.write: src/auth/session.py
#   3. shell.exec: pytest tests/auth/
# Estimated cost: $0.45
# Execute? [y/n]
```

**Determinism Knobs**

For reproducible runs when supported by backend:

```bash
# Seed for reproducible sampling
spawn --seed=42 @agent "generate options"

# Force greedy decoding
spawn --temperature=0 @agent "extract facts"

# Capture full run manifest for replay
spawn --manifest=run_001.json @agent "complex task"
```

The run manifest contains: prompt, context hash, tool results, random seeds, timestamps. This enables replay even if not bitwise identical.

---

## 4. Context Management

### 4.1 Context Structure

Context is organized as a Merkle DAG of content-addressed blocks. Each block contains tokens and metadata. The structure enables:

- Efficient deduplication across processes
- O(1) fork via pointer sharing
- Verifiable integrity via hash chains
- Incremental synchronization for remote contexts

```
ContextHandle:
  root_hash: Hash256
  blocks: Map[Hash256, Block]
  metadata:
    created_at: Timestamp
    parent: ContextHandle?
    compression_history: [CompressionEvent]
```

### 4.2 Context Bind Operator (|=)

The `|=` operator binds one process's initial context to another's final state. It is NOT a pipe--no bytes stream. Process A must finalize before process B starts.

```bash
@researcher "study fusion" |= @writer "summarize findings"
```

Semantics:

1. Spawn researcher, run to completion
2. Capture researcher's final context state
3. Spawn writer with initial context = researcher's final context
4. Writer continues from researcher's knowledge state

**Local case (same machine, KV-cache available):**
- O(1) pointer to parent's KV-cache
- Copy-on-write for new tokens
- Shared blocks remain shared

**Remote case (different machine or no KV-cache):**
- Text serialization of context
- ~2 second overhead
- Semantics preserved, performance degraded

Observable via /proc:

```bash
cat /proc/$pid/ctx/init
# parent: /proc/researcher.0/ctx/final
# mode: copy_on_write
# shared_blocks: 1847
# diverged_blocks: 23
```

### 4.3 Checkpointing

**checkpoint() -> CheckpointID**

Saves current context state. The checkpoint is immutable and can be restored or forked.

```python
cp = session.checkpoint()
```

**restore(checkpoint_id) -> Session**

Creates a new session from a checkpoint. The original session is unaffected.

```python
branch_a = Session.restore(cp)
branch_b = Session.restore(cp)
# Two independent continuations from same point
```

### 4.4 Context Tools

**ctx_pageout**

Pages context blocks to external storage. Replaces content with retrieval stubs.

```bash
ctx_pageout --pid=1234 --selector="age>1h" --to=vectorstore://project
# [PAGEOUT] Paged out 47 blocks (89,000 tokens)
# [PAGEOUT] Context replaced with retrieval stubs
```

**ctx_pagein**

Retrieves paged content based on query.

```bash
ctx_pagein --pid=1234 --query="fusion ignition timeline"
# [PAGEIN] Retrieved 3 blocks (12,000 tokens)
# [PAGEIN] Inserted into context at position 47
```

**ctx_merge**

Combines multiple contexts with explicit strategy.

```bash
# Concatenate with deduplication
ctx_merge --mode=concat --dedupe ctx_a ctx_b > ctx_merged

# Extract and reconcile beliefs
ctx_merge --mode=extract-beliefs ctx_a ctx_b > ctx_merged

# Use judge for conflict resolution
ctx_merge --mode=debate ctx_a ctx_b --judge=@arbiter > ctx_merged
```

**ctx-last**

Shell builtin returning handle to last process's final context.

```bash
@researcher "study topic"
@writer "summarize" --context=$(ctx-last)
```

---

## 5. Security

### 5.1 Three-Level Coloring

Every process has a security color:

| Level | Name | Capabilities | Typical Source |
|-------|------|--------------|----------------|
| BLUE | Trusted | All granted capabilities | System, signed human input |
| YELLOW | Partial | Read, sandboxed write, no exec | Sanitized external, known APIs |
| RED | Untrusted | Read-only, no side effects | Raw user input, untrusted web |

### 5.2 Contamination Rules

When content from different trust levels combines:

```
BLUE + RED    -> RED      # Untrusted fully contaminates
BLUE + YELLOW -> YELLOW   # Partial trust reduces to partial
YELLOW + RED  -> RED      # Untrusted contaminates partial
```

A process's effective color is the minimum of its assigned color and all input colors.

### 5.3 Capability Model

Capabilities are fine-grained permissions:

```yaml
capabilities:
  fs.read: ["/data/**", "/public/**"]
  fs.write: ["/sandbox/**"]
  fs.delete: []                        # Not granted
  net.fetch: ["*.example.com"]
  net.post: []                         # Not granted
  spawn.agent: true
  spawn.limit: 10
```

### 5.4 Capability Intersection

When process A invokes agent B, B's effective capabilities are the intersection of:

1. A's capabilities (caller)
2. B's granted capabilities (agent spec)
3. Session capability ceiling

```python
def invoke_agent(agent_id, prompt, caller_caps):
    agent = load_agent(agent_id)
    effective_caps = caller_caps.intersect(agent.capabilities)
    session = Session(agent=agent, capabilities=effective_caps)
    return session.run(prompt)
```

A caller cannot escalate privileges through an agent.

### 5.5 Capability Leases

Capabilities are granted as short-lived leases:

```python
class CapabilityLease:
    capability: str          # e.g., "fs.write"
    scope: str               # e.g., "/tmp/**"
    granted_at: datetime
    expires_at: datetime     # 30s-5min typical
    token: SignedToken
```

Every sensitive operation requires a valid lease. Revocation invalidates future leases without killing the process. SIGCAPDROP notifies the process of revocation.

```bash
sys revoke 1234 fs.write
# [REVOKE] Capability fs.write revoked for PID 1234
# [SIGNAL] Sent SIGCAPDROP to PID 1234
```

### 5.6 Sanitization

Promoting content from lower to higher trust requires sanitization:

```bash
# Light review for RED -> YELLOW
sanitize --level=yellow < untrusted.txt > reviewed.txt

# Full review with signature for RED -> BLUE
sanitize --level=blue --signer=alice@ed25519 < untrusted.txt > trusted.txt
```

The signature is cryptographic proof of human review. Audit logs record who approved what.

### 5.7 Transactional Side Effects

All side effects follow two-phase commit:

**Phase 1: Intent**
Agent emits intent to perform side effect. Kernel queues intent, does not execute.

**Phase 2: Approval**
Policy engine evaluates (capability check, budget check, color check). If policy requires, human approval is requested. Approved intents receive signed capability token.

**Phase 3: Execution**
Tool executes with capability token. Execution logged to provenance. Token consumed (single-use).

```python
# Agent emits intent
intent = SideEffectIntent(
    action="fs.write",
    args={"path": "/tmp/output.py", "content": code},
)

# Kernel evaluates
approval = kernel.evaluate_intent(intent, process)

# Tool executes with token
if approval.approved:
    tools.fs_write(path, content, capability_token=approval.token)
```

Batch approval is available for workflows with many side effects:

```bash
spawn @refactorer "modernize codebase" --batch-approve
# [INTENT BATCH] 12 side effects proposed
# Review: [a]pprove all / [r]eject all / [i]nteractive
```

---

## 6. Observability

### 6.1 Execution Trace

stderr carries the execution trace--guaranteed observable events independent of model introspection:

```jsonl
{"type":"tool_call","tool":"web.search","args":{"query":"fusion"},"ts":"..."}
{"type":"tool_result","tool":"web.search","status":"ok","tokens":1247}
{"type":"decision","action":"spawn","target":"@analyzer"}
{"type":"budget","spent_usd":0.034,"remaining_usd":0.466}
{"type":"checkpoint","id":"ctx_a3f8...","trigger":"explicit"}
{"type":"compression","before":145000,"after":52000,"method":"lru_summarize"}
```

The OS does not depend on model introspection. Debugging, provenance, and replay work from the execution trace alone.

### 6.2 Optional Rationale Stream

If the model supports extended thinking and policy allows, rationale events supplement the trace:

```jsonl
{"type":"rationale","text":"I should search for recent ignition results..."}
```

Rationale is supplementary, not authoritative. Security and debugging do not depend on it.

### 6.3 Provenance Filesystem

Every process has a provenance record:

```
/proc/$pid/provenance/
+-- inputs/
|   +-- prompt.txt              # Original prompt (hash-addressed)
|   +-- context_parent.ref      # Parent context reference
|   +-- retrieved/              # Documents retrieved during execution
+-- tools/
|   +-- call_001.json           # {tool, args, timestamp, result_hash}
|   +-- call_002.json
+-- artifacts/
|   +-- output.txt              # Final stdout (hash-addressed)
|   +-- files/                  # Created files
+-- decisions.jsonl             # Authorization decisions
+-- manifest.json               # Complete run metadata
```

**Decisions log:**

```jsonl
{"ts":"...","action":"fs.write","path":"/tmp/draft.py","authorizer":"capability","decision":"allowed"}
{"ts":"...","action":"net.post","url":"https://api.x.com","authorizer":"human","decision":"allowed","approver":"alice@ed25519:..."}
```

### 6.4 Replay

Provenance enables replay--rerun with recorded tool results:

```bash
aos-replay --manifest=/proc/12345/provenance/manifest.json --verify
# Replaying run 12345...
# Tool call 1: web.search -> using recorded result
# Tool call 2: fs.read -> using recorded result
# Output comparison: MATCH (hash a3f8c2...)
# Replay: CONSISTENT
```

### 6.5 Debugging

The `agdb` debugger provides:

- Breakpoints on token patterns
- Step through tool calls
- Inspect context state
- Modify working memory

**Breakpoints are heuristic, not security mechanisms.** Token patterns can be rephrased. Use capabilities for security enforcement.

```bash
(agdb) break "delete.*important"
[WARN] Breakpoints match token patterns, not intent.
       Use for debugging, not security.
(agdb) continue
```

---

## 7. Resource Management

### 7.1 Budget Limits

Hard ceilings on total consumption:

```yaml
limits:
  tokens_total: 1000000
  max_cost_usd: 5.00
  context_tokens: 200000
  max_tool_calls: 100
```

When exceeded, SIGXCPU is sent and process exits with code 66.

### 7.2 Velocity Limits

Rate limits on spending:

```yaml
limits:
  velocity_usd_min: 0.50    # Max $0.50/minute
```

When exceeded, SIGBRAKE is sent. Process pauses until authorized to continue or killed.

### 7.3 Provider Rate Limits

Per-provider QPS and TPM limits:

```yaml
providers:
  anthropic:
    qps_limit: 40
    tpm_limit: 80000
  openai:
    qps_limit: 50
    tpm_limit: 100000
```

The global scheduler queues requests to stay within limits:

```bash
spawn @agent1 & spawn @agent2 & spawn @agent3 &
# [SCHED] agent3 queued: anthropic QPS limit reached
# [SCHED] agent3 started after 1.2s delay
```

Use `--no-wait` to fail instead of queue.

### 7.4 Context Pressure

Soft limit at 80%, hard limit at 95% of context window:

```yaml
compression:
  soft_limit_pct: 80    # SIGCTXPRESSURE sent
  hard_limit_pct: 95    # Force compression
```

Compression is observable:

```bash
cat /proc/$pid/ctx/compressions
# TIMESTAMP            BEFORE   AFTER    METHOD          PRESERVED
# 2025-01-03T14:20:00  145000   52000    lru_summarize   [IMPORTANT]
```

---

## 8. Shell Syntax

### 8.1 Operators

| Operator | Name | Semantics |
|----------|------|-----------|
| `\|` | Byte pipe | UTF-8 stream, backpressure supported |
| `\|=` | Context bind | B's context = A's final state; not a stream |
| `&` | Background | Run in background, return PID |
| `>` | Redirect stdout | Write to file |
| `2>` | Redirect stderr | Write execution trace to file |
| `2>&1` | Merge streams | Combine trace with output |
| `@` | Agent address | Invoke agent: `@researcher "query"` |

### 8.2 Job Control

```bash
@researcher "long task" &     # Background, get PID
jobs                          # List running jobs
fg %1                         # Bring to foreground
kill %1                       # Send SIGTERM
kill -9 %1                    # Send SIGKILL
```

### 8.3 Context Binding Examples

```bash
# Sequential binding
@researcher "study fusion" |= @analyst "find gaps" |= @writer "draft report"

# Parallel research, merged for synthesis
@researcher "study physics" &
@researcher "study economics" &
wait
ctx_merge --mode=concat $(ctx-last -n 2) |= @synthesizer "combine findings"
```

### 8.4 Stream Redirection

```bash
# Discard execution trace, keep output
@researcher "query" 2>/dev/null > result.txt

# Log trace, stream output to next stage
@researcher "query" 2>trace.log | @formatter

# Debug mode: see everything
@researcher "query" 2>&1 | less
```

---

## 9. Composition Patterns

### 9.1 Reflexion Pattern

Agent critiques and refines its own output:

```bash
#!/usr/bin/env ash
# reflexion.ash - Self-critique and refinement

initial=$(@solver "$1")
echo "$initial" > /tmp/draft

for i in 1 2 3; do
    critique=$(@critic "Review this solution" < /tmp/draft)
    if echo "$critique" | grep -q "APPROVED"; then
        cat /tmp/draft
        exit 0
    fi
    refined=$(@solver "Improve based on: $critique" --context=$(ctx-last))
    echo "$refined" > /tmp/draft
done

cat /tmp/draft
exit 68  # LOW_CONFIDENCE after max iterations
```

### 9.2 Tree-of-Thoughts Pattern

Explore multiple reasoning branches:

```bash
#!/usr/bin/env ash
# tot.ash - Tree of Thoughts exploration

problem="$1"
checkpoint=$(aos checkpoint)

best_score=0
best_branch=""

for approach in "analytical" "creative" "systematic"; do
    aos restore $checkpoint
    result=$(@reasoner "Solve using $approach approach: $problem")
    score=$(@evaluator "Rate 1-10: $result" | grep -o '[0-9]*')

    if [ "$score" -gt "$best_score" ]; then
        best_score=$score
        best_branch=$result
    fi
done

echo "$best_branch"
```

### 9.3 Debate Pattern

Multiple agents argue, judge decides:

```bash
#!/usr/bin/env ash
# debate.ash - Adversarial debate with judge

question="$1"
rounds=3

pro=$(@advocate "Argue FOR: $question")
con=$(@advocate "Argue AGAINST: $question")

for i in $(seq 1 $rounds); do
    pro=$(@advocate "Respond to opposition: $con" --context=$(ctx-last))
    con=$(@advocate "Respond to opposition: $pro" --context=$(ctx-last))
done

@judge "Evaluate debate and decide: $question" \
    --context="PRO: $pro" --context="CON: $con"
```

### 9.4 JSONL Event Schema

Convention for structured inter-agent communication:

```jsonl
{"v":1,"type":"text","content":"The fusion experiment achieved..."}
{"v":1,"type":"text","content":"...ignition.","final":true}
{"v":1,"type":"citation","claim":"achieved ignition","source":"https://...","confidence":0.95}
{"v":1,"type":"tool_call","tool":"web.search","args":{"q":"NIF ignition"}}
{"v":1,"type":"tool_result","tool":"web.search","status":"ok","content":"..."}
{"v":1,"type":"artifact","name":"analysis.py","hash":"a3f8..."}
{"v":1,"type":"confidence","overall":0.87}
{"v":1,"type":"error","code":"BUDGET_EXHAUSTED","message":"Token limit reached"}
```

Agents that understand JSONL extract structured data:

```bash
@researcher "study fusion" | jq -r 'select(.type=="text") | .content'
@researcher "study fusion" | jq 'select(.type=="citation")'
```

---

## 10. Implementation Layers

### Layer 0: Inference Wrapper

Abstracts model backends. Provides uniform interface for:
- Text generation
- Tool calling
- Token counting
- Cost calculation

### Layer 1: Process Model

Implements:
- ProcessControlBlock management
- spawn/wait/kill primitives
- Signal delivery
- Exit code handling
- Resource accounting

### Layer 2: Pipes and Composition

Implements:
- Byte pipe (|) with backpressure
- Context bind (|=) with KV-cache optimization
- Channel abstraction for stdin/stdout/stderr
- Background execution (&)

### Layer 3: Security

Implements:
- Session Coloring (BLUE/YELLOW/RED)
- Capability intersection
- Capability leases with revocation
- Sanitization workflow
- Transactional side effects

### Layer 4: Observability

Implements:
- Execution trace formatting
- Provenance filesystem
- /proc introspection
- Checkpoint management
- Replay infrastructure

### Layer 5: Shell

Implements:
- ash command parser
- Operator handling
- Job control
- ctx-last builtin
- Script execution

---

## Appendix: Container Specification

```yaml
apiVersion: agent/v1
kind: AgentContainer

metadata:
  name: research-assistant
  labels:
    project: alpha

spec:
  base: claude-sonnet-4-5-20250514

  persona: |
    You are a research assistant focused on technical analysis.
    Be thorough, cite sources, flag uncertainty.

  context:
    mounts:
      - source: /memory/shared/knowledge
        target: /ctx/kb
        mode: ro
      - source: /memory/scratch
        target: /ctx/working
        mode: rw

  capabilities:
    tools: [web.search, web.fetch, fs.read]
    agents: [spawn:true, spawn_limit:5]
    memory: [read:/memory/shared/**, write:/ctx/working/**]

  limits:
    max_cost_usd: 1.00
    tokens_total: 500000
    context_tokens: 200000
    max_tool_calls: 50
    wall_clock_sec: 300

  env:
    OUTPUT_FORMAT: jsonl
    CONFIDENCE_THRESHOLD: 0.7
```
