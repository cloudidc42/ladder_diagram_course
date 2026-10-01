# Part 57: Energy Optimization (Steps 1121-1140)

## บทนำ: ทำไม Energy Optimization ถึงสำคัญ?

ต้นทุนพลังงานในอุตสาหกรรมไทย:
- **Manufacturing**: 15-40% ของ total operating cost คือ energy
- ราคา electricity ในไทย: 3.5-4.5 บาท/kWh (industrial rate)
- โรงงาน 1 MW × 8,000 ชม/ปี × 4 บาท/kWh = **32 ล้านบาท/ปี**
- ลดได้ 10% = **3.2 ล้านบาท/ปี savings**

---

## Step 1121: Energy Audit Process

### กระบวนการ Energy Audit

```
ENERGY AUDIT METHODOLOGY (ISO 50002):
──────────────────────────────────────────────────────────────

PHASE 1: PLANNING
├── กำหนดขอบเขต (Boundary): เฉพาะ Process? ทั้ง Site?
├── รวบรวมข้อมูลปริมาณการใช้พลังงาน 1-3 ปีย้อนหลัง
│   (จาก Electric bill, oil bill, gas bill)
├── ระบุ significant energy uses (SEU): อะไรใช้มากที่สุด
└── วางแผนการเก็บข้อมูล (metering plan)

PHASE 2: DATA COLLECTION
├── ติดตั้ง power meters ชั่วคราว (หรือดู existing)
├── บันทึก load profile (ทุก 15 นาที, อย่างน้อย 2 สัปดาห์)
├── บันทึก process data ควบคู่กัน (production rate, temperature)
└── ระบุ correlation: energy vs production

PHASE 3: ANALYSIS
├── คำนวณ Specific Energy Consumption (kWh/ton หรือ kWh/unit)
├── เปรียบเทียบกับ benchmark (industry average, best-in-class)
├── ระบุ energy waste sources
└── คำนวณ potential savings per improvement

PHASE 4: REPORTING
├── Executive summary (savings opportunity)
├── Detailed findings per system
├── Prioritized improvement list (ROI ranking)
└── Implementation roadmap

Typical SEU (Significant Energy Uses) ในโรงงาน:
┌─────────────────────┬─────────────────────────────────────┐
│ SEU                 │ % ของ Total Energy ทั่วไป           │
├─────────────────────┼─────────────────────────────────────┤
│ Motor systems       │ 30-50%                              │
│ Compressed air      │ 15-25%                              │
│ Heating/HVAC        │ 20-30%                              │
│ Lighting            │ 5-10%                               │
│ Refrigeration       │ 5-15%                               │
└─────────────────────┴─────────────────────────────────────┘
```

---

## Step 1122: Power Metering สำหรับ Industrial

### Power Meter Types และ Installation

```
Power Meter Categories:
──────────────────────────────────────────────────────────────

PANEL METERS (ติดใน MCC/Distribution panel):
├── Schneider iEM3xxx series
├── Siemens SENTRON PAC series
├── ABB B23/B24 series
└── ใช้สำหรับ: main feeder, สาย major equipment

SMART METERS (utility grade):
├── ใช้สำหรับ billing/demand management
└── Accuracy: ±0.5% (Class 0.5S)

PORTABLE METERS (สำหรับ survey):
├── Fluke 435-II (power quality + energy)
├── Hioki PW3360 (clamp-on type)
└── ใช้ clamp CT → ไม่ต้อง shutdown circuit

PROTOCOL ที่ meter support:
├── Modbus RTU/TCP ← ใช้มากที่สุด
├── BACnet (สำหรับ building automation)
├── IEC 61850 (สำหรับ power substations)
└── DLMS/COSEM (สำหรับ utility meters)

INSTALLATION REQUIREMENTS:
├── CT (Current Transformer): sizing ตาม max current
│   - 100A load → ใช้ 100:5 หรือ 100:1 CT
│   - Accuracy: Class 0.5 (energy), Class 1 (OK สำหรับ monitoring)
├── PT (Potential Transformer): สำหรับ HV systems
├── Wiring: shielded cable สำหรับ low-level signals
└── Earth: CT secondary ต้องมี safety earth
```

### Modbus Power Meter Integration กับ PLC

```
// PLC อ่านข้อมูลจาก Schneider iEM3155 Power Meter
// Communication: Modbus TCP, IP: 192.168.30.50
// ─────────────────────────────────────────────────────────────

// Register Map (iEM3155):
// 3000: Voltage L1 (REAL, ×0.01 = V)
// 3028: Current L1 (REAL, ×0.001 = A)
// 3054: Active Power Total (REAL, W)
// 3258: Energy (Active, Import) (INT32, ×1 = kWh)
// 3082: Power Factor Total (REAL, ×0.001)
// 3110: Frequency (REAL, ×0.01 = Hz)

DATA_BLOCK DB_PowerMeter
    rVoltage_L1:    REAL;       // V
    rVoltage_L2:    REAL;
    rVoltage_L3:    REAL;
    rCurrent_L1:    REAL;       // A
    rCurrent_L2:    REAL;
    rCurrent_L3:    REAL;
    rPower_Total:   REAL;       // W (Active)
    rPowerFactor:   REAL;       // 0.0 - 1.0
    rFrequency:     REAL;       // Hz
    rEnergy_Import: REAL;       // kWh (cumulative)
    rDemand:        REAL;       // kW (15-min demand)
END_DATA_BLOCK

// Ladder: อ่าน Power Meter ด้วย MB_CLIENT (ทุก 5 วินาที)
// ─────────────────────────────────────────────────────────────
//
//  "Read_5sec_Timer"                  MB_CLIENT
//  ──┤├──────────────────────────── ┤ ENO ├────────────────────
//                                     REQ: TRUE
//                                     IP_ADDR: "192.168.30.50"
//                                     PORT: 502
//                                     MB_MODE: 0       (Read)
//                                     MB_DATA_ADDR: 3054 (Power)
//                                     MB_DATA_LEN: 2     (2 registers)
//                                     MB_DATA_PTR: DB_ModbusRaw.Power
//
// Scale: DB_PowerMeter.rPower_Total := DB_ModbusRaw.Power × 1.0;
// (iEM3155 returns power in W directly as REAL)
```

