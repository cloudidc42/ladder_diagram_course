# Part 28: EtherNet/IP (Steps 541-560)

## ภาพรวม
EtherNet/IP (Industrial Protocol) คือ Industrial Ethernet Protocol ที่ Allen-Bradley/Rockwell Automation พัฒนาและ ODVA (Open DeviceNet Vendors Association) ดูแล ใช้ CIP (Common Industrial Protocol) เป็นพื้นฐาน ซึ่งเป็น Protocol เดียวกับ DeviceNet และ ControlNet แต่วิ่งบน Ethernet/IP เป็น Protocol มาตรฐานของ Allen-Bradley ControlLogix/CompactLogix ทุกรุ่น

---

## Step 541: EtherNet/IP Fundamentals

```
┌──────────────────────────────────────────────────────────┐
│               EtherNet/IP Overview                       │
│                                                          │
│  CIP (Common Industrial Protocol):                      │
│  - Object-based Protocol Model                          │
│  - ทุก Device เป็น "Object" มี Attributes              │
│  - เหมือนกันใน DeviceNet, ControlNet, EtherNet/IP       │
│                                                          │
│  CIP ใน EtherNet/IP:                                    │
│  - Transport: TCP/UDP                                   │
│  - Port: 44818 (Explicit/TCP), 2222 (UDP/I/O)         │
│  - Physical: Standard Ethernet (IEEE 802.3)            │
│                                                          │
│  Node Types:                                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Scanner (Master):                              │   │
│  │  - ControlLogix, CompactLogix                   │   │
│  │  - ส่ง Request, เปิด Connection                │   │
│  │                                                  │   │
│  │  Adapter (Slave/Target):                        │   │
│  │  - 1734 POINT I/O, 1756 Remote I/O             │   │
│  │  - VFDs, Robots, Sensors                        │   │
│  │  - ตอบสนอง Scanner                              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  ODVA Standards:                                        │
│  - EDS (Electronic Data Sheet): เหมือน GSD/GSDML       │
│  - CIP Motion: สำหรับ Servo Control                    │
└──────────────────────────────────────────────────────────┘
```

---

## Step 542: Explicit vs Implicit Messaging

```
┌──────────────────────────────────────────────────────────┐
│           Explicit vs Implicit Messaging                 │
│                                                          │
│  EXPLICIT MESSAGING (TCP):                              │
│  ─────────────────────────────────────────────────────  │
│  - Client/Server Model                                 │
│  - Request → Response Pattern                          │
│  - ทำงานบน TCP (Port 44818)                             │
│  - ใช้สำหรับ: Configuration, Parameters, Non-cyclic     │
│  - Latency สูงกว่า แต่ Reliable                        │
│  - ตัวอย่าง: อ่าน/เขียน Drive Parameter               │
│                                                          │
│  IMPLICIT MESSAGING (UDP):                              │
│  ─────────────────────────────────────────────────────  │
│  - I/O Connection (Producer/Consumer)                  │
│  - Cyclic, Periodic หรือ Change-of-State               │
│  - ทำงานบน UDP (Port 2222)                              │
│  - ใช้สำหรับ: Process Data (I/O), Real-time Control    │
│  - Latency ต่ำ, Deterministic                           │
│  - ตัวอย่าง: อ่าน Input Status, เขียน Output         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Implicit I/O Connection Types:                 │   │
│  │  Unicast:   1 Scanner → 1 Target                │   │
│  │  Multicast: 1 Scanner → หลาย Targets            │   │
│  │             (Produced Tag = หลาย Consumer)      │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 543: Produced/Consumed Tags

```
// ControlLogix - Produced/Consumed Tags
// ทำให้ PLC แชร์ข้อมูลกันโดยตรง โดยไม่ต้องผ่าน MSG

// PRODUCED TAG:
// - PLC ที่มีข้อมูล "Publish" ข้อมูลนั้นออกมา
// - Consumer ใดๆ บน Network สามารถ Subscribe ได้
// - Tag ต้อง Produced: Check "Produce this Tag" ใน Tag Properties

