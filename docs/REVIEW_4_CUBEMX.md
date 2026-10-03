# Review 4: schematic and .ioc status, plus CubeMX explained

Date: 2026-09-22.

Checked:
- **Schematic:** KiCad ERC on `uc.kicad_sch`, plus the kicad-happy netlist compared with review 3.
- **`code/linefollower.ioc`:** every line.
- **Generated code:** `main.c` (clock and MPU), `adc.c`, `tim.c`, and `STM32H743ZITX_FLASH.ld`.

---

## 1. Schematic: what's left

**Fixed since review 3:**
- **N1:** C38 and C41 are now on +5V; the switch node is only U6 SW, L1 and C37.
- **B6:** IC1 footprint is now `projekt:SSOP24`.
- **N7:** C6 is now 100 nF.
- **N4:** all unused MCU pins have no-connect flags (0 `pin_not_connected` errors).

ERC now: **5 errors, 31 warnings** (was 116).

**Still to do:**
- **N2 (blocks the PCB):** L1 still has the relay footprint `Relay_THT:Relay_DPDT_Finder_30.22`. Choose the real 3.3 µH shielded inductor (I_sat ≥ 3 A) and its footprint.
- **N3:** Y1 is still `Device:Crystal_GND24` (4 pins) on a 2-pad footprint. Change the symbol to `Device:Crystal`, or use the 4-pad `Crystal_SMD_5032-4Pin_5.0x3.2mm`.
- **M7:** D2, D3 and D4 still have no footprint. Use `LED_SMD:LED_0805_2012Metric_Pad1.15x1.40mm_HandSolder`.
- **N5 (the 5 ERC errors):** add one `power:PWR_FLAG` each on GND, VBAT, VIN, +5V and +3V3A.
- **N6:** 18 off-grid endpoints and 1 stray 0.09 mm wire on the `Peripherials` sheet, near U7 pin 19, C5, C6 and #PWR044/045/047/072.
- **Libraries:** `sym-lib-table` still only has the `C:/Users/rataj/...` entry; `linefolower_stm` and `Elements2` are not registered. That is the other 12 ERC warnings. Merge per review 2 section F.
- **Cosmetic:**
  - label `SW0` → `SWO`
  - L1 value `3u3` → `3.3µH`
  - D2–D4 value → colour
  - SW1 value → `PTS645`
  - title blocks empty
- **Open questions:**
  - IMU module type (WitMotion-type or GY-521)
  - FFC pitch (the footprint is 0.5 mm)
  - optional 2.54 mm encoder pads

---

## 2. .ioc: what is done and what is left

**Done and correct:**
- **Clock:** HSE crystal 25 MHz → PLL1 /5 ×192 /2 → SYSCLK 480 MHz. The generated code confirms it:
  - `RCC_HSE_ON` (crystal)
  - `PWR_LDO_SUPPLY`
  - `PWR_REGULATOR_VOLTAGE_SCALE0`
  - `FLASH_LATENCY_4`
- **Bus clocks:** AHB 240 MHz, all APB 120 MHz, timer clocks 240 MHz.
- **ADC clock:** PLL2P = 36 MHz.
- **ICache and DCache:** enabled.
- **Pins:** PF0–PF2 are reset, and PC4 is gone.
- **ADC1:** IN18 on PA4, 810.5 cycles.
- **TIM2 / TIM3:** encoder TI1+TI2 with filter 6 on both channels; periods 0xFFFFFFFF / 0xFFFF.
- **TIM4:** PSC 11, ARR 999, preload on → 20 kHz.
- **TIM6:** PSC 239, ARR 999 → 1 kHz, NVIC on, priority 1.
- **USART1:** 115200; RX DMA circular (DMA1 Stream 0) and TX DMA normal (DMA1 Stream 1).
- **I2C1:** Fast mode.
- **PC13:** EXTI falling edge with pull-up, label BTN_START; EXTI15_10 priority 7.
- **Outputs:** PE0/PE1 LEDs and PG2–PG6 motor pins; the default output level is Low.
- **NVIC:**
  - USART1 and both DMA streams: 5
  - I2C1: 6
  - priority group: 4 bits

