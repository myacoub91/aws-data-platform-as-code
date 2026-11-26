# AWS Data Platform as Code — Reference Architecture

**Status:** Draft for review
**Date:** 2026-11-26
**Audience:** the architect instantiating this blueprint, the Data Platform
Engineer who will operate it, and any engineer inheriting it

---

## 1. How to read this document

This is a **reference architecture for building a Data Platform as Code on AWS** —
a parameterized blueprint, not a single-environment design. Every component is
defined in a git repository and delivered through CI. It is meant to be
re-instantiated across environments by filling in the **environment parameters** of
§2.2 rather than rewriting prose; the body references those parameters instead of
hard-coding them. It is realized on AWS over an S3/Iceberg lake with a **pluggable serving engine**
(Redshift is the reference implementation; the engine's required behaviours are
pinned by a capability contract, §8.1), and is sized against a **reference profile**
(§7) that is an illustration, not the target.

Every statement carries a marker, because the point of this document is that no
decision is hidden:

| Marker | Meaning |
|---|---|
| **[DECIDED]** | Chosen by the platform owner. Not up for renegotiation without a new conversation. |
| **[CALL]** | Made by the architect during design. Override freely — but override explicitly. |
| **[PARAM]** | A blueprint dial (§2.2). Set per adopting environment; the default is stated. |
| **[ASSUMPTION]** | A guess about the adopting organisation. Validate during discovery (§15). |
| **[OPEN]** | Genuinely undecided. Listed in §13. |

If something in the body contradicts §12 (Decision Log), §12 wins. Acronyms are
expanded on first use and collected in Appendix B.

---

## 2. Context

This is a reference architecture for a **Data Platform as Code on AWS**: every
component — ingestion, storage, transformation, serving, access, and the context
catalog — is defined in a git repository and delivered through CI, over an
S3/Iceberg lake with a **pluggable serving engine** beneath it (Redshift is the
reference implementation; see the engine contract, §8.1).

It is **engine-pluggable and environment-neutral**: the shape holds across
organisations *and across serving engines*, and what varies — the warehouse engine
included — is captured as parameters (§2.2), with the engine's required behaviours
pinned by the capability contract (§8.1). To
keep the document concrete it is sized against a **reference profile** (§7) — a data
team of two to four SQL analysts supported by one Data Platform Engineer (DPE),
~10–20 sources, and tables in the hundreds of gigabytes. The profile is an
illustration, not the audience: a larger team, a different sector, or a greenfield
account changes sizing and a few parameters, not the architecture. (The no-DPE
variant is itself a supported parameter; see §2.2 and §2.3.)

Data arrives from external SaaS APIs, internal operational databases, and
AWS-native services. Consumers span business functions such as Marketing, Finance,
and Support — plus, increasingly, AI agents operated by non-technical staff.

The platform aims to survive turnover, to let each role do its own work from
runbooks, and to make changing a freshness requirement a configuration change
rather than a re-architecture.

**Brownfield-aware.** Because most adopters already run a Redshift warehouse in an
unknown state, the architecture assumes it must *co-exist* with existing tables, not
replace them, and it carries a discovery checklist (§15) rather than pretending to
know the starting point. A greenfield account simply skips the adoption steps.

### 2.1 What self-service means here

Self-service is a design property of this architecture, not an aspiration
attached to it. It has three tiers, each with a defined default path:

| Tier | Who | What they can do alone | Guardrail |
|---|---|---|---|
| **Consume** | Anyone in the company | Get an answer from certified marts, via an agent or SQL | Certified surface only; no raw SQL for non-technical users |
| **Model** | Any analyst | On already-onboarded data: build/extend `core` and marts, add metrics | CI contract check; shared-code changes need review |
| **Certify** | Any analyst | Propose a new mart, metric, or domain | Certification requires one non-author approval |

The platform's job is to make the *default path* the *safe path*. Every tier
above works only because the machinery beneath it — the ingestion runtime, the
domain template, the CI contract, the generated context — removes decisions
rather than adding them.

**What this deliberately does not claim:** that a new source or a new domain is
analyst-self-service. **New-source onboarding and new-domain standup are DPE work**
(§2.3): source onboarding involves connector/auth/integration and the raw contract;
domain creation involves SLA, ownership, grants, and schema policy. The realistic
commitment is: **once the DPE has onboarded a source and stood up a domain, an
analyst models and certifies within it alone.** (In the `analyst-only` parameter,
source onboarding returns as a caveated stretch goal, not a default promise.)

### 2.2 Environment parameters — the blueprint dial [PARAM]

Set these per adopting environment. Each has a default; the body references the
parameter rather than the value. Wrong values change sizing or sequencing, not the
shape.

| Parameter | Default | Consumed by |
|---|---|---|
| **Team composition** (analyst-only / analysts + DPE) | analysts + DPE | §2, §2.3, §6, operability claims, §11 |
| **Serving engine** (Redshift / Snowflake / Databricks SQL / Athena-Trino / …) | Redshift (reference) | §8.1, §9, §10.4, §10.5, §10.10, cost model |
| **Redshift form** (provisioned / Serverless / mixed) — *only when engine = Redshift* | mixed (adopt brownfield as-is + Serverless for new) | §10.4.1, cost model |
| **Agent-catalog pattern** (A write-through / C generate-then-load) | C | §10.10, §10.10.1 |
| **OpenMetadata hosting** (self-hosted / managed) | self-hosted (owned by DPE) | §10.12, cost model, §11 |
| **Cadence floor** (daily / hourly / sub-hourly) | hourly | §9.1, §10.2, §10.4.1 |
| **Identity provider present** (yes / no) | yes, federated | §10.10, O6 |
| **Compliance regime** (none / formal) | none formal | §10.6, §10.7 |
| **Domain list** | Marketing, Sales, Support, Finance, Product | §9.2, O7 |
| **Monthly cost ceiling** | low-end + documented upgrade path | O3, §10.12 |

### 2.3 Roles and ownership [CALL]

The default team splits responsibilities so that self-service survives and the
heavy infrastructure has a clear owner:

- **Data Platform Engineer (DPE)** owns the **platform/infra plane**: self-hosted
  OpenMetadata, MWAA + VPC, Terraform, the lake and Iceberg maintenance, the
  ingestion runtime, erasure execution, and the Phase-1 standup. The DPE also owns
  **tool integration** — the connective tissue between dbt, Airflow, Redshift,
  OpenMetadata and the lake, the context pipeline into OpenMetadata, the governed
  service wiring, and CI — and **data onboarding**: bringing a new source in
  (ingestion config, connector/auth, raw contract, and the load into staging).
- **Analysts** own the **modeling self-service tier** (§2.1): building and
  extending `core`/marts and metrics on already-onboarded data, and
  proposing/certifying marts and metrics.

Regardless of who operates it, three things hold, because they are what let the
platform survive turnover: the §5 anti-goal "a platform only its author can
operate," the **runbooks** (§10.11.1), and **everything-as-code** (§10.8). A single
DPE is still a single point of failure (§14), so analyst-operable runbooks are a
requirement, not a nicety.

For the **`analyst-only`** parameter value, the DPE's duties fall back to the
platform owner, OpenMetadata flips to **managed** (§10.12), the operability caveats
re-tighten, and new-source onboarding becomes a caveated stretch goal.

---

## 3. Goals

1. Every source lands in a replayable, immutable raw form before any
   transformation touches it.
2. Changing a dataset's refresh cadence — within the cadence floor (§9.1) — is a
   schedule change, not an engineering project.
3. There is exactly one definition of any business concept, and it is
   reviewable in git.
4. The platform is operable from runbooks rather than tribal knowledge: the DPE
   runs the platform plane, and analysts run their domain work, each from
   documented procedures.
5. Once a source and domain exist, an analyst models and certifies within the
   domain without the architect or DPE.
6. Non-technical users can get trustworthy answers from an AI agent without
   raw SQL access, with an audit trail.
7. Every production change is a reviewed commit. No console clicks.
8. **One automated catalog** (OpenMetadata) serves humans and agents from the
   same content, populated only by the gated CI path (§10.10).

## 4. Non-goals

- Dashboards and BI tooling. The platform's contract ends at certified marts.
- Real-time streaming (sub-minute). Micro-batch is the floor.
- Multi-region or active-active.
- Running more than one serving engine at once. One engine per instantiation
  (the **serving-engine parameter**, §2.2 / §8.1); this is not a cross-engine
  federation layer, and migrating off an existing engine is a separate project.
- Buying a commercial ELT vendor or semantic layer. (Catalog hosting is a
  parameter: self-hosted OpenMetadata by default, managed as the no-DPE fallback.)
- Enterprise-grade data mesh, federated governance, or domain team autonomy.

## 5. Anti-goals

These describe a platform that *works* and is still a failure:

- **A platform only its author can operate.** If the runbooks don't let the DPE
  (or, after turnover, their successor) and the analysts operate their planes, the
  design failed regardless of uptime.
- **An agent that answers confidently and wrongly.** Worse than no agent.
- **Two truths for one metric** — one in Redshift, one in a lake table, one in
  a spreadsheet, one typed into the catalog UI. Any duplication that isn't
  explicitly temporary is a defect.
- **A cadence change that requires touching ingestion code.** This is the
  specific failure the design exists to prevent.
- **A catalog that becomes a second source of truth.** Definitions live in dbt
  YAML in git; the catalog is a *projection*, written only by CI (§10.12).
- **A framework so abstract that adding a domain means learning a DSL.**

---

## 6. Constraints

| Constraint | Consequence |
|---|---|
| **[DECIDED]** Build, don't buy — MWAA is a sanctioned managed exception; managed OpenMetadata is a second one, used only under the `analyst-only` parameter | Connectors and semantic layer are built in-house. Vendor surface stays small; maintenance burden stays internal, owned by the DPE. |
| **[DECIDED]** Everything as code | The repository is the only path to production. |
| **[DECIDED]** Trunk-based development, PRs open to the data team | Requires small PRs, fast CI, and a real revert path. |
| **[DECIDED/PARAM]** One serving engine per instantiation (§2.2); Redshift is the reference and is often already present | The lake sits underneath the engine, not beside it. A brownfield engine is adopted, not migrated (§15); the engine's behaviours are pinned by the contract (§8.1). |
| **[DECIDED/PARAM]** Team is a small analyst group plus a DPE (default); `analyst-only` is supported | The DPE operates the platform plane; analysts operate domain work from runbooks. Self-service is a design property (§2.1), not a training programme. |
| **[DECIDED]** No dashboards in scope | Certified marts + semantic definitions are the platform's public interface. |

---

