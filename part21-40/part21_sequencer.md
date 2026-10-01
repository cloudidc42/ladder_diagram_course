# Part 21: Sequencer และ State Machine (Steps 391-410)

## ภาพรวม
ในงานอุตสาหกรรมจริง กระบวนการผลิตมักมีลำดับขั้นตอนที่ชัดเจน เช่น เครื่องซักผ้า เครื่องเจาะ สายพานลำเลียง ล้วนทำงานแบบ "ขั้นตอนต่อขั้นตอน" การใช้ Ladder Diagram แบบธรรมดาจะทำให้โค้ดซับซ้อนมาก แต่เมื่อใช้แนวคิด **State Machine** หรือ **Sequencer** จะทำให้โปรแกรมอ่านง่ายและบำรุงรักษาได้ง่ายขึ้นมาก

---

## Step 391: แนวคิด Sequential Function Chart (SFC)

SFC (Sequential Function Chart) คือมาตรฐาน IEC 61131-3 ที่ใช้แทนการทำงานแบบลำดับขั้น ประกอบด้วย:

```
┌─────────────────────────────────────────────────┐
│           โครงสร้าง SFC พื้นฐาน                  │
│                                                   │
│         [Initial Step]  ──── Step 0               │
│               │                                   │
│          <Transition>   ──── เงื่อนไข Start        │
│               │                                   │
│           [Step 1]      ──── Fill Water           │
│               │                                   │
│          <Transition>   ──── Water Level OK       │
│               │                                   │
│           [Step 2]      ──── Wash                 │
│               │                                   │
│          <Transition>   ──── Timer Done            │
│               │                                   │
│           [Step 3]      ──── Drain                │
│               │                                   │
│          <Transition>   ──── Level Empty           │
│               │                                   │
│         [Initial Step]  ──── กลับ Step 0          │
└─────────────────────────────────────────────────┘
```

**องค์ประกอบหลักของ SFC:**
1. **Step (ขั้นตอน)** - สถานะที่ระบบอยู่ในขณะนั้น
2. **Transition (การเปลี่ยน)** - เงื่อนไขที่ทำให้เปลี่ยน Step
3. **Action (การกระทำ)** - สิ่งที่ทำระหว่างอยู่ใน Step นั้น

---

## Step 392: State Machine คืออะไร

State Machine คือระบบที่:
- อยู่ในสถานะเดียวในเวลาหนึ่ง
- เปลี่ยนสถานะได้เมื่อเงื่อนไขถูกต้อง
- แต่ละสถานะมีการกระทำของตัวเอง

```
┌──────────────────────────────────────────────────────┐
│           State Machine: Traffic Light               │
│                                                      │
│    ┌─────────┐  Timer Done  ┌─────────┐             │
│    │  GREEN  │ ──────────►  │ YELLOW  │             │
│    └─────────┘              └─────────┘             │
│         ▲                        │                  │
│         │                    Timer Done             │
│         │                        ▼                  │
│    Timer Done              ┌─────────┐             │
│         └──────────────────│   RED   │             │
│                            └─────────┘             │
└──────────────────────────────────────────────────────┘
```

**ข้อดีของ State Machine:**
- โค้ดเป็นระเบียบ ตาม Process ที่เป็นจริง
- Debug ง่าย รู้ว่าอยู่ Step ไหน
- เพิ่ม/แก้ Step ได้โดยไม่กระทบ Step อื่น
- เหมาะกับ PLC Programming มาก

---

## Step 393: การ Implement State Machine ใน Ladder

วิธีที่นิยมคือใช้ **Integer Register** เก็บ State Number แล้วใช้ Contact เปรียบเทียบ

### วิธีที่ 1: ใช้ Compare + Move

```
// Siemens S7 - ตัวอย่าง Traffic Light Controller
// DB1.DBW0 = State Register

// ─────── Rung 1: Initialize ─────────────────────────────
        ┌──[ FIRST SCAN ]──────────────────────────(MOVE)─┐
        │                                     IN: 0        │
        │                                    OUT: DB1.DBW0 │
        └──────────────────────────────────────────────────┘

// ─────── Rung 2: State 0 - GREEN ────────────────────────
        ┌──[CMP ==]────┬──[ GREEN_TIMER.Q ]──────────┐
        │  DB1.DBW0    │                         (MOVE)│
        │  == 0        │                    IN: 1      │
        └──────────────┘                   OUT: DB1.DBW0│
                       │                              │
                       └──────────────(TON)───────────┤
                              GREEN_TIMER T#5S        │
                                                      └─┘

// ─────── Rung 3: State 1 - YELLOW ────────────────────────
        ┌──[CMP ==]────┬──[ YELLOW_TIMER.Q ]─────────┐
        │  DB1.DBW0    │                         (MOVE)│
        │  == 1        │                    IN: 2      │
        └──────────────┘                   OUT: DB1.DBW0│
                       │                              │
                       └──────────────(TON)───────────┤
                              YELLOW_TIMER T#2S       │
                                                      └─┘

// ─────── Rung 4: State 2 - RED ────────────────────────
        ┌──[CMP ==]────┬──[ RED_TIMER.Q ]────────────┐
        │  DB1.DBW0    │                         (MOVE)│
        │  == 2        │                    IN: 0      │
        └──────────────┘                   OUT: DB1.DBW0│
                       │                              │
                       └──────────────(TON)───────────┤
                              RED_TIMER T#6S          │
                                                      └─┘
```

