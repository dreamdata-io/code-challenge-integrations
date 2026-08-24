# Candidate working agreement

Read [`ASSIGNMENT.md`](ASSIGNMENT.md) before changing code. It defines the goal, timebox, deliverables, submission checklist, and review format.

This file is the canonical working agreement and concise service reference for humans and coding agents working on the challenge.

## Required documentation discovery

Before planning or changing code, read all candidate-facing specifications in this order:

1. [`ASSIGNMENT.md`](ASSIGNMENT.md) — scope, required CLI outcomes, exact NDJSON envelope, deliverables, and review expectations.
2. [`docs/crm-api.md`](docs/crm-api.md) — authoritative HTTP resources, schemas, relationships, collection query/response rules, tombstones, errors, and retry contract.
3. [`docs/local-mock.md`](docs/local-mock.md) — authoritative executable startup, flags, readiness, process-local scenario lifecycle, reset behavior, and troubleshooting.
4. This `AGENTS.md` — implementation boundaries and a concise cross-document working reference.

Start with the root [`README.md`](README.md) to select the supplied platform binary. If this summary conflicts with the detailed HTTP API, follow `docs/crm-api.md`; for executable behavior and flags, follow `docs/local-mock.md`. If the documents do not settle required behavior, ask rather than guessing or relying on an undocumented endpoint.

## Working boundaries

- Build a Go CLI source connector; do not modify or replace the supplied `mock-crm` executable.
- Implement at least one declared entity through `--entity`. Strongly encourage all six: they share one collection protocol and should be realistic within the timebox. Document implemented and intentionally omitted entities.
- Keep the implementation within the 2–4 hour challenge scope selected by the candidate.
- Prefer the Go standard library. Add only a small number of justified dependencies.
- Do not add a blob store, cloud deployment, mock authentication service, or dependency on evaluator-only behavior.
- Treat IDs as opaque UUIDs. Do not infer fixture ordinals, ordering, or business meaning from UUID text.
- Use only the documented candidate-facing API. There is no debug, reset, or evaluator-oracle endpoint.
- Keep credentials out of source, logs, commits, generated files, and AI-use notes.
- The candidate submits from a new private repository under their own GitHub account, not only from the supplied repository, and adds `paddie` as a collaborator (Read role where available).

## Required CLI contract

The CLI accepts:

```text
--entity company|contact|lead|deal|pipeline|stage
```

A run with no supplied state performs a full pull. The candidate chooses and documents how a prior emitted state value is supplied; a run with that state performs an incremental pull.

Stdout contains NDJSON protocol messages only. Human-readable logs and diagnostics go to stderr.

### Record messages

```json
{"type":"record","record":{"id":"…","updated_at":"…","deleted":false},"timestamp":"…","entity":"company","id":"…"}
```

All five fields are required:

- `type` is exactly `"record"`.
- `record` is the complete API item, including all fields of an updated live object or the complete minimal tombstone.
- `timestamp` equals the API item's `updated_at`.
- `entity` is the selected singular entity name.
- `id` equals the API item's `id`.

Do not emit ID-only records, inferred deltas, or normalized tombstones. Record-message line order is unspecified.

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

Normal mode has no artificial request limit or injected failures. Add `--enable-fault-profile=true` to the matching startup command only for optional resilience testing.

At startup, `mock-crm` exposes a deterministic materialized current state for the fixed 30 UTC days before `--simulation-start`. HTTP requests only observe it. Repeated full pulls and inclusive incremental pulls remain unchanged until the operator presses Enter on an empty line between completed pulls. Enter simulates one UTC day and atomically publishes a complete successor state across operational resources. Press Enter only between pulls; consistent pagination across a publication is intentionally unsupported.

For changing resources, derive the next checkpoint from the greatest `updated_at` safely emitted. After Enter publication, a new inclusive pull at that checkpoint returns current boundary ties and records touched during the simulated day. A refreshed full pull reads the same current state. Pipelines and stages are fixed reference data.

Keep one mock process alive across the full pull, Enter advancement, and incremental client invocation to preserve one scenario. This is not a connector-architecture requirement: the client may restart freely. Restarting `mock-crm` rematerializes the initial state and discards operator advancement and fault state. Discard client checkpoints from the old process after a server restart.

`/healthz` checks readiness without changing simulation state or triggering optional faults. See [`docs/local-mock.md`](docs/local-mock.md) for the complete lifecycle contract.

## CRM API

The sole data endpoint is:

```text
GET /v1/{resource}?page=P&size=S&since=T&order=asc|desc
```

