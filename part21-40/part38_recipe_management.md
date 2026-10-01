# Part 38: Recipe Management
## Steps 741-760: การจัดการสูตรการผลิตด้วย PLC

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 3-4 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 31 (Data Blocks), Part 36 (SCADA Integration)

---

## บทนำ

Recipe Management หรือการจัดการสูตรการผลิต เป็นหัวใจของการผลิตแบบ Multi-Product การเปลี่ยนสูตรผลิตภัณฑ์อย่างรวดเร็วและถูกต้องช่วยเพิ่ม OEE และลด Waste

ใช้งานในอุตสาหกรรม:
- อาหารและเครื่องดื่ม
- ยา (Pharmaceutical)
- เคมี
- ลาดตาง (Rubber/Plastics)

---

## Step 741: Recipe Concept in Manufacturing

### นิยาม Recipe

```
┌─────────────────────────────────────────────────────────┐
│                  Recipe Concept                         │
│                                                         │
│  Recipe คือชุดของพารามิเตอร์ที่กำหนดว่า:              │
│  - ต้องใช้วัตถุดิบอะไร (Ingredients)                   │
│  - ต้องใช้เท่าไหร่ (Quantities)                        │
│  - กระบวนการทำอะไรบ้าง (Process Steps)                 │
│  - แต่ละขั้นตอนใช้เงื่อนไขอะไร (Conditions)            │
│  - มีลำดับอย่างไร (Sequence)                           │
│                                                         │
│  ตัวอย่าง: Recipe ผลิต Yogurt                          │
│  1. นำนม 1000L ใส่ Tank                               │
│  2. อุ่นที่ 85°C เป็นเวลา 30 นาที (Pasteurize)          │
│  3. เย็นลงที่ 43°C                                      │
│  4. เติม Starter Culture 2%                            │
│  5. บ่มที่ 43°C เป็นเวลา 4-5 ชั่วโมง                   │
│  6. ตรวจสอบ pH (ต้องได้ 4.5-4.7)                       │
│  7. เย็นลงที่ 4°C                                       │
└─────────────────────────────────────────────────────────┘
```

---

## Step 742: ISA-88 Batch Recipe Model

### ลำดับชั้นของ Recipe (ISA-88)

```
┌─────────────────────────────────────────────────────────────┐
│                    ISA-88 Recipe Hierarchy                   │
│                                                             │
│  General Recipe                                             │
│  └── Site Recipe (Factory-specific)                         │
│       └── Master Recipe (Equipment class)                   │
│            └── Control Recipe (Active batch)                │
│                                                             │
│  ─────────────────────────────────────────────────         │
│  Physical Model Hierarchy:                                  │
│  Enterprise → Site → Area → Process Cell → Unit → ...       │
│                                                             │
│  Procedure Hierarchy:                                        │
│  Procedure → Unit Procedure → Operation → Phase             │
│  (Recipe)    (Tank/Vessel)    (Fill/Heat)  (Open Valve)     │
│                                                             │
│  ตัวอย่าง Yogurt:                                          │
│  Procedure: Make Yogurt                                     │
│  Unit Proc 1: Pasteurize (Tank 1)                           │
│    Operation 1: Fill Milk                                   │
│      Phase 1: Open Inlet Valve                             │
│      Phase 2: Wait for Level                               │
│    Operation 2: Heat                                        │
│      Phase 1: Enable Heater                                │
│      Phase 2: Wait for Temperature                          │
│      Phase 3: Hold at Temperature                          │
└─────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Recipe Parameters

```
Master Recipe: Strawberry Yogurt (500L batch)

Formula:
  Fresh Milk      : 450 L
  Cream           : 40 L
  Skim Milk Powder: 7.5 kg
  Sugar           : 50 kg
  Strawberry Base : 25 kg
  Starter Culture : 10 L

Process Parameters:
  Pasteurize_Temp  : 85.0 °C
  Pasteurize_Time  : 30 min
  Cooling1_Temp    : 43.0 °C
  Culture_Temp     : 43.0 °C
  Culture_Hold_Time: 270 min (4.5 hours)
  Target_pH        : 4.6
  Cooling2_Temp    : 4.0 °C
  Batch_Size       : 500 L
```

---

## Step 743: Recipe Storage in PLC DB/DM

### DB Structure สำหรับ Recipe

```
// DB "Recipe_Library" - Global DB