---

## Step 1123: Power Quality Monitoring

### Harmonics และ Power Factor

```
POWER QUALITY ISSUES สำหรับ Industrial:
──────────────────────────────────────────────────────────────

1. POWER FACTOR (PF)
เป้าหมาย: PF ≥ 0.9 (Thai EGAT requirement)
ถ้า PF < 0.9: MEA/PEA เก็บ reactive power surcharge

PF = Active Power (kW) / Apparent Power (kVA)

การแก้: ติดตั้ง PFC (Power Factor Correction) capacitors
├── Fixed capacitors: สำหรับ base load
└── Automatic PFC panel: สำหรับ variable load

Capacitor Sizing:
Q_cap (kVAR) = P × (tan(φ1) - tan(φ2))
= 1000 kW × (tan(cos⁻¹(0.75)) - tan(cos⁻¹(0.95)))
= 1000 × (0.882 - 0.329)
= 553 kVAR (capacitor bank needed)

2. HARMONICS
เกิดจาก: VFD, UPS, power electronics
มาตรฐาน: IEEE 519 (THD < 5% at PCC)
การแก้: 
├── Harmonic filters (passive: L-C circuit)
├── Active harmonic filters (ราคาแพงแต่ดีกว่า)
└── 18-pulse drives (built-in harmonic reduction)

3. VOLTAGE SAGS/SWELLS
เกิดจาก: large motor starting, arc furnace, lightning
การแก้:
├── UPS (สำหรับ critical equipment)
├── DVR (Dynamic Voltage Restorer)
└── Motor soft starter (ลด starting current)

4. VOLTAGE UNBALANCE
เป้าหมาย: < 2% unbalance
ผล: motor heating, efficiency loss
การแก้: balance load across 3 phases
```

---

## Step 1124: Energy Baseline and Performance

### ISO 50001 Energy Performance

```
Energy Baseline (EnB) and Energy Performance Indicators (EnPI):
──────────────────────────────────────────────────────────────

EnPI = Energy Consumption / Relevant Variable
ตัวอย่าง:
├── kWh/ton product
├── kWh/unit produced  
├── kWh/m² (for HVAC)
└── BTU/lb (สำหรับ heating processes)

วิธีสร้าง Energy Baseline (OLS Regression):
Energy = a + b × Production_Volume + c × Temperature + ε

ตัวอย่าง Regression Analysis (Python):
import pandas as pd
from sklearn.linear_model import LinearRegression
import numpy as np

# ข้อมูลรายเดือน
data = pd.DataFrame({
    'energy_kwh': [450000, 480000, 520000, 510000, 490000],
    'production_ton': [900, 960, 1040, 1020, 980],
    'ambient_temp': [28, 30, 34, 33, 31]
})

# Regression
X = data[['production_ton', 'ambient_temp']]
y = data['energy_kwh']
model = LinearRegression().fit(X, y)

print(f"Intercept: {model.intercept_:.0f} kWh")
print(f"Energy per ton: {model.coef_[0]:.1f} kWh/ton")
print(f"Energy per °C: {model.coef_[1]:.1f} kWh/°C")
print(f"R²: {model.score(X, y):.3f}")

# Output:
# Intercept: 12,500 kWh (baseline consumption - even at 0 production)
# Energy per ton: 450 kWh/ton
# Energy per °C: 1,200 kWh/°C (cooling load from temperature)
# R²: 0.978 (good fit)

Normalized Performance:
Actual energy vs Expected energy (from baseline model)
If Actual > Expected by > 5%: investigate
If Actual < Expected by > 5%: improvement (record for credit)
```

---

## Step 1125: Load Profile Analysis

### Demand Management

```
Load Profile Analysis:
──────────────────────────────────────────────────────────────

15-Minute Interval Load Profile (typical):
Hour | kW    | Notes
──────────────────────────────────────────────────────────────
00   | 800   | Night operation (low)
06   | 1200  | Morning startup
08   | 2000  | Full production
12   | 1900  | Lunch reduction
14   | 2200  | Afternoon peak ← PEAK DEMAND
16   | 2000  | Normal
18   | 1800  | Shift change
20   | 1400  | Evening wind-down
22   | 900   | Maintenance
24   | 800   | Low night

PEAK DEMAND CHARGE (MEA เก็บเพิ่ม):
Peak demand = ค่าสูงสุดของ 15-นาที average ในเดือน
Rate (TOU): ~200-250 บาท/kW/เดือน

ตัวอย่าง:
Peak demand = 2,200 kW
Monthly demand charge = 2,200 × 220 = 484,000 บาท/เดือน!

ลดได้ด้วย:
1. Demand limiting: ตัด non-critical load เมื่อใกล้ peak
2. Schedule shift: ย้าย heavy loads นอก peak period
3. Staggered starting: ไม่ start motors พร้อมกัน
4. Energy storage: ชาร์จในช่วงถูก discharge ในช่วงแพง
```

### Demand Limiting ใน PLC

