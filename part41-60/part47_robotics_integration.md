# Part 47: Robotics Integration (Steps 921-940)

## ภาพรวม
การ integrate หุ่นยนต์อุตสาหกรรมกับ PLC เป็นทักษะสำคัญในการสร้าง automated manufacturing cells เนื้อหานี้ครอบคลุมการสื่อสาร การควบคุมความปลอดภัย และการเขียนโปรแกรมสำหรับระบบ Robot-PLC

---

## Step 921: Robot Types

### ประเภทของหุ่นยนต์อุตสาหกรรม

```
Robot Types Overview:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  SCARA (Selective Compliance Assembly Robot Arm):           │
│  ├── 4 axes: X, Y, Z, rotation                            │
│  ├── Fast for horizontal movements                        │
│  ├── Ideal: PCB assembly, small parts handling            │
│  └── Examples: Epson T6, Yamaha RCX340                    │
│                                                             │
│  6-AXIS (Articulated):                                     │
│  ├── Full 6DOF flexibility                                │
│  ├── Can reach complex positions                          │
│  ├── Ideal: Welding, painting, general handling           │
│  └── Examples: FANUC R-2000, KUKA KR 120, ABB IRB 2600   │
│                                                             │
│  DELTA (Parallel Robot):                                   │
│  ├── 3-4 axes, parallel kinematics                       │
│  ├── Very fast (< 0.3s cycle)                            │
│  ├── Ideal: Food industry, small parts at high speed     │
│  └── Examples: ABB FlexPicker, FANUC M-3iA              │
│                                                             │
│  COLLABORATIVE (Cobot):                                    │
│  ├── Force sensing, safe around humans                   │
│  ├── Easy programming (teach by hand)                    │
│  ├── Ideal: Assembly, inspection, limited force tasks    │
│  └── Examples: Universal Robots UR5/10, FANUC CR-7iA    │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 922: Robot Safety Zones

### Safety Zone Configuration

```
Robot Safety Zones:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Zone 3: Restricted Access Area                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Zone 2: Warning Area                                │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  Zone 1: Robot Working Envelope               │  │  │
│  │  │  ┌─────────────────────────────────────────┐  │  │  │
│  │  │  │              ROBOT                       │  │  │  │
│  │  │  │         ┌──────────────┐                 │  │  │  │
│  │  │  │         │   ☆ Robot   │                 │  │  │  │
│  │  │  └─────────────────────────────────────────┘  │  │  │
│  │  │  Light curtain boundary                       │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │  Area scanner boundary                               │  │
│  └──────────────────────────────────────────────────────┘  │
│  Fence/Guarding                                             │
└─────────────────────────────────────────────────────────────┘

Zone 1: Light curtain / Safety mat
Zone 2: Area scanner (reduced speed)
Zone 3: Fence + Interlocked gate
```

### Safety Zone Integration ใน PLC

```ladder
// Safety Zone Logic for Robot Cell
// Network 1: Zone 1 - Emergency Stop (immediate)
//    LightCurtain_Beam_Break    Robot_Safe_Stop
//    ------[ ]---------------------------( )--------

+--[ LC_Beam_Break_OSSD ]--+--( Robot_STO_Command )--+
|   (Safety-rated signal)   |   (Safe Torque Off)     |
+---------------------------+------------------------+

// Network 2: Zone 2 - Speed Reduction
//    AreaScanner_Person_Detected    Robot_Slow_Speed
//    ------[ ]-------------------------------( )-----

+--[ Area_Scanner_Zone2 ]--+--( Robot_Speed_Override_20pct )--+

// Network 3: Gate Interlock
//    Gate_Open    Robot_Prevent_Start
//    ---[ ]------------------( )-------

