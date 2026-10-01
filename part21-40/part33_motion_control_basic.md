# Part 33: Motion Control พื้นฐาน
## Steps 641-660: การควบคุมการเคลื่อนที่ด้วย PLC

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 4-5 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 22 (Analog I/O), Part 32 (Interrupt Routines)

---

## บทนำ

Motion Control คือการควบคุมการเคลื่อนที่ของเครื่องจักรอย่างแม่นยำ ครอบคลุมการควบคุม:
- **ตำแหน่ง (Position):** เคลื่อนไปยังจุดที่กำหนดอย่างแม่นยำ
- **ความเร็ว (Velocity):** รักษาความเร็วให้คงที่
- **แรงบิด (Torque):** ควบคุมแรงที่ใช้

การใช้งานในอุตสาหกรรม: CNC Machines, Robots, Conveyor Systems, Pick & Place

---

## Step 641: Motion Control Concepts

### แนวคิดพื้นฐาน

```
┌─────────────────────────────────────────────────────────┐
│               Motion Control Terminology                 │
│                                                         │
│  Position (ตำแหน่ง)                                    │
│  ├── Absolute: วัดจากจุดอ้างอิง (Home Position)        │
│  └── Relative: วัดจากตำแหน่งปัจจุบัน                   │
│                                                         │
│  Velocity (ความเร็ว)                                    │
│  ├── Units: mm/s, rpm, pulse/sec                        │
│  └── Max Velocity: ขึ้นกับ Motor + Load                 │
│                                                         │
│  Acceleration (ความเร่ง)                                │
│  ├── Ramp Up: เพิ่มความเร็วจากหยุด                     │
│  └── Ramp Down: ลดความเร็วก่อนหยุด                     │
│                                                         │
│  Jerk (อัตราการเปลี่ยนแปลงความเร่ง)                    │
│  └── ควบคุมเพื่อลด Vibration                           │
└─────────────────────────────────────────────────────────┘
```

### Velocity Profile (Trapezoidal)

```
Velocity
    │
Vmax├─────────────────────┐
    │                     │
    │    /‾‾‾‾‾‾‾‾‾‾‾‾‾\ │
    │   /               │ \
    │  /                │  \
    │ /  Accel Phase    │   \ Decel Phase
    │/                  │    \
    └──────────────────────────→ Time
    Start    Constant    End
             Velocity

ระยะ Accel = V²/(2×a)
ระยะ Decel = V²/(2×d)
ระยะรวม = Accel + Constant + Decel
```

### S-Curve Profile (Jerk Control)

```
Velocity
    │
Vmax├────────────────────────┐
    │                        │
    │   ╭──────────────────╮ │
    │  /                    \ │
    │ /     S-curve Ramp     \│
    │/                        \
    └───────────────────────────→ Time

ข้อดี: ลด Vibration และ Mechanical Stress
ข้อเสีย: ต้องการ Jerk parameter เพิ่มเติม
```

---

## Step 642: Encoder Types

### Incremental Encoder

```
┌─────────────────────────────────────────────────────────┐
│               Incremental Encoder Output                 │
│                                                         │
│  Channel A: ─┐  ┌──┐  ┌──┐  ┌──┐  ┌──                 │
│              └──┘  └──┘  └──┘  └──┘                    │
│                                                         │
│  Channel B: ──┐  ┌──┐  ┌──┐  ┌──┐  ┌─                 │
│               └──┘  └──┘  └──┘  └──┘                   │
│                                                         │
│  Index (Z): ──────────────────┐  └──────────────────── │
│              (1 pulse/rev)    │                         │
│                                                         │
│  Quadrature Decoding:                                   │
│  A leads B → Forward                                    │
│  B leads A → Reverse                                    │
│                                                         │
│  ข้อดี: ราคาถูก, ง่าย                                  │
│  ข้อเสีย: ต้อง Home เมื่อ Power Off                    │
└─────────────────────────────────────────────────────────┘
```

### Absolute Encoder

