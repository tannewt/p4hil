# P4HIL

An ESP32-P4 hardware-in-the-loop test fixture. A development board drops into
the socket area, and the fixture drives its pins, powers it, measures what it
draws, and exports its USB over the network so a machine anywhere can use it.

KiCad 9 source, 4 layers, 81.28 x 60.96 mm (3.2 x 2.4 in).

## Firmware

The firmware lives in a separate repository:

- [adafruit/esp-usbip-bridge](https://github.com/adafruit/esp-usbip-bridge),
  built with the `p4hil` board profile

It pulls in [adafruit/esp-harness](https://github.com/adafruit/esp-harness) for
SCPI pin and bus control, and
[adafruit/esp-perfetto-logic](https://github.com/adafruit/esp-perfetto-logic)
for logic capture.

## First power-up

Boards arrive with empty flash. Plug the USB-C connector into a host and the
board powers up, finds no application and reboots, so its USB serial device
appears and disappears every few seconds. That is what an unprogrammed board
looks like, not a fault.

The USB-A host ports sit behind load switches the firmware controls, so they
stay dark until something turns them on. There is nothing to diagnose if the
downstream power LED is off on a board you have not flashed yet.

Build once and flash each board in turn:

```bash
./scripts/build-board.sh p4hil build
./scripts/build-board.sh p4hil flash /dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_AA:BB:CC:DD:EE:FF-if00
```

The trailing field is the board's MAC address, so the `by-id` path names one
specific board. Prefer it over `ttyACM0` when more than one is attached, since
the numbering moves around between reboots.

After flashing, the board takes a DHCP address over Ethernet and advertises
itself, so you do not need to know the address in advance:

```bash
avahi-browse -rtp _usbip._tcp
```

## What is on the board

| Function | Part |
|---|---|
| MCU | ESP32-P4NRW32, 32 MB in-package PSRAM |
| Flash | W25Q128JVS, 16 MB |
| Ethernet | IP101GR PHY, HanRun HR911105A magjack |
| USB host | CH334F 4-port hub, stacked USB-A plus a 2x5 header |
| USB device | USB-C, full speed, console and power in |
| Current sense | INA219 across a 0.1 ohm shunt on the switched host rail |
| IO expanders | 3x PI4IOE5V9535 at 0x20, 0x21, 0x22 |
| Host power | 2x MT9700/SY6280 load switches, software controlled |
| Misc | WS2812B, STEMMA QT, reset and boot buttons |

## The DUT socket

There is one fixed 20-pin row along the bottom edge and six 19-pin rows stacked
above it at 0.1 inch intervals. All six upper rows carry the same 19 signals, so
you populate only the row that matches the board you want to test.

| Row | Spacing | Silkscreen | Fits |
|---|---|---|---|
| 0 | 0.5" | Trinket | Trinket |
| 1 | 0.6" | QTPy/XIAO/ProMicro | QT Py, XIAO, Pro Micro, Nano |
| 2 | 0.7" | Pico | Raspberry Pi Pico |
| 3 | 0.8" | Feather | any Feather |
| 4 | 0.9" | ESP DevKit | ESP32-S3 DevKitC |
| 5 | 1.0" | ESP32 DevKit | ESP32 DevKitC |

Devices are right-aligned against the `<Start Here` mark on the bottom row.
Marks along the bottom edge show how far each form factor reaches.

Arduino Uno is handled separately on the back of the board: J4 and J2 for the
digital headers, J9 for the analog header. The Uno power header is deliberately
skipped.

## Before you populate headers

**Boards arrive with no DUT headers fitted.** They are through-hole, and which
row you solder is up to you.

Two things worth knowing before you do:

- No position on either DUT row carries power or ground. All 39 are signal lines
  running to ESP32-P4 GPIO.
- The 39 series parts named `R_Protect` in the schematic are 0 ohm in the BOM.

So work out where your device's power pins land before soldering, and leave
those positions unpopulated. A 5 V pin such as a Pico's VBUS would otherwise sit
directly on a 3.3 V GPIO. Ground is shared through the USB cable when the device
is plugged into one of the fixture's own USB-A ports, which is also how the
fixture power-cycles it and measures its current.

## Ordering

`jlcpcb/production_files/` has the gerbers, BOM and CPL as submitted. The CPL
places surface-mount parts only, which is why the DUT headers are unpopulated.

## Repository layout

```
p4hil.kicad_pro / .kicad_sch / .kicad_pcb   top level design
dut.kicad_sch                               DUT headers and pull network
ethernet.kicad_sch                          IP101GR and magjack
usb_host.kicad_sch                          CH334F hub, load switches, INA219
usb_device.kicad_sch                        USB-C
power_options.kicad_sch                     regulators
p4.kicad_sch                                ESP32-P4, flash, crystal
jlcpcb/                                     production output
```

Symbol and footprint libraries come from the `chickadee-libs` and
`kicad-libraries` submodules.
