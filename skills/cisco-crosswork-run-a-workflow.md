---
name: cisco-crosswork-run-a-workflow
description: >-
  Find a Crosswork Workflow Manager workflow, validate the input you intend to pass it, execute it as a job, follow
  the run, and stop it if it goes wrong. Use when asked to run, trigger, monitor or cancel a CWM automation.
generated: '2026-08-19'
method: generated
source: openapi/cisco-crosswork-cwm-workflow-api-openapi.yml, openapi/cisco-crosswork-cwm-jobs-api-openapi.yml
api: cisco-crosswork:cisco-crosswork-cwm-workflow-api
base_path: /crosswork/cwm/v2
operations:
  - list_workflows
  - list_workflow_tags
  - get_workflow_by_name_version
  - validate_workflow_input
  - execute_job
  - list_jobs
  - describe_job
  - get_job_history
  - cancel_job
  - terminate_job
---

# Run a Crosswork Workflow Manager workflow

## Before you start

Crosswork is customer-deployed. There is no Cisco-hosted endpoint — every URL below is relative to
`https://{cwm-host}:{cwm-port}/crosswork/cwm/v2`, where the host is the machine the operator installed CWM on.

Get a token first. It is a two-step exchange against that same host:

1. `POST /crosswork/sso/v1/tickets` with `Content-Type: application/x-www-form-urlencoded` and the account's
   username and password → Ticket Granting Ticket.
2. `POST /crosswork/sso/v2/tickets/jwt` with the TGT and the forwarding service URL → JWT.

Send `Authorization: Bearer {jwt}` on every call. Access is further constrained by the RBAC policy configured in
the instance, so a 403 means the account lacks the role, not that the call was malformed.

## Steps

1. **Find the workflow.** `list_workflows` (`GET /workflow`) returns the catalogue. `list_workflow_tags`
   (`GET /workflow/tags`) narrows it if the operator tags their workflows.
2. **Pin the version.** Workflows are addressable two ways and they are not interchangeable — by UUID
   (`get_workflow_by_id`, `GET /workflow/{workflowId}`) or by name and version
   (`get_workflow_by_name_version`, `GET /workflow/name/{workflowName}/version/{workflowVersion}`). Prefer
   name+version when a human named the workflow; prefer the UUID when you were handed one. Never assume the latest
   version is the one intended.
3. **Validate the input before you run anything.** `validate_workflow_input`
   (`POST /workflow/name/{workflowName}/version/{workflowVersion}/validateInputData`) checks the payload against
   the workflow's declared input. Do this every time. Crosswork declares no idempotency mechanism anywhere in its
   contract, so a job you started by mistake cannot be de-duplicated — it can only be cancelled after the fact.
4. **Execute.** `execute_job` (`POST /job`) starts the run. Capture the `jobId` and `runId` from the response;
   every follow-up call needs both.
5. **Follow it.** `describe_job` (`GET /job/{jobId}/runs/{runId}`) for current state, `get_job_history`
   (`GET /job/{jobId}/runs/{runId}/events`) for the event trail. `list_jobs` (`GET /job`) pages the whole set —
   note the pagination parameters here are `filter.pageSize` and `filter.pageNum`, NOT the `pageSize`/`pageNumber`
   used elsewhere in Crosswork.
6. **Stop it if needed.** `cancel_job` (`POST /job/{jobId}/runs/{runId}/cancel`) requests a graceful stop;
   `terminate_job` (`POST /job/{jobId}/runs/{runId}/terminate`) is the hard stop. Prefer cancel.

## Handling failures

Errors from this family use the `server.HTTPError` envelope — `{ "code": <int>, "message": <string> }`. That is
all you get: there is no error code registry, no `type` URI, no `application/problem+json`, and no remediation
field. Treat `message` as human-readable text, not as a machine-parsable code.

There is no rate limiting to back off from — Crosswork declares no 429 and returns no `Retry-After` or
`RateLimit-*` header. If calls start failing under load, the cause is the deployment's own capacity, and the only
signal you will get is latency or a 500.

Because there is no idempotency key, **do not blind-retry `execute_job` on a timeout.** Call `list_jobs` and check
whether the run already started before issuing it again.
