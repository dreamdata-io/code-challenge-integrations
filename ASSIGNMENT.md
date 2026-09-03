# CRM source connector challenge

## Goal

Build a Go CLI that reads CRM records from the supplied local `mock-crm` service and writes source-protocol NDJSON to stdout.

Implement all six entities: `company`, `contact`, `lead`, `deal`, `pipeline`, and `stage`. They share one collection protocol and are designed to be completed within the expected effort of roughly **3–5 hours**. This timebox is guidance rather than a cutoff; focus on correctness, safety, and reasoning rather than unrelated polish or unrequested features.

Use the candidate documents in this order:

1. [`README.md`](README.md) — repository landing page and mock startup.
2. This assignment — deliverables, OpenAPI validation, and acceptance criteria.
3. [`openapi.yaml`](openapi.yaml) — baseline OpenAPI 3.0 specification for runtime validation.
4. [`docs/crm-api.md`](docs/crm-api.md) — authoritative HTTP protocol and schemas.
5. [`docs/local-mock.md`](docs/local-mock.md) — authoritative operator-controlled simulation lifecycle.
6. [`AGENTS.md`](AGENTS.md) — concise agent working agreement.

If these documents conflict, `crm-api.md` wins for HTTP behavior and `local-mock.md` wins for executable behavior. Ask rather than inventing unspecified behavior.

## Required CLI behavior

The CLI must accept `--entity` with the singular name of every entity:

```text
company|contact|lead|deal|pipeline|stage
```

The singular names map to `/v1/companies`, `/v1/contacts`, `/v1/leads`, `/v1/deals`, `/v1/pipelines`, and `/v1/stages`.

The CLI must also support runtime OpenAPI validation via:

```text
--validate
--spec <path>
```

- `--validate` is an optional boolean flag (default `false`). When omitted, the CLI performs normal sync without specification loading or validation overhead. When provided, the CLI performs runtime OpenAPI validation of every response and record.
- `--spec` is an optional string flag specifying the path to an OpenAPI 3.0 specification file (defaults to `openapi.yaml`). It is used when `--validate` is active.
- With no prior state, perform a full pull from page zero through the response without `next_page`.
- Provide and document a flag or input mechanism for a prior emitted state value.
- With prior state, perform an inclusive incremental pull using the corresponding source cursor: `changed_at` as epoch milliseconds for companies, or `updated_at` as UTC RFC 3339 for other entities.
- Make the mock address configurable or clearly document the expected address.
- Exit zero only after a complete successful sync. A failed run exits non-zero; its stdout is an incomplete stream.
- Write human-readable logs and validation diagnostics only to stderr.

Use the Go standard library where practical. Adding a small number of justified dependencies—such as a standard OpenAPI 3.0 validation library (e.g. `github.com/getkin/kin-openapi`)—is expected and acceptable. Document any non-obvious dependency.

## OpenAPI contract validation and remediation

In real production integrations, APIs and published documentation often diverge. The repository supplies `openapi.yaml` defining the CRM collection API and entity schemas with strict validation (`additionalProperties: false`).

When `--validate` is enabled:

1. **Pre-flight spec validation**: Load and structurally check the selected `--spec` file before initiating network activity. If the specification is missing, unreadable, or syntactically invalid, print an actionable diagnostic to stderr and exit non-zero immediately without making HTTP requests.
2. **Operation response validation**: For every successful HTTP response, match the request to its OpenAPI operation and validate the HTTP status, headers (where defined), and full JSON response page envelope.
3. **Component record validation**: Validate each item in the page directly against its entity component schema (`#/components/schemas/{Entity}`) before emitting any records from that page.
4. **Safe failure semantics**: If any validation check fails:
   - Print an actionable diagnostic to stderr describing the schema violation.
   - Halt execution immediately with a non-zero exit code.
   - Do not emit a checkpoint that advances beyond the input state, ensuring no data progress is recorded past the failure point.

