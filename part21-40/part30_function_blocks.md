# Part 30: Function Blocks (FB) (Steps 581-600)

## ภาพรวม
Function Block (FB) คือหัวใจสำคัญของ Structured Programming ใน PLC สมัยใหม่ ความสามารถในการเก็บสถานะ (Stateful) ทำให้ FB เหมาะกับการควบคุมอุปกรณ์ที่ต้องจำสถานะระหว่าง Scan เช่น Motor, PID Controller, Sequencer และ Timer บทนี้จะลงลึกถึงการออกแบบและใช้งาน FB อย่างมืออาชีพ รวมถึง Multi-Instance, Parameter Passing และ Library Design

---

## Step 581: FB vs FC - ความแตกต่างที่สำคัญ

```
┌──────────────────────────────────────────────────────────┐
│               FB (Function Block) vs FC (Function)       │
│                                                          │
│  FC (Function):                                         │
│  ┌────────────────────────────────────────────────┐     │
│  │         FC: Scale_AI                          │     │
│  │  IN: Raw_Value → ─┐                          │     │
│  │  IN: Eng_Min   → ─┤ Logic → OUT: Eng_Value   │     │
│  │  IN: Eng_Max   → ─┘                          │     │
│  │                                               │     │
│  │  ไม่มี Memory   │  ทำงานใหม่ทุก Scan         │     │
│  │  ไม่มี IDB     │  ผลลัพธ์ขึ้น IN เท่านั้น   │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  FB (Function Block):                                   │
│  ┌────────────────────────────────────────────────┐     │
│  │         FB: Motor_Control                     │     │
│  │                                               │     │
│  │  IN: Start → ─┐            ┌── OUT: Running  │     │
│  │  IN: Stop  → ─┤ Logic      └── OUT: Faulted  │     │
│  │  IN: OL    → ─┘                              │     │
│  │               ↕                              │     │
│  │  STATIC: Latch_Run, Run_Hours, Start_Count   │     │
│  │          (จำค่าระหว่าง Scan ใน IDB)          │     │
│  └────────────────────────────────────────────────┘     │
│         IDB: Motor1_Data    IDB: Motor2_Data            │
│  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │ Latch_Run = TRUE    │  │ Latch_Run = FALSE    │      │
│  │ Run_Hours = 1247.5  │  │ Run_Hours = 356.2    │      │
│  │ Start_Count = 423   │  │ Start_Count = 89     │      │
│  └─────────────────────┘  └─────────────────────┘      │
│  Motor 1 Instance           Motor 2 Instance            │
└──────────────────────────────────────────────────────────┘
```

---

## Step 582: Instance Data Blocks (IDB) เชิงลึก

```
// Instance DB คือ "หน่วยความจำส่วนตัว" ของ FB Instance

// เมื่อ Call FB10 ด้วย IDB ที่ต่างกัน:
// แต่ละ IDB เก็บค่า STATIC แยกกัน อิสระจากกัน

// IDB Structure (ภายใน DB - ดูจาก TIA Portal):
// DB1: Motor1_Data (Instance of FB10: Motor_Control)
// ┌─────────────────────────────────────────────────────┐
// │  [IN Parameters]        - มองเห็นแต่ไม่ควรแก้ตรงๆ │
// │    Start_Cmd: BOOL = FALSE                         │
// │    Stop_Cmd: BOOL = FALSE                          │
// │    OL_Trip: BOOL = FALSE                           │
// │                                                    │
// │  [OUT Parameters]       - มองเห็นแต่ไม่ควรแก้ตรงๆ │
// │    Motor_Running: BOOL = FALSE                     │
// │    Motor_Faulted: BOOL = FALSE                     │
// │                                                    │
// │  [STATIC Variables]     - FB เขียนค่านี้ข้าม Scan │
// │    Latch_Run: BOOL = FALSE                        │
// │    Run_Hours: REAL = 0.0                          │
// │    Start_Counter: DINT = 0                        │
// │    Run_Timer: IEC_TIMER                           │
// └─────────────────────────────────────────────────────┘

// DB2: Motor2_Data (Another Instance of FB10)
// ┌─────────────────────────────────────────────────────┐
// │  [STATIC Variables]                                │
// │    Latch_Run: BOOL = TRUE   ← Motor 2 กำลังวิ่ง   │
// │    Run_Hours: REAL = 2341.7 ← วิ่งไปแล้ว 2341 ชม.  │
// │    Start_Counter: DINT = 156                       │
// └─────────────────────────────────────────────────────┘

// การอ่าน STATIC ของ FB Instance จากภายนอก:
// "Motor1_Data".Run_Hours  → 0.0
// "Motor2_Data".Run_Hours  → 2341.7
// ใช้ใน HMI, Report, Maintenance Screen
```

---

## Step 583: FB Design - Motor Control Complete

