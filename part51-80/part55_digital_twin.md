# Part 55: Digital Twin (Steps 1081-1100)

## บทนำ: Digital Twin คืออะไร?

Digital Twin คือ การจำลองดิจิทัลของระบบกายภาพ ที่เชื่อมโยงกับ real-world อย่างต่อเนื่อง ไม่ใช่แค่ simulation ทั่วไป

```
PHYSICAL ASSET          DIGITAL TWIN
─────────────           ─────────────
Real PLC          ←→   PLC Simulation
Real Sensors      ←→   Sensor Model
Real Actuators    ←→   Actuator Model
Real Process      ←→   Process Model
      │                       │
      └──── Data Sync ────────┘
            (bidirectional)
```

---

## Step 1081: ประเภทของ Digital Twin

### 4 ระดับของ Digital Twin

```
LEVEL 1: DESCRIPTIVE TWIN (as-is)
──────────────────────────────────────────────────────────────
- แสดงสถานะปัจจุบันของระบบ
- ข้อมูลมาจาก sensors และ PLC
- Real-time visualization
- ใช้สำหรับ: Remote monitoring, status visualization
- ตัวอย่าง: 3D model ของ pump ที่แสดงสี (เขียว/แดง) ตาม status

LEVEL 2: INFORMATIVE TWIN (as-is + history)
──────────────────────────────────────────────────────────────
- Descriptive + Historical data
- Pattern analysis
- ใช้สำหรับ: Troubleshooting, performance analysis
- ตัวอย่าง: แสดง vibration trend ของ motor ย้อนหลัง 6 เดือน

LEVEL 3: PREDICTIVE TWIN (as-is + future)
──────────────────────────────────────────────────────────────
- Informative + Predictive models
- Forecast future behavior
- ใช้สำหรับ: Predictive maintenance, process optimization
- ตัวอย่าง: ทำนายว่า bearing จะเสียใน X วัน

LEVEL 4: AUTONOMOUS TWIN (self-optimizing)
──────────────────────────────────────────────────────────────
- Predictive + Autonomous decision making
- Self-optimization
- ใช้สำหรับ: Advanced process control, autonomous operation
- ตัวอย่าง: ปรับ PID parameters อัตโนมัติตาม process conditions
- ⚠️ ต้องระวัง: ต้องมี safety boundaries ชัดเจน
```

---

## Step 1082: Simulation Tools

### เครื่องมือสำหรับสร้าง Digital Twin

```
┌───────────────────────────────────────────────────────────────────────┐
│                     DIGITAL TWIN TOOLS                                 │
├───────────────────┬───────────────────────────────────────────────────┤
│ Tool              │ ใช้สำหรับ                                         │
├───────────────────┼───────────────────────────────────────────────────┤
│ Siemens NX MCD    │ Mechatronics Concept Designer                     │
│                   │ - 3D mechanical simulation                         │
│                   │ - PLC connection via OPC UA                        │
│                   │ - PROFINET simulation                              │
│                   │ - ดีสำหรับ: machine tool, assembly, robotics      │
├───────────────────┼───────────────────────────────────────────────────┤
│ Factory I/O       │ - 3D factory simulation                           │
│                   │ - ราคาถูก (สำหรับ training)                       │
│                   │ - Modbus TCP, EtherNet/IP support                 │
│                   │ - ดีสำหรับ: learning, training, prototype         │
├───────────────────┼───────────────────────────────────────────────────┤
│ MATLAB/Simulink   │ - Process simulation (mathematical)               │
│                   │ - Control system design                            │
│                   │ - PLC code generation (Simulink PLC Coder)        │
│                   │ - ดีสำหรับ: process control, algorithm dev        │
├───────────────────┼───────────────────────────────────────────────────┤
│ Visual Components │ - Logistics simulation                            │
│                   │ - Material handling                               │
│                   │ - Robot simulation (ABB, FANUC plugins)           │
│                   │ - ดีสำหรับ: warehouse, conveyor, logistics        │
├───────────────────┼───────────────────────────────────────────────────┤
│ EPLAN Electric    │ - Electrical panel simulation                     │
│ P8 + 3D Panel     │ - Cable routing 3D                                │
│                   │ - Integration with TIA Portal                     │
├───────────────────┼───────────────────────────────────────────────────┤
│ ANSYS Twin Builder│ - Physics-based simulation                        │
│                   │ - Fluid dynamics, thermal, structural             │
│                   │ - ML-augmented models                             │
│                   │ - ดีสำหรับ: process equipment, turbines           │
├───────────────────┼───────────────────────────────────────────────────┤
│ Dassault Systemes │ - Comprehensive PLM + simulation                  │
│ 3DEXPERIENCE      │ - Virtual twin experience                         │
│                   │ - ดีสำหรับ: aerospace, automotive, defense        │
└───────────────────┴───────────────────────────────────────────────────┘
```

---

## Step 1083: Factory I/O + PLC Connection

### ตัวอย่าง: Conveyor System ใน Factory I/O

