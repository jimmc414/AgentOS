# Agent OS: Day 2 - FUSE Exploration

**Status:** Brainstorming Document / Architectural Exploration
**Context:** Critical review of Day 1 architecture, exploring filesystem-native interface layer

---

> Day 2, Session 1

## 1. The Honest Assessment

Day 1 produced an intellectually satisfying Unix-to-agents mapping. Day 2 begins with a harder question: **Is any of this actually useful, or is it overengineering?**

### 1.1 What's Genuinely Valuable

| Component | Value | Notes |
|-----------|-------|-------|
| KV-cache fork | High | O(1) branching for ToT, MCTS. Real capability current frameworks lack. |
| Execution traces | High | Observability is underserved. Structured tool call / decision / cost logs matter. |
| Budget tracking | Medium | Necessary for production, but trivially implementable. |
| Capability restrictions | Medium | Matters for untrusted input, but real sandboxing uses OS-level isolation. |

### 1.2 Day 1 Concepts: Filesystem Mapping Potential

These concepts need evaluation for how naturally they map to filesystem operations. The question is NOT "is this just X?" (everything can be done another way). The question IS "does the filesystem interface create value for AI agents and Unix users?"

| Component | Filesystem Mapping | Evaluation Question |
|-----------|-------------------|---------------------|
| Custom shell (ash) | May be replaced by bash + FUSE | Does FUSE + bash eliminate the need entirely? |
| Signal system | Control files (`echo "stop" > ctl`) | Is file-based control more intuitive than signals? |
| Three-level security | Unix user/group/other permissions | Does existing permission model map naturally? |
| Context paging | Memory as files (`/aos/memory/*`) | Same interface as everything else—grep, cat, write |
| Ring 0/1/2 model | FUSE (Ring 1) / Daemon (Ring 0) | Natural separation or forced abstraction? |
| `/proc` introspection | Already filesystem—native fit | Direct mapping, not metaphor |
| Budget tracking | `cat /aos/system/spend` | Observable without API—same as everything else |

**Key insight:** If the filesystem interface is valuable, it's valuable for ALL of these. They stand or fall together. Don't selectively dismiss some as "just X" while embracing others.

### 1.3 The Question (Not Conclusion)

Day 1 architecture may build an **operating system** when a **library** would suffice. But this is a hypothesis to test, not a verdict.

The FUSE pivot may:
- **Vindicate** some Day 1 concepts (filesystem metaphors become literal)
- **Replace** others (bash + FUSE instead of custom shell)
- **Simplify** others (Unix permissions instead of capability leases)

Defer judgment until end of Day 2.

### 1.4 What Would Actually Be Useful

```python
from agent_tools import Agent, Budget, Sandbox

agent = Agent("researcher", budget=Budget(max_usd=1.00))
result = agent.run("Study fusion")
print(result.trace)  # tool calls, decisions, cost

with Sandbox(capabilities=["read"]):
    agent.run(untrusted_input)

branches = agent.branch(5, "Try different approaches")  # KV-cache fork if available
```

This captures the value without the Unix cosplay.

---

## 2. The FUSE Pivot

But wait. There's one audience where the filesystem metaphor isn't cosplay: **AI CLI agents**.

### 2.1 The Key Insight

Claude Code, and any AI trained on Unix, already knows:
```bash
cat /path/to/file          # Read
echo "x" > /path/to/file   # Write
ls /path/to/dir            # Discover
grep "pattern" /path/**    # Search
tail -f /path/to/log       # Monitor
```

These operations are deeply embedded in training data. An AI agent can manipulate other agents **without learning new APIs**.

This isn't about humans preferring filesystems over Python. It's about **leveraging existing AI capabilities**.

### 2.2 What FUSE Enables

**Discoverability without documentation:**
```bash
ls /aos/
# agents/  models/  prompts/  memory/  conversations/

ls /aos/agents/
# researcher  writer  analyst

cat /aos/agents/researcher/status
# idle
```

The filesystem **is** the documentation.

**Tool reuse:**
```bash
# Search across all agent memories
grep -r "fusion" /aos/memory/

# Diff two agents' understanding
diff <(cat /aos/agents/researcher/output) <(cat /aos/agents/analyst/output)

# Backup entire agent state
tar czf backup.tar.gz /aos/agents/researcher/

# Version control prompts
cd /aos/prompts && git init && git add .

# Monitor spend
watch 'cat /aos/agents/*/cost | paste -sd+ | bc'
```

These aren't toy examples. They're genuine operations using tools that already exist.

### 2.3 The Layered View

**Day 1 thesis:** "Everything is an interlocutor" — build an OS around agent primitives.

**Day 2 addition:** "The filesystem is a UI for agent introspection" — expose agent state through an interface AI already understands.

These may be **complementary, not competing**:
- FUSE provides the **interface layer** (how users/AI interact)
- Day 1 concepts provide the **execution layer** (how the daemon works)

The filesystem is a **view layer**. The Day 1 process model, signals, and security concepts may still govern the daemon beneath it.

---

## 3. Impedance Mismatches

The filesystem metaphor has real problems. Acknowledge them, then solve them.

### 3.1 Latency Mismatch

**Problem:** Unix file operations are fast. Agent operations take seconds to minutes.

```bash
cat /aos/agents/researcher/output
# ... blocks for 30 seconds waiting for LLM?
```

This breaks mental models. `cat` shouldn't hang.

**Solution:** Separate instant-read files from blocking operations.

| File | Behavior | Expectation | Match? |
|------|----------|-------------|--------|
| `output` | Returns cached last result | `cat` is instant | ✓ |
| `status` | Returns current state | Quick check | ✓ |
| `stream` | Blocks for new tokens | `tail -f` blocks | ✓ |
| `inbox` | Async write, returns immediately | Fire and forget | ✓ |

The latency is **explicit**:
- Writes to `inbox` queue work (async)
- Reads from `output` return cached state (instant)
- `tail -f stream` blocks (expected for streaming)

### 3.2 Statefulness Mismatch

**Problem:** Files are stateless. Conversations aren't.

```bash
echo "What is 2+2?" > /aos/agents/calc/inbox
cat /aos/agents/calc/output  # 4

echo "And 3+3?" > /aos/agents/calc/inbox
# Does it remember the previous exchange?
```

**Solution:** Make session state explicit.

