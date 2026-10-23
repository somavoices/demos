# Informatica PowerCenter to AWS Data Integration Migration Strategy

| | |
|---|---|
| **Document status** | Draft for discussion |
| **Version** | 0.4 |
| **Date** | 2026-10-23 |
| **Owner** | [Name / Role] |
| **Target platform** | Amazon EventBridge, AWS Step Functions, AWS Glue (PySpark) |
| **Targets** | SQL Server, Parquet files on Amazon S3 |
| **Delivery model** | AI-driven migration factory, deterministic-guarded, human-approved |

> All durations, volumes and effort figures in this document are planning placeholders. They are calibrated during the Assessment and Pilot phases (Section 12) and must not be treated as commitments before then.

**Reading guide:** architecture views are in Sections 4.5 to 4.9; the factory and baseline capture are in Section 6; a complete worked example follows in Section 19.

---

## 1. Executive Summary

The organization runs its data integration estate on Informatica PowerCenter. This strategy defines how that estate is migrated to a cloud-native, code-based stack: **EventBridge** for scheduling and triggers, **Step Functions** for workflow orchestration, and **Glue (PySpark)** for data processing, writing to **SQL Server** and **Parquet on S3**.

**Recommended approach: an AI-driven migration factory, deterministic-guarded.**

- **AI agents operate the factory end to end.** They read the PowerCenter exports, understand each mapping, plan the target design, compose Glue code from approved templates and patterns, generate orchestration and infrastructure code, write and run tests, fix their own failures, run old-versus-new reconciliation, triage differences, and produce documentation. They work job by job and wave by wave.
- **Deterministic tools supply the facts and the checks.** XML parsing, the dependency graph, expression semantics and data comparison are done by scripts the agents call. Agents do not recall these from model memory.
- **Independent gates decide correctness.** Every output is verified against the legacy system's results, not against the agent's own judgment. Legacy behavior is captured as a golden-master baseline before conversion, so the agent has a concrete target.
- **Humans own decisions and approvals.** Parity rules, risk acceptance, code review of complex constructs and production sign-off stay with people.

**Why this approach**
- Manual rewrite of every mapping is slow, inconsistent and hard to audit.
- Lift-and-shift of PowerCenter keeps the licensing and infrastructure burden.
- A factory driven by agents scales the repetitive work (analysis, generation, testing, reconciliation, documentation) and leaves engineers to review and decide.
- Guardrails make errors detectable instead of relying on trust in the AI.

**Key success factors:** an accurate inventory, explicit decisions on data-semantics parity (Section 8), a trustworthy validation method (Sections 6.8 and 10), and disciplined wave planning (Section 12).

---

## 2. Objectives, Scope and Success Criteria

### 2.1 Objectives
1. Retire PowerCenter by the license end date (to be confirmed, see Q16).
2. Preserve business outputs: migrated jobs produce data equivalent to the Informatica baseline.
3. Establish a maintainable, observable, version-controlled platform.
4. Reduce run cost and operational effort versus the current state.
5. Build a repeatable, auditable AI-assisted migration capability that raises throughput without lowering quality.

### 2.2 In scope
- All PowerCenter mappings, sessions, workflows, worklets, mapplets, reusable transformations, parameter files and schedules.
- Sources: relational (including SQL Server) and flat files. Targets: SQL Server and Parquet on S3.
- Orchestration, scheduling, alerting, restart and recovery behavior.
- Build and operation of the migration factory (tools, templates, skills, agents, gates).
- Test, reconciliation, cutover, hypercare and PowerCenter decommissioning.

### 2.2.1 Out of scope (to be confirmed)
- Redesign of business logic or data models (the migration is like-for-like).
- Downstream consumers' changes, except where file naming or format changes (see Q7).
- Migration of non-PowerCenter Informatica products.

### 2.3 Success criteria (KPIs, targets set after pilot)
| KPI | Measure |
|---|---|
| Automation rate | % of jobs converted with no manual code edits after generation |
| First-pass gate rate | % of jobs passing all automated gates on the agent's first attempt |
| Agent iterations | Average fix-loop iterations per job |
| Human review effort | Review minutes per job, by tier |
| First-pass parity | % of jobs passing reconciliation on first run |
| Defect density | Defects per job found in test and parallel run |
| Defect escape | Defects found after cutover |
| Cost per migrated job | Compute and AI usage plus human effort |
| Run-time | Glue run time vs Informatica baseline |
| Run cost | Glue cost per run vs current run cost |
| Delivery | Jobs cut over per wave vs plan |
| Reliability | Successful scheduled runs / total, post cutover |

---

## 3. Current State Assessment

The baseline is built from the PowerCenter repository, not from interviews alone. Agents run the extraction tools and summarize and question the results; the facts come from the tools.

| Item | Source | Output |
|---|---|---|
| Object inventory | Repository export (`pmrep`), repository MX views | Counts of folders, mappings, sessions, workflows, worklets, mapplets |
| Transformation usage | Parsed XML | Frequency of each transformation type per mapping |
| Session properties | XML | Commit interval/type, target load type, row treatment, pre/post SQL, recovery, error thresholds |
| Dependencies | Workflow links, shared folders, reusable objects, `dependencies.csv` | Dependency graph between workflows and objects |
| Parameters and variables | Parameter files, persisted mapping variables | Parameter inventory; list of jobs with run-to-run state |
| Schedules | Informatica scheduler and any external scheduler | Schedule and calendar matrix |
| Volumes and runtimes | Session logs and run history | Rows per run, duration, batch window, SLA |
| Sources and targets | Connection objects | System list with owners, networking, credentials approach |

**Deliverables:** Inventory workbook, complexity scoring (Section 13), dependency map with critical path, wave candidate list.

---

## 4. Target State Architecture

### 4.1 Component mapping
| PowerCenter | AWS target |
|---|---|
| Mapping + Session | One Glue job (PySpark), 1:1 by default |
| Workflow | One Step Functions state machine (or direct schedule for single-session workflows, Section 9.2) |
| Worklet | Nested state machine |
| Workflow links and conditions | Choice, Parallel and Map states |
| Command / email task | Step Functions task integration, SNS |
| Informatica scheduler | EventBridge Scheduler or rule (time or S3 event) |
| Connections | Glue Connections and Secrets Manager |
| Parameter file / `$$PARAM` | Glue job arguments and SSM Parameter Store |
| Persisted mapping variable | State store (DynamoDB or Parameter Store) |
| Reject / bad files | Rejects path on S3 and CloudWatch alarm |

### 4.2 Logical flow
```
EventBridge Scheduler / S3 event rule
          |
          v
Step Functions state machine (one per PowerCenter workflow)
   |- Task: Glue job (startJobRun.sync), one per session
   |- Choice / Parallel / Map for links and conditions
   |- Retry / Catch -> SNS alert -> Fail state
   '- Run log -> DynamoDB (run id, status, row counts, timings)
          |
          v
AWS Glue (PySpark)
   sources: SQL Server / JDBC, S3 flat files
   targets: SQL Server (batched JDBC write or staging + MERGE), S3 Parquet

Cross-cutting: Secrets Manager, KMS, CloudWatch Logs + Metrics,
               IAM least privilege, VPC connectivity, CloudTrail
```

### 4.3 Design principles
- **Traceable:** every Informatica object maps to a named AWS artifact (traceability matrix).
- **Config over code:** connections, paths, parameters and sizes are job arguments or parameters, never hardcoded.
- **Uniform runtime:** shared library for logging, retries, metrics, JDBC batch writes; one pipeline module per mapping.
- **Idempotent and restartable:** every job can be re-run safely; write modes chosen accordingly.
- **Observable by default:** structured JSON logs, custom metrics, alarms.
- **Secure by default:** no secrets in code, encryption at rest and in transit.

### 4.4 Code repository layout
```
glue/
  lib/            runtime (logging, retry, metrics), sources, sinks
  pipelines/      one module per mapping: PIPELINE config + transform()
  jobs/           entry script
infra/            IaC for Glue jobs, state machines, schedules, IAM
tests/            unit tests and reconciliation checks
migration/
  control/        CSV control files (dependencies, inventory, parity decisions, connections, parameters)
  specs/          parsed specification per mapping/workflow (JSON)
  catalogs/       approved patterns and expression function catalog
  reports/        per-job conversion report and traceability matrix
```


### 4.5 Architecture view: migration factory (control plane)
```
+------------------------------------------------------------------------------------+
| HUMANS: architect | migration engineers | QA | business owners | platform/security    |
|   approve wave plan, parity decisions, pull requests, sign-off, cutover                |
+-----------------------------------------+------------------------------------------+
                                          | instructions / approvals / escalations
                                          v
+------------------------------------------------------------------------------------+
| AGENT LAYER (Cursor agents, driven by skills)                                       |
|   Orchestrator -> Job agents (parallel) -> Test & Reconciliation -> Reviewer -> Docs |
+-----------------------------------------+------------------------------------------+
                                          | call tools, commit to feature branches
                                          v
+------------------------------------------------------------------------------------+
| TOOLBELT (deterministic)                                                            |
|  pc_parse | control_validate | dag_plan | templates | expression + pattern catalogs   |
|  baseline_capture | parallel_run | gate_runner | trace_matrix                          |
+-------------------+--------------------------+-------------------------------------+
                    |                          |                      |
                    v                          v                      v
+-----------------------------+ +-----------------------------+ +-----------------------------+
| GIT REPOSITORY              | | LEGACY ZONE (non-prod)      | | AWS DEV / TEST ACCOUNTS     |
| control CSVs, specs, code,  | | PowerCenter repository      | | Glue, Step Functions,       |
| infra, reports, golden      | | export, pmcmd host, legacy  | | EventBridge, shadow targets,|
| manifests                   | | test sources and targets    | | S3 golden store, Athena     |
+-----------------------------+ +-----------------------------+ +-----------------------------+
```