// UDT "UDT_Recipe"
TYPE UDT_Recipe
  VERSION : 0.1
  STRUCT
    Recipe_ID      : INT;         // หมายเลขสูตร
    Recipe_Name    : STRING[30];  // ชื่อสูตร
    Product_Code   : STRING[10];  // รหัสผลิตภัณฑ์
    Version        : INT;         // เวอร์ชัน
    Active         : BOOL;        // ใช้งานได้
    
    // วัตถุดิบ
    Ingredient_A   : REAL;        // kg หรือ L
    Ingredient_B   : REAL;
    Ingredient_C   : REAL;
    Ingredient_D   : REAL;
    Total_Batch    : REAL;        // ขนาด Batch (kg/L)
    
    // พารามิเตอร์กระบวนการ
    Heat_Temp      : REAL;        // อุณหภูมิการอุ่น (°C)
    Heat_Time      : TIME;        // เวลาอุ่น
    Mix_Speed      : INT;         // ความเร็วกวน (RPM)
    Mix_Time       : TIME;        // เวลากวน
    Cool_Temp      : REAL;        // อุณหภูมิระบายความร้อน
    Hold_Time      : TIME;        // เวลาบ่ม
    
    // Quality Spec
    pH_Min         : REAL;        // pH ต่ำสุด
    pH_Max         : REAL;        // pH สูงสุด
    Viscosity_Min  : REAL;        // ความหนืดต่ำสุด
    Viscosity_Max  : REAL;        // ความหนืดสูงสุด
    
    // Modified Info
    Last_Modified  : DATE_AND_TIME; // แก้ไขล่าสุด
    Modified_By    : INT;          // รหัสผู้แก้ไข
  END_STRUCT;
END_TYPE

// DB Structure:
DB "Recipe_Library"
  Recipe_Count    : INT := 0         // จำนวนสูตรที่มี
  Max_Recipes     : INT := 50        // สูตรสูงสุด
  Recipes         : ARRAY[1..50] OF UDT_Recipe
  Active_Recipe_No: INT              // สูตรที่กำลังใช้
```

### Mitsubishi - Recipe ใน D Registers

```
// Mitsubishi MELSEC - ไม่มี DB concept
// ใช้ Data Registers D + Label Programming

// Global Label:
Recipe_Active    : INT    // D100
Recipe_Library   : ARRAY[50] OF Recipe_Struct  // D200-D2499

// ถ้าใช้ D registers แบบดั้งเดิม:
// Recipe 1: D200-D249 (50 registers)
// Recipe 2: D250-D299
// ...ไปจนถึง Recipe 50: D2700-D2749

// การ Load Recipe:
// D100 = Recipe Number ที่เลือก
// Formula: Source Address = D200 + (D100-1)*50
// BMOV D200 D2800 K50  // Copy Recipe 1 to Active Area

// ใน Label Programming:
[BMOV Recipe_Library[Recipe_Active] Active_Recipe K50]
```

---

## Step 744: Recipe Loading/Saving

### การโหลด Recipe ไปยัง Process

```
// FC "Load_Recipe" - Function สำหรับโหลด Recipe

Network 1: ตรวจสอบ Recipe Number Valid

  |                                                         |
  |--[>=]--[<=]----[SET]-----------------------------------|
  |  Recipe_No >= 1  Recipe_No    "Recipe_Valid"            |
  |  <= "Recipe_Library".Recipe_Count                       |
  |                                                         |

Network 2: Copy Recipe Parameters ไปยัง Process Setpoints

  |                                                         |
  |--[P]--[ ]------[MOVE]----------------------------------|
  |  Load  Recipe   IN: "Recipe_Library"                   |
  |  Cmd   Valid    .Recipes[Recipe_No].Heat_Temp           |
  |                 OUT: "Process_DB".Heat_Setpoint         |
  |                                                         |
  |                [MOVE]                                   |
  |                 IN: "Recipe_Library"                   |
  |                 .Recipes[Recipe_No].Mix_Speed           |
  |                 OUT: "Process_DB".Mix_Speed_SP          |
  |                                                         |
  |                [MOVE]                                   |
  |                 IN: "Recipe_Library"                   |
  |                 .Recipes[Recipe_No].Mix_Time            |
  |                 OUT: "Process_DB".Mix_Time_SP           |
  |                                                         |

