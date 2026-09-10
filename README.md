# inference-gate

A Rust proxy daemon that sits between AI coding CLIs and llama.cpp, enforcing
context discipline, compressing system prompts, and managing KV cache state —
so local LLMs stop wasting your VRAM on yapping.

Built for resource-constrained local inference (consumer GPUs, ≤16GB VRAM).

---

## The problem

Running career-ops or any agentic workload against a local LLM reveals a
recurring failure mode: the model burns your entire context window before it
does any real work. Three root causes:

1. **Unconstrained thinking** — Qwen3 and similar reasoning models emit
   `<think>...</think>` blocks that can consume 5–15k tokens before the actual
   response begins.
2. **Bloated system prompts** — agent mode files (e.g. career-ops `scan.md` at
   33k tokens) load in full regardless of which subset is actually needed.
3. **No KV cache persistence** — your CV, profile, and static context are
   re-tokenized and re-prefilled every session, eating into the budget that
   should go toward real work.

inference-gate solves all three without modifying llama.cpp or the client.

---

## Architecture

```
Qwen Code / any OpenAI-compatible client
        │
        ▼
┌─────────────────────────────┐
│       inference-gate        │  ← port 8081
│                             │
│  ┌─────────────────────┐    │
│  │  prompt compressor  │    │  strips/truncates system prompt
│  └────────┬────────────┘    │
│           │                 │
│  ┌────────▼────────────┐    │
│  │  think budget gate  │    │  monitors SSE stream, kills <think> after N tokens
│  └────────┬────────────┘    │
│           │                 │
│  ┌────────▼────────────┐    │
│  │  prefix cache mgr   │    │  prefills static context, persists KV state
│  └────────┬────────────┘    │
│           │                 │
└───────────┼─────────────────┘
            │
            ▼
     llama.cpp server         ← port 8080
```

Every request passes through three middleware layers before reaching llama.cpp.
Responses stream back through the think budget gate, which monitors and
truncates in real time.

---

## Features

### 1. Thinking budget enforcement

Intercepts the SSE response stream and hard-truncates `<think>` blocks after a
configurable token budget. Once the budget is hit, inference-gate injects a
closing `</think>` tag and lets the model continue to the actual response.

```toml
[thinking]
enabled = true
max_tokens = 512        # kill <think> after 512 tokens
fallback = "no_think"   # or "truncate" — inject </think> and continue
```

Unlike `--no-think` (all or nothing), this gives the model a reasoning budget
rather than eliminating reasoning entirely. Useful for tasks that genuinely
benefit from brief planning.

### 2. System prompt compression

Preprocesses incoming system prompts before forwarding to llama.cpp:

- Strips markdown comments and whitespace-only lines
- Deduplicates repeated instructions across turns
- Extracts only task-relevant sections from large mode files using keyword
  matching against the user message
- Logs compression ratio per request for tuning

A 33k token `scan.md` loading only a `triage` task might compress to 6k —
leaving 27k tokens free for actual reasoning.

```toml
[compression]
enabled = true
strip_comments = true
dedup_threshold = 0.85    # cosine similarity threshold for dedup
section_extraction = true
log_ratio = true
```

### 3. KV cache prefix sharing

Static context (your CV, profile, agent instructions) is prefilled once and
cached to disk as a binary KV snapshot. Subsequent requests skip prefill
entirely for the static portion — the model picks up from the cached state.

```toml
[prefix_cache]
enabled = true
cache_dir = "~/.cache/inference-gate/kv"
static_files = ["cv.md", "profile.yaml"]
ttl_hours = 24
```

This is the same technique used in production inference serving (vLLM's prefix
caching, SGLang's RadixAttention) — inference-gate brings it to single-user
local setups via llama.cpp's `/cache` endpoint.

### 4. Metrics

Exposes a `/metrics` endpoint (Prometheus-compatible) tracking:

- Tokens per second (prefill and decode separately)
- Context utilization % per request
- Think block tokens consumed vs budget
- Compression ratio per request
- Cache hit/miss rate
- VRAM headroom (via `nvidia-smi` polling)

```
GET http://localhost:8081/metrics
```

---

## Tech stack

| Component | Choice | Reason |
|-----------|--------|--------|
| Language | Rust | systems-level, zero-copy streaming, your existing skill |
| Async runtime | Tokio | SSE stream handling, concurrent middleware |
| HTTP | axum | ergonomic, tower middleware composable |
| SSE parsing | custom | thin parser over llama.cpp stream format |
| Config | TOML via `toml` crate | human-editable, no magic |
| Metrics | `prometheus` crate | standard scrape format |
| KV cache I/O | bincode | fast binary serialization for cache snapshots |

---

## Project structure

```
inference-gate/
├── src/
│   ├── main.rs                  # axum server, route setup
│   ├── proxy.rs                 # core forwarding logic
│   ├── middleware/
│   │   ├── compressor.rs        # system prompt compression
│   │   ├── think_gate.rs        # thinking budget enforcement
│   │   └── prefix_cache.rs      # KV cache prefix manager
│   ├── stream/
│   │   ├── sse.rs               # SSE frame parser/emitter
│   │   └── token_counter.rs     # streaming token budget tracker
│   ├── metrics.rs               # Prometheus metrics
│   └── config.rs                # TOML config types
├── config.toml                  # user configuration
├── Cargo.toml
└── README.md
```

---

## Getting started

```bash
git clone https://github.com/yourusername/inference-gate
cd inference-gate
cargo build --release
```

Configure `config.toml`:

```toml
[server]
host = "127.0.0.1"
port = 8081

[upstream]
url = "http://127.0.0.1:8080"   # your llama.cpp server

[thinking]
enabled = true
max_tokens = 512

[compression]
enabled = true
strip_comments = true

[prefix_cache]
enabled = true
static_files = ["~/.career-ops/cv.md", "~/.career-ops/profile.yaml"]
```

Run alongside llama.cpp:

```bash
# Terminal 1 — llama.cpp backend
qwen30B

# Terminal 2 — inference-gate proxy
./target/release/inference-gate

# Point your client at 8081 instead of 8080
```

In Qwen Code, change your base URL from:
```
http://127.0.0.1:8080/v1
```
to:
```
http://127.0.0.1:8081/v1
```

---

## Roadmap

- [ ] Think budget enforcement (streaming truncation)
- [ ] System prompt compression (whitespace + dedup)
- [ ] Section extraction (keyword-based)
- [ ] Prefix cache manager (static file prefill + disk snapshot)
- [ ] Prometheus metrics endpoint
- [ ] GBNF grammar injection (force structured JSON output per task type)
- [ ] Speculative decoding support (draft model proxy)
- [ ] Web UI for real-time context budget visualization

---

## Why this matters

Consumer GPU inference is a context budget allocation problem. The techniques
here — prefix caching, prompt compression, thinking budgets, grammar
constraints — are production inference engineering primitives. Building this
from scratch in Rust, against a real workload (career-ops on a 5060 Ti), is
direct preparation for roles in inference optimization, firmware-adjacent
systems work, and LLM runtime engineering.

---

## License

MIT
