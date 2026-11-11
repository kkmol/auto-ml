# Build spec for the hackathon

Three people build Model Desk in parallel against six fixed contracts. Inside their own workstream, each person decides how to build. The contracts change only if all three agree.

Source: Claude Doc "Model Desk — platform vision and build spec", tab "Build spec (hackathon)" (https://claude.ai/artifact/FDiYtYaNnWNMyqP3d7AGd3). Two embedded diagrams (module map; integration checkpoints) are not exported; see the doc.

## Scope

The hackathon builds one end-to-end path for one problem type: yes/no classification on the SIPP transfer-out example, from the first message to a signed evaluation report. Nothing goes live.

| In | Out (later phases) |
| --- | --- |
| Request workspace and Home (Needs you, Working) | Live model page, Review queue, most of Settings |
| Stages Frame → Discover → Plan → Build → Evaluate → Deliver | Go-live, deployments, monitoring, control groups |
| Ask and Plan modes, cycled with Shift+Tab | Auto mode (stretch goal) |
| `/plan`, `/pause`, `/resume`, `/explain` | Custom and team commands, memory, notifications |
| Plan gate and DS sign-off gate | Confidential-data gate (no real confidential data used) |
| Harness v1 for binary classification | Regression, forecasting, clustering |
| Budget shown and counted | Budget enforcement and resource monitors |
| Runs on SPCS; caller's rights if the spike works | Fallback: a dedicated demo role, same contracts |

### Synthetic data

The hackathon uses a synthetic dataset, so no client data is needed. It is generated with known structure so the demo can prove the harness works:

- Five tables shaped like the real ontology: `CLIENT_DIM`, `SIPP_ACCOUNT_WEEKLY`, `TRANSFER_OUT_EVENTS`, `WEB_SESSIONS`, `CONTACT_INTERACTIONS`.
- About a 2% transfer rate per 90 days, with planted signal in logins, calls about charges and market falls.
- One **planted leak**: `transfer_pack_sent`, recorded a few days after the transfer request. The harness must catch it.
- Recent snapshots whose 90-day outcome hasn't happened yet, so the label maturity check has something to exclude.

If real data is approved later, only the data changes. The contracts stay the same.

### The demo it must run

1. A requester asks the SIPP question. The agent asks for horizon and call capacity, and a feasibility check passes.
2. Discovery streams: ontology search, then SQL profiling, each folding to one line. The planted leak is listed as excluded.
3. The plan card appears with label spec, split, success bar and budget. The requester cycles to Plan mode and back, then approves.
4. Round 1 trains and misses the bar on validation. The agent shows its diagnosis and a plan diff, then runs round 2.
5. The agent nominates a candidate. The harness scores the locked test set and the report opens in the right panel.
6. A second user with the DS reviewer role signs the report. The demo ends on the Snowflake objects it created: feature table, Registry version, report on stage.

## Team split

Each person owns one service and the contracts on its edges, so no two people edit the same code. The services talk only through the contracts below.

| Workstream | Builds | Implements | Consumes |
| --- | --- | --- | --- |
| **A · Interface** | Web app (React, shadcn/ui, AI Elements) and the API service (REST, event stream, sign-in, caller token handling) | C2 (serves it), C3 (calls it) | C1, C2 |
| **B · Agent** | Orchestrator service: run loop, prompts, Cortex calls, tools, plan writing, event emission | C3 (serves it), C4 | C1, C5, C6 |
| **C · Evaluation and data** | Harness service, synthetic data, all Snowflake objects, SPCS specs and deployment | C5 (serves it), C6 | C1 |

- **C1, shared types,** is owned jointly. Person B is the steward who merges changes.
- **Platform work goes to C.** That covers the compute pool, image repository, service specs, roles and grants. C's service is the smallest, and C needs those objects first anyway.
- **The caller-token spike is shared.** A proves the token arrives at ingress, B proves a query runs with it, and C sets the CALLER grants. They do it together in the first checkpoint.

## C1 · Shared types

These types are the vocabulary of all five other contracts. The source of truth is `contracts/schemas/*.json` (JSON Schema 2020-12). TypeScript types for A and Pydantic models for B and C are generated from it. Nobody hand-writes them.

**Conventions:**

- JSON keys are `snake_case`.
- Times are ISO 8601 in UTC.
- IDs are ULIDs with a prefix (`req_`, `evt_`, `gate_`, `out_`, `rep_`).
- Credits are numbers to 2 decimal places.
- Snowflake names are fully qualified and upper case.

```ts
type Stage = "framing" | "discovering" | "plan_ready" | "building" | "evaluating"
           | "candidate" | "signed" | "stopped" | "failed";
type Status = "working" | "waiting" | "paused" | "done" | "stopped" | "failed";
type Mode = "ask" | "auto" | "plan";
type Level = "plain" | "practitioner" | "expert";

interface Request {
  id: string; title: string; owner: string; role_used: string;
  stage: Stage; status: Status; mode: Mode; level: Level;
  plan_version: number | null; budget_credits: number; spent_credits: number;
  created_at: string; updated_at: string;
}

interface Brief {
  request_id: string; version: number; problem_type: "binary_classification";
  decision: string; population: string; horizon_days: number;
  capacity_per_week: number | null; exclusions: string[];
}

interface Plan {
  request_id: string; version: number; status: "draft" | "approved" | "superseded";
  label: { view: string; entity_key: string; snapshot_col: string; label_col: string;
           horizon_days: number; definition: string };
  population: { view: string; definition: string; exclusions: string[] };
  features: { table: string; columns: { name: string; group: string; description: string;
              source_tables: string[] }[] };
  excluded_fields: { field: string; reason: string }[];
  split: { train: [string, string]; validation: [string, string]; test: [string, string];
           embargo_days: number; group_by: string };
  metric: { primary: "precision_at_k" | "recall_at_k" | "pr_auc" | "roc_auc";
            k?: number; target: number; use_ci_lower_bound: boolean };
  baselines: ("random" | "logistic" | { model_ref: string })[];
  fairness: { attribute: string; metric: "fpr_ratio" | "tpr_ratio"; max_ratio: number }[];
  approach: string;
  budget: { credits: number; max_rounds: number; forecast_per_round: number };
  summary_plain: string;
  approved_by?: string; approved_at?: string;
}

interface Gate {
  id: string; request_id: string;
  type: "plan_approval" | "ds_signoff" | "budget_increase" | "step_confirmation";
  status: "open" | "approved" | "rejected"; required_role: string;
  evidence: { kind: "plan" | "report"; ref: string; version: number };
  decided_by?: string; decided_role?: string; decided_at?: string; comment?: string;
}

interface Output {
  id: string; request_id: string; step_id: string;
  kind: "data_profile" | "feature_table" | "dataset" | "run" | "model_version" | "report" | "sample_list";
  title: string; summary: string; snowflake_ref: string; version?: string; created_at: string;
}

interface EvaluationReport {
  id: string; request_id: string; scope: "validation" | "test";
  model_ref: { name: string; version: string }; plan_version: number; harness_version: string;
  verdict: "pass" | "fail" | "blocked";
  headline: { plain: string; metric: string; value: number; ci: [number, number]; target: number };
  checks: { id: CheckId; status: "pass" | "warn" | "fail"; summary: string; details: object }[];
  curves: { precision_at_k: { k: number; precision: number }[];
            reliability: { bin_mid: number; observed: number; predicted: number; n: number }[] };
  repro: { code_hash: string; data_as_of: string; seeds: number[];
           packages: Record<string, string>; feature_table_version: string };
  signed_by?: string; signed_at?: string;
}

type CheckId = "label_maturity" | "split_embargo" | "point_in_time" | "leakage" | "primary_metric"
             | "baselines" | "calibration" | "fairness" | "sanity";
```

**Change rules after the freeze (checkpoint 1):**

- Adding an optional field is allowed. Announce it in the team channel and regenerate.
- Renaming or removing a field, or adding or removing an enum value, needs all three people's agreement.
- `details` objects in report checks are free-form. A renders them generically as key-value pairs, so C can add whatever helps.

## C2 · UI to API

The web app builds every screen from two things only: a snapshot loaded once, and an ordered event stream after it. Owners: A builds both sides. B's events flow through unchanged.

### Endpoints

| Method and path | Body | Returns |
| --- | --- | --- |
| `GET /api/me` | — | `{user, role, app_roles[], level, default_mode}` |
| `GET /api/requests` | — | `{needs_you: Request[], working: Request[], done: Request[]}` |
| `POST /api/requests` | `{text}` | `201 Request` |
| `GET /api/requests/{id}` | — | `{request, brief, plan, gates[], outputs[], reports[], last_seq}` |
| `GET /api/requests/{id}/events?after={seq}` | — | `text/event-stream` (below) |
| `POST /api/requests/{id}/messages` | `{text}` | `202 {queued, queue_id}` |
| `POST /api/requests/{id}/commands` | `{name, args?}` | `202` · `name` is one of `/plan`, `/pause`, `/resume`, `/explain` |
| `PATCH /api/requests/{id}` | `{mode}` | `200 Request` |
| `POST /api/gates/{id}/decision` | `{decision: "approve" \| "reject", comment?}` | `200 Gate` · `403 forbidden_role` without the required role |
| `POST /api/heartbeat` | — | `204` · refreshes the caller token; the UI calls it every 60 s while open |

Errors always look like `{"error": {"code", "message", "retryable"}}`. The codes are `unauthorized`, `forbidden_role`, `not_found`, `conflict_stage`, `budget_exceeded` and `upstream_unavailable`.

### Event stream

- Each SSE message carries `id: <seq>`, `event: <type>` and `data: <Event JSON>`.
- On reconnect the browser sends `Last-Event-ID` and the API replays from there. A comment heartbeat goes out every 15 s.
- Events are idempotent by `seq`.

| Event type | Payload | UI renders it as |
| --- | --- | --- |
| `message.user` | `{text}` | User bubble |
| `message.agent` | `{text}` (markdown, already at the user's level) | Agent prose |
| `step.started` | `{step_id, kind, title}`, kind ∈ reasoning, search, sql, python, train, evaluate | Live line with shimmer and timer |
| `step.delta` | `{step_id, field: "text" \| "code", chunk}` | Streaming text or code, appended in order |
| `step.completed` | `{step_id, summary, duration_ms, credits, preview?}` | Folds to one line: summary, duration, credits |
| `step.failed` | `{step_id, error, will_retry}` | Warning line |
| `plan.proposed` | `{plan, diff?: {added[], removed[], changed[]}}` | Plan card and plan panel |
| `gate.opened` / `gate.decided` | `{gate}` | Decision card, Needs you badge / resolved card |
| `output.created` | `{output}` | Outputs panel |
| `report.ready` | `{report}` | Report panel |
| `request.updated` | Partial `Request` | Header status, mode indicator, budget meter |
| `queue.updated` | `{items: {id, text}[]}` | Queued chips above the composer |
| `checkpoint.created` | `{checkpoint_id, label}` | Divider |

### Rules

- The UI keeps no state the stream can't rebuild. It ignores event types it doesn't know, so B can add types before A renders them.
- `summary` fields are written for the user's level by B. The UI never rewrites them.
- Folding, layout, motion and how reports are drawn are A's choice, within the v3 layout.

## C3 · API to orchestrator

The API and the orchestrator are separate SPCS services that talk over internal HTTP on the service DNS name. The API owns people: sign-in, roles and browsers. The orchestrator owns work: steps, events and gates. Owners: A calls, B serves.

### Commands in

Every command is a `POST` to `/internal/requests/{id}/commands` with this envelope. Starting a request is `POST /internal/requests` with the same envelope and `type: "start"`.

```json
{
  "command_id": "uuid, for idempotency",
  "type": "start | message | command | mode | gate_decision | heartbeat",
  "actor": { "user": "SPATEL", "role": "RETENTION_ANALYST", "app_roles": ["MODELDESK_USER"] },
  "caller_token": "value of Sf-Context-Current-User-Token",
  "body": { }
}
```

| type | body | Orchestrator behaviour |
| --- | --- | --- |
| `start` | `{text}` | Creates the request, emits `message.user`, begins Frame |
| `message` | `{text}` | Applied at the next step boundary; until then, emits `queue.updated` |
| `command` | `{name, args?}` | `/pause` holds at the boundary; `/resume` continues; `/plan` re-emits `plan.proposed`; `/explain` answers about the last step |
| `mode` | `{mode}` | Takes effect at the next tool call; emits `request.updated` |
| `gate_decision` | `{gate_id, decision, comment?}` | The API has already checked `required_role`; the orchestrator records it and emits `gate.decided` |
| `heartbeat` | `{}` | Refreshes the caller token only |

Responses are `202 {accepted: true}`. A repeated `command_id` returns `202` and is ignored.

### Events out

- `GET /internal/requests/{id}/events?after={seq}` streams the same events as C2. The API fans them out to browsers.
- Every event is also written to `MODELDESK.APP.EVENTS` (C6) before it's streamed. After a restart, the API catches up from the table.
- `seq` is per request, starts at 1 and has no gaps.

### Caller token

- The API takes `Sf-Context-Current-User-Token` from each ingress request and forwards the latest one with every command and heartbeat.
- The orchestrator holds the latest token per user in memory only. It's never logged or persisted.
- If a step needs the user's rights and the token has expired, the orchestrator doesn't start the step. It sets `status: "waiting"` with reason `user_away` and resumes on the next heartbeat.

### Run loop guarantees

B can build the loop however they like, but it must show this behaviour from outside:

1. One step at a time per request. Each step emits `step.started`, any number of `step.delta`, then exactly one `step.completed` or `step.failed`.
2. Every SQL statement sets `QUERY_TAG = '{"request":"<id>","step":"<step_id>"}'`.
3. Before each tool call, the side-effect class is checked against the mode (C4). In Ask mode, a compute or write step opens a `step_confirmation` gate and waits.
4. Stage changes are emitted as `request.updated`. `spent_credits` is updated after every step.
5. Stop cancels running queries and jobs at once. Pause waits for the current step to end.

## C4 · Tools

The LLM acts only through tools. Each tool declares a side-effect class, and the orchestrator enforces mode and budget against that class in code, not in the prompt. Owner: B. The objects the tools read and write are defined in C6.

### Definition and result

```json
{
  "name": "features.build",
  "description": "Written for the model: when to use it and what it returns",
  "input_schema": { "type": "object", "properties": { } },
  "side_effect": "read | sandbox_write | compute | external_write",
  "needs_caller": true,
  "estimate": "none | explain | job_forecast"
}
```

```json
{
  "ok": true,
  "summary": "One line in the user's terms, used for the folded step line",
  "data": { "compact, truncated for the model": true },
  "preview": { "columns": [], "rows": [] },
  "outputs": [ "Output objects created (C1)" ],
  "query_ids": [],
  "credits": 0.12,
  "error": { "code": "", "message": "", "retryable": false }
}
```

`preview` holds 20 rows at most. `data` is capped at about 4,000 tokens, and the tool decides what to keep.

### Mode rules

| Side effect | Plan mode | Ask before running | Auto |
| --- | --- | --- | --- |
| `read` | Runs | Runs | Runs |
| `sandbox_write` | Refused | Asks (`step_confirmation`) | Runs |
| `compute` | Refused | Asks, showing the forecast | Runs if the forecast fits the budget |
| `external_write` | Refused | Refused | Refused (none exist in the hackathon) |

### Hackathon tool set

| Tool | Side effect | Input | Result and outputs |
| --- | --- | --- | --- |
| `ontology.search` | read | `{query, k}` | Entities: name, table, description, classification, keys |
| `registry.search` | read | `{query}` | Existing models: name, version, description, headline metric |
| `sql.query` | read | `{sql, max_rows ≤ 1000}` | Preview and row count. Anything but `SELECT` or `WITH` is refused before it runs |
| `profile.table` | read | `{table, time_col?}` | `data_profile` output: rows, nulls, distinct counts, time coverage |
| `label.define` | read | `{entity_key, snapshot_col, horizon_days, outcome_sql}` | Checks the label definition and returns base rate and matured vs immature counts. After plan approval the orchestrator sends it to the harness (C5) |
| `features.build` | compute | `{spine, features: [{name, expr, source, window_days}]}` | `feature_table` output. The tool writes the as-of join from a fixed template, not the model |
| `model.train` | compute | `{feature_table, label_view, algorithm, params, seed}` | `run` and `model_version` outputs, logged to Experiments, registered in the sandbox. Sees train and validation only |
| `harness.score_validation` | read | `{model_ref}` | `EvaluationReport` with `scope: validation` |
| `harness.nominate` | compute | `{model_ref}` | `EvaluationReport` with `scope: test`; at most 2 per request |

### Hard rules

- Writes only go to `ML_PROTO.<USER>`. The tool checks the fully qualified name, whatever SQL the model sends.
- Point-in-time joins come from the tool's template. The model chooses features, never join logic.
- Training and the harness are separate. `model.train` can't read the test window; the harness enforces that (C5).

## C5 · Orchestrator to harness

The harness owns the split, the labels for every window and all scoring. The agent gets views of train and validation only. Owners: B calls, C serves.

The harness runs with its own service role, not the caller's, because it must read the locked schema no user can. That role gets read access to every `ML_PROTO.<USER>` sandbox through future grants.

### Endpoints

| Call | Body | Returns |
| --- | --- | --- |
| `POST /internal/splits` | `{request_id, plan}`: the label definition and split from the approved plan | `{train_view, validation_view, stats}` |
| `POST /internal/evaluate` | `{request_id, plan_version, model_ref, scope: "validation" \| "test"}` | `202 {job_id}` |
| `GET /internal/evaluate/{job_id}` | — | `{status: "running" \| "done" \| "failed", report?: EvaluationReport, error?}` |
| `GET /internal/health` | — | `{harness_version}` |

- **`/splits` is called once per approved plan version.** It materialises labels for all windows into `ML_EVAL.LOCKED.<REQ>_LABELS`, which only the harness role can read.
- It returns two sandbox views that expose only the train and validation windows.
- `stats` holds rows, positives, and the immature snapshots excluded, per window.
- **`scope: test` is allowed twice per request.** A third call returns `409 nomination_limit`.
- **Results come back by polling.** The orchestrator polls every 5 s and turns progress into `step.delta` text for the user.

### What the harness expects of a model

- A Snowflake Model Registry version in the user's sandbox, with a `predict_proba` method.
- Its input columns are exactly the plan's `features.columns` names.
- The feature table covers every spine row in every window. Features aren't labels, so the harness joins them to the locked labels itself.
- If a contract check fails (a missing method, missing columns, or a feature table without the test window), the harness returns `verdict: "blocked"` with the failing check, not an error.

### Harness settings (C's file, versioned with the harness)

| Setting | Hackathon value |
| --- | --- |
| Minimum embargo | Equal to the label horizon |
| Bootstrap | 1,000 resamples, grouped by client |
| Leakage flag | Any single feature with validation ROC-AUC above 0.90, or any source timestamp after the snapshot |
| Calibration tolerance | Expected calibration error up to 0.05 |
| Minimum positives | 200 in train, 50 in each of validation and test |

**Known limitation:** the agent can still query raw source tables for the test period with `sql.query`. The labels it trains on can't come from there, but phase 1 adds a time fence on the agent's reads.

## C6 · Snowflake objects

Every database, role, table and naming rule the services share is defined here. C creates them all with one idempotent setup script (`infra/setup.sql`). B and A only read and write them as the table below says.

### Layout and ownership

| Object | Purpose | Written by | Read by |
| --- | --- | --- | --- |
| `MODELDESK.APP` | App state (tables below) | Orchestrator (service role) | API, orchestrator |
| `ML_PROTO.<USER>` | Per-user sandbox: spine, feature tables, models | Tools, as the caller | Agent as caller, harness role |
| `ML_EVAL.LOCKED` | Labels for all windows | Harness only | Harness only |
| `@ML_EVAL.REPORTS.FILES` | Report JSON per evaluation | Harness | API (service role) |
| `DEMO_DATA.SIPP` | Synthetic source data | Data generator (C) | Agent as caller |

| Role | Holds |
| --- | --- |
| `MODELDESK_APP_OWNER` | Owns the API and orchestrator services. Holds the CALLER grants (`USAGE` on the demo warehouse and databases, `SELECT` on `DEMO_DATA`, `CREATE` in `ML_PROTO`) |
| `MODELDESK_HARNESS` | Owns the harness service. The only role with access to `ML_EVAL.LOCKED`; future-grant read on `ML_PROTO` |
| `MODELDESK_USER`, `MODELDESK_DS_REVIEWER` | App roles, granted to people; checked with `IS_ROLE_IN_SESSION` |
| `RETENTION_ANALYST` | Demo business role: reads `DEMO_DATA.SIPP`, owns its sandbox |

### App state tables

```sql
CREATE TABLE MODELDESK.APP.REQUESTS (
  id STRING PRIMARY KEY, title STRING, owner STRING, role_used STRING,
  stage STRING, status STRING, mode STRING, level STRING, plan_version NUMBER,
  budget_credits NUMBER(10,2), spent_credits NUMBER(10,2),
  created_at TIMESTAMP_TZ, updated_at TIMESTAMP_TZ);

CREATE TABLE MODELDESK.APP.EVENTS (
  request_id STRING NOT NULL, seq NUMBER NOT NULL, id STRING NOT NULL,
  ts TIMESTAMP_TZ NOT NULL, type STRING NOT NULL, step_id STRING,
  payload VARIANT NOT NULL, credits NUMBER(10,2), query_ids ARRAY,
  PRIMARY KEY (request_id, seq));

CREATE TABLE MODELDESK.APP.PLANS (
  request_id STRING, version NUMBER, status STRING, body VARIANT,
  approved_by STRING, approved_at TIMESTAMP_TZ, PRIMARY KEY (request_id, version));

CREATE TABLE MODELDESK.APP.GATES (
  id STRING PRIMARY KEY, request_id STRING, type STRING, status STRING,
  required_role STRING, evidence VARIANT, decided_by STRING, decided_role STRING,
  decided_at TIMESTAMP_TZ, comment STRING);

CREATE TABLE MODELDESK.APP.OUTPUTS (
  id STRING PRIMARY KEY, request_id STRING, step_id STRING, kind STRING,
  title STRING, summary STRING, snowflake_ref STRING, version STRING, created_at TIMESTAMP_TZ);

CREATE TABLE MODELDESK.APP.REPORTS (
  id STRING PRIMARY KEY, request_id STRING, scope STRING, model_name STRING,
  model_version STRING, plan_version NUMBER, harness_version STRING, verdict STRING,
  body VARIANT, stage_path STRING, signed_by STRING, signed_at TIMESTAMP_TZ);

CREATE TABLE MODELDESK.APP.ONTOLOGY (
  entity STRING, table_fqn STRING, description STRING, classification STRING,
  keys ARRAY, time_col STRING, related ARRAY);
```

`body` columns hold the C1 JSON exactly. Snowflake doesn't enforce primary keys, so writers must keep them unique.

### Naming in the sandbox

| Object | Name |
| --- | --- |
| Spine (entity, snapshot) | `ML_PROTO.<USER>.<REQ>_SPINE` |
| Train and validation label views (from the harness) | `<REQ>_LABELS_TRAIN_V`, `<REQ>_LABELS_VAL_V` |
| Feature table | `<REQ>_FEATURES_V<n>` |
| Registry model | Name `<REQ>_MODEL`, version `P<plan>_R<round>_<k>` |
| Report file | `@ML_EVAL.REPORTS.FILES/<request>/<model>/<version>/<report_id>.json` |

`<REQ>` is the last 8 characters of the request ID, upper case. Each model version's comment holds `{request_id, plan_version, run_id, feature_table, code_hash}` as JSON, and validation metrics are set with `set_metric`.

### Synthetic source tables (`DEMO_DATA.SIPP`)

| Table | Columns |
| --- | --- |
| `CLIENT_DIM` | client_id, birth_year, sex, region, tenure_start |
| `SIPP_ACCOUNT_WEEKLY` | client_id, snapshot_date, account_status, pot_value, cash_pct, in_drawdown, transfer_pack_sent_date |
| `TRANSFER_OUT_EVENTS` | client_id, request_date, ceding_scheme_ref |
| `WEB_SESSIONS` | client_id, session_ts, pages |
| `CONTACT_INTERACTIONS` | client_id, contact_ts, channel, reason_code |
| `MARKET_DAILY` | date, ftse_all_share |

`transfer_pack_sent_date` is the planted leak. It's backfilled onto snapshots from before the request, as a bad backfill in real data would be. The ontology rows for these tables are part of the setup script.

## Working in parallel

Nobody waits on anybody. Each person builds against a mock of their neighbour, made from the C1 fixtures, until the integration checkpoints.

### Repository

```text
modeldesk/
  contracts/        C1 schemas, generated types, fixtures (joint; B merges)
    schemas/        *.json, JSON Schema 2020-12
    fixtures/       sipp_demo.events.jsonl, plan.v1.json, report.validation.json, report.test.json
    gen/            ts/ and py/, generated, never edited by hand
  web/              A · React, shadcn/ui, AI Elements
  api/              A · API service (language is A's choice)
  orchestrator/     B · Python 3.11, Snowpark, Snowflake ML, Cortex
  harness/          C · Python 3.11, Snowflake ML
  data/             C · synthetic data generator
  infra/            C · setup.sql, SPCS service specs, compute pool, deploy script
  mocks/            mock-orchestrator (A uses), mock-harness (B uses)
```

### Mocks and fixtures

| Mock | Built by | Used by | Behaviour |
| --- | --- | --- | --- |
| `sipp_demo.events.jsonl` | B, by hand, before checkpoint 1 | A, B | The full demo as an event stream, about 120 events, following the v3 Agent mock |
| `mock-orchestrator` | A | A | Serves C3. Replays the fixture at real or 10× speed and acknowledges commands. A gate pauses the replay until a decision arrives |
| `mock-harness` | C, in the first hours | B | Serves C5. Returns the fixture reports after 10 s and builds naive split views on the synthetic data |
| Hand-built model | C | C | A small logistic model registered by script, so the harness is tested without the agent |
| Test user | C | A, B, C | A Snowflake user with key-pair auth and the demo roles, for local runs outside SPCS |

### Rules that keep the seams honest

- In development mode, every service validates the payloads it sends against `contracts/schemas`. A schema failure is a bug in the sender.
- The fixture stream is the shared acceptance test for C2 and C3. If A renders it and B can produce it, the seam works.
- Locally, everything runs with Docker Compose against the dev Snowflake account. SPCS is used at the integration checkpoints and for the demo.

## Integration plan

The hackathon has four checkpoints. At each one, all three people stop and prove one seam together before going on. Times assume two days; scale them to the real length. (Diagram "integration checkpoints · 3 workstreams, 4 checkpoints" is not exported; see the doc.)

If a checkpoint fails, the owner of the failing side fixes it first. Everyone else carries on against mocks.

### Acceptance tests per contract

| Contract | The test | Owner |
| --- | --- | --- |
| C1 types | All fixtures validate against the schemas; generated TS and Python compile | B |
| C2 UI to API | The UI renders `sipp_demo.events.jsonl` from the real API. A reconnect mid-stream loses and repeats nothing | A |
| C3 API to orchestrator | The same stream comes from the real orchestrator for the demo prompt. A repeated `command_id` is ignored. A plan gate decided in the UI resumes the run | A and B |
| C4 tools | Plan mode refuses `features.build`. Ask mode opens a `step_confirmation`. `sql.query` refuses an `INSERT`. Every query carries the `QUERY_TAG` | B |
| C5 harness | The hand-built model gets a report. A model trained on the planted leak gets `leakage: fail`. A third nomination returns `409` | C |
| C6 objects | `setup.sql` runs twice with no errors. The demo user can't read `ML_EVAL.LOCKED` | C |

## Freedom and rules

Inside your own service, you decide. At the seams, the contracts decide.

| Fixed: changing it needs all three | Yours to decide |
| --- | --- |
| The six contracts: field names, enums, endpoints, table names | **A:** component structure, state management, styling within shadcn/ui, motion, how reports and charts are drawn, the API's language and framework, all microcopy |
| The v3 layout regions: inbox left, stream centre, plan and outputs right, composer at the bottom | **B:** agent architecture (one loop or planner and executor), prompts, which Claude model on Cortex, how diagnoses read, feature ideas, algorithms, retries |
| Gate types and which role decides each | **C:** harness internals and libraries, extra fields in check `details`, data generator design, SPCS sizing |
| The point-in-time join template, and the split owned by the harness | **Everyone:** tests, logging, local tooling |
| Stage names and the demo script | |

### Changing a contract

1. Propose it in the team channel with the diff to `contracts/schemas` and the fixtures.
2. All three agree.
3. B merges, the types are regenerated, and the fixtures are updated in the same change.

After checkpoint 3, only fixes are allowed. No new fields.

### Done means

- The demo script runs twice in a row on SPCS.
- All six acceptance tests pass.
- Every gate decision is in `MODELDESK.APP.GATES` with the decider's role.
- Re-running the harness on the same model and seed gives the same numbers.

### Stretch goals

Once everything above is done, pick any of these freely:

- Auto mode with the budget forecast
- `/notebook` export
- Plain and Expert views of the report
- Approve from Home
- A second synthetic problem
