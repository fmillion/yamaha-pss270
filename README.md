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

### Notes

The keyboard itself is matrix-encoded, with diodes in place to prevent backfeeding and allowing "N-key rollover". The control buttons are also matrixed with diodes.

The CPU appears to strobe the CE pin to write to the chip, the WE pin is held low. The CE strobe lasts a minimum of 2 microseconds - if the chip is running at 1MHz, this suggests it is likely simply writing to the port address twice in a row, spanning two cycles.

The polyphony of the OPL2 (and by extension the OPLL/YM2413) is dependent on the operation mode:

* Tone-only mode: 9 voices, 2 operators each
* Percussion mode: 6 voices, 2 ops each; 5 percussive sounds

The keyboard voice programs can either be single-voice or dual-voice (layered). Layered instruments use two voice slots; the MCU simply activates two channels instead of one.

The polyphony of the keys is dependent on both the selected voice and the auto-accompaniment setting:

* Single-voice, AA off: 9 voices.
* Layered voice, AA off: 4 voices.
* Single-voice, AA on: 2 voices. 
  * AA switches the chip to percussion mode, reducing tone voice polyphony to 6. AA requires four voices - one bass and three chord, along with the percussion voices. This leaves two channels free.
* Layered voice, AA on: 1 voice.

## Reading synth chip data

This section details ideas on intercepting writes to the YM2413. This could be used to dump the settings for each voice, duplicate the auto-accompaniment rhythms, etc.

The YM2413 follows similar logic to many early 8-bit-style synthesizer chips; an 8-bit data bus is connected to the chip, the bits are set on the bus and then either a WE or CE pin is strobed to signal the synthesizer to "read" the bytes off the bus and act accordingly. 

Since the main MCU runs at 1MHz, we should be able to use an Arduino running at 16MHz to sample the data off the bus. 

I have taken a PSS-270 keyboard and replaced the YM2413 chip with Dupont pins. I then created a small PCB on stripboard with an 18-pin IC socket and the necessary pins to allow a passive connection to the bus, the CE/WE pins and ground. 

With an Arduino, we can connect the CE pin to an input with an ISR trigger. When the ISR is triggered, we immediately sample the data off the pins and store it in a RAM buffer. Since the Arduino runs at 16MHz, even the time needed to enter the ISR (minimum 4 cycles, maybe up to 8, but still within limits) should not impact reading from the bus. We can use the Arduino hardware UART to asynchronously send the captured data off to a computer. 

For recording raw timing we can use micros() on Arduino to capture the timestamp at which the bytes arrived and include this in the data being sent via the serial port.

This will also require setting up some type of FIFO buffer in code, since sending the bytes out via serial will be much slower than the theoretical maximum of the input bytes. However, since the synth is only receiving bytes when events occur, there should be plenty of time between events to empty the FIFO. We realistically should only need to store 16 or 32 bytes of data to accommodate the difference in speed between the synth data and serial.

In theory, I should be able to inject data into the YM chip via this same method. Todo: research how to handle the situation where the MCU tries to send data at the same time as my own device. Perhaps an array of MOSFETs to "bank-switch" the YM between my own MCU and the keyboard's internal MCU? Or a more fancy approach would be combining the two approaches - setting up an Arduino as a man-in-the-middle for the YM chip. When the host MCU sends bytes, store them in a FIFO; when the Arduino generates its own bytes, also store in the same FIFO. Then dump the FIFO continuously to the actual YM chip. This needs additional thought/research.

Also todo would be to write a simple script to take this data and produce, say, a VGM file.


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