+--[ Gate_Open_SW ]--+--( Robot_Auto_Mode_Inhibit )--+
```

---

## Step 923: PLC-Robot Communication Methods

### วิธีการสื่อสารระหว่าง PLC และ Robot

#### Method 1: Discrete I/O Handshaking

```
Discrete I/O Handshake Protocol:
┌─────────────────────────────────────────────────────────────┐
│  PLC → Robot (Outputs):                                     │
│  ├── Q0.0: Robot Start (cycle start command)               │
│  ├── Q0.1: Robot Reset                                     │
│  ├── Q0.2: Robot Program Select Bit 0                      │
│  ├── Q0.3: Robot Program Select Bit 1                      │
│  ├── Q0.4: Robot Program Select Bit 2                      │
│  └── Q0.5: Safety Enable (from safety system)             │
│                                                             │
│  Robot → PLC (Inputs):                                     │
│  ├── I0.0: Robot At Home                                   │
│  ├── I0.1: Robot Cycle Complete                            │
│  ├── I0.2: Robot Running                                   │
│  ├── I0.3: Robot Fault                                     │
│  └── I0.4: Part Gripped (gripper feedback)                │
│                                                             │
│  Handshake Sequence:                                       │
│  PLC: Q0.0=1 (Start)                                      │
│  Robot: I0.2=1 (Running)                                  │
│  PLC: Q0.0=0 (Reset Start)                               │
│  Robot: I0.1=1 (Complete), I0.2=0                        │
│  PLC: Note Complete                                       │
└─────────────────────────────────────────────────────────────┘
```

#### Method 2: PROFINET Communication

```
PROFINET Integration (เช่น KUKA iiQKA หรือ FANUC PROFINET):

// TIA Portal: Add Robot as PROFINET Device
// Robot configured as PROFINET IO Device

// Data Exchange (Process Data Object - PDO):
// PLC → Robot:  Cyclic data เช่น speed override, program number
// Robot → PLC: Cyclic data เช่น position, status, fault code

// Non-cyclic data (RDREC/WRREC):
// สำหรับ parameter setting และ diagnostic

// ตัวอย่าง: KUKA.ProfiNet module ใน TIA Portal
```

#### Method 3: EtherNet/IP

```
EtherNet/IP Integration (สำหรับ Rockwell PLC):

// Studio 5000: Add Robot as EtherNet/IP Device
// ตัวอย่าง: FANUC R-J3iC + EtherNet/IP option

// I/O Assembly:
// Input (Robot → PLC):  Assembly #1 (64 bytes)
//   Byte 0: Status (bit0=running, bit1=fault, bit2=done)
//   Byte 1: Current program number
//   Bytes 4-7: Joint 1 position (REAL)
//   Bytes 8-11: Joint 2 position (REAL)
//   ...
//
// Output (PLC → Robot): Assembly #2 (64 bytes)
//   Byte 0: Command (bit0=start, bit1=stop, bit2=reset)
//   Byte 1: Program number to run
//   Bytes 4-7: Speed override % (REAL)
```

---

## Step 924: Robot Program Call from PLC

### การเรียก Robot Program จาก PLC

```ladder
// Robot Program Selection and Start
// FB: Robot_Control

// Network 1: Program Selection (BCD encode to 3 bits)
// Program 1 = 001, Program 2 = 010, Program 3 = 011, etc.

+--[ Robot_Prog_Number_1 ]--+--( Robot_Prog_Bit0 )--+
+--[ Robot_Prog_Number_2 ]--+--( Robot_Prog_Bit1 )--+
+--[ Robot_Prog_Number_3 ]--+--( Robot_Prog_Bit0 )--+--( Robot_Prog_Bit1 )--+

// Network 2: Handshake - Start Robot Cycle
// Only start when robot is at home and ready

//   Robot_At_Home  Robot_Not_Fault  New_Part_Ready   Robot_Start
//   ---[ ]----------[ ]----------[ ]------------------( )--------

+--[ Robot_At_Home ]--+--[ NOT Robot_Fault ]--+--[ New_Part_Ready ]--+--( Robot_Start_Cmd )--+