// CONSUMED TAG:
// - PLC ที่ต้องการข้อมูล "Subscribe" จาก Produced Tag
// - ต้องระบุ: Source PLC, Source Tag Name, Data Type
// - Tag ถูก Update โดยอัตโนมัติทุก Connection RPI

// ตัวอย่าง: 2 ControlLogix PLCs แชร์ข้อมูล
// PLC1 (Producer): Produced Tag "Machine_Status" (DINT)
// PLC2 (Consumer): Consumed Tag "PLC1_Machine_Status"
//                  ที่ Subscribe จาก PLC1

// Configuration in Studio 5000:
// PLC1 - Tag Properties:
// Name: Machine_Status
// Type: DINT
// Check: "Produce this tag"
// Max Connections: 10

// PLC2 - New Consumed Tag:
// Name: PLC1_Machine_Status
// Type: DINT
// Source Tag: Machine_Status
// Source PLC Path: EtherNet/IP path to PLC1
// RPI: 10ms (Update Rate)

// Ladder ใน PLC2 ใช้ PLC1_Machine_Status เหมือน Local Tag:
RUNG 1:
    ┌──[PLC1_Machine_Status.0]───────────────────────────┐
    │  Bit 0 of PLC1's Machine_Status Tag               │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                         Start_My_Process │
                                                          └┘
```

---

## Step 544: EDS Files

```
┌──────────────────────────────────────────────────────────┐
│               EDS (Electronic Data Sheet)                │
│                                                          │
│  EDS เหมือน GSD/GSDML สำหรับ EtherNet/IP:              │
│  - Text File (.eds)                                     │
│  - อธิบาย Device Identity และ Capabilities             │
│  - บอก I/O Assembly Instance และ Size                  │
│  - บอก Configuration Parameters                        │
│                                                          │
│  EDS File Structure:                                    │
│  ─────────────────────────────────────────────────────  │
│  [File]                                                │
│  DescText = "Ethernet/IP Remote I/O"                  │
│  CreateDate = ...                                       │
│  Revision = 1.0                                        │
│                                                          │
│  [Device]                                              │
│  VendCode = 0x0001  (Rockwell)                         │
│  VendName = "Rockwell Automation"                      │
│  ProdType = 0x000C  (Communications Adapter)           │
│  ProdTypeStr = "Communications Adapter"                │
│  ProdCode = 0x0051                                     │
│  MajRev = 1                                            │
│  MinRev = 1                                            │
│                                                          │
│  [Params]                                              │
│  Param1 = ...                                          │
│                                                          │
│  [IO_Info]                                             │
│  Default_Input_Inst  = 100  (Input Assembly 100)       │
│  Default_Output_Inst = 150  (Output Assembly 150)      │
│  Default_Config_Inst = 1                               │
│  ─────────────────────────────────────────────────────  │
│  ติดตั้ง EDS ใน RSLogix/Studio 5000:                    │
│  Tools → EDS Wizard → Register EDS File               │
└──────────────────────────────────────────────────────────┘
```

---

## Step 545: Adding Remote I/O (1734 POINT I/O)

```
// Allen-Bradley 1734 POINT I/O บน EtherNet/IP
// Head Module: 1734-AENTR (EtherNet/IP Adapter)

// Studio 5000 Configuration:
// Step 1: Add POINT I/O Adapter ใน I/O Tree
// Controller Organizer → I/O Configuration
// Right-click "Ethernet" → New Module
// Select: 1734-AENTR

// Step 2: ตั้งค่า
// Name: Station_A_IO
// IP Address: 192.168.1.10
// RPI: 10ms (Input Update Rate)
// Electronic Keying: Compatible Module

// Step 3: เพิ่ม I/O Modules ภายใต้ 1734-AENTR
// Right-click 1734-AENTR → New Module
// Slot 1: 1734-IB8 (8 DC Input)
// Slot 2: 1734-OB8E (8 DC Output)
// Slot 3: 1734-IE4C (4 Analog Input 4-20mA)
// Slot 4: 1734-OE2C (2 Analog Output 4-20mA)