```
/aos/agents/researcher/
├── inbox               # W: send message to current session
├── output              # R: last output from current session
├── stream              # R: live tokens (tail -f)
├── status              # R: idle/running
├── session             # R: current session ID
└── sessions/
    ├── current -> s_047/
    └── s_047/
        ├── messages.jsonl
        └── context.json
```

New session: `echo "new" > /aos/agents/researcher/session`
Continue session: just write to `inbox`

### 3.3 Structured Data as Flat Files

**Problem:** Agent memory isn't a file. It's embeddings, graphs, conversation history.

**Solution:** Sensible projections, not forced flattening.

```
/aos/agents/researcher/memory/
├── summary             # Plain text: "This agent knows about fusion, NIF, ITER..."
├── searchable          # Greppable content fragments
├── recent.jsonl        # Last N interactions
└── stats               # "entries: 1547, tokens: 89234"
```

The filesystem exposes **useful views**, not raw internals. You can `grep searchable` without understanding vector embeddings.

---

## 4. FUSE Implementation Options

This section explores implementation approaches. No decisions made.

### 4.1 File Semantics (Proposed)

```python
class AgentOutputFile:
    """Always returns instantly - cached last completed output."""

    def read(self, size, offset):
        return self.agent.last_completed_output[offset:offset+size]

    def getattr(self):
        return {
            'st_size': len(self.agent.last_completed_output),
            'st_mtime': self.agent.last_output_timestamp,  # Freshness signal
        }

class AgentStreamFile:
    """Blocks until new tokens available - tail -f behavior."""

    def read(self, size, offset):
        return self.agent.token_stream.read_blocking(size)

    def poll(self):
        return POLLIN if self.agent.token_stream.has_data() else 0

class AgentInboxFile:
    """Async write - queues message and returns immediately."""

    def write(self, data, offset):
        self.agent.queue_message(data.decode('utf-8'))
        return len(data)  # Returns immediately
```

### 4.2 Using mtime as Signal

FUSE controls file metadata. Use `mtime` to signal freshness:

```bash
# Check if output updated since last read
stat /aos/agents/researcher/output
# Modified: 2025-01-04 14:23:07

# In scripts
if [ /aos/agents/researcher/output -nt /tmp/last_check ]; then
    echo "New output available"
fi

# Watch for changes
inotifywait -m /aos/agents/researcher/output
```

### 4.3 Filesystem Structure (Working Draft)

One possible structure. Alternatives exist; this is for exploration.

```
/aos/
├── agents/
│   └── researcher/
│       ├── config.yaml         # R/W: prompt, capabilities, limits
│       ├── status              # R: "idle" | "running" | "error"
│       ├── cost                # R: "0.47" (current session spend)
│       ├── inbox               # W: send message (async)
│       ├── output              # R: last completed output (instant)
│       ├── stream              # R: live token stream (tail -f)
│       ├── session             # R/W: current session ID
│       ├── log                 # R: append-only JSONL history
│       └── current/            # R: current interaction details
│           ├── input
│           ├── output
│           ├── trace.jsonl
│           └── meta            # timestamp, cost, tokens
│
├── models/
│   ├── claude-sonnet/
│   │   ├── info                # R: pricing, context window, status
│   │   └── available           # R: "true" | "false"
│   ├── claude-opus/
│   └── gpt-4/
│
├── prompts/                    # R/W: prompt library
│   ├── researcher.md
│   ├── writer.md
│   └── templates/
│
├── memory/                     # R/W: shared memory regions
│   ├── project_alpha/
│   │   ├── summary
│   │   ├── searchable
│   │   └── facts.jsonl
│   └── scratch/
│
├── conversations/              # R: archived sessions
│   ├── 2025-01-04_researcher_s047.jsonl
│   └── ...
│
└── system/
    ├── budget                  # R/W: global budget limit
    ├── spend                   # R: current total spend
    ├── status                  # R: system health
    └── providers/
        ├── anthropic/
        │   ├── status          # R: "available" | "rate_limited"
        │   └── usage           # R: current QPS, TPM
        └── openai/
```

---

## 5. History Management

Three options identified. Decision deferred pending further exploration.

### 5.1 Option A: Numbered Directories (Maildir-style)

```
/aos/agents/researcher/
├── output              # Last completed (quick access)
├── stream              # Live tokens (tail -f)
└── history/
    ├── 001/
    │   ├── input
    │   ├── output
    │   ├── trace.jsonl
    │   └── meta
    ├── 002/
    └── latest -> 002/
```

| Pros | Cons |
|------|------|
| Each interaction is discrete unit | More complex structure |
| Easy to archive/delete individual interactions | Directory management overhead |
| Multiple artifacts grouped naturally | More files to traverse |

### 5.2 Option B: Append-Only Log (syslog-style)

```
/aos/agents/researcher/
├── output              # Last completed (quick access)
├── stream              # Live tokens (tail -f)
└── log                 # Append-only JSONL (all history)
```

| Pros | Cons |
|------|------|
| Simple, single file | Harder to extract single interaction |
| Greppable | Need jq for structured queries |
| Standard log rotation | Large files over time |

### 5.3 Option C: Overwrite Only (stdout-style)

```
/aos/agents/researcher/
├── output              # Last completed only
└── stream              # Live tokens (tail -f)
# No history - user redirects if they want it
```

| Pros | Cons |
|------|------|
| Simplest, most Unix-native | User must explicitly save |
| No storage growth | History lost by default |
| Matches stdout semantics | May frustrate users |

### 5.4 Option D: Hybrid (Log + Current Structure)

```
/aos/agents/researcher/
├── output              # Last completed (quick access)
├── stream              # Live tokens (tail -f)
├── log                 # Append-only JSONL (all history)
└── current/            # Current interaction structure
    ├── input
    ├── output
    ├── trace.jsonl
    └── meta
```

| Pros | Cons |
|------|------|
| Quick access via `output` | Some redundancy |
| Greppable history via `log` | Two places to look |
| Structured current via `current/` | More complex |

### 5.5 Usage Patterns (All Options)

```bash
# Quick access to last result (all options)
cat /aos/agents/researcher/output

# Search history (Options A, B, D)
grep "fusion" /aos/agents/researcher/log              # B, D
grep -r "fusion" /aos/agents/researcher/history/      # A

# Watch live (all options)
tail -f /aos/agents/researcher/stream

# Extract specific interaction
jq 'select(.id==5)' log                               # B, D
cat history/005/output                                 # A
```

**Decision deferred.** Need to evaluate which patterns AI agents and Unix users actually prefer.

---

