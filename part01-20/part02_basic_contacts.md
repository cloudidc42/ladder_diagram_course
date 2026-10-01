# Part 02: Contact และ Coil พื้นฐาน
## Steps 16-30: รายละเอียดทุก Contact และ Coil Type

---

## Step 16: Normally Open (NO) Contact รายละเอียด

### NO Contact คืออะไร?
```
NORMALLY OPEN CONTACT - XIC (Examine If Closed)

Symbol:        ──[ ]──
IEC Symbol:    ──┤ ├──
Siemens:       ──| |──
Allen-Bradley: ──[  ]──  (XIC - Examine If Closed)

การทำงาน:
  ┌─────────────────────────────────────────────┐
  │  Bit Value = 0 (OFF):  Contact = OPEN       │
  │                         ──┤  ├── (ขาด)      │
  │  Bit Value = 1 (ON):   Contact = CLOSED     │
  │                         ──┤×├── (ต่อถึง)    │
  └─────────────────────────────────────────────┘

ตัวอย่าง:
  I0.0 = 0:  Power ไม่ผ่าน  ──[ I0.0 ]──  ขาด
  I0.0 = 1:  Power ผ่าน     ──[ I0.0 ]──  ต่อ
```

### ใช้กับ Physical Input (Sensor, Switch, Button)
```
WIRING FOR NO CONTACT:

Physical NO Switch (Momentary):
    PLC 24V DC ─────┬─────── I0.0
                    │
                   [/] ← Switch (Normally Open)
                    │
    PLC COM ────────┘

Physical NC Switch:
    PLC 24V DC ─────┬─────── I0.0
                    │
                   [x] ← Switch (Normally Closed)
                    │
    PLC COM ────────┘

IMPORTANT: Physical NC switch → Contact ใน PLC อาจเป็น NO
ต้องดูตาม Logic ที่ต้องการ ไม่ใช่ตาม Physical State เสมอ
```

---

## Step 17: Normally Closed (NC) Contact รายละเอียด

```
NORMALLY CLOSED CONTACT - XIO (Examine If Open)

Symbol:        ──[/]──
IEC Symbol:    ──┤/├──
Allen-Bradley: ──[/  ]──  (XIO - Examine If Open)

การทำงาน:
  ┌─────────────────────────────────────────────┐
  │  Bit Value = 0 (OFF):  Contact = CLOSED     │
  │                         ──┤×├── (ต่อถึง)    │
  │  Bit Value = 1 (ON):   Contact = OPEN       │
  │                         ──┤  ├── (ขาด)      │
  └─────────────────────────────────────────────┘

การใช้งานทั่วไป:
  1. Stop Button (E-Stop, Normal Stop)
  2. Interlocking (ป้องกัน 2 Output ทำงานพร้อมกัน)
  3. Seal-In Circuit (ด้านบน)
  4. Fault Conditions
```

### ตัวอย่างการใช้ NC Contact
```
EXAMPLE: MOTOR OVERLOAD PROTECTION

ฟิสิคัล:
  - OL1 = Overload Relay (Normally Closed Contact)
  - เมื่อ Motor กินกระแสเกิน → OL1 เปิด (Contact ขาด)
  - ต้องการ: ถ้า OL1 เปิด → หยุด Motor

Ladder:
    |──[ I0.0  ]──[/I0.5  ]──( Q0.0 )──|
    |  START_PB   OL_FAULT    MOTOR      |

  ปกติ (ไม่ Fault): I0.5 = 0 → Contact NC = ปิด → Motor ทำงาน
  เมื่อ Fault:      I0.5 = 1 → Contact NC = เปิด → Motor หยุด

สิ่งสำคัญ:
  I0.5 = Physical Overload Contact (NC Terminal)
  เมื่อ Overload Trip → Physical Contact เปิด → I0.5 = 0
  ใน Ladder ใช้ NO Contact [I0.5] → เมื่อ I0.5 = 0 Contact เปิด

  (หรือ) ต่อ Physical Contact แบบ NO เข้า PLC
  เมื่อ Overload Trip → Physical Contact ปิด → I0.5 = 1
  ใน Ladder ใช้ NC Contact [/I0.5] → เมื่อ I0.5 = 1 Contact เปิด
```

