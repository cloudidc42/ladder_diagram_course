# Part 14: ระบบ Conveyor พื้นฐาน (Steps 251-270)

## ภาพรวมของบทเรียน

ระบบ Conveyor เป็นหัวใจสำคัญของการผลิตในโรงงานอุตสาหกรรม บทนี้ครอบคลุมการออกแบบและเขียนโปรแกรม PLC สำหรับ Conveyor ตั้งแต่แบบเดี่ยว ไปจนถึงระบบ Conveyor ที่ซับซ้อน รวมถึงการควบคุมความเร็ว การตรวจจับการติดขัด และการออกแบบระบบสมบูรณ์

---

## Step 251: Single Conveyor Control

### โครงสร้าง Conveyor พื้นฐาน

```
Single Conveyor System:

     PB-Start    PB-Stop    E-Stop
         │          │          │
         ▼          ▼          ▼
     ┌───────────────────────────────────────────────────┐
     │                   PLC                             │
     │  I0.0        I0.1        I0.2                    │
     │  Start       Stop        E-Stop                   │
     │                                                   │
     │  Q0.0                                             │
     │  Motor                                            │
     └───────────────────────────────────────────────────┘
         │
         ▼
     KM1 Coil (Contactor)
         │
         ▼
     Motor Drive
         │
     ═══▼════════════════════════════════════════
     Drive Pulley            Tail Pulley
     (Driven)                (Idler)
         │                       │
     ════════════════════════════════════════════
                    Belt
     ════════════════════════════════════════════
         │
     Tension Rollers (Take-up)
```

### Basic Single Conveyor Ladder

```ladder
// Single Conveyor Control
// Siemens S7-1200

// Network 1: Conveyor Start/Stop
|  %I0.2   |  %I0.3   |  %I0.0   |  %I0.1   |
|ESTOP_NC  |OL_NC    |PB_STOP_NC|PB_START  |──────────────────( %Q0.0 )
|E-Stop    |Overload  |Stop      |Start     |                  CONV_Motor
|Normally  |Normally  |NC type   |NO type   |
|Closed    |Closed    |          |          |
|  %Q0.0   |
|CONV_Motor|─────────────────────────────────────────────────────┘
|(Self Hold)|

// Network 2: Running Indicator
|  %Q0.0   |
|CONV_Motor|──────────────────────────────────────────────────( %Q0.1 )
|          |                                                  RUN_Lamp_Green

// Network 3: Overload Alarm
|  %I0.3   |
|OL_NC     |────/────────────────────────────────────────────( %Q0.2 )
|(Inverse) |                                                  OL_Lamp_Red
```

---

## Step 252: Conveyor Speed Control with VFD

### Speed Control Architecture

```
Conveyor Speed Control:

HMI/PLC ──► Speed Setpoint (0-100%) ──► Analog Output ──► VFD
                                                            │
                                                            ▼
                                                       Motor Speed
                                                       0-1500 RPM
                                                            │
                                                            ▼
                                                      Belt Speed
                                                       0-30 m/min
                                                            │
                                          Speed Feedback ◄──┘
                                          (AO from VFD)
```

### VFD Conveyor Program

```ladder
// Conveyor Speed Control with VFD
// Siemens S7-1200

// Network 1: VFD Start/Stop
|  ESTOP_OK |  OL_OK  |  PB_STOP_NC |  PB_START |
|           |         |             |           |──────────────( %Q0.0 )
|           |         |             |           |             VFD_Run_CMD
|  %Q0.0    |
|VFD_Run    |──────────────────────────────────────────────────┘

// Network 2: Scale Speed Setpoint (0.0-100.0% → AO 0-27648)
|  Always   |
|  ON       |──[MUL]───────────────────────────────────────────────────┤
|           │  IN1: DB.Speed_Setpoint (Real: 0.0-100.0%)              │
|           │  IN2: 276.48                                             │
|           │  OUT: DB.Speed_AO_Scaled (Int)                          │

|  Always   |
|  ON       |──[LIMIT]─────────────────────────────────────────────────┤
|           │  MN: 0                                                   │
|           │  IN: DB.Speed_AO_Scaled                                  │
|           │  MX: 27648                                               │
|           │  OUT: %QW80 (Analog Output)                              │

// Network 3: Read Actual Speed (from VFD AO → PLC AI)
|  Always   |
|  ON       |──[SCALE_X]───────────────────────────────────────────────┤
|           │  INPUT: %IW64 (Analog Input, 0-27648)                   │
|           │  LO_LIM: 0.0                                             │
|           │  HI_LIM: 1500.0 (RPM)                                   │
|           │  VALUE: DB.Actual_RPM                                    │

// Network 4: Belt Speed Calculation (RPM → m/min)
|  Always   |
|  ON       |──[MUL]───────────────────────────────────────────────────┤
|           │  IN1: DB.Actual_RPM                                      │
|           │  IN2: 0.02 (meter/revolution)                            │
|           │  OUT: DB.Belt_Speed_mpm (m/min)                         │
```

---

## Step 253: Multiple Conveyor Cascade System

### Cascade Start/Stop Concept

```
Cascade Conveyor Layout:

Feed     Conveyor 1    Conveyor 2    Conveyor 3    Output
Hopper   (Input)       (Main)        (Output)      Bin
  │                                                  │
  │   ┌─────────────────────────────────────────┐   │
  │   │ Belt 1                                  │   │
  └──►│═══════════════════════════════════════► │──►│
      │                                         │   │
      │ Belt 2                                  │   │
      │════════════════════════════════════════►│
      │                                         │
      │ Belt 3                                  │
      │════════════════════════════════════════►│
      └─────────────────────────────────────────┘

Cascade Start: C3 first → C2 → C1 (ป้องกัน Material Jam)
Cascade Stop:  C1 first → C2 → C3 (Flush out material)
```

### Cascade Start Ladder Program