// Address Mapping:
// Station_A_IO.I.Data (Input Image):
// Station_A_IO.I.Data.0 = Slot1 Ch0
// Station_A_IO.I.Data.1 = Slot1 Ch1
// ...
// Station_A_IO.O.Data (Output Image):
// Station_A_IO.O.Data.0 = Slot2 Ch0
// ...

// Analog:
// Station_A_IO:3:I.Ch0Data = Slot3 Analog Input Ch0
// Station_A_IO:4:O.Ch0Data = Slot4 Analog Output Ch0

// Ladder Usage:
RUNG 1:
    ┌──[Station_A_IO:1:I.Data.0]──────────────────────────┐
    │  Remote Digital Input 0                             │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                 Station_A_IO:2:O.Data.0  │
                                 (Remote Digital Output 0) │
                                                          └┘
```

---

## Step 546: VFD on EtherNet/IP (PowerFlex 525)

```
// Allen-Bradley PowerFlex 525 VFD กับ EtherNet/IP

// PowerFlex 525 EtherNet/IP Configuration:
// P053: Speed Reference 0=Ethernet, 1=Analog, etc.
// P046: Start/Stop Source = 1 (Comm)
// F-Class Parameters (from HIM or Studio 5000)

// Studio 5000 - Add PowerFlex 525:
// New Module → PowerFlex 525 (ต้อง EDS ก่อน)
// IP: 192.168.1.20
// RPI: 10ms

// Standard Assembly:
// Input Assembly 21:  Start, Direction, Setpoint
// Output Assembly 71: Drive Status, Frequency, Current

// Input Data (to Drive):
// PF525:O.LogicCommand.Run       = Run bit
// PF525:O.LogicCommand.Direction = Direction bit
// PF525:O.Freq = Speed Setpoint (Hz × 100)

// Output Data (from Drive):
// PF525:I.DriveStatus.Running   = Running bit
// PF525:I.DriveStatus.AtSpeed   = At Speed
// PF525:I.DriveStatus.Faulted   = Fault
// PF525:I.OutputFreq = Actual Frequency
// PF525:I.OutputCurrent = Actual Current

RUNG 1: // Start Drive Forward
    ┌──[Start_FWD_PB]─────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────(OTE)────────┐
       │                               PF525:O.LogicCommand.Run
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)────────┐
                             PF525:O.LogicCommand.Direction
                             (0=FWD, 1=REV)               │
                                                           └┘

RUNG 2: // Set Speed
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  CPT: PF525:O.Freq = TRUNC(Speed_Hz * 100.0)      │
    └───────────────────────────────────────────────────┘

RUNG 3: // Read Running Status
    ┌──[PF525:I.DriveStatus.Running]──────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)────────┐
                                            Drive_Running   │
                                                           └┘
```

---

## Step 547: MSG Instruction (Explicit Messaging)

```
// MSG Instruction ใน Studio 5000 สำหรับ Explicit Messaging

// ใช้สำหรับ: 
// - อ่าน/เขียน Parameter บน Drive ที่ไม่อยู่ใน Implicit
// - อ่าน/เขียน Tag ใน PLC อื่น (CIP Data Table Read/Write)
// - Configuration Commands
// - Non-time-critical Data Exchange

// MSG Configuration:
// Message Type: CIP Generic, CIP Data Table Read/Write
// Path: Connection Path ไปยัง Target
// Service Code: 0x0E (Get Attribute Single) / 0x10 (Set)
// Class: 0x93 (หรือ Class ที่เฉพาะเจาะจง)
// Instance: Device Instance
// Attribute: Parameter Number

RUNG 1: // อ่าน Tag จาก PLC อื่น (CIP Data Table Read)
    ┌──[Read_Trigger]────────────────────────────────────┐
    │                                                    │
    │  MSG                                              │
    │  Message Control: MSG_PLC2_Read                   │
    │  (ตั้งใน Properties:)                             │
    │  Message Type: CIP Data Table Read               │
    │  Source Element: PLC2_Tag_To_Read[0]              │
    │  Number of Elements: 5                            │
    │  Destination Element: Local_Buffer[0]             │
    │  Path: 1,0,2,192.168.1.5,1,0  (Path to PLC2)    │
    └────────────────────────────────────────────────────┘