```
Factory I/O Setup:
──────────────────────────────────────────────────────────────

Scene ตัวอย่าง: "Sorting Station"
├── Conveyor belt (สายพาน)
├── Color sensor (ตรวจสีกล่อง)
├── 3 destination conveyors (แยกตามสี)
├── Pusher actuators (ดันกล่อง)
└── Exit sensors

I/O Mapping (Factory I/O ↔ PLC):
Input signals:
├── I0.0: Sensor at entry (กล่องเข้า)
├── I0.1: Color sensor - Red
├── I0.2: Color sensor - Green
├── I0.3: Color sensor - Blue
├── I0.4: Sensor at destination 1
├── I0.5: Sensor at destination 2
└── I0.6: Sensor at destination 3

Output signals:
├── Q0.0: Main conveyor motor
├── Q0.1: Pusher 1 (เลือกไปจุด 1)
├── Q0.2: Pusher 2 (เลือกไปจุด 2)
├── Q0.3: Pusher 3 (เลือกไปจุด 3)
├── Q0.4: Conveyor 1 motor
├── Q0.5: Conveyor 2 motor
└── Q0.6: Conveyor 3 motor

Connection Method:
├── Modbus TCP: Factory I/O as Modbus server, PLC as client
├── EtherNet/IP: ถ้า PLC เป็น Allen-Bradley
├── OPC UA: Factory I/O มี OPC UA server
└── S7-PLCSIM Advanced: Siemens-specific (ดีที่สุดสำหรับ TIA Portal)
```

### PLC Program สำหรับ Sorting Station

```
// Siemens S7-1500 / S7-PLCSIM Advanced
// Sorting Station Control Program

// ─────────────────────────────────────────────────────────────
// DATA BLOCK: DB_SortingControl
// ─────────────────────────────────────────────────────────────
DATA_BLOCK DB_SortingControl
    xConveyorRun:   BOOL := FALSE;
    xBoxDetected:   BOOL := FALSE;
    iBoxColor:      INT := 0;       // 0=None, 1=Red, 2=Green, 3=Blue
    iBoxCount_R:    INT := 0;       // นับกล่องแดง
    iBoxCount_G:    INT := 0;       // นับกล่องเขียว
    iBoxCount_B:    INT := 0;       // นับกล่องน้ำเงิน
END_DATA_BLOCK

// ─────────────────────────────────────────────────────────────
// OB1: Main Program
// ─────────────────────────────────────────────────────────────

// Network 1: Start/Stop Main Conveyor
// ────────────────────────────────────────────────────────────
//
//  "Start_PB"          "Stop_PB"       "E_Stop"    "Conveyor_Motor"
//  ──┤├──┬──────────────┤/├──────────────┤/├─────────────( )──
//        │
//  "Conveyor_Motor"
//  ──┤├──┘
//
// (Motor latching circuit)

// Network 2: ตรวจจับสีกล่อง
// ────────────────────────────────────────────────────────────
//
//  "Sensor_Entry"                          "Box_Detected"
//  ──┤├──────────────────────────────────────( )──
//
//  "Sensor_Red"        "Color_Red"
//  ──┤├─────────────────────( )──
//
//  "Sensor_Green"      "Color_Green"
//  ──┤├─────────────────────( )──
//
//  "Sensor_Blue"       "Color_Blue"
//  ──┤├─────────────────────( )──

// Network 3: State Machine (ดู State Chart ด้านล่าง)
// ────────────────────────────────────────────────────────────
// FC: FC_SortingStateMachine
// (จัดการ timing ของ pusher)

// Network 4: Pusher Activation
// ────────────────────────────────────────────────────────────
//
//  "At_Sort_Point1"   "Color_Red"     "Pusher_1_Timer"   "Pusher_1"
//  ────────┤├─────────────┤├────────────[TON 0.5s Q]───────( )──
//
//  "At_Sort_Point2"   "Color_Green"   "Pusher_2_Timer"   "Pusher_2"
//  ────────┤├─────────────┤├────────────[TON 0.5s Q]───────( )──
```

---

## Step 1084: Siemens TIA Portal Virtual PLC (S7-PLCSIM Advanced)

### S7-PLCSIM Advanced Setup

```
S7-PLCSIM Advanced - Virtual Commissioning กับ Factory I/O:
──────────────────────────────────────────────────────────────

วิธีการตั้งค่า:
──────────────────────────────────────────────────────────────

STEP 1: ติดตั้ง S7-PLCSIM Advanced
- ดาวน์โหลดจาก Siemens TIA Portal Website
- License: ต้องมี license (trial 21 วัน)
- Minimum specs: Core i5, 8GB RAM

STEP 2: สร้าง Virtual PLC
S7-PLCSIM Advanced → New Instance:
├── Instance name: "VirtualPLC_Unit1"
├── PLC type: S7-1516F-3 PN/DP
├── IP Address: 192.168.0.1 (virtual)
└── Click: Start

STEP 3: Download Program จาก TIA Portal
TIA Portal → PLC → Go Online → Download to Device
- เลือก virtual PLC instance ที่สร้างไว้
- Download program เหมือน real PLC

STEP 4: Connect Factory I/O
Factory I/O → File → Drivers
├── เลือก: S7-PLCSIM Advanced
├── Instance: เลือก virtual PLC ที่สร้าง
├── Map I/O tags ตาม I/O table
└── Start (F5)

STEP 5: Test
- กด RUN บน virtual PLC
- กด Start simulation ใน Factory I/O
- โปรแกรม PLC ควบคุม simulation ได้ทันที

ข้อดีของ S7-PLCSIM Advanced:
├── Native S7 communication (ไม่ต้องผ่าน Modbus)
├── Supports all TIA Portal functions
├── Can simulate PROFINET (virtual network)
├── Digital Twin ready (SIMIT integration)
└── ใช้ test ก่อน download ไป real PLC
```

