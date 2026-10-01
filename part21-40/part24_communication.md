# Part 24: Communication Protocols Overview (Steps 461-480)

## ภาพรวม
ในยุคอุตสาหกรรม 4.0 PLC ไม่ได้ทำงานแยกโดดเดี่ยวอีกต่อไป แต่ต้องสื่อสารกับอุปกรณ์อื่นๆ ในโรงงาน ตั้งแต่ VFD, HMI, Robot, SCADA ไปจนถึง Cloud ความเข้าใจเรื่อง Communication Protocol จึงเป็นทักษะสำคัญของ PLC Programmer

---

## Step 461: Serial Communication Basics

```
┌──────────────────────────────────────────────────────────┐
│              Serial vs Parallel Communication            │
│                                                          │
│  Parallel (เก่า):                                        │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Device A ─────8-bit bus────► Device B           │   │
│  │  ต้องใช้ 8 สาย + Ground                         │   │
│  │  ข้อเสีย: Crosstalk, ราคาแพง, ระยะสั้น          │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Serial (ปัจจุบัน):                                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Device A ─────1-bit/time──► Device B            │   │
│  │  ใช้สายน้อยกว่า                                  │   │
│  │  ข้อดี: ระยะไกล, ราคาถูก, Noise ทนกว่า          │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Serial Communication Parameters:                        │
│  - Baud Rate: ความเร็วในการส่งข้อมูล (bits/second)      │
│  - Data Bits: จำนวน Bit ต่อ Frame (7 หรือ 8)           │
│  - Parity: การตรวจสอบ Error (None/Odd/Even)            │
│  - Stop Bits: จำนวน Stop Bit (1 หรือ 2)                │
│  - Flow Control: Handshaking (None/RTS-CTS/XON-XOFF)   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 462: RS-232 Standard

```
┌──────────────────────────────────────────────────────────┐
│                   RS-232 Characteristics                 │
│                                                          │
│  Standard: EIA/TIA-232-F                                │
│  Topology: Point-to-Point ONLY (1 sender, 1 receiver)  │
│  Distance: ≤ 15 meters (50 feet)                        │
│  Voltage:  Logic 1 = -3V to -15V                        │
│            Logic 0 = +3V to +15V (Inverted logic)       │
│  Speed:    115,200 bps maximum (typical)               │
│                                                          │
│  DB9 Connector Pinout:                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Pin 1: DCD (Data Carrier Detect)               │   │
│  │  Pin 2: RXD (Receive Data)                      │   │
│  │  Pin 3: TXD (Transmit Data)                     │   │
│  │  Pin 4: DTR (Data Terminal Ready)               │   │
│  │  Pin 5: GND (Signal Ground)                     │   │
│  │  Pin 6: DSR (Data Set Ready)                    │   │
│  │  Pin 7: RTS (Request to Send)                   │   │
│  │  Pin 8: CTS (Clear to Send)                     │   │
│  │  Pin 9: RI  (Ring Indicator)                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  Cross-wired (Null Modem) Cable:                        │
│  PLC Pin 2 (RX) ─── PC Pin 3 (TX)                      │
│  PLC Pin 3 (TX) ─── PC Pin 2 (RX)                      │
│  PLC Pin 5 (GND) ── PC Pin 5 (GND)                     │
│                                                          │
│  ใช้งาน: PLC ↔ PC, PLC ↔ HMI (ระยะสั้น), Programming  │
└──────────────────────────────────────────────────────────┘
```

**ข้อดีและข้อเสีย RS-232:**

| ข้อดี | ข้อเสีย |
|-------|---------|
| ง่าย พบในทุก PC | ระยะสั้น (<15m) |
| มาตรฐานกว้างขวาง | Point-to-point เท่านั้น |
| Debug ง่ายด้วย Terminal | ไวต่อ Noise |
| ราคาถูก | ไม่มี Differential Signaling |

---

## Step 463: RS-485 Standard

```
┌──────────────────────────────────────────────────────────┐
│                   RS-485 Characteristics                 │
│                                                          │
│  Standard: EIA/TIA-485-A                                │
│  Topology: Multi-drop Bus (1 Master, หลาย Slaves)       │
│  Devices:  สูงสุด 32 unit load (≈32 devices)           │
│  Distance: สูงสุด 1,200 meters (4,000 feet)            │
│  Voltage:  Differential Signaling                       │
│            Logic 1: A-B = +200mV ถึง +6V               │
│            Logic 0: A-B = -200mV ถึง -6V               │
│  Speed:    สูงสุด 10 Mbps (ระยะสั้น)                   │
│            100 kbps ที่ 1,200m                          │
│                                                          │
│  Differential Advantage:                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Noise → เพิ่มทั้ง A และ B เท่ากัน              │   │
│  │  Receiver ดู A-B เท่านั้น → Noise หักกัน        │   │
│  │  Common-mode noise rejection มาก                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  Wiring:                                                │
│  Master ─── A+ wire ─── Slave1 ─── Slave2 ─── Slave32 │
│  Master ─── B- wire ─── Slave1 ─── Slave2 ─── Slave32 │
│  (Daisy-chain หรือ Star topology พร้อม Hub)            │
│                                                          │
│  ใช้งาน: Modbus RTU, PROFIBUS DP                        │
└──────────────────────────────────────────────────────────┘
```

---

## Step 464: OSI Model (Simplified)

```
┌──────────────────────────────────────────────────────────┐
│         OSI Model - 7 Layers (Simplified)                │
│                                                          │
│  Layer 7: APPLICATION                                   │
│           โปรแกรมที่ผู้ใช้ใช้งาน                        │
│           ตัวอย่าง: Modbus, PROFIBUS Protocol           │
│                                                          │
│  Layer 6: PRESENTATION                                  │
│           แปลงรูปแบบข้อมูล (Encoding/Decoding)         │
│                                                          │
│  Layer 5: SESSION                                       │
│           จัดการ Session การเชื่อมต่อ                   │
│                                                          │
│  Layer 4: TRANSPORT                                     │
│           ควบคุม Data Flow, Error Checking (TCP/UDP)    │
│                                                          │
│  Layer 3: NETWORK                                       │
│           Routing, IP Addressing                        │
│           ตัวอย่าง: IP Protocol                         │
│                                                          │
│  Layer 2: DATA LINK                                     │
│           MAC Address, Frame สำหรับ Physical Link        │
│           ตัวอย่าง: Ethernet Frame                      │
│                                                          │
│  Layer 1: PHYSICAL                                      │
│           สายไฟ, Connector, Voltage Level               │
│           ตัวอย่าง: RS-485, Ethernet Cable              │
│                                                          │
│  Industrial Focus (IEC 61784):                          │
│  Layer 7 + Layer 2 + Layer 1 มักใช้ในโรงงาน            │
│  Layer 3-6 มักถูก Simplify หรือ Skip                   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 465: Industrial Protocol Types

