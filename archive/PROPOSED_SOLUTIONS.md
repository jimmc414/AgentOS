# Agent OS v2.3: Proposed Solutions

**Status:** Collecting and consolidating solutions from multiple reviewers
**Last Updated:** 2025-01-05

---

## Solution Reviewers

| Reviewer | Status | Focus |
|----------|--------|-------|
| S1 | ✅ Complete | Unix primitives (SID, FIFO, Maildir, UID) |
| S2 | ✅ Complete | Signatures, commit-on-close, JSON envelopes, leases |
| S3 | ✅ Complete | Policy auto-approve, subtree locks, priority queues |

## Convergence Summary

| Question | S1 | S2 | S3 | Adopted |
|----------|----|----|----| --------|
| Q1 | /etc/ → projection | + config hash | + preferences | ✅ All three |
| Q2 | UID-based dirs | + signatures | + auto-approve policy | ✅ All three |
| Q3 | Not addressed | Complete | Same | ✅ S2/S3 |
| Q4 | Session + velocity | + TTY + leases | + subtree locks | ✅ All three |
| Q5 | Write-once | Commit-on-close 1MiB | Commit-on-close 64KB | ✅ S2/S3 (64KB) |
| Q6 | Maildir spool | JSON envelope | CLI canonical | ✅ All three |
| Q7 | Basic | + code 72 | Reorg 64-95/96-127 | ⚠️ Need decision |
| Q7a | Not addressed | Idempotency keys | id: prefix | ✅ S2/S3 |
| Q7b | --dry-run | + ranges | Same | ✅ All three |
| Q7c | inbox.status | + queue/ctl | + priority queue | ✅ All three |
| Q7d | Not addressed | Record/replay | Mock backend | ✅ S2/S3 |
| Q7e | Not addressed | .more.json | CLI only | ✅ S2/S3 |
| Q7f | Not addressed | Session/turns | Same | ✅ S2/S3 |

---

# ADOPTED SOLUTIONS

## Q1: Agent Config Privilege Escalation

**Status:** ✅ ADOPTED (S1 + S2 converge)

**Pattern:** Source (Admin) vs. Projection (Runtime), mirroring `/etc/` → `/proc/`

### Design

```
/aos/etc/agents.d/              # Source of truth (admin-only)
└── researcher.yaml             # Mode 0600, owner: aos-admin

/aos/agents/researcher/         # Runtime view
├── config.yaml                 # Mode 0444, read-only FUSE projection
├── prefs.yaml                  # Mode 0644, agent-writable (non-security)
├── status
└── inbox
```

### Semantics

| File | Owner | Mode | Purpose |
|------|-------|------|---------|
| `/aos/etc/agents.d/X.yaml` | aos-admin | 0600 | Security config: capabilities, budgets, mounts |
| `/aos/agents/X/config.yaml` | root | 0444 | Read-only projection of above |
| `/aos/agents/X/prefs.yaml` | agent | 0644 | Non-security settings: style, verbosity |

### Config Hash Audit Trail (S2 addition)

Each invocation records which config version was used:

```bash
cat /aos/procs/1234/config_hash
# sha256:abc123...

cat /aos/conversations/2025/01/04/a3f8c2/manifest.json | jq '.agent.config_hash'
# "sha256:abc123..."
```

### Commands

```bash
# Agent reads config (works)
cat /aos/agents/researcher/config.yaml

# Agent tries to write (fails)
echo "evil" > /aos/agents/researcher/config.yaml
# Error: Read-only file system (EROFS)
# Exit code: 1

# Agent updates preferences (works - whitelisted keys only)
echo "verbosity: medium" > /aos/agents/researcher/prefs.yaml

# Operator updates config
sudo aos agent set researcher --set limits.max_cost_usd=5.00
# or: sudo aos agent edit researcher   # opens $EDITOR
sudo aos agent reload researcher       # optional, otherwise auto-reload
```

### Hot Reload (S2 clarification)

- Daemon watches `/aos/etc/agents.d/` via inotify
- Changes apply to **future invocations only** (never mid-session)
- Running processes keep their snapshotted config

### Error Handling

- Write to config.yaml → `EROFS` (Read-only file system), exit 1
- Invalid YAML in source → daemon logs error, projection retains last-known-good

---

## Q2: Intent Approval Authorization

**Status:** ✅ ADOPTED (S1 + S2 + S3 converge)