---

## Step 18: Output Coil (OTE/Standard Coil)

```
OUTPUT COIL - OTE (Output Energize)

Symbol:        ──( )──
IEC Symbol:    ──( )──
Siemens:       ──( )──
Allen-Bradley: ──(  )── (OTE)

การทำงาน:
  ┌─────────────────────────────────────────────┐
  │  มี Power Flow → Coil = 1 (ON/Energize)     │
  │  ไม่มี Power Flow → Coil = 0 (OFF/De-energize)│
  └─────────────────────────────────────────────┘

กฎสำคัญ:
  1. Coil จะ ON เฉพาะเมื่อ Rung มี Power Flow
  2. ทุก Scan ที่ไม่มี Power Flow → Coil = 0
  3. ใช้ Q (Output), M (Memory), หรือ ตัวแปรอื่น
  4. ห้ามใช้ Input Address (I) เป็น Coil!

MULTIPLE COILS ON SAME ADDRESS:
  ⚠️ ระวัง: ถ้าใช้ Address เดียวกันหลาย Coil
  ค่าสุดท้ายที่ Execute จะชนะ (Last Write Wins)

  Rung 1: ──[ I0.0 ]──( Q0.0 )──  ← Execute ก่อน
  Rung 5: ──[ I0.1 ]──( Q0.0 )──  ← Execute ทีหลัง ← ชนะ!
```

---

## Step 19: Set Coil (Latch) และ Reset Coil (Unlatch)

### Set/Reset คืออะไร?
```
SET COIL - S / OTL (Output Latch)

Symbol:        ──(S)──  หรือ  ──(L)──
Allen-Bradley: ──(OTL)── (Output Latch)

การทำงาน:
  มี Power Flow:     Coil = 1 (ON) และ ค้างอยู่!
  ไม่มี Power Flow:  ไม่เปลี่ยนแปลง (ยังคง ON)
  Reset เท่านั้น:    Coil จะ OFF


RESET COIL - R / OTU (Output Unlatch)

Symbol:        ──(R)──  หรือ  ──(U)──
Allen-Bradley: ──(OTU)── (Output Unlatch)

การทำงาน:
  มี Power Flow:     Coil = 0 (OFF) และ ค้างอยู่!
  ไม่มี Power Flow:  ไม่เปลี่ยนแปลง
```

### Set/Reset ตัวอย่าง
```
EXAMPLE: START/STOP WITH SET-RESET

Rung 1 (Start):
    |──[ I0.0  ]──────────(SET M0.0)──|
    |  START_PB            MOTOR_RUN   |

Rung 2 (Stop):
    |──[ I0.1  ]──────────(RST M0.0)──|
    |  STOP_PB             MOTOR_RUN   |

Rung 3 (Output):
    |──[ M0.0  ]──────────( Q0.0 )──|
    |  MOTOR_RUN            MOTOR    |

Timeline:
t=0: I0.0=0, I0.1=0, M0.0=0, Q0.0=0
t=1: I0.0=1 → SET M0.0 → M0.0=1, Q0.0=1
t=2: I0.0=0 → ไม่ Execute SET, แต่ M0.0 ยังคง=1!
t=3: I0.1=1 → RST M0.0 → M0.0=0, Q0.0=0
t=4: I0.1=0 → ไม่ Execute RST, M0.0 ยังคง=0

ADVANTAGE ของ Set/Reset:
✅ ไม่ต้องใช้ Seal-In Contact
✅ ง่ายกว่าในบาง Application
✅ Start/Stop จากหลายจุดได้ง่าย
```

---

