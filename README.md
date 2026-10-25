# CleverTouch protocol reference

A compact, unofficial reference for the three links used by Purmo/LVI Touch E3
heating systems. It describes known wire formats and behaviour, including both
sending and receiving. It is a reverse-engineered specification and may contain
errors; uncertain fields are marked where they are described.

| Link | Transport | Reference |
| --- | --- | --- |
| App / web ↔ provider server | HTTPS, form-encoded requests and JSON responses | [Human API](docs/human-api.md) |
| Touch E3 ↔ provider server | HTTP, JSON in GET paths and JSON responses | [Machine API](docs/machine-api.md) |
| Touch E3 ↔ radiators | 868 MHz FSK, binary frames | [RF protocol](docs/rf.md) |

Start with [shared values and identifiers](docs/common.md). Additional app
operations are in [schedules and diagnostics](docs/app-operations.md), and
[source coverage](docs/sources.md) explains the basis and limits of this reference.

The human API supports reading state and submitting commands. The machine API's
state upload and idle polling are documented; its downstream command envelope
is still unknown. RF reporting, controller state frames and an inclusion
exchange have been observed, but a replacement transmitter has not been validated.

Examples use invented identifiers and values. No account credentials, device
identities, household captures or investigation logs are included. Public service
hostnames and protocol constants are retained. The scope is the observed
Purmo/LVI equipment; related Watts/Fenix products may differ.