```ladder
// Cascade Conveyor Start/Stop
// 3 Conveyors with Sequence

// ============ START SEQUENCE ============
// Network 1: Start Permission
|  ESTOP_OK |  ALL_OL_OK |
|           |            |────────────────────────────────────( %M0.0 )
|           |            |                                    Start_Permit

// Network 2: Sequence Start Trigger
|  %M0.0    |  %I0.0   |
|Start_     |PB_SEQ_   |────────────────────────────────────( %M0.1 )
|Permit     |START     |                                    Seq_Start_Trig
|  %M0.1    |
|Seq_Start  |────────────────────────────────────────────────┘

// Network 3: Start Conveyor 3 First
|  %M0.1    |
|Seq_Start  |────────────────────────────────────────────────( %Q0.2 )
|           |                                                 CONV3_Motor

// Network 4: Wait 3s, Then Start Conveyor 2
|  %Q0.2    |
|CONV3_Run  |────────────────────────────────────────[TON T_Del23]
|           |                                        PT=T#3S

|  %T_Del23.Q |  %Q0.2   |
|Del23_Done   |CONV3_Run |────────────────────────────────────( %Q0.1 )
|             |          |                                    CONV2_Motor

// Network 5: Wait 3s, Then Start Conveyor 1
|  %Q0.1    |
|CONV2_Run  |────────────────────────────────────────[TON T_Del12]
|           |                                        PT=T#3S

|  %T_Del12.Q |  %Q0.1   |
|Del12_Done   |CONV2_Run |────────────────────────────────────( %Q0.0 )
|             |          |                                    CONV1_Motor

// ============ STOP SEQUENCE ============
// Network 6: Stop Sequence Trigger
|  %I0.1    |  %M0.1   |
|PB_SEQ_    |Seq_Start |────────────────────────────────────( %M0.2 )
|STOP       |(NC)      |                                    Seq_Stop_Trig

// Network 7: Stop Conveyor 1 First (Cut Feed)
|  %M0.2    |
|Seq_Stop   |────────────────────────────────────────────────( RESET %Q0.0 )
|           |                                                 CONV1_Motor OFF

// Network 8: Wait for Belt Empty, Then Stop Conv2
|  %M0.2    |
|Seq_Stop   |────────────────────────────────────────[TON T_Flush12]
|           |                                        PT=T#5S

|  %T_Flush12.Q |
|Flush12_Done   |────────────────────────────────────────────( RESET %Q0.1 )
|               |                                             CONV2_Motor OFF

// Network 9: Wait and Stop Conv3
|  %T_Flush12.Q |
|Flush12_Done   |────────────────────────────────────[TON T_Flush23]
|               |                                    PT=T#5S

|  %T_Flush23.Q |
|Flush23_Done   |────────────────────────────────────────────( RESET %Q0.2 )
|               |                                             CONV3_Motor OFF
```

---

## Step 254: Jam Detection

### Belt Jam Detection Methods

```
Method 1: Underspeed Detector (Speed Switch)
- ติด Tachometer / Pulse Encoder บน Belt
- ตรวจสอบ Belt RPM
- ถ้า RPM < Setpoint → JAM!

Method 2: Zero Speed Relay
- ใช้ Proximity Sensor ตรวจจับ Paddle บน Pulley
- ถ้าไม่มี Pulse ใน 3 วินาที → JAM!

Method 3: Motor Current Monitoring
- ถ้า Current สูงกว่า Normal > X% → JAM!
- ใช้ Current Transformer + Analog Input

Method 4: Belt Tension Monitoring
- ใช้ Load Cell ตรวจสอบ Belt Tension
- Tension ต่ำ → Belt หย่อน หรือ ขาด
- Tension สูง → Jam
```

### Jam Detection Ladder Program

```ladder
// Belt Jam Detection with Zero Speed Relay
// Siemens S7-1200

// Network 1: Jam Detection Timer
//   เมื่อ Motor สั่งรัน แต่ไม่มี Speed Pulse ใน 3 วินาที = JAM
|  %Q0.0    |  %I0.4   |
|Conv_Motor |SPEED_    |────────────────────────────────────────────( RESET %T_Jam.Q )
|Running    |PULSE_NC  |   ถ้ามี Pulse = Reset Timer
|           |(ถ้ามี    |
|           | Pulse    |
|           | = ON)    |

|  %Q0.0    |  %I0.4   |
|Conv_Motor |SPEED_    |────────────────────────────────────────────[TON T_Jam]
|           |PULSE     |                                            PT=T#3S
|           |(NC)      |  ถ้าไม่มี Pulse ครบ 3s = JAM

// Network 2: Jam Alarm
|  %T_Jam.Q |
|Jam_Timeout|────────────────────────────────────────────────────────( %M3.0 )
|           |                                                        CONV_JAM

// Network 3: Stop Conveyor on Jam
|  %M3.0    |
|CONV_JAM   |────────────────────────────────────────────────────────( RESET %Q0.0 )
|           |                                                        Conv_Motor OFF

// Network 4: Jam Alarm Output
|  %M3.0    |
|CONV_JAM   |────────────────────────────────────────────────────────( %Q0.5 )
|           |                                                        JAM_Alarm_Lamp
|           |────────────────────────────────────────────────────────( %Q0.6 )
|           |                                                        JAM_Buzzer
```

### Current-Based Jam Detection

```ladder
// Jam Detection via Motor Current
// อ่านค่ากระแสจาก Current Transformer + Analog Input

// Network 1: Read Motor Current
|  Always   |
|  ON       |──[SCALE_X]────────────────────────────────────────────────┤
|           │  INPUT: %IW66 (CT Signal, 0-27648 = 0-20A)              │
|           │  LO_LIM: 0.0                                              │
|           │  HI_LIM: 20.0                                             │
|           │  VALUE: DB.Motor_Current (Amps)                          │

// Network 2: Compare vs Jam Threshold
//   Normal running current = 9A
//   Jam current threshold = 15A
|  DB.Motor_ |
|  Current   |──[CMP_GT 15.0]────────────────────────────────────( %M3.1 )
|            |                                                    Current_High

// Network 3: Confirm Jam (High Current for 2s)
|  %Q0.0    |  %M3.1   |
|Conv_Run   |Current   |────────────────────────────────────────────[TON T_CurrJam]
|           |High      |                                            PT=T#2S

|  %T_CurrJam.Q |
|Current_Jam    |────────────────────────────────────────────────────( %M3.2 )
|               |                                                    CURR_JAM_Detect
```

---

## Step 255: Product Tracking

### Product Tracking Concept

