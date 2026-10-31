# clevertouch-upstream specification

[Overview](../../README.md) · [Shared values](../common.md) · [Web API](../web-api/specification.md)

The HTTP protocol the Touch E3 panel speaks to the provider server, as observed
on one Purmo/LVI installation. The panel initiates every connection; the server
replies only in HTTP responses. It is not the Web API, and the panel runs no
local HTTP service.

## Transport and request format

| Property | Observed value |
| --- | --- |
| Base | `http://e3.lvi.eu/api/v0.1/` |
| Transport | HTTP/1.1, TCP port 80, no TLS |
| Method | `GET` |
| Parameters | A JSON object, percent-encoded as the last path segment |
| Values | Numeric string keys; mostly string values |
| User-Agent | `Mozilla/5.0 SmartHome` |
| Other headers | `Proxy-Connection: keep-alive`; necessity unknown |
| Authentication | None observed |

```http
GET /api/v0.1/machine/query/check/%7B%221%22%3A%22CURSOR_EXAMPLE%22%7D HTTP/1.1
Host: e3.lvi.eu
User-Agent: Mozilla/5.0 SmartHome
```

- The panel polls every 5–15 s and resolves `e3.lvi.eu` again before most
  connections (**Observed**). A DNS override therefore takes effect within one
  poll.
- The channel does not depend on the account's auth server: it kept working
  through an auth outage that blocked the Web API (**Observed**).

## Messages in each direction

| Request, panel → server | Payload | Response, server → panel |
| --- | --- | --- |
| `machine/device/edit/<JSON>` | One device's state ([fields](#device-upload-fields)) | `200`, `{"code":"8","data":{"100":"<cursor>"}}` |
| `machine/query/check/<JSON>` | `{"1":"<cursor>"}` | Idle: `200`, `{"code":"2","data":{"100":"<cursor>"}}` |
| Same | Same | Command pending: `200`, `code` `"1"`, command in `data["3"]` ([below](#downstream-commands)) |

- A cycle often uploads state, receives a cursor in `data["100"]`, then polls
  with it as field `"1"`. Many connections carry only a poll.
- `code` is a **bare string** here; the Web API nests it as `{code, key, value}`.
  Do not share one envelope parser without handling both.
- Field `"1"` and `data["100"]` behave as opaque cursors. Treat them as private
  session data.

## Device upload fields

Keys are JSON strings, not byte offsets. Each upload reports one device; values
are decimal strings. Temperatures use the [shared scale](../common.md#temperatures).

| Key | Meaning | Evidence |
| --- | --- | --- |
| `1` | Cursor | Unknown semantics |
| `2` | `id_device`, e.g. synthetic `C007-006` | Observed; not a model number |
| `3` | Display label | Observed |
| `4` | Device index | Observed; **not the mode** |
| `6` | Weekly schedule, 336 characters ([encoding](../web-api/specification.md#compact-encoding)) | Observed |
| `7` | Comfort setpoint | Observed |
| `8` | Frost setpoint | Strong |
| `9` | Eco setpoint | Strong |
| `10` | Boost setpoint; Web API `consigne_boost` | Strong: equal in every co-timed read |
| `11` | Manual setpoint; Web API `consigne_manuel` | Strong: equal in every co-timed read |
| `12`, `13` | 5 °C and 30 °C; probably setpoint bounds | Tentative |
| `14` | Boost time, seconds; Web API `time_boost` | Strong: always 60 × RF `5B` |
| `16` | Air temperature; `2124` unavailable | Observed: tracks RF `4B` |
| `17` | Always `2124`; Web API `temperature_sol` | Tentative: an absent second sensor |
| `22` | Active mode ([codes](../common.md#heating-modes)) | Observed: matches RF `8A`/`94` |
| `23` | Rated power, watts; Web API `puissance_app` | Observed: followed two deliberate changes |
| `33` | Element heating now; Web API `heating_up` | Strong: matches RF `DF` byte 2 bit 0 |
| `34` | Status bits; Web API `error_code` | See below |
| `35` | Unknown | Never changed |

`34`: bit `0x2000` appears exactly when field `16` goes unavailable
(**Strong**). `0x1000` appeared minutes earlier on one device (**Tentative**: a
radio error). `0x3000` coincided with Web API errors `ERR_RF_` "Rf error" and
`P_RF` "Rf lost".

Synthetic subset:

```json
{"1": "CURSOR_EXAMPLE", "2": "C007-006", "3": "Example radiator", "4": "7",
 "7": "698", "8": "446", "9": "590", "14": "7200", "16": "680", "17": "2124",
 "22": "0", "23": "1000", "34": "0"}
```

This is not a validated minimal write. Panel count is not uploaded. For a
multi-panel radiator, whether `23` is total or per panel is unknown.

## Receiving and applying state

- Preserve unknown keys, and keep strings, missing fields and unavailable
  temperatures distinct.
- A cached valid reading is not proof of current radio contact. Field `16`
  holding a changing valid value shows the panel is hearing the radiator.
- The panel uploads local changes within 1–7 s (**Observed**). This is
  reporting, not command latency.

## Downstream commands

A queued Web API command arrives in the `query/check` response (**Observed**).
Synthetic mode command:

```json
{"code": "1",
 "data": {"100": "NEXT_CURSOR_EXAMPLE", "2": "1",
          "3": {"2": "C007-006", "22": "0", "15": "0"}}}
```

| Location | Meaning |
| --- | --- |
| `data["100"]` | Next cursor |
| `data["2"]` | `"1"` observed; meaning unknown |
| `data["3"]` | One command, keyed like an upload |
| Command `"2"` | Target `id_device` |
| Command `"6"` | Whole compact schedule (Web API `programme`) |
| Command `"22"`, `"15"` | Mode; both `"0"` for a request with `gv_mode=0`, `nv_mode=0`, so which maps to which is open |

- Observed commands: schedules (`"6"`) and a mode (`"22"`, `"15"`). Other keys are
  not known to be writable. Batching is unknown.
- The panel collects a command at its next poll, seconds after submission, and
  echoes the result in its next `device/edit`.
- The echo shows the panel accepted it, not that the radiator has it
  ([RF delivery](../radiator-rf/specification.md#schedule-execution-and-command-delivery)).
- The panel runs its stored schedule; a schedule edit alone sends nothing to the
  radiators.

## Open questions

- Command IDs, acknowledgement, retries, expiry and errors.
- Boot registration, home association and cursor routing. Without these the
  messages above do not suffice for a replacement server.
- Fields `12`, `13`, `35`; the meaning of `data["2"]`.
