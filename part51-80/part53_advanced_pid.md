# Part 53: Advanced PID และ Process Control (Steps 1041-1060)

## บทนำ: เหนือกว่า PID พื้นฐาน

PID พื้นฐานทำงานได้ดีสำหรับ process ที่ simple แต่สำหรับ process ที่ซับซ้อนกว่านั้น เช่น:
- Process ที่มี dead time มาก (เช่น ท่อยาว, Kiln)
- Process ที่ interacting กัน (pH, distillation column)
- Process ที่ non-linear (pH neutralization)
- Process ที่ต้องการ energy efficiency สูง

เราต้องใช้ Advanced Control Strategies

---

## Step 1041: Cascade PID Control

### หลักการ Cascade Control

```
Cascade Control จะดีกว่า Single PID เมื่อ:
1. Inner loop disturbance เกิดบ่อยและเร็ว
2. Inner loop มี faster dynamics
3. Actuator มี nonlinearity (valve stiction)

Example: Steam Boiler Temperature Control
────────────────────────────────────────

SINGLE PID (ไม่มี Cascade):
Temperature SP → [PID] → Fuel valve → Burner → Temperature

ปัญหา: ถ้า fuel pressure เปลี่ยน PID จะตอบสนองช้า

CASCADE PID (แก้ปัญหา):
Temp SP → [Outer PID] → Flow SP → [Inner PID] → Fuel valve → Temp
           (Temperature)            (Flow)

Outer loop: Temperature controller (slow)
Inner loop: Fuel flow controller (fast - 5-10x faster)

ผล: Inner loop compensates fuel pressure changes quickly
    Outer loop remains stable
```

### Cascade PID ใน Ladder Diagram (Siemens S7-1500)

```
// Siemens S7-1500 - Cascade PID Implementation
// ใช้ FB "PID_Compact" (2 instances)

// ─────────────────────────────────────────────────────────────
// DATA BLOCK: DB_CASCADE_CONTROL
// ─────────────────────────────────────────────────────────────
// Outer Loop (Temperature):
OUTER_PID:
    SP:            REAL := 200.0;    // Temperature setpoint °C
    PV:            REAL;             // Temp measurement (from thermocouple)
    CV:            REAL;             // Output = Flow SP for inner loop
    Kp:            REAL := 5.0;
    Ti:            REAL := 120.0;   // seconds (slow)
    Td:            REAL := 15.0;
    CV_Min:        REAL := 0.0;
    CV_Max:        REAL := 100.0;   // % flow setpoint

// Inner Loop (Fuel Flow):
INNER_PID:
    SP:            REAL;             // From outer loop output
    PV:            REAL;             // Flow measurement
    CV:            REAL;             // Valve position %
    Kp:            REAL := 2.0;
    Ti:            REAL := 15.0;    // seconds (fast)
    Td:            REAL := 1.0;
    CV_Min:        REAL := 0.0;
    CV_Max:        REAL := 100.0;

// ─────────────────────────────────────────────────────────────
// FUNCTION BLOCK: FB_CascadeControl
// ─────────────────────────────────────────────────────────────
FUNCTION_BLOCK FB_CascadeControl
VAR_INPUT
    iEnable:        BOOL;           // เปิดใช้งาน cascade
    rTempSP:        REAL;           // Temperature setpoint
    rTempPV:        REAL;           // Temperature measurement
    rFlowPV:        REAL;           // Flow measurement
    bAutoMode:      BOOL;           // AUTO/MANUAL mode
    rManualCV:      REAL;           // Manual output (0-100%)
END_VAR

VAR_OUTPUT
    rValveCV:       REAL;           // Valve position output
    rFlowSP:        REAL;           // Flow setpoint (diagnostic)
    bOuterActive:   BOOL;           // Outer loop status
    bInnerActive:   BOOL;           // Inner loop status
END_VAR

VAR
    fbOuterPID:     PID_Compact;    // Outer PID instance
    fbInnerPID:     PID_Compact;    // Inner PID instance
    rInnerSP:       REAL;           // ค่า SP จาก outer loop
END_VAR

// ─── Network 1: Outer PID (Temperature) ───────────────────────
//
//  [iEnable]──┤├──── [PID_Compact: fbOuterPID] ────────────────
//
//  fbOuterPID(
//      Setpoint    := rTempSP,
//      Input       := rTempPV,
//      ManualValue := rManualCV,
//      Mode        := (bAutoMode ? 3 : 4),  // 3=Auto, 4=Manual
//      Output      => rInnerSP,
//      Active      => bOuterActive
//  );

// ─── Network 2: Clamp Inner Setpoint ──────────────────────────
//  ป้องกัน Inner SP เกิน range
//  rInnerSP := LIMIT(0.0, rInnerSP, 100.0);

// ─── Network 3: Inner PID (Flow) ──────────────────────────────
//
//  [iEnable]──┤├──── [PID_Compact: fbInnerPID] ────────────────
//
//  fbInnerPID(
//      Setpoint    := rInnerSP,
//      Input       := rFlowPV,
//      Mode        := (bAutoMode ? 3 : 4),
//      Output      => rValveCV,
//      Active      => bInnerActive
//  );

// ─── Network 4: Assign Outputs ────────────────────────────────
//  rFlowSP := rInnerSP;     // diagnostic output
```

---

## Step 1042: Ratio Control

### หลักการ Ratio Control

```
Ratio Control: ใช้เพื่อรักษา ratio ระหว่าง 2 flow rates
ตัวอย่าง: Combustion air-to-fuel ratio (Air/Fuel = 10:1)

Architecture:
────────────────────────────────────────────
     Fuel Flow ──┤× Ratio ├──→ Air SP ──→ [PID] ──→ Air Valve
         │
         └────────────────────→ Fuel Control

หรือ: ผสมสารเคมี A กับ B ในอัตราส่วน 3:1

Flow A ──┤× 3.0 ├──→ SP_B ──→ [PID_B] ──→ Valve_B
   │
   └───────────────→ [PID_A] ──→ Valve_A (track SP_A)
```

