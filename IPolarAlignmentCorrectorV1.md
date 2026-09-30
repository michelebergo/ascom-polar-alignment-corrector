# IPolarAlignmentCorrectorV1 — draft proposal

Status: draft for discussion on ASCOM-Talk Developer. Author: Michele Bergo (OAPA).
Nothing here is final; names, members and semantics are open to change.

## What the device is

A polar alignment corrector is a motorised base or adjuster that sits under an
equatorial mount and moves the mount's polar axis in azimuth and altitude by
small amounts, typically a few arcminutes to a few degrees. It does not track,
slew or know where the sky is. A client (a polar alignment routine in imaging
software) measures the polar alignment error by plate solving, commands the
corrector, waits for the move to finish, and measures again until the error is
below its threshold.

Examples on the market or in active development: Avalon UPAS, MLAstro RPA,
OpenAstroTech AutoPA, Zenit Align Mini, OAPA.

## Why a new interface

Today none of these devices has a standard ASCOM representation. Each one is
reached in its own way: a serial protocol implemented directly inside a client
plugin, emulation of another vendor's protocol, or commands hidden inside a
Telescope driver. Every client has to learn every device.

The existing interfaces do not fit:

- **Switch** can carry two writable values, and that is what we use today as a
  stopgap. Switch V3 can even set a value asynchronously. But a client cannot
  tell a corrector from any other switch; direction and units are a naming
  convention rather than a contract; a switch value is an absolute state, so a
  relative move has to be faked by writing a delta and resetting it to zero;
  there is no per-call move limit; and a failed move has no way to say why. In
  practice clients also treat switches as synchronous.
- **Focuser** has a limited range, but it counts steps rather than angles, its
  conformance tests drive the full travel from 0 to `MaxStep`, and clients
  offer it for autofocus.
- **Rotator** uses degrees, but assumes a full 360° ring. Its conformance tests
  move to 45°, 135°, 225° and 315° and by up to ±375°, several revolutions in
  all, so a corrector with a few degrees of travel cannot pass. Clients also
  offer it for camera framing.

This is the same situation that led to CoverCalibrator in Platform 6.5: Switch
was usable but awkward, and a small dedicated interface was clearer for both
drivers and clients.

INDI added an equivalent interface, `PACInterface`, in INDI 2.2.0 (April 2026),
used by the Ekos Polar Alignment Assistant since KStars 3.8.2. This draft
follows it closely, so that a device can offer the same behaviour under both
frameworks.

## Conventions

- **Units.** All angles are in degrees, as elsewhere in ASCOM.
- **Moves are relative.** A move asks the corrector to change the orientation
  of the polar axis by the given amount from where it is now. Most correctors
  have no absolute encoder, so relative motion is the common capability.
- **Direction.** Defined on the polar axis, so it reads the same in both
  hemispheres:
  - positive azimuth moves the polar axis towards **east**;
  - positive altitude **raises** the polar axis (increases its elevation above
    the horizon).

  Example: the client measures the polar axis 5′ too low. It calls
  `MoveAltitude(+0.0833)`.
- **Accuracy.** A move is a best-effort request. Backlash, gearing and load mean
  the axis may not land exactly on the requested amount. The driver applies
  whatever compensation it has (backlash, calibration); the client is expected
  to re-measure after each move. The interface does not promise a landing
  accuracy.
- **Configuration stays in the driver.** Speed, motor direction reversal,
  backlash compensation and calibration are device setup, handled in the
  driver's setup dialog or web page. A client does not need them to close the
  loop, so they are not interface members.

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
| `MoveAzimuth(Degrees)` | async initiator | Starts a relative azimuth move of the polar axis. Completion: `IsMoving` becomes false. |
| `MoveAltitude(Degrees)` | async initiator | Starts a relative altitude move of the polar axis. Completion: `IsMoving` becomes false. |
| `Move(AzimuthDegrees, AltitudeDegrees)` | async initiator, optional | Starts both moves as one operation. Throws `MethodNotImplementedException` when `CanMoveBoth` is false. |
| `Halt()` | synchronous | Stops motion on both axes. Returns when motion has stopped; on return `IsMoving` is false. Calling it while idle does nothing and does not throw. |

