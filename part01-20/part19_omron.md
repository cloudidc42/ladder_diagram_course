# Part 19: OMRON CP1H/CJ Series - CX-Programmer และ Sysmac Studio

## Steps 351-370: PLC มาตรฐานญี่ปุ่นที่มีความยืดหยุ่นสูง

**[← Part 18: Mitsubishi](part18_mitsubishi.md) | [Part 20: Debugging →](part20_debugging.md)**

---

## บทนำ: OMRON ในอุตสาหกรรมไทย

OMRON เป็น PLC ที่ใช้กันแพร่หลายในอุตสาหกรรมอิเล็กทรอนิกส์ อาหาร และบรรจุภัณฑ์ในประเทศไทย ชื่อเสียงด้านความน่าเชื่อถือและ Feature ที่ครบครันในราคาที่เหมาะสม

**ตระกูล OMRON PLC:**
- **CP1H/CP1L/CP1E**: Compact Series (เหมาะสำหรับ Machine Control)
- **CJ2M/CJ2H**: Modular Mid-range
- **CS1**: Legacy Large-scale
- **NX1/NJ**: Machine Automation Controller (Sysmac Platform ใหม่)
- **NX-SL Safety**: Safety PLC

---

## Step 351: การติดตั้ง CX-Programmer

### CX-Programmer Overview

```
CX-Programmer เป็นส่วนหนึ่งของ CX-One Suite:
- CX-Programmer: LAD Programming
- CX-Designer: HMI Design
- CX-Simulator: Simulation
- Network Configurator: Network Setup
- CX-Integrator: System Integration

System Requirements CX-Programmer:
- OS: Windows 7/8/10 (ใช้ CX-One V4.6+)
- RAM: 512MB ขั้นต่ำ (1GB แนะนำ)
- HDD: 3GB
- Display: 1024×768 ขึ้นไป

CX-One Lite: Version ฟรี (จำกัด Feature)
```

### หน้าจอ CX-Programmer

```
╔══════════════════════════════════════════════════════════════╗
║  CX-Programmer - [Project.cxp]                              ║
╠════════════╦═════════════════════════════╦══════════════════╣
║ Project    ║ Program Workspace           ║ I/O Comment     ║
║ Workspace  ║                             ║                 ║
║            ║ [Ladder Rung View]          ║                 ║
║ ▼ NewPLC1  ║                             ║ 0.00 Start_PB   ║
║   Settings ║ Rung 1:                     ║ 0.01 Stop_PB    ║
║   I/O Table║ ──┤0.00├──┤/0.01├──( 100.00)║ 0.02 E_Stop     ║
║   Symbol   ║                             ║                 ║
║   Programs ║ Rung 2:                     ║ 100.00 Motor_Run║
║    Section1║ ──┤100.00├──[TIM 000]──     ║ 100.01 Fault    ║
║            ║                             ║                 ║
╚════════════╩═════════════════════════════╩══════════════════╝
```

---

## Step 352: CP1H Hardware

### ตระกูล CP1H

```
CP1H Variants:
┌──────────────────┬────────────────────────────────────────┐
│ Model            │ คุณสมบัติ                              │
├──────────────────┼────────────────────────────────────────┤
│ CP1H-X40DT-D     │ 24 DI / 16 DO (Transistor)            │
│                  │ 4 High-speed Counter (100kHz)           │
│                  │ Pulse Output 2 axes (100kHz)           │
│                  │ Built-in USB, RS-232C                  │
├──────────────────┼────────────────────────────────────────┤
│ CP1H-X40DR-A     │ 24 DI / 16 DO (Relay)                 │
│                  │ สำหรับ Load AC/High Current            │
├──────────────────┼────────────────────────────────────────┤
│ CP1H-Y20DT-D     │ 12 DI / 8 DO (Transistor)             │
│                  │ Analog Option Port (2 AI, 1 AO)       │
├──────────────────┼────────────────────────────────────────┤
│ CP1H-XA40DT-D    │ 24 DI / 16 DO + Analog Port           │
│                  │ Built-in 4 AI / 2 AO                  │
└──────────────────┴────────────────────────────────────────┘
```

### โครงสร้าง CP1H

```
CP1H-X40DT-D Layout:

┌─────────────────────────────────────────────────────────┐
│                  CP1H-X40DT-D                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ LED: PWR RUN ERR INH USER COMM                      │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ INPUTS:                    OUTPUTS:                     │
│ 0.00-0.11 (CH0: 12 inputs) 100.00-100.07 (CH100: 8 DO) │
│ 1.00-1.11 (CH1: 12 inputs) 101.00-101.07 (CH101: 8 DO) │
│                                                         │
│ [DIP Switch]  [USB Port]  [RS-232C Port]               │
│                                                         │
│ EXPANSION:                                              │
│ CPM1A Expansion I/O ต่อได้ที่ Port ด้านข้าง             │
└─────────────────────────────────────────────────────────┘

Expansion Modules:
CPM1A-20EDT: 12 DI + 8 DO
CPM1A-8ED: 8 DI
CPM1A-8ET: 8 DO (Transistor)
CPM1A-MAD01: 2 AI + 1 AO
```

---

## Step 353: OMRON Addressing System

### ระบบ Address ของ OMRON