---

## Step 394: Step/Transition Method ใน Ladder

วิธีนี้ใช้ **Bit เดี่ยว** แทนแต่ละ Step (Active = Step กำลังทำงาน)

```
// Allen-Bradley ControlLogix - Step/Transition Method
// ใช้ Bool Array: Step[0] ถึง Step[9]

// ─────── Rung 1: Step 0 Active ───────────────────────────
     Step[0]    Transition_01         Step[1]
   ──┤ ├───────────┤ ├────────────────( )──

// ─────── Rung 2: Reset Step 0 เมื่อ Step 1 Activate ─────
     Step[1]                           Step[0]
   ──┤ ├─────────────────────────────(RST)──

// ─────── Rung 3: Step 1 Active ───────────────────────────
     Step[1]    Transition_12         Step[2]
   ──┤ ├───────────┤ ├────────────────( )──

// ─────── Rung 4: Reset Step 1 ────────────────────────────
     Step[2]                           Step[1]
   ──┤ ├─────────────────────────────(RST)──
```

**ข้อดี:** มองเห็น Step ที่ Active ได้ชัดเจน
**ข้อเสีย:** ต้องการ Bit หลายตัว

---

## Step 395: Grafcet Basics

Grafcet คือ Standard การออกแบบ Sequential Control ที่เป็นต้นแบบของ SFC

```
┌─────────────────────────────────────────────────────┐
│              Grafcet Notation                        │
│                                                      │
│        ╔═══════╗   ← Double box = Initial Step      │
│        ║   0   ║                                     │
│        ╚═══════╝                                     │
│            │                                         │
│        ════════   ← Double line = Initial Transition │
│        ════════                                      │
│            │                                         │
│        ┌───────┐   ← Single box = Normal Step        │
│        │   1   │                                     │
│        └───────┘                                     │
│            │                                         │
│        ────────   ← Single line = Normal Transition  │
│         [a·b]     ← Transition Condition             │
│            │                                         │
│        ┌───────┐                                     │
│        │   2   │                                     │
│        └───────┘                                     │
└─────────────────────────────────────────────────────┘
```

**กฎของ Grafcet:**
1. Step แรกเริ่มต้นเป็น Active (Initial Step)
2. Transition จะ Fire เมื่อ: Step ก่อนหน้า Active **และ** เงื่อนไขถูกต้อง
3. เมื่อ Transition Fire: Step หลังจะ Active, Step ก่อนหน้าจะ Deactive

---

## Step 396: ตัวอย่างจริง - เครื่องซักผ้าอุตสาหกรรม

```
┌─────────────────────────────────────────────────────────┐
│          Washing Machine Sequence (Industrial)          │
│                                                         │
│  Step 0: IDLE         ← รอรับ Start Command            │
│      │                                                  │
│  [Start Button]                                         │
│      │                                                  │
│  Step 1: FILL_WATER   ← เปิด Inlet Valve               │
│      │                                                  │
│  [Water Level High]                                     │
│      │                                                  │
│  Step 2: HEAT_WATER   ← เปิด Heater                    │
│      │                                                  │
│  [Temp >= 60°C]                                         │
│      │                                                  │
│  Step 3: WASH         ← หมุน Motor ไปมา 15 นาที        │
│      │                                                  │
│  [Wash Timer Done]                                      │
│      │                                                  │
│  Step 4: DRAIN        ← เปิด Drain Valve               │
│      │                                                  │
│  [Water Level Low]                                      │
│      │                                                  │
│  Step 5: RINSE_FILL   ← เติมน้ำใหม่                   │
│      │                                                  │
│  [Level High]                                           │
│      │                                                  │
│  Step 6: RINSE_SPIN   ← หมุน 5 นาที                   │
│      │                                                  │
│  [Rinse Timer Done]                                     │
│      │                                                  │
│  Step 7: FINAL_DRAIN  ← ระบายน้ำ                      │
│      │                                                  │
│  [Level Low]                                            │
│      │                                                  │
│  Step 8: SPIN_DRY     ← หมุนเร็ว 3 นาที               │
│      │                                                  │
│  [Spin Timer Done]                                      │
│      │                                                  │
│  Step 0: IDLE         ← กลับสถานะรอ                  │
└─────────────────────────────────────────────────────────┘
```

### โค้ด Ladder สำหรับเครื่องซักผ้า (Siemens S7-1200)

