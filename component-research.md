# Kaboard component and mechanical research

This is the evidence log for component choices that must be resolved before
schematic and PCB entry. It deliberately separates manufacturer/supplier
claims from project assumptions. No switch magnet dimensions, sensor offset,
board outline, or battery connector geometry is inferred here.

## Battery candidates

Both candidates are 1S Li-ion polymer packs and therefore fit the planned
single-cell charger/power-path architecture in principle. The dimensions below
are supplier-listed maximum/nominal dimensions, not a mechanical approval.

| Candidate | Supplier-listed size | Capacity / energy* | Protection / temperature evidence | Connector and polarity | Architecture status |
|---|---:|---:|---|---|---|
| A&S AS584070 | 6.0 × 40.5 × 72.5 mm | 2000 mAh / ~7.4 Wh | A&S page lists DW01 protection and 10 kΩ NTC, B=3435 ±1%; final charge limits still need the actual pack datasheet | Connector not frozen; verify pinout and NTC lead arrangement with supplier | Electrical baseline for now |
| 407090 3000 mAh | 4.0 × 70 × 92 mm listed by YDL; another supplier lists up to 4.2 × 70.5 × 92 mm | 3000 mAh / ~11.1 Wh | Public listing says PCM current limit, but no verified 10 kΩ NTC specification; do not assume protected or thermistor-equipped until the actual supplier datasheet/sample confirms it | YDL lists default PH2 2.0 mm lead and red + / black −, while also requiring polarity confirmation; connector and wire details remain open | Mechanical alternative, not approved |

\*Energy uses 3.7 V nominal and is only a comparison value. It is not a
guaranteed usable-energy figure.

The 407090 option is about 2 mm thinner than AS584070 and stores about 50%
more nominal energy, but its planar footprint is approximately 6440 mm² versus
approximately 2936 mm² for AS584070 (about 2.2× larger). It may fit a broad,
thin case better, but it is not a drop-in replacement for the narrower pack.
The 3000 mAh pack's listed standard charge rate is 0.2C (about 600 mA), so the
planned approximately 355 mA BQ25185 setting is conservative and will increase
charge time. This does not remove the need to verify the actual pack's charge,
discharge, protection, and temperature specifications.

### Battery decision gate

Keep the BQ25185 single-cell power-path architecture and the existing battery
telemetry net. Do not freeze the battery footprint, connector, NTC circuit, or
case cavity until the selected supplier provides:

1. a dimensioned mechanical drawing including thickness tolerance and lead exit;
2. protection-circuit ratings and the guaranteed maximum continuous load;
3. charge-voltage/current limits and temperature limits;
4. confirmation of whether a 10 kΩ NTC is present, including its two wires and
   B value; and
5. a keyed connector part number and unambiguous pinout/polarity.

Until those items are available, AS584070 remains the electrical reference
because its protection and NTC are documented, while 407090 remains a
mechanical/energy alternative. The PCB should reserve a connector and a
replaceable pack cavity rather than hard-coding either outline.

Sources:

- [A&S AS584070 listing](https://www.szaspower.com/products/li-polymer-battery/as584070-3-7v-2000mah-li-po-battery-ul-cb-ce-kc-pse-certified.html)
- [YDL 407090 3000 mAh listing](https://ydlbattery.com/en-gb/products/3-7v-3000mah-407090-lithium-polymer-ion-battery)
- [Supplier 407090 dimensional listing](https://dtpbattery.en.made-in-china.com/product/DNsESymKpjWd/China-407090-3-7V-3000mAh-Lipo-Battery-for-Ultra-Thin-Plate.html)

## Hall sensor package

The selected `TMAG5253BA3IQDMRR` uses TI's DMR 4-pin X2SON package, nominal
1.1 × 1.4 mm, 0.5 mm pitch, with an exposed pad. The exact installed KiCad
10 footprint is:

`Package_SON:Texas_X2SON-4-1EP_1.1x1.4mm_P0.5mm_EP0.8x0.6mm`

The sensor pins are 1=VCC, 2=GND, 3=EN, 4=OUT; exposed pad 5 is not a signal
pin. TI permits the exposed pad to be left floating or tied to GND; the current
baseline ties it to GND for mechanical support and a short thermal path. A
project-local symbol is required because the installed KiCad library does not
contain an exact TMAG5253 symbol. Each sensor gets local 0.1 µF bypassing as
specified by TI. The sensor's internal Hall-element location is not a switch
offset and must not be used to create a mechanical footprint.

Source: [TI TMAG5253 datasheet](https://www.ti.com/lit/gpn/TMAG5253)

## Magnetic switch geometry

The Gateron Full POM Low Profile Magnetic Jade Pro page confirms a linear
switch, 3.5 ± 0.2 mm total travel, 40 ± 10 gf initial force, 18 mm spring, and
square-magnet/multi-Hall behavior. The page does not provide a dimensioned
magnet drawing or a sensor-to-magnet offset in its text. Those dimensions must
come from the exact switch mechanical drawing or a measured supplier sample.

The `GLPMJ_One` open-source starter fixture is useful as a proven Jade Pro test
layout reference, but its plate/PCB dimensions are not an authoritative switch
land pattern. It must not be copied into the production PCB without checking
the exact switch drawing and the chosen TMAG5253 placement.

Source: [Gateron Full POM Low Profile Magnetic Jade Pro](https://www.gateron.com/products/gateron-full-pom-low-profile-magnetic-jade-pro-switch-set)

## Addressable RGB

The required LED is Würth `WL-ICLED 1312020030000`, not a generic
SK6812MINI-E footprint. Its package is 2.0 × 2.0 mm with pins 1=VDD,
2=DIN, 3=DOUT, 4=VSS. The device accepts 3.3–5.5 V, uses a 24-bit one-wire
stream, and requires local bypassing and the manufacturer's recommended input
protection. At a 5 V LED rail its minimum input-high threshold is 3.5 V, so a
3.3 V STM32 data signal needs the planned AHCT-level buffer. Würth publishes an
official KiCad library ZIP for this exact part; it should be inspected in a
temporary directory and not installed globally during architecture work.

Source: [Würth WL-ICLED datasheet](https://www.we-online.com/components/products/datasheet/1312020030000.pdf) and
[Würth product page / KiCad download](https://www.we-online.com/en/components/products/WL-ICLED)

## Display

The display requirement is a 1.3-inch 128×64 SH1106 monochrome OLED at 3.3 V
with four wires (VCC, GND, SCL, SDA). “SH1106-compatible” is not enough to
freeze a footprint: module outline, mounting holes, connector pitch, pin order,
and whether reset is exposed vary by vendor. The exact module and connector
remain open.
