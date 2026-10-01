# Part 17: Allen-Bradley / Rockwell Automation - Studio 5000 และ Logix PLC

## Steps 311-330: มาตรฐาน Tag-Based Programming ของอเมริกา

**[← Part 16: Siemens S7](part16_siemens_s7.md) | [Part 18: Mitsubishi →](part18_mitsubishi.md)**

---

## บทนำ: Allen-Bradley ในอุตสาหกรรม

Allen-Bradley (AB) เป็นแบรนด์ PLC ของ Rockwell Automation ที่ครองตลาดสูงสุดในอเมริกาเหนือ และมีฐานลูกค้าขนาดใหญ่ในอุตสาหกรรมยานยนต์ทั่วโลก รวมถึงโรงงานของ Ford, GM, Toyota Plant ในอเมริกา

**ข้อดีของ Allen-Bradley:**
- Tag-based addressing (ไม่มี Address แบบ I0.0)
- ControlLogix: Platform ระดับ Enterprise ที่ยืดหยุ่นสูง
- Integrated Motion (Servo Control แบบ Native)
- PlantPAx: Process Automation Library
- EtherNet/IP: Protocol มาตรฐานสำหรับ Industrial IoT

---

## Step 311: การติดตั้งและเริ่มต้น Studio 5000

### Studio 5000 Environment

```
Studio 5000 Logix Designer (เดิมชื่อ RSLogix 5000):

System Requirements:
- OS: Windows 10 (64-bit) Professional หรือ Enterprise
- RAM: 8 GB ขั้นต่ำ (16 GB แนะนำ)
- HDD: 20 GB ว่าง
- .NET Framework 4.7.2+

Components ที่ติดตั้ง:
- Studio 5000 Logix Designer
- RSLinx Classic / FactoryTalk Linx (Communication Driver)
- RSLogix Emulate 5000 (Simulator)
- FactoryTalk View Studio (HMI)
```

### หน้าจอหลัก Studio 5000

```
╔══════════════════════════════════════════════════════════════════╗
║  Studio 5000 Logix Designer                                     ║
╠════════════╦═════════════════════════════════╦══════════════════╣
║ Controller ║ Routine Window                  ║ Properties      ║
║ Organizer  ║ (Ladder Diagram)                ║                 ║
║            ║                                 ║                 ║
║ ▼ Controller║ [Rung 0000]                    ║                 ║
║ ▼ Tasks    ║ ──┤ ├──┤ ├──────────( )──      ║                 ║
║   ▼MainTask║                                 ║                 ║
║    MainProg║ [Rung 0001]                     ║                 ║
║    SafeProg║ ──┤ ├──[TON]────────( )──       ║                 ║
║ ▼ Motion   ║                                 ║                 ║
║ ▼ Add-On   ║                                 ║                 ║
║ ▼ Data Types║                                ║                 ║
║ ▼ I/O Config║                                ║                 ║
╚════════════╩═════════════════════════════════╩══════════════════╝
```

---

## Step 312: CompactLogix และ ControlLogix Hardware

### ครอบครัว Logix Controllers

```
ตระกูล Logix Controllers:

┌──────────────┬──────────────────────────────────────────────┐
│ Platform     │ คุณสมบัติ                                    │
├──────────────┼──────────────────────────────────────────────┤
│ MicroLogix   │ Entry-level, ไม่ใช้ Studio 5000              │
│ (Legacy)     │ ใช้ RSLogix 500                              │
├──────────────┼──────────────────────────────────────────────┤
│ CompactLogix │ Mid-range, Compact chassis                   │
│ 5370/5380    │ Built-in EtherNet/IP                         │
│              │ ไม่มี Backplane, ต่อกันแบบ Module             │
├──────────────┼──────────────────────────────────────────────┤
│ ControlLogix │ High-end, แยก Chassis                        │
│ 5570/5580    │ Redundant CPU                                │
│              │ Multi-CPU ใน Chassis เดียว                  │
├──────────────┼──────────────────────────────────────────────┤
│ GuardLogix   │ Safety PLC (SIL 3, PLe)                     │
│ 5570S/5580S  │ Dual CPU ใน 1 Controller                    │
└──────────────┴──────────────────────────────────────────────┘
```

### CompactLogix 5380 Hardware

```
CompactLogix 5380 Layout:

[1769-L33ER CPU]──[1769-IQ16]──[1769-OB16]──[1769-IF4]──[End Cap]
     │                │               │            │
  ETHERNET          16 DI           16 DO        4 AI
  USB Port        24VDC Sink      24VDC Src     (±10V or 4-20mA)
  SD Card
  
ControlLogix 5580 Chassis (7 Slot):

Slot: [0]     [1]     [2]     [3]     [4]     [5]     [6]
      CPU     ENBT    DI      DO      AI      AO      DINT
     5580    1756-   1756-   1756-   1756-   1756-    (ว่าง)
            EN2T    IB32    OB32    IF16    OF8

ENBT = EtherNet/IP Bridge Module
```

### EtherNet/IP Topology

```
ControlLogix Architecture:

[Studio 5000 PC]──── Managed Switch ────[ControlLogix 1756]
       │                   │
[FactoryTalk View]         │──[CompactLogix 5380]
       │                   │──[PowerFlex 755 Drive]
[RSLinx Gateway]           │──[PanelView HMI]
                           │──[Remote I/O - POINT I/O]
```

---

## Step 313: การสร้าง Project ใน Studio 5000

### ขั้นตอนสร้าง Project

