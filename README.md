# Lakebase Skills

Reusable, shareable **skills** for [Databricks Lakebase](https://docs.databricks.com/aws/en/oltp/) — packaged how-tos and runnable agent skills that accelerate common Lakebase work. Organized by **play / pattern** so you can browse by what you're trying to do.

Each skill folder contains:
- a **`SKILL.md`** — an agent skill (frontmatter + instructions) you can drop into an AI coding assistant (Claude Code, Databricks Assistant / Genie Code, etc.)
- a **customer guide** (HTML) — a readable walkthrough of the approach
- a **`.zip`** — the packaged skill, ready to download and install

> Field-built enablement material. Accelerators, not authority — always validate output against the official [Databricks Lakebase documentation](https://docs.databricks.com/aws/en/oltp/) and your own tests.

## Categories

### migration
Moving code and workloads onto Lakebase.
- **[azure-sql-to-lakebase-procedure-migrator](migration/azure-sql-to-lakebase-procedure-migrator/)** — analyze, classify, and migrate Azure SQL / SQL Server T-SQL stored procedures, functions, and triggers to Lakebase PostgreSQL (PL/pgSQL), the application layer, or Databricks SQL / Lakeflow — with safe validation and explicit blocker reporting.

### data-api-access
*(planned)* Skills for the Lakebase Data API, OBO, and role/permission patterns.

### feature-store
*(planned)* Skills for the Lakebase-powered online feature store + model serving.

### genie
*(planned)* Skills for Genie-assisted workflows on Lakebase.

## Using a skill

1. Open the skill folder and read its `README.md` + customer guide.
2. Download the `.zip` (or copy `SKILL.md`) and add it to your AI assistant's skills.
3. Run it on a small batch first; validate with deterministic tests before scaling.

## License

[Apache License 2.0](LICENSE).

## Contributing

Each new skill is its own folder under the matching category, with a `SKILL.md`, a customer guide, a `.zip`, and a short `README.md`. Open a PR.
