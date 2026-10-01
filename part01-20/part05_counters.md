# Part 05: Counter Instructions
## Steps 71-90: CTU, CTD, CTUD และการประยุกต์ใช้งาน

---

## Step 71: Counter คืออะไร?

```
COUNTER FUNDAMENTALS:

Counter คือ Instruction ที่นับ Event ใน PLC

ประเภท Counter:
  CTU  (Count Up)         : นับขึ้น
  CTD  (Count Down)       : นับลง
  CTUD (Count Up/Down)    : นับขึ้น/ลง

โครงสร้าง Counter:
  ┌─────────────────────────────────────────────────────┐
  │  CTU Counter                                        │
  │  ──────────────────────────────────────────         │
  │  CU  (Count Up Input): Rising Edge นับ +1           │
  │  R   (Reset Input):    Reset CV = 0                 │
  │  PV  (Preset Value):   ค่าที่ต้องการนับถึง           │
  │  Q   (Output):         ON เมื่อ CV ≥ PV             │
  │  CV  (Current Value):  ค่าปัจจุบัน                  │
  └─────────────────────────────────────────────────────┘

หน่วยของ PV/CV:
  - ไม่มีหน่วย (จำนวนครั้ง)
  - ขอบเขต: 0 ถึง 32767 (INT) หรือ 0 ถึง 2147483647 (DINT)
```

---

## Step 72: CTU (Count Up Counter)

```
CTU - COUNT UP COUNTER

การทำงาน:
  CU: 0→1 Rising Edge → CV = CV + 1
  R=1: CV = 0 (Reset ทุกอย่าง)
  Q: เมื่อ CV ≥ PV → Q = 1

Timeline (PV = 5):
  CU:   0 0↑0 0↑0 0↑0 0↑0 0↑0 0 0 0 R 0
  CV:   0 0 1 1 2 2 3 3 4 4 5 5 5 5 5 0 0
  Q:    0 0 0 0 0 0 0 0 0 0 1 1 1 1 1 0 0
                                         ↑ Reset

หมายเหตุสำคัญ:
  - CU ต้องเป็น Rising Edge เท่านั้น → ใช้ [P] หรือ Edge ของ Contact
  - ถ้า CU ค้าง = 1 → ไม่นับเพิ่ม!
  - CV อาจ Overflow ถ้าเกิน Max Value
  - Q ค้างเป็น 1 แม้หลัง CV = PV จนกว่า R=1

LADDER:
    |──[ I0.0 ]──[P]──────────────────────────|
    |             ├─[CTU C0 PV=5]──────────────|
    |──[ I0.1 ]──┘                            |
    |  RESET                                   |
    
    |──[ C0.Q ]──────────────────────( Q0.0 )──|
    |  COUNT_DONE                     OUTPUT   |
```

### CTU ตัวอย่างใช้งาน
```
EXAMPLE: COUNT PRODUCTS ON CONVEYOR

โจทย์: นับชิ้นงาน 10 ชิ้น → หยุด Conveyor, แจ้งเตือน

    ─── Rung 1: Count Parts ─────────────────────────────
    |──[ I0.0 ]──[P]──[CTU C0 PV=10]───────────────────|
    |  PART_SENSOR  EDGE  Part Counter                  |

    ─── Rung 2: Reset Counter ───────────────────────────
    |──[ I0.1 ]──[P]──(RST C0)────────────────────────|
    |  RESET_PB   EDGE                                  |

    ─── Rung 3: Conveyor Run ────────────────────────────
    |──[ M0.0 ]──[/C0.Q]──────────────( Q0.0 )──────────|
    |  SYS_RUN   NOT_FULL  CONVEYOR                     |

    ─── Rung 4: Full Alarm ───────────────────────────────
    |──[ C0.Q ]──────────────────────( Q0.1 )────────────|
    |  COUNT_DONE                     FULL_LAMP          |
    
    ─── Rung 5: Display Count (to HMI) ──────────────────
    |──[MOVE C0.CV → MW10]───────────────────────────────|
    |  Copy current count to HMI display                 |
```

---

## Step 73: CTD (Count Down Counter)

```
CTD - COUNT DOWN COUNTER

การทำงาน:
  CD: 0→1 Rising Edge → CV = CV - 1
  LD: 1 → Load CV = PV (Load Preset)
  Q: เมื่อ CV ≤ 0 → Q = 1

Timeline (PV = 5, Load then count down):
  CD:  0 0↑0 0↑0 0↑0 0↑0 0↑0 0
  LD:  1 0 0 0 0 0 0 0 0 0 0 0 0  ← Load at start
  CV:  5 5 4 4 3 3 2 2 1 1 0 0 0
  Q:   0 0 0 0 0 0 0 0 0 0 1 1 1

ใช้สำหรับ:
  - นับถอยหลังจากเต็มสู่ศูนย์
  - Batch Production (วัตถุดิบเหลือ X ชิ้น)
  - Remaining Cycle Count
```

