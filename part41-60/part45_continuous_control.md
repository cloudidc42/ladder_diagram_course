# Part 45: Continuous Process Control (Steps 881-900)

## ภาพรวม
Continuous Process Control เกี่ยวข้องกับการควบคุมตัวแปรกระบวนการ เช่น อุณหภูมิ ความดัน การไหล และระดับ อย่างต่อเนื่องเพื่อรักษาค่าที่ต้องการ

---

## Step 881: Process Variables (PV, SP, CV)

### ตัวแปรพื้นฐานของการควบคุมกระบวนการ

```
Control Loop Variables:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  SP (Setpoint) ──►┌─────────────┐                          │
│                   │  Controller │──► CV ──► Process         │
│  Error = SP - PV ─►   (PID)    │                           │
│                   └─────────────┘           │              │
│         ▲                                   │              │
│         │                                   ▼              │
│         └──────────── PV ◄──────── Sensor/Transmitter       │
│                                                             │
│  PV = Process Variable (ตัวแปรที่วัด เช่น อุณหภูมิ)        │
│  SP = Setpoint (ค่าที่ต้องการ)                              │
│  CV = Control Variable / Manipulated Variable               │
│       (สิ่งที่ controller ควบคุม เช่น valve opening)       │
│  Error (e) = SP - PV (ค่าความเบี่ยงเบิน)                   │
│                                                             │
│  PID Output:                                               │
│  MV = Kp×e + Ki×∫e dt + Kd×de/dt                          │
└─────────────────────────────────────────────────────────────┘
```

### ประเภทของ Process Variables

| ตัวแปร | สัญลักษณ์ | เครื่องมือวัด | หน่วย |
|--------|---------|------------|------|
| Temperature | TIC | RTD, Thermocouple | °C, °F |
| Pressure | PIC | Pressure transmitter | kPa, bar, psi |
| Flow | FIC | Coriolis, Vortex, DP | kg/h, L/min |
| Level | LIC | Radar, DP, Float | %, m |
| pH | AIC | pH electrode | pH |
| Conductivity | AIC | Conductivity probe | mS/cm |

---

## Step 882: Loop Types (Flow, Pressure, Temperature, Level)

### Flow Control Loop

```
Flow Control Loop:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Flow setpoint ──►[FIC-101]──► FCV-101 ──► Pipe           │
│                       ▲                        │           │
│                       └──── FT-101 ────────────┘           │
│                             (Flow Transmitter)             │
│                                                             │
│  Typical PID Settings for Flow:                            │
│  ├── Kp: 0.3 - 0.8 (aggressive, flow responds fast)       │
│  ├── Ti: 0.1 - 0.5 min (fast integral)                    │
│  ├── Td: 0 (ไม่ใช้ derivative - noise sensitive)           │
│  └── Scan: 100-500ms                                       │
│                                                             │
│  Challenges:                                               │
│  - Control valve dead band                                 │
│  - Nonlinear valve characteristic                         │
│  - Noise from turbulent flow                              │
└─────────────────────────────────────────────────────────────┘
```

### Temperature Control Loop