---

## Step 1085: Virtual Commissioning Process

### Virtual Commissioning Workflow

```
VIRTUAL COMMISSIONING WORKFLOW:
──────────────────────────────────────────────────────────────

PHASE 1: MECHANICAL DESIGN (CAD)
├── Machine designed in CAD (SolidWorks, NX, CATIA)
├── Export as STEP/JT for NX MCD
└── Define: joints, actuators, sensors positions

PHASE 2: MECHATRONIC SIMULATION (NX MCD)
├── Import mechanical model
├── Add physics: gravity, friction, mass
├── Define: joints (prismatic, revolute)
├── Add: sensors (proximity, camera)
└── Add: actuators (motors, cylinders)

PHASE 3: PLC PROGRAMMING (TIA Portal)
├── Define I/O list from mechanical model
├── Write PLC program (offline)
├── Simulate locally (PLCSIM Basic)
└── Review logic

PHASE 4: VIRTUAL COMMISSIONING (NX MCD + PLCSIM Advanced)
├── Connect NX MCD signals to PLCSIM via OPC UA
├── Download PLC program to virtual PLC
├── Run simulation
├── Debug: program logic vs mechanical behavior
└── Iterate until behavior correct

PHASE 5: REAL COMMISSIONING
├── Real PLC + virtual machine (VC stays)
│   ← ทำเมื่อ real machine ยังสร้างไม่เสร็จ
└── Real PLC + real machine
    ← Commission ได้เร็วกว่าเดิม 50%
    เพราะ software ถูกต้องแล้วจาก virtual commissioning
```

---

## Step 1086: Hardware-in-Loop Testing

### Hardware-in-Loop (HIL) Testing

```
HIL Testing Architecture:
──────────────────────────────────────────────────────────────

SETUP:
Real PLC ←→ Real I/O Modules ←→ HIL Simulator ←→ Process Model

Real PLC: S7-1500 (hardware, full real PLC)
Real I/O: Actual field terminals
HIL Simulator: dSPACE / NI VeriStand / OPAL-RT
Process Model: Mathematical model ใน Simulink หรือ similar

ข้อแตกต่างกับ PLCSIM:
├── PLCSIM: software PLC เท่านั้น (no real I/O)
├── HIL: real PLC + real I/O hardware
│         แต่ process (motors, valves) เป็น simulation
└── HIL ใกล้เคียง real มากกว่า

ใช้ HIL เมื่อ:
├── Safety-critical applications (SIL2/SIL3)
├── Test ที่ทำบน real machine อันตราย
├── Test ต้องทำซ้ำหลายครั้ง (reproducible)
└── FDA/IEC validation requirements

HIL Test Scenarios:
├── Normal operation
├── Setpoint changes
├── Process disturbances
├── Sensor failures (open circuit, short)
├── Communication failures
├── Power failures (UPS switchover)
└── Safety logic testing (E-stop, interlocks)
```

---

## Step 1087: MATLAB/Simulink สำหรับ Process Control Design

### Process Model ใน Simulink

```
Simulink Process Model Design:
──────────────────────────────────────────────────────────────

ตัวอย่าง: CSTR (Continuous Stirred Tank Reactor)
Chemical Reaction: A → B + C (exothermic)

MASS BALANCE:
dCa/dt = (Ca_in - Ca) × (F/V) - k × Ca
where:
├── Ca: concentration of A in reactor
├── Ca_in: inlet concentration
├── F: volumetric flow rate
├── V: reactor volume
└── k: reaction rate constant (temperature dependent)

ENERGY BALANCE:
dT/dt = (F/V) × (T_in - T) + (-ΔH_r/(ρ × Cp)) × k × Ca
         - (UA/(ρ × Cp × V)) × (T - T_jacket)
where:
├── T: reactor temperature
├── T_in: inlet temperature
├── T_jacket: cooling jacket temperature
├── ΔH_r: heat of reaction
└── UA: heat transfer coefficient × area

Simulink Block Diagram:
┌──────────────────────────────────────────────────────────────────┐
│  F_in ──→ [Gain: 1/V] ──→ (+)──→ [Integrator] ──→ Ca          │
│  Ca_in ──→ [Gain: F/V] ──→ (+)                                  │
│               (-) ←──[k(T)] ←─────────────────────────────────┤│
│                              │                                   ││
│  T_in ──→ [Gain: F/V] ──→ (+)──→ [Integrator] ──→ T           ││
│          (-ΔH/(ρCp)×k(T)×Ca)→(+)                               ││
│          (-UA/(ρCpV))×(T-Tj)→(+)                               ││
│                                                                   ││
│  T_jacket ──→ [Cooling Control PID] ──→ T_jacket               ││
└──────────────────────────────────────────────────────────────────┘

PLC Code Generation (Simulink PLC Coder):
Simulink Model → Auto-generate Structured Text / Ladder
─────────────────────────────────────────────────────
├── ใช้ Simulink PLC Coder toolbox
├── Configure target PLC (Siemens, Rockwell, etc.)
├── Generate code → สร้างไฟล์ .scl หรือ .L5X
├── Import ไปยัง TIA Portal หรือ Studio 5000
└── Test ด้วย PLCSIM หรือ HIL
```

