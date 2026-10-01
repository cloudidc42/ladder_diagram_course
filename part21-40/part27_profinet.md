# Part 27: PROFINET (Steps 521-540)

## ภาพรวม
PROFINET คือ Industrial Ethernet Protocol มาตรฐาน IEC 61158 Type 10 ที่ Siemens พัฒนาเพื่อแทนที่ PROFIBUS DP ในระบบใหม่ๆ PROFINET ใช้ Infrastructure ของ Ethernet มาตรฐาน แต่เพิ่มความสามารถ Real-time และ Deterministic ที่จำเป็นสำหรับงานอุตสาหกรรม ปัจจุบัน PROFINET เป็นมาตรฐานหลักในโรงงาน Siemens ทั่วโลก

---

## Step 521: PROFINET vs Ethernet มาตรฐาน

```
┌──────────────────────────────────────────────────────────┐
│          Standard Ethernet vs PROFINET                   │
│                                                          │
│  Standard Ethernet:                                     │
│  - CSMA/CD: แย่งใช้ Medium → Collision → Retry        │
│  - Non-deterministic: ไม่แน่ใจว่า Packet จะถึงเมื่อไร  │
│  - Best-effort: ไม่รับประกัน Delivery Time            │
│  - เหมาะกับ: IT, Office, Internet                      │
│                                                          │
│  PROFINET:                                              │
│  - Deterministic: รับประกัน Delivery Time             │
│  - RT (Real-Time): Cycle ≥ 1ms ได้                     │
│  - IRT (Isochronous RT): Jitter < 1μs สำหรับ Motion    │
│  - Uses Priority IEEE 802.1Q (VLAN Priority Tag)        │
│  - เหมาะกับ: Field I/O, Robot, Motion Control         │
│                                                          │
│  PROFINET ใช้ Standard Ethernet Hardware:              │
│  - CAT5e/CAT6 Cable                                     │
│  - Standard Ethernet Switch                             │
│  - แต่ต้องการ Managed Switch สำหรับ IRT               │
│  - Connector: RJ45 หรือ M12 D-coded (IP67)            │
└──────────────────────────────────────────────────────────┘
```

---

## Step 522: RT/IRT Classes

```
┌──────────────────────────────────────────────────────────┐
│           PROFINET Communication Classes                  │
│                                                          │
│  Class 1 - RT (Real-Time):                              │
│  ─────────────────────────────────────────────────────  │
│  - Cycle Time: ≥ 1ms (ทั่วไป 2-128ms)                  │
│  - Jitter: ~ 1ms                                        │
│  - ใช้: Remote I/O, Sensors, Actuators                  │
│  - Switch: Standard Managed Switch ก็ได้               │
│  - Priority: VLAN Tag = 6 (PROFINET Standard)          │
│                                                          │
│  Class 3 - IRT (Isochronous Real-Time):                │
│  ─────────────────────────────────────────────────────  │
│  - Cycle Time: ≥ 250μs                                  │
│  - Jitter: < 1μs (< 500ns ด้วย IRT Optimized)         │
│  - ใช้: Servo Drives, High-Speed Motion Control        │
│  - Switch: ต้องการ PROFINET IRT Switch (Siemens)       │
│  - Requires: Synchronized Devices (PTCP Protocol)       │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Comparison:                                    │   │
│  │  NRT  > RT  > IRT                              │   │
│  │  Priority: Low → Medium → High                 │   │
│  │  Latency:  High → Low → Very Low               │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 523: GSDML Files

```
// GSDML = Generic Station Description Markup Language
// XML-based (แทน GSD Text File ของ PROFIBUS)

// GSDML File ประกอบด้วย:
// - Device Identity (Manufacturer, Model, Version)
// - Supported Features (RT, IRT, Cycle Times)
// - I/O Data Modules (ขนาด Input/Output)
// - Parameter Definitions
// - Diagnostics Description

// ตัวอย่าง GSDML Structure:
// <?xml version="1.0" encoding="utf-8"?>
// <ISO15745Profile>
//   <ProfileHeader>
//     <ProfileIdentification>GSDML_File_ET_200SP_CPU</ProfileIdentification>
//     <ProfileRevision>1.0</ProfileRevision>
//     <ProfileName>ET 200SP CPU</ProfileName>
//   </ProfileHeader>
//   <ProfileBody>
//     <DeviceIdentity VendorID="0x002A" DeviceID="0x0453">
//       <InfoText>ET 200SP CPU 1510SP F-2 PN</InfoText>
//       <VendorName Value="Siemens AG"/>
//     </DeviceIdentity>
//     ...
//   </ProfileBody>
// </ISO15745Profile>

