# Part 22: Analog I/O และ Scaling (Steps 411-430)

## ภาพรวม
ในระบบอุตสาหกรรมจริง ไม่ใช่ทุกอย่างที่เป็น ON/OFF ค่าอุณหภูมิ ความดัน ระดับ ความเร็ว ล้วนต้องการ **Analog Signal** เพื่อวัดค่าที่เป็นเลขต่อเนื่อง บทนี้จะสอนการทำงานกับ Analog I/O ตั้งแต่พื้นฐานสัญญาณจนถึงการแปลงค่าให้ใช้งานได้จริง

---

## Step 411: ประเภทของ Analog Signal

```
┌──────────────────────────────────────────────────────────┐
│              Analog Signal Types in Industry             │
│                                                          │
│  CURRENT SIGNAL:                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │  4-20mA  (Industrial Standard - Most Common)       │ │
│  │  0-20mA  (Less Common)                            │ │
│  │  ข้อดี: ไม่ได้รับผลจาก Noise, สาย Drop           │ │
│  │  ข้อดี: ตรวจสอบ Wire Break ได้ (< 4mA = Fault)   │ │
│  │  ระยะทาง: สูงสุด 1,000 เมตร                       │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  VOLTAGE SIGNAL:                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │  0-10V   (Common for short distance)               │ │
│  │  0-5V    (Older systems)                           │ │
│  │  ±10V    (Bidirectional, e.g. servo)               │ │
│  │  1-5V    (4-20mA equivalent in some systems)       │ │
│  │  ข้อเสีย: ได้รับผลจาก Impedance ระยะทาง            │ │
│  │  ระยะทาง: แนะนำ < 50 เมตร                          │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  THERMOCOUPLE SIGNAL:                                    │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Type J: Fe-Constantan, 0-750°C, 0-42.28mV        │ │
│  │  Type K: NiCr-Ni, -200 ถึง 1260°C (ที่นิยมสุด)   │ │
│  │  Type T: Cu-Constantan, -200 ถึง 350°C            │ │
│  │  Type E: NiCr-Constantan, -270 ถึง 1000°C        │ │
│  │  ต้องการ Cold Junction Compensation                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  RTD (Resistance Temperature Detector):                  │
│  ┌────────────────────────────────────────────────────┐ │
│  │  PT100: 100Ω ที่ 0°C (IEC 60751)                 │ │
│  │  PT1000: 1000Ω ที่ 0°C                            │ │
│  │  ความแม่นยำสูงกว่า Thermocouple                    │ │
│  │  ช่วง: -200 ถึง 850°C                              │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## Step 412: ADC Resolution และความละเอียด

```
┌──────────────────────────────────────────────────────────┐
│              ADC Resolution Examples                      │
│                                                          │
│  12-bit ADC:                                             │
│  - 2^12 = 4,096 count (0 to 4095)                       │
│  - ตัวอย่าง: 4-20mA → 0 ถึง 4095 counts                │
│  - Resolution: 16mA / 4096 = 0.00390625 mA/count       │
│  - สำหรับ 0-100°C: 100/4096 ≈ 0.0244°C/count           │
│                                                          │
│  13-bit ADC (Siemens S7):                                │
│  - -4096 to +4095 = 8192 count (สำหรับ Bipolar)         │
│  - 0 to 27,648 count (สำหรับ 4-20mA)                   │
│                                                          │
│  16-bit ADC:                                             │
│  - 2^16 = 65,536 count                                  │
│  - ละเอียดมากสำหรับ Precision Application               │
│                                                          │
│  ตัวอย่าง Siemens S7 Standard:                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  4mA = 0 counts                                  │   │
│  │  20mA = 27,648 counts                            │   │
│  │  Over Range = 27,648 ถึง 32,511                  │   │
│  │  Under Range = -4,864 ถึง -1                     │   │
│  │  Wire Break = -32,768                            │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Step 413: Scaling Formula พื้นฐาน

### สูตร Linear Scaling:

```
                     (Raw - Raw_Min)
Eng_Value = Eng_Min + ─────────────────── × (Eng_Max - Eng_Min)
                    (Raw_Max - Raw_Min)
```

**ตัวอย่าง:** แปลง 4-20mA (0-27648 counts) เป็น 0-100°C

```
Eng = 0 + ((Raw - 0) / (27648 - 0)) × (100 - 0)
Eng = Raw × 100 / 27648
Eng = Raw × 0.003617...
```

### ตัวอย่าง Ladder - Siemens S7-1200:

```
// Analog Input: IW64 (Slot 2, Channel 0)
// Raw: 0 to 27648 (4-20mA)
// Engineering: 0.0 to 100.0 °C

NETWORK 1: // อ่าน Raw Value
    ┌────────────────────────────────────(MOVE)──┐
    │                                   IN: IW64  │
    │                               OUT: Raw_Value│
    └────────────────────────────────────────────┘

NETWORK 2: // Scale to Engineering Value
    ┌────────────────────────────────────(NORM_X)─┐
    │                              MIN: 0          │
    │                           VALUE: Raw_Value   │
    │                              MAX: 27648       │
    │                             OUT: Normalized   │
    └────────────────────────────────────────────┘

NETWORK 3: // Map to Engineering Range
    ┌────────────────────────────────────(SCALE_X)─┐
    │                              MIN: 0.0         │
    │                           VALUE: Normalized    │
    │                              MAX: 100.0        │
    │                             OUT: Temperature   │
    └─────────────────────────────────────────────┘
```

### ตัวอย่าง Ladder - Allen-Bradley:

```
// ControlLogix - Analog Scaling
// AI Tag: Temperature_Raw (INT) - Raw ADC value
// Tag: Temperature_Eng (REAL) - Engineering units

RUNG 1: // Scale using SCL instruction
    ┌───────────────────────────────────────────────────┐
    │     SCL - Scale                                    │
    │     Source: Temperature_Raw                        │
    │     Rate[/10000]: 3617    (= 100/27648 × 10000)  │
    │     Offset[/10000]: 0                              │
    │     Dest: Temperature_Eng                          │
    └───────────────────────────────────────────────────┘
```

### ตัวอย่าง - Mitsubishi FX5U:

```
// Mitsubishi - ใช้ FUN Instruction
// D0 = Raw Input (0-4000 for built-in AD)
// D10 = Engineering Value (0.0 to 100.0 ×10 = 0-1000)

// Method 1: ใช้ Arithmetic
LD      M8000           // Always ON
MUL     D0  K1000  D10  // D10 = D0 × 1000
DIV     D10 K4000  D10  // D10 = D10 / 4000 = 0-1000 (= 0.0-100.0°C ×10)

// Method 2: ใช้ SCAL Instruction (FX5U)
LD      M8000
SCAL    D0  K0  K4000  K0  K1000  D10
//      Source  In_Min  In_Max  Out_Min  Out_Max  Dest
```

---

## Step 414: Dead Band (Hysteresis)

Dead Band ใช้ป้องกันการ Chatter เมื่อ Signal อยู่ใกล้ Setpoint:

```
┌──────────────────────────────────────────────────────────┐
│                    Dead Band Concept                     │
│                                                          │
│  Temperature Setpoint = 80°C                            │
│  Dead Band = ±2°C                                       │
│                                                          │
│    Heater ON ─────────────────────────────              │
│                78°C│                    │               │
│                    │                  82°C│              │
│    Heater OFF ─────│─────────────────────│──            │
│                    └────── Dead Band ────┘               │
│                                                          │
│  ถ้าไม่มี Dead Band:                                    │
│  Heater จะ ON/OFF รัวๆ ที่ 80°C                         │
│  ทำให้ Contact เสีย และ Process ไม่เสถียร                │
└──────────────────────────────────────────────────────────┘
```

### Ladder Implementation:

```
// Siemens S7 - Dead Band Control
// Temperature_PV = Process Variable
// Setpoint = 80.0
// Dead_Band = 2.0

NETWORK 1: // Turn ON Heater เมื่ออุณหภูมิต่ำกว่า (Setpoint - Dead_Band)
    ┌──[CMP <]──────────────────────────────────────────┐
    │  Temperature_PV < (Setpoint - Dead_Band)           │
    │  Temperature_PV < 78.0                             │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(S)────────┐
                                           Heater_ON     │
                                                        └┘

NETWORK 2: // Turn OFF Heater เมื่ออุณหภูมิสูงกว่า (Setpoint + Dead_Band)
    ┌──[CMP >]──────────────────────────────────────────┐
    │  Temperature_PV > (Setpoint + Dead_Band)           │
    │  Temperature_PV > 82.0                             │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(R)────────┐
                                           Heater_ON     │
                                                        └┘
```

---

## Step 415: Noise Filtering - Moving Average

Signal Noise เป็นปัญหาทั่วไป วิธี Averaging ช่วยได้:

```
┌──────────────────────────────────────────────────────────┐
│              Moving Average Filter                       │
│                                                          │
│  เก็บ 8 ค่าล่าสุด แล้วหาค่าเฉลี่ย                      │
│                                                          │
│  Sample Buffer: [val1, val2, val3, val4,                 │
│                  val5, val6, val7, val8]                 │
│                                                          │
│  Average = (val1+val2+...+val8) / 8                     │
│                                                          │
│  ทุก Scan ใหม่:                                          │
│  1. Shift values left                                    │
│  2. Add new value at end                                 │
│  3. Calculate average                                    │
│                                                          │
│  ผลลัพธ์: ลดผลกระทบของ Noise Spike                      │
│  Trade-off: เพิ่ม Response Time                          │
└──────────────────────────────────────────────────────────┘
```

### Ladder Implementation (Siemens S7):

