# Part 12: Safety พื้นฐาน (Steps 211-230)

## ภาพรวมของบทเรียน

Safety ในระบบอัตโนมัติอุตสาหกรรมเป็นเรื่องที่สำคัญที่สุด ในบทนี้เราจะเรียนรู้หลักการ Safety ตามมาตรฐานสากล ตั้งแต่ Safety Category, Safety Relay, Emergency Stop, จนถึง Risk Assessment และเครื่องมือที่ใช้ในงานจริง

> **คำเตือน:** Safety System ที่ออกแบบผิดพลาดอาจทำให้เกิดอุบัติเหตุถึงแก่ชีวิตได้ ควรปรึกษาผู้เชี่ยวชาญด้าน Functional Safety เสมอ

---

## Step 211: มาตรฐาน Safety ที่เกี่ยวข้อง

### มาตรฐานสากลด้าน Safety

```
Hierarchy ของมาตรฐาน Safety:

IEC 61508 (General Functional Safety)
    │
    ├── IEC 62061 (Machinery - SIL)
    │
    ├── ISO 13849 (Machinery - PL)
    │
    ├── IEC 61511 (Process Industry - SIL)
    │
    └── EN 60204-1 (Electrical Equipment of Machines)
```

### ความแตกต่าง Safety Category (EN 954-1) vs PL (ISO 13849)

```
EN 954-1 (เก่า, ยังใช้อยู่บางส่วน):
Category B, 1, 2, 3, 4

ISO 13849-1 (ใหม่, แทนที่ EN 954-1):
Performance Level (PL): a, b, c, d, e

IEC 62061 (สำหรับ Electrical/Electronic):
Safety Integrity Level (SIL): 1, 2, 3
```

---

## Step 212: Safety Categories (EN 954-1)

### Category B: พื้นฐาน

```
Category B - Basic Safety Principle

โครงสร้าง:
Input ──► Safety Function ──► Output
  │                              │
  └──── Single Channel ──────────┘

ลักษณะ:
- Single Channel เท่านั้น
- ไม่มี Redundancy
- ถ้าชิ้นส่วนเสีย = Safety Loss
- ใช้สำหรับความเสี่ยงต่ำมาก

ตัวอย่าง: ปุ่ม Start/Stop ธรรมดา
```

### Category 1: ใช้ชิ้นส่วนที่ผ่านการพิสูจน์แล้ว

```
Category 1 - Well-Tried Components

โครงสร้าง: เหมือน Category B แต่ใช้อุปกรณ์ที่
           ผ่านการใช้งานพิสูจน์แล้ว (Well-tried)

ลักษณะ:
- Single Channel
- ใช้ Safety-rated components
- Forced Opening Contacts
- ตัวอย่าง: Safety Limit Switch, Safety E-Stop

ตัวอย่าง Application:
E-Stop ──[Safety NC Contact]──► Stop Motor
         (Forced Opening = ถ้า Contact เสีย จะ Open ไม่ Close)
```

### Category 2: ตรวจสอบเป็นระยะ

```
Category 2 - Monitoring by Test

โครงสร้าง:
                    ┌──── Test Unit ────┐
                    │                  │
Input ──────────────┤ Safety Function  ├──► Output
                    │                  │
                    └──────────────────┘
                              │
                         Test Output ──► Indicator

ลักษณะ:
- Single Channel + Test Input
- ตรวจสอบการทำงานเป็นระยะ
- ถ้า Test Fail = Output ตัด
- Diagnostic Coverage ≥ 60%
```

### Category 3: Redundant Channels

```
Category 3 - Dual Channel

โครงสร้าง:
Input ──┬─── Channel 1 ───┬──► Output
        │                 │
        └─── Channel 2 ───┘
                 │
            Cross-monitoring

ลักษณะ:
- Dual Channel (Redundancy)
- ถ้า Channel 1 เสีย, Channel 2 ยังทำงาน
- การเสียชิ้นส่วน 1 ตัว ไม่ทำให้ Safety Loss
- Fault Detection = เมื่อมีการ Operate

ตัวอย่าง:
E-Stop 1 (NC) ──┬──► Safety Relay CH1 ──┬──► Output
E-Stop 2 (NC) ──┘    (Cross Monitoring)  └──► Output
```

### Category 4: Highest Level

```
Category 4 - High Diagnostic Coverage

โครงสร้าง: เหมือน Category 3 แต่มี Diagnostic Coverage สูงกว่า

              Diagnostic Coverage ≥ 99%

ลักษณะ:
- Dual Channel + High DC
- Fault Detection ทันที (ไม่รอ Demand)
- Multiple Fault Tolerance
- ใช้สำหรับความเสี่ยงสูงมาก (Amputation, Death Risk)

ตัวอย่าง: Light Curtain ระดับ Safety + Safety PLC
```

---

## Step 213: Performance Level (ISO 13849)

