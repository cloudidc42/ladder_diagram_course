# Part 03: Logic Gates ใน Ladder Diagram
## Steps 31-50: AND, OR, NOT, XOR, NAND, NOR และการประยุกต์ใช้

---

## Step 31: พื้นฐาน Boolean Algebra

```
BOOLEAN ALGEBRA FUNDAMENTALS:

Laws ที่ต้องรู้:
┌────────────────────────────────────────────────────────┐
│ Identity:      A + 0 = A       A · 1 = A              │
│ Null:          A + 1 = 1       A · 0 = 0              │
│ Idempotent:    A + A = A       A · A = A              │
│ Complement:    A + A' = 1      A · A' = 0             │
│ Double Neg:    (A')' = A                              │
│ Commutative:   A + B = B + A   A · B = B · A         │
│ Associative:   (A+B)+C = A+(B+C)                     │
│ Distributive:  A(B+C) = AB + AC                      │
│ DeMorgan 1:    (A + B)' = A' · B'                    │
│ DeMorgan 2:    (A · B)' = A' + B'                    │
└────────────────────────────────────────────────────────┘
```

### DeMorgan's Theorem ใน Ladder
```
DE MORGAN'S THEOREM APPLICATION:

DeMorgan 1: NOT(A OR B) = (NOT A) AND (NOT B)

วิธีที่ 1: ใช้ NOT ทั้ง Output
    |──┬──[ A ]──┐──(NOT Q)──|
    |  └──[ B ]──┘           |
    
วิธีที่ 2: ใช้ NC Contacts (De Morgan equivalent)
    |──[/A]──[/B]──( Q )──|

ทั้งสองวิธีให้ผลเหมือนกัน!

DeMorgan 2: NOT(A AND B) = (NOT A) OR (NOT B)

วิธีที่ 1:
    |──[ A ]──[ B ]──(NOT Q)──|

วิธีที่ 2:
    |──[/A]──┐
    |        ├──( Q )──|
    |──[/B]──┘
```

---

## Step 32: AND Gate ใน Ladder

```
AND GATE:
Y = A AND B AND C

Truth Table:
┌───┬───┬───┬───┐
│ A │ B │ C │ Y │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
│ 0 │ 0 │ 1 │ 0 │
│ 0 │ 1 │ 0 │ 0 │
│ 0 │ 1 │ 1 │ 0 │
│ 1 │ 0 │ 0 │ 0 │
│ 1 │ 0 │ 1 │ 0 │
│ 1 │ 1 │ 0 │ 0 │
│ 1 │ 1 │ 1 │ 1 │ ← ต้องครบทุกตัว
└───┴───┴───┴───┘

Ladder:
    |──[ A ]──[ B ]──[ C ]──( Y )──|

ตัวอย่างจริง: เครื่องจักรทำงานเมื่อ
  - ประตู Safety ปิด (DOOR_CLOSED)
  - E-Stop ไม่ถูกกด (E_STOP_OK)
  - Mode Switch ที่ AUTO (AUTO_MODE)

    |──[DOOR_CLOSED]──[E_STOP_OK]──[AUTO_MODE]──(MACHINE_RUN)──|
```

---

## Step 33: OR Gate ใน Ladder

```
OR GATE:
Y = A OR B OR C

Truth Table:
┌───┬───┬───┬───┐
│ A │ B │ C │ Y │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
│ 0 │ 0 │ 1 │ 1 │ ← อย่างน้อย 1 ตัว ON
│ 0 │ 1 │ 0 │ 1 │
│ 0 │ 1 │ 1 │ 1 │
│ 1 │ 0 │ 0 │ 1 │
│ 1 │ 0 │ 1 │ 1 │
│ 1 │ 1 │ 0 │ 1 │
│ 1 │ 1 │ 1 │ 1 │
└───┴───┴───┴───┘

Ladder:
    |──[ A ]──┐       |
    |         ├──( Y )─|
    |──[ B ]──┤       |
    |         │       |
    |──[ C ]──┘       |

ตัวอย่างจริง: Alarm เมื่อมีอย่างน้อย 1 Fault
  - OverLoad Trip
  - High Temperature
  - Low Pressure

    |──[ OL_FAULT ]──┐               |
    |                ├──(ALARM_ON)──|
    |──[ HIGH_TEMP]──┤               |
    |                │               |
    |──[ LOW_PRES ]──┘               |
```

---

## Step 34: NOT Gate ใน Ladder

