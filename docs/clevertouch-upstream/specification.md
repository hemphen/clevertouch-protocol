# clevertouch-upstream protocol specification

[Overview](../../README.md) · [Shared values](../common.md) · [Clevertouch Web API](../web-api/specification.md)

This is the panel-facing protocol observed on Purmo/LVI Touch E3. The panel
initiates the connection; server-to-panel information travels in HTTP responses.
It is not the Clevertouch Web API and is not evidence of a local HTTP service on the panel.

## Transport and request format

| Property | Observed value |
| --- | --- |
| Base | `http://e3.lvi.eu/api/v0.1/` |
| Transport | HTTP/1.1 over TCP port 80, without TLS in the observed profile |
| Method | `GET` |
| Parameters | JSON object encoded as the final URL path segment |
| Value types | Numeric JSON keys; mostly string values |
| User-Agent | `Mozilla/5.0 SmartHome` |
| Connection header | `Proxy-Connection: keep-alive` was present; necessity unknown |
| Authentication | No bearer header observed; controller identity/session binding remains incompletely understood |

Percent-encode the JSON as a path segment, not as form data or a `?query` string.
A schematic poll is:

```http
GET /api/v0.1/machine/query/check/%7B%221%22%3A%22CURSOR_EXAMPLE%22%7D HTTP/1.1
Host: e3.lvi.eu
User-Agent: Mozilla/5.0 SmartHome
```

**Observed:** the panel polls every 5–15 seconds and re-resolves the hostname
on most connection cycles; neither is a known requirement.

## Messages in each direction

| Request, panel → server | Payload | Response, server → panel |
| --- | --- | --- |
| `machine/device/edit/<JSON>` | One device's state snapshot, with numeric fields below | HTTP 200, `{"code":"8","data":{"100":"<next-cursor>"}}` |
| `machine/query/check/<JSON>` | `{"1":"<cursor>"}` | Idle: HTTP 200, `{"code":"2","data":{"100":"<next-cursor>"}}` |
| Same check, command pending | Same request family | HTTP 200, `code="1"`, command in `data["3"]`; see below |

A common cycle uploads state, receives a cursor in `data["100"]`, then polls
using that cursor as request field `"1"`. Other connections contain only a poll.
A state upload is therefore not required on every check. Connections commonly
close after the poll response; the observed cadence alone does not prove a
particular long-poll timeout contract.

The controller-facing response `code` is a **bare string**. Clevertouch Web API responses use a
nested `{code,key,value}` object. Do not share an envelope parser without
handling this difference.

Field `"1"` and response field `"100"` behave like opaque cursors. Limited idle
checks accepted missing/fabricated cursor input, but this does **not** establish
the authentication or routing requirements for real command delivery, state
writes, device registration or other controller-facing endpoints. Preserve returned
values and treat them as private session material; no cursor-generation or
home-binding algorithm is established.

## Device upload fields

Field numbers below are JSON string keys, not byte offsets. The payload reports
one device, rather than an entire home. Known numbers use decimal string values.

