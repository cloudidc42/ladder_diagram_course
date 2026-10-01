# Part 31: Data Blocks (DB) - บล็อกข้อมูล
## Steps 601-620: การใช้งาน Data Blocks ในการเขียนโปรแกรม PLC

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 3-4 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 29 (Structured Programming), Part 30 (Function Blocks)

---

## บทนำ

Data Block (DB) คือพื้นที่หน่วยความจำที่มีโครงสร้าง ใช้สำหรับเก็บข้อมูลในโปรแกรม PLC Siemens S7 เป็นวิธีที่มีประสิทธิภาพในการจัดการข้อมูลเป็นกลุ่ม เช่น สูตรการผลิต, ข้อมูลผลิตภัณฑ์, ค่าพารามิเตอร์เครื่องจักร

ในระบบ Allen-Bradley จะใช้ UDT (User Defined Type) แทน ส่วน Mitsubishi จะใช้ D registers array หรือ Extension Data Register

---

## Step 601: Global DB vs Instance DB

### Global Data Block

Global DB คือ Data Block ที่สร้างอิสระ ไม่ผูกกับ Function Block ใดๆ ทุก OB, FC, FB สามารถเข้าถึงได้

```
┌─────────────────────────────────────────────────────────┐
│                    Global DB Structure                   │
│                                                         │
│  DB1 "Recipe_Data"                                      │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Name          │ Type    │ Value  │ Comment        │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ Recipe_Number │ INT     │ 1      │ หมายเลขสูตร   │  │
│  │ Temperature   │ REAL    │ 85.0   │ อุณหภูมิ(°C)  │  │
│  │ Pressure      │ REAL    │ 2.5    │ ความดัน(bar)  │  │
│  │ Time_Mix      │ TIME    │ T#30s  │ เวลาผสม       │  │
│  │ Product_Name  │ STRING  │ 'A001' │ ชื่อผลิตภัณฑ์ │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  เข้าถึงได้จากทุกที่: "Recipe_Data".Temperature        │
└─────────────────────────────────────────────────────────┘
```

### Instance Data Block

Instance DB ถูกสร้างอัตโนมัติเมื่อเรียกใช้ Function Block มีข้อมูลเฉพาะของ FB นั้น

```
┌─────────────────────────────────────────────────────────┐
│                   Instance DB Structure                  │
│                                                         │
│  FB10 "PID_Controller" → DB10 (Instance DB)             │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Name      │ Type  │ Value │ Section   │ Comment  │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ Setpoint  │ REAL  │ 75.0  │ IN        │ ค่าตั้ง  │  │
│  │ ProcessV  │ REAL  │ 73.5  │ IN        │ ค่าจริง  │  │
│  │ Output    │ REAL  │ 45.2  │ OUT       │ เอาต์พุต │  │
│  │ Kp        │ REAL  │ 2.0   │ IN_OUT    │ ค่า Kp   │  │
│  │ Integral  │ REAL  │ 0.0   │ STAT      │ ค่าสะสม  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  เข้าถึงได้: "PID_Controller_DB".Setpoint               │
└─────────────────────────────────────────────────────────┘
```

### การเปรียบเทียบ

| คุณสมบัติ | Global DB | Instance DB |
|-----------|-----------|-------------|
| สร้างโดย | ผู้เขียนโปรแกรม | ระบบอัตโนมัติ |
| ผูกกับ FB | ไม่ | ใช่ |
| แชร์ได้ | ทุก Block | เฉพาะ FB นั้น |
| ใช้กับ | ข้อมูลทั่วไป | สถานะของ FB |

---

## Step 602: ARRAY Data Type

### การประกาศ ARRAY ใน Data Block

```
DB5 "Sensor_Array"

Name           | Type              | Comment
---------------|-------------------|------------------
Temperature    | ARRAY[1..10] OF REAL | อุณหภูมิ 10 จุด
Status         | ARRAY[1..10] OF BOOL | สถานะเซ็นเซอร์
Calibration    | ARRAY[1..10] OF REAL | ค่าปรับเทียบ
```

### Ladder Diagram - การอ่าน/เขียน ARRAY