```
NOT GATE (Inverter):
Y = NOT A

Truth Table:
┌───┬───┐
│ A │ Y │
├───┼───┤
│ 0 │ 1 │
│ 1 │ 0 │
└───┴───┘

Ladder:
    |──[/A]──( Y )──|   ← NC Contact = NOT Function

ตัวอย่างจริง: แสดง OK เมื่อไม่มี Fault
    |──[/ FAULT]──( OK_LAMP )──|
    
    FAULT=0 → OK_LAMP=1 (ติด)
    FAULT=1 → OK_LAMP=0 (ดับ)
```

---

## Step 35: NAND Gate ใน Ladder

```
NAND GATE:
Y = NOT(A AND B) = (NOT A) OR (NOT B)  ← De Morgan

Truth Table:
┌───┬───┬───┐
│ A │ B │ Y │
├───┼───┼───┤
│ 0 │ 0 │ 1 │
│ 0 │ 1 │ 1 │
│ 1 │ 0 │ 1 │
│ 1 │ 1 │ 0 │ ← เฉพาะทั้งคู่ ON เท่านั้นที่ Y=0
└───┴───┴───┘

Ladder (วิธีที่ 1 - De Morgan form):
    |──[/A]──┐       |
    |        ├──( Y )─|
    |──[/B]──┘       |

Ladder (วิธีที่ 2 - Direct NOT):
    |──[ A ]──[ B ]──(/ Y)──|  
    (หมายเหตุ: บาง PLC ไม่มี NC Coil ต้องใช้วิธีที่ 1)

ตัวอย่างจริง: Warning เมื่อไม่ครบทั้ง 2 เงื่อนไข
    |──[/SENSOR_A]──┐               |
    |               ├──(WARNING)──|
    |──[/SENSOR_B]──┘               |
    
    Warning ดับเฉพาะเมื่อ Sensor A และ B ทั้งคู่ ON
```

---

## Step 36: NOR Gate ใน Ladder

```
NOR GATE:
Y = NOT(A OR B) = (NOT A) AND (NOT B)  ← De Morgan

Truth Table:
┌───┬───┬───┐
│ A │ B │ Y │
├───┼───┼───┤
│ 0 │ 0 │ 1 │ ← เฉพาะทั้งคู่ OFF เท่านั้น Y=1
│ 0 │ 1 │ 0 │
│ 1 │ 0 │ 0 │
│ 1 │ 1 │ 0 │
└───┴───┴───┘

Ladder:
    |──[/A]──[/B]──( Y )──|

ตัวอย่างจริง: ระบบ OK เฉพาะเมื่อไม่มี Fault ใดๆ
    |──[/FAULT1]──[/FAULT2]──[/FAULT3]──(SYSTEM_OK)──|
    
    SYSTEM_OK = ON เมื่อทุก Fault = 0 เท่านั้น
```

---

## Step 37: XOR Gate ใน Ladder

```
XOR (Exclusive OR):
Y = A XOR B = (A AND NOT B) OR (NOT A AND B)

Truth Table:
┌───┬───┬───┐
│ A │ B │ Y │
├───┼───┼───┤
│ 0 │ 0 │ 0 │
│ 0 │ 1 │ 1 │ ← ต่างกัน = 1
│ 1 │ 0 │ 1 │ ← ต่างกัน = 1
│ 1 │ 1 │ 0 │ ← เหมือนกัน = 0
└───┴───┴───┘

Ladder:
    |──[ A ]──[/B]──┐       |
    |               ├──( Y )─|
    |──[/A]──[ B ]──┘       |

ตัวอย่างจริง: ตรวจจับ Position ไม่ถูกต้อง
  Sensor_A และ Sensor_B ต้องมีค่าเท่ากันเสมอ
  (ทั้งคู่ ON หรือทั้งคู่ OFF)
  ถ้าต่างกัน = Error

    |──[ SEN_A ]──[/SEN_B]──┐               |
    |                       ├──( SENSOR_ERR)─|
    |──[/SEN_A]──[ SEN_B ]──┘               |
```

---

## Step 38: XNOR Gate ใน Ladder

