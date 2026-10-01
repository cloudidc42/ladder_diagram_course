# Part 35: Variable Frequency Drive (VFD) Control
## Steps 681-700: การควบคุม VFD ด้วย PLC

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 3-4 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 22 (Analog I/O), Part 25 (Modbus)

---

## บทนำ

Variable Frequency Drive (VFD) หรือ Inverter คืออุปกรณ์ควบคุมความเร็วมอเตอร์ AC โดยการปรับความถี่และแรงดันไฟฟ้า

ข้อดีของ VFD:
- ประหยัดพลังงาน 30-60% (สำหรับ Fan/Pump)
- ควบคุมความเร็วได้อย่างต่อเนื่อง
- Soft Start/Stop ลดความเสียหายเครื่องจักร
- ลด Water Hammer ในท่อน้ำ
- ป้องกันมอเตอร์ (Overload, Overcurrent)

---

## Step 681: VFD Operating Principles

### หลักการทำงาน

```
┌────────────────────────────────────────────────────────────┐
│                    VFD Internal Structure                   │
│                                                            │
│  AC Input ──→ Rectifier ──→ DC Bus ──→ Inverter ──→ Motor │
│  3ø 380V        (Diode      (Capacitor   (IGBT PWM)  AC   │
│  50Hz           Bridge)      Filter)     Switching)  Variable│
│                                                            │
│  Converter:  AC → DC  (ทำ DC Bus ~540VDC สำหรับ 380V)    │
│  Inverter:   DC → AC  (สร้าง AC ที่ความถี่ที่ต้องการ)     │
│                                                            │
│  V/f (Volts per Hertz):                                    │
│  - รักษา V/f = คงที่ เพื่อ Constant Flux                 │
│  - 380V / 50Hz = 7.6 V/Hz                                 │
│  - ที่ 25Hz → ส่ง 190V                                    │
│  - ที่ 50Hz → ส่ง 380V                                    │
└────────────────────────────────────────────────────────────┘
```

### V/f Characteristic Curve

```
Voltage
(V)
 400 ├───────────────────────────── Full Voltage
     │                         /
     │                        /  Boost Voltage Region
     │                       /   (สำหรับ High Torque Start)
 200 ├─────────────────────/
     │                    /
     │                   /
     │                  /
   0 ├──────────────────────────────────────→ Frequency (Hz)
     0        25        50        60(Override)
         Base Speed    Rated Speed  Max Speed
```

### Control Modes

```
V/f Control (開環路):
- ง่ายที่สุด
- ไม่ต้องการ Feedback
- ความแม่นยำปานกลาง
- ใช้กับ Fan, Pump, Conveyor ทั่วไป

Vector Control (Sensorless):
- คำนวณ Motor Flux โดยประมาณ
- ดีกว่า V/f
- ไม่ต้องการ Encoder
- ใช้กับ Crane, Compressor

Vector Control (Closed-loop, FOC):
- ต้องการ Encoder Feedback
- ดีที่สุด
- ใช้กับ Servo-class applications
- Torque Control ที่ 0 RPM
```

---

## Step 682: Parameter Setup สำหรับ Motor

### พารามิเตอร์ที่ต้องตั้งค่า

```
Motor Nameplate Data:
┌───────────────────────────────────────────────────────┐
│  Motor Parameter        │ Nameplate Value │ VFD Param │
│  ──────────────────────────────────────────────────── │
│  Rated Voltage          │ 380V           │ P01.01    │
│  Rated Current          │ 15.8A          │ P01.02    │
│  Rated Frequency        │ 50Hz           │ P01.03    │
│  Rated Speed            │ 1450 RPM       │ P01.04    │
│  Rated Power            │ 7.5 kW         │ P01.05    │
│  Power Factor (Cos φ)   │ 0.85           │ P01.06    │
│  Motor Poles            │ 4 (1500 RPM)   │ P01.07    │
│  Stator Resistance      │ (Auto-tune)    │ P01.08    │
└───────────────────────────────────────────────────────┘
```

### พารามิเตอร์ควบคุม

