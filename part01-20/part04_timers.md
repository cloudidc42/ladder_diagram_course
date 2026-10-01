# Part 04: Timer Instructions
## Steps 51-70: TON, TOF, RTO, TONR และการประยุกต์ใช้งาน

---

## Step 51: Timer คืออะไรและทำงานอย่างไร?

```
TIMER FUNDAMENTALS:

Timer คือ Instruction ที่นับเวลาใน PLC

โครงสร้าง Timer:
  ┌─────────────────────────────────────────────────────┐
  │  TIMER                                              │
  │  ──────────────────────────────────────────         │
  │  IN  (Input)     : สัญญาณ Enable Timer             │
  │  PT  (Preset Time): เวลาที่ตั้งไว้ (Set Point)      │
  │  Q   (Output)    : เอาต์พุตเมื่อ Timer Done        │
  │  ET  (Elapsed Time): เวลาที่นับไปแล้ว              │
  └─────────────────────────────────────────────────────┘

Time Base (Resolution):
  - 1 ms  : High-resolution (เร็ว)
  - 10 ms : Standard  
  - 100 ms: Normal applications
  - 1 s   : Slow applications

ตัวอย่าง:
  PT = 5s, Time Base = 100ms
  → ET นับ 0, 100ms, 200ms, ... 5000ms
  → เมื่อ ET ≥ PT → Q = ON
```

---

## Step 52: TON (Timer ON Delay)

### TON คือ Timer ที่หน่วงเวลาก่อน ON
```
TON - TIMER ON DELAY

สัญลักษณ์ (IEC 61131-3):
    ┌─────────────┐
────┤ IN   TON  Q ├────
    │      PT     │
    └─────────────┘

การทำงาน:
  IN=1: Timer เริ่มนับ ET
  ET ≥ PT: Q = 1
  IN=0: Timer Reset (ET=0, Q=0)

Timeline:
  IN:  0 0 1 1 1 1 1 1 1 0 0 1 1 1 1 0
  ET:  0 0 0 1 2 3 4 5 5 0 0 0 1 2 3 0  (ต่อวินาที, PT=5s)
  Q:   0 0 0 0 0 0 0 1 1 0 0 0 0 0 0 0
                         ↑               
                    ET=PT → Q=1

หมายเหตุ:
  - Q=1 เฉพาะตอน IN=1 และ ET≥PT
  - เมื่อ IN=0 → ทุกอย่าง Reset ทันที
  - ต้อง IN=1 ค้างนาน ≥ PT จึงจะ Q=1
```

### Ladder Diagram TON
```
TON IN LADDER DIAGRAM:

Siemens TIA Portal:
    ──┤ Coil_Condition ├──[TON T1 PT:=T#5s]──

    Reading ET: T1.ET (Elapsed Time)
    Reading Q:  T1.Q  (Output bit)

CODESYS / IEC Standard:
    ──┤ Condition ├──┤ TON
                      IN  Q ├── Output
                      PT
                       T#5s

Allen-Bradley Studio 5000:
    |──[ I0.0 ]──[TON Timer:T4:0 Preset:5000 Accum:0]──|
    Reading: T4:0/DN (Done Bit), T4:0.ACC (Accumulated Value)

Mitsubishi GX Works3 (FX5U):
    |──[ X0 ]──[OUT T0 K50]──|    (K50 = 50 × 100ms = 5s)
    Reading: T0 as Contact
```

---

## Step 53: TON ตัวอย่างการใช้งาน

### Motor Startup Delay
```
EXAMPLE 1: MOTOR START WITH DELAY

โจทย์: กดปุ่ม Start → รอ 3 วินาที → Motor เริ่มทำงาน
       (ให้เวลาพนักงานถอยออกจากเครื่อง)

    ─── Rung 1: Start/Stop ──────────────────────────────
    |──[ I0.0 ]──┬──[/I0.1]──(SET M0.0)──|
    |  START_PB  │  STOP_PB   SYS_START  |
    |            │                        |
    |──[ M0.0 ]──┘                        |

    ─── Rung 2: Startup Timer ───────────────────────────
    |──[ M0.0 ]──[TON T0 PT=3s]──────────|
    |  SYS_START   Startup Timer          |

    ─── Rung 3: Motor Output ─────────────────────────────
    |──[ T0.Q ]──────────────────( Q0.0 )──|
    |  TIMER_DONE                  MOTOR   |

    Timeline:
    t=0: START_PB กด → M0.0=1
    t=0-3s: T0 นับ → Motor ยังไม่ ON
    t=3s: T0.Q=1 → Motor ON!
    t=?: STOP_PB กด → M0.0=0 → T0 Reset → Motor OFF
```

