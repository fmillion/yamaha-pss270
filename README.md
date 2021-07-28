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

* HD6301 main CPU
  * 4KB of ROM (unlikely to be able to get a ROM dump, sadly)
  * 128 bytes onboard RAM
  * 29 GPIO lines
  * Actual part number is HD6301YB61P/XC194001. No data on these part numbers, likely the specific mask ROM part numbers for this keyboard
* [YM2413](datasheets/YM2413.pdf) "[OPLL](https://en.wikipedia.org/wiki/Yamaha_YM2413)" synthesizer chip, based on OPL2 with cost-reducing modifications
* Other ICs in the device according to the schematic:
  * [LB1214](datasheets/LB1214.pdf) - Sanyo transistor array - drives LED display
  * [MN3206](datasteet/MN3206.pdf) - 128-stage low noise operation [BBD](https://en.wikipedia.org/wiki/Bucket-brigade_device) - used for stereo phasing effect? 
  * [MN3102](datasheet/MN3102.pdf) - Clock chip for MN3206 - same as above
  * [NJM4558DV](datasheet/NJM4558D.pdf) - dual op-amp - used in filtering stages
  * [TA7232P](datasheet/TA7232P.pdf) - Power amplifier

Main system TTL voltage is 5V. Since input voltage comes from 9V-12V of source power (either from DC input jack or batteries) there is a 5V regulator onboard. We could bypass this regulator for a direct 5V power feed from USB.