## 6. Interaction Patterns

### 6.1 AI CLI Agent Workflow

```bash
# AI discovers available agents
ls /aos/agents/
# researcher  writer  analyst

# AI checks capabilities
cat /aos/agents/researcher/config.yaml | grep capabilities
# capabilities: [web_search, file_read]

# AI checks budget before starting
cat /aos/system/spend
# 2.47
cat /aos/system/budget
# 10.00

# AI invokes agent
echo "Study recent fusion breakthroughs" > /aos/agents/researcher/inbox

# AI monitors progress
tail -f /aos/agents/researcher/stream &
# {"type":"thinking","content":"I'll search for..."}
# {"type":"tool_call","tool":"web_search",...}
# {"type":"text","content":"Recent breakthroughs...","final":true}

# AI retrieves structured result
cat /aos/agents/researcher/current/output

# AI checks cost
cat /aos/agents/researcher/cost
# 0.23
```

### 6.2 Unix Power User Workflow

```bash
# Monitor all agents' spend
watch 'for a in /aos/agents/*/; do
    echo -n "$(basename $a): "; cat "$a/cost" 2>/dev/null || echo "0"
done'

# Search across all agent histories
grep -l "quantum computing" /aos/agents/*/log

# Quick prompt iteration
vim /aos/agents/researcher/config.yaml
# Edit prompt, save
# Next invocation uses new prompt

# Backup before experiment
tar czf pre_experiment.tar.gz /aos/agents/researcher/

# Diff outputs from two agents
diff <(jq -r 'select(.type=="output").content' /aos/agents/researcher/log | tail -1) \
     <(jq -r 'select(.type=="output").content' /aos/agents/analyst/log | tail -1)
```

### 6.3 Automation / Cron

```bash
# Daily budget reset
0 0 * * * echo "10.00" > /aos/system/budget

# Alert on high spend
*/5 * * * * [ $(echo "$(cat /aos/system/spend) > 8" | bc) -eq 1 ] && \
    notify-send "Agent OS: Budget warning"

# Archive old logs weekly
0 3 * * 0 find /aos/conversations -mtime +7 -name "*.jsonl" -exec gzip {} \;

# Health check
* * * * * [ "$(cat /aos/system/status)" != "healthy" ] && \
    echo "Agent OS unhealthy" | mail -s "Alert" admin@example.com
```

---

## 7. What This Is and Isn't

### 7.1 This IS

- **A view layer** for agent state via filesystem semantics
- **An interface optimized for AI CLI agents** that already know filesystem operations
- **A proof-of-concept** exploring whether filesystem metaphors help or hinder
- **Unix-native tooling** for agent inspection, configuration, and monitoring

### 7.2 This IS NOT (Current Scope)

- **A complete agent runtime** — the daemon doing inference is separate (though Day 1 concepts may inform daemon design)
- **A replacement for Python APIs** — developers should use libraries (FUSE is for AI agents and Unix users)
- **Production-ready security** — real sandboxing uses OS-level isolation (though Day 1 security concepts may layer on top)
- **The execution model** — filesystem exposes state; execution semantics live in daemon (Day 1 process model may apply there)

### 7.3 Target Users (Hypothesized)

If the FUSE approach is valuable, these would be the likely users:

1. **AI CLI agents** — Leverages existing training on filesystem operations
2. **Unix power users** — Familiar tools, zero learning curve
3. **Automation / scripts** — Cron, shell scripts, standard Unix patterns

**Probably not targeting:** Regular developers (prefer Python SDK), GUI users, enterprises (need proper authz)

**To validate:** Are these actually the right users? Does the FUSE interface serve them better than alternatives?

---

## 8. Day 1 Concepts: Filesystem Mapping Exploration

All Day 1 concepts are preserved. This table explores how each might map to the FUSE approach.

**Evaluation principle:** The question is NOT "is this just X?" (everything can be done another way). The question IS "does exposing this through filesystem create a unified interface for AI agents and Unix users?"

| Concept | Possible FUSE Mapping | Exploration Status |
|---------|----------------------|-------------------|
| Execution traces | `log` file, `current/trace.jsonl` | Natural fit — files are logs |
| Budget tracking | `cost`, `system/spend`, `system/budget` | Natural fit — observable via cat |
| KV-cache fork | Backend optimization (not exposed) | Orthogonal — daemon internal |
| Capability restrictions | Unix permissions (rwx) + config file | To explore — does rwx suffice? |
| Process model | `status` file, `ctl` control file | To explore — control files vs signals |
| Context binding (`|=`) | Symlinks? Context directory references? | To explore — may need daemon API |
| Custom shell (ash) | Potentially replaced by bash + FUSE | To explore — redundant or complementary? |
| Three-level security | Unix user/group/other | To explore — natural mapping exists |
| Signals | Control files (`echo "stop" > ctl`) | To explore — may be more intuitive |
| Ring 0/1/2 model | FUSE (userland) / Daemon (kernel) | Natural fit — this IS the separation |
| Context paging | Memory as files (`/aos/memory/*`) | Natural fit — same interface as everything else |

**Note on context paging:** This is NOT "just RAG with new vocabulary." It's memory operations exposed through the same interface as all other operations. An AI agent uses `cat`, `grep`, `echo >` for memory just like for agent invocation, budget checking, and status monitoring. The value is unified interface, not novelty.

---

## 9. Open Questions

Questions to resolve through further exploration.

### 9.1 Session Management

How explicit should session boundaries be?

**Option A:** Implicit sessions (each inbox write continues current session)
```bash
echo "query 1" > inbox  # Session starts
echo "query 2" > inbox  # Continues session
```

**Option B:** Explicit session control
```bash
echo "new" > session    # Start new session
echo "query" > inbox    # In new session
```

**Option C:** Hybrid (implicit default, explicit when needed)

**To explore:** What do AI agents expect? What do Unix users expect?

### 9.2 Multi-turn in Single Write

Can you write a full conversation to inbox?
```bash
cat << 'EOF' > /aos/agents/researcher/inbox
user: What is fusion?
assistant: Fusion is...
user: Tell me more about NIF.
EOF
```

**Options:**
- Accept JSONL for multi-turn, plain text for single message
- Only accept single messages, multi-turn via repeated writes
- Only accept JSONL always

**To explore:** What's most intuitive for AI agents?

### 9.3 Binary Files

Should agents handle images, PDFs?
```bash
cat image.png > /aos/agents/vision/inbox
```