// วิธีติดตั้ง GSDML ใน TIA Portal:
// Options → Manage General Station Description Files (GSD)
// → Install → Browse GSDML file
// หรือ ใน Network View: Right-click → Install GSDML
```

---

## Step 524: Device Names และ IP Assignment

```
┌──────────────────────────────────────────────────────────┐
│           PROFINET Device Naming                         │
│                                                          │
│  PROFINET ใช้ Device Name (ไม่ใช่ Address เหมือน PROFIBUS)│
│                                                          │
│  Device Name Properties:                                │
│  - Case Insensitive                                     │
│  - ต้องไม่ซ้ำกันบน Network                              │
│  - เป็น DNS-style: "conveyor-left-io-1"               │
│  - ไม่ใช้อักขระพิเศษ (เว้นแต่ -)                       │
│                                                          │
│  How PLC Assigns IP:                                    │
│  1. PLC รู้ Device Name จาก Config                      │
│  2. PLC ส่ง DCP (Discovery and Configuration Protocol)  │
│  3. Device Name ตรงกัน → Device รับ IP ที่กำหนด         │
│  4. PROFINET Communication เริ่มต้น                     │
│                                                          │
│  TIA Portal → Online → Assign PROFINET Device Name:    │
│  1. เชื่อมต่อ PC กับ Network                           │
│  2. Tools → Assign Device Name                         │
│  3. เลือก Device จาก List (MAC Address)                │
│  4. กรอก Device Name ที่ต้องการ                         │
│  5. "Assign Name" → Device รับ Name                    │
│                                                          │
│  IP Address Assignment:                                 │
│  - กำหนดใน TIA Portal Hardware Config                  │
│  - PLC Download → PLC กำหนด IP ให้ Device อัตโนมัติ   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 525: Adding ET200SP on PROFINET

```
// ET200SP คือ Remote I/O Module รุ่นใหม่ทดแทน ET200S
// ต่อผ่าน PROFINET แทน PROFIBUS

// Hardware: 
// Head Module: IM155-6 PN (Interface Module)
// I/O Modules: หลากหลาย แต่ละตัว Plug-in

// TIA Portal - Add ET200SP to PROFINET

// Step 1: ดาวน์โหลด GSDML จาก Siemens
// Industry Support → Search "ET200SP IM155-6 GSDML"

// Step 2: Install GSDML
// Options → Manage GSD Files → Install

// Step 3: เพิ่ม ET200SP ใน Network View
// Hardware Catalog → PROFINET IO → ET200SP → IM155-6 PN
// Drag to Network View → Connect to PLC PROFINET Interface

// Step 4: ตั้ง Device Name และ IP
// Properties of ET200SP:
// Device name: "station-a-io"
// IP Address: 192.168.0.10

// Step 5: เพิ่ม I/O Modules
// Double-click ET200SP → Device View
// Slot 1: IM155-6 PN (Head Module)
// Slot 2: DI 8x24VDC ST (8 Digital Inputs)
// Slot 3: DQ 8x24VDC ST (8 Digital Outputs)
// Slot 4: AI 4xI 2-Wire ST (4 Analog Inputs 4-20mA)
// Slot 5: AQ 4xU ST (4 Analog Outputs 0-10V)
// Slot 6: PM-E 24VDC (Power Module)
// Slot 7: ... (ต่อๆไป)

// Step 6: Address Mapping (Auto-assigned by TIA)
// I Addresses: 
//   I2.0-I2.7 = Slot 2 DI (Digital Inputs)
//   IW4-IW10  = Slot 4 AI (Analog Inputs, 4×2 bytes)
// Q Addresses:
//   Q2.0-Q2.7 = Slot 3 DQ (Digital Outputs)  
//   QW4-QW10  = Slot 5 AQ (Analog Outputs)

NETWORK 1: // ใช้ Remote I/O
    ┌──[I2.0]────────────────────────────────────(Q2.0)──┐
    │  Remote Input 1                    Remote Output 1  │
    └──────────────────────────────────────────────────────┘
```

---

## Step 526: PROFINET Diagnostics

