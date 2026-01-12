# Agent OS: Solutions to Unsolved Questions

**Reviewer:** Claude (Opus 4.5)
**Date:** 2025-01-05
**Status:** Solution Proposal

---

## Q1: Agent Config Privilege Escalation

### Solution

Split configuration into **read-only projection** (agent-visible) and **operator-controlled source** (admin-writable):

1. **Source of truth:** `/aos/etc/agents.d/{name}.yaml` — owned by `aos-admin`, mode `0600`
2. **Agent view:** `/aos/agents/{name}/config.yaml` — FUSE-generated read-only projection, mode `0444`
3. **Agent preferences:** `/aos/agents/{name}/preferences.yaml` — agent-writable, non-security-relevant only

The FUSE layer projects the config read-only. Any write attempt returns `EROFS` (Read-only file system).

### File Layout

```
/aos/etc/agents.d/              # Source of truth (operator only)
├── researcher.yaml             # Mode 0600, owner aos-admin
├── coder.yaml
└── .schema.yaml                # Validation schema

/aos/agents/researcher/
├── config.yaml                 # Mode 0444, FUSE projection (read-only)
├── preferences.yaml            # Mode 0644, agent-writable (non-security)
├── status
├── inbox
└── ...
```

### Operator Commands

```bash
# Read agent config (works for everyone)
cat /aos/agents/researcher/config.yaml

# Agent tries to write (FAILS)
echo "model: opus" > /aos/agents/researcher/config.yaml
# bash: /aos/agents/researcher/config.yaml: Read-only file system
# Exit code: 1

# Operator updates config (requires privilege)
sudo aos config researcher --set model=claude-opus-4-20250514
# Or directly edit source:
sudo vim /aos/etc/agents.d/researcher.yaml

# Hot-reload (optional, if daemon supports)
aos config researcher --reload
```

### Agent Preferences (Non-Security)

Agents can write to `preferences.yaml` for non-security settings:

```yaml
# /aos/agents/researcher/preferences.yaml (agent-writable)
output_format: markdown
citation_style: inline
response_length: concise
```

**What CAN go in preferences:**
- Output formatting preferences
- Response style choices
- Non-security behavioral hints

**What CANNOT go in preferences (enforced by daemon):**
- Model selection
- Capabilities/tools grants
- Budget limits
- Memory access paths
- Spawn permissions

If agent tries to write a security-relevant field:
```bash
echo "capabilities: [fs.write]" >> /aos/agents/researcher/preferences.yaml
# Daemon ignores security fields in preferences.yaml
# Or: FUSE rejects write with EPERM
```

### Hot-Reload Behavior

When operator modifies source config:
1. Daemon watches `/aos/etc/agents.d/` via inotify
2. Validates new config against schema
3. If valid: updates FUSE projection, applies to new sessions
4. Running processes keep their original config (no mid-session changes)

```bash
# Check if config changed
stat /aos/agents/researcher/config.yaml
# mtime reflects last operator change

# Force apply to next invocation
aos config researcher --reload
```

### Error Handling

| Action | Result | Exit Code |
|--------|--------|-----------|
| Agent reads config | Success | 0 |
| Agent writes config | `EROFS` | 1 |
| Agent writes preferences (valid) | Success | 0 |
| Agent writes preferences (security field) | `EPERM` or ignored | 1 |
| Operator edits source (valid) | Reload | 0 |
| Operator edits source (invalid) | Rejected, old config kept | 1 |

### Trade-offs

**Costs:**
- Two config locations adds complexity
- Operators must use sudo/privilege for config changes
- Agent templates need adjustment (copy to `/aos/etc/agents.d/`)

**Rejected Alternatives:**
- **Capability-based write restrictions**: Too complex to enforce at syscall level
- **Signed configs**: Adds PKI complexity; file permissions are simpler
- **Config shadowing**: Agent writes to shadow file, daemon ignores — confusing UX

### Dependencies

- **Q2:** Same pattern applies to approval files (admin-only write)
- **Q6:** Config changes don't race with invocation (config is read at session start)

---

## Q2: Intent Approval Authorization

### Solution

Approvals require **different principal** than intent creator, enforced via **UID separation** plus **optional cryptographic signatures** for high-value operations:

1. **Structural rule:** Approver UID ≠ Creator UID
2. **File permissions:** Approval files in `intents/pending/` are mode `0644`, but daemon checks writer UID
3. **Signature requirement:** Operations above threshold require signed approval

### File Layout

```
/aos/procs/1234/intents/
├── pending/
│   ├── 001.json                # Intent file (created by agent)
│   └── 001.approval            # Approval file (must be written by different UID)
├── approved/
│   └── 000.json                # Completed intents
└── rejected/
    └── 002.json                # Rejected intents
```

### Intent File Format

