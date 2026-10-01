# Part 09: Data Move Instructions
## Steps 151-170: การจัดการข้อมูลใน PLC

---

## สารบัญ

- [Step 151: ภาพรวม Data Move Instructions](#step-151)
- [Step 152: MOV - Move Single Value](#step-152)
- [Step 153: BMOV - Block Move](#step-153)
- [Step 154: FILL - Fill Memory](#step-154)
- [Step 155: SWAP - Byte/Word Swap](#step-155)
- [Step 156: Indirect Addressing](#step-156)
- [Step 157: Data Arrays](#step-157)
- [Step 158: Shift Registers](#step-158)
- [Step 159: FIFO - First In, First Out](#step-159)
- [Step 160: LIFO - Last In, First Out](#step-160)
- [Step 161: Recipe Data Management](#step-161)
- [Step 162: Historical Data Logging](#step-162)
- [Step 163: Data Tables และ Look-up Tables](#step-163)
- [Step 164: String Operations](#step-164)
- [Step 165: Data Packing/Unpacking](#step-165)
- [Step 166: Circular Buffer](#step-166)
- [Step 167: Data Sorting](#step-167)
- [Step 168: Checksum และ Data Validation](#step-168)
- [Step 169: Communication Buffers](#step-169)
- [Step 170: Workshop - Complete Data Management System](#step-170)

---

## Step 151: ภาพรวม Data Move Instructions {#step-151}

### ทำไมต้องจัดการข้อมูลใน PLC?

```
PLC ไม่ได้ทำแค่ Logic ON/OFF:
  - เก็บประวัติการผลิต (Production History)
  - จัดการ Recipe (สูตรการผลิต)
  - Track ชิ้นงาน (Object Tracking)
  - Buffer ข้อมูลก่อนส่งไป SCADA
  - Shift Register สำหรับ Conveyor Tracking
```

### Data Move Instruction Set

```
┌──────────────┬────────────────────────────────────────────────────┐
│ Instruction  │ Function                                           │
├──────────────┼────────────────────────────────────────────────────┤
│ MOV          │ Copy single word: D[Dest] = D[Src]                 │
│ MOVP         │ MOV on Pulse (Edge triggered)                      │
│ BMOV         │ Block Move: Copy N words D[S] → D[D]              │
│ FILL         │ Fill N words with constant                         │
│ SWAP         │ Swap high/low bytes in a word                      │
│ XCH          │ Exchange 2 words                                   │
│ FMOV         │ Fill Move: Copy 1 value to N destinations          │
└──────────────┴────────────────────────────────────────────────────┘
```

### Register Addressing Modes

```
┌────────────────────┬──────────────────────────────────────────────┐
│ Mode               │ ตัวอย่าง (Mitsubishi)                        │
├────────────────────┼──────────────────────────────────────────────┤
│ Direct             │ D100 → ที่อยู่ D100 โดยตรง                   │
│ Indirect (Z-index) │ D100Z0 → D[100 + Z0] (pointer)              │
│ Constant           │ K100 → ค่าคงที่ 100                          │
│ Hex Constant       │ H1A3F → Hex ค่าคงที่                         │
└────────────────────┴──────────────────────────────────────────────┘
```

---

## Step 152: MOV - Move Single Value {#step-152}

### MOV Syntax

```
Mitsubishi FX:
  MOV K100 D0        ← D0 = 100 (constant)
  MOV D0 D100        ← D100 = D0 (register to register)
  MOV D0 Y0          ← Y0 = D0 (register to output)

Siemens S7:
  MOVE: IN: source, OUT: destination
  หรือ: %MW100 := %MW200

Allen-Bradley SLC-500:
  MOV  Source: N7:0 / Dest: N7:10
  COP  (Copy File - เหมือน BMOV)
```

### ตัวอย่าง 1: Load Default Values เมื่อ Startup

```
Network 1: Load Defaults on First Scan

     M8002                              ← First Scan
  ───┤ ├────[MOV K200 D100]─            ← Temperature SP = 200 (20.0°C)
            [MOV K500 D101]─            ← Pressure SP = 500 (50.0 kPa)
            [MOV K100 D102]─            ← Flow SP = 100 (100 L/min)
            [MOV K1 D110]─              ← Mode = 1 (Auto)
```

### ตัวอย่าง 2: Copy Actual เป็น Setpoint (Manual Mode)

```
สถานการณ์: เมื่อสลับจาก Auto เป็น Manual ให้ Setpoint = Actual
(Bumpless Transfer)

Network 1: ตรวจจับ Edge Auto→Manual
     I0.4           M250.0          M250.0          M251.0
  ───┤/├─────────────┤/├──────────────( )────         ( )───
  Not_Auto       Prev_Mode          Curr          Auto-to-Manual Pulse

Network 2: Update Prev Mode
     I0.4              M250.0
  ───┤/├────────────────( )───

Network 3: เมื่อสลับเป็น Manual → Copy Actual → Setpoint
     M251.0
  ───┤ ├────[MOV D10 D12]─           ← Temp_SP = Temp_Actual
  Auto→Manual   [MOV D20 D22]─       ← Pres_SP = Pres_Actual
  Pulse         [MOV D30 D32]─       ← Flow_SP = Flow_Actual
```

### ตัวอย่าง 3: Output Multiplexing

```
สถานการณ์: ส่ง Value ต่างๆ ไปยัง Display เดียว (Multiplexed)

  Display_Value = D200 (ส่งค่านี้ไป HMI Display)
  Select = D201 (0=Temp, 1=Pressure, 2=Flow)


Network 1: เลือก Temperature
     [= D201 K0]
  ─────┤ ├────[MOV D10 D200]─    ← แสดง Temperature

Network 2: เลือก Pressure
     [= D201 K1]
  ─────┤ ├────[MOV D20 D200]─    ← แสดง Pressure

Network 3: เลือก Flow
     [= D201 K2]
  ─────┤ ├────[MOV D30 D200]─    ← แสดง Flow
```

---

## Step 153: BMOV - Block Move {#step-153}

### BMOV คืออะไร?

```
BMOV = Block Move คัดลอกหลาย register พร้อมกัน

Syntax (Mitsubishi):
  BMOV D0 D100 K10    ← Copy D0-D9 ไป D100-D109 (10 words)

Syntax (Allen-Bradley):
  COP  Source: N7:0  Dest: N7:100  Length: 10

Syntax (Siemens):
  BLKMOV หรือ MOVE block ใน TIA Portal
```

### ตัวอย่าง 1: Backup Recipe Data

```
สถานการณ์: สำรองข้อมูล Recipe ก่อนเปลี่ยน Recipe ใหม่

  Current Recipe: D200-D219 (20 words)
  Backup Area:    D500-D519


Network 1: Backup Recipe เมื่อกด Save
     I10.0
  ───┤ ├────[BMOV D200 D500 K20]─   ← Copy D200-D219 → D500-D519
  Save_Button
```

### ตัวอย่าง 2: Production Report Copy

```
สถานการณ์: Copy ข้อมูล Shift ปัจจุบันไปยัง Historical Area

  Current Shift Data: D300-D349 (50 words)
  Shift History:
    Shift 1: D1000-D1049
    Shift 2: D1050-D1099
    Shift 3: D1100-D1149

  Shift_Number = D2 (1, 2, หรือ 3)


Network 1: คำนวณ Destination Address
  (ใช้ Index Register Z0)

     M8000
  ───┤ ├────[SUB D2 K1 Z0]─         ← Z0 = Shift_Number - 1
            [MUL Z0 K50 Z0]─        ← Z0 = (N-1) × 50

Network 2: Copy ข้อมูล Shift
     I15.5
  ───┤ ├────[BMOV D300 D1000Z0 K50]─ ← Copy to D[1000+Z0]
  Shift_End
```

### ตัวอย่าง 3: ล้างข้อมูลเป็น Block

```
Network 1: ล้างข้อมูลทั้งหมด (Clear)

     I0.9
  ───┤ ├────[BMOV D9999 D300 K50]─   ← ถ้า D9999 = 0 → ล้าง D300-D349
  Clear_Btn                            (วิธีที่ดีกว่าคือใช้ FILL)
```

---

## Step 154: FILL - Fill Memory {#step-154}

### FILL คืออะไร?

```
FILL = เติมค่าเดียวไปยัง N addresses ติดกัน

Syntax (Mitsubishi):
  FMOV K0 D100 K50    ← D100-D149 = 0 (50 words)
  FMOV K255 D200 K10  ← D200-D209 = 255

Syntax (Allen-Bradley):
  FLL  Source: 0  Dest: N7:100  Length: 50

Syntax (Siemens):
  FILL_BLK  IN: source, COUNT: n, OUT: dest
```

### ตัวอย่าง 1: Initialize Data Area

```
Network 1: Initialize ทุก Register เป็น 0 ตอน Startup

     M8002
  ───┤ ├────[FMOV K0 D0 K200]─       ← Clear D0-D199 ทั้งหมด
  First_Scan [FMOV K0 M0 K64]─       ← Clear M0-M63 ทั้งหมด
```

### ตัวอย่าง 2: Fill Pattern

```
Network 1: เตรียม Test Pattern (สำหรับ Debug)

     I0.9            I0.8
  ───┤ ├──────────────┤ ├────[FMOV K5555 D0 K100]─ ← Fill D0-D99 with 5555
  Debug_Mode      Test_Pattern
```

### ตัวอย่าง 3: ล้าง Buffer ก่อนเริ่ม Batch

```
Network 1: ล้าง Production Buffer ก่อน Batch ใหม่

     I5.7
  ───┤ ├────[FMOV K0 D300 K100]─     ← Clear Batch Data Buffer
  New_Batch  [MOV K0 D200]─          ← Clear Count
             [MOV K0 D201]─          ← Clear Total Weight
```

---

## Step 155: SWAP - Byte/Word Swap {#step-155}

### SWAP คืออะไร?

```
สำหรับ 16-bit Word:
SWAP สลับ High Byte กับ Low Byte

ก่อน SWAP: D0 = H1234 (High=12, Low=34)
หลัง SWAP: D0 = H3412 (High=34, Low=12)

ใช้เมื่อ:
- สื่อสารกับอุปกรณ์ต่างระบบ (Endian conversion)
- แยก High/Low byte ของ Word
- Modbus Communication (บางอุปกรณ์ใช้ Big Endian ต่างกัน)
```

### ตัวอย่าง: Modbus Communication

```
สถานการณ์: อ่านค่าจาก Modbus Device ที่ใช้ Big Endian
PLC ส่ง: H0001 = Register 1
Modbus ตอบ: H4248 (Temperature = 42.48°C ใน Big Endian)
ต้อง SWAP เพื่อให้ถูกต้อง

Network 1:
     M300.0
  ───┤ ├────[SWAP D500]─              ← สลับ Bytes ใน D500
  Data_Received
```

### DWORD Byte Swap

```
สำหรับ 32-bit DWORD ต้องทำ 2 ขั้นตอน:

ก่อน: D0(High) D1(Low) = H12345678
หลัง: D0(Low)  D1(High) = H56781234 (4-byte swap)

Network 1: Swap D0
     M8000
  ───┤ ├────[SWAP D0]─    ← D0: H1234 → H3412

Network 2: Swap D1
     M8000
  ───┤ ├────[SWAP D1]─    ← D1: H5678 → H7856

Network 3: Exchange D0 and D1
     M8000
  ───┤ ├────[XCH D0 D1]─  ← สลับ D0 กับ D1 ทั้ง word
```

---

## Step 156: Indirect Addressing {#step-156}

### Indirect Addressing คืออะไร?

```
Direct Addressing:
  MOV D100 D200    ← เข้าถึง D100 โดยตรง (ที่อยู่คงที่)

Indirect Addressing:
  MOV D100Z0 D200  ← เข้าถึง D[100 + Z0] (ที่อยู่เป็น Variable)

ถ้า Z0 = 5 → เข้าถึง D105
ถ้า Z0 = 10 → เข้าถึง D110
```

### Index Registers (Z, V)

```
Mitsubishi FX:
  Z0-Z7   = Index Registers (16-bit offset)
  V0-V7   = Index Registers (alternative)

ใช้: D100Z0 หมายถึง D[100 + Z0]

Siemens:
  ใช้ Indirect via Pointer (DB.DBXx.y หรือ [AR1, ...])

Allen-Bradley:
  ใช้ File Indirect: N7:[N7:0] หมายถึง N7[N7:0]
```

### ตัวอย่าง 1: Array Access

```
สถานการณ์: เก็บ Temperature ของ 10 Sensor
  Sensor Data: D100-D109
  Sensor Number = D0 (0-9)
  Read Sensor D0: Result = D200


Network 1: กำหนด Index
     M8000
  ───┤ ├────[MOV D0 Z0]─             ← Z0 = Sensor Number (0-9)

Network 2: อ่านค่า Sensor
     M8000
  ───┤ ├────[MOV D100Z0 D200]─       ← D200 = D[100 + Sensor_Number]
```

### ตัวอย่าง 2: Write ไปยัง Array

```
สถานการณ์: บันทึก Production Count ของ 8 Machine
  Machine Counts: D400-D407
  Machine_ID: D1 (0-7)
  Count_This_Cycle: D2


Network 1: เพิ่ม Count
     I5.0
  ───┤ ├────[MOV D1 Z0]─             ← Z0 = Machine_ID
  Cycle_Done [ADD D400Z0 D2 D400Z0]─ ← D400[ID] += Count
```

### ตัวอย่าง 3: Loop Processing

```
สถานการณ์: ตรวจสอบ 10 Sensor ทีละตัว

  Sensor Data: D100-D109
  HIGH_LIMIT = 800
  Fault_Array: D200-D209 (1 = Fault)


Network 1: Loop Counter
     M8002
  ───┤ ├────[MOV K0 D5]─             ← Init loop counter

Network 2: Process each sensor
     M8000
  ───┤ ├────[MOV D5 Z0]─             ← Z0 = loop index

     [> D100Z0 K800]       D6
  ─────┤ ├────────────[MOV K1 D200Z0]─   ← Mark fault

     [<= D100Z0 K800]      D6
  ─────┤ ├────────────[MOV K0 D200Z0]─   ← Clear fault

     M8000
  ───┤ ├────[INC D5]─                ← increment counter
            [MOV D5 Z0]─             ← update Z0
            [CMP D5 K10]─
            [>= D5 K10][MOV K0 D5]─  ← reset when = 10
```

---

## Step 157: Data Arrays {#step-157}

### Array Operations พื้นฐาน

```
ใน PLC Arrays คือ Consecutive Registers:
  D100[0] = D100
  D100[1] = D101
  D100[2] = D102
  ...
  D100[N-1] = D(100+N-1)
```

### ตัวอย่าง: Running Average (Moving Average)

```
สถานการณ์: Average จาก 8 ค่าล่าสุด (smooth เส้นกราฟ)

  Sample Buffer: D150-D157 (8 values)
  Buffer Index:  D8 (0-7)
  New_Sample:    D9 (ค่าใหม่)
  Average:       D160


Network 1: เพิ่มค่าใหม่เข้า Buffer (Circular)
     M50.0                              ← New Sample Available
  ───┤ ├────[MOV D8 Z0]─               ← Z0 = Index
            [MOV D9 D150Z0]─           ← Store new sample
            [INC D8]─                  ← Increment index
            [CMP D8 K8]─               ← Check if > 7
            [>= D8 K8][MOV K0 D8]─     ← Reset to 0 (circular)

Network 2: คำนวณ Sum
     M50.0
  ───┤ ├────[MOV K0 D161]─             ← Clear sum
  New_Sample [ADD D150 D151 D161]─     ← หรือทำ Loop...
             [ADD D161 D152 D161]─
             [ADD D161 D153 D161]─
             [ADD D161 D154 D161]─
             [ADD D161 D155 D161]─
             [ADD D161 D156 D161]─
             [ADD D161 D157 D161]─

Network 3: คำนวณ Average
     M50.0
  ───┤ ├────[DIV D161 K8 D160]─        ← Average = Sum ÷ 8
```

---

## Step 158: Shift Registers {#step-158}

### Shift Register คืออะไร?

```
Shift Register = ข้อมูลเลื่อนไปทางหนึ่งพร้อมกัน ทุก Clock Pulse

Bit Shift Register:
  [M0][M1][M2][M3][M4][M5][M6][M7]
       ← Shift Left ←

ก่อน: [1][0][0][1][0][0][0][0]
หลัง: [0][0][1][0][0][0][0][0]   ← ข้อมูลเลื่อนซ้าย 1 ตำแหน่ง
       (ค่าใหม่จากขวา = 0)
```

### ใช้ Shift Register สำหรับ Conveyor Tracking

```
สถานการณ์: Conveyor 8 ตำแหน่ง ติดตามว่าชิ้นงานอยู่ตรงไหน

  Bit M0-M7 = ตำแหน่งบน Conveyor (M0=Position1, M7=Position8)
  Sensor ตรวจจับชิ้นงานที่ Position 1


Step 1: ชิ้นงานเข้า Position 1
  [1][0][0][0][0][0][0][0]   ← M0 = 1

Step 2: Shift (Conveyor เคลื่อน 1 ขั้น)
  [0][1][0][0][0][0][0][0]   ← M1 = 1 (ย้ายไป Position 2)

Step 3: ชิ้นงานใหม่เข้า, Shift อีก
  [1][0][1][0][0][0][0][0]   ← ชิ้นงาน 2 ที่ P1, ชิ้นแรกที่ P3
```

### Mitsubishi FX - SFTR (Shift Right) / SFTL (Shift Left)

```
SFTL = Shift Left:
  SFTL M0 M0 K8 K1     ← Shift M0-M7 ซ้าย 1 bit, ค่าใหม่จาก M0

ลำดับ:
  M7 ← M6 ← M5 ← M4 ← M3 ← M2 ← M1 ← M0 ← [New Bit]


Network 1: Detect New Part at Entry (Rising Edge)
     I0.0           M260.0          M261.0
  ───┤ ├─────────────┤/├─────────────( )───   ← Entry Pulse
  Entry_Sensor    Prev

     I0.0              M260.0
  ───┤ ├────────────────( )───               ← Update Prev

Network 2: Shift เมื่อ Conveyor เคลื่อน 1 Position
     I0.1              M261.0
  ───┤ ├──────────────────[SFTL M261.0 M0 K8 K1]─
  Conveyor_Pulse   New_Entry_Bit        ← Shift M0-M7 ← M261.0

Network 3: ทำงานที่ Position 5 (M4 = 1 → ชิ้นงานถึง Station 5)
     M4                Q2.0
  ───┤ ├───────────────( )───           ← Trigger Station 5 Operation
  Part_at_P5
```

### Word Shift Register

```
สำหรับ Tracking ข้อมูลที่มากกว่าแค่ Presence/Absence:

D100-D107 = Position Data (เช่น Part ID หรือ Type)

Network 1: Shift Word Register
     I0.1
  ───┤ ├────[BMOV D100 D101 K7]─      ← Shift D100-D106 → D101-D107
  Conveyor_Pulse

Network 2: Enter New Part Data
     I0.0
  ───┤ ├────[MOV D_New_Part D100]─    ← ใส่ข้อมูลชิ้นงานใหม่ที่ D100
  New_Part
```

---

## Step 159: FIFO - First In, First Out {#step-159}

### FIFO คืออะไร?

```
FIFO Queue = คิวแบบ "เข้าก่อนออกก่อน" เหมือนคิวร้านอาหาร

Write (Enqueue) → [D100][D101][D102][D103][D104] → Read (Dequeue)
                    ↑                               ↑
                 เข้าทางนี้                      ออกทางนี้

ใช้เมื่อ:
- Buffer ข้อมูลระหว่าง Production Rate ไม่สม่ำเสมอ
- Queue ของ Work Orders
- Log Events ตามลำดับเวลา
```

### Mitsubishi FX: FIFO Instructions

```
FIFW (FIFO Write): เพิ่มข้อมูลเข้า FIFO
FIFR (FIFO Read):  อ่านข้อมูลออกจาก FIFO

Setup:
  D200 = FIFO Pointer (PLC จัดการเอง)
  D201-D250 = FIFO Data Area (50 slots)

FIFW D0 D200   ← Write D0 into FIFO starting at D200
FIFR D200 D10  ← Read from FIFO into D10
```

### ตัวอย่าง: Buffer Work Orders

```
สถานการณ์:
  - SCADA ส่ง Work Order (Job ID) มาเรื่อยๆ
  - Machine ทำได้ทีละ 1 Job
  - ต้อง Queue งานที่รอ


FIFO Area: D200-D249 (50 slots)
D200 = Pointer (PLC จัดการ)


Network 1: รับ Work Order ใหม่
     M300.0
  ───┤ ├────[FIFW D301 D200]─         ← เพิ่ม Job ID (D301) เข้า Queue
  New_WO_Received

Network 2: เมื่อ Machine ว่าง อ่านงานถัดไป
     M302.0          M303.0
  ───┤ ├──────────────┤/├────[FIFR D200 D305]─  ← อ่าน Job ถัดไป
  Machine_Idle    Queue_Empty

Network 3: ตรวจสอบว่า Queue ว่างไหม
     [= D200 K0]       M303.0
  ─────┤ ├──────────────( )───         ← Queue Empty เมื่อ Pointer = 0
```

### FIFO Implementation เอง (ถ้า PLC ไม่มี FIFO Instruction)

```
FIFO Buffer: D500-D519 (20 slots)
Write Pointer: D520 (0-19)
Read Pointer:  D521 (0-19)
Count:         D522 (0-20)


Network 1: Write to FIFO
     M400.0          [< D522 K20]       ← เขียนได้ถ้า Buffer ไม่เต็ม
  ───┤ ├──────────────┤ ├────[MOV D520 Z0]─
  Write_Request                         [MOV D_Write D500Z0]─  ← Write data
                                        [INC D520]─           ← Advance write ptr
                                        [CMP D520 K20]─
                                        [>= D520 K20][MOV K0 D520]─ ← Wrap around
                                        [INC D522]─           ← Increase count

Network 2: Read from FIFO
     M401.0          [> D522 K0]        ← อ่านได้ถ้า Buffer ไม่ว่าง
  ───┤ ├──────────────┤ ├────[MOV D521 Z0]─
  Read_Request                          [MOV D500Z0 D_Read]─  ← Read data
                                        [INC D521]─           ← Advance read ptr
                                        [CMP D521 K20]─
                                        [>= D521 K20][MOV K0 D521]─
                                        [DEC D522]─           ← Decrease count
```

---

## Step 160: LIFO - Last In, First Out {#step-160}

### LIFO Stack คืออะไร?

```
LIFO Stack = Stack แบบ "เข้าหลังออกก่อน" เหมือนกองจาน

Push: เพิ่มข้อมูลบนสุด
Pop:  เอาข้อมูลบนสุดออก

[D105] ← Top (อ่านก่อน)
[D104]
[D103]
[D102]
[D101]
[D100] ← Bottom

ใช้เมื่อ:
- Nested Operations (Function Call Stack)
- Undo History
- Depth-first Search
```

### LIFO Implementation

```
Stack Area:  D600-D619 (20 slots)
Stack Top:   D620 (index ของ top, 0 = empty)


Network 1: Push (เพิ่มข้อมูลลง Stack)
     M410.0          [< D620 K20]       ← Push เฉพาะเมื่อไม่เต็ม
  ───┤ ├──────────────┤ ├────[MOV D620 Z0]─
  Push                                  [MOV D_Push D600Z0]─  ← Store data
                                        [INC D620]─           ← Move top up


Network 2: Pop (เอาข้อมูลออก)
     M411.0          [> D620 K0]        ← Pop เฉพาะเมื่อไม่ว่าง
  ───┤ ├──────────────┤ ├────[DEC D620]─
  Pop                                   [MOV D620 Z0]─        ← Point to top
                                        [MOV D600Z0 D_Pop]─   ← Read data
```

---

## Step 161: Recipe Data Management {#step-161}

### ระบบ Recipe Management

```
Recipe = ชุดพารามิเตอร์สำหรับการผลิตผลิตภัณฑ์ต่างๆ

Recipe Structure (5 parameters per recipe):
  D1000 = Recipe 1, Param 1 (Temperature)
  D1001 = Recipe 1, Param 2 (Time)
  D1002 = Recipe 1, Param 3 (Pressure)
  D1003 = Recipe 1, Param 4 (Speed)
  D1004 = Recipe 1, Param 5 (Flow)

  D1005 = Recipe 2, Param 1
  D1006 = Recipe 2, Param 2
  ...
  D1024 = Recipe 5, Param 5

Current Recipe: D100
Active Params:  D200-D204 (5 params ที่ใช้งาน)
```

### ตัวอย่าง: Load Recipe

```
Network 1: คำนวณ Offset ของ Recipe ที่เลือก
     M8000
  ───┤ ├────[SUB D100 K1 D9]─         ← Offset = Recipe# - 1
            [MUL D9 K5 Z0]─           ← Z0 = Offset × 5 params


Network 2: Load Recipe Parameters
     I10.0
  ───┤ ├────[BMOV D1000Z0 D200 K5]─   ← Copy 5 params ไป Active Area
  Load_Recipe
  Button

Visual check:
  D100 = 1 → Z0 = 0 → BMOV D1000 D200 K5 (Recipe 1)
  D100 = 2 → Z0 = 5 → BMOV D1005 D200 K5 (Recipe 2)
  D100 = 3 → Z0 = 10 → BMOV D1010 D200 K5 (Recipe 3)
```

### ตัวอย่าง: Save Modified Recipe

```
Network 3: Save Recipe (เขียนกลับหลังแก้ไข)
     I10.1
  ───┤ ├────[BMOV D200 D1000Z0 K5]─   ← Save Active Params → Recipe Area
  Save_Recipe
  Button
```

### Recipe Table

```
Recipe Table สมบูรณ์:

┌──────────┬────────┬────────┬──────────┬────────┬────────┐
│ Recipe # │ Temp   │ Time   │ Pressure │ Speed  │ Flow   │
│          │ (°C×10)│ (min)  │ (kPa×10) │ (rpm)  │ (L/m)  │
├──────────┼────────┼────────┼──────────┼────────┼────────┤
│ 1 (A)    │ 1500   │ 30     │ 5000     │ 1200   │ 150    │
│ 2 (B)    │ 2000   │ 45     │ 8000     │ 1500   │ 200    │
│ 3 (C)    │ 1800   │ 20     │ 6500     │ 1000   │ 180    │
│ 4 (D)    │ 2500   │ 60     │ 10000   │ 2000   │ 250    │
│ 5 (E)    │ 1200   │ 15     │ 3000     │ 800    │ 100    │
└──────────┴────────┴────────┴──────────┴────────┴────────┘

Network 4: Initialize Recipe Table ตอน Startup
     M8002
  ───┤ ├────[MOV K1500 D1000]─  Recipe 1
            [MOV K30   D1001]─
            [MOV K5000 D1002]─
            [MOV K1200 D1003]─
            [MOV K150  D1004]─

            [MOV K2000 D1005]─  Recipe 2
            [MOV K45   D1006]─
            ...
```

---

## Step 162: Historical Data Logging {#step-162}

### Data Logging สำหรับ Production History

```
บันทึก:
  - Temperature ทุก 1 นาที (60 ค่าต่อชั่วโมง)
  - Alarm เมื่อเกิด Fault
  - Production Count ต่อ Shift


Temperature Log:
  D2000-D2059 = ค่า Temperature 60 ค่า (1 ชั่วโมง)
  D2060 = Log Index (0-59)


Network 1: Log Temperature ทุก 1 นาที (ใช้ M8014 = 1min pulse ใน Mitsubishi)

     M8014              D2060            D10
  ───┤ ├────[MOV D2060 Z0]──[MOV D10 D2000Z0]─ ← เก็บ Temp ที่ Index นี้
  1min_Pulse             [INC D2060]─           ← เพิ่ม Index
                         [CMP D2060 K60]─
                         [>= D2060 K60][MOV K0 D2060]─ ← Wrap
```

### Event Log (Alarm History)

```
Event Log: D3000-D3099 (100 events × 1 word each)
Event_Count: D3100


Network 2: บันทึก Alarm Event
     M100.7          M100.7            D3100
  ───┤ ├─────────────┤P├─────────────[MOV D3100 Z0]─  ← Rising Edge only
  Any_Fault                          [MOV D50 D3000Z0]─ ← D50 = Alarm Code
                                     [INC D3100]─
                                     [CMP D3100 K100]─
                                     [>= D3100 K100][MOV K0 D3100]─
```

### Shift Report

```
Shift Data (D3200-D3209):
  D3200 = Shift Number
  D3201 = Start Count
  D3202 = End Count (= Total Parts)
  D3203 = Good Parts
  D3204 = Reject Parts
  D3205 = Downtime (min)
  D3206 = Start Time (Hour)
  D3207 = End Time

Network 3: บันทึกเมื่อสิ้นสุด Shift
     I20.0
  ───┤ ├────[MOV D200 D3201]─         ← Start Count
  Shift_End  [MOV D201 D3202]─        ← End Count
             [MOV D202 D3203]─        ← Good Parts
             [MOV D203 D3204]─        ← Reject Parts
             [BMOV D3200 D3300 K10]─  ← Archive to History Area
```

---

## Step 163: Data Tables และ Look-up Tables {#step-163}

### Look-up Table คืออะไร?

```
เมื่อสูตรซับซ้อนเกินคำนวณ หรือต้องการความแม่นยำสูง
ใช้ Look-up Table: เก็บผลคำนวณไว้ล่วงหน้า

ตัวอย่าง: Thermocouple Linearization
  ADC 0 = 0°C
  ADC 100 = 5.7°C
  ADC 200 = 11.1°C
  ...
  ADC 4000 = 320°C
```

### Look-up Table สำหรับ Thermocouple

```
Lookup Table: D4000-D4040 (41 values, ADC ทุก 100 counts)
  D4000 = 0    (ADC=0    → 0.0°C × 10)
  D4001 = 57   (ADC=100  → 5.7°C × 10)
  D4002 = 111  (ADC=200  → 11.1°C × 10)
  ...
  D4040 = 3200 (ADC=4000 → 320.0°C × 10)

ADC Input: D0 (0-4095)
Temperature: D10


Network 1: คำนวณ Index (ADC ÷ 100)
     M8000
  ───┤ ├────[DIV D0 K100 Z0]─         ← Z0 = ADC ÷ 100 (integer)

Network 2: ค่าเศษ (สำหรับ Interpolation)
     M8000
  ───┤ ├────[MOD D0 K100 D11]─        ← D11 = ADC mod 100

Network 3: อ่านค่าจาก Table (ค่าต่ำ)
     M8000
  ───┤ ├────[MOV D4000Z0 D12]─        ← D12 = Lower value

Network 4: อ่านค่าจาก Table (ค่าสูง สำหรับ interpolation)
     M8000
  ───┤ ├────[INC Z0]─
            [MOV D4000Z0 D13]─        ← D13 = Upper value
            [DEC Z0]─                 ← Restore Z0

Network 5: Linear Interpolation
// Temp = Lower + (Upper - Lower) × (ADC mod 100) / 100
     M8000
  ───┤ ├────[SUB D13 D12 D14]─        ← Range = Upper - Lower
            [MUL D14 D11 D15]─        ← Range × Remainder
            [DDIV D15 K100 D16]─      ← ÷ 100
            [ADD D12 D16 D10]─        ← D10 = Temperature
```

---

## Step 164: String Operations {#step-164}

### String Handling ใน PLC

```
Siemens ใช้ STRING data type:
  สูงสุด 254 characters
  ใช้สำหรับ:
  - Display messages บน HMI
  - Barcode reading
  - Part numbering

ตัวอย่าง Siemens SCL:
  strTemp := 'Temperature: ';
  strValue := REAL_TO_STRING(temperature);
  strMessage := CONCAT(strTemp, strValue, '°C');
```

### Mitsubishi: ASCII String in D Registers

```
ใน Mitsubishi ใช้ Data Register เก็บ ASCII:
  D เก็บได้ 2 ตัวอักษร/register (Hi byte + Low byte)

ตัวอย่าง: เก็บ "HELLO"
  D700 = H4845 (H=0x48, E=0x45)
  D701 = H4C4C (L=0x4C, L=0x4C)
  D702 = H4F00 (O=0x4F, null=0x00)

ASCI instruction แปลง Numeric → ASCII String
```

---

## Step 165: Data Packing/Unpacking {#step-165}

### Packing Bits เข้า Word

```
สถานการณ์: ส่งสถานะ 16 อุปกรณ์ ผ่าน 1 Word (16 bits)

Device_Status = D600 (1 Word = สถานะ 16 อุปกรณ์)

Network 1: Pack Status Bits into Word
     M8000
  ───┤ ├────[MOV K0 D600]─            ← Clear word

  Method 1: ใช้ Bit Set
     M0.0
  ───┤ ├────[BSET D600 K0]─           ← Set bit 0 ถ้า M0.0 = ON

     M0.1
  ───┤ ├────[BSET D600 K1]─           ← Set bit 1 ถ้า M0.1 = ON

  Method 2: ใช้ OR/AND
     M8000
  ───┤ ├────[WAND D600 H0001 D601]─  ← Mask bit 0
            (ถ้า M0.0 = ON: [WOR D601 H0001 D600]─)
```

### Unpacking Bits จาก Word

```
Network 2: Unpack ค่าจาก Word ที่รับมา (เช่น จาก Modbus)

  Received_Word = D700
  Unpack เป็น M100-M115


  Method: ใช้ ANI (AND Immediate) ตรวจแต่ละ bit

     D700             H0001           M100
  ──[WAND D700 H0001 D710]──[<> D710 K0]──( )─── ← Bit 0

     D700             H0002           M101
  ──[WAND D700 H0002 D711]──[<> D711 K0]──( )─── ← Bit 1

  ...หรือใช้ Loop กับ Z Register...

     M8000
  ───┤ ├────[MOV K0 D9]─               ← Init loop 0-15
  ─── Loop:
  ───┤ ├────[MOV D9 Z0]─
            [MOV K1 D710]─
            [SHL D710 Z0 D711]─        ← Shift mask to bit position
            [WAND D700 D711 D712]─
            [> D712 K0][BSET M100Z0 K0]─ ← Set bit in M array
            [<= D712 K0][BRST M100Z0 K0]─
            [INC D9]─
            [< D9 K16][JMP Loop]─      ← Loop 16 times
```

---

## Step 166: Circular Buffer {#step-166}

### Circular Buffer สำหรับ Data Logging

```
Circular Buffer = ใช้พื้นที่เดิมซ้ำ เมื่อเต็มแล้ว Overwrite ค่าเก่า

ใช้เมื่อ:
- Real-time Data Trend (เก็บ N ค่าล่าสุด)
- Event Buffer
- Communication Buffer


Buffer: D5000-D5059 (60 slots)
Write Index: D5060 (0-59)
Full Flag: M170.0


Network 1: Write ค่าใหม่
     M8013              D5060           D_New
  ───┤ ├────[MOV D5060 Z0]──[MOV D_New D5000Z0]─ ← Write at current index
  1sec                  [INC D5060]─
  (every sec)           [CMP D5060 K60]─
                        [>= D5060 K60]──[MOV K0 D5060]─ ← Wrap
                                         [SET M170.0]─  ← Mark as Full

Network 2: อ่านค่าเก่าสุด (Oldest = Write_Index + 1)
  // ถ้า Not Full: Oldest = D5000
  // ถ้า Full: Oldest = D5000[Write_Index] (ค่าถัดไปที่จะ Overwrite)
```

---

## Step 167: Data Sorting {#step-167}

### Simple Bubble Sort Algorithm

```
สำหรับ Sort 8 ค่า (D700-D707) จาก น้อยไป มาก

ใช้ Subroutine ใน Mitsubishi (CALL/SRET) หรือทำใน Scan loop:

Pseudo Code:
  For i = 0 to 6:
    For j = 0 to 6-i:
      If D700[j] > D700[j+1]:
        Swap D700[j] and D700[j+1]
```

```
Network Sort: (ตัวอย่างง่ายๆ pass 1 เท่านั้น)
     I0.9              [> D700 D701]     D700      D701
  ───┤ ├──────────────────┤ ├──────────[XCH D700 D701]─ ← Swap if needed
  Sort_Trigger

     I0.9              [> D701 D702]
  ───┤ ├──────────────────┤ ├──────────[XCH D701 D702]─

  (... ทำสำหรับทุกคู่ ...)
```

---

## Step 168: Checksum และ Data Validation {#step-168}

### Checksum สำหรับตรวจสอบข้อมูล

```
Checksum = ผลรวมของทุก Byte/Word ใน Block
ใช้ตรวจว่าข้อมูลถูกต้องหรือเสีย (Data Integrity)

ตัวอย่าง: ตรวจสอบ Recipe Data (D200-D209)
  Checksum = D200 + D201 + ... + D209
  Store checksum at D210

ตรวจสอบ: คำนวณ Checksum ใหม่ แล้ว Compare กับ D210
  ถ้าเท่ากัน → ข้อมูลถูกต้อง
  ถ้าต่างกัน → ข้อมูลเสีย!


Network 1: คำนวณ Checksum
     I10.3
  ───┤ ├────[MOV K0 D210]─            ← Clear checksum
  Calc_CRC   [ADD D200 D201 D210]─
             [ADD D210 D202 D210]─
             [ADD D210 D203 D210]─
             ...
             [ADD D210 D209 D210]─    ← D210 = Checksum

Network 2: ตรวจสอบ Checksum
     I10.4
  ───┤ ├────[MOV K0 D215]─            ← Clear temp
  Check_CRC  [ADD D200 D201 D215]─   ← Recalculate
             ...
             [ADD D215 D209 D215]─

  Compare:
     [= D210 D215]       M180.0
  ─────┤ ├──────────────( )───        ← Checksum OK
  Checksums Match
```

---

## Step 169: Communication Buffers {#step-169}

### Modbus RTU Buffer Management

```
สำหรับ Modbus Communication:
  Transmit Buffer: D6000-D6019 (20 words)
  Receive Buffer:  D6100-D6119 (20 words)


Network 1: Build Modbus Request
     M190.0
  ───┤ ├────[MOV H01 D6000]─          ← Device Address = 1
  Send_Request [MOV H03 D6001]─       ← Function Code = 3 (Read Holding)
               [MOV H0064 D6002]─     ← Start Address = 100
               [MOV H000A D6003]─     ← Read 10 registers
               ...

Network 2: Parse Modbus Response
     M191.0
  ───┤ ├────[BMOV D6103 D100 K10]─    ← Copy response data to D100-D109
  Data_Received                         (skip header: D6100=addr, D6101=fc, D6102=len)
```

---

## Step 170: Workshop - Complete Data Management System {#step-170}

### โจทย์: ระบบเก็บข้อมูลการผลิตแบบครบวงจร

**ความต้องการ:**
1. Recipe Management (5 สูตร, 8 พารามิเตอร์)
2. Production Data Logging (ทุก 1 นาที, เก็บ 60 ค่า)
3. Batch Tracking (Conveyor 10 ตำแหน่ง)
4. Shift Report Generation
5. Alarm History (100 events)

### Memory Map

```
┌──────────┬────────────────────────────────────────────────────┐
│ Address  │ Description                                        │
├──────────┼────────────────────────────────────────────────────┤
│ D0-D9    │ Active Process Values (Temp, Pressure, Flow...)    │
│ D10-D19  │ Process Setpoints                                  │
│ D20-D24  │ Status Flags (in D Register form)                  │
│          │                                                    │
│ D100     │ Active Recipe Number (1-5)                        │
│ D101-D108│ Active Recipe Parameters                           │
│ D109     │ Recipe Checksum                                    │
│ D110-D149│ Recipe Table (5 recipes × 8 params each)           │
│          │                                                    │
│ D200-D209│ Conveyor Position Data (Track 10 positions)        │
│ D210     │ Conveyor Write Pointer                             │
│          │                                                    │
│ D300-D359│ Temperature Log (60 values, 1/minute)              │
│ D360     │ Log Write Index                                    │
│          │                                                    │
│ D400-D409│ Current Shift Data (Count, Good, Reject...)        │
│ D410-D449│ Shift History (4 shifts × 10 params)              │
│          │                                                    │
│ D500-D599│ Alarm History (100 events × 1 word = Alarm Code)   │
│ D600     │ Alarm History Write Index                          │
└──────────┴────────────────────────────────────────────────────┘
```

### โปรแกรมสมบูรณ์

```
//============================================================
// Complete Data Management System
//============================================================


// === INITIALIZATION ===

Network 1: Load Recipe Table on Startup
     M8002
  ───┤ ├────[MOV K1500 D110]─   Recipe 1: Temp SP
            [MOV K30   D111]─             Time
            [MOV K5000 D112]─             Pressure
            [MOV K1200 D113]─             Speed
            [MOV K150  D114]─             Flow
            [MOV K70   D115]─             Humidity
            [MOV K10   D116]─             pH
            [MOV K95   D117]─             Viscosity

            [MOV K2000 D118]─   Recipe 2: Temp SP
            // ... (Recipe 2-5 parameters)

Network 2: Initialize all Data Areas
     M8002
  ───┤ ├────[FMOV K0 D200 K200]─  ← Clear all data areas
            [MOV K1 D100]─        ← Default to Recipe 1


// === RECIPE MANAGEMENT ===

Network 3: Load Recipe
     I10.0
  ───┤ ├────[SUB D100 K1 D9]─         ← Offset = Recipe# - 1
  Load_Recipe [MUL D9 K8 Z0]─         ← Z0 = Offset × 8
             [BMOV D110Z0 D101 K8]─   ← Copy Recipe to Active Area


// === CONVEYOR TRACKING (Shift Register) ===

Network 4: New Part Enters Conveyor
     I0.0           M260.0          M261.0
  ───┤ ├─────────────┤/├─────────────( )───
  Entry_Sensor    Prev              Entry_Pulse

     I0.0              M260.0
  ───┤ ├────────────────( )───

Network 5: Shift Conveyor Data
     I0.1
  ───┤ ├────[SFTL M261.0 M0 K10 K1]─  ← Shift M0-M9 ← Entry signal
  Conveyor_Pulse


// === PROCESS DATA LOGGING (every 1 minute) ===

Network 6: Log Temperature
     M8014              D360            D0
  ───┤ ├────[MOV D360 Z0]──[MOV D0 D300Z0]─   ← Log current temp
  1min Pulse            [INC D360]─
                        [CMP D360 K60]─
                        [>= D360 K60][MOV K0 D360]─ ← Circular


// === SHIFT TRACKING ===

Network 7: Count Good Parts
     M90.0             D400
  ───┤ ├────────────[INC D400]─        ← Increment shift good count
  Part_Good_Flag

Network 8: Count Reject Parts
     M90.1             D401
  ───┤ ├────────────[INC D401]─        ← Increment reject count
  Part_Reject

Network 9: Shift End - Save Shift Report
     I20.0
  ───┤ ├────[BMOV D400 D410 K10]─      ← Archive shift data
  Shift_End  [FMOV K0 D400 K10]─       ← Clear for next shift
             [INC D420]─               ← Increment Shift Counter


// === ALARM HISTORY ===

Network 10: Log Alarm Event (Rising Edge only)
     M100.7          M8.0            D600           D50
  ───┤ ├──────────────┤P├─[MOV D600 Z0]──[MOV D50 D500Z0]─
  Any_Fault                          [INC D600]─
                                     [CMP D600 K100]─
                                     [>= D600 K100][MOV K0 D600]─


// === OUTPUTS TO HMI ===

Network 11: Prepare HMI Display Data
     M8000
  ───┤ ├────[BMOV D0 D900 K10]─        ← Process Values → HMI Area
            [BMOV D101 D910 K8]─       ← Active Recipe → HMI Area
            [BMOV D400 D920 K5]─       ← Shift Counts → HMI Area
```

### Lab Projects

**Lab 1:** เพิ่ม CRC Checksum ให้กับ Recipe Data เพื่อตรวจสอบความถูกต้องเมื่อโหลด Recipe

**Lab 2:** สร้าง FIFO Buffer สำหรับ Print Queue (เมื่อต้องพิมพ์ Label หลายใบ)

**Lab 3:** เพิ่ม Trend Display - ส่ง Historical Temperature Log ไป HMI แบบ Array

**Lab 4:** Implement ระบบ Batch Genealogy - ติดตาม Recipe ที่ใช้กับแต่ละ Batch และ Lot Number

**Lab 5:** ทดสอบ SFTL Shift Register ด้วย Simulator:
- สร้าง Conveyor 10 Position
- ใส่ชิ้นงาน 3 ชิ้นในเวลาต่างกัน
- ตรวจสอบว่าชิ้นงานเลื่อนไปตำแหน่งที่ถูกต้อง

---

## สรุป Part 09

```
┌──────────────────────────────────────────────────────────────┐
│ สิ่งที่เรียนรู้ใน Part 09:                                    │
│                                                              │
│ ✓ MOV, BMOV, FILL, SWAP - พื้นฐาน Data Move                  │
│ ✓ Indirect Addressing - Pointer-based access                 │
│ ✓ Data Arrays - การเข้าถึงข้อมูลเป็น Array                   │
│ ✓ Shift Registers - Conveyor Tracking                        │
│ ✓ FIFO/LIFO - Queue และ Stack Operations                     │
│ ✓ Recipe Management - Load/Save/Switch recipes               │
│ ✓ Data Logging - Circular Buffer, Event Log                  │
│ ✓ Look-up Tables - สำหรับ Non-linear sensor                  │
│ ✓ Checksum - Data Integrity Validation                       │
│ ✓ Complete Data Management System                            │
└──────────────────────────────────────────────────────────────┘
```

---

**[← Part 08: Comparison Instructions](part08_comparison.md)** | **[Part 10: โปรแกรม PLC แรก →](part10_first_plc_program.md)**
