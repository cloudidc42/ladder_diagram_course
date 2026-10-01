# Part 23: PID Control (Steps 431-460)

## ภาพรวม
PID Control คือหัวใจของระบบควบคุมอัตโนมัติในอุตสาหกรรม ตั้งแต่การควบคุมอุณหภูมิ ความดัน ระดับ ไปจนถึงการควบคุมความเร็ว PID ย่อมาจาก **P**roportional - **I**ntegral - **D**erivative ซึ่งทั้งสามส่วนนี้ทำงานร่วมกันเพื่อให้ค่า Process Variable ติดตาม Setpoint ได้อย่างแม่นยำ

---

## Step 431: ทฤษฎี PID - Proportional (P)

```
┌──────────────────────────────────────────────────────────┐
│           Proportional Control Concept                   │
│                                                          │
│  Error = Setpoint - Process_Variable                    │
│  Output = Kp × Error                                    │
│                                                          │
│  ตัวอย่าง: Temperature Control                          │
│  Setpoint = 80°C, PV = 70°C                             │
│  Error = 80 - 70 = 10°C                                 │
│  Kp = 5                                                  │
│  Output = 5 × 10 = 50%                                  │
│                                                          │
│  ถ้า PV เพิ่มเป็น 78°C:                                 │
│  Error = 2°C                                             │
│  Output = 5 × 2 = 10%                                   │
│                                                          │
│  ปัญหา: Steady-State Error (Offset)                     │
│  เมื่อ PV ≈ SP แต่ไม่ถึง SP เพราะ Output ต้องมีค่า     │
│  เพื่อชดเชย Heat Loss                                   │
│  P-only ไม่สามารถขจัด Steady-State Error ได้            │
└──────────────────────────────────────────────────────────┘
```

**กราฟ P Control:**

```
SP ─── ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       │           /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
PV     │         /                      ← Offset (ไม่ถึง SP)
       │        /
       │       /
       │      /
       └──────────────────────────────────► time
              Response
```

---

## Step 432: ทฤษฎี PID - Integral (I)

```
┌──────────────────────────────────────────────────────────┐
│              Integral Control Concept                    │
│                                                          │
│  Integral ค่อยๆ เพิ่ม Output ตามสะสมของ Error ตลอดเวลา │
│                                                          │
│  I_Output = (Ki) × Σ(Error × dt)                        │
│  หรือ = (1/Ti) × ∫Error dt                              │
│                                                          │
│  Ti = Integral Time (Reset Time) in seconds             │
│  ค่า Ti น้อย = I Action แรง = ตอบสนองเร็ว              │
│  ค่า Ti มาก = I Action อ่อน = ตอบสนองช้า               │
│                                                          │
│  ข้อดี: ขจัด Steady-State Error                         │
│  ข้อเสีย: อาจทำให้ Overshoot หรือ Oscillate             │
│                                                          │
│  Integral Windup: เมื่อ Output ชน Limit (0% หรือ 100%) │
│  แต่ Error ยังมีอยู่ I จะสะสมต่อไปเรื่อยๆ              │
│  เมื่อ Error หมด I จะมีค่าสะสมมาก → Overshoot รุนแรง  │
│  แก้ด้วย Anti-Windup                                     │
└──────────────────────────────────────────────────────────┘
```

**กราฟ PI Control:**

```
        /‾‾‾\
SP ─── ─ ─ ─│─ ─ ─\─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       │    │      \‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
PV     │    Overshoot
       │   /
       │  /
       │ /
       └──────────────────────────────────► time
         PI เข้าถึง SP แต่อาจ Overshoot
```

---

## Step 433: ทฤษฎี PID - Derivative (D)

```
┌──────────────────────────────────────────────────────────┐
│              Derivative Control Concept                  │
│                                                          │
│  Derivative ดูอัตราการเปลี่ยนแปลงของ Error             │
│                                                          │
│  D_Output = Kd × d(Error)/dt                            │
│  หรือ = Td × d(Error)/dt                                │
│                                                          │
│  Td = Derivative Time in seconds                        │
│                                                          │
│  ถ้า Error กำลังลดลงเร็ว = D จะลด Output ลง            │
│  ป้องกัน Overshoot                                      │
│                                                          │
│  ข้อดี: ลด Overshoot, เพิ่มความเสถียร                  │
│  ข้อเสีย: ไวต่อ Noise มาก                              │
│           ถ้า Signal Noisy → D จะแกว่งรุนแรง            │
│                                                          │
│  Derivative on PV (ไม่ใช่ Error):                      │
│  D = -Td × d(PV)/dt                                     │
│  ป้องกัน Derivative Kick เมื่อ SP เปลี่ยนกะทันหัน      │
└──────────────────────────────────────────────────────────┘
```

**สูตร PID สมบูรณ์:**

```
Output(t) = Kp × [e(t) + (1/Ti)∫e(t)dt + Td × de(t)/dt]

โดย:
  e(t) = Error = SP - PV
  Kp   = Proportional Gain
  Ti   = Integral Time (seconds)
  Td   = Derivative Time (seconds)
```