```json
// /aos/procs/1234/intents/pending/001.json (created by agent, PID 1234)
{
  "id": "001",
  "created_by_pid": 1234,
  "created_by_uid": 1001,
  "action": "fs.delete",
  "path": "/important/file.txt",
  "reason": "cleanup old files",
  "cost_estimate_usd": 0.00,
  "risk_level": "high",
  "requires_signature": true,
  "created_at": "2025-01-05T10:30:00Z",
  "status": "pending"
}
```

### Approval Methods

**Method A: File-based (low-risk operations)**

```bash
# Different UID writes approval
echo '{"approved":true}' > /aos/procs/1234/intents/pending/001.approval

# Daemon checks:
# 1. Writer UID != created_by_uid
# 2. Writer has approval permission (aos-approver group)
```

**Method B: CLI with signature (high-risk operations)**

```bash
# Human approves via CLI
aos approve 1234/001 --reason="Verified cleanup needed"
# Prompts for confirmation, signs with user's key

# Or batch approve
aos approve 1234/001 1234/002 1234/003 --reason="All verified"
```

**Method C: Human channel (interactive)**

```bash
# Approval routed to human channel
cat /aos/procs/1234/intents/pending/001.json
# {"action":"fs.delete",...,"requires_human":true}

# Human responds via channel
echo '{"approved":true,"signer":"alice@ed25519:abc123"}' \
  > /aos/channels/human/alice/response
```

### Approval File Format

```json
// /aos/procs/1234/intents/pending/001.approval
{
  "approved": true,
  "approver_uid": 1000,
  "approver_name": "alice",
  "reason": "Verified this is safe",
  "timestamp": "2025-01-05T10:31:00Z",
  "signature": "ed25519:...",  // Required if requires_signature=true
  "method": "cli"              // cli | file | channel
}
```

### Daemon Enforcement

```python
# Pseudocode for approval validation
def validate_approval(intent, approval, writer_uid):
    # Rule 1: Different principal
    if writer_uid == intent.created_by_uid:
        raise PermissionError("Cannot approve own intent")

    # Rule 2: Writer in approvers group
    if writer_uid not in get_approvers():
        raise PermissionError("Not authorized to approve")

    # Rule 3: Signature if required
    if intent.requires_signature:
        if not verify_signature(approval.signature, approval.approver_name):
            raise PermissionError("Invalid signature")

    return True
```

### Policy-Based Auto-Approval

Low-risk operations can be auto-approved by policy:

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

When policy auto-approves:
```json
// /aos/procs/1234/intents/pending/001.approval
{
  "approved": true,
  "approver_uid": 0,
  "approver_name": "policy:auto_approve_cheap_reads",
  "reason": "Auto-approved by policy",
  "timestamp": "2025-01-05T10:31:00Z"
}
```

### Intent State Machine

```
                    ┌─────────────────────┐
                    │                     │
    ┌───────────────▼───────────────┐     │
    │           PENDING             │     │ timeout
    │                               │     │
    └───────────────┬───────────────┘     │
                    │                     │
         ┌──────────┴──────────┐          │
         │                     │          │
         ▼                     ▼          │
┌─────────────────┐   ┌─────────────────┐ │
│    APPROVED     │   │    REJECTED     │ │
└────────┬────────┘   └─────────────────┘ │
         │                                │
         ▼                                │
┌─────────────────┐                       │
│    EXECUTED     │                       │
└────────┬────────┘                       │
         │                                │
         ▼                                │
┌─────────────────┐   ┌─────────────────┐ │
│    COMPLETED    │   │     EXPIRED     │◄┘
└─────────────────┘   └─────────────────┘
```

### Commands

```bash
# List pending intents
ls /aos/procs/1234/intents/pending/
# 001.json  002.json

# View intent details
cat /aos/procs/1234/intents/pending/001.json

# Approve via CLI (recommended)
aos approve 1234/001

# Reject via CLI
aos reject 1234/001 --reason="Too risky"

# Bulk operations
aos approve --all --proc=1234 --risk-level=low
```

### Error Handling

| Scenario | Error | Exit Code |
|----------|-------|-----------|
| Agent approves own intent | `EPERM: Cannot approve own intent` | 1 |
| Unauthorized UID approves | `EACCES: Not in approvers group` | 1 |
| Missing signature (required) | `EINVAL: Signature required` | 1 |
| Approval after timeout | `ETIMEDOUT: Intent expired` | 110 |
| Invalid intent ID | `ENOENT: Intent not found` | 1 |

### Trade-offs

**Costs:**
- Requires UID separation (agent runs as different user than operator)
- Signature infrastructure for high-value operations
- More files in intents directory

