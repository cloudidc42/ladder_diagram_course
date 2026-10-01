# Part 25: Modbus RTU/TCP (Steps 481-500)

## ภาพรวม
Modbus เป็น Protocol ที่นิยมที่สุดในอุตสาหกรรม ออกแบบโดย Modicon ในปี 1979 และถูกใช้งานมาจนถึงปัจจุบัน เพราะมีความเรียบง่าย เปิดเผย (Open Standard) และ Device เกือบทุกชนิดรองรับ ไม่ว่าจะเป็น VFD, Temperature Controller, Flow Meter, Smart Sensor

---

## Step 481: Modbus Protocol Overview

```
┌──────────────────────────────────────────────────────────┐
│               Modbus Overview                            │
│                                                          │
│  Modbus Architecture:                                   │
│  - Master/Slave (Client/Server ใน Modbus TCP)           │
│  - มี Master 1 ตัว, Slave ได้ถึง 247 ตัว              │
│  - Master เป็นผู้ถาม (Request)                          │
│  - Slave เป็นผู้ตอบ (Response)                          │
│  - Slave ไม่ส่งข้อมูลเองโดยไม่ถูกถาม                   │
│                                                          │
│  Request/Response Pattern:                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │                                                  │   │
│  │  Master ──── Request ────► Slave                 │   │
│  │  Master ◄─── Response ─── Slave                 │   │
│  │                                                  │   │
│  │  Request: [Slave Addr][Function][Data][CRC]      │   │
│  │  Response:[Slave Addr][Function][Data][CRC]      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  Modbus Variants:                                       │
│  Modbus RTU: Serial (RS-232/RS-485), Binary Format      │
│  Modbus ASCII: Serial, ASCII Text Format (ช้ากว่า)      │
│  Modbus TCP: Ethernet, ไม่มี CRC (ใช้ TCP แทน)         │
└──────────────────────────────────────────────────────────┘
```

---

## Step 482: Modbus Register Types

```
┌──────────────────────────────────────────────────────────┐
│           Modbus Register Map                            │
│                                                          │
│  Address Range    Type              R/W   Bits/Words     │
│  ──────────────────────────────────────────────────────  │
│  00001 - 09999    Coils             R/W   1 bit          │
│  10001 - 19999    Discrete Inputs   R     1 bit          │
│  30001 - 39999    Input Registers   R     16 bit (word)  │
│  40001 - 49999    Holding Registers R/W   16 bit (word)  │
│  ──────────────────────────────────────────────────────  │
│                                                          │
│  1. COILS (0x อ้างอิง):                                 │
│     - Output bit, อ่านและเขียนได้                       │
│     - ตัวอย่าง: Motor ON/OFF Command                    │
│     - Address 00001 = Coil 1 (Protocol address 0x0000) │
│                                                          │
│  2. DISCRETE INPUTS (1x อ้างอิง):                       │
│     - Input bit, อ่านได้อย่างเดียว                      │
│     - ตัวอย่าง: Limit Switch Status                     │
│     - Address 10001 = Discrete Input 1 (0x0000)        │
│                                                          │
│  3. INPUT REGISTERS (3x อ้างอิง):                       │
│     - 16-bit Input Word, อ่านได้อย่างเดียว              │
│     - ตัวอย่าง: Current Temperature (actual value)      │
│     - Address 30001 = Input Register 1 (0x0000)        │
│                                                          │
│  4. HOLDING REGISTERS (4x อ้างอิง):                     │
│     - 16-bit Output Word, อ่านและเขียนได้               │
│     - ตัวอย่าง: Setpoint, Configuration                 │
│     - Address 40001 = Holding Register 1 (0x0000)      │
└──────────────────────────────────────────────────────────┘
```

---

## Step 483: Function Codes

```
┌──────────────────────────────────────────────────────────┐
│              Modbus Function Codes                       │
│                                                          │
│  FC 01: Read Coils                                      │
│  ── อ่าน Output bits (Coils)                            │
│  ── Request: [Addr][01][Start_Addr_H][Start_Addr_L]    │
│              [Qty_H][Qty_L][CRC_L][CRC_H]               │
│                                                          │
│  FC 02: Read Discrete Inputs                           │
│  ── อ่าน Input bits                                     │
│                                                          │
│  FC 03: Read Holding Registers                         │
│  ── อ่าน Output Words (ที่ใช้บ่อยที่สุด)               │
│  ── ตัวอย่าง: อ่าน Setpoint, Configuration            │
│                                                          │
│  FC 04: Read Input Registers                           │
│  ── อ่าน Input Words                                    │
│  ── ตัวอย่าง: อ่าน Temperature, Current              │
│                                                          │
│  FC 05: Write Single Coil                              │
│  ── เขียน 1 Output Bit                                  │
│  ── FF00h = ON, 0000h = OFF                            │
│                                                          │
│  FC 06: Write Single Holding Register                  │
│  ── เขียน 1 Output Word                                 │
│  ── ตัวอย่าง: เปลี่ยน Setpoint                         │
│                                                          │
│  FC 15: Write Multiple Coils                           │
│  ── เขียนหลาย Output Bits พร้อมกัน                      │
│                                                          │
│  FC 16: Write Multiple Holding Registers               │
│  ── เขียนหลาย Output Words พร้อมกัน                     │
│  ── ที่ใช้บ่อยสำหรับ Configuration                     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 484: Modbus RTU Frame Format

```
// Modbus RTU Frame (Binary)