**Still to do, in this order:**
1. **ADC3 channels (why PF3, PF5, PF7, PF9 are yellow):**
   - Each pin is assigned its signal (ADC3_INP5, INP4, INP3, INP2), but the matching channel is not switched on in the ADC3 Mode panel.
   - Only IN6, IN7, IN8 and IN9 are enabled.
   - In **Analog → ADC3 → Mode**, set **IN2, IN3, IN4 and IN5 to "INx Single-ended"**. The four pins turn green.
   - Your PF reset was otherwise correct: PF0–PF2 are free, and PF3–PF10 all carry ADC3 signals.
2. **ADC3 parameters** (see §3.6 for why Scan was greyed out):
   - Number Of Conversion = **8**. Scan Conversion Mode then switches itself on.
   - Rank 1…8 = channel 5, 9, 4, 8, 3, 7, 2, 6 (SENS1…SENS8), each **387.5 cycles**. Right now only one rank exists, on channel 9 at 1.5 cycles.
   - End Of Conversion Selection = End of sequence of conversion.
3. **ADC3 DMA:**
   - ADC3 → **DMA Settings → Add → ADC3**. CubeMX picks **BDMA Channel 0** by itself.
   - Mode **Circular**, data width **Half Word** / **Half Word**.
   - Then Parameter Settings → **Conversion Data Management Mode = DMA Circular Mode**.
   - After this, **"BDMA channel0 global interrupt" appears in NVIC**. Enable it at priority **2**.
   - It is not in NVIC yet because no ADC3 DMA request exists; CubeMX only lists interrupts for things you have configured.
4. **MPU region for the ADC3 buffer** (see §3.3):
   - Cortex_M7 → Parameter Settings → **MPU Region 1**
   - Enable
   - Base Address **0x38000000**
   - Size **64KB**
   - SubRegion Disable **0x0**
   - TEX **level 1**
   - Access **ALL ACCESS PERMITTED**
   - Instruction Access **DISABLE**
   - Shareable **DISABLE**
   - Cacheable **DISABLE**
   - Bufferable **DISABLE**
   - Leave region 0 as CubeMX made it. It is ST's standard "block unused address space" region.
5. **TIM4 Clock Source → Internal Clock.** See §3.4.
6. **SysTick priority:**
   - The .ioc has SysTick at **15** (lowest). The audit wrongly said 0 is the default.
   - Set **Time base: System tick timer** to preemption priority **0**.
   - Why: at 15, any HAL function with a timeout (`HAL_Delay`, blocking `HAL_I2C_…`, `HAL_UART_Transmit`) called from inside the TIM6 control-loop interrupt would wait forever, because the tick can't interrupt TIM6.
7. **Optional:**
   - **TIM6 → Trigger Event Selection = Update Event** and **ADC3 → External Trigger = Timer 6 trigger out event**, rising edge. Then the ADC scans exactly at 1 kHz by itself and you never start it from code.
   - **USART2** on PD5/PD6, only if you use the IMU's UART.
8. **Regenerate the code.**

**The ADC3 buffer, without editing the linker script:**
- CubeMX rewrote `STM32H743ZITX_FLASH.ld` when you regenerated on 22 Sep (its timestamp matches), so an edit there may be lost at the next regeneration.
- Nothing else is placed in RAM_D3, so put the buffer at a fixed address instead:
  ```c
  /* USER CODE BEGIN PV */
  /* RAM_D3 (SRAM4) start: the only RAM the BDMA can reach. Made non-cacheable by MPU region 1. */
  #define ADC3_BUF ((volatile uint16_t *)0x38000000UL)
  /* USER CODE END PV */
  ```
- Start it with:
  ```c
  HAL_ADC_Start_DMA(&hadc3, (uint32_t *)ADC3_BUF, 8);
  ```
- Code between `USER CODE BEGIN` and `USER CODE END` survives regeneration.

---

## 3. Answers and explanations

### 3.1 Your quick questions

- **TIM2 global interrupt?** **No.**
  - In encoder mode the timer counts the pulses by itself in hardware.
  - Your TIM6 control loop just reads the counter every 1 ms.
  - TIM2 is 32-bit, so it effectively never overflows; TIM3 is 16-bit, but reading the difference as `int16_t` handles wrap-around.
  - An interrupt would only fire on overflow, and you don't need that.
