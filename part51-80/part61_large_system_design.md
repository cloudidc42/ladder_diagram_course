# Part 61: Large Industrial System Design (Steps 1201-1220)

## บทนำ: ความซับซ้อนของระบบขนาดใหญ่

ระบบ PLC ขนาดใหญ่ (100+ I/O, หลาย CPU, หลาย network):
- โรงงานรถยนต์: Body Shop, Paint Shop, Assembly
- โรงกลั่นน้ำมัน: 50,000+ I/O points
- Power plant: Multiple redundant PLCs
- Water treatment: Distributed control over wide area

ความท้าทาย:
- CPU load ต้องไม่เกิน 70%
- Network bandwidth ต้องเพียงพอ
- Panel design ต้องปลอดภัย (IEC 61439)
- Project management: หลาย engineer ทำพร้อมกัน

---

## Step 1201: System Architecture for Large Plants

### การออกแบบ Architecture

```
LARGE PLANT CONTROL ARCHITECTURE:
──────────────────────────────────────────────────────────────

Example: Automotive Body Shop (500+ I/O per zone, 20 zones)

Level 4: Enterprise (ERP, MES scheduling)
          │ OPC UA / REST API
Level 3: Supervisory (SCADA, Historian, MES execution)
          │ OPC UA / Modbus TCP
Level 2: HMI / Engineering Stations
          │ PROFINET / EtherNet/IP
Level 1: Controllers (PLCs, Safety PLCs)
          │ PROFINET RT / EtherNet/IP
Level 0: Field (Sensors, Actuators, Robots, Drives)

CONTROLLER PARTITIONING (Body Shop):
──────────────────────────────────────────────────────────────

Zone 1: Underbody Assembly
├── CPU: S7-1517-3 PN/DP
├── I/O: 512 DI, 256 DO, 32 AI
├── Safety: F-CPU integrated
└── Devices: 8 robots, 20 welders, 15 conveyors

Zone 2: Side Frame
├── CPU: S7-1515-2 PN (× 2, redundant)
├── I/O: 384 DI, 256 DO, 16 AI
└── ...

Zone 3: Framing Station (most complex)
├── CPU: S7-1518-4 PN/DP (top of range)
├── I/O: 1024 DI, 512 DO, 64 AI
├── PROFINET: 3 rings (48 devices)
└── Safety: F-CPU S7-1516F

Communication:
Zone 1 ↔ Zone 2 ↔ Zone 3 → Plant Controller (coordination)
Plant Controller → SCADA/MES (production tracking)
```

---

## Step 1202: CPU Load Analysis

### วิเคราะห์และจัดการ CPU Load

```
CPU LOAD ANALYSIS:
──────────────────────────────────────────────────────────────

Siemens S7-1500 CPU Load:
สูตร: Cycle Time = Σ(OB1 + Interrupt OBs + System tasks)

Target: Cycle time < 70% of watchdog time
(ถ้า watchdog = 150ms → cycle time < 105ms)
(แนะนำ: < 50ms สำหรับ smooth operation)

ปัจจัยที่ส่งผลต่อ Cycle Time:
├── จำนวน networks ใน OB1
├── จำนวน FB instances (PROFINET read/write cycles)
├── PROFINET update rate (1ms vs 10ms)
├── Interrupt OBs (OB40, OB82, OB83)
└── Communication (OPC UA server)

// ดู Cycle Time ใน TIA Portal:
// Online → PLC → Diagnostics → Cycle Time
// หรือใน Program: SFB51 (READ_CLK)

// วิธีลด Cycle Time:
// 1. แยก FB ไปยัง Cyclic OB (OB30, OB31, OB32)
//    - เช่น Energy monitoring → OB32 (1s interval)
//    - Alarm history → OB31 (100ms interval)
//    - สงวน OB1 สำหรับ time-critical control
//
// 2. ลด PROFINET update rate สำหรับ non-critical devices
//    - Safety I/O: 1ms (ไม่เปลี่ยน)
//    - Standard I/O: 4ms → 8ms (ถ้าเป็นไปได้)
//    - VFDs, non-critical: 16ms → 32ms
//
// 3. ใช้ FB ที่ efficient
//    - หลีกเลี่ยง STRING operations ใน OB1 (slow)
//    - หลีกเลี่ยง REAL division ที่ไม่จำเป็น

// LOAD ESTIMATION TABLE:
// Operation                  | Approx Time (μs)
// 1 FB call (empty)          | 2-5
// 1 ADD/COMPARE (INT)        | 0.1-0.5
// 1 ADD/COMPARE (REAL)       | 1-3
// 1 STRING operation         | 10-100
// 1 PROFINET device poll     | 50-200
// 1 OPC UA tag read          | 100-500
// 1 Modbus TCP transaction    | 5,000-50,000
```

---

## Step 1203: Memory Planning

### การวางแผน Memory