## Step 20: Positive Edge Detect (P Contact)

```
POSITIVE TRANSITION CONTACT

Symbol:        ──[P]──  หรือ  ──↑──
IEC:           ──[P]──
Siemens:       ──[P]── (หรือ P_TRIG Function Block)
Allen-Bradley: ──[OSR]── (One Shot Rising)

การทำงาน:
  ┌──────────────────────────────────────────────────┐
  │  สัญญาณ Input:  0 0 0 1 1 1 1 0 0 0 1 1 0 0     │
  │  P Output:      0 0 0 1 0 0 0 0 0 0 1 0 0 0     │
  │                         ↑                ↑       │
  │              1 Scan เท่านั้น!    1 Scan เท่านั้น! │
  └──────────────────────────────────────────────────┘

ใช้เมื่อ: ต้องการทำงานเพียงครั้งเดียว ณ ขณะที่สัญญาณเปลี่ยน 0→1

ตัวอย่างการใช้งาน:
  - นับจำนวนครั้งที่กดปุ่ม
  - Trigger เฉพาะ Rising Edge
  - ป้องกัน Multiple Execution
```

### ตัวอย่าง Positive Edge
```
EXAMPLE: COUNT BUTTON PRESSES

ไม่มี Edge Detect (ผิด):
    |──[ I0.0 ]──[CTU C0 PV=10]──|

    กดปุ่มค้าง 10 Scans → นับ 10 ครั้ง! (ผิด)

ใช้ Edge Detect (ถูก):
    |──[ I0.0 ]──[P]──[CTU C0 PV=10]──|

    กดปุ่มค้าง 10 Scans → นับ 1 ครั้ง! (ถูก)
    เพราะ [P] ON เพียง 1 Scan
```

---

## Step 21: Negative Edge Detect (N Contact)

```
NEGATIVE TRANSITION CONTACT

Symbol:        ──[N]──  หรือ  ──↓──
IEC:           ──[N]──
Allen-Bradley: ──[OSF]── (One Shot Falling)

การทำงาน:
  ┌──────────────────────────────────────────────────┐
  │  สัญญาณ Input:  0 0 0 1 1 1 1 0 0 0 1 1 0 0     │
  │  N Output:      0 0 0 0 0 0 0 1 0 0 0 0 1 0     │
  │                                 ↑           ↑   │
  │                     1 Scan เท่านั้น!  1 Scan!   │
  └──────────────────────────────────────────────────┘

ใช้เมื่อ: ต้องการทำงานเมื่อสัญญาณเปลี่ยน 1→0

ตัวอย่างการใช้งาน:
  - ตรวจจับปล่อยปุ่ม (Button Release)
  - ตรวจจับสิ้นสุด Pulse
  - Trigger หลังจบ Operation
```

---

## Step 22: ความแตกต่าง Set/Reset กับ Standard Coil

```
COIL TYPE COMPARISON:

STANDARD COIL (OTE):
    |──[Condition]──( Q0.0 )──|

    Q0.0 ติดตาม Condition ทุก Scan
    ไม่มี Memory

    Timeline:
    Cond:  0 0 1 1 1 0 0 0 1 1 0
    Q0.0:  0 0 1 1 1 0 0 0 1 1 0  ← ตามเงื่อนไข 100%


SET + RESET COIL:
    |──[Start]──(SET Q0.1)──|
    |──[Stop ]──(RST Q0.1)──|

    Q0.1 มี Memory - ค้างสถานะ

    Timeline:
    Start: 0 0 1 0 0 0 0 0 0 0 0
    Stop:  0 0 0 0 0 0 1 0 0 0 0
    Q0.1:  0 0 1 1 1 1 0 0 0 0 0  ← SET→ค้าง→RESET→ค้าง


เมื่อไหร่ใช้อะไร:
  Standard Coil:  เมื่อ Output ต้องติดตาม Input ตลอด
                  เช่น Indicator Lamp, Signal Light

  Set/Reset Coil: เมื่อต้องการ Memory/Latch
                  เช่น Motor Start/Stop
                  เมื่อมี Multiple Start/Stop Points
```