```
ตัวอย่าง: Siemens SINAMICS G120

P1120: Ramp-up time (0-50Hz)     = 10.0 seconds
P1121: Ramp-down time (50-0Hz)   = 15.0 seconds
P1082: Maximum frequency         = 60.0 Hz
P1080: Minimum frequency         = 5.0 Hz
P1300: Control mode              = 0 (V/f)
P0840: Speed command source      = 2 (Analog In 1)
P0700: Command source            = 2 (Terminal)
P1000: Frequency setpoint source = 2 (Analog In 1)

// พารามิเตอร์ Protection
P0640: Current limit             = 150% (1.5x rated)
P0610: Motor overtemperature     = 1 (Monitor + trip)
P0730: PNP/NPN Digital inputs    = 1 (PNP)
```

---

## Step 683: Digital Speed Control

### Control via Digital I/O

```
// ใช้ Multi-Speed Preset (Preset Speeds)
// VFD มีหลาย Preset Speed ที่เลือกด้วย Digital Input

Preset Speed Configuration:
DI1 (I0.0): Run/Stop
DI2 (I0.1): Speed Select Bit 0 (LSB)
DI3 (I0.2): Speed Select Bit 1
DI4 (I0.3): Speed Select Bit 2 (MSB)

Speed Table (3-bit = 8 speeds):
Bit2 | Bit1 | Bit0 | Speed
  0  |  0   |  0   | Stop
  0  |  0   |  1   | 10 Hz (20% = 288 RPM)
  0  |  1   |  0   | 20 Hz (40% = 576 RPM)
  0  |  1   |  1   | 30 Hz (60% = 864 RPM)
  1  |  0   |  0   | 40 Hz (80% = 1152 RPM)
  1  |  0   |  1   | 50 Hz (100% = 1440 RPM)
  1  |  1   |  0   | 55 Hz (110% = 1584 RPM)
  1  |  1   |  1   | 60 Hz (120% = 1728 RPM)
```

### Ladder - Multi-Speed Selection

```
Network 1: เลือกความเร็ว Low (Bit0=1, Bit1=0, Bit2=0)

  |                                                    |
  |--[ ]--[NOT]--[NOT]--[SET]------------------------|
  |  Run   Bit1   Bit2   Q0.1 (Drive DI2 - Bit0)    |
  |  AND                                              |
  |  Speed_Low                                        |
  |                                                    |

Network 2: เลือกความเร็ว Medium

  |                                                    |
  |--[ ]--[ ]--[NOT]--[SET]--[RESET]-----------------|
  |  Run   Bit1   Bit2   Q0.2 (Bit1)  Q0.1 (Bit0)   |
  |  AND   Speed_Medium                               |
  |                                                    |

Network 3: เลือกความเร็ว High

  |                                                    |
  |--[ ]--[NOT]--[ ]--[SET]--[SET]------------------|
  |  Run   Bit1   Bit2  Q0.2 (Bit1)  Q0.3 (Bit2)   |
  |  AND   Speed_High                                 |
  |                                                    |
```

---

## Step 684: Analog Speed Control

### Analog Speed Reference

```
// ใช้ Analog Output จาก PLC ส่ง Speed Reference
// 0-10V หรือ 4-20mA → 0-100% Speed

Network 1: แปลง Speed% เป็น Analog Value

  |                                                    |
  |--[ ]----------[MUL]------------------------------|
  |  "Conv_Run"    IN1: "Conveyor_Speed_Percent"       |
  |               IN2: 276 (27600 max / 100%)          |
  |               OUT: "AO_Conv_Speed"                 |
  |                                                    |

Network 2: เขียน Analog Output

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: "AO_Conv_Speed"                |
  |                 OUT: QW256 (Analog Output Channel) |
  |                                                    |

Network 3: Scale สำหรับ 4-20mA

  |                                                    |
  |--[ALWAYS]------[SCALE]---------------------------|
  |  (4-20mA: 4mA = 0%, 20mA = 100%)                  |
  |                 IN: "Speed_Percent"                |
  |                 LO_LIM: 0.0 (4mA)                  |
  |                 HI_LIM: 100.0 (20mA)               |
  |                 OUT: "AO_Raw"  (5530-27648)        |
  |                 → OUT: QW258                       |
  |                                                    |
```

### รับ Feedback จาก VFD

