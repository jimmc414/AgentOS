# Agent OS Specification v2.2

**Status:** Draft Standard
**Lineage:** v2.1 + Feasibility Audit + Gap Analysis + External Review
**Changes from v2.1:**

| Addition | Section | Status |
|----------|---------|--------|
| Anti-Bankruptcy Protocols | §2.3 | Merged |
| Atomic Write Handling | §2.4 | Merged |
| Executable Agents (Shebang) | §3.10 | Merged |
| Man Pages | §11.6 | Merged |
| Core Dumps | §4.7, §13.6 | Merged |
| User Isolation | §12.8 | Merged |
| Context Block Visualization | §5.7 | Revised (was "virtual symlinks") |
| Interactive Steering | Appendix D | Deferred (needs design work) |

---

## 1. Overview

### 1.1 Thesis

Agent OS is a Unix-native runtime for LLM agents where **everything is a file**.

LLMs are treated as processes. Their I/O is file-like (stdin/stdout/stderr streams). Their memory is managed (context as address space). Their permissions are enforced (capabilities). Their lifecycle is controlled (signals). The kernel routes tokens; it does not interpret them.

The primary interface is a FUSE filesystem. AI agents interact with other agents, memory, and external services using filesystem operations embedded in their training data: `cat`, `echo`, `grep`, `ls`, `tail -f`.

### 1.2 Design Principles

1. **Everything is a file.** Not file-like. Actual files that work with actual Unix tools.

2. **AI agents are the primary user.** The interface leverages operations embedded in LLM training data.

3. **Pipes are dumb.** Transformation is a process, not a pipe property. (KV-cache pointer sharing uses explicit binding, not transparent pipe interception.)

4. **Explicit over implicit.** Mounts have documented semantics. Costs are visible. Operations are predictable.

5. **Observable by default.** Every operation has a trace. Every state has a file.

6. **Capabilities intersect down.** A caller cannot escalate privileges through an agent.

7. **Compression is an event, not a daemon.** Agents know when memory is lossy.

8. **Cost is a first-class resource.** Budget limits are as fundamental as memory limits.

9. **Pointers over copies.** Context sharing via KV-cache when available (through explicit binding).

10. **Fail explicit.** Errors identify which component failed and why. Crashes generate core dumps.

11. **Conversations are assets.** Searchable, resumable, forkable, version-controllable.

12. **Human approval is I/O.** Write to request, read to receive. Channels, not special APIs.

13. **Protective defaults.** The OS prevents bankruptcy by detecting and blocking aggressive non-agent readers.

