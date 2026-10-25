# App ↔ provider server

The HTTPS API used by the web app and client libraries to read homes and
devices, submit commands, and manage weekly schedules. Authentication uses an
account bearer token. The literal endpoint paths retain the provider’s `human/`
prefix.

Read the [protocol specification](specification.md). See also
[shared values](../common.md) and the [protocol overview](../../README.md).

For sending a complete week, start at [weekly schedules](specification.md#weekly-schedules):
use named-day intervals with `apply_program/`; copying an encoded template uses
a separate store-and-queue sequence.