```
XNOR (Exclusive NOR):
Y = NOT(A XOR B) = (A AND B) OR (NOT A AND NOT B)

Truth Table:
┌───┬───┬───┐
│ A │ B │ Y │
├───┼───┼───┤
│ 0 │ 0 │ 1 │ ← เหมือนกัน = 1
│ 0 │ 1 │ 0 │
│ 1 │ 0 │ 0 │
│ 1 │ 1 │ 1 │ ← เหมือนกัน = 1
└───┴───┴───┘

Ladder:
    |──[ A ]──[ B ]──┐       |
    |                ├──( Y )─|
    |──[/A]──[/B]───┘       |

ตัวอย่างจริง: ตรวจสอบ Redundancy Sensor
  ต้องการให้ 2 Sensor มีค่าเท่ากัน (AGREE)
  ถ้าต่างกัน → Fault
  
    |──[ SEN_A ]──[ SEN_B ]──┐               |
    |                        ├──( SENSORS_AGREE)─|
    |──[/SEN_A]──[/SEN_B ]──┘               |
```

---

## Step 39: Logic Simplification

### Karnaugh Map (K-Map)
```
KARNAUGH MAP FOR LOGIC MINIMIZATION:

Problem: ลดรูป Q = A'B'C + A'BC + AB'C + ABC'

Truth Table:
┌───┬───┬───┬───┐
│ A │ B │ C │ Q │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
│ 0 │ 0 │ 1 │ 1 │ ← m1
│ 0 │ 1 │ 0 │ 0 │
│ 0 │ 1 │ 1 │ 1 │ ← m3
│ 1 │ 0 │ 0 │ 0 │
│ 1 │ 0 │ 1 │ 1 │ ← m5
│ 1 │ 1 │ 0 │ 1 │ ← m6
│ 1 │ 1 │ 1 │ 0 │
└───┴───┴───┴───┘

K-Map (3 Variables):
     BC
AB   00  01  11  10
00    0   1   1   0
10    0   1   0   1

Groups:
  Group 1 (m1, m3): A=0, C=1 → A'C
  Group 2 (m3, m5): ??? (ไม่ติดกัน)
  Group 3 (m5, m6): A=1, B+C different → AB' + AB แต่... 
  
Simplified: Q = A'C + AB'C + ABC' (ยังไม่ได้ลดสุด)
ลดแล้ว:     Q = A'C + A(B XOR C)

ใน Ladder:
Ladder Q = A'C + AB'C + ABC':

    |──[/A]──[ C ]──────────────────┐         |
    |                               ├──( Q )──|
    |──[ A ]──[/B]──[ C ]──────────┤         |
    |                               │         |
    |──[ A ]──[ B ]──[/C ]──────────┘         |
```

---

## Step 40: ตัวอย่าง Logic ที่ใช้จริง

### Conveyor Belt Control
```
CONVEYOR BELT CONTROL LOGIC:

System:
  - Conveyor A (Q0.0) → ป้อนวัสดุ
  - Conveyor B (Q0.1) → รับวัสดุ  
  - Conveyor C (Q0.2) → ส่งออก

Rule:
  1. Conveyor B ต้องทำงานก่อน Conveyor A เสมอ
     (ป้องกัน วัสดุค้างบน A)
  2. Conveyor A ทำงานได้เมื่อ B ทำงานอยู่
  3. Conveyor C ทำงานได้อิสระ
  4. E-Stop หยุดทุกตัว

I/O List:
  I0.0 = START_A_PB    ปุ่มเริ่ม Conveyor A
  I0.1 = START_B_PB    ปุ่มเริ่ม Conveyor B
  I0.2 = START_C_PB    ปุ่มเริ่ม Conveyor C
  I0.3 = STOP_ALL_PB   ปุ่มหยุดทั้งหมด
  I0.4 = E_STOP        Emergency Stop (NC)
  I0.5 = OL_A          Overload A (NC)
  I0.6 = OL_B          Overload B (NC)
  I0.7 = OL_C          Overload C (NC)
  Q0.0 = CONV_A        Conveyor A Motor
  Q0.1 = CONV_B        Conveyor B Motor
  Q0.2 = CONV_C        Conveyor C Motor

LADDER PROGRAM:

─── Network 1: System Enable ───────────────────────────
  Comment: System enabled when E-Stop OK and no Stop pressed

    |──[/I0.4]──[/I0.3]──────────────( M0.0 )──|
    |  E_STOP    STOP_ALL              SYS_ENABLE |

─── Network 2: Conveyor B Control ─────────────────────
  Comment: Start B - no interlock needed (receives from A)

    |──[ I0.1  ]──┬──[ M0.0 ]──[/I0.6]──( Q0.1 )──|
    |  START_B_PB │  SYS_EN    OL_B       CONV_B    |
    │             │                                  |
    |──[ Q0.1  ]──┘                                 |
    |  CONV_B_FB                                     |

─── Network 3: Conveyor A Control ─────────────────────
  Comment: Start A only when B is running (Interlock)

    |──[ I0.0  ]──┬──[ M0.0 ]──[ Q0.1 ]──[/I0.5]──( Q0.0 )──|
    |  START_A_PB │  SYS_EN    CONV_B      OL_A      CONV_A    |
    │             │                                             |
    |──[ Q0.0  ]──┘                                            |
    |  CONV_A_FB                                                |

─── Network 4: Conveyor C Control ─────────────────────
  Comment: Start C independently

    |──[ I0.2  ]──┬──[ M0.0 ]──[/I0.7]──( Q0.2 )──|
    |  START_C_PB │  SYS_EN    OL_C       CONV_C    |
    │             │                                  |
    |──[ Q0.2  ]──┘                                 |
    |  CONV_C_FB                                     |

─── Network 5: Cascade Stop ───────────────────────────
  Comment: If B stops, A must stop too (no material pileup)

    |──[/Q0.1]──────────────────────(RST Q0.0)──|
    |  CONV_B                          CONV_A    |
```

