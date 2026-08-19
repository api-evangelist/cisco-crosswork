---
name: cisco-crosswork-zero-touch-onboard-a-device
description: >-
  Onboard network devices through Crosswork Zero Touch Provisioning — create the ZTP profile, register the devices
  (individually, in bulk, or from CSV), onboard them and check status. Use when asked to provision, image or
  zero-touch-onboard routers with Crosswork.
generated: '2026-08-19'
method: generated
source: >-
  openapi/cisco-crosswork-cwztpprofile-api-openapi.yml, openapi/cisco-crosswork-cwztpdevice-api-openapi.yml,
  openapi/cisco-crosswork-cwztpdynamicconfigs-api-openapi.yml
api: cisco-crosswork:cisco-crosswork-cwztpdevice-api
base_path: /crosswork/ztp/
operations:
  - GetProfileList
  - SetProfiles
  - UpdateProfile
  - DeleteProfile
  - SetDevices
  - GetDevices
  - UpdateDevice
  - UpdateDeviceStatus
  - DeleteDevice
  - OnboardDevice
  - ImportDevices
  - ExportDevice
  - ZtpsampleService
  - GetDevicesPolicies
  - GetZtpCount
---

# Zero-touch onboard devices with Crosswork ZTP

Base path `/crosswork/ztp/` on the customer's Crosswork host. Authenticate via
`POST /crosswork/sso/v1/tickets` then `POST /crosswork/sso/v2/tickets/jwt`, and send
`Authorization: Bearer {jwt}`.

Note the shape of this API before you start: **reads are POSTs.** `GetDevices` is `POST /v1/devices/query`, not a
GET. That is consistent across the ZTP family (`GetProfileList` is `POST /v1/profiles/query`,
`GetDevicesPolicies` is `POST /v1/devices/policies/query`). Do not assume REST verb conventions here.

## Steps

1. **Check what profiles exist.** `GetProfileList` (`POST /v1/profiles/query`) returns the ZTP profiles — the
   image and configuration a device will receive.
2. **Create the profile if needed.** `SetProfiles` (`POST /v1/profiles`) adds profiles in bulk. `UpdateProfile`
   (`PUT /v1/profiles`) updates an individual profile — Cisco's own description warns a profile cannot be updated
   in certain states, so read the current state first. `DeleteProfile` (`DELETE /v1/profiles`) removes one.
3. **Register the devices.** Three routes, pick one deliberately:
   - `SetDevices` (`POST /v1/devices`) — bulk add from a JSON body. Cisco documents this as all-or-nothing.
   - `ImportDevices` (`POST /v1/devices/import`) — bulk add from CSV. Call `ZtpsampleService`
     (`POST /v1/devices/csvtemplate`) first to get the header row and sample data so the CSV matches.
   - `OnboardDevice` (`POST /v1/onboarding`) — a single device.
4. **Verify.** `GetDevices` (`POST /v1/devices/query`) lists devices with their status. `GetZtpCount`
   (`POST /v1/query/count`) gives totals for devices, profiles and devices with results — the cheapest way to
   confirm a bulk import landed.
5. **Correct.** `UpdateDevice` (`PUT /v1/devices`) only applies when the device is unprovisioned.
   `UpdateDeviceStatus` (`PATCH /v1/devices`) applies when it is in progress or provisioned. Choosing the wrong one
   for the device's current state is the usual cause of a rejected call — query the device first.
6. **Export for the record.** `ExportDevice` (`POST /v1/devices/export`) writes the device list to CSV.
7. **Remove.** `DeleteDevice` (`DELETE /v1/devices`) takes the UUID of each device to remove.

## Handling failures

The ZTP family declares status codes without body schemas — a 404 arrives as "Not Found" and nothing more. There
is no error code registry to look up and no problem+json. Log the status code, the operation and the request body,
because the response will not tell you anything the status code did not.

No idempotency key exists. Bulk operations are the risk: a timed-out `ImportDevices` must be reconciled with
`GetDevices` before being retried, or devices will be double-registered.
