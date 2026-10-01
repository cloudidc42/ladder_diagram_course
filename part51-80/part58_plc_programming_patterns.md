# Part 58: PLC Programming Design Patterns (Steps 1141-1160)

## บทนำ: Design Patterns ใน PLC Programming

Design Patterns คือ แนวทางการออกแบบที่พิสูจน์แล้วว่าใช้งานได้ดี สำหรับปัญหาที่พบบ่อย เหมือนกับ Design Patterns ใน Software Engineering (GoF patterns) แต่ปรับให้เหมาะกับ PLC

ประโยชน์:
- **Reusability**: ใช้ซ้ำได้ในโปรเจกต์ต่าง ๆ
- **Maintainability**: ง่ายต่อการแก้ไข
- **Readability**: คนอื่นอ่านเข้าใจได้เร็ว
- **Testability**: test ได้ง่ายกว่า ad-hoc code

---

## Step 1141: State Machine Pattern

### State Machine - Pattern ที่สำคัญที่สุดใน PLC

```
STATE MACHINE PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Sequential operations (filling, mixing, cleaning)
├── Machine modes (idle, running, fault, maintenance)
├── Protocol handling
└── Process phases

ENUM สำหรับ State (กำหนด states ชัดเจน):
TYPE E_MachineState:
(
    STATE_IDLE      := 0,
    STATE_PRESTART  := 1,
    STATE_RUNNING   := 2,
    STATE_STOPPING  := 3,
    STATE_FAULT     := 4,
    STATE_MAINT     := 5
);
END_TYPE

State Transition Diagram:
         Start_Cmd
IDLE ──────────────→ PRESTART
  ↑                     │
  │       All_Ready     │
  │  ←────────────── RUNNING
  │                     │
  │     Stop_Cmd        │ Fault
  └──────────────── STOPPING → FAULT
                              │
                         Reset_Cmd
                              │
                        (→ IDLE)
```

### State Machine Implementation

```
// Filling Machine State Machine
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_FillingMachine
VAR_INPUT
    xStartCmd:      BOOL;       // Start command
    xStopCmd:       BOOL;       // Stop command
    xResetCmd:      BOOL;       // Reset from fault
    xEmergencyStop: BOOL;       // E-stop
    xBottlePresent: BOOL;       // Sensor: bottle present
    xFillComplete:  BOOL;       // Fill volume reached
    xCapApplied:    BOOL;       // Capper complete
    xFaultInput:    BOOL;       // Any hardware fault
END_VAR

VAR_OUTPUT
    eCurrentState:  INT;        // Current state (use E_MachineState)
    xConveyorRun:   BOOL;       // Conveyor motor
    xFillValve:     BOOL;       // Fill valve
    xCapperEnable:  BOOL;       // Capper enable
    xFaultLight:    BOOL;       // Fault indicator
    xReadyLight:    BOOL;       // Ready indicator
    sCycleState:    STRING;     // State name for HMI
END_VAR

VAR
    eState:         INT := 0;   // Current state
    ePrevState:     INT := -1;  // Previous state (for transition detect)
    tFillTimeout:   TON;        // Fill timeout
    tStateTimer:    TON;        // General state timer
    iCycleCount:    INT;        // Bottles produced
END_VAR

// ─── Network 1: E-Stop Override ──────────────────────────────
//  "xEmergencyStop"          All outputs off, goto FAULT
//  ──┤├────────────────────────── MOVE STATE_FAULT → eState
//  (E-Stop ทำได้จากทุก state)

// ─── Network 2: State Machine ────────────────────────────────
CASE eState OF
    
    0: // STATE_IDLE
        xConveyorRun := FALSE;
        xFillValve   := FALSE;
        xCapperEnable:= FALSE;
        xReadyLight  := TRUE;
        sCycleState  := 'IDLE';
        
        // Transition: Start Command
        IF xStartCmd AND NOT xEmergencyStop THEN
            eState := 1;  // → PRESTART
        END_IF;
    
    1: // STATE_PRESTART
        xConveyorRun := TRUE;   // Start conveyor
        xReadyLight  := FALSE;
        sCycleState  := 'PRESTART';
        
        // Check pre-start conditions
        tStateTimer(IN := TRUE, PT := T#5s);  // 5 sec warm-up
        IF tStateTimer.Q THEN
            tStateTimer(IN := FALSE);
            eState := 2;  // → RUNNING
        END_IF;
    
    2: // STATE_RUNNING - Wait for bottle, fill, cap
        xConveyorRun := TRUE;
        sCycleState  := 'RUNNING';
        
        // Sub-state for filling sequence handled by FB_FillSequence
        IF xBottlePresent THEN
            xFillValve := TRUE;
            tFillTimeout(IN := TRUE, PT := T#30s);
            
            IF xFillComplete THEN
                xFillValve := FALSE;
                xCapperEnable := TRUE;
            END_IF;
            
            IF xCapApplied THEN
                xCapperEnable := FALSE;
                iCycleCount := iCycleCount + 1;
            END_IF;
        END_IF;
        
        // Transitions
        IF xStopCmd THEN
            xFillValve := FALSE;
            eState := 3;  // → STOPPING
        ELSIF xFaultInput OR tFillTimeout.Q THEN
            xFillValve := FALSE;
            eState := 4;  // → FAULT
        END_IF;
    
    3: // STATE_STOPPING
        xConveyorRun := TRUE;   // Let current bottle finish
        sCycleState  := 'STOPPING';
        
        tStateTimer(IN := TRUE, PT := T#3s);
        IF tStateTimer.Q THEN
            tStateTimer(IN := FALSE);
            xConveyorRun := FALSE;
            eState := 0;  // → IDLE
        END_IF;
    
    4: // STATE_FAULT
        xConveyorRun := FALSE;
        xFillValve   := FALSE;
        xCapperEnable:= FALSE;
        xFaultLight  := TRUE;
        sCycleState  := 'FAULT';
        
        IF xResetCmd AND NOT xFaultInput THEN
            xFaultLight := FALSE;
            eState := 0;  // → IDLE
        END_IF;
    
END_CASE;

// ─── Network 3: State Change Detection ──────────────────────
// IF eState <> ePrevState THEN
//     // Log state change to historian
//     DB_StateHistory.transitions[DB_StateHistory.count].from := ePrevState;
//     DB_StateHistory.transitions[DB_StateHistory.count].to   := eState;
//     DB_StateHistory.transitions[DB_StateHistory.count].time := NOW();
//     DB_StateHistory.count := DB_StateHistory.count + 1;
//     ePrevState := eState;
// END_IF;
```