---

## Step 1088: Digital Twin สำหรับ Predictive Maintenance

### Pump Digital Twin ตัวอย่าง

```
PUMP DIGITAL TWIN ARCHITECTURE:
──────────────────────────────────────────────────────────────

Physical Pump:
├── Vibration sensors (X, Y, Z axes)
├── Temperature sensors (bearing, motor winding)
├── Current sensors (3-phase)
├── Suction/discharge pressure
├── Flow measurement
└── Speed encoder

Digital Twin Model:
├── Hydraulic model: Q = f(N, ΔP) (pump curve)
├── Bearing model: vibration ∝ wear
├── Thermal model: T = f(current, ambient)
└── Efficiency model: η = f(Q, N)

Health Indicators:
─────────────────────────────────────────────────────────────
KPI                  Healthy        Warning        Critical
──────────────────   ───────────    ─────────      ─────────
Vibration (mm/s)     < 2.0          2.0 - 4.5      > 4.5
Bearing Temp (°C)    < 60           60 - 80         > 80
Current (A)          < 1.1×FLA      1.1-1.25×FLA   >1.25×FLA
Efficiency (%)       > 85           75-85           < 75
Suction Pressure     > 0.5 bar      0.3-0.5 bar    < 0.3 bar

Digital Twin Update Loop:
1. อ่านข้อมูลจาก PLC (1 second cycle)
2. Update model states
3. Compare model vs actual (residual)
4. If residual > threshold: anomaly detected
5. Update health indicators
6. Calculate RUL (Remaining Useful Life)
7. Generate maintenance recommendation
```

### Digital Twin Implementation ใน Python

```python
# Digital Twin สำหรับ Centrifugal Pump
# ─────────────────────────────────────────────────────────────

import numpy as np
from dataclasses import dataclass
from typing import Optional

@dataclass
class PumpPhysicalData:
    """ข้อมูลจาก Sensors จริง"""
    timestamp: float
    flow_rate: float       # m3/h
    suction_pressure: float # bar
    discharge_pressure: float # bar
    vibration_rms: float   # mm/s
    bearing_temp: float    # °C
    motor_current: float   # A
    speed: float           # RPM

class PumpDigitalTwin:
    """Digital Twin สำหรับ Centrifugal Pump"""
    
    def __init__(self, pump_id: str, nominal_flow: float, rated_power: float):
        self.pump_id = pump_id
        self.nominal_flow = nominal_flow     # m3/h
        self.rated_power = rated_power       # kW
        self.health_score = 100.0            # 0-100%
        self.rul_hours = 8760.0              # 1 year initially
        
        # Model parameters (จะ calibrate จากข้อมูลจริง)
        self.wear_factor = 0.0               # Bearing wear accumulator
        self.vibration_baseline = None       # Baseline vibration
        self.efficiency_baseline = None      # Baseline efficiency
        
        # History
        self.data_history = []
    
    def update(self, physical_data: PumpPhysicalData):
        """Update twin กับข้อมูลจาก sensor"""
        self.data_history.append(physical_data)
        
        # ตั้ง baseline ถ้ายังไม่มี (ช่วง 24 ชม. แรก)
        if len(self.data_history) == 1440:  # 24 ชม. × 60 นาที
            self._calibrate_baseline()
        
        # Update health indicators
        health_indicators = self._calculate_health(physical_data)
        
        # Update health score
        self.health_score = min(health_indicators.values())
        
        # Update RUL
        self.rul_hours = self._estimate_rul()
        
        return health_indicators
    
    def _calculate_health(self, data: PumpPhysicalData) -> dict:
        """คำนวณ health score สำหรับแต่ละ indicator"""
        health = {}
        
        # 1. Vibration Health (0-100%)
        if self.vibration_baseline:
            vib_ratio = data.vibration_rms / self.vibration_baseline
            if vib_ratio < 1.5:
                health['vibration'] = 100.0
            elif vib_ratio < 2.5:
                health['vibration'] = 100 - (vib_ratio - 1.5) * 50
            else:
                health['vibration'] = max(0, 50 - (vib_ratio - 2.5) * 25)
        else:
            health['vibration'] = 100.0 if data.vibration_rms < 2.0 else 70.0
        
        # 2. Temperature Health (0-100%)
        if data.bearing_temp < 60:
            health['temperature'] = 100.0
        elif data.bearing_temp < 80:
            health['temperature'] = 100 - (data.bearing_temp - 60) * 2.5
        else:
            health['temperature'] = max(0, 50 - (data.bearing_temp - 80) * 5)
        
        # 3. Efficiency Health (0-100%)
        hydraulic_power = (data.discharge_pressure - data.suction_pressure) * \
                          data.flow_rate / 36.0  # kW
        if data.motor_current > 0:
            input_power = data.motor_current * 380 * 1.732 * 0.9 / 1000  # kW, pf=0.9
            efficiency = (hydraulic_power / input_power) * 100 if input_power > 0 else 0
            
            if self.efficiency_baseline:
                eff_drop = self.efficiency_baseline - efficiency
                health['efficiency'] = max(0, 100 - eff_drop * 5)
            else:
                health['efficiency'] = 100.0 if efficiency > 70 else 60.0
        else:
            health['efficiency'] = 100.0
        
        return health
    
    def _estimate_rul(self) -> float:
        """Estimate Remaining Useful Life in hours"""
        # Simple linear degradation model
        # TODO: Replace with ML model
        current_degradation = 100 - self.health_score
        if current_degradation < 5:
            return 8760  # > 1 year
        elif current_degradation < 20:
            return 4380  # ~6 months
        elif current_degradation < 40:
            return 720   # ~1 month
        else:
            return 168   # ~1 week
    
    def get_maintenance_recommendation(self) -> str:
        """สร้าง recommendation"""
        if self.health_score >= 85:
            return f"ปกติ - ตรวจตามกำหนด"
        elif self.health_score >= 70:
            return f"แนะนำตรวจสอบ - RUL: {self.rul_hours:.0f} ชม."
        elif self.health_score >= 50:
            return f"⚠️ วางแผนซ่อมบำรุง - RUL: {self.rul_hours:.0f} ชม."
        else:
            return f"🚨 ต้องการซ่อมด่วน - RUL: {self.rul_hours:.0f} ชม."
    
    def _calibrate_baseline(self):
        """ตั้งค่า baseline จาก historical data"""
        recent = self.data_history[-1440:]  # 24 ชม. ล่าสุด
        self.vibration_baseline = np.mean([d.vibration_rms for d in recent])
        print(f"Pump {self.pump_id}: Baseline calibrated. Vib: {self.vibration_baseline:.2f} mm/s")
```

