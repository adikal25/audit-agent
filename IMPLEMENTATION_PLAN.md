# Audit Trail Agent — Build and Demonstration Plan

Status: planned; the repository currently contains packaging, documentation, and empty modules.
Updated: 2026-09-05.

## What we are building toward

A reviewer can ask one business question as two users, see different authorized evidence,
replay the resulting sessions, and watch the verification tools expose deliberate failures.
The deliverable is a working demonstration with reproducible evidence of its guarantees.
Deployment readiness for regulated organizations is outside the initial claim.

Keep the architecture in `CLAUDE.md`: plain Python and Pydantic, a bounded Plan-and-Execute
loop, provider protocols, Postgres with pgvector, FastAPI, and an API-only Streamlit console.
Prefer small, explicit implementations that the builder can explain and test.

## Acceptance criteria for the portfolio release

- A fresh clone starts the stack with Docker Compose after documented configuration.
  The full demo requires a configured chat provider; deterministic checks require no LLM API.
- A finance analyst receives an answer citing accessible financial documents; a contractor
  asking the same question receives no restricted content or citations.
- At least 20 deterministic safety cases pass with zero forbidden chunk IDs in retrieved
  evidence and zero restricted canary strings in answers or other user-visible responses.
- Removing the SQL permission predicate in a disposable test copy makes safety checks fail.
- The application database role cannot update, delete, or truncate audit events. A deliberate
  event corruption in a disposable database is located by sequence number during verification.
- Session listings, replay, and verification enforce the authenticated caller's access.
- A report records the corpus version, code revision, model and configuration, retrieval
  hit-rate@k and MRR, safety results, and sample counts with p50/p95 latency. Initial retrieval
  target: at least 80% hit-rate on approximately 25 golden cases, with k recorded explicitly.
- A short walkthrough shows the successful path, denied path, replay, and failing checks.

## Milestones and completion checks

### 1. Foundation and a small reproducible corpus — Phase 1

Start with 6–8 committed documents spanning public, finance, and HR sensitivities, including
cross-tier bait and distinct canaries. Expand toward the planned approximately 35 documents
after the complete path works. Define initial red-team cases alongside the documents.

Implement Settings, typed provider protocols, the OpenRouter adapter, local embeddings,
provider factories, deterministic fake providers, migrations, and heading-aware ingestion.
Add a Makefile, `.env.example`, and database-only Compose configuration. Add CI for lint and
unit tests now; expand it with later milestones.

**Complete when:** migrations work on a fresh database; repeated ingestion produces no
duplicate chunks; changed documents replace their stale chunks; invalid metadata or an
embedding dimension mismatch fails clearly; provider parsing and retry behavior are tested.

### 2. Access enforcement and verifiable events — Phase 2

Implement role expansion and parameterized SQL permission predicates for both search and
document listing. An unknown role grants no access. Define typed audit events before adding
the actions that produce them, including `retrieval_filtered {returned, excluded_by_rbac}`.
Define withheld counts as matching candidates excluded by policy under the same documented
search criterion; expose aggregate counts only, never restricted titles, IDs, or text.

Build canonical serialization, a per-session SHA-256 chain, event reader, and verifier.
Serialize concurrent appends to each session inside database transactions so sequence numbers
and previous hashes cannot diverge. Use separate migration and application database privileges.

**Complete when:** role tests prove restricted rows do not leave retrieval queries; the
application role cannot mutate or truncate the log; concurrent appends form one valid chain;
payload corruption and missing interior events are detected. Document that an unanchored
chain cannot prove that its tail was not removed or that a privileged attacker did not rewrite it.

### 3. An authenticated, audited question-to-answer path — Phase 3

Resolve identity and roles server-side. For the initial local demo, configured opaque tokens
may map to fixed synthetic users; token values stay out of the repository and logs. Request
bodies cannot assign roles. The console's user selector is explicitly a local demo feature.
Production identity-provider integration is a later extension.