### Ratio Control Implementation

```
// Ratio Control ใน PLC
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_RatioControl
VAR_INPUT
    iEnable:        BOOL;
    rLeadSP:        REAL;           // Lead stream setpoint (e.g., Fuel)
    rLeadPV:        REAL;           // Lead stream actual flow
    rLagPV:         REAL;           // Lag stream actual flow
    rDesiredRatio:  REAL := 10.0;   // Desired Air/Fuel ratio
    rRatioMin:      REAL := 8.0;    // Safety: minimum acceptable ratio
    rRatioMax:      REAL := 15.0;   // Safety: maximum acceptable ratio
END_VAR

VAR_OUTPUT
    rLagSP:         REAL;           // Lag stream setpoint (Air)
    rActualRatio:   REAL;           // Actual measured ratio
    bRatioOK:       BOOL;           // Ratio within limits
    bRatioHigh:     BOOL;           // Ratio too high alarm
    bRatioLow:      BOOL;           // Ratio too low alarm
END_VAR

VAR
    fbLagPID:       PID_Compact;    // Lag stream PID
    rCalcLagSP:     REAL;           // Calculated lag setpoint
END_VAR

// Network 1: คำนวณ Lag SP จาก ratio
// rCalcLagSP := rLeadPV * rDesiredRatio;
// (ถ้า Fuel Flow = 10 m3/h, Air SP = 10 × 10 = 100 m3/h)

// Network 2: Clamp Lag SP within safe limits
// rCalcLagSP := LIMIT(0.0, rCalcLagSP, 200.0);

// Network 3: คำนวณ Actual Ratio
// IF rLeadPV > 0.1 THEN  // ป้องกัน division by zero
//     rActualRatio := rLagPV / rLeadPV;
// ELSE
//     rActualRatio := 0.0;
// END_IF;

// Network 4: Ratio Alarm Checks
// bRatioOK  := (rActualRatio >= rRatioMin) AND (rActualRatio <= rRatioMax);
// bRatioLow  := rActualRatio < rRatioMin;
// bRatioHigh := rActualRatio > rRatioMax;

// Network 5: PID ควบคุม Lag stream
// fbLagPID(
//     Setpoint := rCalcLagSP,
//     Input    := rLagPV,
//     Output   => rLagSP
// );
```

---

## Step 1043: Feedforward-Feedback Control

### หลักการ Feedforward Control

```
Pure Feedback PID: แก้ไขหลังจาก disturbance ส่งผลแล้ว
Feedforward: ชดเชย disturbance ก่อนที่จะส่งผล

Classic Example: Heat Exchanger
──────────────────────────────

Feedback only:
  Inlet temp → ผ่าน HX → Outlet temp → [PID] → Steam valve
  ปัญหา: ถ้า inlet temp เปลี่ยน ต้องรอ outlet เปลี่ยนก่อน

Feedforward + Feedback:
  Inlet temp ─→ [FF Calculator] ─→ (+)─→ Steam valve
                                    │
  Outlet temp → [PID Feedback] ────→ (+)─→

FF Calculator: 
  FF Output = K_FF × (Inlet Temp - Inlet Temp nominal)
  
  ถ้า inlet เย็นลง 10°C → FF เปิด steam valve ก่อนทันที
  PID Feedback → Fine-tune เพื่อ compensate error ที่เหลือ
```

### Feedforward Implementation

```
// Feedforward + Feedback ใน SCL (Siemens)
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_FeedforwardControl
VAR_INPUT
    rOutletSP:      REAL := 80.0;   // เป้าหมาย outlet temp (°C)
    rOutletPV:      REAL;           // วัด outlet temp
    rInletPV:       REAL;           // วัด inlet temp
    rInletNominal:  REAL := 20.0;   // Nominal inlet temp
    rFF_Gain:       REAL := 1.2;    // Feedforward gain (tune)
    rFF_LeadTime:   REAL := 5.0;    // Lead time (seconds)
    rFF_LagTime:    REAL := 30.0;   // Lag time (seconds)
END_VAR

VAR_OUTPUT
    rValvePosition: REAL;           // Steam valve output (0-100%)
    rFF_Output:     REAL;           // FF contribution (diagnostic)
    rFB_Output:     REAL;           // FB contribution (diagnostic)
END_VAR

VAR
    fbFeedbackPID:  PID_Compact;    // Feedback PID
    rFF_Calc:       REAL;           // FF calculation
    rFF_Filtered:   REAL;           // FF after lead-lag filter
    rCombinedOutput:REAL;
END_VAR

// Network 1: Feedback PID
// fbFeedbackPID(
//     Setpoint := rOutletSP,
//     Input    := rOutletPV,
//     Output   => rFB_Output
// );

// Network 2: Feedforward Calculation
// rFF_Calc := rFF_Gain × (rInletNominal - rInletPV);
// (ถ้า inlet เย็นกว่า nominal → FF_Calc > 0 → เปิด steam มากขึ้น)

// Network 3: Lead-Lag Filter สำหรับ FF (optional แต่ดีกว่า)
// Lead-Lag: H(s) = (1 + τ_lead × s) / (1 + τ_lag × s)
// ใน discrete time (Euler method):
// rFF_Filtered := rFF_Filtered + (Ts/rFF_LagTime) × (rFF_Calc - rFF_Filtered)
//               + (rFF_LeadTime/rFF_LagTime) × (rFF_Calc - rFF_Calc_prev);

// Network 4: Sum FF + FB
// rCombinedOutput := rFB_Output + rFF_Filtered;
// rValvePosition  := LIMIT(0.0, rCombinedOutput, 100.0);
// rFF_Output := rFF_Filtered;     // diagnostic
```

---

## Step 1044: Dead Time Compensation

### Smith Predictor

