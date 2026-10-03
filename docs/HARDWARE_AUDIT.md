# Hardware & CubeMX Audit: Linefollower H7 main board (review 1)

Date: 2026-09-15.

This is the first-pass audit. The owner has since fixed part of the schematic. For the current state of the schematic, part choices and how-to steps, read **[`REVIEW_2_AND_HOWTO.md`](REVIEW_2_AND_HOWTO.md)**. For the latest status and CubeMX explanations, read **[`REVIEW_4_CUBEMX.md`](REVIEW_4_CUBEMX.md)**. Items fixed in review 2 are marked **[fixed]**. Items where this audit's first version gave advice that later turned out wrong are marked **[corrected]**.

Files audited in review 1:
- `linefollowerk.kicad_sch` (old sheet 1)
- `uc.kicad_sch`
- `linefollowerk.kicad_pro`
- `linefollowerk.kicad_pcb` (state only)
- the libraries
- `code/linefollower.ioc` and the generated code

---

## 0. How this review was done

- **Analyzer:** kicad-happy schematic analyzer.
- **ERC:** KiCad 10.0.6 `kicad-cli sch erc` found 133 violations in review 1.
- **Visual check:** PDF renders of every sheet.
- **Datasheets actually read:**
  - TI LMR16006
  - TI TPS563201 (Table 7-2, recommended component values)
  - STM32H743ZI datasheet DS12110 (owner-supplied `stm32h743zi.pdf`): pin and alternate-function tables, VCAP C_EXT = 2.2 µF per pin, 480 MHz needs VOS0 on revision V, ADC f_ADC max 36 MHz with BOOST = 1, ADC R_AIN max 50 kΩ
  - ST AN4938: PDR_ON must be tied to VDD
  - Alpha & Omega AO4407A
  - Vishay SQJ465EP and SQJ457EP (via distributor summaries)
  - Pololu magnetic encoder product pages
- **Not done:**
  - PCB, EMC and thermal analysis: the PCB is an unrouted stub.
  - SPICE.
  - BOM lifecycle audit: no MPNs are set.

Severity:
- **BLOCKER**: the board will not work.
- **MAJOR**: works poorly, risks damage, or fails in competition.
- **MINOR**: hygiene.

---

## 1. Schematic findings (review 1)

### Structure and nets

- **S-01 BLOCKER [partly fixed]: Two separate top-level sheets.**
  - Now `uc.kicad_sch` is root and `peripherials.kicad_sch` is its child.
  - Still broken: the labels do not cross sheets. See review 2, item B1.
- **S-02 BLOCKER [fixed]:** The MCU 3.3 V rail had no source (label `3.3V` ≠ power symbol `+3.3V`).
- **S-03 BLOCKER [fixed]:** TB6612 `VCC`/`VDD` supply nets had no source.
- **S-04 / S-05 BLOCKER [fixed]:** `MGND` was isolated and TB6612 pin 18 was floating.
- **S-06 MAJOR [partly fixed]:** `GNDD` mounting holes are gone, but the holes now use a wrong footprint (`Module:Maple_Mini`). See review 2.
- **S-07 MINOR [fixed]:** Empty-name label merged on the CB net.
- **S-08 MINOR:** Title blocks empty. Fill in Title, Rev and Date (File → Page Settings).

### Power input

- **S-09 MAJOR [decision made]:** J1 was JST-PH. The battery is a 2S 200 mAh pack with a JST-SYP (RCY) plug. See review 2 for the board-side solution.
- **S-10 MAJOR [corrected, still open]:** Q1 AO3401A (SOT-23) is too small for motor current.
  - The first suggestion stands: a P-FET with ≤ 20 mΩ at V_GS = −6 V.
  - The owner's proposed SQJ465EP is **not suitable** (115 mΩ at −4.5 V).
  - Use **AO4407A** (SOIC-8, < 17 mΩ at −6 V) or **SQJ457EP** (PowerPAK SO-8L, 35 mΩ at −4.5 V).