---

## Step 1089: Virtual Commissioning ตัวอย่างสมบูรณ์

### Case Study: Bottling Line Virtual Commissioning

```
PROJECT: ไลน์บรรจุน้ำดื่ม (Water Bottling Line)
──────────────────────────────────────────────────────────────

PHYSICAL MACHINES:
1. Bottle Conveyor (สายพาน)
2. Filling Station (เติมน้ำ 12 หัวพร้อมกัน)
3. Capping Station (ปิดฝา)
4. Labeling Station (ติดฉลาก)
5. Quality Inspection (vision system)
6. Packaging (บรรจุลัง)

VIRTUAL COMMISSIONING SCHEDULE:
Week 1-2: Mechanical 3D model (SolidWorks)
Week 3-4: Import to NX MCD, add physics
Week 5-6: PLC programming (TIA Portal + PLCSIM Advanced)
Week 7:   Integration test (NX MCD + PLCSIM)
Week 8:   Bugs fixed, full simulation cycle run
Week 9-10: Real machine arrived, fast commissioning
            (program already validated!)

BUGS FOUND IN VIRTUAL COMMISSIONING:
1. Bottle jam at curve conveyor
   → Fix: ลด conveyor speed, เพิ่ม guide rail
2. Timing issue filling → capping
   → Fix: เพิ่ม timer delay 0.5 seconds
3. Vision system false reject ที่ high speed
   → Fix: ลด line speed, ปรับ camera trigger timing
4. Emergency stop reset ไม่ clear คิว
   → Fix: เพิ่ม reset logic ใน program

RESULT:
├── Real commissioning: 3 days (from 3 weeks estimated)
├── First production run: 99% yield (vs typical 85% first run)
├── Savings: ฿ 2,000,000 (reduced downtime, less rework)
└── Training: Operators trained on simulation before real machine
```

---

## Step 1090: ROI ของ Digital Twin

### Business Case for Digital Twin

```
Digital Twin ROI Analysis:
══════════════════════════════════════════════════════════════

INVESTMENT:
──────────────────────────────────────────────────────────────
Software:
├── NX MCD (Siemens): ฿ 600,000/license
├── PLCSIM Advanced: ฿ 150,000/license
├── Azure Digital Twins: ฿ 200,000/year
└── Analytics platform: ฿ 300,000/year

Services:
├── Model creation: ฿ 1,500,000 (first machine)
├── Integration work: ฿ 500,000
└── Training: ฿ 200,000

Total Investment (Year 1): ฿ 3,450,000

BENEFITS:
──────────────────────────────────────────────────────────────
Virtual Commissioning (per new machine):
├── Reduced commissioning time: 2 weeks → 3 days
├── Saved cost: 11 days × ฿ 200,000/day = ฿ 2,200,000
└── ← ROI จากแค่ 1 machine จ่ายคืนได้!

Predictive Maintenance:
├── Reduced unplanned downtime: 5 incidents × 8 hrs × ฿300k/hr
├── Value: ฿ 12,000,000/year
└── Maintenance cost reduction: 20% = ฿ 1,000,000/year

Energy Optimization:
├── Process optimization from twin insights
└── 3% energy savings = ฿ 600,000/year

Quality Improvement:
├── First-time-right rate: 98% vs 92%
└── Scrap reduction: ฿ 800,000/year

Total Annual Benefit: ฿ 16,600,000
Payback Period: < 3 months
5-year NPV: ฿ 60,000,000+
```

---

