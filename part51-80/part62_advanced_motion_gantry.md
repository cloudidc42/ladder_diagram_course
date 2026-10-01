# Part 62: Advanced Motion - Gantry and Delta Robots (Steps 1221-1240)

## บทนำ: Motion Control ขั้นสูง

Motion Control เป็นส่วนหนึ่งที่ซับซ้อนที่สุดใน PLC programming:
- ต้องการความแม่นยำสูง (±0.1mm)
- ต้องการ real-time performance (< 1ms)
- ต้องเข้าใจ kinematics (ทิศทางการเคลื่อนที่)
- Safety: หยุดอย่างปลอดภัย

---

## Step 1221: Gantry Robot Fundamentals

### Gantry Robot คืออะไร

```
GANTRY ROBOT (Cartesian Robot):
──────────────────────────────────────────────────────────────

Architecture:
     Y-Axis (Left-Right)
     ←────────────────→
     ┌─────────────────┐    ↑ Z-Axis
     │   X-Axis        │    │ (Up-Down)
  ───┼─────────────────┼───
     │   [Carriage]────┼──→ X-Axis (Forward-Back)
     │   [Tool]        │
  ───┼─────────────────┼───
     │                 │
     └─────────────────┘

Applications:
├── Pick and place (small parts)
├── Palletizing (stacking boxes)
├── Laser cutting (flat surfaces)
├── 3D printing
└── PCB drilling

Advantages:
├── Large work envelope
├── High precision (0.05-0.1mm)
├── Simple kinematics (no inverse kinematics)
└── Easy programming

Disadvantages:
├── Slow (limited by acceleration of large moving mass)
├── Large footprint
└── Not suitable for irregular workspaces
```

---

## Step 1222: Axis Kinematics

### การคำนวณการเคลื่อนที่

```
GANTRY KINEMATICS (Easy - Cartesian):
──────────────────────────────────────────────────────────────

Position ใน Cartesian space = Axis positions
ไม่ต้องคำนวณ inverse kinematics (unlike 6-axis robot)

X_Position = X_Motor_Position × mm_per_revolution / encoder_counts
Y_Position = Y_Motor_Position × mm_per_revolution / encoder_counts
Z_Position = Z_Motor_Position × mm_per_revolution / encoder_counts

ตัวอย่าง: Ball screw drive
- Ball screw pitch: 10 mm/revolution
- Encoder: 2048 counts/revolution (quadrature × 4 = 8192)
- Resolution: 10mm / 8192 counts = 0.00122 mm/count ≈ 1.2 μm

VELOCITY PROFILE:
Trapezoidal (most common):
Velocity
   ↑
Vmax ─────────────────────
   /                     \
  /  acceleration          \ deceleration
─/─────────────────────────────────────────→ Time

Trapezoidal Calculation:
t_acc = Vmax / Acceleration                 // Acceleration time
t_const = (Distance - Vmax²/Acceleration) / Vmax  // Constant velocity time
t_total = 2 × t_acc + t_const

ตัวอย่าง: Move 300mm at 500mm/s, acceleration 2000mm/s²
t_acc = 500/2000 = 0.25 sec
distance_during_acc = ½ × 2000 × 0.25² = 62.5mm
distance_const = 300 - 2 × 62.5 = 175mm
t_const = 175/500 = 0.35 sec
t_total = 0.25 + 0.35 + 0.25 = 0.85 sec
```

---

## Step 1223: Siemens S120 / S7-1500T Motion Control

### SINAMICS S120 Drive System

```
SINAMICS S120 + S7-1500T SETUP:
──────────────────────────────────────────────────────────────

Hardware:
├── S7-1515T-2 PN (Technology CPU with Motion Control)
├── SINAMICS S120 CU320-2 PN (Control Unit)
├── SINAMICS S120 Motor Modules (per axis)
└── Encoders (Absolute EnDat หรือ Incremental TTL)

Software:
├── TIA Portal V17+
├── SIMOTION Scout (alternative for complex motion)
└── S7-1500T: built-in PLCopen Motion Control FBs

TECHNOLOGY OBJECTS (TOs):
// ใน TIA Portal → Technology Objects → New Axis
// กำหนด:
// - Mechanics: Ball screw pitch, gear ratio
// - Encoder: Type, resolution
// - Limits: Software limits (travel range)
// - Dynamics: Max velocity, acceleration, jerk

TO_Axis_X: (Technology Object for X axis)
├── Hardware axis → SINAMICS S120 drive
├── Encoder: Absolute, 20-bit
├── Software limit positive: 1200mm
├── Software limit negative: 0mm
├── Max velocity: 1000 mm/s
├── Max acceleration: 5000 mm/s²
└── Max jerk: 50000 mm/s³
```

### PLCopen Motion Control FBs