---

## Step 434: Bumpless Transfer

```
┌──────────────────────────────────────────────────────────┐
│              Bumpless Transfer                           │
│                                                          │
│  ปัญหา: เมื่อสลับจาก Manual → Auto                     │
│  ถ้า Manual Output ≠ PID Output ที่คำนวณ               │
│  → Output จะกระโดดทันที (Bump)                          │
│  → เกิดการรบกวน Process                                │
│                                                          │
│  ─────────────────────────────────────────────────────  │
│                                                          │
│  Bumpless Transfer Solution:                            │
│  ก่อนสลับเป็น Auto:                                     │
│  1. ตั้ง Integral Term = Manual_Output - P_Term          │
│     (ให้ PID Output = Manual Output ทันที)              │
│  2. สลับเป็น Auto                                        │
│  3. PID เริ่มควบคุมอย่างนุ่มนวล ไม่กระชาก              │
│                                                          │
│  ─────────────────────────────────────────────────────  │
│                                                          │
│  Siemens CONT_C มี Bumpless Transfer อัตโนมัติ          │
│  เมื่อตั้ง I_ITL_ON = TRUE และกำหนด I_ITL_VAL          │
└──────────────────────────────────────────────────────────┘
```

---

## Step 435: Ziegler-Nichols Tuning Method

วิธีปรับ PID ที่นิยมที่สุด มี 2 แบบ:

### Method 1: Ultimate Gain Method

```
┌──────────────────────────────────────────────────────────┐
│         Ziegler-Nichols Ultimate Gain Method             │
│                                                          │
│  ขั้นตอน:                                               │
│  1. ปิด I และ D (Ti = ∞, Td = 0)                       │
│  2. ใช้เฉพาะ P                                         │
│  3. เพิ่ม Kp จนระบบ Oscillate แบบ Sustained            │
│     (ไม่เพิ่มและไม่ลด)                                  │
│  4. บันทึก:                                             │
│     Ku = Ultimate Gain (ค่า Kp ที่ทำให้ Oscillate)     │
│     Pu = Period of Oscillation (วินาที)                 │
│                                                          │
│  ────────────────────────────────────────────────────── │
│  คำนวณ PID Parameters:                                  │
│  ────────────────────────────────────────────────────── │
│  Control  Kp          Ti       Td                       │
│  P        0.50×Ku     -        -                        │
│  PI       0.45×Ku     Pu/1.2  -                        │
│  PID      0.60×Ku     Pu/2    Pu/8                     │
│  ────────────────────────────────────────────────────── │
│                                                          │
│  ข้อควรระวัง:                                           │
│  - วิธีนี้ทำให้ระบบ Oscillate ซึ่งอาจอันตราย          │
│  - ใช้เฉพาะกับ Process ที่ไม่อันตราย                   │
│  - ผลลัพธ์อาจ Aggressive (Overshoot มาก)               │
└──────────────────────────────────────────────────────────┘
```

### Method 2: Step Response Method

```
┌──────────────────────────────────────────────────────────┐
│         Step Response (Process Reaction) Method          │
│                                                          │
│  ขั้นตอน:                                               │
│  1. ทำงานใน Manual Mode ที่ Stable Point                │
│  2. เพิ่ม Output ทันที (Step) ประมาณ 10-20%            │
│  3. บันทึก Response ของ PV                             │
│                                                          │
│     100% ─┐ (Output)                                   │
│           │─────────────────────────                    │
│            80%                                           │
│                                                          │
│     PV ─ ─│─ ─ ─ ─ ─ ─ ─ /‾‾‾‾‾‾‾‾‾                 │
│            │             /                              │
│            │       L    /                               │
│            │       ├───/ T                              │
│            └───────────────────────────                 │
│                                                          │
│  วัดค่า:                                                │
│  L = Dead Time (เวลาที่ PV เริ่มขยับ หลัง Step)       │
│  T = Time Constant (เวลาที่ PV ขึ้น 63.2% ของ Total)  │
│  ΔPV = Change in PV                                    │
│  ΔOutput = Change in Output                            │
│                                                          │
│  Process Gain: Kprocess = ΔPV / ΔOutput                │
│                                                          │
│  Tuning (Cohen-Coon):                                   │
│  Kp = (1.35/Kprocess) × (T/L)^0.82                    │
│  Ti = T × [2.5 - 2(L/T)] / [1 - 0.39(L/T)]           │
│  Td = 0.37 × T × (L/T) / [1 - 0.81(L/T)]             │
└──────────────────────────────────────────────────────────┘
```

---

## Step 436: PID ใน Siemens (CONT_C Function Block)