### CTD ตัวอย่าง
```
EXAMPLE: REMAINING BATCH COUNT

โจทย์: เริ่มต้นมี 100 ชิ้น → นับลงทุกครั้งที่ใช้ไป
       เหลือ 0 → แจ้งเติม

    ─── Rung 1: Load Batch Quantity ─────────────────────
    |──[ I0.0 ]──[P]──[CTD C1 PV=100]──────────────────|
    |  LOAD_PB   EDGE  Load 100 parts to CV             |
    
    Actually CTD LOAD:
    |──[ I0.0 ]──────────[CTD C1]──────────────────────|
    |  LOAD_PB  ──────── LD=I0.0  ← Load PV=100 into CV|
    
    ─── Rung 2: Count Down per Use ──────────────────────
    |──[ I0.1 ]──[P]──────────────────────────────────|
    |  USE_SENSOR EDGE ── CD=I0.1  ← Count Down 1      |
    
    ─── Rung 3: Refill Alarm ────────────────────────────
    |──[ C1.Q ]──────────────────────(REFILL_ALARM)──────|
    |  COUNT_ZERO                                        |

Note: CTD ใน IEC Standard ใช้ LD (Load) ไม่ใช่ R (Reset)
      เพราะ Load CV = PV ไม่ใช่ Reset CV = 0
```

---

## Step 74: CTUD (Count Up / Down Counter)

```
CTUD - COUNT UP/DOWN COUNTER

โครงสร้าง:
  CU  : Count Up Input (Rising Edge +1)
  CD  : Count Down Input (Rising Edge -1)
  R   : Reset (CV = 0)
  LD  : Load (CV = PV)
  PV  : Preset Value
  QU  : Output Up (CV ≥ PV)
  QD  : Output Down (CV ≤ 0)
  CV  : Current Value

Timeline (PV = 5):
  CU:  ↑   ↑   ↑   ↑   ↑
  CD:          ↑   ↑
  CV:  0 1 2 3 4 3 4 5 5 5
  QU:  0 0 0 0 0 0 0 1 1 1
  QD:  1 0 0 0 0 0 0 0 0 0  ← QD=1 เฉพาะ CV≤0

ใช้สำหรับ:
  - ตรวจสอบจำนวนคนในห้อง (เข้า +1, ออก -1)
  - Stock Management (รับ +1, จ่าย -1)
  - Buffer Level Monitoring
```

### CTUD People Counter
```
EXAMPLE: ROOM OCCUPANCY COUNTER

โจทย์: นับคนเข้า/ออกห้อง
       - Door Entry Sensor → +1
       - Door Exit Sensor → -1
       - ห้องเต็ม (PV=20) → Lock Entry Door
       - ห้องว่าง (CV=0) → Turn off AC

I/O:
  I0.0 = ENTRY_SENSOR   (IR Sensor ทางเข้า)
  I0.1 = EXIT_SENSOR    (IR Sensor ทางออก)
  Q0.0 = ENTRY_LOCK     (ล็อคประตูเข้า)
  Q0.1 = AC_SYSTEM      (เปิดแอร์)
  Q0.2 = FULL_LAMP      (ไฟ FULL)
  Q0.3 = EMPTY_LAMP     (ไฟว่าง)

    ─── Rung 1: People Counter ──────────────────────────
    CTUD Block:
    CU = I0.0_EDGE (Entry)
    CD = I0.1_EDGE (Exit)
    R  = I0.2 (Reset when everyone out)
    PV = 20
    QU → M0.0 (Room Full)
    QD → M0.1 (Room Empty)
    CV → MW100 (Display Count)

    ─── Rung 2: Entry Lock ───────────────────────────────
    |──[ M0.0 ]──────────────────────( Q0.0 )────────────|
    |  ROOM_FULL                      ENTRY_LOCK         |

    ─── Rung 3: AC Control ───────────────────────────────
    |──[/M0.1]──────────────────────( Q0.1 )─────────────|
    |  NOT_EMPTY                      AC_SYSTEM          |

    ─── Rung 4: Lamps ────────────────────────────────────
    |──[ M0.0 ]──────────────────────( Q0.2 )────────────|
    |──[ M0.1 ]──────────────────────( Q0.3 )────────────|
```

---

## Step 75: Counter Cascade (นับจำนวนมาก)

