# ASCOM Aligner — the road to a standard interface

This repository started as a draft proposal for a new ASCOM device interface for
motorised devices that adjust an orientation by small angles on up to three axes,
`IAxisAlignerV1`, with polar alignment correctors (Avalon UPAS, MLAstro RPA,
OpenAstroTech AutoPA, Zenit Align Mini, EZT Astro MPA, OAPA and similar) as the
motivating use case.

The ASCOM Initiative has since taken the interface up. Its definition is now
maintained by the Initiative, and this repository no longer carries one of its own:

- API definition, `IAlignerV1`: <https://ascom-standards.org/ialigner>
- Alpaca endpoints, device type `aligner`: <https://ascom-standards.org/apialigner>

Drafts 1 to 5 of the proposal that preceded it, with their revision history, remain
in this repository's history: the last one is
[IAlignerV1.md at draft 5](https://github.com/michelebergo/ascom-polar-alignment-corrector/blob/a42f355/IAlignerV1.md).

## What stays here

OpenAstroTech has drafted PolarAlign Actions, the same contract carried over
ASCOM `Action()` so it works on existing device types today, before clients and
drivers support the dedicated type. A field-by-field comparison between the
actions and the interface, and the points to settle for the transition:
[PolarAlignActionsComparison.md](PolarAlignActionsComparison.md).

A companion draft proposes a common serial protocol for these devices, so that
one driver can talk to any of them: [AxisAlignerProtocol.md](AxisAlignerProtocol.md).
It is not part of the ASCOM interface, which defines what a client sees and never
what a driver sends to its device.

A working device: the OAPA Alpaca server (<https://github.com/michelebergo/oapa-alpaca>)
serves an OAPA board as an `aligner` device after the Initiative's draft, next to the
interim PolarAlign actions on a Switch device.

## Feedback

The interface is discussed with the ASCOM Initiative and on ASCOM-Talk Developer
(thread #9611, "I need ASCOM standard for Automatic Polar Alignment"). Comments on the
comparison and on the serial protocol are welcome as issues or pull requests here.

## License

[CC BY 4.0](LICENSE). Anyone, including the ASCOM Initiative, may copy and
adapt this text with attribution.
