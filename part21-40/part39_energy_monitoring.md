# Part 39: Energy Monitoring
## Steps 761-780: การติดตามและจัดการพลังงานในโรงงาน

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 3-4 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 25 (Modbus), Part 36 (SCADA Integration)

---

## บทนำ

Energy Monitoring คือการติดตามและวิเคราะห์การใช้พลังงานในโรงงาน เพื่อระบุโอกาสประหยัดพลังงาน ลดต้นทุน และปฏิบัติตามมาตรฐาน ISO 50001

ผลประโยชน์:
- ลดค่าใช้จ่ายพลังงาน 10-30%
- ปฏิบัติตามกฎหมายพลังงาน
- ลด Carbon Footprint
- รับ ISO 50001 Certification
- ข้อมูลสำหรับ Capital Investment

---

## Step 761: Power Monitoring Basics

### พื้นฐานระบบไฟฟ้า 3 เฟส

```
┌────────────────────────────────────────────────────────────┐
│               3-Phase Power System                         │
│                                                            │
│  Line-to-Line (L-L): 380V (ไทย), 400V (EU), 480V (US)    │
│  Line-to-Neutral (L-N): 220V (380/√3)                     │
│  Frequency: 50 Hz (ไทย), 60 Hz (US)                      │
│                                                            │
│  Power Formulas:                                           │
│  Active Power (P) = √3 × VL × IL × cos(φ) [kW]           │
│  Reactive Power (Q) = √3 × VL × IL × sin(φ) [kVAR]       │
│  Apparent Power (S) = √3 × VL × IL [kVA]                  │
│  Power Factor (PF) = cos(φ) = P/S                         │
│                                                            │
│  Energy = Power × Time [kWh]                               │
│  1 kWh = 1,000 Wh = 3,600,000 J                           │
│                                                            │
│  Demand = Max kW ใน 15-30 นาที (Billing Demand)           │
└────────────────────────────────────────────────────────────┘
```

### ประเภท Power Meters

```
Basic kWh Meter:
- อ่านค่า kWh เท่านั้น
- ราคาถูก
- ใช้งานง่าย

Smart Power Meter:
- อ่านค่า: V, I, P, Q, PF, kWh, kVARh, THD
- สื่อสารได้: Modbus, PROFIBUS, Ethernet
- Logging ในตัว
- ตัวอย่าง: Siemens SENTRON PAC3200, Schneider PM3255

Power Quality Analyzer:
- วัด Harmonics, Flicker, Transients
- ตรวจจับปัญหาคุณภาพไฟฟ้า
- ตัวอย่าง: Fluke 435, Yokogawa CW500
```

---

## Step 762: kWh Energy Meter Integration via Modbus

### Smart Meter Wiring

```
Wiring Diagram:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  3-Phase Power Bus                                      │
│  L1 ──────────────→ CT1 ──→ Smart Meter Input L1       │
│  L2 ──────────────→ CT2 ──→ Smart Meter Input L2       │
│  L3 ──────────────→ CT3 ──→ Smart Meter Input L3       │
│  N  ──────────────────────→ Smart Meter Neutral        │
│                                                         │
│  Smart Meter                                            │
│  ├── RS485 A (+) ──────────────→ PLC RS485 A           │
│  └── RS485 B (-) ──────────────→ PLC RS485 B           │
│      GND ──────────────────────→ PLC GND               │
│                                                         │
│  Current Transformer (CT):                             │
│  - 100A/5A หรือ 200A/5A หรือ 400A/5A                  │
│  - เลือกตาม Load ที่คาดว่าสูงสุด                        │
└─────────────────────────────────────────────────────────┘
```

### Modbus Register Map (Siemens PAC3200)

