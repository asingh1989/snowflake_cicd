# Copilot instructions for this repo

This repository is a Snowflake CI/CD pipeline driven by GitHub Actions. Your goal as an AI coding agent is to help author Snowflake SQL objects and keep the deployment workflow reliable, idempotent, and traceable.

## Architecture and flow

- Source-of-truth SQL lives under `Snowflake/` with numbered folders that enforce dependency order:
  - `Snowflake/00_Schema/` → create schemas first
  - `Snowflake/01_Table/` → create tables next
  - `Snowflake/02_View/` → create views last
- On a push to `main`, the workflow `.github/workflows/prod_deploy.yml`:
  - Detects changed `.sql` files with `git diff` limited to `Snowflake/**/*.sql`
  - Sorts changed paths alphabetically (the numbered prefixes ensure correct dependency order)
  - Executes each file via SnowCLI: `snow sql -f <file> --temporary-connection`
  - Logs each attempt and status to a deployment history table
  - Fails the job if any file fails; success otherwise

## Branching and developer workflow

- Work on `DEV`; open a PR to `main` for production deploys. Avoid direct commits to `main`.
- Place new objects in the correct numbered subfolder; names should be deterministic so alphabetical sort produces a safe order.
- Keep scripts idempotent where possible (e.g., `CREATE ... IF NOT EXISTS`, `CREATE OR REPLACE`, or guards) to allow re-runs.

## Secrets and environment

- The workflow uses GitHub Actions secrets: `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, `SNOWFLAKE_ROLE`, `SNOWFLAKE_DATABASE`, `SNOWFLAKE_WAREHOUSE`, `SNOWFLAKE_PRIVATE_KEY_RAW`, `SNOWFLAKE_PRIVATE_KEY_PASSPHRASE`. Auth is JWT via SnowCLI.
- SnowCLI is set up with `snowflakedb/snowflake-cli-action@v1.5`.

## Deployment logging

- Current workflow logs with:
  - `INSERT INTO CICD_DEMO_DB.TECH.DEPLOYMENT_HISTORY (...) VALUES (...);`
- README shows a different example table: `CICD_PROD_DB.PUBLIC.DEPLOYMENT_HISTORY`. If you change logging, keep it consistent across README, setup SQL, and the workflow.

## Conventions and patterns

- Keep SQL files focused: one object per file is preferred. Example:
  - `Snowflake/00_Schema/TESTSCHEMA.sql` → `CREATE SCHEMA IF NOT EXISTS TESTSCHEMA;`
  - `Snowflake/01_Table/Customer.sql` → `CREATE OR REPLACE TABLE TESTSCHEMA.CUSTOMER (...);`
  - `Snowflake/02_View/TESTVIEW.sql` → `CREATE OR REPLACE VIEW TESTSCHEMA.TESTVIEW AS SELECT ...;`
- Use fully qualified names (database.schema.object) when needed, or rely on `USE DATABASE/SCHEMA` statements at the top of files if consistent.
- Quote handling: the workflow escapes single quotes in error messages via `sed "s/'/''/g"` before inserting into Snowflake.

## What to edit vs. avoid

- Do edit SQL under `Snowflake/**` and ensure safe re-execution.
- If adding new object types (e.g., stages, pipes, functions), create new numbered folders (`03_Stage/`, `04_Pipe/`, etc.) so alphabetical sort preserves dependency order.
- Avoid changing the workflow trigger or checkout depth unless you understand the `git diff` implications.

## Useful references in repo

- Workflow: `.github/workflows/prod_deploy.yml`
- Environment setup example(s): `SetupEnv/ENV_SETUP.sql` (contains setup commands; keep it aligned with README and workflow expectations)
- Big-picture overview: `README.md`

## Examples

- Adding a new table:
  - File: `Snowflake/01_Table/ORDERS.sql`
  - Content pattern: `CREATE OR REPLACE TABLE TESTSCHEMA.ORDERS (ORDER_ID NUMBER, ...);`
- Adding a dependent view:
  - File: `Snowflake/02_View/ORDERS_SUMMARY.sql`
  - Content pattern: `CREATE OR REPLACE VIEW TESTSCHEMA.ORDERS_SUMMARY AS SELECT ... FROM TESTSCHEMA.ORDERS;`

## Known edge cases

- If a file is renamed but not edited, it may not appear as changed depending on the diff; prefer explicit edits in PRs.
- If multiple objects in one file depend on prior creation, ensure proper `USE` statements or fully qualified names.
- Mismatch between logging table names (workflow vs. README) can cause logging failures—verify the target database/schema exists.

If anything above seems off (e.g., different target database/schema, additional folders, or alternate logging tables), tell me and I’ll update this doc to match your actual deployment setup.