Validate all six entity endpoints against the supplied `openapi.yaml`. If you observe any discrepancy between live mock endpoint payloads and `openapi.yaml`, update the specification to accurately reflect the live mock behavior and briefly explain your findings in your submission documentation. You may either:
- update `openapi.yaml` in place, or
- provide an alternate corrected specification file (such as `openapi.proposed.yaml`) selected via `--spec`.

## Stdout protocol

Stdout contains NDJSON only: one complete JSON object per line, with no banners, blank lines, or diagnostics.

Every API item, including a deleted record, is emitted unchanged in this exact envelope:

```json
{"type":"record","record":{"id":"…","changed_at":1735689600000,"deleted":false,"name":"…","domain":"…","lifecycle":"lead"},"timestamp":"2025-01-01T00:00:00.000Z","entity":"company","id":"…"}
```

All five fields are required:

- `type` is `"record"`;
- `record` is the complete API object without normalization or omitted fields (for companies, this contains integer `changed_at`, and neither `created_at` nor `updated_at`; for other entities, it contains `created_at` and `updated_at`);
- `timestamp` is the object's update instant converted to UTC RFC 3339 with millisecond precision (for companies, converted from integer `changed_at`; for all other entities, matching `updated_at`);
- `entity` is the selected singular entity; and
- `id` equals that object's `id`.

Emit at least one state message on every successful run:

```json
{"type":"state","value":{"your":"state representation"}}
```

The state value and checkpoint cadence are your design. State must be sufficient for the documented next invocation to perform an incremental pull.

## Pagination and checkpoint safety

Follow every `next_page` until it is absent. Timestamp ties can cross page boundaries. Derive state from records made recoverable by your CLI, never from wall-clock or response time, and never advance a published checkpoint past unfinished output.

Old mock checkpoints or checkpoints from before this clean-break schema change are explicitly invalid across process restarts or schema upgrades; no compatibility alias or legacy mode is supported.

`since` is inclusive (epoch milliseconds for companies, RFC 3339 for other resources), so exact boundary records may appear again between runs. Prefer safe replay over skipped records and explain your recovery/duplication trade-off in a short decision note.

## Required resilience

Retry `429` and all `5xx` responses. A retry must repeat the identical page request and must not advance state past unprocessed records.

When a response includes a positive `Retry-After-Ms` header, wait at least that many **milliseconds** before retrying. The mock's injected `429` and `503` responses include this header and the same `retry_after_ms` value in their JSON body. For a `5xx` without that header, use a documented bounded backoff policy. If retries are exhausted, fail the sync, exit non-zero, and do not emit state that claims the incomplete pull succeeded.

The supplied binary still supports the fault profile, disabled by default. Enable it when testing resilience:

```sh
"$MOCK_CRM" --listen=:8080 --simulation-start=2031-02-03 --enable-fault-profile=true
```

The profile affects `/v1` only; it does not advance simulation state. Its fault cadence is intentionally unspecified. See [`docs/crm-api.md`](docs/crm-api.md) for the exact retry response format and [`docs/local-mock.md`](docs/local-mock.md) for flag details.

## Operator-controlled simulation and required validation

The mock starts with a stable materialized state for the fixed 30 UTC days before `--simulation-start`. HTTP requests never advance it. Pressing Enter on an empty line in the mock terminal simulates one UTC day and atomically publishes the next current state. Advance only between completed pulls; pagination across an Enter publication is unsupported.

Validate at least **10 simulation ticks for all six entities**:

1. Keep one mock process running and perform the entity's initial full pull.
2. Save the emitted state.
3. Press Enter once and wait for the completed-day confirmation.
4. Run that entity incrementally using its previous state and save the new state.
5. Repeat steps 3–4 until you have retrieved 10 ticks for that entity.

You may validate several entities after each Enter press before advancing again. `pipeline` and `stage` are fixed reference data, so later incremental pulls may contain only inclusive boundary replays; successful retrieval and state emission still count.

After addressing any specification discrepancies discovered during validation, verify that `--validate` succeeds cleanly across both the initial full pulls and the 10 incremental simulation ticks for every entity.

