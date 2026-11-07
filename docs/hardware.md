# OpenMUPP v1 hardware details

## Sheets

| File | Content |
|------|---------|
| `openmupp-v1.kicad_sch` | Root sheet: MCU (U2), USB-C (J1) and its ESD part (U1), VBUS fuse (F1), crystal (Y1), probe debug header (J3), LEDs, key. |
| `target_interface.kicad_sch` | Level shifters, ESD, series resistors, target connector J4, `ADAPTER_ID` clamp, `VTREF` sense divider. |
| `power.kicad_sch` | 3.3 V LDO, 12 V boost, VPP switch, target power switches (U12, U14). |
| `openmupp-v1.kicad_sym` | Project symbols (TPS2553). |

The power sheet has five inputs from the MCU: `VPP_EN` (PC14), `VPP_ON` (PC15), `VPP_12V_SEL` (PC13), `TARGET_PWR_EN` (PB8) and `TARGET_5V_EN` (PB5). It has one output, `VPP_SENSE` (PA4). It makes the global nets `VPP` and `VTREF`.

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
- **Spare channels.** U4 channels 5 to 8 are unused. Their B inputs are tied to GND. The A outputs are not connected. `DIR` and `~OE` are tied to GND, so U4 always works from target to probe. U3 channels 6 to 8 are also unused. 100 kΩ pull-downs (R49 to R51) connect their A pins to GND. Their B pins are not connected.
- **Pull-ups.** 2.2 kΩ to `VTREF` on SWDIO (R22). The one-wire interface needs the fast rise time. 10 kΩ to `VTREF` on TDO, RTCK, UART RX and nRESET. The shifters have dynamic 100 kΩ pull-downs, so an external pull-up must be 15 kΩ or less.
- **UART TX has its own shifter.** U6 is a fixed-direction part, so it always drives the line. U3 shares one direction pin (`DIR_OUT`). While `DIR_OUT` is low, the target side of U3 is high-impedance. The UART bridge must work when the JTAG outputs are released, so UART TX does not use the free channels of U3. R53 (100 kΩ) pulls `UART_TX` (PA9) up to 3.3 V. Thus U6 keeps `T_UART_TX` high when PA9 is high-impedance, for example while the MCU is in reset.
- **Series resistors.** 33 Ω (R11 to R21) on all J4 signals except pin 16 (`ADAPTER_ID`). The shifter edges are fast.
- **VTref sense.** A 100 kΩ / 100 kΩ divider (R5, R6) and a 100 nF capacitor (C4) go to PB0.
- **Isolation.** A shifter isolates the target when `VTREF` is below 100 mV.
- **Round-trip delay.** The two translators add about 26 ns at 3.3 V and 34 ns at 1.8 V (maximum values). The firmware uses a sample delay at high clock rates.

## Target power

Two TPS2553 switches output to `VTREF`. U12 switches the 3.3 V rail. U14 switches `+5V`.

- `TARGET_PWR_EN` (PB8) high: U12 supplies 3.3 V to the target. The limit is about 520 mA.
- `TARGET_5V_EN` (PB5) high: U14 supplies 5 V to the target. The limit is about 300 mA. R42 (10 kΩ) is in series with the enable pin of U14.
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

The MT3608 boost makes the VPP voltage. The boost passes its input through when it is off. Thus a P-FET switch (Q2) gates VPP. When Q3 turns on, R36 and R37 (100 kΩ each) set the gate-source voltage of Q2 to half of the boost output. At 12 V, this is −6 V, in the ±12 V limit of the AO3401A.

R52 (100 Ω) is in series between Q2 and J4 pin 19. It limits the current if pin 19 is shorted to GND or to a clamped target pin.

The default VPP voltage is 8.4 V. This level is safe for PIC parts with VIHH = 8 V to 9 V. VIHH is the high voltage that the part needs on its MCLR pin to start programming. R34 (130 kΩ) and R35 (10 kΩ) set it. To get 12 V, the firmware sets `VPP_12V_SEL` (PC13) high. This turns on Q5, which puts R45 (21.5 kΩ) in parallel with R35. R46 (100 kΩ) pulls the gate low, so a reset or a floating pin gives 8.4 V.

