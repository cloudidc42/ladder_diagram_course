# Part 34: Servo Drive Control
## Steps 661-680: การควบคุม Servo System ด้วย PLC

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 4-5 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 33 (Motion Control Basic)

---

## บทนำ

Servo System คือระบบควบคุมการเคลื่อนที่ที่มี Feedback Loop ทำให้ได้ความแม่นยำสูง ใช้ในงาน:
- CNC Machining Centers
- Industrial Robots
- Packaging Machines
- Electronic Assembly Equipment
- Printing Machines

---

## Step 661: Servo System Components

### โครงสร้างระบบ Servo

```
┌────────────────────────────────────────────────────────────┐
│                  Servo System Architecture                  │
│                                                            │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌────────┐ │
│  │   PLC   │───→│  Servo  │───→│  Servo  │───→│  Load  │ │
│  │(Motion  │    │Controller│   │  Motor  │    │(Machine│ │
│  │Control) │←───│  (Drive) │←───│+ Encoder│   │  Axis) │ │
│  └─────────┘    └─────────┘    └─────────┘    └────────┘ │
│       ↑               ↑                                    │
│  Position/Velocity Command    Feedback (Position, Current) │
│                                                            │
│  Communication:                                            │
│  ├── Analog ±10V (Speed/Torque Reference)                 │
│  ├── Pulse/Dir (Position Control)                          │
│  ├── PROFIBUS/PROFINET (Siemens)                          │
│  ├── EtherCAT (High-speed fieldbus)                       │
│  └── EtherNet/IP (Allen-Bradley)                          │
└────────────────────────────────────────────────────────────┘
```

### Servo Drive Internal Control Loops

```
┌────────────────────────────────────────────────────────────┐
│              Servo Drive Control Loops                      │
│                                                            │
│  Position Loop (outermost, slowest)                        │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Position CMD → [+] → [Position Controller] → V_CMD  │ │
│  │                 [-]                                   │ │
│  │             Pos. Feedback (Encoder)                   │ │
│  │                                                       │ │
│  │  Velocity Loop (middle)                               │ │
│  │  ┌─────────────────────────────────────────────────┐ │ │
│  │  │  V_CMD → [+] → [Velocity Controller] → I_CMD   │ │ │
│  │  │          [-]                                    │ │ │
│  │  │      Vel. Feedback (Encoder differentiation)   │ │ │
│  │  │                                                 │ │ │
│  │  │  Current Loop (innermost, fastest)              │ │ │
│  │  │  ┌─────────────────────────────────────────┐  │ │ │
│  │  │  │  I_CMD → [+] → [Current Ctrl] → PWM    │  │ │ │
│  │  │  │          [-]                             │  │ │ │
│  │  │  │       Current Feedback (Hall Sensor)    │  │ │ │
│  │  │  └─────────────────────────────────────────┘  │ │ │
│  │  └─────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

---

## Step 662: Servo Motor vs Stepper Motor

### การเปรียบเทียบ

```
┌───────────────────────────────────────────────────────────┐
│            Servo Motor vs Stepper Motor                    │
│                                                           │
│  Feature          │ Servo Motor  │ Stepper Motor          │
│  ─────────────────────────────────────────────────────   │
│  Feedback         │ Required     │ Not required           │
│  Position Accuracy│ ±1 pulse     │ Open loop (can lose)  │
│  Speed Range      │ High (3000+) │ Limited (<1500 RPM)   │
│  Torque at Speed  │ Constant     │ Drops at high speed   │
│  Cost             │ Higher       │ Lower                  │
│  Complexity       │ Higher       │ Lower                  │
│  Response Time    │ Fast         │ Medium                 │
│  Applications     │ High perf.   │ Simple positioning     │
└───────────────────────────────────────────────────────────┘
```

### เมื่อไหรควรเลือก Servo

```
ใช้ Servo เมื่อ:
✓ ต้องการความแม่นยำสูง (< 0.1mm)
✓ ความเร็วสูง (>1000 RPM)
✓ ต้องการ Constant Torque ที่ความเร็วสูง
✓ งาน Dynamic (เปลี่ยนทิศทางบ่อย)
✓ ต้องการ Feedback สำหรับ Quality Control