// REQUEST: อ่าน 3 Holding Registers เริ่มจาก 40001 (Slave 1)
// ──────────────────────────────────────────────────────────

// Byte:    01    03    00 00   00 03   05 CB
//          │     │     │       │       │
//          │     │     Start   Qty     CRC
//          │     Function      Register Count
//          Slave Address       (Protocol = 0x0000)

// RTU Frame:
// [01][03][00 00][00 03][05 CB]
//  ↑   ↑    ↑       ↑     ↑
//  SA  FC  Start  Qty   CRC16

// RESPONSE from Slave:
// [01][03][06][00 10][00 20][00 30][54 52]
//  ↑   ↑   ↑    ↑       ↑       ↑    ↑
//  SA  FC Bytes Reg1  Reg2  Reg3  CRC16
//         Count  =16   =32   =48

// CRC16 Calculation (Modbus Standard):
// CRC-16/ARC: Poly=0x8005, Init=0xFFFF, RefIn=true

// ───────────────────────────────────────────────────────

// Timing Rules (RTU):
// Inter-character Delay: < 1.5 character times
// Frame Gap:            ≥ 3.5 character times
// Character Time at 9600 baud, 8N1 = 1/9600 × 10 = 1.04ms
// 3.5T = 3.5 × 1.04 = 3.64ms gap between frames
```

---

## Step 485: Modbus TCP Frame Format

```
// Modbus TCP Frame (เหมือน RTU แต่ไม่มี CRC แต่มี MBAP Header)

// MBAP Header (Modbus Application Protocol Header):
// [Transaction ID - 2 bytes][Protocol ID - 2 bytes]
// [Length - 2 bytes][Unit ID - 1 byte][PDU data...]

// ตัวอย่าง: อ่าน Holding Register 40001, 3 registers, Slave 1

// TCP Frame:
// [00 01][00 00][00 06][01][03][00 00][00 03]
//   ↑       ↑      ↑    ↑   ↑    ↑       ↑
//  Trans   Proto  Len  Unit FC Start    Qty
//  ID=1    =0    =6   ID=1  3  =0      =3

// Port: 502 (Standard Modbus TCP Port)

// ข้อดี Modbus TCP vs RTU:
// - ไม่ต้องคำนวณ CRC (TCP จัดการ Integrity)
// - ส่งได้หลาย Request พร้อมกัน (Transaction ID ต่างกัน)
// - ระยะทางไม่จำกัด (ผ่าน Ethernet)
// - รองรับหลาย Master พร้อมกัน
// - Firewall/VPN Friendly (Port 502)
```

---

## Step 486: Configuring Modbus Master ใน Siemens

```
// Siemens S7-1200 Modbus RTU Master
// ใช้: CM1241 Serial Module
//      MB_COMM_LOAD + MB_MASTER Instructions

// Step 1: Initialize Communication Port (First Scan)
NETWORK 1:
    ┌──[FIRST_SCAN]──────────────────────────────────────┐
    │                                                     │
    │  MB_COMM_LOAD                                      │
    │  REQ: FIRST_SCAN                                  │
    │  PORT: 270          // CM1241 Hardware Identifier  │
    │  BAUD: 19200                                       │
    │  PARITY: 0          // 0=None, 1=Odd, 2=Even      │
    │  FLOW_CTRL: 0       // 0=No Flow Control          │
    │  RTS_ON_DLY: 0ms                                  │
    │  RTS_OFF_DLY: 0ms                                 │
    │  RESP_TO: 1000ms    // Response Timeout           │
    │  MB_DB: "MB_Master_DB"  // IDB for MB_MASTER      │
    │  DONE → InitDone                                  │
    │  ERROR → InitError                                │
    └───────────────────────────────────────────────────┘