**Options:**
- Yes, if model supports multimodal
- No, require base64 encoding
- Separate endpoints (`inbox` for text, `inbox.bin` for binary)

**To explore:** How do multimodal models handle this in practice?

### 9.4 Remote Agents

Mount remote agent systems?
```bash
mount -t aos user@remote:/aos /aos/remote
cat /aos/remote/agents/researcher/output
```

**Options:**
- Network transparency (9P-style)
- Explicit remote commands
- Out of scope

**To explore:** Is this valuable or scope creep?

### 9.5 Relationship to Day 1 Concepts

How do Day 1 concepts map to FUSE?

| Day 1 Concept | Possible FUSE Mapping | Alternative |
|---------------|----------------------|-------------|
| Signals | Control files (`echo "stop" > ctl`) | Daemon API |
| Capabilities | Unix permissions (rwx) | Config file |
| Security levels | Unix user/group | Separate directories |
| Context binding | Symlinks? Pipe to file? | Daemon API only |
| Process model | Status files + control files | Daemon internal only |

**To explore:** Which mappings are natural vs forced?

---

## 10. Implementation Considerations

Architectural options to explore. No commitments made.

### 10.1 Possible Component Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FUSE Filesystem                       │
│  /aos/agents/*/  /aos/models/*/  /aos/system/*          │
└─────────────────────┬───────────────────────────────────┘
                      │ read/write/poll
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    Agent Daemon                          │
│  - Manages agent lifecycles                              │
│  - Handles inference requests                            │
│  - Tracks budget, logs, sessions                         │
│  - Exposes state for FUSE to read                        │
│  - (Day 1 process model may apply here?)                 │
└─────────────────────┬───────────────────────────────────┘
                      │ API calls
                      ▼
┌─────────────────────────────────────────────────────────┐
│                 Model Backends                           │
│  Claude API  │  OpenAI API  │  vLLM  │  Ollama          │
└─────────────────────────────────────────────────────────┘
```

**Open question:** Does the daemon need Day 1's process model (signals, capabilities, etc.) or is simpler sufficient?

### 10.2 Possible PoC Phases (If Built)

These are illustrative, not committed:

**Phase 1: Minimum viable exploration**
- FUSE mount with basic structure
- Single agent with `inbox` / `output` / `status`
- `stream` with tail -f support
- Local model (Ollama) for fast iteration

**Phase 2: Expanded exploration**
- Multiple agents
- Cloud backends
- Budget tracking
- History management (test different options from §5)

**Phase 3: Day 1 integration exploration**
- Can Day 1 signals map to control files?
- Can Day 1 security map to Unix permissions?
- Can Day 1 context binding work through filesystem?

**No implementation commitment made.** This section documents what *could* be built to test hypotheses.

---

## 11. Hypotheses to Test

If a PoC is built, these hypotheses would be validated or invalidated:

### 11.1 Positive Hypotheses (Hope These Are True)

1. **AI agents can discover and invoke agents** using only filesystem operations from training data
2. **Standard Unix tools work naturally** — grep, tail, watch, diff on agent state
3. **No new APIs needed** — everything is read/write/ls for the target users
4. **Latency is manageable** — async patterns (inbox/output) feel natural
5. **Unix users can monitor agents** without reading documentation

### 11.2 Negative Hypotheses (Watch For These)

1. The filesystem metaphor requires constant explanation
2. Operations that should be instant block (latency mismatch)
3. AI agents struggle with async patterns (inbox → poll → output)
4. Standard tools produce confusing results
5. It's easier to just write Python
6. Day 1 concepts don't map cleanly and feel forced

### 11.3 Questions a PoC Would Answer

- Does FUSE add value or just complexity?
- Which history management option (§5) works best?
- Can Day 1 concepts live in the daemon beneath FUSE?
- Is the target user (AI CLI agent) actually better served this way?
- What's the minimum viable structure?

---

## 12. Conclusion (Provisional)

Day 1 built an operating system abstraction. Day 2 explores whether a **filesystem interface** is a better foundation—or a complement.

The core insight: AI CLI agents are a primary user, and they already know filesystems. This may mean:
- Some Day 1 abstractions become **literal** (context as files, process state as files)
- Some become **redundant** (custom shell replaced by bash + FUSE)
- Some become **simpler** (Unix permissions instead of capability leases)
- Some remain **necessary** (signals for long-running operations?)

**Working hypothesis:** The filesystem is a UI layer, and the Day 1 concepts may live in the daemon beneath it.

**End-of-day re-evaluation required for:**
- Custom shell (ash) — redundant or complementary?
- Three-level security — maps to Unix permissions or needs more?
- Signals — control files suffice or need real signals?
- Context binding (`|=`) — filesystem-expressible or daemon API?
- Ring model — natural FUSE/daemon split or forced abstraction?

Keep building. Keep questioning. Revisit after more exploration.

---

*Day 2, Session 1 completed: 2025-01-04*
*Focus: Honest assessment, FUSE exploration, impedance mismatch solutions, practical design*
*Status: Concepts flagged for re-evaluation, not discarded*

---

# Agent OS: Day 2, Session 2 - Deep Exploration

**Status:** Brainstorming Document / Wild Ideas Welcome
**Context:** Building on Session 1's FUSE pivot. Exploring extensions.
**Rule:** 99 bad ideas with 1 good one = success. Defer all decisions.

---

> Day 2, Session 2

## 1. Session 1 Recap and Gaps

Session 1 established:
- FUSE as interface layer (AI agents know filesystems)
- Impedance mismatches are solvable (inbox/output/stream)
- Day 1 concepts may live in daemon beneath FUSE

Session 1 underdeveloped:
- Conversations structure (flat JSONL isn't enough)
- Channels concept (human approval, external destinations)
- External mounts (APIs as filesystems)
- How FUSE and Day 1 actually layer

This session explores these gaps plus wilder ideas.

---

## 2. Conversations: The Compounding Asset

### 2.1 The Insight

Every agent interaction is a reusable artifact:
- Context that can be resumed
- Knowledge that can be searched
- Decisions that can be audited
- Patterns that can be learned from

A flat log loses this. A structured filesystem preserves it.

### 2.2 Proposed Structure

```
/aos/conversations/
├── 2025/
│   └── 01/
│       └── 04/
│           ├── a3f8c2/
│           │   ├── meta.json           # Metadata
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
│           │   └── cost.json           # Spend breakdown
│           │
│           └── b7c2d1/
│               └── ...
│
├── by-tag/                             # Symlink views
│   ├── research/
│   │   └── a3f8c2 -> ../../2025/01/04/a3f8c2
│   ├── coding/
│   └── debugging/
│
├── by-agent/
│   ├── researcher/
│   └── coder/
│
├── by-project/
│   └── fusion-analysis/
│
├── starred/                            # User-curated important convos
│
└── active/                             # Symlinks to in-progress
    └── current -> ../2025/01/04/c8d3e2
```

### 2.3 What This Enables

**Resume any conversation:**
```bash
# See where we left off
cat /aos/conversations/2025/01/03/xyz789/transcript.md

# Resume with full context
aos resume /aos/conversations/2025/01/03/xyz789/

# Or explicitly in new agent call
echo "continue this analysis" > /aos/agents/researcher/inbox \
  --context=/aos/conversations/2025/01/03/xyz789/context.ctx
```

**Fork explorations:**
```bash
# Branch from a decision point
aos fork /aos/conversations/2025/01/03/xyz789/ \
  --prompt="but what if we assumed 10% growth instead?"

# Creates new conversation with xyz789 as parent
# Parent link preserved in meta.json
```

**Search everything:**
```bash
# Find all conversations about fusion
grep -r "fusion" /aos/conversations/

# Find conversations where we approved deletions
grep -r "fs.delete" /aos/conversations/*/decisions.jsonl

