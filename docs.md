# Headroom Documentation Summary

This document summarizes the 63 pages under `docs/content/docs/` and keeps the repository's important concepts, features, compression behavior, deployment options, limits, and operational controls in one place.

## What Headroom is

Headroom is a context-compression layer for LLM applications and coding agents. It compresses tool outputs, logs, JSON, files, RAG chunks, search results, diffs, HTML, images, and other large inputs before they reach a model.

Primary entry points:

- Python `compress()` and `densify()` APIs;
- TypeScript SDK and framework middleware/wrappers;
- `headroom proxy`, an OpenAI- and Anthropic-compatible HTTP proxy;
- `headroom wrap`, for starting coding agents through the proxy;
- MCP tools for on-demand compression, retrieval, and statistics;
- LangChain, LangGraph, LiteLLM, Agno, Strands, CrewAI, AutoGen, Vercel AI SDK, and other integrations;
- Docker, persistent services/tasks, and the `headroom deploy` path.

The package is distributed as `headroom-ai`; the Python import and CLI are `headroom`.

## How compression works

Every request follows a short pipeline:

1. Safety gates decide whether a message or content block may be changed.
2. `CacheAligner` optionally reports prefix drift but does not rewrite traffic.
3. `ContentRouter` detects the content type.
4. A format-specific compressor transforms the block.
5. An inflation guard or lossless verifier can revert unsafe output.
6. In CCR proxy flows, the original is retained locally and a retrieval marker is added.

The pipeline does not maintain a rolling history-dropping stage. It compresses live content in place and leaves the message list intact. System prompts and user intent are not expected to be dropped.

### Automatic routing

| Detected content | Main compressor | What it preserves |
|---|---|---|
| JSON arrays | SmartCrusher | Keys and structure, errors, anomalies, boundaries, relevant items |
| Plain text | TextCrusher or Kompress | Relevant sentences or high-value tokens |
| Build/test logs | LogCompressor | Errors, warnings, stack traces, summaries |
| Search/grep output | SearchCompressor | Query matches, file diversity, relevant matches |
| Unified diffs | DiffCompressor | Changed hunks and enough context |
| Source code | CodeAwareCompressor only when enabled | Imports, signatures, types, decorators |
| HTML | HTMLExtractor | Main article content, structure |
| CSV/TSV/Markdown tables | Tabular ingestion plus SmartCrusher | Table structure and important rows |
| YAML/TOML/INI | ConfigCompressor | Structural configuration fields |
| Images | Image compressor/router | Provider-specific detail strategy |
| Git status | Lossless passthrough | Paths and status sections unchanged |

Detection uses structure and patterns rather than manual hints. Magika and deterministic detectors may help classify content.

### SmartCrusher

SmartCrusher first tries a lossless tabular fold for clean JSON arrays. It re-encodes rows into a compact `csv-schema` form when that saves at least 30%.

When lossy keep/drop is required, it evaluates:

- first and last items;
- error records;
- statistical anomalies;
- relevance to the extracted user query;
- change points;
- sampled/representative content.

Its lossy path targets 15 items by default, although it is allowed to preserve additive safety classes such as errors and anomalies beyond that limit. JSON object keys are not sampled or removed. Wide objects can therefore show no structural savings.

SmartCrusher also:

- reencodes nested arrays;
- minifies unchanged valid JSON separately and losslessly;
- prevents malformed JSON from breaking a call;
- ignores arrays below configured item/token thresholds;
- uses a reversibility contract in CCR flows.

### Logs and text

Log compression keeps failures, warnings, stack traces, summaries, and section markers. It removes repeated passed-test lines and verbose success output.

Search compression keeps exact query matches, BM25/relevance matches, file diversity, and boundary matches. Diff compression keeps changed hunks and limited context.

TextCrusher is extractive: it keeps relevant sentences and headings without rewriting them. Kompress is the ML/ONNX fallback for general text, with configurable keep ratios.

### Code

Code compression is AST-aware and requires the `code` extra. It preserves imports, signatures, types, decorators, and class structure, and can compress function bodies and comments.

Supported tiers include Python, JavaScript, TypeScript, Go, Rust, Java, C, C++, C#, PHP, and Perl in the relevant implementations.

Code is gated conservatively:

- small content is skipped;
- recent code is protected;
- analysis/review intent protects code;
- direct library behavior may leave code unchanged;
- proxy code-aware routing is configurable.