```
// Siemens TIA Portal - CONT_C (Continuous PID Control)
// ใน FB เรียกใช้ทุก 100ms (OB35 หรือ Cyclic Interrupt)

// CONT_C Parameters:
// COM_RST (BOOL) - Complete Restart (Reset)
// MAN_ON (BOOL)  - Manual Mode (TRUE = Manual)
// PVPER_ON (BOOL) - PV Peripheral ON (TRUE = use I/O directly)
// P_SEL (BOOL)   - P Active
// I_SEL (BOOL)   - I Active
// D_SEL (BOOL)   - D Active
// CYCLE (TIME)   - Sample Time
// SP_INT (REAL)  - Setpoint (Internal)
// PV_IN (REAL)   - Process Variable Input
// PV_PER (WORD)  - Process Variable Peripheral (Raw AI)
// MAN (REAL)     - Manual Output Value
// GAIN (REAL)    - Kp
// TI (TIME)      - Integral Time
// TD (TIME)      - Derivative Time
// TM_LAG (TIME)  - Derivative Smoothing Time
// DEADB_W (REAL) - Dead Band Width
// LMN_HLM (REAL) - Output High Limit
// LMN_LLM (REAL) - Output Low Limit
// LMN (REAL)     - Manipulated Variable (Output)
// LMN_PER (WORD) - Manipulated Variable Peripheral (AO)

// ────────────────────────────────────────────────────────
// ตัวอย่าง: Temperature PID Control
// ────────────────────────────────────────────────────────

// DB_PID (Instance DB of CONT_C)

NETWORK 1: // เรียก PID ทุก 100ms (ใน OB35)
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │                                                    │
    │  CONT_C DB_PID                                    │
    │  COM_RST := FALSE                                 │
    │  MAN_ON  := Manual_Mode                           │
    │  PVPER_ON := FALSE                                │
    │  P_SEL   := TRUE                                  │
    │  I_SEL   := TRUE                                  │
    │  D_SEL   := FALSE   (ปิด D สำหรับ Temp Control)  │
    │  CYCLE   := T#100MS                              │
    │  SP_INT  := Temperature_Setpoint                  │
    │  PV_IN   := Temperature_Actual                   │
    │  MAN     := Manual_Output                         │
    │  GAIN    := 2.5                                   │
    │  TI      := T#120S                               │
    │  TD      := T#0S                                 │
    │  DEADB_W := 0.5                                  │
    │  LMN_HLM := 100.0                                │
    │  LMN_LLM := 0.0                                  │
    │  LMN     → Heater_Output                         │
    └──────────────────────────────────────────────────┘

NETWORK 2: // Scale Output ไปยัง AO
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  NORM_X: 0.0, Heater_Output, 100.0 → Norm         │
    │  SCALE_X: 0, Norm, 27648 → QW80                  │
    └──────────────────────────────────────────────────┘
```

---

## Step 437: PID ใน Allen-Bradley (PID Instruction)

```
// Allen-Bradley ControlLogix - PID Instruction
// เรียกใน Periodic Task ทุก 100ms

// PID Instruction Structure:
// ┌─────────────────────────────────────────────────────┐
// │  PID                                                │
// │  Control Block: PID_1 (PID structure tag)           │
// │  Process Variable: Temperature_PV (REAL)            │
// │  Tieback: Manual_Output (REAL)                      │
// │  Output: Valve_Output (REAL)                        │
// └─────────────────────────────────────────────────────┘

// PID Tag Structure (CONTROL):
// PID_1.SP = Setpoint
// PID_1.KP = Proportional Gain
// PID_1.KI = Integral Gain (repeats/min)
// PID_1.KD = Derivative Gain (minutes)
// PID_1.MAXS = Maximum Scale for PV (100%)
// PID_1.MINS = Minimum Scale for PV (0%)
// PID_1.DB = Deadband
// PID_1.SO = Set Output % (for manual)
// PID_1.AM = Auto Mode (True/False)
// PID_1.OUT = Output (0-100%)

// หน่วย KI ใน AB = Repeats/Minute
// Ti (seconds) = 60/KI
// ถ้า Ti = 120s → KI = 60/120 = 0.5 repeats/min

RUNG 1: // Initialize PID on First Scan
    ┌──[OSR First_Scan]─────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └─[MOV 80.0  → PID_1.SP]   // Setpoint
         [MOV 2.5   → PID_1.KP]   // Kp
         [MOV 0.5   → PID_1.KI]   // Ki = 0.5 rep/min (Ti=120s)
         [MOV 0.0   → PID_1.KD]   // Kd = 0
         [MOV 100.0 → PID_1.MAXS] // 100°C scale max
         [MOV 0.0   → PID_1.MINS] // 0°C scale min
         [MOV 0.5   → PID_1.DB]   // 0.5°C deadband
         [MOV 1     → PID_1.AM]   // Auto Mode

RUNG 2: // Run PID Every Scan
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │                                                    │
    │  PID                                              │
    │  Control: PID_1                                    │
    │  PV:      Temperature_PV                          │
    │  Tieback: Manual_Output                           │
    │  Output:  Valve_Output                            │
    └──────────────────────────────────────────────────┘

RUNG 3: // Scale Output to AO
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  CPT: Valve_AO = TRUNC(Valve_Output * 327.67)    │
    └──────────────────────────────────────────────────┘
```

---

## Step 438: PID ใน Mitsubishi (PID Instruction)

