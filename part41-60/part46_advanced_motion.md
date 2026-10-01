# Part 46: Advanced Motion Control (Steps 901-920)

## ภาพรวม
Advanced Motion Control ครอบคลุมการควบคุมการเคลื่อนที่หลายแกนพร้อมกัน การทำ cam profiling, gear ratio อิเล็กทรอนิกส์ และการ synchronize ระหว่างแกน ซึ่งเป็นพื้นฐานของเครื่องจักรสมัยใหม่

---

## Step 901: Multi-Axis Coordination

### หลักการ Multi-Axis Motion

```
Multi-Axis Coordination Types:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. Independent (ไม่เกี่ยวข้องกัน):                        │
│     Axis 1 ─────── (ทำงานอิสระ)                            │
│     Axis 2 ─────── (ทำงานอิสระ)                            │
│                                                             │
│  2. Master-Slave (ตาม Master):                              │
│     Master ──sync──► Slave 1                               │
│                  └──► Slave 2                              │
│                                                             │
│  3. Coordinated (Interpolation):                           │
│     Axis X ──┐                                             │
│     Axis Y ──┼──► Motion Planner ──► Curved path          │
│     Axis Z ──┘                                             │
│                                                             │
│  4. Gantry (Parallel):                                     │
│     X1 ──┐                                                 │
│     X2 ──┤──► Same position always (cross-coupling)       │
│     Y  ──┘                                                 │
└─────────────────────────────────────────────────────────────┘
```

### PLCopen Motion Library Blocks

```
PLCopen Standard Motion Blocks:
┌────────────────────────────────────────────────────────────┐
│  Single Axis:                          Multi-Axis:         │
│  ├── MC_Power                          ├── MC_GearIn       │
│  ├── MC_Home                           ├── MC_GearOut      │
│  ├── MC_MoveAbsolute                   ├── MC_CamIn        │
│  ├── MC_MoveRelative                   ├── MC_CamOut       │
│  ├── MC_MoveVelocity                   ├── MC_GearInPos    │
│  ├── MC_Stop                           └── MC_PhasingAbsolute│
│  ├── MC_Halt                                               │
│  ├── MC_ReadActualPosition             Coordinated:        │
│  ├── MC_ReadActualVelocity             ├── MC_MoveLinear   │
│  └── MC_Reset                          ├── MC_MoveCircular │
│                                        └── MC_SetKinTransform│
└────────────────────────────────────────────────────────────┘
```

---

## Step 902: Electronic Cam (Cam Table)

### Electronic Cam คืออะไร

```
Mechanical Cam vs. Electronic Cam:
┌─────────────────────────────────────────────────────────────┐
│  Mechanical Cam:                                            │
│  ├── Fixed cam profile                                      │
│  ├── ไม่สามารถเปลี่ยนได้ (ต้องเปลี่ยน physical cam)       │
│  └── เหมาะสำหรับ high-speed, simple profiles              │
│                                                             │
│  Electronic Cam:                                            │
│  ├── Cam table (array ของ position pairs)                  │
│  ├── เปลี่ยน profile ได้ทันที (software)                   │
│  ├── หลาย profiles สำหรับหลาย products                   │
│  └── รองรับ polynomial, spline interpolation              │
│                                                             │
│  Cam Table Structure:                                       │
│  Master Position → Slave Position                          │
│  ──────────────   ───────────────                          │
│       0.00  →         0.00                                 │
│      90.00  →        50.00                                 │
│     180.00  →        90.00                                 │
│     270.00  →        50.00                                 │
│     360.00  →         0.00                                 │
└─────────────────────────────────────────────────────────────┘
```

### Cam Implementation ใน Siemens S120

