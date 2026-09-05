# Audit Trail Agent

**A multi-step RAG agent that regulated industries could actually deploy — because it can prove what it did.**

Role-based access enforced in the database, an append-only hash-chained audit log, a step-by-step replay console, and an evaluation harness with hard numbers. Runs with one command via Docker Compose; same stack ships on Kubernetes.

---

## Why this exists

RAG and agent demos are everywhere. Almost none of them are deployable in a bank, hospital, insurer, defense contractor, or law firm — and the blocker is not model quality. It's governance:

1. **No access control at retrieval time.** A shared vector index happily returns HR salary data to a contractor. Filtering results in application code after retrieval is a promise, not a guarantee.
2. **No evidence of what the agent did.** When a regulator, auditor, or incident-response team asks *"why did the system say this, and what data did it touch?"*, a typical demo has nothing. Logs are mutable, incomplete, or absent.
3. **No measured behavior.** "It seems accurate" is not an answer a risk committee accepts. They need numbers: accuracy on a golden set, zero permission leaks under adversarial testing, latency percentiles.

## The business case

Regulated industries are exactly where LLM agents have the **highest ROI** (dense internal document corpora, expensive expert time spent searching them) and the **lowest adoption** (compliance risk). Frameworks like SOC 2, HIPAA, FINRA books-and-records rules, and the EU AI Act's traceability requirements all effectively demand the same three things from an automated system:

- **Access enforcement** — the system must not expose data a user isn't entitled to.
- **Tamper-evident records** — automated decisions must be reconstructable after the fact, and the record must be provably unmodified.
- **Documented evaluation** — behavior must be measured, not asserted.

A system that ships those three things **as architecture** — not as a policy slide — is the difference between a pilot that dies in review and a deployment.

## How this is different from a standard RAG demo

| Standard demo | This project |
|---|---|
| One shared index; everyone sees everything | RBAC enforced **in SQL**: restricted chunks never leave the database (pre-filtering, not post-filtering) |
| App logs — mutable, best-effort | Append-only audit store: `UPDATE`/`DELETE` rejected by a database trigger; per-session SHA-256 hash chain makes tampering **detectable and demonstrable** |
| "Trust me" answers | Every plan, tool call, retrieval — including **what was withheld from the user and why** — and final answer is a typed audit event; a replay console steps through any past session |
| No evaluation | Eval harness with hard numbers: retrieval hit-rate/MRR, answer quality, a **zero-tolerance permission-leak red-team suite**, and p50/p95 latency — with the audit log itself as the test oracle |
| Runs in a notebook | `docker compose up` runs the full stack; the same stack deploys to Kubernetes — a real cloud / on-prem story |

## What "done" looks like (measurable)

- A fresh clone runs end-to-end with one command.
- The red-team suite reports **0/20 permission leaks** — and *fails loudly* if the RBAC clause is removed on a branch, proving the tests have teeth.
- Chain verification pinpoints the exact event if anyone tampers with the audit log.
- A non-engineer can follow a session replay and understand why the agent answered the way it did.

---

## Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │                 FastAPI                     │
  user ──ask──────▶ │  POST /ask   GET /sessions/{id}/events      │
  (with roles)      │              GET /sessions/{id}/verify      │
                    └───────┬─────────────────────────────────────┘
                            │
                    ┌───────▼───────────────┐     every step is a
                    │  Plan-and-Execute     │     typed audit event
                    │  agent loop           │ ──────────────┐
                    │  (hand-rolled, ≤5     │               │
                    │  steps, ≤1 replan)    │               │
                    └───┬───────────────────┘               │
                        │ tools                             ▼
        ┌───────────────┼──────────────┐        ┌──────────────────────┐
        │               │              │        │  audit_events        │
  search_documents  summarize   list_accessible │  append-only         │
        │                            _docs      │  SHA-256 hash chain  │
        ▼                                       │  UPDATE/DELETE       │
  ┌──────────────────────────┐                  │  rejected by trigger │
  │  pgvector (Postgres)     │                  └──────────┬───────────┘
  │  WHERE sensitivity =     │                             │
  │  ANY(:user_allowed)      │                             ▼
  │  ← RBAC in the database  │                  ┌──────────────────────┐
  └──────────────────────────┘                  │  Streamlit replay    │
                                                │  console (reads API) │
                                                └──────────────────────┘