```
// Mitsubishi MELSEC - PID Instruction (FX5U/Q Series)
// PID Instruction: S1=Control Data, S2=PV, D=MV

// Control Data (D registers):
// D0  = Sampling Time (ms)
// D1  = Action Type (Bit flags)
// D2  = Setpoint
// D3  = Kp (Proportional Gain × 1 = 1% unit)
// D4  = Ti (Integral Time, ms)
// D5  = Td (Derivative Time, ms)
// D6  = PV Upper Limit (for alarm)
// D7  = PV Lower Limit (for alarm)
// D8  = Output Upper Limit
// D9  = Output Lower Limit

// Control Data D1 (Action Type):
// Bit 0: 0=Forward(Heat), 1=Reverse(Cool)
// Bit 1: 0=PID, 1=PI
// Bit 2: 0=Normal, 1=Derivative on PV only
// Bit 3-15: Reserved

// ────────────────────────────────────────────────────
// ตัวอย่าง: ควบคุมอุณหภูมิ Oven
// PV = D200 (Temperature from AI)
// SP = 800 (80.0°C × 10)
// Output = D210 (Valve command, 0-1000 = 0-100%)
// ────────────────────────────────────────────────────

// Initialize PID Parameters
LD      M8002           // First Scan
MOV     K1000  D0       // Sampling Time = 1000ms
MOV     K0     D1       // Forward, PID, Normal
MOV     K800   D2       // Setpoint = 80.0°C (×10)
MOV     K25    D3       // Kp = 25 (= 2.5 × 10)
MOV     K12000 D4       // Ti = 120,000ms = 120s
MOV     K0     D5       // Td = 0ms
MOV     K1200  D6       // High Limit = 120.0°C
MOV     K0     D7       // Low Limit = 0.0°C
MOV     K1000  D8       // Output Max = 1000 (100%)
MOV     K0     D9       // Output Min = 0

// PID Execution (ทุก Scan)
LD      M8000           // Always ON
PID     D0  D200  D210  // PID Control D, PV, MV
```

---

## Step 439: ตัวอย่าง Temperature Control System

```
┌──────────────────────────────────────────────────────────┐
│         Industrial Oven Temperature Control              │
│                                                          │
│  System:                                                │
│  - Electric Oven 0-500°C                                │
│  - TC Type K → Siemens SM1231 TC Module                 │
│  - SCR Power Controller ← 4-20mA from AO Module         │
│  - Setpoint from HMI                                    │
│  - Alarm at ±10°C from Setpoint                         │
│                                                          │
│  PID Parameters (After Tuning):                         │
│  Kp = 1.8                                               │
│  Ti = 90s (TI = T#90S)                                 │
│  Td = 15s (TD = T#15S)                                 │
│  Deadband = 0.5°C                                       │
│                                                          │
│  Expected Response:                                     │
│  - Rise Time: ~5 minutes (Cold Start)                   │
│  - Overshoot: < 5°C                                     │
│  - Settling Time: ~8 minutes                            │
│  - Steady-state Error: < 0.5°C                          │
└──────────────────────────────────────────────────────────┘
```

---

## Step 440: Flow Control Example

```
// Flow Control: ควบคุมอัตราการไหลของของเหลว
// ต้องการ Response เร็วกว่า Temp Control

// System:
// Flow Meter: 0-100 L/min → 4-20mA
// Control Valve: 4-20mA → 0-100% Open

// PID Parameters สำหรับ Flow:
// Kp = 0.8   (ค่าต่ำกว่า เพราะ Process ตอบสนองเร็ว)
// Ti = 15s   (Integration เร็ว)
// Td = 2s    (Derivative ช่วย Damping)

// Note: Flow Process มักมี Dead Time น้อย
// แต่ Valve มี Non-linearity
// อาจต้องใช้ Valve Characterization

NETWORK 1: // Flow PID (Siemens CONT_C)
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  CONT_C DB_FlowPID                               │
    │  MAN_ON  := Manual_Mode                           │
    │  P_SEL   := TRUE                                  │
    │  I_SEL   := TRUE                                  │
    │  D_SEL   := TRUE                                  │
    │  CYCLE   := T#100MS                              │
    │  SP_INT  := Flow_Setpoint                         │
    │  PV_IN   := Flow_Actual                           │
    │  GAIN    := 0.8                                   │
    │  TI      := T#15S                                │
    │  TD      := T#2S                                 │
    │  TM_LAG  := T#5S    (D Smoothing)                │
    │  LMN_HLM := 100.0                                │
    │  LMN_LLM := 0.0                                  │
    │  LMN → Valve_Position                            │
    └──────────────────────────────────────────────────┘
```

---

## Step 441: Cascade PID Control