```
1. File → New
2. ตั้งชื่อ Project: "Conveyor_Line_1"
3. เลือก Controller:
   - Vendor: Allen-Bradley
   - Type: CompactLogix 5380 L83E
   - Revision: 32.011
4. กำหนด: Chassis Size, Slot, IP Address
5. Click OK

หน้าจอ Controller Organizer:
┌──────────────────────────────────┐
│ ▼ Conveyor_Line_1                │
│   ▼ Tasks                        │
│     ▼ MainTask (Continuous)      │
│       ▼ MainProgram               │
│           MainRoutine ← Program ที่นี่
│           FaultRoutine            │
│   ▼ Motion Groups                 │
│   ▼ Add-On Instructions          │
│   ▼ Data Types                   │
│   ▼ Trends                       │
│   ▼ I/O Configuration            │
│   ▼ Controller Tags              │
│   ▼ Program Tags (MainProgram)   │
└──────────────────────────────────┘
```

### Task และ Program Structure

```
Allen-Bradley มี Hierarchy: Task → Program → Routine

Task ประเภท:
1. Continuous: ทำงานตลอดเวลา (เหมือน Main Scan)
2. Periodic: ทำงานตาม Period (เช่น ทุก 10ms)
3. Event: ทำงานเมื่อ Event เกิดขึ้น

Program ใน Task:
- มีได้หลาย Program ใน 1 Task
- ทุก Program มี "Scheduled" Tag (เปิด/ปิด Program)
- Program Tags = Local Tags (ใช้ใน Program นั้นเท่านั้น)

Routine ประเภท:
- Ladder Diagram (LAD)
- Function Block Diagram (FBD)
- Structured Text (ST)
- Sequential Function Chart (SFC)

Program Scan Order (Continuous Task):
Task → Program 1 → Program 2 → Program 3 → (วนซ้ำ)
```

---

## Step 314: Tag-Based Programming - หัวใจของ AB

### ความแตกต่างจาก Siemens

```
Siemens S7-1200:
I0.0, I0.1, Q0.0, MW10, DB1.DBW0
→ มี Physical Address

Allen-Bradley Logix:
KHÔNG มี Address แบบ I0.0!
ทุกอย่างเป็น Tag:
Start_PB, Stop_PB, Motor_Run, Speed_SP

Tag ผูกกับ I/O ผ่าน:
Controller Tags → Properties → I/O Connection
"Start_PB" → Local:1:I.Data.0 (Input Module, Slot 1, Bit 0)
"Motor_Run" → Local:2:O.Data.0 (Output Module, Slot 2, Bit 0)
```

### การสร้าง Controller Tags

```
ใน Controller Organizer:
Controller Tags → Double-click → Tag Editor

┌────────────────────────────────────────────────────────────┐
│ Controller Tags - Conveyor_Line_1                         │
├────────────────┬───────────────┬──────────┬───────────────┤
│ Tag Name       │ Type          │ Alias For│ Description   │
├────────────────┼───────────────┼──────────┼───────────────┤
│ Start_PB       │ BOOL          │ Local:1:I│ ปุ่ม Start     │
│                │               │ .Data.0  │               │
│ Stop_PB        │ BOOL          │ Local:1:I│ ปุ่ม Stop      │
│                │               │ .Data.1  │               │
│ E_Stop         │ BOOL          │ Local:1:I│ Emergency Stop │
│                │               │ .Data.2  │               │
│ Motor_Coil     │ BOOL          │ Local:2:O│ Contactor      │
│                │               │ .Data.0  │               │
│ Motor_Timer    │ TIMER         │          │ Run Timer      │
│ Prod_Counter   │ COUNTER       │          │ Production Cnt │
│ Speed_SP       │ DINT          │          │ Speed Setpoint │
└────────────────┴───────────────┴──────────┴───────────────┘
```

### Data Types ใน Allen-Bradley

```
Atomic Types:
BOOL   : 1 bit
SINT   : 8 bit signed  (-128 ถึง 127)
INT    : 16 bit signed (-32768 ถึง 32767)
DINT   : 32 bit signed
LINT   : 64 bit signed
USINT  : 8 bit unsigned
UINT   : 16 bit unsigned
UDINT  : 32 bit unsigned
ULINT  : 64 bit unsigned
REAL   : 32 bit float
LREAL  : 64 bit double
STRING : Variable length string (82 chars default)

Structure Types (Built-in):
TIMER: {PRE, ACC, DN, TT, EN, ...}
COUNTER: {PRE, ACC, DN, OV, UN, CU, CD, ...}
CONTROL: {LEN, POS, EN, EU, DN, EM, ...}
PID: {CTL, SP, PV, OUT, ...}

Array:
BOOL[100]   : Array of 100 BOOL
DINT[10][5] : 2D Array 10x5

User-Defined Types (UDT):
โปรแกรมเมอร์สร้างเอง
```

---

## Step 315: LAD Programming ใน Studio 5000

### Instruction ใน Allen-Bradley

```
Contact Instructions:
XIC (Examine If Closed): ──┤ ├── (Normally Open)
XIO (Examine If Open):   ──┤/├── (Normally Closed)
OSR (One Shot Rising):   ──┤↑├── (Rising Edge, Pulse 1 Scan)
OSF (One Shot Falling):  ──┤↓├── (Falling Edge)

Coil Instructions:
OTE (Output Energize):   ──( )── (Output, จะ Reset เมื่อ FALSE)
OTL (Output Latch):      ──(L)── (Set, จำสถานะ)
OTU (Output Unlatch):    ──(U)── (Reset, ยกเลิก Latch)
ONS (One Shot):          ──(ONS)── (Pulse 1 Scan)

ความแตกต่างจาก Siemens:
OTL/OTU ≈ SET/RESET ของ Siemens
```

### Motor Start/Stop Circuit

```
// Motor Control แบบมาตรฐาน AB

Rung 0: Start/Stop with Seal-In
    Start_PB    E_Stop       Stop_PB    Motor_Coil
    ──┤ ├──┬────┤/├────────────┤/├────────( )──
           │
    Motor_Coil
    ──┤ ├──┘

Rung 1: Fault Latch
    Overload_Trip          Fault_Latch
    ──┤ ├─────────────────────(L)──

    Reset_PB               Fault_Latch
    ──┤ ├─────────────────────(U)──

Rung 2: Fault Indicator
    Fault_Latch            Fault_Light
    ──┤ ├────────────────────( )──

// หมายเหตุ: ใน AB ไม่มี "Memory Bit" แยก
// ใช้ BOOL Tag แทน (เช่น Fault_Latch)
// ต่างจาก Siemens ที่มี M area
```

