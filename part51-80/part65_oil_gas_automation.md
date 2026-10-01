# Part 65: Oil and Gas Automation (Steps 1281-1300)

## บทนำ: Oil and Gas - ความปลอดภัยสูงสุด

Oil and Gas automation มีความเสี่ยงสูงที่สุด:
- **Flammable/explosive materials**: น้ำมัน, ก๊าซ, LPG, H2S
- **High pressure**: 100-1,000+ bar
- **High temperature**: 600°C+ ใน refineries
- **Environmental damage**: spill ทำลายสิ่งแวดล้อม

ผลลัพธ์ถ้า fail:
- **Piper Alpha (1988)**: ระเบิด offshore platform → 167 คนเสียชีวิต
- **Texas City Refinery (2005)**: 15 คนเสียชีวิต, 180 บาดเจ็บ
- **Deepwater Horizon (2010)**: 11 คน, spill ขนาดใหญ่ที่สุดในประวัติศาสตร์

ดังนั้น Safety Instrumented Systems (SIS) เป็นสิ่งจำเป็น

---

## Step 1281: Process Safety Management

### IEC 61511 (Functional Safety)

```
IEC 61511 FRAMEWORK:
──────────────────────────────────────────────────────────────

IEC 61511 = Functional Safety for Process Industry
ครอบคลุม: SIS (Safety Instrumented Systems)
ต่างจาก IEC 62061/ISO 13849 (machinery safety)

KEY CONCEPTS:
1. HAZARD IDENTIFICATION (HAZOP):
   - ระบุ Process Hazards ทั้งหมด
   - HAZOP: Hazard and Operability Study
   - "What if flow is too high?" → consequence + protection

2. RISK ASSESSMENT:
   - Likelihood × Consequence = Risk
   - Risk reduction needed = Risk / Tolerable Risk

3. SAFETY INSTRUMENTED FUNCTION (SIF):
   - Specific safety function ที่ SIS ทำ
   - เช่น: "High High Pressure → Close feed valve + Stop pump"

4. SIL (Safety Integrity Level):
   SIL 1: PFD 0.1-0.01      (10-100x risk reduction)
   SIL 2: PFD 0.01-0.001    (100-1,000x)
   SIL 3: PFD 0.001-0.0001  (1,000-10,000x)
   SIL 4: PFD < 0.0001      (> 10,000x) ← rare, nuclear

   PFD = Probability of Failure on Demand

5. LAYERS OF PROTECTION:
   Process Design → BPCS → Alarms → SIS → Mechanical Protection → Mitigation
```

---

## Step 1282: BPCS vs SIS Architecture

### แยก BPCS และ SIS

```
BPCS vs SIS SEPARATION:
──────────────────────────────────────────────────────────────

BPCS (Basic Process Control System):
├── ควบคุม process ปกติ (temperature, pressure, flow)
├── DCS หรือ PLC ทั่วไป (Siemens, Honeywell, ABB)
└── ไม่ต้องการ SIL rating

SIS (Safety Instrumented System):
├── ทำงานเฉพาะเมื่อ process เกิน limit อันตราย
├── MUST be independent from BPCS
├── Use dedicated Safety PLC (SIL-rated)
└── Examples: Triconex, Hima, Pilz, ABB AC500-S

WHY SEPARATION?
BPCS อาจ fail ทุกวัน (software update, configuration change)
ถ้า SIS อยู่ใน same system → เมื่อ BPCS fail → SIS ก็ fail ด้วย!
Separate systems ลด Common Cause Failure (CCF)

ARCHITECTURE (Offshore Platform):
──────────────────────────────────────────────────────────────

Level 3: MCC (Machinery Control Center)
Level 2: ESD (Emergency Shutdown) → TRICONEX (SIL 3 capable)
         F&G (Fire & Gas) → Dedicated system
Level 2: DCS (Process Control) → Honeywell Experion
Level 1: Field Instruments (separated transmitters!)

SEPARATION OF INSTRUMENTS:
BPCS transmitters ≠ SIS transmitters
ถ้า BPCS pressure transmitter ไม่ทำงาน → BPCS control ล้มเหลว
แต่ SIS pressure transmitter (separate!) ยังทำงาน → ESD triggered

RULE: SIS sensors ต้อง SEPARATE จาก BPCS sensors
(ไม่ share transmitters ระหว่าง BPCS และ SIS)
```

---

## Step 1283: Cause and Effect Matrix

### Cause and Effect Matrix

```
CAUSE AND EFFECT MATRIX:
──────────────────────────────────────────────────────────────

C&E Matrix = ตารางแสดง cause → effect relationship
ใช้ออกแบบ SIS logic

FORMAT:
Columns (Causes/Initiators):
- Process Deviations: High High Pressure, Low Low Level, etc.
- Equipment Faults: Pump failure, Valve fail
- External: Fire, Gas detection, Manual ESD

Rows (Effects/Actions):
- Valve close/open
- Pump stop
- Isolation commands
- Alarm activation

EXAMPLE C&E MATRIX (Simplified):
                           │ HH Pressure │ LL Level │ Fire Z1 │ Manual ESD │
───────────────────────────┼─────────────┼──────────┼─────────┼────────────┤
Inlet Valve V101 Close     │     X       │   X      │   X     │     X      │
Feed Pump P101 Stop        │     X       │   X      │   X     │     X      │
Blowdown Valve V201 Open   │     X       │          │         │     X      │
Flare System Enable        │     X       │          │         │     X      │
Deluge Activate Zone 1     │             │          │   X     │            │
ESD Alarm Panel            │     X       │   X      │   X     │     X      │

X = this cause triggers this effect
```

---

## Step 1284: SIL Verification