---

## Step 23: Coil ประเภทพิเศษ

### Retentive Output Coil
```
RETENTIVE OUTPUT COIL

เมื่อ PLC ดับไฟ → ค่าถูกเก็บ (Retain)
เมื่อ PLC เปิดใหม่ → ค่ากลับมาเหมือนเดิม

ใช้กับ:
  - M1000.0 - M1499.7 (Siemens - Non-volatile Memory)
  - %M ที่ตั้งเป็น Retain (CODESYS)
  - Retentive Memory ใน Allen-Bradley

ตัวอย่างการใช้:
  - เก็บ State ของเครื่องจักร
  - เก็บ Counter ที่ไม่ต้องการ Reset เมื่อปิดเครื่อง
  - Production Count
```

### ตัวอย่างโปรแกรมสมบูรณ์
```
COMPLETE PROGRAM EXAMPLE:
Motor Control with Protection

TAG LIST:
  I0.0 = START_PB      (Momentary NO)
  I0.1 = STOP_PB       (Momentary NC Physical)
  I0.2 = OVERLOAD      (NC - Overload Relay Contact)
  I0.3 = HIGH_TEMP     (NO - Temperature Switch)
  Q0.0 = MOTOR_KM1     (Main Contactor)
  Q0.1 = FAULT_LAMP    (Fault Indicator)
  M0.0 = MOTOR_RUN_BIT (Internal Memory)
  M0.1 = FAULT_BIT     (Internal Fault Flag)

LADDER PROGRAM:

─── Network 1: Start/Stop Control ─────────────────────
  Comment: Start motor with Seal-in, Stop with NC contact
  
    |──[ I0.0 ]──┬──[/I0.1]──[/ M0.1]──(SET M0.0)──|
    |  START_PB  │  STOP_PB   NO_FAULT   MOTOR_RUN   |
    |            │                                    |
    |──[ M0.0 ]──┘                                   |
    |  MOTOR_FB                                       |

─── Network 2: Stop Conditions ────────────────────────
  Comment: Stop motor on any fault
  
    |──[ I0.1 ]──────────────────────(RST M0.0)──|
    |  STOP_PB                        MOTOR_RUN   |
    
    |──[ M0.1 ]──────────────────────(RST M0.0)──|
    |  FAULT_BIT                      MOTOR_RUN   |

─── Network 3: Output ─────────────────────────────────
  Comment: Drive physical motor contactor
  
    |──[ M0.0 ]──────────────────────( Q0.0 )──|
    |  MOTOR_RUN                       MOTOR    |

─── Network 4: Fault Detection ────────────────────────
  Comment: Set fault bit on any fault condition
  
    |──[/I0.2]──────────────────────(SET M0.1)──|
    |  OL_FAULT                       FAULT_BIT  |
    
    |──[ I0.3]──────────────────────(SET M0.1)──|
    |  HIGH_TEMP                      FAULT_BIT  |

─── Network 5: Fault Reset ────────────────────────────
  Comment: Manual fault reset
  
    |──[ I0.4]──────────────────────(RST M0.1)──|
    |  FAULT_RST                      FAULT_BIT  |

─── Network 6: Fault Lamp ─────────────────────────────
  Comment: Flash fault lamp (using Timer flash)
  
    |──[ M0.1 ]──────────────────────( Q0.1 )──|
    |  FAULT_BIT                       FAULT_LAMP|
```

---

## Step 24: Contact ซ้อนกัน (Complex Logic)

