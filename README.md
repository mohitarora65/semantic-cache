# Warmstart Semantic Cache

An OpenAI-compatible caching proxy that reuses safe, semantically equivalent LLM responses while isolating
tenants, prompt versions, models, conversation context, and customer-specific state.

On the included 1,000-request synthetic replay (**60% repeated/paraphrased intents, 40% unique queries**),
the cache records a **59.6% hit rate, 0% known false hits, and 404 provider calls instead of 1,000**. These are
local workload results, not production claims; run `make benchmark` on your own logged-and-redacted traffic
before choosing a threshold.

## Result first

| metric | measured result |
|---|---:|
| Requests | 1,000 |
| Exact hits | 296 |
| Semantic hits | 300 |
| Misses | 404 |
| **Combined hit rate** | **59.6%** |
| **Known false-hit rate** | **0.0%** |
| Provider-call reduction | 59.6% |

The benchmark uses an intentionally short simulated provider delay. Its absolute latency and dollar values
are machine/demo-specific; hit rate, provider-call reduction, and false-hit rate are the portable results.

## Architecture

```text
OpenAI-compatible client
          |
          v
cacheability policy -----> bypass: tools / current data / creative output
          |
          v
exact key (tenant + full request + prompt version)
          |
       miss
          v
semantic namespace + embedding + safety guards
          |
     hit  |  miss
       +--+--------> OpenAI-compatible upstream (Hybrid RAG by default)
       |                    |
       +<--- response + cost/latency metadata
          |
          v
JSON or OpenAI-style SSE response + metrics
```

Redis Stack provides the production HNSW lookup path. SQLite is the zero-infrastructure adapter used by
unit tests and local experiments.

## Run the full two-project stack

From this directory:

```bash
docker compose up --build
```

This starts Redis Stack, Hybrid Search RAG on port 8000, and the semantic-cache proxy on port 8001. Send the
same OpenAI request to port 8001 twice and inspect `X-Cache`:

```bash
curl -i http://localhost:8001/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'X-Tenant-ID: demo-company' \
  -d '{"model":"hybrid-rag","temperature":0,"messages":[{"role":"user","content":"How should an API key be rotated?"}]}'
```

For the local SQLite/deterministic-provider mode:

```bash
make venv
make test
make benchmark
make run
```

## Safety controls that prevent expensive mistakes

- Exact keys include model, temperature, messages, tools, tenant, cache scope, and prompt version.
- Semantic candidates are isolated by tenant, customer scope, system/history context, and model.
- Numbers must match exactly and negation must agree before an approximate match can be accepted.
- User-specific order/account queries bypass caching unless the caller supplies `X-Cache-Scope`.
- Tool calls, current-information requests, creative tasks, and high-temperature generations bypass caching.
- TTLs vary by request type; Redis expires entries automatically.
- Invalidation requires a tenant or model filter. Accidental global deletion is disabled.
- The threshold evaluator reports precision, recall, false hits, and missed hits on labeled prompt pairs.

## API surface

| endpoint | purpose |
|---|---|
| `POST /v1/chat/completions` | OpenAI-compatible cached completion |
| `GET /v1/cache/stats` | hit rate, entries, estimated cost and latency saved |
| `POST /v1/cache/invalidate` | scoped invalidation by tenant and/or model |
| `POST /v1/cache/threshold/evaluate` | precision/recall curve over labeled pairs |
| `GET /metrics` | Prometheus text exposition |
| `GET /healthz` | liveness and entry count |

Streaming requests receive valid server-sent events. The current adapter obtains a complete upstream response
before emitting it; true streaming pass-through with simultaneous buffering is the next production step.

## Design decisions and tradeoffs

**False hits matter more than hit rate.** A lower threshold saves more money but can return the wrong answer.
The policy defaults to 0.82 for factual queries and 0.88 for structured tasks, then adds number and negation
guards that cosine similarity alone cannot provide.

**Context belongs in the namespace.** Matching only the last user message can leak one customer's answer to
another or reuse behavior from an obsolete system prompt. Isolation dimensions reduce hit rate but prevent
that class of correctness and privacy failure.

**Keep exact and semantic hits separate.** Their risks differ, so dashboards, thresholds, and incident
investigations need separate counters.

**Offer two storage adapters.** SQLite keeps the project runnable in an interview with no services. Redis
Stack uses an HNSW vector field so lookup latency does not grow linearly with every cache entry.

## What did not work / limitations

- The offline hashing representation recognizes the included support intents but is not a general semantic
  model. Production should use a versioned embedding model and invalidate entries during migrations.
- SQLite semantic lookup scans a namespace and slows as it grows; it exists for tests, not large traffic.
- The benchmark contains labeled synthetic repetitions. A deployment needs sampled production traffic,
  human review of approximate hits, and a threshold selected from the measured precision curve.
- Provider prices are configuration, not constants. Cost savings are estimates derived from usage metadata.

## Repository map

```text
app/                  policy, cache engine, SQLite/Redis stores, provider, API, metrics
scripts/benchmark.py  1,000-request replay with false-hit measurement
tests/                isolation, expiry, safety guard, streaming, and HTTP tests
results/              generated benchmark report
docker-compose.yml    Redis + Hybrid RAG + semantic cache
```

## Interview discussion

Lead with the false-hit tradeoff rather than Redis. Be ready to explain cache-key dimensions, invalidation,
why a semantic cache is a precision-sensitive retrieval system, how tenant isolation is tested, and what
evidence would justify lowering the threshold.