```
OMRON ใช้ Address แบบ Word.Bit (หรือ Channel.Bit)

หน้าที่ของแต่ละ Area:
┌────────────┬────────────────────────────────────────────┐
│ Area       │ รายละเอียด                                 │
├────────────┼────────────────────────────────────────────┤
│ I/O Bit    │ 0.00-179.15: I/O Area (ขึ้นกับรุ่น)       │
│ (IR)       │ Word (Channel): 0-179                      │
│            │ Bit: 00-15 (16 bits ต่อ Word)             │
├────────────┼────────────────────────────────────────────┤
│ IR Work    │ 1200.00-1535.15: Internal Relay             │
│ (W)        │ Work Area สำหรับ Internal Logic            │
├────────────┼────────────────────────────────────────────┤
│ Holding    │ H0.00-H511.15: Holding Relay               │
│ (H/HR)     │ เก็บข้อมูลระหว่าง Power Cycle             │
├────────────┼────────────────────────────────────────────┤
│ Auxiliary  │ A0.00-A959.15: Auxiliary Relay             │
│ (A/AR)     │ Status Flags, PLC System Information       │
├────────────┼────────────────────────────────────────────┤
│ Data Memory│ D00000-D32767: Data Memory (Word)          │
│ (DM)       │ General Purpose Data Storage               │
├────────────┼────────────────────────────────────────────┤
│ Extended   │ EM0_00000-EM... : Extended Data Memory     │
│ (EM)       │ Large Data Storage (EM Bank 0-C)           │
├────────────┼────────────────────────────────────────────┤
│ Timer      │ TIM 000-TIM 4095: Timer                    │
│            │ CNT 000-CNT 4095: Counter                  │
└────────────┴────────────────────────────────────────────┘
```

### การกำหนด I/O Address

```
CP1H I/O Mapping:
Input CH0 (0.00-0.11) → Physical Input 0-11
Input CH1 (1.00-1.11) → Physical Input 12-23 (ถ้ามี)
Output CH100 (100.00-100.07) → Physical Output 0-7
Output CH101 (101.00-101.07) → Physical Output 8-15

เพิ่ม Expansion Module:
Module 1 (ต่อ CP1H): Input CH2 (2.00-2.11), Output CH102 (102.00-102.07)
Module 2: Input CH3, Output CH103
ฯลฯ

การอ่านชื่อ Address:
0.00   = CH0 (Word 0), Bit 00 = Input 0
1.07   = CH1 (Word 1), Bit 07 = Input 19
100.03 = CH100 (Word 100), Bit 03 = Output 3
D00500 = Data Memory Word 500
H10.05 = Holding Relay Word 10, Bit 05
```

---

## Step 354: Basic Instructions ใน CX-Programmer

### การเขียน Ladder Diagram

```
OMRON Instructions เปรียบเทียบ:
Siemens    ↔  OMRON
──┤├──    ↔  LD (Load NO Contact)
──┤/├──   ↔  LDN (Load NC Contact)
──( )──   ↔  OUT (Output)
──(S)──   ↔  SET
──(R)──   ↔  RSET

ตัวอย่าง Program ใน OMRON:

Rung 1: Motor Start/Stop
   0.00       0.01       100.00
──┤├────┬─────┤/├──────────( )──
        │
   100.00
──┤├────┘

// อ่าน:
// Contact 0.00 (Start_PB) OR Contact 100.00 (Motor Seal-In)
// AND NOT Contact 0.01 (Stop_PB)
// → Output 100.00 (Motor_Coil)

Rung 2: Emergency Stop
   0.02              100.00
──┤├──────────────────(RSET)──
// Contact 0.02 (E_Stop) → RESET 100.00

Rung 3: Fault Indicator (SET)
   0.05              100.01
──┤├──────────────────(SET)──   // Latch Fault

Rung 4: Fault Reset
   0.06              100.01
──┤├──────────────────(RSET)── // Unlatch
```

---

## Step 355: I/O Bit Configuration

### การกำหนด I/O ใน CX-Programmer

```
CX-Programmer → PLC → I/O Table

I/O Table Window:
┌──────────────────────────────────────────────────────────┐
│ I/O Table - CP1H-X40DT-D                                │
├────────────┬─────────────┬──────────────────────────────┤
│ Unit No.   │ Slot        │ Unit Type                    │
├────────────┼─────────────┼──────────────────────────────┤
│ CPU        │ Built-in    │ CP1H-X40DT-D (CPU)           │
│            │             │   Input: CH0, CH1             │
│            │             │   Output: CH100, CH101       │
├────────────┼─────────────┼──────────────────────────────┤
│ EXP1       │ Expansion 1 │ CPM1A-20EDT                  │
│            │             │   Input: CH2                  │
│            │             │   Output: CH102              │
└────────────┴─────────────┴──────────────────────────────┘

Symbol Table:
┌──────────────────────────────────────────────────────────┐
│ Symbol        │ Address  │ Comment                      │
├──────────────────────────────────────────────────────────┤
│ Start_PB      │ 0.00     │ ปุ่ม Start                   │
│ Stop_PB       │ 0.01     │ ปุ่ม Stop                    │
│ E_Stop        │ 0.02     │ Emergency Stop (NC)          │
│ Motor_Coil    │ 100.00   │ Contactor มอเตอร์             │
│ Run_Timer     │ TIM 000  │ Timer วิ่งมอเตอร์              │
│ Part_Counter  │ CNT 000  │ Counter ชิ้นงาน               │
│ Speed_SP      │ D00100   │ ค่าตั้งความเร็ว               │
└──────────────────────────────────────────────────────────┘
```

---

## Step 356: Timer Instructions ใน OMRON

### ประเภท Timer

```
TIM: On-Delay Timer (0.1 วินาที/Count)
TIMH: High-speed Timer (0.01 วินาที/Count)
TIML: Long Timer (0.1 วินาที/Count, 99999.9 วินาที)
TMHH: Very High-speed Timer (0.001 วินาที/Count)
TTIM: Accumulating Timer

ค่าสูงสุด:
TIM:  9999 counts × 0.1s = 999.9 วินาที (16.66 นาที)
TIMH: 9999 counts × 0.01s = 99.99 วินาที
TIML: 99999 counts × 0.1s = 9999.9 วินาที (2.78 ชั่วโมง)

การใช้ TIM:
Rung 1: Start Timer
   100.00(Motor Running)
──┤├────────────────────[TIM 000 #0050]──  // 5.0 วินาที

// TIM 000 = Timer Number
// #0050 = 50 counts × 0.1s = 5.0 วินาที
// # = Decimal Constant

Rung 2: Output หลัง Timer ครบ
   TIM 000              100.05
──┤├─────────────────────( )──             // ไฟแสดง 5 วินาทีผ่าน

หมายเหตุ:
- TIM ใช้ Rising Edge เพื่อเริ่มนับ
- หาก Contact หน้า TIM กลับ FALSE → Timer Reset
- ต่างจาก Siemens: TON Reset เมื่อ IN=FALSE เหมือนกัน
```

