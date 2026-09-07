# Kaboard hardware architecture baseline

This document records the hardware decisions for one keyboard half. The
left and right halves use the same circuit and pin map. The KiCad files remain
unchanged while this architecture is being agreed.

## Operating modes

The half has three firmware-visible modes:

1. **Active:** all 18 Hall sensors are enabled. The STM32 scans continuously
   at the highest validated rate, processes actuation locally, and exchanges
   snapshots with the nRF52840.
2. **Scan sleep:** the STM32 stays in a low-power timer-wake state. Hall
   enable groups are pulsed periodically, one group at a time, and the 18 ADC
   pins remain permanently wired. The nRF, display, and indicator lights can
   remain asleep. The scan interval is firmware-configurable.
3. **Wireless-ready:** after a wake motion is detected, the STM32 keeps the
   sensors active and asserts `STM_WAKE_NRF`. The nRF starts or restores the
   radio link and drives the readiness indicator. The motion that woke the
   half is suppressed and does not generate a key event.

The phrase “deep sleep” means that the radio, display, lights, and normal
processing are off while the STM32 timer and the short Hall scan bursts remain
alive. A state in which every circuit is electrically inactive cannot detect a
magnetic keypress; that would require a separate physical wake input.

The wake press is tracked in firmware. Any keys already down when wake is
detected are placed in a suppression mask and are not reported. A suppressed
key must be released before it can be reported on a later press.

## MCU link

The nRF52840 is the SPI master and the STM32 is the SPI slave.

| Signal | Direction | Purpose |
|---|---|---|
| `SPI_SCK` | nRF → STM32 | SPI3 clock |
| `SPI_MOSI` | nRF → STM32 | Processed-state request/data |
| `SPI_MISO` | STM32 → nRF | Processed-state response |
| `SPI_CS_N` | nRF → STM32 | SPI3 slave select |
| `STM_DATA_READY` | STM32 → nRF | New coherent snapshot available |
| `STM_WAKE_NRF` | STM32 → nRF | Wake request after the first detected motion |

`STM_DATA_READY` and `STM_WAKE_NRF` are separate signals. The wake request is
not a data-ready event and is held until the nRF has acknowledged readiness in
firmware.

## STM32G431RBT6, LQFP64

The following application assignment is the architecture baseline. It is
frozen for discussion and schematic entry only after the sign-off conditions
in `architecture-status.md` are met.
Physical pin
numbers are for the LQFP64 package. ADC channel names are from ST's STM32G431
datasheet; `ADC12_INx` means the input is available to ADC1 and ADC2.

### Hall ADC inputs

| Sensor | STM32 pin | Package pin | ADC channel | Net |
|---:|---|---:|---|---|
| 1 | PC0 | 8 | ADC12_IN6 | `HALL_OUT_01` |
| 2 | PC1 | 9 | ADC12_IN7 | `HALL_OUT_02` |
| 3 | PC2 | 10 | ADC12_IN8 | `HALL_OUT_03` |
| 4 | PC3 | 11 | ADC12_IN9 | `HALL_OUT_04` |
| 5 | PA0 | 12 | ADC12_IN1 | `HALL_OUT_05` |
| 6 | PA1 | 13 | ADC12_IN2 | `HALL_OUT_06` |
| 7 | PA2 | 14 | ADC1_IN3 | `HALL_OUT_07` |
| 8 | PA3 | 17 | ADC1_IN4 | `HALL_OUT_08` |
| 9 | PA4 | 18 | ADC2_IN17 | `HALL_OUT_09` |
| 10 | PA5 | 19 | ADC2_IN13 | `HALL_OUT_10` |
| 11 | PA6 | 20 | ADC2_IN3 | `HALL_OUT_11` |
| 12 | PA7 | 21 | ADC2_IN4 | `HALL_OUT_12` |
| 13 | PC4 | 22 | ADC2_IN5 | `HALL_OUT_13` |
| 14 | PC5 | 23 | ADC2_IN11 | `HALL_OUT_14` |
| 15 | PB0 | 24 | ADC1_IN15 | `HALL_OUT_15` |
| 16 | PB1 | 25 | ADC1_IN12 | `HALL_OUT_16` |
| 17 | PB2 | 26 | ADC2_IN12 | `HALL_OUT_17` |
| 18 | PB11 | 33 | ADC12_IN14 | `HALL_OUT_18` |

The outputs are independent; there is no analog multiplexer and no shared
analog output net. The STM32 still converts the inputs sequentially or in a
scheduled ADC sequence.

### STM32 control and debug