```pascal
// Electronic Cam in TIA Portal / S120
// Function: MC_CamIn (PLCopen)

// 1. Define Cam Table
VAR
    Cam_Table : ARRAY[0..19] OF STRUCT
        Master_Pos : REAL;   // Master axis position (deg)
        Slave_Pos  : REAL;   // Slave axis position (mm)
        Interpolate: INT;    // 0=Linear, 1=Cubic spline
    END_STRUCT;
END_VAR

// กำหนดค่า Cam Table
Cam_Table[0].Master_Pos := 0.0;    Cam_Table[0].Slave_Pos := 0.0;
Cam_Table[1].Master_Pos := 45.0;   Cam_Table[1].Slave_Pos := 25.0;
Cam_Table[2].Master_Pos := 90.0;   Cam_Table[2].Slave_Pos := 80.0;
Cam_Table[3].Master_Pos := 135.0;  Cam_Table[3].Slave_Pos := 95.0;
Cam_Table[4].Master_Pos := 180.0;  Cam_Table[4].Slave_Pos := 80.0;
Cam_Table[5].Master_Pos := 225.0;  Cam_Table[5].Slave_Pos := 25.0;
Cam_Table[6].Master_Pos := 270.0;  Cam_Table[6].Slave_Pos := 0.0;
Cam_Table[7].Master_Pos := 315.0;  Cam_Table[7].Slave_Pos := 0.0;
Cam_Table[8].Master_Pos := 360.0;  Cam_Table[8].Slave_Pos := 0.0;

// 2. Activate Cam
MC_CamIn_Instance(
    Master     := MasterAxis,      // Master axis
    Slave      := SlaveAxis,       // Slave axis
    Execute    := Cam_Active,      // Trigger
    MasterSync := TRUE,            // Sync at next master cycle start
    CamTable   := Cam_Table        // Reference to cam table
);
```

---

## Step 903: Electronic Gear Ratio

### Electronic Gearing

```
Electronic Gear Ratio:
┌─────────────────────────────────────────────────────────────┐
│  Physical Gearbox:                                          │
│  Master ──[Gear 3:1]──► Slave (1/3 speed)                  │
│                                                             │
│  Electronic Gearing:                                        │
│  Master.Position → × (Ratio_Num/Ratio_Den) → Slave.SP     │
│                                                             │
│  Example: Print registration on packaging machine          │
│  ├── Main conveyor: Master axis (encoder)                  │
│  ├── Print roller: Slave axis                              │
│  └── Gear ratio = Print_Circumference / Pattern_Length     │
│                                                             │
│  Changing ratio on-the-fly:                                 │
│  ├── MasterOffset: Shift phase relative to master          │
│  └── RatioNumerator/Denominator: Change ratio             │
└─────────────────────────────────────────────────────────────┘
```

### Electronic Gear ใน Ladder

```ladder
// MC_GearIn - Electronic Gear Coupling
// Network 1: Activate Gear Coupling

+--[MC_GearIn Instance]--+
|  Master      := MasterAxis        |  // Master axis
|  Slave       := PrintRoller       |  // Slave axis
|  Execute     := Start_Gear        |  // Start signal
|  RatioNumerator   := 2            |  // Gear ratio = 2/3
|  RatioDenominator := 3            |
|  MasterSyncMode   := 1            |  // 1 = sync at catch
|  InGear      => In_Gear_Active    |  // Synchronized
|  Busy        => Gear_Busy         |
|  Error       => Gear_Error        |
+----------------------------------+

// Network 2: Phase Adjustment (for registration correction)
// ถ้าต้องปรับ phase เพื่อ registration
+--[ Registration_Error_Active ]--+
   |--[MC_PhasingAbsolute]--+
   |  Slave      := PrintRoller     |
   |  Execute    := Adjust_Phase    |
   |  PhaseAbsolute := Correction_mm|  // mm to correct
   |  Done       => Phase_Done      |
   +--------------------------------+
```

---

## Step 904: Synchronization

### การ Synchronize แกนหลายแกน