- **S-11 MAJOR [decision made]:** Fuse: Littelfuse 0154005.DRT (5 A slow-blow, fuse pre-installed in holder).
- **S-12 MAJOR [fixed]:** Battery divider 100 k / 47 k + 100 nF on PA4.
- **S-13 MAJOR [partly fixed]:** Bulk capacitors.
  - C34 470 µF was added, but its footprint `CP_Elec_5x5.4` is too small for 470 µF / 16 V.
  - See review 2 for the size and placement.

### Regulators

- **S-14 MAJOR [partly fixed]:** Power tree is now TPS563201 → +5V → AMS1117-3.3 → +3V3.
  - The TPS563201 has no inductor yet.
  - The AMS1117 output capacitor needs changing.
  - See review 2, items B3 and M4.
- **S-15 [no longer applies]:** LMR16006 removed.
- **S-16 MINOR:** Value notation. See review 2 for the full list.

### Motor driver (TB6612FNG)

- **S-17 BLOCKER [still open]:** The outputs are labelled but not connected to the motor pads.
- **S-18 BLOCKER [partly fixed]:** The control lines are labelled on both sheets but do not cross the sheet boundary (review 2, B1).
- **S-19 MAJOR [fixed]:** STBY → PG6 with a 10 k pull-down.
- **S-20 MAJOR [note]:** At 8.4 V a 6 V HPCB 10:1 motor can stall at about 2 A, above the TB6612's 1.2 A continuous rating.
  - Limit PWM in firmware for rev 1.
  - Consider 2× DRV8874 later; the schematic already has a note saying so.
- **S-21 MINOR:** ERC output-to-output errors on the paralleled AOx pins. Set those symbol pins to `passive` in the symbol editor.
- **S-22 MINOR:** 10 nF ceramic across each motor's terminals, soldered at the motor.

### Encoders

- **S-23 BLOCKER [partly fixed]:** VCC, GND and A/B are wired; the motor pads M1/M2 are still unconnected (S-17).

### Sensor-board FFC

- **S-24 BLOCKER [partly fixed]:** Signals are mapped. Still to do:
  - no footprint
  - the pin electrical types
  - SENS1–8 are not labelled at the MCU
- **S-24 note [corrected]:** The first version of this audit suggested a 1 nF capacitor at every ADC pin. **Do not do this.** The KTIR outputs come from a 20 kΩ pull-up, so 1 nF would make a 20 ms low-pass filter, far too slow for a line follower. Use a longer ADC sampling time instead (§3.5).
- **S-25 MAJOR (next sensor-board revision):**
  - Add more GND pins on the FFC.
  - Add an emitter-enable line.

### MCU

- **S-26 BLOCKER [fixed]:** PDR_ON now on +3V3.
- **S-27 BLOCKER [still open]:** No SWD connector.
- **S-28 MAJOR [still open]:** No HSE crystal.
- **S-29 MAJOR [partly fixed]:** LEDs D3/D4 added on PE0/PE1; the START button is still missing.
- **S-30 OK:** Items verified as correct:
  - VDD decoupling
  - VBAT and VDD33_USB tied to +3V3
  - VCAP 2 × 2.2 µF (datasheet C_EXT = 2.2 µF per pin)
  - VDDA filter
  - NRST button with 100 nF
  - BOOT0 10 k pull-down
- **S-31 MINOR [still open]:** FB1 has no footprint.
- **S-32 NICE-TO-HAVE:** USB-C on PA11/PA12 for DFU and a CDC telemetry cable.

### Modules

- **S-33 BLOCKER [still open, new error]:** HC-06 is labelled, but TX goes to TX (review 2, B4).
- **S-34 [updated]:** The IMU is an MPU6050 + microcontroller module (WitMotion WT61/JY61-type, UART and I²C). See review 2.

### Libraries

- **L-01 MAJOR [still open]:** `sym-lib-table` still points to `C:/Users/rataj/...`. `linefolower_stm` is not registered, and `Elements2` does not exist on disk.
- **L-02 MINOR [still open]:** Footprint libraries are duplicated; IC1 points to a footprint that is not in the library it names.
- **L-03 MINOR:** Missing footprints: D2, D3, D4, FB1, U7.
- **L-04 MINOR:** No MPN or supplier fields on any part.
- **L-05 MINOR:** Backup clutter in the project root. See the file clean-up list in review 2.

