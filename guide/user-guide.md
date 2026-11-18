# Headroom User Guide

Headroom reduces the number of input tokens sent to an AI model. It runs between your app or coding agent and the model, compressing repetitive and large content such as tool results, logs, JSON arrays, search results, files, and RAG chunks.

Headroom can be used as:

- a Python library;
- a TypeScript SDK plus local proxy;
- an OpenAI- or Anthropic-compatible HTTP proxy;
- a coding-agent wrapper;
- an MCP server.

The package name is `headroom-ai`. The Python import and CLI are `headroom`.

## Documentation overview

The repository's `docs/content/docs/` directory covers installation, compression internals, provider and framework integrations, CCR, memory, MCP, security, deployments, metrics, benchmarks, and troubleshooting. The consolidated extraction is [Headroom Documentation Summary](docs.md).

## Other features

### MCP tools

MCP hosts can expose three tools:

- `headroom_compress` for on-demand compression;
- `headroom_retrieve` for retrieving originals by hash;
- `headroom_stats` for a session or sub-agent summary.

Install and register it with:

```bash
pip install "headroom-ai[mcp]"
headroom mcp install
```

Use `headroom mcp serve --transport http --host 127.0.0.1 --port 8788 --path /mcp` for Streamable HTTP mode.

### Persistent memory

Persistent memory is opt-in. With the proxy:

```bash
headroom proxy --memory
```

Memory stores project, session, or user facts and retrieves relevant context on later turns. It supports project-local SQLite storage and optional vector/full-text indexes.

### Failure learning

Analyze past agent sessions and write project learnings with:

```bash
headroom learn
headroom learn --apply
```

The first command is a dry run. `--apply` writes managed guidance to a personal or team context file.

### Shared context

For multi-agent handoffs, `SharedContext` can store an agent-produced value in compressed form and return the original on demand:

```python
from headroom import SharedContext

ctx = SharedContext()
ctx.put("findings", large_output, agent="researcher")
summary = ctx.get("findings")
original = ctx.get("findings", full=True)
```

### Images

When image support is available, Headroom can reduce vision-model token cost by choosing provider-specific detail, resize, crop, or text-transcription strategies. The behavior is especially sensitive to query type: requests that need exact fine detail should use the preserve strategy.

## Installation

### Python application

The base package is enough for the in-process `compress()` API:

```bash
pip install headroom-ai
```

The proxy extras include the local HTTP server and MCP support:

```bash
pip install "headroom-ai[proxy]"
```

For all common CLI features:

```bash
uv tool install --python 3.13 "headroom-ai[all]"
```

### TypeScript application

```bash
npm install headroom-ai
```

The TypeScript SDK sends messages to a Headroom proxy:

```bash
pip install "headroom-ai[proxy]"
headroom proxy --port 8787
```

## Compress one LLM call

Pass the same OpenAI- or Anthropic-style message list to `compress()`, then send `result.messages` to the provider.

```python
from headroom import compress
from openai import OpenAI

messages = [
    {"role": "user", "content": "Summarize the failed services."},
    {
        "role": "tool",
        "tool_call_id": "logs-1",
        "content": "INFO start\n" * 3000 + "FATAL disk full\n",
    },
]

result = compress(messages, model="gpt-4o")

print(result.tokens_before)
print(result.tokens_after)
print(result.tokens_saved)
print(result.compression_ratio)
print(result.transforms_applied)

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=result.messages,
)
```

By default, user messages are not compressed. Tool and assistant messages are the normal compression targets. Set `compress_user_messages=True` when a large document or RAG payload is stored in a user message.

```python
result = compress(
    messages,
    model="gpt-4o",
    compress_user_messages=True,
    target_ratio=0.5,
)
```

`target_ratio` applies to the text/ML compressor. It is not a guaranteed final ratio: JSON and other structured content use their own routing and deduplication logic.

## Verify compression locally

The following offline example needs no provider key:

```bash
HEADROOM_OFFLINE=1 python -c "from headroom import compress; log='\n'.join(f'INFO step {i} ok' for i in range(3000))+'\nFATAL disk full'; r=compress([{'role':'user','content':'What failed?'},{'role':'assistant','content':None,'tool_calls':[{'id':'c1','type':'function','function':{'name':'logs','arguments':'{}'}}]},{'role':'tool','tool_call_id':'c1','content':log}], model='gpt-4o'); print(r.tokens_before, '->', r.tokens_after, 'FATAL kept:', 'FATAL' in str(r.messages))"
```

`HEADROOM_OFFLINE=1` also disables update checks and network-based model or pricing downloads.

## Lossless live-agent mode

Use `densify()` for tool results that an agent must read back in live mode:

```python
from headroom import densify

result = densify(messages, model="gpt-4o")
print(result.lossless)
print(result.reverted_messages)
```

Live-agent mode:

- does not drop rows or use removal markers;
- does not run the ML text compressor;
- applies deterministic densification;
- reverts a message if lossless round-trip verification fails.

This mode is useful for preserving prompt-cache stability in repeated tool-result turns.

## Wrap a coding agent

Headroom can route an existing coding agent through a local proxy:

```bash
headroom wrap claude
headroom wrap codex
headroom wrap copilot
headroom wrap cursor
headroom wrap aider
headroom wrap opencode
```