```
// Data Block DB10: Washing Machine Data
// DB10.DBW0 = Current_Step (INT)
// DB10.DBX2.0 = Start_Button
// DB10.DBX2.1 = Stop_Button
// DB10.DBX2.2 = Water_Level_High
// DB10.DBX2.3 = Water_Level_Low
// DB10.DBX2.4 = Temp_OK (>=60°C)
// DB10.DBX3.0 = Inlet_Valve    (Output)
// DB10.DBX3.1 = Drain_Valve    (Output)
// DB10.DBX3.2 = Heater         (Output)
// DB10.DBX3.3 = Motor_FWD      (Output)
// DB10.DBX3.4 = Motor_REV      (Output)

NETWORK 1: // Initialize
    ┌──[FIRST_SCAN]────────────────────────(MOVE)──┐
    │                                     IN: 0     │
    │                                OUT: DB10.DBW0 │
    └──────────────────────────────────────────────┘

NETWORK 2: // Step 0→1: รับคำสั่ง Start
    ┌──[CMP==]──┬──[DB10.DBX2.0]──────────(MOVE)──┐
    │ DB10.DBW0 │   Start_Button          IN: 1     │
    │   == 0    │                     OUT: DB10.DBW0│
    └───────────┘                                  │
                                                   └┘

NETWORK 3: // Step 1: เปิด Inlet Valve
    ┌──[CMP==]──────────────────────────────( )────┐
    │ DB10.DBW0                        Inlet_Valve  │
    │   == 1                           DB10.DBX3.0  │
    └──────────────────────────────────────────────┘

NETWORK 4: // Step 1→2: เมื่อน้ำเต็ม
    ┌──[CMP==]──┬──[DB10.DBX2.2]──────────(MOVE)──┐
    │ DB10.DBW0 │  Water_Level_High        IN: 2    │
    │   == 1    │                     OUT: DB10.DBW0│
    └───────────┘                                  │
                                                   └┘

NETWORK 5: // Step 2: เปิด Heater
    ┌──[CMP==]──────────────────────────────( )────┐
    │ DB10.DBW0                           Heater   │
    │   == 2                         DB10.DBX3.2   │
    └──────────────────────────────────────────────┘

NETWORK 6: // Step 2→3: เมื่ออุณหภูมิถึง
    ┌──[CMP==]──┬──[DB10.DBX2.4]──────────(MOVE)──┐
    │ DB10.DBW0 │      Temp_OK             IN: 3   │
    │   == 2    │                     OUT: DB10.DBW0│
    └───────────┘                                  │
                                                   └┘

NETWORK 7: // Step 3: Wash - Motor ไปมา (ใช้ Timer และ Alternating)
    ┌──[CMP==]──┬──────────────────────────────────┤
    │ DB10.DBW0 │                                   │
    │   == 3    │                                   │
    └───────────┘                                   │
                └──[WASH_TIMER.Q]/──────────( )────┐│
                                         Motor_FWD ││
                                     DB10.DBX3.3   ││
                                                   │└─(TON)──
                                              WASH_TIMER T#15M

NETWORK 8: // Step 3→4: Wash Timer Done
    ┌──[CMP==]──┬──[WASH_TIMER.Q]─────────(MOVE)──┐
    │ DB10.DBW0 │                          IN: 4   │
    │   == 3    │                     OUT: DB10.DBW0│
    └───────────┘                                  │
                                                   └┘

NETWORK 9: // Step 4: Drain
    ┌──[CMP==]──────────────────────────────( )────┐
    │ DB10.DBW0                        Drain_Valve  │
    │   == 4                           DB10.DBX3.1  │
    └──────────────────────────────────────────────┘

NETWORK 10: // Step 4→5: น้ำระบายหมด
    ┌──[CMP==]──┬──[DB10.DBX2.3]──────────(MOVE)──┐
    │ DB10.DBW0 │  Water_Level_Low          IN: 5  │
    │   == 4    │                     OUT: DB10.DBW0│
    └───────────┘                                  │
                                                   └┘
```

---

## Step 397: ตัวอย่าง - วงจรเจาะ (Drill Cycle)

วงจรเจาะเป็นตัวอย่างที่ดีมากสำหรับการสอน Sequencer เพราะมีขั้นตอนชัดเจน

```
┌─────────────────────────────────────────────────────────┐
│              Drill Cycle Sequence                        │
│                                                         │
│  Step 0: HOME         ← Drill อยู่ตำแหน่ง Up          │
│      │                                                  │
│  [Start + Part_Present]                                 │
│      │                                                  │
│  Step 1: CLAMP        ← หนีบชิ้นงาน                    │
│      │                                                  │
│  [Clamp_Confirmed + Delay 0.5s]                        │
│      │                                                  │
│  Step 2: RAPID_DOWN   ← ดิ่งลงเร็ว                     │
│      │                                                  │
│  [Approach_Position]                                    │
│      │                                                  │
│  Step 3: DRILL_FEED   ← เจาะด้วยความเร็วเจาะ           │
│      │                                                  │
│  [Drill_Depth_Reached]                                  │
│      │                                                  │
│  Step 4: DWELL        ← หยุดนิ่ง 1 วินาที              │
│      │                                                  │
│  [Dwell Timer Done]                                     │
│      │                                                  │
│  Step 5: RETRACT      ← ถอย Drill ขึ้น                  │
│      │                                                  │
│  [Home_Position]                                        │
│      │                                                  │
│  Step 6: UNCLAMP      ← ปลดหนีบ                        │
│      │                                                  │
│  [Unclamp_Confirmed]                                    │
│      │                                                  │
│  Step 7: EJECT        ← ดันชิ้นงานออก                  │
│      │                                                  │
│  [Ejector_Extended]                                     │
│      │                                                  │
│  Step 8: RETRACT_EJECTOR ← ดึง Ejector กลับ            │
│      │                                                  │
│  [Ejector_Retracted]                                    │
│      │                                                  │
│  Step 0: HOME         ← พร้อมทำงานใหม่                │
└─────────────────────────────────────────────────────────┘
```

### โค้ด Mitsubishi MELSEC (GX Works2)