Network 3: บันทึกการโหลด Recipe

  |                                                         |
  |--[P]----------[MOVE]-----------------------------------|
  |  Load_Cmd      IN: Recipe_No                           |
  |                OUT: "Recipe_Library".Active_Recipe_No  |
  |                                                         |
  |--[P]----------[RD_SYS_T]-------------------------------|
  |  Load_Cmd      OUT: "Batch_DB".Recipe_Load_Time        |
  |                                                         |
```

### การบันทึก Recipe ใหม่

```
// FC "Save_Recipe" - บันทึก Recipe จาก Process Parameters

Network 1: ตรวจสอบสิทธิ์ (Engineer Level)

  |                                                         |
  |--[=]----------[SET]-------------------------------------|
  |  "User_Level"  "Save_Authorized"                        |
  |  = 3  (Engineer)                                        |
  |                                                         |

Network 2: บันทึก Parameters ลง Recipe Slot ที่เลือก

  |                                                         |
  |--[P]--[ ]------[MOVE]----------------------------------|
  |  Save  Auth     IN: "Process_DB".Heat_Setpoint         |
  |  Cmd            OUT: "Recipe_Library"                  |
  |                 .Recipes[Save_Slot].Heat_Temp           |
  |                                                         |
  |                [MOVE]                                   |
  |                 IN: "Process_DB".Mix_Speed_SP           |
  |                 OUT: Recipes[Save_Slot].Mix_Speed       |
  |                                                         |

Network 3: อัพเดต Recipe Info

  |                                                         |
  |--[P]--[ ]------[RD_SYS_T]-----------------------------|
  |  Save  Auth     OUT: Recipes[Save_Slot].Last_Modified  |
  |                                                         |
  |                [MOVE]                                   |
  |                 IN: "Current_User_ID"                  |
  |                 OUT: Recipes[Save_Slot].Modified_By    |
  |                                                         |
  |                [ADD]                                    |
  |                 IN1: Recipes[Save_Slot].Version        |
  |                 IN2: 1                                  |
  |                 OUT: Recipes[Save_Slot].Version        |
  |                 (เพิ่ม Version Number)                  |
  |                                                         |
```

---

## Step 745: HMI Recipe Selection

### HMI Recipe Screen

```
Recipe Management HMI Screen:
┌─────────────────────────────────────────────────────────┐
│                Recipe Management                        │
│                                                         │
│  Recipe Library:            Active Recipe:              │
│  ┌──────────────────────┐  ┌───────────────────────┐  │
│  │ # │ Name     │ Ver  │  │ Recipe #3: Choc Yogurt │  │
│  │───│──────────│──────│  ├───────────────────────-┤  │
│  │ 1 │ Plain    │  2   │  │ Milk:    400L           │  │
│  │ 2 │ Strawbry │  5   │  │ Cream:    45L           │  │
│  │►3 │ Chocolate│  3   │  │ Powder:  10.0 kg        │  │
│  │ 4 │ Bluebrry │  1   │  │ Sugar:   55.0 kg        │  │
│  │ 5 │ Mango    │  4   │  │ Choc Base: 30.0 kg      │  │
│  └──────────────────────┘  └───────────────────────┘  │
│                                                         │
│  [Load Selected] [Save Current] [Edit] [Print]          │
│                                                         │
│  Status: Recipe #3 Loaded at 14:30:25                  │
└─────────────────────────────────────────────────────────┘
```

### HMI ป้อนค่า Recipe

```
WinCC Script สำหรับ Load Recipe จาก HMI:

// VBScript (WinCC Classic)
Sub LoadRecipe_Click()
    Dim recipeNo As Integer
    recipeNo = SmartTags("HMI_Recipe_Select")
    
    ' ตรวจสอบ Range
    If recipeNo < 1 Or recipeNo > 50 Then
        MsgBox "หมายเลขสูตร ต้องอยู่ระหว่าง 1-50"
        Exit Sub
    End If
    
    ' ตั้งค่าใน PLC
    SmartTags("PLC_Load_Recipe_No") = recipeNo
    SmartTags("PLC_Load_Recipe_Cmd") = True
    
    ' รอ 2 วินาที
    Wait 2000
    
    ' Reset Cmd
    SmartTags("PLC_Load_Recipe_Cmd") = False
    
    ' อัพเดต Display
    updateRecipeDisplay(recipeNo)