Restarting the connector is expected. Restarting the mock resets the simulation and invalidates old connector state.

## Submission

Submit from a **private repository owned by your own GitHub account**, not only from the supplied challenge repository:

1. Clone the supplied repository locally.
2. Create a new private repository under your GitHub account and push your completed challenge to it. A private repository import or duplicate is also fine.
3. Add GitHub user **`paddie`** as a collaborator on that repository. Select the **Read** role where GitHub offers repository roles.
4. On the repository's collaborator page, confirm `paddie` is listed as active or invited, and confirm the exact commit you are submitting has been pushed. The invitation may still be awaiting acceptance.
5. After inviting `paddie`, email **prm@dreamdata.io** with the repository URL and submitted commit hash so we know the challenge is ready for review.

If you use organization-funded AI access, send it only the local synthetic challenge materials. Do not commit credentials or the provided API key; a private transcript is not required.

Your repository must contain:

- runnable Go source;
- a root README with exact build, mock startup, full-run, incremental-run, and validated-run commands, as well as a concise schema drift-remediation note;
- the corrected OpenAPI specification (`openapi.yaml` edited in place or an alternate spec file selected via `--spec`);
- documented implemented entities;
- tests you consider appropriate;
- a lightweight context/decision/consequence note for consequential checkpoint or recovery choices;
- `AI_USE.md` with tools/models used, important assistance, and how you verified it; and
- this completed checklist.

Bulk NDJSON output dumps are not required. Reproducible commands and tests are sufficient evidence.

## Completion checklist

- [ ] I created a private submission repository under my own GitHub account and pushed the submitted commit.
- [ ] I added `paddie` as a collaborator (Read role where available), confirmed the active or pending invitation, and pushed the submitted commit.
- [ ] I emailed `prm@dreamdata.io` after inviting `paddie`, including the repository URL and submitted commit hash.
- [ ] I implemented all six CRM entities (`company`, `contact`, `lead`, `deal`, `pipeline`, `stage`).
- [ ] I documented exact build, mock startup, full-run, incremental-run, and validated-run commands.
- [ ] My CLI supports `--validate` and `--spec <path>`, defaulting to `openapi.yaml`.
- [ ] Validation loads and structurally checks the specification before network requests, failing fast with non-zero exit on invalid or missing specifications.
- [ ] Validation validates both the operation page response and each component entity record before emitting records from that page.
- [ ] Validation errors output actionable context to stderr, exit non-zero, and emit no checkpoint that advances beyond input state.
- [ ] I validated all six entities against `openapi.yaml`, remediated any discovered schema drift, and documented the remediation alongside my corrected OpenAPI specification.
- [ ] A run without prior state completes a full pull through the final page.
- [ ] I documented how emitted state is supplied to the next invocation.
- [ ] Stdout contains only valid record/state NDJSON; diagnostics go to stderr.
- [ ] Record messages preserve complete live and deleted objects.
- [ ] My checkpoint handles inclusive timestamp ties without skipping records.
- [ ] My connector retries `429` and `5xx` responses, honors `Retry-After-Ms`, uses bounded fallback backoff, and fails safely when retries are exhausted.
- [ ] I tested the connector with `mock-crm --enable-fault-profile=true`.
- [ ] For all six entities, I retrieved an initial full pull and at least 10 Enter-driven incremental ticks from one running mock process, with `--validate` passing cleanly across all runs after remediation.
- [ ] I recorded important design/recovery decisions.
- [ ] I included tests and a concise credential-free `AI_USE.md`.

## Technical review

The submission is followed by a **30–45 minute** code review. Be ready to discuss scope, structure, dependencies, OpenAPI loading and routing, operation-response versus component-record validation, schema drift diagnosis and remediation, pagination, complete-record handling, checkpoint safety, retry/backoff behavior, recovery trade-offs, ten-tick validation, and how you checked AI-assisted work. Slides, video, screenshots, and a prescribed test framework are not required.
