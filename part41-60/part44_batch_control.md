# Part 44: Batch Control ISA-88 (Steps 861-880)

## ภาพรวม
ISA-88 (ANSI/ISA-88) เป็นมาตรฐานสำหรับการควบคุมกระบวนการแบบ Batch ซึ่งใช้กันอย่างแพร่หลายในอุตสาหกรรมเคมี ยา อาหาร และเครื่องสำอาง

---

## Step 861: ISA-88 Model Overview

### โครงสร้าง ISA-88

```
ISA-88 Hierarchy:
┌─────────────────────────────────────────────────────────────┐
│  Enterprise Level                                           │
│  └── ERP System (SAP, Oracle)                              │
│       └── Batch Orders, Production Schedule                │
│                                                             │
│  Site Level                                                │
│  └── MES / Batch Server                                    │
│       └── Site Recipe Management                           │
│                                                             │
│  Area Level                                                │
│  └── Area Control (SCADA/DCS)                              │
│       └── Production Area (e.g., Mixing Area)             │
│                                                             │
│  Process Cell Level                                        │
│  └── Process Cell Controller (PLC/DCS)                    │
│       └── Reactor Cell, Mixing Cell                       │
│                                                             │
│  Unit Level ◄── หัวใจของ ISA-88                           │
│  └── Unit (Reactor R101, Mixer M201)                      │
│       └── Contains Equipment Modules                      │
│                                                             │
│  Equipment Module Level                                    │
│  └── Functional group (Agitator, Jacket, Valve group)     │
│                                                             │
│  Control Module Level                                      │
│  └── Individual device (Valve XV-101, Pump P-101)         │
└─────────────────────────────────────────────────────────────┘
```

### Physical Model vs. Procedural Model

```
Physical Model (Equipment):           Procedural Model (Recipe):
────────────────────────────          ───────────────────────────
Process Cell                          Procedure
  Unit                                  Unit Procedure
    Equipment Module                      Operation
      Control Module                        Phase
```

---

## Step 862: Recipe Structure

### ลำดับชั้นของ Recipe

```
Recipe Hierarchy:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  GENERAL RECIPE (สูตรทั่วไป)                               │
│  └── ไม่เฉพาะเจาะจงกับ equipment ใด                       │
│       สร้างโดย R&D/Process Engineer                        │
│       เช่น: "วิธีทำ Product X"                            │
│                                                             │
│  SITE RECIPE (สูตรระดับ Site)                              │
│  └── Adapted สำหรับ site นี้ (วัตถุดิบ, equipment)        │
│       เช่น: "ทำ Product X ที่ Plant A"                    │
│                                                             │
│  MASTER RECIPE (สูตรหลัก)                                  │
│  └── เฉพาะสำหรับ Process Cell นี้                         │
│       พร้อม equipment requirements                         │
│       เช่น: "ทำ Product X ใน Reactor R101"               │
│                                                             │
│  CONTROL RECIPE (สูตรควบคุม)                               │
│  └── Instance ที่ถูกสร้างสำหรับ batch นี้                 │
│       มี Batch ID, lot number, parameters                 │
│       Executed โดย Batch Executive                         │
└─────────────────────────────────────────────────────────────┘
```

### Master Recipe Structure

