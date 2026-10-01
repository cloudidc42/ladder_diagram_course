# Part 26: PROFIBUS DP (Steps 501-520)

## ภาพรวม
PROFIBUS DP (Process Field Bus - Decentralized Periphery) คือ Fieldbus Protocol ที่ Siemens พัฒนาและเป็นมาตรฐาน IEC 61158 Type 3 ใช้กันอย่างแพร่หลายในโรงงานอุตสาหกรรมมานับ 30 ปี แม้ปัจจุบัน PROFINET จะเริ่มเข้ามาแทนที่ แต่ Installation เก่าจำนวนมากยังคงใช้ PROFIBUS DP อยู่

---

## Step 501: PROFIBUS DP Fundamentals

```
┌──────────────────────────────────────────────────────────┐
│               PROFIBUS DP Overview                       │
│                                                          │
│  PROFIBUS Variants:                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PROFIBUS DP: Decentralized Periphery           │   │
│  │  → Remote I/O, Drives, Instruments              │   │
│  │  → ที่ใช้มากที่สุด ในระดับ Field               │   │
│  │                                                  │   │
│  │  PROFIBUS PA: Process Automation                 │   │
│  │  → Intrinsic Safety, Hazardous Area             │   │
│  │  → ใช้ในกระบวนการ Oil & Gas, Chemical           │   │
│  │                                                  │   │
│  │  PROFIBUS FMS: Fieldbus Message Specification   │   │
│  │  → PLC-to-PLC Communication (เลิกใช้แล้ว)       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  PROFIBUS DP Specifications:                            │
│  Physical: RS-485 (EIA/TIA-485)                        │
│  Topology: Linear Bus (Daisy Chain)                    │
│  Max Nodes: 126 (Address 0-125, 127=Broadcast)        │
│  Max Segment: 32 Devices + 2 Terminators               │
│  Baud Rates: 9.6k, 19.2k, 93.75k, 187.5k, 500k,      │
│              1.5M, 3M, 6M, 12M bps                     │
│  Cable: Siemens Type A (Purple Cable, 150Ω)            │
└──────────────────────────────────────────────────────────┘
```

---

## Step 502: Master/Slave Architecture

```
┌──────────────────────────────────────────────────────────┐
│           PROFIBUS DP Master/Slave                       │
│                                                          │
│  Class 1 Master (DPM1): PLC, Controller                 │
│  - ส่ง Output Data ไป Slaves                            │
│  - รับ Input Data จาก Slaves                           │
│  - Cyclic Data Exchange                                 │
│  - มีเพียงตัวเดียวใน Segment ที่เป็น Active Master     │
│                                                          │
│  Class 2 Master (DPM2): Engineering Tool, SCADA        │
│  - ใช้สำหรับ Configuration, Diagnostics               │
│  - ไม่ได้ Exchange Process Data                        │
│  - เช่น: STEP 7 Programming Software                   │
│                                                          │
│  Slave: Remote I/O, VFD, Valve Terminal                │
│  - ตอบสนองต่อ Master เท่านั้น                          │
│  - ไม่สื่อสารกับ Slave อื่นโดยตรง                     │
│                                                          │
│  Token Ring (Multi-Master):                            │
│  ถ้ามีหลาย Master → ใช้ Token Passing                  │
│  Master ที่ถือ Token = สามารถสื่อสารกับ Slaves ได้      │
│  Token ส่งต่อกันตาม Address ที่กำหนด                   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 503: GSD Files (General Station Description)

```
┌──────────────────────────────────────────────────────────┐
│               GSD File Overview                          │
│                                                          │
│  GSD File คือ:                                          │
│  - Text File (.gsd หรือ .gse) จาก Manufacturer         │
│  - อธิบายคุณสมบัติของ PROFIBUS Device                   │
│  - เนื้อหา:                                             │
│    - Device Name, Revision                             │
│    - Supported Baud Rates                              │
│    - I/O Data Sizes                                    │
│    - Module Configurations                             │
│    - Parameter Settings                                │
│                                                          │
│  วิธีใช้ GSD File ใน TIA Portal:                       │
│  1. ดาวน์โหลด GSD จาก Manufacturer Website             │
│  2. TIA Portal → Options → Manage General Station       │
│     Description Files                                   │
│  3. Browse to GSD File → Install                       │
│  4. Device จะปรากฏใน Hardware Catalog                  │
│  5. Drag & Drop เพิ่มใน Network View                   │
│                                                          │
│  GSD File Example (partial):                           │
│  ─────────────────────────────────────────────────     │
│  GSD_Revision=3                                        │
│  Vendor_Name="Siemens AG"                              │
│  Model_Name="ET 200S IM 151-1 DP"                      │
│  Revision="V 3.0"                                      │
│  Ident_Number=0x006C                                   │
│  Protocol_Ident=0                                      │
│  Station_Type=0                                        │
│  Max_Diag_Data_Len=244                                 │
└──────────────────────────────────────────────────────────┘
```

---

## Step 504: PROFIBUS Baud Rate vs Distance

```
┌──────────────────────────────────────────────────────────┐
│         Baud Rate vs Maximum Distance                    │
│                                                          │
│  Baud Rate     │ Max Distance per Segment               │
│  ─────────────────────────────────────────────────────  │
│  9.6 kbps      │ 1,200 m                                │
│  19.2 kbps     │ 1,200 m                                │
│  93.75 kbps    │ 1,200 m                                │
│  187.5 kbps    │ 1,000 m                                │
│  500 kbps      │   400 m                                │
│  1.5 Mbps      │   200 m                                │
│  3 Mbps        │   100 m                                │
│  6 Mbps        │   100 m                                │
│  12 Mbps       │   100 m                                │
│  ─────────────────────────────────────────────────────  │
│                                                          │
│  RS-485 Repeater ขยาย Segment ได้:                      │
│  สูงสุด 9 Repeaters = 10 Segments                       │
│  1,200m × 10 = 12,000m total at 9.6kbps               │
│                                                          │
│  เลือก Baud Rate อย่างไร:                               │
│  1. ดู Device Specification (บางตัวรองรับสูงสุด 1.5M)  │
│  2. ดูระยะทาง                                           │
│  3. ดู Cycle Time ที่ต้องการ                            │
│     Cycle Time ≈ (I/O Size × 8 × 32 devices) / Baud    │
└──────────────────────────────────────────────────────────┘
```

---

## Step 505: PROFIBUS Wiring และ Termination

```
// PROFIBUS DP Wiring Requirements