### PCB

- **P-01:** Unrouted 2-layer stub, out of sync with the schematic. Recommended: 4 layers (signal / GND / power / signal), buck hot loop tight, crystal next to PH0/PH1, motor current kept away from the MCU and ADC traces.

---

## 2. Remaining blocker checklist

- [ ] Labels cross the sheet boundary (global labels or sheet pins)
- [ ] SENS1–SENS8 labelled at PF3–PF10
- [ ] TPS563201 inductor and capacitors
- [ ] HC-06 TX/RX crossed
- [ ] TB6612 outputs to motor pads
- [ ] IC1 footprint fixed
- [ ] SWD connector
- [ ] HSE crystal + load capacitors
- [ ] ERC → 0 errors

---

## 3. CubeMX (.ioc) review and recommended settings

### 3.1 What is in the .ioc today

- **Clock:** SYSCLK = HSI 64 MHz, no PLL, VOS scale 3; HSE pins enabled at 25 MHz but unused.
  - Verdict: **change** (§3.2).
- **Power:** `PWR_LDO_SUPPLY`.
  - Verdict: **correct** for H743 (LDO + VCAP capacitors).
- **Debug:** Serial Wire + trace SWO (PA13 / PA14 / PB3).
  - Verdict: correct; needs the SWD connector on the board.
- **S0–S7:** PF0–PF7 as GPIO_Input.
  - Verdict: **wrong.** The KTIR outputs are analog, and PF0–PF2 have no ADC. Move to ADC3 on PF3–PF10.
- **ADC1:** IN18 on PA4, 12-bit, 64.5 cycles.
  - Verdict: keep for battery voltage; raise the sampling time (§3.6).
- **TIM2:** Encoder on PA0/PA1, period 0xFFFFFFFF, input filter on CH1 only.
  - Verdict: add a filter on CH2.
- **TIM3:** Encoder on PA6/PA7, period 0xFFFF, no filter.
  - Verdict: add filters.
- **TIM4:** PWM CH1/CH2 on PD12/PD13, PSC 19, ARR 249 → 12.8 kHz at 64 MHz.
  - Verdict: recompute after the clock change (§3.4).
- **USART1:** PB14/PB15, 9600 baud, RX DMA circular.
  - Verdict: raise to 115200, add TX DMA.
- **I2C1:** PB6/PB7, Fast mode.
  - Verdict: correct.
- **PC4:** EXTI, falling edge.
  - Verdict: **remove.** The chosen IMU module has no INT pad.
- **PE0:** GPIO out "DIODE1".
  - Verdict: correct. Also add PE1 for the second LED.
- **PG2–PG6:** AIN1, AIN2, BIN1, BIN2, STBY outputs.
  - Verdict: matches the schematic; set the initial level to Low.
- **Linker script:** `.data` / `.bss` in RAM_D1 (0x24000000).
  - Verdict: correct. DMA1/DMA2 can reach RAM_D1; they cannot reach DTCM.

### 3.2 Clock (needs the crystal)

**Pinout → System Core → RCC:**
- High Speed Clock (HSE) → **Crystal/Ceramic Resonator**
- Low Speed Clock (LSE) → **Disable**

**Clock Configuration tab** (type the values into the boxes):
- **Input frequency (HSE):** 25 MHz
- **PLL Source Mux:** HSE
- **DIVM1:** /5 → 5 MHz into PLL1 (allowed VCO input range 4–8 MHz)
- **DIVN1:** ×192 → 960 MHz VCO
- **DIVP1:** /2 → **480 MHz**
- **System Clock Mux:** PLLCLK → SYSCLK = 480 MHz
- **D1CPRE:** /1 → CPU at 480 MHz
- **HPRE:** /2 → AHB at 240 MHz
- **D1PPRE, D2PPRE1, D2PPRE2, D3PPRE:** /2 each → APB buses at 120 MHz; timer clocks become 240 MHz

