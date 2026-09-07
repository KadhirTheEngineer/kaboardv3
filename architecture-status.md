# Kaboard architecture status

This file is the checkpoint before schematic work. The KiCad schematic and PCB
remain untouched until the decisions in this document are accepted.

## Agreed direction

- Each half is an independent wireless keyboard with its own battery, charger,
  STM32, nRF52840, Hall array, USB-C power input, debug headers, and status
  indication.
- There are 18 magnetic keys and 18 independent analog Hall outputs per half.
- The STM32 owns Hall acquisition, calibration, filtering, actuation, and the
  latest coherent key-state snapshot.
- The nRF52840 owns wireless scheduling and the link to the receiver.
- The nRF is SPI master; the STM32 is SPI slave. The link has four SPI signals,
  `STM_DATA_READY`, and `STM_WAKE_NRF`.
- During active operation all Hall sensors are enabled. During inactivity the
  STM32 performs periodic low-power scans using three groups of six sensors.
- The wake scan detects the first motion, keeps that motion suppressed, wakes
  the radio, and only enables normal key reporting after the half is ready.
- The 18 Hall outputs stay directly wired to the STM32 ADC pins. The ADC
  inputs are not multiplexed.
- The STM32 and nRF assignments are recorded in
  `hardware-architecture.md`.
- USB-C on each half is power and charging only. It is not a keyboard USB data
  connection.
- Each MCU has its own permanently installed five-pin SWD header.
- Each half has a local 1.3-inch 128×64 SH1106-compatible I²C display
  connector. The receiver has the same connector as an optional population;
  the exact module and connector remain open.
- Each half has 18 individually addressable Würth `WL-ICLED 1312020030000`
  devices, one per key, in a local one-wire chain. The STM32 owns the chain
  with timer/DMA, and a switched 5 V rail plus AHCT level buffer serve the LED
  domain.
- The production radio module is Raytac `MDBT50Q-1MV2` with its integrated
  chip antenna on all three nRF52840 nodes. The Seeed XIAO nRF52840 is a
  bring-up/prototyping option only.
- The receiver is a custom nRF52840 node with native USB 2.0 full-speed HID,
  optional display, and no STM32 companion.
- The nRF radio physical layer starts with Nordic proprietary Enhanced
  ShockBurst at 1 Mbps. Packet framing, acknowledgements, and reconnect
  behavior remain intentionally unspecified until the hardware baseline is
  approved.
- The STM32 uses its internal clock tree. Each Raytac carrier provides an
  external 32.768 kHz crystal for nRF RTC/wake timing and uses the nRF DC/DC
  path.
- The default idle full-scan period is 5 ms, configurable from 2–20 ms. The
  active target is a 1 kHz coherent update; the first-motion detection target
  is at most 5 ms before radio wake and suppression.
- Battery telemetry belongs to the nRF side of each half through `BAT_SENSE`;
  all 18 STM32 ADC channels remain dedicated to Hall outputs.

## Decisions that are reserved in the pin map

These interfaces are now fixed at the architectural level; exact MPNs and
footprints are implementation work after sign-off:

- nRF display bus: I²C on `DISPLAY_SDA`/`DISPLAY_SCL`;
- display power/reset control: `DISPLAY_EN` and `DISPLAY_RST`;
- one readiness/status LED: `STATUS_LED`;
- per-key RGB data/power: STM32 `RGB_DATA` and `RGB_PWR_EN`;
- battery measurement: nRF SAADC on `BAT_SENSE`, leaving all 18 STM32 ADC
  inputs for Hall sensors. The protected AS584070 2000 mAh pack with 10 kΩ
  NTC is the current electrical reference; the 407090 3000 mAh pack remains a
  thinner but larger-footprint alternative pending supplier verification;
- Raytac module: `MDBT50Q-1MV2` with chip antenna;
- receiver USB: nRF52840 native USB full-speed device.

Changing the display to SPI, replacing the addressable chain with an RGB
matrix driver, moving the half displays away from the halves, selecting a
different module antenna, or assigning battery measurement to the STM32 would
change this reservation and must happen before the schematic.

## Decisions still required before architecture sign-off

1. **Display implementation:** choose the exact SH1106-compatible panel,
   connector, and whether `DISPLAY_EN` switches panel power or a backlight.
2. **RGB implementation:** validate the final WL-ICLED implementation and
   size the 5 V boost, bulk capacitance, level shifter, and brightness limit.
3. **Power details:** audit the BQ25185, TPS63031, LED boost, USB protection,
   battery NTC, connector pinout, and regulator enable behavior against current
   manufacturer documents.
4. **Hall mechanics:** validate sensor sensitivity, magnet orientation,
   spacing, and calibration range with the selected switch hardware.
5. **Protocol:** define packet framing, sequence numbers, CRC,
   acknowledgement, reconnect timeout, channel selection, and receiver
   behavior when one half disappears.

## Sign-off rule

The architecture is ready for schematic entry after the implementation
choices above are recorded and the power/Hall validation risks have owners.
At that point this file and
`hardware-architecture.md` become the review baseline; schematic symbols,
footprints, and net names must follow them rather than introducing new
hardware decisions during drawing.

## Schematic capture status

An architecture-level KiCad capture has now been started in
`kaboardv3-hw/kaboardv3-hw.kicad_sch`. It shows both identical keyboard halves
and the USB receiver with named functional blocks for power, Hall acquisition,
STM32, nRF52840, OLED, RGB, USB-C, and SWD. It is intentionally marked
non-release-ready: the next pass must replace the functional blocks with
verified symbols/footprints and connect the detailed power, signal, and
decoupling networks after the open decisions above are closed.