// Cable Type A (IEC 61784-1, TP1):
// Impedance: 135-165Ω
// Conductor: 2 × 0.34mm² (22 AWG) twisted pair
// Shield: 100% coverage braid
// Color: Wire A = Red (+), Wire B = Green (-)
//        (ตามมาตรฐาน Siemens)

// Connector: DB9 (9-pin) หรือ M12 (รุ่นใหม่)
// DB9 Pinout:
// Pin 3: B/B' (RXD/TXD-P = +)
// Pin 8: A/A' (RXD/TXD-N = -)
// Pin 5: DGND (Data Ground, ไม่ควรต่อ PE)
// Pin 6: VP (Voltage Plus = +5V สำหรับ Terminator)
// Pin 9: RTS (Request to Send, optional)

// Termination Network at Each End:
// ─────────────────────────────────────────
//        VP (+5V)
//           │
//          [390Ω]    ← Pull-up
//           │
//         A+ Line ──────────────
//           │
//          [220Ω]    ← Termination
//           │
//         B- Line ──────────────
//           │
//          [390Ω]    ← Pull-down
//           │
//         DGND
// ─────────────────────────────────────────

// Siemens PROFIBUS Connector (6ES7 972-0BA52):
// มี Slide Switch: A=Terminated, B=Not Terminated
// เปิดฝาก่อนเดินสาย จะเห็น 2 Terminal A/B
```

---

## Step 506: Adding Remote I/O ET200S on PROFIBUS

```
// TIA Portal: Add ET200S Remote I/O on PROFIBUS DP

// Step 1: Create PROFIBUS DP Network
// 1. Device View → Add PLC
// 2. Network View → Click on DP port of S7-300/400 CPU
// 3. "Create PROFIBUS Subnet"
// 4. ตั้ง Baud Rate: 1.5 Mbps

// Step 2: Install GSD and Add ET200S
// 1. Install GSD for ET200S IM151-1 DP
// 2. Drag ET200S จาก Catalog ไปที่ Network View
// 3. Click "Connect to DP Bus"
// 4. ตั้ง PROFIBUS Address (เช่น 3)

// Step 3: Configure ET200S Modules
// Double-click ET200S → Device Configuration
// Slot 1: IM151-1 DP (Head Module)
// Slot 2: 2DI DC24V (2 Digital Inputs)
// Slot 3: 2DI DC24V (2 Digital Inputs)
// Slot 4: 2DO DC24V/0.5A (2 Digital Outputs)
// Slot 5: 2DO DC24V/0.5A (2 Digital Outputs)
// Slot 6: 2AI (2 Analog Inputs)