```
┌──────────────────────────────────────────────────────────┐
│           Industrial Protocol Classification             │
│                                                          │
│  FIELDBUS (Serial-based):                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Modbus RTU      → RS-485, Universal Standard   │   │
│  │  PROFIBUS DP     → RS-485, Siemens Standard      │   │
│  │  DeviceNet       → CAN-based, Allen-Bradley      │   │
│  │  CC-Link         → RS-485, Mitsubishi            │   │
│  │  CANopen         → CAN-based, Machine Control    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  INDUSTRIAL ETHERNET:                                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Modbus TCP      → Standard Ethernet            │   │
│  │  PROFINET        → Ethernet, Siemens            │   │
│  │  EtherNet/IP     → Ethernet, Allen-Bradley      │   │
│  │  EtherCAT        → Ethernet, Real-time Motion   │   │
│  │  POWERLINK       → Ethernet, Bernecker+Rainer   │   │
│  │  Sercos III      → Ethernet, Servo Control      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  WIRELESS:                                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │  WirelessHART    → Process Instrumentation      │   │
│  │  ISA100          → Industrial Wireless           │   │
│  │  IO-Link Wireless → Sensor Level                 │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 466: Baud Rate, Parity, Stop Bits

```
// ตัวอย่าง Serial Frame (8-N-1 at 9600 baud)

