# Sources

[Overview](../README.md)

Compiled in October 2026. It describes the provider's service as it was then.

| Source | Establishes |
| --- | --- |
| [hemphen/clevertouch](https://github.com/hemphen/clevertouch/tree/252274f52b5deab81d5d986300f53fb41511ea44) at `252274f` | Web API authentication, reads, writes, device fields, modes, temperature conversion (**Client**) |
| The provider's web app, `script.js` (version 2.8.1) and `vendor.js` | Request construction, form encoding, modes, schedules, diagnostics, statistics (**Client**). `vendor.js` defines no further `human/` or `machine/` endpoints |
| Controlled observation of one Touch E3 installation | Panel uploads and polls, delivered commands, RF frames in both directions, exchanges and timing, controller-state construction, inclusion, schedule execution (**Observed**) |

The observation recorded the panel's HTTP traffic, its 868 MHz traffic and the
Web API at the same time, while changes were made at the panel, at a radiator,
in the web app and through supervised Web API commands. No RF was transmitted.
Counts such as "119 of 119" refer to those recordings. The recordings, addresses,
account data and household activity are not published.

## Web app

The bundles are not redistributed. They are identified by hash:

| File | SHA-256 |
| --- | --- |
| `script.js` | `6efd38442454a8fc08d04f4200cba2c810080e2848eaed4c2e2f8f58b9ba60fa` |
| `vendor.js` | `92c79aaae815bd59fa25114fc38ca433df22cb6d2e8d6adcd81649c5c34397b5` |

| Subject | `script.js` lines / symbols |
| --- | --- |
| Modes and displayed targets | 7074–7137, `gv_mode` switch |
| Device command fields | 14500–14583, `query` construction |
| Week structure and templates | 15455–15480, `arrayDayWeek`, `objJSONProgram`, `programModels` |
| Schedule submission | 16240–16298 (add interval); 16341–16546, `saveEntireProgram` (`new=true`) |
| Quarter-hour boundaries | 16040–16094, `bindStartAndEndTime`; 16884–16894, `stepminute="15"` |
| Decoded schedule | 16577–16740, 17101–17165, `updateTimeLineProgram`, `initEditRangePart2`, `updateProgramOnTimeLine` |
| Copy, store and queue schedules | 17341–17503, `copyProgramFrom`, `deleteEntireProgram` |
| Statistics | 2831–2872, 17540–17615, `StatsService`, `initStats` |
| Client timings | 18833–18877, `config` |
| Liveness | 20274 onward, `checkSmarthomeLastConnexion` |
| Query submission and failure check | 20590–20753, `makeQueryPart2`, `checkFailure` |
| Form and bearer handling | 21259 onward, `askToAPIWithToken`, `askToAPI` |

## Related decoders

- [rtl_433 Watts Vision decoder](https://github.com/merbanan/rtl_433/blob/406f9a33550a2b6b3d0d8a8a486417ce5b3f13ab/src/devices/watts_vision.c)
  and [issue #2885](https://github.com/merbanan/rtl_433/issues/2885): the shared
  envelope, checksums and tags of the Watts family. Their interpretations are not
  assumed for Purmo without observation.
- [Yali Digital installation manual](https://www.purmo.com/docs/34019-Yali-Digital-Installation-Manual-COMBO-WEB.pdf):
  the radiator's parameter menu, including `rF.bi` inclusion mode.