### 1.3 Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      FUSE Filesystem                         │
│                        /aos/*                                │
│  Agents, Processes, Conversations, Memory, Channels, Mounts │
└─────────────────────────────┬───────────────────────────────┘
                              │ read/write/poll
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Agent Daemon                            │
│                                                              │
│  Process Management    │  Security Enforcement               │
│  Context Management    │  Budget Tracking                    │
│  Execution Traces      │  Backend Abstraction                │
└─────────────────────────────┬───────────────────────────────┘
                              │ inference calls
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Model Backends                           │
│   vLLM  │  SGLang  │  llama.cpp  │  OpenAI  │  Anthropic    │
└─────────────────────────────────────────────────────────────┘
```

**FUSE Layer:** Exposes state as files. Translates filesystem operations to daemon calls.

**Daemon Layer:** Manages execution, enforces security, tracks resources. Implements process model.

**Backend Layer:** Abstracts model providers. Handles inference, KV-cache operations.

### 1.4 Ring Model

| Ring | Name | Responsibilities | Interprets Text |
|------|------|------------------|-----------------|
| 0 | Daemon | Token routing, capability enforcement, signal delivery, resource accounting | No |
| 1 | FUSE + Tools | Filesystem interface, format conversion, policy decisions | Yes |
| 2 | Agent Processes | LLM inference, tool execution, reasoning | Yes |

Ring 0 enforces security labels; it does not decide which content deserves which label. Ring 1 (FUSE layer and userland tools) presents content to humans and sets labels based on policy.

### 1.5 Backend Compatibility

| Backend | FUSE Operations | KV-Cache Fork | Shared Prefixes | Context Bind |
|---------|----------------|---------------|-----------------|--------------|
| OpenAI API | Full | No | Prompt caching | Text serialization |
| Anthropic API | Full | No | Prompt caching | Text serialization |
| Google Gemini | Full | Partial | Context caching | Text serialization |
| vLLM | Full | Yes | Yes | O(1) pointer |
| SGLang | Full | Yes | Yes | O(1) pointer |
| llama.cpp | Full | Yes | Yes | O(1) pointer |

When KV-cache operations are unavailable, the daemon falls back to text serialization. Semantics are preserved; performance is degraded.

---

## 2. Filesystem Structure

### 2.1 Top-Level Layout

```
/aos/
├── agents/              # Agent definitions and runtime state
├── procs/               # Running processes
├── conversations/       # Complete interaction history
├── memory/              # Persistent knowledge store
├── channels/            # I/O destinations (humans, services)
├── mnt/                 # External resource mounts (APIs, databases)
├── models/              # Available model backends
├── tools/               # Available tools
├── man/                 # Auto-generated agent documentation
├── cores/               # Symlinks to recent crash dumps
├── system/              # Runtime status and global configuration
└── etc/                 # Configuration files
```

### 2.2 File Semantic Classes

Files behave differently based on their purpose:

| Class | Read Behavior | Write Behavior | Examples |
|-------|--------------|----------------|----------|
| Instant | Returns immediately with cached/current value | Updates value | `status`, `cost`, `output` |
| Streaming | Blocks until new data available | N/A | `stream` |
| Async | N/A | Queues operation, returns immediately | `inbox` |
| Blocking | Blocks until response available | Initiates request | `channels/human/*` |
| Control | Returns current state | Triggers action | `ctl`, `session` |

**mtime as Freshness Signal:**

FUSE controls file metadata. `mtime` indicates when content last changed:

```bash
# Check if output updated since last read
stat /aos/agents/researcher/output
# Modified: 2025-01-04 14:23:07

# Watch for changes
inotifywait -m /aos/agents/researcher/output
```

### 2.3 Anti-Bankruptcy Protocols

Operating systems run aggressive background indexers (Spotlight on macOS, Windows Search, `updatedb` on Linux, VS Code file watchers). Without protection, these could trigger catastrophic API costs by scanning the FUSE mount.

**Protection Mechanisms:**

**1. Process Blocklist**

The daemon checks the `comm` (command name) and `pid` of every `open()` call. Known indexers receive `EACCES` or empty directory listings:

```yaml
# /aos/etc/security/indexer_blocklist.yaml
blocked_processes:
  - mds           # macOS Spotlight
  - mdworker      # macOS Spotlight worker
  - mds_stores    # macOS Spotlight stores
  - updatedb      # Linux locate database
  - locate        # Linux locate
  - fwupdmgr      # Firmware updater
  - tracker-*     # GNOME Tracker
  - baloo_*       # KDE Baloo
```

**2. Virtual Size Reporting**

Files in `/aos/mnt/` and `/aos/conversations/` report `st_size=0` until explicitly opened by an allowed process. This discourages eager scanners that check file size before reading.

```bash
ls -l /aos/mnt/github/issues/
# -rw-r--r-- 1 aos aos 0 Jan  4 10:00 123
# -rw-r--r-- 1 aos aos 0 Jan  4 10:00 124

# Actual size revealed on open by allowed process
cat /aos/mnt/github/issues/123 | wc -c
# 4521
```

**3. Cost Circuit Breaker**

Per-PID cost tracking with automatic cutoff:

```yaml
# /aos/etc/security/circuit_breaker.yaml
circuit_breaker:
  enabled: true
  threshold_usd: 0.10      # Per PID
  window_sec: 60           # Time window
  action: block            # block | warn | log
  cooldown_sec: 300        # How long to block
```

If a non-agent PID triggers more than $0.10 of API calls in 60 seconds, the daemon blocks read access for that PID and logs a warning:

```bash
cat /aos/system/circuit_breaker/blocked
# {"pid":12345,"comm":"mdworker","cost_usd":0.12,"blocked_at":"...","expires":"..."}
```

**4. Directory Markers**

The mount point includes standard ignore markers:

```
/aos/
├── .gitignore           # Contains: *
├── .ignore              # For ripgrep, ag, etc.
├── .noindex             # macOS Spotlight hint
└── CACHEDIR.TAG         # Standard cache directory marker
```

The `CACHEDIR.TAG` file contains the required signature:
```
Signature: 8a477f597d28d172789f06886806bc55
# This file marks the directory as a cache directory for backup tools.
```

### 2.4 Atomic Write Handling

Editors (vim, nano, VS Code) use atomic save patterns: write to temp file, delete original, rename temp. The daemon handles this via inotify event interpretation:

**Watched Events:**

| Event | Interpretation |
|-------|----------------|
| `IN_CLOSE_WRITE` | Direct write completed |
| `IN_MOVED_TO` | Atomic rename completed (use this content) |
| `IN_DELETE` followed by `IN_MOVED_TO` | Atomic replace pattern |

**Debouncing:**

Configuration file changes are debounced with a 100ms window. Multiple rapid events are collapsed to a single config reload.

**Ignored Patterns:**

```yaml
# /aos/etc/daemon.yaml
fuse:
  ignored_patterns:
    - "*.swp"       # vim swap
    - "*.swo"       # vim swap overflow
    - "*~"          # Backup files
    - ".*.swp"      # Hidden vim swap
    - "4913"        # vim test file
    - ".directory"  # KDE
```

### 2.5 Common File Patterns

**Status files:** Plain text, single value
```bash
cat /aos/agents/researcher/status
# idle
```

**Cost files:** Plain text, numeric
```bash
cat /aos/agents/researcher/cost
# 0.47
```

**Config files:** YAML or JSON (supports environment variable interpolation)
```yaml
# ${VAR} expands to environment variable
# ${VAR:-default} provides default if unset
credentials: ${API_TOKEN}
timeout: ${TIMEOUT:-300}
```

**Log files:** Append-only JSONL
```bash
tail -f /aos/procs/1234/stderr
# {"type":"tool_call","tool":"web.search",...}
```

**Stream files:** Blocking read, live data
```bash
tail -f /aos/agents/researcher/stream
# {"type":"token","content":"The"}
# {"type":"token","content":" fusion"}
```

---

## 3. Agents

### 3.1 Agent Directory Structure

```
/aos/agents/
├── researcher/
│   ├── config.yaml          # R/W: Agent definition
│   ├── status               # R: idle | running | error
│   ├── cost                 # R: Current session spend (USD)
│   ├── inbox                # W: Send message (async, queues work)
│   ├── output               # R: Last completed response (instant)
│   ├── stream               # R: Live token stream (tail -f)
│   ├── session              # R/W: Current session ID
│   ├── context              # W: Set context for next invocation
│   ├── log                  # R: Append-only interaction history
│   └── current/             # R: Current interaction details
│       ├── input            # The prompt being processed
│       ├── output           # Output so far
│       ├── trace.jsonl      # Tool calls, decisions
│       └── meta.json        # Timing, token counts
│
├── coder/
│   └── ...
│
└── .templates/              # Agent templates (copy to create new)
    ├── researcher.yaml
    └── assistant.yaml
```

### 3.2 Agent Configuration

```yaml
# /aos/agents/researcher/config.yaml
apiVersion: agent/v1
kind: Agent

metadata:
  name: researcher
  description: Research assistant for technical analysis
  version: 1.0.0

spec:
  # Model selection
  model: claude-sonnet-4-5-20250514
  fallback: claude-haiku-3-5-20241022
  
  # System prompt
  persona: |
    You are a research assistant focused on technical analysis.
    
    ## Approach
    - Search for primary sources first
    - Cross-reference claims across multiple sources
    - Flag uncertainty explicitly with confidence levels
    - Cite sources inline
    
    ## Output Format
    Respond in structured markdown with citations.
  
  # Few-shot examples (optional)
  examples:
    - user: "What is the current state of fusion energy?"
      assistant: |
        ## Summary
        Fusion energy achieved net energy gain in December 2022...
  
  # Capabilities granted to this agent
  capabilities:
    tools: [web.search, web.fetch, fs.read]
    spawn: true
    spawn_limit: 5
    memory:
      read: ["/aos/memory/**"]
      write: ["/aos/memory/scratch/**"]
  
  # Resource limits
  limits:
    max_cost_usd: 1.00
    tokens_total: 100000
    context_tokens: 200000
    max_tool_calls: 50
    timeout_sec: 300
  
  # Output configuration
  output:
    format: text              # text | jsonl
    include_trace: false      # Include tool calls in output
    confidence_threshold: 0.7 # Exit 68 if below
  
  # Binary input handling
  binary:
    accept: [image/png, image/jpeg, application/pdf]
    encoding: base64          # base64 | multimodal (if model supports)
```

### 3.3 Agent Invocation

**Basic invocation:**
```bash
echo "What is the current state of fusion energy?" > /aos/agents/researcher/inbox
```

**Check status:**
```bash
cat /aos/agents/researcher/status
# running
```

**Stream response:**
```bash
tail -f /aos/agents/researcher/stream
```

**Get completed response:**
```bash
cat /aos/agents/researcher/output
```

**Check cost:**
```bash
cat /aos/agents/researcher/cost
# 0.34
```

**Wait for completion:**
```bash
# Using aos wait (recommended)
aos wait /aos/agents/researcher
echo "Agent completed with exit code: $?"

# Using inotifywait
inotifywait -e modify /aos/agents/researcher/status
while [ "$(cat /aos/agents/researcher/status)" = "running" ]; do
  sleep 0.1
done

# Using poll loop
while [ "$(cat /aos/agents/researcher/status)" = "running" ]; do
  sleep 1
done
```

### 3.4 Invocation Overrides

Override agent configuration for a single invocation by writing to control files before inbox:

```bash
# Override model
echo "claude-opus-4-20250514" > /aos/agents/researcher/override/model

# Override budget
echo "5.00" > /aos/agents/researcher/override/max_cost_usd

# Override timeout
echo "600" > /aos/agents/researcher/override/timeout_sec

# Now invoke (overrides apply to this invocation only)
echo "Complex research task" > /aos/agents/researcher/inbox

# Overrides are cleared after invocation completes
```

Or use `aos invoke` for atomic override + invocation:

```bash
aos invoke researcher \
  --model=claude-opus-4-20250514 \
  --max-cost=5.00 \
  --timeout=600 \
  "Complex research task"
```

### 3.5 Context Setting

Set context for the next invocation by writing to the `context` file:

```bash
# Set context from a previous conversation
echo "/aos/conversations/2025/01/04/a3f8c2/context.ctx" \
  > /aos/agents/researcher/context

# Or from a running process
echo "/aos/procs/1234/context" > /aos/agents/researcher/context

# Now invoke (uses the specified context)
echo "Continue the analysis" > /aos/agents/researcher/inbox

# Context reference is cleared after invocation
```

The context file accepts:
- Path to a `.ctx` file (serialized context)
- Path to a process context directory
- Literal context content (JSONL format)

### 3.6 Session Management

Each agent maintains a session for multi-turn conversations:

```bash
# Current session
cat /aos/agents/researcher/session
# s_047

# Continue conversation (implicit)
echo "Tell me more about NIF specifically" > /aos/agents/researcher/inbox

# Start new session (explicit)
echo "new" > /aos/agents/researcher/session
cat /aos/agents/researcher/session
# s_048

# Switch to existing session
echo "s_047" > /aos/agents/researcher/session
```

Session data persists in conversations directory:
```
/aos/conversations/active/s_047 -> /aos/conversations/2025/01/04/a3f8c2/
```

### 3.7 Creating and Deleting Agents

**Create from template:**
```bash
# Copy from template
cp /aos/agents/.templates/researcher.yaml /aos/agents/myagent/config.yaml

# Edit configuration
vim /aos/agents/myagent/config.yaml

# Agent is immediately available
echo "Hello" > /aos/agents/myagent/inbox
```

**Create from scratch:**
```bash
# Create directory
mkdir /aos/agents/myagent

# Create minimal config
cat > /aos/agents/myagent/config.yaml << 'EOF'
apiVersion: agent/v1
kind: Agent
metadata:
  name: myagent
spec:
  model: claude-sonnet-4-5-20250514
  persona: "You are a helpful assistant."
  capabilities:
    tools: []
  limits:
    max_cost_usd: 0.50
EOF
```

**Delete agent:**
```bash
# Remove agent (fails if running)
rm -r /aos/agents/myagent

# Force remove (kills running processes)
rm -rf /aos/agents/myagent

# Archive instead of delete
mv /aos/agents/myagent /aos/agents/.archive/myagent-$(date +%Y%m%d)
```

### 3.8 Binary Input Handling

Agents can accept binary files (images, PDFs) if configured:

```bash
# Check if agent accepts binary
cat /aos/agents/vision/config.yaml | grep -A3 binary
# binary:
#   accept: [image/png, image/jpeg]
#   encoding: multimodal

# Send image
cat image.png > /aos/agents/vision/inbox.bin

# Or with prompt
cat > /aos/agents/vision/inbox.multipart << 'EOF'
--boundary
Content-Type: text/plain

Describe this image in detail.
--boundary
Content-Type: image/png
Content-Transfer-Encoding: base64

EOF
base64 image.png >> /aos/agents/vision/inbox.multipart
echo -e "\n--boundary--" >> /aos/agents/vision/inbox.multipart
```

For simpler usage, use `aos invoke`:
```bash
aos invoke vision --attach=image.png "Describe this image"
```

### 3.9 Compiled Agents (.aexe)

For frequently-used agents, precompute KV-cache for instant startup:

```
/aos/agents/researcher.aexe/
├── manifest.yaml          # Metadata, model compatibility
├── source/
│   └── config.yaml        # Original agent definition
└── cache/
    ├── claude-sonnet-4-5.kvcache    # Precomputed KV-cache
    └── llama-3-70b.kvcache
```

**Compile an agent:**
```bash
aos compile /aos/agents/researcher -o /aos/agents/researcher.aexe

# Compile for specific models
aos compile /aos/agents/researcher \
  --models=claude-sonnet-4-5,llama-3-70b \
  -o /aos/agents/researcher.aexe
```

**Use compiled agent:**
```bash
# Same interface, faster startup (~10ms vs ~500ms)
echo "Query" > /aos/agents/researcher.aexe/inbox
```

**Cache invalidation:**
```bash
# Check if cache is stale
cat /aos/agents/researcher.aexe/manifest.yaml | grep source_hash
# source_hash: abc123...

# Recompile if source changed
aos compile --refresh /aos/agents/researcher.aexe
```

### 3.10 Executable Agents (Shebang)

Agents can be defined as single executable files using the `aos` interpreter. This allows agents to be versioned in Git and executed like standard scripts.

**File format:**

```bash
#!/usr/bin/env aos
# @model: claude-sonnet-4-5-20250514
# @budget: 5.00
# @tools: [fs.read, web.search]
# @timeout: 300

You are a security auditor. Analyze the code provided on stdin 
and identify potential vulnerabilities.

Output format: Markdown with severity ratings.
```

**Header directives:**

| Directive | Required | Description |
|-----------|----------|-------------|
| `@model` | No | Model to use (default: system default) |
| `@budget` | No | Max cost in USD (default: 1.00) |
| `@tools` | No | List of allowed tools (default: none) |
| `@timeout` | No | Timeout in seconds (default: 300) |
| `@context` | No | Path to initial context file |

**Usage:**

```bash
# Make executable
chmod +x audit.aos

# Run with stdin
cat source.py | ./audit.aos > report.md

# Run with arguments (become the prompt)
./audit.aos "Review this for SQL injection" < query.py

# Pipe between agents
cat data.csv | ./analyze.aos | ./visualize.aos > output.html
```

**Implementation:**

The `aos` binary:
1. Parses shebang header directives
2. Creates a transient agent process
3. Connects stdin/stdout/stderr
4. Waits for completion
5. Exits with agent's exit code

**Example workflow:**

```bash
# Create agent script
cat > summarize.aos << 'EOF'
#!/usr/bin/env aos
# @model: claude-haiku-3-5-20241022
# @budget: 0.10

Summarize the following text in 3 bullet points.
EOF

chmod +x summarize.aos

# Use it
curl -s https://example.com/article | ./summarize.aos
```

---

## 4. Processes

### 4.1 Process Directory Structure

Running agent invocations appear in `/aos/procs/`:

```
/aos/procs/
├── 1234/
│   ├── status               # R: RUNNING | STOPPED | ZOMBIE | PAUSED
│   ├── agent                # R: Agent name (researcher)
│   ├── pid                  # R: Process ID
│   ├── ppid                 # R: Parent process ID (0 if root)
│   ├── started              # R: Start timestamp (ISO 8601)
│   ├── exit                 # R: Exit info (appears on completion)
│   │
│   ├── stdin                # W: Send additional input
│   ├── stdout               # R: Output stream
│   ├── stderr               # R: Execution trace (JSONL)
│   │
│   ├── ctl                  # W: Control commands
│   ├── capabilities         # R: Effective capabilities (JSON)
│   ├── color                # R: Security color (BLUE|YELLOW|RED)
│   │
│   ├── budget/
│   │   ├── limit            # R: Budget limit (USD)
│   │   ├── spent            # R: Current spend (USD)
│   │   ├── tokens_limit     # R: Token limit
│   │   ├── tokens_used      # R: Tokens consumed
│   │   └── velocity         # R: Current spend rate (USD/min)
│   │
│   ├── context/
│   │   ├── tokens           # R: Current context size
│   │   ├── limit            # R: Context limit
│   │   ├── pressure         # R: Exists when >80% full
│   │   ├── parent           # R: Parent context reference
│   │   ├── blocks/          # R: Context block metadata
│   │   ├── checkpoints/     # R: Available checkpoints
│   │   │   ├── cp_001
│   │   │   └── cp_002
│   │   └── compressions.log # R: Compression history
│   │
│   ├── leases/              # R: Capability leases
│   │   ├── fs.write.json
│   │   └── web.search.json
│   │
│   ├── intents/             # R/W: Pending side effects
│   │   ├── pending/
│   │   │   └── 001.json
│   │   └── completed/
│   │       └── 000.json
│   │
│   ├── core/                # R: Exists only after crash (exit >= 64)
│   │
│   └── conversation -> /aos/conversations/2025/01/04/a3f8c2/
│
└── 1235/
    └── ...
```

### 4.2 Process Control

**Control file commands:**

| Command | Effect |
|---------|--------|
| `stop` | Graceful shutdown (SIGTERM equivalent) |
| `kill` | Immediate termination (SIGKILL equivalent) |
| `pause` | Pause generation (SIGSTOP equivalent) |
| `resume` | Resume generation (SIGCONT equivalent) |
| `checkpoint` | Create context checkpoint |

```bash
# Graceful stop
echo "stop" > /aos/procs/1234/ctl

# Force kill
echo "kill" > /aos/procs/1234/ctl

# Pause and resume
echo "pause" > /aos/procs/1234/ctl
echo "resume" > /aos/procs/1234/ctl

# Create checkpoint
echo "checkpoint" > /aos/procs/1234/ctl
ls /aos/procs/1234/context/checkpoints/
# cp_001  cp_002  cp_003
```

### 4.3 Signals (Daemon Internal)

The daemon uses signals internally. FUSE exposes them via control files and status:

| Signal | Trigger | FUSE Manifestation |
|--------|---------|-------------------|
| SIGTERM | `echo stop > ctl` | status becomes "stopping" |
| SIGKILL | `echo kill > ctl` | Process directory removed |
| SIGSTOP | `echo pause > ctl` | status becomes "paused" |
| SIGCONT | `echo resume > ctl` | status becomes "running" |
| SIGPIPE | Downstream closed | status becomes "broken_pipe" |
| SIGCTXPRESSURE | Context > 80% | File appears: `context/pressure` |
| SIGXCPU | Budget exhausted | status becomes "budget_exceeded" |
| SIGBRAKE | Velocity exceeded | status becomes "throttled" |
| SIGCAPDROP | Capability revoked | File appears: `capability_revoked` |

### 4.4 Exit Codes

When a process completes, exit information is written before directory cleanup:

```bash
cat /aos/procs/1234/exit
# {"code": 0, "reason": "completed", "cost_usd": 0.47, "duration_sec": 45}
```

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

Exit codes >= 64 trigger core dump generation.

### 4.5 Process Listing

```bash
# List running processes
ls /aos/procs/
# 1234  1235  1236

# Watch all processes
watch 'for p in /aos/procs/*/; do 
  echo "$(basename $p): $(cat $p/status) - $(cat $p/agent)"
