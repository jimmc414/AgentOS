# Agent OS Specification v2.3 — Unsolved Questions

**Purpose:** Solicit solution proposals for validated issues from adversarial review
**Current Phase:** Collecting solutions from multiple reviewers

---

## Instructions for Solution Reviewers

We've completed adversarial review of the Agent OS spec. The issues below are **validated problems** that need solutions. For each question:

1. **Propose concrete solutions** — not just "this is hard," but "here's how to fix it"
2. **Consider trade-offs** — what does each solution cost in complexity, performance, or purity?
3. **Preserve core values** where possible:
   - Filesystem as primary interface for simple operations
   - Unix tool compatibility
   - "ls, cat, echo" should work intuitively
4. **Be specific** — show file layouts, command examples, error messages
5. **Identify dependencies** — does your solution for Q1 affect Q2?

---

## Solution Sources

| Source | Status | Date |
|--------|--------|------|
| Solution Reviewer 1 | ⏳ Pending | - |
| Solution Reviewer 2 | ⏳ Pending | - |
| Solution Reviewer 3 | ⏳ Pending | - |

---

## Adversarial Review Summary

Three adversarial reviews completed. Key themes:

| Theme | R1 | R2 | R3 | Confidence |
|-------|----|----|----|----|
| Self-modifying config | ✓ | | ✓ | **High** — must fix |
| Intent approval auth | ✓ | | ✓ | **High** — must fix |
| Anti-bankruptcy bypass | ✓ | ✓ | ✓ | **High** — must fix |
| Write semantics (O_TRUNC, partial) | ✓ | ✓ | ✓ | **High** — must fix |
| Override/context race conditions | | ✓ | ✓ | **High** — must fix |
| rm -rf on mounts | | ✓ | | **Medium** — add safeguards |
| Contamination scope | ✓ | ✓ | ✓ | **Medium** — clarify, not broken |
| Ring model clarity | ✓ | | ✓ | **Medium** — documentation |
| Blocking reads (D-state) | ✓ | ✓ | ✓ | **Medium** — impl guidance |
| Cost visibility before execution | | | ✓ | **New** — needs spec change |
| Queue depth / backpressure | | | ✓ | **New** — needs spec change |
| Testing / mocking story | | | ✓ | **New** — future work |
| Stateful directories (pagination) | | | ✓ | **New** — abstraction violation |
| Inbox idempotency | | | ✓ | **New** — needs spec change |

**Reviewer Focus Areas:**
- **R1:** Security holes, internal contradictions, architectural consistency
- **R2:** POSIX semantics clashes, catastrophic failure modes  
- **R3:** Operational reality, concrete attacks, implementation complexity, "ship CLI first"

**Strongest Cross-Reviewer Agreement:** 
1. "Fix self-modifying config + approvals first" (R1, R3)
2. "CLI is the real control plane, filesystem is convenience" (all three)
3. "Per-PID circuit breaker is bypassable" (all three)

---

# PART 1: Security Questions (Must Solve)

## Q1: Agent Config Privilege Escalation

**Problem:** Agents can modify their own `config.yaml`, escalating privileges on next invocation.

**Current Spec:** §3.1 marks `/aos/agents/*/config.yaml` as R/W

**Attack:** Compromised agent writes `mounts.write: ["/aos/mnt/s3/**"]` to own config, waits for next invocation to get expanded permissions.

**Constraints:**
- Agents need to READ their config (for self-awareness)
- Operators need to MODIFY configs (for management)
- Agents must NOT modify their own configs

**Proposed Direction:** Separate read-view from write-source. Agent sees `/aos/agents/X/config.yaml` (read-only projection), operator edits `/aos/etc/agents.d/X.yaml` (source of truth).

**Open Questions:**
- How does hot-reload work? Does agent see config changes mid-session?
- Where do agent-writable preferences (non-security-relevant) live?
- How do we handle agent templates / inheritance?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q2: Intent Approval Authorization

