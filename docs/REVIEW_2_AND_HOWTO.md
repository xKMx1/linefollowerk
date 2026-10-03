# Review 2, part choices and how-to

Date: 2026-09-15.

This review covers the schematic after the owner:
- made `uc.kicad_sch` the root sheet with `peripherials.kicad_sch` as its child
- wired the power tree

Basis:
- KiCad 10.0.6 ERC on the root: 219 violations
- kicad-happy analyzer netlist
- PDF render of both sheets
- datasheets listed at the end

Everything is written as lists because pipe tables do not render for the owner.

---

## A. Schematic review 2

### What is now correct

- The hierarchy exists: root `uc` with child `Peripherials`.
- +3V3 is a single power-symbol net fed by the AMS1117-3.3.
- AMS1117 pins are correct: pin 1 GND, pin 2 VO, pin 3 VI.
- PDR_ON is tied to +3V3.
- **TB6612:** VM pins 13, 14 and 24 on VIN; logic VCC pin 20 on +3V3; all GND/PGND pins on GND; STBY on `M_STBY` with R11 10 k pull-down.
- Encoders: VCC on +3V3, GND, and the A/B signals labelled.
- Battery divider R6 100 k / R7 47 k with C33 100 nF, on PA4.
- LEDs D3 and D4 with 1 k resistors on PE0 and PE1.
- TPS563201 feedback R8 54.9 k / R9 10 k, which gives 5.0 V from TI's table.
- TPS563201 EN pulled to VIN through R10 100 k.
- FFC signal mapping, with no-connect flags on the unused pins.

### Blockers (the board will not work until these are fixed)

**B1: Nothing crosses the sheet boundary.**
ERC reports 23 `hier_label_mismatch` errors. Hierarchical labels only connect through **sheet pins** on the sheet symbol. The `Peripherials` box on the root has none, and on the root sheet itself hierarchical labels mean nothing. The simplest fix for a two-sheet design is global labels:
1. Open the root sheet `uc`. Right-click one hierarchical label (e.g. `A_PWM`) → **Change To → Global Label**. Repeat for every label next to U1.
2. Open the child sheet (double-click the `Peripherials` box). Do the same for every hierarchical label there.
3. Names must be identical on both sheets, including upper and lower case.
4. Run **Inspect → Electrical Rules Checker**. The `hier_label_mismatch` errors should be gone.

*Alternative, if you prefer hierarchical labels:* keep them on the child sheet only. On the root, right-click the sheet box → **Sheet Pins → Sync Sheet Pins** (it imports every hierarchical label as a pin). Then wire each sheet pin to U1 with a wire or a local label.

**B2: SENS1–SENS8 are not labelled at the MCU.**
On the root sheet, add a global label on each pin:
- `SENS1` on PF3 (pin 13)
- `SENS2` on PF4 (pin 14)
- `SENS3` on PF5 (pin 15)
- `SENS4` on PF6 (pin 18)
- `SENS5` on PF7 (pin 19)
- `SENS6` on PF8 (pin 20)
- `SENS7` on PF9 (pin 21)
- `SENS8` on PF10 (pin 22)

**B3: The TPS563201 has no inductor.** SW (pin 2) is wired straight to +5V, so the regulator would short its switch node to its own output. Correct circuit (TI datasheet Table 7-2, 5 V row):
- **Pin 3 VIN** → VIN net. Place **2 × 10 µF (25 V X7R, 1206) + 1 × 100 nF** right at pins 3 and 1. These are the "input capacitors at the regulator".
- **Pin 1 GND** → GND.
- **Pin 5 EN** → R10 100 k to VIN (as drawn).
- **Pin 2 SW** → a new net `SW5V`, then **L1 3.3 µH**, then +5V.
- **Pin 6 VBST** → C37 100 nF → **SW5V net**, not +5V. VBST must bootstrap from the switch node.
- **+5V output** → **2 × 22 µF (10 V or higher, X5R/X7R, 1206)**. TI allows 20–68 µF total. You have one (C38); add a second.
- **Pin 4 VFB** → R8 54.9 k from +5V, R9 10 k to GND (as drawn).
- **Inductor choice:**
  - value 3.3 µH, shielded
  - saturation current ≥ 3 A (the 5 V load is small, but the peak ripple is about 1 A at 8.4 V in, and a margin protects against start-up surges)
  - about 4×4 mm to 5×5 mm
  - KiCad footprints exist for Bourns SRN4018, Coilcraft XAL4020, Cenker CKCS4020/5040 and others; search the footprint chooser for `4020` or `5040`
  - on TME, filter: SMD power inductor, 3.3 µH, shielded, I_sat ≥ 3 A, 4×4 or 5×5 mm. Pick one whose footprint is in KiCad.