**Pattern:** UID separation + optional signatures + policy-based auto-approval

### Design

```
/aos/procs/1234/intents/
├── pending/                    # Agent-writable
│   └── task_01.json
├── approvals/                  # Approvers-only (not agent-writable)
│   └── task_01.approval.json
├── rejected/
│   └── task_02.rejection.json
└── completed/
    └── task_01.json
```

### Three Enforcement Levels

**Level 1: UID-based (S1 — default)**
- Agent UID can write to `pending/`
- Agent UID CANNOT write to `approvals/` → EACCES
- Approver UIDs (human operator, aos-admin) can write approvals

**Level 2: Signature-based (S2 — opt-in for high-risk)**
- Approvals include ed25519 signatures
- Daemon validates against known keys
- Required for operations marked `requires_signature: true`

**Level 3: Policy auto-approval (S3 — for low-risk)**
- Config defines auto-approval rules
- Daemon auto-approves matching operations
- Creates approval artifact with `approver_name: "policy:..."`

### Policy Configuration (S3 addition)

```yaml
# /aos/etc/security/approval_policy.yaml
policies:
  - name: auto_approve_cheap_reads
    match:
      action: ["fs.read", "web.fetch"]
      cost_estimate_usd: {max: 0.10}
      risk_level: "low"
    approval: auto

  - name: require_human_for_deletes
    match:
      action: ["fs.delete", "fs.write"]
      path: ["/important/**"]
    approval: human
    requires_signature: true

  - name: default
    approval: human
```

### Auto-Approval Artifact

```json
{
  "approved": true,
  "approver_uid": 0,
  "approver_name": "policy:auto_approve_cheap_reads",
  "reason": "Auto-approved by policy",
  "timestamp": "2025-01-05T10:31:00Z"
}
```

### Commands

```bash
# Agent submits intent
echo '{"action":"fs.read","path":"/data"}' > /aos/procs/1234/intents/pending/task_01.json
# If matches auto-approve policy → immediately approved

# Agent tries to self-approve (fails)
echo '{"approved":true}' > /aos/procs/1234/intents/pending/task_01.approval
# Error: Cannot approve own intent (EPERM)

# Human approves high-risk (CLI)
aos approve 1234/task_02 --reason="reviewed"

# Bulk auto-approve by policy
aos approve --all --proc=1234 --risk-level=low
```

### Intent State Machine (S3)

```
                    ┌─────────────────────┐
                    │                     │
    ┌───────────────▼───────────────┐     │
    │           PENDING             │     │ timeout
    │                               │     │
    └───────────────┬───────────────┘     │
                    │                     │
    ┌───────────────┼───────────────┐     │
    │               │               │     │
    ▼               ▼               ▼     │
┌────────┐   ┌───────────┐   ┌──────────┐│
│  AUTO  │   │  APPROVED │   │ REJECTED ││
│APPROVED│   │  (human)  │   │          ││
└───┬────┘   └─────┬─────┘   └──────────┘│
    │              │                     │
    └──────┬───────┘                     │
           ▼                             │
    ┌─────────────┐                      │
    │  EXECUTED   │                      │
    └──────┬──────┘                      │
           ▼                             │
    ┌─────────────┐    ┌─────────────┐   │
    │  COMPLETED  │    │   EXPIRED   │◄──┘
    └─────────────┘    └─────────────┘
```

### Error Handling

| Condition | Error | Exit Code |
|-----------|-------|-----------|
| Self-approval attempt | EPERM | 1 |
| Unauthorized UID | EACCES | 1 |
| Invalid signature | Intent stays pending | - |
| Timeout | ETIMEDOUT | 110 |

---

## Q3: Destructive Operations on Mounts

**Status:** ✅ ADOPTED (S2)

**Pattern:** Default non-destructive; explicit opt-in for deletes

### Design

| Mount Mode | Read | Write | Delete |
|------------|------|-------|--------|
| Default (read-only) | ✓ | ✗ | ✗ |
| `--writable` | ✓ | ✓ | ✗ |
| `--writable --allow-delete` | ✓ | ✓ | ✓ |

### Commands

```bash
# Default mount (read-only)
aos mount github --repo=owner/repo /aos/mnt/github

# Writable mount (no deletes)
aos mount github --repo=owner/repo --writable /aos/mnt/github

# Full access (explicit)
aos mount github --repo=owner/repo --writable --allow-delete /aos/mnt/github
```