### 4.6 Architecture view: runtime (production AWS)
```
                       +---------------------------------+
                       | EventBridge Scheduler / rules   |
                       | (time, S3 object created,       |
                       |  upstream completion event)     |
                       +----------------+----------------+
                                        |
                                        v
 +--------------------------------------------------------------------------------+
 | Step Functions: one state machine per workflow (example: wf_daily_sales)        |
 |   Start -> Parallel[ Glue: customers | Glue: products ] -> Glue: orders         |
 |   Retry on tasks, Catch -> SNS alert; run log -> DynamoDB                       |
 +---------+--------------------------------------------------+-------------------+
           | startJobRun.sync                                 | events, alerts
           v                                                  v
 +------------------------------------+           +----------------------------+
 | AWS Glue jobs (PySpark)            |           | SNS -> email / chat / ITSM |
 | one job per session, shared        |           | CloudWatch logs, metrics,  |
 | runtime library                    |           | alarms, dashboards         |
 +------+--------------------+--------+           +----------------------------+
        | read               | write
        v                    v
 +---------------+   +----------------------------+   +----------------------------+
 | SQL Server    |   | SQL Server targets         |   | S3                         |
 | sources       |   | batched write or           |   | Parquet outputs, rejects,  |
 | (JDBC)        |   | staging + MERGE            |   | state and golden artifacts |
 +---------------+   +----------------------------+   +----------------------------+

 Supporting services: Secrets Manager (connections), SSM Parameter Store / DynamoDB (parameters,
 state), KMS, IAM role per job, VPC endpoints and VPN/Direct Connect, CloudTrail
```

### 4.7 Architecture view: validation (old vs new)
```
                 frozen input snapshot (database extract or file set, hashed)
                      |                                      |
                      v                                      v
        +-----------------------------+       +--------------------------------+
        | LEGACY PATH                 |       | NEW PATH                       |
        | pmcmd startworkflow         |       | Step Functions start-execution |
        | PowerCenter (non-prod)      |       | Glue jobs                      |
        +--------------+--------------+       +----------------+---------------+
                       | writes                                | writes
                       v                                       v
          shadow target A (legacy output)           shadow target B (new output)
          + session logs and run stats              + run log and metrics
                       |                                       |
                       +-------------------+-------------------+
                                           v
                      +-------------------------------------------+
                      | COMPARATOR (PySpark + datacompy)          |
                      | counts, profiles, hashes, key-level diff  |
                      | tolerances from parity_decisions.csv      |
                      +---------------------+---------------------+
                                            v
        results store (S3 Parquet + Athena) -> report -> agent triage -> human sign-off

 Golden store: baselines captured once from the legacy path (Section 6.13) are reused by unit
 and golden-master tests in CI, without re-running PowerCenter.
```

### 4.8 Architecture view: environments and promotion
```
feature branch (agent work)
   |  pull request
   v
CI: gates G1-G6  (control and spec validation, lint, type check, IaC and state machine
   |              validation, secret scan, unit tests, golden-master tests on local Spark)
   v  merge
DEV account   deploy by IaC, smoke run
   v
TEST account  reconciliation (G7) and parallel run (G8) on frozen snapshots, shadow targets
   v  human review and sign-off (G9)
PROD account  deploy with schedule DISABLED, run validation checks
   v  cutover
enable EventBridge schedule, disable Informatica schedule, hypercare
```

### 4.9 Architecture view: inside one Glue job
```
job start
  -> read arguments (JOB_NAME, connections, parameters, BATCH_SIZE, partitions)
  -> load state (for example last_run_date) from the state store
  -> extract       JDBC hash-partitioned read, or S3 read with explicit schema
  -> transform     pipeline module transform(): generated from the specification
  -> persist and count once -> log "extracted" (rows, seconds)
  -> load          SQL Server batched write or staging + MERGE | Parquet write
  -> rejects       write rejected rows to S3 with reason codes
  -> update state  only after all targets succeed
  -> emit metrics (RowsLoaded, Failed, duration) -> job commit
on error: retry policy -> structured error log -> metric Failed -> state NOT updated -> job fails
         (Step Functions Catch -> SNS alert)
```

---

## 5. Migration Approach

### 5.1 Options considered
| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Lift-and-shift PowerCenter to cloud VMs | Fastest, low change | Keeps licensing, ops burden and monolith | Rejected |
| Move to Informatica cloud product | Familiar tooling | Ongoing license cost, still a conversion effort | Not aligned with objectives |
| Manual rewrite in Glue | Full control | Slow, inconsistent, high defect risk at scale | Only for the hardest jobs |
| Visual ETL tool re-creation | Low code | Poor version control and testing, limited reuse | Rejected |
| **AI-driven factory, deterministic-guarded** | Speed, consistency, traceability, repeatable validation, scales with agents | Needs upfront tooling investment and strong validation | **Recommended** |

### 5.2 Conversion rule set (defaults)
- 1 session = 1 Glue job; 1 workflow = 1 state machine (or direct schedule, Section 9.2).
- Consolidating small sessions into one Glue job is an exception, documented in the job record.
- Write modes: `overwrite` with truncate for full loads, `append` for incremental, staging + MERGE for Update Strategy.
- Constructs that cannot be converted with confidence are escalated to a human, never guessed.
- Execution order is preserved on first migration; order optimization follows after stabilization (Section 9.4).

---

## 6. AI-Driven Migration Factory

### 6.1 Operating principle: AI-driven, deterministic-guarded
AI agents operate the factory. Deterministic tools supply the facts and the checks. Humans own decisions and approvals.

1. **Facts come from tools, not from model memory.** Parsed job definitions, dependencies and expression semantics are produced by scripts and handed to the agents.
2. **Correctness is proven by independent checks** against the legacy system's output, not by the agent's own judgment.
3. **Humans decide and approve.** Parity rules, risk acceptance, review of complex code and production sign-off are human responsibilities.

### 6.2 Agent operating model
| Agent | Mission | What it does | Human checkpoint |
|---|---|---|---|
| **Orchestrator agent** (wave level) | Run a wave end to end | Loads control files and runs G1; builds the wave plan; assigns jobs to job agents in dependency order and in parallel within capacity limits; tracks a status board; escalates blockers; produces the wave report | Wave plan approved before start |
| **Job agent** (per mapping/workflow) | Convert one job to a reviewed, gate-passing pull request | Runs the parser and reads the specification; explains the mapping; chooses patterns (write mode, join strategy, orchestration tier); composes the Glue pipeline from templates and adapts it to the job; translates expressions; handles rare constructs under the pattern catalog; generates orchestration and IaC; writes tests; runs the gates and iterates until they pass or the attempt budget is reached; opens the pull request | Pull request review |
| **Test and reconciliation agent** | Prove equivalence | Generates edge-case test data from the specification; runs unit tests; runs the old-versus-new parallel run; compares outputs; triages differences, groups them and proposes likely causes (verified by queries); files defects | Sign-off on results |
| **Reviewer agent** | Independent second look | In a separate context, compares code to the specification and pattern catalog; checks like-for-like fidelity and flags deviations. Additional check only, never the sole gate | Human reviewer sees its findings |
| **Documentation agent** | Keep the record complete | Writes the conversion report, traceability entry, runbook and release notes | Reviewed with the pull request |

**Humans:** program lead (scope, priorities), architect (parity decisions, patterns, tier B approval), migration engineers (review, Tier C work, supervise agents), QA (validation strategy, sign-off evidence), business owners (UAT, sign-off), platform and security (access, policy).

### 6.3 Toolbelt (deterministic tools the agents call)
| Tool | Provides |
|---|---|
| `pc_parse` | PowerCenter XML to a canonical specification (JSON); inventory; complexity scoring |
| `control_validate` | CSV schema and referential checks; cross-check against XML links |
| `dag_plan` | Dependency graph, layers, critical path, serial/parallel plan, orchestration tier |
| Template library | Pipeline templates, state machine templates, IaC modules, shared runtime |
| Expression catalog | Informatica-to-Spark function mappings with golden tests |
| Pattern catalog | Approved patterns (for example Update Strategy to staging + MERGE) |
| `gate_runner` | Executes gates G1 to G6 and reports results to the agent |
| `baseline_capture` | Runs legacy workflows on frozen inputs with `pmcmd`; stores golden outputs, statistics and a hashed manifest (Section 6.13) |
| `parallel_run` | Snapshot, run, compare and report |
| `trace_matrix` | Traceability between Informatica objects and AWS artifacts |