```
Temperature Control Loop:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Temp setpoint ──►[TIC-101]──► TCV-101 (Steam valve)      │
│                       ▲                     │              │
│                       └── TT-101 ─── Heat Exchanger        │
│                                             │              │
│                                      Process Fluid         │
│                                                             │
│  Typical PID Settings for Temperature:                     │
│  ├── Kp: 2 - 10 (moderate)                                 │
│  ├── Ti: 5 - 30 min (slow integral, large thermal mass)   │
│  ├── Td: 0.5 - 5 min (derivative helps with thermal lag)  │
│  └── Scan: 1-5 seconds                                     │
│                                                             │
│  Characteristics:                                          │
│  - Large dead time (transport delay)                      │
│  - Large thermal lag (time constant)                      │
│  - Asymmetric (heating faster than cooling)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 883: Cascade Control

### หลักการ Cascade Control

```
Cascade Control (Primary/Secondary):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  PRIMARY LOOP (outer/master):                              │
│  Slow variable (Temperature, Level)                        │
│                                                             │
│  SECONDARY LOOP (inner/slave):                             │
│  Fast variable (Flow, Pressure)                            │
│                                                             │
│  Example: Temperature → Flow Cascade                       │
│                                                             │
│  TIC-101 (Primary) ──► FIC-101.SP (Secondary) ──► FCV-101 │
│  (Reactor temp)         (Steam flow SP)            │       │
│      ▲                        ▲                    │       │
│      │                        └── FT-101 ──────────┘       │
│      └── TT-101 ─── Reactor                               │
│                                                             │
│  Benefits:                                                 │
│  1. Disturbances in steam flow corrected before           │
│     affecting temperature                                 │
│  2. Better overall control                               │
│  3. Faster response                                       │
└─────────────────────────────────────────────────────────────┘
```

### Cascade Control ใน TIA Portal (PID_Compact)

```pascal
// PRIMARY PID (Temperature Controller)
"PID_Primary"(
    Setpoint    := Temp_SP,       // °C setpoint
    Input       := TIC_101.PV,   // Temperature PV
    Output      => Flow_SP,      // Output → Secondary SP
    
    // Cascade mode: output range matches secondary SP range
    OutputUpperLimit := 100.0,   // Max flow SP (%)
    OutputLowerLimit := 0.0      // Min flow SP
);

// SECONDARY PID (Flow Controller)
"PID_Secondary"(
    Setpoint    := Flow_SP,      // จาก Primary output
    Input       := FIC_101.PV,  // Flow PV
    Output      => FCV_101.CV,  // Control valve command
    
    // Fast scan (secondary scans faster than primary)
    CycleTime   := T#100ms      // 100ms cycle
);
```

### Ladder Diagram - Cascade Mode Switch

```ladder
// Network: Cascade Mode Control
// เมื่อ Primary ใน Auto → Secondary ใช้ SP จาก Primary
// เมื่อ Primary ใน Manual → Secondary ใช้ local SP

//    Primary_Auto    Cascade_Active
//    ---[ ]--------------( )-------
+--[ TIC_101.Auto_Mode ]--+--( Cascade_Active )--+

// Secondary SP selection
//    Cascade_Active    FlowSP_from_Primary
//    ---[ ]---------[MOVE Primary.Output]--( FIC_101.SP )--
//    ---[/]---------[MOVE Local_Flow_SP]---( FIC_101.SP )--

+--[ Cascade_Active ]--+
                       |--[MOVE TIC_101.Output => FIC_101.SP]--+
                       
+--[ NOT Cascade_Active ]--+
                           |--[MOVE Local_Flow_SP => FIC_101.SP]--+
```

---

## Step 884: Ratio Control

### Ratio Control (การควบคุมอัตราส่วน)

```
Ratio Control:
┌─────────────────────────────────────────────────────────────┐
│  Application: ผสมส่วนประกอบตามอัตราส่วนที่กำหนด           │
│                                                             │
│  Example: Burner Air/Fuel Ratio = 15:1                     │
│                                                             │
│  Wild Flow (FT-FUEL) ──►[×15]──► FIC-AIR.SP               │
│                                      │                     │
│  Air Flow ───────────────────────────┘                     │
│  (measured, controlled)                                    │
│                                                             │
│  Ratio Calculation:                                        │
│  Flow_B_SP = Ratio × Flow_A_Actual                        │
│                                                             │
│  Implementation:                                           │
│  ├── Measure "wild" flow (fuel flow)                       │
│  ├── Multiply by ratio → becomes SP for controlled flow   │
│  └── PID controls air flow to this SP                    │
└─────────────────────────────────────────────────────────────┘
```

### Ratio Control Program

```pascal
// Ratio Control Implementation
// FC: Ratio_Control

VAR
    Fuel_Flow     : Real;    // Wild flow PV (kg/h)
    Air_SP        : Real;    // Calculated air flow SP
    Ratio         : Real := 15.0;  // Air/Fuel ratio
    Ratio_Correct : Real := 1.0;   // Ratio correction factor
END_VAR

// Calculate air SP from fuel flow and ratio
Air_SP := Fuel_Flow * Ratio * Ratio_Correct;

// Limit check
IF Air_SP > Max_Air_Flow THEN
    Air_SP := Max_Air_Flow;
    Fuel_Limit_Active := TRUE;  // Limit fuel if air at max