```
┌─────────────────────────────────────────────────────────┐
│               Absolute Encoder                          │
│                                                         │
│  Single-turn: บอกตำแหน่งใน 1 รอบ (0-4095 for 12-bit)  │
│  Multi-turn:  บอกตำแหน่งหลายรอบ (SSI/EnDat/BiSS)      │
│                                                         │
│  Communication Interfaces:                              │
│  ├── SSI (Synchronous Serial Interface) - ดั้งเดิม     │
│  ├── EnDat 2.2 - Heidenhain                            │
│  ├── BiSS-C - Open Standard                             │
│  └── HIPERFACE DSL - Sick                              │
│                                                         │
│  ข้อดี: ไม่ต้อง Home, ทราบตำแหน่งทันทีเมื่อเปิด       │
│  ข้อเสีย: ราคาแพงกว่า, ต้องใช้ Interface Module       │
└─────────────────────────────────────────────────────────┘
```

### Resolver

```
Resolver - สำหรับสภาพแวดล้อม Harsh

ข้อดี:
- ทนร้อน (ถึง 150°C)
- ทนการสั่น
- ทนน้ำมัน/ฝุ่น

ข้อเสีย:
- ต้องการ R/D Converter (Resolver to Digital)
- Resolution ต่ำกว่า Encoder
- ราคาแพงกว่า

ใช้ใน: Heavy Industry, Steel Mill, Foundry
```

---

## Step 643: Pulse Output จาก PLC

### ประเภท Pulse Output

```
1. PTO (Pulse Train Output) - ใน S7-1200/1500
   - ความถี่สูงสุด: 100 kHz (Standard CPU)
   - เหมาะสำหรับ Stepper Motor
   - ควบคุมด้วย Motion Instructions

2. PWM (Pulse Width Modulation)
   - ใช้ควบคุม Analog ด้วย Digital Output
   - ตัวอย่าง: ควบคุมความสว่าง LED, Speed ของ DC Motor

3. High-Speed Counter (HSC)
   - รับสัญญาณ Encoder ความเร็วสูง
   - นับ Pulse จาก Encoder
```

### S7-1200 Pulse Output Configuration

```
TIA Portal Configuration:
1. CPU Properties → Pulse generators (PTO/PWM)
2. เลือก Pulse generator 1
3. กำหนด:
   ├── Signal type: PTO
   ├── Hardware output: Q0.0
   ├── Direction: Q0.1
   └── Max frequency: 100 kHz
```

### Ladder - Pulse Output สำหรับ Stepper Motor

```
// ใช้ Technology Object หรือ Motion Control Instructions

Network 1: ตั้งค่า Position และ Velocity

  |                                                    |
  |--[P]----------[MC_MoveAbsolute]-------------------|
  |  "Start_Move"   Axis: "Stepper_Axis"              |
  |                 Execute: TRUE                      |
  |                 Position: 1000.0  (mm)             |
  |                 Velocity: 50.0    (mm/s)           |
  |                 Direction: 0 (Positive)            |
  |                 Done: "Move_Complete"              |
  |                 Error: "Move_Error"                |
  |                                                    |

Network 2: ตรวจสอบการเสร็จสิ้น

  |                                                    |
  |--[ ]---------[SET]--------------------------------|
  |  "Move_      "Axis_At_Position"                   |
  |   Complete"                                        |
  |                                                    |
  |--[ ]---------[SET]--------------------------------|
  |  "Move_      "Axis_Error_Alarm"                   |
  |   Error"                                          |
  |                                                    |
```

---

## Step 644: Stepper Motor Control

### Stepper Motor พื้นฐาน

