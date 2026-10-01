# Part 63: Welding Automation (Steps 1241-1260)

## บทนำ: Welding Automation ในอุตสาหกรรม

Welding automation เป็นส่วนสำคัญของอุตสาหกรรม:
- **Automotive**: ตัวถังรถยนต์ (3,000-5,000 welds per vehicle)
- **Shipbuilding**: โครงสร้างเหล็กขนาดใหญ่
- **Pressure vessels**: หม้อต้มน้ำ, ถังแก๊ส (ASME)
- **Pipeline**: ท่อส่งน้ำมัน, ก๊าซ

ประโยชน์:
- ความสม่ำเสมอ (consistency): ทุก weld เหมือนกัน
- ความเร็ว: robot ทำงาน 24/7
- คุณภาพ: ควบคุม parameters แม่นยำ

---

## Step 1241: Welding Process Overview

### ประเภทของการเชื่อม

```
WELDING PROCESSES ที่ใช้ใน AUTOMATION:
──────────────────────────────────────────────────────────────

1. MIG/MAG WELDING (Metal Inert/Active Gas):
   ═══════════════════════════════════
   ใช้: Wire electrode + shielding gas
   Gas: CO2 (MAG), Argon/CO2 mix (MIG)
   Application: Carbon steel, stainless, aluminum
   Automation: ง่ายมาก (wire feed, stable arc)
   ตัวอย่าง: ตัวถังรถยนต์, เฟอร์นิเจอร์เหล็ก
   
   PLC Controls:
   ├── Wire feed speed (0-18 m/min)
   ├── Welding voltage (14-45V)
   ├── Gas flow rate (8-20 L/min)
   └── Travel speed (robot/gantry speed)

2. TIG WELDING (Tungsten Inert Gas / GTAW):
   ═══════════════════════════════════════
   ใช้: Non-consumable W electrode + filler wire (optional)
   Gas: Argon (pure)
   Application: Stainless, titanium, precision welds
   Automation: ยากกว่า MIG (torch angle critical)
   ตัวอย่าง: Pharmaceutical piping (hygienic), aerospace
   
3. SPOT WELDING (Resistance Welding):
   ══════════════════════════════════
   ใช้: High current through clamped metal sheets
   ไม่ต้องมี filler metal
   Application: Sheet metal (car bodies, appliances)
   Automation: มาก (robot spot welder ทั่วไป)
   PLC Controls: Current, time, electrode force
   
4. LASER WELDING:
   ═══════════════
   ใช้: Focused laser beam
   Application: Precision, thin materials, hermetic seals
   Automation: ง่าย (non-contact, fast, precise)
   PLC Controls: Laser power, pulse frequency, speed

5. PLASMA ARC WELDING:
   ═══════════════════
   ใช้: Constricted arc (hotter than TIG)
   Application: Thick materials, faster than TIG
```

---

## Step 1242: Welding Parameter Control

### การควบคุมพารามิเตอร์การเชื่อม

```
WELDING PARAMETERS สำหรับ MIG WELDING:
──────────────────────────────────────────────────────────────

Key Parameters:
1. Wire Feed Speed (WFS): mm/min หรือ m/min
   ↑ WFS → ↑ Amperage, ↑ Deposition rate
   Too high → porosity, spatter
   Too low → cold weld, lack of fusion

2. Voltage: Volts
   ↑ Voltage → ↑ Arc length, wider bead
   Too high → undercut, porosity
   Too low → stubbing, high spatter

3. Travel Speed: mm/min (robot speed)
   ↑ Speed → thinner bead, less heat input
   Too fast → cold weld, incomplete fusion
   Too slow → burn-through, wide bead

4. Contact Tip to Work Distance (CTWD): mm
   Standard: 15-20mm
   ↑ CTWD → ↑ effective resistance → changes arc behavior

5. Shielding Gas Flow: L/min
   Standard: 12-18 L/min
   Too low → porosity (atmospheric contamination)
   Too high → turbulence → porosity

// DATA BLOCK สำหรับ Welding Parameters
DATA_BLOCK DB_WeldParams
    STRUCT
        rWireSpeed_mpm:     REAL;   // Wire feed speed (m/min)
        rVoltage_V:         REAL;   // Welding voltage (V)
        rTravelSpeed_mmpm:  REAL;   // Travel speed (mm/min)
        rGasFlow_Lpm:       REAL;   // Gas flow (L/min)
        rPreheat_C:         REAL;   // Preheat temperature (°C)
        rInterpass_C:       REAL;   // Interpass temperature (°C)
        iWeldProgram:       INT;    // Welding program number
    END_STRUCT;
END_DATA_BLOCK
```

---

## Step 1243: Weld Program Management

### จัดการ Weld Programs / Recipes

```
WELD PROGRAM MANAGEMENT:
──────────────────────────────────────────────────────────────

Weld Program = ชุดพารามิเตอร์สำหรับงานเชื่อมชิ้นหนึ่ง
เก็บใน WPS (Welding Procedure Specification) → เก็บใน PLC DB

// Weld Program Database
DATA_BLOCK DB_WeldPrograms
    programs: ARRAY[1..100] OF STRUCT
        sName:          STRING[20];     // Program name
        sMaterial:      STRING[20];     // Material type
        sJointType:     STRING[10];     // Butt, Fillet, Corner
        rThickness_mm:  REAL;           // Material thickness
        rWireSpeed:     REAL;           // m/min
        rVoltage:       REAL;           // V
        rTravelSpeed:   REAL;           // mm/min
        rGasFlow:       REAL;           // L/min
        rPreheat_Min:   REAL;           // Min preheat °C
        rInterpass_Max: REAL;           // Max interpass °C
        rHeatInput_kJ:  REAL;           // Target heat input kJ/mm
        bQualified:     BOOL;           // WPS qualified?
        sWPS_Number:    STRING[15];     // WPS reference
        sApproved_By:   STRING[20];     // Approved by
        dtApproval:     DATE_AND_TIME;  // Approval date
    END_STRUCT;
END_DATA_BLOCK

// Heat Input Calculation:
// HI (kJ/mm) = (Amps × Volts × 60) / (1000 × Travel_Speed_mm/min)
// rHeatInput := (rAmps × rVoltage × 60.0) / (1000.0 × rTravelSpeed);

// CODE COMPLIANCE:
// AWS D1.1 (Structural Steel): qualified WPS required
// ASME Section IX (Pressure vessel): PQR + WPS
// EN ISO 15614 (European): WPQR + WPS
```

