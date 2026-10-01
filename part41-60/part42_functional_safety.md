# Part 42: Functional Safety IEC 61508/62061/ISO 13849 (Steps 821-840)

## ภาพรวม
Functional Safety คือส่วนของ overall safety ที่ขึ้นอยู่กับการทำงานที่ถูกต้องของระบบควบคุม มาตรฐานหลักที่ใช้ในอุตสาหกรรมได้แก่ IEC 61508, IEC 62061 และ ISO 13849

---

## Step 821: Standards Hierarchy

### ลำดับชั้นของมาตรฐาน Functional Safety

```
Standards Hierarchy:
┌─────────────────────────────────────────────────────────────┐
│                     IEC 61508                               │
│           (Base/Generic Standard - ทุกอุตสาหกรรม)           │
│                         │                                   │
│         ┌───────────────┼───────────────────┐              │
│         ▼               ▼                   ▼              │
│    IEC 62061        ISO 13849           IEC 61511           │
│   (Machinery)      (Machinery)        (Process)            │
│   Safety of        Safety-related     Functional           │
│   machinery        control systems    safety in            │
│   control          for machinery      process              │
│   (SRECS)          (PLr)              industry             │
│         │               │                   │              │
│         ▼               ▼                   ▼              │
│    PFHD, SIL        PL, Cat.          PFD, SIL             │
│    (per hour)    (Performance         (per demand)         │
│                    Level)                                   │
└─────────────────────────────────────────────────────────────┘
```

### การเลือกใช้มาตรฐาน

| อุตสาหกรรม | มาตรฐานที่แนะนำ | เหตุผล |
|-----------|---------------|--------|
| เครื่องจักรทั่วไป | ISO 13849 | ง่ายกว่า ใช้ PL concept |
| เครื่องจักรซับซ้อน | IEC 62061 | ใช้ SIL concept ครอบคลุมมากกว่า |
| Process plant | IEC 61511 | เฉพาะสำหรับ process industry |
| Nuclear | IEC 61513 | มาตรฐานเฉพาะทาง |
| Railway | IEC 62280 | มาตรฐานเฉพาะทาง |

---

## Step 822: IEC 61508 - Base Standard

### โครงสร้าง IEC 61508

```
IEC 61508 Structure (7 Parts):
┌─────────────────────────────────────────────────────────────┐
│  Part 1: General Requirements                               │
│  ├── Scope and definitions                                  │
│  └── Overall safety lifecycle                              │
│                                                             │
│  Part 2: Requirements for E/E/PE Safety-related Systems    │
│  ├── Hardware requirements                                  │
│  └── Architecture constraints                              │
│                                                             │
│  Part 3: Software Requirements                             │
│  ├── Software safety lifecycle                             │
│  └── Software design techniques                           │
│                                                             │
│  Part 4: Definitions and Abbreviations                     │
│                                                             │
│  Part 5: Examples of Methods for SIL Determination        │
│  ├── LOPA (Layer of Protection Analysis)                   │
│  └── Risk graph method                                     │
│                                                             │
│  Part 6: Guidelines on IEC 61508-2 and -3                 │
│                                                             │
│  Part 7: Overview of Techniques and Measures              │
│  └── FFT, FMEA, Markov, etc.                             │
└─────────────────────────────────────────────────────────────┘
```

### Safety Lifecycle (IEC 61508)

```
IEC 61508 Safety Lifecycle:

Phase 1: Concept
    └── Hazard and Risk Analysis

Phase 2: Overall Scope Definition
    └── Define what is in/out of scope

Phase 3: Hazard and Risk Analysis
    └── Identify hazards, determine risk

Phase 4: Overall Safety Requirements
    └── Define safety functions and SIL

Phase 5: Safety Requirements Allocation
    └── Allocate to SIS, BPCS, non-SIS

Phase 6: Overall Planning
    └── V&V plan, operation plan, etc.

Phase 7: Realization (E/E/PE Systems)
    └── Design, implement, test

Phase 8: Overall Installation & Commissioning
    └── Install and commission SIS

Phase 9: Overall Safety Validation
    └── Verify against requirements

Phase 10: Overall Operation, Maintenance & Repair
    └── Operate within safety plan

Phase 11: Decommissioning/Disposal
    └── Safe disposal
```

