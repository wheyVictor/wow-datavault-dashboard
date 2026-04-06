# WoW Data Vault Dashboard — Design Spec

**Date:** 2026-04-06
**Status:** Draft — pending user review
**Author:** Paulo Victor Silva

---

## 1. Executive Summary

Extracts World of Warcraft combat data from the Warcraft Logs GraphQL API, models it as a Data Vault 2.0 in PostgreSQL using dbt, orchestrates pipelines with Apache Airflow, and serves an interactive visualization portal built with Next.js 15 / React 19. The standout feature is a real-time pipeline monitoring dashboard with animated DAG visualizations.

### Key Viability Finding

**automate_dv does NOT support PostgreSQL or DuckDB.** It only supports Snowflake, BigQuery, and SQL Server. This is the single biggest constraint. We have two viable paths forward (see Section 4).

---

## 2. Data Source: Warcraft Logs API

### Confirmed Details

- **API Type:** GraphQL v2 (`https://www.warcraftlogs.com/api/v2/client`)
- **Auth:** OAuth 2.0 Client Credentials grant (free registration at `https://www.warcraftlogs.com/api/clients`)
- **Cost:** Free
- **Rate Limits:** Points-per-hour system (~3,600 points/hour). Complex queries cost more points. Monitor via `rateLimitData` query.
- **Data Format:** JSON (standard GraphQL responses)

### Available Data

| Entity | Description | API Path |
|--------|-------------|----------|
| Reports | Combat log sessions (code-identified) | `reportData.report(code)` |
| Fights | Individual encounters/pulls within a report | `report.fights` |
| Events | Granular combat events (damage, healing, casts) | `report.events` |
| Characters | Player characters with server/region | `characterData.character(name, server, region)` |
| Guilds | Guild info, reports, attendance | `guildData.guild(name, server, region)` |
| Rankings | DPS/HPS percentile per encounter | `characterData.character.encounterRankings` |
| Zones | Raid/dungeon instance definitions | `worldData.zones` |
| Encounters | Boss fight definitions | `worldData.encounters` |
| Classes/Specs | Game class and specialization data | `gameData.classes` |

### Ingestion Strategy

- **Python scripts** using `requests` or `gql` (Python GraphQL client) with OAuth token management
- **Incremental ingestion:** Track last-fetched report codes/timestamps, only pull new data
- **Rate limit handling:** Queue requests, respect points budget, monitor via `rateLimitData` query
- **Pagination:** Use `nextPageTimestamp` for event data pagination
- **Orchestration:** Airflow DAGs schedule and coordinate ingestion + transformation

### Caching Strategy (Critical — ~3,600 points/hour limit)

The WCL API uses a points-per-hour budget, not simple request counting. Complex queries cost more points. Caching is essential.

**Layer 1 — Local filesystem cache (Parquet):**
- Cache raw GraphQL responses as Parquet files on the local filesystem, keyed by query hash + variables hash
- TTL-based expiry: static data (zones, encounters, classes) cached for 7 days; reports/rankings cached for 24 hours
- Eliminates redundant API calls during development and re-runs
- Parquet format enables direct querying with DuckDB during development and doubles as the export format for mart tables
- Mart Parquet exports are uploaded to **Vercel Blob Storage** (free tier: 5 GB, 100 GB transfer/month) for DuckDB-WASM browser queries

**Layer 2 — Incremental ingestion state:**
- Track `last_fetched_timestamp` per guild/zone in a state table (PostgreSQL)
- Only request reports newer than the last successful fetch
- Store report codes already processed to avoid re-fetching

**Layer 3 — Request budgeting:**
- Before each API call, check remaining points via `rateLimitData` query (append to any request at zero extra cost)
- If budget < 10% remaining, pause and wait for reset
- Log points consumed per query type to optimize expensive queries
- Prioritize: rankings (cheap, high value) over fight details (expensive, lower priority)

**Layer 4 — Batch optimization:**
- Combine multiple small queries into fewer large queries where the schema allows
- Use GraphQL query batching to reduce HTTP overhead
- Fetch character rankings in bulk per encounter rather than per-character

### Scope Decision (Assumed)