### ตรวจสอบ SIL Level

```
SIL VERIFICATION:
──────────────────────────────────────────────────────────────

SIL Verification ตรวจสอบว่า SIF design บรรลุ SIL ที่ต้องการ

ใช้ PFD (Probability of Failure on Demand) calculation:
PFD_system = PFD_sensor + PFD_logic_solver + PFD_final_element

Example: SIF "High Pressure → Shut Feed Valve" (Required SIL 2)
Target PFD: < 0.01

Components:
├── Sensor (Pressure Transmitter): 
│   PFD = λ × T_proof / 2 = 1×10⁻⁶ × 8760/2 = 0.00438
│   (MTTF: 1 million hours, annual proof test)
├── Logic Solver (Safety PLC):
│   PFD = 1×10⁻⁷ × 8760/2 = 0.000438 (SIL-certified F-CPU)
└── Final Element (Fail-Safe Valve):
    PFD = 2×10⁻⁶ × 8760/2 = 0.00876

Total PFD = 0.00438 + 0.000438 + 0.00876 = 0.01358

→ 0.01358 > 0.01 (SIL 2 requirement)
→ ไม่บรรลุ SIL 2! ต้องปรับปรุง

แนวทางแก้ไข:
1. เพิ่ม proof test frequency (ทุก 6 เดือน แทน 1 ปี)
2. เพิ่มจำนวน sensors (2oo3 voting ลด PFD sensor)
3. เลือก valve ที่ reliable กว่า (MTTF สูงกว่า)

VOTING LOGIC:
1oo1: 1 out of 1 → single sensor (low availability, high safety)
2oo2: 2 out of 2 → both must agree (high availability, low safety)
2oo3: 2 out of 3 → 2 must agree (BEST BALANCE - high SIL + availability)
1oo2D: 1 out of 2 with diagnostics (fault tolerant)
```

---

## Step 1285: Safety PLC Programming

### Programming สำหรับ Safety PLC

```
SAFETY PLC PROGRAMMING RULES:
──────────────────────────────────────────────────────────────

Safety PLC ต้องปฏิบัติตาม IEC 61511 + IEC 61508:

1. PROVEN IN USE SOFTWARE:
   ใช้ standard FBs ที่ certified (SIL-certified libraries)
   ห้ามใช้ complex algorithms ที่ยังไม่ได้ validate

2. DERATING:
   SIL 2 SIF ต้องการ:
   - SIL 3 certified CPU (derating 1 level for software)
   - หรือ SIL 2 certified with formal verification

3. FAIL-SAFE DESIGN:
   ทุก output ต้อง fail-safe:
   - Valve: fail closed (de-energize = close)
   - Pump: fail stop (de-energize = stop)
   - ไม่ใช้ fail-as-is ใน safety critical

4. WATCHDOG AND SELF-DIAGNOSTICS:
   Safety PLC ต้อง continuous self-test:
   - Memory check
   - I/O diagnostic
   - Power supply monitoring
   - Response time monitoring

5. CHANGE MANAGEMENT:
   ทุก change ต้องมี:
   - Impact assessment
   - Formal change control
   - Re-testing after change
   - SIL re-verification if significant

// SAFETY FUNCTION IMPLEMENTATION (Siemens F-CPU):
FUNCTION_BLOCK FB_HighPressureESD [SAFETY]
VAR_INPUT
    rPressure_1:    REAL;       // Transmitter 1
    rPressure_2:    REAL;       // Transmitter 2
    rPressure_3:    REAL;       // Transmitter 3
    rHH_Limit:      REAL := 100.0;  // High-High limit (bar)
END_VAR
VAR_OUTPUT
    bESD_Trigger:   BOOL;       // Trip output
    bVote_1:        BOOL;
    bVote_2:        BOOL;
    bVote_3:        BOOL;
END_VAR

// 2oo3 VOTING LOGIC:
// bVote_1 := rPressure_1 >= rHH_Limit;
// bVote_2 := rPressure_2 >= rHH_Limit;
// bVote_3 := rPressure_3 >= rHH_Limit;
// 
// 2-out-of-3: trigger if 2 or more vote
// bESD_Trigger := (bVote_1 AND bVote_2) OR
//                 (bVote_1 AND bVote_3) OR
//                 (bVote_2 AND bVote_3);

// NOTE: In Safety FBs on F-CPU:
// ใช้ F-compliant instructions เท่านั้น
// ทุก calculation ต้องใน safety context
// Data path จาก sensors ต้อง F-rated (PROFIsafe)
```

---

## Step 1286: ESD (Emergency Shutdown) Logic

### ระบบ Emergency Shutdown