// Step 2: Read VFD Holding Registers (FC03)
// VFD Slave Address = 1
// Read 2 registers from 40001 (Output Frequency, Current)
NETWORK 2:
    ┌──[100ms_Clock]─────────────────────────────────────┐
    │                                                     │
    │  MB_MASTER  "MB_Master_DB"                         │
    │  REQ: 100ms_Clock                                  │
    │  MB_ADDR: 1         // Slave Address               │
    │  MODE: 0            // 0=Read, 1=Write             │
    │  DATA_ADDR: 40001   // Start at Holding Reg 1      │
    │  DATA_LEN: 2        // Read 2 Registers            │
    │  DATA_PTR: P#DB100.DBX0.0 WORD 2  // Buffer       │
    │  DONE → VFD_Read_Done                             │
    │  BUSY → VFD_Read_Busy                             │
    │  ERROR → VFD_Read_Error                           │
    │  STATUS → VFD_Read_Status                         │
    └───────────────────────────────────────────────────┘

// DB100 Structure:
// DB100.DBW0 = VFD Output Frequency (×0.01 Hz)
// DB100.DBW2 = VFD Output Current (×0.1 A)

// Step 3: Write VFD Speed Setpoint (FC06)
NETWORK 3:
    ┌──[Write_Trigger]───────────────────────────────────┐
    │                                                     │
    │  MB_MASTER  "MB_Master_DB"                         │
    │  REQ: Write_Trigger                                │
    │  MB_ADDR: 1                                        │
    │  MODE: 1            // 1=Write                     │
    │  DATA_ADDR: 40003   // Holding Register 3 (SP)     │
    │  DATA_LEN: 1        // Write 1 Register            │
    │  DATA_PTR: P#DB101.DBX0.0 WORD 1  // Source       │
    └───────────────────────────────────────────────────┘

// DB101.DBW0 = Frequency Setpoint (Hz × 100)
// เช่น 50Hz = 5000
```

---

## Step 487: Configuring Modbus Slave ใน Siemens

```
// Siemens S7-1200 Modbus RTU Slave
// ใช้: MB_COMM_LOAD + MB_SLAVE Instructions

// MB_SLAVE พารามิเตอร์:
// MB_ADDR: Slave Address (1-247)
// MB_HOLD_REG: Pointer to Holding Register area (DB)

// ตัวอย่าง: S7-1200 เป็น Slave สำหรับ SCADA

NETWORK 1: // Initialize
    ┌──[FIRST_SCAN]──────────────────────────────────────┐
    │  MB_COMM_LOAD (เหมือน Master แต่สำหรับ Slave Port) │
    └───────────────────────────────────────────────────┘

NETWORK 2: // Slave Operation (ทุก Scan)
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │                                                     │
    │  MB_SLAVE  "MB_Slave_DB"                           │
    │  MB_ADDR: 5         // This PLC = Slave 5          │
    │  MB_HOLD_REG: P#DB200.DBX0.0 WORD 50              │
    │  // DB200 สามารถ Read/Write ได้จาก Master          │
    │  NDR → SlaveNewData                               │
    │  DR → SlaveDataRequest                            │
    │  ERROR → SlaveError                               │
    └───────────────────────────────────────────────────┘

// DB200 Mapping:
// DB200.DBW0  = Holding Register 40001 (Status Word)
// DB200.DBW2  = Holding Register 40002 (Temperature)
// DB200.DBW4  = Holding Register 40003 (SP from SCADA)
// ...และต่อๆไป
```

---

## Step 488: Configuring Modbus ใน Allen-Bradley

```
// Allen-Bradley ControlLogix - Modbus TCP (Client)
// ใช้ MSG Instruction with Modbus TCP

// 1. ใน RSLogix 5000/Studio 5000:
//    Communication → Ethernet Module (Generic)
//    IP Address: 192.168.1.100 (Slave Device)
//    Connection Type: None (ใช้ MSG)
//    Assembly Instance: ไม่ต้องตั้ง

// 2. สร้าง MSG Configuration:
RUNG 1:
    ┌──[Read_Trigger]────────────────────────────────────┐
    │                                                     │
    │  MSG                                               │
    │  Message Control: Modbus_Read_Msg                  │
    │  (Config ใน Properties:)                           │
    │  Message Type: Modbus Read Holding Registers (FC3) │
    │  Unit ID: 1          // Slave ID                   │
    │  Register Start: 40001                             │
    │  Register Count: 5                                 │
    │  Destination Tag: VFD_Data[0]                     │
    │  Communication Path: "VFD_Ethernet"               │
    └───────────────────────────────────────────────────┘

// 3. สำหรับ Serial (Modbus RTU ผ่าน 1756-CP3):
// MSG Type: Modbus Serial Read/Write
// Path: 1,4 (Serial Module ใน Slot 4)

// Allen-Bradley MicroLogix - Modbus RTU via Channel 0
// Channel 0 Protocol: DF1 Half-Duplex → เปลี่ยนเป็น ASCII
// หรือใช้ 1761-NET-AIC (Modbus to DF1 Converter)
```

---

## Step 489: Configuring Modbus ใน Mitsubishi

```
// Mitsubishi FX5U - Modbus RTU Master via RS-485
// ใช้ FX5-485-BD หรือ FX5-485ADP