Focus on **raid logs** (boss kills, damage/healing parses, rankings) as the primary dataset. This gives the richest Data Vault model with clear business entities:
- Players (characters)
- Guilds
- Encounters (bosses)
- Reports (raid sessions)
- Parses (performance records)

**Explicitly excluded from Phase 1:** The `Events` entity (granular combat events — damage, healing, casts). This is where data volume explodes (hundreds of thousands of rows per fight). We use aggregate data from Rankings and fight summaries instead. Events can be added in a future phase if storage permits.

Mythic+ can be added as a second phase. PvP data is limited on Warcraft Logs.

---

## 3. automate_dv on PostgreSQL

### Decision: Use automate_dv directly

automate_dv v0.11.5+ natively supports PostgreSQL via dbt's `adapter.dispatch()` pattern. All core vault macros (hub, link, sat, eff_sat, ma_sat, pit, bridge, t_link, xts, ref_table) have `postgres__` dispatch targets.

**PostgreSQL-specific adaptations handled by automate_dv:**
- `QUALIFY ROW_NUMBER()` → `DISTINCT ON (...)` for row deduplication
- Binary hash encoding via `ENCODE()`/`DECODE()`
- No MERGE — uses CTE + LEFT JOIN anti-join insert-only pattern (Postgres-compatible)
- Timestamp handling adapted for Postgres types

**Setup:** Add to `packages.yml` and configure `vars` in `dbt_project.yml` (hash algorithm, null placeholder, max datetime).