```
ESD SYSTEM DESIGN:
──────────────────────────────────────────────────────────────

ESD Hierarchy:
Level 1 (Highest Priority): Platform/Plant Shutdown (SD-1)
  → Everything stops, all valves isolate, blowdown
Level 2: Process Unit Shutdown (SD-2)  
  → Specific unit isolates
Level 3: Equipment Shutdown (SD-3)
  → Single equipment stops
Level 4: Alarm Only (SD-4)
  → Alert operator, no automatic action

ESD CAUSE LIST:
SD-1 CAUSES (Platform Shutdown):
├── Manual ESD button (any location)
├── Fire detection Zone A (confirmed)
├── Gas detection: >100% LEL any zone
├── Loss of platform power
└── Structural failure signal

SD-2 CAUSES (Train A Shutdown):
├── HH Pressure separator
├── LL Level separator
├── HH Temperature heater
└── Manual process shutdown button (Train A)

// ESD MATRIX IMPLEMENTATION
FUNCTION_BLOCK FB_ESD_Matrix
VAR_INPUT
    // SD-1 Causes
    bManual_ESD_1:      BOOL;   // Manual ESD (any location)
    bFire_ZoneA:        BOOL;   // Confirmed fire Zone A
    bGas_HH_AnyZone:    BOOL;   // Gas > 100% LEL
    
    // SD-2 Causes (Train A)
    bPressure_HH:       BOOL;   // Separator HH pressure
    bLevel_LL:          BOOL;   // Separator LL level
    
    // Resets
    bReset_SD1:         BOOL;   // Reset SD-1 (supervisor)
    bReset_SD2:         BOOL;   // Reset SD-2
END_VAR

VAR_OUTPUT
    // SD-1 Actions
    xValve_ProductIsolation:    BOOL;   // Platform boundary isolation
    xValve_FlareIsolation:      BOOL;   // Flare header valve
    xPump_AllStop:              BOOL;   // All pumps STOP
    xBlowdown_Valve:            BOOL;   // Blowdown valve OPEN
    
    // SD-2 Actions (Train A)  
    xInletValve_TrainA:         BOOL;   // Inlet valve TrainA (CLOSE)
    xPump_TrainA:               BOOL;   // Train A pump (STOP)
    
    bSD1_Active:                BOOL;   // SD-1 latched
    bSD2_Active:                BOOL;   // SD-2 latched
END_VAR

VAR
    bSD1_SET:       BOOL;
    bSD2_SET:       BOOL;
END_VAR

// SD-1: Set (latch) on any SD-1 cause
// bSD1_SET := bManual_ESD_1 OR bFire_ZoneA OR bGas_HH_AnyZone;
// SR latch:
// bSD1_Active := (bSD1_Active OR bSD1_SET) AND NOT bReset_SD1;
// NOTE: bReset_SD1 only allowed when cause cleared + conditions safe

// SD-1 Actions (FAIL SAFE: de-energize to activate)
// xValve_ProductIsolation := NOT bSD1_Active;  // Energized = OPEN, De-energized = CLOSE
// xBlowdown_Valve := bSD1_Active;              // Energized = OPEN (blowdown when SD1)

// SD-2 (Train A):
// bSD2_SET := bPressure_HH OR bLevel_LL;
// bSD2_Active := (bSD2_Active OR bSD2_SET) AND NOT bReset_SD2 AND NOT bSD1_Active;
// xInletValve_TrainA := NOT (bSD2_Active OR bSD1_Active);  // Open = NOT in shutdown
```

---

## Step 1287: Fire and Gas Detection

### Fire and Gas System

```
FIRE AND GAS DETECTION SYSTEM:
──────────────────────────────────────────────────────────────

F&G Detectors:
├── Flame Detectors (UV/IR): ตรวจเปลวไฟ
├── Gas Detectors:
│   ├── Point Gas (Catalytic Bead): CH4, LPG (0-100% LEL)
│   ├── Ultrasonic Gas (GPD): ตรวจ gas leak by sound
│   └── H2S Detector: ตรวจ Hydrogen Sulfide (toxic)
├── Smoke Detectors (photoelectric/ionization): HVAC rooms
└── Heat Detectors: Linear heat detection cable

ZONE CONFIRMATION (2-out-of-x voting):
ป้องกัน false alarm (false trip = production loss)
ใช้ voting: ต้องมี ≥ 2 detectors ใน zone trigger

Gas Detection Levels:
LEL (Lower Explosive Limit):
├── 20% LEL: Warning alarm → investigate, ventilate
├── 40% LEL: Alarm 2 → automatic ventilation maximum
└── 100% LEL: Shutdown (SD-1 if confirmed)

H2S Levels:
├── 5 ppm: Warning → check equipment, consider evacuation
├── 10 ppm: Alarm → evacuation of non-essential personnel
└── 50 ppm: Emergency → immediate evacuation

// F&G LOGIC (Zone A)
FUNCTION_BLOCK FB_FireGas_Zone
VAR_INPUT
    // Gas detectors (0-100% LEL)
    rGas_D1:    REAL;   // Gas detector D-101
    rGas_D2:    REAL;   // Gas detector D-102
    rGas_D3:    REAL;   // Gas detector D-103
    // Flame detectors
    bFlame_F1:  BOOL;   // Flame detector F-101
    bFlame_F2:  BOOL;   // Flame detector F-102
END_VAR
VAR_OUTPUT
    bGasAlarm_L1:   BOOL;   // 20% LEL (any single)
    bGasAlarm_L2:   BOOL;   // 40% LEL (any single)
    bGasTrip:       BOOL;   // 100% LEL (2oo3 voting)
    bFireDetected:  BOOL;   // Flame confirmed (2oo2)
    bZone_ESD:      BOOL;   // Zone emergency shutdown
END_VAR

// Gas Voting Logic:
// bGasAlarm_L1 := (rGas_D1 >= 20.0) OR (rGas_D2 >= 20.0) OR (rGas_D3 >= 20.0);
// bGasAlarm_L2 := (rGas_D1 >= 40.0) OR (rGas_D2 >= 40.0) OR (rGas_D3 >= 40.0);
// 
// Gas Trip (2oo3: 2 of 3 detectors ≥ 100% LEL):
// bGasTrip := ((rGas_D1 >= 100.0) AND (rGas_D2 >= 100.0)) OR
//             ((rGas_D1 >= 100.0) AND (rGas_D3 >= 100.0)) OR
//             ((rGas_D2 >= 100.0) AND (rGas_D3 >= 100.0));

// Fire (2oo2: both flame detectors must confirm):
// bFireDetected := bFlame_F1 AND bFlame_F2;

// Zone ESD: fire or gas trip
// bZone_ESD := bGasTrip OR bFireDetected;
```