```
Network 1: อ่านอุณหภูมิเซ็นเซอร์ที่ 1

  |                                          |
  |----[ ]----------[MOVE]-------------------|
  |  "Enable"        IN: IW64              |
  |                  OUT: "Sensor_Array"    |
  |                       .Temperature[1]  |
  |                                          |

Network 2: คำนวณอุณหภูมิเฉลี่ย (โดยใช้ SCL ร่วมกัน)

  |                                          |
  |----[ ]----------[ADD]-------------------|
  |  "Calc"          IN1: "Sensor_Array"    |
  |                       .Temperature[1]  |
  |                  IN2: "Sensor_Array"    |
  |                       .Temperature[2]  |
  |                  OUT: "Temp_Sum"        |
  |                                          |
```

### ตัวอย่าง SCL สำหรับวนซ้ำ ARRAY

```pascal
// การวนซ้ำเพื่อคำนวณค่าเฉลี่ย - ภาษา SCL
FOR i := 1 TO 10 DO
    // รวมค่าอุณหภูมิทั้งหมด
    "Sensor_Array".Temperature_Sum := 
        "Sensor_Array".Temperature_Sum + 
        "Sensor_Array".Temperature[i];
END_FOR;

// หารเพื่อหาค่าเฉลี่ย
"Sensor_Array".Temperature_Avg := 
    "Sensor_Array".Temperature_Sum / 10.0;
```

### ARRAY หลายมิติ

```
DB6 "Matrix_Data"

Name        | Type                    | Comment
------------|-------------------------|------------------
Grid_Values | ARRAY[1..5, 1..5] OF INT | ตาราง 5x5
Zone_Temps  | ARRAY[1..3, 1..4] OF REAL | อุณหภูมิโซน 3x4
```

การเข้าถึง: `"Matrix_Data".Grid_Values[2, 3]`

---

## Step 603: STRUCT Data Type

### การสร้าง STRUCT ใน Data Block

STRUCT ช่วยจัดกลุ่มข้อมูลที่เกี่ยวข้องกัน

```
DB7 "Machine_Status"

STRUCT Machine
  Running    : BOOL;        // เครื่องทำงาน
  Speed      : INT;         // ความเร็ว RPM
  Temperature: REAL;        // อุณหภูมิ °C
  Vibration  : REAL;        // การสั่น mm/s
  Error_Code : WORD;        // รหัสข้อผิดพลาด
  Last_Start : DATE_AND_TIME; // เวลาเริ่มทำงานล่าสุด
END_STRUCT;

Name           | Type         | Comment
---------------|--------------|------------------
Motor_1        | STRUCT Machine | มอเตอร์ 1
Motor_2        | STRUCT Machine | มอเตอร์ 2  
Motor_3        | STRUCT Machine | มอเตอร์ 3
```

### Ladder - อ่าน STRUCT

```
Network 1: ตรวจสอบอุณหภูมิมอเตอร์

  |                                                    |
  |--[>]----------[SET]-------------------------------|
  |  "Machine_Status".Motor_1.Temperature > 80.0       |
  |               "Machine_Status".Motor_1.Error_Code  |
  |               := W#16#0001 (Overheat Alarm)        |
  |                                                    |

Network 2: รีเซ็ตสถานะหลังแก้ไข

  |                                                    |
  |--[ ]--[ ]------[MOVE]----------------------------|
  |  Reset  NOT     IN: W#16#0000                     |
  |         Running  OUT: "Machine_Status"            |
  |                       .Motor_1.Error_Code         |
  |                                                    |
```

### STRUCT ซ้อน STRUCT

```
DB8 "Production_Line"

STRUCT Conveyor_Data
  Speed     : REAL;      // ความเร็ว m/min
  Length    : REAL;      // ความยาว m
  Product   : STRUCT
    Count    : DINT;    // จำนวนผลิตภัณฑ์
    Weight   : REAL;    // น้ำหนัก kg
    Quality  : INT;     // เกรดคุณภาพ (1-5)
  END_STRUCT;
END_STRUCT;
```

---

## Step 604: UDT (User Defined Type)

### การสร้าง UDT ใน TIA Portal

UDT ช่วยให้นำ STRUCT มาใช้ซ้ำได้ในหลาย DB