```
MEMORY PLANNING สำหรับ S7-1500:
──────────────────────────────────────────────────────────────

S7-1500 Memory Types:
├── Load Memory: code + initial values (SD card)
├── Work Memory: executing code + instances (RAM)
└── RETAIN Memory: survive power cycle (battery-backed)

Memory Estimations:
Load Memory:
  OB/FB/FC code:    ~5-50 KB per block
  DB data:          variable × count × instances
  Total typical:    5-100 MB (SD card 2GB → ไม่เป็นปัญหา)

Work Memory (more important - limited):
  S7-1515-2 PN: 500 KB work memory
  S7-1516-3 PN: 1 MB work memory
  S7-1518-4 PN: 4 MB work memory

// WORK MEMORY ESTIMATION ตัวอย่าง:
// Large system (500 FB instances):
// 500 × 200 bytes avg = 100,000 bytes = ~100 KB
// Large DB (10,000 tags × 4 bytes) = 40 KB
// Process image (2000 I/O × 1 byte) = 2 KB
// Communication buffers = ~20 KB
// TOTAL: ~162 KB ← ดีมาก, S7-1515 ใช้ได้

RETAIN MEMORY:
// จำกัดมาก! 256 KB สูงสุด (S7-1515)
// ใช้สำหรับ:
// - Production counters (RETAIN INT)
// - Setpoints ที่คนตั้งค่า (RETAIN REAL)
// - Batch IDs (RETAIN STRING)
//
// ห้ามใช้กับ:
// - Temporary variables (ใช้ RAM แทน)
// - Historical data (ใช้ Historian แทน)
// - Large arrays (จะใช้ RETAIN memory หมด!)
```

---

## Step 1204: Network Bandwidth Analysis

### วิเคราะห์ Bandwidth

```
NETWORK BANDWIDTH ANALYSIS:
──────────────────────────────────────────────────────────────

PROFINET Bandwidth Calculation:
ทุก device ส่ง/รับ data ทุก update cycle

Example: Zone 1 (48 PROFINET devices)
Device Type                  | Count | Frame Size | Update | Bandwidth
Servo Drive (SINAMICS S120)  | 16    | 64 bytes   | 1ms    | 16×64×2×1000 = 2MB/s
Standard I/O Module (ET200)  | 24    | 24 bytes   | 4ms    | 24×24×2×250  = 288KB/s
Safety F-CPU Module          | 8     | 32 bytes   | 2ms    | 8×32×2×500   = 256KB/s

Total: ~2.5 MB/s

PROFINET 100Mbps = 12.5 MB/s theoretical
กฎ: ใช้ไม่เกิน 50% = 6.25 MB/s
2.5 MB/s < 6.25 MB/s → OK!

// ถ้า bandwidth เกิน → แก้ไข:
// 1. เพิ่ม switch (แยก devices เป็น 2 subnets)
// 2. เพิ่ม update time ของ non-critical devices
// 3. ใช้ PROFINET switch ที่รองรับ VLAN

EtherNet/IP Bandwidth (Allen-Bradley):
RPI (Requested Packet Interval) = ความถี่ request
ControlLogix max: 10,000 connections
แต่ละ connection ใช้ bandwidth = data_size / RPI

// ตัวอย่าง:
// 100 drives × 20 bytes × 1/10ms = 200 KB/s
// 200 I/O modules × 40 bytes × 1/20ms = 400 KB/s
// Total: 600 KB/s << 100Mbps → OK
```

---

## Step 1205: I/O Philosophy Document

### กำหนด Standard สำหรับ I/O

```
I/O PHILOSOPHY DOCUMENT:
──────────────────────────────────────────────────────────────

I/O Philosophy = Standard ที่กำหนดว่า I/O ต้องเป็นอย่างไร
ทุกโปรเจกต์ต้องมีเอกสารนี้ก่อนเริ่มออกแบบ

1. SIGNAL STANDARDS:
   Digital Input (DI):
   ├── Voltage: 24VDC (EN 61131-2 Type 3)
   ├── Logic: Positive logic (24V = TRUE) ยกเว้น safety
   ├── Safety: Negative logic (0V = safe state) ← IMPORTANT
   ├── Debounce filter: 3ms (standard), 0.5ms (high-speed)
   └── Short circuit protection: Required on all modules

   Digital Output (DO):
   ├── 24VDC source type (P-switching)
   ├── Max current: 0.5A per channel (standard)
   ├── Fusing: 1A per group of 8 outputs
   └── Diagnostic: Short circuit detection

   Analog Input (AI):
   ├── 4-20mA signals: Standard (wire break detectable)
   ├── 0-10V: Only where 4-20mA not available
   ├── PT100/PT1000: For temperature
   ├── TC type K: For high temperature (>300°C)
   └── Resolution: 16-bit minimum

   Analog Output (AO):
   ├── 4-20mA: Standard
   ├── Load: max 600 Ω
   └── Resolution: 12-bit minimum

2. NAMING CONVENTION:
   Format: <AREA>_<UNIT>_<DESCRIPTION>_<IO_TYPE><NUMBER>
   
   DI: P01_T301_LEVEL_LS001 (Plant01, Tank301, Level, LevelSwitch001)
   DO: P01_T301_INLET_VLV001 (Plant01, Tank301, Inlet, Valve001)
   AI: P01_T301_TEMP_TT001   (Plant01, Tank301, Temperature, Transmitter001)
   AO: P01_T301_FLOW_FV001   (Plant01, Tank301, Flow, FlowValve001)

3. SPARE CAPACITY:
   ├── I/O modules: 20% spare channels
   ├── I/O racks: 20% spare slots
   └── CPU: 20% spare memory
```