**B4: The Bluetooth TX and RX are not crossed.** Label `HC_06_TX` is on MCU PB14 (the MCU **TX**) and also on HC-06 pin 3 (the module **TX**), so two outputs are shorted and no data arrives. Rename:
- MCU PB14 (USART1_TX) → label `BT_RX`; HC-06 pin 2 (RX) → label `BT_RX`
- MCU PB15 (USART1_RX) → label `BT_TX`; HC-06 pin 3 (TX) → label `BT_TX`

Naming each wire after the module pin it goes to avoids this mistake in future.

**B5: The motors are not connected to the TB6612 outputs.** `MA_OUT1`, `MA_OUT2`, `MB_OUT1` and `MB_OUT2` only touch IC1. Add the same labels on the encoder symbols:
- M1 pin 6 "M1" → `MA_OUT1`
- M1 pin 5 "M2" → `MA_OUT2`
- M2 pin 6 "M1" → `MB_OUT1`
- M2 pin 5 "M2" → `MB_OUT2`

If a motor spins backwards, swap it in firmware rather than on the board.

**B6: IC1's footprint does not exist.** IC1 uses `moje_elementy:SSOP24`, but `SSOP24.kicad_mod` is in `projekt.pretty`. Change the Footprint field to `projekt:SSOP24`, or better, the consolidated library from section F.

**B7: Still missing entirely:**
- SWD connector (C5)
- HSE crystal and its two capacitors (C6)
- START button (C7)
- fuse and battery pads (C2, C1)

### Major

**M1: Q1 is still the AO3401A.** Replace it (see C3).

**M2: C34 470 µF footprint is too small.** A 470 µF / 16 V aluminium electrolytic does not fit a 5×5.4 mm can.
- Use `Capacitor_SMD:CP_Elec_8x10` for 470 µF / 16 V, or
- use 220 µF / 16 V in `Capacitor_SMD:CP_Elec_6.3x7.7`.

Either is fine with a 200 mAh pack.

**M3: C2 22 µF polarized is redundant.** C34 covers bulk capacitance, so delete C2. The motor-side capacitors are then:
- **C34** 470 µF or 220 µF electrolytic: bulk
- **C10** 10 µF ceramic at VM1 (pin 24)
- **C35 and C36** 10 µF ceramic at VM2/VM3 (pins 13–14)
- **C1** 100 nF

All ceramics: 1206, **25 V**, X7R. A 10 V part loses most of its capacitance at 8.4 V.

**M4: The AMS1117 output capacitor.** The AMS1117 datasheet asks for a **22 µF solid tantalum** capacitor on the output for stability. Low-ESR ceramics alone can make some AMS1117 batches oscillate.
- Change C40 to 22 µF tantalum, 6.3 V or higher, in `Capacitor_Tantalum_SMD:CP_EIA-3528-21_Kemet-B_HandSolder`.
- Mind the polarity: + goes to +3V3.
- Keep C39 10 µF on the input.

