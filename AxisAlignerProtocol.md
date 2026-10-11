# Axis Aligner serial protocol — draft proposal

Status: draft 1, for discussion. Authors: Michele Bergo (OAPA), from a command
mapping proposed by Lutz (OpenAstroTech). Nothing here is final.

This is **not** part of the ASCOM interface. [IAxisAlignerV1](IAxisAlignerV1.md)
defines what a client sees; ASCOM never defines what a driver sends to its
device. This document proposes a common wire protocol for aligner devices, so
that one ASCOM/Alpaca (or INDI) driver can talk to any device that speaks it,
instead of one driver per vendor.

A device may speak this protocol alongside its own. The framing below is chosen
so that a line of this protocol can never be mistaken for a line of another
one the same device already speaks.

## Transport

- Serial over USB or UART, 115200 baud, 8N1, unless the device documents
  otherwise. TCP is allowed with the same framing; the port is device-defined.
- The host sends one command and waits for its reply before sending the next.
- The device replies to every command within 1 s. A move does not keep the
  reply waiting: the reply comes when the move has started (see Moves).

## Framing

LX200-style: a command starts with `:` and ends with `#`; a reply ends with
`#`. Every command of this protocol starts with `:AX`, the namespace.

```
host:   :AXM0.083333,0,0#
device: OK#
```

- Characters are ASCII. No spaces. CR and LF outside a frame are ignored.
- **Numbers** are decimal degrees with `.` as the decimal separator, an
  optional sign, and up to 6 decimals (`-0.0833`, `1.5`, `0`). Booleans are
  `0` or `1`.
- **Lists** are comma-separated, in a fixed order. A field the device cannot
  give is `-`. A host ignores fields it does not know at the end of a list, so
  later versions can append fields without breaking older hosts.
- **Replies**: `OK#` for an accepted command without data; the data itself for
  a query; or an error.
- **Errors**: `E<code>:<text>#`, where `<code>` is the ASCOM/Alpaca error number
  in hex and `<text>` says what went wrong for the user. A driver throws the
  matching exception with `<text>` as its message.

| Code | ASCOM exception | Used for |
|---|---|---|
| `E400` | `MethodNotImplementedException` | unknown `:AX` command, or an optional command the device does not implement |
| `E401` | `InvalidValueException` | amount not a number, outside `MaxMove`, non-zero on a missing axis, or past the known travel |
| `E40B` | `InvalidOperationException` | a move is already running, or the device is not ready (for example not calibrated) |
| `E500`–`EFFF` | `DriverException` | device-specific failures, for example a stall or end of travel during a move |

Example: `E401:X move 3.5 exceeds XMaxMove 2#`.

## Commands

| Command | Reply | Interface member |
|---|---|---|
| `:AXV#` | `<version>#` | protocol version, `1` for this document |
| `:AXI#` | `<maker>,<model>,<firmware>#` | `Description`, `DriverInfo` |
| `:AXC#` | `<x>,<y>,<z>,<absolute>,<home>,<sync>,<application>#` | `CanMoveX`, `CanMoveY`, `CanMoveZ`, `CanMoveAbsolute`, `CanFindHome`, `CanSync` (booleans), `Application` (number) |
| `:AXL#` | `<x>,<y>,<z>#` | `XMaxMove`, `YMaxMove`, `ZMaxMove`; `0` for a missing axis |
| `:AXR#` | `<xmin>,<xmax>,<ymin>,<ymax>,<zmin>,<zmax>#` | travel range per axis; `-` where unknown |
| `:AXP#` | `<x>,<y>,<z>#` | `XPosition`, `YPosition`, `ZPosition`; `-` where not tracked |
| `:AXS#` | `0#`, `1#` or an error | `IsMoving` |
| `:AXM<x>,<y>,<z>#` | `OK#` or an error | `Move(X, Y, Z)` |
| `:AXA<x>,<y>,<z>#` | `OK#` or an error | `MoveAbsolute(X, Y, Z)`, optional |
| `:AXH#` | `OK#` or an error | `FindHome()`, optional |
| `:AXY<x>,<y>,<z>#` | `OK#` or an error | `Sync(X, Y, Z)`, optional |
| `:AXQ#` | `OK#` | `Halt()` |

The capability, limit and range replies are fixed for a given device and
configuration, so a driver reads them once at connect.

Example of a full move, 5′ up on the altitude axis of a polar aligner:

```
:AXM0,0.083333,0#   ->  OK#
:AXS#               ->  1#
:AXS#               ->  1#
:AXS#               ->  0#
:AXP#               ->  0,0.0834,-#
```

## Moves

The rules are the interface's async rules, carried onto the wire:

- `:AXM`, `:AXA` and `:AXH` validate before they reply. If the move cannot
  complete they reply with an error and nothing moves. A refused command leaves
  a running move untouched.
- `OK#` means the move has started: `:AXS#` already replies `1#`.
- A zero move is valid and completes at once.
- A failure during the move stops the axis. From then on `:AXS#` replies the
  error (`E5xx:...#`, naming the axis) on every read, until the next `:AXM`,
  `:AXA`, `:AXH` or `:AXQ`. `:AXP#` still reports where the axes stopped.
- `:AXQ#` replies `OK#` once motion has stopped. After it, `:AXS#` replies
  `0#`. Halting while idle is not an error.
- Signs and axis meanings follow the interface: for a device whose application
  is `PolarAxis` (1), positive `X` moves the polar axis east and positive `Y`
  raises it.

## What stays outside

Calibration, backlash compensation, speed, motor currents and direction
reversal are device setup. They stay in the device's own commands or its setup
page, outside the `:AX` namespace, as they stay outside the interface.

A device converts degrees to its own steps. How it knows its scale (gear
ratio, measured calibration) is its own business.

## Coexistence with existing protocols

- **OAPA** speaks newline-terminated, GRBL-flavoured commands (`?`, `!`,
  `$H`, `$G=`, `$M=`, `SX<n>` ...). None of them starts with `:`, so an OAPA
  board can accept both. OAPA's own alignment (`$E` readings from TPPA, the
  board deciding every move) stays as it is; the `:AX` moves use the factors
  the board already calibrates.
- **LX200 / Meade** commands share the `:...#` framing. The `:AX` namespace has
  to stay clear of the Meade set and of each vendor's extensions; to be
  checked against the OpenAstroTech command set before this is fixed.

## Open questions

1. **Namespace.** Is `:AX` free in the OpenAstroTech firmware and the Meade
   command set it extends? Any vendor with a clash should say so now.
2. **One capability line or one query per member.** This draft returns all
   capabilities in one reply (`:AXC#`) to keep the connect short; a query per
   member is easier to read. Which do device makers prefer?
3. **Status with positions.** Should `:AXS#` also carry the positions, so that
   a driver building `DeviceState` needs one round trip instead of two?
4. **Discovery over USB.** Should `:AXV#` be the probe a driver sends to find
   an aligner among serial ports, and what should a device that does not speak
   this protocol be expected to do with it?

## Revision history

- **Draft 1.** First version, from Lutz's mapping of each interface member to a
  command. Moved out of the interface document; LX200-style framing in its own
  namespace so it cannot collide with OAPA or Meade commands; replies, errors,
  number format and protocol version added.