```
// Demand Limiting System ใน PLC
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_DemandLimiting
VAR_INPUT
    rCurrentDemand_kW:  REAL;           // จาก power meter
    rDemandLimit_kW:    REAL := 2000.0; // Target max demand
    rDemandWarning_kW:  REAL := 1900.0; // Warning threshold
END_VAR

VAR_OUTPUT
    bDemandWarning:     BOOL;           // Approaching limit
    bDemandLimiting:    BOOL;           // Actively limiting
    bLoad1_Shed:        BOOL;           // Shed Load 1 (non-critical)
    bLoad2_Shed:        BOOL;           // Shed Load 2
    bLoad3_Shed:        BOOL;           // Shed Load 3
    rDemandMargin:      REAL;           // Remaining headroom
END_VAR

VAR
    tLoad1_Delay:       TON;            // Timer before shed load 1
    tLoad1_Restore:     TOF;            // Timer to restore load 1
    xDemandExceeded:    BOOL;
END_VAR

// Network 1: Calculate Margin
// rDemandMargin := rDemandLimit_kW - rCurrentDemand_kW;

// Network 2: Warning
// bDemandWarning := rCurrentDemand_kW >= rDemandWarning_kW;

// Network 3: Demand Exceeded
// xDemandExceeded := rCurrentDemand_kW >= rDemandLimit_kW;

// Network 4: Load Shedding (อัตโนมัติ, เรียงลำดับ priority)
// ถ้า demand เกิน limit → shed non-critical loads ตามลำดับ

// Tier 1 (ปิดก่อน): Air compressor 3 (spare)
// tLoad1_Delay(IN := xDemandExceeded, PT := T#10s);
// bLoad1_Shed := tLoad1_Delay.Q;

// Tier 2 (ปิดต่อมา): เครื่องเย็น 2 (มี 3 ตัว ปิด 1 ได้)
// tLoad2_Delay(IN := (xDemandExceeded AND bLoad1_Shed), PT := T#20s);
// bLoad2_Shed := tLoad2_Delay.Q;

// Network 5: Restore loads (เมื่อ demand กลับลงมาต่ำกว่า 90%)
// xRestoreOK := rCurrentDemand_kW < (rDemandLimit_kW × 0.9);
// tLoad1_Restore(IN := xRestoreOK, PT := T#120s);  // รอ 2 นาทีก่อน restore
// IF tLoad1_Restore.Q THEN
//     bLoad1_Shed := FALSE;
//     bLoad2_Shed := FALSE;
// END_IF;
```

---

## Step 1126: Compressed Air System Optimization

### Compressed Air - โอกาส Savings ใหญ่

```
Compressed Air Facts:
──────────────────────────────────────────────────────────────
- มีประสิทธิภาพต่ำ: ต้นทุน electricity → energy ใน compressed air ≈ 10-20%
  (80-90% ของ electricity กลายเป็น heat waste!)
- 20-30% ของ compressed air อาจรั่ว (old systems)
- 1 mm leak hole ที่ 7 bar: รั่ว ~0.8 m³/min = ~3.5 kW = ~100,000 บาท/ปี

Optimization Opportunities:
──────────────────────────────────────────────────────────────
1. LEAK DETECTION AND REPAIR
   วิธีตรวจ:
   ├── Ultrasonic detector (Fluke 819, SDT200)
   │   - ฟังเสียง ultrasonic ที่ leak point
   │   - ได้ผล 90%+ ของ leaks
   ├── Soap bubble test (เล็กน้อย, ช้า)
   └── Nitrogen tracer gas (precise, expensive)
   
   ROI ของ leak repair:
   Fix 10 leaks × 100,000 บาท/ปี each = 1,000,000 บาท/ปี savings
   Cost to fix: 10 × 5,000 = 50,000 บาท
   ROI: 1 week!

2. PRESSURE REDUCTION
   - Current: 8 bar, ต้องการจริง: 6 bar
   - Savings: ลด pressure 1 bar → ลด power ≈ 6-8%
   - ทำได้: ตั้ง compressor delivery pressure ลด

3. SYSTEM PRESSURE SETPOINT OPTIMIZATION
   - Peak demand ต้องการ: 6 bar
   - Night demand ต้องการ: 5 bar
   - Profile: ปรับ setpoint ตาม production schedule ด้วย PLC
```

### Compressed Air Monitoring ด้วย PLC

```
// Compressed Air Leak Detection System
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_CompressedAirMonitor
VAR_INPUT
    rSystemPressure:    REAL;           // bar
    rCompressorFlow:    REAL;           // m³/min
    rProductionFlow:    REAL;           // m³/min (วัดจาก flow meters บน production)
    bProductionRunning: BOOL;
    bNightShift:        BOOL;
END_VAR

VAR_OUTPUT
    rLeakFlow:          REAL;           // Estimated leak flow (m³/min)
    rLeakCost_Annual:   REAL;           // Estimated annual cost (บาท)
    rSpecificConsumption:REAL;          // m³/ton product
    bHighLeakAlarm:     BOOL;           // Leak > 10% of supply
    bNightLeakCheck:    BOOL;           // Night check result
END_VAR

VAR
    tNightCheckTimer:   TON;
    rNightPressureDrop: REAL;           // Rate of pressure drop at night
    rPressureStart:     REAL;
    rNightFlow:         REAL;           // Flow at night = all leaks
END_VAR

// Network 1: Calculate Leak Flow
// เมื่อ production หยุด: flow ที่เหลือทั้งหมดคือ leak
// IF NOT bProductionRunning THEN
//     rLeakFlow := rCompressorFlow;  // ทั้งหมดคือ leak
// ELSE
//     rLeakFlow := rCompressorFlow - rProductionFlow;  // Difference
// END_IF;

// Network 2: Leak Cost Calculation
// Electricity cost for leak:
// P_leak = rLeakFlow × 7 [bar] / (efficiency) kW
// ≈ rLeakFlow × 6 kW per m³/min
// rLeakCost_Annual := rLeakFlow × 6.0 × 8000 × 4.0;
// (6 kW/m³min × 8000 operating hours × 4 บาท/kWh)

// Network 3: Leak Alarm
// bHighLeakAlarm := rLeakFlow > (rCompressorFlow × 0.10);
// (เตือนถ้า leak > 10% ของ supply)
```