RUNG 2: // Check MSG Complete
    ┌──[MSG_PLC2_Read.DN]─────────────────────────────────┐
    │  Message Done                                       │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)────────┐
                                      Read_Data_Valid      │
                                                           └┘

RUNG 3: // Handle MSG Error
    ┌──[MSG_PLC2_Read.ER]─────────────────────────────────┐
    │  Message Error                                      │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)────────┐
                                         Comm_Fault        │
                                                           └┘
```

---

## Step 548: DeviceLogix (Local Logic in Adapter)

```
┌──────────────────────────────────────────────────────────┐
│               DeviceLogix Overview                       │
│                                                          │
│  DeviceLogix คือ:                                       │
│  - Embedded Logic ใน EtherNet/IP Device (Adapter)       │
│  - Device มี Simple Logic Engine ของตัวเอง              │
│  - ทำงานอิสระจาก Scanner PLC                            │
│                                                          │
│  ข้อดี:                                                 │
│  - Local Control ถ้า Network ขาด                        │
│  - ลด Network Load (ไม่ต้องส่งทุก Scan ไป PLC)          │
│  - Faster Response (Local I/O Decision)                 │
│                                                          │
│  รองรับใน: 1734 POINT I/O, 1756 Remote I/O, PowerFlex  │
│                                                          │
│  Logic Types ใน DeviceLogix:                           │
│  - AND, OR, NOT Gates                                  │
│  - Timer (TON, TOFF, TP)                               │
│  - Counter                                             │
│  - PID (บาง Module)                                    │
│  - Simple Comparators                                  │
│                                                          │
│  Configuration: RSNetWorx for EtherNet/IP               │
│  หรือ Studio 5000 DeviceLogix Editor                    │
│                                                          │
│  ตัวอย่าง: E-Stop Logic ใน Field Adapter               │
│  ถ้า E-Stop กด → ปิด Output ทุกตัวทันที                │
│  โดยไม่รอ PLC Scan (0ms response แทน ~10ms)            │
└──────────────────────────────────────────────────────────┘
```

---

## Step 549: EtherNet/IP Network Topology

```
// EtherNet/IP ใช้ Standard Ethernet Topology

// Star Topology (ที่แนะนำ):
// ┌──────────────────────────────────────────────────────┐
// │   [ControlLogix]                                     │
// │       │                                              │
// │   [Managed Switch]                                   │
// │   /   │   │   \                                      │
// │ [IO1][IO2][VFD][HMI]                                 │
// └──────────────────────────────────────────────────────┘

// Device Level Ring (DLR):
// EtherNet/IP Ring Redundancy (เหมือน MRP ของ PROFINET)
// ┌──────────────────────────────────────────────────────┐
// │  [ControlLogix] ─── [Device 1] ─── [Device 2] ──┐  │
// │        └───────────────────────────────────────────┘  │
// └──────────────────────────────────────────────────────┘
// Ring Supervisor ใน ControlLogix จัดการ Failure
// Recovery ใน < 3ms

// Linear Topology (Daisy Chain):
// ต้องใช้ Switch-capable Devices
// บาง Adapter มี Built-in 2-Port Switch

// VLAN Segregation:
// แนะนำแยก VLAN สำหรับ Process Control กับ Business Network
// VLAN 10: Process Control (PLC, I/O, VFD)
// VLAN 20: HMI, SCADA
// VLAN 30: Office/IT Network
// L3 Switch or Firewall ระหว่าง VLAN
```

---

## Step 550: Connecting Siemens to AB via OPC UA

```
// การเชื่อมต่อ Siemens PLC กับ Allen-Bradley PLC
// ผ่าน OPC UA (Open, Vendor-neutral)

// Architecture:
// Siemens S7-1500 ──[OPC UA Server]──►[OPC UA Client]── AB ControlLogix
//                                                         (ผ่าน Gateway)
// หรือ
// Siemens S7-1500 ──[OPC UA]── [SCADA/Middleware] ── [OPC UA] ── AB ControlLogix