//  ─────┐     ┌─┐   ┌─┐   ┌───┐   ┌───┐     ┌─────
//       │     │ │   │ │   │   │   │   │     │
//       └─────┘ └───┘ └───┘   └───┘   └─────┘
//  Idle Start  D0  D1  D2   D3  D4  D5  D6  D7 Stop
//
//  Total time = 1/9600 × 10 bits = 1.04ms per byte

// Common Baud Rates:
// 1200, 2400, 4800, 9600, 19200, 38400, 57600, 115200, 230400

// Parity Bit:
// NONE (N): ไม่มี Error Check, ส่ง 8 Data bits
// ODD  (O): จำนวน 1s ทั้งหมดต้องเป็นเลขคี่
//           ตัวอย่าง: 10110001 (4 ones → add 1 → 5 ones)
// EVEN (E): จำนวน 1s ทั้งหมดต้องเป็นเลขคู่
//           ตัวอย่าง: 10110001 (4 ones → add 0 → 4 ones)
// Parity ตรวจ Single bit error เท่านั้น

// Stop Bits:
// 1 Stop Bit: มาตรฐาน ใช้ทั่วไป
// 2 Stop Bits: ช้ากว่า แต่ให้เวลา Receiver ประมวลผล
//              ใช้เมื่อ Baud Rate ต่ำ หรือ Slow Device

// Common Configurations:
// 8-N-1: 8 Data, No Parity, 1 Stop (Modbus RTU default)
// 7-E-1: 7 Data, Even Parity, 1 Stop (ระบบเก่า)
// 8-E-2: 8 Data, Even Parity, 2 Stop (บางระบบ)
```

---

## Step 467: Cable Specifications

```
┌──────────────────────────────────────────────────────────┐
│           Cable Specifications for Industrial Use        │
│                                                          │
│  RS-232 Cable:                                          │
│  ────────────────────────────────────────────────────── │
│  Impedance: ไม่สำคัญ (ระยะสั้น)                        │
│  Wire Gauge: 24-26 AWG                                  │
│  Shielding: แนะนำสำหรับ Industrial                      │
│  Max Length: 15m at 9600 baud                           │
│                                                          │
│  RS-485 Cable:                                          │
│  ────────────────────────────────────────────────────── │
│  Characteristic Impedance: 100-120 Ω                    │
│  Wire Type: Twisted Pair (คู่บิด)                       │
│  Wire Gauge: 22-24 AWG                                  │
│  Shielding: ต้องมี Shield                               │
│  Max Length: 1,200m at 100kbps                          │
│                                                          │
│  Ethernet Cable (Industrial):                           │
│  ────────────────────────────────────────────────────── │
│  Category: CAT5e หรือ CAT6                             │
│  Industrial Rating: IP67 Connector, Armored Cable       │
│  M12 Connector: ใช้ในสิ่งแวดล้อมอุตสาหกรรม            │
│  Max Segment: 100m (Copper), 2km (Fiber)               │
│                                                          │
│  PROFIBUS Cable (Purple):                               │
│  ────────────────────────────────────────────────────── │
│  Impedance: 150 Ω                                       │
│  Type A Cable (IEC 61158)                               │
│  Core: 2 × 0.64mm (22 AWG)                             │
│  Shield: Braided                                        │
└──────────────────────────────────────────────────────────┘
```

---

## Step 468: Termination Resistors

```
┌──────────────────────────────────────────────────────────┐
│           Termination Resistor - Why Needed?             │
│                                                          │
│  สัญญาณไฟฟ้าเดินทางบนสาย เมื่อถึงปลาย:                │
│  ถ้าไม่มี Terminator → สัญญาณจะ Reflect กลับมา          │
│  → Interference กับสัญญาณใหม่                           │
│  → Bit Error, Data Corruption                           │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │                                                  │   │
│  │  [Master]──┬──[Slave1]──[Slave2]──[Slave3]      │   │
│  │     ▲      │                           ▲         │   │
│  │  Term RT   │ (ไม่ต้อง Term ตรงกลาง)   Term RT  │   │
│  │            │                                     │   │
│  │  RT = Termination Resistor                       │   │
│  │                                                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  RS-485/PROFIBUS:                                       │
│  TerminationResistor = 120Ω (= Cable Impedance)         │
│  ต่อระหว่าง A+ และ B- ที่ปลายทั้งสองด้าน               │
│                                                          │
│  Siemens PROFIBUS Connector:                            │
│  มี Switch สำหรับ Enable/Disable Termination            │
│  ON = ต่อ (ใช้เมื่อเป็น End Device)                    │
│  OFF = ไม่ต่อ (ใช้เมื่ออยู่ตรงกลาง)                   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 469: Network Topology

