# Part 67: Printing & Packaging Automation (Steps 1321-1340)

## การเขียนโปรแกรม PLC สำหรับระบบพิมพ์และบรรจุภัณฑ์

---

### 🔗 Navigation
- [← Part 66: Automotive Automation](part66_automotive_automation.md)
- [→ Part 68: Water Treatment](part68_water_treatment.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: อุตสาหกรรมพิมพ์และบรรจุภัณฑ์

อุตสาหกรรมพิมพ์และบรรจุภัณฑ์มีลักษณะเฉพาะคือความเร็วสูง ต้องการความแม่นยำระดับ ±0.1mm และทำงานต่อเนื่อง 24 ชั่วโมง ระบบ PLC ต้องจัดการกับ:

- **Web Speed**: 200-600 ม./นาที สำหรับ Flexographic Printing
- **Registration**: ความแม่นยำในการพิมพ์หลายสี ±0.1mm
- **Tension Control**: ควบคุมแรงตึงกระดาษ/ฟิล์ม
- **Color Matching**: จับคู่สีกับ Pantone Standard
- **Throughput**: 100,000-500,000 units/ชั่วโมง

---

## Step 1321: Web Tension Control - การควบคุมแรงตึงวัสดุ

### หลักการ Web Tension

```
Unwind Roll → Infeed Nip → Print Stations → Outfeed Nip → Rewind Roll
     ↑              ↑            ↑               ↑             ↑
  Brake         Draw Roll    Draw Roll       Draw Roll      Motor
  Control       (Slave)      (Slave)         (Slave)       (Master)

Tension Zone:
Zone 1: Unwind → Infeed (Tension = T1)
Zone 2: Between Print Stations (Tension = T2)
Zone 3: Outfeed → Rewind (Tension = T3)

Tension ใน Zone 2 มักถูกกำหนดให้มากกว่า Zone 1 เพื่อรักษาการพิมพ์
```

**PLC Program สำหรับ Tension Control:**

```
=== LADDER: Web Tension Control Zone 1 ===

Rung 1: Load Cell Reading
| Load Cell → 4-20mA Analog Input → AI Module
| Tension_N = (AI_Raw - 4.0) / (20.0 - 4.0) × Tension_Max
| ตัวอย่าง: Tension_Max = 500 N, AI_Raw = 12 mA
| Tension = (12-4)/(20-4) × 500 = 250 N

Rung 2: Tension PID Control
|--[WEB_RUNNING]--( TC_ENABLE )--|
|
| PID Controller: TC_PID
| SP: TENSION_SP (Setpoint จาก Recipe)
| PV: TENSION_PV (จาก Load Cell)
| CV: BRAKE_TORQUE (0-100% แรงเบรก)
|
| Typical PID Tuning for Tension:
|   Kp = 2.5 (Proportional)
|   Ki = 0.8 (Integral - ต้องมี I เพื่อ offset)
|   Kd = 0.05 (Derivative - ระวัง Noise)

Rung 3: Roll Diameter Compensation
| เมื่อ Roll คลี่ออก → Diameter ลดลง → ต้องชดเชย Torque
| 
| Dancer_Pos → Filter → Diameter_Calc → Torque_FF
|
| Diameter_Calc:
|   D_current = D_initial - (Web_Speed × Time × Thickness × 2)
|   
| Torque_Feedforward:
|   FF = Tension_SP × D_current / 2  [Nm]
```

**Siemens S7-1500 SCL Code:**

```pascal
// Tension Control with Diameter Compensation
FUNCTION_BLOCK FB_TensionControl
VAR_INPUT
    Web_Running     : BOOL;
    Tension_SP      : REAL;    // N - Setpoint แรงตึง
    Tension_PV      : REAL;    // N - ค่าจริงจาก Load Cell
    Web_Speed       : REAL;    // m/min ความเร็ว
    Roll_Diameter   : REAL;    // mm เส้นผ่านศูนย์กลาง
    Material_Thickness: REAL;  // mm ความหนาวัสดุ
END_VAR
VAR_OUTPUT
    Brake_Output    : REAL;    // 0-100% แรงเบรก
END_VAR
VAR
    PID_Error       : REAL;
    PID_Integral    : REAL;
    PID_Output      : REAL;
    Torque_FF       : REAL;   // Feedforward Torque
    
    // PID Gains
    Kp : REAL := 2.5;
    Ki : REAL := 0.8;
    Kd : REAL := 0.05;
    
    Prev_Error : REAL;
    Sample_Time: REAL := 0.01;  // 10ms scan
END_VAR

IF Web_Running THEN
    // คำนวณ Feedforward จาก Torque ที่ต้องการ
    Torque_FF := Tension_SP * (Roll_Diameter / 1000.0) / 2.0;  // Nm
    
    // PID Calculation
    PID_Error := Tension_SP - Tension_PV;
    PID_Integral := PID_Integral + (PID_Error * Sample_Time);
    
    // Anti-Windup
    PID_Integral := LIMIT(-50.0, PID_Integral, 50.0);
    
    PID_Output := (Kp * PID_Error) +
                  (Ki * PID_Integral) +
                  (Kd * (PID_Error - Prev_Error) / Sample_Time);
    
    // Combine FF + PID
    Brake_Output := LIMIT(0.0, Torque_FF + PID_Output, 100.0);
    
    Prev_Error := PID_Error;
ELSE
    Brake_Output := 0.0;
    PID_Integral := 0.0;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1322: Registration Control - การควบคุมการลงทะเบียนตำแหน่ง

### Color-to-Color Registration

```
Registration Error = Actual Mark Position - Target Mark Position

เครื่องหมาย Register (Eye Mark):
┌─────────────────────────────────────────────┐
│  ▓ Print 1   ▓ Print 2   ▓ Print 3 (ref)   │
│  (Cyan)      (Magenta)   (Yellow)           │
│                                             │
│  Scanner 1   Scanner 2   Scanner 3(Master)  │
└─────────────────────────────────────────────┘

Color 3 = Master (Yellow) → Reference
Color 1, 2 ปรับให้ตรงกับ Color 3
```

**Registration PLC Logic:**

```
=== LADDER: Color Registration Control ===

Rung 1: Mark Detection
|--[REG_SENSOR_1]--[OSR]--|
|   [CAP TIM_1 := T_COUNTER]--|   // Capture timestamp ที่ตรวจจับ Mark
|   +--( MARK_1_DETECTED )--|

Rung 2: Calculate Registration Error
|--[MARK_1_DETECTED]--[MARK_3_DETECTED]--|
|   [SUB TIM_1, TIM_3, ERROR_TICKS_1]--|
|   // Error ในหน่วย Clock Ticks
|   // แปลงเป็น mm: Error_mm = Error_Ticks × (Speed / Encoder_Resolution)

Rung 3: Apply Correction
|--[NEW_ERROR_CALC]--|
|   [MUL ERROR_MM_1, REG_GAIN_1, CORRECTION_1]--|
|   // Correction = Error × Gain (ปกติ 0.3-0.8)
|   
|   // ส่ง Correction ไปที่ Servo Drive ของ Print Station 1
|   [MOV CORRECTION_1 → SERVO_1_PHASE_OFFSET]--|
```

---

## Step 1323: Servo-Controlled Cutoff - การตัดด้วย Servo

### Flying Cutoff System

```
Web Running → Cut Register Mark → Servo Sync → Cut → Return

Timing Sequence:
t=0: ตรวจจับ Cut Mark
t=5ms: คำนวณ Sync Position
t=8ms: เริ่ม Servo Acceleration
t=15ms: Servo Speed = Web Speed (Synchronized!)
t=16ms: Cut!
t=17ms: Servo Deceleration
t=25ms: Servo กลับ Home Position
t=0: รอ Cut Mark ถัดไป
```

**Servo Cutoff Program (Siemens with Servo Library):**

```pascal
// Flying Cutoff Control
FUNCTION_BLOCK FB_FlyingCutoff
VAR_INPUT
    WebSpeed       : REAL;    // m/min
    CutLength      : REAL;    // mm ความยาวที่ต้องการตัด
    CutMark_Detect : BOOL;    // Rising edge = ตรวจจับ Mark
END_VAR
VAR_OUTPUT
    Cut_Solenoid   : BOOL;    // TRUE = ตัด
    Error_mm       : REAL;    // ความผิดพลาดในการตัด
END_VAR
VAR
    SyncRequired   : BOOL;
    SyncPosition   : REAL;   // ตำแหน่งที่ต้อง Sync กับ Web
    CutPosition    : REAL;   // ตำแหน่งที่ตัด (หลัง Sync)
    CutTimer       : TON;    // Timer สำหรับ Cut Pulse
    
    WebSpeed_mps   : REAL;   // m/s
    CutToSync_Dist : REAL;   // ระยะจาก Mark ถึงจุด Sync
END_VAR

// แปลงความเร็ว
WebSpeed_mps := WebSpeed / 60.0;

// เมื่อตรวจจับ Cut Mark
IF CutMark_Detect THEN
    // คำนวณ Position ที่ต้องตัด
    // Web เคลื่อนที่ระหว่าง Detection Delay
    CutPosition := WebSpeed_mps * 0.015;  // 15ms delay
    
    // เริ่ม Sync Sequence
    SyncRequired := TRUE;
END_IF;

// Sync กับ Web Speed แล้วตัด
IF SyncRequired THEN
    // Set Servo to match web speed
    Servo_Speed_SP := WebSpeed_mps * 1000.0;  // mm/s
    
    // เมื่อ Servo Speed ตรงกับ Web (±0.5%)
    IF ABS(Servo_Speed_FB - Servo_Speed_SP) < (Servo_Speed_SP * 0.005) THEN
        Cut_Solenoid := TRUE;     // Fire Cut
        CutTimer(IN:=TRUE, PT:=T#20ms);
        IF CutTimer.Q THEN
            Cut_Solenoid := FALSE;
            SyncRequired := FALSE;
        END_IF;
    END_IF;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1324: Label Printing and Applying

### Print and Apply System

```
Label Stock (Roll) → Thermal Printer → Peel Bar → Applicator → Product
                         ↑                              ↑
                    Print Command                  Apply Trigger
                    from PLC/PC               (Photoeye on Product)
```

**PLC Ladder สำหรับ Label Apply:**

```
=== LADDER: Label Print & Apply ===

Rung 1: Product Detection
|--[PRODUCT_PE]--[OSR]--|
|   [ADD PROD_COUNT, 1, PROD_COUNT]--|
|   +--( PRODUCT_DETECTED )--|

Rung 2: Label Request
|--[PRODUCT_DETECTED]--|
|   // ส่ง VPN/Serial Number ไปยัง Label Printer
|   [MOV PROD_COUNT → LABEL_SERIAL]--|
|   [MOV PROD_LOT   → LABEL_LOT   ]--|
|   [MOV TODAY_DATE → LABEL_DATE  ]--|
|   +--( LABEL_REQUEST )--|

Rung 3: Wait for Print Ready
|--[LABEL_REQUEST]--[T_PRINT_WAIT, EN]--|
| T_PRINT_WAIT: TON, PT=500ms
|
|--[PRINTER_READY]--[T_PRINT_WAIT, DN]--( APPLY_TRIGGER )--|
|--[T_PRINT_WAIT, DN]--[/PRINTER_READY]--( LABEL_FAULT )--|

Rung 4: Apply Label
|--[APPLY_TRIGGER]--|
|   +--( VACUUM_ON )--|      // ดูดฉลากด้วยสุญญากาศ
|   +--( T_VACUUM, EN )--|   // รอ 50ms
|
|--[T_VACUUM, DN]--|
|   +--( ACTUATOR_EXTEND )|-- // ดันออกติดสินค้า
|   +--( T_APPLY, EN )--|    // รอ 100ms
|
|--[T_APPLY, DN]--|
|   +--( ACTUATOR_RETRACT )--| // ดึงกลับ
|   +--( VACUUM_OFF )--|     // ปล่อยสุญญากาศ
|   +--( APPLY_DONE )--|
```

---

## Step 1325: Shrink Wrap System

### Shrink Tunnel Control

```
Product → Film Wrapping → Seal Bar → Shrink Tunnel → Cool Zone → Exit

Shrink Tunnel Zones:
Zone 1: Preheat 120°C (เริ่มต้น)
Zone 2: Shrink  165°C (หดตัว)
Zone 3: Cool    40°C  (คงรูป)

Control Requirements:
- Temperature ±3°C
- Belt Speed สัมพันธ์กับ Product Size
- Consistent Shrink for all Products
```

**Shrink Tunnel Temperature Control:**

```
=== LADDER: Shrink Tunnel Multi-Zone Control ===

Rung 1: Zone 1 Temperature Control
|--[TUNNEL_RUNNING]--( Z1_PID_ENABLE )--|
|
| Z1_PID: SP=120°C, PV=Z1_TEMP, CV=Z1_HEATER_PWM
| PWM Frequency: 1 Hz (1 second cycle)
| Max Heater: 6kW (3-phase)

Rung 2: Zone 2 Temperature Control  
|--[TUNNEL_RUNNING]--( Z2_PID_ENABLE )--|
|
| Z2_PID: SP=165°C, PV=Z2_TEMP, CV=Z2_HEATER_PWM
| Higher setpoint → More Kp needed

Rung 3: Over Temperature Protection
|--[Z1_TEMP > 140]--( Z1_OVER_TEMP )--|
|--[Z1_OVER_TEMP]--( Z1_HEATER_OFF )--|
|--[Z2_TEMP > 190]--( Z2_OVER_TEMP )--|
|--[Z2_OVER_TEMP]--( Z2_HEATER_OFF )--|

Rung 4: Belt Speed to Zone Time
| Belt_Speed (m/min) × Zone_Length (m) = Dwell_Time (s)
| Dwell_Time ต้องพอสำหรับ Shrink สมบูรณ์
|
| Recipe: ชิ้นเล็ก = 3.5 m/min, ชิ้นใหญ่ = 2.0 m/min
```

---

## Step 1326: Case Erecting and Sealing

### Carton Erector and Sealer Control

```
Flat Carton (Magazine) → Erect → Bottom Fold → Glue Bottom → 
Fill → Top Fold → Glue Top → Seal → Label → Palletize

Throughput: 20-60 cases/minute
```

**Case Erector Sequence:**

```
=== LADDER: Case Erector Sequencer ===

State Machine:
State 0: IDLE
State 1: FEED_BLANK
State 2: ERECT_BOX
State 3: FOLD_BOTTOM
State 4: GLUE_BOTTOM
State 5: FILL_POSITION
State 6: WAIT_FILL
State 7: FOLD_TOP
State 8: GLUE_TOP
State 9: DISCHARGE
State 10: FAULT

Rung 1: Transition 0→1 (Start Feed)
|--[STATE=0]--[MAGAZINE_OK]--[GLUE_READY]--[MACHINE_OK]--|
|   +--( FEED_VACUUM_ON )--|
|   +--( STATE := 1 )--|

Rung 2: Transition 1→2 (Blank at Station)
|--[STATE=1]--[BLANK_AT_ERECT_STATION]--|
|   +--( ERECT_CYLINDERS_EXTEND )--|
|   +--( STATE := 2 )--|

Rung 3: Transition 2→3 (Box Erected)
|--[STATE=2]--[BOX_ERECT_CONFIRM]--|
|   +--( BOTTOM_FOLD_START )--|
|   +--( STATE := 3 )--|

// ... ต่อเนื่องจนถึง State 9

Rung 9: Transition 8→9 (Glue Applied)
|--[STATE=8]--[GLUE_APPLIED]--[TOP_FOLD_DONE]--|
|   +--( DISCHARGE_BELT_START )--|
|   +--( CASE_COUNT ++ )--|
|   +--( STATE := 0 )--|   // วนซ้ำ
```

---

## Step 1327: Palletizing Patterns

### Pattern-Based Palletizing

```
Palletizing Patterns:
Pattern A: Brick Pattern
Layer 1: → → → →    Layer 2: ↑ ↑ ↑ ↑
         → → → →             ↑ ↑ ↑ ↑
         → → → →             ↑ ↑ ↑ ↑

Pattern B: Pin-Wheel
Layer 1: → → →      Layer 2: ↑ → ↑
         ↑ ↓ ↑               → ↑ →
         ← ← ←               ↑ → ↑

สลับ Layer ทำให้ Pallet แข็งแรงขึ้น (Interlocking)
```

**Palletizer PLC Program (Omron NJ):**

```pascal
// Palletizing Pattern Database
// เก็บ X,Y,Orientation ของแต่ละ Position ใน Pattern
TYPE T_PalletPosition :
STRUCT
    X_Pos     : REAL;    // mm จากจุดอ้างอิง
    Y_Pos     : REAL;    // mm
    Z_Pos     : REAL;    // mm (Layer Height)
    Rotation  : REAL;    // องศาการหมุน (0=ตั้ง, 90=นอน)
    Layer_No  : USINT;   // เลข Layer
    Pos_No    : USINT;   // เลขตำแหน่งใน Layer
END_STRUCT
END_TYPE

// Pattern Table - Brick Pattern (4x3 = 12 per layer, 6 layers = 72 cases)
VAR
    PalletDB : ARRAY[1..72] OF T_PalletPosition;
END_VAR

// Initialize Pattern (เรียกครั้งแรก)
PROGRAM Init_Brick_Pattern
VAR
    i, layer, pos : INT;
    case_length, case_width, case_height : REAL;
    layer_height : REAL;
END_VAR

case_length := 400.0;  // mm
case_width  := 300.0;
case_height := 250.0;

FOR layer := 1 TO 6 DO
    layer_height := (layer - 1) * case_height + 120.0;  // 120mm = Pallet height
    
    FOR pos := 1 TO 12 DO  // 4 columns × 3 rows
        i := (layer - 1) * 12 + pos;
        
        IF (layer MOD 2) = 1 THEN
            // Odd layers - lengthwise
            PalletDB[i].X_Pos := ((pos - 1) MOD 4) * case_length;
            PalletDB[i].Y_Pos := ((pos - 1) / 4) * case_width;
            PalletDB[i].Rotation := 0.0;
        ELSE
            // Even layers - rotate 90° for interlocking
            PalletDB[i].X_Pos := ((pos - 1) MOD 3) * case_width;
            PalletDB[i].Y_Pos := ((pos - 1) / 3) * case_length;
            PalletDB[i].Rotation := 90.0;
        END_IF;
        
        PalletDB[i].Z_Pos := layer_height;
        PalletDB[i].Layer_No := layer;
        PalletDB[i].Pos_No := pos;
    END_FOR;
END_FOR;
END_PROGRAM
```

---

## Step 1328: Stretch Wrapping Control

### Turntable Stretch Wrapper

```
Pallet Enter → Turntable Rotate → Film Stretch → 
Film Tower Move Up/Down → Film Cut → Wipe Down → Pallet Exit

Control Parameters:
- Turntable Speed: 4-12 RPM
- Film Pre-stretch: 150-300%
- Film Tension: 3-8 kg
- Tower Speed: สัมพันธ์กับ Turntable (overlap setting)
- Bottom Wrap Count: 2-4 รอบ
- Top Wrap Count: 2-4 รอบ
- Main Wrap: cover 100% ด้วย 25% overlap
```

**Stretch Wrapper Program:**

```
=== LADDER: Stretch Wrapper Sequence ===

Rung 1: Pallet in Position
|--[PALLET_PE]--[/TURNTABLE_RUNNING]--( WRAP_START )--|

Rung 2: Lower Wrap Phase
|--[WRAP_START]--|
|   [MOV 2 → WRAP_COUNT_TARGET]--|   // 2 รอบด้านล่าง
|   [MOV 0.3 → TOWER_POS_TARGET]--|  // 300mm from floor
|   +--( TURNTABLE_START )--|
|   +--( TOWER_LOWER_START )--|

Rung 3: Count Turntable Rotations
|--[TURNTABLE_HOME]--[OSR]--|
|   [ADD WRAP_COUNT, 1, WRAP_COUNT]--|

Rung 4: Transition to Main Wrap
|--[WRAP_COUNT >= WRAP_COUNT_TARGET]--[WRAP_PHASE = 1]--|
|   [MOV 0 → WRAP_COUNT]--|
|   [MOV PALLET_HEIGHT - 0.3 → TOWER_TRAVEL_DIST]--|
|   [DIV TOWER_TRAVEL_DIST, OVERLAP_CALC, TOWER_SPEED]--|
|   +--( WRAP_PHASE := 2 )--|   // Main wrap phase

Rung 5: Film Cut and Wipe
|--[TOP_WRAP_DONE]--[TURNTABLE_HOME]--|
|   +--( FILM_CUT )--|
|   +--( WIPE_ARM_EXTEND )--|
|   +--( TURNTABLE_STOP )--|
```

---

## Step 1329: Complete High-Speed Packaging Line

### Integration ของทุก Component

```
Line Layout:
                    Case Packer
                        ↓
Product → Accumulate → Fill → Seal → Check Weight → Date Code → 
                                          ↓               ↓
                                     Metal Detect     Vision Inspect
                                          ↓
                                     Label Apply → Case Pack → Palletize
                                     
Speed: 300 units/min
Efficiency Target: 85% OEE
```

**Master Line Controller:**

```pascal
// Packaging Line Master Controller
FUNCTION_BLOCK FB_PackagingMaster
VAR
    // Line Status
    LineState      : INT;   // 0=Idle, 1=Running, 2=Faulted, 3=Changeover
    LineSpeed      : REAL;  // Units/min
    TargetSpeed    : REAL;  // Units/min
    
    // Counters
    GoodCount      : DINT;
    RejectCount    : DINT;
    DowntimeSecs   : DINT;
    
    // Machine Status Array
    MachineOK      : ARRAY[1..15] OF BOOL;  // 15 เครื่องในไลน์
    MachineFault   : ARRAY[1..15] OF INT;   // รหัสความผิดพลาด
    
    // Product Recipe
    CurrentRecipe  : T_PackRecipe;
END_VAR

// Main Control Logic
CASE LineState OF
    0: // IDLE - Wait for Start Command
        IF StartCmd AND AllMachinesReady() THEN
            LoadRecipe(CurrentRecipe);
            LineState := 1;
        END_IF;
    
    1: // RUNNING - Normal Production
        // Ramp up speed
        IF LineSpeed < TargetSpeed THEN
            LineSpeed := LineSpeed + 5.0;  // +5 units/min per cycle
        END_IF;
        
        // Monitor for faults
        FOR i := 1 TO 15 DO
            IF NOT MachineOK[i] THEN
                FaultMachine := i;
                LineState := 2;
            END_IF;
        END_FOR;
    
    2: // FAULTED
        LineSpeed := 0.0;
        DowntimeSecs := DowntimeSecs + 1;
        
        IF FaultCleared AND ResetCmd THEN
            LineState := 1;
        END_IF;
    
    3: // CHANGEOVER
        LineSpeed := 0.0;
        IF ChangeoverComplete() THEN
            LineState := 0;
        END_IF;
END_CASE;
END_FUNCTION_BLOCK
```

---

## Step 1330: Vision System Integration

### Machine Vision สำหรับ Quality Control

```
Vision System Tasks:
1. Label Position Check - ฉลากถูกต้องตำแหน่งหรือไม่
2. Barcode Verification - Barcode อ่านได้และถูกต้อง
3. Cap Presence - มีฝาหรือไม่
4. Fill Level Check - ระดับน้ำถูกต้อง
5. Package Integrity - ไม่มีรอยแตก รอยบี้
```

**Vision-PLC Interface:**

```
=== LADDER: Vision System Control ===

Rung 1: Trigger Vision Inspection
|--[PRODUCT_AT_INSPECT]--[OSR]--|
|   +--( VISION_TRIGGER )--|
|   +--( T_VISION_WAIT, EN )--|   // รอผล max 100ms

Rung 2: Vision Pass
|--[VISION_PASS_SIGNAL]--|
|   [MOV VISION_DATA → INSPECT_RESULT]--|
|   +--( PRODUCT_PASS )--|
|   +--( GOOD_COUNT ++ )--|

Rung 3: Vision Fail
|--[VISION_FAIL_SIGNAL]--|
|   [MOV VISION_FAIL_CODE → REJECT_REASON]--|
|   +--( PRODUCT_REJECT )--|
|   +--( REJECT_COUNT ++ )--|
|   +--( DIVERT_SOLENOID )--|   // เปิด Diverter ไปถัง Reject

Rung 4: Vision Timeout (No response)
|--[T_VISION_WAIT, DN]--[/VISION_PASS_SIGNAL]--[/VISION_FAIL_SIGNAL]--|
|   +--( VISION_TIMEOUT_ALARM )--|
|   +--( PRODUCT_REJECT )--|   // Safe side: reject ถ้าไม่ได้ผล
```

---

## Step 1331-1340: Advanced Topics

### Step 1331: Recipe Management System

```pascal
// Recipe Structure for Packaging Line
TYPE T_PackRecipe :
STRUCT
    RecipeID        : INT;
    ProductName     : STRING[50];
    
    // Film Parameters
    Film_PreStretch  : REAL;   // %
    Film_Tension     : REAL;   // kg
    
    // Seal Parameters
    Seal_Temp_Upper  : REAL;   // °C
    Seal_Temp_Lower  : REAL;   // °C
    Seal_Dwell_Time  : REAL;   // ms
    
    // Label Parameters
    Label_X_Offset   : REAL;   // mm
    Label_Y_Offset   : REAL;   // mm
    Label_Template   : INT;    // Template ID
    
    // Speed Parameters
    Line_Speed       : REAL;   // Units/min
    
    // Weight Check
    Target_Weight    : REAL;   // g
    Weight_Tol_Plus  : REAL;   // g
    Weight_Tol_Minus : REAL;   // g
END_STRUCT
END_TYPE
```

### Step 1332: Checkweigher Integration

```
=== LADDER: Checkweigher Control ===

Rung 1: Product Enter Checkweigher
|--[CW_ENTRY_PE]--[OSR]--|
|   +--( CW_SAMPLE_TRIGGER )--|   // เริ่ม Sample น้ำหนัก

Rung 2: Weight Measurement Complete
|--[CW_READY]--|
|   [MOV CW_WEIGHT → MEASURED_WEIGHT]--|
|
| CW_WEIGHT มาจาก Load Cell ใน Checkweigher
| อ่านหลังจาก Product อยู่นิ่งบน Belt ประมาณ 50ms

Rung 3: Weight Evaluation
|--[CW_READY]--|
|   [CMP MEASURED_WEIGHT >= MIN_WEIGHT]--|
|   [CMP MEASURED_WEIGHT <= MAX_WEIGHT]--|
|   +--( WEIGHT_OK )--|

|--[CW_READY]--|
|   [CMP MEASURED_WEIGHT < MIN_WEIGHT]--|
|   +--( WEIGHT_UNDERWEIGHT )--|
|   +--( REJECT_COUNT ++ )--|

|--[CW_READY]--|
|   [CMP MEASURED_WEIGHT > MAX_WEIGHT]--|
|   +--( WEIGHT_OVERWEIGHT )--|
|   +--( REJECT_COUNT ++ )--|

Rung 4: SPC Data Capture
|--[WEIGHT_OK]--|
|   // บันทึกน้ำหนักทุกชิ้นสำหรับ SPC (Statistical Process Control)
|   [FIFO_STORE MEASURED_WEIGHT → WEIGHT_FIFO]--|
|   // คำนวณ X-bar และ R Chart ทุก 25 ชิ้น
```

### Step 1333: Inkjet Date Coding

```
=== LADDER: Inkjet Coder Trigger ===

Rung 1: Product at Coder Position
|--[CODER_PE]--[OSR]--|
|   +--( CODER_TRIGGER )--|

Rung 2: Dynamic Message Building
|--[SHIFT_START]--[OSR]--|
|   // อัพเดท Date/Time Message ทุกกะ
|   [FORMAT 'BB{0:dd/MM/yy} {0:HH:mm}' DATE_TIME → CODER_MSG]--|
|   [SEND_MSG CODER_MSG → CODER_RS232]--|

| Format ตัวอย่าง: "BB12/10/26 14:30"
| BB = Best Before, วันที่ผลิต + 365 วัน = BB
```

### Step 1334-1340: Finished Line Summary

**Complete Packaging Line Performance Dashboard (KPIs):**

```
Key Performance Indicators:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OEE               = Availability × Performance × Quality
                  = 92% × 96% × 99.5% = 88.1% ✓ (Target 85%)

Availability      = 552/600 min = 92%
                  (48 min downtime in 10hr shift)

Performance       = 17,280 units / 18,000 target = 96%
                  (300 units/min × 480 min production time)

Quality           = 17,193 good / 17,280 total = 99.5%
                  (87 rejects: 42 weight, 25 label, 20 seal)

MTBF              = 552/6 stops = 92 min average run time
MTTR              = 48/6 stops = 8 min average repair time

Yield Loss Sources:
  Weight Rejects  : 0.24% (target <0.15%) → Need tuning
  Label Rejects   : 0.14% (target <0.15%) ✓
  Seal Rejects    : 0.12% (target <0.10%) → Slightly high
```

---

## Lab Project 67: High-Speed Label Application System

### โจทย์: ออกแบบและตั้งค่า PLC สำหรับระบบพิมพ์และติดฉลากอัตโนมัติ

**ข้อกำหนดระบบ:**
- ความเร็วสาย: 150 units/minute
- ฉลากขนาด: 100mm × 50mm
- Label Stock: 3" core, 500mm OD max
- ระบบตรวจสอบ: Vision ตรวจ barcode + label position
- เกณฑ์ Reject: label offset > ±2mm หรือ barcode ไม่อ่าน
- Reject: Diverter ลงกล่อง Reject โดยอัตโนมัติ

**งาน:**
1. เขียน Ladder Diagram สำหรับ Label Apply Sequence
2. Integrate กับ Vision System (Keyence CV-X Series)
3. สร้าง Recipe สำหรับ 5 ผลิตภัณฑ์แตกต่างกัน
4. ออกแบบ HMI Screen สำหรับ Operator
5. สร้าง Report ส่ง MES ทุก 15 นาที

**โครงสร้าง I/O:**

| Signal | Type | Address (S7-1500) | Description |
|--------|------|-------------------|-------------|
| Product PE | DI | I0.0 | ตรวจสอบสินค้า |
| Label PE | DI | I0.1 | ตรวจสอบฉลากพร้อม |
| Apply Solenoid | DO | Q0.0 | กระตุ้น Apply |
| Reject Diverter | DO | Q0.1 | เบี่ยงของ Reject |
| Vision Trigger | DO | Q0.2 | กระตุ้น Vision |
| Vision Pass | DI | I0.2 | ผล Pass จาก Vision |
| Vision Fail | DI | I0.3 | ผล Fail จาก Vision |
| Label Count AI | AI | IW256 | นับฉลากจาก Encoder |

---

## สรุป Part 67

ใน Part นี้เราได้เรียนรู้:
- Web Tension Control สำหรับวัสดุที่เป็น Roll
- Color Registration สำหรับงานพิมพ์หลายสี
- Servo-Controlled Flying Cutoff
- Label Print & Apply Systems
- Shrink Wrap และ Case Sealing
- Palletizing Pattern Programming
- Vision System Integration สำหรับ QC

**ทักษะที่ได้:**
- PID Control สำหรับ Tension
- High-Speed I/O (< 1ms response)
- Recipe Management
- OEE Monitoring สำหรับ Packaging

---

### 🔗 Navigation
- [← Part 66: Automotive Automation](part66_automotive_automation.md)
- [→ Part 68: Water Treatment](part68_water_treatment.md)
- [📚 สารบัญหลัก](../README.md)