```
Product Tracking on Conveyor:

Sensor 1           Sensor 2           Sensor 3
(Entry)            (Mid)              (Exit)
   │                  │                  │
───┼──────────────────┼──────────────────┼───────────────
   │  [Product 1] →   │  [Product 2] →   │  [Product 3]
   │                  │                  │
Belt Speed: V m/min

Position Calculation:
Position = Position_when_detected + (Belt_Speed × Time_since_detection)
```

### Product Tracking Ladder Program

```ladder
// Product Tracking - Entry to Exit
// Siemens S7-1200

// Network 1: Product Detection at Entry Sensor
|  %I0.0    |
|SENSOR_IN  |────────────────────────────────────────────────────────( SET %M10.0 )
|(Rising    |                                                        Product_Present
| Edge)     |

// Network 2: Start Position Timer When Product Detected
|  %M10.0   |
|Product_   |────────────────────────────────────────────────────────[TON T_Track]
|Present    |                                                        PT=T#10S

// Network 3: Calculate Travel Time
//   Belt Speed = 15 m/min = 0.25 m/s
//   Sensor_to_Exit Distance = 2.5m
//   Travel Time = 2.5 / 0.25 = 10 seconds

|  %T_Track.Q |
|Travel_Done  |────────────────────────────────────────────────────( SET %M10.1 )
|             |                                                     Product_At_Exit

// Network 4: Product Count at Exit
|  %M10.1   |
|Product_At |────────────────────────────────────────────────────[CTU C_Products]
|Exit       |                                                     Count Up
|           |────────────────────────────────────────────────────( RESET %M10.0 )
|           |                                                     Clear Present flag
|           |────────────────────────────────────────────────────( RESET %M10.1 )
|           |                                                     Clear At Exit flag
```

### Multi-Product Tracking with Array

```ladder
// Advanced Tracking using Array (Shift Register)
// เก็บ Product ทุกชิ้นที่อยู่บน Belt

// Track Array: BOOL Array[0..9] - แต่ละ bit = Position บน Belt
// Shift Rate: Shift ทุก 1 วินาที (ความเร็ว = 1 bit/s = X m/s)

// Network 1: Shift Register (every 1 second timer pulse)
|  T_Shift.Q |
|Shift_Pulse |──[SHR DB.Track_Array BY 1]────────────────────────────┤
|            |  (Shift Right = Product moves forward)                │

// Network 2: Load New Product at Bit 0
|  %I0.0    |
|SENSOR_IN  |──────────────────────────────────────────────────────( SET DB.Track_Array[0] )
|(Rising)   |                                                       New product enters

// Network 3: Product Exit Detection (bit 9 = exit position)
|  DB.Track_Array[9] |
|Exit_Position       |────────────────────────────────────────────( %M11.0 )
|                    |                                            Product_Exiting
|                    |────────────────────────────────────────────[INC DB.Product_Count]
```

---

## Step 256: Accumulation Conveyor

### Accumulation Conveyor Concept

```
Accumulation Conveyor (แบบสะสม/ไม่ต่อเนื่อง):

Zone 1 (Input)    Zone 2 (Mid)     Zone 3 (Output)
[Motor 1]         [Motor 2]         [Motor 3]
   │                  │                  │
───┼──────────────────┼──────────────────┼───────
                                              │
Zone Sensor 1      Zone Sensor 2      Downstream
(S1: Zone Full?)   (S2: Zone Full?)   (Gate/Machine)
                                         │
                                    If Downstream BUSY:
                                    Zone 3 STOPS
                                    (Product Accumulates)

Logic: Each Zone stops when downstream is FULL or BUSY
```

### Zone Control Logic

```ladder
// Accumulation Conveyor - 3 Zone Control
// Siemens S7-1200

// Network 1: Zone 3 (Output Zone)
//   Stop if Downstream Gate is CLOSED
|  ESTOP_OK |  %I0.5   |  %I0.3   |
|           |GATE_OPEN |Z3_START  |────────────────────────────────( %Q0.2 )
|           |(Downstream|(Start   |                                CONV_Z3
|           | Ready)   | trigger) |
|  %Q0.2    |
|CONV_Z3    |──────────────────────────────────────────────────────┘

// Network 2: Zone 2
//   Stop if Zone 3 is STOPPED OR Zone 3 Sensor Full
|  ESTOP_OK |  %Q0.2   |  %I0.4   |  Z2_Trigger |
|           |Z3_Running|Z3_NOT_FULL|             |──────────────( %Q0.1 )
|           |          |(NC when  |             |             CONV_Z2
|           |          | sensor   |             |
|           |          | detects) |             |
|  %Q0.1    |
|CONV_Z2    |──────────────────────────────────────────────────────┘

// Network 3: Zone 1 (Input Zone)
//   Stop if Zone 2 is STOPPED OR Zone 2 Sensor Full
|  ESTOP_OK |  %Q0.1   |  %I0.3   |  Z1_Trigger |
|           |Z2_Running|Z2_NOT_FULL|             |──────────────( %Q0.0 )
|           |          |           |             |             CONV_Z1
|  %Q0.0    |
|CONV_Z1    |──────────────────────────────────────────────────────┘

// Network 4: Zone Full Sensors
|  %I0.3    |
|Z2_SENSOR  |────────────────────────────────────────────────────( %M20.0 )
|           |                                                    Zone2_Full

|  %I0.4    |
|Z3_SENSOR  |────────────────────────────────────────────────────( %M20.1 )
|           |                                                    Zone3_Full
```

---

## Step 257: Divert Gate Control

### Divert Gate คืออะไร

```
Divert Gate System:

Main Belt
═══════════════════════════════════════════════════════
                          │
                     ┌────┴────┐
                     │  Divert │
                     │  Gate   │
                     │ (Servo/ │
                     │ Pneum.) │
                     └────┬────┘
                          │
                 ┌─────────┴─────────┐
                 │                   │
           Divert Belt         Main Continues
           (Second Line)
```

### Divert Gate Control Program