---

## Step 1244: Preheat and Interpass Temperature Control

### ควบคุมอุณหภูมิ

```
PREHEAT AND INTERPASS TEMPERATURE:
──────────────────────────────────────────────────────────────

ทำไมต้องควบคุมอุณหภูมิ:
- Preheat (ก่อนเชื่อม): ป้องกัน hydrogen cracking ในเหล็กหนา/high carbon
- Interpass (ระหว่าง passes): จำกัด heat input → ป้องกัน HAZ degradation

// Preheat Check Before Welding
FUNCTION_BLOCK FB_PreheatCheck
VAR_INPUT
    rActualTemp_C:  REAL;           // Measured preheat temp
    rRequired_C:    REAL;           // Required preheat from WPS
    rTolerance_C:   REAL := 25.0;   // Acceptable above required
END_VAR
VAR_OUTPUT
    bPreheatOK:     BOOL;           // OK to weld
    sPreheatStatus: STRING[40];     // Status message
END_VAR

// CHECK:
// MINIMUM preheat: rActualTemp >= rRequired_C
// MAXIMUM preheat: rActualTemp <= rRequired_C + 150°C (too hot also bad)
// IF rActualTemp < rRequired_C THEN
//     bPreheatOK := FALSE;
//     sPreheatStatus := 'Preheat too low - Heat required';
// ELSIF rActualTemp > (rRequired_C + 150.0) THEN
//     bPreheatOK := FALSE;
//     sPreheatStatus := 'Preheat too high - Wait to cool';
// ELSE
//     bPreheatOK := TRUE;
//     sPreheatStatus := 'Preheat OK';
// END_IF;

// Interpass Temperature Check (between weld passes):
// rInterpass_Max_C := DB_WeldParams.rInterpass_Max;
// ถ้า temperature > max → ห้าม start next pass
// rInterpass_Actual := Read from pyrometer or IR sensor

// PLC OUTPUT: 
// ถ้า Preheat < required → enable heating blanket/torch
// ถ้า Interpass > max → wait (stop robot, alert operator)
```

---

## Step 1245: MIG Welding Automation with PLC

### ควบคุม MIG Welding

```
MIG WELDER INTERFACE (FRONIUS CMT / LINCOLN PowerWave):
──────────────────────────────────────────────────────────────

Modern Welding Power Sources ส่วนใหญ่มี Digital Interface:
├── Fronius CMT: Fieldbus (PROFINET, EtherNet/IP)
├── Lincoln PowerWave: DeviceNet, EtherNet/IP
├── ESAB Aristo: PROFINET
└── Kemppi FastMig: EtherNet

PROFINET Interface Register Map (Fronius ตัวอย่าง):
Input to PLC (from welder):
├── Status word (16-bit)
│   bit 0: Power source ready
│   bit 1: Arc established
│   bit 2: Error
│   bit 3: Remote mode active
├── Actual current (REAL, Amps)
├── Actual voltage (REAL, Volts)
├── Wire feed speed actual (REAL, m/min)
└── Error code (INT)

Output from PLC (to welder):
├── Control word (16-bit)
│   bit 0: Enable/Disable
│   bit 1: Weld start
│   bit 2: Gas purge
│   bit 3: Wire inching (forward)
│   bit 4: Wire inching (backward)
├── Program number (INT, selects WPS)
├── Wire speed setpoint (REAL, m/min)
└── Voltage setpoint (REAL, V)

// FB_MIGWelder Interface
FUNCTION_BLOCK FB_MIGWelder_FRONIUS
VAR_INPUT
    bEnable:        BOOL;
    bWeldStart:     BOOL;       // Rising edge starts arc
    bWeldStop:      BOOL;       // Stop welding
    iProgram:       INT;        // Welding program number
    rWireSpeed:     REAL;       // Override wire speed
    rVoltage:       REAL;       // Override voltage
    bGasPurge:      BOOL;       // Gas purge command
END_VAR
VAR_OUTPUT
    bReady:         BOOL;       // Welder ready
    bArcOn:         BOOL;       // Arc established
    bError:         BOOL;       // Error
    rActualAmps:    REAL;       // Actual current
    rActualVolts:   REAL;       // Actual voltage
    iErrorCode:     INT;        // Error code
END_VAR
VAR
    // PROFINET I/O addresses (configured in TIA Portal)
    wStatusWord AT %IW100:  WORD;
    rActAmps    AT %ID104:  REAL;
    rActVolts   AT %ID108:  REAL;
    wControlWord AT %QW100: WORD;
    iProgram_Q  AT %QW102:  INT;
    rWireSpd_Q  AT %QD104:  REAL;
    rVolt_Q     AT %QD108:  REAL;
END_VAR

// Control logic:
// bReady := wStatusWord.0;    // Bit 0
// bArcOn := wStatusWord.1;    // Bit 1
// bError := wStatusWord.2;    // Bit 2
// 
// wControlWord.0 := bEnable;
// wControlWord.1 := bWeldStart;
// wControlWord.2 := bGasPurge;
// 
// iProgram_Q := iProgram;
// rWireSpd_Q := rWireSpeed;
// rVolt_Q    := rVoltage;
```