### Rung Comment ที่ดี

```
// การเพิ่ม Comment สำคัญมากใน AB

Rung 0: (* Motor Start/Stop Control with E-Stop *)
(* Start = I:0/0, Stop = I:0/1, E-Stop = I:0/2 *)
(* Motor_Coil เป็น OTE ดังนั้นจะ Reset เมื่อ Power Flow ไม่ถึง *)

    Start_PB    E_Stop       Stop_PB    Motor_Coil
    ──┤ ├──┬────┤/├────────────┤/├────────( )──
           │
    Motor_Coil
    ──┤ ├──┘
```

---

## Step 316: Timer Instructions ใน Allen-Bradley

### TON - Timer On-Delay

```
TIMER Data Type:
TON.PRE  : Preset (ค่าที่ตั้ง) - หน่วยเป็น ms
TON.ACC  : Accumulated (ค่าสะสม) - ms
TON.DN   : Done Bit (TRUE เมื่อ ACC >= PRE)
TON.TT   : Timer Timing (TRUE ขณะนับ)
TON.EN   : Enable (TRUE เมื่อ Input TRUE)

ตัวอย่าง: ดีเลย์ 5 วินาที
Rung 0: Start Delay Timer
    Start_PB              [TON         ]
    ──┤ ├────────────── Timer: Motor_Timer
                         Time Base: 1.0
                         Preset:    5000  ← 5 วินาที (ms)
                         Accum:     0

Rung 1: Motor Output หลัง Delay
    Motor_Timer.DN    E_Stop      Motor_Coil
    ──┤ ├────────────┤/├────────────( )──

// หมายเหตุ: AB Timer ใช้ ms (ไม่ใช่ T# เหมือน Siemens)
// 5 วินาที = Preset 5000 (ms)
// Time Base: 1.0 = 1 second base (Preset ×1)
// Time Base: 0.01 = 10ms base (Preset ×0.01 sec)
```

### TOF - Timer Off-Delay

```
Rung 0: Motor Fan Off-Delay (ระบายความร้อนหลังหยุด)
    Motor_Running         [TOF         ]
    ──┤ ├────────────── Timer: Fan_Timer
                         Time Base: 1.0
                         Preset:    300000  ← 5 นาที
                         Accum:     0

Rung 1: Fan Motor
    Fan_Timer.DN         [XIO]
    ──┤/├────────────────────────( Fan_Motor )──
// Fan_Timer.DN = FALSE ขณะนับ = Fan วิ่ง
// Fan_Timer.DN = TRUE หลัง 5 นาที = Fan หยุด
```

### RTO - Retentive Timer

```
RTO (Retentive Timer On-Delay):
- สะสมเวลาเมื่อ Input TRUE
- ไม่ Reset เมื่อ Input กลับ FALSE
- ต้อง Reset ด้วย RES Instruction

Rung 0: สะสมเวลาวิ่ง Motor
    Motor_Running         [RTO         ]
    ──┤ ├────────────── Timer: Run_Hours
                         Time Base: 1.0
                         Preset:    28800000  ← 8 ชั่วโมง
                         Accum:     0

Rung 1: แจ้งเตือนถึงเวลา PM
    Run_Hours.DN          Maint_Alarm
    ──┤ ├────────────────────( )──

Rung 2: Reset หลัง Maintenance
    PM_Done_PB            [RES        ]
    ──┤ ├────────────────Timer: Run_Hours
```

---

## Step 317: Counter Instructions

### CTU - Count Up

```
COUNTER Data Type:
CTU.PRE  : Preset Value
CTU.ACC  : Accumulated Value
CTU.DN   : Done Bit (ACC >= PRE)
CTU.OV   : Overflow Bit
CTU.UN   : Underflow Bit (CTD ใช้)
CTU.CU   : Count Up Enable
CTU.CD   : Count Down Enable (CTD)

ตัวอย่าง: นับชิ้นงาน
Rung 0: Count Parts
    Part_Sensor           [CTU        ]
    ──┤ ├────────────── Counter: Parts_Counter
                         Preset:  100
                         Accum:   0

Rung 1: Batch Complete เมื่อนับครบ 100
    Parts_Counter.DN      Batch_Complete_Light
    ──┤ ├────────────────────( )──

Rung 2: Reset Counter เมื่อเริ่ม Batch ใหม่
    New_Batch_PB          [RES        ]
    ──┤ ├────────────── Counter: Parts_Counter
```

### CTD - Count Down

```
Rung 0: Load Preset
    Load_Trigger          [MOV        ]
    ──┤ ├────────────── Source: 50
                         Dest:   Parts_To_Go

Rung 1: Count Down
    Part_Out_Sensor       [CTD        ]
    ──┤ ├────────────── Counter: Remaining_Parts
                         Preset:  Parts_To_Go
                         Accum:   0

Rung 2: Empty Alert
    Remaining_Parts.DN    Empty_Alert
    ──┤ ├────────────────────( )──
```

---

## Step 318: OTL/OTU - Latch Instructions

### ความแตกต่างของ OTE, OTL, OTU

```
OTE (Output Energize): ──( )──
- ทำงานเหมือน Relay Coil ธรรมดา
- TRUE เมื่อ Rung TRUE, FALSE เมื่อ Rung FALSE
- เหมาะสำหรับ Output ที่ไม่ต้อง "จำ" สถานะ

OTL (Output Latch): ──(L)──
- SET Coil ให้ TRUE
- ค้างอยู่ TRUE แม้ Rung กลับ FALSE
- ต้องใช้ OTU เพื่อยกเลิก

OTU (Output Unlatch): ──(U)──
- RESET Coil ให้ FALSE
- ใช้คู่กับ OTL

ตัวอย่างใช้งาน:
```