End Sub
```

---

## Step 746: Multiple Product Changeover

### Changeover Sequence

```
Product Changeover มีขั้นตอน:
1. แจ้งสิ้นสุด Production Product เก่า
2. CIP (Clean In Place) - ล้างอุปกรณ์
3. ตรวจสอบความสะอาด
4. Load Recipe ใหม่
5. ตรวจสอบ Materials (วัตถุดิบพร้อม)
6. Start Production ผลิตภัณฑ์ใหม่
```

### Ladder - Changeover Management

```
// States: 0=Production, 1=Changeover, 2=CIP, 3=Verification

Network 1: เริ่ม Changeover

  |                                                         |
  |--[P]--[ ]------[MOVE]----------------------------------|
  |  Change  Prod   IN: 1 (Changeover State)              |
  |  Cmd     Stop   OUT: "Changeover_DB".State            |
  |                                                         |

Network 2: CIP Phase

  |                                                         |
  |--[=]----------[FC_CIP_Control]-------------------------|
  |  State = 2     CIP Tank 1                              |
  |               CIP Tank 2                               |
  |               Done: "CIP_Complete"                     |
  |                                                         |
  |--[=]--[ ]------[MOVE]----------------------------------|
  |  State  CIP     IN: 3 (Verification)                  |
  |  = 2    Complete OUT: State                           |
  |                                                         |

Network 3: Load New Recipe หลัง Verify

  |                                                         |
  |--[=]--[ ]------[FC_Load_Recipe]------------------------|
  |  State  Verify  Recipe_No: "New_Product_Recipe"        |
  |  = 3    OK      Done: "Recipe_Loaded"                  |
  |                                                         |
```

---

## Step 747: Recipe Version Control

### Version Management

```
Recipe Version Control:
┌─────────────────────────────────────────────────────────┐
│  Version History สำหรับแต่ละ Recipe:                    │
│                                                         │
│  Recipe #3 - Chocolate Yogurt:                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Ver │ Date       │ Changed By │ Change Description│  │
│  ├──────────────────────────────────────────────────┤  │
│  │ 1   │ 2023-01-15 │ Smith      │ Initial create    │  │
│  │ 2   │ 2023-03-20 │ Jones      │ +Sugar 5kg/batch  │  │
│  │ 3   │ 2023-06-10 │ Smith      │ Adjust Heat Temp  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Rollback ไป Version 2 ได้ถ้าจำเป็น                   │
└─────────────────────────────────────────────────────────┘
```

### DB Structure สำหรับ Version History

```
DB "Recipe_History"
  History_Count: INT
  History: ARRAY[1..500] OF STRUCT
    Recipe_ID    : INT
    Version      : INT
    Timestamp    : DATE_AND_TIME
    Changed_By   : INT
    Change_Desc  : STRING[50]
    
    // Snapshot ของ Recipe ณ Version นั้น
    Heat_Temp    : REAL
    Mix_Speed    : INT
    Mix_Time     : TIME
    Ingredient_A : REAL
    Ingredient_B : REAL
    // ... พารามิเตอร์ทั้งหมด
  END_STRUCT
```

---

## Step 748: Recipe Security

### User Level สำหรับ Recipe

```
Security Levels:
┌─────────────────────────────────────────────────────────┐
│  Level 1 - Operator:                                    │
│  ✓ เลือกและ Load Recipe                                  │
│  ✗ แก้ไข Recipe Parameters                              │
│  ✗ สร้าง Recipe ใหม่                                    │
│  ✗ ลบ Recipe                                            │
│                                                         │
│  Level 2 - Shift Supervisor:                            │
│  ✓ ทุกอย่างของ Level 1                                  │
│  ✓ ปรับ Parameter เล็กน้อย (±10%)                      │
│  ✗ สร้าง/ลบ Recipe                                      │
│                                                         │
│  Level 3 - Process Engineer:                            │
│  ✓ ทุกอย่างของ Level 2                                  │
│  ✓ สร้าง/แก้ไข/ลบ Recipe เต็มรูปแบบ                    │
│                                                         │
│  Level 4 - Quality Manager:                             │
│  ✓ ทุกอย่าง + อนุมัติ Recipe ใหม่                       │
└─────────────────────────────────────────────────────────┘
```

### Password Protection Ladder

```
Network 1: ตรวจสอบ Password ก่อน Save Recipe

  |                                                         |
  |--[P]----------[CMP_EQ]--------------------------------|
  |  "Save_       IN1: "HMI_Password"                     |
  |   Attempt"    IN2: "System_DB".Engineer_Password      |
  |               OUT: "Password_OK"                      |
  |               (เปรียบเทียบ STRING)                     |
  |                                                         |
  |--[ ]-----------[SET]-----------------------------------|
  |  Password_OK   "Save_Authorized"                       |
  |                                                         |
  |--[NOT]---------[SET]-----------------------------------|
  |  Password_OK   "Password_Wrong_Alarm"                  |
  |                                                         |
  |--[ ]----------[INC]--------------------------------------|
  |  Password_Wrong "Failed_Login_Count"                   |
  |                                                         |

