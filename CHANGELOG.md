# Changelog

Changes that affect implementations. Cite a commit hash when implementing
against this reference.

## Unreleased

- Restructured the specifications into tables and short rules; no protocol change.
- RF: added exchange and inclusion sequence diagrams and the EU band rules
  (h1.5: 1 % duty cycle or LBT+AFA).
- RF: `DA` reset stated as observed (the next report after an answered one is
  zero) with its trigger unknown, instead of "every exchange resets".
- Web API: added observed token lifetimes and rotation, `ERR_NO_DATA`, the
  `smarthome_id` structure, and the full endpoint inventory.
- Moved open questions from `sources.md` into each specification.

## 5dc2bcf — 2026-10-30

- RF `5B` is the Boost time in minutes, held and counted by the controller.
- RF `DF` byte 2 bit 0 is the heating bit; byte 1 bit 0 marks a change made at
  the radiator.
- RF `97` is derived from the answered report's `DF`: byte 0 and the heating bit.
  Building `controller_state` must derive it, not copy it.
- Upload fields `10`, `11`, `33` and `34` mapped; `DA ≈ T·d` fitted.

## 83afd49 — 2026-10-29

- Decoders report temperatures in °F as sent: `temperature_F`, `setpoint_F`
  replace the Celsius fields in the vectors.

## 367ed28 — 2026-10-29

- Short frames renamed `request` and `ack` (were `request_candidate`,
  `ack_candidate`).
- `E1` bytes 2 and 3 are one element's duty on two scales, not two panels.

## c3ef38a — 2026-10-29

- Only whole observed payload shapes are messages; anything else is `unknown`.
- Fixed the inner-CRC vector.

## 3b22a90 — 2026-10-27

- Added `docs/radiator-rf/vectors.json`.

## b70fab7 — 2026-10-26

- Compact schedule encoding, panel command delivery and RF schedule execution.
