# ble-central-nv

The Central role of Bluetooth LE as a trait, at the level of GATT operations, over any stack.

A phone gives a program scan, connect, discover, read, write and subscribe and never the radio below them.  The noble Host gives the same operations
over a radio it drives itself.  A test gives them over nothing.  A program that codes against `GattCentral` runs over all three, and a service client
written once (a heart rate collector, the client of the SMP update service) is tested on a PC against a double and run on a phone.

The package holds no stack.  An implementation lives where its stack is: the noble Host's in `noble`, a phone's in the package of its platform, the
double's here.

## The trait

```novo ignore
pub trait GattCentral[e]
    fn caps(self) -> Caps
    fn adapter(self) -> AdapterState [e]
    fn scan_start(self, filter: ScanFilter) -> ?BleError [e, mutate]
    fn scan_stop(self) [e, mutate]
    fn connect(self, p: PeerId, opts: ConnectOptions) -> Result<LinkId, BleError> [e, mutate]
    fn disconnect(self, link: LinkId) [e, mutate]
    fn discover(self, link: LinkId) -> ?BleError [e, mutate]
    fn services(self, link: LinkId) -> [ServiceInfo]
    fn read(self, link: LinkId, ch: CharRef) -> ?BleError [e, mutate]
    fn write(self, link: LinkId, ch: CharRef, data: [u8], ack: Bool) -> ?BleError [e, mutate]
    fn subscribe(self, link: LinkId, ch: CharRef, mode: SubMode) -> ?BleError [e, mutate]
    fn request_mtu(self, link: LinkId, mtu: Int) -> ?BleError [e, mutate]
    fn max_write(self, link: LinkId, ack: Bool) -> Int
    fn priority(self, link: LinkId, p: Priority) -> ?BleError [e, mutate]
    fn next_event(self, timeout_ms: Int) -> ?CentralEvent [e, mutate]
```

Four rules shape it, and each follows from what a phone gives:

- **Every operation returns when the implementation has accepted it; its result is an event.**  A program waits in one place, `next_event`, which
  parks the task that calls it.  The one-operation-at-a-time rule of a platform belongs to the implementation, which queues, so a program may issue
  operations back to back.  Results of operations on a link arrive in the order the operations were issued.
- **A peer is an opaque key.**  An address where the platform has one, an identifier where it has not.  A program keeps the key with `peer_key` and makes
  the peer again with `peer`; it never reads the key.
- **Security is a request.**  `ConnectOptions.security` names the level the link must reach; the implementation decides how and whether, and a platform
  shows its own dialog.  No procedure and no passkey entry is part of the trait.
- **Parameters are advice, and a write has a size.**  `priority` asks for a connection interval, `request_mtu` for an MTU; `caps` says what the
  implementation can act on.  `max_write(link, ack)` is the largest write the link takes, which is the MTU less its header where there is an MTU and what the
  platform reports where there is none.

An event that was not waited for stays in the stream.  A stream that overflows says so with `Dropped`.

## The session

`session` makes the calls every service client makes and waits for each result.  It keeps in a `Pump` the events a wait did not ask for, so a
notification that arrives while a read is pending is found by the next `notification`.

```novo ignore
let p = session.pump()
match session.open_link(central, p, peer, central.connect_default(), 247)
    Err(e)   => println(e.message())
    Ok(link) =>
        match session.find_char(central, link, central.uuid16(0x180D), central.uuid16(0x2A37))
            None     => println("no heart rate measurement")
            Some(ch) =>
                let _ = session.subscribe_wait(central, p, link, ch.ref, SubMode.Notify, 5000)
                match session.notification(central, p, link, ch.ref, 5000)
                    Ok(bytes) => println("${list.len(bytes)} bytes")
                    Err(e)    => println(e.message())
```

An error is a `BleError`: a code (`E_TIMEOUT`, `E_NOT_CONNECTED`, `E_ADAPTER_OFF`, `E_NOT_FOUND`, ..., and `E_GATT` plus the platform's status for a status that is
not 0) and a sentence.

## The double and the suite

`scripted.scripted()` is a peripheral inside a `GattCentral`: a test adds devices and a service, and every call is answered with its event.  A test
pushes the events a peripheral makes on its own, turns the next call into a refusal or into a call that is never answered, and reads back the calls.
`faulty.faulty(...)` forwards to it and misbehaves where a test asks.

`conformance.run(central, peer, plan, ms)` makes the calls of the contract on a device and answers the checks that failed, none when all passed.  It runs
against the double in this package's tests and against a board on the bench from a phone or a PC.

| Module | Contents |
| --- | --- |
| `central` | `GattCentral`, `CentralEvent`, `Uuid`, `PeerId`, `LinkId`, `CharRef`, `ServiceInfo`, `CharInfo`, `Caps`, `ScanFilter`, `ConnectOptions`, `BleError` and their constructors |
| `session` | `Pump`, `await_event`, `connect_wait`, `discover_wait`, `find_service`, `find_char`, `read_wait`, `write_wait`, `write_all`, `subscribe_wait`, `notification`, `mtu_wait`, `open_link` |
| `scripted` | `Scripted`, the double |
| `faulty` | `Faulty`, the double with defects |
| `conformance` | `Plan` and `run` |

## Install and test

```
novo pkg add ble-central-nv
novo test tests/session_tests.nv        # one suite; the others are central_tests, scripted_tests and conformance_tests
bash tests/coverage.sh                  # the lines of src/ the suites reach
```