```
// ประเภท PROFINET Diagnostic

// 1. Port LED on Device:
// BF (Bus Fault): แดง = Communication เสีย
// SF (System Fault): แดง = Device Fault
// Green = Normal Operation

// 2. TIA Portal Online Diagnostics:
// Online & Diagnostics → PROFINET Interface → Topology View
// เห็น Network Topology จริง
// Status: Green = OK, Yellow = Warning, Red = Fault

// 3. System Functions สำหรับ PROFINET Diagnostics:
// PNIO_ALARM (สำหรับ Process Alarm)
// RDREC (Read Record) สำหรับ Module Diagnostics

// 4. Diagnostic Alarm OB:
// OB82: Module Diagnostic (DP และ PN)
// OB83: Module Pull/Plug Event
// OB86: Rack/Station Failure (สำหรับ PN IO Station)

// OB82 ตัวอย่าง:
OB82: // Module Diagnostic
NETWORK 1:
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  // Collect Diagnostic Information                 │
    │  // #OB82_MDL_ADDR = Module Address               │
    │  // #OB82_EV_CLASS = Event Class                  │
    │  //   B#16#39 = COMING (Fault appeared)           │
    │  //   B#16#38 = GOING  (Fault disappeared)        │
    │                                                    │
    │  IF #OB82_EV_CLASS = B#16#39 THEN                │
    │    SET DB_Alarm.PN_Module_Fault                   │
    │    MOVE #OB82_MDL_ADDR → DB_Alarm.Fault_Addr     │
    │  ELSE                                              │
    │    RESET DB_Alarm.PN_Module_Fault                 │
    │  END_IF;                                           │
    └───────────────────────────────────────────────────┘
```

---

## Step 527: Media Redundancy Protocol (MRP)

```
┌──────────────────────────────────────────────────────────┐
│               Media Redundancy Protocol (MRP)            │
│                                                          │
│  MRP ป้องกัน Network Failure ด้วย Ring Topology:        │
│                                                          │
│  Physical Ring:                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │   [PLC] ─── [Switch] ─── [Device A] ─── [Dev B] │   │
│  │     └──────────────────────────────────────────┘ │   │
│  │                    Ring = Closed                  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Normal Operation:                                      │
│  MRM (Manager) บล็อก 1 Port ของ Ring                  │
│  → ป้องกัน Broadcast Storm                              │
│  → Data ไหล 1 ทิศทาง                                   │
│                                                          │
│  Link Failure:                                          │
│  MRM ตรวจพบ Link ขาด → เปิด Blocked Port              │
│  → Ring Reconfiguration ใน < 200ms (Typ < 30ms)        │
│  → Network กลับมาทำงานโดยไม่ต้องแทรกแซง               │
│                                                          │
│  MRP Roles:                                             │
│  MRM (MRP Manager): จัดการ Ring, ตรวจ Failure          │
│  MRC (MRP Client): ทุก Device ใน Ring ที่ไม่ใช่ Manager│
│                                                          │
│  ใน TIA Portal:                                         │
│  Device Properties → PROFINET Interface → MRP          │
│  → Enable Ring Port 1 และ Port 2                        │
│  → PLC CPU เป็น MRM อัตโนมัติ                          │
└──────────────────────────────────────────────────────────┘
```

### ตั้งค่า MRP ใน TIA Portal:

```
// Step 1: Network View → ต่อ Cable สร้าง Ring
// [CPU] Port1 ─── [Switch1] ─── [Device1] ─── [Switch2] ─── [CPU] Port2

// Step 2: ตั้ง MRP
// CPU Properties → PROFINET Interface Port [X2 P1 R]
// Select: "Participate in MRP Ring as Client"
// For MRM (Manager): ตั้งที่ CPU

// Step 3: สร้าง MRP Domain
// Network View → Highlight all Ring members
// Properties → MRP Configuration
// Create Domain: "mrp-domain-1"

// Step 4: Download และ Verify
// ดู Topology View → Ring Indicator ควรเป็น Closed
```

---

## Step 528: PROFINET กับ Motion Control

