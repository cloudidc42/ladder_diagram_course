# Part 41: Safety PLC และ SIL (Steps 801-820)

## ภาพรวม
Safety PLC เป็นระบบควบคุมที่ออกแบบมาเพื่อความปลอดภัยของชีวิตและทรัพย์สิน โดยต้องผ่านการรับรองตามมาตรฐาน IEC 61508 และ SIL (Safety Integrity Level)

---

## Step 801: Safety PLC Architecture (F-CPU)

### โครงสร้าง Safety PLC

```
┌─────────────────────────────────────────────────────────────┐
│                    Safety PLC Architecture                   │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   F-CPU 1    │◄──►│   F-CPU 2    │    │  Standard    │  │
│  │ (Primary)    │    │ (Redundant)  │    │   CPU        │  │
│  │  Safety OS   │    │  Safety OS   │    │  (Optional)  │  │
│  └──────┬───────┘    └──────┬───────┘    └──────────────┘  │
│         │                  │                               │
│  ┌──────▼──────────────────▼───────┐                       │
│  │        F-Bus / Safety Bus        │                       │
│  └──────┬────────────┬─────────────┘                       │
│         │            │                                     │
│  ┌──────▼──────┐ ┌───▼──────────┐                          │
│  │  F-DI Module│ │ F-DO Module  │                          │
│  │  (Fail-safe)│ │ (Fail-safe)  │                          │
│  │  Dual-chan. │ │ Dual-chan.   │                          │
│  └─────────────┘ └──────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

### คุณสมบัติหลักของ F-CPU (Fail-safe CPU)

| คุณสมบัติ | รายละเอียด |
|-----------|-----------|
| Dual Core Processing | ประมวลผล 2 ช่องทางแล้วเปรียบเทียบผล |
| CRC Checking | ตรวจสอบ checksum ของโปรแกรมและข้อมูล |
| Watchdog Timer | ตรวจสอบเวลาการทำงานของโปรแกรม |
| Memory Testing | ทดสอบ RAM/ROM ระหว่างการทำงาน |
| F-Address | แต่ละ I/O มี unique address สำหรับความปลอดภัย |

```
// Siemens S7-1500F - การกำหนดค่า CPU ความปลอดภัย
// ใน TIA Portal: CPU Properties > F-Parameters

F_ADDRESS   := 1;        // F-Address ของ CPU (ต้องไม่ซ้ำในระบบ)
F_CRC       := TRUE;     // เปิดใช้งาน CRC checking
F_WATCHDOG  := 150ms;    // Watchdog timeout
MAX_CYCLE   := 100ms;    // เวลา cycle สูงสุด (ต้องกำหนดสำหรับ Safety)
```

---

## Step 802: SIL Levels (SIL 1-3)

### ระดับ Safety Integrity Level

```
┌──────────────────────────────────────────────────────────────┐
│              SIL Level Comparison                            │
│                                                              │
│  SIL 4 ─── ความน่าเชื่อถือสูงสุด (Nuclear, Aerospace)        │
│            PFHd: 10⁻⁹ to 10⁻⁸                               │
│                                                              │
│  SIL 3 ─── ความน่าเชื่อถือสูง (Chemical, Oil & Gas)          │
│            PFHd: 10⁻⁸ to 10⁻⁷                               │
│                                                              │
│  SIL 2 ─── ความน่าเชื่อถือดี (Machine Safety, Process)       │
│            PFHd: 10⁻⁷ to 10⁻⁶                               │
│                                                              │
│  SIL 1 ─── ความน่าเชื่อถือปานกลาง (Simple safety)           │
│            PFHd: 10⁻⁶ to 10⁻⁵                               │
└──────────────────────────────────────────────────────────────┘
```

### การเปรียบเทียบ SIL กับ PL (Performance Level)

| SIL | PL (ISO 13849) | PFHd (ต่อชั่วโมง) | ตัวอย่างการใช้งาน |
|-----|----------------|-------------------|-----------------|
| SIL 1 | PL c | 10⁻⁶ to 10⁻⁵ | Guard monitoring |
| SIL 2 | PL d | 10⁻⁷ to 10⁻⁶ | Emergency stop |
| SIL 3 | PL e | 10⁻⁸ to 10⁻⁷ | Burner management |
| SIL 4 | N/A | 10⁻⁹ to 10⁻⁸ | Nuclear reactor |

---

## Step 803: Safety Integrity Level Determination

### กระบวนการกำหนด SIL - LOPA (Layer of Protection Analysis)

```
LOPA Worksheet ตัวอย่าง: Reactor Overpressure