// Network 3: Cycle Complete Detection (rising edge)
+--[R_TRIG Robot_Cycle_Complete]--+
|  CLK := Robot_Done_Input        |
|  Q   => Cycle_Complete_Pulse    |  // 1 scan pulse
+----------------------------------+

// Network 4: Part Counter
+--[ Cycle_Complete_Pulse ]--+--[CTU Parts_Counter]--+
|                             |   CU := Cycle_Complete|
|                             |   R  := Counter_Reset  |
|                             |   PV => Parts_Produced |
+----------------------------+------------------------+
```

---

## Step 925: Conveyor Tracking (Flying Synchronization)

### Robot Conveyor Tracking

```
Conveyor Tracking System:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Conveyor ──────────────────────────────────────────►      │
│  [Part] [Part]     [Part] [Part]   [Part]                  │
│                         ▲                                   │
│                    ┌────┴────┐                              │
│                    │  Robot  │                             │
│                    └────┬────┘                              │
│                         │                                   │
│  Steps:                 │                                   │
│  1. Encoder on conveyor measures position                  │
│  2. Vision system detects part and its position on belt    │
│  3. Robot calculates interception point                   │
│  4. Robot accelerates to match belt speed                 │
│  5. Pick part while moving with belt                      │
│  6. Robot decelerates and places part                     │
│                                                             │
│  Key parameters:                                           │
│  - Belt velocity: 0.3 m/s                                 │
│  - Part detection range: 500mm upstream                   │
│  - Pick window: 200mm                                     │
│  - Robot max speed: 2000mm/s                              │
└─────────────────────────────────────────────────────────────┘
```

### Conveyor Tracking Program Logic

```pascal
// Conveyor Tracking Logic
FB: ConveyorTracking

VAR_INPUT
    Part_Detected   : BOOL;      // Vision trigger
    Part_Position_X : REAL;      // Part X on belt (mm from encoder)
    Belt_Velocity   : REAL;      // Belt speed (mm/s)
    Robot_Ready     : BOOL;      // Robot ready for pick
END_VAR

VAR_OUTPUT
    Pick_Position_X : REAL;      // Target pick X
    Pick_Start      : BOOL;      // Start pick command
    Tracking_Active : BOOL;      // Currently tracking
END_VAR

VAR
    Robot_Approach_Time : REAL;  // Time for robot to reach pick zone
    Predicted_X        : REAL;  // Predicted part position at pick time
    Window_Start       : REAL;  // Start of pick window (mm)
    Window_End         : REAL;  // End of pick window (mm)
END_VAR

BEGIN
    IF Part_Detected AND Robot_Ready THEN
        
        // Robot approach time (constant, depends on motion profile)
        Robot_Approach_Time := 0.5;  // 500ms (pre-calculated)
        
        // Predicted position when robot arrives
        Predicted_X := Part_Position_X + Belt_Velocity * Robot_Approach_Time;
        
        // Check if pick is feasible within window
        Window_Start := 300.0;  // mm
        Window_End   := 500.0;  // mm
        
        IF Predicted_X >= Window_Start AND Predicted_X <= Window_End THEN
            Pick_Position_X := Predicted_X;
            Pick_Start := TRUE;
            Tracking_Active := TRUE;
        END_IF;
    END_IF;