The repository documentation and tests treat code correctness more important than aggressive code savings.

### Images

The image compressor uses query and image analysis to select an optimization technique:

- `full_low` for general understanding;
- `preserve` for fine detail;
- `crop` for region-specific questions;
- `transcode` for text extraction.

OpenAI uses native `detail` settings, Anthropic resizes, and Gemini applies provider-specific tile behavior. Image support is optional and may require ML dependencies.

## Compression modes and context management

There are two separate mode axes.

### Proxy mode

- `cache` is the default: preserve prior turns and compress the newest live delta;
- `token` prioritizes visible token removal and may rewrite earlier turns.

Cache mode is normally preferred for long coding sessions because provider prefix caches reward byte-identical prefixes.

### SDK mode

`HeadroomClient` supports:

- `audit`: observe only;
- `optimize`: apply transforms;
- `simulate`: preview without a provider call.

These are not the same as proxy `cache`/`token`.

### Live-agent mode

`densify()` is lossless live-agent compression:

- no CCR removal markers;
- no dropped rows;
- no ML text-compression path;
- deterministic output;
- round-trip verification;
- affected messages revert to original if verification fails.

This is suited to tool results that an agent reads repeatedly while preserving cache stability.

## CCR: Compress-Cache-Retrieve

CCR keeps compressed tool output reversible by caching originals locally.

The normal proxy flow is:

```text
compress -> cache original by hash -> insert retrieval marker -> model calls headroom_retrieve -> original returned
```

CCR stores:

- a SQLite store by default;
- memory-only mode for processes that must not persist originals;
- configurable TTL and size;
- optional MCP retrieval.

Direct `compress()` calls may embed markers, but the automatic proxy retrieval loop is not equivalent to direct SDK-only use.

Use `--no-ccr` to disable markers/retrieval tooling or `--lossless` for supported format-native lossless behavior.

## Feature map

### Coding agents

`headroom wrap` supports Claude Code, Codex, Copilot, Cursor, Aider, OpenCode, Grok, Cline, Continue, Goose, OpenHands, OpenClaw, Vibe, OMP, Droid, Zcode, and related agent integrations.

It starts or joins a proxy and configures the selected tool to use it. `unwrap` removes the reversible wrapper configuration.

### Persistent memory

The optional memory subsystem extracts and stores facts across turns and sessions. It supports:

- user/session/agent/turn scope;
- semantic search and full-text search;
- SQLite CRUD and indexing;
- HNSW vector search;
- project, user, or global storage;
- temporal supersession;
- local ONNX or remote embedding backends.

Memory is opt-in through `--memory` or the relevant SDK wrapper.

### Cross-agent context

SharedContext reduces context replayed between agents. A producer stores a large value, another agent receives a compressed summary, and `full=True` can recover the original.

### Failure learning