```
Dead Time คืออะไร?
─────────────────────────────────────────────────
Dead time (θ) = เวลาที่ delay ระหว่าง output เปลี่ยนแล้ว
                input ของ process ยังไม่เห็นผล

ตัวอย่าง: ท่อส่งสารเคมียาว 100m
├── Pump เปิด/ปิด: immediate
├── สารเคมีถึงปลายท่อ: ต้องรอเวลา = L/v
└── ถ้าความเร็ว 0.5 m/s → Dead time = 100/0.5 = 200 seconds

ปัญหาของ PID กับ Dead Time:
├── PID ไม่รู้ว่า process กำลัง respond
├── PID คิดว่าต้อง increase output อีก
├── ผล: oscillation หรือ instability
└── Rule: ถ้า θ/τ > 0.3 ควรใช้ Smith Predictor
```

### Smith Predictor Architecture

```
SMITH PREDICTOR ARCHITECTURE:
──────────────────────────────────────────────────────────────

       SP
        │
        ▼
    ┌───────┐
    │  PID  │──────────────────────────────────────────→ Process Output (PV)
    └───────┘         │
        ▲             ▼
        │     ┌────────────────────────────────────┐
        │     │  SMITH PREDICTOR MODEL             │
        │     │  ┌─────────────────┐   ┌─────────┐│
        │     │  │ Process Model   │   │ Dead    ││
        │     │  │ G_model(s)      │──▶│ Time e  ││──▶ (+)
        │     │  └─────────────────┘   │ -θs     ││    │
        │     │                        └─────────┘│    │
        │     │  ┌─────────────────┐              │    │
        │     │  │ Process Model   │──────────────│──▶ (-)
        │     │  │ G_model(s)      │  (no delay)  │    │
        │     │  └─────────────────┘              │    ▼
        │     └────────────────────────────────────┘ Error
        │                                             signal
        └─────────────────────────────────────────────┘

Model Prediction:
├── G_model(s): model ของ process dynamics (without dead time)
├── e^(-θs): model ของ dead time
└── สร้าง "predicted PV" ที่ไม่มี delay → PID responds faster
```

---

## Step 1045: Model Predictive Control (MPC) Concepts

### MPC คืออะไร?

```
MPC (Model Predictive Control) - Overview:
──────────────────────────────────────────────

Principle:
1. มี Mathematical model ของ process
2. ทุก scan cycle: optimize ลำดับ control moves
   สำหรับ N steps ข้างหน้า (Prediction Horizon)
3. Apply เฉพาะ step แรก
4. Repeat ในรอบถัดไป (Receding Horizon)

Advantages over PID:
├── จัดการ MIMO (Multiple Input, Multiple Output) ได้
├── จัดการ constraints (valve limits, safety limits) ได้
├── Handles dead time ได้ดี
├── Feed-forward disturbance rejection ในตัว
└── Optimal control (minimize cost function)

Typical MPC Cost Function:
J = Σ [w_y × (y_pred - SP)² + w_u × (Δu)²]
    k=1..P

where:
├── y_pred: predicted output
├── SP: setpoint
├── Δu: control move
├── P: prediction horizon
├── w_y: output weight
└── w_u: control effort weight

Industrial MPC Software:
├── Siemens SIMIT APC
├── Rockwell Pavilion8
├── AspenTech DMCplus/GDOT
├── Honeywell Profit Controller
├── Yokogawa SMOC
└── Open source: do-mpc (Python), ACADO
```

---

## Step 1046: Self-Tuning PID

### Auto-Tuning Methods

```
PID Auto-Tuning Methods:
═══════════════════════════

1. RELAY FEEDBACK TEST (Åström-Hägglund)
───────────────────────────────────────────
Step 1: Apply relay (on/off with dead band) ใน closed loop
Step 2: วัด ultimate gain (Ku) และ ultimate period (Pu)
Step 3: คำนวณ PID gains จาก Ziegler-Nichols:
    Kp = 0.6 × Ku
    Ti = 0.5 × Pu
    Td = 0.125 × Pu

Advantages:
├── ไม่ต้องรู้ process model ล่วงหน้า
├── Test เสร็จใน 1-3 oscillation cycles
└── ปลอดภัยกว่า step test

2. STEP RESPONSE METHOD
───────────────────────────────────────────
Step 1: Apply step change ใน manual
Step 2: วัด process reaction curve
Step 3: Identify: Kp_process, τ (time constant), θ (dead time)
Step 4: คำนวณ PID gains

Cohen-Coon Formula:
Kp = (τ/θ × 1/Kp_proc) × [1.33 + θ/(4τ)]
Ti = θ × [2.25τ/(θ + 2.25τ)] × (1 + θ/(4τ))
Td = θ × 0.37τ/(τ + 0.37θ)

3. SIEMENS TIA PORTAL AUTO-TUNE
───────────────────────────────────────────
// PID_Compact มี Auto-tuning built-in
// TIA Portal → Online → PID tuning interface
// Pretuning: ทำก่อน (coarse tune)
// Fine tuning: optimize further
```

### Adaptive PID

```
// Adaptive PID - ปรับ gain ตาม operating condition
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_AdaptivePID
VAR_INPUT
    rSP:            REAL;
    rPV:            REAL;
    rOperatingLoad: REAL;       // 0-100% ของ load
END_VAR

VAR_OUTPUT
    rCV:            REAL;
END_VAR

VAR
    fbPID:          PID_Compact;
    rKp_adapted:    REAL;
    rTi_adapted:    REAL;
    rTd_adapted:    REAL;
END_VAR

// Network 1: Adapt gains based on operating point
// Gain Scheduling: กำหนด gain table ตาม load
//
// IF rOperatingLoad < 30.0 THEN
//     // Low load: process slower, need more gain
//     rKp_adapted := 4.0;
//     rTi_adapted := 90.0;
//     rTd_adapted := 10.0;
// ELSIF rOperatingLoad < 70.0 THEN
//     // Medium load: normal gains
//     rKp_adapted := 2.5;
//     rTi_adapted := 60.0;
//     rTd_adapted := 8.0;
// ELSE
//     // High load: process faster, reduce gain to prevent oscillation
//     rKp_adapted := 1.5;
//     rTi_adapted := 45.0;
//     rTd_adapted := 5.0;
// END_IF;

// Network 2: Apply adapted gains and run PID
// fbPID(
//     Setpoint := rSP,
//     Input    := rPV,
//     Gain     := rKp_adapted,
//     Ti       := rTi_adapted,
//     Td       := rTd_adapted,
//     Output   => rCV
// );
```