```
┌─────────────────────────────────────────────────────────┐
│               Stepper Motor Basics                      │
│                                                         │
│  Step Motor Types:                                      │
│  ├── Permanent Magnet (PM): ราคาถูก, Resolution ต่ำ    │
│  ├── Variable Reluctance (VR): ไม่มี Detent Torque     │
│  └── Hybrid: ดีที่สุด, ใช้กันมากที่สุด                  │
│                                                         │
│  Resolution:                                            │
│  Full Step: 200 steps/rev = 1.8°/step                  │
│  Half Step: 400 steps/rev = 0.9°/step                  │
│  Microstepping: 1600, 3200, 6400 steps/rev             │
│                                                         │
│  Typical Stepper Driver:                               │
│  STEP pin  ← Pulse from PLC (each pulse = 1 step)      │
│  DIR pin   ← Direction signal from PLC                 │
│  ENABLE    ← Enable/Disable driver                     │
│                                                         │
│  ข้อดี: ราคาถูก, ไม่ต้องการ Feedback                  │
│  ข้อเสีย: สูญเสีย Step เมื่อ Overload                 │
└─────────────────────────────────────────────────────────┘
```

### ตัวอย่าง: Stepper Motor Control Program (Siemens S7-1200)

```
// Technology Object "Axis_1" Configuration:
// Type: Positioning axis
// Drive: PTO (Pulse Train Output)
// Pulses/rev: 1000 (Microstepping ×5)
// Distance/rev: 5mm (Lead Screw Pitch)

Network 1: Initialize Axis

  |                                                    |
  |--[P]----------[MC_Power]--------------------------|
  |  "Init"         Axis: "Axis_1"                    |
  |                 Enable: TRUE                       |
  |                 StartMode: 0 (Speed Control)       |
  |                 StopMode: 0 (Ramp)                 |
  |                 Status: "Axis_Enabled"             |
  |                 Error: "Axis_Power_Error"          |
  |                                                    |

Network 2: Homing Sequence

  |                                                    |
  |--[ ]----------[MC_Home]---------------------------|
  |  "Axis_Enabled"  Axis: "Axis_1"                   |
  |  AND             Execute: "Start_Home"             |
  |  "Start_Home"    Mode: 3 (Active Homing)           |
  |                  Position: 0.0 (Home = 0)          |
  |                  Done: "Home_Complete"             |
  |                  Error: "Home_Error"               |
  |                                                    |

Network 3: Move to Position

  |                                                    |
  |--[ ]--[ ]-----[MC_MoveAbsolute]-------------------|
  |  Home_  Start  Axis: "Axis_1"                     |
  |  Done   Move   Execute: TRUE                       |
  |                Position: "Target_Pos"              |
  |                Velocity: "Move_Speed"              |
  |                Done: "At_Target"                   |
  |                                                    |
```

### Allen-Bradley Stepper Control

```
// Allen-Bradley ControlLogix + Kinetix 300 (Stepper Drive)

// Tag Configuration:
Axis_1 : AXIS_SERVO_DRIVE

// Motion Instructions ใน Studio 5000:
Rung 1: [MSO] Axis_1         // Motion Servo On
Rung 2: [MAS] Axis_1 Home    // Motion Axis Set - Home
Rung 3: [MAM] Axis_1         // Motion Axis Move
  Move Type: Absolute
  Position: Target_Pos
  Speed: Move_Speed
  Accel Rate: 100.0
  Decel Rate: 100.0

// AB ใช้ GSV/SSV สำหรับอ่าน/เขียน Axis Attributes
[GSV] Axis_1 ActualPosition Axis1_Position
[GSV] Axis_1 ActualVelocity Axis1_Velocity
```

### Mitsubishi Stepper Control (iQ-R + RD75)

```
// Mitsubishi RD75 Positioning Module
// Connection: PLC ← [CPU Bus] → RD75 → Stepper Driver

// Basic Setup Parameters (via Buffer Memory):
// Un\G1 = 1: Start Bit
// Un\G4 = Positioning Data No.
// Un\G9 = Current Value (×10^-1 μm)

// Ladder Program:
[TO K0 K0 H0000 K1]  // Clear start bit
[MOV K1000 D100]     // Target: 1000 (0.1mm unit)
[TO K0 K600 D100 K1] // Write target position
[TO K0 K0 H0001 K1]  // Start bit ON
```

---

## Step 645: Positioning using Pulse Counter

### High-Speed Counter (HSC) สำหรับ Encoder