```
CASCADED COUNTERS - HIGH COUNT:

INT Counter สูงสุด: 32767
ถ้าต้องนับ 100,000 ชิ้น → ต้องใช้ 2 Counter

วิธีที่ 1: Cascade Counters
    ─── Rung 1: Low Counter (0-999) ─────────────────────
    |──[ I0.0 ]──[P]──[CTU C_LOW PV=1000]──────────────|
    |  PULSE                                             |

    ─── Rung 2: High Counter (นับ ×1000) ────────────────
    |──[ C_LOW.Q ]──[P]──[CTU C_HIGH PV=100]───────────|
    |  EVERY_1000   EDGE   × 1000                       |

    ─── Rung 3: Reset Low Counter ───────────────────────
    |──[ C_LOW.Q ]──[P]──(RST C_LOW)───────────────────|

    ─── Rung 4: Total Count Display ─────────────────────
    ผลลัพธ์: C_HIGH.CV × 1000 + C_LOW.CV = Total Count
    สูงสุด: 100 × 1000 + 999 = 100,999 ชิ้น!

วิธีที่ 2: DINT Counter (32-bit)
    ใช้ DINT Counter ใน Siemens/CODESYS:
    CV : DINT (สูงสุด 2,147,483,647)
    PV : DINT
    ไม่ต้อง Cascade!
```

---

## Step 76: Counter กับ Shift Register

```
SHIFT REGISTER WITH COUNTER:

โจทย์: Conveyor Belt มี 10 Station
       เมื่อ Part ผ่าน Station 0 → ติดตาม Part จนถึง Station 10

    ─── Concept ─────────────────────────────────────────
    Part Position Tracking:
    M0.0 = Part at Station 1
    M0.1 = Part at Station 2
    ...
    M0.9 = Part at Station 10
    
    ทุก Pulse ของ Conveyor Index → Shift ข้อมูลทีละ Bit
    
    ─── Rung 1: Index Pulse ─────────────────────────────
    |──[ I0.1 ]──[P]─────────────────( M1.0 )────────────|
    |  INDEX_PULSE  EDGE              SHIFT_TRIG          |
    
    ─── Rung 2: Shift Register ──────────────────────────
    |──[ M1.0 ]──────[SHL MW10 1 → MW10]─────────────────|
    |  SHIFT_TRIG     Shift Word Left 1 bit               |
    
    ─── Rung 3: Load New Part ───────────────────────────
    |──[ I0.0 ]──[P]──[SET_BIT MW10 Bit0]────────────────|
    |  PART_IN   EDGE  Set bit 0 = Part Present          |
    
    ─── Rung 4: Check Station 5 ─────────────────────────
    |──[GET_BIT MW10 Bit5]──────────( Q0.0 )─────────────|
    |  Part at Station 5             Action at Stn5      |
```

---

## Step 77: Counter ใน Batch Production

```
BATCH PRODUCTION CONTROL:

โจทย์: ระบบ Filling สำหรับขนม
       แต่ละถุงบรรจุขนม 20 ชิ้น
       ผลิตวันละ 500 ถุง
       นับชิ้นขนมและถุง

I/O:
  I0.0 = CANDY_SENSOR    Sensor ตรวจจับขนม
  I0.1 = BAG_SENSOR      Sensor ตรวจจับถุง
  I0.2 = START_PB        ปุ่มเริ่ม
  I0.3 = RESET_PB        ปุ่ม Reset
  Q0.0 = FILL_CONV       Conveyor ป้อนขนม
  Q0.1 = SEAL_MACHINE    เครื่องปิดถุง
  Q0.2 = REJECT_GATE     ประตู Reject
  Q0.3 = DONE_LAMP       ไฟเสร็จ

    ─── Rung 1: Candy Count Per Bag ─────────────────────
    |──[ I0.0 ]──[P]──[CTU C_CANDY PV=20]──────────────|
    |  CANDY_SEN  EDGE  20 candies per bag              |

    ─── Rung 2: Seal Bag when Full ─────────────────────
    |──[ C_CANDY.Q ]──────────────────( Q0.1 )──────────|
    |  20_CANDIES                      SEAL             |

    ─── Rung 3: Count Bags ──────────────────────────────
    |──[ C_CANDY.Q ]──[P]──[CTU C_BAG PV=500]──────────|
    |  BAG_SEALED    EDGE   500 bags per day             |

    ─── Rung 4: Reset Candy Counter ─────────────────────
    |──[ C_CANDY.Q ]──[P]──(RST C_CANDY)───────────────|
    |  BAG_SEALED    EDGE                                |

    ─── Rung 5: Daily Target Reached ────────────────────
    |──[ C_BAG.Q ]────────────────────( Q0.3 )──────────|
    |  500_BAGS                        DONE_LAMP        |
    
    |──[ C_BAG.Q ]────────────────────(RST M0.0)────────|
    |  500_BAGS                        SYS_RUN          |

    ─── Rung 6: Reset All ───────────────────────────────
    |──[ I0.3 ]──[P]──(RST C_CANDY)──────────────────|
    |  RESET_PB  EDGE                                   |
    
    |──[ I0.3 ]──[P]──(RST C_BAG)────────────────────|
```