---

## Step 1047: Anti-Windup Strategies

### Integral Windup Problem

```
Integral Windup คืออะไร?
────────────────────────────────────────────────────────────────
เกิดเมื่อ output ถึง limit (saturation) แต่ integral ยังสะสมต่อ
ผล: เมื่อ output กลับจาก saturation → overshoot มาก

ตัวอย่าง: Temperature controller
SP = 200°C, PV = 20°C (startup)
Error = 180°C ← มาก!
Integral term สะสม: I ← ∞ (ถ้าไม่มี anti-windup)
เมื่อถึง SP: I ยังสูงมาก → CV ยังสูง → Overshoot!

Anti-Windup Methods:
```

### Anti-Windup Implementation

```
// Anti-Windup Methods ใน PLC:

// METHOD 1: CLAMPING (แบบง่าย)
// ─────────────────────────────────────────────────────────────
// ไม่ integrate ถ้า output อยู่ที่ limit

// Pseudo code:
IF (rCV >= CV_Max AND rError > 0) THEN
    // อย่า integrate เพิ่ม (output เต็มแล้วและ error ยังบวก)
    bIntegrate := FALSE;
ELSIF (rCV <= CV_Min AND rError < 0) THEN
    // อย่า integrate ลด (output ต่ำสุดแล้วและ error ยังลบ)
    bIntegrate := FALSE;
ELSE
    bIntegrate := TRUE;
END_IF;

IF bIntegrate THEN
    rIntegral := rIntegral + (rError * Ts / Ti);
END_IF;

// METHOD 2: BACK-CALCULATION (แบบดี)
// ─────────────────────────────────────────────────────────────
// หาค่า integral ที่ทำให้ unclamped output = clamped output
// 
// rCV_unclamped := Kp × (rError + rIntegral + Td × rDerivative)
// rCV_clamped   := LIMIT(CV_Min, rCV_unclamped, CV_Max)
// 
// Anti-windup term:
// rIntegral += (Kp × rError + rTrackingError / Tt) × Ts
// where rTrackingError = rCV_clamped - rCV_unclamped
//       Tt = tracking time constant (typically Ti)

// Siemens PID_Compact: Anti-windup built-in
// Parameters:
PID_COMPACT_ANTIWINDUP {
    UpperLimit: 100.0,    // CV upper limit
    LowerLimit: 0.0,      // CV lower limit
    // Anti-windup mode: Manual (default: automatic clamping)
}
```

---

## Step 1048: Batch PID

### Batch Process Control Challenges

```
Batch Process vs Continuous Process:
─────────────────────────────────────────────────────────────

Continuous Process:
├── PV ≈ SP ตลอดเวลา (steady state)
├── PID ทำงาน near operating point
└── Tuning: optimize for small disturbances

Batch Process (e.g., Reactor, Mixing tank):
├── PV เริ่มที่ 0 → ต้องการถึง SP
├── Large initial error → windup problem
├── Setpoint อาจเปลี่ยนตาม recipe step
└── ต้องการ aggressive tuning (reach SP quickly)
    แต่ไม่ overshoot (อาจทำให้ product เสีย)

Special Considerations:
├── Pre-act (on startup): เปิด valve เต็มก่อน PID เริ่ม
├── Rate limiting SP ramp: เพิ่ม SP ช้าๆ ไม่กระชาก
├── Mode switching: เปลี่ยน tuning สำหรับ heat-up vs near SP
└── Batch end: ปิด PID และ hold ที่ SP
```

### Batch PID Implementation

```
// Batch Temperature Control - Pharmaceutical Reactor
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_BatchTempControl
VAR_INPUT
    iEnable:        BOOL;
    rBatchSP:       REAL;           // Target batch temperature
    rTempPV:        REAL;           // Actual temperature
    bBatchRunning:  BOOL;           // Batch is active
    rRampRate:      REAL := 2.0;    // °C/min ramp rate
END_VAR

VAR_OUTPUT
    rHeaterOutput:  REAL;           // 0-100% heater output
    rCurrentSP:     REAL;           // Ramped SP (diagnostic)
    bAtTemperature: BOOL;           // Reached and stable at SP
    bRamping:       BOOL;           // Currently ramping
END_VAR

VAR
    fbPID:          PID_Compact;
    rRampedSP:      REAL;           // Ramped setpoint
    rLastScanSP:    REAL;           // SP from last scan
    rMaxRampStep:   REAL;           // Max change per scan (derived)
    rScanTime:      REAL := 0.1;    // 100ms scan time
    rAtTempTimer:   TON;            // Timer for stable check
END_VAR

// Network 1: คำนวณ Max ramp step per scan
// rMaxRampStep := rRampRate × rScanTime / 60.0;
// (2°C/min × 0.1s / 60s = 0.00333°C/scan)

// Network 2: Ramp the setpoint
// IF bBatchRunning THEN
//     IF rBatchSP > rRampedSP THEN
//         rRampedSP := MIN(rBatchSP, rRampedSP + rMaxRampStep);
//         bRamping := (rRampedSP < rBatchSP);
//     ELSIF rBatchSP < rRampedSP THEN
//         rRampedSP := MAX(rBatchSP, rRampedSP - rMaxRampStep);
//         bRamping := (rRampedSP > rBatchSP);
//     ELSE
//         bRamping := FALSE;
//     END_IF;
// ELSE
//     rRampedSP := rTempPV;   // Track current temp when not running
// END_IF;

// Network 3: PID with ramped SP
// fbPID(
//     Setpoint := rRampedSP,
//     Input    := rTempPV,
//     Mode     := (bBatchRunning ? 3 : 4),  // Auto/Manual
//     Output   => rHeaterOutput
// );

// Network 4: At Temperature Check (SP ±1°C for 30 seconds)
// rAtTempTimer(
//     IN  := ABS(rTempPV - rBatchSP) < 1.0,
//     PT  := T#30s,
//     Q   => bAtTemperature
// );

// Network 5: Output
// rCurrentSP := rRampedSP;
```