---

## Step 1246: Spot Welding Control

### Resistance Spot Welding

```
SPOT WELDING CONTROL:
──────────────────────────────────────────────────────────────

Spot Welding Process Sequence:
1. SQUEEZE: ขั้ว electrode กด material (ปรับ force)
2. WELD: ไฟฟ้าผ่าน → ความร้อน → nugget forms
3. HOLD: กด electrode ค้าง → nugget cools under pressure
4. OFF: ปล่อย electrode, move to next spot

// Timing Diagram:
Time:    0   10  20  30  40  50  60  70ms
Electrode: ←──SQUEEZE──→←─WELD─→←HOLD→←OFF
Current:           ←──ON──→
Force:   ←──────────────────────────→←

Spot Weld Parameters:
├── Squeeze time: 10-50ms (allow electrode to seat)
├── Weld current: 5,000-15,000 Amps (depends on material)
├── Weld time: 4-20 cycles (1 cycle = 20ms at 50Hz)
├── Hold time: 10-30ms
└── Electrode force: 200-600 daN

// SPOT WELDER FB
FUNCTION_BLOCK FB_SpotWelder
VAR_INPUT
    bExecute:       BOOL;           // Start spot weld
    iWeldProgram:   INT;            // Program number
END_VAR
VAR_OUTPUT
    bComplete:      BOOL;           // Weld complete
    bFault:         BOOL;           // Weld fault
    rActualEnergy_J: REAL;          // Energy delivered
    bNuggetOK:      BOOL;           // Weld quality OK
END_VAR
VAR
    iState:         INT := 0;
    tSqueezeTimer:  TON;
    tWeldTimer:     TON;
    tHoldTimer:     TON;
    rWeldEnergy:    REAL;           // Accumulated energy
END_VAR

// STATE MACHINE:
// 0: IDLE
// 1: SQUEEZE (electrode closing)
// 2: WELDING (current ON)
// 3: HOLD (current OFF, electrode ON)
// 4: OFF (complete)
// 5: FAULT

CASE iState OF
    0: bComplete := FALSE; bFault := FALSE;
       IF bExecute THEN iState := 1; END_IF;
    
    1: // SQUEEZE: close electrode
       xElectrodeClose := TRUE;
       tSqueezeTimer(IN := TRUE, PT := T#20ms);
       IF tSqueezeTimer.Q THEN tSqueezeTimer(IN:=FALSE); iState := 2; END_IF;
    
    2: // WELD: turn on welding current
       xWeldCurrent := TRUE;
       tWeldTimer(IN := TRUE, PT := T#16ms); // 16ms = ~1 cycle
       rWeldEnergy := rWeldEnergy + (rActualCurrent × rActualVoltage × 0.001);
       
       IF tWeldTimer.Q THEN
           xWeldCurrent := FALSE;
           tWeldTimer(IN := FALSE);
           iState := 3;
       END_IF;
       
       // Fault: current too low → poor weld
       IF rActualCurrent < 8000.0 THEN
           xWeldCurrent := FALSE;
           iState := 5;  // FAULT
       END_IF;
    
    3: // HOLD
       tHoldTimer(IN := TRUE, PT := T#20ms);
       IF tHoldTimer.Q THEN
           tHoldTimer(IN := FALSE);
           xElectrodeClose := FALSE;
           bNuggetOK := rWeldEnergy > rMinEnergy_J;
           bComplete := TRUE;
           iState := 0;
       END_IF;
    
    5: // FAULT
       xElectrodeClose := FALSE;
       xWeldCurrent := FALSE;
       bFault := TRUE;
       IF NOT bExecute THEN iState := 0; END_IF;
END_CASE;
```

---

## Step 1247: Weld Quality Monitoring

### ตรวจสอบคุณภาพ Weld

```
WELD QUALITY MONITORING:
──────────────────────────────────────────────────────────────

In-Process Quality Indicators:
1. Weld current/voltage profile: เทียบกับ reference
2. Wire feed speed: สม่ำเสมอไหม?
3. Arc stability: arc length variation
4. Spatter detection: ระดับ spatter สูงไหม?
5. Torch travel speed: ตรงตาม commanded speed?

Post-Weld Quality (สำหรับ spot welding):
1. Weld energy: total joules delivered
2. Expulsion detection: เสียง/flash เมื่อ metal expels
3. Electrode wear: ตรวจ tip dressing cycle count

// ONLINE WELD MONITOR
DATA_BLOCK DB_WeldQuality
    // Per weld record
    welds: ARRAY[1..10000] OF STRUCT
        iWeldID:        DINT;           // Unique weld ID
        dtTimestamp:    DATE_AND_TIME;  // When welded
        iProgram:       INT;            // WPS program used
        rActual_Current:REAL;           // Average current
        rActual_Voltage:REAL;           // Average voltage
        rActual_Speed:  REAL;           // Travel speed
        rHeatInput:     REAL;           // Calculated kJ/mm
        bInTolerance:   BOOL;           // Within spec?
        iAlarmCode:     INT;            // 0=OK, other=fault
    END_STRUCT;
    iWeldCount:     DINT;           // Total welds since reset
    iBadWeldCount:  DINT;           // Out-of-spec welds
    rYield_Pct:     REAL;           // % good welds
END_DATA_BLOCK

// Calculate and log each weld:
// rHeatInput = (rCurrent × rVoltage × 0.06) / rTravelSpeed;
// (kJ/mm) = (A × V × 0.06) / (mm/min)
// ← ×0.06 = 60 seconds/min ÷ 1000 W/kW

// Check tolerance:
// rHeatInput within ±15% of target → OK
// IF ABS(rHeatInput - rTarget) / rTarget > 0.15 THEN
//     bInTolerance := FALSE;
//     iBadWeldCount := iBadWeldCount + 1;
// ELSE
//     bInTolerance := TRUE;
// END_IF;
```

