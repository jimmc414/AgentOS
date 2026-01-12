# Agent OS Solutions — Summary

**Reviewer:** Claude (Opus 4.5)
**Date:** 2025-01-05
**Full Solutions:** See `SOLUTIONS.md`

---

## Critical Solutions (Q1, Q2, Q4, Q5)

### Q1: Agent Config Privilege Escalation

**Problem:** Agents can modify their own `config.yaml`, escalating privileges on next invocation.

**Solution:** Split config into read-only projection and operator-controlled source:

```
/aos/etc/agents.d/researcher.yaml    # Source of truth (mode 0600, owner aos-admin)
/aos/agents/researcher/config.yaml   # FUSE projection (mode 0444, read-only)
/aos/agents/researcher/preferences.yaml  # Agent-writable, non-security only
```

**Key Commands:**
```bash
# Agent reads config (works)
cat /aos/agents/researcher/config.yaml

# Agent writes config (FAILS)
echo "model: opus" > /aos/agents/researcher/config.yaml
# Error: Read-only file system

# Operator updates config
sudo aos config researcher --set model=claude-opus-4-20250514
```

**Training Pattern Match:** `/etc/` for system config, read-only files, file permissions — all familiar Unix patterns.

---

### Q2: Intent Approval Authorization

**Problem:** No specification of WHO can approve intents. Agents can approve their own.

**Solution:** UID separation — approver UID must differ from creator UID:

```
/aos/procs/1234/intents/
├── pending/
│   ├── 001.json        # Intent (created by agent UID 1001)
│   └── 001.approval    # Approval (must be written by different UID)
├── approved/
└── rejected/
```

**Key Commands:**
```bash
# Agent creates intent (automatic)
# Agent tries to approve own intent (FAILS)
echo '{"approved":true}' > /aos/procs/1234/intents/pending/001.approval
# Error: Cannot approve own intent (EPERM)

# Human/operator approves (works)
aos approve 1234/001 --reason="Verified safe"
```

**Policy-Based Auto-Approval:**
```yaml
policies:
  - name: auto_approve_cheap_reads
    match:
      action: ["fs.read"]
      cost_estimate_usd: {max: 0.10}
    approval: auto
```

**Training Pattern Match:** UID-based permissions are kernel-enforced Unix fundamentals.

---

### Q4: Anti-Bankruptcy Circuit Breaker Design

**Problem:** Per-PID cost tracking is bypassable via PID churn (`find -exec`, `xargs`, forking).

**Solution:** Multi-layer defense with progressively broader scope:

| Layer | Scope | Default Limit | Catches |
|-------|-------|---------------|---------|
| 1 | Per-PID | $0.10/60s | Simple cases |
| 2 | Per-UID | $1.00/60s | PID churn |
| 3 | Per-subtree | Configurable | Mount-level |
| 4 | Global | $10.00/60s | Emergency stop |

**Costly Subtrees (Default Locked):**
```bash
# Costly mounts require explicit unlock
cat /aos/mnt/github/issues/1
# Error: Permission denied (costly subtree locked)

# Unlock for limited time
aos unlock /aos/mnt/github --duration=30m
# Unlocked for 30 minutes (token: abc123)

# Now reads work
cat /aos/mnt/github/issues/1
```

**Key Files:**
```
/aos/system/circuit_breaker/
├── status              # "armed" | "tripped"
├── blocked/pid/        # Blocked PIDs
├── blocked/uid/        # Blocked UIDs
└── unlock_tokens/      # Active unlock tokens
```

**Training Pattern Match:** `EACCES` for permission denied, time-limited tokens, explicit unlock commands.

---

### Q5: Action File Write Semantics

**Problem:** POSIX write behavior (partial writes, O_TRUNC, multiple writes) clashes with queue semantics.

**Solution:** Action files behave as **write-once message buffers**:

| Behavior | Semantics |
|----------|-----------|
| `write()` calls | Buffered until close |
| `close()` | Commits message atomically |
| Multiple opens | Each is fresh buffer |
| O_TRUNC flag | Ignored |
| Max size | 64KB (EFBIG if exceeded) |

**Commands:**
```bash
# Simple invocation (works)
echo "What is fusion energy?" > /aos/agents/researcher/inbox

# Multi-line (works - buffered until close)
cat << 'EOF' > /aos/agents/researcher/inbox
Analyze this code:
def foo(): pass
EOF

# Too large (fails)
dd if=/dev/urandom bs=65537 count=1 > /aos/agents/researcher/inbox
# Error: File too large (EFBIG)

# Binary via file reference
echo "@/path/to/image.png Describe this" > /aos/agents/vision/inbox

# Complex invocation (use CLI)
aos invoke researcher --attach=/path/to/image.png "Describe this"
```

**Training Pattern Match:** `echo "text" > file` works exactly as expected — one atomic message.

---

## High Priority Solutions (Q6, Q7b, Q7c)