---

## Step 78: Position Counting ด้วย Encoder

```
ENCODER POSITION TRACKING:

Incremental Encoder:
  - ส่ง Pulse ทุกๆ การหมุน 1 Step
  - 1000 PPR = 1000 Pulses per Revolution
  - ใช้นับตำแหน่ง

Counter Configuration:
  C_POS: นับ Encoder Pulses
  PV = 5000 (หมายถึง 5 รอบถ้า 1000PPR)

  ─── Forward Motion ──────────────────────────────────
  |──[ ENC_A ]──[P]──[CTU C_POS PV=10000]────────────|
  |  ENCODER_A  EDGE   Position Counter                |

  ─── Check Target Position ──────────────────────────
  |──[C_POS.CV = TARGET_POS]──────────( AT_TARGET )──|
  |  Position Reached                                  |

  ─── Home Position ───────────────────────────────────
  |──[ HOME_SW ]──(RST C_POS)────────────────────────|
  |  Home Switch   Reset Position                     |

  ─── Display Position ────────────────────────────────
  |──[MOVE C_POS.CV → DISPLAY_MW]─────────────────────|
```

---

## Step 79: Counter ใน Quality Control

```
QUALITY CONTROL COUNTER SYSTEM:

โจทย์: ระบบ QC สายการผลิต
       - นับชิ้นงานดี (PASS)
       - นับชิ้นงานเสีย (REJECT)
       - คำนวณ % ของเสีย
       - ถ้าของเสีย > 5% → หยุดสาย

    ─── Rung 1: Total Parts ─────────────────────────────
    |──[ I0.0 ]──[P]──[CTU C_TOTAL PV=999]─────────────|
    |  PART_DETECT EDGE   Total Parts Counter            |

    ─── Rung 2: Reject Parts ────────────────────────────
    |──[ I0.1 ]──[P]──[CTU C_REJ PV=999]───────────────|
    |  REJECT_SEN  EDGE   Reject Counter                 |

    ─── Rung 3: Calculate Reject % ──────────────────────
    Structured Text (in PLC that supports):
    REJECT_PCT := (C_REJ.CV * 100) / C_TOTAL.CV;
    
    หรือใน Ladder:
    |──[MUL C_REJ.CV × 100 → MD_TEMP]──────────────────|
    |──[DIV MD_TEMP ÷ C_TOTAL.CV → MD_REJECT_PCT]──────|

    ─── Rung 4: Quality Alarm ───────────────────────────
    |──[CMP MD_REJECT_PCT > 5]──(SET QUALITY_ALARM)────|
    
    ─── Rung 5: Stop Line on High Reject ────────────────
    |──[ QUALITY_ALARM ]──────────────(RST SYS_RUN)────|

    ─── Rung 6: Shift Reset ─────────────────────────────
    |──[ SHIFT_END ]──(RST C_TOTAL)──────────────────|
    |──[ SHIFT_END ]──(RST C_REJ)────────────────────|
```

---

## Step 80: Counter Address ใน PLC ต่างยี่ห้อ

