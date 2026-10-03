# Linefollower H7: Project Context

Handover document. Read this first in any new conversation about this project.

Other documents:
- [`docs/REVIEW_4_CUBEMX.md`](docs/REVIEW_4_CUBEMX.md): **latest.** Schematic and .ioc status (2026-09-22), remaining CubeMX steps, every clock/cache/timer term explained.
- [`docs/REVIEW_3_STATUS.md`](docs/REVIEW_3_STATUS.md): Status of every review-2 item, new findings, CubeIDE import fix, remaining to-do list.
- [`docs/REVIEW_2_AND_HOWTO.md`](docs/REVIEW_2_AND_HOWTO.md): current schematic state, part choices, KiCad/TME how-to steps, file clean-up.
- [`docs/HARDWARE_AUDIT.md`](docs/HARDWARE_AUDIT.md): first audit, plus the CubeMX settings guide (§3).
- [`docs/system_diagram.html`](docs/system_diagram.html): block diagram (open in a browser).

Last updated: 2026-09-23.

---

## 1. What this is

A competition line-follower robot. This repository holds the **main PCB** and its firmware. A separate **sensor PCB already exists**: 8 × KTIR reflective opto sensors, each with a 200 Ω emitter resistor and a 20 kΩ phototransistor pull-up. It connects over a 20-pin FFC.

**Goals:**
- competition-winning performance
- STM32H743ZIT6
- PID first, an ML model later (ST Edge AI / STM32Cube.AI)
- wheel encoders
- IMU
- battery sensing and telemetry over Bluetooth
- IMU and Bluetooth stay plug-in modules for now

**Owner's working rules:**
- **Never touch git or any version control.** The owner does all of it.
- The owner is new to STM32 and CubeMX; explain the *why*.
- The owner **hand-solders everything**. Prefer `_HandSolder` footprints, 0805 passives and leaded or castellated parts.
- The owner **cannot read Markdown pipe tables**. Use lists in documents and replies.
- Parts are bought from **TME**.

## 2. Tools

- **KiCad:** 10.0.6
- **STM32CubeIDE:** 1.18.1, at `/opt/st/stm32cubeide_1.18.1`, with the CubeMX plugin 6.14.1
- **Firmware package:** STM32Cube FW_H7 V1.12.1
- **MCU:** STM32H743ZIT6 (LQFP-144, 2 MB flash, 1 MB RAM)

## 3. Files

**Design sources:**
- `linefollowerk.kicad_pro`: the project
- `uc.kicad_sch`: **root sheet**. It holds the MCU with decoupling, VDDA filter, NRST/BOOT0, battery divider and LEDs, and the sheet symbol for the child.
- `peripherials.kicad_sch`: **child sheet**. Power input, TPS563201, AMS1117, TB6612, encoders, HC-06, IMU module, FFC.
- `linefollowerk.kicad_pcb`: unrouted placement stub, out of sync
- `linefollower.kicad_sym`, `linefollower.pretty/`: project-local custom libraries (merged 2026-09-23); `3D models/`: STEP files
- `code/`: CubeIDE project (`linefollower.ioc` plus generated HAL code)
- `stm32h743zi.pdf`: MCU datasheet

**Not design sources:**
- `linefollowerk.kicad_sch`: the old sheet
- `_restore_backup_*`, `linefollowerk-backups/`, `*.bak`, `uc_ODZYSKANE_*`: backups
- `.history/`: KiCad local history, managed by KiCad

See the clean-up list in review 2.

---

## 4. Hardware decisions

- **Battery:** 2S LiPo, 200 mAh (Redox or Gens Ace), JST-SYP / JST-RCY plug (3 A contacts). The board gets two solder pads with a short mating pigtail.
- **Protection:** Littelfuse 0154005.DRT (5 A slow-blow fuse in an OMNI-BLOK holder), then a reverse-polarity P-FET.
  - Chosen: AO4407A (SOIC-8).
  - Not SQJ465EP (too high RDS(on)); not AO3401A (SOT-23 too small).