**Power parameters** (RCC → Parameter Settings → Power Parameters):
- Supply Source: `PWR_LDO_SUPPLY`
- Power Regulator Voltage Scale: **Scale 0** (required above 400 MHz; datasheet: 480 MHz on revision V)

**ADC clock:**
- Choose PLL2P (or PLL3R) in the ADC clock mux.
- Keep the effective ADC clock **≤ 36 MHz** (BOOST = 1).
- Example: DIVM2 /5, DIVN2 ×72, DIVP2 /10 → 36 MHz.
- CubeMX colours a box red if a limit is broken.

**Is 25 MHz a good crystal?** Yes. Any value from 4 to 48 MHz works; 25 MHz divides cleanly to 5 MHz for the PLL and is cheap and common. Keep it.

### 3.3 Cortex-M7

System Core → CORTEX_M7:
- **ICache:** Enable
- **DCache:** Enable

With DCache on, DMA buffers must either sit in a non-cached region or be flushed with `SCB_CleanDCache_by_Addr()` before a DMA transmit.

MPU (same panel, added in review 4): keep region 0 as generated. Add **Region 1**:
- Base 0x38000000, size 64KB, SubRegion Disable 0x0
- TEX level 1, all access permitted, instruction access disabled
- not shareable, not cacheable, not bufferable

This makes RAM_D3 (the ADC3/BDMA buffer) non-cacheable.

### 3.4 Timers

- **TIM2 (left encoder):**
  - Combined Channels → Encoder Mode, pins PA0 (CH1) / PA1 (CH2)
  - Encoder Mode: TI1 and TI2
  - Counter Period: 0xFFFFFFFF
  - Input Filter: 6 on CH1 and CH2
- **TIM3 (right encoder):**
  - Encoder Mode, pins PA6 / PA7
  - Counter Period: 0xFFFF
  - Input Filter: 6 / 6
  - Read the change each control tick as `int16_t` so wrap-around is harmless
- **TIM4 (motor PWM):**
  - PWM Generation CH1 / CH2 on PD12 / PD13
  - Prescaler 11, Counter Period 999 → 240 MHz / 12 / 1000 = **20 kHz**, with 1000 duty steps
  - auto-reload preload: Enable
- **TIM6 (control loop, new):**
  - Activated
  - Prescaler 239, Counter Period 999 → **1 kHz** interrupt
  - NVIC: enable

### 3.5 ADC3: line sensors (new)

1. **Pins:** click each pin and choose the ADC function (all single-ended):
   - PF3 → ADC3_INP5 (SENS1)
   - PF4 → ADC3_INP9 (SENS2)
   - PF5 → ADC3_INP4 (SENS3)
   - PF6 → ADC3_INP8 (SENS4)
   - PF7 → ADC3_INP3 (SENS5)
   - PF8 → ADC3_INP7 (SENS6)
   - PF9 → ADC3_INP2 (SENS7)
   - PF10 → ADC3_INP6 (SENS8)
   - Reset PF0–PF7 from GPIO_Input first.
2. **Parameter settings:**
   - Resolution: 12-bit
   - Scan Conversion Mode: switches on by itself (greyed) once Number of Conversions > 1
   - Continuous Conversion: Disable
   - Number of Conversions: 8; Rank 1–8 = SENS1…SENS8
   - **Sampling Time: 387.5 cycles** [corrected]
     - Why: the KTIR output impedance is about 20 kΩ, and the datasheet sampling-time table requires several microseconds at kΩ-level sources.
     - At 36 MHz, 387.5 cycles ≈ 11 µs per channel, so a full 8-sensor scan takes about 0.1 ms. That is still 10× faster than the 1 kHz control loop.
   - Conversion Data Management: DMA Circular
   - Oversampling: optional (ratio 4, right shift 2) for noise