```
Siemens SENTRON PAC3200 Modbus Map:

Input Registers (Function Code 4):
Register │ Variable         │ Type    │ Unit    │ Factor
---------|------------------|---------|---------|---------
0x0000  │ Voltage L1-N     │ FLOAT32 │ V       │ 1
0x0002  │ Voltage L2-N     │ FLOAT32 │ V       │ 1
0x0004  │ Voltage L3-N     │ FLOAT32 │ V       │ 1
0x0006  │ Voltage L1-L2    │ FLOAT32 │ V       │ 1
0x0008  │ Voltage L2-L3    │ FLOAT32 │ V       │ 1
0x000A  │ Voltage L3-L1    │ FLOAT32 │ V       │ 1
0x000C  │ Current L1       │ FLOAT32 │ A       │ 1
0x000E  │ Current L2       │ FLOAT32 │ A       │ 1
0x0010  │ Current L3       │ FLOAT32 │ A       │ 1
0x0012  │ Active Power L1  │ FLOAT32 │ kW      │ 1
0x0014  │ Active Power L2  │ FLOAT32 │ kW      │ 1
0x0016  │ Active Power L3  │ FLOAT32 │ kW      │ 1
0x0018  │ Active Power Tot │ FLOAT32 │ kW      │ 1
0x001A  │ Reactive Power   │ FLOAT32 │ kVAR    │ 1
0x001C  │ Apparent Power   │ FLOAT32 │ kVA     │ 1
0x001E  │ Power Factor     │ FLOAT32 │ -       │ 1
0x0020  │ Frequency        │ FLOAT32 │ Hz      │ 1
0x0028  │ Energy Import    │ FLOAT32 │ kWh     │ 1
0x002A  │ Energy Export    │ FLOAT32 │ kWh     │ 1
```

### Ladder - อ่านค่าจาก Smart Meter

```
// DB "Energy_Meter_DB":
//   Voltage_L1, L2, L3 : REAL
//   Current_L1, L2, L3 : REAL
//   Power_Total         : REAL (kW)
//   Power_Factor        : REAL
//   Energy_kWh          : REAL (Accumulated)
//   Frequency           : REAL
//   Demand_Current      : REAL (kW in last 15min)

// Network: อ่านข้อมูลจาก Smart Meter ทุก 1 วินาที

Network 1: Trigger อ่านข้อมูล (ใน OB32 - 1 second)

  |                                                         |
  |--[ALWAYS]------[Modbus_Master]--------------------------|
  |                 MB_ADDR: 1 (Meter Address)              |
  |                 MODE: 2 (Read Input Registers)          |
  |                 DATA_ADDR: 40001 (Register 0x0000)      |
  |                 DATA_LEN: 50 (อ่าน 50 registers)        |
  |                 DATA_PTR: "Meter_Raw_DB".Raw_Data       |
  |                 DONE: "Meter_Read_Done"                 |
  |                 ERROR: "Meter_Comm_Error"               |
  |                                                         |

Network 2: แปลงค่า (เมื่อ Read Done)

  |                                                         |
  |--[ ]----------[MOVE]-----------------------------------|
  |  Read_Done     IN: "Meter_Raw_DB".Raw_Data[0]          |
  |               OUT: "Energy_Meter_DB".Voltage_L1         |
  |               (Float32 = 2 registers combined)          |
  |                                                         |
  |              [MOVE]                                     |
  |               IN: "Meter_Raw_DB".Raw_Data[12]          |
  |               OUT: "Energy_Meter_DB".Current_L1         |
  |                                                         |
  |              [MOVE]                                     |
  |               IN: "Meter_Raw_DB".Raw_Data[24]          |
  |               OUT: "Energy_Meter_DB".Power_Total        |
  |                                                         |
  |              [MOVE]                                     |
  |               IN: "Meter_Raw_DB".Raw_Data[40]          |
  |               OUT: "Energy_Meter_DB".Energy_kWh         |
  |                                                         |
```

---

## Step 763: Current Transformer Setup

### การเลือก CT (Current Transformer)

```
CT Selection:
┌──────────────────────────────────────────────────────────┐
│  ขั้นตอนการเลือก CT:                                     │
│                                                          │
│  1. วัด Peak Current ของ Load                           │
│     ตัวอย่าง: Motor 30kW, 380V → I = 57A               │
│     CT ควรใหญ่กว่า 120-150%: เลือก 75A/5A              │
│                                                          │
│  2. Secondary Current: 5A (Standard) หรือ 1A (Long wire)│
│                                                          │
│  3. Accuracy Class:                                      │
│     Class 0.2S: สำหรับ Revenue Metering                │
│     Class 0.5: สำหรับ Power Meter                       │
│     Class 1: สำหรับ Process Monitoring                   │
│     Class 3: สำหรับ Protection Relay                    │
│                                                          │
│  4. Burden (VA): คำนวณจาก Wire Length                   │
│     VA = I² × R_wire = 5² × 0.4Ω = 10VA                │
│     เลือก CT Burden ≥ 10VA                               │
└──────────────────────────────────────────────────────────┘
```

