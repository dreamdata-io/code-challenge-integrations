# Running the local mock CRM

Use the root [`README.md`](../README.md) to identify your host and select one of the four executables. From the repository root, set `MOCK_CRM` to that path, then start a reproducible simulation:

```sh
# Example only: choose the matching path from README.md.
MOCK_CRM=bin/darwin-arm64/mock-crm
chmod +x "$MOCK_CRM"
"$MOCK_CRM" --listen=:8080 --simulation-start=2031-02-03
```

The service prints the selected date, the exact replay command, and instructions for advancing the simulation. If `--simulation-start` is omitted, it visibly defaults to the current UTC date. In another terminal, check readiness:

```sh
curl --fail http://localhost:8080/healthz
```

The readiness endpoint is outside `/v1`. It observes readiness only and never changes CRM state or triggers optional faults.

## Flags

| Flag | Default | Meaning |
| --- | --- | --- |
| `--listen` | `:8080` | HTTP listen address |
| `--simulation-start` | current UTC date | UTC date (`YYYY-MM-DD`) immediately after the fixed 30-day initial history |
| `--enable-fault-profile` | `false` | install throttling/retryable-failure middleware on `/v1` |

Normal mode has no artificial request limit or injected failures. Enable the fault profile to test the connector's required retry behavior:

```sh
"$MOCK_CRM" --listen=:8080 --simulation-start=2031-02-03 --enable-fault-profile=true
```

## Stable current state and operator advancement

At startup, the service materializes the current CRM state produced by the fixed 30 UTC days before `--simulation-start`. Each collection contains at most one current live record or tombstone per ID. Pipelines and stages are fixed reference data.

HTTP requests only observe this current state. Repeating a full pull or an inclusive incremental pull produces the same records until the operator advances the simulation. A full pull omits `since`; an incremental pull includes a checkpoint and returns current records whose `updated_at` is greater than or equal to it.

Complete a full pull by following `next_page` until it is absent. For example:

```sh
curl --fail 'http://localhost:8080/v1/companies?page=0&size=500&order=asc'
# Continue with page=1, page=2, ... while next_page is present.
```

After the pull finishes, derive a checkpoint from the greatest `updated_at` that the client safely emitted. Before advancement, an inclusive request at that checkpoint returns only current records tied at the boundary:

```sh
CHECKPOINT='<greatest updated_at from the completed pull>'
curl --fail --get 'http://localhost:8080/v1/companies' \
  --data-urlencode page=0 --data-urlencode size=500 \
  --data-urlencode order=asc --data-urlencode "since=$CHECKPOINT"
```

Return to the service terminal and press **Enter on an empty line**. The service deterministically simulates one UTC calendar day, atomically publishes the complete successor state across all operational resources, and confirms the completed simulated date. Now repeat the same inclusive request. It returns the boundary records plus current records touched during the newly completed day. A new full pull returns the refreshed current state.

Advance only between completed pulls. Consistent pagination while Enter is pressed during a multi-page pull is intentionally unsupported; pages requested on opposite sides of publication can observe different current states. HTTP requests, completed pagination, invalid requests, health checks, and optional faults never advance the simulation.

Each additional empty line simulates one more day. Daily changes can add companies with related contacts and deals, move open deals forward by one stage, produce won/lost outcomes and lifecycle updates, and publish company-owned deletion cascades. Related changes are published together across companies, contacts, leads, and deals. A client that misses several days sees each matching ID's latest current version, not intermediate versions that have already been compacted away.

## Process and terminal lifecycle

Keep one mock process alive while testing a full pull, Enter advancement, and a later incremental pull. The connector may exit and restart freely between pulls as long as it receives its prior state through its documented mechanism.

Blank terminal lines are the only advancement input. Non-empty lines are ignored. Redirected stdin follows the same rule. EOF or a stdin read error disables further Enter advancement while HTTP continues serving. Ctrl-C/SIGTERM gracefully shuts down HTTP.

Stopping and restarting the mock is a full reset: the selected/default simulation date is materialized again and all operator advancement is discarded. Discard connector checkpoints from the old process and begin with a full pull. The same explicit simulation start and the same number of Enter presses reproduce the same state.

There is no state file, database, writable data directory, reset endpoint, debug/oracle endpoint, authentication, Docker dependency, or blob-store emulator.

## Troubleshooting

- `400`: inspect query spelling, duplicated values, page/size ranges, order, and timestamp syntax.
- `429`/`503` in optional fault mode: read `Retry-After-Ms` as milliseconds and retry the identical request after waiting at least that long.
- No new incremental records: confirm Enter was sent as an empty line, wait for the completed-day confirmation, and keep the same mock process running.
- Unexpected page results: confirm the operator did not press Enter during the pull, then restart that pull from page zero.
- Unexpected inclusive duplicates: records exactly tied at `since` are intentionally returned again; consumers may de-duplicate exact `(updated_at,id)` replays.
- Startup, terminal activity, and graceful shutdown are logged to stderr. API data is returned only over HTTP.