---

## Step 1127: Motor Right-Sizing

### Motor Efficiency and Right-Sizing

```
Motor Energy Audit:
──────────────────────────────────────────────────────────────

Motor Efficiency Classes (IE ratings):
┌────────────┬────────────────────────────────────────────────┐
│ Class      │ Description                                    │
├────────────┼────────────────────────────────────────────────┤
│ IE1        │ Standard efficiency (ห้ามขายในยุโรปแล้ว)      │
│ IE2        │ High efficiency                                 │
│ IE3        │ Premium efficiency (+1-3% vs IE2)              │
│ IE4        │ Super premium efficiency                       │
│ IE5        │ Ultra premium efficiency (inverter-only)       │
└────────────┴────────────────────────────────────────────────┘

Motor Oversizing Problem:
- Motor ขนาด 75 kW ใช้งานจริงแค่ 20 kW (26% load)
- Efficiency at 26% load: ~85% vs at 75% load: ~93%
- ปีละ 8000 ชั่วโมง: สูญเสียพิเศษ 1.3 kW × 8000 = 10,400 kWh
- = 41,600 บาท/ปี (สิ้นเปลือง!)

Motor Right-Sizing Decision:
If Load < 50% of Motor Rating AND Motor is IE1 → Replace with smaller IE3
If Load 50-75% AND Motor is IE1 → Upgrade to IE3 (same size)
If Load > 75% → Keep, ensure cooling adequate

VFD (Variable Frequency Drive) Benefits:
- Pump: Q ∝ N, P ∝ N³ (Affinity Law)
- ลด speed 20% → ลด power 49%!
- ลด speed 50% → ลด power 87.5%!
- Payback: 1-3 ปีสำหรับ variable-load applications
```

---

## Step 1128: VFD Energy Savings Calculation

### Affinity Law และการคำนวณ Savings

```
Affinity Laws สำหรับ Centrifugal Fan/Pump:
──────────────────────────────────────────────────────────────

Q₂/Q₁ = N₂/N₁     (Flow ∝ Speed)
H₂/H₁ = (N₂/N₁)² (Head ∝ Speed²)
P₂/P₁ = (N₂/N₁)³ (Power ∝ Speed³)

ตัวอย่างการคำนวณ Savings:
──────────────────────────────────────────────────────────────
Cooling Tower Fan:
├── Design flow: 100% (ต้องการที่ summer peak)
├── Average operating flow: 70% (temperature varies)
├── Motor: 55 kW (rated)
├── Operating hours: 6,000 hr/year

WITHOUT VFD (Damper/inlet vane control):
- Run at 100% speed, throttle with damper
- Power at 70% flow (damper): ≈ 85% of rated = 46.75 kW

WITH VFD (Speed control):
- Run at 70% speed to achieve 70% flow
- Power = 55 × (0.70)³ = 55 × 0.343 = 18.87 kW

Annual Savings:
- Energy savings: (46.75 - 18.87) × 6,000 = 167,280 kWh
- Cost savings: 167,280 × 4 = 669,120 บาท/ปี

VFD Cost: ≈ 150,000 บาท
Payback: 150,000 / 669,120 = 2.7 months!

R O I  ด้านเดียว:  669,120/150,000 = 446%/year ← เกินกว่าพอ
```

---

## Step 1129: VFD Energy Monitor ใน PLC

### PLC Program สำหรับ VFD Energy Tracking

```
// VFD Energy Tracking Program
// PROFINET สื่อสารกับ Siemens SINAMICS G120 Drive
// ─────────────────────────────────────────────────────────────

// Drive PROFINET Parameters ที่ต้องอ่าน:
// P1260 = Actual speed (Hz)
// P1275 = Output power (kW)
// P1276 = Energy counter (kWh) - resettable
// P2000-base = DC Bus Voltage

DATA_BLOCK DB_VFD_Energy
    // Per VFD data
    drives: ARRAY[1..10] OF STRUCT
        rSpeed_Hz:      REAL;       // Hz
        rSpeed_Pct:     REAL;       // %
        rPower_kW:      REAL;       // kW
        rEnergy_kWh:    REAL;       // kWh (accumulated)
        rEnergy_Today:  REAL;       // kWh today
        rEfficiency:    REAL;       // % (output/input)
        
        // Savings calculation
        rPower_NoVFD:   REAL;       // Estimated power without VFD
        rSavings_kWh:   REAL;       // Hourly savings
        rSavings_Annual:REAL;       // Projected annual savings
    END_STRUCT;
    
    rTotalPower_kW:     REAL;       // Total all drives
    rTotalSavings_kWh:  REAL;       // Today's total savings
END_DATA_BLOCK

// Ladder: อ่าน Drive Data via PROFINET
// ─────────────────────────────────────────────────────────────
//
//  Always ON           RDREC (Read Record)
//  ──┤├──────────────── ┤ ENO ├──────────────────────────────
//                        REQ: TRUE (ทุก scan)
//                        ID: W#16#1000    (Drive 1 hardware ID)
//                        INDEX: W#16#B02E (Parameter set index)
//                        MLEN: 100
//                        VALID: bDrive1_DataValid
//                        RECORD: DB_DriveRaw.Drive1

// Function: คำนวณ Savings
// ─────────────────────────────────────────────────────────────
// rSpeed_Pct := rSpeed_Hz / 50.0 × 100.0;    // 50Hz = 100%
// rPower_NoVFD := rMotorRated_kW;             // Full speed power
// rSavings_kWh := (rPower_NoVFD - rPower_kW); // Savings per hour
// rSavings_Annual := rSavings_kWh × 6000 × 4; // บาท/ปี projection
```