**Rejected Alternatives:**
- **Token-based**: Tokens can be stolen; UIDs are kernel-enforced
- **Capability-based**: Too complex; UID + group membership is familiar
- **Agent-signed**: Doesn't solve the problem (agent signs own approval)

### Dependencies

- **Q1:** Uses same admin-privilege pattern
- **Q4:** Approval policy can include cost thresholds

---

## Q4: Anti-Bankruptcy Circuit Breaker Design

### Solution

**Multi-layer defense** with progressively broader scope:

| Layer | Scope | Default Limit | Granularity |
|-------|-------|---------------|-------------|
| 1 | Per-PID | $0.10/60s | Individual process |
| 2 | Per-UID | $1.00/60s | User account |
| 3 | Per-subtree | Configurable | Mount point / directory |
| 4 | Global | $10.00/60s | Entire system |

Plus **costly subtree protection** requiring explicit unlock.

### File Layout

```
/aos/system/circuit_breaker/
├── config.yaml                 # Circuit breaker configuration
├── status                      # "armed" | "tripped" | "unlocked"
├── blocked/                    # Currently blocked entities
│   ├── pid/
│   │   └── 12345.json
│   └── uid/
│       └── 1001.json
├── history.jsonl               # Breach history
└── unlock_tokens/              # Active unlock tokens
    └── token_abc123.json

/aos/mnt/github/.aos/
├── cost                        # Current cost for this mount
├── limit                       # Limit for this subtree
└── locked                      # Exists if locked (default)
```

### Configuration

```yaml
# /aos/etc/security/circuit_breaker.yaml
circuit_breaker:
  enabled: true

  layers:
    per_pid:
      threshold_usd: 0.10
      window_sec: 60
      action: block
      cooldown_sec: 300

    per_uid:
      threshold_usd: 1.00
      window_sec: 60
      action: block
      cooldown_sec: 600

    per_subtree:
      # Defined per-mount, see subtrees section

    global:
      threshold_usd: 10.00
      window_sec: 60
      action: emergency_stop

  subtrees:
    "/aos/mnt/**":
      default_locked: true
      requires_unlock: true
      max_unlock_duration_sec: 1800

    "/aos/conversations/**":
      default_locked: false
      threshold_usd: 5.00
      window_sec: 300

  # Process classification
  known_safe:
    - aos-daemon
    - aos

  known_dangerous:
    - mds
    - mdworker
    - updatedb
    - baloo_*
```

### Layer 1: Per-PID (catches simple cases)

```bash
# PID 12345 triggers multiple reads
cat /aos/mnt/github/issues/1
cat /aos/mnt/github/issues/2
cat /aos/mnt/github/issues/3  # Cost now $0.09
cat /aos/mnt/github/issues/4  # Cost now $0.12 - BLOCKED

# Result:
# cat: /aos/mnt/github/issues/4: Permission denied

# Check blocked status
cat /aos/system/circuit_breaker/blocked/pid/12345.json
# {"pid":12345,"comm":"cat","cost_usd":0.12,"blocked_at":"...","expires":"..."}
```

### Layer 2: Per-UID (catches PID churn)

```bash
# User spawns many processes to evade per-PID limit
find /aos/mnt/github/issues -exec cat {} \;
# Each cat is new PID, stays under $0.10
# But total for UID 1001 exceeds $1.00

# Result: All processes for UID 1001 blocked
cat /aos/system/circuit_breaker/blocked/uid/1001.json
# {"uid":1001,"username":"alice","cost_usd":1.23,"pids":[...],"blocked_at":"..."}
```

### Layer 3: Per-Subtree (default locked for costly trees)

```bash
# Costly subtrees require explicit unlock
ls /aos/mnt/github/issues/
# ls: cannot access '/aos/mnt/github/issues/': Permission denied

# Check lock status
cat /aos/mnt/github/.aos/locked
# {"reason":"costly_subtree","requires":"aos unlock"}

# Unlock (time-limited)
aos unlock /aos/mnt/github --duration=30m
# Unlocked /aos/mnt/github for 30 minutes (token: abc123)

# Now reads work
ls /aos/mnt/github/issues/
# 1  2  3  ...

# Token expires, re-locks automatically
cat /aos/system/circuit_breaker/unlock_tokens/token_abc123.json
# {"path":"/aos/mnt/github","uid":1000,"expires":"2025-01-05T11:00:00Z"}
```

### Layer 4: Global (emergency stop)

```bash
# Total system cost exceeds global limit
# ALL costly operations blocked

cat /aos/system/circuit_breaker/status
# tripped

cat /aos/system/circuit_breaker/history.jsonl | tail -1
# {"ts":"...","layer":"global","cost_usd":10.47,"action":"emergency_stop"}

# Manual reset required
sudo aos circuit-breaker reset --confirm
```

### Handling EDR/Backup Agents

For host-level agents we don't control:

