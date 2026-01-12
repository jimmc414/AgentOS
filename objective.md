# Agent OS: Project Objective

## What This Is

Agent OS is a Unix-native runtime for LLM agents. It exposes agent orchestration, memory, and external integrations through **filesystem paths and CLI commands** — the primitives that AI agents already understand.

## The Core Insight

AI coding agents (Claude Code, Codex CLI, aider, etc.) learned to use computers by reading millions of shell sessions, scripts, and Unix documentation. They already know:

```bash
cat /path/to/file              # Read content
echo "content" > /path/to/file # Write content  
ls /path/to/directory          # List contents
grep "pattern" /path/**        # Search
tail -f /path/to/log           # Stream
command --flag=value           # CLI with options
$?                             # Exit codes
```

**The design principle:** If an agent learned a pattern from training data, we use that pattern. If a pattern would be novel or surprising, we find a familiar alternative.

This means:
- **Filesystem paths** for reading state, discovering resources, simple writes
- **CLI commands** for complex operations, atomicity, reliability
- **Exit codes** for status (not novel signaling mechanisms)
- **Familiar flags** like `--wait`, `--dry-run`, `--force`

Both `echo "query" > inbox` and `aos invoke agent "query"` are valid. Use whichever is clearer for the task.

## The Vision

### Mount Agent OS, Get a Familiar Environment

```bash
aos-mount /aos
```

Now agents, memory, and integrations are accessible:

```
/aos/
├── agents/              # Agent definitions and state
├── procs/               # Running processes  
├── conversations/       # Complete interaction history
├── memory/              # Persistent knowledge
├── channels/            # I/O destinations (humans, services)
├── mnt/                 # External resources (APIs, databases)
└── system/              # Runtime status and configuration
```

### Agents Are Directories

```
/aos/agents/researcher/
├── config.yaml          # Model, prompt, capabilities, limits (read-only to agent)
├── status               # idle | running | error
├── cost                 # Current session spend
├── inbox                # Write to invoke (async, queues work)
├── output               # Last completed response
├── log                  # Append-only interaction history
└── session              # Current session ID
```

**Invoke an agent (filesystem):**
```bash
echo "What is the current state of fusion energy?" > /aos/agents/researcher/inbox
```

**Invoke an agent (CLI — preferred for scripts):**
```bash
aos invoke researcher "What is the current state of fusion energy?"
aos invoke researcher --wait "..." # Blocks until complete
aos invoke researcher --model=opus --budget=1.00 "..."
```

**Check status:**
```bash
cat /aos/agents/researcher/status   # "running"
cat /aos/agents/researcher/cost     # "0.23"
```

Both approaches work. CLI provides atomicity and flags; filesystem provides discoverability and familiar read/write patterns.

### Conversations Are Searchable, Resumable, Forkable

Every interaction is preserved as a directory:

```
/aos/conversations/2025/01/04/a3f8c2/
├── meta.json            # Metadata, lineage, cost, outcome
├── transcript.md        # Human-readable record
├── transcript.jsonl     # Machine-parseable record  
├── context.ctx          # Restorable context state
├── summary.md           # Auto-generated summary
├── artifacts/           # Files produced
├── tools/               # Tool calls made
└── decisions.jsonl      # Approvals granted
```

**Search all history:**
```bash
grep -r "fusion" /aos/conversations/
```

**Resume a conversation:**
```bash
aos resume /aos/conversations/2025/01/04/a3f8c2/
```

**Fork an exploration:**
```bash
aos fork /aos/conversations/2025/01/04/a3f8c2/ \
  --prompt="but what if we assumed different growth rates?"
```

**Version control everything:**
```bash
cd /aos/conversations
git init && git add . && git commit -m "Initial history"
```

### Channels Connect to Humans and Services

```
/aos/channels/
├── human/alice          # Human approval channel
├── slack/alerts         # Slack integration
├── email/team@co.com    # Email integration
└── webhooks/pagerduty   # Webhook integration
```

**Request human approval (CLI — recommended):**
```bash
aos approve --channel=human/alice --timeout=5m \
  '{"action":"delete","path":"/important"}'
```

**Send notification:**
```bash
echo "Task completed" > /aos/channels/slack/alerts
# or
aos notify slack/alerts "Task completed"
```

### External APIs Are Mounted

```bash
aos mount github --token=$TOKEN /aos/mnt/github
aos mount postgres --connection=$CONN /aos/mnt/db
aos mount s3 --bucket=my-bucket /aos/mnt/s3
```

Now standard read operations work:

```bash
cat /aos/mnt/github/issues/123              # GET issue
cat /aos/mnt/db/users/query.sql             # Run query from file
ls /aos/mnt/s3/data/                        # List bucket contents
```

Writes that modify remote resources use CLI for safety:
```bash
aos write /aos/mnt/github/issues/123/state --value="closed"
aos write /aos/mnt/github/issues/123/comments --append="Fixed in PR #456"
```