### OMRON Timer ตัวอย่างจริง

```
// โปรแกรม: แจ้งเตือนซ่อมบำรุงทุก 8 ชั่วโมง
// ใช้ TIML (Long Timer)
// 8 ชั่วโมง = 28800 วินาที = 288000 counts ×0.1s

Rung 1: Accumulate Running Time
   100.00(Motor_Running)
──┤├────────────────────[TIML 000 #288000]──  // 8 ชั่วโมง

Rung 2: Maintenance Alarm
   TIML 000              H0.00
──┤├─────────────────────(SET)──   // Latch Maintenance Request

Rung 3: Reset หลัง Maintenance
   0.15(Maint_Done)       H0.00
──┤├────────────────────────(RSET)──
   0.15                   TIML 000
──┤├─────────────────────(CNR 000 000)──  // Reset Timer

// TTIM (Accumulating Timer) - ดีกว่าสำหรับกรณีนี้:
Rung 1: Accumulate Time (ไม่ Reset เมื่อ Motor หยุด)
   100.00
──┤├───────────────[TTIM 001 #288000]──  // สะสมเวลา

Rung 2: Reset TTIM
   0.15(Maint_Done)
──┤├────────────────[CNR 001 001]──  // Reset Timer + Counter
```

---

## Step 357: Counter Instructions ใน OMRON

### ประเภท Counter

```
CNT: Down Counter (นับถอยหลัง)
CNTR: Reversible Counter (นับขึ้น/ลง)

ความแตกต่างจาก Siemens/Mitsubishi:
OMRON CNT นับ DOWN จาก Preset Value ไปหา 0!
(ต่างจาก Siemens/AB/Mitsubishi ที่นับ UP)

CNT: Down Counter
SV (Set Value) = Preset
PV (Present Value) = Count ปัจจุบัน (เริ่มจาก SV ลดลง)
Output = TRUE เมื่อ PV = 0

การใช้ CNT:
   0.10(Count_Sensor)      // Increment Input
─────────────────────┐
   0.11(Reset)        │    [CNT 000]
──┤├─────────────────┤R  SV: #0100  // นับ 100 ครั้ง
                         TC Bit: CNT 000

Rung 2: Output เมื่อนับครบ
   CNT 000              100.10
──┤├─────────────────────( )──

// PV เริ่มจาก 100 ลดไปเรื่อยๆ จนถึง 0 → CNT 000 = TRUE
```

### CNTR - Reversible Counter

```
CNTR (Reversible Counter):
- Increment Input: นับขึ้น
- Decrement Input: นับลง
- Reset Input: Reset เป็น 0

ตัวอย่าง: ตรวจสอบ Parts ใน Buffer Zone
// ชิ้นงานเข้า Buffer → นับขึ้น
// ชิ้นงานออก Buffer → นับลง
// Full เมื่อ Count = 50
// Empty เมื่อ Count = 0

   0.20(Part_In)       // Increment Input
──┤├──────────────────┐
   0.21(Part_Out)      │
──┤├─────────────────┐R│[CNTR 001]
   0.22(Reset)        ├R─  SV: #0050
──┤├─────────────────┘

// Full Alarm:
   CNT 001             // CNTR 001 TC Bit = TRUE เมื่อ CNT >= SV (50)
──┤├──────────────── [CMP D00000 #0050]  // หรือใช้ Compare
                      [GRT D00000 #0045]
──────────────────── ( Full_Alarm )
```

---

## Step 358: Special Flags ใน OMRON

### Auxiliary Area (A) Flags

```
A Area มี Special Flags คล้าย M8000-M8511 ของ Mitsubishi:

P_First_Cycle (A200.11): TRUE เฉพาะ Scan แรก (เหมือน M8002)
P_First_Cycle_Task (A200.12): First Scan ของ Task นั้น
P_Step (A200.10): Scan ปกติ

Clock Bits:
P_0_02s (A200.02): 0.02s Clock (10ms ON, 10ms OFF)
P_0_1s  (A200.00): 0.1s Clock
P_0_2s  (A200.01): 0.2s Clock
P_1s    (A200.02): 1 Second Clock (ต้องตั้งใน PLC Setup)
P_1min  (A200.03): 1 Minute Clock

Error Flags:
P_CY (A200.12): Carry Flag
P_GR (A200.13): Greater Than Flag (บางรุ่น)
P_EQ (A200.14): Equal Flag
P_N  (A200.15): Negative Flag
P_OF (A200.11): Overflow Flag
P_UF (A200.10): Underflow Flag

Battery:
A402.04: Battery Voltage Low Warning
A402.05: Battery Very Low Error

Program Error:
A295: Error Code (เมื่อมี Error)
A100-A199: I/O Error Area
```

### การใช้ Special Flags ตัวอย่าง