```
// Mitsubishi FX5U - Drill Cycle
// D0 = Current Step (Word)
// M0 = Start_Button
// M1 = Part_Present
// M2 = Clamp_Confirmed
// M3 = Approach_Position
// M4 = Drill_Depth
// M5 = Home_Position
// M6 = Unclamp_Confirmed
// M7 = Ejector_Extended
// M8 = Ejector_Retracted
// Y0 = Clamp_Solenoid
// Y1 = Rapid_Down
// Y2 = Feed_Down
// Y3 = Retract_Up
// Y4 = Unclamp_Solenoid
// Y5 = Ejector_Forward

// Network 1: Initialize
    ┌──[M8002]──────────────────────────(MOV K0 D0)──┐
    │  Initial Pulse                                  │
    └─────────────────────────────────────────────────┘

// Network 2: Step 0 → 1
    ┌──[CMP= K0 D0]──┬──[M0]──┬──[M1]──(MOV K1 D0)──┐
    │  Step==0       │ Start  │ Part                  │
    └────────────────┘        └────────────────────────┘

// Network 3: Step 1 (Clamp)
    ┌──[CMP= K1 D0]──────────────────────────(Y0)────┐
    │  Step==1                           Clamp_Sol    │
    └─────────────────────────────────────────────────┘

// Network 4: Step 1 Timer
    ┌──[CMP= K1 D0]──┬──[M2]──────────(TON T0 K5)───┐
    │  Step==1       │ Clamp_Confirmed (50ms * 5 = 0.5s)│
    └────────────────┘                               └┘

// Network 5: Step 1 → 2
    ┌──[CMP= K1 D0]──┬──[T0]──────────(MOV K2 D0)───┐
    │  Step==1       │ Delay_Done                     │
    └────────────────┘                               └┘

// Network 6: Step 2 (Rapid Down)
    ┌──[CMP= K2 D0]──────────────────────────(Y1)────┐
    │  Step==2                          Rapid_Down    │
    └─────────────────────────────────────────────────┘

// Network 7: Step 2 → 3
    ┌──[CMP= K2 D0]──┬──[M3]──────────(MOV K3 D0)───┐
    │  Step==2       │ Approach_Pos                   │
    └────────────────┘                               └┘

// Network 8: Step 3 (Drill Feed)
    ┌──[CMP= K3 D0]──────────────────────────(Y2)────┐
    │  Step==3                           Feed_Down    │
    └─────────────────────────────────────────────────┘

// Network 9: Step 3 → 4
    ┌──[CMP= K3 D0]──┬──[M4]──────────(MOV K4 D0)───┐
    │  Step==3       │ Drill_Depth                    │
    └────────────────┘                               └┘
```

---

## Step 398: ตัวอย่าง - เครื่องบรรจุภัณฑ์ (Packaging Machine)

```
┌──────────────────────────────────────────────────────────┐
│            Packaging Machine Sequence                     │
│                                                          │
│  ┌──── Step 1: DETECT_PRODUCT ────────────────────────┐ │
│  │  Action: Conveyor ON                               │ │
│  │  Exit: Product_Sensor                              │ │
│  └──────────────────────────────────────────────────── ┘ │
│                         │                                │
│  ┌──── Step 2: STOP_CONVEYOR ─────────────────────────┐ │
│  │  Action: Conveyor OFF, Position Confirmed           │ │
│  │  Exit: Stop_Confirmed (0.2s delay)                 │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│  ┌──── Step 3: FILL_BOX ──────────────────────────────┐ │
│  │  Action: Fill Valve OPEN                           │ │
│  │  Exit: Weight_OK or Fill_Time                      │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│  ┌──── Step 4: SEAL_BOX ──────────────────────────────┐ │
│  │  Action: Sealer DOWN + Heat                        │ │
│  │  Exit: Seal_Timer (2 seconds)                      │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│  ┌──── Step 5: LIFT_SEALER ───────────────────────────┐ │
│  │  Action: Sealer UP                                 │ │
│  │  Exit: Sealer_Home                                 │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                                │
│  ┌──── Step 6: ADVANCE_CONVEYOR ──────────────────────┐ │
│  │  Action: Conveyor ON                               │ │
│  │  Exit: Product_Clear                               │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## Step 399: State Bits กับ Timer

การใช้ **Timer ภายใน Step** เป็นเทคนิคที่สำคัญ:

```
// Allen-Bradley - Motor Start Sequence with Timers
// เปิดมอเตอร์ทีละตัว ห่างกัน 3 วินาที ป้องกัน Inrush Current

// Tag Names:
// Step_Register (INT)
// Motor_1_ON, Motor_2_ON, Motor_3_ON (BOOL)
// Delay_Timer (TON)

RUNG 1: // Step 0: ทุกตัวปิด รอ Start
  ┌─[Step_Register EQU 0]──[Start_PB]────────────────────┐
  │                                    ┌─(MOV 1)────────┐│
  │                                    │  → Step_Register││
  └────────────────────────────────────┘                ││
                                                        └┘

RUNG 2: // Step 1: เปิด Motor 1 + เริ่ม Timer
  ┌─[Step_Register EQU 1]─────────────────────────( )───┐
  │                                              Motor_1 │
  └──────────────────────────────────────────────────────┘
  ┌─[Step_Register EQU 1]─────────────────────(TON)─────┐
  │                                          Delay_Timer │
  │                                           PT: 3s     │
  └──────────────────────────────────────────────────────┘

