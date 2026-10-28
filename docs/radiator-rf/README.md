# radiator-rf

The 868 MHz binary radio protocol between the controller and its radiators.
The reference covers both directions, checksums, field encodings and inclusion.
The controller executes schedules and sends phase targets/modes over RF.
Custom-transmitter acceptance remains unverified.

Read the [protocol specification](specification.md); [vectors.json](vectors.json)
is its test data. See also [shared values](../common.md) and the
[protocol overview](../../README.md).

[radiator-rf](https://github.com/hemphen/radiator-rf) is a Python library that
receives and sends this protocol.