## Step 1091: Azure Digital Twins Service

### Azure Digital Twins Implementation

```
Azure Digital Twins (ADT) - Cloud-based Twin Platform:
──────────────────────────────────────────────────────────────

CONCEPTS:
├── Twins: Digital representations (PLC, Motor, Pump, etc.)
├── Relationships: ความสัมพันธ์ระหว่าง twins
├── Models (DTDL): Digital Twin Definition Language
└── Twin Graph: กราฟที่แสดงทุก twin และ relationship

DTDL Model ตัวอย่าง (Motor):
{
  "@context": "dtmi:dtdl:context;2",
  "@id": "dtmi:factory:Motor;1",
  "@type": "Interface",
  "displayName": "Motor",
  "contents": [
    {
      "@type": ["Property", "Temperature"],
      "name": "windingTemperature",
      "schema": "double",
      "unit": "degreeCelsius"
    },
    {
      "@type": ["Property", "Current"],
      "name": "phaseCurrent",
      "schema": "double",
      "unit": "ampere"
    },
    {
      "@type": "Property",
      "name": "healthScore",
      "schema": "double"
    },
    {
      "@type": "Relationship",
      "name": "drivesEquipment",
      "target": "dtmi:factory:Pump;1"
    }
  ]
}

Data Flow สู่ ADT:
PLC → IoT Hub → Azure Functions → ADT API → Update twin properties

Query ใน ADT:
// Find all motors with health < 70%
SELECT * FROM DIGITALTWINS WHERE IS_OF_MODEL('dtmi:factory:Motor;1')
AND healthScore < 70.0
```

---

## Step 1092: Simulation-Based Operator Training

### Operator Training Simulator

```
OPERATOR TRAINING SIMULATOR DESIGN:
──────────────────────────────────────────────────────────────

ทำไมต้องมี Training Simulator?
├── Operator สามารถฝึกโดยไม่กระทบ production
├── Test สถานการณ์ฉุกเฉินได้อย่างปลอดภัย
├── New operator สามารถฝึกได้ก่อน SIT หรือ OJT
└── Procedure validation (ทำถูกต้องไหม?)

Architecture:
──────────────────────────────────────────────────────────────
Training Simulator:
├── Instructor Station: กำหนด scenario, inject faults
├── Trainee Station: HMI เหมือน real (Thin client)
├── Process Simulation: HYSYS/Aspen/Python model
└── Emulated PLC: PLCSIM Advanced or physical PLC

Typical Training Scenarios:
SCENARIO 1: Normal Startup
├── เปิด feed pump
├── ตั้ง setpoints
├── Monitor until steady state
└── Verify product quality

SCENARIO 2: Emergency Shutdown
├── Inject: high temperature alarm
├── Trainee: identify cause
├── Trainee: initiate controlled shutdown
└── Evaluate: correct steps within time limit

SCENARIO 3: Equipment Failure
├── Inject: pump bearing failure (vibration increase)
├── Trainee: identify failing pump
├── Trainee: switchover to spare
└── Evaluate: downtime minimized

Evaluation System:
├── Time to detect abnormal condition
├── Correct procedures followed (Y/N checklist)
├── Communication with shift supervisor
└── Safety actions taken correctly
```

---

## Step 1093: Digital Twin สำหรับ Process Optimization

### Process Optimization ด้วย Digital Twin

```
OPTIMIZATION EXAMPLE: Distillation Column
──────────────────────────────────────────────────────────────

Physical Column:
├── Feed: crude oil mixture
├── Products: naphtha, kerosene, diesel, heavy oil
├── Key Control: temperature profile, reflux ratio, reboiler

Digital Twin Model:
├── Rigorous thermodynamic simulation (Aspen)
├── Updated every 5 minutes with plant data
├── Calibrated to match actual product compositions

OPTIMIZATION LOOP:
1. Twin receives current plant data
2. Run optimization (minimize energy, maximize yield)
3. Calculate optimal: reflux ratio, steam rate, feed temperature
4. Operator reviews recommendation
5. Operator implements changes
6. Monitor results, update model

Results ที่เคยทำได้จริง:
├── Energy savings: 5-8%
├── Product yield improvement: 1-2%
├── Reduced off-spec product: 30%
└── ROI: 6 months

Integration with APC (Advanced Process Control):
Digital Twin → MPC → PLC setpoints (automatic, faster loop)
├── MPC uses twin model for prediction
├── Twin provides calibrated model parameters
└── Continuous model update (EKF, UKF algorithms)
```

---

## Step 1094: Digital Twin Standards และ Frameworks

### Industry Standards สำหรับ Digital Twin

```
DIGITAL TWIN STANDARDS:
══════════════════════════════════════════════════════════════

ISO 23247: Digital Twin Framework for Manufacturing
├── 4 entities: Observable Manufacturing Element,
│               Data Collection & Device Control,
│               Digital Twin Instance,
│               User Entity
└── Focus on manufacturing digital twins

IEC 62890: Life-cycle Management for Systems
└── Digital twin as part of asset lifecycle

Asset Administration Shell (AAS):
├── German Standard (Plattform Industrie 4.0)
├── Standardized digital representation of asset
├── JSON/XML format
├── Open source implementation available
└── Compatible with: Siemens, Beckhoff, Festo

AAS Structure:
Asset Administration Shell
├── Asset Information
│   ├── Global Asset ID
│   ├── Asset Kind (Type vs Instance)
│   └── Thumbnail
├── Submodels
│   ├── Technical Data
│   ├── Operational Data (measurements)
│   ├── Nameplate
│   └── Documentation
└── Views (different perspectives)
```

