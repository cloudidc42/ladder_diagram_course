# Part 16: Siemens S7-1200/1500 - การเขียนโปรแกรม PLC ระดับมืออาชีพ

## Steps 291-310: จาก TIA Portal สู่การใช้งานจริงในโรงงาน

**[← Part 15: HMI Basics](part15_hmi_basics.md) | [Part 17: Allen-Bradley →](part17_allen_bradley.md)**

---

## บทนำ: ทำไมต้องเรียน Siemens S7?

Siemens เป็นผู้ผลิต PLC ที่มีส่วนแบ่งตลาดสูงสุดในยุโรปและมีฐานลูกค้าขนาดใหญ่ทั่วโลก รวมถึงในประเทศไทยที่มีโรงงานอุตสาหกรรมหลายแห่งใช้ Siemens S7-1200 และ S7-1500 โดยเฉพาะในอุตสาหกรรมยานยนต์ อาหารและเครื่องดื่ม และระบบสาธารณูปโภค

**ข้อดีของ Siemens S7:**
- TIA Portal เป็น Software ที่ครบครัน (PLC + HMI + Drive ในที่เดียว)
- รองรับทั้ง LAD, FBD, STL, SCL, และ GRAPH
- มี Library พร้อมใช้งานมากมาย
- OPC UA Built-in สำหรับ IIoT
- ความน่าเชื่อถือสูง (MTBF สูงมาก)

---

## Step 291: การติดตั้งและเริ่มต้น TIA Portal

### ความต้องการระบบ (System Requirements)

```
TIA Portal V17 / V18:
- OS: Windows 10/11 (64-bit)
- RAM: 16 GB ขึ้นไป (แนะนำ 32 GB)
- HDD: 50 GB ว่าง (SSD แนะนำ)
- CPU: Intel Core i7 หรือเทียบเท่า
- Display: 1920x1080 ขึ้นไป
- .NET Framework 4.8
```

### การติดตั้ง TIA Portal

```
ขั้นตอนการติดตั้ง:
1. ดาวน์โหลด TIA Portal จาก Siemens Industry Online Support
2. Disable antivirus ชั่วคราวระหว่างติดตั้ง
3. Run installer ในฐานะ Administrator
4. เลือก Components ที่ต้องการ:
   ✓ STEP 7 Professional (สำหรับ S7-1200/1500)
   ✓ WinCC Advanced (สำหรับ HMI)
   ✓ PLCSIM Advanced (สำหรับ Simulation)
   ✓ Startdrive (สำหรับ Drives)
5. ติดตั้ง License Manager
6. Restart Windows
```

### หน้าจอ TIA Portal

```
╔══════════════════════════════════════════════════════════════╗
║  TIA Portal - Start Page                                      ║
╠══════════════════════════════════════════════════════════════╣
║  ┌─────────────┐  ┌──────────────────────────────────────┐  ║
║  │ Open recent │  │  Create new project                  │  ║
║  │ project     │  │  ┌──────────────────────────────┐   │  ║
║  │             │  │  │ Project name: [__________]   │   │  ║
║  │ ○ Project1  │  │  │ Path: [C:\TIA_Projects\___]  │   │  ║
║  │ ○ Project2  │  │  │ Author: [______________]     │   │  ║
║  │ ○ Project3  │  │  │ Comment: [_____________]     │   │  ║
║  │             │  │  │                              │   │  ║
║  │             │  │  │  [Create]  [Browse...]       │   │  ║
║  └─────────────┘  │  └──────────────────────────────┘   │  ║
║                   └──────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Step 292: S7-1200 Hardware Overview

### ครอบครัว CPU S7-1200

```
รุ่น CPU S7-1200 และคุณสมบัติหลัก:

┌────────────┬──────────┬──────────┬──────────┬──────────────┐
│ CPU Model  │ DI/DO    │ AI/AO    │ Memory   │ Speed        │
├────────────┼──────────┼──────────┼──────────┼──────────────┤
│ CPU 1211C  │ 6DI/4DO  │ 2AI/-    │ 50KB     │ 0.1µs/instr  │
│ CPU 1212C  │ 8DI/6DO  │ 2AI/-    │ 75KB     │ 0.1µs/instr  │
│ CPU 1214C  │ 14DI/10DO│ 2AI/2AO  │ 100KB    │ 0.1µs/instr  │
│ CPU 1215C  │ 14DI/10DO│ 2AI/2AO  │ 125KB    │ 0.1µs/instr  │
│ CPU 1217C  │ 14DI/10DO│ 2AI/2AO  │ 150KB    │ 0.1µs/instr  │
└────────────┴──────────┴──────────┴──────────┴──────────────┘

หมายเหตุ:
- ทุกรุ่นมี PROFINET port 1 port (CPU 1215C/1217C มี 2 port)
- รองรับ Signal Boards (SB) เพิ่ม I/O ใน Housing เดิม
- รองรับ Signal Modules (SM) ขยาย I/O ด้านขวา
- รองรับ Communication Modules (CM) ด้านซ้าย
```

### โครงสร้างทางกายภาพของ S7-1200

```
การจัดวาง Hardware บน DIN Rail:

← CM (ซ้ายสุด)    CPU (กลาง)    SM (ขวา) →
┌──────────┐  ┌──────────────┐  ┌──────┐  ┌──────┐
│ CM 1241  │  │  CPU 1214C   │  │SM 1221│  │SM 1222│
│ RS485    │  │  ┌──────────┐│  │16DI  │  │16DO  │
│          │  │  │ Display  ││  │      │  │      │
│ [PORT1]  │  │  │ LED status││ │24VDC │  │24VDC │
│          │  │  └──────────┘│  │      │  │      │
│          │  │  [PROFINET]  │  │      │  │      │
└──────────┘  └──────────────┘  └──────┘  └──────┘

แต่ละ CM/SM เชื่อมต่อกันผ่าน U-Connector ด้านหลัง
```

### การต่อสาย I/O พื้นฐาน

```
การต่อ Digital Input (Sink/Source):

Source Wiring (PNP Sensor):           Sink Wiring (NPN Sensor):
+24VDC ───────────────────┐          +24VDC ─────────────────┐
                          │                                    │
PNP Sensor ────┬─ I0.0 ──┘          NPN Sensor─┬─ I0.0 ──── M
(OUT +)        │                    (OUT -)     │
               └─ M ─ 0VDC          0VDC ───────┘

