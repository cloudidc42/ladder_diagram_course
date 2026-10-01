# Part 73: Power Generation Control (Steps 1441-1460)

## การเขียนโปรแกรม PLC สำหรับระบบผลิตไฟฟ้า

---

### 🔗 Navigation
- [← Part 72: Mining Automation](part72_mining_automation.md)
- [→ Part 74: Building Automation](part74_building_automation.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: Power Generation และ PLC

ระบบผลิตไฟฟ้าเป็นโครงสร้างพื้นฐานสำคัญของประเทศ PLC Engineer ในสายงานนี้ต้องมีความรู้ทั้งระบบ Mechanical (Turbine, Generator) และ Electrical (Grid, Protection)

### ประเภทของโรงไฟฟ้า

```
1. Thermal Power Plant (โรงไฟฟ้าพลังความร้อน)
   Coal/Gas/Oil → Boiler → Steam → Turbine → Generator
   ขนาด: 100 MW - 1,000 MW

2. Hydro Power Plant (พลังงานน้ำ)
   Dam → Penstock → Turbine → Generator
   ขนาด: 1 MW - 14,000 MW (Three Gorges)

3. Wind Power (พลังงานลม)
   Wind → Blade → Gearbox → Generator
   ขนาด: 1 MW - 15 MW per turbine

4. Solar PV (พลังงานแสงอาทิตย์)
   Solar Cells → Inverter → Grid
   ขนาด: kW - GW (ฟาร์ม)

5. Gas Turbine (เครื่องกังหันแก๊ส)
   Gas → Compressor → Combustor → Turbine → Generator
   ขนาด: 10 MW - 500 MW, Start-up เร็ว
```

---

## Step 1441: Turbine Control Basics

### Steam Turbine Control System

```
Steam Turbine Control Hierarchy:
  
  Electrohydraulic Control (EHC) System
        ↓
  Governor Valve Control (GV) - ควบคุม Steam Flow
        ↓
  Turbine Speed Control → Generator Synchronization
        ↓
  Load Control → Grid Frequency Support
  
  Steam Turbine Types:
    HP (High Pressure) Turbine → IP (Intermediate) → LP (Low Pressure)
    
    Reheat: Steam กลับไปอุ่นซ้ำระหว่าง HP และ IP เพื่อ Efficiency
```

**Governor Control for Speed/Load:**

```pascal
// Steam Turbine Governor Controller
FUNCTION_BLOCK FB_TurbineGovernor
VAR_INPUT
    Speed_rpm       : REAL;    // RPM ความเร็ว Turbine
    Speed_SP_rpm    : REAL;    // RPM Setpoint (ปกติ 3000 สำหรับ 50Hz)
    Load_MW         : REAL;    // MW กำลังผลิตปัจจุบัน
    Load_SP_MW      : REAL;    // MW Setpoint (จาก Grid Operator)
    Grid_Freq_Hz    : REAL;    // Hz ความถี่สาย
    Islanding_Mode  : BOOL;    // TRUE = ตัดออกจาก Grid
END_VAR
VAR_OUTPUT
    Governor_Valve  : REAL;    // 0-100% Steam Flow
    Droop_Correction: REAL;    // Hz/MW Droop
END_VAR
VAR
    Speed_Error     : REAL;
    Load_Error      : REAL;
    Freq_Error      : REAL;
    
    // Droop Setting (ปกติ 4-5% สำหรับ Grid Code)
    Droop_pct       : REAL := 4.0;  // 4% Droop
    Rated_Speed     : REAL := 3000.0;
    Rated_MW        : REAL := 300.0;   // MW Rated
END_VAR

// Speed Droop Control (Primary Frequency Response)
// เมื่อ Grid ความถี่ตก → เพิ่ม Steam → เพิ่ม Output
Freq_Error := Grid_Freq_Hz - 50.0;  // ความถี่เป้าหมาย 50 Hz

// Droop: ΔP = -ΔF × Prated / (Droop × Frated)
Droop_Correction := -(Freq_Error / 50.0) * (100.0 / Droop_pct) * Rated_MW;

// Load Control (Secondary = AGC - Automatic Generation Control)
Load_Error := Load_SP_MW - Load_MW;

// Governor Output
IF Islanding_Mode THEN
    // Island Mode: ควบคุม Speed โดยตรง
    Speed_Error := Speed_SP_rpm - Speed_rpm;
    Governor_Valve := Governor_Valve + Speed_Error * 0.5;  // Proportional
ELSE
    // Grid-Connected: ควบคุม Load + Droop
    Governor_Valve := Governor_Valve + (Load_Error * 0.1) + (Droop_Correction * 0.05);
END_IF;

Governor_Valve := LIMIT(0.0, Governor_Valve, 100.0);
END_FUNCTION_BLOCK
```

---

## Step 1442: Grid Synchronization - การ Synchronize กับสาย

### เงื่อนไขสำหรับ Synchronization

```
ต้องตรงกัน 4 อย่างก่อน Close Breaker:
  1. Voltage Magnitude: ±10% (ปกติ ±5%)
  2. Frequency: ±0.1 Hz (ปกติ ±0.02 Hz)
  3. Phase Angle: ±10° (ปกติ ±5°)
  4. Phase Sequence: ต้องเหมือนกัน (ABC = ABC)

การ Synchronize ทำได้ 2 วิธี:
  A. Manual (Dark Lamp Method): เก่า ไม่แนะนำ
  B. Automatic Synchronizer: ใช้ Relay หรือ PLC
```

**Auto-Synchronizer PLC Program:**

```
=== LADDER: Auto-Synchronizer Logic ===

Rung 1: Voltage Match Check
|--[GEN_VOLT_OK]--|
|   [CMP ABS(GEN_V - BUS_V) <= 0.05 * BUS_V]--|   // ±5%
|   +--( VOLT_MATCH )--|

Rung 2: Frequency Match Check
|--[CMP ABS(GEN_FREQ - BUS_FREQ) <= 0.05]--|  // ±0.05 Hz
|   +--( FREQ_MATCH )--|

Rung 3: Phase Angle Synchronizing
| Phase Difference = GEN_ANGLE - BUS_ANGLE
| ต้องอยู่ใน ±5° ขณะ Close Breaker
|
|--[VOLT_MATCH]--[FREQ_MATCH]--|
|   [CMP PHASE_DIFF >= -5.0 AND PHASE_DIFF <= 5.0]--|
|   [CMP ABS(SLIP_FREQ) < 0.02]--|  // Slip < 0.02 Hz
|   +--( CLOSE_BREAKER_OK )--|

Rung 4: Close Generator Breaker
|--[CLOSE_BREAKER_OK]--[OSR]--|
|   +--( BREAKER_CLOSE_PULSE )--|   // 200ms Pulse
|   +--( START_LOAD_RAMP )--|       // เริ่มรับ Load ทีละน้อย

Rung 5: Load Ramp
|--[GENERATOR_ONLINE]--|
|   // เพิ่ม Load 5 MW ต่อนาที จนถึง Scheduled Load
|   [ADD LOAD_SP, 0.083, LOAD_SP]--|  // 0.083 MW/sec = 5 MW/min
```

---

## Step 1443: Small Hydro Turbine Control

### Francis Turbine Control System

```
Francis Turbine (เหมาะสำหรับ Head 40-600m):
  Water Flow → Spiral Casing → Guide Vanes → Runner → Draft Tube
  
  Control:
    Guide Vane Position (GVP): 0-100% 
    → ควบคุม Water Flow
    → ควบคุม Power Output
    
  Protection:
    Overspeed: > 110% (Runaway Speed)
    High Vibration: Bearing, Shaft
    High Temperature: Generator, Bearing
    Differential Protection: Internal Fault in Generator
```

**Hydro Turbine Full Control System:**

```pascal
// Small Hydro Turbine Controller (1-10 MW)
FUNCTION_BLOCK FB_Hydro_Turbine
VAR_INPUT
    Speed_rpm       : REAL;    // RPM
    Power_MW        : REAL;    // Actual Power
    Power_SP_MW     : REAL;    // Setpoint
    Water_Level_m   : REAL;    // Head (m)
    Guide_Vane_pos  : REAL;    // % 0-100
    Vibration_mm_s  : REAL;    // mm/s
    Bearing_Temp    : REAL;    // °C
    Oil_Temp        : REAL;    // °C
END_VAR
VAR_OUTPUT
    GV_Command      : REAL;    // 0-100% Guide Vane SP
    Brake_Apply     : BOOL;    // Mechanical Brake
    Trip_Signal     : BOOL;    // TRUE = Emergency Stop
    Alarm_Active    : BOOL;
END_VAR
VAR
    Rated_Speed : REAL := 500.0;  // RPM (12 pole, 50Hz = 500 RPM)
    Speed_pct   : REAL;
    Power_Error : REAL;
END_VAR

Speed_pct := Speed_rpm / Rated_Speed * 100.0;

// Protection Checks (Priority Order)
// 1. Overspeed Protection (Hardware Trip, but PLC monitors)
IF Speed_pct > 110.0 THEN
    Trip_Signal := TRUE;
    GV_Command := 0.0;
    // Hardware: Emergency Gate closed by Overspeed Governor
END_IF;

// 2. High Vibration Trip
IF Vibration_mm_s > 15.0 THEN  // ISO 7919-5 Class A = 7.1
    Trip_Signal := TRUE;
    Alarm_Active := TRUE;
END_IF;

// 3. High Temperature
IF Bearing_Temp > 95.0 OR Oil_Temp > 60.0 THEN
    Trip_Signal := TRUE;
END_IF;

// Normal Operation - Load Control
IF NOT Trip_Signal THEN
    Power_Error := Power_SP_MW - Power_MW;
    
    // Guide Vane PID Control
    GV_Command := GV_Command + Power_Error * 0.5;  // Simplified P
    GV_Command := LIMIT(5.0, GV_Command, 100.0);   // Min 5% = no-load
END_IF;

// Mechanical Brake (when stopped)
IF Speed_rpm < 50.0 AND Trip_Signal THEN
    Brake_Apply := TRUE;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1444: Wind Turbine Control

### Wind Turbine Power Control

```
Wind Turbine Control Regions:

Region 1 (Cut-in < 3 m/s): หยุด ลมแรงไม่พอ
Region 2 (3-12 m/s): เพิ่ม Power ตาม Wind Speed (MPPT)
Region 3 (12-25 m/s): Rated Power คงที่ (Pitch Control)
Region 4 (> 25 m/s): หยุด ลมแรงเกิน

Power Curve:
P ∝ V³ (ที่ Region 2)
P = 0.5 × ρ × A × Cp × V³

ρ = 1.225 kg/m³ (อากาศ)
A = π × R² (Rotor Area)
Cp = Power Coefficient (max ~0.59 = Betz Limit)
V = Wind Speed (m/s)
```

**Wind Turbine Pitch Control:**

```
=== LADDER: Wind Turbine Pitch Control ===

Rung 1: Region 2 - MPPT (Max Power Point Tracking)
|--[WIND_SPEED >= 3.0]--[WIND_SPEED < 12.0]--|
|   // Optimal Tip Speed Ratio
|   // TSR = Blade Tip Speed / Wind Speed
|   // Optimal TSR ≈ 7-8 สำหรับ 3-blade
|   
|   [PITCH := 0.0]--|   // Pitch เต็ม (0° = Max Power Capture)
|   [SPEED_SP := WIND_SPEED × TSR_OPT / R]--|  // Generator Speed

Rung 2: Region 3 - Power Limit
|--[WIND_SPEED >= 12.0]--[WIND_SPEED < 25.0]--|
|   [POWER_PV >= RATED_POWER]--|
|   // เพิ่ม Pitch Angle เพื่อ Spill ลม
|   +--( PITCH_PID_ENABLE )--|
|
|   PITCH_PID:
|   SP = RATED_POWER
|   PV = ACTUAL_POWER
|   CV = PITCH_ANGLE (0-90°)
|   ถ้า Power เกิน Rated → เพิ่ม Pitch → ลด Power

Rung 3: Storm Protection
|--[WIND_SPEED > 25.0]--|
|   +--( PITCH_TO_90_DEG )--|   // ปรับ Pitch เต็มสุด = หยุด
|   +--( BRAKE_APPLY )--|

Rung 4: Storm Recovery
|--[WIND_SPEED < 20.0]--[T_CALM, DN]--|  // รอ 10 นาทีหลังลมสงบ
|   +--( RESTART_SEQUENCE )--|
```

---

## Step 1445: Solar Farm Control

### Grid-Tied Solar PV System Control

```
Solar Farm Control System:
  PV Modules → String Combiner → DC Bus → Inverter → Transformer → Grid
                    ↑                          ↓
              String Monitor              Grid Management
              (Fault Detection)           (Active/Reactive Power)
              
Key Controls:
  MPPT (Maximum Power Point Tracking): ทำงานใน Inverter
  Active Power: ควบคุม Active Power Output
  Reactive Power: ควบคุม Power Factor
  Ramp Rate: ไม่ให้ Output เปลี่ยนเร็วเกิน (Grid Code)
  Anti-Islanding: ตรวจสอบ Grid ก่อน Reconnect
```

**Solar Farm SCADA Integration:**

```pascal
// Solar Farm Central Controller
FUNCTION_BLOCK FB_SolarFarm_Control
VAR_INPUT
    Irradiance      : REAL;    // W/m² ความเข้มแสง
    Module_Temp     : REAL;    // °C อุณหภูมิ Panel
    Grid_Voltage    : REAL;    // V แรงดันกริด
    Grid_Freq       : REAL;    // Hz ความถี่
    Active_Power_SP : REAL;    // MW จาก Grid Operator
    PF_SP           : REAL;    // Power Factor Setpoint
    Total_Inverters : INT;     // จำนวน Inverter
END_VAR
VAR_OUTPUT
    Each_Inverter_SP: REAL;    // MW ส่งให้แต่ละ Inverter
    Ramp_Rate_OK    : BOOL;
END_VAR
VAR
    Available_Power  : REAL;  // MW ที่ผลิตได้ตาม Irradiance
    Distributed_SP   : REAL;  // MW ต่อ Inverter
    
    // Ramp Rate Control (ตาม Grid Code: ไม่เกิน 10%/min)
    Max_Ramp_Rate    : REAL := 0.1;  // 10% ต่อนาที
    Previous_SP      : REAL;
    Delta_SP         : REAL;
END_VAR

// ประมาณ Available Power
// P = Irradiance × Panel_Area × Panel_Efficiency × Temp_Derating
Available_Power := Irradiance / 1000.0 * 500000.0 * 0.185 * 
                   (1.0 - 0.004 * (Module_Temp - 25.0));  // 500,000 m², 18.5%, TC=-0.4%/°C

// Ramp Rate Control
Delta_SP := Active_Power_SP - Previous_SP;
IF ABS(Delta_SP) > Available_Power * Max_Ramp_Rate / 60.0 THEN  // Per second
    // Ramp ช้าลง
    Active_Power_SP := Previous_SP + SIGN(Delta_SP) * Available_Power * Max_Ramp_Rate / 60.0;
    Ramp_Rate_OK := FALSE;
ELSE
    Ramp_Rate_OK := TRUE;
END_IF;
Previous_SP := Active_Power_SP;

// Distribute equally to all inverters
Distributed_SP := Active_Power_SP / Total_Inverters;
Each_Inverter_SP := Distributed_SP;
END_FUNCTION_BLOCK
```

---

## Step 1446: Protection Relay Integration

### IEC 61850 Substation Automation

```
IEC 61850 คือ มาตรฐานการสื่อสารใน Digital Substation
  
ก่อน IEC 61850:
  Relay → Hardwired → Control System
  หลาย Vendor = หลาย Protocol (Proprietary)

หลัง IEC 61850:
  Relay → GOOSE Messages (Ethernet) → Control System
  ทุก Vendor ใช้ Protocol เดียวกัน
  
GOOSE (Generic Object Oriented Substation Event):
  Real-time Message (< 4ms) สำหรับ Trip, Interlock
  ทำงานผ่าน Ethernet (Layer 2 Multicast)
  
Benefits:
  - ลด Wiring 60-80%
  - ง่ายต่อการ Modify
  - Self-describing (IED ประกาศ Capability ตัวเอง)
```

**PLC Reading Protection Relay Status via IEC 61850:**

```pascal
// IEC 61850 Client Interface
FUNCTION_BLOCK FB_IEC61850_Client
VAR_INPUT
    Connect_Cmd    : BOOL;
    Server_IP      : STRING[16];   // IP ของ Protection Relay
    GOOSE_Listen   : BOOL;
END_VAR
VAR_OUTPUT
    // Relay Status
    Relay_Trip     : BOOL;         // จาก GOOSE
    Relay_Pickup   : BOOL;
    Circuit_Breaker_Pos : BOOL;    // CB Open=TRUE, Close=FALSE
    
    // Measurements
    Ia_A           : REAL;         // กระแส Phase A
    Ib_A           : REAL;
    Ic_A           : REAL;
    Van_V          : REAL;         // แรงดัน Phase A
    Active_Power_W : REAL;
    
    // Connection Status
    Connected      : BOOL;
    Comms_OK       : BOOL;
END_VAR

// การ Subscribe GOOSE Dataset:
// GOOSE ประกาศ Events เช่น CB Open/Close, Trip
// PLC Subscribe และรับ Real-time

// การอ่าน Measurements:
// ใช้ MMS (Manufacturing Message Specification) Protocol
// อ่านค่าแบบ Poll หรือ Report by Exception
END_FUNCTION_BLOCK
```

---

## Step 1447: DCS vs PLC for Power Generation

### การเลือกใช้ DCS หรือ PLC สำหรับโรงไฟฟ้า

```
DCS (Distributed Control System):
  ข้อดี:
  - Redundancy ทุกระดับ (CPU, I/O, Network, Power)
  - Process Libraries ครบ (PID, Alarm, Historian)
  - Scalable มาก
  - Long-term Vendor Support
  - Certified สำหรับ Critical Process
  
  ข้อเสีย:
  - ราคาแพงมาก (หลายร้อยล้านบาทสำหรับโรงใหญ่)
  - Vendor Lock-in
  - ต้องการ Specialist ในการ Maintain
  
  ใช้เมื่อ:
  - โรงไฟฟ้าขนาดใหญ่ (> 100 MW)
  - กระบวนการต่อเนื่อง Critical
  - ต้องการ High Availability 99.99%

PLC:
  ข้อดี:
  - ราคาถูกกว่า
  - Programming ยืดหยุ่น
  - Fast Response (< 1ms)
  
  ข้อเสีย:
  - ต้อง Custom Develop มาก
  - Redundancy ต้อง Add เอง
  
  ใช้เมื่อ:
  - โรงไฟฟ้าขนาดเล็ก (< 10 MW)
  - Auxiliary Systems
  - Balance-of-Plant (BOP)
```

---

## Step 1448-1460: Complete Hydro Plant Control

### Step 1448: Auxiliary Systems Control

**Cooling Water System for Generator:**

```
Generator Cooling:
  Air-Cooled: ใช้ Fan เป่าลมผ่าน Winding
  Water-Cooled: น้ำหมุนเวียนผ่าน Cooler ใน Air Circuit
  
  Control:
  - ถ้า Generator Hot → เพิ่ม Cooling Water Flow
  - ถ้า Bearing Hot → Alarm + Trip
  - Cooling Water Low Pressure → Alarm
```

### Step 1449: Generator Excitation System

```
Excitation System:
  Generator ต้องการ DC Current ใน Field Winding
  เพื่อสร้าง Magnetic Field → สร้าง Voltage
  
  AVR (Automatic Voltage Regulator):
  - ควบคุม Excitation Current
  - ทำให้ Terminal Voltage คงที่
  - Power Factor Control (ด้วย Reactive Power Q)
  
  Limits:
  - Under-Excitation Limit (UEL): ป้องกัน Gen Absorb Q มากเกิน
  - Over-Excitation Limit (OEL): ป้องกัน Field Winding ร้อนเกิน
  - Stator Current Limit (SCL): ป้องกัน Stator ร้อนเกิน
```

### Step 1450-1460: Plant-wide Summary

**Complete Small Hydro Power Plant (5 MW):**

```
Control System Architecture:
  
  Unit Control Board (UCB):
    Operator Interface สำหรับ 1 Unit
    
  Plant Control Room:
    Master PLC (S7-1516)
    SCADA (Wonderware)
    Historian
    
  Field:
    Unit Controller (Per Turbine)
    Protection Relays (SEL-300G)
    
  Communication:
    Unit ↔ Plant: PROFINET
    Relay ↔ Plant: IEC 61850 GOOSE + MMS
    Plant ↔ Grid Operator: IEC 60870-5-101/104
    
  Critical Sequences:
    Start-up: 45-step sequence
    Shutdown: 30-step sequence
    Emergency Trip: < 100ms
```

---

## Lab Project 73: Small Hydro Plant Control System

### โจทย์: ออกแบบระบบควบคุม Hydro Plant ขนาด 3 × 5 MW

**ข้อมูลโรงงาน:**
- 3 Units × 5 MW = 15 MW Total
- Head: 80 เมตร
- Flow: ±22 m³/s per Unit
- Rated Speed: 500 RPM (12-pole)
- Grid: 22 kV

**สิ่งที่ต้องเขียน:**
1. Unit Start-up Sequence (Step-by-Step)
2. Load Sharing ระหว่าง 3 Units
3. Automatic Synchronization
4. Overspeed Protection
5. Communication กับ Grid Operator (SCADA)

**Start-up Sequence (สำคัญมาก):**

```
Start-up Steps (ลำดับขั้นตอน):
Step 1:  Check All Protections OK
Step 2:  Open Penstock Inlet Valve (รอ 60s)
Step 3:  Release Mechanical Brake
Step 4:  Open Guide Vane to No-Load Position (5%)
Step 5:  Monitor Speed Ramp Up
Step 6:  At 90% Speed → Start AVR, Establish Voltage
Step 7:  At 99% Speed, 0 Hz Slip → Synchronizer checks
Step 8:  Close Generator Breaker
Step 9:  Open Guide Vane to Load Setpoint
Step 10: Monitor All Parameters, Confirm Running
```

---

## สรุป Part 73

ใน Part นี้เราได้เรียนรู้:
- Steam Turbine Governor Control
- Grid Synchronization (Auto-Synchronizer)
- Hydro Turbine Full Control
- Wind Turbine Region Control
- Solar Farm Management
- IEC 61850 Protocol
- DCS vs PLC Decision
- Power Plant Protection Systems

**ทักษะสำคัญ:**
- Critical Sequence Control
- Grid Code Compliance
- Protection Relay Integration
- High Availability System Design

---

### 🔗 Navigation
- [← Part 72: Mining Automation](part72_mining_automation.md)
- [→ Part 74: Building Automation](part74_building_automation.md)
- [📚 สารบัญหลัก](../README.md)