### Warning Before Stop
```
EXAMPLE 2: WARNING BEFORE MACHINE STOP

โจทย์: กด Stop → Buzzer 5 วินาที → หยุดเครื่อง

    ─── Rung 1: Stop Trigger ────────────────────────────
    |──[ I0.1 ]──[P]────────────(SET M0.1)──|
    |  STOP_PB   RISE_EDGE       STOP_SEQ   |

    ─── Rung 2: Warning Timer ───────────────────────────
    |──[ M0.1 ]──[TON T1 PT=5s]──────────|
    |  STOP_SEQ   Warning Timer           |

    ─── Rung 3: Buzzer ON ───────────────────────────────
    |──[ M0.1 ]──[/T1.Q]────────( Q0.1 )──|
    |  STOP_SEQ   NOT_DONE       BUZZER    |
    
    (Buzzer ON ระหว่าง Stop_Seq=1 แต่ยังไม่ Done)

    ─── Rung 4: Machine Stop ────────────────────────────
    |──[ T1.Q ]──────────────────(RST M0.0)──|
    |  TIMER_DONE                  MACHINE    |
```

---

## Step 54: TOF (Timer OFF Delay)

### TOF หน่วงเวลาก่อน OFF
```
TOF - TIMER OFF DELAY

การทำงาน:
  IN=1: Q = 1 ทันที, ET = 0 (ยังไม่นับ)
  IN=0: Timer เริ่มนับ ET
  ET ≥ PT: Q = 0
  IN=1 อีกครั้ง: ET Reset = 0, Q ยังคง = 1

Timeline (PT = 5s):
  IN:  0 0 1 1 1 1 1 0 0 0 0 0 0 1 1 0
  ET:  0 0 0 0 0 0 0 1 2 3 4 5 5 0 0 1
  Q:   0 0 1 1 1 1 1 1 1 1 1 0 0 1 1 1
                               ↑
                           ET=PT → Q=0 (OFF หลัง 5 วินาที)

ใช้สำหรับ:
  - Fan ยังหมุนอยู่หลังจากปิดเครื่อง (Cooling Fan)
  - Conveyor ยังเดินอยู่หลัง Part หมด
  - Lubricator ทำงานต่ออีกสักครู่
```

### TOF Cooling Fan Example
```
EXAMPLE: COOLING FAN WITH TOF

โจทย์: เปิดเครื่อง → พัดลมระบายความร้อนทำงาน
       ปิดเครื่อง → พัดลมยังทำงานต่ออีก 30 วินาที แล้วจึงหยุด

    ─── Rung 1: Machine Running ─────────────────────────
    |──[ I0.0  ]──┬──( M0.0 )──|
    |  START_PB   │   MACHINE   |
    |             │             |
    |──[ M0.0  ]──┘             |
    
    |──[ I0.1  ]──(RST M0.0)──|
    |  STOP_PB     MACHINE     |

    ─── Rung 2: TOF Timer ───────────────────────────────
    |──[ M0.0 ]──[TOF T2 PT=30s]─────────|
    |  MACHINE    Cooling Timer           |
    
    ─── Rung 3: Fan Output ──────────────────────────────
    |──[ T2.Q ]──────────────────( Q0.1 )──|
    |  TIMER_Q                     FAN     |

    Timeline:
    t=0: Machine ON → M0.0=1 → T2.IN=1 → T2.Q=1 → Fan ON
    t=10s: Machine OFF → M0.0=0 → T2 เริ่มนับ (Fan ยัง ON!)
    t=40s: T2.ET=30s → T2.Q=0 → Fan OFF
```

---

## Step 55: RTO / TONR (Retentive Timer ON Delay)

### RTO จำค่าเมื่อ IN ดับ
```
TONR / RTO - RETENTIVE TIMER ON DELAY

ต่างจาก TON ตรงที่:
  - เมื่อ IN=0 → ET ไม่ Reset! (จำค่าไว้)
  - ET จะนับต่อเมื่อ IN=1 อีกครั้ง
  - Reset ด้วย Reset Input เท่านั้น

Timeline (PT = 10s):
  IN:    0 0 1 1 1 0 0 1 1 1 1 0 0 0
  ET:    0 0 0 1 2 2 2 2 3 4 5 5 5 0 ← RST ที่ท้าย
  Q:     0 0 0 0 0 0 0 0 0 0 1 1 1 0
  RST:   0 0 0 0 0 0 0 0 0 0 0 0 0 1

ใช้สำหรับ:
  - นับเวลาทำงานสะสมของเครื่องจักร
  - Machine Running Hours
  - Maintenance Timer
  - Part Cycle Time Accumulate
```