RUNG 3: // Step 1→2: เมื่อ Delay ครบ
  ┌─[Step_Register EQU 1]──[Delay_Timer.DN]──(MOV 2)────┐
  │                                        → Step_Register│
  └──────────────────────────────────────────────────────┘

RUNG 4: // Step 2: เปิด Motor 1,2 + Timer ต่อ
  ┌─[Step_Register EQU 2]─────────────────────────( )───┐
  │                                              Motor_1 │
  └──────────────────────────────────────────────────────┘
  ┌─[Step_Register EQU 2]─────────────────────────( )───┐
  │                                              Motor_2 │
  └──────────────────────────────────────────────────────┘
  ┌─[Step_Register EQU 2]─────────────────────(TON)─────┐
  │                                          Delay_Timer │
  │                                           PT: 3s     │
  └──────────────────────────────────────────────────────┘

RUNG 5: // Step 2→3
  ┌─[Step_Register EQU 2]──[Delay_Timer.DN]──(MOV 3)────┐
  │                                        → Step_Register│
  └──────────────────────────────────────────────────────┘

RUNG 6: // Step 3: Motor ทั้ง 3 ตัวทำงาน
  ┌─[Step_Register EQU 3]─────────────────────────( )───┐
  │                                              Motor_1 │
  └──────────────────────────────────────────────────────┘
  ┌─[Step_Register EQU 3]─────────────────────────( )───┐
  │                                              Motor_2 │
  └──────────────────────────────────────────────────────┘
  ┌─[Step_Register EQU 3]─────────────────────────( )───┐
  │                                              Motor_3 │
  └──────────────────────────────────────────────────────┘
```

---

## Step 400: Transition Conditions (เงื่อนไขการเปลี่ยน Step)

เงื่อนไขที่ดีต้องพิจารณา:

```
┌──────────────────────────────────────────────────────────┐
│           ประเภทของ Transition Conditions                 │
│                                                          │
│  1. SENSOR-BASED: ใช้ตำแหน่ง/สถานะ Sensor              │
│     เช่น: Limit Switch, Proximity Sensor                 │
│     ข้อดี: ตรงตามความเป็นจริง ไม่ขึ้นกับเวลา            │
│                                                          │
│  2. TIMER-BASED: ใช้เวลา                                │
│     เช่น: หลังจากทำงาน X วินาที                         │
│     ข้อดี: ง่าย ข้อเสีย: ไม่รู้ว่าเสร็จจริงหรือเปล่า    │
│                                                          │
│  3. COMBINED: Sensor AND Timer                          │
│     เช่น: (Level_Sensor OR Fill_Timeout)                 │
│     ดีที่สุด: ทำงานเมื่อเสร็จ หรือ Timeout เป็น Alarm  │
│                                                          │
│  4. COUNT-BASED: นับจำนวน                               │
│     เช่น: เมื่อผลิตครบ 100 ชิ้น                         │
│                                                          │
│  5. CALCULATED: คำนวณจากค่า Analog                      │
│     เช่น: เมื่อน้ำหนักถึง 50kg                           │
└──────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Combined Transition (Siemens):

```
// เงื่อนไข: น้ำเต็ม (Level_High) หรือ Timeout 30 วินาที

NETWORK 5:
    ┌──[CMP==]──────────────────────────────────────────┐
    │ Current_Step == 2                                  │
    │ (FILL_WATER)                                       │
    └──┬────────────────────────────────────────────────┘
       │
       ├──[Level_High]──────────────────────────(MOVE)──┐
       │                                        IN: 3   │
       │                                OUT: Current_Step│
       │                                               └┘
       │
       └──[Fill_Timeout.Q]──────────────────────────────┤
                                              ┌─(MOVE)──┐
                                              │  IN: 3  │
                                              │OUT: Curr │
                                              └─────────┘
                                              ┌─( )─────┐
                                              │ Alarm_Fill│
                                              │ _Timeout  │
                                              └─────────┘
```

---

## Step 401: Parallel Sequences (กระบวนการขนาน)

บางครั้งต้องการให้หลาย Sequence ทำงานพร้อมกัน:

```
┌──────────────────────────────────────────────────────────┐
│              Parallel Sequence Example                    │
│           (ไลน์ผลิต 2 สาย ทำงานพร้อมกัน)                │
│                                                          │
│                  ┌────────────┐                          │
│                  │  STEP 0   │                          │
│                  │   IDLE    │                          │
│                  └─────┬─────┘                          │
│                        │  [Start]                        │
│              ══════════╪══════════                       │
│             ╔══════════╧══════════╗                      │
│        ┌────╨────┐         ┌──────╨──┐                  │
│        │ Step 1A │         │ Step 1B │                  │
│        │ Line A  │         │ Line B  │                  │
│        └────┬────┘         └────┬────┘                  │
│             │                   │                        │
│        ┌────╨────┐         ┌────╨────┐                  │
│        │ Step 2A │         │ Step 2B │                  │
│        └────┬────┘         └────┬────┘                  │
│             │                   │                        │
│             ╚══════════╤════════╝                        │
│                        │  [Line A Done AND Line B Done]  │
│                  ┌─────╨─────┐                          │
│                  │  STEP 3   │                          │
│                  │   DONE    │                          │
│                  └───────────┘                          │
└──────────────────────────────────────────────────────────┘
```