```
┌──────────────────────────────────────────────────────────┐
│           PROFINET for Motion Control                    │
│                                                          │
│  PROFINET IRT สำหรับ Motion Control:                   │
│  - Cycle Time: 250μs - 4ms                              │
│  - Jitter: < 1μs                                        │
│  - ใช้ PROFIdrive Profile                               │
│                                                          │
│  PROFIdrive Profile:                                    │
│  - มาตรฐาน IEC 61800-7                                 │
│  - ใช้กับ Drives/Servo Amplifiers                       │
│  - Standard Telegrams สำหรับต่าง Application          │
│                                                          │
│  Standard Telegrams:                                    │
│  Telegram 1:  Standard (Speed Control)                 │
│  Telegram 2:  Advanced (+ Torque)                      │
│  Telegram 3:  Advanced (+ Encoder)                     │
│  Telegram 5:  DSC (Dynamic Servo Control)              │
│  Telegram 102: Standard + Extension                    │
│                                                          │
│  Siemens SINAMICS กับ TIA Portal:                       │
│  - Drag SINAMICS จาก Catalog ไปยัง Network             │
│  - ตั้ง IRT Connection                                  │
│  - เลือก Telegram Type                                  │
│  - ใช้ TIA Portal S120 Drive Wizard                    │
└──────────────────────────────────────────────────────────┘
```

### ตัวอย่าง PROFIdrive Control (Telegram 1):

```
// SINAMICS S120 - Speed Control via PROFINET IRT (Telegram 1)
// STW1 (Control Word 1): QW_Drive_STW (Process Output)
// NSOLL_A (Speed Setpoint 16-bit): QW_Drive_NSP
// ZSW1 (Status Word 1):  IW_Drive_ZSW (Process Input)
// NIST_A (Speed Actual):  IW_Drive_NIST

// STW1 Bit Definitions:
// Bit 0: ON/OFF1       (1=Enable, 0=Stop)
// Bit 1: OFF2          (1=No Coast Stop)
// Bit 2: OFF3          (1=No Quick Stop)
// Bit 3: Enable Operation
// Bit 4: Ramp Function Enable
// Bit 5: Ramp Output Enable
// Bit 6: Setpoint Enable
// Bit 7: Fault Acknowledge

// Speed Setpoint (NSOLL_A):
// 0x4000 (16384) = 100% Speed (Rated Speed)
// 0x2000 (8192)  = 50% Speed
// 0x0000         = 0 (Stop)
// Negative: Reverse direction

NETWORK 1: // Enable Drive
    ┌──[Drive_Enable_PB]─────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[MOVE W#16#047F → QW_Drive_STW]─────────────────┤
          // Bits: ON, noOFF2, noOFF3, ENABLE, RFG, SP, Setpt

NETWORK 2: // Set Speed
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  CPT: NSP_Raw = Speed_Ref_Percent × 163.84        │
    │  REAL_TO_INT: NSP_Raw → QW_Drive_NSP              │
    └───────────────────────────────────────────────────┘

NETWORK 3: // Read Actual Speed
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  INT_TO_REAL: IW_Drive_NIST → Speed_Act_Raw       │
    │  DIV: Speed_Act_Raw / 163.84 → Speed_Actual_Pct   │
    └───────────────────────────────────────────────────┘
```

---

## Step 529: PROFINET Security Basics

```
┌──────────────────────────────────────────────────────────┐
│           PROFINET Security                              │
│                                                          │
│  ความเสี่ยงของ Industrial Ethernet:                     │
│  - IT Network เชื่อมต่อกัน → Malware อาจเข้า OT        │
│  - Unauthorized Access ผ่าน Ethernet                    │
│  - Man-in-the-Middle Attack                            │
│                                                          │
│  Defense in Depth (IEC 62443):                         │
│  Layer 1: Network Segmentation (VLAN, Firewall)         │
│  Layer 2: Device Authentication                         │
│  Layer 3: Secure Communication (Encrypted)             │
│  Layer 4: Monitoring and Logging                       │
│                                                          │
│  Siemens Security Features (S7-1500):                  │
│  ─────────────────────────────────────────────────────  │
│  1. PROFINET Topology Lock:                             │
│     จำกัด Device ที่เชื่อมต่อ Port ได้                  │
│     → ป้องกัน Rogue Device                             │
│                                                          │
│  2. DCP Filtering:                                      │
│     ปิดกั้น DCP Discovery สำหรับ Unauthorized Client    │
│                                                          │
│  3. PLC Access Control:                                 │
│     Password สำหรับ Read/Write Operations              │
│     Protection Level: No Protection → Write Protection │
│                       → Read/Write Protection           │
│                                                          │
│  4. OPC UA Security:                                   │
│     Certificate-based Authentication                    │
│     TLS Encryption ของ OPC UA Traffic                  │
└──────────────────────────────────────────────────────────┘
```

---

## Step 530: PROFINET Network Planning