## 7. Reference profile — sizing assumptions

The profile the document is sized against, to stay concrete. It is an illustration,
not the target audience (§2).

**[ASSUMPTION]** — validate in discovery; wrong values change sizing, not shape.
(Dials that are meant to vary live in §2.2, not here.)

| Assumption | Value |
|---|---|
| AWS accounts | 2 (dev, prod) |
| Region | 1 |
| Data team size | 2–4 analysts + 1 DPE |
| Source count | 10–20 |
| **Largest table** | **~100 GB–1 TB compressed; the design holds unchanged to ~10 TB per table** |
| **Daily landed volume** | **< 100 GB/day** |
| **Query concurrency** | **20–50 concurrent BI/agent users at peak** |
| Freshness | daily default; a small number at the cadence floor (§9.1) |
| BI tool | none today; may be adopted later |
| Existing Redshift state | unknown — see §15 |

**On capacity units.** Row counts are deliberately not the unit of measure here.
What drives this architecture is **bytes scanned, delta size per load, and query
concurrency** — not row count. A 500M-row narrow table and a 50M-row wide table
have completely different cost profiles. The design is sized for tables in the
hundreds of gigabytes, and remains correct at low terabytes; beyond roughly 10
TB per table, external tables stop being an exception and become the default
(§9.1).

---

## 8. Architecture overview

```
Sources            Landing (S3 + Iceberg)      Serving (Redshift)      Context + Consumption
─────────          ───────────────────────      ──────────────────      ─────────────────────
SaaS APIs   ─┐     raw_<source>                 staging (native)        analysts (SQL)
Operational ─┼ ingest ─ append-only          ─> core        ──────────> governed service
  databases  │     partitioned by arrival        marts_<domain>              └─> MCP adapter ─> agents
AWS services ┘     time-travellable               (external = exception)
       │                   │  ▲                         │                 OpenMetadata (single catalog)
       │                   │  └── load task (COPY+MERGE)┘                      ▲      │
       └── orchestrated ───┴── MWAA (Airflow) ──────────┘                      │      └─> humans (browse/search)
                               │                                               │
                         dbt-core + MetricFlow                                 │
                               │                                               │
              manifest.json ───┴──> CI context pipeline (gate) ────────────────┘
                                     (Pattern A push, or Pattern C bundle+load)
```

Three planes, deliberately separate:

- **Data plane** — S3/Iceberg (Apache Iceberg table format) for raw and history,
  the serving engine (reference: Redshift) for hot serving.
- **Control plane** — Airflow for scheduling, dbt for transformation logic.
- **Context plane** — generated from the dbt manifest in CI, and projected into a
  **single catalog (OpenMetadata)** that serves both humans (browse/search) and
  agents (via the governed service).

The separation is the point: context never becomes a second source of truth,
and the lake never becomes a second warehouse.

### 8.1 Engine-agnostic core and the serving-engine contract

**[CALL, D17]** The serving engine is a **parameter** (§2.2); the portable spine is
**S3/Iceberg + dbt + Airflow + CI**. What the platform requires of *any* serving
engine is pinned by this capability contract. **Redshift is the reference
implementation**, and the body's concrete mechanisms (Spectrum in §9, ANALYZE/VACUUM
in §10.4.1, the Data API and WLM in §10.10, dynamic data masking in §10.5) are its
*reference realization* — swap in the mapping row below for another engine; the
*requirement* is unchanged, only the mechanism differs.

An engine qualifies if it provides:

| # | Required capability | Needed by |
|---|---|---|
| 1 | ANSI SQL + a maintained dbt adapter | `core`/`marts` are dbt models (§10.3) |
| 2 | Managed "native" tables + bulk load from the lake | hot serving + the load task (§9, §10.4) |
| 3 | External / federated query over S3/Iceberg | the exception path (§9.1) and the raw→staging read |
| 4 | Column-level masking + role-based grants | PII enforcement, department policy (§10.5, §10.10) |
| 5 | A programmatic query API with result/size limits | the governed agent service (§10.10) |
| 6 | Statistics + a maintenance model (manual or auto) + table-health signals | §10.4.1 |
| 7 | Concurrency isolation (queues / warehouses / slots) | agents must not starve analysts (§10.10) |
| 8 | Automated backup/snapshot + time-travel or equivalent | recovery and the erasure blast radius (§10.7) |

**Reference mapping** (confirm against current vendor docs before building; these move):

| Capability | Redshift (reference) | Snowflake | Databricks SQL | Athena / Trino |
|---|---|---|---|---|
| Native hot tables + load | native tables; COPY | native tables; COPY / Snowpipe | Delta tables; COPY INTO | Iceberg tables; CTAS / INSERT |
| Lake external/federated read | Spectrum | external tables | Unity external / Delta | native query-in-place |
| Masking + grants | dynamic data masking + GRANT | masking policies + RBAC | column masks + Unity Catalog | Lake Formation + Trino rules |
| Programmatic query API | Redshift Data API | SQL API | SQL Statement API | Athena API / Trino client |
| Maintenance | ANALYZE/VACUUM (provisioned) or auto (Serverless) | auto | auto (OPTIMIZE) | Iceberg compaction |
| Concurrency isolation | WLM queue / workgroup | virtual warehouse | SQL warehouse | Trino resource groups |
| Backup + time travel | snapshots; limited time travel | Time Travel | Delta time travel | Iceberg snapshots |

Engines that auto-manage statistics and compaction (Snowflake, Databricks) shrink
§10.4.1 to monitoring — a simplification, not a redesign.

**One engine per instantiation.** The parameter is chosen once per environment; a
migration between engines is a separate project, eased by raw and history living in
the engine-neutral lake. **Staying on AWS:** Redshift, Athena/Trino, and Databricks
on AWS keep the whole platform on AWS; Snowflake runs on AWS too; BigQuery is listed
for adopters who extend beyond AWS, at the cost of cross-cloud movement from the S3
lake.

---

## 9. Storage and zones

**[CALL]** Four logical layers, named by purpose rather than by SQL complexity.
Layers differ by *ownership and guarantees*, and each must be deterministically
rebuildable from the one below.

| Layer | Location | Contents | Guarantees |
|---|---|---|---|
| `raw_<source>` | S3 + Iceberg | Append-only landing. One dataset per source object. | Replayable, time-travellable; mutated only by sanctioned erasure (§10.7) |
| `staging` | engine-native | Typed, deduplicated, current-state relation per source. No business logic. | 1:1 with source, key uniqueness, freshness tested |
| `core` | engine-native | Conformed dimensions, business rules, joins across sources | Documented grain, referential tests |
| `marts_<domain>` | engine-native | Star schemas per domain. The only thing consumers touch. | Certified (§10.3) |

**Why raw lives only in S3:** it is the replay source. If raw is duplicated into
the serving engine, a cadence change or backfill has two possible origins and no way
to tell which is authoritative. **[DECIDED]** Raw is never bulk-duplicated into the
serving engine.

**Why staging is native and materialized by default.** If staging were a view
over an external table, every `core` and mart build would re-scan raw through the
engine's external-table / federated-query path over the lake (reference: Redshift
Spectrum; capability 3, §8.1) — which contradicts the cost argument that justifies
the lake in the first place (§9.1). So the default is an explicit **load task**: an
Airflow task reads raw deltas and writes native staging tables via bulk load +
MERGE (reference: COPY). dbt then transforms native-to-native, which is cheap and
fast.

This means the pipeline has a real **EL step** (extract in Lambda/Glue, load
into staging) and a **T (transform) step** (dbt). dbt is not the only writer, and
that is deliberate: it keeps lake scans bounded to the delta.

**The load path is itself a lake-scan consumer — costed, not hidden.**
Most engines cannot bulk-load the Iceberg table format with zero scan: reading raw
deltas into native staging goes through an external / federated read over the lake
(reference: a Redshift external (Spectrum) table, or Athena; on other engines,
external tables or a query-in-place engine), billed per TB scanned. The
"seconds-long load because delta ≪ table" argument holds **only** with
good arrival-time partition pruning; at the cadence floor this read fires up to
96×/day/source. The raw→staging mechanism is an explicit design choice per source
(external-table read vs. UNLOAD/extract vs. a job writing load-ready Parquet), and
its per-load scan cost is wired to scan-cost alerting (§10.11, §14).

View-backed staging over external tables remains available as an exception for
cold sources that are queried rarely and never joined.

**Why names encode purpose:** `source / aggregated / verified` — a pattern
commonly seen in the wild — states neither ownership nor guarantees. "Verified"
is a claim, not a contract. Names must answer: who owns this, what does it
promise, should an agent query it?

### 9.1 Iceberg's role, its limits, and the cadence floor

**[CALL]** Iceberg covers raw, history, and erasure. Native engine tables (reference:
Redshift) cover hot serving. External/federated tables over Iceberg (reference:
Redshift Spectrum) are a **documented per-table exception**, not a default path.

Rationale: incremental load cost scales with *delta* size, not table size. A
100 GB table with a 500 MB delta is a seconds-long bulk load plus a MERGE. External
lake scans (reference: Spectrum) are billed per TB, get no distribution or sort
keys, and carry higher latency. Putting hot, frequently-joined data behind external
tables buys lake query performance with warehouse operational complexity.

Legitimate exceptions, recorded in model metadata:
- cold or archival data beyond Redshift retention,
- very wide fact tables where queries touch a small column/row slice,
- datasets that must be readable by a non-Redshift engine,
- tables beyond roughly 10 TB, where the COPY cost genuinely dominates.

Iceberg additionally earns its place through **erasure** — row-level deletes
make "delete subject X everywhere" tractable in a way plain Parquet does not.

**The cadence floor [CALL, PARAM].** "Cadence is a config change" (Goal 2, D4)
holds comfortably **to the cadence-floor parameter (§2.2), default hourly.**
Sub-hourly is **not** a schedule flip: a table that loads every 15 minutes has no
low-traffic maintenance window, generates continuous unsorted rows and small-file
sprawl, and needs a *different physical pattern* — continuous auto-optimization
(Serverless) or a dedicated near-real-time micro-batch lane with its own compaction
cadence. Treat sub-hourly as a per-table decision with its own cost line, not a
property of the whole platform.

*Verify current Redshift/Iceberg capability limits (write support, row-level
deletes, time travel) against current AWS documentation before implementation, and
pin them once the commit engine is chosen (§10.1). These evolve quickly.*

### 9.2 What a domain is

A **domain** is a business function with one named accountable owner and a set
of certified marts. Domains are the unit of schema naming (`marts_<domain>`),
DAG families, Terraform modules, and certification ownership.