```
// UDT_Motor_Data - สร้างใน PLC Data Types
TYPE "UDT_Motor_Data"
VERSION : 0.1
   STRUCT
      Running      : BOOL;       // กำลังทำงาน
      Forward      : BOOL;       // หมุนไปข้างหน้า
      Fault        : BOOL;       // มีข้อผิดพลาด
      Speed_Actual : REAL;       // ความเร็วจริง RPM
      Speed_Set    : REAL;       // ความเร็วที่ตั้ง RPM
      Current      : REAL;       // กระแสไฟ A
      Temperature  : REAL;       // อุณหภูมิ °C
      Run_Hours    : REAL;       // ชั่วโมงทำงาน
      Fault_Code   : WORD;       // รหัสข้อผิดพลาด
   END_STRUCT;
END_TYPE
```

### การใช้ UDT ใน DB

```
DB10 "All_Motors"

Name      | Type              | Comment
----------|-------------------|------------------
Motor_001 | "UDT_Motor_Data"  | มอเตอร์ปั๊มน้ำ 1
Motor_002 | "UDT_Motor_Data"  | มอเตอร์ปั๊มน้ำ 2
Motor_003 | "UDT_Motor_Data"  | มอเตอร์พัดลม
Motor_004 | "UDT_Motor_Data"  | มอเตอร์สายพาน
```

### Allen-Bradley UDT

ใน Studio 5000 (Allen-Bradley):

```
// สร้าง User Defined Data Type
Data Type: Motor_Data_T
Members:
  Running    : BOOL
  Forward    : BOOL
  Fault      : BOOL
  Speed      : REAL
  Current    : REAL
  Temperature: REAL
  FaultCode  : DINT
```

การใช้งานใน Tag Database:
```
Motor_Pump1    : Motor_Data_T
Motor_Pump2    : Motor_Data_T
Motor_Fan1     : Motor_Data_T
```

เข้าถึงด้วย: `Motor_Pump1.Speed`

### Mitsubishi MELSEC - Data Register Array

ใน Mitsubishi GX Works3 ใช้ Label Programming:

```
// การกำหนด Structure
Structure Name: Motor_Struct
  Running    : Bit
  Forward    : Bit
  Fault      : Bit
  Speed      : Float (Single)
  Current    : Float (Single)
  Temperature: Float (Single)

// การประกาศตัวแปร Array
Motor[4]     : Motor_Struct  // Array ของมอเตอร์ 4 ตัว
```

หรือใช้แบบ D registers ดั้งเดิม:
```
D100 - D109: Motor_1 Parameters (10 registers)
D110 - D119: Motor_2 Parameters
D120 - D129: Motor_3 Parameters
D130 - D139: Motor_4 Parameters
```

---

## Step 605: การอ่านเขียน DB ใน Ladder Diagram

### การเขียนค่าลง DB

```
Network 1: บันทึกอุณหภูมิเมื่อมีการเปลี่ยนแปลงมาก

  |                                              |
  |--[>]----------[MOVE]------------------------|
  |  "Temp_Delta" > 5.0                          |
  |               IN: MW100 (ค่าอุณหภูมิปัจจุบัน) |
  |               OUT: "Process_DB".Temperature  |
  |                                              |

Network 2: เพิ่มตัวนับการผลิต

  |                                              |
  |--[P]----------[ADD]-------------------------|
  |  "Part_Detect"                               |
  |               IN1: "Product_DB".Count_Total  |
  |               IN2: 1                         |
  |               OUT: "Product_DB".Count_Total  |
  |                                              |

Network 3: บันทึกเวลาเริ่มกะการผลิต

  |                                              |
  |--[P]----------[RD_SYS_T]-------------------|
  |  "Shift_Start"                               |
  |               OUT: "Production_DB"           |
  |                    .Shift_Start_Time         |
  |                                              |
```

### การอ่านค่าจาก DB

```
Network 4: อ่านค่า Setpoint จาก Recipe DB

  |                                              |
  |--[ ]----------[MOVE]------------------------|
  |  "Load_Recipe"                               |
  |               IN: "Recipe_DB".Temperature_SP |
  |               OUT: "PID_Setpoint"            |
  |                                              |

Network 5: เปรียบเทียบค่ากับ Limit ใน DB

  |                                              |
  |--[>=]--[ ]----[SET]-------------------------|
  |  "Temp"       NOT                            |
  |  >=           "Alarm_ACK"                   |
  |  "Limits_DB"  "Alarm_Temp"                  |
  |  .Temp_Max                                   |
  |                                              |
```

### ตัวอย่างสมบูรณ์: Process Control with DB