Network 2: Lock หลังพยายาม 3 ครั้ง

  |                                                         |
  |--[>=]---------[SET]-------------------------------------|
  |  Failed_Count >= 3  "System_Locked"                    |
  |                                                         |
  |--[ ]----------[TON]--------------------------------------|
  |  System_Locked  T_Lockout                              |
  |                 PT: T#30m (ล็อก 30 นาที)               |
  |                                                         |
  |--[ ]----------[RESET]----------------------------------|
  |  T_Lockout.Q   "System_Locked"                         |
  |               "Failed_Login_Count" (= 0)               |
  |                                                         |
```

---

## Step 749: Ingredient Addition Sequence

### Weighing and Dosing System

```
// ระบบชั่งและเติมส่วนผสม

// DB "Dosing_Control":
Step_Active      : INT       // 0=Idle, 1-N=Active Step
Target_Weight_A  : REAL      // น้ำหนักเป้าหมาย A
Target_Weight_B  : REAL
Actual_Weight_A  : REAL      // น้ำหนักจริงที่ชั่งได้
Actual_Weight_B  : REAL
Weight_Tolerance : REAL := 0.5  // ±0.5 kg
Dosing_Error     : BOOL
Pre_Dose_Cutoff  : REAL := 2.0  // ปิดวาล์วก่อน 2kg

// State Machine สำหรับ Dosing:
// 0: Idle
// 1: Tare Scale (ตั้งศูนย์)
// 2: Open Valve A (เริ่มเติม A)
// 3: Pre-close Valve A (ช้าลงก่อนถึง Target)
// 4: Close Valve A (ปิดเมื่อถึง Target)
// 5: Check Weight A
// 6-10: Repeat for B, C, D
// 11: Done
```

### Ladder - Dosing Sequence

```
Network 1: State 1 - Tare Scale

  |                                                         |
  |--[=]----------[FC_Tare_Scale]--------------------------|
  |  Step = 1      Scale_Module: 1                         |
  |               Done: "Tare_Done"                        |
  |                                                         |
  |--[=]--[ ]------[MOVE]----------------------------------|
  |  Step   Tare    IN: 2 (Open Valve A)                  |
  |  = 1    Done    OUT: Step_Active                       |
  |                                                         |

Network 2: State 2 - Open Valve A (เติมเร็ว)

  |                                                         |
  |--[=]----------[SET]-------------------------------------|
  |  Step = 2      "Valve_A_Fast" (Q0.0)                   |
  |               (เปิดวาล์วเร็ว)                          |
  |                                                         |
  |  ตรวจสอบว่าใกล้ Target:
  |--[=]--[>=]-----[MOVE]----------------------------------|
  |  Step   Weight  IN: 3 (Pre-close mode)                 |
  |  = 2    >= Target-  OUT: Step_Active                  |
  |         Pre_Cutoff                                      |
  |                                                         |

Network 3: State 3 - Pre-close (เติมช้า)

  |                                                         |
  |--[=]----------[RESET]-[SET]------------------------------|
  |  Step = 3      Valve_A_Fast  Valve_A_Slow               |
  |               (Q0.0 off)     (Q0.1 on)                 |
  |                                                         |
  |--[=]--[>=]-----[MOVE]----------------------------------|
  |  Step   Weight  IN: 4 (Close)                         |
  |  = 3    >= Target  OUT: Step_Active                    |
  |                                                         |

