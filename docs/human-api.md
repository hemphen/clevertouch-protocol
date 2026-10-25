# Human API: app / web ↔ server

[Overview](../README.md) · [Shared values](common.md) · [Additional operations](app-operations.md)

This is the account-facing API implemented by `hemphen/clevertouch`. Core
operations below are **Client** evidence from that library and the supplied web
app. This reference describes those implementations, not a live service test.

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
separate token mechanisms; do not substitute a machine-channel cursor for either.

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
| `programme` | Encoded schedule; see [schedule operations](app-operations.md#schedules) |

For common modes 0, 1, 2, 3, 4, 8 and 11, see the [shared mode table](common.md#heating-modes).
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

After `OK_SET`, re-read the home to check state. The app's local updates and
countdown are optimistic. There is no documented command-specific transaction
ID or end-to-end acknowledgement in this core API subset. Do not blindly retry
a timed command after an ambiguous transport failure: it may already have run.