### AND-OR Combinations
```
COMPLEX LOGIC EXAMPLES:

Q = (A AND B) OR (C AND D)

    |──[ A ]──[ B ]──┐            |
    |                ├──( Q )──|
    |──[ C ]──[ D ]──┘            |


Q = A AND (B OR C)

    |──[ A ]──┬──[ B ]──┐──( Q )──|
    |         └──[ C ]──┘         |


Q = (A OR B) AND (C OR D)

    |──┬──[ A ]──┐──┬──[ C ]──┐──( Q )──|
    |  └──[ B ]──┘  └──[ D ]──┘         |
```

### Boolean Expression to Ladder
```
CONVERT BOOLEAN TO LADDER:

Expression: Q = (I0.0 OR I0.1) AND I0.2 AND (NOT I0.3)

Step 1: Identify structure
  (I0.0 OR I0.1) = Parallel Branch
  AND I0.2 = Series after branch
  AND NOT I0.3 = Series, NC Contact

Step 2: Draw Ladder
    |──┬──[ I0.0 ]──┐──[ I0.2 ]──[/I0.3]──( Q0.0 )──|
    |  └──[ I0.1 ]──┘                                 |

ทดสอบ Truth Table:
  I0.0=1, I0.1=0, I0.2=1, I0.3=0 → Q = (1 OR 0) AND 1 AND 1 = 1 ✓
  I0.0=0, I0.1=0, I0.2=1, I0.3=0 → Q = (0 OR 0) AND 1 AND 1 = 0 ✓
  I0.0=1, I0.1=0, I0.2=1, I0.3=1 → Q = (1 OR 0) AND 1 AND 0 = 0 ✓
```

---

## Step 25: Interlock Circuit

### ป้องกัน Forward/Reverse พร้อมกัน
```
MOTOR INTERLOCK (Forward/Reverse)

โจทย์: Motor มี Forward (KM1) และ Reverse (KM2)
ป้องกัน: ห้าม KM1 และ KM2 ON พร้อมกัน

Ladder:
─── Network 1: Forward ─────────────────────────────────
    |──[ I0.0  ]──┬──[/I0.1]──[/Q0.1]──( Q0.0 )──|
    |  FWD_START  │  REV_START  REV_KM2   FWD_KM1   |
    |             │                                  |
    |──[ Q0.0  ]──┘                                 |
    |  FWD_FB                                        |

─── Network 2: Reverse ─────────────────────────────────
    |──[ I0.1  ]──┬──[/I0.0]──[/Q0.0]──( Q0.1 )──|
    |  REV_START  │  FWD_START  FWD_KM1   REV_KM2   |
    |             │                                  |
    |──[ Q0.1  ]──┘                                 |
    |  REV_FB                                        |

การทำงาน:
  1. กด Forward → Q0.0=1, [/Q0.0] ใน Rung 2 เปิด → Q0.1 ไม่ได้
  2. กด Reverse → Q0.1=1, [/Q0.1] ใน Rung 1 เปิด → Q0.0 ไม่ได้
  3. Safety: ทั้งคู่ป้องกันกันเอง

ข้อสำคัญ:
  - ต้องมี Hardware Interlock ด้วย (ไม่พอแค่ Software)
  - ใช้ Auxiliary Contact ของ KM1 กับ KM2 ด้วย
```

---

## Step 26: One-Shot Timer Application

### Single Pulse Output
```
ONE-SHOT PULSE CIRCUIT

ต้องการ: กดปุ่ม I0.0 ครั้งเดียว → Output Q0.0 ON 3 วินาที แล้วดับ

Ladder:
─── Network 1: Trigger One-Shot ────────────────────────
    |──[ I0.0 ]──[P]──────────────(SET M0.0)──|
    |  BUTTON    RISE_EDGE          TRIGGER    |

─── Network 2: Timer ───────────────────────────────────
    |──[ M0.0 ]──[TON T0 PT=3s]──────────────|
    |  TRIGGER      3 sec Timer               |

─── Network 3: Output ──────────────────────────────────
    |──[ M0.0 ]──────────────────────( Q0.0 )──|
    |  TRIGGER                        OUTPUT   |

─── Network 4: Reset ───────────────────────────────────
    |──[ T0.Q ]──────────────────(RST M0.0)──|
    |  TIMER_DONE                   TRIGGER   |

Timeline:
  t=0: กดปุ่ม → I0.0=1 → [P] ON → SET M0.0=1
  t=0.1: ปล่อยปุ่ม → I0.0=0 → M0.0 ยังคง=1 (เพราะ SET)
  t=3: T0 Done → RST M0.0=0 → Q0.0=0
```

