# Part 40: Predictive Maintenance
## Steps 781-800: การบำรุงรักษาเชิงพยากรณ์ด้วย PLC

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 4-5 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 36 (SCADA), Part 37 (Alarm Management), Part 39 (Energy Monitoring)

---

## บทนำ

Predictive Maintenance (PdM) คือการบำรุงรักษาที่อาศัยการตรวจสอบสภาพจริงของอุปกรณ์ เพื่อทำนายเมื่อใดที่ควรซ่อมบำรุง ก่อนที่จะเกิดความเสียหาย

เปรียบเทียบกลยุทธ์การบำรุงรักษา:
- **Reactive (Run to Failure):** รอให้พังแล้วซ่อม → ต้นทุนสูง, Downtime มาก
- **Preventive (Time-based):** ซ่อมตามกำหนดเวลา → อาจซ่อมเร็วเกินหรือช้าเกิน
- **Predictive (Condition-based):** ซ่อมเมื่อสภาพบ่งบอก → ประหยัดสุด
- **Prescriptive:** AI แนะนำการซ่อม → ขั้นสูงสุด

---

## Step 781: Condition Monitoring Concepts

### ตัวชี้วัดสภาพอุปกรณ์

```
┌─────────────────────────────────────────────────────────────┐
│               Condition Monitoring Parameters               │
│                                                             │
│  Motor/Rotating Equipment:                                  │
│  ├── Vibration (mm/s or g-force)                           │
│  │   - Overall Level: ค่า RMS ทั้งหมด                     │
│  │   - Frequency Spectrum: FFT Analysis                   │
│  │   - High Frequency: Bearing Defects                    │
│  │                                                         │
│  ├── Temperature                                           │
│  │   - Bearing Temperature: ปกติ < 90°C                    │
│  │   - Winding Temperature: ปกติ < 130°C                   │
│  │   - Thermal Imager: ดู Hot Spots                       │
│  │                                                         │
│  ├── Current Signature                                     │
│  │   - Motor Current FFT: ตรวจหา Mechanical Faults        │
│  │   - Insulation Resistance: > 1 MΩ ปกติ                 │
│  │                                                         │
│  └── Oil Analysis                                          │
│      - Viscosity, Contamination, Metal Particles           │
│                                                             │
│  Static Equipment (Heat Exchanger, Vessel):                │
│  ├── Wall Thickness (Ultrasonic)                           │
│  ├── Corrosion Rate                                        │
│  └── Fouling Factor                                        │
└─────────────────────────────────────────────────────────────┘
```

### PdM Maturity Model

```
Level 1: Basic Monitoring
- ติดตั้ง Sensors พื้นฐาน (Temp, Current)
- แจ้งเตือนเมื่อเกิน Limit
- Log Data

Level 2: Advanced Monitoring  
- Vibration Analysis
- Trend Monitoring
- Pattern Recognition

Level 3: Predictive Analytics
- Machine Learning Models
- Remaining Useful Life (RUL) Prediction
- Automated Recommendations

Level 4: Prescriptive Maintenance
- AI-driven maintenance decisions
- Digital Twin Integration
- Closed-loop optimization
```

---

## Step 782: Vibration Monitoring Basics

### Vibration Parameters

```
Vibration Measurements:
┌──────────────────────────────────────────────────────────┐
│  Parameter    │ Unit  │ Detect                  │ Range  │
├──────────────────────────────────────────────────────────┤
│  Displacement │ μm    │ Low freq imbalance       │ 0-500  │
│  (peak-peak)  │       │ (< 10 Hz)               │        │
├──────────────────────────────────────────────────────────┤
│  Velocity     │ mm/s  │ Mid freq mechanical      │ 0-100  │
│  (RMS)        │       │ faults (10-1000 Hz)      │        │
├──────────────────────────────────────────────────────────┤
│  Acceleration │ g (G) │ High freq bearing        │ 0-50g  │
│  (RMS or peak)│       │ faults (>1000 Hz)        │        │
└──────────────────────────────────────────────────────────┘

ISO 10816 Vibration Severity (Velocity RMS):
Class I (< 15 kW):   Good < 1.8 mm/s, Alert < 4.5, Danger > 7.1
Class II (15-75 kW): Good < 2.8 mm/s, Alert < 7.1, Danger > 11.2
Class III (> 75 kW): Good < 4.5 mm/s, Alert < 11.2, Danger > 18
```

### Vibration Sensor Types

```
1. MEMS Accelerometer (Low-cost IIoT):
   - ราคาถูก, เล็ก
   - Frequency: 0-3200 Hz
   - ตัวอย่าง: ADXL345, MPU-6050
   - สื่อสาร: I2C, SPI, Modbus (via Gateway)

2. Piezoelectric Accelerometer (Industrial):
   - แม่นยำ, ทนร้อน
   - Frequency: 0-10 kHz
   - ตัวอย่าง: Wilcoxon 786A, PCB 603C01
   - Signal: ICP (IEPE), ±10V

3. Velocity Transducer (Classic):
   - Self-generating (ไม่ต้องการไฟ)
   - ช้าลงเมื่ออายุมาก
   - ตัวอย่าง: PCB CS series
```

