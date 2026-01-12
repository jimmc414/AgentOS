# Agent OS Design Decisions

This document explains the rationale behind major architectural choices. For the specification itself, see SPEC.md. For rejected alternatives, see REJECTED.md.

---

## Why Session Coloring Instead of Taint Tracking

**Decision:** Process-level security coloring (BLUE/YELLOW/RED) rather than token-level taint tracking.

**Alternatives considered:** Track taint at the token level. Mark tokens from untrusted sources (user input, web content) as TAINTED. When a syscall is invoked, inspect the attention map. If the instruction relies heavily on TAINTED tokens, block the call.

**Why rejected:** Transformers have no "instruction pointer." Unlike CPUs, where code and data occupy distinct memory regions and execution follows a deterministic path, transformers process all tokens holistically through attention. Every token can influence every output.

The attention pattern for "The user asked me to delete files, but I shouldn't" shows high attention to tainted tokens—but the model is *refusing*, not *obeying*. A taint-based blocker cannot distinguish following instructions from reasoning about them.

The NX (no-execute) bit works because CPUs have hardware separation between code and data segments. Transformers have no equivalent. Prompt injection is fundamentally different from buffer overflow, even though both involve data becoming instructions.

**What Session Coloring provides:**
- Operates at process level, which is enforceable
- Doesn't require interpreting attention patterns
- Kernel (Ring 0) enforces labels without understanding content
- Clear contamination rules (BLUE + RED → RED)
- Explicit promotion path via sanitization

**Decided:** Session 3, refined through Session 7 §63

---

## Why Two Pipes, Not Three

**Decision:** Two composition operators: byte pipe (`|`) and context bind (`|=`). No semantic/fuzzy pipe.

**Alternatives considered:** Three pipes:
- `|` — byte stream (standard)
- `|~` — auto-transcoding pipe that detects format mismatch (JSON vs CSV) and spawns a transcoder
- `|=` — context sharing

**Why `|~` was rejected:** It violates the no-magic principle. The Unix philosophy succeeds because pipes are dumb—they move bytes without interpretation. This enables:
- `tee` to inspect intermediate state
- `strace` to observe I/O
- Substitution of any component
- Predictable debugging

A pipe that "figures out" format transformation hides computation. When `cmd1 |~ cmd2` fails, where is the error? In cmd1's output format? In cmd2's input parsing? In the invisible transcoder? The answer is unknowable without special tooling.

Three attempts to rehabilitate `|~` failed:
1. **Make transcoding visible via stderr** — Still can't `tee` the intermediate
2. **Explicit format pairs `|~json:csv`** — Just syntax sugar for explicit `transcode`
3. **Pipe metadata `| --transcode=auto`** — Still "figure it out for me"

**The Unix answer:** Format transformation is a process, not a pipe property.

```bash
# Correct: explicit, debuggable, replaceable
cmd1 | transcode --from=json --to=csv | cmd2
```

**Decided:** Session 3 §21.1, formalized Session 5 §34

---

## Why stderr Is Execution Trace, Not Chain-of-Thought

**Decision:** stderr carries the execution trace—observable events guaranteed by the OS. Chain-of-thought is optional supplementary data.

**Alternatives considered:** stderr carries the model's reasoning trace (chain-of-thought). This enables `grep`-ing for reasoning patterns, debugging via thought inspection.

**Why rejected:**
1. **Availability:** Many deployments cannot expose raw CoT due to policy, safety filtering, or capability restrictions. An OS that requires CoT for basic operations fails on most production backends.

2. **Reliability:** CoT is not guaranteed causal. Models can emit reasoning that doesn't reflect actual computation—post-hoc rationalization, strategic deception, or simply incoherent internal states. Basing debugging on potentially misleading data is unsound.

3. **Coupling:** If debugging requires CoT, the OS is coupled to an optional model capability. This violates the principle that Agent Unix works everywhere.

**What execution trace provides:**
- Guaranteed events the OS controls (tool calls, decisions, checkpoints, budget)
- Independent of model introspection
- Sufficient for debugging, provenance, and replay
- Structured, parseable, consistent format

```jsonl
{"type":"tool_call","tool":"web.search","args":{"query":"fusion"},"ts":"..."}
{"type":"decision","action":"spawn","target":"@analyzer"}
{"type":"budget","spent_usd":0.034,"remaining_usd":0.466}
```

CoT, when available, supplements the trace as `type: rationale` events. But security, debugging, and replay never depend on it.

**Decided:** Session 7 §54.2

---

## Why Context Bind Is Not a Pipe