---

## Step 1142: Observer Pattern in PLC

### Observer Pattern สำหรับ Event Notification

```
OBSERVER PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── แจ้งเตือน modules หลายๆ ตัวเมื่อเกิด event
├── Alarm distribution (1 alarm → หลาย targets)
└── Log aggregation

Architecture:
Subject (เกิด event) → publishes to → Observer List
                                        ├── Observer 1 (HMI)
                                        ├── Observer 2 (Logger)
                                        ├── Observer 3 (SMS)
                                        └── Observer 4 (Email)
```

### Event Bus Pattern (Simplified Observer)

```
// Event Bus สำหรับ PLC (Simplified)
// ─────────────────────────────────────────────────────────────

// Event Types
CONST
    EVENT_NONE          := 0;
    EVENT_ALARM_HIGH    := 1;
    EVENT_ALARM_LOW     := 2;
    EVENT_ALARM_CLEAR   := 3;
    EVENT_MODE_CHANGE   := 4;
    EVENT_FAULT         := 5;
    EVENT_INFO          := 6;
END_CONST

// Event Structure
TYPE T_Event:
STRUCT
    iType:      INT;            // Event type
    iSource:    INT;            // Source module ID
    rValue:     REAL;           // Associated value
    dtTimestamp:DATE_AND_TIME;  // When occurred
    sDescription:STRING[80];   // Description
END_STRUCT;
END_TYPE

// Event Queue (Circular Buffer)
DATA_BLOCK DB_EventBus
    events:         ARRAY[0..99] OF T_Event;
    iWriteIndex:    INT := 0;
    iReadIndex:     INT := 0;
    iCount:         INT := 0;
END_DATA_BLOCK

// Publisher FB (any module can publish)
FUNCTION_BLOCK FB_EventPublisher
VAR_INPUT
    iEventType:     INT;
    iSourceID:      INT;
    rValue:         REAL;
    sDescription:   STRING[80];
END_VAR

// PUBLISH:
// IF bTrigger AND NOT bTrigger_Last THEN  // One-shot on rising edge
//     DB_EventBus.events[DB_EventBus.iWriteIndex].iType        := iEventType;
//     DB_EventBus.events[DB_EventBus.iWriteIndex].iSource      := iSourceID;
//     DB_EventBus.events[DB_EventBus.iWriteIndex].rValue       := rValue;
//     DB_EventBus.events[DB_EventBus.iWriteIndex].dtTimestamp  := NOW();
//     DB_EventBus.events[DB_EventBus.iWriteIndex].sDescription := sDescription;
//     DB_EventBus.iWriteIndex := (DB_EventBus.iWriteIndex + 1) MOD 100;
//     DB_EventBus.iCount := MIN(DB_EventBus.iCount + 1, 100);
// END_IF;
```

---

## Step 1143: Command Pattern

### Command Pattern สำหรับ Queued Operations

```
COMMAND PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Recipe execution (ทำงานตามลำดับ commands)
├── Robot motion commands queue
├── Valve sequence commands
└── undo/redo operations

Command Queue (Conveyor Control ตัวอย่าง):
```

```
// Command Queue สำหรับ Conveyor Station
// ─────────────────────────────────────────────────────────────

TYPE T_ConveyorCommand:
STRUCT
    iCommandType:   INT;    // CMD_MOVE=1, CMD_STOP=2, CMD_REJECT=3
    iTargetZone:    INT;    // Destination zone
    iBoxID:         INT;    // Box identifier
    bPriority:      BOOL;   // High priority command
END_STRUCT;
END_TYPE

DATA_BLOCK DB_CommandQueue
    commands:       ARRAY[0..19] OF T_ConveyorCommand;
    iHead:          INT := 0;   // Next to execute
    iTail:          INT := 0;   // Next empty slot
    iSize:          INT := 0;   // Number of commands waiting
END_DATA_BLOCK

FUNCTION_BLOCK FB_CommandExecutor
VAR_INPUT
    bExecuteNext:   BOOL;       // Trigger execute (usually: previous done)
END_VAR

VAR
    stCurrentCmd:   T_ConveyorCommand;
    fbConveyorCtrl: FB_ConveyorControl;
END_VAR

// Network 1: Execute next command from queue
// IF bExecuteNext AND DB_CommandQueue.iSize > 0 THEN
//     stCurrentCmd := DB_CommandQueue.commands[DB_CommandQueue.iHead];
//     DB_CommandQueue.iHead := (DB_CommandQueue.iHead + 1) MOD 20;
//     DB_CommandQueue.iSize := DB_CommandQueue.iSize - 1;
//     
//     // Execute based on command type
//     CASE stCurrentCmd.iCommandType OF
//         1: fbConveyorCtrl.MoveToZone(stCurrentCmd.iTargetZone);
//         2: fbConveyorCtrl.Stop();
//         3: fbConveyorCtrl.RejectBox(stCurrentCmd.iBoxID);
//     END_CASE;
// END_IF;
```

---

## Step 1144: Strategy Pattern

### Strategy Pattern สำหรับ Interchangeable Algorithms

```
STRATEGY PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Multiple control algorithms (P, PI, PID, fuzzy)
├── Different filling methods (volume, weight, time-based)
└── Multiple communication protocols

ตัวอย่าง: Filling Strategy
- Strategy 1: Volume-based fill (วัด flow meter)
- Strategy 2: Weight-based fill (วัด loadcell)
- Strategy 3: Time-based fill (timer only)
```

### Strategy Pattern Implementation

