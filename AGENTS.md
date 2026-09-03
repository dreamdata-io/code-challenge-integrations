# Candidate working agreement

Read [`ASSIGNMENT.md`](ASSIGNMENT.md) before changing code. It defines the goal, timebox, deliverables, submission checklist, and review format.

This file is the canonical working agreement and concise service reference for humans and coding agents working on the challenge.

## Required documentation discovery

Before planning or changing code, read all candidate-facing specifications in this order:

1. [`ASSIGNMENT.md`](ASSIGNMENT.md) — scope, required CLI outcomes, exact NDJSON envelope, deliverables, and review expectations.
2. [`docs/crm-api.md`](docs/crm-api.md) — authoritative HTTP resources, schemas, relationships, collection query/response rules, deleted records, errors, and retry contract.
3. [`docs/local-mock.md`](docs/local-mock.md) — authoritative executable startup, flags, readiness, process-local scenario lifecycle, reset behavior, and troubleshooting.
4. This `AGENTS.md` — implementation boundaries and a concise cross-document working reference.

Start with the root [`README.md`](README.md) to select the supplied platform binary. If this summary conflicts with the detailed HTTP API, follow `docs/crm-api.md`; for executable behavior and flags, follow `docs/local-mock.md`. If the documents do not settle required behavior, ask rather than guessing or relying on an undocumented endpoint.

## Working boundaries

- Build a Go CLI source connector; do not modify or replace the supplied `mock-crm` executable.
- Implement all six entities: `company`, `contact`, `lead`, `deal`, `pipeline`, and `stage`. They share one collection protocol and are realistic within the roughly **3–5 hour** guidance scope. Unrelated polish is explicitly de-emphasized.
- Keep the implementation within the challenge scope; focus on correctness, safety, and reasoning.
- Prefer the Go standard library. Adding a small number of justified dependencies (such as an OpenAPI 3.0 validation library) is acceptable.
- Do not add a blob store, cloud deployment, mock authentication service, or dependency on evaluator-only behavior.
- Treat IDs as opaque UUIDs. Do not infer fixture ordinals, ordering, or business meaning from UUID text.
- Use only the documented candidate-facing API. There is no debug, reset, or evaluator-oracle endpoint.
- Keep credentials out of source, logs, commits, generated files, and AI-use notes.
- The candidate submits from a new private repository under their own GitHub account, not only from the supplied repository, adds `paddie` as a collaborator (Read role where available), then emails `prm@dreamdata.io` with the repository URL and submitted commit hash.

## Required CLI contract

The CLI accepts:

```text
--entity company|contact|lead|deal|pipeline|stage
--validate
--spec <path>
```

- `--entity` selects the singular entity name.
- `--validate` is an optional boolean flag (default `false`) enabling runtime OpenAPI validation.
- `--spec` is an optional string flag specifying the OpenAPI 3.0 specification path (default `openapi.yaml`).

A run with no supplied state performs a full pull. The candidate chooses and documents how a prior emitted state value is supplied; a run with that state performs an incremental pull.

### Validation contract

When `--validate` is enabled:
- Load and structurally check the selected OpenAPI specification before any network activity. Missing or invalid specifications report an actionable stderr diagnostic and exit non-zero immediately.
- For each fetched page, validate the complete HTTP operation response and then validate each entity record against its component schema (`#/components/schemas/{Entity}`) before emitting records from that page.
- Validation failures report actionable context to stderr, exit non-zero, and emit no checkpoint that advances beyond input state.
- Validate all six entities against the supplied `openapi.yaml`. When discrepancies between live payloads and `openapi.yaml` are discovered, correct the specification (in place or via `--spec`) and document findings in a concise drift-remediation note.

Stdout contains NDJSON protocol messages only. Human-readable logs and diagnostics go to stderr.

### Record messages

```json
{"type":"record","record":{"id":"…","changed_at":1735689600000,"deleted":false,"name":"…","domain":"…","lifecycle":"lead"},"timestamp":"2025-01-01T00:00:00.000Z","entity":"company","id":"…"}
```