---

## Step 27: หลักการ Program Structure

```
GOOD PROGRAMMING PRACTICES:

1. TOP-DOWN STRUCTURE
   ─────────────────────────────────────────
   Rung 1-N:   Fault Detection & Safety
   Rung N+1:   Enable/Disable Logic
   Rung N+2:   Main Control Logic
   Rung N+3:   Output Assignment
   Rung Last:  Diagnostics & Communication

2. COMMENT EVERY RUNG
   ─────────────────────────────────────────
   ✅ ดี:
   "Network 5: Start motor only if no fault and door closed"
   
   ❌ ไม่ดี:
   "Network 5" หรือ ไม่มี Comment

3. USE MEANINGFUL TAG NAMES
   ─────────────────────────────────────────
   ✅ ดี:
   MOTOR_START_PB     (I0.0)
   MOTOR_STOP_PB      (I0.1)
   MOTOR_OL_FAULT     (I0.2)
   MOTOR_CONTACTOR    (Q0.0)

   ❌ ไม่ดี:
   I0, I1, I2, Q0

4. COIL DUPLICATION RULE
   ─────────────────────────────────────────
   ❌ ผิด: ใช้ Output Coil เดียวกัน 2 Rung
   Rung 3: ──[ I0.0 ]──( Q0.0 )──
   Rung 7: ──[ I0.1 ]──( Q0.0 )──  ← Last Scan Wins!

   ✅ ถูก: รวมเงื่อนไขใน Rung เดียว
   Rung 3: ──[ I0.0 ]──┐
           ──[ I0.1 ]──┴──( Q0.0 )──
   
   หรือใช้ Set/Reset:
   Rung 3: ──[ I0.0 ]──(SET Q0.0)──
   Rung 7: ──[ I0.1 ]──(RST Q0.0)──
```

---

## Step 28: Debug Contact และ Coil

### วิธี Monitor ค่าใน Software
```
ONLINE MONITORING:

เมื่อ Connected กับ PLC ใน Monitor Mode:

  Contact ที่ ON:   ──[■]──  (Filled/Highlighted)
  Contact ที่ OFF:  ──[ ]──  (Empty)
  
  Coil ที่ ON:      ──(■)──  (Filled/Highlighted)
  Coil ที่ OFF:     ──( )──  (Empty)

Power Flow:
  Power ผ่าน:       ═══════  (Bold Line)
  Power ไม่ผ่าน:    ───────  (Normal Line)

ตัวอย่างการ Debug:
  Problem: Q0.0 ไม่ ON ทั้งที่กดปุ่มแล้ว

  Step 1: Monitor Rung ที่ควบคุม Q0.0
  Step 2: ดู Power Flow ไหลถึงไหน
  Step 3: หา Contact ที่ไม่ยอม Pass Power
  Step 4: ตรวจสอบ Physical Wiring / Sensor
```

### Force I/O
```
FORCE I/O FOR TESTING:

Force Input ON:  บังคับ I0.0 = 1 (ไม่ต้องกดปุ่มจริง)
Force Input OFF: บังคับ I0.0 = 0
Force Output ON: บังคับ Q0.0 = 1 (ไม่ต้องรอ Logic)

คำเตือน:
  ⚠️ ห้ามใช้ Force Output บน Machine ที่ Running!
  ⚠️ Force ทำให้ Safety Logic ไม่ทำงาน!
  ⚠️ ลบ Force ทุกอันก่อน Commission!

ใช้สำหรับ:
  - Test Wiring ก่อน Program ลง
  - Debug เฉพาะ Hardware
  - Commission Testing
```

