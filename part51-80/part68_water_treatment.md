# Part 68: Water Treatment Automation (Steps 1341-1360)

## การเขียนโปรแกรม PLC สำหรับระบบบำบัดน้ำ

---

### 🔗 Navigation
- [← Part 67: Printing & Packaging](part67_printing_packaging.md)
- [→ Part 69: HVAC Automation](part69_hvac_automation.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: การบำบัดน้ำประปาและน้ำเสีย

การบำบัดน้ำ (Water Treatment) เป็นหนึ่งในงานที่ PLC Engineer ต้องเจอบ่อยที่สุดในโครงการสาธารณูปโภค โรงงานอุตสาหกรรม และโรงพยาบาล โดยระบบ PLC ต้องควบคุมกระบวนการเคมีและกายภาพที่ซับซ้อน พร้อมทั้งส่งข้อมูลไปยัง SCADA เพื่อการ Monitor และ Report

### ขั้นตอนการบำบัดน้ำดิบ (Raw Water Treatment)

```
Raw Water Intake
       ↓
  Screening    → กรองขยะชิ้นใหญ่
       ↓
  Coagulation  → เติมสาร Alum (สารส้ม) ทำให้ตะกอนรวมกัน
       ↓
  Flocculation → กวนช้าๆ ให้ตะกอนรวมเป็นก้อน (Floc)
       ↓
  Sedimentation → ให้ Floc ตกตะกอน
       ↓
  Filtration   → กรองผ่านทราย/ถ่าน
       ↓
  Disinfection → เติม Chlorine ฆ่าเชื้อโรค
       ↓
  Clear Water Reservoir → เก็บน้ำสะอาด
       ↓
  Distribution → จ่ายไปยังผู้ใช้
```

---

## Step 1341: Coagulation Control - การควบคุมกระบวนการโคแอกกูเลชัน

### Chemical Dosing สำหรับ Coagulation

สาร Alum (Aluminum Sulfate) ถูกเติมลงในน้ำดิบเพื่อทำให้อนุภาคแขวนลอยรวมตัวกัน

```
Alum Dose = Raw Water Flow × Dose Rate
ตัวอย่าง:
  Raw Water: 1,000 m³/hr
  Dose Rate: 30 mg/L (กำหนดจากผล Jar Test)
  Alum Dose: 1,000,000 L/hr × 30 mg/L = 30,000,000 mg/hr = 30 kg/hr
```

**Flow-Proportional Dosing PLC:**

```
=== LADDER: Alum Dosing Control ===

Rung 1: Calculate Dose Requirement
|--[FLOW_METER_OK]--[DOSE_ENABLED]--|
|
| Flow_m3hr = FLOW_PULSES × PULSE_FACTOR
| Dose_Rate = CURRENT_DOSE_RATE  (mg/L จาก Operator หรือ Auto)
| Required_Dose_Lmin = (Flow_m3hr × 1000 / 60) × Dose_Rate / 1000000
|                     = Flow × Dose_Rate / 60000 (L/min)
|
| Pump_Speed_pct = Required_Dose_Lmin / PUMP_MAX_FLOW × 100

Rung 2: Alum Pump VFD Control
|--[DOSE_ENABLED]--[PUMP_READY]--|
|   [MOV Pump_Speed_pct → VFD_SP_AO]--|   // 4-20mA → VFD
|   +--( PUMP_RUN_CMD )--|

Rung 3: Low Level Alarm (Alum Tank)
|--[ALUM_LEVEL < 20]--( LOW_LEVEL_ALARM )--|   // 20% ของ Tank
|--[ALUM_LEVEL < 10]--( CRITICAL_LEVEL_ALARM )--|
|--[ALUM_LEVEL < 10]--( PUMP_STOP )--|   // หยุดปั๊มเมื่อไม่มีสาร

Rung 4: Dose Monitoring
|--[RUNNING_5MIN]--|
|   // ตรวจสอบว่า Flow กับ Dose สอดคล้องกัน
|   [CMP ACTUAL_DOSE vs CALCULATED_DOSE ±10%]--|
|   IF มีความแตกต่างเกิน 10% → DOSE_DEVIATION_ALARM
```

---

## Step 1342: Flocculation Control - การควบคุม Flocculator

### การควบคุม Agitator ใน Flocculation Basin

```
Flocculation Basin แบ่งเป็น 3 Zone:
Zone 1: Rapid Mix    G = 300-500 s⁻¹  (แรงกวน G Value สูง)
Zone 2: Medium Mix   G = 50-100 s⁻¹
Zone 3: Slow Mix     G = 10-30 s⁻¹    (กวนช้า ให้ Floc โต)

G Value = √(Power / (Volume × Viscosity))
G Value ควบคุมโดย VFD ที่ Agitator Motor
```

**Flocculator G-Value Control:**

```pascal
// G-Value Controller for Flocculation
FUNCTION_BLOCK FB_Flocculator
VAR_INPUT
    Zone_ID       : INT;     // 1, 2, 3
    Target_G      : REAL;    // s⁻¹  เป้าหมาย G Value
    Water_Temp    : REAL;    // °C อุณหภูมิน้ำ
    Basin_Volume  : REAL;    // m³
END_VAR
VAR_OUTPUT
    Motor_Speed   : REAL;    // Hz (0-50)
    Actual_G      : REAL;    // s⁻¹
END_VAR
VAR
    Viscosity      : REAL;    // Pa·s (น้ำ)
    Required_Power : REAL;    // W
    Motor_Power_W  : REAL;    // W จาก Power Meter
    Motor_Eff      : REAL := 0.85;  // ประสิทธิภาพมอเตอร์ 85%
END_VAR

// คำนวณ Viscosity ตามอุณหภูมิ
// μ = 0.001 × e^(1808.6/(273+T) - 5.525) [Pa·s]
// ค่าใกล้เคียง: ที่ 20°C = 0.001002, ที่ 25°C = 0.000890 Pa·s
Viscosity := 0.001002 * EXP(-0.025 * (Water_Temp - 20.0));

// คำนวณ Power ที่ต้องการจาก G Target
// G = √(P/(V×μ)) → P = G² × V × μ
Required_Power := (Target_G * Target_G) * Basin_Volume * Viscosity;

// คำนวณ G จาก Power จริง (จาก Power Meter)
IF Basin_Volume > 0 AND Viscosity > 0 THEN
    Actual_G := SQRT(Motor_Power_W * Motor_Eff / (Basin_Volume * Viscosity));
END_IF;

// ปรับความเร็วมอเตอร์เพื่อให้ได้ G ตาม Target
// Power ∝ Speed³ → Speed adjustment
IF Actual_G > 0 THEN
    Motor_Speed := Motor_Speed * (Target_G / Actual_G) ^ (1.0/3.0);
END_IF;

// Limit
Motor_Speed := LIMIT(10.0, Motor_Speed, 50.0);  // 10-50 Hz
END_FUNCTION_BLOCK
```

---

## Step 1343: Sedimentation Control

### Sludge Blanket Level Control

```
Sedimentation Tank (Clarifier):
                  ┌──────────────────────────────┐
    Inflow →  →  │   Floc settling...            │ → Clear Water
                  │                               │
                  │    ▓▓▓▓▓▓▓ ← Sludge Blanket  │
                  │   ▓▓▓▓▓▓▓▓▓                  │
                  └──────────┬───────────────────┘
                             ↓
                       Sludge Drain
                       (Blowdown Valve)

ระดับ Sludge Blanket ต้องไม่สูงจนเกินไป
ถ้าสูงเกินไป → Sludge ไหลออกกับน้ำใส → คุณภาพน้ำแย่
```

**Sludge Blowdown Control:**

```
=== LADDER: Sludge Level and Blowdown Control ===

Rung 1: Sludge Level Measurement
| Sludge Level Sensor: Ultrasonic หรือ Turbidity Sensor
| ค่าสูง = Sludge Blanket สูง (ต้องระบาย)
|
|--[CLARIFIER_RUNNING]--|
|   [MOV AI_TURBIDITY_SENSOR → SLUDGE_LEVEL]--|
|   // ค่า Turbidity สูงที่ระดับ Mid-point = Sludge Blanket location

Rung 2: Sludge High Level Alarm
|--[SLUDGE_LEVEL > 75]--( SLUDGE_HIGH_ALARM )--|   // 75% ของ Tank depth
|--[SLUDGE_LEVEL > 90]--( SLUDGE_CRITICAL )--|

Rung 3: Automatic Blowdown (Time-based)
|--[BLOWDOWN_TIMER, DN]--|   // ทุก 4 ชั่วโมง
|   +--( BLOWDOWN_VALVE_OPEN )--|
|   +--( T_BLOWDOWN, EN )--|    // เปิด 5 นาที

|--[T_BLOWDOWN, DN]--|
|   +--( BLOWDOWN_VALVE_CLOSE )--|
|   +--( BLOWDOWN_TIMER RES )--|  // Reset Timer

Rung 4: Triggered Blowdown (Level-based)
|--[SLUDGE_LEVEL > 75]--[/BLOWDOWN_ACTIVE]--|
|   +--( BLOWDOWN_VALVE_OPEN )--|
|   +--( T_BLOWDOWN, EN )--|    // เปิดจนระดับลดลง

|--[SLUDGE_LEVEL < 40]--[BLOWDOWN_ACTIVE]--|
|   +--( BLOWDOWN_VALVE_CLOSE )--|   // ปิดเมื่อระดับ < 40%
```

---

## Step 1344: Filtration and Backwash Control

### Rapid Sand Filter Control

```
Filter States:
1. FILTERING: น้ำไหลผ่านทรายกรอง
2. BACKWASH:  ล้างทรายด้วยน้ำย้อน
3. RINSE:     ล้างน้ำที่ขุ่นออก
4. DRAIN:     ระบายน้ำใน Filter

Backwash Trigger Conditions:
a) Differential Pressure > 200 mbar (Head Loss สูง)
b) Filtered Water Turbidity > 1.0 NTU
c) Time Since Last Wash > 24 hours (เวลาสูงสุด)
d) Volume Filtered > 3000 m³ (ปริมาตรสูงสุด)
```

**Filter Backwash Sequence:**

```
=== LADDER: Filter Backwash Sequence ===

Rung 1: Backwash Trigger
|--[DIFF_PRESS > 200mbar]--( TRIGGER_BW )--|
|--[TURB_OUTLET > 1.0NTU]--( TRIGGER_BW )--|
|--[T_SINCE_BW, DN]---------( TRIGGER_BW )--|

Rung 2: Backwash State Machine

State 0: FILTERING (Normal)
|--[BW_STATE = 0]--[TRIGGER_BW]--|
|   +--( INLET_VALVE CLOSE )--|
|   +--( OUTLET_VALVE CLOSE )--|
|   +--( BW_STATE := 1 )--|

State 1: DRAIN (ระบายน้ำบน Filter)
|--[BW_STATE = 1]--|
|   +--( DRAIN_VALVE OPEN )--|
|   +--( T_DRAIN, EN )--|      // 2 นาที

|--[BW_STATE = 1]--[T_DRAIN, DN]--|
|   +--( DRAIN_VALVE CLOSE )--|
|   +--( BW_STATE := 2 )--|

State 2: AIR SCOUR (ถ้ามีระบบ Air Scour)
|--[BW_STATE = 2]--|
|   +--( AIR_SCOUR_VALVE OPEN )--|
|   +--( AIR_BLOWER_START )--|
|   +--( T_AIR, EN )--|         // 3 นาที เป่าลมหาก Media

|--[BW_STATE = 2]--[T_AIR, DN]--|
|   +--( AIR_SCOUR_VALVE CLOSE )--|
|   +--( AIR_BLOWER_STOP )--|
|   +--( BW_STATE := 3 )--|

State 3: BACKWASH (น้ำย้อน)
|--[BW_STATE = 3]--|
|   +--( BW_INLET_VALVE OPEN )--|  // เปิดน้ำย้อน
|   +--( BW_PUMP_START )--|
|   +--( WASTE_VALVE OPEN )--|     // ระบายน้ำสกปรก
|   +--( T_BACKWASH, EN )--|       // 8-12 นาที

|--[BW_STATE = 3]--[T_BACKWASH, DN]--|
|   +--( BW_PUMP_STOP )--|
|   +--( BW_INLET_VALVE CLOSE )--|
|   +--( BW_STATE := 4 )--|

State 4: RINSE (ล้างน้ำขุ่น)
|--[BW_STATE = 4]--|
|   +--( INLET_VALVE OPEN )--|    // เปิดน้ำปกติ
|   +--( WASTE_VALVE OPEN )--|    // ทิ้งน้ำแรก
|   +--( T_RINSE, EN )--|         // 3-5 นาที

|--[BW_STATE = 4]--[T_RINSE, DN]--|
|   [CMP RINSE_TURBIDITY < 0.5NTU]--|  // ต้องใสพอก่อนเปิด
|   +--( WASTE_VALVE CLOSE )--|
|   +--( OUTLET_VALVE OPEN )--|   // เปิดจ่ายน้ำ
|   +--( BW_STATE := 0 )--|       // กลับ Filtering
|   +--( BW_COUNT ++ )--|
|   +--( T_SINCE_BW RES )--|      // Reset Timer
```

---

## Step 1345: pH Control - การควบคุมความเป็นกรด-ด่าง

### pH PID Control

```
pH Scale: 0 (กรดแก่) ← 7 (กลาง) → 14 (ด่างแก่)
น้ำประปา Target: 7.2 - 7.6 pH

pH เป็น Logarithmic Scale ทำให้ PID Tuning ยากกว่าปกติ
วิธีแก้: ใช้ Nonlinear PID หรือ pH Neutralization Curve
```

**pH Control with Nonlinear Compensation:**

```pascal
// pH Control - Nonlinear PID
FUNCTION_BLOCK FB_pH_Control
VAR_INPUT
    pH_PV         : REAL;    // ค่า pH จริง (0-14)
    pH_SP         : REAL;    // ค่า pH เป้าหมาย (เช่น 7.4)
    Flow_m3hr     : REAL;    // อัตราการไหลน้ำ
END_VAR
VAR_OUTPUT
    Acid_Pump_pct : REAL;    // 0-100% (เติมกรด ถ้า pH สูง)
    Base_Pump_pct : REAL;    // 0-100% (เติมด่าง ถ้า pH ต่ำ)
END_VAR
VAR
    Error         : REAL;
    Nonlinear_Gain: REAL;
    PID_Output    : REAL;
END_VAR

Error := pH_SP - pH_PV;

// Nonlinear Gain - ยิ่งห่างจาก SP มากยิ่งต้องแก้ไขมาก
// ใกล้ 7 (neutral point): pH curve สูงชัน → gain ต่ำ
// ห่างจาก 7: pH curve ราบ → gain สูง
Nonlinear_Gain := 1.0 + (ABS(Error) * 2.0);

// PID with Variable Gain
PID_Output := Nonlinear_Gain * Error;  // Simplified (จริงๆ ต้องมี I, D)

// ส่ง Output ไปยัง Chemical Pump ที่ถูกต้อง
IF PID_Output > 0 THEN
    // pH ต่ำกว่า SP → เติมด่าง (Caustic Soda NaOH)
    Base_Pump_pct := LIMIT(0.0, PID_Output * Flow_m3hr * 0.5, 100.0);
    Acid_Pump_pct := 0.0;
ELSE
    // pH สูงกว่า SP → เติมกรด (CO2 หรือ H2SO4)
    Acid_Pump_pct := LIMIT(0.0, ABS(PID_Output) * Flow_m3hr * 0.5, 100.0);
    Base_Pump_pct := 0.0;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1346: Chlorine Residual Control - การควบคุมคลอรีนตกค้าง

### Disinfection System

```
การเติม Chlorine:
ระดับที่ต้องการ:
  จุดเติม Chlorine: 2.0-3.0 mg/L (ต้องมากพอ)
  ปลายท่อ (สุดระยะจ่าย): 0.2-0.5 mg/L (มาตรฐาน WHO)
  
  คลอรีนลดลงตาม:
  - ระยะทาง (contact time)
  - อุณหภูมิน้ำ (สูง = ลดเร็ว)
  - ปริมาณสารอินทรีย์ใน Demand
  - pH (สูง = ฆ่าเชื้อได้น้อย)
```

**Chlorine Dosing Control:**

```
=== LADDER: Chlorine Dosing System ===

Rung 1: Compound Loop - Chlorine Control
|--[CHLORINE_ENABLED]--|

| Outer Loop (Slow): Chlorine Analyzer at Outlet
|   SP: 0.8 mg/L (ต้องการที่ outlet)
|   PV: CL_ANALYZER_OUTLET (mg/L)
|   CV: DOSE_SETPOINT (mg/L ที่จุดเติม)
|   Controller: PI, Ti=5min (ช้าเพราะ Dead Time นาน)

| Inner Loop (Fast): Flow-proportional Dose at Inlet
|   SP: DOSE_SETPOINT (จาก Outer Loop)
|   PV: CL_ANALYZER_INLET (mg/L)
|   CV: CL_PUMP_SPEED (0-100%)
|   Controller: PID, เร็ว

Rung 2: CT (Contact Time) Calculation
| CT = Concentration × Time
| กฎหมาย: CT ต้องเพียงพอสำหรับ Log 4 reduction ของ Giardia
| CT_Required = 6 mg·min/L (ที่ pH 7, T=25°C)
|
| Actual_CT = CL_RESERVOIR × RESERVOIR_VOLUME / FLOW_RATE
|
|--[CL_RESERVOIR < 0.5mg/L]--( CT_INSUFFICIENT_ALARM )--|

Rung 3: Chlorine Safety Interlocks
|--[CL_LEAK_DETECTOR]--( CHLORINE_LEAK_ALARM )--|
|--[CL_LEAK_DETECTOR]--( CLOSE_CL_CYLINDER_VALVE )--|
|--[CL_LEAK_DETECTOR]--( START_CL_EXHAUST_FAN )--|
|--[CL_LEAK_DETECTOR]--( ALERT_SAFETY_TEAM )--|
```

---

## Step 1347: Turbidity Control

### Online Turbidity Monitoring and Feedback

```
Turbidity กับ Filter Performance:
  Raw Water: 5-500 NTU (ขึ้นกับฤดูกาลและฝน)
  After Coagulation/Settling: 1-5 NTU
  After Filtration: < 0.1 NTU (WHO Standard)
  
  หน่วย: NTU = Nephelometric Turbidity Unit
  อ่านด้วย: Online Turbidimeter (4-20mA Output)
```

**Turbidity-Based Dose Adjustment:**

```pascal
// Adaptive Coagulant Dosing based on Turbidity
FUNCTION FC_AdaptiveDose : REAL
VAR_INPUT
    Raw_Turbidity   : REAL;   // NTU ของน้ำดิบ
    Filter_Outlet   : REAL;   // NTU หลัง Filter
    Current_Dose    : REAL;   // mg/L Dose ปัจจุบัน
END_VAR
VAR
    Base_Dose   : REAL;
    Adjustment  : REAL;
END_VAR

// คำนวณ Dose พื้นฐานจาก Turbidity ดิบ (จาก Jar Test Model)
IF Raw_Turbidity < 5 THEN
    Base_Dose := 5.0;    // NTU ต่ำ ใช้ Dose น้อย
ELSIF Raw_Turbidity < 20 THEN
    Base_Dose := 10.0 + ((Raw_Turbidity - 5) * 0.5);
ELSIF Raw_Turbidity < 100 THEN
    Base_Dose := 17.5 + ((Raw_Turbidity - 20) * 0.3);
ELSE
    Base_Dose := 41.5 + ((Raw_Turbidity - 100) * 0.2);
END_IF;

// ปรับตาม Filter Outlet Turbidity (Feedback)
IF Filter_Outlet > 0.5 THEN
    // น้ำยังขุ่นเกิน → เพิ่ม Dose
    Adjustment := (Filter_Outlet - 0.1) * 5.0;  // +5 mg/L ต่อ 1 NTU เกิน
ELSIF Filter_Outlet < 0.05 THEN
    // น้ำใสมาก → ลด Dose เพื่อประหยัด
    Adjustment := -2.0;
ELSE
    Adjustment := 0.0;
END_IF;

FC_AdaptiveDose := LIMIT(2.0, Base_Dose + Adjustment, 80.0);
// ไม่น้อยกว่า 2 mg/L และไม่เกิน 80 mg/L
END_FUNCTION
```

---

## Step 1348: SCADA System for Water Treatment

### SCADA Architecture สำหรับ WTP

```
Level 4: Enterprise (SAP, Asset Management)
    ↑
Level 3: SCADA Server (iFIX, AVEVA, Wonderware)
    ↑
Level 2: Historian Server (OSIsoft PI, Wonderware Historian)
    ↑
Level 1: Field Controllers (PLC/RTU at each plant)
    ↑
Level 0: Field Instruments (Sensors, Analyzers, Meters)

Remote Sites ↔ Main WTP ↔ SCADA:
  - ADSL/4G/Fiber communication
  - DNP3 Protocol for water utilities
  - MODBUS TCP for instrument interface
  - OPC UA for MES/ERP integration
```

**SCADA Alarming Configuration:**

```
Alarm Priority สำหรับ Water Treatment:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Priority 1 (Critical - SMS + Siren):
  - Chlorine Gas Leak
  - Total Plant Power Failure
  - No Flow to Distribution
  - Clearwell Tank Empty

Priority 2 (High - SMS):
  - Filter Outlet Turbidity > 1.0 NTU
  - Chlorine Residual < 0.2 mg/L at end of system
  - pH out of range (< 6.5 or > 8.5)
  - Main Pump Failure

Priority 3 (Medium - Email/HMI):
  - Chemical Tank Level < 20%
  - Filter Backwash Required
  - Power Quality Issue
  - Instrument Calibration Due

Priority 4 (Low - HMI Log only):
  - Minor Parameter Deviations
  - Routine Status Changes
  - Maintenance Reminders
```

---

## Step 1349: Remote Site Telemetry

### การส่งข้อมูลจากสถานีสูบน้ำห่างไกล

```
Remote Pump Station → RTU → 4G Network → SCADA → Operators

RTU (Remote Terminal Unit) ทำงาน:
1. รวบรวม Data จาก Sensors และ Switches
2. ส่ง Data ไปยัง SCADA ตาม Schedule (ทุก 1-5 นาที)
3. ส่ง Event-triggered เมื่อมี Alarm
4. รับคำสั่งจาก SCADA (Start/Stop Pump)
5. ทำงานอัตโนมัติแม้ไม่มี Communication
```

**RTU Standalone Control (ทำงานได้ถ้า Communications ขาด):**

```
=== LADDER: RTU Standalone Operation ===

Rung 1: Communication Status Monitor
|--[COMM_WD_TIMER, EN]--|   // Watchdog Timer 5 นาที
| ถ้า SCADA ส่ง Heartbeat มาทุก 1 นาที จะ Reset Timer นี้
|
|--[COMM_WD_TIMER, DN]--( COMM_LOST )--|   // ไม่ได้รับ 5 นาที = Lost

Rung 2: Local Auto Mode (เมื่อ Comm ขาด)
|--[COMM_LOST]--( LOCAL_AUTO )--|
|--[LOCAL_AUTO]--[TANK_LEVEL < 30]--( PUMP_START )--|
|--[LOCAL_AUTO]--[TANK_LEVEL > 80]--( PUMP_STOP )--|

Rung 3: Normal SCADA-Controlled Mode
|--[/COMM_LOST]--[SCADA_PUMP_CMD]--( PUMP_START )--|
|--[/COMM_LOST]--[/SCADA_PUMP_CMD]--( PUMP_STOP )--|

Rung 4: Log Communication Events
|--[COMM_LOST]--[OSR]--|
|   [MOV SYS_TIME → COMM_FAIL_TIME]--|
|   +--( LOG_COMM_FAIL )--|

|--[/COMM_LOST]--[OSF]--|  // OSF = One Shot Falling edge
|   [SUB SYS_TIME, COMM_FAIL_TIME, COMM_DOWN_DURATION]--|
|   +--( LOG_COMM_RESTORE )--|
```

---

## Step 1350: Water Quality Database and Reporting

### Compliance Reporting System

```
กฎหมายที่เกี่ยวข้อง (ประเทศไทย):
- กระทรวงสาธารณสุข: ประกาศกระทรวงฯ เรื่องคุณภาพน้ำประปา
- มาตรฐาน: WHO Guidelines for Drinking-Water Quality (4th Ed)
- รายงาน: ส่ง PEA/กปน/กปภ. ทุกเดือน

Parameter ที่ต้องรายงาน:
- pH: 6.5-8.5
- Turbidity: < 5 NTU (ปกติ < 1 NTU)
- Free Chlorine: 0.2-5.0 mg/L
- Color: < 20 TCU
- TDS: < 1000 mg/L
- Iron: < 0.3 mg/L
- Manganese: < 0.05 mg/L
```

**Compliance Data Logger:**

```pascal
// Data Logger for Regulatory Compliance
FUNCTION_BLOCK FB_ComplianceLog
VAR_INPUT
    pH            : REAL;
    Turbidity     : REAL;    // NTU
    FreeChlorine  : REAL;    // mg/L
    TDS           : REAL;    // mg/L
    Temperature   : REAL;    // °C
    Flow          : REAL;    // m³/hr
    Sample_Time   : DATE_AND_TIME;
END_VAR
VAR_OUTPUT
    All_Compliant  : BOOL;
    Violation_Code : INT;    // 0=OK, 1-7=各parameter
END_VAR

// ตรวจสอบ Compliance
All_Compliant := TRUE;
Violation_Code := 0;

IF pH < 6.5 OR pH > 8.5 THEN
    All_Compliant := FALSE;
    Violation_Code := 1;  // pH violation
END_IF;

IF Turbidity > 5.0 THEN
    All_Compliant := FALSE;
    IF Violation_Code = 0 THEN Violation_Code := 2; END_IF;
END_IF;

IF FreeChlorine < 0.2 OR FreeChlorine > 5.0 THEN
    All_Compliant := FALSE;
    IF Violation_Code = 0 THEN Violation_Code := 3; END_IF;
END_IF;

// บันทึกข้อมูลลง Database ทุก 15 นาที
// ส่งรายงานอัตโนมัติให้หน่วยงานกำกับดูแลผ่าน Email
END_FUNCTION_BLOCK
```

---

## Step 1351-1360: Complete WTP Control System

### Step 1351: Pump Control and Protection

**Centrifugal Pump Control Program:**

```
=== LADDER: Pump Start Sequence (Safe Practice) ===

Rung 1: Pre-Start Checks
|--[START_CMD]--|
|   [DISCHARGE_VALVE_CLOSED]--|   // ปิด Valve ก่อนสตาร์ท
|   [SUCTION_LEVEL_OK]--|         // น้ำพอในถังดูด
|   [MOTOR_TEMP_OK]--|            // มอเตอร์ไม่ร้อน
|   [BEARING_TEMP_OK]--|          // Bearing ไม่ร้อน
|   +--( PRE_START_OK )--|

Rung 2: Start Motor
|--[PRE_START_OK]--|
|   +--( MOTOR_START )--|
|   +--( T_STARTUP, EN )--|       // รอ Motor Speed Up 10 วินาที

Rung 3: Open Discharge Valve
|--[T_STARTUP, DN]--[MOTOR_RUNNING]--|
|   +--( DISCHARGE_VALVE_OPEN )--|

Rung 4: Verify Flow (Anti-Deadhead Protection)
|--[DISCHARGE_VALVE_OPEN]--[T_FLOW_CHECK, DN]--|
|   [CMP FLOW_METER > MIN_FLOW]--|   // ต้องมี Flow
|   +--( PUMP_RUNNING_OK )--|

|--[DISCHARGE_VALVE_OPEN]--[T_FLOW_CHECK, DN]--|
|   [CMP FLOW_METER < MIN_FLOW]--|
|   +--( DEADHEAD_ALARM )--|
|   +--( PUMP_STOP )--|   // หยุดป้องกันปั๊มเสีย
```

### Step 1352: Energy Management for Water Plants

```
Energy Optimization Strategies:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Off-Peak Pumping (ราคาไฟฟ้าถูก = ทำงานมาก)
2. Variable Speed Drives (VFD) ลด Energy 30-50%
3. Hydraulic Optimization (ลด Head Loss)
4. Pump Station Efficiency Curves

Energy Calculation:
Power (kW) = Flow (m³/s) × Head (m) × ρg / (η_pump × η_motor)
            = Q × H × 9810 / (η_pump × η_motor)

ตัวอย่าง:
Q = 0.5 m³/s
H = 40 m
η_pump = 0.75, η_motor = 0.92

Power = 0.5 × 40 × 9810 / (0.75 × 0.92)
      = 196200 / 0.69
      = 284 kW
```

### Step 1353-1360: Integration Summary

**Complete WTP Block Diagram:**

```
RAW WATER                          PROCESS CONTROL
INTAKE                                    ↓
   ↓                            ┌─────────────────┐
SCREENING ← Screen Cleaner      │  Master PLC      │
   ↓         Sequence           │  (S7-1500)       │
CHEM DOSING ← Alum/Polymer      │                  │
   ↓          Dose Control      │  - Recipe Mgmt   │
FLOCCULATION ← G-Value Control  │  - Alarm Mgmt    │
   ↓                            │  - Data Logging  │
SEDIMENTATION ← Sludge Level    │  - Report Gen    │
   ↓                            └────────┬─────────┘
FILTRATION ← BW Control                  │
   ↓                                     │ PROFIBUS/
DISINFECTION ← Cl2 Dose Control          │ ETHERNET
   ↓                                     │
CLEAR WATER ← Level Control     ┌────────┴─────────┐
   ↓                            │   SCADA/HMI      │
DISTRIBUTION ← Pressure Control │   (Wonderware)   │
                                 └──────────────────┘
```

---

## Lab Project 68: Water Treatment Plant PLC System

### โจทย์: ออกแบบระบบ PLC สำหรับโรงผลิตน้ำประปาขนาด 5,000 m³/วัน

**ข้อมูลโรงงาน:**
- กำลังการผลิต: 5,000 m³/วัน = 208 m³/ชั่วโมง
- Raw Water: น้ำแม่น้ำ Turbidity 10-200 NTU (ขึ้นกับฤดู)
- Filter: Rapid Sand Filter 4 Units ขนาด 5m × 5m
- Storage: Clearwell 2,000 m³

**PLC Configuration:**
- Master PLC: Siemens S7-1500 CPU 1515-2 PN
- Remote IO: ET200SP ที่ Filter House, Chemical Room, Pump Station
- HMI: TP1500 Comfort + PC-based SCADA (AVEVA InTouch)
- Field Instruments: 
  - pH: Endress+Hauser Memosens
  - Turbidity: Hach SC200
  - Chlorine: Endress+Hauser CCS51D
  - Flow: Electromagnetic (Siemens SITRANS F)

**งาน:**
1. เขียน Program สำหรับ Backwash Sequence ที่สมบูรณ์
2. Implement Chlorine Residual Compound Control
3. สร้าง Recipe สำหรับ 3 ฤดู (ฤดูแล้ง/ฝน/หนาว)
4. ออกแบบ SCADA Screen และ Alarm List
5. เขียน Compliance Report Generator

**เกณฑ์การประเมิน:**
- Backwash ทำได้ตาม Sequence ถูกต้อง 100% (30 คะแนน)
- Chlorine ควบคุมอยู่ในช่วง 0.5-1.5 mg/L ≥ 95% ของเวลา (25 คะแนน)
- SCADA แสดงสถานะ Real-time ครบทุก Equipment (25 คะแนน)
- Compliance Report ถูกต้องตามรูปแบบที่กำหนด (20 คะแนน)

---

## สรุป Part 68

ใน Part นี้เราได้เรียนรู้:
- กระบวนการบำบัดน้ำตั้งแต่ต้นน้ำถึงปลายน้ำ
- การควบคุม Chemical Dosing แบบ Flow-Proportional
- Backwash Sequence ที่สมบูรณ์
- pH และ Chlorine Control
- SCADA System สำหรับ Water Utilities
- Remote Telemetry และ Standalone Operation
- Compliance Monitoring และ Reporting

**ทักษะสำคัญ:**
- Multi-Zone Control
- Compound Control Loops
- Regulatory Compliance Data Logging
- Fail-Safe Design สำหรับ Critical Infrastructure

---

### 🔗 Navigation
- [← Part 67: Printing & Packaging](part67_printing_packaging.md)
- [→ Part 69: HVAC Automation](part69_hvac_automation.md)
- [📚 สารบัญหลัก](../README.md)