### Properties

| Member | Type | Summary |
|---|---|---|
| `IsMoving` | boolean | True while any move is in progress. The completion property for all three move methods. |
| `CanMoveBoth` | boolean | True if `Move` is implemented. |
| `AzimuthMaxMove` | double | Largest azimuth move, in degrees (absolute value), that a single call accepts. A limit per call, not the travel left. |
| `AltitudeMaxMove` | double | Largest altitude move, in degrees (absolute value), that a single call accepts. A limit per call, not the travel left. |
| `CanReportPosition` | boolean | True if the device tracks its own position. |
| `AzimuthPosition` | double | Azimuth offset of the polar axis from the driver's zero, in degrees. `PropertyNotImplementedException` when `CanReportPosition` is false. |
| `AltitudePosition` | double | Altitude offset of the polar axis from the driver's zero, in degrees. `PropertyNotImplementedException` when `CanReportPosition` is false. |

The driver chooses where zero is (power-on, a home switch, a user reset),
documents it, and says whether it survives a disconnect or power cycle.
Positions may be counted from commanded steps rather than measured; the
interface does not promise a measurement. During a move they return the latest
value the driver has. Positions use the same signs as moves.

`CanMoveBoth` and `CanReportPosition` answer while disconnected. Every other
member in this section throws `NotConnectedException` while disconnected,
including the `MaxMove` properties, which may come from the device.

### DeviceState

`IsMoving`, `TimeStamp`, and, when `CanReportPosition` is true,
`AzimuthPosition` and `AltitudePosition`.

The usual rules apply: the list is empty while disconnected, and a member whose
getter would throw is left out. So after a failed move `IsMoving` is absent
from `DeviceState`; a client that does not find it there reads the `IsMoving`
property, which reports the failure.

## Behaviour of the async operations

These follow the existing Platform 7 rules for asynchronous operations.

- An initiator validates everything it can before returning, in this order.
  It throws instead of returning when it already knows the move cannot
  complete:
  - `InvalidValueException` if the amount is outside
    ±`AzimuthMaxMove` / ±`AltitudeMaxMove`, or not a finite number (checked
    even while disconnected);
  - `NotConnectedException` if not connected;
  - `InvalidOperationException` if a move is already in progress (the client
    calls `Halt` first), or if the device is not ready to move, for example
    because it has not been calibrated; the message says which. A refused
    call leaves the running move untouched.
- An initiator returns as soon as the move has started, never after it has
  finished. `IsMoving` is already true when the initiator returns, so a client
  that polls at once never sees false before the move has begun. The driver
  absorbs any motor start-up delay.
- A zero amount is valid and completes immediately.
- A failure discovered during the move (stall, lost link, end of travel) is
  reported by `IsMoving` throwing `DriverException` on every read until the
  next initiator call (a zero move will do), `Halt`, `Connect` or
  `Disconnect`. The message names the axis. The position properties do not
  throw for this; they throw only when the device itself can no longer answer.
- A driver that waits for the device to confirm a move treats no confirmation
  within its own timeout as a failure, not as arrival.
- After `Halt`, `IsMoving` returns false without an exception. The move is
  cancelled, not failed.

## Alpaca endpoints

Device type in the URL: `polaralignmentcorrector`.