Agents also help build and extend this toolbelt during the foundation phase; new catalog entries proposed by agents are reviewed before use.

### 6.4 Control files (single source of truth)
Version-controlled in `migration/control/`, schema-validated, changed only through pull request. Agents read them and never edit them.

| File | Columns | Purpose |
|---|---|---|
| `dependencies.csv` | workflow, predecessor, successor, condition (SUCCESS / FAILURE / EXPRESSION), expression, trigger (SCHEDULE / EVENT / FILE), schedule | Execution order, serial vs parallel, triggers |
| `job_inventory.csv` | job_id, workflow, session, mapping, source_type, target_type, tier, wave, owner | Scope, complexity tier, wave assignment |
| `parity_decisions.csv` | decision_id, scope (GLOBAL or job_id), rule, value, approved_by, date | Approved semantics (Section 8) applied during generation |
| `connections_map.csv` | informatica_connection, glue_connection, secret_ref, environment | Connection mapping per environment |
| `parameters.csv` | job_id, parameter, source (JOB_ARG / SSM / STATE_STORE), default, environment | Parameter and state handling |

**Validation rules (gate G1):**
- Every job in `dependencies.csv` exists in `job_inventory.csv`.
- The dependency graph has no cycles.
- Every workflow link found in the XML appears in `dependencies.csv`, or is reported as a discrepancy (two independent sources cross-check each other).
- Every parity decision referenced by a job exists and is approved.
- Every Informatica connection used by a job is mapped.

### 6.5 Skills as agent playbooks (Cursor)
Skills package the procedure, conventions, templates and tool usage so every agent and every developer produces the same kind of output.

| Skill | Used by | Playbook |
|---|---|---|
| `informatica-migration` | Orchestrator, job agents | Master procedure: step order, conventions, gates, definition of done |
| `pc-parse` | Job agent | Run the parser, read and explain the specification, flag unparsed items |
| `dag-plan` | Orchestrator | Reconcile CSV and XML, build the graph, choose serial/parallel and the orchestration tier |
| `etl-generate` | Job agent | Compose pipelines from templates and patterns (single and multi-ETL through pipeline config entries); expression translation |
| `orchestration-generate` | Job agent | Generate state machines, schedules and Terraform by tier |
| `etl-test-generate` | Test agent | Unit tests, fixtures and edge cases from the specification |
| `baseline-capture` | Test agent | Prepare inputs, run legacy, repeat for determinism, store golden outputs and statistics |
| `parallel-run` | Test agent | Snapshot, run old and new, compare, triage, report |
| `migration-review` | Reviewer agent | Independent comparison of code against specification and catalogs |
| `migration-report` | Documentation agent | Conversion report, traceability matrix, runbook |

### 6.6 Per-job agent workflow
```
1. Export        PowerCenter XML + parameter file
2. Parse         pc_parse -> specification (G2)
3. Plan          dag_plan + control files -> tier, patterns, orchestration pattern (G1)
4. Baseline      run legacy on frozen inputs -> golden outputs and statistics (Section 6.13)
5. Generate      pipeline code, tests, orchestration, IaC (G3, G4, G5)
6. Test          run unit and golden-master tests; read failures; fix; repeat (G6, G7)
7. Parallel run  agreed cycles (G8)
8. Review        reviewer agent findings + human pull request review (G9)
9. Report        conversion report, traceability entry, runbook
```
Steps 5 and 6 run as a loop, and the golden outputs from step 4 give the agent a concrete target before it writes code. The agent stops when all gates pass or when the attempt budget (set in the pilot) is exhausted, then escalates with a summary of what was tried.

### 6.7 Wave-level operation
1. Orchestrator selects the wave from `job_inventory.csv` and runs G1.
2. `dag_plan` orders jobs: upstream before downstream, independent jobs in parallel.
3. Job agents run in parallel within limits set by review capacity, environment capacity and tool or plan limits.
4. A status board shows each job's stage, gate results, iteration count and tier.
5. Human review queue is ordered by tier and risk: Tier B and C first, Tier A sampled.
6. Blocked jobs escalate with the reason and options.
7. Wave exit: all jobs through G9, wave report issued, cutover readiness reviewed (Section 14).

### 6.8 Verification gates
| Gate | Check | On failure |
|---|---|---|
| G1 | Control files: schema, referential integrity, XML cross-check | Block; discrepancy report |
| G2 | Specification: schema valid; parse coverage 100% or unparsed items flagged | Block |
| G3 | Generation coverage: every transformation and port maps to generated code or a flag; every identifier in generated code exists in the specification | Agent fixes; block merge |
| G4 | Static: lint, type check, Spark SQL parse of expressions, IaC and state machine validation, secret scan | Agent fixes; block merge |
| G5 | Determinism: regeneration from the same specification and tool versions gives equivalent output for rule-based parts | Block |
| G6 | Unit tests pass | Agent fixes; block merge |
| G7 | Reconciliation of the Glue output against the golden-master baseline and the legacy output, within tolerances from `parity_decisions.csv` | Block cutover |
| G8 | Parallel run: agreed number of consecutive passing cycles | Block cutover |
| G9 | Human review and business sign-off | Block cutover |

### 6.9 Hallucination and bias controls
- **Closed world:** agents work from the parsed specification, the control files and the catalogs. Every column, function and parameter in generated code must exist there (identifier whitelist, G3).
- **Pattern catalog:** non-template logic must cite an approved pattern ID. No match means escalate to a human.
- **Like-for-like rule:** simplifying or "improving" legacy logic is a defect. Bias toward common patterns is caught by specification checks and reconciliation.
- **Decisions are data:** parity rules come from `parity_decisions.csv`, not from model judgment.
- **Independent oracle:** correctness is proven by comparison with legacy output. The reviewer agent is an extra check, not the sole gate.
- **Provenance and audit:** each artifact records the specification hash, tool, template and catalog versions, model and prompt versions, tier and gate results.
- **Drift control:** pin tool and model versions per wave; keep a regression suite of reference specifications with expected outputs and re-run it before any change to tools, prompts, templates or model.
- **Least privilege:** agents write only to feature branches and non-production environments; read-only on control files; no production access; no automatic deployment.
- **Data protection:** exports are scrubbed of credentials and sensitive values before agent use (Q19).
- **Budgets:** attempt, time and cost limits per job; breaches escalate.

### 6.10 Conversion modes and review levels
The AI participates in all modes. The mode sets the risk and the review depth.

| Mode | How the work is done | Review |
|---|---|---|
| **A: Template** | Agent applies templates, rules and the expression catalog; no new logic invented | Automated gates; sampled human review |
| **B: Pattern-constrained** | Agent composes code from the pattern catalog for constructs without a template (for example Update Strategy, Sequence Generator, stored procedure calls, T-SQL translation) | Full human review; extra tests |
| **C: Engineer-led** | Engineer owns the design; agent drafts and assists (dynamic lookup, Transaction Control, Java/custom/XML transformations) | Reviewed as new code |

The conversion report shows the mode of every construct, and the B and C share is a KPI (Section 2.3).

### 6.11 Automated parallel run (old vs new)
Run by the test and reconciliation agent using the `parallel_run` tool.
1. Freeze an input snapshot (database extract or file set).
2. Run the PowerCenter workflow on the snapshot with `pmcmd` in a non-production environment, writing to a shadow target.
3. Run the Glue workflow through Step Functions on the same snapshot, writing to a separate shadow target.
4. Compare: row counts, column aggregates and hashes, key-level differences, null and distinct counts, using tolerances from `parity_decisions.csv`.
5. Classify differences (expected by decision, defect, data issue); the agent proposes causes and verifies them with queries.
6. Repeat for the agreed number of cycles; archive reports as sign-off evidence.

If the legacy workflow cannot be re-run on a frozen snapshot, compare against its production output for the same input window. Production targets are never written during parallel run.

### 6.12 Where the productivity comes from
- Inventory and dependency analysis in minutes instead of days.
- Bulk conversion of simple and medium jobs by parallel agents.
- The generate, test, fix loop without manual round-trips.
- Test and fixture generation, including edge cases.
- Reconciliation triage, usually the slowest step of a migration.
- Documentation and traceability produced with every job.
- Faster build of the factory tooling itself.

Measured per mode (A, B, C) in the pilot: automation rate, first-pass gate rate, iterations, human review minutes, defects by gate and cost per job. Results calibrate the estimation model (Section 13) and show where further automation pays off.


### 6.13 Baseline capture (golden master)

**Purpose.** Capture legacy behavior before conversion, so the agent has a concrete target and the team has evidence for parity decisions. These are characterization tests, not classic unit tests: inputs are frozen, the legacy job is run, and its outputs and statistics are stored as the benchmark.