```
Synchronization Methods:
┌─────────────────────────────────────────────────────────────┐
│  1. Time-based Sync:                                        │
│     ทุก axis ใช้ clock เดียวกัน (IEEE 1588 PTP)            │
│     → Synchronize ที่ระดับ microseconds                    │
│                                                             │
│  2. Position-based Sync (Electronic Cam/Gear):             │
│     Slave position = f(Master position)                    │
│     → ไม่ขึ้นกับ speed ของ master                          │
│                                                             │
│  3. EtherCAT Distributed Clock:                            │
│     Hardware sync ผ่าน EtherCAT network                   │
│     → Jitter < 1 µs                                        │
│                                                             │
│  4. PROFINET IRT (Isochronous Real-Time):                  │
│     Cycle time: 250µs - 4ms                                │
│     Jitter: < 1µs                                          │
│     → สำหรับ precision motion control                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 905: Contouring (CNC-like)

### Linear และ Circular Interpolation

```
Contouring on PLC:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  LINEAR Interpolation (MC_MoveLinear):                     │
│                                                             │
│  Start (0,0)                                               │
│       │     End (100, 50)                                  │
│       └──────────────────►                                 │
│              Axis X and Y move simultaneously              │
│              to maintain straight line path                │
│                                                             │
│  CIRCULAR Interpolation (MC_MoveCircular):                 │
│                                                             │
│  Start ──────────────► End                                 │
│          (Arc path)                                        │
│  Center point defines circle                               │
│                                                             │
│  Path Velocity = Constant (programmed velocity)            │
│  X velocity = V × cos(θ)                                   │
│  Y velocity = V × sin(θ)                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 906: PLCopen Motion Standard

### PLCopen Motion Control Overview

```
PLCopen MC Part 1 - Single Axis State Machine:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    Disabled                                 │
│                       │                                     │
│               [MC_Power Enable=TRUE]                        │
│                       │                                     │
│                    Standstill ◄──── MC_Stop (Done)         │
│                    (Ready)         MC_Halt (Done)          │
│                       │                                     │
│          ┌────────────┼────────────────┐                   │
│          ▼            ▼                ▼                   │
│    Homing       Continuous       Discrete                  │
│   (MC_Home)     Motion        Motion                       │
│                 (Velocity)    (Abs/Rel)                    │
│                               │                            │
│                          MC_Stop ──► Stopping              │
│                                              │             │
│                                        Standstill          │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 907: Siemens S120 with TIA Portal

### S120 Drive System

```
Siemens S120 System Architecture:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  CU320-2 (Control Unit)                                    │
│  ├── Connects to TIA Portal via PROFINET                   │
│  ├── Controls up to 6 axes                                 │
│  └── Runs drive functions (speed, torque)                  │
│                                                             │
│  Active Infeed Module (ALM)                                │
│  └── Regenerative power supply                            │
│                                                             │
│  Motor Modules (Single/Double axis):                       │
│  ├── 1FK7 Servo Motor + Encoder                           │
│  └── Supports: 0.05 - 130Nm                               │
│                                                             │
│  Connection to TIA Portal:                                  │
│  ├── S7-1500 (Technology CPU) → PROFINET → CU320          │
│  ├── Motion control via PLCopen blocks in TIA Portal       │
│  └── SINAMICS Startdrive for drive configuration          │
└─────────────────────────────────────────────────────────────┘
```

### Technology Object Configuration (TIA Portal)

```
Technology Object (TO) Configuration:
1. Create TO_Axis in TIA Portal
   ├── Name: "Axis_X"
   ├── Drive: SINAMICS S120 CU320
   └── Encoder: Absolute encoder (EQI-1130)

2. Set Mechanical Parameters:
   ├── Lead screw pitch: 10mm/rev
   ├── Gear ratio: 1:5 (reduction)
   └── Max velocity: 500mm/s

3. Limits:
   ├── Position: -500mm to +500mm
   ├── Velocity: 500mm/s (max)
   └── Acceleration: 2000mm/s²

4. Homing:
   ├── Method: Active + home switch
   ├── Home position: 0mm
   └── Homing velocity: 50mm/s
