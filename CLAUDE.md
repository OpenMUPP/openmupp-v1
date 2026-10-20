# Versaloon-NG probe board

KiCad 10 project for a Versaloon-style multi-protocol probe built around an STM32H523.
The design plan is `/home/armin/Nextcloud/dev-plans/vsl-new-board/vsl-new-board-level-shifting.md`.

## Editing KiCad files

- Never edit `*.kicad_sch`, `*.kicad_pcb`, `*.kicad_pro`, `*.kicad_sym` or `*.kicad_mod` by hand. Use the Konnect MCP tools (`konnect` skill).
- Konnect writes to disk. KiCad keeps a lock (`~*.lck`) while a file is open in the editor. If a write is blocked, ask the user to close the editor, then reload after the edit.
- Wires and parts snap to a 1.27 mm grid. Some connector pins are off-grid (for example J11). Use `connect_pins` to reach them, because `add_wire` creates zero-length wires there.
- Verify with `find_orphan_items`, `find_shorted_nets`, `export_netlist_summary` and `render_schematic_png`. Run `run_erc` after each batch of changes.

## Layout

| File | Content |
|---|---|
| `versaloon-ng.kicad_sch` | Root sheet: STM32H523 (U1), USB-C (J11), VBUS fuse (F1), probe debug headers J2 (STDC14) and J4 (5-pin SWD), sheet instances |
| `target_interface.kicad_sch` | Level shifters, ESD, target connector J3 (2 × 10, 2.54 mm) |
| `power.kicad_sch` | 3.3 V LDO, 12 V boost, VPP switch, target power switch |
| `versaloon-ng.kicad_sym` | Project symbols (TPS2553) |

Power sheet interface: inputs `VPP_EN` (PC14), `VPP_ON` (PC15), `TARGET_PWR_EN` (PB8). Global nets `VPP` and `VTREF`.

## Design decisions

- 12 V for AVR/PIC high-voltage programming: MT3608 boost. The boost passes its input through when off, so a P-FET switch gates VPP. Both `VPP_EN` and `VPP_ON` must be high.
- Target power: TPS2553 on the 3.3 V rail, about 520 mA limit (ILIM 51 kΩ). Its output joins `VTREF`. EN high supplies the target. EN low senses the target voltage, because the part blocks reverse current.
- J3 is the only target connector. Passive adapters map it to each target pinout. J2 and J4 only debug the probe MCU, not the target.
- Target supply is on `VTREF` (J3 pin 1), because the TPS2553 output is `VTREF`. J3 pin 2 (`VTPWR`) is reserved and has a no-connect flag.
- USB VBUS has a 750 mA polyfuse (F1). Expect `+5V` near 4.4 V at full load.
- Use BSS138 (LCSC C112239) for small N-FETs.

## Conventions

- JLCPCB parts: set an `LCSC` field on every part. Prefer the basic library. Extended parts cost a setup fee for each unique part.
- Reference designators are unique across all sheets. Check before you add a part (R22–R41 and C20–C29 are used on the target sheet).
- Footprints: shared libraries live in `/home/armin/dev/electronics/kicad-footprints` (`inductors:`, `connectors:`, `misc:`, ...). Add new footprints there, not in this project. Check pads against the datasheet land pattern.
- Project symbol table `sym-lib-table` is tracked.

## Open items

- TCK link PB13 to PA5: R55 (0 Ω, DNP) is on the root sheet. Fit it only if the JTAG fast path needs it (plan, open question 11).
- Full ERC. No errors. Many `endpoint_off_grid` and library warnings remain.
