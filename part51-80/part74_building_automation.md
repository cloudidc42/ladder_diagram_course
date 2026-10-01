# Part 74: Building Automation System (Steps 1461-1480)

## การออกแบบ Building Automation System สำหรับอาคารพาณิชย์

---

### 🔗 Navigation
- [← Part 73: Power Generation](part73_power_generation.md)
- [→ Part 75: Advanced HMI Design](part75_advanced_hmi.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: Building Automation System (BAS)

BAS หรือ BMS (Building Management System) คือระบบที่รวมการควบคุมระบบต่างๆ ในอาคารไว้ในที่เดียว ประกอบด้วย:

- HVAC (Heating, Ventilation, Air Conditioning)
- Lighting Control
- Access Control  
- Fire Alarm System
- Lift/Elevator Control
- Power Metering
- Security Systems

**ตัวอย่าง BAS ในอาคาร 30 ชั้น:**

```
Management Level: Web Dashboard, Mobile App
    ↕ BACnet/IP
Automation Level: Building Controller (BCG)
    ↕ BACnet MS/TP or BACnet/IP
Field Level: Floor Controllers, Sensor Networks
    ↕ 0-10V, 24VAC, RS485
Device Level: VAV Box, FCU, Sensor, Actuator
```

---

## Step 1461: BACnet Protocol Deep Dive

### BACnet Network Types

```
1. BACnet/IP (ASHRAE Addendum g):
   - ผ่าน Ethernet/IP
   - ความเร็วสูง, ระยะไกล
   - Port 47808 (UDP)
   - ใช้กับ Building Controller, Workstation

2. BACnet MS/TP (Master-Slave/Token-Passing):
   - ผ่าน RS485 (EIA-485)
   - ราคาถูก, Wire 3 สาย (A, B, GND)
   - Speed: 9600-115200 baud
   - ระยะสูงสุด: 1200 เมตร
   - ใช้กับ VAV Box, FCU Controller

3. BACnet/SC (Secure Connect) - ใหม่ล่าสุด:
   - TLS 1.3 Encryption
   - WebSocket over TCP
   - สำหรับ Cloud Integration
```

**BACnet Object Model สำหรับ AHU:**

```
AHU BACnet Object Summary:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Object Type | Instance | Name                 | Present Value
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AI          | 1        | Supply Air Temp       | 13.5 °C
AI          | 2        | Return Air Temp       | 26.2 °C
AI          | 3        | Outdoor Air Temp      | 35.1 °C
AI          | 4        | Duct Static Pressure  | 198 Pa
AI          | 5        | Zone CO2 Average      | 850 ppm
AO          | 1        | CHW Valve Position    | 75.3 %
AO          | 2        | Fan Speed VFD         | 68.0 %
AO          | 3        | OA Damper Position    | 35.0 %
BI          | 1        | Supply Fan Running    | Active
BI          | 2        | Filter Dirty          | Inactive
BO          | 1        | AHU Enable            | Active
AV          | 1        | Space Temp Setpoint   | 24.0 °C
AV          | 2        | CO2 Setpoint          | 1000 ppm
BV          | 1        | Occupied Mode         | Active
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**การใช้ BACnet ใน Siemens Desigo CC:**

```pascal
// BACnet Object Read via Siemens BACnet Library
// สำหรับ S7-1500 + Siemens BACnet Gateway

FUNCTION FC_ReadBACnetAI : REAL
VAR_INPUT
    Device_Instance : DINT;     // BACnet Device ID
    Object_Instance : INT;       // Object Instance No.
    Property_ID     : INT := 85; // 85 = Present_Value
END_VAR
VAR
    ReadResult : T_BACnet_AI_Read;
END_VAR

// Trigger Read Request to BACnet Gateway
BACnet_Gateway.ReadProperty(
    Device := Device_Instance,
    ObjectType := 0,    // 0 = Analog Input
    ObjectInst := Object_Instance,
    Property   := Property_ID,
    Result     := ReadResult
);

FC_ReadBACnetAI := ReadResult.Value;
END_FUNCTION
```

---

## Step 1462: MODBUS Integration สำหรับ BAS

### Modbus RTU/TCP สำหรับ HVAC Equipment

```
อุปกรณ์ที่ใช้ Modbus ใน BAS:
  - Chiller: Daikin, Carrier, Trane (Modbus TCP)
  - VFD: ABB, Danfoss, Siemens (Modbus RTU/TCP)
  - Energy Meter: Schneider PM, Eastron SDM (Modbus RTU)
  - UPS: APC, Eaton (Modbus TCP)
  - Cooling Tower Fan: Siemens G120 (Modbus RTU)
```

**Modbus Master Program สำหรับ Chiller:**

```
=== LADDER: Chiller Modbus Communication ===

Rung 1: Trigger Read (ทุก 5 วินาที)
|--[T_READ_TIMER, DN]--|
|   +--( MODBUS_READ_TRIGGER )--|
|   +--( T_READ_TIMER RES )--|

Rung 2: Modbus Function Code 03 - Read Holding Registers
| อ่าน 20 Registers จาก Chiller
| Starting Address: 40001 (Register 1)
| สำหรับ: Leaving Water Temp, Entering Water Temp, 
|         Compressor Status, Alarm Code
|
|--[MODBUS_READ_TRIGGER]--|
|   MSG: MB_READ (Function Code 3)
|   Device Address: 1
|   Start Register: 1
|   Length: 20
|   Data Target: CHILLER_DATA[1]

Rung 3: Process Chiller Data
|--[MB_READ, DN]--|
|   [/MB_READ, ER]--|   // ไม่ Error
|   // แปลงค่าจาก Raw Register เป็น Engineering Units
|   [DIV CHILLER_DATA[1], 10.0, CHW_LEAVING_T]--|  // 255 → 25.5°C
|   [DIV CHILLER_DATA[2], 10.0, CHW_ENTERING_T]--|
|   [MOV CHILLER_DATA[5] → CHILLER_STATUS]--|
|   +--( CHILLER_DATA_OK )--|

Rung 4: Modbus Communication Error Handling
|--[MB_READ, ER]--|
|   [ADD COMMS_FAIL_COUNT, 1, COMMS_FAIL_COUNT]--|
|   +--( CHILLER_COMMS_ALARM )--|
|--[COMMS_FAIL_COUNT > 5]--( CHILLER_COMMS_CRITICAL )--|
```

---

## Step 1463: Lighting Control (DALI Protocol)

### DALI (Digital Addressable Lighting Interface)

```
DALI Protocol:
  - IEC 62386 มาตรฐาน
  - 2 สาย (DALI+ และ DALI-)
  - Addressable: 64 ตัวต่อ Bus
  - Group: 16 Groups
  - Scene: 16 Scenes

DALI Commands:
  - Direct Arc Power Level: หรี่แสง 0-254 (0=off, 254=100%)
  - Go to Scene X: เปิด Scene ที่กำหนด
  - Add to Group X: เพิ่มใน Group
  - Query Status: อ่านสถานะ

ตัวอย่างการใช้ใน Office:
  Scene 1: Presentation Mode (ห้อง Dark, Screen Bright)
  Scene 2: Working Mode (80% สม่ำเสมอ)
  Scene 3: Meeting Mode (70% + Accent Lighting)
  Scene 4: After Hours (20% ทางเดิน)
```

**DALI Control via PLC:**

```pascal
// DALI Master Control (Schneider DALI Master Gateway)
FUNCTION FB_DALI_Scene
VAR_INPUT
    Zone_ID     : INT;      // พื้นที่ 1-20
    Scene_ID    : INT;      // Scene 1-16
    Occupancy   : BOOL;     // มีคนในพื้นที่
    Lux_Level   : REAL;     // Lux จาก Sensor
    Target_Lux  : REAL;     // Lux เป้าหมาย
END_VAR
VAR_OUTPUT
    DALI_Level  : INT;      // 0-254 DALI Level
END_VAR
VAR
    Lux_Error   : REAL;
    Correction  : REAL;
END_VAR

// Occupancy-based control
IF NOT Occupancy THEN
    DALI_Level := 0;    // ปิดไฟ ไม่มีคน
    RETURN;
END_IF;

// Daylight Harvesting (ปรับตามแสงธรรมชาติ)
Lux_Error := Target_Lux - Lux_Level;
Correction := Lux_Error * 0.5;  // Proportional
DALI_Level := REAL_TO_INT(DALI_Level + Correction);
DALI_Level := LIMIT(20, DALI_Level, 254);  // Min 20 = always some light

// Override by Scene
IF Scene_Override THEN
    DALI_Level := DALI_Scene_Table[Zone_ID, Scene_ID];
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1464: Access Control Integration

### การรวม Access Control กับ BAS

```
Access Control + BAS Integration:
  Person Swipe Card → Access Panel → BAS Server
                                         ↓
                          เปิดไฟในห้อง, เปิด AHU Zone
                          บันทึกใครเข้าออกเวลาใด
                          
  Time Schedules:
    Office Hours: 07:00-20:00 (Full HVAC + Lighting)
    After Hours: 20:00-07:00 (Minimal HVAC, No Lighting unless Access)
    Weekend: Minimal Mode, Override บัตร Executive
    
  Zone Occupation Detection:
    ไม่ต้องรอ CO2 สูง → รู้จาก Access Control ว่ามีคนเมื่อไหร่
    ประหยัดเวลา Warm-up ล่วงหน้า 30 นาที
```

---

## Step 1465: Fire Alarm Integration

### BAS ทำงานร่วมกับ Fire Alarm System

```
Fire Alarm → BAS Integration Actions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HVAC:
  ✓ Stop All AHU/FCU (ป้องกันควันแพร่)
  ✓ Close All Dampers (Fire/Smoke Dampers)
  ✓ Pressurize Stairwells (ช่วยการอพยพ)
  ✓ Smoke Exhaust Fan Start (ถ้ามี)
  
Lighting:
  ✓ Emergency Lighting ON Full (100%)
  ✓ Normal Lighting OFF in affected zones
  ✓ Exit Sign Lights 100%
  
Access Control:
  ✓ Release All Locks (Fail-Safe Open)
  ✓ Unlock all Exit Doors
  
Lifts:
  ✓ Return All Lifts to Ground Floor
  ✓ Lock Lifts - No Use by Public
  ✓ Fireman Lift available for Fire Brigade
  
Power:
  ✓ UPS maintain emergency systems
  ✓ Non-essential power may be cut by fire panel
```

**BAS Fire Integration Program:**

```
=== LADDER: Fire Alarm Interface ===

Rung 1: Fire Alarm Signal from FA Panel
| FA Panel ส่ง Dry Contact หรือ BACnet BO
|--[FA_ALARM_GENERAL]--( FIRE_ALARM_BAS )--|

Rung 2: HVAC Response to Fire
|--[FIRE_ALARM_BAS]--|
|   +--( STOP_ALL_AHU )--|
|   +--( CLOSE_ALL_FIRE_DAMPERS )--|
|   +--( START_STAIRWELL_PRESS_FANS )--|
|   +--( START_SMOKE_EXHAUST )--|

Rung 3: Lighting Response
|--[FIRE_ALARM_BAS]--|
|   +--( EMERG_LIGHTS_100PCT )--|
|   // Broadcast DALI Scene 16 (Emergency) ทุก Group

Rung 4: Access Control Response
|--[FIRE_ALARM_BAS]--|
|   +--( RELEASE_ALL_MAGLOCKS )--|
|   +--( UNLOCK_ALL_EXIT_DOORS )--|

Rung 5: Lift Recall
|--[FIRE_ALARM_BAS]--|
|   // ส่ง Command ไปยัง Lift Control Panel
|   [SEND_FIRE_RECALL_TO_LIFT_SYSTEM]--|

Rung 6: Normal Restoration (After All-Clear)
|--[FA_ALL_CLEAR]--[RESET_CMD]--|
|   [T_RESTORE, EN]--|   // รอ 5 นาทีก่อน Restore (ตรวจสอบก่อน)
|--[T_RESTORE, DN]--|
|   +--( RESTORE_HVAC_NORMAL )--|
|   +--( RESTORE_LIGHTING_NORMAL )--|
```

---

## Step 1466: Energy Management in Buildings

### Energy Monitoring and Optimization

```
Building Energy Breakdown (ทั่วไป):
  HVAC: 45-50%
  Lighting: 25-30%
  Plug Loads: 15-20%
  Others: 5-10%
  
Energy Management Strategies:
  1. Demand Response: ลด Peak Demand เมื่อไฟฟ้าแพง
  2. Off-Peak Pre-Cooling: เย็นไว้ก่อน ตอนไฟถูก
  3. Submetering: วัด Energy ทุก Zone → รู้ว่าใครใช้เยอะ
  4. KPI Reporting: kWh/m², kWh/Person
```

**Demand Response Control:**

```pascal
// Demand Response - Reduce Peak Demand
FUNCTION_BLOCK FB_DemandResponse
VAR_INPUT
    Current_kW      : REAL;    // kW การใช้ไฟปัจจุบัน
    Demand_Limit_kW : REAL;    // kW ขีดจำกัด (ตาม Contract)
    Time_to_Peak    : REAL;    // นาที เวลาจนถึง Peak Period
END_VAR
VAR_OUTPUT
    Chiller_Reduce  : REAL;    // % ลด Chiller Load
    Lighting_Reduce : REAL;    // % ลด Lighting
    HVAC_Setback    : REAL;    // °C เพิ่ม Temperature Setpoint
END_VAR
VAR
    Excess_kW       : REAL;
    Reduction_Needed: REAL;
END_VAR

Excess_kW := Current_kW - (Demand_Limit_kW * 0.95);  // 5% Margin

IF Excess_kW > 0 THEN
    // ต้องลด Load
    // Priority: 1=Lighting, 2=HVAC Setback, 3=Chiller
    
    IF Excess_kW > 0 AND Excess_kW <= 50.0 THEN
        // ลด Lighting 10%
        Lighting_Reduce := 10.0;
        
    ELSIF Excess_kW > 50.0 AND Excess_kW <= 150.0 THEN
        // เพิ่ม HVAC Temperature 1°C
        HVAC_Setback := 1.0;
        
    ELSIF Excess_kW > 150.0 THEN
        // ลด Chiller Load
        Chiller_Reduce := MIN(30.0, (Excess_kW - 150.0) / 10.0);
    END_IF;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1467: Fault Detection and Diagnostics (FDD)

### อัตโนมัติตรวจจับความผิดปกติใน HVAC

```
FDD Rule Examples:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Fault: AHU Duct Static Pressure Sensor Failure
  Rule: Fan Speed > 80% AND Duct Pressure < 50 Pa
  → Sensor Failed (should show higher pressure)

Fault: Chilled Water Valve Stuck Open
  Rule: Valve Command < 10% AND (Return Air Temp - 
        Space Temp) > 5°C
  → Valve passing water when commanded closed

Fault: VAV Box Not Responding
  Rule: VAV Command changed > 20% AND Airflow 
        not changed after 5 min
  → Actuator Stuck or Disconnected

Fault: Economizer Damper Stuck
  Rule: OA_Temp < RA_Temp AND OA_Damper = Min%
  → Not using free cooling opportunity (energy waste)

Fault: Low Refrigerant (Chiller)
  Rule: Evaporator Approach = Entering CHW Temp - 
        Evaporator Sat Temp > 5°C (Normal = 1-2°C)
  → Low Refrigerant Charge
```

**FDD Engine in PLC:**

```pascal
// Fault Detection Engine
FUNCTION_BLOCK FB_FDD_Engine
VAR_INPUT
    AHU_Fan_Speed    : REAL;    // %
    Duct_Pressure    : REAL;    // Pa
    CHW_Valve_CMD    : REAL;    // %
    Return_Air_T     : REAL;    // °C
    Space_T          : REAL;    // °C
    VAV_Command      : REAL;    // %
    VAV_Airflow      : REAL;    // L/s
    OA_T             : REAL;    // °C
    RA_T             : REAL;    // °C
    OA_Damper        : REAL;    // %
    OA_Damper_Min    : REAL;    // % Minimum setting
END_VAR
VAR_OUTPUT
    Fault_DuctSensor   : BOOL;
    Fault_CHW_Valve    : BOOL;
    Fault_VAV_Stuck    : BOOL;
    Fault_Economizer   : BOOL;
    Fault_Count        : INT;
END_VAR
VAR
    Fault_Timer : ARRAY[1..10] OF TON;
END_VAR

// FDD Rule 1: Duct Pressure Sensor
Fault_DuctSensor := (AHU_Fan_Speed > 80.0) AND (Duct_Pressure < 50.0);

// FDD Rule 2: CHW Valve Stuck Open
Fault_CHW_Valve := (CHW_Valve_CMD < 10.0) AND 
                    ((Return_Air_T - Space_T) > 5.0);

// FDD Rule 3: VAV Box Not Responding (use TON timer)
Fault_Timer[3](IN := (ABS(VAV_Command - VAV_Command_Prev) > 20.0),
               PT := T#5m);
IF Fault_Timer[3].Q THEN
    IF ABS(VAV_Airflow - VAV_Airflow_Prev) < (VAV_Airflow_Prev * 0.1) THEN
        Fault_VAV_Stuck := TRUE;  // Airflow ไม่เปลี่ยนแม้สั่ง
    END_IF;
END_IF;

// FDD Rule 4: Missed Economizer Opportunity
Fault_Economizer := (OA_T < RA_T - 3.0) AND 
                     (OA_Damper <= OA_Damper_Min + 5.0);

// Count Active Faults
Fault_Count := BOOL_TO_INT(Fault_DuctSensor) +
               BOOL_TO_INT(Fault_CHW_Valve) +
               BOOL_TO_INT(Fault_VAV_Stuck) +
               BOOL_TO_INT(Fault_Economizer);
END_FUNCTION_BLOCK
```

---

## Step 1468-1480: Complete BAS Implementation

### Step 1468: BAS for Commercial Building - System Sizing

```
ตัวอย่างอาคาร: Office Tower 25 ชั้น

HVAC Equipment:
  2 × 500 RT Chillers
  4 × 100 RT AHU (1 ต่อ Zone per floor)
  500 × VAV Boxes
  
Lighting:
  25 × DALI Controllers (1 ต่อชั้น)
  2,000 Luminaires total
  
Metering:
  3 × Main Meters (Electric, CHW, CW)
  50 × Sub-meters (per zone)

Access Control:
  200 Card Readers
  100 Magnetic Door Locks

BAS Hardware:
  1 × Central Workstation (AVEVA Wonderware)
  1 × Building Controller (Siemens Desigo PXC)
  25 × Floor Controllers (Siemens PXC Compact)
  3 × BACnet IP to MS/TP Routers
```

### Step 1469: Scheduling System

**Occupancy Schedule Program:**

```pascal
// Schedule-based Control
FUNCTION FC_IsOccupied : BOOL
VAR_INPUT
    Current_Time  : TIME_OF_DAY;
    Current_DOW   : INT;   // 1=Mon, 7=Sun
    Zone_Type     : INT;   // 1=Office, 2=Lobby, 3=Meeting, 4=Parking
END_VAR

CASE Zone_Type OF
    1: // Office
        IF Current_DOW >= 1 AND Current_DOW <= 5 THEN
            FC_IsOccupied := (Current_Time >= TOD#07:00:00) AND 
                              (Current_Time <= TOD#20:00:00);
        ELSE
            FC_IsOccupied := FALSE;
        END_IF;
    
    2: // Lobby
        FC_IsOccupied := (Current_Time >= TOD#06:00:00) AND 
                          (Current_Time <= TOD#23:00:00);
        // Lobby เปิดยาวกว่า
    
    3: // Meeting Room
        // ใช้ Booking System Data แทน Fixed Schedule
        FC_IsOccupied := Meeting_Room_Booked[Zone_ID];
    
    4: // Parking
        FC_IsOccupied := (Current_Time >= TOD#05:00:00) AND 
                          (Current_Time <= TOD#23:00:00);
END_CASE;
END_FUNCTION
```

### Step 1470-1480: Integration Testing & Commissioning

**BAS Commissioning Steps:**

```
Phase 1: Point Checkout
  □ ตรวจสอบ Wiring ทุก Point (AI, AO, DI, DO)
  □ Verify Sensor Calibration
  □ Actuator Direction Check (Normal Open/Close)
  □ BACnet Device Discovery

Phase 2: Control Loop Testing
  □ PID Tuning (Temperature, Pressure, Flow)
  □ Sequence Testing (Start/Stop, Changeover)
  □ Alarm Testing

Phase 3: Integration Testing
  □ Fire Alarm → HVAC Response
  □ Access Control → Lighting/HVAC
  □ Demand Response Testing
  □ FDD Rule Verification

Phase 4: Performance Verification
  □ Temperature Comfort (23-26°C, 40-60% RH)
  □ Energy Performance Baseline
  □ ASHRAE Guideline 36 Compliance Check

Phase 5: Handover
  □ As-built Documentation
  □ O&M Manual
  □ Operator Training
  □ Warranty Period Support
```

---

## Lab Project 74: Complete BAS for Commercial Building

### โจทย์: ออกแบบ BAS ครบวงจรสำหรับอาคาร 10 ชั้น

**ระบบที่ต้องรวม:**
1. HVAC (2 AHU, 1 Chiller, 20 FCU)
2. Lighting (DALI, 1 Controller ต่อ 2 ชั้น)
3. Energy Metering (3 Sub-meters)
4. Access Control (20 Doors)
5. Fire Alarm Interface

**ข้อกำหนด:**
- Protocol: BACnet/IP สำหรับ Controller, BACnet MS/TP สำหรับ FCU
- Occupancy: Based on CO2 + Motion Sensor + Schedule
- Energy Target: EUI ≤ 150 kWh/m²/year
- Comfort: 23-26°C ≥ 95% Occupied Hours

**งานที่ต้องทำ:**
1. Network Diagram (IP + MS/TP)
2. BACnet Object List สำหรับทุก Device
3. Control Logic สำหรับ HVAC + Lighting
4. Fire Alarm Integration Sequence
5. Energy Report Template (Daily, Monthly, Annual)

---

## สรุป Part 74

ใน Part นี้เราได้เรียนรู้:
- BACnet Protocol (IP, MS/TP, Object Model)
- Modbus Integration สำหรับ HVAC
- DALI Lighting Control
- Access Control + BAS Integration
- Fire Alarm Integration
- Energy Management
- Fault Detection and Diagnostics (FDD)
- BAS Commissioning Process

**ทักษะสำคัญ:**
- Multi-Protocol Integration
- Energy Optimization Logic
- Automated FDD Rules
- System Integration Testing

---

### 🔗 Navigation
- [← Part 73: Power Generation](part73_power_generation.md)
- [→ Part 75: Advanced HMI Design](part75_advanced_hmi.md)
- [📚 สารบัญหลัก](../README.md)
