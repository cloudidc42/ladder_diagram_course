# Part 66: Automotive Automation (Steps 1301-1320)

## การเขียนโปรแกรม PLC สำหรับอุตสาหกรรมยานยนต์ระดับโลก

---

### 🔗 Navigation
- [← Part 65: Advanced Process Control](../part61-80/part65_advanced_process_control.md)
- [→ Part 67: Printing & Packaging](part67_printing_packaging.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: อุตสาหกรรมยานยนต์และ PLC

อุตสาหกรรมยานยนต์เป็นหนึ่งในอุตสาหกรรมที่ใช้ระบบอัตโนมัติอย่างเข้มข้นที่สุดในโลก ทั้ง Toyota, Honda, Ford, BMW, และ Mercedes-Benz ล้วนพึ่งพา PLC และระบบ Automation ในการผลิต การเข้าใจการทำงานของ PLC ในบริบทนี้จึงเป็นทักษะที่มีมูลค่าสูงมาก

### สายการผลิตรถยนต์ทั่วไป

```
  Body Shop → Paint Shop → Assembly Shop → End-of-Line Test
      ↓            ↓             ↓                ↓
  Press Shop   Primer Coat   Engine Install   Water Test
  Welding      Base Coat     Trim Install     Headlight Align
  Hemming      Clear Coat    Wheel Install    Brake Test
               Quality Gate  Final Inspect    Emission Test
```

---

## Step 1301: Press Shop Control - การควบคุมโรงปั๊มโลหะ

### โครงสร้างการทำงานของ Press Line

```
Blanking Press → Draw Press → Redraw Press → Trim Press → Flanging Press
     ↓               ↓            ↓              ↓             ↓
  Sheet Steel    Form Panel    Reshape        Cut Edge       Fold Edge
  500 ton        800 ton       600 ton        400 ton        300 ton
```

**ตัวอย่าง: การควบคุม Transfer Press 800 ตัน**

```
=== LADDER DIAGRAM: Transfer Press Main Control ===

Rung 1: Press Home Position Verification
|--[PB_START]--[SAFETY_OK]--[DIE_LOCK]--[HOME_POS]--+--( PRESS_READY )--|
|                                                     |                   |
|--[AUTO_MODE]--[PROD_RUNNING]--------------------+  +--( RDY_LAMP )-----|
|                                                 |                       |
| PRESS_READY := PB_START AND SAFETY_OK           |                       |
|               AND DIE_LOCK AND HOME_POS         |                       |

Rung 2: Stroke Initiate
|--[PRESS_READY]--[FEED_COMPLETE]--[TRANSFER_OK]--[T_CYCLE, DN]--( STROKE_INIT )--|
|
| Feed system must complete before stroke begins
| Transfer arms must be retracted
| Minimum cycle time enforced by timer T_CYCLE

Rung 3: Clutch Engage
|--[STROKE_INIT]--[BRAKE_RELEASE]--+--( CLUTCH_SOL )--|
|                                  |
|--[INCH_PB]--[INCH_MODE]---------+
|
| Single stroke or inch mode

Rung 4: Tonnage Monitor
|--[STROKE_IN_PROG]--[TONNAGE_OK]--( STROKE_CONT )--|
|                                                     |
|--[STROKE_IN_PROG]--[/TONNAGE_OK]--( TONNAGE_ALARM )--|
|                                                       |
| [/TONNAGE_OK] = TONNAGE > 850 tons (overload) OR < 200 tons (miss-hit)
```

**การกำหนด Address สำหรับ Press Control:**

| Signal | Allen-Bradley | Siemens S7 | Mitsubishi Q | Omron NJ |
|--------|---------------|------------|--------------|----------|
| PB_START | I:0.0/0 | I0.0 | X000 | CIO 0.00 |
| SAFETY_OK | I:0.0/1 | I0.1 | X001 | CIO 0.01 |
| DIE_LOCK | I:0.0/2 | I0.2 | X002 | CIO 0.02 |
| CLUTCH_SOL | O:0.0/0 | Q0.0 | Y000 | CIO 100.00 |
| TONNAGE_ALARM | O:0.0/1 | Q0.1 | Y001 | CIO 100.01 |

---

## Step 1302: Body Shop Welding Control - การควบคุมการเชื่อม

### Resistance Spot Welding (RSW) ที่พบมากในโรงงานรถยนต์

```
=== LADDER DIAGRAM: Spot Weld Cell ===

Rung 1: Robot Weld Cell Safety Gate
|--[GATE_1_CLOSED]--[GATE_2_CLOSED]--[E_STOP_OK]--+--( CELL_SAFE )--|
|                                                   |
|--[MAINT_MODE]--[MAINT_KEY]--------------------+  |
                                                   |
CELL_SAFE ไม่เป็น TRUE ถ้ามีคนเข้าไปในพื้นที่เชื่อม

Rung 2: Weld Gun Position Check
|--[CELL_SAFE]--[ROBOT_READY]--[GUN_OPEN]--( LOAD_PART )--|
|
| เมื่อ Cell ปลอดภัย Robot พร้อม และ Gun เปิด → อนุญาตโหลดชิ้นงาน

Rung 3: Clamp and Weld Sequence
|--[PART_LOADED]--[FIXTURE_HOME]--( CLAMP_INIT )--|
|
|--[CLAMP_COMPLETE]--( ROBOT_START )--|
|
|--[ROBOT_AT_WELD1]--[GUN_PRESSURE_OK]--( WELD_FIRE_1 )--|
|
|--[WELD_1_DONE]--( ROBOT_NEXT_POINT )--|

Rung 4: Weld Counter
|--[WELD_FIRE_1]--[OSR]--( WELD_COUNT ↑ )--|
|
| นับจำนวน Spot Weld ต่อชิ้นงาน
| ตัวอย่าง: Toyota Camry door = 114 spots
```

**Welding Parameters Control (Siemens S7-1500):**

```pascal
// SCL Function: Calculate Weld Time
FUNCTION FC_WeldTimeCalc : REAL
VAR_INPUT
    Sheet_Thickness_1 : REAL;  // mm ความหนาแผ่นเหล็กที่ 1
    Sheet_Thickness_2 : REAL;  // mm ความหนาแผ่นเหล็กที่ 2
    Material_Type : INT;       // 1=Mild Steel, 2=HSLA, 3=Ultra-HSS
END_VAR
VAR
    Base_Time : REAL;
    Thickness_Factor : REAL;
    Material_Factor : REAL;
END_VAR

// คำนวณเวลาเชื่อมพื้นฐาน
Base_Time := 0.2;  // 0.2 วินาที

// Factor ตามความหนา
Thickness_Factor := (Sheet_Thickness_1 + Sheet_Thickness_2) * 0.05;

// Factor ตามวัสดุ
CASE Material_Type OF
    1: Material_Factor := 1.0;   // Mild Steel ปกติ
    2: Material_Factor := 1.3;   // High Strength ต้องใช้เวลาเพิ่ม 30%
    3: Material_Factor := 1.6;   // Ultra High Strength ต้องเพิ่ม 60%
    ELSE: Material_Factor := 1.0;
END_CASE;

FC_WeldTimeCalc := Base_Time + (Thickness_Factor * Material_Factor);
// ผลลัพธ์: เวลาเชื่อมที่เหมาะสม (วินาที)
END_FUNCTION
```

---

## Step 1303: ANDON System - ระบบสัญญาณเตือนสายการผลิต

### ANDON คืออะไร

ANDON (แอนดอน) มาจากภาษาญี่ปุ่นแปลว่า "โคมไฟ" เป็นระบบสัญญาณภาพที่ทำให้ทุกคนในโรงงานทราบสถานะการผลิต

```
       ┌─────────────────────────────────────────┐
       │              ANDON BOARD                │
       │  Station 1  Station 2  Station 3  ...  │
       │  ╔══════╗  ╔══════╗  ╔══════╗         │
       │  ║GREEN ║  ║YELLOW║  ║ RED  ║         │
       │  ║ NORM ║  ║CALL  ║  ║STOP  ║         │
       │  ╚══════╝  ╚══════╝  ╚══════╝         │
       │                                         │
       │  Takt: 58s  Elapsed: 45s  Remaining: 13s│
       └─────────────────────────────────────────┘
```

**ANDON PLC Program (Mitsubishi Q Series):**

```
=== ANDON SYSTEM LADDER ===

Rung 1: Takt Time Counter
|--[START_PULSE]--[TAKT_TIMER, EN]--|
| TAKT_TIMER: TON, PT=58s
| เมื่อ Takt Time หมด → START_PULSE กระตุ้น Timer ใหม่

Rung 2: Green Status (Normal Production)
|--[LINE_RUNNING]--[/YELLOW_ACTIVE]--[/RED_ACTIVE]--( GREEN_LIGHT )--|

Rung 3: Yellow Status (Operator Call)
|--[ANDON_BUTTON]--[/ASSIST_ARRIVED]--( YELLOW_LIGHT )--|
|                                     |
|--[QUALITY_ISSUE]------------------+ |
|                                     |
|--[STOCK_LOW]----------------------+ |
|
| Yellow = ต้องการความช่วยเหลือ แต่ยังไม่หยุดสาย

Rung 4: Red Status (Line Stop)
|--[ANDON_BUTTON]--[T_RESPONSE, DN]--( RED_LIGHT )--|
|                                    |
|--[SAFETY_VIOLATION]---------------+|
|                                    |
|--[QUALITY_CRITICAL]--------------+ |
|
| Red = หยุดสาย ต้องแก้ไขก่อนเดินต่อ

Rung 5: Response Timer
|--[YELLOW_ACTIVE]--[T_RESPONSE, EN]--|
| T_RESPONSE: TON, PT=60s
| ถ้าไม่มีคนมาช่วยภายใน 60 วินาที → เปลี่ยนเป็น Red
```

**ANDON Data Logging (Structured Text - Omron NJ):**

```pascal
// บันทึกเหตุการณ์ ANDON ลง SQL Database ผ่าน EtherNet/IP
FUNCTION_BLOCK FB_AndonLogger
VAR
    EventTime    : DATE_AND_TIME;
    StationNo    : INT;
    EventType    : INT;  // 1=Yellow, 2=Red, 3=Resolved
    DowntimeMin  : REAL;
    SqlCommand   : STRING[500];
END_VAR

// สร้าง SQL Command สำหรับ Insert
SqlCommand := CONCAT('INSERT INTO andon_events ',
              '(station, event_type, event_time, downtime_min) VALUES (',
              INT_TO_STRING(StationNo), ',',
              INT_TO_STRING(EventType), ',''',
              DT_TO_STRING(EventTime), ''',',
              REAL_TO_STRING(DowntimeMin), ')');

// ส่งผ่าน Database Connection (ใช้ Omron SQL Client Library)
SqlClient.Execute(SqlCommand);
END_FUNCTION_BLOCK
```

---

## Step 1304: Poka-Yoke - การป้องกันความผิดพลาด

### หลักการ Poka-Yoke (Error Proofing)

Poka-Yoke หมายถึงการออกแบบระบบให้ป้องกันความผิดพลาดก่อนที่จะเกิดขึ้น หรือตรวจจับความผิดพลาดทันทีที่เกิด

**ประเภทของ Poka-Yoke:**
1. **Prevention Poka-Yoke** - ป้องกันไม่ให้ผิดพลาดได้เลย (ดีที่สุด)
2. **Detection Poka-Yoke** - ตรวจจับความผิดพลาดทันที
3. **Mitigation Poka-Yoke** - ลดผลกระทบจากความผิดพลาด

**ตัวอย่าง: การตรวจสอบการติดตั้ง Bolt**

```
=== LADDER: Torque Wrench Verification (Poka-Yoke) ===

Rung 1: Sequence บังคับ - ต้องขันน็อตตามลำดับ
| หมุดที่ 1 ก่อน → หมุดที่ 2 → หมุดที่ 3 → หมุดที่ 4
|
|--[STEP=1]--[TORQUE_1_OK]--( STEP_ADVANCE )--( STEP:=2 )--|
|
|--[STEP=2]--[TORQUE_2_OK]--( STEP_ADVANCE )--( STEP:=3 )--|
|
|--[STEP=3]--[TORQUE_3_OK]--( STEP_ADVANCE )--( STEP:=4 )--|
|
|--[STEP=4]--[TORQUE_4_OK]--( ALL_COMPLETE )--|

Rung 2: ห้ามข้ามขั้นตอน
|--[TORQUE_2_OK]--[/TORQUE_1_OK]--( SEQUENCE_ALARM )--|
| ถ้า Bolt 2 ถูกขันก่อน Bolt 1 → ALARM!

Rung 3: Torque Value Verification
|--[TORQUE_SENSOR_1]--[CMP >= 35 Nm]--[CMP <= 45 Nm]--( TORQUE_1_OK )--|
|                                                                          |
| Torque ต้องอยู่ในช่วง 35-45 Nm เท่านั้น
| ต่ำกว่า = ขันไม่แน่น, สูงกว่า = ขันแน่นเกิน (เกลียวเสียหาย)

Rung 4: Time Window Control
|--[TORQUE_1_OK]--[T_WINDOW, EN]--|
| T_WINDOW: TOF, PT=30s
| ต้องขัน Bolt 2 ภายใน 30 วินาทีหลังจาก Bolt 1 เสร็จ
| มิฉะนั้น Reset ทั้งหมด
```

---

## Step 1305: OEE Calculation - การคำนวณประสิทธิภาพโดยรวม

### OEE = Availability × Performance × Quality

```
OEE Components:
┌─────────────────────────────────────────────────────────────┐
│  Available Time = Scheduled Time - Planned Downtime          │
│                                                              │
│  Availability = (Available Time - Unplanned Downtime)        │
│                 ÷ Available Time                             │
│                                                              │
│  Performance = Actual Output ÷ (Available Time × Ideal Rate) │
│                                                              │
│  Quality = Good Parts ÷ Total Parts Produced                │
│                                                              │
│  World-Class OEE Target: 85% or higher                      │
│  Typical Automotive: 65-75%                                  │
└─────────────────────────────────────────────────────────────┘
```

**OEE Calculation in PLC (Allen-Bradley Studio 5000):**

```pascal
// Function Block: OEE Calculator
FUNCTION_BLOCK FB_OEE_Calc

VAR_INPUT
    Scheduled_Time    : REAL;  // นาที เวลาที่วางแผนผลิต
    Planned_Downtime  : REAL;  // นาที PM, Break, Lunch
    Actual_Downtime   : REAL;  // นาที Breakdown, Setup
    Ideal_Cycle_Time  : REAL;  // วินาที/ชิ้น เวลาตามทฤษฎี
    Total_Parts       : DINT;  // จำนวนชิ้นทั้งหมด
    Good_Parts        : DINT;  // จำนวนชิ้นดี
END_VAR

VAR_OUTPUT
    Availability : REAL;  // %
    Performance  : REAL;  // %
    Quality      : REAL;  // %
    OEE          : REAL;  // %
END_VAR

VAR
    Available_Time    : REAL;
    Operating_Time    : REAL;
    Ideal_Output      : REAL;
END_VAR

// คำนวณ Availability
Available_Time := Scheduled_Time - Planned_Downtime;
IF Available_Time > 0 THEN
    Availability := ((Available_Time - Actual_Downtime) / Available_Time) * 100.0;
ELSE
    Availability := 0.0;
END_IF;

// คำนวณ Performance
Operating_Time := Available_Time - Actual_Downtime;  // นาที
IF Operating_Time > 0 THEN
    Ideal_Output := (Operating_Time * 60.0) / Ideal_Cycle_Time;
    IF Ideal_Output > 0 THEN
        Performance := (DINT_TO_REAL(Total_Parts) / Ideal_Output) * 100.0;
    END_IF;
END_IF;
Performance := MIN(Performance, 100.0);  // ไม่เกิน 100%

// คำนวณ Quality
IF Total_Parts > 0 THEN
    Quality := (DINT_TO_REAL(Good_Parts) / DINT_TO_REAL(Total_Parts)) * 100.0;
ELSE
    Quality := 0.0;
END_IF;

// คำนวณ OEE
OEE := (Availability * Performance * Quality) / 10000.0;
// หาร 10000 เพราะ %, %, % → แต่ละตัวหาร 100 แล้ว
END_FUNCTION_BLOCK
```

---

## Step 1306: Paint Shop Automation - ระบบพ่นสีรถยนต์

### กระบวนการทาสีรถยนต์

```
Body-in-White → Cleaning → Phosphating → E-Coat → Sealer → 
Primer → Base Coat → Clear Coat → Quality Inspection

อุณหภูมิ Oven ที่ต้องควบคุม:
- E-Coat Oven: 180°C ± 3°C
- Primer Oven: 165°C ± 5°C  
- Top Coat Oven: 140°C ± 3°C
```

**Paint Booth Climate Control:**

```
=== LADDER: Paint Booth Temperature & Humidity Control ===

Rung 1: Temperature Control Loop
|--[BOOTH_ACTIVE]--( TC_ENABLE )--|
|
| PID Controller: TC_PID
| SP: 24.0°C (Target สำหรับ Base Coat)
| PV: T_SENSOR (Thermocouple)
| CV: SUPPLY_AIR_HEAT_VALVE (0-100%)

Rung 2: Humidity Control Loop  
|--[BOOTH_ACTIVE]--( HC_ENABLE )--|
|
| PID Controller: HC_PID
| SP: 60.0% RH (Relative Humidity)
| PV: H_SENSOR (Humidity transmitter)
| CV: HUMIDIFIER_VALVE (0-100%)

Rung 3: Air Flow Monitoring
|--[BOOTH_ACTIVE]--[DIFF_PRESS_OK]--( FLOW_OK )--|
|
| Differential Pressure ระหว่าง Supply และ Return
| บ่งบอก Air Flow Rate เพียงพอ
| ต้อง > 0.3 Pa เพื่อป้องกัน Spray Dust ย้อนกลับ

Rung 4: Solvent Detection
|--[BOOTH_ACTIVE]--( LEL_MONITOR )--|
|
| LEL Sensor (Lower Explosive Limit)
| ถ้า Solvent Concentration > 25% LEL → เพิ่ม Ventilation
| ถ้า > 50% LEL → หยุดพ่นสีทันที
```

---

## Step 1307: Robotic Welding Cell - เซลล์เชื่อมด้วยหุ่นยนต์

### การบูรณาการ PLC กับหุ่นยนต์เชื่อม (FANUC/KUKA/ABB)

```
PLC ↔ Robot Controller Communication:
- Digital I/O: Start, Stop, Part Present, Weld Complete
- Analog I/O: Wire Speed, Voltage, Current Feedback  
- Ethernet/IP: Program Selection, Alarm Status, Production Data
- PROFINET: (สำหรับ KUKA กับ Siemens S7)
```

**PLC Program สำหรับควบคุม Robot Weld Cell:**

```
=== LADDER: Robot Cell Sequencer ===

Rung 1: Cell Initialization
|--[FIRST_SCAN]--+--( CELL_STATE := 0 )--|
                 |--( PART_COUNT := 0 )--|
                 |--( ALARM_CLEAR   )--|

Rung 2: State Machine - Load Position
|--[CELL_STATE = 0]--[GATE_CLOSED]--[ROBOT_HOME]--( LOAD_OK )--|
|--[LOAD_OK]--( CELL_STATE := 1 )--|

Rung 3: State Machine - Weld Sequence
|--[CELL_STATE = 1]--[PART_DETECTED]--[FIXTURE_CLAMPED]--|
|   +--( ROBOT_PGM_SELECT := PART_NO )--|
|   +--( ROBOT_START_PLS )--|
|   +--( CELL_STATE := 2 )--|

Rung 4: State Machine - Wait for Robot Complete
|--[CELL_STATE = 2]--[ROBOT_CYCLE_DONE]--( CELL_STATE := 3 )--|
|--[CELL_STATE = 2]--[ROBOT_FAULT]--( CELL_STATE := 99 )--|

Rung 5: State Machine - Quality Check
|--[CELL_STATE = 3]--|
|   +--[WELD_CURRENT_OK]--[WELD_VOLTAGE_OK]--( QUALITY_OK )--|
|   +--[/QUALITY_OK]--( REWORK_ALARM )--|
|   +--( CELL_STATE := 4 )--|

Rung 6: State Machine - Unload
|--[CELL_STATE = 4]--[FIXTURE_RELEASED]--( UNLOAD_OK )--|
|--[UNLOAD_OK]--( PART_COUNT ++ )--|
|--[UNLOAD_OK]--( CELL_STATE := 0 )--|  // กลับไปเริ่มต้น
```

---

## Step 1308: Quality Gates - ประตูควบคุมคุณภาพ

### ระบบ Quality Gate ในสายการผลิต

Quality Gate คือจุดตรวจสอบในสายการผลิตที่ไม่อนุญาตให้ชิ้นงานที่บกพร่องผ่านไปยังขั้นตอนถัดไป

```
Body Shop → [QG1] → Paint Shop → [QG2] → Trim Shop → [QG3] → End of Line

QG1: Body Dimension Check
     - Laser measurement system
     - 200+ measurement points
     - ±0.5mm tolerance

QG2: Paint Quality Check  
     - Color spectrophotometer
     - Surface defect scanner
     - Film thickness measurement

QG3: Final Assembly Check
     - Torque verification
     - Gap and flush measurement
     - Functional test
```

**Quality Gate PLC Logic:**

```
=== LADDER: Quality Gate Control ===

Rung 1: Measurement System Trigger
|--[PART_AT_STATION]--[MEAS_SYS_READY]--[OSR]--( MEASURE_START )--|

Rung 2: Collect Measurement Results
|--[MEASURE_DONE]--|
|   [MOV Source:MEAS_DATA Dest:QG_RESULT]--|
|   [MOV Source:SYS_TIME  Dest:QG_TIME  ]--|

Rung 3: Pass/Fail Decision
|--[QG_RESULT.ALL_IN_SPEC]--( QG_PASS )--|
|--[/QG_RESULT.ALL_IN_SPEC]--( QG_FAIL )--|

Rung 4: Routing Decision
|--[QG_PASS]--( OPEN_TRANSFER_GATE )--|   // ส่งต่อสายหลัก
|--[QG_FAIL]--( OPEN_REWORK_GATE )--|    // เบี่ยงไปสาย Rework

Rung 5: Data Recording
|--[MEASURE_DONE]--( DB_WRITE )--|
| บันทึก: Part Serial No., Measurement Values, Pass/Fail, Timestamp
| ลงใน Manufacturing Execution System (MES)
```

---

## Step 1309: Line Balancing - การปรับสมดุลสายการผลิต

### Takt Time Monitoring

```
Takt Time = Available Production Time ÷ Customer Demand
ตัวอย่าง:
- Available Time: 480 นาที/กะ
- Customer Demand: 240 คัน/กะ
- Takt Time = 480/240 = 2 นาที/คัน = 120 วินาที/คัน
```

**Takt Time Monitor PLC:**

```
=== LADDER: Takt Time Monitor ===

Rung 1: Takt Timer Reset
|--[PART_EXITS_STATION]--[OSR]--|
|   +--[MOV Source:T_ELAPSED.ACC Dest:LAST_CYCLE_TIME]--|
|   +--[RES T_ELAPSED]--|

Rung 2: Cycle Time Accumulation
|--[PART_IN_STATION]--[T_ELAPSED, EN]--|
| T_ELAPSED: TON (Accumulating Timer)

Rung 3: Takt Compliance Check
|--[PART_EXITS_STATION]--[CMP LAST_CYCLE_TIME <= TAKT_TIME]--|
|   +--( WITHIN_TAKT )--|

Rung 4: Behind Takt Alert
|--[PART_EXITS_STATION]--|
|   [CMP LAST_CYCLE_TIME > TAKT_TIME]--|
|   [CMP LAST_CYCLE_TIME <= TAKT_TIME*1.1]--|
|   +--( BEHIND_TAKT_WARNING )--|   // เกิน Takt ไม่เกิน 10%

Rung 5: Critical Behind Takt
|--[PART_EXITS_STATION]--[CMP LAST_CYCLE_TIME > TAKT_TIME*1.1]--|
|   +--( BEHIND_TAKT_CRITICAL )--|  // เกิน Takt เกิน 10% → ANDON Yellow
```

---

## Step 1310: End-of-Line Testing - การทดสอบปลายสาย

### การทดสอบที่จำเป็นก่อนส่งมอบรถยนต์

```
EOL Test Sequence:
1. Engine Start Test (ติดเครื่อง)
2. Drive Train Check (ทดสอบระบบขับเคลื่อน)
3. Water Leak Test (ทดสอบน้ำรั่ว - Rain Simulation)
4. Rolling Road Test (ทดสอบบนลูกกลิ้ง)
5. Headlight Aim Test (ปรับไฟหน้า)
6. Brake Test (ทดสอบเบรก)
7. Emission Test (วัดไอเสีย)
8. Final Inspection (ตรวจสอบขั้นสุดท้าย)
```

**EOL Water Test Control:**

```
=== LADDER: Water Leak Test (Rain Chamber) ===

Rung 1: Pre-test Check
|--[VEHICLE_IN_CHAMBER]--[DRAIN_CLOSED]--[PUMP_READY]--|
|   +--( PRE_TEST_OK )--|

Rung 2: Water Spray Pattern 1 - Windshield
|--[PRE_TEST_OK]--[OSR]--( START_TEST )--|
|--[START_TEST]--( ZONE_1_SPRAY )--|   // Top nozzles
|--[START_TEST]--( T_ZONE1, EN )--|    // 45 seconds

Rung 3: Water Spray Pattern 2 - Doors
|--[T_ZONE1, DN]--( ZONE_1_SPRAY CLR )--|
|--[T_ZONE1, DN]--( ZONE_2_SPRAY )--|   // Side nozzles
|--[T_ZONE1, DN]--( T_ZONE2, EN )--|    // 60 seconds

Rung 4: Water Spray Pattern 3 - Underbody
|--[T_ZONE2, DN]--( ZONE_2_SPRAY CLR )--|
|--[T_ZONE2, DN]--( ZONE_3_SPRAY )--|   // Bottom nozzles
|--[T_ZONE2, DN]--( T_ZONE3, EN )--|    // 30 seconds

Rung 5: Drain and Inspect
|--[T_ZONE3, DN]--( ALL_SPRAY CLR )--|
|--[T_ZONE3, DN]--( DRAIN_OPEN )--|
|--[T_ZONE3, DN]--( T_DRAIN, EN )--|    // 120 seconds drain time
|--[T_DRAIN, DN]--( READY_INSPECT )--|  // พร้อมให้ Inspector ตรวจ
```

---

## Step 1311: IATF 16949 Requirements - ข้อกำหนดสำหรับอุตสาหกรรมยานยนต์

### ข้อกำหนด IATF 16949 ที่เกี่ยวข้องกับ PLC/Automation

```
IATF 16949 Key Elements for Automation:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
8.5.1  Control Plan (แผนควบคุมกระบวนการ)
8.5.2  Work Instructions (คำสั่งงาน)
8.5.3  Identification & Traceability (ติดตามสินค้า)
8.5.6  Production Change Control (การควบคุมการเปลี่ยนแปลง)
8.6.1  Acceptance Criteria (เกณฑ์การยอมรับ)
8.6.3  Layout Inspection (การตรวจสอบผัง)
9.1.1  Measurement System Analysis (MSA)
9.2    Internal Audit Requirements
```

**Traceability System in PLC:**

```pascal
// Structure สำหรับ Part Traceability
TYPE T_PartTrace :
STRUCT
    VIN_Number      : STRING[17];    // Vehicle Identification Number
    Part_Serial     : STRING[20];    // Part Serial Number
    Station_ID      : INT;           // สถานีที่ผลิต
    Operator_ID     : STRING[10];    // รหัสพนักงาน
    Timestamp       : DATE_AND_TIME; // วันเวลาผลิต
    Shift           : USINT;         // 1=Day, 2=Afternoon, 3=Night
    Process_Params  : ARRAY[1..10] OF REAL;  // Parameter กระบวนการ
    Result          : BOOL;          // Pass=TRUE, Fail=FALSE
    Defect_Code     : STRING[20];    // รหัสข้อบกพร่อง (ถ้ามี)
END_STRUCT
END_TYPE

// บันทึก Traceability Data ทุกชิ้นงาน
FUNCTION FC_RecordTrace
VAR_INPUT
    TraceData : T_PartTrace;
END_VAR
// ส่ง Data ไปยัง MES/ERP System
// ผ่าน OPC UA หรือ Database Direct Connection
END_FUNCTION
```

---

## Step 1312: Complete Body Shop Control System

### ตัวอย่างระบบควบคุม Body Shop สมบูรณ์

**System Overview:**

```
                    [BODY SHOP MASTER PLC]
                    Siemens S7-1500 CPU 1516
                         ↙    ↓    ↘
              [Framing    [Respot   [Underbody
               Station]   Station]  Station]
              CPU 1513   CPU 1513   CPU 1513
               ↙   ↘      ↙  ↘      ↙   ↘
           [Robot] [Robot] ... ... ... [Robot]
           FANUC   FANUC           FANUC
           R2000iC R2000iC         R1000iA
           
           ↓ PROFINET Network ↓
           
           [Production  [Quality   [Maintenance
            Tracking]    System]    System]
            (MES)        (CMM)      (CMMS)
```

**Master PLC Framing Station Program:**

```
=== LADDER: Framing Station Master Sequence ===

Rung 1: Production Order Loading
|--[MES_ORDER_READY]--[/LINE_RUNNING]--|
|   [MOV Source:MES_VIN Dest:CURRENT_VIN]--|
|   [MOV Source:MES_MODEL Dest:CURRENT_MODEL]--|
|   +--( ORDER_LOADED )--|

Rung 2: Fixture Selection
|--[ORDER_LOADED]--|
|   CASE CURRENT_MODEL OF
|     1: [MOV 101 → FIXTURE_PROG]  // Sedan
|     2: [MOV 102 → FIXTURE_PROG]  // SUV
|     3: [MOV 103 → FIXTURE_PROG]  // Pickup
|   END_CASE;
|   +--( FIXTURE_SELECTED )--|

Rung 3: Load Body Panels
|--[FIXTURE_SELECTED]--[CONVEYOR_POSITIONED]--|
|   +--( TRANSFER_START )--|
|
|--[TRANSFER_COMPLETE]--[ALL_PANELS_PRESENT]--|
|   +--( READY_FOR_FRAME )--|

Rung 4: Framing Sequence Start
|--[READY_FOR_FRAME]--[SAFETY_OK]--|
|   [MOV Source:FIXTURE_PROG Dest:ROBOT_PGM_NO]--|
|   +--( CLAMP_SEQUENCE_START )--|

Rung 5: All Clamps Confirmed
|--[CLAMP_1_CONFIRM]--[CLAMP_2_CONFIRM]--[CLAMP_3_CONFIRM]--|
|   ...all 48 clamps confirmed...
|   +--( ALL_CLAMPED )--|
|   +--( ROBOTS_START )--|   // สั่ง Robot ทั้ง 6 ตัวให้เริ่มเชื่อม

Rung 6: Wait for All Robots Complete
|--[ROBOT1_DONE]--[ROBOT2_DONE]--[ROBOT3_DONE]--|
|   [ROBOT4_DONE]--[ROBOT5_DONE]--[ROBOT6_DONE]--|
|   +--( ALL_WELDING_DONE )--|

Rung 7: Release and Advance
|--[ALL_WELDING_DONE]--[QUALITY_CHECK_OK]--|
|   +--( RELEASE_CLAMPS )--|
|--[ALL_CLAMPS_OPEN]--( ADVANCE_CONVEYOR )--|
|--[ADVANCE_DONE]--( CYCLE_COMPLETE )--|

Rung 8: Production Counting
|--[CYCLE_COMPLETE]--[OSR]--|
|   [ADD SHIFT_COUNT, 1, SHIFT_COUNT]--|
|   [ADD DAILY_COUNT, 1, DAILY_COUNT]--|
|   +--( SEND_COUNT_TO_MES )--|
```

---

## Step 1313: Press Shop Safety - ความปลอดภัยในโรงปั๊ม

### Safety Requirements for Press Machines

ตาม ISO/TR 11688 และ ISO 16092-1:

```
Safety Distances สำหรับ Light Curtain:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Ds = K × (Ts + Tc + Tr + Tbm)

Where:
  Ds = Minimum safety distance (mm)
  K  = Hand speed constant (2000 mm/s)
  Ts = System stopping time (ms)
  Tc = Control system response time (ms)
  Tr = Light curtain response time (ms)
  Tbm = Brake monitor time (ms)

ตัวอย่าง:
  K = 2000 mm/s
  Ts = 250 ms (press brake time)
  Tc = 10 ms (PLC scan time)
  Tr = 15 ms (SICK light curtain)
  Tbm = 20 ms

  Ds = 2000 × (0.250 + 0.010 + 0.015 + 0.020)
  Ds = 2000 × 0.295 = 590 mm
  → ติดตั้ง Light Curtain ห่างจากเขตอันตราย อย่างน้อย 590 มม.
```

**Safety PLC Program (SIL 2 Rated):**

```
=== Safety PLC Ladder (Pilz PNOZ, Safety Relay) ===

Rung 1: Dual Channel Safety Gate (Category 4, PLe)
|--[GATE_SW1_CH1]--[GATE_SW1_CH2]--( GATE_1_SAFE )--|
|
| ใช้ 2 Channel ป้องกัน Single Fault → ระดับ Category 4
| Cross monitoring ระหว่าง 2 Channel

Rung 2: Safety Function - Stop when gate opens
|--[/GATE_1_SAFE]--( STO_PRESS )--|   // Safe Torque Off
|--[/GATE_1_SAFE]--( ENGAGE_BRAKE )-|  // ล็อคเบรคทันที

Rung 3: Manual Reset Required (after gate open)
|--[GATE_1_SAFE]--[/RESET_LATCH]--( RESET_LATCH )--|
|--[RESET_PB]--[GATE_1_SAFE]--[RESET_LATCH]--( SYSTEM_OK )--|
|
| ต้องกด Reset ด้วยมือหลังจากประตูถูกเปิด
| ป้องกันการเริ่มเองอัตโนมัติ
```

---

## Step 1314-1320: Advanced Automotive PLC Topics

### Step 1314: Paint Shop Conveyor Control

**Overhead Conveyor (Power and Free) :**

```
Zone 1 (Pretreatment) → Zone 2 (E-Coat) → Zone 3 (Oven) → Zone 4 (Primer)
     Speed: 2.5 m/min     Speed: 2.5 m/min   Speed: Var    Speed: 2.0 m/min

Oven Zone Speed Control:
- เข้า Oven: ลดความเร็วลง 50% เพื่อให้ร้อนทั่ว
- ใน Oven: ความเร็วคงที่ตาม Curing Time
- ออก Oven: เพิ่มความเร็วกลับปกติ
```

### Step 1315: Final Assembly Sequencing

**Assembly Line Sequencing with JIT (Just-In-Time):**

```pascal
// Sequencing Program: จัดลำดับชิ้นส่วนให้ตรงกับรถ
FUNCTION FC_SequenceEngine
VAR_INPUT
    VIN         : STRING[17];
    Engine_Type : INT;   // 1=1.5L, 2=2.0L, 3=2.5T, 4=EV
    Transmission: INT;   // 1=Manual, 2=CVT, 3=6AT
END_VAR

// ส่งข้อมูลไปยัง Engine Sub-Line เพื่อเตรียม Engine
// ล่วงหน้า 3 ชั่วโมงก่อนถึงสถานีติดตั้ง
Engine_Pull_Signal.VIN := VIN;
Engine_Pull_Signal.Engine_Code := Engine_Type;
Engine_Pull_Signal.Scheduled_Time := CURRENT_TIME + T#3h;
SendToEngineSubline(Engine_Pull_Signal);
END_FUNCTION
```

### Step 1316: Vehicle Identification System

**Barcode/RFID Reading in Production:**

```
=== LADDER: VIN Reading and Routing ===

Rung 1: Trigger Barcode Read
|--[VEHICLE_AT_READER]--[OSR]--( BARCODE_TRIGGER )--|

Rung 2: Read Success
|--[BARCODE_READ_OK]--|
|   [MOV Source:BARCODE_DATA Dest:CURRENT_VIN]--|
|   +--( VIN_CONFIRMED )--|

Rung 3: Read Failure - Retry
|--[BARCODE_READ_FAIL]--[RETRY_COUNT < 3]--|
|   [ADD RETRY_COUNT, 1, RETRY_COUNT]--|
|   +--( BARCODE_TRIGGER )--|   // Retry

Rung 4: Cannot Read - Manual Entry
|--[BARCODE_READ_FAIL]--[RETRY_COUNT >= 3]--|
|   +--( REQUEST_MANUAL_ENTRY )--|
|   +--( ALERT_OPERATOR )--|
```

### Step 1317-1320: System Integration

**OPC UA Server for MES Integration:**

```pascal
// OPC UA Node Configuration
// ทำให้ระบบภายนอก (MES, ERP) อ่าน/เขียนข้อมูลจาก PLC ได้

// Node สำหรับ Production Data
OPC_Production.ShiftCount     // จำนวนผลิตรายกะ
OPC_Production.DailyCount     // จำนวนผลิตรายวัน
OPC_Production.CurrentVIN     // VIN รถที่กำลังผลิต
OPC_Production.CurrentModel   // Model รถ
OPC_Production.LineSpeed      // ความเร็วสาย

// Node สำหรับ Quality Data
OPC_Quality.PassCount         // ชิ้นผ่าน QG
OPC_Quality.FailCount         // ชิ้นไม่ผ่าน QG
OPC_Quality.ReworkCount       // ชิ้นรอแก้ไข
OPC_Quality.ScrapCount        // ชิ้น Scrap

// Node สำหรับ Downtime
OPC_Downtime.CurrentReason    // สาเหตุ Downtime ปัจจุบัน
OPC_Downtime.Duration         // ระยะเวลา Downtime นาที
OPC_Downtime.PlannedStop      // หยุดตามแผน (PM, Break)
OPC_Downtime.UnplannedStop    // หยุดนอกแผน (Breakdown)
```

---

## Lab Project 66: Body Shop ANDON & OEE System

### โจทย์: สร้างระบบ ANDON และคำนวณ OEE สำหรับ Body Shop ที่มี 5 สถานี

**ข้อกำหนด:**
1. แต่ละสถานีมีปุ่ม ANDON Yellow (ขอความช่วยเหลือ) และ Red (หยุดสาย)
2. ANDON Board แสดงสถานะทุกสถานีแบบ Real-time
3. ถ้าไม่ตอบสนองภายใน 60 วินาที → เปลี่ยนจาก Yellow เป็น Red อัตโนมัติ
4. คำนวณ OEE รายกะ (ทุก 8 ชั่วโมง)
5. แสดง OEE บน HMI และส่งไป MES ทุก 15 นาที

**ตารางเวลาการผลิต:**
- Scheduled Time: 480 นาที/กะ
- Planned Breaks: 40 นาที (2 พัก × 10 นาที + 20 นาที อาหาร)
- Takt Time: 120 วินาที/คัน
- Target OEE: 78%

**สเปก Hardware:**
- Main PLC: Siemens S7-1500 CPU 1515
- I/O: ET200SP Distributed I/O (ที่แต่ละสถานี)
- HMI: Siemens TP1500 Comfort 15"
- Network: PROFINET
- Database: SQL Server ผ่าน OPC UA

**เฉลย (โครงสร้างหลัก):**

```
DB_AndonSystem:
  Station[1..5].YellowActive : BOOL
  Station[1..5].RedActive    : BOOL
  Station[1..5].ResponseTime : TIME
  Station[1..5].DowntimeAcc  : TIME

OB1 Main Program:
  CALL FC_AndonMonitor  // ตรวจสอบสถานะ ANDON
  CALL FB_OEE_Calc      // คำนวณ OEE
  CALL FC_MES_Send      // ส่งข้อมูลไป MES

FC_AndonMonitor:
  // Loop ทุกสถานี
  FOR i := 1 TO 5 DO
    IF Station[i].YellowActive AND NOT Station[i].ResponseTimer.Q THEN
      // ยังไม่เกิน 60 วินาที - คงสีเหลือง
    ELSIF Station[i].YellowActive AND Station[i].ResponseTimer.Q THEN
      // เกิน 60 วินาที - เปลี่ยนเป็นสีแดง หยุดสาย
      Station[i].RedActive := TRUE;
      Line_Stop := TRUE;
    END_IF;
  END_FOR;
```

---

## สรุป Part 66

ใน Part นี้เราได้เรียนรู้:
- การควบคุม Press Shop และ Welding Cell
- ระบบ ANDON สำหรับสื่อสารปัญหา
- Poka-Yoke เพื่อป้องกันความผิดพลาด
- การคำนวณ OEE สำหรับวัดประสิทธิภาพ
- ระบบควบคุม Paint Shop
- End-of-Line Testing
- ข้อกำหนด IATF 16949

**ทักษะสำคัญที่ได้:**
- Sequence Control ที่ซับซ้อน
- Quality Data Integration กับ MES
- Safety Function สำหรับงาน Press
- Traceability ตาม IATF 16949

---

### 🔗 Navigation
- [← Part 65: Advanced Process Control](../part61-80/part65_advanced_process_control.md)
- [→ Part 67: Printing & Packaging](part67_printing_packaging.md)
- [📚 สารบัญหลัก](../README.md)