// Option 1: OPC UA Server ใน S7-1500
// TIA Portal → CPU Properties → OPC UA → Enable OPC UA Server
// Export Information Model เพื่อ Client ใช้

// Siemens OPC UA Endpoint:
// opc.tcp://192.168.1.1:4840

// AB ControlLogix ใช้ FactoryTalk Optix หรือ Third-party OPC UA Client
// หรือใช้ EtherNet/IP to Modbus TCP Gateway:
// PROFINET Gateway: HMS Anybus Communicator

// Option 2: Middleware (SCADA)
// Siemens ─OPC UA/PROFINET─► Ignition SCADA ◄─EtherNet/IP─ AB ControlLogix
// Data Exchange ผ่าน SCADA Database

// Option 3: Hardware Gateway
// HMS Anybus Communicator:
// Side 1: EtherNet/IP (AB Side)
// Side 2: Modbus TCP หรือ PROFINET (Siemens Side)
// Configuration: Map Registers ระหว่าง 2 Protocols
```

---

## Step 551: EtherNet/IP Security

```
// EtherNet/IP Security Considerations

// ปัญหา:
// Port 44818 และ 2222 เปิดบน Network ทั่วไป
// ถ้าไม่มี Firewall → ใครก็ส่ง CIP Frame ได้

// CIP Security (อัปเดตล่าสุดของ ODVA):
// TLS 1.2 สำหรับ Explicit Messaging (TCP)
// DTLS 1.2 สำหรับ Implicit Messaging (UDP)
// Certificate-based Authentication

// Rockwell Automation Security Features:
// ControlLogix 5580: Network Access Control
// Role-based Access (Operator, Engineer, Admin)
// Change Detection and Audit Log

// Network Security Best Practices:
// ────────────────────────────────────────────────────
// 1. Network Segregation:
//    Production VLAN แยกจาก Corporate Network
//    Firewall ระหว่าง OT และ IT Network

// 2. Port Filtering:
//    Block Port 44818, 2222 จาก External Network
//    อนุญาตเฉพาะ Engineering Station ที่กำหนด

// 3. Controller Security:
//    Enable Controller Key Match
//    Protect Download with Password

// 4. Monitoring:
//    IDS/IPS สำหรับ Industrial Network
//    Log Network Activity
```

---

## Step 552: EtherNet/IP Diagnostics

```
// ตรวจสอบ EtherNet/IP Connection Status

// Studio 5000 - Online Monitor:
// Controller Organizer → I/O Configuration
// Green = OK, Red = Error

// Tag Status (Connection Status):
// Station_A_IO.I.ConnectionFaulted = TRUE ถ้า Comm Fault
// Station_A_IO.I.RunMode = TRUE ถ้า OK

// RSLinx Classic - EtherNet/IP Browse:
// เปิด RSLinx → Configuration → RSWho
// ดู EtherNet/IP Devices ที่เชื่อมต่ออยู่
// Status: OK, Faulted, No Connection

// ControlLogix I/O Module Status:
// [กด F4 ใน Studio 5000 → Controller Status]

// Studio 5000 I/O Faults:
// Module Fault:     0001h = Connection Failure
//                   0002h = Module Not Found
//                   0004h = Module Changed (Keying Error)
//                   0100h = Module Failure

// ตรวจ Connection Parameters:
RUNG 1: // Monitor Remote I/O Connection
    ┌──[Station_A_IO.I.ConnectionFaulted]────────────────┐
    │  Connection Faulted                                │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)───────┐
                                      IO_Comm_Alarm       │
                                                          └┘
```

---

## Step 553: CIP Path Addressing

```
// CIP Path ใช้ Address Device ใน Network

// Path Format:
// [Port Number, Node Address, ...]

// Port Numbers:
// 1: DP (Backplane)
// 2: EtherNet/IP (via Ethernet)

// ตัวอย่าง Path:
// "1, 0" = Backplane, Slot 0 (CPU ใน Chassis)
// "2, 192.168.1.5" = Ethernet, IP 192.168.1.5
// "1, 2, 2, 192.168.1.5, 1, 0" = 
//   ผ่าน Backplane Slot 2 (ENBT) → Ethernet IP → Backplane Slot 0