```
// VFD ส่ง Analog Output → PLC AI
// ตัวอย่าง: Actual Frequency, Current, Status

Network 4: อ่าน Actual Frequency จาก VFD AO

  |                                                    |
  |--[ALWAYS]------[SCALE]---------------------------|
  |                 IN: IW256 (Analog Input from VFD)  |
  |                 LO_LIM: 0.0                        |
  |                 HI_LIM: 60.0 (Hz)                  |
  |                 OUT: "VFD_Actual_Hz"               |
  |                                                    |

Network 5: คำนวณ RPM จาก Hz

  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |                 IN1: "VFD_Actual_Hz"               |
  |                 IN2: 60.0 (Synchronous RPM factor) |
  |                 OUT: "Synchronous_RPM"             |
  |                                                    |

  |--[ALWAYS]------[MUL]------------------------------|
  |  (Apply slip: 1450/1500 = 0.967)                   |
  |                 IN1: "Synchronous_RPM"             |
  |                 IN2: 0.967                         |
  |                 OUT: "Actual_RPM"                  |
  |                                                    |
```

---

## Step 685: PLC to VFD via Modbus RTU

### Modbus Communication Setup

```
Wiring:
PLC RS485 Port ────→ VFD RS485 Port
Pin A (+) ────────→ A+ (Pin 5)
Pin B (-) ────────→ B- (Pin 4)
GND ──────────────→ GND
(Add 120Ω termination resistor at each end)

VFD Modbus Settings:
  Slave Address: 1
  Baud Rate: 9600
  Parity: None
  Data Bits: 8
  Stop Bits: 1
```

### Modbus Register Map (Siemens SINAMICS G120)

```
Control Registers (Write to VFD):
  40001 (300h): Control Word
    Bit 0: ON/OFF1
    Bit 1: OFF2 (Emergency Stop)
    Bit 2: OFF3 (Fast Stop)
    Bit 3: Enable Operation
    Bit 7: Fault Reset
  40002 (301h): Main Setpoint (Speed Reference)
    Value: 0-16383 = 0-100% speed

Status Registers (Read from VFD):
  40101 (3100h): Status Word
    Bit 0: Ready to switch on
    Bit 1: Ready
    Bit 2: Operation enabled
    Bit 3: Fault present
    Bit 10: Setpoint reached
  40102 (3101h): Actual Speed (0-16383 = 0-100%)
  40103 (3102h): Actual Current (0-16383 = 0-200% rated)
  40104 (3103h): DC Bus Voltage
  40105 (3104h): Output Frequency (0.01 Hz per bit)
```

### Ladder - VFD Control via Modbus

```
// DB "VFD_Modbus_DB":
//   Control_Word   : WORD
//   Speed_Reference: INT  (0-16383)
//   Status_Word    : WORD (read back)
//   Actual_Speed   : INT  (read back)
//   Actual_Current : INT  (read back)
//   Comm_Error     : BOOL
//   Comm_Timeout_Counter: INT

Network 1: สร้าง Modbus Control Word

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |  (Set bit ตามสถานะ)                                 |
  |                 IN: W#16#047F (Run + Enable all)   |
  |                 OUT: "VFD_Modbus_DB".Control_Word  |
  |                                                    |
  |--[ ]-----------[OR]------------------------------|
  |  Fault_Reset    IN1: "VFD_Modbus_DB".Control_Word  |
  |  (rising edge)  IN2: W#16#0480 (Add Fault Reset bit)|
  |                 OUT: "VFD_Modbus_DB".Control_Word  |
  |                                                    |

Network 2: แปลง Speed% เป็น Modbus Value

  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |  (163.83 = 16383/100)                              |
  |                 IN1: "Speed_Percent"               |
  |                 IN2: 163.83                        |
  |                 OUT: "VFD_Modbus_DB".Speed_Reference|
  |                                                    |

Network 3: เขียน Control Word และ Speed Reference (Modbus Write)

  |                                                    |
  |--[ALWAYS]------[Modbus_Comm]---------------------|
  |                 MB_HOLD_REG: "VFD_MB_Regs"         |
  |                 REQ: TRUE                          |
  |                 MB_ADDR: 1 (VFD Slave Address)     |
  |                 MODE: 1 (Write)                    |
  |                 DATA_ADDR: 40001                   |
  |                 DATA_LEN: 2                        |
  |                 DATA_PTR: "VFD_Modbus_DB".Ctrl_Word|
  |                 DONE: "Modbus_Write_Done"          |
  |                 ERROR: "Modbus_Error"              |
  |                                                    |

Network 4: อ่านสถานะจาก VFD (Modbus Read)

  |                                                    |
  |--[ ]----------[Modbus_Comm]---------------------|
  |  Write_Done     MB_HOLD_REG: "VFD_MB_Regs"         |
  |               REQ: TRUE                            |
  |               MB_ADDR: 1                           |
  |               MODE: 0 (Read)                       |
  |               DATA_ADDR: 40101                     |
  |               DATA_LEN: 5 (Status + Speed + Curr)  |
  |               DATA_PTR: "VFD_Modbus_DB".Status_Word|
  |               DONE: "Modbus_Read_Done"             |
  |                                                    |
```