**Procedure** (test agent with `baseline_capture`; baselines for the next wave are captured while the current wave converts):
1. Prepare two input sets: a masked production-like sample, and a synthetic edge-case set generated from the specification.
2. Freeze and hash the inputs.
3. Run the legacy workflow in non-production with `pmcmd`, using a baseline copy whose connections and parameter file point to the frozen inputs and shadow targets.
4. Repeat the run several times to establish determinism and variance; list non-deterministic columns for exclusion or normalization.
5. Normalize and store golden outputs (Parquet), rejects and statistics with a manifest of hashes.
6. Record observed behavior that settles open parity questions (nulls, rounding, lookup policy) as proposed entries for `parity_decisions.csv`, for human approval.
7. Hand the baseline to the Glue tests and to gate G7.

If the legacy workflow cannot be run on test data, compare against its production output for a defined input window (Q22).

**Statistics to capture (per job, per run)**
| Category | Statistics |
|---|---|
| Volume | Source rows read; per-transformation rows in, out and rejected (session log and repository run-stat views such as `REP_SESS_LOG` and `REP_SESS_TBL_LOG`; verify per version); target rows inserted, updated, deleted, rejected |
| Column profile (output) | Null count, empty-string count, distinct count, min/max, sum/avg, string length min/max, date min/max, decimal scale |
| Equivalence | Row-level hash by key, column aggregate hashes, key uniqueness and duplicate count, orphan keys |
| Distribution | Histograms for numeric and date columns; top-N frequencies for low-cardinality columns |
| Rejects | Count, reason codes, sample rows |
| Files | Row count, size, header/trailer, delimiter, encoding, line endings, fixed-width layout, naming |
| Performance | Duration (total and per session), rows per second, reader/transformation/writer busy % from thread statistics, cache sizes, commit count and interval, p50 and p95 over the last 30 to 90 days |
| Reliability | Failure rate and reasons, reruns, rows-per-run trend, batch window and SLA |
| Config and state | Session properties, parameter values per run, mapping variable values before and after each run |
| Timing | Actual start and end times (for the critical path), wait gaps |
| Source quality | Null and duplicate rates, late or missing arrivals |
| Cost | Infrastructure and license cost allocated per job |

**Areas to cover**
| Area | Cases |
|---|---|
| Transformations | Every type used; expression functions; null and empty handling; decimal rounding; dates and time zones; join match, no-match and null keys; lookup single, multiple and no match; aggregation of empty input and null group keys; Router groups; rank ties; Update Strategy insert, update, delete and reject; parameters and mapping variables (first run, incremental, rerun) |
| Input data | Empty, single row, large volume, duplicate keys, malformed rows, wrong column count, encoding and BOM, unicode, very long strings, numeric extremes, schema drift, late, missing or duplicate files |
| Load behavior | Full vs incremental, restart after partial failure, idempotent rerun, commit interval, constraint violations, locks, truncate and load |
| Orchestration | Link conditions, failure branches, decision tasks, timers, worklets, parameter passing, calendars, concurrent runs, alerts, restart from a failed task |
| Environment parity | Code page, SQL Server collation and case sensitivity, trailing-space handling, `datetime` vs `datetime2` precision, time zone |
| Non-functional | Performance at volume, concurrency limits, credentials and permissions, logs and metrics present |

**Tech stack**
| Purpose | Option | Recommendation |
|---|---|---|
| Run and extract legacy | `pmcmd`, `pmrep`, repository MX views, session-log parsing | Python scripts on a PowerCenter client host |
| Test runner | pytest | Yes |
| Compute and comparison | PySpark (same engine as Glue), `datacompy`, DuckDB for fast local checks | PySpark plus `datacompy` |
| SQL Server side | `pyodbc` or JDBC; `HASHBYTES` and `CHECKSUM_AGG` for in-database aggregates | Compute hashes in the database for large tables |
| Data quality rules | Great Expectations, Soda Core, AWS Glue Data Quality (DQDL, Deequ-based) | Glue Data Quality for ongoing monitoring; equivalence stays in the comparator |
| Test data | Faker, Hypothesis (property-based), spec-driven generator, masking or tokenization | Spec-driven generator and masked samples |
| Storage | Git for specs and manifests; versioned S3 for golden Parquet and profiles | Manifest with checksums per baseline |
| Results and reporting | Parquet/JSON in S3 queried with Athena; QuickSight or equivalent | Yes |
| CI and orchestration | Step Functions plus Glue for the new side; the team's CI tool (Q18) for triggering | Yes |

**Practical rules**
- **Determinism first:** run the legacy job several times on the same input. Columns that differ between runs (audit timestamps, sequences, ordering) go on an exclusion or normalization list, and the run-to-run variance sets the tolerances.
- **Depth by tier:** all jobs get volume, column profile, hashes, rejects and duration. Medium and complex jobs add per-transformation counts, the edge-case matrix, variable state and commit behavior. Critical or slow jobs add thread and cache detail, histograms and repeated-run variance.
- **Legacy defects:** the baseline will contain them. Each is logged in `parity_decisions.csv` as "replicate" or "fix", so a mismatch is always intentional.
- **Artifact layout per job:** `inputs/` (with hashes), `golden_output.parquet`, `profile.parquet`, `rejects.parquet`, `run_stats.json`, `manifest.json`.

---

## 7. Transformation Conversion Catalogue

Conversion mode: **A** template, **B** pattern-constrained agent work, **C** engineer-led with agent assistance (Section 6.10).

| PowerCenter construct | Target pattern | Mode | Risk |
|---|---|---|---|
| Source Qualifier (filter, SQL override) | Pushdown query or DataFrame filter | A; B for T-SQL | Medium if T-SQL specific |
| Expression | `withColumn` expressions through the function catalog; variable ports evaluated in order | A; unknown functions proposed as new catalog entries (reviewed) | Medium (evaluation order, types) |
| Filter | `filter` | A | Low |
| Router | Multiple filters, multiple writes | A | Low |
| Joiner | DataFrame join; null-key behavior per parity decision | A | Medium |
| Lookup (connected, cached) | Broadcast or standard join; match policy applied | A | Medium |
| Lookup (unconnected, dynamic cache) | Redesign | C | High |
| Aggregator | `groupBy` / window; sorted-input assumptions checked | A | Medium |
| Sorter | `orderBy` only where order matters downstream | A | Low |
| Rank | Window functions | A | Low |
| Union | `unionByName` | A | Low |
| Normalizer | `explode` | A | Medium |
| Sequence Generator | Surrogate key strategy per decision in Section 8 | B | High |
| Update Strategy | Staging table + MERGE with insert/update/delete flags | B | High |
| Stored Procedure transformation | Post-load step or reimplementation | B | High |
| Transaction Control | Redesign | C | High |
| Java / Custom / XML transformations | Rewrite | C | High |
| Mapplet | Reusable function | A; B if nested | Low to Medium |
| Pre/post-session SQL | Pre/post steps in job or state machine | A; B for T-SQL | Medium |

---

## 8. Data Semantics and Parity Decisions

These decisions are made by people, recorded in `parity_decisions.csv`, and applied by the agents and templates uniformly.

| Topic | Decision needed |
|---|---|
| Commit interval | Informatica commit interval/type vs Glue: Spark commits per write partition. Define whether exact parity is required or batched inserts are acceptable (Q2). |
| Update Strategy | Row flags (insert/update/delete/reject) mapped to staging + MERGE; delete handling policy (Q6). |
| Incremental loads | Mapping variables (e.g. last run date) need an explicit state store; Glue bookmarks are not a drop-in replacement (Q5). |
| Nulls and empty strings | Per-source rule for null vs empty string handling. |
| Decimal and numeric | Precision, scale and rounding; session high-precision setting captured per job. |
| Dates and timestamps | Time zone, precision and type mapping between Informatica, SQL Server and Parquet. |
| Lookup policy | First, last or error on multiple matches; null-key match behavior. |
| Surrogate keys | Source of keys (database identity/sequence vs generated); stability across runs. |
| Rejected rows | Rejects path, format, retention; failure thresholds. |
| Target load order | Constraint-based and target load order reproduced in orchestration. |
| Parquet output | File naming, partitioning, compression, file count; rename step if a fixed file name is required (Q7). |
| SQL Server write | Bulk vs normal mode, batch size, staging pattern (Q10). |

---

## 9. Orchestration, Scheduling and Execution Order

### 9.1 Terms
**Step Functions** is the AWS service. A **state machine** is one workflow defined in it. An **execution** is one run of a state machine. Step Functions is used as the single orchestration tool and holds as many state machines as there are workflows that need one.

### 9.2 Orchestration tiers (chosen by `dag_plan` from the parsed workflow)
| Workflow shape | Orchestration |
|---|---|
| Single session, no branching or downstream steps | EventBridge Scheduler starts the Glue job directly; Glue job retries (`MaxRetries`) cover transient failures |
| Linear chain of sessions | Small state machine: sequence, retry, failure alert |
| Parallel branches, conditions, worklets, event waits | Full state machine |

A shared EventBridge rule on Glue job failures sends alerts to SNS for every job, so alerting stays uniform across all tiers.