ELSIF Air_SP < Min_Air_Flow THEN
    Air_SP := Min_Air_Flow;
END_IF;

// Write to Air flow controller
FIC_Air.SP := Air_SP;
```

---

## Step 885: Split-Range Control

### Split-Range Control

```
Split-Range Control:
┌─────────────────────────────────────────────────────────────┐
│  Application: ควบคุมอุณหภูมิที่ต้องทั้ง Heating และ Cooling │
│                                                             │
│  Controller Output: 0-100%                                  │
│                                                             │
│  Output 0-50% → Cooling valve opens (0→100% open)         │
│  Output 50-100% → Heating valve opens (0→100% open)       │
│                                                             │
│  Output%   Cooling Valve   Heating Valve                   │
│  ────────  ─────────────  ─────────────                   │
│     0%         100%            0%                          │
│     25%         50%            0%                          │
│     50%          0%            0%                          │
│     75%          0%           50%                          │
│    100%          0%          100%                          │
│                                                             │
│  Deadband at 50%: ทั้ง 2 valve ปิด (Deadband zone)        │
└─────────────────────────────────────────────────────────────┘
```

### Split-Range Calculation

```pascal
// Split-Range Valve Position Calculation
// FC: SplitRange_Calc

VAR_INPUT
    Controller_Out : Real;  // PID output (0-100%)
END_VAR
VAR_OUTPUT
    Cool_Valve_Pos : Real;  // Cooling valve position
    Heat_Valve_Pos : Real;  // Heating valve position
END_VAR

// Cooling valve: 100% at 0% output, closes at 50%
IF Controller_Out <= 50.0 THEN
    Cool_Valve_Pos := (50.0 - Controller_Out) / 50.0 * 100.0;
    Heat_Valve_Pos := 0.0;
ELSE
    // Heating valve: opens from 50% to 100% output
    Heat_Valve_Pos := (Controller_Out - 50.0) / 50.0 * 100.0;
    Cool_Valve_Pos := 0.0;
END_IF;
```

---

## Step 886: Feedforward + Feedback Combination

### Feedforward Control

```
Feedback + Feedforward Combined:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  FEEDBACK (Corrective):                                     │
│  SP ──►[Error]──►[PID FB]──────────────────────► Output   │
│  PV ──────────►[────────]                         │        │
│                                                    │        │
│  FEEDFORWARD (Predictive):                         │        │
│  Disturbance ──►[FF Model]──────────────────────► + → CV  │
│                                                             │
│  Example: Heat Exchanger                                   │
│  ├── Feedback: Steam valve controls outlet temp            │
│  └── Feedforward: Inlet flow changes → pre-adjust valve   │
│                                                             │
│  FF Calculation:                                           │
│  FF_Output = (Flow_SP - Flow_Actual) × FF_Gain            │
│  Total_CV = FB_Output + FF_Output                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 887: Dead Time Compensation

### การชดเชย Dead Time

```
Dead Time (Transport Delay):
┌─────────────────────────────────────────────────────────────┐
│  Dead Time = เวลาที่ผ่านไปก่อนที่ process จะเริ่มตอบสนอง  │
│                                                             │
│  Example: Pipeline temperature                             │
│  Steam valve open ────────────────────► Temperature rises │
│                   ←── Dead Time (τd) ──►                   │
│                                                             │
│  Dead Time ทำให้ PID tuning ยากขึ้น:                       │
│  ├── Kp ต้องลดลง (ไม่งั้น oscillation)                    │
│  ├── Ti ต้องเพิ่มขึ้น                                      │
│  └── ประสิทธิภาพลดลง                                       │
│                                                             │
│  Tuning rule with dead time:                               │
│  λ (Lambda) tuning: Ti = τ (process time constant)        │
│  Kp = τ / (Kp_process × (τd + λ))                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 888: Smith Predictor Concept

### Smith Predictor สำหรับ Dead Time

```
Smith Predictor Block Diagram:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│           ┌──── Process Model ──────► Predicted PV         │
│           │     G(s)×e^(-τd×s)           │                 │
│           │                              │                  │
│  SP ──►[+]──►[PID]──► CV ──► Process──►[+]──► PV          │
│          ▲                              │                  │
│          │                    ┌─────────┘                  │
│          │         ┌─────► G(s) (no dead time)            │
│          │         │          │                            │
│          └─────────┴──────────┘                            │
│                                                             │
│  The Smith Predictor:                                       │
│  1. ใช้ Process Model เพื่อ predict PV ล่วงหน้า            │
│  2. PID เห็น process โดยไม่มี dead time                   │
│  3. ผลลัพธ์: สามารถใช้ PID gains สูงขึ้น                  │
│  4. Sensitive to model accuracy (ต้องการ model ที่ดี)     │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 889: Control Valve Sizing Basics