---

## Step 1288: Blowdown System

### ระบบ Blowdown

```
BLOWDOWN SYSTEM:
──────────────────────────────────────────────────────────────

Blowdown = ระบาย pressure ออกจาก vessels ไปยัง Flare อย่างปลอดภัย
เมื่อ: ESD triggered, emergency, planned shutdown

ทำไม Blowdown จำเป็น:
- ป้องกัน BLEVE (Boiling Liquid Expanding Vapor Explosion)
- ลด pressure ก่อน maintenance
- ป้องกัน overpressure ขณะ fire

Blowdown Sequence:
1. ESD-1 triggered
2. Close all feed/product isolation valves
3. Open blowdown valves (ค่อยๆ เพื่อไม่ให้ flare overload)
4. Flare gas ไหลไป knockout drum → flare tip
5. Monitor vessel pressure → confirm depressurization

// BLOWDOWN CONTROL
FUNCTION_BLOCK FB_Blowdown
VAR_INPUT
    bBlowndown_Cmd: BOOL;           // Initiate blowdown
    rVesselPress:   REAL;           // Vessel pressure
    rFlare_Capacity:REAL;           // Flare max capacity (kg/h)
    rActualFlowRate:REAL;           // Current flare flow
END_VAR
VAR_OUTPUT
    xBlowdown_Valve:BOOL;           // Blowdown valve command
    bBlowdownComplete: BOOL;        // Pressure < 3 bar (safe)
    bFlareOverload:    BOOL;        // Flare at capacity
END_VAR

// Staged blowdown to prevent flare overload:
// Stage 1: Open 20% → wait 5 min
// Stage 2: Open 50% → wait 5 min  
// Stage 3: Open 100% until pressure < 3 bar

// Monitor flare flow:
// IF rActualFlowRate > rFlare_Capacity × 0.9 THEN
//     bFlareOverload := TRUE;
//     // Don't open more blowdown valves
//     // Alert operator: "Flare approaching capacity"
// END_IF;

// Complete:
// bBlowdownComplete := rVesselPress < 3.0;  // < 3 bar = depressurized
```

---

## Step 1289: Process Shutdown Logic

### Interlock and Shutdown Logic

```
PROCESS SHUTDOWN LOGIC (BPCS ส่วน):
──────────────────────────────────────────────────────────────

BPCS ควบคุม normal process สามารถ:
- ปิด equipment เมื่อ process เกิน operating limit
- ส่ง demand ไป SIS (SIS เป็น final shutdown)

// Example: Gas Compressor Shutdown (BPCS level)
FUNCTION_BLOCK FB_Compressor_Protection
VAR_INPUT
    rSuction_Press:     REAL;   // Suction pressure (bar)
    rDischarge_Press:   REAL;   // Discharge pressure (bar)
    rBearing_Temp:      REAL;   // Bearing temperature (°C)
    rVibration:         REAL;   // Vibration (mm/s)
    rLube_Press:        REAL;   // Lube oil pressure (bar)
    bOilLevel_Low:      BOOL;   // Oil level low switch
END_VAR
VAR_OUTPUT
    bTrip:          BOOL;       // Compressor trip
    bAlarm_1:       BOOL;       // Warning alarm
    bAlarm_2:       BOOL;       // High alarm (take action)
    sTrip_Reason:   STRING[60]; // Why trip?
END_VAR

// Trip conditions (any one triggers trip):
// 1. Low suction pressure (< 0.5 bar → cavitation)
// 2. High discharge pressure (> 100 bar → over-pressure)
// 3. High bearing temperature (> 85°C)
// 4. High vibration (> 15 mm/s → seal/bearing damage)
// 5. Low lube oil pressure (< 1 bar → lubrication failure)

// IF rSuction_Press < 0.5 THEN
//     bTrip := TRUE;
//     sTrip_Reason := 'LOW SUCTION PRESSURE';
// ELSIF rDischarge_Press > 100.0 THEN
//     bTrip := TRUE;
//     sTrip_Reason := 'HIGH DISCHARGE PRESSURE';
// ELSIF rBearing_Temp > 85.0 THEN
//     bTrip := TRUE;
//     sTrip_Reason := 'HIGH BEARING TEMPERATURE';
// ...

// Trip: ล็อกอยู่จนกว่า reset (SR latch)
// Alarm 1: กำหนด setpoints ต่ำกว่า trip
// Alarm 2: กำหนด setpoints ระหว่าง Alarm1 และ trip
```

---

## Step 1290: Wellhead Control (Remote)

### ควบคุม Wellhead จากระยะไกล

```
REMOTE WELLHEAD CONTROL:
──────────────────────────────────────────────────────────────

Wellhead = หัวหลุมน้ำมัน (อาจอยู่ห่างออกไป 5-100 km)
PLC ที่ wellhead (Remote Terminal Unit - RTU)
เชื่อมต่อกับ Control Room ผ่าน:
├── Fiber optic (onshore, reliable)
├── Satellite (remote areas)
├── Cellular 4G/5G (ถ้าครอบคลุม)
└── Radio (short range)

WELLHEAD SAFETY DEVICES:
├── SSSV (Surface Safety Valve): ปิดเมื่อ emergency
├── DHSV (Downhole Safety Valve): ปิดใต้ดิน (ultimate protection)
├── Wing Valve: isolate production side
└── Choke Valve: ควบคุม flow rate

// RTU WELLHEAD CONTROLLER
FUNCTION_BLOCK FB_WellheadRTU
VAR_INPUT
    rWellheadPress:     REAL;   // Wellhead pressure (bar)
    rFlowline_Press:    REAL;   // Flowline pressure
    rFlowRate_Sm3:      REAL;   // Oil flow rate
    bRemote_ESD:        BOOL;   // ESD from control room
END_VAR

// WATCHDOG FOR COMMS LOSS:
// ถ้า communication กับ control room ขาด > 30 sec → fail safe
// Fail safe for wellhead: 
//   Option 1: Stay open (if flow is needed for production)
//   Option 2: Close (if safety conservative)
// ขึ้นอยู่กับ risk assessment!

// High pressure protection (local, no comms needed):
// IF rWellheadPress > rHH_Press_Limit THEN
//     xSSSV_Close := TRUE;  // Close surface safety valve
// END_IF;
```