---

## Step 1248: Welding Sequence and Distortion Control

### ลำดับการเชื่อมและป้องกัน Distortion

```
WELDING SEQUENCE สำหรับ DISTORTION CONTROL:
──────────────────────────────────────────────────────────────

Distortion (การบิดตัว) เกิดจาก:
- ความร้อนสูงในจุดหนึ่ง → metal expands → เมื่อเย็นตัว → บิด

วิธีลด Distortion:

1. BACKSTEP SEQUENCE:
ทิศทาง: → → →
Weld sequence: ← ← ← (เชื่อมย้อนทิศ)
Segment 3→2→1 แทน 1→2→3
ทำให้ heat distribution สม่ำเสมอกว่า

2. SKIP WELDING:
เว้นระยะ: 1 - skip - 2 - skip - 3
ทำให้ cooling ระหว่าง welds มากขึ้น

3. BALANCED WELDING (สำหรับ symmetric structures):
เชื่อม Side A → Side B → Side A alternately
ป้องกันการบิดไปด้านเดียว

4. PRE-BENDING:
ดัด workpiece เล็กน้อย ก่อนเชื่อม
เมื่อบิดตัวหลังเชื่อม → ได้รูปร่างที่ต้องการ

// PLC Sequence Control:
DATA_BLOCK DB_WeldSequence
    jointCount: INT := 6;       // Total joints
    sequence: ARRAY[1..6] OF INT := [3, 6, 1, 4, 2, 5]; // Optimized order
    currentJoint: INT := 1;
    allComplete: BOOL;
END_DATA_BLOCK

// เชื่อมตาม sequence:
// FOR i := 1 TO jointCount DO
//     ExecuteWeld(sequence[i]);  // เชื่อม joint ที่กำหนดใน sequence
// END_FOR;
```

---

## Step 1249: Robotic Welding Cell Design

### ออกแบบ Robotic Welding Cell

```
ROBOTIC WELDING CELL LAYOUT:
──────────────────────────────────────────────────────────────

Typical Components:
┌─────────────────────────────────────────────────────────────┐
│         SAFETY FENCE (full perimeter)                        │
│  ┌──────────┐           ┌──────────────────────────────┐   │
│  │ Operator │           │     ROBOT (6-axis)            │   │
│  │ Station  │           │     KUKA/Fanuc/ABB/Yaskawa    │   │
│  │ (HMI)    │           └──────────┬───────────────────┘   │
│  └──────────┘                      │ Welding Torch          │
│                                    ▼                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           TURNTABLE (2-station positioner)            │   │
│  │   Station A ←── [Fixturing] ──→ Station B            │   │
│  │   (welding)                    (loading/unloading)   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────┐   ┌──────────────────┐                  │
│  │ Wire Feeder  │   │  Welding PS       │  (behind fence)  │
│  │ FRONIUS/     │   │  Fronius/Lincoln  │                  │
│  │ Lincoln      │   │                  │                   │
│  └──────────────┘   └──────────────────┘                  │
└─────────────────────────────────────────────────────────────┘

Turntable advantages:
├── Operator loads Station B while robot welds Station A
├── ≈100% robot utilization (no waiting for loading)
└── Cycle time: max(robot_weld_time, operator_load_time)
```

---

## Step 1250: PLC Interface กับ Robot Controller

### PLC สื่อสารกับ Robot

```
PLC ↔ ROBOT COMMUNICATION:
──────────────────────────────────────────────────────────────

Modern robot controllers มี standard interfaces:

1. Digital I/O (Hardwired):
   PLC → Robot: Start program, Program select, E-stop OK
   Robot → PLC: Program running, Complete, Error, At home
   Simple แต่ limited

2. PROFINET / EtherNet/IP:
   ส่ง: Program number, Tool data, Workpiece ID
   รับ: Position, Speed, Status, Production count
   ใช้กันมากใน modern cells

3. OPC UA:
   Full data exchange
   ใช้ใน smart factory / digital twin applications

// PLC-Robot Interface (PROFINET):
DATA_BLOCK DB_RobotInterface
    // PLC → Robot (Command)
    toPLC_Command:      WORD;       // Command bits
    toPLC_ProgramNum:   INT;        // Job/program to run
    toPLC_WorkpieceID:  DINT;       // Workpiece identifier
    
    // Robot → PLC (Status)
    fromRobot_Status:   WORD;       // Status bits
    fromRobot_Error:    INT;        // Error code
    fromRobot_JobNum:   INT;        // Current job
    fromRobot_CycleTime:REAL;       // Last cycle time (sec)
    fromRobot_WeldCount:DINT;       // Welds completed
END_DATA_BLOCK

// Command bits (PLC → Robot):
// bit 0: Start program
// bit 1: Stop (pause)
// bit 2: Abort
// bit 3: Reset error
// bit 4: Acknowledge next step

// Status bits (Robot → PLC):
// bit 0: Ready (at home, ready to start)
// bit 1: Running
// bit 2: Complete
// bit 3: Error/Fault
// bit 4: Emergency stop
// bit 5: Waiting for PLC input
```

---

## Step 1251: Fixture and Clamping Automation