| Key | Meaning | Evidence / qualification |
| --- | --- | --- |
| `1` | Opaque cursor/session value | Unknown exact semantics |
| `2` | Local device identifier, e.g. synthetic `C007-006` | Observed; not a model number |
| `3` | Display label | Observed |
| `4` | Device index | Observed; **not mode** |
| `6` | Compact weekly schedule | 336 characters; Monday first; two quarter-hours per character; [Comfort/Eco encoding](../web-api/specification.md#compact-encoding) |
| `7` | Comfort setpoint | Observed deliberate setpoint change |
| `8` | Frost setpoint | Strong |
| `9` | Eco/reduced setpoint | Strong |
| `10`, `11` | Stored temperature values | One corresponds to Boost; their individual mapping is unresolved |
| `12`, `13` | Temperature-like values, possibly lower/upper bounds | Tentative; values corresponding to 5 °C and 30 °C observed |
| `14` | Boost time in seconds: the next Boost's duration, or the time left during Boost | Strong; counts down during Boost; always 60 × RF `5B` |
| `16` | Measured air temperature | Observed correlation with RF `4B`; `2124` means unavailable |
| `17` | Candidate second temperature channel | Tentative; observed as `2124`, unavailable |
| `22` | Active mode | Observed agreement with RF `8A`/`94` for common modes |
| `23` | Configured rated power, watts | Observed deliberate wattage changes; not live electrical power |
| `33` | Element heating at this moment | Strong; agrees with RF `DF` byte 2 bit 0 |
| `34` | Status bitfield | Strong: bit `0x2000` correlates with unavailable temperature; meaning of `0x1000` unresolved |
| `35` | Unknown field | No meaning established |

A synthetic subset of a decoded upload is:

```json
{
  "1": "CURSOR_EXAMPLE",
  "2": "C007-006",
  "3": "Example radiator",
  "4": "7",
  "7": "698",
  "8": "446",
  "9": "590",
  "14": "7200",
  "16": "680",
  "17": "2124",
  "22": "0",
  "23": "1000",
  "34": "0"
}
```

This omits unmapped fields and the schedule for readability; it is **not** a
validated minimal accepted write. Use [temperature conversion](../common.md#temperatures)
only for the mapped temperature fields. Panel count was not identified in the
observed upload; a settings change producing no visible field change does not
prove it is never transferred elsewhere. For a two-panel radiator, whether
field `23` is total or per-panel power is unresolved.

## Receiving and applying state

For a receiver of panel uploads, preserve unknown keys and the distinction
between strings, missing fields and unavailable temperatures. A valid reading
may be cached; it is not by itself proof of current radio liveness. Bit `0x2000`
of field `34` is useful supporting evidence for an unavailable reading, not a
complete documented device-health model.

The panel reports local changes upward within seconds (1–7 seconds observed).
That is **panel → server reporting**, not command latency.

## Downstream commands

**Observed:** a queued Web API command is returned by `machine/query/check/`
in this envelope. This is a synthetic mode-change example:

```json
{
  "code": "1",
  "data": {
    "100": "NEXT_CURSOR_EXAMPLE",
    "2": "1",
    "3": {"2": "C007-006", "22": "0", "15": "0"}
  }
}
```

| Location | Meaning |
| --- | --- |
| `data["100"]` | Next opaque cursor |
| `data["2"]` | Observed string `"1"`; count/type meaning unresolved |
| `data["3"]` | Single observed command object |
| Command `"2"` | Panel-local destination device ID |
| Command `"6"` | Complete compact schedule, corresponding to Web API `programme` |
| Command `"22"`, `"15"` | Both `"0"` in an observed Comfort command sent with `gv_mode=0,nv_mode=0` |

**Observed:** schedule commands (`"6"`) and a mode command (`"22"`, `"15"`). The
mode command carried `"0"` in both fields for a request sent with
`gv_mode=0,nv_mode=0`, so which Web API field maps to which key is open; `"22"`
is the active mode in uploads. Other upload keys are not known to be writable,
and batching of several commands is unknown.

For a schedule command, replace the mode fields with `"6": "<336 characters>"`;
the placeholder is not a valid schedule. The panel collects a queued command
at its next poll, seconds after Web API submission, and uploads the accepted
schedule with `device/edit` seconds later.

An upload echo shows that the panel accepted a command, not that a radiator has
it: in the one tested delivery, a mode command reached the radiator at its next
periodic exchange
([RF delivery](../radiator-rf/specification.md#schedule-execution-and-command-delivery)).
Command IDs, acknowledgements, retries, expiry and error responses are unknown.

The panel runs its stored schedule and sends the active phase target/mode to the
radiator. Schedule edits alone need not emit RF state changes. See the
[compact schedule](../web-api/specification.md#compact-encoding) and
[RF execution](../radiator-rf/specification.md#schedule-execution-and-command-delivery).

Boot registration, home association and cursor routing are not covered, so
these messages are not enough for a replacement server.
