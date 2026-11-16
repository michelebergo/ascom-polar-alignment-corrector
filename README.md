# ASCOM Axis Aligner — interface proposal

A draft proposal for a new ASCOM device interface, `IAlignerV1` (`IAxisAlignerV1` up to draft 4), for
motorised devices that adjust an orientation by small angles on up to three
axes. The motivating use case is polar alignment correctors (Avalon UPAS,
MLAstro RPA, OpenAstroTech AutoPA, Zenit Align Mini, EZT Astro MPA, OAPA and
similar), covered by a polar alignment profile.

The draft is here: [IAlignerV1.md](IAlignerV1.md). Since draft 5 it follows the ASCOM
Initiative's own draft 1 of the interface, <https://ascom-standards.org/ialigner>, and
lists where the two still differ.

A companion draft proposes a common serial protocol for these devices, so
that one driver can talk to any of them:
[AxisAlignerProtocol.md](AxisAlignerProtocol.md). It is not part of the ASCOM
interface.

OpenAstroTech has drafted PolarAlign Actions, the same contract carried over
ASCOM `Action()` so it works on existing device types today. A field-by-field
comparison with this draft, and the points to settle for one standard:
[PolarAlignActionsComparison.md](PolarAlignActionsComparison.md).

## Status

Draft 5, aligned with the ASCOM Initiative's draft 1 (15 November 2026). Nothing is final:
names, members and semantics are open to change. It is not an ASCOM
Initiative document and has not been endorsed by the Initiative.

## Feedback

Discussion happens on ASCOM-Talk Developer, in thread #9611 ("I need ASCOM
standard for Automatic Polar Alignment"). Comments, corrections and
suggestions from device makers and client developers are also welcome as
issues or pull requests here.

## License

[CC BY 4.0](LICENSE). Anyone, including the ASCOM Initiative, may copy and
adapt this text with attribution.
