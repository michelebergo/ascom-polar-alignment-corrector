# IAxisAlignerV1 — draft proposal

Status: draft 2, for discussion on ASCOM-Talk Developer (thread #9611, "I need
ASCOM standard for Automatic Polar Alignment"). Author: Michele Bergo (OAPA).
Contributions: Joey Troy (OpenAstro), from AlpacaBridge driver and ConformU
experience. Nothing here is final; names, members and semantics are open to
change.

## What the device is

An axis aligner is a motorised adjuster that changes the orientation of
something it carries by small angles, typically a few arcminutes to a few
degrees, on up to three axes. It does not track, slew or know where the sky
is. A client measures the misalignment (for example by plate solving),
commands the aligner, waits for the move to finish, and measures again until
the error is below its threshold.

The use case that motivates the interface is **polar alignment**: a motorised
base or wedge under an equatorial mount that moves the mount's polar axis in
azimuth and altitude. Examples on the market or in active development: Avalon
UPAS, MLAstro RPA, OpenAstroTech AutoPA, Zenit Align Mini, EZT Astro MPA,
OAPA. The same interface also fits other small-angle alignment jobs, such as
aligning several telescopes relative to each other on one mount.

## Why a new interface

Today none of these devices has a standard ASCOM representation. Each one is
reached in its own way: a serial protocol implemented directly inside a client
plugin, emulation of another vendor's protocol, or commands hidden inside a
Telescope driver. Every client has to learn every device.

The existing interfaces do not fit:

- **Telescope** was suggested earlier in thread #9611. But an aligner does not
  know where the sky is, and a Telescope driver must report `RightAscension`,
  `Declination`, `SiderealTime`, `Tracking`, `Azimuth`, `AtHome` and `AtPark`,
  so the aligner would have to invent them. Clients that discover it over
  Alpaca would also offer it as a mount.
- **Switch** can carry writable values, and Switch V3 can even set a value
  asynchronously. But a client cannot tell an aligner from any other switch;
  direction and units are a naming convention rather than a contract; a switch
  value is an absolute state, so a relative move has to be faked by writing a
  delta and resetting it to zero; there is no per-call move limit; and a failed
  move has no way to say why. The Interface Principle also rules out building
  complex interfaces from well-known switch names.
- **Focuser** has a limited range, but it counts steps rather than angles, its
  conformance tests drive the full travel from 0 to `MaxStep`, and clients
  offer it for autofocus.
- **Rotator** uses degrees, but assumes a full 360° ring. Its conformance tests
  move to 45°, 135°, 225° and 315° and by up to ±375°, several revolutions in
  all, so an aligner with a few degrees of travel cannot pass. Clients also
  offer it for camera framing.

This is the same situation that led to CoverCalibrator in Platform 6.5: Switch
was usable but awkward, and a small dedicated interface was clearer for both
drivers and clients.

INDI added an interface for polar alignment correctors, `PACInterface`, in
INDI 2.2.0 (April 2026), used by the Ekos Polar Alignment Assistant since
KStars 3.8.2. This draft keeps its behaviour, so that a device can offer the
same thing under both frameworks.

## Conventions

- **Axes.** Up to three axes, `X`, `Y` and `Z`. Each is an angle. Which axes a
  device has is reported by `CanMoveX`, `CanMoveY` and `CanMoveZ`. What each
  axis means is fixed by a profile (below) or, outside a profile, documented by
  the driver.
- **Units.** All angles are in degrees, as elsewhere in ASCOM.
- **Relative moves are the common capability.** Most aligners have no absolute
  encoder, so every device implements relative moves. Devices that know their
  position can also implement absolute moves.
- **Accuracy.** A move is a best-effort request. Backlash, gearing and load mean
  an axis may not land exactly on the requested amount. The driver applies
  whatever compensation it has (backlash, calibration); the client is expected
  to re-measure after each move. The interface does not promise a landing
  accuracy.
- **Configuration stays in the driver.** Speed, motor direction reversal,
  backlash compensation and calibration are device setup, handled in the
  driver's setup dialog or web page. A client does not need them to close the
  loop, so they are not interface members.

### Polar alignment profile

A device used to correct polar alignment follows this profile, so that any
polar alignment client can drive any such device. Directions are defined on
the polar axis, so they read the same in both hemispheres:

- `X` is azimuth: positive `X` moves the polar axis towards **east**;
- `Y` is altitude: positive `Y` **raises** the polar axis (increases its
  elevation above the horizon);
- `Z` is not used: `CanMoveZ` is false.

Example: the client measures the polar axis 5′ too low. It calls
`Move(0, +0.0833, 0)`.

INDI defines "ALT + = North". This profile defines positive altitude as
raising the polar axis, which is the same thing in the northern hemisphere and
removes the ambiguity in the southern one.

## Members

In addition to the common members of every Platform 7 device (`Action`,
`SupportedActions`, `Connect`, `Disconnect`, `Connected`, `Connecting`,
`DeviceState`, `Description`, `DriverInfo`, `DriverVersion`,
`InterfaceVersion` = 1, `Name`).

`Connect` opens the link and reads the device's state. It does not home or
calibrate; clients give up on a device that stays `Connecting` for more than a
few seconds. `Disconnect` while a move is in progress halts the move first.

### Methods

| Member | Kind | Summary |
|---|---|---|
| `Move(X, Y, Z)` | async initiator | Starts a relative move of every axis by the given amounts, in degrees. A driver that cannot move axes together moves them one after another inside the same operation. Completion: `IsMoving` becomes false. |
| `MoveAbsolute(X, Y, Z)` | async initiator, optional | Starts a move of every axis to the given positions, in degrees. `MethodNotImplementedException` when `CanMoveAbsolute` is false. Completion: `IsMoving` becomes false. |
| `FindHome()` | async initiator, optional | Moves every axis to its home position. `MethodNotImplementedException` when `CanFindHome` is false. Completion: `IsMoving` becomes false. |
| `Sync(X, Y, Z)` | synchronous, optional | Tells the driver that the axes are now at the given positions, without moving them. `MethodNotImplementedException` when `CanSync` is false. |
| `Halt()` | synchronous | Stops motion on every axis. Returns when motion has stopped; on return `IsMoving` is false. Calling it while idle does nothing and does not throw. |

For an axis whose `CanMove` property is false, the amount passed to `Move`,
`MoveAbsolute` or `Sync` must be 0.

### Properties

| Member | Type | Summary |
|---|---|---|
| `IsMoving` | boolean | True while any move or homing is in progress. The completion property for `Move`, `MoveAbsolute` and `FindHome`. |
| `CanMoveX`, `CanMoveY`, `CanMoveZ` | boolean | True if the device has that axis. |
| `CanMoveAbsolute` | boolean | True if `MoveAbsolute` is implemented. |
| `CanFindHome` | boolean | True if `FindHome` is implemented. |
| `CanSync` | boolean | True if `Sync` is implemented. |
| `XMaxMove`, `YMaxMove`, `ZMaxMove` | double | Largest relative move of that axis, in degrees (absolute value), that a single `Move` accepts. A limit per call, not the travel left. 0 for an axis the device does not have. |
| `XPosition`, `YPosition`, `ZPosition` | double | Position of that axis from the driver's zero, in degrees. `PropertyNotImplementedException` when the device does not track its position, or does not have that axis. |

A device that implements `MoveAbsolute` or `Sync` must also implement the
position properties of the axes it has.

The driver chooses where zero is (power-on, a home switch, a user reset),
documents it, and says whether it survives a disconnect or power cycle. After
a successful `FindHome` the positions read the driver's home values. Positions
may be counted from commanded steps rather than measured; the interface does
not promise a measurement. During a move they return the latest value the
driver has. Positions use the same signs as moves.

The `Can*` properties answer while disconnected. Every other member in this
section throws `NotConnectedException` while disconnected, including the
`MaxMove` properties, which may come from the device.

### DeviceState

`IsMoving`, `TimeStamp`, and the position properties the device implements.

The usual rules apply: the list is empty while disconnected, and a member whose
getter would throw is left out. So after a failed move `IsMoving` is absent
from `DeviceState`; a client that does not find it there reads the `IsMoving`
property, which reports the failure.

## Behaviour of the async operations

These follow the existing Platform 7 rules for asynchronous operations.

- An initiator validates everything it can before returning, in this order.
  It throws instead of returning when it already knows the move cannot
  complete:
  - `InvalidValueException` if an amount is not a finite number, is outside
    ±`MaxMove` for its axis (`Move`), is not 0 for an axis the device does not
    have, or would take an axis past the end of its travel when the driver
    knows the travel (checked even while disconnected, except the travel
    check);
  - `NotConnectedException` if not connected;
  - `InvalidOperationException` if a move is already in progress (the client
    calls `Halt` first), or if the device is not ready to move, for example
    because it has not been calibrated; the message says which. A refused
    call leaves the running move untouched.
- An initiator returns as soon as the move has started, never after it has
  finished. `IsMoving` is already true when the initiator returns, so a client
  that polls at once never sees false before the move has begun. The driver
  absorbs any motor start-up delay.
- A zero move is valid and completes immediately.
- A failure discovered during the move (stall, lost link, end of travel) stops
  the axis and is reported by `IsMoving` throwing `DriverException` on every
  read until the next initiator call (a zero move will do), `Halt`, `Connect`
  or `Disconnect`. The message names the axis. After an end of travel the
  position properties report where the axis stopped. They do not throw for
  this; they throw only when the device itself can no longer answer.
- A driver that waits for the device to confirm a move treats no confirmation
  within its own timeout as a failure, not as arrival.
- After `Halt`, `IsMoving` returns false without an exception. The move is
  cancelled, not failed.

## Alpaca endpoints

Device type in the URL: `axisaligner`.

| Verb | Path | Parameters |
|---|---|---|
| PUT | `/api/v1/axisaligner/{device_number}/move` | `X`, `Y`, `Z` |
| PUT | `/api/v1/axisaligner/{device_number}/moveabsolute` | `X`, `Y`, `Z` |
| PUT | `/api/v1/axisaligner/{device_number}/findhome` | |
| PUT | `/api/v1/axisaligner/{device_number}/sync` | `X`, `Y`, `Z` |
| PUT | `/api/v1/axisaligner/{device_number}/halt` | |
| GET | `/api/v1/axisaligner/{device_number}/ismoving` | |
| GET | `/api/v1/axisaligner/{device_number}/canmovex`, `canmovey`, `canmovez` | |
| GET | `/api/v1/axisaligner/{device_number}/canmoveabsolute` | |
| GET | `/api/v1/axisaligner/{device_number}/canfindhome` | |
| GET | `/api/v1/axisaligner/{device_number}/cansync` | |
| GET | `/api/v1/axisaligner/{device_number}/xmaxmove`, `ymaxmove`, `zmaxmove` | |
| GET | `/api/v1/axisaligner/{device_number}/xposition`, `yposition`, `zposition` | |

Error numbers, all returned with HTTP 200 and the number in `ErrorNumber`, as
for every Alpaca device:

| Number | Error | Used for |
|---|---|---|
| 0x400 | NotImplemented | `moveabsolute`, `findhome`, `sync` when their `Can*` is false; positions the device does not track |
| 0x401 | InvalidValue | amount out of range, not finite, non-zero on a missing axis, or past the known travel |
| 0x407 | NotConnected | any member other than the `can*` properties while disconnected |
| 0x40B | InvalidOperation | move in progress, or device not ready; never NotImplemented for this |
| 0x500-0xFFF | driver-specific | failure during a move, read from `ismoving` |

## Mapping to INDI PACInterface

For a device following the polar alignment profile:

| INDI `PACInterface` | This draft |
|---|---|
| `MoveAZ(deg)` | `Move(deg, 0, 0)` |
| `MoveALT(deg)` | `Move(0, deg, 0)` |
| `MoveBoth(az, alt)` | `Move(az, alt, 0)` |
| `AbortMotion()` | `Halt()` |
| property state BUSY / OK / ALERT | `IsMoving` true / false / throws |
| `PAC_HAS_POSITION`, `PAC_POSITION` | `XPosition`, `YPosition` |
| `PAC_CAN_HOME`, `PAC_HOME` | `CanFindHome`, `FindHome()` |
| `PAC_CAN_SYNC`, `PAC_SYNC` | `CanSync`, `Sync(x, y, 0)` |
| `PAC_HAS_SPEED`, `PAC_CAN_REVERSE`, `PAC_HAS_BACKLASH` | driver setup, not in the interface |

INDI has no `MaxMove`; its manual step is ±10°, so a bridge to an INDI device
would report 10.

## Open questions

1. **Name.** `AxisAligner` fits the generic interface. `PolarAligner` names the
   main use case and is easier to find, but reads like a software routine and
   undersells the other uses. Other suggestions welcome.
2. **Should a device say what it aligns?** With generic axes, a polar
   alignment client cannot tell a polar aligner from, say, a device that
   aligns a second telescope. A small read-only property such as
   `Application` (polar axis / optical axis / other) would let it pick the
   right device and trust the profile.
3. **Travel range.** Should devices that know their travel expose it per axis
   (minimum and maximum position), so a client can plan absolute moves instead
   of discovering the limit from an `InvalidValueException`?
4. **Conformance.** ConformU would need a small travel budget per test so it
   does not run an aligner to the end of its travel, unlike the Rotator and
   Focuser suites above. A shape that works: each move uses a small fraction
   of `MaxMove`, moves are paired +x then −x so net travel is zero, no test
   exceeds `MaxMove`, and `Halt` is tested on a move of that size. `FindHome`
   is tested only when the device documents that homing is safe to run
   unattended.

## A working reference

I am building an Alpaca driver for OAPA (open hardware and firmware), aimed
at OpenAstro AlpacaBridge, which ships only the standard device types. Until
an interface exists it will expose this behaviour through custom `Action`s
whose names match the members above and are listed in `SupportedActions`, so a
client can recognise an aligner; any other name throws
`ActionNotImplementedException`. Which existing interface should carry those
actions in the meantime is a question for the Initiative. The driver can serve
as a test device while the interface is discussed.

## Revision history

- **Draft 2.** Generic axes `X`, `Y`, optional `Z`, with a polar alignment
  profile; renamed from `IPolarAlignmentCorrectorV1`. One relative `Move` for
  all axes, mandatory. Per-axis `CanMoveX/Y/Z` replace `CanMoveBoth`. Position
  properties throw `PropertyNotImplementedException` instead of a
  `CanReportPosition` flag. Optional `MoveAbsolute`, `FindHome` and `Sync`
  with `Can*` flags. End of travel defined. Telescope added to the rationale.
  (Feedback from Peter Simpson in thread #9611.)
- **Draft 1, revised.** Async, disconnection and error semantics, Alpaca error
  numbers and a conformance shape, from AlpacaBridge experience (Joey Troy).
- **Draft 1.** First version.