ใช้ Stepper เมื่อ:
✓ งานง่าย ตำแหน่งน้อย
✓ ความเร็วต่ำ
✓ ราคาสำคัญ
✓ Load คงที่ ไม่เปลี่ยนมาก
✓ ไม่ต้องการ High Dynamics
```

---

## Step 663: Control Modes

### Position Control Mode (PP Mode)

```
// PLC ส่ง Position Command
// Drive รักษาตำแหน่งอย่างแม่นยำ

ใช้งานกับ:
- Pulse/Direction Interface: PLC ส่ง Pulse
- Fieldbus: PLC ส่ง Target Position ผ่าน Network

ข้อดี: PLC ง่าย, ตำแหน่งแม่นยำ
ข้อเสีย: ต้องกำหนด Motion Profile ใน Drive

PLC → [Target Position] → Drive → Motor
Drive รับผิดชอบ: Velocity Profile, Feedback
```

### Speed Control Mode (CSV Mode)

```
// PLC ส่ง Velocity Setpoint
// Drive รักษาความเร็วอย่างแม่นยำ

ใช้งานกับ:
- Analog ±10V: 10V = Max Speed, 0V = Stop
- Fieldbus: ส่ง Speed Reference ผ่าน Network

ข้อดี: ง่ายสำหรับ Continuous Motion
ข้อเสีย: PLC ต้องรับผิดชอบ Position Tracking

PLC → [Speed Reference] → Drive → Motor
PLC รับผิดชอบ: Position Calculation, Sync
```

### Torque Control Mode (CST Mode)

```
// PLC ส่ง Torque Reference
// Drive รักษาแรงบิดคงที่

ใช้งานกับ:
- Material Handling ที่ต้องการ Constant Tension
- Test Equipment
- Winding/Unwinding Systems

PLC → [Torque Reference] → Drive → Motor
```

---

## Step 664: Siemens SINAMICS S120

### โครงสร้าง SINAMICS S120

```
┌────────────────────────────────────────────────────────────┐
│                 SINAMICS S120 System                       │
│                                                            │
│  Control Unit (CU320-2)                                    │
│  ├── PROFINET/PROFIBUS Interface                          │
│  ├── Drive-CLiQ (High-speed internal bus)                 │
│  └── Safety Integrated Functions                          │
│                                                            │
│  Line Module (ALM/SLM)                                     │
│  ├── Active Line Module: Regen capability                 │
│  └── Smart Line Module: Basic, no regen                  │
│                                                            │
│  Motor Modules (Double/Single Motor Module)                │
│  ├── Power Electronics                                    │
│  ├── Current Measurement                                  │
│  └── Motor Protection                                     │
│                                                            │
│  Motors with DRIVE-CLiQ Encoder                           │
│  └── 1FK7, 1FT7 Series Servomotors                       │
│                                                            │
│  Communication to S7-1500:                                │
│  └── PROFINET IRT (Isochronous Real-Time)                 │
└────────────────────────────────────────────────────────────┘
```

### TIA Portal - S120 Configuration

```
1. Add SINAMICS S120 to PROFINET network
2. Configure Communication:
   ├── Telegram type: Standard Telegram 1 (velocity)
   │   หรือ: Standard Telegram 111 (position)
   ├── Cycle time: 4ms (IRT)
   └── I/O addresses

3. STARTER Software สำหรับ Drive Commissioning:
   ├── Motor Data (Nameplate)
   ├── Encoder Setup
   ├── Controller Tuning
   └── Safety Functions

// Drive Telegram 111 (Position Control):
// Status Word (16 bits) ← Drive
// Actual Position (32 bits) ← Drive
// Control Word (16 bits) → Drive
// Target Position (32 bits) → Drive
// Velocity Override (16 bits) → Drive
```

### Ladder - S120 Control via PROFINET

```
// ใช้ SINA_POS Function Block

