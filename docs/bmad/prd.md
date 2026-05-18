# Product Requirements Document — br/acc Open Graph

**BMAD v6 | Date: 2026-05-17 | Status: Living document**

---

## 1. Problem Statement

Brazil's public data is legally open but practically inaccessible: it is scattered
across dozens of government portals, inconsistently formatted, and lacks any unified
query surface. Civic researchers, journalists, compliance professionals, and developers
must manually download, join, and reconcile multiple multi-gigabyte datasets to answer
even basic questions such as "which companies received contracts from a sanctioned entity?"
or "which public servants hold shares in BNDES-funded companies?".

---

## 2. Product Vision

br/acc is an open-source graph infrastructure that cross-references Brazil's public
databases and exposes a single queryable graph. It does not interpret, score, or accuse
— it surfaces connections and lets users draw their own conclusions.

**North star:** Any person with a CNPJ or entity name can trace connections across
company registry, electoral donations, government contracts, sanctions, and public
finances in a single query without downloading a single CSV.

---

## 3. Users & Personas

| Persona | Goal | Key workflow |
|---|---|---|
| Civic journalist | Find undisclosed connections between contractors and officials | Search CNPJ → graph expand → export investigation |
| Compliance analyst | Screen counterparties against sanction lists + government contracts | Entity lookup → exposure factors → PDF report |
| Developer / researcher | Build downstream analytics on normalised Brazilian data | API access → ETL BYO-data ingestion |
| Open-data contributor | Add a new data source pipeline | Fork → implement Pipeline subclass → PR |

---

## 4. Functional Requirements

### 4.1 ETL

| ID | Requirement |
|---|---|
| F-ETL-01 | Support 45 data source pipelines covering company registry, elections, transparency, sanctions, procurement, health, education, environment, labor, judicial, and offshore data |
| F-ETL-02 | Each pipeline must implement extract / transform / load and register an `IngestionRun` node with status, row counts, and timestamps |
| F-ETL-03 | Pipelines must accept `--limit` for dev/test truncation and `--chunk-size` for memory control |
| F-ETL-04 | A `bracc-etl sources` CLI command must list all registered pipelines with their latest ingestion status |
| F-ETL-05 | Streaming mode (large CSVs) must degrade gracefully with a logged warning when a pipeline does not implement `run_streaming()` |

### 4.2 Graph

| ID | Requirement |
|---|---|
| F-GR-01 | Core node labels: Company, Person, Partner, Contract, Amendment, Sanction, Finance, PublicOffice, Health, Education, Embargo, Election, IngestionRun, Investigation, User |
| F-GR-02 | Uniqueness constraints and fulltext indexes defined in `schema_init.cypher`, applied idempotently on startup |
| F-GR-03 | Entity resolution (`SAME_AS` relationships) via post-load linking hooks |

### 4.3 API

| ID | Requirement |
|---|---|
| F-API-01 | `GET /health` returns `{status, version}` |
| F-API-02 | `GET /api/v1/public/meta` returns node counts, relationship count, and source health summary without authentication |
| F-API-03 | `GET /api/v1/public/graph/company/{cnpj_or_id}` returns the subgraph up to depth 3 for a given company |
| F-API-04 | `GET /api/v1/search?q=` supports fulltext search with pagination, type filter, and Lucene-safe escaping |
| F-API-05 | Auth: JWT + HttpOnly cookie; bcrypt password hashing; invite-code-gated registration |
| F-API-06 | Rate limits: 60/min anonymous, 300/min authenticated, 10/min auth endpoints |
| F-API-07 | CPF numbers in responses must be masked to `***.***.XXX-XX` except for PEPs |
| F-API-08 | Pattern engine (contract concentration, sanction still receiving, split contracts, etc.) behind `PATTERNS_ENABLED` feature flag |

### 4.4 Frontend

| ID | Requirement |
|---|---|
| F-FE-01 | Search page with entity type filter, pagination, and graph preview |
| F-FE-02 | Graph explorer with interactive node/edge expansion |
| F-FE-03 | Entity analysis page showing timeline, exposure factors, and connections |
| F-FE-04 | Investigations workspace: create, annotate, share via token-protected link |
| F-FE-05 | Baseline page for sector/region aggregate comparison |
| F-FE-06 | i18n: pt-BR (default) and English |

### 4.5 Privacy & Compliance

| ID | Requirement |
|---|---|
| F-PRIV-01 | `PUBLIC_MODE=true` hides all Person-labelled nodes and disables entity lookup, investigations, and patterns unless each is explicitly re-enabled |
| F-PRIV-02 | All data sources backed by Brazilian legal instruments (CF/88, LAI, LGPD Art. 7, LC 131/2009) |
| F-PRIV-03 | Neutrality gate: no severity/judgment keywords in source code |
| F-PRIV-04 | No personal data stored beyond what is in the public source datasets |

---

## 5. Non-Functional Requirements

| ID | Requirement |
|---|---|
| NF-01 | Local bootstrap (`make bootstrap-demo`) must complete in < 10 minutes on a standard developer laptop |
| NF-02 | API unit tests must run without a live Neo4j instance |
| NF-03 | All Python code must pass ruff lint and mypy strict type-check |
| NF-04 | Docker images must build from scratch without external network dependencies beyond official registries |
| NF-05 | The graph must scale to 40M+ nodes on a 64 GB RAM server with Neo4j heap/pagecache tuned via env vars |

---

## 6. Out of Scope

- Pre-populated production Neo4j dump distribution
- Guaranteed uptime of third-party government data portals
- Institutional/private modules (operational runbooks, advanced analytics)
- Personal data collection beyond what is already public