- **Both DMA1 streams at priority 5?** **Yes.**
  - They are Stream 0 (RX) and Stream 1 (TX), both belonging to USART1, so they get the same priority as USART1 (5).
  - That is what the .ioc has now.
- **Is EXTI13 the "EXTI line[15:10]" interrupt?** **Yes.**
  - Lines 10–15 share one interrupt vector, `EXTI15_10_IRQn`.
  - Enabling it was right: without it the button would set a flag that nothing ever reads.
  - It is at priority 7.
- **TIM4 "ETR as clearing source" red:** **leave it unchecked.**
  - ETR is an external-trigger input pin. For TIM4 it is **PE0**, which is your LED, so CubeMX shows it red (pin conflict).
  - You don't need it; it is for cutting PWM with an external signal.
- **TIM4 Clock Source yellow:** set it to **Internal Clock**.
  - It works even when left at Disable, because a timer runs on its internal clock after reset. The generated code is just missing the explicit setting, and CubeMX warns about it.
  - Optional: TIM4 has **Fast Mode = Enable** on both channels. It only matters in one-pulse or trigger modes, so it is harmless, but **Disable** is the normal setting.

### 3.2 Clock configuration, term by term

**Oscillators (the clock sources):**
- **HSI (High-Speed Internal):** a 64 MHz RC oscillator inside the chip. It runs at reset, needs no parts, but drifts by about 1 % with temperature and voltage. Too inaccurate for precise UART or timing, and too slow for full speed.
- **HSE (High-Speed External):** your 25 MHz crystal on PH0/PH1. Accurate to about ±20 ppm (0.002 %). "Crystal/Ceramic Resonator" means the chip drives the crystal itself; "BYPASS" would mean a ready-made clock signal comes in.
- **CSI:** a 4 MHz low-power internal oscillator. Unused.
- **HSI48:** a 48 MHz internal oscillator for USB and the random-number generator. Unused except as the RNG source.
- **LSI:** 32 kHz internal oscillator, for the watchdog.
- **LSE:** 32.768 kHz external watch crystal, for the real-time clock. Not fitted, disabled.

**PLL (Phase-Locked Loop):**
- A PLL makes a high frequency from a low one. It has a divider at the input, an oscillator (VCO) that runs at an exact multiple of the input, and dividers after it.
- The H743 has three PLLs, each with three outputs, **P**, **Q** and **R**. Each output can feed different things.
- **PLL Source Mux:** which oscillator feeds all three PLLs. You set HSE.
- **DIVM1 (/5):** input divider, 25 MHz → 5 MHz. The PLL input must be in its allowed range; CubeMX sets the range for you.
- **DIVN1 (×192):** the multiplier, 5 MHz × 192 = **960 MHz VCO**. The VCO has an allowed range too; CubeMX turns the box red if you leave it.
- **FRACN:** a fractional add-on to DIVN for odd frequencies (audio). 0 here.
- **DIVP1 (/2):** 960 / 2 = **480 MHz**. This is the output that can become the system clock.
- **DIVQ1:** 960 / 2 = 480 MHz, an optional "kernel clock" for SPI1–3, SAI, SDMMC and FDCAN. Not used by anything you have enabled.
- **DIVR1:** the R output of PLL1, also 480 MHz.
  - On the H743 it drives the **debug trace clock** (for SWO/trace output) and can be selected by a few kernel-clock muxes. Nothing in your design uses it.
  - Leave it at the default /2. It changes nothing unless you later use SWO trace, and then the debugger only needs to know the core clock (480 MHz).
- **PLL2** (DIVM2 /5, DIVN2 ×72 = 360 MHz VCO, DIVP2 /10 = **36 MHz**): feeds the ADC only. Its Q and R outputs are unused.
- **PLL3:** unused. Its odd numbers such as 50.39 MHz don't matter, because it is off.