```
// การตั้งค่าเริ่มต้นด้วย P_First_Cycle:
   P_First_Cycle          D00100
──┤├──────────────[MOV #1500]──   // ตั้ง Speed Default = 1500

   P_First_Cycle          D00200
──┤├──────────────[MOV #0080]──   // ตั้ง Temp SP Default = 80°C

// ไฟ Blink ด้วย Clock:
   P_0_1s    Alarm_Active    Blink_Light
──┤├──────────────┤├────────────( )──   // กะพริบ 5Hz เมื่อ Alarm

// ไฟวิ่ง (Running Light) ด้วย 1-second Timer:
   P_1s              Running_Index
──┤├───────────[INC D00300]──   // เพิ่มค่า D300 ทุกวินาที
   // ใช้ D300 MOD 8 เพื่อควบคุม 8 LED วิ่ง

// ตรวจสอบ Battery Low:
   A402.04               Battery_Warning_Light
──┤├─────────────────────────( )──   // แจ้งเตือนก่อน Battery หมด

   A402.05               Battery_Error_Alarm
──┤├─────────────────────────( )──   // Error: ต้องเปลี่ยน Battery
```

---

## Step 359: Pulse Output ใน CP1H

### Pulse Output สำหรับ Stepper/Servo Motor

```
CP1H-X40DT-D มี Built-in Pulse Output:
- 2 Axes (Y0.00/Y0.02 = Pulse, Y0.01/Y0.03 = Direction)
- ความเร็วสูงสุด: 100kHz
- Modes: PLS2 (Position Control), SPED (Speed Control)

การตั้งค่า Pulse Output:
CX-Programmer → PLC → PLC Setup → Pulse Output

PLS2: Positioned Pulse Output
   Start_Move           [PLS2(887)]
──┤├──────────────────[Port No: #0000]  // Axis 0
                      [Pulse Freq: D00100]  // ความเร็ว Hz
                      [Acceleration: #1000] // Accel Hz/4ms
                      [Target Pulses: D00200] // จำนวน Pulse
                      [Result: W0.00]        // Busy Flag

// ตัวอย่าง: เคลื่อนที่ไป 5000 Step ที่ 10kHz
Rung 0: ตั้งค่าก่อน Move
   P_First_Cycle
──┤├──────────[MOV #10000  D00100]   // 10kHz
   P_First_Cycle
──┤├──────────[MOV #5000   D00200]   // 5000 steps

Rung 1: สั่ง Move
   Move_Cmd    W0.00(Busy)
──┤├───────────┤/├──────────[PLS2(887) ...]   // Move ถ้าไม่ Busy

Rung 2: Home Position
   Home_Cmd
──┤├──────────[SPED(885) #0 #1 #500]  // Speed 500Hz, CW
   Home_LS
──┤├──────────[INI(880) #0 #0 #0]     // Stop Pulse Output
              [PULS(886) #0 #0 D0]    // Reset Position Counter
```

---

## Step 360: High-Speed Counter ใน CP1H

### Built-in High-speed Counter

```
CP1H-X40DT-D High-speed Counter Specs:
- 4 Channels: CH0, CH1, CH2, CH3
- Input: 0.00-0.03 (Z Phase), ฯลฯ
- Maximum Frequency: 100kHz (Linear Mode)
- 32-bit Counter: ±2,147,483,648

Mode สำหรับ HSC:
1. Differential Phase (ไม่แยกทิศ): Mode 0
2. Pulse + Direction: Mode 1
3. Up/Down Pulse: Mode 2
4. Linear Counter: Mode 3

การตั้งค่า:
CX-Programmer → PLC → PLC Setup → 
Built-in Input Setting → High-speed Counter

การอ่านค่า HSC:
// HSC Channel 0 PV อยู่ที่ A270/A271 (32-bit)
   P_Always_ON
──┤├──────────[MOVL A27000  D00100]  // อ่าน 32-bit เป็น D100,D101

// หรือใช้ PRV Instruction:
   M_Read
──┤├──────────[PRV(881) #0000 #0000 D00200]
// #0000 = HSC Channel 0
// #0000 = Read Present Value
// D00200 = Destination (32-bit = D200,D201)
```

---

## Step 361: Math และ Conversion Instructions

### Arithmetic ใน OMRON

```
ADD(400) : บวก 2 Word → Result
ADDL(401): บวก 2 DWord → Result  
SUB(410) : ลบ
SUBL(411): ลบ DWord
MUL(420) : คูณ (ผลลัพธ์ 2 Word)
MULL(421): คูณ DWord (ผลลัพธ์ 2 DWord)
DIV(430) : หาร (Quotient + Remainder)
DIVL(431): หาร DWord

ตัวอย่าง:
// D00100 + D00200 → D00300
   Calc_Trigger
──┤├──────────[ADD(400) D00100  D00200  D00300]

// D00100 × 1000 → D00300 (32-bit result in D300:D301)
   Calc_Trigger
──┤├──────────[MUL(420) D00100  #1000  D00300]

// แปลง AI (0-4000) เป็น Temperature (0-400.0°C ×10)
// Temp = AI × 4000 / 4000 = AI (หน่วย 0.1°C)
// หรือ Physical Range 4-20mA (Offset 4mA = 0°C):
// Raw = 0-32000 (FX5U ใช้ 0-27648 แต่ CP1H ใช้ 0-4000)
// Temp = (Raw × 400) / 4000 × 10 = Raw (°C ×10)
   M_Always_ON
──┤├──────────[MUL D00050  #0400  D00300]  // Raw×400
   M_Always_ON
──┤├──────────[DIV(430) D00300  #4000  D00400]  // ÷4000 → D400=Temp/10
              // D401 = Remainder (ทิ้งได้)
```

### BIN/BCD Conversion

```
OMRON รองรับทั้ง BCD และ Binary Math:

BIN(023): BCD → Binary
BCD(024): Binary → BCD

ทำไมต้องใช้ BCD?
- LED Display 7-Segment ใช้ BCD
- เครื่องมือวัดรุ่นเก่าส่ง BCD
- Thumbwheel Switch ส่ง BCD

ตัวอย่าง:
// อ่าน Thumbwheel (BCD) → แปลงเป็น Binary
// Thumbwheel CH2 = BCD "0250"
   M_Always_ON
──┤├──────────[BIN(023) CH2  D00050]   // D50 = 250 (Binary)

// ส่ง Value ไป LED Display (ต้องการ BCD)
   M_Always_ON
──┤├──────────[BCD(024) D00100  CH104]  // CH104 = BCD Output
```