Network 1: Control S120 Position Mode

  |                                                         |
  |--[ALWAYS]------[SINA_POS]-------------------------------|
  |                 ModePos: 1 (Absolute)                   |
  |                 EnableAxis: "Servo_Enable"              |
  |                 CancelTraversing: FALSE                  |
  |                 StartTraversing: "Start_Move"           |
  |                 Jog1: FALSE                             |
  |                 Jog2: FALSE                             |
  |                 FlyRef: FALSE                           |
  |                 AckError: "Ack_Error"                   |
  |                 Position: "Target_Position" (mm)        |
  |                 Velocity: "Move_Velocity" (rpm)         |
  |                 AxisEnabled: "Servo_Ready"              |
  |                 Lockout: "Drive_Lockout"                |
  |                 Error: "Drive_Error"                    |
  |                 Pos_reached: "At_Position"              |
  |                 ActVelocity: "Actual_RPM"               |
  |                 ActPosition: "Actual_Position"          |
  |                 ActMode: "Current_Mode"                 |
  |                                                         |
```

---

## Step 665: Allen-Bradley Kinetix

### Kinetix Servo Drive Systems

```
Allen-Bradley Kinetix Family:
┌─────────────────────────────────────────────────────────┐
│  Kinetix 5700 │ High performance, EtherNet/IP CIP Motion│
│  Kinetix 5500 │ Integrated Motion, compact              │
│  Kinetix 5300 │ Stand-alone, EtherNet/IP                │
│  Kinetix 300  │ Pulse Train/Analog, simple apps         │
│  Kinetix 6500 │ Older SERCOS III based                  │
└─────────────────────────────────────────────────────────┘
```

### CIP Motion (EtherNet/IP)

```
// Allen-Bradley ใช้ CIP Motion over EtherNet/IP
// ControlLogix + Kinetix 5700

// Axis Tag: MyAxis (AXIS_CIP_DRIVE type)
// MSO - Motion Servo On
// MAFR - Motion Axis Fault Reset
// MAM - Motion Axis Move
// MAS - Motion Axis Stop
// MAJ - Motion Axis Jog
// MCSV - Motion Calculate Slave Values

// Studio 5000 Ladder:
Rung 1: Motion Servo On
[MSO] MyAxis

Rung 2: Homing
[MASR] MyAxis              // Set Home Position
  Mode: 0 (Active mode)    // or 7 for passive

Rung 3: Absolute Move
[MAM] MyAxis
  Motion_Type: ABSOLUTE
  Position: 100.0
  Speed: 50.0
  Accel: 200.0
  Decel: 200.0
  
Rung 4: Check Position
[GSV] MyAxis ActualPosition Axis_Pos
[GRT] Axis_Pos Target 0.1    // ±0.1 tolerance
[OTE] At_Target
```

### AB Ladder Motion Complete Example

```
// Studio 5000 Ladder - Pick and Place

Rung 1: [XIC] EmergencyStop_OK [XIC] SafetyOK
        [MOV] 1 State

Rung 2: [EQU] State 1 [XIC] HomeButton
        [MSO] XAxis   [MSO] ZAxis

Rung 3: [EQU] State 1 [XIC] XAxis.ServoActionStatus
        [XIC] ZAxis.ServoActionStatus
        [MASR] XAxis Position 0.0
        [MASR] ZAxis Position 0.0
        [MOV] 2 State

Rung 4: [EQU] State 2 [XIC] StartCycle
        [MAM] XAxis
        Destination: 150.0 (Pick X)
        Speed: 100.0
        [MOV] 3 State

Rung 5: [EQU] State 3 [XIC] XAxis.PCCommandComplete
        [MAM] ZAxis
        Destination: -50.0 (Down)
        Speed: 30.0
        [MOV] 4 State
```

---

## Step 666: Mitsubishi MR-J4 Series

### MR-J4 Servo System

```
Mitsubishi MR-J4 Features:
├── Full closed-loop control
├── Built-in vibration suppression
├── One-touch tuning (auto gain)
├── Safety (STO, SS1, SS2)
├── SSCNET III/H communication (800Mbps)
└── Compatible: QD75, LD75, RD75, Simple Motion

Interface Options:
├── Pulse Train Input (P/I mode)
├── Analog (±10V) for speed/torque
└── SSCNET III/H (network control from iQ-R)
```

### Simple Motion Module (RD75) + MR-J4

```
// Mitsubishi GX Works3 - Simple Motion