```
// Filling Strategy Pattern
// ─────────────────────────────────────────────────────────────

// Interface (Base FB)
FUNCTION_BLOCK FB_FillStrategy_Base
VAR_INPUT
    rTarget:        REAL;       // Target fill amount
    rActual:        REAL;       // Actual measurement
END_VAR
VAR_OUTPUT
    xFillValve:     BOOL;       // Fill valve command
    xFillComplete:  BOOL;       // Filling done
END_VAR
// ← Override this in derived classes

// Strategy 1: Volume-based
FUNCTION_BLOCK FB_FillStrategy_Volume EXTENDS FB_FillStrategy_Base
VAR_INPUT
    rFlowMeter:     REAL;       // Volume flow counter (L)
END_VAR
VAR
    rAccumVolume:   REAL;       // Accumulated volume
END_VAR
// xFillValve := (rAccumVolume < rTarget);
// xFillComplete := (rAccumVolume >= rTarget);
// Pre-close: เริ่มปิด valve ก่อนถึง target (lag compensation)

// Strategy 2: Weight-based
FUNCTION_BLOCK FB_FillStrategy_Weight EXTENDS FB_FillStrategy_Base
VAR_INPUT
    rLoadCell:      REAL;       // Weight measurement (g)
    rTareWeight:    REAL;       // Empty bottle weight
END_VAR
// xFillValve := ((rLoadCell - rTareWeight) < rTarget);
// xFillComplete := ((rLoadCell - rTareWeight) >= rTarget);

// Strategy 3: Time-based
FUNCTION_BLOCK FB_FillStrategy_Time EXTENDS FB_FillStrategy_Base
VAR
    tFillTimer:     TON;
END_VAR
// (rTarget treated as seconds)
// tFillTimer(IN := TRUE, PT := REAL_TO_TIME(rTarget × 1000));
// xFillValve := NOT tFillTimer.Q;
// xFillComplete := tFillTimer.Q;

// Context: FillingMachine selects strategy
FUNCTION_BLOCK FB_FillingMachine_v2
VAR_INPUT
    iStrategy:      INT;        // 1=Volume, 2=Weight, 3=Time
    rFillTarget:    REAL;
END_VAR
VAR
    fbVolumeStrategy:   FB_FillStrategy_Volume;
    fbWeightStrategy:   FB_FillStrategy_Weight;
    fbTimeStrategy:     FB_FillStrategy_Time;
    xFillValve_Out:     BOOL;
    xFillDone:          BOOL;
END_VAR

// CASE iStrategy OF
//     1: fbVolumeStrategy(rTarget := rFillTarget, ...);
//        xFillValve_Out := fbVolumeStrategy.xFillValve;
//        xFillDone := fbVolumeStrategy.xFillComplete;
//     2: fbWeightStrategy(rTarget := rFillTarget, ...);
//        xFillValve_Out := fbWeightStrategy.xFillValve;
//        xFillDone := fbWeightStrategy.xFillComplete;
//     3: fbTimeStrategy(rTarget := rFillTarget, ...);
//        xFillValve_Out := fbTimeStrategy.xFillValve;
//        xFillDone := fbTimeStrategy.xFillComplete;
// END_CASE;
```

---

## Step 1145: Template Method Pattern

### Template Method ใน Function Blocks

```
TEMPLATE METHOD PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Standard process with variable steps
├── CIP (Clean In Place) programs ที่คล้ายกัน
├── Machine types ที่มี base behavior เหมือนกัน
└── Batch recipe phases

ตัวอย่าง: CIP Cleaning Program
Base template: PreRinse → MainWash → FinalRinse → Drain
แต่ each machine ทำ steps ด้วย different parameters

// Base CIP Template
FUNCTION_BLOCK FB_CIP_Base
VAR_INPUT
    xStart:         BOOL;
    xStop:          BOOL;
END_VAR
VAR_OUTPUT
    xRunning:       BOOL;
    xComplete:      BOOL;
    sCIPPhase:      STRING;
END_VAR
VAR
    iPhase:         INT := 0;
    tPhaseTimer:    TON;
END_VAR

// Template steps (กำหนดใน base, override ใน child)
// PHASE 0: Idle
// PHASE 1: PreRinse (override duration, temperature)
// PHASE 2: MainWash (override chemical, temperature, duration)
// PHASE 3: FinalRinse (override duration)
// PHASE 4: Drain

// Child FB: CIP สำหรับ Dairy Tank (override parameters)
FUNCTION_BLOCK FB_CIP_DairyTank EXTENDS FB_CIP_Base
VAR
    // Override: Dairy-specific parameters
    rPreRinse_Duration:     REAL := 300.0;    // 5 min
    rMainWash_Temp:         REAL := 80.0;     // 80°C (NaOH)
    rMainWash_Duration:     REAL := 1800.0;   // 30 min
    rFinalRinse_Duration:   REAL := 600.0;    // 10 min
    rChemical_NaOH_Pct:     REAL := 2.0;      // 2% NaOH
END_VAR
```

---

## Step 1146: Singleton Pattern

### Global Enable / Singleton ใน PLC

```
SINGLETON PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Global machine enable (ทุก module ต้องเช็ค)
├── Shared resource (physical asset ที่ share ได้แค่คนเดียว)
└── System coordinator (Scheduler, Token passing)

Global Machine Enable (Safety + Logic):
```