┌─────────────────────────────────────────────────────────────┐
│  อันตราย: Reactor Overpressure → Explosion                  │
│                                                             │
│  ความถี่การเกิดเหตุ (Initiating Event):                      │
│  Control valve fails open: 0.1/year                        │
│                                                             │
│  Independent Protection Layers (IPLs):                     │
│  ├── BPCS (Basic Process Control): PFD = 0.1               │
│  ├── Alarm + Operator response: PFD = 0.1                  │
│  └── Safety Instrumented Function: PFD = ?                 │
│                                                             │
│  Tolerable Risk: 10⁻⁵/year (ตามมาตรฐาน)                    │
│                                                             │
│  คำนวณ: 0.1 × 0.1 × 0.1 × PFD_SIF = 10⁻⁵                 │
│         PFD_SIF = 10⁻⁵ / (0.1 × 0.1 × 0.1)               │
│         PFD_SIF = 10⁻² = 0.01                             │
│                                                             │
│  PFD = 0.01 → SIL 2 (PFD range: 10⁻² to 10⁻³)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 804: TÜV Certification

### ความสำคัญของ TÜV

TÜV (Technischer Überwachungsverein) เป็นองค์กรรับรองมาตรฐานความปลอดภัยจากเยอรมนี

```
กระบวนการขอรับรอง TÜV:

1. Concept Review
   └── ตรวจสอบ Safety Concept และ Risk Assessment

2. Design Review  
   └── ตรวจสอบ Hardware/Software Design

3. Type Testing
   └── ทดสอบฮาร์ดแวร์จริง (EMC, Temperature, Vibration)

4. Software Assessment
   └── ตรวจสอบ Software Development Process

5. Production Assessment
   └── ตรวจสอบกระบวนการผลิต

6. Certificate Issuance
   └── ออกใบรับรอง (มีอายุ 5 ปี)
```

### อุปกรณ์ที่ผ่านการรับรอง TÜV

| ผู้ผลิต | ผลิตภัณฑ์ | SIL Rating | Certificate |
|---------|----------|-----------|------------|
| Siemens | S7-1500F CPU 1516F | SIL 3 | TÜV Rheinland |
| Siemens | ET 200SP F-DI/F-DO | SIL 3 | TÜV Rheinland |
| Rockwell | ControlLogix 5580 | SIL 3 | TÜV SÜD |
| Rockwell | GuardLogix 5580 | SIL 3 | TÜV SÜD |
| ABB | AC500-S | SIL 3 | TÜV Nord |
| Pilz | PSS 4000 | SIL 3 | TÜV Rheinland |

---

## Step 805: Siemens S7-1500F

### สถาปัตยกรรม S7-1500F

```
S7-1500F System Layout:
┌─────────────────────────────────────────────────────────────┐
│  PS 25W 24VDC        CPU 1516F-3 PN/DP        IM 155-5 PN  │
│  ┌──────────┐        ┌──────────────────┐    ┌────────────┐ │
│  │  Power   │        │   CPU 1516F      │    │ ET200SP   │ │
│  │  Supply  │        │   - 5MB work mem │    │ Remote IO │ │
│  │  Module  │        │   - Safety OS    │    │           │ │
│  └──────────┘        │   - PROFIsafe    │    └────────────┘ │
│                      │   - PROFINET     │                   │
│                      └────────┬─────────┘                  │
│                               │ PROFIsafe                  │
│  ┌────────────────────────────▼─────────────────────────┐  │
│  │              F-IO Modules (ET200SP)                   │  │
│  │   F-DI 8x24VDC   F-DO 4x24VDC/2A   F-AI 4xU/I/RTD  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### การกำหนดค่า PROFIsafe ใน TIA Portal

```
// PROFIsafe Parameters
// Device: ET200SP F-Module
// Configuration in TIA Portal > Device Properties