---

## Step 41: Safety Logic Design

```
SAFETY LOGIC PRINCIPLES:

1. FAIL SAFE PRINCIPLE
   ─────────────────────────────────────────
   ระบบต้องอยู่ในสถานะ SAFE เมื่อ:
   - ไฟฟ้าดับ
   - สายขาด
   - PLC Fault

   ตัวอย่าง:
   ❌ ผิด: Normally Open Valve ควบคุมก๊าซ
     (ไฟดับ → Valve เปิด → อันตราย!)
   
   ✅ ถูก: Normally Closed Valve ควบคุมก๊าซ
     (ไฟดับ → Valve ปิด → ปลอดภัย!)

2. SAFETY INPUT WIRING
   ─────────────────────────────────────────
   Safety Devices (E-Stop, Guard, etc.) ต้องต่อเป็น:
   - Normally Closed (NC) ที่ Input PLC
   - วงจรขาดทุกกรณีจะ = STOP

   Correct:
   24V ──┬── E-STOP (NC) ── I0.0 (PLC)
         └── Ground
   
   ปกติ: E-Stop ไม่ถูกกด → NC ปิด → I0.0=1
   กด E-Stop: NC เปิด → I0.0=0 → Machine Stop
   สายขาด:  → I0.0=0 → Machine Stop (SAFE!)

3. TWO-HAND CONTROL
   ─────────────────────────────────────────
   ต้องกด 2 ปุ่มพร้อมกัน (ทั้งสองมือ) จึงทำงาน
   ป้องกันไม่ให้มือค้างอยู่ในเครื่อง

   Ladder:
   |──[ L_HAND ]──[ R_HAND ]──( MACHINE )──|
   
   ต้องกดทั้งสองพร้อมกัน (ภายใน 0.5s)
   ป้องกัน Stick/Clamp:
   
   |──[ L_HAND_P ]──(SET M0.0)──|  ← Rising Edge
   |──[ R_HAND_P ]──(SET M0.1)──|
   
   |──[ M0.0 ]──[ M0.1 ]──(MACHINE)──|
   
   |──(Timer ตรวจ 0.5s)──|
   If เกิน 0.5s: Reset M0.0, M0.1
```

---

## Step 42: ระบบ Voting Logic

```
VOTING LOGIC (Majority Vote):

2-out-of-3 Voting System:
  System เปิดเมื่อ อย่างน้อย 2 ใน 3 Sensor ตรวจเจอ

Truth Table:
┌───┬───┬───┬───┐
│ A │ B │ C │ Y │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
│ 0 │ 0 │ 1 │ 0 │
│ 0 │ 1 │ 0 │ 0 │
│ 0 │ 1 │ 1 │ 1 │ ← 2 ใน 3
│ 1 │ 0 │ 0 │ 0 │
│ 1 │ 0 │ 1 │ 1 │ ← 2 ใน 3
│ 1 │ 1 │ 0 │ 1 │ ← 2 ใน 3
│ 1 │ 1 │ 1 │ 1 │ ← 3 ใน 3
└───┴───┴───┴───┘

Boolean: Y = AB + BC + AC

Ladder:
    |──[ A ]──[ B ]──┐       |
    |                ├──( Y )─|
    |──[ B ]──[ C ]──┤       |
    |                │       |
    |──[ A ]──[ C ]──┘       |

ใช้ในงาน: Tank Level, Fire Detection, Safety System
```