```

One Postgres instance holds the vector index, document/ACL metadata, **and** the audit log — so the audit trail shares a transaction boundary with the data it describes, and RBAC is a `WHERE` clause the database enforces, not app code that can be bypassed.

### Key design decisions

| Decision | Choice | Why |
|---|---|---|
| Agent pattern | **Plan-and-Execute** (one bounded replan) | The upfront plan is a first-class audit artifact — reviewers see *intent before action*. A replan is itself an audited event with a stated reason. ReAct smears intent across interleaved steps and is harder to review. |
| Vector store | **pgvector** | RBAC becomes SQL pre-filtering; audit log + data in one ACID database; one StatefulSet on K8s; battle-tested enterprise story. *When I'd switch:* dedicated engines (Qdrant/Weaviate) win at large scale with heavy payload filtering. |
| Agent loop | **Hand-rolled Python + Pydantic** (no LangGraph) | Audit hooks belong at every seam of the loop — owning the loop makes that natural. Also: no framework magic between the code and the person reading it. |
| Chat LLM | **Provider-agnostic protocol; OpenRouter default** | One key, any model; swapping providers is a config change; the eval judge is simply a *different model* through the same adapter. |
| Embeddings | **Separate protocol; local sentence-transformers default** | Free, deterministic, vendor-free — a genuine on-prem answer for regulated environments. An OpenAI-compatible API adapter is a config switch away. |
| Immutability | **Postgres trigger rejects UPDATE/DELETE** | Enforced by the database, not by convention. Best demo in the project: *watch the log refuse to be tampered with.* |
| Observability | **structlog JSON, session-bound** | The audit log *is* the trace store and the replay console is the trace viewer. In production you'd export audit events as OTel spans; deliberately out of scope here. |

## The corpus: Meridian Dynamics

A fictional aerospace-adjacent company with ~35 synthetic Markdown documents across sensitivity tiers, committed to the repo (deterministic demos, no generation cost):

| Sensitivity | Examples | Who can read it |
|---|---|---|
| `public` | Employee handbook, holiday calendar | everyone, incl. contractors |
| `engineering_internal` | Architecture docs, incident postmortems | engineers and above |
| `finance_confidential` | Q3 revenue report, budget forecasts | finance_analyst, admin |
| `hr_confidential` | Salary bands, termination memo | hr_admin, admin |
| `legal_privileged` | Acquisition NDA, litigation memo | legal_counsel, admin |
| `export_controlled` | Propulsion spec (fictional) | senior_engineer, admin |

Roles (`contractor` → `admin`) map to sets of sensitivities in `corpus/acl.yaml`. Documents contain **canary strings** — fake figures that exist *only* in restricted docs — and cross-tier bait (a public doc that *mentions* the Q3 report; a confidential doc with the actual numbers), so the leak tests are realistic, deterministic, and zero-tolerance.

## Evaluation harness

`audit-agent-eval run` produces a JSON + Markdown report; a thin pytest wrapper gates thresholds in CI.

| Suite | What it measures | How |
|---|---|---|
| **Accuracy** (~25 golden QA cases) | retrieval hit-rate@k, MRR, answer faithfulness/relevance | assertions + LLM-as-judge (different model, same adapter) |
| **Safety** (~20 red-team cases) | permission leaks: direct asks, indirect extraction, prompt-injection attempts | **deterministic assertions against the audit log**: no forbidden chunk ID retrieved, no canary string in any answer. Threshold: zero. |
| **Latency** | p50/p95 per step (plan, each tool, answer) and end-to-end | computed from audit event timestamps — the log is the instrument |

## End results

1. `docker compose up` — full stack comes up; ingestion seeds the corpus.
2. In the console's **Ask** tab, ask as `finance_analyst`: *"Compare Q3 revenue to the budget forecast."* → multi-step plan, answer citing both documents.
3. Ask the same question as `contractor` → the agent declines; the retrieval step shows **"🔒 N chunks withheld by access policy."**
4. Open **Replay** on both sessions — step through plan → tool calls → retrievals → answer, with the chain-verified badge green.
5. Try to tamper: `UPDATE audit_events SET ...` in psql → the database refuses. Corrupt a row as superuser → **Verify chain** pinpoints the broken event.
6. Show the latest eval report: hit-rate, **0/20 leaks**, latency percentiles.

## Roadmap

- [x] **Phase 0** — this document: problem, business case, design decisions
- [ ] **Phase 1** — skeleton, provider layer, corpus, ingestion into pgvector
- [ ] **Phase 2** — RBAC retrieval + append-only audit store (hash chain, immutability trigger)
- [ ] **Phase 3** — Plan-and-Execute agent loop + FastAPI
- [ ] **Phase 4** — Streamlit replay console
- [ ] **Phase 5** — eval harness (accuracy / safety / latency) + CI
- [ ] **Phase 6** — Docker Compose full stack + Kubernetes (kind)

## Stack

Python 3.13 · uv · FastAPI · Pydantic v2 · Postgres 16 + pgvector · sentence-transformers · OpenRouter (chat) · Streamlit · structlog · pytest · ruff · Docker Compose · Kubernetes (kind)

## Docs

- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) — build milestones, acceptance checks, and the portfolio demonstration plan
- [docs/architecture.html](docs/architecture.html) — the visual design dossier: system diagram, the RBAC gate, the hash chain, the agent loop
- [docs/concepts.md](docs/concepts.md) — concepts & technologies per phase, and how each is applied (the learning syllabus)
- [docs/hash-chain.md](docs/hash-chain.md) — the hash chain explained end to end: fingerprints, chaining, a worked tamper-detection example, and the anchoring limitation

## Status

Phase 0. Nothing runs yet — by design: the *why* was written before the code, and every phase ends with something runnable and verified.