done'

# Find by agent
grep -l "researcher" /aos/procs/*/agent | xargs dirname

# Find by status
for p in /aos/procs/*/; do
  [ "$(cat $p/status)" = "running" ] && echo "$p"
done
```

### 4.6 Spawning Sub-Processes

An agent can spawn other agents if it has the `spawn` capability:

```bash
# From within agent execution, spawn is a tool call
# The daemon creates a child process with:
# - ppid = current process ID
# - capabilities = intersection of parent and child agent
# - color = min(parent color, child agent default)
```

Process hierarchy visible via:
```bash
# Find children of process 1234
grep -l "1234" /aos/procs/*/ppid | xargs dirname

# Build tree
aos ps --tree
# 1234 researcher
# ├── 1235 analyzer
# └── 1236 critic
#     └── 1237 fact_checker
```

### 4.7 Core Dumps

When a process exits abnormally (exit code >= 64), the daemon generates a core dump for debugging.

**Core dump location:**

```
/aos/procs/1234/core/              # During process cleanup window
/aos/cores/1234_20250104_103022/   # Persistent (symlinked)
```

**Core dump contents:**

```
/aos/procs/1234/core/
├── meta.json           # Process metadata at time of crash
├── context.hash        # Content-addressed context reference
├── context.tokens      # Token count at crash
├── trace.jsonl         # Last 100 trace events
├── environment.json    # Capabilities, budget, color
├── error.log           # Exception/error details
└── stack.md            # Human-readable "stack trace" (reasoning chain)
```

**meta.json:**
```json
{
  "pid": 1234,
  "agent": "researcher",
  "exit_code": 66,
  "exit_reason": "BUDGET_EXHAUSTED",
  "started": "2025-01-04T10:00:00Z",
  "crashed": "2025-01-04T10:30:22Z",
  "duration_sec": 1822
}
```

**environment.json:**
```json
{
  "capabilities": ["web.search", "fs.read"],
  "color": "YELLOW",
  "budget": {
    "limit_usd": 1.00,
    "spent_usd": 1.00,
    "tokens_limit": 100000,
    "tokens_used": 87432
  },
  "context": {
    "tokens": 145000,
    "limit": 200000,
    "checkpoints": ["cp_001", "cp_002"]
  }
}
```

**Debugging with core dump:**

```bash
# List recent crashes
ls -la /aos/cores/

# Load into debugger
aos debug --core /aos/cores/1234_20250104_103022/

(aos-dbg) meta
# PID: 1234, Agent: researcher, Exit: 66 (BUDGET_EXHAUSTED)

(aos-dbg) trace tail 10
# [Last 10 trace events]

(aos-dbg) context summary
# 145,000 tokens, 2 checkpoints available

(aos-dbg) context restore cp_002
# Restoring to checkpoint cp_002...
# Spawned process 1240 from checkpoint

(aos-dbg) quit
```

**Core dump retention:**

```yaml
# /aos/etc/daemon.yaml
core_dumps:
  enabled: true
  retention_days: 7
  max_count: 100
  max_size_mb: 1000
```

---

## 5. Context Management

### 5.1 Context Structure

Context is organized as a Merkle DAG of content-addressed blocks:

```
ContextHandle:
  root_hash: Hash256
  blocks: Map[Hash256, Block]
  metadata:
    created_at: Timestamp
    parent: ContextHandle?
    compression_history: [CompressionEvent]
```

Properties:
- Efficient deduplication across processes
- O(1) fork via pointer sharing (when KV-cache available)
- Verifiable integrity via hash chains
- Incremental synchronization for remote contexts

### 5.2 Context Files

```
/aos/procs/1234/context/
├── tokens               # Current token count
├── limit                # Token limit
├── pressure             # Exists when >80% full
├── parent               # Parent context reference (if bound)
├── blocks/              # Block metadata (read-only)
├── checkpoints/
│   ├── cp_001           # Checkpoint references
│   └── cp_002
└── compressions.log     # Compression history
```

### 5.3 Context Binding

When one agent's output becomes another's input context:

```bash
# Method 1: Set context file before invocation
echo "/aos/conversations/2025/01/04/a3f8c2/context.ctx" \
  > /aos/agents/writer/context
echo "Summarize the findings" > /aos/agents/writer/inbox

# Method 2: Use aos bind for daemon-level binding (more efficient)
aos bind /aos/procs/1234 /aos/agents/writer
echo "Summarize the findings" > /aos/agents/writer/inbox

# Method 3: Use aos invoke with --context
aos invoke writer \
  --context=/aos/conversations/2025/01/04/a3f8c2/context.ctx \
  "Summarize the findings"
```

**Performance:**
- With KV-cache (same backend, compatible model): O(1) pointer sharing
- Without KV-cache: O(n) text serialization

**Note on shell pipes:** Shell pipes (`|`) are kernel constructs that FUSE cannot intercept. To achieve KV-cache pointer sharing between agents, use explicit `aos bind` or `aos pipe`:

```bash
# This serializes to text (kernel pipe, FUSE not involved):
cat query | aos invoke researcher | aos invoke writer

# This uses KV-cache pointer sharing (daemon-level binding):
aos pipe researcher writer "query"
```

### 5.4 Checkpointing

```bash
# Create checkpoint
echo "checkpoint" > /aos/procs/1234/ctl

# List checkpoints
ls /aos/procs/1234/context/checkpoints/
# cp_001  cp_002  cp_003

# Checkpoint info
cat /aos/procs/1234/context/checkpoints/cp_002
# {"id":"cp_002","tokens":45000,"created":"2025-01-04T10:30:00Z","hash":"abc123"}

# Restore to checkpoint (creates new process)
aos restore /aos/procs/1234/context/checkpoints/cp_002
# Spawned process 1238 from checkpoint cp_002
```

### 5.5 Context Tools

**aos ctx pageout:** Page context to external storage
```bash
aos ctx pageout --proc=1234 --selector="age>1h" --to=vectorstore://project
# [PAGEOUT] Paged out 47 blocks (89,000 tokens)
# [PAGEOUT] Context replaced with retrieval stubs
```

**aos ctx pagein:** Retrieve paged content
```bash
aos ctx pagein --proc=1234 --query="fusion ignition timeline"
# [PAGEIN] Retrieved 3 blocks (12,000 tokens)
```

**aos ctx merge:** Combine multiple contexts
```bash
# Concatenate
aos ctx merge --mode=concat ctx_a ctx_b > ctx_merged

# Extract and reconcile beliefs
aos ctx merge --mode=extract-beliefs ctx_a ctx_b > ctx_merged

# Use judge for conflicts
aos ctx merge --mode=debate ctx_a ctx_b --judge=@arbiter > ctx_merged
```

### 5.6 Context Pressure

When context exceeds 80% of limit:

```bash
# Pressure indicator appears
ls /aos/procs/1234/context/
# tokens  limit  pressure  blocks/  checkpoints/

cat /aos/procs/1234/context/pressure
# {"usage_pct": 85, "tokens": 170000, "limit": 200000}

# Compression log shows history
cat /aos/procs/1234/context/compressions.log
# {"ts":"...","before":145000,"after":52000,"method":"lru_summarize"}
```

### 5.7 Context Block Visualization

To inspect what's in context without incurring inference costs:

```
/aos/procs/1234/context/blocks/
├── 001_system.json
├── 002_user.json
├── 003_assistant.json
├── 004_tool_call.json
├── 005_tool_result.json
└── ...
```

Each file contains block metadata (not the content itself):

```json
// /aos/procs/1234/context/blocks/003_assistant.json
{
  "index": 3,
  "type": "assistant",
  "hash": "sha256:abc123...",
  "tokens": 1247,
  "created": "2025-01-04T10:15:00Z",
  "compressed": false,
  "paged_out": false
}
```

This allows visualization of context structure:

```bash
# Show context composition
for f in /aos/procs/1234/context/blocks/*.json; do
  jq -r '[.index, .type, .tokens] | @tsv' "$f"
done | column -t
# 1   system      2400
# 2   user        156
# 3   assistant   1247
# 4   tool_call   89
# 5   tool_result 3421
```

---

## 6. Conversations

### 6.1 Directory Structure

Every agent interaction is preserved as a directory:

```
/aos/conversations/
├── 2025/
│   └── 01/
│       └── 04/
│           ├── a3f8c2/
│           │   ├── meta.json           # Metadata
│           │   ├── manifest.json       # Complete run manifest (for replay)
│           │   ├── transcript.md       # Human-readable
│           │   ├── transcript.jsonl    # Machine-parseable
│           │   ├── context.ctx         # Restorable context state
│           │   ├── summary.md          # Auto-generated summary
│           │   ├── artifacts/          # Files produced
│           │   │   ├── report.pdf
│           │   │   └── analysis.py
│           │   ├── tools/              # Tool calls made
│           │   │   ├── 001_web_search.json
│           │   │   └── 002_fs_write.json
│           │   ├── decisions.jsonl     # Approvals granted
│           │   ├── cost.json           # Spend breakdown
│           │   └── core/               # Core dump (if crashed)
│           │
│           └── b7c2d1/
│
├── by-tag/                             # Symlink views
│   ├── research/
│   │   └── a3f8c2 -> ../../2025/01/04/a3f8c2
│   └── coding/
│
├── by-agent/
│   ├── researcher/
│   └── coder/
│
├── by-project/
│   └── fusion-analysis/
│
├── starred/                            # User-curated
│
└── active/                             # Currently running
    └── a3f8c2 -> ../2025/01/04/a3f8c2
```

### 6.2 Metadata

```json
// /aos/conversations/2025/01/04/a3f8c2/meta.json
{
  "id": "a3f8c2",
  "created": "2025-01-04T10:30:00Z",
  "ended": "2025-01-04T10:35:22Z",
  "duration_sec": 322,
  
  "entry_point": {
    "agent": "researcher",
    "prompt": "Analyze current fusion energy status"
  },
  
  "participants": {
    "human": "alice",
    "agents": ["researcher", "critic"]
  },
  
  "lineage": {
    "parent": null,
    "forked_from": null,
    "children": ["b7c2d1", "c8d3e2"]
  },
  
  "tags": ["research", "energy", "fusion"],
  "project": "fusion-analysis",
  "starred": false,
  
  "outcome": "success",
  "exit_code": 0,
  "confidence": 0.87,
  
  "cost": {
    "tokens_in": 45000,
    "tokens_out": 12000,
    "tool_calls": 7,
    "total_usd": 0.47
  },
  
  "artifacts": ["report.pdf", "analysis.py"],
  "tools_used": ["web.search", "fs.write"],
  "approvals_required": 2,
  "approvals_granted": 2
}
```

### 6.3 Manifest (for Replay)

```json
// /aos/conversations/2025/01/04/a3f8c2/manifest.json
{
  "version": 1,
  "id": "a3f8c2",
  "created": "2025-01-04T10:30:00Z",
  
  "agent": {
    "name": "researcher",
    "config_hash": "sha256:abc123...",
    "model": "claude-sonnet-4-5-20250514"
  },
  
  "initial_prompt": "Analyze current fusion energy status",
  "initial_context_hash": "sha256:def456...",
  
  "tool_results": [
    {
      "id": "tc_001",
      "tool": "web.search",
      "args": {"q": "fusion energy 2024"},
      "result_hash": "sha256:789abc...",
      "cached": true
    }
  ],
  
  "final_output_hash": "sha256:final123...",
  "deterministic": true
}
```

### 6.4 Auto-Generated Summary

```markdown
# /aos/conversations/2025/01/04/a3f8c2/summary.md

## Summary
Researched current state of fusion energy, focusing on NIF ignition 
achievement and ITER timeline. Produced comprehensive report.

## Key Points
- NIF achieved net energy gain December 2022
- ITER first plasma delayed to 2035
- Private fusion companies raised $6B in 2024

## Decisions Made
- Approved web search (7 calls)
- Approved file write (report.pdf)

## Follow-up Suggested
- Track NIF replication attempts
- Monitor Commonwealth Fusion funding

## Cost: $0.47 | Duration: 5m 22s | Confidence: HIGH
```

### 6.5 Resuming Conversations

```bash
# View past conversation
cat /aos/conversations/2025/01/04/a3f8c2/transcript.md

# Resume with full context
aos resume /aos/conversations/2025/01/04/a3f8c2/
# Spawns new process with context.ctx loaded

# Or manually via context file
echo "/aos/conversations/2025/01/04/a3f8c2/context.ctx" \
  > /aos/agents/researcher/context
echo "Continue the analysis" > /aos/agents/researcher/inbox
```

### 6.6 Forking Conversations

```bash
# Fork from a conversation
aos fork /aos/conversations/2025/01/04/a3f8c2/ \
  --prompt="but what if we assumed 10% growth instead?"

# Creates new conversation with parent link
cat /aos/conversations/2025/01/04/c8d3e2/meta.json | jq '.lineage'
# {"parent": null, "forked_from": "a3f8c2", "children": []}
```

### 6.7 Searching Conversations

```bash
# Full-text search
grep -r "fusion ignition" /aos/conversations/

# Find by tag
ls /aos/conversations/by-tag/research/

# Find expensive conversations
find /aos/conversations -name cost.json -exec \
  sh -c 'jq -e ".total_usd > 1.0" "$1" >/dev/null && echo "$1"' _ {} \;

# Find conversations with specific tool
grep -rl "web.search" /aos/conversations/*/tools/