```
PLCopen Motion Control FBs (S7-1500T):
──────────────────────────────────────────────────────────────

MC_Power:       Enable/Disable axis
MC_Home:        Home axis (find reference position)
MC_MoveAbsolute: Move to absolute position
MC_MoveRelative: Move relative distance
MC_MoveVelocity: Move at constant velocity
MC_Stop:        Stop axis (controlled)
MC_Halt:        Halt axis (with deceleration)
MC_ReadActualPosition: Read current position
MC_ReadStatus:  Read axis status

// Gantry 3-Axis Program (Structured Text)
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_Gantry3Axis
VAR_INPUT
    bEnable:        BOOL;           // System enable
    bHome:          BOOL;           // Home all axes
    bMoveToPos:     BOOL;           // Move to target
    rTarget_X:      REAL;           // Target X (mm)
    rTarget_Y:      REAL;           // Target Y (mm)
    rTarget_Z:      REAL;           // Target Z (mm)
    rVelocity:      REAL := 500.0;  // Speed (mm/s)
END_VAR

VAR_OUTPUT
    bDone:          BOOL;           // Move complete
    bError:         BOOL;           // Any axis error
    rActual_X:      REAL;           // Actual X (mm)
    rActual_Y:      REAL;           // Actual Y (mm)
    rActual_Z:      REAL;           // Actual Z (mm)
END_VAR

VAR
    // Technology Objects (from TIA Portal)
    Axis_X:         TO_Axis_PTO;    // X axis TO
    Axis_Y:         TO_Axis_PTO;    // Y axis TO
    Axis_Z:         TO_Axis_PTO;    // Z axis TO
    
    // PLCopen FBs - X Axis
    fbPower_X:      MC_Power;
    fbHome_X:       MC_Home;
    fbMoveAbs_X:    MC_MoveAbsolute;
    fbStop_X:       MC_Stop;
    fbReadPos_X:    MC_ReadActualPosition;
    
    // Same for Y and Z...
    fbPower_Y:      MC_Power;
    fbMoveAbs_Y:    MC_MoveAbsolute;
    fbPower_Z:      MC_Power;
    fbMoveAbs_Z:    MC_MoveAbsolute;
    
    // Motion state machine
    iMotionState:   INT := 0;
END_VAR

// STATE MACHINE:
// 0: IDLE
// 1: POWER_ON (enabling drives)
// 2: HOMING (finding reference)
// 3: READY (ready for move commands)
// 4: MOVING (executing move)
// 5: FAULT

CASE iMotionState OF
    0: // IDLE - Power on axes
        fbPower_X(Axis := Axis_X, Enable := bEnable, bRegulatorOn := TRUE);
        fbPower_Y(Axis := Axis_Y, Enable := bEnable, bRegulatorOn := TRUE);
        fbPower_Z(Axis := Axis_Z, Enable := bEnable, bRegulatorOn := TRUE);
        
        IF fbPower_X.Status AND fbPower_Y.Status AND fbPower_Z.Status THEN
            iMotionState := 2;  // All axes powered → go home
        END_IF;
    
    2: // HOMING - All axes home simultaneously
        fbHome_X(Axis := Axis_X, Execute := bHome,
                 Position := 0.0, HomingMode := 5);  // Mode 5: switch + encoder
        fbHome_Y(Axis := Axis_Y, Execute := bHome, Position := 0.0);
        fbHome_Z(Axis := Axis_Z, Execute := bHome, Position := 0.0);
        
        IF fbHome_X.Done AND fbHome_Y.Done AND fbHome_Z.Done THEN
            iMotionState := 3;  // Homed → ready
        END_IF;
    
    3: // READY - Execute move command
        IF bMoveToPos THEN
            // Move Z up first (safety: clear any obstacles)
            fbMoveAbs_Z(Axis := Axis_Z, Execute := TRUE,
                       Position := 50.0, Velocity := rVelocity);
            iMotionState := 4;
        END_IF;
    
    4: // MOVING
        // Phase 1: Z up → wait
        IF fbMoveAbs_Z.Done THEN
            // Phase 2: Move X and Y simultaneously
            fbMoveAbs_X(Axis := Axis_X, Execute := TRUE,
                       Position := rTarget_X, Velocity := rVelocity);
            fbMoveAbs_Y(Axis := Axis_Y, Execute := TRUE,
                       Position := rTarget_Y, Velocity := rVelocity);
        END_IF;
        
        // Phase 3: When X,Y done → move Z down to target
        IF fbMoveAbs_X.Done AND fbMoveAbs_Y.Done THEN
            fbMoveAbs_Z(Axis := Axis_Z, Execute := TRUE,
                       Position := rTarget_Z, Velocity := rVelocity * 0.3); // slow on Z down
        END_IF;
        
        // Phase 4: All done
        IF fbMoveAbs_X.Done AND fbMoveAbs_Y.Done AND fbMoveAbs_Z.Done THEN
            bDone := TRUE;
            iMotionState := 3;  // Back to READY
        END_IF;

END_CASE;

// Read actual positions
fbReadPos_X(Axis := Axis_X);  rActual_X := fbReadPos_X.Position;
fbReadPos_Y(Axis := Axis_Y);  rActual_Y := fbReadPos_Y.Position;
fbReadPos_Z(Axis := Axis_Z);  rActual_Z := fbReadPos_Z.Position;
```

---

## Step 1224: Pick and Place Optimization

### เพิ่มความเร็ว Pick and Place

