# Touch E3 controller ↔ provider server

The HTTP protocol used by the Touch E3 itself. The controller uploads device
state and polls for commands. Uploads and idle responses are documented; the
non-idle command response is still unknown. Endpoint paths retain the provider’s
`machine/` prefix.

Read the [protocol specification](specification.md). See also
[shared values](../common.md) and the [protocol overview](../../README.md).