F_Source_Add    := 1;      // F-Address ของ F-CPU
F_Dest_Add      := 100;    // F-Address ของ F-Module (ต้องไม่ซ้ำ)
F_WD_Time       := 150;    // Watchdog time (ms)
F_iPar_CRC      := 16#1234ABCD;  // CRC ของ iParameter
```

---

## Step 806: Rockwell GuardLogix

### GuardLogix Architecture

```
GuardLogix System:
┌─────────────────────────────────────────────────────────────┐
│                  GuardLogix 5580 Controller                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Primary Controller        Safety Controller        │   │
│  │  (Standard tasks)          (Safety tasks only)      │   │
│  │                                                     │   │
│  │  Standard I/O              Safety I/O               │   │
│  │  Standard Comm             CIP Safety comm          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Safety Network:                                            │
│  EtherNet/IP + CIP Safety Protocol                         │
│                                                             │
│  Compatible Safety I/O:                                    │
│  - 1791DS-IB12  (12 Safety DI)                            │
│  - 1791DS-IB12XOB4 (12 DI + 4 DO Safety)                  │
│  - 1791DS-OB8   (8 Safety DO)                             │
└─────────────────────────────────────────────────────────────┘
```

### RSLogix 5000 / Studio 5000 - Safety Task Configuration

```
// GuardLogix Safety Task Configuration
// Task Type: Safety (ต้องเลือก Safety Task เท่านั้น)
// Period: 10ms (ต้องไม่เกิน 500ms)
// Watchdog: 500ms

// Safety I/O Tag Declaration
SAFETY_ESTOP_IN     : SAFETY_INPUT   // Emergency stop input
SAFETY_GUARD_IN     : SAFETY_INPUT   // Guard door input  
SAFETY_RESET_IN     : SAFETY_INPUT   // Reset pushbutton
SAFETY_CONTACTOR_OUT: SAFETY_OUTPUT  // Main contactor output
```

---

## Step 807: Safety I/O Modules

### ประเภทของ Safety I/O

```
Safety Digital Input Module (F-DI):
┌─────────────────────────────────────────────────────────────┐
│  Dual-Channel Input (Category 3/4, PL d/e)                  │
│                                                             │
│  Channel A ──┬──► Comparator ──► Safety Logic              │
│              │         ▲                                   │
│  Channel B ──┴──────────┘                                  │
│                                                             │
│  Features:                                                  │
│  - Discrepancy monitoring (ตรวจจับความไม่สอดคล้อง)           │
│  - Cross-circuit detection                                  │
│  - Short circuit detection                                  │
│  - Pulse testing (Test pulses สำหรับตรวจจับ wire break)     │
└─────────────────────────────────────────────────────────────┘

Safety Digital Output Module (F-DO):
┌─────────────────────────────────────────────────────────────┐
│  Dual-Channel Output with Feedback                          │
│                                                             │
│  Safety CPU ─► Channel A ──┬──► Load                       │
│                            │    (e.g., Contactor coil)     │
│              Channel B ────┘                               │
│                                                             │
│  Feedback ◄── Status monitoring ◄── Load                  │
│                                                             │
│  Features:                                                  │
│  - Output current monitoring                               │
│  - Load feedback verification                              │
│  - De-energize on fault (Fail-safe state = OFF)            │
└─────────────────────────────────────────────────────────────┘
```

### การคำนวณ PFD ของ Safety I/O Module

```
// PFD Calculation Example
// Siemens F-DI Module 1oo2 (1 out of 2) configuration
// λDU = 0.5 × 10⁻⁹ per hour (Dangerous Undetected failure rate)
// T_proof = 8760 hours (1 year proof test interval)
// DC = 99% (Diagnostic Coverage)

// PFD calculation for 1oo2 system:
// PFD_1oo2 = λDU² × (T_proof/3)²
// PFD_1oo2 = (0.5×10⁻⁹)² × (8760/3)²
// PFD_1oo2 = 0.25×10⁻¹⁸ × 8,497,600
// PFD_1oo2 ≈ 2.1 × 10⁻¹² (ดีกว่า SIL 3!)
```

---

## Step 808: F-DB and F-Program

### F-Data Block (F-DB)

F-DB คือ Data Block พิเศษที่ใช้เก็บข้อมูลความปลอดภัย

```pascal
// Siemens TIA Portal - F-DB Declaration
// DB Type: F-global DB (ต้องเลือก "F-DB" attribute)

DATA_BLOCK "Safety_DB"
{ S7_Optimized_Access := 'FALSE' }  // ต้องใช้ Classic (non-optimized) สำหรับ F-DB
VERSION : 0.1
NON_RETAIN
VAR
    E_Stop_Status    : Bool;     // สถานะ Emergency Stop
    Guard_Status     : Bool;     // สถานะ Guard Door
    Safety_OK        : Bool;     // Safety Output OK
    F_ACK            : Bool;     // F-Acknowledge bit
    Error_Code       : Int;      // Error code