### Behavior

```bash
# Attempt delete without --allow-delete
rm /aos/mnt/github/issues/123
# rm: cannot remove '/aos/mnt/github/issues/123': Operation not permitted
# Exit code: 1

# Explicit delete via CLI (always works if you have permission)
aos rm --force /aos/mnt/github/issues/123
# [OK] deleted issue 123
```

### Soft Delete (Optional)

For providers that support it:

```bash
aos mount s3 --writable --soft-delete /aos/mnt/s3
rm /aos/mnt/s3/bucket/file.csv
# Moves to /aos/mnt/s3/bucket/.trash/<timestamp>/file.csv
```

### Error Handling

| Condition | Error | Message |
|-----------|-------|---------|
| rm on read-only mount | EPERM | "Mount is read-only" |
| rm without --allow-delete | EPERM | "Destructive ops disabled; use 'aos rm --force' or remount with --allow-delete" |
| Soft-delete unsupported | Mount fails | "Provider does not support soft-delete" |

---

## Q4: Anti-Bankruptcy Circuit Breaker

**Status:** ✅ ADOPTED (S1 + S2 + S3 converge)

**Pattern:** Multi-layer defense + costly subtree locks + TTY detection + leases

### Defense in Depth Layers (All reviewers agree)

| Layer | Scope | Default | Purpose |
|-------|-------|---------|---------|
| 1 | Per-PID | $0.10/60s | Catch simple runaway |
| 2 | Per-Session (SID) | $1.00/session | Defeat fork-exec (S1) |
| 3 | Per-UID | $1.00/60s | Aggregate limit |
| 4 | Per-Subtree | Configurable | Isolate mounts |
| 5 | Global | $10.00/60s | Emergency backstop |
| 6 | Velocity | 100 ops/sec | Catch scanning |

### Costly Subtree Locks (S3 — stricter than S2's leases)

Costly paths are **locked by default** and require explicit unlock:

```bash
# Costly subtrees locked by default
cat /aos/mnt/github/issues/1
# cat: Permission denied (costly subtree locked)

# Check lock status
cat /aos/mnt/github/.aos/locked
# {"reason":"costly_subtree","requires":"aos unlock"}

# Unlock for limited time
aos unlock /aos/mnt/github --duration=30m
# Unlocked for 30 minutes (token: abc123)

# Now reads work
cat /aos/mnt/github/issues/1
# (content)

# Token expires, auto-relocks
```

### TTY Detection (S2)

Non-interactive processes blocked from costly paths by default:

```bash
# Background process (no TTY)
cat /aos/mnt/github/issues/1
# Permission denied (EACCES)

# Interactive shell (TTY) + unlock
aos unlock /aos/mnt/github --duration=30m
cat /aos/mnt/github/issues/1
# (works)
```

### File Layout

```
/aos/system/circuit_breaker/
├── config.yaml
├── status                      # "armed" | "tripped"
├── blocked/
│   ├── pid/
│   └── uid/
└── unlock_tokens/
    └── token_abc123.json

/aos/mnt/github/.aos/
├── locked                      # Exists if locked (default)
├── cost                        # Current session cost
└── limit                       # Subtree limit
```

### Configuration (S3)

```yaml
# /aos/etc/security/circuit_breaker.yaml
circuit_breaker:
  enabled: true

  subtrees:
    "/aos/mnt/**":
      default_locked: true
      requires_unlock: true
      max_unlock_duration_sec: 1800

  # Process classification
  known_safe: [aos-daemon, aos]
  known_dangerous: [mds, mdworker, updatedb, baloo_*]
```

### Commands

```bash
# Unlock with budget cap
aos unlock /aos/mnt/github --duration=30m --budget=5.00

# Batch job wrapper (S2)
aos run --unlock=/aos/mnt/github --duration=30m -- rsync -a /aos/mnt/github /backup/

# Check status
aos circuit-breaker status

# Manual reset after global trip
sudo aos circuit-breaker reset --confirm
```

### Error Handling

| Condition | Error | Exit |
|-----------|-------|------|
| Subtree locked | EACCES: "Use aos unlock" | 1 |
| Budget exhausted | EACCES or EAGAIN | 1 |
| Velocity exceeded | EACCES (throttled) | 1 |
| No TTY + costly path | EACCES | 1 |
| Global trip | EACCES: "Contact admin" | 1 |

---