### CT Installation

```
CT ต้องต่ออย่างถูกต้อง:
┌─────────────────────────────────────────────────────────┐
│  ⚠️ ห้ามเปิดวงจร CT Secondary เมื่อ Primary มีกระแส!   │
│  (จะเกิดแรงดันสูงมาก อาจเสียชีวิต)                     │
│                                                         │
│  Polarity Mark (P1/K1):                                 │
│  - ต้องชี้ไปทาง Source (จาก Generator ไป Load)          │
│  - ถ้าต่อผิด: Power Reading จะเป็น Negative             │
│                                                         │
│  Safety Feature: CT Short-circuit Terminal              │
│  - ต่อ CT ก่อนต่อ Burden (Meter)                       │
│  - ถอด Burden ต้อง Short ก่อน                          │
└─────────────────────────────────────────────────────────┘
```

---

## Step 764: Power Factor Correction

### Power Factor Basics

```
Power Factor (PF) = Active Power (kW) / Apparent Power (kVA)
                  = cos(φ)

PF = 1.0: อุดมคติ (Resistive Load เท่านั้น)
PF = 0.9: ดี (มอเตอร์โหลดเต็ม)
PF = 0.8: พอใช้ (มอเตอร์โหลดต่ำ)
PF < 0.7: ไม่ดี (ค่าไฟสูง, ต้องแก้ไข)

ผลเสียของ PF ต่ำ:
- การไฟฟ้าเก็บค่า Reactive Demand (kVAR Charge)
- กระแสสูงขึ้น → สูญเสียในสาย (I²R losses)
- ต้องใช้ Transformer และ Cable ขนาดใหญ่ขึ้น
```

### Power Factor Correction with Capacitor Banks

```
// ระบบ Automatic PF Correction
// วัด PF ทุก 15 วินาที
// เพิ่ม/ลด Capacitor Steps อัตโนมัติ

// DB "PFC_Control":
Target_PF     : REAL := 0.95  // เป้าหมาย PF
Actual_PF     : REAL          // PF ปัจจุบัน
Reactive_Power: REAL          // kVAR ที่ต้องชดเชย
Cap_Steps     : INT := 0      // Capacitor Steps ที่เปิด (0-8)
Cap_kVAR      : ARRAY[1..8] OF REAL  // ขนาดแต่ละ Step (kVAR)
// Step 1: 25 kVAR
// Step 2: 50 kVAR
// Step 3: 100 kVAR
// ...

// Ladder - PF Correction Control:
Network 1: คำนวณ kVAR ที่ขาด

  |                                                         |
  |--[<]----------[FC_Calc_Reactive]------------------------|
  |  Actual_PF     P_kW: "Energy_Meter_DB".Power_Total     |
  |  < Target_PF   Target_PF: 0.95                         |
  |               Required_kVAR: "Req_Compensation"        |
  |                                                         |

// FC_Calc_Reactive (SCL):
// Q_required = P × (tan(arccos(PF_current)) - tan(arccos(PF_target)))
// Q_required = P_kW × (SIN(ACOS(PF_current))/PF_current 
//                      - TAN(ACOS(PF_target)))

Network 2: เพิ่ม Capacitor Step

  |                                                         |
  |--[>=]--[<]----[INC]-[SET]------------------------------|
  |  Req_kVAR      Cap_Steps < 8  Cap_Steps               |
  |  > 0           (ยังมี Step)   Q0_Capacitor_Bank        |
  |                               (เปิด Capacitor)         |
  |                                                         |

Network 3: ลด Capacitor Step (Over-compensate)

  |                                                         |
  |--[<]--[>]-----[DEC]-[RESET]----------------------------|
  |  Req_kVAR      Cap_Steps > 0  Cap_Steps               |
  |  < -5          (มี Step เปิดอยู่)  Q0_Capacitor_Bank   |
  |                                                         |
```

---

## Step 765: Demand Management

### Demand คืออะไร

