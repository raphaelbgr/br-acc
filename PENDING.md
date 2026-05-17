# PENDING — br/acc Open Graph

**Date:** 2026-05-17
**Branch:** auto/advance-2026-05-17

## Project Purpose

Open-source graph infrastructure that cross-references Brazilian public databases
(CNPJ company registry, TSE electoral data, Portal da Transparência, sanctions lists,
procurement, health, education, environment, judiciary, and more) and normalises them
into a Neo4j graph. Exposes data via a FastAPI backend, a React 19 frontend, and
45+ ETL pipeline modules. Privacy-first; LGPD compliant; public-safe defaults.

## Current State

- 45 ETL pipeline modules implemented and registered in the runner
- FastAPI API with auth, search, graph, entity, patterns, investigation, and meta routers
- React 19 + Vite + TypeScript frontend
- Docker Compose local stack; bootstrap-demo and bootstrap-all orchestration scripts
- Source registry CSV tracking implementation/load/quality state of all data sources
- Unit test suites for API and ETL; integration tests skipped unless Neo4j is available
- Working CI workflow (GitHub Actions)
- LGPD, ethics, security, legal, abuse-response documentation in place

## Prioritised Pending Work Items

### P1 — Health endpoint missing API version field
The `GET /health` route in `api/src/bracc/main.py` returns `{"status": "ok"}` only.
The `GET /api/v1/meta/health` route returns `{"neo4j": "connected"}`.
PR #28 added version to `/health` but the main health endpoint at root still has no version.
Consistent versioned health responses are needed for monitoring integrations.

### P2 — Stats cache is module-level global (not thread/process-safe)
`api/src/bracc/routers/meta.py` uses `_stats_cache` and `_stats_cache_time` as module-level
mutable globals. Under multiple uvicorn workers (production), each worker holds its own stale
cache with no coordination. Should use `app.state` or a shared cache keyed by the FastAPI
app instance.

### P3 — ETL runner does not surface `streaming` and `history` flags to all pipelines
`runner.py` passes `history` to pipeline constructors but not `streaming` mode per-pipeline —
only calls `run_streaming()` if the attribute exists. There is no logging or CLI feedback
when a pipeline requested with `--streaming` does not support it, making failures silent.

### P4 — Missing `.env.example` referenced in README Quick Start
`README.md` quick start says `cp .env.example .env` but `.env.example` is not committed to
the repository. Developers following the README get a `cp: .env.example: No such file` error.

### P5 — `test_base.py` in ETL tests is an empty stub
`etl/tests/test_base.py` exists with only an import; the base pipeline interface has no
dedicated unit tests covering the `_upsert_ingestion_run` path, the error-status branch, or
the `run()` orchestration flow.

### P6 — `caged.py` pipeline has an explicit `pass` in `transform()`
`etl/src/bracc_etl/pipelines/caged.py` line 103 has `pass  # Transform happens per chunk in load()`
which is correct behaviour but looks like an unfinished stub to reviewers. A brief docstring
clarifying the design decision would eliminate ambiguity.

### P7 — `docs/bmad/` directory does not exist
No BMAD v6 PRD or architecture documents have been produced. These are required for
project governance and institutional review.

### P8 — No CHANGELOG
There is no `CHANGELOG.md` tracking what has changed across releases.

### P9 — No ARCHITECTURE.md
There is no `ARCHITECTURE.md` documenting module layout and data flow beyond the brief
mermaid diagram in README.
