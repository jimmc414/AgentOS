# Agent OS

A Unix-native runtime for LLM agents. Agents interact via filesystem paths and CLI commands rather than framework-specific APIs.

## Thesis

LLM coding agents learned to use computers by reading shell sessions, scripts, and Unix documentation. They already know:

```bash
cat /path/to/file
echo "content" > /path/to/file
ls /directory
grep "pattern" /path/**
command --flag=value
```

Agent OS exposes agent orchestration, memory, and external services through these primitives. No new SDK to document. No novel interaction patterns to learn.

## Quick Example

```bash
# Mount the filesystem
aos-mount /aos

# Invoke an agent
echo "What is the current state of fusion energy?" > /aos/agents/researcher/inbox

# Or use CLI (preferred for scripts)
aos invoke researcher "What is the current state of fusion energy?"
aos invoke researcher --wait --model=opus --budget=1.00 "..."

# Check status
cat /aos/agents/researcher/status   # "running"
cat /aos/agents/researcher/cost     # "0.23"

# Get output
cat /aos/agents/researcher/output

# Search conversation history
grep -r "ignition" /aos/conversations/

# Resume a conversation
aos resume /aos/conversations/2025/01/04/a3f8c2/

# Fork an exploration
aos fork /aos/conversations/2025/01/04/a3f8c2/ \
  --prompt="what if we assumed different growth rates?"
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      FUSE Filesystem                        │
│                        /aos/*                               │
│  Agents, Processes, Conversations, Memory, Channels, Mounts │
└─────────────────────────────┬───────────────────────────────┘
                              │ read/write/poll
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Agent Daemon                           │
│                                                             │
│  Process Management    │  Security Enforcement              │
│  Context Management    │  Budget Tracking                   │
│  Execution Traces      │  Backend Abstraction               │
└─────────────────────────────┬───────────────────────────────┘
                              │ inference calls
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Model Backends                          │
│   vLLM  │  SGLang  │  llama.cpp  │  OpenAI  │  Anthropic   │
└─────────────────────────────────────────────────────────────┘
```

**FUSE Layer:** Exposes state as files. Translates filesystem operations to daemon calls.

**Daemon Layer:** Manages execution, enforces security, tracks resources.

**Backend Layer:** Abstracts model providers. Handles inference and KV-cache operations.

## Filesystem Layout

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
└── system/              # Runtime status and configuration
```

### Agents

Agents are directories:

```
/aos/agents/researcher/
├── config.yaml          # Model, prompt, capabilities (read-only)
├── status               # idle | running | error
├── cost                 # Current session spend (USD)
├── inbox                # Write to invoke
├── output               # Last completed response
├── stream               # Live token stream (tail -f)
├── session              # Current session ID
└── log                  # Append-only interaction history
```

### Conversations

Every interaction is preserved:

```
/aos/conversations/2025/01/04/a3f8c2/
├── meta.json            # Metadata, lineage, cost
├── transcript.md        # Human-readable record
├── transcript.jsonl     # Machine-parseable record
├── context.ctx          # Restorable context state
├── summary.md           # Auto-generated summary
├── artifacts/           # Files produced
└── decisions.jsonl      # Approvals granted
```

Version control works as expected:

```bash
cd /aos/conversations
git init && git add . && git commit -m "Initial history"
```

### Process Management

```bash
# List processes
aos ps
ls /aos/procs/

# Stop a process
echo "stop" > /aos/procs/1234/ctl
aos kill 1234

# Wait for completion
aos wait 1234

# View live trace
tail -f /aos/procs/1234/stderr
```

### External Mounts

```bash
# Mount external services
aos mount github --token=$TOKEN /aos/mnt/github
aos mount postgres --connection=$CONN /aos/mnt/db

# Read operations work as expected
cat /aos/mnt/github/issues/123
ls /aos/mnt/s3/data/