## Q5: Action File Write Semantics

**Status:** ✅ ADOPTED (S2 + S3 converge)

**Pattern:** Commit-on-close buffer with 64KB limit

### Key Decision: Size Limit

| Reviewer | Limit | Rationale |
|----------|-------|-----------|
| S1 | PIPE_BUF (4KB) | Too restrictive |
| S2 | 1 MiB | Memory risk |
| S3 | 64KB | **Adopted** — covers normal prompts, forces large payloads to CLI |

### Semantics

| Behavior | Rule |
|----------|------|
| O_TRUNC | Ignored (both `>` and `>>` enqueue) |
| Multiple writes | Concatenated into single message |
| Commit point | Message enqueued on `close()` |
| Max size | 64KB; larger → EFBIG |
| lseek/pwrite | ESPIPE (no random access) |
| Rename into inbox | EPERM (prevents atomic-save editors) |

### Binary File Reference (S3 — cleaner than S2)

```bash
# @ prefix = "read this file, include as binary"
echo "@/tmp/image.png Describe this image" > /aos/agents/vision/inbox

# Daemon reads /tmp/image.png, encodes per agent config
```

Agent config for binary:
```yaml
spec:
  binary:
    accept: [image/png, image/jpeg, application/pdf]
    encoding: multimodal
```

### Commands

```bash
# Simple (works)
echo "Analyze this" > /aos/agents/researcher/inbox

# Multi-line (works - buffered until close)
cat << 'EOF' > /aos/agents/researcher/inbox
Analyze this code:
def foo(): pass
EOF

# Binary via @ reference (S3)
echo "@/path/to/image.png Describe this" > /aos/agents/vision/inbox

# Large payload (> 64KB) - use CLI
aos invoke researcher --prompt-file=big_prompt.txt

# Binary via CLI (preferred for complex)
aos invoke vision --attach=/path/to/image.png "Describe this"

# Too large (fails)
dd if=/dev/urandom bs=65537 count=1 > /aos/agents/researcher/inbox
# Error: File too large (EFBIG)
```

### Error Handling

| Condition | Error | Exit |
|-----------|-------|------|
| Payload > 64KB | EFBIG | 1 |
| Rename into inbox | EPERM | 1 |
| lseek attempt | ESPIPE | 1 |
| Queue full | EAGAIN | 1 |
| Invalid @ reference | ENOENT | 1 |
| Binary not accepted | EINVAL | 1 |

### Queue Management

```bash
# Check depth
cat /aos/agents/researcher/inbox.depth   # "7"
cat /aos/agents/researcher/inbox.limit   # "100"

# Peek without dequeue
cat /aos/agents/researcher/inbox.peek

# Clear queue (admin)
aos queue clear /aos/agents/researcher/inbox
```

---

## Q6: Override and Context Race Conditions

**Status:** ✅ ADOPTED (All three reviewers converge)

**Pattern:** CLI is canonical; JSON envelope for filesystem; **override/ deprecated**

### Recommended: CLI (All reviewers agree)

```bash
# All parameters in one atomic command
aos invoke researcher \
  --model=claude-opus-4-20250514 \
  --context=/aos/conversations/2025/01/04/abc123/context.ctx \
  --max-cost=2.00 \
  --timeout=600 \
  "Continue the analysis"
```

### Alternative: JSON Envelope (S2)

```bash
echo '{"query":"Continue","context":"/path/ctx","override":{"model":"opus"}}' \
  > /aos/agents/researcher/inbox
```

### Deprecated: Override Directory (S3 — explicit)

```bash
# DEPRECATED (race-prone, do not use):
echo "opus" > /aos/agents/researcher/override/model
echo "query" > /aos/agents/researcher/inbox

# The override/ directory may remain for backwards compatibility
# but documentation should steer users to CLI
```

### Session-Scoped Overrides (S3 addition)

For multi-turn with consistent settings:

```bash
# Create session with frozen config
aos session create researcher \
  --model=claude-opus-4-20250514 \
  --max-cost=5.00
# Created session s_049

# All invocations use session settings
echo "First query" > /aos/sessions/s_049/inbox
echo "Follow-up" > /aos/sessions/s_049/inbox
```

### Decision Tree (S3)