```ladder
// Divert Gate Control with Product Sorting
// Siemens S7-1200

// Network 1: Read Product Barcode/RFID Result
//   0 = Good, 1 = Reject, 2 = Grade A, 3 = Grade B
// (Assume reading already done, result in DB.Product_Grade)

// Network 2: Divert Decision based on Grade
|  DB.Product_Grade = 1 |
|  (REJECT)             |────────────────────────────────────( SET %M30.0 )
|                       |                                    Divert_Needed

// Network 3: Wait for Product to Reach Gate
//   Product detected at Sensor_In (I0.0)
//   Gate Position = 1.5m from Sensor_In
//   Belt Speed = 15 m/min → Time = 6 seconds

|  %M30.0   |  %I0.0   |  %I0.0   |
|Divert_    |SENSOR_IN |Rising    |────────────────────────────────────[TON T_ToGate]
|Needed     |(Rising   | Edge     |                                    PT=T#6S
|           | Edge)    |          |

// Network 4: Activate Gate When Product Arrives
|  %T_ToGate.Q |
|Gate_Timing   |────────────────────────────────────────────────────( %Q0.3 )
|              |                                                     Divert_Gate
|              |───────────────────────────────────────────────[TON T_GateOpen]
|              |                                                PT=T#500MS

// Network 5: Release Gate After Product Passes
|  %T_GateOpen.Q |
|Gate_Open_Done  |────────────────────────────────────────────────( RESET %Q0.3 )
|                |                                                 Divert_Gate OFF
|                |────────────────────────────────────────────────( RESET %M30.0 )
|                |                                                 Clear Divert flag
```

---

## Step 258: Belt Speed Synchronization

### Synchronization ระหว่าง Conveyors

```
Speed Synchronization Concept:

Conv 1 Speed → Conv 2 Speed should be SAME or SLIGHTLY FASTER
                 to prevent product tipping

Conv 1 Speed = 15 m/min
Conv 2 Speed = 15.5 m/min (slightly faster → no pile up)

Speed Ratio = V2/V1 = 1.033 (3.3% faster)
```

### Speed Sync Ladder Program

```ladder
// Belt Speed Synchronization
// Conv 2 tracks Conv 1 speed with ratio

// Network 1: Read Conv 1 Speed
|  Always   |
|  ON       |──[SCALE_X]────────────────────────────────────────────────┤
|           │  INPUT: %IW64 (VFD1 Frequency Output)                   │
|           │  LO_LIM: 0.0                                              │
|           │  HI_LIM: 50.0 (Hz)                                       │
|           │  VALUE: DB.Conv1_Hz                                       │

// Network 2: Calculate Conv 2 Setpoint
//   Conv 2 Speed = Conv 1 Speed × 1.033
|  Always   |
|  ON       |──[MUL]────────────────────────────────────────────────────┤
|           │  IN1: DB.Conv1_Hz                                        │
|           │  IN2: 1.033                                               │
|           │  OUT: DB.Conv2_Hz_Setpoint                               │

// Network 3: Limit Conv 2 Speed
|  Always   |
|  ON       |──[LIMIT]──────────────────────────────────────────────────┤
|           │  MN: 0.0                                                  │
|           │  IN: DB.Conv2_Hz_Setpoint                                │
|           │  MX: 55.0 (max Hz)                                       │
|           │  OUT: DB.Conv2_Hz_Limited                                │

// Network 4: Scale to AO and Write
|  Always   |
|  ON       |──[MUL → QW82]───────────────────────────────────────────┤
|           │  IN1: DB.Conv2_Hz_Limited                                │
|           │  IN2: 27648/50 = 552.96                                  │
|           │  OUT: %QW82 (Conv 2 Speed AO)                           │
```

---

## Step 259: Conveyor Safety Features

### Safety Switches บน Conveyor

```
Required Safety Devices:

1. Pull Cord / Rope Pull E-Stop
   ─ ตลอดความยาว Belt ทั้งสองด้าน

2. Belt Drift Switch (Belt Misalignment)
   ─ ทุก 30-50m หรือตามมาตรฐาน
   ─ ตรวจสอบ Belt ไม่วิ่งออกนอกแนว

3. Tail End Guard (Nip Point Guard)
   ─ ป้องกันมือเข้าไปใน Tail Pulley

4. Blocked Chute Sensor
   ─ ตรวจจับวัสดุอุดตัน

5. Zero Speed Switch
   ─ ตรวจสอบ Belt หยุดทั้งที่ Motor รัน
```

### Belt Drift Detection

```ladder
// Belt Drift Switch (Misalignment)
// สวิตช์ตรวจจับ Belt วิ่งออกนอกแนว

// Network 1: Belt Drift Detection
|  %I1.0    |
|DRIFT_SW_L |────────────────────────────────────────────────────( SET %M5.0 )
|(Left Drift|                                                    Belt_Drift
| Switch)   |
|  %I1.1    |
|DRIFT_SW_R |────────────────────────────────────────────────────( SET %M5.0 )
|(Right     |                                                    Belt_Drift
| Drift SW) |

// Network 2: Stop All when Drift Detected
|  %M5.0    |
|Belt_Drift |────────────────────────────────────────────────────( RESET %Q0.0 )
|           |                                                    CONV1_Motor OFF
|           |────────────────────────────────────────────────────( RESET %Q0.1 )
|           |                                                    CONV2_Motor OFF
|           |────────────────────────────────────────────────────( RESET %Q0.2 )
|           |                                                    CONV3_Motor OFF

// Network 3: Drift Alarm
|  %M5.0    |
|Belt_Drift |────────────────────────────────────────────────────( %Q0.7 )
|           |                                                    DRIFT_ALARM

// Network 4: Drift Reset (Maintenance must acknowledge)
|  Maint_   |  %M5.0   |
|Reset_PB   |Belt_Drift|────────────────────────────────────────( RESET %M5.0 )
|           |          |                                        Clear Drift Fault
```

---

## Step 260: Conveyor with Weighing Scale

### Dynamic Weighing System

```
Weighing Conveyor System:

Product → [Conveyor] → [Weigh Belt] → [Sort Gate] → Destination

Weigh Belt:
- Load Cell under Belt
- Belt Speed controlled precisely
- Weight = LoadCell Reading / Belt Speed Correction

Sort Decision:
< 95% Target → UNDERWEIGHT → Reject
95-105% Target → GOOD → Pass
> 105% Target → OVERWEIGHT → Reject
```

### Weighing Conveyor Ladder