Network 4: State 4 - Close and Verify

  |                                                         |
  |--[=]----------[RESET]-[RESET]--------------------------|
  |  Step = 4      Valve_A_Fast  Valve_A_Slow              |
  |               (ปิดทั้งคู่)                              |
  |                                                         |
  |--[=]----------[TON]--------------------------------------|
  |  Step = 4      T_Settle                                 |
  |               PT: T#2s (รอให้น้ำหนักนิ่ง)              |
  |                                                         |

Network 5: ตรวจสอบน้ำหนัก

  |                                                         |
  |--[=]--[ ]--[>=]--[<=]--[MOVE]------------------------|
  |  Step   T_   Weight >= Target-Tol  IN: 5 (Next)        |
  |  = 4   Settle  Weight <= Target+Tol  OUT: Step        |
  |                                                         |
  |--[=]--[ ]--[NOT_IN_RANGE]--[SET]----------------------|
  |  Step   Settle  (Weight out of tolerance)              |
  |  = 4             "Dosing_Weight_Error"                 |
  |                   Step = 99 (Error state)              |
  |                                                         |
```

---

## Step 750: Complete Recipe System for Food Production

### ระบบสมบูรณ์: Yogurt Production

```
// ============================================
// COMPLETE YOGURT PRODUCTION RECIPE SYSTEM
// ============================================
// Equipment:
// - Mix Tank (1000L capacity, Agitator, Heater, Cooler)
// - Ingredient Hoppers: A(Milk), B(Cream), C(Powder), D(Culture)
// - Load Cells (Scale)
// - Temperature Sensor (RTD PT100)
// - pH Meter (Analog 4-20mA)
// - Agitator with VFD
// ============================================

// Production Sequence:
// Phase 1: Preparation
//   1.1 CIP Complete Check
//   1.2 Tare Scale
//   1.3 Load Recipe Parameters

// Phase 2: Ingredient Addition (Dosing)
//   2.1 Dose Milk (A)
//   2.2 Dose Cream (B)
//   2.3 Dose Skim Milk Powder (C) - Manual Add
//   2.4 Dose Sugar (D)

// Phase 3: Pasteurization
//   3.1 Start Agitator (100 RPM)
//   3.2 Heat to 85°C
//   3.3 Hold at 85°C for 30 minutes
//   3.4 Cool to 43°C

// Phase 4: Culture Addition
//   4.1 Start Slow Agitator (30 RPM)
//   4.2 Dose Starter Culture
//   4.3 Mix for 5 minutes

// Phase 5: Incubation
//   5.1 Stop Agitator
//   5.2 Hold at 43°C
//   5.3 Monitor pH every 30 minutes
//   5.4 End when pH reaches Target (4.5-4.7)

// Phase 6: Cooling and Transfer
//   6.1 Cool to 4°C
//   6.2 Quality Check (pH, Viscosity)
//   6.3 Transfer to Storage Tank

// State Machine Implementation:
// =============================
// Step 0: IDLE
// Step 1-4: Dosing Sequence (A,B,C,D)
// Step 10: Heat to 85°C
// Step 11: Hold 85°C (30 min)
// Step 12: Cool to 43°C
// Step 20: Culture Dose
// Step 21: Mix 5 min
// Step 30: Incubation (Hold 43°C + Monitor pH)
// Step 40: Cool to 4°C
// Step 50: QC Check
// Step 60: Transfer
// Step 99: COMPLETE
// Step 100: ERROR
```

### PLC Ladder - Phase 3 (Pasteurization)

```
// Step 10: Heat to 85°C
Network: Heat Phase

  |                                                         |
  |--[=]----------[FC_PID_Control]-------------------------|
  |  Step = 10     Setpoint: "Recipe_DB"                   |
  |  OR             .Recipes[Active].Heat_Temp             |
  |  Step = 11     Process: "Tank_Temp_PV"                 |
  |               Output: "Heater_Output"                  |
  |               Kp: 2.0, Ki: 0.1, Kd: 0.0               |
  |               Running: "PID_Running"                   |
  |                                                         |
  |--[=]--[>=]-----[MOVE]-[RD_SYS_T]----------------------|
  |  Step   Temp    IN: 11  Timestamp: "Hold_Start_Time"   |
  |  = 10   >= 85°C OUT: Step                             |
  |                                                         |

