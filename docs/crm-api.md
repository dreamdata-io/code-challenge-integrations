# Mock CRM API

The mock serves unauthenticated JSON over HTTP. All IDs are opaque UUIDs and all timestamps are UTC RFC 3339 values with exactly millisecond precision. Do not infer ordering or meaning from UUID text.

## First request

With the mock running in normal mode, fetch the first two companies:

```sh
curl --fail 'http://localhost:8080/v1/companies?page=0&size=2'
```

A response has this shape (the values below are invented illustrations, not fixture values):

```json
{"data":[{"id":"<opaque-company-id-1>","created_at":"2025-01-01T00:00:00.000Z","updated_at":"2025-01-01T00:00:00.000Z","deleted":false,"name":"Example Company One","domain":"example-one.invalid","lifecycle":"lead"},{"id":"<opaque-company-id-2>","created_at":"2025-01-01T00:00:00.000Z","updated_at":"2025-01-01T00:00:00.000Z","deleted":false,"name":"Example Company Two","domain":"example-two.invalid","lifecycle":"opportunity"}],"next_page":1}
```

`next_page` means more pages exist. Continue with its integer value, preserving the same resource, `size`, `since`, and `order`, until a response omits `next_page`. Requests only observe the current materialized state; they never advance the simulation. The operator must not press Enter during a multi-page pull because cross-publication pagination consistency is intentionally unsupported. See [Running the local mock CRM](local-mock.md) for the operator journey.

## Collection protocol

The sole data endpoint is:

```text
GET /v1/{resource}?page=P&size=S&since=T&order=asc|desc
```

Resources are `companies`, `contacts`, `leads`, `deals`, `pipelines`, and `stages`. Every resource uses the same protocol:

- `page` is zero-based and defaults to `0`.
- `size` defaults to `100`; valid values are 1–500.
- `since` is optional and **inclusive**. It is a UTC RFC 3339 timestamp with exactly millisecond precision. Use an emitted record's `updated_at`, not client wall-clock time, as a checkpoint.
- `order` defaults to `asc`; `desc` reverses the complete `(updated_at, id)` tuple.
- A successful response is `{"data":[...]}` and includes integer `next_page` only when another page exists.

Live records and tombstones share the collection and participate in the same filtering, tuple ordering, and pagination. Timestamp ties are intentional and can cross page boundaries. Page until `next_page` is absent before advancing a checkpoint. While the current state is unchanged, pagination does not skip or repeat boundary items. Inclusive checkpoint replay between separate pulls is normal, so consumers may need to de-duplicate exact `(updated_at,id)` replays.

Unknown query keys, repeated scalar values, malformed values, negative pages, and invalid sizes produce JSON `400` responses. Unknown paths are `404`; unsupported methods are `405`.

## Exact item schemas

Every live item has these common fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | Opaque UUID. |
| `created_at` | string | UTC RFC 3339 timestamp with millisecond precision. |
| `updated_at` | string | UTC RFC 3339 timestamp with millisecond precision; the incremental field. |
| `deleted` | boolean | Exactly `false` for a live item. |

A live item also has all fields for its resource:

| Resource | Additional fields |
| --- | --- |
| `companies` | `name` string, `domain` string, `lifecycle` string (`lead`, `opportunity`, or `customer`) |
| `contacts` | `label` string, `company_id` string, `lifecycle` string (`lead`, `opportunity`, or `customer`) |
| `leads` | `label` string, `company_id` string or `null` |
| `deals` | `label` string, `amount` decimal string, `company_id` string, `contact_ids` array of zero to three strings, `pipeline` string, `pipeline_stage` string |
| `pipelines` | `name` string |
| `stages` | `name` string, `pipeline` string, `position` one-based integer, `category` string (`open`, `closed_won`, or `closed_lost`) |

For example, a complete illustrative live lead is:

```json
{"id":"<opaque-lead-id>","created_at":"2025-01-01T00:00:00.000Z","updated_at":"2025-01-02T00:00:00.000Z","deleted":false,"label":"Example Lead","company_id":null}
```

A tombstone for every resource is exactly this minimal object—there is no `created_at` or resource-specific field:

```json
{"id":"<opaque-id>","updated_at":"2025-01-03T00:00:00.000Z","deleted":true}
```

Preserve and emit the complete object returned by the API, whether it is live or deleted.

## CRM vocabulary and relationships

- A **company** is an external account. Its lifecycle is `lead`, `opportunity`, or `customer`.
- A **contact** is an external person associated with exactly one company. Its lifecycle matches that company.
- A **lead** is a potential-customer record and may be associated with one company or no company (`company_id: null`).
- A **deal** belongs to exactly one company. It names zero to three contacts, and every named contact belongs to that same company. A company may have multiple deals in different pipelines.
- A **pipeline** is fixed reference data identified in relationships by its canonical `name`.
- A **stage** is fixed reference data with a globally unique canonical `name`, its canonical pipeline name, one-based position, and category.

Company/contact lifecycle is distinct from a deal's pipeline stage and is derived from the company's aggregate deal state: any historical closed-won deal makes a live company a `customer`; otherwise active open business makes it an `opportunity`; otherwise it is a `lead`. Every live contact has the same lifecycle as its company.

### Operator-controlled changes across simulated days

At startup, the API exposes a deterministic materialized current state for the fixed 30 days before `--simulation-start`. Repeated HTTP pulls do not change it. When the operator presses Enter on an empty line between completed pulls, the service simulates one UTC day and atomically publishes a complete successor state across all operational resources:

- New companies can appear with contacts and deals. New deals begin in the first open stage of one pipeline.
- A live open deal moves forward through that pipeline's ordered stages without skipping, moving backward, or changing pipeline. It eventually receives a live `closed_won` or `closed_lost` stage and has no later progression updates.
- Deal outcomes can produce complete company and contact updates when their derived lifecycle changes.
- When a company is deleted, the company and all of its contacts, associated leads, and deals receive minimal tombstones. A cascaded deal is deleted directly rather than moved to a lost stage.

All related resource changes become visible together after publication. Each collection contains only the latest current record or tombstone per ID, so clients that miss multiple days do not receive superseded intermediate versions. A refreshed full pull reads the new current state; an inclusive incremental pull filters that same state. Pipelines and stages remain fixed reference data. The fixed pipelines and ordered stages are:

| Pipeline | Ordered stages |
| --- | --- |
| `new-business` | `new-business-qualified`, `new-business-discovery`, `new-business-proposal`, `new-business-negotiation`, `new-business-won`, `new-business-lost` |
| `renewals` | `renewals-upcoming`, `renewals-engaged`, `renewals-review`, `renewals-negotiation`, `renewals-renewed`, `renewals-churned` |
| `partner-sales` | `partner-sales-registered`, `partner-sales-qualified`, `partner-sales-co-selling`, `partner-sales-contracting`, `partner-sales-won`, `partner-sales-lost` |

## Incremental request example

A real `since` value must come from the greatest `updated_at` among API records your connector has safely emitted; do not copy the illustrative timestamp below and do not use the client clock. Percent-encode it through your HTTP library:

```text
GET /v1/companies?page=0&size=100&since=2025-01-02T00%3A00%3A00.000Z&order=asc
```

Because `since` is inclusive, all records tied at that timestamp are eligible to appear again. A safe checkpoint must not pass tied records that have not been made recoverable.

## Retry responses

Normal mode has no artificial throttling or injected failures. With the optional fault profile, `/v1` can return retryable `429` or `503` responses. Both carry the custom positive-integer header `Retry-After-Ms` and an identical JSON `retry_after_ms` field. Wait at least that many **milliseconds** and retry the identical request. Standard `Retry-After` is never used.