---

## Step 1049: PID Performance Metrics

### IAE, ISE, ITAE Metrics

```
PID Performance Metrics:
═════════════════════════════════════════════════════════════

IAE (Integral Absolute Error):
IAE = ∫₀^∞ |e(t)| dt

├── วัด total absolute deviation from SP
├── Good general purpose metric
└── ใช้บ่อยที่สุดสำหรับ compare tuning methods

ISE (Integral Squared Error):
ISE = ∫₀^∞ e²(t) dt

├── Penalizes large errors more (squared)
├── Tolerates small errors
└── Good when large errors are very undesirable

ITAE (Integral Time-weighted Absolute Error):
ITAE = ∫₀^∞ t × |e(t)| dt

├── Penalizes errors that persist long
├── Time weighting pushes system to settle faster
└── Produces more oscillatory initial response but settles quickly

ITSE (Integral Time-weighted Squared Error):
ITSE = ∫₀^∞ t × e²(t) dt

├── Combination of ISE and time weighting
└── Good compromise

Comparison:
┌──────────┬───────────────────────────────────────────────┐
│ Metric   │ Best for                                      │
├──────────┼───────────────────────────────────────────────┤
│ IAE      │ General performance, servo tracking           │
│ ISE      │ When large deviations are dangerous           │
│ ITAE     │ When settling speed is important              │
│ ITSE     │ Both large deviations and settling speed      │
└──────────┴───────────────────────────────────────────────┘
```

### Computing IAE/ISE in PLC

```
// PID Performance Monitoring ใน SCL
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_PIDPerformance
VAR_INPUT
    rSP:            REAL;           // Setpoint
    rPV:            REAL;           // Process Variable
    bReset:         BOOL;           // Reset metrics
    rScanTime:      REAL := 0.1;    // Scan time (seconds)
END_VAR

VAR_OUTPUT
    rIAE:           REAL;           // Integral Absolute Error
    rISE:           REAL;           // Integral Squared Error
    rITAE:          REAL;           // Integral Time Absolute Error
    rMaxError:      REAL;           // Maximum error seen
    rSettleTime:    REAL;           // Time to reach ±2% band
    bSettled:       BOOL;           // Currently in settle band
END_VAR

VAR
    rError:         REAL;
    rElapsedTime:   REAL;           // Total elapsed time
    bFirstSettled:  BOOL;           // Has settled at least once
END_VAR

// Network 1: Reset metrics
// IF bReset THEN
//     rIAE := 0.0;  rISE := 0.0;  rITAE := 0.0;
//     rMaxError := 0.0;  rSettleTime := 0.0;
//     rElapsedTime := 0.0;  bFirstSettled := FALSE;
// END_IF;

// Network 2: Calculate error
// rError := rSP - rPV;
// rElapsedTime := rElapsedTime + rScanTime;

// Network 3: Update metrics (trapezoidal integration)
// rIAE  += ABS(rError) × rScanTime;
// rISE  += (rError × rError) × rScanTime;
// rITAE += rElapsedTime × ABS(rError) × rScanTime;

// Network 4: Max error tracking
// rMaxError := MAX(rMaxError, ABS(rError));

// Network 5: Settle time (±2% of SP)
// bSettled := ABS(rError) <= ABS(rSP × 0.02);
// IF bSettled AND NOT bFirstSettled THEN
//     rSettleTime := rElapsedTime;
//     bFirstSettled := TRUE;
// END_IF;
```

---

## Step 1050: Process Modeling Basics

### First-Order Plus Dead Time (FOPDT) Model

```
FOPDT Model: Y(s)/U(s) = (Kp × e^(-θs)) / (τs + 1)

Parameters:
├── Kp: Process gain (ΔOutput / ΔInput, units depend on process)
├── τ:  Time constant (seconds/minutes)
└── θ:  Dead time (seconds/minutes)

Identifying FOPDT from Step Test:
──────────────────────────────────────────────────────────────

Step 1: From manual, apply step change (usually 5-10% of range)
        ΔU = 10% (e.g., from 40% to 50% valve)
        
Step 2: Record response curve until steady state
        ΔY = final PV change - initial PV

Step 3: Calculate Kp:
        Kp = ΔY/ΔU = (PV_final - PV_initial) / (10%)
        Example: Kp = (55°C - 40°C) / 10% = 1.5 °C/%

Step 4: Identify dead time (θ):
        θ = time from step applied until PV first starts moving

Step 5: Identify time constant (τ):
        τ = time from θ until PV reaches 63.2% of final change
        
        OR use 2-point method:
        t1 = time to reach 28.3% of ΔY → τ = 1.5 × (t1 - θ)
        t2 = time to reach 63.2% of ΔY → verify τ

Example:
Step applied at t=0, ΔU = 10%
PV starts moving at t=5s   → θ = 5s
PV at 63.2% of ΔY at t=35s → τ = 35 - 5 = 30s
ΔY = 15°C, ΔU = 10%         → Kp = 1.5 °C/%

FOPDT: G(s) = 1.5 × e^(-5s) / (30s + 1)
```

---

## Step 1051: Internal Model Control (IMC)

