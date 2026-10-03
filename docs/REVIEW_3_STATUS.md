# Review 3: status check and what is left

Date: 2026-09-15 (evening).

This review checks the schematic against every item in [`REVIEW_2_AND_HOWTO.md`](REVIEW_2_AND_HOWTO.md) and adds new findings. It also gives the real fix for the CubeIDE import problem.

Basis:
- **KiCad ERC:** `kicad-cli sch erc` 10.0.6 on `uc.kicad_sch` found **116 violations** (review 2 had 219).
- **Netlist:** kicad-happy schematic analyzer, 79 components and 149 nets. Every item below was traced pin by pin in that netlist.
- **Libraries:** the custom footprint libraries, plus the KiCad stock crystal footprints.
- **CubeIDE:** workspace metadata and `.metadata/.log` in `~/STM32CubeIDE/workspace_1.18.1`.
- **Not run:**
  - PCB, EMC, thermal and cross-analysis: the PCB is still an unrouted stub.
  - SPICE: nothing to simulate that the regulator datasheets don't already fix.
  - Lifecycle audit: no MPNs are set.
  - Datasheets: not re-read. The checks compare the schematic against review 2, which was based on the datasheets.

---

## 1. CubeIDE: "already exist in the workspace" and "Invalid Input"

### Why it happens

- The CubeIDE workspace `~/STM32CubeIDE/workspace_1.18.1` still has a project called `linefollower` registered at the **old location `~/Desktop/linefollowerk/code`**.
- The folder was moved to `~/Projects/Robotics/linefollowerk`, but Eclipse stores the absolute path, so the old entry points at a folder that no longer exists. Its log says: *"The project description file (.project) for 'linefollower' is missing."*
- Import refuses because a project with the same name already exists. Clicking the dead entry gives the errors, and the `.ioc` fails with "Must be IFileEditorInput".

### Already done

- The leftover `isbadioc=true` line has been removed from `code/linefollower.ioc`. CubeMX writes it after a failed load, and it can make the next load fail too.
- Nothing else in the `.ioc` was changed.

### What you do (CubeIDE can stay open)

1. In **Project Explorer**, find `linefollower`. It may show a red mark or look like a closed folder.
   - If you don't see it: open the Project Explorer's **⋮ / View Menu → Filters and Customization…** and untick **Closed projects**.
2. Right-click `linefollower` → **Delete**.
   - **Make sure "Delete project contents on disk" is NOT ticked.** The old folder is gone anyway, but never tick this box for this project.
   - Click **OK**. This only removes the dead workspace entry.
3. **File → Import… → General → Existing Projects into Workspace → Next.**
4. **Select root directory:** `/home/simsalagrimm-ubuntu/Projects/Robotics/linefollowerk/code`. `linefollower` can now be ticked.
   - Leave **Copy projects into workspace** unticked.
   - Click **Finish**.
5. Expand `linefollower` and double-click `linefollower.ioc`. If it opens as text: right-click → **Open With → STM32CubeMX**.
6. If you move the project folder again in future, repeat steps 2–4.

Then continue with the `.ioc` changes in `HARDWARE_AUDIT.md` §3. **None of them are in the `.ioc` yet.** It still has:
- HSI with no PLL
- PF0–PF7 as GPIO inputs
- PC4 EXTI
- TIM4 PSC 19 / ARR 249
- USART1 at 9600 baud

---

## 2. Review 2 items: status

### Blockers

- **B1 labels cross sheets: FIXED.** No `hier_label_mismatch` errors left. For example, `A_PWM` now joins U1 PD12 and IC1 PWMA.
- **B2 SENS1–SENS8: FIXED.** PF3…PF10 connect to U7 pins 9, 10, 11, 12, 5, 6, 7, 8, as in the FFC mapping.
- **B3 TPS563201 inductor: PARTLY FIXED, with a new wiring error.**
  - Correct:
    - L1 sits between SW and +5V
    - C37 bootstraps VBST to the SW node
    - R8/R9 feedback
    - R10 EN pull-up
    - input capacitors C42, C43 (10 µF) and C44 (100 nF) on VIN
  - **Wrong: see N1 below.** C38 and C41 are on the switch node.
  - **Wrong: see N2 below.** L1 has a relay footprint.
- **B4 Bluetooth TX/RX: FIXED.**
  - `HC_06_RX` joins PB14 (MCU TX) and HC-06 RX.
  - `HC_06_TX` joins PB15 (MCU RX) and HC-06 TX.