// ControlLogix Path ตัวอย่าง:
// Controller ใน Chassis ที่ IP 192.168.1.1, Slot 0:
// Path: "192.168.1.1, 1, 0"

// MSG Instruction Path:
// "2, 192.168.1.100"  = Direct Ethernet to PLC
// "192.168.1.100"     = Shorthand
// "1,2,2,192.168.1.100,1,0" = ผ่าน ENBT Module

// ตั้งค่าใน MSG Properties:
// Path: "AB_ETHIP-1, 2, 192.168.1.100"
// (AB_ETHIP-1 = RSLinx Driver Name)
```

---

## Step 554: EtherNet/IP Performance Tuning

```
// การปรับ EtherNet/IP Performance

// RPI (Requested Packet Interval):
// กำหนด Update Rate ของ I/O Connection
// Min: 0.5ms (ControlLogix 5580 กับ I/O ใกล้ๆ)
// Typical: 10-20ms สำหรับ Process I/O
// Max: 3200ms

// ข้อควรระวัง:
// RPI ต่ำมาก → Network Load สูง
// RPI ต่ำมาก → อาจทำให้ Connection Timeout (ถ้า Network ช้า)

// คำนวณ Bandwidth:
// Bandwidth per device = (Input Size + Output Size) × 8 / RPI
// ตัวอย่าง: 8 bytes I, 8 bytes O, 10ms RPI
// = 16 × 8 / 0.01 = 12,800 bps = 12.8 kbps per device

// 100 Devices × 12.8 kbps = 1.28 Mbps << 100 Mbps (OK)

// UniCast vs MultiCast:
// UniCast: 1 Producer → 1 Consumer (Port 2222)
// MultiCast: 1 Producer → หลาย Consumer (224.0.23.x)
// MultiCast ลด Bandwidth ถ้า Producer Data ใช้โดย > 1 Scanner

// InhibitTimer ใน MSG:
// กำหนด Minimum Gap ระหว่าง MSG Request
// ป้องกัน Flood ของ Explicit MSG
// ตั้งค่า: MSG Properties → InhibitTimer
```

---

## Step 555: ตัวอย่างสมบูรณ์ - Production Line Control

```
// Allen-Bradley Production Line
// ControlLogix L71 (แชสซี 10 Slot)
// Slot 0: 1756-L71 CPU
// Slot 1: 1756-EN2T Ethernet Module
// Slot 2: 1756-IF8 Analog Input
// Slot 3: 1756-OF8 Analog Output

// Remote I/O (EtherNet/IP):
// 1734-AENTR Station A: 192.168.1.10 (Fill Station)
// 1734-AENTR Station B: 192.168.1.11 (Cap Station)
// PowerFlex 525 Drive: 192.168.1.20 (Main Conveyor)

// Program Structure:
// Main Task (10ms):
//   MainRoutine: Sequencer (calls subroutines)
//   FillStation: Control Fill Station
//   CapStation: Control Cap Station
//   DriveControl: Control PowerFlex 525

// FillStation Routine:
RUNG 1: // Detect Product
    ┌──[Station_A_IO:1:I.Data.0]──────────────────────────┐
    │  Product Sensor                                     │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                     Product_At_Fill      │
                                                          └┘

RUNG 2: // Start Fill ถ้า Product Present
    ┌──[Product_At_Fill]──[NOT Fill_Complete]─────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                  Station_A_IO:2:O.Data.0 │
                                  Fill_Valve ON           │
                                                          └┘

RUNG 3: // Check Fill Level (Analog)
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  CPT: Fill_Level_Pct = Station_A_IO:3:I.Ch0Data  │
    │                        * 100.0 / 32767.0          │
    └───────────────────────────────────────────────────┘

RUNG 4: // Fill Complete Detection
    ┌──[GEQ Fill_Level_Pct, 98.0]──────────────────────────┐
    │  Fill Level >= 98%                                  │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)────────┐
                                         Fill_Complete     │
                                                           └┘