```
// FB10: Motor_Control
// Version: 3.0 | Complete Motor Management
// Author: ทีมวิศวกรรม
// ──────────────────────────────────────────────────────

// [INTERFACE - TIA Portal Declaration]:
// IN:
//   Start_Cmd      : BOOL   // Start push button (NO)
//   Stop_Cmd       : BOOL   // Stop push button (NC=TRUE=Stop)
//   E_Stop         : BOOL   // Emergency Stop
//   OL_Trip        : BOOL   // Overload relay NC contact
//   Fault_Reset    : BOOL   // Fault acknowledgment button
//   Manual_Mode    : BOOL   // Manual override active
//   Manual_Run     : BOOL   // Manual run command (ใช้เมื่อ Manual_Mode=TRUE)
//   Interlock_OK   : BOOL   // External interlock conditions
//
// OUT:
//   Motor_Out      : BOOL   // Motor contactor output
//   Running        : BOOL   // Motor is running (status)
//   Faulted        : BOOL   // Fault present
//   Auto_Mode_Active : BOOL // System in auto mode
//
// STATIC:
//   Run_Latch      : BOOL   // Internal seal-in
//   Fault_Latch    : BOOL   // Latched fault
//   Run_Hours      : REAL   // Accumulated run hours
//   Start_Count    : DINT   // Total number of starts
//   Hour_Timer     : TON    // 1-hour timer for run hours
//   Trip_Delay     : TOF    // Delay to confirm OL trip
//
// TEMP: (ไม่เก็บข้าม Scan)
//   Allow_Start    : BOOL   // Temporary start condition
//   Reset_Valid    : BOOL   // Reset is valid

// ─────────────────────── BODY ──────────────────────────

NETWORK 1: // START CONDITIONS
// เงื่อนไขที่ต้อง True ทั้งหมดก่อน Start ได้
    ┌──[NOT #Fault_Latch]──[#Interlock_OK]──────(= #Allow_Start)──┐
    └─────────────────────────────────────────────────────────────┘

NETWORK 2: // AUTO MODE - SET RUN LATCH
    ┌──[NOT #Manual_Mode]──[#Start_Cmd]──[#Allow_Start]──────────┐
    │  Auto Mode   Start Button   เงื่อนไข OK                   │
    └──┬──────────────────────────────────────────────────────────┘
       └──────────────────────────────────────────(S #Run_Latch)─┐
                                                                  └┘

NETWORK 3: // MANUAL MODE - SET RUN LATCH
    ┌──[#Manual_Mode]──[#Manual_Run]──[#Allow_Start]─────────────┐
    │  Manual Mode   Manual Run    เงื่อนไข OK                   │
    └──┬──────────────────────────────────────────────────────────┘
       └──────────────────────────────────────────(S #Run_Latch)─┐
                                                                  └┘

NETWORK 4: // STOP CONDITIONS - RESET RUN LATCH
    ┌──[#Stop_Cmd]───────────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────────────┘
       └──────────────────────────────────────────(R #Run_Latch)─┐
                                                                  └┘
    ┌──[NOT #E_Stop]─────────────────────────────────────────────┐
    │  E-Stop ถูกกด (NC Contact Open = FALSE)                    │
    └──┬──────────────────────────────────────────────────────────┘
       └──────────────────────────────────────────(R #Run_Latch)─┐
                                                                  └┘

NETWORK 5: // OL FAULT DETECTION (with delay to avoid nuisance)
    ┌──[#OL_Trip]────────────────────────────────────────────────┐
    │  OL Relay Trip                                            │
    └──(TOF: #Trip_Delay  PT:T#2S)──[NOT #Trip_Delay.Q]─────────┐
    │  2 sec off-delay confirms real trip                       │
    └──┬──────────────────────────────────────────────────────────┘
       ├──────────────────────────────────────────(S #Fault_Latch)┐
       └──────────────────────────────────────────(R #Run_Latch)──┐
                                                                   └┘

NETWORK 6: // FAULT RESET
    ┌──[#Fault_Reset]──[NOT #OL_Trip]───────────────────────────┐
    │  Reset กด         OL ไม่ได้ Trip    ← ถึงรีเซ็ตได้       │
    └──┬──────────────────────────────────────────────────────────┘
       └─────────────────────────────────────────(R #Fault_Latch)┐
                                                                  └┘

NETWORK 7: // OUTPUT ASSIGNMENT
    ┌──[#Run_Latch]──────────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────────────┘
       ├──────────────────────────────────────────(= #Motor_Out)──┐
       └──────────────────────────────────────────(= #Running)────┐
                                                                   └┘
    ┌──[#Fault_Latch]────────────────────────────────────────────┐
    └──────────────────────────────────────────────(= #Faulted)──┐
                                                                  └┘
    ┌──[NOT #Manual_Mode]────────────────────────────────────────┐
    └──────────────────────────────────────────(= #Auto_Mode_Active)┐
                                                                    └┘

NETWORK 8: // COUNT STARTS (Rising Edge Detection)
// นับจำนวน Start แต่ละครั้ง
    ┌──[#Run_Latch]────────────────────────────────────────────┐
    │  R_TRIG (Rising Edge)                                    │
    └──(R_TRIG_Start)──[Output.Q]────(ADD: #Start_Count +1)───┐
                                                               └┘

NETWORK 9: // ACCUMULATE RUN HOURS
    ┌──[#Running]────────────────────────────────────────────┐
    └──(TON: #Hour_Timer  PT:T#1H)──[#Hour_Timer.Q]──────────┐
    │  Timer ครบ 1 ชั่วโมง                                   │
    └──(ADD: #Run_Hours + 1.0)──────────────────────────────┐
                                                             └┘
```

---

## Step 584: FB Design - PID Wrapper

```
// FB30: PID_Controller_Wrapper
// หุ้ม CONT_C ของ Siemens ให้ใช้งานง่ายขึ้น
// เพิ่ม: Bumpless Transfer, Anti-Windup, Output Clamping
//
// INTERFACE:
// IN:
//   Enable         : BOOL   // Enable PID
//   Auto_Mode      : BOOL   // TRUE=Auto, FALSE=Manual
//   Setpoint       : REAL   // Target Value
//   Process_Value  : REAL   // Actual Measured Value
//   Manual_Output  : REAL   // Output ใน Manual Mode
//   Kp             : REAL   // Proportional Gain
//   Ti_Seconds     : REAL   // Integral Time (seconds)
//   Td_Seconds     : REAL   // Derivative Time (seconds)
//   Out_Min        : REAL   // Minimum Output (%)
//   Out_Max        : REAL   // Maximum Output (%)
//   Cycle_Time_ms  : REAL   // Execution Cycle (ms)
//
// OUT:
//   Output         : REAL   // PID Output (0-100%)
//   Error          : REAL   // Current Error
//   In_Auto        : BOOL   // Auto Mode Active
//   Sat_High       : BOOL   // Output Saturated High
//   Sat_Low        : BOOL   // Output Saturated Low
//
// STATIC:
//   PID_Instance   : "CONT_C"   // Siemens CONT_C FB
//   Last_Auto_Mode : BOOL       // Previous Auto State
//   Prev_Output    : REAL       // Previous Output (Bumpless)

// ─────────────────── BODY (SCL) ────────────────────────

IF #Enable THEN
  // Bumpless Transfer Detection
  IF #Auto_Mode AND NOT #Last_Auto_Mode THEN
    // Manual → Auto: Set Internal Setpoint = PV
    // เพื่อไม่ให้ Output กระโดดเมื่อ Switch
    #PID_Instance.SP_INT := #Process_Value;
  END_IF;
  
  // Clamp Setpoint to valid range
  IF #Setpoint > 100.0 THEN
    #PID_Instance.SP_INT := 100.0;
  ELSIF #Setpoint < 0.0 THEN
    #PID_Instance.SP_INT := 0.0;
  ELSE
    #PID_Instance.SP_INT := #Setpoint;
  END_IF;
  
  // Configure CONT_C
  #PID_Instance(
    COM_RST := FALSE,
    MAN_ON  := NOT #Auto_Mode,
    P_SEL   := TRUE,
    I_SEL   := (#Ti_Seconds > 0.0),
    D_SEL   := (#Td_Seconds > 0.0),
    CYCLE   := REAL_TO_TIME(#Cycle_Time_ms),
    SP_INT  := #Setpoint,
    PV_IN   := #Process_Value,
    MAN     := #Manual_Output,
    GAIN    := #Kp,
    TI      := REAL_TO_TIME(#Ti_Seconds * 1000.0),
    TD      := REAL_TO_TIME(#Td_Seconds * 1000.0),
    LMN_HLM := #Out_Max,
    LMN_LLM := #Out_Min
  );
  
  // Output Assignment
  #Output := #PID_Instance.LMN;
  #Error  := #PID_Instance.ER;
  #In_Auto := NOT #PID_Instance.MAN_ON;
  #Sat_High := #PID_Instance.QLMN_HLM;
  #Sat_Low  := #PID_Instance.QLMN_LLM;
  
  // Save state for next Scan
  #Last_Auto_Mode := #Auto_Mode;
  
ELSE
  // Disabled: Force output to 0
  #Output := 0.0;
  #Error  := 0.0;
  // Reset CONT_C
  #PID_Instance(COM_RST := TRUE);
END_IF;
```

