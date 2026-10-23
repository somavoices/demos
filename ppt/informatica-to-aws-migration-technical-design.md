# Technical Design Document: PowerCenter to AWS Migration Factory and Runtime

| | |
|---|---|
| **Document status** | Draft for review |
| **Version** | 0.1 |
| **Date** | 2026-10-23 |
| **Owner** | [Name / Role] |
| **Companion document** | Informatica PowerCenter to AWS Data Integration Migration Strategy, v0.4 |
| **Target platform** | Amazon EventBridge, AWS Step Functions, AWS Glue (PySpark), SQL Server, Parquet on Amazon S3 |

> Items marked **(verify)** depend on your PowerCenter version, driver versions or AWS service behavior at build time and must be confirmed in the pilot. Names, thresholds and numbers are design defaults unless stated otherwise.

---

## 1. Introduction

### 1.1 Purpose
This document specifies the technical design of two systems:
1. **The migration factory (build-time plane):** tools, catalogs, templates, skills and agents that convert PowerCenter jobs into AWS code, tests and infrastructure, and prove equivalence.
2. **The target runtime (run-time plane):** the EventBridge, Step Functions and Glue architecture, its libraries, data stores, security, observability and operations.

It gives engineers enough detail to build, review and operate both.

### 1.2 Scope
In scope: design of components, interfaces, schemas, algorithms, infrastructure, test approach, operations and the build plan.
Out of scope: business logic redesign, downstream consumer changes, non-PowerCenter Informatica products, procurement and commercial topics.

### 1.3 Relationship to the strategy
| Strategy section | Covered here in |
|---|---|
| 4 Target state architecture | Sections 2 and 4 |
| 6 AI-driven migration factory | Section 3 |
| 6.13 Baseline capture | Section 3.10 and Section 6 |
| 7 Conversion catalogue | Sections 3.6, 3.7 and 5 |
| 8 Parity decisions | Sections 3.3, 3.7 and 5 |
| 9 Orchestration | Sections 3.5, 3.9 and 4.7 |
| 10 Testing | Section 6 |
| 11 Non-functional requirements | Sections 4.8, 4.9 and 7 |

### 1.4 Design goals
| ID | Goal |
|---|---|
| DG-1 | Deterministic first: facts come from tools; agents never recall definitions from memory |
| DG-2 | Like-for-like: the conversion preserves legacy behavior; changes are explicit decisions |
| DG-3 | Reproducible: same specification and tool versions produce the same rule-based output |
| DG-4 | Traceable: every Informatica object maps to a named AWS artifact with provenance |
| DG-5 | Idempotent and restartable: any job or workflow can be rerun safely |
| DG-6 | Observable: structured logs, metrics, run log and alarms on every job |
| DG-7 | Secure: least privilege, no secrets in code or prompts, no agent access to production |
| DG-8 | Scalable delivery: many agents work in parallel within review and environment capacity |
| DG-9 | Testable: every generated unit has tests; every job has a golden-master benchmark |

### 1.5 Constraints and assumptions
- Glue 4.0 or later; the design is version-agnostic and uses only stable Glue features. Confirm the recommended Glue version at build time **(verify)**.
- Targets are SQL Server and Parquet on S3. Sources are SQL Server and flat files.
- Infrastructure as code uses Terraform unless Q18 selects another tool.
- A non-production PowerCenter environment can run workflows on test data (Q22). If not, the fallback in Section 3.10.5 applies.
- Agents run in Cursor on developer or CI workspaces with access to the factory CLI, the repository and the AWS dev and test accounts only.

### 1.6 Conventions
| Item | Convention |
|---|---|
| Job id | `J` + three digits, for example `J003` |
| Parity decision id | `D` + three digits |
| Pattern id | `P-<AREA>-<NN>`, for example `P-UPSERT-01` |
| Gate id | `G1` to `G9` |
| Unsupported construct code | `U-<TYPE>`, for example `U-JAVA` |
| Decision record | `ADR-NNN` (Section 10) |
| Run id | Step Functions execution name, passed to Glue as `--RUN_ID` |

---

## 2. Architecture Overview

### 2.1 Two planes
```
BUILD-TIME PLANE (factory)                         RUN-TIME PLANE (target runtime)
PowerCenter XML + control CSVs                     EventBridge -> Step Functions -> Glue
   -> parse -> specification                          -> SQL Server / S3 Parquet
   -> plan -> generate -> test -> gates              Support: DynamoDB, Secrets, SSM, KMS,
   -> baseline, parallel run -> PR                   CloudWatch, SNS, IAM
        |                                                      ^
        '---- generated code, IaC, tests, reports ---> Git ----'  (CI deploys by IaC)
```

### 2.2 Component inventory
**Factory plane**
| ID | Component | Type | Responsibility |
|---|---|---|---|
| F1 | `mig` CLI | Python package | Single entry point for all deterministic tools (Section 3.2) |
| F2 | Parser (`pc_parse`) | Library | PowerCenter XML to specification JSON |
| F3 | Control validator | Library | CSV schema, referential and XML cross-checks |
| F4 | DAG planner (`dag_plan`) | Library | Graph, layers, critical path, orchestration tier |
| F5 | Expression translator | Library | Informatica expressions to Spark SQL expressions |
| F6 | Catalogs | YAML files | Function catalog, pattern catalog |
| F7 | Template library | Jinja2 templates | Pipeline, Glue job, state machine and IaC templates |
| F8 | Orchestration generator | Library | Tier-based state machine and schedule generation |
| F9 | Baseline capture | Library plus scripts | Run legacy on frozen inputs; store golden outputs and statistics |
| F10 | Comparator and parallel run | Library | Equivalence checks, tolerances, difference classification |
| F11 | Gate runner | Library | Execute gates G1 to G6 and report results |
| F12 | Agents and skills | Cursor skills | Orchestrator, job, test, reviewer and documentation playbooks |
| F13 | Trace matrix | Library | Informatica object to AWS artifact mapping with provenance |
| F14 | Results store | S3 + Athena | Baselines, comparison results, gate reports |

**Runtime plane**
| ID | Component | Responsibility |
|---|---|---|
| R1 | EventBridge Scheduler and rules | Time, S3 event and completion-event triggers |
| R2 | Step Functions state machines | Workflow orchestration, retry, catch, redrive |
| R3 | Glue jobs | PySpark data processing, one job per session |
| R4 | Runtime library | Logging, metrics, retry, state access, sources and sinks |
| R5 | State store (DynamoDB) | Mapping variable equivalents, watermarks |
| R6 | Run log (DynamoDB) | Per-run status, row counts, timings |
| R7 | S3 data areas | Parquet outputs, rejects, staging files, golden artifacts |
| R8 | SQL Server | Sources and targets, staging tables |
| R9 | Secrets Manager and SSM Parameter Store | Credentials, job parameters |
| R10 | Observability stack | CloudWatch Logs, metrics, alarms, dashboards, SNS |
| R11 | Security and network | IAM, KMS, VPC, endpoints, CloudTrail |

### 2.3 Build-time data flow
```
export XML ---> mig parse ---> spec.json -----------+
control CSVs -> mig validate-control ---> OK --------+--> mig plan ---> plan.json
                                                     |
frozen inputs -> mig baseline -> golden + stats ----+--> agent: generate code and tests
                                                     |        (templates, catalogs, spec)
                                                     v
                                              mig gates (G1-G6) <-- loop until pass
                                                     |
                                              mig compare (G7) -> parallel run (G8)
                                                     |
                                              PR + report + trace matrix -> human review (G9)
```

### 2.4 Technology stack
| Layer | Choice | Notes |
|---|---|---|
| Factory language | Python 3.11 or later | Same language family as Glue PySpark code |
| XML parsing | `lxml` (iterparse) with entity resolution disabled | Large exports; XXE protection |
| Schemas | JSON Schema (validated with `jsonschema`), `pydantic` models | Specification and plan |
| Templates | Jinja2 | Strict undefined mode |
| CLI | `typer` or `argparse` | JSON output option |
| Tests | pytest, local PySpark, `datacompy` | Same Spark family as Glue |
| Lint and types | ruff, mypy, sqlfluff (T-SQL dialect) | CI gates |
| IaC | Terraform with AWS provider | Q18 may select CDK |
| Runtime | Glue (Spark, Python 3), Step Functions Standard, EventBridge Scheduler | Standard needed for `.sync` Glue integration |
| State and logs | DynamoDB on-demand, CloudWatch | |
| Results store | S3 Parquet queried with Athena | |
| Agents | Cursor agents and skills | Q19, Q23 constraints apply |
| CI | Team's CI tool (Q18) | Stages in Section 4.10 |