---

## Step 43: ตัวอย่างโปรแกรมซับซ้อน

### Pump Alternation System
```
PUMP ALTERNATION SYSTEM:
  มีปั๊ม 2 ตัว (P1, P2)
  ทำงานสลับกัน (ใช้งานทีละตัว)
  เมื่อปั๊มหลักเสีย → สลับไปปั๊มสำรองอัตโนมัติ

I/O:
  I0.0 = PUMP_START      ปุ่ม Start
  I0.1 = PUMP_STOP       ปุ่ม Stop
  I0.2 = OL_PUMP1        Overload Pump1 (NC)
  I0.3 = OL_PUMP2        Overload Pump2 (NC)
  I0.4 = LEVEL_LOW       Level Switch ต่ำ
  I0.5 = LEVEL_HIGH      Level Switch สูง
  Q0.0 = PUMP1_KM        Pump 1 Contactor
  Q0.1 = PUMP2_KM        Pump 2 Contactor
  Q0.2 = PUMP1_LED       Pump 1 Indicator
  Q0.3 = PUMP2_LED       Pump 2 Indicator
  M0.0 = SYSTEM_RUN      System Running Flag
  M0.1 = P1_SELECTED     Pump 1 Selected
  M0.2 = P1_FAULT        Pump 1 Fault
  M0.3 = P2_FAULT        Pump 2 Fault

LADDER PROGRAM:

─── Network 1: System Start/Stop ────────────────────────
    |──[ I0.0  ]──┬──( SET M0.0 )──|
    |  SYS_START  │   SYS_RUN      |
    |             │                 |
    |──[ M0.0  ]──┘                 |

    |──[ I0.1 ]──( RST M0.0 )──|
    |  SYS_STOP   SYS_RUN       |

─── Network 2: Initial Selection (P1 default) ──────────
    |──[ M0.0 ]──[/M0.1]──[/M0.2]──( SET M0.1 )──|
    |  SYS_RUN   P1_NOT_SEL  P1_OK    P1_SELECT   |

─── Network 3: Pump 1 Fault Detection ──────────────────
    |──[/I0.2 ]──[ M0.0 ]──( SET M0.2 )──|
    |  OL_P1      SYS_RUN    P1_FAULT    |
    
    |──[/M0.0 ]──( RST M0.2 )──|
    |  SYS_STOP    P1_FAULT    |

─── Network 4: Switch to Pump 2 on Fault ───────────────
    |──[ M0.2 ]──[ M0.1 ]──( RST M0.1 )──|
    |  P1_FAULT   P1_SEL     P1_SELECT   |
    
    |──[ M0.2 ]──[ M0.1 ]──( SET Q0.1 )──|
    |  P1_FAULT   P1_SEL_FB  P2_PUMP     |

─── Network 5: Pump 1 Control ───────────────────────────
    |──[ M0.1 ]──[ M0.0 ]──[/M0.2]──( Q0.0 )──|
    |  P1_SEL    SYS_RUN   P1_FAULT   PUMP1    |

─── Network 6: Pump 2 Control ───────────────────────────
    |──[/M0.1]──[ M0.0 ]──[/M0.3]──( Q0.1 )──|
    |  P2_SEL    SYS_RUN   P2_FAULT   PUMP2    |

─── Network 7: Level Protection ────────────────────────
    |──[ I0.5 ]──( RST M0.0 )──|
    |  HIGH_LEVEL  SYS_RUN     |

─── Network 8: Auto Start on Low Level ─────────────────
    |──[ I0.4 ]──[ M0.0 ]──( SET M0.0 )──|
    |  LOW_LEVEL   SYS_OFF    START_SYS  |
```

---

## Step 44: Logic ใน Manufacturing