```

---

## Step 556: EtherNet/IP กับ Safety (CIP Safety)

```
┌──────────────────────────────────────────────────────────┐
│               CIP Safety Overview                        │
│                                                          │
│  เหมือน PROFIsafe แต่สำหรับ EtherNet/IP/DeviceNet        │
│  IEC 61784-3-8 Standard                                 │
│  SIL 2/3, Category 4 / PLe                             │
│                                                          │
│  Allen-Bradley Safety Products:                         │
│  - 1756-L7x Safety Controller (GuardLogix)              │
│  - 1791DS Safety I/O (POINT Guard I/O)                  │
│  - 440G Safety Interlock                                │
│                                                          │
│  Program Structure:                                     │
│  Standard Task: Standard Logic                         │
│  Safety Task: Safety Logic (แยกกัน, Protected)         │
│                                                          │
│  Safety Tags:                                           │
│  Safety Input: GuardIO:SI.SafetyInputData              │
│  Safety Output: GuardIO:SO.SafetyOutputData            │
│                                                          │
│  Studio 5000 Safety Routine:                           │
│  Safety Rung:                                          │
│  ┌──[E_Stop_Safe_Input]──[Guard_Door_Safe_In]──────────┐│
│  │                                  (Safety Output ON) ││
│  └─────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

---

## Step 557: Commissioning EtherNet/IP

```
// ขั้นตอน Commission EtherNet/IP Network

// Step 1: Physical Check
// ตรวจ RJ45 Connector ทำได้มาตรฐาน
// ตรวจ Cable Category (CAT5e ขั้นต่ำ)
// ตรวจ IP Addresses ไม่ซ้ำ (ใช้ Laptop Ping)

// Step 2: RSLinx Setup
// Install RSLinx Classic
// Add Driver: EtherNet/IP Driver
// Browse: จะเห็น Device ทุกตัวบน Network

// Step 3: Set IP Addresses
// บาง Device ต้องตั้ง IP ก่อน (BOOTP/DHCP/Static)
// ControlLogix Default: DHCP หรือ 192.168.1.1
// Tools → 1756-ENBT Module → Set IP via BOOTP Server

// Step 4: Configure in Studio 5000
// I/O Configuration → Add Modules
// ตั้ง IP, RPI, Electronic Keying
// Download Project

// Step 5: Go Online
// Controller → Online
// ดู I/O Status: ทุก Module ควร Green
// Test I/O ทีละ Point

// Step 6: Performance Verification
// RSLogix 5000 → Controller Properties → Tasks
// ดู Scan Time, CPU Load
// ตรวจ Connection Timeout Count
```

---

## Step 558: EtherNet/IP vs PROFINET Comparison

```
┌──────────────────────────────────────────────────────────┐
│           EtherNet/IP vs PROFINET                       │
├─────────────────┬──────────────────┬────────────────────┤
│  Feature        │  EtherNet/IP     │  PROFINET          │
├─────────────────┼──────────────────┼────────────────────┤
│ Standard Body   │ ODVA             │ PROFIBUS Int'l     │
│ Lead Vendor     │ Rockwell/AB      │ Siemens            │
│ Physical        │ Standard Eth     │ Standard Eth       │
│ Protocol Base   │ CIP (UDP/TCP)    │ PN RT/IRT          │
│ Min Cycle       │ ~0.5ms           │ 250μs (IRT)        │
│ Jitter          │ ~1ms             │ <1μs (IRT)         │
│ Device Desc     │ EDS              │ GSDML              │
│ Device ID       │ IP Address       │ Device Name        │
│ Ring Redund     │ DLR              │ MRP                │
│ Safety          │ CIP Safety       │ PROFIsafe          │
│ Motion          │ CIP Motion       │ PROFIdrive         │
│ Topology        │ Star/Ring/Linear │ Star/Ring/Line     │
│ Market Focus    │ Americas         │ Europe/Asia        │
└─────────────────┴──────────────────┴────────────────────┘
```

---

## Step 559: ตัวอย่าง - Siemens Gateway to AB