```yaml
# /aos/etc/security/circuit_breaker.yaml
external_agents:
  strategy: subtree_lock

  # Known backup/security tools
  allow_metadata_only:
    - rsync
    - tar
    - backup-agent

  # Complete blocklist
  block:
    - endpoint-security-*
    - carbon-black-*
```

**Metadata-only mode:**
```bash
# rsync sees files but can't read content
rsync --list-only /aos/mnt/
# Returns file listing with st_size=0

# Actual read requires unlock
rsync /aos/mnt/github/ /backup/
# rsync: error reading /aos/mnt/github/issues/1: Permission denied
```

### Commands

```bash
# Unlock subtree (time-limited)
aos unlock /aos/mnt/github --duration=30m
aos unlock /aos/mnt/s3 --duration=1h --budget=5.00

# Check circuit breaker status
aos circuit-breaker status
# Layer    | Status  | Blocked
# per_pid  | armed   | 2
# per_uid  | armed   | 0
# subtree  | armed   | 3 locked
# global   | armed   | -

# View blocked entities
aos circuit-breaker list-blocked

# Unblock specific PID (manual override)
sudo aos circuit-breaker unblock --pid=12345

# Reset global trip
sudo aos circuit-breaker reset --confirm

# Dry-run mode (shows what would be charged)
aos unlock --dry-run /aos/mnt/github
# Subtree /aos/mnt/github
# Current lock: yes
# Estimated cost for ls: $0.00 (metadata only)
# Estimated cost for full read: $47.23 (15,743 objects)
```

### Error Messages

| Scenario | Error | Exit Code |
|----------|-------|-----------|
| Per-PID limit | `EACCES: Cost limit exceeded (PID $0.10/60s)` | 1 |
| Per-UID limit | `EACCES: Cost limit exceeded (UID $1.00/60s)` | 1 |
| Subtree locked | `EACCES: Costly subtree locked. Use: aos unlock PATH` | 1 |
| Global trip | `EACCES: System cost limit exceeded. Contact admin.` | 1 |
| Blocked indexer | `EACCES: Indexer blocked by policy` | 1 |

### Trade-offs

**Costs:**
- Multiple tracking layers add complexity
- Unlock tokens require management
- False positives may block legitimate batch jobs

**Mitigations:**
- Clear error messages with remediation
- `aos unlock` is easy to use
- Configurable thresholds per use case

**Rejected Alternatives:**
- **Per-PID only**: Bypassable via PID churn
- **Blocklist only**: Fragile, can't predict all indexer names
- **Always-unlocked**: Defeats the purpose

### Dependencies

- **Q2:** Unlock tokens can be approval-gated
- **Q7b:** Dry-run mode shows cost before execution

---

## Q5: Action File Write Semantics

### Solution

Action files (`inbox`, `ctl`) behave as **write-once message buffers**:

1. **Buffered write**: Content is buffered on each `write()` call
2. **Atomic commit on close**: Message is committed when file handle is closed
3. **Single message per open**: Second open starts fresh buffer
4. **Size limit**: Messages must be ≤ 64KB (enforced)
5. **O_TRUNC ignored**: Truncate flag has no effect

This matches how agents think about `echo "text" > file` — one atomic message.

### Semantics Table

| Operation | Behavior |
|-----------|----------|
| `echo "query" > inbox` | Buffer "query\n", commit on close |
| `echo "a"; echo "b"` (same handle) | Buffer "a\nb\n", commit on close |
| `printf "long..." > inbox` | Multiple write() calls, buffer all, commit on close |
| Open with O_TRUNC | Ignored (not an error) |
| Second open (new handle) | Fresh buffer, previous message already committed |
| Write > 64KB | `EFBIG` on close |

### Implementation

```python
# FUSE handler pseudocode
class InboxFile:
    def __init__(self):
        self.buffers = {}  # fd -> bytes

    def open(self, flags):
        fd = allocate_fd()
        self.buffers[fd] = b""
        return fd

    def write(self, fd, data, offset):
        # Ignore offset - always append to buffer
        self.buffers[fd] += data
        if len(self.buffers[fd]) > 65536:
            return -EFBIG
        return len(data)

    def release(self, fd):
        # Commit message on close
        message = self.buffers.pop(fd)
        if message:
            self.queue.append(message)
        return 0
```

### Commands and Outputs