```
// การวางแผน PROFINET Network

// Step 1: คำนวณ I/O Update Rate ที่ต้องการ
// Application → Required Update Time
// Remote I/O: 2-8ms
// Motion Control: 0.25-2ms

// Step 2: ประเมิน Network Load
// Bandwidth ที่ใช้ต่อ Device ≈ (I/O Size × 8) / Cycle Time
// ตัวอย่าง: 32 bytes I/O, 4ms Cycle = 64kbps/device
// 20 Devices × 64kbps = 1.28Mbps << 100Mbps (ปลอดภัย)

// Step 3: เลือก Switch
// RT Application: Standard Managed Switch (VLAM Priority)
// IRT Application: SCALANCE X-200IRT หรือ Hirschmann

// Step 4: Topology Design
// Star: Reliability ดี
// Ring + MRP: Fault Tolerant
// Line/Daisy-chain: ง่าย แต่ Single Point of Failure

// Step 5: IP Addressing Plan
// Subnet: 192.168.0.0/24
// PLC:    192.168.0.1
// ET200SP Station A: 192.168.0.10
// ET200SP Station B: 192.168.0.11
// VFD 1:  192.168.0.20
// VFD 2:  192.168.0.21
// HMI:    192.168.0.100
// SCADA:  192.168.0.200

// Step 6: Cable Routing
// แยก Industrial Ethernet จาก Power Cable > 30cm
// ใช้ Shielded Cable ในสิ่งแวดล้อม EMI สูง
// ใช้ M12 IP67 Connector สำหรับ Wet/Outdoor Area
```

---

## Step 531: SCALANCE Switches

```
┌──────────────────────────────────────────────────────────┐
│           Siemens SCALANCE Industrial Switches           │
│                                                          │
│  SCALANCE X-100 Series (Unmanaged):                     │
│  - ง่าย ไม่ต้องตั้งค่า                                  │
│  - ไม่รองรับ PROFINET IRT                               │
│  - ราคาถูก                                              │
│                                                          │
│  SCALANCE X-200 Series (Managed):                       │
│  - PROFINET RT รองรับ                                  │
│  - VLAN, QoS, Port Mirroring                           │
│  - X-200IRT: รองรับ PROFINET IRT ด้วย                  │
│  - Topology Detection                                   │
│                                                          │
│  SCALANCE X-300 Series (Industrial L3):                 │
│  - Layer 3 Switching/Routing                           │
│  - VLAN Routing                                        │
│  - PROFINET RT                                         │
│                                                          │
│  SCALANCE X-500 Series (Industrial L3 Advanced):        │
│  - Advanced Security Features                          │
│  - PROFINET RT                                         │
│  - IEEE 802.1X Authentication                         │
│                                                          │
│  การใช้ SCALANCE ใน TIA Portal:                         │
│  - มี GSDML สำหรับ SCALANCE                             │
│  - Drag ไปใน Network View                              │
│  - Config ผ่าน TIA Portal หรือ Web Interface           │
│  - ดู Topology View จาก TIA Portal                     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 532: IO-Link via PROFINET

```
// IO-Link: Smart Sensor/Actuator Protocol
// เชื่อมต่อ Sensor/Actuator เข้ากับ IO-Link Master
// IO-Link Master เชื่อมต่อกับ PLC ผ่าน PROFINET

// IO-Link ข้อดี:
// - อ่าน Process Data + Diagnostics + Parameter จาก Sensor
// - Sensor สามารถส่ง Multiple Data ได้ (ไม่ใช่แค่ ON/OFF)
// - Remote Parameterization (เปลี่ยน Setpoint จาก PLC)
// - Predictive Maintenance (อ่าน Operating Hours, Error Log)

// ตัวอย่าง: Balluff BIS0045 RFID Reader กับ IO-Link
// Process Data: Tag Data (Read/Write)
// Device Parameter: Tag Type, Read Mode
// Diagnostics: Error Count, Last Error

// IO-Link Master: ET200SP IO-Link Module (AL1000 or similar)
// In TIA Portal:
// - ตั้งค่า IO-Link Master Module ใน ET200SP
// - ดาวน์โหลด IODD (IO-Link Device Description) ของ Sensor
// - Map IO-Link Process Data → PROFINET I/O Area

// Ladder: ใช้ IO-Link Data เหมือน Input Register ทั่วไป
NETWORK 1: // อ่าน RFID Tag Data
    ┌──[Read_Tag_Trigger]────────────────────────────────┐
    │  // IO-Link Data อยู่ที่ I Address ของ AL1000 Module│
    │  MOVE: IW100 → Tag_Data  (2 bytes ของ Tag ID)     │
    └───────────────────────────────────────────────────┘
