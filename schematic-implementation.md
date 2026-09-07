# Kaboard schematic implementation

## Captured architecture

- Three electrically independent domains are represented in one root sheet:
  left half (`L_*` nets), right half (`R_*` nets), and receiver (`RX_*` nets).
  Their rails are intentionally not shorted together.
- Each keyboard half has an STM32G431RB, Raytac MDBT50Q-1MV2 nRF52840, BQ25185
  charger/power path, TPS63031 3.3 V regulator, TPS61022 5 V RGB boost,
  74AHCT1G125 level translator, TPS22919 OLED load switch, USB-C power input,
  protected-battery connector, OLED header, and independent STM32/nRF SWD
  headers.
- The receiver has an nRF52840, AP2112K-3.3 LDO, TPS22919 OLED switch, USB-C
  device connector, OLED header, and nRF SWD header.
- Each half has 18 TMAG5253BA3IQDMRR channels with three shared enable groups,
  18 individual STM32 ADC nets, 18 local 100 nF bypass capacitors, and 18
  Würth WL-ICLED 1312020030000 devices in a one-wire chain.

## Important electrical decisions

- RGB boost `EN` is driven by `RGB_PWR_EN`; `MODE` is tied to the local ground
  for PFM operation. The boost `VIN` is local 3.3 V.
- BQ25185 `/CE` is tied low so charging is enabled; STAT1/STAT2 are intentionally
  unused and marked no-connect. The pack connector is currently pin 1 BAT+,
  pin 2 NTC, pin 3 BAT−, pending supplier confirmation.
- STM32 PG10/NRST is exposed on each STM32 SWD header. nRF reset and both clock
  pins are wired to the nRF debug/crystal networks.
- Receiver battery sensing and keyboard-half battery sensing are nRF-owned;
  no STM32 ADC channel is shared with telemetry.
- The current OLED interfaces are deliberately four-wire (`VCC`, `GND`, `SCL`,
  `SDA`); the nRF `DISPLAY_RST` candidates are marked no-connect. If the final
  module requires a reset pin, that is an explicit schematic revision after the
  module is selected.
- PWR_FLAG markers identify each USB VBUS and local ground source for ERC while
  preserving the independent `L_*`, `R_*`, and `RX_*` net names. The USB-C
  connectors are power-only on the halves; the receiver nRF uses native USB.

## Verification snapshot

Konnect audits on the saved root sheet currently report:

- component connections: 0 unconnected pins;
- wire endpoints/orphans: 0;
- shorted nets: 0;
- schematic overlaps: 0;
- ERC: 0 errors, 0 warnings;
- decoupling: 66/66 power pins covered;
- power-rail and connection audits: 0 findings.

The root sheet also has explicit section headings for the three domains and
their power, Hall/ADC, MCU/radio, display/debug, and RGB blocks. Repeated Hall
and RGB values are shortened for readability (`TMAG5253 A01…A18`,
`WL-ICLED A01…A18`, and corresponding `B` channels); the exact MPNs remain
defined by the selected symbols, footprints, and this document.

## Wiring and readability convention

The schematic now follows the style of the earlier Kaboard reference design:

- local power/charger/regulator paths are drawn with physical orthogonal wires;
- Hall sensor supply, ground, and exposed-pad connections use local buses;
- each RGB row has physical DIN→DOUT links and local VDD/VSS buses;
- net labels are retained at block boundaries, for long-distance MCU/radio
  signals, and for each individual ADC channel rather than drawing page-wide
  wire crossings.

This keeps the electrical connectivity explicit where a human is likely to
debug it, while avoiding a single-page wire bundle for repeated channels.

## Release blockers

The electrical baseline must not be treated as a PCB-release package until the
following are frozen and checked against supplier/manufacturer drawings:

1. exact SH1106 module outline, connector pitch, pin order, and reset exposure;
2. selected battery's final dimensions, connector polarity, and 10 kΩ NTC
   wiring (AS584070 remains the current reference; 407090 is an alternative);
3. Gateron Jade Pro switch/magnet drawing, Hall placement, and calibration range;
4. board outlines, mounting holes, antenna keepouts, and case clearances;
5. final RGB current/thermal limits and firmware brightness policy.
