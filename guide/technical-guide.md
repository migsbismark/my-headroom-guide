# Headroom Technical Guide

This document describes Headroom's runtime architecture, compression pipeline, public APIs, deployment modes, privacy controls, and implementation structure in this repository.

## Project identity

- Distribution: `headroom-ai`
- Python package: `headroom`
- CLI: `headroom`
- Current repository version: `0.44.0`
- Supported Python floor: 3.11
- License: Apache 2.0

Headroom is implemented primarily in Python, with a Rust/PyO3 core under `crates/`, a native TypeScript SDK under `sdk/typescript/`, and a FastAPI proxy for HTTP traffic.

## Feature summary from the docs

The published documentation defines the full product surface:

- in-process `compress()` and lossless `densify()` APIs;
- transparent HTTP proxy and coding-agent wrappers;
- TypeScript SDK and framework integrations;
- MCP on-demand compress/retrieve/stats tools;
- SmartCrusher, log/search/diff/text, AST code, image, HTML, table, config, and prose compressors;
- CCR reversible originals;
- live-zone context management without history dropping;
- provider-prefix cache-aware modes;
- persistent memory and cross-agent SharedContext;
- failure-session learning;
- savings/durability ledgers and Prometheus/OTLP observability;
- persistent deployment and runtime rollout controls.

For a page-by-page feature summary, see [Docs Summary](docs.md).

## System overview

```text
Application or coding agent
            |
            v
Headroom entry point
  - direct compress()
  - FastAPI proxy
  - SDK/host integration
  - MCP server
            |
            v
Ordered transform pipeline
  - CacheAligner
  - Tool-result interceptors
  - ContentRouter
            |
            v
Content-specific compressors
  - SmartCrusher
  - CodeCompressor
  - Log/Search/Diff compressors
  - HTML/tabular/config compressors
  - Kompress text fallback
            |
            v
LLM provider
OpenAI | Anthropic | Gemini | Bedrock | compatible providers
```

The provider response is returned to the caller unchanged. Headroom's work is concentrated in request compression and optional provider/cache behavior.

## Entry points

| Entry point | Runtime | Typical use |
|---|---|---|
| `compress()` | In-process Python | Library users that control LLM calls |
| `densify()` | In-process Python | Lossless live-agent tool results |
| `headroom proxy` | FastAPI/Uvicorn | Zero-code-change HTTP proxy |
| `headroom wrap` | Proxy plus agent launch/configuration | Coding agents |
| TypeScript SDK | Calls a local proxy | Node/TypeScript applications |
| MCP server | MCP protocol | MCP-compatible clients |
| Integrations | Framework callbacks/hooks | LangChain, LiteLLM, Agno, Strands, and others |

The direct Python API and proxy share compression primitives but do not expose identical surrounding features. For example, full `headroom_retrieve` interception and tool injection belong to the proxy handlers.

The documentation also exposes these integration families:

- SDK wrappers for Anthropic, OpenAI, and Vercel AI SDK;
- LangChain/LangGraph, LiteLLM, Agno, Strands, CrewAI, and AutoGen integrations;
- MCP stdio and HTTP tooling;
- persistent memory and SharedContext;
- `headroom learn` for offline session-failure analysis;
- persistent Docker/service/task deployment.

## Python public API

### `compress`

```python
from headroom import compress

result = compress(
    messages,
    model="gpt-4o",
    model_limit=200000,
    config=None,
    optimize=True,
)
```

The input is a list of OpenAI- or Anthropic-style dictionaries.

`CompressResult` exposes:

| Field | Meaning |
|---|---|
| `messages` | Compressed messages |
| `tokens_before` | Estimated tokens before compression |
| `tokens_after` | Estimated tokens after compression |
| `tokens_saved` | `tokens_before - tokens_after` |
| `compression_ratio` | `tokens_saved / tokens_before` |
| `transforms_applied` | Ordered transform names |
| `diagnostics` | Optional per-message decisions |
| `lossless` | Verification result when lossless mode is requested |
| `reverted_messages` | Count reverted by lossless verification |

Inflated output is protected by an inflation guard: if compression produces more estimated tokens than it received, the original messages are returned and the result reports zero savings.

### `CompressConfig`

Key options:

| Option | Default | Behavior |
|---|---:|---|
| `compress_user_messages` | `False` | Compress large user content only when explicitly enabled |
| `compress_system_messages` | `True` | Allow system-message compression |
| `protect_recent` | `4` | Protect the newest source-code context |
| `protect_analysis_context` | `True` | Preserve code when an analysis/review intent is detected |
| `frozen_message_count` | `0` | Treat leading cached messages as immutable |
| `target_ratio` | `None` | Optional text-compression retention ratio |
| `min_tokens_to_compress` | `250` | Skip smaller messages |
| `kompress_model` | `None` | Select or disable the ML text compressor |
| `savings_profile` | `None` | Apply a named savings profile |
| `diagnostics` | `False` | Return per-message decisions |
| `mode` | `None` | Use normal or live-agent regime |
| `verify_lossless` | `False` | Verify and revert non-lossless messages |