Typical set at this size is the domain-list parameter (§2.2): Marketing, Sales,
Support, Finance, Product.

A domain should be broad enough that its marts answer real questions, and narrow
enough that one owner can certify its definitions. **If certifying a domain
needs two owners, it is two domains.** Domain boundaries are organisational
decisions, not technical ones — the list is a parameter and belongs in
discovery (§15).

---

## 10. Components

### 10.1 Ingestion

**[CALL]** One declarative ingestion runtime, three source classes, one contract.
New-source onboarding is DPE work (§2.3).

If each source gets a bespoke script, the team owns ten small products, each
with its own pagination, rate limits, auth rotation, schema drift, and backfill
bugs. Instead: **a source is a configuration file in git.**

| Source class | Mechanism | Notes |
|---|---|---|
| SaaS APIs | Lambda runtime driven by declarative config | Pagination strategy, rate limits, auth reference, watermark field |
| Operational databases | Glue job, watermark or CDC (change data capture) | Prefer watermark batch; CDC only where delete/update fidelity is required |
| AWS-native | Event-driven Lambda | EventBridge / S3 events into the same raw contract |

**The per-source config schema [CALL] — a defined contract, not left to the
builder.** Because the runtime *parses* this file, the config grammar and the
runtime are one deliverable; the "new source = one file" promise rests on it. The
v0 schema (extend per source class, keep required keys stable):

```yaml
source: zendesk                 # unique id; drives raw_<source> and stg naming
class: saas_api                 # saas_api | operational_db | aws_native
schedule: "@hourly"             # cadence; must be >= cadence floor (§9.1)
auth:
  secret_ref: secretsmanager://data/zendesk   # resource ref only; never a value
extract:
  pagination: cursor            # cursor | page | offset | none
  rate_limit_per_min: 700
  watermark_field: updated_at   # source watermark for incremental pulls
  page_size: 100
schema:
  declared_columns: [...]       # expected source columns (drives drift policy)
  primary_key: [ticket_id]      # becomes the staging MERGE key
retention:                      # authored here for raw/staging (non-dbt layers)
  raw_days: 1825
  staging_days: 60
```

**The raw contract** — every ingestion writes these columns regardless of class.
Physical types are fixed so the drift gate and the load task are deterministic:

| Column | Type | Meaning |
|---|---|---|
| *(source payload)* | **one `super`/JSON column `_payload`** (default), or exploded typed columns per the source's `schema.declared_columns` | the source record |
| `_ingest_ts` | `timestamptz` | when the platform received it |
| `_batch_id` | `varchar(64)` | idempotency key; see below |
| `_source_ts` | `timestamptz` (nullable) | the source's own watermark, where available |

**[CALL]** Default payload representation is a single semi-structured column; a
source promotes to exploded typed columns when it is hot and frequently filtered.
The choice is recorded in the source config and determines how the drift policy
reads "a new column."

**`_batch_id` idempotency [CALL] — enforced at the commit job.** Writes to raw are
append-only, and deduplication of *records* happens in staging (never in raw). The
apparent contradiction with "re-running a batch is a no-op" is resolved by scope:
idempotency is a **batch-level** guarantee enforced by the single commit job
(§below), which refuses to append a `_batch_id` already present in the raw table's
committed snapshots. Re-running an extract that produced batch *B* is a no-op
because the commit job skips *B*; record-level dedup is still staging's job.

**Schema drift policy [CALL, logged as C39]:** additive changes (new columns) are
accepted automatically and flagged. Type changes and column removals fail the run
loudly and require a reviewed config change. **Renames are caught, not laundered:**
because the config declares `schema.declared_columns`, a run that simultaneously
drops a known column and adds an unknown one is flagged as a *possible rename* for
human review — not silently accepted as "additive," which is the failure mode that
produces wrong answers months later. The loud failure is owned by the commit job,
so extraction and gate ownership are not split.

**Committing to Iceberg [CALL] — single writer, chosen engine.** Lambda and Glue do
not commit Iceberg directly. Concurrent commits from many small invocations produce
lock contention and small-file sprawl. Instead: ingestion writes immutable Parquet
objects to a staging prefix, and **one commit job** appends them to the raw table.
**The commit engine is chosen, not left open:** default is a Glue/Spark Iceberg
writer (for row-level-delete + compaction support required by erasure, §10.7);
Athena `INSERT INTO` is the lightweight alternative where erasure is out of scope.
The chosen engine is where the Iceberg capability check of §9.1 is pinned.

**Two writers, acknowledged [CALL].** Compaction, orphan cleanup, and snapshot
expiry are *also* writers to the raw table (§10.4.1). Iceberg's optimistic
concurrency means a maintenance commit retries when ingestion lands. For
daily/hourly tables this is fine (maintenance in the quiet window); for sub-hourly
tables, compaction is scheduled per-table and partition-scoped to avoid the live
partition.

**Connector risk, named once:** connector maintenance is where build-not-buy
platforms die, because SaaS vendors change APIs without warning. Mitigations,
designed in from the start: a contract test per source, drift alerting, a
documented runbook per source, and idempotent writes so a re-run is always safe.
Owner: the DPE.

**Backfill [CALL].** SaaS backfills of full history will not fit a single 15-minute
Lambda. Backfill is **chunked and watermark-resumable** (self-re-invoking Lambda or
a Glue job), and the **watermark store** (a small control table, not an Airflow
Variable) is locked so a backfill resetting the watermark cannot race live
incremental runs into a skip/dup. Operational-DB backfills use Glue.

### 10.2 Orchestration

**[DECIDED]** MWAA (Managed Workflows for Apache Airflow) is the single
orchestrator. Step Functions was considered and dropped — a second orchestrator is
a second thing to upgrade and a second place to look when something breaks. (The
one place a second primitive is tempting is chunked SaaS backfill, §10.1; it is
handled by self-re-invocation instead.)

**[CALL]** Acquisition, load, transformation, and maintenance are scheduled
**independently**. One DAG family per domain for transformation; one per source
class for acquisition; one maintenance DAG. This is what makes cadence a
contract property *within the cadence floor* (§9.1): acquisition cadence and model
cadence move without touching each other.

Operational facts to bake into Terraform from day one (DPE-owned):
- MWAA must live in private subnets in a VPC — networking, NAT, and VPC
  endpoints (S3 gateway, Secrets Manager, Redshift) are part of the build.
- Floor cost is roughly **$350–400/month per environment** before a single task
  runs; with a real dev environment that is **~$700–800/month across dev+prod**
  (see cost model, §10.12).
- Pin the MWAA Airflow version in Terraform; the DAG factory (§10.9) and dbt
  `state:modified` selection depend on it.
- Airflow orchestrates; it does not extract. Extraction lives in Lambda/Glue.

### 10.3 Modeling

**[DECIDED]** dbt-core, orchestrated by Airflow, developed in GitHub PRs. No
dbt Cloud (build-not-buy). dbt-core plus the `dbt-metricflow` package supplies
metric compilation for the agent layer without a commercial semantic layer.
**Pin the dbt-core + MetricFlow version** (the manifest schema and `mf` behaviour
are version-specific), and run the `run_metric` feasibility spike in Phase 1
(§10.10, §11), not Phase 3.

**[CALL]** Certification is the central concept. A **mart** is certified only
when it carries all of:

| Requirement | Enforced by |
|---|---|
| Owner | contract checker |
| Grain, stated explicitly | contract checker |
| Description of the model and its columns | contract checker |
| Primary key tests (`unique` + `not_null`) | dbt test |
| Referential tests to upstream dimensions, **where the mart declares a dimension FK** | dbt test |
| Freshness SLA (structured, §below) | dbt source freshness / custom test |
| PII classification on every column | contract checker |
| Retention declaration | contract checker |

**The single authoritative contract checker [CALL, C35].** There is **one** checker
(`platform/contracts/`), run in CI, and the context pipeline (§10.10) *calls the
same checker* rather than re-implementing it — so there is one place certification
is defined and enforced, not two. Three rules it applies:
- **An unclassified column blocks certification.** This single rule keeps PII and
  retention from becoming a later hardening pass that never happens.
- **Referential test required only when the mart declares a foreign key to a
  dimension** (a mart with no dimension FK legitimately has none) — this avoids both
  false blocks and false passes.
- **`pii_class` is a closed enum** (`email | name | financial | health`); a value
  outside the set fails the check.

**Structured freshness SLA [CALL, C34].** The SLA is structured fields, not prose,
so the freshness alert can compute "stale beyond SLA":
`freshness: { cadence: daily, deadline: "07:00", tz: UTC }`. The human-readable
"daily by 07:00 UTC" is derived from it.

**Metrics are certified separately from marts [CALL].** MetricFlow is canonical
for *metrics*; marts are canonical for *entities, facts, and dimensions*. A mart
exposes facts and dimensions — it does **not** expose a pre-aggregated `revenue`
column competing with the metric definition. **A metric's certification field set
[C36]:** `owner`, `description`, `grain` (the entity/time grain it is valid at),
`type`, and at least one validity test (e.g. not-null / bounded range on the
compiled result for a fixture). These fields are mirrored in the Appendix A
example.

This is what prevents the §5 anti-goal: one definition of revenue, compiled by
the semantic layer, traced to marts that supply its inputs.

Certification is a promise to the organisation, so it requires one non-author
approval. Everything else can be self-merged.

### 10.4 Serving

**[CALL]** Native engine tables (reference: Redshift) for hot data; external schema
for documented exceptions only. Marts are per-domain star schemas, and they are the *only*
surface BI and agents are pointed at.

dbt performs **T only**. `staging` tables are created by the load task's
declarative schema (§9); `core` and `marts` come from dbt materializations.
Zero hand-written DDL, and zero DDL outside those two paths.

**One definition of the staging contract [CALL, C40].** The load task's
declarative schema and dbt's `sources.yml` for staging would otherwise be two
hand-maintained truths. The dbt `sources.yml` for staging is **generated from the
load-task schema** in CI, so the load output and dbt's view of it cannot drift.
("Zero hand-written DDL" is honest about relocation: the load task's declarative
schema *is* the DDL, authored once and generated into both places.)

#### 10.4.1 Scheduled maintenance — provisioned and Serverless paths

*This section is the Redshift **reference realization** of capability 6 (§8.1).
Engines that auto-manage statistics and compaction (Snowflake, BigQuery, Databricks)
reduce it to the monitoring + health-report steps; the requirement — stats are
current, tables stay healthy, and every action is logged — is unchanged.*