---

## Step 1206: Panel Design IEC 61439

### ออกแบบตู้ Panel ตามมาตรฐาน

```
PANEL DESIGN STANDARDS (IEC 61439):
──────────────────────────────────────────────────────────────

IEC 61439-1: Low Voltage Switchgear and Controlgear Assemblies
General requirements สำหรับ electrical panels

KEY REQUIREMENTS:
1. TEMPERATURE RISE:
   Panel internal temp ≤ 40°C above ambient
   ต้องคำนวณ: Power loss in panel → Heat dissipation needed
   
   Power Dissipation Calculation:
   Component              | Power Loss
   MCB (Circuit Breaker)  | 5-10W per unit
   Contactor              | 5-15W (when energized)
   PLC CPU                | 10-25W
   I/O Module (8pt DI)    | 1-3W
   24VDC PSU (10A, 90%)   | 27W (loss at full load)
   Servo Drive 2.2kW      | 110W (5% loss)
   
   Total loss → Ventilation fan/Heat exchanger sizing

2. BUSBAR SIZING:
   Current Density: Copper busbar ≤ 2 A/mm²
   40A continuous → minimum 20mm² Cu

3. SHORT CIRCUIT WITHSTAND:
   Panel rated for: Ik (short circuit current at point)
   MCB/fuses must clear fault before busbar damaged

4. IP RATING:
   ├── Clean room: IP54 (dust + splash)
   ├── Outdoor: IP65 (dust + water jets)
   ├── Washdown area: IP66 (powerful water jets)
   └── Submersion risk: IP67/68

5. CLEARANCE AND CREEPAGE:
   240VAC: Min clearance 5.5mm, creepage 8mm (pollution deg 2)
   480VAC: Min clearance 6mm, creepage 10mm

6. LABELING:
   ├── All terminal blocks: labeled
   ├── All cables: both ends labeled
   ├── All devices: reference designator
   └── Warning signs: ⚡ Danger, voltage levels

PANEL VENTILATION CALCULATION:
P_dissipation = 500W (all heat sources)
Ambient = 35°C, Max internal = 45°C
ΔT = 10°C
Fan airflow required: Q = P / (ρ × Cp × ΔT)
  = 500 / (1.2 × 1005 × 10) = 0.041 m³/s = 148 m³/hr
→ เลือก fan: 200 m³/hr (add 30% margin)
```

---

## Step 1207: Control System Design Methodology

### ขั้นตอนการออกแบบระบบควบคุม

```
CONTROL SYSTEM DESIGN METHODOLOGY (V-Model):
──────────────────────────────────────────────────────────────

Phase 1: CONCEPT DESIGN
├── Plant process description
├── Control philosophy (batch, continuous, semi-auto)
├── Safety philosophy (SIL requirements)
├── Utility requirements (power, air, network)
└── Rough equipment list

Phase 2: FEED (Front End Engineering Design)
├── User Requirements Specification (URS)
├── Cause and Effect Matrix (C&E)
├── Instrument Index
├── Control System Architecture
├── Equipment List (PLCs, HMIs, instruments)
└── Estimate: cost, schedule

Phase 3: DETAILED ENGINEERING
├── Functional Specification (FS)
├── Hardware Design:
│   ├── Panel drawings (SolidWorks Electrical)
│   ├── I/O List (all points)
│   ├── Cable schedules
│   └── Network topology
└── Software Design:
    ├── Software Design Spec (SDS)
    ├── State machine diagrams
    ├── DB structures
    ├── HMI screen designs
    └── Alarm list

Phase 4: PROCUREMENT
├── PLC hardware order (12-16 weeks lead time!)
├── Cable and components
└── Instruments and sensors

Phase 5: FABRICATION
├── Panel build
├── Software development
└── Factory testing (internal)

Phase 6: FAT (Factory Acceptance Test)

Phase 7: INSTALLATION AND COMMISSIONING

Phase 8: SAT (Site Acceptance Test)

Phase 9: HANDOVER
├── As-built documentation
├── Training
├── Spare parts list
└── Maintenance manual
```

---

## Step 1208: FEED Engineering

### Front End Engineering Design