---

## Step 823: IEC 62061 - Machinery Safety

### SRECS (Safety-Related Electrical Control Systems)

```
IEC 62061 SRECS Architecture:

┌─────────────────────────────────────────────────────────────┐
│           SRECS (Safety-Related Control System)             │
│                                                             │
│  ┌──────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │   Initiating │  │   Logic        │  │   Final        │ │
│  │   Element    │  │   Solver       │  │   Element      │ │
│  │              │  │                │  │                │ │
│  │  E-Stop btn  │  │  Safety PLC    │  │  Contactor     │ │
│  │  Light crtn  │  │  Safety relay  │  │  Valve         │ │
│  │  Door switch │  │  SIS logic     │  │  Drive         │ │
│  └──────────────┘  └────────────────┘  └────────────────┘ │
│         │                  │                  │            │
│         └──────────────────┼──────────────────┘            │
│                            │                               │
│                    SRECS Function                          │
└─────────────────────────────────────────────────────────────┘
```

### PFHD Calculation Method (IEC 62061)

```
PFHD Calculation for Series Architecture:

PFHD_system = PFHD_IE + PFHD_LS + PFHD_FE

ตัวอย่าง:
- Initiating Element (IE): E-Stop button dual-channel
  PFHD_IE = 5×10⁻⁹/h (from manufacturer's safety manual)

- Logic Solver (LS): Safety PLC (SIL 2 certified)
  PFHD_LS = 1×10⁻⁹/h

- Final Element (FE): Safety contactor dual-channel
  PFHD_FE = 8×10⁻⁹/h

PFHD_total = 5×10⁻⁹ + 1×10⁻⁹ + 8×10⁻⁹
           = 14×10⁻⁹/h
           = 1.4×10⁻⁸/h

→ SIL 2 (requirement: PFHD < 10⁻⁷/h) ✓
```

---

## Step 824: ISO 13849 - Machinery Control

### Performance Level (PL)

```
ISO 13849 Performance Level:

┌─────────────────────────────────────────────────────────────┐
│  PL a: ต่ำสุด   PFHD = 10⁻⁵ to 10⁻⁴ per hour              │
│  PL b:          PFHD = 3×10⁻⁶ to 10⁻⁵ per hour            │
│  PL c:          PFHD = 10⁻⁶ to 3×10⁻⁶ per hour            │
│  PL d:          PFHD = 10⁻⁷ to 10⁻⁶ per hour              │
│  PL e: สูงสุด   PFHD = 10⁻⁸ to 10⁻⁷ per hour              │
└─────────────────────────────────────────────────────────────┘
```

### Categories (ISO 13849)

```
Safety Categories:

Category B: 
- ใช้ components ตามมาตรฐาน
- ไม่มี diagnostic
- Single channel

Category 1:
- ใช้ proven components
- ไม่มี diagnostic
- Well-tried components and principles

Category 2:
- ใช้ proven components
- มี diagnostic (periodic test)
- Fault detected at next test cycle

Category 3:
- Dual channel architecture
- มี diagnostic
- Single fault ไม่ทำให้ safety function สูญเสีย
- ไม่รับรองการสะสม faults

Category 4:
- Dual channel architecture
- High diagnostic coverage
- Single fault ถูกตรวจจับก่อน/เมื่อ safety function ถูกเรียก
- รับรองการสะสม faults (accumulation of faults)
```

### Risk Graph สำหรับกำหนด PLr

