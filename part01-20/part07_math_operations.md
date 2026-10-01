# Part 07: Math Operations
## Steps 111-130: การคำนวณทางคณิตศาสตร์ใน PLC

---

## สารบัญ

- [Step 111: ภาพรวม Math Instructions](#step-111)
- [Step 112: ADD - การบวก](#step-112)
- [Step 113: SUB - การลบ](#step-113)
- [Step 114: MUL - การคูณ](#step-114)
- [Step 115: DIV - การหาร](#step-115)
- [Step 116: MOD - เศษของการหาร](#step-116)
- [Step 117: SQR - รากที่สอง](#step-117)
- [Step 118: ABS - ค่าสัมบูรณ์](#step-118)
- [Step 119: Integer vs Float](#step-119)
- [Step 120: Scaling Analog Signals (4-20mA)](#step-120)
- [Step 121: Engineering Units Conversion](#step-121)
- [Step 122: Flow Calculation](#step-122)
- [Step 123: Temperature Conversion](#step-123)
- [Step 124: Tank Volume Calculation](#step-124)
- [Step 125: PID Setpoint Calculation](#step-125)
- [Step 126: Ratio Control](#step-126)
- [Step 127: Batch Weight Calculation](#step-127)
- [Step 128: Math Error Handling](#step-128)
- [Step 129: Fixed-Point Arithmetic](#step-129)
- [Step 130: Workshop - Complete Process Control](#step-130)

---

## Step 111: ภาพรวม Math Instructions {#step-111}

### ทำไมต้องใช้ Math ใน PLC?

```
โรงงานอุตสาหกรรมใช้คณิตศาสตร์ทุกที่:

┌────────────────────────────────────────────────────────────┐
│  Sensor ให้ค่า Raw Signal (0-4095 counts จาก ADC)          │
│  แต่เราต้องการ Engineering Units (เช่น 0-100°C)            │
│                                                            │
│  Raw: 2048 → คืออะไร? → คำนวณ → 50°C                      │
│                                                            │
│  ใช้: Temperature = (Raw / 4095) × 100                     │
└────────────────────────────────────────────────────────────┘
```

### Math Instruction Overview

```
┌──────────────┬────────────────────────────┬──────────────────────┐
│ Instruction  │ Function                   │ ตัวอย่าง             │
├──────────────┼────────────────────────────┼──────────────────────┤
│ ADD          │ บวก: C = A + B             │ ADD 10 D0 D1         │
│ SUB          │ ลบ: C = A - B              │ SUB D0 5 D1          │
│ MUL          │ คูณ: C = A × B             │ MUL D0 D1 D2         │
│ DIV          │ หาร: C = A ÷ B             │ DIV D0 4095 D1       │
│ MOD          │ เศษหาร: C = A mod B        │ MOD D0 10 D1         │
│ SQR/SQRT     │ รากที่สอง: C = √A          │ SQR D0 D1            │
│ ABS          │ ค่าสัมบูรณ์: C = |A|       │ ABS D0 D1            │
│ NEG          │ เปลี่ยนเครื่องหมาย: C = -A │ NEG D0 D1            │
│ INC          │ เพิ่ม 1: A = A + 1         │ INC D0               │
│ DEC          │ ลด 1: A = A - 1            │ DEC D0               │
└──────────────┴────────────────────────────┴──────────────────────┘
```

### Syntax Comparison - 3 แบรนด์

```
┌──────────────────┬─────────────────────────────────────────────────┐
│ แบรนด์           │ ADD Syntax                                      │
├──────────────────┼─────────────────────────────────────────────────┤
│ Siemens S7       │ ADD_I, ADD_R (Integer/Real)                     │
│                  │ IN1: D0 (MW10), IN2: D1 (MW12), OUT: D2 (MW14) │
│                  │ หรือ: MW14 := MW10 + MW12 (SCL)                 │
├──────────────────┼─────────────────────────────────────────────────┤
│ Allen-Bradley    │ ADD  Source A: N7:0                             │
│ SLC-500          │      Source B: N7:1                             │
│                  │      Dest:     N7:2                             │
├──────────────────┼─────────────────────────────────────────────────┤
│ Mitsubishi FX    │ ADD D0 D1 D2   (D2 = D0 + D1)                  │
│                  │ ADD K10 D0 D1  (D1 = 10 + D0)                  │
│                  │ DADD D0 D2 D4  (Double word 32-bit)             │
└──────────────────┴─────────────────────────────────────────────────┘
```

---

## Step 112: ADD - การบวก {#step-112}

### ตัวอย่าง 1: บวกค่าจาก 2 Sensor

```
สถานการณ์: วัดน้ำหนักรวมจาก 2 Scale
  Scale1 = D100 (kg × 10, ทศนิยม 1 ตำแหน่ง)
  Scale2 = D101
  Total = D102


Network 1: คำนวณน้ำหนักรวม (Mitsubishi FX)

     M8000                              ← M8000 = Always ON
  ───┤ ├────[ADD D100 D101 D102]─       ← D102 = D100 + D101


Siemens TIA Portal (LAD):
  ──── [ADD_I] ────
        IN1: %MW100
        IN2: %MW101
        OUT: %MW102


Allen-Bradley SLC-500:
  ──── [ADD] ────
    Source A: N7:0    (Scale 1)
    Source B: N7:1    (Scale 2)
    Dest:     N7:2    (Total Weight)
```

### ตัวอย่าง 2: ค่าสะสม (Accumulator)

```
สถานการณ์: สะสมน้ำหนักสินค้าที่แพ็คแล้ว

Network 1: เพิ่มน้ำหนักเมื่อแพ็คเสร็จ

     M50.0                               ← Pack Complete Signal
  ───┤ ├────[ADD D200 D100 D200]─        ← D200 = D200 + D100
  Pack_Done                              (D100 = น้ำหนักชิ้นนี้)
                                         (D200 = น้ำหนักสะสม)
```

### Overflow Checking

```
Integer Max Values:
  INT  (16-bit signed):  -32768 ถึง 32767
  DINT (32-bit signed):  -2,147,483,648 ถึง 2,147,483,647

ตรวจสอบ Overflow หลัง ADD:
  Siemens:  ENO output → 1 ถ้า result valid
  Mitsubishi: M8022 = Carry Flag (overflow)
  AB: S:0/1 = Overflow Bit
```

---

## Step 113: SUB - การลบ {#step-113}

### ตัวอย่าง 1: คำนวณ Error

```
สถานการณ์: ควบคุมอุณหภูมิ - คำนวณ Error = Setpoint - Actual

  Setpoint = D300 (°C × 10)
  Actual   = D301
  Error    = D302


Network 1: คำนวณ Error (Mitsubishi)

     M8000
  ───┤ ├────[SUB D300 D301 D302]─     ← D302 = D300 - D301


ถ้า Setpoint = 800 (80.0°C) และ Actual = 750 (75.0°C)
Error = 800 - 750 = 50 (แสดงว่าต่ำกว่า 5.0°C)
```

### ตัวอย่าง 2: คำนวณเวลาที่เหลือ

```
  Total_Time = D400 (วินาที)
  Elapsed_Time = D401
  Remaining = D402


Network 1:
     M8000
  ───┤ ├────[SUB D400 D401 D402]─     ← เวลาที่เหลือ = ทั้งหมด - ที่ผ่านไป
```

### Negative Number Handling

```
ระวัง! ถ้าผลลัพธ์เป็นลบใน Integer:

ตัวอย่าง: D300 = 50 (Setpoint 5°C)
           D301 = 80 (Actual 8°C)
           D302 = 50 - 80 = -30 (ต้องการน้ำเย็นลง 3°C)

การจัดการ:
- ตรวจสอบ Sign Bit: M8020 (Mitsubishi = Zero), S:0/3 (AB = Sign)
- ใช้ ABS ถ้าต้องการค่าสัมบูรณ์
- ใช้ Comparison ก่อนตัดสินใจทิศทาง
```

---

## Step 114: MUL - การคูณ {#step-114}

### ระวัง: ผลลัพธ์การคูณอาจ Overflow!

```
16-bit × 16-bit = 32-bit result

ตัวอย่าง: 1000 × 1000 = 1,000,000 (เกิน INT แต่ไม่เกิน DINT)

Mitsubishi FX:
  MUL D0 D1 D2    → D2 = D0 × D1 (ผลลัพธ์เป็น 32-bit, เก็บใน D2+D3)

ระวัง: D2 เก็บ Lower Word, D3 เก็บ Upper Word ของผลลัพธ์!
```

### ตัวอย่าง 1: Scaling (Linear Interpolation)

```
สถานการณ์: แปลง ADC (0-4095) เป็น Temperature (0-200°C)

สูตร: Temp = (ADC_Value × 200) ÷ 4095


Network 1: คูณ
     M8000
  ───┤ ├────[MUL D0 K200 D10]─      ← D10 = D0 × 200 (ผลใน D10+D11)

Network 2: หาร (ใช้ Double-word register)
     M8000
  ───┤ ├────[DDIV D10 K4095 D20]─   ← D20 = (D10+D11) ÷ 4095
                                      (DDIV = Double word DIV)
```

### ตัวอย่าง 2: คำนวณ Area

```
สถานการณ์: คำนวณพื้นที่ถังสี่เหลี่ยม
  Length = D50 (mm)
  Width  = D51 (mm)
  Area   = D52 (mm²) หรือ D52+D53 ถ้าใหญ่มาก


Network 1:
     M8000
  ───┤ ├────[MUL D50 D51 D52]─      ← D52+D53 = Length × Width
```

### ตัวอย่าง 3: คำนวณ Power (กำลังไฟฟ้า)

```
P = V × I  (Power = Voltage × Current)

  Voltage = D60 (V × 10)
  Current = D61 (A × 100)
  Power   = D62+D63 (W × 1000)


Network 1:
     M8000
  ───┤ ├────[MUL D60 D61 D62]─      ← D62 = V × I

  หลังคำนวณ: หาร 1000 เพื่อให้ได้ Watts:
  ───┤ ├────[DDIV D62 K1000 D64]─   ← D64 = Power ใน Watts
```

---

## Step 115: DIV - การหาร {#step-115}

### ระวัง: Division by Zero!

```
ถ้าหารด้วย 0 → PLC อาจ Fault หรือผลลัพธ์ผิด

ต้องตรวจสอบก่อนเสมอ:

Network 1: ตรวจสอบ Divisor ก่อนหาร

     D1            D1            D2              D3
  ──[CMP K0 D1]──┤>├──[DIV D0 D1 D2]──     ← หารเฉพาะเมื่อ D1 > 0

หรือใช้ Comparison block:
     D1                  D2
  ──[GT K0 D1]──────[DIV D0 D1 D2]──
   ถ้า D1 > 0            หารได้
```

### ตัวอย่าง: คำนวณค่าเฉลี่ย

```
สถานการณ์: หาค่าเฉลี่ยจาก 8 ตัวอย่าง

  Sum = D100 (ผลรวม 8 ตัวอย่าง)
  Count = D101 (= 8)
  Average = D102


Network 1:
     D101               D102
  ──[GT K0 D101]────[DIV D100 D101 D102]──
  ตรวจว่า Count > 0   Average = Sum ÷ Count
```

### ตัวอย่าง: แปลง RPM เป็น m/s

```
Belt Speed (m/s) = (RPM × π × Diameter) ÷ 60

  RPM      = D70
  Diameter = D71 (mm)
  Speed    = D72 (mm/s)

สูตรประมาณ (ไม่ใช้ π จริง): Speed = (RPM × 31416 × Diameter) ÷ (60 × 10000)
หรือ: Speed = (RPM × Diameter × 524) ÷ 100000  (ค่าประมาณ)
```

---

## Step 116: MOD - เศษของการหาร {#step-116}

### MOD ใช้ทำอะไร?

```
MOD = เศษที่เหลือจากการหาร

ตัวอย่าง:
  17 MOD 5 = 2   (17 ÷ 5 = 3 เหลือ 2)
  10 MOD 3 = 1   (10 ÷ 3 = 3 เหลือ 1)
  15 MOD 5 = 0   (15 ÷ 5 = 3 เหลือ 0 → หารลงตัว!)
```

### ตัวอย่าง 1: ตรวจสอบชิ้นงานทุก N ชิ้น

```
สถานการณ์: ตรวจสอบคุณภาพทุก 10 ชิ้น (ชิ้นที่ 10, 20, 30...)

  Part_Count = D200
  N = 10 (ตรวจทุก 10 ชิ้น)
  Remainder = D201


Network 1: คำนวณ MOD
     M8000
  ───┤ ├────[MOD D200 K10 D201]─   ← D201 = D200 mod 10

Network 2: ถ้า Remainder = 0 → ตรวจสอบ
     D201               M60.0
  ──[CMP K0 D201]──┤=├───( )───    ← M60.0 = Quality Check Required
  ถ้า remainder = 0
```

### ตัวอย่าง 2: วนซ้ำ Pattern

```
สถานการณ์: ไฟ LED วิ่ง 8 ดวง วนซ้ำ

  Step = D210  (0-7)

Network 1: เพิ่ม Step ทุก 500ms
     T1 (500ms)
  ───┤ ├────[INC D210]─

Network 2: Reset เมื่อถึง 8 (คือ keep 0-7)
     M8000
  ───┤ ├────[MOD D210 K8 D210]─   ← D210 = D210 mod 8 (0-7 เสมอ)
```

---

## Step 117: SQR - รากที่สอง {#step-117}

### การใช้งาน SQR

```
ใช้ในการคำนวณ:
- พื้นที่วงกลม (จาก diameter)
- ค่า RMS ของสัญญาณ
- คำนวณ Flow จาก Differential Pressure (Orifice Plate)
- Distance ใน 2D Space
```

### ตัวอย่าง: Flow Calculation ด้วย Orifice Plate

```
สูตร: Flow = K × √(ΔP)

  ΔP (Differential Pressure) = D80 (Pascal × 10)
  K = ค่าคงที่ Orifice = 150 (ต้อง calibrate)
  Flow = D81 (L/min × 10)


Network 1: คำนวณรากที่สอง (Mitsubishi FX)
     M8000
  ───┤ ├────[SQR D80 D82]─         ← D82 = √D80

Network 2: คูณด้วย K
     M8000
  ───┤ ├────[MUL D82 K150 D81]─    ← D81 = √D80 × 150
```

### Siemens TIA Portal: SQRT Instruction

```
Network 1:
     %M0.0
  ────┤ ├──── [SQRT] ────
              IN:  %MD100   (REAL input)
              OUT: %MD104   (REAL result)
```

### Allen-Bradley: SQR Instruction

```
     B3:0/0
  ────┤ ├──── [SQR] ────────────────
              Source:  N7:10
              Dest:    N7:11
              (Integer square root, truncated)
```

---

## Step 118: ABS - ค่าสัมบูรณ์ {#step-118}

### เมื่อไหรต้องใช้ ABS?

```
ใช้เมื่อต้องการ "ระยะห่าง" ไม่สนทิศทาง:
- Deviation = |Setpoint - Actual|
- Distance ระหว่าง 2 ตำแหน่ง
- Error magnitude สำหรับ alarm
```

### ตัวอย่าง: Temperature Deviation Alarm

```
  Setpoint = D300 (80°C)
  Actual   = D301 (อ่านจาก Sensor)
  Error    = D302 (อาจเป็นบวกหรือลบ)
  |Error|  = D303


Network 1: คำนวณ Error
     M8000
  ───┤ ├────[SUB D300 D301 D302]─  ← D302 = Setpoint - Actual

Network 2: ค่า Absolute ของ Error
     M8000
  ───┤ ├────[ABS D302 D303]─       ← D303 = |D302|

Network 3: Alarm ถ้า Deviation > 5°C
     D303               M100.0
  ──[CMP K50 D303]──┤<├───( )───   ← Alarm ถ้า |Error| > 5.0°C
                                     (D303 > 50 = เกิน 5.0°C)
```

---

## Step 119: Integer vs Float {#step-119}

### ความแตกต่างของ Data Types

```
┌───────────────┬──────────────────┬────────────────────────────┐
│ Type          │ Range            │ ใช้เมื่อ                   │
├───────────────┼──────────────────┼────────────────────────────┤
│ INT (16-bit)  │ -32768 to 32767  │ Count, Status, Small values│
│ DINT (32-bit) │ ±2.1 billion     │ Large counts, long timers  │
│ REAL (32-bit) │ ±3.4E38          │ Temperature, Flow, PID     │
│               │ Precision: 7 dig │ ต้องการทศนิยม              │
└───────────────┴──────────────────┴────────────────────────────┘
```

### เทคนิค Fixed-Point Arithmetic (ไม่ใช้ Float)

```
วิธีทำทศนิยมด้วย Integer:
  แทน 3.14 ด้วย 314 (คูณ 100 ไว้)
  แทน 98.6°C ด้วย 986 (คูณ 10 ไว้)

ข้อดี: เร็วกว่า, ไม่ต้องการ Math Coprocessor
ข้อเสีย: ต้องระวัง Scale factor เมื่อคูณ/หาร

ตัวอย่าง: คำนวณ 5.5 × 3.2 = 17.6
  แทน: 55 × 32 = 1760 แล้ว ÷ 10 = 176 (= 17.6)
  ดังนั้น: MUL K55 K32 D0  → D0 = 1760
           DIV D0 K10 D1   → D1 = 176 (= 17.6°)
```

### Conversion: INT to REAL (Siemens)

```
Network 1: แปลง INT เป็น REAL สำหรับการคำนวณ

     %M0.0
  ────┤ ├──── [INT_TO_REAL] ────────────
              IN:  %MW100   (INT: 0-4095)
              OUT: %MD200   (REAL)

     %M0.0
  ────┤ ├──── [MUL_R] ────────────────
              IN1: %MD200   (REAL ADC value)
              IN2: 100.0    (Max engineering value)
              OUT: %MD204

     %M0.0
  ────┤ ├──── [DIV_R] ────────────────
              IN1: %MD204
              IN2: 4095.0   (Max ADC count)
              OUT: %MD208   (Temperature in °C, REAL)
```

---

## Step 120: Scaling Analog Signals (4-20mA) {#step-120}

### ทำไม 4-20mA ถึงใช้ 4 ไม่ใช่ 0?

```
4-20mA Standard:
  4 mA  = 0% (ค่าต่ำสุด)
  20 mA = 100% (ค่าสูงสุด)

ข้อดีของ 4mA minimum:
  1. ตรวจสอบ Wire Break ได้ (0mA = สาย断)
  2. ส่งสัญญาณไกลได้ (current loop)
  3. ทนต่อ Noise ดีกว่า Voltage signal
```

### ADC Values สำหรับ 4-20mA

```
12-bit ADC (0-4095):
  4mA  → ADC = 819  (= 4095 × 4/20)
  20mA → ADC = 4095

16-bit ADC (0-27648 Siemens):
  4mA  → 5530  (= 27648 × 4/20)
  20mA → 27648
```

### สูตร Scaling ทั่วไป

```
Engineering Value = (Raw_Input - Raw_Min) × (EU_Max - EU_Min)
                    ─────────────────────────────────────────── + EU_Min
                              (Raw_Max - Raw_Min)

ตัวอย่าง: ADC 819-4095 → Temperature 0-100°C

Temp = (ADC - 819) × (100 - 0)
       ───────────────────────── + 0
              (4095 - 819)

Temp = (ADC - 819) × 100 / 3276
```

### ตัวอย่าง Scaling ใน Mitsubishi FX

```
สถานการณ์:
  Input: D0 = ADC Value (819-4095 จาก 4-20mA)
  Output: D10 = Temperature (0-1000 หน่วยของ 0-100.0°C × 10)


Network 1: ลบ Offset (4mA offset)
     M8000
  ───┤ ├────[SUB D0 K819 D1]─       ← D1 = D0 - 819 (remove 4mA offset)

Network 2: คูณ Scale Factor
     M8000
  ───┤ ├────[MUL D1 K1000 D2]─      ← D2 = D1 × 1000 (scale up for precision)

Network 3: หาร Range
     M8000
  ───┤ ├────[DDIV D2 K3276 D10]─    ← D10 = D2+D3 ÷ 3276
                                      (3276 = 4095 - 819)
```

### ตัวอย่าง Siemens TIA Portal

```
(ใช้ SCALE_X instruction)

Network 1:
     %M0.0
  ────┤ ├──── [SCALE_X] ─────────────────
              VALUE:   %IW64   (Analog Input)
              MIN_IN:  5530    (4mA)
              MAX_IN:  27648   (20mA)
              MIN_OUT: 0.0     (0°C)
              MAX_OUT: 100.0   (100°C)
              OUT:     %MD100  (Temperature in °C, REAL)
```

### ตัวอย่าง Allen-Bradley

```
(ใช้ SCL - Scale with Parameters)

Network 1:
     B3:0/0
  ────┤ ├──── [SCL] ─────────────────
              Source:     N7:0      (ADC 0-4095)
              Rate/10000: 2442      (= 100/4095 × 100000)
              Offset:     0         
              Dest:       N7:10     (0-100%)
```

---

## Step 121: Engineering Units Conversion {#step-121}

### ตารางการแปลงหน่วยที่ใช้บ่อย

```
┌──────────────────┬────────────────────────────────────────────┐
│ การแปลง          │ สูตร                                       │
├──────────────────┼────────────────────────────────────────────┤
│ °C → °F          │ F = (C × 9/5) + 32                         │
│ °F → °C          │ C = (F - 32) × 5/9                         │
│ Bar → PSI        │ PSI = Bar × 14.504                          │
│ PSI → Bar        │ Bar = PSI ÷ 14.504                          │
│ kPa → Bar        │ Bar = kPa ÷ 100                             │
│ m/s → ft/min     │ ft/min = m/s × 196.85                       │
│ L/min → GPM      │ GPM = L/min ÷ 3.785                         │
│ kW → HP          │ HP = kW × 1.341                             │
└──────────────────┴────────────────────────────────────────────┘
```

### ตัวอย่าง: แปลง Bar เป็น PSI

```
PSI = Bar × 14504 ÷ 1000   (ใช้ Fixed Point × 10 สำหรับ Bar)

  Pressure_Bar = D50  (units: 0.1 Bar, ค่า 100 = 10.0 Bar)
  Pressure_PSI = D51  (units: 0.1 PSI)


Network 1: คูณด้วย 14504
     M8000
  ───┤ ├────[MUL D50 K14504 D60]─  ← D60+D61 = D50 × 14504

Network 2: หารด้วย 1000 เพื่อ normalize
     M8000
  ───┤ ├────[DDIV D60 K1000 D51]─  ← D51 = D50 × 14.504
```

---

## Step 122: Flow Calculation {#step-122}

### Flow จาก Differential Pressure (Orifice Plate)

```
สูตรพื้นฐาน: Q = Cd × A × √(2ΔP/ρ)

ในทางปฏิบัติ:
  Q = K × √(ΔP)

โดย K = ค่าคงที่ที่ calibrate จากขนาด orifice


สถานการณ์:
  ΔP (Differential Pressure) = D80 (Pa, 0-5000)
  K  = 2500 (ค่าคงที่ calibrate)
  Q  = D85 (L/min × 10)


Network 1: คำนวณรากที่สองของ ΔP
     M8000
  ───┤ ├────[SQR D80 D82]─          ← D82 = √(ΔP)

Network 2: คูณด้วย K
     M8000
  ───┤ ├────[MUL D82 K2500 D84]─    ← D84+D85 = √ΔP × 2500

Network 3: Scale ลงมา (÷100 เพื่อให้ได้ L/min × 10)
     M8000
  ───┤ ├────[DDIV D84 K100 D85]─    ← D85 = Flow ใน L/min × 10
```

### Flow จาก Electromagnetic Flowmeter (ตรง)

```
Electromagnetic flowmeter ให้ Analog Output โดยตรง:
  4mA = 0 L/min
  20mA = 500 L/min (max flow)

ใช้ Scaling formula จาก Step 120:
  Flow = (ADC - 819) × 500 / 3276
```

### Total Flow (Integration)

```
Total Flow = Σ (Flow Rate × Time Interval)

สำหรับ Scan time = 10ms:
  Total (L) = Total (L) + Flow (L/min) × 10ms / 60000ms

Network 1: คำนวณ Volume ต่อ Scan (10ms scan time)
     M8000
  ───┤ ├────[MUL D85 K10 D90]─      ← D90 = Flow × 10ms (units: L/min × 10 × ms)

Network 2: แปลงเป็น ลิตร (÷ 600000)
     M8000
  ───┤ ├────[DDIV D90 K600000 D92]─ ← D92 = ปริมาณต่อ scan ใน mL

Network 3: สะสม
     M8000
  ───┤ ├────[DADD D92 D94 D94]─     ← D94+D95 = Total Volume สะสม
```

---

## Step 123: Temperature Conversion {#step-123}

### แปลง °C เป็น °F

```
สูตร: F = (C × 9 / 5) + 32

ถ้าใช้ Integer (°C × 10):
  C × 10 = D100  (เช่น 253 = 25.3°C)
  F × 10 = D101

  F × 10 = ((C × 10) × 9 / 5) + 320

Network 1: คูณ C × 9
     M8000
  ───┤ ├────[MUL D100 K9 D102]─    ← D102+D103 = C × 9

Network 2: หาร 5
     M8000
  ───┤ ├────[DDIV D102 K5 D104]─   ← D104 = (C × 9) ÷ 5

Network 3: บวก 320 (= 32°F × 10)
     M8000
  ───┤ ├────[ADD D104 K320 D101]─  ← D101 = F × 10
```

### ตัวอย่าง: Thermocouple Linearization

```
K-Type Thermocouple:
  ADC Value → mV → °C (ต้องใช้ Linearization table)

วิธีง่าย: ใช้ Linear Approximation ช่วง 0-1000°C
  Temp(°C) = ADC × 1000 / 4095  (ประมาณ)

วิธีแม่นยำ: ใช้ Polynomial หรือ Lookup Table
  Temp = A × ADC² + B × ADC + C
  (ค่า A, B, C ต้องหาจาก Calibration)
```

---

## Step 124: Tank Volume Calculation {#step-124}

### ถังทรงกระบอก (Cylindrical Tank)

```
Volume = π × r² × h

โดย:
  r = รัศมีถัง (m)
  h = ความสูงของของเหลว (m)
  V = ปริมาตร (m³ หรือ L)

สำหรับ PLC (Integer, ×10 precision):
  Diameter = D200 (mm × 10, เช่น 2000 = 200.0 mm = 0.2 m)
  Height   = D201 (mm, อ่านจาก Level Sensor)
  Volume   = D202 (Liters × 10)
```

```
ขั้นตอนการคำนวณ:

Step 1: คำนวณ radius² = (D/2)²
  Radius × 10 = D200 ÷ 20 (mm × 10 / 2 / 10 = radius mm)

  Network 1:
       M8000
    ───┤ ├────[DIV D200 K20 D210]─   ← D210 = Radius (mm)

  Network 2:
       M8000
    ───┤ ├────[MUL D210 D210 D212]─  ← D212+D213 = Radius²

Step 2: คูณด้วย π × Height
  Network 3:
       M8000
    ───┤ ├────[DMUL D212 D201 D214]─ ← D214 = Radius² × Height

  Network 4:
       M8000
    ───┤ ├────[DDIV D214 K318000 D202]─ ← D202 = V ÷ (1000/π)
                                           (318000 ≈ 1,000,000 / π)
                                           ผล: Liters × 10

หมายเหตุ: π ≈ 3.14159 → 31416 (×10000)
  1L = 1dm³ = 1,000,000 mm³
  V(L) = π × r(mm)² × h(mm) / 1,000,000
  V(L×10) = 31416 × r² × h / 10,000,000
           = 31416 × r² × h / 10,000,000
```

### ถังทรงเหลี่ยม (Rectangular Tank)

```
Volume = L × W × H

Network 1:
     M8000
  ───┤ ├────[MUL D220 D221 D224]─  ← D224 = Length × Width (mm²)

Network 2:
     M8000
  ───┤ ├────[DMUL D224 D222 D226]─ ← D226 = L × W × H (mm³)

Network 3: แปลงเป็น Liters (÷ 1,000,000)
     M8000
  ───┤ ├────[DDIV D226 K1000000 D228]─ ← D228 = Volume ใน Liters
```

### ถังแนวนอน (Horizontal Cylinder) - สูตรซับซ้อนขึ้น

```
สำหรับถังแนวนอน ใช้ Look-up Table ดีกว่า สูตรตรง
เพราะ Volume ไม่ linear กับความสูง

ตัวอย่าง: สร้าง Lookup Table ใน D300-D399 (100 จุด)
  D300 = Volume ที่ Level 0%  = 0
  D301 = Volume ที่ Level 1%
  D302 = Volume ที่ Level 2%
  ...
  D400 = Volume ที่ Level 100% = Total Volume
```

---

## Step 125: PID Setpoint Calculation {#step-125}

### การคำนวณ Setpoint ที่ Dynamic

```
สถานการณ์: Ramp Setpoint (ค่อยๆ เพิ่มจาก 20°C ไป 80°C ใน 10 นาที)

  Start_SP   = 200 (20.0°C)
  Target_SP  = 800 (80.0°C)
  Ramp_Time  = 600 (วินาที = 10 นาที)
  Rate       = (800-200)/600 = 1 unit/second
  Current_SP = D400


Network 1: คำนวณ Rate per Scan (ถ้า scan = 100ms = 0.1s)
  Rate/scan = (800-200) / (600 × 10) = 0.1 unit/scan
  ≈ 1 unit ทุก 10 scan

Network 2: เพิ่ม Setpoint ทุก 1 วินาที (ใช้ 1sec Timer)
     T5 (1sec)           D400
  ───┤ ├────[ADD D400 K1 D400]─    ← เพิ่ม 0.1°C ทุกวินาที

Network 3: Limit Setpoint (ไม่เกิน Target)
     D400                D400
  ──[CMP D400 K800]──┤>├──[MOV K800 D400]─  ← Cap ที่ 800 (80.0°C)
```

---

## Step 126: Ratio Control {#step-126}

### Ratio Control คืออะไร?

ระบบที่ต้องผสมสารในอัตราส่วนคงที่ เช่น:
- น้ำ:สารเคมี = 10:1
- Component A:B = 3:2

```
สถานการณ์: ผสมสาร A กับ B ในอัตราส่วน A:B = 3:2

  Flow_A = D500 (L/min × 10, จาก Flowmeter)
  Flow_B_Setpoint = D501 (ต้องการ = Flow_A × 2/3)

Network 1: คำนวณ Setpoint ของ B
     M8000
  ───┤ ├────[MUL D500 K2 D510]─    ← D510+D511 = Flow_A × 2
                [DDIV D510 K3 D501]─ ← D501 = Flow_A × 2/3
```

---

## Step 127: Batch Weight Calculation {#step-127}

### ระบบชั่งน้ำหนัก Batch

```
โรงงานผสมอาหาร: ต้องชั่งส่วนผสม 3 อย่าง

  Material A: 150 kg
  Material B: 75 kg
  Material C: 25 kg
  Total: 250 kg


D600 = Total_Recipe_Weight (250)
D601 = Material_A_Target  (150)
D602 = Material_B_Target  (75)
D603 = Material_C_Target  (25)
D610 = Current_Weight (อ่านจาก Load Cell)
D611 = Batch_Phase (0=idle, 1=filling_A, 2=filling_B, 3=filling_C)


Network 1: ตรวจว่าเติม A ได้ถึงเป้าหรือยัง
     D611               D610                    D611
  ──[CMP K1 D611]──┤=├──[CMP D601 D610]──┤>=├──[MOV K2 D611]─
   Phase = Fill_A      Weight >= Target_A      Next Phase = Fill_B

Network 2: คำนวณ Remaining Weight ต้องเติม
     D611                                D612
  ──[CMP K1 D611]──┤=├──[SUB D601 D610 D612]─
   Phase A          Remaining A = Target - Current
```

---

## Step 128: Math Error Handling {#step-128}

### ข้อผิดพลาดที่พบบ่อยใน Math Operations

```
┌────────────────────┬─────────────────────────────────────────┐
│ ข้อผิดพลาด        │ สาเหตุและวิธีแก้                        │
├────────────────────┼─────────────────────────────────────────┤
│ Division by Zero   │ Divisor = 0 → ตรวจสอบก่อนหาร           │
│ Overflow           │ ผลลัพธ์เกิน Data Type range             │
│ Underflow          │ ผลลัพธ์น้อยกว่า min (negative overflow) │
│ Wrong Scale        │ ลืม scale factor → ผิดหน่วย             │
│ Integer Truncation │ DIV ตัดทศนิยมทิ้ง 7÷2=3 ไม่ใช่ 3.5    │
└────────────────────┴─────────────────────────────────────────┘
```

### ป้องกัน Overflow

```
Network 1: ตรวจสอบก่อนคูณ (ป้องกัน INT overflow)

ถ้า D0 × D1 อาจ Overflow ตรวจสอบว่า D0 < 32767/D1

     D0               D1               M80.0
  ──[DIV K32767 D1 D5]──[CMP D0 D5]──┤<=├──(M80.0)─  ← Safe to Multiply

     M80.0
  ───┤ ├────[MUL D0 D1 D2]─            ← คูณเฉพาะเมื่อปลอดภัย
```

### ป้องกัน Divide by Zero

```
Network 1: ตรวจสอบก่อนหาร

     D1                             D2
  ──[CMP K0 D1]──┤<>├──[DIV D0 D1 D2]─  ← หารเฉพาะเมื่อ D1 ≠ 0
  (ถ้า D1 = 0 ข้ามไป)

     D1
  ──[CMP K0 D1]──┤=├──[MOV K9999 D2]─   ← ถ้า D1 = 0 ให้ D2 = error value
```

---

## Step 129: Fixed-Point Arithmetic {#step-129}

### เทคนิค Fixed-Point อย่างละเอียด

```
หลักการ: เก็บค่า N ทศนิยม โดยคูณ 10^N ไว้ก่อน

Scale Factor:
  ×10   = ทศนิยม 1 ตำแหน่ง (เช่น 253 = 25.3°C)
  ×100  = ทศนิยม 2 ตำแหน่ง (เช่น 3141 = 31.41%)
  ×1000 = ทศนิยม 3 ตำแหน่ง

การคำนวณ:
  บวก/ลบ: ทำได้เลย ผลลัพธ์ scale เท่าเดิม
    (25.3 + 10.1) → (253 + 101 = 354 = 35.4°C) ✓

  คูณ: ผลลัพธ์ scale เพิ่ม 2 เท่า ต้องหารคืน
    (2.5 × 4.0) → (25 × 40 = 1000 ÷ 10 = 100 = 10.0) ✓

  หาร: ต้องคูณตัวตั้งก่อนหาร เพื่อรักษา precision
    (25 ÷ 4) → (25 × 10 ÷ 4 = 62 ÷ 10 = 6.2) ✓
```

---

## Step 130: Workshop - Complete Process Control {#step-130}

### โจทย์: ระบบควบคุมอุณหภูมิถังผสม

**ความต้องการ:**
1. อ่านอุณหภูมิจาก Thermocouple (4-20mA → 0-200°C)
2. คำนวณ Error (Setpoint - Actual)
3. คำนวณ % Level ของสาร (4-20mA Level Sensor)
4. คำนวณปริมาตรถัง (D = 1000mm, ความสูงตามระดับ)
5. แสดงค่าทั้งหมดบน HMI

### I/O List

```
┌──────────┬───────────────────────────────────────────────────┐
│ Address  │ Description                                       │
├──────────┼───────────────────────────────────────────────────┤
│ AIW0     │ Temperature sensor (4-20mA → 0-200°C)            │
│ AIW2     │ Level sensor (4-20mA → 0-2000mm)                 │
│ D0       │ ADC Raw Value (Temp)                              │
│ D2       │ ADC Raw Value (Level)                             │
│ D10      │ Temperature (°C × 10, 0-2000)                    │
│ D12      │ Temperature Setpoint (°C × 10)                   │
│ D14      │ Temperature Error (°C × 10)                      │
│ D20      │ Level (mm, 0-2000)                                │
│ D22      │ Level Percentage (% × 10, 0-1000)                │
│ D24      │ Tank Volume (Liters × 10)                         │
│ D26      │ Temperature in °F × 10                           │
└──────────┴───────────────────────────────────────────────────┘
```

### โปรแกรมสมบูรณ์

```
//============================================================
// Tank Temperature & Level Control System
// All values scaled ×10 for 1 decimal place precision
//============================================================

// --- Analog Input Scaling ---

Network 1: อ่านและ Scale อุณหภูมิ (ADC 819-4095 → 0-2000)
// Temp(×10) = (ADC - 819) × 2000 / 3276

     M8000
  ───┤ ├────[SUB D0 K819 D30]─        ← D30 = ADC - 819 (offset)

     M8000
  ───┤ ├────[MUL D30 K2000 D32]─      ← D32 = D30 × 2000

     M8000
  ───┤ ├────[DDIV D32 K3276 D10]─     ← D10 = Temperature ×10


Network 2: Scale Level (ADC 819-4095 → 0-20000mm×10)
// Level(mm) = (ADC - 819) × 2000 / 3276

     M8000
  ───┤ ├────[SUB D2 K819 D34]─

     M8000
  ───┤ ├────[MUL D34 K2000 D36]─

     M8000
  ───┤ ├────[DDIV D36 K3276 D20]─     ← D20 = Level ใน mm


// --- Temperature Control ---

Network 3: คำนวณ Error
     M8000
  ───┤ ├────[SUB D12 D10 D14]─        ← Error = Setpoint - Actual


Network 4: แปลง °C เป็น °F
// F×10 = (C×10 × 9 / 5) + 320

     M8000
  ───┤ ├────[MUL D10 K9 D40]─         ← D40 = C×10 × 9

     M8000
  ───┤ ├────[DDIV D40 K5 D42]─        ← D42 = (C×10 × 9) / 5

     M8000
  ───┤ ├────[ADD D42 K320 D26]─       ← D26 = F×10


// --- Level Calculation ---

Network 5: คำนวณ Level Percentage
// % × 10 = Level(mm) × 1000 / 2000

     M8000
  ───┤ ├────[MUL D20 K1000 D44]─      ← D44 = Level × 1000

     M8000
  ───┤ ├────[DDIV D44 K2000 D22]─     ← D22 = Level %×10


Network 6: คำนวณ Volume (ถัง D=1000mm = r=500mm)
// V(L×10) = π × r² × h / 1,000,000 × 10
//         = 31416 × 500² × h / 10,000,000,000
//         = 31416 × 250000 × h / 10,000,000,000
//         = 7854 × h / 100000

     M8000
  ───┤ ├────[MUL D20 K7854 D46]─      ← D46+D47 = Level × 7854

     M8000
  ───┤ ├────[DDIV D46 K100000 D24]─   ← D24 = Volume ×10 (Liters)


// --- Alarms ---

Network 7: High Temperature Alarm (> 180°C)
     D10               M110.0
  ──[CMP K1800 D10]──┤<├──( S )─    ← Alarm ถ้า Temp > 180.0°C
  (D10 > 1800 = Temp > 180.0°C)

Network 8: Low Level Alarm (< 10%)
     D22               M110.1
  ──[CMP D22 K100]──┤<├──( S )─     ← Alarm ถ้า Level < 10.0%


// --- HMI Data Display ---
// D10 = Temp ×10 (แสดงบน HMI เป็น XX.X °C)
// D22 = Level ×10 (แสดงบน HMI เป็น XX.X %)
// D24 = Volume ×10 (แสดงบน HMI เป็น XXXX.X L)
// D26 = Temp °F ×10
```

### Verification Table

```
ตรวจสอบความถูกต้องของ Scaling:

┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ ADC (D0) │ D30      │ D32      │ D10      │ Meaning  │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│ 819      │ 0        │ 0        │ 0        │ 0.0°C    │
│ 1228     │ 409      │ 818,000  │ 249 ≈250 │ ~25.0°C  │
│ 2457     │ 1638     │ 3,276,000│ 1000     │ 100.0°C  │
│ 4095     │ 3276     │ 6,552,000│ 2000     │ 200.0°C  │
└──────────┴──────────┴──────────┴──────────┴──────────┘
```

### Lab Projects

**Lab 1:** ปรับโปรแกรมให้ใช้ Floating Point (REAL) แทน Fixed-Point แล้วเปรียบเทียบผลลัพธ์

**Lab 2:** เพิ่มการคำนวณ Average Temperature จาก 10 sample ล่าสุด (ใช้ Array และ FIFO)

**Lab 3:** เพิ่มฟีเจอร์ Trend Logging - บันทึกอุณหภูมิทุก 1 นาที เก็บ 60 ค่า (1 ชั่วโมง)

**Lab 4:** เพิ่มการแปลง Pressure จาก Bar เป็น PSI และ kPa

**Lab 5:** สร้างระบบ Calibration - ใส่ค่า Offset และ Gain เพื่อ Fine-tune ค่าที่อ่านได้

---

## สรุป Part 07

```
┌──────────────────────────────────────────────────────────────┐
│ สิ่งที่เรียนรู้ใน Part 07:                                    │
│                                                              │
│ ✓ Math Instructions: ADD, SUB, MUL, DIV, MOD, SQR, ABS     │
│ ✓ Integer vs Float และเมื่อไหรใช้อะไร                        │
│ ✓ Fixed-Point Arithmetic - ทศนิยมโดยไม่ใช้ Float             │
│ ✓ Analog Signal Scaling (4-20mA → Engineering Units)         │
│ ✓ Flow Calculation (Orifice Plate)                           │
│ ✓ Temperature Conversion (°C ↔ °F)                           │
│ ✓ Tank Volume Calculation (Cylindrical & Rectangular)        │
│ ✓ Error Handling (Divide by Zero, Overflow)                  │
│ ✓ Complete Process Control Program                           │
└──────────────────────────────────────────────────────────────┘
```

---

**[← Part 06: Memory Bits](part06_memory_bits.md)** | **[Part 08: Comparison Instructions →](part08_comparison.md)**