```
PICK AND PLACE OPTIMIZATION:
──────────────────────────────────────────────────────────────

Cycle Time Reduction Strategies:

1. OVERLAP MOTION:
Bad:  Z up → X,Y move → Z down (sequential)
Good: Z up AND start X,Y move simultaneously

2. PRE-POSITION:
เริ่ม move ไป target ก่อนที่ source operation เสร็จ
เช่น: Gripper กำลัง close → เริ่ม move ไป destination แล้ว

3. VELOCITY OVERLAP:
แทนที่ Stop-Start ทุก waypoint
→ ใช้ Blending: ผ่าน waypoint โดยไม่หยุดสนิท
velocity_at_waypoint = 10-20% ของ max velocity

4. PREDICTIVE HOMING:
หลัง deposit → เริ่ม home ไปยัง pick position ทันที
ไม่รอ confirm ว่า product placed

CYCLE TIME ANALYSIS ตัวอย่าง:
                    Without Opt | With Opt | Savings
Z up (50mm, 200mm/s)   0.25s   |   0.25s  |    -
X move (300mm, 500mm/s) 0.60s  |   0.44s* |  0.16s  ← overlap with Z
Y move (200mm, 500mm/s) 0.40s  |   0.44s* |  (parallel with X)
Z down (50mm, 200mm/s)  0.25s  |   0.25s  |    -
Pick/Place dwell         0.20s  |   0.20s  |    -
Return trip (same)       1.50s  |   1.38s  |  0.12s
-----------------------------------------------------------
TOTAL                   3.20s  |   2.76s  | 0.44s (14%)

*X,Y run in parallel: max(0.60, 0.40) = 0.60 → but optimized path 0.44s
```

---

## Step 1225: Delta Robot Principles

### Delta Robot Kinematics

```
DELTA ROBOT (Parallel Kinematics):
──────────────────────────────────────────────────────────────

Architecture:
        Motor 1    Motor 2    Motor 3
           │          │          │
        Arm 1      Arm 2      Arm 3   (Upper arms)
           │          │          │
        Link 1     Link 2     Link 3  (Lower arms, parallel linkage)
           └──────────┴──────────┘
                   TCP (Tool Center Point)
                      ↓
                   End effector (gripper/suction cup)

Advantages over gantry:
├── MUCH faster (up to 150 picks/min)
├── Lightweight arms → low inertia → fast acceleration
├── 3 DoF (X,Y,Z) - perfect for pick-and-place
└── Clean room friendly (motors above work area)

Applications:
├── Chocolate packaging (150+ picks/min)
├── Pharmaceutical blister pack
├── Small part sorting
└── Food processing

DELTA ROBOT INVERSE KINEMATICS:
(คำนวณ: target X,Y,Z → motor angles θ1, θ2, θ3)

This is non-trivial! ต้องใช้ math:
Given: TCP position (x, y, z)
Find: θ1, θ2, θ3

// Simplified model:
// Constants:
// r1 = upper arm length (e.g., 300mm)
// r2 = lower arm length (e.g., 500mm)
// Rb = base radius (platform size) (e.g., 300mm)
// Re = end effector radius (e.g., 100mm)

// For each arm i (i = 1,2,3):
// xi' = x*cos(120°×i) + y*sin(120°×i)  (rotate coordinate)
// zi' = z
// Then solve:
// (xi' - Rb + Re × cos(θi))² + (zi' - r1×sin(θi))² = r2²
// → quadratic equation → solve for θi
```

---

## Step 1226: Delta Robot PLC Programming

### Programming Delta Robot ใน PLC

```
DELTA ROBOT IN SIEMENS S7-1500T:
──────────────────────────────────────────────────────────────

Option 1: Siemens Kinematic Transformation (Built-in)
TIA Portal V15.1+: Technology Object "Kinematics"
├── Define robot geometry
├── Built-in inverse kinematics
└── Use MC_MoveLinear FB for TCP movements

Option 2: Custom Inverse Kinematics FB
สำหรับ non-standard robots

// FB_DeltaRobot_IK (Inverse Kinematics)
FUNCTION_BLOCK FB_DeltaRobot_IK
VAR_INPUT
    rTarget_X:      REAL;       // Target TCP X (mm)
    rTarget_Y:      REAL;       // Target TCP Y (mm)
    rTarget_Z:      REAL;       // Target TCP Z (mm)
    rArm_Upper:     REAL := 300.0;  // Upper arm length (mm)
    rArm_Lower:     REAL := 500.0;  // Lower arm length (mm)
    rBase_Radius:   REAL := 300.0;  // Base platform radius (mm)
    rEff_Radius:    REAL := 100.0;  // End effector radius (mm)
END_VAR
VAR_OUTPUT
    rAngle_1:       REAL;       // Motor 1 angle (degrees)
    rAngle_2:       REAL;       // Motor 2 angle (degrees)
    rAngle_3:       REAL;       // Motor 3 angle (degrees)
    bReachable:     BOOL;       // Target is within workspace?
END_VAR
VAR
    rTemp_X:        REAL;
    rTemp_Z:        REAL;
    rTemp_A:        REAL;
    rTemp_B:        REAL;
    rTemp_C:        REAL;
    rDiscriminant:  REAL;
    i:              INT;
END_VAR

// Calculate for each motor (simplified):
// For motor i at angle 120×(i-1) degrees:
// Transform to arm's coordinate system
// Solve quadratic for arm angle
// Convert to motor position

// If discriminant < 0 → point not reachable → bReachable = FALSE
```

---

## Step 1227: Linear and Circular Interpolation

### Path Interpolation