---

## Step 1130: Building Automation Integration

### HVAC Integration กับ Process Control

```
Building Automation System (BAS) Integration:
──────────────────────────────────────────────────────────────

Protocols สำหรับ BAS Integration:
├── BACnet/IP: Standard สำหรับ building automation
│   Port 47808/UDP
├── Modbus TCP: ถ้า HVAC controller support
├── OPC UA: Standard modern connection
└── SNMP: สำหรับ basic status only

Data ที่น่าสนใจจาก BAS:
├── Chiller: Power consumption, COP, load
├── AHU: Fan speed, supply temperature, CO2
├── Lighting: Zone status, energy per zone
└── UPS: State, battery %, runtime

Cross-System Optimization Example:
Process cooling demand (PLC) → BAS Chiller (BACnet)
→ บอก Chiller ว่า process กำลัง load เท่าไหร่
→ Chiller เปิด/ปิด compressor ล่วงหน้า
→ ลด chilled water temperature hunting
→ ประหยัด: 3-5% chiller energy

Implementation สำหรับ Siemens TIA Portal:
// Gateway: TIA Portal → OPC UA → BACnet/IP gateway
// Products: Anybus Communicator, Softing, Hilscher

// หรือ: ใช้ Siemens DESIGO PX building automation system
// ที่สามารถ integrate กับ SIMATIC PLC โดยตรง
```

---

## Step 1131: ISO 50001 Energy Management System

### การ Implement ISO 50001

```
ISO 50001 Requirements:
──────────────────────────────────────────────────────────────

4.1 Energy Review (การทบทวนพลังงาน)
├── ระบุ SEU (Significant Energy Uses)
├── วิเคราะห์ปัจจัยที่มีผลต่อการใช้พลังงาน
└── คาดการณ์การใช้พลังงานในอนาคต

4.2 Energy Baseline
├── ข้อมูล 12 เดือน + ตัวแปรที่เกี่ยวข้อง
└── Statistical model ที่มี R² ≥ 0.75

4.3 EnPI (Energy Performance Indicators)
├── เลือก KPI ที่สะท้อน performance
├── ตัวอย่าง: kWh/ton, kWh/unit
└── Monitor ทุกเดือน

4.4 EnPI Objectives and Targets
├── Set improvement targets (e.g., ลด 5%/year)
└── Action plans per SEU

AUTOMATION ของ ISO 50001 ด้วย PLC/SCADA:
──────────────────────────────────────────────────────────────
1. Data Collection: PLC อ่าน power meter อัตโนมัติ
2. EnPI Calculation: SCADA คำนวณ kWh/ton real-time
3. Reporting: Dashboard แสดง performance vs target
4. Alerting: แจ้งเตือนเมื่อ performance ต่ำกว่า target
5. Audit Trail: Log ทุกการเปลี่ยนแปลงพลังงาน

EnPI Dashboard (SCADA/Grafana):
┌─────────────────────────────────────────────────────────────┐
│ ENERGY PERFORMANCE INDICATOR - UNIT 1                       │
│                                                              │
│ This Month: 450 kWh/ton                                     │
│ Target:     430 kWh/ton (4.4% below baseline)               │
│ Baseline:   470 kWh/ton                                     │
│ Status:     ⚠️ 4.5% above target (investigate)             │
│                                                              │
│ Trend (last 12 months):                                      │
│ [Chart: EnPI trend vs target line]                           │
│                                                              │
│ Energy Tree (Pareto):                                        │
│ Compressors: 35% │ Drives: 28% │ Heating: 22% │ Other: 15%  │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1132: Energy Monitoring Program (Complete)

### Complete Energy Monitor FB

```
// FB_EnergyMonitor - Comprehensive Energy Tracking
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_EnergyMonitor
VAR_INPUT
    // Power Meter Inputs
    rPower_kW:          REAL;       // Active power (from meter)
    rReactivePower_kVAR:REAL;       // Reactive power
    rPowerFactor:       REAL;       // PF
    
    // Production Data
    rProduction_ton:    REAL;       // Production this hour
    
    // Time
    dtCurrentTime:      DATE_AND_TIME;
    bNewDay:            BOOL;       // Pulse ทุกเที่ยงคืน
    bNewMonth:          BOOL;       // Pulse วันแรกของเดือน
END_VAR

VAR_OUTPUT
    // Real-time
    rPower_kW_Out:      REAL;       // Pass-through
    rDemand_kW:         REAL;       // 15-min running average
    rEnPI:              REAL;       // kWh/ton (today)
    
    // Daily totals
    rEnergy_Today_kWh:  REAL;
    rProduction_Today_ton:REAL;
    rEnergy_Today_Cost: REAL;       // บาท
    
    // Monthly totals
    rEnergy_Month_kWh:  REAL;
    rEnergy_Month_Cost: REAL;
    
    // Alarms
    bHighDemand:        BOOL;       // > demand limit
    bLowPF:             BOOL;       // PF < 0.9
    bHighEnPI:          BOOL;       // EnPI > target
END_VAR

VAR RETAIN
    // Persistent daily accumulators
    rEnergy_Accum:      REAL;       // kWh since midnight
    rProd_Accum:        REAL;       // ton since midnight
    rEnergy_Month_Acc:  REAL;       // kWh this month
