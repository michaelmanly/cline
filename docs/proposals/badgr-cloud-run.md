# Proposal: `cline cloud run` — run Cline tasks on Badgr's cloud compute

Status: **Draft / RFC**

## Summary

Add a new CLI subcommand, `cline cloud run "<task>"`, that offloads a Cline
task to [Badgr](https://aibadgr.com)'s cloud compute instead of running it on
the local machine. This lets users kick off Cline agent runs (including
long-running or resource-heavy ones) from a laptop or CI job without keeping
a local process alive, and pull the results back once the run finishes.

## Motivation

Today every `cline` invocation (`bun run cli`, the `cline` binary, the TUI,
ACP mode, connectors) runs the agent loop locally against `@cline/core` /
`@cline/agents`. There's no built-in way to:

- Run a task on a disposable cloud VM and disconnect (close the laptop, let
  it finish).
- Fan out several tasks in parallel without spinning up local resources.
- Run Cline from environments where installing/building the full toolchain
  locally is undesirable (thin CI runners, low-power machines).

Badgr already ships a first-class primitive for exactly this: `badgr launch
cline "<task>"` and the tracked `badgr job cline "<instruction>" --check
"<command>"` job type, both of which run the **Cline agent CLI itself** on a
Badgr-provisioned VM. Notably `cline` is one of Badgr's four zero-config
agents, and it's the one case where **Badgr provides and pays for model
access**, so no BYOK/API-key setup is required for a quick cloud run.

## Proposed design

### New command: `apps/cli/src/commands/cloud.ts`

```
cline cloud run "<task description>" [options]
```

Registered alongside the existing subcommands in
`apps/cli/src/commands/program.ts` (same pattern as `schedule`, `connect`,
`dashboard`, `kanban`).

Options (mirroring Badgr's `badgr job` flags where they map cleanly):

| Flag | Description |
|---|---|
| `--repo <url>` | Repo to run against (default: current git remote, matching `badgr launch`'s bare-task repo resolution) |
| `--ref <ref>` | Branch/tag/commit to check out |
| `--check <command>` | Optional pass/fail acceptance command. Routes to `badgr job cline ... --check` instead of `badgr launch cline`, so the run is a trackable job with a polled result |
| `--max-cost <usd>` | Hard spend cap (default $2.00, same default as `badgr job`) |
| `--max-runtime <sec>` | Hard runtime cap (default 1800s) |
| `--provider <badgr\|openai\|anthropic>` | Who pays for model usage; defaults to `badgr` (no account needed) so the zero-config path stays zero-config |
| `--detach` | Submit and return immediately instead of polling/streaming logs |
| `--json` | Machine-readable status output, consistent with the existing root `--json` option |

### Execution flow

1. Validate a Badgr API key is available (`BADGR_API_KEY` env var, or a
   config entry — see Open Questions).
2. Resolve the repo the same way `badgr launch`'s bare-task form does:
   explicit `--repo` → current local git remote → error if neither.
3. Submit the job. Recommend calling `POST /v1/jobs`
   (`type: "agent"`, `input.agent: "cline"`) directly over HTTP rather than
   shelling out to a separately-installed `badgr-cli`, to avoid a second CLI
   dependency and keep this consistent with how Cline's own provider clients
   talk to remote APIs.
4. Poll `GET /v1/jobs/{id}` (unless `--detach`) and stream `status`/`logs` to
   stdout the same way the local agent loop streams tool output today.
5. On completion, fetch the resulting git diff/branch (equivalent of `badgr
   pull <job-id>`) and print or apply it locally, gated by an `--apply` flag.

### Out of scope for v1

- `claude` / `codex` / `playwright` agents via Badgr — only `cline` (the
  zero-config, Badgr-pays path) for the first version.
- `badgr serve` / `badgr run` (model-serving, generic GPU jobs) — unrelated
  to running the Cline agent itself.
- Any change to the local agent runtime (`@cline/core`, `@cline/agents`) —
  this is purely a new CLI entry point that delegates execution elsewhere.

## Open questions

1. **Credential storage** — should the Badgr API key live alongside existing
   provider credentials (`cline auth`), or as a separate `cline cloud auth`
   command / `BADGR_API_KEY` env var only?
2. **Result retrieval** — should `cline cloud run` block and apply the
   resulting patch directly to the local working tree (`git apply` on the
   pulled diff), or always leave that as an explicit `cline cloud pull
   <job-id>` step?
3. **Cost guardrails** — bake in CLI-level defaults matching Badgr's $2.00 /
   1800s, or require the user to pass `--max-cost`/`--max-runtime` explicitly
   the first time, given the CLI doesn't currently track spend anywhere?
4. Should this ship behind a feature flag / opt-in during initial rollout,
   given it's a third-party paid dependency?

## References

- Badgr Compute API docs: https://aibadgr.com/docs/compute-api
- Relevant primitives: `badgr launch cline "<task>"`, `badgr job cline
  "<instruction>" --check "<command>"`, `POST /v1/jobs` with `type: "agent"`
- Existing CLI command pattern to follow: `apps/cli/src/commands/program.ts`,
  `apps/cli/src/commands/schedule.ts`, `apps/cli/src/commands/connect.ts`
