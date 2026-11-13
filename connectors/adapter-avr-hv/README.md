# adapter-avr-hv

This adapter programs AVR parts that need a high voltage (12 V) on one pin. This is necessary when a fuse disables the normal programming interface. Examples are the RSTDISBL fuse and the RSTPINCFG fuse. The adapter has slot 8 (R1 = 39 kΩ).

The probe supplies all levels on J1 pin 19: 0 V, `VTREF` and VPP. See [Pin 19 states](../../docs/hardware.md#pin-19-states). The circuit details are in [connectors/README.md](../README.md).

## Connectors

| Ref | Type | Interface | Target parts | 12 V goes to |
|-----|------|-----------|--------------|--------------|
| J1 | 2 × 10 socket | Probe J4 | — | — |
| J2 | DIP-8 socket | HVSP | ATtiny13, ATtiny25/45/85 | Pin 1 (RESET) |
| J3 | DIP-14 socket | HVSP | ATtiny24/44/84 | Pin 4 (RESET) |
| J4 | 2 × 3 header | TPI and UPDI | ATtiny4/5/9/10, tinyAVR 0, 1 and 2 series, AVR DD | Pin 5 (RESET). With JP1 closed, also pin 1 (UPDI). |
| J5 | 1 × 4 header | Debug UART | Any | — |
| JP1 | 2-pin jumper header | HV select for J4 | — | — |

HVSP is high-voltage serial programming. TPI is the Tiny Programming Interface. UPDI is the Unified Program and Debug Interface.

Connect only one target at a time. VPP goes to J2, J3 and J4 at the same time.

### J2 and J3: HVSP sockets

Put the part directly into the socket. The probe supplies `VCC` from `VTREF`.

| HVSP signal | J1 pin | J2 (DIP-8) | J3 (DIP-14) |
|-------------|--------|------------|-------------|
| RESET (12 V) | 19 | 1 (PB5) | 4 (PB3) |
| SCI (clock) | 6 | 2 (PB3) | 2 (PB0) |
| SDI (data in) | 10 | 5 (PB0) | 7 (PA6) |
| SII (instruction in) | 17 | 6 (PB1) | 8 (PA5) |
| SDO (data out) | 8 | 7 (PB2) | 9 (PA4) |
| `VCC` | 1 | 8 | 1 |
| GND | 3, 5, 7, 9, 20 | 4 | 14 |

HVSP entry needs three Prog_enable pins at 0 while `VCC` comes up. On J2, these pins are SDI, SII and SDO. The probe drives SDI and SII low, and R4 holds SDO low. On J3, these pins are PA0, PA1 and PA2 (pins 13, 12 and 11). R7, R6 and R5 hold them low. The pin data is from the ATtiny25/45/85 data sheet (table 20-13) and the ATtiny24A/44A/84A data sheet (table 19-13).

### J4: TPI and UPDI header

J4 uses the Microchip 6-pin layout for TPI and UPDI (Atmel-ICE user guide, tables 3-4 and 3-7).

| Pin | Function | Net | J1 pin |
|-----|----------|-----|--------|
| 1 | TPI data or UPDI | `UPDI_HV` | 12, through Q2 |
| 2 | `VTREF` | `VTREF` | 1 |
| 3 | TPI clock | `SCI_TPICLK` | 6 |
| 4 | — | Open | — |
| 5 | RESET | `RESET_VPP` | 19 |
| 6 | GND | GND | GND |

TPI data and UPDI use the same pin, J1 pin 12. The probe drives this pin open drain, with a 10 kΩ pull-up. The target pulls `UPDI_HV` high with its internal pull-up. See [UPDI on J4 pin 12](../README.md#updi-on-j4-pin-12).

### JP1: HV select

JP1 selects the J4 pin that gets 12 V. Set JP1 before you connect the target.

| JP1 | 12 V on | Use for |
|-----|---------|---------|
| Open | Pin 5 only | TPI (ATtiny4/5/9/10). UPDI parts with a separate RESET pin, for example AVR DD. |
| Closed | Pin 1 (through D2) and pin 5 | tinyAVR 0, 1 and 2 series. UPDI and RESET are on the same pin. |

**CAUTION:** Do not close JP1 for a TPI part or an AVR DD part. Pin 1 then gets 12 V, and this can damage the part.

- TPI needs 12 V on RESET only when the RSTDISBL fuse is programmed.
- The AVR DD setting is not verified against a part. Read the data sheet of the part before you use it.
- With JP1 closed, pin 5 also gets VPP. Some tinyAVR target boards connect pin 5 to the UPDI pin. Check the target board before you use it.
- The firmware can read the JP1 position before it sets VPP on. See [UPDI high-voltage pulse](../README.md#updi-high-voltage-pulse).

### J5: debug UART

| Pin | Signal | J1 pin |
|-----|--------|--------|
| 1 | GND | GND |
| 2 | RX (probe input) | 14 |
| 3 | TX (probe output) | 13 |
| 4 | `VTREF` | 1 |

All adapters have this header (see [Rules for all adapters](../README.md#rules-for-all-adapters)).

## Other parts

| Ref | Value | Function |
|-----|-------|----------|
| R1 | 39 kΩ | Adapter ID, slot 8 |
| D2 | BAT54W | With JP1 closed, connects `RESET_VPP` to `UPDI_HV` (J4 pin 1). It blocks while pin 19 is at 0 V. |
| Q2 | FDV301N | Pass switch between `UPDI_HV` and `UPDI`. It blocks 12 V from J1 pin 12. |
| R4 | 10 kΩ | Holds SDO (J2 pin 7) low at HVSP entry |
| R5, R6, R7 | 10 kΩ | Hold PA0, PA1 and PA2 on J3 low at HVSP entry |

J1 pins 2, 4, 11, 15 and 18 are not used.
