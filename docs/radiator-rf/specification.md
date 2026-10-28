# radiator-rf protocol specification

[Overview](../../README.md) · [Shared values](../common.md) · [clevertouch-upstream protocol](../clevertouch-upstream/specification.md)

This describes the observed Purmo/LVI Touch E3 profile of a Watts-family
protocol. Frame layouts and CRCs are established; some field meanings and the
requirements for successful custom transmission remain unresolved. Message
names such as `controller_state` are descriptive names used here, not on-wire
strings or vendor terminology.

## Physical layer

| Parameter | Value |
| --- | --- |
| Centre frequency | 868.3 MHz |
| Modulation / coding | 2-FSK / PCM |
| Symbol duration | 27 µs, about 37 kbit/s |
| Deviation | About ±17 kHz (16–18 kHz measured) |
| Preamble and sync | Alternating-bit preamble (`AA` bytes), then the sync word `D3 91` twice |
| Receiver gap threshold | 1000 µs |

27 µs is the receiver symbol setting that recovers the most valid frames, not
a hardware baud-rate measurement or a CC1101 register profile. Receivers match
`AA D3 91 D3 91`; the preamble's full length is unknown. Post-sync data is
represented bytewise below, most significant bit first, and validates under
both CRCs without whitening. The gap threshold is a receiver setting. Transmit
timing tolerances, frequency tolerance and channel-access rules are unknown.

## Frame envelope

Offsets are zero-based from the length byte **after synchronisation**. Hex pairs
represent bytes in transmission order. Let `L = frame[0]`.

| Offset | Bytes | Meaning |
| --- | ---: | --- |
| `0` | 1 | `L`, body length excluding itself and the outer CRC |
| `1..4` | 4 | Source address |
| `5` | 1 | Marker `C6` |
| `6..9` | 4 | Destination address |
| `10..L-2` | `L-11` | Application payload |
| `L-1..L` | 2 | Inner CRC-16/MODBUS, **little-endian** |
| `L+1..L+2` | 2 | Outer CRC-16/CMS, **big-endian** |

Total frame size is `L + 3` bytes; minimum envelope size is 14 (`L >= 11`).
A receiver validates the envelope separately from the application grammar.
A CRC-valid pairing offer is not a malformed ordinary report.

### Checksums

| Property | Inner | Outer |
| --- | --- | --- |
| Algorithm | CRC-16/MODBUS | CRC-16/CMS |
| Polynomial | `0x8005`, reflected implementation `0xA001` | `0x8005` |
| Initial register | `0xFFFF` | `0xFFFF` |
| RefIn / RefOut | true / true | false / false |
| XorOut | `0x0000` | `0x0000` |
| Covered bytes, inclusive | `1..L-2` | `0..L` |
| Stored order | Low byte first | High byte first |
| Check on ASCII `123456789` | `0x4B37` | `0xAEE7` |

Build the header and payload, set `L`, calculate and append the inner CRC, then
calculate and append the outer CRC. Both checks are required for an intact frame;
they provide error detection, not sender authentication. A capture may contain
extra receiver bits after the frame; they are not part of either CRC range.
Do not silently present repaired data as independently validated traffic.

### Addressing

Addresses occupy four bytes, shown as eight hex digits. During inclusion the
controller changes its first source byte with the destination family (`F1` or
`D0`) while keeping its last three bytes fixed. This **strongly suggests** a
class/family byte plus a three-byte identity; the complete address allocation
scheme is unknown. Normal observed radiator traffic uses the `F1` family.

A radiator retained its address after deletion and re-inclusion. That supports a
persistent device identity, not proof of factory assignment on every model.
No installation's actual addresses are included here. The [examples below](#synthetic-frame-examples)
use invented addresses and are not valid discovery targets for real equipment.

## Ordinary traffic

Ordinary traffic has four fixed shapes: every observed frame of each type had
exactly this length and record order.
Lengths include both CRCs in the total column. Tags are listed in on-wire order.

| Message | Direction | `L` | Total | Record order |
| --- | --- | ---: | ---: | --- |
| `radiator_report` | Radiator → controller | `23` hex / 35 | 38 | `DF 4B 8A E1 DA 0A` |
| `controller_state` | Controller → radiator | `38` hex / 56 | 59 | `45 4B 3B 4C 8E CC 97 94 E1 DA 5B B2` |
| `ack_candidate` | Radiator → controller | `0F` hex / 15 | 18 | `01 0A` |
| `request_candidate` | Controller → radiator | `0D` hex / 13 | 16 | `19` |

### Message types