# Find high-cost conversations
find /aos/conversations -name "cost.json" -exec sh -c \
  'jq -e ".total_usd > 1.0" "$1" >/dev/null && echo "$1"' _ {} \;
```

**Git your history:**
```bash
cd /aos/conversations
git init
git add .
git commit -m "Initial history"

# Now you can:
git log --oneline --all  # See all conversations
git diff HEAD~10         # What changed in last 10 convos
git blame 2025/01/04/a3f8c2/transcript.md  # Who said what when
```

**Build memory from history:**
```bash
# Extract all facts learned
find /aos/conversations -name "transcript.jsonl" -exec \
  jq -r 'select(.type=="fact") | .content' {} \; \
  > /aos/memory/extracted_facts.jsonl

# Or use an agent to synthesize
cat /aos/conversations/by-project/fusion-analysis/*/transcript.md | \
  @summarizer "extract key findings" > /aos/memory/projects/fusion/synthesis.md
```

### 2.4 The meta.json Structure

```json
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

### 2.5 Auto-Generated Summary

Every conversation gets an auto-summary on completion:

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

This summary is searchable, readable, and useful for quick triage.

### 2.6 Live Conversations

While running, conversations exist in both places:

```
/aos/procs/1234/
├── status
├── stdin
├── stdout
├── stderr
└── conversation -> /aos/conversations/2025/01/04/a3f8c2

/aos/conversations/active/
└── a3f8c2 -> ../2025/01/04/a3f8c2
```

On completion:
- `/aos/procs/1234/` disappears
- `/aos/conversations/active/a3f8c2` symlink removed
- Conversation remains in chronological location
- `summary.md` generated

### 2.7 Questions to Explore

- Should conversations be immutable after completion?
- How to handle conversation branching UI?
- Should `context.ctx` be human-readable or binary?
- Tag management: freeform or controlled vocabulary?
- Retention policy: auto-archive? auto-delete?

---

## 3. Channels: Destinations as Files

### 3.1 The Insight

Day 1 had complex machinery for human approval, sanitization, signing. What if it's just:

```bash
# Request human approval
echo "Delete production database?" > /aos/channels/human/alice
response=$(cat /aos/channels/human/alice)  # Blocks until response
```

Write to request. Read blocks until response. The channel handles the mechanics.

### 3.2 Proposed Structure

```
/aos/channels/
├── terminal                    # Default stdout (current terminal)
│
├── human/                      # Human approval channels
│   ├── alice                   # Write = request, read = response
│   ├── bob
│   └── ops-team               # Group channel
│
├── slack/
│   ├── general                # Write = post message
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
│   └── custom_endpoint
│
├── queues/                     # Async job queues
│   ├── batch_processing
│   └── nightly_reports
│
└── null                        # /dev/null equivalent
```

### 3.3 Human Approval Flow

**Simple approval:**
```bash
# Agent needs permission
echo '{"action":"fs.delete","path":"/important/*","reason":"cleanup"}' \
  > /aos/channels/human/alice

# Alice sees notification (Slack, email, desktop—configured elsewhere)
# Alice responds via UI or CLI
# Channel file becomes readable

response=$(cat /aos/channels/human/alice)
# {"approved": true, "signer": "alice@ed25519:...", "timestamp": "..."}
```

**Batch approval:**
```bash
# Multiple requests
echo '{"batch":[{"action":"fs.delete","path":"/tmp/1"},...]}' \
  > /aos/channels/human/ops-team

# Any ops-team member can approve
```

**Timeout handling:**
```bash
# Non-blocking check
if timeout 60 cat /aos/channels/human/alice > /tmp/response 2>/dev/null; then
    # Approved
else
    # Timed out, handle gracefully
fi
```

### 3.4 Notification Channels

```bash
# Alert to Slack
echo "Build failed: $error" > /aos/channels/slack/alerts

# Email report
cat /aos/conversations/2025/01/04/a3f8c2/summary.md \
  > /aos/channels/email/boss@company.com

# Trigger webhook
echo '{"event":"completed","id":"a3f8c2"}' > /aos/channels/webhooks/datadog
```

These are fire-and-forget. Write returns immediately.

### 3.5 Channel Configuration

Where does the channel know how to reach Slack? Configuration files:

```yaml
# /aos/etc/channels/slack.yaml
provider: slack
credentials: ${SLACK_BOT_TOKEN}  # From environment
channels:
  general: C01234567
  alerts: C89012345
  dev-team: C45678901
default_channel: general
```

```yaml
# /aos/etc/channels/human.yaml
provider: human
notification:
  method: slack  # or email, desktop, all
  channel: alerts
timeout_sec: 3600
require_signature: true
```

### 3.6 Agent Output Routing

Agents can write to channels:

```bash
# Agent config specifies output channel
cat /aos/agents/reporter/config.yaml
# output_channel: /aos/channels/slack/daily-reports

# Or override at invocation
echo "Generate daily report" > /aos/agents/reporter/inbox \
  --output=/aos/channels/email/team@company.com
```

### 3.7 Chaining Through Channels

```bash
# Agent A produces output, Agent B consumes it
# Instead of pipes, use channel as rendezvous

echo "Research fusion" > /aos/agents/researcher/inbox \
  --output=/aos/channels/queues/research_results

# Separate process (or cron job)
cat /aos/channels/queues/research_results | \
  while read result; do
    echo "Summarize: $result" > /aos/agents/writer/inbox
  done
```

### 3.8 Questions to Explore

- Should channels be push (write-only) or bidirectional?
- How to handle channel backpressure?
- Should human channels have memory (conversation)?
- Group channels: any member approves, or quorum?
- Channel discovery: `ls /aos/channels/available/`?

---

## 4. External Mounts: APIs as Filesystems

### 4.1 The Insight

The rejected "semantic FUSE" was bad because reads triggered hidden computation. But what about **explicit mounts** where the mapping is predictable?

```bash
# Explicit mount with known semantics
aos mount github --token=$GITHUB_TOKEN /aos/mnt/github

# Now operations have predictable meanings
cat /aos/mnt/github/issues/123              # GET /repos/.../issues/123
echo "Fixed" >> /aos/mnt/github/issues/123/comments  # POST comment
echo "closed" > /aos/mnt/github/issues/123/state     # PATCH state
```

This isn't magic—it's a documented protocol. Like NFS, but for APIs.

### 4.2 Proposed Mount Points

```
/aos/mnt/
├── github/
│   ├── issues/
│   │   ├── 123/
│   │   │   ├── body           # R/W: issue body
│   │   │   ├── state          # R/W: open/closed
│   │   │   ├── labels/        # R/W: add/remove labels
│   │   │   ├── comments/      # R: list, W: add comment
│   │   │   │   ├── 1
│   │   │   │   └── 2
│   │   │   └── meta.json      # R: author, created, etc.
│   │   └── new                # W: create issue
│   │
│   ├── pulls/
│   │   └── 456/
│   │       ├── diff           # R: the diff
│   │       ├── files/         # R: changed files list
│   │       └── reviews/
│   │
│   └── repos/
│       └── owner/
│           └── repo/
│               ├── README.md   # R/W: actual file
│               └── src/
│
├── linear/
│   ├── issues/
│   └── projects/
│
├── postgres/
│   └── mydb/
│       ├── users/             # R: list users
│       │   ├── schema         # R: table schema
│       │   ├── query          # W: write SQL, R: results
│       │   └── 123.json       # R: user ID 123
│       └── orders/
│
├── s3/
│   └── my-bucket/
│       ├── data/
│       │   └── file.csv       # R/W: actual S3 object
│       └── logs/
│
├── slack/
│   └── channels/
│       └── general/
│           ├── messages       # R: history, W: post
│           └── topic          # R/W: channel topic
│
└── web/                        # Generic HTTP
    └── api.example.com/
        └── v1/
            └── data.json      # R: GET, W: POST
```

### 4.3 Mount Command

```bash
# Mount GitHub
aos mount github \
  --token=$GITHUB_TOKEN \
  --repo=owner/repo \
  /aos/mnt/github

# Mount S3
aos mount s3 \
  --bucket=my-bucket \
  --region=us-east-1 \
  /aos/mnt/s3/my-bucket

# Mount Postgres (read-only by default)
aos mount postgres \
  --connection="postgres://user:pass@host/db" \
  --readonly \
  /aos/mnt/postgres/mydb

# Mount generic REST API
aos mount http \
  --base-url=https://api.example.com/v1 \
  --auth-header="Bearer $TOKEN" \
  /aos/mnt/web/api.example.com
```

### 4.4 Operation Semantics

Document explicitly what each operation does:

| Operation | GitHub Issues | S3 | Postgres |
|-----------|--------------|-----|----------|
| `cat file` | GET resource | GET object | SELECT row |
| `echo > file` | PUT/PATCH | PUT object | INSERT/UPDATE |
| `echo >> file` | POST (comment) | Append (if supported) | INSERT |
| `rm file` | DELETE/close | DELETE object | DELETE row |
| `ls dir` | List resources | List objects | List tables/rows |
| `mkdir` | Create resource | Create prefix | N/A |

### 4.5 Cost Visibility

Mounts can have cost:

```bash
# Mount with cost tracking
aos mount github --token=$TOKEN /aos/mnt/github

# Check mount cost
cat /aos/mnt/github/.aos/cost
# {"api_calls": 47, "rate_limit_remaining": 4953}

# S3 with cost
cat /aos/mnt/s3/my-bucket/.aos/cost
# {"get_requests": 123, "put_requests": 7, "bytes_transferred": 1547234}
```

### 4.6 Agent Use Cases

**AI triages GitHub issues:**
```bash
# AI can use standard tools
ls /aos/mnt/github/issues/
# 123  124  125  126

for issue in /aos/mnt/github/issues/*/; do
  content=$(cat "$issue/body")
  labels=$(ls "$issue/labels/")
  
  # AI decides based on content
  if should_label_bug "$content"; then
    echo "bug" > "$issue/labels/bug"
  fi
