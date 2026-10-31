# Shared values

[Web API](web-api/specification.md) · [clevertouch-upstream](clevertouch-upstream/specification.md) · [radiator-rf](radiator-rf/specification.md)

## Evidence labels

Every claim that is not plain layout carries one of these labels.

| Label | Meaning |
| --- | --- |
| **Observed** | Seen in recorded traffic, or caused by a deliberate action. Limited to the equipment examined. |
| **Client** | Implemented or consumed by the Python client or the web app; not necessarily seen on the wire. |
| **Strong** | Consistent evidence supports it; no decisive controlled test. |
| **Tentative** | Plausible, with unresolved alternatives. |
| **Unknown** | The bytes or fields are known; their meaning is not. |

Unlabelled encoding and layout rules describe the observed format, not every
firmware version. Preserve unknown fields and keep unrecognised modes visible as
raw values.

## Identifiers

| Identifier | Scope and use |
| --- | --- |
| `user_id` | Cloud account, returned by `human/user/read/` |
| `smarthome_id` | Cloud home; used by home reads and commands |
| `id_device` | Panel-local device ID, e.g. synthetic `C007-006`; used in `query[...]`. Prefix `C` radiator, `E` light, `O` outlet (Client) |
| `id` / `device_id` | Cloud database device ID; used by schedule operations. **Not interchangeable with `id_device`** |
| `num_zone` | Panel-local zone number; joins `zones[].num_zone` |
| Upload field `"2"` | `id_device`, in the `Cnnn-nnn` form. Not a model number or RF address |
| Upload field `"4"` | Device index; equals the first numeric group of field `"2"`. Not the mode |
| RF address | Four bytes: family byte plus three-byte identity; see [addressing](radiator-rf/specification.md#addressing) |

No formula converts a cloud or local device ID to an RF address. Matching
changing temperature and mode across two links establishes the correspondence
for one installation.

## Temperatures

Named Web API temperature fields, mapped upload fields and RF temperature words
are **tenths of a degree Fahrenheit**, whatever the display unit:

```text
F   = raw / 10
C   = (raw - 320) / 18
raw = round(18 * C + 320)
```

| Raw | Hex | Meaning |
| ---: | --- | --- |
| 320 | `0140` | 0 °C; RF target while Off |
| 680 | `02A8` | 20 °C |
| 689 | `02B1` | 20.5 °C |
| 698 | `02BA` | 21 °C |
| 2124 | `084C` | Unavailable; not 100.22 °C |

- RF words are unsigned big-endian; HTTP values usually arrive as decimal strings.
- °F is exact; °C rarely terminates (raw 637 is 17.611… °C).
- Setpoints move in 0.5 °C steps of 9 raw units.
- Keep unavailable separate from a valid zero.
- The formula does not apply to every numeric field: upload field `"23"` is watts.
- The app's `param_c_f` (`c` or `f`) selects only the display unit.

## Heating modes

One code namespace is shared by the Web API's `gv_mode`, upload field `"22"` and
the RF mode byte of `8A`/`94`, **for the observed radiator profile**. Names are
the `HeatMode` and `TempType` enums of
[hemphen/clevertouch](https://github.com/hemphen/clevertouch), lowercase.
`temp_type` names the stored setpoint the mode makes active.

| Code | `heat_mode` | `temp_type` | Active target | RF |
| ---: | --- | --- | --- | --- |
| 0 | `comfort` | `comfort` | Comfort setpoint | Observed |
| 1 | `off` | `none` | raw 320 | Observed |
| 2 | `frost` | `frost` | Frost setpoint | Observed |
| 3 | `eco` | `eco` | Eco setpoint | Observed |
| 4 | `boost` | `boost` | Boost setpoint, for a timed period | Observed |
| 8 | `program` | `comfort` | Comfort setpoint | Observed |
| 11 | `program` | `eco` | Eco setpoint | Observed |

- Program runs the weekly schedule; the panel calls it Auto. Codes 8 and 11 are
  its phases. An active mode moved 0 → 11 → 8 → 0 identically on all three links.
- Boost selects a timed setpoint; it need not increase heating.
- Further **Client**-only modes are listed in the
  [Web API](web-api/specification.md#radiator-fields).

## Completion is layer-specific

| Event | Shows |
| --- | --- |
| Web API `OK_SET` | The server accepted the request |
| HTTP response to a panel upload | The server answered the panel |
| Valid RF CRCs | The frame is intact |
| RF `ack` | An exchange closed |
| Later `radiator_report` with the new values | The radiator applied it |

Only the last confirms a command.