**[CALL, C29]** Redshift maintenance is a scheduled, monitored DAG — not a hope
that background auto-vacuum keeps up. It runs daily in the low-traffic window,
after the last load and before the first report. **The DAG has two paths, selected
by the Redshift-form parameter (§2.2), because the two engines are not the same
system** (resolve O5 before writing it):

**Provisioned path** (brownfield cluster):

| Step | Action | Trigger condition |
|---|---|---|
| 1. Analyze | `ANALYZE` on tables with stale statistics | `SVV_TABLE_INFO.stats_off` above threshold |
| 2. Vacuum | `VACUUM` for sort; `VACUUM RECLUSTER` for interleaved sorts | Unsorted % above threshold, table above size floor |
| 3. Health report | Skew, unsorted rows, missing stats, dist-key warnings from `SVV_TABLE_INFO` and `STL_ALERT_EVENT_LOG` | Always; posted to Slack |
| 4. Snapshot check | Verify automated snapshots exist and retention matches policy | Always |
| 5. Iceberg maintenance | Compaction, orphan cleanup, snapshot expiry (§10.1) | Always |
| 6. Compression review | `ANALYZE COMPRESSION` recommendations | Monthly; applied as a reviewed migration, never automatically |

**Serverless path** (new workgroups): vacuum/analyze/table-optimization are
auto-managed and manual `VACUUM`/interleaved `RECLUSTER` are largely unavailable,
so this path **shrinks to monitoring + targeted `ANALYZE` + health reporting**,
reading `SYS_*` monitoring views (not the deprecated `STL_*` tables), plus steps 4
and 5 unchanged. Most "maintenance" here is "trust auto-optimization and alert on
skew."

Principles (both paths):
- **Missing statistics cause the worst query plans and are invisible** until
  something is slow. Analyzing is cheaper than diagnosing.
- Vacuum (provisioned) only above a size floor — small tables cost more to vacuum
  than they save.
- Every action the job takes is logged, so a slow query can be traced back to a
  maintenance run.
- Maintenance failure is **P2**, not P1, unless it blocks a load or a certified
  mart's SLA (§10.11).

### 10.5 Access control

**[CALL]** Access is a **Phase 1** deliverable, not a later hardening pass.
Certification requires PII classification (§10.3), and a certified mart with no
roles is reachable only as an admin.

- Department-level engine roles, declared in git.
- Column-level grants for sensitive fields, plus engine-native dynamic data masking
  (reference: Redshift DDM; Snowflake masking policies; BigQuery policy tags;
  Databricks column masks).
- Lake Formation tags for lake-side access.
- Grants applied by one declarative module reading metadata — not by hand.
- **Drift detection** on grants, because as-code without drift alerts means "it
  was code once." (Phase 2.)

**Access-module input schema [CALL, part of C35].** The module consumes a declared
map of `department → {roles, mart/domain grants}`, plus a **`pii_class` → masking
policy** table: which `pii_class` values are masked for which department roles.
The Appendix "masked unless support role" behaviour is the output of this table,
not an ad-hoc rule.

*Redshift RBAC (role-based access control)-as-code is genuinely awkward: dbt's
`grants` config is coarse. The recommended shape is a small declarative access
module run as a job, rather than spreading grants across model configs.*

### 10.6 PII

**[CALL]** Classification is column-level metadata in dbt YAML
(`pii: true`, `pii_class: email | name | financial | health`), authored in git.
Classification for the first domain's columns is **Phase 1**; org-wide automated
detection is Phase 2.

- **Detection** (Macie, Glue PII scans) *proposes* classifications; a human
  confirms and it lands in git. Automated detection is a check, never the
  source of truth.
- **Enforcement** in three places: Redshift masking + column grants, Lake
  Formation tags, and exclusion/masking at the agent layer (§10.10).
- **Structured outputs:** agents never receive raw PII in query results, enforced
  by the engine (§10.10 masking). **Free-text is best-effort, stated honestly:**
  a user can paste a customer's email into a question, and the audit log captures
  prose; a scrubber reduces but does not guarantee removal of PII from free-text
  prompts and answers. This is a residual risk, not a guarantee (§10.10, §14).
- **Certification gate:** unclassified columns block certification (§10.3).

### 10.7 Retention and erasure

**[CALL]** Retention is declared metadata, not a cron job. For dbt layers it is
authored in model YAML (`retention_days`, `legal_basis`, `layer`); **for the
non-dbt layers `raw_<source>` and `staging` it is authored in the source config
(§10.1)**. One enforcement job reads all declarations and acts.

| Layer | Typical retention | Mechanism |
|---|---|---|
| `raw_<source>` | long (1–7 years) | Iceberg snapshot expiry, partition deletes, S3 lifecycle |
| `staging` | short (30–90 days) | rebuildable; safe to drop |
| `core` | medium | archival-then-drop |
| `marts_<domain>` | as consumers require | archival-then-drop |

Redshift has no native TTL, so marts need an explicit archival-then-drop job.
That is exactly the kind of job that gets forgotten, which is why it lives in
the same enforcement path as everything else.

**Raw is immutable *except* for sanctioned erasure.** Erasure is a privileged,
audited operation (DPE-executed) that deletes or rewrites the affected rows **and
then expires the snapshots that still contain them** — otherwise the "deleted" data
remains readable through time travel, and the immutability guarantee is a fiction.

**Erasure blast radius [CALL].** "Delete subject X everywhere" is **not** proven by
deleting from raw alone. The erasure runbook enumerates every store that can hold
the subject's PII and the action for each: raw Iceberg; native `staging`/`core`/
`marts`; the **agent audit log** (§10.10); **OpenMetadata** (only if a value leaked
into a description — normally it holds no row data); any Parquet UNLOADs; and —
critically — **Redshift automated snapshots/backups**, which retain deleted rows
until they roll off. The backup-retention lag is recorded as an explicit
**residual-risk window** in the runbook.

**Replay horizon honesty [CALL].** Expiring snapshots to erase subject X removes
time-travel for *all* data in those snapshots, so each erasure ratchets the
historical time-travel horizon forward. The design separates **logical replay**
(deterministic rebuild from *current* raw — unaffected by erasure) from
**historical time-travel** (bounded, and capped by the most recent erasure). Goal 1
means the former; frequent erasure is incompatible with a long time-travel window,
and the doc says so rather than implying both.

- Replay after erasure is bounded to snapshots created after the erasure; the
  procedure records that boundary.
- The domain owner requests; the DPE executes; the action is logged.
- Not automated from a request queue in v1. Phase 1 ships a **documented manual
  runbook**; Phase 4 ships the tested automation.

### 10.8 Everything as code, and delivery

**[DECIDED]** The repository is the only path to production.

As-code surface:

| Surface | Artifact |
|---|---|
| Infrastructure | Terraform |
| DAGs | Python in git |
| Ingestion definitions | Declarative per-source configs (§10.1) |
| Models, tests, contracts | dbt YAML + SQL |
| IAM, Redshift roles, grants | Terraform + declarative access module |
| Lake Formation permissions | Terraform |
| Secret resources and rotation | Terraform (values never in git) |
| CI/CD | GitHub Actions workflows |
| Runbooks | Markdown in git (§10.11.1) |
| Agent context / catalog population | **Generated in CI**, pushed/loaded to OpenMetadata, never hand-edited |

Known friction points, stated rather than hidden: Redshift grants (§10.5), BI
dashboards (out of scope), and secret *values* (never in git — only resources
and rotation).

**[CALL]** Monorepo. A new source touches ingestion config, staging, core, a
mart, a DAG, Terraform, and a grant — one atomic, revertible PR instead of four
coordinated ones. Revisit only when a domain team needs independent release
cadence.

Monorepo does **not** mean monostate: Terraform state is split per environment
and per component. One giant state file turns a typo in a DAG's IAM role into a
plan that wants to touch MWAA.

CI gates, identical for everyone:
- Terraform `plan` on PR, always against dev
- dbt slim CI against a real dev target
- SQL linting
- dbt tests
- secret scanning
- contract check (the single checker, §10.3)
- generated-context freshness check (catalog population matches manifest)
- path-filtered execution behind one required check (§10.8.2), so CI stays under ~10 minutes

**Review policy [CALL]:**

| Change | Requirement |
|---|---|
| Leaf (new mart, source config, test) | Self-merge on green CI |
| Shared (`core/`, `access/`, `infra/`, contract changes) | One non-author approval |
| Certification of a mart or metric | One non-author approval |

**Bus-factor note:** "one non-author approval" on shared code assumes at least two
people competent in `core/access/infra`. With one DPE and junior analysts, that
reviewer pool may be one person; §14 names single-operator SPOF as a High risk, and
the honest mitigation is cross-training plus the runbooks, not the review gate alone.

Trunk-based discipline: small PRs, branches measured in hours, and a revert path
fast enough that reverting is always preferable to hotfixing in the console.

**Deployment model [DECIDED]:**

| Environment | Trigger | Approval |
|---|---|---|
| dev | Merge to `main`, automatic | None — the reviewed PR is the approval |
| prod | **Manual `workflow_dispatch`** targeting prod | GitHub Environment `prod` with a required reviewer |

Prod is never deployed by a merge. The manual workflow runs from `main` (or a
tag on `main`), re-runs the plan, and records the deployed commit SHA in the
workflow summary and in a `deploys` log. Two consequences are designed for
rather than ignored:

1. **`main` and prod drift, by design.** A nightly check compares prod state
   against `main` and posts the drift to Slack; a `prod-deployed` tag marks what
   actually shipped. Slim CI's `state:modified` compares against the **last
   *deployed* manifest** (keyed by that tag), not a possibly-stale prod crawl, and
   the first deploy bootstraps an empty manifest.
2. **Prod deploys must be idempotent and re-runnable.** A failed deploy is
   retried by re-running the same workflow. Concurrency is locked so two prod
   deploys can never overlap.

#### 10.8.1 Repository structure

