---
name: Trigger QA Wolf on deploy and gate the pipeline
description: >-
  Notify QA Wolf of a successful deployment to start a test run, then poll the
  run outcome to pass or fail a CI pipeline. Grounded in QA Wolf's v0 REST API.
api: openapi/qa-wolf-rest-openapi.yml
operations:
  - notifyDeploySuccess
  - getCiGreenlight
  - notifyEnvironmentTerminated
---

# Trigger QA Wolf on deploy and gate the pipeline

Use this skill to run QA Wolf end-to-end tests as part of a deployment and block
promotion until the run resolves. Prefer the `@qawolf/ci-sdk` package or the
`qawolf` CLI for production use; the raw REST calls below are the underlying
contract.

## Authentication

All calls use a bearer token — the workspace API key. Set it as `QAWOLF_API_KEY`
(Workspace Settings -> Integrations -> API Access) and send
`Authorization: Bearer $QAWOLF_API_KEY`. See `authentication/qa-wolf-authentication.yml`.

## Steps

1. **Notify QA Wolf of the deploy** — call `notifyDeploySuccess`
   (`POST https://app.qawolf.com/api/webhooks/deploy_success`) with `branch`,
   `sha`, and optionally `deployment_url`, `deployment_type`, `variables`, and a
   `deduplication_key`. A `200` returns a `results[]` array; capture each
   `created_suite_id` (a `duplicate_suite_id` means the run was deduplicated, a
   `failure_reason` means no run was created).

2. **Poll for the outcome** — call `getCiGreenlight`
   (`GET /api/v0/ci-greenlight/{rootRunId}`) on an interval until the run reaches
   a terminal state, then pass or fail the pipeline on the reported outcome.

3. **Clean up ephemeral environments** — if you spun up a preview environment,
   call `notifyEnvironmentTerminated`
   (`POST /api/webhooks/environment_terminated`) when it is torn down so QA Wolf
   stops targeting it and promotes flows to the static base environment.

## Error handling

Errors are plain HTTP status codes (see `errors/qa-wolf-problem-types.yml`):
`401` invalid API key, `403` disabled team, `405` wrong method. Do not retry on
`401`/`403`. The `deduplication_key` on `notifyDeploySuccess` controls run
dedup/cancellation — reuse a stable key per logical deploy to avoid duplicate
suites (see `conventions/qa-wolf-conventions.yml`).