```
// DB1 "Process_Parameters" - Global DB
Structure:
  Temp_Setpoint  : REAL := 75.0;   // อุณหภูมิที่ต้องการ
  Temp_High_Limit: REAL := 90.0;   // ขีดจำกัดสูง
  Temp_Low_Limit : REAL := 60.0;   // ขีดจำกัดต่ำ
  Pressure_SP    : REAL := 3.5;    // ความดันที่ต้องการ
  Mix_Time       : TIME := T#2m;   // เวลาผสม
  Batch_Count    : DINT := 0;      // จำนวน Batch

// Ladder Network สำหรับ Temperature Control
Network 1: Temperature Alarm High

  |                                                    |
  |--[>=]-------------------------[SET]---------------|
  |  "Temp_Actual" >= "Process_Parameters".Temp_High   |
  |                   [S] "Temp_High_Alarm"            |
  |                                                    |

Network 2: Temperature Alarm Low

  |                                                    |
  |--[<=]-------------------------[SET]---------------|
  |  "Temp_Actual" <= "Process_Parameters".Temp_Low    |
  |                   [S] "Temp_Low_Alarm"             |
  |                                                    |
```

---

## Step 606: DB สำหรับระบบ Recipe

### โครงสร้าง Recipe DB

```
DB20 "Recipe_Storage"

// สูตรผลิตภัณฑ์ประเภทอาหาร
Name              | Type                    | Comment
------------------|-------------------------|------------------
Active_Recipe_No  | INT                     | หมายเลขสูตรใช้งาน
Total_Recipes     | INT := 10               | จำนวนสูตรทั้งหมด
Recipes           | ARRAY[1..10] OF STRUCT  | ข้อมูลสูตร 10 สูตร

// STRUCT สำหรับแต่ละ Recipe
  Recipe_Name   : STRING[20];  // ชื่อสูตร
  Temp_Zone1    : REAL;        // อุณหภูมิโซน 1
  Temp_Zone2    : REAL;        // อุณหภูมิโซน 2
  Temp_Zone3    : REAL;        // อุณหภูมิโซน 3
  Speed_Belt    : REAL;        // ความเร็วสายพาน
  Time_Dwell    : TIME;        // เวลาค้างอยู่
  Ingredient_A  : REAL;        // วัตถุดิบ A (kg)
  Ingredient_B  : REAL;        // วัตถุดิบ B (kg)
  Ingredient_C  : REAL;        // วัตถุดิบ C (kg)
  Batch_Size    : REAL;        // ขนาด Batch (kg)
```

### Ladder - โหลด Recipe

```
Network 1: เลือก Recipe ที่ต้องการ (จาก HMI)

  |                                                         |
  |--[P]----------[MOVE]------------------------------------|
  |  "HMI_Load_  IN: "HMI_Recipe_Select"                   |
  |   Recipe"    OUT: "Recipe_Storage".Active_Recipe_No     |
  |                                                         |

Network 2: โหลดพารามิเตอร์จาก Recipe ที่เลือก

  |                                                         |
  |--[P]--            --[MOVE]------------------------------|
  |  Load   [INT_TO_INT]         IN: Recipes[Active_No]     |
  |         "Active_Recipe_No"        .Temp_Zone1           |
  |                               OUT: "Zone1_Setpoint"     |
  |                                                         |
  |                              [MOVE]--                   |
  |                               IN: Recipes[Active_No]    |
  |                                    .Temp_Zone2          |
  |                               OUT: "Zone2_Setpoint"     |
  |                                                         |

Network 3: แสดงสถานะ Recipe ที่โหลด

  |                                                         |
  |--[ ]----------[MOVE]------------------------------------|
  |  "Recipe_  IN: "Recipe_Storage"                         |
  |   Loaded"      .Recipes[Active_No].Recipe_Name          |
  |              OUT: "HMI_Active_Recipe_Name"               |
  |                                                         |
```

### ตัวอย่าง: Recipe สำหรับสายการผลิตขนมปัง

```
Recipe #1: "White Bread"
  Temp_Zone1: 180°C  (โซนอุ่น)
  Temp_Zone2: 220°C  (โซนอบ)
  Temp_Zone3: 200°C  (โซนสุก)
  Speed_Belt: 0.5 m/min
  Time_Dwell: T#12m
  Ingredient_A (แป้ง): 10.0 kg
  Ingredient_B (ยีสต์): 0.2 kg
  Ingredient_C (เกลือ): 0.15 kg

Recipe #2: "Whole Wheat"
  Temp_Zone1: 185°C
  Temp_Zone2: 225°C
  Temp_Zone3: 205°C
  Speed_Belt: 0.45 m/min
  Time_Dwell: T#14m
  Ingredient_A: 8.0 kg (แป้งสาลี)
  Ingredient_B: 2.0 kg (แป้งโฮลวีท)
  Ingredient_C: 0.2 kg (ยีสต์)
```