```
.
├── .github/workflows/
│   ├── ci-required.yml        # always runs; aggregates the single gate
│   ├── pr.yml                 # lint, contract check, dbt slim CI, tf plan
│   ├── dev.yml                # on merge to main: apply + deploy dev
│   ├── deploy-prod.yml        # manual workflow_dispatch -> prod
│   └── scheduled.yml          # nightly full plan, drift + cost checks
├── infra/
│   ├── modules/
│   │   ├── network/           # VPC, subnets, endpoints
│   │   ├── mwaa/              # managed Airflow
│   │   ├── redshift/          # workgroup/cluster, roles, masking
│   │   ├── lake/             # S3, Glue catalog, Lake Formation
│   │   ├── ingestion/         # Lambda runtime, Glue jobs, IAM
│   │   ├── openmetadata/      # catalog host (self-hosted) + its backing stores
│   │   └── domain/            # per-domain scaffolding module
│   └── envs/{dev,prod}/       # separate state per env
├── platform/
│   ├── ingestion/
│   │   ├── runtime/           # the single ingestion runtime
│   │   └── sources/           # one config per source — data, not code
│   │       ├── salesforce.yaml
│   │       └── zendesk.yaml
│   ├── dags/
│   │   ├── factory/           # narrow DAG factory + escape hatch
│   │   ├── acquisition/ transformation/ maintenance/
│   ├── load/                  # raw -> native staging (COPY + MERGE)
│   ├── access/                # declarative roles, grants, masking policy
│   ├── retention/             # declarations + enforcement job
│   ├── registry/
│   │   └── products.yml       # THE registry (§10.9): one line per data product
│   └── contracts/             # the single certification checker + CI checkers
├── dbt/
│   ├── models/{staging,core,marts/<domain>}/
│   └── semantic/              # MetricFlow metric definitions
├── agent/
│   ├── service/               # governed HTTP service (governance + catalog reads)
│   ├── mcp/                   # thin MCP adapter over the service
│   ├── context/               # Pattern C: bundle generator + OpenMetadata loader
│   └── evals/                 # questions with known-correct answers
├── docs/
│   ├── runbooks/              # one per source, plus platform runbooks
│   └── design.md
└── CODEOWNERS                 # core/, access/, infra/ -> required review
```

Rules that keep this structure honest:
1. **A new source is one file** in `platform/ingestion/sources/` plus a dbt
   staging model (generated), plus **one line in `platform/registry/products.yml`**.
2. **A new domain is one folder** under `dbt/models/marts/`, **one `domain/`
   Terraform module instantiation**, and **one registry line** — the §10.9 health
   test. (This is the single, reconciled statement; §10.9 and this list agree.)
3. **Catalog population is written only by CI.** A human edit to generated context
   fails the build.
4. **`platform/contracts/` holds the one checker**, so the guardrails are code.

#### 10.8.2 Path-filtered CI

CI must be path-aware — a docs-only PR should not run Terraform — but naive path
filtering breaks required status checks.

**The trap:** a workflow with `on: pull_request: paths: ['infra/**']` that is
also a required check never starts on a `dbt/`-only PR. The check never reports,
so the PR is blocked forever. Teams then "fix" it by un-requiring the check.

**The pattern [CALL]:** every workflow triggers unconditionally on
`pull_request`; each job decides for itself whether to act using a path-filter
step; and a single `ci-required` job with `if: always()` depends on all of them
and fails if any job that ran failed. Branch protection requires **only**
`ci-required`.

| Job | Runs when |
|---|---|
| `terraform-plan` | `infra/**` — **including `infra/modules/**`**, not just `infra/envs/**` |
| `dbt-ci` | always; model selection uses `state:modified` against the last deployed manifest |
| `lint` | always |
| `contract-check` | `dbt/**`, `platform/contracts/**` |
| `context-generate` | `dbt/**` |

Two rules:
- **Path filters must include module paths.** Filtering only on
  `infra/envs/prod/**` misses a change to a shared module that prod consumes.
- **dbt does not use path filters.** `state:modified` selects affected models
  *and their downstream dependents*, which a path filter cannot see.

**Safety net:** a nightly unfiltered `terraform plan` on both environments.

### 10.9 Modularity and the data product template

**[DECIDED]** Both goals: an analyst models alone *and* the architect/DPE move
fast. These converge only if **the template is the guardrail**.

**[CALL]** Convention over configuration, not configuration over code. Three
distinct mechanisms, and conflating them is the failure:

1. **Conventions** — folder layout, naming, layer rules, schema naming. Zero
   code; enforced by CI and review.
2. **Configuration** — only for things that are genuinely *data*: source
   definitions, schedules, retention, PII, grants. A source definition is data.
   A mart's business logic is not, and must never become a YAML key.
3. **Scaffolding, generated once at creation time** — a template that stamps
   out the DAG, staging models, mart stubs, grants, and Terraform module as
   *real code you then own and edit*. Generate once, never at runtime.

**The trap this avoids:** push configuration far enough and you have built a
DSL (domain-specific language). The DSL becomes a product to maintain, cannot
express the weird 20%, and couples every domain to one generator. This is
framework rot, and it is worse than duplication, because duplication is at least
debuggable.

**The registry and DAG factory — a defined contract [CALL, C41].** The "one
registry line" is a row in `platform/registry/products.yml`:

```yaml
products:
  - name: support_tickets
    domain: support
    kind: mart              # source | mart | domain
    owner: data-team@example.com
    dag: transformation     # which DAG family the factory attaches it to
```

The **DAG factory** consumes this registry: for each entry it emits (or attaches
to) the right DAG family. The factory's input is the registry plus the dbt
selectors for the product; its output is a DAG, with a documented **escape hatch**
for a bespoke hand-written DAG when the factory cannot express a product.

**Health test:** adding a data product touches its own folder plus one registry
line (and, for a domain, one Terraform module instantiation). If it requires
editing shared code, boundaries are wrong. If it requires learning a DSL, the
design is over-abstracted.

**Escape hatches are mandatory.** Every abstraction needs a documented way to
bypass it, or it will be bypassed — undocumented, at 11pm.

Reusable primitives, not a framework: one ingestion runtime; one narrow DAG
factory with a bespoke-DAG escape hatch; dbt macros for repeated SQL shapes; one
grants module reading metadata; one Terraform module per domain.

**Anyone can model; only certification is reviewed.**

### 10.10 The agent layer and the single catalog

**[DECIDED]** A governed HTTP service with a **thin MCP (Model Context Protocol)
adapter** over it. Governance logic is not coupled to a young protocol; if MCP
changes shape, an adapter is rewritten, not the governance layer. The same service
can later back a Slack bot or a scheduled report.

**[DECIDED]** Agents reach certified marts and metrics only. Never raw SQL.

**One automated catalog [CALL, C25].** OpenMetadata is the **single serving
catalog** for both humans (browse/search) and agents (via the governed service).
The decision §10.12 previously made — "agents read git-generated skills, humans
read OpenMetadata" — is replaced by one catalog, with the determinism it required
preserved by separating **source of truth** from **serving**:

- **Source of truth** stays dbt YAML in git + the single contract checker (§10.3).
- **Serving** is OpenMetadata, populated **only by the gated CI path** — never by
  an autonomous crawl of the agent surface.

The governed service reads *definitions* from OpenMetadata and runs *queries* via
the engine's programmatic query API (capability 5, §8.1; reference: the Redshift
Data API) with guardrails and masking.

**Catalog population — a parameter: Pattern A or Pattern C [CALL, C26].**

- **Pattern A — CI write-through (push).** After prod deploy, the CI job pushes
  certified entities (descriptions, grain, owner, PII tags, metrics, lineage) into
  OpenMetadata via its API, stamped with contract-version + deployed SHA.
  OpenMetadata's native pull-connectors are demoted to **discovery of
  uncertified/raw only**. *Pick A for the fewest moving parts.*
- **Pattern C — generate-then-load (hybrid) [default].** CI emits a **versioned
  context bundle** (the gate and a reviewable, diffable artifact); a loader applies
  it into OpenMetadata, which becomes **deterministically rebuildable from git**
  (replay the bundle = DR + drift repair). *Pick C for reproducibility, DR replay,
  pre-land review, or multiple publish targets.*
- **Pattern B — rejected, recorded.** Letting OpenMetadata crawl dbt/Glue/Redshift
  and serve that to agents is rejected: non-deterministic (depends on last night's
  crawl), no atomic version stamp, reintroduces the stale/"table not live" failure,
  and demotes the certification gate to advisory.

**The invariant both A and C hold:** certified catalog content is written only by
the gated CI path; **publish-after-deploy** (the catalog never describes a mart not
yet live); and every agent answer **carries the context version it used**, so a
wrong answer is traceable to a stale catalog state. The context pipeline is also the
**enforcement point**: it calls the single checker and **fails the build** if a
certified mart is missing owner, grain, description, tests, PII classification, or
retention.

**Availability [CALL].** Because the catalog is now on the agent's critical path,
the governed service keeps a **context cache / fallback** so agent context survives
OpenMetadata downtime; a stale-but-known context is served with its version, never
a guess.

**Business nuance** cannot be generated ("revenue excludes intercompany transfers";
"churned means 90 days with no login"). It is authored in git, next to the model,
and compiled into the catalog by the same pipeline.

**Tool surface [CALL]** — deliberately small; each maps onto OpenMetadata entities:

| Tool | Purpose |
|---|---|
| `list_certified` | What exists and is trustworthy |
| `get_metric_definition` | Definition, owner, grain, freshness, caveats, PII |
| `run_metric` | Compile a metric request to parameterized SQL against marts |
| `get_lineage` | Why this number exists |
| `get_freshness` | How old is this |

**Guardrails [CALL]:** row limits, timeouts, cost caps, result-size caps tied to
the **query API's limits** (for the reference Redshift Data API: async
submit-then-poll, ~100 MB result cap, 24h result retention, 100 KB statement size,
account quotas — confirm the chosen engine's equivalents before Phase 3), and
parameterized SQL only. Agent traffic gets its own **concurrency lane** (reference: a
Redshift WLM queue or a separate Serverless workgroup; a dedicated warehouse or slot
pool on other engines) so it does not contend with analysts at peak.

**Identity [CALL].** Policy is **department-level in v1**; per-person row predicates
are out of scope. This assumes a **federated identity provider** (identity-provider
parameter, §2.2 / O6). **Masking is enforced by the engine, not the app [C37]:** the
query-API caller **assumes the caller's department engine role** (per-department
credentials), so engine-native masking and column grants apply to the session —
rather than one service role plus fragile app-layer redaction.

**The raw-SQL escape hatch — controlled, not just "supervised" [CALL, C38].** The
generic engine SQL MCP server (reference: the Redshift MCP server; arbitrary SQL, no
certification/metric/policy notion) stays available to engineers, but with real
controls: a **separate restricted engine role** (no access to PII columns / masked
by default), its queries
**logged to the same audit store**, and access **gated to named engineers** — so it
is not a silent bypass of the whole governance model.

**Hosting [CALL]:** the governed service is Lambda behind the engine's query API
(reference: the Redshift Data API) — out of the VPC, less networking to debug.
(Concurrency is an engine property; see the concurrency-lane note above.)

**Evaluation is not optional.** Without a question set with known-correct answers,
there is no way to know the agent is right. **Evals are structural** (correct tool,
metric, filter, and masking) **plus a small answer set against a frozen fixture** —
never pinned to live production numbers, which change daily.