```
// Global Machine Enable - Singleton Pattern
// ─────────────────────────────────────────────────────────────

// DB_GlobalEnable: เข้าถึงได้จากทุก FB
DATA_BLOCK DB_GlobalEnable
    xSafetyOK:              BOOL := FALSE;  // Safety system OK
    xEStopOK:               BOOL := FALSE;  // No E-Stop active
    xMaintenanceMode:       BOOL := FALSE;  // Maintenance mode
    xOperatorEnabled:       BOOL := FALSE;  // Operator authorization
    xAllPermissives:        BOOL := FALSE;  // All permissives met
    xGlobalEnable:          BOOL := FALSE;  // Final enable
    
    // Priority levels
    xProductionMode:        BOOL := FALSE;
    xJogMode:               BOOL := FALSE;
    xSetupMode:             BOOL := FALSE;
END_DATA_BLOCK

// FB_GlobalEnableManager: อัพเดท global enable
// (เรียกใน OB1 เป็นอันดับแรก)
FUNCTION_BLOCK FB_GlobalEnableManager
BEGIN
    // Network 1: Calculate Global Enable
    // DB_GlobalEnable.xGlobalEnable :=
    //     DB_GlobalEnable.xSafetyOK
    //     AND DB_GlobalEnable.xEStopOK
    //     AND NOT DB_GlobalEnable.xMaintenanceMode
    //     AND DB_GlobalEnable.xOperatorEnabled;
    
    // Network 2: Mode Exclusion (เปิดได้แค่ 1 mode)
    // (ถ้า Production ON → Jog และ Setup ต้อง OFF)
    // IF DB_GlobalEnable.xProductionMode THEN
    //     DB_GlobalEnable.xJogMode   := FALSE;
    //     DB_GlobalEnable.xSetupMode := FALSE;
    // END_IF;
END_FUNCTION_BLOCK

// ทุก FB ต้องเช็ค:
// IF DB_GlobalEnable.xGlobalEnable THEN
//     // ทำงานปกติ
// ELSE
//     // หยุดทำงาน
// END_IF;
```

---

## Step 1147: Interlock Matrix Pattern

### Interlock Matrix สำหรับ Safety

```
INTERLOCK MATRIX PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── ระบบที่มี equipment หลายตัวที่ต้องป้องกัน operation ที่อันตราย
└── Process interlocks ที่ complex

Interlock Matrix: กำหนดว่า A ทำงานได้ก็ต่อเมื่อ B,C,D เป็นอย่างไร

EXAMPLE: Chemical Process
┌─────────────────┬──────┬──────┬──────┬──────┬──────┐
│ Equipment       │ P101 │ V201 │ V202 │ V203 │ T301 │
├─────────────────┼──────┼──────┼──────┼──────┼──────┤
│ P101 (Pump)     │  -   │  ON  │ OFF  │  -   │ OK   │
│ V201 (Inlet V)  │  -   │  -   │  -   │  -   │ OK   │
│ V202 (Outlet V) │  ON  │  -   │  -   │  -   │  -   │
│ V203 (Bypass V) │  -   │  -   │  -   │  -   │  -   │
└─────────────────┴──────┴──────┴──────┴──────┴──────┘

P101 can start ONLY IF: V201 is OPEN AND V202 is OPEN AND T301 is OK
```

### Interlock Matrix Implementation

```
// Interlock Matrix ใน PLC
// ─────────────────────────────────────────────────────────────

DATA_BLOCK DB_InterlockMatrix
    // Column headers (conditions)
    bPump_P101_Running:     BOOL;
    bValve_V201_Open:       BOOL;
    bValve_V202_Open:       BOOL;
    bTank_T301_LevelOK:     BOOL;
    bTank_T301_TempOK:      BOOL;
    
    // Row headers (what can start)
    bP101_StartPermit:      BOOL;   // P101 ต้องการอะไร
    bV201_OpenPermit:       BOOL;   // V201 เปิดได้เมื่อไหร่
    bV202_OpenPermit:       BOOL;
    
    // Interlock violations (for diagnostics)
    bP101_Blocked_V201:     BOOL;   // P101 blocked เพราะ V201 ปิด
    bP101_Blocked_V202:     BOOL;   // P101 blocked เพราะ V202 ปิด
    bP101_Blocked_TankLvl:  BOOL;   // P101 blocked เพราะ level ต่ำ
END_DATA_BLOCK

// Calculate Permits:
// P101 Start Permit = V201_Open AND V202_Open AND T301_LevelOK AND T301_TempOK
// bP101_StartPermit := bValve_V201_Open AND bValve_V202_Open
//                      AND bTank_T301_LevelOK AND bTank_T301_TempOK;

// Violation details (สำหรับ HMI display - "Why can't I start?")
// bP101_Blocked_V201    := NOT bValve_V201_Open;
// bP101_Blocked_V202    := NOT bValve_V202_Open;
// bP101_Blocked_TankLvl := NOT bTank_T301_LevelOK;

// HMI Diagnostic Message:
// ถ้า P101 ไม่สามารถ start → แสดง:
// "P101 blocked: V201 closed" (ถ้า V201 ปิด)
// "P101 blocked: Low level T301" (ถ้า level ต่ำ)
```

---

## Step 1148: Heartbeat Pattern

### Heartbeat สำหรับ Communication Monitoring

```
HEARTBEAT PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── ตรวจว่า communication ยังมีชีวิตอยู่
├── Monitor ว่า remote device (SCADA, OPC) ยัง connected
└── Watchdog สำหรับ software faults

Architecture:
Device A → sends heartbeat (toggle bit) every 1 second → Device B
Device B → monitors: if heartbeat doesn't change in 3 seconds → alarm

// Heartbeat Transmitter (PLC ส่งไปยัง SCADA)
FUNCTION_BLOCK FB_HeartbeatTx
VAR_INPUT
    tInterval:  TIME := T#1s;   // Heartbeat interval
END_VAR
VAR_OUTPUT
    xHeartbeat: BOOL;           // Toggle this bit
    iCounter:   INT;            // Incrementing counter (alternative)
END_VAR
VAR
    tTimer: TON;
END_VAR
// tTimer(IN := TRUE, PT := tInterval);
// IF tTimer.Q THEN
//     tTimer(IN := FALSE);
//     xHeartbeat := NOT xHeartbeat;   // Toggle
//     iCounter := iCounter + 1;       // Or increment
// END_IF;

// Heartbeat Receiver (Device B monitors)
FUNCTION_BLOCK FB_HeartbeatRx
VAR_INPUT
    xHeartbeatIn:   BOOL;       // Received heartbeat bit
    tTimeout:       TIME := T#3s; // Timeout before alarm
END_VAR
VAR_OUTPUT
    bAlive:         BOOL;       // Communication alive
    bTimeout:       BOOL;       // Communication lost
END_VAR
VAR
    xLastHeartbeat: BOOL;
    tWatchdog:      TON;
END_VAR
// IF xHeartbeatIn <> xLastHeartbeat THEN   // Heartbeat changed
//     tWatchdog(IN := FALSE);   // Reset watchdog
//     xLastHeartbeat := xHeartbeatIn;
// END_IF;
// tWatchdog(IN := TRUE, PT := tTimeout);
// bAlive   := NOT tWatchdog.Q;
// bTimeout := tWatchdog.Q;
```