# Writes use CLI for safety
aos write /aos/mnt/github/issues/123/state --value="closed"
```

## Cost Controls

Background indexers (Spotlight, Windows Search, VS Code file watchers) can trigger catastrophic API costs. Agent OS implements defense in depth:

| Layer | Scope | Default Limit |
|-------|-------|---------------|
| Per-PID | Single process | $0.10/60s |
| Per-Session | Process group | $1.00/session |
| Per-UID | User aggregate | $1.00/60s |
| Per-Subtree | Mount paths | Configurable |
| Global | Emergency stop | $10.00/60s |

Expensive paths are locked by default:

```bash
cat /aos/mnt/github/issues/1
# Permission denied

aos unlock /aos/mnt/github --duration=30m
cat /aos/mnt/github/issues/1
# (content)
```

Cost estimation before execution:

```bash
aos fork /aos/conversations/huge-ctx --prompt="test" --dry-run
# Estimated cost: $0.60 (range: 0.45-0.75)
```

## Backend Compatibility

| Backend | FUSE Operations | KV-Cache Fork | Context Bind |
|---------|----------------|---------------|--------------|
| OpenAI API | Full | No | Text serialization |
| Anthropic API | Full | No | Text serialization |
| vLLM | Full | Yes | O(1) pointer |
| SGLang | Full | Yes | O(1) pointer |
| llama.cpp | Full | Yes | O(1) pointer |

When KV-cache operations are unavailable, the daemon falls back to text serialization. Semantics are preserved; performance is degraded.

## Design Principles

1. **Match training patterns.** If agents learned it from bash/Unix, use it. Novel patterns require justification.

2. **Filesystem for reading, CLI for writing.** Reading via `cat` is safe. Writes with side effects benefit from CLI flags (`--wait`, `--dry-run`, `--force`).

3. **Both interfaces are first-class.** `echo > inbox` and `aos invoke` are equally valid.

4. **Conversations are assets.** Searchable, resumable, forkable, version-controllable.

5. **Cost is a first-class resource.** Budget limits are as fundamental as memory limits.

6. **Capabilities intersect down.** A caller cannot escalate privileges through an agent.

7. **Fail explicit.** Errors identify which component failed and why.

## Limitations

**FUSE complexity.** Running a FUSE filesystem adds operational overhead. It requires appropriate permissions, can fail in ways that affect the mount, and adds a layer between your code and the daemon.

**KV-cache benefits require local backends.** The performance advantage of O(1) context sharing only applies when using vLLM, SGLang, or llama.cpp. Cloud APIs (OpenAI, Anthropic) fall back to text serialization.

**Not for simple use cases.** If you're building a single-agent CLI tool that talks to one API, a direct SDK call is simpler. Agent OS is designed for multi-agent orchestration, persistent workflows, and environments where audit trails and cost controls matter.

## When to Use This

- Multi-agent systems with complex coordination
- Workflows requiring conversation persistence and forking
- Environments needing audit trails and cost controls
- Self-hosted LLM deployments where KV-cache sharing provides real benefit
- Research on agent coordination patterns

## When Not to Use This

- Simple single-agent applications
- Quick scripts that make a few API calls
- Environments where FUSE is unavailable or impractical
- Cases where the operational overhead exceeds the benefit

## Status

This is a specification and reference implementation. The spec (SPEC_v2.3.md) is stable. Implementation is in progress.

## Related Work

- [Zed's agent abstraction](https://zed.dev/) - Editor-integrated agent system
- [Claude Code](https://github.com/anthropics/claude-code) - CLI tool with built-in tool interface
- [aider](https://github.com/paul-gauthier/aider) - Git-aware coding assistant
- [OpenInterpreter](https://github.com/OpenInterpreter/open-interpreter) - Natural language to code execution

Agent OS differs in treating the filesystem as the primary interface rather than a programmatic API. The hypothesis is that this reduces the documentation burden for LLM agents.

## License

MIT