### Vibration Monitoring ใน PLC

```
// ใช้ Analog Input อ่านค่าจาก Vibration Transmitter (4-20mA)
// 4mA = 0 mm/s, 20mA = 50 mm/s

// DB "Vibration_DB":
Vibration_Raw   : INT       // Raw AI Value
Vibration_mms   : REAL      // ค่าในหน่วย mm/s
Vibration_Peak  : REAL      // Peak ใน 1 นาที
Alert_Level     : REAL := 4.5  // Alert Threshold
Danger_Level    : REAL := 7.1  // Danger Threshold
Alert_Active    : BOOL      // Alert สถานะ
Danger_Active   : BOOL      // Danger สถานะ

// Trend Data
Sample_Count    : INT        // จำนวน Sample ใน Window
Sum_Squares     : REAL       // สำหรับ RMS Calculation
RMS_Value       : REAL       // RMS ทุก 1 นาที

// Ladder:
Network 1: อ่านค่า Vibration Analog Input (ใน OB35 - 100ms)

  |                                                         |
  |--[ALWAYS]------[SCALE]----------------------------------|
  |                 IN: IW256 (AI from Vibration Sensor)    |
  |                 LO_LIM: 0.0 (4mA)                      |
  |                 HI_LIM: 50.0 (20mA = 50mm/s)           |
  |                 OUT: "Vibration_DB".Vibration_mms       |
  |                                                         |

Network 2: คำนวณ RMS (สะสม Sum of Squares)

  |                                                         |
  |--[ALWAYS]------[MUL]-----------------------------------|
  |                 IN1: Vibration_mms                      |
  |                 IN2: Vibration_mms                      |
  |                 OUT: "Temp_Squared"                     |
  |                                                         |
  |--[ALWAYS]------[ADD]-----------------------------------|
  |                 IN1: Sum_Squares                        |
  |                 IN2: Temp_Squared                      |
  |                 OUT: Sum_Squares                        |
  |                                                         |
  |--[ALWAYS]------[INC]-----------------------------------|
  |                 Sample_Count                            |
  |                                                         |

Network 3: คำนวณ RMS ทุก 1 นาที

  |                                                         |
  |--[>=]---------[DIV]-------------------------------------|
  |  Sample_Count  IN1: Sum_Squares                        |
  |  >= 600        IN2: Sample_Count (600 samples/min)     |
  |  (600 samples) OUT: "Mean_Squares"                     |
  |                                                         |
  |              [SQRT]  (Square Root)                     |
  |               IN: Mean_Squares                         |
  |               OUT: RMS_Value                           |
  |                                                         |
  |              [MOVE]  (Reset)                           |
  |               IN: 0.0                                  |
  |               OUT: Sum_Squares                         |
  |              [MOVE]  IN:0  OUT:Sample_Count            |
  |                                                         |

Network 4: ตรวจสอบ Alert/Danger

  |                                                         |
  |--[>=]---------[SET]-------------------------------------|
  |  RMS_Value     "Vibration_DB".Alert_Active             |
  |  >= Alert_Level                                        |
  |                                                         |
  |--[>=]---------[SET]-------------------------------------|
  |  RMS_Value     "Vibration_DB".Danger_Active            |
  |  >= Danger_Level                                       |
  |                                                         |
```

---

## Step 783: Current Signature Analysis

### Motor Current Signature Analysis (MCSA)

```
MCSA ใช้หลักการ: 
- กระแสมอเตอร์มี Frequency Components ที่สัมพันธ์กับ Fault

Fault Frequencies:
- Eccentricity: fs × (1 ± 2ks) → fs = Supply freq, k=1,2,3...
- Broken Rotor Bar: fs × (1 ± 2ks) ด้วย
- Bearing Outer Race: fr × (N/2) × (1 - Bd/Pd × cos(φ))
- Bearing Inner Race: fr × (N/2) × (1 + Bd/Pd × cos(φ))

สรุป: ต้องการ FFT Analysis (DSP Processing)
สำหรับ PLC พื้นฐาน: วัดค่าเฉลี่ยกระแส + Trend

ข้อดี MCSA:
- ไม่ต้องติดตั้ง Sensor เพิ่ม (ใช้ CT เดิม)
- ตรวจจับได้แต่เนิ่นๆ
- วิเคราะห์ได้ขณะรัน
```

### ตรวจสอบ Motor Current ด้วย PLC