### Running Hours Counter
```
EXAMPLE: MACHINE RUNNING HOURS

โจทย์: นับชั่วโมงทำงานสะสมของ Motor
       เก็บค่าไว้แม้ PLC ดับแล้วเปิดใหม่

    ─── Rung 1: Running Hour Timer ──────────────────────
    |──[ Q0.0 ]──[TONR T3 PT=3600s]──────|
    |  MOTOR_RUN   1 Hour Timer           |
    
    ─── Rung 2: Increment Hour Counter ─────────────────
    |──[ T3.Q ]──[CTU C0 PV=9999]────────|
    |  1HR_DONE    Hour Counter           |
    
    ─── Rung 3: Reset Timer after 1 Hour ────────────────
    |──[ T3.Q ]──────────────────(RST T3)──|
    |  1HR_DONE                   TIMER    |
    
    ─── Rung 4: Hour Display (to HMI) ──────────────────
    |──[MOVE C0.ACC → MW100]──────────────|
    |  Copy Counter to HMI Display        |

    โปรแกรมนี้: ทุกๆ ชั่วโมงที่ Motor ทำงาน → C0 +1
    MW100 = ชั่วโมงรวมที่ Motor ทำงาน
```

---

## Step 56: TOFF (Timer OFF Delay Retentive)

```
TP - PULSE TIMER (IEC 61131-3)

การทำงาน:
  IN: 0→1 Rising Edge → Q=1 เป็นเวลา PT วินาที
  ไม่ว่า IN จะดับเมื่อไหร่ระหว่างนั้น → Q ยังคง ON
  จนกว่า ET = PT → Q=0

Timeline (PT = 5s):
  IN:  0 0 1 1 0 0 0 0 0 0 0 0 0 0 1 0
  ET:  0 0 0 1 2 3 4 5 5 0 0 0 0 0 0 1
  Q:   0 0 1 1 1 1 1 0 0 0 0 0 0 0 1 1
  
  ↑ ไม่ว่า IN จะดับที่ t=4 → Q ยัง ON จน t=7

ใช้สำหรับ:
  - Solenoid Pulse
  - One-Shot Output
  - Monoflop (Single Pulse)
  - Warning Lamp Flash (ครั้งเดียว)
```

---

## Step 57: Cascaded Timers

### วิธีต่อ Timer หลายตัว
```
CASCADED TIMERS - Sequential Timing:

โจทย์: 
  Phase 1: 5 วินาที → Phase 2: 3 วินาที → Phase 3: 2 วินาที

    ─── Rung 1: Start Sequence ──────────────────────────
    |──[ I0.0 ]──┬──────────────(SET M0.0)──|
    |  START     │               SEQ_RUN    |
    |            │                          |
    |──[ M0.0 ]──┘                          |

    ─── Rung 2: Phase 1 Timer ───────────────────────────
    |──[ M0.0 ]──[TON T0 PT=5s]──────────|
    |  SEQ_RUN    Phase 1 Timer           |

    ─── Rung 3: Phase 2 Timer ───────────────────────────
    |──[ T0.Q ]──[TON T1 PT=3s]──────────|
    |  P1_DONE    Phase 2 Timer           |

    ─── Rung 4: Phase 3 Timer ───────────────────────────
    |──[ T1.Q ]──[TON T2 PT=2s]──────────|
    |  P2_DONE    Phase 3 Timer           |

    ─── Rung 5: Phase Outputs ───────────────────────────
    |──[ M0.0 ]──[/T0.Q]─────────( Q0.0 )──|  Phase 1
    |──[ T0.Q ]──[/T1.Q]─────────( Q0.1 )──|  Phase 2
    |──[ T1.Q ]──[/T2.Q]─────────( Q0.2 )──|  Phase 3

    ─── Rung 6: End Sequence ────────────────────────────
    |──[ T2.Q ]──────────────────(RST M0.0)──|
    |  ALL_DONE                   SEQ_RUN    |

    Timeline:
    t=0:  START → M0.0=1 → Q0.0=1 (Phase 1)
    t=5:  T0 Done → Q0.0=0, Q0.1=1 (Phase 2)
    t=8:  T1 Done → Q0.1=0, Q0.2=1 (Phase 3)
    t=10: T2 Done → Q0.2=0, M0.0=0, Sequence End
```