---

## Step 607: DB สำหรับ Historical Data

### โครงสร้าง Historical Data DB

```
DB30 "Process_History"

Name              | Type                    | Comment
------------------|-------------------------|------------------
Buffer_Size       | INT := 100              | ขนาดบัฟเฟอร์
Write_Index       | INT := 0                | ตำแหน่งเขียน
Read_Index        | INT := 0                | ตำแหน่งอ่าน
Buffer_Full       | BOOL                    | บัฟเฟอร์เต็ม
History           | ARRAY[1..100] OF STRUCT | ข้อมูลประวัติ

// STRUCT สำหรับแต่ละ Record
  Timestamp    : DATE_AND_TIME;  // เวลาบันทึก
  Temperature  : REAL;           // อุณหภูมิ
  Pressure     : REAL;           // ความดัน
  Flow_Rate    : REAL;           // อัตราการไหล
  Batch_No     : DINT;           // หมายเลข Batch
  Operator_ID  : INT;            // รหัสผู้ควบคุม
  Status       : WORD;           // สถานะระบบ
```

### Circular Buffer Ladder Program

```
Network 1: บันทึกข้อมูลทุก 1 นาที (ใช้กับ Time Interrupt OB35)

  |                                                    |
  |--[TON]-------[CMP>]------[MOVE]------------------|
  |  T_Record    Timer.Q     IN: MW200                 |
  |  PT:T#1m                  OUT: History             |
  |                                [Write_Index]       |
  |                                .Temperature        |
  |                                                    |
  |                          [RD_SYS_T]               |
  |                           OUT: History             |
  |                                [Write_Index]       |
  |                                .Timestamp          |
  |                                                    |

Network 2: อัพเดตตัวชี้ Write Index (Circular)

  |                                                    |
  |--[ ]--[<]--------[INC]-----------------------------|
  |  Write "Write_Idx" < 100    "Write_Index"           |
  |        IF Write = 100:      (เพิ่มตัวนับ)           |
  |        [MOVE] 1 → Write_Index (วนกลับต้น)           |
  |                                                    |
```

### Ladder - Circular Buffer Implementation

```
Network 1: เพิ่ม Write Index

  |                                                         |
  |--[P]------[ADD]----------------------------------------|
  |  "Record_  IN1: "Process_History".Write_Index           |
  |   Trigger" IN2: 1                                       |
  |             OUT: "Temp_Index"                           |
  |                                                         |

Network 2: ตรวจสอบ Buffer Wrap-around

  |                                                         |
  |--[>]----------[MOVE]-----------------------------------|
  |  "Temp_Index" > 100  IN: 1                              |
  |                       OUT: "Process_History".Write_Index|
  |                                                         |
  |--[<=]---------[MOVE]-----------------------------------|
  |  "Temp_Index" <= 100 IN: "Temp_Index"                   |
  |                       OUT: "Process_History".Write_Index|
  |                                                         |

Network 3: บันทึกข้อมูล

  |                                                         |
  |--[ ]----------[WRIT_DBL]-------------------------------|
  |  "Record"     SRCBLK: P#M200.0 BYTE 20                  |
  |               DB_NO: 30                                 |
  |               DBADDR: การคำนวณ offset จาก Write_Index  |
  |               LEN: 20                                   |
  |                                                         |
```

---

## Step 608: Indirect DB Access

### การเข้าถึง DB ทางอ้อม

ใช้ได้ใน Siemens S7-300/400 (Classic)

```
// การใช้ pointer เพื่อเข้าถึง DB ต่างๆ
// ตัวอย่าง: เขียนค่าใน DB ที่ระบุด้วย DB Number

// วิธี 1: ใช้ AR1 (Address Register 1)
// ใน STL (Statement List):
L    DB_Number     // โหลด DB Number
T    DBNO          // กำหนด DB Number
OPN  DB            // เปิด DB
L    D [AR1, P#0.0] // อ่านค่า DWORD จาก offset 0
```