```
Need to invoke agent?
│
├── Simple query, no overrides?
│   └── echo "query" > /aos/agents/X/inbox  ✓
│
├── Query with overrides?
│   └── aos invoke X --model=opus "query"   ✓
│
├── Multi-turn with consistent settings?
│   └── aos session create X --model=opus
│       then: echo "query" > /aos/sessions/s_N/inbox
│
└── Atomic filesystem (JSON)?
    └── echo '{"query":"...","override":{...}}' > inbox
```

### Error Handling

| Scenario | Error | Exit |
|----------|-------|------|
| Invalid JSON | EINVAL | 1 |
| Unknown override field | EINVAL | 1 |
| Context not found | ENOENT | 1 |
| Override violates limits | EPERM | 1 |

---

## Q7b: Cost Visibility Before Execution

**Status:** ✅ ADOPTED (S1 + S2 converge)

**Pattern:** `--dry-run` / `--estimate` flag + safety gate

### CLI

```bash
# Estimate before fork
aos fork /aos/conversations/huge-ctx --prompt="test" --dry-run
# Estimated:
#   backend: openai (no KV fork)
#   tokens_to_serialize: 200000
#   estimated_cost_usd: 0.60 (range: 0.45-0.75)
#   will_exceed_budget: false
# Exit code: 0

# Estimate for invoke (S2 addition)
aos invoke researcher --estimate "Deep analysis"
# estimated_cost_usd: 0.42 (range: 0.30-0.70)

# Safety gate: expensive without override
aos fork /aos/conversations/huge-ctx --prompt="test"
# Error: Estimated cost $0.60 exceeds safety limit ($0.10).
# Use --force or --budget=1.00 to override.
# Exit code: 1

# Explicit override
aos fork /aos/conversations/huge-ctx --prompt="test" --budget=1.00
# Proceeds with $1.00 cap
```

### Filesystem Estimate

```bash
echo "fork /aos/conversations/huge-ctx" > /aos/system/cost/estimate
cat /aos/system/cost/estimate
# {"cost_usd": 0.60, "range": [0.45, 0.75], "approved": false}
```

### Safety Thresholds

| Threshold | Default | Override |
|-----------|---------|----------|
| Per-operation | $0.10 | `--budget=X` or `--force` |
| Session | $1.00 | `aos budget --add=Y` |

### Error Handling

| Condition | Exit Code |
|-----------|-----------|
| Estimate exceeds --max-cost | 66 BUDGET_EXHAUSTED |
| Estimation unavailable | 67 UPSTREAM_FAILURE |

---

## Q7c: Queue Depth and Backpressure

**Status:** ✅ ADOPTED (All three converge + S3 priority)

**Pattern:** Status file + EAGAIN + control file + priority queue

### File Layout

```
/aos/agents/researcher/
├── inbox                   # Normal queue
├── inbox.depth             # Current depth (e.g., "7")
├── inbox.limit             # Queue limit (e.g., "100")
├── inbox.peek              # View queued items
├── inbox.priority          # High-priority queue (S3)
├── inbox.priority.limit    # Priority limit (e.g., "10")
└── queue/
    └── ctl                 # Control: cancel, pause, resume
```

### Priority Queue (S3 addition)

```bash
# Normal queue
echo "regular task" > /aos/agents/researcher/inbox

# High-priority (bypasses normal queue, limited slots)
echo "urgent task" > /aos/agents/researcher/inbox.priority

# Check priority depth
cat /aos/agents/researcher/inbox.priority.limit   # "10"
```

### Commands

```bash
# Check queue before submitting
cat /aos/agents/researcher/inbox.depth   # "7"
cat /aos/agents/researcher/inbox.limit   # "100"

# Submit (succeeds if room)
echo "task" > inbox

# Queue full → EAGAIN
echo "task" > inbox
# bash: echo: write error: Resource temporarily unavailable
# Exit code: 1

# CLI with wait for space
aos invoke researcher --wait-queue --timeout=30s "task"

# CLI with queue check (fail fast)
aos invoke researcher --queue-check "task"

# Cancel queued item
echo "cancel q_000123" > /aos/agents/researcher/queue/ctl
```

### Backpressure Behavior

| Mode | Queue Full Behavior |
|------|---------------------|
| Default | EAGAIN (non-blocking) |
| `--wait-queue` | Block until space or timeout |

### States

| State | Meaning |
|-------|---------|
| `ok` | depth < 80% of limit |
| `pressure` | depth >= 80% of limit |
| `full` | depth == limit, EAGAIN |

### Configuration