```
// DB20: Filter Data Block
// DB20.DBW0 = Buf[0]  (Newest)
// DB20.DBW2 = Buf[1]
// DB20.DBW4 = Buf[2]
// ...
// DB20.DBW14 = Buf[7]  (Oldest)
// DB20.DBW16 = Sum
// DB20.DBW18 = Average

NETWORK 1: // Shift Buffer (ทุก Scan Cycle)
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │                                                    │
    │  MOVE DB20.DBW12 → DB20.DBW14  (Buf[6]→Buf[7])  │
    │  MOVE DB20.DBW10 → DB20.DBW12  (Buf[5]→Buf[6])  │
    │  MOVE DB20.DBW8  → DB20.DBW10  (Buf[4]→Buf[5])  │
    │  MOVE DB20.DBW6  → DB20.DBW8   (Buf[3]→Buf[4])  │
    │  MOVE DB20.DBW4  → DB20.DBW6   (Buf[2]→Buf[3])  │
    │  MOVE DB20.DBW2  → DB20.DBW4   (Buf[1]→Buf[2])  │
    │  MOVE DB20.DBW0  → DB20.DBW2   (Buf[0]→Buf[1])  │
    │  MOVE Raw_Input  → DB20.DBW0   (New→Buf[0])      │
    └──────────────────────────────────────────────────┘

NETWORK 2: // Calculate Sum and Average
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │                                                    │
    │  ADD (Buf[0] + Buf[1] + Buf[2] + Buf[3])          │
    │  ADD (Buf[4] + Buf[5] + Buf[6] + Buf[7])          │
    │  → Sum                                             │
    │                                                    │
    │  DIV Sum / 8 → Average                            │
    └──────────────────────────────────────────────────┘
```

### ตัวอย่าง Mitsubishi ด้วย Loop:

```
// Mitsubishi - Moving Average ด้วย FOR Loop
// D200-D207 = Buffer Array (8 values)
// D210 = New Sample
// D220 = Average Output

// Shift Array
LD      M8000
FOR     K7             // 7 ครั้ง (index 7 ลงถึง 1)
MOV     Z0    D0       // Z0 = Loop Counter
INC     Z0             // Z0+1
MOV     D200Z0 D201Z0  // Move D201 → D202, etc.
NEXT

// Insert New Value
MOV     D210  D200    // New value → D200[0]

// Calculate Sum
LD      M8000
MOV     K0    D215    // Sum = 0
FOR     K8
ADD     D200Z0 D215 D215
NEXT

// Average
DIV     D215  K8   D220  // Average = Sum / 8
```

---

## Step 416: Signal Conditioning

```
┌──────────────────────────────────────────────────────────┐
│          Signal Conditioning Overview                    │
│                                                          │
│  PROBLEM: Raw Signal ไม่พร้อมสำหรับ PLC                 │
│  SOLUTION: Signal Conditioning Module                    │
│                                                          │
│  Signal Conditioner ทำหน้าที่:                          │
│                                                          │
│  1. AMPLIFICATION                                       │
│     ขยาย Signal เล็กๆ จาก Thermocouple (mV) ให้ใหญ่    │
│                                                          │
│  2. ISOLATION                                           │
│     แยก Ground Loop เพื่อป้องกัน Noise                  │
│     ป้องกัน Voltage Spike เสียหาย PLC                   │
│                                                          │
│  3. CONVERSION                                          │
│     แปลงจาก TC/RTD เป็น 4-20mA ก่อนเข้า AI Module     │
│                                                          │
│  4. FILTERING                                           │
│     Low-pass Filter ตัด High Frequency Noise           │
│                                                          │
│  5. LINEARIZATION                                       │
│     TC มี Non-linear Response → แปลงให้เป็น Linear     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 417: Analog Output Scaling

```
// Analog Output: แปลงค่า Engineering → Raw Value สำหรับ Actuator

// ตัวอย่าง: Control Valve 0-100% → 4-20mA

//          (Eng - Eng_Min)
// Raw = Raw_Min + ─────────────────── × (Raw_Max - Raw_Min)
//          (Eng_Max - Eng_Min)

// Raw = 0 + (Valve_Pos - 0) / (100 - 0) × (27648 - 0)
// Raw = Valve_Pos × 27648 / 100
// Raw = Valve_Pos × 276.48

// Siemens S7 - Analog Output
// Valve_Position_Percent = Engineering Value (0.0 to 100.0)
// QW80 = Analog Output Module (4-20mA)

NETWORK 1: // Scale Output
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │                                                    │
    │  NORM_X:                                          │
    │    MIN: 0.0                                       │
    │    VALUE: Valve_Position_Percent                  │
    │    MAX: 100.0                                     │
    │    OUT: Normalized_Value                          │
    │                                                    │
    │  SCALE_X:                                         │
    │    MIN: 0                                         │
    │    VALUE: Normalized_Value                        │
    │    MAX: 27648                                     │
    │    OUT: QW80                                      │
    └──────────────────────────────────────────────────┘
