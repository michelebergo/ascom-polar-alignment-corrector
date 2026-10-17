# PolarAlign Actions and IAxisAligner — comparison

Status: working notes, 2026-10-17, for discussion between the authors of the
two drafts. Compares:

- **PolarAlign Actions**, draft v0.1 (2026-10-14), OpenAstroTech:
  <https://wiki.openastrotech.com/en/Development/pa-actions>
- **IAxisAlignerV1**, draft 3, and the **Axis Aligner serial protocol**, draft 1,
  in this repository: [IAxisAlignerV1.md](IAxisAlignerV1.md),
  [AxisAlignerProtocol.md](AxisAlignerProtocol.md)

Both drafts describe how a host such as N.I.N.A.'s Three Point Polar Alignment
drives a motorised polar axis, and they overlap almost entirely. This page lines
them up field by field, so that the differences can be settled and we end up
with one standard.

## Three layers

The drafts mostly sit at different layers. They collide only in the middle one.

| Layer | IAxisAligner | PolarAlign Actions |
|---|---|---|
| **1. ASCOM device type** (long term) | A new interface, `axisaligner` in Alpaca, proposed in ASCOM-Talk thread #9611. | Not defined. Its section 8 expects a dedicated type later, once the actions have proven themselves. |
| **2. `Action()` on an existing type** (usable today) | Interim plan: custom actions named after the interface members; the carrier type is left to the ASCOM Initiative. | The whole draft: `PolarAlign:Info`, `Move`, `Status`, `Stop` and Tier 2, on Telescope (motors built into a mount) or Switch (separate base). |
| **3. Device wire protocol** | `:AX…#` commands in degrees, one per interface member, so one driver can talk to any device. | Out of scope. OpenAstroTracker maps the actions onto its existing `:MAZ`/`:MAL`/`:GX` commands (its section 7.1). |

## Points to settle

**D1. Sign of azimuth in the southern hemisphere — conflict.**
PolarAlign: positive `az` increases the polar axis azimuth (clockwise seen from
above), so east in the north and **west in the south**, chosen to match TPPA's
azimuth error. IAxisAligner: positive `X` moves the polar axis **east in both
hemispheres**. The two agree in the north and are opposite in the south, so a
device built to one draft turns the wrong way under a host built to the other.
Altitude agrees: positive raises the polar axis in both drafts.

**D2. Units — differs.**
PolarAlign uses arcminutes everywhere (moves, positions, range, `maxMove`,
`resolution`). IAxisAligner uses degrees everywhere, as elsewhere in ASCOM: 5′ up
is `Move(0, +0.0833, 0)`. Either works; the conversion is a factor of 60. One
option is arcminutes in the actions, as TPPA reports them, and degrees in the
ASCOM type, with the conversion stated once.

**D3. Axis names and what the device is — differs.**
PolarAlign has two named axes, `az` and `alt`, for polar alignment only.
IAxisAligner has generic `X`, `Y`, `Z` plus `Application` (`PolarAxis`,
`OpticalAxis`, `Other`); its polar profile maps `X` to azimuth and `Y` to
altitude.

**D4. Shape of a move — differs.**
PolarAlign moves one axis per call (`{"axis":"az","arcmin":-1.25}`); both axes
take two calls, allowed at once only when `simultaneousMoves` is true, and the
reply gives the amount actually commanded after rounding. IAxisAligner moves all
axes in one call, `Move(X, Y, Z)`; a device that cannot move them together moves
them one after the other inside the same operation, and there is no reply value.

**D5. Stopping — differs.**
PolarAlign: `PolarAlign:Stop`, optional (`canStop`), decelerating, returns
promptly and the host polls `Status`; per axis or both; it must not stop
tracking. IAxisAligner: `Halt()`, mandatory and synchronous: it returns when
motion has stopped, and `IsMoving` is then false.

**D6. A failure during a move — differs.**
PolarAlign Tier 1 has no failure channel: `Status` only says moving or not
(Tier 2 has `phase: failed` with a reason). IAxisAligner: a stall, lost link or
end of travel stops the axis, and `IsMoving` throws `DriverException` naming the
axis until the next command, while positions still report where the axis
stopped. Without a failure signal a host cannot tell a stalled move from a
finished one.

