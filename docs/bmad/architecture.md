# Architecture Document — br/acc Open Graph

**BMAD v6 | Date: 2026-05-17 | Status: Living document**

---

## 1. System Context

```
Actors
  [Civic researcher / journalist / developer]
       │
       ▼
  React 19 SPA  ──HTTP──►  FastAPI (bracc-api)  ──Bolt──►  Neo4j 5
  (frontend/)              (api/src/bracc/)                 (:7687)
                                                               ▲
                                                               │ Bolt
  ETL CLI (bracc-etl) ──────────────────────────────────────────┘
  (etl/src/bracc_etl/)
       ▲
       │ HTTP / FTP / ZIP
  Brazilian government portals (45+ sources)
```

---

## 2. Component Breakdown

### 2.1 ETL (`etl/`)

**Package:** `bracc_etl` (installed as `bracc-etl` CLI)

| Module | Responsibility |
|---|---|
| `base.Pipeline` | Abstract base: extract / transform / load lifecycle; `IngestionRun` upsert |
| `runner.py` | Click CLI; `PIPELINES` registry dict; `bracc-etl run/sources/download` commands |
| `loader.Neo4jBatchLoader` | Chunked UNWIND-based Neo4j batch writes with error isolation |
| `linking_hooks.run_post_load_hooks` | Post-load entity resolution (SAME_AS); tier-configurable |
| `entity_resolution/` | Splink-based probabilistic matching (optional extra) |
| `transforms/` | Shared pandas utilities: date formatting, CNPJ/CPF formatting, deduplication, value sanitisation, name normalisation |
| `schemas/` | Pandera DataFrameSchema definitions for CNPJ, TSE, Transparência, DOU, PGFN inputs |
| `pipelines/<source>.py` | 45 concrete pipeline implementations |

**Data flow per pipeline:**
```
extract()          transform()          load()
  │                   │                  │
  ▼                   ▼                  ▼
Download raw   Normalise/dedup    UNWIND batch →
files to       DataFrame in       Neo4j MERGE
data/<source>/ memory (or noop    in chunk_size
               if streaming)      chunks
```

### 2.2 API (`api/`)

**Package:** `bracc` (FastAPI application)

| Module | Responsibility |
|---|---|
| `main.py` | FastAPI app factory; lifespan (Neo4j init + schema bootstrap); middleware wiring |
| `config.py` | Pydantic Settings; all env var defaults and validation |
| `dependencies.py` | Async Neo4j driver lifecycle; `get_session` dependency; JWT auth resolution |
| `routers/` | Route handlers (see section 2.3) |
| `services/neo4j_service.py` | `CypherLoader` (named .cypher files); `execute_query`; `sanitize_props` |
| `services/auth_service.py` | bcrypt hashing; JWT encode/decode; Neo4j user CRUD |
| `services/public_guard.py` | Public mode enforcement; person label detection; CPF/CNPJ policy |
| `services/source_registry.py` | CSV-backed source registry; summary metrics |
| `services/intelligence_provider.py` | Pattern engine dispatch (community / full tier) |
| `services/investigation_service.py` | Investigation workspace CRUD; share token management |
| `services/pdf_service.py` | PDF export via WeasyPrint + Jinja2 templates |
| `middleware/cpf_masking.py` | Response-body CPF masking (regex, PEP-exempt) |
| `middleware/rate_limit.py` | SlowAPI limiter configuration |
| `middleware/security_headers.py` | CSP, HSTS, X-Frame-Options injection |
| `queries/*.cypher` | All Cypher queries (loaded by name; not inline strings) |
| `models/` | Pydantic response models: entity, graph, search, pattern, user |

### 2.3 Router Responsibilities

| Router | Key endpoints |
|---|---|
| `public` | `/api/v1/public/meta`, `/graph/company/{ref}`, `/patterns/company/{ref}` |
| `meta` | `/api/v1/meta/health`, `/stats` (cached 5 min), `/sources` |
| `auth` | `/register`, `/login`, `/me`, `/logout` |
| `search` | `/api/v1/search?q=` (fulltext, paginated, Lucene-escaped) |
| `entity` | Entity detail, connections, timeline, exposure score |
| `graph` | Authenticated subgraph expansion |
| `patterns` | Pattern engine (feature-flagged) |
| `baseline` | Sector and region aggregate comparisons |
| `investigation` | Workspace CRUD, entity add/remove, annotation/tag, share |

