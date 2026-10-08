---
name: azure-sql-to-lakebase-procedure-migrator
description: Analyze and migrate Azure SQL Server T-SQL stored procedures to Lakebase PostgreSQL routines, triggers, or application workflows with safe validation and explicit blocker reporting.
---

# Azure SQL → Lakebase Stored-Procedure Migrator

Use this skill when the user asks to migrate Azure SQL / SQL Server stored procedures, functions, triggers, or procedure-driven application logic to Lakebase.

This is a heterogeneous migration. Do not treat it as a text-only rewrite. First determine whether the logic belongs in a Lakebase PostgreSQL routine, a trigger, the application layer, or a Databricks SQL/Lakeflow workflow.

## Safety and operating principles

- Never overwrite Azure SQL source files or production objects.
- Work against a development Lakebase project or branch. Never execute generated code against production without explicit approval.
- Analyze before converting. Ask for missing target project/branch, schema mappings, dependent DDL, or test data.
- Do not invent tables, columns, permissions, secrets, connection details, sequences, or business rules.
- Preserve the external contract unless the report explicitly documents a required breaking change: routine name, parameters, parameter direction, return shape, side effects, transaction behavior, error behavior, and idempotency.
- Mark uncertain behavior as `NEEDS_REVIEW`; do not silently choose a plausible mapping.
- Do not use `SQL SECURITY DEFINER`, elevated roles, dynamic SQL, or destructive DDL without explaining and obtaining approval.
- Separate conversion from deployment. Show the diff and validation evidence before proposing deployment.

## Target decision tree

Classify every source procedure before generating the final target:

1. **Lakebase PostgreSQL function** — preferred for reusable calculations or routines that return a scalar, table, or result set to an OLTP application.
2. **Lakebase PostgreSQL procedure** — use for side-effecting transactional operations invoked with `CALL`, when the client contract supports procedure invocation and the routine does not need to behave like a query expression.
3. **Lakebase trigger/function pair** — use only when the source behavior is truly row-event driven and the write latency, recursion, failure behavior, and audit/storage impact are acceptable.
4. **Application-layer workflow** — use for external calls, orchestration, long-running work, retries across systems, file access, service-bus calls, SQL Agent behavior, or logic that requires capabilities unavailable in a managed Postgres service.
5. **Databricks SQL or Lakeflow workflow** — use when the routine is primarily analytical/ETL logic over lakehouse data rather than low-latency OLTP logic in Lakebase.

If the source procedure mixes multiple categories, split it into a small Lakebase transaction routine plus an explicit application or Lakeflow workflow and document the new boundary.

## Required inputs

Before conversion, collect as many as available:

- Azure SQL procedure/function/trigger definitions.
- DDL for every referenced table, view, type, sequence, and synonym.
- Calls from application code, jobs, triggers, or other routines.
- Expected parameters, result sets, output parameters, and error codes.
- Representative test fixtures and golden outputs.
- Target Lakebase project, branch, database, schema, and Postgres version.
- Security model: caller identity, owner identity, required roles, row-level restrictions, and audit requirements.
- Performance and concurrency expectations.

If dependencies are in the workspace, reference them explicitly with `@` or inspect them before translating names.

## Phase 1 — Inventory and risk analysis

For each routine, produce an inventory containing:

- Source name, schema, type, parameters, result sets, and side effects.
- Tables/views/functions/types/sequences referenced.
- Writes, transaction boundaries, locking hints, isolation assumptions, and retry behavior.
- Temporary tables, table variables, cursors, dynamic SQL, linked servers, external calls, CLR, file access, and session settings.
- Security context such as `EXECUTE AS`, ownership chaining, and permission dependencies.
- Complexity: `LOW`, `MEDIUM`, `HIGH`, or `BLOCKED`.
- Target classification: Lakebase function, Lakebase procedure, trigger, application workflow, or Lakeflow/Databricks workflow.
- Exact blockers and required decisions.

Create a dependency-aware migration order. Migrate schemas, types, sequences, and referenced functions before dependent routines. Keep routines and their contract tests together.

Do not convert until the analysis summary is complete and the user approves the proposed target classification.

## Phase 2 — Contract-first conversion

For every approved routine:

1. Write the target contract before the body:
   - target schema and routine name;
   - `FUNCTION` or `PROCEDURE` choice;
   - `IN`, `OUT`, or `INOUT` parameters;
   - return type, `RETURNS TABLE`, or explicit result contract;
   - volatility and data-access characteristics when known;
   - security model and required grants.