The wrapped session starts a local proxy and launches the agent with the proxy configuration. Stop the wrapped session to stop its session-scoped proxy.

To remove an agent wrapper configuration, use:

```bash
headroom unwrap claude
```

For a persistent proxy:

```bash
headroom proxy --port 8787
headroom deploy
```

Use `headroom doctor` to check the local setup and `headroom dashboard` to inspect live proxy savings.

## Run as an HTTP proxy

```bash
headroom proxy --host 127.0.0.1 --port 8787
```

Point an OpenAI-compatible client at the proxy:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8787/v1",
    api_key="local",
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
)
```

Useful proxy flags include:

| Flag | Purpose |
|---|---|
| `--no-optimize` | Bypass compression for a baseline or debugging |
| `--no-ccr` | Disable reversible retrieval markers and retrieval-tool injection |
| `--lossless` | Use a marker-free, format-native lossless path where supported |
| `--mode cache` | Favor provider-prefix stability in repeated turns |
| `--mode token` | Prioritize token removal over cache stability |
| `--target-ratio 0.5` | Ask text compression to retain about half of eligible text |
| `--memory` | Enable the persistent memory subsystem |

Bind to loopback unless remote clients explicitly need access. Use authentication and a trusted network boundary before exposing the proxy beyond the local machine.

## Use TypeScript

With a proxy running:

```typescript
import { compress } from "headroom-ai";

const result = await compress(messages, {
  model: "gpt-4o",
  baseUrl: "http://127.0.0.1:8787",
  timeout: 15000,
  fallback: true,
});

console.log(result.tokensBefore, result.tokensAfter, result.tokensSaved);
console.log(result.messages);
```

The TypeScript package provides no `headroom` CLI command. The CLI comes from the Python package.

## Reversible compression (CCR)

When proxy CCR is active, a compressed tool result can include a retrieval marker. Headroom caches the original locally and can inject `headroom_retrieve` so the model can request the original if the compressed view is insufficient.

The proxy uses a local store for this data. CCR is intended for reversible compression and is separate from prompt compression itself.

For direct SDK use:

- `compress()` may create CCR markers;
- the full automatic retrieval-tool loop is wired through the HTTP proxy;
- direct callers that need marker resolution must integrate the CCR resolution API themselves.

## Privacy and offline operation

Headroom compression runs locally. No prompt or file content is sent to Headroom for compression.

The project documentation states that an anonymous usage beacon is enabled by default. It reports usage metadata such as counters, provider/model IDs, OS, and architecture, not prompt or completion content.

Disable network and beacon behavior with:

```bash
HEADROOM_OFFLINE=1 headroom proxy --port 8787
```

Or:

```bash
HEADROOM_BEACON=off headroom proxy --port 8787
```

Use `HEADROOM_DIAGNOSTICS=1` when you want per-message compression decisions.

## Choosing settings

| Situation | Recommended starting point |
|---|---|
| Normal coding agent | `compress()` defaults or `headroom wrap` |
| Live tool-result loop | `densify(messages)` |
| Large JSON/search/RAG output | Keep as a `tool` message; begin with defaults |
| Large document in a user message | `compress_user_messages=True` |
| Stable provider cache prefixes | `--mode cache` / proxy defaults |
| Maximum token reduction | `--mode token`; test answer quality |
| Conservative text compression | `target_ratio=0.7` or proxy `--target-ratio 0.7` |

### Proxy profiles

The proxy savings profile controls the default compression posture:

| Profile | Proxy mode | Intended use |
|---|---|---|
| `coding` | `cache` | Default coding-agent profile |
| `balanced` | `token` | Moderate general-purpose compression |
| `agent-90` | `token` | Aggressive cost-sensitive workloads |
| `general` | `token` | Non-coding chat |

Select a profile explicitly:

```bash
HEADROOM_SAVINGS_PROFILE=balanced headroom proxy --port 8787
```

## Troubleshooting

### No savings

Likely causes:

- the content is short (below `min_tokens_to_compress`);
- the content is prose or already dense;
- the block is a user message and user-message compression is disabled;
- the payload is source code and code compression is not enabled.

Start with a tool-result or JSON payload containing clear repetition.

### Compression changed the answer

Try:

- `target_ratio=0.7` for more conservative text retention;
- `protect_recent=0` or enable lossless/agent mode depending on the use case;
- `--no-ccr` or `--no-optimize` to isolate the compression layer;
- a focused test with repeated runs to check determinism.

For live agents, prefer `densify()` because it verifies losslessness and does not remove data.

### The proxy did not start

Install proxy dependencies:

```bash
pip install "headroom-ai[proxy]"
```

Then run:

```bash
headroom doctor
headroom proxy --host 127.0.0.1 --port 8787
```

Check whether another process already owns the selected port.

### TypeScript says it cannot connect

Confirm that `headroom proxy --port 8787` is running and that `baseUrl` matches the proxy address. The npm package does not run the compression server itself.

## Typical savings and limitations

Repository benchmarks report 21–57% input-token savings on four representative agent scenarios. Repeated JSON arrays and logs can exceed 90%, while prose and already-dense output usually save little.

Headroom is not an answer-quality or context-window guarantee. Quality and savings depend on content shape, model, routing, configuration, and task. Compare against a passthrough baseline before production rollout.