### IMC และการแปลงเป็น PID

```
IMC (Internal Model Control):
──────────────────────────────────────────────────────────────

IMC Structure:
SP ──(+)──→ [IMC Controller] ──→ Process ──→ PV
       │                              │
       │    ┌─────────────────────┐   │
       └────│   Process Model     │   │
            └─────────────────────┘   │
                      │               │
                      └──(-)──────────┘
                         (Model error)

IMC to PID conversion (for FOPDT):
Kp_PID = (τ + θ/2) / (Kp_process × (λ + θ/2))
Ti_PID = τ + θ/2
Td_PID = τ × θ / (2τ + θ)

where λ = tuning parameter (closed-loop time constant)
       λ = 0.2 × max(θ, τ) → aggressive
       λ = 0.5 × max(θ, τ) → moderate  
       λ = 1.0 × max(θ, τ) → conservative

ตัวอย่าง (process: Kp=1.5, τ=30s, θ=5s):
λ = 0.5 × 30 = 15s (moderate)
Kp_PID = (30 + 2.5) / (1.5 × (15 + 2.5)) = 32.5 / 26.25 = 1.24
Ti_PID = 30 + 2.5 = 32.5 seconds
Td_PID = 30 × 5 / (60 + 5) = 150 / 65 = 2.31 seconds
```

---

## Step 1052: Split Range Control

### Split Range สำหรับ Heating/Cooling

```
Split Range Control:
ใช้ 2 actuators สำหรับ 1 process variable
ตัวอย่าง: Reactor temperature จาก -20°C ถึง +150°C
          ใช้ทั้ง Heater และ Cooler

Split Range Design:
───────────────────────────────────────────────────────────────

Controller Output 0-100%:
├── 0-50%:   Cooler control (0%=full cooling, 50%=cooler off)
└── 50-100%: Heater control (50%=heater off, 100%=full heating)

Valve Characterization:
CV = 0%:   Cooler fully open (max cooling)
CV = 50%:  Both heater and cooler at zero
CV = 100%: Heater fully open (max heating)

Dead band at 50% (optional): ป้องกัน hunting
```

### Split Range Implementation

```
// Split Range Control ใน SCL
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_SplitRange
VAR_INPUT
    rControllerOutput:  REAL;       // PID output 0-100%
    rDeadBand:          REAL := 2.0;// Dead band at 50% (+/- %)
END_VAR

VAR_OUTPUT
    rCoolerOutput:      REAL;       // 0-100% cooler valve
    rHeaterOutput:      REAL;       // 0-100% heater valve
END_VAR

// Network 1: Cooler Output (0-50% → 100-0%)
// Inverse: เมื่อ PID = 0% → Cooler = 100% (full cool)
//          เมื่อ PID = 50% → Cooler = 0% (off)
//
// IF rControllerOutput < (50.0 - rDeadBand) THEN
//     rCoolerOutput := 100.0 - (rControllerOutput / (50.0 - rDeadBand)) × 100.0;
// ELSE
//     rCoolerOutput := 0.0;
// END_IF;

// Network 2: Heater Output (50-100% → 0-100%)
// IF rControllerOutput > (50.0 + rDeadBand) THEN
//     rHeaterOutput := ((rControllerOutput - (50.0 + rDeadBand)) / 
//                       (50.0 - rDeadBand)) × 100.0;
// ELSE
//     rHeaterOutput := 0.0;
// END_IF;

// Network 3: Clamp outputs
// rCoolerOutput := LIMIT(0.0, rCoolerOutput, 100.0);
// rHeaterOutput := LIMIT(0.0, rHeaterOutput, 100.0);
```

---

## Step 1053: PID for Flow Control

### Flow Control ด้วย Equal Percentage Valve

```
Valve Characteristics:
──────────────────────────────────────────────────────────────

Linear Valve: Flow ∝ Valve position
  CV=50% → Flow = 50%
  ตรงไปตรงมา แต่ gain เปลี่ยนตาม differential pressure

Equal Percentage (EQ%):
  Flow = Rangeability^(CV/100 - 1) × Full_flow
  CV=50% → Flow ≈ 10% ของ full flow (ขึ้นอยู่กับ Rangeability)
  
  ใช้กับ: most flow/pressure applications
  ข้อดี: ชดเชย pressure drop ที่เปลี่ยนตาม flow

Quick Opening:
  Flow ≈ √CV (ใกล้เคียง)
  เปิดนิดเดียว flow เยอะแล้ว
  ใช้กับ: on/off applications, emergency flows

Linearization สำหรับ PID:
เมื่อ valve เป็น Equal Percentage → PID gain เปลี่ยนตาม position
แก้โดย: ใช้ flow transmitter เป็น PV แทน valve position
         Flow PID → output = valve position
         Process ที่ PID เห็น: linear (flow ∝ SP)
```

---

## Step 1054: Advanced Control in Allen-Bradley

### Allen-Bradley PIDE Block

```
// Allen-Bradley ControlLogix - PIDE (Enhanced PID)
// ─────────────────────────────────────────────────────────────

// PIDE มีความสามารถมากกว่า PID:
// 1. Cascade tracking
// 2. Feed-forward input
// 3. Setpoint limits
// 4. Rate limiting
// 5. Derivative filter

// PIDE Configuration (Studio 5000):
PIDE_CONFIG {
    // Basic
    Kp:         2.5,
    Ki:         0.5,        // integral gain (not Ti)
    Kd:         0.2,
    
    // Cascade
    CVEU_Min:   0.0,
    CVEU_Max:   100.0,
    
    // Feed-forward
    FF_Enable:  TRUE,
    FF_Gain:    0.8,
    
    // Rate limiting
    SP_Rate:    TRUE,       // Enable SP rate limiting
    SP_Rate_Pos: 5.0,       // Max positive rate (EU/s)
    SP_Rate_Neg: 5.0,       // Max negative rate (EU/s)
    
    // Derivative filter
    Td_Filter:  TRUE,
    N:          5,          // Filter factor (2-20)
}

// Ladder Rung: PIDE Block
// ─────────────────────────────────────────────────────────────
//
//  ┌─────────────────────────────────────────┐
//  │  PIDE                                    │
//  │  Enhanced PID Control                    │
//  │                          Reactor_PID     │
//  │  SP     :    Setpoint     120.0  ← rSP  │
//  │  PV     :    Process Var  115.3  ← rPV  │
//  │  FF     :    Feed Fwd       0.0  ← rFF  │
//  │  CV     :    Control Var   45.2  → rCV  │
//  │  CVINI  :    CV Initial     0.0          │
//  │  CVEU   :    CV Eng Units  45.2  → rCV  │
//  └─────────────────────────────────────────┘
```