`headroom learn` analyzes coding-agent sessions offline, correlates failures with later successful actions, and writes project instructions. It can target `CLAUDE.local.md`, `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `GROK.md`, or a custom file.

Default behavior is dry-run; `--apply` writes managed marker-delimited sections.

### MCP

The MCP server exposes:

- `headroom_compress`;
- `headroom_retrieve`;
- `headroom_stats`.

It supports stdio and Streamable HTTP transports. MCP compression and proxy compression serve different roles and should not be treated as a requirement to double-compress traffic.

### Framework integrations

The docs cover Anthropic SDK, OpenAI SDK, Vercel AI SDK, LangChain/LangGraph, LiteLLM, Agno, Strands, CrewAI, and AutoGen. Integration style varies:

- direct SDK wrapper;
- callback or middleware;
- model/provider wrapper;
- proxy/base-URL routing.

### Persistent deployment

Headroom supports:

- `headroom proxy` for on-demand use;
- `headroom wrap` for session-scoped proxy use;
- `headroom deploy` for turnkey local deployment;
- native service/task supervisors;
- restartable Docker deployments;
- Unix sockets and detached runtime management.

Persistent provider/configuration scope is intentionally reversible and differs across Claude, Codex, OpenCode, Copilot, Aider, and environment-based tools.

### Observability

Available surfaces include:

- `headroom perf`;
- `headroom savings`;
- `headroom dashboard`;
- `/stats`;
- `/stats-history`;
- `/metrics`;
- Prometheus;
- OTLP/HTTP;
- OpenTelemetry;
- Langfuse/self-configured tracing;
- per-session and durable savings ledgers.

Proxy metrics distinguish completed, failed, and rate-limited requests. `/metrics` has counters and sum/count pairs but no compression-ratio histogram buckets.

## Configuration and safety

Important settings:

| Setting | Purpose |
|---|---|
| `compress_user_messages` | Opt into large user-message compression |
| `compress_system_messages` | Allow or preserve system messages |
| `protect_recent` | Protect recent code messages |
| `protect_analysis_context` | Protect code during analysis/review intent |
| `target_ratio` | Text/Kompress keep ratio |
| `min_tokens_to_compress` | Skip small messages |
| `kompress_model` | Choose or disable ML text compression |
| `HEADROOM_OFFLINE=1` | Disable network/model/beacon behavior |
| `HEADROOM_BEACON=off` | Disable anonymous usage beacon |
| `HEADROOM_CCR_BACKEND=memory` | Avoid persisted CCR originals |
| `HEADROOM_REQUEST_BODY_MAX_BYTES` | Whole-request hard admission ceiling |
| `HEADROOM_SAVINGS_PROFILE` | Select coding, balanced, agent-90, or general posture |
| `HEADROOM_ROLLOUT_CHANNEL` | Stable/beta/canary/dev runtime feature control |

Proxy request admission defaults to a 100 MiB whole-request limit. Large admitted histories may bypass selected request processing after provider-specific thresholds.

## Savings and benchmarks

Savings depend on repetition and content type. The repository's published proof table reports four agent scenarios from 21% to 57% savings. Repeated JSON arrays and logs can compress much more; prose and dense code usually save less.

Representative documented measurements include:

- JSON arrays: highly workload-dependent;
- search results: approximately 80–95%;
- build/test logs: approximately 85–95%;
- diffs: approximately 60–80%;
- HTML extraction: approximately 70–90%;
- text: workload-dependent;
- code: often zero or opt-in only.

The benchmark harness is:

```bash
uv run python benchmarks/index_proof_table.py --seed 20260902
```

Use benchmarked output quality and passthrough baselines for validation. Compression overhead is normally under milliseconds for many structural workloads, but model calls, ML initialization, and large inputs may dominate.

## Privacy and security

Headroom is a local middleware/MITM proxy. It sees plaintext prompts, tool results, source code, and provider credentials passed through it.

Important boundaries:

- compression runs locally;
- data is not sent to Headroom merely to compress it;
- provider traffic remains TLS upstream;
- the anonymous beacon is opt-out and sends metadata, not prompt text;
- `HEADROOM_OFFLINE=1` disables network/update/model/beacon behavior;
- request logs and proxy logs may contain sensitive operational data;
- CCR and memory stores write originals or facts to disk unless disabled;
- loopback is the default proxy boundary;
- network exposure requires explicit authentication and threat-model review;
- a shared proxy token is an identity-free shared secret;
- full message logging is opt-in and content-bearing;
- client-supplied upstream routing is SSRF-bounded.

The proxy keeps sensitive control routes on loopback or explicitly trusted CIDRs and protects mutating routes against cross-origin requests. The workspace and control files are not encrypted at rest; filesystem permissions and OS account security are the boundary.

## Known limitations

- Savings are not guaranteed and vary by workload.
- Short payloads may cost more than they save.
- Already-dense prose and code usually compress poorly.
- Large Markdown/HTML and mixed documents may route inconsistently.
- Malformed JSON and unsupported syntax fail open to the original.
- Direct SDK and proxy feature sets are not identical.
- CCR entries expire and can become unrecoverable.
- Provider cache savings and compression savings are separate numbers.
- Live agent guarantees are stronger in `densify()` than ordinary lossy proxy compression.
- Agent answer quality must be tested against a representative baseline.
- Optional ML features require extra dependencies or model downloads.

## Primary user-facing commands

```bash
pip install headroom-ai
headroom proxy --port 8787
headroom wrap claude
headroom unwrap claude
headroom doctor
headroom perf
headroom savings
headroom dashboard
headroom mcp install
headroom learn
headroom deploy
```

For detailed pages, start with Quickstart, Architecture, How Compression Works, SmartCrusher, Text & Log Compression, CCR, Proxy, Memory, Security Model, Benchmarks, and Troubleshooting under `docs/content/docs/`.