# Version control
cd /aos/conversations && git init && git add . && git commit -m "History"
```

### 6.8 Live Conversations

While running, conversations are linked from both locations:

```
/aos/procs/1234/conversation -> /aos/conversations/2025/01/04/a3f8c2/
/aos/conversations/active/a3f8c2 -> ../2025/01/04/a3f8c2/
```

On completion:
- Process directory is removed (after cleanup window)
- Active symlink is removed
- summary.md is generated
- Conversation remains in chronological location

### 6.9 Retention Policies

Configure conversation retention in `/aos/etc/retention.yaml`:

```yaml
# /aos/etc/retention.yaml
policies:
  default:
    keep_days: 90
    archive_after_days: 30
    compress_after_days: 7
  
  by_tag:
    important:
      keep_days: 365
      archive_after_days: 90
    scratch:
      keep_days: 7
      archive_after_days: 1
  
  starred:
    keep_days: forever
  
archive:
  destination: /aos/archive
  compress: true
  
cleanup:
  schedule: "0 3 * * *"  # Daily at 3am
  dry_run: false
```

Run retention manually:
```bash
aos retention --dry-run  # Preview
aos retention            # Execute
```

---

## 7. Memory

### 7.1 Directory Structure

```
/aos/memory/
├── global/                    # System-wide knowledge
│   ├── facts.jsonl           # Append-only fact store
│   ├── entities/             # Known entities
│   │   ├── openai.md
│   │   └── anthropic.md
│   └── relationships.jsonl   # Entity relationships
│
├── projects/                  # Project-scoped memory
│   └── fusion-analysis/
│       ├── context.md        # Project context
│       ├── findings.jsonl    # Discovered facts
│       ├── decisions.md      # Decisions made
│       └── open_questions.md # Unresolved items
│
├── agents/                    # Per-agent memory
│   └── researcher/
│       ├── learned.jsonl     # Things this agent learned
│       └── preferences.yaml  # Behavioral adjustments
│
└── scratch/                   # Temporary working memory
    └── ...
```

### 7.2 Memory Operations

**Read memory:**
```bash
cat /aos/memory/global/facts.jsonl
grep "fusion" /aos/memory/global/facts.jsonl
```

**Write memory:**
```bash
echo '{"fact":"NIF achieved ignition","confidence":0.95,"source":"..."}' \
  >> /aos/memory/global/facts.jsonl