**Phasing:** the metric-serving agent is Phase 3, but the **`run_metric`
feasibility spike** (Lambda + MetricFlow + Data API, and the compiled-and-cached
vs. live-compute decision for non-additive metrics like medians) runs in **Phase
1**. The cheap v1 is metadata plus a dozen pre-approved parameterized questions.

#### 10.10.1 Catalog population and access, by pattern

Generation (the gate) and serving (the catalog) are separate concerns.

- **Generation lives in the monorepo** (§10.8.1), because that is where the dbt
  manifest lives and because context must version atomically with the models it
  describes.
- **Serving is OpenMetadata.** Under **Pattern A**, CI pushes to it directly; under
  **Pattern C**, CI produces the versioned bundle and a loader applies it. Either
  way, **publish after deploy, never before.**

**Version semantics describe the data contract, not the commit:**

| Bump | Trigger |
|---|---|
| major | A certified mart or metric is removed, renamed, or its grain changes |
| minor | A mart, metric, or column is added |
| patch | Descriptions and metadata only |

Two distinct versions exist and are named as such: the **context/data-contract
version** (this semver, carried on every served answer) and the
**certification-rules version** (the contract checker's own version, §10.3, which
evolves via §10.10.2). They are not the same number.

**Access control.** The catalog names tables, columns, metrics, and PII
classifications, so it is permissioned on the **same department boundaries as the
data** (§10.5), enforced by **OpenMetadata's own policies/RBAC** — a Support user
cannot read Finance definitions. Per-domain scoping, not one global bundle.

**Consistency.** Every answer from the governed service carries the context
version it used, so a wrong answer can be traced to a stale catalog state.

#### 10.10.2 The feedback loop

An agent that detects a mismatch should route it — but not "fix" it directly.

**False assumption 1: the fix belongs in the catalog.** Served context is
generated; a correction applied there is overwritten by the next build. **Every
proposal targets the source**: a dbt YAML annotation, a model description, or a
metric definition.

**False assumption 2: the agent knows which side is wrong.** It does not. A user
contradicting a definition may be mistaken; treating the user as authoritative is
how documentation drifts toward misconceptions.

**Triage by signal type [CALL]:**

| Signal | Actually is | Route |
|---|---|---|
| Column missing, mart absent, metric unknown | Generation failed — a pipeline bug | **P2 alert**, not a PR |
| Definition incomplete — an unwritten caveat | Genuine metadata gap | Issue with evidence |
| Definition contradicts the business | Business decision | Domain owner, with evidence |

**Proposal, not patch.** The default output is a **structured feedback record**,
written to a queryable store (a dedicated Redshift table, so records cluster with
the data they describe):

```
feedback_record:
  skill_or_domain, context_version, metric_or_mart,
  claim_text, conversation_id, observed_result,
  proposed_change, content_hash   # dedup key: hash(metric + claim + proposed_change)
```

An agent may open a **draft PR** only when it can produce both a constrained diff
and evidence: target `dbt/models/<domain>/**` only (never `access/`, PII
classification, or certification metadata); description/annotation fields only;
always human-reviewed; one PR per distinct claim, deduplicated by content hash.

**User-facing wording matters:** *"this looks possibly out of date — should I flag
it for review?"*, not *"this is wrong, should I fix it?"*

**The loop improves the contract.** Clustering feedback records by mart/metric
shows which annotation is repeatedly missing — which is how the certification
contract (§10.3) learns what to require next.

### 10.11 Observability and alerting

Checks that page a human with a runbook, not a dashboard nobody watches:

- **Freshness** — per certified mart, against its structured SLA (§10.3)
- **Volume** — row counts outside expected bounds
- **Schema drift** — breaking changes fail loudly (§10.1)
- **Pipeline SLA** — DAG duration against budget
- **Test failures** — dbt test results, surfaced to the owning domain
- **Grant drift** — declared vs actual
- **Scan cost** — raw→staging Spectrum/Athena scan above budget (§9)
- **Table health** — unsorted rows, stale stats, skew (§10.4.1)
- **Catalog freshness** — context version served vs. last deployed

**Routing [DECIDED]:** Slack is the default and the system of record for
everything non-urgent. PagerDuty carries P1s only.

| Severity | Examples | Route |
|---|---|---|
| **P1** | Certified mart stale beyond SLA; prod deploy failed and not rolled back; grant drift on sensitive columns; critical source down for a whole domain | **PagerDuty** |
| **P2** | Single source failing with retries exhausted; dbt test failure on a non-certified model; maintenance DAG failed; cost anomaly above threshold | Slack `#data-platform` |
| **P3** | Schema drift detected and auto-accepted; freshness warning at 80% of SLA; compaction backlog | Slack, domain channel |

**On-call reality [CALL].** The default team has a DPE but **no 24×7 on-call
rotation**. So the honest recovery SLA for an off-hours P1 is **next business
morning**, which bounds how tight any freshness SLA can credibly be. PagerDuty is
reserved for the small set the DPE has agreed to be woken for; if no one is on
call for a given window, that window's P1s are "first thing," and the freshness
SLAs are set accordingly. Do not advertise a 15-minute mart's recovery as faster
than the staffing supports.

Rules that keep this from rotting:
- **Every alert links a runbook.** An alert without one is noise.
- **P1 must stay rare.** If PagerDuty fires weekly, it gets muted.
- **The Slack thread is the investigation record.**
- Acknowledging in Slack does not clear PagerDuty; only resolution does.

#### 10.11.1 The runbook contract

**[CALL]** Runbooks are the artifact that makes the platform operable beyond its
author, so they have a required shape, not just a folder. Every runbook
(`docs/runbooks/`) has:

1. **Symptom** — what the alert says / what was observed.
2. **Confirm** — how to verify it is real (the query or check to run).
3. **Triage steps** — ordered actions, each safe to run.
4. **Escalation** — who/what next if triage fails.
5. **Revert** — how to get back to a known-good state.

One runbook per source (DPE-authored) and one per platform concern (freshness,
maintenance, erasure, deploy failure, catalog-down). A worked stub ships in Phase 1.

### 10.12 OpenMetadata — the single catalog

**[CALL, revises C9]** OpenMetadata is the **single serving catalog** (§10.10),
**promoted to Phase 1** critical path (it was a Phase-2 read-only projection). It
provides what git cannot: a browsable cross-system lineage graph, ownership and
stewardship, PII workflows, and search for non-technical people — *and* it serves
the agent context, via the gated CI pipeline.

It is **never a source of truth**. Definitions live in dbt YAML in git; the catalog
is populated only by CI (Pattern A or C). A description edited only in the
OpenMetadata UI is a second truth — blocked by access policy for governed fields and
caught by the nightly catalog-vs-manifest drift check.

**Hosting [PARAM, C27]:** **self-hosted by default, owned by the DPE.**
Self-hosting is a JVM service + an OpenSearch cluster + a Postgres instance — the
heaviest single component in the stack — so its operability (patching, OpenSearch
ops, upgrades, backups) is a **Phase-1 obligation on the DPE**, not a background
task. Under the **`analyst-only`** parameter, hosting flips to **managed**
(e.g. Collate), a second sanctioned buy-exception under D2, because a team with no
DPE cannot absorb that load.

**Cost, stated plainly (feeds O3).** Beyond the MWAA floor (~$700–800/mo across
dev+prod, §10.2): NAT gateway (~$32/mo + per-GB) and interface VPC endpoints
(~$7/mo each × AZs); raw→staging Spectrum/Athena scans per load (§9); and
self-hosted OpenMetadata (JVM + OpenSearch + Postgres), realistically **$150–300+/mo
plus operational burden**. Honest all-in floor across both accounts is roughly
**$900–1,400/mo before compute** — not the ~$400 a single-MWAA figure implies.
Managed OpenMetadata trades some of that burden for a subscription.

---

## 11. Phasing

| Phase | Deliverable | Exit criterion |
|---|---|---|
| **1** | DPE stands up: landing zone; ingestion runtime for 2–3 sources; load task; MWAA in VPC; **self-hosted OpenMetadata as the single catalog** (pattern A or C); dbt with one domain to certified marts; access module + PII classification for that domain; maintenance DAG (correct path per Redshift form); Terraform for two accounts; CI/CD; manual erasure runbook; **`run_metric` feasibility spike**; the runbook contract + one stub | One domain's numbers are trusted by a real consumer, browsable in the catalog |
| **2** | Grant drift detection; org-wide PII detection; retention enforcement automation; second domain | An analyst models and certifies in an existing domain without the DPE |
| **3** | Governed service + MCP adapter reading the catalog; metric catalog; eval set; audit log | A non-technical user gets a correct answer without asking a human |
| **4** | Automated erasure; drift and cost hardening; catalogue stewardship; maintenance tuning | Platform runs for a month without the architect's intervention |

**Phase 1 is where the project most likely dies** — on Iceberg/MWAA/VPC/Terraform/
OpenMetadata standup, *before the first certified mart*. That is why the DPE owns
Phase 1 and why it is deliberately one domain, end to end. The "operable by
analysts" property is a **steady-state** claim, not a build-time one: building
Phase 1 needs the DPE for roughly a quarter even though day-two domain work does
not. To de-risk, an adopter may start raw-in-Parquet (§17) and defer Iceberg
erasure machinery and the DAG factory to the second domain.

Note the ordering constraint: **access and PII classification cannot be deferred
past Phase 1**, because certification requires them and a certified mart with no
roles is reachable only as an admin.

---

## 12. Decision log

### Decided by the platform owner

| # | Decision |
|---|---|
| D1 | Reference architecture for a Data Platform as Code on AWS (Redshift serving engine, S3/Iceberg lake); environment-neutral, sized against a reference profile (§7); small analyst team **plus a Data Platform Engineer** (DPE default; `analyst-only` is a supported parameter, §2.2) |
| D2 | Build not buy; MWAA a sanctioned managed exception; managed OpenMetadata a second exception, used under `analyst-only` |
| D3 | Step Functions dropped |
| D4 | Cadence is a contract property within the cadence floor (§9.1); acquisition and transformation scheduled independently; raw always replayable |
| D5 | Daily freshness by default; tighter freshness earned per dataset, bounded by the cadence floor |
| D6 | Iceberg for raw/history; native engine tables for hot serving; raw never bulk-duplicated into the serving engine |
| D7 | Trunk-based development; data team may open PRs |
| D8 | Everything as code; repo is the only path to production |
| D9 | Agentic consumption by non-technical users is in scope |
| D10 | Dashboards and BI tooling out of scope |
| D11 | Governed HTTP service with a thin MCP adapter |
| D12 | Both onboarding goals: analyst modeling self-service and architect/DPE speed |
| D13 | Target is the whole organisation, delivered through one vertical slice |
| D14 | Alerting: Slack primary, PagerDuty for P1 only; no 24×7 on-call by default |
| D15 | Prod deploys are manual via `workflow_dispatch`; dev deploys automatically on merge |
| D16 | **One automated catalog (OpenMetadata) serves humans and agents; source of truth stays dbt-in-git** |
| D17 | **Engine-agnostic: the serving engine is a parameter (§2.2) bound by a capability contract (§8.1); Redshift is the reference implementation; the portable spine is S3/Iceberg + dbt + Airflow + CI** |