- **5 V rail:** TPS563201 buck: R_top 54.9 k, R_bot 10 k, L 3.3 µH, 2 × 22 µF out, 2 × 10 µF + 100 nF in, 100 nF bootstrap, EN via 100 k to VIN.
- **3.3 V rail:** AMS1117-3.3 from +5V.
- **Analog rail:** +3V3A through ferrite FB1 to VDDA/VREF+.
- **Motor driver:** TB6612FNG. VM from VIN, VCC from +3V3, STBY on PG6 with a 10 k pull-down. DRV8874 is noted as a future upgrade.
- **Motors:** Pololu micro metal gearmotor HPCB 6 V, probably 10:1.
- **Encoders:** Pololu 3081 magnetic encoder kit (12 CPR). It solders onto the motor terminals and connects to the main board with 6 wires. Board pad order is M1, M2, GND, VCC, A, B at 2 mm pitch; verify against the silkscreen. VCC = +3V3.
- **IMU:** WitMotion module (confirmed by owner): MPU6050 + on-board MCU, 8 castellated pads, UART and I²C.
- **Bluetooth:** HC-06 module on USART1, VCC +5V, 3.3 V logic.
- **KTIR ADC:** 20 kΩ source impedance, so the ADC3 sampling time is 387.5 cycles, with no extra capacitors on the ADC pins.
- **Clock:** 25 MHz HSE crystal → PLL → 480 MHz (VOS0, LDO supply).
- **Debug:** 1×6 2.54 mm SWD header (J2), Nucleo CN4 pin order.

## 5. MCU pin plan

- **SENS1…SENS8** → PF3, PF4, PF5, PF6, PF7, PF8, PF9, PF10 (ADC3 INP5, 9, 4, 8, 3, 7, 2, 6; BDMA)
- **Battery voltage** → PA4 (ADC1_INP18), divider 100 k / 47 k
- **Left encoder A / B** → PA0 / PA1 (TIM2 encoder mode)
- **Right encoder A / B** → PA6 / PA7 (TIM3 encoder mode)
- **Motor PWM A / B** → PD12 / PD13 (TIM4 CH1/CH2, 20 kHz)
- **A_IN1, A_IN2, B_IN1, B_IN2** → PG2, PG3, PG4, PG5
- **M_STBY** → PG6
- **Bluetooth:** MCU TX PB14 → HC-06 RX; HC-06 TX → MCU RX PB15 (USART1)
- **IMU I²C:** PB6 SCL, PB7 SDA (I2C1)
- **IMU UART (optional):** PD5 TX → module RX; module TX → PD6 RX (USART2)
- **SWD:** PA13 SWDIO, PA14 SWCLK, PB3 SWO, NRST
- **HSE crystal:** PH0 / PH1
- **LEDs:** PE0, PE1
- **START button:** PC13 (EXTI13, pull-up)

## 6. Sensor-board FFC (20-pin, pin 1 on the right; confirmed by owner)

- Pin 1: +3V3
- Pin 5: SENS5
- Pin 6: SENS6
- Pin 7: SENS7
- Pin 8: SENS8
- Pin 9: SENS1
- Pin 10: SENS2
- Pin 11: SENS3
- Pin 12: SENS4
- Pin 20: GND
- Pins 2–4 and 13–19: not connected

Emitter current is roughly (3.3 V − ~1.2 V) / 200 Ω ≈ 10 mA per sensor, about 85 mA for all eight.

## 7. Status (2026-09-23, evening)

- **Schematic:** ERC 0 errors / 0 warnings. Libraries are merged into project-local `linefollower` (symbols and footprints).
  - **Blocker:** FFC connector J3 (formerly U7) pins 5–8 are shifted: 5 = SENS6, 6 = SENS7, 7 = SENS8, 8 = SENS5. They must be SENS5–SENS8.
  - Title blocks are empty.
- **3D model:** `ZC142200.step` must go back into `3D models/`.
- **PCB:** stub with all 79 parts. 6 footprints still use the old library names; run Update PCB from Schematic. No outline or copper yet.
- **.ioc:** complete. Generate code once more; `isbadioc=true` reappeared (see review 4 §5). Optional: USART2 for the WitMotion UART.
- **Firmware:** `main.c` starts ADC3 calibration, ADC3 DMA into 0x38000000 and TIM6.

## 8. Open questions

1. FFC contact side (same side at both cable ends, or opposite)?
2. HC-06: 6-pin breakout (STATE, RXD, TXD, GND, VCC, EN) as the footprint assumes, or 4-pin?
3. Is the battery lead's JST-SYP housing the plug (SYP-02T-1) or the receptacle (SYR-02T)?

Resolved:
- SWD is a 1×6 header (ST-LINK/V2 or Nucleo style).
- The P-FET is the AO4407A.
- FFC pin 7 is SENS7; FFC pitch is 0.5 mm (measured).
- IMU is a WitMotion module.
- L1 = Vishay IHLP2525CZER3R3M01, footprint `Inductor_SMD:L_Vishay_IHLP-2525`.

## 9. Next steps

1. Fix the review 2 blockers, then run ERC to zero errors.
2. Consolidate libraries; add footprints and TME order codes.
3. Redo the `.ioc` per the audit §3 and regenerate code.
4. PCB layout, 4 layers.
5. Firmware milestones: blink → UART telemetry → ADC3 sensor scan → encoders → PID → logging → ML.