END_FUNCTION_BLOCK
```

---

## Step 926: Robot Fault Handling

### การจัดการ Fault ของ Robot

```
Robot Fault Handling Strategy:
┌─────────────────────────────────────────────────────────────┐
│  Level 1: Warning (ไม่หยุดการผลิต)                          │
│  ├── Low air pressure warning                              │
│  ├── High temperature warning                             │
│  └── Action: Alert operator, log to historian             │
│                                                             │
│  Level 2: Recoverable Fault                                │
│  ├── Part drop (gripper miss)                             │
│  ├── Reach limit (singularity)                            │
│  └── Action: Stop cycle, retry up to N times, alert      │
│                                                             │
│  Level 3: Critical Fault (ต้องหยุดและ reset)               │
│  ├── Servo drive fault                                    │
│  ├── Collision detection                                  │
│  └── Action: Stop robot, require manual reset            │
│                                                             │
│  Level 4: Emergency (Safety-related)                       │
│  ├── Safety zone violation                               │
│  ├── E-Stop                                               │
│  └── Action: STO, require safety reset + inspector       │
└─────────────────────────────────────────────────────────────┘
```

### Fault Handling Ladder

```ladder
// Robot Fault Recovery Logic
// Network 1: Fault Detection and Classification
+--[ Robot_Fault_Input ]--+--[MUX Fault_Code]--+--( Set Fault_Level )--+
|   (Robot fault signal)   |   0 → Level 1      |                      |
|                          |   1 → Level 2      |                      |
|                          |   2 → Level 3      |                      |
|                          |   3 → Level 4      |                      |
+-------------------------+-------------------+----------------------+

// Network 2: Auto-retry for Level 2 faults
+--[ Fault_Level_2 ]--+--[ Retry_Count LT 3 ]--+--( Auto_Reset_Cmd )--+
|   (Level 2 fault)    |   (< 3 retries)         |   (Auto reset)      |
+--------------------+------------------------+---------------------+

// Network 3: Retry counter
+--[CTU Retry_CTU]--+
|  CU := Auto_Reset_Pulse |
|  R  := Fault_Cleared    |
|  CV => Retry_Count       |
+-------------------------+
```

---

## Step 927: Collaborative Robot (Cobot) Safety

### Cobot Safety Requirements (ISO/TS 15066)

```
Cobot Collaboration Modes:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Mode 1: Safety-rated monitored stop                       │
│  ├── Robot stops when human enters zone                   │
│  ├── Robot resumes when human leaves                      │
│  └── No direct contact intended                           │
│                                                             │
│  Mode 2: Hand guiding                                      │
│  ├── Operator physically guides robot (hand teaching)     │
│  ├── Speed < 250mm/s                                      │
│  └── Force sensor or enabling device required             │
│                                                             │
│  Mode 3: Speed and separation monitoring                   │
│  ├── Robot slows as human approaches                      │
│  ├── Stops when human < minimum distance                  │
│  └── Distance sensor (lidar/vision) required             │
│                                                             │
│  Mode 4: Power and force limiting (PFL)                   │
│  ├── Robot can contact human                              │
│  ├── Max force: 140N transient, 65N quasi-static         │
│  └── Max pressure: 190N/cm² transient, 110N/cm² static  │
│                                                             │
│  For UR (Universal Robots):                               │
│  ├── Built-in force sensing                              │
│  ├── Configurable safety planes                          │
│  └── Certified SIL 2 / PL d                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 928-940: Complete Pick-and-Place Integration Example

### โครงการ: FANUC Robot + Siemens S7-1500 Integration

#### 1. System Architecture