```
S7-1200 High-Speed Counter Configuration:
1. CPU Properties → High speed counters
2. เลือก HSC1
3. กำหนด:
   ├── Count Type: Single phase / AB (Quadrature)
   ├── Initial value: 0
   ├── Reference value: 0
   ├── Count direction: Forward/Backward
   └── Hardware input: I0.0 (A), I0.1 (B), I0.3 (Z)
```

### Ladder - Position Control with HSC

```
// ใช้ HSC เพื่อติดตามตำแหน่งของ Conveyor

Network 1: อ่านค่า Position จาก HSC

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: HC1 (High-Speed Counter 1)     |
  |                 OUT: "Current_Position_Pulse"      |
  |                                                    |

Network 2: แปลง Pulse เป็น mm

  |                                                    |
  |--[ALWAYS]------[DIV]------------------------------|
  |                 IN1: "Current_Position_Pulse"      |
  |                 IN2: 400 (pulses/mm)               |
  |                 OUT: "Current_Position_mm"         |
  |                                                    |

Network 3: เปรียบเทียบกับ Target Position

  |                                                    |
  |--[>=]----------[RESET]---------------------------|
  |  "Current_Pos"  "Motor_Forward" (ปิดมอเตอร์)      |
  |  >= "Target_Pos"                                   |
  |                                                    |

Network 4: Deceleration Zone (50mm ก่อนถึง Target)

  |                                                    |
  |--[>=]--[<]-----[MOVE]------------------------------|
  |  "Curr" >=     IN: "Slow_Speed" (ความเร็วต่ำ)     |
  |  Target-50                                         |
  |  "Curr" <      OUT: "Motor_Speed_Reference"        |
  |  Target                                            |
  |                                                    |
```

### ตัวอย่างสมบูรณ์: Linear Axis Control

```
// ระบบ: Linear Conveyor ด้วย Stepper Motor + Encoder

// I/O Assignment:
// Q0.0: Step Pulse Output
// Q0.1: Direction (0=Forward, 1=Backward)
// Q0.2: Motor Enable
// I0.0: Encoder A
// I0.1: Encoder B
// I0.2: Home Sensor (Inductive Proximity)
// I0.3: Forward Limit Switch
// I0.4: Reverse Limit Switch

// DB "Axis_Data":
Position_Actual : DINT    // ตำแหน่งจริง (pulses)
Position_Target : DINT    // ตำแหน่งเป้าหมาย (pulses)
Velocity_Current: REAL    // ความเร็วปัจจุบัน (Hz)
Velocity_Max    : REAL    // ความเร็วสูงสุด (Hz)
Accel_Rate      : REAL    // อัตราเร่ง (Hz/s)
Is_Homed        : BOOL    // ได้ Home แล้ว
At_Target       : BOOL    // ถึง Target แล้ว
Error_Code      : INT     // รหัสข้อผิดพลาด

// State Machine:
0 = IDLE
1 = HOMING
2 = AT_HOME
3 = MOVING
4 = AT_TARGET
5 = ERROR

Network 1: Error Check (สูงสุด Priority)

  |  ─────────────────────────────────────────────────  |
  |--[ ]----------[MOVE]--[MOVE]-----------------------|
  |  "Fwd_Limit"   IN:5    IN:0                        |
  |  OR            OUT:State OUT:All_Outputs            |
  |  "Rev_Limit"                                       |
  |                                                    |

Network 2: Homing State

  |                                                    |
  |--[=]--[ ]-----[SET]-------------------------------|
  |  State  Start  Q0.1=0 (Forward direction)         |
  |  = 0    Home   Q0.2=1 (Enable motor)              |
  |                "FC_Generate_Pulse"                 |
  |                 Speed: Slow_Home_Speed             |
  |                                                    |
  |--[=]--[ ]-----[MOVE]------------------------------|
  |  State  Home   IN: 1 (Homing state)               |
  |  = 0    Switch OUT: State                          |
  |                                                    |

Network 3: Home Detected

  |                                                    |
  |--[=]--[ ]-----[MOVE]--[MOVE]----------------------|
  |  State  Home   IN: 2   IN: 0 (Reset position)     |
  |  = 1    Sensor OUT:State OUT: HC1 (Clear counter)  |
  |  AND                                               |
  |  Home_Sensor                                       |
  |                                                    |
```