---

## Step 1149: Watchdog Pattern

### Watchdog สำหรับ Software Fault Detection

```
WATCHDOG PATTERN:
──────────────────────────────────────────────────────────────

Software Watchdog ใน PLC:
ใช้เมื่อ: OB1 อาจค้างเพราะ loop ไม่ออก, infinite wait state

// Software Watchdog ใน OB1
// ─────────────────────────────────────────────────────────────

DATA_BLOCK DB_Watchdog
    iWatchdogCounter:       INT := 0;   // Increments each scan
    iWatchdogLastReset:     INT := 0;   // Last reset value
    bWatchdogFault:         BOOL;       // Watchdog triggered
    tLastReset:             TIME;       // Last reset time
END_DATA_BLOCK

// OB1 (Main): Increment watchdog every scan
// DB_Watchdog.iWatchdogCounter := DB_Watchdog.iWatchdogCounter + 1;

// Critical FB must "pat" watchdog:
// Called from FB: DB_Watchdog.iWatchdogLastReset := DB_Watchdog.iWatchdogCounter;

// Watchdog Monitor OB (Cyclic Interrupt, every 100ms):
// IF (DB_Watchdog.iWatchdogCounter - DB_Watchdog.iWatchdogLastReset) > 10 THEN
//     // OB1 has not been patted for > 1 second (10 scans × 100ms)
//     DB_Watchdog.bWatchdogFault := TRUE;
//     // Take action: safe shutdown, alarm
// END_IF;

// External Watchdog (Hardware):
// PLC output → Watchdog relay → Motor enable relay
// PLC must toggle output bit ทุก < 1 วินาที
// ถ้า toggle หยุด → relay drops out → motor stops
// ป้องกัน: CPU fault, program crash
```

---

## Step 1150: Data Pipeline Pattern

### Data Pipeline สำหรับ Processing Data

```
DATA PIPELINE PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Raw sensor data → Engineering units → Validation → Historian
├── Recipe data → Parameter scaling → Safety check → Apply to equipment
└── Vision system data → Filter → Decision → Action

Pipeline Stages:
Stage 1: ACQUIRE (รับข้อมูลดิบ)
Stage 2: CONVERT (แปลงหน่วย)
Stage 3: VALIDATE (ตรวจสอบ)
Stage 4: FILTER (กรองสัญญาณรบกวน)
Stage 5: PROCESS (ประมวลผล)
Stage 6: DISTRIBUTE (ส่งไปยัง consumers)
```

```
// Analog Input Pipeline
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_AnalogPipeline
VAR_INPUT
    iRawAI:         INT;        // Stage 1: Raw ADC value (0-27648)
    rEU_Min:        REAL;       // Engineering unit minimum (e.g., 0.0 bar)
    rEU_Max:        REAL;       // Engineering unit maximum (e.g., 10.0 bar)
    rValidMin:      REAL;       // Valid range minimum (e.g., -0.5 bar)
    rValidMax:      REAL;       // Valid range maximum (e.g., 12.0 bar)
    rFilterTC:      REAL := 5.0; // Filter time constant (seconds)
    rScanTime:      REAL := 0.1; // Scan time (seconds)
END_VAR

VAR_OUTPUT
    rEU_Raw:        REAL;       // Stage 2: After conversion
    rEU_Validated:  REAL;       // Stage 3: After validation
    rEU_Filtered:   REAL;       // Stage 4: After filtering
    bValidSignal:   BOOL;       // Signal validity flag
    bLiveSignal:    BOOL;       // Signal in live range (4-20mA OK)
    bAlarm_High:    BOOL;       // Above valid max
    bAlarm_Low:     BOOL;       // Below valid min
    bWireBreak:     BOOL;       // Below 4mA (raw < 3000)
END_VAR

VAR
    rFilterAlpha:   REAL;       // Filter coefficient
END_VAR

// Stage 1→2: Convert raw AI to engineering units
// Siemens standard: 0=0mA, 6912=4mA, 27648=20mA
// rEU_Raw := (INT_TO_REAL(iRawAI) - 6912.0) / 20736.0 × (rEU_Max - rEU_Min) + rEU_Min;

// Stage 3: Validate
// bWireBreak := iRawAI < 3000;      // < 4mA → wire break
// bLiveSignal := (iRawAI >= 6912) AND (iRawAI <= 27648);  // 4-20mA range
// bAlarm_Low  := rEU_Raw < rValidMin AND bLiveSignal;
// bAlarm_High := rEU_Raw > rValidMax AND bLiveSignal;
// bValidSignal := bLiveSignal AND NOT bAlarm_Low AND NOT bAlarm_High;
//
// IF bValidSignal THEN
//     rEU_Validated := rEU_Raw;
// ELSE
//     rEU_Validated := 0.0;   // or last good value, or bad value
// END_IF;

// Stage 4: First-order Filter
// α = Ts / (Ts + TC)
// rFilterAlpha := rScanTime / (rScanTime + rFilterTC);
// rEU_Filtered := rEU_Filtered + rFilterAlpha × (rEU_Validated - rEU_Filtered);
// (Exponential Moving Average - low-pass filter)
```

---

## Step 1151: Event-Driven Pattern

### Event-Driven Programming ใน PLC