### PL a ถึง PL e

```
PL  │  MTTFd  │  DC   │ CCF │ Application
────┼─────────┼───────┼─────┼──────────────────────────
 a  │   Low   │  None │  -  │ ความเสี่ยงต่ำมาก
 b  │   Low   │  None │  -  │ ความเสี่ยงต่ำ
 c  │  Medium │  Low  │  -  │ ความเสี่ยงปานกลาง
 d  │   High  │  Low  │ 100pts│ ความเสี่ยงสูง
 e  │   High  │  High │ 100pts│ ความเสี่ยงสูงมาก

MTTFd = Mean Time To Failure (Dangerous)
DC = Diagnostic Coverage
CCF = Common Cause Failure (Score 100 = Category 3/4)
```

### Required PL Determination

```
Risk Graph ตาม ISO 13849-1 Figure 3:

            ┌─── C1 ─── F1 ─── P1 ──► PLr = a
            │              └── P2 ──► PLr = b
     S1 ────┤     └── F2 ─── P1 ──► PLr = b
            │              └── P2 ──► PLr = c
            │
            │         ─── F1 ─── P1 ──► PLr = c
     S2 ────┤    C2 ──    └── P2 ──► PLr = d
            │         ─── F2 ─── P1 ──► PLr = d
            └────               └── P2 ──► PLr = e

S = Severity (S1=Light, S2=Serious/Death)
F = Frequency (F1=Rarely, F2=Frequently)
P = Possibility to Avoid (P1=Possible, P2=Seldom possible)
C = (ไม่มีในมาตรฐานล่าสุด, แต่บางรุ่นอาจมี)
```

---

## Step 214: Safety Relay พื้นฐาน

### Safety Relay คืออะไร

Safety Relay เป็น Relay พิเศษที่:
1. มี Redundant Output Contacts
2. ตรวจสอบ Cross-Short ระหว่าง Channels
3. มี Manual Reset (Monitored)
4. ตรวจสอบการทำงานตัวเอง (Self-monitoring)

### วงจร Safety Relay ทั่วไป

```
PILZ PNOZ X2P (หรือเทียบเท่า)

         A1 (+24V) ──┐
                     │         ┌── 13 ──[NO]── 14 ──► Safe Output 1
         A2 (0V)  ──┤         │
                     │         ├── 23 ──[NO]── 24 ──► Safe Output 2
         S11 ──[E-STOP NC]──S12│
                               ├── 31 ──[NO]── 32 ──► Auxiliary
         S21 ──[E-STOP NC]──S22│
                               └── 41 ──[NC]── 42 ──► Monitoring Out
         Y11 ──[RESET PB]──Y12

วงจรต้องต่อครบทุก Terminal ตามที่กำหนด!
```

### Wiring Diagram Safety Relay

```
E-Stop Circuit with Safety Relay

+24V ──┬────────────────────────────────────────┐
       │                                        │
       │  Safety Relay (PILZ PNOZ X2)           │
       │  ┌─────────────────────────────────┐  │
  A1 ←─┤  │                                 │  │
       │  │                                 │  │
  S11 ──┤  │ E-Stop 1 (NC) ─────────────────┤ S12
       │  │                                 │  │
  S21 ──┤  │ E-Stop 2 (NC) ─────────────────┤ S22
       │  │                                 │  │
  Y11 ──┤  │ Reset PB (NO) ─────────────────┤ Y12
       │  │                                 │  │
  A2 ←─┤  │                                 │  │
       │  └─────────────────────────────────┘  │
       │                    │                   │
       │              Output Contacts           │
       │         13/14 ──────────────► KM1 Coil │
       │         23/24 ──────────────► KM2 Coil │
       │                                        │
      0V ───────────────────────────────────────┘
```

### Ladder Diagram สำหรับ Safety Relay Monitoring

```ladder
// Monitor Safety Relay Output

Network 1: Safety OK Signal
|  %I0.5   |
|SAFETY_OK |────────────────────────────────────( %M10.0 )
|(Safety   |                                    Safety_Status
| Relay    |
| Feedback)|

Network 2: Allow Run Only When Safe
|  %M10.0  |  %I0.0   |  %I0.1   |
|Safety_OK |──PB_START──PB_STOP_N──────────────( %Q0.0 )
|          |           |                       Motor_Run
|  %Q0.0   |
|Motor_Run |────────────────────────────────────┘

Network 3: Safety Fault Alarm
|  %M10.0  |
|Safety_OK |────────/────────────────────────────( %Q0.5 )
| (NC)     |                                     Safety_Alarm
```

---

## Step 215: Emergency Stop Circuit Design

### ประเภทของ Emergency Stop