**From SYSCLK down to the buses:**
- **System Clock Mux:** chooses SYSCLK from HSI, CSI, HSE or PLL1P. You chose PLLCLK, which is PLL1P = 480 MHz.
- **D1CPRE (/1):** the divider from SYSCLK to the **CPU core** (Cortex-M7). 480 MHz.
- **HPRE (/2):** the **AHB/AXI bus** divider. 240 MHz is the maximum for the buses, flash and RAM.
- **D1PPRE, D2PPRE1, D2PPRE2, D3PPRE (/2 each):** the four **APB peripheral bus** dividers: APB3, APB1, APB2 and APB4. The maximum is 120 MHz.
- **Timer clocks:** when an APB divider is not /1, the timers on that bus get **2× the APB clock**, so 240 MHz. That is why TIM4 PSC 11 / ARR 999 gives 240 MHz / 12 / 1000 = 20 kHz.

**D1, D2 and D3 (power domains):** the H7 is split into three internal domains, each with its own RAM.
- **D1:** CPU, flash, AXI SRAM (RAM_D1, 0x24000000, where your variables live).
- **D2:** DMA1/DMA2 and most peripherals (USART1, TIM2–TIM6, I2C1, ADC1). RAM_D2 is at 0x30000000.
- **D3:** a small always-on domain with **BDMA**, **ADC3** and SRAM4 (RAM_D3, 0x38000000).
- Why it matters to you: ADC3's DMA (the BDMA) can only reach RAM_D3.

**Right-hand side of the Clock tab:**
- **Kernel clock muxes:** many peripherals can run their internal logic from a different clock than their bus clock. Examples are the "ADC mux", "USART16 mux" and "I2C123 mux". Leave the defaults unless a box goes red. The ADC mux is PLL2P, which you set.
- **CKPER:** a spare "peripheral clock" (HSI by default) that some kernel muxes can choose.
- **MCO1 / MCO2:** can put a clock out on a pin for measuring. Unused.

**Power settings:**
- **VOS (Voltage Output Scaling):** how high the internal regulator sets the core voltage.
  - **Scale 3** is the lowest voltage and power, and allows only low speeds.
  - **Scale 0** is the highest voltage and allows **480 MHz**. It uses more power and heat, and the datasheet requires it for 480 MHz on revision V.
- **Supply Source `PWR_LDO_SUPPLY`:** the core voltage comes from the internal linear regulator, which is why the board has the two 2.2 µF VCAP capacitors. Other H7 parts have an SMPS option; yours uses the LDO.
- **Flash latency (wait states):** flash is slower than the CPU, so the CPU waits a few cycles per flash read. CubeMX set 4 automatically for 240 MHz AXI at Scale 0.

### 3.3 ICache, DCache and the MPU

- **Caches:** the Cortex-M7 has a **16 KB instruction cache (ICache)** and a **16 KB data cache (DCache)**. These are tiny, very fast memories next to the core.
  - At 480 MHz the flash can't keep up, so without ICache the CPU would spend most of its time waiting for instructions.
  - DCache does the same for data in RAM.
  - Both give a large speed-up, so keep them on.
- **The DCache catch:** DMA writes to RAM directly and doesn't know about the cache.
  - **Reading** (ADC or UART RX): the CPU may read an old copy from its cache instead of the new value that the DMA put in RAM. The ADC values would look frozen.
  - **Sending** (UART TX): new data the CPU wrote may still sit only in the cache, so the DMA sends old RAM content.
  - **Fixes:**
    - Make the DMA buffer memory **non-cacheable** with the MPU. This is simplest, and it is item 4 in §2 for the ADC3 buffer.
    - Or call `SCB_InvalidateDCache_by_Addr()` before reading a DMA buffer, and `SCB_CleanDCache_by_Addr()` before starting a DMA transmit. The buffer must be 32-byte aligned. This is what you will do for the USART1 buffers in RAM_D1.
- **MPU (Memory Protection Unit):** hardware that sets rules for address ranges: whether they can be accessed, executed or cached.
  - CubeMX already created **region 0**, which blocks the unused address space. ST recommends it, because it stops the M7 from making stray speculative reads there.
  - **Region 1**, which you add, marks the 64 KB RAM_D3 as non-cacheable.

### 3.4 Timers

- **Input filter** (TIM2/TIM3 encoder channels):
  - The timer only accepts a change on an encoder line after it has seen the new level several times in a row. That rejects short spikes from motor noise.
  - Value **6** means: sample at timer clock / 4 = 60 MHz and require 6 equal samples. The pulse must be stable for about **0.1 µs**.
  - Your encoder edges are at least ~100 µs apart even at full motor speed, so you could raise the filter as high as **15** (about 1 µs) for more noise immunity without ever losing counts.
  - 6 is fine as it is.