**M5: IMU module power and pads.** The module in the photo has an MPU6050, a small microcontroller and its own SOT-23 regulator (U3). It matches WitMotion WT61/JY61-type modules, which are specified for 3.3–5 V with a 5 V lead in WitMotion's manual.
- Power it from **+5V**, not +3V3, so its on-board regulator has headroom. Its TX/SDA/SCL stay at 3.3 V logic.
- **Read the labels printed next to each half-hole pad** and make the symbol pin names match. The current symbol (VCC, RX, TX, GND, VCC, SCL, SDA, GND) is a guess until then.
- The current footprint `projekt:MPU_6050` uses **through-hole** pads. For a castellated module use **SMD pads** at the module's pitch (measure it; 15.24 mm modules usually use 2.54 mm), extending about 1 mm beyond the module edge so you can reach them with the iron.
- On WitMotion modules the I²C interface talks to the module's own MCU, not the raw MPU6050, and UART is the primary interface. Wire both:
  - **I²C:** SCL → PB6, SDA → PB7
  - **UART:** module TX → PD6 (USART2_RX); PD5 (USART2_TX) → module RX
  - Choose which one to use in firmware.

**M6: Mounting holes use a wrong footprint.** `Module:Maple_Mini` is a whole dev-board footprint. Change all four to `MountingHole:MountingHole_3.2mm_M3` (or `_Pad` if you want them grounded).

**M7: Missing footprints:**
- **D2, D3, D4:** `LED_SMD:LED_0805_2012Metric_Pad1.15x1.40mm_HandSolder`
- **FB1:** see C8
- **U7:** see C9

**M8: Encoder connection.** The Pololu 3081 board solders onto the motor. From it you run 6 wires to the main board. The current footprint `projekt:POLOLU_ENCODER_FOOTPRINT` has 6 pads at 2.0 mm pitch, tight for hand-soldering wires.
- Use `Connector_PinHeader_2.54mm:PinHeader_1x06_P2.54mm_Vertical` as solder pads, or fit a header later.
- Put the signal names on the silkscreen (M1, M2, GND, VCC, A, B).
- Pololu lists the encoder board's pad order as **M1, M2, GND, VCC, A, B**. Check the silkscreen on your boards and solder by name, not by position.

### Minor

- **Power symbols:** there are 5 × `power:+3.3V` and 7 × `power:+3V3`, both carrying the value +3V3, plus 7 leftover `power:VCC` symbols. Replace them all with `power:+3V3`, and use `power:VBUS`/`power:+BATT` or global labels for VIN/VBAT. It works today, but it is confusing.
- **R4:** the 330 Ω power LED resistor wastes ~4 mA. Use 1 k like the other LEDs.
- **TB6612 symbol:** in the Symbol Editor, set the doubled output pins (AO1', AO2', BO1', BO2') to type `passive`. That removes 4 false ERC errors.
- **U7 FFC symbol:** pins 1 and 20 are `power_out`; set all pins to `passive`.
- **Root file name:** the root is `uc.kicad_sch`, but the project is `linefollowerk`. The KiCad GUI handles this, but command-line tools look for `uc.kicad_pro` and miss the project library tables. Optionally, rename the root later to `linefollowerk.kicad_sch` (after deleting the old file of that name).

### Value notation changes

