# U-tec / Xthings OpenAPI: field notes and corrections

Unofficial notes on the U-tec (Ultraloq, "Xthings Home", formerly "U home") OpenAPI, written while building a
small self-hosted app that manages guest door codes on Ultraloq locks. Everything marked **verified** was observed
against real locks in September 2026. Where the official documentation says something different, the difference
is called out.

This is not affiliated with U-tec or Xthings. Behaviour can change without notice; treat the dates as part of
the claim.

Tested on locks that report these values in Discovery: models `Bolt-F`, `Bolt-F-W`, `U-Bolt-WiFi`; handle types
`utec-lock` and `utec-lock-sensor`.

## Where the documentation is wrong or silent

| Topic | Documentation | What actually happens (verified) |
|---|---|---|
| Passcode type | `password` is "a 4 to 8-digit integer" | Send it as a **string**. `"0993"` is stored and returned with its leading zero. `get` returns it as a string too. |
| Door sensor attribute | `st.doorSensor` attribute `sensorState` | The live attribute is **`doorState`** (`Closed` / `Open`). |
| User types | 0 Normal, 2 Temporary, 3 Admin | Also **1 = Owner**. It appears in `list`; it cannot be created through the API. |
| User management on sensor locks | The handler table lists only Lock, BatteryLevel, DoorSensor for `utec-lock-sensor` | `st.lockUser` works on `utec-lock-sensor` locks as well. |
| Queued changes | Not mentioned | A change accepted while the lock is offline is **not delivered when the lock reconnects**. See "Sync behaviour". |
| Pending create vs pending delete | Not mentioned | `list` cannot tell them apart. See "Sync behaviour". |
| Normal user fields | The same user object for all types | A Normal user has **no** schedule fields at all. See "User records". |
| `limit: 0` | "0 or absent = unlimited" | Correct, and `get` then **omits** `limit` entirely. Treat absent and 0 as equal when comparing. |
| Homes and rooms | Not mentioned | Discovery has **no** home, room, location or group field. Grouping exists only in the vendor app. |
| Header field name | Some official request examples spell it `playloadVersion` | `payloadVersion` is the spelling that is known to work. |

## Transport

All calls are `POST https://api.u-tec.com/action` with `Content-Type: application/json` and
`Authorization: Bearer <access token>`.

```json
{
  "header": { "namespace": "Uhome.Device", "name": "Command", "messageId": "<uuid v4>", "payloadVersion": "1" },
  "payload": { "devices": [ { "id": "AA:BB:CC:DD:EE:FF",
    "command": { "capability": "st.lockUser", "name": "list" } } ] }
}
```

Namespaces: `Uhome.Device` (`Discovery`, `Query`, `Command`), `Uhome.User` (`Get`, `Logout`),
`Uhome.Configure` (`Set`, registers a push URL; documented, not tested here).

Errors come back in two places. A request-level failure is `payload.error`. A per-device failure is
`payload.devices[n].error` with HTTP 200, so check both. Codes seen live:

| Code | When |
|---|---|
| `LOCK_PASSCODE_EXISTS` | `add` with a passcode already on that lock |
| `NO_PERMISSION` | Any command for a device that has been removed from the account |

## OAuth

Credentials come from the Xthings Home app (My Account, OpenAPI, Activate; app version 3.5.5 or later).

Authorize at `https://oauth.u-tec.com/authorize?response_type=code&client_id=...&redirect_uri=...&scope=openapi`,
then exchange at `POST https://oauth.u-tec.com/token` (`authorization_code`, later `refresh_token`).

Two things to know. The token response is wrapped (verified): read `data.access_token`, not `access_token`, from
`{ "code": ..., "data": { "access_token", "refresh_token", "expires_in", ... } }`. And a refresh response may
carry a new refresh token, so persist whatever comes back and make sure only one refresh runs at a time. Two
parallel refreshes, or a second client (for example Postman) refreshing with the same token, risk leaving the
stored token dead.

## Discovery and status

`Discovery` returns, per device: `id`, `name`, `category`, `handleType`, `deviceInfo` (`manufacturer`, `model`,
`hwVersion`), `attributes` (only populated for lights), `customData`. On the tested locks the `id` is a MAC-style string.

`Query` returns `states[]` of `{ capability, name, value }`:

| capability / name | Values | Note |
|---|---|---|
| `st.healthCheck` / `status` | `Online`, `Offline` | The only reliable online signal |
| `st.lock` / `lockState` | `Locked`, `Unlocked`, `Jammed`, `Unknown` | |
| `st.batteryLevel` / `level` | integer 1 to 5 | A level, not a percentage |
| `st.doorSensor` / `doorState` | `Closed`, `Open` | Only on `utec-lock-sensor`; docs call it `sensorState` |

## User records (`st.lockUser`)

Commands: `list`, `get {id}`, `add {user}`, `update {user + id}`, `delete {id}`.