```
┌──────────────────────────────────────────────────────────┐
│              Cascade PID Control                         │
│                                                          │
│  ใช้เมื่อ: มี 2 Variables ที่เกี่ยวข้องกัน              │
│  Outer Loop (Primary) ควบคุม Process Variable หลัก      │
│  Inner Loop (Secondary) ควบคุม Actuator                 │
│                                                          │
│  ตัวอย่าง: ระบบ Heat Exchanger                          │
│                                                          │
│  OUTER PID:                                             │
│    SP: Product Temp 80°C                               │
│    PV: Product Temperature                              │
│    Output → Inner PID Setpoint (Hot Water Flow SP)      │
│                                                          │
│  INNER PID:                                             │
│    SP: Hot Water Flow SP (from Outer PID)               │
│    PV: Hot Water Flow (Flow Meter)                      │
│    Output → Flow Control Valve Position                 │
│                                                          │
│  ├── Outer Loop ──────────────────────────────────────  │
│  │  SP=80°C                                             │
│  │  PV=Product Temp                              PV    │
│  │                                          ┌────────┐ │
│  │  Outer PID ────SP(Flow)──► Inner PID ───►│ Valve  │ │
│  │                            PV=Flow Meter │        │ │
│  │                                          └────────┘ │
│  │                                              │       │
│  │  Product ◄──────────────────────────────────┘       │
│  │  Temperature                                         │
│  └──────────────────────────────────────────────────── │
│                                                          │
│  ข้อดี: Inner Loop ตอบสนองเร็ว กันการรบกวน             │
│  Outer Loop สามารถ Tune ได้ช้ากว่าปกติ                  │
└──────────────────────────────────────────────────────────┘
```

### โค้ด Cascade PID (Siemens):

```
// Outer Loop: Product Temperature
// Inner Loop: Hot Water Flow

// DB1: Outer_PID (CONT_C)
// DB2: Inner_PID (CONT_C)

NETWORK 1: // Outer PID (ทำงานช้ากว่า - ทุก 500ms)
    ┌──[Outer_Clock]────────────────────────────────────┐
    │  CONT_C DB1                                       │
    │  SP_INT  := Product_Temp_SP (80.0°C)             │
    │  PV_IN   := Product_Temp_Actual                   │
    │  GAIN    := 2.0                                   │
    │  TI      := T#300S  (ช้า เพราะ Thermal Process)  │
    │  TD      := T#0S                                 │
    │  LMN_HLM := 50.0    (Flow Max 50 L/min)          │
    │  LMN_LLM := 5.0     (Flow Min 5 L/min)           │
    │  LMN     → Hot_Water_Flow_SP                     │
    └──────────────────────────────────────────────────┘

NETWORK 2: // Inner PID (ทำงานเร็ว - ทุก 100ms)
    ┌──[Inner_Clock]────────────────────────────────────┐
    │  CONT_C DB2                                       │
    │  SP_INT  := Hot_Water_Flow_SP (จาก Outer Output) │
    │  PV_IN   := Hot_Water_Flow_Actual                 │
    │  GAIN    := 0.5                                   │
    │  TI      := T#10S   (เร็ว เพราะ Flow ตอบสนองเร็ว)│
    │  TD      := T#1S                                 │
    │  LMN_HLM := 100.0                                │
    │  LMN_LLM := 0.0                                  │
    │  LMN     → Valve_Position                        │
    └──────────────────────────────────────────────────┘
```

---

## Step 442: Feedforward Control

```
┌──────────────────────────────────────────────────────────┐
│              Feedforward Control                         │
│                                                          │
│  ปัญหาของ Feedback (PID):                               │
│  รอให้ Disturbance เกิดก่อน แล้วค่อย Correct           │
│                                                          │
│  Feedforward:                                           │
│  วัด Disturbance โดยตรง แล้วชดเชยก่อนที่จะกระทบ PV    │
│                                                          │
│  ตัวอย่าง: Heat Exchanger                              │
│  ถ้า Inlet Flow เพิ่ม → ต้องการ Hot Water เพิ่ม        │
│  ไม่ต้องรอให้ Outlet Temp ลดก่อน                       │
│                                                          │
│  FF_Output = FF_Gain × Disturbance_Measurement          │
│  Total_Output = PID_Output + FF_Output                  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  SP ──► PID ──►(+)──► Output ──► Process        │   │
│  │                  ▲                               │   │
│  │  Disturbance ──► FF ──────────────────────────── │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Siemens CONT_C + Feedforward:

```
// CONT_C ไม่มี FF Built-in ต้องเพิ่มเอง

NETWORK 1: // PID Output
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  CONT_C DB_PID                                    │
    │  SP_INT := Product_Temp_SP                       │
    │  PV_IN  := Product_Temp                          │
    │  LMN    → PID_Output                             │
    └──────────────────────────────────────────────────┘

NETWORK 2: // Feedforward Calculation
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  CPT: FF_Output = Inlet_Flow * FF_Gain           │
    │  (FF_Gain = calibrated experimentally)           │
    └──────────────────────────────────────────────────┘

NETWORK 3: // Combine PID + FF
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  ADD: PID_Output + FF_Output → Combined_Output   │
    └──────────────────────────────────────────────────┘

NETWORK 4: // Clamp Combined Output
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  LIMIT: 0.0, Combined_Output, 100.0 → Valve_Pos │
    └──────────────────────────────────────────────────┘