---

## Step 58: Timer ใน State Machine

### Traffic Light (Part 01 เฉลย)
```
TRAFFIC LIGHT WITH TIMERS (Full Solution):

I/O:
  I0.0 = START_PB
  Q0.0 = GREEN_LIGHT  (5 วินาที)
  Q0.1 = YELLOW_LIGHT (2 วินาที)
  Q0.2 = RED_LIGHT    (5 วินาที)
  M0.0 = STATE_GREEN
  M0.1 = STATE_YELLOW
  M0.2 = STATE_RED
  M1.0 = SYSTEM_RUN

    ─── Rung 1: Start System ─────────────────────────────
    |──[ I0.0 ]──────────────────(SET M1.0)──|
    |  START_PB                    SYS_RUN   |
    
    ─── Rung 2: Initial State (Green) ───────────────────
    |──[ M1.0 ]──[/M0.0]──[/M0.1]──[/M0.2]──(SET M0.0)──|
    |  SYS_RUN   NOT_GRN  NOT_YEL  NOT_RED   SET_GREEN   |
    
    ─── Rung 3: Green Timer ──────────────────────────────
    |──[ M0.0 ]──[TON T0 PT=5s]──────────────────────────|
    |  GREEN_ST   Green Timer                             |
    
    ─── Rung 4: Green → Yellow Transition ───────────────
    |──[ T0.Q ]──[ M0.0 ]──────────────(RST M0.0)──|
    |  GRN_DONE  GREEN_ST                GREEN_ST   |
    
    |──[ T0.Q ]──[ M0.0 ]──────────────(SET M0.1)──|
    |  GRN_DONE  GREEN_ST               YELLOW_ST  |
    
    ─── Rung 5: Yellow Timer ─────────────────────────────
    |──[ M0.1 ]──[TON T1 PT=2s]──────────────────────────|
    |  YELLOW_ST  Yellow Timer                            |
    
    ─── Rung 6: Yellow → Red Transition ─────────────────
    |──[ T1.Q ]──[ M0.1 ]──────────────(RST M0.1)──|
    |──[ T1.Q ]──[ M0.1 ]──────────────(SET M0.2)──|
    
    ─── Rung 7: Red Timer ────────────────────────────────
    |──[ M0.2 ]──[TON T2 PT=5s]──────────────────────────|
    |  RED_ST     Red Timer                               |
    
    ─── Rung 8: Red → Green Transition ──────────────────
    |──[ T2.Q ]──[ M0.2 ]──────────────(RST M0.2)──|
    |──[ T2.Q ]──[ M0.2 ]──────────────(SET M0.0)──|
    
    ─── Rung 9: Outputs ──────────────────────────────────
    |──[ M0.0 ]──────────────────────( Q0.0 )──|  Green
    |──[ M0.1 ]──────────────────────( Q0.1 )──|  Yellow
    |──[ M0.2 ]──────────────────────( Q0.2 )──|  Red
```

---

## Step 59: Flashing/Blinking Output ด้วย Timer

```
FLASHING OUTPUT (Blink):

วิธีที่ 1: Self-Oscillating Timer
    ─── Rung 1: ─────────────────────────────────────────
    |──[/T_FLASH.Q]──[TON T_FLASH PT=0.5s]──────────────|
    |                 Flash Timer                        |
    
    ─── Rung 2: ─────────────────────────────────────────
    |──[ T_FLASH.Q ]──────────────────( M_FLASH )────────|
    
    การทำงาน:
    T_FLASH.Q=0 → TON Enable → นับ 0.5s → Q=1
    T_FLASH.Q=1 → TON Disable (IN=0) → Reset → Q=0
    → วนซ้ำ Flash 0.5s ON / 0.5s OFF = 1Hz

วิธีที่ 2: Siemens System Clock (ง่ายกว่า)
    Siemens มี System Clock Bits:
    M0.5 = Clock 0.5Hz (1s ON / 1s OFF)
    M1.0 = Clock 1Hz (0.5s ON / 0.5s OFF)
    M2.0 = Clock 2Hz
    
    |──[ M1.0 ]──[ FAULT_BIT ]──( FAULT_LAMP )──|
    (Fault Lamp กระพริบ 1Hz เมื่อมี Fault)

วิธีที่ 3: TP Pulse Timer
    |──[/T_BLINK.Q]──[TP T_BLINK PT=0.5s]──|
    |──[ T_BLINK.Q ]──────────(LAMP)────────|
```