---

## Step 362: Subroutine และ Task

### Subroutines ใน OMRON

```
OMRON ใช้ Jump/Subroutine:
JMP: Unconditional Jump ไป Label
RJMP: Return Jump (กลับจาก JMP)
SBN: Subroutine Begin (กำหนด Label)
RET: Return จาก Subroutine
MCRO: Macro Call

ตัวอย่าง Subroutine:
// Main Program:
Rung 1: เรียก Subroutine เมื่อ Calc Trigger
   Calc_Trigger
──┤├──────────[SBS(091) #100]   // Call Subroutine 100

// ด้านล่าง Main Program (หลัง END):
// Subroutine 100:
SBN(092) #100
Rung: คำนวณ OEE
   M_Always_ON
──┤├──[MUL D00100  D00200  D00300]   // Avail × Perf
   M_Always_ON
──┤├──[DIV D00300  #100    D00400]   // / 100
   M_Always_ON
──┤├──[MUL D00400  D00500  D00600]   // × Quality
   M_Always_ON
──┤├──[DIV D00600  #100    D00700]   // OEE = D00700
RET(093)
```

### Task ใน CP1H

```
CP1H รองรับ Cyclic Tasks และ Interrupt Tasks:

Cyclic Tasks (0-31): ทำงานตาม Scan Cycle
Interrupt Tasks:
- Scheduled Interrupt: เรียกตาม Timer (0.1ms - 100ms)
- I/O Interrupt: เรียกเมื่อ Input เปลี่ยน
- External Interrupt: Power Off interrupt

การสร้าง Task:
CX-Programmer → Programs → Right-click → New Task
- Task Type: Cyclic หรือ Interrupt
- Task Number: 0-31

ตัวอย่าง Scheduled Interrupt Task (100ms):
Task Settings:
- Type: Scheduled
- Period: 100ms
- Task No: 2

Program ใน Task 2 (100ms):
// PID Control ต้องการ Sampling time คงที่
[PID Control Calculation]
```

---

## Step 363: CX-Designer HMI Basics

### ความสัมพันธ์ระหว่าง CX-Programmer และ CX-Designer

```
CX-Designer ออกแบบ HMI สำหรับ NS/NT Series HMI:

การทำงานร่วมกัน:
1. สร้าง Symbol Table ใน CX-Programmer
2. Export Symbol Table (.SYM)
3. Import ใน CX-Designer → Tag Linking อัตโนมัติ

หรือ:
1. ลิงก์ Project โดยตรง (CX-One)
2. Symbol ใน PLC Program ↔ Tag ใน HMI

HMI Objects:
- Momentary Button: Toggle Output ชั่วคราว
- Maintained Button: Set/Reset Output
- Indicator Lamp: แสดงสถานะ Input/Output
- Numeric Display: แสดงค่า DM Register
- Numeric Entry: ป้อนค่าลง DM Register
- Trend: แสดง Trend ของ Data
```

### การตั้งค่า Address สำหรับ HMI

```
NS HMI Connection to CP1H:
- Serial (RS-232C): Direct Connect
- Ethernet: ผ่าน EtherNet/IP

HMI Object → Address:
Button ON/OFF → 100.00
Numeric Display → D00100
Alarm → W10.00

CX-Designer Address Entry:
[W: Word Area] [D: DM Area] [H: Holding]
Example: D00100 = DM Word 100
         100.00 = IR Word 100, Bit 00
```

---

## Step 364: CJ Series Overview

### CJ2M/CJ2H Hardware

```
CJ Series = Compact Modular PLC

Chassis-less Design:
ไม่มี Chassis, Module ต่อกันแบบ Rail

CJ2M-CPU11 → CJ1W-PA205R → CJ1W-ID211 → CJ1W-OD211 → End Plate

CJ2M-CPU31 Spec:
- Program Memory: 60K Steps
- Data Memory: 128K Words
- 32-bit Floating Point: ใช่
- Cycle Time: 40ns/Basic Instruction
- Built-in EtherNet/IP (CJ2H-CPU6□-EIP)

CJ1W Expansion Modules:
CJ1W-ID211: 16 DI (24VDC Sink/Source)
CJ1W-OD211: 16 DO (24VDC Transistor Source)
CJ1W-OD231: 16 DO (Relay)
CJ1W-AD041: 4 AI (±10V / 4-20mA)
CJ1W-DA041: 4 AO (0-10V / 4-20mA)
CJ1W-SCU41-V1: RS-232C/422/485 Communication
CJ1W-EIP21: EtherNet/IP Unit
CJ1W-CLK23: Controller Link
```

### CJ Series Address System

```
CJ I/O Address (แตกต่างจาก CP1H):

สำหรับ CJ, Channel ขึ้นกับ Unit No.:
Unit 0 (CPU Built-in): CH0-CH1 Input, CH100-CH101 Output
Unit 1 (แรก): CH2-CH3 Input, CH102-CH103 Output
Unit 2: CH4-CH5 Input, CH104-CH105 Output

DM: D00000-D32767 (ปกติ)
    D32768+ (Extended DM บางรุ่น)

Work: W000-W511
Holding: H000-H511

Task ใน CJ:
- Cyclic Task 0-31
- Interrupt Task (I/O Interrupt, Scheduled)
- Event Task 0-255
```

---

## Step 365: EtherNet/IP ใน OMRON NX/CJ

### CJ2H-CPU6□-EIP (Built-in EtherNet/IP)