### Architect's calls

| # | Call | Reversible? |
|---|---|---|
| C1 | Layer names: `raw_<source>` / `staging` / `core` / `marts_<domain>` | Cheaply |
| C2 | One declarative ingestion runtime with per-source configs | Cheaply |
| C3 | External tables are a documented per-table exception | Cheaply |
| C4 | dbt-core + MetricFlow (pinned); no dbt Cloud | Moderately |
| C5 | Context generated and gated in CI; **served via the single catalog**, never hand-edited | Cheaply |
| C6 | Retention and PII as declared metadata with one enforcement job | Cheaply |
| C7 | Monorepo, path-filtered CI, CODEOWNERS, split Terraform state | Expensively |
| C8 | Two AWS accounts, one region; Serverless for new workgroups, existing cluster adopted as-is until a separate migration decision | Moderately |
| C9 | OpenMetadata role — **superseded by C25/C27**: now the single catalog, Phase 1 | — |
| C10 | Four-phase delivery, one domain first | Moderately |
| C11 | Tiered review: leaf self-merge, shared and certification need approval | Cheaply |
| C12 | Convention over configuration; scaffold once, never runtime generation | Expensively |
| C13 | Governed service on Lambda + the engine query API (reference: Redshift Data API) | Cheaply |
| C14 | Agents see certified marts and metrics only | Cheaply |
| C15 | Staging is native and materialized by a load task; dbt performs T only | Moderately |
| C16 | Single-writer Iceberg commits; compaction and expiry owned by the maintenance job; chosen commit engine | Cheaply |
| C17 | MetricFlow canonical for metrics; marts canonical for entities/facts/dimensions | Moderately |
| C18 | Raw immutable except sanctioned erasure, which also expires containing snapshots | Cheaply |
| C19 | Department-level policy; per-user row predicates out of v1 | Moderately |
| C20 | Access module and PII classification in Phase 1 | Cheaply |
| C21 | Redshift maintenance as a scheduled, monitored DAG (§10.4.1) | Cheaply |
| C22 | Capacity expressed in bytes scanned and delta size, not row count | — |
| C23 | Catalog population published after prod deploy, per-domain permissioned | Cheaply |
| C24 | Agent feedback produces a structured record by default; draft PRs target dbt sources only, constrained to annotations, never auto-merged | Cheaply |
| C25 | **OpenMetadata is the single serving catalog for humans and agents; source of truth stays dbt-in-git (serving separated from truth)** | Moderately |
| C26 | **Catalog population is a parameter: Pattern A (write-through) or C (generate-then-load, default); B rejected** | Cheaply |
| C27 | **OpenMetadata self-hosted by default (DPE-owned); managed under `analyst-only`** | Moderately |
| C28 | **DPE owns infra, tool integration, and data onboarding; analysts own modeling self-service (revises D1)** | Moderately |
| C29 | **Maintenance DAG split into provisioned and Serverless paths** | Cheaply |
| C30 | **Cadence floor: cadence-as-config holds to hourly; sub-hourly is a distinct physical pattern** | Cheaply |
| C31 | **Per-source config schema is a defined contract (out of "left to the builder")** | Cheaply |
| C32 | **Raw physical schema defined: metadata column types + payload representation** | Moderately |
| C33 | **`_batch_id` idempotency enforced at the commit job (reconciles append-only)** | Cheaply |
| C34 | **Structured freshness SLA** | Cheaply |
| C35 | **One authoritative contract checker; required-referential rule; closed `pii_class` enum; access-module + masking-policy schema** | Cheaply |
| C36 | **Metric-certification field set, mirrored in Appendix A** | Cheaply |
| C37 | **Masking enforced by per-department engine role assumed via the query API** | Moderately |
| C38 | **Raw-SQL escape hatch gets a restricted role, audit logging, and named-engineer gating** | Cheaply |
| C39 | **Schema-drift policy (additive accept / rename flag / type+removal fail) — logged** | Cheaply |
| C40 | **Staging `sources.yml` generated from the load-task schema (single definition)** | Cheaply |
| C41 | **Registry (`products.yml`) + DAG-factory input contract defined; §10.9 and §10.8.1 reconciled** | Cheaply |

---

## 13. Open decisions

These are unresolved. The document is honest about them rather than guessing.
(Values meant to vary per environment are **parameters** in §2.2, not open
decisions.)

| # | Question | Why it matters | Default if unresolved |
|---|---|---|---|
| O1 | **Which domain is the vertical slice?** | Determines the first schema, first consumers, and the shape of the conformed layer | Selection criteria in §13.1; no domain assumed |
| O2 | **What drives retention and PII — legal regime, or cost and tidiness?** | A formal regime requires evidence, documentation, and audit trails | Compliance-regime parameter (§2.2) default: none formal; enforcement only |
| O3 | **Monthly infrastructure ceiling?** | Decides Serverless sizing and whether self-hosted OpenMetadata is realistic vs. managed | Costed per §10.12 (~$900–1,400/mo before compute); upgrade path documented |
| O4 | Whether any source needs view-backed external staging | Affects cost and latency for that source only | Native load for all sources |
| O5 | **Adopt the existing Redshift as-is, or migrate to Serverless?** | Drives Terraform, cost, migration, **and which §10.4.1 maintenance path applies** | Adopt as-is; migration is a separate decision |
| O6 | Is there a federated identity provider? | If not, Phase 3 grows by a user-mapping project | Identity-provider parameter (§2.2) default: yes |
| O7 | What are the domains? | Sets schema, DAG, Terraform, and ownership structure | Domain-list parameter (§2.2) |
| O8 | **Pattern A or Pattern C for catalog population?** | Moving-parts vs. reproducibility/DR trade-off (§10.10) | Pattern C |

### 13.1 Domain selection criteria (for O1)

Rank candidate domains on:

1. **Clear grain** — the entity is unambiguous.
2. **Tolerant of iteration** — a wrong number embarrasses, not audits.
3. **Real consumer, today** — someone will use it within weeks.
4. **Small source count** — two or three sources, not twelve.
5. **Low political ambiguity** — nobody disputes the definition.

Finance is usually the worst first choice: highest stakes, most definitional
disagreement, least tolerance for a first-draft schema. Support and Marketing
usually score better.

---

## 14. Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Connector maintenance overwhelms the team | High | One runtime, contract tests, drift alerts, per-source runbooks; DPE-owned |
| Agent gives confident wrong answers | High | Certified-only surface, eval set, audit log, PII redaction (structured) |
| Access layer bypassed with a shared admin role | Medium | First-class module, drift detection, column grants, per-dept role at the agent |
| Raw-SQL escape hatch bypasses governance | Medium | Restricted role, audit logging, named-engineer gating (C38) |
| Framework rot in the scaffolding | Medium | Scaffold once; escape hatches; health test in §10.9 |
| Cadence creep — everything becomes sub-hourly | Medium | Cadence floor (C30); SLA is declared metadata; cost review per dataset |
| Scope explosion into "whole org at once" | High | One domain first; expansion is onboarding work |
| Metadata drifts from reality | Medium | Context generated + gated in CI; catalog written only by CI; nightly drift check |
| Catalog becomes a second source of truth | Medium | Serving separated from truth; UI edits blocked/caught; rebuildable from git (Pattern C) |
| Self-hosted OpenMetadata operability load | High | DPE-owned Phase-1 obligation; managed fallback under `analyst-only` |
| OpenMetadata down takes agent context down | Medium | Context cache/fallback in the governed service (§10.10) |
| Single operator (DPE) is a SPOF | High | Runbooks for analysts and successor; everything-as-code; cross-training; phased exit criteria |
| Iceberg small-file sprawl / rising scan cost | Medium | Single-writer commits; compaction/expiry in maintenance DAG; scan-cost alerting |
| Load-path Spectrum scan cost underestimated | Medium | Explicit raw→staging mechanism per source; scan-cost budget + alert (§9) |
| Redshift performance degrades silently | Medium | Scheduled analyze/vacuum (provisioned) or auto-opt monitoring (Serverless) + daily health report |
| Maintenance model mismatched to Redshift form | Medium | Two-path maintenance DAG (C29); resolve O5 before writing it |
| "Cadence is free" vs. "maintenance is nightly batch" | Medium | Reconciled by the cadence floor (§9.1): sub-hourly is a distinct pattern |
| PagerDuty fatigue / off-hours P1 with no on-call | Medium | Narrow P1 list; recovery SLA honest to staffing (§10.11); every alert links a runbook |
| Identity federation turns out not to exist | Medium | Validated in discovery (O6) before Phase 3 |
| Prod drifts from `main` because deploys are manual | High, by design | Nightly drift check; `prod-deployed` tag; slim CI vs. last deployed manifest |
| Erasure incomplete across stores / backups | Medium | Blast-radius runbook enumerates every store; backup-retention residual window recorded |
| Erasure erodes the replay/time-travel horizon | Medium | Logical replay vs. historical time-travel separated and stated (§10.7) |
| Feedback loop launders user misconceptions | Medium | Proposals not patches; evidence required; human review; never auto-merge |
| Shared-code review pool is one person | Medium | Cross-training + runbooks; review gate alone is insufficient (§10.8) |

---

## 15. Discovery checklist

Run this before applying the architecture. Each answer sets a parameter (§2.2) or
changes sizing/sequencing, not the shape.

**Serving engine (reference: Redshift)**
- [ ] **Which engine is in place or chosen? (sets the §2.2 parameter and the §8.1 mapping)**
- [ ] Does it meet the §8.1 capability contract? Confirm the query API, masking, and external-lake read.
- [ ] Curated warehouse or dumping ground?
- [ ] What loads it today — scripts, an ELT vendor, manual bulk load?
- [ ] *(Redshift)* Serverless or provisioned? (sets the §10.4.1 path and O5) Current size and cost?
- [ ] How many schemas, and do they map to any layer convention?
- [ ] Who has admin? Shared credentials in use?
- [ ] Table sizes in GB, and the largest table's growth rate
- [ ] Engine health signals (reference Redshift: `SVV_TABLE_INFO` / `SYS_*` — unsorted %, stale stats, skew)