### 2.5 Interfaces
| ID | From to | Mechanism | Contract |
|---|---|---|---|
| I1 | Agent to factory tools | CLI, JSON on stdout | Section 3.2 |
| I2 | Factory to PowerCenter | `pmcmd` and `pmrep` over the non-prod client host | Section 3.10 |
| I3 | Factory to Git | Feature branch, pull request | Section 3.1 |
| I4 | CI to AWS | IaC deploy using CI roles | Section 4.10 |
| I5 | EventBridge to Step Functions | `StartExecution` | Section 4.7 |
| I6 | Step Functions to Glue | `startJobRun.sync` with arguments | Section 4.7 |
| I7 | Glue to SQL Server | JDBC through a Glue connection | Section 4.4 |
| I8 | Glue to S3 | S3 API through a VPC endpoint | Section 4.5 |
| I9 | Glue to DynamoDB, Secrets, SSM, CloudWatch | AWS SDK | Sections 4.3 and 4.6 |
| I10 | Comparator to results store | Parquet write to S3 | Section 3.10 |

---

## 3. Factory Design (Build-Time Plane)

### 3.1 Repository layout
```
repo/
  .cursor/
    rules/                      project rules (conventions, forbidden actions)
    skills/                     agent playbooks (Section 3.12)
  factory/                      the mig package (F1 to F13)
    mig/
      cli.py
      parse/                    XML parser, spec models
      control/                  CSV schemas and validators
      plan/                     DAG planner
      expr/                     expression lexer, parser, translator
      gen/                      generators, template rendering
      baseline/                 legacy runner, capture
      compare/                  comparator, tolerances
      gates/                    gate runner and checks
      trace/                    traceability matrix
    catalogs/
      functions.yaml            expression function catalog
      patterns.yaml             approved patterns
    templates/                  Jinja2 templates
    tests/                      factory unit tests and regression suite
  glue/
    lib/                        runtime library (Section 4.3)
    pipelines/                  one module per mapping (generated)
    jobs/entrypoint.py          single Glue entry script
  infra/
    modules/                    reusable Terraform modules
    workflows/<workflow>/       generated Terraform per workflow
  migration/
    control/                    CSV control files
    specs/                      one spec JSON per mapping and workflow
    plans/                      one plan JSON per workflow
    reports/                    conversion, gate, parallel-run reports; trace matrix
  baseline/<job_id>/            manifest, inputs (hashed), golden outputs, statistics
  tests/<job_id>/               generated unit and golden-master tests
```
Rules: generated files carry a header with the specification hash, tool and template versions and `DO NOT EDIT BY HAND`; hand edits go through template or catalog changes, or are recorded as a reviewed manual override file `overrides/<job_id>.py` that the generator preserves.

### 3.2 Factory CLI (`mig`)
All deterministic tools are exposed through one CLI so agents use a uniform interface.

| Command | Purpose | Key arguments | Output |
|---|---|---|---|
| `mig parse` | XML to specification | `--xml`, `--out`, `--params` | `spec.json`, parse report |
| `mig validate-control` | Gate G1 | `--control-dir`, `--specs-dir` | Validation report |
| `mig plan` | Dependency graph and tier | `--workflow` | `plan.json` |
| `mig baseline` | Capture golden master | `--job`, `--inputs`, `--repeats` | `baseline/<job>/` |
| `mig generate` | Render rule-based artifacts | `--job`, `--what pipeline,orchestration,iac,tests` | Files plus coverage matrix |
| `mig gates` | Run gates | `--job`, `--upto G6` | Gate report JSON |
| `mig compare` | Equivalence check (G7) | `--job`, `--golden`, `--candidate` | Comparison report |
| `mig parallel-run` | Old vs new cycles (G8) | `--workflow`, `--cycles` | Cycle reports |
| `mig report` | Conversion report | `--job` | Markdown and JSON |
| `mig trace` | Update traceability matrix | `--job` | Matrix rows |
| `mig status` | Wave status board | `--wave` | Board JSON |

**Conventions**
- `--json` prints a single JSON envelope: `{"ok": bool, "command": str, "data": {...}, "errors": [...], "warnings": [...]}`.
- Exit codes: `0` success, `1` gate or check failed (work needed), `2` usage or configuration error, `3` tool error (bug or environment).
- All outputs are redacted by the secret filter (Section 3.14).
- Idempotent: rerunning a command with the same inputs gives the same outputs.

### 3.3 Control files and validation
Stored in `migration/control/`; changed only by pull request.

**Schemas**
| File | Column | Type | Constraint |
|---|---|---|---|
| `job_inventory.csv` | job_id | string | unique, pattern `J\d{3,}` |
| | workflow | string | must exist in parsed specs |
| | session, mapping | string | must exist in parsed specs |
| | source_type, target_type | enum | `sqlserver`, `flatfile`, `parquet`, joined with `+` for several |
| | tier | enum | `Simple`, `Medium`, `Complex` |
| | wave | integer | >= 1 |
| | owner | string | non-empty |
| `dependencies.csv` | workflow | string | exists |
| | predecessor | string | `START` or a job_id |
| | successor | string | job_id |
| | condition | enum | `SUCCESS`, `FAILURE`, `EXPRESSION` |
| | expression | string | required when condition is `EXPRESSION` |
| | trigger | enum | `SCHEDULE`, `EVENT`, `FILE`, blank |
| | schedule | string | EventBridge cron or rate, when trigger is `SCHEDULE` |
| `parity_decisions.csv` | decision_id | string | unique, `D\d{3,}` |
| | scope | string | `GLOBAL` or a job_id |
| | rule | enum | from the rule registry (Section 3.7) |
| | value | string | validated per rule |
| | approved_by, date | string | required |
| `connections_map.csv` | informatica_connection | string | used by at least one job |
| | glue_connection, secret_ref, environment | string | non-empty |
| `parameters.csv` | job_id, parameter | string | job exists |
| | source | enum | `JOB_ARG`, `SSM`, `STATE_STORE` |
| | default, environment | string | |

**Validation rules (gate G1)**
| Rule | Check |
|---|---|
| C-01 | Every CSV validates against its schema |
| C-02 | Every `job_id` in any file exists in `job_inventory.csv` |
| C-03 | `dependencies.csv` forms a DAG per workflow (no cycles) |
| C-04 | Every workflow link in the parsed XML appears in `dependencies.csv`; otherwise a discrepancy is reported |
| C-05 | Every link in `dependencies.csv` not found in the XML is reported as an addition for human confirmation |
| C-06 | Every parity rule referenced by a pipeline decision point has an approved decision, or a default marked `provisional` |
| C-07 | Every Informatica connection used by a job is in `connections_map.csv` |
| C-08 | Every `$$` variable and parameter in a specification is in `parameters.csv` |
| C-09 | Tier in the inventory is consistent with the computed complexity score, else a warning |

Conflict handling between XML and CSV follows Q21; until answered, discrepancies block G1 and require a human decision recorded in the decision log.

### 3.4 PowerCenter XML parser and specification

#### 3.4.1 Input
PowerCenter repository export XML (root `POWERMART`), one file per workflow with its mappings, sessions, sources, targets and reusable objects, plus optional parameter files. Element names below follow the common export format; confirm against the DTD of your version **(verify)**.

#### 3.4.2 Element mapping
| XML element | Meaning | Specification target |
|---|---|---|
| `FOLDER` | Container | `folder` |
| `SOURCE`, `TARGET` | Definitions with fields and database type | `sources[]`, `targets[]` with typed ports |
| `MAPPING` | Mapping | `mapping` |
| `TRANSFORMATION` (with `TRANSFORMFIELD`, `TABLEATTRIBUTE`) | Transformation definition, ports, properties | `transformations[]` |
| `INSTANCE` | Use of a source, target or transformation in the mapping | `instances[]` |
| `CONNECTOR` | Port-level link (`FROMINSTANCE`, `FROMFIELD`, `TOINSTANCE`, `TOFIELD`) | `edges[]` |
| `TARGETLOADORDER` | Target load order | `target_load_order[]` |
| `MAPPINGVARIABLE` | `$$` variable with aggregation type | `variables[]` |
| `MAPPLET` | Reusable mapplet | Inlined with origin recorded |
| `SESSION` with `SESSTRANSFORMATIONINST`, `ATTRIBUTE` | Session properties and per-instance overrides | `session` |
| `WORKFLOW` with `TASKINSTANCE`, `WORKFLOWLINK`, `SCHEDULER`, `WORKFLOWVARIABLE` | Workflow structure | `workflow` |

#### 3.4.3 Algorithm
1. Stream-parse with `lxml.etree.iterparse` (entity resolution and network access disabled).
2. Build typed objects (pydantic models) for sources, targets, transformations, instances, connectors, variables, session, workflow.
3. Resolve references: shortcuts to shared folders, reusable transformations and mapplets are expanded; each expanded element keeps `origin` (object name, folder).
4. Apply session-level overrides on top of mapping definitions (SQL overrides, filters, connection names, properties) and record that an override applied.
5. Resolve parameter files: substitute known `$$` values for documentation, but keep the symbolic parameter in the specification.
6. Build the port graph from connectors; check every port is connected or explicitly unused.
7. Parse expressions into ASTs using the translator front end (Section 3.6); store both the original text and the AST.
8. Compute the complexity score from transformation types, expression complexity, overrides and state usage.
9. Emit unsupported-construct records with stable codes.
10. Sort all collections by stable keys, compute a content hash, write `spec.json`.