### ควบคุม Fixture Clamps

```
FIXTURE CLAMPING AUTOMATION:
──────────────────────────────────────────────────────────────

Welding fixtures คล้ำ workpiece ก่อนเชื่อม
PLC ควบคุม pneumatic clamps

// Typical Fixture:
// - 12 clamps (6 horizontal, 6 vertical)
// - Locating pins (2-3 pins)
// - Proximity sensors: confirm part seated

FUNCTION_BLOCK FB_WeldFixture
VAR_INPUT
    bClampAll:      BOOL;           // Clamp all
    bUnclampAll:    BOOL;           // Unclamp all
    bCheckPresence: BOOL;           // Verify part seated
END_VAR
VAR_OUTPUT
    bAllClamped:    BOOL;           // All clamps confirmed closed
    bAllUnclamped:  BOOL;           // All clamps open
    bPartSeated:    BOOL;           // Part correctly positioned
    bReadyToWeld:   BOOL;           // Safe to start welding
    iMissingClamp:  INT;            // Which clamp not confirmed (0=all OK)
END_VAR
VAR
    xClamps:        ARRAY[1..12] OF BOOL;  // Clamp outputs
    xSensors:       ARRAY[1..12] OF BOOL;  // Clamp confirmation sensors
    xLocators:      ARRAY[1..3]  OF BOOL;  // Part locator sensors
    iCheck:         INT;
    tClampTimeout:  TON;
END_VAR

// CLAMP ALL:
// IF bClampAll THEN
//     FOR iCheck := 1 TO 12 DO
//         xClamps[iCheck] := TRUE;  // Energize all clamp solenoids
//     END_FOR;
//     
//     // Wait 2 seconds for clamps to close
//     tClampTimeout(IN := TRUE, PT := T#2s);
//     IF tClampTimeout.Q THEN
//         // Verify all confirmation sensors
//         bAllClamped := TRUE;
//         iMissingClamp := 0;
//         FOR iCheck := 1 TO 12 DO
//             IF NOT xSensors[iCheck] THEN
//                 bAllClamped := FALSE;
//                 iMissingClamp := iCheck;  // Report first unclamped
//             END_IF;
//         END_FOR;
//         
//         // Check part seated (all 3 locators must be active)
//         bPartSeated := xLocators[1] AND xLocators[2] AND xLocators[3];
//         
//         bReadyToWeld := bAllClamped AND bPartSeated;
//     END_IF;
// END_IF;
```

---

## Step 1252: Weld Sequence Programming

### เขียน Sequence Program ครบ

```
COMPLETE WELD SEQUENCE (State Machine):
──────────────────────────────────────────────────────────────

STATE MACHINE สำหรับ Robotic Welding Cell:

// States
CONST
    STATE_IDLE          := 0;
    STATE_LOAD_WAIT     := 1;   // Wait for operator to load part
    STATE_FIXTURE        := 2;   // Close fixture clamps
    STATE_PREHEAT_CHECK  := 3;   // Verify preheat temperature
    STATE_ROBOT_WELD     := 4;   // Robot welding in progress
    STATE_INTERPASS_CHECK:= 5;   // Check interpass temperature
    STATE_COMPLETE       := 6;   // Weld done, open fixtures
    STATE_UNLOAD         := 7;   // Operator unloads
    STATE_FAULT          := 99;
END_CONST

FUNCTION_BLOCK FB_WeldCellMaster
VAR
    iState:         INT := STATE_IDLE;
    fbFixture:      FB_WeldFixture;
    fbPreheat:      FB_PreheatCheck;
    fbRobot:        FB_RobotControl;
    fbWelder:       FB_MIGWelder_FRONIUS;
    fbQuality:      FB_WeldQualityMonitor;
    iPassNumber:    INT := 1;           // Current weld pass
    iMaxPasses:     INT := 3;           // Total passes required
    tStateTimer:    TON;
END_VAR

CASE iState OF
    STATE_IDLE:
        IF bStartCmd AND bTurntable_At_WeldStation THEN
            iPassNumber := 1;
            iState := STATE_LOAD_WAIT;
        END_IF;
    
    STATE_LOAD_WAIT:
        // Operator loads part, presses ready
        IF bOperatorReady THEN
            iState := STATE_FIXTURE;
        END_IF;
    
    STATE_FIXTURE:
        fbFixture(bClampAll := TRUE);
        IF fbFixture.bReadyToWeld THEN
            iState := STATE_PREHEAT_CHECK;
        ELSIF fbFixture.bAllClamped AND NOT fbFixture.bPartSeated THEN
            // Part not seated correctly
            fbFixture(bUnclampAll := TRUE);
            iState := STATE_FAULT;
        END_IF;
    
    STATE_PREHEAT_CHECK:
        // Measure temperature
        rActualTemp := rPyrometer;
        IF rActualTemp >= DB_WeldPrograms.programs[iWeldProgram].rPreheat_Min THEN
            iState := STATE_ROBOT_WELD;
        ELSE
            // Preheat needed → output to heating blanket control
            xHeatBlanket := TRUE;
        END_IF;
    
    STATE_ROBOT_WELD:
        xHeatBlanket := FALSE;
        // Send start command to robot
        DB_RobotInterface.toPLC_ProgramNum := iWeldProgram;
        DB_RobotInterface.toPLC_Command.0 := TRUE;  // Start
        
        IF DB_RobotInterface.fromRobot_Status.2 THEN  // Complete
            DB_RobotInterface.toPLC_Command.0 := FALSE;
            
            IF iPassNumber < iMaxPasses THEN
                iPassNumber := iPassNumber + 1;
                iState := STATE_INTERPASS_CHECK;
            ELSE
                iState := STATE_COMPLETE;
            END_IF;
        END_IF;
        
        IF DB_RobotInterface.fromRobot_Status.3 THEN  // Robot fault
            iState := STATE_FAULT;
        END_IF;
    
    STATE_INTERPASS_CHECK:
        // Check interpass temperature between passes
        IF rPyrometer <= DB_WeldPrograms.programs[iWeldProgram].rInterpass_Max THEN
            iState := STATE_ROBOT_WELD;  // Continue with next pass
        END_IF;
    
    STATE_COMPLETE:
        // Open fixtures, rotate turntable
        fbFixture(bUnclampAll := TRUE);
        IF fbFixture.bAllUnclamped THEN
            xTurntable_Rotate := TRUE;  // Rotate to load station
            iState := STATE_UNLOAD;
        END_IF;
    
    STATE_FAULT:
        xHeatBlanket := FALSE;
        fbFixture(bUnclampAll := TRUE);
        // Wait for reset
        IF bResetFault THEN
            iState := STATE_IDLE;
        END_IF;

END_CASE;
```

