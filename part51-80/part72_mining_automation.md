# Part 72: Mining Automation (Steps 1421-1440)

## การเขียนโปรแกรม PLC สำหรับอุตสาหกรรมเหมืองแร่

---

### 🔗 Navigation
- [← Part 71: Press & Stamping](part71_press_stamping.md)
- [→ Part 73: Power Generation](part73_power_generation.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: การทำเหมืองแร่และระบบอัตโนมัติ

อุตสาหกรรมเหมืองแร่มีลักษณะพิเศษที่ท้าทายสำหรับ PLC Engineer:

1. **สภาพแวดล้อมรุนแรง** - ฝุ่น, ความสั่นสะเทือน, อุณหภูมิสูง, น้ำ
2. **Hazardous Area** - อาจมีก๊าซระเบิดได้ (เหมืองถ่านหิน)
3. **Remote Location** - ห่างไกล การสื่อสารยาก
4. **Large Scale** - ระบบใหญ่มาก หลายกิโลเมตร
5. **High Value Equipment** - Crusher, Mill ราคาหลายร้อยล้านบาท

### กระบวนการ Ore Processing ทั่วไป

```
Mine → Crushing → Grinding → Classification → 
Flotation/Leaching → Thickening → Filtration → 
Smelting/Refining → Final Product

สำหรับทองแดง (Copper):
  Open Pit Mine → Primary Crusher (1.5m stones → 300mm)
  → Secondary Crusher (300mm → 75mm)
  → SAG Mill (75mm → 2mm)
  → Ball Mill (2mm → 0.1mm pulp)
  → Flotation (concentrate copper minerals)
  → Thickener (remove water)
  → Filter Press (dry concentrate)
  → Ship to Smelter
```

---

## Step 1421: Crusher Control - การควบคุมเครื่องบด

### Primary Crusher Control (Jaw Crusher / Gyratory Crusher)

```
Gyratory Crusher Design:
          Feed
          ↓
    ┌─────────────────┐
    │  Outer Shell    │
    │  ┌──────────┐   │
    │  │  Mantle  │   │  ← Eccentric movement
    │  └──────────┘   │
    └─────────────────┘
          ↓
       Discharge

Control Challenges:
  - ขนาดหินที่ออก (CSS - Closed Side Setting)
  - Power Draw (กำลังไฟฟ้าที่ใช้)
  - Feed Rate (อัตราการป้อนแร่)
  - Tramp Metal (เหล็กปนมาในแร่)
```

**Crusher PLC Control:**

```
=== LADDER: Primary Crusher Control ===

Rung 1: Start Sequence
|--[START_CMD]--[SAFETY_OK]--|
|   // ต้องเริ่มส่วนล่างก่อน (Conveyor ก่อน)
|   +--( DISCHARGE_CONVEYOR_START )--|
|   +--( T_CONV_START, EN )--|   // รอ 30 วินาที

|--[T_CONV_START, DN]--[DISCHARGE_CONVEYOR_RUNNING]--|
|   +--( CRUSHER_MOTOR_START )--|
|   +--( T_CRUSH_START, EN )|

|--[T_CRUSH_START, DN]--[CRUSHER_RUNNING]--|
|   +--( APRON_FEEDER_START )--|   // เปิด Feed สุดท้าย

Rung 2: Power-Based Feed Control
| ควบคุม Feed Rate ตาม Motor Power (ไม่ให้ Overload)
|
|--[CRUSHER_RUNNING]--|
| POWER_PERCENT = (MOTOR_AMPS × MOTOR_VOLTS × PF × 1.732) / MOTOR_KW × 100
|
| IF POWER_PERCENT < 70% → เพิ่ม Feed Rate
| IF POWER_PERCENT 70-85% → คง Feed Rate
| IF POWER_PERCENT > 85% → ลด Feed Rate
| IF POWER_PERCENT > 95% → หยุด Feed ชั่วคราว

Rung 3: Tramp Metal Detection and Protection
|--[TRAMP_METAL_DETECT]--|   // Metal Detector บน Conveyor
|   +--( TRAMP_HORN )--|
|   +--( OPERATOR_ALERT )--|
|
|--[TRAMP_DETECTED_IN_CRUSHER]--|  // Excessive Vibration = Metal ใน Crusher
|   +--( CRUSHER_STOP )--|
|   +--( TRAMP_EVENT_LOG )--|

Rung 4: CSS (Closed Side Setting) Adjustment
| Hydraulic Jack ใต้ Mantle ปรับขนาดหิน Output
|
|--[CSS_ADJUST_UP_CMD]--[CRUSHER_STOPPED]--|
|   +--( CSS_JACK_EXTEND )--|   // เพิ่ม CSS = ชิ้นโต
|--[CSS_ADJUST_DN_CMD]--[CRUSHER_STOPPED]--|
|   +--( CSS_JACK_RETRACT )--|  // ลด CSS = ชิ้นเล็ก
```

---

## Step 1422: SAG Mill Control - การควบคุม Semi-Autogenous Grinding Mill

### SAG Mill (Semi-Autogenous Grinding)

```
SAG Mill เป็น Rotating Cylinder ขนาดใหญ่ (10-12 เมตรเส้นผ่าน):
  ภายในมี Steel Balls + Ore
  การบด: Ore บดกันเองด้วยแรงหมุน

Key Variables:
  Mill Speed: 70-80% Critical Speed
  Mill Load (Load Cell Weight): ตัวบ่งชี้ความเต็มของ Mill
  Mill Power Draw: ตัวบ่งชี้ประสิทธิภาพ
  Bearing Pressure: ตัวบ่งชี้ Load
  Discharge Density: ความหนาของ Slurry ที่ออก
  
Critical Speed Formula:
  N_c = 42.3 / √D  (RPM, D = Mill Diameter in meters)
  ตัวอย่าง: D = 10m → N_c = 42.3/√10 = 13.4 RPM
  Operating Speed = 0.75 × 13.4 = 10 RPM
```

**SAG Mill Optimization Controller:**

```pascal
// SAG Mill Expert Controller
FUNCTION_BLOCK FB_SAG_Mill_Control
VAR_INPUT
    Mill_Power_kW    : REAL;    // กำลังไฟฟ้าที่ใช้
    Mill_Speed_pct   : REAL;    // % of Critical Speed
    Load_Cell_ton    : REAL;    // น้ำหนักแร่ใน Mill
    Bearing_Press    : REAL;    // bar แรงดัน Bearing
    Discharge_Density: REAL;    // t/m³ ความหนาแน่น Slurry
    Feed_Rate_tph    : REAL;    // t/hr อัตราป้อนแร่
END_VAR
VAR_OUTPUT
    Feed_Rate_SP     : REAL;    // t/hr Setpoint ให้ Feeder
    Water_Add_SP     : REAL;    // m³/hr น้ำที่เติม
    Mill_Speed_SP    : REAL;    // % Setpoint ความเร็ว
END_VAR
VAR
    Optimal_Power_kW : REAL := 15000.0;  // Target Power (Design Point)
    Min_Load         : REAL := 180.0;    // ton ต่ำสุด
    Max_Load         : REAL := 280.0;    // ton สูงสุด
    Load_Error       : REAL;
    Power_Error      : REAL;
END_VAR

// Load Control Strategy
Load_Error := (Min_Load + Max_Load) / 2.0 - Load_Cell_ton;

IF Load_Cell_ton < Min_Load THEN
    // Mill ว่างเกินไป → เพิ่ม Feed
    Feed_Rate_SP := Feed_Rate_tph + 50.0;
ELSIF Load_Cell_ton > Max_Load THEN
    // Mill เต็มเกินไป → ลด Feed
    Feed_Rate_SP := Feed_Rate_tph - 50.0;
ELSE
    // OK - ปรับตาม Power
    Power_Error := Optimal_Power_kW - Mill_Power_kW;
    IF Power_Error > 500.0 THEN
        Feed_Rate_SP := Feed_Rate_tph + 20.0;
    ELSIF Power_Error < -500.0 THEN
        Feed_Rate_SP := Feed_Rate_tph - 20.0;
    END_IF;
END_IF;

// Water Addition for Discharge Density Control
// Target density: 2.4 t/m³ (60% solids by weight)
IF Discharge_Density > 2.5 THEN
    Water_Add_SP := Water_Add_SP + 5.0;  // เพิ่มน้ำ
ELSIF Discharge_Density < 2.3 THEN
    Water_Add_SP := Water_Add_SP - 5.0;  // ลดน้ำ
END_IF;

// Limit
Feed_Rate_SP := LIMIT(200.0, Feed_Rate_SP, 2000.0);
Water_Add_SP := LIMIT(100.0, Water_Add_SP, 1000.0);
END_FUNCTION_BLOCK
```

---

## Step 1423: Flotation Circuit Control

### การควบคุม Flotation ใน Copper Processing

```
Flotation Process:
  Ore Slurry + Reagents (Collector, Frother) + Air
  → Mineral Particles เกาะฟอง (Hydrophobic)
  → Froth ลอยขึ้น = Concentrate
  → Gangue ตกลง = Tailings

Key Controls:
  Air Flow Rate: ควบคุมฟอง
  pH: ส่งผลต่อ Mineral Selectivity
  Reagent Dosing: ปรับตาม Feed Grade
  Froth Level: ตัดแต่งฟองให้พอดี
  Froth Speed: VFD Froth Pump
```

**Flotation Cell Control:**

```
=== LADDER: Flotation Cell Control ===

Rung 1: Air Flow Control
|--[CELL_RUNNING]--|
|   // Air Flowmeter → PID → Air Control Valve
|   [PID_AIR_FLOW]--|
|   SP: AIR_FLOW_SP (L/min จาก Operator/Auto)
|   PV: AIR_FLOW_PV (Flowmeter)
|   CV: AIR_VALVE_POS (0-100%)

Rung 2: Froth Level Control
|--[CELL_RUNNING]--|
|   // Ultrasonic Level → PID → Dart Valve (Feed control)
|   [PID_FROTH_LEVEL]--|
|   SP: FROTH_LEVEL_SP (mm จาก overflow)
|   PV: FROTH_SENSOR (Ultrasonic)
|   CV: DART_VALVE_POS (0-100%)

Rung 3: pH Control
|--[CELL_RUNNING]--|
|   // pH Meter → PID → Lime Addition Pump
|   [PID_pH]--|
|   SP: pH_SP (ปกติ 10.5-11.0 สำหรับ Cu)
|   PV: pH_SENSOR
|   CV: LIME_PUMP_SPEED (0-100%)

Rung 4: Froth Recovery Camera Analysis
| Vision Camera ส่องที่ Froth → Image Analysis Software
| → Froth Velocity, Bubble Size, Texture Score
| → Feedback ให้ปรับ Air, Reagent
|
|--[FROTH_SCORE < 60]--|   // Froth ไม่ดี
|   [ADD FROTHER_DOSE, 1.0, FROTHER_DOSE]--|   // เพิ่ม Frother
```

---

## Step 1424: Belt Conveyor Protection System

### Protection Sensors สำหรับ Belt Conveyor

```
Belt Conveyor Hazards:
  1. Belt Tear/Rip: สายพานขาด ของหล่น
  2. Belt Mistracking: สายพานหลุดออกด้านข้าง
  3. Blocked Chute: วัสดุอุดตัน
  4. Belt Slip: สายพานลื่นไถลบน Pulley
  5. Motor Overload: Load เกิน
  6. Hot Bearing: ความร้อนผิดปกติ

Protection Devices:
  1. Belt Rip Detector: ลวดรอบ Belt ขาด = Alarm
  2. Belt Sway Switch: สายพานเบี้ยว
  3. Zero Speed Switch: Belt หยุดโดยไม่คาดคิด
  4. Pull Cord: ดึง Stop ฉุกเฉินตลอดความยาว
  5. Belt Weigher: วัดน้ำหนักวัสดุบน Belt
  6. Blockage Detector: วัสดุล้น Chute
```

**Comprehensive Belt Conveyor Protection:**

```
=== LADDER: Belt Conveyor Protection ===

Rung 1: Pre-Start Clear Check
|--[START_CMD]--|
|   [/PULL_CORD_TRIPPED]--|      // Pull Cord ไม่ถูกดึง
|   [/BELT_SWAY_LEFT]--|         // สายพานไม่เบี้ยวซ้าย
|   [/BELT_SWAY_RIGHT]--|        // สายพานไม่เบี้ยวขวา
|   [CHUTE_CLEAR]--|             // Chute โล่ง
|   +--( PRE_START_OK )--|

Rung 2: Warning Horn Before Start
|--[PRE_START_OK]--|
|   +--( START_HORN_ON )--|      // แตรเตือน 10 วินาที
|   +--( T_PRE_START, EN )--|

|--[T_PRE_START, DN]--|
|   +--( START_HORN_OFF )--|
|   +--( MOTOR_START_CMD )--|    // เริ่มมอเตอร์

Rung 3: Belt Running Confirmation
|--[MOTOR_START_CMD]--[T_VERIFY, DN]--|  // รอ 10 วินาที
|   [ZERO_SPEED_SW]--|    // Zero Speed Switch ต้องหยุดแสดง (Belt กำลังวิ่ง)
|   // ถ้า Zero Speed ยัง = Belt ไม่วิ่ง → FAULT
|   +--( BELT_RUNNING_OK )--|

Rung 4: Pull Cord Protection (ตลอดความยาว)
|--[PULL_CORD_TRIPPED]--( BELT_EMERGENCY_STOP )--|
| Pull Cord switch ตลอดความยาว Belt
| ดึงตรงไหนก็ได้ → หยุดทันที
| ต้องมี Manual Reset

Rung 5: Belt Sway Protection
|--[BELT_SWAY_LEFT]--( SWAY_ALARM )--|   // แจ้งเตือน
|--[BELT_SWAY_LEFT]--[T_SWAY_ALARM, DN]--( BELT_STOP )--|
| รอ 5 วินาที ถ้าไม่แก้ไข → หยุด Belt

Rung 6: Belt Rip Detection
|--[BELT_RIP_LOOP < RIP_LOOP_NORMAL * 0.5]--|
|   // ลวดขาด = Resistance ลดลงมาก
|   +--( BELT_RIP_ALARM )--|
|   +--( EMERGENCY_STOP_ALL )--|   // หยุดทั้งสายการขนส่ง

Rung 7: Speed-Based Interlock (Sequence)
|--[/UPSTREAM_BELT_RUNNING]--[THIS_BELT_RUNNING]--|
|   // Belt ต้นทางหยุด แต่ Belt ปลายทางยังวิ่ง
|   // ปกติ ถ้าต้นทางหยุด ปลายทางต้องหยุดด้วย
|   +--( SPEED_INTERLOCK_ALARM )--|
```

---

## Step 1425: Hazardous Area (Ex) Equipment

### Zone Classification สำหรับเหมือง

```
ATEX / IECEx Zone Classification:
  Zone 0: Explosive atmosphere ตลอดเวลา (หายาก)
  Zone 1: Explosive atmosphere บางครั้ง (Normal Operation)
  Zone 2: Explosive atmosphere ไม่บ่อย (Abnormal)

Equipment Category:
  Category 1G: Zone 0, 1, 2
  Category 2G: Zone 1, 2
  Category 3G: Zone 2 เท่านั้น

Protection Methods:
  Ex d: Flameproof Enclosure
  Ex e: Increased Safety
  Ex ia/ib: Intrinsic Safety (ปลอดภัยที่สุด สำหรับ Instrument)
  Ex p: Pressurized Enclosure (PLC Cabinet ที่มี N2 Purge)
  Ex n: Non-sparking (Zone 2)
```

**Intrinsically Safe Barrier Design:**

```
มีเหมือนตัวจำกัด Voltage และ Current:
  
  PLC (Safe Area) ↔ Zener Barrier / Galvanic Isolator ↔ Field Device (Zone 1)
  
  Zener Barrier Spec:
  - Ui (Max Input Voltage): 28V
  - Ii (Max Input Current): 93mA
  - Pi (Max Input Power): 660mW
  
  Field Device (Transmitter in Zone 1):
  - Ui ≥ Vo (Barrier Output Voltage)
  - Ci, Li ≤ Allowed by Barrier
  
  Entity Concept:
  - Power from Safe area → Through Barrier → To Field device
  - Even if Fault → Energy limited → Cannot ignite gas
```

---

## Step 1426: Underground Mining Control

### ระบบควบคุมในเหมืองใต้ดิน

```
Underground Mine Systems:
  1. Ventilation Control
     - Main Fan Speed (VFD)
     - Auxiliary Fan สำหรับ Working Faces
     - Gas Monitoring (CH4, CO, NO2, O2)
     
  2. Dewatering (สูบน้ำ)
     - Sump Level Control
     - Multi-Pump Staging
     - Emergency Flooding Detection
     
  3. Hoisting System
     - Shaft Hoists (Man/Material)
     - Skip Hoists (Ore)
     
  4. Communications
     - Leaky Feeder Radio
     - Ethernet Ring (Hardwired)
     - Tag Tracking (RFID) สำหรับ Personnel
```

**Underground Ventilation Control:**

```
=== LADDER: Mine Ventilation Control ===

Rung 1: Gas Monitoring Priority Control
|--[CH4_SENSOR > 1.0%]--|   // 1% CH4 = Warning (LEL = 5%)
|   +--( CH4_ALARM_WARN )--|
|   +--( INCREASE_VENTILATION )--|   // เพิ่ม Fan Speed

|--[CH4_SENSOR > 2.5%]--|   // 2.5% = 50% LEL
|   +--( CH4_ALARM_HIGH )--|
|   +--( EVACUATE_SIGNAL )--|
|   +--( FAN_MAX_SPEED )--|

|--[CH4_SENSOR > 4.0]--|    // 4% = เกือบถึง LEL
|   +--( CH4_CRITICAL )--|
|   +--( POWER_CUTOFF_FACE )--|  // ตัดไฟในพื้นที่ (ยกเว้น Safety Systems)
|   +--( EMERGENCY_EVAC )--|

Rung 2: CO Monitoring (Diesel Equipment)
|--[CO_SENSOR > 25ppm]--|   // TLV = 25ppm
|   +--( CO_ALARM )--|
|   +--( ALERT_OPERATORS )--|

|--[CO_SENSOR > 50ppm]--|   // STEL = 50ppm
|   +--( EVACUATE_DIESEL_AREA )--|
|   +--( DIESEL_EQUIPMENT_STOP )--|

Rung 3: Personnel Tracking Integration
| RFID Tag Readers ตลอดทางในเหมือง
| → รู้ตำแหน่งพนักงานทุกคน
| ถ้า Emergency → ยืนยันว่าทุกคนออกมาแล้ว
|
|--[EVACUATE_SIGNAL]--|
|   [CHECK_PERSONNEL_UNDERGROUND]--|
|   // Count คนที่ยังอยู่ใต้ดิน
|   [SEND_TO_SURFACE_CONTROL]--|
```

---

## Step 1427: Thickener Control

### Paste Thickener สำหรับ Tailings

```
Thickener Function:
  Slurry เจือจาง (5% solids) → Thickener → Underflow (60-70% solids)
  Overflow (clear water) → กลับใช้ใหม่ใน Process
  Underflow → Tailings Storage หรือ Paste Fill
  
Key Controls:
  Underflow Density (Rheology)
  Bed Level / Bed Pressure
  Flocculant Addition
  Rake Torque (ป้องกัน Rake ถูกฝัง)
```

**Thickener Rake Torque Protection:**

```pascal
// Thickener Rake Torque Protection
FUNCTION_BLOCK FB_Thickener
VAR_INPUT
    Rake_Torque_pct  : REAL;    // % ของ Torque สูงสุด
    Bed_Level_m      : REAL;    // เมตร ระดับตะกอน
    Underflow_Density: REAL;    // t/m³
    Overflow_Clarity : REAL;    // NTU
END_VAR
VAR_OUTPUT
    Underflow_Pump_SP: REAL;    // % ความเร็วปั๊ม Underflow
    Flocculant_SP    : REAL;    // m³/hr Flocculant
    Rake_Speed_SP    : REAL;    // % ความเร็ว Rake
    Alarm_High_Torque: BOOL;
END_VAR

// Torque Protection
IF Rake_Torque_pct > 80.0 THEN
    Alarm_High_Torque := TRUE;
    // ลด Rake Speed
    Rake_Speed_SP := Rake_Speed_SP * 0.8;
END_IF;

IF Rake_Torque_pct > 95.0 THEN
    // Rake Lift (ยก Rake ขึ้น)
    // ป้องกัน Rake ถูก Bury และ Shaft หัก
    Rake_Speed_SP := 0.0;
    // Signal to Hydraulic Lift System
END_IF;

// Bed Level Control (via Underflow Pump Speed)
IF Bed_Level_m > 3.0 THEN
    // Bed สูงเกิน → เพิ่ม Pump ออก
    Underflow_Pump_SP := 100.0;
ELSIF Bed_Level_m < 1.5 THEN
    // Bed ต่ำเกิน → ลด Pump ออก
    Underflow_Pump_SP := 30.0;
ELSE
    // PID Control ตาม Bed Level
    // ...
END_IF;

// Flocculant Dosing (proportional to feed)
// ปรับตาม Overflow Clarity
IF Overflow_Clarity > 50.0 THEN
    Flocculant_SP := Flocculant_SP + 5.0;
ELSIF Overflow_Clarity < 10.0 THEN
    Flocculant_SP := Flocculant_SP - 2.0;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1428-1440: Advanced Mining Topics

### Step 1428: Wireless Communication for Mobile Equipment

```
Mining Mobile Equipment:
  Haul Trucks, Dozers, Drills, Shovels
  
  Challenges:
  - อยู่กลางแจ้ง เคลื่อนที่ตลอด
  - Area Coverage กว้างมาก
  
  Solutions:
  - Wi-Fi (802.11ac/ax): สำหรับ Open Pit, ต้องมี AP หลายตัว
  - LTE/4G Private Network: Coverage ดีกว่า Wi-Fi
  - Satellite: สำหรับ Remote Mine
  
  Data Transmitted:
  - GPS Position
  - Payload Weight
  - Engine Hours/Fuel
  - Fault Codes (J1939 CANbus)
  - Operator ID
  - Speed/Heading
```

### Step 1429: Predictive Maintenance for Mining Equipment

**Vibration Analysis for Mills:**

```pascal
// Mill Vibration Analysis (Simplified FFT)
FUNCTION FC_VibrationHealth : REAL
VAR_INPUT
    Accel_Samples : ARRAY[0..511] OF REAL;  // 512 จุด Acceleration (g)
    Sample_Rate   : REAL;    // Hz (ตัวอย่าง 1000 Hz)
END_VAR
VAR
    // Frequency Bands
    Overall_RMS    : REAL;
    SubSync_RMS    : REAL;   // < 1× (0-40Hz)
    Sync_RMS       : REAL;   // 1× (40-80Hz)
    HarmonicBand   : REAL;   // 2-10× (80-400Hz)
    HighFreq_RMS   : REAL;   // > 10× (400Hz+)
    
    Sum_Sq : REAL;
    i      : INT;
END_VAR

// คำนวณ Overall RMS
Sum_Sq := 0.0;
FOR i := 0 TO 511 DO
    Sum_Sq := Sum_Sq + (Accel_Samples[i] * Accel_Samples[i]);
END_FOR;
Overall_RMS := SQRT(Sum_Sq / 512.0);

// ISO 10816-3 Vibration Severity
// Zone A: < 2.8 mm/s (New machinery)
// Zone B: 2.8-7.1 mm/s (Acceptable)
// Zone C: 7.1-18 mm/s (Unsatisfactory, schedule repair)
// Zone D: > 18 mm/s (Danger, immediate action)

FC_VibrationHealth := Overall_RMS;
END_FUNCTION
```

### Step 1430: Complete Ore Processing Control System

**System Integration:**

```
Ore Processing Plant Control Architecture:
  
  Level 3: Business System (SAP Mining)
       ↕ OPC UA
  Level 2: Process Information (Historian - OSIsoft PI)
       ↕ OPC UA  
  Level 1: Process Control (DCS/PLC System)
  
  DCS: Yokogawa Centum VP
    - Grinding Circuit
    - Flotation Circuit
    - Thickening
    - Water Treatment
    
  PLC: Siemens S7-1500
    - Crusher Systems
    - Belt Conveyors
    - Pump Stations
    
  SCADA: Wonderware AVEVA InTouch
    - Operator Interface
    - Alarm Management
    - Reporting
    
  Communication:
    Field: PROFIBUS PA, HART
    Plant: PROFINET, Ethernet/IP
    Enterprise: OPC UA
```

---

## Lab Project 72: Copper Mine Ore Processing PLC

### โจทย์: ออกแบบระบบควบคุมสำหรับโรงแต่งแร่ทองแดง

**ข้อมูลโรงงาน:**
- กำลังการผลิต: 10,000 ตัน/วัน (ดิน/หิน)
- ทองแดง Grade เฉลี่ย: 0.8% Cu
- Recovery Target: 85% ของ Cu ใน Ore
- Concentrate Grade: 25% Cu

**Equipment ที่ต้องควบคุม:**
1. Primary Crusher 1 × 500 tph
2. SAG Mill 1 × 10,000 tph (สื่งที่บดได้)
3. Ball Mill 2 × 5,000 tph  
4. Flotation Cells 20 cells × 200 m³ each
5. Thickener 2 units Ø 50m
6. Concentrate Filter Press 3 units
7. Belt Conveyors 15 sections รวม 8 km

**PLC System:**
- Main Control: Siemens S7-1515 SP Safety (3 units)
- Distributed IO: ET200SP via PROFINET
- Motor Control: Siemens SINAMICS G120P (VFD)
- Field Instruments: HART via ET200SP HA modules
- Safety: TÜV SIL 2 Certified Functions

**งาน:**
1. Belt Conveyor Interlock Logic (Sequential Start/Stop)
2. SAG Mill Load Control (Power-Load Expert Control)
3. Flotation pH Control with Lime Dosing
4. Thickener Rake Torque Protection
5. Emergency Shutdown (ESD) ทั้งระบบ

**ตัวอย่าง Address Table (S7-1500):**

| Signal | Type | Address | Description |
|--------|------|---------|-------------|
| CV01_RUNNING | BOOL | I0.0 | Conveyor 1 Running |
| CV01_FAULT | BOOL | I0.1 | Conveyor 1 Fault |
| SAG_POWER_KW | REAL | IW256 | SAG Mill Power kW |
| SAG_LOAD_TON | REAL | IW258 | SAG Mill Load ton |
| FLOT_pH_01 | REAL | IW300 | Flotation Cell 1 pH |
| LIME_PUMP_01 | REAL | QW256 | Lime Pump 1 Speed % |

---

## สรุป Part 72

ใน Part นี้เราได้เรียนรู้:
- Crusher Control พร้อม Tramp Metal Protection
- SAG Mill Optimization Controller
- Flotation Circuit Control (Air, pH, Level, Reagents)
- Belt Conveyor Protection ครบวงจร
- Hazardous Area (ATEX/IECEx) Equipment
- Underground Mine Control
- Thickener Control

**ทักษะสำคัญ:**
- Harsh Environment PLC Selection
- Intrinsically Safe Circuit Design
- Expert Control for Complex Processes
- Large-Scale System Integration

---

### 🔗 Navigation
- [← Part 71: Press & Stamping](part71_press_stamping.md)
- [→ Part 73: Power Generation](part73_power_generation.md)
- [📚 สารบัญหลัก](../README.md)