`list` returns `{ id, name, type, status, sync_status }` per user. The passcode and the schedule are only in `get`,
so reading everything costs one `get` per user per lock.

Temporary user (type 2), as sent and as returned:

```json
{ "name": "Guest", "type": 2, "password": "0913",
  "daterange": ["2026-09-10 15:00", "2026-09-12 11:00"],
  "weeks": [0,1,2,3,4,5,6], "timerange": ["00:00","23:59"], "limit": 0 }
```

`weeks` uses 0 = Sunday to 6 = Saturday. `timerange` is a daily window inside `daterange`. `limit` is the number
of unlocks allowed, 0 or absent meaning unlimited.

Normal user (type 0), verified 2026-09-06: `add` with only `{ "name", "type": 0, "password" }` is accepted and
stored. `get` returns `{ id, name, type, status, sync_status, password }` and nothing else. Do not send schedule
fields for a Normal user and do not expect them back.

`update` with changed values (name, `daterange`) changes the record in place: same `id`, `sync_status` returns to 1
once the lock has applied it (verified 2026-09-06). Changing the passcode through `update` was not tested.

Identity: the lock rejects a second user with the same **passcode**, but accepts any number of users with the
same **name**. If you manage several locks at one property, the passcode is the only stable key for matching a
person across locks; record ids differ per lock.

## Sync behaviour

Every write returns immediately with `st.deferredResponse` (`seconds`, 5 in practice) and no error. That only
means the cloud accepted it. The lock applies it later; the proof is `sync_status: 1` in a following `list`.
Older Wi-Fi models can take up to a minute.

`sync_status: 1` is durable: it means the lock physically has the record, and it stays 1 if the lock later goes
offline.

A lock that is offline accepts `add` with **no error**. The record shows up in `list` with `sync_status: 0`.
To tell "accepted and applied" from "accepted and queued" you need `st.healthCheck` from `Query`.

**Queued changes are not pushed when the lock comes back.** Verified 2026-09-05: a create queued while a lock was
offline was still `sync_status: 0` well after the lock reported `Online` again. Sending the same
record again as `update` (same data, plus the record `id` from `list`) made the lock take it within a minute.
Re-sending is idempotent. The same applies to deletes: re-send `delete` with the record id.

**`list` cannot distinguish a pending create from a pending delete.** A record waiting to be removed still
appears with `status: 1` and `sync_status: 0`, exactly like one waiting to be created. If you need to show the
difference, keep your own log of the deletes you issued (lock id, passcode, record id) and clear an entry when
the record disappears from `list`.

A pattern that works: poll each lock on an interval; for every lock that is `Online`, re-send each record with
`sync_status: 0` as `update`, and each record in your delete log as `delete`; back off per record and stop after
a few attempts so a lock that ignores a record does not get hammered.

To cut the number of `get` calls, cache each record's detail keyed by lock and record id and reuse it while its
`list` entry (`name`, `type`, `status`, `sync_status`) is unchanged, with a time limit of a few minutes.

## Time zones (observation, not fully explained)

Schedule values written through the API are plain wall-clock strings (`"YYYY-MM-DD HH:MM"`) in the lock's local
time. They read back unchanged and behaved as local time in use.

The vendor app does its own time handling. A schedule created in the app read back through the API shifted by
several hours and with a malformed `timerange` of `["00:00","00:23"]`, while the app itself displayed the
intended times. Used from a phone in a different time zone than the lock, the app also showed times that did not
match the API values. Deleting such a record and creating it again through the API produced values that read
back correctly.

Practical rule: if you write schedules through the API, also edit them through the API, and do not compare them
with what the vendor app shows on a phone in another time zone.

## Not available

There is no property, home or room structure in the API, so any grouping of locks has to live in your own
configuration. The only lock field you can set to carry meaning is the device name.

There is no official way to learn that a record's `sync_status` changed other than polling. The documented push
registration (`Uhome.Configure` / `Set`, a URL plus a secret you choose) may cover device state; it was not
tested for these notes.

## Related: lock shows Online but cloud keeps dropping

If `st.healthCheck` flips between `Online` and `Offline`, writes stay at `sync_status: 0`, or the vendor app shows
*Cloud Status: Disconnected* while Wi‑Fi signal is good, check the network before the API. On an ASUS AiMesh
setup the cause was the lock roaming between access points; see
[wifi-aimesh-roaming.md](wifi-aimesh-roaming.md).

## Sources

Official documentation: <https://doc.api.u-tec.com/> and the article set at <https://developer.xthings.com/>
(Developer Foundational APIs, Production Capabilities, Lock User Management).

Community implementations that parse real responses and were used to cross-check the Discovery fields:
<https://github.com/LF2b2w/Uhome-HA>, <https://github.com/jellojank/uhome>, <https://github.com/aljopro/ha-xthings>.

## Licence

MIT. See [LICENSE](LICENSE).