#### 3.4.4 Specification schema (abridged)
| Field | Type | Description |
|---|---|---|
| `spec_version` | string | Schema version |
| `hash` | string | SHA-256 over canonical content |
| `job_id` | string | From inventory |
| `mapping` | object | Name, description, folder |
| `session` | object | `commit_interval`, `commit_type`, `target_load_type`, `row_treatment`, `high_precision`, `recovery`, `error_threshold`, `pre_sql`, `post_sql`, `connections`, `overrides[]` |
| `variables[]` | array | `name`, `type`, `aggregation`, `initial_value` |
| `sources[]` | array | `name`, `kind` (`relational` or `flatfile`), `table`, `ports[]`, `filter`, `sql_override`, `file` (delimiter, header, encoding, fixed layout) |
| `transformations[]` | array | `name`, `type`, `ports[]`, `properties`, `origin` |
| `ports[]` | array | `name`, `datatype`, `precision`, `scale`, `direction`, `expression_text`, `expression_ast` |
| `edges[]` | array | `from_instance`, `from_port`, `to_instance`, `to_port` |
| `targets[]` | array | `name`, `kind`, `table` or `file`, `ports[]`, `load_type`, `truncate`, `pre_sql`, `post_sql` |
| `target_load_order[]` | array | Ordered target instance names |
| `unsupported[]` | array | `code`, `object`, `detail` |
| `complexity` | object | `score`, `tier`, `factors[]` |

Workflow specification: `tasks[]` (session, command, email, decision, timer, event wait, assignment, worklet) with properties; `links[]` (`from`, `to`, `condition_expression`); `scheduler` (type, start, repeat, calendar); `variables[]`; `parameter_file`.

#### 3.4.5 Unsupported construct codes
| Code | Meaning |
|---|---|
| `U-JAVA` | Java transformation |
| `U-CUSTOM` | Custom transformation |
| `U-XML` | XML parser or generator transformation |
| `U-TXCTRL` | Transaction Control |
| `U-DYNLKP` | Lookup with dynamic cache |
| `U-STOREDPROC` | Stored Procedure transformation (pattern may exist) |
| `U-FUNC-<name>` | Expression function not in the catalog |
| `U-SQLOVR` | SQL override that cannot be pushed down as is |
| `U-OTHER` | Anything else not recognized |

#### 3.4.6 Quality attributes
- **Determinism:** the same XML produces byte-identical `spec.json`.
- **Completeness (G2):** parse coverage is the number of recognized elements over total elements in scope; anything unrecognized is listed, never ignored.
- **Safety:** parsing never executes content; XML external entities and DTD loading are disabled.
- **Performance:** streaming parse; a large export is processed per workflow.

### 3.5 Dependency planner (`dag_plan`)

#### 3.5.1 Inputs and outputs
Inputs: workflow specification, `dependencies.csv`, job inventory, table lineage from specifications (sources and targets per session), optional run history (durations). Output: `plan.json`.