// Step 4: Address Mapping
// ET200S at DP Address 3:
// Input Bytes:  I100 - I103 (Digital + Analog)
// Output Bytes: Q100 - Q101 (Digital)

// I100.0 = Slot 2, Channel 0 (DI 1.1)
// I100.1 = Slot 2, Channel 1 (DI 1.2)
// I100.2 = Slot 3, Channel 0 (DI 2.1)
// I100.3 = Slot 3, Channel 1 (DI 2.2)
// Q100.0 = Slot 4, Channel 0 (DO 1.1)
// Q100.1 = Slot 4, Channel 1 (DO 1.2)
// Q100.2 = Slot 5, Channel 0 (DO 2.1)
// Q100.3 = Slot 5, Channel 1 (DO 2.2)
// IW102  = Slot 6, Channel 0 (AI 1) - Raw Analog

NETWORK 1: // ใช้ Remote I/O ปกติเหมือน Local I/O
    ┌──[I100.0]──────────────────────────────(Q100.0)───┐
    │  DI Remote 1                        DO Remote 1   │
    └──────────────────────────────────────────────────┘
```

---

## Step 507: Adding VFD on PROFIBUS DP

```
// ตัวอย่าง: Siemens MICROMASTER 440 VFD on PROFIBUS DP

// Step 1: ตั้งค่า VFD สำหรับ PROFIBUS
// P0700 = 6 (Control from PROFIBUS DP)
// P1000 = 6 (Speed Reference from PROFIBUS)
// P0918 = 5 (PROFIBUS Address = 5)
// P2050 = 1 (Enable PROFIBUS Communication)
// ตรวจสอบ Baud Rate auto-detected จาก PROFIBUS Bus

// Step 2: ติดตั้ง GSD สำหรับ MM440
// GSD File: SIEM80B3.GSD (ดาวน์โหลดจาก Siemens)

// Step 3: เพิ่ม MM440 ใน TIA Portal
// Drag MM440 จาก Catalog → Connect to PROFIBUS DP Bus
// Set PROFIBUS Address = 5

// Step 4: Telegram Configuration
// MM440 รองรับ Standard Telegrams:
// Telegram 1: Speed Control (2 Words In, 2 Words Out)
//   STW (Control Word) → Drive
//   HSW (Main Setpoint) → Drive
//   ZSW (Status Word) ← Drive
//   HIW (Main Actual Value) ← Drive

// Telegram 1 - Control Word (STW) Bits:
// Bit 0: ON/OFF1 (1=Run, 0=Stop)
// Bit 1: OFF2 (1=Permit, 0=Coast Stop)
// Bit 2: OFF3 (1=Permit, 0=Quick Stop)
// Bit 3: Enable Operation (1=Enabled)
// Bit 6: Setpoint Enable (1=Enable)
// Bit 7: Fault Acknowledge

// Speed Setpoint (HSW):
// 16384 (0x4000) = 100% of Rated Speed
// 8192 (0x2000) = 50%
// 0 = 0%

// Status Word (ZSW) Bits:
// Bit 0: Ready
// Bit 1: Ready to Operate
// Bit 2: Operation Enabled
// Bit 3: Fault
// Bit 10: At Setpoint

NETWORK 1: // Start VFD สำหรับ Forward
    ┌──[Start_FWD_PB]────────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[MOVE 047Eh → QW_Drive_STW]────────────────────┤
          // 0x047E = 0000 0100 0111 1110
          // Bits: ON1, OFF2_OK, OFF3_OK, ENABLE, FWD, SP_OK
```

---

## Step 508: PROFIBUS Diagnostics

```
// PROFIBUS Diagnostic Information
// DP Slave ส่ง Diagnostic Data กลับมายัง Master

// Diagnostic Byte 0 (Station Status 1):
// Bit 0: Station_Nonexistent  (Slave ไม่ตอบสนอง)
// Bit 1: Station_Not_Ready    (Slave ยังไม่พร้อม)
// Bit 2: Configuration_Fault  (Config ไม่ตรงกัน)
// Bit 3: Extended_Diag        (มี Extended Diag Data)
// Bit 4: Not_Supported        (Function ไม่รองรับ)
// Bit 5: Invalid_Slave_Response
// Bit 6: Parameterization_Fault
// Bit 7: Master_Lock          (Slave ถูก Lock ไว้)

// การดู Diagnostics ใน TIA Portal:
// Online → Diagnostics → PROFIBUS → Network Diagnostics
// จะเห็น Status ของทุก Slave บน Network