```bash
# Simple invocation (works)
echo "What is fusion energy?" > /aos/agents/researcher/inbox
# Exit code: 0

# Multi-line query (works - buffered)
cat << 'EOF' > /aos/agents/researcher/inbox
Please analyze this code:
```python
def foo(): pass
```
EOF
# Exit code: 0

# Large message (fails if > 64KB)
dd if=/dev/urandom bs=65537 count=1 > /aos/agents/researcher/inbox
# dd: error writing '/aos/agents/researcher/inbox': File too large
# Exit code: 1

# Binary via reference (use @ prefix for file reference)
echo "@/path/to/image.png" > /aos/agents/vision/inbox
# Daemon reads /path/to/image.png, encodes as configured
# Exit code: 0

# Complex invocation (use CLI)
aos invoke researcher \
  --model=claude-opus-4-20250514 \
  --attach=/path/to/image.png \
  "Describe this image"
```

### Binary Input Handling

For agents that accept binary (images, audio):

```yaml
# Agent config
spec:
  binary:
    accept: [image/png, image/jpeg, application/pdf]
    encoding: multimodal
```

**Option A: File reference (simple)**
```bash
# @ prefix means "read this file and include as binary"
echo "@/tmp/image.png Describe this image" > /aos/agents/vision/inbox
```

**Option B: CLI (recommended for complex)**
```bash
aos invoke vision --attach=/tmp/image.png "Describe this image"
```

**Option C: Base64 inline (fallback)**
```bash
echo "data:image/png;base64,$(base64 /tmp/image.png) Describe this" \
  > /aos/agents/vision/inbox
```

### Queue Management

```bash
# Check queue depth
cat /aos/agents/researcher/inbox.depth
# 3

# Check queue limit
cat /aos/agents/researcher/inbox.limit
# 100

# Clear queue (admin only)
aos queue clear /aos/agents/researcher/inbox

# Peek at queue (doesn't dequeue)
cat /aos/agents/researcher/inbox.peek
# {"position":0,"content":"first query","submitted":"..."}
```

### Error Handling

| Scenario | Error | Exit Code |
|----------|-------|-----------|
| Message > 64KB | `EFBIG: Message too large (max 64KB)` | 1 |
| Queue full | `EAGAIN: Queue full (100/100)` | 1 |
| Binary without support | `EINVAL: Agent does not accept binary` | 1 |
| Invalid file reference | `ENOENT: Referenced file not found` | 1 |

### Trade-offs

**Costs:**
- 64KB limit may be restrictive for some use cases
- File reference syntax (`@/path`) is non-standard
- Binary handling adds complexity

**Why 64KB?**
- Well above typical prompt sizes
- Safely under memory pressure
- Forces large payloads through CLI (proper error handling)

**Rejected Alternatives:**
- **PIPE_BUF (4KB)**: Too restrictive for normal prompts
- **Unlimited buffering**: Memory exhaustion risk
- **Multiple endpoints**: Fragments the interface

### Dependencies

- **Q6:** Eliminates some race conditions (atomic commit)
- **Q7a:** Idempotency can be achieved via message ID prefix

---

## Q6: Override and Context Race Conditions

### Solution

Adopt **CLI as canonical interface** for complex invocations, with **atomic JSON inbox** as filesystem alternative:

**Primary recommendation:** Use CLI for any invocation with overrides or context:
```bash
aos invoke researcher \
  --model=claude-opus-4-20250514 \
  --context=/path/to/context.ctx \
  "Query"
```

**Filesystem alternative:** Atomic JSON message to inbox:
```bash
echo '{"query":"Continue analysis","context":"/path/to/ctx","override":{"model":"opus"}}' \
  > /aos/agents/researcher/inbox
```

### Method A: CLI (Recommended)

```bash
# All parameters in one atomic command
aos invoke researcher \
  --model=claude-opus-4-20250514 \
  --context=/aos/conversations/2025/01/04/abc123/context.ctx \
  --max-cost=2.00 \
  --timeout=600 \
  "Continue the analysis"

# Advantages:
# - Single atomic operation
# - Proper error handling
# - Shell completion for flags
# - No race conditions possible
```

### Method B: Atomic JSON Inbox

```bash
# JSON message includes all parameters
cat > /tmp/request.json << 'EOF'
{
  "query": "Continue the analysis",
  "context": "/aos/conversations/2025/01/04/abc123/context.ctx",
  "override": {
    "model": "claude-opus-4-20250514",
    "max_cost_usd": 2.00,
    "timeout_sec": 600
  },
  "idempotency_key": "req-abc123"
}
EOF

cat /tmp/request.json > /aos/agents/researcher/inbox
```

### Method C: Session-Scoped Overrides (Filesystem)

For multi-turn conversations where you want to set overrides once:

```bash
# Create a session with overrides
aos session create researcher \
  --model=claude-opus-4-20250514 \
  --max-cost=5.00
# Created session s_049

# Session overrides persist
echo "First query" > /aos/sessions/s_049/inbox
echo "Follow-up" > /aos/sessions/s_049/inbox
# Both use the session's override settings

# Session directory
/aos/sessions/s_049/
├── config.yaml         # Frozen at session creation
├── inbox               # Write to invoke
├── output              # Read results
└── status
```