### ตัวอย่าง Alarm System ด้วย OTL/OTU

```
// ระบบ Alarm ที่ต้องยืนยันการรับรู้ (Acknowledge)

Rung 0: Alarm Detection (OTL - Latch เมื่อเกิด Alarm)
    Temp_High_Switch    E_Stop_Pressed     Alarm_Active
    ──┤ ├──────┬──────────┤ ├─────────────────(L)──
               │
    Pressure_High
    ──┤ ├──────┘

Rung 1: Alarm Acknowledge (OTU - Unlatch เมื่อ ACK)
    ACK_Button    Alarm_Active      Alarm_Active
    ──┤ ├──────────┤ ├─────────────────(U)──

Rung 2: Alarm Horn ดังตลอดจนกว่า ACK
    Alarm_Active          Alarm_Horn
    ──┤ ├────────────────────( )──

Rung 3: Alarm Light กะพริบเมื่อ Active, ค้างเมื่อ ACK แล้ว
    Alarm_Active    Alarm_ACK_Done     Flash_Light
    ──┤ ├────────────┤/├─────────────────( )──

    Alarm_ACK_Done        Steady_Light
    ──┤ ├────────────────────( )──
```

---

## Step 319: Math และ Comparison Instructions

### Math Instructions

```
ADD - บวก
    Trigger              [ADD        ]
    ──┤ ├──────────── Source A: Value_A
                      Source B: Value_B
                      Dest:     Result

SUB - ลบ | MUL - คูณ | DIV - หาร

MOD - หารเอาเศษ (Modulo):
    Trigger              [MOD        ]
    ──┤ ├──────────── Source A: Parts_Count
                      Source B: 10
                      Dest:     Remainder
// Remainder = Parts_Count MOD 10

SQR - Square Root:
    Trigger              [SQR        ]
    ──┤ ├──────────── Source: Flow_Squared
                      Dest:   Flow_Rate

ABS - Absolute Value:
    Trigger              [ABS        ]
    ──┤ ├──────────── Source: Error_Value
                      Dest:   Error_Abs

CPT - Compute (Expression):
    Trigger              [CPT        ]
    ──┤ ├──────────── Dest: Result
                      Expression: (A*B + C) / D
// CPT ยืดหยุ่นสุด ใช้ Expression ได้
```

### Comparison Instructions

```
EQU  : Equal          (A == B)
NEQ  : Not Equal      (A != B)
GRT  : Greater Than   (A > B)
GEQ  : Greater or Equal (A >= B)
LES  : Less Than      (A < B)
LEQ  : Less or Equal  (A <= B)
LIM  : Limit Test     (Low <= Test <= High)

ตัวอย่าง LIM (Range Check):
Rung 0: ตรวจสอบว่า Speed อยู่ใน Range
    [LIM                   ]     Speed_OK
    ── Low Limit:  800    ──────────( )──
       Test:       Motor_Speed
       High Limit: 1800

// หาก 800 <= Motor_Speed <= 1800 → Speed_OK = TRUE

ตัวอย่าง EQU:
Rung 0: ตรวจสอบ Recipe Number
    [EQU            ]       Recipe_1_Active
    ── Source A: Recipe_No ──────────( )──
       Source B: 1
```

---

## Step 320: MOV และ Data Handling

### MOV - Move (Copy) Data

```
MOV - ย้ายข้อมูล:
    Trigger              [MOV        ]
    ──┤ ├──────────── Source: 1500
                      Dest:   Speed_SP
// ตั้ง Speed_SP = 1500

MVM - Masked Move (คัดลอกเฉพาะบาง Bit):
    Trigger              [MVM        ]
    ──┤ ├──────────── Source: Status_Word
                      Mask:   16#00FF  (เฉพาะ Byte ล่าง)
                      Dest:   Status_Low_Byte

CLR - Clear (Set to 0):
    Trigger              [CLR        ]
    ──┤ ├──────────── Dest: Error_Code
```

### FIFO และ LIFO Instructions

```
FIFO (First In First Out):
- FFU: FIFO Load (เพิ่ม Element)
- FFD: FIFO Unload (เอา Element แรกออก)

LIFO (Last In First Out):
- LFU: LIFO Load
- LFD: LIFO Unload

ใช้สำหรับ:
- Queue Management
- Shift Register ของข้อมูล
- Recipe Queue

ตัวอย่าง Queue System:
    New_Order             [FFU        ]
    ──┤ ├──────────── Source:  Order_ID
                      FIFO:    Order_Queue
                      Control: FIFO_Control
                      Length:  100
                      Position: 0

    Process_Next          [FFD        ]
    ──┤ ├──────────── FIFO:    Order_Queue
                      Dest:    Current_Order
                      Control: FIFO_Control
                      Length:  100
                      Position: 0
```

---

## Step 321: User Defined Types (UDT)

### การสร้าง UDT

```
UDT ใน AB = Struct ใน Siemens
ใช้สร้าง Data Structure ที่ใช้ซ้ำได้

การสร้าง UDT Motor:
1. Controller Organizer → Data Types → User-Defined
2. New Data Type: "MOTOR_STATUS"

MOTOR_STATUS UDT:
┌───────────────────────────────────────────────────────┐
│ MOTOR_STATUS (User-Defined Type)                      │
├────────────────┬──────────┬──────────────────────────┤
│ Name           │ Data Type│ Description               │
├────────────────┼──────────┼──────────────────────────┤
│ Running        │ BOOL     │ Motor is running          │
│ Faulted        │ BOOL     │ Motor fault active        │
│ Manual_Mode    │ BOOL     │ Manual control mode       │
│ Auto_Mode      │ BOOL     │ Auto control mode         │
│ Speed_Actual   │ REAL     │ Actual speed RPM          │
│ Speed_Command  │ REAL     │ Commanded speed RPM       │
│ Run_Hours      │ DINT     │ Total running hours       │
│ Fault_Code     │ INT      │ Last fault code           │
└────────────────┴──────────┴──────────────────────────┘

การใช้งาน:
Controller Tags → Add Tag:
  Name: Motor1
  Type: MOTOR_STATUS
  → อัตโนมัติมี Motor1.Running, Motor1.Speed_Actual ฯลฯ
```