### Quality Check Station
```
QUALITY CHECK STATION LOGIC:

System:
  - วัสดุมาบน Conveyor
  - ตรวจสอบ 3 Sensor (Height, Width, Weight)
  - Pass: ส่งต่อ
  - Fail: เบี่ยงออก Reject

I/O:
  I0.0 = PART_PRESENT    Sensor ตรวจพบชิ้นงาน
  I0.1 = HEIGHT_OK       ความสูงถูกต้อง
  I0.2 = WIDTH_OK        ความกว้างถูกต้อง
  I0.3 = WEIGHT_OK       น้ำหนักถูกต้อง
  Q0.0 = PASS_GATE       ประตูผ่าน
  Q0.1 = REJECT_GATE     ประตู Reject
  Q0.2 = PASS_COUNTER    นับชิ้นงานดี
  Q0.3 = REJECT_COUNTER  นับชิ้นงานเสีย
  Q0.4 = PASS_LAMP       ไฟสีเขียว
  Q0.5 = REJECT_LAMP     ไฟสีแดง
  M0.0 = PART_PASS       ชิ้นงานผ่าน
  M0.1 = PART_REJECT     ชิ้นงานไม่ผ่าน

─── Network 1: Part Present Detection (Edge) ───────────
    |──[ I0.0 ]──[P]────────────────( M1.0 )──|
    |  PART_SEN  RISE_EDGE            PART_IN  |

─── Network 2: Quality Check Logic ─────────────────────
  Comment: Pass = ALL sensors OK when part present
  
    |──[ M1.0 ]──[ I0.1 ]──[ I0.2 ]──[ I0.3]──(SET M0.0)──|
    |  PART_IN   HEIGHT    WIDTH     WEIGHT      PASS       |

─── Network 3: Reject Logic ─────────────────────────────
  Comment: Reject = any sensor fail when part present
  
    |──[ M1.0 ]──[/M0.0]──────────────(SET M0.1)──|
    |  PART_IN   NOT_PASS                REJECT    |

─── Network 4: Clear Flags ──────────────────────────────
    |──[/I0.0]──( RST M0.0 )──|
    |  NO_PART   PASS         |
    
    |──[/I0.0]──( RST M0.1 )──|
    |  NO_PART   REJECT       |

─── Network 5: Gate Control ─────────────────────────────
    |──[ M0.0 ]──────────────────( Q0.0 )──|  Pass gate
    |──[ M0.1 ]──────────────────( Q0.1 )──|  Reject gate

─── Network 6: Count ────────────────────────────────────
    |──[ M0.0 ]──[P]──────────( Q0.2 )──|  Count pass
    |──[ M0.1 ]──[P]──────────( Q0.3 )──|  Count reject

─── Network 7: Indicators ───────────────────────────────
    |──[ M0.0 ]──────────────────( Q0.4 )──|  Green
    |──[ M0.1 ]──────────────────( Q0.5 )──|  Red
```

---

## Step 45: Combinatorial vs Sequential Logic

```
COMBINATORIAL LOGIC:
  Output ขึ้นอยู่กับ Input ปัจจุบัน เท่านั้น
  ไม่มี Memory ของสถานะก่อนหน้า
  
  ตัวอย่าง: 
  |──[ A ]──[ B ]──( Y )──|
  Y = A AND B (ขึ้นอยู่กับ A, B ณ ปัจจุบัน)


SEQUENTIAL LOGIC:
  Output ขึ้นอยู่กับ Input ปัจจุบัน AND สถานะก่อนหน้า
  มี Memory (Set/Reset, Timer, Counter)
  
  ตัวอย่าง:
  |──[ A ]──(SET Y)──|    ← Y จำค่าไว้
  |──[ B ]──(RST Y)──|


ใน PLC ส่วนใหญ่เป็น Sequential Logic:
  - Motor Start/Stop (จำสถานะ)
  - Counter (จำจำนวน)
  - Timer (จำเวลา)
```

---

## Step 46: Practice - อุตสาหกรรมจริง

### Level Control in Tank
```
TANK LEVEL CONTROL:

System:
  ┌──────────────────────────────────┐
  │           Tank                   │
  │  LS_HH ─── ─── ─── (100%)       │
  │  LS_H  ─── ─── ─── (80%)        │
  │  LS_L  ─── ─── ─── (20%)        │
  │  LS_LL ─── ─── ─── (5%)         │
  │                                  │
  │  ← Inlet Valve (FV-101)          │
  │                    → Outlet Pump │
  └──────────────────────────────────┘

Logic:
  - Level < 20% → Open Inlet Valve (Fill)
  - Level > 80% → Close Inlet Valve
  - Level < 5% → Emergency! Stop Outlet Pump
  - Level > 100% → Overflow Alarm

Hysteresis (Deadband) Logic:
  เพื่อป้องกัน Valve เปิด-ปิดบ่อย (Chatter)
  
    |──[ LS_L  ]──────────────(SET M0.0)──|  Level Low → Open Valve
    |──[ LS_H  ]──────────────(RST M0.0)──|  Level High → Close Valve
    |──[ M0.0  ]──────────────( FV_101 )──|  Valve Control
    
    |──[ LS_LL ]──────────────(RST PUMP_RUN)──|  Emergency Stop Pump
    |──[ LS_HH ]──────────────(SET ALARM)─────|  Overflow Alarm
```

