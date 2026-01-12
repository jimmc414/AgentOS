# Rejected Ideas

Ideas that were proposed during Agent OS design sessions but ultimately rejected. This document preserves the reasoning to prevent re-litigation.

## Summary Table

| Idea | Proposed | Rejected | Reason |
|------|----------|----------|--------|
| Taint tracking via attention | Sessions 2, 4 | Session 3, Session 5 audit | No "instruction pointer" in transformers; attention is holistic |
| Semantic pipe `\|~` | Session 2 §17.2, Session 4 §26.2 | Session 3 §21.1, Session 5 §34 | Violates no-magic principle; hidden computation in plumbing |
| ASLR for prompts | Session 2 §18.2 | Session 7 A.5 | Security theater; doesn't address root cause |
| Speculative reasoning branches | Sessions 2-3 §16.2 | Session 3 §21.3 | Only economical for >80% predictable branches; most are <50% |
| Semantic FUSE mounting | External review | Session 7 A.5 | "Read triggers invisible RAG" = magic; violates explicitness |
| Background learning daemon | External review | Session 7 A.5 | Scope expansion; building runtime, not learning system |
| Breakpoints as security mechanism | Session 7 §64 | Session 7 §64.3 | Token patterns are heuristic; model can rephrase |
| CoT-dependent debugging | Sessions 2-6 | Session 7 §54.2 | Many deployments can't expose CoT; OS must work without it |

---

## Detailed Rejections

### 1. Taint Tracking via Attention

**Proposal:** Mark tokens from untrusted sources (user input, web content) as TAINTED. When a syscall is invoked, check the attention map. If the instruction relies heavily on TAINTED tokens, block the call.

**Why it seemed promising:** Direct analogy to CPU NX bit. Data marked as non-executable can't become code.

**Why rejected:**
- Transformers have no "instruction pointer." All tokens contribute to all outputs via attention.
- Attention patterns don't cleanly separate "following instructions from X" vs "reasoning about content from X."
- Example failure: "The user asked me to delete files, but I shouldn't do that." High attention to tainted content while *correctly refusing* would trigger false positive.
- The NX bit works because CPUs have a fundamental code/data distinction. Transformers don't.

**Replacement:** Session Coloring (BLUE/YELLOW/RED) operates at process level, which is enforceable.

---

### 2. Semantic Pipe `|~`

**Proposal:** Auto-transcoding pipe. `cmd1 |~ cmd2` inspects input/output formats and transparently spawns a transcoder agent if they mismatch (e.g., JSON → CSV).

**Why it seemed promising:** Reduces boilerplate. Users don't need to know format details.

**Why rejected:**
- Violates no-magic principle: "If this fails, can I debug it with `tee`, `strace`, and `grep`?" No.
- Hidden computation in the pipe. Can't `tee` the intermediate state. Can't substitute a different transcoder.
- Silent failures: What happens when transcoding fails? Error attribution is impossible.
- Three attempts to save it all failed:
  1. Make transcoding visible via stderr → Still hidden in pipe
  2. Explicit transcoder registry `|~json:csv` → Just syntax sugar for explicit `transcode`
  3. Transcoding as pipe metadata → Still "figure it out for me"

**Replacement:** Explicit transcode process: `cmd1 | transcode --from=json --to=csv | cmd2`

---

### 3. ASLR for Prompts

**Proposal:** Randomize structural framing of system prompts on every run. Prevents attackers from overfitting jailbreaks to specific prompt syntax.

**Why it seemed promising:** Works for memory exploits; maybe works for prompt exploits.

**Why rejected:**
- Security theater. Doesn't address root cause (lack of instruction/data separation).
- Adds complexity without measurable security benefit.
- Attackers can probe for randomization patterns.
- Better mitigations exist: capability enforcement, session coloring, output validation.

**Replacement:** Defense in depth via capability intersection, session coloring, and transactional side effects.

---

### 4. Speculative Reasoning Branches

**Proposal:** `/dev/future` filesystem. Pre-generate likely reasoning branches during idle time. When main thread needs that branch, tokens are already computed (0ms latency).

**Why it seemed promising:** CPU branch prediction is hugely valuable. Why not for LLMs?