### Array of UDT

```
การสร้าง Array of Motor (10 motors):
  Name: All_Motors
  Type: MOTOR_STATUS[10]
  
การเข้าถึง:
  All_Motors[0].Running      // Motor 1 status
  All_Motors[1].Speed_Actual // Motor 2 speed
  All_Motors[9].Fault_Code   // Motor 10 fault

ใช้ FOR Loop ตรวจสอบทุก Motor:
(ใน Structured Text routine)
FOR Motor_Index := 0 TO 9 DO
    IF All_Motors[Motor_Index].Faulted THEN
        Any_Motor_Fault := TRUE;
    END_IF;
END_FOR;
```

---

## Step 322: Add-On Instructions (AOI)

### AOI คืออะไร?

```
AOI = Add-On Instruction
คล้ายกับ FB ของ Siemens แต่ใช้งานเป็น Instruction

ข้อดี:
1. Reusable เหมือน Standard Instruction
2. มี Documentation ใน Instruction เอง
3. ทดสอบได้แยกจาก Project หลัก
4. Distribute ให้ทีมได้
5. Version Control

ความแตกต่างจาก Subroutine:
Subroutine: เรียกใช้เป็น JSR/RET
AOI: ใช้เป็น Instruction ใน Rung โดยตรง
```

### การสร้าง AOI สำหรับ Motor Control

```
สร้าง AOI ชื่อ "Motor_Controller":

Parameters:
┌────────────────┬──────────┬───────┬────────────────────┐
│ Name           │ Type     │ Usage │ Description        │
├────────────────┼──────────┼───────┼────────────────────┤
│ EnableIn       │ BOOL     │ Input │ (Built-in)         │
│ EnableOut      │ BOOL     │ Output│ (Built-in)         │
│ Start_Cmd      │ BOOL     │ Input │ Start command      │
│ Stop_Cmd       │ BOOL     │ Input │ Stop command       │
│ E_Stop         │ BOOL     │ Input │ Emergency stop     │
│ Fault_Input    │ BOOL     │ Input │ External fault     │
│ Reset_Cmd      │ BOOL     │ Input │ Reset fault        │
│ Motor_Output   │ BOOL     │ Output│ Motor contactor    │
│ Fault_Output   │ BOOL     │ Output│ Fault indication   │
│ Status         │ DINT     │ Output│ Status word        │
│ Start_Timer    │ TIMER    │ Local │ Start delay timer  │
│ Fault_Latch    │ BOOL     │ Local │ Fault memory       │
└────────────────┴──────────┴───────┴────────────────────┘

Logic ใน AOI:
Rung 0: E-Stop Priority
    .E_Stop               .Motor_Output
    ──┤ ├────────────────────(U)──

Rung 1: Start with Delay
    .Start_Cmd    .Fault_Latch    .E_Stop     [TON      ]
    ──┤ ├─────────┤/├────────────┤/├────── Timer: .Start_Timer
                                             Preset: 2000
                                             Accum: 0

Rung 2: Motor Enable
    .Start_Timer.DN    .Stop_Cmd     .Motor_Output
    ──┤ ├────────────────┤/├───────────────( )──
    
    .Motor_Output
    ──┤ ├──────────────────────────────────┘

Rung 3: Fault Detection
    .Fault_Input              .Fault_Latch
    ──┤ ├─────────────────────────(L)──
    
    .Reset_Cmd                .Fault_Latch
    ──┤ ├─────────────────────────(U)──
```

### การใช้ AOI ใน Main Program

```
// ใช้งาน AOI เหมือน Standard Instruction

Rung 0: Motor 1
    [Motor_Controller                         ]
    M1_Start────┤Start_Cmd    Motor_Output├──── M1_Output
    M1_Stop─────┤Stop_Cmd     Fault_Output├──── M1_Fault
    E_Stop_Btn──┤E_Stop       Status      ├──── M1_Status
    M1_OL───────┤Fault_Input              │
    M1_Reset────┤Reset_Cmd               │

Rung 1: Motor 2 (ใช้ AOI เดิม)
    [Motor_Controller                         ]
    M2_Start────┤Start_Cmd    Motor_Output├──── M2_Output
    M2_Stop─────┤Stop_Cmd     Fault_Output├──── M2_Fault
    E_Stop_Btn──┤E_Stop                   │
    M2_OL───────┤Fault_Input              │
    M2_Reset────┤Reset_Cmd               │
```

---

## Step 323: การ Download และ Go Online

### RSLinx / FactoryTalk Linx Setup

```
ก่อน Download ต้อง Configure Communication Driver:

RSLinx Classic:
1. เปิด RSLinx Classic
2. Configure Drivers → Add New Driver
3. เลือก "EtherNet/IP Driver"
4. Name: "AB-ETH"
5. กำหนด IP Range: 192.168.1.1 ถึง 192.168.1.254
6. Click Apply

ใน RSLinx RSWho (Topology View):
┌──────────────────────────────────────────┐
│ RSWho                                    │
│ ▼ Workstation (192.168.1.100)            │
│   ▼ AB-ETH Driver                        │
│     ▼ 192.168.1.10 (CompactLogix)        │
│       BackPlane - CompactLogix 5380      │
│         [0] 1769-L33ER (CPU)             │
│         [1] 1769-IQ16 (Input)            │
│         [2] 1769-OB16 (Output)           │
└──────────────────────────────────────────┘
```