- **B5 motors to TB6612: FIXED.** `MA_OUT1/2` and `MB_OUT1/2` reach M1/M2 pins 6 and 5.
- **B6 IC1 footprint: STILL OPEN.**
  - IC1 still says `moje_elementy:SSOP24`.
  - `moje_elementy.pretty` has no SSOP24; the file is in `projekt.pretty`.
  - Change the field to `projekt:SSOP24`.
- **B7 missing parts: FIXED.**
  - **SWD:** J2 1×6. Pins 1 +3V3, 2 SWCLK, 3 GND, 4 SWDIO, 5 NRST, 6 SWO, matching the Nucleo order.
  - **Crystal:** Y1 with C7/C8 10 pF on PH0/PH1. The footprint is wrong, see N3.
  - **START button:** SW2 on PC13 with C9 100 nF.
  - **Fuse:** F1 Nano2 holder.
  - **Battery pads:** J1 solder-wire pads.
  - **Power path:** J1 pin 2 → F1 → VBAT → Q2 drain; Q2 source → VIN; gate → R3 → GND. Correct.

### Major

- **M1 P-FET: FIXED.** Q2 is AO4407A in SOIC-8, symbol IRF7404: pins 1–3 S on VIN, 4 G, 5–8 D on VBAT.
- **M2 C34 footprint: FIXED.** `CP_Elec_8x10`.
- **M3 delete C2: FIXED.**
- **M4 AMS1117 output capacitor: FIXED.** C40 22 µF tantalum, + on +3V3.
- **M5 IMU: WIRING FIXED, module type still open.**
  - U2 VCC on +5V, I²C on PB6/PB7, UART on PD5/PD6.
  - The footprint now has SMD pads at 2.54 mm, rows 15.24 mm apart.
  - **Open:** you described the IMU as a "6 DoF MPU6050 accelerometer + gyroscope module". If it is actually a plain **GY-521** board (pin header VCC, GND, SCL, SDA, XDA, XCL, AD0, INT; I²C only), the symbol, the footprint and the UART lines are all wrong for it. Confirm which module you have.
- **M6 mounting holes: FIXED.** `MountingHole_3.2mm_M3`.
- **M7 missing footprints: PARTLY FIXED.**
  - FB1 and U7 are done.
  - **D2, D3 and D4 still have no footprint.** Use `LED_SMD:LED_0805_2012Metric_Pad1.15x1.40mm_HandSolder`.
- **M8 encoder pads: STILL OPEN (optional).**
  - `projekt:POLOLU_ENCODER_FOOTPRINT` is still 6 through-hole pads at 2.0 mm.
  - It works, but 2.54 mm is easier for wires.

### Minor

- **R4 1 k: FIXED.**
- **Value notation: mostly FIXED.** Still to change:
  - L1 `3u3` → `3.3µH`
  - D2–D4 `LED` → colour
  - SW1 `SW_MEC_5E` → `PTS645`
- **Power symbols: STILL OPEN (cosmetic).** 5 × `power:+3.3V` and 7 × `power:VCC` remain. Their Value fields (VIN, VBAT, +3V3, +3V3A) name the nets, so the connections are correct.
- **TB6612 and FFC pin types: STILL OPEN.** Hidden for now by other errors; see §4.
- **Title blocks: STILL EMPTY** on both sheets.
- **Library tables (L-01): STILL OPEN.**
  - `sym-lib-table` only lists the `C:/Users/rataj/...` library.
  - `linefolower_stm` is not registered.
  - `Elements2` (used by U6 and U7) does not exist on disk.
  - This causes the 12 `lib_symbol_issues` / `footprint_link_issues` warnings. Do the library merge in review 2 section F.

---

## 3. New findings

**N1 BLOCKER: the TPS563201 output capacitors are on the switch node.**
- C38 and C41 (22 µF each) have their pin 2 on the same net as U6 SW, L1 pin 2 and C37. That net is the switch node.
- At 500 kHz the regulator would drive 44 µF straight to ground on every switching edge, so it will not regulate and can be damaged.
- Meanwhile the +5V net has no output capacitor at all, only C39 at the AMS1117 input.
- **Fix:**
  1. Disconnect C38 pin 2 and C41 pin 2 from the SW / L1 pin 2 side.
  2. Connect both to the **+5V side of L1** (L1 pin 1, the net with R8 and U3 VI).
- **Result after the fix:**
  - SW node: U6 pin 2, L1 pin 2 and C37 only.
  - +5V net: L1 pin 1, C38, C41, R8, U3 VI, C39, U2 and U5.

**N2 BLOCKER for the PCB: L1 has a relay footprint.**
- L1's footprint is `Relay_THT:Relay_DPDT_Finder_30.22`.
- Pick a shielded 3.3 µH inductor, I_sat ≥ 3 A, and its footprint. For example:
  - `Inductor_SMD:L_Bourns_SRN4018`, or
  - `Inductor_SMD:L_Coilcraft_XAL4020-222`, which has the right size; check the part's datasheet land pattern.