```
E-Stop Mushroom Head Types:

1. Turn-to-Release (เป็นแบบ Twist)
   - หมุนปลด กดล็อค
   - ป้องกันการกดโดยไม่ตั้งใจ

2. Key Release (ใช้กุญแจปลด)
   - ต้องใช้กุญแจปลด
   - ความปลอดภัยสูงขึ้น

3. Pull-to-Release (ดึงปลด)
   - ดึงขึ้นเพื่อปลด

ทุกแบบต้อง:
- สีแดง ฝาปิดสีเหลือง
- Forced Guided Contacts
- Direct Opening Action
- มาตรฐาน IEC 60947-5-1
```

### E-Stop Categories (IEC 62061)

```
Stop Category 0: Power-off ทันที
- ตัดไฟทุก Actuator ทันที
- ไม่มีการ Controlled Deceleration
- ใช้เมื่อหยุดทันทีปลอดภัยกว่า

Stop Category 1: Controlled Stop → Power Off
- Decelerate → หยุด → ตัดไฟ
- ใช้กับ Servo, VFD
- ต้องมี Timer ติดตาม

Stop Category 2: Controlled Stop → Power On
- Decelerate → หยุด → ไฟยังอยู่
- ใช้กับงานที่ต้องการ Hold Position
```

### E-Stop Reset Requirement

```
Safety Standard กำหนดว่า E-Stop ต้อง:

1. Manual Reset เท่านั้น (ห้าม Auto Reset)
2. Reset Action ต้องไม่ทำให้ Machine Start
3. ต้องปลด E-Stop ก่อน Reset
4. ต้อง Reset ด้วยการกด Reset Button

Ladder Program:
|          |           |            |           |
|E_STOP_OK | RST_PULSE |            |           |
|  (NC)    |──────────────────────────────────( RESET_OK )
|          |
|RESET_OK  | PB_START  | PB_STOP_NC |           |
|──────────|──────────|─────────────────────────( MOTOR )
```

---

## Step 216: Two-Hand Control

### หลักการ Two-Hand Control

Two-Hand Control บังคับให้ Operator ใช้มือทั้งสองข้างกด Button พร้อมกัน เพื่อให้แน่ใจว่ามือทั้งสองอยู่ในที่ปลอดภัย

```
Two-Hand Control Principle:

BOTH hands on BOTH buttons → Machine RUNS

Release EITHER hand → Machine STOPS

Cannot reach danger zone while running!
```

### วงจร Two-Hand Control

```
Two-Hand Control Circuit (Type IIIC - Highest)

        Hand A (NO)      Hand B (NO)
           │                │
    +24V──[PB-A]────┬───[PB-B]────┬──► Safety Relay
                    │             │
                    │             │
           ← Sync Time ≤ 500ms →

ถ้า กด A แล้วกด B ภายใน 500ms = ALLOW
ถ้า กด A แล้วปล่อย A = STOP
ถ้า กด B ก่อน = STOP (ต้องกดพร้อมกัน)
```

### Ladder Program Two-Hand Control

```ladder
// Two-Hand Control (IEC 574 Type IIIC)

Network 1: Detect Both Pressed
|  %I0.0   |  %I0.1   |
| HAND_A   |  HAND_B  |────────────────────────( %M0.0 )
|  (PB-A)  |  (PB-B)  |                        Both_Pressed

Network 2: Synchronization Timer
|  %I0.0   |
| HAND_A   |────────────────────────────────────[TON T1]
|          |                                    PT=T#500MS

|  %I0.1   |
| HAND_B   |────────────────────────────────────[TON T2]
|          |                                    PT=T#500MS

Network 3: Sync OK = Both within 500ms
|  %T1.Q   |  %T2.Q   |
|Timer_A_ET|Timer_B_ET |────/──────────────────( %M0.1 )
|(A pressed|(B pressed  |  NOT             Sync_Fail
| within   | within     |  (หมายความว่า ถ้า
| 500ms)   | 500ms)     |   ทั้ง 2 กดใน 500ms
|          |            |   Sync_Fail = 0 = OK)

Network 4: Allow Run
|  %M0.0   |  %M0.1   |
| Both_    | Sync_Fail|
| Pressed  |   (NC)   |────────────────────────( %Q0.0 )
|          |           |                        MACHINE_RUN
```

---

## Step 217: Light Curtain (Safety Light Guard)

### Light Curtain คืออะไร

Light Curtain สร้าง "ม่านแสง" เพื่อตรวจจับเมื่อมีคนหรือวัตถุเข้าในพื้นที่อันตราย

```
Light Curtain Layout:

Emitter Side              Receiver Side
    │                           │
    │ ●──────────────────────── ●  ← Beam 1
    │ ●──────────────────────── ●  ← Beam 2
    │ ●──────────────────────── ●  ← Beam 3
    │ ●──────────────────────── ●  ← Beam 4
    │ ●──────────────────────── ●  ← Beam 5
    │                           │
    │  DANGER ZONE (Machine)    │
    │                           │
    └───────────────────────────┘

ถ้า Beam ใด Beam หนึ่งถูกขัด = STOP
```