### 9.3 Execution order and dependencies
- **Within a workflow:** sequential steps use `startJobRun.sync`, which waits for the Glue job to finish before the next state. Independent steps become Parallel or Map states.
- **One state machine per workflow** is the default, which keeps 1:1 traceability, isolates failures and respects different schedules and owners. A single state machine for everything is not used.
- **Cross-workflow dependencies:** event-based hand-off (EventBridge rule on the upstream execution or job completion) or an optional parent state machine that starts child state machines in order and waits. Use the parent pattern only where a strict end-to-end chain is needed.
- **Long chains:** Standard executions have a history limit of about 25,000 events and each Glue task uses several, so very long chains are split into nested child state machines per stage or subject area.
- **Restart:** a failed Standard execution can be redriven from the failed state; the run log in DynamoDB supports restart decisions; writes are idempotent (see write-mode rules in Section 5.2).
- **Merging back-to-back workflows** into one chain is an optimization after stabilization, not a first-migration default.

### 9.4 Order and critical-path optimization
**Stage 1 (migration):** preserve the existing execution order so parity is easy to prove.
**Stage 2 (after stabilization):** optimize using the dependency graph.

| Optimization | Effect |
|---|---|
| Replace clock-time chains with event-driven triggers (completion events, S3 arrival) | Removes idle gaps between workflows |
| Run independent workflows and tasks in parallel | Shortens the batch window |
| Size workers by critical path (larger on the critical path, smaller off it) | Faster batch for similar cost |
| Merge many tiny sequential jobs with the same source | Avoids repeated Glue start-up time |
| Throttle concurrency (`MaxConcurrency`, Glue concurrent runs) | Protects SQL Server connections and quotas |
| Read a shared source once into S3 when several workflows use it | Fewer source reads |

The graph combines explicit workflow links, `dependencies.csv` and implicit dependencies inferred from table lineage (workflow A writes a table that workflow B reads), checked against run history for actual start and end times. Hidden dependencies are the main risk of parallelizing; verify with lineage and the parallel run, and test the added load on SQL Server.

### 9.5 Operational concerns
| Concern | Approach |
|---|---|
| Triggers | EventBridge Scheduler for time-based; EventBridge rule on S3 event for file arrival |
| Conditions (link conditions, decision tasks) | Choice states |
| Failure handling | Retry with backoff on transient errors; Catch routes to SNS and a failure state |
| Restart and recovery | Redrive; run log in DynamoDB; idempotent writes |
| Calendars and holidays | Reproduced via schedule rules or a calendar check step |
| External schedulers | If an enterprise scheduler triggers workflows today, decide: keep it calling Step Functions, or replace with EventBridge (Q11) |
| Alerting | SNS to email/chat/ITSM; CloudWatch alarms on failure and on duration anomalies |

---

## 10. Testing and Validation Strategy

| Level | Purpose | Method |
|---|---|---|
| Unit | Transformation logic correct | pytest with local Spark and generated datasets (agent-generated, spec-driven) |
| Baseline (golden master) | Legacy behavior captured before conversion | Frozen inputs run through PowerCenter; outputs and statistics stored as the benchmark (Section 6.13) |
| Component | Source read and target write work | Dev environment against test systems |
| Reconciliation | Output equals Informatica baseline | Row counts, column aggregates and checksums, key-based sample diff, null and distinct counts |
| Parallel run | Equivalence over real cycles | Run both systems on the same input for an agreed number of cycles and compare outputs (Section 6.11) |
| Performance | Meets batch window and SLA | Production-like volumes; worker sizing tuned |
| Failure and recovery | Retry, restart, alerting work | Fault injection (source unavailable, bad data, partial failure) |
| Factory regression | Tools, templates and agents behave consistently | Reference specifications with expected outputs, re-run on every tool, prompt or model change |
| UAT | Business sign-off | Business-owned acceptance on key outputs |

**Exit criteria per job:** all gates passed, reconciliation with no unexplained differences, agreed number of successful parallel cycles, performance within window, alerts verified, documentation complete.

**Test data:** masked or synthetic where production data is restricted (Q13).

---

## 11. Non-Functional Requirements

- **Performance:** Glue worker type and count sized per job from baseline volumes; parallel read and write partitions tuned to database capacity.
- **Cost (FinOps):** right-size workers, avoid idle capacity, tag every resource by job and wave, report cost per run against Informatica baseline; track AI usage cost per job.
- **Observability:** structured JSON logs, custom metrics (rows loaded, failures, duration), dashboards, run-log table.
- **Security:** IAM least privilege per job, credentials in Glue Connections or Secrets Manager, KMS encryption, private connectivity (VPC; VPN/Direct Connect for on-premises sources), CloudTrail auditing.
- **Environments:** separate dev, test and production accounts or equivalent isolation; promotion through CI/CD only.
- **Availability and recovery:** documented recovery steps per workflow; infrastructure reproducible from code.

---

## 12. Roadmap and Waves

Durations are indicative and finalized after Assessment.

| Phase | Activities | Key outputs |
|---|---|---|
| 0. Mobilize (2 to 4 weeks) | Governance, team, environments, access, tooling, AI usage policy and approvals | Charter, RACI, landing zone, approved AI tooling |
| 1. Assess (3 to 6 weeks) | Run inventory tools; control files; complexity scoring; dependency map; decisions in Section 8; baseline capture design; wave plan | Inventory workbook, control files, decision log, wave plan |
| 2. Foundation and Pilot (4 to 8 weeks) | Build the toolbelt, templates, catalogs, skills, agent configuration and gate runner; migrate 5 to 10 representative jobs across all tiers | Working factory, calibrated effort, automation rate and review effort |
| 3. Waves (repeat) | Per wave: orchestrator agent plans, job agents convert in parallel, test agent reconciles, humans review, parallel run; baselines for the next wave are captured while the current wave converts | Migrated and validated workflows |
| 4. Cutover and Decommission | Final cutover, hypercare, retention, PowerCenter shutdown | Decommission certificate, lessons learned |

**Pilot selection:** include at least one job of each complexity tier, one with Update Strategy, one with mapping variables, one flat-file source and one Parquet target.

**Wave planning rules:** group by subject area and dependency; migrate upstream before downstream; balance complexity within a wave; size waves to human review capacity; keep a stabilization buffer before cutover.

---

## 13. Complexity Tiers and Estimation

| Tier | Typical characteristics |
|---|---|
| Simple | Single source and target; expressions and filters; no lookups; no state |
| Medium | Joins, lookups, aggregations, parameters, multiple targets |
| Complex | Update Strategy with deletes, mapping variables or state, dynamic lookups, T-SQL overrides, stored procedure/Java/XML transformations, nested mapplets, transaction control |

**Estimation model:** effort per job = base effort by tier x adjustment factors (volume, number of targets, open parity decisions) + human review effort by conversion mode + validation effort. Base values are calibrated in the pilot, with and without agents, and then applied to the full inventory.

---

## 14. Cutover, Rollback and Decommissioning

1. **Pre-cutover gate:** G1 to G9 passed, runbooks and alerts in place, business approval.
2. **Cutover:** disable the Informatica schedule for the workflow, enable the EventBridge schedule, confirm first production run.
3. **Rollback:** Informatica remains available and unchanged until decommission; rollback is re-enabling its schedule. Rollback criteria are defined per wave.
4. **Hypercare:** monitored operation with the delivery team on call for an agreed period after each wave.
5. **Decommission:** after the retention period and final sign-off, archive repository exports and run history, shut down servers, close the license.

---

## 15. Governance and Operating Model

### 15.1 RACI (summary)
| Activity | Program lead | Architect | Migration engineers | QA | Business owner | Platform / Security |
|---|---|---|---|---|---|---|
| Strategy and decisions | A | R | C | C | C | C |
| Inventory and wave plan | A | R | R | C | C | I |
| Factory build (tools, catalogs, skills, agents) | C | A | R | C | I | C |
| Agent-run conversion (supervision and review) | I | A | R | C | I | I |
| Parity decisions | C | R | C | C | A | I |
| Reconciliation and sign-off | C | C | R | R | A | I |
| Cutover | A | R | R | R | C | R |
| Security, access and AI usage policy | I | C | I | I | I | A/R |

A = accountable, R = responsible, C = consulted, I = informed. Agents act under the responsibility of the migration engineers who supervise them.

### 15.2 Human checkpoints (never delegated to agents)
- Approval of the wave plan.
- Parity decisions and risk acceptance.
- Review and merge of pull requests (full review for modes B and C).
- Business sign-off and the cutover go/no-go.
- Any change to control files, catalogs, gates or agent permissions.

### 15.3 Cadence
- Weekly delivery review (progress, KPIs, risks, agent metrics).
- Per-wave go/no-go review.
- Decision log reviewed weekly until Section 8 decisions are closed.
- Monthly steering committee.

### 15.4 Change control
Scope or parity-decision changes go through the decision log; generated code changes only through pull requests; changes to tools, prompts, templates or model versions require a pass of the factory regression suite.

---