// GX Works3 → Module Parameter → RS-485 → Predefined Protocol
// Protocol: Modbus RTU
// Baud Rate: 19200
// Parity: None
// Data Length: 8
// Stop Bits: 1

// ADPRW Instruction (Advanced Predefined Protocol):
// SP.ADPRW คำสั่งใน Ladder Instruction

// ตัวอย่าง: อ่าน Holding Registers จาก VFD (Slave 1)
LD      M0              // Read Trigger
SP.ADPRW  K3  K1        // Channel 3, Predefined Protocol Num 1
         K1  K0         // Slave Address 1, Response Wait
         K3  K40001     // Function 3, Start Register 40001
         K5  D200       // Read 5 registers, Store in D200-D204

// ผลลัพธ์:
// D200 = Register 40001 (Frequency Command)
// D201 = Register 40002 (Output Frequency)
// D202 = Register 40003 (Output Current)
// D203 = Register 40004 (Output Voltage)
// D204 = Register 40005 (Status Word)

// เขียน Frequency Setpoint:
LD      M1              // Write Trigger
MOV     K5000 D300      // 50Hz = 5000
SP.ADPRW  K3  K1
         K1  K0
         K6  K40001     // Function 6, Write Register 40001
         K1  D300       // Write 1 register from D300
```

---

## Step 490: อ่าน VFD ผ่าน Modbus RTU

### ตัวอย่าง: Mitsubishi FR-E800 VFD Register Map

```
┌──────────────────────────────────────────────────────────┐
│        Mitsubishi FR-E800 Modbus Register Map            │
│                                                          │
│  Holding Registers (Read/Write):                        │
│  40001: Frequency Command (0-12000 = 0.00-120.00 Hz)    │
│  40002: Operation Command (Bit 0=Run, Bit 1=Rev)        │
│  40003: Speed Control Mode (0=Freq, 1=Torque)           │
│                                                          │
│  Input Registers (Read Only):                           │
│  30001: Output Frequency (×0.01 Hz)                     │
│  30002: Output Current (×0.01 A)                        │
│  30003: Output Voltage (V)                              │
│  30004: Fault Code (0=No Fault)                         │
│  30005: Status Word                                     │
│          Bit 0: Running                                  │
│          Bit 1: Forward                                  │
│          Bit 2: Reverse                                  │
│          Bit 3: Fault                                    │
│          Bit 4: At Speed                                 │
│                                                          │
│  Coils (Write):                                         │
│  0x0001: Run Command (1=Run, 0=Stop)                    │
│  0x0002: Reset Fault                                    │
└──────────────────────────────────────────────────────────┘
```

### โค้ด Siemens - อ่านและควบคุม VFD:

```
// DB_VFD: VFD Data
// DB_VFD.OutputFreq (REAL)   = Output Frequency Hz
// DB_VFD.OutputCurrent (REAL) = Output Current A
// DB_VFD.Fault (BOOL)         = Fault Status
// DB_VFD.Running (BOOL)        = Running Status
// DB_VFD.SpeedRef (REAL)      = Speed Reference Hz (Setpoint)
// DB_VFD.StartCmd (BOOL)      = Start Command
// DB_VFD.StopCmd (BOOL)       = Stop Command

NETWORK 1: // อ่าน VFD Status (ทุก 200ms)
    ┌──[200ms_Clock]─────────────────────────────────────┐
    │  MB_MASTER  "MB_Master_DB"                         │
    │  REQ: 200ms_Clock                                  │
    │  MB_ADDR: 5         // VFD Slave Address           │
    │  MODE: 4            // 4=Read Input Registers (FC4)│
    │  DATA_ADDR: 30001   // Start at Input Reg 1        │
    │  DATA_LEN: 5        // Read 5 Registers            │
    │  DATA_PTR: P#DB_VFD_Raw.DBX0.0 WORD 5            │
    └───────────────────────────────────────────────────┘

NETWORK 2: // Scale Frequency (Raw ÷ 100 = Hz)
    ┌──[MB_MASTER_Done]──────────────────────────────────┐
    │  INT_TO_REAL: DB_VFD_Raw.DBW0 → Freq_Real         │
    │  DIV: Freq_Real / 100.0 → DB_VFD.OutputFreq       │
    └───────────────────────────────────────────────────┘

NETWORK 3: // Extract Status Bits
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  AND: DB_VFD_Raw.DBW8 AND 2#0000000000001000       │
    │  → Fault_Mask                                      │
    │  CMP>: Fault_Mask > 0 → DB_VFD.Fault              │
    └───────────────────────────────────────────────────┘