---

## Step 585: FB Design - Conveyor System

```
// FB40: Conveyor_System
// ควบคุม Conveyor พร้อมตรวจจับ Jam, Speed Feedback
//
// INTERFACE:
// IN:
//   Start_Auto     : BOOL   // Auto start command
//   Stop_Cmd       : BOOL   // Stop command
//   E_Stop         : BOOL   // Emergency stop
//   OL_Relay       : BOOL   // Motor overload (NC)
//   Encoder_Pulse  : BOOL   // Speed encoder pulse
//   Product_Sensor : BOOL   // Product detected sensor
//   No_Flow_Time   : TIME   // Timeout หาก Product หยุด
//
// OUT:
//   Motor_Forward  : BOOL   // Motor run output
//   Motor_Running  : BOOL   // Motor running status
//   Jam_Fault      : BOOL   // Jam detected
//   No_Product_Alarm : BOOL // No product timeout
//   Speed_RPM      : REAL   // Calculated speed
//   Product_Count  : DINT   // Products passed
//
// STATIC:
//   Run_Latch      : BOOL
//   Fault_Latch    : BOOL
//   Jam_Timer      : TON    // Jam detection timer
//   No_Flow_Timer  : TON    // No product flow timer
//   Last_Encoder   : BOOL   // For rising edge detection
//   Pulse_Counter  : DINT   // Encoder pulse count
//   Speed_Timer    : TON    // 1-second speed calc timer
//   Last_Pulse_Count : DINT // Pulse count at last speed calc
//   Last_Product   : BOOL   // For product counting

// ─────────────────── BODY ───────────────────────────────

NETWORK 1: // RUN LATCH
    ┌──[#Start_Auto]──[NOT #Fault_Latch]──[#E_Stop]────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(S #Run_Latch)┐
                                                            └┘
    ┌──[NOT #Stop_Cmd]──[NOT #E_Stop]──────────────────────┐
    └──────── // กรณี Stop: Reset Latch ──────────────────┘

NETWORK 2: // JAM DETECTION
// ถ้า Motor Running แต่ไม่มี Encoder Pulse → Jam
    ┌──[#Run_Latch]──[NOT #Encoder_Pulse]──(TON: #Jam_Timer PT:T#5S)─┐
    │                                                                  │
    └──(#Jam_Timer.Q)────────────────────(S #Fault_Latch)────────────┐
    └──(#Jam_Timer.Q)────────────────────(= #Jam_Fault)──────────────┐
                                                                       └┘

NETWORK 3: // ENCODER PULSE COUNTING
    ┌──[#Encoder_Pulse]──[NOT #Last_Encoder]────────────────────────┐
    │  Rising Edge                                                  │
    └──(ADD: #Pulse_Counter + 1)────────────────────────────────────┐
                                                                     └┘
    ┌──[ALWAYS]──────────────────────────────────────────────────────┐
    └──(= #Last_Encoder, #Encoder_Pulse)─────────────────────────────┐
                                                                      └┘

NETWORK 4: // SPEED CALCULATION (ทุก 1 วินาที)
    ┌──[ALWAYS]──(TON: #Speed_Timer PT:T#1S)──[#Speed_Timer.Q]──────┐
    └──┬──────────────────────────────────────────────────────────────┘
       │  Pulses per second = #Pulse_Counter - #Last_Pulse_Count
       │  Speed_RPM = Pulses_per_sec × 60 / Pulses_per_Rev
       └──(SCL: #Speed_RPM := (#Pulse_Counter - #Last_Pulse_Count)
                              * 60.0 / 100.0)  // 100 pulses/rev
       └──(= #Last_Pulse_Count, #Pulse_Counter)
       └──(R #Speed_Timer: Reset Timer for next 1 sec)

NETWORK 5: // PRODUCT COUNT
    ┌──[#Product_Sensor]──[NOT #Last_Product]──────────────────────┐
    │  Rising Edge on Product Sensor                               │
    └──(ADD: #Product_Count + 1)───────────────────────────────────┐
                                                                    └┘
    ┌──[ALWAYS]──────────────────────────────────────────────────────┐
    └──(= #Last_Product, #Product_Sensor)────────────────────────────┐
                                                                      └┘
```

---

## Step 586: Multi-Instance FB

```
// Multi-Instance: FB เรียก FB อื่นโดยเก็บ Data ใน IDB ตัวเอง

// ตัวอย่าง: FB50 (Machine_Control) ที่ใช้ FB10 (Motor_Control) ภายใน

// FB50 STATIC Variables:
// STATIC:
//   Motor_Pump     : FB10   // ← Multi-Instance: Motor for Pump
//   Motor_Agit     : FB10   // ← Multi-Instance: Motor for Agitator
//   PID_Temp       : FB30   // ← Multi-Instance: PID for Temp

// FB50 Body:
NETWORK 1: // Call Motor Pump
    ┌──[ALWAYS_ON]───────────────────────────────────────────────┐
    │  #Motor_Pump(                                             │
    │    Start_Cmd    := #Pump_Start,                          │
    │    Stop_Cmd     := #Pump_Stop,                           │
    │    E_Stop       := #E_Stop_Global,                       │
    │    OL_Trip      := #Pump_OL,                             │
    │    Fault_Reset  := #Fault_Reset_All,                     │
    │    Motor_Out    => #Pump_Contactor,                      │
    │    Running      => #Pump_Running,                        │
    │    Faulted      => #Pump_Faulted                         │
    │  );                                                       │
    └───────────────────────────────────────────────────────────┘

NETWORK 2: // Call Motor Agitator
    ┌──[ALWAYS_ON]───────────────────────────────────────────────┐
    │  #Motor_Agit(                                             │
    │    Start_Cmd    := #Agit_Start,                          │
    │    Stop_Cmd     := #Agit_Stop,                           │
    │    E_Stop       := #E_Stop_Global,                       │
    │    OL_Trip      := #Agit_OL,                             │
    │    Fault_Reset  := #Fault_Reset_All,                     │
    │    Motor_Out    => #Agit_Contactor,                      │
    │    Running      => #Agit_Running,                        │
    │    Faulted      => #Agit_Faulted                         │
    │  );                                                       │
    └───────────────────────────────────────────────────────────┘

// OB1 เรียก FB50 เพียงครั้งเดียว:
NETWORK 1:
    ┌──[ALWAYS_ON]───────────────────────────────────────────────┐
    │  FB50 "Machine_Control" [IDB: Machine1_DB]               │
    │  // Machine_DB เก็บ Data ของ Motor_Pump, Motor_Agit, PID  │
    │  // ทั้งหมดใน IDB เดียว                                   │
    └───────────────────────────────────────────────────────────┘
```

