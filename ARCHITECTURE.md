# Architecture — br/acc Open Graph

## Overview

br/acc is a multi-layer data platform built on Neo4j that ingests, normalises, and
cross-references Brazilian public databases. It exposes the resulting graph through a
FastAPI backend and a React 19 frontend, with full Docker Compose orchestration.

---

## Layer Map

```
┌─────────────────────────────────────────────────────────────┐
│  Public Data Sources (45+ Brazilian government portals)     │
│  CNPJ · TSE · Portal da Transparência · BNDES · IBAMA · ... │
└───────────────────────────┬─────────────────────────────────┘
                            │  HTTP / FTP / ZIP download
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  ETL Layer  (bracc-etl Python package)                      │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐  │
│  │ Pipeline │ │Transforms│ │  Schemas  │ │Neo4jBatchLoad│  │
│  │  base.py │ │ (pandas) │ │ (pandera) │ │  loader.py   │  │
│  └──────────┘ └──────────┘ └───────────┘ └──────────────┘  │
│  runner.py — CLI entry point (bracc-etl run --source <id>)  │
└───────────────────────────┬─────────────────────────────────┘
                            │  Bolt (neo4j driver)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Graph Database  (Neo4j 5 Community)                        │
│  schema_init.cypher — constraints + indexes on startup      │
│  Named .cypher files loaded by CypherLoader at runtime      │
└─────────┬──────────────────────────────────────────┬────────┘
          │ Bolt (async neo4j driver)                 │
          ▼                                           ▼
┌─────────────────────────────┐     ┌─────────────────────────┐
│  API Layer (bracc-api)      │     │  Investigation Service   │
│  FastAPI + uvicorn          │     │  (shared-link, PDF gen) │
│  ┌───────────────────────┐  │     └─────────────────────────┘
│  │  Routers              │  │
│  │  auth / public /      │  │
│  │  entity / search /    │  │
│  │  graph / patterns /   │  │
│  │  meta / baseline /    │  │
│  │  investigation        │  │
│  └───────────────────────┘  │
│  Middleware:                 │
│  CPFMasking · RateLimit ·    │
│  SecurityHeaders · CORS      │
└─────────────┬───────────────┘
              │ HTTP (JSON)
              ▼
┌─────────────────────────────────────────────────────────────┐
│  Frontend (Vite + React 19 + TypeScript)                    │
│  Pages: Landing · Search · GraphExplorer · EntityAnalysis   │
│         Patterns · Investigations · Baseline · Login        │
│  Stores: Zustand  │  I18n: i18next (pt-BR / en)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Graph Schema — Core Node Labels

| Label | Key Property | Source |
|---|---|---|
| `Company` | `cnpj` | CNPJ, DATASUS, BNDES, COMPRASNET, … |
| `Person` | `cpf` | CNPJ socios, TSE, PEP-CGU, CEAF |
| `Partner` | `partner_id` | CNPJ socios (CPF-masked public) |
| `Contract` | `contract_id` | Transparência, COMPRASNET, PNCP |
| `Amendment` | `amendment_id` | Transparência |
| `Sanction` | `sanction_id` | CEIS/CNEP, OFAC, EU, UN, WorldBank |
| `Finance` | `finance_id` | BNDES, TransfereGov, SICONFI |
| `PublicOffice` | `office_id` | Servidores, PEP-CGU |
| `Health` | `cnes_code` | DATASUS/CNES |
| `Education` | `school_id` | INEP |
| `Embargo` | `embargo_id` | IBAMA |
| `Election` | `election_id` | TSE candidatos/votos |
| `IngestionRun` | `run_id` | Internal operational traceability |
| `Investigation` | `id` | User-created workspace |
| `User` | `id` | Auth (bcrypt + JWT) |

Common relationship types: `OWNS`, `HAS_PARTNER`, `RECEIVED_CONTRACT`, `APPLIED_SANCTION`,
`RECEIVED_FINANCE`, `HOLDS_OFFICE`, `DONATED_TO`, `SAME_AS`.

---

## API Router Map

| Prefix | Module | Auth required |
|---|---|---|
| `/health` | `main.py` | No |
| `/api/v1/public/` | `routers/public.py` | No |
| `/api/v1/meta/` | `routers/meta.py` | No |
| `/api/v1/auth/` | `routers/auth.py` | Login/register |
| `/api/v1/search` | `routers/search.py` | Optional |
| `/api/v1/entity/` | `routers/entity.py` | Yes |
| `/api/v1/graph/` | `routers/graph.py` | Yes |
| `/api/v1/patterns/` | `routers/patterns.py` | Yes |
| `/api/v1/baseline/` | `routers/baseline.py` | Yes |
| `/api/v1/investigations/` | `routers/investigation.py` | Yes |

All Cypher queries are stored as named `.cypher` files under `api/src/bracc/queries/`
and loaded at runtime by `CypherLoader` (cached after first read per process).

---

## ETL Pipeline Architecture

Every pipeline extends `bracc_etl.base.Pipeline` and implements three methods:

- `extract()` — download or locate raw data files
- `transform()` — normalise/deduplicate in memory (or no-op if done per-chunk in `load()`)
- `load()` — write to Neo4j via `Neo4jBatchLoader` in configurable chunk sizes

The `runner.py` CLI registers all 45 pipeline classes in the `PIPELINES` dict and
exposes them via `bracc-etl run --source <name>`. Post-load entity linking is
triggered via `run_post_load_hooks()`.

Shared transform utilities live in `bracc_etl/transforms/`:
- `date_formatting.py` — normalise date strings to ISO-8601
- `deduplication.py` — dedup DataFrames before load
- `document_formatting.py` — format/validate CNPJ and CPF strings
- `name_normalization.py` — trim/uppercase legal names
- `value_sanitization.py` — coerce numeric fields, handle BRL comma-decimal

Schema validation (input contracts) lives in `bracc_etl/schemas/`.

---

## Security & Privacy Controls

- **CPFMaskingMiddleware** — masks all 11-digit CPF patterns in JSON responses;
  PEP CPFs are left unmasked per LGPD Art. 7 public interest provisions.
- **Public mode** — `PUBLIC_MODE=true` hides Person nodes, disables entity lookup,
  patterns, and investigations unless explicitly enabled per flag.
- **Rate limiting** — SlowAPI: 60/min anonymous, 300/min authenticated, 10/min auth endpoints.
- **JWT** — HS256, cookie+bearer, startup fails if key < 32 chars outside dev/test.
- **Lucene escaping** — search queries are escaped before being passed to Neo4j fulltext indexes.
- **Neutrality** — automated `make neutrality` check rejects severity/judgment keywords in source.

---

## Infrastructure

| Component | Technology |
|---|---|
| Container orchestration | Docker Compose (dev), `docker-compose.prod.yml` + Caddy (prod) |
| Neo4j | 5 Community; APOC plugin; heap/pagecache tuned via env vars |
| API | uvicorn (standard), async Neo4j driver, connection pool 50 |
| Frontend | Nginx (prod container), Vite (dev server) |
| CI | GitHub Actions: ruff, mypy, pytest (unit only), frontend lint |