NETWORK 4: // เขียน Speed Setpoint (FC6)
    ┌──[Speed_Write_Trigger]─────────────────────────────┐
    │  // Convert Hz to Raw (Hz × 100)                  │
    │  MUL: DB_VFD.SpeedRef × 100.0 → Speed_Raw_Real   │
    │  REAL_TO_INT: Speed_Raw_Real → DB_Cmd.DBW0        │
    │                                                    │
    │  MB_MASTER  "MB_Master_DB"                        │
    │  MODE: 1            // Write                       │
    │  DATA_ADDR: 40001                                  │
    │  DATA_LEN: 1                                       │
    │  DATA_PTR: P#DB_Cmd.DBX0.0 WORD 1                │
    └───────────────────────────────────────────────────┘

NETWORK 5: // Start VFD (FC5 Write Coil)
    ┌──[DB_VFD.StartCmd ONS]─────────────────────────────┐
    │  MB_MASTER  "MB_Master_DB"                         │
    │  MODE: 5           // FC5 Write Single Coil        │
    │  DATA_ADDR: 1      // Coil 00001                   │
    │  DATA_LEN: 1                                       │
    │  DATA_PTR: P#True  // TRUE = ON                   │
    └───────────────────────────────────────────────────┘
```

---

## Step 491: Modbus Exception Responses

```
┌──────────────────────────────────────────────────────────┐
│           Modbus Exception Codes                         │
│                                                          │
│  ถ้า Slave ไม่สามารถตอบสนองตามที่ขอได้:                │
│  จะส่ง Exception Response กลับมา                        │
│                                                          │
│  Exception Response:                                    │
│  [Slave Addr][FC + 0x80][Exception Code][CRC]           │
│  ตัวอย่าง: FC03 Exception → Function Code = 0x83       │
│                                                          │
│  Exception Codes:                                       │
│  ──────────────────────────────────────────────────     │
│  01: Illegal Function                                   │
│      Slave ไม่รู้จัก Function Code นั้น                 │
│                                                          │
│  02: Illegal Data Address                               │
│      ที่อยู่ Register ที่ขอไม่มีใน Slave                 │
│                                                          │
│  03: Illegal Data Value                                 │
│      ค่าที่ส่งไปไม่ถูกต้อง (เช่น Coil ส่งค่าที่ไม่ใช่ FF00/0000) │
│                                                          │
│  04: Slave Device Failure                               │
│      Slave มีข้อผิดพลาดภายใน                            │
│                                                          │
│  06: Slave Device Busy                                  │
│      Slave กำลังทำงานอื่น ไม่พร้อม Process Request     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 492: Modbus Timing และ Performance

```
// ปัจจัยที่ส่งผลต่อ Modbus RTU Performance

// 1. Baud Rate:
//    9600 bps:  1 frame read (10 bytes) = 10.4ms
//    19200 bps: 1 frame read           = 5.2ms
//    115200 bps:1 frame read           = 0.87ms

// 2. Number of Slaves:
//    ถ้ามี 10 Slaves อ่านทีละตัว:
//    Cycle Time = 10 × (Request + Response + Gap)
//    เช่น: 10 × (5ms + 5ms + 4ms) = 140ms per cycle

// 3. Register Count:
//    อ่านทีละ 1 reg = เสียเวลา overhead มาก
//    อ่านทีละ 20 regs พร้อมกัน = มีประสิทธิภาพกว่า

// 4. Timeout Setting:
//    Timeout ควรตั้ง = Response_Time × 2-3
//    เพื่อรองรับ Occasional Delay

// Calculation Example:
// บาง VFD ต้องการ 3ms ก่อนตอบ
// Frame size = 100 bytes = 52ms at 19200
// Timeout = max(3ms, 52ms) × 2 = 104ms

// Poll Rate Design:
// VFD Status: ทุก 100ms (Fast, ต้องการ Real-time)
// Energy Meter: ทุก 1s (Medium)
// Alarm Status: ทุก 500ms
// Configuration: เฉพาะเมื่อต้องการ (On Demand)
```

---

## Step 493: Modbus TCP Configuration (Siemens)

```
// Siemens S7-1200 Modbus TCP Client (ไม่ต้องการ CM1241)
// ใช้ Built-in Ethernet Port

// Instructions: MB_CLIENT (Modbus TCP Client)
//              MB_SERVER (Modbus TCP Server)

NETWORK 1: // Modbus TCP Client
    ┌──[100ms_Clock]─────────────────────────────────────┐
    │                                                     │
    │  MB_CLIENT  "MB_Client_DB"                         │
    │  REQ: 100ms_Clock                                  │
    │  DISCONNECT: FALSE                                 │
    │  MB_ADDR: 1           // Unit ID ใน MBAP Header    │
    │  MODE: 0              // 0=Read                    │
    │  DATA_ADDR: 40001     // Holding Register          │
    │  DATA_LEN: 10         // Read 10 Registers         │
    │  DATA_PTR: P#DB_Recv.DBX0.0 WORD 10              │
    │  CONNECT: P#DB_Conn.DBX0.0  // Connection Config   │
    │  CONNECTED → Connected_Status                     │
    │  DONE → Read_Done                                 │
    │  ERROR → Read_Error                               │
    └───────────────────────────────────────────────────┘

// DB_Conn Structure (TCON_IP_v4):
// InterfaceId: 64  (PROFINET Interface)
// ID: W#16#0001   (Connection ID)
// ConnectionType: B#16#0B  (TCP)
// ActiveEstablished: TRUE   (Client initiates connection)
// RemoteAddress: IP 192.168.1.100
// RemotePort: 502   (Modbus TCP Port)
```