```
FEED DELIVERABLES (สิ่งที่ต้องออก):
──────────────────────────────────────────────────────────────

1. CONTROL PHILOSOPHY DOCUMENT:
   กำหนด: 
   - Manual / Semi-auto / Fully-auto modes
   - Start-up and shutdown procedures
   - Emergency response
   - Maintenance modes

2. CAUSE AND EFFECT MATRIX:
   Cause (input event) → Effect (output action)
   
   ตัวอย่าง:
   Cause                  | Affect
   TT-301 High High (110°C)| Close V-201, Open V-202, Stop P-101
   LS-302 Low Low (10%)    | Stop P-101, Alarm, Close V-201
   E-Stop activated        | Stop ALL motors, Close ALL valves

3. INSTRUMENT INDEX:
   TAG     | Description           | Type    | Range    | Location
   TT-301  | Tank T301 Temperature | PT100   | 0-150°C  | Tank T301
   LS-302  | Tank T302 Level       | Ultrasonic | 0-100% | Tank T302
   FT-303  | Feed Flow             | Coriolis | 0-50 kg/h | Line L1

4. SYSTEM ARCHITECTURE DRAWING:
   Network topology, PLC locations, HMI locations
   (Visio, AutoCAD, or E3.series)

5. PRELIMINARY I/O COUNT:
   Zone | DI  | DO  | AI  | AO  | Safety DI | Safety DO
   Z1   | 320 | 256 | 64  | 32  | 48        | 24
   Z2   | 256 | 192 | 48  | 16  | 32        | 16
   ...

6. PRELIMINARY COST ESTIMATE (±30% accuracy):
   Hardware: PLC, I/O, instruments
   Software: Engineering, programming
   Installation: Panel build, cables, commissioning
   Validation: FAT, SAT, documentation
```

---

## Step 1209: Vendor Evaluation

### การประเมินและเลือก Vendor

```
VENDOR EVALUATION สำหรับ PLC/DCS:
──────────────────────────────────────────────────────────────

TECHNICAL CRITERIA (60%):
├── CPU performance: scan time, memory
├── I/O capacity: module types, density
├── Communication: PROFINET, EtherNet/IP, Modbus
├── Safety: Integrated F-CPU หรือ separate Safety PLC
├── Redundancy: Hot standby options
├── Engineering: TIA Portal / Studio 5000 / GX Works ease of use
└── Diagnostics: Built-in + OPC UA export

COMMERCIAL CRITERIA (25%):
├── ราคาต่อ I/O point
├── License model (Engineering software)
├── Spare parts availability (10+ year horizon)
├── Local support and training
└── Integration cost สำหรับ existing systems

VENDOR SUPPORT (15%):
├── Local service engineers
├── Remote support (TeamViewer, OPC)
├── Response time SLA
└── Reference sites (visit existing installations)

EVALUATION MATRIX:
Criteria          | Weight | Siemens | AB    | Mitsubishi
------------------|--------|---------|-------|----------
CPU Performance   | 15%    | 9.0     | 8.5   | 7.5
I/O Variety       | 10%    | 9.5     | 8.5   | 8.0
Safety Integration| 15%    | 9.5     | 8.0   | 7.0
Comm Protocols    | 10%    | 9.0     | 8.5   | 8.0
Local Support     | 15%    | 8.5     | 7.0   | 9.0
Price (inverted)  | 15%    | 7.0     | 6.5   | 9.5
Engineering SW    | 10%    | 8.5     | 8.0   | 8.5
Spare Parts       | 10%    | 9.0     | 8.5   | 8.0
------------------|--------|---------|-------|----------
TOTAL             | 100%   | 8.7     | 7.9   | 8.3

→ เลือก Siemens (8.7 highest score)
```

---

## Step 1210: Project Execution Plan

### วางแผนการดำเนินโปรเจกต์

```
PROJECT EXECUTION PLAN:
──────────────────────────────────────────────────────────────

PROJECT: Automotive Body Shop Automation
Scale: 3 zones, 15,000 I/O, 25 PLCs
Duration: 24 months

PHASE SCHEDULE:
Phase 1: FEED                    3 months  (M1-M3)
Phase 2: Detailed Engineering    5 months  (M3-M8)
Phase 3: Hardware Procurement    4 months  (M4-M8) [parallel]
Phase 4: Panel Fabrication       4 months  (M8-M12)
Phase 5: Software Development    6 months  (M6-M12) [parallel]
Phase 6: FAT                     2 months  (M12-M14)
Phase 7: Installation            3 months  (M14-M17)
Phase 8: Commissioning           3 months  (M17-M20)
Phase 9: SAT + Handover          2 months  (M20-M22)
Phase 10: Support period         2 months  (M22-M24)

TEAM STRUCTURE:
Project Manager (1)
├── Lead Electrical Engineer (1)
│   ├── Electrical Engineers (3)
│   └── Designers (2)
├── Lead PLC Engineer (1)
│   ├── PLC Engineers (5) [zone leads]
│   ├── Safety Engineer (1)
│   └── Junior Engineers (4)
├── Network Engineer (1)
├── HMI/SCADA Engineer (2)
└── Commissioning Team (5) [starts M17]

CRITICAL PATH:
FEED → FS → Software Development → FAT → SAT → Handover
       ↓
       Hardware Procurement → Panel Build → (parallel with SW dev)

RISK REGISTER:
Risk                           | Likelihood | Impact | Mitigation
Long lead time hardware (20wk) | High       | High   | Order early M4
Customer scope changes          | Medium     | High   | Freeze scope M5
PROFINET commissioning issues  | Medium     | Medium | PROFINET expert + ProfiTrace tool
FAT failure → delay             | Low        | High   | Internal pre-FAT + simulation
```