```xml
<!-- Master Recipe Example (XML Format) -->
<MasterRecipe id="RCP-001" version="2.1" product="Product_X">
  <Header>
    <Name>Polymer Synthesis Recipe</Name>
    <ProcessCell>PC_Reactor_Area</ProcessCell>
    <BatchSize unit="kg">500</BatchSize>
  </Header>
  
  <FormulaProcedure>
    <Procedure>
      
      <UnitProcedure id="UP1" unit="Reactor_R101">
        
        <Operation id="OP1" name="Charge">
          <Phase id="PH1" name="Charge_Monomer">
            <ProcessParameter name="Charge_Volume" value="250" unit="L"/>
            <ProcessParameter name="Charge_Rate" value="50" unit="L/min"/>
          </Phase>
        </Operation>
        
        <Operation id="OP2" name="React">
          <Phase id="PH2" name="Heat_To_SP">
            <ProcessParameter name="Target_Temp" value="85" unit="degC"/>
            <ProcessParameter name="Heat_Rate" value="2" unit="degC/min"/>
          </Phase>
          <Phase id="PH3" name="React_Hold">
            <ProcessParameter name="React_Time" value="120" unit="min"/>
            <ProcessParameter name="Agit_Speed" value="200" unit="rpm"/>
          </Phase>
        </Operation>
        
        <Operation id="OP3" name="Transfer">
          <Phase id="PH4" name="Transfer_Out">
            <ProcessParameter name="Transfer_Vol" value="490" unit="L"/>
          </Phase>
        </Operation>
        
      </UnitProcedure>
      
    </Procedure>
  </FormulaProcedure>
</MasterRecipe>
```

---

## Step 863: Phase Logic

### State Machine ของ Phase

```
ISA-88 Phase State Machine:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    IDLE                                     │
│                     │                                       │
│               [RUNNING command]                             │
│                     │                                       │
│                  RUNNING ──[PAUSE]──► PAUSING              │
│                     │                    │                  │
│               [COMPLETE]           [PAUSED]                │
│                     │                    │                  │
│                COMPLETING          [RESTART]               │
│                     │                    │                  │
│                 COMPLETE ◄─────────────                    │
│                     │                                       │
│               [RESET command]                              │
│                     │                                       │
│                    IDLE                                     │
│                                                             │
│  Fault States:                                             │
│  RUNNING ──[FAULT]──► ABORTING ──► ABORTED                 │
│  RUNNING ──[STOP]───► STOPPING ──► STOPPED                 │
└─────────────────────────────────────────────────────────────┘
```

### Phase Logic Implementation ใน PLC

```pascal
// Phase State Machine Implementation
// FC: Phase_Control
// ใช้ State variable เพื่อติดตาม phase state

FUNCTION_BLOCK Phase_Control
VAR_INPUT
    Cmd_Start    : Bool;   // คำสั่ง Start จาก Batch Executive
    Cmd_Pause    : Bool;   // คำสั่ง Pause
    Cmd_Restart  : Bool;   // คำสั่ง Restart
    Cmd_Stop     : Bool;   // คำสั่ง Stop
    Cmd_Abort    : Bool;   // คำสั่ง Abort
    Cmd_Reset    : Bool;   // คำสั่ง Reset
    Phase_Logic_Complete : Bool;  // โปรแกรมแจ้งว่า phase เสร็จสิ้น
END_VAR

VAR_OUTPUT
    State       : Int;    // Current state
    Phase_Active: Bool;   // Phase กำลังทำงาน
    Phase_Done  : Bool;   // Phase เสร็จสิ้นแล้ว
    Phase_Fault : Bool;   // Phase มีความผิดปกติ
END_VAR

VAR
    Prev_State  : Int;    // Previous state สำหรับ edge detection
END_VAR

CONST
    IDLE        := 0;
    RUNNING     := 1;
    PAUSING     := 2;
    PAUSED      := 3;
    COMPLETING  := 4;
    COMPLETE    := 5;
    STOPPING    := 6;
    STOPPED     := 7;
    ABORTING    := 8;
    ABORTED     := 9;
END_CONST

BEGIN
    CASE State OF
    
        IDLE:
            Phase_Active := FALSE;
            Phase_Done := FALSE;
            Phase_Fault := FALSE;
            IF Cmd_Start THEN
                State := RUNNING;
            END_IF;
        
        RUNNING:
            Phase_Active := TRUE;
            IF Cmd_Abort THEN
                State := ABORTING;
            ELSIF Cmd_Stop THEN
                State := STOPPING;
            ELSIF Cmd_Pause THEN
                State := PAUSING;
            ELSIF Phase_Logic_Complete THEN
                State := COMPLETING;
            END_IF;
        
        PAUSING:
            // บันทึก current values ก่อน pause
            State := PAUSED;
        
        PAUSED:
            Phase_Active := FALSE;
            IF Cmd_Abort THEN State := ABORTING;
            ELSIF Cmd_Restart THEN State := RUNNING;
            END_IF;
        
        COMPLETING:
            // Wrap-up actions (close valves, etc.)
            State := COMPLETE;
        
        COMPLETE:
            Phase_Active := FALSE;
            Phase_Done := TRUE;
            IF Cmd_Reset THEN State := IDLE; END_IF;
        
        STOPPING:
            // Controlled stop
            State := STOPPED;
        
        STOPPED:
            Phase_Active := FALSE;
            IF Cmd_Reset THEN State := IDLE; END_IF;
        
        ABORTING:
            // Emergency stop, abort
            Phase_Fault := TRUE;
            State := ABORTED;
        
        ABORTED:
            Phase_Active := FALSE;
            Phase_Fault := TRUE;
            IF Cmd_Reset THEN
                Phase_Fault := FALSE;
                State := IDLE;
            END_IF;
    
    END_CASE;
END_FUNCTION_BLOCK
```