// Ladder Logic สำหรับ Monitor DP Slave Status:

// SFB56 (DP_DIAG): อ่าน Diagnostic ของ Slave
// หรือใช้ SFB52/SFB53 สำหรับ Send/Receive Data

NETWORK 1: // ตรวจสอบ DP Slave Status
    ┌──[100ms_Clock]─────────────────────────────────────┐
    │                                                     │
    │  SFB54 "RALRM"  (หรือ SFB56 DP_DIAG)             │
    │  EN_DT: TRUE                                       │
    │  MODE: 0                                           │
    │  MLEN: 10                                          │
    │  ASTAT → Slave_Status_DB                          │
    └───────────────────────────────────────────────────┘

// หรือใช้ System Function SFC13 (DP_SLVDIAG):
NETWORK 2: // DP Slave Diagnostic
    ┌──[Diag_Trigger]────────────────────────────────────┐
    │                                                     │
    │  CALL SFC 13                                       │
    │  REQ: Diag_Trigger                                │
    │  LADDR: W#16#0040  (DP Slave Hardware ID)          │
    │  RET_VAL → Diag_Return                            │
    │  RECORD → DB_Diag                                 │
    └───────────────────────────────────────────────────┘
```

---

## Step 509: Slot/Index Addressing ใน PROFIBUS

```
┌──────────────────────────────────────────────────────────┐
│           PROFIBUS DP-V1 Slot/Index Addressing           │
│                                                          │
│  DP-V0: Cyclic Data Exchange (Process Data)             │
│  DP-V1: Acyclic Services (Parameter Access)             │
│                                                          │
│  DP-V1 ใช้ Slot/Index เพื่อเข้าถึง Parameters:         │
│                                                          │
│  Slot: ตำแหน่งของ Module ใน Device                     │
│  Index: Parameter Number ใน Module นั้น                 │
│                                                          │
│  ตัวอย่าง: อ่าน Parameter จาก SINAMICS Drive             │
│  Slot 1 = Drive Object                                  │
│  Index = Parameter Number (เช่น P2 = Rated Speed)       │
│                                                          │
│  Siemens TIA Portal:                                    │
│  ใช้ SFB52/SFB53 (RDREC/WRREC) สำหรับ DP-V1           │
│                                                          │
│  SFB52 (RDREC): อ่าน Record (Acyclic Read)             │
│  SFB53 (WRREC): เขียน Record (Acyclic Write)           │
└──────────────────────────────────────────────────────────┘
```

### ตัวอย่าง SFB52 RDREC:

```
// อ่าน Parameter P2 (Rated Speed) จาก SINAMICS DP Slave

NETWORK 1:
    ┌──[Read_Trigger]────────────────────────────────────┐
    │                                                     │
    │  SFB52 "RDREC"                                    │
    │  REQ: Read_Trigger                                │
    │  ID: W#16#4040  (DP Address 4, Slot 1 = Object 1) │
    │  INDEX: W#16#0002  (Index 2 = P2 Parameter)        │
    │  MLEN: 4           (4 bytes = INT32)               │
    │  VALID → Read_Done                                │
    │  BUSY → Read_Busy                                 │
    │  ERROR → Read_Error                               │
    │  RECORD → Drive_Parameter_DB                      │
    └───────────────────────────────────────────────────┘
```

---

## Step 510: PROFIBUS vs Other Protocols

```
┌──────────────────────────────────────────────────────────┐
│        PROFIBUS DP vs Other Field Protocols              │
├──────────────┬──────────────┬──────────────┬────────────┤
│  Feature     │ PROFIBUS DP  │ DeviceNet    │ CC-Link    │
├──────────────┼──────────────┼──────────────┼────────────┤
│  Physical    │ RS-485       │ CAN          │ RS-485     │
│  Max Nodes   │ 126          │ 64           │ 64         │
│  Max Baud    │ 12 Mbps      │ 500 kbps     │ 10 Mbps    │
│  Max Dist    │ 1200m@9.6k  │ 500m         │ 1200m      │
│  IEC Std     │ 61158 Type3  │ 61158 Type2  │ 61158 Type7│
│  Vendor      │ Siemens      │ A-B/Rockwell │ Mitsubishi │
│  DP-V0       │ Cyclic I/O   │ Implicit Msg │ Cyclic     │
│  DP-V1       │ Acyclic Cfg  │ Explicit Msg │ Transient  │
│  Redundancy  │ Optical Ring │ Limited      │ Limited    │
└──────────────┴──────────────┴──────────────┴────────────┘