```

---

## Step 908: Rockwell Kinetix 5700

### Kinetix 5700 System

```
Kinetix 5700 Architecture:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Kinetix 5700 Dual Axis Servo Drive:                       │
│  ├── EtherNet/IP for communication                        │
│  ├── Integrated safety (STO - Safe Torque Off)            │
│  └── Connects to GuardLogix/ControlLogix                  │
│                                                             │
│  MPL Series Servo Motors:                                  │
│  ├── Absolute multi-turn encoder                          │
│  └── 0.35 - 173 Nm                                        │
│                                                             │
│  Studio 5000 Motion Configuration:                         │
│  ├── Axis Properties → Drive: K5700                       │
│  ├── Motor: MPL-A4540F                                    │
│  └── Motion Group: 1ms update rate                        │
│                                                             │
│  CIP Motion Protocol:                                      │
│  ├── Time synchronization: CIP Sync (IEEE 1588)           │
│  └── Jitter: < 100µs                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 909: Multi-Axis Homing Sequence

### การ Home หลายแกนพร้อมกัน

```pascal
// Multi-Axis Homing Sequence
// FB: Home_All_Axes

FUNCTION_BLOCK Home_All_Axes
VAR_INPUT
    Start_Home  : Bool;   // เริ่ม homing
    Abort_Home  : Bool;   // ยกเลิก homing
END_VAR

VAR_OUTPUT
    Homing_Done : Bool;   // Homing เสร็จสิ้น
    Homing_Error: Bool;   // มี error
    Active_Axis : Int;    // แกนที่กำลัง home
END_VAR

VAR
    Home_Z   : MC_Home;  // Z axis home (ขึ้นก่อน - safety)
    Home_X   : MC_Home;  // X axis home
    Home_Y   : MC_Home;  // Y axis home
    Step     : Int := 0;
END_VAR

BEGIN
    CASE Step OF
    
        0: // Wait for start
            Homing_Done := FALSE;
            IF Start_Home THEN Step := 10; END_IF;
        
        10: // Step 1: Home Z axis first (safety - move UP first)
            Active_Axis := 3;  // Z axis
            Home_Z(
                Axis     := Axis_Z,
                Execute  := TRUE,
                HomingMode := 3  // Reference pulse + switch
            );
            IF Home_Z.Done THEN Step := 20;
            ELSIF Home_Z.Error THEN Step := 99;
            END_IF;
        
        20: // Step 2: Home X and Y simultaneously
            Active_Axis := 12;  // X and Y
            Home_X(
                Axis     := Axis_X,
                Execute  := TRUE,
                HomingMode := 3
            );
            Home_Y(
                Axis     := Axis_Y,
                Execute  := TRUE,
                HomingMode := 3
            );
            // Wait for BOTH to complete
            IF Home_X.Done AND Home_Y.Done THEN Step := 30;
            ELSIF Home_X.Error OR Home_Y.Error THEN Step := 99;
            END_IF;
        
        30: // All homed
            Homing_Done := TRUE;
            Active_Axis := 0;
            Step := 0;
        
        99: // Error
            Homing_Error := TRUE;
            // Stop all axes
            MC_Stop_X(Axis := Axis_X, Execute := TRUE, Deceleration := 500.0);
            MC_Stop_Y(Axis := Axis_Y, Execute := TRUE, Deceleration := 500.0);
            MC_Stop_Z(Axis := Axis_Z, Execute := TRUE, Deceleration := 500.0);
    
    END_CASE;
END_FUNCTION_BLOCK
```

---

## Step 910: Coordinated Stop on Fault

### การหยุดแบบ Coordinated บน Fault