---

## Step 47: Boolean ถึง Ladder Systematically

### วิธีแปลง SOP (Sum of Products) → Ladder
```
SYSTEMATIC CONVERSION:

Problem:
Q = A'BC + AB'C + ABC' + ABC

Step 1: วาด Truth Table
┌───┬───┬───┬───┐
│ A │ B │ C │ Q │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
│ 0 │ 0 │ 1 │ 0 │
│ 0 │ 1 │ 0 │ 0 │
│ 0 │ 1 │ 1 │ 1 │ A'BC
│ 1 │ 0 │ 0 │ 0 │
│ 1 │ 0 │ 1 │ 1 │ AB'C
│ 1 │ 1 │ 0 │ 1 │ ABC'
│ 1 │ 1 │ 1 │ 1 │ ABC
└───┴───┴───┴───┘

Step 2: K-Map
     BC
AB   00  01  11  10
00    0   0   1   0
01    0   0   0   0  (หมายเหตุ: ไม่มี minterm ที่ A=0,B=1,C=0)
11    0   1   1   1
10    0   1   0   0

Wait, let me redo:
AB=00→A=0,B=0; AB=01→A=0,B=1; AB=11→A=1,B=1; AB=10→A=1,B=0
BC=00→B=0,C=0; BC=01→B=0,C=1; BC=11→B=1,C=1; BC=10→B=1,C=0

     BC
AB   00  01  11  10
00    0   0   0   0
01    0   0   1   0   m3 (A=0,B=1,C=1)
11    0   1   1   1   m5=A=1,B=0,C=1? No...

ขอ redo ใหม่ตรงๆ กว่า:
minterm ที่ Q=1: m3(011), m5(101), m6(110), m7(111)

K-Map 3 variables (A, BC):
     BC
A    00  01  11  10
0     0   0   1   0     m3=A'BC
1     0   1   1   1     m5=AB'C, m7=ABC, m6=ABC'

Groups:
  {m5,m7}: B=0→? A=1, C=1 → AC
  Wait no: m5=101 (A=1,B=0,C=1), m7=111 (A=1,B=1,C=1)
  m5 and m7 → A=1, C=1, B varies → AC

  {m6,m7}: m6=110 (A=1,B=1,C=0), m7=111 (A=1,B=1,C=1)
  → A=1, B=1, C varies → AB

  {m3,m7}: m3=011, m7=111
  → B=1, C=1, A varies → BC

So Q = AC + AB + BC (2-out-of-3 voting!)

Step 3: Ladder
    |──[ A ]──[ C ]──┐       |
    |                ├──( Q )─|
    |──[ A ]──[ B ]──┤       |
    |                │       |
    |──[ B ]──[ C ]──┘       |
```

---

## Step 48: ตัวอย่างโปรแกรมจาก Industry Standard

### 7-Segment Display Control
```
7-SEGMENT DISPLAY CONTROL:

Segments:
       _
      |_|  ← a=top, b=upper-right, c=lower-right
      |_|    d=bottom, e=lower-left, f=upper-left, g=middle

     aaa
    f   b
    f   b
     ggg
    e   c
    e   c
     ddd

Digit → Segments:
0: a,b,c,d,e,f   = 1111110
1: b,c           = 0110000
2: a,b,d,e,g     = 1101101
3: a,b,c,d,g     = 1111001
4: b,c,f,g       = 0110011
5: a,c,d,f,g     = 1011011
6: a,c,d,e,f,g   = 1011111
7: a,b,c         = 1110000
8: a,b,c,d,e,f,g = 1111111
9: a,b,c,d,f,g   = 1111011

BCD Inputs: D3,D2,D1,D0 (4 bits)
Outputs: a,b,c,d,e,f,g (7 bits)

Segment 'a' = D3'D2'D1'D0' + D3'D2'D1D0' + 
              D3'D2D1'D0'  + D3'D2D1D0  + ...

(Complex Logic - ใน PLC จริงใช้ Lookup Table แทน)
```