```
// DB "Motor_Health_DB":
Motor_ID          : INT
Motor_Name        : STRING[20]
Rated_Current     : REAL     // กระแสพิกัด (A)
Actual_Current_L1 : REAL     // กระแส L1
Actual_Current_L2 : REAL
Actual_Current_L3 : REAL
Current_Imbalance : REAL     // % ความไม่สมดุล
Load_Factor       : REAL     // % โหลด

// Current Imbalance Calculation:
// Normal: < 5%, Warning: 5-10%, Critical: > 10%

Network 1: คำนวณกระแสเฉลี่ย

  |                                                         |
  |--[ALWAYS]------[ADD]--[DIV]-----------------------------|
  |                 I_L1+I_L2+I_L3   / 3                   |
  |                 = "I_Average"                           |
  |                                                         |

Network 2: คำนวณ Max Deviation

  |                                                         |
  |--[ALWAYS]------[MAX]-----------------------------------|
  |                 Values: I_L1, I_L2, I_L3               |
  |                 OUT: "I_Max"                            |
  |                                                         |
  |--[ALWAYS]------[MIN]-----------------------------------|
  |                 Values: I_L1, I_L2, I_L3               |
  |                 OUT: "I_Min"                            |
  |                                                         |
  |--[ALWAYS]------[SUB]-[DIV]--[MUL]--------------------|
  |                 (I_Max - I_Min) / I_Average × 100       |
  |                 = "Current_Imbalance_%"                 |
  |                                                         |

Network 3: Load Factor

  |                                                         |
  |--[ALWAYS]------[DIV]-[MUL]------------------------------|
  |                 I_Average / Rated_Current × 100         |
  |                 = "Load_Factor_%"                       |
  |                                                         |

Network 4: แจ้งเตือน Imbalance

  |                                                         |
  |--[>=]---------[SET]-------------------------------------|
  |  Imbalance     "Current_Imbalance_Alarm"               |
  |  >= 5.0%                                                |
  |                                                         |
```

---

## Step 784: Thermal Monitoring

### อุณหภูมิ Bearing

```
// Bearing Temperature Monitoring
// ใช้ PT100 RTD หรือ Thermocouple

// อุณหภูมิปกติ:
// Rolling Element Bearing: < 70°C (Ambient +20°C)
// Journal Bearing: < 65°C
// คำเตือน: ถ้าอุณหภูมิเพิ่ม > 10°C/ชั่วโมง = มีปัญหา

// DB "Bearing_Temp_DB":
DE_Bearing_Temp    : REAL    // Drive End Bearing
NDE_Bearing_Temp   : REAL    // Non-Drive End Bearing
DE_Temp_1hour_ago  : REAL    // ค่าเมื่อ 1 ชั่วโมงที่แล้ว
NDE_Temp_1hour_ago : REAL
DE_Rate_of_Change  : REAL    // °C/hour
NDE_Rate_of_Change : REAL
Alert_Temp         : REAL := 70.0
Danger_Temp        : REAL := 85.0

// ใน OB30 (5 วินาที):
Network 1: อ่าน Bearing Temperature

  |                                                         |
  |--[ALWAYS]------[FC_RTD_Read]---------------------------|
  |                 Channel: 1 (DE Bearing)                 |
  |                 OUT: "Bearing_Temp_DB".DE_Bearing_Temp  |
  |                                                         |
  |--[ALWAYS]------[FC_RTD_Read]---------------------------|
  |                 Channel: 2 (NDE Bearing)                |
  |                 OUT: "Bearing_Temp_DB".NDE_Bearing_Temp |
  |                                                         |

// ใน OB32 (1 วินาที × 3600 = 1 ชั่วโมง):
Network 2: บันทึกค่าอุณหภูมิทุก 1 ชั่วโมง

  |                                                         |
  |--[CTU]--------[R]------[MOVE]--[MOVE]-----------------|
  |  Hour_Secs     >=3600   IN: DE_Temp  IN: 0            |
  |               Reset     OUT: DE_1h   OUT: Count        |
  |                                                         |

Network 3: คำนวณ Rate of Change (°C/hour)

  |                                                         |
  |--[ALWAYS]------[SUB]-----------------------------------|
  |                 IN1: DE_Bearing_Temp                    |
  |                 IN2: DE_Temp_1hour_ago                  |
  |                 OUT: DE_Rate_of_Change                  |
  |                                                         |
```

---

## Step 785: OEE Calculation

### OEE Components ใน PLC

```
// DB "OEE_Calculation_DB":
// ── Availability ──
Planned_Time_min     : REAL    // เวลาตามแผน (นาที/กะ)
Planned_Stop_min     : REAL    // หยุดตามแผน (break, lunch)
Unplanned_Stop_min   : REAL    // หยุดฉุกเฉิน
Net_Available_min    : REAL    // = Planned - Planned_Stop
Run_Time_min         : REAL    // = Net_Available - Unplanned
Availability         : REAL    // = Run_Time / Net_Available

// ── Performance ──
Ideal_Cycle_Time_s   : REAL    // เวลา Cycle ที่เร็วสุด (วินาที/ชิ้น)
Total_Count          : DINT    // ชิ้นทั้งหมดที่ผลิต
Actual_Rate          : REAL    // ชิ้น/นาทีจริง
Ideal_Rate           : REAL    // ชิ้น/นาทีที่ควรได้
Performance          : REAL    // = (Total × Ideal_CT) / Run_Time

// ── Quality ──
Good_Count           : DINT    // ชิ้นที่ผ่าน QC
Reject_Count         : DINT    // ชิ้นที่ไม่ผ่าน
Quality              : REAL    // = Good / Total

// ── OEE ──
OEE                  : REAL    // = A × P × Q

// ── Downtime Log ──
Downtime_Events      : INT     // จำนวนครั้งหยุด
Current_Downtime_Reason: INT   // รหัสเหตุผลหยุด
```

