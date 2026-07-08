# SYLLABUS — P1 · The LLM Gateway

(Extracted from docs/CHARTERS.md — that file remains the canonical source. If they ever disagree, CHARTERS.md wins and this file is stale — regenerate it.)

**One-liner:** An OpenAI-compatible multi-provider LLM gateway with streaming, fallbacks, caching, cost accounting, and a playground — the front door for every subsequent project.
**Problem:** Teams need one stable API over many unstable providers, with cost control and observability. You're building the internal platform every AI company runs (à la LiteLLM), then living on it for 6 months.

**Locked stack:** FastAPI service (Railway) · Next.js playground (Vercel) · Neon (keys, usage) · Upstash Redis (cache, rate limits) · Providers: Anthropic, OpenAI, Google (deployed) + Ollama profile (local dev only).
**Locked API shape:** OpenAI-compatible `POST /v1/chat/completions` (+ `stream=true` SSE) with adapters translating to native provider APIs (incl. Anthropic Messages). Tool/function-call params translate across providers. `POST /v1/embeddings` arrives in v1.1 (P2).

**Exports (consumed later):** deployed base URL + per-project API keys · Model Registry config (single source of model IDs + prices) · thin typed clients `gateway-client` (Python + TS) installed from this repo's git URL · usage/cost dashboard.
**Imports:** none (root of the ladder).

## Milestones

**M1 — Core proxy.**
Non-streaming chat completions via Anthropic adapter; Model Registry (YAML: id→provider, price, limits); per-client API keys (hashed, Neon); request logging.
_DoD:_ curl with a project key returns a completion; unknown model → 404 from registry; keys revocable.

**M2 — Streaming [CORE].**
SSE end-to-end incl. adapter translation of provider stream events; graceful client-disconnect handling.
_DoD:_ playground-less curl streams tokens; disconnect mid-stream logs partial usage.

**M3 — Multi-provider + resilience [CORE: retry/fallback].**
OpenAI + Google + Ollama adapters; tool-calling translation; timeouts; retries with exponential backoff + jitter; fallback chains (`model → fallback[]` in registry); simple circuit breaker per provider.
_DoD:_ kill one provider key → requests transparently fail over; jitter visible in logs; tool call round-trips on all three cloud providers.

**M4 — Metering.**
Token counting (provider-reported + tokenizer fallback), per-request cost from registry prices, per-key budgets with 429 on breach, usage rollup tables.
_DoD:_ `GET /v1/usage?key=` returns spend; budget breach blocks; numbers reconcile with provider dashboards ±2%.

**M5 — Caching + templates.**
Exact-match response cache (Redis, keyed on normalized request, TTL + bypass header); Anthropic prompt-caching passthrough (`cache_control`); prompt template registry with versions + render endpoint.
_DoD:_ repeated request hits cache (<20ms, cost $0); cache_control measurably cuts input cost on a long-context test; templates are versioned and immutable once used.

**M6 — Guardrail + playground + ship.**
Input moderation hook (registry-flagged models get pre-checked); structured-outputs support (JSON schema mode translated per provider); Next.js playground: streaming chat, side-by-side model comparison, cost per message; deploy both; publish clients.
_DoD:_ live URLs; side-by-side streams two models concurrently; a P2 hello-world consumes `gateway-client` from git.

**Out of scope:** embeddings (v1.1), semantic caching, A/B routing (v1.2), self-hosted providers (v1.3), org/team hierarchies, UI auth beyond a shared password.
