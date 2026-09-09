# Kaboard V3 PCB placement plan

This is the placement baseline for each independent keyboard half. It is a
planning document only; it does not change KiCad source files.

## Placement zones

| Functional block | Keep close together |
|---|---|
| USB/charger input | J1, D1, R1, R2, U3, C1, C2, C3, R3, R4 |
| Battery connection | J2 immediately beside U3; keep the battery-cavity side clear |
| 3.3 V regulator | U4, L1, C4, C5; keep the switching loop compact |
| STM32 | U1, C6, C7, R6; put C6/C7 close to the MCU power pins |
| nRF52840 | U2, L6, C8, C9, C10, Y1, C11, C12, R5; crystal parts go beside the assigned nRF pins |
| nRF debug | J5 at an accessible board edge, near U2 |
| STM32 debug | J4 at an accessible board edge, near U1 |
| OLED power/interface | J3, U7, C16, C17, R7, R8 |
| RGB supply | U5, L2, C13, C14, C15, R12, R13 |
| RGB logic interface | U6 and R11 between the STM32 and the first RGB key |
| Status indicator | D2 and R14 at a visible board edge |
| Battery telemetry | R9 and R10 close to the nRF BAT_SENSE input |
| Hall channels | Each C18-C35 capacitor beside its matching TMAG5253 sensor, not in a remote bank |

## Mechanical key rules

- Preserve the 18 combined key footprints and their switch/magnet geometry.
- Keep each Hall bypass capacitor local to its sensor, preferably on the back
  side directly near the sensor if mechanical clearance allows.
- Keep the RGB LED in the switch's separate light-pipe box.
- Bring RGB data into the first key from U6, then follow an orderly chain
  through the key field.
- Do not place parts beneath switch magnet/sensor keepouts or the unverified
  battery cavity.

## Separation and keepouts

- Keep the nRF antenna keepout clear of all components, copper, traces,
  battery, USB connector, boost inductors, and RGB wiring.
- Keep Hall analog outputs and ADC traces away from U3/U4/U5 switching nodes,
  L1/L2, the 5 V RGB rail, and charger high-current paths.
- Keep Y1/C11/C12 away from RGB data, SPI clocks, boost inductors, and other
  fast/noisy signals.
- Keep the battery pack area clear except for its connector and protected power
  entry.
- Keep SWD, OLED, and USB connectors accessible and clear of tall components.

## Layer strategy

- Front: keys, RGB LEDs, connectors, power ICs, main MCU, and accessible headers.
- Back: nRF module if its antenna keepout is satisfied, Hall bypass capacitors,
  and selected small passives only where case clearance is verified.
- Do not move all passives to one side; each part stays near the circuit it serves.

## Placement order

1. Preserve the existing key field and board outline.
2. Reserve the battery cavity, antenna keepout, and connector access zones.
3. Place power-entry and regulator ICs with their local inductors/capacitors.
4. Place U1 and U2 with their decoupling, crystal, and debug headers.
5. Place the OLED/status and RGB interface blocks.
6. Place each Hall bypass capacitor locally.
7. Check courtyard overlaps, outline containment, antenna clearance, and case access.
8. Route only after placement passes those checks.

The receiver follows the same rules: USB/ESD/CC close together, nRF at the
antenna edge, regulator capacitors beside the regulator, and OLED/SWD
connectors at accessible edges.

## Current-board feasibility gate

The present left-board `Edge.Cuts` is the irregular key-field outline. The
saved placement score reports 66 of 85 footprints outside that outline. The
remaining interior margins are not large enough for all support electronics on
one side without courtyard collisions or violations of the key, sensor,
antenna, battery, and connector keepouts.

Therefore the next placement pass must use one of these approved strategies:

1. retain the outline and place only parts with verified back-side and case
   clearance, or
2. revise the mechanical outline to include a real electronics bay before
   placement.

No component should be forced into the key field merely to make the score pass.

## Layout pass completed

The left-half PCB has now been placed without changing the board outline or any
key footprint position/rotation. Functional support electronics were arranged
inside the outline; the MCU, radio, regulator/charger blocks, RGB/OLED support,
and Hall bypass capacitors use the back side where the confirmed case clearance
allows it. USB, battery, OLED, SWD, and status connectors remain accessible.

Konnect placement validation reports:

- all 85 footprints contained by the outline;
- no same-side courtyard overlap hard failures;
- placement verdict: `pass` (score 70).

The board is intentionally still unrouted. KiCad DRC therefore reports the
expected missing-connection errors. It also reports pre-existing footprint
issues: malformed courtyards in the custom key footprints and the intrinsic
0.18 mm pad-to-pad clearance in the generic D1 0201 footprint. Those are not
placement failures and must be resolved during footprint/routing cleanup before
fabrication.