```
┌──────────────────────────────────────────────────────────┐
│           Network Topologies                             │
│                                                          │
│  1. BUS Topology (Daisy Chain):                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  [A]──────[B]──────[C]──────[D]                 │   │
│  │  T                                  T            │   │
│  │  (T = Terminator)                               │   │
│  │  ใช้กับ: PROFIBUS, Modbus RTU                   │   │
│  │  ข้อดี: ใช้สายน้อย                              │   │
│  │  ข้อเสีย: ถ้าสายขาด = ทุก Device หยุด           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  2. STAR Topology:                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │          [Switch/Hub]                           │   │
│  │         /     |     \                           │   │
│  │       [A]    [B]    [C]                         │   │
│  │  ใช้กับ: Ethernet, PROFINET, EtherNet/IP         │   │
│  │  ข้อดี: ถ้าสายขาด = เฉพาะ Device นั้นหยุด       │   │
│  │  ข้อเสีย: ใช้สายมาก, ต้องมี Switch               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  3. RING Topology:                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │   [A]──[B]──[C]──[D]──[A] (loop back)          │   │
│  │  ใช้กับ: PROFINET MRP, EtherNet/IP DLR          │   │
│  │  ข้อดี: Redundancy (ถ้าสายขาดจุดเดียว ยังทำงานได้) │  │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  4. TREE Topology:                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  [Root]──[Branch1]──[Leaf1]                     │   │
│  │        └──[Branch2]──[Leaf2]                    │   │
│  │                    └──[Leaf3]                   │   │
│  │  ใช้กับ: ระบบขนาดใหญ่ที่มี Sub-network          │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 470: PLC-to-PLC Communication

```
┌──────────────────────────────────────────────────────────┐
│           PLC-to-PLC Communication Methods               │
│                                                          │
│  1. I/O Link (Hard Wired):                              │
│     ─────────────────────────────────────────────────   │
│     PLC1_Out → สาย → PLC2_In                           │
│     ใช้ Output ของ PLC1 เป็น Input ของ PLC2             │
│     ง่าย แต่จำกัด (ทีละ Bit)                           │
│                                                          │
│  2. Modbus RTU:                                         │
│     ─────────────────────────────────────────────────   │
│     PLC1 (Master) ─RS-485─ PLC2 (Slave)                │
│     ส่ง/รับ Word ได้หลายตัวพร้อมกัน                     │
│     ความเร็ว: 9600-115200 bps                           │
│                                                          │
│  3. Ethernet (Modbus TCP/PROFINET/EtherNet/IP):         │
│     ─────────────────────────────────────────────────   │
│     เร็วที่สุด ใช้ Infrastructure ที่มีอยู่แล้ว          │
│     ส่งข้อมูลจำนวนมากได้                                │
│                                                          │
│  4. Proprietary (S7 Communication):                     │
│     ─────────────────────────────────────────────────   │
│     Siemens S7-S7 Communication                         │
│     PUT/GET Instructions                                 │
│     ง่ายมากระหว่าง Siemens PLC                         │
└──────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Siemens S7-S7 (PUT/GET):