```
CJ2H-CPU6□-EIP มี EtherNet/IP Built-in:
- IO Messaging (Implicit): Real-time I/O
- Explicit Messaging: Configuration/Data

CX-Programmer → Network Configuration:
1. Add EtherNet/IP Unit
2. กำหนด IP Address: 192.168.1.10
3. กำหนด I/O Connection ไปยัง Remote I/O

Sysmac Gateway (สำหรับ OPC UA/EtherNet/IP):
- PC Software สำหรับ Monitor CJ/CS PLC ผ่าน Network
- OPC DA/UA Server: SCADA เชื่อมต่อได้

TAG Data Link (ใน NJ/NX):
- Share ข้อมูลระหว่าง NJ Controllers
- ผ่าน EtherNet/IP Network Variables
- ประสิทธิภาพสูงและ Deterministic
```

---

## Step 366: Sysmac Studio สำหรับ NX Series

### NX/NJ Sysmac Platform

```
OMRON NX/NJ = ระบบใหม่ "Machine Automation Controller"
ใช้ Sysmac Studio แทน CX-Programmer

ความแตกต่าง NX/NJ vs CP1H/CJ:
┌─────────────────┬───────────────────────────────────────┐
│ Feature         │ NX/NJ                                 │
├─────────────────┼───────────────────────────────────────┤
│ Programming     │ IEC 61131-3 (LD, FBD, ST, SFC, IL)   │
│ Motion          │ Integrated Motion (MECHATROLINK-III)  │
│ Safety          │ Built-in Safety (NX-SL Safety)        │
│ Vision          │ FH Vision System Integration          │
│ Software        │ Sysmac Studio (แทน CX-Programmer)     │
│ OPC UA          │ Built-in OPC UA Server                │
│ Database        │ SQL Server Direct Connection          │
└─────────────────┴───────────────────────────────────────┘
```

### Sysmac Studio Interface

```
╔══════════════════════════════════════════════════════════════╗
║  Sysmac Studio - [Project_Name.smc2]                        ║
╠══════════════╦═══════════════════════════╦═════════════════╣
║ Multiview    ║ Workspace                 ║ Toolbox         ║
║ Explorer     ║                           ║                 ║
║              ║ [Ladder Diagram]          ║ Contacts        ║
║ ▼ Controller ║                           ║ Coils           ║
║   ▼ Motion   ║ Network 1:                ║ Math            ║
║   ▼ Safety   ║ ──┤Start_PB├──( Motor )── ║ Timer/Counter   ║
║   ▼ Program  ║                           ║ Motion          ║
║     POUs     ║ Network 2:                ║ Safety          ║
║     Programs ║ ──┤Motor├──[TON]──────    ║ FB Library      ║
║   ▼ Device   ║                           ║                 ║
║     I/O Map  ║                           ║                 ║
╚══════════════╩═══════════════════════════╩═════════════════╝
```

### Variable Declaration ใน Sysmac Studio

```
Sysmac Studio ใช้ Variable แทน Address:
(เหมือน Siemens TIA Portal + Allen-Bradley ผสมกัน)

Variable Declaration (Global Variables):
┌──────────────────────────────────────────────────────────┐
│ Name          │ Data Type │ Initial │ AT       │ Comment │
├──────────────┼───────────┼─────────┼──────────┼─────────┤
│ Start_PB      │ BOOL      │ FALSE   │ %IX0.0   │ Start   │
│ Stop_PB       │ BOOL      │ FALSE   │ %IX0.1   │ Stop    │
│ Motor_Out     │ BOOL      │ FALSE   │ %QX0.0   │ Motor   │
│ Speed_Cmd     │ INT       │ 0       │          │ Speed   │
│ Temp_PV       │ REAL      │ 0.0     │ %IW2     │ Temp AI │
└──────────────┴───────────┴─────────┴──────────┴─────────┘

AT Declaration ผูก Variable กับ I/O:
%IX0.0 = Input Bit, Word 0, Bit 0
%QX0.0 = Output Bit, Word 0, Bit 0
%IW2 = Input Word 2 (Analog)
%QW100 = Output Word 100
```

---

## Step 367: โปรแกรมตัวอย่าง - Packaging Machine

### Project Overview

```
เครื่องบรรจุภัณฑ์ (Packaging Machine):
- Product Infeed Conveyor
- Weighing Station
- Rejection System
- Packing Conveyor
- Box Fill Counter

Hardware (CP1H-XA40DT-D):
Built-in Analog:
  AI1: Weight Scale (4-20mA = 0-2000g)
  AI2: Product Height (0-10V = 0-200mm)
  AO1: Conveyor Speed (0-10V = 0-100%)

Digital I/O:
  0.00: Start_PB           100.00: Infeed_Conv_Run
  0.01: Stop_PB            100.01: Pack_Conv_Run
  0.02: E_Stop             100.02: Reject_Gate
  0.03: Product_Sensor     100.03: Reject_Air_Blast
  0.04: Scale_Ready        100.04: Pack_Complete_Light
  0.05: Height_OK          100.05: Alarm_Horn
  0.06: Reject_Sensor
  0.07: Pack_Full_Sensor
```

### โปรแกรม Logic

