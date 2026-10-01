# Part 77: Advanced Safety System Design (Steps 1521-1540)

## การออกแบบระบบความปลอดภัยขั้นสูงตามมาตรฐาน IEC 61511

---

### 🔗 Navigation
- [← Part 76: DCS กับ PLC Integration](part76_dcs_plc_integration.md)
- [→ Part 78: Simulation & Testing](part78_simulation_testing.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: Functional Safety ในกระบวนการอุตสาหกรรม

Functional Safety เป็นส่วนหนึ่งของ Overall Safety ที่เกี่ยวกับการทำงานที่ถูกต้องของระบบอัตโนมัติเพื่อป้องกันอันตราย

### มาตรฐานที่เกี่ยวข้อง

```
IEC 61511: Functional Safety - Safety Instrumented Systems
  สำหรับ: Process Industry (Chemical, Oil & Gas, Power)
  
IEC 62061: Functional Safety of Machinery
  สำหรับ: Machinery (Manufacturing, Press, Crane)
  
ISO 13849: Safety of Machinery - Safety-Related Parts
  สำหรับ: Machinery (Performance Level PL a-e)
  
IEC 61508: Functional Safety - E/E/PE Systems (Base Standard)
  ครอบคลุมทุก Industry

SIL (Safety Integrity Level):
  SIL 1: PFD (Probability of Failure on Demand) = 0.01-0.1
  SIL 2: PFD = 0.001-0.01
  SIL 3: PFD = 0.0001-0.001
  SIL 4: PFD = 0.00001-0.0001 (หายากมาก)
```

---

## Step 1521: Process Hazard Analysis (PHA/HAZOP)

### HAZOP (Hazard and Operability Study)

```
HAZOP คืออะไร:
  วิธีการระบุอันตรายและปัญหาการทำงานในกระบวนการ
  
  Guide Words + Parameter = Deviation
  
  Guide Words: NO, LESS, MORE, PART OF, AS WELL AS, 
               REVERSE, OTHER THAN, EARLY, LATE, 
               BEFORE, AFTER
               
  Parameters: FLOW, PRESSURE, TEMPERATURE, 
               LEVEL, COMPOSITION, PHASE
  
  ตัวอย่าง Node: Reactor Feed Line
  Parameter: FLOW
  Guide Word: HIGH (MORE) → Deviation: High Flow
  → Cause: Control Valve Fails Open
  → Consequence: Reactor Overheat, Possible Runaway
  → Safeguards: High Flow Alarm, SIS Trip
  → SIL Required: 2
```

**HAZOP Documentation Format:**

```
HAZOP Study: Reactor Feed System
Study Leader: Chemical Engineer
Date: 2026-10-01
P&ID No.: P-101-RXR

Node: Feed Line to Reactor R-101
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Guide Word | Deviation      | Causes       | Consequences
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MORE       | High Flow      | FV-101 Fail  | Reactor Overheat
           |                | Open         | → Runaway Reaction
           |                | FT-101 Error |
───────────────────────────────────────────────────────────────
LESS       | Low/No Flow    | FV-101 Fail  | Loss of Cooling
           |                | Closed       | → Temp Rise
           |                | Pump Fail    |
───────────────────────────────────────────────────────────────
HIGH       | High Temp      | Reaction     | Equipment Damage
(Temp)     | Feed           | Heat         | Downstream Effect
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Step 1522: Layer of Protection Analysis (LOPA)

### การวิเคราะห์ชั้นป้องกัน

```
LOPA ใช้คำนวณ SIL ที่ต้องการ:

Scenario: Reactor Overpressure → Rupture → Toxic Release

Initiating Cause:
  Failure of Cooling Water (IC): Frequency = 0.1/year

Independent Protection Layers (IPL):
  IPL1: Process Control (BPCS High Pressure Alarm): PFD = 0.1
  IPL2: Relief Valve: PFD = 0.01
  IPL3: Fire & Gas Detection + Emergency Response: PFD = 0.1

Mitigated Consequence Frequency:
  = 0.1 × 0.1 × 0.01 × 0.1 = 1×10⁻⁵ per year

Target Frequency (Regulatory or Company):
  = 1×10⁻⁵ per year (Significant Injury)
  
ถ้าไม่มี SIS: 1×10⁻⁵ ≥ 1×10⁻⁵ → OK ไม่ต้อง SIS
ถ้าต้องการ 10× safer: 1×10⁻⁶ → SIS SIL 1 ต้องการ (PFD ≤ 0.1)
```

---

## Step 1523: Safety Requirements Specification (SRS)

### Safety Function Definition

**ตัวอย่าง SRS สำหรับ High Pressure Trip:**

```
Safety Function: SF-001
  Description: High Pressure Reactor Trip
  
  Process Safety Time (PST): 
    Time from initiation to dangerous condition: 30 minutes
    
  Response Time of Safety Function:
    Must trip within: 5 minutes (PST/6)
    
  Safety Integrity Level (SIL): 2
    Required PFD: 0.001 - 0.01
    
  Inputs:
    PT-101A: Pressure Transmitter A (4-20mA, 0-10 bar)
    PT-101B: Pressure Transmitter B (4-20mA, 0-10 bar)  
    PT-101C: Pressure Transmitter C (4-20mA, 0-10 bar)
    
  Voting Logic: 2oo3 (2 out of 3 must agree)
  
  Trip Point: 8.0 bar (Safety Critical Action)
  Alarm Point: 7.0 bar (Non-safety)
  
  Outputs:
    XV-101A: Feed Isolation Valve A (Fail-Close) → Close
    XV-101B: Feed Isolation Valve B (Fail-Close) → Close
    PV-101: Pressure Control Valve → Drive to Full Open (release pressure)
    
  Final State: Reactor isolated from feed, pressure relieved
  
  Manual Reset Required: Yes (SIL 2 requires intentional restart)
    Reset Location: Local Control Room + Field Reset Confirmation
    
  Proof Test Interval: 1 year (every Annual Shutdown)
  
  Spurious Trip Rate Target: < 0.5 per year
```

---

## Step 1524: Safety Function Design

### Sensor Voting Logic

```
Sensor Voting Configurations:
  1oo1 (1 out of 1):    Single Sensor
    Reliability: Low, Spurious Trip Risk: High
    
  1oo2 (1 out of 2):    Either sensor trips
    Reliability: High, Spurious Trip Risk: Very High (2× chance)
    PFD = PFD1 × PFD2 (series from safety perspective)
    
  2oo2 (2 out of 2):    Both sensors must trip
    Reliability: Lower, Spurious Trip Risk: Low
    PFD = PFD1 + PFD2 - PFD1×PFD2 (parallel from safety perspective)
    
  2oo3 (2 out of 3):    Best of both worlds
    Good Safety Reliability + Lower Spurious Trip Risk
    Recommended for SIL 2-3
    PFD = 3 × PFD² - 2 × PFD³ (for equal failure rates)
    
  1oo2D (1oo2 with Diagnostics): ใช้ระบบ Diagnostic ช่วย
    Better than simple 1oo2
```

**2oo3 Voting Logic in Safety PLC:**

```
=== LADDER: 2oo3 Pressure Voting (Triconex / Hima) ===

Rung 1: Individual Sensor High Trip
|--[PT101A > 8.0]--( PT_A_TRIP )--|
|--[PT101B > 8.0]--( PT_B_TRIP )--|
|--[PT101C > 8.0]--( PT_C_TRIP )--|

Rung 2: 2oo3 Vote
|--[PT_A_TRIP]--[PT_B_TRIP]--( PRESSURE_TRIP )--|
|--[PT_A_TRIP]--[PT_C_TRIP]--( PRESSURE_TRIP )--|
|--[PT_B_TRIP]--[PT_C_TRIP]--( PRESSURE_TRIP )--|
| ถ้า 2 ตัวใดก็ได้ใน 3 ตัว Trip → PRESSURE_TRIP = TRUE

Rung 3: Sensor Discrepancy Alarm (Diagnostics)
|--[PT_A_TRIP]--[/PT_B_TRIP]--[/PT_C_TRIP]--|
|   +--( SENSOR_DISCREPANCY_ALARM )--|
|   // Sensor A คนเดียว Trip = สงสัยว่า Sensor A เสีย
|   // Alarm แต่ยังไม่ Trip (ยังมี 2 ตัวเห็นปกติ)
```

---

## Step 1525: Common Cause Failure (CCF)

### การป้องกัน Common Cause Failure

```
Common Cause Failure คืออะไร:
  เมื่อหลาย Component เสียในเวลาเดียวกัน เพราะสาเหตุเดียว
  
  ตัวอย่าง:
  - Transmitter 3 ตัวเสียพร้อมกัน เพราะ Power Supply เสีย
  - Safety PLC Redundant เสียพร้อม เพราะ Firmware Bug
  - Pressure Switches เสียพร้อม เพราะ Physical Shock (Explosion)

IEC 61508/61511 CCF Mitigation:
  β Factor: ส่วนของ Failure เนื่องจาก CCF
  β = 5-10% ทั่วไป
  ลด β ด้วย:
  1. Diversity: ใช้ Transmitter จากต่าง Vendor
  2. Separation: ติดตั้งห่างกัน ไม่แชร์ Conduit
  3. Staggered Testing: Test สลับกัน ไม่ทีเดียว
  4. Good Process: FMEA, Root Cause Analysis
```

---

## Step 1526: Proof Test Design

### Proof Test คืออะไรและทำไมสำคัญ

```
Proof Test:
  การทดสอบ Safety Function เพื่อยืนยันว่า
  ระบบ Safety ยังทำงานได้ตามที่ออกแบบ
  
  ทำไมต้องทดสอบ:
  Safety PLC อาจมี "Dangerous Undetected Failure"
  = เสียแบบที่ Diagnostics ตรวจไม่พบ
  = ดูเหมือน OK แต่จริงๆ ไม่ทำงานเมื่อจำเป็น
  
  Proof Test Interval (PTI):
  ยิ่งทดสอบบ่อย → PFD ดีขึ้น
  ยิ่งทดสอบน้อย → PFD แย่ลง
  
  PFD Average = λDU × PTI / 2
  
  ตัวอย่าง:
  Transmitter λDU = 2×10⁻⁶ /hr
  PTI = 8760 hr (1 year)
  PFD = 2×10⁻⁶ × 8760 / 2 = 0.00876 → SIL 1 (0.001-0.01)
  
  ถ้า PTI = 4380 hr (6 months):
  PFD = 2×10⁻⁶ × 4380 / 2 = 0.00438 → ยังคง SIL 1 แต่ดีกว่า
```

**Proof Test Procedure:**

```
Safety Function SF-001 Proof Test Procedure
(Must be done during planned shutdown):

Step 1: Obtain Work Permit
  □ Hot Work Permit ถ้าจำเป็น
  □ Override Permit (bypass safety function during test)
  □ Notify Control Room
  
Step 2: Bypass Safety Function
  □ Enable Maintenance Bypass ใน Safety PLC
  □ Confirm Bypass Active on HMI
  □ Install Backup Safety Measure (if required)
  
Step 3: Test Sensors
  For each Pressure Transmitter:
  □ Inject Known Pressure (using Test Pressure Point)
  □ Verify Reading on Safety PLC Console
  □ Inject Trip Pressure (8.0 bar)
  □ Verify PT_A_TRIP bit goes TRUE in Safety PLC
  □ Record Actual Trip Point in Test Sheet
  □ Repeat for PT-101B, PT-101C
  
Step 4: Test 2oo3 Logic
  □ Simulate 2 Transmitters in Trip
  □ Verify PRESSURE_TRIP = TRUE
  □ Verify Final Elements Energized
  
Step 5: Test Final Elements (Valves)
  □ XV-101A: Command Close, Verify Closed
  □ XV-101B: Command Close, Verify Closed
  □ PV-101: Command Open, Verify Open
  □ Measure Stroke Time
  
Step 6: Restore
  □ Remove Bypass
  □ Confirm Safety Function Active
  □ Sign and Submit Test Sheet
```

---

## Step 1527: Bypass Management

### การจัดการ Bypass อย่างปลอดภัย

```
Bypass Types:
  1. Maintenance Bypass: ระหว่างซ่อมบำรุง
  2. Test Bypass: ระหว่าง Proof Test
  3. Commissioning Bypass: ตอน Start-up ครั้งแรก
  4. Inhibit: ปิด Alarm ชั่วคราว (ไม่ใช่ Safety Function)

Bypass Management Requirements (IEC 61511):
  - ต้องมี Procedure และ Work Permit
  - ต้องมีการ Approve โดย Responsible Person
  - ต้องมี Time Limit (ห้าม Bypass ค้าง)
  - ต้องมีการแสดงผลบน HMI ชัดเจน
  - ต้องมีการบันทึก Log
```

**Bypass Management in Safety PLC:**

```
=== LADDER: Bypass Management System ===

Rung 1: Bypass Authorization
|--[BYPASS_REQUEST]--[BYPASS_KEY_ENABLED]--|
|   [SUPERVISOR_APPROVED]--|
|   +--( BYPASS_ARMED )--|

Rung 2: Bypass Timer (Maximum Time)
|--[BYPASS_ARMED]--[T_BYPASS, EN]--|
| T_BYPASS: TON, PT=8hr (ไม่เกิน 8 ชั่วโมง)
|
|--[T_BYPASS, DN]--( BYPASS_EXPIRED )--|
|--[BYPASS_EXPIRED]--( AUTO_REMOVE_BYPASS )--|

Rung 3: Bypass Active Indication
|--[BYPASS_ACTIVE]--( BYPASS_LAMP )--|
|--[BYPASS_ACTIVE]--( BYPASS_HORN )--|   // แตรเตือนทุก 15 นาที
|--[BYPASS_ACTIVE]--( SCADA_BYPASS_TAG )--|  // แสดงบน HMI

Rung 4: Safety Logic with Bypass
|--[PRESSURE_TRIP]--[/BYPASS_ACTIVE]--( SIS_TRIP )--|
| ถ้า Bypass Active → ไม่ Trip (แต่ยังมี Alarm)
|--[PRESSURE_TRIP]--[BYPASS_ACTIVE]--( SIS_TRIP_BYPASSED_ALARM )--|
```

---

## Step 1528: SIL 3 System Design Example

### ตัวอย่างระบบ SIL 3

```
SIL 3 Requirements:
  PFD = 0.0001 - 0.001
  Architecture: ต้องการ High Redundancy
  Typical: 1oo2 Safety Logic + 2oo3 Sensors + Redundant Final Elements
  
Example: Emergency Shutdown System (ESD)
  Application: Natural Gas Pipeline
  
Architecture:
  Input: 3 × Pressure Transmitters (2oo3 voting)
  Logic: Dual Safety PLC (1oo2) with Cross-Monitoring
  Output: 2 × ESD Valves in Series (ใช้ 2 ต้องปิดทั้งคู่)
  
Triconex (Schneider) Configuration:
  3 × MP (Main Processor) in Triplicate
  TMR (Triple Modular Redundancy) Architecture
  2oo3 Voting ทั้ง Hardware และ Software
  Continuous Diagnostics (Fault Coverage > 99%)
```

---

## Step 1529: Management of Change (MOC)

### การจัดการการเปลี่ยนแปลง

```
MOC (Management of Change) สำคัญมากสำหรับ Safety:
  Texas City Refinery: การเปลี่ยนแปลง Procedure โดยไม่มี MOC
  
MOC สำหรับ Safety System Changes:
  1. Define the Change
  2. Safety Review (HAZOP/SIL Assessment)
  3. Engineering Review
  4. Approval (Plant Manager + Safety Manager)
  5. Pre-Startup Safety Review (PSSR)
  6. Training
  7. Implementation
  8. Documentation Update
  9. Post-Implementation Review
```

---

## Step 1530-1540: Complete SIL System

### Step 1530: Hardware Failure Mode Analysis

**FMEA สำหรับ Safety Function:**

```
Failure Mode and Effects Analysis:
Component: Pressure Transmitter
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Failure     | Effect              | Detection   | λDU (per hr)
Mode        |                     |             |
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
4mA (Low)   | Reads Below Range   | BPCS Alarm  | 8×10⁻⁷
            | → Misses High Press  |             |
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
20mA (High) | Reads Above Range   | SIS Alarm   | 4×10⁻⁷
            | → Spurious Trip      |             |
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Drift       | Reads Wrong Value   | Calibration | 1.2×10⁻⁶
            | → May miss or trip  | Check       |
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Lab Project 77: SIL 2 ESD System for Chemical Reactor

### โจทย์: ออกแบบ SIS SIL 2 สำหรับ Chemical Reactor

**กำหนด Hazard:**
- Overpressure: ถ้า Reactor > 8 bar → Rupture → Toxic Release
- SIL Required: 2 (จาก LOPA)

**Safety Function:**
- ตรวจสอบด้วย 3 Pressure Transmitters (2oo3)
- Trip: Close Feed Valve + Close Product Valve + Open Relief
- Manual Reset Required

**Hardware Specification:**
- Safety PLC: Pilz PSS4000 หรือ Hima HIMatrix F35
- Transmitters: Emerson Rosemount 3051 (SIL 2 Rated)
- Valves: Fisher Type 667 (Fail-Close, Spring Return)

**งาน:**
1. คำนวณ PFD สำหรับ Configuration 2oo3 Sensor + 1oo1 Logic + 1oo1 Valve
2. ตรวจสอบว่าผ่าน SIL 2 (PFD < 0.01) หรือไม่
3. เขียน SRS Document (ย่อ)
4. เขียน Safety PLC Program (Ladder/SFC)
5. เขียน Proof Test Procedure

**ข้อมูลสำหรับคำนวณ PFD:**
- Transmitter λDU = 2×10⁻⁶ /hr
- Safety PLC λDU = 5×10⁻⁷ /hr (single CPU with diagnostics)
- Valve λDU = 8×10⁻⁶ /hr
- Proof Test Interval = 8760 hr (1 year)
- β Factor = 5%

---

## สรุป Part 77

ใน Part นี้เราได้เรียนรู้:
- HAZOP สำหรับระบุอันตราย
- LOPA สำหรับกำหนด SIL ที่ต้องการ
- Safety Requirements Specification (SRS)
- Sensor Voting Logic (1oo2, 2oo3)
- Common Cause Failure Mitigation
- Proof Test Design
- Bypass Management
- SIL 3 System Architecture
- Management of Change (MOC)

**ทักษะสำคัญ:**
- PFD Calculation
- SRS Writing
- Safety PLC Programming
- IEC 61511 Compliance

---

### 🔗 Navigation
- [← Part 76: DCS กับ PLC Integration](part76_dcs_plc_integration.md)
- [→ Part 78: Simulation & Testing](part78_simulation_testing.md)
- [📚 สารบัญหลัก](../README.md)