**D7. Device-driven alignment (Tier 2) — one draft only.**
PolarAlign has an optional Tier 2: `Begin` (tolerance, maxMove), `Error` (az,
alt, exposure time), `End`, and an `alignment` block in `Status`. IAxisAligner
leaves it out: the client measures and commands every move. OAPA already works
the Tier 2 way in its native protocol (`$E=` readings, `$T=` tolerance, `$L?`
state), so Tier 2 maps onto it directly. Open question: does Tier 2 belong in
the ASCOM type, or only in the actions?

**D8. One set of interim action names — conflict.**
PolarAlign uses `PolarAlign:*`, discovered through `PolarAlign:Info` in
`SupportedActions`. IAxisAligner draft 3 ("A working reference") plans actions
named after the interface members for the OAPA Alpaca driver. Two names for the
same command would split hosts and devices. PolarAlign already has a reference
implementation in the OpenAstroTracker driver and a TPPA fork, so the simplest
path is to keep `PolarAlign:*` and map the interface onto it (see below).

**D9. Fields only one draft has — one draft only.**
Only in PolarAlign: `resolution`, `backlashHandled`, `simultaneousMoves`,
`movesWhileTracking`, `movesWhileSlewing`, `settleSeconds`, the reply to `Move`,
Tier 2. Only in IAxisAligner: `MoveAbsolute`, `Sync`, `Application`, a third
axis, a conformance plan, a mapping to INDI `PACInterface`. Each could take the
other's useful fields: `settleSeconds` and `backlashHandled` help any host, and
`Application` lets a client tell a polar aligner from another kind of aligner.

## Side by side

Status: **conflict** · **differs** · **one only** (one draft only) · **same**

### Finding and identifying the device

| Concept | PolarAlign Actions v0.1 | IAxisAlignerV1 draft 3 | Status |
|---|---|---|---|
| Discovery | `SupportedActions` contains `PolarAlign:Info`, and `specVersion` has a known major version. | Device type `axisaligner`; a polar aligner reports `Application = PolarAxis`. | differs |
| Carrier for a separate base | Switch device (Telescope only for motors built into a mount). | Its own device type; interim carrier left to the Initiative. | differs |
| Versioning | `specVersion` "major.minor"; a minor adds optional fields only. | `InterfaceVersion` = 1; wire protocol `:AXV#` = 1. | differs |
| Capabilities from versions | Never from a device or firmware version. | From the `Can*` properties, never from versions. | same |
| Device name and version | `deviceName`, `deviceVersion`, display only. | Standard `Name`, `DriverInfo`, `DriverVersion`. | same |

### Axes, units, signs

| Concept | PolarAlign Actions v0.1 | IAxisAlignerV1 draft 3 | Status |
|---|---|---|---|
| Axes | `az`, `alt`; `axes` lists only those present. | `X`, `Y`, `Z`; `CanMoveX/Y/Z`. | differs |
| Units | Arcminutes. | Degrees. | differs |
| Positive altitude | Raises the polar axis, both hemispheres. | Raises the polar axis, both hemispheres. | same |
| Positive azimuth | Azimuth coordinate increases: east in the north, west in the south. | Towards east in both hemispheres. | **conflict** |
| Reversal, gearing | Handled by the device. | Handled by the device (driver setup). | same |

### Moving

| Concept | PolarAlign Actions v0.1 | IAxisAlignerV1 draft 3 | Status |
|---|---|---|---|
| Relative move | `PolarAlign:Move`, one axis, mandatory. | `Move(X, Y, Z)`, all axes, mandatory. | differs |
| Largest move | `axes.*.maxMove`, required; hosts split larger corrections. | `XMaxMove` etc., required, per call. | same |
| Smallest move | `axes.*.resolution`; the reply gives the rounded amount. | Not defined. | one only |
| Absolute move | Not defined. | `MoveAbsolute`, optional. | one only |
| Home | `PolarAlign:Home`, optional. | `FindHome()`, optional, `CanFindHome`. | same |
| Sync | Not defined. | `Sync(X, Y, Z)`, optional. | one only |
| Both axes at once | `simultaneousMoves` flag; otherwise refused. | Always one call; the driver sequences if needed. | differs |
| Move while one runs | Refused (`InvalidOperationException`); never queued. | Refused (`InvalidOperationException`). | same |
| Zero move | Accepted, does nothing. | Valid, completes immediately. | same |
| Mount state | `movesWhileTracking`, `movesWhileSlewing`; a parked mount refuses. | Not defined (an aligner knows nothing about the mount). | one only |
| Backlash | `backlashHandled`: if false the host may compensate. | Driver setup; moves are best effort, the client re-measures. | differs |
| Settling | `settleSeconds`, suggested wait. | Not defined. | one only |