```
Rung 1: System Start/Stop
   0.00(Start) 0.02(E_Stop)  0.01(Stop)    W0.00(System_On)
──┤├──────┬────┤/├───────────┤/├────────────( )──
          │
   W0.00
──┤├──────┘

Rung 2: Infeed Conveyor
   W0.00   0.07(Pack_Full)   100.00(Infeed_CV)
──┤├─────────┤/├─────────────( )──

Rung 3: Read Weight Scale (ทุก Scan)
   P_Always_ON
──┤├──────[MOV OW0(AI1 Word)  D00100]  // D100 = Weight Raw (0-4000)
   P_Always_ON
──┤├──[MUL D00100  #2000  D00300]      // ×2000
   P_Always_ON
──┤├──[DIV D00300  #4000  D00200]      // ÷4000 → D200 = Weight (g)

Rung 4: Weight Check
   0.04(Scale_Ready)
──┤├──[LT(425) D00200  #0900]─── W1.00(Underweight)  // < 900g
   0.04
──┤├──[GT(425) D00200  #1100]─── W1.01(Overweight)   // > 1100g

Rung 5: Reject Control
   W1.00    0.03(Product_Sensor)     100.02(Reject_Gate)
──┤├──────────────┤├──────────────────( )──
   W1.01
──┤├──────────────┘

Rung 6: Air Blast (0.5 วินาที)
   100.02(Reject_Gate)          TIM 010
──┤├──────────────────────────(TIM 010 #0005)──  // 0.5s
   TIM 010                      100.03(Air_Blast)
──┤├───────────────────────────[NOT]──( )──       // OFF หลัง 0.5s

Rung 7: Pack Counter
   0.07(Pack_Full_Sensor)       CNT 000 #0010
──┤├──────────────────────────────────────────
   0.06(Reject_Sensor)
──┤├──────────────────────────[CNR 000 000]──  // ไม่นับถ้า Reject

Rung 8: Cycle Counter (นับครบ 1000 แล้วแจ้ง PM)
   0.03(Product_Sensor)         CNT 001 #1000
──┤├──────────────────────────────────────────
   CNT 001                      W5.00(PM_Required)
──┤├──────────────────────────────(SET)──

Rung 9: END
END(001)
```

---

## Step 368: Serial Communication ใน CP1H

### RS-232C Communication

```
CP1H มี Built-in RS-232C Port:
- Speed: 1200-115200 bps
- Modes: Host Link (FINS), No-protocol, NT Link, Modbus RTU

Host Link (FINS):
PC อ่าน/เขียน PLC ด้วย FINS Protocol
ใช้สำหรับ:
- CX-Programmer Connect
- Custom SCADA Program
- MES Integration

Modbus RTU Slave Mode:
CP1H ทำหน้าที่เป็น Modbus Slave:
- Register Mapping: D Area → Holding Register
- Read Coil: Channel Bit
- Write Coil: Channel Bit

การตั้งค่า:
CX-Programmer → PLC → PLC Setup → Port Settings:
  Port 1 (RS-232C):
  Mode: Modbus RTU Slave
  Address: 1
  Baud: 9600
  Data Bits: 8, Parity: None, Stop: 1
```

### RS-485 Communication

```
CP1H RS-485 (Option Board):
ต่อ Serial Communication Board CP1W-CIF11 (RS-485)

ใช้สำหรับ:
- Modbus RTU Master (อ่าน Sensor, Drive)
- Multi-drop Network (สูงสุด 32 Nodes)

PMCR Instruction (Protocol Macro Command):
ใช้ส่ง Custom Frame ผ่าน Serial Port

ตัวอย่าง Modbus RTU Master อ่าน Holding Register:
// สร้าง Modbus Message ใน D Area
// Function Code 03: Read Holding Registers
// Slave Address: 1
// Start Register: 40001 (0000H)
// Quantity: 5

D00400 = 0x0103  // Slave=01, FC=03
D00401 = 0x0000  // Register Start High
D00402 = 0x0005  // Quantity

Rung: ส่ง Modbus Request
   Read_Trigger    
──┤├──[PMCR D00400  D00500  K6  K15]  // ส่งข้อมูล

// Response เก็บใน D00500 เป็นต้นไป
```

---

## Step 369: การ Download และ Online Monitoring

### การ Connect และ Download

```
CX-Programmer Connection Methods:
1. USB Direct (CP1H): USB-B → PC USB-A
2. RS-232C: Serial Port ผ่าน Programmer Cable
3. Ethernet (CJ Series): ผ่าน EtherNet/IP

การ Download:
1. PLC → Work Online → Yes (Connect)
2. PLC → Transfer → To PLC

Transfer Dialog:
┌──────────────────────────────────────────────────────┐
│ Transfer                                             │
├──────────────────────────────────────────────────────┤
│ ✓ Programs                                          │
│ ✓ I/O Table                                        │
│ ✓ PLC Setup                                        │
│ ✓ Symbol Table                                     │
│ ✓ Device Comments                                  │
│ ✓ Data Memory (DM)                                 │
├──────────────────────────────────────────────────────┤
│ [Transfer]  [Cancel]                               │
└──────────────────────────────────────────────────────┘
3. ยืนยัน → PLC Switches to PROGRAM Mode
4. Download เสร็จ → Switch PLC to RUN

Run/Stop ผ่าน Software:
PLC → Operate → Run
PLC → Operate → Stop
```

### Online Monitoring และ Force

```
Online Monitoring:
- Double-click บน Rung เพื่อเปิด Monitor
- Contacts ที่ ON = สีฟ้า/เขียว
- Coil ที่ ON = สีฟ้า
- ค่า Data แสดงใน []

Watch Window:
1. View → Watch (Ctrl+W)
2. เพิ่ม Address: 0.00, 100.00, D00100 ฯลฯ

Monitor Table:
┌───────────────────────────────────────────────┐
│ Address   │ Value │ Description                │
├───────────┼───────┼───────────────────────────┤
│ 0.00      │ OFF   │ Start_PB                  │
│ 100.00    │ ON    │ Motor_Coil                │
│ TIM000    │ 0025  │ Timer 000 (2.5s)          │
│ D00100    │ 1500  │ Speed_SP                  │
└───────────┴───────┴───────────────────────────┘

Force Bit:
Right-click Address → Force Bit → ON หรือ OFF
⚠️ ระวัง: Force ระหว่าง Machine Run อาจเป็นอันตราย

Release Force:
Right-click → Release Force
หรือ PLC → Force Bit → Release All

Force DM:
Right-click D Address → Set Present Value → ป้อนค่า
```

---

## Step 370: Lab Projects

### Lab 19-1: Temperature Controller ด้วย CP1H-XA