```
Coordinated Stop Categories (PLCopen):
┌─────────────────────────────────────────────────────────────┐
│  Stop Category 0 (Uncontrolled):                            │
│  ├── Immediate power removal                               │
│  ├── STO (Safe Torque Off)                                │
│  └── Use: Safety emergency, SIL 2+                        │
│                                                             │
│  Stop Category 1 (Controlled → then STO):                  │
│  ├── Decelerate at max rate to standstill                  │
│  ├── Then remove power                                    │
│  └── Use: Normal emergency stop                           │
│                                                             │
│  Stop Category 2 (Controlled, hold position):              │
│  ├── Decelerate to standstill                             │
│  ├── Hold position (servo remains enabled)               │
│  └── Use: Normal stop, pause                              │
└─────────────────────────────────────────────────────────────┘
```

### Fault Stop Program

```pascal
// Coordinated Fault Stop
// เมื่อเกิด fault ใน multi-axis system

IF Motion_Fault OR Safety_Stop THEN
    
    // Category 1 Stop: Decelerate then STO
    MC_Stop_X(
        Axis         := Axis_X,
        Execute      := TRUE,
        Deceleration := Max_Decel  // 5000 mm/s² (max)
    );
    MC_Stop_Y(
        Axis         := Axis_Y,
        Execute      := TRUE,
        Deceleration := Max_Decel
    );
    
    // Wait for all axes stopped
    IF MC_Stop_X.Done AND MC_Stop_Y.Done THEN
        // Disable drives
        MC_Power_X(Axis := Axis_X, Enable := FALSE);
        MC_Power_Y(Axis := Axis_Y, Enable := FALSE);
        All_Axes_Safe := TRUE;
    END_IF;

END_IF;
```

---

## Step 911: Registration (Flying Shear)

### Flying Shear / Registration

```
Flying Shear Operation:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Material (moving at V_line):                              │
│  ──────────────────────────────────────────────────►       │
│  │────────────│────────────│────────────│           │      │
│  Cut target   Cut target                             │     │
│                                                      │     │
│  Shear (moves with material during cut):             │     │
│  ←─────────────[     ]──────────────►               │     │
│         Retract  CUT  Advance                              │
│                                                            │
│  Sequence:                                                 │
│  1. Detect cut mark (registration sensor)                 │
│  2. Calculate shear velocity = line velocity             │
│  3. MC_GearIn: Sync shear to line                        │
│  4. MC_MoveRelative: Move to cut position               │
│  5. Actuate cutter                                       │
│  6. MC_GearOut: Decouple from line                      │
│  7. Return to start position                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 912-920: Complete 2-Axis Pick and Place Program

### โครงการ: 2-Axis Cartesian Pick and Place

#### 1. System Description

```
2-Axis Cartesian Pick and Place:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Pick Position (0, 0)      Place Position (200, 100)       │
│       ×                              ×                     │
│       │                              │                     │
│       │    X-axis (Horizontal)       │                     │
│  ─────┼──────────────────────────────┼──────────────────   │
│       │                              │                     │
│       │    Y-axis (Vertical)         │                     │
│                                                             │
│  Specs:                                                    │
│  - X Stroke: 300mm                                        │
│  - Y Stroke: 150mm (up/down)                              │
│  - Max velocity: 500mm/s                                   │
│  - Acceleration: 2000mm/s²                                │
│  - Repeatability: ±0.05mm                                 │
│  - Cycle time target: 2 seconds                           │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Motion Profile

```
Pick and Place Motion Profile:
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Home (if not homed)                                │
│  Step 2: Move to Pick Position                             │
│    ├── X → 0mm (MoveAbsolute)                              │
│    └── Y → 0mm (simultaneously with X)                    │
│  Step 3: Descend (Y down)                                  │
│    └── Y → 50mm (pick height)                             │
│  Step 4: Pick (vacuum on, wait 200ms)                      │
│  Step 5: Ascend (Y up)                                     │
│    └── Y → 0mm                                            │
│  Step 6: Move to Place Position                            │
│    ├── X → 200mm                                           │
│    └── Y → 0mm (move X while Y is up)                    │
│  Step 7: Descend (Y down)                                  │
│    └── Y → 100mm (place height, terrain may vary)         │
│  Step 8: Place (vacuum off, wait 200ms)                    │
│  Step 9: Ascend (Y up)                                     │
│  Step 10: Return for next cycle                            │
└─────────────────────────────────────────────────────────────┘
```