Explicit `CompressConfig` values can be overridden by keyword arguments. Callers' config objects are copied rather than mutated during overrides.

### `densify`

```python
from headroom import densify

result = densify(messages, model="gpt-4o")
```

`densify()` is the `mode="agent"` shorthand. It disables ML text compression and CCR removal, applies deterministic transforms, and forces lossless verification. Any message that cannot be proved to round-trip is reverted.

## Compression pipeline

The pipeline is defined by `headroom/transforms/pipeline.py`.

### Transform order

1. Optional tool-result interceptors
2. `CacheAligner`
3. `ContentRouter`

The pipeline does not maintain the old rolling-window stage that deleted messages. It now leaves the message list intact and compresses content in place. This is a live-zone/content-replacement model rather than an out-of-context history-removal model.

On transform failure, the pipeline follows a fail-open policy: the affected content remains usable and continues upstream rather than blocking the provider request.

### CacheAligner

`CacheAligner` evaluates dynamic-prefix stability. It is not a destructive rewrite stage in normal operation. Provider caching behavior is managed at the request and proxy layers.

Two proxy modes control cache behavior:

- `cache`: only compress the newest applicable delta and preserve prior turns byte-faithfully;
- `token`: recompress earlier turns when needed to maximize raw token removal.

Provider caching is provider-specific:

| Provider | Cache mechanism | Effect |
|---|---|---|
| Anthropic | Prefix `cache_control` | Cached-prefix reads receive discounted pricing |
| OpenAI | Automatic prefix caching | Matching prefixes can be read at a lower rate |
| Google | `CachedContent` | Cached content reduces repeated input cost |

### ContentRouter

`ContentRouter` detects content shape and dispatches each block to one compressor:

| Content | Compressor |
|---|---|
| JSON arrays | `SmartCrusher` |
| Source code | `CodeCompressor` |
| Search/grep output | `SearchCompressor` |
| Build/test logs | `LogCompressor` |
| Diffs | `DiffCompressor` |
| HTML | `HTMLExtractor` |
| CSV/TSV/Markdown tables | Tabular ingest/compressor |
| YAML/TOML/INI | `ConfigCompressor` |
| Plain text | `TextCrusher` |
| Unrecognized text | Kompress ML fallback |

Routing caches use TTL-bounded skip and result caches to avoid repeated compression work.

### Rust core

The native Rust/PyO3 extension is built as `headroom._core`. Python-facing modules delegate supported high-volume operations to Rust-backed implementations.

The Rust workspace includes:

- `crates/headroom-core`;
- `crates/headroom-proxy`;
- `crates/headroom-py`;
- `crates/headroom-parity`;
- `crates/headroom-simulators`.

The Rust core is an implementation accelerator and compatibility layer, not a separate public compression contract.

## Compressor behavior

### SmartCrusher

SmartCrusher handles repeated JSON records and structured arrays. It preserves:

- error and failure records;
- statistical anomalies;
- first and last records;
- records relevant to the extracted user query;
- structurally important boundaries.

The Rust implementation generally handles array compression and detection. A Python shim preserves package-level API compatibility.

For clean tabular arrays, SmartCrusher tries a lossless `csv-schema`/table fold first and adopts it only when it saves at least 30%. The lossy fallback scores first/last items, errors, anomalies, query relevance, and change points. The lossy target is approximately 15 items; errors and anomalies are additive safety classes rather than simple budget casualties.

JSON object keys are preserved. Wide JSON objects with unchanged values are not structurally sampled or dropped. Valid JSON that remains unchanged may pass through a separate lossless whitespace-minification step; malformed JSON passes through.

### Lossless and reversible behavior

Lossless agent mode verifies round-trip preservation and reverts messages that fail verification.

The normal proxy mode also supports CCR:

```text
Tool output
  -> compressed representation
  -> original stored by hash
  -> retrieval marker added
  -> model calls headroom_retrieve
  -> original returned
```

The CCR store may run with SQLite or memory backends. The proxy exposes markers and retrieval-tool handling. Direct SDK callers do not receive the proxy's automatic response-handler loop unless they implement it themselves.

The proxy supports:

- `--no-ccr`;
- `--lossless`;
- configurable retrieval/instruction behavior;
- SQLite persistence and other registered CCR backend entry points.

### Minimum thresholds

Short blocks below `min_tokens_to_compress` pass through unchanged. This prevents compression overhead from exceeding the value of a small payload.

## Proxy architecture

The proxy entry point is `headroom/cli/proxy.py`; runtime handlers live in `headroom/proxy/`.

Provider handlers are implemented for OpenAI, Anthropic, Gemini, Bedrock, batch paths, and compatible provider backends. Before forwarding, handlers apply provider-wire adaptation, compression decisions, cache policy, and optional retrieval-tool injection.

Important proxy subsystems include:

- provider routing and backend selection;
- request and response handlers;
- streaming and SSE buffering;
- CCR marker resolution;
- compression and output savings;
- semantic caching;
- memory and context tools;
- observability and persistent metrics;
- authentication and request limits;
- provider and TLS policy.