```
ISO 13849 Risk Graph:

                    C1 (ผลกระทบเล็กน้อย)
                   /
               F1 (ไม่บ่อย)─── P1 (หลีกเลี่ยงได้) → PL a
              /               └── P2 (ยากหลีกเลี่ยง) → PL b
          S1 /
            / F2 (บ่อย)──── P1 → PL b
           /               └── P2 → PL c
   Start──┤
           \ C2 (ผลกระทบรุนแรง)
            \          F1 ─── P1 → PL c
          S2 \            └── P2 → PL d
              \ F2 ─── P1 → PL d
               \      └── P2 → PL e

S = Severity  C = Contribution to risk  F = Frequency  P = Possibility to avoid
```

---

## Step 825: PFHD Calculation

### วิธีการคำนวณ PFHD

```
PFHD Calculation Methods:

Method 1: Simple (สำหรับ series architecture ไม่ซับซ้อน)
─────────────────────────────────────────────────────────────
PFHD = Σ PFHD_i (บวก PFHD ของแต่ละ component)

Method 2: Markov Model (สำหรับ redundant systems)
─────────────────────────────────────────────────────────────
1oo2 Architecture:
PFHD_1oo2 = PFHD_1 × PFHD_2 × T_proof / 3

2oo3 Architecture (Voting):
PFHD_2oo3 = 3 × PFHD_1 × PFHD_2 × T_proof (เมื่อ PFHD_1=PFHD_2=PFHD_3)

Method 3: Reliability Block Diagram
─────────────────────────────────────────────────────────────
ใช้ RBD software เช่น Isograph, ReliaSoft

Method 4: SISTEMA Software (Free, from BGIA/IFA)
─────────────────────────────────────────────────────────────
โปรแกรมฟรีจาก German BGIA (now IFA)
รองรับ ISO 13849-1 calculation
```

### ตัวอย่างการคำนวณ PFHD แบบละเอียด

```
ตัวอย่าง: Category 3 E-Stop System

Components:
┌─────────────────────────────────────────────────────────────┐
│  Actuator (E-Stop Button):                                  │
│  ├── B10d = 100,000 cycles (จาก spec sheet)                 │
│  ├── nop = 10 operations/year                               │
│  ├── MTTFd = B10d / (0.1 × nop) = 100,000 / 1 = 100,000 yr│
│  └── λDU = 1 / (2 × MTTFd) = 1/(2×100,000×8760) = 5.7×10⁻¹⁰│
│                                                             │
│  Safety Module Input (F-DI):                               │
│  ├── PFHD from datasheets: 3×10⁻¹⁰/h                      │
│  └── (ใช้ค่าจาก manufacturer certificate)                   │
│                                                             │
│  Safety PLC (F-CPU):                                        │
│  ├── PFHD from datasheets: 1×10⁻⁹/h                       │
│  └── (ใช้ค่าจาก TÜV certificate)                           │
│                                                             │
│  Final Element (Safety Relay):                             │
│  ├── B10d = 200,000 cycles                                 │
│  ├── nop = 50 operations/year                              │
│  ├── MTTFd = 200,000/5 = 40,000 years                     │
│  └── λDU = 1/(2×40,000×8760) = 1.43×10⁻⁹/h               │
│                                                             │
│  PFHD_total = 5.7×10⁻¹⁰ + 3×10⁻¹⁰ + 1×10⁻⁹ + 1.43×10⁻⁹  │
│            = 3.3×10⁻⁹/h                                   │
│                                                             │
│  ✓ Meets SIL 2 (< 10⁻⁷/h) and PL d (< 10⁻⁶/h)          │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 826: Diagnostic Coverage (DC)

### การวัด Diagnostic Coverage

```
DC = (λDD) / (λDD + λDU) × 100%

โดยที่:
- λDD = Dangerous Detected failure rate
- λDU = Dangerous Undetected failure rate