### Safety Category ของ Light Curtain

```
ชนิด Light Curtain:
- Type 2 = Category 2, PLc (Basic safety)
- Type 4 = Category 4, PLe (Highest safety)

เลือกตาม Application:
- Finger Detection: Resolution ≤ 14mm
- Hand Detection: Resolution ≤ 30mm  
- Body Detection: Resolution ≤ 90mm
```

### การต่อ Light Curtain กับ Safety Relay

```
Light Curtain + Safety Relay Connection

+24V ────────────────────────────────────┐
  │                                     │
  │  Light Curtain                       │
  │  ┌──────────────────────────────┐   │
  ├─►│ +V                           │   │
  │  │                              │   │
  │  │  OSSD1 (Output Safety 1) ───────►│ S11 Safety
  │  │  OSSD2 (Output Safety 2) ───────►│ S21 Relay
  │  │                              │   │
  │  │ GND ────────────────────────────►│ A2
  │  └──────────────────────────────┘   │
 0V ─────────────────────────────────────
```

### Muting Function

```
Muting คือการ Bypass Light Curtain ชั่วคราว
เพื่อให้ Product ผ่านได้โดยที่ Machine ไม่หยุด

Muting Logic:
1. Muting Sensor 1 (ก่อน Light Curtain)
2. Muting Sensor 2 (ก่อน Light Curtain)
→ ทั้งสองต้องตรวจจับพร้อมกัน (T ≤ 2s)
→ Muting Active = Light Curtain Bypassed
→ Product ผ่านไปได้
→ หลัง Product ผ่าน = Muting End

Muting Sensor Layout:
   Mute1  Mute2
     │      │
     ▼      ▼
────[M1]──[M2]────[Light Curtain]────[Machine]
         Product →→→→→→→→→→→→→→→→
```

---

## Step 218: Safety Mat และ Area Scanner

### Safety Mat

```
Safety Mat คือแผ่นรับน้ำหนักที่ตรวจจับเมื่อมีคนเหยียบ

Safety Mat Circuit:

+24V ────────────────────────────────────────────┐
  │                                              │
  │  ┌─────────────────────────────────────┐    │
  │  │           Safety Mat                │    │
  │  │  Channel 1 ─────────────────────────┼───►│ S11
  │  │  Channel 2 ─────────────────────────┼───►│ S21 Safety Relay
  │  └─────────────────────────────────────┘    │
 0V ─────────────────────────────────────────────

เมื่อมีคนเหยียบ = Channel Short Circuit = Safety Relay OPEN
```

### Area Scanner (Laser Scanner)

```
Area Scanner ใช้ Laser สร้าง Safety Zone ได้หลาย Zone

Zone Layout:
                    Area Scanner
                         │
               ┌─────────┘─────────┐
               │                   │
     Warning   │    Protection     │
      Zone     │      Zone         │
      (Slow)   │      (Stop)       │
               │                   │
               └───────────────────┘
               ←── 2m ──►←── 1m ──►

Protection Zone: STOP Machine
Warning Zone: SLOW DOWN Machine
```

---

## Step 219: Risk Assessment พื้นฐาน

### กระบวนการ Risk Assessment (ISO 12100)

```
Risk Assessment Process:

1. กำหนด Machine Limits
   - Space Limits (พื้นที่ใช้งาน)
   - Time Limits (อายุการใช้งาน)
   - Use Limits (ผู้ใช้, Mode)

2. ระบุ Hazards
   - Mechanical (บาด, ตัด, บี้, หนีบ)
   - Electrical (ไฟช็อต)
   - Thermal (ความร้อน)
   - Chemical (สารเคมี)
   - Ergonomic (ท่าทาง)

3. ประเมินความเสี่ยง
   - Probability of Occurrence
   - Severity of Harm
   - Possibility to Avoid

4. ลดความเสี่ยง
   - Inherently Safe Design (ดีที่สุด)
   - Safeguarding (Safety devices)
   - Information & Warning (น้อยที่สุด)

5. Document และ Verify
```

### Risk Matrix

```
Risk Matrix (Probability vs Severity):

          │ Negligible │ Minor │ Moderate │ Serious │ Fatal
          │            │       │          │         │
──────────┼────────────┼───────┼──────────┼─────────┼───────
Rare      │   LOW      │  LOW  │  MEDIUM  │  HIGH   │  HIGH
Unlikely  │   LOW      │  LOW  │  MEDIUM  │  HIGH   │  CRITICAL
Possible  │   LOW      │ MEDIUM│   HIGH   │CRITICAL │  CRITICAL
Likely    │  MEDIUM    │ HIGH  │  CRITICAL│CRITICAL │  CRITICAL
Certain   │   HIGH     │ HIGH  │  CRITICAL│CRITICAL │  CRITICAL

LOW = Acceptable
MEDIUM = Monitor
HIGH = Reduce Risk
CRITICAL = Stop and Fix!
```

