# Capabilities plan — base components

Status: draft for review · 2026-10-12

What each base component must do, in which phase, and how we know it is done. The architecture itself is in [architecture.md](architecture.md).

## Phases at a glance

| Phase | Goal | Exit gate |
|---|---|---|
| 0 · Foundations | Repo, local stack skeleton, sample data, Wren spike | `make up` starts MinIO, metastore and Trino; the spike questions in architecture.md §6 have written answers |
| 1 · Data and semantics | Warehouse, Trino, semantic definitions, Wren core, gateway skeleton, agent v0, eval set v0 | The agent answers the eval set end to end, only through the gateway, with measured accuracy |
| 2 · Guardrails | Auth, access control, rate limiting, query guard, LLM controls, audit — enforced | Every guardrail has passing negative tests; zero policy bypasses in a red-team pass |
| 3 · Management in product | Semantic-definition management and guardrail management as product surfaces | A data owner can add a table's definition and an admin can change a policy without a code change |
| Later | Product features (Q&A UI, KPIs, reports, RCA and so on) | Planned separately |

Phase 1 already routes everything through the gateway with pass-through policies, so Phase 2 switches enforcement on rather than re-plumbing calls.

## Phase 0 · Foundations

| Capability | Done when |
|---|---|
| Repository layout, Python tooling (uv, ruff, pytest), pre-commit, CI running lint and tests | A fresh clone passes CI |
| Local stack skeleton in `deploy/local/docker-compose.yml`, with pinned image versions and separate `agent` (internal), `gateway` and `data` networks | `make up` / `make down` work on a laptop with 16 GB RAM |
| Synthetic credit card dataset: daily aggregate tables (no customer-level rows, no PII) across nine business domains, 36 months of history, at least 10,000 rows per table. Kept local only: the generator, MinIO setup, schemas and data live in the git-ignored `local/` folder | Generator writes Parquet to MinIO deterministically from a seed; a validator checks schemas, row counts, PII absence and cross-table reconciliation |
| Wren spike | Written answers to the five questions in architecture.md §6, plus a go/adjust decision |
| Eval seed | 30 questions with verified SQL and expected result sets over the sample data |

## Phase 1 · Data and semantics

### Warehouse (S3 / MinIO)

| Capability | Done when |
|---|---|
| Bucket layout `warehouse/<schema>/<table>/`, Hive-style partitions where a date column exists | Documented in `warehouse/README.md`; sample data follows it |
| Table registration: a command that creates or updates the external table in the metastore from a Parquet prefix (schema inferred, then reviewed) | `warehouse register cards.transactions` makes it queryable in Trino |
| Partition sync for new date partitions | New partitions appear without manual DDL |
| Same layout works on real S3 by changing endpoint settings only | Verified once against a test S3 bucket |

### Trino

| Capability | Done when |
|---|---|
| `lake` catalog via the Hive connector over MinIO, Postgres-backed Hive Metastore | All sample tables queryable |
| Resource groups: `interactive` (agent queries) and `background` (builds, evals) with concurrency and memory limits | Background load cannot starve interactive queries in a load test |
| Query-level limits: max execution time, max scan size | Runaway query is killed with a clear error |
| Impersonation: the gateway's service user may set the session user | `current_user` in Trino equals the end user for gateway queries |
| File-based access control as a backstop (deny everything except the gateway and Wren metadata user) | Direct connections with any other user are refused |

### Semantic definitions and Wren core