// Configuration:
// RD75P2 → 2 axes
// Axis 1: X-axis MR-J4-70A
// Axis 2: Y-axis MR-J4-70A

// Positioning Data Setup:
// D[n] register mapping:
// Un\G600-G699: Axis 1 Position Data (10 points)
// Un\G700-G799: Axis 2 Position Data (10 points)

// Ladder - Start Positioning:
[MOV K1 Un\G400]      // Axis 1: Select positioning data #1
[MOV K0001 Un\G0]     // Axis 1: Set start bit

// Monitor:
[MOV Un\G200 D100]    // Axis 1: Current position → D100
[MOV Un\G210 D102]    // Axis 1: Current speed → D102
[MOV Un\G204 D104]    // Axis 1: Current value (pulses) → D104

// อ่านสถานะ:
[BIT TEST Un\G100.0 M100]  // Bit 0: Busy flag
[BIT TEST Un\G100.1 M101]  // Bit 1: Start complete
[BIT TEST Un\G100.2 M102]  // Bit 2: In-position
```

---

## Step 667: EtherCAT Motion

### EtherCAT Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    EtherCAT Network                        │
│                                                            │
│  Master (PLC/Motion Controller)                           │
│  │                                                        │
│  └─→ [Slave 1: Drive 1] → [Slave 2: Drive 2] → ...       │
│         ↑Process Data        ↑Process Data                │
│         └─────────────────────┘                           │
│              (Daisy-chain)                                │
│                                                            │
│  Key Features:                                             │
│  ├── Cycle Time: 125μs - 1ms                              │
│  ├── Sync Accuracy: < 1μs (for multi-axis sync)           │
│  ├── Protocol: IEC 61800-7 CiA 402                       │
│  └── Topology: Daisy-chain, Star, Ring                    │
│                                                            │
│  Supported by:                                             │
│  ├── Siemens S7-1500T (with EtherCAT CM module)           │
│  ├── Beckhoff TwinCAT                                     │
│  ├── OMRON Sysmac                                         │
│  └── Many third-party drives                              │
└────────────────────────────────────────────────────────────┘
```

### CoE (CANopen over EtherCAT) Objects

```
Key CoE Object Dictionary Entries:

Object 6040h: Control Word
  Bit 0: Switch On
  Bit 1: Enable Voltage
  Bit 2: Quick Stop
  Bit 3: Enable Operation
  Bit 4-6: Mode specific
  Bit 7: Fault Reset

Object 6041h: Status Word
  Bit 0: Ready to Switch On
  Bit 1: Switched On
  Bit 2: Operation Enabled
  Bit 3: Fault
  Bit 10: Target Reached
  Bit 12: Following Error

Object 6060h: Mode of Operation
  1 = Profile Position Mode (PP)
  3 = Profile Velocity Mode (PV)
  6 = Homing Mode
  8 = Cyclic Sync Position (CSP) - ใช้กับ Real-time

Object 607Ah: Target Position (CSP)
Object 60FFh: Target Velocity (CSV)
Object 6062h: Demand Position (actual)
```

---

## Step 668: Electronic Gearing และ Camming

### Electronic Gearing

```
Electronic Gearing:
- Slave Axis ติดตาม Master Axis
- Ratio สามารถกำหนดได้ (N:M)
- ใช้ในงาน: Synchronization ระหว่าง Axis

ตัวอย่าง: สายการผลิตพิมพ์ (Print Line)
- Master: Conveyor Speed
- Slave: Print Drum Speed
- Ratio: 1:1 หรือปรับตาม Print Size

ตัวอย่าง: Coiling Machine
- Master: Infeed Roll Speed  
- Slave: Coiling Drum Speed
- Ratio: เปลี่ยนตาม Coil Diameter (เพิ่มขึ้น)
```

### Cam Profile (Electronic Cam)