---

## Step 1291: HIPPS (High Integrity Pressure Protection)

### High Integrity Pressure Protection System

```
HIPPS:
──────────────────────────────────────────────────────────────

HIPPS = High Integrity Pressure Protection System
ป้องกัน pressure vessel จาก overpressure
แทน PSV (Pressure Safety Valve) ในบาง applications

เมื่อไหร่ใช้ HIPPS แทน PSV:
├── ก๊าซ/ของเหลวที่เป็นพิษ (PSV ปล่อยออกสู่ atmosphere)
├── ติดตั้ง PSV ยาก (offshore, deep water)
└── Process conditions ที่ PSV ทำงานไม่ได้ดี

SIL Requirement: HIPPS ต้องเป็น SIL 3 เสมอ (IEC 61511)
เพราะ PSV removed → HIPPS คือ last line of defense

// HIPPS DESIGN REQUIREMENTS:
// 1. Response time: < 2 seconds (fast!)
// 2. 2oo3 pressure measurement
// 3. 2 parallel final elements (valves) 
// 4. Proof test interval: 1-3 years
// 5. Dedicated SIS logic solver

// HIPPS PRESSURE VOTING + ACTION:
// IF 2 of 3 pressures > HIPPS setpoint:
//     Close main isolation valve (V-001) → fast close (< 1 sec)
//     Close backup valve (V-002) simultaneously
//     Activate alarm (HIPPS TRIGGERED)
//     Log to SIS historian
```

---

## Step 1292: Flare System Management

### การจัดการ Flare System

```
FLARE SYSTEM MANAGEMENT:
──────────────────────────────────────────────────────────────

Flare System เผาทำลาย waste gas ที่ปล่อยออกมาจาก:
- PSVs (Pressure Safety Valves)
- Blowdown
- Emergency vents
- Purge gas

Components:
├── Flare Header (รวม gas จากทุกแหล่ง)
├── Knockout Drum (ดัก liquid ออก)
├── Seal Drum (ป้องกัน flashback)
├── Flare Stack (ท่อสูง 50-100m)
└── Flare Tip (จุดเผา)

PLC Controls:
├── Pilot Flame Monitoring: ตรวจว่า pilot เปิดอยู่
├── Knockout Drum Level: ระบายน้ำออก
├── Smokeless Flare: Steam injection เพื่อลดควัน
└── Flare Gas Recovery: recover ก๊าซที่ flow ต่ำ

// PILOT FLAME MONITOR
FUNCTION_BLOCK FB_FlarePilot
VAR_INPUT
    bUVDetector:    BOOL;       // UV flame detector
    bThermCouple:   BOOL;       // Thermocouple (temperature)
    bPilotGas:      BOOL;       // Pilot gas supply
END_VAR
VAR_OUTPUT
    bPilotOK:       BOOL;       // Pilot burning
    bPilotAlarm:    BOOL;       // Pilot failure
    xIgniteCmd:     BOOL;       // Ignition command
    iRetrieCount:   INT;        // Number of ignition attempts
END_VAR

// ถ้า pilot ดับ → ignite ใหม่ (auto-ignition)
// IF NOT bUVDetector AND bPilotGas THEN
//     xIgniteCmd := TRUE;  // Send ignition spark
//     iRetrieCount := iRetrieCount + 1;
//     IF iRetrieCount > 3 THEN
//         bPilotAlarm := TRUE;  // Alert: pilot won't light
//         // Investigate: gas supply? Igniter fault?
//     END_IF;
// ELSE
//     bPilotOK := bUVDetector;
//     iRetrieCount := 0;
// END_IF;
```

---

## Step 1293: Pipeline Leak Detection

### ตรวจจับ Pipeline Leak

```
PIPELINE LEAK DETECTION:
──────────────────────────────────────────────────────────────

Methods:
1. SCADA Based (Mass Balance):
   Flow in - Flow out = 0 (ถ้า no leak)
   ถ้า Flow in > Flow out → potential leak
   Accuracy: 0.5-2% of throughput

2. Pressure Point Analysis (PPA):
   Monitor pressure patterns
   Transients indicate leak location

3. Acoustic Emission:
   Leak creates characteristic sound
   Fiber optic distributed acoustic sensing (DAS)

// MASS BALANCE LEAK DETECTION
FUNCTION_BLOCK FB_LeakDetection
VAR_INPUT
    rFlow_In:       REAL;       // Flow at pipeline inlet (m³/h)
    rFlow_Out:      REAL;       // Flow at pipeline outlet
    rLine_Volume:   REAL;       // Pipeline volume (m³) for pack/unpack
    rPressure_In:   REAL;       // Inlet pressure
    rPressure_Out:  REAL;       // Outlet pressure
END_VAR
VAR_OUTPUT
    rBalance_m3h:   REAL;       // Mass balance (flow in - flow out)
    bLeakAlarm:     BOOL;       // Potential leak detected
    rLeakRate_m3h:  REAL;       // Estimated leak rate
END_VAR

// REAL-TIME TRANSIENT MODEL (simplified):
// Correct for line packing/unpacking (pressure change → volume change)
// rLinepackCorrection := rLine_Volume × (rPressure_In - rPressure_Out) / (rBasePress × rCompressibility);
// 
// rBalance_m3h := rFlow_In - rFlow_Out - rLinepackCorrection;
// 
// If balance > threshold for > 5 minutes → leak alarm
// bLeakAlarm := rBalance_m3h > rLeak_Threshold AND tAlarmTimer.Q;
// rLeakRate_m3h := rBalance_m3h;  // Estimated leak rate
```