```
// S7 Communication - GET Instruction
// อ่านข้อมูลจาก Remote PLC

// GET มีพารามิเตอร์:
// REQ      = Request trigger (Rising Edge)
// ID       = Connection ID (ตั้งใน Hardware Config)
// NDR      = New Data Received (Output)
// ERROR    = Error flag
// STATUS   = Status/Error code
// ADDR_1   = Address in remote PLC (P#DB10.DBX0.0 BYTE 4)
// RD_1     = Local buffer to store received data

NETWORK 1: // Trigger GET ทุก 100ms
    ┌──[100ms_Clock]───────────────────────────────────┐
    │                                                   │
    │  GET                                             │
    │  REQ: 100ms_Clock                               │
    │  ID: W#16#0001  (Connection to PLC2)            │
    │  ADDR_1: P#DB10.DBX0.0 BYTE 20  (Remote DB10)   │
    │  RD_1:   P#DB20.DBX0.0 BYTE 20  (Local DB20)    │
    └──────────────────────────────────────────────────┘

NETWORK 2: // PUT - ส่งข้อมูลไป PLC2
    ┌──[100ms_Clock]───────────────────────────────────┐
    │                                                   │
    │  PUT                                             │
    │  REQ: 100ms_Clock                               │
    │  ID: W#16#0001                                  │
    │  ADDR_1: P#DB10.DBX0.0 BYTE 10  (Remote Area)   │
    │  SD_1:   P#DB30.DBX0.0 BYTE 10  (Send Buffer)   │
    └──────────────────────────────────────────────────┘
```

---

## Step 471: PLC-to-SCADA Communication

```
┌──────────────────────────────────────────────────────────┐
│           PLC to SCADA Communication                     │
│                                                          │
│  SCADA = Supervisory Control and Data Acquisition       │
│                                                          │
│  Architecture:                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │                                                  │   │
│  │  Field Level:                                   │   │
│  │  [Sensor]──[PLC1]──[PLC2]──[PLC3]               │   │
│  │               │       │       │                  │   │
│  │               └───────┴───────┘                  │   │
│  │                       │                          │   │
│  │  Control Level:    [Ethernet Network]            │   │
│  │                       │                          │   │
│  │  SCADA Level:     [SCADA Server]                │   │
│  │                   [Historian Server]             │   │
│  │                   [HMI Clients]                  │   │
│  │                       │                          │   │
│  │  Enterprise Level: [MES/ERP System]              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  Communication Protocols PLC ↔ SCADA:                  │
│  - OPC DA/UA (Universal, Modern Standard)              │
│  - Modbus TCP (Simple, Universal)                       │
│  - PROFINET (Siemens)                                   │
│  - EtherNet/IP (Allen-Bradley)                          │
│  - Proprietary Drivers (IG, ClearSCADA, etc.)          │
└──────────────────────────────────────────────────────────┘
```

---

## Step 472: OPC-UA Overview