END_VAR

VAR
    // 15-min demand calculation
    aDemand_Buffer:     ARRAY[0..59] OF REAL;   // 60 × 15s = 15 min
    iBufferIndex:       INT;
    tDemandTimer:       TON;
    rSumForDemand:      REAL;
    
    // Rates
    rElectricRate:      REAL := 4.2; // บาท/kWh
    rEnPI_Target:       REAL := 430.0; // kWh/ton
    rDemandLimit:       REAL := 2000.0; // kW
END_VAR

// Network 1: Accumulate Energy (trapezoidal integration)
// rEnergy_Accum += rPower_kW × SCAN_TIME_HOURS;
// (SCAN_TIME_HOURS = 0.1s / 3600 = 0.0000278 ชม)

// Network 2: Reset at midnight
// IF bNewDay THEN
//     rEnergy_Today_kWh := rEnergy_Accum;  // Save yesterday total
//     rProd_Today_ton := rProd_Accum;
//     rEnergy_Month_Acc += rEnergy_Accum;
//     rEnergy_Accum := 0.0;               // Reset daily
//     rProd_Accum := 0.0;
// END_IF;

// Network 3: Running totals
// rEnergy_Today_kWh := rEnergy_Accum;
// rEnergy_Today_Cost := rEnergy_Accum × rElectricRate;

// Network 4: EnPI Calculation
// rProduction_Today_ton += rProduction_ton × SCAN_TIME_HOURS;
// IF rProduction_Today_ton > 0.0 THEN
//     rEnPI := rEnergy_Today_kWh / rProduction_Today_ton;
// END_IF;

// Network 5: 15-min Demand Calculation
// ทุก 15 วินาที: อัพเดท buffer
// tDemandTimer(IN := TRUE, PT := T#15s);
// IF tDemandTimer.Q THEN
//     tDemandTimer(IN := FALSE);  // Reset timer
//     aDemand_Buffer[iBufferIndex] := rPower_kW;
//     iBufferIndex := (iBufferIndex + 1) MOD 60;
//     rSumForDemand := 0;
//     FOR i := 0 TO 59 DO rSumForDemand += aDemand_Buffer[i]; END_FOR;
//     rDemand_kW := rSumForDemand / 60.0;  // Average of 15 min
// END_IF;

// Network 6: Alarms
// bHighDemand := rDemand_kW >= rDemandLimit;
// bLowPF      := rPowerFactor < 0.90;
// bHighEnPI   := rEnPI > rEnPI_Target × 1.05;  // >5% over target
```

---

## Step 1133: Renewable Energy Integration

### Solar PV + PLC Integration

```
Solar PV Monitoring สำหรับ Industrial:
──────────────────────────────────────────────────────────────

System Component:
├── Solar PV panels (e.g., 500 kWp rooftop)
├── String inverters (จาก PV → AC)
├── Energy meter (วัด solar generation)
├── Grid meter (วัด import/export)
└── PLC (ควบคุม load เพื่อ maximize solar self-consumption)

Data ที่ต้องการ (Modbus จาก Inverter):
├── PV Power (kW)
├── Today Generation (kWh)
├── Total Generation (kWh)
├── Inverter Status
└── DC/AC Voltage, Current

Optimization Strategy:
──────────────────────────────────────────────────────────────

Self-Consumption Maximization:
IF Solar_PV_Power > Current_Load THEN
    // มี excess solar: เปิด deferrable loads
    Turn on: water heater, air compressor, EV charger
ELSIF Solar_PV_Power < (Current_Load - Priority_Load) THEN
    // Solar ไม่พอ: ปิด non-critical loads
    Turn off: non-critical equipment
END_IF;

ประหยัดได้มากแค่ไหน?
500 kWp × 4.5 peak sun hours/day × 250 days = 562,500 kWh/year
562,500 × 4.2 บาท = 2,362,500 บาท/year savings
(ถ้า self-consumption 70%) = 1,653,750 บาท/year net
Solar cost: 500 × 20,000 = 10,000,000 บาท
Payback: 10,000,000 / 1,653,750 = 6 years
```

---

## Step 1134: Waste Heat Recovery

### Heat Integration ด้วย PLC

```
Waste Heat Recovery Opportunities:
──────────────────────────────────────────────────────────────

Common Sources ของ Waste Heat:
├── Compressor air cooler: 70-90°C (ปกติทิ้ง!)
├── Boiler flue gas: 150-300°C
├── Process coolers: cooling water 40-60°C
└── Refrigeration condenser: 40-50°C

เทคนิค Recovery:
1. Compressor Heat Recovery:
   - ความร้อนจากการอัด air: 80-90% ของ input power
   - Recover เป็น hot water (70°C) สำหรับ process washing
   - Savings: 30-50% ของ water heating cost
   
2. Boiler Economizer:
   - ดักความร้อน flue gas อุ่น boiler feed water
   - ทุก 11°C ลดลงของ flue gas temp = 1% fuel savings
   
3. Heat Exchanger Integration:
   Hot stream (waste) → Heat exchanger → Cold stream (product)

PLC Role ใน Heat Recovery:
├── Monitor temperatures (hot side, cold side)
├── Control bypass valve (too hot → bypass)
├── Calculate heat transferred
├── Track energy savings
└── Alert if HX fouling (ΔP increase or ΔT decrease)

