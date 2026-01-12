**Task: Extract canonical specification from Agent OS design sessions**

You have 7 design sessions plus an appendix. These are a design journal—ideas evolved, some were rejected, terminology changed. Your job is to extract the *final* state into clean reference documents.

**Critical rule:** Later sessions supersede earlier sessions. When in conflict, Session 7 > Session 6 > ... > Session 1. Appendix A is the authority on what was resolved vs rejected.

**Produce these documents in order:**

### Document 1: REJECTED.md

Extract every idea that was proposed then rejected. Format:

```markdown
## Rejected Ideas

| Idea | Proposed | Rejected | Reason |
|------|----------|----------|--------|
| Semantic pipe `\|~` | Session 2 §17.2 | Session 3 §21.1, Session 5 §34 | Violates no-magic principle; hidden computation |
| ... | ... | ... | ... |
```

Include: `|~`, taint tracking via attention, ASLR for prompts, speculative reasoning branches, semantic FUSE mounting, background learning daemon, CoT-dependent debugging.

### Document 2: QUICK_REFERENCE.md

Extract final versions of all tables:

- Signals (Session 7 §65 is authoritative)
- Exit codes (Session 7 §66 is authoritative)
- Operators (`|`, `|=`, `&`, `>`) with current semantics
- Security levels (BLUE/YELLOW/RED from Session 7 §63)
- Ring model (Ring 0/1/2)
- Backend compatibility matrix

No prose. Just tables with brief descriptions.

### Document 3: SPEC.md

The canonical specification. Structure:

```markdown
# Agent OS Specification v1.0

## 1. Overview
- Thesis (Session 7 §53 version: "LLMs are processes whose I/O can be made file-like")
- Design principles (Session 5 §48, all 8)
- Two-layer architecture (Agent Unix + Kernel Accelerators)

## 2. Architecture
- Ring model (0/1/2)
- Backend requirements (Session 7 §55)

## 3. Process Model
- Process Control Block structure
- Lifecycle: spawn/wait/kill
- Signals (full list)
- Exit codes (full list)

## 4. Context Management
- Merkle DAG structure
- Context bind operator (|=) — NOT called a pipe
- Checkpointing
- ctx_pageout / ctx_pagein
- ctx_merge strategies

## 5. Security
- Three-level coloring (BLUE/YELLOW/RED)
- Contamination rules
- Capability model
- Capability intersection on invoke
- Capability leases and SIGCAPDROP
- Sanitization with cryptographic signatures
- Transactional side effects (two-phase commit)

## 6. Observability  
- Execution trace (NOT Chain-of-Thought) — guaranteed events
- Optional rationale stream
- Provenance filesystem (/proc/$pid/provenance/)
- Debugging (agdb) — noted as heuristic, not security mechanism

## 7. Resource Management
- Budget limits (total spend)
- Velocity limits (SIGBRAKE)
- Provider rate limits (net namespace, QPS/TPM)

## 8. Shell Syntax
- Operators with semantics
- ctx-last builtin
- Job control

## 9. Composition Patterns
- Include final versions of: reflexion.ash, tot.ash, debate.ash
- JSONL event schema convention

## 10. Implementation Layers
- Layer 0: Inference wrapper
- Layer 1: Process model
- Layer 2: Pipes and composition
- Layer 3: Security
- Layer 4: Observability
- Layer 5: Shell
```

**Rules for SPEC.md:**
- No history ("we considered", "Session 3 proposed")
- No alternatives ("instead of X, we chose Y")
- No justifications (save for DECISIONS.md)
- Write as if this is the only document that ever existed
- Use imperative mood ("The kernel routes tokens" not "The kernel should route tokens")

### Document 4: DECISIONS.md

For major design choices, explain why:

```markdown
## Why Session Coloring Instead of Taint Tracking

**Decision:** Process-level coloring (BLUE/YELLOW/RED) rather than token-level taint tracking.

**Alternatives considered:** Taint tracking via attention patterns (Session 2 §18.1)

**Why rejected:** Transformers have no "instruction pointer." Attention is holistic across full context. Cannot identify which tokens caused an action. Session Coloring operates at process level, which is enforceable.

**Decided:** Session 3 §21.2, refined Session 7 §63
```

Cover: pipe semantics, security model, stderr definition, `|=` semantics, backend requirements, debugging limitations.

---