---

## Step 60: Timer Address ใน PLC ต่างยี่ห้อ

```
TIMER ADDRESS BY PLC BRAND:

SIEMENS S7-1200/1500 (TIA Portal):
─────────────────────────────────────────────────────────
  TON สร้างเป็น Function Block:
  
  DB1.myTON.IN    = Input Enable
  DB1.myTON.PT    = Preset Time (TIME Type: T#5s)
  DB1.myTON.Q     = Output Done Bit
  DB1.myTON.ET    = Elapsed Time
  
  ตัวอย่าง TIA Portal:
  [DB1.myTON(IN:=I0.0, PT:=T#5s)] → Q→Q0.0
  
  Time Data Type: T#5s, T#1m30s, T#2h, T#500ms

ALLEN-BRADLEY (Studio 5000):
─────────────────────────────────────────────────────────
  T4:0      = Timer File 4, Element 0
  T4:0.PRE  = Preset (in ms): 5000 = 5 seconds
  T4:0.ACC  = Accumulated Value
  T4:0/EN   = Enable Bit
  T4:0/TT   = Timer Timing Bit
  T4:0/DN   = Done Bit
  
  Ladder:
  |──[ I:0/0 ]──[TON Timer:T4:0 Preset:5000 Accum:0]──|
  |──[ T4:0/DN ]──────────────────────( O:0/0 )────────|

MITSUBISHI FX3U/FX5U:
─────────────────────────────────────────────────────────
  T0 - T199: 100ms Timer (K1 = 100ms)
  T200-T245: 10ms Timer
  T246-T249: 1ms Timer (FX3U)
  
  Ladder:
  |──[ X0 ]──[OUT T0 K50]──|    (K50 × 100ms = 5s)
  |──[ T0 ]──────────────(Y0)──|
  
  FX5U:
  |──[ X0 ]──[TOUT T0 K5000]──|   (K5000 × 1ms = 5s)

OMRON CP1H:
─────────────────────────────────────────────────────────
  TIM 0000 - TIM 0004: Default (100ms base)
  Ladder:
  |──[ 0.00 ]──[TIM 0000 #0050]──|   (#0050 × 100ms = 5s)
  |──[ T0000 ]─────────────(100.00)──|
```

---

## Step 61: Interval Timer สำหรับ Periodic Tasks

```
PERIODIC INTERVAL TIMER:

โจทย์: ต้องการ Output ON ทุกๆ 10 วินาที เป็นเวลา 1 วินาที

    ─── Rung 1: Interval Timer (10s) ────────────────────
    |──[/T_INT.Q]──[TON T_INT PT=10s]────────────────────|
    |  NOT_PULSE    Interval Timer                        |
    
    ─── Rung 2: Pulse Output (1s) ───────────────────────
    |──[ T_INT.Q ]──[TP T_PUL PT=1s]──────────────────────|
    |  PULSE_TRIG    1 Second Pulse                        |
    
    ─── Rung 3: Output ───────────────────────────────────
    |──[ T_PUL.Q ]──────────────────────( Q0.0 )──────────|
    
    Timeline:
    t=0:  T_INT เริ่มนับ...
    t=10: T_INT.Q=1 → T_PUL เริ่ม 1s → Q0.0=1
    t=11: T_PUL.Q=0 → Q0.0=0, T_INT Reset และเริ่มใหม่
    t=21: Q0.0=1 อีกครั้ง
    ... วนซ้ำ

ใช้สำหรับ:
  - Lubricator ทำงานทุก 10 นาที
  - Check Valve Test ทุก 1 ชั่วโมง
  - Send Data ทุก 5 วินาที
```

---

## Step 62: Timer กับ Analog Value