```
PATH INTERPOLATION:
──────────────────────────────────────────────────────────────

Types:
1. Linear (G01): เส้นตรงจาก A ไป B
2. Circular (G02/G03): วงกลม
3. Spline: เส้นโค้งที่ smooth (ผ่าน control points)

LINEAR INTERPOLATION ใน PLC (หลาย axes พร้อมกัน):
สำหรับ Gantry: ต้องทำให้ X,Y ถึง target พร้อมกัน

ตัวอย่าง: Move from (0,0) to (300mm, 200mm)
Total distance = √(300² + 200²) = 360.6mm

X velocity: Vx = Vtotal × (ΔX / Distance) = 500 × (300/360.6) = 415.7 mm/s
Y velocity: Vy = Vtotal × (ΔY / Distance) = 500 × (200/360.6) = 277.2 mm/s

// ทำให้ถึงพร้อมกัน → เส้นตรง!

// Siemens S7-1500T: MC_MoveLinear (built-in)
fbMoveLinear(
    Axis_X := Axis_X_Instance,
    Axis_Y := Axis_Y_Instance,
    Execute := bTrigger,
    Position := {x := 300.0, y := 200.0},  // Target
    Velocity := 500.0
);

// Allen-Bradley (Logix): MCLM (Motion Coordinated Linear Move)
MCLM(Conveyor_System.GRP1,      // Axis group
     bTrigger, FALSE, FALSE,    // Execute, Decel, Direction
     [300.0, 200.0],            // Target position [X, Y]
     500.0,                     // Speed (mm/s)
     ...);                      // Other params

CIRCULAR INTERPOLATION:
// MC_MoveCircularAbsolute (center + endpoint)
fbMoveCirc(
    Execute := TRUE,
    CircleMode := 1,             // 1 = via center point
    AuxPoint := {x:=150.0, y:=0.0},  // Center
    EndPoint := {x:=300.0, y:=0.0},  // End point
    PathChoice := 0,             // 0 = clockwise
    Velocity := 300.0
);
```

---

## Step 1228: Teach Mode Programming

### สอน Robot ด้วย Teach Mode

```
TEACH MODE:
──────────────────────────────────────────────────────────────

วิธี program motion โดยไม่ต้องคำนวณ coordinates:
1. Move robot to position ด้วยมือ (jog mode)
2. "Teach" position: บันทึก X,Y,Z ปัจจุบัน
3. ทำซ้ำสำหรับทุก waypoints
4. Play back sequence

// FB_TeachMode สำหรับ Gantry
FUNCTION_BLOCK FB_TeachMode
VAR_INPUT
    bTeachMode:     BOOL;           // Enable teach mode
    bTeachPoint:    BOOL;           // Record current position
    bClearAll:      BOOL;           // Clear all taught points
    bPlayback:      BOOL;           // Execute taught sequence
END_VAR
VAR_OUTPUT
    iPointCount:    INT;            // How many points taught
    bPlaybackDone:  BOOL;           // Sequence complete
END_VAR
VAR
    // Taught positions stored in DB
    teachPoints:    ARRAY[0..49] OF STRUCT
        rX:     REAL;
        rY:     REAL;
        rZ:     REAL;
        rVel:   REAL;
    END_STRUCT;
    iCount:         INT := 0;       // Number of taught points
    iPlayIndex:     INT := 0;       // Current playback index
    bPointAdded_Last: BOOL;
END_VAR

// Teach: bTeachPoint rising edge → record current X,Y,Z
// IF bTeachPoint AND NOT bPointAdded_Last THEN
//     teachPoints[iCount].rX := rActual_X;
//     teachPoints[iCount].rY := rActual_Y;
//     teachPoints[iCount].rZ := rActual_Z;
//     teachPoints[iCount].rVel := rTeachVelocity;
//     iCount := iCount + 1;
//     iPointCount := iCount;
// END_IF;
// bPointAdded_Last := bTeachPoint;
```

---

## Step 1229: Collision Detection and Avoidance

### ป้องกัน Collision

```
COLLISION DETECTION:
──────────────────────────────────────────────────────────────

Methods:
1. Software Zone Check: ตรวจว่า path ผ่าน forbidden zone?
2. Torque Monitoring: แรงบิดเพิ่มขึ้น → อาจชนอะไรบางอย่าง
3. Force/Torque Sensor: ตรวจ collision โดยตรง
4. Vision System: camera ตรวจ obstacles

// Zone Exclusion Check
FUNCTION_BLOCK FB_ZoneExclusion
VAR_INPUT
    rTarget_X:      REAL;
    rTarget_Y:      REAL;
    rTarget_Z:      REAL;
END_VAR
VAR_OUTPUT
    bSafeToMove:    BOOL;
    sBlockedReason: STRING[40];
END_VAR
VAR
    // Zone definitions (rectangular boxes)
    rZ1_Xmin: REAL := 200.0;  rZ1_Xmax: REAL := 400.0;
    rZ1_Ymin: REAL := 100.0;  rZ1_Ymax: REAL := 300.0;
    rZ1_Zmin: REAL := 0.0;    rZ1_Zmax: REAL := 150.0;
    // Zone 2 (another obstacle)
    rZ2_Xmin: REAL := 600.0;  rZ2_Xmax: REAL := 800.0;
    ...
END_VAR

// Check Zone 1:
// IF (rTarget_X >= rZ1_Xmin AND rTarget_X <= rZ1_Xmax) AND
//    (rTarget_Y >= rZ1_Ymin AND rTarget_Y <= rZ1_Ymax) AND
//    (rTarget_Z >= rZ1_Zmin AND rTarget_Z <= rZ1_Zmax) THEN
//     bSafeToMove := FALSE;
//     sBlockedReason := 'Target in exclusion zone 1';
// ELSE
//     bSafeToMove := TRUE;
// END_IF;

// TORQUE MONITORING:
// ถ้า motor torque > 120% ของ normal → potential collision
// MC_ReadAxisError → check following error
// Following error ใหญ่ขึ้น → อาจมี mechanical obstruction
```