---

## Step 646: Siemens S7-1200 Technology Objects

### Technology Object (TO) Configuration

```
ใน TIA Portal สำหรับ S7-1200:

1. Project Tree → Technology Objects
2. Add New Object: Axis
3. กำหนดพารามิเตอร์:
```

```
TO Configuration:
┌─────────────────────────────────────────────────────┐
│ Axis Name: X_Axis                                   │
│                                                     │
│ Drive type: PTO (via PLC output)                   │
│ PTO: PTO1                                           │
│ Direction: Q0.1                                     │
│                                                     │
│ Mechanics:                                          │
│ ├── Lead screw pitch: 5mm/rev                      │
│ ├── Gear ratio: 1:1                                 │
│ └── Pulses/rev: 1000                               │
│                                                     │
│ Dynamic limits:                                     │
│ ├── Max velocity: 100 mm/s                          │
│ ├── Max acceleration: 500 mm/s²                    │
│ ├── Max deceleration: 500 mm/s²                    │
│ └── Max jerk: 1000 mm/s³                           │
│                                                     │
│ Homing:                                             │
│ ├── Mode: Active (use limit switch)                │
│ ├── Approach speed: 5 mm/s                         │
│ └── Home position value: 0.0                       │
└─────────────────────────────────────────────────────┘
```

### Motion Control Instructions

```
MC_Power     - เปิด/ปิด Axis
MC_Reset     - Reset Error
MC_Home      - Homing Sequence
MC_Halt      - หยุดชั่วคราว
MC_MoveAbsolute - เคลื่อนที่ไปตำแหน่ง Absolute
MC_MoveRelative - เคลื่อนที่แบบ Relative
MC_MoveVelocity - ควบคุมความเร็ว
MC_MoveJog   - Jog Mode (กดค้างเคลื่อนที่)
MC_ReadActualPosition - อ่านตำแหน่งจริง
MC_WriteParameter - เขียนพารามิเตอร์
```

### ตัวอย่าง Complete Motion Program

```
// Program สำหรับ Pick & Place Machine
// Axis X: เคลื่อนซ้าย-ขวา
// Axis Z: เคลื่อนขึ้น-ลง

// State Machine สำหรับ Pick and Place:
// 0: Home All
// 1: Move X to Pick Position
// 2: Move Z Down
// 3: Activate Suction Cup (Gripper)
// 4: Move Z Up
// 5: Move X to Place Position
// 6: Move Z Down
// 7: Release Suction Cup
// 8: Move Z Up
// 9: Return to Home / Cycle Complete

Network 1: Homing

  |                                                    |
  |--[=]----------[MC_Home]---------------------------|
  |  State = 0     Axis: X_Axis                       |
  |               Execute: Home_Start                  |
  |               Mode: 3                             |
  |               Done: X_Homed                       |
  |                                                    |
  |--[=]----------[MC_Home]---------------------------|
  |  State = 0     Axis: Z_Axis                       |
  |               Execute: Home_Start                  |
  |               Mode: 3                             |
  |               Done: Z_Homed                       |
  |                                                    |

Network 2: เมื่อ Home เสร็จทั้งคู่

  |                                                    |
  |--[=]--[ ]--[ ]--[MOVE]---------------------------|
  |  State  X_Homed  Z_Homed  IN: 1                  |
  |  = 0                       OUT: State             |
  |                                                    |

Network 3: Move X to Pick

  |                                                    |
  |--[=]----------[MC_MoveAbsolute]-------------------|
  |  State = 1     Axis: X_Axis                       |
  |               Execute: TRUE                        |
  |               Position: "Pick_X_Pos"              |
  |               Velocity: 50.0                      |
  |               Done: X_At_Pick                     |
  |                                                    |
  |--[=]--[ ]-----[MOVE]------------------------------|
  |  State   X_At_  IN: 2                             |
  |  = 1     Pick   OUT: State                        |
  |                                                    |
```

---

## Step 647: Speed Control Basics

### Open-Loop Speed Control