```ladder
// Dynamic Weighing on Conveyor
// Siemens S7-1200

// Network 1: Read Load Cell (via Analog Input)
|  Always   |
|  ON       |──[SCALE_X]────────────────────────────────────────────────┤
|           │  INPUT: %IW70 (Load Cell AI, 0-27648)                   │
|           │  LO_LIM: 0.0                                              │
|           │  HI_LIM: 10.0 (kg max)                                   │
|           │  VALUE: DB.Raw_Weight_kg                                  │

// Network 2: Tare (Zero) the Scale
|  %I0.5    |
|TARE_PB    |──────────────────────────────────────────────────[MOVE]
|           │  IN: DB.Raw_Weight_kg                                    │
|           │  OUT: DB.Tare_Weight                                     │

// Network 3: Calculate Net Weight
|  Always   |
|  ON       |──[SUB]────────────────────────────────────────────────────┤
|           │  IN1: DB.Raw_Weight_kg                                   │
|           │  IN2: DB.Tare_Weight                                     │
|           │  OUT: DB.Net_Weight                                      │

// Network 4: Accept/Reject Decision
//   Target Weight = 5.0 kg (stored in DB)
//   Tolerance = ±5%

|  DB.Net_Weight >= 4.75 |  DB.Net_Weight <= 5.25 |
|  (Lower Limit OK)      |  (Upper Limit OK)       |──────────────( %M40.0 )
|                        |                         |             Weight_PASS

// Network 5: Divert Reject to Reject Lane
|  %M10.0   |  %M40.0  |
|Product_   |Weight_   |────────────────────────────────────────( SET %M40.1 )
|Present    |PASS (NC) |                                        Reject_This_Product
```

---

## Step 261: Conveyor Maintenance Features

### Maintenance Mode

```ladder
// Maintenance Mode Control
// ช่างสามารถรันแต่ละ Motor โดยอิสระ ด้วยความเร็วต่ำ

// Network 1: Enter Maintenance Mode
|  Maint_   |  ESTOP_OK  |
|Key_Switch |            |────────────────────────────────────────( %M100.0 )
|           |            |                                        Maint_Mode

// Network 2: Maintenance Mode Indicator
|  %M100.0  |
|Maint_Mode |────────────────────────────────────────────────────( %Q1.0 )
|           |                                                    MAINT_Lamp

// Network 3: Jog Conv 1 in Maintenance Mode
|  %M100.0  |  %I2.0   |
|Maint_Mode |JOG_C1_PB |────────────────────────────────────────( %Q0.0 )
|           |(Hold     |                                        CONV1 Jog
|           | Button)  |

// Network 4: Maintenance Speed Limit
//   ใน Maintenance Mode ตั้งความเร็ว VFD เป็น 20% ของ Max
|  %M100.0  |
|Maint_Mode |──────────────────────────────────────────────────[MOVE]
|           │  IN: 5530 (=20% of 27648)                              │
|           │  OUT: %QW80 (AO to VFD)                               │

// Network 5: Block Normal Production in Maint Mode
|  %M100.0  |
|Maint_Mode |────────────────────────────────────────────────────( %M100.9 )
|(NC)       |                                                    Prod_Allowed
```

---

## Step 262: Conveyor HMI Data

### Data สำหรับ HMI Display

```
HMI Screen Data Structure:

Conveyor Status Screen:
├── Conv 1 Status: Running/Stopped/Fault
├── Conv 2 Status: Running/Stopped/Fault
├── Conv 3 Status: Running/Stopped/Fault
├── Belt Speed: xx.x m/min
├── Product Count: xxxx pieces
└── Daily Throughput: xxxxx kg

Fault History:
├── Timestamp
├── Fault Code
└── Fault Description

Trend View:
├── Belt Speed (Last 1 hour)
└── Motor Current (Last 1 hour)
```

### Data Block Design

```
// DB_Conveyor (Siemens S7-1200)

TYPE DB_Conveyor :
  STRUCT
    // Motor Status
    Conv1_Run    : BOOL;
    Conv2_Run    : BOOL;
    Conv3_Run    : BOOL;
    
    // Speeds
    Conv1_Speed  : REAL;     // m/min
    Conv2_Speed  : REAL;
    Conv3_Speed  : REAL;
    
    // Production Data
    Piece_Count  : DINT;     // Total pieces today
    Rejects      : DINT;     // Rejected pieces
    Throughput_kg : REAL;    // kg today
    
    // Fault Bits
    Jam_Fault    : BOOL;
    Drift_Fault  : BOOL;
    OL_Fault     : BOOL;
    
    // Alarms
    Alarm_Active : BOOL;
    Alarm_Code   : INT;
    
    // Setpoints
    Speed_SP     : REAL;     // Speed setpoint %
    Target_Weight: REAL;     // kg
    
  END_STRUCT;
END_TYPE
```

---

## Step 263: Energy Monitoring for Conveyor

### Conveyor Energy Tracking

```ladder
// Energy Monitoring System
// วัดและบันทึกการใช้พลังงาน

// Network 1: Read Power from Energy Meter (Modbus)
|  Always   |
|  ON       |──[MB_COMM_LOAD]──────────────────────────────────────────┤
|           │  Station: 1 (Energy Meter)                              │
|           │  Function: 04 (Read Input Registers)                    │
|           │  Address: 40001 (Active Power kW)                       │
|           │  Length: 4 (kW, kVAR, kVA, PF)                        │
|           │  Destination: DB_Energy.Raw_Data[0]                     │

// Network 2: Calculate Energy Consumption
//   Sample every 1 minute
|  T_1Min.Q  |
|Timer_1Min  |──[ADD]────────────────────────────────────────────────────┤
|            │  IN1: DB_Energy.Total_kWh                               │
|            │  IN2: DB_Energy.Power_kW / 60 (per minute)             │
|            │  OUT: DB_Energy.Total_kWh                               │

// Network 3: Daily Reset at Midnight
|  MIDNIGHT  |
|  (Special  |──────────────────────────────────────────────────────[MOVE]
|   Timer)   │  IN: 0.0                                               │
|            │  OUT: DB_Energy.Daily_kWh                              │
```

---

## Step 264: Complete Conveyor System Design

### Real-World: Food Processing Conveyor System

```
Complete System Layout:

Raw Material     Washing      Inspection     Packing
   Input         Station       Station       Station
     │               │              │             │
[Receive Conv] ─► [Wash Conv] ─► [Inspect Conv] ─► [Pack Conv]
     │                               │
   (Weigh)                    (Camera/Vision)
                                     │
                                  Sort Gate
                                  │       │
                              Pass    Reject
                              Conv    Conv
                                │       │
                             Packing  Waste Bin
```