| STM32 pin | Package pin | Net | Assignment |
|---|---:|---|---|
| PC6 | 38 | `HALL_EN_A` | Enable sensors 1–6 |
| PC7 | 39 | `HALL_EN_B` | Enable sensors 7–12 |
| PC8 | 40 | `HALL_EN_C` | Enable sensors 13–18 |
| PC9 | 41 | `STM_WAKE_NRF` | Wake request to nRF |
| PB10 | 30 | `STM_DATA_READY` | Snapshot-ready output |
| PD2 | 55 | `STM_STATUS_LED` | Optional local status/debug LED |
| PA15 | 51 | `SPI_CS_N` / `SPI3_NSS` | nRF-controlled slave select |
| PC10 | 52 | `SPI_SCK` / `SPI3_SCK` | SPI clock input |
| PC11 | 53 | `SPI_MISO` / `SPI3_MISO` | SPI response output |
| PC12 | 54 | `SPI_MOSI` / `SPI3_MOSI` | SPI request input |
| PA13 | 49 | `SWDIO` | STM32 debug |
| PA14 | 50 | `SWCLK` | STM32 debug |
| PG10 | 7 | `NRST` | Reset and SWD reset access |
| PB8 | 61 | `BOOT0` | Strap low for normal boot; expose for recovery |
| PA8 | 42 | `RGB_DATA` | Timer/DMA data output to the local per-key RGB chain |
| PA9 | 43 | `RGB_PWR_EN` | Enable the local 5 V RGB boost/load-switch rail |

The STM32 uses its internal clock tree (HSI plus PLL as required) in the
initial design. No external STM32 crystal is needed for the ADC scan or the
SPI link.

The STM32 battery ADC is intentionally not assigned. All 18 external ADC
inputs are reserved for Hall sensors. Battery measurement is assigned to the
nRF SAADC through `BAT_SENSE` below. If later testing requires STM32 battery
measurement, one Hall channel must be surrendered or an external ADC added.

## Raytac nRF52840 module

The selected module is the chip-antenna `MDBT50Q-1MV2`. The
`MDBT50Q-P1MV2` PCB-antenna variant has the same logical module pin names but
requires a different RF/layout decision and is not a drop-in layout substitute.

| Module pad | nRF pin | Net | Assignment |
|---:|---|---|---|
| 42 | P0.19 | `SPI_SCK` | SPI master clock |
| 43 | P0.21 | `SPI_MOSI` | SPI master output |
| 46 | P0.22 | `SPI_MISO` | SPI master input |
| 44 | P0.20 | `SPI_CS_N` | SPI chip select |
| 36 | P0.14 | `STM_DATA_READY` | GPIO interrupt input |
| 37 | P0.13 | `STM_WAKE_NRF` | GPIO sense input for System OFF wake |
| 48 | P0.24 | `STATUS_LED` | Link/readiness LED output |
| 49 | P0.25 | `DISPLAY_EN` | Display power or backlight enable |
| 45 | P0.23 | `DISPLAY_RST` | Display reset, if required by the selected panel |
| 16 | P0.27 | `DISPLAY_SDA` | I²C display data |
| 19 | P0.26 | `DISPLAY_SCL` | I²C display clock |
| 10 | P0.29 / AIN5 | `BAT_SENSE` | Divided battery-voltage input |
| 40 | P0.18 / nRESET | `NRF_RESET_N` | nRF reset, recovery, and debug access |
| 51 | SWDIO | `NRF_SWDIO` | nRF debug |
| 53 | SWDCLK | `NRF_SWCLK` | nRF debug |
| 17/18 | P0.00/P0.01 | `NRF_XL1`/`NRF_XL2` | External 32.768 kHz crystal on the carrier |

The nRF module's Raytac reference circuit still governs VDD, VDDH, DCCH,
VBUS, decoupling, and the 32.768 kHz crystal load capacitors. The design
uses the nRF DC/DC path and an external 32.768 kHz crystal for dependable RTC
wake timing. The module's RF keepout and antenna placement are layout
constraints, not schematic choices.

The display interface is I²C so it does not compete with the low-latency MCU
SPI link. Each half gets a local 1.3-inch 128×64 monochrome OLED connector
using the specified SH1106 controller family, 3.3 V power, `DISPLAY_EN`, and
`DISPLAY_RST` where the selected module exposes reset. The exact module,
connector, dimensions, and pin order remain open.
The receiver has the same interface as an optional population; it may show
both-half link and battery status but is not required for key operation.

## Per-key RGB lighting

Each half has 18 individually addressable RGB LEDs, one LED physically
associated with each Hall key. The current component baseline is 18× Würth
`WL-ICLED 1312020030000` devices in a one-wire chain. Its protocol is in the
WS2812/SK6812-style family, but its exact package and electrical limits govern
the schematic; it is not interchangeable with a generic SK6812MINI-E
footprint. QMK and Zephyr/ZMK provide useful reference drivers for this
addressable-chain model.

The LEDs are not powered from the 3.3 V logic rail. Their specified supply is
treated as a 5 V `LED_5V` rail generated by a separately enabled boost
converter. `RGB_PWR_EN` disables that converter/load switch during scan sleep
and deep idle. `RGB_DATA` is generated by the STM32 with a timer/DMA waveform
and passes through a 3.3 V-to-5 V AHCT-level buffer before the first LED. A
series data resistor and bulk/local LED bypass capacitors are required.