```

---

## Step 443: Auto-Tuning Concepts

```
┌──────────────────────────────────────────────────────────┐
│              Auto-Tuning Overview                        │
│                                                          │
│  วิธีที่ 1: Relay Feedback Auto-Tuning (Åström-Hägglund)│
│  ────────────────────────────────────────────────────── │
│  1. สลับ Output เป็น Relay Mode (ON/OFF ±d)             │
│  2. Process จะ Oscillate                                │
│  3. วัด Period (Pu) และ Amplitude ของ Oscillation       │
│  4. คำนวณ Ku จาก Relay Amplitude และ PV Amplitude       │
│     Ku = 4d/(π × a)  โดย d = Relay Amplitude           │
│                            a = PV Oscillation Amplitude │
│  5. ใช้ Ziegler-Nichols คำนวณ Kp, Ti, Td               │
│                                                          │
│  ────────────────────────────────────────────────────── │
│                                                          │
│  Siemens TIA Portal - PID Compact/Advanced:             │
│  มี Built-in Pretuning และ Fine Tuning                  │
│  1. Pretuning: Step Response → ประเมิน Initial Params   │
│  2. Fine Tuning: Relay Method → ปรับ Fine               │
│                                                          │
│  Allen-Bradley - Auto Tune:                             │
│  PID Instruction มี Auto Tune Mode                      │
│  ตั้ง AT_MODE = 1 → เริ่ม Auto Tune                    │
│  AT_DONE = 1 → Auto Tune เสร็จ                         │
└──────────────────────────────────────────────────────────┘
```

---

## Step 444: PID Tuning Guidelines

```
┌──────────────────────────────────────────────────────────┐
│           Practical PID Tuning Guidelines                │
│                                                          │
│  ขั้นตอน Manual Tuning:                                 │
│                                                          │
│  1. เริ่มด้วย P เท่านั้น (I=0, D=0)                    │
│     - เพิ่ม Kp จนกว่า Response เร็วพอ                   │
│     - ถ้า Oscillate ลด Kp ลง 50%                       │
│                                                          │
│  2. เพิ่ม I เพื่อขจัด Steady-State Error               │
│     - เริ่มด้วย Ti = 10×Process_Time_Constant           │
│     - ลด Ti ลงจนกว่า Offset หาย แต่ยังไม่ Oscillate   │
│                                                          │
│  3. เพิ่ม D เพื่อลด Overshoot (ถ้าจำเป็น)              │
│     - เริ่มด้วย Td = Ti/4 (rule of thumb)              │
│     - ดู Noise ก่อนเพิ่ม D                             │
│                                                          │
│  สัญญาณบอกว่า Tuning ผิด:                              │
│  ─────────────────────────────────────────────────────  │
│  Kp สูงเกิน: Oscillation, Noisy Output                 │
│  Kp ต่ำเกิน: ตอบสนองช้า, Sluggish                      │
│  Ti สูงเกิน: Steady-State Error ยังมีอยู่               │
│  Ti ต่ำเกิน: Oscillation, Integral Windup              │
│  Td สูงเกิน: Output กระชาก, Sensitive to Noise         │
│  Td ต่ำเกิน: Overshoot ยังมีอยู่                        │
└──────────────────────────────────────────────────────────┘
```

---

## Step 445: Anti-Windup Implementation

```
// Anti-Windup ป้องกัน Integral สะสมเกินเมื่อ Output ชน Limit

// วิธี Back-Calculation Anti-Windup:
// ถ้า Output ถึง Limit:
//   I_Term = Output_Limit - P_Term - D_Term
// ป้องกันไม่ให้ I สะสมต่อไปเมื่อ Saturated

// Siemens CONT_C:
// - มี Anti-Windup อัตโนมัติเมื่อตั้ง LMN_HLM/LMN_LLM

// สำหรับ Custom PID (Allen-Bradley CPT):

RUNG 1: // PID Calculation
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  CPT:                                             │
    │  Error := SP - PV                                │
    │  P_Term := Kp * Error                            │
    │  I_Accum := I_Accum + (Kp/Ti) * Error * dt      │
    │  D_Term := Kp * Td * (PV_Last - PV) / dt        │
    │  PV_Last := PV                                   │
    │  Raw_Output := P_Term + I_Accum + D_Term         │
    └──────────────────────────────────────────────────┘

RUNG 2: // Anti-Windup: Clamp I_Accum
    ┌──[GRT Raw_Output, 100.0]──────────────────────────┐
    │  Output would be > 100%                           │
    └──┬─────────────────────────────────────────────────┘
       └──[CPT I_Accum = 100.0 - P_Term - D_Term]───────┤
          (Recalculate I to keep output exactly at 100%)

    ┌──[LES Raw_Output, 0.0]─────────────────────────────┐
    │  Output would be < 0%                              │
    └──┬──────────────────────────────────────────────────┘
       └──[CPT I_Accum = 0.0 - P_Term - D_Term]──────────┤

RUNG 3: // Clamp Final Output
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  LIMIT: 0.0, Raw_Output, 100.0 → Output          │
    └──────────────────────────────────────────────────┘
