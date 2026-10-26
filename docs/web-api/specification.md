# Clevertouch Web API specification

[Overview](../../README.md) · [Shared values](../common.md)

This is the account-facing API implemented by `hemphen/clevertouch`. Core
operations below combine **Client** evidence from that library and the supplied web
app with observed account reads and schedule/mode delivery from 26 October 2026.
Observations describe one Purmo/LVI installation, not all providers.

## Transport and authentication

| Setting | Purmo/LVI value |
| --- | --- |
| API base | `https://e3.lvi.eu/api/v0.1/` |
| Token endpoint | `https://auth.e3.lvi.eu/realms/purmo/protocol/openid-connect/token` |
| OAuth client ID | `app-front` (public client identifier, not a secret) |
| API method | `POST` |
| Request content type | `application/x-www-form-urlencoded` |
| API authorization | `Authorization: Bearer <access-token>` |
| Response | JSON |

The Python client obtains tokens by posting either form below to the token
endpoint. Password-grant availability is provider policy, not guaranteed by this
reference; the web app instead manages a Keycloak login session.

| Grant | Form fields |
| --- | --- |
| Password | `grant_type=password`, `client_id=app-front`, `username=<email>`, `password=<password>` |
| Refresh | `grant_type=refresh_token`, `client_id=app-front`, `refresh_token=<refresh-token>` |

On HTTP 200, the client reads `access_token`, `refresh_token` and `expires_in`
(seconds). Retain a returned replacement refresh token and use `expires_in`
rather than the library's fallback lifetimes as a protocol constant. The client
treats HTTP 400/401 at the token endpoint as authentication failure. The web app
attempts refresh with 40 seconds of remaining validity; that is a client choice.

The app adds `lang` (default `en_GB`). It also sends a legacy body field `token`
in many calls while supplying the bearer header. The Python implementation does
not require that body field for the three core endpoints below. These are
separate token mechanisms; do not substitute a clevertouch-upstream cursor for either.

## Form encoding and response envelope

Nested objects are flattened to bracketed form names. For example:

```text
query[id_device]=C007-006
query[consigne_confort]=680
```

On the wire, percent-encode form keys and values normally. This is not a JSON
request body. Brackets are structural field names, not literal JSON text.

The response envelope has this shape (placeholder prose is illustrative):

```json
{
  "code": {"code": "1", "key": "OK", "value": "<status description>"},
  "data": {},
  "parameters": {}
}
```

| Application status | Client interpretation |
| --- | --- |
| `code.code = 1`, `code.key = OK` | Successful read |
| `code.code = 8`, `code.key = OK_SET` | Successful mutation / accepted query |
| Other values | Not success under the Python client's checks; preserve status details |

The Python client accepts 1 or 8 for reads, only 8 for writes; the app commonly
checks the named key. `parameters` is optional in the Python parser. HTTP status
and application status must both be checked. A complete error-code catalogue is
not established, and a timeout after a write leaves delivery uncertain.

## Core endpoints

All paths are relative to the API base and include their trailing slash.

| Path | Request fields | Response `data` used by clients |
| --- | --- | --- |
| `human/user/read/` | `email` | `user_id`, `smarthomes[]` |
| `human/smarthome/read/` | `smarthome_id` | Home record, `zones[]`, `devices[]` |
| `human/query/push/` | `smarthome_id`, `context`, `peremption`, `query[...]` | No stable success payload is required by the Python client; check envelope status |

A home entry supplies `smarthome_id` and `label`. Zone entries supply `num_zone`
and `zone_label`. A minimal schematic home response is:

```json
{
  "smarthome_id": "HOME_EXAMPLE",
  "label": "Example home",
  "zones": [{"num_zone": "7", "zone_label": "Example room"}],
  "devices": [{
    "id": "DB_DEVICE_EXAMPLE",
    "id_device": "C007-006",
    "num_zone": "7",
    "label_interface": "Example radiator",
    "gv_mode": "0",
    "nv_mode": "0",
    "temperature_air": "680",
    "consigne_confort": "698",
    "heating_up": "1"
  }]
}
```

