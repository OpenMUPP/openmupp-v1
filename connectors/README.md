# Target adapters for OpenMUPP v1

Each folder is a KiCad 10 project with the schematic of one target adapter. The projects have no board layout yet.

| Folder                                     | Adapter          | Slot | R<sub>ID</sub> | Target connectors                                              |
|--------------------------------------------|------------------|------|----------------|----------------------------------------------------------------|
| [adapter-arm](adapter-arm/)                | Arm              | 4    | 6.8 kΩ         | ARM JTAG 20, Cortex Debug 10, STDC14                           |
| [adapter-pin-header](adapter-pin-header/)  | Pin header       | 5    | 10 kΩ          | 1 × 6 SWD / CH32 / SWIM, 1 × 4 ST SWIM, 2 × 5 WCH-LinkE layout |
| [adapter-avr-hv](adapter-avr-hv/)          | AVR high voltage | 8    | 39 kΩ          | DIP-8 and DIP-14 sockets (HVSP), 2 × 3 TPI / UPDI              |
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

- J1 pin 19 (VPP) connects directly to the RESET or MCLR net.
- R2 (10 kΩ) goes from `VTREF` to the RESET or MCLR net.
- Q1 (BSS138) pulls the net to GND. `AUX2` (J1 pin 18) drives the gate. R3 (100 kΩ) holds the gate low.

| State   | `AUX2` | VPP | Line level            |
|---------|--------|-----|-----------------------|
| Reset   | High   | Off | 0 V                   |
| Run     | Low    | Off | `VTREF` through 10 kΩ |
| Program | Low    | On  | VPP                   |

The firmware must never set `AUX2` high while VPP is on, because Q1 then shorts VPP to GND. The BSS138 threshold is up to 1.5 V. Use `VTREF` of 1.8 V or more, or a FET with a lower threshold.

## Items to check before layout

- **AVR HVSP pull-downs (R4 to R7).** HVSP entry needs the Prog_enable pins at 0. J4 pin 8 is an input only, so R4 holds SDO low. R5 to R7 hold PA0 to PA2 of the DIP-14 part low. The adapter plan does not have these resistors. Check the pins against the ATtiny datasheets.
- **UPDI high-voltage pulse.** On `adapter-avr-hv`, the TPI / UPDI data pin goes directly to J4 pin 4. The ESD diode on J4 pin 4 clamps at about 6 V. Thus the HV pulse is possible only on the RESET pin, not on the UPDI pin. A protection part for the UPDI pin is not designed yet.
- **WCH-LinkE layout.** The positions of GND, 3V3 and 5V on the WCH-LinkE header come from the schematic text only. Check them on a real WCH-LinkE. Pin 8 (3V3) connects to `VTREF`. Pin 10 (5V) stays open.
- **ST SWIM order.** Check the order (VDD, SWIM, GND, NRST) against ST UM1075.
- **1.27 mm headers.** Cortex Debug 10 and STDC14 use plain 1.27 mm pin headers in the schematic. Use shrouded, keyed parts (Samtec FTSH-105 and FTSH-107, or equal). Put their footprints in the `kicad-footprints` repository.
- **Footprint libraries.** R1 to R7 use `resistors:R_0402` from the `kicad-footprints` repository, as on the main board.