### Implementation ใน Ladder:

```
// ใช้ 2 State Registers: StateA, StateB
// เริ่มพร้อมกัน แล้ว Synchronize ที่ Step สุดท้าย

RUNG 1: // เริ่ม Parallel ทั้งสอง
  ┌─[Main_State EQU 0]──[Start_PB]──────────────────────┐
  │                               ┌─(MOV 1)→StateA      │
  │                               ├─(MOV 1)→StateB      │
  │                               └─(MOV 1)→Main_State  │
  └──────────────────────────────────────────────────────┘

RUNG 2: // Line A - Step 1A
  ┌─[StateA EQU 1]────────────────────────────( )───────┐
  │                                       Conveyor_A_ON  │
  └──────────────────────────────────────────────────────┘

RUNG 3: // Line A - Step 1A → 2A
  ┌─[StateA EQU 1]──[Sensor_A]────────────(MOV 2)───────┐
  │                                           → StateA   │
  └──────────────────────────────────────────────────────┘

RUNG 4: // Line B - Step 1B  
  ┌─[StateB EQU 1]────────────────────────────( )───────┐
  │                                       Conveyor_B_ON  │
  └──────────────────────────────────────────────────────┘

RUNG 5: // Synchronize: เมื่อทั้งสอง Line เสร็จ
  ┌─[StateA EQU 2]──[StateB EQU 2]────────(MOV 2)───────┐
  │                                       → Main_State   │
  └──────────────────────────────────────────────────────┘
```

---

## Step 402: Sequencer ใน Allen-Bradley (SQO/SQI/SQL)

Allen-Bradley มี Instruction พิเศษสำหรับ Sequencer:

```
// SQO - Sequencer Output
// SQI - Sequencer Input  
// SQL - Sequencer Load

// ┌─────────────────────────────────────────────────────┐
// │  SQO (Sequencer Output)                             │
// │  File:    #N7:0  (ข้อมูล Output แต่ละ Step)        │
// │  Mask:    0x00FF (กำหนดว่า Bit ไหนใช้งาน)           │
// │  Dest:    O:0.0  (Output Word)                      │
// │  Control: R6:0   (Control Register)                 │
// │  Length:  8      (จำนวน Step)                       │
// │  Position: 0     (ตำแหน่งปัจจุบัน)                 │
// └─────────────────────────────────────────────────────┘

// N7:0 ถึง N7:7 เก็บ Output Pattern แต่ละ Step:
// N7:0 = 00000001 → เฉพาะ Output 0 ทำงาน
// N7:1 = 00000011 → Output 0 และ 1 ทำงาน
// N7:2 = 00000110 → Output 1 และ 2 ทำงาน
// N7:3 = 00001100 → Output 2 และ 3 ทำงาน
// ...และอื่นๆ ตามการออกแบบ

RUNG 1:
    ┌──[Step_Clock]──────────────────(SQO)─────────────┐
    │  (Pulse ทุกรอบที่ต้องการเปลี่ยน)  File: #N7:0    │
    │                                   Mask: FFFFh     │
    │                                   Dest: O:0.0     │
    │                                   Control: R6:0   │
    │                                   Length: 8       │
    │                                   Position: 0     │
    └──────────────────────────────────────────────────┘
```

---

## Step 403: Error Handling ใน Sequencer

ต้องมี Error Handling เสมอ:

```
// Emergency Stop และ Error Recovery

RUNG E-STOP: // Emergency Stop ทุกที่
    ┌──[E_STOP_PB]─────────────────────────(MOV 99)────┐
    │  Emergency Button                  → Current_Step │
    │  (Normally Closed ต่อเป็น NC)                     │
    └──────────────────────────────────────────────────┘

RUNG ERROR_STATE: // Error State 99
    ┌──[CMP== 99]───────────────────────────────────────┐
    │  Current_Step                                      │
    └──┬────────────────────────────────────────────────┘
       │
       ├──────────────────────────────────( )──── (Output Off All)
       ├──────────────────────────────────( )──── (Alarm_Light ON)
       └──────────────────────────────────( )──── (Alarm_Horn ON)

RUNG RESET: // Reset จาก Error
    ┌──[CMP== 99]──┬──[Reset_PB]────────────(MOV 0)────┐
    │              │                       → Current_Step│
    └──────────────┘                                   └┘
```

---

## Step 404: ตัวอย่างสมบูรณ์ - Conveyor System