---

## Step 29: Contact Bounce และ Debounce

### สัญญาณรบกวนจาก Mechanical Switch
```
CONTACT BOUNCE PROBLEM:

Physical Button Press:
           ┌──────────────────────────────
           │  Ideal Signal
    ───────┘

           ┌─┐ ┌──┐  ┌────────────────────
           │ └─┘  └──┘
    ───────┘            ← Bounce (5-20ms)
           │← Bounce →│

ปัญหา: PLC อาจนับ 3-5 ครั้งจากการกด 1 ครั้ง!

วิธีแก้ 1: Hardware Debounce
  ต่อ RC Circuit (R=10kΩ, C=0.1µF) ขนาน Switch

วิธีแก้ 2: Software Debounce (ใน Ladder)
  ─── Network 1: Debounce Timer ────────────────────────
      |──[ I0.0 ]──[TON T_DB PT=20ms]──|
      |  RAW_INPUT   Debounce Timer     |
  
  ─── Network 2: Clean Signal ──────────────────────────
      |──[T_DB.Q]──────────────────( M0.0 )──|
      |  TIMER_DONE                 CLEAN_INPUT|

  ผล: สัญญาณจะ ON หลังจาก Stable 20ms → ไม่มี Bounce
```

---

## Step 30: แบบฝึกหัดและ Project Part 02

### Lab Project 02: Door Control System
```
PROJECT: AUTOMATIC DOOR CONTROL

Hardware:
  I0.0 = IR_SENSOR_ENTER  (IR Sensor ด้านนอก, NO)
  I0.1 = IR_SENSOR_EXIT   (IR Sensor ด้านใน, NO)
  I0.2 = DOOR_OPEN_LS     (Limit Switch เปิดสุด, NO)
  I0.3 = DOOR_CLOSE_LS    (Limit Switch ปิดสุด, NO)
  I0.4 = EMERGENCY_STOP   (E-Stop, NC)
  Q0.0 = MOTOR_OPEN       (Motor เปิดประตู)
  Q0.1 = MOTOR_CLOSE      (Motor ปิดประตู)
  Q0.2 = ALARM_BUZZER     (Buzzer เตือน)
  M0.0 = DOOR_OPEN_CMD    (คำสั่งเปิด)
  M0.1 = DOOR_CLOSE_CMD   (คำสั่งปิด)
  T0   = AUTO_CLOSE_TMR   (Timer ปิดอัตโนมัติ 5 วินาที)

Logic:
  1. ตรวจ Sensor เข้า/ออก → เปิดประตู
  2. หลังเปิด ถ้าไม่มีคนอยู่ 5 วินาที → ปิดประตู
  3. มี Interlock: ห้ามเปิด-ปิดพร้อมกัน
  4. E-Stop: หยุดทุกอย่าง

ให้ผู้เรียนเขียน Ladder Diagram เอง
เฉลยจะอยู่ใน Part 04 (หลังเรียน Timer แล้ว)
```

### ข้อสอบ Step 30
```
QUIZ:

1. Contact ชนิดใดที่จะ ON เพียง 1 Scan เมื่อสัญญาณ 0→1?
   ก) NO Contact    ข) NC Contact
   ค) P Contact     ง) SET Coil

2. Set Coil ต่างจาก Standard Coil อย่างไร?

3. เขียน Ladder สำหรับ: Q = A AND (NOT B) AND (C OR D)

4. ทำไมต้องใช้ Edge Detect Contact เมื่อนับจำนวนครั้ง?

5. Interlock Circuit มีประโยชน์อย่างไรในงาน Motor Control?
```

---

*[← Part 01: บทนำ] [→ Part 03: Logic Gates ใน Ladder]*