---

## Step 1211: Software Architecture for Large Systems

### Software Design สำหรับระบบใหญ่

```
SOFTWARE ARCHITECTURE LAYERS:
──────────────────────────────────────────────────────────────

Layer 5: APPLICATION (ระดับ Process)
  OB1 → calls Zone FBs → coordinates all equipment

Layer 4: SEQUENCE (ระดับ Machine/Zone)
  FB_ZoneController → State Machine สำหรับ zone operation

Layer 3: EQUIPMENT (ระดับ Equipment)
  FB_Motor, FB_Valve, FB_Conveyor → ควบคุม 1 equipment

Layer 2: DEVICE (ระดับ Physical I/O)
  FB_AnalogPipeline, FB_Digital_Filter → process raw I/O

Layer 1: FIELD (Physical sensors/actuators)

CALL HIERARCHY ใน OB1:
OB1
├── FB_GlobalEnable.Execute()        // Singleton
├── FB_Plant_Controller.Execute()   // Application layer
│   ├── FB_Zone1.Execute()          // Zone sequencing
│   │   ├── FB_Conveyor_Z1.Execute()// Equipment control
│   │   ├── FB_Motor_AG1.Execute()  // Motor control
│   │   └── FB_Valve_V101.Execute() // Valve control
│   ├── FB_Zone2.Execute()
│   └── FB_Zone3.Execute()
└── FB_Communication.Execute()      // OPC UA, Modbus server

GLOBAL DATA STRATEGY:
// ใช้ Interface DBs เพื่อ pass data ระหว่าง zones
DATA_BLOCK DB_Z1_to_Z2_Interface
    // Data Zone 1 ส่งไป Zone 2
    bProductReady:      BOOL;       // Zone 1 product ready
    iProductType:       INT;        // Which product
    iProductID:         DINT;       // Unique ID
END_DATA_BLOCK

// Zone 1 เขียน, Zone 2 อ่าน
// ไม่มี direct call ระหว่าง zone FBs
// ป้องกัน coupling สูง
```

---

## Step 1212: I/O List Management

### จัดการ I/O List ขนาดใหญ่

```
I/O LIST MANAGEMENT:
──────────────────────────────────────────────────────────────

สำหรับ 15,000 I/O points: ต้องใช้ database หรือ Excel

I/O List Fields (minimum):
Col | Field          | Example
----|----------------|------------------------------------------
A   | IO_ID          | IO-Z1-DI-0001
B   | Tag_Name       | P01_CV001_RUN_XS
C   | Description    | Conveyor Zone 1 Running Status
D   | IO_Type        | DI (DI/DO/AI/AO/FDI/FDO)
E   | PLC            | PLC-Z1
F   | Module         | DI-Module-Slot3
G   | Channel        | CH4
H   | HW_Address     | %I3.4 (Siemens)
I   | Tag_Address    | DB_IO.DI_Z1.bits.4
J   | Panel          | PNL-Z1-01
K   | Terminal       | X1:24
L   | Cable          | CAB-001-DI-001
M   | Instrument_Tag | XS-Z1-001
N   | P&ID_Drawing   | P-001-Rev3
O   | Signal_Level   | 24VDC
P   | Sensor_Type    | Proximity (NPN NO)
Q   | Alarm_Point    | None
R   | Interlock      | IL-001 (ดู Interlock Matrix)
S   | Comments       | Speed > 0 indicates conveyor running
T   | Status         | CONFIRMED / PRELIMINARY / TBC
U   | Rev            | R1 (2024-01-15)

Total: 15,000 rows × 21 columns = 315,000 cells
→ Excel ได้ แต่ควรใช้ database (Access, SQL)

// Auto-generate TIA Portal Tags จาก I/O List:
// Python script อ่าน Excel → generate .xlsx import file
// Import ไป TIA Portal tag table
```

---

## Step 1213: Communication Architecture

### Architecture ของ Communication

```
COMMUNICATION ARCHITECTURE สำหรับ LARGE SYSTEM:
──────────────────────────────────────────────────────────────

OPC UA SERVER HIERARCHY:
Plant Controller → OPC UA Server (expose all tags)
  SCADA Client ← subscribe → OPC UA Server
  MES Client   ← subscribe → OPC UA Server
  Historian    ← subscribe → OPC UA Server

OPC UA Node Structure:
Root/
├── PlantA/
│   ├── Zone1/
│   │   ├── Equipment/
│   │   │   ├── Conveyor_Z1/
│   │   │   │   ├── Speed (r, rw)
│   │   │   │   ├── Running (r)
│   │   │   │   └── Faults/ (object)
│   │   │   └── Motor_AG1/ ...
│   │   └── Production/
│   │       ├── CycleCount (r)
│   │       └── Throughput (r)
│   └── Zone2/...
└── PlantB/...

Siemens S7-1500 OPC UA Configuration:
// TIA Portal: Properties → OPC UA → Server
// Enable: OPC UA Server
// Port: 4840
// Security: Basic256Sha256 (sign + encrypt)
// Authentication: Username/Password
// Max sessions: 10
// Publish interval: 100ms

// Export Tag List as OPC UA nodes:
// Tag Properties → OPC UA → Accessible from HMI/OPC UA: Yes
// หรือ Global DB: set as "Accessible from HMI/OPC UA"
```