---

## Step 587: Allen-Bradley AOI (Add-On Instruction) เชิงลึก

```
// AOI คือ FB ของ Allen-Bradley ใน Studio 5000

// AOI: Temperature_Controller
// สร้างจาก: Add-On Instructions → New

// Parameters:
// Required Parameters (ทุก AOI ต้องมี):
//   EnableIn (BOOL): Enable input
//   EnableOut (BOOL): Enable output
//
// Custom Parameters:
// IN:
//   PV            REAL   // Process Variable (°C)
//   SP            REAL   // Setpoint (°C)
//   Kp            REAL   // Proportional Gain
//   Ki            REAL   // Integral (repeats/min)
//   Kd            REAL   // Derivative (min)
//   Auto_Mode     BOOL   // Auto=TRUE, Manual=FALSE
//   Man_Output    REAL   // Manual output (%)
// OUT:
//   CV            REAL   // Control Variable (%)
//   Error         REAL   // Current Error
//   High_Alarm    BOOL   // SP + Deviation
//   Low_Alarm     BOOL   // SP - Deviation

// LOCAL (เหมือน STATIC):
//   PID_Data     PD     // PID Control Data Structure
//   Prev_PV      REAL
//   Initialized  BOOL

// LOGIC (Ladder ใน AOI):
RUNG 1: // PID Instruction
    ┌──[Auto_Mode]────────────────────────────────────────────┐
    │  PID                                                    │
    │  Control Structure: PID_Data                           │
    │  Process Variable: PV                                  │
    │  Output: CV                                            │
    │  PID Gains (in PID_Data): Kp, Ki, Kd                  │
    └─────────────────────────────────────────────────────────┘

RUNG 2: // Manual Mode Passthrough
    ┌──[NOT Auto_Mode]──────────────────────────────────────┐
    └──(MOV Man_Output → CV)──────────────────────────────────┘

RUNG 3: // Error Calculation
    ┌──[ALWAYS_ON]───────────────────────────────────────────┐
    └──(CPT: Error = SP - PV)────────────────────────────────┘

// เรียกใช้ AOI:
RUNG 1:
    ┌──[EnableIn]────────────────────────────────────────────┐
    │  Temperature_Controller                               │
    │  PV         := AI_Temp_Actual                        │
    │  SP         := Temp_Setpoint                         │
    │  Kp         := PID_Gains.Kp                          │
    │  Ki         := PID_Gains.Ki                          │
    │  Kd         := PID_Gains.Kd                          │
    │  Auto_Mode  := System_Auto                           │
    │  CV         => Heater_Output                         │
    │  Error      => Temp_Error                            │
    └────────────────────────────────────────────────────────┘
```

---

## Step 588: Mitsubishi Function Block ใน GX Works3

```
// Mitsubishi FX5U - Function Block

// สร้าง FB ใน GX Works3:
// Project Tree → Program Files → Add New → Function Block
// ตั้งชื่อ: FB_MOTOR_CTL
// Language: Ladder (หรือ ST)

// FB_MOTOR_CTL Interface Definition:
// Input Variables (ประกาศใน VAR_INPUT):
//   bStart   : BOOL;   // Start Command
//   bStop    : BOOL;   // Stop Command
//   bOL      : BOOL;   // Overload NC Contact
//   bReset   : BOOL;   // Fault Reset
//
// Output Variables (ประกาศใน VAR_OUTPUT):
//   bRun     : BOOL;   // Run Output
//   bFault   : BOOL;   // Fault Output
//
// Static Variables (ประกาศใน VAR):
//   bLatch   : BOOL;   // Run Latch
//   tTimer   : TON;    // Run Timer (ถ้าต้องการ)

// FB Body (Ladder):
// Network 1: Latch Circuit
LD      bStart
OR      bLatch
ANI     bStop
ANI     bFault
OUT     bLatch

// Network 2: Fault Detection
LD      bOL          // OL ถูก Trip (NC Open = FALSE → แต่ในที่นี้สมมุติ NO)
SET     bFault
RST     bLatch

// Network 3: Fault Reset
LD      bReset
ANI     bOL
RST     bFault

// Network 4: Output
LD      bLatch
OUT     bRun

// เรียกใช้ FB ใน MAIN Program:
// Structured Text:
Motor1(
  bStart := X0,
  bStop  := X1,
  bOL    := X2,
  bReset := X3,
  bRun   => Y0,
  bFault => Y1
);

Motor2(
  bStart := X4,
  bStop  := X5,
  bOL    := X6,
  bReset := X7,
  bRun   => Y4,
  bFault => Y5
);

// FB Instance ต่างกัน Data ก็แยกกัน
// Motor1.bLatch ไม่ใช่ตัวเดียวกับ Motor2.bLatch
```

---

## Step 589: FB Inheritance และ Template Pattern

