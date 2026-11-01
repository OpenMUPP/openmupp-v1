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
- `VPP_SENSE` is on PA4. It was `AUX2`, which is now on PB3 (a plain GPIO). Only PA3, PA4, PA6, PA7, PB0 and PB1 have ADC channels on this package.
- Firmware sequences for the power changes are tracked in issues #3 (target supply) and #4 (VPP level).

## Conventions

- Reference designators: the highest numbers in use are R48, C32, D5, Q5 and U14.
- New footprints go in the private `kicad-footprints` repository (`inductors:`, `connectors:`, `misc:`, ...), not in this project.
- Project symbol table `sym-lib-table` is tracked.

## Open items

- TCK link PB13 to PA5: R10 (0 Ω) is on the root sheet and is fitted (plan, open question 11).
- Full ERC. One error: the outputs of U12 and U14 are both power outputs on `VTREF` (on purpose, Q4 is the interlock). Also 15 `pin_to_pin` warnings, many `endpoint_off_grid` and library warnings.
- R34 must be 130 kΩ. The schematic has 191 kΩ (issue #6).