### การเลือกขนาด Control Valve

```
Control Valve Sizing (Liquid):
┌─────────────────────────────────────────────────────────────┐
│  Cv (Valve Flow Coefficient):                               │
│                                                             │
│  Cv = Q × √(G / ΔP)                                       │
│                                                             │
│  Where:                                                    │
│  Q  = Flow rate (US gallons per minute)                    │
│  G  = Specific gravity of fluid                            │
│  ΔP = Pressure drop across valve (psi)                     │
│                                                             │
│  Example:                                                  │
│  Q = 100 GPM, G = 1.0 (water), ΔP = 10 psi               │
│  Cv = 100 × √(1.0/10) = 100 × 0.316 = 31.6               │
│                                                             │
│  Select valve with Cv_max ≈ 1.3 × Cv_required            │
│  Cv_max = 1.3 × 31.6 = 41.1                              │
│                                                             │
│  Valve Characteristics:                                    │
│  ├── Linear: Flow ∝ Valve Opening (straightforward flow)  │
│  ├── Equal %: Same % change in flow per % valve travel    │
│  │   (preferred for pressure-drop dominated loops)         │
│  └── Quick opening: Most flow at small opening            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 890-900: Complete Process Control Loop Design

### โครงการ: Heat Exchanger Control System

#### 1. Process Description

```
Shell-and-Tube Heat Exchanger:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Process Fluid (Shell Side):                               │
│  ├── Inlet: Chemical at 30°C                              │
│  ├── Outlet: Must be 75°C ± 2°C                          │
│  └── Flow: 50 m³/h (varies 30-80 m³/h)                   │
│                                                             │
│  Heating Medium (Tube Side):                               │
│  ├── Steam at 4 bar (144°C)                               │
│  └── Condensate return                                    │
│                                                             │
│  Control Strategy:                                         │
│  ├── Primary: Temperature control (TIC-201)               │
│  ├── Feedforward: Process flow changes                    │
│  └── Dead time compensation via IMC tuning               │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Instrumentation List

```
Instruments for Heat Exchanger Control:

Tag         Description           Type              Range
─────────────────────────────────────────────────────────────
TT-201      Process outlet temp   RTD PT100         0-150°C
TT-202      Process inlet temp    RTD PT100         0-150°C  
FT-201      Process flow          Vortex meter      0-100 m³/h
PT-201      Steam pressure        Pressure xmtr     0-10 bar
TCV-201     Steam control valve   Globe, FO         2"
XV-201      Condensate drain      On/Off solenoid   1"
TSH-201     High temp switch      Bimetal switch    >90°C
```

#### 3. PID Tuning สำหรับ Heat Exchanger

```
Process Identification:
Step test results:
- Process Gain (Kp): 2.0 °C/%  (2°C change per 1% valve change)
- Dead time (τd): 30 seconds
- Time constant (τ): 5 minutes (300 seconds)
- Final value reached: 10°C per 5% step

IMC (Internal Model Control) Tuning:
λ (desired closed-loop time constant) = max(0.1τ, 0.8τd)
λ = max(0.1×300, 0.8×30) = max(30, 24) = 30 seconds

Kp = τ/(Kp_process × (τd + λ))
   = 300 / (2.0 × (30 + 30))
   = 300 / 120
   = 2.5 %/°C

Ti = τ = 300 seconds = 5 minutes
Td = τd/2 = 15 seconds (optional, if noise permits)
```

#### 4. Complete Ladder Program