```
EVENT-DRIVEN PATTERN:
──────────────────────────────────────────────────────────────

PLC ทั่วไป: Scan-based (ตรวจสอบทุก scan)
Event-driven: ทำงานเฉพาะเมื่อ event เกิดขึ้น

ใน Siemens S7-1500:
├── OB40: Process Interrupt (hardware interrupt)
├── OB82: Diagnosis Alarm (PROFINET alarm)
├── OB83: Pull/Plug (module events)
├── OB120: Time Delay Interrupt
└── OB1: Main (scan-based, ยังจำเป็น)

OB40 (Hardware Interrupt) ตัวอย่าง:
ติดตั้ง: Digital input กำหนดเป็น "Hardware interrupt"
เมื่อ DI เปลี่ยน state → OB40 รัน ทันที (ไม่ต้องรอ scan)

// OB40: Hardware Interrupt Processing
// ─────────────────────────────────────────────────────────────
ORGANIZATION_BLOCK OB40_HardwareInterrupt
VAR_TEMP
    xInterruptInput:    BOOL;
    iInterruptAddress:  INT;
END_VAR

BEGIN
    // อ่าน interrupt source
    iInterruptAddress := OB40_MDL_ADDR;    // Which module triggered
    xInterruptInput   := OB40_POINT_ADDR;  // Which bit
    
    // Process interrupt immediately
    IF iInterruptAddress = 16#200 AND xInterruptInput = TRUE THEN
        // บน rising edge ของ specific input
        // เช่น: Product detection sensor → increment counter immediately
        DB_Counter.products := DB_Counter.products + 1;
        DB_Counter.lastDetected := NOW();
    END_IF;
    
    // สำคัญมาก: OB40 ต้องสั้น! < 1ms
    // ห้ามทำงานหนักใน interrupt OB
END_ORGANIZATION_BLOCK
```

---

## Step 1152: Timeout Pattern

### Timeout Pattern สำหรับ Safety

```
TIMEOUT PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── ป้องกัน equipment ทำงานนานเกินไป
├── Communication timeout
└── Phase timeout ใน batch process

// Universal Timeout FB
FUNCTION_BLOCK FB_Timeout
VAR_INPUT
    xMonitor:       BOOL;       // Signal to monitor (should be TRUE)
    tTimeout:       TIME;       // Time limit
    bEnable:        BOOL := TRUE; // Enable monitoring
END_VAR

VAR_OUTPUT
    bTimedOut:      BOOL;       // Timeout occurred
    rTimeRemaining: REAL;       // Seconds remaining
END_VAR

VAR
    tTimer:         TON;
END_VAR

// Timer runs when monitoring and NOT yet complete
// tTimer(IN := bEnable AND NOT xMonitor, PT := tTimeout);
// bTimedOut := tTimer.Q;
// rTimeRemaining := TIME_TO_REAL(tTimeout - tTimer.ET) / 1000.0;

// Usage Example:
// xValveOpen ← should become TRUE within 10 seconds
// FB_Timeout(xMonitor := xValveOpen, tTimeout := T#10s)
// IF bTimedOut → Alarm: "Valve failed to open"
//              → Initiate safe shutdown
```

---

## Step 1153: Retry Pattern

### Retry สำหรับ Communication

```
RETRY PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Modbus communication (อาจ fail เป็นครั้งคราว)
├── MQTT publish retry
└── OPC UA reconnection

// Retry Pattern สำหรับ Modbus Communication
FUNCTION_BLOCK FB_ModbusWithRetry
VAR_INPUT
    xTrigger:           BOOL;       // Trigger new request
    bEnable:            BOOL := TRUE;
    iMaxRetries:        INT := 3;   // Max attempts
    tRetryDelay:        TIME := T#2s; // Delay between retries
END_VAR

VAR_OUTPUT
    bSuccess:           BOOL;       // Request succeeded
    bFailed:            BOOL;       // All retries failed
    iRetryCount:        INT;        // Current retry count
END_VAR

VAR
    iAttempts:          INT;
    xTriggerOnce:       BOOL;       // One-shot
    fbModbus:           MB_CLIENT;   // Siemens Modbus FB
    tRetryTimer:        TON;
    bRequestActive:     BOOL;
END_VAR

// State: IDLE → REQUESTING → RETRY_WAIT → SUCCESS or FAIL

// CASE iState OF
//     0: // IDLE
//         IF xTrigger THEN iState := 1; iAttempts := 0; END_IF;
//     
//     1: // REQUESTING
//         fbModbus(REQ := TRUE, ...);
//         IF fbModbus.DONE THEN bSuccess := TRUE; iState := 0; END_IF;
//         IF fbModbus.ERROR THEN
//             iAttempts := iAttempts + 1;
//             IF iAttempts >= iMaxRetries THEN
//                 bFailed := TRUE; iState := 0;
//             ELSE
//                 iState := 2;  // Wait before retry
//             END_IF;
//         END_IF;
//     
//     2: // RETRY_WAIT
//         tRetryTimer(IN := TRUE, PT := tRetryDelay);
//         IF tRetryTimer.Q THEN
//             tRetryTimer(IN := FALSE);
//             iState := 1;  // Try again
//         END_IF;
// END_CASE;
```

---

## Step 1154: Object Pool Pattern

### Object Pool สำหรับ Multiple Instances

```
OBJECT POOL PATTERN:
──────────────────────────────────────────────────────────────

เหมาะสำหรับ:
├── Manage multiple identical equipment (pumps, motors)
├── Conveyor zones
└── Batch reactors (หลาย unit ทำงานพร้อมกัน)

// Pump Pool: 3 pumps, 2 ต้อง running (1 spare)
FUNCTION_BLOCK FB_PumpPool
VAR
    fbPumps:        ARRAY[1..3] OF FB_PumpControl;
    bActivePump:    ARRAY[1..3] OF BOOL;
    iActivePumps:   INT;        // จำนวน pump ที่ active
    iTargetPumps:   INT := 2;   // เป้าหมาย
END_VAR

// Strategy: Run pumps 1,2 by default
//           ถ้า pump 1 fail → activate pump 3 (spare)
//           Lead-lag rotation (ป้องกัน wear imbalance)

// Lead-Lag: สลับ lead pump ทุก 24 ชั่วโมง (เพื่อให้ wear เท่ากัน)
// IF tLeadLagTimer.Q THEN
//     RotatePumps();  // เปลี่ยน pump ที่ run นานสุดเป็น spare
// END_IF;
```

