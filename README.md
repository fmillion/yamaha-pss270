# Yamaha PSS-270

This repo contains my work on reverse-engineering, hacking, modifying and upgrading the Yamaha PSS-270 portable electronic "toy" keyboard.

My goal is to outfit the device with various modern features, including but not limited to:

* MIDI in and out capability, with ability to control all functions of the device
* USB sound output capability (a USB sound device that appears as a "mic" to a PC)
* USB power and Rechargeable Li-Ion battery power
* Onboard MIDI playback (SD card, USB, etc.)
* Direct access to the OPL chip (e.g. use the keyboard as an external OPL synth, playback OPL dumps, etc)
* And more...

This repository will archive documentation and code that I find and write to further this effort.

## Research

The Yamaha PSS-270 hardware consists of the following:

* HD6301 main CPU (CMOS, Motorola MC6801 compatible)
  * Actual part number is `HD6301YB61P/XC194001`. The closest data sheet
    seems to be [HD6301Y0], a 64-pin (for the PDIP version), 16K ROM, 256b
    RAM model otherwise compatible with the 40-pin HD6301V series. The
    `B61` and second number may identify the custom ROM mask.
  * Probably 16 KB ROM. It may be possible to dump this; see below.
* [YM2413](datasheets/YM2413.pdf) "[OPLL](https://en.wikipedia.org/wiki/Yamaha_YM2413)" synthesizer chip, based on OPL2 with cost-reducing modifications
* Other ICs in the device according to the schematic:
  * [LB1214](datasheets/LB1214.pdf) - Sanyo transistor array - drives LED display
  * [MN3206](datasteet/MN3206.pdf) - 128-stage low noise operation [BBD](https://en.wikipedia.org/wiki/Bucket-brigade_device) - used for stereo phasing effect? 
  * [MN3102](datasheet/MN3102.pdf) - Clock chip for MN3206 - same as above
  * [NJM4558DV](datasheet/NJM4558D.pdf) - dual op-amp - used in filtering stages
  * [TA7232P](datasheet/TA7232P.pdf) - Power amplifier

Main system TTL voltage is 5V. Since input voltage comes from 9V-12V of source power (either from DC input jack or batteries) there is a 5V regulator onboard. We could bypass this regulator for a direct 5V power feed from USB.


Dumping the ROM
---------------

### Address Space Mappings

The traditional MC6801 and compatible HD6301V1 support seven "modes" with
different mappings of internal ROM and external memory; mappings that allow
addressing external memory repurpose some of the I/O ports as data and
address buses. The mode is chosen by setting certain logic levels on
specific pins at startup.

The [HD6301Y0] officially supports only three of the seven modes, selected
via dedicated `MP₀` and `MP₁` pins. The mode numbering does not directly
correspond to the modes of the MC6801/HD6301V1.

- Mode 3: Single-chip Mode (6801 mode 7). This maps only internal RAM and
  ROM, and all I/O ports are available. The PSS-270 is hardwired to this
  mode via pullups, according to the schematic.
- Mode 1: Expanded mode, on-chip ROM disabled (similar to 6801 mode 1 or
  5).
- Mode 2: Expanded mode, on-chip ROM enabled (6801 mode 6).

Notably missing from the documentation is "mode 0" (`MP₀` and `MP₁` both
low). On the MC6801/HD6301V1 this is a test mode that enables the internal
ROM but, "Addresses $FFFE and $FFFF [the reset vector] are considered
external if accessed within 3 or 4 cycles after a positive edge of `R̅E̅S̅`
and internal at all other times." This allows one to start executing code
in external memory after a reset but have the internal ROM available to
that code; projects like [6801reader] use this to dump ROMs from 6801 MCUs.

### Dumping the ROM

Thus, there seem to be two possiblities for dumping the ROM.
1. It could be the case that the undocumented mode 0 of the HD6301Y0 is the
   same as mode 0 on the MC6801 and HD6301V1, in which case [6801reader] or
   a similar technique could be used to run one's own code to dump the ROM.
2. If mode 0 doesn't exist, the MCU could still be started in mode 2
   (external addressing but on-chip ROM enabled) and if a way could be
   found to make the code jump to any external address, that code could
   then dump the ROM.

To do this ideally the MCU would be removed from the board and run
independently. However, it may be possible to do some experimentation just
with lifting pins.



<!-------------------------------------------------------------------->
[HD6301Y0]: datasheets/HD6301Y0-CMOS-MCU.pdf
[6801reader]: https://github.com/MattisLind/6801reader