---

## Step 220: SISTEMA Tool

### SISTEMA คืออะไร

SISTEMA (Safety Integrity Software Tool for the Evaluation of Machine Applications) เป็นซอฟต์แวร์ฟรีจาก IFA (สถาบัน DGUV) ใช้สำหรับ:
- คำนวณ PL (Performance Level)
- ตรวจสอบว่า Safety Function ถึง PL ที่ต้องการหรือไม่
- สร้าง Safety Report

```
SISTEMA Calculation Process:

1. สร้าง Safety Function
   ├── Input (I): Safety Device (E-Stop, Light Curtain)
   ├── Logic (L): Safety Relay หรือ Safety PLC
   └── Output (O): Actuator (Contactor, Valve)

2. ใส่ข้อมูลแต่ละชิ้นส่วน:
   ├── B10d (Dangerous failures / 1M operations)
   ├── nop (Operations/year)
   └── Category

3. SISTEMA คำนวณ:
   ├── MTTFd (ของแต่ละ Channel)
   ├── DCavg (Diagnostic Coverage รวม)
   └── PL (ผลลัพธ์สุดท้าย)

4. เปรียบเทียบกับ PLr (Required PL)
   PLachieved ≥ PLrequired = PASS ✓
```

### ตัวอย่างการคำนวณด้วย SISTEMA

```
Example: E-Stop Circuit

Safety Function: Emergency Stop of Conveyor
Hazard: Operator trapped in conveyor
Required PL: PLe (Category 4)

Components:
┌─────────────────┬────────────────┬───────────────────────┐
│ Component       │ Type           │ Data                  │
├─────────────────┼────────────────┼───────────────────────┤
│ E-Stop 1        │ PILZ PSENpush  │ B10d=100,000          │
│ E-Stop 2        │ PILZ PSENpush  │ B10d=100,000          │
│ Safety Relay    │ PILZ PNOZ X2P  │ PL=e certified        │
│ Contactor KM1   │ Siemens 3RT    │ B10d=1,300,000        │
│ Contactor KM2   │ Siemens 3RT    │ B10d=1,300,000        │
└─────────────────┴────────────────┴───────────────────────┘

nop = 20 operations/hour × 8h/day × 250 days = 40,000/year

MTTFd per channel = B10d / (0.1 × nop)
                  = 100,000 / (0.1 × 40,000)
                  = 25 years (High category)

Category: 4 (Dual Channel + DC ≥ 99%)
DCavg: 99%

Result: PL = e ✓ (meets PLr = e)
```

---

## Step 221: IEC 62061 และ SIL

### SIL (Safety Integrity Level)

```
SIL Levels ตาม IEC 62061:

SIL 1: PFHD = 10^-5 to 10^-6 (failures/hour)
       ตัวอย่าง: Warning system, Simple guard

SIL 2: PFHD = 10^-6 to 10^-7
       ตัวอย่าง: Robot safety, Press guard

SIL 3: PFHD = 10^-7 to 10^-8
       ตัวอย่าง: Chemical plant, Nuclear (partial)

PFHD = Probability of Dangerous Failure per Hour
```

### SILCL (SIL Claim Limit)

```
ผู้ผลิตอุปกรณ์ Safety จะระบุ SILCL ว่า
อุปกรณ์สามารถใช้ใน SIL ได้สูงสุดเท่าไหร่

ตัวอย่าง:
Safety Relay PILZ PNOZ X2P: SILCL = 3
Light Curtain SICK C4000: SIL 3
E-Stop PILZ PSENpush: SILCL = 3

กฎ: SILCL ≥ SIL ที่ต้องการ
```

---

## Step 222: Safety PLC (Fail-Safe PLC)

### ความแตกต่าง Standard PLC กับ Safety PLC

```
Standard PLC:
- Single CPU
- ไม่มี Self-diagnostic
- ผิดพลาด = Unpredictable output
- ไม่เหมาะกับ Safety Function

Safety PLC (F-PLC / Fail-Safe PLC):
- Redundant CPU (2 CPU ทำงานคู่กัน)
- Cross-checking ระหว่าง CPU
- ผิดพลาด = Safe State (Output = 0)
- Certified ตามมาตรฐาน Safety
- ตัวอย่าง: 
  Siemens S7-1500F
  Allen-Bradley GuardLogix
  Mitsubishi MELSEC iQ-R Safety
```

### Safety PLC Architecture