```
COUNTER ADDRESS BY PLC BRAND:

SIEMENS S7-1200/1500 (TIA Portal):
─────────────────────────────────────────────────────────
  Counter เป็น Function Block ใน Data Block:
  DB1.myCTU.CU     = Count Up Input (BOOL)
  DB1.myCTU.R      = Reset Input (BOOL)
  DB1.myCTU.PV     = Preset Value (INT)
  DB1.myCTU.Q      = Done Output (BOOL)
  DB1.myCTU.CV     = Current Value (INT)
  
  ประเภท: CTU_INT, CTU_DINT, CTU_UDINT, CTU_LINT
  
  ตัวอย่าง:
  DB1.myCTU(CU:=I0.0, R:=I0.1, PV:=100);
  Q0.0 := DB1.myCTU.Q;
  MW10 := DB1.myCTU.CV;

ALLEN-BRADLEY (Studio 5000):
─────────────────────────────────────────────────────────
  C5:0.PRE  = Preset Value
  C5:0.ACC  = Accumulated Value (Current Count)
  C5:0/CU   = Count Up Enable
  C5:0/CD   = Count Down Enable
  C5:0/DN   = Done Bit (ACC ≥ PRE)
  C5:0/OV   = Overflow Bit
  C5:0/UN   = Underflow Bit
  
  ตัวอย่าง:
  |──[ I:0/0 ]──[CTU Counter:C5:0 Preset:100 Accum:0]──|
  |──[ C5:0/DN ]──────────────────────────( O:0/0 )────|

MITSUBISHI FX3U/FX5U:
─────────────────────────────────────────────────────────
  C0 - C15:  16-bit General Counter (Preset K1-K32767)
  C100-C199: 32-bit High-Speed Counter
  
  ตัวอย่าง:
  |──[ X0 ]──[OUT C0 K100]──|   (Count to 100)
  |──[ C0 ]──────────(Y0)──|
  |──[ X1 ]──[RST C0]──────|

OMRON CP1H:
─────────────────────────────────────────────────────────
  CNT 0000 - CNT 4095: Down Counter
  #0100 = Preset 100 decimal
  
  ตัวอย่าง:
  |──[ 0.01 ]──[CNT 0001 #0100]──|
  |──[ C0001 ]──────────(100.00)──|
  
  Note: OMRON Counter นับลง (CTD)
  Reset ด้วย CNTR (Reset Input)
```

---

## Step 81: Encoder Counter ความเร็วสูง (High-Speed Counter)

```
HIGH-SPEED COUNTER (HSC):

ปัญหา: PLC Scan Time ช้า (10ms)
       Encoder 1000 RPM × 1000 PPR = 16,667 Hz
       Scan Time 10ms → ต้องรับ 167 Pulses/Scan
       → Software Counter พลาดได้!

วิธีแก้: ใช้ Hardware High-Speed Counter
  - รับ Pulse ด้วย Hardware โดยตรง
  - ไม่ขึ้นกับ Scan Time
  - ความถี่สูงถึง 100kHz - 1MHz

SIEMENS S7-1200 HSC:
─────────────────────────────────────────────────────────
  6 HSC (High-Speed Counter) built-in
  Input: I0.0 (HSC1 Clock A), I0.1 (HSC1 Direction/B)
  Max Frequency: 100kHz (Standard), 1MHz (High-speed)
  
  Configuration (TIA Portal):
  1. ไปที่ Properties → High Speed Counters
  2. Enable HSC1
  3. เลือก Mode: Single Phase / AB Quadrature
  4. กำหนด Address: %ID1000 (32-bit DINT)
  
  อ่านค่าใน Ladder:
  |──[MOVE %ID1000 → MD_POSITION]──|
  (ID1000 = HSC1 Current Value)

MITSUBISHI FX5U HSC:
─────────────────────────────────────────────────────────
  X0-X5: High-Speed Counter Input
  C235-C255: 32-bit High-Speed Counters
  
  ตัวอย่าง:
  |──[ X10 ]──[DMOV K0 D0]──|   (Reset Position)
  |──[ X11 ]──[DHSCS K1000 C235 Y0]──|  (Interrupt at 1000)
```

---

## Step 82: Preset Counter Application

```
PRESET COUNTER - VARIABLE SETPOINT:

โจทย์: ตั้งจำนวนชิ้นงานต่อ Batch จาก HMI
       Operator เปลี่ยนได้ระหว่าง Production

    ─── Concept ─────────────────────────────────────────
    PV ของ Counter ถูกกำหนดจาก HMI Setpoint
    Operator พิมพ์ค่า → เก็บใน MW_BATCH_SIZE
    Counter ใช้ MW_BATCH_SIZE เป็น PV
    
    ─── Rung 1: Counter with Variable PV ────────────────
    CTU Block:
    CU = PART_SENSOR_EDGE
    R  = RESET_INPUT
    PV = MW_BATCH_SIZE  ← จาก HMI!
    Q  → BATCH_DONE
    CV → MW_BATCH_COUNT
    
    ─── Rung 2: Write PV from HMI ───────────────────────
    HMI หรือ Operator Panel เขียนค่าไปยัง MW_BATCH_SIZE
    PLC อ่าน MW_BATCH_SIZE เป็น PV ของ Counter
    
    ─── Rung 3: Batch Done ───────────────────────────────
    |──[ BATCH_DONE ]──────────────(STOP_CONVEYOR)────────|
    |──[ BATCH_DONE ]──────────────(ALARM_BUZZER)─────────|
    |──[ BATCH_DONE ]──[P]──(RST CTU)────────────────────|
```

---

