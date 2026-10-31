# CleverTouch protocol reference

Unofficial specification of the three links in a Purmo/LVI Touch E3 heating
system, reverse-engineered from one installation. It may contain errors; each
uncertain claim is labelled.

| Link | Transport | Specification | Status |
| --- | --- | --- | --- |
| App / web ↔ provider server | HTTPS, form POST, JSON | [Web API](docs/web-api/specification.md) | Reads and command delivery observed; schedule editor Client-only |
| Touch E3 ↔ provider server | HTTP GET, JSON in the path | [clevertouch-upstream](docs/clevertouch-upstream/specification.md) | Uploads, polls, mode and schedule commands observed; registration unknown |
| Touch E3 ↔ radiators | 868.3 MHz 2-FSK | [radiator-rf](docs/radiator-rf/specification.md), [vectors](docs/radiator-rf/vectors.json) | All observed frames decoded and rebuilt; a non-panel transmitter untested |

## Reading this reference

- [Shared values](docs/common.md): evidence labels, identifiers, temperatures,
  modes. Read first.
- [Sources](docs/sources.md): what each claim rests on.
- Each specification ends with its open questions.
- Examples use invented identifiers and values. No credentials, device
  identities or household recordings are included.
- Scope: the observed Purmo/LVI equipment. Related Watts and Fenix products may
  differ.

## Implementations

| Project | Role |
| --- | --- |
| [radiator-rf](https://github.com/hemphen/radiator-rf) | Python: receive, decode and build RF frames; generates the vectors |
| [cleverbox](https://github.com/hemphen/cleverbox) | ESP32-C6 + CC1101 firmware bridging radiators to Home Assistant |
| [clevertouch](https://github.com/hemphen/clevertouch) | Python Web API client |
| [hass-clevertouch](https://github.com/hemphen/hass-clevertouch) | Home Assistant integration on the Web API client |

## Changes

[CHANGELOG.md](CHANGELOG.md) lists changes that affect implementations. Cite a
commit hash when implementing. Contribution rules are in [AGENTS.md](AGENTS.md).

## License

[MIT](LICENSE)