## 16. Risks and Mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Incomplete inventory (hidden schedulers, parameter files, shared objects) | Missed jobs, late surprises | Repository-based inventory plus scheduler review; reconcile against run history |
| R2 | Silent data differences (types, nulls, rounding) | Incorrect business data | Explicit parity decisions; reconciliation and parallel run |
| R3 | Complex constructs more frequent than expected | Effort and timeline overrun | Early complexity scoring; pilot calibration; engineer-led track |
| R4 | Mapping variable and state handling errors | Missed or duplicate loads | Dedicated state store pattern; targeted tests |
| R5 | SQL Server write performance or lock contention | Missed batch windows | Batch size and partition tuning; staging + MERGE; off-peak scheduling |
| R6 | Downstream consumers break on Parquet file naming or format | Integration failures | Consumer inventory; rename step or contract agreement |
| R7 | AI-generated code defects or hallucination | Incorrect logic | Facts from tools; closed-world inputs and identifier whitelist; independent gates (Section 6.8); legacy-output reconciliation; mandatory review for modes B and C |
| R8 | Sensitive data exposure through exports, prompts or test data | Compliance breach | Scrubbing, masking, approved AI tooling, least-privilege agents |
| R9 | License end date pressure | Rushed cutovers | Early timeline baseline; prioritize license-critical workflows |
| R10 | Skills gap in Spark/AWS | Slow reviews, quality issues | Training; pairing; shared library and standards |
| R11 | Parallel run requires double capacity | Cost and source load | Plan windows; limit duration; sample where acceptable |
| R12 | Model, tool or prompt change alters output mid-program | Inconsistent or regressed conversions | Pin versions per wave; factory regression suite before any change |
| R13 | Review bottleneck as agents produce faster than humans can review | Queue growth, rubber-stamping | Size waves to review capacity; tiered review depth; strong automated gates; review KPIs |
| R14 | Agent cost or run time exceeds plan | Budget overrun | Attempt, time and cost budgets per job; cost per job KPI; pilot calibration |
| R15 | Over-trust in the agent's explanations | Undetected errors | Decisions and sign-off stay human; evidence comes from reconciliation, not from agent statements |
| R16 | Legacy non-determinism or defects distort the benchmark | False mismatches or replicated bugs | Determinism baseline from repeated runs; exclusion and normalization list; each legacy defect logged in `parity_decisions.csv` as replicate or fix |
| R17 | Legacy cannot be run on test data in non-production | No golden master | Fall back to production output for a defined input window; prioritize environment access (Q22) |

---

## 17. Assumptions and Dependencies

- PowerCenter repository access (or bulk export) is available.
- AWS accounts, networking and access to source and target systems are provided on schedule.
- Business owners are available for parity decisions and UAT.
- Informatica remains operational until each wave's cutover is accepted.
- Cursor and the selected models are approved for use with the migration artifacts (Q19, Q23).
- Targets are SQL Server and Parquet on S3 only; other targets trigger a scope change.

---

## 18. Discovery Questionnaire and Open Decisions

| ID | Question | Why it matters | Owner | Status |
|---|---|---|---|---|
| Q1 | How many mappings, sessions and workflows? Rough split of simple, medium and complex? | Sizing, waves, effort | Informatica admin | Open |
| Q2 | Is exact commit-interval parity required, or is batched insert acceptable? | Write design, performance | Architect / business | Open |
| Q3 | Can Informatica and Glue run side by side on the same input for validation? | Validation method, cost | Program lead | Open |
| Q4 | Is bulk export access to the PowerCenter repository available (`pmrep`, MX views)? | Inventory and automation | Informatica admin | Open |
| Q5 | Which jobs use persisted mapping variables or run-to-run state? Preferred state store? | Incremental design | Architect | Open |
| Q6 | How widely is Update Strategy used, and how are deletes handled? | MERGE design | Architect / business | Open |
| Q7 | Parquet targets: consumers, naming, partitioning, compression, fixed file name needed? | Output contract | Data consumers | Open |
| Q8 | How much T-SQL and SQL override logic exists in Source Qualifiers and pre/post SQL? Push down or rewrite? | Conversion approach | Architect | Open |
| Q9 | How are rejected and bad rows handled today, and what should happen after migration? | Error handling design | Business / ops | Open |
| Q10 | SQL Server: versions, volumes, bulk-load permitted, network path (VPC, VPN, Direct Connect)? | Write pattern, connectivity | DBA / network | Open |
| Q11 | Current triggers: Informatica scheduler or external scheduler? Time-based or file-arrival? Calendars and SLAs? | Orchestration design | Operations | Open |
| Q12 | Environment model: number of AWS accounts, promotion path? | IaC and CI/CD | Platform | Open |
| Q13 | Compliance and data constraints: PII, masking, residency, encryption? | Security design, test data | Security / compliance | Open |
| Q14 | Source list: flat file locations, transfer methods (SFTP, shares), formats, encodings, layouts? | Ingestion design | Source owners | Open |
| Q15 | Batch windows and SLA per workflow? | Sizing, scheduling | Business / ops | Open |
| Q16 | PowerCenter license and support end date? | Timeline baseline | Procurement / program lead | Open |
| Q17 | Team skills and the target operating model (who runs and supports jobs)? | Training, handover | Program lead | Open |
| Q18 | Preferred IaC and CI/CD tooling (Terraform or CDK, pipeline tool)? | Infrastructure standards | Platform | Open |
| Q19 | Is Cursor, and the selected models, approved for use with PowerCenter exports and sample data? What must be scrubbed, and where is data processed? | Governance, compliance | Security | Open |
| Q20 | Monitoring and ITSM standards (alerting channels, ticketing)? | Observability design | Operations | Open |
| Q21 | Who provides and maintains the control CSVs (dependencies, inventory)? Which source wins if CSV and XML disagree? | Source of truth, gate G1 | Program lead / Informatica admin | Open |
| Q22 | Can PowerCenter be run in non-production on frozen input snapshots (`pmcmd`)? Are shadow targets available? | Parallel run design (Section 6.11) | Informatica admin / platform | Open |
| Q23 | Which agent capabilities and limits apply on the chosen plan (parallel or background agents, usage limits, budget)? | Wave throughput, cost | Platform / program lead | Open |
| Q24 | Reviewer capacity: how many engineers, and what turnaround is expected per pull request? | Wave sizing, review bottleneck | Program lead | Open |


---

## 19. Worked Example: One Workflow, End to End

> Illustrative only. Names, numbers and dates are invented to show how the factory behaves. Commands and options must be verified against your PowerCenter and AWS versions.

### 19.1 The legacy workflow

`wf_daily_sales` runs daily at 01:30 on the Informatica scheduler.

```
wf_daily_sales
  START
    |-- s_m_load_customers  (SQL Server dbo.customers -> dw.dim_customer, upsert)
    '-- s_m_load_products   (SQL Server dbo.products  -> dw.dim_product, truncate and load)
              \__________ both succeed __________/
                              |
                      s_m_load_orders   (mapping m_load_orders)
                              |
                  any task fails -> Email task
```

Mapping `m_load_orders`:
```
SQ_orders      SQL Server sales.orders, filter order_date >= $$LAST_RUN_DATE
  -> EXP_calc  amount_usd = amount * fx_rate ; status_clean = UPPER(LTRIM(RTRIM(status)))
  -> LKP_customer  dw.dim_customer on customer_id (multiple match: first)
  -> FIL_active    status_clean != 'CANCELLED'
  -> RTR_value
       group GRP_HIGH_VALUE (amount_usd >= 10000) -> flat file high_value_orders.csv
       group GRP_ALL        (TRUE)                -> UPD_flag (insert or update on order_id)
                                                  -> SQL Server dw.fact_orders
  mapping variable $$LAST_RUN_DATE updated with SetMaxVariable(order_date)
```
Session properties of interest: commit interval 10,000; target load type normal; row treatment data driven; high precision enabled.

### 19.2 Control files for this workflow

`job_inventory.csv`
```
job_id,workflow,session,mapping,source_type,target_type,tier,wave,owner
J001,wf_daily_sales,s_m_load_customers,m_load_customers,sqlserver,sqlserver,Medium,1,data-eng
J002,wf_daily_sales,s_m_load_products,m_load_products,sqlserver,sqlserver,Simple,1,data-eng
J003,wf_daily_sales,s_m_load_orders,m_load_orders,sqlserver,sqlserver+parquet,Complex,1,data-eng
```

`dependencies.csv`
```
workflow,predecessor,successor,condition,expression,trigger,schedule
wf_daily_sales,START,J001,SUCCESS,,SCHEDULE,cron(30 1 * * ? *)
wf_daily_sales,START,J002,SUCCESS,,SCHEDULE,cron(30 1 * * ? *)
wf_daily_sales,J001,J003,SUCCESS,,,
wf_daily_sales,J002,J003,SUCCESS,,,
```

`parity_decisions.csv`
```
decision_id,scope,rule,value,approved_by,date
D001,GLOBAL,null_vs_empty_string,empty string treated as NULL for SQL Server sources,architect,2026-11-02
D002,J003,lookup_multiple_match,first by customer_key ascending,architect,2026-11-02
D003,J003,decimal_arithmetic,decimal(28,10) for amount_usd; round half up at 2 dp on write,architect,2026-11-03
D004,J003,commit_interval,batched insert of 10000 rows; commit per partition accepted,business,2026-11-03
D005,J003,high_value_output,Parquet partitioned by order_date; fixed file name not required,data-consumer,2026-11-03
```