```pascal
// ===================================================
// FB: HeatExchanger_Control
// Cascade + Feedforward Control for HE-201
// ===================================================

FUNCTION_BLOCK HeatExchanger_Control
VAR_INPUT
    // Measurements
    TT_201_PV    : Real;    // Outlet temperature (°C)
    TT_202_PV    : Real;    // Inlet temperature (°C)
    FT_201_PV    : Real;    // Process flow (m³/h)
    PT_201_PV    : Real;    // Steam pressure (bar)
    
    // Setpoints
    Temp_SP      : Real;    // Temperature setpoint (°C)
    
    // Mode control
    Auto_Mode    : Bool;    // TRUE = Auto, FALSE = Manual
    Manual_Out   : Real;    // Manual output (%)
END_VAR

VAR_OUTPUT
    TCV_201_Out  : Real;    // Steam valve position (%)
    Alarm_HH     : Bool;    // High-High temp alarm
    Alarm_H      : Bool;    // High temp alarm
    PID_Status   : Int;     // 0=OK, 1=Saturated, 2=Error
END_VAR

VAR
    // PID Instance
    PID_HE       : PID_Compact;  // TIA Portal PID block
    
    // Feedforward
    FF_Term      : Real;
    FF_Gain      : Real := 0.5;   // Feedforward gain (tunable)
    Flow_SP      : Real := 50.0;  // Nominal flow
    
    // Alarm limits
    HH_Limit     : Real := 85.0;  // High-High: 85°C
    H_Limit      : Real := 80.0;  // High: 80°C
    
    // Output
    Total_Out    : Real;
    Prev_Out     : Real;
END_VAR

BEGIN
    // ---- PID Control ----
    PID_HE(
        Setpoint    := Temp_SP,
        Input       := TT_201_PV,
        ManualEnable := NOT Auto_Mode,
        ManualValue := Manual_Out,
        Output      => Total_Out,
        
        Kp          := 2.5,
        Ti          := 300.0,  // 5 minutes in seconds
        Td          := 15.0    // 15 seconds derivative
    );
    
    // ---- Feedforward Calculation ----
    // เมื่อ flow เพิ่มขึ้น ต้องการ steam มากขึ้น
    FF_Term := (FT_201_PV - Flow_SP) * FF_Gain;
    
    // ---- Combine FB + FF ----
    IF Auto_Mode THEN
        TCV_201_Out := LIMIT(0.0, Total_Out + FF_Term, 100.0);
    ELSE
        TCV_201_Out := Manual_Out;
    END_IF;
    
    // ---- Alarms ----
    Alarm_HH := TT_201_PV > HH_Limit;
    Alarm_H  := TT_201_PV > H_Limit;
    
    // ---- Safety: Close valve on HH Alarm ----
    IF Alarm_HH THEN
        TCV_201_Out := 0.0;  // Force close steam valve
    END_IF;
END_FUNCTION_BLOCK
```

#### 5. Alarm Management

```
Alarm Configuration for Heat Exchanger:
┌─────────────────────────────────────────────────────────────┐
│  Tag: TT-201 (Outlet Temperature)                           │
│                                                             │
│  Alarm Level    Value    Priority   Action                  │
│  ─────────────  ─────    ────────   ──────────────────────  │
│  LL (Low-Low)   20°C     Critical   Open steam, alert ops  │
│  L  (Low)       30°C     High       Alert operator          │
│  H  (High)      80°C     High       Alert operator          │
│  HH (High-High) 85°C     Critical   Close steam, alert ops │
│                                                             │
│  Deadband: ±1°C (ป้องกัน chattering)                       │
│  On-delay: 5 seconds (ป้องกัน spurious alarms)            │
│                                                             │
│  EEMUA 191 Guideline:                                       │
│  Max alarm rate: < 1 alarm per 10 minutes (normal ops)     │
│  Unacceptable: > 10 alarms per 10 minutes                  │
└─────────────────────────────────────────────────────────────┘
```

#### 6. HMI Faceplate

