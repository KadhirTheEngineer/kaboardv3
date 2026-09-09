# Kaboard V3 board split

The hardware is now organized as three independent KiCad project roots:

- `left/kaboard-left.kicad_*` — STM32 + nRF52840 half, 18 Hall sensors, OLED, RGB chain, charger and battery connector.
- `right/kaboard-right.kicad_*` — the corresponding right-half circuit.
- `receiver/kaboard-receiver.kicad_*` — USB-C power/input, nRF52840 receiver, OLED/status circuitry.

The original `kaboardv3-hw/` project remains the architecture/reference source. The split schematics were derived from that saved source and retain the existing component footprints, UUIDs, and net naming. Left, right, and receiver schematics contain only their own `L_`, `R_`, and `RX_` domains respectively.

## PCB status

The split PCB files are staged from the saved monolithic board, but have not yet been regenerated from the filtered schematics. Konnect requires the KiCad IPC API for safe footprint removal and schematic-to-PCB synchronization. The current KiCad instance is running without a responsive IPC socket, so no PCB deletion or sync has been performed.

## RGB integration decision

Adding RGB pads to the mechanical/Hall footprint is not electrically sufficient by itself: the Hall symbol and RGB LED symbol currently map to two separate footprints. The correct implementation is a shared multi-unit key symbol (Hall unit + RGB unit) with one combined footprint, so the LED nets receive real PCB pad assignments. The LED's exact local placement and side must be frozen before that symbol/footprint migration.
