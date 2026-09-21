# Contract: Telemetry & Ops Analytics

- Status: Proposed
- Updated: 2026-09-20

> Event taxonomy and the ops analytics surface. Companion: [admin-panel.md](admin-panel.md) · observability. Rationale: ADR-0010.

## 1. Taxonomy

Events are categorized by **domain × side** (per GameDevMind `2.2.3`):

| Domain | Examples |
|---|---|
| `system` | errors, lifecycle, `loop_progress` |
| `feature` | feature entry counts, play failure rate, activity participation |
| `ops` | activation, retention, revenue (server-side) |

- **Side**: `client` (login/register/pay/key actions) and `server` (combat, resource changes, system events).
- Uploads are structured, PII-free, and non-blocking; consent is required (`Privacy.consent()`).

## 2. Existing (implemented)

- `telemetry_sink` (ingest, consent-gated, capped) → `g3/telemetry_<user>`.
- `telemetry_report` (cross-user `loop_progress` funnel aggregation).

## 3. Aggregation (Postgres + Grafana — this phase)

- Aggregations for the ops panel are computed via **Postgres** (mirrored/rolled-up from telemetry) and visualized in **Grafana**. No ClickHouse/Kafka in this phase (see ADR-0010).
- Provided ops actions: `telemetry.overview` (DAU/events), `telemetry.funnel` (step counts), `telemetry.query` (filter by step/time).
- Retention and rollup window are configured in the `admin` schema; dashboards live in Grafana.

## 4. Crash / performance

- `CrashGuard` detection + `crash_report` upload already exist; symbolication is a follow-up.
- Server/perf metrics go to **Prometheus → Grafana** (OpenTelemetry as the pipeline).

## 5. Migration path

- If data volume outgrows Postgres, introduce a columnar store (ClickHouse) behind the same `telemetry.*` actions; the contract surface does not change.