END_VAR

BEGIN
END_DATA_BLOCK
```

### F-Program Structure

```
F-Program Organization (Siemens):
┌─────────────────────────────────────────────────────────────┐
│  F-OB (F-Organization Block)                                │
│  └── OB 123: F-Startup (ทำงานเมื่อเริ่มต้น Safety mode)    │
│                                                             │
│  F-FB (F-Function Block) - Safety Function Blocks          │
│  ├── F_ESTOP1: Emergency Stop monitoring                   │
│  ├── F_FDBACK: Feedback monitoring                         │
│  ├── F_TOF: Safety Timer Off-delay                         │
│  └── F_TON: Safety Timer On-delay                          │
│                                                             │
│  F-FC (F-Function) - Safety Functions                      │
│  ├── Custom safety logic                                   │
│  └── Complex calculations                                  │
│                                                             │
│  NOTA: F-programs ต้องเรียกจาก F-OB เท่านั้น!              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 809: Safety Program Restrictions

### ข้อจำกัดของโปรแกรม Safety

```
Safety Program ไม่สามารถใช้:
┌─────────────────────────────────────────────────────────────┐
│  PROHIBITED in F-Program:                                   │
│                                                             │
│  ✗  Indirect addressing (L[AR1, P#0.0])                    │
│  ✗  CASE...OF statement (บาง PLC)                          │
│  ✗  Pointers (POINTER, ANY type ใน Siemens)               │
│  ✗  Standard function blocks ที่ไม่ได้รับรอง               │
│  ✗  Jump instructions (JMP, JC, JNBI) บาง PLC             │
│  ✗  STOP instruction ภายใน Safety code                    │
│  ✗  การเขียนไปยัง non-F DB จาก F-program                  │
│  ✗  Recursive calls                                        │
│                                                             │
│  ALLOWED in F-Program:                                     │
│                                                             │
│  ✓  F-certified function blocks เท่านั้น                   │
│  ✓  Direct addressing                                      │
│  ✓  Boolean, Integer, Real operations                      │
│  ✓  Certified Timer/Counter blocks                         │
│  ✓  F-DB data                                             │
└─────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง F-Program ใน Siemens (LAD)

```
// F-FB "Safety_EStop_Control"
// เรียกจาก Safety OB (F-OB)

// Network 1: Emergency Stop Monitoring
// ใช้ F_ESTOP1 (Safety Function Block)

CALL "F_ESTOP1", "Safety_EStop_DB"
   E_STOP   := "F_DI_Safety".E_Stop_Channel_A  // Input จาก F-DI
   S_ESTOP  := "Safety_DB".E_Stop_Status        // Safety Status output
   ACK      := "Safety_DB".F_ACK               // Acknowledge input
   ACK_REI  := "Safety_DB".ACK_REI             // Re-initialization ACK

// Network 2: Guard Door Monitoring  
// ใช้ F_FDBACK (Feedback Monitoring)
CALL "F_FDBACK", "Safety_Guard_DB"
   I_CLS    := "F_DI_Safety".Guard_Door_A      // Guard closed input
   I_OPN    := "F_DI_Safety".Guard_Door_B      // Guard open input (N.C.)
   S_GUARD  := "Safety_DB".Guard_Status        // Safety output
```

---

## Step 810: Proof Test Requirements

### การทดสอบ Proof Test

```
Proof Test Procedure สำหรับ Safety Function:

┌─────────────────────────────────────────────────────────────┐
│  SIL 2 Emergency Stop System                                │
│  Proof Test Interval: 1 ปี (8760 ชั่วโมง)                  │
│                                                             │
│  Step 1: เตรียมการ (ทำในขณะ production หยุด)               │
│  ├── Notify all personnel                                   │
│  ├── Issue work permit                                      │
│  └── Document test record                                   │
│                                                             │
│  Step 2: ทดสอบ E-Stop Button                              │
│  ├── กด E-Stop button (ทุกตัว)                             │
│  ├── ตรวจสอบว่า Safety output = OFF                        │
│  ├── ตรวจสอบ response time < 100ms                         │
│  └── บันทึกผลการทดสอบ                                      │
│                                                             │
│  Step 3: ทดสอบ Safety I/O                                  │
│  ├── ทดสอบ channel A และ B แยกกัน                          │
│  ├── ทดสอบ discrepancy fault detection                     │
│  └── ทดสอบ cross-circuit detection                         │
│                                                             │
│  Step 4: ทดสอบ Final Element (Contactor/Valve)             │
│  ├── ตรวจสอบ contactor dropout                             │
│  ├── ตรวจสอบ feedback signal                              │
│  └── วัดเวลาตอบสนอง                                        │
│                                                             │
│  Step 5: บันทึกและรายงาน                                   │
│  ├── บันทึกผลทุก step                                      │
│  ├── ลงนามผู้ทดสอบและผู้ตรวจสอบ                           │
│  └── อัพเดท Safety Case Document                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 811-820: Complete SIL 2 E-Stop System Design