`connections_map.csv`
```
informatica_connection,glue_connection,secret_ref,environment
SQLSRV_SALES_SRC,glue-conn-sales-src,secrets/sales-src,dev
SQLSRV_DW_TGT,glue-conn-dw-tgt,secrets/dw-tgt,dev
```

`parameters.csv`
```
job_id,parameter,source,default,environment
J003,LAST_RUN_DATE,STATE_STORE,1900-01-01,all
J003,HIGH_VALUE_THRESHOLD,JOB_ARG,10000,all
```

### 19.3 End-to-end flow

| # | Actor | Action | Output or gate |
|---|---|---|---|
| 1 | Orchestrator agent | Loads control files, runs `control_validate`; cross-checks XML links against `dependencies.csv` | G1 passed |
| 2 | Human (architect) | Reviews and approves the wave plan | Wave 1 approved |
| 3 | Job agent | Runs `pc_parse` on the exported XML; reads and explains the specification | G2 passed; specification JSON |
| 4 | Job agent | Runs `dag_plan`: layers, critical path, orchestration tier | Plan (19.5) |
| 5 | Test agent | Runs `baseline_capture` for J001 to J003 with `pmcmd` on frozen inputs, three repeats | Golden outputs, statistics (19.6) |
| 6 | Job agent | Generates pipeline modules, Glue job definitions, state machine, schedule, tests | Code and IaC (19.7, 19.8) |
| 7 | Job agent | Runs the gates and iterates (see 19.9) | G3 to G6 passed |
| 8 | Test agent | Runs reconciliation in the test account on the sample set | G7 failed once (19.10) |
| 9 | Test agent | Groups the differences and proposes a cause, verified by a query | Triage note |
| 10 | Human (architect) | Approves the updated parity decision D003 | Decision recorded |
| 11 | Job agent | Regenerates; reruns tests and reconciliation | G7 passed |
| 12 | Test agent | Runs five consecutive daily parallel-run cycles on shadow targets | G8 passed |
| 13 | Reviewer agent and human engineer | Reviewer agent findings; human reviews the pull request, in full for mode B items | PR approved |
| 14 | Documentation agent | Conversion report, traceability matrix, runbook | Reports (19.12) |
| 15 | Human (business owner) | Signs off | G9 passed |
| 16 | Platform engineer and agents | Cutover: enable the EventBridge schedule, disable the Informatica schedule; hypercare | Production |

### 19.4 Specification excerpt (output of `pc_parse`, abbreviated)

```json
{
  "mapping": "m_load_orders",
  "job_id": "J003",
  "session": {"commit_interval": 10000, "target_load_type": "normal",
              "row_treatment": "data_driven", "high_precision": true},
  "variables": [{"name": "$$LAST_RUN_DATE", "type": "date", "aggregation": "SetMaxVariable"}],
  "sources": [{"name": "SQ_orders", "table": "sales.orders",
               "filter": "order_date >= $$LAST_RUN_DATE",
               "ports": ["order_id", "customer_id", "amount", "fx_rate", "status", "order_date"]}],
  "transformations": [
    {"name": "EXP_calc", "type": "Expression",
     "ports": [{"name": "amount_usd", "expr": "amount * fx_rate"},
               {"name": "status_clean", "expr": "UPPER(LTRIM(RTRIM(status)))"}]},
    {"name": "LKP_customer", "type": "Lookup", "table": "dw.dim_customer",
     "condition": "customer_id = IN_customer_id", "multiple_match": "first"},
    {"name": "FIL_active", "type": "Filter", "condition": "status_clean != 'CANCELLED'"},
    {"name": "RTR_value", "type": "Router",
     "groups": [{"name": "GRP_HIGH_VALUE", "condition": "amount_usd >= 10000"},
                {"name": "GRP_ALL", "condition": "TRUE"}]},
    {"name": "UPD_flag", "type": "UpdateStrategy", "expr": "IIF(ISNULL(order_key_in_target), DD_INSERT, DD_UPDATE)"}
  ],
  "targets": [{"name": "fact_orders", "table": "dw.fact_orders", "type": "sqlserver"},
              {"name": "high_value_orders", "file": "high_value_orders.csv", "type": "flatfile"}]
}
```
Every port and transformation in this file must appear in the generated code or be flagged (gate G3).

### 19.5 Plan (output of `dag_plan`)

```yaml
workflow: wf_daily_sales
layers:
  - [J001, J002]            # independent, run in parallel
  - [J003]                  # after both succeed
critical_path: [J001, J003] # from historical durations: 14 min + 22 min (illustrative)
orchestration_tier: full_state_machine   # parallel branch, sequence, failure alert
trigger: EventBridge Scheduler cron(30 1 * * ? *), Asia/Kolkata
modes: {J001: A+B (upsert), J002: A, J003: A+B (upsert, state variable)}
```

### 19.6 Baseline capture for J003 (illustrative)

Legacy run, on frozen inputs, against a baseline copy of the workflow (options to be verified for your version):
```
pmcmd startworkflow -sv INT_SVC -d DOMAIN -uv PC_USER -pv PC_PASS \
      -f SALES -paramfile /baseline/J003/run1.par -wait wf_daily_sales_baseline
```

Input sets:
- `sample_250k`: masked production-like sample, 250,000 orders.
- `edge_cases`: synthetic, generated from the specification:

| Case | Purpose |
|---|---|
| E1 `amount` exactly 10000.00 | Router boundary, inclusive or exclusive |
| E2 `amount` x `fx_rate` just below 10000 (9999.9999995) | Decimal precision at the boundary |
| E3 `status` NULL | Filter and Expression null behavior |
| E4 `status` ' cancelled ' with spaces | Trim and case handling |
| E5 `status` empty string | Null vs empty string (D001) |
| E6 two `dim_customer` rows for one `customer_id` | Lookup multiple-match policy (D002) |
| E7 `customer_id` not in `dim_customer` | Lookup no-match |
| E8 `order_date` equal to `$$LAST_RUN_DATE` | Incremental boundary |
| E9 `order_id` already in target and new `order_id` | Update and insert paths |

Captured statistics (illustrative):

| Statistic | Value |
|---|---|
| Source rows read | 250,000 |
| Rows out of FIL_active | 231,418 (18,582 filtered) |
| Router GRP_HIGH_VALUE / GRP_ALL | 4,106 / 231,418 |
| `dw.fact_orders` inserted / updated / rejected | 205,986 / 25,432 / 0 |
| Lookup no-match (NULL `customer_key`) | 312 |
| Duration, rows per second | 6 min 48 s, about 610 rows/s |
| Determinism (3 runs) | Identical except `load_ts`, excluded |

### 19.7 Generated code excerpts

`glue/pipelines/orders.py`
```python
from pyspark.sql import functions as F, Window

PIPELINE = {
    "name": "orders", "job_id": "J003",
    "source": {"type": "jdbc", "table": "sales.orders", "pk": "order_id",
               "incremental": {"column": "order_date", "state_key": "LAST_RUN_DATE"}},
    "targets": [
        {"name": "fact_orders", "type": "jdbc_merge", "table": "dw.fact_orders",
         "staging": "dw.stg_fact_orders", "keys": ["order_id"]},        # pattern P-UPSERT-01
        {"name": "high_value_orders", "type": "s3_parquet",
         "path": "s3://<bucket>/high_value_orders/", "partition_by": ["order_date"]},  # D005
    ],
}

def transform(orders, dim_customer, high_value_threshold):
    # EXP_calc (decimal handling per D003; D001 empty string -> NULL)
    df = (orders
          .withColumn("amount_usd", F.col("amount").cast("decimal(28,10)")
                                    * F.col("fx_rate").cast("decimal(28,10)"))
          .withColumn("status_clean", F.upper(F.trim(F.col("status"))))
          .withColumn("status_clean",
                      F.when(F.col("status_clean") == "", None).otherwise(F.col("status_clean"))))

    # LKP_customer (D002: first match by customer_key ascending)
    w = Window.partitionBy("customer_id").orderBy(F.col("customer_key").asc())
    lkp = (dim_customer.withColumn("rn", F.row_number().over(w))
                       .filter("rn = 1").select("customer_id", "customer_key"))
    df = df.join(F.broadcast(lkp), "customer_id", "left")

    # FIL_active (NULL status behavior confirmed by baseline case E3)
    df = df.filter(F.col("status_clean") != "CANCELLED")

    # RTR_value
    high = df.filter(F.col("amount_usd") >= F.lit(high_value_threshold))   # GRP_HIGH_VALUE
    return {"fact_orders": df, "high_value_orders": high}                   # GRP_ALL = all rows
```

Update Strategy pattern P-UPSERT-01 (staging load, then one MERGE in a single transaction):
```sql
MERGE dw.fact_orders WITH (HOLDLOCK) AS t
USING dw.stg_fact_orders AS s
   ON t.order_id = s.order_id
WHEN MATCHED THEN
  UPDATE SET t.customer_key = s.customer_key, t.amount_usd = s.amount_usd,
             t.status = s.status_clean, t.order_date = s.order_date,
             t.updated_at = SYSUTCDATETIME()
WHEN NOT MATCHED BY TARGET THEN
  INSERT (order_id, customer_key, amount_usd, status, order_date, updated_at)
  VALUES (s.order_id, s.customer_key, s.amount_usd, s.status_clean, s.order_date, SYSUTCDATETIME());
```