### Decision Tree

```
Need to invoke agent?
│
├── Simple query, no overrides?
│   └── echo "query" > /aos/agents/X/inbox  ✓
│
├── Query with overrides?
│   └── aos invoke X --model=opus "query"   ✓
│
├── Query with context binding?
│   └── aos invoke X --context=/path "query" ✓
│
├── Multi-turn with consistent settings?
│   └── aos session create X --model=opus
│       then: echo "query" > /aos/sessions/s_N/inbox
│
└── Atomic filesystem alternative?
    └── echo '{"query":"...","override":{...}}' > inbox
```

### Deprecating Override Directory

The current spec's override directory is **deprecated** in favor of CLI/JSON:

```bash
# DEPRECATED (race-prone):
echo "opus" > /aos/agents/researcher/override/model
echo "query" > /aos/agents/researcher/inbox

# RECOMMENDED:
aos invoke researcher --model=opus "query"
```

The `override/` directory can remain for backwards compatibility but documentation should steer users to CLI.

### Error Handling

| Scenario | Error | Exit Code |
|----------|-------|-----------|
| Invalid JSON in inbox | `EINVAL: Invalid JSON message` | 1 |
| Unknown override field | `EINVAL: Unknown override 'foo'` | 1 |
| Context not found | `ENOENT: Context file not found` | 1 |
| Override violates limits | `EPERM: Override exceeds agent limits` | 1 |

### Trade-offs

**Costs:**
- CLI dependency for complex operations
- JSON syntax less intuitive than plain text
- Deprecation of override directory

**Benefits:**
- No race conditions
- Clear, documented behavior
- Single source of truth per invocation

**Rejected Alternatives:**
- **Per-PID override state**: Complex to implement, debugging nightmare
- **File locking**: Doesn't work across processes, FUSE complications
- **Separate override+query file**: Still two operations, still raceable

### Dependencies

- **Q5:** JSON inbox uses same atomic commit semantics
- **Q7a:** Idempotency key supported in JSON message

---

## Q7b: Cost Visibility Before Execution

### Solution

**Mandatory cost estimation** via `--dry-run` flag and **threshold warnings** for expensive operations:

1. **All costly commands support `--dry-run`**: Shows estimated cost without executing
2. **Automatic warning at threshold**: Operations > $1.00 prompt for confirmation (unless `--yes`)
3. **Cost file for filesystem operations**: Check before executing

### Commands with Cost Estimation

```bash
# Dry-run shows cost without executing
aos fork /aos/conversations/large/ --prompt="variant" --dry-run
# Estimated cost: $0.60
#   Context: 200,000 tokens
#   Backend: anthropic (no KV-cache sharing)
#   Operation: text serialization + inference
# Run without --dry-run to execute

aos invoke researcher --model=opus --dry-run "Complex query"
# Estimated cost: $0.15 - $0.45
#   Model: claude-opus-4-20250514
#   Est. input tokens: 5,000
#   Est. output tokens: 2,000 - 6,000
# Run without --dry-run to execute

aos bind /aos/procs/1234 /aos/agents/writer --dry-run
# Estimated cost: $0.00
#   Backend: vLLM (KV-cache pointer sharing)
#   Operation: O(1) context bind
```

### Threshold Warnings

```bash
# Operations above threshold prompt for confirmation
aos fork /aos/conversations/huge/ --prompt="variant"
# WARNING: Estimated cost $2.34 exceeds threshold ($1.00)
# Context: 500,000 tokens × text serialization
#
# Proceed? [y/N] n
# Aborted.

# Skip confirmation with --yes
aos fork /aos/conversations/huge/ --prompt="variant" --yes
# Proceeding with estimated cost $2.34...

# Or set budget cap
aos fork /aos/conversations/huge/ --prompt="variant" --max-cost=3.00
# Proceeding (within budget $3.00)...
```

### Filesystem Cost Visibility

```bash
# Before reading costly directory
cat /aos/mnt/github/.aos/cost_estimate
# {
#   "operation": "full_read",
#   "objects": 15743,
#   "estimated_usd": 47.23,
#   "warning": "Consider using --filter or limiting scope"
# }

# Cost for specific operation
cat /aos/mnt/github/issues/123/.aos/cost_estimate
# {"operation": "read", "estimated_usd": 0.003}

# Current accumulated cost
cat /aos/mnt/github/.aos/cost
# {"session_usd": 0.47, "today_usd": 2.31}
```

### Configuration

```yaml
# /aos/etc/daemon.yaml
cost_visibility:
  warning_threshold_usd: 1.00
  interactive_confirm: true        # Prompt above threshold
  show_estimates: true             # Always show estimates in dry-run

  # Estimation accuracy disclaimer
  disclaimer: |
    Estimates based on current pricing and token counts.
    Actual cost may vary ±20%.
```

