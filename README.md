# MPC Linux notes

Unofficial notes on how Akai MPC standalone hardware works under the hood: the Linux system, the MPC app, audio, the controller's MIDI, LEDs, the display, storage and USB. The aim is that nobody has to repeat the testing to find these facts.

**[Read the MPC One+ notes →](mpc-one-plus.md)**

- **Part 1, How the stock MPC works:** what runs where, in plain terms
- **Part 2, Tech specs:** device paths, audio formats and measured latency, the full controller MIDI and LED maps, display and touch details, USB gadget support, kernel features, performance headroom
- **Part 3, Lessons learned:** how to work on the device without breaking it, and what worked (and didn't) for audio, MIDI, drawing and building programs
- **Part 4, Open questions:** what nobody has measured yet

So far everything was measured on an **MPC One+ on firmware 2.14**, or read in inMusic's own open-source release. Other first-generation MPCs share the same platform, so much of it probably carries over, but per-board details need checking.

## Status tags

Every fact is tagged **[measured]** (seen on a real unit), **[source]** (read in inMusic's or the Linux kernel's source), **[reported]** (published by another project, credited in the notes, not yet confirmed on the One+) or **[unverified]** (expected, not yet checked).

## Contributing

Corrections and measurements from other units are very welcome, especially from other models (One, Live, Live II, X, Key 61, Force). Open an issue or a pull request with:

- the model and MPC firmware version
- what you measured and how (the command or tool), so others can repeat it
- the tag it deserves

A model with enough differences gets its own file next to `mpc-one-plus.md`.

## Where this came from

The testing was done while building [MPDJ](https://github.com/longcutter/mpdj), a tracker that runs natively on the MPC One+ in place of the MPC app. MPDJ's [stand-alone probes](https://github.com/longcutter/mpdj/tree/mpc-port/platforms/mpc/source) (audio round trip, ALSA controls, LED mapper) are the tools behind several of the numbers here. Shell access came from TheKikGen's [MPC-LiveXplore](https://github.com/TheKikGen/MPC-LiveXplore) project, whose wiki and libraries also supplied several [reported] facts; kernel and driver details from inMusic's [open-source release](https://github.com/inmusic-software/az0x-open-source). Full credits are at the end of [the notes](mpc-one-plus.md#credits-and-related-work).

## Disclaimer

Not affiliated with, endorsed by or supported by Akai Professional or inMusic. MPC and Akai Professional are their trademarks. Opening a shell on your MPC, stopping its app or changing its system is at your own risk and may void your warranty. These notes may be wrong or out of date for your unit or firmware.

## Licence

Written by longcutter. Licensed under [Creative Commons Attribution 4.0 International](LICENSE) (CC BY 4.0): copy, adapt and share it, including commercially, as long as you give credit and link back.