### โครงการ: SIL 2 Emergency Stop System สำหรับ Robot Cell

#### 1. Risk Assessment

```
FMEA/HAZOP Results:
┌──────────────────────────────────────────────────────────────┐
│  อันตราย: บุคคลเข้าพื้นที่ Robot ขณะทำงาน                   │
│                                                             │
│  Severity: Catastrophic (S3) - อาจเสียชีวิต               │
│  Frequency: Possible (F2) - เกิดขึ้นได้บ่อย               │
│  Probability of Avoidance: Difficult (P2) - หลีกเลี่ยงยาก  │
│                                                             │
│  Risk = S3 + F2 + P2 → ต้องการ SIL 2 / PL d               │
└──────────────────────────────────────────────────────────────┘
```

#### 2. System Architecture Design

```
SIL 2 E-Stop Architecture:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  E-Stop Button (Dual-channel NC contacts)                  │
│  ┌──────┐   Channel A ──────► F-DI CH_A ──┐               │
│  │  🔴  │                                  ├─► F-CPU       │
│  │E-STOP│   Channel B ──────► F-DI CH_B ──┘   Safety      │
│  └──────┘                                     Logic        │
│                                                  │         │
│  Door Guard (Coded Magnetic Switch)              │         │
│  ┌──────┐   Channel A ──────► F-DI CH_A ──┐    ▼         │
│  │ DOOR │                                  ├─► AND Gate   │
│  │GUARD │   Channel B ──────► F-DI CH_B ──┘    │         │
│  └──────┘                                       ▼         │
│                                          F-DO ──► Relay   │
│  Light Curtain (OSSD1, OSSD2)                    │         │
│  ┌──────┐   OSSD1 ──────────► F-DI CH_A ──┐    ▼         │
│  │  |||  │                                 ├─► K1 (Main   │
│  │  |||  │   OSSD2 ──────────► F-DI CH_B ──┘  Contactor) │
│  └──────┘                                                  │
└─────────────────────────────────────────────────────────────┘
```

#### 3. Ladder Diagram - Safety Function

```ladder
// =============================================
// Safety Program: Robot_Cell_Safety
// F-FB: SafetyRobotCell
// SIL 2 Certified
// =============================================

// Network 1: Emergency Stop Monitoring
// F_ESTOP1 (TÜV certified function block)
+-------[F_ESTOP1]-------+
|  E_STOP  :=  E_Stop_A  |  // Input จาก F-DI (Channel A+B รวม)
|  S_ESTOP =>  EStop_Safe |  // Safety output
|  ACK     :=  Reset_PB   |  // Reset button
+------------------------+

// Network 2: Door Guard Monitoring
+-------[F_FDBACK]-------+
|  I_CLS  := Door_Closed  |  // Guard closed signal
|  I_OPN  := Door_Open    |  // Guard open signal (complementary)
|  S_GUARD => Guard_Safe  |  // Safety output  
+------------------------+

// Network 3: Light Curtain Monitoring
+-------[F_OSSD]----------+
|  OSSD1  := LightCurt_1  |  // OSSD output 1
|  OSSD2  := LightCurt_2  |  // OSSD output 2
|  S_OSSD => Curtain_Safe |  // Safety output
+-------------------------+

// Network 4: Safety Output Logic
//    EStop_Safe   Guard_Safe   Curtain_Safe   Robot_Enable
//    ---[/]---+---[/]---+---[/]---+-----------( )---
//             |                   |
//  NOTE: ทุก safety input ต้องเป็น TRUE (ไม่มีอันตราย)
//        ก่อนจึงจะ enable robot ได้

  +--[ /FAULT ]-+-[ EStop_Safe ]-+-[ Guard_Safe ]-+-[ Curtain_Safe ]-+--( Robot_Enable )--+
  |             |                |                |                  |                    |
  |  (ตรวจสอบ  | (E-Stop ไม่ถูก | (ประตูปิด     | (ไม่มีคนใน     |   (อนุญาตให้        |
  |   ไม่มี    |  กด = Safe)    |  สมบูรณ์)      |  พื้นที่)        |   Robot ทำงาน)     |
  |   F-Error) |                |                |                  |                    |
  +------------------------------------------------------------------+--------------------+

// Network 5: Safety Contactor Control
//    Robot_Enable   K1_Feedback   
//    ---[ ]---+---[ ]---+-----------( F-DO K1_Coil )---
                                   
  +--[ Robot_Enable ]--+--[ K1_Feedback ]--+--( Safety_Contactor_K1 )--+
  |                    |                   |                            |
  | (Robot enabled)    | (Contactor OK)    | (Energize contactor)       |
  +--------------------+-------------------+----------------------------+
```

