# Target adapters for OpenMUPP v1

Each folder is a KiCad 10 project with the schematic of one target adapter. The projects have no board layout yet.

| Folder                                     | Adapter          | Slot | R<sub>ID</sub> | Target connectors                                              |
|--------------------------------------------|------------------|------|----------------|----------------------------------------------------------------|
| [adapter-arm](adapter-arm/)                | Arm              | 4    | 6.8 kΩ         | ARM JTAG 20, Cortex Debug 10, STDC14                           |
| [adapter-pin-header](adapter-pin-header/)  | Pin header       | 5    | 10 kΩ          | 1 × 6 SWD / CH32 / SWIM, 1 × 4 ST SWIM, 2 × 5 WCH-LinkE layout |
| [adapter-avr-hv](adapter-avr-hv/)          | AVR high voltage | 8    | 39 kΩ          | DIP-8 and DIP-14 sockets (HVSP), 2 × 3 TPI / UPDI, 2 × 3 UPDI with HV on the UPDI pin |
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

## Reset circuit of the high-voltage adapters

`adapter-avr-hv` and `adapter-pic-hv` have the same reset circuit on the target RESET or MCLR pin:

- J1 pin 19 (VPP) connects to the RESET or MCLR net. On `adapter-pic-hv` the connection is direct. On `adapter-avr-hv` it goes through D1 (BAT54W), see [UPDI high-voltage pulse](#updi-high-voltage-pulse).
- R2 (10 kΩ) goes from `VTREF` to the RESET or MCLR net.
- Q1 (BSS138) pulls the net to GND. `AUX2` (J1 pin 18) drives the gate. R3 (100 kΩ) holds the gate low.

| State   | `AUX2` | VPP | Line level            |
|---------|--------|-----|-----------------------|
| Reset   | High   | Off | 0 V                   |
| Run     | Low    | Off | `VTREF` through 10 kΩ |
| Program | Low    | On  | VPP                   |

The firmware must never set `AUX2` high while VPP is on, because Q1 then shorts VPP to GND. The BSS138 threshold is up to 1.5 V. Use `VTREF` of 1.8 V or more, or a FET with a lower threshold.

## UPDI high-voltage pulse

The tinyAVR 0, 1 and 2 series parts get the HV pulse on the UPDI pin. Some newer AVR parts get it on the RESET pin. `adapter-avr-hv` has a header for each case:

- J4 (2 × 3 TPI / UPDI): the data pin goes directly to J1 pin 4. VPP goes only to pin 5 (RESET).
- J6 (2 × 3 UPDI): the data pin is `UPDI_HV`. VPP goes to this pin.

The parts of the UPDI path on J6:

1. D2 (BAT54W) connects VPP to `UPDI_HV`. D1 (BAT54W) connects VPP to `RESET_HV`. While VPP is off, the two diodes isolate the two nets from each other and from the VPP sense divider on the probe. Each diode has a drop of about 0.3 V. Set VPP to about 12.3 V for 12 V on the target pin.
2. Q2 (BSS138) is a pass switch between `UPDI_HV` (drain) and J1 pin 4 (source). The gate is on `VTREF`. Thus the probe side stays below `VTREF` − V<sub>GS(th)</sub> when the target side is at 12 V. The body diode is then reverse biased. The ESD diode on J4 pin 4 of the probe clamps at about 6 V, so this protection is necessary.
3. While VPP is off, Q2 passes UPDI data in both directions. The 2.2 kΩ pull-up on the probe pulls the probe side high. R8 (4.7 kΩ) pulls `UPDI_HV` high.
4. During the pulse, about 1.5 mA flows from VPP through R8 into `VTREF` (at 12 V and 5 V). The probe supplies `VTREF`, so this is acceptable.

Limits:

- Q2 passes a low level only when `VTREF` is above V<sub>GS(th)</sub>. For the BSS138 this is up to 1.5 V. Use `VTREF` of 1.8 V or more.
- A low-to-high edge on the target side above `VTREF` − V<sub>GS(th)</sub> comes from R8 only. With about 30 pF, the rise time is about 0.3 µs. This is sufficient for the default UPDI baud rate. Check it before you use high UPDI baud rates. These values are estimates, not measurements.
- VPP goes to `RESET_HV` and to `UPDI_HV` at the same time. Connect only one target to J2, J3, J4 or J6.
- The TPS2553 on the probe has a soft start of some milliseconds. Check the pulse timing in the target datasheet before the firmware design.

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