---

## Step 1253: Arc Welding Safety

### ความปลอดภัย Welding

```
WELDING SAFETY SYSTEMS:
──────────────────────────────────────────────────────────────

HAZARDS:
├── Arc flash: UV + IR radiation → eye and skin damage
├── Fumes: toxic gases (Mn, Cr, Ni compounds)
├── Electrical: high voltage/current
├── Fire: spatter, hot workpieces
└── Crush: robot movement + clamping fixtures

SAFETY SYSTEMS ที่ PLC ควบคุม:
1. LIGHT CURTAINS / AREA SCANNERS:
   ปิด cell boundary → any intrusion → E-stop

2. SMOKE EXTRACTION:
   สูบควันออก ตลอดเวลา welding
   PLC: xWeldStart → xExhaustFan := TRUE;
   PLC: xExhaustFan_Feedback = TRUE ก่อน allow weld

3. WELDING SCREEN:
   Opaque screen เปิดเมื่อ robot เชื่อม
   ปิดเมื่อ operator load/unload

4. GAS DETECTION:
   CO2, CO, Ozone detectors → alarm + stop ถ้าสูง

5. COOLING WATER MONITORING:
   บาง welder ต้องการ water-cooled torch
   PLC ตรวจ: flow, temperature, pressure
   IF xCoolingFlow_Low → ห้ามเชื่อม!

// Weld Safety Interlock Program:
// bPermitToWeld :=
//     xFence_Closed       // Safety fence closed
//     AND xLightCurtain_OK // Light curtain not triggered
//     AND xExhaustFan_OK  // Exhaust fan running
//     AND xCoolingWater_OK // Cooling water OK (if applicable)
//     AND xGasDetect_OK   // No gas alarm
//     AND xFixture_Clamped; // Part properly clamped
```

---

## Step 1254: Laser Welding Automation

### Laser Welding ใน PLC

```
LASER WELDING AUTOMATION:
──────────────────────────────────────────────────────────────

Laser Welding Systems:
├── IPG Photonics fiber laser (most common)
├── Trumpf disk laser
└── CO2 laser (older)

PLC Interface to Laser Controller:
Digital:
├── PLC → Laser: Enable, Gate (allow emission)
├── Laser → PLC: Ready, Emission Active, Error

Analog:
├── PLC → Laser: Power setpoint (0-10V = 0-100% power)
└── Laser → PLC: Power feedback, Temperature

// FB_LaserWelder
FUNCTION_BLOCK FB_LaserWelder
VAR_INPUT
    bEnable:        BOOL;
    bGate:          BOOL;       // Allow laser emission
    rPower_Pct:     REAL;       // Power 0-100%
    rDuty_Pct:      REAL;       // Duty cycle 0-100% (pulsed laser)
    rFrequency_Hz:  REAL;       // Pulse frequency (Hz)
END_VAR

// SAFETY FOR LASER:
// Class 4 laser → extremely hazardous
// MUST have:
// ├── Key switch (only authorized persons)
// ├── Beam stop (mechanical shutter)
// ├── Interlock on all access points
// └── Warning lights outside cell

// bLaserEmissionPermit :=
//     bKeySwitch_On
//     AND bShutter_Closed_Feedback   // Beam path enclosed
//     AND bInterlocks_All_OK
//     AND bPilotLight_On;            // Warning lights on

// ONLY then: bGate := TRUE (allow emission)
```

---

## Step 1255: TIG Welding for Pharmaceutical

### TIG Welding สำหรับ Pharmaceutical Piping

```
ORBITAL TIG WELDING (Pharmaceutical Piping):
──────────────────────────────────────────────────────────────

Used for: Sanitary piping (304L, 316L stainless)
Standard: ASME BPE (Bio-Processing Equipment)
ทำไม: Hygienic, no crevices, no contamination

Orbital Weld Head:
├── Rotates 360° around pipe
├── Fixed on pipe, torch rotates
└── No filler metal (autogenous TIG)

Key Parameters for Orbital Welding:
├── Travel speed: mm/sec (varies by position - uphill slower)
├── Current: Amperage (may vary 0°/90°/180°/270° position)
├── Rotation angle: Position of torch
└── Purge gas: Internal argon purge essential!

// PURGE GAS CONTROL (สำคัญมาก!)
// ก่อนเชื่อม: purge O2 ออกจากภายใน pipe
// Target: O2 < 20 ppm (ป้องกัน sugaring / oxidation ภายใน)

FUNCTION_BLOCK FB_PipelinePurge
VAR_INPUT
    bStartPurge:    BOOL;
    rFlowRate_lpm:  REAL := 10.0;  // 10 L/min argon
END_VAR
VAR_OUTPUT
    bPurgeComplete: BOOL;       // O2 level < 20ppm
    rO2_ppm:        REAL;       // O2 analyzer reading
END_VAR
VAR
    tPurgeTimer:    TON;
    rRequiredTime:  REAL;       // Calculated purge time
    fbO2Analyzer:   FB_ModbusDevice;  // O2 analyzer via Modbus
END_VAR

// Purge time = (Pipe volume × 5 volumes) / Flow rate
// rRequiredTime := (rPipeVolume_L × 5.0) / rFlowRate_lpm;

// Purge:
// xPurgeValve := TRUE;
// Read O2 from analyzer every 10 seconds
// bPurgeComplete := (rO2_ppm < 20.0);  // < 20ppm O2 = ready
```