```
// Pattern: Base FB + Specialized FB

// Base FB: FB_Device_Base
// ──────────────────────────────────────────────────
// IN:  Enable, E_Stop, Fault_Reset
// OUT: Running, Faulted, Ready
// STATIC: Fault_Latch, Enable_Latch
// (Logic พื้นฐาน: Fault, Reset, Enable/Disable)
//
// Specialized: FB_Motor (ใช้ Logic ของ Base ผ่าน Multi-Instance)
// ──────────────────────────────────────────────────
// IN:  + Start_Cmd, Stop_Cmd, OL_Trip
// OUT: + Motor_Out, Run_Hours
// STATIC: Base: FB_Device_Base, Run_Latch, Hour_Timer
//
// Specialized: FB_Valve (ใช้ Logic ของ Base)
// ──────────────────────────────────────────────────
// IN:  + Open_Cmd, Close_Cmd, Limit_Open, Limit_Closed
// OUT: + Open_Coil, Close_Coil, Open_Status, Closed_Status
// STATIC: Base: FB_Device_Base, Motion_Timer

// ตัวอย่าง FB_Valve Body:
NETWORK 1: // Base Safety Logic
    ┌──[ALWAYS_ON]─────────────────────────────────────────┐
    │  #Base(                                             │
    │    Enable      := #Enable,                         │
    │    E_Stop      := #E_Stop,                         │
    │    Fault_Reset := #Fault_Reset,                    │
    │    Faulted     => #Fault_From_Base                 │
    │  );                                                 │
    └──────────────────────────────────────────────────────┘

NETWORK 2: // Valve Open Coil
    ┌──[#Open_Cmd]──[NOT #Fault_From_Base]──[#Enable]───────┐
    └──────────────────────────────────────────(= #Open_Coil)┐
                                                             └┘

NETWORK 3: // Valve Close Coil
    ┌──[#Close_Cmd]──[NOT #Open_Coil]──[NOT #Fault_From_Base]──┐
    └───────────────────────────────────────────(= #Close_Coil)─┐
                                                                 └┘

NETWORK 4: // Limit Switch Status
    ┌──[#Limit_Open]─────────────────────────────────────────┐
    └──────────────────────────────────────(= #Open_Status)──┐
                                                             └┘
    ┌──[#Limit_Closed]───────────────────────────────────────┐
    └──────────────────────────────────────(= #Closed_Status)─┐
                                                              └┘

NETWORK 5: // Motion Timeout (Valve ไม่ถึง Limit ใน 30 วินาที)
    ┌──[#Open_Coil]──[NOT #Limit_Open]────────────────────────┐
    └──┬──────────────────────────────────────────────────────┘
       └──(TON: #Motion_Timer PT:T#30S)──[#Motion_Timer.Q]────┐
          └──────────────────────────────────(S #Fault_Latch)─┐
                                                               └┘
```

---

## Step 590: FB Testing Strategy

```
┌──────────────────────────────────────────────────────────┐
│           FB Testing Strategy                            │
│                                                          │
│  1. Unit Test (Offline Simulation):                     │
│     - ใช้ TIA Portal Simulation / Emulator             │
│     - Force Input Values                               │
│     - ตรวจสอบ Output ตาม Truth Table                   │
│     - ตรวจสอบ STATIC Variables ใน Watch Table          │
│                                                          │
│  2. Truth Table Testing:                                │
│     FB_Motor_Control Test Cases:                       │
│                                                          │
│  Test | Start | Stop | E_Stop | OL  | Expected          │
│  ─────┼───────┼──────┼────────┼─────┼──────────────     │
│    1  │  ↑    │  1   │   1    │  0  │ Running=TRUE      │
│    2  │  0    │  0   │   1    │  0  │ Running=FALSE     │
│    3  │  1    │  1   │   1    │  0  │ Running=FALSE     │
│    4  │  1    │  1   │   0    │  0  │ Running=FALSE     │
│    5  │  1    │  1   │   1    │  1  │ Faulted=TRUE      │
│    6  │  0    │  1   │   1    │  0  │ (Reset clears)    │
│                                                          │
│  3. Stress Test:                                        │
│     - Start/Stop 1000 ครั้ง Run_Hours ถูกต้อง?         │
│     - OL Trip ทันทีหลัง Start                          │
│     - E_Stop ระหว่าง Running                           │
│                                                          │
│  4. Integration Test:                                   │
│     - Test FB ร่วมกับ FB อื่น (FB_Sequencer เรียก FB_Motor)│
│     - ดู Interaction ระหว่าง FBs                        │
│                                                          │
│  5. FAT (Factory Acceptance Test):                      │
│     - Test ด้วย Hardware จริง                          │
│     - Real Motor, Real Sensors                         │
└──────────────────────────────────────────────────────────┘
```

---

## Step 591: FB Parameter Best Practices

```
// DO: Good FB Design

// ตั้งชื่อ Parameter ชัดเจน:
IN:
  Temperature_Setpoint : REAL;  // ✓ ชัดเจน
  //Temp : REAL;               // ✗ ย่อเกินไป
  //Value1 : REAL;             // ✗ ไม่สื่อความหมาย

// ใส่ Default Value ที่สมเหตุสมผล:
IN:
  Scan_Time_ms    : REAL := 100.0;   // Default 100ms
  Derivative_Filter : REAL := 0.1;  // Default filter coefficient

// ใช้ IN_OUT สำหรับ Accumulators:
IN_OUT:
  Run_Hours_Total : REAL;  // เพื่อให้ภายนอกอ่านและ Reset ได้
  Product_Counter : DINT;

// Comment ทุก Parameter:
// Temperature_Setpoint: Target temperature in degrees Celsius (0-300°C)
// Kp: Proportional gain, dimensionless (typical 0.5-5.0)
// Ti_Seconds: Integral time in seconds (0=disable, typical 30-300)

// ──────────────────────────────────────────────────────────

// DON'T: Bad FB Design

// ✗ อย่า Hardcode Addresses ใน FB:
//   IF IW64 > 27648 THEN ... END_IF;  ← ผูกกับ Hardware Address

// ✗ อย่าใช้ Global Variables โดยตรง:
//   #Output := Global_Mode_Flag * #PID_Output;  ← Coupling กับ Global

// ✗ อย่าทำ FB ใหญ่เกินไป (> 100 Networks):
//   แตกเป็น FB ย่อยๆ ดีกว่า

// ✗ อย่าเปลี่ยน IN Parameters ใน Body:
//   #Start_Cmd := FALSE;  ← ผิด! IN Parameter อ่านได้อย่างเดียว

// ✗ อย่า Read OUT Parameters ใน Body:
//   IF #Motor_Out THEN ...  ← ควรอ่านจาก #Run_Latch แทน
```

---

## Step 592: Library Design Pattern