```

**Search memory:**
```bash
grep -r "ignition" /aos/memory/
```

**Build memory from conversations:**
```bash
cat /aos/conversations/by-project/fusion/*/transcript.md | \
  aos invoke summarizer "extract key findings" \
  > /aos/memory/projects/fusion/synthesis.md
```

### 7.3 Memory Access Control

Agents have scoped memory access via capabilities:

```yaml
# In agent config
capabilities:
  memory:
    read: ["/aos/memory/global/**", "/aos/memory/projects/fusion/**"]
    write: ["/aos/memory/scratch/**", "/aos/memory/projects/fusion/**"]
```

Enforced at FUSE level based on process capabilities.

---

## 8. Channels

### 8.1 Directory Structure

```
/aos/channels/
├── terminal               # Default stdout
│
├── human/                 # Human approval channels
│   ├── alice              # Write=request, read=response (blocking)
│   ├── bob
│   └── ops-team           # Group channel
│
├── slack/
│   ├── general            # Write=post message
│   ├── alerts
│   └── dev-team
│
├── email/
│   ├── alice@company.com
│   └── alerts@company.com
│
├── webhooks/
│   ├── pagerduty
│   ├── datadog
│   └── custom
│
├── queues/                # Async job queues
│   ├── batch_processing
│   └── nightly_reports
│
└── null                   # /dev/null equivalent
```

### 8.2 Human Approval

Human channels implement blocking I/O for approval workflows:

```bash
# Request approval
echo '{"action":"fs.delete","path":"/important/*","reason":"cleanup"}' \
  > /aos/channels/human/alice

# Blocks until alice responds
response=$(cat /aos/channels/human/alice)
# {"approved":true,"signer":"alice@ed25519:...","timestamp":"..."}
```

Alice receives notification (configured separately) and responds via UI or CLI.

**Batch approval:**
```bash
echo '{"batch":[
  {"action":"fs.delete","path":"/tmp/1"},
  {"action":"fs.delete","path":"/tmp/2"}
]}' > /aos/channels/human/ops-team
```

**Timeout handling:**
```bash
if timeout 300 cat /aos/channels/human/alice > /tmp/response 2>/dev/null; then
  # Approved
else
  # Timed out - handle gracefully
  echo "Approval timed out" >&2
  exit 1
fi
```

**Group channels:**

Group channels (e.g., `ops-team`) can be configured for different approval modes:

```yaml
# /aos/etc/channels/human.yaml
groups:
  ops-team:
    members: [alice, bob, charlie]
    mode: any           # any | all | quorum
    quorum_count: 2     # Required if mode=quorum
```

### 8.3 Notification Channels

Fire-and-forget writes:

```bash
# Slack
echo "Build completed successfully" > /aos/channels/slack/alerts

# Email
cat /aos/conversations/2025/01/04/a3f8c2/summary.md \
  > /aos/channels/email/team@company.com

# Webhook
echo '{"event":"task_complete","id":"a3f8c2"}' \
  > /aos/channels/webhooks/datadog
```

### 8.4 Job Queues

Async work queues with acknowledgment:

```bash
# Submit job
echo '{"task":"generate_report","params":{...}}' \
  > /aos/channels/queues/batch_processing

# Worker consumes (blocking read, auto-ack on read)
while job=$(cat /aos/channels/queues/batch_processing); do
  process_job "$job"
done
```

**Manual acknowledgment mode:**

```bash
# Read without ack
job=$(cat /aos/channels/queues/batch_processing/peek)
job_id=$(echo "$job" | jq -r '.id')

# Process...
if process_job "$job"; then
  # Acknowledge success
  echo "$job_id" > /aos/channels/queues/batch_processing/ack
else
  # Negative ack (requeue)
  echo "$job_id" > /aos/channels/queues/batch_processing/nack
fi
```

**Dead letter queue:**

Failed jobs (exceeded retry limit) go to dead letter:
```bash
ls /aos/channels/queues/batch_processing.dlq/
# job_001.json  job_002.json

# Inspect failed job
cat /aos/channels/queues/batch_processing.dlq/job_001.json
# {"original":{...},"error":"timeout","attempts":3}

# Retry manually
cat /aos/channels/queues/batch_processing.dlq/job_001.json | \
  jq '.original' > /aos/channels/queues/batch_processing
rm /aos/channels/queues/batch_processing.dlq/job_001.json
```

### 8.5 Channel Configuration

```yaml
# /aos/etc/channels/slack.yaml
provider: slack
credentials: ${SLACK_BOT_TOKEN}
channels:
  general: C01234567
  alerts: C89012345
default_channel: general
retry:
  max_attempts: 3
  backoff_sec: [1, 5, 30]

# /aos/etc/channels/human.yaml
provider: human
notification:
  method: slack
  channel: approvals
timeout_sec: 3600
require_signature: true
signature_algorithm: ed25519

# /aos/etc/channels/queues.yaml
provider: queue
queues:
  batch_processing:
    max_size: 1000
    visibility_timeout_sec: 300
    max_retries: 3
    dlq: batch_processing.dlq
```

### 8.6 Channel Error Handling

**Retry policy:**

Channels retry failed deliveries according to configuration:

```yaml
retry:
  max_attempts: 3
  backoff_sec: [1, 5, 30]  # Exponential backoff
  on_failure: dlq          # dlq | drop | block
```

**Error visibility:**

```bash
# Check channel health
cat /aos/channels/slack/alerts/.status
# {"healthy":true,"last_delivery":"2025-01-04T10:30:00Z"}

# Check pending retries
ls /aos/channels/slack/alerts/.retry/
# msg_001.json  msg_002.json

# Check dead letters
ls /aos/channels/slack/alerts/.dlq/
```

---

## 9. External Mounts

### 9.1 Mount Command

```bash
# GitHub
aos mount github \
  --token=$GITHUB_TOKEN \
  --repo=owner/repo \
  /aos/mnt/github

# S3
aos mount s3 \
  --bucket=my-bucket \
  --region=us-east-1 \
  /aos/mnt/s3

# PostgreSQL
aos mount postgres \
  --connection="postgres://user:pass@host/db" \
  --readonly \
  /aos/mnt/db

# Generic REST API
aos mount http \
  --base-url=https://api.example.com/v1 \
  --auth-header="Bearer $TOKEN" \
  /aos/mnt/api
```

### 9.2 Directory Structure

```
/aos/mnt/
├── github/
│   ├── issues/
│   │   ├── 123/
│   │   │   ├── body           # R/W: Issue body
│   │   │   ├── state          # R/W: open/closed
│   │   │   ├── labels/        # R/W: Labels
│   │   │   ├── comments/      # R: List, append: Add
│   │   │   │   ├── 1
│   │   │   │   └── 2
│   │   │   └── meta.json      # R: Metadata
│   │   ├── new                # W: Create issue
│   │   └── .cursor            # R/W: Pagination cursor
│   └── pulls/
│       └── 456/
│           ├── diff           # R: The diff
│           ├── files/         # R: Changed files
│           └── reviews/
│
├── s3/
│   └── my-bucket/
│       ├── data/
│       │   └── file.csv       # R/W: S3 object
│       └── .aos/
│           ├── cost           # R: Request counts
│           └── cache.policy   # R/W: Cache settings
│
├── db/
│   └── users/
│       ├── schema             # R: Table schema
│       ├── query              # W: SQL, R: Results
│       ├── 123.json           # R: Row by ID
│       └── .cursor            # R/W: Query pagination
│
└── api/
    └── v1/
        └── data.json          # R: GET, W: POST
```

### 9.3 Operation Semantics

| Operation | GitHub Issues | S3 | PostgreSQL | HTTP |
|-----------|--------------|-----|------------|------|
| `cat file` | GET resource | GET object | SELECT row | GET |
| `echo > file` | PUT/PATCH | PUT object | UPDATE | PUT |
| `echo >> file` | POST (comment) | Append | INSERT | POST |
| `rm file` | Close/delete | DELETE | DELETE | DELETE |
| `ls dir` | List (metadata only) | List objects | List tables | N/A |

**Explicit semantics (avoiding the latency trap):**

- `ls /aos/mnt/github/issues/` returns **metadata only** (issue numbers). It does not fetch issue bodies.
- `cat /aos/mnt/github/issues/123/body` triggers the actual API call.

This ensures `ls` is fast and predictable, while `cat` has explicit cost.

### 9.4 Pagination

Large result sets use cursor files:

```bash
# First page (default limit)
ls /aos/mnt/github/issues/
# 1  2  3  ... 100  .cursor

# Check if more pages
cat /aos/mnt/github/issues/.cursor
# {"next":"abc123","has_more":true,"total":547}

# Get next page
echo "abc123" > /aos/mnt/github/issues/.cursor
ls /aos/mnt/github/issues/
# 101  102  103  ... 200  .cursor

# Reset to beginning
echo "reset" > /aos/mnt/github/issues/.cursor
```

### 9.5 Caching

Mount caching is configurable:

```bash
# Check cache status
cat /aos/mnt/s3/my-bucket/.aos/cache.policy
# {"mode":"ttl","ttl_sec":300,"size_mb":100}

# Update cache policy
echo '{"mode":"ttl","ttl_sec":60}' > /aos/mnt/s3/my-bucket/.aos/cache.policy

# Invalidate cache
echo "invalidate" > /aos/mnt/s3/my-bucket/.aos/cache.ctl

# Cache modes: none, ttl, etag, manual
```

### 9.6 Cost Visibility

```bash
# Mount-level cost tracking
cat /aos/mnt/github/.aos/cost
# {"api_calls":47,"rate_limit_remaining":4953,"reset_at":"..."}

cat /aos/mnt/s3/my-bucket/.aos/cost
# {"get_requests":123,"put_requests":7,"bytes_transferred":1547234}
```

### 9.7 Why This Isn't Magic

Unlike rejected "semantic FUSE" (where reads triggered hidden RAG/LLM calls), mount operations have **documented, predictable semantics**:

- `cat /aos/mnt/github/issues/123/body` = `GET /repos/.../issues/123`
- One operation, one API call, known cost

The mapping is explicit, not inferred.

---

## 10. Models

### 10.1 Directory Structure

```
/aos/models/
├── claude-sonnet-4-5-20250514/
│   ├── info                  # R: Model information
│   ├── available             # R: true | false
│   ├── pricing               # R: Cost per token
│   ├── limits                # R: Context window, rate limits
│   └── status                # R: Current status
│
├── claude-opus-4-20250514/
│   └── ...
│
├── gpt-4-turbo/
│   └── ...
│
├── llama-3-70b/
│   └── ...
│
└── .local/                   # Locally-loaded models
    └── custom-finetuned/
        └── ...
```

### 10.2 Model Information

```bash
cat /aos/models/claude-sonnet-4-5-20250514/info
```
```json
{
  "id": "claude-sonnet-4-5-20250514",
  "provider": "anthropic",
  "family": "claude-4.5",
  "context_window": 200000,
  "output_limit": 64000,
  "supports": {
    "vision": true,
    "function_calling": true,
    "streaming": true,
    "extended_thinking": true
  },
  "training_cutoff": "2025-03-01"
}
```

### 10.3 Pricing

```bash
cat /aos/models/claude-sonnet-4-5-20250514/pricing
```
```json
{
  "input_per_1m_tokens": 3.00,
  "output_per_1m_tokens": 15.00,
  "cached_input_per_1m_tokens": 0.30,
  "currency": "USD"
}
```

### 10.4 Availability

```bash
# Check if model is available
cat /aos/models/claude-sonnet-4-5-20250514/available
# true

# Check status details
cat /aos/models/claude-sonnet-4-5-20250514/status
# {"available":true,"latency_ms":234,"queue_depth":0}

# List all available models
for m in /aos/models/*/; do
  [ "$(cat $m/available 2>/dev/null)" = "true" ] && basename "$m"