```
Safety PLC Internal Structure:

     ┌──────────────────────────────────────────┐
     │             Safety PLC                    │
     │                                           │
     │  ┌─────────┐    ┌─────────┐              │
     │  │ CPU 1   │    │ CPU 2   │              │
     │  │(Primary)│    │(Check)  │              │
     │  └────┬────┘    └────┬────┘              │
     │       │              │                    │
     │       └──────┬───────┘                    │
     │              │                            │
     │         Comparator                        │
     │              │                            │
     │       ┌──────┴──────┐                     │
     │       │Same Result? │                     │
     │       └──────┬──────┘                     │
     │         YES  │  NO                        │
     │              │   └──► Safe State (Stop)   │
     │         Output OK                         │
     └──────────────────────────────────────────┘
```

### Siemens S7-1500F Safety Program

```ladder
// Safety Function ใน TIA Portal Safety Edition

// F-Block (Safety Function Block)
// ชื่อ Function Block ขึ้นต้นด้วย F_ เสมอ

Network 1: F_ESTOP1 (Safety E-Stop)
|               |
| "F_ESTOP1"    |
|    ┌─────┐   |
|    │ESTOP│   |
|    │     │   |
E_STOP─►│S_EStopIn│   
|    │     │   
|    │ SafetyActive ──►"Safety_OK"
|    │     │   
|    │ DiagCode ──►"ESTOP_Diag"
|    └─────┘   

// F-Block ผ่านการ Certify แล้ว ใช้แทน Custom Logic
// ห้ามเขียน Safety Logic เองใน Standard Block!
```

---

## Step 223: การออกแบบ Safety System

### Safety System Design Process

```
1. Define Safety Functions
   ตัวอย่าง: "หยุด Conveyor เมื่อ E-Stop ถูกกด"

2. Determine Required PLr/SILr
   ใช้ Risk Graph หรือ Risk Matrix

3. Select Architecture
   Category B/1/2/3/4 หรือ SIL 1/2/3

4. Select Components
   ตรวจสอบ PL/SIL ของแต่ละชิ้นส่วน

5. Calculate PL/SIL
   ใช้ SISTEMA หรือ คำนวณด้วยมือ

6. Verify PLachieved ≥ PLrequired

7. Document (Safety Report)

8. Validation Testing
   ทดสอบในทุก Fault Mode
```

### Safety Documentation Checklist

```
เอกสารที่ต้องมีสำหรับ Safety System:

☐ Risk Assessment Report
☐ Safety Function List
☐ Required PL/SIL for each function
☐ Safety Architecture Diagram
☐ Component Certificates (TÜV, CE)
☐ SISTEMA Calculation Report
☐ Safety Wiring Diagrams
☐ Test Procedure
☐ Validation Test Records
☐ User Manual (Safety Chapter)
☐ Maintenance Schedule
```

---

## Step 224: Validation Testing

### การทดสอบ Safety System

```
Validation Test Types:

1. Functional Test (ทดสอบการทำงาน)
   - กด E-Stop → Machine หยุด
   - Light Curtain Break → Machine หยุด
   - Safety Mat → Machine หยุด
   - ทุก Safety Function ต้องทำงาน

2. Fault Injection Test (ทดสอบเมื่อเกิด Fault)
   - ตัด Wire 1 ใน Dual Channel → System ต้องไม่ Start
   - Short 2 Channels → System ต้องตรวจพบ
   - ถอด Safety Device → Safe State

3. Reset Test
   - ปลด E-Stop แล้ว Machine ต้องไม่ Start Auto
   - ต้อง Reset ก่อน

4. Response Time Test
   - วัดเวลาจาก E-Stop กด ถึง Motor หยุด
   - ต้องน้อยกว่า Safety Distance Requirement
```

### Safety Distance Calculation

```
Minimum Safety Distance (ISO 13855):

S = K × T

S = Safety Distance (mm)
K = Approach Speed (mm/s)
    สำหรับมือ: K = 2000 mm/s
    สำหรับร่างกาย: K = 1600 mm/s
T = Total System Response Time (s)
    T = Ts (Stopping time) + Tc (Safety device response)

ตัวอย่าง:
Ts (Motor stop) = 0.5s
Tc (Light Curtain) = 0.015s
T = 0.5 + 0.015 = 0.515s

S = 2000 × 0.515 = 1030mm

ติดตั้ง Light Curtain ห่างจาก Danger Zone อย่างน้อย 1030mm!
```

---

## Step 225: Safety ใน Conveyor Application

### Safety Requirements สำหรับ Conveyor

```
Conveyor Safety Points:

1. Emergency Stop ทุกสถานี (ทุก 10m หรือน้อยกว่า)
2. Rope Pull Emergency Stop ตลอดความยาว
3. Safety Guard บริเวณ Nip Points
4. Tail End Guard (จุดที่ Belt วนกลับ)
5. Drive Guard (Motor, Drive Mechanism)
6. Zero Speed Detector (ป้องกัน Stall)
```

### Rope Pull Emergency Stop

