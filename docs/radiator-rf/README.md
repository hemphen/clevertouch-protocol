# radiator-rf

The 868 MHz binary radio protocol between the controller and its radiators.
The reference covers both directions, checksums, field encodings and inclusion.
The controller executes schedules and sends phase targets/modes over RF.
Custom-transmitter acceptance remains unverified.

Read the [protocol specification](specification.md). See also
[shared values](../common.md) and the [protocol overview](../../README.md).

The research daemon for this protocol is `radiatord` in
[clevertouch-radio](https://github.com/hemphen/clevertouch-radio/tree/main/radiatord).