## Proxy configuration

Start a proxy:

```bash
headroom proxy --host 127.0.0.1 --port 8787
```

Principal controls:

| Control | Description |
|---|---|
| `--no-optimize` | Disable compression for passthrough baselines |
| `--no-cache` | Disable semantic caching |
| `--no-rate-limit` | Disable proxy rate limits |
| `--no-ccr` | Disable reversible retrieval markers/tooling |
| `--lossless` | Use marker-free lossless behavior where supported |
| `--mode cache` | Preserve prefix-cache stability |
| `--mode token` | Prioritize token removal |
| `--target-ratio` | Set text retention target |
| `--disable-kompress` | Skip ML text compression |
| `--memory` | Enable persistent memory |
| `--code-graph` | Enable live code-graph reindexing |
| `--backend` | Select a provider forwarding backend |

Production operators should review the proxy help and deployment documentation for provider credentials, upstream URLs, headers, timeouts, concurrency, and TLS settings.

## Storage and state

The repository separates state by subsystem:

- CCR: compressed originals and hash-indexed retrieval records;
- memory: project-scoped persistent memory and optional vector storage;
- savings: request/session metrics;
- cache: semantic and provider cache state;
- learn: failed-session analysis and local/generated agent instructions;
- deployment: supervisor/install state and runtime configuration.

Project memory is designed to remain local and project-scoped. Cross-project isolation is a specific documented invariant.

## Security and privacy

Headroom compression operates locally. The library, proxy, and MCP server do not send prompt or file content to Headroom servers for compression.

The default anonymous beacon reports content-free usage metadata. Disabling switches include:

```bash
HEADROOM_OFFLINE=1
HEADROOM_BEACON=off
DO_NOT_TRACK=1
```

Additional security boundaries include:

- loopback binding by default for local use;
- upstream SSRF/target validation;
- TLS trust handling for corporate inspection roots;
- redaction and log safety in wire diagnostics;
- authentication policy and request limits;
- opt-in pipeline extensions;
- explicit provider/backend targeting.

Pipeline extension entry points are opt-in. Installing a package does not discover or enable an extension automatically. Operators enable named entry points with `HEADROOM_PIPELINE_EXTENSIONS`.

## Observability

The package emits OpenTelemetry-compatible compression spans and metrics through `headroom/observability/`.

Useful local surfaces include:

```bash
headroom doctor
headroom perf
headroom savings
headroom dashboard
```

The proxy tracks:

- original and compressed token counts;
- tokens saved per request and per session;
- transform/routing decisions;
- cache reads/writes;
- provider usage;
- output-token shaping;
- failure and circuit-breaker state.

## Offline operation

Set `HEADROOM_OFFLINE=1` to disable network-dependent update, model, pricing, and beacon behavior. Some optional features may require an initial model or package download before they can operate offline.

The smallest network-free verification is the public `compress()` call in the user guide.

## Repository layout

```text
headroom/             Python package
crates/               Rust core, proxy, and PyO3 extension
sdk/typescript/       TypeScript SDK
docs/content/docs/    Published documentation source
tests/                Python test suite
e2e/                  End-to-end tests
benchmarks/           Compression and performance benchmarks
examples/             Runnable examples
plugins/              Agent/editor integrations
scripts/              Repository verification and release tooling
guide/                Standalone technical and user guides
```

## Development setup

```bash
uv sync --extra dev
make install-git-hooks
```

Run focused tests before broader checks:

```bash
uv run pytest tests/<area>
uv run ruff check .
uv run ruff format .
uv run pytest
```

Rust changes use:

```bash
make ci-precheck-rust
```

The repository supports Python 3.11 and newer. Public APIs should include type hints and Google-style docstrings; formatting uses Ruff with a 100-character line limit.

## Failure modes

| Failure | Mitigation |
|---|---|
| Transform exception | Fail open and leave content unchanged |
| Token estimate regresses | Revert inflated compression |
| Lossless verification fails | Revert the affected message |
| Pipeline circuit breaker opens | Pass requests through during cooldown |
| Short payload | Skip compression below the minimum threshold |
| Unsupported content | Pass through or use a conservative fallback |
| Provider request failure | Proxy forwarding/retry policy applies |

## Accuracy and benchmark guidance

Savings vary by content:

- repeated JSON arrays and log lines: commonly 90% or more;
- agent scenarios: roughly 21–57% on measured repository scenarios;
- prose or dense structured text: little savings.

Repository benchmark command:

```bash
uv run python benchmarks/index_proof_table.py --seed 20260902
```

Benchmark answer-quality and latency on the model and workload that will be used in production. Headroom can reduce tokens, but savings alone do not prove task correctness.

## Limits

Headroom does not:

- guarantee equal answer quality for every payload;
- guarantee a context-window fit;
- rewrite natural-language prompt intent;
- send model responses through compression;
- remove all token use;
- replace provider-specific cache tuning;
- provide automatic correctness for every arbitrary binary or malformed format.

Use the official live documentation at `docs.headroomlabs.ai` for API and deployment pages as they evolve.