---

## Step 1230: Pick and Place with Vision

### Vision-Guided Robot

```
VISION-GUIDED PICK AND PLACE:
──────────────────────────────────────────────────────────────

Vision System Integration:
Camera → Vision Controller → PLC (via Ethernet/Modbus/OPC UA)
PLC request: "Take picture"
Vision response: "Part found at X=145.3mm, Y=223.1mm, Angle=12.5°"

// Communication Protocol (Vision via Modbus TCP):
// Register Map (example COGNEX In-Sight):
// R0: Command (0=idle, 1=trigger)
// R1: Result (0=no result, 1=found, 2=not found)
// R2: X position (× 0.01mm) → INT to REAL
// R3: Y position (× 0.01mm)
// R4: Angle (× 0.1 degree)
// R5: Score (0-100)

FUNCTION_BLOCK FB_VisionPickPlace
VAR_INPUT
    bPick:          BOOL;           // Trigger pick operation
    rConveyorSpeed: REAL;           // Conveyor speed (for tracking)
END_VAR
VAR
    fbModbus:       MB_CLIENT;
    iState:         INT;
    rVision_X:      REAL;           // Part position from vision
    rVision_Y:      REAL;
    rVision_Angle:  REAL;
    rOffset_X:      REAL := 5.0;    // Tool calibration offset
    rOffset_Y:      REAL := -3.5;
    tConveyorDelay: TIME;           // Delay for conveyor movement
END_VAR

// Step 1: Trigger camera
// Step 2: Wait for result (timeout 500ms)
// Step 3: ถ้า result = FOUND:
//   rPickTarget_X := rVision_X + rOffset_X;  // Apply calibration
//   rPickTarget_Y := rVision_Y + rOffset_Y;
// Step 4: Move to pick position
// Step 5: Pick (vacuum on / gripper close)
// Step 6: Move to place position
// Step 7: Place (vacuum off / gripper open)
```

---

## Step 1231: Payload and Dynamics

### คำนวณ Payload และ Dynamics

```
PAYLOAD AND DYNAMICS:
──────────────────────────────────────────────────────────────

Payload Calculation:
Total load = Tool weight + Product weight + Gripper weight

ตัวอย่าง: KUKA KR10 R1100 (10kg payload)
Tool (vacuum cup assembly): 2.5 kg
Max product: 3.5 kg
Total: 6 kg < 10 kg ✓ (OK with margin)

Safety factor: ≥ 1.5 → 6 × 1.5 = 9 kg < 10 kg ✓

INERTIA CALCULATION:
ส่งผลต่อ acceleration และ deceleration capability
M_total = (Product + Tool + Gripper) kg
J_load = M_total × r² (r = distance from axis)

ถ้า J_load สูงเกินไป → acceleration ต้องลดลง → cycle time เพิ่ม

// ตรวจสอบ Inertia Ratio (ใน SINAMICS S120 configurator):
// Target: J_load / J_motor < 10 (ideal < 5)
// ถ้าสูงเกินไป → ใช้ motor ขนาดใหญ่ขึ้น หรือ add gearbox

Acceleration Force:
F = m × a
F = 6 kg × 20 m/s² = 120 N
Add gravity (vertical): +60 N
Total: 180 N

Motor must provide > 180 N × safety_factor

DYNAMIC STIFFNESS:
ความแข็งแกร่งของ structure ส่งผลต่อ vibration
ถ้า structure อ่อน → robot จะสั่นหลัง rapid move
→ ต้อง S-curve velocity profile (add jerk limit)
```

---

## Step 1232: Safety for Motion Systems

### Safety สำหรับ Motion Control

