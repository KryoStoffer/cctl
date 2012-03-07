CC Tiny Loader
==============

CCTL is a serial bootloader for the Chipcon CC1110/CC1111 using only one 1KB page of flash.
It allows update of the the microcontroller firmware over its serial port.

The bootloader consists of two components, a piece of firmware that is flashed
onto the device and a C utility for downloading code and manipulating the
flash memory.

Authors
-------

Joby Taffey (c) 2012 <jrt-cctl@hodgepig.org>

Portions originally from CC Bootloader, Fergus Noble (c) 2011

Usage
-----

The first step is to flash your device with the `cctl.hex` firmware file. A prebuilt version `prebuilt-cctl.hex` is provided.

Once the device is flashed with this firmware it will identify itself over the serial interface with the message "CCTL" on reset.

The microcontroller is configured for 115200, 8N1 transmitting on P0.3, receiving on P0.2.
Note that the CC111x is not 5v tolerant, so be sure to use a 3v serial adapter.


You can now use cctl-prog to download your payload code.

For usage instructions, run cctl-prog with no arguments:

`./cctl-prog`

Preparing your user code for usage with the bootloader is very simple. All you
need to do is set your linker to start the code section at 0x400. For an
example of this see the `Makefile` file in the `example_payload` subdirectory.
This is the relevant line:

`LDFLAGS_FLASH = ... --code-loc 0x400 ...`

Building
--------

This requires [sdcc](http://sdcc.sourceforge.net/) (Small Device C Compiler).
Then it should be as simple as issuing

`make`

from the root directory of the project.

The code has been tested with SDCC version 3.0.0


Serial protocol
---------------

CCTL uses a custom binary serial protocol, running at 115200bps, 8 bits, no party, 1 stop bit.

On reset, the bootloader prints "\r\nCCTL\r\n" followed by "B" up to 8 times with a delay between each.
If the bootloader receives any character before printing "B" 8 times, it will enter upgrade mode.

If no character is received, the bootloader will attempt to launch user code from 0x400.
The bootloader enables the watchdog with a 1s timeout before jumping to user code, so launching a broken app should cause a reset.

Once in upgrade mode, the bootloader expects to receive at least one character per second, else it will reset using the hardware watchdog.

In upgrade mode, the following commands are available:

### Jump to user code

Jumps to user code. On failure, the device will reset.

-> `j`

## Erase page

Erase a 1KB page of flash. On completion, `\0` is sent

-> `e`, `uint8_t page` (0-31)

<- `\0`

## Read page

Read a 1KB page from flash. Sends 1024 raw bytes. On completion, '\0' is sent

-> `r`, `uint8_t page` (0-31)

<- `uint8_t data[1024]`, `\0`

## Load page

Loads 1KB page from serial into a RAM buffer. Receives 1024 raw bytes. On completion, '\0' is sent

-> `l`, `uint8_t data[1024`

<- `\0`

## Program page

Program a 1KB page of flash from RAM buffer. On completion, `\0` is sent

-> `p`, `uint8_t page` (0-31)

<- `\0`


Interrupts
----------

CCTL resides in the first flash page, 0x0000 to 0x0400. On reset, the CC1110 begins executing from address 0x0000 where it finds the CCTL reset vector.

CCTL provides a vector table which jumps to the relevant vector in application code at 0x400 + offset.

For interrupts which are used by both CCTL and application code, CCTL looks at the F1 user flag in the PSW register. When this flag is 0, the bootloader isr is called, when 1, the application isr is called.

Application code should not modify PSW.F1.