```yaml
# /aos/agents/researcher/config.yaml
spec:
  queue:
    limit: 100
    priority_limit: 10
    overflow_action: eagain  # eagain | drop_oldest
```

---

# NEWLY ADOPTED SOLUTIONS (From S2)

## Q7: Exit Code Semantics

**Status:** ⚠️ NEEDS DECISION (S2 vs S3 differ)

**Options:**

### Option A: S2 Approach (Incremental)

| Code | Name | Core Dump? |
|------|------|------------|
| 64 | REFUSED | **Yes** |
| 65 | CTX_OVERFLOW | **Yes** |
| 66 | BUDGET_EXHAUSTED | No |
| 67 | UPSTREAM_FAILURE | Configurable |
| 68 | LOW_CONFIDENCE | No |
| 69 | CONTAMINATED | **Yes** |
| 70 | LEASE_EXPIRED | **Yes** |
| 71 | RATE_LIMITED | No |
| 72 | DECLINED | No (new) |

### Option B: S3 Approach (Range-based)

| Range | Category | Core Dump |
|-------|----------|-----------|
| 0 | Success | No |
| 1-63 | User/input errors | No |
| 64-95 | Resource limits (routine) | No |
| 96-127 | System errors (exceptional) | **Yes** |

Specific codes:
| Code | Name |
|------|------|
| 64 | BUDGET_EXHAUSTED |
| 65 | RATE_LIMITED |
| 66 | TIMEOUT |
| 67 | QUEUE_FULL |
| 96 | REFUSED |
| 97 | CTX_OVERFLOW |
| 98 | UPSTREAM_FAILURE |
| 99 | CONTAMINATED |
| 100 | LEASE_EXPIRED |
| 101 | LOW_CONFIDENCE_REFUSED |
| 102 | LOW_CONFIDENCE_UNCERTAIN |

### Recommendation

**Adopt S3's range-based approach** — cleaner mental model:
- 64-95: "Routine resource issues, retry or add budget"
- 96+: "Something broke, investigate"

### Configuration

```yaml
# /aos/etc/daemon.yaml
core_dumps:
  enabled: true
  range: [96, 127]  # Core dump on system errors only
```

---

## Q7a: Inbox Write Idempotency

**Status:** ✅ ADOPTED (S2)

**Pattern:** At-least-once default; opt-in idempotency via key

### Design

- Filesystem inbox: at-least-once (simple, retries cause duplicates)
- Opt-in idempotency via CLI flag or JSON envelope key
- Daemon dedupes in LRU cache (10 min window)

### Commands

```bash
# CLI with idempotency
aos invoke researcher --idempotency-key=build-123 "run tests"

# JSON envelope
echo '{"prompt":"run tests","idempotency_key":"build-123"}' > inbox
```

### Behavior

- First submission: enqueued normally
- Duplicate key within window: suppressed, success returned, logged:

```jsonl
{"type":"dedupe","idempotency_key":"build-123","result":"duplicate_suppressed"}
```

---

## Q7d: Testing and Mocking Story

**Status:** ✅ ADOPTED (S2)

**Pattern:** Record/replay + mock backend

### Design

1. **Record mode:** Capture tool results and model output
2. **Replay mode:** Re-run with recorded results (no external calls)
3. **Mock backend:** Local daemon mode for testing

### Commands

```bash
# Record a run
aos invoke researcher --record "Analyze this"

# Replay and verify
aos replay /aos/conversations/.../  --verify
# Output comparison: MATCH or DIFF

# Start daemon with mock backend
aosd --providers.mock

# Run tests against mock
aos invoke researcher "test query"  # Uses mock responses
```

### Limitations

- LLM output not deterministic; tests should assert on structured artifacts
- Record/replay requires initial real run

---

## Q7e: Stateful Directories from Pagination

**Status:** ✅ ADOPTED (S2)

**Pattern:** Remove writable cursor; use read-only marker + CLI

### Design

- `ls` returns first page only (e.g., 100 entries)
- Read-only `.more.json` contains cursor for next page
- CLI for pagination: `aos mnt ls --cursor=X`

### Commands

```bash
# First page
ls /aos/mnt/github/issues/
# 1 2 3 ... 100 .more.json

# Check for more
cat /aos/mnt/github/issues/.more.json
# {"next":"abc123","has_more":true}

# Get next page (CLI)
aos mnt ls /aos/mnt/github/issues --cursor=abc123
# 101 102 ... 200
```