┌──────────────────────────────────────────────────────────┐
│        PROFIBUS DP vs PROFINET                          │
├──────────────────┬──────────────────────────────────────┤
│  PROFIBUS DP     │  PROFINET                           │
├──────────────────┼──────────────────────────────────────┤
│  RS-485, Serial  │  Ethernet, 100Mbps                  │
│  Purple Cable    │  CAT5e/CAT6 Cable                   │
│  9.6k-12Mbps    │  100Mbps+                           │
│  126 nodes       │  65535 nodes (theo.)               │
│  GSD Files       │  GSDML Files                        │
│  Mature, Stable  │  Newer, More Features               │
│  Replace soon    │  Current Standard                   │
│  Deterministic   │  RT/IRT Deterministic               │
└──────────────────┴──────────────────────────────────────┘
```

---

## Step 511: PROFIBUS Repeater

```
// ใช้ Repeater เมื่อต้องการขยาย Segment
// หรือเพิ่ม Number of Devices > 32

// Siemens RS 485 Repeater (6ES7 972-0AA02-0XA0)
// Max 9 Repeaters in Series
// แต่ละ Repeater แบ่งเป็น 2 Segments

// Layout ตัวอย่าง: 3 Segments
//
// [S7-CPU]──[Device 1]──...──[Device 15]──[Term]
//                  ↕ Repeater
//           [Device 16]──...──[Device 32]──[Term]
//                  ↕ Repeater
//           [Device 33]──...──[Device 48]──[Term]
//
// แต่ละ Segment มีได้สูงสุด 32 Devices
// Total: 3 × 32 = 96 Devices ใน 3 Segments

// ข้อสำคัญ:
// Repeater ไม่ใช่ Hub! Repeater นับเป็น Device 2 ตัว
// (1 ตัวในแต่ละ Segment)
// Repeater ทำ Signal Regeneration ไม่ใช่แค่ Boost
```

---

## Step 512: PROFIBUS Network Commissioning

```
// ขั้นตอนการ Commission PROFIBUS Network

// Phase 1: Physical Verification
// 1. ตรวจสอบ Wiring (A/B ไม่สลับ, ไม่มี Short)
// 2. ตรวจ Termination (เปิดที่ปลายทั้ง 2 ด้าน)
// 3. วัด Resistance A-B: ควรได้ ~100-150Ω
// 4. ตรวจ Shield Connection
// 5. ตรวจสอบ PROFIBUS Address ไม่ซ้ำกัน

// Phase 2: Configuration
// 1. ตั้ง PROFIBUS Address บน Device ก่อน power-up
//    (โดยใช้ DIP Switch หรือ Rotary Switch)
// 2. ดาวน์โหลด GSD Files
// 3. Config ใน TIA Portal: Baud Rate, Device Addresses
// 4. Download Configuration ไปยัง PLC

// Phase 3: Testing
// 1. Power-on Devices
// 2. ตรวจ LED บน Devices:
//    - BF (Bus Fault) LED ควร OFF
//    - RUN LED ควร ON
// 3. Online Diagnostics → Network Diagnostics
//    ทุก Device ควรแสดงสถานะ Green/OK
// 4. ทดสอบ I/O ทีละจุด

// Phase 4: Performance Verification
// วัด Actual Cycle Time:
// DF: System SFCX DF13 หรือดูจาก TIA Trace
// เปรียบเทียบกับ Expected Cycle Time
```

---

## Step 513: ตัวอย่างสมบูรณ์ - Remote I/O System

```
// System: Packaging Machine
// S7-300 CPU315-2 DP (DP Master)
// ET200S DP Slave ที่ Station A (Address 3)
// ET200S DP Slave ที่ Station B (Address 4)
// SINAMICS G120 Drive (Address 5)

// ──────────────────────────────────────────────────────
// Station A (ET200S, Address 3):
// I10.0-I10.7 : 8 Digital Inputs
// Q10.0-Q10.3 : 4 Digital Outputs
// IW12        : Analog Input (Pressure Sensor)
// ──────────────────────────────────────────────────────

// Station B (ET200S, Address 4):
// I20.0-I20.7 : 8 Digital Inputs
// Q20.0-Q20.3 : 4 Digital Outputs
// QW22        : Analog Output (Control Valve)
// ──────────────────────────────────────────────────────

// Main Conveyor Drive (G120, Address 5):
// QW30 = STW (Control Word)
// QW32 = HSW (Speed Setpoint)
// IW30 = ZSW (Status Word)
// IW32 = HIW (Actual Speed)
// ──────────────────────────────────────────────────────