| Capability | Done when |
|---|---|
| YAML schema for a table definition plus context (see architecture.md §2.2), with a JSON Schema for editors and CI | Invalid files fail CI with readable errors |
| `semantic validate`: every table, column and type in the YAML matches Trino; relationships join; metrics compile; sample-question SQL runs | Runs in CI against the local stack |
| `semantic build`: compiles to the Wren MDL manifest and the context index | Wren loads the manifest; the index is searchable |
| Context index: keyword plus embedding search over tables, columns, metrics, rules and sample questions (Postgres with pgvector; embeddings via the gateway's LLM proxy) | Top-5 retrieval contains the right tables for at least 90% of eval questions |
| Wren core deployed plan-only; dry-plan from MDL SQL to Trino SQL | All sample-question SQL round-trips through dry-plan and runs |
| Definitions for every sample table, reviewed by a named owner | 100% of columns have descriptions; every PII column is tagged |

### Gateway skeleton

| Capability | Done when |
|---|---|
| FastAPI service exposing the v1 routes in architecture.md §2.3 with OpenAPI docs | Contract tests pass for every route |
| Request chain in place with pass-through implementations: identity propagation, counters, allow-all policy decision, audit write | Every call produces one audit row with user, agent, route, trace id and latency |
| Trino connector (the only Trino client in the system), with the query guard's static checks: single SELECT, forced LIMIT, timeout, EXPLAIN | Non-SELECT and multi-statement SQL are rejected even in Phase 1 |
| Context API reading the compiled MDL and context index | Agent can list, describe and search tables |
| LLM proxy for Claude, plus the OpenAI-compatible shim; provider keys only in gateway config | Agent and Wren make model calls without holding a key |
| OpenTelemetry tracing across gateway, agent, Wren and Trino query ids | One trace shows a question from ask to answer |

### Agent v0

| Capability | Done when |
|---|---|
| LangGraph graph: understand → retrieve context → draft SQL → plan → run → repair (max 2) → answer | Runs end to end against the local stack |
| Tools are gateway clients only; container on the internal network with no secrets but its service credential | A network test from inside the agent container cannot reach Trino, MinIO, Wren or the internet |
| Answer contract: text, the SQL that ran, the interpretation (tables, filters, metric definitions used), query id | Every answer in the eval run includes all four |
| Says "can't answer" or asks one clarifying question instead of guessing | Unanswerable eval questions are declined, not fabricated |
| Numbers in the narrative are checked against the result set before returning | Narrative with an unmatched figure is regenerated or flagged |

### Evals v0

| Capability | Done when |
|---|---|
| Eval runner scoring execution accuracy (result-set match, not SQL text match), latency and token cost | `make eval` prints a scorecard |
| Runs in CI on changes to prompts, agent code, semantic files or model config | A regression fails the build |

**Phase 1 targets** on the sample data: execution accuracy at least 80% on the eval set, p90 answer latency under 20 seconds, zero calls from the agent to anything but the gateway.

## Phase 2 · Guardrails enforced

### Authentication and identity

| Capability | Done when |
|---|---|
| JWT validation with an OIDC-compatible issuer; locally the gateway's dev issuer or Keycloak | Expired, wrongly signed or wrong-audience tokens get 401 |
| Service credentials for the agent and internal services | Rotation without downtime |
| Delegated tokens: minted at `/v1/ask`, bound to one user and conversation, valid for minutes, accepted only from the agent | A delegated token replayed from another service, or after expiry, is refused |
| Users, groups and roles in Postgres, with group sync from the IdP later | Policies can target users, groups and roles |

### Access control

| Capability | Done when |
|---|---|
| Policy model: table allow lists; column rules (allow, mask, deny); row filters as SQL predicates with user attributes (e.g. `region = :user.region`) | Policies stored, versioned and validated against the semantic layer |
| Default deny; policies derived from PII tags in semantic definitions (e.g. `pii: phone` masked unless the role allows it) | A new table with PII columns is masked before anyone writes a policy |
| Enforcement in `/v1/query`: resolve all referenced columns, deny or rewrite, inject row filters, apply masks | Negative-test suite: `SELECT *`, aliases, subqueries, CTEs, joins, `UNION`, functions over masked columns all stay masked or denied |
| Context filtering: forbidden tables and columns removed from context responses and search results | A user without access to a table cannot learn its name or columns through the agent |
| Trino backstop policies generated from the same policy store | Even a gateway bug cannot read a table the user's Trino rules deny |

### Rate limiting and quotas

| Capability | Done when |
|---|---|
| Token-bucket limits per user, per agent and per route, in Redis | 429 with `Retry-After` when exceeded |
| Concurrent-query cap per user and global, plus scan-size budget per user per day | Limits configurable per role |
| LLM token budgets per user and team, with an alert at 80% | Budget breach blocks further model calls with a clear message |

### Query guard (full)

| Capability | Done when |
|---|---|
| Static checks from Phase 1, plus deny lists for functions and system tables | `system.*`, `information_schema` probing and side-effecting functions are refused |
| Cost cap from `EXPLAIN` estimates, tuned on real plans | Cartesian joins and unpartitioned scans of large tables are rejected before running |
| Result shaping: row cap to the agent, size cap in bytes, masked columns listed in the response | Large results are truncated with a flag, never silently |

### LLM controls

| Capability | Done when |
|---|---|
| Model allowlist and aliases per role and environment | Unknown model names are refused |
| PII redaction on prompts (pattern and tag based) | Masked or PII-tagged values never appear in an outgoing prompt in tests |
| Result-egress rules: max rows and columns sent to the model; tagged columns never sent | Configurable per data classification |
| Prompt and response logging with retention controls; prompt versions recorded | Any answer can be traced to its prompt version and model |
| Provider fallback and timeouts | Provider outage degrades to a clear error, not a hang |

### Audit

| Capability | Done when |
|---|---|
| Append-only audit log (who, as which agent, what route, what SQL, which policy decisions, rows returned, tokens used) | Rows cannot be updated or deleted by the application role |
| Query history per user | A user can see what ran on their behalf |
| Export for compliance review | CSV or Parquet export by date range |

**Phase 2 targets**: zero policy bypasses in the negative-test suite and a red-team pass; gateway overhead under 50 ms p95 per call, excluding Trino and LLM time.

## Phase 3 · Management in the product

Semantic definitions and guardrails move from files edited by engineers to product surfaces, while Git stays the system of record.

### Semantic-definition management

| Capability | Done when |
|---|---|
| Table onboarding: point at a Parquet prefix, get a draft definition (columns, types, AI-suggested descriptions via the gateway), edit, submit | A data owner onboards a table without touching YAML |
| Review workflow: draft → in review → published, with a named approver; publishing opens a Git change and runs validate, build and evals | No definition reaches the agent without validation and an approver |
| Context editing: business rules, synonyms, caveats, sample questions with verified SQL | Edits show up in retrieval after publish |
| Drift detection: schema changes in Trino flag affected definitions | A dropped or renamed column opens a fix task |

### Guardrail management

| Capability | Done when |
|---|---|
| Admin console for users, groups, roles and policies, with a "who can see this column" view | Policy change takes effect without a deploy |
| Policy simulation: test a policy against a user and a query before publishing | Simulated decision matches the enforced one |
| Rate-limit and budget configuration per role and team | Changes are audited |
| Audit and usage views: queries, denials, LLM tokens and Trino CPU per team | Admin can answer "who saw this column last month" |

## Gateway API v1 summary

| Method and path | Caller | Phase |
|---|---|---|
| `POST /v1/ask` | Clients | 1 |
| `GET /v1/context/tables` · `GET /v1/context/tables/{name}` · `POST /v1/context/search` | Agent | 1 |
| `POST /v1/sql/plan` | Agent | 1 |
| `POST /v1/query` | Agent | 1 |
| `POST /v1/llm/messages` · `POST /v1/llm/openai/chat/completions` · `POST /v1/llm/openai/embeddings` | Agent, internal services | 1 |
| `POST /v1/auth/token` (dev issuer) | Clients, local only | 2 |
| `GET /v1/me/queries` | Clients | 2 |
| `/v1/admin/policies` · `/v1/admin/principals` · `/v1/admin/audit` | Admins | 3 |
| `/v1/semantic/*` (definition management) | Data owners | 3 |

An MCP surface for the agent's tools can sit on the same gateway later; it adds a protocol, not a new path to the data.

## Proposed repository layout

```
ai_analytics_platform/
├── docs/                    architecture, plans, decision records (docs/adr/)
├── deploy/local/            docker-compose, Trino, metastore and MinIO config
├── warehouse/               sample data generator, table registration
├── semantic/                one YAML per table, schema, build and validate CLI
├── services/
│   ├── gateway/             FastAPI: request chain, routes, Trino connector, LLM proxy
│   └── agent/               LangGraph agent, gateway tool clients
├── evals/                   questions, expected results, runner
└── Makefile                 up, down, seed, semantic-build, eval, test
```

## Risks

| Risk | Mitigation |
|---|---|
| Wren core cannot plan without a live connection, or its scope shifts again | Phase 0 spike; the gateway hides Wren behind `/v1/sql/plan` and `/v1/context/*`, so it can be replaced by our own MDL compiler without agent changes |
| SQL-rewrite access control misses a construct | Trino backstop policies from the same store; a negative-test suite that grows with every bug |
| Gateway becomes a bottleneck or single point of failure | Stateless replicas; Redis and Postgres hold state; overhead budget tracked in Phase 2 |
| Synthetic data hides real-world mess | Run Phase 1 evals on one real, de-identified mart before Phase 2 ends |
| Context index grows stale against the warehouse | `semantic validate` in CI; drift detection in Phase 3 |

## Open questions

1. Is the target cloud AWS (S3, Glue, EKS), or should the cloud column stay provider-neutral?
2. Which identity provider will issue user tokens in Phase 2?
3. May query results (not just schema) be sent to the LLM provider, and must model calls stay in a specific region?
4. Who owns semantic definitions per domain, and who approves them?
5. Do we need an offline local model from Phase 1, or is the Claude API acceptable for local development?