```
TIMER WITH ANALOG SCALING:

โจทย์: ตั้งเวลาจาก HMI Potentiometer (0-100%)
       0% = 0 วินาที, 100% = 60 วินาที

    ─── Rung 1: Scale Pot Value ─────────────────────────
    Input: AI0 = 0-27648 (Siemens 12-bit Analog)
    Target: PT = 0-60000 ms
    
    Formula: PT_ms = (AI0 / 27648) × 60000
    
    ใน Ladder (Structured Text ใน TIA Portal):
    PT_ms := INT_TO_TIME((WORD_TO_INT(AI0) * 60000) / 27648);
    
    ─── Rung 2: Timer with Variable PT ─────────────────
    |──[ I0.0 ]──[TON T_VAR PT:=PT_ms]──────────────────|
    
    ─── Rung 3: Output ──────────────────────────────────
    |──[T_VAR.Q]──────────────────────( Q0.0 )──────────|
```

---

## Step 63: WDT (Watchdog Timer) Application

```
WATCHDOG TIMER PATTERN:

โจทย์: ตรวจจับว่า Sensor ส่งสัญญาณมาทุกๆ ไม่เกิน 5 วินาที
       ถ้าไม่ส่งมา 5 วินาที → แจ้งเตือน Sensor Fail

    ─── Rung 1: Sensor Heartbeat ────────────────────────
    Sensor ส่ง Pulse ทุก 1 วินาที → I0.0 กระพริบ
    
    ─── Rung 2: Reset WDT on Pulse ──────────────────────
    |──[ I0.0 ]──[P]──────────────(RST T_WDT)──|
    |  SENSOR    RISE_EDGE          WDT_TIMER   |
    
    ─── Rung 3: WDT Timer ───────────────────────────────
    |──[/T_WDT.Q]──[TON T_WDT PT=5s]────────────────────|
    |              Watchdog 5s                           |
    
    ─── Rung 4: Fail Alarm ──────────────────────────────
    |──[ T_WDT.Q ]──────────────────(SET M_FAIL)──────────|
    |  WDT_EXPIRED                    SENSOR_FAIL         |
    
    การทำงาน:
    - ทุกครั้งที่ Sensor ส่ง Pulse → Reset T_WDT
    - T_WDT เริ่มนับใหม่จาก 0
    - ถ้า Sensor ไม่ส่งมา 5 วินาที → T_WDT.Q=1 → Alarm!
```

---

## Step 64: Timer ใช้กับ Startup/Shutdown Sequence

### Chiller Plant Startup Sequence
```
CHILLER PLANT STARTUP SEQUENCE:

Order:
  1. Start Cooling Tower Fan (30s stabilize)
  2. Start Condenser Water Pump (20s stabilize)
  3. Open Condenser Water Valve (10s open)
  4. Start Chiller (Main Unit)
  5. Start Chilled Water Pump

LADDER PROGRAM:

    ─── Rung 1: Startup Trigger ──────────────────────────
    |──[ I0.0  ]──[P]────────────(SET M0.0)──|
    |  START_PB  RISE             SEQ_RUN    |

    ─── Rung 2: Step 1 - CT Fan ─────────────────────────
    |──[ M0.0 ]──────────────────( Q0.0 )──|  CT Fan
    |──[ M0.0 ]──[TON T0 PT=30s]──────────|  Stabilize Timer

    ─── Rung 3: Step 2 - CW Pump ────────────────────────
    |──[ T0.Q ]──────────────────( Q0.1 )──|  CW Pump
    |──[ T0.Q ]──[TON T1 PT=20s]──────────|  Stabilize Timer

    ─── Rung 4: Step 3 - CW Valve ───────────────────────
    |──[ T1.Q ]──────────────────( Q0.2 )──|  Open Valve
    |──[ T1.Q ]──[TON T2 PT=10s]──────────|  Open Time

    ─── Rung 5: Step 4 - Chiller ────────────────────────
    |──[ T2.Q ]──────────────────( Q0.3 )──|  Chiller Start
    |──[ T2.Q ]──[TON T3 PT=60s]──────────|  Stabilize Timer

    ─── Rung 6: Step 5 - CHW Pump ───────────────────────
    |──[ T3.Q ]──────────────────( Q0.4 )──|  CHW Pump
    
    ─── Rung 7: Startup Complete ────────────────────────
    |──[ T3.Q ]──────────────────( M0.1 )──|  Startup Done Indicator

    Timeline:
    t=0:  CT Fan ON
    t=30: CW Pump ON
    t=50: CW Valve OPEN
    t=60: Chiller START
    t=120: CHW Pump ON → System Running!
```

---

## Step 65: Timer Troubleshooting