#### 3.5.2 Algorithm
1. Build the graph from workflow links; merge `dependencies.csv` edges (rule C-04, C-05).
2. Infer implicit table-level edges: job A to job B when a target table of A is a source table of B and no explicit path exists; mark as `inferred`.
3. Detect cycles (Kahn's algorithm); a cycle fails G1.
4. Compute topological layers (longest distance from the start); jobs in the same layer are independent.
5. With run history, compute the critical path as the longest path by median duration, and slack for every job.
6. Choose the orchestration tier (3.5.3).
7. Emit parallel groups, edge conditions, trigger and schedule.

#### 3.5.3 Orchestration tier rules
| Condition | Tier |
|---|---|
| One session, no conditions, no downstream links, no pre/post tasks | `direct_schedule` |
| Only sequential success links, no parallel branches, no decision, timer or event wait | `linear_chain` |
| Any parallel branch, link condition other than SUCCESS, decision, command, timer, event wait, or worklet | `full_state_machine` |
| Chain longer than the length limit (default 40 tasks) | Split into nested state machines per layer group |

#### 3.5.4 `plan.json` (example)
```json
{
  "workflow": "wf_daily_sales",
  "tier": "full_state_machine",
  "layers": [["J001", "J002"], ["J003"]],
  "edges": [
    {"from": "J001", "to": "J003", "condition": "SUCCESS", "source": "xml+csv"},
    {"from": "J002", "to": "J003", "condition": "SUCCESS", "source": "xml+csv"}
  ],
  "critical_path": ["J001", "J003"],
  "slack_minutes": {"J002": 11},
  "trigger": {"type": "SCHEDULE", "expression": "cron(30 1 * * ? *)", "timezone": "Asia/Kolkata"},
  "warnings": []
}
```

### 3.6 Expression translator

#### 3.6.1 Design
Informatica expressions are translated into **Spark SQL expression strings** (used with `F.expr(...)`) rather than a Column-API code tree, because strings are easy to snapshot-test, diff and parse-check. Pipeline: lexer, parser to AST, type inference, translation through the function catalog, emission, validation by Spark's parser (gate G4).

#### 3.6.2 Grammar coverage
Literals (string, numeric, date via `TO_DATE`), ports, `$$` variables and parameters, arithmetic and comparison operators, logical operators, string concatenation (`||`), function calls, nested calls, variable ports referencing earlier ports in port order.

#### 3.6.3 Evaluation order
Expression transformation ports are evaluated in port order; variable ports can reference earlier variable and input ports and retain values between rows. The generator emits ports as sequential `withColumn` steps in the same order. A variable port that depends on a previous row's value (state across rows) is flagged `U-OTHER` and routed to mode C.

#### 3.6.4 Function catalog (`functions.yaml`)
Each entry defines the mapping, null semantics, version status and tests.
```yaml
- name: IIF
  informatica: "IIF(cond, a [, b])"
  spark: "CASE WHEN {cond} THEN {a} ELSE {b} END"
  null_semantics: "NULL condition takes the else branch"   # verify by baseline
  verified_by_baseline: false
  tests:
    - {inputs: {cond: true, a: 1, b: 2}, expected: 1}
- name: SUBSTR
  informatica: "SUBSTR(str, start [, len])"
  spark: "substring({str}, {start}, {len})"
  notes: "Both use 1-based positions; negative start handling differs, verify"
  verified_by_baseline: false
- name: LTRIM
  informatica: "LTRIM(str [, chars])"
  spark: "ltrim({str})"              # single-argument form only
  unsupported_forms: ["two-argument form: flagged U-FUNC-LTRIM"]
```
`verified_by_baseline` becomes true only when an edge-case baseline run confirms the Spark behavior equals the legacy behavior; until then the generator emits the translation and attaches a review flag.

#### 3.6.5 Translation examples
| Informatica | Spark SQL | Notes |
|---|---|---|
| `IIF(amount > 0, 'P', 'N')` | `CASE WHEN amount > 0 THEN 'P' ELSE 'N' END` | Null handling per catalog |
| `ISNULL(x)` | `x IS NULL` | |
| `UPPER(LTRIM(RTRIM(s)))` | `upper(ltrim(rtrim(s)))` | |
| `TO_DATE(s, 'YYYY-MM-DD')` | `to_date(s, 'yyyy-MM-dd')` | Format token mapping table; Spark 3 datetime patterns |
| `s1 \|\| s2` | `concat(s1, s2)` or `concat_ws('', s1, s2)` | NULL behavior differs by engine; choose by parity decision and verify by baseline |
| `DECODE(v, 1, 'A', 2, 'B', 'C')` | `CASE WHEN v = 1 THEN 'A' WHEN v = 2 THEN 'B' ELSE 'C' END` | NULL matching verified by baseline |
| `INSTR(s, 'a', 1, 2)` | Not translated | Multi-argument form flagged `U-FUNC-INSTR` |

#### 3.6.6 Types and numerics
Port datatypes and precision come from the specification. Arithmetic on decimals uses explicit casts according to the numeric parity decision (for example D003) because Spark's decimal multiplication can reduce result scale. The generator never relies on implicit widening for decimals.

#### 3.6.7 Quality attributes
- Every catalog entry has golden tests; the catalog regression suite runs on every change (Section 6.8).
- Unknown functions produce `U-FUNC-<name>` and a mode B or C work item; they are never guessed.
- Catalog additions proposed by an agent require human review before merge.

### 3.7 Parity rule registry and pattern catalog

#### 3.7.1 Parity rule registry
Each rule has a closed set of allowed values. A decision row in `parity_decisions.csv` selects a value; generators read it, never infer it.

| Rule | Allowed values | Effect on generated code |
|---|---|---|
| `commit_interval` | `per_partition` (default), `batch_size:<n>` | JDBC `batchsize`, partition count; not a transactional equivalent (Spark commits per write partition) |
| `update_strategy` | `staging_merge`, `append_only`, `delete_insert` | Target writer type (Section 4.4) |
| `null_vs_empty` | `preserve`, `empty_to_null` | Cast and normalize step on string ports |
| `decimal_rounding` | `half_up`, `half_even`, `truncate` | Explicit rounding in decimal expressions |
| `lookup_multi_match` | `first`, `last`, `error`, `all` | Lookup join with deterministic ordering key |
| `lookup_no_match` | `null`, `default:<v>`, `reject` | Left join plus fill or reject branch |
| `surrogate_key` | `identity_db`, `sequence_table`, `hash_key` | Key generation step |
| `reject_handling` | `file`, `table`, `fail` | Reject writer, reason codes |
| `string_compare` | `case_sensitive`, `case_insensitive`, `collation:<name>` | Comparison and join key normalization |
| `timestamp_precision` | `datetime`, `datetime2(<n>)` | Cast before write |
| `mapping_variable_store` | `dynamodb` (default), `ssm`, `table` | State read/write calls |
| `parquet_naming` | `part_files`, `single_file`, `named:<pattern>` | Writer repartition and rename step |
| `parquet_partitioning` | `none`, `by:<cols>` | `partitionBy` |

Unresolved rules default to the value marked `provisional` and appear in the conversion report; G9 blocks a job with provisional decisions that affect output (cutover rule).

#### 3.7.2 Pattern catalog (`patterns.yaml`)
Patterns are named, tested, parameterized code fragments. Mode B conversion may only compose patterns; it may not invent new constructs.

| Pattern | Covers | Parameters |
|---|---|---|
| `P-READ-JDBC-01` | Hash-partitioned JDBC read | table or query, key, partitions |
| `P-READ-FILE-01` | Delimited and fixed-width read with explicit schema | layout, encoding, header |
| `P-FILTER-01` | Filter transformation | condition |
| `P-EXPR-01` | Expression transformation | ordered ports |
| `P-JOIN-01` | Joiner (normal, master/detail outer) | type, keys, null-key policy |
| `P-LOOKUP-01` | Connected lookup with policy | keys, return ports, multi-match, no-match |
| `P-AGG-01` | Aggregator | group keys, aggregates, sorted-input flag |
| `P-ROUTER-01` | Router groups with default group | conditions |
| `P-SORT-01`, `P-RANK-01` | Sorter, Rank with tie policy | keys, ties |
| `P-UNION-01` | Union | port mapping |
| `P-SEQ-01` | Sequence Generator | key strategy |
| `P-UPDSTRAT-01` | Update Strategy flag column | flag expressions |
| `P-UPSERT-01` | Staging table plus `MERGE ... WITH (HOLDLOCK)` | keys, columns |
| `P-DELINS-01` | Delete then insert by key | keys |
| `P-WRITE-JDBC-01` | Batched append or overwrite | batch size, partitions |
| `P-WRITE-PARQUET-01` | Parquet write | partition, naming |
| `P-WRITE-FILE-01` | Delimited and fixed-width write | layout |
| `P-REJECT-01` | Reject capture with reason codes | rule set |
| `P-STATE-01` | Mapping variable read and write | key, aggregation |
| `P-AUDIT-01` | Row counts and log events | none |

### 3.8 Generators

#### 3.8.1 Generation pipeline
```
spec.json + plan.json + parity decisions + catalogs + templates
   -> select pattern per transformation (rule table, Section 5.1)
   -> render pipeline module, entry config, tests, state machine, IaC
   -> write coverage matrix (every port and transformation mapped, or flagged)
   -> format (ruff), header with hashes and versions
```

#### 3.8.2 Mode assignment (per transformation)
| Condition | Mode | Producer |
|---|---|---|
| Transformation type in rule table; all functions in catalog | A | Generator (no agent text) |
| Composable from patterns; needs parameter choices or glue code | B | Agent composes patterns; validated by G3 to G6 |
| Unsupported code present, or state across rows, or custom logic | C | Engineer-led; agent assists with tests and documentation |

Job mode is the highest mode among its transformations. The coverage matrix records the mode per transformation.

#### 3.8.3 Determinism
Rendering uses sorted inputs, fixed template versions and no timestamps in generated bodies (timestamps live in the header only, and the header is excluded from diff checks). Re-running `mig generate` with unchanged inputs yields no diff.

#### 3.8.4 Overrides
Hand or agent-written code that cannot come from a template lives in `glue/pipelines/overrides/<job_id>.py`, is imported by the generated module at a named hook (`pre_transform`, `post_transform`, `custom_load`), and is never overwritten. An override requires a reason line and a reviewer.

### 3.9 Orchestration generation

#### 3.9.1 Inputs and outputs
Input: `plan.json`, job inventory, parameters. Output: ASL JSON (or a Terraform `templatefile` rendering), EventBridge Scheduler or rule resources, alarms, IAM policy documents.

#### 3.9.2 State machine rules
| Plan element | ASL construct |
|---|---|
| Job | `Task` with `arn:aws:states:::glue:startJobRun.sync`, `Arguments` from the plan |
| Parallel layer (more than one job) | `Parallel` with one branch per job, each branch with its own `Retry` and `Catch` |
| Edge condition `FAILURE` | `Catch` to a named recovery state |
| Edge condition `EXPRESSION` | `Choice` state over a prior result field |
| Timer task | `Wait` |
| Event wait | Task token pattern (`.waitForTaskToken`) with callback from EventBridge rule |
| Command or email task | Lambda task (small, reviewed) or SNS publish |
| Worklet | Nested state machine (`startExecution.sync:2`) |
| More than 40 states, or iteration over a list | Nested state machine or `Map` with `MaxConcurrency` |

Common template parts: `Retry` for `Glue.ConcurrentRunsExceededException`, `Glue.InternalServiceException` and `States.TaskFailed` (limited attempts, backoff); a final `Catch` that publishes to SNS and writes a failed record to the run log; a start state that writes a `STARTED` run log record; an end state that writes `SUCCEEDED`.

#### 3.9.3 Example task (abridged)
```json
"J003_orders": {
  "Type": "Task",
  "Resource": "arn:aws:states:::glue:startJobRun.sync",
  "Parameters": {
    "JobName": "mig-j003-orders",
    "Arguments": {
      "--RUN_ID.$": "$$.Execution.Name",
      "--BUSINESS_DATE.$": "$.business_date"
    }
  },
  "Retry": [{"ErrorEquals": ["Glue.ConcurrentRunsExceededException"],
             "IntervalSeconds": 60, "MaxAttempts": 3, "BackoffRate": 2.0}],
  "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyFailure", "ResultPath": "$.error"}],
  "Next": "MarkSucceeded"
}
```

#### 3.9.4 Scheduling and triggers
| Informatica | AWS |
|---|---|
| Scheduler: run at time or interval | EventBridge Scheduler (cron or rate, explicit time zone) with `StartExecution` target |
| File arrival (event wait) | S3 event to EventBridge rule, then `StartExecution` or task token callback |
| Upstream workflow completion | EventBridge rule on Step Functions execution status `SUCCEEDED`, or a parent state machine for strict chains |
| Calendar or holiday exclusions | Choice state reading a calendar table, or schedule exclusion list |
| Concurrent run protection | Execution name derived from business date (duplicate start is rejected) plus Glue `MaxConcurrentRuns = 1` |

Schedules are created `DISABLED` in production until cutover (strategy Section 14).

#### 3.9.5 Redrive and restart
Failed executions are restarted with Step Functions redrive from the failed state, which preserves completed upstream results. Jobs are idempotent (Section 4.3.6), so rerunning a completed job is safe. Restart guidance is part of each workflow runbook.

### 3.10 Baseline capture and parallel run

#### 3.10.1 Baseline capture components
| Component | Function |
|---|---|
| Input freezer | Extracts source data to snapshot files, hashes each file, writes manifest |
| Legacy runner | Copies the workflow with connections pointed to frozen inputs and shadow targets; runs `pmcmd startworkflow -wait`; captures exit code and logs |
| Stats collector | Reads session logs and repository views for row counts and timings |
| Profiler | PySpark column profiling of outputs (Section 6.5) |
| Determinism analyzer | Compares repeated runs; produces exclusion/normalization list and tolerances |
| Packager | Writes `baseline/<job_id>/` with manifest and checksums |

#### 3.10.2 Baseline manifest (`manifest.json`)
```json
{
  "job_id": "J003",
  "captured_at": "2026-10-23T09:12:00Z",
  "legacy": {"workflow": "wf_daily_sales", "session": "s_orders", "version": "<PowerCenter version>"},
  "inputs": [{"path": "inputs/orders.parquet", "sha256": "...", "rows": 1250000}],
  "runs": 3,
  "outputs": [{"path": "golden_output.parquet", "sha256": "...", "rows": 1248812}],
  "nondeterministic_columns": ["load_ts"],
  "normalizations": [{"column": "load_ts", "rule": "exclude"}],
  "tolerances": {"amount": {"abs": 0.0}, "ratio": {"abs": 1e-9}},
  "stats_file": "run_stats.json"
}
```

#### 3.10.3 Comparator
| Check | Method | Default tolerance |
|---|---|---|
| Row counts | Equal per target | 0 |
| Key set | Anti-joins both ways on business key | 0 differing keys |
| Row equality | Row hash over non-excluded columns by key; `datacompy` for column-level diff on mismatches | 0 mismatched rows |
| Column profile | Null, distinct, min/max, sum | Exact; floats within the tolerance in the manifest |
| Rejects | Same rows rejected with same reason codes | Exact |
| Files | Row count, schema, encoding, delimiter; content hash when naming and ordering are decided | Exact |
| State | Mapping variable value after the run | Exact |

Difference classification: `DEFECT` (new code wrong), `PARITY_GAP` (needs a decision), `LEGACY_DEFECT` (legacy behavior arguably wrong), `NON_DETERMINISTIC` (excluded by baseline), `ENVIRONMENT` (collation, precision, time zone). The agent proposes a class with evidence; a human approves any class other than `DEFECT`.

For large SQL Server tables the comparator computes per-partition aggregates in the database (`CHECKSUM_AGG`, `HASHBYTES` over a concatenated key and column list) and drills into mismatching partitions only.

#### 3.10.4 Parallel run (gate G8)
1. Freeze an input snapshot for a business date.
2. Start the legacy workflow (`pmcmd`) and the Step Functions execution against shadow targets A and B.
3. Compare per Section 3.10.3; store results in the results store.
4. Repeat for the required cycles (default: 5 business cycles, including one month-end and one incremental, one rerun and one failure-injection cycle).
5. Gate passes when all cycles are clean or every difference is an approved decision.

#### 3.10.5 Fallback when legacy cannot be run on test data
Compare against legacy production output for a defined window (Q22): source extracts are taken at the same point as the legacy run, outputs are compared read-only, and the report flags reduced confidence. Golden-master tests use the captured output as expected data.

### 3.11 Gate runner
| Gate | Name | Checks (automated) | Pass condition |
|---|---|---|---|
| G1 | Control files | Rules C-01 to C-09 | No errors |
| G2 | Parse completeness | Parse coverage report | 100% recognized or listed as unsupported |
| G3 | Static checks | ruff, mypy, sqlfluff, import allowlist, identifier whitelist, secret scan | Clean |
| G4 | Expression validation | Spark parser accepts each expression; catalog regression | Clean |
| G5 | Unit tests | pytest on generated and catalog tests; coverage floor | Pass; coverage >= 80% lines on pipelines |
| G6 | Golden master (local) | Run pipeline on frozen inputs with local Spark; comparator | Equivalent per tolerances |
| G7 | Reconciliation | Deployed pipeline in test account vs baseline, full volume | Equivalent or approved differences |
| G8 | Parallel run | Section 3.10.4 | Required cycles clean |
| G9 | Sign-off | Human review checklist (Section 9.4) | Approvals recorded |

Gate reports are JSON: `{gate, status, started, ended, checks[{id, status, detail}], artifacts[]}`. The report for each job is stored in `migration/reports/<job_id>/`. A later gate does not run until earlier gates pass for the same specification hash; a changed hash invalidates earlier results.

### 3.12 Agents and skills

#### 3.12.1 Agent roles
| Agent | Responsibility | Can write | Cannot |
|---|---|---|---|
| Orchestrator | Reads wave plan and status; creates work items; assigns job agents; enforces budgets | Status files, branches names | Change catalogs, control files |
| Job agent | Runs the per-job workflow (Section 3.12.3) | Own branch: `glue/pipelines/<job>`, `tests/<job>`, `infra/workflows/<wf>`, reports | Edit shared libs, catalogs, control CSVs, other jobs |
| Test and reconciliation agent | Baseline, comparator runs, triage, proposes decisions | Baselines, reports, proposed decision rows (PR) | Approve decisions |
| Reviewer agent | Independent check of diffs against specification and catalogs | Review comments | Edit code |
| Documentation agent | Conversion report, runbook, trace matrix entries | `migration/reports`, `docs/` | Edit code |

#### 3.12.2 Skills (Cursor playbooks)
| Skill | Purpose | Main tools |
|---|---|---|
| `wave-orchestration` | Pick next jobs, spawn job agents, track status | `mig status`, `mig plan` |
| `convert-job` | End-to-end per-job workflow | `mig parse/generate/gates` |
| `baseline-capture` | Run the baseline procedure | `mig baseline` |
| `compose-pattern` | Mode B conversion rules and limits | catalogs |
| `triage-diff` | Classify comparator differences, propose decisions | `mig compare` |
| `review-job` | Reviewer checklist | `mig gates`, git diff |
| `write-report` | Report and runbook templates | `mig report`, `mig trace` |

Each skill states: inputs, ordered steps with the exact commands, stop conditions, outputs, and what it must not do. Skills reference tools by command, never embed definitions that tools can provide.

#### 3.12.3 Per-job workflow (convert-job)
1. `mig parse` and read `spec.json` and parse report. Stop on `U-*` codes unless mode C is assigned.
2. `mig validate-control` for the job.
3. `mig baseline` (or load existing baseline); record proposed parity decisions.
4. `mig generate --what pipeline,tests`; read the coverage matrix.
5. For mode B items, compose patterns; record each composition in the report.
6. `mig gates --upto G6`; read failures; fix only inside the job branch; repeat within the attempt budget.
7. `mig generate --what orchestration,iac` for the workflow (once per workflow).
8. Open a PR with the conversion report; hand off to the reviewer agent.
9. After deployment to test: `mig compare`; triage; update decisions through PRs.

#### 3.12.4 Safeguards (summary)
Closed-world inputs (specification, catalogs, templates); an identifier whitelist built from the specification (any table, column or function in generated code must come from it); a prohibition on network calls except the factory CLI and approved AWS test endpoints; attempt budget (default 5 fix loops per job) and token/cost budget per job, after which the job escalates to a human; every agent action logged with command, arguments and result hash.

### 3.13 Traceability and provenance

`mig trace` writes one row per Informatica object (mapping, transformation, session property, workflow task, link, parameter) to `migration/reports/trace_matrix.csv`.

| Column | Meaning |
|---|---|
| object_type, object_name | Informatica object |
| job_id | Owning job |
| aws_artifact | File and symbol, state, or resource |
| conversion_mode | A, B or C |
| catalog_ref | Pattern or function id |
| decisions | Parity decision ids that apply |
| tests | Test ids covering it |
| evidence | Gate report reference |
| spec_hash, tool_versions | Provenance |

Every generated function has a comment `# source: <mapping>.<transformation>.<port>` so reviewers can navigate both ways.

### 3.14 Factory security and data handling
- **Secret filter:** a redaction layer on all tool output matches known secret patterns and values pulled from Secrets Manager; matches are replaced before output reaches an agent.
- **Data classes:** specifications and control files contain no data. Baselines contain data (masked or synthetic) and live in encrypted S3 buckets with access limited to the test account; production data is never read by agents.
- **Agent permissions:** agents use a role that can read the dev/test account only, cannot assume production roles, and cannot read Secrets Manager values (connections are used by Glue at run time, not by the agent).
- **Sandboxing:** file access limited to the repository; no outbound network except approved endpoints; package installs through a pinned internal index.
- **Prompt content:** no credentials, no personal data; sample rows shown to agents are limited to masked or synthetic baselines.
- **Auditability:** agent command logs retained for the audit period; pull requests are the record of change.

---

## 4. Runtime Design (Run-Time Plane)

### 4.1 Account and environment topology
| Account | Purpose | Deploy path | Data |
|---|---|---|---|
| Dev | Build and smoke run | CI on merge | Synthetic |
| Test | Reconciliation, parallel run, performance | CI on merge, manual promote | Masked or frozen snapshots |
| Prod | Production | CI with approval | Production |
| Shared tooling (optional) | CI runners, artifact bucket, catalogs | | None |

Resource naming: `mig-<env>-<job_id>-<name>` for jobs; `mig-<env>-wf-<workflow>` for state machines; buckets `<org>-mig-<env>-<purpose>`.

### 4.2 Glue job design
| Item | Design |
|---|---|
| Granularity | One Glue job per PowerCenter session (mapping); mapplets inlined |
| Entry script | One shared `entrypoint.py`; the pipeline module is selected by argument `--PIPELINE` |
| Libraries | Runtime library packaged as a wheel in S3 and passed through `--additional-python-modules` or `--extra-py-files` |
| Glue version and worker type | Per tier default: Simple G.1X x 2; Medium G.1X x 5; Complex G.2X x 10 with auto scaling; tuned from the parallel run |
| Timeout and retries | Timeout per tier (2x p95 legacy duration); `MaxRetries = 0` (Step Functions owns retry) |
| Concurrency | `MaxConcurrentRuns = 1` unless the job is partition-parameterized |
| Bookmarks | Disabled by default; incremental state is explicit (Section 4.6) so behavior matches the mapping variable semantics |
| Connections | Glue connection per database; network settings in the connection; credentials in Secrets Manager |
| Default arguments | `--enable-metrics`, `--enable-continuous-cloudwatch-log`, `--enable-job-insights`, `--enable-spark-ui`, `--TempDir`, `--RUN_ID`, `--BUSINESS_DATE`, `--ENV` |
| Tags | `job_id`, `workflow`, `wave`, `owner`, `env`, `cost_center` |

### 4.3 Runtime library (`glue/lib`)

#### 4.3.1 Modules
| Module | Responsibility |
|---|---|
| `args.py` | Typed argument parsing with defaults and required checks |
| `log.py` | JSON structured logger (`jlog`) with run id, job id, stage, row counts, seconds |
| `metrics.py` | CloudWatch metrics (`RowsRead`, `RowsLoaded`, `RowsRejected`, `DurationSeconds`, `Failed`) |
| `retry.py` | `with_retry` for transient errors, exponential backoff, no retry for non-idempotent operations |
| `io_jdbc.py` | JDBC read (hash partitions), batched write, staging and MERGE writer |
| `io_files.py` | Delimited, fixed-width and Parquet read/write with explicit schemas |
| `state.py` | Read and write of mapping variable equivalents in the state store |
| `runlog.py` | Writes run records and row counts to the run log |
| `rejects.py` | Reject capture and write with reason codes |
| `contracts.py` | `Pipeline` protocol and schema checks |
| `secrets.py` | Connection details through the Glue connection; never logs values |

#### 4.3.2 Pipeline module contract
```python
# glue/pipelines/<job>.py  (generated)
PIPELINE = {
    "job_id": "J003",
    "source":  {...},                    # type, table/path, keys, incremental settings
    "targets": [ {...}, ... ],           # type, table/path, load method, keys
    "state":   [ {"name": "LAST_RUN_DATE", "aggregation": "max", "column": "order_date"} ],
    "parity":  {"update_strategy": "staging_merge", "null_vs_empty": "preserve"},
}

def extract(ctx, cfg) -> DataFrame: ...
def transform(df, ctx) -> dict[str, DataFrame]:   # target name -> frame; pure, no I/O
    ...
def load(frames, ctx, cfg) -> dict[str, int]: ...   # returns rows per target
```
`transform` is pure so it can be unit-tested locally with small DataFrames; all I/O is in `extract` and `load`.

#### 4.3.3 Entrypoint flow
```
parse args -> start run record -> load state -> extract -> transform
 -> persist + single count -> load each target (retry only if idempotent)
 -> write rejects -> update state (only after all targets succeed)
 -> emit metrics -> finish run record -> job.commit()
on failure: structured error log -> metric Failed=1 -> run record FAILED -> raise (no commit, state unchanged)
```

#### 4.3.4 Logging schema
JSON line per event: `ts`, `level`, `run_id`, `job_id`, `stage` (`start`, `extracted`, `transformed`, `loaded`, `rejected`, `state_updated`, `error`, `end`), `target`, `rows`, `seconds`, `message`, and `error_class` when relevant. No row data and no secrets in logs.

#### 4.3.5 Retry policy
| Error class | Action |
|---|---|
| Transient JDBC/network (connection reset, deadlock victim, timeout) | Retry up to `MAX_RETRIES` (default 3) with 5, 10, 20 second backoff, only for idempotent steps |
| Constraint violation, data error | No retry; reject handling per decision or fail |
| Non-idempotent step (plain append) | No library retry; Step Functions retries the whole job only if the load is idempotent (see 4.3.6) |
| Glue service errors | Handled by Step Functions `Retry` |

#### 4.3.6 Idempotency rules
| Load type | Rerun safety mechanism |
|---|---|
| Staging + MERGE | Staging truncated at start; MERGE keyed on business key; safe to rerun |
| Delete + insert by key in one transaction | Safe to rerun |
| Overwrite (truncate and load) | Safe to rerun; table is unavailable during load, so use a staging table and swap when consumers require it |
| Plain append | Not safe; add a run-id column and delete-by-run-id before insert, or convert to MERGE |
| Parquet | Write to `run_id=` prefix then promote, or overwrite partition dynamically |

State updates occur only after every target has succeeded, so a failed run reruns from the same state.

### 4.4 SQL Server integration
| Concern | Design |
|---|---|
| Read | Glue DynamicFrame or Spark JDBC with `hashfield`/`hashpartitions` (or numeric bounds) and `fetchsize`; pushdown predicates for incremental filters |
| Write (append/overwrite) | Spark JDBC `batchsize` (default 10,000), `numPartitions` (default 4) to limit concurrent connections; `truncate=true` for overwrite to preserve table definition |
| Upsert | Write to a staging table, then execute a stored `MERGE` statement through a short JDBC call; use `WITH (HOLDLOCK)`; keys and update columns from the specification |
| Transactions | Spark commits per partition; the staging+MERGE pattern gives a single final transactional step |
| Identity and keys | Per `surrogate_key` decision; identity columns excluded from insert lists where required |
| Data types | Explicit casts for decimal, `datetime2`, `nvarchar` length; mapping table in Section 5.2 |
| Collation | Compared in baseline; mismatches recorded as `ENVIRONMENT` differences and handled by `string_compare` |
| Connections | Glue connection with VPC subnet and security group; TLS required; secret rotation supported through Secrets Manager |
| Limits | Connection count budgeted per job and per workflow (parallel branches multiply); documented in the plan |
| Bulk load option | Optional `bcp`/bulk copy path for very large loads through a reviewed pattern (mode C) |

### 4.5 S3 and Parquet design
| Concern | Design |
|---|---|
| Buckets | Separate buckets or prefixes for `landing`, `curated` (Parquet outputs), `rejects`, `staging`, `state`, `golden` (test account only) |
| Layout | `s3://<bucket>/<dataset>/[partition=value/]` with `run_id` prefix for staged writes |
| Format | Parquet, snappy; explicit schema; decimal types preserved; timestamps in UTC unless the decision says otherwise |
| Naming | Per `parquet_naming` decision |
| Partitioning | Per `parquet_partitioning`; dynamic partition overwrite for idempotent reruns |
| Small files | Coalesce to a target file size (default 128 to 256 MB) before write |
| Lifecycle | Staging and rejects expire per retention; versioning on curated and golden |
| Encryption | SSE-KMS with per-environment keys; bucket policies deny non-TLS access |
| Catalog | Glue Data Catalog tables for Athena/QA access, created by IaC |

### 4.6 State, parameters and secrets
| Item | Store | Notes |
|---|---|---|
| Mapping variable values (e.g. `$$LAST_RUN_DATE`) | DynamoDB table `mig-<env>-state` (pk `job_id`, sk `variable`) | Conditional write with version to avoid lost updates |
| Static parameters | SSM Parameter Store `/mig/<env>/<job_id>/<name>` | Read at start; override by job argument |
| Runtime parameters | Step Functions input and Glue arguments | `BUSINESS_DATE`, `RUN_ID` |
| Connection credentials | Secrets Manager via Glue connection | Never in code, arguments or logs |
| Run log | DynamoDB table `mig-<env>-runlog` (pk `workflow`, sk `run_id#job_id`) | Status, start/end, rows, error summary; TTL per retention |

State semantics reproduce Informatica's mapping variable behavior: the value is read at start, the new value (for example max of a column over processed rows) is written only after success, and aggregation type follows the specification (`max`, `min`, `count`).

### 4.7 Orchestration runtime
- **State machines:** generated per Section 3.9. Standard workflow type (needed for `.sync` and long durations).
- **Input contract:** `{"business_date": "YYYY-MM-DD", "overrides": {...}}`. The scheduler supplies it from the schedule context; manual runs supply it explicitly.
- **Execution naming:** `<workflow>-<business_date>-<attempt>` for duplicate protection and traceability.
- **Run log integration:** DynamoDB `PutItem` through the service integration at start, per job and at end.
- **History size:** stay below the execution history event limit (about 25,000 events) by nesting for long chains and by using `Map` with concurrency limits rather than unrolled loops.
- **Notifications:** SNS topic per environment; message includes workflow, job, run id, error, and a console link.
- **Events:** EventBridge rules on Glue job state change and Step Functions status change feed dashboards and optional downstream triggers.

### 4.8 Observability
| Signal | Source | Use |
|---|---|---|
| Structured logs | Glue continuous logging to CloudWatch Logs group per job | Search and metric filters |
| Metrics | `GlueETL` namespace (custom) plus Glue metrics | Dashboards and alarms |
| Run log | DynamoDB | Status board, SLA tracking, reconciliation history |
| Traces of orchestration | Step Functions execution history | Troubleshooting, redrive |
| Spark UI | Glue Spark history in S3 | Performance tuning |

**Alarms (per workflow):** execution failed; execution duration above p95 x 1.5; job `Failed` metric; zero rows loaded where non-zero expected; SLA breach (workflow not finished by deadline); DynamoDB throttling; Glue concurrency exceeded.
**Dashboards:** workflow status, duration trend vs legacy, rows per run, failure rate, cost per run.

### 4.9 Security
| Area | Design |
|---|---|
| IAM | One role per Glue job (read its sources and secrets, write its targets, put metrics, read/write own state keys); one role per state machine (start its jobs, write run log, publish SNS); CI deploy roles per environment; no wildcard resources |
| Encryption | KMS CMKs per environment for S3, DynamoDB, Secrets Manager, CloudWatch Logs |
| Network | Glue jobs in private subnets; VPC endpoints for S3, DynamoDB, Secrets Manager, CloudWatch, STS; private connectivity to SQL Server (VPN or Direct Connect); security groups limited to database ports |
| Data protection | Masking in non-production; PII columns tagged in the specification and excluded from logs |
| Audit | CloudTrail on all accounts; Config rules for public buckets and unencrypted resources |
| Secrets | Rotation schedule; no secrets in Terraform state (use references) |
| Supply chain | Pinned library versions; scanned wheels in a private index; image/runtime baselines reviewed per Glue version |

### 4.10 CI/CD and infrastructure as code

#### 4.10.1 Pipeline stages
```
PR opened -> G1..G6 (local Spark in CI container)
merge     -> package wheel + upload artifacts -> terraform plan/apply (dev) -> smoke run
promote   -> test: deploy -> G7 reconciliation -> G8 parallel run
approve   -> prod: deploy (schedule DISABLED) -> validation checks
cutover   -> enable schedule (change ticket)
```

#### 4.10.2 Terraform structure
| Module | Creates |
|---|---|
| `glue_job` | Job, role, log group, alarms, tags |
| `state_machine` | State machine, role, log group, alarms |
| `scheduler` | EventBridge schedule or rule with target role |
| `data_platform` | Buckets, KMS keys, DynamoDB tables, SNS topics |
| `network` | Glue connections, security groups, endpoints (references existing VPC) |
| `workflow/<name>` | Generated composition of the above for one workflow |

State is stored in a remote backend with locking per environment. Generated Terraform passes `terraform validate`, `tflint` and policy checks (for example no wildcard IAM) as part of G3.

---

## 5. Conversion Design Details

### 5.1 Transformation conversion rules
| PowerCenter | Spark/AWS implementation | Mode | Key notes |
|---|---|---|---|
| Source Qualifier (filter, SQL override) | JDBC read with pushdown predicate; SQL override as a vetted query | A (B if override) | Override text is stored and tested; no rewriting by agent |
| Expression | Sequential `withColumn` in port order | A | Catalog functions only |
| Filter | `filter` | A | Null condition drops row; matches Informatica |
| Router | One filtered frame per group, default group = negation of all | A | Rows can match multiple groups |
| Joiner | `join` with type; master/detail semantics | A | Null keys do not match (Informatica joiner matches nulls only if configured; verify) |
| Lookup (connected) | Left join with deterministic multi-match policy | A | `lookup_multi_match`, `lookup_no_match` decisions |
| Lookup (unconnected) | Join on the call site's key, or scalar subquery | B | Used inside expressions |
| Aggregator | `groupBy.agg`; sorted-input flag ignored | A | Empty input and null group keys tested |
| Sorter | `orderBy` only where order matters downstream | A | Spark order is not preserved after joins |
| Rank | Window `row_number`/`rank`/`dense_rank` | A | Tie policy from decision |
| Sequence Generator | Per `surrogate_key` decision | B | Gap and restart semantics are decisions |
| Update Strategy | Flag column and routing to merge/delete/insert | A | `update_strategy` decision |
| Union | `unionByName` with allowMissing as per ports | A | |
| Normalizer, Transaction Control, Custom, Java, XML | Not auto-converted | C | Engineer-led |
| Stored Procedure | Pattern if side-effect free; otherwise call from a pre/post step | B/C | |
| Mapplet | Inlined with origin tag | A | |
| Target (relational) | Writer per `update_strategy` | A | |
| Target (flat file) | Parquet writer or delimited writer | A | Naming per decision |
| Pre/post SQL | Pre/post step hooks with the statement stored verbatim | B | Reviewed |

### 5.2 Data type mapping
| Informatica (SQL Server source) | Spark type | SQL Server target type | Note |
|---|---|---|---|
| `string(n)` / `nstring(n)` | `StringType` | `varchar(n)` / `nvarchar(n)` | Length checked at write |
| `integer`, `small integer`, `bigint` | `IntegerType`, `ShortType`, `LongType` | `int`, `smallint`, `bigint` | |
| `decimal(p,s)` | `DecimalType(p,s)` | `decimal(p,s)` | High-precision mode decision; explicit casts |
| `double` | `DoubleType` | `float` | Tolerance in comparison |
| `date/time` | `TimestampType` | `datetime2(n)` | Precision decision; Informatica default precision is milliseconds |
| `binary` | `BinaryType` | `varbinary` | |

### 5.3 Flat file handling
| Concern | Design |
|---|---|
| Read | Explicit schema from the specification; mode `PERMISSIVE` with `_corrupt_record` captured to rejects; BOM and encoding per file definition |
| Fixed width | Substring extraction by layout from the specification |
| Header/trailer | Handled by dedicated steps; trailer counts validated |
| Delimited write | Explicit delimiter, quote, null token, line ending, encoding; single file only when the decision requires it |
| File arrival | Landing prefix; EventBridge rule on object created; completion marker file where the source system supports it |
| Archive | Move processed files to an archive prefix after success |

---

## 6. Test Design

### 6.1 Test levels
| Level | Scope | Runs in | Gate |
|---|---|---|---|
| Static | Lint, types, SQL lint, IaC validation, secret scan | CI | G3 |
| Expression | Catalog golden tests; per-job expression validation | CI | G4 |
| Unit | Transform functions on small DataFrames; state logic; reject logic | CI (local Spark) | G5 |
| Golden master | Pipeline on frozen inputs vs legacy outputs | CI (local Spark; sampled) | G6 |
| Integration | Glue job in test account against test SQL Server and S3; orchestration smoke | Test account | G7 |
| Reconciliation | Full-volume comparison vs baseline | Test account | G7 |
| Parallel run | Old vs new cycles | Test account | G8 |
| Non-functional | Performance, concurrency, failure injection, security checks | Test account | G9 evidence |

### 6.2 Test generation
Unit tests are generated from the specification: one test per transformation with input rows from the baseline edge-case set and expected rows from the golden output where available, otherwise from reviewed hand-written expectations. Tests carry the job id and decision ids they cover.

### 6.3 Test data
| Set | Source | Use |
|---|---|---|
| Masked sample | Production sample with masking/tokenization (deterministic tokens keep join integrity) | Golden master, reconciliation |
| Synthetic edge set | Spec-driven generator plus Hypothesis strategies | Unit, golden master |
| Volume set | Scaled copy of masked sample | Performance |
| Failure set | Malformed, duplicate and late data | Reject and restart tests |

### 6.4 Local test environment
`pytest` with a local `SparkSession` (Glue-compatible Spark and Python versions in the CI container). SQL Server is replaced by a containerized SQL Server or by Parquet fixtures through the same reader/writer interfaces. MERGE and T-SQL behavior is tested against the container, not simulated.

### 6.5 Profiling and statistics
The profiler writes the statistics listed in strategy Section 6.13 as Parquet (`profile.parquet`) and JSON (`run_stats.json`): volume, column profile, equivalence hashes, distribution, rejects, files, performance, config and state. The same profiler runs on the new output so the comparison is like for like.

### 6.6 Failure injection
| Scenario | Method | Expected |
|---|---|---|
| Source unavailable | Block network from job | Retry then fail; alert fires; state unchanged |
| Target constraint violation | Insert conflicting row | Reject or fail per decision |
| Job killed mid-load | Stop run during write | Rerun yields identical final state |
| Duplicate trigger | Start same business date twice | Second execution rejected |
| Upstream late | Delay predecessor | Successor waits; SLA alarm if breached |
| Concurrency limit | Start many workflows | `ConcurrentRunsExceeded` retried |

### 6.7 Performance tests
Run at production volume in the test account; record duration, DPU/worker hours, throughput and database load; compare with legacy p50/p95; tune workers, partitions and batch size; set timeout and alarm thresholds from results.

### 6.8 Regression suite against drift
A fixed set of pilot jobs and catalog golden tests runs on every change to templates, catalogs, runtime library, factory code and prompts/skills. Output of a pinned job set must be byte-identical (rule-based output) or pass equivalence tests (agent-composed output). A failure blocks the change.

---

## 7. Non-Functional Design

| Attribute | Target (default; confirm with Q-list) | Design response |
|---|---|---|
| Correctness | Zero unexplained differences at G7/G8 | Baselines, comparator, decisions |
| Performance | Workflow finishes within legacy p95 or agreed window | Partitioned reads, batching, tuning, critical-path scheduling |
| Availability of batch window | Meets SLA each cycle | Alarms, redrive, retries |
| Recoverability | Restart from failure without manual data repair | Idempotent loads, state-after-success |
| Scalability | 2x current volume without redesign | Worker scaling, partitioning |
| Security | No secrets in code; least privilege; encryption at rest and in transit | Section 4.9 |
| Auditability | Every change and run traceable | Git, trace matrix, run log, CloudTrail |
| Maintainability | Generated code readable; one pattern per construct | Templates, overrides policy |
| Cost | Cost per run tracked against legacy allocation | Tags, dashboards, worker right-sizing |
| Portability of design | Tool choices replaceable (CI, IaC) | Interfaces in Section 2.5 |

---

## 8. Operations Design

### 8.1 Runbooks (per workflow)
Contents: purpose and schedule; dependency diagram; inputs and outputs; parameters; normal duration; alarms and owners; restart procedure (redrive, rerun from state, manual run with overrides); data repair procedures; contact list.

### 8.2 Monitoring and support
| Item | Design |
|---|---|
| Status board | Dashboard from run log: today's executions by workflow and state |
| On-call | SNS to the on-call channel; severity by workflow criticality |
| Incident flow | Alert, triage using logs and execution history, redrive or fix, post-incident note |
| Change control | All changes through PR and pipeline; emergency path documented |

### 8.3 Hypercare
Defined window after cutover with daily reconciliation against the legacy output for the retained period, a fast rollback path (Section 8.4), and daily review of alarms and durations.

### 8.4 Rollback
Legacy workflows remain runnable (schedule disabled) until decommission. Rollback = disable EventBridge schedule, enable the Informatica schedule, restore target state from the last reconciled snapshot where the new run changed targets; state store values are reset to the last verified values.

### 8.5 Decommission
After the retention window and sign-off: archive PowerCenter repository export and logs, retire connections and licenses, remove legacy schedules, record in the trace matrix.

---

## 9. Delivery Plan for the Build

### 9.1 Build order
| Phase | Deliverables | Exit |
|---|---|---|
| 0 Foundations | Repo layout, CI skeleton, accounts, network, data platform module | Dev deploy works |
| 1 Parser and control | `mig parse`, `validate-control`, `dag_plan`, spec schema, G1/G2 | Pilot XML parsed with full coverage |
| 2 Runtime library and one pipeline | `glue/lib`, entrypoint, one hand-built pilot job on AWS | Pilot runs end to end |
| 3 Generators | Expression translator, catalogs, templates, generator, G3 to G6 | Pilot regenerated from specification matches hand-built job |
| 4 Orchestration and IaC | State machine and Terraform generators, run log, alarms | Pilot workflow deployed from generated code |
| 5 Baseline and comparator | `baseline`, comparator, results store, G7 | Pilot baseline and reconciliation clean |
| 6 Agents and skills | Skills, rules, safeguards, orchestrator workflow | Agents convert a second workflow with human review only |
| 7 Parallel run and cutover tooling | `parallel-run`, report, runbook generator, G8/G9 | Pilot cutover rehearsed |

### 9.2 Pilot selection
One workflow of each shape: independent jobs, a fan-in, a flat-file source, a SQL Server upsert, an incremental job with a mapping variable, and one mode B transformation. The pilot workflow should include at least one known legacy quirk to prove the decision flow.

### 9.3 Definition of done (per job)
All gates through G9 passed for the current specification hash; trace rows written; runbook present; alarms deployed; parity decisions approved (none provisional that affect output); owner sign-off recorded.

### 9.4 Human review checklist (G9)
1. Conversion report read; every mode B/C item reviewed.
2. Parity decisions approved by the data owner.
3. Reconciliation and parallel-run results reviewed, differences classified.
4. Security review items closed (roles, secrets, network).
5. Runbook and alarms verified.
6. Rollback tested or confirmed.
7. Cutover window agreed.

---

## 10. Design Decisions (ADR Log)

| ADR | Decision | Reason | Alternatives rejected |
|---|---|---|---|
| ADR-001 | Deterministic parser and generators; agents only compose, test, triage | Minimizes hallucination, gives reproducibility | Agent reads XML directly; agent writes all code |
| ADR-002 | One Glue job per session | Maps to legacy unit, simple restart and monitoring | One job for the whole workflow; one per transformation |
| ADR-003 | One state machine per workflow; nested only for long chains or worklets | Clear ownership, avoids history limit, matches legacy workflow | Single global state machine |
| ADR-004 | Step Functions owns retry; Glue `MaxRetries = 0` | One place for retry policy, visible in history | Glue-level retries |
| ADR-005 | Explicit state store instead of Glue bookmarks | Reproduces mapping variable semantics; supports rerun and manual override | Job bookmarks |
| ADR-006 | Staging table plus MERGE for upserts | Atomic final step, idempotent, supports Update Strategy | Row-by-row updates; delete/insert for all |
| ADR-007 | Expressions translated to Spark SQL strings | Snapshot-testable, parseable, diff-friendly | Generated Column API code |
| ADR-008 | Golden-master baseline before conversion | Gives benchmark and evidence for decisions | Test only after conversion |
| ADR-009 | Parity decisions recorded in CSV and read by generators | Explicit, auditable, no silent behavior change | Decisions embedded in code |
| ADR-010 | Gates run in order and are invalidated by specification hash change | Prevents stale approvals | Manual tracking |
| ADR-011 | Terraform for IaC (Q18 may change) | Team standard assumption | CDK, CloudFormation |
| ADR-012 | Agents have no production access and no secret values | Limits blast radius | Shared roles |

---

## 11. Risks to the Design and Mitigations
| ID | Risk | Mitigation |
|---|---|---|
| T-01 | Export XML variants differ from assumed element names | Parser built and tested on real exports from the pilot; schema versioning; parse completeness gate |
| T-02 | Function semantics differ in edge cases | Catalog entries start unverified; edge-case baselines flip them to verified |
| T-03 | Legacy cannot be run on test data | Fallback comparison (3.10.5); reduced-confidence flag |
| T-04 | SQL Server connection saturation by parallel branches | Per-workflow connection budget; `numPartitions` limits; `MaxConcurrency` on Map |
| T-05 | Glue cold start and cost for many small jobs | Group tiny jobs by decision (Simple tier option), use auto scaling, G.1X minimum |
| T-06 | Agent drift after prompt or skill change | Regression suite (6.8); versioned skills |
| T-07 | Hidden state in Informatica (persisted caches, variables) | Discovery of variables in parser; state store design; baseline of variable values |
| T-08 | Orchestration semantics gaps (e.g. complex decision tasks) | Escalate to `full_state_machine` with Choice; mode C for exotic cases |
| T-09 | Non-determinism in outputs hides defects | Determinism analysis; explicit normalization list approved by QA |
| T-10 | Throughput of human review limits wave speed | Reviewer agent first pass; reports designed for fast review; WIP limits |

---

## Appendix A. Worked Example Reference
The end-to-end example `wf_daily_sales` (jobs J001 customers, J002 products, J003 orders) in Section 19 of the strategy document illustrates: control CSVs, specification, plan, baseline statistics, transform code, MERGE, state machine, scheduler, tests, fix loop, reconciliation triage with decision D003, cutover and traceability. The designs in this document are the implementation of that example.

### A.1 Pipeline configuration (J003)
```python
PIPELINE = {
  "name": "orders", "job_id": "J003",
  "source": {"type": "jdbc", "table": "sales.orders", "pk": "order_id",
             "incremental": {"column": "order_date", "state_key": "LAST_RUN_DATE"}},
  "targets": [
    {"name": "fact_orders", "type": "jdbc_merge", "table": "dw.fact_orders",
     "staging": "dw.stg_fact_orders", "keys": ["order_id"]},
    {"name": "high_value_orders", "type": "s3_parquet",
     "path": "s3://<bucket>/high_value_orders/", "partition_by": ["order_date"]}]
}
```

### A.2 MERGE statement (generated)
```sql
MERGE dw.fact_orders WITH (HOLDLOCK) AS t
USING dw.stg_fact_orders AS s
  ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET
  t.customer_id = s.customer_id, t.order_date = s.order_date, t.amount = s.amount
WHEN NOT MATCHED BY TARGET THEN
  INSERT (order_id, customer_id, order_date, amount)
  VALUES (s.order_id, s.customer_id, s.order_date, s.amount);
```

### A.3 Test sketch (J003)
```python
def test_high_value_router_boundary(spark, load_case):
    df = load_case("J003", "edge_cases")           # from baseline edge-case set
    out = transform(df, ctx)
    assert out["high_value_orders"].filter("amount <= 1000").count() == 0
    assert out["fact_orders"].count() == df.count()   # all rows reach fact target
```

## Appendix B. Open Items (carry from discovery)
| Ref | Item | Needed by |
|---|---|---|
| Q18 | CI and IaC standard | Phase 0 |
| Q19, Q23 | Agent platform constraints and data policy | Phase 0 |
| Q21 | XML vs CSV conflict rule | Phase 1 |
| Q22 | Legacy runnable on test data | Phase 5 |
| (verify) | PowerCenter version, export DTD, repository view names | Phase 1 |
| (verify) | Glue version and Spark/Python versions for CI parity | Phase 2 |