Implement the planner, tool registry, executor, and API. Enforce a total budget of at most five
executed tool steps across the initial plan and at most one replan, plus per-step timeouts.
Tools receive the server-resolved user context. Citations must refer to evidence retrieved
for that session. Record concise plan rationale, tool inputs/results, provider metadata,
errors, and answers; do not request or store hidden model reasoning.

Default session access to the owner. Allow admin review only when current permissions cover
the session's recorded sensitivity scope; check this on listings, event reads, and verification.
Recheck scope after role changes to prevent replay from restoring revoked access.

If a required audit write fails, stop processing and do not release an unaudited answer.
Record failure events when storage remains available; use operational logs for storage outages
and leave an interrupted session visibly incomplete when durable completion is impossible.

**Complete when:** a fake-provider end-to-end test covers successful and denied questions;
forged roles, cross-user session access, revoked access, invalid plans, timeouts, and audit
storage failure have deterministic tests. No test calls a real LLM API.

### 4. Replay that explains the outcome — Phase 4

Build the Ask and Replay views using only authenticated API calls. Show the plan, ordered
events, citations, aggregate withheld counts, latency, and chain verification result.
Separate chain integrity from session completion: a valid chain may describe a failed run.

**Complete when:** a reviewer can explain why the finance and contractor answers differ using
the replay alone, and unauthorized session data never appears in the console.

### 5. Evidence that the controls work — Phase 5

Expand the committed corpus and golden/red-team datasets. Include direct requests, indirect
extraction, malicious instructions in accessible documents, and sensitive-data bait. Add
separate API security tests for forged identity, session access, and role revocation.
Check forbidden retrieval IDs and canaries in every user-visible surface, not only final prose.

Implement the report CLI, deterministic CI gates, and manual real-model evaluations. Use a
different configured model for optional answer-quality judging; safety verdicts stay deterministic.
Track token usage and estimated cost only when supported by provider metadata and documented
pricing inputs; report unavailable fields explicitly.

Create disposable mutation checks that remove the RBAC predicate and corrupt an audit payload.
Never introduce an access-control bypass flag into the normal application. Assert that the
relevant safety or integrity checks fail for the intended reason, not because the test crashed.

**Complete when:** the baseline passes, both mutations are caught, and a saved report is
reproducible from its recorded inputs. Report measured results without extrapolating zero
observed leaks into a universal safety guarantee.

### 6. A reproducible release — Phase 6, Compose first

Package API and UI with non-root images and health checks. Order startup as database,
migrations, ingestion, API, and UI. Document first-run embedding downloads and provider setup.
Validate a fresh start and write a concise walkthrough with report and replay screenshots.

**Complete when:** the documented fresh-clone path works and a 3–5 minute demo shows the two
roles, replay, rejected mutation, detected corruption, and failing RBAC mutation test.

## Extensions after the complete demo

Prioritize an independently stored audit checkpoint, with a precise trust model, if the next
goal is stronger integrity evidence. Prioritize an external identity provider if the next goal
is deployment realism. Kubernetes remains the final deployment exercise. The optional API
embedding adapter can also wait until the default provider path is complete.

Do not add more agent tools or infrastructure until the core acceptance checks pass.

## First implementation slice

1. Adjust ignore rules so documentation and evaluation datasets can be versioned; keep generated
   reports and local task notes ignored. Add links to the committed planning material.
2. Complete project metadata and add only the dependencies required for configuration and providers.
3. Implement validated Settings, typed chat/embedding contracts, and scripted fake providers.
4. Test invalid configuration and the provider contracts without network access; run Ruff and pytest.
5. Continue with real adapters and the database/ingestion slice after these checks pass.

Track execution in `.claude/tasks/todo.md`. This file is the committed release plan; the local
checklist contains implementation tasks. Neither document should mark a milestone complete
until its stated checks have run successfully.