DC Categories:
┌─────────────────────────────────────────────────────────────┐
│  DC < 60%:    None    (Category B, 1)                       │
│  60% ≤ DC < 90%: Low  (Category 2)                         │
│  90% ≤ DC < 99%: Medium (Category 2, 3)                    │
│  DC ≥ 99%:    High   (Category 3, 4)                       │
└─────────────────────────────────────────────────────────────┘
```

### เทคนิคการเพิ่ม Diagnostic Coverage

```
Techniques for High DC:

1. Self-test (ทดสอบตัวเอง)
   └── PLC CPU ทำ self-diagnostic ทุก cycle

2. Cross-checking (เปรียบเทียบ channel)
   └── Dual-channel input: ตรวจสอบว่า Channel A = Channel B

3. Proof test (ทดสอบเป็นระยะ)
   └── ทดสอบ safety function ตามกำหนดเวลา

4. Dynamic test (ทดสอบแบบไดนามิก)
   └── F-DI: ส่ง test pulses เพื่อตรวจสอบ wire break

5. Feedback monitoring
   └── ตรวจสอบ output ผ่าน feedback contact
```

---

## Step 827: CCF (Common Cause Failure)

### การป้องกัน Common Cause Failure

```
CCF Score (IEC 62061 Method):

Measure                              Score
─────────────────────────────────────────────────
Separation/Segregation               15 pts
Diversity (different technology)     20 pts
Design/Application/Experience       20 pts
Assessment/Analysis                 5 pts
Competence/Training                 5 pts
Environmental control               10 pts
Environmental testing               35 pts
─────────────────────────────────────────────────
Maximum Score                       110 pts
Required: ≥ 65 pts for SIL 2/3
          ≥ 65 pts for PL d/e

ตัวอย่าง CCF measures:
- ใช้ different manufacturers สำหรับ dual-channel sensors
- แยก cable routing สำหรับ channel A และ B
- ใช้ different technology (NC + OSSD)
- มีขั้นตอน work permit สำหรับ maintenance
```

---

## Step 828: MTTFd Calculation

### การคำนวณ MTTFd (Mean Time to Dangerous Failure)

```
MTTFd Calculation:

For Electromechanical Components:
───────────────────────────────────────────────────────────────
MTTFd = B10d / (0.1 × nop)

ตัวอย่าง: E-Stop button
B10d  = 100,000 cycles (จาก spec)
nop   = ปริมาณการใช้งาน = 10 ครั้ง/ปี × 1 ปี = 10 ครั้ง/ปี
      = 10/8760 ชั่วโมง ≈ 0.00114/ชั่วโมง

MTTFd = 100,000 / (0.1 × 10) = 100,000 years

λd = 1 / (2 × MTTFd × 8760) = 5.7×10⁻¹⁰/hour

MTTFd Rating:
┌─────────────────────────────────────────────────────────────┐
│  Low:    3 years ≤ MTTFd < 10 years                        │
│  Medium: 10 years ≤ MTTFd < 30 years                       │
│  High:   30 years ≤ MTTFd ≤ 100 years                      │
│                                                             │
│  Note: MTTFd > 100 years ให้ใช้ค่า 100 years             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 829: SISTEMA Software

### การใช้ SISTEMA สำหรับ ISO 13849 Assessment

```
SISTEMA Workflow:
┌─────────────────────────────────────────────────────────────┐
│  1. Create New Project                                      │
│     └── ตั้งชื่อ machine, revision number                  │
│                                                             │
│  2. Define Safety Function                                  │
│     ├── ชื่อ Safety Function (เช่น "Emergency Stop")       │
│     ├── Required PLr (เช่น PLr = d)                        │
│     └── Required SIL (optional)                            │
│                                                             │
│  3. Add Subsystems                                          │
│     ├── Input subsystem (E-Stop button)                    │
│     ├── Logic subsystem (Safety PLC)                       │
│     └── Output subsystem (Contactor)                       │
│                                                             │
│  4. Configure Each Subsystem                               │
│     ├── Category (B, 1, 2, 3, 4)                          │
│     ├── DC (Diagnostic Coverage)                           │
│     ├── MTTFd (from library or manual input)               │
│     └── CCF (Common Cause Failure score)                   │
│                                                             │
│  5. Calculate and Verify                                    │
│     ├── SISTEMA calculates PL achieved                     │
│     └── Compare PL with PLr requirement                    │
│                                                             │
│  6. Generate Report                                         │
│     └── PDF report พร้อม calculation details              │
└─────────────────────────────────────────────────────────────┘
```