The boost cannot pull its output down. Thus the firmware must change the level in this order:

1. Set `VPP_ON` low.
2. Set `VPP_12V_SEL` to the new level.
3. Keep `VPP_EN` high. Wait until C29 discharges to the new level (0.5 s to 1 s when it goes from 12 V down to 8.4 V). If this takes more than 2 s, stop and report an error.
4. Set `VPP_ON` high.

`VPP_SENSE` measures the boost output through a divider (R47 100 kΩ, R48 33 kΩ). At 8.4 V, the ADC pin gets about 2.1 V. At 12 V, it gets about 3.0 V. The firmware reads it before it sets `VPP_ON`, and after a level change to see that C29 has discharged.

The firmware must set **both** `VPP_EN` (PC14) and `VPP_ON` (PC15) high to apply VPP on J4 pin 19 (through R52). Both pins have pull-downs. The firmware applies VPP only when the adapter ID shows a high-voltage adapter and the protocol needs VPP.

The VPP circuitry is on the main board. Thus a high-voltage adapter can be passive, like all other adapters.

## MCU support parts

The MCU (U2) is a CH32V307RCT6 in LQFP64. It has a QingKe V4F RISC-V core at 144 MHz, 256 KB flash and 64 KB SRAM.

- **No VCAP.** The CH32V307 has no `VCAP` pins. The supply pins `VBAT`, `VDDA`, `VDD_x` and `VIO_x` connect to 3.3 V.
- **Crystal.** Y1 is an 8 MHz crystal (YXC X50328MSB2GI, 5.0 × 3.2 mm, 2 pads) on PD0 (`OSC_IN`) and PD1 (`OSC_OUT`). The PLL multiplies 8 MHz by 18 to get 144 MHz. The USBHS PHY reference also comes from HSE. The datasheet recommends a crystal ESR of 80 Ω or less, and this crystal has 80 Ω maximum. Smaller 3.2 × 2.5 mm crystals at 8 MHz have a higher ESR, so do not use them here.
- **Load capacitors.** The crystal load capacitance (CL) is 20 pF. C2 and C3 are 30 pF C0G: 2 × (20 pF − 5 pF stray) = 30 pF. If the measured frequency is high, increase C2 and C3. If it is low, decrease them.
- **NRST.** C22 (100 nF) connects `NRST` (U2 pin 7) to GND. `NRST` also goes to J3 pin 5.
- **BOOT0 and key.** SW1 connects `BOOT0` (U2 pin 60) to 3.3 V. R7 (10 kΩ) pulls it down. Hold SW1 during reset to start the ROM bootloader. PB3 also reads the `BOOT0` net, so the firmware can use SW1 as a key. The key is active high.
- **BOOT1.** R54 (10 kΩ) pulls PB2 (`BOOT1`) down. With `BOOT0` high, `BOOT1` selects the boot source, so it must not float.
- **USB.** USB uses the high-speed port USBHS on PB6 (D−) and PB7 (D+). These pins connect to U1 directly, with no series resistors. The USB PHY of the MCU has the matching impedance. Route `USB_D-` and `USB_D+` as a 90 Ω differential pair. PA11 and PA12 (USBFS) are not used.

## MCU pin map

Each probe signal uses a pin that exists on LQFP64 (`CH32V307RCT6`) and on QFN68 (`CH32V307WCU6`). No signal needs a remap that only the LQFP100 has.