KiCad accepts both styles, but pick one and use it everywhere. The recommended style is the unit spelled out for capacitors and inductors (`100nF`, `10µF` or `10uF`), lowercase `k` for kilo-ohms, and the ferrite impedance with its test frequency.
- **R3, R5, R9, R11:** `10K` → `10k`
- **R10:** `100K` → `100k` (R6 already `100k`)
- **R12, R13:** `1K` → `1k`
- **R8:** `54.9K` → `54.9k`
- **C1, C4, C11–C22, C24, C25, C29, C30, C33, C37:** `100n` → `100nF`
- **C27:** `10n` → `10nF`
- **C26, C28:** `1u` → `1µF`
- **C31, C32:** `2u2` → `2.2µF`
- **C3, C10, C23, C35, C36, C39, C40:** `10u` → `10µF`
- **C38:** `22u` → `22µF`
- **C34:** `470u` → `470µF` (add `16V` to its Voltage field or a note)
- **FB1:** `120R` → `120Ω@100MHz` (or the part's real value, e.g. `220Ω@100MHz`)
- **D2, D3, D4:** `LED` → the colour, e.g. `LED green`
- **U7:** empty value → `FFC 20p`
- **SW1:** `SW_MEC_5E` → the real part, e.g. `PTS645`

---

## B. Answers on KTIR, bulk capacitors and placement

**Which KTIR resistor is the emitter's?**
The **200 Ω** is the IR LED (emitter) resistor: (3.3 V − ~1.2 V) / 200 Ω ≈ 10 mA per sensor, about 85 mA for eight. The **20 kΩ** is the phototransistor pull-up, and it sets the output impedance the ADC sees.

**What "10 µF + 100 nF near the connector" means.**
Add two capacitors to the schematic next to U7:
- one 10 µF (0805 or 1206) with one pin on +3V3 and the other on GND
- one 100 nF (0805) with one pin on +3V3 and the other on GND

On the PCB, place both right next to FFC pin 1 and pin 20. They supply the emitters' current locally, so the sensor supply doesn't dip.

**The "1 nF at each ADC pin" advice is withdrawn.** With a 20 kΩ source, 1 nF makes a 20 ms filter, far too slow. Do not add capacitors on the SENS lines. Instead set the ADC3 sampling time to 387.5 cycles (audit §3.5).

**Where does C34 (470 µF) go?**
Its position on the schematic does not matter; only the PCB placement does. On the PCB, place C34 between the fuse/P-FET and the TB6612, **as close to the TB6612 VM pins as possible**. The motors draw fast current pulses, and the bulk capacitor must supply them with a short loop. Place the 10 µF ceramics C10 (pin 24) and C35/C36 (pins 13–14) directly at those pins, closer than C34.

**Are C35, C36 and C10 correct?**
Yes, as local decoupling at the VM pins. Make them 25 V X7R 1206. They are not bulk capacitors: C34 is the bulk, and C2 can go.

**"At each regulator input":**
- **TPS563201:** 2 × 10 µF + 100 nF right at its VIN (pin 3) and GND (pin 1). Currently it has none nearby.
- **AMS1117:** 10 µF at VI (pin 3). C39 already does this.

---

## C. Parts: KiCad symbol, KiCad footprint, and what to buy on TME

**C1. Battery connector: JST-SYP (JST-RCY)**
- KiCad has **no board footprint** for JST-SYP, because it is a wire-to-wire connector (plug SYP-02T-1, receptacle SYR-02T, contacts rated 3 A).
- **Solution:** solder a short mating pigtail to two pads on the board.
  - **Symbol:** `Connector:Conn_01x02_Pin` (keep J1)
  - **Footprint:** `Connector_Wire:SolderWire-0.5sqmm_1x02_P4.8mm_D0.9mm_OD2.3mm` (fits 20–22 AWG battery wire). Use the `_Relief` variant for strain-relief holes.
  - **TME:** `JST-SYP-2P-LIPO` (lead with an SYR-02T socket, 150 mm).
- **Before buying, check your battery's plug:**
  - If the battery lead ends in the **SYP** plug, you need an **SYR** socket pigtail, like the TME part above.
  - If it ends in **SYR**, buy an SYP plug lead instead.
- Mark + and − on the silkscreen. Red = +.

**C2. Fuse: Littelfuse 0154005.DRT**
- It **comes with the holder**: it is the OMNI-BLOK 154 series, a holder with a 5 A slow-blow Nano2 fuse already inside. The fuse can be replaced without desoldering.
- **Symbol:** `Device:Fuse`
- **Footprint:** `Fuse:Fuseholder_Littelfuse_Nano2_154x`
- Size about 9.7 × 5.0 mm, hand-solderable. It is often backordered, so check TME stock.
- Wiring: J1 + → fuse → Q1 drain.

**C3. Reverse-polarity P-FET**
- **SQJ465EP-T1-GE3 (your link): not recommended.** Its R_DS(on) is 85 mΩ at −10 V and 115 mΩ at −4.5 V, so with 2S (V_GS ≈ −6 to −8 V) and 3 A motor peaks it would dissipate about 1 W. That is worse than the AO3401A.
- **Recommended: AO4407A** (Alpha & Omega)
  - −30 V, −12 A, V_GS ±25 V
  - **R_DS(on) < 17 mΩ at −6 V**, < 13 mΩ at −10 V
  - SOIC-8, easy to hand-solder
  - About 0.15 W at 3 A
- **Alternative: SQJ457EP-T1-GE3** (Vishay, same family as your pick)
  - −60 V, −36 A
  - 25 mΩ at −10 V, 35 mΩ at −4.5 V
  - PowerPAK SO-8L. Its large bottom pad is harder to hand-solder; use a hot-air station or put vias under the pad.
- **KiCad for AO4407A:**
  - **Symbol:** `Transistor_FET:IRF7404` (same SO-8 pinout: pins 1–3 source, pin 4 gate, pins 5–8 drain). Change its Value to `AO4407A`.
  - **Footprint:** `Package_SO:SOIC-8_3.9x4.9mm_P1.27mm`.
- **Wiring:** drain (5–8) → fuse (battery side), source (1–3) → VIN, gate → R3 to GND.
  - Optional: a 12 V zener (e.g. BZT52C12, SOD-123) from gate (anode) to source (cathode) against spikes when plugging in the battery.
  - Optional: raise R3 to 100 k.

**C4. TPS563201 inductor and capacitors**
See B3.

**C5. SWD connector.** Which one depends on the programmer you own:
- **ST-LINK/V2 USB-stick clone, or a Nucleo board's ST-LINK:** use a simple **1×6 2.54 mm** header.
  - **Symbol:** `Connector_Generic:Conn_01x06`
  - **Footprint:** `Connector_PinHeader_2.54mm:PinHeader_1x06_P2.54mm_Vertical`
  - **Pin order** (same as the Nucleo "CN4 SWD" connector):
    - 1 = +3V3 (VDD_TARGET)
    - 2 = SWCLK (PA14)
    - 3 = GND
    - 4 = SWDIO (PA13)
    - 5 = NRST
    - 6 = SWO (PB3)
  - Easiest to hand-solder, and the clone's jumper wires plug straight in.
- **ST-LINK V3 MINIE, J-Link, or any "10-pin Cortex debug" cable:**
  - **Symbol:** `Connector:Conn_ARM_JTAG_SWD_10`
  - **Footprint:** `Connector_PinHeader_1.27mm:PinHeader_2x05_P1.27mm_Vertical` (through-hole, easier to solder than the `_SMD` variant)
  - **Pins:**
    - 1 = VTref → +3V3
    - 2 = SWDIO → PA13
    - 3, 5, 9 = GND
    - 4 = SWCLK → PA14
    - 6 = SWO → PB3
    - 7 = key (no connection)
    - 8 = no connection
    - 10 = nRESET → NRST
  - **TME:** search "pin header 1.27 mm 2x5 THT", or a shrouded box header "IDC 1.27mm 10 pin".
  - Note that the ST-LINK V3 MINIE's own cable is the 14-pin STDC14. Its pins 3–12 are the same signals, so a 14-to-10-pin adapter or cable is needed.

**C6. HSE crystal: 25 MHz (keep 25 MHz; the CubeMX clock settings in the audit §3.2 match it)**
- **Symbol:** `Device:Crystal_GND24` (4-pin crystal: pins 1 and 3 are the crystal, pins 2 and 4 are GND)
- **Footprint:** `Crystal:Crystal_SMD_3225-4Pin_3.2x2.5mm_HandSoldering`. For easier soldering, choose a 5×3.2 mm crystal and `Crystal:Crystal_SMD_5032-2Pin_5.0x3.2mm_HandSoldering` with `Device:Crystal` instead.
- **Wiring:**
  1. Crystal pin 1 → PH0 (OSC_IN, MCU pin 23).
  2. Crystal pin 3 → PH1 (OSC_OUT, pin 24).
  3. Pins 2 and 4 → GND.
  4. Add one load capacitor from PH0 to GND and one from PH1 to GND.
- **Load capacitor value:** C = 2 × (CL − C_stray), with C_stray ≈ 5 pF.
  - For a crystal with CL = 10 pF → **10 pF** each.
  - For CL = 12 pF → 15 pF.
  - For CL = 18 pF → 27 pF.
  - Use C0G/NP0 0805 capacitors, `Device:C`.
- **TME filter:** quartz crystal, SMD, 25.000 MHz, 3.2×2.5 mm (or 5×3.2 mm), load capacitance 8–12 pF, tolerance ±20 ppm or better, ESR ≤ 80 Ω.
- **Layout:** crystal and capacitors within ~5 mm of pins 23/24, a ground pour around them, and no other signals underneath.

**C7. START tact switch** (the NRST button SW1 uses the same part)
- **Symbol:** `Switch:SW_Push` (2-pin, simplest)
- **Footprint:** `Button_Switch_SMD:SW_SPST_PTS645Sx43SMTR92` (6×6 mm SMD, easy to solder)
- **TME:** C&K `PTS645SM43SMTR92 LFS`, or any 6×6 mm SMD tact switch with the same 4-pad layout.
- **Wiring:** one side → PC13, other side → GND. PC13 uses the internal pull-up (CubeMX: GPIO_EXTI13, pull-up, falling edge). An optional 100 nF from PC13 to GND reduces bounce.

**C8. Ferrite bead FB1**
- **Symbol:** `Device:FerriteBead_Small` (or keep `Device:FerriteBead`)
- **Footprint:** `Inductor_SMD:L_0805_2012Metric_Pad1.05x1.20mm_HandSolder`
- **TME:** Murata `BLM21PG221SN1D` (0805, 220 Ω at 100 MHz, 2 A), or any 0805 chip ferrite of 120–600 Ω at 100 MHz rated ≥ 200 mA.
- Set Value to `220Ω@100MHz`.

**C9. FFC connector U7**
- KiCad has 20-pin FFC footprints in `Connector_FFC-FPC`, for example:
  - **1.0 mm pitch:** `Molex_200528-0200_1x20-1MP_P1.00mm_Horizontal`, `TE_2-84952-0_1x20-1MP_P1.0mm_Horizontal`, `JUSHUO_AFA07-S20FCA-00_1x20-1MP_P1.0mm_Horizontal`
  - **0.5 mm pitch:** `Hirose_FH12-20S-0.5SH_1x20-1MP_P0.50mm_Horizontal`, `TE_2-1734839-0_1x20-1MP_P0.5mm_Horizontal`
- **Check two things on your cable and sensor board first:**
  1. **Pitch:** 20 contacts spanning about 20 mm is 1.0 mm pitch; about 10 mm is 0.5 mm.
  2. **Contacts on the same side at both ends, or opposite sides?** This decides whether pin 1 stays pin 1 on the main board, or becomes pin 20.
- Buy the exact connector whose footprint you pick. 1.0 mm is much easier to hand-solder.

**C10. The 1×6 encoder pads:** see M8.

---

## D. CubeIDE "Invalid Input: Must be IFileEditorInput"

**Update (review 3):** the real cause on this machine is a stale workspace entry. The workspace still points `linefollower` at the old `~/Desktop/linefollowerk/code`, so Import says the project "already exists". Follow **[`REVIEW_3_STATUS.md`](REVIEW_3_STATUS.md) §1** (delete the dead entry without deleting contents, then import). The steps below are the general procedure.

CubeIDE 1.18.1 shows this when the `.ioc` is opened from outside a workspace project (File → Open File, a file manager, or a recent-files entry). Fix:
1. In CubeIDE: **File → Import… → General → Existing Projects into Workspace → Next**.
2. **Select root directory:** browse to `…/linefollowerk/code`. The project `linefollower` appears ticked. Leave "Copy projects into workspace" **unticked** → **Finish**.
3. In the **Project Explorer** panel, expand `linefollower` and double-click `linefollower.ioc`. If it opens as text, right-click → **Open With → STM32CubeMX**.
4. If it still fails:
   - close CubeIDE
   - make sure the workspace folder (chosen at start-up) is **not** the `code` folder itself or inside it
   - back up `linefollower.ioc`, then delete the last line `isbadioc=true`; CubeMX writes that flag after a failed load
   - reopen

---

## E. The CubeMX settings

The settings (formerly tables) are rewritten as lists in `HARDWARE_AUDIT.md` §3, including:
- §3.1 current state with verdicts
- §3.2 clock
- §3.4 timers
- §3.5 ADC3 with the corrected sampling time

---

## F. Files: what to keep, what to delete, how to merge libraries

Close KiCad and CubeIDE before deleting anything. Move files to a temporary folder first, open the project, check that nothing is missing, and only then delete. Version control is yours to handle.

**Keep: project and design**
- `linefollowerk.kicad_pro`: project settings
- `uc.kicad_sch`: root sheet
- `peripherials.kicad_sch`: child sheet
- `linefollowerk.kicad_pcb`: PCB (stub for now)
- `linefollowerk.kicad_prl`: window and layer view state (harmless; KiCad recreates it)
- `sym-lib-table`, `fp-lib-table`: keep, but fix them (below)
- `docs/`, `PROJECT_CONTEXT.md`
- `stm32h743zi.pdf`: suggest moving it to `docs/datasheets/`

**Keep: libraries (until merged)**
- `linefolower_stm.kicad_sym`: holds STM32H743ZIT6, TB6612FNG, MPU6050, BLUETOOTH-SERIAL-HC-06
- `projekt.pretty/`: MPU_6050, POLOLU_ENCODER_FOOTPRINT, MODULE_ZC142200, SSOP24, SOT95P280X110-6N
- `moje_elementy.pretty/`: SOT95P280X110-6N (used by U6)
- `3D models/`: footprints reference `${KIPRJMOD}/3D models/TB6612FNG.step`, `TPS563201DDCR.step`, `ZC142200.step`, `LMR16006XDDCR.step`

**Keep: firmware**
- `code/`: everything, including `Drivers/`. CubeMX can regenerate Drivers, but the project needs them to build.

**Safe to delete**
- `linefollowerk.kicad_sch`: the old first sheet, no longer part of the project. Everything moved to `peripherials.kicad_sch`; check that first.
- `uc.kicad_sch.PRZED_ODZYSKANIEM_2026-08-25_191545.bak` and `uc_ODZYSKANE_2026-08-20_0943.kicad_sch.txt`: recovery leftovers
- `Elements.bak`, `Elementy_linefollower.bak`, `linefolower_stm.bak`: library backups
- `_restore_backup_2026-08-18T19-36-10-512/` and `_restore_backup_2026-08-20T09-03-28-391/`: old restore snapshots
- `linefollowerk-backups/`: old KiCad zip backups from June–July. KiCad makes new ones automatically; set the count in Preferences → Common → Project Backup.
- `modele_3D/`: empty folder
- `New_Library.kicad_sym`: empty library
- `Elementy_linefollower.kicad_sym`: only an old `STM32H7` symbol, not used
- Root-level duplicates of files that also exist in `3D models/`: `ZC142200.kicad_sym`, `ZC142200.step`, `MODULE_ZC142200.kicad_mod`
- `Elements.kicad_sym`: only after you copy its `LLC-Connector` symbol into the merged library (below)

**Do not touch by hand**
- `.history/`: KiCad 10's Local History (browse it with File → Local History). It can be pruned or disabled in Preferences → Common → Project Backup.
- `~*.lck` files: lock files while KiCad is open; they disappear when KiCad closes.

**How to merge everything into one project library**
1. **Create the symbol library:**
   - KiCad project window → **Symbol Editor**
   - **File → New Library…** → choose **Project** → save as `linefollower.kicad_sym` in the project folder
   - This also adds it to the project `sym-lib-table` with `${KIPRJMOD}`.
2. **Copy symbols that already live in a library file** (`linefolower_stm`, `Elements`):
   - Preferences → Manage Symbol Libraries → Project Specific Libraries tab
   - Temporarily add `linefolower_stm.kicad_sym` and `Elements.kicad_sym` with the folder button; the path should become `${KIPRJMOD}/…`
   - In the Symbol Editor, right-click a symbol (e.g. `STM32H743ZIT6`) → **Copy**, then right-click the `linefollower` library → **Paste**
   - Repeat for TB6612FNG, MPU6050 and LLC-Connector
3. **Copy symbols that exist only inside the schematic** (`Elements2:TPS563201DDCR`, `linefolower_stm:POLOLU_ENCODER`, `linefolower_stm:ZC142200`, which are missing from any library on disk):
   - Open the sheet, right-click the symbol → **Edit Library Symbol…**
   - In the Symbol Editor: **File → Save As…** → library `linefollower`, same name
4. **Point the schematic at the new library:**
   - Schematic Editor → **Tools → Change Symbols…**
   - Select "all symbols matching library identifier", e.g. `linefolower_stm:STM32H743ZIT6`
   - New library identifier: `linefollower:STM32H743ZIT6` → **Change**
   - Repeat per symbol, then run **Tools → Update Symbols from Library…**
5. **Footprints, the same way:**
   - Footprint Editor → **File → New Library…** → Project → `linefollower.pretty`
   - Right-click each custom footprint in `projekt` / `moje_elementy` → Copy → paste into `linefollower`
   - In the schematic, **Tools → Assign Footprints…** (or the Symbol Fields Table) → change `projekt:…` / `moje_elementy:…` to `linefollower:…`
   - Fix IC1 to `linefollower:SSOP24`
6. **Clean the tables:** Preferences → Manage Symbol Libraries / Manage Footprint Libraries → Project Specific tab.
   - Remove `Elementy_linefollower` (the `C:/Users/rataj/...` path), `linefolower_stm`, `Elements`, `projekt`, `moje_elementy` and the missing `linefollower_stm` footprint entry
   - Keep only `linefollower`
7. Run ERC: there should be no `lib_symbol_issues` or `footprint_link_issues`. Only then delete the old library files.

---

## Sources

- TI TPS563201 datasheet (Table 7-2): https://www.ti.com/lit/ds/symlink/tps563201.pdf
- STM32H743ZI datasheet DS12110: local `stm32h743zi.pdf`
- AO4407A datasheet: https://www.aosmd.com/res/datasheets/AO4407A.pdf
- SQJ465EP datasheet: https://www.vishay.com/docs/67996/sqj465ep.pdf
- SQJ457EP datasheet: https://www.vishay.com/docs/76628/sqj457ep.pdf
- Littelfuse 0154005.DRT: https://www.littelfuse.com/products/fuse-blocks-fuseholders-and-fuse-accessories/fuse-blocks/154/0154005_drt.aspx
- JST-SYP LiPo lead at TME: https://www.tme.eu/en/details/jst-syp-2p-lipo/batteries-containers-and-holders/jst/
- JST RCY / SYP overview: https://keszoox.com/blogs/news/jst-rcy-connector-complete-guide
- Pololu 3081 encoder kit: https://www.pololu.com/product/3081
- WitMotion WT61: https://www.wit-motion.com/6-axis/witmotion-wt61-6-axis-ahrs-sensor-digital.html