```
Rope Pull E-Stop:

 Conveyor Frame
      │
      ├── Post ──[Rope]──────[Rope]──────[Rope]──── Post
      │            │                        │
      │         Switch A                 Switch B
      │         (NC)                     (NC)
      │
      │
All in Series: +24V──[A NC]──[B NC]──► Safety Relay

ดึง Rope ที่ไหน = ตรงนั้น Open → Safety Relay OPEN → STOP
```

---

## Step 226: Lockout/Tagout (LOTO)

### Lockout/Tagout คืออะไร

LOTO คือขั้นตอนการล็อคพลังงานก่อนซ่อมบำรุง เพื่อป้องกันการ Start Machine โดยไม่ตั้งใจ

```
LOTO Procedure:

1. NOTIFY - แจ้งผู้เกี่ยวข้องทุกคน

2. IDENTIFY - ระบุ Energy Sources ทั้งหมด
   (ไฟฟ้า, ลม, ไฮดรอลิก, ความร้อน, แรงโน้มถ่วง)

3. ISOLATE - ตัด/ปิด Energy Sources
   (ปิด Disconnect Switch, ปิด Valve)

4. LOCK - ใส่กุญแจล็อค (ช่างแต่ละคนใส่ Lock ของตัวเอง)

5. TAG - ติด Tag "DO NOT OPERATE"

6. VERIFY - ตรวจสอบว่า Energy หมดแล้ว
   (กด Start Button, วัดแรงดัน = 0V)

7. WORK - ทำงานได้อย่างปลอดภัย

8. RESTORE - ปิดงาน, ถอด Lock, คืนสู่ Operation
```

### LOTO Ladder Diagram Indication

```ladder
// LOTO Indication in PLC

Network 1: LOTO Status from Key Switch
|  %I2.0   |
| LOTO_KEY |──────────────────────────────────────( %M20.0 )
|(Key Switch|                                      LOTO_Active
| แบบ Key  |
| Release) |

Network 2: Block All Outputs When LOTO Active
|  %M20.0  |
| LOTO_Act |────/──────────────────────────────────( %Q0.0 )
|  (NC)    |                                        MOTOR_1
|  %M20.0  |
| LOTO_Act |────/──────────────────────────────────( %Q0.1 )
|  (NC)    |                                        MOTOR_2
```

---

## Step 227: Safety ใน Robot Applications

### Safety Zone สำหรับ Robot

```
Robot Safety Zones:

         ┌─────────────────────────────────┐
         │        Warning Zone (Yellow)    │
         │   ┌─────────────────────────┐  │
         │   │    Safeguarding Zone    │  │
         │   │   (Light Curtain/Gate) │  │
         │   │  ┌─────────────────┐  │  │
         │   │  │   Danger Zone   │  │  │
         │   │  │   (Max Reach)  │  │  │
         │   │  │   [ROBOT]      │  │  │
         │   │  └─────────────────┘  │  │
         │   └─────────────────────────┘  │
         └─────────────────────────────────┘

Warning Zone: Slow down production
Safeguarding: Stop robot motion
Danger Zone: Never enter while running
```

### Robot Safety Speed Monitoring

```ladder
// Robot Safety: Speed Monitoring with Safety PLC
// (Conceptual - ต้องใช้ Safety PLC จริงๆ)

Network 1: Check Robot in Safe Slowdown
|  Speed    |  Speed  |
| > 250mm/s |< 250mm/s|
|    (NC)   |         |──────────────────────( Safe_Speed )

Network 2: Guard Open vs Speed
|  GUARD   |  Safe   |
|  OPEN    |  Speed  |────────────────────( ALLOW_GUARD_OPEN )
|          |         |

Network 3: Emergency if Guard Open + Fast Speed
|  GUARD   |  Safe   |
|  OPEN    |  Speed  |
|          |  (NC)   |────────────────────( ROBOT_STOP )
```

---

## Step 228: Functional Safety Testing Tools

### การใช้เครื่องมือทดสอบ Safety

```
Safety Test Equipment:

1. Loop Calibrator
   - ทดสอบ 4-20mA Safety Transmitter
   - จำลอง Signal ต่างๆ

2. Safety Relay Tester
   - ทดสอบ Response Time
   - ทดสอบ Fault Detection

3. Oscilloscope
   - วัด Safety Response Time
   - ตรวจสอบ Signal Quality

4. PROFINET/EtherNet/IP Analyzer
   - ตรวจสอบ Safety over Network
   - Monitor Black Channel communication

5. SISTEMA Software
   - คำนวณ PL/MTTFd
   - Generate Safety Report
```

---

## Step 229: บทบาทและความรับผิดชอบ

### Safety Responsibilities