// OB1: Main Program
NETWORK 1: // ตรวจ E-Stop (Remote Station A)
    ┌──[I10.7]/──────────────────────────────────────────┐
    │  E-Stop Station A (NC)                             │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(S)────────┐
                                        E_Stop_Active    │
                                                        └┘

NETWORK 2: // ควบคุม Conveyor Drive
    ┌──[Run_Conveyor]──[E_Stop_Active]/───────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──[MOVE W#16#047E → QW30]  // STW: Enable + Run  │

NETWORK 3: // ตั้ง Conveyor Speed (50 Hz = 50%)
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  MUL: Speed_Percent × 163.84 → QW32               │
    │  // 50% × 163.84 = 8192 = 0x2000 = 50%            │
    └───────────────────────────────────────────────────┘

NETWORK 4: // อ่าน Actual Speed
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  INT_TO_REAL: IW32 → Actual_Raw                    │
    │  DIV: Actual_Raw / 163.84 → Actual_Speed_Percent  │
    └───────────────────────────────────────────────────┘
```

---

## Step 514: PROFIBUS with Optical Fiber

```
// Optical Fiber สำหรับ PROFIBUS
// ใช้เมื่อ:
// - ระยะทางเกิน Copper Limit
// - Environment มี EMI สูง (เช่น ใกล้ Variable Frequency Drive)
// - ต้องการ Electrical Isolation
// - Potential Difference สูงระหว่าง 2 Location

// Siemens OLM (Optical Link Module):
// OLM/G11: Glass Fiber, 1 Optical Port
// OLM/G12: Glass Fiber, 2 Optical Ports (for Ring)
// OLM/P11: Plastic Fiber, 1 Port

// ระยะทาง:
// Glass Fiber (62.5/125): สูงสุด 3 km ต่อ Segment
// Plastic Fiber:          สูงสุด 80 m ต่อ Segment

// Ring Topology ด้วย OLM:
// ──────────────────────────────────────────────────────
//   [PLC CPU] ── [OLM] ══ (Fiber) ══ [OLM] ── [Device A]
//       │                                         │
//   [Device C] ── [OLM] ══ (Fiber) ══ [OLM] ── [Device B]
//       └────────── RS-485 ─────────────────────────────
//
// ถ้า Fiber ขาด 1 เส้น → OLM reconfigures Ring → ยังทำงานได้
// Redundant Path ป้องกัน Single Point of Failure
```

---

## Step 515: PROFIBUS Proxy สำหรับ Modbus RTU

```
// Gateway/Proxy: แปลง Modbus RTU เป็น PROFIBUS DP
// ใช้เมื่อต้องการต่อ Modbus Device เข้า PROFIBUS Network

// ตัวอย่าง: Siemens SIMATIC NET IE/PB Link
// หรือ Third-party Gateway เช่น HMS Anybus Gateway

// Gateway ทำหน้าที่:
// PROFIBUS Side: เป็น DP Slave
// Modbus Side:   เป็น Modbus RTU Master
// Data Mapping:  PROFIBUS I/O ↔ Modbus Registers

// Configuration:
// 1. ตั้งค่า Gateway ด้วย Software
// 2. Map PROFIBUS I/O → Modbus Register
// 3. ติดตั้ง GSD ของ Gateway ใน TIA Portal
// 4. PLC ใช้ Gateway เหมือน Remote I/O ทั่วไป

// ประโยชน์:
// ไม่ต้องเปลี่ยน Modbus Device ที่มีอยู่แล้ว
// PLC Program ง่ายขึ้น ไม่ต้องเขียน Modbus Code
```

---

## Step 516: PROFIBUS Network Documentation

```
┌──────────────────────────────────────────────────────────┐
│           PROFIBUS Network Documentation Template        │
│                                                          │
│  PROFIBUS Network Layout (Example)                      │
│                                                          │
│  Baud Rate: 1.5 Mbps                                    │
│  Max Segment Length: 200m                               │
│  Termination: Both ends                                 │
│                                                          │
│  Device List:                                           │
│  ┌──────┬────────────────────┬──────┬──────────────────┐│
│  │ Addr │ Device             │ Type │ Location         ││
│  ├──────┼────────────────────┼──────┼──────────────────┤│
│  │  0   │ S7-315-2 DP        │ DPM1 │ Control Panel    ││
│  │  3   │ ET200S IM151-1     │ Slave│ Station A        ││
│  │  4   │ ET200S IM151-1     │ Slave│ Station B        ││
│  │  5   │ MM440 VFD          │ Slave│ MCC              ││
│  │  6   │ ET200M IM153-2     │ Slave│ Station C        ││
│  │ 127  │ Broadcast (No Dev) │  -   │ -                ││
│  └──────┴────────────────────┴──────┴──────────────────┘│
│                                                          │
│  Wiring Diagram:                                        │
│  [Ctrl Panel]──[Term]──[Station A]──[Station B]──[MCC] │
│                              ↕                           │
│              [Term at Each End]                         │
└──────────────────────────────────────────────────────────┘
```

---

## Step 517: Troubleshooting PROFIBUS

```
┌──────────────────────────────────────────────────────────┐
│           PROFIBUS Troubleshooting Guide                 │
│                                                          │
│  อาการ: Slave ไม่ตอบสนอง (BF LED แดง)                  │
│  ─────────────────────────────────────────────────────  │
│  1. ตรวจ PROFIBUS Address ไม่ซ้ำกัน                     │
│  2. ตรวจ Baud Rate ตรงกัน                               │
│  3. ตรวจสาย A/B ไม่สลับ                                 │
│  4. ตรวจ Termination                                    │
│  5. ตรวจ Configuration ตรงกับ GSD                       │
│  6. วัด Voltage ระหว่าง A และ B: ควรประมาณ ±200mV      │
│                                                          │
│  อาการ: SF LED แดงที่ CPU                                │
│  ─────────────────────────────────────────────────────  │
│  1. ตรวจ OB86 (สำหรับ DP Station Error)                │
│  2. ดู System Diagnostics ใน TIA                        │
│  3. ตรวจ Diagnostic Buffer (เหตุการณ์ล่าสุด)            │
│                                                          │
│  อาการ: Data ผิดพลาด (อ่านได้แต่ค่าผิด)                │
│  ─────────────────────────────────────────────────────  │
│  1. ตรวจ Address Mapping ใน TIA Portal                  │
│  2. ตรวจ Module Slot Order ตรงกัน                       │
│  3. ตรวจ Data Length ตรงกัน                             │
│  4. ตรวจ Byte/Word Ordering                             │
│                                                          │
│  Tools:                                                 │
│  1. PROFIBUS Tester (Softing, etc.)                     │
│  2. TIA Portal Online Diagnostics                       │
│  3. LED Status บน Devices                               │
│  4. Oscilloscope บน A/B Lines                           │
└──────────────────────────────────────────────────────────┘
```

---

## Step 518: OB86 Handling (Station Failure)

```
// OB86 - Module or Rack Failure OB
// เรียกใช้เมื่อ DP Slave หรือ Rack เสียหาย

// ถ้าไม่มี OB86 ใน Program → CPU goes to STOP mode!
// จึงต้อง Create OB86 เสมอในโปรแกรม

// OB86 Start Information:
// OB86_MDL_DEFECT   (BOOL): Module Defect
// OB86_PHASE        (BYTE): Phase (0=Fault, 1=OK)
// OB86_MID_ADDR     (WORD): Hardware ID ของ Module
// OB86_MNR          (WORD): Error Code

OB86:
NETWORK 1: // Log DP Station Fault
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │                                                     │
    │  // OB86 Variables:                                │
    │  // #OB86_MDL_DEFECT = TRUE เมื่อ Fault           │
    │  // #OB86_MDL_DEFECT = FALSE เมื่อ Recover        │
    │                                                     │
    │  IF #OB86_MDL_DEFECT THEN                         │
    │    // Slave ขาด: ตั้ง Fault Flag                   │
    │    S DB_Fault.DP_Fault                            │
    │    MOVE #OB86_MID_ADDR → DB_Fault.Last_DP_Fault  │
    │  ELSE                                              │
    │    // Slave กลับมา: Clear Fault Flag               │
    │    R DB_Fault.DP_Fault                            │
    │  END_IF;                                           │
    └───────────────────────────────────────────────────┘

NETWORK 2: // Emergency Safe State เมื่อ DP Fault
    ┌──[DB_Fault.DP_Fault]───────────────────────────────┐
    │  DP Fault Active                                   │
    └──┬─────────────────────────────────────────────────┘
       └──[ตั้ง Safe State สำหรับ Process]────────────────┤
          // ปิด Motors, ปิด Heaters, Safe Position
```

---

## Step 519: Migration จาก PROFIBUS DP เป็น PROFINET

```
┌──────────────────────────────────────────────────────────┐
│           PROFIBUS → PROFINET Migration                  │
│                                                          │
│  Options สำหรับ Migration:                              │
│                                                          │
│  Option 1: DP/PN Coupler                               │
│  ─────────────────────────────────────────────────────  │
│  ติดตั้ง DP/PN Coupler (6GK1411-2AB10)                 │
│  DP Side: ต่อกับ PROFIBUS Network เดิม                  │
│  PN Side: ต่อกับ PROFINET Network ใหม่                   │
│  ข้อดี: ไม่ต้องเปลี่ยน Device เดิม                     │
│  ข้อเสีย: เพิ่ม Latency, ราคา Coupler                  │
│                                                          │
│  Option 2: Replace Devices Gradually                   │
│  ─────────────────────────────────────────────────────  │
│  เปลี่ยน ET200S (DP) เป็น ET200SP (PN) ทีละจุด          │
│  ข้อดี: Migration ค่อยๆ ทำได้                           │
│  ข้อเสีย: ใช้เวลา และต้องหยุด Process                  │
│                                                          │
│  Option 3: Replace All at Once                         │
│  ─────────────────────────────────────────────────────  │
│  เปลี่ยน CPU ใหม่, DP Slaves → PN Devices ทั้งหมด       │
│  ข้อดี: Clean Solution                                 │
│  ข้อเสีย: Cost สูง, Downtime มาก                       │
│                                                          │
│  ความแตกต่าง Program:                                  │
│  PROFIBUS: ใช้ I/O Addresses (IW, QW)                  │
│  PROFINET: ใช้ I/O Addresses เหมือนกัน (Compatible)    │
│  โปรแกรม Logic ไม่ต้องเปลี่ยน!                         │
└──────────────────────────────────────────────────────────┘
```

---

## Step 520: Lab Project - PROFIBUS Network

**โจทย์:** ติดตั้งและตั้งค่า PROFIBUS DP Network

```
┌──────────────────────────────────────────────────────────┐
│           PROFIBUS DP Lab Project                        │
│                                                          │
│  Equipment:                                             │
│  - Siemens S7-300 CPU315-2DP (Master)                  │
│  - ET200S IM151-1DP (Slave, Address 3)                  │
│    - 4DI, 4DO, 2AI Modules                             │
│  - SIMATIC MM420 VFD (Slave, Address 5)                 │
│  - PROFIBUS Type A Cable (Purple)                       │
│  - PROFIBUS Connector × 3                               │
│                                                          │
│  Tasks:                                                 │
│  1. ตั้ง PROFIBUS Address บน ET200S (DIP Switch)        │
│  2. ตั้ง PROFIBUS Address บน MM420 (Parameter P0918)    │
│  3. สร้าง PROFIBUS DP Network ใน TIA Portal             │
│  4. เพิ่ม ET200S และตั้งค่า Modules                    │
│  5. เพิ่ม MM420 ด้วย GSD File                          │
│  6. Download Configuration                             │
│  7. Test Digital I/O จาก Remote ET200S                 │
│  8. ควบคุม MM420: Start/Stop/Speed                     │
│  9. ตรวจสอบ BF LED และ Online Diagnostics              │
│ 10. ทดสอบ Fault Scenario: ถอดสาย → ดู OB86 Response   │
│                                                          │
│  Wiring:                                               │
│  CPU DP Port → Cable → ET200S (Term ON)                │
│  ET200S → Cable → MM420 (Term ON)                      │
│  CPU DP end (Term OFF ถ้าใช้ External)                  │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 26

ใน Part นี้ได้เรียนรู้:

1. **PROFIBUS DP Fundamentals** - Types, Architecture
2. **Master/Slave** - DPM1/DPM2/Slave Roles
3. **GSD Files** - Device Description Files
4. **Baud Rate vs Distance** - Selection Guidelines
5. **Wiring and Termination** - Cable Specs, Connections
6. **ET200S Remote I/O** - Adding and Configuring
7. **VFD on PROFIBUS** - MM440 Telegram Configuration
8. **Diagnostics** - Status Bytes, TIA Portal
9. **Slot/Index Addressing** - DP-V1 Acyclic Services
10. **Protocol Comparison** - PROFIBUS vs DeviceNet vs CC-Link
11. **OB86 Handling** - Fault Organization Block
12. **Migration** - PROFIBUS → PROFINET Path

---

## Navigation
- [← Part 25: Modbus RTU/TCP](part25_modbus.md)
- [→ Part 27: PROFINET](part27_profinet.md)
- [กลับหน้าหลัก](../README.md)