| Function | Pin | Notes |
|----------|-----|-------|
| TCK / SWCLK / ISP SCK | PB13 | SPI2_SCK, master |
| TCK link | PC10 | SPI3_SCK (SPI3 remap), slave. R10 (0 Ω, fitted) connects PC10 to the TCK net. |
| TDI / ISP MOSI | PB15 | SPI2_MOSI |
| TDO / ISP MISO | PB14 | SPI2_MISO. Same net as PA3 (USART2_RX, for SWO). |
| TMS / SWDIO | PC11 | SPI3_MISO (SPI3 remap) for the JTAG fast path. TIM10_CH4 (default map) for the one-wire engine. One pin for output and input. |
| SWIM input | PA1 | TIM2_CH2 (default map), TIM5_CH2 as an alternative. R55 (0 Ω, fitted) connects PA1 to the TMS / SWDIO net. Keep PA1 a floating input when SWIM does not run. |
| `DIR_SWDIO` | PC3 | GPIO, with external pull-down. TIM10_CH3 (default map), the same timer as TMS. |
| `DIR_OUT` | PB10 | GPIO, with external pull-down |
| TRST, RTCK | PB9, PA8 | GPIO |
| nRESET / debugWIRE drive (`SRST`) | PB12 | GPIO, to the gate of Q1. The CH32V307 USART cannot invert TX, so the firmware drives debugWIRE output in software. |
| nRESET / debugWIRE read-back (`SRST_RX`) | PB11 | USART3_RX, from U4 |
| UART TX, UART RX | PA9, PA10 | USART1 |
| `AUX1`, `AUX2` | PA2, PA0 | Outputs in the `DIR_OUT` group |
| USB | PB6, PB7 | USBHS D− and D+ |
| `VTREF_SENSE`, `ADAPTER_ID` | PB0, PB1 | ADC channels 8 and 9 |
| `TARGET_PWR_EN` | PB8 | 3.3 V target switch (U12) |
| `TARGET_5V_EN` | PB5 | 5 V target switch (U14), through R42 |
| `VPP_EN`, `VPP_ON` | PC14, PC15 | Backup-domain pins: 3 mA, 2 MHz and 30 pF maximum. Static levels only. |
| `VPP_12V_SEL` | PC13 | Low: 8.4 V. High: 12 V. Backup-domain pin, same limits as PC14. |
| `VPP_SENSE` | PA4 | ADC channel 4. Divider R47, R48 and C32 on the boost output (before Q2). |
| LED state 1 (`LED_STATE1`) | PC4 | D1 through R8 (1 kΩ). Active high. |
| LED state 2 (`LED_STATE2`) | PA7 | D4 through R40 (1 kΩ). Active high. |
| LED USB (`LED_USB`) | PA6 | D2 through R9 (1 kΩ). Active high. |
| Key | PB3 | Reads the `BOOT0` net (SW1), active high |
| `BOOT1` | PB2 | R54 pull-down |
| Crystal | PD0, PD1 | Y1, 8 MHz |
| Probe debug | PA13, PA14 | SWDIO, SWCLK on J3 |

R10 is fitted on purpose. The JTAG fast path of the firmware uses two SPI peripherals. SPI2 is the master and makes TCK on PB13. SPI3 (remap) is a slave and shifts TMS out on its MISO pin, PC11. SPI3 gets its clock from TCK through R10 on PC10. Thus TMS stays in step with TCK. Do not remove R10, or the fast path cannot shift TMS.

R55 is fitted for SWIM, the debug protocol of the STM8. The STM8 sends its bits with its own clock. Thus the probe measures the low time of each bit with a timer in PWM input mode. This timer runs at the same time as TIM10, which sends the probe bits on PC11. Thus the input needs a second pin and a second timer. The counter reset needs channel 1 or 2 of the timer, and PA1 is TIM2_CH2. TIM2 has DMA requests for CC1 and CC2. TIM3_CH2 has no DMA request, so PB4 and PC6 do not work.

These pins are not connected: PA5, PA11, PA12, PA15, PB4, PC0 to PC2, PC5 to PC9, PC12 and PD2. The CH32V307 has no JTAG port, so PB3 is a GPIO after reset. The GPIO pins are floating inputs at reset. Thus the direction pins need their external pull-down resistors.