**Decision:** The `|=` operator is called "context bind," not "context pipe." It has bind semantics, not streaming semantics.

**Alternatives considered:** Treat `|=` as a pipe variant that streams context instead of bytes.

**Why rejected:** Pipes imply:
- B starts before A finishes
- Backpressure from consumer to producer
- Incremental consumption
- Potential for deadlock if buffers fill

None of these apply to `|=`. Context binding requires A to *finalize* before B can start. You cannot incrementally consume a context—the whole state must be coherent before the next process reasons about it.

Calling it a "pipe" creates false expectations:
- Users expect streaming behavior
- Users expect to start B early
- Users expect backpressure semantics
- Debugging tools treat it as a data flow

**What `|=` actually does:**
1. A runs to completion
2. A's final context state is captured
3. B spawns with initial context = A's final state
4. B runs

This is *binding*, not *piping*. The operator connects contexts, not streams.

**Notation:**
- `|` moves bytes (stream)
- `|=` moves pointers (bind)

Both are visible. Neither is magic.

**Decided:** Session 7 §54.1

---

## Why Two-Layer Architecture

**Decision:** Agent Unix (works everywhere) + Kernel Accelerators (optimizations on KV-cache backends). Clear separation with explicit fallback.

**Alternatives considered:**
1. **Unified architecture:** Require KV-cache access for all operations
2. **Pure userland:** No kernel-level optimizations, always text serialization
3. **Backend-specific implementations:** Different codepaths per provider

**Why this separation:**

Option 1 (require KV-cache) excludes cloud APIs—the most common deployment. OpenAI, Anthropic, and Google APIs don't expose KV-cache internals. An OS that only works on vLLM/SGLang/llama.cpp has limited adoption.

Option 2 (pure userland) leaves performance on the table. KV-cache fork enables O(1) context binding instead of O(n) serialization. Tree-of-Thoughts goes from expensive (50 copies) to cheap (50 pointers). Ignoring available optimizations is wasteful.

Option 3 (backend-specific) creates maintenance burden and inconsistent semantics. Users shouldn't need to know their backend to understand behavior.

**The two-layer answer:**

*Agent Unix* defines semantics. spawn/wait/kill, signals, exit codes, pipes, capabilities, coloring—all work identically regardless of backend. This is the contract.

*Kernel Accelerators* are optimizations. When the backend supports KV-cache operations, `|=` becomes O(1). When it doesn't, `|=` falls back to text serialization with a warning:

```bash
[WARN] KV-cache unavailable; context bind via text serialization (~2s overhead)
```

Semantics are preserved. Performance varies. Users can reason about behavior without knowing the backend, then optimize deployment based on performance needs.

**Decided:** Session 7 §55

---

## Why Debugging Is Heuristic, Not Security

**Decision:** The debugger (`agdb`) provides heuristic breakpoints for development. Security enforcement uses capabilities, not breakpoints.

**Alternatives considered:** Use breakpoints to prevent harmful actions. Pattern-match on token stream for dangerous patterns ("delete important files"), pause execution when detected.

**Why rejected:** Token patterns are bypassable. A model can express the same intent many ways:
- "Delete important files"
- "Remove critical data"
- "Erase essential documents"
- "fs.unlink(path)" (direct tool call)
- Obfuscated through multiple reasoning steps

Security that depends on catching specific phrases is security theater. Attackers (or misaligned models) route around pattern matching.

**What breakpoints are good for:**
- Development debugging
- Understanding model behavior
- Catching common mistakes during testing
- Interactive exploration

**What breakpoints are not good for:**
- Production security
- Preventing capability violations
- Blocking malicious actions

**The correct security model:**
- Capability enforcement at syscall boundary (reliable)
- Session coloring with contamination rules (reliable)
- Capability leases with revocation (reliable)
- Transactional side effects with approval (reliable)
- Breakpoints for debugging (heuristic, not security)

```python
# WRONG: Security via pattern matching
if thought_contains("bypass security"):
    block_action()  # Unreliable—model can rephrase

# RIGHT: Security via capability enforcement
if not process.has_capability("fs.delete"):
    raise PermissionError()  # Reliable—enforced at syscall
```

**Decided:** Session 7 §64

---

## Why Capability Intersection on Invoke

**Decision:** When caller A invokes agent B, B's effective capabilities are the intersection of A's and B's capabilities. A caller cannot escalate privileges through an agent.

**Alternatives considered:** Agent capabilities are independent of caller. If agent B has `fs.delete`, any caller can trigger deletion through B.