```
Electricity Demand = กำลังไฟสูงสุดที่ใช้ในช่วงเวลา 15-30 นาที
การไฟฟ้าคิดค่า Demand Charge แยกจาก Energy Charge

ตัวอย่างค่าไฟ:
- Energy Charge: 3.50 บาท/kWh
- Demand Charge: 160 บาท/kW/เดือน (Peak Demand)
- ถ้า Peak Demand = 500 kW → 500 × 160 = 80,000 บาท/เดือน

วิธีลด Demand:
1. ไม่เปิดอุปกรณ์ทุกชิ้นพร้อมกัน
2. ใช้ Soft Starter/VFD (ลด Inrush Current)
3. Load Shifting (เลื่อนงานไม่เร่งด่วน)
4. Peak Shaving (ใช้ Battery หรือ Generator เสริม)
5. Smart Scheduling: เปิดอุปกรณ์สลับกัน
```

### Demand Limiting Program

```
// DB "Demand_DB":
Demand_Limit_kW  : REAL := 450.0  // ขีดจำกัด kW
Current_Power_kW : REAL           // กำลังปัจจุบัน
Peak_Demand_kW   : REAL           // Peak Demand เดือนนี้
Demand_15min_kW  : REAL           // Average ใน 15 นาที

// Non-essential Loads (สามารถ Shed ได้)
Load_Priority_1  : BOOL           // โหลดที่ Shed ได้ก่อน
Load_Priority_2  : BOOL           // โหลดที่ Shed ได้รอง
Load_Priority_3  : BOOL           // โหลดที่ Shed ได้ล่าสุด

// Ladder - Demand Limiting:
Network 1: คำนวณ Average Demand 15 นาที (ใน OB32 - 1 second)

  |                                                         |
  |--[ALWAYS]------[ADD]-----------------------------------|
  |  (สะสม Power ทุก 1 วินาที)                              |
  |                 IN1: "Demand_DB".Power_Sum_15min       |
  |                 IN2: "Energy_Meter_DB".Power_Total     |
  |                 OUT: "Demand_DB".Power_Sum_15min       |
  |                                                         |

Network 2: อัพเดต Average ทุก 15 นาที

  |                                                         |
  |--[CTU]--------[R]-------------------------------------|
  |  Count_Secs    Counter >= 900  (15min × 60s = 900)     |
  |  PV: 900       Reset Counter                           |
  |                                                         |
  |--[ ]----------[DIV]-------------------------------------|
  |  Counter.Q     IN1: Power_Sum_15min                   |
  |               IN2: 900 (วินาที)                        |
  |               OUT: "Demand_DB".Demand_15min_kW        |
  |                                                         |
  |              [MOVE]                                     |
  |               IN: 0                                    |
  |               OUT: Power_Sum_15min (Reset Sum)         |
  |                                                         |

Network 3: Demand Limiting - Shed Load Priority 1

  |                                                         |
  |--[>]----------[RESET]----------------------------------|
  |  Current_Power  "Non_Essen_Load_1"                     |
  |  > Demand_Limit                                         |
  |                (ปิด Load ที่สำคัญน้อยที่สุด)             |
  |                                                         |

Network 4: Restore Load เมื่อ Power ลดลง

  |                                                         |
  |--[<]----------[SET]-------------------------------------|
  |  Current_Power  "Non_Essen_Load_1"                     |
  |  < Demand_Limit × 0.9  (ต้องต่ำกว่า 90% จึง Restore)   |
  |                                                         |
```

---

## Step 766: Energy Data Logging

### Logging System

```
// DB "Energy_Log":
Log_Interval    : INT := 15    // บันทึกทุกกี่นาที
Log_Count       : INT := 0     // จำนวน Record
Log_Max         : INT := 10000 // Records สูงสุด
Write_Index     : INT := 0     // ตำแหน่งเขียน

Log_Data : ARRAY[1..10000] OF STRUCT
  Timestamp     : DATE_AND_TIME
  kWh_Import    : REAL          // พลังงานนำเข้า
  kWh_Export    : REAL          // พลังงานส่งออก
  kW_Active     : REAL          // กำลังจริง
  kVAR_Reactive : REAL          // กำลังรีแอคทีฟ
  Power_Factor  : REAL          // ค่า PF
  Voltage_Avg   : REAL          // แรงดันเฉลี่ย
  Current_Avg   : REAL          // กระแสเฉลี่ย
  Demand_kW     : REAL          // Demand ใน period นี้
END_STRUCT
```

### Ladder - Energy Logger (OB32 - 1 Second + Counter)