---

## Step 49: Anti-Tie Down สำหรับ Press Machine

```
ANTI-TIE DOWN CIRCUIT:
  ป้องกัน Operator วาง/มัดปุ่มไว้โดยไม่ตั้งใจ

Requirements:
  1. ต้องกด 2 ปุ่มพร้อมกันภายใน 0.5 วินาที
  2. ถ้ากดปุ่มใดปุ่มหนึ่งค้างไว้ → ไม่ทำงาน
  3. ต้องปล่อยปุ่มทั้งคู่ก่อน กดใหม่

I/O:
  I0.0 = L_HAND_PB    Left Hand Button
  I0.1 = R_HAND_PB    Right Hand Button
  Q0.0 = PRESS_DOWN   Press Solenoid

LADDER (Simplified):

─── Network 1: Left Hand Edge ──────────────────────────
    |──[ I0.0 ]──[P]──────────────(SET M0.0)──|
    |  L_HAND    RISE               L_TRIGGER  |
    
    Timer clear M0.0 after 0.5s

─── Network 2: Right Hand Edge ─────────────────────────
    |──[ I0.1 ]──[P]──────────────(SET M0.1)──|
    |  R_HAND    RISE               R_TRIGGER  |
    
    Timer clear M0.1 after 0.5s

─── Network 3: Anti-Tie Down Check ─────────────────────
    |──[ M0.0 ]──[ M0.1 ]──────────(SET M0.2)──|
    |  L_TRIG    R_TRIG              APPROVED   |

─── Network 4: Press Output ─────────────────────────────
    |──[ M0.2 ]──[ I0.0 ]──[ I0.1 ]──( Q0.0 )──|
    |  APPROVED  L_HAND    R_HAND     PRESS_DN   |

─── Network 5: Reset on Release ─────────────────────────
    |──[/I0.0 ]──────────────────(RST M0.0)──|
    |  L_RELEASE                   L_TRIGGER  |

    |──[/I0.1 ]──────────────────(RST M0.1)──|
    |  R_RELEASE                   R_TRIGGER  |
    
    |──[/I0.0 ]──────────────────(RST M0.2)──|
    |──[/I0.1 ]──────────────────(RST M0.2)──|
```

---

## Step 50: สรุป Logic Gates และ Project

### Lab Project 03: Bottle Filling Machine
```
PROJECT: BOTTLE FILLING MACHINE

System Description:
  ┌───────────────────────────────────────────────────────┐
  │                                                       │
  │  [IN_CONV] → [FILL_STATION] → [CAP_STATION] → [OUT] │
  │                                                       │
  └───────────────────────────────────────────────────────┘

I/O List:
  I0.0 = BOTTLE_SENSOR    ขวดอยู่ที่ Fill Station
  I0.1 = FILL_LEVEL_OK    ของเต็มแล้ว (Float Sensor)
  I0.2 = CAP_SENSOR       ขวดอยู่ที่ Cap Station
  I0.3 = CAP_OK           ปิดฝาแล้ว (Proximity)
  I0.4 = START_PB         ปุ่มเริ่ม
  I0.5 = STOP_PB          ปุ่มหยุด (NC)
  I0.6 = E_STOP           Emergency (NC)
  
  Q0.0 = IN_CONV_MTR      Conveyor เข้า
  Q0.1 = OUT_CONV_MTR     Conveyor ออก
  Q0.2 = FILL_VALVE       วาล์วเติม
  Q0.3 = CAP_CYLINDER     กระบอก Cylinder ปิดฝา
  Q0.4 = FILL_LAMP        ไฟแสดง Fill
  Q0.5 = CAP_LAMP         ไฟแสดง Cap

Requirements:
  1. กด Start → ระบบเริ่มทำงาน
  2. ขวดเข้า Fill Station → หยุด Conveyor, เปิดวาล์ว
  3. ของเต็ม → ปิดวาล์ว, เดิน Conveyor ต่อ
  4. ขวดเข้า Cap Station → หยุด Conveyor, ปิดฝา
  5. ปิดฝาแล้ว → เดิน Conveyor ต่อ
  6. E-Stop → หยุดทุกอย่างทันที

ให้ผู้เรียนเขียน Ladder Program เอง
(เฉลยอยู่ใน Part 05)
```

---

*[← Part 02: Contact และ Coil] [→ Part 04: Timer Instructions]*
