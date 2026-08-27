---
name: mainframe-jcl-migration
description: Evidence-first migration of a JCL-bounded COBOL workload to Python, one selected program at a time, using deterministic mainframe-toolkit MCP evidence.
argument-hint: <workspace-root> <JCL-name-or-path> [PROGRAM-ID] [target-dir]
---

# Mainframe JCL Migration

Use the `mainframe-toolkit` MCP server for dependency discovery, preflight, impact, copybook resolution, deterministic generators, and bounded artifact queries. Do not reconstruct facts from model memory when an MCP tool can establish them.

## Required Inputs

Obtain the workspace root and JCL job name or workspace-relative path. Accept an optional `PROGRAM-ID` and target directory. Default the target to `<workspace>/migration/<JOB-NAME>/`.

Treat `mainframe-migration.json` as authoritative for source paths, library precedence, encoding, extensions, known externals, TODO syntax, and transport profiles. Ask only when a required input is missing or deterministic lookup is ambiguous.

## MCP Rules

- Execute the migration in the main conversation so every MCP call, result, and phase transition remains visible. Do not delegate to subagents unless the user explicitly requests `parallel-agents` execution.
- Default to one campaign worker. Parallelize only independent deterministic MCP/CLI operations in the main conversation; report each operation before it starts and summarize its result when it finishes.
- When the user explicitly requests agent parallelism, state the worker count and program assignments first, pass `--parallel-agents`, use isolated program-only scopes, and keep campaign status in the main conversation. Parallelism reduces elapsed time, not token cost.
- Start with `mainframe_run_migration_tool` running `migration_campaign init`; pass and require a current similarity report when the campaign uses reuse decisions, and never request a complete job preflight JSON. Claim units with campaign `next` and complete them with contained evidence.
- Default each unit to `program-only`. Use `program-with-dependencies` only after an explicit user request.
- Stop the affected phase whenever `continuationAllowed` is false. Present the structured error or findings before another migration call.
- Reuse `runId`, artifact paths, inventory digest, and scope digest while source, configuration, JCL, program, and scope are unchanged.
- Never load, grep, or search a complete JSON artifact. Use `mainframe_query_artifact` with JSON Pointer, filter, projection, pagination, and `nextOffset`.
- Use `mainframe_dependency_graph`, `mainframe_get_callers`, `mainframe_resolve_copybook`, and `mainframe_impact_analysis` for interactive evidence.
- Run packaged deterministic CLIs only with `mainframe_run_migration_tool`. The tool allowlists commands, rejects workspace escapes, caps output, and persists stdout/stderr.
- Inspect `reusableArtifacts` before generating shared evidence. Register validated contracts, readers, and fixtures with campaign `artifact-register`; matching subject identities and SHA-256 digests must be reused by other workers.
- A truncated preview is not a reason to rerun. Query the persisted result by `run_id`.
- Missing copybooks, PROCs, configuration, encoding, compiler facts, and conflicting sources are blockers. Unknown called programs become localized external-adapter TODOs and do not block unrelated units.
- Never put credentials in tool arguments. Keep every path inside the workspace.

## Tool Guide

| Tool | Use |
| --- | --- |
| `mainframe_preflight` | Resolve JCL/program scope, blockers, environment facts, capabilities, and stable digests; persist the full report. |
| `mainframe_dependency_graph` | Page deterministic CALL, COPY, and EXEC relationships, optionally focused by name and direction. |
| `mainframe_get_callers` | Find direct COBOL callers and JCL-to-program backtraces. |
| `mainframe_resolve_copybook` | Apply configured library precedence and list copybook users. |
| `mainframe_impact_analysis` | Persist transitive upstream/downstream impact and return compact risk summaries. |
| `mainframe_run_migration_tool` | Run one of the packaged migration CLIs with an argument array and persisted bounded output. |
| `mainframe_query_artifact` | Query JSON by `path` or `run_id` using `summary`, `get`, `keys`, `page`, or `filter`. |