```
โจทย์: ควบคุมอุณหภูมิ Water Bath

Hardware:
- AI1 (Built-in): PT100 Temperature Sensor (4-20mA = 0-200°C)
- AO1 (Built-in): Heater SSR Control (0-10V = 0-100% Duty)
- 0.00: Start/Stop Toggle
- 0.01: Alarm Reset
- 0.02: Auto/Manual Mode
- 0.03: SP Up (เพิ่ม SP 1°C)
- 0.04: SP Down (ลด SP 1°C)
- 100.00: Heater ON Indicator
- 100.01: Alarm Horn
- 100.02: Ready (Temperature ใกล้ SP ±5°C)

Requirements:
1. อ่าน Temperature ทุก Scan Cycle
   - Raw = 0-4000 → Temperature = 0-200°C
   - แปลงและเก็บใน D00200 (หน่วย 0.1°C, เช่น 1234 = 123.4°C)

2. Setpoint Control:
   - ค่า Default SP = 60.0°C (D00100 = 600)
   - 0.03: D00100 + 10 (เพิ่ม 1.0°C ต่อครั้ง)
   - 0.04: D00100 - 10 (ลด 1.0°C ต่อครั้ง)
   - SP Range: 20.0 ถึง 95.0°C

3. Proportional Control (ง่าย):
   Error = SP - PV
   ถ้า Error > 50 (5°C) → Output = 100%
   ถ้า Error < -10 (-1°C) → Output = 0%
   ถ้า -10 <= Error <= 50 → Output = Error × 2 (%)

4. Alarms:
   - High Alarm: PV > SP + 10°C → SET Alarm
   - Over-temp: PV > 98°C → Heater OFF + Alarm Latch

5. Display (Numeric Entry/Display บน HMI):
   D00100 = SP (×0.1°C)
   D00200 = PV (×0.1°C)
   D00300 = Output (%)
```

### Lab 19-2: Modbus RTU Master Communication

```
โจทย์: อ่านข้อมูลจาก Variable Speed Drive (VSD)

Hardware:
- CP1H + RS-485 Option Board (CP1W-CIF11)
- Modbus RTU Slave: ABB ACS355 Drive (Address 1)
- Baud: 9600, 8N1

ACS355 Modbus Register Map:
40001: Speed Reference (0-10000 = 0-100%)
40002: Control Word (Run/Stop/Fault Reset)
40003: Status Word
40004: Actual Speed (0-10000 = 0-100%)
40005: Output Current (0-1000 = 0-100A ×0.1)
40006: Output Voltage (0-5000 = 0-500V ×0.1)

Requirements:
1. อ่าน Register 40001-40006 ทุก 500ms
   - เก็บใน D00100-D00105

2. เขียน Speed Reference (D00200 → Register 40001)
   - เมื่อมีการ Modify D00200

3. ควบคุม Run/Stop ผ่าน Control Word:
   - 0.10: Run Command → Register 40002 = 0x0006
   - 0.11: Stop Command → Register 40002 = 0x0002

4. แสดงค่าบน Watch Window:
   - Actual Speed (%)
   - Output Current (A ×0.1)
   - Output Voltage (V ×0.1)
   - Drive Status (Running/Stopped/Fault)
```

---

## ข้อผิดพลาดที่พบบ่อยใน OMRON

```
1. Timer ไม่นับ
   สาเหตุ: Contact นำ TIM ต้องเป็น Rising Edge
   แก้ไข: ใช้ DIFU สร้าง Rising Edge ก่อน TIM
   หรือ ลบ TIM แล้วใส่ใหม่ (Check Logic ด้วย)

2. CNT นับผิดทิศ (ลดลงแทนเพิ่ม)
   สาเหตุ: OMRON CNT นับ DOWN เป็น Default
   แก้ไข: ใช้ CNTR หรือ ปรับ Logic

3. Output ไม่ทำงาน
   สาเหตุ: PLC Mode = PROGRAM หรือ MONITOR
   แก้ไข: Switch PLC to RUN Mode

4. DM ค่าหายหลัง Power Off
   สาเหตุ: DM บางส่วนไม่ได้ตั้งเป็น Held
   แก้ไข: CX-Programmer → PLC Setup → Memory Hold

5. CX-Programmer ต่อ PLC ไม่ได้
   สาเหตุ: Driver ผิด หรือ IP ไม่ตรง
   แก้ไข:
   a. USB: Driver อาจต้องติดตั้งแยก (CP1H USB Driver)
   b. Serial: ตรวจ COM Port, Speed, Parity
   c. Network: Ping PLC ก่อน

6. Special Flag (A Area) ถูกเขียนทับ
   สาเหตุ: ใช้ A Area สำหรับ User Data
   แก้ไข: A Area Reserve ไว้สำหรับ System เท่านั้น
   ใช้ W Area หรือ D Area สำหรับ User Data
```

---

## สรุปและทบทวน

**ความรู้ที่ได้รับจาก Part 19:**

1. **OMRON Address** ใช้รูปแบบ Word.Bit (เช่น 0.00, 100.00)
2. **CNT ของ OMRON นับลง** (ต่างจาก Siemens/Mitsubishi ที่นับขึ้น)
3. **Special A Area** มี Flags คล้าย M8000 ของ Mitsubishi
4. **CP1H-XA** มี Built-in Analog เหมาะสำหรับงานที่ต้องการ AI/AO น้อย
5. **Pulse Output** สำหรับ Stepper/Servo Control Built-in ใน CP1H
6. **High-speed Counter** รองรับ Encoder ด้วย Hardware Input
7. **Sysmac Studio/NX** คือทิศทางใหม่ของ OMRON ใช้ IEC Standard
8. **Modbus RTU** Support ทั้ง Master และ Slave ใน CP1H

---

**[← Part 18: Mitsubishi](part18_mitsubishi.md) | [Part 20: Debugging →](part20_debugging.md)**

*เนื้อหา Part 19 ครอบคลุม Steps 351-370*
*ลิขสิทธิ์ © 2024 Ladder Diagram PLC Course*