```
COMMON TIMER PROBLEMS & SOLUTIONS:

Problem 1: Timer ไม่นับ
─────────────────────────────────────────
Cause:    IN = 0 (Enable ไม่ได้)
Check:    - Monitor IN signal
          - Check wiring
          - Check logic conditions before IN

Problem 2: Timer นับแล้ว Reset เอง (ไม่ถึง PT)
─────────────────────────────────────────
Cause:    IN Pulsing (ไม่ Stable)
Check:    - Monitor IN on oscilloscope
          - หา Contact Bounce
          - เพิ่ม Debounce

Problem 3: Q ไม่ ON ทั้งที่ ET ถึง PT แล้ว
─────────────────────────────────────────
Cause:    IN กลับเป็น 0 ก่อน ET ถึง PT
Solution: ใช้ TONR แทน TON (Retentive)

Problem 4: Time ไม่แม่นยำ
─────────────────────────────────────────
Cause 1:  Time Base ผิด (ตั้ง 10ms แต่คิด 100ms)
Cause 2:  Scan Time ใหญ่เกิน
Cause 3:  CPU Load สูง
Solution: ตรวจสอบ Time Base Setting
          ใช้ Interrupt Routine สำหรับ Precision Timing

Problem 5: Timer Reset ไม่ได้
─────────────────────────────────────────
Solution:  TONR ต้องมี RESET Input ON
           TON Reset เองเมื่อ IN=0
```

---

## Step 66: Multiple Timer Patterns

```
MULTI-TIMER PATTERNS:

PATTERN 1: Sequential (ทำงานต่อเนื่อง)
    T0.Q → Enable T1 → T1.Q → Enable T2 → T2.Q → Done

PATTERN 2: Parallel (ทำงานพร้อมกัน)
    Trigger → Enable T0, T1, T2 พร้อมกัน
    T0.Q AND T1.Q AND T2.Q = All Done

PATTERN 3: One of N (อย่างน้อยหนึ่งตัว Done)
    T0.Q OR T1.Q OR T2.Q = Any Done

PATTERN 4: Repeat (วนซ้ำ)
    T0.Q → Reset T0 → นับใหม่ (Oscillator)

PATTERN 5: Timeout (Watchdog)
    Start_Signal → Reset T0
    T0.Q → Alarm (ถ้า Start_Signal ไม่มาภายใน PT)
```

---

## Step 67: Timer ใน IEC Standard Blocks

```
IEC 61131-3 STANDARD TIMER BLOCKS:

All timers: IN (BOOL), PT (TIME), Q (BOOL), ET (TIME)

TON - Timer On Delay:
  Q: IN=1 และ ET≥PT → Q=1
  Reset: IN=0 → Q=0, ET=0

TOF - Timer Off Delay:
  Q: IN=1 → Q=1 ทันที
     IN=0 → นับ PT แล้ว Q=0
  Reset: IN=1 → ET=0

TP - Pulse:
  Q: Rising Edge of IN → Q=1 เป็นเวลา PT
  ระหว่างนับ: ไม่สนใจ IN

TONR - Retentive Timer On Delay:
  IN: Enable การนับ
  R (Reset): Reset ET=0, Q=0
  Q: IN=1 และ ET≥PT (Retentive)

Time Constants:
  T#1s     = 1 second
  T#500ms  = 500 milliseconds  
  T#1m     = 1 minute
  T#1h     = 1 hour
  T#1d     = 1 day
  T#1m30s  = 1 minute 30 seconds
  T#2h30m15s200ms = Complex
```

---

## Step 68: Timer กับ Production Control

```
PRODUCTION CYCLE TIME MONITORING:

โจทย์: วัดเวลาของแต่ละ Production Cycle
       ถ้าเกิน 30 วินาที → Alarm ช้าเกินไป
       เก็บข้อมูล Cycle Time สำหรับ OEE Calculation

    ─── Rung 1: Cycle Start Detection ──────────────────
    |──[ I0.0 ]──[P]────────────(RST T_CYCLE)──|
    |  PART_IN   RISE             Cycle Timer  |
    
    |──[ I0.0 ]──[P]────────────(SET M0.0)──────|
    |  PART_IN   RISE              CYCLE_RUN    |

    ─── Rung 2: Cycle Timer ─────────────────────────────
    |──[ M0.0 ]──[TONR T_CYCLE PT=9999s]────────────────|
    |  CYCLE_RUN   Accumulate Cycle Time                 |
    
    ─── Rung 3: Cycle End ───────────────────────────────
    |──[ I0.1 ]──[P]────────────(RST M0.0)──────|
    |  PART_OUT  RISE             CYCLE_RUN     |
    
    ─── Rung 4: Save Cycle Time ─────────────────────────
    |──[ I0.1 ]──[P]──[MOVE T_CYCLE.ET → MD_LAST_CYCLE]──|
    
    ─── Rung 5: Timeout Alarm ───────────────────────────
    |──[ T_CYCLE.Q ]──────────────────(SET SLOW_ALARM)──|
    |  30s Exceeded                     Slow Cycle Alarm |
    
    Wait - T_CYCLE PT = 30s for timeout:
    |──[ M0.0 ]──[TON T_TIMEOUT PT=30s]─────────────────|
    |──[ T_TIMEOUT.Q ]────────────────(SET SLOW_ALARM)──|
```