```
// ใช้ Analog Output ควบคุม VFD (Variable Frequency Drive)
// ไม่มี Feedback จาก Motor Speed

Network 1: ตั้ง Speed Reference

  |                                                    |
  |--[ALWAYS]------[SCALE]----------------------------|
  |                 IN: "Speed_Percent" (0-100%)       |
  |                 LO_LIM: 0                          |
  |                 HI_LIM: 27648                      |
  |                 OUT: QW256 (AO to VFD)             |
  |                                                    |
```

### Closed-Loop Speed Control

```
// ใช้ Encoder Feedback เพื่อรักษาความเร็ว
// (Velocity Control Mode)

Network 1: คำนวณความเร็วจาก Encoder (ใน OB35 - 100ms)

  |                                                    |
  |--[ALWAYS]------[SUB]------------------------------|
  |                 IN1: HC1 (Current encoder count)  |
  |                 IN2: "Last_Count"                  |
  |                 OUT: "Delta_Count"                 |
  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |  (Convert to rpm: pulse/0.1s × 600 / PulsesPerRev) |
  |                 IN1: "Delta_Count"                 |
  |                 IN2: 600                           |
  |                 OUT: "Temp_Speed"                  |
  |                                                    |
  |--[ALWAYS]------[DIV]------------------------------|
  |                 IN1: "Temp_Speed"                  |
  |                 IN2: 1000 (Pulses/rev)             |
  |                 OUT: "Actual_RPM"                  |
  |                                                    |

Network 2: PID Speed Control

  |                                                    |
  |--[ALWAYS]------[FB_PID]---------------------------|
  |                 SP: "Speed_Setpoint"               |
  |                 PV: "Actual_RPM"                   |
  |                 Output: "Speed_Output"             |
  |                                                    |

Network 3: เขียน Output ไปยัง VFD

  |                                                    |
  |--[ALWAYS]------[SCALE]----------------------------|
  |                 IN: "Speed_Output"                 |
  |                 LO_LIM: 0                          |
  |                 HI_LIM: 27648                      |
  |                 OUT: QW256                         |
  |                                                    |
```

---

## Step 648: Homing Procedure

### Homing Methods

```
┌─────────────────────────────────────────────────────────┐
│               Homing Methods Comparison                  │
│                                                         │
│  Method 1: Limit Switch Homing                         │
│  ─────────────────────────────                         │
│  1. เคลื่อนที่ไป Home Direction (ช้า)                  │
│  2. ตรวจจับ Limit Switch                               │
│  3. ถอยออก (เร็วกว่า)                                  │
│  4. เคลื่อนกลับ (ช้า) จนเจอ Switch อีกครั้ง            │
│  5. Reset Position Counter = 0                          │
│                                                         │
│  Method 2: Home Sensor + Index Pulse                   │
│  ────────────────────────────────────                   │
│  1. เคลื่อนไปหา Home Sensor                            │
│  2. เมื่อผ่าน Sensor รอ Index Pulse ถัดไป              │
│  3. Index Pulse = ตำแหน่ง Home แน่นอน                  │
│  4. Reset Position = 0                                  │
│                                                         │
│  Method 3: Torque Homing (Stall Detection)             │
│  ──────────────────────────────────────────            │
│  1. เคลื่อนที่ไปจนชนกับ Mechanical Stop               │
│  2. ตรวจสอบ Current Spike (Stall)                     │
│  3. Reset Position = 0                                  │
│  ⚠️ อาจทำให้เครื่องจักรเสียหายถ้าไม่ระวัง            │
└─────────────────────────────────────────────────────────┘
```

### Ladder - Homing Sequence