#### 3. Complete Ladder Program (Siemens TIA Portal)

```pascal
// ======================================================
// FB: PickAndPlace_Control
// 2-Axis Cartesian Robot Control
// Using PLCopen MC blocks
// ======================================================

FUNCTION_BLOCK PickAndPlace_Control
VAR_INPUT
    Start_Cycle    : Bool;   // เริ่ม cycle
    Emergency_Stop : Bool;   // E-Stop (Safety)
    Reset          : Bool;   // Reset after fault
    
    // External signals
    Part_Present   : Bool;   // Sensor: part ที่ pick position
    Vac_Feedback   : Bool;   // Vacuum sensor feedback
    
    // Recipe parameters
    Pick_X         : REAL;   // Pick X position (mm)
    Pick_Y         : REAL;   // Pick Y position (mm)
    Pick_Height    : REAL;   // Z height for picking
    Place_X        : REAL;   // Place X position (mm)
    Place_Y        : REAL;   // Place Y position (mm)
    Place_Height   : REAL;   // Z height for placing
END_VAR

VAR_OUTPUT
    Robot_Running  : Bool;   // Robot กำลังทำงาน
    Cycle_Complete : Bool;   // Cycle เสร็จสิ้น
    Fault_Active   : Bool;   // Fault
    Fault_Code     : INT;    // Error code
    
    Vacuum_On      : Bool;   // Vacuum solenoid command
    Step_Display   : INT;    // Current step สำหรับ HMI
END_VAR

VAR
    // Axis instances (Technology Objects)
    Axis_X : TO_SpeedAxis;   // X axis
    Axis_Y : TO_SpeedAxis;   // Y axis
    
    // Motion blocks instances
    MC_Home_X       : MC_Home;
    MC_Home_Y       : MC_Home;
    MC_MoveAbs_X    : MC_MoveAbsolute;
    MC_MoveAbs_Y    : MC_MoveAbsolute;
    MC_Stop_X       : MC_Stop;
    MC_Stop_Y       : MC_Stop;
    MC_Power_X      : MC_Power;
    MC_Power_Y      : MC_Power;
    
    // Internal
    Step            : INT := 0;
    Homed           : Bool;
    Vac_Timer       : TON;
    Settle_Timer    : TON;
    X_Move_Done     : Bool;
    Y_Move_Done     : Bool;
END_VAR

BEGIN
    // ---- Power Enable ----
    MC_Power_X(Axis := Axis_X, Enable := TRUE, EnablePositive := TRUE, EnableNegative := TRUE);
    MC_Power_Y(Axis := Axis_Y, Enable := TRUE, EnablePositive := TRUE, EnableNegative := TRUE);
    
    // ---- Emergency Stop Handling ----
    IF Emergency_Stop THEN
        MC_Stop_X(Axis := Axis_X, Execute := TRUE, Deceleration := 5000.0);
        MC_Stop_Y(Axis := Axis_Y, Execute := TRUE, Deceleration := 5000.0);
        Fault_Active := TRUE;
        Fault_Code := 9999;  // E-Stop
        Step := 0;
    END_IF;
    
    // ---- Main State Machine ----
    CASE Step OF
    
        0: // Idle - Wait for start
            Robot_Running := FALSE;
            Cycle_Complete := FALSE;
            Step_Display := 0;
            
            IF Start_Cycle AND NOT Emergency_Stop AND NOT Fault_Active THEN
                IF NOT Homed THEN
                    Step := 5;  // Go to homing
                ELSE
                    Step := 10; // Go to main cycle
                END_IF;
            END_IF;
            
            IF Reset AND Fault_Active THEN
                Fault_Active := FALSE;
                Fault_Code := 0;
            END_IF;
        
        // ==================
        // HOMING SEQUENCE
        // ==================
        5: // Start homing Y first (safety - move up)
            Robot_Running := TRUE;
            Step_Display := 5;
            MC_Home_Y(Axis := Axis_Y, Execute := TRUE, HomingMode := 3,
                      Position := 0.0, Velocity := 50.0);
            IF MC_Home_Y.Done THEN Step := 6;
            ELSIF MC_Home_Y.Error THEN
                Fault_Code := 100; Fault_Active := TRUE; Step := 0;
            END_IF;
        
        6: // Home X
            MC_Home_X(Axis := Axis_X, Execute := TRUE, HomingMode := 3,
                      Position := 0.0, Velocity := 100.0);
            IF MC_Home_X.Done THEN
                Homed := TRUE;
                Step := 10;
            ELSIF MC_Home_X.Error THEN
                Fault_Code := 101; Fault_Active := TRUE; Step := 0;
            END_IF;
        
        // ==================
        // PICK CYCLE
        // ==================
        10: // Move to pick X and Y simultaneously (Y at safe height)
            Step_Display := 10;
            MC_MoveAbs_X(Axis := Axis_X, Execute := TRUE,
                         Position := Pick_X, Velocity := 400.0,
                         Acceleration := 2000.0, Done => X_Move_Done);
            MC_MoveAbs_Y(Axis := Axis_Y, Execute := TRUE,
                         Position := 0.0,    // Safe height first
                         Velocity := 300.0,  Acceleration := 2000.0,
                         Done => Y_Move_Done);
            IF X_Move_Done AND Y_Move_Done THEN Step := 20; END_IF;
        
        20: // Descend to pick height
            Step_Display := 20;
            MC_MoveAbs_Y(Axis := Axis_Y, Execute := TRUE,
                         Position := Pick_Height, Velocity := 100.0,
                         Acceleration := 1000.0, Done => Y_Move_Done);
            IF Y_Move_Done THEN Step := 30; END_IF;
        
        30: // Check part present, turn on vacuum
            Step_Display := 30;
            IF NOT Part_Present THEN
                // No part at pick position
                Fault_Code := 200;
                Fault_Active := TRUE;
                Step := 0;
            ELSE
                Vacuum_On := TRUE;
                Vac_Timer(IN := TRUE, PT := T#200MS);
                IF Vac_Timer.Q THEN
                    Vac_Timer(IN := FALSE);
                    Step := 35;
                END_IF;
            END_IF;
        
        35: // Verify vacuum
            IF NOT Vac_Feedback THEN
                Fault_Code := 201;  // Vacuum not achieved
                Fault_Active := TRUE;
                Vacuum_On := FALSE;
                Step := 0;
            ELSE
                Step := 40;
            END_IF;
        
        40: // Ascend (with part)
            Step_Display := 40;
            MC_MoveAbs_Y(Axis := Axis_Y, Execute := TRUE,
                         Position := 0.0, Velocity := 150.0,
                         Acceleration := 1500.0, Done => Y_Move_Done);
            IF Y_Move_Done THEN Step := 50; END_IF;
        
        // ==================
        // PLACE CYCLE
        // ==================
        50: // Move to place X (Y stays at safe height)
            Step_Display := 50;
            MC_MoveAbs_X(Axis := Axis_X, Execute := TRUE,
                         Position := Place_X, Velocity := 400.0,
                         Acceleration := 2000.0, Done => X_Move_Done);
            IF X_Move_Done THEN Step := 60; END_IF;
        
        60: // Descend to place height
            Step_Display := 60;
            MC_MoveAbs_Y(Axis := Axis_Y, Execute := TRUE,
                         Position := Place_Height, Velocity := 100.0,
                         Acceleration := 1000.0, Done => Y_Move_Done);
            IF Y_Move_Done THEN Step := 70; END_IF;
        
        70: // Release vacuum
            Step_Display := 70;
            Vacuum_On := FALSE;
            Vac_Timer(IN := TRUE, PT := T#200MS);
            IF Vac_Timer.Q THEN
                Vac_Timer(IN := FALSE);
                Step := 80;
            END_IF;
        
        80: // Ascend
            Step_Display := 80;
            MC_MoveAbs_Y(Axis := Axis_Y, Execute := TRUE,
                         Position := 0.0, Velocity := 200.0,
                         Acceleration := 2000.0, Done => Y_Move_Done);
            IF Y_Move_Done THEN Step := 90; END_IF;
        
        90: // Cycle Complete
            Step_Display := 90;
            Cycle_Complete := TRUE;
            Robot_Running := FALSE;
            Step := 0;  // Ready for next cycle
    
    END_CASE;
END_FUNCTION_BLOCK
```