## Step 83: Counter Reset Strategies

```
COUNTER RESET METHODS:

1. MANUAL RESET
   Operator กดปุ่ม Reset
   |──[ RESET_PB ]──[P]──(RST COUNTER)──|

2. AUTOMATIC RESET WHEN DONE
   Counter Done → Reset อัตโนมัติ → เริ่มนับใหม่
   |──[ C0.Q ]──[P]──(RST C0)──|

3. SHIFT/PERIODIC RESET
   Reset เมื่อขึ้น Shift ใหม่
   |──[ SHIFT_START ]──(RST C0)──|
   ใช้ Real Time Clock หรือ Timer

4. CONDITIONAL RESET
   Reset เมื่อเงื่อนไขเฉพาะ
   |──[ FAULT ]──(RST C0)──|
   |──[ PROD_END ]──(RST C0)──|

5. POWER-UP RESET
   บาง Application ต้องการ Reset เมื่อเปิด PLC
   |──[ FIRST_SCAN ]──(RST C0)──|
   (First Scan Bit: SM0.1 ใน S7, S:1/15 ใน AB)
```

---

## Step 84: Counter ใน Maintenance Application

```
MAINTENANCE COUNTER APPLICATION:

โจทย์: แจ้งเตือน Maintenance ตามจำนวน Cycle
       - Lubrication ทุก 1000 Cycles
       - Inspection ทุก 5000 Cycles
       - Major Service ทุก 25000 Cycles

    ─── Rung 1: Cycle Counter ───────────────────────────
    |──[ MACHINE_START ]──[P]──[CTU C_CYCLE PV=25001]──|
    |  EACH CYCLE          EDGE   Cycle Counter         |
    
    ─── Rung 2: Lubrication Reminder (every 1000) ───────
    |──[CMP C_CYCLE.CV MOD 1000 = 0]──(LUBE_REMIND)────|
    
    OR ใช้ Sub-Counter:
    |──[ MACHINE_START ]──[P]──[CTU C_LUBE PV=1000]──|
    |──[ C_LUBE.Q ]──────────────(LUBE_REMINDER)──────|
    |──[ C_LUBE.Q ]──[P]──────────────(RST C_LUBE)───|

    ─── Rung 3: Inspection Reminder (every 5000) ────────
    |──[ MACHINE_START ]──[P]──[CTU C_INSP PV=5000]──|
    |──[ C_INSP.Q ]──────────────(INSP_REMINDER)──────|
    |──[ C_INSP.Q ]──[P]──────────────(RST C_INSP)───|
    
    ─── Rung 4: Major Service ────────────────────────────
    |──[ C_CYCLE.Q ]──────────────(MAJOR_SERVICE)─────|
    |──[ C_CYCLE.Q ]──[P]──────────(RST C_CYCLE)──────|
    
    ─── Rung 5: Acknowledge Reminders ───────────────────
    |──[ LUBE_ACK_PB ]──(RST LUBE_REMINDER)─────────|
    |──[ INSP_ACK_PB ]──(RST INSP_REMINDER)─────────|
    
    ─── Rung 6: Display Values ───────────────────────────
    |──[MOVE C_CYCLE.CV → HMI_TOTAL_CYCLE]────────────|
    |──[MOVE C_LUBE.CV  → HMI_SINCE_LUBE]─────────────|
    |──[MOVE C_INSP.CV  → HMI_SINCE_INSP]─────────────|
```

---

## Step 85: Counter กับ Reject Analysis

```
STATISTICAL PROCESS CONTROL (SPC) BASIC:

โจทย์: วิเคราะห์ของเสียแต่ละชั่วโมง
       เปรียบเทียบกับ Target

    ─── System ──────────────────────────────────────────
    C_HOUR_GOOD  : ชิ้นดีในชั่วโมงนี้
    C_HOUR_REJ   : ชิ้นเสียในชั่วโมงนี้
    C_SHIFT_GOOD : ชิ้นดีรวม Shift
    C_SHIFT_REJ  : ชิ้นเสียรวม Shift

    ─── Rung 1-2: Count Good/Reject ─────────────────────
    |──[ GOOD_SENSOR ]──[P]──[CTU C_HOUR_GOOD PV=9999]──|
    |──[ REJ_SENSOR  ]──[P]──[CTU C_HOUR_REJ  PV=9999]──|

    ─── Rung 3: Hourly Reset ─────────────────────────────
    |──[ HOUR_TICK ]──[P]──────────────────────────────|
    |                    Move CV to History before reset|
    |──[MOVE C_HOUR_GOOD.CV → HIST_GOOD[HOUR_IDX]]────|
    |──[MOVE C_HOUR_REJ.CV  → HIST_REJ[HOUR_IDX]]─────|
    |──(RST C_HOUR_GOOD)──(RST C_HOUR_REJ)────────────|
    |──[ADD HOUR_IDX + 1 → HOUR_IDX]──────────────────|

    ─── Rung 4: Hourly Reject % ─────────────────────────
    REJECT_PCT = (C_HOUR_REJ.CV × 100) / 
                 (C_HOUR_GOOD.CV + C_HOUR_REJ.CV)

    ─── Rung 5: Alert if Target Exceeded ────────────────
    |──[CMP REJECT_PCT > TARGET_PCT]──(QUALITY_ALERT)──|
```