```

---

## Step 418: Flow Control Valve Example

```
┌──────────────────────────────────────────────────────────┐
│           Flow Control System                            │
│                                                          │
│    ┌──────────┐   4-20mA    ┌──────────┐               │
│    │ Flow     │ ──────────► │  PLC     │               │
│    │ Meter    │             │  AI 0    │               │
│    └──────────┘             │          │               │
│                             │          │               │
│    ┌──────────┐   4-20mA   │          │               │
│    │ Control  │ ◄────────── │  AO 0    │               │
│    │ Valve    │             └──────────┘               │
│    └──────────┘                                        │
│                                                          │
│  Flow Meter: 0-100 L/min → 4-20mA                      │
│  Control Valve: 0-100% Open ← 4-20mA                   │
│  Setpoint: 50 L/min                                     │
│                                                          │
│  Simple On/Off Control (ก่อนเรียน PID):                │
│  If Flow < Setpoint - 2: Valve Open More               │
│  If Flow > Setpoint + 2: Valve Close More              │
│  Rate: ±0.5% per Scan                                   │
└──────────────────────────────────────────────────────────┘
```

### โค้ด Allen-Bradley:

```
// ControlLogix - Flow Control
// FlowMeter_Raw (INT) - AI input 0-32767
// Flow_LPM (REAL) - Flow in L/min
// Valve_Pos (REAL) - Valve position 0.0-100.0%
// Flow_Setpoint (REAL) = 50.0
// Valve_AO (INT) - AO output

RUNG 1: // Scale Flow Meter Input
    ┌────────────────────────────────────────────────────┐
    │  CPT - Compute                                     │
    │  Dest: Flow_LPM                                    │
    │  Expression: FlowMeter_Raw * 100.0 / 32767.0      │
    └────────────────────────────────────────────────────┘

RUNG 2: // Open Valve More ถ้า Flow ต่ำเกิน
    ┌──[LES Flow_LPM, (Flow_Setpoint - 2.0)]────────────┐
    │  Flow < Setpoint - Deadband                        │
    └──┬─────────────────────────────────────────────────┘
       └──[CPT Valve_Pos = Valve_Pos + 0.5]─────────────┤
          (เพิ่ม 0.5% ทุก Scan ไม่เกิน 100%)

RUNG 3: // Close Valve More ถ้า Flow สูงเกิน
    ┌──[GRT Flow_LPM, (Flow_Setpoint + 2.0)]────────────┐
    │  Flow > Setpoint + Deadband                        │
    └──┬─────────────────────────────────────────────────┘
       └──[CPT Valve_Pos = Valve_Pos - 0.5]─────────────┤

RUNG 4: // Clamp Valve Position 0-100%
    ┌──[LES Valve_Pos, 0.0]──────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──[MOV 0.0 → Valve_Pos]

    ┌──[GRT Valve_Pos, 100.0]────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──[MOV 100.0 → Valve_Pos]

RUNG 5: // Scale Valve Output
    ┌────────────────────────────────────────────────────┐
    │  CPT - Compute                                     │
    │  Dest: Valve_AO                                    │
    │  Expression: TRUNC(Valve_Pos * 32767.0 / 100.0)  │
    └────────────────────────────────────────────────────┘
```

---

## Step 419: Thermocouple Input Module

```
┌──────────────────────────────────────────────────────────┐
│          Thermocouple (TC) Fundamentals                  │
│                                                          │
│  หลักการ Seebeck Effect:                                │
│  เมื่อโลหะ 2 ชนิดต่อกันที่ปลายสองด้าน แล้วมีอุณหภูมิต่างกัน │
│  จะเกิดแรงดันไฟฟ้าตามสัดส่วนกับ ΔTemp                   │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Type K Thermocouple Characteristics            │   │
│  │  Materials: NiCr (+) and Ni (-)                 │   │
│  │  Range: -200°C to 1260°C                        │   │
│  │  Sensitivity: ~41 μV/°C                         │   │
│  │  Color Code: Yellow(+) Red(-)  [IEC]            │   │
│  │  Most common for general industrial use         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  Cold Junction Compensation (CJC):                      │
│  - TC วัดอุณหภูมิ "ความต่าง" ระหว่าง Hot และ Cold Junction │
│  - Cold Junction = จุดต่อที่ Module                     │
│  - Module ต้องมี CJC Sensor วัดอุณหภูมิที่ Module ด้วย │
│  - TC Value = Hot Temp - Module Temp                    │
│  - จริงๆ: Hot Temp = TC Value + Module Temp             │
└──────────────────────────────────────────────────────────┘
```

### การตั้งค่า Siemens TC Module:

```
// Siemens SM 1231 AI TC Module
// Hardware Config:
//   Channel 0: Type K, 0.1°C resolution
//   Diagnostics: Wire break detection ON
//   Temperature unit: Celsius