```
// ระบบสายพานลำเลียงที่มีสถานี 4 จุด
// Station A → Station B → Station C → Station D

//===========================================================
// SYMBOL TABLE:
// D100 = State_Register
// X0 = Start_PB
// X1 = Stop_PB  
// X2 = E_Stop (NC)
// X3 = Sensor_A (ชิ้นงานถึง Station A)
// X4 = Sensor_B (ชิ้นงานถึง Station B)
// X5 = Sensor_C (ชิ้นงานถึง Station C)
// X6 = Sensor_D (ชิ้นงานถึง Station D)
// X7 = Op_A_Complete (งาน Station A เสร็จ)
// X8 = Op_B_Complete (งาน Station B เสร็จ)
// X9 = Op_C_Complete (งาน Station C เสร็จ)
// Y0 = Conveyor_Motor
// Y1 = Station_A_Actuator
// Y2 = Station_B_Actuator
// Y3 = Station_C_Actuator
// Y4 = Discharge_Pusher
// Y5 = System_Ready_Light
// Y6 = Run_Light
// Y7 = Alarm_Light
//===========================================================

[Ladder Diagram - Mitsubishi MELSEC]

// N1: Initialize
LD   M8002          // Initial Pulse
MOV  K0  D100       // State = 0

// N2: State 0 - Ready/Idle
LD   CMP= K0 D100   // State == 0 ?
AND  X0             // Start Button?
MOV  K1  D100       // → State 1

// N3: State 1 - Conveyor Running
LD   CMP= K1 D100   // State == 1?
OUT  Y0             // Conveyor ON
LD   X3             // Sensor A?
AND  CMP= K1 D100
MOV  K2  D100       // → State 2

// N4: State 2 - Process at Station A
LD   CMP= K2 D100   // State == 2?
OUT  Y1             // Station A Actuator
OUT  Y0             // Conveyor OFF (ไม่ต้องการ Conveyor ขณะทำงาน)
// แก้ไข: Y0 ต้องดับ
ANI  CMP= K2 D100   // NOT State 2
OUT  Y0             // Conveyor OFF when State 2

// ใช้ SET/RST แทน:
LD   CMP= K2 D100
RST  Y0             // Stop Conveyor
SET  Y1             // Start Station A

// N5: State 2 → 3: Operation A เสร็จ
LD   CMP= K2 D100
AND  X7             // Op_A_Complete
MOV  K3  D100       // → State 3
RST  Y1             // Stop Station A
```

---

## Step 405: Best Practices สำหรับ Sequencer

```
┌──────────────────────────────────────────────────────────┐
│             Sequencer Best Practices                      │
│                                                          │
│  1. ใช้ Meaningful Step Numbers                          │
│     ✓ Step 0 = IDLE                                     │
│     ✓ Step 1 = START_UP                                 │
│     ✗ Step 1 = Step 1                                  │
│                                                          │
│  2. เสมอมี Initial/Idle State                          │
│     ระบบต้องเริ่มต้นที่ State ที่ปลอดภัย                │
│                                                          │
│  3. มี Error/Fault State                                │
│     State 99 หรือ 255 = Error State                     │
│                                                          │
│  4. มี Manual/Auto Mode                                 │
│     Auto = ลำดับอัตโนมัติ                               │
│     Manual = Control แต่ละ Output โดยตรง               │
│                                                          │
│  5. Log State Changes                                   │
│     เก็บประวัติ State เพื่อ Debug                       │
│                                                          │
│  6. Timeout ทุก State                                   │
│     หาก State อยู่นานเกิน X วินาที = Alarm             │
│                                                          │
│  7. Comment ทุก Rung                                    │
│     บอกว่า Rung ทำอะไร State ไหน → ไหน                 │
└──────────────────────────────────────────────────────────┘
```

---

## Step 406: ตัวอย่าง Auto/Manual Mode Selector

```
// การสลับระหว่าง Auto และ Manual Mode

// Tag: Mode_Switch (BOOL) - 0=Manual, 1=Auto
// Tag: Auto_State (INT)
// Tag: Manual_xxx = Manual Override แต่ละตัว

RUNG 1: // Auto Mode - ลำดับอัตโนมัติ
    ┌──[Mode_Switch]────────────────────────────────────┐
    │  (Auto Mode)                                       │
    └──┬────────────────────────────────────────────────┘
       ├── [Sequence Logic ตามปกติ]
       └── [Output Assignments from State]

RUNG 2: // Manual Mode - ควบคุมโดยตรง
    ┌──[Mode_Switch]/────────────────────────────────────┐
    │  (NOT Auto = Manual Mode)                          │
    └──┬─────────────────────────────────────────────────┘
       │
       ├──[Manual_Motor1_PB]─────────────────────( )────┐
       │                                        Motor_1  │
       └─────────────────────────────────────────────────┘
       │
       ├──[Manual_Motor2_PB]─────────────────────( )────┐
       │                                        Motor_2  │
       └─────────────────────────────────────────────────┘

RUNG 3: // Safety: เมื่อสลับ Mode ต้อง Reset State
    ┌──[Mode_Switch RTO]────────────────────────────────┐
    │  (Rising Edge of Mode Switch)                      │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(MOV 0)───┐
                                              → Auto_State│
                                                        └┘
```

---

## Step 407: Step Timeout Monitoring

```
// ระบบ Monitor ว่าแต่ละ Step ทำงานนานเกินไปหรือเปล่า

// Data:
// Step_Entry_Time (DINT) - เวลาที่เข้า Step
// Step_Timeout[0..9] (DINT Array) - Timeout แต่ละ Step เป็น ms
// Step_Alarm (BOOL)

RUNG 1: // เมื่อ State เปลี่ยน บันทึกเวลา
    ┌──[State_Changed]──────────────────────────────────┐
    │  (Rising Edge)                                     │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(MOV)──────┐
                                           IN: SYS_TIME  │
                                     OUT: Step_Entry_Time│
                                                        └┘

RUNG 2: // ตรวจสอบ Timeout ทุก Scan
    ┌──[CMP >]──────────────────────────────────────────┐
    │  Current_Time - Step_Entry_Time                    │
    │  > Step_Timeout[Current_State]                     │
    └──┬─────────────────────────────────────────────────┘
       │
       ├──────────────────────────────────────( )────────┐
       │                                   Step_Alarm    │
       └─────────────────────────────────────────────────┘
       │
       └──────────────────────────────────────(MOV 99)──┐
                                            → State     │
                                                        └┘
```