---

## Step 1294: SCADA for Oil and Gas

### SCADA System ใน Oil and Gas

```
OIL AND GAS SCADA ARCHITECTURE:
──────────────────────────────────────────────────────────────

Honeywell Experion PKS (Process Knowledge System):
ใช้ใน: Refineries, LNG Plants, Petrochemical

ABB System 800xA:
ใช้ใน: Offshore platforms, pipelines

Yokogawa CENTUM VP:
ใช้ใน: Refineries, chemical plants

Siemens SIMATIC PCS 7 / PCS neo:
ใช้ใน: Process industry, chemical

OFFSHORE PLATFORM ARCHITECTURE:
Control Room → Operator Workstations → DCS Server
  ↓ (via VLAN)
Safety Network → ESD Controller (Triconex)
  ↓ (via PROFIsafe)
Field Instruments (SIL-rated)
  + Manual ESD panels (hardwired, not via network)

// DCS TAG NAMING (ISA 5.1):
// Format: UNIT-INSTRUMENT_ID
// Example: 
// 01-FT-001  = Unit 01, Flow Transmitter 001
// 01-PT-005A = Unit 01, Pressure Transmitter 005, train A
// 01-PV-005  = Unit 01, Pressure Control Valve 005

// DCS Faceplates (operator views):
// Normal: tag, current value, setpoint, mode (AUTO/MAN/CAS)
// Alarm: red highlight, acknowledge button
// Trend: 1h/24h/7d trending
```

---

## Step 1295: Custody Transfer Metering

### วัดปริมาณสำหรับการส่งมอบ

```
CUSTODY TRANSFER METERING:
──────────────────────────────────────────────────────────────

Custody Transfer = การวัดเพื่อซื้อขาย (billing purposes)
ต้องการ:
├── Certified meters (PTB, OIML, API MPMS)
├── Audit trail สำหรับทุก measurement
├── Tamper detection
└── Regular calibration

Meter Types:
├── Turbine meter (gas, high accuracy ±0.25%)
├── Ultrasonic meter (gas/liquid, ±0.25%)
├── Coriolis (mass flow, ±0.1%)
└── Orifice plate (lower accuracy, common in gas)

// CUSTODY TRANSFER METER FB
FUNCTION_BLOCK FB_CustodyMeter
VAR_INPUT
    rRawFlow_m3h:   REAL;       // Raw volume flow
    rPressure:      REAL;       // Line pressure
    rTemperature:   REAL;       // Fluid temperature
    rCompressibility: REAL;     // Z factor
END_VAR
VAR_OUTPUT
    rFlowRate_Sm3h: REAL;       // Standard conditions flow
    rTotalizer_Sm3: LREAL;      // Accumulated volume (Sm³)
    bMeterAlarm:    BOOL;       // Meter out of range
END_VAR
VAR
    rFlow_Corrected:REAL;
    tScanTimer:     TON;
END_VAR

// VOLUME CORRECTION (AGA-8 / ISO 6976):
// Convert actual conditions → standard conditions (15°C, 1.01325 bar)
// rFlow_Corrected := rRawFlow_m3h × (rPressure / rStdPress) 
//                  × (rStdTemp / rTemperature) 
//                  × (1.0 / rCompressibility);
// rFlowRate_Sm3h := rFlow_Corrected;

// Totalizer (accumulate every scan):
// rTotalizer_Sm3 := rTotalizer_Sm3 + rFlowRate_Sm3h × rScanTime_hr;
// rScanTime_hr = 0.1s / 3600 = 2.78×10⁻⁵ hours

// RETAIN: Totalizer ต้อง RETAIN (survive power cycle)
// Audit: ทุก totalizer reset ต้องมี audit trail
```

---

## Step 1296: H2S Safety System

### Hydrogen Sulfide Safety

```
H2S SAFETY SYSTEM:
──────────────────────────────────────────────────────────────

H2S เป็นพิษสูงมาก (IDLH = 100 ppm, สูดดม 1000 ppm → ตายทันที)
พบใน: sour gas fields, refineries, geothermal

H2S Alarm Levels (OSHA):
├── 1 ppm: Odor detection (rotten egg smell)
├── 10 ppm: PEL (8-hour exposure limit)
├── 20 ppm: Ceiling (short-term)
├── 50 ppm: Immediately Dangerous → evacuate
└── 100 ppm: IDLH (Immediately Dangerous to Life/Health)

PLC Control Actions:
├── 10 ppm: Warning alarm → check area
├── 20 ppm: Alarm → ventilation max, personnel clear area
├── 50 ppm: Emergency → evacuation PA announcement, non-essential shutdown
└── 100 ppm: Emergency shutdown → SD-1

// H2S RESPONSE LOGIC
FUNCTION_BLOCK FB_H2S_Response
VAR_INPUT
    rH2S_D1:    REAL;   // H2S detector 1 (ppm)
    rH2S_D2:    REAL;   // H2S detector 2 (ppm)
END_VAR
VAR_OUTPUT
    bAlarm_10ppm:   BOOL;   // Warning
    bAlarm_20ppm:   BOOL;   // Alert
    bAlarm_50ppm:   BOOL;   // Emergency
    bTrip_100ppm:   BOOL;   // Shutdown demand
    xMaxVentilation:BOOL;   // Max ventilation
    xPA_Announce:   BOOL;   // PA system announcement
END_VAR

// 1oo2 voting for trip (either detector → trip):
// bTrip_100ppm := (rH2S_D1 >= 100.0) OR (rH2S_D2 >= 100.0);

// bAlarm_50ppm := (rH2S_D1 >= 50.0) OR (rH2S_D2 >= 50.0);
// IF bAlarm_50ppm THEN
//     xPA_Announce := TRUE;  // "Attention: H2S alarm, evacuate area"
//     xMaxVentilation := TRUE;
// END_IF;
```