```
SAFETY REQUIREMENTS FOR MOTION SYSTEMS:
──────────────────────────────────────────────────────────────

ISO 10218-1: Robots and Robotic Devices - Safety Requirements

SAFETY FUNCTIONS:
1. Safe Torque Off (STO):
   - ตัดกระแสไฟฟ้าไป motor ทันที
   - Motor coasts to stop (ไม่มี braking)
   - ใช้เมื่อ: E-stop, light curtain triggered
   - S7-1500F + SINAMICS: STO via PROFIsafe

2. Safe Stop 1 (SS1):
   - Controlled deceleration + then STO
   - Motor decelerates ก่อน ตัดกระแส
   - ใช้เมื่อ: ต้องการ stop อย่าง smooth

3. Safely Limited Speed (SLS):
   - จำกัด speed ไม่เกิน safe speed
   - ใช้เมื่อ: คนเข้าไปใน work zone

4. Safe Position (SP):
   - ตรวจว่า axis อยู่ใน safe position range
   - Dual encoder สำหรับ functional safety

PLC SAFETY PROGRAM:
// Safety OB (OB1 สำหรับ F-CPU):
// ตรวจสอบ E-stop → ส่ง STO command ไปทุก drives
// ตรวจสอบ Light curtain → SLS mode

AREA SCANNERS (Safety Light Curtains / Area Scanners):
ทำ Zone protection:
├── Zone A (outermost): Speed limit 250mm/s (SLS)
├── Zone B (inner): Stop (SS1)
└── Zone C (collision): STO

Zone monitoring code:
// IF xZoneB_Intruded THEN
//     // Command all axes: SS1 (controlled deceleration)
//     MC_Stop(Axis := Axis_X, Execute := TRUE, Deceleration := 5000.0);
//     MC_Stop(Axis := Axis_Y, Execute := TRUE, Deceleration := 5000.0);
//     MC_Stop(Axis := Axis_Z, Execute := TRUE, Deceleration := 5000.0);
// END_IF;
```

---

## Step 1233: Multi-Robot Coordination

### ประสานงาน Robot หลายตัว

```
MULTI-ROBOT COORDINATION:
──────────────────────────────────────────────────────────────

ปัญหา: 2 gantry robots share work area
→ ต้องไม่ชนกัน!

APPROACHES:

1. Zones (Simple):
   Robot 1: work in X = 0-600mm
   Robot 2: work in X = 600-1200mm
   ไม่ overlap → ไม่ชน
   
   ข้อเสีย: ไม่ flexible

2. Token Passing:
   มีแค่ 1 robot ที่ "owns" shared zone ในเวลาหนึ่ง
   
// Token Passing Protocol:
DATA_BLOCK DB_SharedZone
    bZoneOwner:     INT := 0;  // 0=free, 1=Robot1, 2=Robot2
END_DATA_BLOCK

// Robot 1 ขอ enter shared zone:
// IF DB_SharedZone.bZoneOwner = 0 THEN
//     DB_SharedZone.bZoneOwner := 1;  // Claim zone
//     // Execute motion in shared zone
//     // ...
//     DB_SharedZone.bZoneOwner := 0;  // Release zone
// ELSE
//     // Wait or do other work
// END_IF;
// ⚠️ ต้องใช้ Semaphore หรือ Atomic operation ป้องกัน race condition

3. Path Planning with Collision Avoidance:
   ซับซ้อนที่สุด แต่ optimal
   Plan path ของทั้ง 2 robots พร้อมกัน
   ใช้ algorithm เช่น CBS (Conflict-Based Search)
   มักใช้ MATLAB/Python นอก PLC สำหรับ planning
```

---

## Step 1234: PROFINET IRT สำหรับ Motion

### PROFINET Isochronous Real-Time

```
PROFINET IRT สำหรับ MOTION CONTROL:
──────────────────────────────────────────────────────────────

PROFINET IRT (Isochronous Real-Time):
├── Cycle time: 0.25ms to 4ms
├── Jitter: < 1μs (very accurate)
├── Use case: synchronized multi-axis motion
└── Requires: IRT-capable switches and devices

ทำไม IRT สำคัญ:
- ถ้า jitter สูง → axes ไม่ sync กัน → path error
- ตัวอย่าง: 0.5ms jitter × 500mm/s = 0.25mm position error
- IRT ลด jitter เหลือ < 1μs → < 0.0005mm error

IRT Configuration ใน TIA Portal:
// Device Properties → PROFINET → IRT Settings
// Sync Master: S7-1515T (ส่ง sync signal)
// Sync Slaves: ทุก SINAMICS drives + ET200 I/O modules

// SINAMICS S120 IRT Setup:
// Drive → Properties → PROFINET → Synchronization
// Sync Master: Yes (สำหรับ drive ที่ lead)
// Sync Slave: Yes (สำหรับ drives อื่น)
// Cycle time: 0.5ms (matching PLC)

// Siemens Isochrone Mode (OB91):
// OB91 รัน ทุก IRT cycle (0.5ms)
// ใน OB91: อ่าน encoder positions → calculate next command → send
// ทุก axis sync กันใน cycle เดียวกัน!
```

---

## Step 1235: Troubleshooting Motion Systems

### วิธี Debug Motion Problems

```
MOTION TROUBLESHOOTING:
──────────────────────────────────────────────────────────────

COMMON PROBLEMS AND SOLUTIONS:

1. Axis Following Error too large:
Symptom: MC_MoveAbsolute Error, Error number 0xCF...
Cause:
   ├── Mechanical friction (ทำความสะอาด, lubricate)
   ├── Load too heavy (reduce speed/acceleration)
   ├── Encoder problem (check cable, replace encoder)
   └── Drive gain too low (tuning needed)

Solution:
   a. ลด velocity/acceleration ก่อน (ดูว่า error หาย?)
   b. Check mechanics: manual move smooth?
   c. Auto-tune drive (SINAMICS: function menu)

2. Position Overshoot:
Symptom: Axis goes past target then corrects
Cause: Controller gain too high / inertia mismatch
Solution: Reduce Kp gain in position controller

3. Vibration after stop:
Symptom: Axis vibrates after reaching position
Cause: KP too high, low mechanical stiffness
Solution: Add jerk limit (Kdot), increase D gain

4. Homing Failure:
Symptom: MC_Home Error
Cause:
   ├── Home switch not found (switch problem)
   ├── Motor moves wrong direction (wiring)
   └── Encoder counting wrong direction
Solution: Force output, check direction; swap encoder channels

DIAGNOSTIC TOOLS:
├── TIA Portal → Online → Drive → Trace (Scope function)
├── SINAMICS: Diagnostics → Trace → Record position/speed/torque
└── Python script + snap7 → log actual vs commanded positions
```

