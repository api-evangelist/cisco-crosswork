---
name: cisco-crosswork-check-health-and-triage-alarms
description: >-
  Read the health of a Crosswork deployment and its applications, query alarms, acknowledge/note/clear them, and
  set up the syslog and REST destinations alarms are forwarded to. Use when asked whether Crosswork is healthy,
  what is alarming, or where alarms are being sent.
generated: '2026-08-19'
method: generated
source: >-
  openapi/cisco-crosswork-crosswork-health-api-api-openapi.yml, openapi/cisco-crosswork-alarms-api-openapi.yml,
  openapi/cisco-crosswork-alert-api-api-openapi.yml
api: cisco-crosswork:cisco-crosswork-crosswork-health-api-api
base_path: /crosswork/platform
operations:
  - GetSystemSummary
  - GetAppsSummary
  - GetAppServicesSummary
  - GetAppServiceDetail
  - GetAlarms
  - AcknowledgeAlarms
  - NoteAlarms
  - ClearAlarms
  - RaiseEvent
  - GetSyslogDestinations
  - AddSyslogDestination
  - DelSyslogDestination
  - GetRestDestinations
  - AddRestDestination
  - DelRestDestination
---

# Check Crosswork health and triage alarms

Base path `/crosswork/platform` on the customer's Crosswork host. Authenticate via
`POST /crosswork/sso/v1/tickets` then `POST /crosswork/sso/v2/tickets/jwt`; send `Authorization: Bearer {jwt}`.

This is the one Crosswork family where health really is a GET. Alarm queries are not — `GetAlarms` is
`POST /v1/query`.

## Triage order

1. **Whole platform first.** `GetSystemSummary` (`GET /v1/platform/health`) returns a health summary for every
   application running on the Crosswork cluster. Start here; it tells you whether you are looking at one broken
   service or a sick deployment.
2. **Then applications.** `GetAppsSummary` (`GET /v1/apps/health`) for all applications,
   `GetAppServicesSummary` (`GET /v1/apps/{AppName}/health`) for one application's services.
3. **Then the specific service.** `GetAppServiceDetail` (`GET /v1/apps/{AppName}/{ServiceName}/health`).
4. **Query the alarms.** `GetAlarms` (`POST /v1/query`) takes the selection criteria in the request body.
5. **Work them.** `AcknowledgeAlarms` (`PUT /v1/ack`) to take ownership, `NoteAlarms` (`PUT /v1/note`) to record
   what you found, `ClearAlarms` (`PUT /v1/clear`) once resolved. Acknowledge before clearing — clearing an alarm
   nobody acknowledged loses the triage record.
6. **Raise one yourself** when an external check finds something Crosswork cannot see: `RaiseEvent`
   (`POST /v1/event`).

## Where alarms go

Crosswork forwards alarms to destinations you register. Two kinds, managed separately:

- Syslog: `GetSyslogDestinations` (`POST /v1/syslog-dest/query`, always returns the full list),
  `AddSyslogDestination` (`POST /v1/syslog-dest`), `DelSyslogDestination` (`DELETE /v1/syslog-dest`).
- REST: `GetRestDestinations` (`POST /v1/rest-dest/query`), `AddRestDestination` (`POST /v1/rest-dest`),
  `DelRestDestination` (`DELETE /v1/rest-dest`).

Both "query" operations return the complete list rather than a page, so there is nothing to paginate — but also no
way to filter server-side.

## Handling failures

Responses in this family use the platform's own schemas; error responses largely carry a status code and a
description with no body schema. There is no problem+json, no error registry and no rate limiting.

Health endpoints are cheap and safe to poll. Alarm mutations are not idempotent — `AcknowledgeAlarms` and
`ClearAlarms` should be issued once and their effect confirmed with `GetAlarms`, not retried blind.