### System Program Structure

```
Program Organization:

Main (OB1)
├── FC_Safety_Monitor    - ตรวจสอบ Safety สัญญาณทั้งหมด
├── FC_Conv_Control      - Control Logic แต่ละ Conveyor
│   ├── FB_Conv1         - Receive Conveyor (Instance DB1)
│   ├── FB_Conv2         - Wash Conveyor (Instance DB2)
│   ├── FB_Conv3         - Inspect Conveyor (Instance DB3)
│   └── FB_Conv4         - Pack Conveyor (Instance DB4)
├── FC_Product_Track     - Product Tracking
├── FC_Weigh             - Weighing Logic
├── FC_Sort              - Sorting Decision
├── FC_Alarms            - Alarm Management
└── FC_HMI_Interface     - HMI Data Exchange

Interrupt (OB35 - 100ms cycle)
└── FC_Speed_Control     - PID Speed Control (fast cycle)
```

### FB_Conv Function Block

```ladder
// Function Block: FB_Conv (Reusable Conveyor Controller)

// Input Parameters:
//   Start_CMD: BOOL
//   Stop_CMD: BOOL
//   E_Stop_OK: BOOL
//   OL_OK: BOOL
//   Belt_OK: BOOL (Jam + Drift OK)
//   Speed_SP: REAL (%)

// Output Parameters:
//   Motor_Run: BOOL
//   VFD_Run: BOOL
//   Speed_AO: INT
//   Running: BOOL
//   Fault: BOOL
//   Fault_Code: INT

// Network 1: Run Permission
|  E_Stop_OK |  OL_OK   |  Belt_OK  |
|            |          |           |────────────────────────────( #Run_Permit )

// Network 2: Motor Command
|  #Run_Permit|  #Stop_CMD |  #Start_CMD |
|             |    (NC)    |             |────────────────────────( #Motor_Run )
|  #Motor_Run |
|(Self Hold)  |──────────────────────────────────────────────────┘

// Network 3: VFD Speed Output
|  #Motor_Run |
|             |──[MUL]──────────────────────────────────────────────┤
|             │  IN1: #Speed_SP / 100.0                             │
|             │  IN2: 27648.0                                       │
|             │  OUT: #Speed_AO                                     │

// Network 4: Status Output
|  #Motor_Run |
|             |────────────────────────────────────────────────────( #Running )

|  #Run_Permit |
|   (NC)       |────────────────────────────────────────────────────( #Fault )
```

---

## Step 265: Addressing สำหรับ Conveyor System

### Siemens S7-1200 Complete Address List

```
=== DIGITAL INPUTS ===
I0.0 = PB_SEQ_START    I1.0 = DRIFT_SW_C1_L
I0.1 = PB_SEQ_STOP     I1.1 = DRIFT_SW_C1_R
I0.2 = ESTOP_1 (NC)    I1.2 = DRIFT_SW_C2_L
I0.3 = ESTOP_2 (NC)    I1.3 = DRIFT_SW_C2_R
I0.4 = OL_C1 (NC)      I1.4 = SPEED_PULSE_C1
I0.5 = OL_C2 (NC)      I1.5 = SPEED_PULSE_C2
I0.6 = OL_C3 (NC)      I1.6 = PHOTO_ENTRY
I0.7 = PULL_CORD (NC)  I1.7 = PHOTO_EXIT

=== DIGITAL OUTPUTS ===
Q0.0 = KM1_C1          Q1.0 = VFD1_RUN
Q0.1 = KM2_C2          Q1.1 = VFD2_RUN
Q0.2 = KM3_C3          Q1.2 = VFD3_RUN
Q0.3 = SOL_SORT_GATE   Q1.3 = SORT_GATE_2
Q0.4 = C1_RUN_LAMP     Q1.4 = C2_RUN_LAMP
Q0.5 = C3_RUN_LAMP     Q1.5 = FAULT_LAMP
Q0.6 = JAM_BUZZER      Q1.6 = MAINT_LAMP
Q0.7 = SPARE           Q1.7 = SPARE

=== ANALOG INPUTS ===
IW64 = AI_VFD1_SPEED   (4-20mA, 0-60Hz)
IW66 = AI_VFD2_SPEED
IW68 = AI_VFD3_SPEED
IW70 = AI_LOAD_CELL    (4-20mA, 0-10kg)

=== ANALOG OUTPUTS ===
QW80 = AO_VFD1_SPEED   (0-10V, 0-60Hz)
QW82 = AO_VFD2_SPEED
QW84 = AO_VFD3_SPEED
```

### Allen-Bradley CompactLogix

```
// Tag Declarations
Motor_Conv1          : BOOL  (O:0/0)
Motor_Conv2          : BOOL  (O:0/1)
Motor_Conv3          : BOOL  (O:0/2)
VFD1_Run             : BOOL  (O:0/4)
VFD2_Run             : BOOL  (O:0/5)
Conv1_Speed_SP       : REAL  (O:1.Ch0.Data) [0.0-100.0%]
Conv2_Speed_SP       : REAL  (O:1.Ch1.Data)
Conv1_Speed_FB       : REAL  (I:2.Ch0.Data) [0.0-60.0 Hz]
Load_Cell_Weight     : REAL  (I:2.Ch1.Data) [0.0-10.0 kg]
PB_Start             : BOOL  (I:0/0)
PB_Stop              : BOOL  (I:0/1)
ESTOP_1              : BOOL  (I:0/2)
OL_Conv1             : BOOL  (I:0/4)
Jam_Sensor_C1        : BOOL  (I:0/8)
```

### Mitsubishi FX3U