### ตัวอย่างการใช้ ANY pointer ใน SCL

```pascal
// การใช้ ANY Pointer ใน Siemens S7-400
// เข้าถึง DB dynamically

FUNCTION "Read_Dynamic_DB" : VOID
VAR_INPUT
  DB_Number : INT;     // หมายเลข DB ที่ต้องการอ่าน
  Offset    : INT;     // ตำแหน่งข้อมูลใน DB
END_VAR
VAR_TEMP
  Source_Ptr : ANY;   // ANY pointer
  Data_Value : REAL;  // ค่าที่อ่านได้
END_VAR

// กำหนด ANY pointer
Source_Ptr := P#DB(DB_Number).DBD(Offset) REAL 1;

// อ่านค่า
BLKMOV(SRCBLK := Source_Ptr,
       DSTBLK := Data_Value);
```

### S7-1200/1500 Indirect Access ด้วย Variant

```pascal
// ใน S7-1200/1500 ใช้ VARIANT type
FUNCTION "Read_Any_DB" : BOOL
VAR_IN_OUT
  Target : VARIANT;  // รับ pointer ไปยัง DB
END_VAR

// ใช้ VariantGet
VariantGet(SRC := Target, DST := "Result_Value");
```

---

## Step 609: Optimized vs Non-Optimized DB Access

### Non-Optimized DB (Standard Layout)

```
┌─────────────────────────────────────────────────────┐
│ Non-Optimized DB - Fixed Memory Layout              │
│                                                     │
│ Offset │ Name        │ Type │ Size                  │
│--------|-------------|------|-------                │
│ 0.0    │ Bool_1      │ BOOL │ 1 bit                 │
│ 0.1    │ Bool_2      │ BOOL │ 1 bit                 │
│ 2.0    │ Int_1       │ INT  │ 2 bytes               │
│ 4.0    │ Real_1      │ REAL │ 4 bytes               │
│ 8.0    │ DInt_1      │ DINT │ 4 bytes               │
│                                                     │
│ สามารถใช้ absolute address: DB1.DBX0.0, DB1.DBW2   │
│ รองรับ cross-platform access                        │
└─────────────────────────────────────────────────────┘
```

### Optimized DB (S7-1200/1500)

```
┌─────────────────────────────────────────────────────┐
│ Optimized DB - Flexible Memory Layout               │
│                                                     │
│ ไม่มี Fixed Offset                                 │
│ ระบบจัดการ Memory Layout อัตโนมัติ                 │
│                                                     │
│ ข้อดี:                                              │
│ + เร็วกว่าในการเข้าถึง                              │
│ + ประหยัดหน่วยความจำ                               │
│ + รองรับ Retain ได้เลือกรายการ                     │
│                                                     │
│ ข้อจำกัด:                                           │
│ - ไม่รองรับ Absolute Access (DB1.DBX0.0)           │
│ - ต้องใช้ Symbolic Access เท่านั้น                  │
│ - BLKMOV อาจมีข้อจำกัด                             │
└─────────────────────────────────────────────────────┘
```

### การเลือกใช้

```
เมื่อใช้ Non-Optimized:
- ต้องการ Absolute Address
- สื่อสารกับอุปกรณ์ภายนอกด้วย Direct DB Access
- โปรแกรมเก่าที่ใช้ pointer/ANY type
- HMI เก่าที่ใช้ DB address แบบ raw

เมื่อใช้ Optimized (แนะนำ):
- โปรแกรมใหม่บน S7-1200/1500
- ต้องการ Performance ดีที่สุด
- ใช้ Symbolic programming เท่านั้น
- ต้องการ Retain เฉพาะบางตัวแปร
```

---

## Step 610: DB ใน Siemens vs Allen-Bradley vs Mitsubishi

### Siemens S7 Data Block

```
// TIA Portal - สร้าง DB
1. Add new block → Data Block
2. กำหนดชื่อและหมายเลข (DB1-DB65535)
3. เลือก Global หรือ Instance
4. เพิ่มตัวแปรใน interface

// การเข้าถึง
"Recipe_DB".Temperature        // Symbolic
DB1.DBD4                       // Absolute (Non-optimized)
```

### Allen-Bradley - UDT and Tags