---

## Step 1214: Redundancy Design

### ออกแบบ Redundancy

```
REDUNDANCY DESIGN:
──────────────────────────────────────────────────────────────

REDUNDANCY LEVELS:
Level 1: Power Supply Redundancy
  2 × 24VDC PSU → redundancy module → ทำงาน parallel
  ถ้า PSU 1 fail → PSU 2 take over ทันที (hot standby)

Level 2: I/O Module Redundancy
  PROFINET S2 Redundancy:
  - 2 PLCs connect ไปยัง same I/O station
  - ถ้า PLC 1 offline → PLC 2 takes over I/O ทันที
  - Applicable to: ET 200SP with R1/S2 CPU

Level 3: CPU Redundancy
  Siemens S7-400H / S7-1500R:
  - 2 CPUs run identical programs in parallel
  - Sync via fiber optic link (< 1ms sync)
  - ถ้า CPU 1 fail → CPU 2 takes over (< 300ms)
  - Suitable for: continuous process, no stop allowed

Level 4: Network Redundancy
  MRP Ring: Recovery < 200ms (standard)
  PROFINET IRT MRP: < 30ms
  Spanning Tree Protocol (STP): 30-50 seconds (too slow!)

Level 5: Site Redundancy
  Primary site + Backup site
  Real-time data sync
  Manual failover (human decision)

REDUNDANCY DECISION MATRIX:
Process                    | SIL | Redundancy Level
Safety shutdown (ESD)      | SIL2| CPU Redundancy + Safety PLC
Continuous process (refinery)| SIL1| CPU + Network Redundancy
Batch process (pharma)     | None| I/O + Network Redundancy
Filling line (food)        | None| Network Redundancy (MRP)
Conveyor (automotive)      | None| Basic (no redundancy needed)
```

---

## Step 1215: Cybersecurity in System Design

### รวม Security ตั้งแต่ Design

```
SECURITY BY DESIGN:
──────────────────────────────────────────────────────────────

Security ต้องเป็น part of design ตั้งแต่เริ่ม (ไม่ใช่ add-on)

IEC 62443-3-3 Security Levels สำหรับ Control System:
SL 1: Protection against casual/unintentional violation
SL 2: Protection against intentional attack (simple means)
SL 3: Protection against sophisticated attacks
SL 4: Nation-state level attacks

สำหรับ Industrial plant ส่วนใหญ่: SL 2 พอเพียง

SECURITY ARCHITECTURE CHECKLIST:
NETWORK SEGMENTATION:
☐ OT Network แยกจาก IT Network (DMZ or air gap)
☐ Safety network ไม่ต่อ internet
☐ VLAN segmentation ตาม zone
☐ Firewall ระหว่างทุก zone

ACCESS CONTROL:
☐ Role-based access (Operator/Engineer/Admin)
☐ MFA สำหรับ remote access (VPN)
☐ Automatic session timeout
☐ USB ports ปิดทุก port ที่ไม่ใช้งาน

SOFTWARE:
☐ Firmware update policy (ใครอนุมัติ, บ่อยแค่ไหน)
☐ Antivirus สำหรับ Engineering Workstations
☐ Application whitelisting บน SCADA servers
☐ Backup และ restore tested

MONITORING:
☐ SIEM (Security Information Event Management)
☐ Anomaly detection บน OT network
☐ Intrusion detection (IDS)
☐ Log สำหรับทุก access

Security Design Document:
ทำ Threat Assessment ก่อนออกแบบ:
1. Identify assets (PLCs, SCADA, Historian)
2. Identify threats (insider, ransomware, nation-state)
3. Identify vulnerabilities (unpatched, open ports)
4. Assess risk (likelihood × impact)
5. Mitigate (controls, monitoring)
6. Document and review annually
```

---

## Step 1216: Large-Scale HMI Design

### ออกแบบ HMI สำหรับระบบใหญ่

