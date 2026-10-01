# Part 76: DCS กับ PLC Integration (Steps 1501-1520)

## การบูรณาการระบบ DCS และ PLC ในโรงงานกระบวนการ

---

### 🔗 Navigation
- [← Part 75: Advanced HMI Design](part75_advanced_hmi.md)
- [→ Part 77: Advanced Safety System](part77_advanced_safety.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: DCS กับ PLC ต่างกันอย่างไร

ในโรงงานอุตสาหกรรมกระบวนการ (Chemical, Refinery, Power Plant, Pharma) มักพบทั้ง DCS และ PLC ทำงานร่วมกัน การเข้าใจความแตกต่างและการบูรณาการเป็นทักษะสำคัญ

### เปรียบเทียบ DCS กับ PLC

```
DCS (Distributed Control System):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Vendor: Yokogawa, Honeywell, ABB, Emerson, Siemens PCS7
  
  ข้อดี:
  ✓ Redundancy ทุกระดับ (CPU, Network, I/O, Power)
  ✓ Process Libraries ครบ (PID Blocks, Alarm Management)
  ✓ Scalability สูง (1,000-100,000+ I/O)
  ✓ Long Lifecycle Support (15-25 ปี)
  ✓ HAZOP/SIL Ready
  ✓ Native Historian Integration
  ✓ IEC 61511 Safety Certified (บางรุ่น)
  
  ข้อเสีย:
  ✗ ราคาแพงมาก ($500/I/O - $2,000/I/O)
  ✗ Vendor Lock-in สูง
  ✗ ต้องการ Specialist Support
  ✗ Hardware Customization จำกัด
  
Suitable For: Continuous Process, Critical Plant, High I/O Count

PLC (Programmable Logic Controller):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Vendor: Siemens, Rockwell, Mitsubishi, Omron, Schneider

  ข้อดี:
  ✓ ราคาถูกกว่า DCS ($50/I/O - $200/I/O)
  ✓ Programming ยืดหยุ่น (IEC 61131-3)
  ✓ Fast Scan (< 1ms possible)
  ✓ Modular Hardware
  ✓ Many Vendors = Competition = Price Down
  
  ข้อเสีย:
  ✗ Redundancy ต้อง Custom Design
  ✗ Process Library น้อยกว่า DCS
  ✗ Historian ต้อง Add Third Party
  
Suitable For: Discrete Manufacturing, Safety Systems, BOP, Small Plant
```

---

## Step 1501: DCS Architecture - Yokogawa CENTUM VP

### ระบบ DCS ยุคใหม่

```
Yokogawa CENTUM VP Architecture:
  
  HIS (Human Interface Station)
       ↕ Vnet/IP (100 Mbps Ethernet Ring)
  ENG (Engineering Station)
       ↕ Vnet/IP
  FCS (Field Control Station) × N
       ↕ ESB Bus (ESB = Extended Serial Bus)
  FCU (Field Control Unit)
       ↕ RIO Bus หรือ HART
  Field Devices (Sensors, Actuators)
  
FCS Specs:
  - CPU: Dual Redundant
  - Scan: 0.1-2 seconds (user configurable)
  - Max Tags: 3,000 per FCS
  - I/O: Up to 2,048 Channels
  - Languages: Function Block Diagram (FBD)

HIS Functions:
  - Process Overview
  - Alarm Management
  - Trend Display
  - Batch Recipe
  - Maintenance
```

---

## Step 1502: HART Protocol Integration

### HART (Highway Addressable Remote Transducer)

```
HART Protocol:
  - ทับซ้อนบน 4-20mA (Dual Signal!)
  - 4-20mA: ค่าหลัก (Process Variable)
  - HART: ข้อมูลเพิ่มเติม (Device Info, Config, Diagnostics)
  
HART Signal:
  ───────────────────────────────────────
  4-20mA Analog Signal (Primary Variable)
    + 
  HART Digital Signal (1200/2200 Hz FSK)
  ───────────────────────────────────────
  
ข้อมูลที่ส่งผ่าน HART:
  - Device Tag, Serial Number
  - 4 Variables (PV, SV, TV, QV)
  - Alarm Status, Error Flags
  - Last Calibration Date
  - Self-diagnostic results
  - Configuration (Range, Units)
  
ตัวอย่าง: Endress+Hauser Promag Flow Meter
  4-20mA: Flow Rate (Primary)
  HART: Flow Rate + Totalizer + Alarm + Density + Temperature
```

**HART Integration in DCS:**

```
Honeywell Experion PKS + HART:
  
  Field Device → 4-20mA + HART → Multiplexer → DCS Controller
  
  HART Multiplexer: รับ HART จากหลาย Device
  เช่น VIATOR HART Multiplexer (จาก MacTek)
  ส่งข้อมูลไปยัง DCS ผ่าน Serial/Ethernet
  
  Benefits:
  - ไม่ต้องเดิน 4-wire (RS485) เพิ่ม
  - ใช้สาย 4-20mA เดิม
  - Remote Configuration (ไม่ต้องไป Field)
  - Plant Asset Management (PAM) Integration
```

---

## Step 1503: DCS-PLC Gateway Design

### การเชื่อมต่อ DCS กับ PLC

```
Scenarios ที่ต้องมี DCS-PLC Gateway:
  1. PLC ควบคุม Package Unit (Compressor, Boiler Package)
     → ต้องส่งข้อมูลไป DCS ให้ Operator เห็น
     
  2. Safety PLC (SIS/ESD) → ส่ง Status ไป DCS (Non-Safety Read)
  
  3. Legacy PLC ที่ไม่ต้อง Upgrade
     → Gateway ทำให้สื่อสารกับ DCS ได้
     
  4. Batch Process: PLC ควบคุม Equipment
     → DCS Batch Manager สั่งการ

Gateway Options:
  A. Hardwired I/O: ง่าย, แต่ I/O Count สูง (ราคาแพง)
  B. Modbus TCP Gateway: ราคาปานกลาง, Scalable
  C. OPC UA: ยืดหยุ่นสูงสุด, มาตรฐาน
  D. EtherNet/IP: สำหรับ Allen-Bradley PLC กับ DCS ที่รองรับ
```

**OPC UA Gateway Configuration:**

```
OPC UA Gateway (เช่น Matrikon OPC เชื่อม PLC กับ DCS):

PLC (Siemens S7-1500) → OPC UA Server (Native in S7-1500)
                              ↓
                        OPC UA Network (Ethernet)
                              ↓
               DCS OPC UA Client (ABB/Yokogawa)
                              ↓
               DCS Tag Database (External Source Tag)

OPC UA Advantages:
  - Security: Certificate-based Authentication
  - Self-describing: Information Model
  - Pub/Sub or Client/Server
  - Platform Independent
  - IEC 62443 Security Compliant
  
Tag Mapping ตัวอย่าง:
  PLC Tag: "CompressorPackage.Speed_RPM"
  → OPC UA Node ID: ns=2;s=CompressorPackage.Speed_RPM
  → DCS External Tag: "CPS_SPEED_RPM"
  → DCS Display: Compressor Speed
```

---

## Step 1504: Marshalling Cabinets

### Concept ของ Marshalling Cabinet

```
Marshalling Cabinet ทำอะไร:
  Field Wiring ←→ Marshalling Cabinet ←→ DCS/PLC I/O
  
  เหตุผลที่มี Marshalling Cabinet:
  1. Field Cable ยาว เดินมาจาก Plant
     → Terminate ที่ Marshalling ก่อน
     
  2. DCS I/O Cabinet อยู่ห่างกัน
     → Marshalling อยู่กลาง, Distribution ง่ายกว่า
     
  3. Safety: ถ้า DCS เสีย → Marshalling ยังอยู่
     → ง่ายต่อการ Rewire

Traditional vs Fieldbus Marshalling:
  Traditional: 1 Wire = 1 Signal (มาก Wire)
  Fieldbus: 1 Bus = หลาย Device
  HART: 1 สาย = 1 Device แต่ได้หลาย Variable
  
Smart Marshalling (Universal I/O):
  Emerson Ovation Universal I/O
  Honeywell CHARMED
  ย้าย Channel ได้โดย Software (ไม่ต้อง Rewire)
```

---

## Step 1505: Migration from DCS to PLC

### การย้ายจาก Legacy DCS ไปยัง Modern PLC

```
Why Migrate DCS to PLC:
  - DCS หมดอายุ (15-20 ปี)
  - Spare Parts ไม่มีแล้ว
  - Vendor ไม่ Support แล้ว
  - ต้องการ Open System
  
Migration Strategies:
  A. Forklift (Big Bang): เปลี่ยนทั้งหมดพร้อมกัน
     ✓ Fast, Clean
     ✗ High Risk, Long Shutdown
     
  B. Phased Migration: เปลี่ยนทีละ Area
     ✓ Lower Risk
     ✗ ต้องรัน DCS และ PLC ควบคู่กัน
     
  C. Emulation: PLC จำลอง DCS Behavior
     ✓ Minimize Retraining
     ✗ Complex, Hidden Problems

Migration Checklist:
  □ Export all I/O Lists from DCS
  □ Export all Loop Diagrams
  □ Document all Control Strategies (including undocumented ones!)
  □ Check all Interlocks and Overrides
  □ Test all Alarms
  □ Operator Training on new HMI
  □ Parallel Run Period (3-6 months recommended)
```

---

## Step 1506-1520: System Integration Projects

### Step 1506: Complete Integration Architecture

**Typical Process Plant Integration:**

```
Enterprise Level (SAP, AspenTech):
    ↕ Web Services / OPC UA
Business Process Management Level:
  LIMS (Lab Info Mgmt)
  MES (Manufacturing Execution)
  APC (Advanced Process Control)
    ↕ OPC DA/UA, REST API
Process Control Level:
  DCS (Yokogawa) - Main Process
  Safety PLC (Triconex) - SIS/ESD
  PLC (AB) - Package Units
  PLC (S7) - Utilities
    ↕ HART, Fieldbus, 4-20mA
Field Level:
  Sensors, Analyzers, Valves, Pumps, Motors
```

### Step 1507: ISA-95 for Plant Integration

```
ISA-95 (IEC 62264) มาตรฐาน Plant Integration:
  Level 4: Business Systems (ERP)
  Level 3: Manufacturing Operations (MES)
  Level 2: Supervisory Control (DCS/SCADA)
  Level 1: Process Control (PLC/DCS)
  Level 0: Process Equipment

Data Flows:
  Upward (Production Data):
    Level 1 → Level 2: Measurements, Status, Alarms
    Level 2 → Level 3: Production Records, Quality Data
    Level 3 → Level 4: Finished Goods, Cost Data
    
  Downward (Instruction Data):
    Level 4 → Level 3: Orders, Schedules
    Level 3 → Level 2: Recipe, Batch Instructions
    Level 2 → Level 1: Setpoints, Mode Changes
```

### Step 1508: Batch Control (ISA-88)

**Batch Recipe Integration (DCS + MES):**

```pascal
// ISA-88 Recipe Execution in DCS
// Master Recipe → Control Recipe → Process

TYPE T_BatchRecipe :
STRUCT
    RecipeID       : STRING[20];
    ProductName    : STRING[50];
    BatchSize_kg   : REAL;
    
    // Procedure Steps (ISA-88 Hierarchy)
    // Procedure → Unit Procedure → Operation → Phase
    
    Phase1_Charge   : T_ChargePhase;    // เติมวัตถุดิบ
    Phase2_React    : T_ReactPhase;     // ทำปฏิกิริยา
    Phase3_Distill  : T_DistillPhase;   // กลั่น
    Phase4_Transfer : T_TransferPhase;  // ถ่ายเทผลผลิต
    Phase5_Clean    : T_CleanPhase;     // ล้างถัง
END_STRUCT
END_TYPE

// Batch Execute in DCS Function Block
FUNCTION_BLOCK FB_BatchExecute
VAR_INPUT
    Recipe      : T_BatchRecipe;
    Start       : BOOL;
    Abort       : BOOL;
    Hold        : BOOL;
END_VAR
VAR_OUTPUT
    State       : INT;   // 0=Idle, 1=Running, 2=Hold, 3=Aborted, 4=Complete
    Progress    : REAL;  // % สำเร็จ
    Phase_Name  : STRING[20];
END_VAR
// ดำเนินการ Phase ตามลำดับ
// แต่ละ Phase เรียก Sub-Function Block ของตัวเอง
END_FUNCTION_BLOCK
```

---

## Lab Project 76: DCS-PLC Integration Project

### โจทย์: ออกแบบระบบ Integration ระหว่าง Yokogawa DCS กับ Siemens PLC

**สถานการณ์:**
โรงงาน Chemical Plant มี:
- Yokogawa CENTUM VP ควบคุม Main Reactor + Distillation
- Siemens S7-1515 ควบคุม Utilities (Cooling Tower, Boiler, Compressed Air)

**ต้องการ Integration:**
1. DCS เห็นสถานะ Utility Equipment บน SCADA
2. DCS สั่ง Start/Stop Utility Pumps ได้
3. Energy Data (kWh) จาก PLC → DCS Historian
4. Process Data (Temp, Pressure) จาก DCS → MES

**วิธีการ:**
Protocol: OPC UA
  - Siemens S7-1500 built-in OPC UA Server
  - Yokogawa CENTUM VP OPC UA Client
  
**งานที่ต้องทำ:**
1. กำหนด OPC UA Information Model (Node IDs)
2. Security Configuration (Certificates)
3. Tag Mapping Table (DCS ↔ PLC)
4. Alarm Forwarding Rules
5. Testing และ Commissioning Plan

**Tag Mapping Example:**

| PLC Tag | PLC Address | OPC UA Node | DCS External Tag |
|---------|-------------|-------------|------------------|
| CT_FAN1_STATUS | Q0.0 | ns=2;s=CT.Fan1.Status | CT_FAN1_RUN |
| CT_FAN1_SPEED | MD100 | ns=2;s=CT.Fan1.Speed | CT_FAN1_SPD |
| BOILER_STEAM_P | IW256 | ns=2;s=BLR.Steam.Press | BLR_STM_P |
| CT_ENERGY_KWH | MD200 | ns=2;s=CT.Energy.kWh | CT_ENGY_KWH |

---

## สรุป Part 76

ใน Part นี้เราได้เรียนรู้:
- DCS vs PLC Comparison ในเชิงลึก
- DCS Architecture (Yokogawa, Honeywell)
- HART Protocol Integration
- DCS-PLC Gateway (OPC UA, Modbus)
- Marshalling Cabinet Design
- DCS to PLC Migration Strategies
- ISA-95 Plant Integration Model
- ISA-88 Batch Control

**ทักษะสำคัญ:**
- System Architecture Design
- Protocol Selection
- Data Mapping & Integration
- OPC UA Security Configuration

---

### 🔗 Navigation
- [← Part 75: Advanced HMI Design](part75_advanced_hmi.md)
- [→ Part 77: Advanced Safety System](part77_advanced_safety.md)
- [📚 สารบัญหลัก](../README.md)