### OEE Ladder Program

```
// OB35 (100ms) - สะสมข้อมูล

Network 1: สะสม Run Time

  |                                                         |
  |--[ ]----------[ADD]-------------------------------------|
  |  Machine_Run   IN1: Run_Time_min                        |
  |               IN2: 0.00167 (0.1s ÷ 60 = 0.00167 min)  |
  |               OUT: Run_Time_min                        |
  |                                                         |

Network 2: สะสม Unplanned Downtime

  |                                                         |
  |--[NOT]---------[ADD]-----------------------------------|
  |  Machine_Run   IN1: Unplanned_Stop_min                 |
  |               IN2: 0.00167                             |
  |               OUT: Unplanned_Stop_min                  |
  |               (เวลาที่ Machine ควรรันแต่ไม่รัน)          |
  |                                                         |

Network 3: คำนวณ Availability

  |                                                         |
  |--[ALWAYS]------[SUB]-----------------------------------|
  |  Net = Planned - Planned_Stop                          |
  |                 IN1: Planned_Time_min                   |
  |                 IN2: Planned_Stop_min                   |
  |                 OUT: Net_Available_min                  |
  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |  Availability = Run / Net                              |
  |                 IN1: Run_Time_min                       |
  |                 IN2: Net_Available_min                  |
  |                 OUT: Availability                       |
  |                                                         |

Network 4: คำนวณ Performance

  |                                                         |
  |--[ALWAYS]------[MUL]-----------------------------------|
  |  Ideal Output = Run × Ideal_Rate                       |
  |                 IN1: Run_Time_min                       |
  |                 IN2: Ideal_Rate (pcs/min)              |
  |                 OUT: "Ideal_Output"                    |
  |                                                         |
  |--[ALWAYS]------[DINT_TO_REAL]--------------------------|
  |                 IN: Total_Count                         |
  |                 OUT: "Total_Count_Real"                 |
  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |                 IN1: Total_Count_Real                   |
  |                 IN2: Ideal_Output                      |
  |                 OUT: Performance                       |
  |                                                         |

Network 5: คำนวณ Quality และ OEE

  |                                                         |
  |--[ALWAYS]------[DINT_TO_REAL]-[DIV]-------------------|
  |                 Good_Count / Total_Count               |
  |                 OUT: Quality                           |
  |                                                         |
  |--[ALWAYS]------[MUL]-[MUL]----------------------------|
  |                 Avail × Perf × Quality                 |
  |                 OUT: OEE                               |
  |                                                         |
```

---

## Step 786: MTBF และ MTTR

### ความหมาย

```
MTBF (Mean Time Between Failures):
= เวลาเฉลี่ยระหว่างการเสียหาย
= Total Operating Time / Number of Failures

MTTR (Mean Time to Repair):
= เวลาเฉลี่ยในการซ่อม
= Total Repair Time / Number of Repairs

Availability = MTBF / (MTBF + MTTR)

ตัวอย่าง:
Motor ทำงาน 500 ชั่วโมง เสีย 2 ครั้ง แต่ละครั้งซ่อม 2 ชั่วโมง
MTBF = 500/2 = 250 ชั่วโมง
MTTR = (2+2)/2 = 2 ชั่วโมง
Availability = 250/(250+2) = 99.2%
```

### MTBF/MTTR Calculation Program