done
```

---

## 11. Tools

### 11.1 Directory Structure

```
/aos/tools/
├── builtin/                  # System-provided tools
│   ├── web.search/
│   │   ├── schema.json       # Tool schema (JSON Schema)
│   │   ├── description       # Human-readable description
│   │   ├── cost              # Cost model
│   │   └── enabled           # R/W: Enable/disable
│   ├── web.fetch/
│   ├── fs.read/
│   ├── fs.write/
│   ├── fs.list/
│   └── spawn/
│
├── plugins/                  # User-installed tools
│   └── custom_api/
│       ├── schema.json
│       ├── handler            # Executable or script
│       ├── config.yaml
│       └── enabled
│
└── mcp/                      # MCP server tools (auto-discovered)
    └── github/
        ├── create_issue/
        ├── list_issues/
        └── ...
```

### 11.2 Tool Schema

```json
// /aos/tools/builtin/web.search/schema.json
{
  "name": "web.search",
  "description": "Search the web for information",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query"
      },
      "num_results": {
        "type": "integer",
        "default": 10,
        "maximum": 50
      }
    },
    "required": ["query"]
  },
  "returns": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "title": {"type": "string"},
        "url": {"type": "string"},
        "snippet": {"type": "string"}
      }
    }
  }
}
```

### 11.3 Tool Cost Model

```bash
cat /aos/tools/builtin/web.search/cost
```
```json
{
  "type": "per_call",
  "base_cost_usd": 0.001,
  "rate_limit": {
    "calls_per_minute": 60
  }
}
```

### 11.4 Installing Plugin Tools

```bash
# Install from directory
cp -r /path/to/my_tool /aos/tools/plugins/

# Or use aos tool install
aos tool install ./my_tool

# Enable/disable
echo "true" > /aos/tools/plugins/my_tool/enabled
echo "false" > /aos/tools/plugins/my_tool/enabled
```

### 11.5 MCP Integration

MCP servers are mounted as tools:

```bash
# Register MCP server
aos mcp register github --command="npx @modelcontextprotocol/server-github"

# Tools appear automatically
ls /aos/tools/mcp/github/
# create_issue  list_issues  create_pr  ...

# Disable specific MCP tool
echo "false" > /aos/tools/mcp/github/create_pr/enabled
```

### 11.6 Man Pages (Auto-Generated Documentation)

Agent documentation is auto-generated and available for both humans and other agents:

```
/aos/man/
├── researcher           # Agent manual
├── coder
├── analyst
└── ...
```

**View agent documentation:**

```bash
# Using man command (if configured)
man -M /aos/man researcher

# Or directly
cat /aos/man/researcher
```

**Man page content (auto-generated from config.yaml):**

```
RESEARCHER(1)                Agent OS Manual                RESEARCHER(1)

NAME
    researcher - Research assistant for technical analysis

DESCRIPTION
    Research assistant focused on technical analysis. Searches for
    primary sources, cross-references claims, flags uncertainty.

CAPABILITIES
    tools: web.search, web.fetch, fs.read
    spawn: yes (limit: 5)
    memory.read: /aos/memory/**
    memory.write: /aos/memory/scratch/**

LIMITS
    max_cost_usd: 1.00
    context_tokens: 200000
    timeout_sec: 300

COST
    Average cost per invocation: $0.34 (based on last 50 runs)
    
USAGE
    echo "query" > /aos/agents/researcher/inbox
    aos invoke researcher "query"
    cat data.txt | ./researcher.aos

EXAMPLES
    echo "What is the current state of fusion energy?" > \
        /aos/agents/researcher/inbox
    
    aos invoke researcher --max-cost=2.00 "Deep analysis of NIF"

SEE ALSO
    analyst(1), coder(1)

AUTHOR
    Auto-generated from /aos/agents/researcher/config.yaml

                           January 2025                    RESEARCHER(1)
```

**Service discovery:**

Agents can read each other's man pages to understand capabilities:

```bash
# From within an agent's reasoning
cat /aos/man/analyst
# Now the agent knows analyst's capabilities and can spawn it appropriately
```

---

## 12. Security

### 12.1 Three-Level Coloring

Every process has a security color:

| Level | Name | Capabilities | Typical Source |
|-------|------|--------------|----------------|
| BLUE | Trusted | All granted capabilities | System, signed human input |
| YELLOW | Partial | Read, sandboxed write, no exec | Sanitized external, known APIs |
| RED | Untrusted | Read-only, no side effects | Raw user input, untrusted web |

```bash
cat /aos/procs/1234/color
# BLUE
```

### 12.2 Contamination Rules

When content from different trust levels combines:

```
BLUE + RED    -> RED      # Untrusted fully contaminates
BLUE + YELLOW -> YELLOW   # Partial trust reduces to partial
YELLOW + RED  -> RED      # Untrusted contaminates partial
```

A process's effective color is the minimum of its assigned color and all input colors.

### 12.3 Capability Model

Capabilities are fine-grained permissions in agent config:

```yaml
capabilities:
  tools:
    - web.search
    - web.fetch
    - fs.read
  spawn: true
  spawn_limit: 5
  memory:
    read: ["/aos/memory/**"]
    write: ["/aos/memory/scratch/**"]
  mounts:
    read: ["/aos/mnt/github/**"]
    write: []
```

Effective capabilities viewable at runtime:
```bash
cat /aos/procs/1234/capabilities
# {"tools":["web.search","fs.read"],"spawn":false,...}
```

### 12.4 Capability Intersection

When process A invokes agent B, B's effective capabilities are the intersection of:
1. A's capabilities (caller)
2. B's granted capabilities (agent config)
3. Session capability ceiling

A caller cannot escalate privileges through an agent.

### 12.5 Capability Leases

Capabilities are granted as short-lived leases:

```
/aos/procs/1234/leases/
├── fs.write.json     # {"scope":"/tmp/**","expires":"...","token":"..."}
└── web.search.json
```

Lease revocation:
```bash
# Revoke capability
rm /aos/procs/1234/leases/fs.write.json
# Process receives SIGCAPDROP (reflected in status)
```

### 12.6 Sanitization via Channels

Promoting content from lower to higher trust uses human channels:

```bash
# RED content needs human review
echo '{"content":"...","requested_level":"BLUE"}' \
  > /aos/channels/human/alice

response=$(cat /aos/channels/human/alice)
# {"approved":true,"level":"BLUE","signer":"alice@ed25519:..."}
```

The signature is cryptographic proof of human review.

### 12.7 Transactional Side Effects

All side effects follow two-phase commit:

**Phase 1: Intent**
```bash
# Intent file appears
cat /aos/procs/1234/intents/pending/001.json
# {"action":"fs.write","path":"/tmp/out.py","awaiting":"approval"}
```

**Phase 2: Approval**
```bash
# Approve intent
echo "approve" > /aos/procs/1234/intents/pending/001.json
# Or via human channel for sensitive operations
```

**Phase 3: Execution**
```bash
# Intent moves to completed
ls /aos/procs/1234/intents/
# completed/001.json
```

### 12.8 User Isolation

The daemon runs as a service user, but agent processes execute in restricted OS contexts:

**Sandbox user:**

```yaml
# /aos/etc/security/isolation.yaml
isolation:
  enabled: true
  default_user: aos_sandbox
  default_group: aos_sandbox
  
  # Agents can specify stricter isolation
  paranoid_user: nobody
  
  # Filesystem restrictions
  readonly_paths:
    - /etc
    - /usr
    - /bin
  
  blocked_paths:
    - /etc/shadow
    - /etc/passwd
    - ~/.ssh
    - ~/.gnupg
```

**Per-agent isolation:**

```yaml
# In agent config.yaml
spec:
  isolation:
    user: nobody           # Run as nobody
    network: none          # No network access
    filesystem: readonly   # Read-only filesystem
```

**Defense in depth:**

Even if an agent is "jailbroken" and successfully requests malicious tool calls:
- OS kernel prevents reading `/etc/shadow`, `~/.ssh/`
- Network namespace prevents unauthorized connections
- Filesystem namespace limits writable paths

---

## 13. Observability

### 13.1 Execution Trace

stderr carries structured execution trace:

```bash
tail -f /aos/procs/1234/stderr
```

```jsonl
{"ts":"...","type":"tool_call","tool":"web.search","args":{"q":"fusion"}}
{"ts":"...","type":"tool_result","tool":"web.search","status":"ok","tokens":1247}
{"ts":"...","type":"decision","action":"spawn","target":"@analyzer"}
{"ts":"...","type":"budget","spent_usd":0.034,"remaining_usd":0.466}
{"ts":"...","type":"checkpoint","id":"cp_002","trigger":"explicit"}
{"ts":"...","type":"compression","before":145000,"after":52000}
```

The trace is guaranteed—it doesn't depend on model introspection.

### 13.2 Optional Rationale

If model supports extended thinking:

```jsonl
{"ts":"...","type":"rationale","content":"I should search for recent results..."}
```

Rationale is supplementary. Security and debugging don't depend on it.

### 13.3 Provenance

Full provenance in conversation directory:

```
/aos/conversations/2025/01/04/a3f8c2/
├── tools/
│   ├── 001_web_search.json    # Full request/response
│   └── 002_fs_write.json
├── decisions.jsonl            # All approval decisions
└── manifest.json              # Complete run metadata
```

### 13.4 Replay

```bash
aos replay /aos/conversations/2025/01/04/a3f8c2/ --verify
# Replaying with recorded tool results...
# Output comparison: MATCH
# Replay: CONSISTENT
```

### 13.5 Debugging

Interactive debugging:

```bash
aos debug /aos/procs/1234

(aos-dbg) status
# RUNNING at token 4521

(aos-dbg) break "delete"
# Breakpoint set on pattern "delete"

(aos-dbg) continue
# ... runs until pattern matched ...

(aos-dbg) context
# Shows current context

(aos-dbg) step
# Process one tool call

(aos-dbg) quit
```

**Note:** Breakpoints are heuristic (pattern matching). They're for debugging, not security.

### 13.6 Core Dump Debugging

For crashed processes:

```bash
# List recent crashes
ls -la /aos/cores/
# lrwxrwxrwx 1 aos aos 45 Jan  4 10:30 1234_20250104_103022 -> ...

# Load core dump
aos debug --core /aos/cores/1234_20250104_103022/

(aos-dbg) meta
# PID: 1234
# Agent: researcher
# Exit Code: 66 (BUDGET_EXHAUSTED)
# Duration: 30m 22s

(aos-dbg) trace tail 20
# [Shows last 20 trace events before crash]

(aos-dbg) budget
# Limit: $1.00
# Spent: $1.00
# Tokens: 87,432 / 100,000

(aos-dbg) context summary
# Tokens: 145,000 / 200,000
# Checkpoints: cp_001, cp_002
# Last compression: 10m ago (145k -> 52k)

(aos-dbg) checkpoint list
# cp_001: 45,000 tokens, 2025-01-04T10:15:00Z
# cp_002: 89,000 tokens, 2025-01-04T10:25:00Z

(aos-dbg) restore cp_001
# Restoring to checkpoint cp_001...
# Spawned process 1245 with context from cp_001

(aos-dbg) quit
```

---

## 14. Resource Management

### 14.1 Budget Files

```
/aos/procs/1234/budget/
├── limit            # 1.00 (USD)
├── spent            # 0.47 (USD)
├── tokens_limit     # 100000
├── tokens_used      # 45000
└── velocity         # 0.12 (USD/min)
```

### 14.2 System-Wide Limits

```
/aos/system/
├── budget           # R/W: Global budget limit
├── spend            # R: Current total spend
├── velocity         # R: Current spend rate
└── providers/
    ├── anthropic/
    │   ├── status       # available | rate_limited | error
    │   ├── qps          # Current QPS
    │   ├── qps_limit    # QPS limit
    │   └── tpm          # Tokens per minute
    └── openai/
        └── ...
```

### 14.3 Budget Enforcement

When limits exceeded:
- `budget/spent` exceeds `budget/limit`: Process receives SIGXCPU, status becomes "budget_exceeded"
- Velocity exceeds threshold: Process receives SIGBRAKE, status becomes "throttled"

### 14.4 Provider Rate Limits

```yaml
# /aos/etc/providers.yaml
providers:
  anthropic:
    qps_limit: 40
    tpm_limit: 80000
  openai:
    qps_limit: 50
    tpm_limit: 100000
```

When limits would be exceeded, requests queue rather than fail:
```bash
cat /aos/system/providers/anthropic/status
# queued (3 requests waiting)
```

---

## 15. System Configuration

### 15.1 Configuration Directory

```
/aos/etc/
├── daemon.yaml           # Daemon configuration
├── providers.yaml        # Backend configuration
├── retention.yaml        # Conversation retention
├── channels/             # Channel configurations
│   ├── slack.yaml
│   ├── email.yaml
│   ├── human.yaml
│   └── queues.yaml
├── security/
│   ├── colors.yaml           # Color capability mapping
│   ├── policies.yaml         # Security policies
│   ├── isolation.yaml        # User isolation config
│   ├── indexer_blocklist.yaml # Anti-bankruptcy
│   └── circuit_breaker.yaml  # Cost circuit breaker
├── budgets/
│   └── default.yaml      # Default budget limits
└── mounts/               # Mount configurations
    ├── github.yaml
    └── s3.yaml
```

### 15.2 Daemon Configuration

```yaml
# /aos/etc/daemon.yaml
daemon:
  socket: /var/run/aos/daemon.sock
  pid_file: /var/run/aos/daemon.pid
  log_level: info
  log_file: /var/log/aos/daemon.log
  
fuse:
  mount_point: /aos
  allow_other: true
  default_permissions: true
  ignored_patterns:
    - "*.swp"
    - "*.swo"
    - "*~"
    - ".*.swp"
    - "4913"
  
defaults:
  agent:
    model: claude-sonnet-4-5-20250514
    max_cost_usd: 1.00
    timeout_sec: 300
  
storage:
  conversations: /var/lib/aos/conversations
  memory: /var/lib/aos/memory
  contexts: /var/lib/aos/contexts
  cache: /var/cache/aos
  
performance:
  max_concurrent_processes: 100
  context_cache_size_mb: 1000

core_dumps:
  enabled: true
  retention_days: 7
  max_count: 100
  max_size_mb: 1000
```

---

## 16. Shell Integration

### 16.1 Bash Compatibility

Standard bash works with FUSE:

```bash
# All standard operations work
echo "query" > /aos/agents/researcher/inbox
cat /aos/agents/researcher/output
grep -r "pattern" /aos/conversations/
tail -f /aos/procs/*/stderr
watch cat /aos/system/spend
```

### 16.2 Composition Patterns

**Sequential:**
```bash
echo "Research fusion" > /aos/agents/researcher/inbox
aos wait /aos/agents/researcher
cat /aos/agents/researcher/output > /tmp/research.txt