```
System Architecture:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Siemens S7-1515 PLC                                       │
│  ├── Safety: S7-1515F (F-CPU for safety functions)        │
│  ├── HMI: TP1500 (15" Comfort Panel)                     │
│  └── Communication to FANUC R30iB Controller              │
│                                                             │
│  FANUC R2000iC/125L Robot                                  │
│  ├── 6 axes                                               │
│  ├── Payload: 125kg                                       │
│  └── Reach: 3100mm                                        │
│                                                             │
│  Communication:                                             │
│  ├── PROFINET: S7-1515 → FANUC (I/O and data)            │
│  ├── Safety: PROFIsafe (E-Stop, enable, safety I/O)       │
│  └── Area Scanner → F-DI → F-CPU                         │
│                                                             │
│  I/O:                                                      │
│  ├── PLC Digital I/O: 32 DI + 32 DO                      │
│  ├── Robot I/O: 64 DI + 64 DO (via PROFINET)             │
│  └── Safety: F-DI 8ch + F-DO 4ch                         │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Signal List (Interface)

```
PLC ←→ Robot Interface:
┌─────────────────────────────────────────────────────────────┐
│  PLC → FANUC (Outputs):                                    │
│  Q Byte  Bit  Signal Name          Description              │
│  ──────  ───  ──────────────────   ────────────────────── │
│  QB100   0    Robot_Prog_Bit0      Program select bit 0   │
│  QB100   1    Robot_Prog_Bit1      Program select bit 1   │
│  QB100   2    Robot_Prog_Bit2      Program select bit 2   │
│  QB100   3    Robot_Start          Cycle start            │
│  QB100   4    Robot_Reset          Fault reset            │
│  QB100   5    Robot_Home_Cmd       Go to home position    │
│  QB100   6    Robot_Speed_100      Speed 100%             │
│  QB100   7    Robot_Speed_50       Speed 50% override     │
│                                                             │
│  FANUC → PLC (Inputs):                                     │
│  IB Byte Bit  Signal Name          Description             │
│  ──────  ───  ──────────────────   ────────────────────── │
│  IB100   0    Robot_At_Home        At home position       │
│  IB100   1    Robot_Auto_Mode      In Auto mode           │
│  IB100   2    Robot_Running        Cycle running          │
│  IB100   3    Robot_Cycle_Done     Cycle complete         │
│  IB100   4    Robot_Fault_Ext      External fault         │
│  IB100   5    Robot_Alarm          Alarm active           │
│  IB100   6    Gripper_Closed       Gripper closed fb      │
│  IB100   7    Part_In_Gripper      Part detected in grip  │
└─────────────────────────────────────────────────────────────┘
```

#### 3. Complete Integration Program (TIA Portal)

```pascal
// =====================================================
// FB: Robot_Cell_Controller
// FANUC Robot + S7-1515 Integration
// =====================================================

FUNCTION_BLOCK Robot_Cell_Controller
VAR_INPUT
    // HMI Commands
    HMI_Start_Auto    : BOOL;   // เริ่ม auto mode
    HMI_Stop          : BOOL;   // หยุด
    HMI_Reset_Fault   : BOOL;   // Reset fault
    HMI_Robot_Reset   : BOOL;   // Reset robot alarm
    
    // Sensors
    Part_On_Conveyor  : BOOL;   // Part detected on input conveyor
    Place_Position_OK : BOOL;   // Destination clear
    
    // Robot Feedback (from PROFINET)
    Robot_At_Home     : BOOL;
    Robot_Auto_Mode   : BOOL;
    Robot_Running     : BOOL;
    Robot_Cycle_Done  : BOOL;
    Robot_Fault       : BOOL;
    Robot_Alarm       : BOOL;
    Part_In_Gripper   : BOOL;
    
    // Safety (from F-CPU via standard signal)
    Safety_OK         : BOOL;   // Safety system OK
END_VAR

VAR_OUTPUT
    // Robot Commands (to PROFINET)
    Robot_Start       : BOOL;
    Robot_Reset       : BOOL;
    Robot_Prog_Num    : BYTE;   // Program number
    
    // Cell Outputs
    Conveyor_Run      : BOOL;   // Input conveyor enable
    
    // Status
    Cell_Running      : BOOL;
    Cell_Fault        : BOOL;
    Cycle_Count       : DINT;
    Fault_Description : STRING[80];
END_VAR

VAR
    Cell_State        : INT := 0;
    Robot_Done_Edge   : R_TRIG;
    Fault_Latch       : BOOL;
    Start_Edge        : R_TRIG;
END_VAR