---

## Step 1297: Offshore Platform Control Philosophy

### Control Philosophy สำหรับ Offshore

```
OFFSHORE PLATFORM CONTROL PHILOSOPHY:
──────────────────────────────────────────────────────────────

Operating Modes:
1. NORMAL: Full automatic operation, personnel on board
2. MINIMUM MANNED: Reduced staff, higher automation
3. UNMANNED: Fully automated, remote monitoring
4. SHUTDOWN: Emergency or planned

Mode Changes:
- Controlled by platform manager
- PLC adjusts alarm setpoints, response strategies
- SD thresholds may change (stricter in unmanned mode)

// OPERATING MODE MANAGEMENT
DATA_BLOCK DB_PlatformMode
    iOperatingMode:     INT;    // 1=Normal, 2=MinManned, 3=Unmanned, 4=SD
    sMode_Name:         STRING;
    bPersonnel_OnBoard: BOOL;
    iPersonnelCount:    INT;    // Muster count
END_DATA_BLOCK

// Autonomous response in UNMANNED mode:
// All alarms: auto-diagnose and attempt corrective action
// Persistent fault → remote notification → on-call engineer response
// Severe fault → automatic safety shutdown (don't wait for human)

PRODUCTION OPTIMIZATION SYSTEM:
├── Maximize oil/gas production
├── Stay within operating envelope
├── Predict problems before they occur (predictive maintenance)
├── Dynamic optimization based on reservoir conditions
└── Report to corporate SCADA (real-time production data)

// Remote Operations Center (ROC):
// Multiple platforms monitored from single control room
// ≥ 500 km away
// 24/7 monitoring by engineers
// Can command equipment remotely
// Intervention via ESD/BPCS override
```

---

## Step 1298: Process Safety Documentation

### Safety Documentation Requirements

```
PROCESS SAFETY DOCUMENTATION:
──────────────────────────────────────────────────────────────

REQUIRED DOCUMENTATION (IEC 61511):
1. HAZOP Report:
   - Process description
   - Deviations analyzed
   - Risk assessment
   - Recommended safeguards

2. SIS Design Documentation:
   - SIS architecture diagram
   - SIF list (all safety functions)
   - Cause and Effect Matrix
   - SIL verification calculations

3. Functional Specification (FS):
   - Detail of each SIF logic
   - Response times
   - Proof test procedures

4. Hardware Documentation:
   - Instrument datasheets (all SIS instruments)
   - Loop diagrams (each SIF)
   - Logic solver configuration

5. FAT / SAT Documentation:
   - Test cases for each SIF
   - Witness signatures

6. Proof Test Procedures (PTPs):
   - Step-by-step test procedure
   - Frequency (annual, every 3 years, etc.)
   - Required test equipment
   - Acceptance criteria

// PROOF TEST RECORD FORMAT:
// SIF ID:           SIF-001
// Description:      High Pressure ESD
// Test Date:        2024-03-15
// Test Frequency:   Annual (per SIL verification)
// Tested By:        J. Smith (Certified CFSE)
// Witnessed By:     Platform Manager
// 
// Test Step 1: Isolate process
// Result: Done
// Test Step 2: Simulate HH pressure (inject signal to transmitter)
// Result: All 3 transmitters respond correctly
// Test Step 3: Verify 2oo3 logic → check output
// Result: Feed valve closes within 1.2 seconds (spec: < 2s)
// ...
// RESULT: PASS
```

---

## Step 1299: Emergency Response Integration

### ระบบ Emergency Response

```
EMERGENCY RESPONSE INTEGRATION:
──────────────────────────────────────────────────────────────

Oil and Gas Emergency Response involves multiple systems:
├── PLC/DCS: Process control + alarms
├── F&G System: Detection
├── ESD System: Shutdown
├── PA/GA System: Public Address/General Alarm
├── HVAC: Isolate ventilation + pressurize escape routes
├── Deluge/Fire Suppression: Activate water/foam
└── Muster System: Account for personnel

PLC INTEGRATION:
When F&G confirms fire:
1. ESD triggers (SD-1)
2. PA system announces: "Fire in Zone A - Muster at Assembly Point B"
3. HVAC: Close dampers (prevent smoke spread), pressurize escape routes
4. Deluge: Activate Zone A water deluge
5. Notify Coast Guard (automatic via satellite)
6. Helicopter landing lights: Enable
7. Lifeboat stations: Alert signal

// INTEGRATED EMERGENCY RESPONSE (PLC Output)
// IF bSD1_Active THEN
//     // PA/GA System (via digital output)
//     xPA_Fire_Alarm := TRUE;      // Continuous alarm tone
//     xGA_Announcement := TRUE;   // Trigger pre-recorded announcement
//     
//     // HVAC
//     xHVAC_Isolate := TRUE;       // Close all ventilation
//     xEscape_Route_Pressurise := TRUE;
//     
//     // Deluge (zone specific based on F&G location)
//     xDeluge_ZoneA := bFire_ZoneA;
//     xDeluge_ZoneB := bFire_ZoneB;
//     
//     // Navigation aids
//     xHelicopter_Lights := TRUE;
//     
//     // Muster System
//     xMuster_Alert := TRUE;
// END_IF;
```