```

---

## Step 446: PID สำหรับระบบต่างๆ

```
┌──────────────────────────────────────────────────────────┐
│         Typical PID Settings by Process Type            │
│                                                          │
│  TEMPERATURE CONTROL:                                   │
│  - Process: Slow (minutes to hours)                     │
│  - Kp: 1-10                                             │
│  - Ti: 60-600 seconds                                   │
│  - Td: 10-100 seconds                                   │
│  - D often used (thermal lag)                          │
│                                                          │
│  PRESSURE CONTROL:                                      │
│  - Process: Medium (seconds)                            │
│  - Kp: 1-5                                              │
│  - Ti: 5-60 seconds                                     │
│  - Td: 0-5 seconds                                      │
│  - D often omitted                                      │
│                                                          │
│  FLOW CONTROL:                                          │
│  - Process: Fast (milliseconds to seconds)              │
│  - Kp: 0.3-2                                            │
│  - Ti: 1-20 seconds                                     │
│  - Td: 0-2 seconds                                      │
│  - PI usually sufficient                                │
│                                                          │
│  LEVEL CONTROL:                                         │
│  - Process: Slow to Medium (depends on tank size)       │
│  - Kp: 1-10                                             │
│  - Ti: 30-300 seconds                                   │
│  - Td: Rarely used                                      │
│  - P-only often acceptable                              │
└──────────────────────────────────────────────────────────┘
```

---

## Step 447: PID Mode Control (Auto/Manual/Cascade)

```
// Mode Management สำหรับ PID

// Modes:
// 0 = Manual     - Operator ควบคุม Output โดยตรง
// 1 = Auto       - PID ควบคุม Output
// 2 = Cascade    - SP มาจาก Outer PID Loop
// 3 = Remote     - SP มาจาก Remote System (SCADA)

RUNG 1: // Manual Mode
    ┌──[PID_Mode EQU 0]──────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[MOV Manual_Output → Valve_Position]────────────┤

RUNG 2: // Auto Mode
    ┌──[PID_Mode EQU 1]──────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[PID SP=Local_SP, PV=PV_Actual, OUT→Valve_Pos]──┤

RUNG 3: // Cascade Mode  
    ┌──[PID_Mode EQU 2]──────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[PID SP=Cascade_SP_from_Master, PV=PV, OUT→Valve]┤

RUNG 4: // Mode Switch Bumpless Transfer
    ┌──[PID_Mode CHANGED]────────────────────────────────┐
    │  (Mode เพิ่งเปลี่ยน = One Shot)                    │
    └──┬─────────────────────────────────────────────────┘
       └──[Initialize I_Term for Bumpless]────────────────┤
```

---

## Step 448: Troubleshooting PID

```
┌──────────────────────────────────────────────────────────┐
│           PID Troubleshooting Guide                      │
│                                                          │
│  อาการ: PV Oscillate ไม่หยุด                           │
│  ─────────────────────────────────────────────────      │
│  1. Kp สูงเกิน → ลด Kp                                 │
│  2. Ti ต่ำเกิน → เพิ่ม Ti                               │
│  3. D มากเกิน + Noise → ลด Td หรือเพิ่ม Filter         │
│  4. Dead Time มากกว่าที่คิด → Detune (ลด Kp, เพิ่ม Ti) │
│                                                          │
│  อาการ: PV ไม่ถึง SP (Steady-State Offset)             │
│  ─────────────────────────────────────────────────      │
│  1. I ถูกปิด → เปิด I                                   │
│  2. Ti สูงเกิน → ลด Ti                                  │
│  3. Output ชน Limit → เพิ่ม Capacity ของ Actuator       │
│                                                          │
│  อาการ: PV ตอบสนองช้ามาก                               │
│  ─────────────────────────────────────────────────      │
│  1. Kp ต่ำเกิน → เพิ่ม Kp                              │
│  2. Ti สูงเกิน → ลด Ti                                  │
│  3. Output Scale ผิด → ตรวจ Scaling                     │
│  4. Actuator ตัน/เสียหาย → ตรวจ Physical               │
│                                                          │
│  อาการ: Output กระชากเมื่อ SP เปลี่ยน                  │
│  ─────────────────────────────────────────────────      │
│  1. D ทำงานบน Error แทน PV → เปลี่ยนเป็น D-on-PV       │
│  2. SP Rate Limiter → ใช้ Ramp Generator ใน SP          │
└──────────────────────────────────────────────────────────┘
```

---

## Step 449: SP Ramp Generator

```
// SP Ramp: ไม่เปลี่ยน SP ทันที แต่ Ramp ขึ้น/ลงอย่างช้าๆ
// ป้องกัน Output Saturation และ Process Shock

// Tags:
// SP_Target (REAL) = Desired SP from Operator
// SP_Actual (REAL) = SP ที่ส่งไปยัง PID (Ramped)
// SP_Rate (REAL) = Rate of Change per second (°C/s)

RUNG 1: // Ramp UP
    ┌──[GRT SP_Target, SP_Actual]────────────────────────┐
    │  Target > Actual (ต้องเพิ่ม SP)                   │
    └──┬─────────────────────────────────────────────────┘
       └──[CPT SP_Actual = SP_Actual + SP_Rate * Scan_Time]

