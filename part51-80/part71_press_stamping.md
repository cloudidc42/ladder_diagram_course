# Part 71: Press & Stamping Control (Steps 1401-1420)

## การเขียนโปรแกรม PLC สำหรับเครื่องปั๊มและงาน Stamping

---

### 🔗 Navigation
- [← Part 70: Crane & Hoist](part70_crane_hoist.md)
- [→ Part 72: Mining Automation](part72_mining_automation.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: อุตสาหกรรมการปั๊มและ Stamping

งาน Press และ Stamping เป็นกระบวนการขึ้นรูปโลหะที่พบมากในอุตสาหกรรมยานยนต์ อุปกรณ์ไฟฟ้า และ Metal Fabrication ความปลอดภัยเป็นเรื่องสำคัญอย่างยิ่ง เพราะเครื่องปั๊มมีแรงกดมหาศาลและเคลื่อนที่รวดเร็ว

### ประเภทของ Press

```
1. Mechanical Press (เครื่องปั๊มกล)
   - ใช้ Flywheel + Clutch/Brake
   - ความเร็วสูง: 40-400 SPM (Strokes Per Minute)
   - Force: คงที่ตาม Stroke Position

2. Hydraulic Press (เครื่องปั๊มไฮดรอลิก)
   - ใช้ Hydraulic Cylinder
   - ความเร็วต่ำกว่า: 2-30 SPM
   - Force: ปรับได้ตลอด Stroke

3. Servo Press (เครื่องปั๊ม Servo)
   - ใช้ Servo Motor แทน Flywheel
   - ยืดหยุ่นสูง: ปรับ Speed, Force, Position ได้อิสระ
   - ราคาสูงแต่คุณภาพงานดีที่สุด
```

---

## Step 1401: Mechanical Press Safety - ความปลอดภัย Clutch/Brake

### Clutch-Brake System

```
Mechanical Press Operation:
1. Motor หมุน Flywheel เก็บ Kinetic Energy
2. เมื่อสั่ง Stroke → Clutch Engage → Flywheel ส่งพลังงาน
3. Ram เคลื่อนลงปั๊ม → TDC (Top Dead Center) → BDC → กลับ TDC
4. Clutch Disengage → Brake Apply → หยุดที่ TDC

Safety Requirements (ตาม ISO 16092-1):
  ต้องหยุดที่ TDC เท่านั้น!
  ห้ามหยุดกลางทาง (Partial Stroke ≠ Safe!)
  Two-Hand Control: ต้องกด 2 มือพร้อมกัน (Anti-Tie-Down)
  Brake Performance Monitoring: ตรวจสอบว่า Brake ทำงานใน Time เฉพาะ
```

**Press Safety PLC Program (Safety PLC Pilz/Sick):**

```
=== LADDER: Press Safety Circuit (Safety PLC) ===

Rung 1: Two-Hand Control (Anti-Tie-Down)
| ต้องกดทั้ง 2 ปุ่ม ภายใน 500ms (ตาม ISO 13851)
|
|--[TH_L_BTN]--[OSR]--( L_TIMESTAMP := NOW )--|   // บันทึกเวลา กด ซ้าย
|--[TH_R_BTN]--[OSR]--( R_TIMESTAMP := NOW )--|   // บันทึกเวลา กด ขวา
|
|--[TH_L_BTN]--[TH_R_BTN]--|
|   [ABS(L_TIMESTAMP - R_TIMESTAMP) < 500ms]--|
|   +--( TWO_HAND_OK )--|

Rung 2: Two-Hand Anti-Repeat
|--[TWO_HAND_OK]--[/STROKE_DONE]--( INITIATE_STROKE )--|
|
| หลังจากกด 2 มือ → stroke เริ่ม → ต้องปล่อยและกดใหม่
| ป้องกันการกดค้างทำให้ stroke ต่อเนื่อง

Rung 3: Brake Performance Monitor
|--[CLUTCH_DISENGAGE]--[OSR]--|
|   [CAP BRAKE_START_TIME := NOW]--|
|
|--[TDC_SENSOR]--[OSR]--|
|   [CAP BRAKE_STOP_TIME := NOW]--|
|   [SUB BRAKE_STOP_TIME, BRAKE_START_TIME, STOP_TIME]--|
|
|--[STOP_TIME > BRAKE_TIME_LIMIT]--|   // เกิน limit = เบรกเสื่อม
|   +--( BRAKE_FAULT )--|
|   +--( LOCK_PRESS )--|   // ล็อคปั๊ม ห้ามใช้จนกว่าซ่อม

Rung 4: Light Curtain Integration (SIL 2)
|--[/LC_CLEAR]--[STROKE_IN_PROGRESS]--|
|   // มีอะไรผ่าน Light Curtain ขณะปั๊ม
|   +--( EMERGENCY_CLUTCH_OFF )--|   // ปลด Clutch ทันที
|   +--( EMERGENCY_BRAKE_ON )--|     // เบรกทันที
|   +--( STROKE_ABORT )--|
```

---

## Step 1402: Stroke Counting and Die Protection

### Die Protection System

Die Protection ตรวจสอบให้แน่ใจว่าแม่พิมพ์ไม่เสียหาย:

```
Die Protection Sensors:
1. Material Feed Sensor: ชิ้นงานเข้ามาครบหรือไม่
2. Part In-Die Sensor: มีชิ้นงานใน Die ก่อนปั๊มหรือไม่
3. Part Out-of-Die Sensor: ชิ้นงานออกหลังปั๊มหรือไม่
4. Misfeed Detector: วัสดุป้อนครบหรือไม่
5. Part Buckle Detector: วัสดุงอหรือเสียรูปทรงหรือไม่
```

**Die Protection Ladder:**

```
=== LADDER: Die Protection ===

Rung 1: Material In-Die Confirmation
|--[STROKE_APPROACHING]--|
|   // ก่อน Ram ลงปั๊ม ตรวจสอบว่า Material อยู่ใน Die
|   [MATERIAL_SENSOR_OK]--|
|   +--( DIE_CLEAR_FOR_STROKE )--|

|--[STROKE_APPROACHING]--|
|   [/MATERIAL_SENSOR_OK]--|
|   +--( MISFEED_FAULT )--|
|   +--( PREVENT_STROKE )--|   // ห้ามปั๊ม!

Rung 2: Part Ejection Verification
|--[STROKE_COMPLETE]--|
|   // หลัง Stroke เสร็จ ตรวจสอบว่าชิ้นงานออกจาก Die
|   [T_EJECT_CHECK, EN]--|   // รอ 200ms
|
|--[T_EJECT_CHECK, DN]--|
|   [/PART_IN_DIE_SENSOR]--|   // ไม่มีชิ้นงานใน Die แล้ว
|   +--( EJECT_OK )--|

|--[T_EJECT_CHECK, DN]--|
|   [PART_IN_DIE_SENSOR]--|   // ยังมีชิ้นงาน!
|   +--( PART_STUCK_FAULT )--|   // ชิ้นงานติดใน Die
|   +--( LOCK_PRESS )--|

Rung 3: Stroke Counter
|--[TDC_SENSOR]--[OSR]--|
|   [ADD STROKE_COUNT, 1, STROKE_COUNT]--|
|   [ADD DIE_STROKE_COUNT, 1, DIE_STROKE_COUNT]--|
|
| DIE_STROKE_COUNT นับสำหรับแม่พิมพ์นี้
| ถ้าถึง PM_INTERVAL → แจ้ง PM แม่พิมพ์

Rung 4: Die Life Monitor
|--[DIE_STROKE_COUNT >= PM_INTERVAL]--( DIE_PM_DUE )--|
|--[DIE_STROKE_COUNT >= MAX_LIFE]--( DIE_EOL_ALARM )--|
|   // แม่พิมพ์ใช้ครบอายุ ต้องเปลี่ยน
```

---

## Step 1403: Hydraulic Press Control - การควบคุม Press ไฮดรอลิก

### Hydraulic System Components

```
Hydraulic Circuit:
Pump → Pressure Relief Valve → Proportional Valve → Cylinder
         ↓                          ↓
    Safety Pressure          Speed Control
    (Max Press)               (Flow Control)

Proportional Hydraulic Valve:
  0-10V หรือ 4-20mA จาก PLC
  0V/4mA = หยุด
  5V/12mA = กลาง (หยุด/ช้า)
  10V/20mA = ความเร็วสูงสุด
```

**400-Ton Hydraulic Press Control:**

```pascal
// Hydraulic Press Control - Main Sequence
FUNCTION_BLOCK FB_Hydraulic_Press
VAR_INPUT
    Start_Cycle    : BOOL;
    E_Stop_OK      : BOOL;
    Safety_Gate_OK : BOOL;
    Ram_Position   : REAL;    // mm จาก Top
    System_Pressure: REAL;    // bar
    Tonnage_Reading: REAL;    // tons
END_VAR
VAR_OUTPUT
    Prop_Valve_SP  : REAL;    // mA 4-20
    Direction_Down : BOOL;    // Cylinder Down
    Direction_Up   : BOOL;    // Cylinder Up
    Clamp_On       : BOOL;    // Blank Holder
END_VAR
VAR
    Press_State    : INT;
    Ram_Target     : REAL;    // Target Position mm
    Press_Target   : REAL;    // Target Tonnage tons
    Dwell_Timer    : TON;
END_VAR

CASE Press_State OF
    0: // IDLE - Wait for Start
        Prop_Valve_SP := 12.0;  // 4-20mA: 12=Stop
        Direction_Down := FALSE;
        IF Start_Cycle AND E_Stop_OK AND Safety_Gate_OK THEN
            Press_State := 1;
        END_IF;
    
    1: // RAPID APPROACH - Move Down Fast
        Direction_Down := TRUE;
        Prop_Valve_SP := 20.0;   // Full Speed Down
        IF Ram_Position > 50.0 THEN  // 50mm ก่อนถึง Material
            Press_State := 2;
        END_IF;
    
    2: // PRESS APPROACH - Slow Down
        Prop_Valve_SP := 15.0;   // 50% Speed
        IF Ram_Position > 100.0 THEN  // ถึงจุดเริ่ม Press
            Clamp_On := TRUE;  // หนีบ Blank
            Press_State := 3;
        END_IF;
    
    3: // PRESS STROKE - Force Control
        // สลับจาก Position → Pressure Control
        IF Tonnage_Reading < Press_Target THEN
            // เพิ่มแรง
            Prop_Valve_SP := 18.0;
        ELSIF Tonnage_Reading > Press_Target * 1.05 THEN
            // ลดแรง (อย่าให้เกิน 5%)
            Prop_Valve_SP := 14.0;
        ELSE
            // Maintain pressure
            Prop_Valve_SP := 16.0;
        END_IF;
        
        // Bottom Dead Center Detection
        IF Ram_Position > BDC_Position - 1.0 THEN  // ±1mm
            Dwell_Timer(IN := TRUE, PT := T#500ms);
            Press_State := 4;
        END_IF;
    
    4: // DWELL - Hold at bottom
        IF Dwell_Timer.Q THEN
            Press_State := 5;
        END_IF;
    
    5: // RETURN STROKE - Move Up
        Direction_Down := FALSE;
        Direction_Up := TRUE;
        Prop_Valve_SP := 18.0;  // Fast Up
        Clamp_On := FALSE;
        IF Ram_Position < 10.0 THEN  // กลับ Top
            Press_State := 6;
        END_IF;
    
    6: // CYCLE COMPLETE
        Direction_Up := FALSE;
        Prop_Valve_SP := 12.0;
        // ออก Part, Advance Feed
        Press_State := 0;
END_CASE;
END_FUNCTION_BLOCK
```

---

## Step 1404: Tonnage Monitoring System

### การวัดแรงกดจริงใน Press Stroke

```
Tonnage Measurement Methods:
1. Strain Gauge on Frame: วัดการยืดตัวของโครงเครื่อง
2. Hydraulic Pressure Transducer: P × Area = Force
3. Piezoelectric Load Cell: ไวมาก, แม่นยำสูง

Tonnage Signature Analysis:
  สร้างกราฟ Force vs Position สำหรับแต่ละ Part Type
  Save Signature ปกติ → Compare กับ Stroke จริง
  ความแตกต่างมาก = Die Problem, Material Problem
```

**Tonnage Monitoring Program:**

```pascal
// Tonnage Signature Recording
FUNCTION_BLOCK FB_TonnageMonitor
VAR_INPUT
    Ram_Position   : REAL;    // mm (0=TDC, 250=BDC)
    Tonnage_PV     : REAL;    // tons
    In_Stroke      : BOOL;    // TRUE during down stroke
END_VAR
VAR_OUTPUT
    Signature_Saved : BOOL;
    Signature_Match : BOOL;
    Deviation_pct   : REAL;
END_VAR
VAR
    Signature_Points : ARRAY[0..99] OF REAL;  // 100 Points along stroke
    Reference_Sig    : ARRAY[0..99] OF REAL;  // Reference (Golden) Signature
    Sample_Index     : INT;
    Max_Deviation    : REAL;
END_VAR

IF In_Stroke THEN
    // Sample Tonnage ที่ทุกๆ 2.5mm ตลอด 250mm Stroke
    Sample_Index := REAL_TO_INT(Ram_Position / 2.5);
    IF Sample_Index >= 0 AND Sample_Index < 100 THEN
        Signature_Points[Sample_Index] := Tonnage_PV;
    END_IF;
END_IF;

// หลัง Stroke เสร็จ - Compare กับ Reference
IF NOT In_Stroke AND Signature_Saved THEN
    Max_Deviation := 0.0;
    FOR i := 0 TO 99 DO
        IF Reference_Sig[i] > 0 THEN
            DEV := ABS(Signature_Points[i] - Reference_Sig[i]) / Reference_Sig[i] * 100.0;
            IF DEV > Max_Deviation THEN
                Max_Deviation := DEV;
            END_IF;
        END_IF;
    END_FOR;
    
    Deviation_pct := Max_Deviation;
    
    // Pass/Fail ตาม Tolerance
    IF Max_Deviation < 15.0 THEN   // < 15% = Pass
        Signature_Match := TRUE;
    ELSE
        Signature_Match := FALSE;  // ผิดปกติ - Die เสีย? Material เสีย?
    END_IF;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1405: Transfer Press Synchronization

### Multi-Station Transfer Press

```
Transfer Press มีหลาย Station:
  Station 1 (Draw)  → Station 2 (Redraw) → Station 3 (Pierce) → 
  Station 4 (Flange) → Station 5 (Trim) → Station 6 (Out)

ทุก Station ทำงานพร้อมกัน 1 Stroke = 6 ชิ้น
Transfer Bar: โอนชิ้นงานไปทุก Station พร้อมกัน

Synchronization Requirements:
  Transfer Bar ต้องเข้า/ออก พร้อมกับ Ram
  ไม่ซ้อน = ชนกัน!
  Cam Profile: ตำแหน่ง Transfer Bar ตาม Ram Position (Degree)
```

**Transfer Synchronization:**

```
=== LADDER: Transfer Press Sync ===

Rung 1: Ram Angle Measurement
| ใช้ Resolver หรือ Absolute Encoder ที่ Crankshaft
| 0° = TDC, 180° = BDC
| ส่งค่า Angle ไปยัง Transfer Control

Rung 2: Transfer Move Forward (Advance)
| ตาม Cam Profile: Transfer เริ่มเดินหน้าที่ Ram = 200° (หลัง BDC)
|--[RAM_ANGLE >= 200]--[RAM_ANGLE < 280]--|
|   +--( TRANSFER_FWD )--|   // Transfer Bar เดินหน้า

Rung 3: Transfer Move Backward (Return)
| Return เมื่อ Ram ขึ้นแล้ว
|--[RAM_ANGLE >= 300]--[RAM_ANGLE < 360]--|
|   +--( TRANSFER_REV )--|   // Transfer Bar กลับ

Rung 4: Interlock - Ram must be clear before Transfer moves
|--[TRANSFER_FWD_CMD]--[RAM_ANGLE < 180]--|
|   // Ram ยังอยู่ขาลง แต่ Transfer พยายามเดิน
|   +--( TRANSFER_INTERLOCK_FAULT )--|   // อันตราย!
|   +--( EMERGENCY_STOP_PRESS )--|
```

---

## Step 1406: Progressive Die Control

### Progressive Stamping Die

```
Progressive Die มีหลาย Stage ใน Die เดียว:
  Strip Feed → Stage 1 (Pilot + Pierce) → Stage 2 (Draw) → 
  Stage 3 (Trim) → Stage 4 (Form) → Stage 5 (Cut-off) → Finished Part

ทุก Stroke:
  - วัสดุเดินหน้า 1 Step (Feed Length)
  - ทุก Stage ทำงานพร้อมกัน
  - ออก 1 ชิ้นสำเร็จ

ควบคุม:
  - Feed Length ต้องแม่นยำ ±0.1mm
  - Pilot Pin ล็อคตำแหน่งก่อนปั๊ม
```

**Progressive Feed Control:**

```pascal
// Servo Roll Feed for Progressive Die
FUNCTION_BLOCK FB_ServoFeed
VAR_INPUT
    Stroke_Complete : BOOL;   // 1 Stroke เสร็จ
    Feed_Length     : REAL;   // mm ระยะป้อน (จาก Recipe)
    Pilot_Detect    : BOOL;   // TRUE = Pilot Pin ในรู
END_VAR
VAR_OUTPUT
    Feed_Start      : BOOL;   // เริ่มป้อน
    Feed_Length_SP  : REAL;   // mm ส่งไป Servo Drive
    Feed_Complete   : BOOL;   // ป้อนเสร็จ
    Feed_Error      : BOOL;   // ป้อนผิดพลาด
END_VAR
VAR
    Feed_Pos_FB     : REAL;   // mm ตำแหน่งจริงจาก Encoder
    Feed_Start_Pos  : REAL;   // ตำแหน่งเริ่มต้น
    Position_Error  : REAL;   // mm ความผิดพลาด
END_VAR

// เริ่มป้อนหลัง Stroke Complete
IF Stroke_Complete THEN
    Feed_Start_Pos := Feed_Pos_FB;
    Feed_Length_SP := Feed_Start_Pos + Feed_Length;
    Feed_Start := TRUE;
END_IF;

// ตรวจสอบว่าป้อนถึงเป้าหมาย
IF Feed_Start THEN
    Position_Error := ABS(Feed_Pos_FB - Feed_Length_SP);
    IF Position_Error < 0.05 THEN  // ±0.05mm = 50 micron
        Feed_Complete := TRUE;
        Feed_Start := FALSE;
        Feed_Error := FALSE;
    END_IF;
END_IF;

// ตรวจสอบ Pilot Pin (ล็อคตำแหน่งก่อนปั๊ม)
IF NOT Pilot_Detect AND Feed_Complete THEN
    // Feed เสร็จแล้วแต่ Pilot ไม่ปรากฏ = ผิดพลาด!
    Feed_Error := TRUE;
    // ห้าม Stroke!
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1407-1420: Advanced Topics

### Step 1407: Press Network (Link Connect - Komatsu/IHI)

**Press Line Communication:**

```
Press Line Network (ตัวอย่าง Komatsu Servo Press):
  Host PLC ↔ Press 1 ↔ Press 2 ↔ Press 3 ↔ Press 4
  
  Data Shared:
  - Crank Angle (ทุก Press รู้ตำแหน่ง Press ทุกตัว)
  - Transfer Timing
  - Alarm Status
  - Production Count
  - Speed Command

  Protocol: Proprietary High-Speed Network
  Update Rate: < 1ms (ต้องเร็วมากสำหรับ Synchronization)
```

### Step 1408: Servo Press Programming

**Servo Press Motion Profile:**

```
Servo Press Advantages:
  - ปรับ Speed ตลอด Stroke ได้อิสระ
  - ปรับ Tonnage ตามตำแหน่งได้
  - Pendulum Motion: ที่ BDC ไม่ต้องลงไปสุด สั้นกว่า = เร็วกว่า

Motion Profile Examples:

Profile A (Standard):
  100mm/s  ─────────────────────
  (Fast)                              ↑ Return Fast

Profile B (Quality Forming):
  100mm/s  ──────┐
                  │ ← ลดความเร็วขณะ Contact
  10mm/s         └───────────────
  (Contact)     BDC

Profile C (Deep Draw - Reduce Material Thinning):
  100mm/s ──────┐
  20mm/s        └───────────┐  ← ช้าในช่วง Draw
  5mm/s                     └──── (ช้ามากที่ BDC)
```

```pascal
// Servo Press Motion Profile Definition
TYPE T_MotionProfile :
STRUCT
    Profile_ID    : INT;
    Profile_Name  : STRING[20];
    
    // Approach Phase
    Approach_Speed: REAL;    // mm/s
    Contact_Pos   : REAL;    // mm จาก TDC ที่เริ่ม Contact
    
    // Forming Phase
    Form_Speed_1  : REAL;    // mm/s ช่วงแรก
    Form_Speed_2  : REAL;    // mm/s ช่วงสอง (ช้าลง)
    Form_Switch_Pos: REAL;   // mm ตำแหน่งสลับ Speed
    
    // BDC Phase
    BDC_Pos       : REAL;    // mm ตำแหน่งล่างสุด
    Dwell_Time    : REAL;    // ms หยุดที่ BDC
    
    // Return Phase
    Return_Speed  : REAL;    // mm/s ความเร็วขึ้น
    
    // Press Parameters
    Blank_Force   : REAL;    // ton แรงหนีบ
    Max_Tonnage   : REAL;    // ton แรงปั๊มสูงสุด
END_STRUCT
END_TYPE
```

### Step 1409: Complete Hydraulic Press Control System

**Full System Architecture:**

```
400-Ton Hydraulic Press Control:
  
  Main PLC: Siemens S7-1515SP Safety
  HMI: Siemens TP2200 Comfort 22"
  Safety: SICK safeguard i800 Light Curtain
  
  Hydraulic:
    Main Pump: 90kW Servo-Pump
    Proportional Valve: Bosch Rexroth
    Pressure Transducer: 0-400 bar
    Linear Position: Balluff BML magnetostrictive 0-500mm
  
  I/O Summary:
    Digital Inputs: 64 (Limit Switches, Buttons, Safety)
    Digital Outputs: 32 (Solenoids, Signals)
    Analog Inputs: 16 (Pressure, Force, Position, Temperature)
    Analog Outputs: 8 (Proportional Valve, Speed Commands)
```

---

## Lab Project 71: 400-Ton Hydraulic Press System

### โจทย์: เขียน PLC Program สำหรับ 400-Ton Hydraulic Deep Draw Press

**ข้อมูลเครื่อง:**
- Maximum Force: 400 tons (4,000 kN)
- Blank Holder Force: 80 tons (800 kN)
- Ram Stroke: 500mm
- Max Speed Down: 200mm/s (Rapid Approach)
- Press Speed: 10-50mm/s (Adjustable)
- Return Speed: 300mm/s

**ข้อมูล Part:**
- Material: DC04 (Deep Draw Steel) t=0.8mm
- Part: Automotive Fuel Tank Inner Panel
- Draw Depth: 250mm
- Blank Holder Pressure: 200 bar

**ข้อกำหนดการเขียน Program:**
1. Sequence Control (State Machine 10 States)
2. Tonnage Signature Recording (100 Points)
3. Die Protection (Material, Part-in-Die, Part-out-of-Die)
4. Safety Interlock (Light Curtain, Two-Hand, E-Stop)
5. Recipe Management (10 Part Types)
6. Maintenance Mode สำหรับ Die Change

**Address Table (Siemens S7-1500):**

| Signal | Type | Siemens Address | Description |
|--------|------|----------------|-------------|
| PROP_VALVE_DN | AO | QW256 | Proportional Valve Down |
| PROP_VALVE_UP | AO | QW258 | Proportional Valve Up |
| RAM_POSITION | AI | IW256 | Ram Position mm |
| MAIN_PRESSURE | AI | IW258 | System Pressure bar |
| TONNAGE | AI | IW260 | Tonnage ton |
| BH_PRESSURE | AI | IW262 | Blank Holder bar |
| START_BTN_L | DI | I0.0 | Left Start Button |
| START_BTN_R | DI | I0.1 | Right Start Button |
| LC_CLEAR | DI (Safety) | I_Safe0.0 | Light Curtain Clear |

---

## สรุป Part 71

ใน Part นี้เราได้เรียนรู้:
- Mechanical Press Safety (Clutch/Brake, Two-Hand Control)
- Brake Performance Monitoring
- Die Protection System
- Hydraulic Press State Machine
- Tonnage Signature Analysis
- Transfer Press Synchronization
- Progressive Die Control
- Servo Press Motion Profiles

**ทักษะสำคัญ:**
- Safety Circuit Design ตาม ISO 16092
- Force/Position Control Transition
- Die Protection ป้องกันแม่พิมพ์เสียหาย
- Synchronization ระหว่าง Press และ Transfer

---

### 🔗 Navigation
- [← Part 70: Crane & Hoist](part70_crane_hoist.md)
- [→ Part 72: Mining Automation](part72_mining_automation.md)
- [📚 สารบัญหลัก](../README.md)