---

## Step 408: การ Debug Sequencer

เทคนิคการ Debug:

```
┌──────────────────────────────────────────────────────────┐
│                Debug Techniques                           │
│                                                          │
│  1. HMI Display แสดง Current State                      │
│     - ชื่อ State เป็น Text                              │
│     - ระยะเวลาใน State นั้น                             │
│     - Previous State                                    │
│                                                          │
│  2. State History Buffer                                │
│     เก็บ 10 State ล่าสุดใน Array                        │
│     เมื่อ State เปลี่ยน → Shift Array แล้วเก็บ State ใหม │
│                                                          │
│  3. Force State (สำหรับ Debug เท่านั้น)                 │
│     ✓ ใช้ HMI ส่งค่า State โดยตรง                       │
│     ✗ อย่าใช้ใน Production                             │
│                                                          │
│  4. Status Word                                         │
│     Bit 0 = System Running                              │
│     Bit 1 = In Auto Mode                               │
│     Bit 2 = Fault                                       │
│     Bit 3-7 = Current State                             │
└──────────────────────────────────────────────────────────┘
```

---

## Step 409: การเปรียบเทียบ Methods

| วิธี | ข้อดี | ข้อเสีย | เหมาะกับ |
|------|--------|---------|---------|
| Integer State | ง่าย ใช้ Register เดียว | เข้าใจยากเมื่อไม่มี Comment | ระบบทั่วไป |
| Bit-per-Step | มองเห็น Step Active ชัด | ใช้ Bit หลายตัว | < 10 Steps |
| SFC (IEC) | มาตรฐาน อ่านง่าย | ต้องการ Software รองรับ | Siemens S7 |
| SQO/SQI | AB มี Built-in | เฉพาะ AB | Allen-Bradley |
| STL | ยืดหยุ่นมาก | ซับซ้อน | Experienced |

---

## Step 410: Lab Project - เครื่องผสมสารเคมี

**โจทย์:** เขียน Sequencer สำหรับเครื่องผสมสารเคมี

```
┌──────────────────────────────────────────────────────────┐
│         Chemical Mixing Machine - Lab Project            │
│                                                          │
│  อุปกรณ์:                                               │
│  - Valve A: เติมสาร A                                    │
│  - Valve B: เติมสาร B                                    │
│  - Heater: ให้ความร้อน                                   │
│  - Mixer Motor: กวนสาร                                   │
│  - Drain Valve: ระบายของเหลว                            │
│  - Level Sensor: ตรวจจับระดับ (Low/High)                │
│  - Temp Sensor: วัดอุณหภูมิ                              │
│                                                          │
│  Sequence:                                              │
│  1. รอ Start Command                                     │
│  2. เปิด Valve A จนถึง Level High → ปิด Valve A        │
│  3. เปิด Valve B 5 วินาที                               │
│  4. เปิด Heater จนอุณหภูมิ 80°C                        │
│  5. เปิด Mixer 10 นาที                                  │
│  6. ปิด Heater รอ Cool Down 70°C                        │
│  7. เปิด Drain Valve จน Level Low                      │
│  8. กลับ Idle                                           │
│                                                          │
│  เพิ่มเติม:                                             │
│  - E-Stop ทุก State                                     │
│  - Timeout 30 วินาทีต่อ Step (ยกเว้น Step 5)           │
│  - Manual Override ทุก Output                           │
│  - HMI Display State และ Alarm                         │
└──────────────────────────────────────────────────────────┘
```

**แนวทางการทำ Lab:**

```
// Step 1: กำหนด I/O
// Step 2: วาด State Diagram
// Step 3: เขียน Transition Table
// Step 4: เขียน Ladder (เลือก Platform ที่มี)
// Step 5: Simulate ใน Software
// Step 6: ทดสอบ E-Stop และ Timeout
// Step 7: เพิ่ม Manual Mode
// Step 8: สร้าง HMI Screen
```

---

## สรุป Part 21

ใน Part นี้ได้เรียนรู้:

1. **SFC Concept** - โครงสร้าง Step/Transition/Action
2. **State Machine** - แนวคิดการจัดการ State
3. **Implementation** - วิธีทำใน Ladder (Integer, Bit, SQO)
4. **Real Examples** - เครื่องซักผ้า, เจาะ, บรรจุภัณฑ์
5. **Timer Integration** - ใช้ Timer ภายใน State
6. **Transition Conditions** - เงื่อนไขที่ดี
7. **Parallel Sequences** - หลาย Sequence พร้อมกัน
8. **Error Handling** - E-Stop, Timeout, Recovery
9. **Auto/Manual Mode** - การสลับโหมด
10. **Debug Techniques** - วิธี Debug Sequencer

---

## Navigation
- [← Part 20: Advanced Timer และ Counter](../part01-20/part20_advanced_timer_counter.md)
- [→ Part 22: Analog I/O และ Scaling](part22_analog_io.md)
- [กลับหน้าหลัก](../README.md)