---

## Step 864: Unit Procedure

### การเขียน Unit Procedure

```
Unit Procedure Execution Flow:

Reactor R101 Unit Procedure:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  UP1: Reactor_Run (Unit Procedure)                         │
│  │                                                          │
│  ├─ OP1: CHARGE                                            │
│  │   ├─ PH1: Charge_Solvent → Fill tank with solvent      │
│  │   └─ PH2: Charge_Monomer → Add monomer                 │
│  │                                                          │
│  ├─ OP2: REACT                                             │
│  │   ├─ PH3: Heat_To_SP    → Ramp temperature to setpoint │
│  │   ├─ PH4: Hold_React    → Hold at temperature + time   │
│  │   └─ PH5: Sample_Check  → Request sample               │
│  │                                                          │
│  ├─ OP3: QUENCH                                            │
│  │   ├─ PH6: Add_Quench    → Add quench agent             │
│  │   └─ PH7: Cool_Down     → Cool to safe temperature     │
│  │                                                          │
│  └─ OP4: TRANSFER                                          │
│      ├─ PH8: Transfer_Out  → Transfer product             │
│      └─ PH9: Rinse_Reactor → Rinse reactor                │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 865: Procedural Control

### การควบคุมแบบ Procedural

```pascal
// Unit Procedure Controller
// FB: UnitProc_R101

FUNCTION_BLOCK UnitProc_R101
VAR_INPUT
    Start_UP    : Bool;   // เริ่ม Unit Procedure
    Stop_UP     : Bool;   // หยุด
    Abort_UP    : Bool;   // Abort
    Recipe_SP   : Struct_Recipe;  // Recipe parameters
END_VAR

VAR_OUTPUT
    UP_State    : Int;    // 0=Idle, 1=Running, ...
    Op_Active   : Int;    // Current operation number
    Ph_Active   : Int;    // Current phase number
    UP_Complete : Bool;   // Unit Procedure เสร็จสิ้น
END_VAR

VAR
    // Phase instances
    PH_Charge_Solvent  : Phase_Charge_Solvent;
    PH_Charge_Monomer  : Phase_Charge_Monomer;
    PH_Heat_To_SP      : Phase_HeatToSP;
    PH_Hold_React      : Phase_HoldReact;
    PH_Transfer_Out    : Phase_TransferOut;
    
    Current_Phase : Int := 0;
    Phase_Done    : Bool;
END_VAR