```
Network 1: นับวินาที

  |                                                         |
  |--[ALWAYS]------[INC]-----------------------------------|
  |                 "Log_Seconds_Counter"                   |
  |                                                         |

Network 2: ตรวจสอบว่าถึงเวลาบันทึก (ทุก 15 นาที = 900 วินาที)

  |                                                         |
  |--[>=]---------[MOVE]-----------------------------------|
  |  Log_Counter   IN: 0 (Reset Counter)                  |
  |  >= 900        OUT: "Log_Seconds_Counter"              |
  |                                                         |
  |              [MOVE]  // เพิ่ม Write Index              |
  |               IN1: Write_Index + 1                     |
  |               OUT: Write_Index                         |
  |               (Circular: wrap at Log_Max)              |
  |                                                         |
  |              [RD_SYS_T]  // บันทึก Timestamp          |
  |               OUT: Log_Data[Write_Index].Timestamp     |
  |                                                         |
  |              [MOVE]  // บันทึก Energy                  |
  |               IN: "Energy_Meter_DB".Energy_kWh         |
  |               OUT: Log_Data[Write_Index].kWh_Import    |
  |                                                         |
  |              [MOVE]  // บันทึก Power                   |
  |               IN: "Energy_Meter_DB".Power_Total        |
  |               OUT: Log_Data[Write_Index].kW_Active     |
  |                                                         |
  |              [MOVE]  // บันทึก PF                      |
  |               IN: "Energy_Meter_DB".Power_Factor       |
  |               OUT: Log_Data[Write_Index].Power_Factor  |
  |                                                         |
```

---

## Step 767: OEE and Energy KPIs

### OEE คืออะไร

```
OEE (Overall Equipment Effectiveness) = A × P × Q

A = Availability = Run Time / Available Time
P = Performance = Actual Rate / Ideal Rate
Q = Quality = Good Parts / Total Parts

ตัวอย่าง:
Available Time = 480 min (8 hour shift)
Planned Downtime = 60 min (break, lunch)
Net Available = 420 min

Run Time = 370 min (หยุด 50 min = Unplanned Downtime)
A = 370/420 = 88%

Ideal Rate = 60 pcs/min
Actual Parts = 19,500 pcs
P = 19,500/(370×60) = 87.8%

Good Parts = 18,900
Q = 18,900/19,500 = 96.9%

OEE = 0.88 × 0.878 × 0.969 = 74.9%
```

### Energy KPIs

```
Energy Intensity:
  = kWh / Unit Production
  ตัวอย่าง: 2.5 kWh/kg product

Specific Energy Consumption:
  = Energy Used / Output
  เปรียบเทียบกับ Baseline

Energy Cost per Unit:
  = Total Energy Cost / Production Units

Carbon Footprint:
  = kWh × Emission Factor
  ไทย: 0.5182 kgCO2/kWh (MEA/PEA)

Load Factor:
  = Average Demand / Peak Demand
  (สูงดี = ใช้ไฟสม่ำเสมอ)
```

### OEE Ladder Calculation

