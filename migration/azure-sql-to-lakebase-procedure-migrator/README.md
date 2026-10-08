# Azure SQL → Lakebase Stored-Procedure Migrator

**Category:** migration · **Source → target:** Azure SQL / SQL Server (T-SQL) → Databricks Lakebase (PostgreSQL 17 / PL/pgSQL)

An agent skill that analyzes, classifies, and migrates Azure SQL / SQL Server **stored procedures, functions, triggers**, and procedure-driven application logic to Lakebase — with safe validation and explicit blocker reporting.

## The one principle: classify before you convert

Not all T-SQL belongs in Lakebase. Lakebase is an **operational OLTP** engine, not a batch/analytics engine. The skill first decides where each piece of logic should go:

- **Request-path transactions, validation, row-level business rules** → Lakebase **PL/pgSQL**
- **Set-based transforms, aggregations, SCD, batch/reporting** → **Databricks SQL / Lakeflow / Spark** (the lakehouse)
- **Scheduler-driven ETL / orchestration** → **Lakeflow + Workflows**

Get the classification right and you convert far *less* code — and what remains fits Lakebase's job. Do **not** lift-and-shift every proc into Postgres.

## What's here

- **`SKILL.md`** — the agent skill (drop into Claude Code / Databricks Assistant / Genie Code). Analyzes → converts → validates, and reports blockers explicitly.
- **`genie-code-lakebase-customer-guide.html`** — a readable customer guide to the Genie Code approach for this migration.
- **`azure-sql-to-lakebase-procedure-migrator.zip`** — the packaged skill, ready to download/install.

## How to use

1. Add `SKILL.md` to your AI assistant (or unzip the package).
2. Point it at the T-SQL you want to migrate.
3. Work in **small batches** (2–5 procs, mixing simple + hard) and validate each with deterministic tests before scaling. Expect roughly **30–50%** of procedural code (cursors, dynamic SQL, TRY/CATCH, temp tables, collation) to need human review.

## Reference

- Convert proprietary code to open ANSI SQL with Genie Code (Databricks blog): https://www.databricks.com/blog/convert-proprietary-code-open-ansi-sql-genie-code
- Lakebase overview: https://docs.databricks.com/aws/en/oltp/

> Accelerator, not final authority. Validate every converted routine against tests and the official docs.