#### 4. Cycle Time Analysis

```
Cycle Time Calculation:

Movement Times (using kinematics: t = 2√(d/a) for triangular profile):
┌─────────────────────────────────────────────────────────────┐
│  Step 10: Move X=200mm @ 400mm/s, a=2000mm/s²             │
│           t = 200/400 + 400/2000 = 0.5 + 0.2 = 0.7s      │
│                                                             │
│  Step 20: Descend Y=50mm @ 100mm/s, a=1000mm/s²           │
│           t = 50/100 + 100/1000 = 0.5 + 0.1 = 0.6s       │
│                                                             │
│  Step 30: Vacuum wait = 0.2s                               │
│  Step 35: Vacuum verify = 0.1s                             │
│                                                             │
│  Step 40: Ascend Y=50mm = 0.6s                             │
│                                                             │
│  Step 50: Move X=200mm = 0.7s                              │
│                                                             │
│  Step 60: Descend Y=100mm @ 100mm/s = 1.2s                │
│                                                             │
│  Step 70: Release wait = 0.2s                              │
│                                                             │
│  Step 80: Ascend Y=100mm @ 200mm/s = 0.9s                 │
│                                                             │
│  TOTAL ≈ 0.7+0.6+0.2+0.1+0.6+0.7+1.2+0.2+0.9 = 5.2s    │
│                                                             │
│  Target was 2s → Need optimization!                        │
│  Solution: Move X and Y simultaneously + reduce pick height│
└─────────────────────────────────────────────────────────────┘
```

