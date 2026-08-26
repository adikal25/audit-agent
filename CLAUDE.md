# Audit Trail Agent — Project Instructions

Multi-step RAG agent for regulated industries, differentiated by provable auditability: RBAC enforced in SQL, an append-only hash-chained audit log, a replay console, and an eval harness. This is a **learning-first portfolio project** — the builder is learning production deployment, evals, and agent engineering by building each from first principles. Prefer the explicit, teachable implementation over the clever one.

Read `README.md` first — it holds the problem statement, business case, architecture, and design-decision table. The full phased plan lives in `.claude/tasks/todo.md` (and the original at `~/.claude/plans/the-project-in-the-crispy-tiger.md`).

## Settled architecture decisions — do not relitigate

- **Agent:** hand-rolled Plan-and-Execute loop, plain Python + Pydantic. No LangGraph or agent frameworks. Hard caps: ≤5 plan steps, ≤1 replan (audited, with reason), per-step timeout.
- **Chat LLM:** provider-agnostic `LLMProvider` protocol; default adapter is **OpenRouter** (OpenAI-compatible API). Model choice is config, never hardcoded. Eval judge = a different model via the same adapter.
- **Embeddings:** separate `EmbeddingProvider` protocol; default is **local sentence-transformers**; optional OpenAI-compatible API adapter. Embedding dimension is config, threaded into the schema.
- **Storage:** one Postgres 16 + pgvector instance for vectors, doc/ACL metadata, and the audit log. RBAC is a SQL pre-filter (`WHERE sensitivity = ANY(:allowed)`) — restricted rows never leave the DB. Never filter permissions in Python after retrieval.
- **Audit log:** append-only `audit_events` table; per-session SHA-256 hash chain (`event_hash = sha256(prev_hash || canonical_json(...))`); a Postgres trigger rejects UPDATE/DELETE. Every agent action emits a typed event. Nothing is ever overwritten.
- **API:** FastAPI. **UI:** Streamlit replay console that reads only the API, never the DB.
- **Migrations:** plain ordered SQL files in `migrations/` with a small runner. No Alembic.
- **Packaging:** uv, src layout (`src/audit_agent/`), `pydantic-settings` for config (one `Settings` object, no scattered `os.environ`).
- **Observability:** structlog JSON bound to `session_id`. No OpenTelemetry (README notes it as future work).

## Invariants (violating any of these breaks the project's whole point)

1. RBAC filtering happens **in SQL**, before rows leave Postgres.
2. `audit_events` is append-only — never write an UPDATE/DELETE against it, never "fix" an event; corrections are new events.
3. Every agent step (plan, tool call, tool result, retrieval filtering stats, replan, LLM call, answer) is audited. If you add a new agent capability, add its event type first.
4. The retriever always emits `retrieval_filtered {returned, excluded_by_rbac}` — the replay console and safety evals depend on it.
5. Safety evals are deterministic assertions (forbidden chunk IDs, canary strings) — never LLM-judged, threshold always zero.
6. Corpus docs and their canary strings are committed and deterministic. If you edit a restricted doc, keep its canaries consistent with `evals/datasets/redteam.yaml`.

## Conventions

- Python 3.13, fully typed, Pydantic v2 models for all data crossing a boundary (API, DB, LLM, audit payloads). `Literal`/enums over strings.
- Async throughout the API/agent path (asyncpg, httpx).
- Idiomatic over clever; explicit over implicit; simplicity first. Correctness > speed.
- Tests: unit tests need no network and no DB (use `FakeLLMProvider` from `tests/conftest.py`); integration tests may use the dockerized Postgres. Never call real LLM APIs from tests — real-model evals run manually via the eval CLI.
- Lint/format: ruff. Run it before claiming a task done.
- Secrets only via env / `.env` (gitignored); `.env.example` documents every variable. Never commit keys.

## Commands (as phases land, keep this current)

```
make db-up      # start Postgres (docker compose)
make migrate    # apply migrations/*.sql in order
make ingest     # chunk + embed corpus into pgvector
make api        # run FastAPI dev server
make ui         # run Streamlit replay console
make test       # unit + integration tests
make eval       # run eval suites → report.json + report.md
```

## Workflow for every phase

1. Track work in `.claude/tasks/todo.md`; check items off as they complete; record corrections in `.claude/tasks/lessons.md`.
2. Each phase ends **runnable and verified** — run its verification step from the plan (they're listed per phase in todo.md) before marking it done.
3. Update the README roadmap checkbox and the Commands section here when a phase lands.
4. Show diffs, summarize what changed and why.