```
HMI DESIGN สำหรับ LARGE SYSTEM:
──────────────────────────────────────────────────────────────

ISA-101 HMI Standards:
├── Level 1: Overview (plant level - 1 screen)
├── Level 2: Group (zone/unit - 1 screen per zone)
├── Level 3: Detail (single equipment)
└── Level 4: Diagnostic (trend, history)

VISUAL DESIGN:
├── Background: medium gray (#707070) ← ลดตา fatigue
├── Normal state: ไม่สีสัน (gray tones)
├── Alarm: ใช้สีเฉพาะ Emergency + ต้องการ attention
│   - Critical alarm: flashing red + sound
│   - High alarm: red (solid)
│   - Low alarm: yellow
│   - Return to normal: green (brief)
└── Context: สีเพื่อ context เท่านั้น (ไม่ใช่ทุกวาล์ว)

ALARM MANAGEMENT (ISA-18.2):
├── Max alarm rate: < 10 alarms/10 min (steady state)
├── No alarm flood (>10 alarms/min) ← operator overwhelmed
├── Prioritize: only 5% Critical, 15% High, 80% Low
├── Eliminate nuisance alarms ก่อน go-live
└── Alarm rationalization workshop ก่อน commissioning

PERFORMANCE:
├── Screen load: < 2 seconds
├── Tag update: < 1 second
└── Historical trend: load 1 hour in < 5 seconds

// Alarm Rationalization Template:
// For each alarm, answer:
// 1. Is this alarm needed? (remove if not)
// 2. What action does operator take?
// 3. What is consequence of not acting?
// 4. What is time to act?
// 5. Is alarm correctly set (right threshold)?
// 6. Can alarm be automated? (interlock instead)
```

---

## Step 1217: Testing Large Systems

### Testing Strategy สำหรับระบบใหญ่

```
TESTING STRATEGY สำหรับ LARGE SYSTEMS:
──────────────────────────────────────────────────────────────

ปัญหา: ไม่สามารถ test ทุก combination ของ 15,000 I/O
วิธีแก้: Risk-based testing

RISK-BASED TEST PRIORITIZATION:
High Priority (test 100%):
├── Safety functions (ALL safety interlocks)
├── Emergency stops (ALL zones)
├── High-consequence faults (fire, flood, injury risk)
└── Production-critical paths

Medium Priority (test 80%):
├── Normal production sequences
├── Fault recovery procedures
├── HMI critical functions
└── Communication interfaces

Low Priority (test 50%):
├── Cosmetic HMI items
├── Non-critical reporting
└── Nice-to-have features

TEST EXECUTION PLAN (2-day FAT):
Day 1 (8 hours):
├── Hour 1-2: Hardware check (panel wiring, I/O check)
├── Hour 2-3: Safety functions (ALL E-stops, interlocks)
├── Hour 3-5: Zone 1 full functional test
└── Hour 5-8: Zone 2 full functional test

Day 2 (8 hours):
├── Hour 1-3: Zone 3 full functional test
├── Hour 3-5: Integration test (Zone 1↔2↔3 interaction)
├── Hour 5-6: Communication test (OPC UA, historian)
└── Hour 6-8: Performance test + debrief

PUNCH LIST:
// บันทึก items ที่ไม่ผ่านแต่ไม่ critical:
// Punch List Item - PAT (Pre-Acceptance Test)
// A-items: Critical → ต้องแก้ก่อน FAT close
// B-items: Non-critical → แก้ก่อน SAT
// C-items: Cosmetic → แก้ใน next release
```

---

## Step 1218: Commissioning Planning

### วางแผน Commissioning

```
COMMISSIONING PLAN สำหรับ LARGE SYSTEM:
──────────────────────────────────────────────────────────────

Commissioning Phases:
Phase 1: Pre-commissioning (Panel checks)
├── Visual inspection (all terminations)
├── Continuity test (all cables)
├── Insulation test (all power circuits)
├── Power-on: check voltages before PLC connects
└── I/O loop check (one by one)

Phase 2: Cold Commissioning (no process, no product)
├── PLC program load
├── All I/O verify (force output, check field)
├── Network verify (all devices communicating)
├── Safety system test (with maintenance permits)
└── HMI navigation and display check

Phase 3: Warm Commissioning (process active, no product)
├── PLC at safe mode (auto → manual first)
├── Test all sequences in manual
├── Test interlocks with real process
├── Alarm test (simulate each alarm)
└── Operator training

Phase 4: Hot Commissioning (with product)
├── First run at 20% speed/capacity
├── Gradually increase to 100%
├── Monitor all parameters
└── Fine-tune control loops (PID tuning)

COMMISSIONING LOG:
ทุกกิจกรรมต้องบันทึก:
Date | Time | Item | Description | Result | By | Remarks
-----|------|------|-------------|--------|----|---------
2024-04-01 | 08:00 | I/O | Loop check P01-DI-001 | PASS | A. Smith |
2024-04-01 | 09:30 | I/O | Loop check P01-DI-002 | FAIL | A. Smith | Wire reversed, corrected

ไม่มี pencil อย่างเดียว! ทุกอย่างต้อง written record
```

---

## Step 1219: Handover Documentation

### เอกสารการส่งมอบระบบ