---

## Step 1095: Siemens SIMATIC Virtual PLC + HMI

### ตัวอย่าง Complete Virtual System

```
VIRTUAL COMMISSIONING SETUP สำหรับ Packaging Machine:
──────────────────────────────────────────────────────────────

HARDWARE (Virtual):
├── S7-PLCSIM Advanced v5.0
│   ├── S7-1516F-3 PN/DP (Safety PLC)
│   └── ET200SP distributed I/O
├── WinCC RT Advanced (HMI simulation)
└── Factory I/O (3D factory scene)

SOFTWARE CONNECTIONS:
S7-PLCSIM Advanced ←→ TIA Portal (download, monitor, diagnose)
S7-PLCSIM Advanced ←→ Factory I/O (virtual I/O)
S7-PLCSIM Advanced ←→ WinCC RT Advanced (HMI tags)
TIA Portal ←→ WinCC RT Advanced (tag sync)

TESTING SEQUENCE:
1. Download PLC program to PLCSIM Advanced
2. Download HMI to WinCC RT Advanced simulation
3. Start Factory I/O simulation
4. Test all operator screens match simulation behavior
5. Test all alarms: inject faults in Factory I/O
6. Test safety: trigger E-stop, verify STO
7. Test interlocks: all permissives work correctly
8. Performance test: 8-hour continuous run

ISSUES FOUND AND FIXED (Virtual):
├── HMI screen: wrong tag mapping (pump status inverted)
├── Alarm: temperature alarm setpoint unit wrong (F vs C)
├── Safety: E-stop reset needed 2 key turns (1 is enough)
├── Interlock: wrong permissive for conveyor start
└── Performance: memory leak in HMI after 4 hours → patched
```

---

## Step 1096: Digital Twin for Energy Management

### Energy Digital Twin

```
ENERGY DIGITAL TWIN CONCEPT:
──────────────────────────────────────────────────────────────

Energy Flow Model:
Power Meter (Main) → Distribution Model → Equipment Models

Equipment Models:
├── Motor: P = √3 × V × I × cos(φ)
├── Compressor: P = Q × ΔP / (η × 6000)
├── Heater: P = I² × R
└── Chiller: COP = Q_cold / P_input

Digital Twin Calculations:
1. Real-time energy balance: Meter vs Model
   → If discrepancy > 5%: leak or theft detection
2. Energy efficiency tracking per equipment
   → Identify degrading equipment
3. Demand forecasting
   → Predict peak load for demand management
4. "What-if" analysis
   → If we shut compressor 2 → what happens to system?
5. Optimization
   → Find optimal equipment loading for minimum energy

Result Dashboard:
┌───────────────────────────────────────────────────────────┐
│  ENERGY TWIN - LAST HOUR                                  │
│  Total Consumption: 245 kWh  (Budget: 250 kWh)  ✓        │
│                                                           │
│  Equipment Efficiency:                                    │
│  Chiller 1: COP 5.2 ✓    Chiller 2: COP 3.8 ⚠️ (↓15%)  │
│  Air Comp 1: 98% ✓        Air Comp 2: 91% ✓              │
│                                                           │
│  Anomaly Detected:                                        │
│  ⚠️ Chiller 2 efficiency degraded 15% vs baseline        │
│     Recommendation: Clean condenser coils                 │
└───────────────────────────────────────────────────────────┘
```

---

## Step 1097: Collaborative Digital Twin

### Multi-stakeholder Digital Twin

```
COLLABORATIVE DIGITAL TWIN สำหรับ Supply Chain:
══════════════════════════════════════════════════════════════

แต่ละ stakeholder มี view ของ twin ที่แตกต่างกัน:

OPERATIONS VIEW:
├── Real-time production status
├── Equipment health
├── Batch tracking
└── Quality metrics

MAINTENANCE VIEW:
├── Equipment health scores
├── Work orders
├── Spare parts inventory (linked)
└── Maintenance history

MANAGEMENT VIEW:
├── OEE (Overall Equipment Effectiveness)
├── Production vs plan
├── Energy cost
└── Quality KPIs

ENGINEERING VIEW:
├── Detailed process parameters
├── Control loop performance
├── Configuration history
└── Change management

SUPPLIER VIEW (Limited access):
├── Consumption rate of their equipment/consumables
├── Service schedules
└── Technical support data

Access Control:
├── Role-based data filtering
├── Supplier sees only their equipment
├── Operations sees plant-wide view
└── Management sees aggregated KPIs
```

---

## Step 1098: Challenges และ Best Practices

### ความท้าทายในการ Implement Digital Twin