การต่อ Digital Output (Relay/Transistor):
+24VDC ── Q0.0 ── [Load] ── M
```

---

## Step 293: การสร้าง Project ใน TIA Portal

### ขั้นตอนสร้าง Project ใหม่

```
1. เปิด TIA Portal
2. Click "Create new project"
3. กรอกข้อมูล:
   - Project name: "Learning_S7_1200"
   - Path: C:\TIA_Projects\
   - Author: ชื่อของคุณ
4. Click "Create"

หน้าจอ Project View จะเปิดขึ้น:

╔══════════════════════════════════════════════════════════╗
║ TIA Portal - Learning_S7_1200                           ║
╠═══════════╦══════════════════════════════╦══════════════╣
║ Project   ║ Work Area                    ║ Inspector    ║
║ Tree      ║                              ║ Window       ║
║           ║                              ║              ║
║ ▼ Project ║ [Add new device]             ║ Properties   ║
║   ▼ Devices║                             ║              ║
║    Add...  ║                             ║ Info         ║
║   Common  ║                              ║              ║
║   ▶ Lang. ║                              ║ Diagnostics  ║
╚═══════════╩══════════════════════════════╩══════════════╝
```

### การเพิ่ม CPU ใน Project

```
ขั้นตอน:
1. Double-click "Add new device"
2. เลือก Controller → SIMATIC S7-1200 → CPU
3. เลือกรุ่น CPU (เช่น CPU 1214C DC/DC/DC)
   - DC/DC/DC = Power/Input/Output ทั้งหมดเป็น DC
   - DC/DC/RLY = DC Power, DC Input, Relay Output
   - AC/DC/RLY = AC Power, DC Input, Relay Output
4. เลือก Firmware Version (แนะนำ V4.5 ล่าสุด)
5. Click "Add"

Hardware Configuration จะแสดงขึ้น:

┌─────────────────────────────────────────────────────────┐
│ Device configuration - PLC_1 [CPU 1214C DC/DC/DC]      │
├──────────┬────────────────────────────────────────────┤
│ Hardware │  [Rack view]                               │
│ Catalog  │  ┌──────┬────────────────┬──────┬──────┐  │
│          │  │  3   │  CPU 1214C     │  4   │  5   │  │
│ CPU 1214C│  │(CM)  │ [ETHERNET]     │ (SM) │ (SM) │  │
│ SM 1221  │  │      │ [DI14/DO10]    │      │      │  │
│ SM 1222  │  │      │ [AI2/AO2]      │      │      │  │
│ CM 1241  │  └──────┴────────────────┴──────┴──────┘  │
└──────────┴────────────────────────────────────────────┘
```

### การตั้งค่า IP Address ของ CPU

```
1. Click ที่ CPU ใน Hardware view
2. Inspector window → Properties → PROFINET interface
3. กำหนด IP address:
   - IP address: 192.168.1.10
   - Subnet mask: 255.255.255.0
4. Compile & Save

หมายเหตุ: PC ที่ใช้ต้องอยู่ใน Subnet เดียวกัน
เช่น PC: 192.168.1.100, Subnet: 255.255.255.0
```

---

## Step 294: Tag Table - การกำหนดชื่อสัญญาณ

### ความสำคัญของ Tag Table

Tag Table ใน TIA Portal ช่วยให้เขียนโปรแกรมโดยใช้ชื่อที่เข้าใจง่ายแทนที่อยู่แบบ I0.0, Q0.0 ซึ่งเป็นหลักการ **Symbolic Addressing**

### การสร้าง Tag Table

```
ใน Project Tree:
PLC_1
  └── PLC tags
       └── Default tag table ← Double-click เพื่อเปิด

หน้าจอ Tag Table:
┌───────────────────────────────────────────────────────────┐
│ Default tag table                                          │
├───────────────┬──────────────────┬──────────┬────────────┤
│ Name          │ Data type        │ Address  │ Comment    │
├───────────────┼──────────────────┼──────────┼────────────┤
│ Start_PB      │ Bool             │ %I0.0    │ ปุ่มสตาร์ท │
│ Stop_PB       │ Bool             │ %I0.1    │ ปุ่มสต็อป  │
│ E_Stop        │ Bool             │ %I0.2    │ ฉุกเฉิน    │
│ Motor_Run     │ Bool             │ %Q0.0    │ คอยล์มอเตอร์│
│ Motor_Fault   │ Bool             │ %Q0.1    │ ไฟแสดงขัดข้อง│
│ Run_Timer     │ TON_TIME         │ %DB1     │ Timer วิ่ง  │
│ Speed_SP      │ Int              │ %MW10    │ Setpoint   │
└───────────────┴──────────────────┴──────────┴────────────┘
```

### ประเภท Data Types ใน S7-1200

```
Bit Types:
- Bool    : 1 bit (TRUE/FALSE)

Integer Types:
- Byte    : 8 bit  (0 ถึง 255)
- Word    : 16 bit (0 ถึง 65535)
- DWord   : 32 bit (0 ถึง 4,294,967,295)
- Int     : 16 bit (-32768 ถึง 32767)
- DInt    : 32 bit (-2,147,483,648 ถึง 2,147,483,647)
- LInt    : 64 bit
- USInt   : 8 bit unsigned
- UInt    : 16 bit unsigned
- UDInt   : 32 bit unsigned

Real Types:
- Real    : 32 bit float (±3.4 × 10^38)
- LReal   : 64 bit double float