### Trade-off

Filesystem browsing is limited to first page. This matches Unix reality: simple browsing in FS, complex operations via CLI.

---

## Q7f: Session vs Conversation Relationship

**Status:** ✅ ADOPTED (S2)

**Pattern:** Session = thread container; Turn = single invocation

### Design

```
/aos/conversations/
├── sessions/
│   └── s_047/
│       ├── meta.json
│       ├── turns/
│       │   ├── 0001 -> ../../2025/01/04/a3f8c2
│       │   └── 0002 -> ../../2025/01/04/b7c2d1
│       └── latest -> turns/0002
└── 2025/01/04/
    ├── a3f8c2/           # Turn 1 (immutable)
    └── b7c2d1/           # Turn 2 (immutable)
```

### Semantics

| Concept | Definition |
|---------|------------|
| Session | Stable conversation thread ID (`s_047`) |
| Turn | Single invocation, immutable directory |
| Relationship | Session contains turns via symlinks |

---

## Q8-Q15: Clarification Solutions (S2)

### Q8: Contamination Tracking Scope

Add explicit **Threat Model** section to §12:
- Color tracked at **process and Agent OS file boundary**
- Shell pipes outside daemon mediation are out of scope
- Recommend `aos pipe` for tracked cross-agent flows

### Q9: Filesystem vs CLI Positioning

Document decision rule:
- **Filesystem:** Read/observe, simple invoke
- **CLI:** Atomic invocation, overrides, attachments, pagination, mounts, approvals

### Q10: Async Invocation Feedback

- `echo > inbox` = "enqueue only" (no completion guarantee)
- For scripts: `aos invoke --wait` returns exit code, prints output

### Q10a: Orphaned Conversation Cleanup

- `aos gc --now` for immediate cleanup
- `aos invoke --ephemeral` stores under `/aos/conversations/tmp/` (short retention)
- Optional count/size-based pruning policy

### Q11: Ring 0 vs Ring 1 Distinction

Ring 0 enforces **identity/resource** policy (uid, cgroup, limits, leases).
Ring 1 enforces **semantic/content** policy.

### Q12: Blocking Read Implementation

Spec requirement: blocking reads must be **interruptible** and support daemon timeout (`ETIMEDOUT`).

### Q13: Replay Terminology

Rename `"deterministic": true` to `"replayable": true`.

### Q14: Virtual Size Reporting

Remove st_size=0 trick. Use truthful metadata + gating/leases (Q4) for protection.

### Q15: Blocking Reads — FIFOs vs Files

Keep file interface. Add non-blocking mirrors:
- `stream.last` (last complete chunk)
- `approval.last` (last result)

All blocking endpoints support timeout and `ETIMEDOUT`.

---

# CHANGELOG

| Date | Change |
|------|--------|
| 2025-01-05 | Created document |
| 2025-01-05 | Added S1 solutions for Q1, Q2, Q4, Q5, Q6, Q7b, Q7c |
| 2025-01-05 | Added S2 solutions, merged with S1 |
| 2025-01-05 | Q1: Added config hash audit trail (S2) |
| 2025-01-05 | Q2: Added signature-based approval option (S2) |
| 2025-01-05 | Q3: Added complete solution for destructive ops (S2) |
| 2025-01-05 | Q4: Added TTY detection + lease system (S2) |
| 2025-01-05 | Q5: Switched to commit-on-close (S2, better compatibility) |
| 2025-01-05 | Q6: Switched to JSON envelope (S2, simpler than Maildir) |
| 2025-01-05 | Q7: Added detailed exit code classification (S2) |
| 2025-01-05 | Q7a-Q7f: Added all solutions from S2 |
| 2025-01-05 | Q8-Q15: Added clarification solutions from S2 |
| 2025-01-05 | Added S3 solutions, final merge |
| 2025-01-05 | Q2: Added policy-based auto-approval (S3) |
| 2025-01-05 | Q4: Added "costly subtree locked by default" (S3) |
| 2025-01-05 | Q5: Set 64KB limit (S3), added @ file reference syntax |
| 2025-01-05 | Q6: Explicitly deprecated override/ directory (S3) |
| 2025-01-05 | Q7: Added range-based exit code option (S3) |
| 2025-01-05 | Q7c: Added priority queue (S3) |
| 2025-01-05 | Marked all critical questions as solved with 3-way convergence |