### SISTEMA Libraries

```
Available SISTEMA Libraries (ดาวน์โหลดฟรี):
- Siemens SIRIUS Safety Library
- Rockwell Automation Safety Library
- Pilz Safety Library
- Schmersal Safety Library
- SICK Safety Library
- Omron Safety Library

การใช้งาน:
1. ดาวน์โหลด library file (.stdlib)
2. Import ใน SISTEMA
3. เลือก component จาก library
4. ค่า PFHD/MTTFd จะถูกใส่อัตโนมัติ
```

---

## Step 830: Safety Lifecycle

### การจัดการ Safety Lifecycle

```
Safety Lifecycle Management:

Phase 1: HAZARD IDENTIFICATION
───────────────────────────────────────────────────────────────
- HAZOP (Hazard and Operability Study)
- FMEA (Failure Mode and Effect Analysis)  
- What-if Analysis
- Checklist Review
Output: Hazard List, Risk Matrix

Phase 2: RISK ASSESSMENT
───────────────────────────────────────────────────────────────
- Determine risk level (Severity × Frequency)
- Apply risk graph or LOPA
- Determine required SIL/PL
Output: SIL/PLr for each safety function

Phase 3: SAFETY REQUIREMENTS SPECIFICATION (SRS)
───────────────────────────────────────────────────────────────
- Define safety functions
- Define response times
- Define safe states
- Define testing requirements
Output: SRS Document

Phase 4: DESIGN & IMPLEMENTATION
───────────────────────────────────────────────────────────────
- Hardware design (architecture, CCF)
- Software design (F-Program)
- Calculation verification (PFHD)
Output: Design Documents, Code

Phase 5: VERIFICATION & VALIDATION
───────────────────────────────────────────────────────────────
- Factory Acceptance Test (FAT)
- Site Acceptance Test (SAT)
- Functional testing
Output: Test Records, Safety Case

Phase 6: OPERATION & MAINTENANCE
───────────────────────────────────────────────────────────────
- Proof testing schedule
- Maintenance procedures
- MOC (Management of Change)
Output: Maintenance Records

Phase 7: DECOMMISSIONING
───────────────────────────────────────────────────────────────
- Safe decommission plan
- Documentation update
```

---

## Step 831: Functional Safety Management

### FSM (Functional Safety Management)

```
Functional Safety Management Requirements:

1. Competency Management
   ├── FSE (Functional Safety Engineer) training
   ├── Certification (TÜV FSE, CFSE)
   └── Record keeping

2. Configuration Management
   ├── Version control สำหรับ safety documents
   ├── Change management procedure
   └── Software configuration control

3. Audit and Assessment
   ├── Internal safety audit
   ├── Independent safety assessment (ISA)
   └── Pre-startup safety review

4. Functional Safety Plan
   ├── Scope of work
   ├── Safety lifecycle phases
   ├── Resources and responsibilities
   └── Review milestones
```

---

## Step 832-840: Complete PL d Assessment Example

### โครงการ: PL d Assessment สำหรับ Two-Hand Control

#### 1. Safety Function Definition