```
CHALLENGES:
══════════════════════════════════════════════════════════════

1. DATA QUALITY
   Problem: Sensor data ไม่ดี → model ไม่แม่นยำ
   Solution: 
   ├── Data validation (range check, rate-of-change check)
   ├── Sensor calibration program
   └── Bad value handling (substitution, flagging)

2. MODEL FIDELITY
   Problem: Model ไม่ตรงกับ reality
   Solution:
   ├── Regular model calibration
   ├── Physics-based vs data-driven hybrid
   └── Uncertainty quantification

3. ORGANIZATIONAL
   Problem: IT/OT ไม่ร่วมมือ, ไม่มีผู้รับผิดชอบ twin
   Solution:
   ├── Define "twin owner" ชัดเจน
   ├── Budget สำหรับ maintenance ของ twin
   └── Success metrics ที่วัดได้

4. SCALABILITY  
   Problem: เริ่มจาก 1 machine → 100 machines คือ 100× complexity
   Solution:
   ├── Template-based twin creation
   ├── Standardized DTDL models
   └── Auto-provisioning pipeline

5. CYBERSECURITY
   Problem: Twin เชื่อมต่อกับ OT → risk
   Solution:
   ├── Read-only connection from OT (data diode)
   ├── No command from twin to real PLC (except APC)
   └── Security by design
```

---

## Step 1099: Getting Started with Digital Twin

### Roadmap สำหรับ Digital Twin Program

```
DIGITAL TWIN ROADMAP:
══════════════════════════════════════════════════════════════

PHASE 0: FOUNDATION (Month 1-3)
├── Asset inventory และ criticality ranking
├── Sensor assessment (ติดเพิ่มถ้าจำเป็น)
├── Data infrastructure (historian, network)
├── Team training (digital twin concepts)
└── Pilot machine selection

PHASE 1: PILOT (Month 4-9)
├── สร้าง digital twin สำหรับ 1 critical machine
├── Descriptive twin (real-time monitoring)
├── Validate data accuracy
├── Train team on tools
└── Document lessons learned

PHASE 2: EXPAND (Month 10-18)
├── ขยายไปยัง 5-10 machines
├── Informative twin (add historical analysis)
├── Predictive models (vibration, efficiency)
├── Integrate with CMMS (SAP PM)
└── ROI measurement

PHASE 3: ADVANCED (Month 19-36)
├── Plant-wide deployment
├── Predictive maintenance fully operational
├── Process optimization twin
├── Energy digital twin
└── Virtual commissioning standard process

PHASE 4: AUTONOMOUS (Month 37+)
├── APC (Advanced Process Control) integration
├── Self-optimizing control loops
├── Autonomous anomaly response
└── Continuous model improvement (ML)

SUCCESS METRICS:
├── Unplanned downtime reduction: target 50%
├── Commissioning time reduction: target 40%
├── Energy savings: target 5-10%
└── ROI: target 200% in Year 1
```

---

## Step 1100: Lab Project - Digital Twin Commissioning

### Lab: Virtual Commissioning สำหรับ Chemical Dosing System

**โจทย์**: ระบบเติมสารเคมีสำหรับโรงบำบัดน้ำ
- Mixing tank 1000L
- 3 chemical dosing pumps (acid, base, chlorine)
- pH sensor, conductivity sensor, level sensor
- Outlet to distribution

**Tasks**:

```
PART 1: Process Simulation (40%)
─────────────────────────────────────────────────────────────
1. สร้าง mathematical model ใน Python/MATLAB:
   - Tank volume balance: dV/dt = Q_in - Q_out
   - Chemical mixing: dC/dt = (Q_in × C_in - Q_out × C) / V
   - pH model (simplified titration curve)

2. Test simulation:
   - Startup from empty
   - Steady state operation
   - Setpoint change response

PART 2: PLC Virtual Commissioning (40%)
─────────────────────────────────────────────────────────────
1. PLC Program (TIA Portal):
   - Level control (with setpoints)
   - pH control (with dosing pumps)
   - Conductivity monitoring
   - Alarm management

2. Connect PLCSIM Advanced กับ Python simulation:
   - Use OPC UA สำหรับ data exchange
   - Python updates PLC inputs (sensor values)
   - PLC outputs control simulation (pumps)

3. Test cases:
   - Normal operation
   - High pH disturbance
   - Low level alarm
   - Emergency stop

PART 3: Monitoring Dashboard (20%)
─────────────────────────────────────────────────────────────
1. Grafana dashboard:
   - pH trend (last 4 hours)
   - Chemical dosing rates
   - Level trend
   - Alarm list

2. Digital twin health metrics:
   - Pump efficiency (compare model vs actual)
   - Dosing accuracy (% error)
   - System stability index

EVALUATION:
├── Simulation accuracy: model vs real within 5%
├── PLC control: pH within ±0.1 pH units
├── Virtual commissioning bugs found: document all
├── Dashboard: real-time display < 5s delay
└── Final report: complete and professional
```

---

## สรุป Part 55

| ระดับ Digital Twin | ใช้ทำอะไร |
|-------------------|----------|
| Descriptive | Real-time monitoring, status visualization |
| Informative | Historical analysis, troubleshooting |
| Predictive | Predictive maintenance, RUL |
| Autonomous | Self-optimization, APC integration |

---

## Navigation

- [← Part 54: PLC to Cloud Integration](./part54_plc_to_cloud.md)
- [→ Part 56: Advanced Diagnostics and PROFINET](./part56_advanced_diagnostics.md)
- [↑ สารบัญหลัก](../README.md)