```

---

## Step 533: PROFINET กับ Safety (PROFIsafe)

```
┌──────────────────────────────────────────────────────────┐
│               PROFIsafe (Safety over PROFINET)           │
│                                                          │
│  PROFIsafe คือ:                                         │
│  - Extension ของ PROFINET สำหรับ Safety                │
│  - IEC 61784-3-3 Standard                               │
│  - รองรับ SIL 3 (IEC 62061) / PLe (ISO 13849)          │
│                                                          │
│  หลักการ:                                               │
│  Safety Data ถูกห่อด้วย PROFIsafe Frame                │
│  PROFIsafe Frame ส่งผ่าน PROFINET ปกติ                 │
│  เสมือน "Tunnel" ผ่าน Standard Network                 │
│                                                          │
│  PROFIsafe Black Channel:                               │
│  ─────────────────────────────────────────────────────  │
│  [ Standard Process Data ] [ PROFIsafe Data ]           │
│       ↑                          ↑                      │
│  ข้อมูลปกติ                F-Data + CRC + Counter       │
│                                                          │
│  Sequence Number + Watchdog ป้องกัน:                    │
│  - Replay Attack                                       │
│  - Delay/Insertion                                     │
│  - Loss                                                │
│  - Incorrect Sequence                                  │
│                                                          │
│  TIA Portal + F-CPU (Safety CPU):                      │
│  - ต้องการ F-CPU (เช่น S7-1515F)                        │
│  - และ F-Peripheral (ET200SP F-Module)                  │
│  - Program ใน STEP 7 Safety Program (F-OB, F-DB)       │
└──────────────────────────────────────────────────────────┘
```

---

## Step 534: Topology Detection

```
// PROFINET มี Topology Detection ที่ทรงพลัง
// ทำให้ TIA Portal รู้ว่าแต่ละ Device ต่ออยู่ที่ Port ใด

// LLDP (Link Layer Discovery Protocol):
// Device ส่ง LLDP Frame ไปยัง Neighbor
// บอก: Device Name, Port Name, IP Address
// TIA Portal รวบรวมข้อมูลสร้าง Topology Map

// ประโยชน์:
// 1. TIA Portal แสดง Physical Connection Map
// 2. ตรวจสอบว่า Cable ต่อถูก Port หรือเปล่า
// 3. ช่วย Troubleshoot Network Fault

// Forced Topology (Topology Guard):
// ตั้งค่าใน TIA Portal ว่า Device A Port 1 → Device B Port 2
// ถ้าใครต่อผิด Port → Alarm!
// ป้องกัน Wiring Error และ Unauthorized Change

// Configuration:
// TIA Portal → Network View → Device → Properties
// → PROFINET Interface → Topology
// → Check "Monitor Topology" 
// → ตั้ง Expected Neighbor

NETWORK 1: // OB82 → ตรวจ Topology Error
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  // ถ้า Cable ต่อผิด Port                          │
    │  // OB82 จะถูกเรียก ด้วย Topology Error           │
    │  IF #OB82_EV_CLASS = B#16#39 THEN                │
    │    SET Topology_Error                             │
    │  END_IF                                            │
    └───────────────────────────────────────────────────┘
```

---

## Step 535: PROFINET ใน Mixed Vendor Environment

```
// PROFINET รองรับ Multi-Vendor Devices
// ไม่ใช่เฉพาะ Siemens เท่านั้น

// Vendors ที่รองรับ PROFINET:
// Siemens: S7-300, S7-400, S7-1200, S7-1500, ET200 Series
// Wago: I/O System 750
// Phoenix Contact: AXL F PN IB
// Beckhoff: I/O Coupler (EK9300)
// Festo: CPX Terminal
// SMC: Valve Terminal
// Lenze: Drives
// SEW-EURODRIVE: Drives
// Bosch Rexroth: Drives, Hydraulics

// ข้อควรระวัง Multi-Vendor:
// 1. ตรวจสอบ Telegram Type ตรงกัน
// 2. ตรวจสอบ GSDML Version เข้ากันได้
// 3. บาง Device ไม่รองรับ IRT (เฉพาะ RT)
// 4. Cycle Time อาจต่างกัน → ตั้ง Minimum Cycle