---

## Step 494: Modbus Diagnostics

```
// การ Diagnose ปัญหา Modbus

// 1. ตรวจสอบด้วย Modbus Poll/Slave Software:
// ─────────────────────────────────────────────
// ลง Modbus Poll บน PC
// ต่อ USB-RS485 Converter
// ตั้ง Connection: 19200-8-N-1
// ตั้ง Slave Address, Register, Function
// ดู Response: ถ้ามีข้อมูลกลับมา = สายและ Config ถูก

// 2. Siemens TIA Portal - Communication Diagnostics:
// ─────────────────────────────────────────────
// Online & Diagnostics → Module Status
// ดู MB_MASTER/MB_SLAVE Error Codes:
//
// Status = 0000: No error
// Status = 7001: Busy
// Status = 80Cx: Communication Error
// 80C1: No response received (Timeout)
// 80C2: Framing error
// 80C3: CRC error
// 80D1: Function not supported (by slave)
// 80D2: Invalid address
// 80D3: Invalid data value

// 3. LED Indicators:
// CM1241 Module LEDs:
// TX LED: กะพริบเมื่อส่งข้อมูล
// RX LED: กะพริบเมื่อรับข้อมูล
// ถ้า TX กะพริบแต่ RX ไม่กะพริบ = Slave ไม่ตอบ

// 4. Oscilloscope / Logic Analyzer:
// วัดสัญญาณ RS-485 (A และ B)
// ดู Frame Timing และ Voltage Levels
```

---

## Step 495: ตัวอย่างสมบูรณ์ - Modbus RTU Network

```
// ─────────────────────────────────────────────────────
// System: 
// - Siemens S7-1200 (Modbus Master)
// - Slave 1: Mitsubishi FR-D800 VFD
// - Slave 2: Yokogawa Temperature Controller
// - Slave 3: Honeywell Level Transmitter (4-20mA + Modbus)
//
// All on RS-485 at 19200-8-N-1
// ─────────────────────────────────────────────────────

// PROGRAM STRUCTURE:
//
// OB1 (Main)
//   ├── FC1: Comm_Manager    (จัดการลำดับการสื่อสาร)
//   ├── FC2: VFD_Control     (ควบคุม VFD)
//   ├── FC3: Temp_Monitor    (ตรวจสอบอุณหภูมิ)
//   └── FC4: Level_Monitor   (ตรวจสอบระดับ)
//
// FC1: Comm_Manager
// ใช้ Round-Robin เพื่อถามทีละ Slave

// ─────────────────────────────────────────────────────
// Communication Schedule:
// Tick 1 (100ms): อ่าน VFD Status (Slave 1)
// Tick 2 (100ms): เขียน VFD Command ถ้าจำเป็น
// Tick 3 (100ms): อ่าน Temp Controller (Slave 2)
// Tick 4 (100ms): อ่าน Level Transmitter (Slave 3)
// วน Tick 1-4 ซ้ำ → Cycle = 400ms
// ─────────────────────────────────────────────────────

// FC1: Communication Manager
VAR:
  Comm_State (INT) = 0
  Comm_Timer (IEC_Timer)
  VFD_Done, Temp_Done, Level_Done (BOOL)

NETWORK 1: // State Machine สำหรับ Round-Robin
    ┌──[Comm_State EQU 0]──[NOT MB_MASTER.BUSY]──────────┐
    │  State 0: Read VFD                                  │
    └──┬──────────────────────────────────────────────────┘
       └──[MB_MASTER Read Slave1, Regs 30001-30005]───────┤
          [MOV 1 → Comm_State]

    ┌──[Comm_State EQU 1]──[MB_MASTER.DONE]──────────────┐
    │  VFD Read Complete                                  │
    └──┬──────────────────────────────────────────────────┘
       └──[Process VFD Data]──────────────────────────────┤
          [MOV 2 → Comm_State]

    ┌──[Comm_State EQU 2]──[NOT MB_MASTER.BUSY]──────────┐
    │  State 2: Read Temp Controller                      │
    └──┬──────────────────────────────────────────────────┘
       └──[MB_MASTER Read Slave2, Regs 30001-30003]───────┤
          [MOV 3 → Comm_State]

    // ... ต่อ State 3 (Read Level), State 4 (Write if needed)
```