### 2.4 Frontend (`frontend/`)

**Stack:** Vite 5, React 19, TypeScript, Zustand (state), i18next (i18n), Vitest (tests)

| Directory | Content |
|---|---|
| `src/pages/` | Landing, Search, GraphExplorer, EntityAnalysis, Patterns, Investigations, Baseline, Login, Register, SharedInvestigation |
| `src/components/` | Shared UI components |
| `src/api/` | Typed API client functions |
| `src/stores/` | Zustand stores |
| `src/hooks/` | Custom React hooks |
| `src/actions/` | Server-action-style async operations |
| `src/i18n.ts` | i18next configuration (pt-BR / en) |

### 2.5 Infrastructure (`infra/`)

| File | Purpose |
|---|---|
| `docker-compose.yml` | Development stack: neo4j, api, frontend, etl (profile) |
| `docker-compose.prod.yml` | Production stack with Caddy reverse proxy |
| `neo4j/init.cypher` | Neo4j startup Cypher |
| `scripts/seed-dev.sh` | Load deterministic demo seed data |
| `scripts/backup-neo4j.sh` | Neo4j volume snapshot |

---

## 3. Key Design Decisions

### 3.1 Named Cypher files, not inline strings
All database queries live in `api/src/bracc/queries/*.cypher`. This separates query
logic from Python, enables syntax highlighting and linting of Cypher independently,
and makes the query surface auditable. `CypherLoader` caches files after first read.

### 3.2 Chunk-based ETL loading
Large sources (CNPJ: ~50 GB uncompressed) are processed in configurable chunks
(default 50,000 rows). Pipelines that require streaming (e.g. CAGED) defer
transformation to per-chunk processing inside `load()`, keeping memory usage bounded.

### 3.3 Public mode and tiered access
A single settings flag (`PUBLIC_MODE`) shifts the API into a privacy-safe community
deployment mode. Fine-grained flags (`PUBLIC_ALLOW_PERSON`, `PUBLIC_ALLOW_ENTITY_LOOKUP`,
`PUBLIC_ALLOW_INVESTIGATIONS`, `PATTERNS_ENABLED`) allow selective capability exposure
without code changes.

### 3.4 Response-layer CPF masking
CPF masking is applied as a Starlette middleware on the response body, not at the
query level. This ensures that CPFs surfacing through any route — including future
routes — are always masked, without requiring each endpoint to implement masking logic.
PEPs are exempted because their CPFs are already public under LGPD Art. 7.

### 3.5 Source registry as a CSV contract
`docs/source_registry_br_v1.csv` is the single source of truth for all data sources:
their implementation state, load state, quality status, legal basis, and access mode.
The API exposes this registry via `/api/v1/meta/sources` and includes summary metrics
in `/api/v1/public/meta`, making pipeline health externally observable.

---

## 4. Data Flow — Search Request

```
Browser → GET /api/v1/search?q=petrobras
    │
    ├─ RateLimitMiddleware (30/min)
    ├─ CPFMaskingMiddleware (no-op for non-JSON path skipped)
    │
    ▼
search_entities()
    ├─ _escape_lucene(q) → safe query string
    ├─ execute_query(session, "search", {...})
    │       └─ CypherLoader.load("search") → search.cypher
    │               └─ Neo4j fulltext index query
    ├─ execute_query_single(session, "search_count", {...})
    │
    ├─ Filter: hide_person_entities if PUBLIC_MODE
    ├─ Build SearchResult list (name extraction per entity type)
    │
    └─ return SearchResponse → CPFMaskingMiddleware → Browser
```

---

## 5. Deployment Topology (Production)

```
Internet
   │
   ▼
Caddy (TLS termination + reverse proxy)
   ├── / → frontend:3000 (nginx)
   └── /api/ → api:8000 (uvicorn)
                   │
               neo4j:7687 (Bolt, internal only)
```

Neo4j data volume is persistent. Backups via `infra/scripts/backup-neo4j.sh`
(Docker volume snapshot). Neo4j is not exposed on any public port.