---

## Step 686: VFD via PROFIBUS/PROFINET

### PROFINET Communication (Siemens G120 + S7-1200)

```
// Standard Telegram 20 (Speed Control)
// PZD1 out: Control Word (WORD)
// PZD2 out: Speed Setpoint (INT, %)
// PZD1 in: Status Word (WORD)
// PZD2 in: Actual Speed (INT, %)

// ใน TIA Portal:
// 1. Add G120 to PROFINET Network
// 2. Configure Standard Telegram 20
// 3. กำหนด I/O Addresses

// I/O Addresses ตัวอย่าง:
// QW64: Control Word → Drive
// QW66: Speed Setpoint → Drive
// IW64: Status Word ← Drive
// IW66: Actual Speed ← Drive
```

### Ladder - PROFINET VFD Control

```
// Simple Telegram 20 Control

Network 1: กำหนด Control Word

  |                                                    |
  |--[ ]----------[MOVE]------------------------------|
  |  "VFD_Run"     IN: W#16#047F (Run + Enable)       |
  |               OUT: QW64 (Control Word to Drive)   |
  |                                                    |
  |--[NOT]--------[MOVE]------------------------------|
  |  "VFD_Run"     IN: W#16#047E (Disable Operation)  |
  |               OUT: QW64                           |
  |                                                    |

Network 2: Speed Setpoint (0-16383 = 0-100%)

  |                                                    |
  |--[ALWAYS]------[SCALE]---------------------------|
  |                 IN: "Speed_Pct" (0.0-100.0%)      |
  |                 LO_LIM: 0                          |
  |                 HI_LIM: 16383                      |
  |                 OUT: QW66                          |
  |                                                    |

Network 3: อ่านสถานะ

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: IW64 (Status Word from Drive)  |
  |                 OUT: "VFD_Status_Word"             |
  |                                                    |

Network 4: ดึง Status Bits

  |                                                    |
  |--[ALWAYS]------[AND]------------------------------|
  |  (Check bit 2: Operation Enabled)                  |
  |                 IN1: "VFD_Status_Word"             |
  |                 IN2: W#16#0004                     |
  |                 OUT: "VFD_Op_Enabled" (WORD)       |
  |                                                    |
  |--[<>]----------[MOVE]------------------------------|
  |  Status & 4    IN: TRUE                            |
  |  <> 0          OUT: "VFD_Running"                  |
  |                                                    |
```

---

## Step 687: Speed Setpoint Scaling

### การ Scale Speed Reference

```
// ตัวอย่าง: สายพานโรงงาน
// Speed ที่ต้องการ: 0-60 m/min
// VFD: 0-50 Hz = 0-100%
// Motor: 1450 RPM ที่ 50Hz
// Belt: ขับด้วย Roller ø = 200mm

// คำนวณ:
// Belt Speed (m/s) = RPM × π × Diameter / 60
// ที่ 50Hz: 1450 × π × 0.2 / 60 = 15.18 m/s = 910 m/min
// (มากเกินไป ต้องใช้ Gearbox)
// Gearbox Ratio: 15:1
// ที่ 50Hz: 910/15 = 60.7 m/min ≈ 60 m/min ✓

// Scaling Formula:
// Speed_Hz = (Belt_Speed_mpm × 50) / 60.7
// Speed_Percent = (Speed_Hz / 50) × 100

Network 1: แปลง m/min เป็น Hz

  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |                 IN1: "Belt_Speed_mpm"              |
  |                 IN2: 50.0                          |
  |                 OUT: "Temp_Result"                 |
  |                                                    |
  |--[ALWAYS]------[DIV]------------------------------|
  |                 IN1: "Temp_Result"                 |
  |                 IN2: 60.7 (max speed at 50Hz)      |
  |                 OUT: "Freq_Setpoint_Hz"            |
  |                                                    |

Network 2: แปลง Hz เป็น %

  |                                                    |
  |--[ALWAYS]------[DIV]------------------------------|
  |                 IN1: "Freq_Setpoint_Hz"            |
  |                 IN2: 50.0 (max frequency)          |
  |                 OUT: "Speed_Percent"               |
  |                                                    |
```

