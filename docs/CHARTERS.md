# AI Engineering Project Charters — Canonical Spec v1.0

### The single source of truth for P1–P8 + Capstone. Every Claude Chat window and Claude Code session obeys this document.

**How to use (anti-drift protocol):**

1. This file lives in EVERY repo as `docs/CHARTERS.md`. The current project's section is ALSO copied to `docs/SYLLABUS.md`.
2. Upload this file to every claude.ai Project's knowledge.
3. Add this line to every Project's instructions and every CLAUDE.md: _"docs/CHARTERS.md is law. If a request conflicts with the charter (scope, stack, naming, interfaces), refuse and cite the charter line. Changes require the human to amend CHARTERS.md first and log the change in DECISIONS.md with a version bump."_
4. Model IDs are NEVER hardcoded anywhere except the P1 gateway's Model Registry config. Charters name provider + role; exact model IDs get pinned in the registry at build time (verify current names against provider docs when pinning).

---

## GLOBAL INVARIANTS (apply to all projects)

**Languages & tooling (locked):** Python 3.12 + uv + ruff + pytest + FastAPI + Pydantic v2 · TypeScript strict + Node 22 LTS + pnpm + Next.js (App Router) + Tailwind + shadcn/ui + ESLint/Prettier · every Python service has a Dockerfile.

**Platform stack (locked):**
| Concern | Choice |
|---|---|
| Frontend hosting | Vercel |
| Python services & workers | Railway (Dockerfile deploys) |
| Postgres (+pgvector) | Neon (one Neon project per repo) |
| Redis (cache/rate-limit) | Upstash |
| Object storage | Cloudflare R2 |
| Auth (multi-tenant, P2+) | Clerk |
| Tracing/observability | OpenTelemetry → Langfuse Cloud (from P2) |
| GPU compute (P7/P8) | Modal |
| Payments (capstone only) | Stripe |

**Conventions (locked):** services listen on `:8000` · every service exposes `GET /healthz` · env vars `GATEWAY_URL` + `GATEWAY_API_KEY` in every consumer repo · secrets never committed; `.env.example` always current · conventional commits · every repo README contains an architecture diagram + live demo URL.

**Definition of "production-grade" (every project must check all):** deployed and reachable · authenticated · request tracing (P2+) · tests for core logic · an eval suite runnable via `make eval` (P2+) · CI on PRs (P5 adds eval gates retroactively) · README + diagram + demo URL.