- **Clock Source:** where the counter's ticks come from.
  - "Internal Clock" is the 240 MHz timer clock.
  - "ETR2" would be an external pin.
  - Encoder mode doesn't need it, because the encoder signals are the clock.

### 3.5 BDMA

- **What it is:** the "basic DMA" in domain D3. It serves D3 peripherals, and ADC3 is one of them on the H743.
- **Memory it can reach:** only RAM_D3 (0x38000000).
- **Why you don't see it:** CubeMX only lists it in NVIC after you add the ADC3 DMA request (§2 item 3).

### 3.6 Scan Conversion Mode greyed out

- On the H7, CubeMX controls this field itself.
- It turns on automatically when **Number Of Conversion > 1**, and you can't click it directly.
- Set Number Of Conversion to 8, and the Rank 1–8 fields appear.

**ADC timing:**
- Revision V chips halve the ADC clock inside the ADC, so the effective ADC clock is about 18 MHz. At 387.5 cycles that is about 22 µs per sensor, or **about 0.2 ms for all 8**.
- That is still well within the 1 ms control loop.

---

## 4. Update 2026-09-23

**Decisions confirmed by the owner:**
- The IMU is a WitMotion module.
- The FFC pitch is 0.5 mm (measured).
- The encoder footprint stays as it is for now.

**.ioc: complete.**
- Two MPU regions: region 0 blocks unused memory (0x0, 4GB, 0x87, No Access), and region 1 makes RAM_D3 non-cacheable (0x38000000, 64KB, TEX1). Region 1 is higher, so it wins where they overlap. Correct.
- ADC3: 8 ranks at 387.5 cycles, DMA circular on BDMA Channel 0, triggered by TIM6 TRGO.
- NVIC: BDMA priority 2, SysTick 0.
- TIM4: internal clock source.
- Optional: TIM4 Fast Mode is still Enable. It is harmless.
- Optional: USART2 on PD5/PD6 for the WitMotion UART. Its default baud rate is usually 115200; check the module manual.

**Schematic:**
- **Fixed:**
  - PWR_FLAGs added; 0 ERC errors
  - LED footprints (1206 HandSolder)
  - Y1 footprint is now the 4-pad `Crystal_SMD_5032-4Pin_5.0x3.2mm`, which matches `Crystal_GND24`: pads 1/3 are the crystal, 2/4 are GND. Buy a 4-pad 5032 crystal.
  - L1 value is now `3.3uH`, SW1 value is now `PTS645`
- **L1 part:** the Vishay **IHLP2525CZER3R3M01** is suitable:
  - 3.3 µH, shielded
  - 28 mΩ
  - current rating (6 A) far above the TPS563201's 3 A
  - 6.5 × 6.5 mm, easy to hand-solder

  Footprint: **`Inductor_SMD:L_Vishay_IHLP-2525`**. **L1 still has the relay footprint; change it.**
- **Off-grid warnings (17):**
  - The cause is the **U7 symbol**, not its footprint. Its pins sit 0.635 mm off the 1.27 mm grid; for example, pin 1 is at x = 23.495 mm.
  - The wires, C5, C6 and the power symbols connected to it inherited the offset.
  - **Fix:** replace U7 with the stock **`Connector_Generic:Conn_01x20`** (right-click → Change Symbol…). It keeps pin numbers 1–20, the footprint, and all-passive pin types.
  - Then select the connected wires, C5, C6 and power symbols → right-click → **Align Elements to Grid**.
- **Stray wire:** a 0.09 mm wire on the `Peripherials` sheet at about (19.05 mm, 40.64 mm). Delete it.
- **SW0:** two labels on the root sheet `uc`:
  - one at about (152.4, 189.23) mm, near J2
  - one at about (144.78, 40.64) mm, at PB3

  Rename both to `SWO` (letter O).
