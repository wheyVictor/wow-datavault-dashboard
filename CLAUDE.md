# CLAUDE.md

## Project: WoW Data Vault Dashboard

Portfolio project — Warcraft Logs data pipeline with Data Vault modeling, dbt, Airflow, and React visualization portal.

## Critical Rules

- **This is a production repository.** Do not run destructive commands (rm -rf, git reset --hard, DROP TABLE, etc.) without explicit user approval.
- **Never force-push to main.** Always use feature branches and PRs.
- **Do not commit secrets, API keys, or credentials.** Use environment variables and .env files (which must be in .gitignore).
- **Tests are a deployment gate.** All changes to main must pass CI. No deployment without passing tests. This is non-negotiable.
- **Write tests for every feature.** Unit tests, integration tests, and dbt tests must accompany all code changes.

## Branch Strategy

- `main` — production branch, deploys to Vercel on merge
- Feature branches: `feature/<name>`
- Bugfix branches: `fix/<name>`
- Always create PRs, never commit directly to main.

## Tech Stack

- **Ingestion:** Python scripts
- **Database:** TBD (PostgreSQL or DuckDB — under evaluation)
- **Transformation:** dbt + automate_dv (Data Vault 2.0)
- **Orchestration:** Apache Airflow
- **Frontend:** Next.js + React 19 + TypeScript + Tailwind CSS
- **Auth:** TBD
- **Deployment:** Vercel (frontend), TBD (backend/data infra)
- **CI/CD:** GitHub Actions (tests gate deployment — no merge to main without green CI)

## Testing Strategy

- **Frontend:** Vitest for unit tests, Playwright for E2E tests
- **dbt:** dbt tests (unique, not_null, relationships, custom) on all vault and mart models
- **Ingestion:** pytest for Python ingestion scripts
- **GitHub Actions CI:** Lint + type-check + unit tests + dbt tests must all pass before merge is allowed
- **Branch protection:** main branch requires passing CI status checks
