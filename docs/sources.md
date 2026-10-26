# Source coverage

[Overview](../README.md)

Initially compiled on 25 October 2026; schedule submission clarified on 26 October 2026. It describes a source snapshot,
not a promise that the provider's deployed service remains unchanged. No live
account commands or RF transmissions were made to prepare it.

| Source | What it establishes |
| --- | --- |
| [hemphen/clevertouch](https://github.com/hemphen/clevertouch/tree/252274f52b5deab81d5d986300f53fb41511ea44), revision `252274f52b5deab81d5d986300f53fb41511ea44` | Implemented Clevertouch Web API authentication, read/write requests, device fields, modes and temperature conversion |
| Supplied `script.js`, app version string `2.8.1` | App request construction, form encoding, modes, schedules, diagnostics and statistics; static client evidence |
| Supplied `vendor.js` | Supporting JavaScript dependencies; no additional `human/` or `machine/` endpoint definitions identified |
| Private Touch E3 research material, source snapshot 25 October 2026 | Machine uploads and idle responses, RF frame validation, controlled mode/settings changes and observed inclusion exchange |

The private research was consulted to extract protocol facts, not copied as
public evidence. Its captures, tokens, addresses, device mappings, account data,
locations and household activity remain outside this repository. Original app
bundles are also not redistributed here. The following hashes identify the
supplied files without publishing their contents:

| File | SHA-256 |
| --- | --- |
| `script.js` | `6efd38442454a8fc08d04f4200cba2c810080e2848eaed4c2e2f8f58b9ba60fa` |
| `vendor.js` | `92c79aaae815bd59fa25114fc38ca433df22cb6d2e8d6adcd81649c5c34397b5` |

For maintainers checking the supplied app, useful source locations are:

| Subject | `script.js` lines / symbols |
| --- | --- |
| Common modes and displayed targets | 7074–7137, `gv_mode` switch |
| Device command fields | 14500–14583, `query` construction |
| Week structure and built-in compact templates | 15455–15480, `arrayDayWeek`, `objJSONProgram`, `programModels` |
| Structured schedule submission | 16240–16298 (add interval); 16341–16546, `saveEntireProgram` (`new=true`) |
| Quarter-hour boundaries | 16040–16094, `bindStartAndEndTime`; 16884–16894, `stepminute="15"` |
| Decoded schedule response | 16577–16740 and 17101–17165, `updateTimeLineProgram`, `initEditRangePart2`, `updateProgramOnTimeLine` |
| Copy/store/queue schedules | 17341–17503, `copyProgramFrom`, `deleteEntireProgram` |
| Statistics | 2831–2872 and 17540–17615, `StatsService`, `initStats` |
| Default client timings | 18833–18877, `config` |
| Liveness | 20274 onward, `checkSmarthomeLastConnexion` |
| Query submission and failure check | 20590–20753, `makeQueryPart2`, `checkFailure` |
| Form and bearer handling | 21259 onward, `askToAPIWithToken`, `askToAPI` |

## Coverage limits

| Link | Established subset | Main unresolved part |
| --- | --- | --- |
| Clevertouch Web API | Core library requests; additional app request/response usage | Full server validation, complete error schemas, provider-specific support |
| clevertouch-upstream protocol | State-upload keys, success and idle envelopes, polling pattern | Non-idle command response, acknowledgements, identity/registration and error handling |
| RF | Dual-CRC frame envelope, ordinary records, state in both directions, inclusion exchange | Custom transmission acceptance, timing/retries, control loop, schedule transfer, opaque fields |

Evidence labels apply to individual claims. A field appearing in client code is
not by itself a controlled hardware result. Related products and inherited
Watts decoder interpretations are not silently treated as Purmo observations.

Examples are constructed, not redacted captures. The RF examples have synthetic
unit identities and recomputed CRCs; temperature, schedule and HTTP examples
carry no installation state. Future contributions should retain that distinction
and describe uncertainty next to the affected field rather than reproducing an
investigation diary.