// Example: ControlLogix ต่อกับ ET200SP
// (AB PLC ใช้ EtherNet/IP, Siemens ใช้ PROFINET)
// ต้องการ Gateway: Siemens IE/PB Link หรือ HMS Anybus
```

---

## Step 536: การ Download PROFINET Config

```
// ขั้นตอน Download Config ไปยัง PROFINET Devices

// Method 1: Download ผ่าน TIA Portal (ไปยัง PLC)
// PLC รับ Config แล้ว Download ไปยัง PROFINET Devices

// Method 2: Direct Download ไปยัง Device
// Online → Assign IP and Device Name
// Then Download Project

// ลำดับที่ถูกต้อง:
// 1. ตั้ง Device Name ด้วย DCP (TIA: Accessible Devices)
// 2. Download Project ไปยัง PLC
// 3. PLC ตั้ง IP ให้ Device โดยอัตโนมัติ
// 4. PROFINET Communication เริ่มต้น

// หมายเหตุ:
// Device Name ต้องตั้งก่อน Download Project
// ถ้าไม่มี Device Name → PLC หา Device ไม่เจอ
// → Communication ไม่ขึ้น

// Factory Reset PROFINET Device:
// กด DCP Reset (ถ้ามีปุ่ม)
// หรือ Tools → Reset to Factory Defaults
// Device Name จะถูกลบออก → ต้องตั้งใหม่
```

---

## Step 537: Performance Monitoring

```
// PROFINET Performance Monitoring Tools

// 1. TIA Portal Trace:
// Online & Diagnostics → Trace
// บันทึก Process Values ตามเวลา (ถึง μs Resolution)
// ดู Cycle Time Jitter, Signal Timing

// 2. STEP 7 / TIA Portal Online Monitor:
// Real-time Monitor ค่า I/O ของ PROFINET Devices
// ดู Update Time, Last Communication

// 3. Wireshark กับ Industrial Dissector:
// Capture Ethernet Frame บน PROFINET Network
// Dissector ถอดรหัส PROFINET Protocol
// ดู Cycle Time, Packet Loss

// 4. SCALANCE Switch Statistics:
// Port Statistics: Tx/Rx Packets, Errors
// RMON: Network Performance History

// สิ่งที่ Monitor:
// - Actual Cycle Time (vs Configured)
// - Packet Loss/Error Count
// - Network Load (%)
// - Response Time of Devices
// - Number of Missed Frames

// Normal Values:
// Cycle Time ≤ Configured × 1.1 (90% on time)
// Error Rate: 0 (Ethernet FCS errors should be 0)
// Network Load: < 60% of Bandwidth
```

---

## Step 538: PROFINET Commissioning Checklist

```
┌──────────────────────────────────────────────────────────┐
│           PROFINET Commissioning Checklist               │
│                                                          │
│  Pre-Installation:                                      │
│  ☐ IP Address Plan สร้างแล้ว ไม่มี Duplicate           │
│  ☐ Device Names กำหนดแล้ว ไม่ซ้ำกัน                    │
│  ☐ Cable Routing Plan (แยกจาก Power)                    │
│  ☐ Switch Selection เหมาะกับ Application               │
│  ☐ GSDML Files สำหรับทุก Device                         │
│                                                          │
│  Physical Installation:                                 │
│  ☐ Cable ต่อถูก Port                                    │
│  ☐ RJ45/M12 Connector ทำได้มาตรฐาน                     │
│  ☐ Shield ต่อถูก (ที่ Ground Bar)                       │
│  ☐ Power Supply ถูก Spec                               │
│  ☐ ทุก Device มี Power                                  │
│                                                          │
│  Configuration:                                         │
│  ☐ ตั้ง Device Names บนทุก Device                       │
│  ☐ TIA Portal Config: Devices, Modules, I/O             │
│  ☐ Download Config ไปยัง PLC                           │
│  ☐ PLC Download ไปยัง PROFINET Devices                 │
│                                                          │
│  Testing:                                               │
│  ☐ ทุก Device มี Status OK ใน TIA Online               │
│  ☐ BF/SF LED ทุกตัวดับ (ไม่มี Fault)                   │
│  ☐ ทดสอบ I/O ทีละ Point                               │
│  ☐ ทดสอบ Fault: ถอดสาย → Network Recovery             │
│  ☐ ตรวจสอบ Cycle Time สอดคล้องกับ Design               │
│  ☐ Document Network Layout สุดท้าย                     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 539: ตัวอย่างสมบูรณ์ - PROFINET Network