```
┌─────────────────────────────────────────────────────────┐
│                  Electronic Cam Profile                  │
│                                                         │
│  Slave Position                                         │
│   │    ╭────╮                                           │
│   │   /      \                                          │
│   │  /        \                                         │
│   │ /          \    ╭────────╮                          │
│   │/            \  /          \                         │
│   └─────────────────────────────→ Master Position      │
│       Lobe (Dwell)    Rest                              │
│                                                         │
│  ใช้ใน:                                                │
│  ├── Packaging: Cut & Place synchronized to conveyor   │
│  ├── Cam-follower replacement                          │
│  └── Variable motion profiles                          │
└─────────────────────────────────────────────────────────┘
```

### Ladder - Electronic Gearing (Siemens)

```
// S7-1500T + SINAMICS S120 Electronic Gearing

// MC_GearIn Instruction
Network 1: Enable Gear Coupling

  |                                                         |
  |--[ ]----------[MC_GearIn]------------------------------|
  |  Enable_Gear   Slave: "PrintDrum_Axis"                 |
  |               Master: "Conveyor_Axis"                  |
  |               RatioNumerator: 1                        |
  |               RatioDenominator: 1                      |
  |               Acceleration: 200.0                      |
  |               Execute: TRUE                            |
  |               InSync: "Gear_In_Sync"                   |
  |               Error: "Gear_Error"                      |
  |                                                         |

// MC_GearOut - ยกเลิก Gearing
Network 2: Disengage Gear

  |                                                         |
  |--[ ]----------[MC_GearOut]------------------------------|
  |  Disengage_Gear  Slave: "PrintDrum_Axis"               |
  |                  Execute: TRUE                          |
  |                  Done: "Gear_Out_Done"                  |
  |                                                         |
```

---

## Step 669: Multi-Axis Coordination

### Coordinated Motion (Interpolation)

```
Linear Interpolation (G1 in CNC):
- เคลื่อนที่หลาย Axis พร้อมกัน
- ผลลัพธ์: เส้นตรง ณ ทุกมุม
- ต้องการ: Path Speed, Target Point

Circular Interpolation (G2/G3 in CNC):
- เคลื่อนที่เป็นวงกลม/โค้ง
- ต้องการ: Center Point, Radius, Direction
```

### Path Motion ใน S7-1500T

```
// Technology Object: Path Axis Group

// กำหนด Axis Group
Group: XY_Group
  Axis_1: X_Axis (Linear)
  Axis_2: Y_Axis (Linear)

// MC_MoveLinearAbsolute
Network 1: เคลื่อนที่ตรงไปยังจุด (100, 200)

  |                                                         |
  |--[ ]----------[MC_MoveLinearAbsolute]------------------|
  |  Start_Line   Group: "XY_Group"                        |
  |               Execute: TRUE                            |
  |               Position[1]: 100.0 (X)                  |
  |               Position[2]: 200.0 (Y)                  |
  |               PathVelocity: 50.0 (mm/s)               |
  |               Done: "Line_Done"                        |
  |                                                         |

// MC_MoveCircularAbsolute
Network 2: เคลื่อนที่เป็นวงกลม

  |                                                         |
  |--[ ]----------[MC_MoveCircularAbsolute]----------------|
  |  Start_Circle  Group: "XY_Group"                       |
  |               Execute: TRUE                            |
  |               AuxPoint[1]: 150.0 (X mid)              |
  |               AuxPoint[2]: 100.0 (Y mid)              |
  |               EndPoint[1]: 100.0 (X end)              |
  |               EndPoint[2]: 0.0 (Y end)                |
  |               PathVelocity: 30.0                      |
  |               Done: "Circle_Done"                      |
  |                                                         |
```

---

## Step 670: Complete Point-to-Point Motion Program

### ระบบ Pick and Place - Complete Example