```
// State Machine สำหรับ Homing (Method 1)

// States:
// 0 = IDLE
// 1 = Moving to Home (Fast)
// 2 = Backing Off
// 3 = Moving to Home (Slow)
// 4 = HOMED

Network 1: เริ่ม Homing

  |                                                    |
  |--[=]--[P]---------[MOVE]--------------------------|
  |  Homing  Start_   IN: 1                           |
  |  State=0  Home     OUT: Homing_State              |
  |                                                    |

Network 2: Phase 1 - เคลื่อนที่เร็ว

  |                                                    |
  |--[=]--[NOT]--[SET]----[MOVE]---------------------|
  |  Homing  Home_SW  Motor_Fwd  IN: Fast_Speed       |
  |  State=1           (Q0.0=1)  OUT: Speed_Ref       |
  |                                                    |

Network 3: ตรวจจับ Home Switch ครั้งแรก

  |                                                    |
  |--[=]--[ ]-----[RESET]-[MOVE]---------------------|
  |  Homing  Home_SW  Motor_Fwd  IN: 2               |
  |  State=1           (Q0.0=0)  OUT: Homing_State   |
  |                                                    |

Network 4: Phase 2 - ถอยออก

  |                                                    |
  |--[=]--[NOT]--[SET]----[MOVE]---------------------|
  |  Homing  Home_SW  Motor_Rev  IN: Back_Speed       |
  |  State=2           (Q0.1=1)  OUT: Speed_Ref       |
  |                                                    |

Network 5: ถอยออกพอแล้ว

  |                                                    |
  |--[=]--[ ]-----[RESET]-[MOVE]---------------------|
  |  Homing  Home_SW  Motor_Rev  IN: 3               |
  |  State=2  = FALSE  (Q0.1=0)  OUT: Homing_State   |
  |  (edge: เมื่อ Switch ปิด)                         |
  |                                                    |

Network 6: Phase 3 - เคลื่อนที่ช้า (Creep Speed)

  |                                                    |
  |--[=]--[NOT]--[SET]----[MOVE]---------------------|
  |  Homing  Home_SW  Motor_Fwd  IN: Creep_Speed     |
  |  State=3           (Q0.0=1)  OUT: Speed_Ref       |
  |                                                    |

Network 7: ตรวจจับ Home Switch ครั้งที่ 2

  |                                                    |
  |--[=]--[ ]-----[RESET]-[MOVE]-[MOVE]-------------|
  |  Homing  Home_SW  Motor_Fwd  IN: 4    IN: 0      |
  |  State=3           (Q0.0=0)  OUT:State OUT: HC1  |
  |                                        (Reset Pos)|
  |                                                    |
```

---

## Step 649-655: Velocity Profile Implementation

### Trapezoidal Velocity Profile

```pascal
// SCL Code สำหรับ Trapezoidal Profile Generator
// ทำงานใน OB35 (100ms)

FUNCTION_BLOCK "FB_Velocity_Profile"
VAR_INPUT
  Target_Velocity  : REAL;     // ความเร็วเป้าหมาย
  Accel_Rate       : REAL;     // อัตราเร่ง (units/s²)
  Decel_Rate       : REAL;     // อัตราหน่วง (units/s²)
  Enable           : BOOL;     // เปิดใช้งาน
END_VAR
VAR_OUTPUT
  Current_Velocity : REAL;     // ความเร็วปัจจุบัน
  At_Target        : BOOL;     // ถึงความเร็วเป้าหมาย
END_VAR
VAR
  Delta_V          : REAL;     // ความแตกต่างความเร็ว
  Sample_Time      : REAL := 0.1; // 100ms = 0.1 second
  Max_Accel        : REAL;     // ความเร่งสูงสุดต่อ Sample
END_VAR

// คำนวณ Max Accel per Sample Time
Max_Accel := Accel_Rate * Sample_Time;

IF Enable THEN
  Delta_V := Target_Velocity - Current_Velocity;
  
  IF ABS(Delta_V) <= Max_Accel THEN
    // ถึง Target ได้ใน Sample นี้
    Current_Velocity := Target_Velocity;
    At_Target := TRUE;
  ELSIF Delta_V > 0 THEN
    // Accelerating
    Current_Velocity := Current_Velocity + Max_Accel;
    At_Target := FALSE;
  ELSE
    // Decelerating
    Current_Velocity := Current_Velocity - Max_Accel;
    At_Target := FALSE;
  END_IF;
ELSE
  // Decelerate to 0
  IF Current_Velocity > Max_Accel THEN
    Current_Velocity := Current_Velocity - Max_Accel;
  ELSE
    Current_Velocity := 0.0;
  END_IF;
  At_Target := (Current_Velocity = 0.0);
END_IF;
```