```
// DB "Maintenance_History":
Equipment_ID        : INT
Total_Run_Hours     : REAL      // ชั่วโมงสะสม
Failure_Count       : INT       // จำนวนครั้งที่เสีย
Total_Repair_Hours  : REAL      // ชั่วโมงซ่อมสะสม
MTBF                : REAL      // คำนวณ
MTTR                : REAL      // คำนวณ
Calculated_Avail    : REAL      // คำนวณ

Last_Failure_Time   : DATE_AND_TIME
Last_Repair_Done    : DATE_AND_TIME
Current_Run_Hours   : REAL      // ชั่วโมงหลังซ่อมล่าสุด

// Predict Next Failure:
Predicted_Failure_Hours: REAL   // = Current + MTBF
Days_to_Failure     : REAL      // = Predicted_Hours - Current_Hours / 24

// Ladder:
Network 1: สะสม Run Hours (OB32 - 1 second = 1/3600 hour)

  |                                                         |
  |--[ ]----------[ADD]-------------------------------------|
  |  Equipment_Run IN1: Total_Run_Hours                    |
  |               IN2: 0.000278 (1/3600 hour)              |
  |               OUT: Total_Run_Hours                     |
  |                                                         |
  |--[ ]----------[ADD]-------------------------------------|
  |  Equipment_Run IN1: Current_Run_Hours                  |
  |               IN2: 0.000278                            |
  |               OUT: Current_Run_Hours                   |
  |                                                         |

Network 2: บันทึกเมื่อเกิด Failure

  |                                                         |
  |--[P]-----------[INC]--[RD_SYS_T]----------------------|
  |  Equipment_Fail  Failure_Count  Last_Failure_Time       |
  |                                                         |

Network 3: บันทึกเมื่อซ่อมเสร็จ

  |                                                         |
  |--[P]-----------[ADD]--[MOVE]---------------------------|
  |  Repair_Done    Total_Repair_Hours + Repair_Duration    |
  |                [MOVE] 0.0 → Current_Run_Hours          |
  |                [RD_SYS_T] → Last_Repair_Done           |
  |                                                         |

Network 4: คำนวณ MTBF/MTTR

  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |  MTBF = Total_Run / Failure_Count                      |
  |                 IN1: Total_Run_Hours                    |
  |                 IN2: Failure_Count (as REAL)            |
  |                 OUT: MTBF                              |
  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |  MTTR = Total_Repair / Failure_Count                   |
  |                 IN1: Total_Repair_Hours                 |
  |                 IN2: Failure_Count (as REAL)            |
  |                 OUT: MTTR                              |
  |                                                         |

Network 5: คาดการณ์ Failure ถัดไป

  |                                                         |
  |--[ALWAYS]------[SUB]-----------------------------------|
  |  Days remaining = (MTBF - Current_Run_Hours) / 24      |
  |                 IN1: MTBF                              |
  |                 IN2: Current_Run_Hours                  |
  |                 OUT: "Remaining_Hours"                  |
  |                                                         |
  |--[ALWAYS]------[DIV]-----------------------------------|
  |                 IN1: Remaining_Hours                    |
  |                 IN2: 24.0                              |
  |                 OUT: Days_to_Failure                   |
  |                                                         |
```

---

## Step 787: Maintenance KPIs

### Maintenance Dashboard

```
Maintenance KPIs:
┌─────────────────────────────────────────────────────────┐
│  KPI                │ Formula          │ Target         │
├─────────────────────────────────────────────────────────┤
│  MTBF               │ Hours/Failure     │ > 2000 hrs    │
│  MTTR               │ Hours/Repair      │ < 4 hrs       │
│  Availability       │ MTBF/(MTBF+MTTR) │ > 98%         │
│  Planned Maint %    │ PM/(PM+CM)       │ > 80%         │
│  PM Compliance      │ Done/Planned     │ > 95%         │
│  Spare Parts Cost   │ $/Month          │ < Budget       │
│  Maintenance Cost   │ % of Asset Value │ 2-4%          │
│  Backlog            │ Open Work Orders │ < 2 weeks      │
└─────────────────────────────────────────────────────────┘
```

---

## Step 788: PLC-Based Condition Monitoring

### Complete Condition Monitor Program

```
// FB "FB_Condition_Monitor" - สำหรับแต่ละมอเตอร์

FB Interface:
VAR_INPUT:
  Motor_Running      : BOOL
  Current_L1         : REAL    // A
  Current_L2         : REAL    // A
  Current_L3         : REAL    // A
  Vibration_RMS      : REAL    // mm/s
  Bearing_Temp_DE    : REAL    // °C
  Bearing_Temp_NDE   : REAL    // °C
  Rated_Current      : REAL    // A (from nameplate)
  Vibration_Alert    : REAL    // mm/s alert threshold
  Vibration_Danger   : REAL    // mm/s danger threshold
  Bearing_Temp_Alert : REAL    // °C alert threshold
  Bearing_Temp_Danger: REAL    // °C danger threshold
END_VAR

VAR_OUTPUT:
  Health_Score       : INT     // 0-100 (100=Perfect)
  Alert_Active       : BOOL    // Any alert condition
  Danger_Active      : BOOL    // Any danger condition
  Current_Imbalance  : REAL    // %
  Load_Factor        : REAL    // %
  Status_Word        : WORD    // Bitfield of conditions
END_VAR

VAR_STAT:
  // Trend Data
  Vibration_1h_Ago   : REAL
  Vibration_24h_Ago  : REAL
  Temp_DE_1h_Ago     : REAL
  // Counters
  Run_Hours          : REAL
  Fault_Count        : INT
END_VAR
```

### Health Score Calculation