```
// ============================================
// COMPLETE P&P SYSTEM - SIEMENS S7-1200 + 
// SINAMICS S120 VIA PROFINET
// ============================================

// DB "Motion_Control":
X_Target_Pos   : REAL := 0.0   // X ตำแหน่งเป้าหมาย
Z_Target_Pos   : REAL := 0.0   // Z ตำแหน่งเป้าหมาย
X_Actual_Pos   : REAL          // X ตำแหน่งจริง
Z_Actual_Pos   : REAL          // Z ตำแหน่งจริง
State          : INT := 0      // State Machine
Cycle_Count    : DINT          // จำนวน Cycle
Error_Code     : INT           // รหัสผิดพลาด

// Pick/Place Positions:
X_Pick_Pos     : REAL := 50.0  // X ตำแหน่ง Pick
Z_Pick_Down    : REAL := -80.0 // Z ลงไป Pick
Z_Safe_Height  : REAL := 0.0   // Z ความสูงปลอดภัย
X_Place_Pos    : REAL := 300.0 // X ตำแหน่ง Place
Z_Place_Down   : REAL := -60.0 // Z ลงไป Place
```

### State Machine Ladder

```
// State 0: IDLE - รอคำสั่ง
Network 1:
  |--[=]--[ ]---[MOVE]----------------------------------|
  |  State  Start  IN: 1 (Change to HOMING state)       |
  |  = 0    Button OUT: State                           |
  |                                                     |

// State 1: HOMING X
Network 2:
  |--[=]--------[MC_Home]------------------------------|
  |  State = 1   Axis: X_Axis                          |
  |             Execute: TRUE                          |
  |             Mode: 3                               |
  |             Done: X_Home_Done                     |
  |             Error: X_Home_Error                   |
  |                                                     |

  |--[=]--[ ]---[MOVE]----------------------------------|
  |  State  X_Home  IN: 2 (Homing Z)                   |
  |  = 1    Done    OUT: State                         |
  |                                                     |

// State 2: HOMING Z
Network 3:
  |--[=]--------[MC_Home]------------------------------|
  |  State = 2   Axis: Z_Axis                          |
  |             Execute: TRUE                          |
  |             Mode: 3                               |
  |             Done: Z_Home_Done                     |
  |                                                     |

  |--[=]--[ ]---[MOVE]----------------------------------|
  |  State  Z_Home  IN: 3 (Move to Pick X)             |
  |  = 2    Done    OUT: State                         |
  |                                                     |

// State 3: MOVE X TO PICK
Network 4:
  |--[=]--------[MC_MoveAbsolute]---------------------|
  |  State = 3   Axis: X_Axis                          |
  |             Execute: TRUE                          |
  |             Position: "MC_DB".X_Pick_Pos           |
  |             Velocity: 100.0                        |
  |             Done: X_At_Pick                        |
  |                                                     |

  |--[=]--[ ]---[MOVE]----------------------------------|
  |  State  X_At_  IN: 4 (Move Z Down)                 |
  |  = 3    Pick   OUT: State                         |
  |                                                     |

// State 4: MOVE Z DOWN TO PICK
Network 5:
  |--[=]--------[MC_MoveAbsolute]---------------------|
  |  State = 4   Axis: Z_Axis                          |
  |             Execute: TRUE                          |
  |             Position: "MC_DB".Z_Pick_Down          |
  |             Velocity: 30.0  (ช้าๆ เมื่อลง)         |
  |             Done: Z_At_Pick_Down                   |
  |                                                     |

// State 5: ACTIVATE GRIPPER
Network 6:
  |--[=]--[ ]---[SET]----[TON]------------------------|
  |  State  Z_At   Gripper  Timer_Grip               |
  |  = 4    Pick   Solenoid  PT: T#500ms (รอ grip)   |
  |         Down                                       |
  |                                                     |
  |--[=]--[TQ]--[MOVE]--------------------------------|
  |  State  Timer  IN: 5 (Move Z Up)                  |
  |  = 4    Done   OUT: State                         |
  |                                                     |

// State 5-9: Continue similar pattern for:
// State 5: Move Z Up (with part)
// State 6: Move X to Place Position
// State 7: Move Z Down to Place
// State 8: Release Gripper
// State 9: Move Z Up → Complete cycle, State → 3 (Next Pick)

// State 99: ERROR
Network Error:
  |--[ ]-----------[MOVE]------------------------------|
  |  Any_Error       IN: 99                            |
  |                  OUT: State                        |
  |                  (All outputs OFF already)         |
  |                                                     |
```

---

## Step 671-680: Advanced Topics

### Gain Tuning สำหรับ Servo