### ตัวอย่าง Velocity Profile ใน Ladder

```
// Ladder Implementation ของ Ramp Up/Down

Network 1: คำนวณ Speed Error

  |                                                    |
  |--[ALWAYS]------[SUB]------------------------------|
  |                 IN1: "Target_Speed"                |
  |                 IN2: "Current_Speed"               |
  |                 OUT: "Speed_Error"                 |
  |                                                    |

Network 2: ถ้า Error มาก → Ramp Up

  |                                                    |
  |--[>]----------[ADD]------------------------------|
  |  Speed_Error > Accel_Step                          |
  |               IN1: "Current_Speed"                 |
  |               IN2: "Accel_Step" (Speed increment)  |
  |               OUT: "Current_Speed"                 |
  |                                                    |

Network 3: ถ้า Error น้อย → จบ Ramp

  |                                                    |
  |--[<=]---------[MOVE]-----------------------------|
  |  ABS(Error) <= Accel_Step                          |
  |               IN: "Target_Speed"                   |
  |               OUT: "Current_Speed"                 |
  |                                                    |
```

---

## Step 656-660: Lab Projects

### Lab 1: Linear Positioning System

**วัตถุประสงค์:** สร้างระบบ Linear Positioning ด้วย Stepper Motor

**อุปกรณ์:**
- Siemens S7-1200 CPU 1214C
- Stepper Motor 200 steps/rev
- Driver (Step/Dir interface)
- Linear Encoder 0.01mm/pulse
- Home Sensor (Inductive Proximity)
- Limit Switches (2 pcs)

**โจทย์:**
```
1. Implement Homing Sequence
2. เคลื่อนที่ไปตำแหน่งที่กำหนด (±0.1mm accuracy)
3. ทำซ้ำ 100 ครั้งโดยไม่หลุดตำแหน่ง
4. วัด Actual Position ด้วย Encoder
5. แสดงผลบน HMI: Position, Speed, Status
```

### Lab 2: Conveyor Speed Control

**วัตถุประสงค์:** ควบคุมความเร็ว Conveyor ด้วย Encoder Feedback

**โจทย์:**
```
1. ควบคุมความเร็ว 0-100 m/min
2. รักษาความเร็วคงที่ ±2%
3. Soft Start/Stop ด้วย Ramp Profile
4. ตรวจสอบ Slip (ความเร็วต่างจาก Reference > 5%)
5. บันทึก Speed History ด้วย DB
```

### Lab 3: Pick & Place with Two Axes

**วัตถุประสงค์:** ระบบ Pick and Place ด้วย 2 แกน

```
อุปกรณ์:
- X-Axis: Stepper + Encoder
- Z-Axis: Servo (Pulse/Dir or Analog)
- Suction Cup + Solenoid Valve
- 5 Pick Positions + 5 Place Positions

โปรแกรมที่ต้องสร้าง:
1. Homing ทั้ง 2 แกน
2. Pick & Place Sequence (State Machine)
3. Recipe สำหรับ Pick/Place Positions
4. Error Handling (Axis Error, No-Part Detection)
5. Production Counter + Cycle Time
```

---

## สรุป Part 33

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Motion Concepts | Position, Velocity, Acceleration |
| Encoder Types | Incremental, Absolute, Resolver |
| Pulse Output | PTO สำหรับ Stepper Control |
| Stepper Motor | การควบคุมพื้นฐาน, Microstepping |
| Technology Objects | S7-1200 Motion Instructions |
| Velocity Profile | Trapezoidal Profile |
| Homing | Limit Switch Homing Procedure |
| Speed Control | Open-loop & Closed-loop |

---

## การนำทาง

- [← Part 32: Interrupt Routines](part32_interrupt_routines.md)
- [→ Part 34: Servo Drive Control](part34_servo_drive.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 33 of 40 | Steps 641-660*
