# OpenMUPP v1 probe board

KiCad 10 project for a Versaloon-style multi-protocol probe built around an STM32H523.

## Editing KiCad files

- Never edit `*.kicad_sch`, `*.kicad_pcb`, `*.kicad_pro`, `*.kicad_sym` or `*.kicad_mod` by hand. Use the Konnect MCP tools (`konnect` skill).
- Konnect writes to disk. KiCad keeps a lock (`~*.lck`) while a file is open in the editor. If a write is blocked, ask the user to close the editor, then reload after the edit.
- Wires and parts snap to a 1.27 mm grid. Some connector pins are off-grid (for example J1). Use `connect_pins` to reach them, because `add_wire` creates zero-length wires there.
- Verify with `find_orphan_items`, `find_shorted_nets`, `export_netlist_summary` and `render_schematic_png`. Run `run_erc` after each batch of changes.

## Layout

| File | Content |
|---|---|
| `openmupp-v1.kicad_sch` | Root sheet: STM32H523 (U2), USB-C (J1), VBUS fuse (F1), probe debug header J3 (5-pin SWD), sheet instances |
| `target_interface.kicad_sch` | Level shifters, ESD, target connector J4 (2 × 10, 2.54 mm), `ADAPTER_ID` clamp (R44, D5), `VTREF` sense divider (R5, R6, C4) |
| `power.kicad_sch` | 3.3 V LDO, 12 V boost, VPP switch, target power switches (U12, U14) |
| `openmupp-v1.kicad_sym` | Project symbols (TPS2553) |

Power sheet interface: inputs `VPP_EN` (PC14), `VPP_ON` (PC15), `TARGET_PWR_EN` (PB8), `TARGET_5V_EN` (PB10), `VPP_12V_SEL` (PA8). Output: `VPP_SENSE` (PA4, ADC). Global nets `VPP` and `VTREF`.

## Design decisions

- 12 V for AVR/PIC high-voltage programming: MT3608 boost. The boost passes its input through when off, so a P-FET switch gates VPP. Both `VPP_EN` and `VPP_ON` must be high.
- VPP level: default 8.4 V (R34 130 kΩ design value, the schematic has 191 kΩ, issue #6; R35 10 kΩ) for PIC parts with VIHH 8 V to 9 V. `VPP_12V_SEL` (PA8) turns on Q5 (BSS138), which adds R45 (21.5 kΩ) in parallel with R35 for about 12 V. R46 pulls the gate low, so reset gives 8.4 V. The boost cannot pull its output down. Firmware order: `VPP_ON` low, select level, wait, `VPP_ON` high (issue #4). The FB net is `VPP_FB`.
- `VPP_SENSE` (PA4): divider R47 (100 kΩ) and R48 (33 kΩ) with C32 (100 nF) on the boost output (net `VPP_BOOST`, before Q2). PA4 was `AUX2`. `AUX2` is now on PB3, which is fine because `AUX2` is a plain GPIO. Only PA3, PA4, PA6, PA7, PB0 and PB1 have ADC channels on this package.
- Target power: TPS2553 on the 3.3 V rail, about 520 mA limit (ILIM 51 kΩ). Its output joins `VTREF`. EN high supplies the target. EN low senses the target voltage, because the part blocks reverse current.
- 5 V target supply: a second TPS2553 (U14) switches `+5V` to `VTREF`. ILIM 91 kΩ gives about 300 mA. `TARGET_5V_EN` (PB10) drives its EN through a 10 kΩ resistor (R42). Q4 (BSS138) pulls U14 EN low when `TARGET_PWR_EN` is high, so 3.3 V has priority. Firmware must still turn both switches off between changes (see issue #3).
- `ADAPTER_ID` reaches PB1 through R44 (10 kΩ). D5 (BAT54C, both anodes) clamps PB1 to `+3.3V`, so 5 V on J4 pin 16 does not harm the MCU. R44 and D5 are on the target interface sheet. The PB1 side of R44 is net `ADAPTER_ID_MCU`, a sheet output. The `VTREF` divider (R5, R6, C4) is on that sheet too. Its output is the sheet output `VTREF_SENSE` (PB0).
- J4 is the only target connector. Passive adapters map it to each target pinout. J3 only debugs the probe MCU, not the target.
- Target supply is on `VTREF` (J4 pin 1), because the TPS2553 output is `VTREF`. J4 pin 2 (`VTPWR`) is reserved and has a no-connect flag.
- USB VBUS has a 750 mA polyfuse (F1). Expect `+5V` near 4.4 V at full load.
- Use BSS138 (LCSC C112239) for small N-FETs.

## Conventions

- JLCPCB parts: set an `LCSC` field on every part. Prefer the basic library. Extended parts cost a setup fee for each unique part.
- Reference designators are unique across all sheets. Check before you add a part (the highest numbers are R48, C32, D5, Q5 and U14).
- Footprints: shared libraries live in the private `kicad-footprints` repository (`inductors:`, `connectors:`, `misc:`, ...). Add new footprints there, not in this project. Check pads against the datasheet land pattern.
- Project symbol table `sym-lib-table` is tracked.

## Open items

- TCK link PB13 to PA5: R10 (0 Ω) is on the root sheet and is fitted (plan, open question 11).
- Full ERC. One error: the outputs of U12 and U14 are both power outputs on `VTREF` (on purpose, Q4 is the interlock). Also 15 `pin_to_pin` warnings, many `endpoint_off_grid` and library warnings.
- R34 must be 130 kΩ. The schematic has 191 kΩ (issue #6).