```
Servo Gain Parameters:
┌─────────────────────────────────────────────────────────┐
│  Position Loop Gain (Kp)                                │
│  - สูง = ตอบสนองเร็ว แต่อาจสั่น                        │
│  - ต่ำ = ตอบสนองช้า แต่ Stable                         │
│                                                         │
│  Velocity Loop Gain (Kv)                               │
│  - ควบคุมความเร็วใน Position Loop                      │
│                                                         │
│  Integral Gain (Ki)                                     │
│  - ลด Steady-State Error                               │
│  - สูงเกินไป = Oscillation                             │
│                                                         │
│  Feed Forward (Kff)                                     │
│  - ชดเชย Lag ของ System                               │
│  - ปรับปรุง Following Error                            │
└─────────────────────────────────────────────────────────┘
```

### Servo Fault Handling

```
// OB40 หรือ Hardware Interrupt สำหรับ Servo Fault

Network 1: ตรวจสอบ Drive Fault (จาก Status Word)

  |                                                    |
  |--[BIT]--------[SET]-------------------------------|
  |  Status_Word   "Drive_Fault_Alarm"                 |
  |  Bit 3 (Fault)                                     |
  |                                                    |

Network 2: อ่าน Fault Code

  |                                                    |
  |--[ ]----------[RDREC]------------------------------|
  |  Fault_Active  Read Diagnostics Record from Drive   |
  |               Record: 0xA0A0 (Fault buffer)        |
  |               Result: "Drive_Fault_Code"           |
  |                                                    |

Network 3: Fault Reset เมื่อ ACK

  |                                                    |
  |--[P]----------[MC_Reset]--------------------------|
  |  HMI_Fault_   Axis: "X_Axis"                      |
  |  Reset         Execute: TRUE                       |
  |               Done: "Fault_Reset_Done"             |
  |                                                    |
```

---

## Lab Projects

### Lab 1: Servo Commissioning

**วัตถุประสงค์:** Commissioning SINAMICS S120

**ขั้นตอน:**
1. ต่อวงจร SINAMICS S120 + 1FK7 Motor
2. กำหนด Motor Data จาก Nameplate
3. Run One-Button Tuning
4. Test JOG Mode
5. Test Positioning
6. Verify Accuracy ด้วย Dial Gauge

### Lab 2: Point-to-Point Motion with 2 Axes

```
โจทย์: เจาะรู 9 จุดบนแผ่น Workpiece (Grid 3x3)
- ระยะห่าง X: 50mm
- ระยะห่าง Y: 50mm
- ความลึก: 20mm
- ลำดับ: Serpentine (ซิกแซก)

ต้องการ:
1. Home ทั้ง 2 แกน
2. เคลื่อนไปแต่ละจุดตามลำดับ
3. เจาะ (ลง + ขึ้น)
4. บันทึกเวลาที่ใช้ต่อจุด
5. Error Handling
```

### Lab 3: Electronic Gearing - Coating Line

```
โจทย์: ระบบ Coating
- Conveyor speed: 0-20 m/min (Variable)
- Coating roller: ต้องตามความเร็ว Conveyor
- Gear ratio: 1:1.02 (Coating ออก 2% มากกว่า)

ต้องการ:
1. Master: Conveyor Speed (from Encoder)
2. Slave: Coating Roller (Servo)
3. Electronic Gearing Ratio: 1:1.02
4. Engage/Disengage Gearing
5. Sync Fault Detection (slip > 0.5%)
```

---

## สรุป Part 34

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Servo Components | Drive, Motor, Encoder, Feedback |
| Control Modes | Position, Speed, Torque |
| Siemens S120 | SINA_POS, PROFINET |
| Allen-Bradley Kinetix | CIP Motion Instructions |
| Mitsubishi MR-J4 | Simple Motion Module |
| EtherCAT | Real-time Fieldbus |
| Electronic Gearing | MC_GearIn/GearOut |
| Multi-Axis | Path Interpolation |

---

## การนำทาง

- [← Part 33: Motion Control พื้นฐาน](part33_motion_control_basic.md)
- [→ Part 35: Variable Frequency Drive](part35_vfd_control.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 34 of 40 | Steps 661-680*