The 5 V rail is budgeted for at least 1 A per half, including startup and
transient margin. Firmware defaults to a brightness limit and turns the rail
off whenever RGB is not requested; full-white operation is an allowed test
case, not the expected battery-life mode. The left and right chains are
independent and are never connected through the wireless link as a physical
data chain.

## Keyboard-half power and wake policy

Each half retains the BQ25185 1-cell LiPo charger/power-path and 3.3 V
buck-boost architecture for the STM32, nRF module, Hall sensors, and display.
The LED boost is a separate switched load. USB-C on a half is power and
charging only; it has no USB data role. The current electrical battery
reference is the protected 2000 mAh AS584070 with a documented 10 kΩ NTC; the
thinner, larger-footprint 407090 3000 mAh pack is a candidate alternative.
Neither pack is mechanically frozen until supplier drawings confirm the
outline, protection, connector polarity, and NTC wiring.

The STM32 performs an entire idle scan cycle every 5 ms by default, pulsing
the three six-sensor enable groups in sequence. Each group gets enough settle
time for the Hall device worst-case power-up delay before the six ADC inputs
are sampled. The interval is a firmware setting with a supported range of
2–20 ms. Active mode targets a 1 kHz coherent sensor update. The maximum
first-motion detection delay target is therefore 5 ms, after which the nRF is
woken and the first motion remains suppressed.

## Receiver architecture

The receiver is a separate nRF52840 design using the same
`MDBT50Q-1MV2` chip-antenna module family. It has USB-C USB 2.0 full-speed
device connectivity to the host, USB ESD/CC components, a local 3.3 V rail,
an optional SH1106-compatible I²C display, and a simple status LED. It is
the radio central node for the two halves and is the only keyboard node that
reports HID input over USB. The radio physical layer is Nordic's proprietary
2.4 GHz Enhanced ShockBurst family, initially at 1 Mbps for link margin; the
packet format and reconnect policy remain a software decision.

## Module decision: Raytac versus XIAO

The production architecture uses the bare Raytac `MDBT50Q-1MV2` module on
the two halves and the receiver. It has the desired integrated chip antenna,
keeps the RF layout under the module vendor's reference design, and leaves
the charger, LED boost, USB role, and GPIO assignment under this project's
control.

The Seeed XIAO nRF52840 is retained as a bring-up/prototyping option, not the
production module. Its reference design integrates USB-C, a BQ25101 charger,
and a 3.3 V regulator, which is useful for firmware experiments, but it also
fixes the power and connector geometry and exposes a board-level pin budget
that is a poor fit for the custom Hall, display, RGB-power, and debug
architecture. A XIAO can stand in for the receiver or an early radio test;
it does not replace the Raytac-based final hardware decision.

## Debug headers

Each half gets two permanently installed right-angle headers:

| Header | Pins |
|---|---|
| STM32 SWD | `VTREF`, `GND`, `SWDIO`, `SWCLK`, `NRST` |
| nRF SWD | `VTREF`, `GND`, `SWDIO`, `SWDCLK`, `NRF_RESET_N` |

`VTREF` is a voltage reference for the debugger. The target board supplies
itself and the header must not back-power the board.

## Items still requiring an explicit decision

The MCU pin assignment above is reserved around the following remaining
system decisions:

- exact display panel/connector footprint and final SH1106-compatible MPN;
- final WL-ICLED implementation details and boost-converter sizing after the
  battery budget is audited;
- wireless packet format, acknowledgement, timeout, and reconnect behavior;
- Hall sensor sensitivity/geometry validation with the selected switch;
- detailed power-path, charger, battery NTC, and module reference-circuit audit.

These items do not change the 18-channel STM32 ADC allocation. The display
and lighting GPIO reservations above are now fixed: I²C + reset/enable for
the display, and a dedicated STM32 timer/DMA chain plus switched 5 V rail for
RGB.

## Reference architectures used

- Nordic nRF52840 proprietary-radio/ESB capability and full-speed USB device
  capability: <https://docs.nordicsemi.com/bundle/nRF52840_PS_v1.8/resource/enus/nRF52840_PS_v1.8.pdf>
- Nordic Enhanced ShockBurst packet transport model: <https://docs.nordicsemi.com/r/bundle/nrf5_sdk_v15.3.0/page/group_nrf_esb.html>
- Raytac MDBT50Q module variants and chip-antenna choice:
  <https://www.raytac.com/product/ins.php?index_id=24>
- Seeed XIAO nRF52840 reference schematic and integrated charger/regulator:
  <https://files.seeedstudio.com/wiki/XIAO-BLE/Seeed_Studio_XIAO_nRF52840_PDF.pdf>
- Established addressable-RGB firmware model (WS2812/SK6812 family):
  <https://zmk.dev/docs/features/lighting> and
  <https://docs.qmk.fm/features/rgblight>