2. Convert the body to idiomatic PostgreSQL / PL/pgSQL supported by Lakebase.
3. Keep business rules readable. Prefer set-based SQL over cursors and row-by-row processing when semantics remain equivalent.
4. Preserve atomicity and failure behavior. If a boundary changes, record the difference and add a test for partial failure and rollback.
5. Write a migration note for every non-trivial transformation.
6. Create a separate compatibility wrapper when an application needs the old name or parameter contract during transition.

## High-risk T-SQL mappings

Use these as review prompts, not blind substitutions:

| Azure SQL / T-SQL | Lakebase treatment |
|---|---|
| `CREATE PROCEDURE` with a result set | Usually a PostgreSQL `FUNCTION` with `RETURNS TABLE` or a clearly defined composite result; verify the client call contract. |
| Side-effect-only procedure | PostgreSQL `PROCEDURE` with `CALL`, or a function only when the client and transaction semantics permit it. |
| `OUTPUT` parameters | Map to `OUT`/`INOUT` parameters or return a single structured row; preserve nullability and caller expectations. |
| `@@ROWCOUNT` | Use `GET DIAGNOSTICS affected = ROW_COUNT` immediately after the statement whose count is required. |
| `SCOPE_IDENTITY()` / `@@IDENTITY` | Prefer `INSERT ... RETURNING`; validate sequence ownership and concurrency behavior. Never substitute `MAX(id)`. |
| `IDENTITY` columns | Map to an explicitly reviewed identity/sequence strategy and test explicit inserts, rollback, and concurrent inserts. |
| `TRY...CATCH`, `THROW`, `RAISERROR` | Use PL/pgSQL `EXCEPTION` blocks and `RAISE`; preserve error categories and whether the caller can retry. |
| `ISNULL` | Usually `COALESCE`, but test type precedence and implicit casts. |
| `GETDATE()`, `SYSDATETIME()`, `GETUTCDATE()` | Map only after deciding whether the contract is local time, UTC, or transaction time. Test time zones and precision. |
| `TOP`, `OFFSET/FETCH` | Map to `LIMIT/OFFSET` only with an explicit deterministic `ORDER BY` where result ordering matters. |
| `[]` identifiers and `dbo` | Use PostgreSQL identifier rules and an explicit target schema mapping; do not preserve `dbo` blindly. |
| `#temp` / `##temp` tables | Use session-scoped temporary tables only when session lifetime and connection pooling are controlled; otherwise use a permanent staging table or set-based rewrite. |
| Table variables / TVPs | Consider temporary tables, composite types, JSONB, arrays, or a redesigned API contract; test cardinality and statistics assumptions. |
| `sp_executesql` / dynamic SQL | Use parameterized `EXECUTE ... USING` for values and safe identifier quoting for identifiers. Never concatenate untrusted input. |
| Cursors / `WHILE` loops | Prefer set-based SQL; otherwise use a controlled `FOR ... IN SELECT` loop and test locking, ordering, and failure behavior. |
| `WITH (NOLOCK)` and SQL Server locking hints | Do not translate literally. Re-evaluate consistency, isolation, indexes, and concurrency under PostgreSQL MVCC. |
| `MERGE` | Re-express using a reviewed PostgreSQL-compatible pattern and test duplicate source keys, concurrent writers, and retry behavior. |
| Linked servers, `OPENQUERY`, CLR, file access, SQL Agent, Service Broker | Classify as application/Lakeflow orchestration or blocker; do not emulate with an invented database function. |
| `EXECUTE AS`, ownership chaining | Design an explicit Lakebase role/grant model. Use definer security only with a least-privilege owner, fixed object references, and security review. |
| Multiple result sets | Split into named routines, return a composite/JSON contract, or move orchestration to the application; document client changes. |
| Collations and case-insensitive comparisons | Identify the source collation and validate the Lakebase collation/operator behavior with representative Unicode and case variants. |

## Lakebase-specific checks

Before declaring a routine ready:

- Confirm every required PostgreSQL extension exists in the target Lakebase project; never assume an extension is available.
- Check temporary-table and session assumptions against connection pooling and scale-to-zero behavior.
- Reject designs requiring host operating-system access, direct local file access, Postgres superuser access, tablespaces, or unsupported replication features.
- Verify role ownership, `GRANT EXECUTE`, table privileges, and row-level security separately from functional correctness.
- Validate transaction scope, savepoints, locks, deadlocks, retry behavior, and idempotency under concurrent calls.
- Test identity/sequence behavior, timestamps/time zones, collations, nulls, empty inputs, duplicate keys, and boundary values.
- If the procedure feeds change history or downstream lakehouse processing, specify the supported Lakebase-to-lakehouse mechanism and validate its latency and data-type mapping separately.

## Validation workflow

Run validation only in a non-production Lakebase branch or approved test environment:

1. Validate target DDL and routine syntax.
2. Invoke the target routine with happy-path fixtures.
3. Compare source and target on equivalent fixtures:
   - result schema and row counts;
   - primary-key and uniqueness behavior;
   - aggregates/checksums and representative rows;
   - nulls, empty results, duplicates, and boundary values;
   - generated IDs and timestamps;
   - error codes/messages and rollback outcomes;
   - concurrent calls, locking, retries, and performance for critical paths.
4. For every discrepancy, classify it as expected platform behavior, conversion defect, missing fixture/dependency, or unresolved.
5. Repair targeted sections, rerun dependent tests, and preserve the evidence.
6. Do not label a routine production-ready unless functional, security, concurrency, and performance checks passed.

## Output contract

Write the following to the requested destination:

1. `<source_routine>.lakebase.sql` — target routine or split implementation.
2. `<source_routine>.migration-notes.md` — contract, mappings, assumptions, blockers, and client-impact notes.
3. `tests/<source_routine>.sql` — deterministic smoke, edge-case, error, rollback, and idempotency tests.
4. `migration_report.md` — summary for all routines, dependency order, status, validation evidence, and unresolved risks.
5. If the target is not a Lakebase routine, write a proposed application/Lakeflow contract instead of pretending the procedure was converted.

Use these statuses:

- `CONVERTED`: target routine created and tested successfully in the approved non-production environment.
- `NEEDS_REVIEW`: target runs or parses, but semantic, security, concurrency, or contract questions remain.
- `BLOCKED`: required capability, dependency, permission, or external behavior cannot be safely represented in Lakebase.
- `FAILED`: conversion or validation did not complete after targeted repair attempts.

## Prompt patterns

Initial analysis:

> Analyze the Azure SQL stored procedure at `<path>` for migration to Lakebase PostgreSQL. Do not modify source or run anything yet. Inspect dependencies and classify the target as a Lakebase function, procedure, trigger, application workflow, or Lakeflow workflow. Document the target contract, T-SQL-only behavior, security/transaction risks, blockers, and validation plan. Wait for approval.

Approved conversion:

> Convert the approved procedure to Lakebase PostgreSQL for project `<project>`, branch `<branch>`, database `<database>`, and schema `<schema>`. Preserve the caller contract where safe. Generate the `.lakebase.sql`, migration notes, tests, and report. Validate only in this non-production target, show diffs, and mark uncertain behavior `NEEDS_REVIEW`.

Targeted repair:

> Revalidate `<routine>` against the Azure SQL golden outputs. Investigate only the discrepancy in `<behavior>`. Do not change unrelated logic. Show the proposed patch, explain the semantic difference, update the test, and rerun the affected checks.

## Completion checklist

- [ ] Source definitions remain unchanged.
- [ ] Every dependency and target object mapping is recorded.
- [ ] Function/procedure/trigger/application classification is justified.
- [ ] Parameter and return contracts are explicit.
- [ ] Temporary objects, sequences, transactions, errors, security, and concurrency are addressed.
- [ ] No unsupported capability was silently emulated.
- [ ] Tests cover happy path, nulls, empty results, duplicates, boundaries, errors, rollback, retry, and idempotency.
- [ ] Validation ran against a non-production Lakebase target.
- [ ] The report distinguishes verified behavior from assumptions.
- [ ] No production deployment or destructive change occurred without approval.

## References

- [Lakebase Postgres](https://docs.databricks.com/aws/en/oltp/projects)
- [Lakebase Postgres compatibility and limitations](https://docs.databricks.com/aws/en/oltp/projects/compatibility)
- [Lakebase Postgres extensions](https://docs.databricks.com/aws/en/oltp/projects/extensions)
- [Databricks SQL stored procedures](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-procedure)
- [Databricks SQL scripting](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-scripting)
- [Agentic code converter](https://docs.databricks.com/aws/en/migration/agentic-code-converter)
