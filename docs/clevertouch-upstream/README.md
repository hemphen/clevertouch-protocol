# clevertouch-upstream

The HTTP protocol used by the Touch E3 itself. The controller uploads device
state and polls for commands. Uploads, idle responses and observed schedule/mode command
responses are documented. Endpoint paths retain the provider’s
`machine/` prefix.

Read the [protocol specification](specification.md). See also
[shared values](../common.md) and the [protocol overview](../../README.md).

The research daemon for this protocol is `clevertouched` in
[clevertouch-radio](https://github.com/hemphen/clevertouch-radio/tree/main/clevertouched).
