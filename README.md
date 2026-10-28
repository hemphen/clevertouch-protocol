# CleverTouch protocol reference

A compact, unofficial reference for the three links used by Purmo/LVI Touch E3
heating systems. It describes known wire formats and behaviour, including both
sending and receiving. It is a reverse-engineered specification and may contain
errors; uncertain fields are marked where they are described.

| Link | Transport | Reference |
| --- | --- | --- |
| App / web ↔ provider server | HTTPS, form-encoded requests and JSON responses | [Clevertouch Web API](docs/web-api/README.md) |
| Touch E3 ↔ provider server | HTTP, JSON in GET paths and JSON responses | [clevertouch-upstream protocol](docs/clevertouch-upstream/README.md) |
| Touch E3 ↔ radiators | 868 MHz FSK, binary frames | [radiator-rf](docs/radiator-rf/README.md) |

Start with [shared values and identifiers](docs/common.md). The Clevertouch Web API
specification includes [weekly schedule submission](docs/web-api/specification.md#weekly-schedules)
and diagnostics; the radiator-rf reference has [test vectors](docs/radiator-rf/vectors.json).
[Source coverage](docs/sources.md) explains the basis and limits of this reference.

The Clevertouch Web API supports reading state and submitting commands. The clevertouch-upstream protocol's
state upload, idle polling and schedule/mode command responses are documented. RF reporting,
controller state, its construction and the inclusion exchange have been observed, but a replacement
transmitter has not been validated.

Examples use invented identifiers and values. No account credentials, device
identities, household captures or investigation logs are included. Public service
hostnames and protocol constants are retained. The scope is the observed
Purmo/LVI equipment; related Watts/Fenix products may differ.