BEGIN
    // Sequence control
    CASE Current_Phase OF
    
        0: // Wait for start
            IF Start_UP THEN
                Current_Phase := 1;
                Op_Active := 1;
                Ph_Active := 1;
            END_IF;
        
        1: // Phase: Charge_Solvent
            PH_Charge_Solvent(
                Start         := TRUE,
                Charge_Vol    := Recipe_SP.Solvent_Vol,
                Charge_Rate   := Recipe_SP.Solvent_Rate
            );
            IF PH_Charge_Solvent.Done THEN
                Current_Phase := 2;
                Ph_Active := 2;
            END_IF;
        
        2: // Phase: Charge_Monomer
            PH_Charge_Monomer(
                Start         := TRUE,
                Charge_Vol    := Recipe_SP.Monomer_Vol,
                Charge_Rate   := Recipe_SP.Monomer_Rate
            );
            IF PH_Charge_Monomer.Done THEN
                Current_Phase := 3;
                Op_Active := 2;  // Next operation
                Ph_Active := 3;
            END_IF;
        
        3: // Phase: Heat_To_SP
            PH_Heat_To_SP(
                Start      := TRUE,
                Target_SP  := Recipe_SP.React_Temp,
                Heat_Rate  := Recipe_SP.Heat_Rate
            );
            IF PH_Heat_To_SP.Done THEN
                Current_Phase := 4;
                Ph_Active := 4;
            END_IF;
        
        4: // Phase: Hold_React
            PH_Hold_React(
                Start       := TRUE,
                Hold_Time   := Recipe_SP.React_Time,
                Agit_Speed  := Recipe_SP.Agit_Speed
            );
            IF PH_Hold_React.Done THEN
                Current_Phase := 5;
                Op_Active := 3;
                Ph_Active := 5;
            END_IF;
        
        5: // Phase: Transfer_Out
            PH_Transfer_Out(
                Start        := TRUE,
                Transfer_Vol := Recipe_SP.Transfer_Vol
            );
            IF PH_Transfer_Out.Done THEN
                Current_Phase := 99;
            END_IF;
        
        99: // Unit Procedure Complete
            UP_Complete := TRUE;
            Op_Active := 0;
            Ph_Active := 0;
    
    END_CASE;
END_FUNCTION_BLOCK
```

---

## Step 866-867: S88 Phase Implementation in Ladder

### Phase: Heat_To_SetPoint ใน Ladder Diagram

```ladder
// ====================================================
// FB: Phase_HeatToSP
// วัตถุประสงค์: Ramp temperature ไปยัง setpoint
// ====================================================

// Network 1: Phase State Machine
// (เรียก Phase_Control FB เพื่อจัดการ state)
+--[Phase_Control_Instance]--+
|  Cmd_Start    := PH_Start  |
|  Cmd_Pause    := PH_Pause  |
|  Cmd_Abort    := PH_Abort  |
|  Phase_Logic_Complete := Temp_At_SP  |
|  State        => PH_State  |
|  Phase_Active => PH_Running|
+---------------------------+

// Network 2: Calculate Ramp Setpoint
// เมื่อ phase กำลังทำงาน: คำนวณ SP ที่ ramp ขึ้นไป
//    PH_Running                      Ramp_Calc_EN
//    ---[ ]------------------------------( )------

+--[ PH_Running ]--+--( Ramp_Enable )--+

// Network 3: Ramp Calculation (Structured Text inside LAD block)
+--[CALCULATE_RAMP]--+
|  Enable  := Ramp_Enable |
|  Current := TIC_101.PV  |  // อุณหภูมิปัจจุบัน
|  Target  := Target_SP   |  // SP จาก recipe
|  Rate    := Heat_Rate   |  // degC/min จาก recipe
|  Ramp_SP => TIC_101.SP  |  // ส่งไปยัง PID setpoint
+--------------------+

// Network 4: Check if temperature reached setpoint
// (PV >= SP - 0.5 degC)
//    TIC101_PV >= (Target_SP - 0.5)    Temp_At_SP
//    ─────────────────────────────────────( )──────

+--[ GE TIC_101.PV (Target_SP - 0.5) ]--+--( Temp_At_SP )--+

// Network 5: Safety limit - maximum temperature
//    TIC101_PV > Max_Safe_Temp    Phase_Abort
//    ─────────────────────────────────( )──────

+--[ GT TIC_101.PV Max_Safe_Temp ]--+--( Phase_Fault_Abort )--+

// Network 6: Heater Control (from PID output)
//    PH_Running    TIC_HeatCV > 0      Heater_On
//    ---[ ]----------[ ]------------------( )------