---

## Step 1055: PID for pH Control

### pH Control - Highly Nonlinear Process

```
pH Titration Curve:
──────────────────────────────────────────────────────────────

pH
14 |
   |                                    ╭───
   |                                   /
   |                               ___/
 7 |──────────────────────────────────
   |              ___
   |             /
   |         ╭──/
   |        /
 0 +──────────────────────────────────
   0%                              100%
           Reagent Added

ปัญหา: Gain ของ process เปลี่ยนมาก!
- Near pH 7: Gain สูงมาก (slope steep)
- Away from pH 7: Gain ต่ำ

วิธีแก้:
1. Non-linear controller gain (ลด gain เมื่อใกล้ pH 7)
2. Cascade control (flow control inner)
3. Titration curve compensation
4. MPC (best)
```

### pH Control Implementation

```
// Nonlinear pH Control ด้วย Gain Scheduling
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_pHControl
VAR_INPUT
    rSP_pH:         REAL := 7.0;    // pH setpoint
    rPV_pH:         REAL;           // pH measurement
    rFlow_Process:  REAL;           // Process flow (for ratio)
END_VAR

VAR_OUTPUT
    rReagentFlow:   REAL;           // Reagent flow output
    rGain_Applied:  REAL;           // Actual gain (diagnostic)
END_VAR

VAR
    fbPID:          PID_Compact;
    rAdaptedGain:   REAL;
    rError:         REAL;
END_VAR

// Network 1: Calculate error
// rError := rSP_pH - rPV_pH;

// Network 2: Gain scheduling based on distance from neutral
// (Gain ลดลงเมื่ออยู่ใกล้ pH 7 เพราะ process gain สูง)
// 
// IF ABS(rError) > 3.0 THEN
//     rAdaptedGain := 0.5;    // Far from SP, conservative
// ELSIF ABS(rError) > 1.0 THEN
//     rAdaptedGain := 0.3;    // Getting closer
// ELSIF ABS(rError) > 0.3 THEN
//     rAdaptedGain := 0.1;    // Near SP (process gain high here)
// ELSE
//     rAdaptedGain := 0.05;   // Very near SP, very conservative
// END_IF;

// Network 3: Apply gain and run PID
// fbPID(
//     Setpoint := rSP_pH,
//     Input    := rPV_pH,
//     Gain     := rAdaptedGain,
//     Ti       := 120.0,
//     Td       := 5.0,
//     Output   => rReagentFlow
// );

// Network 4: Feedforward from process flow
// rReagentFlow += rFlow_Process × 0.001;  // Nominal ratio
// rGain_Applied := rAdaptedGain;
```

---

## Step 1056: Distillation Column Control

### Distillation Control Strategies

```
Distillation Column Control:
═════════════════════════════════════════════════════════════

Basic Variables:
├── Controlled: Top composition (overhead), Bottom composition
├── Manipulated: Reflux ratio, Reboiler duty
└── Disturbances: Feed rate, Feed composition, Feed temperature

Simple PID Approach (Dual Composition):
Top temperature → [PID] → Reflux flow
Bottom temperature → [PID] → Steam (Reboiler)

Problems:
└── Interacting loops (RGA analysis needed)

Better: Material Balance Control
Top composition ─→ [PID] ─→ Distillate flow (D)
Bottom composition → [PID] → Bottoms flow (B)
Pressure ─────────→ [PID] → Condenser cooling
Level (Reboiler) → [PID] → Steam
Level (Accumulator)→[PID] → Reflux

RGA (Relative Gain Array) Analysis:
Helps determine: which input controls which output
λ = 1: Perfect pairing (no interaction)
λ > 1: Interaction, detuning needed
λ < 0: Avoid this pairing!
```

---

## Step 1057: PID Tuning Lab - Heat Exchanger

### Practical Tuning Exercise

```
Lab Setup: Shell and Tube Heat Exchanger
────────────────────────────────────────────────────────────────

Process:
├── Hot side: Steam (160°C, 5 bar)
├── Cold side: Process water (in: 20°C, target out: 80°C)
├── Control: Steam flow valve (0-100%)
├── Measurement: PT100 thermocouple on outlet

Step Test Procedure:
1. Set to Manual, CV = 50%
2. Wait for steady state (5-10 min)
3. Record initial PV (approximately 80°C)
4. Step CV from 50% to 60% (10% step)
5. Record PV response every 10 seconds
6. Wait until new steady state

Sample Data:
Time(s) | PV(°C) | ΔPV | %ΔPV
   0    |  79.5  |  0  |   0%
  10    |  79.5  |  0  |   0%  ← dead time
  20    |  79.6  |  0.1|   1%  ← first movement
  30    |  80.2  |  0.7|   7%
  50    |  81.5  |  2.0|  20%
  80    |  83.5  |  4.0|  40%
 110    |  85.2  |  5.7|  57%  ← 63.2% threshold at ~6.3°C
 120    |  85.8  |  6.3|  63%  ← τ ≈ 110s
 180    |  87.5  |  8.0|  80%
 300    |  89.3  |  9.8|  98%  ← near steady state
 400    |  89.5  |  10 | 100%  ← Final: +10°C

Identified Parameters:
├── θ = 15 seconds (dead time)
├── τ = 110 seconds (time constant from 15s to +63.2%×10 = 6.32°C)
├── ΔY = 10°C
├── ΔU = 10%
└── Kp = 10/10 = 1.0 °C/%

IMC Tuning (λ = 0.5 × max(θ,τ) = 55s):
Kp_PID = (τ + θ/2) / (Kp × (λ + θ/2))
        = (110 + 7.5) / (1.0 × (55 + 7.5))
        = 117.5 / 62.5 = 1.88

Ti_PID = τ + θ/2 = 110 + 7.5 = 117.5 seconds
Td_PID = τ × θ / (2τ + θ) = 110 × 15 / 235 = 7.0 seconds
```