---

## Step 1236: Conveyor Tracking (Flying Shear)

### ติดตาม Object บน Conveyor

```
CONVEYOR TRACKING:
──────────────────────────────────────────────────────────────

Application: Pick moving objects from conveyor

Steps:
1. Object detected (sensor/vision) → record conveyor position
2. Calculate: when object reaches pick zone?
3. Move robot to intercept position (leading)
4. Pick at exact moment object arrives
5. Conveyor keeps moving → robot follows while gripping

// ตัวอย่าง: Flying Cut
// Conveyor speed: 200 mm/s
// Object detected at: X_conveyor = 500mm
// Pick zone center: X_conveyor = 800mm (300mm ahead)
// Time to reach pick zone: 300/200 = 1.5 sec
// Robot move time to pick zone: 0.8 sec
// → Robot must start 0.8 sec after detection (wait 0.7 sec)
// → Or calculate: start moving immediately, arrive 0.7 sec early and wait

CONVEYOR TRACKING CODE:
DATA_BLOCK DB_ConvTrack
    rConveyorPosition:  REAL;       // Encoder position (mm)
    rConveyorSpeed:     REAL;       // mm/s
    bObjectDetected:    BOOL;
    rObjectPosition:    REAL;       // Where object was detected
    rPickZone_Center:   REAL := 800.0;
    rObjectAtPickZone:  BOOL;
END_DATA_BLOCK

// Calculate when object reaches pick zone:
// rTime_to_Pick := (rPickZone_Center - rObjectPosition) / rConveyorSpeed;
// 
// Start robot: รอ (rTime_to_Pick - rRobotMoveTime) วินาที แล้ว move
// ระหว่าง pick: robot position_X = rConveyorPosition + offset
//              robot ต้อง track conveyor ขณะ pick!
```

---

## Step 1237: Energy-Efficient Motion

### ลดพลังงานใน Motion Control

```
ENERGY-EFFICIENT MOTION:
──────────────────────────────────────────────────────────────

1. S-CURVE PROFILE (ลด peak torque):
   Trapezoidal: สูงสุดที่ acceleration onset → peak energy
   S-curve: gradual acceleration → spread over time
   S-curve ลด peak power 20-30%

2. REGENERATIVE BRAKING:
   SINAMICS: Energy recovery during deceleration
   Regenerated back to DC bus → reused by other drives
   Configuration: Drive → Energy Recovery → Enable

3. SLEEP MODE ระหว่าง Idle:
   ถ้า axis ไม่ได้ move > 30 seconds → reduce holding torque
   // MC_Power → set TorqueReduction := 0.3 (30% of rated)
   // สำหรับ vertical axes: ห้าม reduce (axis จะตก!)

4. PATH OPTIMIZATION:
   Shorter path = less energy
   Minimize unnecessary Z movements
   Combine return trip (pick + return = 1 trip แทน 2)

5. STANDSTILL MONITORING:
   // เมื่อ production หยุดชั่วคราว > 5 min:
   // Disable drive (MC_Power Enable := FALSE)
   // → Motor ไม่ energized → ไม่มี holding losses
   // แต่ต้อง re-home เมื่อ restart!
```

---

## Step 1238: Allen-Bradley Motion Programming

### Studio 5000 Motion Commands

```
ALLEN-BRADLEY MOTION PROGRAMMING:
──────────────────────────────────────────────────────────────

Hardware: ControlLogix L8x + Kinetix 5700 (servo drive)
Communication: SERCOS III / EtherNet/IP

Key Motion Instructions:
MSO:   Motion Servo On (enable drive)
MSF:   Motion Servo Off
MHOME: Motion Home axis
MCLM:  Motion Coordinated Linear Move (multi-axis)
MCCM:  Motion Coordinated Circular Move
MAS:   Motion Axis Stop
MAM:   Motion Axis Move (single axis)
MAPC:  Motion Apply Position Cam (electronic cam)
MATC:  Motion Apply Torque Control

// Studio 5000 Structured Text Example:
// MAM: Move Axis - Single Axis

IF xTriggerMove THEN
    MAM(
        Axis           := AxisX,          // Axis tag
        MotionControl  := MC_Axis_X,      // Motion control struct
        MoveType       := 1,              // 1=Absolute, 0=Incremental
        Position       := rTargetX,       // Target (mm)
        Speed          := 500.0,          // mm/s
        SpeedUnits     := 1,              // 1=mm/s
        AccelRate      := 2000.0,         // mm/s²
        AccelJerk      := 0,
        DecelRate      := 2000.0,
        DecelJerk      := 0,
        MergeType      := 0,              // 0=No merge
        MergeSpeed     := 0.0
    );
END_IF;

// MCLM: Move Coordinated Linear (XY together)
IF xTriggerLinear THEN
    MCLM(
        Group          := MotionGroup1,   // Axis group (X+Y)
        MotionControl  := MC_Group1,
        MoveType       := 1,              // Absolute
        Position       := [rTargetX, rTargetY],
        Speed          := 500.0,
        SpeedUnits     := 1,
        AccelRate      := 2000.0,
        DecelRate      := 2000.0
    );
END_IF;
```