Useful artifact queries include `get` at `/environmentFacts/compiler`, `filter` at `/findings` with field `/classification` equal to `BLOCK`, and paged graph reads at `/nodes`.

## Workflow

1. Initialize `migration_campaign` with workspace and JCL. Reuse its SQLite database and bounded manifest.
2. Resolve the job execution boundary with graph, callers, impact, JCL flow, and complexity evidence.
3. Validate the requested program. When the user explicitly requests the complete JCL without a program, enter full-job mode and select every unit automatically in deterministic risk order; otherwise select the sole candidate or ask when several remain.
4. Claim a program with campaign `next`; it runs program-only preflight and stores digests. Use the main conversation and one worker by default. Distinct worker IDs and subagents require an explicit `parallel-agents` request.
5. Resolve every direct copybook and establish physical byte layout separately from target transport layout. Never guess `COMP-3`, binary, ODO, or ambiguous REDEFINES behavior.
6. Run SQL and business-rule extraction. Write a semantic brief covering purpose, inputs/outputs, invariants, rules, state transitions, effects, transactions, failures, restart behavior, volume, and source traceability.
7. Generate exactly one program capsule with the same JCL, program, and scope. Preserve generated skeletons as evidence only.
8. Hold a target-design review before implementation. Require business names, cohesive functions, explicit adapters, source-to-target mapping, and restart/checkpoint semantics.
9. Implement the selected program separately from generated evidence. Prefer pure rules, typed domain values, repositories/adapters for effects, and evidence-driven SQL or PySpark choices.
	For PySpark, require canonical contracts and digests before generation; use typed `SparkInput`/`SparkOutput` descriptors so transformations do not know S3, Glue Catalog, or Iceberg. Preserve `_source_uri`, `_source_record_number`, `_run_id`, `_ingested_at`, and `_contract_digest`, and route invalid rows to declared outputs with stable rejection codes. Keep Glue imports lazy, prohibit global Iceberg overwrite, and record bookmark, connector/JAR, IAM, and Lake Formation limitations.
	For a full-job PySpark target, call each module's `transform_frames()` from one Spark application through `SparkPipelineStep`/`run_spark_pipeline`; exchange DataFrames in process, write only final/rejection outputs, and expose optional `--capture-intermediates` Parquet diagnostics that are never consumed by downstream stages or included in the Golden manifest.
10. Validate focused semantic tests and authoritative golden CSV outputs with `golden_validate`. Require the Rust validator result at campaign completion; never generate a per-migration comparator or fall back to Python row/cell loops. Synthetic fixtures prove ingestion boundaries only, not semantic equivalence.
11. Mark the unit complete before selecting another program. In full-job mode, immediately claim the next deterministic pending unit without asking, continue unaffected units when one is blocked, and stop only after all units are completed/blocked or a job-wide blocker requires human input. Report exact tools, artifacts, digests, tests, blockers, and proven equivalence scope.
12. After all job programs are complete, generate `jcl_flow_extractor --format python` and `--format developer-json` under `python/orchestration/`, configure every step command and reviewed DD role, and run the complete developer-only pipeline against at least two isolated sets of local files. Verify automatic same-DSN/backward-reference wiring and preserve each `run-report.json`.

## Architecture Guardrails

Do not promote paragraph-shaped scaffolds into final Python. Reject one method per COBOL paragraph, numeric paragraph names, `WS-`/`FD-` names, simulated `GO TO` or `PERFORM`, mutable global working storage, return-code spaghetti, and class-per-program design unless external compatibility strictly requires it.

Use `Decimal`, dates, and enums for domain values; Pydantic at external validation boundaries; dataclasses or value objects internally; repositories and adapters for files, databases, and external programs; and services or use cases for orchestration. Technology selection must follow evidence, not source-language syntax.

## Completion Status

Finish with `COMPLETE`, `PARTIAL`, or `BLOCKED`. Never claim equivalence beyond authoritative mainframe or verified-equivalent outputs. List unresolved TODOs and blockers explicitly.