// IW96 = Channel 0 Raw Value
// ค่า: -2000 = -200.0°C
//      0     =   0.0°C
//      5000  = 500.0°C
//      12600 = 1260.0°C

NETWORK 1: // อ่านค่า TC แล้วหาร 10 เพื่อได้ °C
    ┌────────────────────────────────────────────────────┐
    │  DIV: IW96 / 10.0 → Temperature_Actual            │
    └────────────────────────────────────────────────────┘

NETWORK 2: // ตรวจ Wire Break (< -32768 หรือ ค่าผิดปกติ)
    ┌──[CMP < IW96, -4864]──────────────────────────────┐
    │  Under Range = Wire Break                         │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────( )────────┐
                                         TC_Fault        │
                                                        └┘
```

---

## Step 420: RTD Input Module

```
// Siemens SM 1231 AI RTD Module
// Hardware Config:
//   Channel 0: PT100, 4-wire, 0.1°C resolution
//   Diagnostics: Wire break ON, Short circuit ON

// PT100 Raw Value:
// 0°C = 0 raw
// 100°C = 1000 raw (= 100.0°C × 10)
// Scale Factor = 0.1 (raw × 0.1 = Temperature in °C)

NETWORK 1: // Scale PT100 Reading
    ┌────────────────────────────────────────────────────┐
    │  INT_TO_REAL: IW100 → Temp_Raw_Real               │
    │  DIV: Temp_Raw_Real / 10.0 → Temperature_PT100    │
    └────────────────────────────────────────────────────┘
```

---

## Step 421: ระบบวัดอุณหภูมิสมบูรณ์

### ตัวอย่าง: ระบบควบคุมอุณหภูมิเตาอบ (Oven Temperature System)

```
┌──────────────────────────────────────────────────────────┐
│           Complete Oven Temperature System               │
│                                                          │
│  ┌─────────┐  TC Type K  ┌────────────┐                │
│  │  Oven   │ ──────────► │ TC Module  │                │
│  │  800°C  │             │ SM1231 TC  │                │
│  └─────────┘             └─────┬──────┘                │
│      ▲                         │ IW96 (Raw)             │
│      │                    ┌────▼──────┐                │
│      │ 4-20mA             │    PLC    │                │
│      │                    │  S7-1200  │                │
│  ┌───┴────┐               └─────┬──────┘                │
│  │SCR     │      4-20mA         │ QW80 (Output)         │
│  │Heater  │ ◄────────────────── │                       │
│  │Control │                     │                       │
│  └────────┘               ┌─────▼──────┐                │
│                            │    HMI     │                │
│                            │ KTP700     │                │
│                            └────────────┘                │
└──────────────────────────────────────────────────────────┘
```

### โปรแกรม PLC สมบูรณ์:

```
// OB1: Main Program
// FB1: Temperature_Controller (เรียกใช้ทุก 100ms)

// FB1 Interface:
// IN: Setpoint (REAL) - °C
// IN: Deadband (REAL) - °C default 2.0
// IN: Max_Temp (REAL) - Overtemp protection
// OUT: Heater_Command (REAL) - 0-100%
// STAT: Current_Temp (REAL)
// STAT: Fault (BOOL)

// FB1 Implementation:

NETWORK 1: // อ่านค่าอุณหภูมิ
    ┌────────────────────────────────────────────────────┐
    │  DIV: IW96 / 10.0 → #Current_Temp                 │
    └────────────────────────────────────────────────────┘

