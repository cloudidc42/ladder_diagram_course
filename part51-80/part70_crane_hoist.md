# Part 70: Crane และ Hoist Control (Steps 1381-1400)

## การเขียนโปรแกรม PLC สำหรับปั้นจั่นและเครื่องยกระดับโลก

---

### 🔗 Navigation
- [← Part 69: HVAC Automation](part69_hvac_automation.md)
- [→ Part 71: Press & Stamping](part71_press_stamping.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: ระบบปั้นจั่นและเครื่องยก

ปั้นจั่น (Crane) และเครื่องยก (Hoist) เป็นอุปกรณ์ที่พบในโรงงาน อู่ต่อเรือ ท่าเรือ และไซต์ก่อสร้าง การเขียน PLC Program สำหรับอุปกรณ์เหล่านี้ต้องพิเศษกว่าปกติเพราะ:

1. **ความปลอดภัยสูงมาก** - น้ำหนักตกอาจทำให้เสียชีวิต
2. **Dynamic Load** - น้ำหนักที่แกว่งสร้างแรงกระทำเพิ่ม
3. **Anti-Sway** - ต้องหยุดการแกว่งของ Load
4. **Overload Protection** - ห้ามยกเกินพิกัด

### ประเภทของปั้นจั่น

```
1. Overhead Travelling Crane (EOT Crane)
   Bridge + Trolley + Hoist
   ใช้ใน: โรงงานเหล็ก, อู่ต่อเรือ, โรงงานผลิต

2. Gantry Crane
   เหมือน EOT แต่วิ่งบนรางระดับพื้น
   ใช้ใน: ท่าเรือ Container, โรงงานกลางแจ้ง

3. Tower Crane
   ตั้งตรงขึ้นสูง มี Jib แขนงหมุน
   ใช้ใน: งานก่อสร้างอาคารสูง

4. Mobile Crane
   ติดบนรถ เคลื่อนที่ได้
   ใช้ใน: งานก่อสร้าง, ซ่อมบำรุง
```

---

## Step 1381: Overhead Crane Components

### EOT Crane Main Components

```
EOT Crane Structure:
           ←─────── Bridge Travel ────────→
         ┌─────────────────────────────────┐
         │         BRIDGE GIRDER           │
         │    ←── Trolley Travel ──→       │
         │    ┌────────────────────┐       │
         │    │    CRAB/TROLLEY    │       │
         │    │  ┌──────────────┐  │       │
         │    │  │     HOIST    │  │       │
         │    └──┴──────────────┴──┘       │
         └─────────────────────────────────┘
               │                     │
           ┌───┴──┐               ┌──┴────┐
           │ End  │               │  End  │
           │Truck │               │ Truck │
           │(Long │               │(Long  │
           │Travel│               │Travel)│
           └──────┘               └───────┘
              Rail                   Rail

Movements:
1. Long Travel (Bridge): East-West
2. Cross Travel (Trolley): North-South  
3. Hoist: Up-Down
4. (Some cranes have rotation too)
```

---

## Step 1382: Basic Crane Control - การควบคุมพื้นฐาน

**ประเภทของการควบคุม:**
1. **Pendant Control** - ควบคุมด้วยสายห้อย มือถือ
2. **Remote Control (Radio)** - ควบคุมไร้สาย
3. **Cabin Control** - ควบคุมจากห้องขับในปั้นจั่น
4. **Automatic** - ควบคุมอัตโนมัติด้วย PLC เต็มรูปแบบ

**Allen-Bradley Compact LogiX สำหรับ EOT Crane:**

```
=== LADDER: Crane Basic Motion Control ===

Rung 1: Long Travel Forward
|--[LT_FWD_PB]--[/LT_REV_PB]--[/LT_FWD_LS]--[CRANE_SAFE]--|
|   +--( LT_FWD_OUTPUT )--|
|
| LT_FWD_LS = Limit Switch หยุดจาก Forward End
| CRANE_SAFE = E-Stop OK + Overload OK + No Fault

Rung 2: Long Travel Reverse
|--[LT_REV_PB]--[/LT_FWD_PB]--[/LT_REV_LS]--[CRANE_SAFE]--|
|   +--( LT_REV_OUTPUT )--|

Rung 3: Hoist Up
|--[HOIST_UP_PB]--[/HOIST_DN_PB]--[/HOIST_UP_LS]--|
|   [CRANE_SAFE]--[LOAD_WITHIN_LIMIT]--|
|   +--( HOIST_UP_OUTPUT )--|

Rung 4: Hoist Down
|--[HOIST_DN_PB]--[/HOIST_UP_PB]--[/HOIST_DN_LS]--|
|   [CRANE_SAFE]--|
|   +--( HOIST_DN_OUTPUT )--|
|
| หมายเหตุ: ยกลงไม่ต้องตรวจ Overload เพราะกำลังปล่อย Load

Rung 5: Slow Zone (Near Limit Switch)
|--[LT_FWD_PB]--[LT_FWD_SLOW_ZONE]--|
|   +--( LT_FWD_SLOW )--|     // เปลี่ยนจากความเร็วสูงเป็นต่ำ
|
| ติดตั้ง Slow Zone Cam ก่อนถึง Limit Switch ประมาณ 0.5-1 เมตร
```

---

## Step 1383: VFD Hoist Control - การควบคุมด้วย VFD

### ทำไมต้องใช้ VFD กับ Hoist

```
ปัญหาของ Direct-On-Line (DOL) Hoist:
  1. Jerk ตอน Start/Stop → Load แกว่ง
  2. ไม่มี Speed Control → ยกสิ่งบอบบางไม่ได้
  3. Motor สึกหรอเร็ว (Current Spike ทุก Start)
  4. ไม่มี Anti-Rollback (Load อาจหล่น)

VFD Benefits:
  1. Smooth Start/Stop → ลด Jerk
  2. Variable Speed (2-100%)
  3. Torque Control Mode → Anti-Rollback
  4. Motor Protection Built-in
  5. Energy Saving (ลด 20-40%)
```

**VFD Hoist Control with Torque Control:**

```pascal
// VFD Hoist Control - Anti-Rollback
FUNCTION_BLOCK FB_Hoist_VFD
VAR_INPUT
    Up_Cmd      : BOOL;    // ยกขึ้น
    Down_Cmd    : BOOL;    // ยกลง
    Speed_Cmd   : REAL;    // % ความเร็ว (จาก Joystick)
    Load_Weight : REAL;    // kg น้ำหนัก Load
    Motor_Torque: REAL;    // % Torque ปัจจุบัน
    Motor_Speed : REAL;    // RPM ความเร็วปัจจุบัน
END_VAR
VAR_OUTPUT
    VFD_Speed_SP: REAL;    // Hz Setpoint ไปยัง VFD
    VFD_Direction: BOOL;   // TRUE=Up, FALSE=Down
    Brake_Release: BOOL;   // TRUE=ปล่อยเบรก
    Safety_Fault : BOOL;   // TRUE=มี Fault
END_VAR
VAR
    Required_Holding_Torque : REAL;
    Holding_OK : BOOL;
    Ramp_Up_Timer : TON;
END_VAR

// คำนวณ Torque ที่ต้องการสำหรับยกน้ำหนัก
// Torque ∝ Load Weight × Drum Radius / Gear Ratio
Required_Holding_Torque := (Load_Weight * 9.81 * 0.15) / 150.0 * 100.0;
// 0.15m = Drum Radius, 150 = Gear Ratio → แปลงเป็น % of rated

// Anti-Rollback: ต้องมี Torque พอก่อนปล่อยเบรก
IF Up_Cmd THEN
    // ยกขึ้น: รอให้ VFD สร้าง Torque พอก่อนปล่อยเบรก
    IF Motor_Torque > (Required_Holding_Torque * 1.1) THEN
        Brake_Release := TRUE;
        VFD_Direction := TRUE;  // Up
        VFD_Speed_SP := Speed_Cmd * 50.0 / 100.0;  // 0-50 Hz
        Holding_OK := TRUE;
    ELSE
        Brake_Release := FALSE;  // ยังไม่ปล่อยเบรก
        VFD_Speed_SP := 5.0;     // ต่ำๆ ก่อน เพื่อสร้าง Torque
    END_IF;
    
ELSIF Down_Cmd THEN
    // ยกลง: ระวัง Overspeed
    Brake_Release := TRUE;
    VFD_Direction := FALSE;  // Down
    // ควบคุมความเร็วลง
    VFD_Speed_SP := Speed_Cmd * 25.0 / 100.0;  // ลงช้ากว่าขึ้น
    
    // Anti-Runaway: ถ้าความเร็วเกิน → ดึงเบรก
    IF Motor_Speed > 110.0 THEN  // > 110% rated
        Brake_Release := FALSE;
        Safety_Fault := TRUE;
    END_IF;
    
ELSE
    // ไม่มีคำสั่ง: หยุด
    VFD_Speed_SP := 0.0;
    // Brake ต้องปิดเมื่อ Speed = 0 (ป้องกัน Brake Wear)
    IF ABS(Motor_Speed) < 5.0 THEN
        Brake_Release := FALSE;  // ล็อคเบรกเมื่อหยุด
    END_IF;
END_IF;
END_FUNCTION_BLOCK
```

---

## Step 1384: Anti-Sway Control

### การป้องกันการแกว่งของ Load

```
ปรากฏการณ์การแกว่ง (Pendulum Effect):
  Load แขวนด้วยเชือก/โซ่ → เมื่อปั้นจั่นเคลื่อนที่ Load จะแกว่ง
  
  Period of Swing: T = 2π × √(L/g)
  ตัวอย่าง: L = 10 เมตร
  T = 2π × √(10/9.81) = 6.34 วินาที
  
  ปัญหา: ทำให้ Load ยากต่อการวาง ต้องรอให้หยุดแกว่ง
  แก้ไขด้วย: Input Shaping (Anti-Sway Algorithm)
```

**Input Shaping Anti-Sway:**

```
Input Shaping Algorithm:
  แทนที่จะส่ง Step Command → ส่ง Shaped Command
  
  Original Command: ──────────────────
  Shaped Command:   ──────┐  ┌────────
                          └──┘
  (หยุดกลาง ช่วงสั้นๆ แล้วไปต่อ)
  ระยะเวลา ขึ้นกับ Pendulum Period
  
  ผลลัพธ์: Load แกว่งน้อยมาก
```

```pascal
// Anti-Sway Input Shaping for Crane
FUNCTION_BLOCK FB_AntiSway
VAR_INPUT
    Bridge_Speed_Cmd : REAL;    // m/s คำสั่งความเร็วจาก Operator
    Rope_Length      : REAL;    // m ความยาวเชือก (ปรับตามตำแหน่ง Hoist)
    Load_Weight      : REAL;    // kg
END_VAR
VAR_OUTPUT
    Bridge_Speed_SP  : REAL;    // m/s คำสั่งที่ส่งไปยัง VFD จริง
    Sway_Angle_Est   : REAL;    // rad มุมแกว่งที่ประมาณ
END_VAR
VAR
    // Pendulum Parameters
    Natural_Freq     : REAL;    // rad/s
    Damped_Freq      : REAL;    // rad/s
    Damping_Ratio    : REAL := 0.02;  // โดยทั่วไป 2% สำหรับเชือก
    
    // Input Shaping Impulse Times
    T1, T2, T3      : REAL;    // เวลาส่ง Impulse
    A1, A2, A3      : REAL;    // ขนาด Impulse
    
    // State Variables
    SampleTime       : REAL := 0.01;  // 10ms
    
    // Anti-sway state estimation
    Sway_Vel_Est     : REAL;    // Angular velocity estimation
END_VAR

// คำนวณ Natural Frequency จากความยาวเชือก
// ωn = √(g/L)
IF Rope_Length > 0.1 THEN
    Natural_Freq := SQRT(9.81 / Rope_Length);
ELSE
    Natural_Freq := 10.0;  // Default สั้นมาก
END_IF;

Damped_Freq := Natural_Freq * SQRT(1.0 - Damping_Ratio * Damping_Ratio);

// ZV (Zero Vibration) Input Shaper
// ส่ง 2 Impulse ห่างกัน T/2 (T = period of oscillation)
A1 := 1.0 / (1.0 + EXP(-Damping_Ratio * 3.14159 / Damped_Freq));
A2 := 1.0 - A1;

T1 := 0.0;  // ส่งทันที
T2 := 3.14159 / Damped_Freq;  // ส่งอีกครั้งหลัง T/2

// Apply shaped command (implementation depends on platform)
// ในทางปฏิบัติต้องใช้ Buffer และ Delay
// Bridge_Speed_SP = A1 × Cmd(t - T1) + A2 × Cmd(t - T2)
END_FUNCTION_BLOCK
```

---

## Step 1385: Zone Protection - การป้องกันพื้นที่

### Zone-Based Access Control for Cranes

```
ปั้นจั่นในโรงงานมักต้องจำกัดพื้นที่:
1. No-Fly Zone: พื้นที่ที่ปั้นจั่นห้ามผ่าน
2. Slow Zone: พื้นที่ที่ต้องลดความเร็ว
3. Lifting Zone: พื้นที่ที่อนุญาตยกเท่านั้น

ตัวอย่างในโรงงาน:
Zone A (Normal): เต็มความเร็ว
Zone B (Slow): ใกล้คลังสินค้า ลด 50%
Zone C (Caution): เหนือสายการผลิต ลด 25%
Zone D (Prohibited): เหนืออุปกรณ์ไฟฟ้า ห้ามผ่านเลย
```

**Zone Protection Program:**

```
=== LADDER: Crane Zone Protection ===

Rung 1: Position Reading
| Absolute Encoder หรือ Distance Laser Sensor
| X_Position = Long Travel Position (mm)
| Y_Position = Cross Travel Position (mm)

Rung 2: Zone Detection
|--[X_POS > 5000]--[X_POS < 8000]--|
|   [Y_POS > 2000]--[Y_POS < 4000]--|
|   +--( IN_ZONE_B )--|   // Slow Zone B

|--[X_POS > 10000]--[X_POS < 12000]--|
|   [Y_POS > 0]--[Y_POS < 6000]--|
|   +--( IN_ZONE_D )--|   // Prohibited Zone D

Rung 3: Apply Speed Limits
|--[IN_ZONE_B]--|
|   [MOV MAX_SPEED * 0.5 → SPEED_LIMIT]--|   // 50% ของ Max Speed

|--[IN_ZONE_C]--|
|   [MOV MAX_SPEED * 0.25 → SPEED_LIMIT]--|  // 25% ของ Max Speed

|--[IN_ZONE_D]--[MOTION_TOWARD_ZONE_D]--|
|   +--( EMERGENCY_STOP_MOTION )--|   // หยุดทันที!

Rung 4: Zone D Approach Warning
|--[DISTANCE_TO_ZONE_D < 1000mm]--|   // ห่างน้อยกว่า 1 เมตร
|   +--( ZONE_WARN_HORN )--|   // แตรเตือน
|   +--( SLOW_TO_CREEP )--|    // ลด 10% เพื่อให้ operator หยุด
```

---

## Step 1386: Overload Protection System

### Load Cell and Overload Detection

```
Overload Protection Levels:
  Safe Working Load (SWL):         100% → ยกได้ปกติ
  Warning Zone:               100-110% → เตือน
  Rated Capacity Limiter (RCL): 110% → ห้ามยกขึ้น (แต่ยกลงได้)
  Safety Stop:                   120% → หยุดทุก Motion

ตาม ISO 4301 / FEM 1.001:
  ต้องมี RCL (Rated Capacity Limiter)
  ต้องมี Load Indicator
  ต้องมี Safety Device
```

**Overload Protection Ladder:**

```
=== LADDER: Overload Protection (Safety Circuit) ===

Rung 1: Load Reading
| Load Cell → Weight Transmitter (4-20mA) → PLC AI
| Load_kg = (AI_Raw - 4.0) / 16.0 × LOAD_CELL_MAX_KG
| ตัวอย่าง: AI=12mA, Max=10,000 kg
| Load = (12-4)/16 × 10000 = 5,000 kg

Rung 2: SWL Percentage Calculation
| Load_pct = Load_kg / SWL_kg × 100.0

Rung 3: Warning Level (100%)
|--[Load_pct >= 100]--[Load_pct < 110]--( OL_WARNING )--|
|--[OL_WARNING]--( WARN_HORN )--|

Rung 4: RCL - Prevent Hoist Up Only
|--[Load_pct >= 110]--( RCL_ACTIVE )--|
|--[RCL_ACTIVE]--[HOIST_UP_CMD]--( HOIST_UP_BLOCK )--|
|--[/HOIST_UP_BLOCK]--( HOIST_UP_OUTPUT )--|   // ยกขึ้นไม่ได้
| หมายเหตุ: ยกลงยังได้ เพื่อให้วาง Load ลงปลอดภัย

Rung 5: Safety Stop (120%)
|--[Load_pct >= 120]--( SAFETY_STOP_OL )--|
|--[SAFETY_STOP_OL]--( ALL_MOTION_STOP )--|
|--[SAFETY_STOP_OL]--( EMERGENCY_ALARM )--|
| ต้องมี Manual Reset หลังแก้ไข

Rung 6: Dynamic Load Compensation
| ตอน Hoist เริ่ม → Dynamic Load สูงกว่า Static
| Load_Dynamic = Load_Static × (1 + a/g)
| a = Acceleration, g = 9.81 m/s²
| ตัวอย่าง: a = 0.5 m/s², Load = 5000 kg
| Load_Dynamic = 5000 × (1 + 0.5/9.81) = 5255 kg
```

---

## Step 1387: End Limit Switches Design

### ประเภทของ Limit Switch สำหรับ Crane

```
Limit Switch ที่จำเป็น:
1. Hoist Upper Limit: หยุดยกขึ้นเมื่อ Hook สูงสุด
2. Hoist Lower Limit: หยุดยกลงเมื่อ Hook ต่ำสุด
3. Long Travel FWD Limit: หยุดเดินหน้า
4. Long Travel REV Limit: หยุดถอยหลัง
5. Cross Travel FWD Limit: หยุดเดินหน้า (แกน Y)
6. Cross Travel REV Limit: หยุดถอยหลัง (แกน Y)

Two-Stage Limit System (Redundancy):
  Stage 1 (Slow Zone Cam): ลดความเร็วเหลือ Creep Speed
  Stage 2 (Final Limit): หยุดสนิท (Emergency)

ตาม EN 15011: ต้องมี 2 ขั้นตอน
  Working Limit (ปกติ)
  Safety Limit (Emergency Backup)
```

**Dual-Stage Limit Switch Program:**

```
=== LADDER: Dual-Stage Hoist Limits ===

Rung 1: Stage 1 - Slow Down
|--[HOIST_UP_CMD]--[/HOIST_UPPER_SLO_LS]--|
|   // LS ยังไม่ถึง → ยกได้ปกติ

|--[HOIST_UP_CMD]--[HOIST_UPPER_SLO_LS]--|
|   [/HOIST_UPPER_STOP_LS]--|
|   +--( HOIST_CREEP )--|   // เข้า Slow Zone → Creep Speed 5%

Rung 2: Stage 2 - Stop
|--[HOIST_UPPER_STOP_LS]--( HOIST_UP_STOP )--|
|--[HOIST_UP_STOP]--( HOIST_UP_OUTPUT CLR )--|
|--[HOIST_UP_STOP]--( ALARM_UPPER_LIMIT )--|
|
| Stage 2 Limit → ตัด Output ทันที ไม่ต้องรอ PLC Scan
| ควรต่อ Hardware Interlock (Relay Contact) แทน Software เพียงอย่างเดียว

Rung 3: Recovery
|--[HOIST_DN_CMD]--[HOIST_UP_STOP]--|
|   // ยกลงได้เพื่อออกจาก Limit Position
```

---

## Step 1388: Safe Working Practices (SWP)

### มาตรฐานความปลอดภัยสำหรับปั้นจั่น

```
กฎหมายและมาตรฐาน:
  ไทย: กฎกระทรวง กำหนดมาตรฐานปั้นจั่น พ.ศ.2564
  สากล: ISO 4301, ISO 9927 (Inspection), EN 15011
  สหรัฐ: OSHA 1926.1400 (Construction), 1910.179 (General Industry)

Pre-Shift Inspection (ก่อนใช้งานทุกกะ):
□ ตรวจสอบ Wire Rope (ไม่มีเส้นขาด, Kink, Corrosion)
□ ตรวจสอบ Hook (Crack, Deformation, Safety Latch)
□ ตรวจสอบ Brake (Hold Load ได้ 125% SWL)
□ ตรวจสอบ Limit Switch ทุกตัว
□ ทดสอบ E-Stop
□ ตรวจสอบ Warning Devices (Horn, Light)
```

**PLC Pre-Shift Self-Test:**

```pascal
// Automated Pre-Shift Test Sequence
FUNCTION_BLOCK FB_CranePreShiftTest
VAR_OUTPUT
    Test_Pass       : BOOL;
    Fail_Code       : INT;
    Test_Complete   : BOOL;
END_VAR
VAR
    Test_State      : INT := 0;
    Test_Timer      : TON;
END_VAR

CASE Test_State OF
    0: // Start Test
        Test_Pass := FALSE;
        Test_Timer(IN := TRUE, PT := T#5s);
        Test_State := 1;
        
    1: // Test Hoist Up Limit
        // Move Hoist Up slowly until limit
        Hoist_Up_Slow := TRUE;
        IF Upper_Limit_SW THEN
            Test_State := 2;
            Hoist_Up_Slow := FALSE;
        END_IF;
        IF Test_Timer.Q THEN
            // ไม่พบ Limit ภายใน 5 วินาที → Fault
            Fail_Code := 1;  // Hoist Upper Limit Failed
            Test_State := 99;
        END_IF;
    
    2: // Test Down from upper limit
        Hoist_Down_Slow := TRUE;
        Test_Timer(IN := TRUE, PT := T#3s);
        Test_State := 3;
    
    3: // Continue down tests...
        IF Test_Timer.Q THEN
            Test_State := 4;  // Continue to next test
        END_IF;
    
    // ... ทดสอบ Travel Limits, E-Stop, Horn, Light
    
    10: // All tests passed
        Test_Pass := TRUE;
        Test_Complete := TRUE;
        Test_State := 0;
    
    99: // Test Failed
        Test_Pass := FALSE;
        Test_Complete := TRUE;
        Crane_Lock_Out := TRUE;  // ล็อคปั้นจั่น ห้ามใช้จนกว่าซ่อมเสร็จ
END_CASE;
END_FUNCTION_BLOCK
```

---

## Step 1389: Sway Calculation Algorithm (Advanced)

### Mathematical Model สำหรับ Pendulum

```
สมการการเคลื่อนที่ของ Pendulum:
  θ̈ + 2ζωₙθ̇ + ωₙ²θ = -1/L × ẍ_trolley

โดย:
  θ = มุมแกว่ง (rad)
  ζ = Damping ratio (~0.02 สำหรับเชือก)
  ωₙ = Natural frequency = √(g/L) rad/s
  L = ความยาวเชือก (m)
  ẍ = Trolley Acceleration (m/s²)

การแก้สมการเพื่อ Anti-Sway:
  ต้องหา Trolley Acceleration Profile ที่ทำให้ θ = 0 ตลอด
```

```pascal
// State Space Sway Observer (Kalman Filter approach simplified)
FUNCTION_BLOCK FB_SwayEstimator
VAR_INPUT
    Trolley_Acc    : REAL;    // m/s² Acceleration จาก Encoder
    Rope_Length    : REAL;    // m
    Sample_Time    : REAL;    // s (0.01 = 10ms)
END_VAR
VAR_OUTPUT
    Sway_Angle     : REAL;    // rad มุมแกว่งที่ประมาณ
    Sway_Rate      : REAL;    // rad/s อัตราการแกว่ง
END_VAR
VAR
    Wn : REAL;   // Natural frequency
    Wd : REAL;   // Damped frequency
    Zeta : REAL := 0.02;
    
    // State variables [angle, angular_velocity]
    x1, x2 : REAL;
    dx1, dx2 : REAL;
END_VAR

IF Rope_Length > 0.1 THEN
    Wn := SQRT(9.81 / Rope_Length);
ELSE
    Wn := 10.0;
END_IF;
Wd := Wn * SQRT(1.0 - Zeta * Zeta);

// State equations:
// dx1/dt = x2
// dx2/dt = -Wn² × x1 - 2ζWn × x2 - (1/L) × Trolley_Acc

dx1 := x2;
dx2 := -(Wn*Wn) * x1 - (2.0 * Zeta * Wn) * x2 - Trolley_Acc / Rope_Length;

// Euler integration
x1 := x1 + dx1 * Sample_Time;
x2 := x2 + dx2 * Sample_Time;

Sway_Angle := x1;
Sway_Rate  := x2;
END_FUNCTION_BLOCK
```

---

## Step 1390-1400: Complete EOT Crane Control System

### Step 1390: Full System Architecture

```
EOT Crane Control System:
┌────────────────────────────────────────────────────────┐
│                  CRANE CONTROL PANEL                   │
│                                                        │
│  Main PLC (Allen-Bradley 1756-L73S Safety)            │
│    ┌─────────────────────────────────────────┐        │
│    │  Safety CPU (GuardLogix)                │        │
│    │  Standard CPU (ControlLogix)            │        │
│    └─────────────────────────────────────────┘        │
│                                                        │
│  I/O Modules:                                          │
│    1756-OB16 → VFD Commands (Long Travel, Cross Travel)│
│    1756-OB16 → VFD Command (Hoist)                    │
│    1756-IB16 → Limit Switches, Safety Inputs          │
│    1756-IF8  → Load Cell, Encoder Position            │
│    1756-OF8  → VFD Speed References (0-10V)           │
│                                                        │
│  Safety I/O:                                           │
│    1791DS-IB8XOB8 → E-Stop, Limit Switches (SIL 2)   │
└────────────────────────────────────────────────────────┘
           ↓ EtherNet/IP ↓
┌────────────────────────────────────────────────────────┐
│                  REMOTE PENDANT/HMI                    │
│    PanelView Plus 7 (15") หรือ Wireless Remote        │
└────────────────────────────────────────────────────────┘
```

### Step 1391: Automatic Storage and Retrieval (AS/RS)

**อัตโนมัติเต็มรูปแบบสำหรับคลังสินค้า:**

```pascal
// AS/RS Crane Position Control
FUNCTION_BLOCK FB_ASRS_Control
VAR_INPUT
    Target_Column  : INT;    // ช่องเก็บ (Column)
    Target_Row     : INT;    // แถวเก็บ (Row)
    Target_Level   : INT;    // ระดับชั้น (Level)
    Operation      : INT;    // 1=Store, 2=Retrieve
END_VAR
VAR
    Target_X       : REAL;   // mm ตำแหน่ง X
    Target_Y       : REAL;   // mm ตำแหน่ง Y
    Target_Z       : REAL;   // mm ตำแหน่ง Z
    
    // Storage Array Dimensions
    Bay_Width      : REAL := 1200.0;  // mm ระหว่างช่อง
    Tier_Height    : REAL := 2000.0;  // mm ระหว่างชั้น
    Bay_Depth      : REAL := 1500.0;  // mm ความลึก
END_VAR

// แปลง Column/Row/Level → X/Y/Z Position
Target_X := (Target_Column - 1) * Bay_Width + Bay_Width / 2.0;
Target_Y := Bay_Depth / 2.0;  // ความลึกเข้าไปใน Rack
Target_Z := (Target_Level - 1) * Tier_Height + Tier_Height / 2.0;

// สั่งให้ Crane เคลื่อนไปยังตำแหน่ง
// (Calls Position Control Function Blocks)
END_FUNCTION_BLOCK
```

---

## Lab Project 70: EOT Crane with Anti-Sway

### โจทย์: ออกแบบระบบควบคุม EOT Crane ขนาด 20 ตัน พร้อม Anti-Sway

**ข้อมูลปั้นจั่น:**
- SWL: 20,000 kg (20 ตัน)
- Bridge Span: 30 เมตร
- Bay Length: 60 เมตร
- Hoist Height: 12 เมตร
- Maximum Speed: Bridge 120m/min, Trolley 60m/min, Hoist 8m/min

**ข้อกำหนดระบบ:**
1. Anti-Sway Control (Input Shaping)
2. Zone Protection (3 Zones)
3. Overload Protection (Warning + RCL + Safety Stop)
4. Pre-Shift Self-Test Sequence
5. Remote Radio Control (Hetronic NOVA 9)
6. Data Recording (Lifts per day, Total Hours, Overload Events)

**Address Table (Allen-Bradley ControlLogix):**

| Tag Name | Data Type | I/O | Address | Description |
|----------|-----------|-----|---------|-------------|
| HOIST_UP_CMD | BOOL | DI | Local:1:I.Data.0 | Pendant ยกขึ้น |
| HOIST_DN_CMD | BOOL | DI | Local:1:I.Data.1 | Pendant ยกลง |
| LT_FWD_CMD | BOOL | DI | Local:1:I.Data.2 | Bridge Forward |
| HOIST_UPPER_LS | BOOL | DI | Local:1:I.Data.8 | Upper Limit |
| LOAD_CELL_AI | INT | AI | Local:2:I.Ch0Data | Load Cell mA |
| HOIST_VFD_SP | REAL | AO | Local:3:O.Ch0Data | VFD Speed Hz |

---

## สรุป Part 70

ใน Part นี้เราได้เรียนรู้:
- ประเภทและโครงสร้างของปั้นจั่น
- VFD Control สำหรับ Hoist พร้อม Anti-Rollback
- Anti-Sway Control ด้วย Input Shaping
- Zone Protection System
- Overload Protection (SWL, RCL, Safety Stop)
- Dual-Stage Limit Switches
- Pre-Shift Automated Testing
- Safety Requirements ตาม ISO/EN Standards

**ทักษะสำคัญ:**
- Safety Function Design
- Dynamic Load Compensation
- Position Control ด้วย Encoder
- AS/RS Automation

---

### 🔗 Navigation
- [← Part 69: HVAC Automation](part69_hvac_automation.md)
- [→ Part 71: Press & Stamping](part71_press_stamping.md)
- [📚 สารบัญหลัก](../README.md)