All five fields are required:

- `type` is exactly `"record"`.
- `record` is the complete raw API item without normalization, including all fields of an updated live or deleted record (companies expose integer `changed_at` and neither `created_at` nor `updated_at`; other entities retain `created_at` and `updated_at`).
- `timestamp` is normalized to UTC RFC 3339 with millisecond precision (converted from integer `changed_at` for companies; equal to `updated_at` for other entities).
- `entity` is the selected singular entity name.
- `id` equals the API item's `id`.

Do not emit ID-only records, inferred deltas, or stripped records. Record-message line order is unspecified.

### State messages

```json
{"type":"state","value":{"candidate_defined":"state"}}
```

At least one state message is required on every successful run. The JSON `value`, checkpoint cadence, and prior-state input mechanism are candidate decisions. State must be sufficient for the documented next invocation to perform an incremental pull.

A successful complete sync exits 0. A failed sync exits non-zero, and its stdout is a partial stream that must not be treated as a completed result.

## Running the supplied service

Select and start the matching executable using the copy/paste commands in the root [`README.md`](README.md), then check readiness:

```sh
curl --fail http://localhost:8080/healthz
```

Normal mode has no artificial request limit or injected failures. Resilience is required of the connector; add `--enable-fault-profile=true` to the matching startup command to test its retry behavior.

At startup, `mock-crm` exposes a deterministic materialized current state for the fixed 30 UTC days before `--simulation-start`. HTTP requests only observe it. Repeated full pulls and inclusive incremental pulls remain unchanged until the operator presses Enter on an empty line between completed pulls. Enter simulates one UTC day and atomically publishes a complete successor state across operational resources. Press Enter only between pulls; consistent pagination across a publication is intentionally unsupported.

For changing resources, derive the next checkpoint from the greatest change cursor safely emitted (`changed_at` epoch milliseconds for companies, `updated_at` RFC 3339 for other resources). After Enter publication, a new inclusive pull at that checkpoint returns current boundary ties and records touched during the simulated day. A refreshed full pull reads the same current state. Pipelines and stages are fixed reference data.

Keep one mock process alive across the full pull, Enter advancement, and incremental client invocation to preserve one scenario. This is not a connector-architecture requirement: the client may restart freely. Restarting `mock-crm` rematerializes the initial state and discards operator advancement and fault state. Discard client checkpoints from the old process after a server restart. Checkpoints from older mock versions or from before the company-native clean-break schema change are explicitly invalid.

`/healthz` checks readiness without changing simulation state or triggering optional faults. See [`docs/local-mock.md`](docs/local-mock.md) for the complete lifecycle contract.

## CRM API

The sole data endpoint is:

```text
GET /v1/{resource}?page=P&size=S&since=T&order=asc|desc
```

Plural resources are `companies`, `contacts`, `leads`, `deals`, `pipelines`, and `stages`.

- `page` is zero-based and defaults to `0`.
- `size` defaults to `100`; valid values are 1–500.
- `since` is optional and inclusive. For `companies`, it accepts non-negative decimal Unix epoch milliseconds; for other resources, it uses an RFC 3339 UTC timestamp with millisecond precision.
- `order` is `asc` by default. For `companies`, it orders by `(changed_at, id)`; for other resources, it orders by `(updated_at, id)`. In both cases, `desc` reverses the complete tuple.
- A successful response is `{"data":[...]}` with integer `next_page` only when another page exists.
- Continue from page zero until `next_page` is absent. Timestamp ties can cross page boundaries.
- Live and deleted records occur in the same stream and both participate in filtering, ordering, and pagination.

Pagination itself does not skip or repeat boundary items. Inclusive `since` means exact boundary records may be replayed between separate runs. A safe checkpoint comes from records already made recoverable and must not pass incomplete tied records.

Malformed or repeated query values and unknown query keys return JSON `400`; unknown paths return `404`; unsupported methods return `405`.

### Required retry behavior