---

## Step 69: Timer Best Practices

```
TIMER BEST PRACTICES:

1. ตั้งชื่อ Timer ให้สื่อความหมาย
   ✅ T_MOTOR_START_DELAY, T_COOLING_FAN
   ❌ T0, T1, T2

2. Comment เหตุผลของค่า PT
   ✅ PT=5s  // Minimum startup time per datasheet
   ❌ PT=5s

3. ใช้ Symbolic Time Constants
   ✅ PT := STARTUP_DELAY (= T#5s กำหนดที่ Constant)
   ❌ PT=5000 (มิลลิวินาที แต่ไม่รู้ว่าหน่วยอะไร)

4. ระวัง Timer Reuse
   ❌ ใช้ Timer เดียวกันใน 2 Rung พร้อมกัน
   ✅ สร้าง Timer ใหม่สำหรับแต่ละ Function

5. Test Timer ด้วย Simulation
   - ทดสอบค่า Boundary (PT-1, PT, PT+1)
   - ทดสอบการ Cancel (IN=0 ก่อนครบเวลา)
   - ทดสอบ Repeat Operation

6. สำหรับ Critical Timing ใช้ Interrupt
   - Scan-based Timer มี ±1 Scan Time Error
   - High-Speed ใช้ Hardware Timer / Interrupt OB
```

---

## Step 70: Project Part 04 - Washing Machine Control

```
PROJECT: WASHING MACHINE CYCLE CONTROL

Sequence:
  ┌────────────────────────────────────────────────────────┐
  │ 1. Fill Water    (2 min)                              │
  │ 2. Wash Cycle    (30 min, Motor CW)                   │
  │ 3. Drain Water   (3 min)                              │
  │ 4. Rinse Fill    (1 min)                              │
  │ 5. Spin          (10 min, Motor CCW High Speed)       │
  │ 6. Final Drain   (2 min)                              │
  │ 7. Done Buzzer   (5 sec)                              │
  └────────────────────────────────────────────────────────┘

I/O:
  I0.0 = START_PB         ปุ่มเริ่ม
  I0.1 = STOP_PB          ปุ่มหยุด
  I0.2 = DOOR_SWITCH      ประตู (NC = ปิดอยู่)
  I0.3 = WATER_LEVEL_OK   น้ำเต็มแล้ว
  I0.4 = WATER_EMPTY      น้ำหมดแล้ว
  
  Q0.0 = FILL_VALVE       วาล์วน้ำเข้า
  Q0.1 = DRAIN_PUMP       ปั๊มระบาย
  Q0.2 = MOTOR_WASH       Motor ซัก (CW)
  Q0.3 = MOTOR_SPIN       Motor ปั่น (CCW High Speed)
  Q0.4 = DONE_BUZZER      Buzzer แจ้งเสร็จ
  Q0.5 = CYCLE_LED        ไฟแสดงทำงาน

  M0.0 = SYS_RUN
  M1.0-M1.6 = STEP 1-7 Flags

State Diagram:
IDLE → FILL → WASH → DRAIN → RINSE → SPIN → FINAL_DRAIN → DONE → IDLE

Condition Transitions:
  FILL  → WASH         : Water Level OK
  WASH  → DRAIN        : Timer 30min Done
  DRAIN → RINSE        : Water Empty
  RINSE → SPIN         : Timer 1min Done
  SPIN  → FINAL_DRAIN  : Timer 10min Done
  FINAL_DRAIN → DONE   : Water Empty
  DONE  → IDLE         : Buzzer 5sec Done

ให้ผู้เรียนเขียน Ladder Program ด้วยตัวเอง
(เฉลยอยู่ใน Part 06)
```

---

*[← Part 03: Logic Gates] [→ Part 05: Counter Instructions]*