```
// ออกแบบ PLC Library สำหรับโรงงาน

// Library Structure:
// ┌─────────────────────────────────────────────────────┐
// │  Plant_Library v1.5                                │
// │                                                    │
// │  Tier 1: Utilities (FC - Stateless)               │
// │  ├── FC_Scale_AI      (Analog Scaling)            │
// │  ├── FC_Eng_To_Raw    (Reverse Scaling)           │
// │  ├── FC_CLAMPING      (Clamp to Min/Max)          │
// │  ├── FC_BCD_TO_INT    (BCD Conversion)            │
// │  └── FC_Check_Range   (Range Validation)          │
// │                                                    │
// │  Tier 2: Device Control (FB - Stateful)           │
// │  ├── FB_Motor_Control  (Motor Logic)              │
// │  ├── FB_Valve_Control  (Valve Logic)              │
// │  ├── FB_Conveyor       (Conveyor Logic)           │
// │  └── FB_Analog_AI      (Analog + Filter + Alarm)  │
// │                                                    │
// │  Tier 3: Process Control (FB - Complex)           │
// │  ├── FB_PID_Controller (PID Wrapper)              │
// │  ├── FB_Sequencer      (State Machine Base)       │
// │  └── FB_Batch_Timer    (Batch Timer)              │
// │                                                    │
// │  Tier 4: Communication (FB - Protocol)            │
// │  ├── FB_Modbus_Master  (Modbus RTU Master)        │
// │  ├── FB_Modbus_Slave   (Modbus RTU Slave)         │
// │  └── FB_VFD_Control    (Generic VFD Interface)    │
// └─────────────────────────────────────────────────────┘

// Version Naming Convention:
// v1.0.0 = Major.Minor.Patch
// Major: Breaking Change (Interface เปลี่ยน)
// Minor: New Feature (Backward Compatible)
// Patch: Bug Fix

// Changelog (ใน Block Comment):
// v1.5.0 - 2024-01-15 - Added Manual_Mode to FB_Motor
// v1.4.2 - 2023-12-20 - Fixed Run Hours calculation bug
// v1.4.0 - 2023-11-01 - Added FB_Batch_Timer
```

---

## Step 593: FB Versioning และ Change Management

```
┌──────────────────────────────────────────────────────────┐
│           FB Change Management                           │
│                                                          │
│  Rules for Safe FB Changes:                             │
│                                                          │
│  1. ห้ามเปลี่ยน Parameter Order:                        │
│     v1: Start_Cmd, Stop_Cmd, OL_Trip → Output          │
│     v2: Start_Cmd, OL_Trip, Stop_Cmd → Output ← WRONG! │
│     (ถ้าเปลี่ยน Order = Breaking Change = Major Version)│
│                                                          │
│  2. เพิ่ม Parameter ได้ ลบไม่ได้:                      │
│     v1: Start_Cmd, Stop_Cmd                            │
│     v2: Start_Cmd, Stop_Cmd, Fault_Reset ← OK!        │
│     v2: Start_Cmd ← WRONG! (ลบ Stop_Cmd)              │
│                                                          │
│  3. ใส่ Default Value เสมอเมื่อเพิ่ม Parameter:         │
│     Fault_Reset : BOOL := FALSE; // Default ปลอดภัย     │
│                                                          │
│  4. Test ก่อน Deploy:                                   │
│     - Offline Test ก่อน                                │
│     - Deploy ใน Test PLC ก่อน                         │
│     - Production ต้องได้ Approval                       │
│                                                          │
│  5. Document Breaking Changes:                          │
│     อธิบายว่าต้องแก้ Caller อะไรบ้าง                   │
│     เมื่อ Upgrade จาก v1 → v2                          │
└──────────────────────────────────────────────────────────┘
```

---

## Step 594: ตัวอย่างจริง - Chemical Dosing System

```
// Chemical Dosing System - Full FB Architecture

// MACHINE DESCRIPTION:
// ระบบ Dosing สารเคมี 2 ชนิดลงในถัง 1 ถัง
// - Chemical A Pump (FB_Motor)
// - Chemical B Pump (FB_Motor)
// - Mixer (FB_Motor)
// - Level Sensor → Analog AI (FB_Analog_AI)
// - pH Sensor → Analog AI (FB_Analog_AI)
// - Temp Sensor → Analog AI (FB_Analog_AI)
// - pH Controller → FB_PID
// - Dosing Sequence → FB_Sequencer

// DB Structure After Instantiation:
// ┌─────────────────────────────────────────────────────┐
// │  DB1: ChemA_Pump_IDB (FB_Motor_Control)            │
// │  DB2: ChemB_Pump_IDB (FB_Motor_Control)            │
// │  DB3: Mixer_IDB      (FB_Motor_Control)            │
// │  DB4: Level_AI_IDB   (FB_Analog_AI)               │
// │  DB5: pH_AI_IDB      (FB_Analog_AI)               │
// │  DB6: Temp_AI_IDB    (FB_Analog_AI)               │
// │  DB7: pH_PID_IDB     (FB_PID_Controller_Wrapper)  │
// │  DB8: Dosing_Seq_IDB (FB_Sequencer)               │
// └─────────────────────────────────────────────────────┘

// OB1 Structure:
NETWORK 1: // Process Analog Inputs
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  FB_Analog_AI [DB4: Level_AI_IDB]                 │
    │  Raw_Input := IW64                                │
    │  Eng_Min := 0.0  Eng_Max := 100.0                │
    │  Eng_Value → Level_Actual                         │
    └─────────────────────────────────────────────────────┘

    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  FB_Analog_AI [DB5: pH_AI_IDB]                    │
    │  Raw_Input := IW66                                │
    │  Eng_Min := 0.0  Eng_Max := 14.0                 │
    │  Eng_Value → pH_Actual                            │
    └─────────────────────────────────────────────────────┘

NETWORK 2: // pH Control
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  FB_PID_Controller_Wrapper [DB7: pH_PID_IDB]      │
    │  Process_Value := pH_Actual                       │
    │  Setpoint := pH_Setpoint                          │
    │  Auto_Mode := System_Auto                         │
    │  Output → ChemA_Flow_SP                           │
    └─────────────────────────────────────────────────────┘

NETWORK 3: // Dosing Sequencer
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  FB_Sequencer [DB8: Dosing_Seq_IDB]               │
    │  // ขั้นตอน: รอ Level → เปิด Pump A → รอ pH →     │
    │  // เปิด Pump B → เปิด Mixer → รอ Batch Complete   │
    └─────────────────────────────────────────────────────┘

NETWORK 4: // Motor Control
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  FB_Motor_Control [DB1: ChemA_Pump_IDB]           │
    │  Start_Cmd := Dosing_Start_PumpA                  │
    │  Motor_Out → Q4.0                                 │
    └─────────────────────────────────────────────────────┘
```

---

## Step 595: FB Diagnostic Interface