```
HMI Faceplate for TIC-201 (Heat Exchanger Temperature):

┌──────────────────────────────────────────────────┐
│  TIC-201  Heat Exchanger Outlet Temperature      │
│  ──────────────────────────────────────────────  │
│                                                  │
│  PV: [  75.3 °C  ]    SP: [  75.0 °C  ]         │
│                                                  │
│  Error: +0.3°C   Output: 65%                    │
│                                                  │
│  ┌────────────────────────────────────┐          │
│  │ PV Trend (last 30 minutes)         │          │
│  │  80°C ─────────────────────────── │          │
│  │  75°C ─── ─── SP line ─── ─── ─── │          │
│  │  70°C     ____PV line_____         │          │
│  │  60°C  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │          │
│  └────────────────────────────────────┘          │
│                                                  │
│  Mode: [  AUTO  ]  [MANUAL]                      │
│                                                  │
│  Alarm Status: [  NORMAL  ]                      │
│                                                  │
│  [DETAIL] [TREND] [TUNE] [CLOSE]                 │
└──────────────────────────────────────────────────┘
```

#### 7. Advanced: Auto-Tuning

```pascal
// Auto-Tuning Trigger (Relay Feedback Method)
// ใช้ Relay Oscillation เพื่อหา Ultimate Gain และ Period

FUNCTION_BLOCK AutoTune_PID
VAR
    Relay_Amp    : Real := 5.0;    // Relay amplitude (% output)
    Oscillating  : Bool;           // Oscillation detected
    Period_u     : Real;           // Ultimate period (seconds)
    Kp_u         : Real;           // Ultimate gain
    
    // Calculated PID (Ziegler-Nichols)
    Kp_new       : Real;
    Ti_new       : Real;
    Td_new       : Real;
END_VAR

BEGIN
    // Auto-tuning steps:
    // 1. Apply relay (+/- 5% output around nominal)
    // 2. Detect sustained oscillation
    // 3. Measure period (Period_u)
    // 4. Calculate Ultimate Gain: Ku = 4×Relay_Amp/(π×Amplitude_of_PV)
    
    IF Tuning_Complete THEN
        // Tyreus-Luyben (less aggressive than Z-N)
        Kp_new := Kp_u / 2.2;
        Ti_new := Period_u * 2.2;  // seconds
        Td_new := Period_u / 6.3;  // seconds
        
        // Apply new PID parameters
        PID_HE.Kp := Kp_new;
        PID_HE.Ti := Ti_new;
        PID_HE.Td := Td_new;
    END_IF;
END_FUNCTION_BLOCK
```

---

## Lab Project 45: Multi-Loop Distillation Column Control

### โจทย์
ออกแบบ Control Strategy สำหรับ Distillation Column ที่ประกอบด้วย:

### Control Loops ที่ต้องออกแบบ
1. **Overhead Temperature Control** - Reflux ratio control
2. **Bottom Level Control** - Bottom product flow
3. **Pressure Control** - Condenser cooling water
4. **Feed Flow Ratio Control** - Feed/steam ratio
5. **Column Bottom Temperature** (Cascade with reboiler steam)

### งานที่ต้องทำ
- วาด P&ID ของระบบ
- เลือก Control Strategy (Cascade, Ratio, Feedforward)
- ทำ Dynamic Process Identification
- Tune PIDs ด้วย IMC method
- เขียน Control Program ใน TIA Portal หรือ Studio 5000
- ออกแบบ Alarm Management System (EEMUA 191 compliant)
- สร้าง HMI Faceplate สำหรับแต่ละ loop

---

## สรุป Part 45

| Control Strategy | ใช้เมื่อ | ข้อดี |
|----------------|---------|------|
| Simple PID | Process เดี่ยว ไม่ซับซ้อน | ง่าย ปรับแต่งง่าย |
| Cascade | มี disturbance ใน secondary | ตอบสนองเร็วขึ้น |
| Ratio | ต้องรักษาสัดส่วน | ควบคุม composition |
| Split-Range | ต้องทั้ง heat และ cool | ยืดหยุ่น |
| Feedforward | รู้ disturbance ล่วงหน้า | Proactive correction |
| Smith Predictor | Dead time สูง | ปรับ gain สูงขึ้นได้ |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 44: Batch Control ISA-88](part44_batch_control.md) |
| ► ถัดไป | [Part 46: Advanced Motion Control](part46_advanced_motion.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