---

## Step 1239: Mitsubishi Motion Control

### Mitsubishi Q Series / iQ-R Motion

```
MITSUBISHI MOTION PROGRAMMING:
──────────────────────────────────────────────────────────────

Hardware:
├── iQ-R: R64MTCPU (64-axis motion CPU)
├── Servo: MR-J5 series
└── Communication: SSCNETIII/H (300Mbps fiber)

SSCNETIII/H Advantages:
├── Optical fiber → immune to EMI
├── 0.1ms cycle time
├── Up to 96 axes per system
└── Dedicated motion network (separate from PROFINET)

GX Works3 Motion Programming:
// JOG (Manual mode)
// MOVP: Point-to-point move
// MOVC: Circular move (2 axes)
// MOVL: Linear interpolation

// Structured Text example (GX Works3):
LATCH_POSITION(iAxis:=1, EXECUTE:=xTrigger);  // Latch current position

// MOVP: Move to absolute position (Point to Point)
// [D100]: Target position (×10μm units)
// [D102]: Speed (×1mm/min)
SET M100  // Execute flag
MOV K3000 D100  // Target: 300.0mm (3000 × 10μm)
MOV K30000 D102  // Speed: 300mm/s (30000 × 10μm/s... depends on units)

// ใช้ dedicated Motion SFC (Sequential Function Chart) ใน MT Works2
// สำหรับ complex multi-axis sequences:
// Step 1: MOVE X_AXIS HOME
// Step 2: MOVE Y_AXIS to pos1
// Step 3: (coordinate) MOVE X,Y simultaneously to target
```

---

## Step 1240: Lab Project - 3-Axis Gantry Pick and Place

### Lab: Programming Gantry Robot

```
LAB: 3-AXIS GANTRY ROBOT PICK AND PLACE
══════════════════════════════════════════════════════════════

Simulation Environment:
- Software: Factory I/O (or PLCSIM simulation)
- System: 3-axis gantry with suction cup gripper
- Conveyor: Infeed (random parts) + Outfeed (sorted)

Task Description:
- Parts arrive on infeed conveyor (3 types by color)
- Gantry picks part and places in correct bin:
  - Red parts → Bin A (Position 1)
  - Blue parts → Bin B (Position 2)
  - Green parts → Bin C (Position 3)
- Cycle target: 15 picks/minute

TASK 1: Kinematics Setup (20%)
─────────────────────────────────────────────────────────────
1. Define axis parameters:
   - Ball screw pitch: 10mm/rev
   - Encoder: 8192 counts/rev
   - Calculate position resolution
   - Define software limits (workspace)

2. Homing sequence:
   - Z home first (safety)
   - X, Y home simultaneously
   - Verify homed indicators

TASK 2: Pick and Place Sequence (40%)
─────────────────────────────────────────────────────────────
1. State machine:
   IDLE → HOMING → WAITING → DETECTING → MOVING_TO_PICK → PICKING → MOVING_TO_PLACE → PLACING → WAITING

2. Implement sequence:
   - Z up (safe travel height = 100mm)
   - XY move to pick position
   - Z down to pick height
   - Vacuum on + dwell 200ms
   - Z up
   - XY move to place position
   - Z down to place height  
   - Vacuum off + dwell 100ms
   - Z up
   
3. Part type routing:
   - Read part type from color sensor
   - Select correct bin coordinates

TASK 3: Optimization (25%)
─────────────────────────────────────────────────────────────
1. Measure baseline cycle time
2. Implement 2 optimizations:
   - XY move while Z rising
   - Pre-position: start moving to next pick while placing

3. Measure improved cycle time
4. Document: % improvement

TASK 4: Safety and Diagnostics (15%)
─────────────────────────────────────────────────────────────
1. Implement E-stop (SS1 deceleration)
2. Following error detection
3. Limit switch violation response
4. Cycle counter + production report HMI page

DELIVERABLES:
├── TIA Portal project (or Factory I/O + PLC program)
├── State machine diagram
├── Cycle time analysis (before/after optimization)
├── Test report (all test cases)
└── 2-minute video demo (optional bonus)

GRADING:
├── Correct sequence: 30%
├── Optimization: 20%
├── Safety implementation: 20%
├── Code quality (patterns, comments): 20%
└── Documentation: 10%
```

---

## สรุป Part 62

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| Gantry | Cartesian, simple kinematics, large workspace |
| Delta | Fast, parallel kinematics, need IK calculation |
| PLCopen | Standard FBs: MC_Power, MC_Home, MC_MoveAbsolute |
| Safety | STO, SS1, SLS, Area scanners |
| Optimization | Overlap motion, velocity blending, path optimization |
| PROFINET IRT | < 1μs jitter for synchronized multi-axis |

---

## Navigation

- [← Part 61: Large System Design](./part61_large_system_design.md)
- [→ Part 63: Welding Automation](./part63_welding_automation.md)
- [↑ สารบัญหลัก](../README.md)