```
Safety Function: Two-Hand Control for Press
─────────────────────────────────────────────────────────────
Function: ป้องกันไม่ให้กดปุ่ม 2 มือในเวลาเดียวกัน เพื่อป้องกัน
          operator จากการเข้าพื้นที่อันตราย

Inputs:
- Left hand button (LH)  - Momentary, NC+NO contacts  
- Right hand button (RH) - Momentary, NC+NO contacts

Output:
- Press enable signal

Logic: Press ทำงานได้เมื่อ LH AND RH ถูกกดภายใน 500ms

Safe State: Press หยุดทำงาน (de-energized)

Response Time Requirement: ≤ 100ms
```

#### 2. Risk Assessment

```
Risk Graph Analysis:

S2 (Serious injury - loss of finger/hand)
+ F2 (Frequent - operator กดหลายครั้ง/วัน)
+ P2 (Not possible to avoid - press กดทันที)
→ PLr = e

(ต้องการ PL e!)
```

#### 3. Architecture Design (Category 4)

```
Two-Hand Control Architecture (Category 4, PL e):

┌─────────────────────────────────────────────────────────────┐
│  Left Button        Right Button                            │
│  ┌─────────┐        ┌─────────┐                            │
│  │  LH_NO  │        │  RH_NO  │  → Channel A (Normally Open)│
│  │  LH_NC  │        │  RH_NC  │  → Channel B (Normally Closed)│
│  └─────────┘        └─────────┘                            │
│       │                  │                                  │
│  ┌────▼──────────────────▼────────────────────────────┐   │
│  │          Two-Hand Control Module                    │   │
│  │          (e.g., Pilz PNOZ m B0, SIL 3)            │   │
│  │                                                     │   │
│  │  Logic:                                             │   │
│  │  1. Verify both buttons pressed simultaneously     │   │
│  │     (within 500ms synchrony window)               │   │
│  │  2. Verify NC/NO contact integrity                 │   │
│  │  3. Require release before re-enabling             │   │
│  └────────────────────────────┬────────────────────── ┘   │
│                               │                            │
│                    ┌──────────▼──────────┐                 │
│                    │    Safety PLC       │                  │
│                    │    (GuardLogix)     │                  │
│                    └──────────┬──────────┘                 │
│                               │                            │
│                    ┌──────────▼──────────┐                 │
│                    │  Press Solenoid     │                  │
│                    │  (Dual coil valve)  │                  │
│                    └─────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

#### 4. PFHD Calculation

```
Component PFHD Values:
─────────────────────────────────────────────────────────────
Left Button (Category 4, dual-channel):
  B10d = 500,000, nop = 100/year
  MTTFd = 500,000/(0.1×100) = 50,000 years (= 100yr max)
  PFHD_LH = 5.7×10⁻¹⁰/h

Right Button (same as left):
  PFHD_RH = 5.7×10⁻¹⁰/h

Two-Hand Control Module (from Pilz datasheet):
  PFHD_THC = 1×10⁻⁹/h (SIL 3 certified)

Safety PLC (GuardLogix):
  PFHD_PLC = 7.53×10⁻¹⁰/h (from Allen-Bradley safety manual)

Press Solenoid Valve (dual coil):
  B10d = 500,000, nop = 200/year
  MTTFd = 500,000/20 = 25,000 years (= 100yr max)
  PFHD_SV = 5.7×10⁻¹⁰/h (for 1oo2 configuration)

PFHD_total = 5.7×10⁻¹⁰ + 5.7×10⁻¹⁰ + 1×10⁻⁹ + 7.53×10⁻¹⁰ + 5.7×10⁻¹⁰
           = 3.5×10⁻⁹/h

PL e requirement: PFHD < 10⁻⁷/h
3.5×10⁻⁹ < 10⁻⁷ ✓ PASS PL e!
```

#### 5. Ladder Diagram สำหรับ Two-Hand Control

```ladder
// =============================================
// Safety Program: Two_Hand_Control
// SIL 3 / PL e
// =============================================

// Network 1: Left Hand Button Monitoring
// F_FDBACK checks NC and NO contact consistency
+--[F_FDBACK LH_Instance]--+
|  I_CLS := LH_NC_Input    |  // NC contact input
|  I_OPN := LH_NO_Input    |  // NO contact (complementary)
|  S_LEFT => LH_Safe        |  // Safety output
+---------------------------+