RUNG 2: // Ramp DOWN
    ┌──[LES SP_Target, SP_Actual]────────────────────────┐
    │  Target < Actual (ต้องลด SP)                      │
    └──┬─────────────────────────────────────────────────┘
       └──[CPT SP_Actual = SP_Actual - SP_Rate * Scan_Time]

RUNG 3: // Snap to Target เมื่อใกล้ถึง
    ┌──[LES ABS(SP_Target - SP_Actual), 0.1]────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[MOV SP_Target → SP_Actual]────────────────────┤
```

---

## Step 450: ตัวอย่างสมบูรณ์ - Temperature Profile Control

```
// Batch Process: ควบคุมอุณหภูมิตาม Profile
// Profile:
// Phase 1: Ramp from 25°C to 150°C at 2°C/min
// Phase 2: Hold at 150°C for 30 minutes
// Phase 3: Ramp from 150°C to 250°C at 1°C/min
// Phase 4: Hold at 250°C for 60 minutes
// Phase 5: Cool Down (natural) to 100°C

// ────────────────────────────────────────────────────────
// Data:
// Phase_Number (INT) = 0-5
// Profile_SP (REAL)  = Current profile setpoint
// Actual_Temp (REAL) = Process temperature
// Phase_Timer (TON)  = Timer for hold phases
// ────────────────────────────────────────────────────────

RUNG 1: // Phase 1: Ramp to 150°C
    ┌──[Phase_Number EQU 1]─────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[CPT Profile_SP = Profile_SP + (2.0/60.0)*0.1]─┤
          (2°C/min → 2/60 °C/s → ×0.1s scan = 0.00333°C/scan)

RUNG 2: // Phase 1 → 2: SP ถึง 150°C
    ┌──[Phase_Number EQU 1]──[GEQ Profile_SP, 150.0]────┐
    └──┬─────────────────────────────────────────────────┘
       └──[MOV 2 → Phase_Number]───────────────────────── │
          [MOV 150.0 → Profile_SP]

RUNG 3: // Phase 2: Hold at 150°C, 30 min
    ┌──[Phase_Number EQU 2]────────────────────────(TON)─┐
    │                                      Phase_Timer   │
    │                                         PT: 30min  │
    └─────────────────────────────────────────────────────┘

RUNG 4: // Phase 2 → 3: Hold Done
    ┌──[Phase_Number EQU 2]──[Phase_Timer.Q]─────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──[MOV 3 → Phase_Number]───────────────────────── │
          [RST Phase_Timer]

// ... Phase 3, 4, 5 similar pattern
```

---

## Step 451-460: Lab Projects

### Lab 1: Simple Temperature PID

```
┌──────────────────────────────────────────────────────────┐
│  Lab 1: Temperature PID Control                         │
│                                                          │
│  Hardware:                                              │
│  - Siemens S7-1200 + SM1231 AI                         │
│  - TC Type K                                            │
│  - SSR (Solid State Relay) or SCR Controller           │
│                                                          │
│  Tasks:                                                 │
│  1. Configure TC Module                                 │
│  2. Scale TC reading                                    │
│  3. Implement CONT_C PID                               │
│  4. Tune using Step Response Method                     │
│  5. Add Auto/Manual mode                               │
│  6. Add Alarm (HH/H/L/LL)                              │
│  7. Add SP Ramp                                         │
│  8. Create HMI Faceplate                               │
└──────────────────────────────────────────────────────────┘
```

### Lab 2: Cascade Control

```
┌──────────────────────────────────────────────────────────┐
│  Lab 2: Cascade PID - Heat Exchanger Simulation          │
│                                                          │
│  เรียนรู้:                                               │
│  - Master/Slave PID Configuration                       │
│  - Inner Loop Tuning (ก่อน Outer Loop)                 │
│  - Bumpless Transfer ระหว่าง Auto/Cascade               │
│  - Safety: ถ้า Inner Loop Fault → Outer ไม่ทำงาน        │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 23

ใน Part นี้ได้เรียนรู้:

1. **PID Theory** - P, I, D Components และ Effects
2. **Bumpless Transfer** - Auto/Manual Switching
3. **Ziegler-Nichols** - Ultimate Gain และ Step Response Method
4. **Siemens CONT_C** - Parameters และ Configuration
5. **Allen-Bradley PID** - PID Instruction Structure
6. **Mitsubishi PID** - PID Instruction
7. **Temperature Control** - Practical Example
8. **Flow Control** - Fast Process Tuning
9. **Cascade PID** - Master/Slave Configuration
10. **Feedforward** - Disturbance Rejection
11. **Anti-Windup** - Integral Windup Prevention
12. **Troubleshooting** - Common PID Problems

---

## Navigation
- [← Part 22: Analog I/O และ Scaling](part22_analog_io.md)
- [→ Part 24: Communication Protocols Overview](part24_communication.md)
- [กลับหน้าหลัก](../README.md)
