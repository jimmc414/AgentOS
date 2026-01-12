# Agent OS Quick Reference

Tables only. For rationale, see DECISIONS.md. For full specification, see SPEC.md.

---

## Signals

| Signal | Trigger | Default Handler |
|--------|---------|-----------------|
| `SIGTERM` | `kill PID` | Graceful shutdown, emit partial result |
| `SIGKILL` | `kill -9 PID` | Immediate termination |
| `SIGSTOP` | Debugger breakpoint, `kill -STOP` | Pause generation |
| `SIGCONT` | Debugger continue, `kill -CONT` | Resume generation |
| `SIGPIPE` | Downstream closed | Stop generating, exit 141 |
| `SIGCTXPRESSURE` | Context > 80% limit | Page out or summarize |
| `SIGXCPU` | Token budget exhausted | Exit 66 |
| `SIGBRAKE` | Spend velocity exceeded | Pause for authorization |
| `SIGCAPDROP` | Capability revoked | Degrade gracefully |
| `SIGCHLD` | Child completed | Reap zombie |

---

## Exit Codes

| Code | Name | Meaning |
|------|------|---------|
| 0 | SUCCESS | Completed |
| 1 | FAILURE | General error |
| 2 | INVALID_INPUT | Malformed request |
| 64 | REFUSED | Capability violation |
| 65 | CTX_OVERFLOW | Context limit exceeded |
| 66 | BUDGET_EXHAUSTED | Token/cost limit |
| 67 | UPSTREAM_FAILURE | Dependency failed |
| 68 | LOW_CONFIDENCE | Declined due to uncertainty |
| 69 | CONTAMINATED | Color violation |
| 70 | LEASE_EXPIRED | Capability lease invalid |
| 71 | RATE_LIMITED | Provider QPS/TPM exceeded |
| 141 | SIGPIPE | Downstream closed (128 + 13) |

---

## Operators

| Operator | Name | Semantics |
|----------|------|-----------|
| `\|` | Byte pipe | UTF-8 stream. Dumb conduit. Backpressure supported. |
| `\|=` | Context bind | Binds B's initial context to A's final state. NOT a stream. A must finalize before B starts. |
| `&` | Background | Run process in background. Returns immediately with PID. |
| `>` | Redirect stdout | Write stdout to file. Standard Unix semantics. |
| `2>` | Redirect stderr | Write execution trace to file. |
| `2>&1` | Merge streams | Combine execution trace with output. |
| `@` | Agent address | Invoke agent by name. `@researcher "query"` |

---

## Security Levels

| Level | Name | Capabilities | Typical Source |
|-------|------|--------------|----------------|
| BLUE | Trusted | All capabilities | System prompt, signed human input |
| YELLOW | Partial | Read, sandboxed write, no exec | Sanitized external, known-good APIs |
| RED | Untrusted | Read only, no side effects | Raw user input, untrusted web |

### Contamination Rules

```
BLUE + RED    -> RED      (untrusted fully contaminates)
BLUE + YELLOW -> YELLOW   (partial trust reduces to partial)
YELLOW + RED  -> RED      (untrusted contaminates partial)
```

### Promotion Paths

| From | To | Mechanism |
|------|----|-----------|
| RED | YELLOW | `sanitize --level=yellow` (light review) |
| RED | BLUE | `sanitize --level=blue` (full review + signature) |
| YELLOW | BLUE | `sanitize --level=blue` (full review + signature) |

---

## Ring Model

| Ring | Name | Responsibilities | Interprets Text? |
|------|------|------------------|------------------|
| 0 | Kernel | Token routing, capability enforcement, signal delivery | No |
| 1 | Userland Tools | Sanitization, format conversion, policy decisions | Yes |
| 2 | Agent Processes | LLM inference, tool execution, reasoning | Yes |

---

## Backend Compatibility

| Backend | Agent Unix | KV-Cache Fork | Shared Prefixes | Context Bind |
|---------|------------|---------------|-----------------|--------------|
| OpenAI API | Full | No | Prompt caching (approx) | Text serialization |
| Anthropic API | Full | No | Prompt caching (approx) | Text serialization |
| Google Gemini | Full | Partial | Context caching (coarse) | Text serialization |
| vLLM | Full | Yes | Yes | O(1) pointer |
| SGLang | Full | Yes | Yes | O(1) pointer |
| llama.cpp | Full | Yes | Yes | O(1) pointer |
| Ollama | Full | Partial | No | Text serialization |

### Feature Availability by Layer

| Feature | Agent Unix (everywhere) | Kernel Accelerators (KV backends) |
|---------|------------------------|-----------------------------------|
| spawn/wait/kill | Yes | Yes |
| Signals | Yes | Yes |
| Byte pipes | Yes | Yes |
| Capability enforcement | Yes | Yes |
| Session Coloring | Yes | Yes |
| Execution trace | Yes | Yes |
| Context bind `\|=` | Text fallback (~2s) | O(1) (~10ms) |
| Shared prefixes `.sc` | No | Yes |
| Fast checkpoint | No | Yes |
| Tree-of-Thoughts branching | Expensive | Cheap |

---

## Resource Limits

| Resource | Signal | Exit Code | Limit Type |
|----------|--------|-----------|------------|
| Total tokens | SIGXCPU | 66 | Hard ceiling |
| Total cost (USD) | SIGXCPU | 66 | Hard ceiling |
| Spend velocity ($/min) | SIGBRAKE | - | Pause for auth |
| Context size | SIGCTXPRESSURE | 65 | Soft warning at 80%, hard at 95% |
| Tool calls | - | 67 | Hard ceiling |
| Provider QPS | - | 71 | Queue or fail |
| Provider TPM | - | 71 | Queue or fail |

---

## Provenance Filesystem

```
/proc/$pid/provenance/
+-- inputs/
|   +-- prompt.txt
|   +-- context_parent.ref
|   +-- retrieved/
+-- tools/
|   +-- call_001.json
|   +-- ...
+-- artifacts/
|   +-- output.txt
|   +-- files/
+-- decisions.jsonl
+-- manifest.json
```

---

## Context Management Tools

| Tool | Function |
|------|----------|
| `ctx_pageout` | Page context to vector store |
| `ctx_pagein` | Retrieve paged content |
| `ctx_merge` | Combine multiple contexts |
| `ctx-last` | Shell builtin: return handle to last process's final context |

---

## JSONL Event Types (Convention)

| Type | Fields | Description |
|------|--------|-------------|
| `text` | content, final? | Output text fragment |
| `citation` | claim, source, confidence | Source attribution |
| `tool_call` | tool, args | Tool invocation |
| `tool_result` | tool, status, content | Tool response |
| `artifact` | name, hash, path | Created file reference |
| `confidence` | overall, factors | Output confidence score |
| `error` | code, message | Error event |
