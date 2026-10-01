# Part 10: โปรแกรม PLC แรกของคุณ
## Steps 171-190: คู่มือสมบูรณ์ตั้งแต่เริ่มต้นถึงเดินระบบ

---

## สารบัญ

- [Step 171: ภาพรวมโปรเจกต์ - ระบบสายพาน Conveyor](#step-171)
- [Step 172: วิเคราะห์ความต้องการ](#step-172)
- [Step 173: สร้าง I/O List](#step-173)
- [Step 174: Tag Naming Convention](#step-174)
- [Step 175: วางแผนโครงสร้างโปรแกรม](#step-175)
- [Step 176: สร้าง Hardware Configuration](#step-176)
- [Step 177: เขียน Safety Logic ก่อน](#step-177)
- [Step 178: เขียน Mode Selection Logic](#step-178)
- [Step 179: เขียน Conveyor Control Logic](#step-179)
- [Step 180: เขียน Sensor และ Detection Logic](#step-180)
- [Step 181: เขียน Speed Control Logic](#step-181)
- [Step 182: เขียน Alarm และ Fault Logic](#step-182)
- [Step 183: เขียน HMI Interface Logic](#step-183)
- [Step 184: เขียน Data Logging Logic](#step-184)
- [Step 185: Program Documentation](#step-185)
- [Step 186: Simulation Testing](#step-186)
- [Step 187: Factory Acceptance Test (FAT)](#step-187)
- [Step 188: Commissioning Steps](#step-188)
- [Step 189: Troubleshooting Guide](#step-189)
- [Step 190: Workshop - Complete Commissioning Checklist](#step-190)

---

## Step 171: ภาพรวมโปรเจกต์ {#step-171}

### ระบบที่จะสร้าง: Conveyor Sorting System

```
┌─────────────────────────────────────────────────────────────────────┐
│              CONVEYOR SORTING SYSTEM                                │
│                                                                     │
│  [FEED]──[C1]──[SENSOR]──[C2]──┬──[CHUTE_A]  (Product A)          │
│                                 │                                   │
│                                 └──[C3]──[CHUTE_B]  (Product B)    │
│                                                                     │
│  Legend:                                                            │
│    C1 = Feed Conveyor (รับสินค้า)                                   │
│    C2 = Main Conveyor (ขนส่ง)                                       │
│    C3 = Divert Conveyor (เบี่ยงสินค้า)                              │
│    SENSOR = Vision/Barcode Scanner (แยกประเภท)                      │
│    CHUTE_A/B = ช่องคัดแยก                                           │
└─────────────────────────────────────────────────────────────────────┘

ความสามารถ:
1. ทำงานแบบ Auto/Manual
2. แยกสินค้า 2 ประเภท (A และ B)
3. นับจำนวนสินค้าแต่ละประเภท
4. Emergency Stop ทุก Conveyor
5. Speed Control (ปรับความเร็วได้)
6. Fault Alarm (Motor OL, Jammed)
7. Production Report
```

---

## Step 172: วิเคราะห์ความต้องการ {#step-172}

### Functional Requirements

```
FR-01: โหมดการทำงาน
  - Auto Mode: ระบบทำงานอัตโนมัติ ตามสัญญาณ Sensor
  - Manual Mode: Operator ควบคุมแต่ละ Conveyor แยกกัน

FR-02: Start/Stop Sequence
  Auto Start: C3 → C2 → C1 (เริ่มจากปลายทาง)
  Auto Stop:  C1 → C2 → C3 (หยุดจากต้นทาง รอของผ่าน)
  E-Stop:     หยุดทันทีทุกตัว

FR-03: Product Sorting
  - Scanner อ่านบาร์โค้ด
  - Product A → ตรงไป CHUTE_A (C2 เดิน, C3 หยุด)
  - Product B → เบี่ยง CHUTE_B (C2 เดิน, C3 เดิน)

FR-04: Counter
  - นับ Product A แยกจาก Product B
  - นับ Total ทั้งหมด
  - Reset ได้

FR-05: Speed Control
  - ปรับ Speed ของ C1, C2, C3 ได้ (0-100%)
  - ผ่าน Analog Output หรือ VFD Drive

FR-06: Protection
  - Motor Overload Relay
  - Jam Detection (ชิ้นงานค้าง)
  - E-Stop
```

### Non-Functional Requirements

```
NFR-01: Reliability
  - ทำงานได้ 24/7
  - MTTR < 30 นาที

NFR-02: Safety
  - E-Stop Response < 100ms
  - Motor Interlock ป้องกัน Simultaneous Start
  - ทุก Output ต้องผ่าน E-Stop Interlock

NFR-03: Maintainability
  - ทุก Tag ต้องมีชื่อที่เข้าใจง่าย
  - Comment ทุก Network
  - Fault Code แสดงบน HMI

NFR-04: Operator Interface
  - HMI แสดงสถานะทุก Conveyor
  - มี Fault Beacon (กะพริบเมื่อมี Fault)
```

---

## Step 173: สร้าง I/O List {#step-173}

### Digital Inputs

```
┌──────────┬─────────────────────────────┬──────────────────────┐
│ Tag Name │ Description (TH)            │ Address              │
│          │                             │ Siemens/AB/Mitsu     │
├──────────┼─────────────────────────────┼──────────────────────┤
│ C1_START │ C1 Start Button (Local)     │ I0.0 / I:0/0 / X0   │
│ C1_STOP  │ C1 Stop Button (Local)      │ I0.1 / I:0/1 / X1   │
│ C2_START │ C2 Start Button (Local)     │ I0.2 / I:0/2 / X2   │
│ C2_STOP  │ C2 Stop Button (Local)      │ I0.3 / I:0/3 / X3   │
│ C3_START │ C3 Start Button (Local)     │ I0.4 / I:0/4 / X4   │
│ C3_STOP  │ C3 Stop Button (Local)      │ I0.5 / I:0/5 / X5   │
│ E_STOP   │ Emergency Stop (NC, ต่อบวก)│ I0.6 / I:0/6 / X6   │
│ MODE_SW  │ Auto/Manual Switch (1=Auto) │ I0.7 / I:0/7 / X7   │
├──────────┼─────────────────────────────┼──────────────────────┤
│ C1_OL    │ C1 Motor Overload (NC)      │ I1.0 / I:1/0 / X10  │
│ C2_OL    │ C2 Motor Overload (NC)      │ I1.1 / I:1/1 / X11  │
│ C3_OL    │ C3 Motor Overload (NC)      │ I1.2 / I:1/2 / X12  │
│ C1_SPEED │ C1 Speed FB (VFD OK signal) │ I1.3 / I:1/3 / X13  │
│ C2_SPEED │ C2 Speed FB                 │ I1.4 / I:1/4 / X14  │
│ C3_SPEED │ C3 Speed FB                 │ I1.5 / I:1/5 / X15  │
├──────────┼─────────────────────────────┼──────────────────────┤
│ SENSOR_A │ Product A Detected          │ I2.0 / I:2/0 / X20  │
│ SENSOR_B │ Product B Detected          │ I2.1 / I:2/1 / X21  │
│ C1_JAM   │ C1 Jam Sensor (Proximity)   │ I2.2 / I:2/2 / X22  │
│ C2_JAM   │ C2 Jam Sensor               │ I2.3 / I:2/3 / X23  │
│ C3_JAM   │ C3 Jam Sensor               │ I2.4 / I:2/4 / X24  │
│ RESET_PB │ Fault Reset Push Button     │ I2.5 / I:2/5 / X25  │
│ SYS_START│ System Start (Auto)         │ I2.6 / I:2/6 / X26  │
│ SYS_STOP │ System Stop (Auto)          │ I2.7 / I:2/7 / X27  │
└──────────┴─────────────────────────────┴──────────────────────┘
```

### Digital Outputs

```
┌──────────┬─────────────────────────────┬──────────────────────┐
│ Tag Name │ Description (TH)            │ Address              │
├──────────┼─────────────────────────────┼──────────────────────┤
│ C1_RUN   │ C1 Motor Contactor          │ Q0.0 / O:0/0 / Y0   │
│ C2_RUN   │ C2 Motor Contactor          │ Q0.1 / O:0/1 / Y1   │
│ C3_RUN   │ C3 Motor Contactor          │ Q0.2 / O:0/2 / Y2   │
│ DIVERT   │ Divert Gate Actuator        │ Q0.3 / O:0/3 / Y3   │
│ SIREN    │ Alarm Siren (Horn)          │ Q0.4 / O:0/4 / Y4   │
│ BEACON_G │ Green Beacon (Running OK)   │ Q0.5 / O:0/5 / Y5   │
│ BEACON_A │ Amber Beacon (Warning)      │ Q0.6 / O:0/6 / Y6   │
│ BEACON_R │ Red Beacon (Fault/E-Stop)   │ Q0.7 / O:0/7 / Y7   │
├──────────┼─────────────────────────────┼──────────────────────┤
│ C1_IND   │ C1 Running Indicator Lamp   │ Q1.0 / O:1/0 / Y10  │
│ C2_IND   │ C2 Running Indicator Lamp   │ Q1.1 / O:1/1 / Y11  │
│ C3_IND   │ C3 Running Indicator Lamp   │ Q1.2 / O:1/2 / Y12  │
│ AUTO_IND │ Auto Mode Indicator         │ Q1.3 / O:1/3 / Y13  │
│ FAULT_IND│ Fault Indicator Lamp        │ Q1.4 / O:1/4 / Y14  │
│ CHUTE_A  │ Chute A Gate Open           │ Q1.5 / O:1/5 / Y15  │
│ CHUTE_B  │ Chute B Gate Open           │ Q1.6 / O:1/6 / Y16  │
└──────────┴─────────────────────────────┴──────────────────────┘
```

### Analog Outputs (VFD Speed Control)

```
┌──────────┬─────────────────────────────┬──────────────────────┐
│ Tag Name │ Description (TH)            │ Address              │
├──────────┼─────────────────────────────┼──────────────────────┤
│ C1_SPEED │ C1 Speed Command (0-100%)   │ AQW0 / O:3/0 / D600 │
│ C2_SPEED │ C2 Speed Command            │ AQW2 / O:3/1 / D601 │
│ C3_SPEED │ C3 Speed Command            │ AQW4 / O:3/2 / D602 │
└──────────┴─────────────────────────────┴──────────────────────┘

หมายเหตุ: 0-27648 (Siemens) หรือ 0-4095 → 0-100% VFD Speed
```

### Internal Memory Bits (M Bits)

```
┌──────────────┬──────────────────────────────────────────────────┐
│ M Bit        │ Description                                      │
├──────────────┼──────────────────────────────────────────────────┤
│ M0.0         │ System Auto Mode Flag                            │
│ M0.1         │ System Running Flag                              │
│ M0.2         │ System Stop Requested                            │
│ M0.3         │ E-Stop Active                                    │
├──────────────┼──────────────────────────────────────────────────┤
│ M1.0         │ C1 Running Flag                                  │
│ M1.1         │ C2 Running Flag                                  │
│ M1.2         │ C3 Running Flag                                  │
├──────────────┼──────────────────────────────────────────────────┤
│ M2.0         │ C1 Fault (OL หรือ Jam)                          │
│ M2.1         │ C2 Fault                                         │
│ M2.2         │ C3 Fault                                         │
│ M2.7         │ Any Fault Flag                                   │
├──────────────┼──────────────────────────────────────────────────┤
│ M3.0         │ Product A in Transit (อยู่บน Conveyor)           │
│ M3.1         │ Product B in Transit                             │
│ M3.2         │ Sort Request Active                              │
│ M3.3         │ Divert Gate Open Request                         │
├──────────────┼──────────────────────────────────────────────────┤
│ M10.0        │ Auto Start Sequence Active                       │
│ M10.1        │ Auto Stop Sequence Active                        │
│ M10.2-M10.9  │ Sequence Steps                                   │
└──────────────┴──────────────────────────────────────────────────┘
```

### Data Registers (D Registers - Mitsubishi) / MW (Siemens)

```
┌──────────┬──────────────────────────────────────────────────────┐
│ Register │ Description                                          │
├──────────┼──────────────────────────────────────────────────────┤
│ D0       │ C1 Speed Setpoint (0-100%)                          │
│ D1       │ C2 Speed Setpoint                                    │
│ D2       │ C3 Speed Setpoint                                    │
│ D3       │ Divert Gate Delay (ms)                               │
│          │                                                      │
│ D10      │ Count Product A                                      │
│ D11      │ Count Product B                                      │
│ D12      │ Count Total                                          │
│ D13      │ Count Reject (Jam กรณีเสีย)                         │
│          │                                                      │
│ D20      │ Alarm Code (รหัส Fault ล่าสุด)                      │
│ D21      │ Fault Count (จำนวนครั้งที่ Fault)                    │
│          │                                                      │
│ D50      │ Shift Counter (นับ Shift)                            │
│ D51      │ Shift Start Count                                    │
│ D52      │ Shift Target (ชิ้น/Shift)                            │
└──────────┴──────────────────────────────────────────────────────┘
```

---

## Step 174: Tag Naming Convention {#step-174}

### กฎการตั้งชื่อ Tags

```
Format: [Equipment][Function][Type]

Equipment: C1=Conveyor1, C2=Conveyor2, SYS=System, ALM=Alarm
Function:  RUN=Running, STOP=Stop, START=Start button, FAULT=fault
Type:      FB=Feedback, CMD=Command, SP=Setpoint, ACT=Actual

ตัวอย่าง:
  C1_RUN_CMD  = Conveyor 1 Run Command (Output)
  C1_RUN_FB   = Conveyor 1 Running Feedback (Input from VFD)
  C1_SPEED_SP = Conveyor 1 Speed Setpoint
  SYS_FAULT   = System Fault Flag
  ALM_C1_OL   = Alarm: Conveyor 1 Overload
```

### ตัวอย่าง Tags ที่ดีและไม่ดี

```
ไม่ดี:                    ดี:
  B1                   →  C1_RUN_CMD
  Sensor               →  C2_JAM_PROX
  Alarm                →  ALM_C1_OVERLOAD
  Flag                 →  SYS_AUTO_MODE
  Output1              →  C1_MOTOR_CONT
  D100                 →  PROD_COUNT_A
```

---

## Step 175: วางแผนโครงสร้างโปรแกรม {#step-175}

### Program Structure (OB/Function Blocks)

```
Main Program (OB1 / Main Routine):
  Network 1-10:   Safety Logic (E-Stop, OL, Jam)
  Network 11-20:  Mode Selection (Auto/Manual)
  Network 21-30:  Auto Sequence Control
  Network 31-40:  Manual Control
  Network 41-50:  Sorting Logic
  Network 51-60:  Speed Control
  Network 61-70:  Counter Logic
  Network 71-80:  Alarm Management
  Network 81-90:  Output Control
  Network 91-100: HMI Interface
  Network 101-110: Data Logging
```

### Execution Order (สำคัญมาก!)

```
1. อ่าน Inputs (PLC ทำเอง)
   ↓
2. Safety Check → ถ้า Fault → Force Output Off
   ↓
3. Mode Determination → Auto or Manual?
   ↓
4. Process Logic (Sequencing, Sorting)
   ↓
5. Alarm Check → Update Alarm Bits
   ↓
6. Output Assignment → Write to Physical Output
   ↓
7. Write Outputs (PLC ทำเอง)
```

---

## Step 176: สร้าง Hardware Configuration {#step-176}

### Hardware Setup

```
PLC Selection: Mitsubishi FX3U-32MT/ES (ตัวอย่าง)

CPU: FX3U-32MT/ES
  - 16 Digital Inputs (X0-X17)
  - 16 Digital Outputs (Y0-Y17, Transistor)

Expansion Module: FX3U-4DA (4 Analog Outputs)
  - AO0-AO3 สำหรับ VFD Speed Control (0-10V หรือ 4-20mA)

HMI: GOT2000 Series (ต่อ RS-485)

VFD: Mitsubishi FR-A700 Series (3 ตัว สำหรับ C1,C2,C3)

Wiring Notes:
  - E-Stop: NC Contact, wired in series with 24V
  - OL Relays: NC Contact, wired in series with each input
  - Start Buttons: NO Contact
  - Stop Buttons: NC Contact (safety preferred)
```

### I/O Address Assignment (Mitsubishi FX3U)

```
Digital Inputs:
  X0  = C1_START    X1  = C1_STOP
  X2  = C2_START    X3  = C2_STOP
  X4  = C3_START    X5  = C3_STOP
  X6  = E_STOP (NC) X7  = MODE_SWITCH

  X10 = C1_OL (NC)  X11 = C2_OL (NC)
  X12 = C3_OL (NC)  X13 = C1_FB
  X14 = C2_FB       X15 = C3_FB

  X16 = JAM_DETECT  X17 = PRODUCT_SENSOR

  (ต้องขยาย) X20 = SENSOR_A   X21 = SENSOR_B
  X22 = C1_JAM      X23 = C2_JAM
  X24 = C3_JAM      X25 = RESET_PB
  X26 = SYS_START   X27 = SYS_STOP

Digital Outputs:
  Y0  = C1_RUN      Y1  = C2_RUN
  Y2  = C3_RUN      Y3  = DIVERT_GATE
  Y4  = ALARM_SIREN Y5  = BEACON_GREEN
  Y6  = BEACON_AMBER Y7 = BEACON_RED
  Y10 = C1_IND      Y11 = C2_IND
  Y12 = C3_IND      Y13 = AUTO_IND
  Y14 = FAULT_IND   Y15 = CHUTE_A_GATE
  Y16 = CHUTE_B_GATE
```

---

## Step 177: เขียน Safety Logic ก่อน {#step-177}

### Safety Rule: Safety Logic ต้องอยู่ก่อนเสมอ!

```
//============================================================
// SAFETY LOGIC - Network 1-10
// ต้องเขียนก่อนทุกอย่าง!
// กฎ: Output ทุกตัวต้องผ่าน Safety Check
//============================================================


Network 1: Emergency Stop Detection
// E-Stop = NC Contact → เมื่อกด = Input จะ OFF (0)

     X6                 M0.3
  ───┤/├─────────────────( )───     ← M0.3 = E-Stop Active (เมื่อ X6 = 0)
  E_STOP (NC)         E_STOP_FLAG

  ← หมายเหตุ: X6 ต่อเป็น NC (Normally Closed)
    ปกติ: X6 = ON (วงจรปิด) → M0.3 = 0 (ไม่มี E-Stop)
    กดปุ่ม E-Stop: X6 = OFF → M0.3 = 1 (E-Stop Active)


Network 2: Conveyor 1 Motor Overload

     X10                M2.0
  ───┤/├─────────────────( S )───   ← Set C1 Fault เมื่อ OL ทำงาน
  C1_OL (NC)          C1_FAULT

  หมายเหตุ: X10 = NC Contact ของ OL Relay
    ปกติ: X10 = ON → M2.0 ไม่ Set
    เมื่อ OL Trip: X10 = OFF → M2.0 Set (Latched Fault)


Network 3: Conveyor 2 Motor Overload

     X11                M2.1
  ───┤/├─────────────────( S )───
  C2_OL (NC)          C2_FAULT


Network 4: Conveyor 3 Motor Overload

     X12                M2.2
  ───┤/├─────────────────( S )───
  C3_OL (NC)          C3_FAULT


Network 5: Jam Detection (ตรวจของค้าง)

     X22              T0_Done         M2.0
  ───┤ ├──────────────[TON T0 5s]─────( S )───  ← C1 Jam = อยู่นาน 5 วินาที
  C1_JAM_SENSOR

     X23              T1_Done         M2.1
  ───┤ ├──────────────[TON T1 5s]─────( S )───
  C2_JAM_SENSOR

     X24              T2_Done         M2.2
  ───┤ ├──────────────[TON T2 5s]─────( S )───
  C3_JAM_SENSOR


Network 6: Alarm Code Assignment
     M2.0              [MOV K11 D20]─    ← Code 11 = C1 OL Fault
  ───┤ ├────────────────

     M2.1              [MOV K21 D20]─    ← Code 21 = C2 OL Fault
  ───┤ ├────────────────

     M2.2              [MOV K31 D20]─    ← Code 31 = C3 OL Fault
  ───┤ ├────────────────

     M0.3              [MOV K99 D20]─    ← Code 99 = E-Stop
  ───┤ ├────────────────


Network 7: Any Fault Flag (OR ทุก Fault)

     M2.0               M2.7
  ───┤ ├────────────────( )───

     M2.1               M2.7
  ───┤ ├────────────────( )───

     M2.2               M2.7
  ───┤ ├────────────────( )───

     M0.3               M2.7
  ───┤ ├────────────────( )───     ← E-Stop ก็เป็น Fault หนึ่ง


Network 8: Fault Reset (ต้องล้าง Fault ก่อน Restart)

     X25              M2.7            M2.0
  ───┤ ├──────────────┤/├──────────────( R )───  ← Reset C1 Fault
  RESET_PB        Any_Fault                      (เฉพาะเมื่อ Fault หายแล้ว)

     X25              M2.7            M2.1
  ───┤ ├──────────────┤/├──────────────( R )───

     X25              M2.7            M2.2
  ───┤ ├──────────────┤/├──────────────( R )───

     X25              M0.3            M0.3
  ───┤ ├──────────────┤/├──────────────( R )───  ← Reset E-Stop
  RESET_PB       E_Stop_NOT_Active
                 (X6 ต้องดึง E-Stop ออกก่อน)


Network 9: Stop All on Any Fault

     M2.7               M1.0
  ───┤ ├────────────────( R )───    ← Stop C1
     M2.7               M1.1
  ───┤ ├────────────────( R )───    ← Stop C2
     M2.7               M1.2
  ───┤ ├────────────────( R )───    ← Stop C3
     M2.7               M0.1
  ───┤ ├────────────────( R )───    ← System Stop
  Any_Fault


Network 10: Fault Counter
     M2.7           M250.0          D21
  ───┤ ├─────────────┤P├──────────[INC D21]─   ← นับจำนวน Fault ทั้งหมด
  Any_Fault       Prev_Fault
```

---

## Step 178: เขียน Mode Selection Logic {#step-178}

```
//============================================================
// MODE SELECTION - Network 11-20
//============================================================


Network 11: Auto Mode Flag

     X7                 M0.0
  ───┤ ├─────────────────( )───     ← M0.0 = Auto Mode (follows switch)
  MODE_SWITCH (1=Auto)


Network 12: Auto Mode Indicator

     M0.0               Y13
  ───┤ ├─────────────────( )───     ← Auto Mode Lamp ON


Network 13: System Start (Auto Mode)

     X26              M0.0           M2.7           M0.1
  ───┤ ├──────────────┤ ├────────────┤/├─────────────( S )───
  SYS_START       Auto_Mode       Any_Fault       SYS_RUNNING


Network 14: System Stop (Auto Mode)

     X27              M0.1
  ───┤ ├──────────────────( R )───
  SYS_STOP


Network 15: หยุดระบบเมื่อสลับเป็น Manual Mode

     M0.0              M0.1
  ───┤/├────────────────( R )───    ← หยุดทุกอย่างเมื่อเปลี่ยนเป็น Manual
  Not_Auto          System_Stop
```

---

## Step 179: เขียน Conveyor Control Logic {#step-179}

```
//============================================================
// CONVEYOR CONTROL - Network 21-40
//============================================================


// --- AUTO SEQUENCE START ---

Network 21: Auto Start Sequence (C3 ก่อน)

     M0.1           M1.2           M2.2           M1.2
  ───┤ ├────────────┤/├─────────────┤/├─────────────( S )───
  SYS_Running    C3_Running     C3_Fault        C3_Start


Network 22: C3 ต้องเดินก่อน C2 จึงจะเริ่มได้

     M0.1           X15            M2.1            M1.1
  ───┤ ├────────────┤ ├─────────────┤/├─────────────( S )───
  SYS_Running   C3_SpeedFB      C2_Fault        C2_Start


Network 23: C2 เดินแล้ว C1 จึงเริ่มได้

     M0.1           X14            M2.0            M1.0
  ───┤ ├────────────┤ ├─────────────┤/├─────────────( S )───
  SYS_Running   C2_SpeedFB      C1_Fault        C1_Start


// --- AUTO STOP SEQUENCE ---

Network 24: Auto Stop Sequence (C1 ก่อน)

     M0.1              M1.0
  ───┤/├────────────────( R )───   ← Stop C1 เมื่อ System Stop
  Not_Running


Network 25: Stop C2 หลัง C1 หยุด

     M0.1           M1.0            M1.1
  ───┤/├─────────────┤/├─────────────( R )───
  Not_Running    C1_Stopped       Stop_C2


Network 26: Stop C3 หลัง C2 หยุดและ Conveyor ว่าง

     M0.1           M1.1            T5(Done)        M1.2
  ───┤/├─────────────┤/├─────────────┤ ├─────────────( R )───
  Not_Running    C2_Stopped      5sec_Wait        Stop_C3

Network 27: Wait Timer (รอให้ Conveyor ว่าง)
     M0.1           M1.1
  ───┤/├─────────────┤/├──────────[TON T5 5s]─     ← รอ 5 วินาที


// --- MANUAL MODE CONTROL ---

Network 28: Manual C1 Control

     M0.0             X0            X1              M2.0            M1.0
  ───┤/├─────────┬─────┤ ├────────────┤/├─────────────┤/├─────────────( )───
  Not_Auto      │   C1_START      C1_STOP          C1_Fault
                │
     M1.0       │
  ───┤ ├─────────┘    ← Seal


Network 29: Manual C2 Control

     M0.0             X2            X3              M2.1            M1.1
  ───┤/├─────────┬─────┤ ├────────────┤/├─────────────┤/├─────────────( )───
  Not_Auto      │   C2_START      C2_STOP          C2_Fault
                │
     M1.1       │
  ───┤ ├─────────┘


Network 30: Manual C3 Control

     M0.0             X4            X5              M2.2            M1.2
  ───┤/├─────────┬─────┤ ├────────────┤/├─────────────┤/├─────────────( )───
  Not_Auto      │   C3_START      C3_STOP          C3_Fault
                │
     M1.2       │
  ───┤ ├─────────┘
```

---

## Step 180: เขียน Sensor และ Detection Logic {#step-180}

```
//============================================================
// SORTING LOGIC - Network 41-50
//============================================================


Network 41: Detect Product Type

     X20              M3.0
  ───┤ ├──────────────( S )───     ← Product A Detected
  SENSOR_A

     X21              M3.1
  ───┤ ├──────────────( S )───     ← Product B Detected
  SENSOR_B


Network 42: Divert Gate Control (Product B → Divert)

     M3.1           T10(Done)      M3.3
  ───┤ ├─────────────┤ ├────────────( S )───   ← Divert after position delay
  Prod_B_Flag    Position_Delay  Divert_Request

Network 43: Position Delay Timer (ชิ้นงานเคลื่อนไปถึง Divert Point)

     M3.1
  ───┤ ├──────────────[TON T10 2s]─  ← 2 วินาที = เวลาที่ชิ้นงานเดินจาก Sensor ถึง Gate


Network 44: Close Divert after Product Passes

     M3.3
  ───┤ ├──────────────[TON T11 1s]─  ← รอ 1 วินาทีหลัง Gate เปิด

     T11(Done)          M3.3
  ───┤ ├────────────────( R )───     ← ปิด Divert

     T11(Done)          M3.1
  ───┤ ├────────────────( R )───     ← Clear Product B flag


Network 45: Counting Products

     X20           M3.0              D10
  ───┤ ├──────────────┤P├──────────[INC D10]─   ← Count Product A
  SENSOR_A        Pulse

     X21           M3.1              D11
  ───┤ ├──────────────┤P├──────────[INC D11]─   ← Count Product B
  SENSOR_B        Pulse

     D12
  ──[ADD D10 D11 D12]─────────────────────────  ← Total Count


Network 46: Clear Product A Flag หลัง Pass Through

     M3.0
  ───┤ ├──────────────[TON T12 3s]─  ← รอ 3 วินาที (A ผ่าน Chute A แล้ว)

     T12(Done)          M3.0
  ───┤ ├────────────────( R )───     ← Clear Product A flag
```

---

## Step 181: เขียน Speed Control Logic {#step-181}

```
//============================================================
// SPEED CONTROL - Network 51-60
//============================================================


Network 51: Default Speed Settings (First Scan)

     M8002
  ───┤ ├────[MOV K70 D0]─            ← C1 Speed = 70% (default)
            [MOV K80 D1]─            ← C2 Speed = 80%
            [MOV K60 D2]─            ← C3 Speed = 60%


Network 52: Limit Speed Range (0-100%)

     M8000
  ───┤ ├────[CMP D0 K0]─   [< D0 K0][MOV K0 D0]─      ← Min 0%
            [CMP D0 K100]─ [> D0 K100][MOV K100 D0]─   ← Max 100%
            [CMP D1 K0]─   [< D1 K0][MOV K0 D1]─
            [CMP D1 K100]─ [> D1 K100][MOV K100 D1]─
            [CMP D2 K0]─   [< D2 K0][MOV K0 D2]─
            [CMP D2 K100]─ [> D2 K100][MOV K100 D2]─


Network 53: Scale Speed % เป็น Analog Value (0-4000 สำหรับ FX3U-4DA)

     M8000
  ───┤ ├────[MUL D0 K40 D600]─       ← C1: 0-100% → 0-4000 (4DA range)
            [MUL D1 K40 D601]─       ← C2: 0-100% → 0-4000
            [MUL D2 K40 D602]─       ← C3: 0-100% → 0-4000


Network 54: Output Speed (เฉพาะเมื่อ Running)

     M1.0             Y0             D600
  ───┤ ├──────────────┤ ├──[MOV D600 AO0]─   ← C1 Speed to VFD

     M1.1             Y1             D601
  ───┤ ├──────────────┤ ├──[MOV D601 AO1]─   ← C2 Speed to VFD

     M1.2             Y2             D602
  ───┤ ├──────────────┤ ├──[MOV D602 AO2]─   ← C3 Speed to VFD


Network 55: ลด Speed เมื่อ Manual Mode (Safety)

     M0.0              D0
  ───┤/├──────────────[CMP D0 K50]─[> D0 K50][MOV K50 D0]─ ← Max 50% in Manual
  Not_Auto
```

---

## Step 182: เขียน Alarm และ Fault Logic {#step-182}

```
//============================================================
// ALARM MANAGEMENT - Network 71-80
//============================================================


Network 71: Alarm Horn (กะพริบเมื่อ Fault, Steady เมื่อ E-Stop)

     M2.7           M0.3            M8012            Y4
  ───┤ ├─────────────┤/├─────────────┤ ├──────────────( )───
  Any_Fault     Not_EStop         100ms_Clk       ALARM_SIREN (กะพริบ)

     M0.3                            Y4
  ───┤ ├────────────────────────────( )───          ← Steady เมื่อ E-Stop
  E_STOP


Network 72: Beacon Green (ระบบทำงานปกติ)

     M0.1           M2.7             Y5
  ───┤ ├─────────────┤/├──────────────( )───
  SYS_Running    No_Fault          GREEN_BEACON


Network 73: Beacon Amber (Warning - Running กับ Minor Fault)

     M0.1           M2.7             Y6
  ───┤ ├─────────────┤ ├──────────────( )───
  SYS_Running    Any_Fault         AMBER_BEACON


Network 74: Beacon Red (E-Stop หรือ Major Fault)

     M0.3                            Y7
  ───┤ ├─────────────────────────────( )───
  E_STOP

     M2.7           M0.3             Y7
  ───┤ ├─────────────┤/├──────────────( )───        ← Fault (not E-Stop)
  Any_Fault     Not_EStop


Network 75: Fault Indicator Lamp (กะพริบ)

     M2.7           M8013            Y14
  ───┤ ├──────────────┤ ├─────────────( )───
  Any_Fault       1sec_Clock       FAULT_IND


Network 76: Alarm History Logging

     M2.7           M250.0            D600           D20
  ───┤ ├─────────────┤P├─[MOV D600 Z0]──[MOV D20 D700Z0]─
  Any_Fault      Rising_Edge        [INC D600]─
                                    [CMP D600 K50]─
                                    [>= D600 K50][MOV K0 D600]─
```

---

## Step 183: เขียน HMI Interface Logic {#step-183}

```
//============================================================
// HMI INTERFACE - Network 91-100
//============================================================


Network 91: Copy Status to HMI Area

     M8000
  ───┤ ├────[MOV D10 D900]─          ← Count A → HMI
            [MOV D11 D901]─          ← Count B → HMI
            [MOV D12 D902]─          ← Total → HMI
            [MOV D20 D903]─          ← Alarm Code → HMI
            [MOV D0 D904]─           ← C1 Speed SP → HMI
            [MOV D1 D905]─           ← C2 Speed SP → HMI
            [MOV D2 D906]─           ← C3 Speed SP → HMI


Network 92: Pack Status Bits for HMI (16 bits = 16 Status)

     M8000
  ───┤ ├────[MOV K0 D910]─           ← Clear Status Word

     M0.0              D910
  ───┤ ├────────────[BSET D910 K0]─  ← Bit 0 = Auto Mode
  Auto_Mode

     M0.1              D910
  ───┤ ├────────────[BSET D910 K1]─  ← Bit 1 = System Running
  SYS_Running

     M1.0              D910
  ───┤ ├────────────[BSET D910 K2]─  ← Bit 2 = C1 Running
  C1_Running

     M1.1              D910
  ───┤ ├────────────[BSET D910 K3]─  ← Bit 3 = C2 Running

     M1.2              D910
  ───┤ ├────────────[BSET D910 K4]─  ← Bit 4 = C3 Running

     M2.7              D910
  ───┤ ├────────────[BSET D910 K7]─  ← Bit 7 = Any Fault

     M0.3              D910
  ───┤ ├────────────[BSET D910 K15]─ ← Bit 15 = E-Stop
  E_STOP


Network 93: Read HMI Commands

     D920.0            M0.1           ← HMI System Start Command
  ───┤ ├────────────────( S )───

     D920.1            M0.1
  ───┤ ├────────────────( R )───      ← HMI System Stop Command

     D920.15           M0.3
  ───┤ ├────────────────( R )───      ← HMI E-Stop Reset Request
```

---

## Step 184: เขียน Data Logging Logic {#step-184}

```
//============================================================
// DATA LOGGING - Network 101-110
//============================================================


Network 101: Log Production Count ทุก 1 นาที

     M8014              D950           D12
  ───┤ ├────[MOV D950 Z0]──[MOV D12 D1000Z0]─
  1min_Pulse             [INC D950]─
                         [CMP D950 K480]─      ← 480 = 8 ชั่วโมง log
                         [>= D950 K480][MOV K0 D950]─


Network 102: Record Shift Summary เมื่อสิ้น Shift (8 ชั่วโมง)

     D950              I20.0
  ──[= D950 K0]─────────┤ ├────[BMOV D10 D1500 K5]─  ← Save shift summary
  Wrap Around       Shift_End      to history area
```

---

## Step 185: Program Documentation {#step-185}

### หลักการเขียน Comment ที่ดี

```
หลักการ:
1. ทุก Network ต้องมี Title
2. ทุก Rung ที่ไม่ชัดเจนต้องมี Comment
3. ใช้ภาษาที่ Maintenance Engineer เข้าใจ
4. ระบุ Reference (เช่น เกี่ยวกับ FR ข้อไหน)

ตัวอย่าง Comment ที่ดี:
//-----------------------------------------------------------
// Network 21: Auto Start Conveyor C3 (FR-02)
// C3 ต้องเริ่มทำงานก่อนเพื่อรองรับสินค้าจาก C2
// Condition: SYS_Running AND NOT C3_Running AND NOT C3_Fault
//-----------------------------------------------------------

ตัวอย่าง Comment ที่ไม่ดี:
// Network 21 ← ไม่มีข้อมูลเพิ่ม
// Start C3 ← ไม่บอกว่าทำไม
```

### Version Control ในโปรแกรม

```
เพิ่ม Version Block ที่หัวโปรแกรม:

//============================================================
// Program: CONVEYOR_SORTING_SYSTEM
// Version: 1.2
// Date:    2024-01-15
// Author:  [ชื่อ Engineer]
// PLC:     Mitsubishi FX3U-32MT/ES
//
// Revision History:
// V1.0 - Initial Release
// V1.1 - Added Jam Detection (T0-T2)
// V1.2 - Added HMI Interface (Network 91-100)
//============================================================
```

---

## Step 186: Simulation Testing {#step-186}

### การทดสอบใน Simulator

```
ขั้นตอนการทดสอบ:

1. Upload โปรแกรมไป GX Simulator (Mitsubishi) หรือ PLCSIM (Siemens)

2. Test Case ที่ต้องทดสอบ:
```

### Test Cases

```
┌─────┬──────────────────────────────┬───────────────────────────────────┐
│ TC# │ Test Case                    │ Expected Result                   │
├─────┼──────────────────────────────┼───────────────────────────────────┤
│ TC1 │ กด SYS_START ใน Auto Mode    │ C3→C2→C1 เริ่มทำงานตามลำดับ      │
│ TC2 │ กด E-Stop ขณะทำงาน           │ ทุก Motor หยุดทันที               │
│ TC3 │ C1_OL Trip ขณะทำงาน          │ C1 หยุด, Fault Lamp กะพริบ        │
│ TC4 │ กด RESET หลัง Fault หาย      │ Fault ล้าง, พร้อม Start ใหม่      │
│ TC5 │ Sensor ตรวจเจอ Product A     │ ผ่านตรงไป CHUTE_A                 │
│ TC6 │ Sensor ตรวจเจอ Product B     │ Divert Gate เปิด → CHUTE_B        │
│ TC7 │ Product A Count เพิ่มขึ้น     │ D10 เพิ่ม 1 ทุกครั้งที่ A ผ่าน  │
│ TC8 │ สลับ Manual Mode ขณะทำงาน    │ ระบบหยุด พร้อม Manual Control     │
│ TC9 │ Manual Start C2 ก่อน C3      │ C2 เดินได้ (ไม่มี Interlock Manual)│
│ TC10│ JAM_SENSOR ON นาน 5 วินาที  │ Fault Set, Motor หยุด              │
│ TC11│ C1_STOP button กด            │ C1 หยุด (Manual), C2,C3 ยังเดิน  │
│ TC12│ กด SYS_STOP                  │ C1→C2→C3 หยุดตามลำดับ            │
└─────┴──────────────────────────────┴───────────────────────────────────┘
```

### Test Record Template

```
Test Date: ____________
Tester: ____________
PLC: ____________
Software Version: ____________

┌─────┬──────────────────────┬──────────┬──────────────────────────┐
│ TC# │ Test Description     │ Pass/Fail│ Notes                    │
├─────┼──────────────────────┼──────────┼──────────────────────────┤
│ TC1 │ Auto Start Sequence  │          │                          │
│ TC2 │ E-Stop               │          │                          │
│ TC3 │ OL Fault             │          │                          │
│ ... │ ...                  │          │                          │
└─────┴──────────────────────┴──────────┴──────────────────────────┘
Signature: ____________   Date: ____________
```

---

## Step 187: Factory Acceptance Test (FAT) {#step-187}

### FAT ขั้นตอน

```
FAT = การทดสอบที่โรงงาน (ก่อน ส่งมอบงาน)

เอกสารที่ต้องเตรียม:
1. FAT Procedure Document
2. Test Cases (จาก Step 186)
3. I/O List สำหรับ verify
4. Wiring Diagram

ขั้นตอน FAT:
Step 1: Visual Inspection
  □ Cabinet Wiring ถูกต้อง
  □ Label ครบทุกเส้น
  □ Fuse/CB ขนาดถูกต้อง

Step 2: Power On Test
  □ Voltage Measurement ทุก Circuit
  □ I/O Test (force แต่ละ I/O)

Step 3: Function Test
  □ ทดสอบทุก TC# จากรายการ
  □ Witness โดย Customer

Step 4: Performance Test
  □ Run 1 ชั่วโมงต่อเนื่อง
  □ ตรวจสอบ Count ถูกต้อง

Step 5: Documentation Sign-off
  □ Customer ลงนาม FAT Report
```

---

## Step 188: Commissioning Steps {#step-188}

### ขั้นตอน Site Commissioning

```
การ Commissioning = การติดตั้งและ Start-up ที่ Site จริง

Phase 1: Pre-Commissioning (ก่อน Power On)
┌────────────────────────────────────────────────────────────────┐
│ □ ตรวจสอบ Cable Routing ถูกต้อง                                │
│ □ ตรวจสอบ Motor Rotation Direction (ก่อนต่อสายพาน)            │
│ □ Megger Test Motor Cables (Insulation Resistance ≥ 1 MΩ)     │
│ □ ตรวจสอบ Earthing/Grounding ครบ                               │
│ □ ตรวจสอบ E-Stop ทุกจุดทำงาน                                  │
│ □ ตรวจสอบ Safety Guard ครบ                                     │
└────────────────────────────────────────────────────────────────┘

Phase 2: Power-On Checks
┌────────────────────────────────────────────────────────────────┐
│ □ PLC Power On → ไม่มี ERROR LED                               │
│ □ ตรวจ Supply Voltage (24VDC, 220VAC)                         │
│ □ Upload Program to PLC                                        │
│ □ Verify Program Checksum                                      │
│ □ Test I/O ทุกจุด (force test)                                 │
│ □ ตรวจสอบ E-Stop ทุก Push Button ทำงาน                        │
└────────────────────────────────────────────────────────────────┘

Phase 3: Functional Test
┌────────────────────────────────────────────────────────────────┐
│ □ Motor Rotation Direction ถูกต้อง                             │
│ □ Start/Stop Sequence ถูกต้อง                                  │
│ □ Safety Interlock ทำงาน                                       │
│ □ Sensor ตรวจจับได้ถูกต้อง (calibrate ถ้าต้อง)                │
│ □ Sorting Function ถูกต้อง                                     │
│ □ Speed Control ทำงาน                                          │
│ □ HMI แสดงผลถูกต้อง                                           │
│ □ Alarm ทำงาน                                                  │
└────────────────────────────────────────────────────────────────┘

Phase 4: Performance Test
┌────────────────────────────────────────────────────────────────┐
│ □ Run 4 ชั่วโมงต่อเนื่อง                                       │
│ □ Production Count ถูกต้อง (compare กับ manual count)         │
│ □ Accuracy Test (Sort ถูกต้อง 100%)                            │
│ □ ตรวจสอบ Temperature Motor (ไม่ร้อนผิดปกติ)                  │
│ □ ตรวจสอบ Vibration (ผิดปกติไหม)                              │
└────────────────────────────────────────────────────────────────┘

Phase 5: Documentation
┌────────────────────────────────────────────────────────────────┐
│ □ As-Built Drawing Update                                      │
│ □ I/O Termination List                                         │
│ □ Test Report                                                  │
│ □ Operator Manual                                              │
│ □ Spare Parts List                                             │
│ □ Customer Sign-off                                            │
└────────────────────────────────────────────────────────────────┘
```

### Critical Points ที่ต้องระวัง

```
⚠ ระวัง: Sensor Calibration
  - Proximity Sensor: ปรับ Sensing Distance
  - Barcode Scanner: ตรวจสอบ FOV และ Focus
  - ทดสอบทุก Product Type จริง

⚠ ระวัง: Motor Rotation
  - ตรวจสอบ Forward/Reverse ก่อนใส่สายพาน
  - ถ้าผิด: สลับ 2 เส้นใน Motor Terminal (U-V สลับ)

⚠ ระวัง: Conveyor Tension
  - Conveyor Belt ต้องตึงพอดี ไม่ตึงหรือหย่อนเกินไป
  - ทดสอบ Load Test ด้วยของจริง
```

---

## Step 189: Troubleshooting Guide {#step-189}

### Fault Code Table

```
┌──────────┬─────────────────────────────┬────────────────────────────┐
│ Code     │ Fault Description            │ Solution                   │
├──────────┼─────────────────────────────┼────────────────────────────┤
│ 11       │ C1 Motor Overload           │ 1. ตรวจ Motor Temperature   │
│          │                             │ 2. ตรวจ Mechanical jam      │
│          │ ตรวจ: I1.0 = OFF            │ 3. Reset OL Relay           │
│          │ (OL Relay Tripped)          │ 4. ลด Load หรือ Adjust OL  │
├──────────┼─────────────────────────────┼────────────────────────────┤
│ 12       │ C1 Jam Detected             │ 1. หยุดสายพาน               │
│          │                             │ 2. นำสิ่งที่ค้างออก          │
│          │ ตรวจ: I2.2 = ON > 5s        │ 3. ตรวจ Sensor Sensitivity  │
│          │ (Prox sensor ON นานเกิน)    │ 4. Reset Fault              │
├──────────┼─────────────────────────────┼────────────────────────────┤
│ 99       │ Emergency Stop Active       │ 1. ตรวจหาสาเหตุ E-Stop     │
│          │                             │ 2. แก้ไขปัญหา               │
│          │ ตรวจ: I0.6 = OFF            │ 3. ดึง E-Stop Button        │
│          │                             │ 4. กด RESET                 │
└──────────┴─────────────────────────────┴────────────────────────────┘
```

### Diagnostic Monitor Points

```
ตรวจสอบสถานะด้วย Force Watch:

ถ้า C1 ไม่เริ่ม ตรวจลำดับนี้:
  1. M0.1 (SYS_Running) = ?     ถ้า 0 → System ไม่ Run
  2. M2.0 (C1_Fault) = ?        ถ้า 1 → C1 มี Fault
  3. X14 (C2_Speed_FB) = ?      ถ้า 0 → C2 ยังไม่ถึงความเร็ว
  4. I0.0 (C1_START) = ?        ถ้า 0 → ปุ่ม Start ไม่ทำงาน

ถ้า Sort ผิด ตรวจ:
  1. X20 (SENSOR_A) = ? เมื่อ Product A ผ่าน
  2. M3.1 (Prod_B_Flag) = ?
  3. T10 (Position Delay) = ?
  4. Y3 (DIVERT_GATE) = ?
```

---

## Step 190: Workshop - Complete Commissioning Checklist {#step-190}

### Commissioning Checklist สมบูรณ์

```
============================================================
CONVEYOR SORTING SYSTEM
COMMISSIONING CHECKLIST
============================================================

Project:     _____________________________
Location:    _____________________________
Date:        _____________________________
Engineer:    _____________________________
Customer:    _____________________________

------------------------------------------------------------
SECTION A: PRE-POWER CHECKS
------------------------------------------------------------

1. Physical Installation
   □ PLC mounted securely in cabinet
   □ All I/O modules seated correctly
   □ Power supply connections verified
   □ Earth bonding complete
   □ Cable management neat and labeled

2. Wiring Verification
   □ I0.0 - C1 Start Button: TESTED OK ____
   □ I0.1 - C1 Stop Button: TESTED OK  ____
   □ I0.2 - C2 Start Button: TESTED OK ____
   □ I0.3 - C2 Stop Button: TESTED OK  ____
   □ I0.4 - C3 Start Button: TESTED OK ____
   □ I0.5 - C3 Stop Button: TESTED OK  ____
   □ I0.6 - E-Stop: TESTED OK (NC type) ____
   □ I0.7 - Mode Switch: TESTED OK      ____
   □ I1.0 - C1 OL Relay: TESTED OK (NC) ____
   □ I1.1 - C2 OL Relay: TESTED OK (NC) ____
   □ I1.2 - C3 OL Relay: TESTED OK (NC) ____
   □ I2.0 - Product A Sensor: TESTED OK ____
   □ I2.1 - Product B Sensor: TESTED OK ____
   □ I2.2-24 - Jam Sensors: TESTED OK  ____

   □ Q0.0 - C1 Motor Contactor: TESTED  ____
   □ Q0.1 - C2 Motor Contactor: TESTED  ____
   □ Q0.2 - C3 Motor Contactor: TESTED  ____
   □ Q0.3 - Divert Gate: TESTED         ____
   □ Q0.4 - Alarm Siren: TESTED         ____
   □ Q0.5 - Green Beacon: TESTED        ____
   □ Q0.6 - Amber Beacon: TESTED        ____
   □ Q0.7 - Red Beacon: TESTED          ____

------------------------------------------------------------
SECTION B: PLC PROGRAM VERIFICATION
------------------------------------------------------------

   □ Program Uploaded Successfully
   □ Program Version: _______________
   □ No Syntax Errors
   □ All Tags Named Correctly
   □ Comments Present on all Networks

   Safety Network Verification:
   □ NW1: E-Stop Detection - VERIFIED
   □ NW2-4: OL Detection - VERIFIED
   □ NW5: Jam Detection - VERIFIED
   □ NW7: Any Fault Flag - VERIFIED
   □ NW8: Fault Reset Logic - VERIFIED
   □ NW9: Stop on Fault - VERIFIED

------------------------------------------------------------
SECTION C: FUNCTIONAL TESTS
------------------------------------------------------------

Test #1: E-Stop Function
   Procedure: กด E-Stop ขณะ Conveyors ทำงาน
   Expected: ทุก Output = OFF ทันที (<100ms)
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #2: Auto Start Sequence
   Procedure: Mode=Auto, กด SYS_START
   Expected: C3 เริ่ม → C2 เริ่ม → C1 เริ่ม
   Timing: C3 Start → C2 Start delay ≈ 2s (SpeedFB)
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #3: Auto Stop Sequence
   Procedure: กด SYS_STOP
   Expected: C1 Stop → C2 Stop (5s delay) → C3 Stop
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #4: Product A Sorting
   Procedure: ส่ง Product A ผ่าน Sensor
   Expected: ไม่เบี่ยง, Chute A Gate เปิด, Count A +1
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #5: Product B Sorting
   Procedure: ส่ง Product B ผ่าน Sensor
   Expected: Divert Gate เปิดหลัง 2s, Chute B Gate เปิด
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #6: Motor OL Fault
   Procedure: Simulate OL Trip (กด Test Button บน OL Relay)
   Expected: Motor หยุด, Amber Beacon, Fault Code แสดง
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #7: Jam Detection
   Procedure: ขัดขวาง JAM Sensor นาน 5 วินาที
   Expected: Fault Set, Motor Stop, Alarm
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #8: Speed Control
   Procedure: ปรับ Speed SP ที่ HMI
   Expected: VFD ปรับความเร็วตาม SP
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #9: Counting Accuracy
   Procedure: ส่ง Product A 10 ชิ้น, B 10 ชิ้น
   Expected: Count A=10, Count B=10, Total=20
   Result: PASS □  FAIL □
   Notes: _____________________________

Test #10: 4-Hour Continuous Run
   Start Time: _________ End Time: _________
   Products Processed: A=_____ B=_____ Total=_____
   Faults During Test: _____
   Result: PASS □  FAIL □

------------------------------------------------------------
SECTION D: DOCUMENTATION
------------------------------------------------------------

   □ As-Built Drawings Updated
   □ I/O Test Sheet Completed
   □ Test Report Completed
   □ Spare Parts List Provided
   □ Operator Training Completed
   □ Maintenance Manual Provided

------------------------------------------------------------
APPROVAL
------------------------------------------------------------

Engineer Sign-off:
   Name: _______________________
   Signature: __________________
   Date: _______________________

Customer Sign-off:
   Name: _______________________
   Signature: __________________
   Date: _______________________

Punch List (Items to Complete):
   1. _________________________________
   2. _________________________________
   3. _________________________________
   4. _________________________________
============================================================
```

### Final Tips สำหรับ PLC Programmer มือใหม่

```
┌──────────────────────────────────────────────────────────────┐
│ คำแนะนำสำหรับ PLC Programmer                                  │
│                                                              │
│ 1. Safety First: เขียน Safety Logic ก่อนเสมอ                 │
│ 2. Start Small: ทดสอบทีละ Network ไม่ใช่ทั้งโปรแกรม           │
│ 3. Comment Everything: อนาคตคุณจะขอบคุณตัวเอง                │
│ 4. Simulate First: ทดสอบใน Simulator ก่อน Real PLC            │
│ 5. Document Changes: ใส่วันที่และเหตุผลทุกครั้งที่แก้โปรแกรม │
│ 6. Backup Often: Save ทุกครั้งก่อน Edit                       │
│ 7. Use Tags: ห้ามใช้ Raw Address เช่น I0.0 แทน "E_STOP"       │
│ 8. Test Edge Cases: ทดสอบสถานการณ์ขอบเขต (Power Loss, etc.)  │
│ 9. Ask Why: เข้าใจว่าทำไม ไม่ใช่แค่ทำอะไร                    │
│ 10. Keep Learning: PLC Technology เปลี่ยนเร็ว อัพเดทตัวเอง    │
└──────────────────────────────────────────────────────────────┘
```

---

## สรุป Part 10

```
┌──────────────────────────────────────────────────────────────┐
│ สิ่งที่เรียนรู้ใน Part 10:                                    │
│                                                              │
│ ✓ วิเคราะห์ความต้องการระบบ (Functional Requirements)         │
│ ✓ สร้าง I/O List อย่างละเอียด                                │
│ ✓ Tag Naming Convention ที่ดี                                 │
│ ✓ Program Structure - การจัดระเบียบโปรแกรม                    │
│ ✓ Safety First - เขียน Safety Logic ก่อนเสมอ                 │
│ ✓ Sequence Control - Auto Start/Stop                         │
│ ✓ Sorting Logic - Product Detection and Sorting              │
│ ✓ Alarm Management - Fault Detection and Reporting           │
│ ✓ HMI Interface - PLC-HMI Communication                     │
│ ✓ Test Cases และ FAT Procedure                               │
│ ✓ Commissioning Checklist ครบถ้วน                            │
│ ✓ Troubleshooting Guide                                      │
│                                                              │
│ ยินดีด้วย! คุณเรียนรู้การเขียนโปรแกรม PLC จริงครั้งแรก!     │
└──────────────────────────────────────────────────────────────┘
```

---

**[← Part 09: Data Move Instructions](part09_data_move.md)** | **[Part 11: Advanced Instructions →](part11_advanced_instructions.md)**

---

*หมายเหตุ: โปรแกรมในบทนี้เป็นตัวอย่างสำหรับการเรียนรู้ ก่อนนำไปใช้งานจริงควรผ่านการ Review และทดสอบอย่างละเอียดโดยวิศวกรที่มีประสบการณ์*