```pascal
// SCL: คำนวณ Health Score

// คะแนน 0-100 โดยพิจารณาจากหลายปัจจัย

// Vibration Score (40 points max)
IF Vibration_RMS < Vibration_Alert THEN
  Vibration_Score := 40.0
ELSIF Vibration_RMS < Vibration_Danger THEN
  // Linear interpolation between alert and danger
  Vibration_Score := 40.0 * (Vibration_Danger - Vibration_RMS) / 
                     (Vibration_Danger - Vibration_Alert)
ELSE
  Vibration_Score := 0.0
END_IF;

// Temperature Score (30 points max)
IF Bearing_Temp_DE < Bearing_Temp_Alert THEN
  Temp_Score := 30.0
ELSIF Bearing_Temp_DE < Bearing_Temp_Danger THEN
  Temp_Score := 30.0 * (Bearing_Temp_Danger - Bearing_Temp_DE) /
                (Bearing_Temp_Danger - Bearing_Temp_Alert)
ELSE
  Temp_Score := 0.0
END_IF;

// Current Balance Score (20 points max)
IF Current_Imbalance < 5.0 THEN
  Current_Score := 20.0
ELSIF Current_Imbalance < 10.0 THEN
  Current_Score := 20.0 * (10.0 - Current_Imbalance) / 5.0
ELSE
  Current_Score := 0.0
END_IF;

// Trend Score (10 points max) - is it getting worse?
Vib_Trend := Vibration_RMS - Vibration_24h_Ago
IF Vib_Trend < 0.5 THEN     // Stable or Improving
  Trend_Score := 10.0
ELSIF Vib_Trend < 2.0 THEN  // Slowly increasing
  Trend_Score := 5.0
ELSE                          // Rapidly increasing - concern
  Trend_Score := 0.0
END_IF;

// Total Health Score
Health_Score := REAL_TO_INT(Vibration_Score + Temp_Score + 
                             Current_Score + Trend_Score);
```

---

## Step 789: Alert Thresholds

### Dynamic Threshold Management

```
// DB "Threshold_DB":
// เก็บ Threshold สำหรับทุกอุปกรณ์

Thresholds : ARRAY[1..50] OF STRUCT
  Equipment_ID     : INT
  Equipment_Type   : INT    // 1=Motor, 2=Pump, 3=Fan
  Vib_Alert        : REAL   // mm/s
  Vib_Danger       : REAL   // mm/s
  Temp_Alert_Brg   : REAL   // °C
  Temp_Danger_Brg  : REAL   // °C
  Current_Imbal_Al : REAL   // %
  MTBF_Target      : REAL   // hours
END_STRUCT

// สามารถปรับจาก HMI ได้ (Engineer Level เท่านั้น)
```

### Adaptive Thresholds (ขั้นสูง)

```
// Adaptive Threshold: ปรับตามประวัติ
// ถ้า Machine A มี Vibration ปกติ 2.0 mm/s
// ก็ควรตั้ง Alert ที่ 3.0 mm/s (150% ปกติ)
// ไม่ใช้ Fixed Threshold เหมือนกันทุกเครื่อง

// คำนวณ Baseline จากข้อมูล 30 วันที่ผ่านมา:
// Baseline = Average + 1 Standard Deviation
// Alert = Baseline × 1.5
// Danger = Baseline × 2.0

// ต้องใช้ SCADA/External Script สำหรับการคำนวณนี้
// PLC ใช้ค่า Threshold ที่คำนวณแล้ว
```

---

## Step 790: Trend Data Collection

### Trend Monitoring Program

```
// DB "Trend_DB" - เก็บข้อมูล Trend ระยะยาว
Record_Count     : INT
Records          : ARRAY[1..8760] OF STRUCT  // 1 ปี (ทุก 1 ชั่วโมง)
  Timestamp      : DATE_AND_TIME
  Equipment_ID   : INT
  Vibration_RMS  : REAL
  Bearing_DE_Temp: REAL
  Bearing_NDE_Temp: REAL
  Current_Avg    : REAL
  Load_Factor    : REAL
  Health_Score   : INT
END_STRUCT

// ใน OB32 (1 วินาที × 3600 = 1 ชั่วโมง):
// บันทึกข้อมูลทุก 1 ชั่วโมง
```

### Trend Analysis (Simple Moving Average)

```
// คำนวณ SMA (Simple Moving Average) ใน PLC
// ใช้สำหรับ Smoothing Trend Data

// DB "SMA_Buffer":
Buffer_24h : ARRAY[1..24] OF REAL  // Hourly values 24h
Buffer_Index: INT := 1
SMA_24h     : REAL                  // Average ของ 24h

// ใน OB30 (5 วินาที × 720 = 1 ชั่วโมง):

Network 1: เพิ่ม Index (Circular)

  |                                                         |
  |--[ALWAYS]------[ADD]-----------------------------------|
  |  (Hourly trigger)  IN1: Buffer_Index                   |
  |                    IN2: 1                              |
  |                    OUT: "Temp_Index"                    |
  |                                                         |
  |--[>]----------[MOVE]-----------------------------------|
  |  Temp_Index > 24   IN: 1 (กลับต้น)                    |
  |                    OUT: Buffer_Index                   |
  |                                                         |

Network 2: บันทึกลง Buffer

  |                                                         |
  |--[ALWAYS]------[MOVE]----------------------------------|
  |  (Hourly trigger)  IN: Current_Vibration              |
  |                    OUT: Buffer_24h[Buffer_Index]       |
  |                                                         |

Network 3: คำนวณ SMA

  |                                                         |
  |--[ALWAYS]------[ADD_ARRAY]--[DIV]--------------------|
  |  (Sum all 24 elements)       / 24                     |
  |                               OUT: SMA_24h             |
  |                                                         |
```