### Error Handling

| Scenario | Result |
|----------|--------|
| `--dry-run` | Shows estimate, exit 0 |
| Above threshold, interactive | Prompts, user declines → exit 1 |
| Above threshold, `--yes` | Proceeds, logs warning |
| Above `--max-cost` | Refuses to execute, exit 1 |
| Non-interactive, above threshold | Warning to stderr, proceeds |

### Trade-offs

**Costs:**
- Estimation adds latency (token counting)
- Estimates can be inaccurate (±20%)
- Extra flag for non-interactive scripts

**Benefits:**
- No surprise bills
- Users understand cost before committing
- Clear decision points

---

## Q7c: Queue Depth and Backpressure

### Solution

**Bounded queues** with **visible depth** and **EAGAIN on full**:

1. **Default queue limit**: 100 messages per agent
2. **Depth visibility**: `/aos/agents/X/inbox.depth`
3. **Backpressure**: Return `EAGAIN` when full, client retries
4. **Priority lanes**: Optional high-priority queue

### File Layout

```
/aos/agents/researcher/
├── inbox                   # Write messages here
├── inbox.depth             # R: Current queue depth (e.g., "7")
├── inbox.limit             # R: Queue limit (e.g., "100")
├── inbox.peek              # R: View queued items without dequeuing
└── inbox.priority          # W: High-priority queue (optional)
```

### Commands

```bash
# Check queue before submitting
depth=$(cat /aos/agents/researcher/inbox.depth)
limit=$(cat /aos/agents/researcher/inbox.limit)
echo "Queue: $depth/$limit"
# Queue: 7/100

# Submit (succeeds if queue has room)
echo "task" > /aos/agents/researcher/inbox

# Queue full - returns EAGAIN
echo "task" > /aos/agents/researcher/inbox
# bash: echo: write error: Resource temporarily unavailable
# Exit code: 1 (errno EAGAIN)

# Peek at queue
cat /aos/agents/researcher/inbox.peek
# [
#   {"position":0,"content":"first query","submitted":"2025-01-05T10:00:00Z"},
#   {"position":1,"content":"second query","submitted":"2025-01-05T10:00:05Z"}
# ]

# Submit with retry logic
for i in {1..5}; do
  if echo "task" > /aos/agents/researcher/inbox 2>/dev/null; then
    break
  fi
  sleep $((i * 2))
done
```

### Priority Queue

```bash
# High-priority bypasses normal queue (limited slots)
echo "urgent task" > /aos/agents/researcher/inbox.priority

# Priority queue has smaller limit (default: 10)
cat /aos/agents/researcher/inbox.priority.limit
# 10
```

### CLI Support

```bash
# Submit with blocking wait for queue space
aos invoke researcher --wait-queue "task"
# Blocks until queue has room, then submits

# Submit with queue check
aos invoke researcher --queue-check "task"
# Checks queue depth first, fails fast if full

# Bulk submit with backpressure handling
aos invoke researcher --batch tasks.txt --rate-limit=10/s
```

### Configuration

```yaml
# /aos/agents/researcher/config.yaml
spec:
  queue:
    limit: 100
    priority_limit: 10
    overflow_action: eagain    # eagain | block | drop_oldest
```

### Error Handling

| Scenario | Error | Exit Code |
|----------|-------|-----------|
| Queue has room | Success | 0 |
| Queue full | `EAGAIN: Queue full` | 1 |
| Priority queue full | `EAGAIN: Priority queue full` | 1 |

### Trade-offs

**Costs:**
- Clients must handle EAGAIN
- Queue management adds complexity

**Benefits:**
- Predictable backpressure
- No unbounded memory growth
- Visible queue state

---

## Q3: Destructive Operations on Mounts

### Solution

**Read-only by default** with **explicit writable opt-in** and **soft-delete for destructive operations**:

```yaml
# Default mount (read-only)
aos mount github --token=$TOKEN /aos/mnt/github
# All writes return EROFS

# Writable mount (explicit opt-in)
aos mount github --token=$TOKEN --writable /aos/mnt/github
# Writes allowed

# With soft-delete protection
aos mount github --token=$TOKEN --writable --soft-delete /aos/mnt/github
# rm moves to .trash/, actual delete requires aos mount gc
```

### File Layout with Soft-Delete

```
/aos/mnt/github/
├── issues/
│   ├── 123/
│   └── 124/
├── .trash/                 # Soft-deleted items
│   └── issues/
│       └── 125/            # rm'd but not yet deleted
│           ├── body
│           └── .deleted_at # Timestamp
└── .aos/
    ├── mode                # "readonly" | "writable"
    └── trash_policy        # Retention before hard delete
```

### Commands

