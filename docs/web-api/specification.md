# Clevertouch Web API specification

[Overview](../../README.md) · [Shared values](../common.md)

The account API used by the web app and by
[hemphen/clevertouch](https://github.com/hemphen/clevertouch). Evidence combines
**Client** code (the library and the web app, `script.js` 2.8.1) with
**Observed** reads, schedule pushes and mode pushes on one Purmo/LVI
installation. Other providers on the same platform may differ.

## Transport and authentication

| Setting | Purmo/LVI value |
| --- | --- |
| API base | `https://e3.lvi.eu/api/v0.1/` |
| Token endpoint | `https://auth.e3.lvi.eu/realms/purmo/protocol/openid-connect/token` (Keycloak, realm `purmo`) |
| OAuth client | `app-front`, public |
| Requests | `POST`, `application/x-www-form-urlencoded` |
| Authorization | `Authorization: Bearer <access-token>` |
| Responses | JSON |

| Grant | Form fields |
| --- | --- |
| Password | `grant_type=password`, `client_id=app-front`, `username`, `password` |
| Refresh | `grant_type=refresh_token`, `client_id=app-front`, `refresh_token` |

- A refresh returns `access_token`, `refresh_token`, `expires_in` and
  `refresh_expires_in` (**Observed**: 300 s and 7776000 s, 90 days). Use the
  returned values, not constants.
- Every refresh returns a new refresh token; store it. The previous one stayed
  usable for at least two minutes (**Observed**).
- An invalid refresh token returns HTTP 400 `invalid_grant`. The client treats
  400 and 401 from the token endpoint as authentication failure.
- Password-grant availability is provider policy. The web app uses a Keycloak
  login session instead and refreshes 40 s before expiry (**Client**).
- The app adds `lang` (default `en_GB`) and a legacy body field `token`. Requests
  without `token` succeeded for the library's endpoints (**Observed**). Neither
  is a clevertouch-upstream cursor.

## Form encoding and response envelope

Nested objects flatten to bracketed form names, percent-encoded as usual:

```text
query[id_device]=C007-006
query[consigne_confort]=680
```

```json
{"code": {"code": "1", "key": "OK", "value": "<description>"}, "data": {}, "parameters": {}}
```

| `code.code` | `code.key` | Meaning |
| --- | --- | --- |
| `1` | `OK` | Successful read |
| `8` | `OK_SET` | Write accepted; also returned by `user/read/` |
| `2` | `ERR_NO_DATA` | Nothing to report, e.g. `query/check_failure/` with nothing pending |

The library accepts 1 or 8 for reads and only 8 for writes; the app checks
`code.key`. Check HTTP and application status both. The error catalogue is
unknown. A timeout after a write leaves delivery uncertain.

## Core endpoints

Paths are relative to the API base and keep their trailing slash.

| Path | Request fields | Response `data` |
| --- | --- | --- |
| `human/user/read/` | `email` | `user_id`, `smarthomes[]`. The server identifies the account from the token and ignores `email` (**Observed**) |
| `human/smarthome/read/` | `smarthome_id` | Home record, `zones[]`, `devices[]` |
| `human/query/push/` | `smarthome_id`, `context`, `peremption`, `query[...]` | No success payload; check the envelope |

- `smarthome_id` encodes the panel's MAC address in base64 with a suffix
  (**Observed**). Treat it as opaque.
- Room names come from `smarthome/read/` `zones[]` (`num_zone`, `zone_label`).
  The copy in `user/read/` `smarthomes[]` was empty.

Synthetic subset of a home:

```json
{
  "smarthome_id": "HOME_EXAMPLE",
  "label": "Example home",
  "zones": [{"num_zone": "7", "zone_label": "Example room"}],
  "devices": [{
    "id": "DB_DEVICE_EXAMPLE", "id_device": "C007-006", "num_zone": "7",
    "label_interface": "Example radiator", "gv_mode": "0", "nv_mode": "0",
    "temperature_air": "680", "consigne_confort": "698", "heating_up": "1"
  }]
}
```

Use identifiers as opaque values and tolerate extra fields.

### Other endpoints

The web app calls 29 endpoints (**Client**). Those beyond the core:

| Endpoint | Fields | Evidence |
| --- | --- | --- |
| `human/stats/read/` | `smarthome_id`, optional `generated_at` | Observed |
| `human/sandbox/convert_program/` | `device_id`, `now`, `width` or `height` | Observed |
| `human/smarthome/get_errors/` | `smarthomeId` (camel case), `type_id=2` | Observed |
| `human/sandbox/check_last_connexion/` | `smarthome_id` | Observed |
| `human/sandbox/get_current_timestamp/` | none | Observed: server Unix time, UTC |
| `human/query/check_failure/` | `smarthome_id` | Observed |
| `human/device/edit/`, `apply_program/`, `copy_program/` | see [weekly schedules](#weekly-schedules) | Client |
| `human/smarthome/edit/`, `edit_mode/`, `time_offset/` | see [whole-home modes](#whole-home-mode-operations), [liveness](#delivery-and-liveness) | Client |
| `human/smarthome/send_weather/`, `user_manual/`; `human/zone/edit/`; `human/user/edit/`, `auth/`, `twin/`, `unpair/`, `delete/`, `get_lang/`, `forgotten_password/`, `add_pushnotification_token/`; `human/sandbox/get_last_cgu/`, `get_db_version/`; `human/mobile/read_demo/` | | Names only; out of scope |

## Radiator fields

| Field | Meaning |
| --- | --- |
| `gv_mode` | Active mode ([mode table](../common.md#heating-modes)) |
| `nv_mode` | Written with `gv_mode`; equal in every observation; separate meaning unknown |
| `temperature_air` | Measured temperature; read-only |
| `consigne_confort`, `consigne_eco`, `consigne_hg`, `consigne_boost` | Stored Comfort, Eco, Frost and Boost setpoints |
| `consigne_manuel` | Manual setpoint; mirrored by the app during several mode changes |
| `consigne_off` | Used by the app to display Off; not a control field |
| `min_set_point`, `max_set_point` | Setpoint bounds; 5 and 30 °C observed |
| `heating_up` | `"1"` while the element heats; upload field `33`, RF `DF` byte 2 bit 0 |
| `time_boost` | Boost time in seconds; upload field `14`, RF `5B` × 60. Below the configured duration outside Boost it can be a leftover countdown |
| `time_boost_format_chrono` | `{d, h, m, s}`; seconds = `86400d + 3600h + 60m + s` |
| `date_start_boost` | Consumed by the app; semantics unknown |
| `puissance_app` | Rated power, watts; upload field `23` |
| `error_code` | Status bitfield; upload field `34` |
| `temperature_sol` | `2124` (unavailable) on every device; upload field `17` |
| `programme` | Compact weekly schedule ([encoding](#compact-encoding)) |

The app also recognises modes 5 (fan), 6 (fan disabled), 13 (program),
15 (manual) and 16 (program, Boost phase). These are **Client**-only and
provider-dependent, not observed Purmo RF modes. The library writes 11 for
Program; the app picks a phase per device. There is no universal Program write
code.

## Submitting device commands

Send `human/query/push/` with `context=1`, `peremption` and `query[id_device]`,
plus:

| Operation | `query[...]` fields (Client) |
| --- | --- |
| Set a stored temperature | The `consigne_*` field, raw units |
| Select a mode | `gv_mode` and `nv_mode`, same code |
| Configure Boost duration | `time_boost=<seconds>` |
| Start Boost | `gv_mode=4`, `nv_mode=4`, `consigne_boost=<raw>`, `time_boost=<seconds>` |
| Switch a light or outlet | `on_off=1` or `on_off=0` |

`peremption` is in milliseconds: the library sends 15000; the app starts at
20000 and adapts within 15000–30000. Its server-side effect is unknown.

Synthetic example, Comfort at 21 °C:

```http
POST /api/v0.1/human/query/push/ HTTP/1.1
Host: e3.lvi.eu
Authorization: Bearer <access-token>
Content-Type: application/x-www-form-urlencoded

smarthome_id=HOME_EXAMPLE&context=1&peremption=15000&query%5Bid_device%5D=C007-006&query%5Bgv_mode%5D=0&query%5Bnv_mode%5D=0&query%5Bconsigne_confort%5D=698
```

- The app's non-Fenix path also mirrors the target into `consigne_manuel` (zero
  for Off) and may send `time_boost=0` when leaving a timer. These are client
  choices, not requirements. Fenix branches differ; do not apply them to Purmo.
- `OK_SET` means the server queued the command. The panel collects it on its
  next poll ([downstream commands](../clevertouch-upstream/specification.md#downstream-commands)).
- There is no transaction ID or end-to-end acknowledgement. Do not retry a timed
  command blindly after an ambiguous failure.

## Weekly schedules

The app submits an edited week as named-day intervals to
`human/device/apply_program/`; the **server** builds the compact `programme`
string (**Client**). Built-in templates and clearing send a compact string
directly (**Observed** delivery).

### Operations and identifiers

| Operation | Endpoint | Fields besides `lang` |
| --- | --- | --- |
| Save a whole edited week | `human/device/apply_program/` | `token`, `device_id`, `peremption`, `new=true`, `program[...]` |
| Add an interval on selected days | Same | Same, without `new` |
| Copy another device's week | `human/device/copy_program/` | `token`, `device_id_from`, `device_id_to`, `peremption` |
| Store a compact template | `human/device/edit/` | `token`, `smarthome_id`, `id_device`, `programme` |
| Queue that template to the panel | `human/query/push/` | `token`, `smarthome_id`, `context=1`, `peremption`, `query[id_device]`, `query[programme]` |
| Read a decoded week | `human/sandbox/convert_program/` | `device_id`, `now` (Unix s), `height` or `width` (pixels) |

`device_id*` are cloud database IDs (`devices[].id`); `id_device` is the
panel-local ID. Mixing them changes which object the server receives. Merge rules
for omitted `new`, overlaps and existing intervals are unknown; send whole weeks.

### Week and interval schema

| Field | Representation |
| --- | --- |
| `program` | Object with keys `Sun`, `Mon`, `Tue`, `Wed`, `Thu`, `Fri`, `Sat` |
| Each day | Intervals in chronological order |
| `start`, `end` | `HH:MM`, quarter-hour steps in the app; `end` `00:00` is end of day |
| `value` | `comfort`, `eco`, `booster`, or the string `false` (default period, not Off) |
| `new` | `true` for a whole-week save |

- The app fills gaps with `false` and sends `{start:"00:00", end:"00:00", value:"false"}`
  for a day with no selection. Split a period crossing midnight between days.
- Treating the `false` background as Eco everywhere is not established.
- Server rounding of other minute values is unknown.

### Complete full-week request

Synthetic: Comfort on Monday 08:15–20:45, all else default. Shown before form
encoding:

```json
{
  "token": "BODY_TOKEN_EXAMPLE", "device_id": "DB_DEVICE_EXAMPLE",
  "peremption": 20000, "new": "true", "lang": "en_GB",
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

Flattened, part of the body is:

```text
program%5BMon%5D%5B1%5D%5Bstart%5D=08%3A15&program%5BMon%5D%5B1%5D%5Bend%5D=20%3A45&program%5BMon%5D%5B1%5D%5Bvalue%5D=comfort
```

```python
from urllib.parse import urlencode

fields = {k: v for k, v in request.items() if k != "program"}
for day, intervals in request["program"].items():
    for i, interval in enumerate(intervals):
        for key in ("start", "end", "value"):
            fields[f"program[{day}][{i}][{key}]"] = interval[key]
body = urlencode(fields)
```

On `OK_SET` the app closes the editor. It sends **no** `query/push/` after
`apply_program/` or `copy_program/`. The success `data` schema is unknown.

### Readback and activation

- `convert_program/` returns `data.program["0"…"6"]`, **Sunday first**, each with a
  `label` and a `nice` list of `{start, end, value}` plus layout fields. The app
  derives the Program phase from it: 8 inside `comfort`, 16 inside `booster`,
  otherwise 11.
- Saving a week does not select Program; send a [mode command](#submitting-device-commands)
  separately. Server-side implicit mode changes are untested.
- `OK_SET` and readback do not show that the panel or radiator has the week.

### Already-encoded templates and clearing

1. `human/device/edit/` with the compact `programme`.
2. After `OK_SET`, the same string as `query[programme]` to `human/query/push/`.
3. Expect `OK_SET` again. A failure here leaves the cloud copy changed and the
   panel's unchanged.

Clearing sends 336 `0` characters the same way.

### Compact encoding

336 characters: seven days of 48, **Monday first**. Each character covers two
quarter-hours (**Observed** for Comfort/Eco):

| Character | First quarter | Second quarter |
| --- | --- | --- |
| `0` | Eco | Eco |
| `1` | Eco | Comfort |
| `4` | Comfort | Eco |
| `5` | Comfort | Comfort |

```text
character = 4 * first + second            # Eco = 0, Comfort = 1
index     = 48 * day + 2 * hour + minute // 30   # Monday = day 0
quarter   = first if minute % 30 < 15 else second
```

- Reject other characters; do not map them to Eco. Booster encoding is unknown.
- Synthetic Monday with Eco only 08:15–08:30: `"5" * 16 + "4" + "5" * 31`.
- All `0` is all Eco, not Off and not "no schedule"; Comfort mode overrides it.
- Times are the panel's local clock; DST handling is unknown.
- A pushed `query[programme]` arrived as panel command field `"6"`, was uploaded
  back, and ran at quarter-hour boundaries (**Observed**). The panel, not the
  radiator, runs it ([RF](../radiator-rf/specification.md#schedule-execution-and-command-delivery)).

## Delivery and liveness

In the one tested delivery, the panel reported a cloud mode change upstream
before the radiator had it; the radiator received it 115 s later at its next
periodic exchange (**Observed**). Server readback is not radiator confirmation.

| Path | Fields | Response |
| --- | --- | --- |
| `human/query/check_failure/` | `token`, `smarthome_id` | `OK` widens the app's query timing; `ERR_NO_DATA` when nothing is pending. Not a per-command acknowledgement |
| `human/sandbox/check_last_connexion/` | `token`, `smarthome_id` | `diff` (seconds since the panel's last contact) and `diffObj`; the app warns above 60 s |
| `human/smarthome/time_offset/` | `smarthome_id`, `token` | `time_offset`, seconds |
| `human/smarthome/get_errors/` | `token`, `smarthomeId`, `type_id=2` | Entries with `error_id`, `error`, `comment`; e.g. `ERR_RF_` "Rf error", `P_RF` "Rf lost" |

## Statistics

`human/stats/read/` takes `token`, `smarthome_id` and optionally the previous
`generated_at`. `data` holds `generated_at`, `devices` (per `id_device`:
`device_power`, `daily_time_use`, `yesterday_conso`, `weekly_conso`,
`monthly_conso` by month, `yearly_conso`, …), `stats_by_day`, `stats_by_zone`
and `advanced_stats`. Units are not stated, and `daily_time_use` × power does not
match the consumption values (**Observed**): neither is calibrated energy. The
app refreshes after 600 s.

## Whole-home mode operations

Holiday mode is a separate multi-step workflow (**Client**):

| Operation | Request |
| --- | --- |
| Store dates and state | `human/smarthome/edit/`: `token`, `smarthome_id`, `holiday_start`, `holiday_end`, `holiday_mode` |
| Store per-family mode | `human/smarthome/edit_mode/`: `token`, `smarthome_id`, `smarthome_mode_type_id=2`, `bundle_code` (`C`, `O`), `num_mode` |
| Activate | `human/query/push/`: `context=2`, `peremption=3600000`, `query[general_mode]=0`, `query[holiday_mode]=1` |
| Deactivate | Same with `query[general_mode]=1`, `query[holiday_mode]=0` |

Date time-zone semantics and the full activation workflow are unknown.

## Open questions

- The server's validation, error catalogue, rate limits, `context` and
  `peremption` semantics.
- Whether `apply_program/` reaches the panel, and how fast.
- Units of `stats/read/`.