```
// Input Addresses (Octal)
X0  = PB_Start         X10 = Drift_Switch_1
X1  = PB_Stop          X11 = Drift_Switch_2
X2  = ESTOP_1 (NC)     X12 = Drift_Switch_3
X3  = ESTOP_2 (NC)     X13 = Speed_Pulse_C1
X4  = OL_C1 (NC)       X14 = Speed_Pulse_C2
X5  = OL_C2 (NC)       X15 = Photo_Entry
X6  = OL_C3 (NC)       X16 = Photo_Exit
X7  = Pull_Cord (NC)   X17 = Weigh_Complete

// Output Addresses (Octal)
Y0  = KM1_Conv1        Y10 = Run_Lamp_C1
Y1  = KM2_Conv2        Y11 = Run_Lamp_C2
Y2  = KM3_Conv3        Y12 = Run_Lamp_C3
Y3  = Sort_Gate_1      Y13 = Fault_Lamp
Y4  = Sort_Gate_2      Y14 = Jam_Buzzer
Y5  = VFD1_Run         Y15 = Maint_Lamp
Y6  = VFD2_Run         Y16 = Spare
Y7  = VFD3_Run         Y17 = Spare

// Data Registers
D0  = Conv1_Speed_SP     D100 = Piece_Count_Lo
D2  = Conv2_Speed_SP     D101 = Piece_Count_Hi
D10 = Conv1_Speed_FB     D200 = Daily_Total_Lo
D20 = Weight_Net         D201 = Daily_Total_Hi
D22 = Weight_Target      D300 = Alarm_Code
```

---

## Step 266: Conveyor Performance Metrics (OEE)

### การคำนวณ OEE ของ Conveyor Line

```
OEE = Availability × Performance × Quality

Availability = Running Time / Planned Production Time
Performance = Actual Throughput / Target Throughput  
Quality = Good Products / Total Products

ตัวอย่าง:
Planned Production = 8 hours = 480 min
Downtime = 30 min (Jam + Maintenance)
Running Time = 450 min
Availability = 450/480 = 93.75%

Target Speed = 60 products/min
Actual Average = 55 products/min
Performance = 55/60 = 91.7%

Total Products = 25,000
Rejects = 500
Quality = (25000-500)/25000 = 98%

OEE = 93.75% × 91.7% × 98% = 84.3%
```

### OEE Data Collection in Ladder

```ladder
// OEE Data Collection
// Siemens S7-1200

// Network 1: Count Planned Time (in minutes)
|  Always   |
|  ON       |────────────────────────────────────────[TON T_Min_Tick]
|           |                                         PT=T#1M

|  T_Min_Tick.Q |
|Minute_Tick    |────────────────────────────────────[INC DB_OEE.Planned_Min]
|               |────────────────────────────────────[R T_Min_Tick]

// Network 2: Count Running Time
|  %Q0.0    |
|Conv1_Run  |────────────────────────────────────────[TON T_Run_Tick]
|           |                                         PT=T#1M

|  T_Run_Tick.Q |
|Run_Tick       |────────────────────────────────────[INC DB_OEE.Run_Min]

// Network 3: Count Downtime (Faults)
|  %M1.9    |
|Any_Fault  |────────────────────────────────────────[TON T_Fault_Tick]
|           |                                         PT=T#1M

|  T_Fault_Tick.Q |
|Fault_Tick       |────────────────────────────────────[INC DB_OEE.Down_Min]

// Network 4: Count Good and Reject Products
|  Photo_Exit  |  %M40.0  |
|Exit_Sensor  |Weight_PASS|────────────────────────────────[INC DB_OEE.Good_Pieces]

|  Photo_Exit  |  %M40.1  |
|Exit_Sensor  |REJECT    |────────────────────────────────[INC DB_OEE.Reject_Pieces]
```

---

## Step 267: Advanced: Flying Saw / Cutter

### Flying Saw Synchronization

```
Flying Saw Concept (ตัดวัสดุขณะเคลื่อนที่):

Material Flow →→→→→→→→→→→→→→→→→→→→→→→→→→
                         │
                   ┌─────┴──────┐
                   │ Flying Saw │ ← ต้องเคลื่อนที่
                   │ (Axis X)   │   ตามความเร็ว Belt
                   └─────┬──────┘   ขณะตัด
                         │
                   ตัดแล้วกลับ
                   ตำแหน่งเดิม

Requirements:
- Saw Speed MUST match Belt Speed during cut
- Encoder on Belt to measure position
- Servo on Saw axis
```

### Flying Saw Ladder Concept

```ladder
// Flying Saw Synchronization (Simplified Concept)

// Network 1: Calculate Cut Position
|  DB.Material_Len_Req |
|  (Length Setpoint)   |──[ADD current position]-►│CUT_POSITION
|                      |                          │= Start + Req Length

// Network 2: Trigger Sync and Cut
|  Encoder_Position >= CUT_POSITION |
|                                   |────────────────────────────( Sync_Trigger )

// Network 3: Sync Saw to Belt Speed
|  Sync_Trigger  |
|                |──[SET Saw_Speed = Belt_Speed]─────────────────( Saw_Motor_EN )

// Network 4: Execute Cut
|  Saw_At_Speed  |
|                |────────────────────────────────────────────────( CUT_Solenoid )
|                |────────────────────────────────────────────────[TON T_Cut_Time]
|                |                                                PT=T#200MS

// Network 5: Return Saw to Home
|  T_Cut_Time.Q  |
|Cut_Done        |────────────────────────────────────────────────( RESET CUT_Sol )
|                |────────────────────────────────────────────────( Return_Home_CMD )
```

---

## Step 268: Conveyor System Integration

### การเชื่อม Conveyor กับระบบอื่น

```
System Integration:

┌─────────────────────────────────────────────────────────────────┐
│                    ERP/MES System                               │
│                  (Production Orders)                            │
└─────────────────────────────────────────────────────────────────┘
                           │ OPC-UA / MQTT
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SCADA System                                  │
│              (Monitoring + Reporting)                            │
└─────────────────────────────────────────────────────────────────┘
                           │ OPC-DA / Modbus TCP
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PLC Control Layer                              │
│           (Conveyor + Sorting + Weighing)                       │
└─────────────────────────────────────────────────────────────────┘
                           │ PROFIBUS/PROFINET
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│           Field Devices (VFDs, Sensors, Gates)                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 269: Troubleshooting Conveyor Systems

### Common Conveyor Problems

```
Problem 1: Belt Slipping
Symptom: Motor runs but belt speed low, smell
Cause: Low tension, Wet pulley, Overload
Check:
  ✓ Belt Tension (Visual + Sound)
  ✓ Drive Pulley lagging
  ✓ Load on conveyor
  ✓ Motor current vs rated

Problem 2: Belt Tracking Off
Symptom: Belt wanders to one side
Cause: Misaligned idlers, Uneven tension
Check:
  ✓ Idler alignment (all perpendicular to belt direction)
  ✓ Tracking idler adjustment
  ✓ Load distribution (center of belt)