```
HANDOVER DOCUMENTATION PACKAGE:
──────────────────────────────────────────────────────────────

เมื่อโปรเจกต์ complete → ส่งมอบเอกสารครบถ้วน

1. AS-BUILT DRAWINGS:
   ├── Panel electrical drawings (ตาม actual wiring)
   ├── Network topology (actual IP addresses)
   ├── I/O list (ครบถ้วน 15,000 points)
   └── P&ID (updated with all installed instruments)

2. SOFTWARE DOCUMENTATION:
   ├── PLC program (backup + source)
   ├── Software Design Specification (final)
   ├── Change log (all software changes)
   └── Git repository access

3. VALIDATION DOCUMENTATION (pharma/food):
   ├── IQ, OQ, PQ reports (signed)
   ├── FAT/SAT reports (signed)
   └── Validation Summary Report

4. OPERATION MANUAL:
   ├── Normal operation procedures
   ├── Startup/shutdown procedures
   ├── Emergency procedures
   └── HMI user manual

5. MAINTENANCE MANUAL:
   ├── Preventive maintenance schedule
   ├── Spare parts list (with part numbers)
   ├── Calibration procedures
   └── Troubleshooting guide

6. TRAINING:
   ├── Operator training (minimum 8 hours)
   ├── Maintenance training (minimum 8 hours)
   └── Training records (signed attendance)

7. WARRANTIES AND SUPPORT:
   ├── Hardware warranty certificates
   ├── Software license certificates
   ├── Support contract details
   └── Emergency contact list (24/7)

SIGN-OFF:
System ถือว่า handed over เมื่อ:
- ลูกค้าเซ็น FAT/SAT report
- ลูกค้าเซ็น Handover Certificate
- Training completed and signed
- All A-items closed
```

---

## Step 1220: Lab Project - System Architecture Design

### Lab: ออกแบบ Architecture สำหรับ Petrochemical Plant

```
LAB: CONTROL SYSTEM ARCHITECTURE FOR PETROCHEMICAL PLANT
══════════════════════════════════════════════════════════════

โปรเจกต์: Ethylene Storage and Loading Facility
Scale: Medium (1,500 I/O points)
Safety: SIL 2 (flammable material storage)

GIVEN INFORMATION:
- 4 storage tanks (50,000 ton each)
- 2 truck loading arms
- 1 ship loading arm
- Fire and gas detection: 80 detectors
- Utility systems: cooling water, nitrogen, steam

TASK 1: Architecture Design (30%)
─────────────────────────────────────────────────────────────
1. Design control system architecture:
   - Level 0-4 diagram
   - BPCS (Basic Process Control System) PLCs
   - SIS (Safety Instrumented System) PLCs (separate!)
   - Fire & Gas system
   - Network topology

2. I/O Count estimation:
   - Per system (BPCS vs SIS)
   - Per zone (tanks, loading)
   - Include 20% spare

3. PLC Selection:
   - BPCS: recommend model + justify
   - SIS: recommend F-CPU model + justify
   
4. Network Design:
   - PROFINET rings per zone
   - Firewalls between BPCS/SIS/IT

TASK 2: I/O Philosophy (20%)
─────────────────────────────────────────────────────────────
1. สร้าง I/O Philosophy Document สำหรับ project นี้:
   - DI/DO/AI/AO standards
   - Naming convention
   - Spare capacity requirements
   - Signal validation rules
   
2. สร้าง sample I/O list (minimum 50 entries):
   - Both BPCS and SIS signals
   - All required columns

TASK 3: Redundancy Design (20%)
─────────────────────────────────────────────────────────────
ตัดสินใจและ justify:
- CPU Redundancy: ที่ไหน? ทำไม?
- Network Redundancy: topology?
- Power Redundancy: ทุก panel?
- I/O Redundancy: เฉพาะ safety?

TASK 4: Project Plan (30%)
─────────────────────────────────────────────────────────────
สร้าง Project Execution Plan:
- Phase list และ duration
- Gantt chart (12-month project)
- Team structure (who does what)
- Risk register (5 risks minimum)
- Critical path identification

DELIVERABLES:
├── Architecture diagram (Visio หรือ draw.io)
├── I/O Philosophy Document
├── Sample I/O list (Excel)
├── Redundancy design document
├── Project Gantt chart
└── Presentation (10 minutes)

EVALUATION:
├── Architecture appropriateness: 25%
├── I/O Philosophy completeness: 20%
├── Redundancy justification: 20%
├── Project plan realism: 20%
└── Presentation clarity: 15%
```

---

## สรุป Part 61

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| CPU Load | ไม่เกิน 70%, แยก FB ไป Cyclic OBs |
| Memory | Work memory limited, plan RETAIN carefully |
| I/O Philosophy | กำหนด standard ก่อนเริ่ม design |
| Panel Design | IEC 61439, IP rating, temperature rise |
| Redundancy | CPU/Network/Power ตาม criticality |
| Project Plan | V-model, FEED → Detailed → FAT → SAT |

---

## Navigation

- [← Part 60: Testing and Validation](./part60_testing_validation.md)
- [→ Part 62: Advanced Motion - Gantry and Delta Robots](./part62_advanced_motion_gantry.md)
- [↑ สารบัญหลัก](../README.md)