```
// Studio 5000 - สร้าง UDT
1. Data Types → User-Defined
2. New Data Type: "Recipe_UDT"
3. เพิ่ม Members

// สร้าง Tag ใช้ UDT
Tag Name: Recipe_01
Tag Type: Recipe_UDT

// การเข้าถึง
Recipe_01.Temperature          // ใน Ladder
Recipe_01.Temperature[3]       // ARRAY element
```

### Mitsubishi MELSEC - Structured Labels

```
// GX Works3 - Label Programming
1. Global Label → เพิ่ม Structure
2. กำหนด Label ใช้ Structure type

// ประกาศ Label
Label: Recipe_Data
Type : Recipe_Struct

// การเข้าถึง
Recipe_Data.Temperature        // Symbolic
D100                           // Direct register (ดั้งเดิม)

// ตัวอย่างการใช้ใน Ladder (GX Works3)
[MOV  Recipe_Data.Temperature  D200]
```

### ตารางเปรียบเทียบ

| คุณสมบัติ | Siemens S7 | Allen-Bradley | Mitsubishi |
|-----------|------------|---------------|------------|
| ชื่อ | Data Block (DB) | UDT + Tags | Structured Labels |
| การเข้าถึง | DB1.DBD0 | Tag.Member | D register / Label |
| Array | ARRAY[1..n] OF | Data Type Array | Array Label |
| Nested Struct | รองรับ | รองรับ | รองรับ (GX Works3) |
| Version Control | มี | มี | มีใน iQ-R |
| Online Edit | จำกัด | รองรับ | รองรับ |

---

## Step 611-615: ตัวอย่างระบบจริง - Batch Control System

### โครงสร้าง DB สำหรับ Batch Control

```
// DB_Batch_Control - ระบบควบคุม Batch การผลิต

DB40 "Batch_Control"

Name                  | Type                     | Comment
----------------------|--------------------------|------------------
// สถานะ Batch ปัจจุบัน
Current_Batch_No      | DINT                     | หมายเลข Batch
Batch_Status          | INT                      | 0=Idle,1=Run,2=Hold
Start_Time            | DATE_AND_TIME            | เวลาเริ่ม Batch
End_Time              | DATE_AND_TIME            | เวลาสิ้นสุด
Elapsed_Time          | TIME                     | เวลาที่ผ่านไป
Operator_ID           | INT                      | รหัสผู้ควบคุม

// พารามิเตอร์ Recipe ที่กำลังใช้
Active_Recipe         | INT                      | หมายเลข Recipe
Recipe_Name           | STRING[30]               | ชื่อ Recipe

// ข้อมูลการผลิต
Target_Weight         | REAL                     | น้ำหนักเป้าหมาย
Actual_Weight         | REAL                     | น้ำหนักจริง
Weight_Tolerance      | REAL := 0.5              | ค่าความคลาดเคลื่อน

// ข้อมูลเฉพาะ Step
Current_Step          | INT                      | Step ปัจจุบัน
Step_Status           | INT                      | สถานะ Step
Step_Start_Time       | DATE_AND_TIME            | เวลาเริ่ม Step
Step_Elapsed          | TIME                     | เวลาผ่านไปใน Step
Total_Steps           | INT := 8                 | จำนวน Step ทั้งหมด

// Log ประวัติ
Log_Count             | INT                      | จำนวน Log
Event_Log             | ARRAY[1..50] OF STRUCT   | บันทึกเหตุการณ์
  Log_Time            | DATE_AND_TIME            |
  Log_Type            | INT                      | 1=Info,2=Warn,3=Error
  Log_Message         | STRING[50]               |
  Log_Value           | REAL                     |
```

### Ladder Program - Batch Control FSM