```
// DB "OEE_DB":
Shift_Duration_min: REAL := 480.0  // เวลากะ (นาที)
Planned_DT_min    : REAL := 60.0   // Planned Downtime
Net_Available_min : REAL           // = Duration - Planned
Run_Time_min      : REAL           // เวลาจริงที่รัน
Downtime_min      : REAL           // Unplanned Downtime

Ideal_Rate        : REAL           // ชิ้น/นาที (Ideal)
Total_Parts       : DINT           // ชิ้นทั้งหมด
Good_Parts        : DINT           // ชิ้นดี
Reject_Parts      : DINT           // ชิ้นเสีย

OEE_Availability  : REAL
OEE_Performance   : REAL
OEE_Quality       : REAL
OEE_Total         : REAL

// ใน OB35 (100ms) สะสมข้อมูล:
Network 1: สะสมเวลา Run (0.1 วินาที = 0.0017 นาที)

  |                                                         |
  |--[ ]----------[ADD]-------------------------------------|
  |  Machine_Run   IN1: Run_Time_min                        |
  |               IN2: 0.0017 (0.1s in minutes)            |
  |               OUT: Run_Time_min                        |
  |                                                         |

// คำนวณ OEE ทุก 1 นาที:
Network 2: คำนวณ Availability

  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |                 IN1: Run_Time_min                       |
  |                 IN2: Net_Available_min                  |
  |                 OUT: OEE_Availability                   |
  |                                                         |

Network 3: คำนวณ Performance

  |                                                         |
  |--[ALWAYS]------[MUL]-----------------------------------|
  |  (Actual Rate = Total_Parts / Run_Time_min)             |
  |                 IN1: Total_Parts (DINT to REAL)         |
  |                 IN2: 1.0 (convert)                     |
  |                 OUT: Temp_Parts_Real                   |
  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |                 IN1: Temp_Parts_Real                    |
  |                 IN2: Run_Time_min × Ideal_Rate          |
  |                 OUT: OEE_Performance                    |
  |                                                         |

Network 4: คำนวณ Quality

  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |                 IN1: Good_Parts (as REAL)               |
  |                 IN2: Total_Parts (as REAL)              |
  |                 OUT: OEE_Quality                        |
  |                                                         |

Network 5: คำนวณ OEE รวม

  |                                                         |
  |--[ALWAYS]------[MUL]-----------------------------------|
  |                 IN1: OEE_Availability                   |
  |                 IN2: OEE_Performance                    |
  |                 OUT: "Temp_OEE"                         |
  |                                                         |
  |--[ALWAYS]------[MUL]-----------------------------------|
  |                 IN1: Temp_OEE                          |
  |                 IN2: OEE_Quality                        |
  |                 OUT: OEE_Total                          |
  |                                                         |
```

---

## Step 768: ISO 50001 Basics

### หลักการ ISO 50001

```
ISO 50001:2018 - Energy Management Systems

PDCA Cycle สำหรับ Energy:
┌─────────────────────────────────────────────────────────┐
│  PLAN:                                                  │
│  ├── กำหนด Energy Policy                               │
│  ├── Energy Review (ดูสถานะปัจจุบัน)                   │
│  ├── Energy Baseline                                    │
│  ├── Energy Performance Indicators (EnPIs)             │
│  └── Energy Objectives & Targets                       │
│                                                         │
│  DO:                                                    │
│  ├── Operational Control                               │
│  ├── Energy Monitoring & Measurement                   │
│  └── Training & Awareness                              │
│                                                         │
│  CHECK:                                                 │
│  ├── Internal Audit                                    │
│  ├── Management Review                                 │
│  └── Compare vs Baseline                               │
│                                                         │
│  ACT:                                                   │
│  ├── Corrective Actions                                │
│  └── Continual Improvement                             │
└─────────────────────────────────────────────────────────┘
```

### EnPI (Energy Performance Indicators)

```
ตัวอย่าง EnPIs สำหรับโรงงาน:

เป้าหมาย: ลดพลังงาน 5% ต่อปีจาก Baseline (2023)

EnPI 1: Energy Intensity
  = kWh / kg Product
  Baseline 2023: 2.8 kWh/kg
  Target 2024: 2.66 kWh/kg (-5%)
  
EnPI 2: Peak Demand
  Baseline: 500 kW
  Target: 475 kW (-5%)
  
EnPI 3: Power Factor
  Baseline: 0.87
  Target: > 0.95

EnPI 4: Annual Energy Cost
  Baseline: 5,000,000 THB
  Target: 4,750,000 THB
```

---

## Step 769: Energy Dashboard in SCADA

### SCADA Energy Screen Design

```
Energy Dashboard Layout:
┌──────────────────────────────────────────────────────────┐
│                   ENERGY DASHBOARD                       │
├──────────────┬───────────────┬──────────────────────────┤
│ CURRENT      │ TODAY         │ THIS MONTH               │
│ 350 kW       │ 4,200 kWh     │ 85,000 kWh               │
│ PF: 0.94     │ Cost: 14,700  │ Cost: 297,500            │
│ Demand: 420kW│ Peak: 450kW   │ Peak Demand: 475kW       │
├──────────────┴───────────────┴──────────────────────────┤
│ ENERGY TREND (Last 24h)                                 │
│  kW │                                                   │
│ 500 ├─────────────────────────────────────────────────  │
│ 400 ├──────╮  ╭──────╮    ╭─────────────────            │
│ 300 ├──────╯──╯      ╰────╯                             │
│ 200 ├─────────────────────────────────────────────────  │
│ 100 ├─────────────────────────────────────────────────  │
│   0 └─────────────────────────────────────────────────→ │
│     00:00   06:00   12:00   18:00   24:00               │
├──────────────────────┬───────────────────────────────────┤
│ POWER FACTOR TREND   │ OEE & ENERGY KPI                 │
│ 1.0 ├───────────────  │ OEE: 74.9%                      │
│ 0.9 ├─────────────    │ Energy/Unit: 2.72 kWh/kg        │
│ 0.8 ├─────────        │ vs Target: 2.66 kWh/kg          │
│     └──────────────   │ Status: ▲ 2.3% above target     │
└──────────────────────┴───────────────────────────────────┘
```