NETWORK 2: // ตรวจสอบ Overtemperature
    ┌──[CMP > #Current_Temp, #Max_Temp]─────────────────┐
    │                                                    │
    └──┬─────────────────────────────────────────────────┘
       ├──────────────────────────────────────(S)────────┐
       │                                    #Fault       │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────(MOVE 0.0)─┐
                                          → #Heater_Command│
                                                         └┘

NETWORK 3: // On/Off Control with Deadband (ถ้าไม่ Fault)
    ┌──[#Fault]/──────────────────────────────────────────┐
    │  Not Faulted                                        │
    └──┬──────────────────────────────────────────────────┘
       │
       ├──[CMP < #Current_Temp, (#Setpoint-#Deadband)]──┐
       │  Too Cold                                       │
       └──┬──────────────────────────────────────────────┘
          └────────────────────────────(MOVE 100.0)──────┐
                                    → #Heater_Command    │
                                                        └┘
       │
       ├──[CMP > #Current_Temp, (#Setpoint+#Deadband)]──┐
       │  Too Hot                                        │
       └──┬──────────────────────────────────────────────┘
          └────────────────────────────(MOVE 0.0)────────┐
                                    → #Heater_Command    │
                                                        └┘

NETWORK 4: // Scale Output
    ┌────────────────────────────────────────────────────┐
    │  NORM_X: 0.0, #Heater_Command, 100.0 → Normalized │
    │  SCALE_X: 0, Normalized, 27648 → QW80             │
    └────────────────────────────────────────────────────┘

NETWORK 5: // Reset Fault เมื่ออุณหภูมิลดลง + Reset Button กด
    ┌──[Reset_Button]─[CMP < #Current_Temp, #Max_Temp]──┐
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────(R)────────┐
                                             #Fault      │
                                                        └┘
```

---

## Step 422: Analog Input Wiring

```
┌──────────────────────────────────────────────────────────┐
│           2-Wire vs 4-Wire Transmitter                   │
│                                                          │
│  2-Wire Transmitter (Loop Powered):                     │
│                                                          │
│    +24V ──────┬──────────────┐                          │
│               │              │                          │
│            ┌──┴──┐        ┌──┴──┐                       │
│            │ PLC │        │Xmtr │ (Transmitter)         │
│            │ AI  │        │     │ วัดค่า Process         │
│            └──┬──┘        └──┬──┘                       │
│               │              │                          │
│            0V ──────────────┘                           │
│                                                          │
│  กระแส 4-20mA ไหลใน Loop                                │
│  Transmitter ควบคุมกระแส ตามค่าที่วัดได้                │
│                                                          │
│  ───────────────────────────────────────────────────── │
│                                                          │
│  4-Wire Transmitter (Self Powered):                     │
│                                                          │
│    +24V ──────────────────── Transmitter (+)            │
│    0V ──────────────────────  Transmitter (-)           │
│    AI(+) ───────────────────  Signal Out (+)            │
│    AI(-) ───────────────────  Signal Out (-)            │
│                                                          │
│  Power และ Signal แยกกัน                                │
│  ใช้กับ Transmitter ที่ต้องการพลังงานมาก               │
└──────────────────────────────────────────────────────────┘
```

---

## Step 423: Input Range Configuration

```
// Siemens S7-1200 AI Module - Configuration Options
// Hardware: SM 1231 AI 8 x 13bit

// Channel Types Available:
// ┌─────────────────────────────────────────────────┐
// │  Voltage:  ±10V, ±5V, ±2.5V, 0-10V            │
// │  Current:  4-20mA, 0-20mA                       │
// │  Resistance: 0-150Ω, 0-300Ω, 0-600Ω           │
// │  Thermocouple: J, K, E, R, S, B, N, C         │
// │  RTD: Pt10, Pt50, Pt100, Pt200, Pt500, Pt1000  │
// │       Ni100, Ni120, Ni200, Ni500, Ni1000        │
// └─────────────────────────────────────────────────┘

// Raw Value Ranges (Siemens Standard):
// 4-20mA:  0 to 27648
// 0-20mA:  0 to 27648
// 0-10V:   0 to 27648
// ±10V: -27648 to +27648
// TC K: -2731 = -273.1°C to 13500 = 1350.0°C
```

---

## Step 424: Analog Output Wiring

```
┌──────────────────────────────────────────────────────────┐
│           Analog Output Connections                      │
│                                                          │
│  Current Output (4-20mA) to Control Valve:              │
│                                                          │
│    PLC AO (+) ──────────────── Valve IN (+)             │
│    PLC AO (-) ──────────────── Valve IN (-)             │
│                                                          │
│  External Power Supply (for High-Load Actuators):        │
│                                                          │
│    +24V ──────┬───────────────────────────────          │
│               │                                         │
│            ┌──┴──┐                                      │
│            │ AO  │──────────────── Load (+)             │
│            └──┬──┘                                      │
│               │                                         │
│            0V ─────────────────── Load (-)              │
│                                                          │
│  Voltage Output (0-10V):                                │
│    PLC AO_V+ ─── Device IN                             │
│    PLC AO_GND ── Device GND (ต้องมี Common Ground)     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 425: Alarm Limits สำหรับ Analog

```
// ระบบ Alarm สำหรับ Analog Values
// 4 ระดับ: HH (High-High), H (High), L (Low), LL (Low-Low)

// ตัวอย่าง: Tank Level Monitoring
// Level Range: 0-100%
// LL = 10% (Pump Damage)
// L  = 20% (Low Warning)
// H  = 80% (High Warning)  
// HH = 90% (Overflow Risk)

// Allen-Bradley ControlLogix:
// Level_PV (REAL) = Current Level %
// Level_LL_Alarm, Level_L_Alarm,
// Level_H_Alarm, Level_HH_Alarm (BOOL)

RUNG 1: // HH Alarm
    ┌──[GEQ Level_PV, 90.0]──────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────( )─────────┐
       │                                 Level_HH_Alarm   │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                        Emergency_Pump_Off │
                                                          └┘

RUNG 2: // H Alarm
    ┌──[GEQ Level_PV, 80.0]──────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                          Level_H_Alarm   │
                                                         └┘

RUNG 3: // L Alarm
    ┌──[LEQ Level_PV, 20.0]──────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                          Level_L_Alarm   │
                                                         └┘

RUNG 4: // LL Alarm - Critical
    ┌──[LEQ Level_PV, 10.0]──────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────( )─────────┐
       │                                 Level_LL_Alarm   │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────( )─────────┐
                                        Pump_Shutdown     │
                                                         └┘
```

---

## Step 426: Analog Trending และ Peak Hold

```
// Peak Hold: เก็บค่าสูงสุดและต่ำสุดที่เคยเกิดขึ้น

// Tags:
// Pressure_PV (REAL) = Current Pressure
// Pressure_Max (REAL) = Peak Pressure
// Pressure_Min (REAL) = Minimum Pressure
// Reset_Peak (BOOL) = Reset Peak Values

RUNG 1: // Update Peak Maximum
    ┌──[GRT Pressure_PV, Pressure_Max]──────────────────┐
    │  Current > Peak?                                   │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────(MOV Pressure_PV)──┐
                                         → Pressure_Max  │
                                                         └┘

RUNG 2: // Update Peak Minimum
    ┌──[LES Pressure_PV, Pressure_Min]──────────────────┐
    │  Current < Minimum?                                │
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────(MOV Pressure_PV)──┐
                                         → Pressure_Min  │
                                                         └┘

RUNG 3: // Reset Peak Values
    ┌──[Reset_Peak ONS]─────────────────────────────────┐
    │  One Shot on Reset                                 │
    └──┬─────────────────────────────────────────────────┘
       ├──────────────────────────────(MOV Pressure_PV)──┐
       │                                 → Pressure_Max  │
       └─────────────────────────────────────────────────┘
       └──────────────────────────────(MOV Pressure_PV)──┐
                                         → Pressure_Min  │
                                                         └┘
```

---

## Step 427: ตัวอย่างสมบูรณ์ - ระบบวัดน้ำหนัก

```
// Weighing System
// Load Cell + Transmitter → 4-20mA → AI Module
// Range: 0-1000 kg

// ────────────────────────────────────────────────────
// CALIBRATION PROCEDURE:
// 1. Zero with empty scale: Raw_Zero = IW64
// 2. Apply known weight (500 kg): Raw_500kg = IW64
// 3. Span = Raw_500kg - Raw_Zero
// 4. Scale Factor = 500 / Span kg/count
// ────────────────────────────────────────────────────

// Tags:
// Weight_Raw (INT) = Raw AI value
// Weight_Zero (INT) = Zero calibration
// Weight_Span (INT) = Full span counts
// Weight_kg (REAL) = Engineering value in kg
// Tare_Weight (REAL) = Tare/subtract value
// Net_Weight (REAL) = Gross - Tare

NETWORK 1: // Read Raw
    ┌────────────────────────────────────────────────────┐
    │  MOVE: IW64 → Weight_Raw                           │
    └────────────────────────────────────────────────────┘

NETWORK 2: // Tare Function (Zero with load)
    ┌──[Tare_Button ONS]────────────────────────────────┐
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────(MOV Weight_kg)────┐
                                         → Tare_Weight   │
                                                         └┘

NETWORK 3: // Calculate Gross Weight
    ┌────────────────────────────────────────────────────┐
    │  CPT:                                              │
    │  Weight_kg = (Weight_Raw - Weight_Zero) *         │
    │               1000.0 / Weight_Span               │
    └────────────────────────────────────────────────────┘

NETWORK 4: // Calculate Net Weight
    ┌────────────────────────────────────────────────────┐
    │  SUB: Weight_kg - Tare_Weight → Net_Weight        │
    └────────────────────────────────────────────────────┘

NETWORK 5: // Fill Control - เติมจนถึง Target Weight
    ┌──[Fill_Running]──[LES Net_Weight, Target_Weight]──┐
    └──┬─────────────────────────────────────────────────┘
       └──────────────────────────────────────( )────────┐
                                           Fill_Valve    │
                                                         └┘
```

---

## Step 428: การแก้ปัญหา Analog Signal Noise

```
┌──────────────────────────────────────────────────────────┐
│            Analog Troubleshooting Guide                  │
│                                                          │
│  ปัญหา: Signal กระโดดไปมา (Noisy)                      │
│  ────────────────────────────────────────────────────── │
│  สาเหตุที่เป็นไปได้:                                    │
│  1. Ground Loop - หลาย Ground Point ต่างศักย์          │
│     แก้: ใช้ Isolated AI Module หรือ Signal Isolator   │
│                                                          │
│  2. ไม่ได้ใช้ Shielded Cable                            │
│     แก้: ใช้ Shielded Cable ต่อ Shield ที่ปลายเดียว    │
│           (Controller End Only)                          │
│                                                          │
│  3. Cable วิ่งใกล้ Power Cable                          │
│     แก้: แยก Signal Cable ออกห่าง Power > 30cm         │
│                                                          │
│  4. Mechanical Vibration                                │
│     แก้: Software Filter (Moving Average)               │
│                                                          │
│  5. Poor Power Supply                                   │
│     แก้: ใช้ Separate Power Supply สำหรับ AI           │
│                                                          │
│  ปัญหา: ค่า Offset (ผิดตลอด)                           │
│  ────────────────────────────────────────────────────── │
│  1. Zero Calibration ผิด → Re-calibrate                 │
│  2. Input Range ตั้งผิด → Check Hardware Config         │
│  3. Scaling Formula ผิด → Check Math                    │
│  4. Wrong Sensor Type → Verify TC/RTD Type              │
└──────────────────────────────────────────────────────────┘
```

---

## Step 429: Analog I/O Summary Table

| Parameter | 4-20mA | 0-10V | TC Type K | PT100 |
|-----------|--------|-------|-----------|-------|
| Range | 4-20mA | 0-10V | -200~1260°C | -200~850°C |
| Siemens Raw | 0-27648 | 0-27648 | ×10 °C | ×10 °C |
| AB Raw | 0-32767 | 0-32767 | Module-dep | Module-dep |
| Wire Break | < 3.6mA | N/A | Error code | Error code |
| Accuracy | ±0.1% | ±0.1% | ±1-3°C | ±0.1°C |
| Cable Length | 1000m | 50m | 100m+Ext | 100m |
| Interference | Low | High | Very Low | Low |

---

## Step 430: Lab Project - ระบบ Multi-Parameter Monitoring

**โจทย์:** สร้างระบบ Monitoring สำหรับ Chemical Tank

```
┌──────────────────────────────────────────────────────────┐
│         Chemical Tank Monitoring System                  │
│                                                          │
│  Sensors:                                               │
│  AI0: Level Transmitter 4-20mA → 0-100%                 │
│  AI1: Temperature Transmitter 4-20mA → 0-150°C          │
│  AI2: Pressure Transmitter 4-20mA → 0-10 Bar            │
│  AI3: pH Meter 4-20mA → pH 0-14                        │
│                                                          │
│  Actuators:                                             │
│  AO0: Fill Valve Position 4-20mA → 0-100%              │
│  AO1: Cooling Water Valve 4-20mA → 0-100%              │
│                                                          │
│  Requirements:                                          │
│  1. Scale ทุก Input เป็น Engineering Units              │
│  2. Moving Average Filter (16 samples) ทุก Channel      │
│  3. Alarm: HH/H/L/LL ทุก Channel                       │
│  4. Log Peak Values พร้อม Reset                         │
│  5. Manual Override ทุก Output                          │
│  6. Safety Interlock: ถ้า Level > 90% → Close Fill Valve│
│  7. ถ้า Temp > 80°C → Open Cooling Valve 100%           │
└──────────────────────────────────────────────────────────┘
```

**เฉลยแนวทาง:**

```
// ────────────────────────────────────────────────────
// STEP 1: I/O Configuration
// STEP 2: Scaling Function (ใช้ FB หรือ Function)
// STEP 3: Filter Function (16-sample Moving Average)
// STEP 4: Alarm Logic (4 ระดับ ทุก Channel)
// STEP 5: Peak Hold Logic
// STEP 6: Output Control Logic
// STEP 7: Safety Interlock
// STEP 8: HMI (ถ้ามี)
// ────────────────────────────────────────────────────

// Scaling FB (นำกลับมาใช้ได้ทุก Channel)
// Input: Raw (INT), Raw_Min (INT), Raw_Max (INT)
//        Eng_Min (REAL), Eng_Max (REAL)
// Output: Eng_Value (REAL)
//
// Formula: Eng = Eng_Min + (Raw-Raw_Min)*(Eng_Max-Eng_Min)
//                          / (Raw_Max-Raw_Min)
```

---

## สรุป Part 22

ใน Part นี้ได้เรียนรู้:

1. **ประเภท Analog Signal** - 4-20mA, 0-10V, TC, RTD
2. **ADC Resolution** - Bits, Count Range, Accuracy
3. **Scaling Formulas** - Linear Scaling, NORM_X/SCALE_X
4. **Dead Band** - ป้องกัน Chatter ที่ Setpoint
5. **Noise Filtering** - Moving Average Algorithm
6. **Signal Conditioning** - Isolation, Amplification
7. **Analog Output** - Scaling สำหรับ Actuators
8. **Flow Control** - Simple Control Valve Application
9. **TC/RTD** - Sensor Types, CJC, Configuration
10. **Alarm Limits** - HH/H/L/LL Alarm Structure

---

## Navigation
- [← Part 21: Sequencer และ State Machine](part21_sequencer.md)
- [→ Part 23: PID Control](part23_pid_control.md)
- [กลับหน้าหลัก](../README.md)