---

## Step 86: Counter ใน Material Handling

```
AUTOMATIC STORAGE AND RETRIEVAL (ASAR) BASIC:

โจทย์: Pallet Rack System
       ติดตาม Stock ในแต่ละ Location

    ─── Concept ─────────────────────────────────────────
    15 Locations → 15 Counters (C0 - C14)
    แต่ละ Counter นับ Pallet ในแต่ละ Location
    
    ─── Rung 1: Put Pallet (Location 5) ─────────────────
    |──[ PUT_SENSOR_5 ]──[P]──[CTU C5 PV=5]──|
    |  Location 5 Put   EDGE   Max 5 pallets  |
    
    ─── Rung 2: Get Pallet (Location 5) ─────────────────
    |──[ GET_SENSOR_5 ]──[P]──[CTD C5 PV=5]──|
    |  Location 5 Get   EDGE                  |
    
    OR CTUD:
    CTUD: CU=PUT, CD=GET, PV=5
    QU = LOCATION_5_FULL
    QD = LOCATION_5_EMPTY
    CV = PALLET_COUNT_5
    
    ─── Rung 3: Full Indicator ───────────────────────────
    |──[ C5_QU ]──────────────────(LOC5_FULL_LAMP)────────|
    
    ─── Rung 4: Empty Warning ────────────────────────────
    |──[ C5_QD ]──────────────────(LOC5_EMPTY_ALARM)──────|
```

---

## Step 87: Counter กับ Interrupt (สำหรับ High-Speed)

```
HIGH-SPEED COUNTER WITH INTERRUPT:

ใน Siemens S7-1200 การตั้ง HSC:

1. ใน TIA Portal → Hardware Configuration
   - Properties → High Speed Counters
   - Enable HSC1
   - Function: Count
   - Input: ใช้ I0.0 (Channel A), I0.1 (Direction)
   - Event: Count value reached → Call OB40 (Interrupt)
   - Comparison Value: เปรียบเทียบ CV กับ Threshold

2. สร้าง OB40 (Hardware Interrupt):
   - โปรแกรมใน OB40 จะ Execute ทันทีเมื่อ Count Reached
   - ไม่รอ Scan Cycle!
   
3. ใน OB40:
   Network 1: ทำ Action ที่ต้องการเมื่อถึงตำแหน่ง

จุดเด่น:
  - Response Time: < 100µs
  - ไม่ขึ้นกับ Scan Time
  - เหมาะสำหรับ High-Speed Positioning
```

---

## Step 88: Counter Best Practices

```
COUNTER BEST PRACTICES:

1. ตั้งชื่อให้ชัดเจน
   ✅ C_PART_BATCH, C_HOUR_REJECT, C_MAINTENANCE_CYCLE
   ❌ C0, C1, C2

2. ระวัง Overflow
   - INT Counter: Max 32767
   - ถ้านับเกิน PV → CV ไม่ Reset อัตโนมัติ!
   - ต้อง Reset โดยโปรแกรม
   - สำหรับ Long-term → ใช้ DINT Counter

3. ใช้ Edge Detect (P Contact) สำหรับ CU Input
   ❌ ผิด: |──[ I0.0 ]──[CTU]──|  (นับทุก Scan ที่ I0.0=1!)
   ✅ ถูก: |──[ I0.0 ]──[P]──[CTU]──|  (นับเฉพาะ Rising Edge)

4. เก็บ Counter Value ก่อน Reset (ถ้าต้องการ)
   |──[ RESET_TRIGGER ]──[MOVE C0.CV → ARCHIVE_MW]──|
   |──[ RESET_TRIGGER ]──(RST C0)────────────────────|

5. Retentive Counter สำหรับ Running Totals
   ใช้ Memory ที่เป็น Retentive (Non-volatile)
   เพื่อให้ค่าไม่หายเมื่อ PLC ดับ

6. สำหรับ Production Reports
   บันทึก Count ทุก Shift/Day เข้า Data Array
   เพื่อ Historical Analysis
```

