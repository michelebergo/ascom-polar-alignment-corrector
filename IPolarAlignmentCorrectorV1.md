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
  stopgap. But clients cannot tell a corrector from any other switch, the
  direction and units are a naming convention rather than a contract, and a
  move is a state to set rather than an operation to start and wait for.
- **Focuser** has a limited range, but it counts steps rather than angles, and
  clients offer it for autofocus.
- **Rotator** uses degrees, but assumes a full 360° ring. A corrector with a few
  degrees of travel cannot pass the Rotator conformance tests, and clients
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

### Methods

| Member | Kind | Summary |
|---|---|---|
| `MoveAzimuth(Degrees)` | async initiator | Starts a relative azimuth move of the polar axis. Completion: `IsMoving` becomes false. |
| `MoveAltitude(Degrees)` | async initiator | Starts a relative altitude move of the polar axis. Completion: `IsMoving` becomes false. |
| `Move(AzimuthDegrees, AltitudeDegrees)` | async initiator, optional | Starts both moves as one operation. Throws `MethodNotImplementedException` when `CanMoveBoth` is false. |
| `Halt()` | synchronous | Stops all motion immediately. On return `IsMoving` is false. |

### Properties

| Member | Type | Summary |
|---|---|---|
| `IsMoving` | boolean | True while any move is in progress. The completion property for all three move methods. |
| `CanMoveBoth` | boolean | True if `Move` is implemented. |
| `AzimuthMaxMove` | double | Largest azimuth move, in degrees (absolute value), that a single call accepts. |
| `AltitudeMaxMove` | double | Largest altitude move, in degrees (absolute value), that a single call accepts. |
| `CanReportPosition` | boolean | True if the device tracks its own position. |
| `AzimuthPosition` | double | Azimuth offset of the polar axis from the driver's zero, in degrees. `PropertyNotImplementedException` when `CanReportPosition` is false. |
| `AltitudePosition` | double | Altitude offset of the polar axis from the driver's zero, in degrees. `PropertyNotImplementedException` when `CanReportPosition` is false. |

The driver chooses where zero is (power-on, a home switch, a user reset) and
documents it. Positions use the same signs as moves.

### DeviceState

`IsMoving`, `TimeStamp`, and, when `CanReportPosition` is true,
`AzimuthPosition` and `AltitudePosition`.

## Behaviour of the async operations

These follow the existing Platform 7 rules for asynchronous operations.

- An initiator validates everything it can before returning. It throws
  instead of returning when it already knows the move cannot complete:
  - `InvalidValueException` if the amount is outside
    ±`AzimuthMaxMove` / ±`AltitudeMaxMove`, or not a finite number;
  - `InvalidOperationException` if a move is already in progress (the client
    calls `Halt` first), or if the device is not ready to move, for example
    because it has not been calibrated; the message says which;
  - `NotConnectedException` if not connected.
- A zero amount is valid and completes immediately.
- A failure discovered during the move (stall, lost link, end of travel) is
  reported by `IsMoving` throwing `DriverException`, and it keeps throwing
  until the next move starts.
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
   That would remove `CanMoveBoth` and make clients simpler.
3. **Home and sync.** Left out of V1 because a polar alignment client does not
   need them. Worth adding now, or later as V2?
4. **Conformance.** ConformU would need a small travel budget per test so it
   does not run a corrector to the end of its travel. The `MaxMove`
   properties give it the bound.

## A working reference

I am building an Alpaca driver for OAPA (open hardware and firmware), aimed
at OpenAstro AlpacaBridge. Until an interface exists it exposes this behaviour
through a Switch device plus custom `Action`s whose names match the members
above. It can serve as a test device while the interface is discussed, and
can move to the real interface with no change in behaviour.
