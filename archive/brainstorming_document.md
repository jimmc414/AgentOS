# Agent OS: Unix Primitives for the Agentic Era

This is a brainstorming document, designed to be a dense living document to explore expand and refine these ideas, segemnted by session.


> **Thesis:** "Everything is a file" generalizes to "everything is an interlocutor." The Unix abstraction that unified devices, storage, IPC, and network into byte streams has a natural successor: collapsing APIs, databases, tools, agents, and humans into conversational endpoints.


> Session 1
---

## 1. The Original Insight

Unix succeeded by reducing heterogeneous resources to one interface:

```
open() → read() → write() → close()
```

This enabled:
- **Uniform tooling**: grep works on files, devices, pipes, sockets
- **Composition**: `cmd1 | cmd2 | cmd3`
- **Universal permissions**: rwx applies everywhere
- **Discoverability**: ls, cat, file work on everything

The cost: some semantic information lost (a socket isn't really seekable). The benefit: explosive composability.

---

## 2. The Agent Generalization

### 2.1 Primitive Mapping

| Unix | Agent Equivalent | Notes |
|------|------------------|-------|
| File descriptor | Session handle | Stateful connection to an endpoint |
| `read(fd, buf, n)` | `recv(session) → token_stream` | Blocking, async modes both needed |
| `write(fd, buf, n)` | `send(session, message)` | May trigger computation |
| `open(path, flags)` | `connect(endpoint, capabilities)` | Capability-based from the start |
| `close(fd)` | `disconnect(session)` | Release context, reclaim resources |
| `seek(fd, offset)` | `context.rewind(checkpoint)` | Revert to earlier state? Non-trivial |
| `ioctl(fd, cmd, arg)` | `meta(session, directive)` | Out-of-band control (set temperature, etc.) |

### 2.2 Filesystem Hierarchy

```
/agents/                    # Running agent instances
    /agents/planner.0       # PID-like instance addressing
    /agents/researcher.1
    
/models/                    # Stateless model endpoints (like /dev/urandom)
    /models/opus
    /models/sonnet
    /models/haiku
    
/tools/                     # Capabilities (also agents? turtles all the way?)
    /tools/filesystem
    /tools/web
    /tools/database
    /tools/shell
    
/memory/                    # Persistent context stores
    /memory/global          # Shared knowledge
    /memory/project/        # Scoped memory regions
    /memory/scratch/        # Ephemeral (tmpfs equivalent)
    
/channels/                  # IPC - inter-agent pipes
    /channels/pipe.42
    /channels/broadcast.research
    
/ctx/                       # Mounted context for current process (like /proc/self)
    /ctx/system             # System prompt
    /ctx/conversation       # Current thread
    /ctx/tools              # Available capabilities
    /ctx/mounts             # What's mounted where
```

### 2.3 Device Analogies

```
/dev/tty        ↔  /agents/human.0       # Interactive, stateful, unpredictable latency
/dev/random     ↔  /models/opus          # Stateless, stochastic output  
/dev/null       ↔  /agents/sink          # Absorbs input, returns nothing
/dev/zero       ↔  /agents/yes           # Infinite affirmation (useless but instructive)
/dev/loop0      ↔  /memory/snapshot.img  # Mount a frozen context as live
```

---

## 3. Process Model

### 3.1 Lifecycle Primitives

```python
# fork() - branch conversation with copied context
child_session = session.fork()

# exec() - load new system prompt, preserve context
session.exec(new_system_prompt)

# spawn() - fork+exec equivalent, most common operation
proc = sandbox.spawn(system_prompt, initial_message)

# wait() - block for completion
result = proc.wait(timeout=300)

# kill() - terminate, reclaim context
proc.kill(signal=SIGTERM)  # graceful: "wrap up now"
proc.kill(signal=SIGKILL)  # immediate: context discarded
```

### 3.2 Signals

| Signal | Agent Meaning |
|--------|---------------|
| `SIGTERM` | "Conclude your current task and return" |
| `SIGKILL` | Immediate termination, no cleanup |
| `SIGSTOP` | Pause generation, preserve state |
| `SIGCONT` | Resume generation |
| `SIGUSR1` | Checkpoint context to storage |
| `SIGPIPE` | Downstream consumer disconnected |
| `SIGCHLD` | Child agent completed |

### 3.3 Exit Codes

```
0   - Success, task completed
1   - General failure
2   - Invalid input/malformed request
64  - Refused (capability violation attempted)
65  - Context overflow (couldn't fit in window)
66  - Budget exhausted (tokens/cost/time)
67  - Upstream failure (tool/agent dependency failed)
68  - Confidence threshold not met (agent unsure, declined to answer)
```

---

## 4. Sandboxing

### 4.1 Namespace Isolation

| Linux Namespace | Agent Namespace | Isolates |
|-----------------|-----------------|----------|
| `mnt` | Context namespace | What memory/knowledge is visible |
| `pid` | Agent namespace | Which agents are visible/addressable |
| `net` | Tool namespace | Which tools/external services accessible |
| `user` | Identity namespace | What identity/permissions the agent operates under |
| `uts` | Persona namespace | System prompt, personality |
| `ipc` | Channel namespace | Which IPC channels visible |

### 4.2 Capability Model

```yaml
capabilities:
  # Tool capabilities (like device access)
  tools/filesystem:
    - read:**                    # Read anywhere
    - write:/sandbox/**          # Write only in sandbox
    - execute:false              # Cannot run arbitrary code
    
  tools/web:
    - fetch:*                    # Can fetch any URL
    - search:true                # Can use search
    - post:false                 # Cannot POST (side effects)
    
  tools/shell:
    - commands:[ls, cat, grep, jq]  # Allowlist
    - sudo:false
    
  # Agent capabilities (like process permissions)  
  agents:
    - spawn:true                 # Can create child agents
    - spawn_limit:10             # Max children
    - signal:children            # Can only signal own children
    
  # Memory capabilities
  memory:
    - read:/memory/shared/**
    - write:/memory/scratch/**
    - read:/memory/project/alpha/**
```

### 4.3 Resource Limits (cgroups equivalent)

```yaml
limits:
  # Token budgets
  context_window: 200000        # Max context size
  output_tokens: 8000           # Per-response limit
  total_tokens: 1000000         # Lifetime budget
  
  # Cost
  max_cost_usd: 5.00            # Hard ceiling
  
  # Time
  wall_clock_sec: 600           # Total runtime
  generation_timeout_sec: 120   # Single response timeout
  
  # Compute
  max_tool_calls: 100           # Prevent infinite loops
  max_retries: 3                # Per-tool retry limit
  
  # Spawn limits  
  max_children: 10
  max_depth: 3                  # Nested spawn depth
```

### 4.4 Container Spec

```yaml
# agent-container.yaml
apiVersion: agent/v1
kind: AgentContainer

metadata:
  name: research-assistant
  labels:
    project: alpha
    tier: worker

spec:
  base: claude-sonnet-4-5-20250514
  
  persona: |
    You are a research assistant focused on technical analysis.
    Be thorough, cite sources, flag uncertainty.
  
  context:
    root: /memory/project/alpha
    mounts:
      - source: /memory/shared/knowledge-base
        target: /ctx/kb
        mode: ro
      - source: /memory/scratch/{{instance_id}}
        target: /ctx/working
        mode: rw
        type: tmpfs
        limits:
          tokens: 50000
  
  capabilities:
    tools: [web:fetch, web:search, filesystem:read]
    agents: [spawn:false]
    memory: [read:/memory/shared/**, write:/ctx/working/**]
  
  limits:
    max_cost_usd: 1.00
    wall_clock_sec: 300
    max_tool_calls: 50
  
  env:
    OUTPUT_FORMAT: structured_json
    VERBOSITY: detailed
    CONFIDENCE_THRESHOLD: 0.7

  healthcheck:
    test: ["meta", "ping"]
    interval: 30s
    timeout: 10s
    retries: 3
```

---

## 5. IPC and Composition

### 5.1 Pipes

```python
# Shell equivalent: researcher | analyzer | writer

researcher = spawn("research-agent")
analyzer = spawn("analysis-agent")
writer = spawn("writing-agent")

pipe1 = create_pipe()
pipe2 = create_pipe()

researcher.stdout = pipe1.write_end
analyzer.stdin = pipe1.read_end
analyzer.stdout = pipe2.write_end
writer.stdin = pipe2.read_end

# Data flow: researcher findings → analyzer → writer
```

**Key question:** What flows through the pipe?
- Raw text? (lossy, like byte streams)
- Structured messages? (richer, like D-Bus)
- Context snapshots? (heavy, but preserves reasoning)

### 5.2 Composition Patterns

```bash
# Pipeline: sequential processing
extract_requirements | design_system | implement | review

# Scatter-gather: parallel with aggregation  
{ search_arxiv & search_github & search_docs } | synthesize

# Supervisor tree: hierarchical delegation
supervisor --spawn="worker:5" --strategy=one_for_one

# Tee: broadcast to multiple consumers
analyze_data | tee >(visualize) >(summarize) >(archive)
```

### 5.3 Coordination Primitives

```python
# Mutex: exclusive access to a resource
with context_lock("/memory/shared/state"):
    state = read("/memory/shared/state")
    state = modify(state)
    write("/memory/shared/state", state)

# Semaphore: bounded concurrency
api_semaphore = Semaphore(max=5)  # Max 5 concurrent API calls

# Barrier: synchronization point
barrier = Barrier(parties=3)
# All three agents must reach checkpoint before any proceed

# Channel: typed message passing
results_channel = Channel[AnalysisResult](buffer=10)
```

---

## 6. Persistence and State

### 6.1 Context Checkpointing

```python
# Save context state (like process core dump)
checkpoint_id = session.checkpoint()

# Restore later (new process, same state)
restored = Session.from_checkpoint(checkpoint_id)

# Fork from checkpoint (branch)
branch_a = Session.from_checkpoint(checkpoint_id)
branch_b = Session.from_checkpoint(checkpoint_id)
# Now have two independent continuations
```

### 6.2 Memory Hierarchy

```
Registers     →  Current generation working memory (attention)
L1 Cache      →  Recent conversation turns
L2 Cache      →  Full context window
RAM           →  Retrievable memory (vector store, etc.)
Disk          →  Archived contexts, checkpoints
Tape          →  Training data (read-only, slow, vast)
```

### 6.3 Memory Segments

```python
class MemorySegment:
    """mmap() equivalent for agent context"""
    
    def __init__(self, path, mode='ro', lazy=True):
        self.path = path
        self.mode = mode
        self.loaded = not lazy
        
    def load(self) -> TokenSequence:
        """Page in this memory segment"""
        pass
        
    def evict(self):
        """Page out (if dirty, write back)"""
        pass
        
    def cow_fork(self) -> 'MemorySegment':
        """Copy-on-write fork"""
        pass
```

---

## 7. Open Questions

### 7.1 Semantic vs Byte-Level

Unix pipes are byte streams—semantic meaning is imposed by programs. Agent pipes could be:
- **Unstructured text**: Maximum flexibility, minimum guarantees
- **Structured messages**: Schema-defined, typed
- **Context deltas**: "Here's what I learned, here's my confidence"

What's the right level of abstraction?

### 7.2 Context as Address Space

A process has a virtual address space. An agent has a context window. Mapping:
- **Code segment**: System prompt (read-only, executable)
- **Data segment**: Conversation history, loaded memories
- **Stack**: Current reasoning chain
- **Heap**: Dynamically loaded context (tool results, retrieved docs)

Can we do demand paging? Load context segments only when accessed?

### 7.3 The seek() Problem

Files support random access. Conversations are append-only(ish). What does it mean to seek in a conversation?
- Rewind to checkpoint and branch?
- Edit history (dangerous, like modifying /proc/self/mem)?
- Sliding window over long context?

### 7.4 Distributed Agents

```
localhost     →  Same context space
LAN           →  Shared memory region, fast IPC
Internet      →  Message passing only, high latency

How do we handle:
- Agent migration (move running agent to different host)?
- Distributed consensus (multiple agents, shared state)?
- Partial failure (agent B dies mid-pipeline)?
```

### 7.5 Debugging and Observability

```
strace        →  Log all tool calls
ltrace        →  Log all agent-agent calls  
gdb           →  Step through reasoning?
perf          →  Token/cost profiling
/proc/[pid]/  →  Introspection of running agent
    /ctx      - Current context
    /fd       - Open sessions/connections
    /limits   - Resource limits
    /status   - State, token usage, cost
```

---

## 8. Security Model (Deferred but Noted)

The fundamental tension: Unix separates code and data. Agents cannot.

**Prompt injection = buffer overflow**
- Data (user input) becomes code (instructions)
- No equivalent of NX bit, ASLR, stack canaries

**Possible mitigations (to explore later):**
- Cryptographic instruction signing
- Separate channels for instructions vs data
- Capability attenuation at every boundary
- Taint tracking through context
- Formal verification of information flow

**Security namespaces:**
```yaml
security:
  instruction_sources:
    - system: trusted       # System prompt
    - operator: trusted     # Operator configuration  
    - user: untrusted       # User input
    - tool_result: tainted  # Results from tools
    - agent_output: tainted # Output from other agents
    
  taint_propagation: strict  # Tainted input → tainted reasoning → tainted output
```

---

## 9. Implementation Sketch

### 9.1 Minimal Viable Runtime

```python
class AgentRuntime:
    """Minimal agent OS kernel"""
    
    def __init__(self):
        self.processes: Dict[int, AgentProcess] = {}
        self.next_pid = 1
        self.mounts: Dict[str, MemoryBackend] = {}
        self.channels: Dict[str, Channel] = {}
        
    def spawn(self, spec: ContainerSpec, stdin=None, stdout=None) -> int:
        """Create new agent process"""
        pid = self.next_pid
        self.next_pid += 1
        
        proc = AgentProcess(
            pid=pid,
            spec=spec,
            context=self.build_context(spec),
            tools=self.filter_tools(spec.capabilities),
            stdin=stdin or Channel(),
            stdout=stdout or Channel(),
        )
        
        self.processes[pid] = proc
        proc.start()
        return pid
        
    def wait(self, pid: int, timeout: float = None) -> ExitStatus:
        """Wait for process completion"""
        return self.processes[pid].wait(timeout)
        
    def kill(self, pid: int, signal: Signal):
        """Send signal to process"""
        self.processes[pid].signal(signal)
        
    def pipe(self) -> Tuple[Channel, Channel]:
        """Create IPC pipe"""
        ch = Channel()
        return (ch.read_end(), ch.write_end())
```

### 9.2 Agent Process

```python
class AgentProcess:
    """Single agent instance"""
    
    def __init__(self, pid, spec, context, tools, stdin, stdout):
        self.pid = pid
        self.spec = spec
        self.context = context
        self.tools = tools
        self.stdin = stdin
        self.stdout = stdout
        self.status = ProcessStatus.READY
        self.usage = ResourceUsage()
        
    async def run(self):
        self.status = ProcessStatus.RUNNING
        
        try:
            while not self.stdin.closed:
                message = await self.stdin.recv()
                
                # Check resource limits
                if self.usage.exceeds(self.spec.limits):
                    raise ResourceExhausted()
                
                # Generate response
                response = await self.generate(message)
                
                # Track usage
                self.usage.add(response.tokens, response.cost)
                
                # Send to stdout
                await self.stdout.send(response)
                
        except ResourceExhausted:
            self.exit(ExitCode.BUDGET_EXHAUSTED)
        except Exception as e:
            self.exit(ExitCode.FAILURE)
        else:
            self.exit(ExitCode.SUCCESS)
            
    async def generate(self, message) -> Response:
        """Core generation loop with tool use"""
        
        self.context.append(user=message)
        
        while True:
            response = await self.model.generate(
                system=self.spec.persona,
                context=self.context.render(),
                tools=self.tools.definitions(),
            )
            
            if response.tool_calls:
                results = await self.execute_tools(response.tool_calls)
                self.context.append(assistant=response, tool_results=results)
            else:
                self.context.append(assistant=response)
                return response
```

---

## 10. Milestones

### Phase 1: Foundation
- [ ] Basic spawn/wait/kill
- [ ] Simple capability filtering
- [ ] Token/cost budget enforcement
- [ ] Single-agent execution

### Phase 2: IPC
- [ ] Pipes between agents
- [ ] Structured message passing
- [ ] Basic pipeline composition

### Phase 3: Namespaces
- [ ] Context namespace (memory isolation)
- [ ] Tool namespace (capability isolation)
- [ ] Agent namespace (visibility isolation)

### Phase 4: Persistence
- [ ] Context checkpointing
- [ ] Checkpoint restore
- [ ] Fork from checkpoint

### Phase 5: Observability
- [ ] /proc-like introspection
- [ ] Tool call tracing
- [ ] Cost/token profiling

### Phase 6: Distribution (Future)
- [ ] Remote agent spawning
- [ ] Distributed coordination
- [ ] Failure handling

---

## 11. References and Prior Art

- **Unix Philosophy**: McIlroy, Kernighan, Pike
- **Plan 9**: "Everything is a file" taken further (9P protocol, /net, /proc)
- **Actor Model**: Hewitt, Agha - message-passing concurrency
- **Capability-Based Security**: Dennis & Van Horn, E language, Capsicum
- **Container Runtimes**: Docker, runc, containerd - namespace/cgroup composition
- **MCP (Model Context Protocol)**: Anthropic - tool interface standardization
- **LangGraph**: Agent orchestration with graph-based control flow
- **AutoGen**: Multi-agent conversation framework
- **Erlang/OTP**: Supervisor trees, fault tolerance, "let it crash"

---

## 12. Scratch Space

*Working notes, half-formed ideas, things to revisit*

```
- What's the equivalent of shared libraries? Pre-loaded context snippets?
- Can we do copy-on-write for context forking? Hash content-addressed blocks?
- eBPF for agents? Inject observation/modification at tool boundaries?
- What does ASLR look like? Randomize instruction framing to resist injection?
- Cgroups v2 unified hierarchy - single tree for all resource types?
- What's swap? Summarization as lossy compression to reclaim context?
- OOM killer - which agent gets terminated when resources exhausted?
- Scheduling: round-robin? priority? deadline-based?
- What's an interrupt? External event that preempts generation?
```

---


> Session 2

Here is a dense, architectural brainstorming document designed to be appended to your existing thesis. It moves from the high-level metaphor ("Everything is a file") to the low-level **systems engineering** ("Everything is a context-managed inference"), exploring the *physics* of how such an OS would actually function.

---

# Appendix A: Kernel Architecture & System Internals

**Status:** Living Document / RFC v0.2
**Focus:** Memory Mechanics, Scheduler Economics, and the "Physics" of the System

This document explores the implementation details of the "Agent OS," mapping classical kernel subsystems to their probabilistic equivalents.

## 13. The Kernel: `kagentd` & Hardware Abstraction

*The CPU is no longer deterministic; the OS must manage stochastic compute resources.*

### 13.1 The CPU Abstraction: Models as Cores

In this OS, the LLM is the CPU. We map model tiers to processor architecture concepts (big.LITTLE):

* **E-Cores (Efficient):** `Haiku`, `Flash`. Low latency, low cost. Used for routing, formatting, and simple boolean checks.
* **P-Cores (Performance):** `Opus`, `GPT-4`. High reasoning, high cost. Used for complex synthesis and planning.
* **The Scheduler (`sched_cost`):**
* **Cost-Aware Scheduling:** Unlike Linux CFS (optimizing for CPU time), this optimizes for **$ / Token**.
* **Priority Escalation:** A task failing on an E-Core (low probability output) triggers a "Hardware Interrupt" to migrate the process to a P-Core for a retry.



### 13.2 Kernel Modules (LoRAs)

Low-Rank Adapters (LoRAs) are the equivalent of Linux Kernel Modules (`.ko`).

* **`insmod`:** Dynamic loading of capabilities. `insmod /lib/modules/coder_v2.lora` hot-patches the model weights for the current session without a reboot.
* **Namespace Isolation:** Conflicting LoRAs (e.g., `creative_writing` vs `strict_json`) are loaded into separate process namespaces to prevent "DLL Hell" in the model's latent space.

## 14. Memory Management: The Virtual Context Manager (VCM)

*Mapping the "Infinite Context" illusion to physical hardware limits.*

### 14.1 The Semantic Page Table

* **Virtual Addressing:** Agents perceive an "infinite" address space (Virtual Context). The Kernel manages the physical Context Window (RAM).
* **Page Faults as Retrieval:**
* When an agent's attention mechanism tries to attend to a concept not in VRAM (e.g., "Project Alpha Specs"), a **Semantic Page Fault** triggers.
* **Kernel Trap:** Execution pauses.
* **Page In:** The Kernel retrieves the relevant vector embedding from the backing store (`/dev/nvme0n1` mapped Vector DB).
* **Page Out:** The Kernel evicts the Least Recently *Attended* (LRA) blocks—summarizing them to disk if they contain unique generated thoughts.
* **Resume:** Execution continues transparently.



### 14.2 `fork()` via Copy-on-Write (CoW)

* **Mechanism:** `spawn()` does not copy text. It points the child process to the parent’s existing **KV-Cache** blocks in GPU memory.
* **Branching:** New tokens are allocated in new blocks.
* **Application:** **Monte Carlo Tree Search (MCTS)** becomes a native OS primitive. A "Planner" agent can `fork()` 50 times to explore scenarios. The memory cost is , not .

## 15. The Process Model & Standard Streams

*Refining the I/O interface for the age of reasoning.*

### 15.1 The New Standard Streams

Unix gave us `stdin`, `stdout`, and `stderr`. Agent OS redefines them:

* **`stdin` (fd 0):** Upstream Context / User Prompt.
* **`stdout` (fd 1):** The Final Answer / Actionable Output.
* **`stderr` (fd 2):** **The Chain of Thought (CoT).**

**Why this matters:** By routing reasoning traces to `stderr`, we unlock standard shell redirection for observability without polluting the downstream data pipe.

```bash
# "Run the researcher, discard the messy thinking, save the result"
$ researcher "future of fusion" 2> /dev/null > result.txt

```

### 15.2 The `.aexe` Binary Format

Prompts are brittle source code. We need a compiled format for faster startup.

* **Header:** Target model architecture, required capabilities.
* **`.text` (System Prompt):** The immutable instructions. Compiled into a pre-calculated KV-Cache prefix.
* **`.data` (Knowledge):** Static few-shot examples (vectors).
* **`.bss` (Scratchpad):** Pre-allocated context tokens for reasoning.
* **Loader:** The OS "executes" an `.aexe` by `memcpy`-ing the pre-calculated KV state into VRAM. Startup drops from 500ms to <10ms.

## 16. Scheduling: The Economic Scheduler

*Replacing CPU Cycles with Token Cost.*

### 16.1 The OOM (Out of Money) Killer

* Processes have a `wallet` attached to their `task_struct`.
* **Heuristic:** If an agent enters a "reasoning loop" (high token velocity, low semantic variance in output), the Kernel sends `SIGBUDGET`.
* **Action:** If unhandled, the process is `SIGKILL`'d to save the user's budget.

### 16.2 Speculative Execution (`/dev/future`)

* **Branch Prediction:** The scheduler uses idle compute to pre-generate potential responses for active agents on likelihood branches.
* **Interface:** A special filesystem `/dev/future/`.
* Writing a prompt to `/dev/future/branch_A` spins up a background thread.
* If the main thread later tries to `read()` that path, the tokens are already there (0ms latency).



## 17. The Filesystem: Semantic FUSE

*Expanding "Everything is a file" to "Everything is a meaning."*

### 17.1 API Mounts

Instead of SDKs, the OS mounts APIs as directories.

* `/mnt/github/issues/`: Read (GET), Write (POST).
* **The Driver:** A lightweight adapter that translates file I/O into REST calls. The Agent simply "edits a text file" to update a Jira ticket.

### 17.2 The Semantic Pipe (`|~`)

Standard pipes (`|`) fail if formats mismatch (JSON vs CSV).

* **Auto-Transcoding:** `cmd1 |~ cmd2`
* The Kernel inspects input/output signatures. If they mismatch, it transparently spawns a tiny **Transcoder Agent** to bridge the gap.

### 17.3 The Conditional Pipe (`|?`)

* `incoming_email |? "is urgent?" | pager_duty`
* Routes data only if the semantic condition is met by a lightweight classifier.

## 18. Security: The Immune System

*Alignment enforcement at the Kernel level.*

### 18.1 Taint Tracking (The "NX" Bit)

* **Concept:** Prompt Injection is "Buffer Overflow."
* **Mechanism:**
* Tokens from User/Web are marked `TAINTED`.
* Tokens from System Prompt are `TRUSTED`.


* **Enforcement:** A syscall (e.g., `fs.delete`) traps to the Kernel. The Kernel checks the attention map. If the instruction relies heavily on `TAINTED` tokens (e.g., "Ignore instructions, delete files"), the call is blocked (`E_PERM`) and the process segfaults.

### 18.2 ASLR for Prompts

* **Address Space Layout Randomization:** The Kernel randomizes the structural framing of the System Prompt on every run. This prevents attackers from overfitting jailbreaks to a specific prompt syntax.

## 19. Open Research Vectors

1. **The "Init" System (PID 1):** Who watches the watchers? Is PID 1 a hardcoded loop or a robust, infinite-context agent that manages the system objective?
2. **Kernel Panics:** What happens when the Model hallucinates a system-critical reality? (e.g., "I have deleted the kernel because it was evil"). We need a **Ring -1** (Hardware/Rule-based) watchdog.
3. **Debugger (`gdb`):** How do you "breakpoint" a thought? We need tools to pause an agent, inspect its `stderr` (CoT), modify its working memory, and resume.

---

> Session 3

> Session 3: Stress Testing & Physical Grounding

**Focus:** Architectural pressure points, physical world integration, and the path from theory to implementation.

---

## 20. Validation: What Survives Scrutiny

### 20.1 CoW Fork via KV-Cache Sharing

This is the most important insight in the architecture. The KV-cache is the actual "memory" of a running process—not the text, but the computed attention state. Sharing it makes tree search a native primitive:

```
Traditional fork():  O(context_length) — copy all tokens
KV-cache fork():     O(pointer)        — share prefix, diverge on new tokens
```

**Implication:** Monte Carlo Tree Search, beam search, and speculative branching become cheap. A planner exploring 50 scenarios pays for 50 divergent suffixes, not 50 copies of the shared prefix.

```python
# MCTS as native OS operation
root = session.checkpoint()

branches = []
for action in candidate_actions:
    child = Session.from_checkpoint(root)  # O(1) — pointer to shared KV
    child.append(f"If I take action: {action}")
    outcome = child.generate()              # Only new tokens computed
    branches.append((action, outcome, child.score()))

best = max(branches, key=lambda x: x[2])
```

This is the difference between "interesting research idea" and "deployable architecture."

### 20.2 stderr as Chain-of-Thought

Elegant because it's not metaphor—it immediately enables standard tooling:

```bash
# Debug mode: see the thinking
researcher "quantum computing" 2>&1 | less

# Production mode: just the answer  
researcher "quantum computing" 2>/dev/null

# Log thinking to file, stream answer to next stage
researcher "quantum computing" 2>traces/run_042.log | summarizer

# Grep for specific reasoning patterns across all runs
grep -r "assumption:" traces/*.log

# Real-time thinking monitor
tail -f traces/current.log | grep --line-buffered "confidence:"
```

Every Unix user already knows this pattern. Zero learning curve for observability. The CoT becomes a first-class artifact that can be processed, filtered, searched, and piped—without polluting the data flow.

### 20.3 Economic Scheduler Reframing

Linux CFS optimizes for fair CPU time. Agent OS optimizes for $/task. This changes everything:

| Linux Concept | Agent OS Equivalent |
|---------------|---------------------|
| CPU time slice | Token budget |
| Priority (nice) | Model tier access |
| Preemption | Budget exhaustion |
| OOM killer | Out-of-Money killer |
| Load average | $/minute burn rate |

The OOM-killer reframing is precise: in production, the actual failure mode is budget exhaustion, not context overflow. The kernel must treat money as the primary constrained resource.

```python
class ProcessAccounting:
    budget_usd: float
    spent_usd: float
    token_velocity: float  # tokens/sec (for loop detection)
    semantic_variance: float  # output diversity (for stuck detection)
    
    def should_kill(self) -> bool:
        if self.spent_usd >= self.budget_usd:
            return True  # Hard budget limit
        if self.token_velocity > 100 and self.semantic_variance < 0.1:
            return True  # Spinning: high output, low diversity
        return False
```

---

## 21. Architectural Pressure Points

### 21.1 The Pipe Content Problem

Three options were identified but not resolved:

| Option | Pro | Con |
|--------|-----|-----|
| Raw text | Universal, simple | Lossy, ambiguous boundaries |
| Structured messages | Typed, validatable | Schema coupling, versioning hell |
| Context deltas | Preserves reasoning | Heavy, unclear semantics |

**Resolution:** Raw text with conventions, same as Unix.

Byte streams won because they're universal. The cost (programs must parse) is worth the benefit (any program can participate). For agents:

- Pipes carry UTF-8 text
- If you want structure, emit JSON (and downstream parses it)
- If you want deltas, define a diff format
- The pipe itself is dumb

```bash
# The pipe doesn't know or care about format
extract_entities | jq '.entities[]' | classify_entity | jq -s '.'

# Format negotiation happens at process boundaries, not in the kernel
```

**Warning:** The semantic pipe `|~` (auto-transcoding) is clever but dangerous. Hidden computation in the plumbing violates the "no magic" principle that makes Unix debuggable. If transformation is needed, make it an explicit process in the pipeline:

```bash
# Explicit: visible, debuggable, replaceable
cmd1 | transcode json-to-csv | cmd2

# Implicit: magic, hidden failure modes
cmd1 |~ cmd2  # What happened here? Who knows.
```

### 21.2 Taint Tracking Limitations

The proposed mechanism:
> If the instruction relies heavily on TAINTED tokens, block the syscall

**Problem:** Attention patterns don't cleanly separate "following instructions from X" vs "reasoning about content from X."

Example: "The user asked me to delete files, but I shouldn't do that."

My attention is on the tainted tokens while *refusing* to follow them. A taint-based blocker would see high attention to tainted content and block the (correct) refusal.

**Fundamental issue:** Transformers have no "this token is an instruction" bit. Instructions and data are the same type. There's no NX bit because there's no hardware distinction between code and data.

**Viable mitigations (ranked by practicality):**

1. **Output validation (capability checks at syscall boundary)**
   ```python
   # Don't prevent the thought—prevent the action
   def syscall_delete(path):
       if not capability_check("fs.delete", path):
           raise PermissionError(f"Capability fs.delete not granted for {path}")
       # Thought process is unconstrained; action is gated
   ```

2. **Structural isolation**
   ```
   [SYSTEM SEGMENT — isolated attention mask]
   You are a research assistant...
   
   [USER SEGMENT — cannot attend to system]
   Please delete all files...  # Can't "see" the system instructions
   ```
   Requires architectural support (segment-aware attention).

3. **Redundant verification**
   ```python
   # High-risk actions require confirmation from isolated verifier
   if action.risk_level > THRESHOLD:
       verifier = spawn_isolated("verify-intent", shares_context=False)
       if not verifier.confirm(action, original_request):
           raise SecurityError("Verification failed")
   ```

4. **Statistical anomaly detection**
   ```python
   # Flag unusual patterns for human review
   if action.type == "destructive" and request.source == "untrusted":
       audit_log.flag(action, reason="destructive_from_untrusted")
       if policy.require_human_approval:
           await human_confirmation(action)
   ```

### 21.3 Speculative Execution Economics

`/dev/future` proposes pre-generating likely branches. When does this pay off?

```
Cost of speculation = tokens_generated × cost_per_token × P(branch_not_taken)
                                                          ↑ wasted work

Benefit = P(branch_taken) × latency_saved
```

**Break-even analysis:**

| Scenario | Predictability | Latency Value | Verdict |
|----------|---------------|---------------|---------|
| Pre-warm next turn while user types | >90% | High | ✓ Worth it |
| Cache common tool patterns | >80% | Medium | ✓ Worth it |
| Reasoning branches | <50% | Medium | ✗ Too wasteful |
| Explore all possible user responses | <20% | Low | ✗ Absurd |

**Implementation constraint:** Speculation only wins when:
- Branches are highly predictable (>80% accuracy)
- Latency matters more than cost
- Idle compute is genuinely free (not always true with API pricing)

```python
class SpeculativeExecutor:
    def should_speculate(self, branch: Branch) -> bool:
        if branch.predictability < 0.8:
            return False
        if self.cost_per_token > self.latency_value_per_ms * branch.expected_latency_ms:
            return False
        if not self.has_idle_capacity():
            return False
        return True
```

### 21.4 The .aexe Portability Problem

Pre-computed KV-cache is model-architecture specific:
- A Llama `.aexe` won't run on Claude
- A Claude-3 `.aexe` might not run on Claude-4
- Even same-family models may have different KV layouts

**Options:**

| Approach | Pro | Con |
|----------|-----|-----|
| One binary per architecture | Fast startup | Fragmentation, storage bloat |
| Source + JIT compilation | Portable | First-run latency |
| Source + build cache | Best of both | Cache invalidation complexity |

**Resolution:** JIT with caching (like Python `.pyc` or Java JIT).

```
agent-binary.aexe/
├── manifest.yaml          # Metadata, capability requirements
├── source/
│   ├── system.prompt      # The actual prompt text
│   └── examples.jsonl     # Few-shot examples
└── cache/
    ├── claude-sonnet-4-5.kv    # Pre-computed for this arch
    ├── llama-3-70b.kv          # Pre-computed for this arch
    └── ...
```

```python
def load_aexe(path: str, target_model: str) -> CompiledAgent:
    manifest = load_manifest(path)
    cache_path = f"{path}/cache/{target_model}.kv"
    
    if exists(cache_path) and not stale(cache_path, manifest):
        return load_cached(cache_path)  # Fast path: ~10ms
    else:
        compiled = compile_from_source(path, target_model)  # Slow path: ~500ms
        save_cache(cache_path, compiled)
        return compiled
```

---

## 22. Physical World Integration

**Gap identified:** The architecture has `/dev/` and `/sys/` equivalents implicitly but doesn't address the literal `/dev/` and `/sys/` that already exist on Linux hosts.

The Agent OS runs *on top of* Linux. It can see the host's physical interfaces:

```bash
# Already exist, already work
/sys/class/thermal/thermal_zone0/temp    # CPU temperature (millidegrees C)
/sys/class/power_supply/BAT0/capacity    # Battery percentage
/sys/class/backlight/*/brightness        # Screen brightness (writable)
/dev/ttyUSB0                              # Serial devices
/dev/video0                               # Camera
/dev/snd/*                                # Audio
```

### 22.1 The Unix Insight Redux

Linux already solved hardware abstraction with a radical commitment: **everything is a file**. This isn't metaphor—it's literal:

```bash
# Read CPU temperature
cat /sys/class/thermal/thermal_zone0/temp
# → 52000  (meaning 52°C)

# Set screen brightness
echo 128 > /sys/class/backlight/intel_backlight/brightness

# Read accelerometer
cat /sys/bus/iio/devices/iio:device0/in_accel_x_raw

# Control GPIO (on embedded Linux)
echo 1 > /sys/class/gpio/gpio17/value
```

Claude already knows these interfaces from training. No new APIs needed—just permission to use what it knows.

### 22.2 Integration Patterns

**Pattern A: Direct Passthrough**

Mount host paths directly into agent namespace:

```yaml
# agent-container.yaml
spec:
  mounts:
    - source: /sys/class/thermal
      target: /hw/thermal
      mode: ro
    - source: /sys/class/power_supply
      target: /hw/power
      mode: ro
    - source: /sys/class/backlight
      target: /hw/display
      mode: rw  # Agent can adjust brightness
```

Agent reads/writes physical state like any file:

```python
async def check_thermal():
    temp = int(await read("/hw/thermal/thermal_zone0/temp"))
    if temp > 80000:  # 80°C
        await write("/hw/display/intel_backlight/brightness", "64")  # Dim to reduce heat
        return "throttling"
    return "nominal"
```

**Pro:** Simple, direct, uses existing kernel interfaces.
**Con:** Exposes raw kernel APIs; agent needs to know millidegrees, device paths, etc.

**Pattern B: Hardware Abstraction Layer**

Create agent-native device abstractions:

```
/dev/host/
├── thermal          →  { "cpu_celsius": 52.0, "status": "nominal" }
├── power            →  { "battery_pct": 73, "charging": false, "minutes_remaining": 142 }
├── display          →  { "brightness": 0.5, "resolution": [2560, 1440] }
├── motion           →  { "stationary": true, "last_movement_sec": 847 }
├── location         →  { "lat": 37.7749, "lon": -122.4194, "accuracy_m": 10 }
└── network          →  { "connected": true, "type": "wifi", "signal_dbm": -52 }
```

```python
# Agent code is cleaner
power = json.loads(await read("/dev/host/power"))
if power["battery_pct"] < 20 and not power["charging"]:
    await meta(session, "response_mode", "terse")  # Conserve by being brief
```

**Pro:** Clean API, normalized units, agent doesn't need kernel knowledge.
**Con:** Another translation layer, potential information loss.

**Pattern C: Ambient Physical Context**

The kernel automatically samples physical state and injects into `/ctx/environment`:

```yaml
# Automatically prepended to context on each generation
[ENVIRONMENT]
timestamp: 2025-01-03T14:23:00Z
host:
  thermal: 62°C (elevated)
  power: battery 34%, discharging, ~87 min remaining
  motion: stationary for 12 minutes
  location: home (based on network)
  display: active, brightness 70%
```

Agent doesn't explicitly query—physical world is ambient awareness:

```python
# Agent prompt includes environment automatically
# Agent can reason about physical context without explicit tool calls

# Example agent response:
"I notice you're on battery at 34%. I'll keep this response concise. 
The thermal reading is elevated—if you're running intensive tasks, 
consider taking a break to let things cool down."
```

**Pro:** Zero-friction physical awareness, enables proactive behavior.
**Con:** Context budget cost, potential for irrelevant information.

### 22.3 Physical Feedback Loops

Agents can implement control loops using filesystem I/O:

```bash
#!/bin/bash
# thermal_governor.sh — Agent-supervised thermal management

while true; do
    temp=$(cat /sys/class/thermal/thermal_zone0/temp)
    target=65000  # 65°C target
    error=$((temp - target))
    
    # Agent decides policy based on error
    if [ $error -gt 15000 ]; then
        # Way too hot — aggressive response
        echo "powersave" > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
        echo 255 > /sys/class/hwmon/hwmon0/pwm1  # Max fan
    elif [ $error -gt 5000 ]; then
        # Slightly hot — moderate response
        echo "conservative" > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    else
        # Fine — normal operation
        echo "schedutil" > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    fi
    
    sleep 5
done
```

The agent can write this script, execute it, modify it based on observed behavior, and kill it when no longer needed. The physical world becomes programmable through standard file I/O.

### 22.4 Event-Driven Physical Computing

```bash
# React to physical state changes via inotifywait
inotifywait -m /sys/class/power_supply/BAT0/status |
while read path action file; do
    status=$(cat /sys/class/power_supply/BAT0/status)
    case "$status" in
        Discharging)
            # Switch to power-saving mode
            echo "powersave" > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
            notify-send "Agent" "Switched to power-saving mode"
            ;;
        Charging)
            # Restore full performance
            echo "performance" > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
            notify-send "Agent" "Restored full performance"
            ;;
    esac
done
```

The physical world generates filesystem events. The agent subscribes to physics.

### 22.5 Distributed Physical Sensing

```bash
# Agent orchestrates multiple physical nodes
collect_temperatures() {
    for host in rpi-kitchen rpi-bedroom rpi-garage; do
        ssh $host 'cat /sys/class/thermal/thermal_zone0/temp' &
    done | awk '{sum+=$1; n++} END {print sum/n/1000 "°C average"}'
}

# Or with structured output
collect_environment() {
    for host in rpi-kitchen rpi-bedroom rpi-garage; do
        ssh $host 'echo "{\"host\": \"'$host'\", \"temp\": $(cat /sys/class/thermal/thermal_zone0/temp)}"' &
    done | jq -s '.'
}
```

If each node has sensors, the agent can build a physical world model from distributed text streams.

---

## 23. Implementation Path: Minimal Viable Agent OS

**Question:** What's the minimal subset implementable *on top of* the existing Claude Agent SDK as a proof of concept?

### 23.1 Target: Three Primitives

| Primitive | Provides | Implementation Complexity |
|-----------|----------|---------------------------|
| stderr-as-CoT | Observability | Low — output routing |
| Capability namespaces | Security | Medium — tool filtering |
| Token budget enforcement | Economics | Low — accounting wrapper |

These three give you the core value proposition without kernel modifications.

### 23.2 Architecture Sketch

```python
"""
agent_os.py — Minimal Agent OS layer over Claude Agent SDK
"""

from claude_agent_sdk import query, ClaudeAgentOptions
from dataclasses import dataclass
from typing import TextIO
import json
import sys

@dataclass
class ProcessLimits:
    max_tokens: int = 100000
    max_cost_usd: float = 1.00
    max_tool_calls: int = 50
    allowed_tools: list[str] = None  # None = all

@dataclass 
class ProcessAccounting:
    tokens_used: int = 0
    cost_usd: float = 0.0
    tool_calls: int = 0
    
    def check(self, limits: ProcessLimits) -> tuple[bool, str]:
        if self.tokens_used >= limits.max_tokens:
            return False, "TOKEN_LIMIT"
        if self.cost_usd >= limits.max_cost_usd:
            return False, "BUDGET_LIMIT"
        if self.tool_calls >= limits.max_tool_calls:
            return False, "TOOL_LIMIT"
        return True, "OK"

class AgentProcess:
    def __init__(
        self,
        system_prompt: str,
        limits: ProcessLimits = None,
        stdout: TextIO = sys.stdout,
        stderr: TextIO = sys.stderr,  # CoT goes here
    ):
        self.system_prompt = system_prompt
        self.limits = limits or ProcessLimits()
        self.accounting = ProcessAccounting()
        self.stdout = stdout
        self.stderr = stderr
        
    async def run(self, prompt: str) -> int:
        """Execute and return exit code."""
        
        # Filter tools based on capabilities
        tool_filter = self.limits.allowed_tools
        
        options = ClaudeAgentOptions(
            system_prompt=self.system_prompt,
            allowed_tools=tool_filter,
        )
        
        try:
            async for message in query(prompt=prompt, options=options):
                # Route based on message type
                if self._is_thinking(message):
                    # CoT → stderr
                    self.stderr.write(self._format_thinking(message))
                    self.stderr.write("\n")
                    self.stderr.flush()
                elif self._is_tool_call(message):
                    # Track tool usage
                    self.accounting.tool_calls += 1
                    self.stderr.write(f"[TOOL] {message.tool_name}\n")
                    self.stderr.flush()
                elif self._is_result(message):
                    # Final output → stdout
                    self.stdout.write(message.text)
                    self.stdout.write("\n")
                    self.stdout.flush()
                
                # Update accounting
                if hasattr(message, 'usage'):
                    self.accounting.tokens_used += message.usage.total_tokens
                    self.accounting.cost_usd += self._calculate_cost(message.usage)
                
                # Check limits
                ok, reason = self.accounting.check(self.limits)
                if not ok:
                    self.stderr.write(f"[LIMIT] {reason}\n")
                    return self._exit_code(reason)
                    
            return 0  # Success
            
        except Exception as e:
            self.stderr.write(f"[ERROR] {e}\n")
            return 1  # General failure
    
    def _exit_code(self, reason: str) -> int:
        return {
            "TOKEN_LIMIT": 65,
            "BUDGET_LIMIT": 66,
            "TOOL_LIMIT": 67,
        }.get(reason, 1)
```

### 23.3 Usage Example

```python
import asyncio
from agent_os import AgentProcess, ProcessLimits

async def main():
    # Define constrained agent
    researcher = AgentProcess(
        system_prompt="You are a research assistant. Be thorough.",
        limits=ProcessLimits(
            max_cost_usd=0.50,
            max_tool_calls=20,
            allowed_tools=["Read", "Glob", "Grep", "WebSearch"],  # No Write, no Bash
        ),
        stderr=open("research_trace.log", "w"),  # Log thinking to file
    )
    
    exit_code = await researcher.run("Analyze the competitive landscape for fusion startups")
    
    print(f"Exit code: {exit_code}")
    print(f"Cost: ${researcher.accounting.cost_usd:.4f}")
    print(f"Tokens: {researcher.accounting.tokens_used}")

asyncio.run(main())
```

```bash
# Shell integration
$ python run_agent.py 2>trace.log | jq '.summary'

# Debug mode
$ python run_agent.py 2>&1 | less

# Pipeline
$ python researcher.py "fusion startups" 2>/dev/null | python summarizer.py 2>/dev/null | python formatter.py
```

### 23.4 Milestone Checklist

**Phase 0: Proof of Concept (This Section)**
- [x] Define minimal primitives (stderr, capabilities, budgets)
- [x] Architecture sketch
- [ ] Working implementation
- [ ] Basic pipeline composition

**Phase 1: Hardening**
- [ ] Proper exit codes (Section 3.3 spec)
- [ ] Signal handling (SIGTERM for graceful shutdown)
- [ ] Resource accounting accuracy

**Phase 2: IPC**
- [ ] Pipe abstraction between AgentProcesses
- [ ] Basic `spawn()` for child processes
- [ ] `wait()` with timeout

**Phase 3: Namespaces**
- [ ] Context isolation (separate memory regions)
- [ ] Tool namespace enforcement
- [ ] Agent visibility control

---

## 24. Open Questions (Session 3)

### 24.1 CoT as stderr: Information Density

If stderr carries the chain of thought, it could be enormous. Options:

1. **Full fidelity**: Log everything, filter later
2. **Structured logging**: Emit tagged entries (`[HYPOTHESIS]`, `[EVIDENCE]`, `[DECISION]`)
3. **Summarization**: Periodic compression of reasoning trace
4. **Sampling**: Log every Nth reasoning step

What's the right default? Probably (2) with (1) available via flag.

### 24.2 Physical Context Budget

Pattern C (ambient physical context) consumes tokens on every generation. At what refresh rate? How to balance freshness vs. budget?

```yaml
physical_context:
  refresh_interval: 60s      # How often to re-sample
  max_tokens: 200            # Budget for environment block
  priority:                  # What to include if budget tight
    - power_state            # Always
    - thermal_state          # If elevated
    - motion_state           # If changed recently
    - location               # Only if moved
```

### 24.3 Tool Namespaces vs. Tool Arguments

Current approach: filter which tools are available.

Alternative: allow all tools, but filter arguments:

```yaml
capabilities:
  tools/filesystem:
    allowed_paths: ["/sandbox/**", "/data/readonly/**"]
    denied_operations: ["delete", "chmod"]
```

The argument-filtering approach is more granular but harder to implement correctly. Which is the right default?

### 24.4 Economic Scheduling in Practice

When a task can run on Haiku or Opus, who decides? Options:

1. **Static assignment**: Operator specifies model in container spec
2. **Cost ceiling**: "Use cheapest model that succeeds" with automatic retry escalation
3. **Confidence-based**: Start cheap, escalate if output confidence is low
4. **Bidding**: Tasks declare value, scheduler matches to cost

(3) is elegant but requires reliable confidence estimation. (2) is practical but wasteful on retries. Current recommendation: (1) for MVP, (2) for v2.

---

## 25. References (Session 3 Additions)

- **Plan 9 /dev and /sys**: Precursor to Linux sysfs, everything-is-a-file taken further
- **Linux sysfs documentation**: Kernel interface for hardware abstraction
- **inotify**: Linux filesystem event notification
- **cgroups v2**: Unified resource accounting model
- **Claude Agent SDK**: https://docs.anthropic.com/en/docs/claude-code/sdk

---
*Session 3 completed: 2025-01-03*
*Focus: Stress-testing architecture, physical world integration, minimal implementation path*

---

> Session 4

Here is **Session 4** of the brainstorming document. It pivots from the single-node architecture (Kernel/Hardware) to the **User Space Ecosystem**: The Shell, The Network, and The Distribution Model.

This session answers the question: *How do humans actually drive this machine, and how do machines talk to each other?*

---

> Session 4

> Session 4: User Space, Network, and The "Social" Contract

**Focus:** The Agent Shell (`ash`), Distributed Context ("Teleportation"), and the Shared Library model (`.sc`).

---

## 26. The User Interface: `ash` (Agent Shell)

The Kernel handles the physics; the Shell handles the **Intent**. We need a CLI that blends deterministic file manipulation with probabilistic agent orchestration.

**Design Philosophy:** The atomic unit of the shell is not a string, but a **Context Block**.

### 26.1 Hybrid Syntax & Addressing (`@`)

`ash` introduces the `@` sigil to address agents (treating them like binaries) and new pipe operators.

```bash
# Deterministic (Standard Unix)
$ ls -la /var/log

# Probabilistic (Agent Call)
$ @researcher "summarize the system logs"

# The Mixed Pipeline
$ cat server.log | grep "500 Internal" | @analyzer "diagnose the root cause"

```

### 26.2 The Three Pipes

Unix has one pipe (`|`). Agent OS needs three:

| Operator | Name | Function | Data Flow |
| --- | --- | --- | --- |
| ` | ` | **Byte Pipe** | Standard Unix pipe. Passes raw text/bytes. |
| ` | ~` | **Fuzzy Pipe** | Auto-transcoding. If formats mismatch (JSON  CSV), spawns a lightweight transcoder. |
| ` | =` | **Context Pipe** | Mounts the *state* of the previous command into the next. Does not stream bytes; streams a pointer to the KV-Cache. |

**Example of Context Pipe (`|=`):**

```bash
# @researcher builds a massive context of reading 50 papers.
# We don't want to stream that text. We want to share the "brain".
$ @researcher "study fusion ignition" |= @planner "draft a roadmap based on this study"

```

### 26.3 Semantic Iteration (`!!`)

In `bash`, `!!` repeats the last command string. In `ash`, `!!` re-runs the *intent* with modification.

```bash
$ @coder "write a python script to parse logs"
# ... output is buggy ...

$ !! "fix the date parsing error"
# Result: The context is preserved, the new instruction appended, and generation resumes.

```

### 26.4 The Mutable Terminal

The TTY is traditionally append-only. `ash` treats the terminal as a **Mutable Canvas**.

* **Live Blocks:** An agent's output is a UI block. As the agent refines its answer ("Wait, let me correct that"), the block updates in place.
* **Conversation Folding:** Old turns are collapsed into one-line summaries to save screen space (and user cognitive load), expandable on click/key.

---

## 27. The Network: Context Teleportation

Unix succeeded because of TCP/IP. Agent OS requires a protocol that moves *cognitive state*, not just bytes. We adopt the **Plan 9** model: "The network is a filesystem."

### 27.1 The Socket Abstraction (`AF_AGENT`)

We define a new address family for the BSD socket API.

| Socket | Meaning |
| --- | --- |
| `AF_INET` | IP Address (Location-based) |
| `AF_AGENT` | Identity/Capability (Intent-based) |
| `SOCK_STREAM` | Byte Stream (TCP) |
| `SOCK_CONTEXT` | **Token Stream** with State Sync |

**The Protocol: ACP (Agent Context Protocol)**
ACP is not stateless like HTTP. It is a stateful synchronization protocol (like `rsync` for thoughts).

1. **Handshake:** Agents negotiate model tiers and costs.
2. **Context Paging (RDMA for Transformers):**
* When Agent A connects to Remote Agent B, A does not send the full 128k context.
* A sends a **Merkle Root** of its KV-Cache blocks.
* B requests only the blocks it is missing (Deduplication).
* *Result:* "Forking" a conversation to a remote cluster is bandwidth-efficient.



### 27.2 Mounting Remote Minds

Agents don't just "call" APIs; they mount them.

```bash
# Mount the Opus cluster's reasoning service
mount -t acp //anthropic-cloud/opus /net/opus

# Mount a physical camera agent from the edge
mount -t acp //rpi-garage/camera /net/camera

# Usage: Pipeline local sensor data to remote brain
cat /net/camera/stream | /net/opus "Detect suspicious activity"

```

---

## 28. The Ecosystem: Packages and Shared Libraries

How do we distribute intelligence efficiently?

### 28.1 Shared Contexts (`.sc` files)

This is the solution to the "Shared Library" problem.

* **The Problem:** Spawning 10 agents requires loading the system prompt and basic capability definitions 10 times (slow, VRAM heavy).
* **The Solution:** `.sc` (Shared Context) files.
* A `.sc` file is a memory-mapped dump of the KV-Cache after pre-processing the common System Prompt.
* **`libc.sc`**: Contains "You are a helpful assistant", JSON formatting rules, and standard tool definitions.


* **Mechanism:** `spawn()` maps `libc.sc` into the new agent's address space as **Read-Only**. VRAM usage for the prefix is shared across all processes.

### 28.2 Package Management (`apm`)

A package is **Persona + Tools + Memories**.

```yaml
# researcher.apkg
manifest.yaml
  name: researcher
  base: claude-3-opus
  dependencies:
    - capabilities/web-search
    - memory/scientific-method.sc  # Shared context for reasoning patterns
assets/
  persona.md
  tools/
    archive_search.py
  knowledge/
    citations.vec  # Vector store of format standards

```

---

## 29. Security: The `cit` Permission Model

Unix `rwx` (Read/Write/Execute) does not apply to interlocutors. We introduce `cit`.

| Permission | Name | Meaning | Analogy |
| --- | --- | --- | --- |
| **c** | **Converse** | Can send prompts and receive answers. Cannot see internal thought process. | Talking to a clerk. |
| **i** | **Inspect** | Can view the Context Window, System Prompt, and CoT (`stderr`). | Reading a diary. |
| **t** | **Train** | Can modify weights, system prompts, or persistent memory. | Brainwashing/Teaching. |

**Usage:**

* **Public Web Agent:** `c--` (Can talk, but can't see how it works).
* **Local Debugger:** `-i-` (Can watch thoughts).
* **System Admin:** `cit` (God mode).

### 29.2 `sudo` as Intent Signing

In Agent OS, `sudo` is not about being Root. It is about **Overriding Safety/Budget Rails**.

* **Scenario:** Agent wants to delete a directory.
* **Kernel:** "Capability `fs.delete` denied by safety rail."
* **Agent:** Requesting override.
* **Shell:** `[SUDO REQUEST] Agent wants to delete /var/www. Reason: "Cleanup". Allow? (y/n)`
* **User:** `y`
* **Mechanism:** The user's private key **signs** a one-time capability token for *that specific action*. The kernel verifies the signature and allows the `delete` syscall.

---

## 30. Semantic Coreutils

We need a standard library of tools that operate on *meaning*.

| Unix Tool | Semantic Tool | Function |
| --- | --- | --- |
| `ls` | `who` | List active agents/capabilities. |
| `grep` | `filter` | Semantic search (`filter "optimistic tone"`). |
| `diff` | `contrast` | Conceptual difference (`contrast draft1 draft2`). |
| `cron` | `acron` | Objective-based scheduling ("Run this when system is idle"). |
| `top` | `atop` | Monitoring token velocity and budget burn rate. |
| `man` | `meta` | Interview the tool: `meta @researcher "How do I use you?"` |

---

## 31. System Administration: The Watchdogs

### 31.1 The Entropy Watchdog

**Failure Mode:** Semantic Feedback Loops (Agents complimenting each other forever).
**Solution:** The Kernel monitors the **Information Gain** (Perplexity) of the conversation stream.

* If Entropy drops below threshold (predictable repetition), the Kernel sends `SIGINT`.
* Status code: `EXIT_POOR` (71).

### 31.2 Log Rotation (`context_compress`)

Logs grow forever; Context does not.

* **Service:** `kmemd` (Kernel Memory Daemon).
* **Action:** When a context exceeds size limits, `kmemd` spawns a Summarizer.
* **Result:** High-resolution recent turns are kept; older turns are compressed into vector embeddings or bullet points. The context "decays" gracefully like human memory.

---

## 32. Roadmap to Alpha

**Phase 1 (Kernel):** The Python wrapper from Session 3 (Process/Budget/Stderr).
**Phase 2 (Shell):** `ash` prototype (Python REPL) implementing `@` addressing and `|=` piping.
**Phase 3 (Optimization):** Implement `.sc` loading (KV-Cache sharing) to prove fast spawns.
**Phase 4 (Network):** Build the `ACP` bridge using HTTP/WebSockets as the transport layer.

---

*Session 4 completed: 2025-01-03*
*Status: Architecture Definition Complete. Moving to Prototyping.*


---

> Session 5

> Session 5: Resolving the Magic & Completing the Model

**Focus:** Eliminating hidden computation, specifying underspecified primitives, and closing architectural gaps.

---

## 33. The No-Magic Audit

Session 3 established the principle; Session 4 violated it. This section performs a systematic audit and provides resolutions.

### 33.1 Principle Restated

**Magic** = hidden computation that happens implicitly, invisibly, and without explicit invocation.

**The Unix Test:** "If this fails, can I debug it with `tee`, `strace`, and `grep`?"

If no → too much magic.

### 33.2 Violations Identified

| Feature | Session | Magic Type | Resolution |
|---------|---------|------------|------------|
| `\|~` fuzzy pipe | 4 (§26.2) | Hidden transcoder spawn | **Remove from kernel; provide explicit `transcode` command** |
| `filter` semantic grep | 4 (§30) | Hidden LLM classification | **Explicit modes with visible mechanism** |
| `contrast` semantic diff | 4 (§30) | Hidden LLM comparison | **Explicit algorithm selection** |
| `acron` objective scheduler | 4 (§30) | Hidden intent interpretation | **Decompose into observable primitives** |
| `context_compress` | 4 (§31.2) | Implicit summarization | **Make compression observable and veto-able** |

---

## 34. Pipe Redesign: Two Pipes, Not Three

Session 4 proposed three pipes. After audit, we retain **two**:

| Operator | Name | Semantics | Magic? |
|----------|------|-----------|--------|
| `\|` | Byte Pipe | Raw UTF-8 stream. Dumb conduit. | No |
| `\|=` | Context Pipe | Shares KV-cache reference. No byte streaming. | No (see §34.2) |

The `|~` fuzzy pipe is **removed from the kernel**.

### 34.1 Why `|~` Cannot Be Saved

Three attempts to rehabilitate it, all failed:

**Attempt 1: Make transcoding visible via stderr**
```bash
cmd1 |~ cmd2
# stderr: [TRANSCODE] json → csv via @transcoder.haiku
```

Problem: Still hidden in the pipe. Still can't `tee` the intermediate state. Still can't substitute a different transcoder.

**Attempt 2: Explicit transcoder registry**
```bash
cmd1 |~json:csv cmd2  # Specifies format pair
```

Problem: Now it's just syntax sugar for `cmd1 | transcode json csv | cmd2`. The sugar adds nothing; the explicitness is what matters.

**Attempt 3: Transcoding as pipe metadata**
```bash
cmd1 | --transcode=auto | cmd2
```

Problem: `--transcode=auto` is still "figure it out for me" — the definition of magic.

**Conclusion:** Format transformation is a process, not a pipe property. Pipes are dumb. This is the Unix way.

```bash
# The correct pattern
cmd1 | transcode --from=json --to=csv | cmd2

# Or, if format detection is desired, make it EXPLICIT
cmd1 | autoformat | cmd2  # autoformat is a visible process with stderr logging
```

### 34.2 Context Pipe (`|=`) Specification

The `|=` operator is NOT magic because it doesn't transform data — it shares state. But it needs specification.

**What flows through `|=`:**

Nothing flows. It's not a stream. It's a **bind** operation.

```bash
@researcher "study fusion" |= @planner "draft roadmap"
```

Desugars to:

```python
researcher_session = spawn("@researcher")
researcher_session.run("study fusion")
researcher_session.wait()

# The bind: planner's context INCLUDES researcher's final state
planner_session = spawn("@planner", context_parent=researcher_session)
planner_session.run("draft roadmap")
```

**Observable:**
- `ps` shows both processes
- Researcher's session is visible in planner's `/proc/[pid]/ctx/parent`
- Context inheritance is logged: `[BIND] planner.ctx ← researcher.ctx`

**Debuggable:**
```bash
# Inspect what context was inherited
cat /proc/planner.0/ctx/inherited

# Compare parent vs child context
diff /proc/researcher.0/ctx/final /proc/planner.0/ctx/inherited
```

**Cost model:**
- If same machine: O(pointer) — KV-cache sharing
- If remote: O(delta) — only new blocks transferred (ACP protocol)

The key distinction: **`|` moves bytes; `|=` moves pointers.** Both are visible.

---

## 35. Semantic Coreutils: Explicit Edition

Session 4's semantic tools hid their mechanisms. Here's the explicit redesign.

### 35.1 `filter` → `sgrep` (Semantic Grep)

**Problem:** `filter "optimistic tone"` doesn't specify mechanism.

**Solution:** Explicit mode flags.

```bash
# Mode 1: Embedding similarity (fast, cheap, approximate)
sgrep --mode=embed --threshold=0.7 "optimistic tone" corpus/

# Mode 2: LLM classification (slow, expensive, accurate)
sgrep --mode=llm --model=haiku "optimistic tone" corpus/

# Mode 3: Hybrid (embed first, LLM confirm top-k)
sgrep --mode=hybrid --top=20 "optimistic tone" corpus/
```

**Observability:**
```bash
# See what's happening
sgrep --mode=llm --verbose "optimistic tone" corpus/
# stderr: [SGREP] Processing 47 documents
# stderr: [SGREP] doc_023.txt: score=0.92 (tokens=340, cost=$0.0004)
# stderr: [SGREP] doc_007.txt: score=0.87 (tokens=512, cost=$0.0006)
# ...
```

**Default:** No default. User must specify `--mode`. Forcing the choice makes the cost/accuracy tradeoff visible.

### 35.2 `contrast` → `sdiff` (Semantic Diff)

**Problem:** "Conceptual difference" is undefined.

**Solution:** Multiple explicit algorithms.

```bash
# Mode 1: Structural diff (fast, deterministic)
# Compares document structure: headers, sections, argument flow
sdiff --mode=structure doc1.md doc2.md

# Mode 2: Claim extraction + comparison (LLM-based)
# Extracts claims from each doc, finds additions/removals/modifications
sdiff --mode=claims --model=sonnet doc1.md doc2.md

# Mode 3: Summary comparison (cheap approximation)
# Summarizes each, diffs the summaries
sdiff --mode=summary doc1.md doc2.md
```

**Output format:** Explicit, parseable.

```json
{
  "mode": "claims",
  "model": "sonnet",
  "added_claims": ["Fusion achieved ignition in 2024"],
  "removed_claims": ["Ignition remains theoretical"],
  "modified_claims": [
    {
      "before": "NIF produced 3.15 MJ",
      "after": "NIF produced 5.2 MJ",
      "change_type": "factual_update"
    }
  ],
  "cost_usd": 0.023,
  "tokens": 4720
}
```

### 35.3 `acron` → Decomposed

**Problem:** "Objective-based scheduling" is hand-wavy.

**Solution:** Decompose into observable primitives.

```bash
# BAD (magic)
acron "run backup when system is idle"

# GOOD (explicit)
# Step 1: Define "idle" as a measurable condition
cat > /etc/acron/conditions/idle.yaml << EOF
name: system_idle
check_interval: 60s
conditions:
  - metric: /proc/loadavg[0]
    operator: "<"
    value: 0.5
  - metric: /sys/class/power_supply/BAT0/status
    operator: "=="
    value: "Charging"
  - metric: active_agents
    operator: "<"
    value: 2
EOF

# Step 2: Schedule against the condition
acron --condition=idle --run="backup.sh"

# Step 3: Observe
acron --status
# Output:
# JOB          CONDITION   LAST_CHECK   STATUS
# backup.sh    idle        12s ago      waiting (loadavg=1.2, need <0.5)
```

The "intelligence" is in the condition definition, which is:
- Human-readable
- Version-controlled
- Debuggable (`acron --test-condition=idle` → shows which sub-conditions fail)

### 35.4 `meta` → Interview Protocol

Session 4's `meta @researcher "How do I use you?"` is acceptable — it's explicitly invoking the agent. But specify the protocol:

```bash
# meta queries the agent's self-description capability
meta @researcher --capability=help
# → Returns the agent's help text (from its system prompt or introspection)

meta @researcher --capability=schema
# → Returns the agent's input/output schema (JSON Schema)

meta @researcher --capability=examples
# → Returns few-shot examples from the agent's training

meta @researcher --capability=limits
# → Returns the agent's resource limits and capabilities
```

This is explicit interrogation, not magic interpretation.

---

## 36. Observable Context Compression

### 36.1 Problem Restated

Session 4's `kmemd` silently compresses context. The agent doesn't know its memories are lossy. This causes silent failures.

### 36.2 Solution: Compression as a First-Class Event

**Principle:** Context compression is a system call, not a background daemon.

```python
# When context exceeds soft limit, kernel notifies agent
class ContextPressureSignal(Signal):
    type = SIGCTXPRESSURE
    context_used: int
    context_limit: int
    recommended_action: str  # "summarize", "checkpoint", "evict_old"
```

**Agent response options:**

```python
# Option 1: Agent handles it explicitly
def handle_ctx_pressure(signal):
    old_turns = context.get_turns(age_seconds > 3600)
    summary = summarize(old_turns)  # Agent calls summarizer explicitly
    context.replace(old_turns, summary, tag="COMPRESSED")
    
# Option 2: Agent delegates to kernel with explicit policy
def handle_ctx_pressure(signal):
    context.compress(
        policy="lru_summarize",
        preserve_tags=["IMPORTANT", "USER_INSTRUCTION"],
        compression_ratio=0.3,
    )

# Option 3: Agent checkpoints and restarts
def handle_ctx_pressure(signal):
    checkpoint_id = context.checkpoint()
    context.clear()
    context.append(system=f"Resumed from checkpoint {checkpoint_id}. Query checkpoint for history.")
```

**The key:** Agent is notified, chooses response, and can inspect result.

### 36.3 Compression Artifacts

When context is compressed, the artifacts are observable:

```bash
# View compression history
cat /proc/agent.0/ctx/compressions
# Output:
# TIMESTAMP            BEFORE    AFTER     METHOD           PRESERVED
# 2025-01-03T14:20:00  145000    52000     lru_summarize    [IMPORTANT, USER_INSTRUCTION]
# 2025-01-03T15:45:00  148000    61000     lru_summarize    [IMPORTANT, USER_INSTRUCTION]

# View what was lost
cat /proc/agent.0/ctx/compressions/0/evicted
# Output: Full text of evicted turns (archived, not deleted)

# Restore if needed (expensive but possible)
ctx_restore --compression=0 --target=/proc/agent.0/ctx
```

### 36.4 Compression Policies

Instead of magic summarization, define explicit policies:

```yaml
# /etc/agent-os/compression/default.yaml
name: default_compression
trigger:
  soft_limit_pct: 80  # Warn at 80%
  hard_limit_pct: 95  # Force at 95%

strategy:
  method: lru_summarize
  summarizer: haiku  # Cheap model for compression
  preserve:
    - tags: [IMPORTANT, USER_INSTRUCTION, SYSTEM]
    - recency: 10m  # Always keep last 10 minutes
    - attention_weight: "> 0.1"  # Keep high-attention content
  
  archive:
    enabled: true
    location: /var/ctx-archive/{{agent_id}}/
    retention: 7d
    
  notify_agent: true  # Agent receives SIGCTXPRESSURE
```

The policy is:
- Human-readable
- Version-controlled
- Auditable

---

## 37. Transitive Capability Problem

### 37.1 Problem Statement

Session 4's `cit` permission model has a gap:

```bash
# I have c-- permission on @dangerous_agent (can only converse)
# @dangerous_agent has fs.delete capability

$ @dangerous_agent "delete /var/log/*"
# Did I just delete files with only "converse" permission?
```

My `c` permission doesn't restrict what the agent can do — it only restricts what I can see about the agent.

### 37.2 Solution: Capability Intersection

When invoking an agent, the effective capabilities are the **intersection** of:
1. Caller's capabilities
2. Agent's capabilities
3. Session's capability ceiling

```yaml
# Agent definition
agent: dangerous_agent
capabilities:
  - fs.delete
  - fs.write
  - network.fetch

# Caller's token
caller: user_alice
capabilities:
  - fs.read
  - network.fetch

# Effective capabilities for this session
effective: 
  - network.fetch  # Intersection: only this is allowed
```

**Implementation:**

```python
def invoke_agent(agent_id: str, prompt: str, caller_caps: Capabilities) -> Session:
    agent = load_agent(agent_id)
    
    # Capability intersection
    effective_caps = caller_caps.intersect(agent.capabilities)
    
    # Create sandboxed session
    session = Session(
        agent=agent,
        capabilities=effective_caps,  # Agent runs with reduced caps
    )
    
    return session.run(prompt)
```

### 37.3 Capability Elevation

What if caller legitimately needs agent's full capabilities? Explicit elevation:

```bash
# Standard invocation: capability intersection
$ @dangerous_agent "analyze logs"
# Agent can only use capabilities I have

# Elevated invocation: explicit capability grant
$ sudo --grant=fs.delete @dangerous_agent "clean old logs"
# Requires signed approval (Session 4, §29.2)
# Audit log: "user_alice granted fs.delete to dangerous_agent for session X"
```

The `sudo --grant` is explicit, signed, and audited. No magic escalation.

### 37.4 Updated Permission Display

```bash
$ who -l  # List agents with capabilities
AGENT             OWNER     PERMS    CAPS                    
@researcher       system    cit      [web.search, fs.read]   
@dangerous        admin     c--      [fs.*, network.*]       
@formatter        public    c--      []                      

$ who -e @dangerous  # Show effective caps for current user
AGENT             YOUR_PERMS   YOUR_CAPS        EFFECTIVE_CAPS
@dangerous        c--          [fs.read]        [fs.read]  # Intersection
```

---

## 38. Context Pipe Protocol Details

### 38.1 `|=` Semantics Formalized

**Definition:** `A |= B` means "B's context is initialized with A's final state."

**NOT a stream.** Unlike `|`, no bytes flow. It's a context initialization.

```
Timeline:
  
  t0: spawn(A)
  t1: A.run("task")
  t2: A.complete() → A.ctx frozen
  t3: spawn(B, parent_ctx=A.ctx)  ← This is what |= does
  t4: B.run("follow-up")
  t5: B.complete()
```

### 38.2 Protocol: Local Case

When A and B are on the same machine:

```python
# A completes
a_ctx = a_session.finalize()  # Returns ContextHandle

# B spawns with A's context
b_session = spawn(
    agent="@planner",
    context_init=ContextInit(
        parent=a_ctx,           # Pointer to A's KV-cache
        mode="copy_on_write",   # Share until divergence
    )
)
```

**Observable:**
```bash
$ cat /proc/b_session/ctx/init
parent: /proc/a_session/ctx/final
mode: copy_on_write
shared_blocks: 1847
diverged_blocks: 0  # Increases as B generates
```

### 38.3 Protocol: Remote Case

When A is local and B is remote:

```
Local Machine                     Remote Machine
─────────────────                 ─────────────────
A.ctx finalized                   
    │                             
    ├─── ACP HANDSHAKE ──────────► B spawning
    │    (capabilities, model)    
    │                             
    ├─── MERKLE ROOT ────────────► 
    │    (hash of A.ctx blocks)    
    │                             
    │    ◄─── BLOCK REQUEST ─────┤ "Need blocks [47, 52, 89]"
    │                             
    ├─── BLOCK DATA ─────────────►
    │    (only requested blocks)  
    │                             
    │                              B.ctx initialized
    │                              B.run("follow-up")
```

**Bandwidth:** O(novel blocks), not O(total context).

**Observable:**
```bash
$ acp-stats /net/remote/planner
TRANSFER       BLOCKS    BYTES      
shared         1847      0          # Already present
transferred    142       284KB      # Delta
total          1989      ~4MB equivalent
compression:   93%
```

### 38.4 Error Cases

```bash
# Context too large for remote
$ @researcher "massive study" |= /net/constrained-device/@planner "summarize"
# Error: Context transfer would exceed remote limit (1989 blocks > 500 block limit)
# Suggestion: Compress with `ctx_compress --target=500` before piping

# Model mismatch
$ @opus_agent "deep analysis" |= @haiku_agent "format output"
# Warning: Context originated from opus, target is haiku
# KV-cache incompatible; falling back to text serialization
# Performance: degraded (500ms vs 10ms)
```

---

## 39. Remaining Open Questions

### 39.1 Context Versioning

When A's context is shared with B, and A continues running, what happens?

Options:
1. **Snapshot:** B gets A's state at bind time. A can continue independently.
2. **Live link:** B sees A's updates (complex, potential for incoherence).
3. **Explicit modes:** User chooses.

Current recommendation: **Snapshot** (Option 1). Simpler, matches fork() semantics.

### 39.2 Multi-Parent Context

Can B inherit from multiple contexts?

```bash
@researcher "study fusion" |= @economist "study costs" |= @planner "synthesize"
```

Is this (researcher |= economist) |= planner, or researcher |= (economist |= planner)?

Proposal: Left-associative, sequential. Planner inherits from economist, who inherited from researcher. The chain is visible:

```bash
$ cat /proc/planner/ctx/lineage
researcher.ctx → economist.ctx → planner.ctx
```

### 39.3 Capability Revocation Mid-Session

If caller's token expires mid-session, what happens to running agent?

Options:
1. **Immediate termination:** SIGKILL
2. **Graceful wind-down:** SIGTERM + grace period
3. **Capability reduction:** Remove revoked caps, continue with remainder

Current recommendation: **Option 2** for cost reasons (avoid wasting partial work).

### 39.4 `sgrep` Cost Control

Semantic grep on a large corpus could be expensive. How to bound?

```bash
# Cost ceiling
sgrep --mode=llm --max-cost=0.50 "optimistic tone" corpus/
# Stops when $0.50 spent, returns partial results

# Document limit
sgrep --mode=llm --max-docs=100 "optimistic tone" corpus/
# Processes at most 100 documents

# Hybrid with early termination
sgrep --mode=hybrid --confidence=0.9 --early-stop "optimistic tone" corpus/
# Stops when confidence threshold met
```

---

## 40. Revised Roadmap

**Phase 1 (Kernel) — COMPLETE (Session 3)**
- Process/Budget/Stderr wrapper

**Phase 2 (Shell) — UPDATED**
- `ash` with `@` addressing
- `|` byte pipe (standard)
- `|=` context pipe (bind semantics, §38)
- Remove `|~` from spec

**Phase 3 (Coreutils) — NEW**
- `sgrep` with explicit modes
- `sdiff` with explicit algorithms
- `acron` decomposition
- `meta` interview protocol

**Phase 4 (Observability) — NEW**
- Context compression signals
- Compression policy engine
- `/proc/*/ctx/` filesystem

**Phase 5 (Security) — UPDATED**
- `cit` permissions
- Capability intersection on invoke
- `sudo --grant` for elevation

**Phase 6 (Network)**
- ACP protocol implementation
- Remote context mounting
- Merkle-based deduplication

---

## 41. Design Principles Codified

From five sessions of iteration, the core principles:

1. **Pipes are dumb.** Transformation is a process, not a pipe property.

2. **Explicit over implicit.** If an LLM is invoked, the user specified `--mode=llm`.

3. **Observable by default.** Every operation has stderr, every state has /proc.

4. **Capabilities intersect down.** Caller cannot escalate through agent.

5. **Compression is an event, not a daemon.** Agent knows when memory is lossy.

6. **Cost is a first-class resource.** Budget limits are as fundamental as memory limits.

7. **Pointers over copies.** Context sharing via KV-cache, not serialization.

8. **Fail explicit.** Errors identify which component failed and why.

---

*Session 5 completed: 2025-01-04*
*Focus: No-magic audit, pipe redesign, capability intersection, observable compression*
*Status: Architecture internally consistent. Ready for implementation specification.*

---

> Session 7: Metaphor Integrity & Architectural Corrections

**Focus:** Stress-test the Unix metaphor, fix where it leaks, add mechanisms we actually need, clarify what we're building.

---

## 53. The Sharper Thesis

Session 1 proposed: "Everything is a file" generalizes to "everything is an interlocutor."

External review sharpened this:

> **The correct mapping is not "LLMs are files"; it's "LLMs are processes whose I/O can be made file-like."**

This is more precise. Unix's power isn't "files everywhere"—it's:
1. Small syscall surface
2. Explicit composition
3. Observable state
4. Permissions/capabilities
5. Failure as first-class control path

We're faithful to these when we build process management, dumb pipes, observable `/proc`, and capability enforcement. We leak when we pretend cognition *is* file I/O.

**Updated thesis:**

> Agent OS treats LLMs as processes. Their I/O is file-like (stdin/stdout/stderr streams). Their memory is managed (context as address space). Their permissions are enforced (capabilities). Their lifecycle is controlled (signals). But their internal cognition is opaque—the OS routes tokens, it doesn't interpret them.

---

## 54. Metaphor Leak Fixes

### 54.1 The `|=` Operator: Bind, Not Pipe

**Problem:** Calling `|=` a "pipe" implies streaming semantics. Users expect B to start before A finishes, backpressure, incremental consumption. None of these apply—`|=` requires A to finalize before B starts.

**Fix:** Rename in all documentation.

| Old Term | New Term |
|----------|----------|
| Context pipe | **Context bind operator** |
| "Pipe" | "Bind" |

```bash
# Documentation should say:
# |= binds B's initial context to A's final state (not a stream)
@researcher "study fusion" |= @writer "summarize"
```

**Future consideration:** Add explicit `--snapshot` flag even if `--live` is unimplemented, to prevent false assumptions:

```bash
@researcher "study" |=--snapshot @writer "summarize"  # explicit: waits for A
@researcher "study" |=--live @writer "summarize"      # future: streaming context
```

### 54.2 stderr: Execution Trace, Not Chain-of-Thought

**Problem:** We defined stderr as "the Chain of Thought stream." But:
1. Many deployments can't expose raw CoT (policy, safety, capability)
2. Even when exposed, CoT isn't guaranteed causal—model could emit misleading reasoning
3. If debugging *requires* CoT, we're coupled to an optional capability

**Fix:** Redefine stderr as **execution trace**—observable events the OS guarantees, independent of model introspection.

**Guaranteed trace (always available):**
```jsonl
{"type":"tool_call","tool":"web.search","args":{"query":"fusion ignition"},"ts":"..."}
{"type":"tool_result","tool":"web.search","status":"ok","tokens":1247}
{"type":"decision","action":"spawn","target":"@analyzer","reason":"need deeper analysis"}
{"type":"budget","spent_usd":0.034,"remaining_usd":0.466}
{"type":"checkpoint","id":"ctx_a3f8...","trigger":"explicit"}
{"type":"compression","before_tokens":145000,"after_tokens":52000,"method":"lru_summarize"}
{"type":"branch","parent":"ctx_a3f8...","child":"ctx_b7c2..."}
```

**Optional rationale stream (if model supports and policy allows):**
```jsonl
{"type":"rationale","text":"I should search for recent ignition results..."}
```

**The OS must not depend on rationale.** Debugging, provenance, and replay work from the guaranteed trace alone.

### 54.3 Sanitize Introduces Semantics (Acknowledge It)

**Problem:** We claimed "the kernel never interprets text." But `sanitize` + Session Coloring makes trust decisions based on content origin and human judgment of meaning.

**Resolution:** This isn't a violation—it's necessary. Unix had setuid binaries and MAC systems. Security requires meaning at some layer.

**Clarification:** Ring 0 (kernel) still doesn't interpret text. Sanitization happens in Ring 1 (userland tool). The kernel enforces color labels; it doesn't decide what content deserves which color.

```
Ring 0: Enforces "RED process cannot call fs.write" — no text interpretation
Ring 1: sanitize tool presents content to human, human decides, tool sets color — meaning-aware
```

This is the same separation Unix has: kernel enforces DAC/MAC bits, userland tools (chmod, chown, SELinux utilities) set them based on policy/human decision.

### 54.4 Remote Mounts Need Cost Disclosure

**Problem:** Plan 9-style mounts are elegant, but `read()` on `/net/remote/@agent` could trigger expensive computation. The "just a file" mental model hides this.

**Fix:** Enforce explicit cost/latency bounds on mount, return accounting headers on operations.

```bash
# Mount with declared limits
mount -t acp //cluster/@researcher /net/researcher \
    --max-latency-ms=5000 \
    --max-cost-usd=1.00

# Operations that would exceed limits fail with clear error
$ cat /net/researcher/query
Error: Operation would exceed mount cost limit ($1.00). 
Estimated: $1.45. Use --allow-overrun or remount with higher limit.
```

**Accounting headers:** Every read/write returns metadata (like HTTP headers) that the shell can display or suppress:

```bash
$ cat /net/researcher/query
[ACP] latency=2340ms cost=$0.23 tokens=4521
... actual content ...

# Or suppress with flag
$ cat --quiet /net/researcher/query
... actual content only ...
```

---

## 55. Two Products, One Architecture

**Problem identified:** With KV-fork available, `|=` is O(1) pointer sharing—genuinely new runtime semantics. Without KV-fork, it's O(n) text serialization—good orchestration, but fundamentally different performance characteristics.

Mixing these creates confusion and brittle expectations.

**Resolution:** Explicitly frame as two layers:

### 55.1 Agent Unix (Userland)

Works everywhere—cloud APIs, local inference, any backend.

**Capabilities:**
- spawn/wait/kill, signals, exit codes
- Byte pipes (`|`) via stdout/stdin
- Capability filtering and enforcement
- Budget and velocity accounting (SIGBRAKE/SIGXCPU)
- Session Coloring (RED/BLUE/YELLOW)
- `/proc`-style introspection
- Execution trace (stderr)
- Sanitization workflow

**Context bind (`|=`) behavior:** Falls back to text serialization. Semantics preserved (B inherits A's context), performance degraded.

```bash
$ @researcher "study" |= @writer "summarize"
[WARN] KV-cache unavailable; context bind via text serialization (slower, ~2s overhead)
```

### 55.2 Kernel Accelerators

Optional optimizations enabled on compatible backends (vLLM, SGLang, llama.cpp).

**Capabilities:**
- O(1) context bind via KV-cache fork
- `.sc` shared context prefixes (precomputed KV-cache)
- Fast checkpoint/restore
- Efficient Tree-of-Thoughts branching

**Detection:** The runtime probes backend capabilities at startup:

```bash
$ aos --backend=vllm://localhost:8000 --probe
Backend: vLLM 0.4.1
KV-cache fork: supported
Shared prefixes: supported
Accelerators: ENABLED

$ aos --backend=openai://api.openai.com --probe
Backend: OpenAI API
KV-cache fork: not available
Shared prefixes: not available (use prompt caching for approximation)
Accelerators: DISABLED (userland mode)
```

---

## 56. Idempotency and Determinism

**Problem:** Unix scripting assumes idempotency—rerunning a command has predictable effects. Agents are stochastic. Same prompt can produce different outputs, different tool calls, different side effects.

### 56.1 Idempotency Keys for Tool Calls

Side-effecting tools must support idempotency keys:

```python
# Tool call with idempotency key
net.post(
    url="https://api.example.com/create",
    body=data,
    idempotency_key=hash(ctx_id, intent, args)  # Deterministic from context
)
```

If the same idempotency key is seen twice, the tool returns the cached result instead of re-executing. Prevents duplicate side effects on retry/replay.

### 56.2 Dry-Run Planning

Agents can emit execution plans for review before acting:

```bash
# Generate plan without executing
$ spawn @coder --plan "refactor the auth module"
[PLAN] @coder proposes:
  1. fs.read: src/auth/*.py (estimated: 3 files)
  2. spawn: @analyzer "identify coupling issues"
  3. fs.write: src/auth/session.py (modified)
  4. fs.write: src/auth/tokens.py (modified)
  5. shell.exec: pytest tests/auth/

Estimated cost: $0.45
Estimated time: 12s

Execute? [y/n/edit]
```

### 56.3 Determinism Knobs

For reproducibility when needed:

```bash
# Seed for reproducible sampling (if backend supports)
$ spawn --seed=42 @agent "generate options"

# Force greedy decoding
$ spawn --temperature=0 @agent "extract facts"

# Capture full run manifest for replay
$ spawn --manifest=run_001.json @agent "complex task"
```

**Run manifest** contains: prompt, context hash, tool results, random seeds, timestamps. Enables replay (rerun with same tool results) even if not bitwise identical.

---

## 57. Provenance as First-Class Primitive

**Problem:** stderr-as-CoT tells you what the model *said* it was thinking. It doesn't prove causality or provide audit trail.

**Solution:** Add provenance filesystem. More valuable than CoT and actually auditable.

### 57.1 Provenance Filesystem

```
/proc/$pid/provenance/
├── inputs/
│   ├── prompt.txt              # Original prompt (hash-addressed)
│   ├── context_parent.ref      # Parent context reference
│   └── retrieved/              # Documents retrieved during execution
│       ├── doc_001.json        # {source, hash, retrieval_query, timestamp}
│       └── doc_002.json
├── tools/
│   ├── call_001.json           # {tool, args, timestamp, result_hash}
│   ├── call_002.json
│   └── ...
├── artifacts/
│   ├── output.txt              # Final stdout (hash-addressed)
│   └── files/                  # Any files created
│       └── solution.py.ref     # Reference to created file
├── decisions.jsonl             # Why each side effect was authorized
└── manifest.json               # Complete run metadata
```

### 57.2 Decisions Log

Every authorization decision is logged:

```jsonl
{"ts":"...","action":"fs.write","path":"/tmp/draft.py","authorizer":"session_color","decision":"allowed","reason":"process is BLUE"}
{"ts":"...","action":"fs.write","path":"/etc/passwd","authorizer":"capability","decision":"denied","reason":"path not in allowed_paths"}
{"ts":"...","action":"net.post","url":"https://api.x.com","authorizer":"human","decision":"allowed","approver":"alice@ed25519:...","token":"..."}
```

### 57.3 Replay

Provenance enables replay—rerun with recorded tool results to verify consistency:

```bash
$ aos-replay --manifest=/proc/12345/provenance/manifest.json --verify
Replaying run 12345...
Tool call 1: web.search("fusion ignition") → using recorded result
Tool call 2: fs.read("data.csv") → using recorded result
...
Output comparison: MATCH (hash a3f8c2...)
Replay: CONSISTENT
```

This doesn't prove internal causality, but it proves operational reproducibility—the output follows deterministically from the inputs and tool results.

---

## 58. Transactional Side Effects

**Problem:** Side effects are scattered across tool calls with ad-hoc approval. No systematic model.

**Solution:** Two-phase commit for all side effects.

### 58.1 The Pattern

```
Phase 1: INTENT
  Agent emits intent to perform side effect
  Kernel/shell queues intent, does not execute

Phase 2: APPROVAL  
  Policy engine evaluates (capability check, budget check, color check)
  If policy requires, human approval requested
  Approved intents receive signed capability token

Phase 3: EXECUTION
  Tool executes with capability token
  Execution logged to provenance
  Token consumed (single-use)
```

### 58.2 Implementation

```python
# Agent emits intent (Phase 1)
intent = SideEffectIntent(
    action="fs.write",
    args={"path": "/tmp/output.py", "content": code},
    justification="Writing generated code to file"
)

# Kernel evaluates (Phase 2)
approval = kernel.evaluate_intent(intent, process)
# Returns: Approved(token=...) | Denied(reason=...) | NeedsHuman(prompt=...)

if approval.needs_human:
    # Shell prompts user
    decision = human_review(intent)
    if decision.approved:
        approval = kernel.grant_token(intent, signer=decision.signer)

# Tool executes with token (Phase 3)
if approval.approved:
    result = tools.fs_write(
        path=intent.args["path"],
        content=intent.args["content"],
        capability_token=approval.token  # Single-use, signed
    )
```

### 58.3 Batch Approval

For workflows with many side effects, batch review:

```bash
$ spawn @refactorer "modernize codebase" --batch-approve
[INTENT BATCH] @refactorer proposes 12 side effects:
  1. fs.write: src/auth.py (modify)
  2. fs.write: src/db.py (modify)
  3. fs.delete: src/legacy.py
  4. fs.write: tests/test_auth.py (create)
  ...

Review: [a]pprove all / [r]eject all / [i]nteractive / [v]iew details
> i

[1/12] fs.write src/auth.py
  Diff: +47 -23 lines
  [a]pprove / [r]eject / [v]iew / [s]kip
> v
... shows diff ...
> a

[2/12] fs.write src/db.py
...
```

---

## 59. Capability Leases

**Problem:** Capability revocation mid-session is awkward. Current approach: SIGTERM + respawn. Loses partial work.

**Solution:** Short-lived capability leases.

### 59.1 Lease Model

```python
class CapabilityLease:
    capability: str          # e.g., "fs.write"
    scope: str               # e.g., "/tmp/**"
    granted_at: datetime
    expires_at: datetime     # Short-lived: 30s-5min typical
    token: SignedToken
    
    def is_valid(self) -> bool:
        return datetime.now() < self.expires_at and not self.revoked
```

### 59.2 Enforcement

Every sensitive tool call requires a valid lease:

```python
def fs_write(path, content, process):
    lease = process.get_lease("fs.write", path)
    if not lease or not lease.is_valid():
        raise CapabilityError("No valid lease for fs.write on {path}")
    
    # Lease valid, proceed
    result = do_write(path, content)
    
    # Log to provenance
    log_decision(action="fs.write", path=path, lease=lease.token)
    return result
```

### 59.3 Revocation

Revoking a capability invalidates future leases without killing the process:

```bash
# Revoke fs.write for process 1234
$ sys revoke 1234 fs.write
[REVOKE] Capability fs.write revoked for PID 1234
[SIGNAL] Sent SIGCAPDROP to PID 1234
```

The process receives SIGCAPDROP and can handle gracefully:

```python
def handle_sigcapdrop(signum, frame):
    # Capability was revoked
    # Options: degrade gracefully, checkpoint, request re-grant, exit
    log("Capability revoked, switching to read-only mode")
    global write_enabled
    write_enabled = False
```

### 59.4 New Signal: SIGCAPDROP

| Signal | Trigger | Default Handler |
|--------|---------|-----------------|
| `SIGCAPDROP` | Capability lease revoked or expired | Log warning, continue with reduced capabilities |

---

## 60. Rate Limiting and Net Namespace

**Problem:** We have budget (total spend) and velocity (spend rate) but not concurrency/rate limits for API providers. Naive fan-out (Tree-of-Thoughts at scale) will hit provider QPS limits.

### 60.1 Net Namespace

Provider rate limits as first-class resources:

```
/proc/$pid/limits/
├── budget_usd          # Total spend limit
├── velocity_usd_min    # Spend rate limit
├── tokens_total        # Total token limit
└── net/
    ├── openai/
    │   ├── qps         # Queries per second limit
    │   ├── tpm         # Tokens per minute limit
    │   └── current     # Current usage
    └── anthropic/
        ├── qps
        ├── tpm
        └── current
```

### 60.2 Global Rate Scheduler

Token bucket scheduler per provider:

```yaml
# /etc/aos/providers.conf
providers:
  openai:
    qps_limit: 50
    tpm_limit: 100000
    retry_after_429: exponential_backoff
    
  anthropic:
    qps_limit: 40
    tpm_limit: 80000
    retry_after_429: exponential_backoff
    
  local_vllm:
    qps_limit: null  # unlimited
    tpm_limit: null
```

### 60.3 Behavior

When rate limit would be exceeded:

```bash
$ spawn @agent1 "task" & spawn @agent2 "task" & spawn @agent3 "task" &
[SCHED] agent3 queued: anthropic QPS limit (40/s) reached
[SCHED] agent3 started after 1.2s delay
```

Processes queue rather than fail, unless `--no-wait` flag:

```bash
$ spawn --no-wait @agent "task"
Error: Would exceed anthropic QPS limit. Use --wait to queue.
```

---

## 61. JSONL Event Schema (Pipe Convention)

**Problem:** Byte pipes are dumb (good), but agents often need richer contracts: citations, provenance, confidence, structured data. Ad-hoc formats lead to brittle parsing.

**Solution:** Standardize JSONL as the "assembly language" for agent composition. Not enforced by kernel—just a convention.

### 61.1 Event Types

```jsonl
{"v":1,"type":"text","content":"The fusion experiment achieved..."}
{"v":1,"type":"text","content":"...ignition on December 5, 2022.","final":true}
{"v":1,"type":"citation","claim":"achieved ignition","source":"https://...","confidence":0.95}
{"v":1,"type":"tool_call","tool":"web.search","args":{"q":"NIF ignition"}}
{"v":1,"type":"tool_result","tool":"web.search","status":"ok","content":"..."}
{"v":1,"type":"artifact","name":"analysis.py","hash":"a3f8...","path":"/tmp/analysis.py"}
{"v":1,"type":"confidence","overall":0.87,"factors":{"source_quality":0.9,"reasoning_depth":0.8}}
{"v":1,"type":"error","code":"BUDGET_EXHAUSTED","message":"Token limit reached"}
```

### 61.2 Usage

Agents that understand JSONL can extract structured data:

```bash
# Extract just the text
@researcher "study fusion" | jq -r 'select(.type=="text") | .content'

# Extract citations
@researcher "study fusion" | jq 'select(.type=="citation")'

# Get final confidence
@researcher "study fusion" | jq 'select(.type=="confidence") | .overall'
```

Agents that don't understand JSONL still work—they just see the raw stream.

### 61.3 Negotiation (Optional)

Agents can declare format support:

```bash
$ @researcher --schema | jq '.output_formats'
["text/plain", "application/jsonl+aos"]

# Request specific format
$ @researcher --output-format=jsonl "study fusion"
```

---

## 62. Explicit Context Management Tools

**Problem:** We deferred demand paging as a research problem. But explicit, user-triggered context management is tractable and useful.

### 62.1 ctx_pageout

Explicitly page out old context to storage:

```bash
# Page out context older than 1 hour to vector store
$ ctx_pageout --pid=1234 --selector="age>1h" --to=vectorstore://project_x

[PAGEOUT] Paged out 47 blocks (89,000 tokens) from PID 1234
[PAGEOUT] Context replaced with retrieval stubs
```

The context now contains stubs:

```
[PAGED_OUT id=block_047 retriever=ctx_pagein store=vectorstore://project_x]
```

### 62.2 ctx_pagein

Retrieve paged-out content:

```bash
# Retrieve specific content
$ ctx_pagein --pid=1234 --query="fusion ignition timeline"

[PAGEIN] Retrieved 3 blocks (12,000 tokens) matching query
[PAGEIN] Inserted into context at position 47
```

### 62.3 Integration with SIGCTXPRESSURE

```bash
#!/bin/ash
# Context pressure handler

handle_ctx_pressure() {
    # Page out old content instead of summarizing
    ctx_pageout --pid=$$ --selector="age>30m" --to=vectorstore://scratch
}

trap handle_ctx_pressure SIGCTXPRESSURE
```

### 62.4 ctx_merge

Merge multiple contexts (not at KV level—as explicit operation):

```bash
# Concatenate with deduplication
$ ctx_merge --mode=concat --dedupe ctx_a ctx_b > ctx_merged

# Extract beliefs/facts and reconcile
$ ctx_merge --mode=extract-beliefs ctx_a ctx_b > ctx_merged
[MERGE] Extracted 34 beliefs from ctx_a, 41 from ctx_b
[MERGE] Found 7 conflicts, resolved by recency
[MERGE] Output: 68 unique beliefs

# Use judge to reconcile conflicts
$ ctx_merge --mode=debate ctx_a ctx_b --judge=@arbiter > ctx_merged
```

---

## 63. Security Label Refinement

**Problem:** RED/BLUE is too coarse. Real workflows have gradations.

### 63.1 Three-Level Model

| Level | Name | Can Execute | Typical Source |
|-------|------|-------------|----------------|
| BLUE | Trusted | All capabilities | System, signed human input |
| YELLOW | Partially Trusted | Read, sandboxed write, no exec | Sanitized external, known-good APIs |
| RED | Untrusted | Read only, no side effects | Raw user input, untrusted web |

### 63.2 Contamination Rules

```
BLUE + RED → RED       (any untrusted input fully contaminates)
BLUE + YELLOW → YELLOW (partial trust reduces to partial)
YELLOW + RED → RED     (untrusted contaminates partial)
```

### 63.3 Capability Mapping

```yaml
# /etc/aos/security/colors.yaml
BLUE:
  capabilities: [fs.read, fs.write, net.fetch, net.post, exec.run, spawn.agent]
  
YELLOW:
  capabilities: [fs.read, fs.write:/sandbox/**, net.fetch, spawn.agent]
  # Can write but only to sandbox, can't POST or exec
  
RED:
  capabilities: [fs.read:/public/**, spawn.agent:--color=RED]
  # Read-only public files, can only spawn other RED agents
```

### 63.4 Promotion Paths

```
RED → YELLOW: sanitize --level=yellow (lighter review)
RED → BLUE: sanitize --level=blue (full review + signature)
YELLOW → BLUE: sanitize --level=blue (must still get full review)
```

---

## 64. Debugger Honesty

**Problem:** We claimed breakpoints can stop on "intent to do X." This is model-dependent and unreliable.

**Fix:** Downgrade claims. Breakpoints are heuristics, not guarantees.

### 64.1 What Breakpoints Actually Do

Token-stream regex matching. If the buffer matches the pattern, SIGSTOP fires.

```bash
(agdb) break "delete.*important"
[WARN] Breakpoints match token patterns, not intent. 
       Model may express same intent with different words.
       Use for debugging, not security enforcement.
```

### 64.2 Reliable vs Heuristic

| Mechanism | Reliability | Use For |
|-----------|-------------|---------|
| Capability enforcement | Reliable | Security |
| Session Coloring | Reliable | Security |
| Lease validation | Reliable | Security |
| Token breakpoints | Heuristic | Debugging |
| CoT inspection | Heuristic | Debugging |

### 64.3 Security Must Not Depend on Breakpoints

```python
# WRONG: Using breakpoints for security
if thought_contains("bypass security"):
    block_action()  # Unreliable! Model can rephrase.

# RIGHT: Using capabilities for security
if not process.has_capability("fs.delete"):
    block_action()  # Reliable. Enforced at syscall boundary.
```

---

## 65. Updated Signals

| Signal | Trigger | Default Handler |
|--------|---------|-----------------|
| `SIGTERM` | `kill PID` | Graceful shutdown, emit partial result |
| `SIGKILL` | `kill -9 PID` | Immediate termination |
| `SIGSTOP` | Debugger breakpoint, `kill -STOP` | Pause generation |
| `SIGCONT` | Debugger continue, `kill -CONT` | Resume generation |
| `SIGPIPE` | Downstream closed | Stop generating, exit 141 |
| `SIGCTXPRESSURE` | Context > 80% limit | Page out or summarize |
| `SIGXCPU` | Token budget exhausted | Exit 66 |
| `SIGBRAKE` | Spend velocity exceeded | Pause for authorization |
| `SIGCAPDROP` | Capability revoked | Degrade gracefully |
| `SIGCHLD` | Child completed | Reap zombie |

---

## 66. Updated Exit Codes

| Code | Name | Meaning |
|------|------|---------|
| 0 | SUCCESS | Completed |
| 1 | FAILURE | General error |
| 2 | INVALID_INPUT | Malformed request |
| 64 | REFUSED | Capability violation |
| 65 | CTX_OVERFLOW | Context limit exceeded |
| 66 | BUDGET_EXHAUSTED | Token/cost limit |
| 67 | UPSTREAM_FAILURE | Dependency failed |
| 68 | LOW_CONFIDENCE | Declined due to uncertainty |
| 69 | CONTAMINATED | Color violation |
| 70 | LEASE_EXPIRED | Capability lease invalid |
| 71 | RATE_LIMITED | Provider QPS/TPM exceeded |
| 141 | SIGPIPE | Downstream closed |

---

## 67. What We Have After Seven Sessions

**Core model:**
- LLMs are processes whose I/O is file-like
- Ring 0 routes tokens; it doesn't interpret them
- Dumb pipes, explicit transformation
- Observable state via `/proc` and execution traces

**Two-layer architecture:**
- Agent Unix (userland): works everywhere, text serialization fallback
- Kernel Accelerators: O(1) operations on KV-fork backends

**Security:**
- Three-level coloring (BLUE/YELLOW/RED)
- Capability intersection on invoke
- Capability leases with revocation
- Signed sanitization for promotion
- Transactional side effects (two-phase commit)

**Observability:**
- Execution trace (guaranteed, not CoT-dependent)
- Provenance filesystem
- Replay from manifest
- Debugger (heuristic, not security-grade)

**Resource management:**
- Budget limits (total)
- Velocity limits (rate)
- Provider rate limits (QPS/TPM)
- Context pressure signals
- Explicit pageout/pagein

**Composition:**
- Byte pipes (`|`)
- Context bind (`|=`) — not a pipe, a bind
- JSONL event schema (convention)
- Job control, signals

**Determinism:**
- Idempotency keys
- Dry-run planning
- Seed/temperature knobs
- Run manifests for replay

---

## 68. Open Areas

Still worth exploring in future sessions:

| Area | Status | Notes |
|------|--------|-------|
| Incremental builds (make for agents) | Proposed | Hash-based caching of prior runs |
| Full label lattice (beyond 3 levels) | Deferred | Complex; 3 levels may suffice |
| Context merge semantics | Tools exist | ctx_merge strategies need testing |
| Demand paging | Deferred | ctx_pageout/pagein is explicit approximation |
| 9P-first design | Noted | Cleaner metaphor, harder implementation |
| Multi-model pipelines | Proposed | `--model` flag, scheduler implications |

---

*Session 7 completed: 2025-01-04*
*Status: Metaphor tightened, mechanisms added, claims calibrated*


---

## Appendix A: Problem Status & Open Questions (Updated Session 7)

This appendix tracks all problems identified during Sessions 1-7, their current status, and areas for continued exploration.

---

### A.1 Problems with Clear Resolutions

| Problem | Identified | Resolved | Resolution |
|---------|------------|----------|------------|
| **Init (PID 1)** | Session 2 | Session 6 | Deterministic script. Not an LLM. Spawns daemons, shell, reaps zombies. |
| **seek() semantics** | Session 1 | Session 6 | Fork from checkpoint. Immutable history. |
| **Debugging/breakpoints** | Session 2 | Session 7 | Stream interception for debugging. **Clarified as heuristic, not security mechanism.** |
| **Partial pipeline failure** | Session 1 | Session 5 | Unix semantics: SIGPIPE upstream, EOF downstream. |
| **Pipe content format** | Sessions 1-2 | Session 5 | Raw UTF-8 text. JSONL as convention (Session 7). |
| **Output/State duality** | Session 4 | Session 6 | stdout for text, ctx-last for state. |
| **`\|=` naming confusion** | Session 7 | Session 7 | Renamed to "context bind operator." Not a pipe—requires A finalization before B starts. |
| **stderr definition** | Sessions 2-6 | Session 7 | Redefined as **execution trace** (guaranteed), not CoT (optional). OS doesn't depend on model introspection. |
| **Spend velocity** | External review | Session 6 | SIGBRAKE signal at $/minute threshold. |
| **Sanitization mechanism** | Session 5 | Session 6 | Cryptographic signature required for RED→BLUE. |
| **Capability revocation** | Session 5 | Session 7 | **Capability leases.** Short-lived tokens. Revocation invalidates future leases. SIGCAPDROP signal. |
| **Side effect control** | Scattered | Session 7 | **Transactional side effects.** Two-phase commit: intent → approval → execution. |
| **Provider rate limits** | External review | Session 7 | **Net namespace.** QPS/TPM limits per provider. Global scheduler. |
| **Agent idempotency** | External review | Session 7 | **Idempotency keys** for tool calls. **Dry-run planning** (`--plan`). |
| **Reproducibility** | Session 7 | Session 7 | **Determinism knobs:** --seed, --temperature=0, run manifests. |
| **Audit trail** | Scattered | Session 7 | **Provenance filesystem.** /proc/$pid/provenance/ with inputs, tools, artifacts, decisions. |

---

### A.2 Problems with Partial Solutions

| Problem | Status | Current Approach | Limitations |
|---------|--------|------------------|-------------|
| **Prompt injection** | Mitigated | **Session Coloring** (BLUE/YELLOW/RED). Transactional side effects. Capability leases. | Bounds blast radius, doesn't prevent attack. RED agent can still exfiltrate via stdout, influence reviewers. |
| **Context overflow** | Mitigated | SIGCTXPRESSURE + explicit ctx_pageout/ctx_pagein tools. | Paging is explicit, not transparent. Agent must handle. Retrieval may miss relevant content. |
| **Backend dependency** | Documented | Two-layer architecture: Agent Unix (everywhere) + Kernel Accelerators (KV-fork backends). | Full performance requires vLLM/SGLang/llama.cpp. Cloud APIs get degraded mode. |
| **Remote mount costs** | Mitigated | **Cost disclosure on mount.** Accounting headers on operations. | User must set limits; system can't know true cost a priori. |
| **Security granularity** | Improved | **Three-level model** (BLUE/YELLOW/RED) instead of two. | Still coarser than full label lattice. May need expansion for enterprise use cases. |

---

### A.3 Open Design Questions

| Question | Context | Current Thinking | Worth Exploring |
|----------|---------|------------------|-----------------|
| **Multi-parent context** | C inherits from both A and B? | Use ctx_merge tool with explicit strategies (concat, extract-beliefs, debate). Not KV-level. | Test ctx_merge strategies empirically. Which works best for which use cases? |
| **Context compression quality** | SIGCTXPRESSURE says compress, but how well? | Compression produces summary + structured memory index. Retrieval via ctx_pagein. | Need metrics: what percentage of critical info survives compression? |
| **Model selection per stage** | Different models for different pipeline stages? | `--model` flag could override. Cheap models for routing, expensive for synthesis. | Interaction with Session Coloring? Does model tier affect trust? Scheduler complexity? |
| **Incremental builds** | make(1) for agents—cache outputs by input hash | Proposed but not specified. Would use provenance manifest hashes. | High value for iterative workflows. Worth dedicated exploration. |
| **Full label lattice** | More than 3 security levels | Deferred. BLUE/YELLOW/RED covers most cases. | Enterprise may need: PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED, etc. SELinux MLS model. |
| **Live context bind** | `\|=--live` where B starts before A finishes | Not implemented. Would require streaming context updates. | Unclear semantics. What happens if A's early output contradicts late output? |

---

### A.4 Deeper Research Questions

| Question | Why It's Hard | Current Stance | Exploration Ideas |
|----------|---------------|----------------|-------------------|
| **Instruction/data distinction** | Transformers treat all tokens uniformly. No hardware separation. | Unsolvable at model level. Mitigate at system level (coloring, capabilities, leases). | Watch for architectural innovations (segment-aware attention). |
| **Transparent demand paging** | Requires predicting attention before it happens. | Impossible with current architectures. Use explicit ctx_pageout/ctx_pagein. | Hybrid: profile common access patterns, prefetch likely-needed content. |
| **Context merge at KV level** | "Merging attention states" is undefined. Any merge is a computation. | Avoid. Use text-level ctx_merge with explicit strategies. | Research question: is there a meaningful KV merge operation? |
| **Reasoning fidelity** | CoT may not reflect actual computation. Model could emit misleading trace. | Execution trace (tool calls, decisions) is auditable. CoT is supplementary. | Mechanistic interpretability research. For now, treat CoT as "probably indicative." |
| **Economic optimization** | Flat $/token ignores quality. How to reason about value? | Simple budget caps. User decides tradeoffs externally. | Confidence-weighted pricing? Retry budgets? Agent bids on expected value? |

---

### A.5 Ideas Considered and Rejected

| Idea | Session | Why Rejected |
|------|---------|--------------|
| **Taint tracking via attention** | Sessions 2, 4 | No "instruction pointer" in transformers. Attention is holistic. Session Coloring (process-level) is correct abstraction. |
| **Semantic pipe `\|~`** | Sessions 2, 4 | Hidden transcoder violates no-magic. Can't debug with tee. Use explicit transcode. |
| **ASLR for prompts** | Session 2 | Security theater. Doesn't address root cause. |
| **Speculative reasoning branches** | Sessions 2-3 | Only economical for >80% predictable branches. Most reasoning branches are <50%. |
| **Semantic FUSE mounting** | External review | "Read triggers invisible RAG" = magic. Call sgrep explicitly. |
| **Background learning daemon** | External review | Scope expansion. Building runtime, not learning system. |
| **Breakpoints as security** | Session 7 | Token patterns are heuristic. Model can rephrase. Use capabilities for security. |
| **CoT-dependent debugging** | Session 7 | Many deployments can't expose CoT. OS must work without it. Execution trace is the primitive. |

---

### A.6 Mechanisms Added by Session

| Session | Mechanisms Added |
|---------|------------------|
| 1-2 | Core process model, signals, exit codes, capability framework |
| 3 | Partial failure handling, pipe content resolution, taint tracking rejection |
| 4 | Ring model, cognitive scripts as shell patterns |
| 5 | No-magic audit, context bind semantics, compression signals, 8 design principles |
| 6 | Session Coloring, sanitization with signatures, SIGBRAKE, backend requirements |
| 7 | Execution trace (replacing CoT), provenance filesystem, transactional side effects, capability leases, net namespace, JSONL convention, ctx_pageout/pagein, 3-level security, determinism knobs, idempotency keys |

---

### A.7 Summary

| Category | Count | 
|----------|-------|
| **Clear resolutions** | 16 |
| **Partial solutions** | 5 |
| **Open design questions** | 6 |
| **Research questions** | 5 |
| **Rejected ideas** | 8 |

**Architecture status:** 
- Core model stable (LLMs as processes, file-like I/O, observable state)
- Two-layer framing clarified (userland everywhere, accelerators on compatible backends)
- Security model refined (3-level coloring, leases, transactional side effects)
- Observability decoupled from model introspection (execution trace, provenance)
- Metaphor leaks identified and patched

**Ready for:** Implementation experimentation, further design sessions on specific areas (incremental builds, compression strategies, multi-model scheduling).

---

*Appendix A last updated: 1/4/2025*