done
```

**AI queries database:**
```bash
echo "SELECT * FROM users WHERE active = true LIMIT 10" \
  > /aos/mnt/postgres/mydb/users/query

cat /aos/mnt/postgres/mydb/users/query
# [{"id": 1, "name": "Alice"}, ...]
```

**AI backs up to S3:**
```bash
cp /aos/conversations/2025/01/04/a3f8c2/transcript.jsonl \
   /aos/mnt/s3/backups/conversations/a3f8c2.jsonl
```

### 4.7 Why This Isn't "Magic" FUSE

The rejected semantic FUSE: `cat /knowledge/fusion` triggers hidden RAG, embedding computation, LLM summarization. Cost unknown. Mechanism opaque.

This: `cat /aos/mnt/github/issues/123` does GET /issues/123. One API call. Known cost. Documented behavior.

The difference is **predictability**. An AI (or human) can reason about what operations cost and do.

### 4.8 Questions to Explore

- Which APIs are worth implementing mounts for?
- How to handle pagination (ls returns 100 items but there are 10,000)?
- How to handle API rate limits?
- Should writes be synchronous or queued?
- How to expose API-specific features (GitHub reactions, S3 metadata)?
- Is this actually better than just giving agents API tools?

---

## 5. The Layer Relationship (Clarification)

### 5.1 The Confusion

Session 1 repeatedly asked: "Do Day 1 concepts survive? Are they redundant with FUSE?"

The answer: **They're different layers.**

```
┌─────────────────────────────────────────────────────────────┐
│                      User Interfaces                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │  FUSE       │  │  Python SDK  │  │  REST API         │   │
│  │  /aos/*     │  │  agent_os.py │  │  api.aos.local    │   │
│  └──────┬──────┘  └──────┬───────┘  └─────────┬─────────┘   │
│         │                │                    │              │
│         └────────────────┼────────────────────┘              │
│                          │                                   │
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                   Agent Daemon                         │  │
│  │                                                        │  │
│  │   Day 1 Concepts:                                      │  │
│  │   - Process model (spawn, wait, kill)                  │  │
│  │   - Signals (SIGTERM, SIGCTXPRESSURE, SIGCAPDROP)     │  │
│  │   - Capabilities (intersection, leases)                │  │
│  │   - Security colors (BLUE/YELLOW/RED)                  │  │
│  │   - Context management (Merkle DAG, checkpoints)       │  │
│  │   - Transactional side effects                         │  │
│  │   - Execution traces                                   │  │
│  │   - Budget/velocity enforcement                        │  │
│  │                                                        │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Model Backends                        │  │
│  │   vLLM  │  SGLang  │  OpenAI  │  Anthropic  │  Ollama │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 How FUSE Maps to Day 1

| FUSE Operation | Daemon Action |
|----------------|---------------|
| `echo "stop" > /aos/procs/1234/ctl` | Daemon sends SIGTERM to process |
| `cat /aos/procs/1234/status` | Daemon reports process state |
| `cat /aos/procs/1234/capabilities` | Daemon reports effective caps after intersection |
| `chmod 000 /aos/agents/dangerous/inbox` | Daemon enforces no-execute capability |
| `cat /aos/procs/1234/stderr` | Daemon streams execution trace |
| `echo "msg" > /aos/channels/human/alice` | Daemon initiates transactional side effect approval |

### 5.3 What This Means

- FUSE is a **view layer**. It exposes state.
- Daemon is the **execution layer**. It enforces rules.
- Day 1 concepts **live in the daemon**.
- FUSE makes them **accessible via filesystem**.

Custom shell (ash)? Maybe redundant—bash + FUSE might suffice.
Day 1 process model? **Not redundant**—it's what the daemon implements.

### 5.4 The Ring Model Revisited

Day 1 Ring Model:
- Ring 0: Kernel (doesn't interpret text)
- Ring 1: Userland tools (sanitize, transcode)
- Ring 2: Agent processes (LLM inference)

FUSE mapping:
- Ring 0: Agent Daemon (enforces, doesn't interpret)
- Ring 1: FUSE layer + CLI tools (expose state, accept commands)
- Ring 2: Agent processes (still LLM inference)

The daemon **is** Ring 0. FUSE **is** the syscall interface.

---

## 6. Wilder Ideas (Exploratory)

These might be terrible. That's fine.

### 6.1 Agents as Users

What if agents are Unix users?

```bash
# Each agent runs as its own user
ps aux | grep aos
# researcher  1234  ...  aos-agent
# coder       1235  ...  aos-agent
# critic      1236  ...  aos-agent

# Standard Unix permissions
ls -la /aos/memory/project/
# drwxr-x---  researcher  researcher  secrets/
# drwxr-xr-x  researcher  agents      public/

# Agent "coder" can't read researcher's secrets
# Because Unix permissions
```

Benefits:
- Leverage existing Unix permission model
- Standard tools for permission management
- Process isolation via OS-level mechanisms
- Audit via standard Unix logging

Questions:
- How to create/manage agent users?
- Does this complicate deployment?
- Container implications?

### 6.2 Conversations as Git Branches

What if the conversation tree is literally a git repo?

```bash
cd /aos/conversations

git log --graph --oneline
# * a3f8c2 (HEAD) Follow-up on fusion timeline
# * b7c2d1 Explored cost implications
# | * c8d3e2 Alternative: focus on private fusion
# |/
# * xyz789 Initial fusion research

git checkout c8d3e2
# Now agent context is the "private fusion" branch

git merge b7c2d1
# Context merge? What does this even mean?
```

Benefits:
- Existing tooling (git log, git diff, git blame)
- Natural branching model
- Built-in versioning

Questions:
- Does merge make sense for contexts?
- Is this confusing or intuitive?
- Performance implications?

### 6.3 Pub/Sub via Filesystem

```
/aos/topics/
├── research/
│   ├── publishers           # Who can publish
│   ├── subscribers          # Who receives
│   └── messages/            # Message queue
│       ├── 001.json
│       └── 002.json
│
└── alerts/
    ├── publishers
    ├── subscribers
    └── messages/
```

```bash
# Publisher
echo '{"event":"discovery","data":"..."}' > /aos/topics/research/messages/new

# Subscriber (blocking read)
inotifywait -m /aos/topics/research/messages/ | while read event; do
  # Process new message
done
```

Benefits:
- Decoupled agent communication
- Standard tools (inotifywait)
- Natural fit for event-driven workflows

Questions:
- Is this better than just using message queues?
- How to handle message acknowledgment?
- Ordering guarantees?

### 6.4 Capabilities as Extended Attributes

```bash
# Set capabilities via xattr
setfattr -n user.aos.capabilities -v "read,write" /aos/agents/coder/

# Read capabilities
getfattr -n user.aos.capabilities /aos/agents/coder/
# user.aos.capabilities="read,write"

# Security color as xattr
setfattr -n user.aos.color -v "YELLOW" /aos/conversations/a3f8c2/

# Query by attribute
find /aos/conversations -xattr user.aos.color=RED
```

Benefits:
- Native filesystem feature
- No special files needed
- Standard tools work (getfattr, setfattr)

Questions:
- xattr support varies by filesystem
- Is this intuitive?
- Backup/restore implications?

### 6.5 Agent Definitions as Executables

```bash
# Agent definition is executable
cat /aos/agents/researcher
#!/usr/bin/env aos-agent
# model: claude-sonnet-4-5-20250514
# capabilities: web.search, fs.read
# max_cost_usd: 1.00

You are a research assistant...

# Run directly
/aos/agents/researcher "What is fusion?"

# Or via standard mechanisms
echo "query" | /aos/agents/researcher

# Shebangs work
cat > /aos/scripts/research.sh << 'EOF'
#!/aos/agents/researcher
Study the economic implications of fusion energy.
EOF
chmod +x /aos/scripts/research.sh
./aos/scripts/research.sh
```

Benefits:
- Agents are just programs
- Compose with standard shell
- Familiar execution model

Questions:
- Does this break the inbox/output model?
- How to handle streaming output?
- How does context persistence work?

### 6.6 Time-Travel Debugging

```
/aos/procs/1234/
├── history/
│   ├── t_0000/                 # Initial state
│   │   ├── context
│   │   └── trace.jsonl
│   ├── t_0001/                 # After first tool call
│   │   ├── context
│   │   └── trace.jsonl
│   └── t_0002/
│
└── restore                     # W: write timestamp to restore

# Restore to earlier state
echo "t_0001" > /aos/procs/1234/restore

# Process rewinds to that checkpoint
```

Benefits:
- Debug by rewinding
- Fork from any point
- Understand decision paths

Questions:
- Storage implications?
- Which checkpoints to keep?
- How does this interact with side effects?

### 6.7 Agent Pipes (Literal)

What if agents ARE pipes?

```bash
# Create named pipe backed by agent
mkfifo -m 0644 /aos/pipes/researcher

# Configure it
echo "model: claude-sonnet" > /aos/pipes/researcher.conf
echo "You are a researcher..." > /aos/pipes/researcher.prompt

# Use like any pipe
cat questions.txt > /aos/pipes/researcher &
cat /aos/pipes/researcher > answers.txt &
```

Benefits:
- Pure Unix abstraction
- Standard tools work
- Natural composition

Questions:
- How to handle multi-turn?
- How to handle slow responses?
- How to configure capabilities?

### 6.8 /aos/future/ - Speculative Pre-computation

```
/aos/future/
└── predictions/
    ├── researcher_likely_next.ctx    # Pre-computed likely next context
    └── coder_common_patterns.ctx     # Cached common invocations

# Daemon predicts likely next queries
# Pre-warms KV cache
# If prediction correct: instant response
# If wrong: discarded, compute normally
```

Benefits:
- Latency reduction
- Leverage idle compute

Questions:
- Prediction accuracy?
- Cost of wrong predictions?
- This was partially rejected in Day 1—reconsider?

### 6.9 Filesystem-Level Transactions

```bash
# Start transaction
echo "begin" > /aos/transaction

# Multiple operations
echo "Research fusion" > /aos/agents/researcher/inbox
echo "Review findings" > /aos/agents/critic/inbox

# Commit atomically (both succeed or both fail)
echo "commit" > /aos/transaction

# Or rollback
echo "rollback" > /aos/transaction
```

Benefits:
- Atomic multi-agent operations
- Consistent state

Questions:
- What does rollback even mean for LLM calls?
- Is this useful or over-engineered?
- How to handle partial completion?

### 6.10 /aos/www/ - Web Interface as Filesystem

```
/aos/www/
├── index.html              # Auto-generated dashboard
├── agents/
│   └── researcher/
│       └── index.html      # Agent status page
├── conversations/
│   └── a3f8c2/
│       └── index.html      # Conversation view
└── api/                    # REST endpoints as files
    └── v1/
        └── agents.json
```

```bash
# Serve directly
python -m http.server --directory /aos/www 8080

# Or nginx
location /aos/ {
    alias /aos/www/;
}
```

Benefits:
- Web UI for free
- Standard web servers work
- API is just files

Questions:
- How dynamic?
- Security implications?
- Is this useful or gimmick?

---

## 7. Implementation Considerations

### 7.1 FUSE Library Options

| Library | Language | Notes |
|---------|----------|-------|
| fusepy | Python | Simple, good for prototyping |
| pyfuse3 | Python | Async, more performant |
| fuser | Rust | High performance, type safe |
| go-fuse | Go | Good concurrency model |

For PoC: fusepy (fast iteration)
For production: fuser or go-fuse

### 7.2 Daemon Architecture Options

**Option A: Single daemon, FUSE in-process**
```
[FUSE] <-> [Daemon] <-> [Backends]
```
Simple, but FUSE operations block daemon.

**Option B: Separate FUSE process**
```
[FUSE] <-IPC-> [Daemon] <-> [Backends]
```
More complex, but FUSE can't block daemon.

**Option C: FUSE per-agent**
```
[FUSE: researcher] <-> [Daemon] <-> [Backends]
[FUSE: coder]      <->     |
```
Isolation, but complex.

### 7.3 State Storage Options

Where does daemon store state?

**Option A: Filesystem (SQLite + files)**
- Simple, portable
- Works with backups
- Performance limits

**Option B: Embedded database (RocksDB)**
- Better performance
- More complex
- Custom backup

**Option C: External database (Postgres)**
- Scalable
- Deployment complexity
- Overkill for single-user?

For PoC: SQLite + files
Scale later if needed.

---

## 8. Updated Hypotheses

### 8.1 Positive Hypotheses (From Session 1 + New)

1. AI agents can discover and invoke agents using only filesystem operations
2. Standard Unix tools work naturally on agent state
3. **Conversations-as-directories enable resume, fork, search**
4. **Channels simplify human approval to read/write**
5. **External mounts provide predictable API access**
6. The filesystem interface is self-documenting (ls is the docs)

### 8.2 Negative Hypotheses (Watch For)

1. Filesystem metaphor requires constant explanation
2. Latency mismatches are worse than expected
3. **Conversation structure is too complex for simple use cases**
4. **Channels don't actually simplify approval flow**
5. **External mounts are slower than direct API calls**
6. The layering (FUSE over daemon) adds overhead without benefit

### 8.3 New Questions

- Is conversation resume actually valuable, or do users start fresh?
- Do AI agents actually struggle with APIs, or is filesystem overkill?
- Is mounting GitHub better than GitHub MCP tool?
- How much does the daemon need Day 1 complexity vs. simpler model?

---

## 9. End of Session 2

**Explored:**
- Conversations as structured directories (resume, fork, search)
- Channels as I/O destinations (human approval, notifications)
- External mounts (APIs as filesystems)
- FUSE/Daemon layer relationship
- 10 wilder ideas (some probably bad)

**Deferred:**
- All decisions
- Implementation choices
- Which wild ideas to pursue

**For Session 3:**
- Synthesize Sessions 1-2
- Identify minimum viable PoC scope
- Start making tentative decisions (still reversible)
- Consider: what's the smallest thing that tests the core hypotheses?

---

*Day 2, Session 2 completed: 2025-01-04*
*Status: Deep exploration complete, decisions pending*