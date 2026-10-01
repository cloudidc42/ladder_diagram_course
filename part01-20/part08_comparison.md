# Part 08: Comparison Instructions
## Steps 131-150: การเปรียบเทียบค่าและการควบคุมแบบ Threshold

---

## สารบัญ

- [Step 131: ภาพรวม Comparison Instructions](#step-131)
- [Step 132: EQ - Equal (เท่ากัน)](#step-132)
- [Step 133: NEQ - Not Equal (ไม่เท่ากัน)](#step-133)
- [Step 134: GT - Greater Than (มากกว่า)](#step-134)
- [Step 135: GTE - Greater Than or Equal (มากกว่าหรือเท่ากัน)](#step-135)
- [Step 136: LT - Less Than (น้อยกว่า)](#step-136)
- [Step 137: LTE - Less Than or Equal (น้อยกว่าหรือเท่ากัน)](#step-137)
- [Step 138: LIM - Limit Test](#step-138)
- [Step 139: Comparison Chaining](#step-139)
- [Step 140: Level Control with Hysteresis](#step-140)
- [Step 141: Speed Monitoring](#step-141)
- [Step 142: Pressure Control with Deadband](#step-142)
- [Step 143: Temperature Control with Hysteresis](#step-143)
- [Step 144: Count-Based Control](#step-144)
- [Step 145: Time-Based Comparison](#step-145)
- [Step 146: Analog Setpoint Adjustment](#step-146)
- [Step 147: Multi-Zone Control](#step-147)
- [Step 148: Quality Control with Tolerance Bands](#step-148)
- [Step 149: Cascade Control](#step-149)
- [Step 150: Workshop - Complete Process Monitor](#step-150)

---

## Step 131: ภาพรวม Comparison Instructions {#step-131}

### Comparison ทำงานอย่างไร?

```
Comparison Instruction = ใช้เป็น Contact ใน Ladder Diagram
  → ถ้าเงื่อนไขเป็นจริง: ผ่าน (Closed) เหมือน NC contact
  → ถ้าเงื่อนไขเป็นเท็จ: ไม่ผ่าน (Open) เหมือน NO contact

ตัวอย่าง:
  ─── [>  D0 K50] ──      ← ถ้า D0 > 50 → Circuit ปิด (pass)
                           ← ถ้า D0 ≤ 50 → Circuit เปิด (no pass)
```

### Comparison Instruction Set

```
┌─────────┬────────────────────┬────────────────────────────────┐
│ Symbol  │ Condition          │ True เมื่อ                     │
├─────────┼────────────────────┼────────────────────────────────┤
│ EQ (=)  │ Equal              │ A = B                          │
│ NEQ(≠)  │ Not Equal          │ A ≠ B                          │
│ GT (>)  │ Greater Than       │ A > B                          │
│ GTE(≥)  │ Greater or Equal   │ A ≥ B                          │
│ LT (<)  │ Less Than          │ A < B                          │
│ LTE(≤)  │ Less or Equal      │ A ≤ B                          │
│ LIM     │ Limit Test         │ Min ≤ A ≤ Max                  │
└─────────┴────────────────────┴────────────────────────────────┘
```

### Syntax Comparison - 3 แบรนด์

```
┌──────────────────┬─────────────────────────────────────────────────┐
│ แบรนด์           │ Greater Than (GT) Syntax                        │
├──────────────────┼─────────────────────────────────────────────────┤
│ Siemens S7-1200  │ ─── [CMP >I] ── หรือ ─── [>I] ───              │
│                  │     IN1: %MW100  IN2: 500                       │
│                  │                                                 │
│ Allen-Bradley    │ ─── [GRT] ───                                   │
│ SLC-500          │     Source A: N7:0    Source B: N7:1            │
│                  │                                                 │
│ Mitsubishi FX    │ ─── [>  D0 K500] ───                            │
│                  │ หรือ [CMP K500 D0 M0] (CMP instruction)         │
└──────────────────┴─────────────────────────────────────────────────┘
```

---

## Step 132: EQ - Equal (เท่ากัน) {#step-132}

### ใช้งาน EQ

```
ตัวอย่าง: ตรวจสอบว่า Step ปัจจุบันเท่ากับ Step ที่ต้องการ

     [= D10 K3]           Q0.3
  ─────┤ ├─────────────────( )───    ← ถ้า Step = 3 → เปิด Output 3
   D10 = 3?

     [= D10 K5]           Q0.5
  ─────┤ ├─────────────────( )───    ← ถ้า Step = 5 → เปิด Output 5
   D10 = 5?
```

### ตัวอย่าง: Counter แสดงผลตาม Count

```
Network 1: Counter
     I0.0              C0
  ───┤ ├──────────────[CTU C0 K100]─

Network 2: ทำงานเมื่อนับครบ 10
     [= C0 K10]           M20.0
  ─────┤ ├─────────────────( )───    ← M20.0 ON เมื่อ Count = 10

Network 3: ทำงานเมื่อนับครบ 50
     [= C0 K50]           M20.1
  ─────┤ ├─────────────────( )───    ← M20.1 ON เมื่อ Count = 50

Network 4: ทำงานเมื่อนับครบ 100
     [= C0 K100]          M20.2
  ─────┤ ├─────────────────( )───    ← M20.2 ON เมื่อ Count = 100
```

### Truth Table: EQ

| D0 | K | ผลลัพธ์ EQ |
|----|---|------------|
| 50 | 50 | TRUE (1) |
| 49 | 50 | FALSE (0) |
| 51 | 50 | FALSE (0) |
| 0  | 0  | TRUE (1) |
| -5 | -5 | TRUE (1) |

---

## Step 133: NEQ - Not Equal (ไม่เท่ากัน) {#step-133}

### ใช้งาน NEQ

```
ตัวอย่าง: ทำงานเมื่อค่าไม่ตรงกับ Setpoint (Deviation Detection)

     [<> D10 D11]         M30.0
  ─────┤ ├─────────────────( )───    ← ON เมื่อ Actual ≠ Setpoint
   Actual ≠ Setpoint

ใช้บ่อย: ตรวจสอบว่า Sensor ยังทำงานอยู่
     [<> D0 K0]           M31.0
  ─────┤ ├─────────────────( )───    ← ON เมื่อ Sensor มีค่า (ไม่ใช่ 0)
   Signal > 0?
```

### ตัวอย่าง: ตรวจสอบ Sensor Health

```
ถ้า 4-20mA sensor ส่งค่า 0 → สาย断 หรือ sensor เสีย

Network 1: ตรวจสอบ Sensor
     [<> D0 K0]           M35.0
  ─────┤ ├─────────────────( )───    ← Sensor OK ถ้า ADC ≠ 0

     M35.0             Q0.3
  ───┤/├────────────────( )───       ← Sensor Fault Lamp ถ้า ADC = 0
  Sensor OK
```

---

## Step 134: GT - Greater Than (มากกว่า) {#step-134}

### ตัวอย่าง: High Level Alarm

```
สถานการณ์: ถังน้ำ ระดับ HIGH ALARM เมื่อเกิน 90%

  Level = D20 (% × 10, 0-1000)
  HIGH ALARM = 900 (90.0%)


Network 1: High Level Alarm
     [> D20 K900]         M100.0
  ─────┤ ├─────────────────( S )───  ← Set Alarm เมื่อ Level > 90%
   Level > 90%?

Network 2: Clear Alarm เมื่อลดลงมา
     [<= D20 K850]        M100.0
  ─────┤ ├─────────────────( R )───  ← Reset Alarm เมื่อ Level ≤ 85%
   Level ≤ 85%?        (Hysteresis 5%)
```

### ตัวอย่าง: Over Speed Detection

```
  Motor_Speed = D50 (RPM)
  Max_Speed   = D51 (RPM, กำหนดไว้ใน Setpoint)


Network 1: Overspeed Fault
     [> D50 D51]           M110.0
  ─────┤ ├──────────────────( S )───  ← Fault เมื่อ Speed > Max
   Speed > Max?

Network 2: หยุด Motor เมื่อ Overspeed
     M110.0             M2.0
  ───┤ ├────────────────( R )───     ← Stop motor
  Overspeed
```

---

## Step 135: GTE - Greater Than or Equal {#step-135}

### ตัวอย่าง: Batch Complete

```
สถานการณ์: เติมน้ำจนถึงระดับที่กำหนด

  Current_Level = D20 (mm)
  Target_Level  = D21 (mm)


Network 1: หยุดเติมน้ำเมื่อถึงระดับเป้าหมาย
     [>= D20 D21]         Q0.0
  ─────┤/├─────────────────( )───    ← Fill Valve (ปิดเมื่อถึงเป้า)
   Level ≥ Target?
```

### GTE vs GT

```
ความแตกต่าง:
  GT  (>):  ผ่านเมื่อ D20 = 1001 ถ้า Target = 1000
  GTE (≥):  ผ่านเมื่อ D20 = 1000 ถ้า Target = 1000 ด้วย

สำหรับ Level Control: ใช้ GTE เพื่อให้หยุดพอดีที่ Target
```

---

## Step 136: LT - Less Than (น้อยกว่า) {#step-136}

### ตัวอย่าง: Low Level Control

```
สถานการณ์: เปิดปั๊มเมื่อระดับน้ำต่ำ

  Level = D20 (% × 10)
  START pumping เมื่อ Level < 200 (20.0%)


Network 1: เปิดปั๊ม
     [< D20 K200]         M5.0
  ─────┤ ├─────────────────( S )───  ← Start Pump เมื่อ Level < 20%
   Level < 20%?

Network 2: ปิดปั๊ม
     [>= D20 K800]        M5.0
  ─────┤ ├─────────────────( R )───  ← Stop Pump เมื่อ Level ≥ 80%
   Level ≥ 80%?
```

---

## Step 137: LTE - Less Than or Equal {#step-137}

### ตัวอย่าง: Cooling System

```
สถานการณ์: เปิด Cooling เมื่ออุณหภูมิสูงเกิน Setpoint

  Temperature = D10 (°C × 10)
  Cooling_SP  = D11 (°C × 10)


Network 1: Cooling ON เมื่อ Temp > SP
     [> D10 D11]           M7.0
  ─────┤ ├──────────────────( S )───  ← Cooling ON

Network 2: Cooling OFF เมื่อ Temp ≤ SP - Deadband
     D12                               D13
  ──[SUB D11 K100 D13]── [<= D10 D13]──( R M7.0 )─  ← OFF เมื่อ ≤ SP-10°C
  SP - 10°C
```

---

## Step 138: LIM - Limit Test {#step-138}

### LIM คืออะไร?

**LIM (Limit Test)** ทดสอบว่าค่าอยู่ **ภายใน** หรือ **ภายนอก** ช่วงที่กำหนด

```
LIM Test:
  ถ้า Low ≤ Test Value ≤ High → TRUE (IN RANGE)
  ถ้า Test Value < Low หรือ > High → FALSE (OUT OF RANGE)

Syntax Allen-Bradley:
  [LIM]
  Low Limit:  500
  Test:       N7:0
  High Limit: 1500
  → TRUE เมื่อ 500 ≤ N7:0 ≤ 1500
```

### ตัวอย่าง: ตรวจสอบค่าอยู่ในช่วง

```
สถานการณ์: Temperature ต้องอยู่ในช่วง 70-90°C สำหรับ Process

  Temperature = D10 (°C × 10)
  Low Limit   = 700 (70.0°C)
  High Limit  = 900 (90.0°C)


Network 1 (Mitsubishi): ใช้ 2 Comparison Blocks

     [>= D10 K700]     [<= D10 K900]     M15.0
  ──────┤ ├──────────────┤ ├──────────────( )───
   Temp ≥ 70°C       Temp ≤ 90°C       In Range

     M15.0             Q0.5
  ───┤/├────────────────( )───          ← Out of Range Alarm
  In_Range


Network 2 (Allen-Bradley): ใช้ LIM Instruction

     [LIM: 700 ≤ D10 ≤ 900]      M15.0
  ──────────┤ ├────────────────────( )───
         In Range?
```

### LIM สำหรับ Safety Monitoring

```
สถานการณ์: ตรวจสอบค่าหลายพารามิเตอร์ว่าอยู่ในช่วง safe

Network 1: Pressure in Safe Range?
     [>= D30 K500]     [<= D30 K2000]     M20.0
  ──────┤ ├──────────────┤ ├──────────────( )───
  Pres ≥ 50kPa      Pres ≤ 200kPa    Pressure_OK

Network 2: Temperature in Safe Range?
     [>= D10 K100]     [<= D10 K1500]     M20.1
  ──────┤ ├──────────────┤ ├──────────────( )───
  Temp ≥ 10°C       Temp ≤ 150°C    Temperature_OK

Network 3: All Parameters OK → Allow Process
     M20.0           M20.1             Q5.0
  ───┤ ├──────────────┤ ├──────────────( )───
  Pres_OK         Temp_OK          Process_Enable
```

---

## Step 139: Comparison Chaining {#step-139}

### การเชื่อม Comparison หลายตัว

```
AND Conditions (ทุกเงื่อนไขต้องจริง):

     [> D0 K100]    [< D0 K500]    [= D1 K1]     M25.0
  ──────┤ ├───────────┤ ├────────────┤ ├────────────( )───
   D0 > 100       D0 < 500       D1 = 1        All Conditions Met


OR Conditions (เงื่อนไขใดก็ได้ต้องจริง):

     [< D0 K50]                      M26.0
  ──────┤ ├────────────────────────────( )───
   D0 < 50

     [> D0 K950]                     M26.0
  ──────┤ ├────────────────────────────( )───    ← M26.0 = D0 < 50 OR D0 > 950
   D0 > 950
```

### ตัวอย่าง: Complex Safety Interlock

```
Process ทำงานได้เมื่อ:
  1. Temperature 60-120°C (อยู่ในช่วง)
  2. Pressure 1-10 Bar
  3. Flow > 5 L/min
  4. E-Stop ไม่ได้กด


Network 1: ตรวจสอบ Temperature Range
     [>= D10 K600]    [<= D10 K1200]     M40.0
  ──────┤ ├──────────────┤ ├──────────────( )───
  Temp ≥ 60°C       Temp ≤ 120°C      Temp_OK

Network 2: ตรวจสอบ Pressure Range
     [>= D20 K100]    [<= D20 K1000]     M40.1
  ──────┤ ├──────────────┤ ├──────────────( )───
  Pres ≥ 1 Bar      Pres ≤ 10 Bar     Pres_OK

Network 3: ตรวจสอบ Flow
     [> D30 K50]                         M40.2
  ──────┤ ├────────────────────────────────( )───
  Flow > 5 L/min                       Flow_OK

Network 4: Process Enable (ทุกเงื่อนไขต้องผ่าน)
     M40.0          M40.1          M40.2          I15.0          Q5.0
  ───┤ ├─────────────┤ ├─────────────┤ ├────────────┤/├────────────( )───
  Temp_OK         Pres_OK        Flow_OK        E_Stop       Process_Enable
```

---

## Step 140: Level Control with Hysteresis {#step-140}

### Hysteresis (Deadband) คืออะไร?

```
ปัญหาโดยไม่มี Hysteresis:
  Setpoint = 500mm
  → ระดับ 500: ปั๊มหยุด
  → ระดับ 499: ปั๊มเดิน
  → ระดับ 500: ปั๊มหยุด
  → ระดับ 499: ปั๊มเดิน
  → ปั๊มเปิด/ปิดเร็วมาก! (Chattering) → อายุสั้น

แก้ด้วย Hysteresis:
  Start Point = 300mm (เปิดปั๊มเมื่อ < 300mm)
  Stop Point  = 700mm (ปิดปั๊มเมื่อ > 700mm)
  Hysteresis  = 400mm
```

```
ไดอะแกรม Level Control with Hysteresis:

Level (mm)
  ├──────────────────────────────────────────────────────────────── 1000
  │                                                          HIGH ALARM
  ├──────────────────────────────────────────────────────────────── 800
  │                ←─────────────────── Pump OFF (≥ 700mm) ───────  700
  │                                                                  │
  │                                                            NORMAL │
  │                                    ──────────────────────────────│
  │                                                                   │
  │                ←─────────────────── Pump ON  (≤ 300mm) ───────  300
  ├──────────────────────────────────────────────────────────────── 200
  │                                                          LOW ALARM
  ├──────────────────────────────────────────────────────────────── 0
```

### โปรแกรม Level Control with Hysteresis

```
  Level = D20 (mm)
  Pump_Flag = M5.0


Network 1: เปิดปั๊มเมื่อ Level ต่ำกว่า LOW setpoint
     [<= D20 K300]        M5.0
  ─────┤ ├─────────────────( S )───   ← Start Pump เมื่อ Level ≤ 300mm

Network 2: ปิดปั๊มเมื่อ Level สูงกว่า HIGH setpoint
     [>= D20 K700]        M5.0
  ─────┤ ├─────────────────( R )───   ← Stop Pump เมื่อ Level ≥ 700mm

Network 3: Pump Output
     M5.0              I15.0           Q0.0
  ───┤ ├───────────────┤/├─────────────( )───
  Pump_Flag          E_Stop          Pump_Motor

Network 4: High Alarm
     [> D20 K900]          M100.0
  ─────┤ ├──────────────────( S )───

Network 5: Low Alarm
     [< D20 K100]          M100.1
  ─────┤ ├──────────────────( S )───
```

### Truth Table: Level Control States

| Level (mm) | Pump Before | Pump After | Action |
|------------|-------------|------------|--------|
| 250 | OFF | ON | Start pump (< 300) |
| 400 | ON | ON | Keep running |
| 600 | ON | ON | Keep running |
| 700 | ON | OFF | Stop pump (≥ 700) |
| 500 | OFF | OFF | Stay off |
| 299 | OFF | ON | Start pump |

---

## Step 141: Speed Monitoring {#step-141}

### ระบบ Speed Monitoring สำหรับมอเตอร์

```
ข้อกำหนดระบบ:
  Nominal Speed: 1500 RPM
  Overspeed:     1650 RPM (110%)
  Underspeed:    1350 RPM (90%)
  Zero Speed:    50 RPM

  Speed = D50 (RPM)


Network 1: Zero Speed Detection
     [<= D50 K50]         M60.0
  ─────┤ ├─────────────────( )───    ← M60.0 = Motor Stopped

Network 2: Underspeed Warning
     [< D50 K1350]        M60.1
  ─────┤ ├─────────────────( )───    ← M60.1 = Underspeed

     [> D50 K50]                     ← ต้องไม่ใช่ zero speed
                                       (ไม่แสดง underspeed ตอน stop)

Network 3: Normal Speed Range
     [>= D50 K1350]    [<= D50 K1650]      M60.2
  ──────┤ ├──────────────┤ ├───────────────( )───
  Speed ≥ 1350       Speed ≤ 1650      Normal_Speed

Network 4: Overspeed Fault
     [> D50 K1650]         M60.3
  ─────┤ ├─────────────────( S )───   ← Overspeed Fault (Latched)

Network 5: Emergency Shutdown on Overspeed
     M60.3              M1.0
  ───┤ ├────────────────( R )───      ← Stop motor
  Overspeed          Motor_Run
```

### Speed Ramp Monitoring

```
สถานการณ์: ตรวจสอบว่ามอเตอร์เร่งความเร็วตามปกติ

  Expected Speed = D55 (ตาม Ramp function)
  Actual Speed   = D50
  Speed Error    = D56
  MAX_ERROR      = 100 RPM


Network 1: คำนวณ Speed Error
     M8000
  ───┤ ├────[SUB D55 D50 D56]─        ← Error = Expected - Actual

     M8000
  ───┤ ├────[ABS D56 D57]─            ← |Error|

Network 2: Alarm ถ้า Error เกิน 100 RPM
     [> D57 K100]         M61.0
  ─────┤ ├─────────────────( )───     ← Speed Tracking Error Alarm
```

---

## Step 142: Pressure Control with Deadband {#step-142}

### Pressure Control System

```
ข้อกำหนด:
  Setpoint:      8.0 Bar (D80 = 800)
  Deadband:      ±0.5 Bar
  High Setpoint: 8.5 Bar (D80 + 50) → ปิด Compressor
  Low Setpoint:  7.5 Bar (D80 - 50) → เปิด Compressor
  HIGH ALARM:    9.5 Bar
  LOW ALARM:     6.5 Bar
  EMERGENCY:     10.0 Bar

  Pressure = D81 (Bar × 100)
  High_SP   = D82 = D80 + 50
  Low_SP    = D83 = D80 - 50


Network 1: คำนวณ High/Low Setpoints จาก Deadband
     M8000
  ───┤ ├────[ADD D80 K50 D82]─        ← High SP = Setpoint + Deadband
  ───┤ ├────[SUB D80 K50 D83]─        ← Low SP = Setpoint - Deadband


Network 2: Compressor ON (Pressure ต่ำกว่า Low SP)
     [<= D81 D83]         M70.0
  ─────┤ ├─────────────────( S )───   ← Start Compressor

Network 3: Compressor OFF (Pressure เกิน High SP)
     [>= D81 D82]         M70.0
  ─────┤ ├─────────────────( R )───   ← Stop Compressor

Network 4: High Pressure Alarm
     [>= D81 K950]        M71.0
  ─────┤ ├─────────────────( S )───   ← HIGH ALARM

Network 5: Low Pressure Alarm
     [<= D81 K650]        M71.1
  ─────┤ ├─────────────────( S )───   ← LOW ALARM

Network 6: Emergency Shutdown
     [>= D81 K1000]       M70.0       M71.2
  ─────┤ ├─────────────────( R )────   ( S )───
  P ≥ 10 Bar           Stop ALL       EMERGENCY
```

### Pressure Control State Diagram

```
State Diagram:

     P ≤ 7.5 Bar          P ≥ 8.5 Bar
  ┌──────────┐          ┌──────────┐
  │COMPRESSOR│──────────│COMPRESSOR│
  │  ON      │          │  OFF     │
  └──────────┘          └──────────┘
       │    P ≥ 8.5 Bar →    │
       └────────────────────┘
       │    P ≤ 7.5 Bar ←    │
       └────────────────────┘

     P ≥ 9.5 Bar: HIGH ALARM
     P ≤ 6.5 Bar: LOW ALARM
     P ≥ 10.0 Bar: EMERGENCY SHUTDOWN
```

---

## Step 143: Temperature Control with Hysteresis {#step-143}

### Oven/Furnace Temperature Control

```
ข้อกำหนดเตาอบอุตสาหกรรม:
  Setpoint:     200°C (D10 = 2000)
  Deadband:     ±5°C
  Heater ON:    เมื่อ Temp < 195°C (D10 < 1950)
  Heater OFF:   เมื่อ Temp > 205°C (D10 > 2050)
  HIGH ALARM:   220°C (D10 > 2200)
  LOW ALARM:    170°C (D10 < 1700)

  Temperature = D10 (°C × 10)
  Setpoint     = D11 (°C × 10, จาก HMI)


Network 1: คำนวณ Band จาก Setpoint
     M8000
  ───┤ ├────[SUB D11 K50 D13]─        ← Low Band = SP - 5°C
  ───┤ ├────[ADD D11 K50 D14]─        ← High Band = SP + 5°C


Network 2: Heater ON
     [< D10 D13]          M80.0
  ─────┤ ├─────────────────( S )───   ← เปิด Heater เมื่อต่ำกว่า Low Band

Network 3: Heater OFF
     [> D10 D14]          M80.0
  ─────┤ ├─────────────────( R )───   ← ปิด Heater เมื่อเกิน High Band

Network 4: Heater Output
     M80.0              I20.0          Q2.0
  ───┤ ├───────────────┤/├─────────────( )───
  Heater_Flag        E_Stop          Heater

Network 5: Over Temperature Alarm
     [> D10 K2200]        M85.0
  ─────┤ ├─────────────────( S )───   ← OVER TEMP ALARM

Network 6: Under Temperature Alarm
     [< D10 K1700]        M85.1
  ─────┤ ├─────────────────( S )───   ← UNDER TEMP ALARM

Network 7: Alarm Indicator
     M85.0             M8013           Q2.7
  ───┤ ├──────────────┤ ├──────────────( )───
  Any_Alarm         1sec_Clock       Alarm_Lamp

     M85.1             M8013
  ───┤ ├──────────────┤ ├──────────────( )───
```

### Temperature Hysteresis Graph

```
°C
 220 ──────────────────────────────────── HIGH ALARM
     |
 205 ─────────────────────────────────── HEATER OFF
     |                    ↑ Heater Cycles Here ↑
 200 ─────────────────────────────────── SETPOINT
     |                    ↓
 195 ─────────────────────────────────── HEATER ON
     |
 170 ──────────────────────────────────── LOW ALARM

Time: ──────────────────────────────────>
      Heater:  ON   OFF ON  OFF ON  OFF
```

---

## Step 144: Count-Based Control {#step-144}

### ตัวอย่าง: ระบบ Batch Control ด้วย Count

```
สถานการณ์: บรรจุขวด 100 ขวดต่อ Batch

  Count = D200 (นับจาก Sensor)
  Batch_Size = 100


Network 1: นับขวด
     I5.0            M8.0           D200
  ───┤ ├────────────[INC D200]─     ← นับทุกครั้งที่ขวดผ่าน Sensor
  Bottle_Sensor

Network 2: Batch Complete เมื่อนับครบ 100
     [>= D200 K100]       M90.0
  ─────┤ ├─────────────────( )───   ← Batch Complete Flag

Network 3: หยุดสายพานเมื่อ Batch Complete
     M90.0             M2.0
  ───┤ ├────────────────( R )───    ← Stop Conveyor

Network 4: Reset Count สำหรับ Batch ถัดไป
     I5.5              D200
  ───┤ ├────────────[MOV K0 D200]─  ← Reset Button → Clear Count
  Reset_Batch
```

### Batch Size Adjustment ด้วย HMI

```
  Batch_Size = D201 (กำหนดจาก HMI, default = 100)


Network 1: Batch Complete เมื่อถึงขนาดที่กำหนด
     [>= D200 D201]       M90.0
  ─────┤ ├─────────────────( )───   ← Batch Complete (dynamic size)
   Count ≥ Batch_Size?
```

---

## Step 145: Time-Based Comparison {#step-145}

### เปรียบเทียบค่า Timer กับ Threshold

```
สถานการณ์: Cycle Time Monitoring
  ถ้า Machine Cycle ใช้เวลานานเกิน 30 วินาที → แจ้งเตือน
  ถ้า Cycle เร็วเกินไป (< 5 วินาที) → แจ้งเตือน

  Cycle_Timer = T5 (นับเวลา Cycle)
  T5.ET = Elapsed Time ใน 100ms units


Network 1: Start Cycle Timer
     I6.0              M8000           T5
  ───┤ ├────────────────[TON T5 300s]─  ← max 30 วินาที

Network 2: ตรวจ Overtime
     [> T5.ET K300]      M95.0
  ─────┤ ├─────────────────( )───        ← Cycle > 30s (300 × 100ms = 30s)

Network 3: ตรวจ Fast Cycle (< 5 วินาที)
     [< T5.ET K50]       M95.1
  ─────┤ ├─────────────────( )───        ← Cycle < 5s
```

---

## Step 146: Analog Setpoint Adjustment {#step-146}

### Dynamic Setpoint จาก Potentiometer หรือ HMI

```
สถานการณ์: อนุญาตให้ Operator ปรับ Setpoint ผ่าน Panel

  Raw_SP = AIW0 (0-27648 Siemens, 0-100% จาก Pot)
  Min_SP = 50 (50°C × 10)
  Max_SP = 1500 (150°C × 10)
  Actual_SP = D300

สูตร: Actual_SP = Raw_SP × (Max_SP - Min_SP) / 27648 + Min_SP


Network 1: Scale Pot Input เป็น Setpoint

  Siemens:
    [SUB AIW0 0 MW200]─────────────────────────────────
    [MUL MW200 1450 MD202]─  (1450 = 1500-50)
    [DIV_I MD202 27648 MW210]─
    [ADD MW210 50 D300]─      ← D300 = Actual Setpoint
```

---

## Step 147: Multi-Zone Control {#step-147}

### ระบบควบคุมหลาย Zone

```
สถานการณ์: เตาอบ 3 Zone ควบคุมอุณหภูมิอิสระ

Zone 1: Preheating  SP=150°C
Zone 2: Processing  SP=200°C
Zone 3: Cooling     SP=80°C


Zone 1: D10=Actual, D11=SP=1500
Zone 2: D20=Actual, D21=SP=2000
Zone 3: D30=Actual, D31=SP=800

Heaters: Q0.0=Zone1, Q0.1=Zone2
Cooling: Q0.2=Zone3


Network 1-2: Zone 1 Heater Control
     [< D10 D11]          M110.0
  ─────┤ ├─────────────────( S )─── ← Zone 1 Heater ON

     [> D10 D11]          M110.0
  ─────┤ ├─────────────────( R )─── ← Zone 1 Heater OFF

// (ในทางปฏิบัติควรใช้ SP ± Deadband)
// (Network 3-6: Zone 2, 7-8: Zone 3 ทำเหมือนกัน)

Network 9: Zone Outputs
     M110.0             Q0.0
  ───┤ ├────────────────( )─── ← Zone 1 Heater

     M111.0             Q0.1
  ───┤ ├────────────────( )─── ← Zone 2 Heater

     M112.0             Q0.2
  ───┤ ├────────────────( )─── ← Zone 3 Cooler
```

---

## Step 148: Quality Control with Tolerance Bands {#step-148}

### ระบบ QC Check - วัด Dimension

```
สถานการณ์: วัดความยาวชิ้นงาน ตรวจสอบว่าอยู่ในพิกัด

  Nominal    = 1000 (100.0 mm)
  Tolerance  = ±5 (±0.5 mm)
  UPPER LIMIT = 1005
  LOWER LIMIT = 995
  REWORK BAND = ±8 (±0.8 mm)  ← ชิ้นงานที่อาจแก้ไขได้
  SCRAP LIMIT = >±10 mm


  Measurement = D60 (mm × 10)


Network 1: OK Grade (อยู่ในพิกัด ±0.5mm)
     [>= D60 K995]    [<= D60 K1005]     M120.0
  ──────┤ ├──────────────┤ ├───────────────( )─── ← Grade A: OK

Network 2: Rework (อยู่ในช่วง ±0.5 ถึง ±0.8mm)
     [>= D60 K992]    [<= D60 K1008]     M120.0   M120.1
  ──────┤ ├──────────────┤ ├──────────────┤/├───────( )─── ← Grade B: Rework

Network 3: Scrap (เกิน ±0.8mm)
     [< D60 K992]                         M120.2
  ──────┤ ├────────────────────────────────( )─── ← Scrap: Short

     [> D60 K1008]                        M120.2
  ──────┤ ├────────────────────────────────( )─── ← Scrap: Long

Network 4: Sorting Outputs
     M120.0             Q6.0
  ───┤ ├────────────────( )─── ← OK Chute

     M120.1             Q6.1
  ───┤ ├────────────────( )─── ← Rework Chute

     M120.2             Q6.2
  ───┤ ├────────────────( )─── ← Scrap Chute
```

### Measurement Statistics

```
เพิ่มฟีเจอร์ Statistics:

Network 5: นับแต่ละ Grade
     M120.0          M8.0       D70
  ───┤ ├─────────────────[INC D70]─   ← OK Count

     M120.1          M8.0       D71
  ───┤ ├─────────────────[INC D71]─   ← Rework Count

     M120.2          M8.0       D72
  ───┤ ├─────────────────[INC D72]─   ← Scrap Count

Network 6: คำนวณ Yield Rate
     M8000
  ───┤ ├────[ADD D70 D71 D73]─   ← D73 = OK + Rework
     ───────[ADD D73 D72 D74]─   ← D74 = Total Parts

  Yield = D70 × 100 / D74 (OK %)
```

---

## Step 149: Cascade Control {#step-149}

### Cascade Control คืออะไร?

ระบบที่มี Master Controller ควบคุม Setpoint ของ Slave Controller

```
ตัวอย่าง: ควบคุมอุณหภูมิโดยใช้ Flow เป็นตัวกลาง

  Master: Temperature Controller
  → Output: Flow Setpoint

  Slave: Flow Controller
  → Input: Flow Setpoint จาก Master
  → Output: Valve Position

Master Loop: Temp Error → Calculate Required Flow
Slave Loop:  Flow Error → Adjust Valve
```

```
Network 1: Master Loop - Temperature Error
     M8000
  ───┤ ├────[SUB D11 D10 D300]─   ← Temp Error = SP - Actual

Network 2: Calculate Flow Setpoint จาก Temp Error
// ง่ายๆ: Flow_SP = Error × Gain + Bias
     M8000
  ───┤ ├────[MUL D300 K5 D302]─   ← Error × Gain (5)
  ───┤ ├────[ADD D302 K500 D305]─  ← + Bias 50 L/min
  ───┤ ├────[CMP D305 K0]─         ← ตรวจสอบไม่ให้ negative
       ─ [<= D305 K0]──[MOV K0 D305]─   ← Min = 0
       ─ [>= D305 K1000]──[MOV K1000 D305]─ ← Max = 100 L/min

Network 3: Slave Loop - Flow Error
     M8000
  ───┤ ├────[SUB D305 D31 D310]─  ← Flow Error = SP - Actual

Network 4: Valve Control จาก Flow Error
  (ใช้ Comparison เปิด/ปิด Valve based on Flow Error)
     [> D310 K50]          M130.0
  ─────┤ ├─────────────────( S )─── ← Open Valve more

     [< D310 K-50]         M130.1
  ─────┤ ├─────────────────( S )─── ← Close Valve more
```

---

## Step 150: Workshop - Complete Process Monitor {#step-150}

### โจทย์: ระบบ Monitoring ไลน์ผลิต

**ความต้องการ:**
1. ตรวจสอบ Temperature 3 Zone (อยู่ใน range?)
2. ตรวจสอบ Speed (ไม่เกิน max, ไม่ต่ำกว่า min)
3. ตรวจสอบ Product Count ต่อ Batch
4. ตรวจสอบ Quality Check (dimension)
5. แสดงผลสถานะรวม (GREEN/YELLOW/RED)

### I/O List

```
┌──────────┬───────────────────────────────────────────────────┐
│ D-Reg    │ Description                                       │
├──────────┼───────────────────────────────────────────────────┤
│ D10-D12  │ Zone 1,2,3 Temperature (°C × 10)                 │
│ D20      │ Conveyor Speed (m/min × 10)                       │
│ D30      │ Part Count (ชิ้น)                                  │
│ D40      │ Part Dimension (mm × 10)                          │
│ D50      │ Batch Target (ชิ้น, จาก HMI)                       │
├──────────┼───────────────────────────────────────────────────┤
│ M150.0   │ Zone 1 Temp OK                                    │
│ M150.1   │ Zone 2 Temp OK                                    │
│ M150.2   │ Zone 3 Temp OK                                    │
│ M150.3   │ Speed OK                                          │
│ M150.4   │ Batch Complete                                    │
│ M150.5   │ Quality OK                                        │
│ M151.0   │ System Green (All OK)                             │
│ M151.1   │ System Yellow (Warning)                           │
│ M151.2   │ System Red (Fault/Stop)                           │
└──────────┴───────────────────────────────────────────────────┘
```

### โปรแกรมสมบูรณ์

```
//============================================================
// Production Line Monitoring System
// Using Comparison Instructions for all checks
//============================================================


// --- Zone Temperature Checks ---

Network 1: Zone 1 Temperature Check (140-160°C)
     [>= D10 K1400]    [<= D10 K1600]     M150.0
  ──────┤ ├──────────────┤ ├───────────────( )───
  Z1 ≥ 140°C         Z1 ≤ 160°C       Zone1_OK

Network 2: Zone 2 Temperature Check (190-210°C)
     [>= D11 K1900]    [<= D11 K2100]     M150.1
  ──────┤ ├──────────────┤ ├───────────────( )───
  Z2 ≥ 190°C         Z2 ≤ 210°C       Zone2_OK

Network 3: Zone 3 Temperature Check (70-90°C)
     [>= D12 K700]     [<= D12 K900]      M150.2
  ──────┤ ├──────────────┤ ├───────────────( )───
  Z3 ≥ 70°C          Z3 ≤ 90°C        Zone3_OK


// --- Speed Check ---

Network 4: Speed Check (80-120 m/min ±20%)
     [>= D20 K800]     [<= D20 K1200]     M150.3
  ──────┤ ├──────────────┤ ├───────────────( )───
  Speed ≥ 80         Speed ≤ 120       Speed_OK


// --- Batch Complete Check ---

Network 5: Batch Complete
     [>= D30 D50]          M150.4
  ─────┤ ├─────────────────( )───           ← Count ≥ Target


// --- Quality Check ---

Network 6: Quality Check (Dimension ±10 = ±1.0mm around 100.0mm)
     [>= D40 K990]     [<= D40 K1010]     M150.5
  ──────┤ ├──────────────┤ ├───────────────( )───
  Dim ≥ 99.0mm       Dim ≤ 101.0mm     Quality_OK


// --- System Status Calculation ---

Network 7: GREEN - All conditions OK
     M150.0         M150.1         M150.2         M150.3         M151.0
  ───┤ ├─────────────┤ ├─────────────┤ ├─────────────┤ ├─────────────( )───
  Zone1_OK       Zone2_OK       Zone3_OK       Speed_OK


Network 8: RED - Any Critical Fault

  // Temperature way out of range (> ±20°C from setpoint)
     [> D10 K1700]          M151.2
  ─────┤ ├────────────────────( S )───   ← Zone1 > 170°C → RED

     [< D10 K1300]          M151.2
  ─────┤ ├────────────────────( S )───   ← Zone1 < 130°C → RED

     // (Repeat for Zone 2 and 3)

  // Overspeed
     [> D20 K1500]          M151.2
  ─────┤ ├────────────────────( S )───   ← Speed > 150 → RED


Network 9: YELLOW - Warning (not all OK, not RED)
     M151.0         M151.2          M151.1
  ───┤/├─────────────┤/├──────────────( )───
  Not_Green      Not_Red          YELLOW


// --- Output Indicators ---

Network 10: Green Light (All OK)
     M151.0             Q8.0
  ───┤ ├────────────────( )───

Network 11: Yellow Light (Warning, กะพริบ)
     M151.1           M8013           Q8.1
  ───┤ ├──────────────┤ ├──────────────( )───
  YELLOW          1sec Clock

Network 12: Red Light (Fault)
     M151.2             Q8.2
  ───┤ ├────────────────( )───

Network 13: Alarm Horn (เมื่อ RED)
     M151.2           M8012           Q8.3
  ───┤ ├──────────────┤ ├──────────────( )───
  RED             100ms Clock     Alarm Horn
```

### Lab Projects

**Lab 1:** เพิ่ม Comparison สำหรับ Energy Monitoring - คำนวณ Energy Consumption rate และแจ้งเตือนถ้าเกินค่าที่กำหนด

**Lab 2:** สร้างระบบ SPC (Statistical Process Control) อย่างง่าย - เก็บ 10 ค่าล่าสุด คำนวณ Average และ ตรวจว่าค่าอยู่ใน ±3 Sigma

**Lab 3:** เพิ่ม Comparison สำหรับ Maintenance Schedule - นับ Running Hours และแจ้งเตือนเมื่อถึงเวลา Preventive Maintenance

**Lab 4:** ทดสอบโปรแกรมใน Simulator โดยจำลองสถานการณ์:
- อุณหภูมิ Zone 2 ค่อยๆ ดริฟท์จาก 200°C ไป 215°C (เกิน Warning แต่ไม่ถึง Red)
- Speed เพิ่มขึ้นเกิน 150 m/min (Red condition)
- ชิ้นงานวัดได้ 99.5mm (อยู่ใน Tolerance)

---

## สรุป Part 08

```
┌──────────────────────────────────────────────────────────────┐
│ สิ่งที่เรียนรู้ใน Part 08:                                    │
│                                                              │
│ ✓ EQ, NEQ, GT, GTE, LT, LTE - ทุก Comparison Type          │
│ ✓ LIM - Limit Test สำหรับ Range Check                        │
│ ✓ Comparison Chaining - AND/OR Conditions                    │
│ ✓ Hysteresis/Deadband - ป้องกัน Chattering                   │
│ ✓ Level Control with ON/OFF bands                            │
│ ✓ Speed Monitoring - Overspeed/Underspeed                    │
│ ✓ Pressure Control with Deadband                             │
│ ✓ Temperature Control for Furnace/Oven                       │
│ ✓ Quality Control with Tolerance Bands                       │
│ ✓ Multi-Zone Control                                         │
│ ✓ Complete Production Line Monitor                           │
└──────────────────────────────────────────────────────────────┘
```

---

**[← Part 07: Math Operations](part07_math_operations.md)** | **[Part 09: Data Move Instructions →](part09_data_move.md)**