---

## Step 688: Acceleration/Deceleration Ramp

### การตั้งค่า Ramp ใน VFD

```
VFD Ramp Parameters:
┌─────────────────────────────────────────────────────────┐
│  Ramp-up time: เวลาเพิ่มความเร็ว 0→50Hz                │
│  Ramp-down time: เวลาลดความเร็ว 50→0Hz                 │
│                                                         │
│  ตัวอย่าง Ramp 2 ชุด:                                  │
│  Ramp 1 (ปกติ): Up=10s, Down=15s                       │
│  Ramp 2 (เร่งด่วน): Up=3s, Down=5s                     │
│                                                         │
│  S-curve Ramp:                                          │
│  - เพิ่ม Jerk Smoothing                                │
│  - ลด Mechanical Stress                                 │
│  - ช่วยสำหรับ Belt, Chain Systems                      │
└─────────────────────────────────────────────────────────┘
```

### PLC-Controlled Ramp (Software Ramp)

```
// บางครั้งต้องการ Ramp ที่ยืดหยุ่นกว่า VFD Ramp
// ทำ Ramp ใน PLC แล้วส่ง Stepped Speed Reference ไป VFD

// DB "Ramp_DB":
Current_Speed: REAL := 0.0    // ความเร็วปัจจุบัน
Target_Speed : REAL           // ความเร็วเป้าหมาย
Accel_Rate   : REAL := 5.0    // %/s
Decel_Rate   : REAL := 3.0    // %/s
Ramp_Active  : BOOL

// ใน OB35 (100ms):
Network 1: คำนวณ Speed Increment

  |                                                    |
  |--[ALWAYS]------[SUB]------------------------------|
  |                 IN1: "Target_Speed"                |
  |                 IN2: "Current_Speed"               |
  |                 OUT: "Speed_Error"                 |
  |                                                    |

Network 2: Accelerate

  |                                                    |
  |--[>]--[>]------[ADD]------------------------------|
  |  Error > 0     IN1: "Current_Speed"                |
  |  Error > Accel_Step  IN2: Accel_Step (0.5%/100ms) |
  |               OUT: "Current_Speed"                 |
  |                                                    |

Network 3: ลบ ถ้า Error น้อย

  |                                                    |
  |--[<=]----------[MOVE]------------------------------|
  |  ABS(Error) <= Accel_Step  IN: "Target_Speed"     |
  |                             OUT: "Current_Speed"   |
  |                                                    |

Network 4: Decelerate

  |                                                    |
  |--[<]--[ABS>]---[SUB]------------------------------|
  |  Error < 0    IN1: "Current_Speed"                 |
  |  ABS(Error)>Decel  IN2: Decel_Step (0.3%/100ms)  |
  |               OUT: "Current_Speed"                 |
  |                                                    |
```

---

## Step 689: Energy Saving Calculation

### การคำนวณการประหยัดพลังงาน

```
Fan/Pump Affinity Laws:
- Flow ∝ Speed (N)
- Head ∝ Speed² (N²)
- Power ∝ Speed³ (N³)

ตัวอย่าง:
ที่ 100% Speed: Power = 7.5 kW
ที่ 80% Speed:  Power = 7.5 × (0.8)³ = 7.5 × 0.512 = 3.84 kW
ประหยัดได้: 3.66 kW หรือ 48.8%!

ประหยัดต่อวัน (16 ชั่วโมง):
3.66 kW × 16h = 58.56 kWh/day
ต่อปี: 58.56 × 365 = 21,374 kWh
ค่าไฟ (3.5 บาท/kWh): 74,810 บาท/ปี
```

### Ladder - Energy Monitoring