---

## Step 770: Peak Demand Limiting

### Advanced Demand Control

```
// Peak Shaving ด้วย Load Shedding
// เมื่อ Demand เข้าใกล้ Limit → ปิดโหลดที่ไม่จำเป็น

// Priority ของ Loads:
// P1 (Must Run): Safety Equipment, Control Systems
// P2 (Should Run): Production Lines
// P3 (Can Shed): HVAC, Lighting (Non-critical)
// P4 (Can Shed): Water Heating, Compressed Air Backup

// DB "Load_Shed_DB":
Shed_Limit_kW     : REAL := 450  // เริ่ม Shed ที่นี่
Critical_Limit_kW : REAL := 480  // Critical level
Current_kW        : REAL
Load_P3_Status    : BOOL[1..5]   // สถานะ P3 Loads
Load_P4_Status    : BOOL[1..3]   // สถานะ P4 Loads

// Ladder - Demand Control:
Network 1: ระดับ 1 - เตือนเมื่อใกล้ Limit

  |                                                         |
  |--[>=]---------[SET]-------------------------------------|
  |  Current_kW    "Demand_Warning"                         |
  |  >= Shed_Limit × 0.90                                   |
  |  (90% ของ Limit = 405 kW)                               |
  |                                                         |

Network 2: ระดับ 2 - Shed P4 Loads

  |                                                         |
  |--[>=]---------[RESET]----------------------------------|
  |  Current_kW    "Load_P4_Enable"                         |
  |  >= Shed_Limit × 0.95                                   |
  |  (95% = 427.5 kW)                                       |
  |                                                         |

Network 3: ระดับ 3 - Shed P3 Loads

  |                                                         |
  |--[>=]---------[RESET]----------------------------------|
  |  Current_kW    "Load_P3_Enable"                         |
  |  >= Shed_Limit                                          |
  |  (100% = 450 kW)                                        |
  |                                                         |

Network 4: Restore เมื่อกลับสู่ปกติ

  |                                                         |
  |--[<]----------[SET]-------------------------------------|
  |  Current_kW    "Load_P3_Enable"                         |
  |  < Shed_Limit × 0.85                                    |
  |  (85% = 382.5 kW) (Hysteresis)                         |
  |                                                         |
```

---

## Step 771-780: Complete Energy Monitoring System

### ระบบสมบูรณ์สำหรับโรงงาน

```
// ============================================
// COMPLETE ENERGY MONITORING SYSTEM
// ============================================
// Hardware:
// - 3x Siemens PAC3200 Power Meters (Modbus)
//   Meter 1: Main Incomer (ทั้งโรงงาน)
//   Meter 2: Production Area
//   Meter 3: Utilities (HVAC, Lighting)
// - 1x Siemens S7-1200 PLC
// - Ignition SCADA
// ============================================

// Data Points per Meter:
// - Active Power (kW)
// - Reactive Power (kVAR)
// - Power Factor
// - Voltage (L-L, L-N)
// - Current (L1, L2, L3)
// - Energy (kWh accumulated)
// - Frequency

// Report Structure:
// 15-min Intervals: Power, Demand, PF
// Hourly: kWh, Energy Cost
// Daily: Peak Demand, OEE, Energy/Unit
// Monthly: Trend, Budget vs Actual
```

### Energy Report Generation