---

## Step 496: Modbus TCP Server ใน PLC

```
// Siemens S7-1200 Modbus TCP Server
// ให้ SCADA/HMI อ่านข้อมูลจาก PLC ผ่าน Modbus TCP

NETWORK 1: // Server Operation (ทำงานทุก Scan)
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │                                                     │
    │  MB_SERVER  "MB_Server_DB"                         │
    │  DISCONNECT: FALSE                                 │
    │  MB_HOLD_REG: P#DB_SCADA.DBX0.0 WORD 100         │
    │  // DB_SCADA เก็บข้อมูลที่ SCADA จะอ่าน/เขียน   │
    │  CONNECT: "Server_Connect_DB"                     │
    │  CONNECTED → SCADA_Connected                      │
    │  NDR → SCADA_NewData                             │
    │  DR → SCADA_DataRequest                          │
    └───────────────────────────────────────────────────┘

// DB_SCADA Register Mapping:
// Offset DBW0  = Reg 40001 = System Status Word
// Offset DBW2  = Reg 40002 = Temperature (×10)
// Offset DBW4  = Reg 40003 = VFD Frequency (×100)
// Offset DBW6  = Reg 40004 = Tank Level (%)
// Offset DBW8  = Reg 40005 = SCADA Setpoint (×10)  [Writeable]
// Offset DBW10 = Reg 40006 = SCADA Command          [Writeable]
```

---

## Step 497: Multi-Master Modbus TCP

```
// Modbus TCP รองรับหลาย Master พร้อมกัน
// ต่างจาก RTU ที่มีได้ Master เดียว

// ตัวอย่าง:
// SCADA อ่าน Register ทุก 5 วินาที
// HMI อ่าน Register ทุก 1 วินาที
// MES อ่าน Register ทุก 30 วินาที
// ทุก Client ต่อ Port 502 ของ PLC พร้อมกัน

// Siemens S7-1200 รองรับ MB_SERVER Connections สูงสุด 16 ตัว
// (ขึ้นอยู่กับ CPU รุ่น)

// Connection Management:
// แต่ละ MB_SERVER Instance รองรับ 1 Client
// ต้องการ 3 Client → ต้องมี 3 MB_SERVER Instances
// แต่ละ Instance ใช้ DB แยกกัน (Connection DB)

// ตัวอย่าง: 2 SCADA Client + 1 HMI

NETWORK 1: // MB_SERVER Instance 1 (SCADA 1)
    ┌──[ALWAYS_ON]──(MB_SERVER "MB_Srv1_DB" Connect:"Conn1")─┐
    └──────────────────────────────────────────────────────────┘

NETWORK 2: // MB_SERVER Instance 2 (SCADA 2)
    ┌──[ALWAYS_ON]──(MB_SERVER "MB_Srv2_DB" Connect:"Conn2")─┐
    └──────────────────────────────────────────────────────────┘

NETWORK 3: // MB_SERVER Instance 3 (HMI)
    ┌──[ALWAYS_ON]──(MB_SERVER "MB_Srv3_DB" Connect:"Conn3")─┐
    └──────────────────────────────────────────────────────────┘
```

---

## Step 498: Modbus Best Practices

```
┌──────────────────────────────────────────────────────────┐
│           Modbus Best Practices                          │
│                                                          │
│  ADDRESSING:                                            │
│  1. ใช้ Document Register Map เสมอ                      │
│  2. ระวัง Off-by-One Error                              │
│     Protocol Address = User Address - 1                 │
│     Register 40001 = Protocol Address 0x0000            │
│  3. ตรวจสอบ Endianness: Big Endian หรือ Little Endian   │
│     Modbus Standard = Big Endian (MSB First)           │
│                                                          │
│  PERFORMANCE:                                           │
│  1. อ่าน Register ต่อเนื่องกัน ไม่แยกกัน               │
│     ✓ อ่าน Reg 40001-40010 ใน 1 Request               │
│     ✗ อ่านแยก 10 Request ทีละ 1 Register               │
│  2. ตั้ง Timeout ให้เหมาะสม (ไม่น้อยเกินไป)            │
│  3. ตั้ง Poll Rate ตามความต้องการจริง                   │
│                                                          │
│  RELIABILITY:                                           │
│  1. มี Timeout Detection และ Retry Logic               │
│  2. Log Communication Errors                           │
│  3. ใช้ Watchdog สำหรับ Critical Data                  │
│  4. ทดสอบ Network Failure Scenario                     │
│                                                          │
│  DOCUMENTATION:                                         │
│  1. Document ทุก Register ที่ใช้งาน                    │
│  2. Version Register Map พร้อม Date                    │
│  3. ระบุ Scale Factor และ Unit สำหรับทุก Register       │
└──────────────────────────────────────────────────────────┘
```