```
┌──────────────────────────────────────────────────────────┐
│              OPC-UA (Unified Architecture)               │
│                                                          │
│  OPC UA = Open Platform Communications Unified Arch.    │
│  IEC 62541 Standard                                     │
│                                                          │
│  ข้อดีของ OPC UA:                                       │
│  1. Platform Independent (Windows, Linux, Embedded)     │
│  2. Secure (Built-in Authentication, Encryption)        │
│  3. Information Model (ข้อมูลมี Context)               │
│  4. Cross-vendor (ใช้งานระหว่าง Vendor ต่างๆ)          │
│  5. Cloud Ready                                          │
│                                                          │
│  OPC UA vs Classic OPC DA:                             │
│  Classic OPC: เฉพาะ Windows, COM/DCOM (ปัญหา Firewall) │
│  OPC UA:      ทุก OS, TCP Socket, Firewall Friendly     │
│                                                          │
│  Siemens S7-1500: มี OPC UA Server Built-in             │
│  Allen-Bradley: ต้องใช้ Gateway หรือ Module             │
│  Mitsubishi: iQ-R มี OPC UA Server                     │
│                                                          │
│  Port: 4840 (Standard OPC UA Port)                     │
│  URL: opc.tcp://192.168.1.100:4840                     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 473: Industrial Protocol Comparison

```
┌──────────────────────────────────────────────────────────┐
│           Protocol Comparison Table                      │
├───────────────┬────────────┬──────────┬──────────────────┤
│ Protocol      │ Physical   │ Speed    │ Typical Use       │
├───────────────┼────────────┼──────────┼──────────────────┤
│ Modbus RTU    │ RS-485     │ 115kbps  │ Instruments, VFD │
│ Modbus TCP    │ Ethernet   │ 100Mbps  │ Instruments, SCADA│
│ PROFIBUS DP   │ RS-485     │ 12Mbps   │ Remote I/O        │
│ PROFINET      │ Ethernet   │ 100Mbps  │ Remote I/O, IRT   │
│ EtherNet/IP   │ Ethernet   │ 100Mbps  │ AB Remote I/O     │
│ DeviceNet     │ CAN        │ 500kbps  │ AB Field Devices  │
│ CC-Link       │ RS-485     │ 10Mbps   │ Mitsubishi        │
│ CANopen       │ CAN        │ 1Mbps    │ Machine Control   │
│ EtherCAT      │ Ethernet   │ 100Mbps  │ Servo, Motion     │
│ IO-Link       │ 3-wire     │ 230kbps  │ Smart Sensor      │
└───────────────┴────────────┴──────────┴──────────────────┘
```

---

## Step 474: RS-485 Network Setup

```
// การวาง RS-485 Network ที่ถูกต้อง

// ────────────────────────────────────────────────────────
// WIRING RULES:
// 1. ใช้ Twisted Pair เสมอ
// 2. ต่อ Termination ที่ปลาย 2 ข้าง เท่านั้น
// 3. Keep segment ≤ 1,200m
// 4. No more than 32 unit loads per segment
//    (หลาย Vendor มี 1/4 หรือ 1/8 Unit Load Transceivers)
//    → ใช้ 1/8 Unit Load → ต่อได้ 256 devices
// 5. Ground Shield ที่จุดเดียว (Control Panel End)
// 6. แนะนำ Bias Resistors ที่ Master
// ────────────────────────────────────────────────────────

// Bias Resistors:
// ป้องกัน Floating Line เมื่อทุก Device อยู่ใน Tri-state
// A+ Terminal: 560Ω ต่อ +5V
// B- Terminal: 560Ω ต่อ GND
// (หรือ 680Ω ขึ้นกับ Impedance)

// Practical Wiring Diagram:
//
//  +5V ─[560Ω]─┐
//               A+ ────────────────────────── A+  B-
//              [120Ω Term]   [Slave1] [Slave2] [120Ω Term]
//               B- ────────────────────────── A+  B-
//  GND ─[560Ω]─┘
//
//  [Master]                                   [Last Slave]

// Common Mistakes:
// ✗ ต่อ Termination ผิดที่ (เช่น ตรงกลาง)
// ✗ ลืม Termination
// ✗ Shield ต่อทั้ง 2 ปลาย (Ground Loop)
// ✗ ต่อสาย A/B สลับกัน (B=A, A=B)
// ✗ ใช้ Unshielded Cable ใน Environment มี Noise
```

---

## Step 475: การ Test Serial Communication

```
// Tools สำหรับ Test และ Debug Serial

// 1. Software Tools:
// ─────────────────────────────────────────────────────
// Hercules Terminal: เป็น Serial Terminal ง่ายๆ
// RealTerm: Advanced Terminal สำหรับ Binary Data
// Modbus Poll: Test Modbus Master
// Modbus Slave: Simulate Modbus Slave
// ModRSsim2: Modbus RTU Simulator
// Wireshark: Capture Ethernet Frames (Modbus TCP)

// 2. Hardware Tools:
// ─────────────────────────────────────────────────────
// USB-to-RS485 Converter: ต่อ PC เข้า RS-485 Network
// RS-232/485 Analyzer: บันทึก/แสดง Raw Data
// Multimeter: ตรวจ Voltage Levels

