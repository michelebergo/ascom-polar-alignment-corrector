# IAlignerV1 — draft proposal

Status: draft 5, aligned with the ASCOM Initiative's draft 1 of the same interface
(Peter Simpson, 15 November 2026: <https://ascom-standards.org/ialigner>, Alpaca
definition <https://ascom-standards.org/apialigner>). Drafts 1 to 4 were discussed on
ASCOM-Talk Developer (thread #9611, "I need ASCOM standard for Automatic Polar
Alignment") under the name `IAxisAligner`. Author: Michele Bergo (OAPA). Contributions:
Joey Troy and Diego Nascimento (OpenAstro), from AlpacaBridge driver and ConformU
experience and from the MLAstro RPA; Peter Simpson (ASCOM Initiative); Lutz
(OpenAstroTech). Nothing here is final; names, members and semantics are open to
change.

Where this draft and the ASCOM draft 1 differ, the difference is marked **(open)** and
listed under "Open questions": the sign rule of the polar alignment profile, the
reporting of a failed move, and whether `Halt` is optional.

## What the device is

An aligner is a motorised adjuster that changes the orientation or position of
something it carries by small amounts, typically a few arcminutes to a few
degrees, or a few millimetres, on up to three axes. It does not track, slew or
know where the sky is. A client measures the misalignment (for example by plate
solving), commands the aligner, waits for the move to finish, and measures again
until the error is below its threshold.

The use case that motivates the interface is **polar alignment**: a motorised
base or wedge under an equatorial mount that moves the mount's polar axis in
azimuth and altitude. Examples on the market or in active development: Avalon
UPAS, MLAstro RPA, OpenAstroTech AutoPA, Zenit Align Mini, EZT Astro MPA,
OAPA. The same interface also fits other small alignment jobs, such as aligning
several telescopes relative to each other on one mount, or an optical axis.

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

- **Axes.** Up to three axes, `X`, `Y` and `Z`. Which axes a device has is
  reported by `CanMoveX`, `CanMoveY` and `CanMoveZ` in its configuration. What
  each axis means is fixed by a profile (below) or, outside a profile,
  documented by the driver.
- **Units.** The device states the unit of every move, position and travel
  value in `Configuration.Units`: `Degrees`, `Millimeters` or `Counts`
  (`AlignerUnits`). A polar aligner reports `Degrees`, as elsewhere in ASCOM: a
  client takes the error it measured and sends it as it is, to any polar
  aligner, without knowing the device's scale; the driver converts to its own
  steps. `Counts` is for devices that know nothing but motor steps; a client of
  such a device has to learn its scale itself.
- **Relative moves are the common capability.** Most aligners have no absolute
  encoder, so every device implements relative moves. Devices that know their
  position can also implement absolute moves.
- **Accuracy.** A move is a best-effort request. Backlash, gearing and load mean
  an axis may not land exactly on the requested amount. The driver applies
  whatever compensation it has (backlash, calibration); the client is expected
  to re-measure after each move. The interface does not promise a landing
  accuracy. A position counted from commanded steps includes the driver's
  backlash take-up, so the change in position across a reversal can exceed the
  amount requested.
- **Configuration stays in the driver.** Speed, motor direction reversal,
  backlash compensation and calibration are device setup, handled in the
  driver's setup dialog or web page. A client does not need them to close the
  loop, so they are not interface members.
- **One object for the fixed values.** Everything a client reads once, at
  connect, comes in one `Configuration` object (`IAlignerConfigurationV1`), so
  that a client makes one call instead of eighteen. The common members of every
  device type stay individual calls, as existing code expects.

### Polar alignment profile

A device whose `AlignerPurpose` is `PolarAligner` follows this profile, so that
any polar alignment client can drive any such device. Its `Units` is `Degrees`.
Directions are mechanical, so that a device can execute them without knowing
where on Earth it is: **(open: not yet in the ASCOM draft 1)**

- `X` is azimuth: positive `X` **increases the azimuth** of the polar axis,
  that is, turns it clockwise seen from above. This is the sense of
  `Telescope.Azimuth` in ASCOM and of the azimuth error reported by the polar
  alignment routines in N.I.N.A. and KStars. It is towards east in the
  northern hemisphere and towards west in the southern one;
- `Y` is altitude: positive `Y` **raises** the polar axis (increases its
  elevation above the horizon);
- `Z` is not used: `CanMoveZ` is false.

Example: the client measures the polar axis 5′ too low. It calls
`Move(0, +0.0833, 0)`.

The client knows the site latitude and converts its own error into these
directions; the device never needs a hemisphere setting. Drafts 1 to 3 defined
positive azimuth as "towards east in both hemispheres", which would have
required one.

INDI's `PACInterface` documents "AZ + = East, ALT + = North"; its drivers pass
the azimuth error through to the motors unchanged, so in practice they follow
the azimuth coordinate as this profile does.

## Members

In addition to the common members of every Platform 7 device (`Action`,
`SupportedActions`, `CommandBlind`, `CommandBool`, `CommandString`, `Connect`,
`Disconnect`, `Connected`, `Connecting`, `DeviceState`, `Description`,
`DriverInfo`, `DriverVersion`, `InterfaceVersion` = 1, `Name`).

`Connect` opens the link and reads the device's state. It does not home or
calibrate; clients give up on a device that stays `Connecting` for more than a
few seconds. `Disconnect` while a move is in progress halts the move first.

### Configuration

| Member | Type | Summary |
|---|---|---|
| `Configuration` | `IAlignerConfigurationV1` | One object with every value below. Must be implemented; throws `NotConnectedException` while disconnected, because the values may come from the device. Fixed for a given device and configuration: a client reads it once after connecting. |
| `ConfigurationVersion` | short | The version of the configuration object's interface, 1, independent of `InterfaceVersion`. |

The configuration object (`IAlignerConfigurationV1`):

| Member | Type | Summary |
|---|---|---|
| `AlignerPurpose` | `AlignerPurpose` | What the device aligns: `Unset` (0), `PolarAligner` (1), `OpticalAxisAligner` (2); 3 to 20 reserved. A client looking for a polar aligner picks a device whose value is `PolarAligner`, and can then rely on the polar alignment profile. A device must not report `Unset`; a client treats it as an error. |
| `Units` | `AlignerUnits` | The unit of every move, position and travel value: `Unset` (0), `Degrees` (1), `Millimeters` (2), `Counts` (3). A device must not report `Unset`. |
| `CanMoveX`, `CanMoveY`, `CanMoveZ` | boolean | True if the device has that axis. |
| `CanMoveAbsolute` | boolean | True if `MoveAbsolute` is implemented. |
| `CanFindHome` | boolean | True if `FindHome` is implemented. |
| `CanSync` | boolean | True if `Sync` is implemented. |
| `CanHalt` | boolean | In the ASCOM draft 1: true if `Halt` is implemented. This draft keeps `Halt` mandatory, so the value is always true **(open)**. |
| `XMaxMove`, `YMaxMove`, `ZMaxMove` | double | Largest relative move of that axis (absolute value), in `Units`, that a single `Move` accepts. A limit per call, not the travel left. Not implemented (left out of the Alpaca object) for an axis the device does not have, or when the device does not know it. |
| `XMinPosition`, `XMaxPosition`, `YMinPosition`, `YMaxPosition`, `ZMinPosition`, `ZMaxPosition` | double | Travel of that axis: the lowest and highest position it can reach, in `Units`, on the same scale as the position properties. Not implemented when the driver does not know the travel, or the device does not have that axis; the minimum and the maximum of an axis come together or not at all. |

In Alpaca, `Configuration` answers one JSON object whose members are the names
above; a member that is not implemented is left out.

### Methods

| Member | Kind | Summary |
|---|---|---|
| `Move(X, Y, Z)` | async initiator | Starts a relative move of every axis by the given amounts, in `Units`. A driver that cannot move axes together moves them one after another inside the same operation. Completion: `IsMoving` becomes false. |
| `MoveAbsolute(X, Y, Z)` | async initiator, optional | Starts a move of every axis to the given positions, in `Units`. `MethodNotImplementedException` when `CanMoveAbsolute` is false. Completion: `IsMoving` becomes false. |
| `FindHome()` | async initiator, optional | Moves every axis to its home position. `MethodNotImplementedException` when `CanFindHome` is false. Completion: `IsMoving` becomes false. |
| `Sync(X, Y, Z)` | synchronous, optional | Tells the driver that the axes are now at the given positions, without moving them. `MethodNotImplementedException` when `CanSync` is false. |
| `Halt()` | synchronous | Stops motion on every axis. Returns when motion has stopped; on return `IsMoving` is false. Calling it while idle does nothing and does not throw. Mandatory in this draft: a client that drives a motor near a mount needs a stop it can rely on, and a device whose moves are too short to stop simply returns **(open)**. |

For an axis whose `CanMove` value is false, the amount passed to `Move`,
`MoveAbsolute` or `Sync` must be 0.

### Properties

| Member | Type | Summary |
|---|---|---|
| `IsMoving` | boolean | True while any move or homing is in progress. The completion property for `Move`, `MoveAbsolute` and `FindHome`. |
| `XPosition`, `YPosition`, `ZPosition` | double | Position of that axis from the driver's zero, in `Units`. `PropertyNotImplementedException` when the device does not track its position, or does not have that axis. |

A device that implements `MoveAbsolute` or `Sync` must also implement the
position properties of the axes it has. A device that reports the travel of an
axis must also implement its position property, so a client can tell how far
it is from either end.

The driver chooses where zero is (power-on, a home switch, a user reset),
documents it, and says whether it survives a disconnect or power cycle. After
a successful `FindHome` the positions read the driver's home values. Positions
may be counted from commanded steps rather than measured; the interface does
not promise a measurement. During a move they return the latest value the
driver has. Positions use the same signs as moves.

Every member in this section, and `Configuration`, throws
`NotConnectedException` while disconnected.

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
    knows the travel, that is outside the range given by the travel values;
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
  or `Disconnect`. The message names the axis and the reason. After an end of
  travel the position properties report where the axis stopped. They do not
  throw for this; they throw only when the device itself can no longer answer.
  **(open: the ASCOM draft 1 has no way to report a failed move; see the open
  questions for the alternative of an explicit member.)**
- A driver that waits for the device to confirm a move treats no confirmation
  within its own timeout as a failure, not as arrival.
- After `Halt`, `IsMoving` returns false without an exception. The move is
  cancelled, not failed.

## Alpaca endpoints

Device type in the URL: `aligner`, as in the ASCOM Alpaca definition.

| Verb | Path | Parameters |
|---|---|---|
| GET | `/api/v1/aligner/{device_number}/configuration` | |
| GET | `/api/v1/aligner/{device_number}/configurationversion` | |
| PUT | `/api/v1/aligner/{device_number}/move` | `X`, `Y`, `Z` |
| PUT | `/api/v1/aligner/{device_number}/moveabsolute` | `X`, `Y`, `Z` |
| PUT | `/api/v1/aligner/{device_number}/findhome` | |
| PUT | `/api/v1/aligner/{device_number}/sync` | `X`, `Y`, `Z` |
| PUT | `/api/v1/aligner/{device_number}/halt` | |
| GET | `/api/v1/aligner/{device_number}/ismoving` | |
| GET | `/api/v1/aligner/{device_number}/xposition`, `yposition`, `zposition` | |

`X`, `Y` and `Z` are all required on `move`, `moveabsolute` and `sync`; a
missing one is an `InvalidValue`.

Error numbers, all returned with HTTP 200 and the number in `ErrorNumber`, as
for every Alpaca device:

| Number | Error | Used for |
|---|---|---|
| 0x400 | NotImplemented | `moveabsolute`, `findhome`, `sync` when their `Can*` is false; positions the device does not track or axes it does not have |
| 0x401 | InvalidValue | amount out of range, not finite, missing, non-zero on a missing axis, or past the known travel |
| 0x407 | NotConnected | `configuration`, the positions, `ismoving` and the methods while disconnected |
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
would report 10. INDI has no travel range and no purpose property; a bridge
leaves the travel out and reports `AlignerPurpose` as `PolarAligner`.

## Conformance

ConformU needs a small travel budget per test, so that it does not run an
aligner to the end of its travel, unlike the Rotator and Focuser suites above:

- each relative move uses a small fraction of `MaxMove`, and moves are paired
  +x then −x so net travel is zero; no test exceeds `MaxMove`;
- `Halt` is tested on a move of that size;
- for a device that reports its travel, absolute moves stay within the reported
  minimum and maximum, and a target just outside them must throw
  `InvalidValueException` without moving;
- `FindHome` is tested only when the device documents that homing is safe to
  run unattended.

## Open questions

Against the ASCOM draft 1, which this draft otherwise follows:

1. **The sign rule.** The ASCOM draft 1 does not say which way a positive `X`
   or `Y` turns a polar aligner. The polar alignment profile above was settled
   with OpenAstro in November 2026 after checking what N.I.N.A.'s TPPA, Ekos,
   the OpenAstroTech driver, the MLAstro plugin and the INDI drivers do today;
   it belongs under the `PolarAligner` value of `AlignerPurpose`, so that it is
   part of the definition. Without it a device and a client written
   independently can turn opposite ways in the southern hemisphere.
2. **A failed move.** The ASCOM draft 1 has `IsMoving` as the only completion
   property, so a client cannot tell a finished move from one that stopped half
   way (a hard limit on the MLAstro RPA, a stall or a timeout on OAPA). This
   draft reports it through `IsMoving` throwing, as above, with no new member.
   An explicit member (a `MoveFault` string, empty when the last move
   completed) would do as well, as long as the client has one place to look.
3. **`Halt` optional.** The ASCOM draft 1 has `CanHalt`. This draft keeps
   `Halt` mandatory, for the reason given in the methods table.
4. **A purpose for other uses.** Drafts 3 and 4 had an `Other` value for an
   aligner that is neither polar nor optical. The ASCOM draft 1 has `Unset`,
   which a client treats as an error, and reserved values; a device of another
   kind has no value to report until one is assigned.

Settled since draft 4: the name (`IAligner`), the units (an enumeration, with
degrees for polar aligners), and the configuration object.

## A working reference

The OAPA Alpaca server (<https://github.com/michelebergo/oapa-alpaca>) serves an
OAPA board (open hardware and firmware) as an `aligner` device after this draft,
at `/api/v1/aligner/0`, next to the interim PolarAlign actions on a Switch
device that N.I.N.A.'s Polar Alignment Correctors plugin drives today. It
reports `AlignerPurpose` 1, `Units` 1, relative moves in degrees on `X` and `Y`,
`Halt` and `FindHome`, positions when the board is calibrated, and a failed move
through `IsMoving`. It runs on Windows, Linux and macOS, including a Raspberry
Pi with the board on USB, and can serve as a test device while the interface is
discussed; the C++ driver for OpenAstro AlpacaBridge will follow the same
contract.

## Revision history

- **Draft 5.** Aligned with the ASCOM Initiative's draft 1 of `IAlignerV1`
  (Peter Simpson, 15 November 2026): renamed from `IAxisAligner`; the fixed
  values move into one `Configuration` object with `ConfigurationVersion`;
  `Application` becomes `AlignerPurpose` with the ASCOM values (`Unset`,
  `PolarAligner`, `OpticalAxisAligner`, reserved); the unit is stated by the
  device in `Units` (`AlignerUnits`: degrees, millimetres or counts), degrees
  for polar aligners; Alpaca device type `aligner` with the ASCOM endpoints;
  `CanHalt` noted. The polar alignment profile, the reporting of a failed move
  and a mandatory `Halt` are kept and marked as open against the ASCOM draft.
  Backlash take-up noted under accuracy. A working reference exists: the OAPA
  Alpaca server.
- **Draft 4.** Azimuth sign of the polar alignment profile changed to the
  azimuth coordinate (positive turns the polar axis clockwise seen from above),
  the convention of OpenAstroTech's PolarAlign Actions, ASCOM's
  `Telescope.Azimuth` and the existing polar alignment clients. A device can
  execute it without a hemisphere setting; the client, which knows the site,
  converts. Settles point D1 of `PolarAlignActionsComparison.md`. (Diego
  Nascimento, OpenAstro; Lutz, OpenAstroTech.)
- **Draft 3.** `Application` property (`Other`, `PolarAxis`, `OpticalAxis`);
  the polar alignment profile applies to `PolarAxis` devices. Optional travel
  range per axis (`XMinPosition` ... `ZMaxPosition`), in degrees, used for the
  end-of-travel check. Conformance moved out of the open questions, with
  absolute moves kept inside the reported travel. Units stated for positions and
  range. (Feedback from Lutz, OpenAstroTech, in thread #9611.)
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