// Step 11: Hold 85°C for 30 min
Network: Hold Phase

  |                                                         |
  |--[=]----------[TON]--------------------------------------|
  |  Step = 11     T_Hold_Timer                             |
  |               PT: "Recipe_DB".Recipes[Active].Heat_Time|
  |                                                         |
  |--[=]--[ ]------[MOVE]----------------------------------|
  |  Step   T_Hold  IN: 12 (Cool Phase)                   |
  |  = 11   .Q      OUT: Step                             |
  |                                                         |

// Step 12: Cool to 43°C
Network: Cooling Phase

  |                                                         |
  |--[=]----------[SET]-------------------------------------|
  |  Step = 12     "Cooling_Valve" (Q0.4)                  |
  |               (เปิดวาล์วน้ำเย็น)                        |
  |                                                         |
  |--[=]--[<=]-----[MOVE]-[RESET]--------------------------|
  |  Step   Temp    IN: 20  "Cooling_Valve"                |
  |  = 12   <= 43°C OUT: Step  (ปิดน้ำเย็น)              |
  |                                                         |
```

---

## Step 751-760: Lab Projects

### Lab 1: Recipe Library System

**วัตถุประสงค์:** สร้างระบบจัดการสูตรสมบูรณ์

**โจทย์:**
```
1. สร้าง DB ที่เก็บ Recipe ได้ 20 สูตร
2. แต่ละ Recipe มี 10 Parameters
3. สร้าง Function สำหรับ:
   - Load Recipe
   - Save Recipe
   - Validate Parameters
   - Increment Version

4. HMI Screen:
   - Recipe List (แสดง 20 สูตร)
   - Recipe Detail (แสดง Parameters)
   - Load/Save Buttons
   - Status Messages
```

### Lab 2: Dosing Sequence

**วัตถุประสงค์:** ระบบชั่งและเติมส่วนผสม 4 ชนิด

```
Equipment Simulation:
- Scale: Simulated with Counter
- Valve A: Q0.0 (Fast), Q0.1 (Slow)
- Valve B: Q0.2, Q0.3
- Valve C: Q0.4, Q0.5
- Valve D: Q0.6, Q0.7
- Weight: MW100 (simulated)

Recipe:
  Ingredient A: 400.0 kg (±0.5)
  Ingredient B: 45.0 kg (±0.2)
  Ingredient C: 10.0 kg (±0.1) [Manual]
  Ingredient D: 10.0 L (±0.1)

ต้องการ:
1. Automatic Dosing Sequence
2. Pre-cutoff 2kg ก่อน Target
3. Weight Verification (±Tolerance)
4. Error Handling (Over/Under weight)
5. Auto Retry เมื่อ Under Weight
```

### Lab 3: Complete Batch System

```
โจทย์: ระบบ Batch ครบวงจรสำหรับ Paint Manufacturing

Recipe:
- Resin A: 500 kg
- Pigment B: 50 kg  
- Solvent C: 100 L
- Additive D: 5 kg

Process Steps:
1. Load Ingredients A, B, C
2. Mix at 200 RPM 30 minutes
3. Heat to 60°C (for better mixing)
4. Cool to 30°C
5. Add Additive D (slow mix 50 RPM)
6. Mix 15 minutes
7. Sample & QC (Viscosity, Color)
8. Transfer if OK

ต้องการ:
1. Full State Machine (20+ states)
2. Recipe Library (5 สูตรสี)
3. Production History (100 records)
4. Alarm Integration
5. Batch Report
```

---

## สรุป Part 38

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Recipe Concept | ความหมาย, การใช้งาน |
| ISA-88 | Batch Recipe Model Hierarchy |
| DB Storage | UDT, ARRAY Structure |
| Load/Save | Dynamic Recipe Loading |
| HMI Selection | Screen Design |
| Changeover | Sequence Management |
| Version Control | History Tracking |
| Security | User Level, Password |
| Dosing | Weighing Sequence |
| Complete System | Food Production Example |

---

## การนำทาง

- [← Part 37: Alarm Management](part37_alarm_management.md)
- [→ Part 39: Energy Monitoring](part39_energy_monitoring.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 38 of 40 | Steps 741-760*