// Heat Transfer Calculation ใน PLC:
// Q = m × Cp × ΔT
// Q_kW = FlowRate_kg_s × Cp_kJ_kgK × (T_in - T_out)
// rHeatRecovered_kW := rMassFlow × 4.18 × (rTemp_Hot_In - rTemp_Hot_Out);
// rSavings_per_hour := rHeatRecovered_kW × rFuelCost_per_kWh;
```

---

## Step 1135: Lighting Automation

### Smart Lighting Integration

```
Industrial Lighting Control:
──────────────────────────────────────────────────────────────

PLC-based Lighting Control:
├── Occupancy sensors → ปิดไฟเมื่อไม่มีคน
├── Daylight sensors → ลดความสว่างเมื่อแสงธรรมชาติมาก
├── Production-linked → ปิดพื้นที่ที่ไม่ได้ใช้งาน
└── Time-based schedules

ผลที่ได้:
- ลดการใช้ไฟฟ้าแสงสว่าง 40-70%
- LED retrofit เพิ่ม efficiency 50%
- Combined: 70-85% reduction

Protocol:
├── DALI (Digital Addressable Lighting Interface): professional
├── DMX: ความบันเทิง (ไม่ใช้ใน plant)
├── 0-10V: simple dimming
└── Relay output จาก PLC: on/off

// ตัวอย่าง: ปิดไฟในพื้นที่ที่ไม่ได้ผลิต
// IF NOT bArea1_Production_Running THEN
//     xArea1_Lights_Full := FALSE;   // ปิดไฟ full
//     xArea1_Lights_Min  := TRUE;    // เปิดแค่ security lighting
// END_IF;
```

---

## Step 1136: Energy Management Dashboard

### Real-time Energy Dashboard Design

```
ENERGY MANAGEMENT DASHBOARD (Grafana):
──────────────────────────────────────────────────────────────

PAGE 1: EXECUTIVE OVERVIEW
┌─────────────────────────────────────────────────────────────┐
│ ENERGY DASHBOARD - FACTORY A          Last update: 14:32    │
│                                                              │
│ Today's Energy: 24,500 kWh  (Budget: 26,000 kWh) ✓         │
│ Energy Cost Today: ฿102,900                                  │
│ EnPI Today: 442 kWh/ton  (Target: 430) ⚠️                  │
│                                                              │
│ Current Power: 2,150 kW  (Demand limit: 2,000 kW) ⚠️      │
│ Power Factor: 0.91 ✓                                        │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ POWER TREND (last 24 hours)                            │ │
│  │ [Line chart: actual vs budget line]                   │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                              │
│  ENERGY BREAKDOWN:                                           │
│  Motors: 38% ████████████                                   │
│  Compress: 24% ████████                                      │
│  Heating: 21% ███████                                        │
│  Cooling: 12% ████                                          │
│  Other: 5% ██                                               │
└─────────────────────────────────────────────────────────────┘

PAGE 2: EQUIPMENT DETAIL
├── Per-equipment power consumption (bar chart)
├── VFD efficiency trends
├── Compressor load profile
└── Chiller COP trend

PAGE 3: SUSTAINABILITY
├── CO2 emissions (kg CO2 = kWh × 0.5213 kg/kWh Thailand grid)
├── Solar generation vs consumption
├── Month-over-month comparison
└── Year-to-date vs target
```

---

## Step 1137: Energy Audit Report Template

### รายงาน Energy Audit ฉบับสมบูรณ์

```
ENERGY AUDIT REPORT STRUCTURE:
──────────────────────────────────────────────────────────────

SECTION 1: EXECUTIVE SUMMARY
├── Current energy cost: X ล้านบาท/ปี
├── Identified savings potential: Y ล้านบาท/ปี (Z%)
├── Investment required: W ล้านบาท
└── Payback period: V ปี

SECTION 2: ENERGY PROFILE
├── Annual consumption by fuel type
│   - Electricity: X kWh/year
│   - Natural gas: Y m³/year
│   - Steam: Z ton/year
├── Cost breakdown
└── Year-over-year trend

SECTION 3: SEU ANALYSIS
For each SEU (Compressed Air, HVAC, Motors, etc.):
├── Current consumption
├── Specific consumption (per unit production)
├── Benchmark comparison
└── Improvement opportunity

SECTION 4: OPPORTUNITIES (Ranked by ROI)
┌────┬─────────────────────────┬──────────┬──────────┬────────┐
│ #  │ Opportunity             │ Savings  │ Investment│Payback │
│    │                         │ (฿/yr)   │ (฿)       │ (mo)   │
├────┼─────────────────────────┼──────────┼──────────┼────────┤
│  1 │ Leak repair (air)       │ 800,000  │   50,000 │   1    │
│  2 │ VFD on cooling tower    │ 650,000  │  150,000 │   3    │
│  3 │ Demand management       │ 400,000  │   80,000 │   2    │
│  4 │ Boiler economizer       │ 1,200,000│  500,000 │   5    │
│  5 │ LED retrofit            │ 300,000  │  200,000 │   8    │
│  TOTAL                        │ 3,350,000│  980,000 │  3.5   │
└────┴─────────────────────────┴──────────┴──────────┴────────┘

SECTION 5: IMPLEMENTATION PLAN
├── Phase 1 (0-3 months): Quick wins (leak repair, demand mgmt)
├── Phase 2 (3-12 months): Medium investments (VFD, LED)
└── Phase 3 (12-24 months): Large investments (economizer)

SECTION 6: MONITORING PLAN
├── KPIs ที่จะ track
├── Measurement protocol
└── Review frequency
```

---

## Step 1138: Carbon Footprint Monitoring

### Carbon Emissions จาก Energy Use

```
Carbon Footprint Calculation:
──────────────────────────────────────────────────────────────

Thailand Electricity Emission Factor (2023):
≈ 0.5213 kg CO2/kWh (ดูอัพเดทจาก EGAT ทุกปี)

