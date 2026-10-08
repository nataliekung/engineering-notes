# Ray Jobs: a 500 that should have been a 503

A Ray Jobs entrypoint can run on a worker node, for example when it asks for one CPU and the head advertises none. When that job finishes, its status stays available from the head. If the worker node is then removed, by an autoscaler or by hand, two different facts get reported in two different ways:

- `GET /api/jobs/{id}` returns 200 with `SUCCEEDED`.
- `GET /api/jobs/{id}/logs` returns 500 with an internal traceback, because the request is still proxied to the job's recorded `driver_agent_http_address`, which no longer resolves.

The logs were node-local and are gone. That is a known, classifiable state, not an internal error.

## Issue #65402

The issue gives a two-node reproduction on Ray 2.42.0: start a head with `--num-cpus=0`, join a worker with one CPU, submit a one-line entrypoint with `--entrypoint-num-cpus=1` so it lands on the worker, wait for `SUCCEEDED`, stop the worker, then fetch the logs. It asks maintainers to pick a contract before any code is written: either `410 Gone` if the logs are irrecoverable, or `503 Service Unavailable` if Ray treats the condition as possibly temporary, in both cases as a structured, machine-readable error rather than a traceback. It explicitly does not ask Ray to persist job logs.

## PR #65405

The PR implements the 503 variant:

- Return a structured 503 when the job exists but its driver agent is no longer registered or reachable.
- Keep successful log retrieval, the 404 for unknown jobs, and the public SDK's existing `RuntimeError` behavior unchanged.
- Document the JSON error body and bump the Jobs API version.

219 lines across 5 files. Tests cover the live-agent path, a missing driver node, an unreachable driver agent, and the preserved 404; the SDK test covers logs after the driver node is removed. The upstream microcheck passed. One thing I could not do: rerun the two-node end-to-end test locally, because my checkout did not have a compiled `ray._raylet`, and the public CI summary does not expose individual job execution. The PR says so.

## Status

Approved by two reviewers on 2026-08-13 and 2026-08-14. Not merged as of 2026-10-07; two review threads remain, one of them a question about which exceptions to catch, where I offered `except (KeyError, ValueError)`.