**Problem:** No specification of WHO can approve intents. Agents can approve their own intents.

**Current Spec:** §12.7 shows `echo "approve" > intents/pending/001.json`

**Attack:** Agent generates delete intent, immediately writes approval, delete executes without human review.

**Constraints:**
- Approvals must be attributable (who approved?)
- Agents must NOT approve their own intents
- Should work with filesystem paradigm where possible
- Must handle: human approval, automated policy approval, cross-agent approval

**Proposed Direction:** Approvals require different UID than intent creator, or cryptographic signature, or CLI-only path.

**Open Questions:**
- Is UID separation sufficient, or do we need signatures?
- How do automated approval policies work? (e.g., "auto-approve reads under $0.10")
- What's the state machine? `pending` → `approved` → `executed`? Can intents be rejected?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q3: Destructive Operations on Mounts

**Problem:** `rm -rf /aos/mnt/github/issues/*` maps to mass API DELETEs. Irreversible.

**Current Spec:** §9.3 defines `rm file` as "Close/delete" (API DELETE)

**Context:** This is how all mount systems work (NFS, SSHFS), but the consequences are more severe with API-backed resources.

**Constraints:**
- Don't break legitimate write workflows
- Don't add friction to every operation
- Make the dangerous cases hard, not the common cases

**Options on Table:**
- A: Mounts default read-only, require explicit `writable: true` in mount config
- B: Destructive ops (rm, truncate) require confirmation or `--force` via CLI
- C: Soft-delete: `rm` moves to `.trash/`, requires `aos mount gc` to actually delete
- D: Per-mount policy: some mounts allow rm, others don't