```
// System: Bottling Line
// CPU: S7-1500 1516-3 PN/DP
// ET200SP Station 1 (Fill Station): 192.168.0.10
// ET200SP Station 2 (Cap Station): 192.168.0.11
// ET200SP Station 3 (Label Station): 192.168.0.12
// SINAMICS G120 Drive: 192.168.0.20
// KTP700 HMI: 192.168.0.100
// SCALANCE XC208: 192.168.0.200 (Managed Switch)

// ──────────────────────────────────────────────────────
// PROGRAM STRUCTURE:
// OB1: Main Sequencer (ดู Part 21)
// OB30 (200ms): Analog Monitoring
// OB82: PROFINET Diagnostic OB
// OB83: Module Insert/Remove
// OB86: Station Failure
// FB10: ET200SP_Station_A (Control Station 1)
// FB11: ET200SP_Station_B (Control Station 2)
// FB12: Drive_Control (G120 Control)
// ──────────────────────────────────────────────────────

OB1: Main Program (Bottling Sequence)
NETWORK 1: // เรียก Station A Control
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB10 "Station_A_Control" DB_StationA             │
    │  IN: I10.0 (Fill Sensor)                          │
    │  IN: IW10 (Fill Level Analog)                     │
    │  OUT: Q10.0 (Fill Valve)                          │
    │  OUT: Q10.1 (Conveyor Forward)                    │
    └───────────────────────────────────────────────────┘

NETWORK 2: // เรียก Drive Control
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB12 "Drive_Control" DB_Drive                    │
    │  IN: Run_Command                                  │
    │  IN: Speed_Setpoint_RPM                           │
    │  OUT: QW20 (STW to G120)                         │
    │  OUT: QW22 (Speed SP to G120)                    │
    │  IN: IW20 (ZSW from G120)                        │
    │  IN: IW22 (Actual Speed from G120)               │
    └───────────────────────────────────────────────────┘
```

---

## Step 540: Lab Project - PROFINET Setup

**โจทย์:** ติดตั้ง PROFINET Network สมบูรณ์

```
┌──────────────────────────────────────────────────────────┐
│           PROFINET Lab Project                           │
│                                                          │
│  Equipment:                                             │
│  - Siemens S7-1200 CPU 1214C (PROFINET CPU)            │
│  - ET200SP IM155-6 PN + 4DI + 4DO + 2AI modules       │
│  - KTP400 Basic HMI (PROFINET)                         │
│  - CAT5e Ethernet Cable × 2                            │
│  - Standard Managed Switch × 1                         │
│                                                          │
│  Tasks:                                                 │
│  1. Create PROFINET Network in TIA Portal               │
│  2. Install GSDML for ET200SP                          │
│  3. Assign Device Name to ET200SP                      │
│  4. Configure I/O Modules                              │
│  5. Assign IP Addresses                                │
│  6. Download Configuration                             │
│  7. Test Remote I/O from Ladder Program                │
│  8. Add HMI and Configure Communication                │
│  9. Test Fault Scenario (disconnect cable)             │
│ 10. Verify OB86 response                               │
│ 11. Enable MRP Ring (if have 2nd cable path)           │
│ 12. Test Ring Redundancy                               │
│                                                          │
│  Network Layout:                                        │
│  [PC] ─── [Switch] ─── [S7-1200] ─── [ET200SP]         │
│                     └────────────── [KTP400]            │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 27

ใน Part นี้ได้เรียนรู้:

1. **PROFINET vs Ethernet** - Determinism, RT/IRT
2. **RT/IRT Classes** - Cycle Time, Jitter Requirements
3. **GSDML Files** - XML Device Description
4. **Device Names** - DCP Assignment
5. **ET200SP Configuration** - Modules, I/O Addressing
6. **Diagnostics** - OB82, OB83, OB86, TIA Online
7. **MRP Redundancy** - Ring Topology, Failure Recovery
8. **Motion Control** - PROFIdrive, IRT, Telegram Types
9. **Security** - Topology Guard, Access Control
10. **SCALANCE Switches** - Product Range
11. **Multi-Vendor** - PROFINET Interoperability
12. **Commissioning** - Checklist and Procedure

---

## Navigation
- [← Part 26: PROFIBUS DP](part26_profibus.md)
- [→ Part 28: EtherNet/IP](part28_ethernet_ip.md)
- [กลับหน้าหลัก](../README.md)