- Buy the exact inductor whose footprint you choose.

**N3 MAJOR: crystal symbol and footprint don't match.**
- Y1 uses `Device:Crystal_GND24` (4 pins: pins 2 and 4 are GND), but its footprint is `Crystal_SMD_5032-2Pin_5.0x3.2mm_HandSoldering`, which has only 2 pads.
- Updating the PCB will report missing pads 2 and 4.
- **Pick one:**
  - **Easier to hand-solder (recommended):** change the symbol to `Device:Crystal` (2 pins) and keep the 2-pin HandSoldering footprint. Buy a 2-pad 5×3.2 mm 25 MHz crystal.
  - Or keep `Crystal_GND24` and use `Crystal:Crystal_SMD_5032-4Pin_5.0x3.2mm`. Buy a 4-pad crystal.

**N4 MAJOR: 80 unused MCU pins without no-connect flags.**
- All 80 `pin_not_connected` errors are unused U1 GPIOs. On the root sheet: press **Q** (no-connect flag) and click each unused pin end.
- **Keep these free** rather than flagging them, if you may use them:
  - PA11/PA12 (USB)
  - PA9/PA10 (spare UART)
  - PB10/PB11 (I2C2)
- Unused pins:
  - **PA:** PA2, PA3, PA5, PA8–PA12, PA15
  - **PB:** PB0–PB2, PB4, PB5, PB8–PB13
  - **PC:** PC0, PC1, PC2_C, PC3_C, PC4–PC12, PC14, PC15
  - **PD:** PD0–PD4, PD7–PD11, PD14, PD15
  - **PE:** PE2–PE15
  - **PF:** PF0–PF2, PF11–PF15
  - **PG:** PG0, PG1, PG7–PG15

**N5 MINOR (ERC error): no PWR_FLAG.**
- 5 `power_pin_not_driven` errors, on:
  - GND
  - VBAT
  - VIN
  - +5V
  - +3V3A
- They come from a connector pin, a fuse, a P-FET, an inductor and a ferrite, which ERC does not treat as sources.
- **Fix:** place one `power:PWR_FLAG` on each of those nets. The board works either way, but this clears the errors.

**N6 MINOR: 18 off-grid endpoints and 1 dangling wire on `Peripherials`.**
- They are around U7 pin 19, C5, C6 and power symbols #PWR044, #PWR045, #PWR047, #PWR072.
- The netlist shows they are connected today, but off-grid ends break easily when you move things.
- **Fix:**
  1. Select those parts → right-click → **Align Elements to Grid**.
  2. Delete the 0.09 mm stray wire.
  3. Re-run ERC.

**N7 QUESTION: C6 is 100 µF in 0805 on +3V3.**
- A 100 µF ceramic in 0805 is rare, expensive and loses most of its capacitance at 3.3 V.
- If C5 (10 µF) and C6 are the pair near the FFC connector (review 2, section B), C6 should be **100 nF**.
- If you really want bulk on +3V3, use a 1206 22 µF.

**N8 CHECK: U7 is now the 0.5 mm pitch TE FFC footprint.**
- Measure the cable first: 20 contacts over about 10 mm is 0.5 mm; about 20 mm is 1.0 mm.
- 0.5 mm is much harder to hand-solder; use flux and drag-solder.
- Also confirm whether the contacts are on the same side or opposite sides at the two ends.

**N9 COSMETIC:** the SWO label is spelled `SW0` (zero) on PB3 / J2 pin 6.

---

## 4. What is left, in order

- [ ] **CubeIDE:** remove the stale project and re-import (§1)
- [ ] **N1:** move C38/C41 to +5V
- [ ] **N2:** real inductor footprint for L1
- [ ] **N3:** crystal symbol/footprint match
- [ ] **B6:** IC1 footprint `projekt:SSOP24`
- [ ] **M7:** LED footprints D2, D3, D4
- [ ] **M5:** confirm the IMU module type (GY-521 or WitMotion-type)
- [ ] **N7:** C6 value
- [ ] **N8:** FFC pitch and contact side
- [ ] **N4, N5, N6:** no-connect flags, PWR_FLAGs, grid
- [ ] **Libraries:** merge (review 2 section F), which also fixes the `Elements2` / `C:/` table problems
- [ ] **ERC → 0 errors**
- [ ] **`.ioc`:** apply `HARDWARE_AUDIT.md` §3, regenerate code
- [ ] **PCB:** update from schematic, 4-layer layout