---

## Step 1256: Weld Data Recording

### บันทึก Weld Data สำหรับ Traceability

```
WELD TRACEABILITY DATA:
──────────────────────────────────────────────────────────────

สำหรับ regulated industries (nuclear, pressure vessel, pharma):
ต้องบันทึก ALL weld data และ traceability

Required Records:
├── Weld ID (unique)
├── WPS number (which procedure used)
├── Welder/Robot qualification
├── Date and time
├── Actual parameters (current, voltage, speed, heat input)
├── Preheat and interpass temperatures
├── Consumables (wire heat/lot number, gas)
├── NDE results (XT-ray, UT if required)
└── Inspector sign-off

// DB_WeldTraceability
DATA_BLOCK DB_WeldTraceability
    records: ARRAY[1..50000] OF STRUCT
        iWeldID:        DINT;
        sJointID:       STRING[20];     // Which joint (J-001)
        sWPS_Number:    STRING[15];     // WPS-001-2024
        iRobotProgram:  INT;            // Robot program number
        dtStart:        DATE_AND_TIME;
        dtEnd:          DATE_AND_TIME;
        rPreheat_C:     REAL;           // Measured preheat
        rInterpass_C:   REAL;           // Peak interpass
        rAvgCurrent_A:  REAL;
        rAvgVoltage_V:  REAL;
        rAvgSpeed_mmpm: REAL;
        rHeatInput:     REAL;           // kJ/mm
        sWireHeat:      STRING[15];     // Wire heat number
        sGasBatch:      STRING[15];     // Gas batch number
        bInTolerance:   BOOL;
        sInspector:     STRING[20];     // Inspector ID
        bNDE_Required:  BOOL;
        sNDE_Result:    STRING[10];     // PASS/FAIL/PENDING
    END_STRUCT;
    iRecordCount:   DINT;
END_DATA_BLOCK

// Export to CSV/XML สำหรับ QA department:
// (ทำบน SCADA หรือ Node-RED)
// → ส่งไป Quality Management System (QMS)
```

---

## Step 1257: Tip Dressing and Gun Maintenance

### ดูแลรักษา Welding Equipment

```
WELDING GUN MAINTENANCE AUTOMATION:
──────────────────────────────────────────────────────────────

Contact Tip Life (MIG):
├── Standard tip: 5,000-10,000 welds
├── Heavy duty: 15,000-20,000 welds
└── Replace before: reduced arc stability, quality issues

// TIP DRESS CYCLE (Automatic Tip Dresser):
DATA_BLOCK DB_WeldMaintenance
    iWeldCount:             DINT := 0;      // Welds since last dress
    iTipDress_Interval:     INT := 200;     // Dress every 200 welds
    iTipReplace_Interval:   INT := 5000;    // Replace every 5000 welds
    bTipDressRequired:      BOOL;
    bTipReplaceRequired:    BOOL;
    iDressCount:            INT := 0;       // Number of dresses on current tip
    rNozzleInspect_Interval:INT := 100;     // Check nozzle every 100 welds
END_DATA_BLOCK

// After each weld:
// DB_WeldMaintenance.iWeldCount := DB_WeldMaintenance.iWeldCount + 1;
// 
// Tip dress check:
// IF DB_WeldMaintenance.iWeldCount MOD DB_WeldMaintenance.iTipDress_Interval = 0 THEN
//     bTipDressRequired := TRUE;
// END_IF;
// 
// Tip replace check:
// IF DB_WeldMaintenance.iWeldCount >= DB_WeldMaintenance.iTipReplace_Interval THEN
//     bTipReplaceRequired := TRUE;
//     // Stop production, alert maintenance
// END_IF;

// TIP DRESSER SEQUENCE (PLC controls automatic dresser):
// 1. Robot moves to tip dresser position
// 2. Tip dresser motor on for 2 seconds
// 3. Robot moves away
// 4. Resume production
```

---

## Step 1258: Welding Cell HMI Design

### ออกแบบ HMI

```
WELDING CELL HMI DESIGN:
──────────────────────────────────────────────────────────────

Main Screen Elements:
┌───────────────────────────────────────────────────────────┐
│  WELDING CELL Z3 - BODY SHOP STATION 12        [14:32:05] │
├───────────────────────────────────────────────────────────┤
│  STATE: WELDING                     PASS: 2/3             │
│                                                           │
│  [ROBOT VIEW]         [PARAMETERS]                        │
│  [3D model of cell]   Current: 182.3A (target: 180A ±5%) │
│                       Voltage: 24.1V  (target: 24V ±5%)  │
│                       Speed:   420mm/min                  │
│                       Heat In: 1.56 kJ/mm                 │
│                                                           │
│  [STATISTICS]          [ALARMS]                          │
│  Total: 1,247 welds    [No active alarms]                │
│  Today: 87 welds                                          │
│  Rejects: 2 (2.3%)                                        │
│  Next tip dress: 47 welds                                 │
│                                                           │
│  [PRODUCTION]    [MAINTENANCE]    [SETTINGS]              │
└───────────────────────────────────────────────────────────┘

Trend Screen:
├── Real-time current/voltage trend (1 minute rolling)
├── Weld quality trend (last 100 welds: pass/fail)
└── Tip dress history

Maintenance Screen:
├── Tip dress countdown
├── Gas cylinder level
├── Wire spool remaining
└── Equipment hours
```