### Q6: Override and Context Race Conditions

**Problem:** Writing to `override/model` then `inbox` are two operations — raceable.

**Solution:** CLI is canonical for complex invocations:

```bash
# RECOMMENDED: Atomic CLI command
aos invoke researcher \
  --model=claude-opus-4-20250514 \
  --context=/aos/conversations/.../context.ctx \
  "Continue the analysis"

# ALTERNATIVE: Atomic JSON message
echo '{"query":"Continue","context":"/path/ctx","override":{"model":"opus"}}' \
  > /aos/agents/researcher/inbox

# DEPRECATED: Sequential writes (race-prone)
echo "opus" > /aos/agents/researcher/override/model
echo "query" > /aos/agents/researcher/inbox
```

**Decision Tree:**
```
Simple query, no overrides?  → echo "query" > inbox
Query with overrides?        → aos invoke X --model=opus "query"
Multi-turn session?          → aos session create X --model=opus
```

---

### Q7b: Cost Visibility Before Execution

**Problem:** Fork/bind/replay costs vary wildly by backend, not visible until after execution.

**Solution:** Mandatory `--dry-run` and threshold warnings:

```bash
# Dry-run shows cost without executing
aos fork /aos/conversations/large/ --prompt="variant" --dry-run
# Estimated cost: $0.60
#   Context: 200,000 tokens
#   Backend: anthropic (no KV-cache sharing)

# Threshold warning (>$1.00)
aos fork /aos/conversations/huge/ --prompt="variant"
# WARNING: Estimated cost $2.34 exceeds threshold
# Proceed? [y/N]

# Skip confirmation
aos fork /aos/conversations/huge/ --prompt="variant" --yes
```

---

### Q7c: Queue Depth and Backpressure

**Problem:** No way to check inbox queue state before submitting work.

**Solution:** Bounded queues with visible depth:

```bash
# Check queue before submitting
cat /aos/agents/researcher/inbox.depth   # "7"
cat /aos/agents/researcher/inbox.limit   # "100"

# Submit (succeeds if room)
echo "task" > /aos/agents/researcher/inbox

# Queue full (EAGAIN)
echo "task" > /aos/agents/researcher/inbox
# Error: Resource temporarily unavailable
```

---

## Additional Solutions

### Q3: Destructive Operations on Mounts

**Solution:** Read-only by default, explicit `--writable` opt-in, soft-delete for destructive ops:

```bash
# Default (read-only)
aos mount github --token=$TOKEN /aos/mnt/github
rm /aos/mnt/github/issues/123  # EROFS

# Writable with soft-delete
aos mount github --token=$TOKEN --writable --soft-delete /aos/mnt/github
rm /aos/mnt/github/issues/123  # Moves to .trash/
aos mount gc /aos/mnt/github   # Actually deletes
```

### Q7: Exit Code Semantics

**Solution:** Reorganized exit codes:

| Range | Category | Core Dump |
|-------|----------|-----------|
| 0 | Success | No |
| 1-63 | User errors | No |
| 64-95 | Resource limits (routine) | No |
| 96-127 | System errors (exceptional) | Yes |

---

## Design Principles Applied

| Solution | Training Pattern |
|----------|-----------------|
| Q1: Config split | `/etc/` for config, 0600/0444 permissions |
| Q2: UID separation | Unix UID-based security |
| Q4: Circuit breaker | `EACCES`, time-limited tokens |
| Q5: Atomic buffer | `echo > file` is one message |
| Q6: CLI canonical | `command --flag=value` |
| Q7b: Dry-run | `--dry-run` flag pattern |
| Q7c: EAGAIN | Standard errno for backpressure |

---

## Implementation Priority

### Immediate (blocks v1)
1. Q1: Agent config privilege (file permissions)
2. Q2: Intent approval (UID separation)
3. Q5: Inbox write semantics (atomic buffer)

### High (needed for safety)
4. Q4: Circuit breaker (multi-layer)
5. Q6: Override races (CLI canonical)

### Medium (needed for usability)
6. Q7b: Cost visibility (--dry-run)
7. Q7c: Queue depth (bounded + EAGAIN)
8. Q3: Mount safety (read-only default)

### Documentation
- Q7-Q15: Clarifications and future work markers

---

## Cross-Reviewer Agreement Addressed

| Issue | Solution |
|-------|----------|
| Self-modifying config | Q1: Read-only projection |
| Unsigned approvals | Q2: UID separation + signatures |
| PID churn bypass | Q4: Multi-layer + UID tracking |
| Write semantics clash | Q5: Atomic buffer |
| CLI is real control plane | Q6: CLI canonical |

---

## Files Created

- `SOLUTIONS.md` — Full detailed solutions with file layouts, commands, error handling
- `SOLUTIONS_SUMMARY.md` — This summary document

---

*Solutions proposed by Claude (Opus 4.5) — 2025-01-05*
