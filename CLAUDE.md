# OpenMUPP v1 probe board

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
| `openmupp-v1.kicad_sch` | Root sheet: STM32H523 (U1), USB-C (J11), VBUS fuse (F1), probe debug header J4 (5-pin SWD), sheet instances |
| `target_interface.kicad_sch` | Level shifters, ESD, target connector J3 (2 × 10, 2.54 mm), `ADAPTER_ID` clamp (R44, D5), `VTREF` sense divider (R5, R6, C4) |
| `power.kicad_sch` | 3.3 V LDO, 12 V boost, VPP switch, target power switch |
| `openmupp-v1.kicad_sym` | Project symbols (TPS2553) |

Power sheet interface: inputs `VPP_EN` (PC14), `VPP_ON` (PC15), `TARGET_PWR_EN` (PB8), `TARGET_5V_EN` (PB10), `VPP_12V_SEL` (PA8). Global nets `VPP` and `VTREF`.

## Design decisions

- 12 V for AVR/PIC high-voltage programming: MT3608 boost. The boost passes its input through when off, so a P-FET switch gates VPP. Both `VPP_EN` and `VPP_ON` must be high.
- VPP level: default 8.4 V (R34 130 kΩ, R35 10 kΩ) for PIC parts with VIHH 8 V to 9 V. `VPP_12V_SEL` (PA8) turns on Q5 (BSS138), which adds R45 (21.5 kΩ) in parallel with R35 for about 12 V. R46 pulls the gate low, so reset gives 8.4 V. The boost cannot pull its output down. Firmware order: `VPP_ON` low, select level, wait, `VPP_ON` high (issue #4). The FB net is `VPP_FB`.
- Target power: TPS2553 on the 3.3 V rail, about 520 mA limit (ILIM 51 kΩ). Its output joins `VTREF`. EN high supplies the target. EN low senses the target voltage, because the part blocks reverse current.
- 5 V target supply: a second TPS2553 (U14) switches `+5V` to `VTREF`. ILIM 91 kΩ gives about 300 mA. `TARGET_5V_EN` (PB10) drives its EN through a 10 kΩ resistor (R42). Q4 (BSS138) pulls U14 EN low when `TARGET_PWR_EN` is high, so 3.3 V has priority. Firmware must still turn both switches off between changes (see issue #3).
- `ADAPTER_ID` reaches PB1 through R44 (10 kΩ). D5 (BAT54C, both anodes) clamps PB1 to `+3.3V`, so 5 V on J3 pin 16 does not harm the MCU. R44 and D5 are on the target interface sheet. The PB1 side of R44 is net `ADAPTER_ID_MCU`, a sheet output. The `VTREF` divider (R5, R6, C4) is on that sheet too. Its output is the sheet output `VTREF_SENSE` (PB0).
- J3 is the only target connector. Passive adapters map it to each target pinout. J4 only debugs the probe MCU, not the target.
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
