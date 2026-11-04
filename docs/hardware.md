# OpenMUPP v1 hardware details

## Sheets

| File | Content |
|------|---------|
| `openmupp-v1.kicad_sch` | Root sheet: MCU (U2), USB-C (J1) and its ESD part (U1), VBUS fuse (F1), crystal (Y1), probe debug header (J3), LEDs, key. |
| `target_interface.kicad_sch` | Level shifters, ESD, series resistors, target connector J4, `ADAPTER_ID` clamp, `VTREF` sense divider. |
| `power.kicad_sch` | 3.3 V LDO, 12 V boost, VPP switch, target power switches (U12, U14). |
| `openmupp-v1.kicad_sym` | Project symbols (TPS2553). |

The power sheet has five inputs from the MCU: `VPP_EN` (PC14), `VPP_ON` (PC15), `VPP_12V_SEL` (PA8), `TARGET_PWR_EN` (PB8) and `TARGET_5V_EN` (PB13). It has one output, `VPP_SENSE` (PA4). It makes the global nets `VPP` and `VTREF`.

## Level shifting

The probe MCU always runs at 3.3 V. The target side of each shifter runs from `VTREF`, the target voltage.

| Target signal | Direction | Shifter |
|---------------|-----------|---------|
| TCK, TDI, TRST, `AUX1`, `AUX2` | Out | U3, direction pin `DIR_OUT` |
| TDO, RTCK, UART RX, nRESET read-back | In | U4, fixed target to probe |
| TMS / SWDIO | Both | U5, direction pin `DIR_SWDIO` |
| UART TX | Out | U6, fixed |
| nRESET / debugWIRE, drive | Out, open-drain | BSS138 (Q1), low side |

- **Idle state.** `DIR_OUT` and `DIR_SWDIO` have external 100 kΩ pull-downs (R26, R27). A floating direction pin selects target to probe, so the target side is high-impedance. This also holds while the MCU is in reset.
- **nRESET path.** Q1 has its source on GND and its drain on nRESET. The MCU never drives nRESET high. A 100 kΩ pull-down (R28) keeps the gate low while the MCU is in reset. U4 channel 4 reads the real line level back. The USART sees its own transmit, so it can detect collisions.
- **Spare channels.** U4 channels 5 to 8 are unused. Their B inputs are tied to GND. The A outputs are not connected. `DIR` and `~OE` are tied to GND, so U4 always works from target to probe.
- **Pull-ups.** 2.2 kΩ to `VTREF` on SWDIO (R22). The one-wire interface needs the fast rise time. 10 kΩ to `VTREF` on TDO, RTCK, UART RX and nRESET. The shifters have dynamic 100 kΩ pull-downs, so an external pull-up must be 15 kΩ or less.
- **UART TX has its own shifter.** U6 is a fixed-direction part, so it always drives the line. U3 shares one direction pin (`DIR_OUT`). While `DIR_OUT` is low, the target side of U3 is high-impedance, and `T_UART_TX` has no pull-up. The UART bridge must work when the JTAG outputs are released, so UART TX does not use the free channels of U3.
- **Series resistors.** 33 Ω (R11 to R21) on all J4 signals except pin 16 (`ADAPTER_ID`). The shifter edges are fast.
- **VTref sense.** A 100 kΩ / 100 kΩ divider (R5, R6) and a 100 nF capacitor (C4) go to PB0.
- **Isolation.** A shifter isolates the target when `VTREF` is below 100 mV.
- **Round-trip delay.** The two translators add about 26 ns at 3.3 V and 34 ns at 1.8 V (maximum values). The firmware uses a sample delay at high clock rates.

## Target power

Two TPS2553 switches output to `VTREF`. U12 switches the 3.3 V rail. U14 switches `+5V`.

- `TARGET_PWR_EN` (PB8) high: U12 supplies 3.3 V to the target. The limit is about 520 mA.
- `TARGET_5V_EN` (PB13) high: U14 supplies 5 V to the target. The limit is about 300 mA. R42 (10 kΩ) is in series with the enable pin of U14.
- Both low: `VTREF` senses the target voltage. The parts block reverse current.

Q4 (BSS138) pulls the enable pin of U14 low when `TARGET_PWR_EN` is high. Thus 3.3 V has priority if the firmware sets both pins by mistake.

The interlock does not remove the overlap in a change from 5 V to 3.3 V. The firmware must use this sequence for each change:

1. If both switches are off, measure `VTREF_SENSE`. If the target has its own supply, do not turn on a switch.
2. Turn off both switches.
3. Wait at least 10 ms.
4. For a change from 5 V to 3.3 V, wait until `VTREF` is less than 3.4 V. `VTREF_SENSE` shows half of `VTREF`, so wait for less than 1.7 V.
5. Turn on one switch.

Expect about 4.4 V on `VTREF` at 5 V and full board load, because of the polyfuse F1 and the drop in U14.

## High voltage (VPP)

The MT3608 boost makes the VPP voltage. The boost passes its input through when it is off. Thus a P-FET switch gates VPP.

The default VPP voltage is 8.4 V. This level is safe for PIC parts with VIHH = 8 V to 9 V. VIHH is the high voltage that the part needs on its MCLR pin to start programming. R34 (130 kΩ) and R35 (10 kΩ) set it. To get 12 V, the firmware sets `VPP_12V_SEL` (PA8) high. This turns on Q5, which puts R45 (21.5 kΩ) in parallel with R35. R46 (100 kΩ) pulls the gate low, so a reset or a floating pin gives 8.4 V.

The boost cannot pull its output down. Thus the firmware must change the level in this order:

1. Set `VPP_ON` low.
2. Set `VPP_12V_SEL` to the new level.
3. Keep `VPP_EN` high. Wait until C29 discharges to the new level (0.5 s to 1 s when it goes from 12 V down to 8.4 V). If this takes more than 2 s, stop and report an error.
4. Set `VPP_ON` high.

`VPP_SENSE` measures the boost output through a divider (R47 100 kΩ, R48 33 kΩ). At 8.4 V, the ADC pin gets about 2.1 V. At 12 V, it gets about 3.0 V. The firmware reads it before it sets `VPP_ON`, and after a level change to see that C29 has discharged.

The firmware must set **both** `VPP_EN` (PC14) and `VPP_ON` (PC15) high to apply VPP on J4 pin 19. Both pins have pull-downs. The firmware applies VPP only when the adapter ID shows a high-voltage adapter and the protocol needs VPP.

The VPP circuitry is on the main board. Thus a high-voltage adapter can be passive, like all other adapters.

## MCU pin map

| Function | Pin | Notes |
|----------|-----|-------|
| TCK / SWCLK / ISP SCK | PB10 | SPI2_SCK, master |
| TDI / ISP MOSI | PB15 | SPI2_MOSI |
| TDO / ISP MISO | PB14 | SPI2_MISO. Same net as PA3 (USART2_RX, for SWO). |
| TMS / SWDIO | PA6, PA7 | One net on the A side of U5 |
| `DIR_SWDIO` | PB5 | GPIO (TIM3_CH2, AF2, for the one-wire engine), with external pull-down |
| `DIR_OUT` | PB12 | GPIO, with external pull-down |
| TRST, RTCK | PB4, PA0 | GPIO |
| nRESET / debugWIRE drive (`SRST`) | PB6 | LPUART1_TX, inverted, to the gate of Q1 |
| nRESET / debugWIRE read-back (`SRST_RX`) | PB7 | LPUART1_RX, from U4 |
| UART TX, UART RX | PA9, PA10 | USART1 |
| `AUX1`, `AUX2` | PA2, PB3 | Outputs in the `DIR_OUT` group |
| USB | PA11, PA12 | USB D− and D+ |
| `VTREF_SENSE`, `ADAPTER_ID` | PB0, PB1 | ADC |
| `TARGET_PWR_EN` | PB8 | 3.3 V target switch (U12) |
| `TARGET_5V_EN` | PB13 | 5 V target switch (U14), through R42. PB13 is an `FT_c` pin (UCPD CC). Its drive is low and SPI2_SCK on PB13 is limited to 3 MHz. Thus TCK is on PB10. |
| `VPP_EN`, `VPP_ON` | PC14, PC15 | |
| `VPP_12V_SEL` | PA8 | Low: 8.4 V. High: 12 V. |
| `VPP_SENSE` | PA4 | ADC. Divider R47, R48 and C32 on the boost output (before Q2). |
| LED state 1, LED state 2, LED USB | PB2, PA1, PA15 | |
| Key and BOOT0 | PC13 | Also the jump to the ROM USB bootloader |
| Probe debug | PA13, PA14 | SWDIO, SWCLK on J3 |

PB9 is unused. Most H5 pins are in analog mode at reset, which is the reason for the external pull resistors on the direction pins.