3. **DMA:** [corrected in review 4]
   - ADC3 lives in the D3 power domain and uses **BDMA**, which can only reach **RAM_D3 (0x38000000)**.
   - In CubeMX: ADC3 → DMA Settings → Add → ADC3 (CubeMX picks BDMA Channel 0), Circular, Half Word / Half Word. Then Conversion Data Management Mode = DMA Circular Mode. "BDMA channel0 global interrupt" then appears in NVIC.
   - Do **not** edit the linker script: CubeMX rewrites `STM32H743ZITX_FLASH.ld` when it regenerates. Put the buffer at a fixed address in `USER CODE` instead:
     ```c
     #define ADC3_BUF ((volatile uint16_t *)0x38000000UL)
     HAL_ADC_Start_DMA(&hadc3, (uint32_t *)ADC3_BUF, 8);
     ```
   - Make RAM_D3 non-cacheable with MPU region 1 (§3.3). Otherwise the CPU reads stale cached values.
4. **Trigger:** start one scan from the TIM6 1 kHz interrupt, or select a timer TRGO as the external trigger for jitter-free sampling.

### 3.6 ADC1: battery

- Keep IN18 on PA4.
- Sampling Time: 810.5 cycles (the divider source impedance is about 32 kΩ).
- Formula: `V_bat = raw / 4095 × 3.3 × (147 / 47)`.
- Read at about 10 Hz.

### 3.7 USART1: HC-06

- Baud rate: 115200, 8N1.
  - Set the HC-06 once: connect at 9600 and send `AT+BAUD8`. The command varies by module firmware.
- DMA: USART1_RX Circular (already set); add USART1_TX Normal.
- NVIC: USART1 global interrupt Enable; receive with `HAL_UARTEx_ReceiveToIdle_DMA()`.

### 3.8 IMU module (I2C1, optionally USART2)

- **I2C1:** PB6 / PB7, Fast Mode 400 kHz.
- **Optional USART2 on PD5 (TX) / PD6 (RX):** only if you also wire the module's UART pads; WitMotion modules stream fused angles over UART by default. PD5 = USART2_TX and PD6 = USART2_RX per the datasheet alternate-function table.

### 3.9 GPIO

- **PG2, PG3, PG4, PG5:** Output push-pull, initial Low; labels `A_IN1`, `A_IN2`, `B_IN1`, `B_IN2`
- **PG6:** Output push-pull, initial **Low**; label `M_STBY` (motors off at boot)
- **PE0, PE1:** Output push-pull, initial Low; labels `LED1`, `LED2`
- **PC13:** GPIO_EXTI13, falling edge, pull-up; label `BTN_START` (after adding the button)
- **PC4:** reset to default (not used)

### 3.10 NVIC priorities

Priority group: 4 bits for pre-emption. A lower number means more urgent.
- **TIM6** (control loop): 1
- **ADC3 / BDMA:** 2
- **USART1 + its DMA streams:** 5
- **I2C1 event / error:** 6
- **EXTI13** (START): 7
- **SysTick** (HAL time base): **0** [corrected]. CubeMX's default here is 15, not 0 as this audit first said. At 15, a HAL function with a timeout called from the TIM6 interrupt would wait forever. Even at 0, avoid blocking HAL calls inside interrupts.

### 3.11 Later: ML

- Add Middleware → STM32Cube.AI / ST Edge AI once a model exists.
- Log these over Bluetooth from the start, so you have training data:
  - sensor vector
  - encoder speeds
  - IMU yaw rate
  - battery voltage
  - PWM commands

---

## 4. Sources

- TI LMR16006 datasheet: https://www.ti.com/lit/ds/symlink/lmr16006.pdf
- TI TPS563201 datasheet: https://www.ti.com/lit/ds/symlink/tps563201.pdf
- ST AN4938: https://www.st.com/resource/en/application_note/an4938-getting-started-with-stm32h74xig-and-stm32h75xig-hardware-development-stmicroelectronics.pdf
- STM32H743ZI datasheet DS12110 (local copy: `stm32h743zi.pdf`)
- AO4407A datasheet: https://www.aosmd.com/res/datasheets/AO4407A.pdf
- SQJ457EP datasheet: https://www.vishay.com/docs/76628/sqj457ep.pdf
- SQJ465EP datasheet: https://www.vishay.com/docs/67996/sqj465ep.pdf
- Pololu 3081 encoder kit: https://www.pololu.com/product/3081
