# Part 96: PLC Interview Questions - 100 ข้อกับคำตอบ
## Steps 1901-1920 | ระดับ Professional

---

## การนำทาง
- [← Part 95: Career Path International](part95_career_path_international.md)
- [→ Part 97: Advanced Troubleshooting](part97_advanced_troubleshooting.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1901: Junior Level Questions (Q1-Q30)

### Basic PLC Concepts

**Q1: PLC ย่อมาจากอะไร และทำงานอย่างไร?**
```
A: PLC = Programmable Logic Controller
   ทำงานแบบ Scan Cycle:
   1. Input Scan: อ่านสถานะ Input ทั้งหมดเก็บใน Input Image Register
   2. Program Execution: รันโปรแกรมจาก Memory โดยอ้างอิง Input Image
   3. Output Scan: เขียนค่าจาก Output Image Register ไปยัง Output Module จริง
   
   Scan Time ทั่วไป: 1-50ms (ขึ้นกับขนาด Program)
   Key Concept: PLC ไม่อ่าน Input ขณะรัน Program (ใช้ Snapshot ที่เก็บไว้)
```

**Q2: ความแตกต่างระหว่าง NO และ NC Contact คืออะไร?**
```
A: NO (Normally Open):
   - ปกติ: Circuit Open (ไม่ผ่านกระแส)
   - เมื่อ Coil ON: Contact ปิด (Circuit Closed)
   - ใช้: Start Button, Sensor Signal

   NC (Normally Closed):
   - ปกติ: Circuit Closed (ผ่านกระแส)
   - เมื่อ Coil ON: Contact เปิด (Circuit Open)
   - ใช้: Stop Button, E-Stop, Overload

   ตัวอย่างใน Ladder:
   |---[NO_Contact]---| = เปิดเมื่อ Contact ON
   |---[/NC_Contact]--| = ปิดเมื่อ Contact ON (ขีด = NC)
```

**Q3: Timer ใน PLC มีกี่ประเภท? อธิบายแต่ละประเภท**
```
A: หลักๆ มี 3 ประเภท:
   
   TON (Timer ON-Delay):
   - นับเวลาเมื่อ Input = 1
   - Output = 1 หลังจากนับครบ Preset
   - ใช้: Delay Start, Confirmation Time
   
   TOF (Timer OFF-Delay):
   - Input = 1: Output = 1 ทันที
   - Input = 0: เริ่มนับ หลังครบ Preset Output = 0
   - ใช้: Cooling Fan หลังปิดมอเตอร์, Seal-in Timer
   
   RTO (Retentive Timer):
   - นับสะสม ไม่ Reset เมื่อ Input = 0
   - ต้อง Reset ด้วย RES หรือ Bit Reset
   - ใช้: Total Run Time, Maintenance Timer
   
   บาง Brand เพิ่ม:
   TP (Pulse Timer) - Siemens: Output = 1 ตาม Preset Time เสมอ
```

**Q4: Counter ใน PLC มีกี่ประเภท?**
```
A: 3 ประเภทหลัก:
   
   CTU (Count Up):
   - นับเพิ่มทุกครั้งที่ CU Input เปลี่ยนจาก 0→1
   - Output (DN) = 1 เมื่อ ACC >= Preset
   
   CTD (Count Down):
   - นับลดทุกครั้งที่ CD Input เปลี่ยนจาก 0→1
   - Output (DN) = 1 เมื่อ ACC <= 0
   
   CTUD (Count Up/Down) - Siemens:
   - รวม CTU และ CTD ใน Block เดียว
   - มีทั้ง QU (Count Up Done) และ QD (Count Down Done)
```

**Q5: อธิบาย Latch/Unlatch หรือ Set/Reset**
```
A: Set (S) / Reset (R) หรือ Latch / Unlatch:
   
   Set (Latch): เมื่อ Input = 1 → Output = 1 และ ค้างอยู่
                แม้ Input = 0 ก็ยังค้าง Output = 1
   
   Reset (Unlatch): เมื่อ Input = 1 → Output = 0 และ ค้างอยู่
   
   ใช้: E-Stop Latch, Fault Memory, Production Counter
   
   ตัวอย่าง:
   |---[START_BTN]----(S: Motor_Run)---|
   |---[STOP_BTN]-----(R: Motor_Run)---|
   
   Priority: ถ้าทั้ง Set และ Reset เป็น 1 พร้อมกัน
   ใน Allen-Bradley: Reset ชนะ (Last wins)
   ใน Siemens: SET ชนะ (SR) หรือ RESET ชนะ (RS) เลือกได้
```

**Q6: Data Types ใน PLC มีอะไรบ้าง?**
```
A: Common Data Types:
   
   BOOL    : 1 bit  (0/1, True/False)
   BYTE    : 8 bits (0-255)
   WORD    : 16 bits (0-65535) หรือ INT (-32768 to 32767)
   DWORD   : 32 bits (DINT หรือ UDINT)
   REAL    : 32 bits Floating Point (IEEE 754)
   LREAL   : 64 bits Double Precision
   STRING  : Variable length text
   TIME    : Duration (T#5s, T#10m)
   DATE    : Calendar date
   TOD     : Time of Day
   ARRAY   : Array ของ Data Type ใดก็ได้
   STRUCT/UDT: User-Defined Type (รวม หลาย Type)
```

**Q7: อธิบาย PID Control คืออะไร**
```
A: PID = Proportional-Integral-Derivative Controller
   
   Output = Kp*e + Ki*∫e dt + Kd*de/dt
   
   e = Error = Setpoint - Process Variable
   
   Kp (Proportional): ตอบสนองตามขนาด Error
                      สูง = Fast Response แต่ Overshoot
   
   Ki (Integral):     แก้ Steady-State Error
                      สูง = ลบ Offset แต่ Wind-up ได้
   
   Kd (Derivative):   ตอบสนองต่อการเปลี่ยนแปลง
                      สูง = ลด Overshoot แต่ Noisy
   
   Tuning Methods:
   - Manual (ลองผิดลองถูก)
   - Ziegler-Nichols (Closed-Loop / Open-Loop)
   - Auto-Tune (ใน PLC Modern)
   
   ใช้: Temperature Control, Pressure, Flow, Level, Speed
```

**Q8: Analog Signal ใน PLC คืออะไร?**
```
A: Analog Signal เป็น Continuous Signal (ไม่ใช่ 0/1)
   
   มาตรฐาน Input:
   4-20mA    : อุตสาหกรรม (ตรวจจับ Wire Break ได้ที่ < 4mA)
   0-10VDC   : ง่าย แต่ต้านทาน Noise น้อยกว่า
   ±10VDC    : สำหรับ Position/Speed (มี Direction)
   Thermocouple (TC): อุณหภูมิ (Type K, J, T, etc.)
   RTD (PT100/PT1000): อุณหภูมิ (แม่นยำกว่า TC)
   
   Resolution: 12-bit = 4096 Steps, 16-bit = 65536 Steps
   
   Engineering Units Conversion:
   เช่น AI = 4-20mA = 0-100 bar
   Pressure = (AI_raw - 4) / (20 - 4) * 100
            = (AI_raw - 4) * 6.25
   
   ใน PLC Modern ใช้ FB/FC สำหรับ Scaling อัตโนมัติ
```

---

## Step 1902: Intermediate Questions (Q31-Q60)

**Q31: PROFINET vs EtherNet/IP vs Modbus TCP ต่างกันอย่างไร?**
```
A: 
┌─────────────────┬──────────────┬──────────────┬─────────────┐
│ Feature         │ PROFINET     │ EtherNet/IP  │ Modbus TCP  │
├─────────────────┼──────────────┼──────────────┼─────────────┤
│ Origin          │ Siemens/PI   │ Rockwell/ODVA│ Modicon     │
│ Speed (RT)      │ < 1ms        │ < 1ms        │ 50-500ms    │
│ Speed (IRT)     │ < 250µs      │ N/A          │ N/A         │
│ Protocol        │ Ethernet     │ Ethernet/UDP │ TCP/IP      │
│ Topology        │ Star/Ring    │ Star         │ Client/Srvr │
│ Motion Control  │ Yes (IRT)    │ Yes (CIP Sync)│ No         │
│ Cost            │ Medium       │ Medium       │ Low         │
│ Complexity      │ Medium-High  │ Medium       │ Low         │
│ Used With       │ Siemens PLC  │ AB/Rockwell  │ Any Device  │
└─────────────────┴──────────────┴──────────────┴─────────────┘
```

**Q32: อธิบาย OPC UA และทำไมถึงสำคัญ?**
```
A: OPC UA (Unified Architecture) = Open Platform Communications

   ทำไมสำคัญ:
   1. Platform Independent (Windows, Linux, Embedded)
   2. Transport Independent (TCP, HTTPS, WebSocket, MQTT)
   3. Information Model (กำหนด Data Structure มาตรฐาน)
   4. Security Built-In (Authentication, Encryption)
   5. Scalable (Embedded Device → Enterprise)
   
   ใช้ใน Industry 4.0:
   - PLC → Edge → Cloud (OPC UA over MQTT)
   - Unified Namespace (UNS)
   - Digital Twin Data Source
   
   Port: 4840 (Default TCP), 4843 (TLS)
   
   บาง PLC มี OPC UA Server Built-in:
   - Siemens S7-1500: ใช้งานได้ทันที
   - Codesys Runtime: OPC UA Built-in
   - Rockwell: ต้องใช้ Software Add-on
```

**Q33: Redundancy ใน PLC มีกี่แบบ?**
```
A: Levels ของ Redundancy:
   
   1. CPU Redundancy (Hot Standby):
      - 2 CPU ทำงานพร้อมกัน Primary + Standby
      - Switchover < 100ms (ไม่ขาดหาย)
      - Example: Siemens S7-400H, Allen-Bradley GuardLogix Pair
   
   2. Communication Redundancy (MRP/PRP/HSR):
      - Media Redundancy Protocol: Ring Topology
      - Recovery < 200ms
      - Parallel Redundancy Protocol: ใช้ 2 Path พร้อมกัน
   
   3. Power Supply Redundancy:
      - 2 PSU เชื่อมต่อผ่าน ORing Module
      - 1 ล้มเหลว อีก 1 รับภาระต่อ
   
   4. I/O Redundancy:
      - Siemens: HART-based Redundant ET200
      - สำหรับ Critical Measurements
   
   5. Full Redundancy (High Availability):
      - CPU + Power + Network + I/O
      - ใช้ใน: Nuclear, Oil & Gas (SIL 3)
      - Availability: 99.999% (5-Nine)
```

**Q34: Safety PLC คืออะไร และต่างจาก Standard PLC อย่างไร?**
```
A: Safety PLC (Failsafe PLC):
   
   ความแตกต่าง:
   ┌──────────────────┬─────────────────┬─────────────────┐
   │ Feature          │ Standard PLC    │ Safety PLC      │
   ├──────────────────┼─────────────────┼─────────────────┤
   │ CPU              │ Single          │ Dual (Compare)  │
   │ I/O Testing      │ None            │ Continuous Self │
   │ Memory Check     │ No              │ CRC Check       │
   │ Certification    │ Not Required    │ PLe/SIL 2-3     │
   │ PFHD             │ Not Specified   │ < 10E-7/h       │
   │ Response to Fault│ Alarm           │ Safe State      │
   │ Cost             │ Low             │ High (2-3x)     │
   └──────────────────┴─────────────────┴─────────────────┘
   
   Example:
   - Siemens: S7-300F/400F/1500F (F = Failsafe)
   - Rockwell: GuardLogix (Safety Application)
   - Pilz: PNOZ / PSS (Dedicated Safety)
   - HIMA: HIMax (SIL 3, Nuclear/Oil&Gas)
   
   Safety Standards:
   - IEC 61508: Base Standard (กำหนด SIL)
   - ISO 13849: Machinery Safety (กำหนด PL)
   - IEC 61511: Process Industry (กำหนด SIL)
   - IEC 62061: Machinery (SIL-based)
```

**Q35: Function Block (FB) vs Function (FC) ใน Siemens ต่างกันอย่างไร?**
```
A: 
   Function Block (FB):
   - มี Instance Data Block (DB) เป็นของตัวเอง
   - เก็บ State ระหว่าง Scan Cycle ได้
   - ใช้เมื่อต้องการ: Timer, Counter, State Machine
   - เรียกใช้: CALL FB_Name, DB_Instance
   
   Function (FC):
   - ไม่มี Memory ของตัวเอง (Stateless)
   - คืนค่าผ่าน Return Value
   - เร็วกว่า FB (ใช้ Stack Memory)
   - ใช้เมื่อ: คำนวณ, แปลงหน่วย, Utility Functions
   
   ตัวอย่าง:
   FB: Motor Control (มี State: Running/Stopped/Faulted)
   FC: Unit Conversion (Celsius to Fahrenheit = สูตรสั้น)
   
   ใน IEC 61131-3:
   Function Block = Class (มี Instance)
   Function = Subroutine/Procedure
```

**Q36: อธิบาย Software Engineering Best Practices สำหรับ PLC Programming**
```
A: Best Practices:

   1. Structured Programming:
      - แบ่งโปรแกรมเป็น Modules/FB ตาม Function
      - ไม่เขียน Program เดียวยาวพันบรรทัด
      
   2. Naming Convention:
      - ชื่อที่อ่านเข้าใจ: Motor_Conveyor_A01_Run
      - ไม่ใช้: M1, X0, รหัสอักษรย่อที่ไม่มีความหมาย
      
   3. Version Control:
      - ใช้ Git หรือ TIA Portal Version Control
      - Comment ทุก Commit
      
   4. Documentation:
      - Network Comment ทุก Network
      - Variable Description ทุกตัว
      - P&ID Reference
      
   5. Testing:
      - Unit Test ทุก FB
      - Integration Test
      - FAT ก่อน Site
      
   6. Alarm Philosophy:
      - จำกัด Alarm ที่จำเป็นจริงๆ
      - ไม่ออก Alarm ที่ไม่ต้องการ Action
```

---

## Step 1903: Senior Level Questions (Q61-Q85)

**Q61: อธิบาย ISA-88 Batch Control Model**
```
A: ISA-88 (S88) เป็นมาตรฐาน Batch Control:
   
   Physical Model:
   Enterprise → Site → Area → Process Cell → Unit → 
   Equipment Module → Control Module
   
   Procedural Model:
   Procedure → Unit Procedure → Operation → Phase → 
   Step → Transition
   
   Recipe Types:
   - General Recipe: ไม่ขึ้นกับ Equipment
   - Site Recipe: ปรับสำหรับ Site
   - Master Recipe: สำหรับ Process Cell
   - Control Recipe: ใช้ Run Batch จริง
   
   Batch Status: IDLE → RUNNING → COMPLETE/ABORTED/STOPPED
   
   ตัวอย่าง: Pharmaceutical Batch, Food Processing
```

**Q62: ISA-95 คืออะไร และเกี่ยวข้องกับ Industry 4.0 อย่างไร?**
```
A: ISA-95 = Industrial Automation Level Hierarchy

   5 Levels:
   Level 4: Business (ERP/SAP)
   Level 3: Operations (MES/MOM)
   Level 2: Control (SCADA/DCS)
   Level 1: Sensing/Manipulation (PLC)
   Level 0: Physical Process

   ISA-95 Defines:
   - Data Exchange Formats Between Levels
   - Production Scheduling, Work Orders
   - Product Definition (Recipe)
   - Performance (OEE)
   - Quality

   Industry 4.0 Connection:
   - Unified Namespace: Real-time Data ทุก Level
   - Digital Twin: Virtual Model ของ Level 0-2
   - Edge Computing: Intelligence ที่ Level 2-3
   - Cloud: Analytics ที่ Level 3-4

   B2MML (Business to Manufacturing Markup Language):
   XML Schema สำหรับ ISA-95 Data Exchange
```

**Q63: วิธีออกแบบ OEE System จาก PLC**
```
A: OEE = Availability × Performance × Quality

   Data Sources จาก PLC:
   Availability:
   - Machine Start/Stop Time (Time-stamped Events)
   - Alarm/Fault Events + Duration
   - Planned Downtime (Shift Calendar)
   
   Performance:
   - Actual Production Count (Counter)
   - Ideal Cycle Time (Recipe/Design)
   - Theoretical Maximum Output
   
   Quality:
   - Total Count (ทุกชิ้น)
   - Good Count (ผ่าน QC)
   - Rework/Reject Count
   
   PLC Implementation:
   1. Timestamp ทุก State Change
   2. Count ทุก Part (Total + Good)
   3. บันทึก Reason Code สำหรับ Downtime
   4. ส่งข้อมูลขึ้น MES ผ่าน OPC UA / Database
   
   หลีกเลี่ยง: คำนวณ OEE ใน PLC เอง
   แนะนำ: ส่ง Raw Data ขึ้น MES/SCADA ให้คำนวณ
```

**Q64: อธิบาย Functional Safety Life Cycle (IEC 61508)**
```
A: Safety Life Cycle Phases:

   Phase 1: Hazard Analysis (HAZOP)
   - ระบุ Hazard และ Risk
   
   Phase 2: Risk Assessment
   - คำนวณ Required SIL
   - Risk Graph / LOPA Method
   
   Phase 3: Safety Requirements Spec (SRS)
   - กำหนด Safety Function
   - กำหนด SIL Target
   
   Phase 4: Design
   - เลือก Safety PLC/Component
   - Architecture (1oo1, 1oo2, 2oo3)
   
   Phase 5: Implementation
   - Program Safety Function
   - Hardware Installation
   
   Phase 6: Verification
   - Test ว่าตรงตาม SRS
   - PFHD Calculation
   
   Phase 7: Validation
   - ทดสอบ End-to-End
   
   Phase 8: Operation & Maintenance
   - Proof Test Interval (ทุก 1-5 ปี)
   
   Phase 9: Modification
   - Change Management
   
   SIL Levels:
   SIL 1: PFD 10E-2 to 10E-1 (อุตสาหกรรมทั่วไป)
   SIL 2: PFD 10E-3 to 10E-2 (Oil & Gas, Heavy Industry)
   SIL 3: PFD 10E-4 to 10E-3 (Nuclear, High Hazard)
```

**Q65: Digital Twin คืออะไร และ PLC มีส่วนอย่างไร?**
```
A: Digital Twin = Virtual Model ของ Physical System ที่ Sync Real-time

   Components:
   1. Physical Asset (เครื่องจักรจริง)
   2. Digital Model (3D + Behavior Model)
   3. Data Connection (OPC UA, MQTT)
   4. Analytics/AI Layer

   PLC Role:
   - Data Source: ส่ง Real-time Process Data
   - Actuator Target: รับ Setpoints จาก Twin
   
   Types of Digital Twin:
   - Descriptive: แสดงสถานะปัจจุบัน
   - Informative: เพิ่ม Historical Data
   - Predictive: AI Prediction จาก Simulation
   - Autonomous: Self-Optimizing (ปรับ Setpoint อัตโนมัติ)
   
   Tools:
   - Siemens Plant Simulation + MindSphere
   - Rockwell Emulate3D + Plex
   - Azure Digital Twins
   - AWS IoT TwinMaker
   - Nvidia Omniverse

   Value:
   - Virtual Commissioning (Test ก่อน Build)
   - Predictive Maintenance
   - Process Optimization
   - Operator Training Simulator
```

---

## Step 1904: System Design Questions (Q86-Q100)

**Q86: ออกแบบ Safety System สำหรับ Robot Cell**
```
A: Robot Cell Safety Design:

   Step 1: Risk Assessment (ISO 12100)
   - ระบุ Hazards: Robot Collision, Pinch, Crush, Electrical
   - ประเมิน Risk Level
   - กำหนด PLr (Required Performance Level)
   
   Step 2: Safety Architecture
   Perimeter Guarding:
   - Light Curtain (SICK C4000): PLe
   - Safety Scanner (SICK S3000): PLd
   - Safety Gate + Safety Switch: PLd
   
   Robot Safety:
   - Safety Speed Monitoring
   - Safety Position Monitoring (DCS)
   - Force/Torque Limiting (Cobot)
   
   Step 3: Safety Circuit
   E-Stop → Safety Relay → Robot Power
   Light Curtain → Safety PLC → Robot Safety Input
   Safety PLC → Dual Channel Output → Contactor
   
   Step 4: Verification
   - Functional Test ทุก Safety Device
   - Response Time Measurement
   - PFHD Calculation ≤ 10E-7/h
   
   Step 5: Documentation
   - Safety Plan
   - Safety Requirement Specification
   - Test Reports
   - CE Declaration
```

**Q87: ออกแบบ Network สำหรับ Factory ทั้งหมด**
```
A: Factory Network Design (Defense in Depth):

   Zone Segmentation:
   Level 4: Corporate IT (SAP, Email)
   └── Firewall (DMZ)
   Level 3.5: DMZ (Historian, Remote Access)
   └── Industrial Firewall (Fortinet, Palo Alto)
   Level 3: MES Zone (Ignition, MES Server)
   └── Managed Switches (VLAN Separation)
   Level 2: Control Zone (SCADA, HMI Servers)
   └── PROFINET/EtherNet/IP Networks
   Level 1: Device Zone (PLCs, Drives, Robots)
   
   Redundancy:
   - Fiber Ring (MRP Recovery < 200ms)
   - Dual WAN for Remote Access
   - Redundant SCADA Server
   
   Security:
   - Network Segmentation (VLAN)
   - Industrial Firewall
   - Encrypted Remote Access (VPN)
   - No USB Ports on PLC Panels
   - Asset Management (NIST CSF)
```

**Q88-Q100: Behavioral Questions**
```
Q88: เล่าถึงโปรเจกต์ที่ยากที่สุดที่ผ่านมา

A Framework (STAR Method):
Situation: บริบทโปรเจกต์
Task: หน้าที่ความรับผิดชอบ
Action: สิ่งที่ทำ (เน้น Technical Decision)
Result: ผลลัพธ์เป็นตัวเลข (OEE เพิ่ม X%, Downtime ลด Y%)

---

Q89: มี Bug ที่ทำให้สาย Production หยุดกลางดึก จะทำอย่างไร?

A: Emergency Response:
1. รีบไปหน้างาน (หรือ Remote Login)
2. ระบุ Symptom อย่างรวดเร็ว
3. Short-term Fix: Bypass/Workaround เพื่อให้สายเดิน
4. Document สิ่งที่ทำ (Change Log)
5. Root Cause Analysis ตอนเช้า
6. Permanent Fix ผ่าน Change Management

---

Q90: ทำงานกับทีม IT เรื่อง OT/IT Convergence อย่างไร?

A: Bridge IT and OT Worlds:
- เข้าใจ Priorities ต่างกัน: IT → CIA Triad, OT → Availability First
- ใช้ Common Language: Risk, Business Impact
- เสนอ Solutions ที่ Balance Security + Availability
- สร้าง Trust ด้วยการ Deliver Results
- Joint Team สำหรับ Industrial Cybersecurity

---

Q91-Q100: (สั้นๆ)
Q91: ทำไมเลือก PLC สาย Siemens/Rockwell/Mitsubishi?
A: ขึ้นกับ Plant Standard, Local Support, Budget, Application

Q92: อธิบาย UNS (Unified Namespace)
A: Single Source of Truth, OPC UA Hierarchy ครอบคลุมทุก Level

Q93: 5-Why Analysis ใช้อย่างไร?
A: ถาม "Why" ซ้ำ 5 ครั้ง เพื่อหา Root Cause ที่แท้จริง

Q94: OEE World Class คือเท่าไหร่?
A: 85%+ (Availability 90%, Performance 95%, Quality 99.9%)

Q95: อธิบาย FMEA
A: Failure Mode and Effects Analysis: ระบุ Failure Mode + Effect + Cause + Action

Q96: ค่า CT Value คืออะไร? (Water Treatment)
A: Concentration x Time = Disinfection Efficacy

Q97: PLC และ DCS ต่างกันอย่างไร?
A: PLC: Discrete/High Speed, DCS: Continuous/Process Industry

Q98: HART Protocol คืออะไร?
A: Highway Addressable Remote Transducer: Digital Signal บน 4-20mA

Q99: ISA-18.2 เกี่ยวกับอะไร?
A: Alarm Management Standard (Rationale, Shelving, Flooding Prevention)

Q100: Cybersecurity Framework สำหรับ OT คืออะไร?
A: IEC 62443 = OT Security Standard, NIST CSF = Framework สำหรับ Critical Infrastructure
```

---

## สรุป Part 96

รวม 100 ข้อสัมภาษณ์:
- ✅ Q1-Q30: Junior Level (Basic Concepts, Timers, Counters, Analog, PID)
- ✅ Q31-Q60: Intermediate (Networking, Redundancy, Safety, FB vs FC)
- ✅ Q61-Q85: Senior (ISA-88, ISA-95, OEE, Functional Safety, Digital Twin)
- ✅ Q86-Q100: System Design + Behavioral Questions

---

## การนำทาง
- [← Part 95: Career Path International](part95_career_path_international.md)
- [→ Part 97: Advanced Troubleshooting](part97_advanced_troubleshooting.md)
- [📚 สารบัญหลัก](../README.md)