```
// เพิ่ม Diagnostic Interface ให้ FB ทุกตัว

// Pattern: Diagnostic OUT Group
// แยก Diagnostic Output ออกจาก Control Output ชัดเจน

// FB_Motor_Control: Diagnostic Outputs
// OUT (Diagnostic):
//   Diag_Run_Hours     : REAL    // Hours วิ่งสะสม
//   Diag_Start_Count   : DINT    // จำนวน Start ทั้งหมด
//   Diag_Last_Fault    : INT     // Fault Code ล่าสุด
//   Diag_State         : INT     // State machine state
//   Diag_Uptime_Ratio  : REAL    // % Time Running

// Fault Codes:
// 0 = No Fault
// 1 = Overload Trip
// 2 = E-Stop Active
// 3 = Interlock Fault
// 4 = Motion Timeout

// เหตุผล: HMI สามารถแสดง Diagnostic ได้
// Maintenance System สามารถ Log ค่าได้

// Diagnostic สำหรับ HMI Screen:
// Motor 1 Status:
// ┌───────────────────────────────────────────────┐
// │  Status: RUNNING                             │
// │  Run Hours: 2,341.7 hrs                      │
// │  Total Starts: 156 times                     │
// │  Uptime Ratio: 78.3%                         │
// │  Last Fault: Overload Trip (2024-01-10)      │
// └───────────────────────────────────────────────┘
```

---

## Step 596: FB ใน OB Cyclic (ทุก 100ms)

```
// บาง FB ต้องเรียกด้วยความถี่สูง เช่น PID

// OB35 (Cyclic 100ms):
NETWORK 1: // PID ต้องทำงานทุก 100ms
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  FB_PID_Controller [DB_pH_PID]                    │
    │  // Cycle_Time ต้องตรงกับ OB Period               │
    │  Cycle_Time_ms := 100.0                           │
    └─────────────────────────────────────────────────────┘

// OB1 (Main ~10ms):
// ไม่ต้องเรียก PID FB ที่นี่ เพราะ OB35 ทำแล้ว
// แต่ต้องการอ่าน Output ของ PID:
NETWORK 1: // อ่าน PID Output เพื่อควบคุม Valve
    ┌──[System_Auto]──────────────────────────────────────┐
    └──(MOVE: DB_pH_PID.Output → pH_Valve_SP)──────────────┐
                                                           └┘

// สำคัญ: อย่าเรียก FB เดิมจาก OB1 และ OB35 พร้อมกัน
// เพราะ PLC ไม่มี Thread Safety โดยปริยาย
// ถ้า OB35 กำลัง Execute FB อยู่ แล้ว OB1 เรียกอีก
// STATIC Data จะ Corrupt

// วิธีแก้: เรียก FB_PID จาก OB35 เท่านั้น
//          OB1 อ่าน Output จาก IDB โดยตรง
```

---

## Step 597: FB Comparison - Siemens vs AB vs Mitsubishi

```
┌──────────────────────────────────────────────────────────┐
│           FB Comparison Across Platforms                 │
│                                                          │
│  Feature          │ Siemens TIA   │ AB Studio 5000      │
│  ─────────────────┼───────────────┼──────────────────── │
│  FB Name          │ FB (Function  │ AOI (Add-On         │
│                   │  Block)       │  Instruction)       │
│  ─────────────────┼───────────────┼──────────────────── │
│  Instance Storage │ Instance DB   │ Backing Tag (UDT)   │
│  ─────────────────┼───────────────┼──────────────────── │
│  Static Variables │ STATIC in FB  │ LOCAL in AOI        │
│  ─────────────────┼───────────────┼──────────────────── │
│  Parameter Modes  │ IN/OUT/IN_OUT │ IN/OUT/IN_OUT       │
│                   │ /STATIC/TEMP  │ /LOCAL              │
│  ─────────────────┼───────────────┼──────────────────── │
│  Library          │ Global Library│ Template Project    │
│  ─────────────────┼───────────────┼──────────────────── │
│  Languages        │ LD/FBD/SFC/ST │ LD/FBD/ST          │
│  ─────────────────┼───────────────┼──────────────────── │
│  Call Syntax      │ FB Instance   │ AOI Instruction    │
│                   │ + IDB Name    │ + Backing Tag       │
│  ─────────────────┼───────────────┼──────────────────── │
│  Nested FB        │ Multi-Instance│ AOI within AOI      │
│                   │               │ (limited)           │
│  ─────────────────┼───────────────┼──────────────────── │
│                                                          │
│  Mitsubishi GX Works3:                                  │
│  FB Name: Function Block                                │
│  Instance Storage: กำหนดใน Variable Declaration        │
│  Parameter: VAR_INPUT / VAR_OUTPUT / VAR                │
│  Library: ใช้ Project Import/Export                     │
│  Languages: LD / ST / FBD                               │
└──────────────────────────────────────────────────────────┘
```

---

## Step 598: ตัวอย่างการใช้งาน FB จริง - Water Treatment Plant

```
// Water Treatment Plant - FB Architecture Example

// PROCESS:
// Raw Water → Coagulation → Sedimentation → Filtration → Chlorination → Clear Water

// FB Instances:
// ┌─────────────────────────────────────────────────────┐
// │  DB_Coag_Pump    (FB_Motor)  - สูบน้ำ ส่วน Coag    │
// │  DB_Flocculant_Pump (FB_Motor)- สูบ Flocculant     │
// │  DB_Mixer_1      (FB_Motor)  - กวนผสม Coagulation  │
// │  DB_Sedimentation (FB_Sequencer) - ควบคุม Sediment │
// │  DB_Filter_1     (FB_Conveyor) - ควบคุม Filter     │
// │  DB_Chlorine_Pump (FB_Motor) - สูบ Chlorine        │
// │  DB_Turbidity_AI (FB_Analog_AI) - ค่า Turbidity    │
// │  DB_pH_AI        (FB_Analog_AI) - ค่า pH           │
// │  DB_Chlorine_AI  (FB_Analog_AI) - ค่า Free Cl2     │
// │  DB_pH_PID       (FB_PID)    - ควบคุม pH           │
// │  DB_Cl2_PID      (FB_PID)    - ควบคุม Chlorine     │
// │  DB_Process_Ctrl (FB_Sequencer) - Main Sequence    │
// └─────────────────────────────────────────────────────┘

// OB1 โครงสร้าง:
// FC_Safety_Check
// ├── FC_Analog_Processing
// │   ├── FB_Analog_AI[DB_Turbidity_AI]
// │   ├── FB_Analog_AI[DB_pH_AI]
// │   └── FB_Analog_AI[DB_Chlorine_AI]
// ├── FC_Motor_Control
// │   ├── FB_Motor[DB_Coag_Pump]
// │   ├── FB_Motor[DB_Flocculant_Pump]
// │   ├── FB_Motor[DB_Mixer_1]
// │   └── FB_Motor[DB_Chlorine_Pump]
// ├── FC_Sequencer
// │   └── FB_Sequencer[DB_Process_Ctrl]
// └── FC_Alarms

// OB35 (100ms Periodic):
// FC_PID_Control
// ├── FB_PID[DB_pH_PID]
// └── FB_PID[DB_Cl2_PID]
```