Sources:
- [automate_dv GitHub](https://github.com/Datavault-UK/automate-dv)
- [automate_dv website](https://automate-dv.com/what-is-automate-dv/)

---

## 5. Database Architecture

### Decision: PostgreSQL on Neon

| Aspect | Detail |
|--------|--------|
| Provider | Neon (free tier) |
| Storage | 0.5 GB (tight — scope to 1-2 guilds, exclude raw Events entity. Budget $19/mo for Neon Launch if needed) |
| Compute | 100 CU-hours/month, auto-suspend after 5 min |
| Branching | Supported (great for dbt dev branches) |
| Connection | Serverless driver for Next.js edge functions |
| Cold start | ~1-3s on first query after suspend |

Sources: [Neon pricing](https://neon.com/pricing), [Neon plans](https://neon.com/docs/introduction/plans)

### Optional: DuckDB-WASM for Ad-Hoc Exploration

Export mart tables as Parquet files to Vercel Blob Storage and expose a DuckDB-WASM "explore" page where users can run ad-hoc SQL queries in the browser. This offloads analytical queries from the backend.

**Trade-off:** DuckDB-WASM queries return raw SQL results — no TypeScript type safety, no server-side validation, no Drizzle ORM integration. This makes it unsuitable as the primary data path. The main frontend should use PostgreSQL via Next.js Server Actions + Drizzle ORM for full TypeScript control.

**Architecture:**
```
dbt (PostgreSQL) → export marts as Parquet → upload to Vercel Blob Storage → DuckDB-WASM on "explore" page
```

This is a stretch goal. The primary read path is PostgreSQL + Drizzle ORM + Next.js Server Actions.

---

## 6. Data Model: Data Vault 2.0

### Raw Vault Layer

**Hubs (business keys):**
- `hub_player` — player characters (name + server + region)
- `hub_guild` — guilds (name + server + region)
- `hub_encounter` — boss encounters (encounter_id + difficulty — difficulty is part of the business key since Normal vs Mythic are fundamentally different content)
- `hub_zone` — raid/dungeon zones (zone_id)
- `hub_report` — combat log reports (report_code)

**Removed from original draft:** `hub_ability`, `hub_class`, `hub_spec` — these are static reference data with no temporal changes. Modeled as dbt seeds (`seeds/class_spec_mapping.csv`) instead.

**Links (relationships):**
- `link_player_guild` — player belongs to guild
- `link_report_encounter` — report contains encounter (fight)
- `link_player_encounter` — player participates in encounter (includes spec_id as dependent child key — spec varies per fight)
- `link_player_report` — player appears in report (enables "all reports for a player" queries)

**Removed:** `link_encounter_zone` (degenerate — encounter always belongs to one zone; zone_id stored on hub_encounter). `link_player_class_spec` (class is immutable on player; spec varies per fight, moved to link_player_encounter).

**Satellites (descriptive attributes):**
- `sat_player_details` — player name, server, region, class_id (immutable), item level
- `sat_guild_details` — guild name, server, faction
- `sat_encounter_details` — encounter name, kill/wipe, fight duration
- `sat_report_details` — report title, owner, start/end time, zone_id
- `sat_player_encounter_performance` — DPS/HPS, parse percentile, duration, deaths, ilvl, spec_id
- `sat_zone_details` — zone name, expansion, type (raid/dungeon)

**Satellite deduplication:** Insert-only semantics — only insert if hashdiff differs from latest record for that hash key. Marts use `ROW_NUMBER() OVER (PARTITION BY hash_key ORDER BY load_date DESC) = 1` for current state.

### Business Vault / Marts Layer

- `mart_player_rankings` — aggregated player performance across encounters
- `mart_guild_progression` — guild boss kill history and progression
- `mart_encounter_statistics` — average DPS/HPS per encounter, per spec
- `mart_class_performance` — class/spec performance comparison by encounter
- `mart_recent_reports` — latest reports with summary stats

### dbt Project Structure

```
dbt_project/
├── dbt_project.yml
├── packages.yml
├── macros/                       # Project-specific macros only (automate_dv handles vault patterns)
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── stg_wcl_reports.sql
│   │   ├── stg_wcl_fights.sql
│   │   ├── stg_wcl_players.sql
│   │   ├── stg_wcl_rankings.sql
│   │   └── stg_wcl_guilds.sql
│   ├── raw_vault/
│   │   ├── hubs/
│   │   ├── links/
│   │   └── satellites/
│   └── marts/
│       ├── mart_player_rankings.sql
│       ├── mart_guild_progression.sql
│       ├── mart_encounter_statistics.sql
│       └── mart_class_performance.sql
├── tests/
│   ├── generic/
│   │   └── test_hub_integrity.sql
│   └── staging/
└── seeds/
    └── class_spec_mapping.csv
```

---

## 7. Orchestration: Apache Airflow

### Strategy: Astro CLI (local, free) + Optional VPS for Live Demo

Astronomer has **no free tier** (starts at $0.35/hr after 14-day trial). Strategy:

- **Development:** Astro CLI (`astro dev start`) runs Airflow locally via Docker for $0
- **Live demo (optional):** Self-host on a $4-6/mo VPS (Hetzner/DigitalOcean) with Docker Compose + LocalExecutor
- **CI:** GitHub Actions validates DAGs on push

Sources: [Astronomer pricing](https://www.astronomer.io/pricing/)

### DAG Design

**DAG 1: `wow_datavault_ingest`** (scheduled daily or on-demand)
```
fetch_new_reports → fetch_report_details → fetch_fights → fetch_player_data → fetch_rankings
```

**DAG 2: `wow_datavault_transform`** (triggered after ingestion)
```
dbt_staging → dbt_raw_vault → dbt_marts → dbt_test → export_parquet (optional)
```

**DAG 3: `wow_datavault_full_pipeline`** (orchestrator)
```
ingest_task_group → transform_task_group → notify_complete
```

### Airflow REST API for Monitoring

The Airflow Stable REST API (`/api/v1/`) exposes:
- `GET /dags/{dag_id}/dagRuns` — list runs by state
- `GET /dags/{dag_id}/dagRuns/{run_id}/taskInstances` — task states
- `GET /dags/{dag_id}/details` — DAG structure (task dependencies)

This is how the real-time monitoring frontend gets its data.

---

## 8. Frontend: Visualization Portal

### Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15 (App Router) |
| UI Library | React 19 |
| Language | TypeScript (strict mode) |
| Styling | Tailwind CSS v4 |
| Components | shadcn/ui |
| Tables | TanStack Table v8 (via shadcn/ui DataTable) |
| Auth | Auth.js v5 (Credentials provider) |
| ORM | Drizzle ORM |
| State | Zustand (for pipeline monitoring state) |
| Data Fetching | TanStack Query |
| Hosting | Vercel (Hobby/free tier) |

### Pages

1. **Login** (`/login`) — email/password auth via Auth.js v5
2. **Dashboard** (`/dashboard`) — overview cards (latest reports, active players, guild progression)
3. **Player Rankings** (`/rankings`) — dynamic table with filters: encounter, class, spec, difficulty, metric (DPS/HPS)
4. **Guild Progression** (`/guilds`) — guild boss kill timeline, progression tracking
5. **Encounter Stats** (`/encounters`) — per-boss statistics, class/spec breakdown
6. **Pipeline Monitor** (`/pipelines`) — real-time animated DAG visualization (the hero feature)
7. **Pipeline History** (`/pipelines/history`) — past runs, timing, success/failure

### Dynamic Tables

Using TanStack Table + shadcn/ui DataTable:
- Server-side filtering via URL search params (Next.js Server Components)
- Column sorting, multi-column filtering, pagination
- Filters: dropdown for class/spec/encounter, range for DPS/HPS, date range
- Export to CSV

### Vercel Free Tier Limits

| Resource | Limit |
|----------|-------|
| Bandwidth | 100 GB/month |
| Serverless execution | 100 GB-hours/month |
| Function timeout | Up to 60s |
| Builds | 6,000 minutes/month |
| Preview deployments | Unlimited |

Sufficient for this project's scale.

---

## 9. Real-Time Pipeline Monitoring (Hero Feature)

### Architecture (Revised after adversarial review)

**Primary mode: Demo/Replay** — pre-recorded pipeline runs replayed with animations. No backend infrastructure required.

**Optional live mode (Phase 2, requires VPS):** Only when the developer is running the pipeline locally or on a VPS with Airflow exposed.

```
PRIMARY (Demo):   [pipeline-replay.json] ──requestAnimationFrame──→ [React Frontend]
OPTIONAL (Live):  [Airflow API] ──poll──→ [FastAPI on VPS] ──WebSocket──→ [React Frontend]
```

The `/pipelines` page defaults to Demo mode. Live mode is hidden behind an env var (`NEXT_PUBLIC_PIPELINE_LIVE_URL`). If unset, live toggle is not shown.

### Key Technologies

- **React Flow** — node-based DAG visualization with animated edges
- **Framer Motion** — node status transitions (pulse when running, glow on success, shake on failure)
- **dagre** — automatic DAG layout algorithm
- **FastAPI (Python)** — lightweight WebSocket relay (Phase 2 only, for live mode on VPS)

### Visual States

| State | Visual |
|-------|--------|
| `queued` | Gray node, no animation |
| `running` | Blue node, pulsing glow, animated edge from parent |
| `success` | Green node, brief ripple effect |
| `failed` | Red node, shake animation |
| `skipped` | Dim/transparent node |

### Demo/Replay Mode (Critical for Portfolio)

Pre-record pipeline runs as a JSON event sequence and replay them on loop. This way visitors see the animated pipeline without needing live infrastructure.

```json
[
  {"t": 0, "task": "fetch_reports", "state": "running"},
  {"t": 3200, "task": "fetch_reports", "state": "success", "duration": 3.2},
  {"t": 3300, "task": "fetch_fights", "state": "running"},
  ...
]
```

The frontend plays this back with `requestAnimationFrame`, creating the full animated experience. A toggle switches between "Live" and "Demo" mode.

### Feasibility Assessment

**Highly feasible.** Complexity breakdown:
- Basic colored nodes updating state: 1-2 days
- Smooth animations with Framer Motion: 2-3 days
- WebSocket relay + Airflow integration: 2-3 days
- dbt log parsing + streaming: 1-2 days
- Demo/replay mode: 1-2 days
- Polish (dark mode, glow effects, edge animations): 2-3 days

---

## 10. Authentication

### Auth.js v5 with Credentials Provider

Kept minimal — the data is public WoW logs, so auth is lightweight.

- Email/password login via Auth.js v5 Credentials provider
- JWT sessions (stateless, works well with Vercel serverless)
- Next.js middleware for route protection
- Password hashing with bcrypt
- Pre-seeded demo account (`demo@example.com` / `demo123`) for quick access
- Simple role: `viewer` (all visitors get the same access)
- User table stored in PostgreSQL via Drizzle ORM

**Alternative if auth becomes a time sink:** Fall back to a shared demo password via env var (`DEMO_PASSWORD`), no user table needed. This is the escape hatch.

---

## 11. Testing Strategy (Deployment Gate)

### Frontend Tests

| Type | Tool | What |
|------|------|------|
| Unit | Vitest | Component logic, utility functions, hooks |
| Component | Vitest + Testing Library | UI component rendering, user interactions |
| E2E | Playwright | Full user flows (login, navigate, filter tables) |
| Type check | `tsc --noEmit` | TypeScript compilation |
| Lint | ESLint | Code quality |

### dbt Tests

| Type | What |
|------|------|
| Schema tests | `unique`, `not_null` on all hub PKs and satellite hashdiffs |
| Relationship tests | `relationships` between links and hubs |
| Custom tests | Hub load integrity, satellite deduplication, hashdiff correctness |
| Integration tests | Seed data → run automate_dv models → assert expected vault output |
| SQL linting | `sqlfluff lint` on generated SQL from `dbt compile` |
| DAG validation | `dag_bag.import_errors` check, task dependency validation |
| Source freshness | `dbt source freshness` for raw tables |
| API contract tests | Periodic canary test against live WCL API to detect schema changes |

### Python Tests (Ingestion)

| Type | Tool | What |
|------|------|------|
| Unit | pytest | API response parsing, rate limit logic, hash generation |
| Integration | pytest + responses/VCR | API call mocking, end-to-end ingestion flow |

### CI Pipeline (GitHub Actions)

```yaml
on:
  pull_request:
    branches: [main]

jobs:
  lint-and-typecheck:
    - eslint .
    - tsc --noEmit

  frontend-tests:
    - vitest run

  dbt-tests:
    - dbt build --select +marts --target ci  # runs models + tests
    
  python-tests:
    - pytest ingestion/tests/

  # All jobs must pass before merge is allowed
```

Branch protection: main requires all CI checks to pass.

---

## 12. CI/CD & Deployment

### GitHub Actions + Vercel

- **Vercel auto-deploys** on push (production on `main`, previews on feature branches)
- **GitHub Actions** adds quality gates: lint, type-check, tests, dbt validation
- **Branch protection** on `main`: require PR, require passing CI

### Branch Strategy

```
main (production) ← feature/<name> (preview deployments)
                  ← fix/<name>
```

### Deployment Flow

1. Developer pushes to `feature/xyz`
2. GitHub Actions runs: lint → type-check → unit tests → dbt tests → python tests
3. Vercel creates preview deployment with unique URL
4. PR review + CI must pass
5. Merge to `main` → Vercel deploys to production

---

## 13. Project Folder Structure

```
wow-datavault-dashboard/
├── CLAUDE.md
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Lint, typecheck, test
│       └── dbt-ci.yml                # dbt build + test
├── frontend/                         # Next.js 15 app
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   ├── drizzle.config.ts
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── login/
│   │   │   ├── dashboard/
│   │   │   ├── rankings/
│   │   │   ├── guilds/
│   │   │   ├── encounters/
│   │   │   └── pipelines/
│   │   ├── components/
│   │   │   ├── ui/                   # shadcn/ui components
│   │   │   ├── tables/               # DataTable with filters
│   │   │   └── pipeline/             # React Flow DAG components
│   │   ├── lib/
│   │   │   ├── auth.ts               # Auth.js config
│   │   │   ├── db.ts                 # Drizzle client
│   │   │   └── pipeline-events.ts    # WebSocket client
│   │   └── hooks/
│   │       └── use-pipeline-status.ts
│   ├── tests/
│   │   ├── unit/
│   │   ├── components/
│   │   └── e2e/
│   └── public/
│       └── demo/
│           └── pipeline-replay.json  # Pre-recorded pipeline run
├── ingestion/                        # Python ingestion scripts
│   ├── pyproject.toml
│   ├── src/
│   │   ├── __init__.py
│   │   ├── wcl_client.py             # Warcraft Logs GraphQL client
│   │   ├── extractors/
│   │   │   ├── reports.py
│   │   │   ├── fights.py
│   │   │   ├── players.py
│   │   │   └── rankings.py
│   │   └── loaders/
│   │       └── postgres_loader.py
│   └── tests/
│       ├── test_wcl_client.py
│       └── test_extractors.py
├── dbt_project/                      # dbt + Data Vault
│   ├── dbt_project.yml
│   ├── packages.yml
│   ├── profiles.yml.example    # Template only — actual profiles.yml uses env vars, NOT checked in
│   ├── macros/
│   │   └── data_vault/
│   │       ├── hub.sql
│   │       ├── link.sql
│   │       ├── satellite.sql
│   │       ├── stage.sql
│   │       └── hash.sql
│   ├── models/
│   │   ├── staging/
│   │   ├── raw_vault/
│   │   │   ├── hubs/
│   │   │   ├── links/
│   │   │   └── satellites/
│   │   └── marts/
│   ├── tests/
│   └── seeds/
├── airflow/                          # Airflow DAGs (Astro CLI project)
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── packages.txt
│   ├── dags/
│   │   ├── wow_ingest.py
│   │   ├── wow_transform.py
│   │   └── wow_full_pipeline.py
│   └── plugins/
├── pipeline-relay/                   # Phase 2: FastAPI WebSocket server (only for live mode)
│   ├── pyproject.toml
│   ├── src/
│   │   ├── main.py                   # FastAPI app with WebSocket
│   │   ├── airflow_poller.py         # Polls Airflow REST API
│   │   └── dbt_streamer.py           # Parses dbt JSON logs
│   └── tests/
├── .gitignore
├── .env.example                      # Template for all env vars (Neon, WCL API, Auth.js secret)
└── docker-compose.yml                # Local dev: Airflow only (Neon is remote)
```

---

## 14. Cost Summary

| Component | Cost |
|-----------|------|
| Warcraft Logs API | Free |
| PostgreSQL (Neon) | Free (0.5 GB, 100 CU-hours) |
| Airflow (Astro CLI local) | Free |
| Airflow (VPS for live demo) | $4-6/month (optional) |
| Frontend (Vercel Hobby) | Free |
| Vercel Blob Storage | Free (5 GB storage, 100 GB transfer) |
| Pipeline relay (Phase 2, Railway) | Free tier ($5 credit) |
| GitHub Actions | Free (2,000 min/month) |
| Domain name | ~$10/year (optional) |
| **Total (minimum)** | **$0/month** |
| **Total (with live demo)** | **~$5/month** |

---

## 15. Runtime & Deployment Story

### Where does each component run?

| Component | Local Dev | Production |
|-----------|-----------|------------|
| **Frontend (Next.js)** | `npm run dev` | Vercel (auto-deploy on merge to main) |
| **PostgreSQL** | Neon (remote, even in dev — use Neon branching for dev/prod separation) | Neon (same instance, `main` branch) |
| **Ingestion (Python)** | Run manually: `python -m ingestion.src.extractors.reports` | Airflow `PythonOperator` imports from `ingestion.src` (shared Python env in Docker) |
| **dbt** | `dbt run --target dev` (Neon dev branch) | Airflow `BashOperator("dbt run --target prod")` in Docker |
| **Airflow** | `astro dev start` (Docker, local only) | Optional: Docker Compose on VPS ($4-6/mo) |
| **Pipeline Relay** | Not needed (demo mode) | Phase 2: Railway free tier, only if VPS Airflow exists |

### Secrets Management

- **Local:** `.env` file (gitignored) with all credentials
- **Vercel:** Environment variables in dashboard (Production + Preview)
- **GitHub Actions CI:** Repository secrets for `DATABASE_URL`, `WCL_CLIENT_ID`, `WCL_CLIENT_SECRET`
- **Airflow (local):** `.env` file mounted in Docker
- **dbt:** `profiles.yml` uses `env_var('DBT_POSTGRES_HOST')` etc. — NEVER hardcoded credentials. `profiles.yml` itself is gitignored; only `profiles.yml.example` is committed.

### Database Initialization

1. First `dbt run` creates all schemas/tables on Neon automatically (dbt handles DDL)
2. Drizzle ORM migrations handle frontend tables (users, sessions)
3. `dbt seed` loads static reference data (classes, specs)
4. First ingestion run populates raw tables

---

## 16. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Neon 0.5 GB storage fills up | Can't ingest more data | Scope to top guilds/encounters; prune old raw data |
| Neon cold starts (1-3s) | Slow first page load | Implement loading states; consider DuckDB-WASM for read path |
| WCL API rate limits | Slow ingestion | Smart caching, incremental loads, off-peak scheduling |
| automate_dv edge cases on Postgres | Incorrect data model | Comprehensive dbt tests on every hub/link/satellite |
| Airflow not accessible online | Can't show live pipelines | Demo/replay mode as primary showcase |
| Vercel function timeout (60s) | Complex queries fail | Optimize SQL, add indexes, paginate results |

---

## 17. Phases (Revised — Realistic Timeline)

**The plan below targets a working MVP in 6-8 weeks by cutting scope strategically.**

### Phase 1: Foundation + README (Week 1-2)
- README with architecture diagram
- Project scaffolding (Next.js, dbt, Python)
- PostgreSQL setup on Neon (dev branch)
- WCL API client + OAuth token management
- Basic ingestion scripts (reports, fights, players, rankings)
- automate_dv package setup and configuration for PostgreSQL

### Phase 2: Data Model (Week 3-4)
- Full staging models
- Raw vault (all hubs, links, satellites)
- Mart models (player_rankings, encounter_statistics, class_performance)
- Comprehensive dbt tests (schema + integration tests with seed data)
- Seed data for demo environment

### Phase 3: Frontend + Auth (Week 4-5)
- Auth (login page with demo account)
- Dashboard overview
- Dynamic tables with filters (rankings, encounters)
- shadcn/ui + Tailwind styling, dark mode

### Phase 4: Pipeline Visualization (Week 5-7)
- React Flow DAG visualization with dagre layout
- Framer Motion animations (state transitions, progressive illumination)
- Demo/replay mode (pre-recorded JSON, defaults on `/pipelines`)
- Pipeline history page

### Phase 5: CI/CD + Polish (Week 7-8)
- GitHub Actions CI (lint, type-check, vitest, dbt tests, pytest)
- Vercel deployment + branch protection
- Playwright E2E tests
- Responsive design
- Update README with GIF/video of pipeline animation

### Phase 6 (Stretch Goals — Post-MVP)
- Airflow DAGs (local via Astro CLI, screenshots for README)
- Live pipeline monitoring (FastAPI relay on VPS)
- DuckDB-WASM frontend for mart queries
- Guild progression page
- Data quality layer

---

## 18. Review Summary

Key changes from design review:
1. **Killed the live WebSocket relay as primary mode** — demo/replay is now the default
2. **Fixed data model:** difficulty in encounter business key, removed degenerate links, added link_player_report, moved spec to link_player_encounter
3. **Revised macro estimate** from 2-3 days to 5-7 days
4. **Added deployment story** (Section 15) — where each component runs, secrets management, DB initialization
5. **Added integration tests** and DAG validation tests
6. **Fixed profiles.yml** — gitignored, uses env vars
7. **Revised timeline** to 6-8 weeks with realistic estimates and strategic scope cuts
8. **Added README as Phase 1** priority
9. **Flagged 0.5 GB Neon storage** as tight — scope to 1-2 guilds, exclude Events entity

---

## 19. Assumptions Made (User Was Away)

1. **Raid logs are the primary focus** — not Mythic+ or PvP (can add later)
2. **"React 19 for backend" means Next.js 15 App Router** with Server Components/Actions
3. **Using automate_dv directly** — native Postgres support confirmed in v0.11.5
4. **PostgreSQL over DuckDB** for the primary database — automate_dv-like patterns work natively
5. **Auth.js v5 over Clerk** — keeps it free, demonstrates auth skills
6. **Demo/replay mode** for pipeline monitoring — visitors don't need live infra
7. **Astro CLI for local Airflow** — no Astronomer cloud (too expensive)
8. **Drizzle ORM over Prisma** — lighter, better TypeScript inference, better for serverless

---

## Sources

- [Warcraft Logs API v2 Documentation](https://www.warcraftlogs.com/v2-api-docs/warcraft/)
- [Warcraft Logs API Rate Limits](https://www.warcraftlogs.com/v2-api-docs/warcraft/ratelimitdata.doc.html)
- [automate_dv GitHub](https://github.com/Datavault-UK/automate-dv)
- [automate_dv Supported Platforms](https://automate-dv.com/what-is-automate-dv/)
- [Neon Free Tier](https://neon.com/pricing)
- [Astronomer Pricing](https://www.astronomer.io/pricing/)
- [Neon Serverless Postgres Pricing 2026](https://vela.simplyblock.io/articles/neon-serverless-postgres-pricing-2026/)