---

## Step 791: Integration with CMMS

### CMMS (Computerized Maintenance Management System)

```
ข้อมูลที่ PLC/SCADA ส่งให้ CMMS:
┌─────────────────────────────────────────────────────────┐
│  Equipment Health Data                                  │
│  ├── Health Score (0-100)                               │
│  ├── Vibration Trend                                    │
│  ├── Temperature Trend                                  │
│  └── Run Hours                                          │
│                                                         │
│  Work Order Generation Triggers                        │
│  ├── Health Score < 50 → Generate PM Work Order        │
│  ├── MTBF Approaching → Schedule PM                    │
│  └── Alarm Condition → Generate CM Work Order          │
│                                                         │
│  CMMS ส่งกลับ:                                         │
│  ├── Next PM Date                                       │
│  ├── Work Order Status                                  │
│  └── Spare Parts Availability                          │
└─────────────────────────────────────────────────────────┘
```

### Auto Work Order Generation

```python
# Ignition Script: สร้าง Work Order อัตโนมัติ

def checkEquipmentHealth():
    """
    ตรวจสอบ Health Score ทุก 15 นาที
    สร้าง Work Order เมื่อจำเป็น
    """
    # อ่านข้อมูลจาก PLC Tags
    equipment_list = system.db.runQuery(
        "SELECT id, name, health_score, mtbf, current_run_hours "
        "FROM equipment_health "
        "WHERE is_active = 1"
    )
    
    for equip in equipment_list:
        health = equip['health_score']
        days_to_fail = (equip['mtbf'] - equip['current_run_hours']) / 24
        
        # Health Score ต่ำ → สร้าง Work Order
        if health < 50:
            priority = 'URGENT' if health < 30 else 'HIGH'
            createWorkOrder(
                equipment_id=equip['id'],
                type='PREDICTIVE',
                priority=priority,
                description=f"Health Score ต่ำ: {health}/100",
                due_days=3 if health < 30 else 7
            )
        
        # ใกล้ MTBF → สร้าง PM Work Order
        elif days_to_fail < 14:
            createWorkOrder(
                equipment_id=equip['id'],
                type='PREVENTIVE',
                priority='MEDIUM',
                description=f"คาดว่าต้องซ่อมใน {days_to_fail:.0f} วัน",
                due_days=int(days_to_fail * 0.7)  # 70% ของเวลาที่เหลือ
            )

def createWorkOrder(equipment_id, type, priority, description, due_days):
    """
    ส่ง Work Order ไปยัง CMMS ผ่าน API
    """
    import system.net
    
    payload = {
        "equipment_id": equipment_id,
        "type": type,
        "priority": priority,
        "description": description,
        "due_date": str(system.date.addDays(system.date.now(), due_days)),
        "created_by": "PdM_System",
        "source": "SCADA_AUTO"
    }
    
    response = system.net.httpPost(
        url="http://cmms.company.com/api/workorders",
        postData=system.util.jsonEncode(payload),
        contentType="application/json"
    )
    
    if response.good:
        system.tag.writeBlocking(
            ["[default]CMMS/LastWO_Created"],
            [response.json['work_order_id']]
        )
```

---

## Step 792: IIoT Sensor Integration

### Wireless Vibration Sensor ผ่าน IIoT

```
IIoT Vibration Sensors:
├── SKF IMx-1: Wireless, Bluetooth, LoRa
├── Fluke 3563: Wireless mesh, 10kHz
├── Siemens SITRANS MS200: IO-Link
└── Bosch CISS: Multi-sensor (Vib, Temp, Shock)

Communication:
IoT Sensor → Gateway → MQTT Broker → SCADA/Historian

MQTT Topic Structure:
factory/{plant}/{area}/{equipment_id}/{sensor_type}
ตัวอย่าง: factory/th01/production/motor-001/vibration
```

### MQTT Data Ingestion

```python
# Ignition Gateway Script: รับข้อมูลจาก MQTT

import json

def onMessageReceived(topic, payload):
    """
    ประมวลผล MQTT Message จาก IIoT Sensors
    """
    try:
        data = json.loads(payload)
        
        # Parse Topic: factory/th01/production/motor-001/vibration
        parts = topic.split('/')
        equipment_id = parts[3]
        sensor_type = parts[4]
        
        # เขียนลง PLC Tag
        tag_path = f"[default]Equipment/{equipment_id}/{sensor_type}"
        system.tag.writeBlocking([tag_path], [data['value']])
        
        # บันทึกลง Database
        query = """
            INSERT INTO iot_measurements 
            (timestamp, equipment_id, sensor_type, value, unit)
            VALUES (?, ?, ?, ?, ?)
        """
        system.db.runPrepUpdate(query, [
            data.get('timestamp', system.date.now()),
            equipment_id,
            sensor_type,
            data['value'],
            data.get('unit', '')
        ])
        
    except Exception as e:
        system.util.getLogger("MQTT").error(f"Error: {str(e)}")
```

