# Target adapters for OpenMUPP v1

Each folder is a KiCad 10 project with the schematic of one target adapter. The projects have no board layout yet.

| Folder                                     | Adapter          | Slot | R<sub>ID</sub> | Target connectors                                              |
|--------------------------------------------|------------------|------|----------------|----------------------------------------------------------------|
| [adapter-arm](adapter-arm/)                | Arm              | 4    | 6.8 kΩ         | ARM JTAG 20, Cortex Debug 10, STDC14                           |
| [adapter-pin-header](adapter-pin-header/)  | Pin header       | 5    | 10 kΩ          | 1 × 6 SWD / CH32 / SWIM, 1 × 4 ST SWIM, 2 × 5 WCH-LinkE layout |
| [adapter-avr-hv](adapter-avr-hv/)          | AVR high voltage | 8    | 39 kΩ          | DIP-8 and DIP-14 sockets (HVSP), 2 × 3 TPI, two 2 × 3 UPDI (HV on the UPDI pin, HV on the RESET pin) |
| [adapter-pic-hv](adapter-pic-hv/)          | PIC high voltage | 9    | 91 kΩ          | PICkit ICSP 1 × 6                                              |

The slot table and the J4 pinout are in the [main README](../README.md#adapter-boards-and-adapter-id).

## Rules for all adapters

1. J1 is a female 2 × 10 header, 2.54 mm. It plugs directly onto J4 of the probe. The pin numbers of J1 are the J4 pin numbers.
2. One 1 % resistor (R1) goes from J1 pin 16 to GND. J1 pin 16 never goes to the target.
3. J1 pin 2 (reserved) stays open. J1 pin 19 (VPP) stays open on each adapter that is not a high-voltage adapter.
4. Each adapter has a 1 × 4 debug UART header: 1 GND, 2 RX (probe input), 3 TX (probe output), 4 `VTREF`. The labels are from the probe side. Pins 1 to 3 have the same order as J2 on the main board. Pin 4 lets the UART work without a debug connector to the target.
5. Connect only one target connector at a time.

## Female header on J4

J4 on the probe is a shrouded, right-angle header (`IDC-Header_2x10_P2.54mm_Horizontal`). The adapter schematics use `PinSocket_2x10_P2.54mm_Horizontal` for J1, so the adapter board is in the same plane as the probe. Before layout, check these items:

- **Fit.** The socket body must go into the shroud of J4. Measure the shroud opening and the socket body.
- **Pin mirror.** The socket and the header mate face to face. Make sure that J1 pin 1 touches J4 pin 1 in the layout.
- **Polarity.** A plain socket has no key, so it can go in rotated by 180°. Then J1 pin 16 touches a probe GND pin, and the probe pin 16 touches an adapter GND pin. The adapter ID reads 0 Ω (the fault slot). The firmware must then keep the shifter outputs, the target power and VPP off. Use a keyed socket, or add a mechanical key on the adapter board, if possible.

## RESET and MCLR on J4 pin 19

`adapter-avr-hv` and `adapter-pic-hv` connect the target RESET or MCLR pin directly to J1 pin 19. The nets are `RESET_VPP` and `MCLR_VPP`. The probe gives pin 19 three levels: 0 V (reset), `VTREF` (run) and VPP (program). See [Pin 19 states](../docs/hardware.md#pin-19-states). Thus the adapters have no reset circuit, and J1 pin 18 (`AUX2`) is not used.

HVSP entry needs RESET at 0 V while `VCC` comes up. Then RESET must go to 12 V after a short time. Q6 on the probe holds pin 19 at 0 V. The firmware can set `VPP_ON` before `VCC` comes up, because Q6 holds pin 19 low against the current limit. Then the firmware releases Q6 at the time that the data sheet gives. Pin 19 then goes to VPP in some µs.

## UPDI on J4 pin 12

UPDI uses J4 pin 12 (nRESET / debugWIRE), not J4 pin 4 (SWDIO). The reasons:

- The UPDI pin of a tinyAVR is a weak driver. The ATtiny1614/16/17 data sheet (DS40002204A, table 36-16, VOL2) guarantees only 0.1 mA at 1.8 V and 0.5 mA at 5 V.
- J4 pin 4 has a 2.2 kΩ pull-up on the probe. The target must then sink 0.8 mA to 2.3 mA. See issue [#6](https://github.com/OpenMUPP/openmupp-v1/issues/6).
- J4 pin 12 has a 10 kΩ pull-up. The target must sink 0.18 mA at 1.8 V and 0.5 mA at 5 V.
- J4 pin 12 is open-drain (Q1 on the probe). This is the physical layer that UPDI expects.

Effects on the firmware:

- The read-back of J4 pin 12 goes to PB11 (USART3_RX). The USART receives UPDI and sees its own transmit.
- The CH32V307 USART cannot invert TX, so the firmware sends UPDI in software through Q1 (PB12). debugWIRE uses the same method.
- The rise time with 10 kΩ and about 30 pF to 50 pF is about 0.3 µs to 0.5 µs. This is sufficient for 225 kbps (the maximum UPDI rate at 1.8 V). It is not sufficient for 0.9 Mbps. These values are estimates, not measurements.
- The debug UART (J4 pins 13 and 14) stays free during UPDI.

TPI data stays on J4 pin 4. On the TPI parts (ATtiny4/5/9/10), TPIDATA is a normal I/O pin with full drive strength.

## UPDI high-voltage pulse

`adapter-avr-hv` has three 2 × 3 headers for TPI and UPDI:

| Header | Use | Pin 1 (data) | Pin 5 | HV goes to |
|--------|-----|--------------|-------|------------|
| J4 | TPI, HV on RESET | `TPIDATA` (J1 pin 4) | `RESET_VPP` (J1 pin 19) | Pin 5 |
| J6 | UPDI, HV on the UPDI pin (tinyAVR 0, 1 and 2 series) | `UPDI_HV` (through Q2 to J1 pin 12) | Open | Pin 1 |
| J7 | UPDI, HV on a separate RESET pin | `UPDI` (J1 pin 12) | `RESET_VPP` (J1 pin 19) | Pin 5 |

On the tinyAVR 0, 1 and 2 series, the UPDI pin and the RESET pin are the same pin. The data sheet calls the HV entry "UPDI Enable with High-Voltage Override of RESET Pin". The HV detection level (V<sub>HVRST</sub>) is 11.5 V to 12.5 V. J7 is for parts with a separate RESET pin, for example the AVR DD family. Check the data sheet of each part before you use J7.

The parts of the UPDI path:

1. D2 (BAT54W) connects J1 pin 19 to `UPDI_HV`. D2 has a drop of about 0.3 V. Set VPP to about 12.3 V for 12 V on the UPDI pin. `RESET_VPP` has no diode, so it gets VPP directly.
2. Q2 (FDV301N) is a pass switch between `UPDI_HV` (drain) and `UPDI` (source, J1 pin 12). The gate is on `VTREF`. When `UPDI_HV` is at 12 V, Q2 is off and the 10 kΩ pull-up on the probe holds `UPDI` at `VTREF`. The body diode is then reverse biased. The ESD diode on J4 pin 12 of the probe clamps at about 6 V, so this protection is necessary.
3. While VPP is off, Q2 passes UPDI data in both directions. The probe pulls `UPDI` high (10 kΩ). The internal pull-up of the target pulls `UPDI_HV` high. The adapter has no pull-up on `UPDI_HV`, so it adds no load to the weak UPDI pad.
4. Keep pin 19 at 0 V (`VPP_PD` high) during UPDI on J6, except during the HV pulse. In the run state, pin 19 is at `VTREF`. Then R39 on the probe and D2 pull `UPDI_HV` up. When the target sends a 0, it must also sink this current: about 0.13 mA at 1.8 V and 0.45 mA at 5 V. The UPDI pad guarantees only 0.1 mA at 1.8 V. At 0 V on pin 19, D2 blocks and adds no load.

Limits:

- Q2 passes a low level only when `VTREF` is above V<sub>GS(th)</sub>. The FDV301N has 1.06 V maximum, so `VTREF` of 1.8 V is sufficient. The tinyAVR parts run from 1.8 V to 5.5 V.
- The gate-source voltage of the FDV301N must stay above −0.3 V. The source (`UPDI`) has a 10 kΩ pull-up to the gate potential (`VTREF`), so it does not go above the gate. Q2 leakage (I<sub>DSS</sub> 1 µA maximum at 20 V) moves `UPDI` by about 10 mV.
- A low-to-high edge on `UPDI_HV` above `VTREF` − V<sub>GS(th)</sub> comes from the target pull-up only. This is slower than on `UPDI`. Check the rise time at the UPDI baud rate.
- VPP goes to `RESET_VPP` and to `UPDI_HV` at the same time. Connect only one target to J2, J3, J4, J6 or J7.
- After the pulse, the line must return to the normal level quickly: the debugger drives it low 1 µs to 10 µs after the HV ends (data sheet figure 33-6). D2 cannot pull `UPDI_HV` down. The probe pulls J4 pin 12 low through its Q1. The adapter Q2 then conducts, because its gate is at `VTREF`, and pulls `UPDI_HV` low. At the same time, Q6 on the probe pulls pin 19 to 0 V in less than 1 µs, so D2 blocks at once.
- The TPS2553 on the probe has a soft start of some milliseconds. Check the pulse timing in the target data sheet.

## HVSP pull-downs

HVSP entry needs the three Prog_enable pins at 0 while `VCC` comes up. R4 to R7 (10 kΩ) hold them low on `adapter-avr-hv`. These resistors are not in the adapter plan. The pins come from the ATtiny datasheets:

| Part              | Prog_enable[0..2]           | Resistors                                          |
|-------------------|-----------------------------|----------------------------------------------------|
| ATtiny13/25/45/85 | SDI, SII, SDO (PB0 to PB2)  | The probe drives SDI and SII low. R4 holds SDO low |
| ATtiny24A/44A/84A | PA0, PA1, PA2 (pins 13, 12, 11) | R5, R6, R7                                     |

J4 pin 8 (SDO input) is an input only, so the probe cannot hold SDO low. The target drives SDO against R4 after entry.

## Items to check before layout

- **WCH-LinkE layout.** The positions of GND, 3V3 and 5V on the WCH-LinkE header come from the schematic text only. Check them on a real WCH-LinkE. Pin 8 (3V3) connects to `VTREF`. Pin 10 (5V) stays open.
- **ST SWIM order.** Check the order (VDD, SWIM, GND, NRST) against ST UM1075.
- **1.27 mm headers.** Cortex Debug 10 and STDC14 use plain 1.27 mm pin headers in the schematic. Use shrouded, keyed parts (Samtec FTSH-105 and FTSH-107, or equal). Put their footprints in the `kicad-footprints` repository.
- **Footprint libraries.** All resistors are 0603 and use `resistors:R_0603` from the `kicad-footprints` repository, as on the main board.
