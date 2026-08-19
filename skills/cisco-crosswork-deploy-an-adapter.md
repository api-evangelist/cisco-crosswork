---
name: cisco-crosswork-deploy-an-adapter
description: >-
  Upload, deploy and inspect a Crosswork Workflow Manager adapter, and wire up the resource type, resource and
  secret it needs to run. Use when asked to add a CWM integration, install an adapter, or find out which
  activities a workflow can call.
generated: '2026-08-19'
method: generated
source: >-
  openapi/cisco-crosswork-cwm-adapters-api-openapi.yml, openapi/cisco-crosswork-cwm-resources-api-openapi.yml,
  openapi/cisco-crosswork-cwm-secrets-api-openapi.yml
api: cisco-crosswork:cisco-crosswork-cwm-adapters-api
base_path: /crosswork/cwm/v2
operations:
  - list_adapters
  - file_upload
  - deploy_adapter
  - get_adapter
  - update_adapter
  - patch_adapter
  - delete_adapter
  - list_resource_types
  - get_resource_type
  - list_resources
  - create_resource
  - get_resource
  - list_secret_types
  - create_secret
  - get_secret
---

# Deploy a Crosswork Workflow Manager adapter

Adapters are how CWM reaches anything outside itself — an OSS, a BSS, a device controller. Each adapter carries a
set of **activities** that workflows can call, and each activity declares its own `inputSchema` and `outputSchema`.
Reading the adapter list is therefore the fastest way to learn what a given CWM deployment can actually do.

Authenticate first: `POST /crosswork/sso/v1/tickets` then `POST /crosswork/sso/v2/tickets/jwt`, then
`Authorization: Bearer {jwt}` on every call.

## Steps

1. **See what is already there.** `list_adapters` (`GET /adapter`) returns every uploaded adapter with its
   `adapterId`, `deployedStatus`, `defaultStatus`, `inUseStatus`, `resourceTypeId`, `hashSum`, `cwmVersionId`,
   `createdAt`/`updatedAt` and the full `activities[]` array. Read `activities[]` before writing any workflow —
   it is the authoritative capability list.
2. **Upload the adapter file.** `file_upload` (`POST /adapter`) takes the adapter package as a multipart upload.
   The request media type is `application/x-gzip` on this operation, not JSON.
3. **Deploy it.** Uploading is not deploying. `deploy_adapter` (`POST /adapter/{adapterId}/deploy`) creates the
   plugin from the uploaded file. Confirm with `get_adapter` (`GET /adapter/{adapterId}`) that `deployedStatus` is
   true before you rely on it.
4. **Wire the resource type.** An adapter points at a `resourceTypeId`. Use `list_resource_types`
   (`GET /resourceType`) and `get_resource_type` (`GET /resourceType/{resourceTypeId}`) to read the shape, then
   `create_resource` (`POST /resource`) to create the concrete resource — the connection details for the system
   the adapter talks to.
5. **Store the credentials as a secret, not in the resource.** `list_secret_types` (`GET /secretType`) then
   `create_secret` (`POST /secret`). Reference the `secretId` from the workflow. Never inline a credential into a
   workflow definition or a resource body.
6. **Updating.** `update_adapter` (`PUT /adapter/{adapterId}`) replaces the adapter file; `patch_adapter`
   (`PATCH /adapter/{adapterId}`) changes metadata only. `delete_adapter` (`DELETE /adapter/{adapterId}`) will
   fail or orphan workflows if `inUseStatus` is true — check it first.

## Handling failures

`server.HTTPError` (`{code, message}`) is the envelope for this family. A 409 is the most common meaningful
failure here and usually means the adapter or resource already exists under that identifier.

There is no idempotency key. If `file_upload` times out, call `list_adapters` and compare `hashSum` before
uploading again — re-uploading blind is how you end up with duplicate adapters.