```python
# Python Script (Ignition) - Energy Report

def generateEnergyReport(period, startDate, endDate):
    """
    สร้าง Energy Report
    period: 'daily', 'monthly'
    """
    
    # ดึงข้อมูลจาก Historian
    query = """
        SELECT 
            DATE_FORMAT(timestamp, '%Y-%m-%d %H:00') as period,
            AVG(active_power_kw) as avg_power_kw,
            MAX(active_power_kw) as peak_power_kw,
            AVG(power_factor) as avg_pf,
            SUM(energy_kwh_increment) as kwh_sum,
            SUM(production_kg) as production_sum
        FROM energy_log
        WHERE timestamp BETWEEN ? AND ?
        GROUP BY DATE_FORMAT(timestamp, '%Y-%m-%d %H:00')
        ORDER BY period
    """
    data = system.db.runPrepQuery(query, [startDate, endDate])
    
    # คำนวณ KPIs
    total_kwh = sum([row['kwh_sum'] for row in data])
    total_production = sum([row['production_sum'] for row in data])
    
    kpi_intensity = total_kwh / total_production if total_production > 0 else 0
    peak_demand = max([row['peak_power_kw'] for row in data])
    avg_pf = sum([row['avg_pf'] for row in data]) / len(data)
    
    # ต้นทุนพลังงาน (บาท)
    energy_cost = total_kwh * 3.50  # 3.50 บาท/kWh
    demand_cost = peak_demand * 160  # 160 บาท/kW/เดือน
    total_cost = energy_cost + demand_cost
    
    # สร้าง Report
    params = {
        "period": period,
        "start_date": startDate,
        "end_date": endDate,
        "total_kwh": round(total_kwh, 1),
        "peak_demand_kw": round(peak_demand, 1),
        "avg_pf": round(avg_pf, 3),
        "energy_intensity": round(kpi_intensity, 3),
        "energy_cost": round(energy_cost, 0),
        "demand_cost": round(demand_cost, 0),
        "total_cost": round(total_cost, 0),
        "chart_data": data
    }
    
    # Generate PDF
    pdf_bytes = system.report.executeReport(
        "Energy_Report_Template",
        "Production",
        params,
        "PDF"
    )
    
    return pdf_bytes
```

---

## Lab Projects

### Lab 1: Smart Meter Integration

**วัตถุประสงค์:** เชื่อมต่อ Smart Meter และอ่านข้อมูล

**ขั้นตอน:**
1. ต่อ Modbus RS485 กับ Smart Meter (จำลองด้วย Software)
2. อ่านค่า Voltage, Current, Power, Energy
3. แสดงผลบน HMI
4. Alarm เมื่อ PF < 0.9

### Lab 2: Demand Management System

**วัตถุประสงค์:** สร้างระบบควบคุม Demand

```
โจทย์:
- Demand Limit: 400 kW
- Non-essential Loads: 
  - Compressed Air Backup (50 kW)
  - Water Heating (30 kW)
  - Lighting Non-critical (20 kW)

ต้องการ:
1. Monitor Current Demand (15-min rolling average)
2. Shed Load เมื่อ > 380 kW
3. Restore เมื่อ < 330 kW
4. Alert เมื่อ Near Limit
5. Log Demand Events
```

### Lab 3: Energy Dashboard + Report

```
โจทย์: สร้าง Energy Monitoring System ครบวงจร

1. PLC: อ่านข้อมูลจาก 3 Smart Meters
2. SCADA: Energy Dashboard แสดง:
   - Real-time Power, PF, Voltage
   - 24h Energy Trend
   - OEE (ถ้ามีข้อมูล Production)
   - Energy Cost (Real-time estimate)

3. Database: บันทึกทุก 15 นาที

4. Reports:
   - Daily Summary (Email อัตโนมัติตอนเช้า)
   - Monthly Energy Report
   - Carbon Footprint Report (kgCO2)

5. Alerts:
   - PF < 0.90
   - Demand > 90% Limit
   - Energy Spike > 20% ใน 15 min
```

---

## สรุป Part 39

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Power Basics | 3-Phase, Power Factor, Demand |
| Smart Meter | Modbus Integration, Register Map |
| CT Setup | Selection, Safety, Calibration |
| PF Correction | Capacitor Bank Control |
| Demand Management | Load Shedding, Priority |
| Data Logging | Circular Buffer, 15-min Intervals |
| OEE & KPIs | Availability, Performance, Quality |
| ISO 50001 | PDCA, EnPIs |
| SCADA Dashboard | Real-time Display |
| Reports | Auto-generation, Email |

---

## การนำทาง

- [← Part 38: Recipe Management](part38_recipe_management.md)
- [→ Part 40: Predictive Maintenance](part40_predictive_maintenance.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 39 of 40 | Steps 761-780*