- **Library location (new, major):**
  - The merged `linefollower` symbol library was saved as a **global** library at `~/Projects/Hexapod/PCB/new_PCB/new_PCB/linefollower.kicad_sym`, inside the Hexapod project.
  - The linefollower project doesn't contain its own symbols. It breaks if the Hexapod folder moves or the project is opened on another computer.
  - **Fix:**
    1. Close KiCad.
    2. Copy that file into the `linefollowerk` folder.
    3. In **Preferences → Manage Symbol Libraries**, remove `linefollower` from the **Global** tab.
    4. On the **Project Specific** tab, add it with path `${KIPRJMOD}/linefollower.kicad_sym`, and remove the old `Elementy_linefollower` (`C:/…`) entry.
    5. Reopen and run ERC.
  - Footprints are still in `projekt` / `moje_elementy`; merge them later.
- **ZC142200:** the HC-06 symbol (U5).
  - It is the SnapEDA part number (manufacturer "YKS") of a 6-pin HC-05/HC-06 breakout.
  - The footprint `projekt:MODULE_ZC142200` is a 1×6 2.54 mm header: 1 STATE, 2 RXD, 3 TXD, 4 GND, 5 VCC, 6 EN.
  - Check that your HC-06 has 6 pins in that order; many HC-06 boards have only 4 (VCC, GND, TXD, RXD).

---

## 5. Full re-check 2026-09-23 (evening)

**ERC:**
- **0 errors, 0 warnings** on both sheets. Checked on a copy with matching project/root names, so the project library tables were loaded.
- Library tables: only `linefollower` (symbols) and `linefollower` (footprints), both `${KIPRJMOD}`.

**Nets checked pin by pin: all correct, except J3.**
- TPS563201: SW node, VBST, feedback and EN
- +5V, VIN, VBAT, P-FET gate
- crystal
- HC-06 TX/RX crossed
- IMU I²C + UART
- SWD, including `SWO` (the SW0 rename is done), NRST, START
- TB6612 control lines and outputs to the motors
- encoders, battery divider, +3V3A, BOOT0

**BLOCKER, new: J3 (the FFC, formerly U7) pins 5–8 are shifted.** Probably from the symbol swap to `Conn_01x20`.
- Now:
  - pin 5 = SENS6
  - pin 6 = SENS7
  - pin 7 = SENS8
  - pin 8 = SENS5
- Must be (confirmed by owner):
  - pin 5 = SENS5
  - pin 6 = SENS6
  - pin 7 = SENS7
  - pin 8 = SENS8
- Fix the four labels at J3 pins 5–8. Pins 9–12 (SENS1–SENS4), 1 (+3V3), 20 (GND) and the no-connects are correct.

**3D model:**
- `ZC142200.step` (HC-06) was moved to `archive/`, but `linefollower:MODULE_ZC142200` still loads `${KIPRJMOD}/3D models/ZC142200.step`. Move it back to `3D models/`.
- `3D models/` also holds non-model leftovers you can archive:
  - `SOT95P280X110-6N.kicad_mod`
  - `SSOP24.kicad_mod`
  - `TB6612FNG.kicad_sym`
  - `LMR16006XDDCR.*` (that part was removed)

**PCB:**
- All 79 parts are present, but 6 footprints still point to the old libraries: U6, U2, U5, IC1, M1, M2 (`projekt:` / `moje_elementy:`).
- Run **Tools → Update PCB from Schematic (F8)** in the PCB Editor, after fixing J3.
- The board has no outline and no copper yet.

**Root-folder leftovers** (move to `archive/`):
- `linefolower_stm.kicad_sym`
- `linefollowerk.kicad_sch` (the old sheet)

**Title blocks:** still empty on both sheets.

**.ioc:**
- Settings are complete: the clock at 480 MHz, VOS0, and MPU regions 0 and 1 are in the generated code.
- ADC3 ×8, BDMA and the TIM6 trigger are set; so are the NVIC priorities.
- **But:**
  - The `.ioc` was saved on 23 Sep 19:59, and the code was last generated on 22 Sep. Open the `.ioc` and click **Generate Code** once so they match.
  - The file ends with `isbadioc=true` again. CubeIDE's log shows no error. If the `.ioc` opens normally, saving/generating rewrites it. If it refuses to open, delete that line (with CubeIDE closed).
- **Optional:** USART2 for the WitMotion UART; TIM4 Fast Mode → Disable.
