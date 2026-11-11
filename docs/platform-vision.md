# Model Desk — platform vision and build spec

Nov 9, 2026 · Kieran

Source: Claude Doc "Model Desk — platform vision and build spec", tab 1 (https://claude.ai/artifact/FDiYtYaNnWNMyqP3d7AGd3). Embedded diagrams (request lifecycle, Snowflake architecture, delivery roadmap) are not exported; see the doc. Hackathon build plan: [build-spec-hackathon.md](build-spec-hackathon.md). UI mocks: Claude design canvas "Internal Auto-ML — chat UX" (https://claude.ai/artifact/HesUzLJDsxfqTTBsMqQwTF).

## Vision

Model Desk lets anyone at Hargreaves Lansdown turn a business decision into a governed, monitored ML model. They delegate the work to an agent that runs inside Snowflake as them, and Data Science signs off the result.

**The problem.** Demand for models (churn, propensity, forecasting, segmentation) outstrips Data Science capacity. Requests queue, teams fall back on spreadsheets, and DS time goes on repeat builds of the same patterns. When models do ship, ownership, monitoring and evidence for model risk are inconsistent.

**The outcome we want.** A business user describes a decision in plain English. Within hours they get a candidate model with an independent evaluation report. Within days they get a reviewed model their team can act on. Every live model has an owner, a report, monitoring and a change history. Target times are an open decision (section 12).

**What it is not.** It is not a BI tool, it does not replace Data Science, and it never deploys to clients without a human approver.

### Principles

1. **Agents, not chats.** The unit of work is a long-running agent with a plan, outputs and state. Conversation steers the agent; it doesn't drive each step.
2. **The plan is the contract.** The user approves a plan once: target, data, success bar, budget. The agent then works on its own inside it. Modes set how often it checks in.
3. **Runs as you.** Identity and data access come from Snowflake roles. The app adds no permissions of its own to data.
4. **Builder and judge are separate.** A fixed evaluation harness the building agent cannot change or see into scores every candidate. A data scientist signs the report.
5. **Honest about time.** Outcomes like "transferred within 90 days" mature slowly. The platform tracks label maturity and never shows a measurement before it exists.
6. **Outputs are objects, not messages.** Data profiles, feature views, model versions, reports and call lists live in Snowflake with lineage, and the UI links to them.
7. **Reversible and auditable.** Every approval, data access and model change is logged and can be rolled back.
8. **Disclosure by level, not by removal.** Plain-English users see summaries, but the full rigour (code, splits, leakage checks, calibration) is always one click away.

## Users, roles and permissions

Four kinds of people use Model Desk. What each can do comes from two layers: Snowflake roles decide what **data** they can touch, and Model Desk app roles decide which **decisions** they can make.

| Persona | Typical person | Job in Model Desk | Default level |
| --- | --- | --- | --- |
| Requester | Retention lead, ops planner, marketing analyst | Asks, steers, approves plans for their own work, uses outputs | Plain |
| DS reviewer | Central Data Science | Co-owns models, signs evaluation reports, takes over code | Expert |
| Approver | Model owner or model risk delegate | Approves Confidential data use, go-live and material changes | Practitioner |
| Admin | MLOps | Budgets, warehouses, role mapping, platform health | Expert |

### Layer 1: Snowflake roles (data)

- The app runs on Snowpark Container Services with **caller's rights**: queries run as the signed-in user, so the agent sees only what their role sees ([SPCS SQL execution](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/spcs-execute-sql)).
- Caller's rights are **restricted**: an action needs the user's privilege *and* a CALLER grant to the service's owner role. Those grants set the ceiling on what the app can ever do for anyone, and the platform team controls them.
- All agent writes go to a per-user sandbox schema (e.g. `ML_PROTO.<USER>`). Writing anywhere else is a gated action.
- Constraints that shape the design:
  - Browser sessions use the user's **default role**, and roles can't be switched mid-session. The v3 "role switcher" therefore becomes a choice between the default role and `DEFAULT_SECONDARY_ROLES = ('ALL')`, recorded per request (section 12).
  - The caller token lasts **120 seconds** by default, up to 7 days if extended. Long-running and background agents need a deliberate design for acting on a user's behalf while they're away (section 5).
  - Queries use the **user's default warehouse**, not the service's, so warehouse sizing and cost limits attach to users.

### Layer 2: Model Desk app roles (decisions)

These are Snowflake roles (`MODELDESK_USER`, `MODELDESK_DS_REVIEWER`, `MODELDESK_APPROVER`, `MODELDESK_ADMIN`) granted to people. The app checks them in session. `DS_REVIEWER` is new since v3, added after the Data Science review.

| Action | User | DS reviewer | Approver | Admin |
| --- | --- | --- | --- | --- |
| Create a request, discover, train in own sandbox | Yes | Yes | Yes | Yes |
| Approve a plan that uses no sensitive data | Yes | Yes | Yes | Yes |
| Use Confidential data in a model | Requests | Requests | Approves | No |
| Sign an evaluation report | No | Yes | No | No |
| Produce a client-usable list or score | No | No | Yes, after a signed report | No |
| Take a model live | Requests | Signs | Approves | No |
| Material change to a live model (new data, new features, new algorithm) | Requests | Signs | Approves | No |
| Same-spec refit of a live model | Automatic within policy | Notified | Notified | No |
| Edit pipeline code or take a model over | No | Yes | No | Yes |
| Set budgets, warehouses, role mapping, policies | No | No | No | Yes |

**Separation of duties.** Nobody approves their own go-live. The person who signs an evaluation report can't be the person who built the model. Every gate records who, when, the evidence version, and the role in session.

## Product structure

Model Desk has five surfaces built around one object, the **request**: an agent working on one business question, from the first message to the model's retirement.

### Surfaces

| Surface | Job | Layout (from v3) |
| --- | --- | --- |
| Home | Route attention across all of a person's requests | Agent inbox (left) · Needs you, Working, Live with inline decisions (centre) |
| Request workspace | Watch, steer and approve one agent | Agent inbox (left) · activity stream and composer (centre) · plan, outputs, running-as (right) |
| Model page | Look after a live model | Same three panes · health and suggestions (centre) · standing orders, changes, cost (right) |
| Review queue | DS reviewers and approvers clear their gates | List of pending gates · evidence pack opens beside it. **New: not in the mocks** |
| Settings | Personal and admin configuration | Section nav · one section at a time (v3 Settings) |

The request workspace and the model page are the same object at different lifecycle stages. A request becomes a model page when it goes live.

### Core objects

| Object | What it is | Stored in |
| --- | --- | --- |
| Request | One business question, its conversation, its agent state, its owner | App state store |
| Brief | The framed question: decision, population, horizon, capacity, exclusions | App state, versioned |
| Plan | The contract: label spec, data, approach, success criteria, budget, gates | App state, versioned; each approval pins a version |
| Step | One unit of agent work (search, query, code run) with inputs, outputs, cost, duration | Event log |
| Output | Something the agent produced: data profile, feature view, dataset, model version, report, list | Snowflake objects, linked by ID |
| Run | One training attempt: config, code hash, data snapshot, metrics on validation | Snowflake ML Experiments |
| Evaluation report | The harness's verdict on a candidate, pinned to a model version | Snowflake stage + registry metadata |
| Gate | A decision only a person can make, with its evidence and outcome | App state + audit log |
| Deployment | A model version wired to a schedule and a destination | Registry + Snowflake Tasks |
| Live model | A deployment plus monitoring, standing orders and change history | Registry + ML Observability |
| Command | A saved instruction a user or team can run with `/` | App state |
| Memory | Facts about a user that tailor framing and explanations | App state, per user |

### Request lifecycle

*(Diagram: request lifecycle · 10 states, 3 human gates. Not exported.)*

Paused and Stopped can apply to any working state. A material change to a live model sends it back to Building, and then through every gate again.

## Agent behaviour

The agent works through eight stages. Modes set how often it stops, but a fixed set of gates stops it in every mode.

### Stages

1. **Frame.** Turn the ask into a brief by asking only business questions (decision, horizon, capacity, exclusions). Run a feasibility check in the first minute: is there enough history, are there enough positive cases, and can the user's role see the data? If any fails, stop and say so.
2. **Discover.** Search the ontology and existing models, profile tables, check label maturity, and screen for leakage (fields recorded after the outcome, suspiciously strong single features).
3. **Plan.** Draft the plan: label spec, population and exclusions, split design, success criteria, approach, budget and a cost forecast per round. **Gate: plan approval.**
4. **Build.** Write point-in-time feature views, train candidates as ML Jobs, and log every run to Experiments. Diagnose only on validation data.
5. **Evaluate.** Hand each candidate to the independent harness (section 6). Below the bar: diagnose, update the plan as a diff, and loop to Discover or Build within budget. Meets the bar: it becomes a candidate.
6. **Deliver.** Present the candidate, the harness report and sample outputs (rows with reasons). **Gate: DS sign-off.**
7. **Go live.** Shadow run, then approver. **Gate: go-live** (section 8).
8. **Look after.** Follow the model's standing orders: monitor, refit, propose changes.

### Modes

The mode is per request, cycled with Shift+Tab or a visible dropdown, and its default comes from settings. Admins can cap the highest mode per app role. For example, requesters may not get Auto in phase 1.

| Mode | Stops | Never |
| --- | --- | --- |
| Ask before running (default) | Before each training run, each write outside temp tables, each new data source | Skips a gate |
| Auto within budget | Only at gates and when the budget runs out | Skips a gate, exceeds the budget |
| Plan | After the plan is drafted | Writes data, trains, or spends beyond read-only queries |

### Gates that apply in every mode

- Label definition, population, exclusions and test window: the agent proposes, a person signs within the plan.
- Using Confidential data or LLM-derived features from free text.
- Signing an evaluation report (DS reviewer). The test set is never opened by the building agent, in any mode.
- Producing a client-usable list or score, going live, and any material change to a live model.
- Changing list size or thresholds that alter who is contacted. These are business decisions, not settings.
- Writing outside the user's sandbox, and raising a budget.

### Budgets

- Each request has a credit budget, defaulted from settings. The plan shows a cost forecast per round before anything runs, in credits and pounds.
- Users have a monthly limit set by an admin, with separate pots for building, background improvement and live-model upkeep.
- When a budget runs out, the agent finishes the current step and asks, or stops at once (a user setting).

### Steering

- Messages sent mid-step are queued and applied at the next step boundary. The agent replies with the resulting plan diff, which creates a new plan version.
- `/pause` holds at the next step boundary. Stop cancels running Snowflake queries and jobs at once.
- Every step and checkpoint can be restored or branched from.

### Stopping instead of looping

The agent stops and reports when:

- it has no access to data it needs
- there are too few positive cases
- labels haven't matured
- leakage is suspected
- the budget is spent
- 3 consecutive rounds bring no improvement on validation

It never keeps spending to find a pass.

## Architecture on Snowflake

Everything runs inside the Snowflake account: the app and agent on Snowpark Container Services, the ML on Snowflake ML, and the model calls on Cortex. Data never leaves Snowflake's governance boundary unless an external integration is approved.

*(Diagram: architecture · services inside the Snowflake account. Not exported.)*

The orchestrator reaches data, ML and the LLM through tools that run as the user. The harness reads candidates from the Registry and is the only service that can read the locked test sets.

### Components

| Component | Runs on | Responsibility |
| --- | --- | --- |
| Web app | SPCS public endpoint | React UI with shadcn/ui and AI Elements; Snowflake sign-in at ingress |
| API and session service | SPCS, `executeAsCaller: true` | Caller's-rights connections, app-role checks, event stream (SSE) to the UI |
| Agent orchestrator | SPCS long-running service | Plan and execute loop, tool calls, checkpoint after every step, resume after restart |
| LLM | Snowflake Cortex (Claude models via AI_COMPLETE or the REST API) | Reasoning, planning, code writing, explanations ([models and regions](https://docs.snowflake.com/user-guide/snowflake-cortex/aisql-regional-availability)) |
| Tool layer | Inside the orchestrator | SQL (as caller), ontology and model search, profiling, Feature Store, ML Jobs, Experiments, Registry, notebook export, notify |
| Evaluation harness | Separate SPCS service, separate owner role | Scores candidates on the locked test set and writes the report (section 6) |
| Scheduler | Snowflake Tasks | Batch scoring for deployments, monitor refreshes, standing-order wake-ups |
| State store | Snowflake tables in a `MODELDESK` database (hybrid tables if row latency needs it) | Requests, plans, events, gates, memory, commands |
| Notifications | External access integration | Microsoft Teams and email |
| Monitoring | Snowflake ML Observability | Drift, performance once labels mature, alerts ([Snowflake ML overview](https://docs.snowflake.com/developer-guide/snowflake-ml/overview)) |

### Design decisions

1. **Every query is tagged.** The orchestrator sets `QUERY_TAG` to the request and step IDs. Cost per request, lineage and the audit trail then come from `QUERY_HISTORY` and `ACCESS_HISTORY` without a separate ledger.
2. **The builder can't see the test set.** Locked test sets live in a schema only the harness's role can read. The app's owner role gets no CALLER grant on it, so no user session can reach it through the agent.
3. **Durable, event-sourced agent.** Each step is written to the event log before it runs, and tool calls are idempotent. A restarted container resumes from the last checkpoint and replays nothing that cost money.
4. **Acting while the user is away.** Caller tokens last 120 seconds by default and 7 days at most. During a build, the agent uses an extended token and pauses for re-authentication when it lapses. A live model must not depend on a person's session, so at go-live the approver binds it to a service role scoped to that model's inputs and outputs.
5. **Budgets are enforced twice.** The orchestrator checks a cost forecast before each round, and a Snowflake resource monitor on the user's warehouse is the hard stop.
6. **Packaging.** Build it as SPCS services in HL's own account first. Packaging as a Snowflake Native App is a later option and isn't required for internal use.

## Independent evaluation harness

A fixed, versioned service scores every candidate. The building agent can propose a model but never grades it. This is the core mechanism for trusting the models Model Desk produces.

### How it separates building from judging

- The harness owns the label maturity cutoff, the split, and the locked test set. The agent gets the train and validation windows only.
- During the build loop the agent sees **validation** results, scored by the harness, and diagnoses on those.
- The agent **nominates** a final candidate, and only then is it scored on the test set. Each request gets 2 nominations; more need a DS reviewer's approval. The report shows how many candidates have touched the test set.
- DS and MLOps own the harness code, change it only through reviewed pull requests, and run it as its own SPCS service. The agent can't modify or call around it.

### Checks

| Check | What it does | Blocks sign-off when |
| --- | --- | --- |
| Label maturity | Uses only snapshots whose outcome window has closed (snapshot date ≤ today minus horizon) | Any immature label is in train, validation or test |
| Temporal split with embargo | Train, validation and test by time, with a gap of at least the horizon between them, grouped by client | Windows overlap, or the gap is shorter than the horizon |
| Point-in-time features | Every feature value is as of its snapshot, and features are available at scoring time | A feature timestamp falls after its snapshot, or a feature lands too late for the schedule |
| Leakage screen | Flags fields recorded after the outcome, outlier single-feature power, and text-derived features that restate the outcome | A flagged feature is kept without a recorded reason |
| Primary metric | The plan's metric (e.g. precision in the top K) with a bootstrap confidence interval over clients | The lower bound misses the bar, if the plan says so |
| Baselines | Random, a simple model, and any existing model named in the plan | It doesn't beat the baselines the plan requires |
| Calibration | Reliability curve, Brier score, calibration by segment | Scores are shown as likelihoods but aren't calibrated within tolerance |
| Fairness | The plan's fairness metric across approved groups (e.g. false-positive ratio by age band and sex) | Above the plan's threshold |
| Stability | Performance by month and by segment | Variation is beyond tolerance |
| Sanity | Minimum positive cases, duplicate rows, prevalence shift between windows | Any hard minimum fails |

### The report

- One report per nominated candidate, stored on a stage and linked to the model version in the Registry as metadata.
- It records what's needed to reproduce the result: harness version, code hash, plan version, feature view versions, data as-of timestamp (Time Travel or a zero-copy clone), random seeds and package versions.
- It's written for two readers. The top is the business summary in the user's terms ("about 1 in 4 of 500 weekly calls"). Below it, each check has its evidence.
- A DS reviewer signs a specific report version. Any change to the model, data, plan or harness version voids the signature.

## Data and ML conventions

Everything the agent builds is a named, versioned Snowflake object in a predictable place. A data scientist can pick up any model without asking the agent what it did.

| Artifact | Where it lives while building | Where it lives once live |
| --- | --- | --- |
| Scratch tables | `ML_PROTO.<USER>`, prefixed with the request ID | Not kept |
| Label spec | A SQL view in the sandbox, versioned with the plan | Copied with the model |
| Feature views | Feature Store, sandbox namespace, with a `timestamp_col` | Shared Feature Store namespace, after DS review |
| Training snapshots | Snowflake ML Datasets, versioned per plan version | Kept for the model's lifetime |
| Runs | Snowflake ML Experiments, one experiment per request | Kept |
| Model versions | Registry in the sandbox schema | Registry in a shared `ML_MODELS` schema |
| Evaluation reports | Stage `@MODELDESK.REPORTS/<request>/<version>/` | Same, retained under model risk policy |
| Scores and lists | Sample tables in the sandbox | `ML_SCORES.<model>`, plus the destination feed |

### Rules

- **Point-in-time by construction.** Every request builds a spine table of entity and snapshot time. Features join to the spine as of the snapshot, never by period labels.
- **Labels are code.** The label is a reviewed SQL view with its horizon, exclusions and maturity rule, not a description in chat.
- **Pipeline as code.** Every step's SQL and Python is saved. A request exports as a Snowflake Notebook and a git repository. A DS reviewer can edit the pipeline and hand it back, and the agent continues from their version.
- **Reuse first.** The agent prefers reviewed feature views and models already in the registry. It proposes new shared features to DS instead of duplicating them.
- **Tags and policies carry through.** Objects get tags for request ID, owner and inherited data classification. Masking and row access policies apply because queries run as the user.
- **Retention.** Sandbox objects expire a set number of days after a request closes unless promoted. Anything that went live follows the model risk retention policy, and a user deleting their history doesn't touch it.

## Go-live and operations

A model reaches clients by one path only. Once it's live, every change is classed by how much it can alter who gets contacted, and the class decides who approves it.

### Path to live

1. **Signed candidate.** A DS reviewer has signed the harness report.
2. **Deployment spec.** Covers:
   - the destination (a table, the dialler queue, Power BI) and the schedule
   - the list size or threshold
   - post-model exclusion rules: do-not-contact, vulnerability flags, open complaints, consent. Business and compliance own the defaults.
   - owners: business owner, DS co-owner, MLOps on-call
   - control group size
3. **Shadow run.** It scores on schedule into a shadow table and nobody acts on it. It checks what can be checked before outcomes mature: the pipeline runs, features are fresh and arrive on time, volumes and score distribution match the test period, and ranks are stable. It does **not** check precision. Its length is set by these checks, not by a fixed number of weeks.
4. **Approval.** The approver reviews the report, the shadow checks and the deployment spec, then binds the model to a service role scoped to its inputs and outputs.
5. **Live.** A Snowflake Task scores and writes to the destination, and the request becomes a model page.

Before step 4, any list is a watermarked sample for review only and can't be sent to a destination.

### Change classes

| Class | Examples | Approved by | Takes effect |
| --- | --- | --- | --- |
| Rule | Exclusions, publish day, output format | Business owner | Next run; logged and undoable |
| Capacity | List size, score threshold | Business owner and approver (changes who is contacted) | Next run |
| Refit | Same features and algorithm, newer data | Pre-approved by policy; harness must pass; owner confirms the switch | After shadow comparison |
| Material | New data source, new features, new algorithm, changed label | Full path: DS sign-off and approver | After shadow |
| Retire | Switch off a model | Business owner or approver | Immediately |

### Monitoring that respects label lag

| Signal | Available | Source |
| --- | --- | --- |
| Pipeline success, freshness, volumes | From day 1 | Task history |
| Feature drift and score drift | From day 1 | ML Observability |
| Rank stability between runs | From day 1 | Score tables |
| Outcomes so far, with % of the window elapsed | Partial, labelled as partial | Label view |
| Precision in top K, calibration, fairness by cohort | Once a cohort's horizon has passed | Harness, rerun on matured cohorts |

Thresholds raise ML Observability alerts. Each alert becomes an agent suggestion on the model page, or an incident for MLOps.

### Control group

- A random share of flagged clients is held out from contact; its size is set at go-live. Without it, a working model looks worse over time because the calls prevent the transfers it predicted.
- The holdout measures impact in business terms (clients and assets retained versus control).
- Refits train on labels the intervention hasn't contaminated. Moving from risk to uplift modelling is a later step.

### Rollback and standing orders

- A deployment always points at a Registry version. Rollback repoints it in one click and keeps the previous version available.
- Each live model has standing orders: which checks run and how often, which actions the agent may take alone (within the change classes above), and a monthly budget. Changing standing orders is itself a change and follows the classes above.

## UI spec

The v3 mocks are the baseline layout ([design canvas](https://claude.ai/artifact/HesUzLJDsxfqTTBsMqQwTF), page "v3 · agentic"). This section fixes the rules so it can be built with shadcn/ui and AI Elements.

### Layout

- **Three panes.**
  - Left: agent inbox, 248 px, collapsible to icons.
  - Centre: activity stream, max 700 px.
  - Right: context panel, 320 px, collapsible.
- **The right panel changes with the stage.**
  - Building: Plan, Outputs, Running as.
  - At a gate: the evidence for that gate.
  - Live: Standing orders, Changes, Cost.
- **Narrow screens.** Below 1100 px the right panel becomes a drawer. Below 760 px the left pane becomes a sheet.

### Activity stream rules

1. The stream is a projection of the event log, so a request can be replayed and audited from what's on screen.
2. Only the active step is expanded. A finished step folds to one line: its result in the user's terms, duration and cost. Clicking reopens it.
3. SQL and Python stream as they're written. They're expanded by default for Practitioner and Expert, and folded for Plain.
4. A gate appears as one decision card at the end of the stream. Each request has at most one open decision card; the rest wait on Home.
5. Checkpoints appear as dividers with Restore and Branch.

### Composer

- `/` opens the command menu (filtered as you type; arrow keys, Enter, Esc). `@` mentions an ontology entity, table or model.
- Under the input are the mode indicator (Shift+Tab or click to change) and a budget meter.
- Queued steering messages sit above the input as removable chips. While the agent runs, the send button becomes Stop.
- Custom commands are saved prompt templates with parameters, scoped to a user or a team. They run as ordinary messages, so they can't bypass a gate.

| Built-in command | Does | Available |
| --- | --- | --- |
| `/plan` | Show or edit the plan (creates a new version) | Always |
| `/data` | Browse data the user's role can read | Always |
| `/budget` | Change this request's budget (raising it is a gate) | Always |
| `/pause`, `/resume` | Hold or continue at the next step boundary | While working |
| `/explain` | Explain the last step at the user's level | Always |
| `/compare` | Compare runs side by side | After the first run |
| `/notebook` | Export the pipeline to a Snowflake Notebook | After the first run |
| `/review` | Ask a DS reviewer to look at this request | Always |
| `/golive` | Start the deployment spec | After sign-off |
| `/share` | Give a person read access to this request | Always |
| `/level`, `/settings` | Change explanation level, open settings | Always |

### Home and review queue

- **Home: Needs you.** Gate cards with inline actions only where no evidence needs reading (e.g. a plan with no sensitive data). Gates involving sensitive data, sign-off or go-live open the request.
- **Review queue.** New in this spec. For DS reviewers and approvers: pending gates with age, an evidence pack viewer (report, diff from the previous version, code link), and Sign or Reject with a comment.
- **Notifications.** A Teams card per gate with a deep link. Quiet hours hold messages; agents keep working.

### Component mapping

| UI element | Component |
| --- | --- |
| Stream container, messages | AI Elements Conversation, Message, Response |
| Thinking line | Reasoning, collapsed when done |
| Discovery steps | Chain of Thought |
| SQL and Python execution | Tool with Code Block |
| Live status | Shimmer, Loader |
| Plan panel and plan card | Plan, Task |
| Queued messages and planned experiments | Queue |
| Gate cards | Confirmation |
| Checkpoint dividers | Checkpoint |
| Data and model references | Sources, Inline Citation |
| Outputs | Artifact |
| Composer and command menu | Prompt Input with shadcn Command |
| Budget meter | Context |
| Next questions | Suggestion |
| Settings | shadcn Tabs, Switch, Select, Form |

## Data model and APIs

The app state is a small set of versioned entities plus an append-only event log. The UI, audit trail and cost reports are all projections of that log.

### Entities

| Entity | Key fields |
| --- | --- |
| request | id, owner, title, problem type, stage, mode, status, budget, spend, role used, live model id |
| brief | request id, version, decision, population, horizon (days), capacity, exclusions |
| plan | request id, version, label view, population view, split spec, metric spec, fairness spec, baselines, approach, budget, cost forecast, approved by, approved at |
| event | id, request id, sequence, type, step id, payload, Snowflake query IDs, credits, timestamp |
| gate | id, request id, type, evidence refs, status, decided by, role in session, decided at, comment |
| output | id, request id, kind (profile, feature view, dataset, run, model version, report, list), Snowflake reference, version, producing step |
| deployment | id, model version, destination, schedule, list size, rules, control share, service role, owners |
| standing order | deployment id, check, cadence, allowed actions, monthly budget |
| command | id, scope (user or team), name, template, parameters |
| memory item | user, text, source request, created at, active |
| policy | admin settings: highest mode per role, budgets, gate rules, retention |

### Events

`message.user`, `message.agent`, `step.started`, `step.completed`, `step.failed`, `tool.called`, `plan.versioned`, `gate.opened`, `gate.decided`, `output.created`, `budget.updated`, `mode.changed`, `queue.added`, `queue.applied`, `checkpoint.created`, `notify.sent`.

### API

| Endpoint | Purpose |
| --- | --- |
| `POST /requests` · `GET /requests?view=home` | Create; list grouped as Needs you, Working, Live |
| `GET /requests/{id}/events` (SSE) | Stream events, resumable from a sequence number |
| `POST /requests/{id}/messages` | Steer; queued if a step is running |
| `POST /requests/{id}/commands` | Run a slash command |
| `PATCH /requests/{id}` | Change mode; a budget increase opens a gate |
| `POST /requests/{id}/nominate` | Send a candidate to the harness test set (counted) |
| `POST /gates/{id}/decision` | Approve, sign or reject, with comment |
| `POST /deployments` · `POST /deployments/{id}/rollback` | Go live; repoint to a previous version |
| `GET/PUT /settings/me` · `GET/PUT /admin/policies` | User settings; admin policy |

### Agent tools

Every tool declares a side-effect class, and the mode and gates are enforced against that class in the orchestrator, not in the prompt.

| Tool | Side effect | Notes |
| --- | --- | --- |
| `sql.query` | Read | Runs as caller; tagged; row cap for display; cost estimate before large scans |
| `ontology.search`, `registry.search` | Read | Cortex Search over ontology and registry metadata |
| `profile.table` | Read | Counts, nulls, distributions, time coverage |
| `sandbox.create_table` | Sandbox write | Only in `ML_PROTO.<USER>` |
| `features.define_view` | Sandbox write | Point-in-time against the spine |
| `dataset.snapshot` | Sandbox write | Versioned per plan |
| `ml.submit_job` | Sandbox write, uses compute | Credit forecast checked against budget first |
| `harness.score_validation` | Read | Validation metrics only |
| `harness.nominate` | Gate-counted | Limited per request |
| `notebook.export` | Sandbox write | Notebook and git export |
| `notify.send` | External write | Approved templates only |

Mode rules by class:

- **Plan mode:** reads only.
- **Ask before running:** asks before anything that uses compute or writes.
- **Auto:** runs reads and sandbox writes within budget.
- **Every mode:** writes outside the sandbox always go through a gate.

## Delivery roadmap

Build trust before breadth. One team and one problem type prove the harness and the review loop before anything reaches clients. Breadth comes only after the first model clears model risk review.

*(Diagram: delivery roadmap · 5 phases, 4 gates. Not exported.)*

Phase 0 is the next step. Phase 1 deliberately has no go-live path, so the pilot tests the agent and the harness without client risk.

## Open decisions and risks

Three decisions shape phase 0 and should be settled first: where the LLM runs, whether to build on Snowflake's own agentic ML, and how the agent acts for a user who isn't there.

| Decision | Options | Leaning |
| --- | --- | --- |
| Where the LLM runs | Claude on Cortex in-region · Cortex cross-region inference · an external API via an access integration | Cortex. Confirm which Claude models HL's region serves, and whether data governance accepts cross-region inference |
| Build or extend | Own orchestrator · build on Snowflake's agentic ML, which plans and runs ML workflows from prompts ([overview](https://docs.snowflake.com/developer-guide/snowflake-ml/overview)) | Spike both in phase 0. The harness, gates, roles and UI are needed either way |
| Role per request | User's default role only · `DEFAULT_SECONDARY_ROLES = ('ALL')` | Default role, recorded per request; secondary roles enabled per user by an admin |
| Identity while away | Extended caller token (up to 7 days) · service role during builds | Extended token for builds; service role only for live models, granted at go-live |
| Refit policy | What counts as a pre-approved refit, and who confirms the switch | Agree with model risk before phase 2 |
| Reviewers | Who holds DS reviewer and approver; review turnaround | Name them before phase 1 |
| Default exclusions | Do-not-contact, vulnerability, consent, open complaints | Compliance owns the list; applied after the model |
| Targets | Time to a candidate, time to live, quality bars per phase | Set before phase 1 |
| State store | Standard tables · hybrid tables | Standard first; hybrid if UI latency needs it and the region supports it |

### Risks

- **Optimistic or leaky models reach clients.** Mitigation: harness, nomination limits, DS sign-off, shadow run, approver.
- **DS reviewers become the bottleneck.** Mitigation: phase 1 limited to one team, a review turnaround agreed up front, and harness reports that make review fast.
- **Business users over-trust the headline number.** Mitigation: calibration checks, confidence ranges shown in plain terms, "prototype" watermarks until go-live.
- **Contact decisions harm clients (Consumer Duty).** Mitigation: default exclusions, a defined fairness metric, control groups, approver for any capacity change.
- **Cost runs away.** Mitigation: forecasts before every round, budgets, resource monitors as a hard stop.
- **Platform limits surface late.** Caller's rights token lifetime, role switching, Cortex model regions. Mitigation: phase 0 spikes each one before UI work depends on it.

### Sources

- [Snowpark Container Services: SQL execution and caller's rights](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/spcs-execute-sql)
- [Snowflake ML overview](https://docs.snowflake.com/developer-guide/snowflake-ml/overview)
- [Cortex AI Functions: models and regional availability](https://docs.snowflake.com/user-guide/snowflake-cortex/aisql-regional-availability)
- [Model Desk design canvas (v1–v3)](https://claude.ai/artifact/HesUzLJDsxfqTTBsMqQwTF)