echo "/tmp/research.txt" > /aos/agents/writer/context
echo "Summarize this research" > /aos/agents/writer/inbox
aos wait /aos/agents/writer
```

**Parallel:**
```bash
echo "Research physics" > /aos/agents/researcher/inbox &
echo "Research economics" > /aos/agents/economist/inbox &
wait
```

**Pipeline with KV-cache sharing:**
```bash
# Use aos pipe for efficient context passing
aos pipe researcher writer "Analyze and summarize fusion energy"

# Equivalent to:
# 1. aos invoke researcher "Analyze..."
# 2. aos bind (KV-cache pointer)
# 3. aos invoke writer "Summarize..."
```

### 16.3 Cron Integration

```bash
# /etc/cron.d/aos

# Daily budget reset
0 0 * * * root echo "10.00" > /aos/system/budget

# Daily summary
0 9 * * * root aos invoke summarizer "$(cat /aos/memory/global/facts.jsonl)" \
  > /aos/channels/slack/daily-summary

# Alert on high spend
*/5 * * * * root [ "$(echo "$(cat /aos/system/spend) > 8" | bc)" -eq 1 ] && \
    echo "Budget warning: $(cat /aos/system/spend)" > /aos/channels/slack/alerts

# Archive old logs weekly
0 3 * * 0 root find /aos/conversations -mtime +7 -name "*.jsonl" -exec gzip {} \;

# Health check
* * * * * root [ "$(cat /aos/system/status)" != "healthy" ] && \
    echo '{"status":"unhealthy"}' > /aos/channels/webhooks/pagerduty
```

---

## 17. CLI Reference

### 17.1 Core Commands

| Command | Description |
|---------|-------------|
| `aos mount` | Mount Agent OS filesystem |
| `aos unmount` | Unmount Agent OS filesystem |
| `aos status` | Show daemon status |
| `aos version` | Show version information |

### 17.2 Agent Commands

| Command | Description |
|---------|-------------|
| `aos invoke <agent> [--flags] <prompt>` | Invoke agent with options |
| `aos wait <agent\|proc>` | Wait for completion |
| `aos ps [--tree]` | List processes |
| `aos kill <pid>` | Kill process |
| `aos compile <agent> -o <output>` | Compile agent to .aexe |
| `aos pipe <agent1> <agent2> <prompt>` | Chain agents with KV-cache sharing |

### 17.3 Context Commands

| Command | Description |
|---------|-------------|
| `aos bind <source> <target>` | Bind context between processes |
| `aos restore <checkpoint>` | Restore from checkpoint |
| `aos ctx pageout <proc> [options]` | Page out context |
| `aos ctx pagein <proc> [options]` | Page in context |
| `aos ctx merge <ctx1> <ctx2> [options]` | Merge contexts |

### 17.4 Conversation Commands

| Command | Description |
|---------|-------------|
| `aos resume <conversation>` | Resume conversation |
| `aos fork <conversation> --prompt=<p>` | Fork conversation |
| `aos replay <conversation> [--verify]` | Replay conversation |
| `aos retention [--dry-run]` | Run retention policy |

### 17.5 Mount Commands

| Command | Description |
|---------|-------------|
| `aos mount github [options] <path>` | Mount GitHub |
| `aos mount s3 [options] <path>` | Mount S3 |
| `aos mount postgres [options] <path>` | Mount PostgreSQL |
| `aos mount http [options] <path>` | Mount HTTP API |
| `aos umount <path>` | Unmount |

### 17.6 Tool Commands

| Command | Description |
|---------|-------------|
| `aos tool list` | List available tools |
| `aos tool install <path>` | Install plugin tool |
| `aos tool enable <tool>` | Enable tool |
| `aos tool disable <tool>` | Disable tool |
| `aos mcp register <name> [options]` | Register MCP server |

### 17.7 Debug Commands

| Command | Description |
|---------|-------------|
| `aos debug <pid>` | Start interactive debugger |
| `aos debug --core <path>` | Debug from core dump |
| `aos trace <pid>` | Stream execution trace |
| `aos logs [--follow]` | View daemon logs |

---

## 18. Daemon Operations

### 18.1 Starting the Daemon

```bash
# Start daemon (foreground)
aosd

# Start daemon (background)
aosd --daemon

# Start with specific config
aosd --config=/path/to/daemon.yaml
```

### 18.2 Systemd Integration

```ini
# /etc/systemd/system/aos.service
[Unit]
Description=Agent OS Daemon
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/aosd --daemon
ExecStop=/usr/bin/aos unmount
Restart=on-failure
User=aos
Group=aos

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl enable aos
sudo systemctl start aos