// 3. Checking Serial Connection:
// ─────────────────────────────────────────────────────
// ตรวจ RS-485 Voltage:
// A-B Idle: ประมาณ +200mV ถึง +5V (Logic 1)
// ถ้า 0V หรือ < 200mV → Bias ขาด หรือสายขาด

// ตรวจ Termination:
// วัด Resistance ระหว่าง A+ และ B-
// ถ้ามี 2 Terminators 120Ω → วัดได้ 60Ω
// ถ้าไม่มี Terminator → วัดได้ > 1kΩ
```

---

## Step 476: Siemens PLC Serial Communication Configuration

```
// Siemens S7-1200/1500 - Configuring RS-485
// ใช้ CM 1241 Serial Communication Module

// TIA Portal Configuration:
// 1. เพิ่ม CM 1241 ใน Hardware Config
// 2. ตั้งค่า:
//    - Protocol: None / Freeport / 3964(R) / Modbus
//    - Baud Rate: 9600 / 19200 / 38400 / ... / 115200
//    - Parity: None / Even / Odd
//    - Stop Bits: 1 / 2
//    - Data Bits: 7 / 8
//    - Flow Control: None / RTS/CTS

// สำหรับ Modbus RTU ใช้ MB_COMM_LOAD และ MB_MASTER/MB_SLAVE

// MB_COMM_LOAD: Configure Port
NETWORK 1:
    ┌──[FIRST_SCAN]─────────────────────────────────────┐
    │                                                    │
    │  MB_COMM_LOAD                                     │
    │  REQ: FIRST_SCAN                                 │
    │  PORT: 270 (CM1241 Hardware ID)                  │
    │  BAUD: 9600                                      │
    │  PARITY: 0 (None)                                │
    │  FLOW_CTRL: 0 (None)                             │
    │  RTS_ON_DLY: 0                                   │
    │  RTS_OFF_DLY: 0                                  │
    │  RESP_TO: 1000ms                                 │
    │  MB_DB: "MB_Master_DB" (IDB for MB_MASTER)       │
    └──────────────────────────────────────────────────┘
```

---

## Step 477: Allen-Bradley Serial Configuration

```
// Allen-Bradley SLC/MicroLogix - Serial Port Config

// MicroLogix 1100 Channel 0 Configuration:
// ─────────────────────────────────────────────────────
// Channel 0 Default: 19200-8-N-1
// Protocol: DF1 Full-Duplex, DF1 Half-Duplex, ASCII, DNP3

// ControlLogix - 1756-CP3 Serial Module
// Or 1769-ASCII for CompactLogix

// MSG Instruction for Serial Communication:
// ─────────────────────────────────────────────────────

// ControlLogix MSG Instruction:
RUNG 1:
    ┌──[Send_Trigger]───────────────────────────────────┐
    │                                                    │
    │  MSG                                              │
    │  Message Control: Msg_Config_1                    │
    │  (Properties set in Message Configuration:)       │
    │  Type: Modbus Serial                              │
    │  Path: 1,0 (Serial Module in Slot 0)             │
    └──────────────────────────────────────────────────┘
```

---

## Step 478: Mitsubishi Serial Configuration

```
// Mitsubishi FX5U - Modbus RTU via RS-485

// Hardware: FX5-485-BD (RS-485 board) หรือ FX5-485ADP

// GX Works3 Configuration:
// Module Parameter → RS-485 Board
// Baud Rate: 19200
// Parity: Even
// Stop Bits: 1
// Data Length: 8

// ADPRW Instruction (Predefined Protocol):
// หรือใช้ MC Protocol

// Modbus Master via RS-485:
// LD      SM402       // Initial Pulse (First Scan)
// MODBUS_SET  ...     // Set Modbus Parameters

// Reading Holding Registers from Slave:
// LD      SM400           // Always ON
// MODBUS_WAIT K1 K3       // Slave 1, Func 3 (Read HR)
//             K40001 K10  // Start Reg 40001, Read 10 Regs
//             D100        // Store in D100-D109
```

---

## Step 479: Communication Error Handling

```
// การจัดการ Communication Error

