# radiator-rf protocol specification

[Overview](../../README.md) · [Shared values](../common.md) · [clevertouch-upstream protocol](../clevertouch-upstream/specification.md)

This describes the observed Purmo/LVI Touch E3 profile of a Watts-family
protocol. Frame layouts and CRCs are established; some field meanings and the
requirements for successful custom transmission remain unresolved. Message
names such as `controller_state` are descriptive names used here, not on-wire
strings or vendor terminology.

## Physical layer

| Parameter | Observed / working receiver value |
| --- | --- |
| Centre frequency | 868.3 MHz; inherited setting that successfully receives these frames |
| Modulation / coding | 2-FSK / PCM |
| Symbol duration | Approximately 27 µs, about 37 kbit/s; best of tested receiver slicer settings |
| Deviation | Approximately ±16–18 kHz from IQ analysis |
| Working synchronisation pattern | Hex `AA D3 91 D3 91` |
| Receiver reset/gap threshold | 1000 µs in the working decoder; not a measured transmit requirement |

The 27 µs value improves validated frame recovery over the earlier 26 µs receiver
setting. It is not a precise hardware baud-rate measurement or a validated
CC1101 register profile. The listed pattern is the receiver's synchronisation
match, not a complete specification of preamble length. Post-sync data is
represented bytewise below, with bits read most-significant first by the receiver.
The recovered bytes validate directly under both CRCs; no additional whitening
step is needed for that decoded representation. Transmit timing tolerances,
frequency tolerance, repetition policy and channel-access rules are unknown.

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

These four shapes were invariant within the examined ordinary-traffic samples.
Lengths include both CRCs in the total column. Tags are listed in on-wire order.

| Message | Direction | `L` | Total | Record order |
| --- | --- | ---: | ---: | --- |
| `radiator_report` | Radiator → controller | `23` hex / 35 | 38 | `DF 4B 8A E1 DA 0A` |
| `controller_state` | Controller → radiator | `38` hex / 56 | 59 | `45 4B 3B 4C 8E CC 97 94 E1 DA 5B B2` |
| `ack_candidate` | Radiator → controller | `0F` hex / 15 | 18 | `01 0A` |
| `request_candidate` | Controller → radiator | `0D` hex / 13 | 16 | `19` |

`radiator_report` supplies measured temperature and current state.
`controller_state` carries a target/mode and controller-side state; panel-driven
changes were observed in this direction. Its reported temperature is an echo,
not an independent sensor measurement. Duty may be computed by the controller,
but sparse radiator reporting prevents a firm conclusion about where the
control loop runs.

The short frames contain `01=00, 0A=47` and `19=00`, respectively. Their names
are **tentative**: no sequence number, result code or acknowledged-command ID
has been identified in them. Do not infer success from their presence alone.
Controller state/request frames were concentrated around activity in the sampled
sessions; no universal idle heartbeat or retry interval is established.

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
| `DA` | 4 | Report, state | Two uint16 candidate interval on-times in seconds | Tentative; zero in sampled controller states |
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

Constant values describe the sample, not guaranteed constants across models.
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
to zero; it must not be labelled a simple heating-active bit. `97` values showed
byte 0 matching `DF` byte 0, byte 1 `30`, and byte 2 `04` or `05`. No bit-level
meaning is established.

`5B` varies both between devices and slightly over time. Changing a configured
wattage did not change it. `B2` is consistent with a binary clock: require
hour < 24 and minute/second < 60 before displaying it; do not infer a date,
time zone or synchronisation requirement.

## Inclusion / pairing

Observed with an already-used radiator placed in its `rF.bi` inclusion mode and
removed from the controller's device list. A factory-fresh device and a custom
transmitter have not been tested. The four-byte addressing and dual-CRC envelope
remain the same; the payload grammar differs.

| Step | Direction | `L` | Payload only | Interpretation |
| --- | --- | ---: | --- | --- |
| Offer | Controller → candidate address | 15 | `00 00 03 01` | Inclusion offer, opaque fixed-layout payload |
| Reply | Radiator → controller | 13 | `00 00` | Empty ordinary record stream / padding |
| Bind | Controller → replying radiator | 13 | `01 00` | Single tag `01` with value zero |
| Ack | Radiator → controller | 15 | `01 00 0A 47` | Same shape as ordinary ack candidate |

The controller swept candidate destinations `F1 10 00 01`, `D0 00 00 02`,
`F1 10 00 03`. These are observed discovery values, **not installed device IDs**.
It changed its source family byte to match each candidate family, then used the
radiator's returned address for binding. The full candidate enumeration range,
retry policy and stopping rules are unknown.

One refused/already-installed attempt used offer payload `00 00 03 64` instead
of `00 00 03 01`. The final byte's meaning is unknown; it is not established as
a mode, count or error code. Replies followed offers in the observed inclusion
window; this does not prove that unsolicited discovery never occurs.

The exchange took about three seconds with roughly one-second gaps. Those are
observations, not transmit deadlines. Re-inclusion produced identical handshake
bytes on two attempts, with no visible changing challenge or counter. That does
not establish replay acceptance, absence of hidden state, or acceptance of an
arbitrary controller identity.

Re-inclusion reset the tested radiator's stored values to Comfort 19 °C,
Frost 7 °C, Eco 15.5 °C, both unresolved controller upload temperature fields `10`/`11`
to 25 °C, and timer to 7200 seconds. These are **observed reset values for that
unit**, not universal factory defaults. Panel count and wattage settings are
separate from the four-frame exchange; their RF encoding is not established.

## Schedule execution and command delivery

**Observed on 26 October 2026:** the Touch E3 executes the weekly Comfort/Eco
schedule. Panel edits and cloud-delivered schedule changes did not produce a
whole-week RF message in the observed experiment. At phase boundaries the
controller initiated an exchange with `request_candidate`, then sent
`controller_state` with tag `94` containing the phase target and mode:
8 for Auto Comfort, 11 for Auto Reduced.

Quarter-hour boundaries executed 2–4 seconds after the boundary according to
the controller's `B2` clock in this sample. The radiator receives the selected
target and mode; a controller or external automation must execute the schedule.
No RF schedule-upload format is established. These observations do not prove
that every firmware lacks other schedule functionality.

A cloud-requested Comfort change waited for the next radiator periodic exchange,
about 115 seconds after panel acceptance in one sample. The panel had already
reported the new mode to the server. Neither server readback nor a panel upload
proves the radiator has received it.

Schedule execution is distinct from thermostat regulation: these findings do
not settle which device computes heating duty or what Auto does after controller
loss. They do not establish a fixed heartbeat or retransmission interval.

## Sending and receiving boundaries

A receiver can validate both CRCs, recognise the ordinary layouts and decode the
established fields above. Preserve unfamiliar but CRC-valid payloads as unknown
message types rather than discarding them as noise.

A transmitter can construct a structurally valid envelope from this reference,
but successful control is **not validated**. In particular, do not assume that
changing only `94` in a recorded state frame is sufficient: the role of opaque
state fields, control-loop responsibility, exchange timing and retry behaviour
remain open. No RF schedule upload format is known. Observe later radiator
reports to establish an applied target/mode; neither a valid CRC nor a short
ack-shaped frame alone proves command success.

## Synthetic frame examples

The following examples use invented controller `F1010203` and radiator
`F1040506` addresses. They were generated from the documented layouts and checked
with both CRC algorithms. They test parsing/encoding only, not radio acceptance.
Discovery destination `F1100001` is a protocol candidate value, not a private ID.

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