### Download Program

```
ขั้นตอน Download:
1. Studio 5000 → Communications → Who Active
2. เลือก CPU จาก Browse
3. Click "Download"

Mode Options:
- Download: ส่ง Program ทั้งหมด (ต้อง STOP ก่อน)
- Go Online with Download: Download แล้ว Go Online
- Update Controller (Changes only): ส่งเฉพาะที่เปลี่ยน

Safety Warning:
⚠️ ตรวจสอบ Mode Switch บน CPU: REM/PROG/RUN
- PROG: หยุดทุกอย่าง (ปลอดภัยสุดสำหรับ Download)
- REM: Remote control จาก Software
- RUN: Hardware RUN (Software ไม่สามารถ Stop ได้)
```

### Go Online และ Monitoring

```
After Download:
1. Controller → Transition to RUN
2. Communications → Go Online (Ctrl+W)

Online Mode แสดง:
- Rung สีเขียว = Rung TRUE (Power Flow มาถึง)
- Rung สีขาว/เทา = Rung FALSE
- ค่า Tag แสดงใน [] ใต้ Tag Name

Run 0: (Online)
    Start_PB    E_Stop       Stop_PB    Motor_Coil
    [TRUE]      [FALSE]      [FALSE]    [TRUE]
    ──┤ ├──┬────┤/├────────────┤/├────────( )──
    ████████│████████████████████████████████████
           │
    Motor_Coil
    [TRUE]
    ──┤ ├──┘
    ████████

// เส้นหนา (████) = Power flow TRUE
// เส้นบาง = Power flow FALSE
```

---

## Step 324: Watch Windows และ Trend Tool

### Controller Tags Monitor

```
Tag Browser (Online):
Controller Tags → Right-click → Monitor Tags

┌────────────────────────────────────────────────────────┐
│ Tag Monitor                                            │
├────────────────┬─────────┬───────────┬────────────────┤
│ Tag Name       │ Value   │ Force Mask│ Style          │
├────────────────┼─────────┼───────────┼────────────────┤
│ Start_PB       │ FALSE   │ N/A       │ Decimal        │
│ Motor_Coil     │ TRUE    │ N/A       │ Decimal        │
│ Motor_Timer.ACC│ 2345    │ N/A       │ Decimal        │
│ Motor_Speed    │ 1487    │ N/A       │ Decimal        │
│ Temperature    │ 72.5    │ N/A       │ Float          │
└────────────────┴─────────┴───────────┴────────────────┘
```

### Force Values

```
⚠️ ระวัง: Force ค่า I/O ในระบบที่กำลังทำงานอาจเป็นอันตราย

การ Force I/O:
1. Click ที่ Tag ที่ต้องการ Force
2. Right-click → Force

Force Options:
- Force to 1 (TRUE)
- Force to 0 (FALSE)
- Enable Forces (ต้อง Enable ก่อน Force จะมีผล)
- Remove Forces

Enable Forces (จำเป็น):
Controller → Enable Forces (Force จะไม่มีผลจนกว่าจะ Enable)

Force Indicator:
I Forced = F ปรากฏที่ Rung

การ Clear Force ทั้งหมด:
Controller → Remove All Forces
```

### Trend Tool

```
Trend Tool ใช้ Plot ค่า Tag ตามเวลา:

Setup Trend:
1. Logic → Trends → New Trend
2. Add Tags ที่ต้องการ Plot
3. กำหนด Sample Rate: 100ms, 500ms, 1s
4. Duration: 60 seconds

ใช้สำหรับ:
- Debug PLC ที่มีปัญหา Intermittent
- ดู Process Variable เปลี่ยนแปลงตามเวลา
- Tune PID Control

ตัวอย่างที่ควร Plot:
- Temperature (Process Variable)
- Heater Output (%)
- Speed Command vs Actual Speed
- Current Consumption
```

---

## Step 325: Structured Text (ST) ใน AB

### ทำไมต้องใช้ ST?

```
Ladder Diagram เหมาะกับ:
- Boolean Logic
- Timer/Counter Control
- Sequential Control

Structured Text เหมาะกับ:
- Math Computations ซับซ้อน
- Loop (FOR, WHILE)
- State Machine
- String Processing
- ลดจำนวน Rung ได้มาก

AB รองรับ ST เป็น Routine ประเภทหนึ่ง
สามารถใช้ ST และ LAD ใน Program เดียวกันได้
```

### ตัวอย่าง Structured Text Routine

```
(* Production Statistics Calculation *)
(* คำนวณสถิติการผลิต *)

(* คำนวณ OEE = Availability × Performance × Quality *)

(* Availability *)
IF Scheduled_Time > 0 THEN
    Availability := (Scheduled_Time - Downtime) / Scheduled_Time * 100.0;
ELSE
    Availability := 0.0;
END_IF;

(* Performance *)
IF (Scheduled_Time - Downtime) > 0 THEN
    Performance := (Parts_Produced * Ideal_Cycle_Time) / 
                   (Scheduled_Time - Downtime) * 100.0;
ELSE
    Performance := 0.0;
END_IF;

(* Quality *)
IF Parts_Produced > 0 THEN
    Quality := DINT_TO_REAL(Good_Parts) / 
               DINT_TO_REAL(Parts_Produced) * 100.0;
ELSE
    Quality := 0.0;
END_IF;

(* OEE *)
OEE := (Availability * Performance * Quality) / 10000.0;

(* Clamp to 0-100% *)
IF OEE > 100.0 THEN OEE := 100.0; END_IF;
IF OEE < 0.0   THEN OEE := 0.0;   END_IF;
```

---

## Step 326: SFC - Sequential Function Chart

### การใช้ SFC สำหรับ Process Sequence

