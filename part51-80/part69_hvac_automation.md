# Part 69: HVAC Automation (Steps 1361-1380)

## การเขียนโปรแกรม PLC/BAS สำหรับระบบปรับอากาศและระบายอากาศ

---

### 🔗 Navigation
- [← Part 68: Water Treatment](part68_water_treatment.md)
- [→ Part 70: Crane & Hoist Control](part70_crane_hoist.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: HVAC และ Building Automation

HVAC (Heating, Ventilation, and Air Conditioning) เป็นระบบที่ใช้พลังงานมากที่สุดในอาคารพาณิชย์ (40-60% ของ Total Energy) การควบคุมที่ดีสามารถประหยัดพลังงานได้ 20-40% ซึ่งคิดเป็นเงินจำนวนมาก

### องค์ประกอบหลักของระบบ HVAC

```
Cooling System:
Chiller Plant → Chilled Water Pump → AHU/FCU → Room Cooling

Heating System (ประเทศหนาว):
Boiler → Hot Water Pump → AHU/FCU → Room Heating

Ventilation:
Supply Fan → Air Distribution → Extract Fan → Exhaust

Air Side:
Return Air → Mixing Box → Filter → Coil → Supply Fan → Diffuser → Room
```

---

## Step 1361: AHU Control - Air Handling Unit

### AHU Components และการควบคุม

```
AHU Diagram:
                         Supply Air
                              ↓
Return Air → [Return Damper] → [Mixing Box] → [Filter] → 
  [Cooling/Heating Coil] → [Supply Fan] → [Duct System] → Rooms

Fresh Air → [Fresh Air Damper] ↗
Exhaust   ← [Exhaust Damper] ↙ (from Return)
```

**AHU Control Sequence:**

```
=== LADDER: AHU Control Sequence ===

Rung 1: AHU Enable Conditions
|--[BUILDING_OCC]--[AHU_ENABLE_SW]--[FIRE_SYS_OK]--|
|   +--( AHU_ENABLE )--|

Rung 2: Pre-Start Checks
|--[AHU_ENABLE]--[FAN_OK]--[FILTER_OK]--|
|   +--( PRE_START_OK )--|

Rung 3: Start Sequence
|--[PRE_START_OK]--[START_CMD]--|
|   +--( OPEN_DAMPERS )--|       // Open Fresh Air + Return Air Damper
|   +--( T_DAMPER_OPEN, EN )--|

|--[T_DAMPER_OPEN, DN]--[DAMPERS_OPEN_CONFIRM]--|
|   +--( START_FAN )--|          // Start Supply Fan (VFD Command)
|   +--( T_FAN_UP, EN )--|       // รอ Fan Speed Up

Rung 4: Supply Air Temperature Control
|--[AHU_RUNNING]--( SAT_PID_ENABLE )--|
|
| SAT_PID:
|   SP: SAT_SETPOINT (ปกติ 13-16°C เป็น Fixed หรือ Reset ตาม Load)
|   PV: SAT_SENSOR (Supply Air Temperature Sensor)
|   CV: CHW_VALVE_POSITION (0-100%)  ← ปรับ Chilled Water Valve
|
| Kp=2.0, Ti=180s, Td=15s (Typical HVAC tuning - ช้ากว่า Process)

Rung 5: Space Temperature Control (VAV System)
|--[AHU_RUNNING]--( SPACE_CONTROL )--|
|
| For VAV (Variable Air Volume):
|   Space_Temp_SP = 24°C (Target)
|   Space_Temp_PV = Room Sensor
|   Error → VAV_BOX_POSITION (0-100% open)
|
| Fan Speed adjusts to maintain Duct Static Pressure
| Duct SP = 150-250 Pa (typical)
```

**Siemens SCL for AHU Control:**

```pascal
// Complete AHU Control Function Block
FUNCTION_BLOCK FB_AHU_Control
VAR_INPUT
    Space_Temp    : REAL;    // °C อุณหภูมิห้อง
    Space_Temp_SP : REAL;    // °C Setpoint (เช่น 24.0)
    Supply_Air_T  : REAL;    // °C อุณหภูมิอากาศส่ง
    Return_Air_T  : REAL;    // °C อุณหภูมิอากาศกลับ
    Outside_Air_T : REAL;    // °C อุณหภูมิภายนอก
    Duct_Press    : REAL;    // Pa แรงดันท่อ
    CO2_Level     : REAL;    // ppm ระดับ CO2
    Occupancy     : BOOL;    // มีคนในพื้นที่หรือไม่
END_VAR
VAR_OUTPUT
    CHW_Valve     : REAL;    // 0-100% Chilled Water Valve
    Fan_Speed     : REAL;    // 0-100% Fan Speed (VFD)
    OA_Damper     : REAL;    // 0-100% Outside Air Damper
    RA_Damper     : REAL;    // 0-100% Return Air Damper
END_VAR
VAR
    // PID States
    SAT_PID_State : T_PID_State;
    Fan_PID_State : T_PID_State;
    
    Duct_Press_SP : REAL := 200.0;  // Pa Target
    SAT_SP        : REAL;           // Supply Air Temp SP (Calculated)
    Min_OA_Frac   : REAL;           // Minimum Fresh Air Fraction
    CO2_Control   : REAL;           // CO2-based damper signal
END_VAR

// Calculate Supply Air Temp Setpoint (Reset based on outdoor temp)
// อากาศเย็นนอก → ต้องการ Supply Air เย็นน้อย → ประหยัด Chiller
IF Outside_Air_T < 30.0 THEN
    SAT_SP := 16.0 + ((30.0 - Outside_Air_T) * 0.3);
    SAT_SP := LIMIT(13.0, SAT_SP, 18.0);  // 13-18°C range
ELSE
    SAT_SP := 13.0;  // Hottest outside = coldest supply
END_IF;

// CO2-Based Ventilation Control (Demand Control Ventilation)
// ASHRAE 62.1: CO2 ในห้อง ≤ 700 ppm เหนือ Outdoor (≈ 1100 ppm total)
IF CO2_Level > 1100.0 THEN
    CO2_Control := (CO2_Level - 1100.0) / 400.0 * 100.0;  // 0-100%
    CO2_Control := LIMIT(0.0, CO2_Control, 100.0);
ELSE
    CO2_Control := 0.0;  // CO2 OK
END_IF;

// Outside Air Damper Control
// Minimum 20% OA for code, more if CO2 high
Min_OA_Frac := 20.0 + CO2_Control * 0.5;  // 20-70% OA
OA_Damper := LIMIT(20.0, Min_OA_Frac, 100.0);
RA_Damper := 100.0 - OA_Damper;  // RA + OA = 100% (simplification)

// Supply Air Temperature PID (controls CHW valve)
SAT_PID_State.SP := SAT_SP;
SAT_PID_State.PV := Supply_Air_T;
FB_PID(State := SAT_PID_State, Kp := 2.0, Ti := 180.0, Td := 15.0);
CHW_Valve := SAT_PID_State.Output;

// Duct Static Pressure Control (controls fan speed)
Fan_PID_State.SP := Duct_Press_SP;
Fan_PID_State.PV := Duct_Press;
FB_PID(State := Fan_PID_State, Kp := 1.5, Ti := 30.0, Td := 5.0);
Fan_Speed := LIMIT(20.0, Fan_PID_State.Output, 100.0);  // Min 20% speed

// Unoccupied Mode (Night Setback)
IF NOT Occupancy THEN
    OA_Damper := 0.0;    // ปิด Fresh Air ตอนไม่มีคน
    Fan_Speed := 30.0;   // ลดความเร็วลง
    SAT_SP := 18.0;      // ไม่ต้องเย็นมาก
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1362: Chiller Control - การควบคุมเครื่องทำน้ำเย็น

### Chiller Plant Optimization

```
Chiller Plant Hierarchy:
  Chiller 1 (1000 RT) → Main Chilled Water Header
  Chiller 2 (1000 RT) → Main Chilled Water Header
  Chiller 3 (500 RT)  → Peak Load Chiller
  
  Primary CHW Pump 1, 2, 3 (เดินตาม Chiller)
  Secondary CHW Pump 1, 2, 3 (VFD, ส่งน้ำเย็นไปอาคาร)
  
  Cooling Tower 1, 2 → Condenser Water Supply
  CW Pump 1, 2, 3
  
Chiller Sequencing Logic:
  Lead-Lag Strategy: สลับเครื่องหลัก-เครื่องสำรองทุกสัปดาห์
  Load-based Staging: เพิ่ม/ลดเครื่องตาม Building Load
```

**Chiller Staging Control:**

```
=== LADDER: Chiller Staging ===

Rung 1: Calculate System Load
| CHW_Load_RT = Flow_GPM × ΔT_F / 24
| ΔT_F = (Return_CHW_T - Supply_CHW_T) × 9/5  [จาก °C เป็น °F]
| หรือ
| CHW_Load_kW = Flow_m3s × ΔT_C × 4.186  [kJ/kg·°C สำหรับน้ำ]

Rung 2: Chiller Add (Stage Up)
|--[T_STABLE_HIGH, DN]--|           // เสถียรมา 15 นาที
|   [CHW_Load_pct > 75]--|          // Load > 75% ของ Chillers ที่เดิน
|   [STANDBY_CHILLER_OK]--|
|   +--( ADD_CHILLER )--|

Rung 3: Chiller Remove (Stage Down)
|--[T_STABLE_LOW, DN]--|            // เสถียรมา 30 นาที
|   [CHW_Load_pct < 40]--|          // Load < 40% → ลดเครื่อง
|   [CHILLERS_RUNNING > 1]--|       // ต้องมีอย่างน้อย 1 เครื่อง
|   +--( REMOVE_CHILLER )--|

Rung 4: Chilled Water Temperature Reset
|--[CHILLER_RUNNING]--|
| CHW_SP_Reset Strategy:
| ถ้า VAV Boxes ทุกตัวไม่ถึง 90% open → CHW ไม่เย็นพอ
| ถ้า VAV Boxes ส่วนใหญ่ < 70% open → CHW เย็นเกินพอ → ขึ้น SP 0.5°C
|
| นี่เรียก "Chilled Water Reset" ช่วยประหยัดไฟ Chiller ได้ 2-3%
```

---

## Step 1363: Cooling Tower Control

### Cooling Tower Fan and Bypass Control

```
Cooling Tower Function:
  ระบาย Heat จาก Chiller Condenser ออกสู่บรรยากาศ
  
  Condenser Water เข้า: 35-37°C
  Condenser Water ออก: 29-32°C (Target)
  
  ควบคุมโดย: Fan Speed (VFD) + Bypass Valve (3-way)
```

**Cooling Tower Basin Temperature Control:**

```pascal
// Cooling Tower Control with Multiple Cells
FUNCTION_BLOCK FB_CoolingTower
VAR_INPUT
    CW_Return_T   : REAL;    // °C น้ำกลับจาก Condenser
    CW_Supply_T   : REAL;    // °C น้ำไปยัง Condenser (Measured)
    CW_Supply_SP  : REAL;    // °C Setpoint (ปกติ 30°C)
    Outside_WB    : REAL;    // °C Wet Bulb Temperature ภายนอก
    Cell_Count    : INT;     // จำนวน Cell ที่ Active
END_VAR
VAR_OUTPUT
    Fan1_Speed    : REAL;    // 0-100% Hz
    Fan2_Speed    : REAL;
    Bypass_Valve  : REAL;    // 0-100% Bypass (100=bypass, 0=full tower)
END_VAR

// คำนวณ Approach Temperature
// Approach = CW_Supply_T - Outside_WB
// ต่ำ = ประสิทธิภาพดี, สูง = ประสิทธิภาพแย่

// Fan Speed Control (PID)
Fan_PID.SP := CW_Supply_SP;
Fan_PID.PV := CW_Supply_T;
Fan_Speed := Fan_PID.Output;  // Higher T → More fan

// Cold Weather Protection: ถ้าหนาว CW เย็นเกินไป
IF Outside_WB < 15.0 THEN
    // ใช้ Bypass Valve บางส่วน ไม่ให้น้ำเย็นเกิน
    IF CW_Supply_T < CW_Supply_SP - 2.0 THEN
        Bypass_Valve := 100.0;  // Bypass ทั้งหมด
    ELSE
        Bypass_Valve := 0.0;    // ผ่าน Tower ทั้งหมด
    END_IF;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1364: BACnet Protocol - มาตรฐานการสื่อสาร Building Automation

### BACnet คืออะไร

BACnet (Building Automation and Control Networks) เป็น Protocol มาตรฐาน ASHRAE/ANSI/ISO สำหรับการสื่อสารระหว่างอุปกรณ์ Building Automation

```
BACnet Object Types ที่สำคัญ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Analog Input (AI)   : อ่านค่า Sensor (Temperature, Pressure, Flow)
Analog Output (AO)  : ส่ง Command (Valve Position, Fan Speed)
Binary Input (BI)   : สถานะ Switch/Sensor (On/Off, Open/Close)
Binary Output (BO)  : Command สวิตช์ (Start/Stop, Open/Close)
Analog Value (AV)   : Parameter/Setpoint ที่ปรับได้
Binary Value (BV)   : Flag/Status (Occupied/Unoccupied)

BACnet Network Types:
  BACnet/IP    : ผ่าน Ethernet/IP Network
  BACnet MS/TP : RS485 Bus, ถูกกว่า BACnet/IP
  BACnet SC    : Secure Connect (ใหม่, TLS encrypted)
```

**BACnet Device Configuration Example:**

```
AHU-1 BACnet Device:
  Device Instance: 10001
  Device Name: "AHU-01-FL03"
  
  Objects:
    AI-1: "Supply Air Temperature"
    AI-2: "Return Air Temperature"  
    AI-3: "Outdoor Air Temperature"
    AI-4: "Duct Static Pressure"
    AI-5: "CO2 Level"
    AO-1: "CHW Valve Command"
    AO-2: "Fan Speed Command"
    AO-3: "OA Damper Command"
    BI-1: "Supply Fan Status"
    BI-2: "Filter DP High"
    BO-1: "AHU Start/Stop"
    AV-1: "Space Temperature Setpoint"
    AV-2: "Min Fresh Air Percentage"
    BV-1: "Occupied/Unoccupied"
    BV-2: "Night Purge Enable"
```

---

## Step 1365: Demand-Based Ventilation (DCV)

### CO2-Based Ventilation Control

```
หลักการ DCV:
  อากาศบริสุทธิ์ที่ต้องการ ∝ จำนวนคน
  CO2 เพิ่ม ∝ จำนวนคน (CO2 ที่คนหายใจออก = 38,000 ppm)
  
  วัด CO2 → คาดการณ์จำนวนคน → ปรับ Ventilation Rate
  
  ประหยัดพลังงาน: 15-30% เทียบกับ Fixed Ventilation Rate
  
  ASHRAE 62.1 Formula:
  Vbz = Rpz × Pz + Rza × Az
  
  Rpz = 0.0025 m³/s per Person (งาน Office)
  Rza = 0.0003 m³/s per m² (Area component)
  Az = Area of Zone (m²)
  Pz = Number of People in Zone
```

**DCV with Multiple Zones:**

```pascal
// DCV System with CO2 Reset
FUNCTION FC_Calculate_OA_Required : REAL
VAR_INPUT
    Zone_CO2      : ARRAY[1..10] OF REAL;  // ppm ของ 10 zones
    Zone_Area     : ARRAY[1..10] OF REAL;  // m² พื้นที่แต่ละ Zone
    Zone_Count    : INT;                    // จำนวน Zone ที่ Active
    Outdoor_CO2   : REAL;                  // ppm CO2 ภายนอก (ปกติ ~415)
END_VAR
VAR
    i             : INT;
    Zone_People   : REAL;
    Zone_OA       : REAL;
    Total_OA_Req  : REAL;
    CO2_Excess    : REAL;
    Rpz : REAL := 0.0025;  // m³/s per Person
    Rza : REAL := 0.0003;  // m³/s per m²
END_VAR

Total_OA_Req := 0.0;

FOR i := 1 TO Zone_Count DO
    // ประมาณจำนวนคนจาก CO2
    CO2_Excess := Zone_CO2[i] - Outdoor_CO2;
    IF CO2_Excess > 0 THEN
        Zone_People := CO2_Excess / 300.0;  // คร่าวๆ: 300 ppm ต่อคน
        Zone_People := LIMIT(0.0, Zone_People, Zone_Area[i] / 5.0);  // Max 1 คน/5m²
    ELSE
        Zone_People := 0.0;
    END_IF;
    
    // คำนวณ OA ที่ต้องการ
    Zone_OA := (Rpz * Zone_People) + (Rza * Zone_Area[i]);
    Total_OA_Req := Total_OA_Req + Zone_OA;
END_FOR;

FC_Calculate_OA_Required := Total_OA_Req;  // m³/s รวมทุก Zone
END_FUNCTION
```

---

## Step 1366: Heat Recovery System

### Energy Recovery Ventilation (ERV)

```
วิธีการ Heat Recovery:
1. Rotary Wheel (ประสิทธิภาพสูงสุด 80-85%)
2. Plate Heat Exchanger (70-80%)  
3. Run-Around Coil (40-60%)

Rotary Wheel Operation:
  Supply Air → [Wheel] → Conditioned Supply Air
  Return Air → [Wheel] → (ความร้อน/ความชื้นถ่ายเทไปแล้ว) → Exhaust

กฎ: ใช้เมื่อ |T_outside - T_inside| > 5°C
    ไม่ใช้เมื่อ T_outside ใกล้เคียง T_inside (Economizer Mode แทน)
```

**Heat Recovery Control Logic:**

```
=== LADDER: Heat Recovery Wheel ===

Rung 1: Enable Conditions
|--[OA_TEMP < 30]--[OA_TEMP > 18]--|   // อากาศเย็นพอที่จะ Recover
|   [/EXHAUST_AIR_RECIRCULATION]--|    // ไม่ให้อากาศเสียกลับเข้า
|   +--( HR_ENABLE )--|

Rung 2: Frost Protection (สำหรับ Climate หนาว)
|--[OA_TEMP < 5]--|
|   [HR_WHEEL_RUNNING]--|
|   // ถ้าอากาศหนาวมาก → น้ำแข็งก่อตัวบน Wheel
|   [FROST_SENSOR]--|
|   +--( HR_BYPASS_OPEN )--|   // เปิด Bypass หลีกเลี่ยง Frost
|   +--( PRE_HEATER_ON )--|    // เปิด Pre-Heater บน OA ก่อน Wheel

Rung 3: Efficiency Monitoring
|--[HR_RUNNING]--|
| HR_Efficiency = (T_supply_after - T_outside) / (T_exhaust - T_outside)
| ถ้า HR_Efficiency < 50% → แจ้งเตือน ล้างทำความสะอาด Wheel

Rung 4: Economizer Override
|--[FREE_COOLING_POSSIBLE]--|  // อากาศนอกเย็นพอใช้โดยตรง
|   +--( HR_STOP )--|          // ไม่ต้อง Recover, ใช้ Outside Air โดยตรง
|   +--( OA_DAMPER 100% )--|   // เปิดอากาศนอกเต็มที่
```

---

## Step 1367: Psychrometric Calculations

### การคำนวณคุณสมบัติของอากาศชื้น

```
Psychrometric Properties:
  Dry Bulb Temperature (Tdb): อุณหภูมิปกติ °C
  Wet Bulb Temperature (Twb): อุณหภูมิ Wet Bulb °C
  Dew Point (Tdp): อุณหภูมิ Dew Point °C
  Relative Humidity (RH): %
  Absolute Humidity (W): kg moisture/kg dry air
  Enthalpy (h): kJ/kg ความร้อนรวม
  
  ใช้สำหรับ: คำนวณ Load, กำหนด Coil Performance, Humidification
```

**Psychrometric Calculations in PLC (Siemens SCL):**

```pascal
// Calculate Enthalpy of Moist Air
FUNCTION FC_Enthalpy : REAL
VAR_INPUT
    Tdb : REAL;   // Dry Bulb °C
    RH  : REAL;   // Relative Humidity 0-100%
END_VAR
VAR
    Psat  : REAL;    // Saturation Pressure kPa
    Pv    : REAL;    // Vapor Pressure kPa
    W     : REAL;    // Humidity Ratio kg/kg
    P_atm : REAL := 101.325;  // Atmospheric Pressure kPa
END_VAR

// Antoine Equation สำหรับ Saturation Pressure
// log10(Psat) = 8.10765 - 1750.286/(235.0 + T) [mm Hg]
// แปลงเป็น kPa:
Psat := EXP(23.7836 - 3782.89 / (Tdb + 233.426)) / 1000.0;

// Vapor Pressure
Pv := (RH / 100.0) * Psat;

// Humidity Ratio
W := 0.62198 * Pv / (P_atm - Pv);

// Enthalpy: h = 1.006×Tdb + W×(2501 + 1.86×Tdb)
FC_Enthalpy := 1.006 * Tdb + W * (2501.0 + 1.86 * Tdb);
// หน่วย: kJ/kg dry air
END_FUNCTION
```

---

## Step 1368: Complete BAS Design - ระบบ Building Automation ครบวงจร

### BAS Architecture for Commercial Building

```
Building Automation System (BAS) Architecture:
                                                   
  Management Level     ┌─────────────────────────┐
  (Web Browser, App)   │   Building Management    │
                       │   Software (SCADA)       │
                       │   Schneider EcoStruxure  │
                       │   or Siemens Desigo CC   │
                       └──────────┬──────────────┘
                                  │ BACnet/IP
                                  │ Ethernet
  Automation Level     ┌──────────┴──────────────┐
                       │  Building Controllers    │
                       │  BCG (Central)           │
                       │  PXC Compact (per floor) │
                       └──────┬──────────────┬───┘
                              │ BACnet MS/TP  │ BACnet/IP
                              │ RS485 Bus     │ 
  Field Level          ┌──────┴──┐    ┌───────┴────┐
                       │ AHU FCU │    │  Chillers  │
                       │ VAV Box │    │  Boilers   │
                       │ Sensors │    │  Pumps     │
                       └─────────┘    └────────────┘
```

**Building Integration Points:**

```pascal
// Integrated Systems in BAS
TYPE T_BuildingSystem :
STRUCT
    // HVAC
    AHU_Count        : INT;
    FCU_Count        : INT;
    Chiller_Count    : INT;
    
    // Lighting (DALI)
    DALI_Groups      : INT;
    Lux_Setpoints    : ARRAY[1..50] OF REAL;
    
    // Access Control
    Door_Count       : INT;
    Card_Reader_Count: INT;
    
    // Fire Alarm Integration
    FA_Panel_Connected: BOOL;
    FA_Zone_Count    : INT;
    
    // Metering
    Elec_Meter_Count : INT;
    Water_Meter_Count: INT;
    
    // Lift (Elevator)
    Lift_Count       : INT;
    Lift_Integrated  : BOOL;
END_STRUCT
END_TYPE
```

---

## Step 1369-1380: Advanced HVAC Topics

### Step 1369: Free Cooling Mode

```
Free Cooling (Economizer) คืออะไร:
เมื่อ อุณหภูมิภายนอก < อุณหภูมิห้อง - 2°C
→ ใช้อากาศนอกโดยตรง ไม่ต้องเปิด Chiller
→ ประหยัดพลังงาน Chiller ได้มาก

ตัวอย่าง: ห้องต้องการ 24°C, ข้างนอก 20°C
→ เปิด Outside Air Damper 100%, ปิด Chiller = Free Cooling!
```

**Free Cooling Control:**

```
=== LADDER: Free Cooling Mode ===

Rung 1: Free Cooling Available Check
|--[OA_ENTHALPY < RA_ENTHALPY - 2]--|
|   // เปรียบเทียบ Enthalpy ดีกว่า Temperature เดียว (คำนึง Humidity)
|   +--( FREE_COOLING_AVAIL )--|

Rung 2: Activate Free Cooling
|--[FREE_COOLING_AVAIL]--[AHU_RUNNING]--|
|   [/CHILLER_ONLY_OVERRIDE]--|
|   +--( OA_DAMPER_FULL_OPEN )--|   // 100% Fresh Air
|   +--( CHW_VALVE_CLOSE )--|       // ปิด Chilled Water
|   +--( CHILLER_UNLOAD_CMD )--|    // ลด Chiller Load

Rung 3: Hybrid Mode (บางครั้งต้องใช้ทั้งสอง)
|--[OA_T < SPACE_T_SP]--[OA_T > OA_T_MIN]--|
|   // อากาศนอกเย็นพอแต่ไม่เย็นเกินไป
|   [SPACE_T > SPACE_T_SP + 1]--|
|   +--( HYBRID_MODE )--|
|   // ใช้ OA 60-80% + Chiller เสริม
```

### Step 1370: KNX Integration

```
KNX Protocol:
  มาตรฐานยุโรปสำหรับ Home/Building Automation
  ใช้มากสำหรับ Lighting, Blinds, HVAC ในอาคาร
  
  KNX Topology:
  IP Router → KNX/TP (Twisted Pair) Bus → Devices
  
  PLC ↔ KNX Integration:
  - ใช้ KNX IP Router (เช่น Weinzierl BAOS)
  - BACnet to KNX Gateway
  - OPC UA to KNX Bridge
```

### Step 1371-1380: System Commissioning

**HVAC Commissioning Checklist สำหรับ BAS:**

```
Pre-Commissioning:
□ ตรวจสอบ Wiring ตาม Drawing
□ Verify Sensor Calibration (Temperature, Pressure, CO2)
□ Check Actuator Direction (Normally Open vs Normally Closed)
□ Confirm Network Communication (IP Address, BACnet Instance)
□ Load Control Program

Functional Testing:
□ AHU Start/Stop Sequence
□ Temperature Control Response Time (< 30 min reach SP)
□ Damper Travel Check (0-100%)
□ VFD Speed Response
□ Alarm Testing (Disconnect sensors one-by-one)
□ Interlock Testing (Fire Alarm → AHU Stop)
□ Communication Failure Test

Balancing:
□ Air Balance (Duct Air Measurement)
□ Hydraulic Balance (CHW Flow through each AHU)
□ Thermal Performance (Actual vs Design Cooling Capacity)
```

---

## Lab Project 69: HVAC BAS for Office Building

### โจทย์: ออกแบบ BAS สำหรับอาคารสำนักงาน 10 ชั้น

**ข้อมูลอาคาร:**
- พื้นที่: 3,000 m² ต่อชั้น × 10 ชั้น = 30,000 m² รวม
- Cooling Load: 1,200 TR (Tons of Refrigeration)
- AHU: 2 ตัว/ชั้น = 20 AHU รวม (แต่ละตัว 60 TR)
- Chiller: 3 × 400 TR = 1,200 TR
- ชั่วโมงทำงาน: 07:00-20:00 วันธรรมดา

**ระบบที่ต้องควบคุม:**
1. Chiller Staging + Chilled Water Reset
2. AHU (20 units) + DCV
3. Cooling Tower (3 units)
4. Energy Metering
5. Fault Detection and Diagnostics

**Target KPIs:**
- COP ของ Plant: ≥ 3.5 (ดี = ≥ 4.0)
- Energy Use Intensity: ≤ 150 kWh/m²/year
- Thermal Comfort: 23-26°C, 40-60% RH ≥ 95% occupied hours

---

## สรุป Part 69

ใน Part นี้เราได้เรียนรู้:
- AHU Control ด้วย Temperature และ Duct Pressure
- Chiller Staging และ CHW Reset
- Cooling Tower Control
- BACnet Protocol สำหรับ Building
- Demand Control Ventilation ด้วย CO2
- Free Cooling / Economizer Mode
- Heat Recovery Systems
- Psychrometric Calculations
- Complete BAS Architecture

**ทักษะสำคัญ:**
- Multi-Loop Control สำหรับ HVAC
- Protocol Integration (BACnet, KNX)
- Energy Optimization Strategies
- Building Commissioning Process

---

### 🔗 Navigation
- [← Part 68: Water Treatment](part68_water_treatment.md)
- [→ Part 70: Crane & Hoist Control](part70_crane_hoist.md)
- [📚 สารบัญหลัก](../README.md)