// Network 2: Right Hand Button Monitoring
+--[F_FDBACK RH_Instance]--+
|  I_CLS := RH_NC_Input    |
|  I_OPN := RH_NO_Input    |
|  S_RIGHT => RH_Safe       |
+---------------------------+

// Network 3: Synchrony Monitoring (both pressed within 500ms)
+--[F_2HAND_2 Instance]----+
|  S_SETL := LH_Safe       |  // Left hand safe signal
|  S_SETR := RH_Safe       |  // Right hand safe signal
|  S_2H   => Both_Pressed  |  // Both pressed simultaneously
+---------------------------+

// Network 4: Press Enable Output
//  Both_Pressed   No_Error     Press_Enable
//  ----[ ]--------[ ]----------( F-DO )----

+--[ Both_Pressed ]--+--[ NOT Safety_Error ]--+--( Press_Enable_Output )--+
|                    |                        |                           |
| (ทั้ง 2 มือกด)     | (ไม่มี safety error)   | (อนุญาตให้กด press)       |
+--------------------+------------------------+---------------------------+
```

#### 6. Validation Test Procedure

```
Validation Test for Two-Hand Control:

Test 1: Normal Operation
─────────────────────────────────────────────────────────────
กด LH และ RH พร้อมกัน → Press ทำงาน ✓
ปล่อย LH → Press หยุด ✓
ปล่อย RH → Press หยุด ✓

Test 2: Synchrony Window
─────────────────────────────────────────────────────────────
กด LH ก่อน แล้วกด RH ภายใน 500ms → Press ทำงาน ✓
กด LH ก่อน แล้วกด RH หลัง 500ms → Press ไม่ทำงาน ✓

Test 3: Contact Integrity
─────────────────────────────────────────────────────────────
กด LH ปกติ → NC=OPEN, NO=CLOSED ✓
Short circuit บน NC contact → Fault detected ✓
Wire break บน NO contact → Fault detected ✓

Test 4: Anti-tie-down
─────────────────────────────────────────────────────────────
กด RH ค้างไว้ แล้วกด LH → Press ไม่ทำงาน ✓
(ต้องปล่อยทั้ง 2 มือก่อนเสมอ)

Test 5: Response Time
─────────────────────────────────────────────────────────────
ปล่อย LH → วัดเวลาจนกว่า press หยุด < 100ms ✓
```

---

## Lab Project 42: Complete Functional Safety Assessment

### โจทย์
ทำ complete functional safety assessment สำหรับ Automatic Palletizer

### ข้อกำหนด
1. ทำ HAZOP/FMEA เพื่อหา hazards ทั้งหมด
2. กำหนด PLr สำหรับทุก safety function
3. ออกแบบ architecture สำหรับ PL d requirement
4. คำนวณ PFHD ด้วย SISTEMA
5. เขียน SRS (Safety Requirements Specification)
6. ทำ Validation Test Plan

### Safety Functions ที่ต้องพิจารณา
- Emergency Stop (หลายจุด)
- Area Scanner (พื้นที่การทำงาน robot)
- Pallet presence detection (ป้องกัน collision)
- Gripper open/close monitoring
- Robot servo power enable

---

## สรุป Part 42

| มาตรฐาน | ขอบเขต | ตัวชี้วัด |
|---------|--------|---------|
| IEC 61508 | ทุกอุตสาหกรรม | SIL 1-4, PFH/PFD |
| IEC 62061 | Machinery (electrical) | SIL, PFHD |
| ISO 13849 | Machinery control | PL a-e, Category |
| IEC 61511 | Process industry | SIL, PFD |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 41: Safety PLC และ SIL](part41_safety_plc.md) |
| ► ถัดไป | [Part 43: Redundancy Systems](part43_redundancy.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