---

## Step 1259: Welding Automation for Different Industries

### Welding ใน Industry ต่างๆ

```
INDUSTRY-SPECIFIC WELDING:
──────────────────────────────────────────────────────────────

AUTOMOTIVE (Body Shop):
├── Process: Resistance spot welding (primary), MIG (secondary)
├── Volume: 3,000-5,000 spots per vehicle, 1,000 cars/day
├── Requirement: Speed + consistency
├── PLC: Fast response, Turntable coordination
└── Quality: 100% monitoring, zero defect (safety critical parts)

SHIPBUILDING:
├── Process: MIG/MAG, SAW (Submerged Arc Welding)
├── Volume: Long seams (100s of meters)
├── Requirement: Large gantry robots, high deposition rate
├── PLC: Seam tracking with vision/arc sensor
└── Quality: UT inspection required for class welds

PRESSURE VESSELS (ASME):
├── Process: GMAW, GTAW, SAW
├── Standard: ASME Section VIII + Section IX
├── Requirement: WPS qualified, weld traceability
├── PLC: Parameter logging mandatory, audit trail
└── Quality: Radiography required for code welds

FOOD/PHARMA (Hygienic):
├── Process: Orbital TIG (no filler), plasma
├── Standard: ASME BPE, EHEDG
├── Requirement: Internal surface finish Ra < 0.8μm
├── PLC: O2 purge monitoring, parameter traceability
└── Quality: 100% internal visual + borescope inspection

PIPELINE (Oil & Gas):
├── Process: SMAW root, then GMAW fill
├── Standard: API 1104, DNV-OS-F101
├── Requirement: Extreme reliability (no repair in field)
├── PLC: Automatic welding machine (AUTOpipe etc.)
└── Quality: 100% AUT (Automated Ultrasonic Testing)
```

---

## Step 1260: Lab Project - Robotic Welding Cell Control

### Lab: Programming Complete Welding Cell

```
LAB: ROBOTIC WELDING CELL CONTROL PROGRAM
══════════════════════════════════════════════════════════════

Task: เขียน PLC program สำหรับ MIG welding cell ที่มี:
- 1 Robot (6-axis, Fanuc/KUKA/ABB simulation)
- 1 MIG Welder (Fronius interface)
- 1 Turntable (2-station)
- 12 Fixture clamps per station
- Preheat monitor (IR pyrometer)
- Safety fence with light curtain

TASK 1: Data Blocks Design (15%)
─────────────────────────────────────────────────────────────
สร้าง:
1. DB_WeldPrograms: 10 programs (vary material/thickness)
2. DB_WeldQuality: ที่เก็บ last 500 weld records
3. DB_CellStatus: real-time cell status
4. DB_Maintenance: tip/consumable tracking

TASK 2: Fixture Control (20%)
─────────────────────────────────────────────────────────────
FB_WeldFixture:
1. Clamp all 12 clamps (with timeout)
2. Verify all clamp sensors
3. Verify part seated on locators
4. Unclamp sequence
5. Fault if any clamp fails to confirm

TASK 3: Weld Cell Master (40%)
─────────────────────────────────────────────────────────────
FB_WeldCellMaster state machine:
1. States: IDLE → LOAD → CLAMP → PREHEAT → WELD → INTERPASS → COMPLETE → FAULT
2. Robot interface (send program number, receive status)
3. Welder interface (enable, monitor actual parameters)
4. Quality check per weld (tolerance check)
5. Auto tip dress trigger
6. Production statistics update

TASK 4: Safety System (15%)
─────────────────────────────────────────────────────────────
1. Light curtain → E-stop (SS1 ถ้ามี robot motion)
2. Gas low alarm → pause welding, alert
3. Exhaust fan fault → stop welding
4. Fire detection → emergency stop + alarm

TASK 5: HMI Design (10%)
─────────────────────────────────────────────────────────────
1. Main status screen
2. Weld parameter display (actual vs target)
3. Quality trend (last 50 welds)
4. Maintenance countdown screen
5. Alarm page

DELIVERABLES:
├── TIA Portal project (FB_WeldFixture, FB_WeldCellMaster, etc.)
├── DB structure documentation
├── State machine diagram
├── Test plan + results
└── Weld quality report (simulated 100 welds)

EVALUATION:
├── State machine correctness: 30%
├── Safety implementation: 25%
├── Quality monitoring: 20%
├── HMI design: 15%
└── Documentation: 10%
```

---

## สรุป Part 63

| กระบวนการ | ข้อดี | การ Interface |
|-----------|------|---------------|
| MIG/MAG | ง่าย, เร็ว, สำหรับ carbon steel | PROFINET / EtherNet/IP |
| TIG Orbital | สะอาด, pharma grade | Digital I/O |
| Spot welding | Mass production | เวลา/กระแส/แรง |
| Laser | แม่นยำ, ไม่มี filler | Gate signal + power setpoint |

---

## Navigation

- [← Part 62: Advanced Motion - Gantry and Delta Robots](./part62_advanced_motion_gantry.md)
- [→ Part 64: Food and Pharmaceutical Automation](./part64_food_pharma.md)
- [↑ สารบัญหลัก](../README.md)