---

## Step 89: Counter กับ Timer รวมกัน

```
COMBINED COUNTER AND TIMER:

โจทย์: ผลิต 100 ชิ้น โดยแต่ละชิ้นใช้เวลาไม่เกิน 30 วินาที
       ถ้าชิ้นไหนเกิน 30 วินาที → แจ้งเตือนช้า + นับชิ้นช้า

    ─── Rung 1: Part Start ───────────────────────────────
    |──[ I0.0 ]──[P]────────────(SET M0.0)──────────────|
    |  PART_START  EDGE          PART_PROCESSING        |

    ─── Rung 2: Part Timer ───────────────────────────────
    |──[ M0.0 ]──[TON T_PART PT=30s]────────────────────|
    |  PART_PROC   Part Processing Timer                 |

    ─── Rung 3: Part Done ────────────────────────────────
    |──[ I0.1 ]──[P]────────────(RST M0.0)──────────────|
    |  PART_DONE  EDGE           PART_PROCESSING        |
    
    |──[ I0.1 ]──[P]────────────[CTU C_PART PV=100]─────|
    |  PART_DONE  EDGE            Count Total Parts      |

    ─── Rung 4: Slow Part ────────────────────────────────
    |──[ T_PART.Q ]──[ M0.0 ]──────────────(SLOW_ALARM)─|
    |  30s EXCEEDED   PART_PROC              Slow Warning|
    
    |──[ T_PART.Q ]──[ M0.0 ]──[P]──[CTU C_SLOW PV=999]|
    |  30s EXCEEDED   PART_PROC   EDGE   Count Slow Parts|

    ─── Rung 5: Batch Done ───────────────────────────────
    |──[ C_PART.Q ]────────────────────(BATCH_COMPLETE)─|
    
    ─── Rung 6: Reset Timer on Part Done ────────────────
    |──[ I0.1 ]──[P]────────────(RST T_PART)────────────|
```

---

## Step 90: Project Part 05 - Injection Molding Machine

```
PROJECT: INJECTION MOLDING MACHINE CYCLE COUNTER

System:
  ┌─────────────────────────────────────────────────────┐
  │  INJECTION MOLDING MACHINE                          │
  │                                                     │
  │  [CLAMP] → [INJECT] → [COOL] → [OPEN] → [EJECT]   │
  │     ↑                                    ↓          │
  │  [CLOSE]                              [PART OUT]    │
  └─────────────────────────────────────────────────────┘

I/O:
  I0.0 = START_PB           ปุ่มเริ่ม
  I0.1 = STOP_PB            ปุ่มหยุด (NC)
  I0.2 = MOLD_CLOSE_LS      Mold ปิดสุด
  I0.3 = MOLD_OPEN_LS       Mold เปิดสุด
  I0.4 = INJECT_DONE        การฉีดเสร็จ
  I0.5 = PART_EJECT_SENSOR  ชิ้นงานหลุดออก
  I0.6 = EMERGENCY_STOP     E-Stop (NC)
  
  Q0.0 = MOLD_CLOSE         ปิด Mold
  Q0.1 = MOLD_OPEN          เปิด Mold
  Q0.2 = INJECT_CYLINDER    กระบอกฉีด
  Q0.3 = EJECT_CYLINDER     กระบอก Eject
  Q0.4 = COOLING_WATER      น้ำหล่อเย็น
  Q0.5 = CYCLE_LED          ไฟ Cycle Running
  
Timers:
  T_INJECT  = เวลาฉีด 3 วินาที
  T_COOL    = เวลาเย็น 15 วินาที
  T_EJECT   = เวลา Eject 2 วินาที

Counters:
  C_CYCLE   = นับรอบการผลิตทั้งหมด (PV=9999999)
  C_SHIFT   = นับรอบในแต่ละ Shift (PV=9999, Reset ทุก Shift)
  C_MAINT   = นับรอบสำหรับ Maintenance (PV=50000)

Requirements:
  1. Auto Cycle ทำงานต่อเนื่อง
  2. นับทุกรอบที่ผลิตสำเร็จ
  3. แจ้ง Maintenance ทุก 50,000 รอบ
  4. แสดง Cycle Count บน HMI
  5. E-Stop หยุดทุกอย่างทันที

State Sequence:
  IDLE → CLOSE_MOLD → INJECT → COOL → OPEN_MOLD → EJECT → IDLE

ให้ผู้เรียนเขียน Ladder Program
(เฉลยอยู่ใน Part 07)
```

---

*[← Part 04: Timer Instructions] [→ Part 06: Memory Bits และ Internal Coils]*