The connector must retry `429` and all `5xx` responses by repeating the identical page request. With the fault profile, `/v1` injects `429` or `503`; both include:

```text
Retry-After-Ms: <positive integer>
```

The JSON body contains the same positive integer as `retry_after_ms`. Treat the header as milliseconds and wait at least that long. For a `5xx` without the header, use a documented bounded backoff. Exhausted retries fail the sync without publishing successful completion state. Standard `Retry-After` is not part of this API, and fault cadence is intentionally unspecified.

## CRM vocabulary and relationships

- A **company** is an external account. It has `name`, a synthetic `domain`, and lifecycle `lead`, `opportunity`, or `customer`.
- A **contact** is an external person associated with one company. It has `label`, `company_id`, and the company's lifecycle.
- A **lead** is a potential-customer record with `label` and nullable `company_id`.
- A **deal** belongs to one company. It has `label`, decimal-string `amount`, zero to three same-company `contact_ids`, and canonical `pipeline` and `pipeline_stage` strings.
- A **pipeline** is fixed reference data with a canonical `name`.
- A **stage** is fixed reference data with globally unique `name`, canonical `pipeline`, one-based `position`, and category `open`, `closed_won`, or `closed_lost`.

Company/contact lifecycle is distinct from a deal's pipeline stage and follows aggregate deal state: any historical win makes a live account a customer; otherwise open business makes it an opportunity; otherwise it is a lead. Every live contact matches its company. A company may have multiple deals in different pipelines.

Later simulated days can add companies with related contacts and first-stage deals. Open deals progress forward one ordered stage at a time without changing pipeline, eventually reach a live won/lost terminal stage, and then stop changing. Deal outcomes can emit complete company/contact lifecycle updates. Company deletion cascades complete deleted records to all of its contacts, associated leads, and deals; cascaded deals are deleted rather than moved to a lost stage.

Each Enter press atomically publishes related changes across companies, contacts, leads, and deals. Each collection exposes only the latest current record per ID (live or deleted), so a client that misses several simulated days does not receive superseded intermediate versions. This business simulation does not change independent collection retrieval, inclusive checkpoints, paging, deleted-record schemas, or the required NDJSON output.

The fixed pipelines and ordered stages are:

| Pipeline | Ordered stages |
| --- | --- |
| `new-business` | `new-business-qualified`, `new-business-discovery`, `new-business-proposal`, `new-business-negotiation`, `new-business-won`, `new-business-lost` |
| `renewals` | `renewals-upcoming`, `renewals-engaged`, `renewals-review`, `renewals-negotiation`, `renewals-renewed`, `renewals-churned` |
| `partner-sales` | `partner-sales-registered`, `partner-sales-qualified`, `partner-sales-co-selling`, `partner-sales-contracting`, `partner-sales-won`, `partner-sales-lost` |

## Validation and documentation

For all six entities, validate one full pull followed by at least 10 incremental simulation ticks while the same mock process remains alive. Verify that after remediating specification discrepancies, runs with `--validate` succeed cleanly across both initial full pulls and the 10 incremental simulation ticks. For each tick, press Enter once between completed pulls, wait for publication confirmation, then invoke the connector with that entity's prior emitted state. Several entities may be retrieved after each Enter press. Fixed `pipeline` and `stage` resources may legitimately return only inclusive boundary replays.

Document:

- exact build, mock startup, full-run, incremental-run, and validated-run commands;
- the corrected OpenAPI specification (`openapi.yaml` or alternate file selected via `--spec`);
- a concise schema drift-remediation note;
- the prior-state input mechanism;
- non-obvious dependencies;
- consequential recovery/checkpoint decisions as context, decision, and consequence; and
- concise AI use and verification notes in `AI_USE.md`.

No separate blob inspection, screenshot, video, prescribed test framework, `make validate` target, or prototype-evidence package is required.

For the complete service contracts, return to [`docs/crm-api.md`](docs/crm-api.md) and [`docs/local-mock.md`](docs/local-mock.md); do not treat this summary as a replacement for them.