Or with explicit opt-in for direct writes:
```bash
aos mount github --token=$TOKEN --writable /aos/mnt/github
echo "closed" > /aos/mnt/github/issues/123/state  # Now works, use carefully
```

### Memory Is Searchable

```
/aos/memory/
├── global/
│   ├── facts.jsonl      # Append-only fact store
│   └── entities/        # Known entities
└── projects/
    └── fusion-analysis/
        ├── context.md
        └── findings.jsonl
```

**Search knowledge:**
```bash
grep -r "ignition" /aos/memory/
```

**Add to memory:**
```bash
echo '{"fact":"NIF achieved ignition Dec 2022"}' >> /aos/memory/global/facts.jsonl
```

### Process Management

Running agents appear in `/aos/procs/`:

```
/aos/procs/1234/
├── status               # running | stopped | completed | error
├── stdin                # Write to send input
├── stdout               # Read output  
├── stderr               # Execution trace (JSONL)
├── ctl                  # Write "stop", "kill", "pause"
├── exit_code            # Exit code when complete
├── budget/
│   ├── limit            # Spend limit
│   └── spent            # Current spend
└── conversation -> /aos/conversations/2025/01/04/a3f8c2
```

**Stop a process:**
```bash
echo "stop" > /aos/procs/1234/ctl
# or
aos kill 1234
```

**Wait for completion:**
```bash
aos wait 1234            # Blocks until done, returns exit code
```

**Monitor all processes:**
```bash
aos ps                   # Familiar interface
# or
ls /aos/procs/           # Directory listing
```

## Design Principles

1. **Match training patterns.** If agents learned it from bash/Unix, we use it. Novel patterns require justification.

2. **Filesystem for reading, CLI for writing.** Reading state via `cat` is always safe. Writes with side effects often benefit from CLI flags (`--wait`, `--dry-run`, `--force`).

3. **Both interfaces are first-class.** `echo > inbox` and `aos invoke` are equally valid. Use filesystem for simplicity, CLI for reliability.

4. **Predictable over elegant.** If a "file" would behave surprisingly (block forever, ignore flags, have hidden state), prefer an explicit CLI alternative.

5. **Exit codes and paths are the API.** Agents parse these reliably. Status in files, results in files, errors as exit codes.

6. **Explicit over magic.** Costs visible before execution. Destructive operations require confirmation. No silent failures.

7. **Conversations are assets.** Searchable, resumable, forkable, version-controllable. Full audit trail by default.

## The Daemon Beneath

A FUSE filesystem exposes state. A daemon manages execution:

- Process lifecycle (spawn, signal, terminate)
- Capability enforcement (agents can't exceed granted permissions)  
- Budget tracking (cost limits, velocity limits, pre-execution estimates)
- Context management (checkpoints, compression, resume)
- Security coloring (trusted/untrusted content separation)
- Execution traces (tool calls, decisions, costs)

The filesystem is the **read layer**. The CLI is the **control layer**. The daemon is the **execution layer**.

## What Success Looks Like

An AI agent, given a complex task, can:

1. **Discover** available agents: `ls /aos/agents/`
2. **Check** capabilities: `cat /aos/agents/*/config.yaml`
3. **Invoke** agents: `aos invoke researcher "task"` or `echo "task" > inbox`
4. **Wait** for completion: `aos wait $PID`
5. **Check** results: `cat /aos/agents/researcher/output`
6. **Search** history: `grep -r "relevant" /aos/conversations/`
7. **Resume** prior work: `aos resume /aos/conversations/.../`
8. **Request** approval: `aos approve --channel=human/operator "request"`
9. **Read** external data: `cat /aos/mnt/github/issues/123`
10. **Store** knowledge: `echo "fact" >> /aos/memory/project/facts.jsonl`

All using patterns it already learned from training data.

## What This Enables

- **Agent-to-agent orchestration** with familiar patterns
- **Complete audit trails** as searchable files  
- **Resumable workflows** via persistent context
- **Human-in-the-loop** via approval channels
- **External integration** via mounted APIs
- **Standard tooling** for monitoring, backup, version control
- **Testability** via `--dry-run` and mock modes

## Why Not Just an SDK?

SDKs require learning new APIs. Every agent framework invents its own:

```python
# Framework A
agent.run(prompt, callbacks=[...])

# Framework B  
result = await agent.execute({"input": prompt})

# Framework C
agent.invoke(prompt, config=AgentConfig(...))
```

An agent trained on Framework A's docs can't use Framework B without examples.

But *every* coding agent knows:
```bash
cat /path/to/file
echo "content" > /path/to/file
command --flag=value
```

Agent OS doesn't require new documentation in training data. It requires patterns already there.

---

*This document describes the aspirational vision. See spec.md for implementation details.*