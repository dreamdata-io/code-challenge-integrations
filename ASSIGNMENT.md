# CRM source connector challenge

## Goal

Build a Go command-line source connector that reads CRM records from the supplied local mock service and writes a machine-readable NDJSON stream to stdout.

Your connector must support:

- an initial full pull for a selected CRM entity;
- a later incremental pull when given state from a previous successful run;
- safe pagination and checkpointing; and
- complete live records and deletion tombstones.

The challenge is deliberately open-ended around internal design. We care more about a small, coherent implementation and clear source-protocol reasoning than broad but shallow coverage.

## Baseline success criteria

A baseline submission must:

- build and run from the exact commands in your README;
- implement at least one declared entity and accept its singular name through `--entity` (all six are welcome, but not required);
- document implemented and intentionally omitted entities;
- perform a full pull when no prior state is supplied, following `next_page` until it is absent;
- emit complete live records and complete tombstones in the exact record envelope under [Stdout protocol](#stdout-protocol);
- write only valid record/state NDJSON to stdout and human-readable diagnostics to stderr;
- emit recoverable state on every successful run, document how its `value` is supplied to the next invocation, and use supplied state for the corresponding inclusive incremental pull without skipping timestamp ties;
- exit zero only after a successful complete sync and non-zero on failure; and
- include runnable Go source, your README, tests you consider appropriate, decision note(s), `AI_USE.md`, this updated checklist, and read-only collaborator access for `paddie`.

Optional fault-profile resilience is not a baseline requirement. Implementing additional entities does not compensate for unsafe pagination, output, or checkpoint behavior.

## Timebox and scope

Spend **2–4 hours** on the challenge.

Choose which of these entities to implement:

- `company`
- `contact`
- `lead`
- `deal`
- `pipeline`
- `stage`

Your CLI must accept the selected singular name through `--entity`. Implement at least one entity you can explain well. Completing all six is welcome, but not required.

Use the Go standard library where practical. A small number of well-justified external libraries is allowed; document non-obvious dependency choices.

## Documentation map and supplied materials

Use these documents together:

1. **This [`ASSIGNMENT.md`](ASSIGNMENT.md)** defines the goal, timebox, required behavior, deliverables, checklist, and review.
2. **[`docs/crm-api.md`](docs/crm-api.md)** is the detailed client-facing HTTP API specification: resources, object fields, relationships, pagination, inclusive incremental retrieval, tombstones, errors, and retry responses.
3. **[`docs/local-mock.md`](docs/local-mock.md)** is the executable operations specification: startup, flags, readiness, Enter-driven simulation behavior, restart/reset semantics, and troubleshooting.
4. **[`AGENTS.md`](AGENTS.md)** is the canonical agent-readable working agreement and concise service reference. Coding agents must read it and this assignment before changing code.

The challenge package includes four prebuilt platform-specific `mock-crm` executables under `bin/`. Their source is intentionally absent. Use the selection and startup commands in the root [`README.md`](README.md).

If a summary here or in `AGENTS.md` appears to conflict with the detailed HTTP API, follow `crm-api.md`. For mock process lifecycle and flags, follow `local-mock.md`. Ask for clarification rather than inventing behavior that none of the supplied documents defines.

Start the matching executable and keep it running while testing a full client run, one Enter advancement, and a later incremental run; see the root [`README.md`](README.md) for exact platform commands.

At startup, the mock exposes a deterministic materialized current state for the fixed 30 UTC days before `--simulation-start`. HTTP requests only observe it: repeated full and inclusive incremental pulls remain unchanged until the operator presses Enter on an empty line between completed pulls. Enter simulates one UTC day and atomically publishes a complete successor state across companies, contacts, leads, and deals. Advance only between pulls; consistent pagination across an Enter publication is intentionally unsupported.

Simulated days can add companies with related contacts and deals, move open deals forward without changing pipelines to live won/lost terminal stages, update company/contact lifecycle from aggregate deal outcomes, and cascade company deletion to its contacts, associated leads, and deals. Each collection contains only the latest current record or tombstone per ID. A refreshed full pull reads that new current state, and an inclusive incremental pull filters the same state. Pipelines and stages remain fixed reference data. See [`docs/crm-api.md`](docs/crm-api.md) and [`docs/local-mock.md`](docs/local-mock.md) for the complete model.

Keeping the mock process alive provides scenario continuity; it is not a constraint on connector architecture. Restarting the client is expected. Restarting the mock CRM rematerializes its initial state and invalidates checkpoints from the old process.

## Required CLI behavior

### Inputs

- Accept `--entity` with one of the singular entity names above.
- With no prior state, perform a full pull.
- Provide and document a flag or other input mechanism through which a state value emitted by an earlier successful run can be supplied. When prior state is supplied, use it to perform an incremental pull.
- Make the mock CRM address configurable or clearly document the address your CLI expects.

The state representation and checkpoint cadence are yours to design.

### Stdout protocol

Stdout is exclusively newline-delimited JSON: one complete JSON object per line, with no banners, blank lines, progress text, or other output. Write diagnostics and human-readable progress to stderr.

A record line has this exact envelope:

```json
{"type":"record","record":{"id":"…","updated_at":"…","deleted":false},"timestamp":"…","entity":"company","id":"…"}
```

Every record message contains:

- `type`: exactly `"record"`;
- `record`: the complete object returned in the CRM response, without normalization or omitted fields;
- `timestamp`: that object's `updated_at` value;
- `entity`: the selected singular entity name; and
- `id`: that object's source `id`.

Emit tombstones in the same envelope and preserve the complete tombstone object in `record`.

A state line has this envelope:

```json
{"type":"state","value":{"your":"state representation"}}
```

Emit at least one state line on every successful run. `value` may be any valid JSON value and is evaluator-opaque. Document how a reviewer should take an emitted state value and supply it to the next invocation.

Record order on stdout is not part of the contract. A successful completed sync exits with status 0. On failure, write diagnostics to stderr and exit non-zero; consumers will treat any stdout produced by a failed process as a partial, unsuccessful run.

### Checkpointing outcome

Derive checkpoints from CRM records that your CLI has made recoverable, not from client wall-clock time or response arrival time. The API's `since` filter is inclusive, timestamp ties are intentional, and exact boundary records can therefore reappear between runs. Do not advance a published checkpoint past unfinished work.

Your design should avoid unnecessary duplicate delivery without sacrificing safe recovery. Record consequential recovery and duplication choices in a lightweight decision note.

## Deliverables

Submit a private GitHub repository containing:

1. runnable Go source for the CLI;
2. a README with build and exact full/incremental run commands;
3. tests you consider appropriate for your chosen scope;
4. one or more lightweight decision notes using context, decision, and consequence;
5. `AI_USE.md`, a concise journal of the AI tools/models used, the important tasks or decisions they assisted with, and how you checked their output; and
6. this checklist updated to show what you completed.

Do not include personal credentials or the provided Anthropic API key in the repository. A private transcript is not required.

Add GitHub user **`paddie`** as a read-only collaborator when you are ready to submit.

## Completion checklist

Update this section in your submitted repository.

- [ ] I documented how to build and run the CLI.
- [ ] `--entity` works for at least one declared entity; I documented implemented and intentionally omitted entities.
- [ ] A run without prior state performs a full pull.
- [ ] I documented and demonstrated how emitted state is supplied to an incremental run.
- [ ] Stdout contains only valid record/state NDJSON messages; logs go to stderr.
- [ ] Record messages preserve complete CRM live objects and tombstones.
- [ ] My implementation follows pagination through the final page.
- [ ] I tested a full run, pressed Enter between pulls, and tested an incremental run while keeping one mock CRM process alive.
- [ ] I recorded important design/recovery decisions.
- [ ] I included a concise AI-use journal without credentials or required private transcripts.
- [ ] I added `paddie` to the private repository with read access.

## Optional stretch work

You may test resilience by restarting your matching mock executable with the optional flag (set `MOCK_CRM` to the matching path from the root README):

```sh
"$MOCK_CRM" --listen=:8080 --enable-fault-profile=true
```

In this mode, `/v1` may return retryable `429` and `503` responses. Optional fault handling can demonstrate additional depth, but it is not a baseline requirement and should not displace a sound full/incremental implementation.

## Technical review

The submission is followed by a **30–45 minute technical review**. Using your code as the primary artifact, be prepared to discuss:

- what you completed and deliberately left out;
- your structure and dependency choices;
- pagination and complete-record handling;
- checkpoint safety, timestamp ties, and incremental behavior;
- recovery and duplication trade-offs;
- how you validated the implementation; and
- how AI contributed to the work and how you verified its output.

Slides are optional. A separate video, screenshot package, prescribed test framework, or live-demo presentation is not required.

## AI access and safety

You will receive a distinct organization-controlled Anthropic API key that expires seven calendar days after issuance. You may use any compatible client or agent and any Claude model enabled for the challenge; Claude Code is not required. You may instead use your own tools.

Send only the local synthetic challenge materials to organization-funded AI access. Never commit the supplied key or other personal credentials. The concise `AI_USE.md` journal is required, but private chat transcripts are not.
