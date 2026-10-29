# Shared values and identifiers

[Clevertouch Web API](web-api/specification.md) · [clevertouch-upstream protocol](clevertouch-upstream/specification.md) · [RF](radiator-rf/specification.md)

## Evidence labels

| Label | Meaning |
| --- | --- |
| **Observed** | Present in validated traffic, or verified by a deliberate device action. Limited to the equipment examined. |
| **Client** | Implemented or consumed by the Python client or supplied web app; not necessarily independently observed on the wire. |
| **Strong** | Consistent evidence supports the interpretation, but no decisive controlled test. |
| **Tentative** | Plausible interpretation with unresolved alternatives. |
| **Unknown** | Bytes or fields are known; meaning or behaviour is not. |

Unlabelled encoding and layout rules describe the observed format. They are not
claims about every firmware version. Unknown fields should be preserved, and
unrecognised modes should remain visible as raw values.

## Identifiers

| Identifier | Scope and use |
| --- | --- |
| `user_id` | Cloud account identifier returned by `human/user/read/`. |
| `smarthome_id` | Cloud home identifier; used by home reads and command submission. |
| `id_device` | Panel-local device identifier, e.g. synthetic `C007-006`; used inside `query[...]`. The Python client recognises prefixes `C` radiator, `E` light, `O` outlet. |
| `id` / request parameter `device_id` | Cloud device database identifier. The app uses it for schedule operations. **Not interchangeable with `id_device`.** |
| `num_zone` | Panel-local zone number; join to `zones[].num_zone`. |
| Controller upload field `"2"` | Local device identifier in the `Cnnn-nnn` form. It is not a model number or RF address. |
| Controller upload field `"4"` | Device index; observed to match the first numeric group in field `"2"`. It is not the heating mode. |
| RF address | Four on-wire bytes. The first appears to select a class/family; the last three identify a unit. See [RF addressing](radiator-rf/specification.md#addressing). |

No general algorithm converting a cloud/local device ID to an RF address is
known. Matching changing temperature and mode across the two links can establish
a correspondence for an installation; it does not establish an address formula.

## Temperatures

The named Clevertouch Web API temperature fields, mapped controller upload fields and RF temperature
words use **tenths of a degree Fahrenheit**, irrespective of the UI display unit:

```text
C = (raw - 320) / 18
F = raw / 10
raw = round(18 * C + 320)
```

| Raw decimal | Raw hex | Interpretation |
| ---: | --- | --- |
| 320 | `0140` | 0 °C; observed RF target while Off |
| 680 | `02A8` | 20 °C |
| 689 | `02B1` | 20.5 °C |
| 698 | `02BA` | 21 °C |
| 2124 | `084C` | Unavailable temperature; do not display as 100.22 °C |

RF words are unsigned big-endian. HTTP numeric values commonly arrive as decimal
strings. Preserve unavailable values separately from valid zero temperatures.
The app's rounding and special handling of zero are UI implementation details,
not a different temperature scale. Its `param_c_f` selects `c` or `f` for display.
Do not apply this temperature formula to every numeric field: controller upload `"23"`,
for example, is a configured wattage.

## Heating modes

The following values agree between the app client's `gv_mode`, controller upload `"22"`
and RF `8A`/`94` **for the observed radiator profile**. This does not make every
mode in the wider web app valid on every radiator.

Each code has one name pair, `heat_mode` and `temp_type`, both lowercase. They
are the `HeatMode` and `TempType` enums of the
[hemphen/clevertouch](https://github.com/hemphen/clevertouch) client, which
spells `HeatMode` values capitalised (`Comfort`). Implementations report them
next to the code; `temp_type` names the stored setpoint the mode makes active.

| Code | `heat_mode` | `temp_type` | Active temperature | RF evidence |
| ---: | --- | --- | --- | --- |
| 0 | `comfort` | `comfort` | Comfort setpoint | Observed |
| 1 | `off` | `none` | RF target raw 320 | Observed |
| 2 | `frost` | `frost` | Frost setpoint | Observed |
| 3 | `eco` | `eco` | Eco setpoint | Observed |
| 4 | `boost` | `boost` | Boost setpoint, for a timed period | Observed |
| 8 | `program` | `comfort` | Comfort setpoint | Observed |
| 11 | `program` | `eco` | Eco setpoint | Observed |

Program runs the weekly schedule; the panel calls it Auto. Codes 8 and 11 are
its Comfort and Eco phases in the same namespace as the other modes: an active
mode moves 0 → 11 → 8 → 0 identically on all three links (**Observed**).

Boost selects a timed setpoint; it need not increase heating. Switching modes
can restore a stored setpoint. The RF active target is not a permanent Comfort
register. Additional **Client-only** modes are listed in the Clevertouch Web API reference.

## Completion is layer-specific

A successful Clevertouch Web API write means the server accepted the request. An HTTP
response to a controller upload means the server answered the panel. Valid RF CRCs
mean a frame is intact. None of these alone proves that a radiator applied a
command. Read back device state; a subsequent radiator report provides stronger
confirmation than an optimistic app update or an RF `ack`.