```
// คำนวณพลังงานที่ใช้ และ Compare กับ Baseline

Network 1: อ่านกำลังไฟ (คำนวณจาก V, I, pf)

  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |  (P = V × I × pf × √3 สำหรับ 3-phase)             |
  |                 IN1: "VFD_Output_Voltage"          |
  |                 IN2: "VFD_Actual_Current"          |
  |                 OUT: "Temp_P"                      |
  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |                 IN1: "Temp_P"                      |
  |                 IN2: 1.732 (√3)                   |
  |                 OUT: "Temp_P2"                     |
  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |                 IN1: "Temp_P2"                     |
  |                 IN2: "Power_Factor"                |
  |                 OUT: "Actual_Power_W"              |
  |                                                    |

Network 2: คำนวณ kWh (ทำงานใน OB32 - 1 second timer)

  |                                                    |
  |--[ ]----------[ADD]------------------------------|
  |  "Motor_Running"  IN1: "kWh_Accumulator"          |
  |                   IN2: Actual_Power_W/3600000      |
  |                   OUT: "kWh_Accumulator"           |
  |                   (หาร 3,600,000 เพื่อแปลง W.s → kWh)|
  |                                                    |

Network 3: เปรียบเทียบกับ Baseline (ไม่มี VFD)

  |                                                    |
  |--[ALWAYS]------[SUB]------------------------------|
  |                 IN1: "Baseline_kWh"               |
  |                 IN2: "kWh_Accumulator"             |
  |                 OUT: "Energy_Saved_kWh"            |
  |                                                    |
```

---

## Step 690: Fault Handling and Reset

### VFD Fault Types

```
Common VFD Faults:
┌──────────────────────────────────────────────────────┐
│  F001: Overcurrent              │ ลดโหลด, ตรวจ Motor│
│  F002: Overvoltage (DC Bus)     │ เพิ่ม Ramp-down   │
│  F003: Undervoltage             │ ตรวจไฟเข้า         │
│  F004: Overtemperature (Drive)  │ ตรวจ Cooling       │
│  F005: Overtemperature (Motor)  │ ลดโหลด             │
│  F006: Motor Stall              │ ตรวจ Mechanical     │
│  F011: Ground Fault             │ ตรวจ Insulation    │
│  F021: Communication Timeout   │ ตรวจสาย/Parameter  │
└──────────────────────────────────────────────────────┘
```

### Fault Handling Ladder

```
// DB "VFD_Fault_DB":
Fault_Active  : BOOL
Fault_Code    : WORD
Fault_Count   : INT
Last_Fault_Time: DATE_AND_TIME
Auto_Reset_Enable: BOOL
Auto_Reset_Count: INT

Network 1: ตรวจสอบ Fault จาก Status Word

  |                                                    |
  |--[AND]--------[SET]------------------------------|
  |  Status_Word   "VFD_Fault_DB".Fault_Active        |
  |  Bit3 (Fault)                                      |
  |                                                    |
  |--[ ]----------[INC]------------------------------|
  |  Fault_Active  "VFD_Fault_DB".Fault_Count         |
  |  (Rising edge)                                     |
  |                                                    |

Network 2: Manual Fault Reset

  |                                                    |
  |--[P]----------[SET]------------------------------|
  |  "HMI_Reset"   Control_Word Bit7 (Fault Reset)    |
  |                                                    |
  |--[TOF]--------[RESET]------------------------------|
  |  Reset_Pulse   Bit7 ปิดหลัง 200ms                |
  |  ET: T#200ms                                       |
  |                                                    |

Network 3: Auto Reset (สำหรับ Minor Faults)

  |                                                    |
  |--[ ]--[ ]--[<]---[TON]---------------------------|
  |  Fault  Auto_  Reset_  T_Auto_Reset               |
  |  Active Enable Count<3  PT: T#10s (รอ 10 วิ)      |
  |                                                    |
  |--[ ]----------[SET]-[INC]-----------------------|
  |  T_Auto_Reset.Q Bit7   Reset_Count               |
  |               (Fault Reset)                       |
  |                                                    |
```

---

## Step 691-695: Multi-Speed Preset System

### ระบบสายพานหลายความเร็ว

```
// ระบบ Conveyor มี 4 ความเร็ว ตาม Product Type
// Product A: 10 m/min
// Product B: 15 m/min  
// Product C: 20 m/min
// Product D: 25 m/min

// DB "Conv_Speed_DB":
Products : ARRAY[1..4] OF STRUCT
  Product_Name  : STRING[20]
  Belt_Speed_mpm: REAL
  VFD_Freq_Hz   : REAL
  VFD_Percent   : REAL
  Accel_Time    : REAL
  Decel_Time    : REAL
END_STRUCT
Active_Product: INT
Actual_Belt_Speed: REAL
Target_Belt_Speed: REAL
```

