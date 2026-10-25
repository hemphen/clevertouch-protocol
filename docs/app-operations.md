# App operations: schedules and diagnostics

[Human API](human-api.md) · [Overview](../README.md)

Everything here is **Client** evidence from the supplied app (`script.js`,
version string 2.8.1). Requests use the human API's POST/form/bearer conventions.
Response fields listed are those consumed by the app, not complete schemas.
These operations extend the Python client's core read/write subset and have not
been independently exercised for this reference.

The app includes a legacy body `token` on most of these requests. The table names
it where present; its necessity alongside bearer authentication is unverified.
`lang` is added by the common request wrapper.

## Schedules

There are two distinct schedule representations:

- `programme`: a compact string. Observed machine schedules and app templates
  have 336 characters, consistent with seven days × 48 half-hour slots. Values
  `0` and `5` occur. Their mode meanings and the first day of the compact string
  have not been independently established. Do not interpret them as `gv_mode`.
- `program`: a structured week submitted to the server for conversion. Keys
  are `Sun`, `Mon`, `Tue`, `Wed`, `Thu`, `Fri`, `Sat`; each contains an array of
  `{start, end, value}` intervals. The app uses `HH:MM` strings and values
  `comfort`, `eco`, `booster`, and the **string** `"false"` for unselected periods.
  These day names do not prove the ordering of the compact representation.

| Path | Request fields | Response handling |
| --- | --- | --- |
| `human/sandbox/convert_program/` | `device_id` (cloud `id`), `now` (Unix seconds), `width` or `height` for rendering | Code 1; `data` supplies timeline data including `program` |
| `human/device/apply_program/` | `token`, `device_id` (cloud `id`), `peremption`, `program[...]`; edited-week path also sends `new=true` | `OK_SET`; returned data not needed by this caller |
| `human/device/copy_program/` | `token`, `device_id_from`, `device_id_to` (cloud IDs), `peremption` | `OK_SET` |
| `human/device/edit/` | `token`, `smarthome_id`, `id_device` (local ID), `programme` | `OK_SET`; app then queues the same `programme` through `human/query/push/` |

Structured input, shown as an object **before form encoding**, for one day:

```json
{
  "Mon": [
    {"start": "00:00", "end": "08:00", "value": "false"},
    {"start": "08:00", "end": "20:00", "value": "comfort"},
    {"start": "20:00", "end": "00:00", "value": "false"}
  ]
}
```

A complete submission supplies all seven day arrays. An end of `00:00` denotes
the end-of-day boundary in this construction; the app also constructs all-day
intervals from `00:00` to `00:00`. A flattened field is, for example,
`program[Mon][1][value]=comfort`. Overlap handling and server validation rules
are unknown.

The app replaces a schedule from a compact template in two steps:

1. `human/device/edit/` stores `programme` against the local device/home.
2. `human/query/push/` sends `context=1`, `peremption`,
   `query[id_device]` and `query[programme]` to queue delivery.

Its clear-program action uses 336 zero characters in those same two steps.
This establishes the client's clear representation, not the thermal meaning of
zero in every slot. Neither successful database editing nor `OK_SET` confirms
that the panel has applied the schedule. RF schedule transfer is unknown.

## Delivery and liveness

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