```bash
# Read-only mount (default) - rm fails
rm /aos/mnt/github/issues/123
# rm: cannot remove '/aos/mnt/github/issues/123': Read-only file system

# Writable mount with soft-delete
rm /aos/mnt/github/issues/123
# Moved to .trash/ (not actually deleted)

# View trash
ls /aos/mnt/github/.trash/issues/
# 123

# Restore from trash
mv /aos/mnt/github/.trash/issues/123 /aos/mnt/github/issues/

# Permanently delete (requires explicit gc)
aos mount gc /aos/mnt/github --older-than=7d
# Permanently deleting 5 items from .trash/
# Proceed? [y/N]
```

### Trade-offs

- Read-only default is safe but adds friction
- Soft-delete adds complexity but prevents accidents

---

## Q7: Exit Code Semantics

### Solution

**Reorganize exit codes** with clear categories:

| Range | Category | Core Dump |
|-------|----------|-----------|
| 0 | Success | No |
| 1-63 | User/input errors | No |
| 64-95 | Resource limits (routine) | No |
| 96-127 | System errors (exceptional) | Yes |

### Revised Exit Codes

| Code | Name | Category | Core Dump |
|------|------|----------|-----------|
| 0 | SUCCESS | Success | No |
| 1 | FAILURE | General error | No |
| 2 | INVALID_INPUT | Bad request | No |
| 64 | BUDGET_EXHAUSTED | Resource | No |
| 65 | RATE_LIMITED | Resource | No |
| 66 | TIMEOUT | Resource | No |
| 67 | QUEUE_FULL | Resource | No |
| 96 | REFUSED | Security violation | Yes |
| 97 | CTX_OVERFLOW | System error | Yes |
| 98 | UPSTREAM_FAILURE | System error | Yes |
| 99 | CONTAMINATED | Security violation | Yes |
| 100 | LEASE_EXPIRED | Security | Yes |
| 101 | LOW_CONFIDENCE_REFUSED | Refused to answer | Yes |
| 102 | LOW_CONFIDENCE_UNCERTAIN | Answered but uncertain | No |

### Split Exit 68

Original exit 68 is split:
- **101 LOW_CONFIDENCE_REFUSED**: Agent declined to answer (exceptional)
- **102 LOW_CONFIDENCE_UNCERTAIN**: Agent answered but flagged uncertainty (routine)

---

## Medium Priority: Documentation Notes

### Q7a: Inbox Idempotency

**Solution:** Support optional message ID prefix:
```bash
echo "id:abc123 analyze this" > inbox
# Duplicate writes with same ID are deduplicated within 5-minute window
```

### Q7d: Testing Story

**Recommended approach:** Backend configuration points to mock server:
```yaml
# /aos/etc/providers.yaml (test mode)
providers:
  mock:
    type: mock
    responses_file: /path/to/canned_responses.jsonl
```

Full testing story deferred to v1.1.

### Q7e: Stateful Directories (Pagination)

**Solution:** Remove cursor files from filesystem, pagination only via CLI:
```bash
aos list /aos/mnt/github/issues --page=2 --per-page=100
```

Directories remain stateless.

### Q7f: Session vs Conversation

**Clarification:**
- **Session**: Multi-turn container with persistent context
- **Conversation**: Single invocation record (archived)
- Relationship: Session contains multiple turns, each turn generates a conversation record

### Q8-Q15: Summary

| Question | Resolution |
|----------|------------|
| Q8 Contamination | Document: per-agent tracking, not byte-level |
| Q9 FS vs CLI | Document: CLI for complex, FS for simple |
| Q10 Async feedback | `aos invoke --wait` handles polling |
| Q10a Cleanup | `aos gc --now` for immediate cleanup |
| Q11 Ring model | Documentation clarification |
| Q12 Blocking reads | Implementation guide, not spec |
| Q13 Replay | Rename to `tool_deterministic` |
| Q14 st_size | Report accurate size, use EACCES for protection |
| Q15 Blocking files | Use FIFOs for streams, files for last_response |

---

## Implementation Priority

Based on cross-reviewer agreement:

1. **Immediate (blocks v1):**
   - Q1: Agent config privilege (file permissions)
   - Q2: Intent approval (UID separation)
   - Q5: Inbox write semantics (atomic buffer)

2. **High (needed for safety):**
   - Q4: Circuit breaker (multi-layer)
   - Q6: Override races (CLI canonical)

3. **Medium (needed for usability):**
   - Q7b: Cost visibility (--dry-run)
   - Q7c: Queue depth (bounded + EAGAIN)
   - Q3: Mount safety (read-only default)

4. **Documentation:**
   - Q7-Q15: Clarifications and future work markers

---

*Solutions proposed by Claude (Opus 4.5) — 2025-01-05*