### Ladder - Product Speed Selection

```
Network 1: อ่าน Speed จาก DB ตาม Product ที่เลือก

  |                                                    |
  |--[P]----------[MOVE]------------------------------|
  |  "Product_    IN: "Conv_Speed_DB"                  |
  |   Changed"        .Products["Active_Product"]      |
  |                        .Belt_Speed_mpm             |
  |               OUT: "Target_Belt_Speed"             |
  |                                                    |

Network 2: อัพเดต VFD Parameters via Modbus

  |                                                    |
  |--[P]----------[Modbus_Write]----------------------|
  |  Product      Write P1120 (Accel Time)             |
  |  Changed      Value: Products[Active].Accel_Time   |
  |                                                    |
  |--[P]----------[Modbus_Write]----------------------|
  |  Product      Write P1121 (Decel Time)             |
  |  Changed      Value: Products[Active].Decel_Time   |
  |                                                    |
```

---

## Step 696-700: Complete VFD Conveyor System

### ระบบสมบูรณ์: Conveyor Speed Control System

```
// ============================================
// COMPLETE CONVEYOR SPEED CONTROL SYSTEM
// PLC: Siemens S7-1200
// VFD: Siemens SINAMICS G120 via PROFINET
// Motor: 7.5 kW, 4P, 1450 RPM
// Belt: 0-60 m/min
// ============================================

// I/O List:
// Digital Inputs:
//   I0.0: Emergency Stop (NC)
//   I0.1: Run Button (NO)
//   I0.2: Stop Button (NC)
//   I0.3: Fault Reset Button
//   I0.4: Speed Up Button
//   I0.5: Speed Down Button
//   I0.6: Part Sensor (Product Detection)
//   I0.7: Belt Slack Switch (NC - หย่อน = OFF)

// Digital Outputs:
//   Q0.0: Ready Indicator
//   Q0.1: Running Indicator
//   Q0.2: Fault Indicator
//   Q0.3: Speed High Warning

// Analog Outputs:
//   QW64: VFD Control Word (PROFINET)
//   QW66: VFD Speed Reference (PROFINET)

// Analog Inputs:
//   IW64: VFD Status Word (PROFINET)
//   IW66: VFD Actual Speed

// PROFINET:
//   VFD at Subnet Address
```

### State Machine สมบูรณ์