Plural resources are `companies`, `contacts`, `leads`, `deals`, `pipelines`, and `stages`.

- `page` is zero-based and defaults to `0`.
- `size` defaults to `100`; valid values are 1–500.
- `since` is optional, uses an RFC 3339 UTC timestamp with millisecond precision, and is inclusive.
- `order` is `asc` by default; `desc` reverses the complete `(updated_at,id)` tuple.
- A successful response is `{"data":[...]}` with integer `next_page` only when another page exists.
- Continue from page zero until `next_page` is absent. Timestamp ties can cross page boundaries.
- Live records and tombstones occur in the same stream and both participate in filtering, ordering, and pagination.

Pagination itself does not skip or repeat boundary items. Inclusive `since` means exact `(updated_at,id)` records may be replayed between separate runs. A safe checkpoint comes from records already made recoverable and must not pass incomplete tied records.

Malformed or repeated query values and unknown query keys return JSON `400`; unknown paths return `404`; unsupported methods return `405`.

### Retry convention

With the optional fault profile, `/v1` may return `429` or retryable `503`. Both include:

```text
Retry-After-Ms: <positive integer>
```

The JSON body contains the same positive integer as `retry_after_ms`. Treat the header as milliseconds, wait at least that long, and retry the same request. Standard `Retry-After` is not part of this API. The fault cadence is intentionally unspecified.

## CRM vocabulary and relationships

- A **company** is an external account. It has `name`, a synthetic `domain`, and lifecycle `lead`, `opportunity`, or `customer`.
- A **contact** is an external person associated with one company. It has `label`, `company_id`, and the company's lifecycle.
- A **lead** is a potential-customer record with `label` and nullable `company_id`.
- A **deal** belongs to one company. It has `label`, decimal-string `amount`, zero to three same-company `contact_ids`, and canonical `pipeline` and `pipeline_stage` strings.
- A **pipeline** is fixed reference data with a canonical `name`.
- A **stage** is fixed reference data with globally unique `name`, canonical `pipeline`, one-based `position`, and category `open`, `closed_won`, or `closed_lost`.

Company/contact lifecycle is distinct from a deal's pipeline stage and follows aggregate deal state: any historical win makes a live account a customer; otherwise open business makes it an opportunity; otherwise it is a lead. Every live contact matches its company. A company may have multiple deals in different pipelines.

Later simulated days can add companies with related contacts and first-stage deals. Open deals progress forward one ordered stage at a time without changing pipeline, eventually reach a live won/lost terminal stage, and then stop changing. Deal outcomes can emit complete company/contact lifecycle updates. Company deletion cascades minimal tombstones to all of its contacts, associated leads, and deals; cascaded deals are deleted rather than moved to a lost stage.

Each Enter press atomically publishes related changes across companies, contacts, leads, and deals. Each collection exposes only the latest current record or tombstone per ID, so a client that misses several simulated days does not receive superseded intermediate versions. This business simulation does not change independent collection retrieval, inclusive checkpoints, paging, tombstone schemas, or the required NDJSON output.

The fixed pipelines and ordered stages are:

| Pipeline | Ordered stages |
| --- | --- |
| `new-business` | `new-business-qualified`, `new-business-discovery`, `new-business-proposal`, `new-business-negotiation`, `new-business-won`, `new-business-lost` |
| `renewals` | `renewals-upcoming`, `renewals-engaged`, `renewals-review`, `renewals-negotiation`, `renewals-renewed`, `renewals-churned` |
| `partner-sales` | `partner-sales-registered`, `partner-sales-qualified`, `partner-sales-co-selling`, `partner-sales-contracting`, `partner-sales-won`, `partner-sales-lost` |

## Validation and documentation

For every implemented entity, validate one full pull followed by at least 10 incremental simulation ticks while the same mock process remains alive. For each tick, press Enter once between completed pulls, wait for publication confirmation, then invoke the connector with that entity's prior emitted state. Several entities may be retrieved after each Enter press. Fixed `pipeline` and `stage` resources may legitimately return only inclusive boundary replays.

Document:

- exact build and run commands;
- implemented and omitted entity scope;
- the prior-state input mechanism;
- non-obvious dependencies;
- consequential recovery/checkpoint decisions as context, decision, and consequence; and
- concise AI use and verification notes in `AI_USE.md`.

No separate blob inspection, screenshot, video, prescribed test framework, `make validate` target, or prototype-evidence package is required.

For the complete service contracts, return to [`docs/crm-api.md`](docs/crm-api.md) and [`docs/local-mock.md`](docs/local-mock.md); do not treat this summary as a replacement for them.