---

## Step 499: ตัวอย่าง Register Map Documentation

```
┌──────────────────────────────────────────────────────────┐
│  Example: Register Map for Conveyor System               │
│  PLC: Siemens S7-1200, Modbus TCP Server                │
│  IP: 192.168.1.10, Port: 502, Unit ID: 1               │
│  Date: 2024-01-15, Version: 1.2                        │
├──────┬────────┬──────────────────────┬──────┬──────────┤
│  Reg │  FC    │  Description         │  R/W │  Units   │
├──────┼────────┼──────────────────────┼──────┼──────────┤
│40001 │  3,16  │  System Status Word  │  R   │  Bits    │
│      │        │  Bit0: System Ready  │      │          │
│      │        │  Bit1: Running       │      │          │
│      │        │  Bit2: Fault         │      │          │
│      │        │  Bit3: E-Stop        │      │          │
├──────┼────────┼──────────────────────┼──────┼──────────┤
│40002 │  3,16  │  Conveyor Speed      │  R   │  m/min×10│
│      │        │  Range: 0-1000       │      │  (0-100) │
├──────┼────────┼──────────────────────┼──────┼──────────┤
│40003 │  3,16  │  Product Count       │  R   │  pieces  │
│      │        │  Resets each shift   │      │          │
├──────┼────────┼──────────────────────┼──────┼──────────┤
│40004 │  3,16  │  Temperature         │  R   │  °C×10   │
│      │        │  Range: 0-3000       │      │  (0-300) │
├──────┼────────┼──────────────────────┼──────┼──────────┤
│40010 │  3,16  │  Speed Setpoint      │  R/W │  m/min×10│
│      │        │  Write to change     │      │  (0-1000)│
├──────┼────────┼──────────────────────┼──────┼──────────┤
│40011 │  3,16  │  Command Register    │  R/W │  Bits    │
│      │        │  Bit0: Start (Write) │      │          │
│      │        │  Bit1: Stop (Write)  │      │          │
│      │        │  Bit2: Reset Fault   │      │          │
└──────┴────────┴──────────────────────┴──────┴──────────┘
```

---

## Step 500: Lab Project - Complete Modbus System

**โจทย์:** สร้างระบบ Modbus สมบูรณ์

```
┌──────────────────────────────────────────────────────────┐
│           Complete Modbus Lab Project                    │
│                                                          │
│  Hardware Requirements:                                 │
│  1. Siemens S7-1200 + CM1241 RS-485 Module             │
│  2. Mitsubishi FR-D800 VFD (Modbus Slave)              │
│  3. PC with Modbus Poll Software                        │
│  4. RS-485 Cable with Termination                       │
│                                                          │
│  Software Requirements:                                 │
│  1. Siemens TIA Portal V16+                            │
│  2. Modbus Poll (for testing)                           │
│                                                          │
│  Tasks:                                                 │
│  1. Configure VFD for Modbus RTU (Slave 1, 19200-8-N-1)│
│  2. Configure S7-1200 CM1241 Module                     │
│  3. Write PLC Program:                                  │
│     a. MB_COMM_LOAD for initialization                  │
│     b. Read VFD Input Registers (30001-30005)           │
│     c. Scale Output Frequency to Hz                     │
│     d. Write Speed Setpoint (FC6, Register 40001)       │
│     e. Write Run/Stop Command (FC5, Coil 00001)        │
│     f. Error Handling and Watchdog                      │
│  4. Test with Modbus Poll:                             │
│     a. Read same registers independently               │
│     b. Verify data matches                              │
│  5. Add Modbus TCP Server for SCADA connection          │
│  6. Document Register Map                               │
│                                                          │
│  Bonus: เพิ่ม Slave 2 (Temperature Controller)          │
│         และเขียน Round-Robin Communication Manager      │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 25

ใน Part นี้ได้เรียนรู้:

1. **Modbus Overview** - Master/Slave Architecture
2. **Register Types** - Coils, Discrete, Input, Holding
3. **Function Codes** - FC01-FC16
4. **RTU Frame Format** - Binary, CRC16
5. **TCP Frame Format** - MBAP Header
6. **Siemens Configuration** - MB_COMM_LOAD, MB_MASTER, MB_SLAVE
7. **Allen-Bradley Configuration** - MSG Instruction
8. **Mitsubishi Configuration** - SP.ADPRW
9. **VFD Control via Modbus** - Read/Write Examples
10. **Exception Handling** - Error Codes
11. **Performance** - Timing, Optimization
12. **Best Practices** - Documentation, Reliability

---

## Navigation
- [← Part 24: Communication Protocols Overview](part24_communication.md)
- [→ Part 26: PROFIBUS DP](part26_profibus.md)
- [กลับหน้าหลัก](../README.md)
