# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Companion repo for Chapter 7 ("Implementing Data Contracts") of O'Reilly's *Data Contracts*. It is a teaching sandbox: code favors explicitness over abstraction on purpose, and README.md is the tutorial (it embeds code snippets from these files — keep them in sync when changing referenced code). The tutorial scenario has learners break and fix the `object_images` contract via the Alembic migration, so `main` is the "passing" baseline.

## Environment and commands

Everything runs in Docker against Postgres 17 (`postgres:postgres@postgres:5432/postgres`). Python 3.12; deps in `requirements.txt` (no pinned versions, no linter/formatter configured).

```bash
docker compose up                                   # postgres + etl (alembic upgrade head, seed_db.py) + devcontainer
docker compose down -v --remove-orphans && docker compose up   # full reset (deletes the volume)
docker compose -f docker-compose.test.yml up --abort-on-container-exit   # CI-style: migrate then run tests -> test_output.log
```

Inside the devcontainer, from the repo root (`/workspace`) — imports are absolute (`data_contract_components.…`) and the tests use relative paths, so run from root:

```bash
python -m unittest data_contract_components/prevention/test_data_contract_violations.py -v
python -m unittest data_contract_components.prevention.test_data_contract_violations.TestContractViolations.test_data_contracts_against_data_catalog
```

Migrations run from `data_contract_components/data_assets` (where `alembic.ini` lives):

```bash
alembic upgrade head
alembic downgrade base
python3 seed_db.py      # reseeds from db_migrations/raw_data/objects.json
```

`alembic.ini` and `seed_db.py` hardcode host `postgres` (only resolvable inside the compose network). `_query_postgres_helper.PostgresDB` probes `postgres:5432` and falls back to `localhost:5432`, so detection/tests can also run from the host while compose is up.

## Architecture

Four components under `data_contract_components/`, matching the README's A–D:

- **A. `data_assets/`** — the source of truth for schema is the single Alembic migration `db_migrations/versions/00e9b3375a5f_create_met_museum_seed_tables.py` (Met Museum objects normalized into `object_*` tables, with column comments). `seed_db.py` loads `raw_data/objects.json`.
- **B. `contract_definition/`** — one JSON spec per contracted table (`*_contract_spec.json`; file stem = contract name). `schema.table_catalog/table_schema/table_name` identify the table; `schema.properties.<column>.constraints` holds `primaryKey`, `data_type`, `numeric_precision`, `character_maximum_length`, `datetime_precision`, `is_nullable`, `is_updatable`; array columns add `array_element` with element-type fields. Constraints omitted from a spec are not checked.
- **C. `detection/`** — only metadata is compared, never row data:
  - `_get_data_catalog.get_data_catalog()` builds a "data catalog" DataFrame by querying Postgres `information_schema` (columns + element_types for arrays + constraint usage for PKs), `public` schema only.
  - `ContractCoverageDetector` left-joins specs to the catalog on table identity → contracted tables missing from the DB.
  - `ContractViolationDetector.transform_contract_specs_to_catalog_format()` flattens specs into catalog column shape (e.g. `is_nullable` bool → `"YES"/"NO"`, `primaryKey` → `"PRIMARY KEY"`), then joins on table + column; reports missing columns and per-field mismatches. Adding a new checked constraint requires updating the catalog SQL, this transform, and `constraint_fields`.
- **D. `prevention/`** — `unittest` tests wrapping both detectors; the contract spec is the source of truth.

CI (`.github/workflows/docker-image.yml`): on push/PR to `main`, one job runs `docker-compose.yml`, another runs `docker-compose.test.yml`; on PR test failure it posts `test_output.log` as a PR comment.

`run_sql_queries_here.ipynb` is a scratch notebook for querying the seeded DB.