```
Network 1: Batch State Machine

  |  [Batch_Status = 0 (IDLE)]                              |
  |--[ ]--[ ]--[ ]--[MOVE]----------------------------------|
  |  HMI    Recipe  Safety  IN: 1                           |
  |  Start  Loaded  OK       OUT: Batch_Status              |
  |                          (เปลี่ยนเป็น RUNNING)          |
  |                                                         |
  |                         [RD_SYS_T]                      |
  |                          OUT: Batch_Control.Start_Time  |
  |                                                         |
  |                         [INC] Batch_Control.Current_    |
  |                               Batch_No                  |
  |                                                         |

Network 2: Batch Running - ติดตามเวลา

  |                                                         |
  |--[=]----------[TON]------------------------------------|
  |  Batch_Status = 1  T_Batch_Elapsed                     |
  |  (RUNNING)         PT: T#24h (ป้องกัน timeout)         |
  |                                                         |
  |               [MOVE]                                    |
  |                IN: T_Batch_Elapsed.ET                   |
  |                OUT: Batch_Control.Elapsed_Time          |
  |                                                         |

Network 3: ตรวจสอบน้ำหนักเมื่อเสร็จ

  |                                                         |
  |--[=]--[>]------[SET]-----------------------------------|
  |  Step_Done  Actual_Weight >  "Batch_OK"                |
  |  Step_Done  (Target - Tol)                              |
  |             AND                                         |
  |             Actual_Weight <                             |
  |             (Target + Tol)                              |
  |                                                         |
  |--[=]--[NOT]----[SET]-----------------------------------|
  |  Step_Done  Batch_OK      "Batch_Weight_Error"          |
  |                                                         |
```

---

## Step 616-620: Lab Projects

### Lab 1: สร้าง Recipe Management System

**วัตถุประสงค์:** สร้างระบบจัดการสูตรการผลิตด้วย DB

**ขั้นตอน:**

1. สร้าง UDT "Recipe_Type":
```
Temperature_1  : REAL
Temperature_2  : REAL
Belt_Speed     : REAL
Mix_Time       : TIME
Ingredient_A   : REAL
Ingredient_B   : REAL
Active         : BOOL
Recipe_Name    : STRING[20]
```

2. สร้าง Global DB "Recipe_Library":
```
Recipes : ARRAY[1..20] OF Recipe_Type
Active_Recipe : INT
```

3. เขียน Ladder Program สำหรับ:
   - โหลด Recipe จาก DB ไปยัง Process Variables
   - บันทึก Recipe ใหม่ลง DB
   - เปรียบเทียบค่าจริงกับ Recipe

**ผลลัพธ์ที่คาดหวัง:**
```
✓ โหลด Recipe ที่เลือกจาก HMI
✓ ตัวแปร Process ถูกอัพเดตโดยอัตโนมัติ
✓ สามารถบันทึก Recipe ใหม่ได้
```

### Lab 2: Historical Data Logger ด้วย Circular Buffer

**วัตถุประสงค์:** บันทึกข้อมูล Process ด้วย Circular Buffer

**ขั้นตอน:**

1. สร้าง DB "Data_Log" ด้วย ARRAY 500 records
2. เขียน Program บันทึกข้อมูลทุก 10 วินาที
3. Implement Circular Buffer (ทับข้อมูลเก่าเมื่อเต็ม)
4. สร้าง Function อ่านข้อมูลย้อนหลัง N records

### Lab 3: Multi-Product Quality Control DB

**วัตถุประสงค์:** สร้าง DB สำหรับเก็บข้อมูลคุณภาพหลายผลิตภัณฑ์

**โครงสร้าง:**
```
DB "QC_Database"
  Products : ARRAY[1..50] OF STRUCT
    Product_ID    : INT
    Product_Name  : STRING[20]
    Spec_Min      : REAL
    Spec_Max      : REAL
    Samples_Count : INT
    Pass_Count    : INT
    Fail_Count    : INT
    Defect_Rate   : REAL
    Last_Sample   : REAL
    Last_Test_Time: DATE_AND_TIME
  END_STRUCT
```

**โปรแกรมที่ต้องสร้าง:**
1. บันทึกผลการทดสอบใหม่
2. คำนวณ Pass/Fail Rate อัตโนมัติ
3. ส่งสัญญาณเตือนเมื่อ Defect Rate > 5%

---

## สรุป Part 31

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Global vs Instance DB | การเลือกใช้ DB ที่เหมาะสม |
| ARRAY | การจัดกลุ่มข้อมูลประเภทเดียวกัน |
| STRUCT/UDT | การสร้างโครงสร้างข้อมูลซับซ้อน |
| Recipe Storage | การเก็บสูตรการผลิตใน DB |
| Historical Data | Circular Buffer สำหรับ Log |
| Indirect Access | การเข้าถึง DB แบบ dynamic |
| Optimized DB | ความแตกต่างและการเลือกใช้ |
| Cross-Platform | Siemens/AB/Mitsubishi |

---

## การนำทาง

- [← Part 30: Function Blocks](part30_function_blocks.md)
- [→ Part 32: Interrupt Routines](part32_interrupt_routines.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 31 of 40 | Steps 601-620*