**Sources**
- [ ] Full inventory: name, class, auth method, volume, update frequency
- [ ] Which support incremental extraction (watermark or CDC)?
- [ ] Which APIs have rate limits or pagination quirks that will hurt?

**Consumers**
- [ ] Who queries Redshift today, and with what tool?
- [ ] Which reports would break if a table were renamed?

**Team**
- [ ] **Is there a Data Platform Engineer, or is this `analyst-only`?** (sets §2.2)
- [ ] Who can review shared `core/access/infra` changes?

**Domains and ownership**
- [ ] Which business functions exist, and who would own each domain?
- [ ] A named business stakeholder per candidate domain?

**Identity**
- [ ] Which identity provider do staff use today? Federation into AWS in place?
- [ ] How do analysts authenticate to Redshift today?

**Governance**
- [ ] Compliance regime in force?
- [ ] Where does PII exist today, and is it classified?
- [ ] Existing retention or erasure obligations?

**Catalog**
- [ ] **Pattern A or C (O8)? Self-hosted or managed OpenMetadata (§2.2)?**

**AWS**
- [ ] Account structure, Control Tower or landing-zone constraints
- [ ] Existing VPC layout and whether MWAA + OpenMetadata can sit in it
- [ ] Existing Terraform state and module conventions

---

## 16. Left to the builder

Deliberately not specified here, because a competent builder should decide them
with the code in front of them:

- Exact Terraform module boundaries below "one per domain"
- Lambda memory/timeout sizing and Glue worker configuration
- dbt project structure within a domain folder
- Threshold values for analyze/vacuum triggers in §10.4.1
- Log and metric retention for the platform's own observability
- The specific metric catalog contents for Phase 3

*(The per-source config schema is **no longer** here — it is a defined contract,
§10.1 / C31.)*

---

## 17. What this document deliberately does not claim

- That Iceberg is necessary at this scale. It is justified by replay, history, and
  erasure — not by volume. An adopter at smaller volume may reasonably start
  raw-in-Parquet and defer the erasure machinery.
- That the phased plan is fast. It is sized for a small team, and it will take
  months. Phase 1 needs the DPE for roughly a quarter.
- That a team without a DPE delivers this easily. The `analyst-only` parameter is
  supported but shifts the platform plane onto the owner and flips OpenMetadata to
  managed; it is the harder road.
- That "self-service" means zero support. It means the default path is safe and
  documented for the tier that owns it, not that nobody ever asks a question.
- That every serving engine is equally cheap or simple. The capability contract
  (§8.1) is portable; the per-engine cost, concurrency model, and operability differ,
  and the cost model (§10.12) is written against the Redshift reference.
- That sub-hourly freshness is free. Above the cadence floor it is a distinct
  physical pattern with its own cost.

---

## Appendix A — Worked example: manifest to catalog

Illustrative, trimmed to one model. Field names follow dbt's manifest schema; the
exact schema version changes between dbt releases, so **pin and confirm against the
version in use** (§10.3).

### A.1 `target/manifest.json` (emitted by every `dbt build`)

```json
{
  "metadata": {
    "dbt_schema_version": "https://schemas.getdbt.com/dbt/manifest/v12.json",
    "dbt_version": "1.9.x",
    "generated_at": "2026-11-25T06:12:44Z",
    "project_name": "analytics"
  },
  "nodes": {
    "model.analytics.mart_support_tickets": {
      "unique_id": "model.analytics.mart_support_tickets",
      "resource_type": "model",
      "schema": "marts_support",
      "name": "mart_support_tickets",
      "config": {
        "materialized": "table",
        "tags": ["support", "certified"],
        "contract": { "enforced": true },
        "meta": {
          "owner": "data-team@example.com",
          "domain": "support",
          "grain": "one row per support ticket",
          "freshness": { "cadence": "daily", "deadline": "07:00", "tz": "UTC" },
          "retention_days": 730,
          "certified": true
        }
      },
      "description": "One row per support ticket, with SLA outcome and resolution time.",
      "columns": {
        "ticket_id": {
          "name": "ticket_id", "description": "Primary key.", "data_type": "varchar",
          "meta": { "pii": false },
          "constraints": [{ "type": "not_null" }, { "type": "primary_key" }]
        },
        "customer_id": {
          "name": "customer_id", "description": "FK to core_dim_customer.",
          "data_type": "varchar", "meta": { "pii": false },
          "constraints": [{ "type": "foreign_key", "to": "core_dim_customer" }]
        },
        "requester_email": {
          "name": "requester_email", "description": "Email of the requester.",
          "data_type": "varchar", "meta": { "pii": true, "pii_class": "email" },
          "constraints": []
        },
        "first_response_minutes": {
          "name": "first_response_minutes",
          "description": "Minutes to first human reply. Null if never answered.",
          "data_type": "integer", "meta": { "pii": false }, "constraints": []
        }
      },
      "depends_on": {
        "nodes": ["model.analytics.core_dim_customer", "model.analytics.stg_zendesk__tickets"]
      }
    }
  },
  "metrics": {
    "metric.analytics.support_first_response_minutes": {
      "name": "support_first_response_minutes",
      "type": "median",
      "description": "Median minutes from ticket creation to first human reply.",
      "meta": {
        "owner": "support-ops", "certified": true,
        "grain": "ticket", "validity_test": "result >= 0"
      },
      "depends_on": { "nodes": ["model.analytics.mart_support_tickets"] }
    }
  }
}
```

(The metric `meta` carries the full C36 field set — `owner`, `grain`, `type`,
`description`, `validity_test` — so the one worked example satisfies the same
certification rule the body states.)

### A.2 `target/run_results.json` (the trust signal)

```json
{
  "results": [
    { "unique_id": "test.analytics.unique_mart_support_tickets_ticket_id.a1b2", "status": "pass" },
    { "unique_id": "test.analytics.not_null_mart_support_tickets_ticket_id.c3d4", "status": "pass" },
    { "unique_id": "test.analytics.relationships_mart_support_tickets_customer_id.e5f6", "status": "pass" }
  ]
}
```

(For brevity three of the mart's passing tests are shown; the generated catalog
entry's test count is computed from the full `run_results.json`, not the excerpt.)

### A.3 `target/catalog.json`

Produced by `dbt docs generate`. Supplies column types and row counts measured
against the **prod** manifest/catalog at publish time (not dev's), so counts
reflect what is live.

### A.4 Generated catalog content (served via OpenMetadata)

Under **Pattern C** this is the versioned bundle a loader applies to OpenMetadata;
under **Pattern A** the same content is pushed via the OpenMetadata API. Human
browsing and the agent's `get_metric_definition` read the identical entity.

```markdown
<!-- GENERATED from dbt manifest @ abc1234, context v2.3.0, deployed SHA abc1234. Do not edit. -->

# Support · mart_support_tickets — analytics.marts_support.mart_support_tickets
One row per support ticket, with SLA outcome and resolution time.

- Grain: one row per support ticket
- Owner: data-team@example.com
- Freshness: daily by 07:00 UTC (last built 2026-11-25 06:40 UTC)
- Retention: 730 days
- Tests: computed from run_results (e.g. 14 passing, 0 failing)
- Volume: measured from prod catalog.json

| Column | Type | Meaning | PII |
|---|---|---|---|
| ticket_id | varchar | Primary key | no |
| customer_id | varchar | FK to core_dim_customer | no |
| requester_email | varchar | Email of the requester — masked unless support role | email |
| first_response_minutes | integer | Minutes to first human reply; null if never answered | no |

Caveats (authored): tickets migrated before 2024-03 have no first-response
timestamps and are excluded from SLA metrics.

## Metric · support_first_response_minutes
Median minutes from ticket creation to first human reply.
Owner: support-ops · Grain: ticket · Built from: mart_support_tickets

## Not certified — do not answer from these
- stg_zendesk__comments — raw text, uncertified, contains PII

## How to answer
Call run_metric for numbers. Do not write SQL. If a question needs a column not
listed above, say so rather than approximating.
```

### A.5 Field mapping

| Manifest field | Becomes |
|---|---|
| `config.meta.certified` + `tags` | whether the entity appears at all |
| `description`, `columns[].description` | the prose and the column table |
| `config.meta.grain`, `owner`, `freshness`, `retention_days` | the bullet list |
| `columns[].meta.pii_class` | the PII column and the masking note |
| `columns[].constraints` (foreign_key) | triggers the required referential test (C35) |
| `depends_on` + `parent_map` | lineage answers from `get_lineage` |
| `metrics` (incl. `grain`, `validity_test`) | the metrics section (C36) |
| `run_results.json` statuses | the passing/failing count |
| `catalog.json` (prod) | types and row counts |

### A.6 The generation gate

The build **fails**, it does not warn, when a model tagged `certified` is missing
any of: `owner`, `grain`, a description, a primary-key test, a referential test
*where it declares a dimension FK*, a `pii` classification on any column, or
`retention_days`; or when a certified metric is missing any C36 field. The **same
single checker** (§10.3) runs both in the PR `contract-check` and in the context
pipeline — one definition, enforced in one place. That gate is what makes
"generated" mean "true" rather than "recently generated."

Authored content — caveats, business nuance, synonyms — is written by a human in
the dbt YAML (`description`, `meta`) and compiled into the catalog. It is never
written into the catalog UI.

---

## Appendix B — Glossary

| Term | Expansion / meaning |
|---|---|
| CDC | Change data capture — streaming row-level inserts/updates/deletes from a source DB |
| DPE | Data Platform Engineer — owner of the platform/infra plane (§2.3) |
| DSL | Domain-specific language — a custom mini-language; §10.9 treats building one as a failure mode |
| EL / ELT / T | Extract-Load / Extract-Load-Transform / Transform step |
| Iceberg | Apache Iceberg — the open table format over S3 used for raw/history/erasure |
| MCP | Model Context Protocol — the protocol the thin agent adapter speaks (§10.10) |
| MWAA | Amazon Managed Workflows for Apache Airflow — the single orchestrator |
| PII | Personally identifiable information |
| RBAC | Role-based access control |
| RLS | Row-level security — per-row access predicates; out of v1 scope (§10.10) |
| Spectrum | Redshift Spectrum — Redshift's external-table query engine over S3 |
| SLA | Service-level agreement — here, a structured freshness promise (§10.3) |
| WLM | Workload management — Redshift's query-queue/concurrency mechanism |
