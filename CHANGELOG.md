# Changelog — br/acc Open Graph

All notable changes to this project are documented in this file.
Entries are dated and ordered from newest to oldest.

---

## 2026-05-17 — Maintenance pass

### Fixed
- `GET /health` now returns `{"status": "ok", "version": "0.1.0"}` alongside the
  existing status field, consistent with the `/api/v1/meta/health` endpoint and
  monitoring integrations that expect a version in the health response.
- ETL runner (`runner.py`) now emits a `WARNING` log when `--streaming` is passed
  to a pipeline that does not implement `run_streaming()`, instead of silently
  falling back to standard `run()` with no diagnostic output.

### Improved
- `CagedPipeline.transform()` stub replaced with a descriptive docstring explaining
  that transformation is intentionally deferred to per-chunk processing inside
  `load()`, clarifying the design for contributors.

### Added
- `PENDING.md` — project purpose, current state, and prioritised pending work items.
- `ARCHITECTURE.md` — full module map, graph schema, API router table, ETL pipeline
  architecture, security controls, and infrastructure summary.
- `CHANGELOG.md` — this file.
- `docs/bmad/prd.md` — BMAD v6 Product Requirements Document derived from codebase.
- `docs/bmad/architecture.md` — BMAD v6 Architecture Document.

---

## 2026-03-02 — Upstream convergence

- Merged upstream convergence sync; hardened auth, sharing, search, and ETL
  security controls; added bootstrap-all orchestration and public trust hardening;
  full-text search Lucene escaping; API version included in `/health` meta router.

## 2026-03-01 — CI effectiveness

- CI pipeline improvements; corrected shields.io badge repo references.