```
SFC (Sequential Function Chart) เหมาะสำหรับ:
- Process ที่มีขั้นตอนชัดเจน
- Batch Process
- Robot Control Sequence
- ช่วยให้มองเห็น Flow ของ Process

SFC Structure:
- Step: สถานะ (มี Action)
- Transition: เงื่อนไขเปลี่ยน Step
- Branch: แยกขนาน (AND) หรือเลือก (OR)

ตัวอย่าง Batch Mixing Process:

    [Initial]
        │
        ↓ Start_PB
    [Step 1: Fill Water]
        │ Water_Level_OK
        ↓
    [Step 2: Heat to 80C]
        │ Temp_At_80C
        ↓
    [Step 3: Add Chemical]
        │ Chemical_Added
        ↓
    [Step 4: Mix 5 min]
        │ Mix_Timer.DN
        ↓
    [Step 5: Drain]
        │ Level_Empty
        ↓
    [Complete]
        │ Reset_PB
        ↓ (กลับ Initial)
```

---

## Step 327: Fault Handling และ Program Fault Routine

### Fault Routine

```
ทุก Program ควรมี Fault Routine:
Program → FaultRoutine (แยกจาก MainRoutine)

Fault Routine เรียกเมื่อ Major Fault:
- เปลี่ยน Major Fault เป็น Minor Fault (ถ้าทำได้)
- บันทึก Fault Information
- Set Safe State ของ Machine

ตัวอย่าง Fault Routine:
Rung 0: บันทึก Fault Code
    [MOV                    ]
    Source: Controller.ProgrammedFaultCodes
    Dest:   Last_Fault_Code

Rung 1: บันทึก Fault Time
    [COP                    ]
    Source: System_Time
    Dest:   Fault_Time
    Length: 1

Rung 2: Clear Fault (Minor Fault เท่านั้น)
    [CLR                    ]
    Dest: Controller.MinorFaultBits

Rung 3: Set Outputs ไปที่ Safe State
    [OTU                    ]
    Dest: All_Outputs_Enable
```

### Major vs Minor Fault

```
Major Fault: CPU เข้า FAULT Mode (หยุดทำงาน)
ตัวอย่าง:
- Program array out of bounds
- Divide by zero
- Watchdog timeout
- Hardware failure

Minor Fault: CPU ยังทำงาน แต่มี Flag
ตัวอย่าง:
- Connection timeout
- Tag not found in some routines

การ Clear Major Fault:
1. Physical: Mode Switch PROG → REM/RUN
2. Online: Controller Properties → Clear Faults
3. Program: Fault Routine + CLR instruction (limited)
```

---

## Step 328: Motion Control ใน ControlLogix

### Integrated Motion (IMC)

```
ControlLogix รองรับ Motion Control ผ่าน SERCOS หรือ EtherNet/IP

Kinetix Servo Drive ต่อผ่าน:
- SERCOS II/III (Fiber Optic)
- EtherNet/IP (Category ล่าสุด)

Motion Instructions:
MAM : Motion Axis Move (Absolute Position)
MAJ : Motion Axis Jog (Continuous)
MAS : Motion Axis Stop
MAH : Motion Axis Home
MAR : Motion Axis Run (Velocity Mode)

ตัวอย่าง Servo Motion:
Rung 0: Axis Power On
    Enable_Drive         [MSO           ]
    ──┤ ├───────────── Axis: X_Axis
                       Execute: TRUE

Rung 1: Home Axis
    Home_CMD    X_Axis.ServoActionStatus   [MAH        ]
    ──┤ ├─────────────┤/├─────────────── Axis: X_Axis
                                          Direction: 0 (Forward)
                                          Speed: 100.0
                                          Home Mode: 0

Rung 2: Move to Position
    Move_CMD    X_Axis.PC                  [MAM        ]
    ──┤ ├─────────┤/├──────────────────── Axis: X_Axis
                                          Position: Target_Pos
                                          Speed: 500.0
                                          Accel: 1000.0
                                          Decel: 1000.0
```

---

## Step 329: EtherNet/IP และ CIP

### EtherNet/IP Protocol

```
EtherNet/IP (Ethernet Industrial Protocol) ใช้ CIP (Common Industrial Protocol)
ทำงานบน TCP/IP + UDP มาตรฐาน

ประเภท Connection:
1. Implicit Messaging (I/O): Real-time, Cyclic
   - ส่งข้อมูล I/O อัตโนมัติ
   - Response time: 1-500ms

2. Explicit Messaging: On-demand, Request/Response
   - อ่าน/เขียนข้อมูลเฉพาะ
   - ใช้สำหรับ Configuration, Data Collection

MSG Instruction สำหรับ Explicit Messaging:
    Trigger              [MSG           ]
    ──┤ ├──────────── Message: Config_Msg
                      (Message Configuration:
                       Type: CIP Generic
                       Destination: 192.168.1.20
                       Service: Get_Attribute
                       Class: 4, Instance: 1, Attribute: 3)

ใช้สำหรับ:
- อ่านข้อมูลจาก Drive (Current, Voltage, Speed)
- เขียน Parameter ไปยัง Drive
- Communicate กับ SCADA/MES ผ่าน CIP
```

---

## Step 330: โปรแกรมตัวอย่างสมบูรณ์ - Bottle Filling Line

### Project Overview