// ผลกระทบของ Comm Error:
// - ข้อมูลจาก Remote Device หายไป
// - PLC อาจใช้ค่าเก่า (Stale Data)
// - อาจทำให้ Process ทำงานผิดพลาด

// Strategies:
// 1. Timeout Detection
//    - ถ้าไม่ได้รับข้อมูลใน X วินาที = Fault
//    - ใช้ Timer เริ่ม Reset ทุกครั้งที่ได้รับข้อมูลดี

// 2. Data Validity Flag
//    - มี Bit บอกว่าข้อมูลล่าสุดยังใช้ได้
//    - Control Logic ต้องตรวจ Flag ก่อนใช้ข้อมูล

// 3. Fallback Value
//    - ถ้า Comm Error → ใช้ค่า Default ที่ปลอดภัย
//    - เช่น Valve = 0% ถ้า Flow Meter Comm หาย

RUNG 1: // Watchdog Timer - Reset ทุกครั้งที่ได้ข้อมูล
    ┌──[Modbus_Data_Received]────────────────(RES)───────┐
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────────────────(TON)
                                               Comm_Watchdog T#5S

RUNG 2: // ตรวจ Timeout
    ┌──[Comm_Watchdog.Q]──────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────( )──────────┐
       │                                    Comm_Fault     │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────(MOV 0.0)────┐
                                     → Instrument_Value    │
                                     (Use safe default)    │
                                                           └┘
```

---

## Step 480: Lab Project - Multi-Device Communication Setup

**โจทย์:** ออกแบบและตั้งค่าระบบ Communication สำหรับ

```
┌──────────────────────────────────────────────────────────┐
│         Multi-Device Communication Lab                   │
│                                                          │
│  ระบบ:                                                  │
│  - Siemens S7-1200 PLC (Master)                         │
│  - VFD Mitsubishi FR-E800 (Slave 1 - RS-485)           │
│  - Temperature Controller Omron E5CC (Slave 2 - RS-485)│
│  - HMI Siemens KTP700 (PROFINET)                        │
│                                                          │
│  Tasks:                                                 │
│  1. Configure CM1241 RS-485 Module                     │
│  2. Configure Modbus RTU parameters                     │
│  3. Read VFD: Frequency, Current, Status               │
│  4. Write VFD: Speed Setpoint, Run/Stop                 │
│  5. Read Temp Controller: PV, SP, Output               │
│  6. Write Temp Controller: SP                          │
│  7. Error Handling: Timeout, Fault Alarm               │
│  8. Display data on HMI                                │
│                                                          │
│  Connection:                                            │
│  S7-1200 CM1241 ─RS-485─ VFD ─ Temp Controller         │
│  (All on same RS-485 bus, different Slave addresses)    │
│                                                          │
│  Wiring:                                               │
│  PLC A+ ─── VFD A+ ─── TempCtrl A+                    │
│  PLC B- ─── VFD B- ─── TempCtrl B-                    │
│  Terminator at PLC end and TempCtrl end                 │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 24

ใน Part นี้ได้เรียนรู้:

1. **Serial Communication Basics** - Bits, Frames, Parameters
2. **RS-232** - Standard, Pinout, Usage
3. **RS-485** - Differential Signaling, Multi-drop
4. **OSI Model** - 7 Layers Overview
5. **Protocol Types** - Fieldbus vs Industrial Ethernet
6. **Baud Rate/Parity/Stop Bits** - Configuration
7. **Cable Specifications** - Shielding, Impedance
8. **Termination Resistors** - Why and Where
9. **Network Topology** - Bus, Star, Ring, Tree
10. **PLC-to-PLC** - Methods and Examples
11. **PLC-to-SCADA** - OPC-UA, Architectures
12. **Error Handling** - Timeout, Fallback Values

---

## Navigation
- [← Part 23: PID Control](part23_pid_control.md)
- [→ Part 25: Modbus RTU/TCP](part25_modbus.md)
- [กลับหน้าหลัก](../README.md)