```
// Scenario: โรงงานมี Siemens S7-1500 ควบคุม Process
// แต่มี AB ControlLogix สำหรับ Packaging Machine
// ต้องการแชร์ข้อมูล Production Status ระหว่างกัน

// Solution: OPC UA Gateway หรือ Middleware

// Option A: Kepware OPC Server
// Kepware รัน บน Engineering PC
// Driver Siemens S7: อ่าน PLC1 Data
// Driver AB EtherNet/IP: อ่าน PLC2 Data
// Share Data via OPC UA

// Option B: Direct OPC UA (S7-1500 Server → AB OPC Client)
// S7-1500: Enable OPC UA Server (Port 4840)
// AB: ใช้ Kepware หรือ FactoryTalk Optix เป็น OPC UA Client

// Option C: MQTT Bridge
// S7-1500 → Siemens Industrial Edge → MQTT Broker
// AB → Kepware MQTT → MQTT Broker
// Shared via MQTT Topic

// Data Mapping (Example):
// Siemens Tag: Production_Status (INT)
// → OPC UA Node: ns=2;s="Production_Status"
// → AB Tag: Siemens_Production_Status (INT)

// Latency:
// OPC UA Path: ~10-100ms (ขึ้นกับ Poll Rate)
// เหมาะกับ: Status, Setpoints ไม่ใช่ High-speed I/O
```

---

## Step 560: Lab Project - EtherNet/IP Setup

**โจทย์:** ติดตั้ง EtherNet/IP Remote I/O System

```
┌──────────────────────────────────────────────────────────┐
│           EtherNet/IP Lab Project                        │
│                                                          │
│  Equipment:                                             │
│  - Allen-Bradley CompactLogix L33ER                    │
│  - 1734-AENTR POINT I/O Adapter                       │
│    - 1734-IB8: 8 DC Input                             │
│    - 1734-OB8E: 8 DC Output                            │
│    - 1734-IE4C: 4 Analog Input 4-20mA                  │
│  - PowerFlex 525 VFD                                   │
│  - CAT5e Cable, RJ45 Connectors                        │
│  - Standard Managed Switch                             │
│                                                          │
│  IP Addressing:                                         │
│  CompactLogix: 192.168.1.1                             │
│  1734-AENTR: 192.168.1.10                              │
│  PowerFlex 525: 192.168.1.20                           │
│                                                          │
│  Tasks:                                                 │
│  1. ตั้ง IP Address บน 1734-AENTR (BOOTP Server)        │
│  2. ตั้ง IP บน PowerFlex 525 (HIM)                     │
│  3. ดาวน์โหลด EDS สำหรับ PowerFlex 525                 │
│  4. Configure Studio 5000:                             │
│     a. Add POINT I/O Adapter                           │
│     b. Add I/O Modules                                 │
│     c. Add PowerFlex 525                              │
│  5. Download Project                                   │
│  6. Test Digital I/O                                   │
│  7. Test Analog Input Scaling                          │
│  8. Control PowerFlex 525: Start/Stop/Speed            │
│  9. Test Connection Fault (ถอด Cable)                  │
│ 10. สร้าง Produced Tag ใน CompactLogix                 │
│ 11. สร้าง Second PLC Consumed Tag (ถ้ามี Equipment)    │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 28

ใน Part นี้ได้เรียนรู้:

1. **EtherNet/IP Fundamentals** - CIP Protocol, ODVA
2. **Explicit vs Implicit** - TCP/UDP Messaging Types
3. **Produced/Consumed Tags** - PLC-to-PLC Data Sharing
4. **EDS Files** - Device Description
5. **POINT I/O** - Remote I/O Configuration
6. **PowerFlex 525** - VFD on EtherNet/IP
7. **MSG Instruction** - Explicit Messaging
8. **DeviceLogix** - Local Logic in Adapter
9. **CIP Path** - Address Format
10. **Performance** - RPI, Bandwidth Calculation
11. **Security** - CIP Security, Network Segmentation
12. **vs PROFINET** - Protocol Comparison

---

## Navigation
- [← Part 27: PROFINET](part27_profinet.md)
- [→ Part 29: Structured Programming](part29_structured_programming.md)
- [กลับหน้าหลัก](../README.md)
