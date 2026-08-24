# CRM source connector challenge

Start here. Clone this challenge repository locally, then create a **new private repository under your own GitHub account** and push your completed work there. Add GitHub user **`paddie`** as a collaborator (select the **Read** role where available), and confirm the repository shows `paddie` as active or invited. Then email **prm@dreamdata.io** with the repository URL and submitted commit hash. The invitation may still be pending when you submit. Do not submit only in the supplied challenge repository.

Add your Go connector, tests, documentation, decision notes, and `AI_USE.md`, then complete the checklist in [`ASSIGNMENT.md`](ASSIGNMENT.md). Extend this README with your connector's exact build, full-run, and incremental-run commands so it remains the repository landing page for review.

The supplied `mock-crm` executables are challenge infrastructure. Their source is intentionally absent; do not modify or replace them.

## Read before coding

1. [`ASSIGNMENT.md`](ASSIGNMENT.md) — task, baseline success criteria, deliverables, checklist, and technical review.
2. [`docs/crm-api.md`](docs/crm-api.md) — authoritative HTTP resources, schemas, relationships, pagination, tombstones, and retry contract.
3. [`docs/local-mock.md`](docs/local-mock.md) — authoritative startup and process-local scenario behavior.
4. [`AGENTS.md`](AGENTS.md) — concise working agreement for you and coding agents.

If a summary conflicts with the detailed API, follow `docs/crm-api.md`; for executable behavior and flags, follow `docs/local-mock.md`. Ask rather than inventing unspecified behavior.

## Select and start the mock CRM

Identify your host:

```sh
uname -s
uname -m
```

Select the matching executable. `x86_64` means `amd64`; `arm64` and `aarch64` mean `arm64`.

| `uname -s` | `uname -m` | Executable |
| --- | --- | --- |
| `Darwin` | `arm64` | `bin/darwin-arm64/mock-crm` |
| `Darwin` | `x86_64` | `bin/darwin-amd64/mock-crm` |
| `Linux` | `aarch64` or `arm64` | `bin/linux-arm64/mock-crm` |
| `Linux` | `x86_64` | `bin/linux-amd64/mock-crm` |

Windows executables are not supplied.

From the repository root, copy/paste the pair for your host:

```sh
# macOS on Apple silicon
chmod +x bin/darwin-arm64/mock-crm
bin/darwin-arm64/mock-crm --listen=:8080

# macOS on Intel
chmod +x bin/darwin-amd64/mock-crm
bin/darwin-amd64/mock-crm --listen=:8080

# Linux on arm64
chmod +x bin/linux-arm64/mock-crm
bin/linux-arm64/mock-crm --listen=:8080

# Linux on amd64
chmod +x bin/linux-amd64/mock-crm
bin/linux-amd64/mock-crm --listen=:8080
```

Run only the matching command and leave that terminal running. For an exactly reproducible scenario, append `--simulation-start=2031-02-03`; otherwise startup visibly selects the current UTC date. In another terminal, check readiness and make the first page of a full pull:

```sh
curl --fail http://localhost:8080/healthz
curl --fail 'http://localhost:8080/v1/companies?page=0&size=2'
```

Follow each `next_page` until it is absent. HTTP requests never advance the simulation, so repeated full and inclusive incremental pulls remain unchanged. After the full pull completes, save its greatest `updated_at` and make an inclusive boundary pull:

```sh
CHECKPOINT='<greatest updated_at from the completed full pull>'
curl --fail --get 'http://localhost:8080/v1/companies' \
  --data-urlencode page=0 --data-urlencode size=500 \
  --data-urlencode order=asc --data-urlencode "since=$CHECKPOINT"
```

Follow `next_page` until it is absent. Then return to the mock terminal, press **Enter on an empty line**, and wait for the completed-day confirmation. Repeat the same inclusive command and follow its pages again; it now returns boundary ties plus records touched during the simulated day. Finally, rerun the full-pull command to observe the refreshed current state.

Press Enter only between completed pulls; pagination across a publication is intentionally unsupported. In connector code, derive the checkpoint from the greatest `updated_at` safely emitted, not merely observed. See [`docs/local-mock.md`](docs/local-mock.md) for the complete terminal and process lifecycle.

Keep this mock process alive for the required validation: for every implemented entity, perform an initial full pull and retrieve at least 10 Enter-driven incremental ticks using the prior emitted state each time. You can retrieve several entities after each Enter press. Your connector process may restart freely; restarting the mock starts a fresh scenario and invalidates old connector checkpoints.

## Maintainer build

With [mise](https://mise.jdx.dev/) and Go available, build all supplied mock CRM targets from source into this repository's generated `bin/` directory:

```sh
# Required once when mise first sees this repository's configuration.
mise trust
mise run build
```

This produces `bin/{darwin,linux}-{arm64,amd64}/mock-crm`. The generated binaries are ignored; use `mock-crm/scripts/build-release.sh` when creating the sanitized candidate package under `mock-crm/dist/`.