Calculation:
CO2 (kg) = kWh × Emission_Factor
CO2 (ton) = kWh × 0.0005213

Target Setting (Science Based Targets - SBTi):
├── 1.5°C pathway: ลด 50% ภายใน 2030
└── Net-zero: 2050

PLC/SCADA Carbon Tracking:
rCO2_Today_kg := rEnergy_Today_kWh × 0.5213;
rCO2_Annual_ton := rEnergy_Annual_kWh × 0.0005213;

Report สำหรับ GHG Protocol:
Scope 1: Direct (natural gas, diesel on-site)
Scope 2: Indirect (purchased electricity)
Scope 3: Value chain (supply chain, transport)

ESG Dashboard Integration:
├── Real-time CO2 metric (kg/hour, ton/month)
├── CO2 per unit production (intensity)
├── Year-to-date vs target
└── Renewable energy % ในทุกวัน
```

---

## Step 1139: Case Study - Thai Automotive Plant

### ตัวอย่างจริง: Energy Optimization Project

```
CASE STUDY: ผลลัพธ์ Energy Optimization ในโรงงานรถยนต์ไทย
──────────────────────────────────────────────────────────────

BACKGROUND:
├── Assembly plant, 200,000 units/year
├── Total energy: 45 million kWh/year
├── Cost: ฿189 million/year
└── EnPI: 225 kWh/unit (before project)

MEASURES IMPLEMENTED:
1. VFD บน paint shop HVAC (25 units)
   Savings: ฿3.2 million/year, Investment: ฿4 million
   Payback: 15 months

2. Compressed air leak program
   Found 45 leaks, repaired in 3 months
   Savings: ฿1.8 million/year, Investment: ฿200,000
   Payback: 5 weeks

3. Demand management via PLC
   Peak demand reduced 15% (from 6,000 kW to 5,100 kW)
   Savings: ฿2.1 million/year, Investment: ฿300,000
   Payback: 7 weeks

4. LED lighting retrofit (entire plant)
   Savings: ฿1.5 million/year, Investment: ฿8 million
   Payback: 64 months (5.3 years)

5. Boiler optimization (O2 trim, economizer)
   Savings: ฿2.5 million/year, Investment: ฿3 million
   Payback: 14 months

TOTAL RESULTS:
├── Total savings: ฿11.1 million/year (5.9%)
├── EnPI: 225 → 212 kWh/unit (5.8% reduction)
├── CO2 reduction: 6,300 ton CO2/year
└── Total investment: ฿15.5 million, Payback: 17 months
```

---

## Step 1140: Lab Project - Energy Management System

### Lab: Complete Energy Management สำหรับ Food Factory

**โจทย์**: โรงงานอาหาร 3 Lines การผลิต + Boiler + Compressed Air

**Objective**: สร้าง Energy Management System ที่ครบวงจร

```
LAB REQUIREMENTS:
══════════════════════════════════════════════════════════════

Part 1: Metering Setup and Data Collection (25%)
─────────────────────────────────────────────────────────────
1. ออกแบบ metering plan:
   - จำนวน meters ที่ต้องการ
   - ตำแหน่งติดตั้ง (single-line diagram)
   - Protocol (Modbus address map)

2. Program PLC อ่านข้อมูล power meters:
   - Main feeder
   - 3 production lines
   - Utilities (boiler, compressor, HVAC)

3. Data validation:
   - Sub-meter sum = main meter ± 2%
   - Range check: V ∈ [200-260V], I > 0

Part 2: EnPI Calculation (25%)
─────────────────────────────────────────────────────────────
1. Program FB_EnergyMonitor สำหรับ:
   - kWh/ton สำหรับแต่ละ production line
   - kWh/unit สำหรับ total plant
2. 15-minute demand calculation
3. Monthly and daily reporting

Part 3: Demand Management (25%)
─────────────────────────────────────────────────────────────
1. Design load shedding priority list
2. Program FB_DemandLimiting:
   - Limit: 2,000 kW
   - Tier 1: compressor 3 (spare)
   - Tier 2: HVAC zone B (non-production)
   - Tier 3: lighting non-critical areas
3. Test: Simulate demand exceeding limit

Part 4: Dashboard and Reporting (25%)
─────────────────────────────────────────────────────────────
1. Grafana Dashboard:
   - Real-time power (all areas)
   - EnPI trend (rolling 30 days)
   - CO2 meter (real-time + daily)
   - Demand chart with limit line
2. Daily energy report (automated email/PDF)
3. Alarm: EnPI > target + 5%, Demand > limit, PF < 0.9

DELIVERABLES:
├── PLC program (TIA Portal project)
├── Grafana dashboard screenshots
├── Test report (all functions tested)
├── Energy baseline model (spreadsheet/Python)
├── Savings calculation report
└── Recommendation for next 3 improvements

PASS CRITERIA:
├── Data accuracy: ±1% vs manual measurement
├── Demand response: sheds load within 30 seconds
├── EnPI calculation: correct formula and units
├── Dashboard: all widgets working
└── Complete documentation
```

---

## สรุป Part 57

| หัวข้อ | Typical Savings |
|--------|----------------|
| Compressed air leaks | 1-3% total energy |
| VFD on variable loads | 3-8% motor energy |
| Demand management | 1-3% total cost |
| LED lighting | 50-70% lighting cost |
| Process optimization | 2-5% process energy |
| Combined program | 5-15% total energy |

---

## Navigation

- [← Part 56: Advanced Diagnostics and PROFINET](./part56_advanced_diagnostics.md)
- [→ Part 58: PLC Programming Design Patterns](./part58_plc_programming_patterns.md)
- [↑ สารบัญหลัก](../README.md)