### State and stopping

| Concept | PolarAlign Actions v0.1 | IAxisAlignerV1 draft 3 | Status |
|---|---|---|---|
| Moving? | `Status.*.moving`, per axis, includes deceleration and backlash take-up. | `IsMoving`, any axis; already true when the initiator returns. | differs |
| Position | `Status.*.position`, optional, informational only. | `XPosition` etc., optional; required with absolute moves, sync or travel. | differs |
| Travel limits | `axes.*.range` min/max, optional; moves outside refused. | `XMinPosition` … `ZMaxPosition`, optional; moves outside refused. | same |
| Stop | `Stop`, optional, decelerating, asynchronous. | `Halt()`, mandatory, synchronous. | differs |
| Failure during a move | No channel in Tier 1. | `IsMoving` throws `DriverException`. | differs |
| Polling rate | At most 4 per second. | Not defined. | one only |
| Disconnect | Last client gone: stop motion, end Tier 2. | `Disconnect` halts a running move first. | same |

### Errors, alignment loop, testing

| Concept | PolarAlign Actions v0.1 | IAxisAlignerV1 draft 3 | Status |
|---|---|---|---|
| Error reporting | Standard ASCOM exceptions; a refused command changes nothing. | Standard ASCOM exceptions and Alpaca numbers; a refused call leaves a running move untouched. | same |
| Device-driven loop | Tier 2: `Begin`, `Error`, `End`, `alignment` block. | Not defined. | one only |
| Error sign (Tier 2) | Current minus target; the ideal correction is `Move(-az)`, `Move(-alt)`. TPPA's altitude error is negated in the south. | Not applicable. | one only |
| Probe move | Recommended host flow: a small probe per axis to learn direction and scale. | The client re-measures after each move. | same |
| Conformance | Relies on the carrier type's Conform. | Small-travel ConformU plan, paired moves, net travel zero. | one only |
| INDI | Not covered. | Maps onto `PACInterface` (KStars/Ekos). | one only |
| Licence of the text | CC BY 4.0 (proposed). | CC BY 4.0. | same |

## Mapping the interface onto the actions

If `PolarAlign:*` becomes the interim layer and the ASCOM type comes later, a
driver can expose both from the same code. Amounts convert by 60 (arcminutes to
degrees), and azimuth needs the sign of D1 agreed first.

| PolarAlign action | IAxisAligner member | Note |
|---|---|---|
| `PolarAlign:Info` | `CanMoveX/Y`, `XMaxMove`/`YMaxMove`, `X/YMin/MaxPosition`, `Application`, `InterfaceVersion` | Info is read once; the properties answer one at a time. |
| `PolarAlign:Move {axis, arcmin}` | `Move(X, Y, 0)` | One axis per action, all axes per method. |
| `Status.*.moving` | `IsMoving` | Per axis vs any axis. |
| `Status.*.position` | `XPosition`, `YPosition` | Units differ (D2). |
| `PolarAlign:Stop` | `Halt()` | Asynchronous vs synchronous (D5). |
| `PolarAlign:Home` | `FindHome()` | Both optional. |
| — | `MoveAbsolute`, `Sync` | Could become optional actions in a minor version. |
| `Begin`, `Error`, `End`, `Status.alignment` | — | Tier 2; see D7. |

## A possible path to one standard

1. Agree the azimuth sign in the southern hemisphere (D1) in both documents
   before anything ships.
2. Use one set of interim names, `PolarAlign:*`, and drop the member-named
   actions from the IAxisAligner draft (D8).
3. Fix units and axis names once, and state the conversion in both documents
   (D2, D3).
4. Add a failure signal to Tier 1 `Status`, so that a stalled move does not look
   finished (D6).
5. Exchange the useful one-sided fields: `settleSeconds` and `backlashHandled`
   into the interface, `Application` into `Info` (D9).
6. Keep IAxisAligner as the dedicated type to propose to the ASCOM Initiative,
   built so that the actions map onto it one to one.

Comments and corrections are welcome as issues or pull requests here, or on the
OpenAstroTech Discord.