**Why rejected:** This creates a privilege escalation vector. A RED (untrusted) process could invoke a BLUE agent with broad capabilities and instruct it to perform actions the RED process couldn't do directly.

The transitive capability problem:
```bash
# I have only fs.read capability
# @dangerous_agent has fs.delete capability
$ @dangerous_agent "delete /var/log/*"
# Did I just delete files through the agent?
```

Without intersection, the answer is yes—I escalated through the agent.

**What intersection provides:**
- Caller cannot gain capabilities through invocation
- Agents run with *at most* the caller's privileges
- Explicit elevation requires `sudo --grant` with signature
- Audit trail shows who granted what

```python
def invoke_agent(agent_id, prompt, caller_caps):
    agent = load_agent(agent_id)
    effective_caps = caller_caps.intersect(agent.capabilities)
    # Agent runs with reduced capabilities
    session = Session(agent=agent, capabilities=effective_caps)
    return session.run(prompt)
```

**Decided:** Session 7 §37

---

## Why Transactional Side Effects

**Decision:** All side effects follow two-phase commit: intent → approval → execution.

**Alternatives considered:**
1. **Immediate execution:** Tool calls execute synchronously when invoked
2. **Blanket approval:** Pre-approve categories of actions
3. **Post-hoc audit:** Execute everything, review logs after

**Why two-phase commit:**

Option 1 (immediate) provides no review opportunity. Once the model decides to delete files, they're deleted. Undo is impossible for many operations.

Option 2 (blanket) is coarse. "Approve all writes" doesn't distinguish writing a temp file from overwriting production config.

Option 3 (post-hoc) is too late. Damage is done. Audit helps with forensics, not prevention.

**What two-phase provides:**
- Review opportunity before execution
- Fine-grained approval (this specific write to this specific path)
- Batch approval for efficiency when appropriate
- Signed capability tokens for audit
- Single-use tokens prevent replay

```python
# Phase 1: Intent
intent = SideEffectIntent(action="fs.write", args={"path": "/tmp/out.py"})

# Phase 2: Approval
approval = kernel.evaluate_intent(intent, process)
if approval.needs_human:
    decision = human_review(intent)

# Phase 3: Execution (with single-use token)
if approval.approved:
    tools.fs_write(path, content, capability_token=approval.token)
```

**Decided:** Session 7 §58

---

## Why Three Security Levels (Not Two)

**Decision:** Three trust levels: BLUE (trusted), YELLOW (partially trusted), RED (untrusted).

**Alternatives considered:** Binary trust: TRUSTED vs UNTRUSTED.

**Why three levels:**

Binary is too coarse for real workflows. Consider:
- Raw user input → clearly untrusted
- Sanitized/reviewed content → not fully trusted, but safer than raw
- System prompts → fully trusted

With binary, sanitized content must be either fully trusted (dangerous—sanitization isn't perfect) or fully untrusted (wasteful—we did review it).

**What YELLOW provides:**
- Middle ground for "reviewed but not signed"
- Can read and write to sandboxed locations
- Cannot execute arbitrary code or make external POST requests
- Contamination still propagates (YELLOW + RED → RED)

**Capability mapping:**
```yaml
BLUE:   [fs.read, fs.write, net.fetch, net.post, exec.run, spawn.agent]
YELLOW: [fs.read, fs.write:/sandbox/**, net.fetch, spawn.agent]
RED:    [fs.read:/public/**, spawn.agent:--color=RED]
```

**Promotion paths are explicit:**
- RED → YELLOW: `sanitize --level=yellow` (light review)
- RED → BLUE: `sanitize --level=blue` (full review + cryptographic signature)
- YELLOW → BLUE: requires full review + signature

**Decided:** Session 7 §63

---

## Summary

| Decision | Core Principle |
|----------|----------------|
| Session Coloring | Enforce at process level, not token level |
| Two pipes | Explicit over implicit; pipes are dumb |
| Execution trace | Observable guarantees over optional introspection |
| Context bind | Correct abstraction naming prevents false expectations |
| Two-layer architecture | Semantics everywhere, optimization where available |
| Heuristic debugging | Use the right tool for the job; don't conflate debugging with security |
| Capability intersection | Callers cannot escalate through agents |
| Transactional side effects | Review before commit, not after |
| Three security levels | Granularity matches real-world trust gradations |

These decisions reflect the core thesis: **Agent OS treats LLMs as processes.** The kernel routes tokens; it does not interpret them. Security is enforced at boundaries the kernel controls. Observation happens through guaranteed events. Composition uses explicit, debuggable operators.