```
ระบบ Bottle Filling Line:
- Station 1: Empty Bottle Detection
- Station 2: Filling (Flow Control)
- Station 3: Capping
- Station 4: Labeling
- Reject Station: ขวดที่ไม่ผ่านมาตรฐาน

Controller: CompactLogix 5380 L33ER

I/O Configuration:
Slot 1 (1769-IQ16): 16 Digital Inputs
  Local:1:I.Data.0  Bottle_Present_S1
  Local:1:I.Data.1  Bottle_Filled_S2
  Local:1:I.Data.2  Cap_Detected_S3
  Local:1:I.Data.3  Label_Applied_S4
  Local:1:I.Data.4  Weight_OK
  Local:1:I.Data.5  Start_PB
  Local:1:I.Data.6  Stop_PB
  Local:1:I.Data.7  E_Stop (NC)

Slot 2 (1769-OB16): 16 Digital Outputs
  Local:2:O.Data.0  Conveyor_Run
  Local:2:O.Data.1  Fill_Valve
  Local:2:O.Data.2  Cap_Actuator
  Local:2:O.Data.3  Label_Motor
  Local:2:O.Data.4  Reject_Gate
  Local:2:O.Data.5  Ready_Light
  Local:2:O.Data.6  Fault_Light

Slot 3 (1769-IF4): 4 Analog Inputs
  Local:3:I.Ch0Data  Fill_Weight (0-10V → 0-1000g)
```

### UDT และ AOI Design

```
UDT: BOTTLE_STATUS
  At_Station_1 : BOOL
  At_Station_2 : BOOL
  Filled        : BOOL
  Capped        : BOOL
  Labeled       : BOOL
  Weight        : REAL
  Rejected      : BOOL

Controller Tags:
  System_Running  : BOOL
  Line_Speed      : DINT   // ความเร็ว Conveyor (bottles/min)
  Fill_Target     : REAL   // น้ำหนักที่ต้องการ (กรัม)
  Total_Bottles   : DINT   // จำนวนทั้งหมด
  Good_Bottles    : DINT   // ผ่านมาตรฐาน
  Reject_Bottles  : DINT   // ไม่ผ่าน

Program Structure:
  MainRoutine (LAD): Sequence Control
  Fill_Control (LAD): Filling Station Control
  Weight_Check (ST): Weight Validation + Statistics
  Alarms (LAD): Alarm Management
```

---

## ข้อผิดพลาดที่พบบ่อยใน Allen-Bradley

```
1. "Tag does not exist"
   สาเหตุ: พิมพ์ชื่อ Tag ผิด หรือยังไม่ได้สร้าง Tag
   แก้ไข: ตรวจสอบ Tag Browser, สร้าง Tag ที่ถูกต้อง

2. "I/O Module not found" (Controller Fault)
   สาเหตุ: Module ขาดหายหรือ Firmware ไม่ตรง
   แก้ไข: ตรวจสอบ I/O Config, อัปเดต Firmware

3. Timer/Counter ไม่นับ
   สาเหตุ: ลืม Enable Rung (Rung ก่อนหน้า FALSE)
   แก้ไข: ตรวจสอบ Power Flow ถึง TON/CTU หรือไม่

4. Force ไม่มีผล
   สาเหตุ: ลืม Enable Forces
   แก้ไข: Controller → Enable Forces

5. Major Fault หลัง Download
   สาเหตุ: Array Index Out of Bounds, Divide by Zero
   แก้ไข: ดู Controller Fault ใน Module Properties

6. AOI Parameter ไม่อัปเดต
   สาเหตุ: ลืม Update เมื่อเปลี่ยน AOI Definition
   แก้ไข: Right-click AOI → Update All Instances
```

---

## Lab Projects

### Lab 17-1: Conveyor Control System

```
โจทย์: ระบบสายพาน 2 เส้นพร้อม Junction

Requirements:
1. สายพาน A และ B ส่งชิ้นงานมา Junction
2. Junction เลือกส่งไปทาง C หรือ D ตาม Recipe
3. จำนวนชิ้นงานที่ส่งแต่ละทางต้องนับแยก
4. มี Speed Control (3 ระดับ: Low/Medium/High)
5. Jam Detection (ถ้า Sensor ค้าง > 10 วินาที)

Tag Design (ออกแบบเอง):
- ใช้ UDT สำหรับ Conveyor Status
- ใช้ AOI สำหรับ Conveyor Control

เป้าหมาย:
- สร้าง UDT CONVEYOR
- สร้าง AOI Conveyor_Ctrl
- OEE Calculation ใน ST Routine
```

### Lab 17-2: PID Temperature Control

```
โจทย์: ควบคุมอุณหภูมิ Oven

Hardware:
- AI: Thermocouple (4-20mA → 0-400°C)
- AO: Heater Control (0-10V → 0-100% Power)
- DI: Over-temperature Switch (NC)
- DO: Heater Enable, Alarm

Requirements:
1. PID Control (ใช้ PIDE Instruction)
2. Auto/Manual Mode Switch
3. Setpoint Ramp (ค่อยๆ เพิ่มจนถึง SP)
4. Over-temperature Alarm (OTL)
5. Trend ของ PV, SP, Output

PIDE (Enhanced PID):
- รองรับ Bumpless Transfer
- Feed-forward
- Anti-windup
- Output Limits
```

---

## สรุปและทบทวน

**ความรู้ที่ได้รับจาก Part 17:**

1. **Studio 5000** เป็น IDE สำหรับ Logix Controllers ทุกรุ่น
2. **Tag-Based Programming** ไม่มี Physical Address เลย
3. **UDT** ช่วยสร้าง Data Structure ที่ Reusable
4. **AOI** คือ Custom Instruction ที่มี Documentation ในตัว
5. **OTL/OTU** ใช้สำหรับ Latching แทน Memory Bit แบบ Siemens
6. **Task/Program/Routine** Hierarchy ช่วยจัดระเบียบ Code
7. **EtherNet/IP** เป็น Protocol มาตรฐานสำหรับ I/O และ Communication
8. **Structured Text** ช่วยเขียน Algorithm ที่ซับซ้อนได้ดีกว่า LAD

---

**[← Part 16: Siemens S7](part16_siemens_s7.md) | [Part 18: Mitsubishi →](part18_mitsubishi.md)**

*เนื้อหา Part 17 ครอบคลุม Steps 311-330*
*ลิขสิทธิ์ © 2024 Ladder Diagram PLC Course*