A frame's type follows from its payload's shape, never from its addresses or
its length alone. The first matching rule applies:

| Payload | Message | Source |
| --- | --- | --- |
| Not a record stream, beginning `00 00` | `inclusion_offer` | Controller; the destination is a discovery candidate |
| Not a record stream otherwise | `unknown`, bytes preserved | Not inferred |
| Records including `DF` | `radiator_report` | Radiator |
| Records including `45` | `controller_state` | Controller |
| A single `01` record | `inclusion_bind` | Controller |
| Records beginning `01` | `ack_candidate` | Radiator |
| Records beginning `19` | `request_candidate` | Controller |
| All zero | `inclusion_reply` | Radiator |
| Anything else | `unknown` | Not inferred |

The other address is the counterpart: the radiator of a controller frame and
the controller of a radiator frame. "Not a record stream" means the
[record grammar](#record-grammar) fails: a record overruns the payload, or a
nonzero byte follows tag `00`. Such a frame is kept with its bytes when both
CRCs pass.

`radiator_report` supplies measured temperature and current state.
`controller_state` carries a target/mode and controller-side state; panel-driven
changes were observed in this direction. Its reported temperature is an echo,
not an independent sensor measurement. Duty may be computed by the controller,
but sparse radiator reporting prevents a firm conclusion about where the
control loop runs.

The short frames contain `01=00, 0A=47` and `19=00`, respectively. Their names
are **tentative**: no sequence number, result code or acknowledged-command ID
has been identified in them. Do not infer success from their presence alone.

### Exchanges

**Observed:** traffic comes in short exchanges between the controller and one
radiator, of two kinds.

| Exchange | Opened by | Frames | When |
| --- | --- | --- | --- |
| Periodic | Radiator | `radiator_report`, `controller_state`, `ack_candidate`, `radiator_report` | On a per-radiator cycle of about 2.5–4 minutes |
| Controller-initiated | Controller | `request_candidate`, `radiator_report`, `controller_state`, `ack_candidate`, repeated two or three times | When the radiator's menu is opened or its mode changed at the panel, and at schedule phase boundaries |

`controller_state` is part of every periodic exchange; between changes it
repeats the radiator's own values back with the controller's clock. A radiator answered `request_candidate` within
0.35 seconds, and the controller sent its next `request_candidate` 0.2–0.7
seconds after a `controller_state`. A `request_candidate` was not always
followed by a received report, and the controller then sent `controller_state`
regardless. In a periodic exchange the first report carries the interval's
`DA` and the second reports zero. Retry rules are unknown.

### Record grammar

An ordinary payload is a sequence of `[tag][value]` records:

```text
value_length = (tag >> 6) + 1
```

Tags `01..3F` have one value byte, `40..7F` two, `80..BF` three, and `C0..FF`
four. Values must fit wholly before the inner CRC. Unknown nonzero tags can be
skipped by this length rule while retaining their raw bytes.

Tag `00` is treated as a terminator with only zero padding allowed after it in
the ordinary-record grammar. The four ordinary shapes fill their payloads
exactly, without a terminator. **Do not apply this grammar indiscriminately to
inclusion offers**, whose payload begins `00 00` but continues with nonzero data.
Do not infer direction from tag `01` alone: it also appears in controller-to-
radiator inclusion binding.

## Record reference

Multi-byte numeric words below are big-endian unless explicitly stated.

| Tag | Value bytes | Message | Meaning / observed value | Evidence |
| --- | ---: | --- | --- | --- |
| `DF` | 4 | Report | Opaque radiator state/configuration word | Unknown |
| `4B` | 2 | Report, state | Air temperature, uint16 | Strong; agrees with controller upload field `16` |
| `8A` | 3 | Report | `[target:uint16][mode:uint8]`, active target and mode | Observed |
| `94` | 3 | State | Same layout, controller-side target and mode | Observed with panel changes |
| `E1` | 4 | Report, state | `[00][front maximum %][channel 1 duty?][channel 2 duty?]` | Maximum confirmed; duty scaling tentative |
| `DA` | 4 | Report, state | Two uint16 candidate interval on-times in seconds | Tentative; zero in every observed controller state |
| `B2` | 3 | State | Binary hour, minute, second | Strong; time zone/date unknown |
| `5B` | 2 | State | Slowly varying per-device uint16 | Unknown; not identified as watts or energy |
| `97` | 3 | State | Opaque controller state/configuration word | Unknown |
| `0A` | 1 | Report, ack | `47` | Unknown meaning |
| `01` | 1 | Ack; also inclusion bind | `00` | Unknown meaning beyond observed exchange position |
| `19` | 1 | Request | `00` | Unknown meaning |
| `45` | 2 | State | `00 00` | Unknown |
| `3B` | 1 | State | `00` | Unknown |
| `4C` | 2 | State | `00 00` | Unknown |
| `8E` | 3 | State | `00 00 84` | Unknown; do not import floor-thermostat semantics |
| `CC` | 4 | State | `00 00 00 00` | Unknown |

Constant values are the observed ones, not guaranteed across models.
Tag `5E` is implemented as a second temperature in related Watts decoders but
was not observed on this equipment. It is not part of the established inventory.

### Temperature and mode (`4B`, `8A`, `94`)

Use `(raw - 320) / 18` °C; `084C` means unavailable. Mode is the final byte of
`8A`/`94`, with values in the [common mode table](../common.md#heating-modes).
Examples: `8A 02 A8 00` reports Comfort at 20 °C; `94 01 40 01` carries Off
with a 0 °C target. Changing to Eco changes the active target without proving
that the stored Comfort setting has been overwritten.

### Power-related fields (`E1`, `DA`)

`E1` byte indices are zero-based within the four-byte value:

| Byte | Interpretation |
| ---: | --- |
| 0 | Observed zero; meaning unknown |
| 1 | Configured maximum power rate of the front panel, percent |
| 2 | Candidate channel 1 duty, front panel |
| 3 | Candidate channel 2 duty |

Selecting a 55% front-panel limit changed byte 1 from 100 to 55 and capped
channel 1 at 55 while channel 2 remained at 100. This establishes independent
channels and the front-panel association. Duty values also respond to heating
demand, but electrical output has not been calibrated against them. If a duty
byte exceeds 100, retain it as raw data rather than clamping it into a percentage.

`DA` contains two candidate per-channel interval counters. Values return to zero
within exchanges and are not monotonic lifetime totals. A recurring sequence is
report with nonzero `DA` → controller state with zero `DA` → ack-shaped frame →
report with zero `DA`. The precise reset trigger and the mapping of `DA` words
to physical panels are unknown. Never sum repeated reports as independent energy.

If the duty/on-time interpretation and channel ratings were independently
verified, estimates would be:

```text
power_W = (P1_W * duty1_pct + P2_W * duty2_pct) / 100
interval_Wh = (P1_W * on_time1_s + P2_W * on_time2_s) / 3600
```

These are conditional estimates, not measured consumption. Controller upload field `23`
gives a configured rating, but its division across panels is unresolved.

### Opaque state and clock fields

`DF` values include `00/20` in byte 0, `30/31` in byte 1, `00/01` in byte 2,
and `40` in byte 3. The `30→31` change persisted after heating demand returned
to zero; it must not be labelled a simple heating-active bit. In `97`, byte 0
equals byte 0 of the radiator's preceding `DF` (**Observed**, every recorded
controller frame), byte 1 is `30`, and byte 2 is `04` or `05` without a known
rule. No bit-level meaning is established.

`5B` varies both between devices and slightly over time. Changing a configured
wattage did not change it. `B2` is consistent with a binary clock: require
hour < 24 and minute/second < 60 before displaying it; do not infer a date,
time zone or synchronisation requirement.

## Inclusion / pairing

Observed with an already-used radiator placed in its `rF.bi` inclusion mode and
removed from the controller's device list. A factory-fresh device and a custom
transmitter have not been tested. The four-byte addressing and dual-CRC envelope
remain the same; the payload grammar differs.

| Step | Message | Direction | `L` | Payload only | Interpretation |
| --- | --- | --- | ---: | --- | --- |
| Offer | `inclusion_offer` | Controller → candidate address | 15 | `00 00 03 01` | Opaque fixed-layout payload, outside the record grammar |
| Reply | `inclusion_reply` | Radiator → controller | 13 | `00 00` | Empty ordinary record stream / padding |
| Bind | `inclusion_bind` | Controller → replying radiator | 13 | `01 00` | Single tag `01` with value zero |
| Ack | `ack_candidate` | Radiator → controller | 15 | `01 00 0A 47` | Same shape as the ordinary acknowledgement |

The controller swept candidate destinations `F1 10 00 01`, `D0 00 00 02`,
`F1 10 00 03`. These are observed discovery values, **not installed device IDs**.
It changed its source family byte to match each candidate family, then used the
radiator's returned address for binding. The full candidate enumeration range,
retry policy and stopping rules are unknown.

One refused/already-installed attempt used offer payload `00 00 03 64` instead
of `00 00 03 01`. The final byte's meaning is unknown; it is not established as
a mode, count or error code. Replies were seen only after offers.

The exchange takes about three seconds, with roughly one-second gaps. Repeated
inclusion produced identical bytes, with no visible challenge or counter;
whether a replayed exchange or a foreign controller identity is accepted is
untested.

Re-inclusion reset the tested radiator's stored values to Comfort 19 °C,
Frost 7 °C, Eco 15.5 °C, both unresolved controller upload temperature fields `10`/`11`
to 25 °C, and timer to 7200 seconds. These are **observed reset values for that
unit**, not universal factory defaults. Panel count and wattage settings are
separate from the four-frame exchange; their RF encoding is not established.

## Schedule execution and command delivery

**Observed:** the Touch E3 executes the weekly Comfort/Eco schedule. Editing a
schedule, at the panel or through the cloud, sends nothing over RF. At each
phase boundary the controller opens an exchange and sends `controller_state`
whose `94` carries the phase's target and mode: 8 (Program, comfort) or 11
(Program, eco). It does so 2–4 seconds after the quarter-hour by its `B2`
clock. The radiator receives only the resulting target and mode, so whatever
replaces the controller must run the schedule itself. No RF schedule format is
known.

A mode change requested through the cloud reaches the radiator at its next
periodic exchange, not at once (**Observed**: 115 seconds after the panel
accepted it, in one delivery). The panel reports the new mode to the server
before that, so neither server readback nor a panel upload shows that the
radiator has it.

Which device computes heating duty, and what a radiator in Program does without
its controller, are unknown.

## Sending and receiving boundaries

A receiver validates both CRCs, recognises the message types above and decodes
the established fields. It keeps unfamiliar but CRC-valid payloads as unknown
messages rather than discarding them as noise.

### Constructing controller state

No rule produces the opaque fields (`45`, `3B`, `4C`, `8E`, `CC`, `97` bytes 1–2,
`5B`) for a given radiator, so a `controller_state` is built from one the
controller sent to that radiator earlier. **Observed** in every recorded
controller frame:

| Record | Content |
| --- | --- |
| `4B`, `E1` | Copied from the radiator's report earlier in the same exchange (119 of 119 such exchanges) |
| `94` | The target and mode; Off carries raw 320 |
| `DA` | Zero (502 of 502) |
| `97` byte 0 | Byte 0 of that report's `DF` (502 of 502) |
| `B2` | The controller's clock |
| Others | Unchanged from the earlier frame |

Built this way from the controller's previous frame to the same radiator, 107
of 128 recorded controller frames are reproduced exactly; the rest differ only
in `97` byte 2 (`04` against `05`).

A transmitter can build every message from this reference, but control by
anything other than the radiator's own controller is **not validated**:
acceptance of another controller identity, timing tolerance and behaviour
after controller loss remain open. No RF schedule format is known. Confirm a
command by a later radiator report carrying the new mode and target; neither a
valid CRC nor an acknowledgement-shaped frame shows success. A report can show
the target one raw unit (1/18 °C) below the value sent: 11 times, against 239
exact matches.

## Synthetic frame examples

The following examples use invented controller `F1010203` and radiator
`F1040506` addresses. They were generated from the documented layouts and checked
with both CRC algorithms. They test parsing/encoding only, not radio acceptance.
Discovery destination `F1100001` is a protocol candidate value, not a private ID.

[vectors.json](vectors.json) holds these and further cases as test data: each
frame with its payload, records, roles and decoded values in the terms of this
specification, and frames a receiver must reject, with the reason. They were
cross-checked with a second, independent decoder.

| Message | Complete frame, starting at length | Bytes |
| --- | --- | ---: |
| Radiator report | `23F1040506C6F1010203DF203000404B02A88A02BA00E100641919DA000A000A0A47CD48C59D` | 38 |
| Controller state | `38F1010203C6F10405064500004B02A83B004C00008E000084CC00000000972030049402BA00E100641919DA000000005B0078B20C0000A20F60D9` | 59 |
| Request candidate | `0DF1010203C6F10405061900A01B4377` | 16 |
| Ack candidate | `0FF1040506C6F101020301000A47E6E64D5A` | 18 |
| Inclusion offer | `0FF1010203C6F1100001000003011FF3EA1F` | 18 |
| Inclusion reply | `0DF1040506C6F1010203000080E1D2A1` | 16 |
| Inclusion bind | `0DF1010203C6F10405060100AA1B1F72` | 16 |
| Inclusion ack | `0FF1040506C6F101020301000A47E6E64D5A` | 18 |
