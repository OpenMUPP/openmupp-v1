# OpenMUPP v1 probe board

KiCad 10 project. Design documentation is in `README.md` and `docs/hardware.md`. This file has only what they do not say.

## Editing KiCad files

- Never edit `*.kicad_sch`, `*.kicad_pcb`, `*.kicad_pro`, `*.kicad_sym` or `*.kicad_mod` by hand. Use the Konnect MCP tools (`konnect` skill).
- Konnect writes to disk. KiCad keeps a lock (`~*.lck`) while a file is open in the editor. If a write is blocked, ask the user to close the editor, then reload after the edit.
- Wires and parts snap to a 1.27 mm grid. Some connector pins are off-grid (for example J1). Use `connect_pins` to reach them, because `add_wire` creates zero-length wires there.
- Verify with `find_orphan_items`, `find_shorted_nets`, `export_netlist_summary` and `render_schematic_png`. Run `run_erc` after each batch of changes.

## Schematic notes

- VPP net names: the boost feedback net is `VPP_FB`. The boost output before Q2 is `VPP_BOOST`.
- `ADAPTER_ID_MCU` (PB1 side of R44) and `VTREF_SENSE` (PB0) are outputs of the target interface sheet.
- The MCU is a CH32V307RCT6. The pin map is in `docs/hardware.md`. Probe signals must use pins that exist on LQFP64 and QFN68.
- ADC pins on this package: PA0 to PA7, PB0, PB1 and PC0 to PC5. `VPP_SENSE` is on PA4, `VTREF_SENSE` on PB0, `ADAPTER_ID_MCU` on PB1, `VPP_SENSE2` on PC1, `BOARD_ID` on PC2, `V5_SENSE` on PC0. The DAC output for VPP (`VPP_12V_ADJ`) is on PA5.

## Conventions

- Reference designators: the highest numbers in use are R61, C34, D5, J4, Q4 and U14. Q5 is free.
- New footprints go in the private `kicad-footprints` repository (`inductors:`, `connectors:`, `misc:`, ...), not in this project.
- Project symbol table `sym-lib-table` is tracked.