```
States:
0 = IDLE (รอคำสั่ง, VFD Disabled)
1 = STARTING (Enable VFD, Ramp Up)
2 = RUNNING (Stable Speed)
3 = SPEED_CHANGING (กำลังเปลี่ยนความเร็ว)
4 = STOPPING (Ramp Down)
5 = FAULT (Error State)

Network 1: Safety Check (ทำงานทุก Scan)

  |                                                    |
  |--[NOT]--------[MOVE]--[MOVE]---------------------|
  |  "EStop"       IN: 5   IN: 0 (Reset Speed)       |
  |  OR             OUT:State OUT: QW66 (Stop VFD)    |
  |  Belt_Slack                                       |
  |                                                    |

Network 2: State 0 → 1 (Start)

  |                                                    |
  |--[=]--[ ]---[MOVE]--------------------------------|
  |  State  Run   IN: 1  (Start)                      |
  |  = 0    Btn   OUT: State                          |
  |  AND                                               |
  |  EStop_OK                                         |
  |                                                    |

Network 3: State 1 - Starting

  |                                                    |
  |--[=]----------[MOVE]------------------------------|
  |  State = 1     IN: W#16#047F (Run Control Word)   |
  |               OUT: QW64                           |
  |                                                    |
  |  (Speed Ramp จะทำงานใน OB35 → ค่อยๆ เพิ่ม QW66)  |
  |                                                    |
  |--[=]--[>=]----[MOVE]------------------------------|
  |  State  Speed  IN: 2 (Running State)              |
  |  = 1    >= 95% OUT: State                         |
  |         of Target                                  |
  |                                                    |

Network 4: State 2 - Running

  |                                                    |
  |--[=]----------[SET]--[SET]------------------------|
  |  State = 2     "Running_Indicator" (Q0.1)          |
  |               "Ready_Indicator" (Q0.0)             |
  |                                                    |

Network 5: Speed Adjustment (Up)

  |                                                    |
  |--[=]--[P]-----[ADD]--[LIM]------------------------|
  |  State  SpeedUp  IN1: Target_Speed                 |
  |  = 2    Btn      IN2: 2.0 (m/min)                  |
  |                  OUT: Target_Speed                 |
  |                  [LIMIT] 0 to 60                  |
  |                                                    |

Network 6: Speed Adjustment (Down)

  |                                                    |
  |--[=]--[P]-----[SUB]--[LIM]------------------------|
  |  State  SpeedDn  IN1: Target_Speed                 |
  |  = 2    Btn      IN2: 2.0 (m/min)                  |
  |                  OUT: Target_Speed                 |
  |                                                    |

Network 7: Stop Command

  |                                                    |
  |--[=]--[ ]-----[MOVE]------------------------------|
  |  State  Stop   IN: 4 (Stopping State)             |
  |  = 2    Btn    OUT: State                         |
  |                                                    |

Network 8: State 4 - Stopping (Ramp Down)

  |                                                    |
  |--[=]----------[MOVE]------------------------------|
  |  State = 4     IN: 0.0 (Target = 0)               |
  |               OUT: Target_Speed                   |
  |               (OB35 Ramp will reduce speed)        |
  |                                                    |
  |--[=]--[<=]----[MOVE]--[MOVE]---------------------|
  |  State  Speed  IN: W#16#047E (Disable)            |
  |  = 4    <= 2%  OUT: QW64                          |
  |               IN: 0                               |
  |                OUT: State (Back to IDLE)           |
  |                                                    |

Network 9: Fault Handling

  |                                                    |
  |--[AND]-[NOT]--[MOVE]--[SET]----------------------|
  |  Status  Fault  IN: 5  "Fault_Indicator"          |
  |  Bit3    ACK    OUT: State                        |
  |                                                    |
```

---

## Lab Projects

### Lab 1: VFD Commissioning and Calibration

**วัตถุประสงค์:** Commissioning VFD และตั้งค่า Parameters

**ขั้นตอน:**
1. ต่อวงจร VFD กับมอเตอร์
2. กำหนด Motor Parameters จาก Nameplate
3. รัน Auto-tune (Motor Data Identification)
4. ทดสอบ Run/Stop ด้วย Panel
5. ตั้งค่า Communication Parameters
6. ทดสอบควบคุมจาก PLC
7. วัด Actual Speed เทียบกับ Reference

### Lab 2: Conveyor System with Variable Speed

**วัตถุประสงค์:** สร้างระบบสายพานควบคุมความเร็วแบบ Full System

**โจทย์:**
```
1. ควบคุมความเร็ว 0-50 m/min
2. Soft Start (Ramp 10 วินาที)
3. Soft Stop (Ramp 15 วินาที)
4. Emergency Stop (ทันที)
5. Speed Display บน HMI
6. Fault Display และ Reset
7. Energy Monitoring (kWh)
8. Production Counter
```

### Lab 3: Multi-Zone Temperature Control with VFD Fans

```
โจทย์: ระบบ HVAC ควบคุม 3 Zone
- Zone 1: Server Room (อุณหภูมิ 20°C)
- Zone 2: Office (อุณหภูมิ 25°C)
- Zone 3: Warehouse (อุณหภูมิ 28°C)
- Fan speed ปรับตาม Temperature Error (PID)
- Energy Saving: ลด Fan Speed เมื่อ Error น้อย
```

---

## สรุป Part 35

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| VFD Principles | V/f, Vector Control |
| Parameter Setup | Motor Data, Protection |
| Digital Control | Multi-Speed Preset |
| Analog Control | 0-10V, 4-20mA |
| Modbus RTU | VFD Register Mapping |
| PROFINET | Telegram 20 |
| Ramp Control | Accel/Decel, Software Ramp |
| Energy Saving | Affinity Laws Calculation |
| Fault Handling | Auto Reset, Manual Reset |

---

## การนำทาง

- [← Part 34: Servo Drive Control](part34_servo_drive.md)
- [→ Part 36: SCADA Integration](part36_scada_integration.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 35 of 40 | Steps 681-700*