This deliberately shows only a subset of fields, not a complete server response.
Use returned identifiers as opaque values, and tolerate additional fields.

## Radiator fields

| Field | Meaning / use |
| --- | --- |
| `gv_mode` | Mode read by both clients; values below |
| `nv_mode` | Written alongside `gv_mode`; its separate semantics are not established |
| `temperature_air` | Measured temperature; read-only |
| `consigne_confort` | Stored Comfort setpoint |
| `consigne_eco` | Stored Eco setpoint |
| `consigne_hg` | Stored Frost setpoint |
| `consigne_boost` | Stored Boost setpoint |
| `consigne_manuel` | Manual setpoint; also mirrored by the app during several mode changes |
| `consigne_off` | Used by the app for Off display; not an established temperature-control command |
| `min_set_point`, `max_set_point` | App-consumed setpoint bounds, in temperature units |
| `heating_up` | Python client considers string `"1"` active heating |
| `time_boost` | Boost duration in seconds in client writes; do not equate every read with a fresh configured duration |
| `time_boost_format_chrono` | Remaining-time object with `d`, `h`, `m`, `s`; convert as `86400*d + 3600*h + 60*m + s` |
| `date_start_boost` | App-consumed start marker; wire date/time semantics are not established |
| `programme` | Encoded schedule; see [schedule operations](specification.md#weekly-schedules) |

For common modes 0, 1, 2, 3, 4, 8 and 11, see the [shared mode table](../common.md#heating-modes).
The app additionally recognises 5 (fan), 6 (fan disabled), 13 (program),
15 (manual) and 16 (program, Boost phase). These extra modes are **Client-only**,
with device/provider-dependent support; they are not observed Purmo RF modes.
The library chooses 11 when requesting Program, whereas the app chooses a
program phase based on the device. There is no established universal write code
for every provider's Program mode.

## Submitting device commands

For the operations below, use `context=1`. The Python client sends
`peremption=15000`; the app starts at 20000 and adapts among 15000–30000.
These are client expiry/timing values in milliseconds, not a guaranteed delivery
latency or a proven server-side expiry contract.

Every command includes `query[id_device]`. Add the fields for the desired change:

| Operation | Additional `query[...]` fields sent by the Python client |
| --- | --- |
| Set stored temperature | The corresponding `consigne_*` field, in raw units |
| Select mode | Both `gv_mode` and `nv_mode`, with the same mode code |
| Configure Boost duration | `time_boost=<seconds>` |
| Start Boost with a target and duration | `gv_mode=4`, `nv_mode=4`, `consigne_boost=<raw>`, `time_boost=<seconds>` |
| Switch a light or outlet | `on_off=1` for on, `on_off=0` for off |

Example: request Comfort at 21 °C. This is a synthetic, form-encoded request;
`HOME_EXAMPLE`, `C007-006` and the token must be replaced with authorised values.

```http
POST /api/v0.1/human/query/push/ HTTP/1.1
Host: e3.lvi.eu
Authorization: Bearer <access-token>
Content-Type: application/x-www-form-urlencoded

smarthome_id=HOME_EXAMPLE&context=1&peremption=15000&query%5Bid_device%5D=C007-006&query%5Bgv_mode%5D=0&query%5Bnv_mode%5D=0&query%5Bconsigne_confort%5D=698
```

The app's non-Fenix mode-change path also mirrors the selected target into
`consigne_manuel` (and uses zero for Off). It may send `time_boost=0` when leaving
a timer. Those are observed client construction choices, not demonstrated
requirements for a simple setpoint write. The Fenix branches construct some
commands differently, so do not apply them to Purmo by analogy.

After `OK_SET`, re-read the home to check the panel/server state, not RF delivery. The app's local updates and
countdown are optimistic. There is no documented command-specific transaction
ID or end-to-end acknowledgement in this core API subset. Do not blindly retry
a timed command after an ambiguous transport failure: it may already have run.

The operations below are **Client** evidence from the supplied app (`script.js`,
version string 2.8.1). Requests use the Clevertouch Web API's POST/form/bearer conventions.
Response fields listed are those consumed by the app, not complete schemas.
The 26 October observations additionally cover schedule rendering, statistics,
errors, liveness and a compact schedule queued through `query/push/`.
Structured `apply_program/` and `copy_program/` remain Client-only evidence.

The app includes a legacy body `token` on most of these requests. The table names
it where present; its necessity alongside bearer authentication is unverified.
`lang` is added by the common request wrapper.

## Weekly schedules

**Client:** the web app sends a custom weekly schedule as named-day intervals to
`human/device/apply_program/`. It does **not** encode those intervals into the
compact `programme` string in JavaScript. The conversion takes place on the
server. Sending intervals therefore does not require reverse-engineering the
compact string first.

### Operations and identifiers

| Operation | Endpoint (POST) | Fields in addition to `lang` |
| --- | --- | --- |
| Save an entire edited week | `human/device/apply_program/` | `token`, `device_id`, `peremption`, `new=true`, `program[...]` |
| Add a selected interval on selected days | Same | `token`, `device_id`, `peremption`, `program[...]`; the app omits `new` |
| Copy another device's week | `human/device/copy_program/` | `token`, `device_id_from`, `device_id_to`, `peremption` |
| Store an already-encoded template | `human/device/edit/` | `token`, `smarthome_id`, `id_device`, `programme` |
| Queue that encoded template to the panel | `human/query/push/` | `token`, `smarthome_id`, `context=1`, `peremption`, `query[id_device]`, `query[programme]` |
| Read a decoded week for display | `human/sandbox/convert_program/` | `device_id`, `now` (Unix seconds), `height` or `width` (rendering size) |

`device_id`, `device_id_from` and `device_id_to` are cloud database IDs from
`devices[].id`. `id_device` is the panel-local ID from `devices[].id_device`.
Confusing these two identifiers changes which kind of object the server receives.
`token` is the app's legacy body token, alongside its bearer header; whether it
can be omitted on schedule endpoints has not been verified.

The full-week path is `saveEntireProgram`; the interval-add path builds a week
containing the new interval and default periods. The differing use of `new`
strongly suggests replacement versus incremental application. The exact server
merge rules for omitted `new`, overlaps and existing intervals are unknown;
follow the full-week request shape when submitting an entire replacement.

### Week and interval schema

| Field | Representation |
| --- | --- |
| `program` | Object with all seven keys: `Sun`, `Mon`, `Tue`, `Wed`, `Thu`, `Fri`, `Sat` |
| Each day | Array of intervals in chronological order |
| `start`, `end` | `HH:MM` strings, with 15-minute boundaries supported by the app |
| `value` | String `comfort`, `eco`, `booster`, or `false` |
| `new` | Form value `true` for a full-week save; not the word `new` inside each day |
| `peremption` | Milliseconds; use the same command timing convention as other app requests (app initially 20000) |

`"false"` is a string meaning an unselected/default period in the app, **not** a
JSON boolean and not an explicit Off command. The Purmo editor displays Comfort
periods over a default background; its full-week path fills gaps with `"false"`.
It is tempting to equate the background to Eco universally, but the complete
server/device mapping has not been established. Explicit `eco` and `booster`
intervals appear in the shared app; availability depends on product/provider.

The app emits leading/trailing gap intervals and a full-day
`{start:"00:00", end:"00:00", value:"false"}` for days without a selected period.
An `end` of `00:00` marks the end of the day in this construction. Split a period
crossing midnight between the two named days. Apart from the end-of-day marker,
the add-period UI rejects end times at or before the start. UI controls use
`stepminute="15"` and calculate a default end 15 minutes after the start.
The server's rounding and validation of other minute values are unknown.

### Complete full-week request

This synthetic example selects Comfort on Monday from 08:15 to 20:45 and leaves
all other periods at the app's default. The object below is shown **before form
encoding**, not as a JSON request body:

```json
{
  "token": "BODY_TOKEN_EXAMPLE",
  "device_id": "DB_DEVICE_EXAMPLE",
  "peremption": 20000,
  "new": "true",
  "lang": "en_GB",
  "program": {
    "Sun": [{"start": "00:00", "end": "00:00", "value": "false"}],
    "Mon": [
      {"start": "00:00", "end": "08:15", "value": "false"},
      {"start": "08:15", "end": "20:45", "value": "comfort"},
      {"start": "20:45", "end": "00:00", "value": "false"}
    ],
    "Tue": [{"start": "00:00", "end": "00:00", "value": "false"}],
    "Wed": [{"start": "00:00", "end": "00:00", "value": "false"}],
    "Thu": [{"start": "00:00", "end": "00:00", "value": "false"}],
    "Fri": [{"start": "00:00", "end": "00:00", "value": "false"}],
    "Sat": [{"start": "00:00", "end": "00:00", "value": "false"}]
  }
}
```

Flatten nested objects and array indices into bracketed names. An exact portion
of the encoded body is:

```text
program%5BMon%5D%5B1%5D%5Bstart%5D=08%3A15&program%5BMon%5D%5B1%5D%5Bend%5D=20%3A45&program%5BMon%5D%5B1%5D%5Bvalue%5D=comfort
```

The request headers and target are:

```http
POST /api/v0.1/human/device/apply_program/ HTTP/1.1
Host: e3.lvi.eu
Authorization: Bearer <access-token>
Content-Type: application/x-www-form-urlencoded
```

For example, this standard-library Python snippet turns the complete object
above (as `request`) into the form body. It performs no network request:

```python
from urllib.parse import urlencode

fields = {key: value for key, value in request.items() if key != "program"}
for day, intervals in request["program"].items():
    for index, interval in enumerate(intervals):
        for key in ("start", "end", "value"):
            fields[f"program[{day}][{index}][{key}]"] = interval[key]
body = urlencode(fields)
```

On a response with `code.key == "OK_SET"`, the app exits the schedule editor.
Unlike the encoded-template path below, **it makes no separate `query/push/`
call after `apply_program/` or `copy_program/`**. These endpoints are the app's
single submission point; internal server queueing is not visible in the source.
A complete success `data` schema is unknown and the caller does not consume it.

### Readback and activation

To retrieve the decoded week, the app posts `device_id`, `now` and, for example,
`height=265` to `human/sandbox/convert_program/`. It expects application code 1.
`data.program[dayIndex]` contains `label` and a `nice` array of intervals with
`start`, `end`, `value`, and rendering fields such as `top` and `height`.
The editor maps indices 0–6 to Sunday–Saturday; its configurable first day of the
week only changes display order. The compact string instead starts on Monday,
as established by a controlled panel edit. The complete response schema is not known.

The compact field is named `programme` in the store/queue requests. Read back
the decoded week to compare the requested boundaries and values, and refresh
the home/device state after submission. Saving a week and
selecting Program mode are separate client operations: the schedule-save request
does not contain `gv_mode` or `nv_mode`. Use the existing [mode command](#submitting-device-commands)
when selecting Program; the app uses device-specific program phases. Server-side
implicit mode changes have not been tested. `OK_SET` and server readback alone
do not prove that the panel or radiator has received the week.

### Already-encoded templates and clearing

For a built-in template the app already has a compact `programme` string. It:

1. Posts it to `human/device/edit/` with the home and local device ID.
2. After `OK_SET`, posts the **same string** as `query[programme]` to
   `human/query/push/`, with `context=1` and `query[id_device]`.
3. Expects `OK_SET` again. A failed second call can leave the stored cloud
   schedule changed without confirmed delivery to the panel.

The clear-program action sends 336 zero characters through that same two-step
path. Reusing a known encoded string is supported by the app; this is distinct
from inventing an encoder for custom intervals.

### Compact encoding

**Observed for Comfort/Eco:** 336 characters, seven Monday–Sunday days of
48 characters each. Each character represents two consecutive quarter-hours.

| Character | First quarter-hour | Second quarter-hour |
| --- | --- | --- |
| `0` | Eco | Eco |
| `1` | Eco | Comfort |
| `4` | Comfort | Eco |
| `5` | Comfort | Comfort |

For these four characters, encode as `4 * first + second`, with Eco = 0 and
Comfort = 1. This is a complete Comfort/Eco subset, not evidence for other
two-bit values or Booster encoding. Reject unsupported values rather than
silently mapping them to Eco.

Character index is `48 * day + 2 * hour + floor(minute / 30)`, where Monday
is day 0. Within the character, minutes 00–14 or 30–44 select the first
quarter; 15–29 or 45–59 select the second. Days use the panel's local schedule
clock; DST transition rules are not established.

A synthetic Monday with Eco only from 08:15 to 08:30 is
`"5" * 16 + "4" + "5" * 31`. Appending `"5" * 288` gives a complete week
with Comfort for the remaining six days. These expressions describe string
construction, not literal values to send. An all-zero week means all Eco,
not Off and not an absent schedule; selecting Comfort mode overrides it.

A compact `query[programme]` push was observed in the next panel poll as
command field `"6"`, followed by a panel upload of the same string.
The panel then executed quarter-hour transitions. This establishes the compact
queue path; it does not independently validate the structured editor endpoints.

The panel executes the weekly schedule and sends phase targets/modes over RF;
there was no whole-week transfer in the observed schedule edits. See
[downstream commands](../clevertouch-upstream/specification.md#downstream-commands)
and [RF scheduling](../radiator-rf/specification.md#schedule-execution-and-command-delivery).

## Delivery and liveness

In one cloud Comfort-mode change, the panel uploaded the new mode before
sending it to the radiator at its next periodic exchange, about 115 seconds
after panel acceptance. This is one observed delay, not a timeout or guaranteed
poll interval. Server readback is not radiator confirmation.


| Path | Request fields | Response data / interpretation |
| --- | --- | --- |
| `human/query/check_failure/` | `token`, `smarthome_id` | App uses `code.key == OK` to increase its next query timing allowance; other outcomes reset it. No per-query success schema is established. |
| `human/sandbox/check_last_connexion/` | `token`, `smarthome_id` | `diff`, plus `diffObj.days`, `.hours`, `.minutes`; app compares `diff` with a 60-second threshold |
| `human/smarthome/time_offset/` | `smarthome_id`, `token` (from the home record in this caller) | `time_offset`, applied to the app's clock in seconds |
| `human/smarthome/get_errors/` | `token`, `smarthomeId`, `type_id=2` for panel errors | App reads array entries with `error_id`, `error`, `comment` on `OK` |

Note the camel-case `smarthomeId` for `get_errors/`. Do not normalise it to the
underscore spelling used by most other endpoints. The app checks failures after
a delay and pauses normal refresh around pending writes; its delays are not wire
requirements. `check_failure/` is not a documented per-command acknowledgement.

## Statistics

`human/stats/read/` takes `token`, `smarthome_id` and optionally the previous
`generated_at`. The app consumes `data.generated_at`, `data.devices` and
`data.stats_by_zone`; per-device `monthly_conso` has `jan` through `dec` values.
It derives yearly totals by summing those months. It refreshes cached statistics
after 600 seconds. The complete schema and measurement provenance are unknown;
these values do not establish that RF duty fields are calibrated energy meters.

## Whole-home mode operations

The app has a separate multi-step holiday workflow. This is distinct from a
single-device `gv_mode` command:

| Operation | Observed client request |
| --- | --- |
| Store dates/state | `human/smarthome/edit/`: `token`, `smarthome_id`, `holiday_start`, `holiday_end`, `holiday_mode` |
| Store per-device-family holiday mode | `human/smarthome/edit_mode/`: `token`, `smarthome_id`, `smarthome_mode_type_id=2`, `bundle_code` (e.g. `C` or `O`), `num_mode` |
| Activate delivery | `human/query/push/`: `context=2`, `query[general_mode]=0`, `query[holiday_mode]=1` |
| Deactivate delivery | Same endpoint: `context=2`, `query[general_mode]=1`, `query[holiday_mode]=0` |

The query requests also include `token`, `smarthome_id` and
`peremption=3600000` in this workflow. Date strings are constructed by the app;
time-zone semantics and the complete activation workflow are not specified here.
These field combinations are a partial reference, not a validated recipe for
replacing all holiday-mode behaviour.

Account creation/deletion, weather, notifications, branding and UI rendering
endpoints are outside this radiator protocol reference.