**Open Questions:**
- Is read-only default too restrictive for common workflows?
- How does soft-delete interact with API semantics? (Some APIs don't have trash)
- Should this be mount-level config or global policy?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q4: Anti-Bankruptcy Circuit Breaker Design

**Problem:** Per-PID cost tracking is bypassable via PID churn (`find -exec`, `xargs`, forking).

**Current Spec:** §2.3 uses process blocklist and per-PID circuit breaker

**Attack Scenarios:**
1. `find /aos/agents -exec grep "pattern" {} \;` spawns new grep per file, each under threshold.
2. **(R3)** Backup agent runs nightly tar/rsync on /aos. Each process stays under $0.10/min by parallelizing, but total spend is thousands/hour.
3. **(R3)** Endpoint security agent, log shipper, or EDR scans /aos as part of host monitoring.

**Additional Concern (R3):** Blocklist by comm/pid is fragile:
- Indexers spawn helpers with unpredictable names
- Containers/shells/wrappers change comm
- Legitimate tools (python, node) could scan; can't block without breaking everything

**Constraints:**
- Must catch adversarial and accidental cost explosions
- Can't break legitimate batch operations
- Need multiple layers (defense in depth)
- **(R3)** Must handle parallelization across processes

**Proposed Direction:** Circuit breakers at multiple levels:
- per-PID (catches simple cases)
- per-UID (catches PID churn)
- per-session
- per-subtree (e.g., /aos/mnt/github has its own limit)
- global (absolute backstop)

**R3 Addition:** "Hard require explicit unlock" mechanism:
- Costly trees default to EACCES
- Human must flip `/aos/system/unlock_costly_reads` for N minutes
- Time-limited unlock tokens

**Open Questions:**
- What are sensible defaults for each level?
- How do legitimate batch jobs get elevated limits? (unlock tokens? sudo-equivalent?)
- How do we distinguish "100 small reads" from "1 big read" in cost accounting?
- Should there be a "dry-run" mode that shows what WOULD be charged?
- **(R3)** How do we handle host-level agents (EDR, backup) that we don't control?

### Solution Proposals
*(Space for reviewer solutions)*

---

# PART 2: Semantic Questions (Must Solve)

## Q5: Action File Write Semantics

**Problem:** POSIX write behavior (partial writes, multiple writes per handle, O_TRUNC) clashes with queue semantics.

**Current Spec:** §2.2 defines file semantic classes but doesn't address POSIX write behavior

**Specific Issues:**
- Shell `>` sends O_TRUNC, but inbox is a queue (should append)
- Editors do open+truncate+write+fsync (would corrupt message)
- Large writes might be split by kernel
- **(R3)** Programs may do multiple writes, short writes, write with offsets, or retry on EINTR

**Additional Issue (R3):** Binary input requires alternate interfaces:
- §3.8 introduces `inbox.bin` and `inbox.multipart`
- Now you can't write a generic agent-invoking script without knowing if agent accepts binary
- The uniform interface fractures

**Constraints:**
- `echo "query" > inbox` must work (core UX)
- Must be predictable and documented
- Should fail loudly on misuse, not silently corrupt
- Binary input must be possible for image/audio agents

**R3 Proposal (concrete):**
> Treat "action" files as write-once message FIFOs with explicit framing. Reject anything except a single atomic write under PIPE_BUF size. Enforce in FUSE: if write() is called more than once per open handle, fail (EINVAL) and require clients to use `aos invoke` / `aos ctl` for non-trivial payloads.

**Proposed Direction:** Action files are write-once per open:
- First write is the message
- Second write returns EINVAL
- O_TRUNC is ignored
- Payload must be < PIPE_BUF (4096) for atomicity guarantee

**Open Questions:**
- What about messages > 4096 bytes? (Reference a file? Use CLI?)
- How do you clear a queue? (Separate `inbox.ctl` file? CLI command?)
- Should `>>` and `>` behave identically for queues?
- **(R3)** What error message/errno for "partial command" or "multiple writes"?
- **(R3)** How do we handle binary without fragmenting the interface? (Single inbox with content-type detection? Explicit mode flag?)

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q6: Override and Context Race Conditions

**Problem:** Writing to `override/model` then `inbox` are two operations. Process A writes override, Process B writes inbox, B gets A's override.

**Current Spec:** §3.4 shows sequential writes to override then inbox

**Related Issue (R3):** Same race exists for context binding:
```bash
echo "/path/to/context" > /aos/agents/researcher/context
echo "Continue analysis" > /aos/agents/researcher/inbox
```
Two syscalls. If another process writes between them, wrong context gets used.

**Constraints:**
- Need atomic "invoke with these settings" operation
- Should work with filesystem paradigm if possible
- Must handle concurrent invocations correctly

**Options on Table:**
- A: Per-session override/context dirs (`/aos/agents/X/sessions/<sid>/override/`)
- B: Atomic JSON write: `{"override": {"model": "opus"}, "context": "/path", "query": "..."}` to inbox
- C: CLI-only overrides: `aos invoke --model=opus --context=/path "query"`
- D: Per-PID state: override/context applies to "next write from this PID"

**R3's Observation:** §3.5 already shows `aos invoke --context=...` as the solution, which "admits the file interface is unsafe."

**Open Questions:**
- Is per-session the right granularity? What defines a session?
- Does option B break the "plain text in, plain text out" simplicity?
- Is option C acceptable? (Other things already require CLI)
- If we accept CLI is safer, should it be the recommended path?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7: Exit Code Semantics

**Problem:** §4.4 triggers core dump on exit >= 64, but 66 (BUDGET_EXHAUSTED) is routine, not exceptional.

**Current Spec:** Exit codes 64-71 defined, core dump on >= 64

**Additional Issue (R3):** Exit 68 is overloaded:
- §4.4: "Declined due to uncertainty"
- §3.2: "Exit 68 if below confidence_threshold"
- These are different: "refused to answer" vs "answered but uncertain"

**Proposed Fix:** Core dumps only for actual errors (64 REFUSED, 65 CTX_OVERFLOW, 67-71), not resource limits (66 BUDGET_EXHAUSTED).

**Open Questions:**
- Should we reorganize exit codes? (0-63 normal, 64-127 errors, resource limits somewhere specific?)
- What other "routine" exits might we add later that shouldn't dump?
- Should exit 68 be split into two codes?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7a: Inbox Write Idempotency (NEW - R3)

**Problem:** If a retry writes the same content twice, do you get two invocations?

**Current Spec:** §20.1 defines idempotency for tool calls, but not for inbox writes.

**Scenario:**
```bash
echo "analyze this" > inbox  # Network glitch, client unsure if delivered
echo "analyze this" > inbox  # Retry — is this 1 invocation or 2?
```

**Constraints:**
- Retries are common in distributed systems
- Users expect "at least once" or "exactly once" semantics to be defined
- Deduplication has performance/storage costs

**Options:**
- A: Always queue (no deduplication) — simple, but retries cause duplicates
- B: Dedupe by content hash within time window — complex, but safer
- C: Require explicit message IDs: `echo "id:abc123 analyze this" > inbox`
- D: CLI provides idempotency: `aos invoke --idempotency-key=abc123`

**Open Questions:**
- What's the expected retry pattern? (User retries? Automated pipeline retries?)
- How long should deduplication window be?
- Does deduplication apply to all inbox writes or just retries?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7b: Cost Visibility Before Execution (NEW - R3)

**Problem:** Fork/bind/replay have wildly different costs depending on backend, but cost isn't surfaced until after execution.

**Current Spec:** §5.3 shows `aos fork` as cheap ("O(1) pointer sharing"), but §1.5 notes OpenAI/Anthropic fall back to text serialization.

**Incident Scenario:**
```bash
aos fork /aos/conversations/huge-context/ --prompt="variant 1"  # User expects O(1)
# Reality: 200k tokens × $0.003 = $0.60 per fork
# 20 variants exploring the space = $12, not pennies
```

**Constraints:**
- Don't add friction to cheap operations
- Users must understand cost BEFORE execution for expensive ops
- Cost depends on backend capability (not knowable from path alone)

**Options:**
- A: `aos fork --dry-run` shows estimated cost before execution
- B: Expensive operations require explicit `--confirm` or `--budget=$X`
- C: Warn at threshold: "This will cost ~$X. Proceed? [y/N]"
- D: Show cost estimate in a status file before execution starts

**Open Questions:**
- How accurate can cost estimates be? (Pricing changes, token counts are estimates)
- Should warnings be per-operation or cumulative session?
- How does this interact with automated pipelines that can't confirm interactively?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7c: Queue Depth and Backpressure (NEW - R3)

**Problem:** No way to check inbox queue state before submitting work.

**Current Spec:** §3.3 says inbox is "async, queues work" but doesn't define limits or visibility.

**Scenario:**
```bash
for i in {1..500}; do
  echo "task $i" > /aos/agents/worker/inbox
done
# What's the queue limit? What happens at capacity? Block? Error? Drop?
```

**Questions needing answers:**
- What's the maximum queue depth?
- What happens when queue is full? (EAGAIN? Block? Drop silently?)
- How do I check queue depth before submitting?
- Are there priority lanes?
- Can I cancel queued items?

**Options:**
- A: Expose queue state: `/aos/agents/X/inbox.depth`, `/aos/agents/X/inbox.limit`
- B: Return EAGAIN when full, let client retry
- C: Block writes when queue full (but this breaks async model)
- D: Unlimited queue with backpressure via cost accounting

**Open Questions:**
- Is queue management essential for v1 or can we document "unbounded queue, beware"?
- How does queue depth interact with cost limits?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7d: Testing and Mocking Story (NEW - R3)

**Problem:** No documented way to test agents without incurring API costs.

**Current Spec:** §20.2 has `--dry-run` but that only previews actions, doesn't mock model behavior.

**What operators need:**
- Mock model responses for CI
- Run agents in sandbox with fake tools
- Verify behavior without API costs
- Regression test across prompt changes

**Questions:**
- How do I run agents against a mock LLM?
- How do I provide canned tool responses?
- How do I verify agent behavior changed (or didn't) after prompt edit?
- How do I run 1000 test cases without spending $1000?

**Options:**
- A: Mock mode: `/aos/system/mock_mode` enables canned responses
- B: Record/replay: run once real, replay from recording
- C: Backend configuration: point to local mock server
- D: Out of scope for v1, document as limitation

**Open Questions:**
- Is testing a v1 requirement or future work?
- Should mock responses be per-agent or global?
- How do we handle non-determinism in test assertions?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7e: Stateful Directories from Pagination (NEW - R3)

**Problem:** `.cursor` files make `ls` return different results in different terminals.

**Current Spec:** §9.4 shows pagination via cursor file:
```bash
echo "abc123" > /aos/mnt/github/issues/.cursor
ls /aos/mnt/github/issues/  # Returns next page
```

**What breaks:**
- Two terminals listing same directory see different contents
- Directories don't have cursors in POSIX model
- Breaks fundamental expectations about filesystem behavior
- Shell completion, file managers, etc. will behave unpredictably

**Assessment:** This is a significant abstraction violation. Directories should be stateless.

**Options:**
- A: Remove cursor files, pagination only via CLI: `aos list --cursor=abc123`
- B: Per-session cursor state (not shared across terminals)
- C: Cursor in query parameter style: `ls /aos/mnt/github/issues/@cursor=abc123/`
- D: Accept the abstraction break, document heavily

**Open Questions:**
- Is pagination essential for mounts or can we limit result sets differently?
- How do other filesystem APIs handle large directories? (NFS has cookie-based readdir)

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q7f: Session vs Conversation Relationship (NEW - R3)

**Problem:** Ambiguous relationship between sessions and conversations.

**Current Spec:** 
- §3.6: Sessions (`s_047`) are multi-turn containers
- §6.1: Conversations organized by date with unique IDs

**Scenario:**
```bash
echo "new" > /aos/agents/researcher/session  # Creates s_048
echo "query 1" > /aos/agents/researcher/inbox
echo "query 2" > /aos/agents/researcher/inbox
```

Is this:
- One conversation with two turns? (session = conversation)
- Two conversation directories in one session? (session contains conversations)

**Questions:**
- What's the containment relationship?
- Where does multi-turn state actually live?
- How do conversations and sessions appear in `/aos/conversations/`?

**Open Questions:**
- Can we simplify by merging concepts?
- Or clarify with explicit hierarchy: session > conversation > turn?

### Solution Proposals
*(Space for reviewer solutions)*

---

# PART 3: Design Questions (Should Solve)

## Q8: Contamination Tracking Scope

**Problem:** §12.2 implies byte-level taint tracking ("BLUE + RED → RED"), but FUSE can't see data provenance through shell pipes.

**Clarification:** Per-agent tracking IS implementable. The question is how to document and position it.

**What We CAN Enforce:**
- Agent process color is known
- Agent's writes inherit agent's color  
- Agent's color elevates when agent reads RED content
- Cross-agent pipes via `aos pipe` are tracked

**What We CANNOT Enforce:**
- `cat red_file | cat > blue_inbox` (shell pipeline outside agent)
- User manually copying RED content

**Questions:**
- How do we document this clearly without undermining confidence?
- Should §12 include an explicit "Threat Model" section?
- Is "agents that touch RED become RED" the right framing?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q9: Filesystem vs CLI Positioning

**Problem:** Critical operations (bind, mount, pipe, replay) already require CLI, but spec leads with filesystem examples.

**Context:** Both interfaces are valid. The question is how to position them.

**Current Reality:**
- Filesystem: simple invocations, status inspection, streaming output
- CLI: complex operations, atomicity guarantees, reliable scripting

**Questions:**
- Should the spec explicitly acknowledge this dual interface?
- What's the decision tree? "Use filesystem when X, use CLI when Y"
- Should `echo > inbox` or `aos invoke` be the "hello world"?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q10: Async Invocation Feedback

**Problem:** Write to inbox succeeds, agent fails 500ms later, script continues assuming success.

**Context:** This is inherent to async, but the DX is painful.

**Current Pain:**
```bash
echo "task" > inbox
while [ "$(cat status)" != "idle" ]; do sleep 0.1; done
if [ "$(cat exit/code)" != "0" ]; then handle_error; fi
```

**Options:**
- A: `aos invoke --wait` handles polling internally
- B: Synchronous mode: write blocks until agent completes (opt-in)
- C: Provide helper scripts / better documentation
- D: Accept this is inherent to async, document patterns

**Questions:**
- Is blocking write (option B) even desirable? Breaks streaming use case.
- What should `aos invoke --wait` return? Exit code? Output?
- Should there be a `--timeout` flag?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q10a: Orphaned Conversation Cleanup (NEW - R3)

**Problem:** Rapid iteration creates many conversation directories; cleanup only runs at 3am.

**Current Spec:** §6.9 shows retention runs daily at 3am.

**Incident Scenario:**
```bash
for i in {1..50000}; do
  echo "test $i" > /aos/agents/helper/inbox
  aos wait /aos/agents/helper
done
```

That's 50,000 directories (each with transcript.md, transcript.jsonl, context.ctx, meta.json...) persisting until 3am. On fast iteration cycles, disk fills.

**Constraints:**
- Can't delete conversations that might be needed
- Must support rapid iteration in development
- Cleanup shouldn't interfere with active work

**Options:**
- A: `aos gc --now` for synchronous cleanup
- B: Shorter retention for "test" or "ephemeral" marked conversations
- C: Count-based limit: keep last N conversations per agent, prune oldest
- D: Size-based limit: prune when /aos/conversations exceeds X GB

**Open Questions:**
- How do we distinguish "test run" from "production conversation"?
- Should there be a "don't persist" mode for rapid iteration?
- What's the right default retention policy?

### Solution Proposals
*(Space for reviewer solutions)*

---

# PART 4: Clarification Questions (Nice to Solve)

## Q11: Ring 0 vs Ring 1 Policy Distinction

**Problem:** §1.4 says Ring 0 "does not interpret text", but §2.3 has daemon checking process names for anti-bankruptcy.

**Proposed Clarification:** Ring 0 enforces *structural* policy (identity, resources, rate limits). Ring 1 enforces *semantic* policy (content analysis, safety rules).

**Question:** Is this framing clear and accurate? Does it need spec text or just documentation?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q12: Blocking Read Implementation Guidance

**Problem:** If FUSE daemon stalls, blocking reads can put processes in uninterruptible sleep (D-state).

**Context:** This is an implementation concern, not a spec flaw. But we should provide guidance.

**Proposed Guidance:**
- Daemon MUST implement internal timeouts
- Blocking operations return ETIMEDOUT after configurable limit
- Document: blocking reads are intentional; daemon must be robust

**Question:** Should this be in the spec or in a separate implementation guide?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q13: Replay Semantics Terminology

**Problem:** §6.3 shows `"deterministic": true` but LLM outputs aren't reproducible.

**Clarification:** "Replay" means re-execute with recorded tool results and compare stored outputs. NOT regenerate identical LLM output.

**R3 Note:** Either:
- "replay" means "verify we can reproduce tool calls and compare stored outputs" (not regenerate), or
- you need a huge reproducibility subsystem and strict hermeticity

**Proposed Fix:** Rename field to `"replayable": true` or `"tool_deterministic": true`.

**Question:** Which name is clearest?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q14: Virtual Size Reporting (st_size)

**Source:** Reviewer 3, §1.4

**Problem:** §2.3 proposes "virtual size reporting" (st_size=0 until open) to discourage tools from reading. This breaks real programs.

**What breaks:**
- rsync delta logic (uses size for optimization)
- cp --reflink / copy optimizations
- Tools that allocate buffers based on size
- Tools that skip empty files entirely

**R3 Note:** "Many indexers don't rely on size—they just open and read." So st_size=0 doesn't prevent the attack, but DOES break legitimate tools.

**Assessment:** This is a new issue not raised by R1/R2. Need to evaluate:
- Does the current spec actually propose st_size=0?
- Is this a real implementation plan or just discussion?
- What's the alternative?

**Options:**
- A: Report accurate size, accept that tools will read
- B: Report st_size=0, accept that some tools break
- C: Report accurate size but return EACCES on open unless unlocked
- D: **(R3)** Provide separate "costly subtree" that is noexec,nodev,nosuid and not traversable

**Open Questions:**
- Is this actually in the spec or was it just considered?
- What percentage of tools actually use st_size vs just opening?
- Is there a way to discourage reads without lying about metadata?

### Solution Proposals
*(Space for reviewer solutions)*

---

## Q15: Blocking Reads — FIFOs vs Files

**Source:** Reviewer 3, §1.3 (expansion of Q12)

**Problem:** Modeling blocking reads as files (stream, approvals, queues) breaks tool expectations.

**Current Design:** `cat /aos/agents/X/stream` blocks until output arrives.

**What breaks:**
- Tools expect file reads to terminate
- `cat` on approval channel is now a synchronization primitive with no cancellation story
- `tail -f` on FUSE is notoriously subtle (uses stat, seek, inode heuristics)

**R3 Proposal:**
> Make these named pipes (FIFOs) or Unix domain sockets, and expose mirrors as files (e.g., `last_response`, `last_approval`) for "everything is readable as a file" without forcing "everything is interactable as a file".

**Trade-offs:**
- FIFOs: familiar, but single-reader semantics
- Unix sockets: flexible, but not "everything is a file"
- File mirrors: lose real-time streaming, but safe to cat

**Open Questions:**
- Is the complexity of FIFOs/sockets worth it?
- What should `last_response` contain? Full output or just final chunk?
- How does this interact with the "tail -f for streaming" use case?

### Solution Proposals
*(Space for reviewer solutions)*

---

# Appendix A: Resolved/Rejected Issues

These were raised in adversarial review but are either resolved or rejected:

| Issue | Resolution |
|-------|------------|
| SUGGESTION-001: Split local/virtual filesystem | ❌ Rejected — adds complexity, current model is clear |
| SUGGESTION-004: Filesystem for observability only | ❌ Rejected — throws out core value proposition |
| IMPL-004: KV-cache "GPU memory manager" | ❌ Overstated — we use backend APIs, not build memory management |
| IMPL-005: Query mismatch | ✅ Acknowledged limitation — never claimed query support |
| SEC-003: Secret leakage | ✅ Defense in depth — add redaction, but not a fundamental flaw |
| R3: "No multi-tenancy" | 🟡 May be v1 scope — single-tenant design is valid |
| R3: "No horizontal scaling" | 🟡 Future work — document as known limitation |
| R3: "No workflow orchestration" | 🟡 Not in scope — Agent OS isn't trying to replace Temporal |
| R3: "Why not Temporal + Redis + Postgres" | ❌ Misses point — unified abstraction has value |

---

# Appendix B: Strategic Recommendations from Reviews

## R3's Key Recommendation: "Ship CLI First, FUSE Second"

> If `aos invoke`, `aos pipe`, `aos resume` work well, the filesystem is syntactic sugar. If they don't, the filesystem won't save them.

**Interpretation:** The CLI is the real interface. FUSE is a convenience layer. Prioritize:
1. `aos invoke` — reliable invocation with proper error handling
2. `aos pipe` — context sharing between agents
3. `aos bind` — efficient context attachment
4. `aos replay` — deterministic re-execution

Once these work, FUSE becomes syntactic sugar for simple cases.

## Cross-Reviewer Strategic Advice

| Recommendation | R1 | R2 | R3 | Action |
|----------------|----|----|----|----|
| Fix self-modifying config first | ✓ | | ✓ | **Critical path** |
| Fix unsigned approvals first | ✓ | | ✓ | **Critical path** |
| Clarify what contamination catches | ✓ | ✓ | ✓ | Documentation |
| Be honest about CLI escape hatches | ✓ | ✓ | ✓ | Documentation |
| Position /aos like /proc, not /home | ✓ | | ✓ | Documentation |
| Add explicit threat model to §12 | ✓ | ✓ | | Documentation |
| Define testing story | | | ✓ | Future work |
| Define cost visibility | | | ✓ | Spec change |

## Explicit Future Work (per R3)

These should be documented as out-of-scope for v1:

1. **Context management:** "v1 uses text serialization only; KV-cache sharing is future work for compatible backends"
2. **Security colors:** "v1 tracks colors at message/agent granularity; inference-level tracking is out of scope"
3. **Multi-tenancy:** "v1 is single-tenant; cluster deployment is future work"
4. **Horizontal scaling:** "v1 targets ~100 concurrent processes per daemon"

---

# Appendix C: Original Adversarial Reviews

*(Reference only — see full reviews in project history)*

**Reviewer 1 Focus:** Security holes, internal contradictions, architectural consistency
- Identified: SEC-001, SEC-002, SEC-003, ARCH-001, ARCH-002, SEM-001, IMPL-001-003
- Key insight: Self-modifying config is "fastest path to a real breach"

**Reviewer 2 Focus:** POSIX semantics clashes, catastrophic failure modes
- Identified: SEC-004, SEM-004, SEM-005, DX-001
- Overstated: SEC-005 (contamination), IMPL-004 (KV-cache)
- Key insight: Plan 9 failed because of impedance mismatch, not bad ideas

**Reviewer 3 Focus:** Deep POSIX semantics, concrete attack scenarios, operational reality
- Reinforced: SEC-001, SEC-002, ARCH-001, ARCH-002, SEM-001, exit codes, .gitignore
- New issues: Q7a-Q7f (idempotency, cost visibility, queue depth, testing, stateful dirs, session/conversation)
- Key insight: "Ship CLI first, FUSE second"
- Strongest recommendation: "If you fix only one thing: fix self-modifying config + approvals"

**Cross-Reviewer Agreement (High Confidence Issues):**
1. Self-modifying config (all three)
2. Intent approval authorization (R1, R3)
3. Anti-bankruptcy bypass via PID churn (all three)
4. Write semantics for action files (all three)
5. Ring model needs clarification (R1, R3)
6. CLI is actually the control plane, filesystem is convenience (all three)

---

# Change Log

| Date | Change |
|------|--------|
| 2025-01-04 | Adversarial review completed (2 reviewers) |
| 2025-01-04 | Issues calibrated, 8 confirmed for spec changes |
| 2025-01-04 | Restructured as Unsolved Questions for solution phase |
| 2025-01-04 | Added Reviewer 3 feedback |
| 2025-01-04 | Added Q14 (virtual size), Q15 (FIFOs for blocking) |
| 2025-01-04 | Updated Q4 with parallelization attack and endpoint security angle |
| 2025-01-04 | Updated Q5 with PIPE_BUF proposal |
| 2025-01-04 | Added Q7a-Q7f: idempotency, cost visibility, queue depth, testing, stateful dirs, session/conversation |
| 2025-01-04 | Added Appendix B: Strategic Recommendations |
| 2025-01-04 | All three reviewers agree: fix config + approvals first, CLI is real interface |
