# Changelog

Every published version, newest first.  This file is on the publish allow-list, so it travels with the package.

## 0.1.0

First release.

- `central`: `GattCentral`, the Central role at the level of GATT operations (scan, connect, discover, read, write, subscribe, a request for the
  MTU, the largest write, advisory connection priority and one `next_event` to wait in), the events it reports, an opaque `PeerId`, `Uuid`
  with the Bluetooth base, the errors, and the `Caps` an implementation reports.
- `session`: the calls every service client makes (connect and wait, discover, find a service or a characteristic, read, write in pieces,
  subscribe, the next notification, the MTU, `open_link`) over a `Pump` that keeps the events a wait did not ask for.
- `scripted`: a peripheral inside a `GattCentral`, for tests.
- `faulty`: the scripted double with defects a test can switch on.
- `conformance`: the checks every implementation passes, run against a device named by a `Plan`.