---

## Lab Project 46: 3-Axis CNC-like Dispensing Robot

### โจทย์
เขียนโปรแกรมควบคุม 3-axis dispensing robot สำหรับงาน PCB glue dispensing

### ข้อกำหนด
1. 3 axes: X, Y (horizontal gantry), Z (vertical)
2. Dispense path: Follow pre-programmed XY coordinates
3. Z: ลงมา dispense แล้วขึ้น ที่แต่ละจุด
4. Circular arc interpolation สำหรับ IC pad dispensing
5. Speed: 200mm/s (XY), 50mm/s (Z)
6. Path file: CSV format (X, Y, Z, arc/linear)

### งานที่ต้องทำ
1. ออกแบบ Motion Profile (velocity profile for each segment)
2. Implement MC_MoveLinear + MC_MoveCircular
3. Read path data จาก recipe DB
4. Implement auto-homing sequence
5. ทำ Fault detection (drive fault, limit switch hit)
6. วัด actual cycle time และ compare กับ expected

---

## สรุป Part 46

| ฟีเจอร์ | Siemens S120 | Rockwell K5700 |
|--------|-------------|----------------|
| Protocol | PROFINET IRT | CIP Motion (EtherNet/IP) |
| Time Sync | IEEE 1588 | CIP Sync |
| PLCopen | Yes | Yes |
| Electronic Cam | Yes | Yes |
| Multi-axis | 6 axes/CU | 8 axes/controller |
| Min Cycle | 125µs | 250µs |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 45: Continuous Process Control](part45_continuous_control.md) |
| ► ถัดไป | [Part 47: Robotics Integration](part47_robotics_integration.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