---

## Step 1300: Lab Project - Oil and Gas Safety System

### Lab: SIS Design for Offshore Separator

```
LAB: SAFETY INSTRUMENTED SYSTEM FOR OFFSHORE SEPARATOR
══════════════════════════════════════════════════════════════

Process: Three-Phase Separator (Oil/Water/Gas)
Hazards: High pressure gas, H2S content (sour service)
Required SIL: SIL 2 (determined by HAZOP)

GIVEN PROCESS DATA:
Design Pressure: 100 bar
Operating Pressure: 60-70 bar
Operating Temperature: 50-80°C
Fluid: Crude oil + associated gas (H2S: 500 ppm)

SIFs to Design:
SIF-001: High High Pressure → Close inlet valve + ESD
SIF-002: Low Low Level (oil) → Stop booster pump
SIF-003: High High Temperature → Trip heater
SIF-004: Gas Detector → ESD (Zone: Separator area)
SIF-005: H2S Detection → Alarm + Ventilation

TASK 1: Cause and Effect Matrix (20%)
─────────────────────────────────────────────────────────────
สร้าง C&E Matrix:
- Causes: 8 process deviations + 3 equipment faults + 1 manual ESD
- Effects: 10 output actions
- Include logic (1oo1, 2oo2, 2oo3 as appropriate)
- Define ESD levels (SD-1, SD-2, SD-3)

TASK 2: SIL Verification (20%)
─────────────────────────────────────────────────────────────
สำหรับ SIF-001 (HH Pressure):
1. Select sensor architecture (1oo2, 2oo3)
2. Select final element (specify valve type)
3. Calculate PFD using given failure rates:
   - Pressure transmitter λ = 2×10⁻⁶ /hr
   - Safety valve λ = 3×10⁻⁶ /hr
   - Safety PLC λ = 1×10⁻⁷ /hr
4. Verify SIL 2 (PFD < 0.01) with annual proof test
5. If not SIL 2: propose improvement

TASK 3: Safety PLC Programming (35%)
─────────────────────────────────────────────────────────────
Implement in TIA Portal (F-CPU):

1. SIF-001: HH Pressure ESD
   - 2oo3 pressure voting
   - Close inlet valve (F-digital output)
   - Response time: < 2 seconds
   
2. SIF-004: Gas Detection (Zone A)
   - 2oo2 confirmation
   - SD-1 trigger
   - F&G response sequence
   
3. SIF-005: H2S Detection
   - 1oo2 voting
   - 3 alarm levels (10/20/50 ppm)
   - 100 ppm → SD-1 demand
   
4. ESD Latch:
   - SR latch for all trips
   - Reset only when: cause cleared + operator authorization
   - Audit trail for all resets

TASK 4: Proof Test Procedure (25%)
─────────────────────────────────────────────────────────────
เขียน Proof Test Procedure สำหรับ SIF-001:
1. Pre-test checklist
2. Step-by-step test
3. Expected results for each step
4. Pass/fail criteria
5. Document format (ready to use)

DELIVERABLES:
├── Cause and Effect Matrix (Excel)
├── SIL Verification calculation (Excel/Word)
├── TIA Portal project with F-CPU safety logic
├── Proof Test Procedure document
└── Brief report: Describe safety architecture decisions

EVALUATION:
├── C&E Matrix: 20%
├── SIL Verification: 20%
├── SIS Program: 35%
├── Proof Test Document: 15%
└── Report quality: 10%

BONUS:
- Implement F0 (Blowdown sequence after SD-1)
- Add custody transfer metering
- Create SCADA mimics for operator view
```

---

## สรุป Part 65

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| IEC 61511 | SIS design framework, SIL verification |
| BPCS vs SIS | ต้อง separate! ไม่ share sensors/logic |
| C&E Matrix | Cause → Effect relationships |
| ESD System | Latched, fail-safe, hierarchical |
| F&G System | 2oo2 confirmation, voting logic |
| SIL Verification | PFD calculation, architecture selection |

---

## Navigation

- [← Part 64: Food and Pharmaceutical Automation](./part64_food_pharma.md)
- [→ Part 66: Advanced SCADA and Historian Systems](./part66_scada_historian.md)
- [↑ สารบัญหลัก](../README.md)

---

## สรุปภาพรวม Parts 51-65

Parts 51-65 ครอบคลุมหัวข้อระดับ Expert:
- **Security**: ICS/OT Cybersecurity (IEC 62443)
- **Networking**: Industrial networks, PROFINET, 5G
- **Control**: Advanced PID, Cascade, MPC, Smith Predictor
- **IIoT**: Cloud integration, MQTT, OPC UA
- **Digital Twin**: Virtual commissioning
- **Diagnostics**: PROFINET advanced diagnostics
- **Energy**: ISO 50001, VFD optimization
- **Design Patterns**: State machine, Observer, Strategy
- **Version Control**: Git for PLC programs
- **Testing**: FAT/SAT, GAMP 5, 21 CFR Part 11
- **Large Systems**: CPU load, redundancy, I/O philosophy
- **Motion**: Gantry, Delta robot, PLCopen
- **Welding**: MIG, Spot, Safety, Quality monitoring
- **Food/Pharma**: GMP, CIP, Batch records, HACCP
- **Oil/Gas**: IEC 61511, SIS, ESD, F&G

ทั้งหมดนี้เป็นความรู้ระดับ World-Class Expert
ที่ใช้ได้ใน projects จริงในอุตสาหกรรมระดับสากล