BEGIN
    // Edge detection
    Robot_Done_Edge(CLK := Robot_Cycle_Done);
    Start_Edge(CLK := HMI_Start_Auto);
    
    // Fault Reset
    IF HMI_Reset_Fault THEN
        Fault_Latch := FALSE;
        Cell_Fault := FALSE;
        Fault_Description := '';
    END_IF;
    
    // Robot Alarm Reset
    IF HMI_Robot_Reset THEN
        Robot_Reset := TRUE;
    ELSIF NOT Robot_Alarm THEN
        Robot_Reset := FALSE;
    END_IF;
    
    // Main State Machine
    CASE Cell_State OF
    
        0: // IDLE
            Cell_Running := FALSE;
            Robot_Start := FALSE;
            Conveyor_Run := FALSE;
            
            IF Start_Edge.Q AND Safety_OK AND NOT Fault_Latch THEN
                Cell_State := 10;
            END_IF;
        
        10: // CHECK ROBOT READY
            IF NOT Robot_At_Home THEN
                // Send robot home first
                Robot_Prog_Num := 255;   // Home program
                Robot_Start := TRUE;
                Cell_State := 11;
            ELSIF Robot_At_Home AND Robot_Auto_Mode THEN
                Cell_State := 20;
            END_IF;
        
        11: // WAIT FOR ROBOT HOME
            IF Robot_Cycle_Done AND Robot_At_Home THEN
                Robot_Start := FALSE;
                Cell_State := 20;
            END_IF;
        
        20: // WAIT FOR PART (Run conveyor)
            Cell_Running := TRUE;
            Conveyor_Run := TRUE;
            
            IF Part_On_Conveyor THEN
                Conveyor_Run := FALSE;   // Stop conveyor (part stopped)
                Cell_State := 30;
            END_IF;
        
        30: // START ROBOT CYCLE
            IF Robot_At_Home AND NOT Robot_Running AND Safety_OK THEN
                Robot_Prog_Num := 1;    // Pick and place program #1
                Robot_Start := TRUE;
                Cell_State := 40;
            END_IF;
        
        40: // WAIT ROBOT RUNNING CONFIRMATION
            IF Robot_Running THEN
                Robot_Start := FALSE;   // Reset start signal
                Cell_State := 50;
            ELSIF NOT Robot_Running AND Robot_Fault THEN
                Fault_Latch := TRUE;
                Fault_Description := 'Robot failed to start';
                Cell_State := 90;
            END_IF;
        
        50: // WAIT CYCLE COMPLETE
            IF Robot_Done_Edge.Q THEN
                // Cycle completed
                IF Part_In_Gripper THEN
                    // Robot picked but didn't place? Check
                    Fault_Latch := TRUE;
                    Fault_Description := 'Part not released at destination';
                    Cell_State := 90;
                ELSE
                    Cycle_Count := Cycle_Count + 1;
                    Cell_State := 20;   // Ready for next part
                END_IF;
            END_IF;
            
            // Safety interrupt during cycle
            IF NOT Safety_OK THEN
                Cell_State := 80;
            END_IF;
        
        80: // SAFETY STOP
            Cell_Running := FALSE;
            Conveyor_Run := FALSE;
            Robot_Start := FALSE;
            IF Safety_OK THEN
                Cell_State := 20;   // Resume when safe
            END_IF;
        
        90: // FAULT STATE
            Cell_Running := FALSE;
            Cell_Fault := TRUE;
            Conveyor_Run := FALSE;
            Robot_Start := FALSE;
            IF NOT Fault_Latch THEN
                Cell_State := 0;
            END_IF;
        
    END_CASE;
    
    // Robot Fault Monitoring (all states)
    IF Robot_Fault OR Robot_Alarm THEN
        IF Cell_State >= 30 AND Cell_State <= 60 THEN
            Fault_Latch := TRUE;
            Fault_Description := 'Robot fault during cycle';
            Cell_State := 90;
        END_IF;
    END_IF;
    
    // E-Stop override
    IF NOT Safety_OK AND Cell_State > 0 AND Cell_State <> 80 THEN
        Cell_State := 80;
    END_IF;
    