Time Types:
- Time    : 32 bit (T#-24d_20h_31m_23s_648ms ถึง T#24d...)
- Date    : 16 bit
- Time_Of_Day : 32 bit
- LTime   : 64 bit

String Types:
- String  : ความยาวสูงสุด 254 chars
- WString : Unicode 16 bit chars

S7-1500 เพิ่มเติม:
- LWord, LInt, ULInt, LReal, LTime, LDATE_AND_TIME
```

---

## Step 295: LAD Programming ใน TIA Portal

### การสร้าง Program Block

```
ประเภท Block ใน TIA Portal:

OB (Organization Block): จุดเริ่มต้น Program
  - OB1: Main Program Cycle
  - OB30-OB38: Cyclic Interrupt
  - OB40-OB47: Hardware Interrupt
  - OB80: Time Error
  - OB82: Diagnostic Interrupt
  - OB86: Rack Failure
  - OB100: Startup

FC (Function): Subroutine ไม่มีหน่วยความจำเป็นของตัวเอง
FB (Function Block): Subroutine มี Data Block (Instance DB) เป็นของตัวเอง
DB (Data Block): หน่วยความจำเก็บข้อมูล

การสร้าง OB1:
1. Project Tree → PLC_1 → Program blocks
2. Double-click "Main [OB1]"
3. หน้าจอ LAD Editor เปิดขึ้น
```

### โครงสร้าง Rung ใน TIA Portal

```
LAD Editor Interface:

┌─────────────────────────────────────────────────────────────┐
│ Main [OB1] - LAD                                            │
├────────┬────────────────────────────────────────────────────┤
│Toolbar │ [Contact][Coil][Box][Branch][Network]              │
├────────┴────────────────────────────────────────────────────┤
│ Network 1: Motor Start/Stop Circuit                         │
│                                                             │
│  "Start_PB"    "E_Stop"          "Motor_Run"                │
│    ──┤├─────────┤/├──────────────────( )──                  │
│                │                                            │
│  "Motor_Run"   │                                            │
│    ──┤├────────┘                                            │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ Network 2: Fault Indicator                                  │
│                                                             │
│  "Motor_Fault_Internal"       "Motor_Fault"                 │
│    ──┤├──────────────────────────( )──                      │
└─────────────────────────────────────────────────────────────┘
```

### การเขียนโปรแกรม Motor Start/Stop

```
// ตัวอย่างโปรแกรม Motor Control แบบ Seal-In Circuit
// Network 1: Start/Stop Motor with Seal-In

Network 1:
    "Start_PB"    "E_Stop"            "Motor_Run"
    ──┤├──┬────────┤/├─────────────────────( )──
          │
    "Motor_Run"
    ──┤├──┘

// Network 2: Run Timer - นับเวลาที่มอเตอร์วิ่ง
Network 2:
    "Motor_Run"
    ──┤├──────────────────[TON]──────────────────
                           Timer: "Run_Timer"
                           PT: T#8H        // 8 ชั่วโมง
                           ET: "Running_Time"
                           Q: "Maint_Due"  // แจ้งเตือนซ่อมบำรุง

// Network 3: Maintenance Due Alarm
Network 3:
    "Maint_Due"               "Maint_Alarm"
    ──┤├────────────────────────( )──

    "Motor_Fault"             "Fault_Alarm"
    ──┤├────────────────────────( )──
```

---

## Step 296: Timer Instructions ใน S7-1200

### ประเภท Timer ใน S7-1200 (IEC Standard)

S7-1200 ใช้ IEC Timer ซึ่งแตกต่างจาก Timer แบบเก่าของ S7-300/400

```
4 ประเภท Timer ที่ใช้บ่อย:

1. TON (Timer On-Delay):
   - IN=TRUE → เริ่มนับ → ครบ PT → Q=TRUE
   - IN=FALSE → Q=FALSE, ET=0 (Reset)
   
   ┌────────────┐
   "Start" ┤IN    Q├── "Delayed_Output"
           ┤PT   ET├── "Elapsed_Time"
   T#5S ───┘ TON  └
   
2. TOF (Timer Off-Delay):
   - IN=TRUE → Q=TRUE ทันที
   - IN=FALSE → เริ่มนับ → ครบ PT → Q=FALSE
   
3. TP (Timer Pulse):
   - IN=TRUE (Rising Edge) → Q=TRUE นาน PT แล้ว FALSE
   - ไม่ถูก Reset ระหว่างนับ
   
4. TONR (Timer Retentive On-Delay):
   - สะสมเวลา R=FALSE
   - R=TRUE → Reset ET=0

ความแตกต่างจาก S7-300:
- S7-300: T0, T1, T2... (Timer ผูกกับ Address)
- S7-1200: FB Timer ทุก Instance มี DB เป็นของตัวเอง
- Resolution: 1ms (S7-300 มี 10ms, 100ms, 1s, 10s)
```

### ตัวอย่างการใช้ TON Timer

```
// โปรแกรม: เปิดมอเตอร์หลังกดปุ่ม 3 วินาที (Anti-Tamp Protection)

// จำเป็นต้องสร้าง DB สำหรับ Timer:
// Project Tree → PLC_1 → Program blocks → Add new block → DB
// หรือใช้ Multi-instance DB ใน FB

Network 1: Start Delay Timer
    "Start_PB"          [TON]
    ──┤├──────────────[IN    Q]────── "Motor_Enable"
                    [PT   ET]────── "Start_Delay_Time"
              T#3S──┘   TON

Network 2: Motor Output
    "Motor_Enable"    "E_Stop"      "Motor_Out"
    ──┤├─────────────┤/├────────────────( )──
    
// การสร้าง Timer ใน FB (แนะนำ):
// เมื่อสร้าง FB และเรียกใช้ Timer ภายใน
// Timer จะถูกเก็บใน Instance DB ของ FB นั้นโดยอัตโนมัติ
```

---

## Step 297: Counter Instructions ใน S7-1200

### ประเภท Counter (IEC Standard)

```
3 ประเภท Counter:

1. CTU (Count Up):
   CU: Count Up Input (Rising Edge)
   R: Reset Input
   PV: Preset Value
   Q: Output (Q=TRUE เมื่อ CV >= PV)
   CV: Current Value
   
   ┌─────────┐
   ┤CU     Q├── "Count_Done"
   ┤R     CV├── "Current_Count"
   ┤PV     └
   100────┘ CTU

2. CTD (Count Down):
   CD: Count Down Input
   LD: Load Input
   PV: Preset Value
   Q: Output (Q=TRUE เมื่อ CV <= 0)
   CV: Current Value

3. CTUD (Count Up/Down):
   CU: Count Up
   CD: Count Down
   R: Reset
   LD: Load
   PV: Preset Value
   QU: Q Up (CV >= PV)
   QD: Q Down (CV <= 0)
   CV: Current Value
```

### ตัวอย่าง Production Counter

```
// โปรแกรม: นับจำนวนชิ้นงานที่ผลิต

// Tag Table:
// Part_Sensor   : Bool : %I0.0  (Proximity Sensor)
// Reset_Counter : Bool : %I0.1  (ปุ่ม Reset)
// Target_Count  : Int  : %MW20  (จำนวนเป้าหมาย)
// Current_Count : Int  : %MW22  (จำนวนปัจจุบัน)
// Batch_Done    : Bool : %Q0.3  (ไฟ Batch เสร็จ)

Network 1: Production Counter
    "Part_Sensor" │CTU
    ───────────┤CU      Q├──── "Batch_Done"
    "Reset_Counter"┤R      CV├──── "Current_Count"
    "Target_Count"─┘CTU    └

Network 2: Reset Output เมื่อ Reset
    "Reset_Counter"     "Batch_Done"
    ──┤├────────────────────(R)──

// ข้อสังเกต:
// S7-1200 CTU นับด้วย Rising Edge
// CV เป็น Int หรือ UInt ขึ้นกับการตั้งค่า
// สูงสุด 32767 (Int) หรือ 65535 (UInt)
```

---

## Step 298: MOVE และ Comparison Instructions

### MOVE Instruction

```
MOVE ใช้คัดลอกค่าจาก Source ไปยัง Destination

ตัวอย่าง:
Network 1: คัดลอกค่า Speed Setpoint
    "Write_Speed"         [MOVE]
    ──┤├──────────────[EN    ENO]──
               100──[IN     OUT]──── "Motor_Speed"

// MOVE หลาย Output (S7-1200 V4.1+):
Network 2: คัดลอกค่าไปหลายที่
    "Copy_Trigger"        [MOVE]
    ──┤├──────────────[EN    ENO]──
         "Source_Value"──[IN   OUT1]──── "Dest_1"
                              [OUT2]──── "Dest_2"
                              [OUT3]──── "Dest_3"
```

### Comparison Instructions

```
S7-1200 รองรับ Comparison แบบ Contact (In-Line):

ประเภท Comparison:
- ==  (Equal)
- <>  (Not Equal)
- >   (Greater Than)
- <   (Less Than)
- >=  (Greater Than or Equal)
- <=  (Less Than or Equal)

ตัวอย่าง: เปิด Alarm เมื่ออุณหภูมิสูงเกิน 80°C

Network 1: Temperature High Alarm
                     ┌────┐
    "Temp_Enable" ───┤>=  ├─── "Temp_High_Alarm"
   "Temperature" ───┤    │
        80 ─────────┘Int │
                         └─────────────( )──

// หรือแบบ Box:
    "Temp_Enable"    [CMP>=Int]     "Temp_High_Alarm"
    ──┤├──────────[EN   ENO]─────────────( )──
    "Temperature"─[IN1       ]
           80────[IN2       ]

// ตัวอย่าง Range Check:
Network 2: ตรวจสอบว่าค่าอยู่ใน Range (50-100)
                   ┌───┐       ┌───┐
    "Check" ──┬────┤ >= ├────┬─┤ <= ├─── "In_Range"
              │    │ 50 │    │ │100 │
    "Value" ──┘    └───┘    └─└───┘
```

---

## Step 299: Data Block (DB) - การสร้างและใช้งาน

### ประเภท Data Block

```
1. Global DB: เก็บข้อมูลที่ใช้ร่วมกันทั้งโปรแกรม
2. Instance DB: สร้างอัตโนมัติเมื่อเรียก FB
3. Array DB (S7-1500): DB ที่เป็น Array ของ UDT เดียว

การสร้าง Global DB:
1. Project Tree → Program blocks → Add new block
2. เลือก "Data block (DB)"
3. ตั้งชื่อ: "Motor_Data"
4. เลือก "Global DB"
5. Click OK
```

### โครงสร้าง DB

```
โครงสร้าง DB Motor_Data:
┌──────────────────────────────────────────────────────────┐
│ Motor_Data [DB1]                                          │
├──────────────┬──────────────┬──────────┬─────────────────┤
│ Name         │ Data type    │ Default  │ Comment         │
├──────────────┼──────────────┼──────────┼─────────────────┤
│ Motor_Speed  │ Real         │ 0.0      │ ความเร็วมอเตอร์  │
│ Running_Hours│ Real         │ 0.0      │ ชั่วโมงวิ่ง       │
│ Fault_Code   │ Int          │ 0        │ รหัสขัดข้อง      │
│ Setpoint     │ Real         │ 50.0     │ ค่าที่ตั้ง        │
│ Auto_Mode    │ Bool         │ FALSE    │ โหมดอัตโนมัติ     │
│ ▼ Config    │ Struct       │          │ การตั้งค่า        │
│   Max_Speed  │   Real       │ 100.0    │ ความเร็วสูงสุด   │
│   Min_Speed  │   Real       │ 10.0     │ ความเร็วต่ำสุด   │
│   Ramp_Time  │   Time       │ T#5S     │ เวลา Ramp       │
└──────────────┴──────────────┴──────────┴─────────────────┘

การเข้าถึงใน Program:
"Motor_Data".Motor_Speed := 75.5;     // กำหนดค่า
"Motor_Data".Config.Max_Speed         // เข้าถึง Struct
```

### Optimized Block Access vs Standard Access

```
TIA Portal มี 2 โหมด Block Access:

1. Optimized Block Access (แนะนำ):
   - ข้อมูลเรียงตาม Type (ประหยัด Memory)
   - ไม่มี Absolute Address
   - เข้าถึงผ่านชื่อเท่านั้น
   - ได้รับประสิทธิภาพดีกว่า

2. Standard Access (Legacy):
   - ข้อมูลเรียงตาม Offset เหมือน S7-300
   - มี Absolute Address (DBW0, DBD2, ฯลฯ)
   - ใช้สำหรับ HMI รุ่นเก่าหรือ S7 Communication

การเลือก:
- Block Properties → Attributes → Optimized block access
```

---

## Step 300: Symbolic Addressing - การใช้ชื่อแทนที่อยู่

### หลักการ Symbolic Addressing

```
แบบ Absolute (ไม่แนะนำ):
    %I0.0    %Q0.0    %MW10
    ──┤├────────( )──   MOVE 100 → %MW10

แบบ Symbolic (แนะนำ):
    "Start_PB"   "Motor_Run"
    ──┤├────────────( )──   MOVE 100 → "Speed_SP"

ข้อดีของ Symbolic Addressing:
1. อ่านเข้าใจง่าย
2. เปลี่ยน Hardware ได้ง่าย (เปลี่ยนแค่ Tag Table)
3. ลด Error จากการพิมพ์ผิด Address
4. Software Documentation ดีขึ้น
5. Collaborative Programming ง่ายขึ้น
```

### การใช้ Cross-Reference

```
TIA Portal มี Cross-Reference Tool:
Tools → Cross-reference (Ctrl+Shift+Alt+X)

แสดงข้อมูล:
- ใช้ Tag ไหนที่ไหนบ้าง
- Block ไหนเรียก Block ไหน
- Address ซ้ำซ้อนหรือไม่

ตัวอย่าง Cross-Reference:
Tag: Motor_Run
┌──────────────────────────────────────────────────────┐
│ Used in:                                             │
├──────────────┬────────────┬──────────────────────────┤
│ Object       │ Network    │ Usage                    │
├──────────────┼────────────┼──────────────────────────┤
│ Main [OB1]   │ Network 1  │ Coil (Write)             │
│ Main [OB1]   │ Network 2  │ Contact (Read)           │
│ Safety [FB2] │ Network 3  │ Contact (Read)           │
│ HMI_DB [DB5] │ -          │ HMI Tag                  │
└──────────────┴────────────┴──────────────────────────┘
```

---

## Step 301: การ Compile และ Download Program

### การ Compile

```
Compile มี 2 ระดับ:
1. Software (symbols): ตรวจสอบ Program Logic เท่านั้น
2. Hardware: ตรวจสอบ Hardware Configuration + Program

ขั้นตอน:
1. เลือก PLC_1 ใน Project Tree
2. คลิก Compile → Hardware and software (only changes)
   หรือ Ctrl+B

ผลลัพธ์ใน Output window:
┌──────────────────────────────────────────────────────┐
│ Compile results                                      │
├─────────────────────────────────────────────────────┤
│ ✓ Compilation finished (Errors: 0, Warnings: 2)    │
│ ⚠ Warning: Tag "Temp_Sensor" not used              │
│ ⚠ Warning: Network 5 has no comment                │
└─────────────────────────────────────────────────────┘
```

### การ Download Program

```
ขั้นตอน Download:
1. ต่อ PROFINET Cable (หรือใช้ USB Programming Cable)
2. คลิก Download to device → Hardware and software
   หรือ Ctrl+L
3. หน้าต่าง "Extended download to device":
   ┌─────────────────────────────────────────────┐
   │ Configure PC interface                      │
   │ PG/PC interface: Intel Ethernet Card        │
   │ Connection: PN/IE                           │
   │ [Start search]                              │
   │                                             │
   │ Found devices:                              │
   │ ○ PLC_1 [192.168.1.10] - CPU 1214C         │
   │                                             │
   │ [Load]                                      │
   └─────────────────────────────────────────────┘
4. Click "Load" → Confirm "Start module" → Finish

โหมดการ Download:
- STOP mode: ต้องหยุด PLC ก่อน
- RUN mode: บางส่วน Download ได้ระหว่าง Run (Hot Reload)
```

---

## Step 302: Online Monitoring และ Diagnostics

### Online Mode

```
เมื่อ PLC Running และ Connect แล้ว:
1. คลิก "Go online" (Ctrl+K) หรือ Online → Go online
2. สีของ Block เปลี่ยนเป็น สีส้ม = Online

Status indicator ใน Project Tree:
● สีเขียว = Online, Running OK
● สีส้ม = Online, มี Warning
● สีแดง = Online, มี Error/Fault
○ สีเทา = Offline
```

### LAD Monitor Mode

```
ใน Online Mode LAD จะแสดงสถานะ Real-time:

Network 1 (Online Monitoring):
  "Start_PB"=TRUE   "E_Stop"=FALSE        "Motor_Run"=TRUE
  ──┤├ TRUE──────────┤/├ TRUE──────────────────( ) TRUE──
  (สีน้ำเงิน=TRUE)  (สีขาว=FALSE)          (สีน้ำเงิน)
        │
  "Motor_Run"=TRUE
  ──┤├ TRUE──┘

การอ่าน Status:
- เส้นสีน้ำเงิน/เหลือง = Power Flow มาถึง (TRUE)
- เส้นสีขาว = No Power Flow (FALSE)
- ค่าตัวเลขแสดงใต้ Symbol
```

### Watch Tables

```
Watch Table ใช้ Monitor และ Force ค่าตัวแปร

สร้าง Watch Table:
1. Project Tree → Monitoring & forcing tables
2. Add new watch table: "Motor_Watch"
3. เพิ่ม Variables ที่ต้องการ Monitor

┌────────────────────────────────────────────────────────┐
│ Watch Table: Motor_Watch                               │
├──────────────┬────────┬───────────┬────────────────────┤
│ Name         │ Address│ Monitor   │ Modify             │
├──────────────┼────────┼───────────┼────────────────────┤
│ Start_PB     │ %I0.0  │ FALSE     │ [  ]               │
│ Motor_Run    │ %Q0.0  │ TRUE      │ [  ]               │
│ Motor_Speed  │ %MW10  │ 1500      │ [1500]             │
│ Temperature  │ %MD20  │ 65.3      │ [  ]               │
└──────────────┴────────┴───────────┴────────────────────┘

Buttons:
[Monitor all] = เริ่ม Monitor
[Modify all once] = เปลี่ยนค่าทีเดียว
[Modify now] = เปลี่ยนค่าทันที

⚠️ ระวัง: Force/Modify ขณะ Machine Running อาจเป็นอันตราย
```

---

## Step 303: S7-1200 Specific Instructions

### String Instructions

```
S7-1200 รองรับ String Operations:
(ต้องเปิด "String & Char" Library)

S_MOVE: คัดลอก String
    "Trigger"          [S_MOVE]
    ──┤├──────────[EN      ENO]──
    "Source_Str"──[IN      OUT]──── "Dest_Str"

S_LEN: หาความยาว String
    "Trigger"          [S_LEN]
    ──┤├──────────[EN      ENO]──
    "My_String"──[IN      OUT]──── "String_Length"

CONCAT: รวม String
    "Trigger"          [CONCAT]
    ──┤├──────────[EN      ENO]──
    "First_Name"──[IN1     OUT]──── "Full_Name"
    "Last_Name"───[IN2        ]
```

### Math Instructions

```
คณิตศาสตร์ใน S7-1200:

ADD: บวก          SUB: ลบ
MUL: คูณ          DIV: หาร
MOD: หารเอาเศษ    NEG: เปลี่ยนเครื่องหมาย
ABS: ค่าสัมบูรณ์  SQR: ยกกำลังสอง
SQRT: รากที่สอง   LN: Logarithm ธรรมชาติ
LOG: Log10        EXP: e^x
SIN: sin(x)       COS: cos(x)
TAN: tan(x)       ASIN, ACOS, ATAN

ตัวอย่าง: คำนวณ Flow Rate
Network 1: คำนวณ Flow Rate
    "Calc_Enable"      [MUL]
    ──┤├──────────[EN      ENO]──
    "Pressure"────[IN1     OUT]──── "Flow_Raw"
    "K_Factor"────[IN2        ]

Network 2: Scale Flow Rate
    "Calc_Enable"      [DIV]
    ──┤├──────────[EN      ENO]──
    "Flow_Raw"────[IN1     OUT]──── "Flow_Rate"
    1000.0────────[IN2        ]
```

---

## Step 304: Interrupt Organization Blocks

### Hardware Interrupt (OB40-OB47)

```
Hardware Interrupt ใช้ตอบสนองต่อ Input เปลี่ยนแปลง
โดยไม่ต้องรอ Scan Cycle

การตั้งค่า:
1. Hardware configuration → CPU → Digital inputs
2. เลือก Input ที่ต้องการ
3. Enable "Hardware interrupt"
4. กำหนด OB ที่จะเรียก (OB40)
5. เลือก Trigger: Rising edge / Falling edge

OB40 (Hardware Interrupt):
Network 1: บันทึกเวลาเมื่อ Product ผ่าน Sensor
    [RD_SYS_T]        // อ่าน System Time
    [MOVE]            // บันทึกใน DB

ข้อดี: Response time < 1ms (vs Scan time 10-20ms)
ใช้กับ: High-speed pulse counting, Safety input
```

### Cyclic Interrupt (OB30)

```
Cyclic Interrupt เรียกตาม Period ที่กำหนด

การตั้งค่า OB30:
1. Add new block → Organization block
2. เลือก "Cyclic interrupt"
3. Cyclic time: 100ms (เรียกทุก 100ms)

ใช้สำหรับ:
- PID Control (ต้องการ Sampling time คงที่)
- Data Logging ตาม Period
- Watchdog functions

OB30 (100ms Cyclic):
Network 1: PID Temperature Control
    [CONT_C]         // Continuous PID Control
    Setpoint: "Temp_SP"
    Process: "Temp_PV"
    Output: "Heater_Output"
```

---

## Step 305: Function Block (FB) - การเขียน Reusable Code

### ทำไมต้องใช้ FB?

```
ข้อดีของ FB vs FC:
1. FB มี Instance DB เก็บสถานะ
2. เรียกใช้หลาย Instance (หลาย Motor ได้จาก FB เดียว)
3. Timer/Counter ใน FB ทำงานได้ถูกต้อง
4. Parameter ครบถ้วน (IN, OUT, IN_OUT, STAT, TEMP)

ตัวอย่าง FB สำหรับ Motor Control:
```

### การสร้าง FB Motor Controller

```
สร้าง FB ชื่อ "Motor_Ctrl":

Interface ของ FB:
┌────────────────────────────────────────────────┐
│ Input:                                         │
│  Start_Cmd    : Bool  // คำสั่งสตาร์ท          │
│  Stop_Cmd     : Bool  // คำสั่งสต็อป           │
│  Fault_Input  : Bool  // Fault จากภายนอก       │
│  Reset_Cmd    : Bool  // คำสั่ง Reset          │
│                                                │
│ Output:                                        │
│  Motor_Output : Bool  // Output มอเตอร์         │
│  Fault_Output : Bool  // Output Fault           │
│  Running      : Bool  // สถานะวิ่ง              │
│                                                │
│ Static:                                        │
│  Run_Timer    : TON   // Timer สำหรับ Anti-Tamper│
│  Fault_Timer  : TOF   // Timer สำหรับ Fault     │
│  Fault_Memory : Bool  // จำสถานะ Fault          │
└────────────────────────────────────────────────┘

LAD Program ใน FB:
Network 1: Motor Control Logic
    #Start_Cmd    #Fault_Memory    #Stop_Cmd  #Motor_Output
    ──┤├──────┬────┤/├──────────────┤/├──────────( )──
              │
    #Motor_Output
    ──┤├──────┘

Network 2: Anti-Tamper Start Timer
    #Start_Cmd           [TON]
    ──┤├──────────────[IN    Q]────── Enable_Start
                   [PT   ET]
             T#2S──┘ TON  └

Network 3: Fault Detection
    #Fault_Input          #Fault_Memory
    ──┤├────────────────────────(S)──
    
    #Reset_Cmd            #Fault_Memory
    ──┤├────────────────────────(R)──
```

### การเรียกใช้ FB ใน OB1

```
การเรียก FB สร้าง Instance DB อัตโนมัติ:

Network 1: Motor 1 Control
    [Motor_Ctrl]                Instance DB: "Motor1_DB"
    "M1_Start"────┤Start_Cmd Motor_Output├──── "M1_Output"
    "M1_Stop"─────┤Stop_Cmd  Fault_Output├──── "M1_Fault"
    "M1_Fault"────┤Fault_Input  Running  ├──── "M1_Running"
    "M1_Reset"────┤Reset_Cmd              │

Network 2: Motor 2 (ใช้ FB เดิม คนละ Instance)
    [Motor_Ctrl]                Instance DB: "Motor2_DB"
    "M2_Start"────┤Start_Cmd Motor_Output├──── "M2_Output"
    "M2_Stop"─────┤Stop_Cmd  Fault_Output├──── "M2_Fault"
    ...

// ข้อสังเกต: FB เดียวกัน แต่สถานะแยกกัน
// Motor1_DB.Run_Timer != Motor2_DB.Run_Timer
```

---

## Step 306: Library ใน TIA Portal

### การใช้ Standard Library

```
TIA Portal มี Library พร้อมใช้งาน:

เปิด Library:
View → Libraries หรือ Ctrl+Shift+L

Library ที่มีใน TIA Portal V17+:
1. Standard library:
   - LGF (Library of General Functions): Math, String
   - Drives library: สำหรับ Siemens Drives
   - PID: PID Control blocks
   - Communication: สำหรับ PROFIBUS, PROFINET

2. Global Library vs Project Library:
   - Global: ใช้ได้ทุก Project
   - Project: ใช้เฉพาะ Project นี้

การ Drag & Drop จาก Library:
1. เปิด Library window
2. Find Block ที่ต้องการ
3. Drag ไปวางใน LAD Editor
```

### PID Control ด้วย PID_Compact

```
PID_Compact เป็น FB สำหรับ PID Control:

การใช้งาน:
1. Library → PID Control → Continuous control → PID_Compact
2. Drag ลงใน OB30 (Cyclic 100ms)

Interface PID_Compact:
    [PID_Compact]
    Setpoint ─────┤Setpoint    Output├──── "Heater_PWM"
    "Temp_PV" ────┤Input              │
    TRUE ─────────┤ManualEnable       │
    0.0 ──────────┤ManualValue        │
    FALSE ────────┤Reset              │
                  └───────────────────┘

การ Tune PID:
1. Online → PID Parameter ใน TIA Portal
2. เลือก "Start pretuning"
3. ระบบจะทดสอบและหา Kp, Ti, Td อัตโนมัติ

หรือ Manual Tune:
- Kp: Proportional Gain
- Ti: Integration Time (วินาที)
- Td: Derivative Time (วินาที)

ตัวอย่างค่า PID สำหรับ Temperature Control:
Kp = 2.0, Ti = 120s, Td = 30s
```

---

## Step 307: S7-1500 Advanced Features Overview

### S7-1500 vs S7-1200

```
เปรียบเทียบ:
┌──────────────────────┬─────────────────┬──────────────────┐
│ Feature              │ S7-1200         │ S7-1500          │
├──────────────────────┼─────────────────┼──────────────────┤
│ CPU Speed            │ 0.1µs/bit inst  │ 1ns/bit inst     │
│ Program Memory       │ ถึง 150KB       │ ถึง 60MB         │
│ Data Memory          │ ถึง 4MB         │ ถึง 8GB (ET200MP)│
│ PROFINET ports       │ 1-2 (Copper)    │ 2+ (incl. SFP)  │
│ OPC UA Server        │ V4.4+           │ ใช่ (Built-in)   │
│ Motion Control       │ Basic           │ Advanced         │
│ Safety               │ -               │ F-CPU            │
│ Display              │ Text display    │ TFT Color display│
│ Trace function       │ -               │ ใช่              │
│ Technology Objects   │ Basic           │ Advanced         │
└──────────────────────┴─────────────────┴──────────────────┘
```

### S7-1500 Unique Features

```
1. Technology Objects (TO):
   - Axis TO: Servo Motor Control
   - External Encoder TO
   - Cam TO: Electronic Cam
   
   ตัวอย่าง Motion Control:
   [MC_Power]      // เปิด Axis
   [MC_Home]       // กำหนด Home Position
   [MC_MoveAbsolute] // เคลื่อนที่ไป Position ที่กำหนด
   [MC_MoveRelative] // เคลื่อนที่ระยะทางที่กำหนด

2. Trace Function:
   - บันทึก Variable ตาม Trigger
   - แสดงเป็น Waveform
   - ช่วย Debug Transient Problems

3. SIMATIC Memory Card:
   - SD Card สำหรับ Backup Program
   - Program Transfer ง่าย
   - Boot from Card

4. OPC UA:
   - Server Built-in (ไม่ต้องซื้อ Software เพิ่ม)
   - Connect กับ SCADA, MES, ERP
   - IIoT Ready

5. Security:
   - Know-How Protection (เข้ารหัส Block)
   - Copy Protection (ผูกกับ CPU Serial)
   - Access Levels (Password)
```

---

## Step 308: การจัดการ Alarm และ Diagnostics

### CPU Diagnostics Buffer

```
อ่าน Diagnostic Buffer:
Online → Diagnostics buffer (Alt+D)

แสดง Event ล่าสุด:
┌──────────────────────────────────────────────────────────┐
│ Diagnostic buffer - PLC_1                               │
├────────┬────────────┬────────────────────────────────────┤
│ No.    │ Timestamp  │ Event                             │
├────────┼────────────┼────────────────────────────────────┤
│ 1      │ 14:23:05   │ Mode transition STOP to RUN       │
│ 2      │ 14:22:58   │ Download: Software                │
│ 3      │ 13:45:12   │ I/O access error: Channel 3       │
│ 4      │ 13:44:55   │ Mode transition RUN to STOP       │
│ 5      │ 08:01:23   │ Power ON                          │
└────────┴────────────┴────────────────────────────────────┘
```

### Alarm System ใน TIA Portal

```
S7-1200/1500 มี Alarm System สำหรับ HMI:

การสร้าง Program Alarm:
1. เพิ่ม Alarm Instruction ใน LAD:

    "Temp_High"           [ALARM_DQ]
    ──┤├──────────────[SIG     ENO]──
              "Temp_High_Alarm"──[ALARM_ID]
    
2. กำหนด Alarm Text ใน TIA Portal:
   PLC tags → HMI alarms → Program alarms
   
3. Alarm Text (หลายภาษา):
   EN: "Temperature too high: %1"
   TH: "อุณหภูมิสูงเกินกำหนด: %1"

ProDiag (S7-1500):
- Automatic Process Diagnostics
- ตรวจจับ Wiring Error, Sensor Fail
- Alarm ที่มี Context Information ครบถ้วน
```

---

## Step 309: Networking และ Communication

### PROFINET IO

```
PROFINET คือ Industrial Ethernet Standard ของ Siemens

Components:
- IO Controller: CPU (S7-1200/1500)
- IO Device: Remote I/O (ET 200, Drives)
- IO Supervisor: TIA Portal สำหรับ Config

การตั้งค่า PROFINET IO:
1. Hardware configuration
2. Drag IO Device จาก Hardware catalog
3. กำหนด IP address ให้ IO Device
4. กำหนด PROFINET name (สำคัญมาก!)
5. Compile & Download

IO Cycle:
- Update time ขั้นต่ำ: 0.25ms (ขึ้นกับ CPU)
- Typical: 1-4ms

ET 200SP (PROFINET IO Device):
┌─────────────┐    PROFINET    ┌───────────────────┐
│ S7-1500     │◄──────────────►│ ET 200SP           │
│ IO Controller│               │ IM 155-6 PN HF     │
│             │               │ ┌──────┬──────────┐ │
│             │               │ │DI 8  │DQ 8      │ │
│             │               │ │24VDC │24VDC     │ │
└─────────────┘               └─└──────┴──────────┘┘
```

### S7 Communication

```
PUT/GET Instructions สำหรับ S7 CPU-to-CPU:

PUT (เขียนข้อมูลไปยัง PLC อื่น):
    "Trigger"          [PUT]
    ──┤├──────────[REQ     DONE]────
    1.0 ─────────[ID      ERROR]────
    "Remote_DB"──[ADDR_1  STATUS]───
    "Local_Data"─[SD_1         ]

GET (อ่านข้อมูลจาก PLC อื่น):
    "Trigger"          [GET]
    ──┤├──────────[REQ     DONE]────
    1.0 ─────────[ID      ERROR]────
    "Remote_DB"──[ADDR_1  STATUS]───
                 [RD_1         ]──── "Local_Storage"

ตัวอย่างการใช้งาน:
- Line 1 PLC ส่งข้อมูล Production ไปยัง Line 2 PLC
- ส่ง Setpoint จาก Supervisor PLC ไปยัง Slave PLC
```

---

## Step 310: โปรแกรมตัวอย่างสมบูรณ์ - Conveyor System

### Project Overview

```
ระบบ Conveyor พร้อม QC Station:
- Conveyor 1 (ป้อน)
- QC Camera Station (ตรวจสอบ)
- Conveyor 2 (ผ่าน) / Reject Chute (ไม่ผ่าน)
- Production Count & Report

Hardware:
I0.0: Start_PB          I1.0: QC_Pass_Signal
I0.1: Stop_PB           I1.1: QC_Fail_Signal
I0.2: E_Stop            I1.2: Part_At_QC_Sensor
I0.3: CV1_Fault_Signal  I1.3: CV2_Fault_Signal
I0.4: Part_Entry_Sensor

Q0.0: CV1_Motor_Run     Q1.0: CV2_Motor_Run
Q0.1: CV1_Fast         Q1.1: Reject_Gate
Q0.2: System_Ready_Light Q1.2: Production_Alarm
Q0.3: QC_Trigger
```

### Program Structure

```
OB1 (Main):
  CALL FC10 (System_Status)   // ตรวจสถานะระบบ
  CALL FB20 (CV1_Control)     // ควบคุม Conveyor 1
  CALL FB21 (QC_Control)      // ควบคุม QC Station
  CALL FB22 (CV2_Control)     // ควบคุม Conveyor 2
  CALL FC30 (Production_Count) // นับ Production

DB100 (System_Data):
  System_Running: Bool
  Total_Parts: DInt
  Good_Parts: DInt
  Reject_Parts: DInt
  Shift_Start: Time_Of_Day
```

### FB20 - Conveyor 1 Control

```
// FB20: CV1_Control
// Input: Start_Cmd, Stop_Cmd, E_Stop, Fault_Input
// Output: Motor_Output, Fast_Speed, Fault_Output
// Static: Start_Timer (TON), Fault_Latch (Bool)

Network 1: E-Stop Priority
    #E_Stop               #Motor_Output
    ──┤├────────────────────────(R)──

Network 2: Start Conveyor
    #Start_Cmd    #E_Stop    #Fault_Latch     [TON IN]
    ──┤├────────┬──┤/├──────────┤/├──────────────────
               │                         T#2S──┘PT
    #Motor_Output                    [Q]────(S)──#Motor_Output
    ──┤├───────┘

Network 3: Stop Conveyor
    #Stop_Cmd             #Motor_Output
    ──┤├────────────────────────(R)──

Network 4: Speed Control (Fast when Load Light)
    #Motor_Output    "DB100".System_Running   #Fast_Speed
    ──┤├───────────────────────┤├──────────────────( )──
```

---

## ข้อผิดพลาดที่พบบ่อยและวิธีแก้

```
1. Error: "Address already used"
   สาเหตุ: กำหนด Address เดิมให้ 2 Output
   แก้ไข: ตรวจสอบ Cross-reference

2. Error: "DB not found"
   สาเหตุ: ลืม Compile หรือ Download DB
   แก้ไข: Compile Hardware and Software ใหม่

3. Warning: "Symbolic access without HW identifier"
   สาเหตุ: I/O ที่ใช้ไม่ได้กำหนด Hardware Module
   แก้ไข: เพิ่ม Module ใน Hardware Configuration

4. CPU เข้า STOP mode หลัง Download
   สาเหตุ: Program Error (เช่น Division by Zero)
   แก้ไข: ดู Diagnostic Buffer หา Error

5. Timer ไม่ทำงาน
   สาเหตุ: ใช้ Timer โดยตรงใน FC (ไม่มี Static)
   แก้ไข: ใช้ Timer ใน FB หรือสร้าง Global DB

6. I/O ไม่ตอบสนอง
   สาเหตุ: Force value ยังเปิดอยู่
   แก้ไข: Monitoring & forcing tables → Stop forcing
```

---

## Lab Projects

### Lab 16-1: Motor Starter with TIA Portal

```
โจทย์: สร้างโปรแกรม Motor Starter สมบูรณ์

Requirements:
1. Start/Stop ด้วย Pushbutton
2. Emergency Stop (NC Contact)
3. Overload Protection (NC Contact)  
4. Star-Delta Starting (Y → Δ หลัง 5 วินาที)
5. Run Indicator Light
6. Fault Indicator Light
7. Hour Meter (นับชั่วโมงวิ่ง)

I/O:
I0.0 Start, I0.1 Stop, I0.2 E-Stop, I0.3 Overload
Q0.0 Contactor_Main, Q0.1 Contactor_Star
Q0.2 Contactor_Delta, Q0.3 Run_Light, Q0.4 Fault_Light

เป้าหมาย:
- Y/Δ switching ต้องมี Interlock
- Fault ต้อง Latch จนกว่า Reset
- Hour Meter บันทึกลง DB
```

### Lab 16-2: Batch Process Control

```
โจทย์: ระบบ Batch Process อย่างง่าย

Process:
1. กด Start → เปิด Valve A (น้ำ) 
2. ระดับน้ำถึง Level H → ปิด Valve A
3. เปิด Heater → รอจนอุณหภูมิ 80°C
4. Stir 30 วินาที
5. Drain → รอจนระดับ Level L
6. รีเซ็ต พร้อม Cycle ถัดไป

I/O:
I0.0 Start, I0.1 Stop, I0.2 Level_H
I0.3 Level_L, I1.0 Temp_High (Thermostat)
Q0.0 Valve_A, Q0.1 Heater, Q0.2 Stirrer, Q0.3 Drain_Valve

พิเศษ: บันทึก Batch Count และ เวลาต่อ Batch ลง DB
```

---

## สรุปและทบทวน

**ความรู้ที่ได้รับจาก Part 16:**

1. **TIA Portal** เป็น Integrated Software สำหรับ PLC, HMI, Drive
2. **S7-1200** เหมาะกับ Machine-level Control ขนาดกลาง
3. **Tag Table** และ **Symbolic Addressing** เพิ่มความชัดเจนของ Program
4. **Data Blocks** แยก Data จาก Logic ได้อย่างมีระเบียบ
5. **FB** ใช้สร้าง Reusable Code ที่มี State
6. **IEC Timers/Counters** แตกต่างจาก S7-300 แบบเก่า
7. **Online Monitoring** และ **Watch Tables** สำคัญต่อการ Debug
8. **S7-1500** มีประสิทธิภาพสูงกว่า เหมาะกับงาน Advanced

---

**[← Part 15: HMI Basics](part15_hmi_basics.md) | [Part 17: Allen-Bradley →](part17_allen_bradley.md)**

*เนื้อหา Part 16 ครอบคลุม Steps 291-310*
*ลิขสิทธิ์ © 2024 Ladder Diagram PLC Course*