**Docs protocol (every repo):** `docs/CHARTERS.md` (this file) · `docs/SYLLABUS.md` (this project's section) · `docs/STATE.md` · `docs/CONCEPTS.md` · `docs/CC-SKILLS.md` · `docs/CC-TRACK.md` · `docs/DECISIONS.md` · `docs/specs/` · `docs/build-notes/`.

**LLM access rule:** ALL model calls in P2–P8 + capstone go through the P1 gateway. Direct provider calls are charter violations, with exactly two sanctioned exceptions: (a) P7 provider Batch API jobs, (b) provider SDK exploration inside a `sandbox/` folder that never ships.

**Gateway versioning:** the gateway evolves via planned upgrades executed in later projects: v1.1 (P2/M1), v1.2 (P5/M4), v1.3 (P8/M3). Semver + CHANGELOG.md required.

---

## P1 — `p1-gateway` · The LLM Gateway

**One-liner:** An OpenAI-compatible multi-provider LLM gateway with streaming, fallbacks, caching, cost accounting, and a playground — the front door for every subsequent project.
**Problem:** Teams need one stable API over many unstable providers, with cost control and observability. You're building the internal platform every AI company runs (à la LiteLLM), then living on it for 6 months.

**Locked stack:** FastAPI service (Railway) · Next.js playground (Vercel) · Neon (keys, usage) · Upstash Redis (cache, rate limits) · Providers: Anthropic, OpenAI, Google (deployed) + Ollama profile (local dev only).
**Locked API shape:** OpenAI-compatible `POST /v1/chat/completions` (+ `stream=true` SSE) with adapters translating to native provider APIs (incl. Anthropic Messages). Tool/function-call params translate across providers. `POST /v1/embeddings` arrives in v1.1 (P2).

**Exports (consumed later):** deployed base URL + per-project API keys · Model Registry config (single source of model IDs + prices) · thin typed clients `gateway-client` (Python + TS) installed from this repo's git URL · usage/cost dashboard.
**Imports:** none (root of the ladder).

**Milestones:**

- **M1 — Core proxy.** Non-streaming chat completions via Anthropic adapter; Model Registry (YAML: id→provider, price, limits); per-client API keys (hashed, Neon); request logging. _DoD:_ curl with a project key returns a completion; unknown model → 404 from registry; keys revocable.
- **M2 — Streaming [CORE].** SSE end-to-end incl. adapter translation of provider stream events; graceful client-disconnect handling. _DoD:_ playground-less curl streams tokens; disconnect mid-stream logs partial usage.
- **M3 — Multi-provider + resilience [CORE: retry/fallback].** OpenAI + Google + Ollama adapters; tool-calling translation; timeouts; retries with exponential backoff + jitter; fallback chains (`model → fallback[]` in registry); simple circuit breaker per provider. _DoD:_ kill one provider key → requests transparently fail over; jitter visible in logs; tool call round-trips on all three cloud providers.
- **M4 — Metering.** Token counting (provider-reported + tokenizer fallback), per-request cost from registry prices, per-key budgets with 429 on breach, usage rollup tables. _DoD:_ `GET /v1/usage?key=` returns spend; budget breach blocks; numbers reconcile with provider dashboards ±2%.
- **M5 — Caching + templates.** Exact-match response cache (Redis, keyed on normalized request, TTL + bypass header); Anthropic prompt-caching passthrough (`cache_control`); prompt template registry with versions + render endpoint. _DoD:_ repeated request hits cache (<20ms, cost $0); cache_control measurably cuts input cost on a long-context test; templates are versioned and immutable once used.
- **M6 — Guardrail + playground + ship.** Input moderation hook (registry-flagged models get pre-checked); structured-outputs support (JSON schema mode translated per provider); Next.js playground: streaming chat, side-by-side model comparison, cost per message; deploy both; publish clients. _DoD:_ live URLs; side-by-side streams two models concurrently; a P2 hello-world consumes `gateway-client` from git.

**Out of scope:** embeddings (v1.1), semantic caching, A/B routing (v1.2), self-hosted providers (v1.3), org/team hierarchies, UI auth beyond a shared password.

---

## P2 — `p2-rag` · Production RAG Platform

**One-liner:** Multi-tenant "chat with your documents" with a real ingestion pipeline, hybrid retrieval, reranking, citations — and an eval harness that proves retrieval quality.
**Problem:** Naive RAG demos die in production. You'll build ingestion→retrieval→generation with measurable quality, and learn to distrust vibes.

**Locked stack:** FastAPI ingestion+query services (Railway) · Docling for parsing (PDF/DOCX/HTML, incl. OCR + tables) · R2 for raw files · Neon pgvector (HNSW) + Postgres FTS for lexical · fusion via RRF · Cohere Rerank · embeddings via gateway v1.1 (default provider: Voyage; comparator: OpenAI — exact IDs pinned in registry) · Clerk multi-tenancy · Langfuse tracing · Next.js app (Vercel).

**Exports:** `POST /retrieve` (tenant-scoped hybrid+rerank, citations payload) — used by P3 as a tool and P4 via MCP · the eval-harness pattern (`make eval`, golden datasets, LLM-judge) reused everywhere · gateway v1.1.
**Imports:** gateway (chat + embeddings).

**Milestones:**

- **M1 — Gateway v1.1 + skeleton.** Add `/v1/embeddings` + OTel→Langfuse to gateway (cross-repo task); ingestion service with R2 upload, doc registry tables, tenant model (Clerk orgs). _DoD:_ uploaded file lands in R2 + DB row; traces visible in Langfuse; gateway CHANGELOG bumped.
- **M2 — Parsing + chunking lab.** Docling pipeline (text, tables, OCR fallback); implement 4 chunkers: fixed, recursive, semantic, parent-document; chunk inspector UI page. _DoD:_ one gnarly PDF (tables + scanned page) parses correctly; all 4 strategies runnable side-by-side on same doc.
- **M3 — Hybrid retrieval [CORE].** pgvector HNSW + FTS (tsvector) + RRF fusion; tenant isolation enforced at query layer. _DoD:_ keyword-heavy and paraphrase queries both retrieve correctly where either alone fails (documented example each); cross-tenant leakage test passes.
- **M4 — Rerank + rewrite + citations.** Cohere rerank stage; LLM query rewriting (multi-query); answer synthesis with inline citation markers mapped to chunk spans. _DoD:_ citations click through to highlighted source; rerank measurably reorders a documented case.
- **M5 — Eval harness [CORE].** 50+ item golden dataset (question, relevant-chunk ids, reference answer); metrics: recall@k, MRR, faithfulness + answer-relevance via LLM-judge through gateway; `make eval` prints a scorecard; compare chunkers + embedding providers + rerank on/off. _DoD:_ scorecard table in README with your chosen config justified by numbers, not vibes.
- **M6 — Ship.** Streaming chat UI with source panel; collection management; deploy. _DoD:_ live demo URL; a fresh tenant can upload→ask→get cited answer in <2 min.

**Out of scope:** agentic/multi-hop retrieval (P3+), graph RAG, fine-tuned embedders, websocket ingestion progress.

---

## P3 — `p3-agent` · Agents From Scratch → LangGraph

**One-liner:** A data-analyst agent built first as a raw loop (no framework), then rebuilt on LangGraph with memory, human-in-the-loop, and task-success evals.
**Problem:** Agent = LLM + tools + loop + state. You'll own every line of that sentence before any framework hides it.

**Locked stack:** raw loop: pure Python + `gateway-client` (tool-calling via gateway) · tools: read-only SQL over a seeded Neon analytics DB, E2B code sandbox, Tavily web search, P2 `/retrieve` via HTTP · rebuild: LangGraph with Postgres checkpointer · Next.js UI streaming intermediate steps (Vercel) · agent service on Railway.

**Exports:** tool implementations (wrapped as MCP servers in P4) · trajectory-eval pattern · the raw-loop reference implementation (cohort gold).
**Imports:** gateway, P2 retriever.

**Milestones:**

- **M1 — Raw loop [CORE].** ReAct-style loop: model→tool_call→execute→append→repeat; max-iteration + loop-detection guards; scratchpad; two toy tools. _DoD:_ multi-step task ("compare X and Y then compute Z") completes; forced infinite-loop scenario is caught and surfaced.
- **M2 — Real tools.** SQL tool (schema in system prompt, read-only role, row limits), E2B code exec, Tavily, retriever tool. _DoD:_ "pull last quarter's numbers, chart the trend, cite the policy doc" → SQL + code + retrieval in one trajectory; SQL injection attempt via prompt is blocked by the read-only role.
- **M3 — Memory.** Conversation summarization past token threshold; long-term memory notes (pgvector reuse) with retrieval-on-start. _DoD:_ fact stated in session 1 is recalled in session 2; context stays under budget on a 50-turn stress test.
- **M4 — LangGraph rebuild + HITL.** Same agent as a graph; Postgres checkpointer; interrupt-before on destructive tools; resume from approval. _DoD:_ kill the process mid-run → resumes from checkpoint; approval UI gate works; written comparison doc: raw vs framework (what LangGraph bought you).
- **M5 — Agent evals [CORE].** 20-task suite with programmatic success checks + LLM-judge trajectory grading (tool-choice quality, efficiency); `make eval` scorecard. _DoD:_ baseline score recorded; one prompt improvement demonstrably moves the score.
- **M6 — Ship.** Streaming step-by-step UI (tool calls, args, results, approvals); deploy. _DoD:_ live URL; non-technical friend completes an analysis task unassisted.

**Out of scope:** multi-agent (P4), voice (P6), browser automation (P4 stretch), write-access tools.

---

## P4 — `p4-research` · MCP Servers + Multi-Agent Deep Research

**One-liner:** Your tools become MCP servers (Python + TS), then an orchestrator–worker research system produces cited reports via durable, resumable pipelines.
**Problem:** Interop (MCP) and coordination (multi-agent + durable execution) are what separate demos from systems.

**Locked stack:** official MCP SDKs — Python (retriever-mcp, analyst-tools-mcp) + TypeScript (gateway-admin-mcp) · transport: stdio locally, streamable HTTP deployed (Railway) with bearer auth · orchestration: Inngest (durable steps, retries, `wait_for_event` approvals) · workers call gateway via async fan-out · report artifacts to R2 · Next.js progress UI (Vercel).

**Exports:** deployed MCP endpoints (used by Claude Code, claude.ai, and the capstone) · durable-pipeline pattern · report generator.
**Imports:** gateway, P2 retriever, P3 tools.

**Milestones:**

- **M1 — Python MCP servers.** Wrap P2 retrieve + P3 SQL/code/search as two MCP servers (tools + resources); register in Claude Code via `.mcp.json`. _DoD:_ from Claude Code, `retrieve` and `sql_query` callable with correct schemas; inspector shows clean tool descriptions.
- **M2 — TS MCP server.** `gateway-admin-mcp` in TypeScript (usage stats, key ops, registry lookup). _DoD:_ both SDK languages shipped; Claude Code manages gateway keys via MCP.
- **M3 — Orchestrator–worker [CORE].** Planner decomposes a research question → parallel workers (search+retrieve+summarize with per-worker context) → synthesizer merges with source-tracked claims. _DoD:_ 5-worker run completes in parallel (traced); every claim in output maps to a worker citation.
- **M4 — Durable pipeline [CORE].** Rebuild orchestration on Inngest: steps, retries, midpoint human approval (wait-for-event), resume after crash. _DoD:_ kill the process at step 3 → run resumes; approval e-mail/UI gate releases the pipeline; step retries visible.
- **M5 — Report + streaming UI.** Structured report (sections, citations, confidence notes) rendered + stored in R2; live progress UI (per-worker states). _DoD:_ shareable report URL; progress reflects real step events, not polling fakery.
- **M6 — Ship + remote MCP.** Deploy MCP servers (streamable HTTP + auth); connect from claude.ai as a custom connector. _DoD:_ claude.ai chat uses your deployed retriever on real data; auth blocks anonymous calls.
  **Stretch (optional, not gated):** Playwright-driven browser worker.

**Out of scope:** agent-to-agent negotiation protocols, GUI computer use, per-worker fine-tuned models.

---

## P5 — `p5-llmops` · Evals, Observability & Guardrails Platform

**One-liner:** A mini-Braintrust you build yourself — datasets, runners, calibrated LLM-judges, CI gates that block prompt regressions, A/B routing, and a security layer — then retrofitted onto P1–P4.
**Problem:** Shipping AI without evals is shipping blind. This is the discipline layer, and the rarest skill in the market.

**Locked stack:** FastAPI eval service + Postgres (datasets, runs, scores) · runners as a pip-installable `evalkit` (from this repo) · LLM-judge via gateway · GitHub Actions CI (+ headless `claude -p` for auto-triage comments) · gateway v1.2 (A/B) · Presidio for PII · your own injection attack corpus · dashboard in Next.js.

**Exports:** `evalkit` + CI workflow templates adopted by ALL repos · attack corpus + guardrail middleware for the capstone · gateway v1.2.
**Imports:** gateway, Langfuse traces from P2–P4, P3 agent as guardrail test subject.

**Milestones:**

- **M1 — Eval core.** Dataset/run/score schema; runner supporting code assertions (exact, regex, JSON-schema, numeric tolerance); unified `make eval` adapter for P2/P3 suites. _DoD:_ P2 and P3 evals both execute through `evalkit` with stored, diffable runs.
- **M2 — Calibrated judge [CORE].** LLM-judge rubrics; label 100 items by hand; measure judge↔human agreement (Cohen's kappa); iterate rubric until κ ≥ 0.7; document bias findings (position, verbosity). _DoD:_ κ report committed; a documented rubric change that moved κ.
- **M3 — CI regression gates.** GH Action: PR → run affected evals → block on regression beyond threshold → post scorecard comment; headless Claude Code drafts a failure-triage comment. _DoD:_ a deliberately-bad prompt PR is auto-blocked in P2's repo with a readable diff of scores.
- **M4 — Online: A/B + feedback.** Gateway v1.2: weighted routing by experiment header, exposure logging; 👍/👎 + freeform feedback endpoint feeding datasets ("bad case → eval case" flywheel). _DoD:_ live 50/50 experiment across two prompts with per-arm scores; one thumbs-down becomes a regression test.
- **M5 — Security layer [CORE].** 100+ item injection/jailbreak corpus (direct, indirect via retrieved docs, tool-output injection); run vs P3 agent, measure attack success rate; add defenses (input hardening, tool allowlists, output validation, Presidio PII redaction); re-measure. _DoD:_ before/after ASR table; indirect injection via a poisoned P2 document demonstrably blocked.
- **M6 — Dashboard + retrofit.** Runs/experiments/security dashboards; instrument P1 with OTel (back-fill); all repos on CI gates. _DoD:_ one screen answers "did quality regress this week, at what cost, under what attack posture" for every project.

**Out of scope:** SOC2-style compliance tooling, human-labeling marketplace, full Braintrust UI parity.

---

## P6 — `p6-voice` · Realtime Voice + Vision Agent

**One-liner:** A voice agent that holds a natural conversation, uses your MCP tools mid-call to book appointments, and a vision pipeline that turns invoices into structured records.
**Problem:** Multimodal + realtime is where latency engineering, turn-taking, and streaming architecture get real.

**Locked stack:** v1 pipelined: FastAPI WebSocket, Deepgram (STT) → gateway → Cartesia (TTS) · v2 realtime: Pipecat with Daily WebRTC transport, VAD + barge-in · booking tool via P4 MCP (calendar tables in Neon) · vision: gateway vision route + structured outputs → records into P2 store · Next.js voice client (Vercel), voice service on Railway.

**Exports:** voice channel module + latency-measurement harness (capstone reuses both) · invoice-extraction pipeline.
**Imports:** gateway, P4 MCP tools, P2 ingestion, P5 evalkit.

**Milestones:**

- **M1 — Pipelined voice.** Push-to-talk WS loop STT→LLM→TTS with streaming at every seam. _DoD:_ spoken question → spoken answer; per-stage timestamps logged.
- **M2 — Realtime + barge-in [CORE].** Pipecat/Daily: continuous audio, VAD turn detection, interruption cancels TTS + truncates context correctly. _DoD:_ you can talk over it naturally; interrupted responses don't ghost into the transcript.
- **M3 — Tools mid-call.** Booking flow via MCP calendar tool with spoken confirmation loop; filler acknowledgments during tool latency. _DoD:_ "book me Tuesday 4pm" → row in DB → confirmation utterance; double-booking refused conversationally.
- **M4 — Vision intake.** Invoice/receipt → JSON-schema extraction (vendor, lines, totals) with confidence; low-confidence → human-review queue; records land in P2 collection. _DoD:_ 20-doc test set ≥90% field accuracy; sub-threshold docs actually queue.
- **M5 — Latency engineering [CORE].** Instrument TTFB per stage; optimize to voice-to-voice p50 < 1.0s (streaming, sentence-chunked TTS, speculative endpointing); publish a latency-budget doc. _DoD:_ before/after waterfall charts; target met on deployed infra, not localhost.
- **M6 — Ship + eval.** Deploy; P5-based conversation-success suite (scripted personas, task completion). _DoD:_ live URL anyone can call from a browser; success rate on 10 scripted scenarios reported.

**Out of scope:** telephony (SIP/Twilio), speech-to-speech native models (comparison note only), multilingual, wake words.

---

## P7 — `p7-finetune` · Distill a Small Model That Beats the Big One

**One-liner:** Mine your gateway logs for one high-volume task, distill a synthetic dataset, QLoRA-tune Qwen3-8B, and prove — with your own eval harness — that it beats the prompted frontier model on that task.
**Problem:** Knowing WHEN and HOW to fine-tune (and proving it worked) is the difference between an AI engineer and a prompt hobbyist.

**Locked stack:** task: the P6 invoice/structured-extraction task (guaranteed volume + objective metrics) · synthetic gen: Anthropic Batch API (sanctioned direct-provider exception) · curation: dedupe (minhash), schema-validation filters, difficulty stratification · training: Unsloth QLoRA on Modal GPUs (A10G→A100 as needed), metrics to Weights & Biases · base model: Qwen3-8B (pin exact ID at build time; 4B for cheap iteration) · eval: P5 `evalkit` head-to-head.

**Exports:** trained adapter + merged model in R2 (P8 serves it) · data-curation skill (Claude Code) · model card.
**Imports:** gateway logs, P6 task, P5 evalkit, Modal.

**Milestones:**

- **M1 — Task + data mining.** Extract real cases from gateway/P6 logs; define output schema + metrics (field-F1, exact-match, schema-validity); split train/dev/test with leakage checks. _DoD:_ frozen test set (≥200 items) that never touches training; metric script runs on frontier baseline.
- **M2 — Synthetic distillation [CORE].** Batch-API generation of 3–5k examples (teacher = frontier via prompt that includes reasoning); filters: schema-valid, dedupe, judge-scored quality floor; stratify by difficulty. _DoD:_ dataset card documenting yield/rejection rates per filter; spot-check of 50 items ≥95% correct.
- **M3 — SFT.** Unsloth QLoRA run on Modal; loss curves + eval-during-training on dev slice; 2–3 hyperparameter iterations. _DoD:_ reproducible training script (`modal run train.py`); W&B report linked; best checkpoint selected on dev, not vibes.
- **M4 — Head-to-head [CORE].** `evalkit` on the frozen test set: tuned-8B vs frontier-prompted vs base-8B-prompted; cost & latency columns included. _DoD:_ results table in README; tuned model ≥ frontier on primary metric at <10% of the cost — or an honest analysis of why not and iteration.
- **M5 — DPO + model card.** Preference pairs from M4 error analysis; DPO pass; final card (intended use, data lineage, evals, limits). _DoD:_ DPO delta reported; merged model artifact in R2 with card.

**Out of scope:** full-parameter training, RLHF/PPO, pretraining, multi-task tuning, >8B models.

---

## P8 — `p8-inference` · Serve Your Model in Production

**One-liner:** Quantize the P7 model, serve it with vLLM on autoscaling GPUs, benchmark it honestly, and wire it into the gateway with confidence-based cascade routing to the frontier.
**Problem:** Tokens/sec, TTFT, and cost-per-million are the physics of AI products. You'll own the serving stack end to end.

**Locked stack:** quantization: AWQ (serving) + GGUF (local comparison) · serving: vLLM on Modal (OpenAI-compatible endpoint, continuous batching) · load testing: Locust · gateway v1.3: self-hosted provider + cascade routing · dashboards on P5.

**Exports:** self-hosted endpoint as a first-class gateway provider · cascade-routing pattern + cost report (capstone's economics) · gateway v1.3.
**Imports:** P7 model, gateway, P5 dashboards.

**Milestones:**

- **M1 — Quantize.** AWQ + GGUF variants; quality sanity via `evalkit` on the frozen P7 test set (fp16 vs AWQ vs GGUF-Q4). _DoD:_ quality-vs-size table; chosen variant justified (≤1pt metric drop).
- **M2 — Serve + benchmark [CORE].** vLLM on Modal (paged KV, continuous batching); Locust sweeps: concurrency vs TTFT/throughput/cost; compare against Ollama single-stream. _DoD:_ benchmark report with p50/p95 TTFT, tokens/sec, $/1M tokens at 3 concurrency levels; GPU memory math documented (weights+KV vs batch size).
- **M3 — Gateway provider (v1.3).** Register the vLLM endpoint in the Model Registry; health-aware routing; streaming + tool-calling parity through adapters. _DoD:_ P6's extraction traffic flips to the self-hosted model via one registry change, zero client edits.
- **M4 — Cascade [CORE].** Confidence-gated routing: self-hosted first; escalate to frontier on low-confidence (schema-invalid, judge-flagged, or logprob threshold); exposure + cost logged per tier. _DoD:_ live cascade with measured escalation rate; blended $/1M vs frontier-only shown on the P5 dashboard.
- **M5 — Ops.** Modal autoscaling config (scale-to-zero, warm pool), cold-start mitigation measured, alerting on health/latency budget. _DoD:_ cold vs warm TTFT table; a documented monthly cost model at 1M/10M/100M tokens.

**Out of scope:** multi-GPU tensor parallelism, TensorRT-LLM, speculative decoding (reading note only), training-serving co-location.

---

## CAPSTONE — `cap-support-engineer` · The AI Support Engineer (composition)

**One-liner:** A deployable AI support engineer product: ingests a company's docs (P2), resolves tickets via tools/MCP (P3/P4), guarded + evaluated (P5), answers phones (P6), runs cheap on your model with frontier cascade (P7/P8), all through the gateway (P1) — built on the Claude Agent SDK, with Clerk auth + Stripe billing + your cohort plugin.
**Milestones (6):** M1 SDK core agent (query/options/custom tools, your MCP servers) · M2 product surfaces (web widget + dashboard) · M3 voice channel (P6 module) · M4 guardrails+evals wired (P5, CI-gated) · M5 economics (cascade default, per-tenant usage/billing via Stripe) · M6 launch: demo tenant, docs site, and the `ai-eng-cohort` plugin (skills+agents+hooks+MCP) published to your marketplace repo.
**Out of scope:** SLAs, SSO/SCIM, mobile apps.

---

_Charter v1.0 — amendments require a version bump + DECISIONS.md entry in the repo where the change originated._