Problem 3: Product Jam at Transfer
Symptom: Products pile up at belt transfer point
Cause: Speed mismatch, Gap too large
Check:
  ✓ Speed ratio between belts (downstream faster)
  ✓ Transfer plate gap < product length

Problem 4: Intermittent Stop
Symptom: Conveyor stops randomly
Cause: Loose connection, Intermittent E-Stop contact
Check:
  ✓ All E-Stop cable connections
  ✓ Safety relay connection
  ✓ Rope Pull switch condition
```

---

## Step 270: Lab Project - Complete Conveyor System

### Lab 14: 3-Belt Cascade Conveyor System

**โจทย์:**
ออกแบบและเขียน PLC Program สำหรับระบบ Conveyor 3 เส้น พร้อม:
- Cascade Start/Stop
- Jam Detection
- Product Counting
- Speed Control (VFD)
- Fault Management

**Hardware:**
```
PLC: Siemens S7-1200 CPU 1214C DC/DC/DC
AI Module: SM 1231 (4 AI)
AO Module: SM 1232 (2 AO)
VFD: 3× Siemens V20 (0.37kW Demo)
HMI: KTP400 Basic (Touch Panel)
Sensors: 3× Proximity NPN
         3× Speed Sensor
         1× Load Cell
Push Buttons: Start, Stop, Reset, E-Stop
```

**Complete Program Structure:**

```ladder
// Main OB1

// ===== SAFETY CHECK =====
// Network 1: Safety OK Signal
|  %I0.2    |  %I0.3    |
|ESTOP_1_NC |ESTOP_2_NC |────────────────────────────────────────( %M0.0 )
|           |           |                                        Safety_OK

// ===== SEQUENCE START =====
// Network 2: Start Sequence Trigger
|  %M0.0    |  %I0.0    |  %M1.0   |
|Safety_OK  |PB_START   |Seq_Stopped|────────────────────────────( %M1.1 )
|           |           |(System   |                             Seq_Start_Trig
|           |           | Stopped) |

// Network 3: Conv 3 Starts First
|  %M1.1    |
|Seq_Start  |────────────────────────────────────────────────────( %Q0.2 )
|           |                                                    CONV3_Motor

// Network 4: 3s delay → Conv 2
|  %Q0.2    |
|CONV3      |────────────────────────────────────────────────[TON T_Del23]
|           |                                                 PT=T#3S

|  %T_Del23.Q |  %Q0.2   |
|Del23_Done   |CONV3_Run  |────────────────────────────────────────( %Q0.1 )
|             |           |                                        CONV2_Motor

// Network 5: 3s delay → Conv 1
|  %Q0.1    |
|CONV2      |────────────────────────────────────────────────[TON T_Del12]
|           |                                                 PT=T#3S

|  %T_Del12.Q |  %Q0.1   |
|Del12_Done   |CONV2_Run  |────────────────────────────────────────( %Q0.0 )
|             |           |                                        CONV1_Motor

// ===== STOP SEQUENCE =====
// Network 6: Stop Sequence Trigger
|  %M0.0    |  %I0.1    |  %M1.1   |
|Safety_OK  |PB_STOP    |Seq_Runn  |────────────────────────────( %M1.2 )
|           |           |(System   |                             Seq_Stop_Trig
|           |           | Running) |

// Network 7: Stop C1 First
|  %M1.2    |
|Seq_Stop   |────────────────────────────────────────────────────( RESET %Q0.0 )
|           |                                                    CONV1 OFF

// Network 8-9: Flush delays
|  %M1.2    |
|Seq_Stop   |────────────────────────────────────────────────[TON T_Flush12]
|           |                                                 PT=T#5S

|  %T_Flush12.Q |
|Flush12_Done   |────────────────────────────────────────────────( RESET %Q0.1 )
|               |────────────────────────────────────────────────[TON T_Flush23]
|               |                                                 PT=T#5S

|  %T_Flush23.Q |
|Flush23_Done   |────────────────────────────────────────────────( RESET %Q0.2 )
|               |────────────────────────────────────────────────( SET %M1.0 )
|               |                                                 Seq_Stopped=True

// ===== JAM DETECTION =====
// Network 10-12 (as shown in Step 254)

// ===== PRODUCT COUNTING =====
// Network 13: Count Products
|  %I0.6    |
|PHOTO_ENTRY|────────────────────────────────────────────────[CTU C0]
|(Rising)   |                                                Count Up

// ===== VFD SPEED CONTROL =====
// Network 14-16 (as shown in Step 252)

// ===== ALARMS =====
// Network 17: Alarm Summary
|  %M3.0    |
|JAM_Fault  |────────────────────────────────────────────────────( %Q0.5 )
|           |                                                    FAULT_Lamp
|  OL_Fault |
|           |────────────────────────────────────────────────────┘
|  Drift_   |
|  Fault    |────────────────────────────────────────────────────┘
```

**Validation Checklist:**
```
☐ Sequential Start: C3→C2→C1 ในระยะเวลาที่กำหนด
☐ Sequential Stop: C1→C2→C3 ในระยะเวลาที่กำหนด
☐ E-Stop หยุดทันทีทุก Conveyor
☐ Jam Detection ทำงาน
☐ Overload หยุด Conveyor ที่เกิด Fault เท่านั้น
☐ Speed Control ทำงาน (ปรับได้ 0-100%)
☐ Product Count ทำงาน
☐ HMI แสดงสถานะถูกต้อง
☐ Reset หลัง Fault ทำงาน
☐ สาย Label ครบ
```

---

## สรุป Part 14

ในบทนี้เราได้เรียนรู้:
1. Single Conveyor Control พื้นฐาน
2. Speed Control ด้วย VFD
3. Cascade Conveyor Start/Stop Sequence
4. Jam Detection หลายวิธี
5. Product Tracking บน Belt
6. Accumulation Conveyor Control
7. Divert Gate Control
8. Belt Speed Synchronization
9. Dynamic Weighing Integration
10. OEE Data Collection
11. Complete System Design

---

## การนำทาง

- [← Part 13: Motor Control พื้นฐาน](part13_motor_control.md)
- [→ Part 15: HMI พื้นฐาน](part15_hmi_basics.md)
- [กลับหน้าหลัก](../README.md)

---

*Part 14 จบ | ต่อไป Part 15: HMI พื้นฐาน*