#### 4. Safety Response Time Calculation

```
Safety Response Time Calculation (SIL 2):

┌─────────────────────────────────────────────────────────────┐
│  Process Safety Time (PST): 500ms (จาก Risk Assessment)    │
│                                                             │
│  Maximum Allowable Response Time = PST × 0.5 = 250ms       │
│                                                             │
│  Actual Response Time:                                      │
│  ├── Sensor response time:    10ms  (E-Stop button)        │
│  ├── F-DI filter time:        5ms   (Hardware filter)      │
│  ├── F-CPU cycle time:        10ms  (Safety OB)            │
│  ├── PROFIsafe protocol:      3ms   (Transmission)         │
│  ├── F-DO response time:      3ms   (Output module)        │
│  ├── Contactor dropout time:  20ms  (K1 contactor)         │
│  └── Robot brake time:       150ms  (Robot specification)  │
│                                                             │
│  Total: 10+5+10+3+3+20+150 = 201ms < 250ms ✓              │
│                                                             │
│  ✓ Meets SIL 2 requirement!                                │
└─────────────────────────────────────────────────────────────┘
```

#### 5. PFH Calculation

```
PFH (Probability of Failure per Hour) Calculation:

Component PFH values:
- E-Stop Button (Dual-channel 1oo2): PFH = 5×10⁻⁹/h
- F-DI Module (1oo2):                PFH = 3×10⁻¹⁰/h
- F-CPU 1516F:                       PFH = 1×10⁻⁹/h
- F-DO Module (1oo2):                PFH = 4×10⁻¹⁰/h
- Safety Relay K1 (Dual):            PFH = 8×10⁻⁹/h

Total PFH = Sum of all components
         = 5×10⁻⁹ + 3×10⁻¹⁰ + 1×10⁻⁹ + 4×10⁻¹⁰ + 8×10⁻⁹
         = 14.7×10⁻⁹/h
         ≈ 1.47 × 10⁻⁸/h

SIL 2 requirement: PFH < 10⁻⁷/h
1.47 × 10⁻⁸ < 10⁻⁷ ✓ PASS SIL 2!
```

#### 6. Complete Program พร้อม Error Handling