```
ผู้ออกแบบ (Designer):
- Risk Assessment
- Safety Architecture
- Safety Calculation (SISTEMA)
- Documentation

ผู้ผลิต/ติดตั้ง (Manufacturer/Installer):
- ติดตั้งตาม Drawing
- Validation Test
- Training

ผู้ใช้งาน (User):
- Follow Safety Procedures
- LOTO
- Report Hazards

ผู้บำรุงรักษา (Maintenance):
- Periodic Testing
- Replacement of Safety Devices
- Record Keeping
```

---

## Step 230: Lab Project - Safety Circuit Design

### Lab 12: ออกแบบ Safety Circuit สำหรับ Conveyor

**วัตถุประสงค์:**
1. ออกแบบ Safety Circuit ตาม Category 3, PLd
2. ต่อวงจร Safety Relay (PILZ PNOZ X2 หรือเทียบเท่า)
3. เขียน Ladder Program ที่ตอบสนอง Safety Signal
4. ทดสอบ Validation

**Specification:**
- E-Stop Mushroom Head x2 (Series)
- Safety Relay: Category 3
- Required PLr: PLd
- Conveyor Motor: 3-Phase 5.5kW
- Application: Food Processing Conveyor

**Safety Architecture (Category 3):**
```
                   E-Stop 1 (NC)
+24V ──┬──────────────────────────┐
       │         E-Stop 2 (NC)    │
       ├──────────────────────────┤
       │                          │
       │  PILZ PNOZ X2P           │
       │  ┌──────────────────────┐│
A1 ←──┤  │                      ││
S11 ─────┤ E-Stop 1 NC ──────────┤ S12
S21 ─────┤ E-Stop 2 NC ──────────┤ S22
Y11 ─────┤ Reset PB NO ──────────┤ Y12
       │  │                      ││
       │  │ 13/14 ───────────────── KM1 (Safety Output 1)
       │  │ 23/24 ───────────────── KM2 (Safety Output 2)
A2 ←──┤  │ 41/42 ───────────────── I0.5 (PLC Monitor)
       │  └──────────────────────┘│
      0V─────────────────────────┘
```

**Ladder Program:**
```ladder
// Lab 12: Safety Conveyor Control
// Siemens S7-1200

Network 1: Safety Status Monitor
|  %I0.5   |
|SAFETY_OK |──────────────────────────────────────( %M10.0 )
|(Safety   |                                       Safety_Good
| Relay    |
| Feedback)|

Network 2: Conveyor Start/Stop with Safety
|  %M10.0  |  %I0.0   |  %I0.1   |  %I0.2  |
|Safety_   |PB_START  |PB_STOP_NC|OL_NC    |────( %Q0.0 )
|Good      |          |          |         |    Conveyor
|          |                                      Motor
|  %Q0.0   |
|Conv_Motor|──────────────────────────────────────┘
|(Self     |
| Hold)    |

Network 3: Fault Alarm
|  %M10.0  |
|Safety_   |────/────────────────────────────────( %Q0.5 )
|Good (NC) |                                     Fault_Lamp

Network 4: Run Indicator
|  %Q0.0   |
|Conv_Motor|──────────────────────────────────────( %Q0.4 )
|          |                                      Run_Lamp

Network 5: Reset Button (Cannot Start, only Reset)
|  %I0.3   |  %M10.0  |
|RESET_PB  |Safety_OK |──────────────────────────( %M0.1 )
|          |          |                           Reset_Done
```

**Validation Test Checklist:**
```
☐ E-Stop 1 กด → Conveyor หยุด
☐ E-Stop 2 กด → Conveyor หยุด
☐ ทั้งสอง กด → Conveyor หยุด
☐ ปลด E-Stop → Conveyor ไม่ Start อัตโนมัติ
☐ กด Reset → Conveyor ไม่ Start (ต้องกด Start ด้วย)
☐ ถอด Wire S11 → Safety Relay ไม่ Enable
☐ ถอด Wire S21 → Safety Relay ไม่ Enable
☐ Overload → Conveyor หยุด
☐ Reset Time: ≤ 2s
```

---

## สรุป Part 12

ในบทนี้เราได้เรียนรู้:
1. มาตรฐาน Safety สากล (ISO 13849, IEC 62061)
2. Safety Categories B, 1, 2, 3, 4
3. Performance Level (PL a-e)
4. Safety Relay การออกแบบและการต่อวงจร
5. Emergency Stop Circuit Design
6. Two-Hand Control
7. Light Curtain และ Safety Mat
8. Risk Assessment ด้วย Risk Matrix
9. SISTEMA Tool สำหรับคำนวณ PL
10. Safety PLC (Fail-Safe PLC)
11. LOTO Procedure
12. Validation Testing

---

## การนำทาง

- [← Part 11: การต่อวงจร I/O](part11_io_wiring.md)
- [→ Part 13: Motor Control พื้นฐาน](part13_motor_control.md)
- [กลับหน้าหลัก](../README.md)

---

*Part 12 จบ | ต่อไป Part 13: Motor Control พื้นฐาน*