State handling for `$$LAST_RUN_DATE` (state store, updated only after all targets succeed):
```python
last_run = state.get("LAST_RUN_DATE", default="1900-01-01")   # validated as an ISO date
orders = read_jdbc(table="sales.orders", where=f"order_date >= '{last_run}'")
...
state.put("LAST_RUN_DATE", max_order_date)   # only after the MERGE and the Parquet write succeed
```

### 19.8 Generated orchestration

State machine `wf_daily_sales` (abbreviated):
```json
{
  "Comment": "wf_daily_sales, generated from the specification; do not edit by hand",
  "StartAt": "LoadDimensions",
  "States": {
    "LoadDimensions": {
      "Type": "Parallel",
      "Branches": [
        {"StartAt": "J001_customers",
         "States": {"J001_customers": {
           "Type": "Task", "Resource": "arn:aws:states:::glue:startJobRun.sync",
           "Parameters": {"JobName": "wf_daily_sales__J001_customers"},
           "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 2,
                      "IntervalSeconds": 60, "BackoffRate": 2}],
           "End": true}}},
        {"StartAt": "J002_products",
         "States": {"J002_products": {
           "Type": "Task", "Resource": "arn:aws:states:::glue:startJobRun.sync",
           "Parameters": {"JobName": "wf_daily_sales__J002_products"},
           "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 2,
                      "IntervalSeconds": 60, "BackoffRate": 2}],
           "End": true}}}
      ],
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyFailure"}],
      "Next": "J003_orders"
    },
    "J003_orders": {
      "Type": "Task", "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {"JobName": "wf_daily_sales__J003_orders"},
      "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 1}],
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyFailure"}],
      "End": true
    },
    "NotifyFailure": {
      "Type": "Task", "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {"TopicArn": "arn:aws:sns:<region>:<account>:etl-alerts",
                     "Message.$": "States.JsonToString($)"},
      "Next": "Failed"
    },
    "Failed": {"Type": "Fail"}
  }
}
```
Retries sit on the individual tasks, not on the Parallel state, so a retry does not rerun the branch that already succeeded. `J003_orders` retries once: both of its writes are idempotent (MERGE upsert and partition overwrite of the Parquet output), and the state variable advances only after success.

Schedule (created disabled, enabled at cutover):
```hcl
resource "aws_scheduler_schedule" "wf_daily_sales" {
  name                         = "wf-daily-sales"
  schedule_expression          = "cron(30 1 * * ? *)"
  schedule_expression_timezone = "Asia/Kolkata"
  state                        = "DISABLED"
  flexible_time_window { mode = "OFF" }
  target {
    arn      = aws_sfn_state_machine.wf_daily_sales.arn
    role_arn = aws_iam_role.scheduler_to_sfn.arn
  }
}
```

### 19.9 Tests and the agent fix loop

Unit and golden-master tests (pytest, local Spark):
```python
def test_high_value_boundary_inclusive(spark, make_orders, make_dim):
    orders = make_orders([("o1", 10000.00, 1.0), ("o2", 9999.99, 1.0)])
    out = transform(orders, make_dim(), 10000)
    assert [r.order_id for r in out["high_value_orders"].collect()] == ["o1"]

def test_matches_golden(spark, run_pipeline_local):
    new = run_pipeline_local("J003", inputs="baseline/J003/inputs/edge_cases")
    golden = spark.read.parquet("baseline/J003/golden_output.parquet")
    result = compare(new, golden, keys=["order_id"],
                     tolerances="migration/control/parity_decisions.csv",
                     exclude=["load_ts"])
    assert result.passed, result.summary()
```

Fix loop for J003 (illustrative):

| Iteration | Gate result | Agent action |
|---|---|---|
| 1 | G3 failed: generated code used `fxrate`, not in the specification | Corrected to `fx_rate` |
| 2 | G6 failed: golden test for E1 (amount exactly 10000) | Router comparison changed to inclusive per the specification |
| 3 | G3 to G6 passed | Pull request opened |

### 19.10 Reconciliation and triage (illustrative)

```
J003  fact_orders  keys=order_id  input=sample_250k
  row count            legacy 231,418   new 231,418   OK
  key diff             missing_in_new 0   extra_in_new 0   OK
  column amount_usd    mismatches 3 of 231,418   FAIL (tolerance 0)
  other columns (hash) OK
J003  high_value_orders  legacy 4,106   new 4,103   FAIL
```

Triage by the test agent:
1. All three mismatching keys have `amount_usd` within 0.000001 of 10000 and differ in the 7th to 10th decimal place.
2. Hypothesis: decimal multiplication in Spark can reduce the result scale, so the product differs slightly from the legacy high-precision result and falls on the other side of the threshold.
3. Verification: a query on the three keys, recomputing the product with operands cast to the source column precision, reproduces the legacy values.
4. Proposal: update decision D003 (operand precision and rounding rule). The architect approves; the agent regenerates; reconciliation passes with zero differences.

The agent proposed and verified. The human decided.

### 19.11 Parallel run and cutover

- **Parallel run:** five consecutive daily cycles on frozen snapshots with shadow targets, all comparisons passing (G8).
- **Pre-cutover checks:** G1 to G9 passed, runbook and alerts verified, business sign-off recorded.
- **Cutover (day 0):** disable the Informatica schedule, enable the EventBridge schedule, watch the first production run.
- **Rollback:** re-enable the Informatica schedule; PowerCenter remains untouched until decommission.
- **Hypercare:** monitored operation for the agreed period; then archive and decommission the workflow's PowerCenter objects.

### 19.12 Artifacts produced and traceability

Repository additions for this workflow:
```
migration/control/        job_inventory.csv, dependencies.csv, parity_decisions.csv, ...
migration/specs/          m_load_customers.json, m_load_products.json, m_load_orders.json
glue/pipelines/           customers.py, products.py, orders.py
infra/                    glue jobs, state machine wf_daily_sales, scheduler, IAM
tests/                    unit tests, golden-master tests per job
baseline/J001..J003/      inputs (hashed), golden_output.parquet, profile.parquet, run_stats.json, manifest.json
migration/reports/        conversion reports, traceability matrix, parallel-run reports, runbook
```

Traceability matrix (excerpt):

| Informatica object | AWS artifact | Mode | Parity decision |
|---|---|---|---|
| `wf_daily_sales` | State machine `wf_daily_sales`, scheduler `wf-daily-sales` | A | n/a |
| `s_m_load_orders` | Glue job `wf_daily_sales__J003_orders` | A | D004 |
| `EXP_calc` | `orders.py::transform` (expressions) | A | D001, D003 |
| `LKP_customer` | `orders.py::transform` (broadcast join, first match) | A | D002 |
| `RTR_value` | `orders.py::transform` (high-value branch), Parquet sink | A | D005 |
| `UPD_flag` + `dw.fact_orders` | Sink `jdbc_merge`, pattern P-UPSERT-01 | B | D004 |
| `$$LAST_RUN_DATE` | State store key `LAST_RUN_DATE` | B | n/a |

### 19.13 Who did what

| Humans | Agents (with tools) |
|---|---|
| Approved the wave plan | Validated control files and cross-checked them against the XML |
| Approved parity decisions D001 to D005, including the D003 update | Parsed XML, planned the graph and orchestration tier |
| Reviewed the pull request in full for mode B items | Captured baselines, generated code, orchestration, IaC and tests |
| Signed off and approved cutover | Ran the fix loop, reconciliation, parallel run and triage |
| | Produced the conversion report, traceability matrix and runbook |

---

## Appendix A: Definition of Done per Job

- [ ] Specification generated and stored; conversion report reviewed
- [ ] Baseline captured and stored (hashed inputs, golden outputs, statistics, manifest)
- [ ] Gates G1 to G6 passed (results attached)
- [ ] Glue pipeline code reviewed and merged through pull request
- [ ] Unit tests pass; coverage of all transformation logic
- [ ] Step Functions or schedule defined; failure alerts verified
- [ ] Parameters, connections and secrets configured per environment
- [ ] Reconciliation passes with no unexplained differences (G7)
- [ ] Parallel run cycles completed as agreed (G8)
- [ ] Performance within batch window
- [ ] Runbook and restart procedure documented
- [ ] Traceability matrix entry complete, with conversion mode and provenance
- [ ] Business sign-off recorded (G9)

## Appendix B: Deliverables by Phase

| Phase | Deliverables |
|---|---|
| Mobilize | Charter, RACI, environment readiness, AI usage policy and approvals |
| Assess | Inventory workbook, control files, complexity scoring, dependency map, decision log, wave plan |
| Foundation and Pilot | Toolbelt, templates, catalogs, skills, agent configuration, gate runner, regression suite, pilot results, calibrated estimates |
| Waves | Migrated code, tests, reconciliation reports, runbooks, traceability matrix, wave reports |
| Cutover and Decommission | Cutover reports, hypercare summary, decommission certificate, lessons learned |