```pascal
// =============================================
// F-FB: SafetyRobotCell
// Version: 1.0
// SIL 2 / PL d Certified Function
// =============================================

FUNCTION_BLOCK SafetyRobotCell
{ S7_Optimized_Access := 'FALSE' }  // Required for F-FB
VERSION : 1.0
// Input Parameters
VAR_INPUT
    EStop_Channel_A  : Bool;  // E-Stop ช่องที่ 1 (NC contact)
    EStop_Channel_B  : Bool;  // E-Stop ช่องที่ 2 (NC contact)
    Door_Channel_A   : Bool;  // Guard door ช่องที่ 1
    Door_Channel_B   : Bool;  // Guard door ช่องที่ 2
    LCurtain_OSSD1   : Bool;  // Light curtain OSSD1
    LCurtain_OSSD2   : Bool;  // Light curtain OSSD2
    Reset_Button     : Bool;  // Reset pushbutton (Momentary)
    K1_Feedback      : Bool;  // Contactor K1 auxiliary contact
END_VAR

// Output Parameters
VAR_OUTPUT
    Safety_Output    : Bool;  // Main safety output (TRUE = Safe, enable robot)
    F_Error          : Bool;  // Safety function error
    Error_ID         : Int;   // Error identification code
END_VAR

// Static Variables (Retained in F-DB)
VAR
    EStop_Instance   : F_ESTOP1;  // E-Stop SFB instance
    Guard_Instance   : F_FDBACK;  // Feedback monitoring instance
    OSSD_Instance    : F_OSSD;    // OSSD monitoring instance
    EStop_OK         : Bool;
    Guard_OK         : Bool;
    OSSD_OK          : Bool;
    Output_OK        : Bool;
    Latched_Error    : Bool;      // Error latch
END_VAR

BEGIN
    // ----------------------------------------
    // Network 1: E-Stop Monitoring
    // ----------------------------------------
    EStop_Instance(
        E_STOP  := NOT EStop_Channel_A,  // Convert NC to active-high
        ACK     := Reset_Button,
        S_ESTOP => EStop_OK
    );

    // ----------------------------------------
    // Network 2: Door Guard Monitoring
    // ----------------------------------------
    Guard_Instance(
        I_CLS  := Door_Channel_A,
        I_OPN  := NOT Door_Channel_B,   // Complementary channel
        S_GUARD => Guard_OK
    );

    // ----------------------------------------
    // Network 3: Light Curtain (OSSD)
    // ----------------------------------------
    OSSD_Instance(
        OSSD1  := LCurtain_OSSD1,
        OSSD2  := LCurtain_OSSD2,
        S_OSSD => OSSD_OK
    );

    // ----------------------------------------
    // Network 4: Safety Output Logic
    // ----------------------------------------
    IF EStop_OK AND Guard_OK AND OSSD_OK AND NOT Latched_Error THEN
        Output_OK := TRUE;
    ELSE
        Output_OK := FALSE;
    END_IF;

    // ----------------------------------------
    // Network 5: Output with Feedback Verification
    // ----------------------------------------
    Safety_Output := Output_OK;

    // Verify feedback (K1 should close when output is TRUE)
    IF Output_OK AND NOT K1_Feedback THEN
        // Contactor did not close → Hardware fault
        Latched_Error := TRUE;
        Error_ID := 1001;  // Contactor fail-to-close fault
    END_IF;

    IF NOT Output_OK AND K1_Feedback THEN
        // Contactor did not open → Welded contact fault
        Latched_Error := TRUE;
        Error_ID := 1002;  // Contactor welded contact fault
    END_IF;

    // Reset latched error (ต้องการ Reset Button กด)
    IF Reset_Button AND NOT Output_OK THEN
        Latched_Error := FALSE;
        Error_ID := 0;
    END_IF;

    // Output error status
    F_Error := Latched_Error OR EStop_Instance.ERROR OR Guard_Instance.ERROR;

END_FUNCTION_BLOCK
```

---

## Lab Project 41: SIL 2 E-Stop System for Press Machine

### โจทย์
ออกแบบและ implement ระบบ Safety E-Stop สำหรับ Hydraulic Press Machine ที่ต้องการ SIL 2

### ข้อกำหนด
1. E-Stop buttons: 4 จุดรอบเครื่อง (Dual-channel)
2. Light curtains: ด้านหน้า 2 ชุด (Front access)
3. Interlocked guards: ด้านข้างและด้านหลัง
4. Two-hand control: สำหรับ operator (ป้องกันมือถูกกด)
5. Muting function: สำหรับ material feeding

### งานที่ต้องทำ
- [ ] Risk assessment และกำหนด SIL requirement
- [ ] เลือก hardware components (ระบุ part number)
- [ ] คำนวณ PFH ของระบบ
- [ ] เขียน Safety Program (F-FB)
- [ ] ทำ Proof Test Procedure document
- [ ] ทำ Safety Case document

---

## สรุป Part 41

| หัวข้อ | สาระสำคัญ |
|--------|----------|
| F-CPU | ประมวลผลแบบ Dual-channel พร้อม CRC checking |
| SIL Levels | SIL 1-4 กำหนดโดย PFH/PFD ของระบบ |
| TÜV | การรับรองมาตรฐานความปลอดภัยโดย third party |
| F-DB/F-Program | โปรแกรมและข้อมูลพิเศษสำหรับ Safety functions |
| Proof Test | การทดสอบสม่ำเสมอเพื่อรักษา SIL rating |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 40: Advanced Diagnostics](../part21-40/part40_advanced_diagnostics.md) |
| ► ถัดไป | [Part 42: Functional Safety IEC 61508/62061/ISO 13849](part42_functional_safety.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