**Why rejected:**
- Economics don't work. Break-even analysis:
  - Cost of speculation = `tokens_generated × cost_per_token × P(branch_not_taken)`
  - Benefit = `P(branch_taken) × latency_saved`
- Only economical when branches are >80% predictable
- Most reasoning branches are <50% predictable
- API pricing means "idle compute" isn't free

**When it does work:**
- Pre-warming next turn while user types (>90% predictable)
- Caching common tool patterns (>80% predictable)

**Replacement:** Limited speculative execution for high-predictability scenarios only. Not a general mechanism.

---

### 5. Semantic FUSE Mounting

**Proposal:** Mount knowledge sources as filesystems. Reading a file triggers invisible RAG retrieval. `cat /knowledge/fusion` returns relevant documents.

**Why it seemed promising:** Clean abstraction. Knowledge as files.

**Why rejected:**
- Violates no-magic principle. Read operation has hidden side effects (embedding computation, vector search, LLM summarization).
- Cost unpredictable. `cat` might cost $0.001 or $5.00 depending on retrieval depth.
- Can't debug with standard tools. What did the RAG retrieve? How was it ranked?

**Replacement:** Explicit `sgrep` with mode flags. Cost and mechanism are visible.

---

### 6. Background Learning Daemon

**Proposal:** Daemon that observes agent behavior and continuously improves system prompts, few-shot examples, or fine-tuning.

**Why it seemed promising:** Systems should get better over time.

**Why rejected:**
- Scope expansion. Agent OS is a runtime, not a learning system.
- Introduces non-determinism. Same input could produce different outputs over time.
- Security implications. What if daemon learns from adversarial inputs?
- Separate concern. Learning should be explicit, versioned, auditable.

**Replacement:** Out of scope. Learning is a separate system that produces versioned artifacts (prompts, LoRAs) that the runtime loads.

---

### 7. Breakpoints as Security Mechanism

**Proposal:** Use debugger breakpoints to stop execution when agent intends harmful actions. Pattern match on token stream for dangerous patterns.

**Why it seemed promising:** Proactive prevention. Stop bad actions before they happen.

**Why rejected:**
- Token patterns are heuristic, not guarantee.
- Model can rephrase intent. "Delete files" → "Remove data" → "Clear storage" → "fs.unlink"
- Security must not depend on catching specific phrases.

**Replacement:** Capability enforcement at syscall boundary. Breakpoints are useful for debugging, not security.

```python
# WRONG: Using breakpoints for security
if thought_contains("bypass security"):
    block_action()  # Unreliable! Model can rephrase.

# RIGHT: Using capabilities for security
if not process.has_capability("fs.delete"):
    block_action()  # Reliable. Enforced at syscall boundary.
```

---

### 8. CoT-Dependent Debugging

**Proposal:** Primary debugging mechanism is inspecting Chain-of-Thought output. stderr carries reasoning traces that explain agent behavior.

**Why it seemed promising:** CoT provides insight into "why" the agent did something.

**Why rejected:**
- Many deployments can't expose raw CoT (policy, safety, capability restrictions).
- Even when exposed, CoT isn't guaranteed causal—model could emit misleading reasoning.
- If debugging *requires* CoT, we're coupled to an optional capability.
- OS must work without model introspection.

**Replacement:** Execution trace (guaranteed events: tool calls, decisions, budget, checkpoints, compression) is the primitive. CoT is optional supplementary information.

```jsonl
// Guaranteed trace (always available):
{"type":"tool_call","tool":"web.search","args":{"query":"fusion ignition"},"ts":"..."}
{"type":"decision","action":"spawn","target":"@analyzer"}
{"type":"budget","spent_usd":0.034,"remaining_usd":0.466}

// Optional rationale (if model supports and policy allows):
{"type":"rationale","text":"I should search for recent ignition results..."}
```

---

## Cross-Reference

For the accepted alternatives to these rejected ideas, see:
- SPEC.md §5 (Security) — Session Coloring, Capability Intersection
- SPEC.md §6 (Observability) — Execution Trace
- SPEC.md §8 (Shell Syntax) — Explicit operators
- DECISIONS.md — Full rationale for each design choice