| Verb | Path | Parameters |
|---|---|---|
| PUT | `/api/v1/polaralignmentcorrector/{device_number}/moveazimuth` | `Degrees` |
| PUT | `/api/v1/polaralignmentcorrector/{device_number}/movealtitude` | `Degrees` |
| PUT | `/api/v1/polaralignmentcorrector/{device_number}/move` | `AzimuthDegrees`, `AltitudeDegrees` |
| PUT | `/api/v1/polaralignmentcorrector/{device_number}/halt` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/ismoving` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/canmoveboth` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/azimuthmaxmove` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/altitudemaxmove` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/canreportposition` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/azimuthposition` | |
| GET | `/api/v1/polaralignmentcorrector/{device_number}/altitudeposition` | |

Error numbers, all returned with HTTP 200 and the number in `ErrorNumber`, as
for every Alpaca device:

| Number | Error | Used for |
|---|---|---|
| 0x400 | NotImplemented | `move` when `CanMoveBoth` is false; positions when `CanReportPosition` is false |
| 0x401 | InvalidValue | amount out of range or not finite |
| 0x407 | NotConnected | any member other than `canmoveboth` and `canreportposition` while disconnected |
| 0x40B | InvalidOperation | move in progress, or device not ready; never NotImplemented for this |
| 0x500-0xFFF | driver-specific | failure during a move, read from `ismoving` |

## Mapping to INDI PACInterface

| INDI `PACInterface` | This draft |
|---|---|
| `MoveAZ(deg)` | `MoveAzimuth(Degrees)` |
| `MoveALT(deg)` | `MoveAltitude(Degrees)` |
| `MoveBoth(az, alt)` | `Move(AzimuthDegrees, AltitudeDegrees)` |
| `AbortMotion()` | `Halt()` |
| property state BUSY / OK / ALERT | `IsMoving` true / false / throws |
| `PAC_HAS_POSITION`, `PAC_POSITION` | `CanReportPosition`, `AzimuthPosition`, `AltitudePosition` |
| `PAC_HAS_SPEED`, `PAC_CAN_REVERSE`, `PAC_HAS_BACKLASH` | driver setup, not in the interface |
| `PAC_CAN_HOME`, `PAC_CAN_SYNC` | not included in V1 |

One deliberate difference: INDI defines "ALT + = North". This draft defines
positive altitude as raising the polar axis, which is the same thing in the
northern hemisphere and removes the ambiguity in the southern one.

## Open questions

1. **Name.** `PolarAlignmentCorrector` is long but cannot be confused with a
   software polar alignment routine. `PolarAligner` is shorter but reads like
   the routine. Other suggestions welcome.
2. **Should `Move` be mandatory?** A driver without simultaneous motion could
   run azimuth then altitude inside one operation, which is INDI's default.
   That would remove `CanMoveBoth` and make clients simpler. My lean is yes.
   INDI's `MoveBoth` default already runs azimuth then altitude in the base
   class, and every optional member costs a `Can*` flag, a conformance case
   and a client branch. (INDI has no `MaxMove`; its manual step is ±10°, so a
   bridge to an INDI device would default `MaxMove` to 10.)
3. **Home and sync.** Left out of V1 because a polar alignment client does not
   need them. Worth adding now, or later as V2? Whatever the answer, homing
   does not belong in `Connect` (see Members). A `Sync` would be a driver-side
   offset, like Rotator `Position` against `MechanicalPosition`, and fits V2.
4. **Conformance.** ConformU would need a small travel budget per test so it
   does not run a corrector to the end of its travel, unlike the Rotator and
   Focuser suites above. A shape that works: each move uses a small fraction
   of `MaxMove`, moves are paired +x then −x so net travel is zero, no test
   exceeds `MaxMove`, and `Halt` is tested on a move of that size.

## A working reference

I am building an Alpaca driver for OAPA (open hardware and firmware), aimed
at OpenAstro AlpacaBridge, which ships only the standard device types. Until
an interface exists it exposes this behaviour through a Switch device plus
custom `Action`s whose names match the members above and are listed in
`SupportedActions`, so a client can recognise a corrector; any other name
throws `ActionNotImplementedException`. It can serve as a test device while
the interface is discussed, and can move to the real interface with no change
in behaviour.