END_FUNCTION_BLOCK
```

#### 4. FANUC Robot Program (KAREL/TP)

```
FANUC TP Program: PR001_PickPlace.TP
(สำหรับ reference - ฝั่ง Robot controller)

/PROG  PR001_PickPlace
/ATTR
OWNER           = MNEDITOR;
COMMENT         = "Pick and Place Program";
PROG_SIZE       = 1024;
/MN
   1:  UFRAME_NUM=1 ;
   2:  UTOOL_NUM=1 ;
   3:  OVERRIDE=100% ;
   
   : !-- Move to approach position --
   4:  J P[HOME] 100% CNT0 ;
   5:  DO[101:Cycle_Running]=ON ;
   
   : !-- Move to pick approach --
   6:  L P[PICK_APPROACH] 1000mm/sec CNT50 ;
   
   : !-- Descend to pick position --
   7:  L P[PICK_POS] 200mm/sec FINE ;
   
   : !-- Activate gripper --
   8:  DO[102:Gripper_Close]=ON ;
   9:  WAIT 0.30sec ;
   10: IF DI[102:Gripper_Closed]=OFF THEN JMP LBL[99] ;
   
   : !-- Ascend with part --
   11: L P[PICK_APPROACH] 500mm/sec CNT50 ;
   
   : !-- Move to place approach --
   12: L P[PLACE_APPROACH] 1500mm/sec CNT50 ;
   
   : !-- Descend to place --
   13: L P[PLACE_POS] 200mm/sec FINE ;
   
   : !-- Release gripper --
   14: DO[102:Gripper_Close]=OFF ;
   15: WAIT 0.30sec ;
   
   : !-- Ascend --
   16: L P[PLACE_APPROACH] 500mm/sec CNT50 ;
   
   : !-- Return home --
   17: J P[HOME] 100% FINE ;
   18: DO[103:Cycle_Done]=ON ;
   19: WAIT 0.10sec ;
   20: DO[103:Cycle_Done]=OFF ;
   21: DO[101:Cycle_Running]=OFF ;
   22: END ;
   
   LBL[99]:
   : !-- Gripper fault --
   23: DO[101:Cycle_Running]=OFF ;
   24: DO[104:Robot_Fault_Ext]=ON ;
   25: PAUSE ;
/END
```

---

## Lab Project 47: Complete Robot Cell Integration

### โจทย์
ออกแบบและ implement Robot Cell สำหรับ Engine Block Machining

### ระบบที่ต้องออกแบบ
1. Input conveyor (รับ rough engine block)
2. 6-axis robot (load/unload จาก CNC machine)
3. CNC machining center (ทำการกลึง)
4. Inspection station (CMM หรือ vision check)
5. Output conveyor (ส่ง finished parts)

### งานที่ต้องทำ
1. วาด System Layout และ Material Flow
2. ออกแบบ Signal Interface (PLC ↔ Robot ↔ CNC)
3. เขียน Cell Controller Program
4. ออกแบบ Robot Safety Zones ตาม ISO 10218
5. Implement Production Statistics (OEE calculation)
6. ทำ Fault Tree Analysis สำหรับ cell

---

## สรุป Part 47

| วิธีการสื่อสาร | ความเร็ว | ความซับซ้อน | ใช้กับ |
|--------------|---------|-----------|-------|
| Discrete I/O | ช้า | ง่าย | ระบบเก่า, Budget |
| PROFINET | เร็ว | ปานกลาง | Siemens ecosystem |
| EtherNet/IP | เร็ว | ปานกลาง | Rockwell ecosystem |
| DeviceNet | ปานกลาง | ปานกลาง | Rockwell legacy |
| RS-232/485 | ช้า | ยาก | ระบบเก่ามาก |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 46: Advanced Motion Control](part46_advanced_motion.md) |
| ► ถัดไป | [Part 48: Vision System Integration](part48_vision_system.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