# Check status
sudo systemctl status aos
```

### 18.3 Startup Sequence

1. Load configuration from `/aos/etc/`
2. Initialize storage backends
3. Connect to model providers
4. Mount FUSE filesystem
5. Load agent definitions
6. Initialize indexer protection
7. Start accepting requests
8. Notify systemd (if applicable)

### 18.4 Shutdown Sequence

1. Stop accepting new requests
2. Wait for running processes (timeout: 30s)
3. Force-kill remaining processes
4. Flush pending writes
5. Unmount FUSE filesystem
6. Close provider connections
7. Exit

### 18.5 Health Checks

```bash
# Check daemon health
cat /aos/system/status
# healthy

# Detailed health
cat /aos/system/health
```
```json
{
  "status": "healthy",
  "uptime_sec": 3600,
  "processes": {
    "running": 3,
    "total_spawned": 147
  },
  "providers": {
    "anthropic": "available",
    "openai": "rate_limited"
  },
  "storage": {
    "conversations_count": 1234,
    "memory_size_mb": 567
  },
  "circuit_breaker": {
    "blocked_pids": 0,
    "total_blocks_24h": 3
  }
}
```

---

## 19. JSONL Event Schema

Standard event format for traces, streams, and inter-agent communication:

```jsonl
{"v":1,"ts":"...","type":"text","content":"The fusion experiment..."}
{"v":1,"ts":"...","type":"text","content":"achieved ignition.","final":true}
{"v":1,"ts":"...","type":"tool_call","id":"tc_001","tool":"web.search","args":{"q":"NIF"}}
{"v":1,"ts":"...","type":"tool_result","id":"tc_001","status":"ok","content":"..."}
{"v":1,"ts":"...","type":"decision","action":"spawn","target":"analyzer","reason":"..."}
{"v":1,"ts":"...","type":"budget","spent_usd":0.034,"remaining_usd":0.466}
{"v":1,"ts":"...","type":"checkpoint","id":"cp_002"}
{"v":1,"ts":"...","type":"compression","before_tokens":145000,"after_tokens":52000}
{"v":1,"ts":"...","type":"citation","claim":"...","source":"...","confidence":0.95}
{"v":1,"ts":"...","type":"artifact","name":"report.pdf","path":"/aos/...","hash":"..."}
{"v":1,"ts":"...","type":"approval","action":"fs.write","approved":true,"signer":"..."}
{"v":1,"ts":"...","type":"error","code":"BUDGET_EXHAUSTED","message":"..."}
{"v":1,"ts":"...","type":"rationale","content":"I should search..."}  // Optional
```

---

## 20. Determinism and Reproducibility

### 20.1 Idempotency Keys

Tool calls that produce side effects accept idempotency keys:

```jsonl
{"type":"tool_call","tool":"net.post","args":{...},"idempotency_key":"hash(ctx,intent,args)"}
```

If the same key is seen twice, the tool returns the cached result.

### 20.2 Dry-Run Mode

Preview actions without executing:

```bash
aos invoke researcher --dry-run "Refactor auth module"

cat /aos/agents/researcher/current/plan
# {"proposed":[
#   {"action":"fs.read","path":"src/auth/*.py"},
#   {"action":"fs.write","path":"src/auth/session.py"},
# ]}
```

### 20.3 Reproducibility

```bash
# Capture full run manifest
cat /aos/conversations/2025/01/04/a3f8c2/manifest.json
# Contains: prompt, context hash, tool results, timestamps

# Replay for verification
aos replay /aos/conversations/2025/01/04/a3f8c2/ --verify
```

---

## Appendix A: Quick Reference

### Signals (Daemon Internal)

| Signal | Trigger | Status Value |
|--------|---------|--------------|
| SIGTERM | `echo stop > ctl` | stopping |
| SIGKILL | `echo kill > ctl` | (removed) |
| SIGSTOP | `echo pause > ctl` | paused |
| SIGCONT | `echo resume > ctl` | running |
| SIGPIPE | Downstream closed | broken_pipe |
| SIGCTXPRESSURE | Context >80% | (pressure file appears) |
| SIGXCPU | Budget exhausted | budget_exceeded |
| SIGBRAKE | Velocity exceeded | throttled |
| SIGCAPDROP | Capability revoked | (file appears) |

### Exit Codes

| Code | Name | Meaning | Core Dump |
|------|------|---------|-----------|
| 0 | SUCCESS | Completed normally | No |
| 1 | FAILURE | General error | No |
| 2 | INVALID_INPUT | Malformed request | No |
| 64 | REFUSED | Capability violation | Yes |
| 65 | CTX_OVERFLOW | Context limit exceeded | Yes |
| 66 | BUDGET_EXHAUSTED | Cost/token limit | Yes |
| 67 | UPSTREAM_FAILURE | Dependency failed | Yes |
| 68 | LOW_CONFIDENCE | Declined due to uncertainty | Yes |
| 69 | CONTAMINATED | Color violation | Yes |
| 70 | LEASE_EXPIRED | Capability lease invalid | Yes |
| 71 | RATE_LIMITED | Provider QPS/TPM exceeded | Yes |

### Security Colors

| Color | Capabilities | Source |
|-------|--------------|--------|
| BLUE | All granted | System, signed human input |
| YELLOW | Read, sandboxed write | Sanitized external |
| RED | Read-only | Untrusted input |

### Contamination

```
BLUE + RED    -> RED
BLUE + YELLOW -> YELLOW
YELLOW + RED  -> RED
```

### FUSE Error Codes

| Condition | errno |
|-----------|-------|
| Agent not found | ENOENT |
| Permission denied | EACCES |
| Process not running | ESRCH |
| Invalid operation | EINVAL |
| Resource exhausted | ENOSPC |
| Capability denied | EPERM |
| Timeout | ETIMEDOUT |
| Blocked by circuit breaker | EACCES |

---

## Appendix B: Filesystem Layout Summary

```
/aos/
├── agents/                  # Agent definitions
│   └── {name}/
│       ├── config.yaml
│       ├── status, cost, inbox, output, stream, session, context, log
│       ├── override/
│       └── current/
│
├── procs/                   # Running processes
│   └── {pid}/
│       ├── status, agent, pid, ppid, started, exit
│       ├── stdin, stdout, stderr, ctl
│       ├── capabilities, color
│       ├── budget/, context/, leases/, intents/
│       ├── core/            # If crashed
│       └── conversation -> ../conversations/...
│
├── conversations/           # History
│   ├── {YYYY}/{MM}/{DD}/{id}/
│   │   ├── meta.json, manifest.json
│   │   ├── transcript.md, transcript.jsonl
│   │   ├── context.ctx, summary.md
│   │   ├── artifacts/, tools/, decisions.jsonl, cost.json
│   │   └── core/            # If crashed
│   ├── by-tag/, by-agent/, by-project/
│   ├── starred/, active/
│
├── memory/                  # Knowledge
│   ├── global/, projects/, agents/, scratch/
│
├── channels/                # I/O destinations
│   ├── terminal, null
│   ├── human/, slack/, email/, webhooks/, queues/
│
├── mnt/                     # External mounts
│   └── {provider}/
│
├── models/                  # Available models
│   └── {model}/
│       ├── info, available, pricing, limits, status
│
├── tools/                   # Available tools
│   ├── builtin/, plugins/, mcp/
│
├── man/                     # Auto-generated documentation
│   └── {agent}
│
├── cores/                   # Symlinks to crash dumps
│   └── {pid}_{timestamp} -> ...
│
├── system/                  # Runtime status
│   ├── status, health, budget, spend, velocity
│   ├── circuit_breaker/
│   └── providers/
│
├── etc/                     # Configuration
│   ├── daemon.yaml, providers.yaml, retention.yaml
│   ├── channels/, security/, budgets/, mounts/
│
├── .gitignore               # Anti-indexer markers
├── .ignore
├── .noindex
└── CACHEDIR.TAG
```

---

## Appendix C: Shebang Reference

**Format:**
```bash
#!/usr/bin/env aos
# @directive: value
# @directive: value

<system prompt / persona>
```

**Directives:**

| Directive | Type | Default | Description |
|-----------|------|---------|-------------|
| `@model` | string | System default | Model to use |
| `@budget` | float | 1.00 | Max cost in USD |
| `@tools` | list | [] | Allowed tools |
| `@timeout` | int | 300 | Timeout in seconds |
| `@context` | path | none | Initial context file |
| `@output` | string | text | Output format (text/jsonl) |

**Example:**
```bash
#!/usr/bin/env aos
# @model: claude-sonnet-4-5-20250514
# @budget: 2.00
# @tools: [web.search, fs.read]
# @timeout: 600

You are a research assistant. Analyze the input and produce
a comprehensive report with citations.
```

---

## Appendix D: Future Work (Deferred)

### D.1 Interactive Steering

**Concept:** Allow users to edit an agent's plan mid-execution, with the agent adapting to the modified plan.

**Status:** Deferred pending design work.

**Open Questions:**

1. **Interruption semantics:** What happens if the agent is mid-tool-call when the plan changes? Is the tool result discarded? Does the agent wait for completion then re-evaluate?

2. **Plan format:** How is the plan structured? Free-form text? Structured YAML? How does the agent reliably produce and parse it?

3. **Validation:** What if the user's edit is incoherent or malicious? Does the agent validate? Re-plan from scratch? Error out?

4. **Race conditions:** User edits while agent executes. Agent completes step 2 while user deletes step 2. Conflict resolution?

5. **Opt-in:** Should this be available for all agents or require `steerable: true` in config?

**Proposed Signal:** `SIGPLAN` — delivered when `current/plan` is modified by an external process.

**Proposed Interaction:**

```yaml
# /aos/procs/1234/current/plan (written by agent)
status: executing
current_step: 2
steps:
  - id: 1
    action: "Search for fusion history"
    status: completed
  - id: 2
    action: "Search for NIF data"
    status: in_progress
  - id: 3
    action: "Compile report"
    status: pending
```

User modifies, agent receives SIGPLAN, re-reads plan, adapts.

**Required design:** Atomicity guarantees, format specification, validation rules.

---

*Specification v2.2 - Draft Standard*
*Status: Under exploration, not finalized*