---

## Step 793-800: Lab Projects

### Lab 1: Motor Health Monitor

**วัตถุประสงค์:** สร้างระบบ Monitor สุขภาพมอเตอร์

**อุปกรณ์:**
- Siemens S7-1200
- Vibration Sensor (Analog 4-20mA)
- PT100 RTD Temperature Sensor
- Current Transformer (3-phase)

**โปรแกรมที่ต้องสร้าง:**
```
1. อ่านค่า Vibration, Temperature, Current ทุก 100ms
2. คำนวณ RMS Vibration (ทุก 1 นาที)
3. คำนวณ Current Imbalance
4. คำนวณ Health Score (0-100)
5. Trend Data (บันทึกทุก 1 ชั่วโมง, เก็บ 30 วัน)
6. Alert/Danger Alarms
7. HMI Screen: Gauges, Trend Chart, Health Score
```

### Lab 2: Predictive Maintenance Dashboard

**วัตถุประสงค์:** สร้าง PdM Dashboard ใน Ignition/WinCC

```
Dashboard Components:
1. Equipment List with Health Score (Color coded)
   - Green: > 75%
   - Yellow: 50-75%
   - Red: < 50%

2. Vibration Trend (7 วัน)
   - Overall Level
   - Alert/Danger Lines

3. MTBF/MTTR Summary
   - By Equipment
   - By Equipment Type

4. Upcoming Maintenance (Next 30 days)

5. Energy + OEE Correlation
   - High Energy often → Low Health?

6. Alerts List (Active PdM Alerts)
```

### Lab 3: Complete PdM System

```
โจทย์: ระบบ PdM ครบวงจรสำหรับ 5 มอเตอร์

Hardware Simulation:
- S7-1200 PLC
- 5 Analog Inputs (Vibration)
- 5 PT100 RTD (Bearing Temp)
- Current Measurement (via Analog)

Software:
1. PLC Program:
   - อ่านข้อมูลทุก 100ms
   - คำนวณ RMS, Health Score
   - Alarm Management

2. SCADA:
   - Real-time Dashboard
   - Trend Charts
   - Alarm List

3. Database:
   - 15-min Interval Logging
   - Failure History
   - MTBF Calculation

4. Reports:
   - Weekly PdM Summary
   - Equipment Health Report
   - Top-5 Bad Actors

5. Work Order Integration:
   - Auto-create เมื่อ Health < 50
   - Email Alert to Maintenance Team
```

---

## สรุป Part 40 และ Course

### สิ่งที่เรียนรู้ใน Part 40

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Condition Monitoring | Vibration, Temperature, Current |
| PdM vs PM | ความแตกต่าง, เมื่อใช้ |
| Vibration | ISO 10816, RMS Calculation |
| MCSA | Current Signature Analysis |
| OEE | Availability × Performance × Quality |
| MTBF/MTTR | การคำนวณ, การทำนาย |
| Health Score | Multi-parameter Score |
| CMMS | Auto Work Order Generation |
| IIoT | Wireless Sensor Integration |
| Dashboard | PdM Visualization |

---

### สรุป Course ทั้งหมด (Part 1-40)

```
Part 1-10:    Ladder Diagram พื้นฐาน (Bit Logic, Timer, Counter)
Part 11-20:   Intermediate (Analog, Math, Comparison, Move)
Part 21-30:   Advanced (FC, FB, Structured Programming)
Part 31-40:   Advanced Intermediate (DB, Motion, VFD, SCADA, PdM)
```

**ทักษะที่ได้จาก Course นี้:**
- เขียนโปรแกรม Ladder Diagram ระดับ Advanced
- ทำงานกับ Siemens S7-1200/1500, Allen-Bradley, Mitsubishi
- สื่อสาร Modbus, PROFIBUS, PROFINET, EtherCAT
- ควบคุม Motion, Servo, VFD
- เชื่อมต่อ SCADA, Historian, MES
- Alarm Management (ISA-18.2)
- Recipe Management (ISA-88)
- Energy Monitoring (ISO 50001)
- Predictive Maintenance

---

## การนำทาง

- [← Part 39: Energy Monitoring](part39_energy_monitoring.md)
- [↑ สารบัญหลัก](../README.md)
- [🏠 Part 1: Introduction](../part1-20/part01_introduction.md)

---

## ขอบคุณที่เรียนจนครบหลักสูตร! 🎓

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 40 of 40 | Steps 781-800 | COURSE COMPLETE*