---

## Step 599: FB Best Practices Summary

```
┌──────────────────────────────────────────────────────────┐
│           FB Best Practices                              │
│                                                          │
│  DESIGN:                                                │
│  ✓ Single Responsibility: 1 FB ทำ 1 อย่าง              │
│  ✓ Small FB: < 50 Networks/Rungs                        │
│  ✓ Meaningful Names: FB_Motor_Control ไม่ใช่ FB10       │
│  ✓ Document Every Parameter with comments              │
│  ✓ Default Values ที่ปลอดภัยสำหรับทุก IN               │
│                                                          │
│  SAFETY:                                                │
│  ✓ Initial State ปลอดภัย (Outputs = FALSE ตอน Start)   │
│  ✓ E-Stop มีผลต่อ FB เสมอ (Pass เป็น IN Parameter)      │
│  ✓ Fault Latching สำหรับ Critical Devices              │
│  ✓ Timeout สำหรับทุก Motion/State                       │
│                                                          │
│  REUSABILITY:                                           │
│  ✓ ไม่ Hardcode Physical Address ใน FB                  │
│  ✓ ทุก Hardware Reference ผ่าน Parameter               │
│  ✓ ใช้ Multi-Instance แทนการสร้าง IDB แยก             │
│                                                          │
│  MAINTENANCE:                                           │
│  ✓ Version Header Comment ใน Block Properties          │
│  ✓ Add Diagnostic Outputs (Run Hours, Start Count)     │
│  ✓ Use Constants ไม่ใช่ Magic Numbers                   │
│  ✓ Test FB แยกก่อน Integrate                           │
│                                                          │
│  PERFORMANCE:                                           │
│  ✓ Heavy Computation FB อยู่ใน Cyclic OB              │
│  ✓ ไม่ใช้ Array Size ใหญ่ใน STATIC โดยไม่จำเป็น        │
│  ✓ Monitor Scan Time เมื่อเพิ่ม FB จำนวนมาก           │
└──────────────────────────────────────────────────────────┘
```

---

## Step 600: Lab Project - Complete FB Library

**โจทย์:** สร้าง PLC Program สำหรับ Mixing Station โดยใช้ FB Library

```
┌──────────────────────────────────────────────────────────┐
│           Lab: Mixing Station FB Library                 │
│                                                          │
│  PROCESS DESCRIPTION:                                   │
│  Mixing Station มี:                                     │
│  - Feed Pump A (Motor) - ป้อนสารตั้งต้น A              │
│  - Feed Pump B (Motor) - ป้อนสารตั้งต้น B              │
│  - Mixer Motor - กวนผสม                                 │
│  - Level Sensor (Analog) - วัดระดับ 0-500L             │
│  - Temperature Sensor (Analog) - วัดอุณหภูมิ 0-200°C   │
│  - Heating Element (Analog Output) - ความร้อน 0-100%   │
│  - Temperature PID Controller                          │
│                                                          │
│  TASK 1 - Create FBs:                                  │
│  a) FB_Motor_Control (ตาม Spec ใน Step 583)            │
│  b) FB_Analog_AI (Scale + Filter + Alarm)              │
│  c) FB_Analog_AO (Scale Output)                        │
│  d) FB_PID_Wrapper (ตาม Step 584)                     │
│  e) FB_Mix_Sequencer (State Machine)                   │
│                                                          │
│  TASK 2 - Sequencer States:                            │
│  State 0: IDLE (รอ Start Command)                     │
│  State 1: FILL_A (เปิด Pump A จนถึง Level SP)         │
│  State 2: FILL_B (เปิด Pump B จนถึง Level SP)         │
│  State 3: HEAT (เปิด Heater จนถึง Temp SP ±2°C)       │
│  State 4: MIX (เปิด Mixer 10 นาที)                    │
│  State 5: COMPLETE (สัญญาณ Batch Complete)             │
│                                                          │
│  TASK 3 - OB1 Structure:                               │
│  - FC_Safety (E-Stop, Interlock)                       │
│  - FC_Analog_Process (AI/AO FBs)                       │
│  - FC_Motor_Control (3 Motors)                         │
│  - FC_Sequencer (Mix Sequence)                         │
│                                                          │
│  TASK 4 - OB35 (100ms):                               │
│  - FB_PID_Wrapper สำหรับ Temperature                   │
│                                                          │
│  TASK 5 - Test:                                        │
│  - Step through Sequence manually                      │
│  - Verify Alarms work correctly                        │
│  - Check PID tracks Setpoint                          │
│                                                          │
│  TASK 6 - Documentation:                               │
│  - Architecture Diagram                               │
│  - FB Interface Specifications                         │
│  - Test Results                                       │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 30

ใน Part นี้ได้เรียนรู้:

1. **FB vs FC** - Stateful vs Stateless, เมื่อใช้อะไร
2. **Instance Data Blocks** - หน่วยความจำส่วนตัวของ FB Instance
3. **FB_Motor_Control** - Complete Motor FB Design
4. **FB_PID_Wrapper** - PID FB พร้อม Bumpless Transfer
5. **FB_Conveyor** - Conveyor พร้อม Jam Detection
6. **Multi-Instance** - FB เรียก FB อื่นใน IDB เดียว
7. **AOI (Allen-Bradley)** - FB Equivalent ใน Studio 5000
8. **Mitsubishi FB** - GX Works3 Function Block
9. **Library Design** - Tier Structure, Versioning
10. **Best Practices** - Design, Safety, Reusability
11. **Testing Strategy** - Unit Test, Integration Test

---

## Navigation
- [← Part 29: Structured Programming](part29_structured_programming.md)
- [กลับหน้าหลัก](../README.md)

---

## สรุปภาพรวม Parts 21-30

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 21 | Sequencer & State Machine | 391-410 |
| 22 | Analog I/O & Scaling | 411-430 |
| 23 | PID Control | 431-460 |
| 24 | Communication Protocols | 461-480 |
| 25 | Modbus RTU/TCP | 481-500 |
| 26 | PROFIBUS DP | 501-520 |
| 27 | PROFINET | 521-540 |
| 28 | EtherNet/IP | 541-560 |
| 29 | Structured Programming | 561-580 |
| 30 | Function Blocks (FB) | 581-600 |