+--[ PH_Running ]--+--[ GT TIC_101.CV 0.0 ]--+--( Heater_Enable )--+
```

---

## Step 868: Equipment Arbitration

### การจัดการ Equipment Arbitration

```
Equipment Arbitration (การจองใช้ equipment):
┌─────────────────────────────────────────────────────────────┐
│  Problem: หลาย Unit Procedure ต้องการใช้ equipment เดียวกัน │
│  Solution: Equipment Arbitration / Token passing           │
│                                                             │
│  ตัวอย่าง: CIP Tank ใช้ได้ครั้งละ 1 Unit เท่านั้น         │
│                                                             │
│  Unit R101 ──► Request CIP_Tank ──► Wait if busy          │
│  Unit R102 ──► Request CIP_Tank ──► ได้ทันที (ถ้าว่าง)    │
│                                                             │
│  Arbitration Logic:                                        │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  CIP_Tank_Status: 0=Free, 1=Busy (R101), 2=Busy(R102)│ │
│  │                                                       │ │
│  │  R101 Request:                                        │ │
│  │  IF CIP_Tank_Status = 0 THEN                          │ │
│  │      CIP_Tank_Status := 1  // Allocate to R101       │ │
│  │      R101_CIP_Granted := TRUE                         │ │
│  │  ELSE                                                 │ │
│  │      R101_CIP_Wait := TRUE  // Wait in queue         │ │
│  │  END_IF                                               │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 869: Material Tracking

### การติดตาม Material ใน Batch System

```
Material Tracking in Batch:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Input Materials:           Output Material:               │
│  ┌──────────────────┐      ┌────────────────┐              │
│  │ Monomer A: 250L  │      │ Product X      │              │
│  │ Catalyst B: 5L   │──►   │ Lot: 2024-001  │             │
│  │ Solvent C: 200L  │      │ 490L           │              │
│  └──────────────────┘      │ Grade: A       │              │
│                            └────────────────┘              │
│                                                             │
│  Genealogy Record:                                         │
│  Batch 2024-001 ← RM: Mon-A Lot 123, Cat-B Lot 456        │
│                  ← RM: Solv-C Lot 789                     │
│                                                             │
│  Electronic Batch Record (EBR):                            │
│  ├── Phase start/end timestamps                           │
│  ├── Recipe parameters vs. Actual values                  │
│  ├── Operator comments/deviations                         │
│  ├── Alarm history during batch                           │
│  └── Electronic signatures (21 CFR Part 11)              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 870-880: Complete Reactor Batch Example

### โครงการ: Polymer Reactor Batch System

#### 1. System Overview

```
Reactor R101 System:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│        ┌──── Monomer Feed ────────────────┐                 │
│        │     (FIC-101, FCV-101)           │                 │
│        │                                 ▼                 │
│        │   Nitrogen ────────────────► [REACTOR R101]        │
│        │   (XV-102)                   │         │          │
│        │                              │  TIC-101 │          │
│        │                              │  PIC-102 │          │
│        │                              │  LT-103  │          │
│        │   Steam ──────► Jacket ──────►          │          │
│        │   (TCV-104)    Coils    ◄─── CW         │          │
│        │                                         │          │
│        └──── Product Transfer ◄────────────── ──┘          │
│              (FIC-105, FCV-105)                            │
└─────────────────────────────────────────────────────────────┘

Instruments:
- FIC-101: Monomer flow controller (Coriolis meter + control valve)
- TIC-101: Reactor temperature controller (RTD + heating/cooling)
- PIC-102: Reactor pressure controller (pressure transmitter + relief)
- LT-103: Reactor level (guided wave radar)
- FIC-105: Product transfer (Coriolis + pump)
```

#### 2. Control Modules

```pascal
// Control Module: Valve XV-101 (Monomer Feed Valve)
FUNCTION_BLOCK CM_Valve
VAR_INPUT
    Auto_Open  : Bool;   // Open command from upper level
    Auto_Close : Bool;   // Close command
    Manual_Open: Bool;   // Manual override
    Manual_Mode: Bool;   // Switch to manual