---

## Step 1155: Complete Application using Multiple Patterns

### Case Study: Pasteurizer Control

**การนำ Design Patterns มาใช้ร่วมกัน**

```
PASTEURIZER CONTROL - Using Multiple Patterns:
──────────────────────────────────────────────────────────────

Process:
1. Feed milk → Heat to 72°C
2. Hold at 72°C for 15 seconds
3. Cool to 4°C
4. Package

Patterns Used:
├── State Machine: Main process phases
├── Strategy: Heating method (steam vs hot water)
├── Command Queue: Recipe steps
├── Observer: Alarm distribution (to SCADA, SMS, email)
├── Singleton: Global machine enable
├── Heartbeat: Communication to SCADA monitoring
├── Template Method: CIP cleaning program
└── Data Pipeline: Temperature sensor processing

// MAIN ORCHESTRATOR FB
FUNCTION_BLOCK FB_Pasteurizer_Master
VAR
    // Pattern instances
    fbStateMachine:     FB_PasteurizerState;    // State machine
    fbHeatStrategy:     FB_HeatStrategy_Steam;  // Strategy
    fbCommandQueue:     FB_CommandQueue;         // Command pattern
    fbEventBus:         FB_EventBus;            // Observer
    fbGlobalEnable:     FB_GlobalEnableManager; // Singleton
    fbHeartbeat:        FB_HeartbeatTx;         // Heartbeat
    fbCIP:              FB_CIP_DairyTank;       // Template method
    fbTempPipeline:     FB_AnalogPipeline;      // Data pipeline
    
    // Interlock Matrix
    fbInterlocks:       FB_InterlockMatrix;
    
    // Timeouts
    fbHoldTimeout:      FB_Timeout;             // 15-sec hold
    fbCommTimeout:      FB_Timeout;             // SCADA heartbeat
END_VAR

// OB1 Execution Order:
// 1. fbGlobalEnable.Execute()       ← Singleton: update global enable
// 2. fbTempPipeline.Execute(...)    ← Pipeline: process sensors
// 3. fbInterlocks.Evaluate()        ← Interlock matrix
// 4. fbStateMachine.Execute(...)    ← State machine
// 5. fbHeatStrategy.Execute(...)    ← Strategy: based on current state
// 6. fbHeartbeat.Execute()          ← Heartbeat tx
// 7. fbCommTimeout.Monitor(...)     ← Monitor SCADA heartbeat
// 8. fbEventBus.Distribute()        ← Distribute events
```

---

## Step 1156: Pattern Documentation

### Documenting Patterns ใน PLC Code

```
DOCUMENTATION STANDARDS:
──────────────────────────────────────────────────────────────

1. FUNCTION BLOCK HEADER:
// ╔════════════════════════════════════════════════════════════╗
// ║  FB_FillingMachine                                         ║
// ║  Pattern: State Machine                                    ║
// ║  Description: Controls bottle filling sequence           ║
// ║  Author: Somchai Engineer                                 ║
// ║  Date: 2024-01-15                                         ║
// ║  Version: 2.1                                             ║
// ║  Revision: 2024-03-01 - Added timeout check              ║
// ╚════════════════════════════════════════════════════════════╝

2. STATE DOCUMENTATION:
// STATES:
// │ ID │ Name       │ Actions                  │ Transitions
// ├────┼────────────┼──────────────────────────┼─────────────
// │ 0  │ IDLE       │ All outputs off           │ →1 on Start
// │ 1  │ PRESTART   │ Check permissives         │ →2 on Ready
// │ 2  │ RUNNING    │ Normal operation          │ →3 on Stop
// │ 3  │ STOPPING   │ Controlled stop           │ →0 when done
// │ 4  │ FAULT      │ All outputs safe          │ →0 on Reset

3. INTERFACE DOCUMENTATION:
// INPUT: xStartCmd         - Rising edge starts machine
//        xEmergencyStop    - Immediate stop (any state)
//        xFaultInput       - Hardware fault
// OUTPUT: xConveyorRun     - Conveyor motor enable
//         eCurrentState    - State ID for HMI
//         sCycleState      - State name string for display
```

---

## Step 1157: Refactoring Legacy PLC Code

### วิธีปรับปรุง Code เก่าให้ใช้ Patterns

```
LEGACY CODE REFACTORING STRATEGY:
──────────────────────────────────────────────────────────────

BEFORE (Bad code):
// Network 1: Check start
//  xStartPB   xRunning    xReady
//  ──┤├─────────┤/├──────────( )──
//
// Network 2: Check stop
//  xStopPB   xRunning    xStopping
//  ──┤├────────┤├──────────( )──
//
// 50 networks ต่อมา... (nobody understands)

AFTER (State Machine Pattern):
CASE eState OF
    STATE_IDLE:
        // Clear comments, obvious transitions
        ...
    STATE_RUNNING:
        // Only running-related logic here
        ...
END_CASE;

Steps for Refactoring:
1. UNDERSTAND: อ่านและ document ว่า code ทำอะไร (อย่าเปลี่ยนก่อน)
2. TEST FIRST: สร้าง test cases สำหรับ current behavior
3. SMALL STEPS: refactor ทีละนิด, test ทุกครั้ง
4. PARALLEL: run old และ new code parallel ในช่วง transition
5. VERIFY: ยืนยันว่า behavior เหมือนเดิม

WARNING:
├── อย่า refactor ระหว่าง production run
├── อย่า refactor ทั้ง program พร้อมกัน
├── ทำใน test environment ก่อน
└── ได้รับ approval ก่อน implement ใน production
```

---

## Step 1158: Pattern Anti-Patterns

### สิ่งที่ต้องหลีกเลี่ยง