---

## Step 1058: PID for Level Control

### Level Control Strategies

```
Level Control Challenges:
═════════════════════════════════════════════════════════════

Type 1 (Tight control needed): Storage tank before critical equipment
    ├── Needs tight level control (±5%)
    ├── Use PID with narrow band
    └── Inlet flow = manipulated variable

Type 2 (Buffer tank): Used to absorb flow disturbances  
    ├── Wide level band is OK (20-80%)
    ├── "Averaging level control" - let level vary, smooth flow
    ├── Use P-only (no integral) to avoid flow fighting
    └── Low Kp = smoother outlet flow

Type 3 (Surge vessel): Minimal level control
    ├── Level range very wide (5-95%)
    ├── Controller only acts at limits
    └── "Proportional with tight at limits"

Averaging Level Control:
────────────────────────────────────────────────────────────
Goal: Smooth outlet flow, let level vary

PID Tuning for Averaging:
├── Use P-only (Ti = very large or off)
├── Kp = 0.5 - 2.0 (low gain)
└── Result: Level drifts but flow is smooth
```

---

## Step 1059: Pressure Control

### Pressure Control in Vessels

```
Pressure Control Strategies:
═════════════════════════════════════════════════════════════

Gas Pressure Control (faster dynamics):
├── Faster than liquid/temperature
├── Often P+I is sufficient (no D needed)
└── Watch for: pressure shocks, fast disturbances

Liquid Pressure Control (pump system):
├── เร็วมาก (nearly instantaneous change)
├── Often P-only (I can cause oscillation)
└── Consider: pump curve, pipe friction

Compressed Air Header Control:
────────────────────────────────────────────────────────────

Multiple compressors + 1 header pressure:
├── Lead compressor: throttle control (PID)
├── Lag compressors: on/off based on pressure band
└── Load sharing: proportional to capacity

Pressure Relief Control (safety):
├── Normal: PID controls pressure
├── If pressure too high: pressure relief valve (hardware)
├── If PID fails: SIS takes over (SIL)
└── Never rely on PID alone for safety-critical pressure
```

---

## Step 1060: Lab Project - Advanced PID Commissioning

### Lab: Commissioning Advanced Control สำหรับ Evaporator

**โจทย์**: Double-effect evaporator สำหรับ concentration ของ fruit juice

**Process**:
- Effect 1: Feed juice → steam heating → 50% concentration, 80°C
- Effect 2: Vapor from Effect 1 → further concentration → 65%, 65°C
- Product: 65°Brix juice at 10 L/min

**Control Requirements**:
1. Cascade Temperature Control (each effect)
2. Level Control (averaging) สำหรับ each effect
3. Concentration Control (product quality)
4. Steam economy optimization

```
COMMISSIONING TASKS:
══════════════════════

Task 1: Perform Step Tests (30 min each loop)
├── Effect 1 temperature (steam valve)
├── Effect 2 temperature (steam valve)
├── Effect 1 level (feed pump speed)
└── Effect 2 level (draw-off valve)

Task 2: Identify FOPDT Models
Document in table:
┌───────────┬──────┬─────┬──────┬────────────────────────────┐
│ Loop      │  Kp  │  τ  │  θ   │ Notes                      │
├───────────┼──────┼─────┼──────┼────────────────────────────┤
│ Eff1 Temp │      │     │      │                            │
│ Eff2 Temp │      │     │      │                            │
│ Eff1 Lvl  │      │     │      │                            │
│ Eff2 Lvl  │      │     │      │                            │
└───────────┴──────┴─────┴──────┴────────────────────────────┘

Task 3: Calculate and Implement PID Tuning
├── Use IMC method for temperatures
└── Use averaging control for levels

Task 4: Test and Optimize
├── Setpoint step response test
├── Disturbance rejection test (change feed rate ±20%)
├── Calculate IAE for each loop
└── Adjust tuning if needed

Task 5: Documentation
├── Loop sheets สำหรับแต่ละ control loop
├── Tuning parameters table
├── Performance metrics (IAE, ISE)
└── Operating manual

EVALUATION:
├── Step test execution and data quality
├── Model identification accuracy
├── Tuning methodology
├── Final performance (IAE target: 50% reduction from default)
└── Documentation completeness

PASS CRITERIA: Performance within ±2% of SP at steady state
               No oscillation at closed loop steady state
```

---

## สรุป Part 53

| หัวข้อ | เมื่อใช้ |
|--------|---------|
| Cascade | Inner loop disturbance เร็ว, actuator nonlinear |
| Ratio | ควบคุมอัตราส่วน 2 streams |
| Feedforward | Measurable disturbance มีผลมาก |
| Smith Predictor | Dead time/τ > 0.3 |
| Anti-Windup | Actuator saturation เกิดบ่อย |
| Adaptive/Scheduling | Process ทำงานหลาย operating points |

---

## Navigation

- [← Part 52: Industrial Network Architecture](./part52_network_architecture.md)
- [→ Part 54: PLC to Cloud Integration](./part54_plc_to_cloud.md)
- [↑ สารบัญหลัก](../README.md)