END_VAR
VAR_OUTPUT
    Valve_Open : Bool;   // Actual open command to field
    Status     : Int;    // 0=Closed, 1=Open, 2=Fault
    Is_Open    : Bool;   // Position feedback (limit switch)
    Is_Closed  : Bool;   // Position feedback (limit switch)
END_VAR
VAR
    FB_Open    : Bool;   // Open feedback from field
    FB_Closed  : Bool;   // Closed feedback from field
    Open_Timer : TON;    // Travel time monitoring
    Fault      : Bool;
END_VAR

BEGIN
    // Determine output
    IF Manual_Mode THEN
        Valve_Open := Manual_Open;
    ELSE
        IF Auto_Open THEN Valve_Open := TRUE; END_IF;
        IF Auto_Close THEN Valve_Open := FALSE; END_IF;
    END_IF;
    
    // Monitor travel time (fault if stuck)
    Open_Timer(IN := Valve_Open XOR FB_Open,
               PT := T#10S);
    Fault := Open_Timer.Q;
    
    // Update status
    IF Fault THEN
        Status := 2;
    ELSIF FB_Open THEN
        Status := 1;
    ELSE
        Status := 0;
    END_IF;
END_FUNCTION_BLOCK
```

#### 3. Equipment Module: Agitator

```pascal
// Equipment Module: Agitator EM-AGIT-101
FUNCTION_BLOCK EM_Agitator
VAR_INPUT
    Start      : Bool;   // Start command
    Stop       : Bool;   // Stop command
    Speed_SP   : Real;   // Speed setpoint (RPM)
END_VAR
VAR_OUTPUT
    Running    : Bool;   // Agitator running
    Speed_PV   : Real;   // Actual speed
    Fault      : Bool;   // Fault status
    EM_State   : Int;    // 0=Stopped, 1=Starting, 2=Running, 3=Fault
END_VAR

BEGIN
    CASE EM_State OF
        0: // Stopped
            Running := FALSE;
            IF Start THEN EM_State := 1; END_IF;
        
        1: // Starting
            // Enable drive, set speed SP
            Drive_Enable := TRUE;
            Drive_SP := Speed_SP;
            // Wait for speed to reach SP ± 5%
            IF ABS(Speed_PV - Speed_SP) < (Speed_SP * 0.05) THEN
                EM_State := 2;
            END_IF;
        
        2: // Running
            Running := TRUE;
            IF Stop THEN
                Drive_Enable := FALSE;
                EM_State := 0;
            END_IF;
            IF Drive_Fault THEN EM_State := 3; END_IF;
        
        3: // Fault
            Running := FALSE;
            Fault := TRUE;
            Drive_Enable := FALSE;
    END_CASE;
END_FUNCTION_BLOCK
```

#### 4. Complete Ladder Diagram สำหรับ Phase: HoldReact

```ladder
// =====================================================
// FB: Phase_HoldReact
// วัตถุประสงค์: Hold reaction ที่ temperature เป็นเวลาที่กำหนด
// Recipe Parameters: Hold_Temp, Hold_Time, Agit_Speed
// =====================================================

// Network 1: Phase State Management
+--[CALL Phase_Control]--+
|  Cmd_Start    := PH_Start      |
|  Cmd_Pause    := PH_Pause      |
|  Cmd_Abort    := PH_Abort      |
|  Phase_Logic_Complete := Hold_Complete |
|  State        => PH_State      |
|  Phase_Active => PH_Running    |
+--------------------------------+

// Network 2: Temperature Control (เมื่อ phase กำลังทำงาน)
//  PH_Running                TIC101_SP_Write
//  ---[ ]----+--[MOVE Recipe_Hold_Temp]--( )--

+--[ PH_Running ]--+
                   |--[MOVE]--+
                   |  IN := Recipe_Hold_Temp |
                   |  OUT => TIC_101.SP      |
                   +----------+

// Network 3: Agitator Speed Control
+--[ PH_Running ]--+
                   |--[MOVE]--+
                   |  IN := Recipe_Agit_Speed |
                   |  OUT => Agitator_101.SP  |
                   +----------+

// Network 4: Start Agitator (ถ้ายังไม่ได้ start)
//  PH_Running   NOT Agit_Running   Agit_Start_Cmd
//  ---[ ]----------[/]------------------( )------

+--[ PH_Running ]--+--[ NOT Agitator_101.Running ]--+--( Start_Agitator )--+

// Network 5: Hold Timer (เริ่มนับเมื่ออุณหภูมิถึง SP)
//    PH_Running   Temp_At_SP    Hold_Timer
//    ---[ ]----------[ ]----------[TON]------

+--[ PH_Running ]--+--[ Temp_At_SP ]--+--[TON Hold_Timer PT=Recipe_Hold_Time]--+

// Network 6: Phase Complete (เมื่อ timer ครบ)
//    Hold_Timer.Q     Hold_Complete
//    ---[ ]---------------( )-------

+--[ Hold_Timer.Q ]--+--( Hold_Complete )--+

// Network 7: Temperature Check - Alarm หากเบี่ยงเบินมาก
//    |TIC101_PV - TIC101_SP| > 5.0    Temp_Deviation_Alarm
//    ─────────────────────────────────────( )──────────────

+--[ GT ABS(TIC_101.PV - TIC_101.SP) 5.0 ]--+--( Temp_Deviation_Alarm )--+

// Network 8: Phase Abort on Critical Alarm
//    PH_Running   PIC102_HIGH_HIGH    PH_Abort
//    ---[ ]──────────[ ]─────────────( )------

+--[ PH_Running ]--+--[ PIC_102.HH_Alarm ]--+--( PH_Abort )--+
|   (Phase active) |   (High-High Pressure)  |   (Abort phase)|
+------------------+------------------------+----------------+
```

#### 5. Batch Executive Program

```pascal
// FB: Batch_Executive_R101
// จัดการการทำงานของ Batch ทั้งหมด

FUNCTION_BLOCK Batch_Executive_R101
VAR_INPUT
    Batch_Start    : Bool;   // เริ่ม batch
    Batch_Stop     : Bool;   // หยุด batch
    Batch_Abort    : Bool;   // Abort
    Recipe         : Struct_Recipe;  // Recipe parameters
END_VAR

VAR_OUTPUT
    Batch_State    : Int;
    Current_Op     : Int;    // Operation ที่กำลังทำ
    Current_Phase  : Int;    // Phase ที่กำลังทำ
    Batch_Complete : Bool;
    Batch_ID       : DInt;   // Batch ID
    EBR_Tag        : String; // Electronic Batch Record tag
END_VAR

VAR
    // Phase instances
    PH_Charge_Solvent : Phase_Charge;
    PH_Charge_Monomer : Phase_Charge;
    PH_Heat_To_SP     : Phase_HeatToSP;
    PH_Hold_React     : Phase_HoldReact;
    PH_Transfer_Out   : Phase_Transfer;
    
    Step : Int := 0;
    
    // Timing for EBR
    Phase_Start_Time  : DInt;
    Phase_End_Time    : DInt;
END_VAR

BEGIN
    CASE Step OF
    
        0: // Wait for batch start
            Batch_Complete := FALSE;
            IF Batch_Start THEN
                Batch_ID := Get_Next_Batch_ID();
                Step := 10;  // OP1: CHARGE
                Current_Op := 1;
            END_IF;
        
        10: // OP1, PH1: Charge Solvent
            Current_Phase := 1;
            PH_Charge_Solvent(
                Start      := TRUE,
                Vol_SP     := Recipe.Solvent_Vol,
                Rate_SP    := Recipe.Solvent_Rate,
                Valve      := XV_Solvent,
                Flow_PV    := FIC_Solvent.PV
            );
            IF PH_Charge_Solvent.Done THEN
                Log_Phase_Complete(Batch_ID, 1, 1);
                Step := 20;
            END_IF;
        
        20: // OP1, PH2: Charge Monomer
            Current_Phase := 2;
            PH_Charge_Monomer(
                Start      := TRUE,
                Vol_SP     := Recipe.Monomer_Vol,
                Rate_SP    := Recipe.Monomer_Rate
            );
            IF PH_Charge_Monomer.Done THEN
                Log_Phase_Complete(Batch_ID, 1, 2);
                Step := 30;
                Current_Op := 2;
            END_IF;
        
        30: // OP2, PH3: Heat to Setpoint
            Current_Phase := 3;
            PH_Heat_To_SP(
                Start      := TRUE,
                Target_SP  := Recipe.React_Temp,
                Heat_Rate  := Recipe.Heat_Rate
            );
            IF PH_Heat_To_SP.Done THEN
                Log_Phase_Complete(Batch_ID, 2, 3);
                Step := 40;
            END_IF;
        
        40: // OP2, PH4: Hold Reaction
            Current_Phase := 4;
            PH_Hold_React(
                Start       := TRUE,
                Hold_Temp   := Recipe.React_Temp,
                Hold_Time   := Recipe.React_Time,
                Agit_Speed  := Recipe.Agit_Speed
            );
            IF PH_Hold_React.Done THEN
                Log_Phase_Complete(Batch_ID, 2, 4);
                Step := 50;
                Current_Op := 3;
            END_IF;
        
        50: // OP3, PH5: Transfer Out
            Current_Phase := 5;
            PH_Transfer_Out(
                Start      := TRUE,
                Vol_SP     := Recipe.Transfer_Vol
            );
            IF PH_Transfer_Out.Done THEN
                Log_Phase_Complete(Batch_ID, 3, 5);
                Batch_Complete := TRUE;
                Step := 99;
            END_IF;
        
        99: // Batch Complete
            Finalize_EBR(Batch_ID);
            Step := 0;
    
    END_CASE;
    
    // Global abort handling
    IF Batch_Abort AND Step > 0 AND Step < 99 THEN
        Log_Batch_Abort(Batch_ID);
        Step := 0;
    END_IF;
END_FUNCTION_BLOCK
```

---

## Lab Project 44: ISA-88 Batch System for Pharmaceutical Tablet Coating

### โจทย์
ออกแบบ ISA-88 Batch Control System สำหรับ Tablet Coating Process ในโรงงานยา (21 CFR Part 11 compliant)

### Process Overview
```
Coating Process:
1. Load tablets into coating pan
2. Preheat pan to 40°C
3. Spray coating solution (6 hours at 45°C)
   - Spray rate: 100ml/min
   - Pan speed: 10 RPM
   - Inlet air: 60°C, 1000 CFM
4. Dry (30 minutes at 50°C)
5. Cool down to 25°C
6. Discharge tablets
```

### งานที่ต้องทำ
1. สร้าง Recipe Structure (Procedure → Unit Procedure → Operation → Phase)
2. เขียน Phase logic สำหรับทุก phase
3. ออกแบบ Equipment Module สำหรับ: Spray Nozzle, Pan Motor, Heater, Air Handler
4. สร้าง Electronic Batch Record (EBR) structure
5. Implement equipment arbitration สำหรับ Clean-In-Place system
6. ทำ Batch Genealogy Tracking

---

## สรุป Part 44

| ระดับ ISA-88 | รายละเอียด | ตัวอย่าง |
|-------------|-----------|--------|
| Unit | Equipment unit | Reactor R101 |
| Equipment Module | Functional group | Agitator, Jacket |
| Control Module | Individual device | Valve XV-101 |
| Procedure | Full batch | Polymer Synthesis |
| Unit Procedure | Work in one unit | R101 Reaction |
| Operation | Major step | Charge, React, Transfer |
| Phase | Smallest controllable | Heat_To_SP |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 43: Redundancy Systems](part43_redundancy.md) |
| ► ถัดไป | [Part 45: Continuous Process Control](part45_continuous_control.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