```
COMMON ANTI-PATTERNS ใน PLC PROGRAMMING:
──────────────────────────────────────────────────────────────

1. SPAGHETTI LOGIC ❌
ปัญหา: coil ถูก drive จาก 5 networks ต่างกัน
ผล: เมื่อเกิด fault ไม่รู้ว่า network ไหนทำ
แก้ไข: ใช้ State Machine หรือ กำหนด 1 coil = 1 network

2. TIMER ABUSE ❌
ปัญหา: ใช้ timer แทน state (เช่น "delay 5 sec" แทน proper state)
ผล: ยากต่อ troubleshoot, เปลี่ยน timing ยาก
แก้ไข: ใช้ State Machine ที่ชัดเจน

3. MAGIC NUMBERS ❌
ปัญหา: IF iValue = 25648 THEN ... (ไม่รู้ว่า 25648 คืออะไร)
แก้ไข: 
  CONST MAX_AI_VALUE := 27648;
  IF iValue = MAX_AI_VALUE THEN...

4. COPY-PASTE PROGRAMMING ❌
ปัญหา: code เดิมๆ ซ้ำ 10 ครั้ง
ผล: แก้ที่เดียวต้องแก้อีก 9 ที่
แก้ไข: ใช้ FB (Function Block) ที่ parameterized

5. GLOBAL DATA EVERYWHERE ❌
ปัญหา: ทุก FB เข้าถึง global DB โดยตรง
ผล: coupling สูง, test ยาก
แก้ไข: ผ่าน VAR_INPUT/OUTPUT ชัดเจน

6. MISSING STATE MACHINE ❌
ปัญหา: ใช้แค่ boolean flags แทน proper state
ผล: ไม่รู้ว่า machine อยู่ state ไหน, transition ผิดพลาด
แก้ไข: CASE statement + enum states
```

---

## Step 1159: Code Review Checklist

### Review PLC Code ก่อน Deploy

```
PLC CODE REVIEW CHECKLIST:
══════════════════════════════════════════════════════════════

DESIGN:
☐ Function blocks มี clear purpose (SRP - Single Responsibility)
☐ State machine ใช้ CASE/SWITCH ไม่ใช่ nested IF
☐ Magic numbers กำหนดเป็น CONST ทั้งหมด
☐ Interface (VAR_INPUT/OUTPUT) มี comment
☐ Global data เข้าถึงผ่าน interface ไม่ใช่ direct

SAFETY:
☐ Division by zero มีการป้องกัน (IF denominator > 0)
☐ Array index อยู่ใน bounds
☐ Safety-critical outputs ไม่ถูก force ได้
☐ E-stop ทำงานใน all states
☐ Interlock matrix มี documentation

QUALITY:
☐ ทุก coil/timer มี name ที่เข้าใจได้
☐ Comments อธิบาย WHY ไม่ใช่แค่ WHAT
☐ FB headers มีข้อมูล author/date/version
☐ State transitions มี documentation
☐ ทดสอบทุก state ที่เป็นไปได้

COMMUNICATION:
☐ Communication errors มี handling
☐ Timeout สำหรับ remote device
☐ Retry logic สำหรับ unreliable channels

DIAGNOSTICS:
☐ Counter สำหรับ equipment life tracking
☐ Event logging สำหรับ state changes
☐ Fault codes ที่ readable (ไม่ใช่แค่ number)
```

---

## Step 1160: Lab Project - Design Pattern Application

### Lab: Refactor Legacy Filling Machine Program

**โจทย์**: program เดิมของ filling machine มีปัญหา:
- ติด bugs บ่อย (overflow, state ผิด)
- Engineer ใหม่เข้าใจยาก
- ยากต่อการเพิ่ม features

**Tasks**:

```
LAB TASKS:
══════════════════════════════════════════════════════════════

Task 1: Analyze Current Code (20%)
─────────────────────────────────────────────────────────────
1. อ่าน legacy program (ให้ไว้ใน lab materials)
2. Document: states ปัจจุบัน (implicit)
3. ระบุ: anti-patterns ที่พบ
4. Document: I/O list สมบูรณ์
5. สร้าง current state diagram

Task 2: Refactoring Plan (20%)
─────────────────────────────────────────────────────────────
1. ออกแบบ proper state machine (5-7 states)
2. ระบุ patterns ที่จะใช้
3. เขียน test cases ก่อน refactor

Task 3: Implement Refactored Code (40%)
─────────────────────────────────────────────────────────────
1. Implement State Machine pattern:
   - IDLE, PRESTART, FILLING, CAPPING, COMPLETE, FAULT states
   - Clear entry/exit actions per state

2. Implement Data Pipeline สำหรับ sensors:
   - Flow meter signal processing
   - Level sensor validation

3. Implement Interlock Matrix:
   - Filling permissives
   - Capper permissives

4. Implement Observer สำหรับ alarms:
   - Level low → all subscribers notified
   - Fault → log + notify HMI + activate beacon

Task 4: Testing and Validation (20%)
─────────────────────────────────────────────────────────────
1. Test all state transitions
2. Test interlock violations (ดูว่า message ถูกต้อง)
3. Test fault and recovery
4. Test timeout scenarios
5. Inject communication fault → ดู retry logic

DELIVERABLES:
├── Before/After comparison (design)
├── TIA Portal project (refactored)
├── Test report (all test cases)
├── Code review report (by another student)
└── Lessons learned document

EVALUATION:
├── Pattern correctness: state machine ถูกต้อง
├── Code quality: readable, maintainable
├── Functionality: same behavior as before
├── Tests: all pass
└── Documentation: clear and complete

BONUS: Demonstrate HMI showing state name and
       "Why can't I start?" message from interlock matrix
```

---

## สรุป Part 58

| Pattern | เมื่อใช้ |
|---------|---------|
| State Machine | Sequential, mode-based control |
| Observer | Event notification to multiple targets |
| Command | Queued operations, recipe |
| Strategy | Interchangeable algorithms |
| Heartbeat | Communication liveness |
| Data Pipeline | Sensor processing chain |
| Interlock Matrix | Complex safety dependencies |

---

## Navigation

- [← Part 57: Energy Optimization](./part57_energy_optimization.md)
- [→ Part 59: Version Control for PLC Programs](./part59_version_control.md)
- [↑ สารบัญหลัก](../README.md)
