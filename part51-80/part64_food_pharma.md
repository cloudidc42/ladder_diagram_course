# Part 64: Food and Pharmaceutical Automation (Steps 1261-1280)

## บทนำ: Food and Pharma ต่างจากอุตสาหกรรมทั่วไป

ความต่างหลัก:
- **GMP (Good Manufacturing Practice)**: ทุก action ต้องมี document
- **Data Integrity**: บันทึกต้องไม่แก้ไขได้ (tamper-proof)
- **Validation**: ต้อง validate ระบบก่อน use ใน production
- **Traceability**: trace ทุก lot ได้ถึง raw material
- **Hygiene**: design เพื่อ clean ง่าย, ไม่มี harbor point

---

## Step 1261: GMP Requirements

### Good Manufacturing Practice

```
GMP OVERVIEW:
──────────────────────────────────────────────────────────────

GMP = Good Manufacturing Practice
กฎที่ FDA, EMA, WHO กำหนดสำหรับ pharmaceutical manufacturing

Key Areas:
1. DOCUMENTATION: ถ้าไม่ได้บันทึก = ไม่ได้ทำ
2. PERSONNEL: training requirements, health monitoring
3. PREMISES: clean rooms, zoning
4. EQUIPMENT: qualified, calibrated, maintained
5. PRODUCTION: validated processes, SOPs
6. QUALITY CONTROL: testing, release procedures
7. RECORDS: data integrity, audit trail

DATA INTEGRITY (ALCOA+ Principles):
A - Attributable: รู้ว่าใครทำ
L - Legible: อ่านได้ตลอด lifecycle
C - Contemporaneous: บันทึกทันทีเมื่อทำ
O - Original: บันทึก original ไม่ใช่ copy
A - Accurate: ถูกต้อง

+ Principles:
C - Complete: บันทึกครบถ้วน
C - Consistent: สอดคล้องกัน
E - Enduring: เก็บตลอดอายุที่กำหนด
A - Available: เรียกดูได้เมื่อต้องการ

// PLC ต้อง support ALCOA+:
// ├── Timestamp: NTP synchronized, tamper-proof
// ├── User log: ใครทำ what when
// ├── Audit trail: ทุก parameter change recorded
// └── Data storage: cannot be modified/deleted
```

---

## Step 1262: Hygienic Design Principles

### Design for Cleanability

```
HYGIENIC DESIGN (EHEDG / 3-A Standards):
──────────────────────────────────────────────────────────────

Principles:
1. NO DEAD ENDS (ไม่มี crevice ที่ product ค้างได้):
   - Pipe tees: 3-A flow-through
   - Valves: zero dead leg design
   - Bends: radius ≥ 3× pipe diameter

2. SURFACE FINISH:
   Food grade: Ra ≤ 3.2 μm (125 μin)
   Pharmaceutical: Ra ≤ 0.8 μm (32 μin) (product contact)

3. SLOPE FOR DRAINAGE:
   All surfaces slope ≥ 1/8 inch per foot (1%)
   ไม่มีน้ำขัง

4. MATERIAL:
   - 316L SS (product contact)
   - FDA-approved elastomers (PTFE, EPDM)
   - Glass/ceramic (alternatives)

5. CONNECTIONS:
   - Tri-clamp/SMS/DIN connections (cleanable)
   - ห้าม threaded connections บน product surfaces

6. INSTRUMENT CONNECTIONS:
   - Flush diaphragm pressure transmitters
   - ไม่มี pockets (ที่ product ค้าง)

// PLC-RELATED HYGIENIC CONCERNS:
// ├── Sensors: flush-mount, IP69K (high-pressure washdown)
// ├── Cables: IP69K connectors
// ├── Panel: IP65/66, stainless enclosure option
// └── Pushbuttons: waterproof, no crevices
```

---

## Step 1263: CIP (Clean In Place) System

### ระบบ Clean In Place

```
CIP SYSTEM OVERVIEW:
──────────────────────────────────────────────────────────────

CIP = Clean In Place
ทำความสะอาดโดยไม่ต้องถอดชิ้นส่วน
ส่งสารเคมีไหลผ่านระบบ

Typical CIP Sequence (Dairy):
1. PRE-RINSE:      น้ำเย็น/อุ่น ล้างตะกอน (5 min)
2. CAUSTIC WASH:   NaOH 2% at 80°C (20-30 min) - ล้างโปรตีน
3. INTERMEDIATE RINSE: น้ำ ล้าง caustic (5 min)
4. ACID WASH:      HNO3 1% at 70°C (10-15 min) - ล้าง mineral/scale
5. FINAL RINSE:    น้ำ RO quality (5-10 min)
6. DRAIN:          ระบาย + blow dry

PLC Controls for CIP:
├── xCaustic_Valve: เปิดวาล์ว caustic supply
├── xCIP_Pump: CIP return pump
├── rCaustic_Concentration: dosing control
├── rCIP_Temperature: heat exchanger control
└── rCIP_FlowRate: ≥ minimum flow for effective cleaning

// CIP STATE MACHINE
FUNCTION_BLOCK FB_CIP_Controller
VAR
    iPhase:             INT := 0;
    tPhaseTimer:        TON;
    rCurrentTemp:       REAL;       // Actual CIP fluid temp
    rCurrentFlow:       REAL;       // Actual flow rate
    rConductivity:      REAL;       // For clean rinse verification
    bPhaseComplete:     BOOL;
    sCIPPhase:          STRING;
END_VAR

CONST
    CIP_PRERINSE    := 1;
    CIP_CAUSTIC     := 2;
    CIP_INT_RINSE   := 3;
    CIP_ACID        := 4;
    CIP_FINAL_RINSE := 5;
    CIP_DRAIN       := 6;
END_CONST

CASE iPhase OF
    0:  // IDLE
        IF bStartCIP THEN
            iPhase := CIP_PRERINSE;
            sCIPPhase := 'PRE-RINSE';
        END_IF;
    
    CIP_PRERINSE:  // Phase 1: Pre-rinse
        xColdWaterValve := TRUE;
        xCIPPump := TRUE;
        sCIPPhase := 'PRE-RINSE';
        tPhaseTimer(IN := TRUE, PT := T#5m);  // 5 minutes
        IF tPhaseTimer.Q THEN
            tPhaseTimer(IN := FALSE);
            xColdWaterValve := FALSE;
            iPhase := CIP_CAUSTIC;
        END_IF;
    
    CIP_CAUSTIC:   // Phase 2: Caustic wash
        xCausticValve := TRUE;
        xHeater := TRUE;
        xCIPPump := TRUE;
        sCIPPhase := 'CAUSTIC WASH';
        
        // Temperature control (80°C)
        IF rCurrentTemp >= 78.0 THEN  // At temperature
            tPhaseTimer(IN := TRUE, PT := T#20m);  // 20 min hold
        END_IF;
        
        IF tPhaseTimer.Q THEN
            tPhaseTimer(IN := FALSE);
            xCausticValve := FALSE;
            xHeater := FALSE;
            iPhase := CIP_INT_RINSE;
        END_IF;
        
        // Safety: max temperature 90°C
        IF rCurrentTemp > 90.0 THEN
            xHeater := FALSE;
        END_IF;
    
    CIP_FINAL_RINSE:
        xRinseWater := TRUE;
        xCIPPump := TRUE;
        sCIPPhase := 'FINAL RINSE';
        
        // Complete when conductivity < 10 μS/cm (clean water)
        IF rConductivity < 10.0 THEN
            tPhaseTimer(IN := TRUE, PT := T#2m);  // Hold 2 min after conductivity OK
            IF tPhaseTimer.Q THEN
                xRinseWater := FALSE;
                xCIPPump := FALSE;
                iPhase := CIP_DRAIN;
            END_IF;
        END_IF;

    // ...etc for all phases
END_CASE;
```

---

## Step 1264: SIP (Steam In Place) System

### Sterilization In Place

```
SIP (STEAM IN PLACE):
──────────────────────────────────────────────────────────────

SIP = Sterilization In Place
ฆ่าเชื้อโดยใช้ไอน้ำ ไม่ต้องถอดชิ้นส่วน
ใช้หลัง CIP สำหรับ sterile manufacturing

SIP Requirements (FDA/EMA):
├── Steam at 121°C minimum (saturated steam)
├── Hold time: 15-30 minutes at 121°C
├── All points must reach temperature (hot spot validation)
└── Lethality: F0 value ≥ 15 (equivalent to 15 min at 121°C)

F0 VALUE CALCULATION:
F0 = integral of L × dt
where L = 10^((T - 121)/10)

// ตัวอย่าง: 30 minutes at 121°C
// F0 = 30 × 10^((121-121)/10) = 30 × 1 = 30 → more than enough

// FB_SIP (Steam Sterilization)
FUNCTION_BLOCK FB_SIP_Controller
VAR_INPUT
    bStartSIP:      BOOL;
    rMinTemp_C:     REAL := 121.0;  // Minimum sterilization temp
    tHoldTime:      TIME := T#30m;  // Hold time
END_VAR
VAR_OUTPUT
    bSIPComplete:   BOOL;           // SIP done (sterile)
    bSIPFault:      BOOL;
    rF0Value:       REAL;           // Accumulated F0
    sCIPPhase:      STRING;
END_VAR
VAR
    iState:         INT := 0;
    tHoldTimer:     TON;
    rL_Value:       REAL;           // Lethality rate
    rScanTime_s:    REAL := 0.1;    // Scan time for integration
END_VAR

// F0 Integration ใน state HOLD:
// rL_Value := EXPT(10.0, (rCurrentTemp - 121.0) / 10.0);
// rF0Value := rF0Value + rL_Value × rScanTime_s / 60.0; // min units
// 
// SIP complete when: F0 >= 15 AND hold time elapsed
// IF rF0Value >= 15.0 AND tHoldTimer.Q THEN
//     bSIPComplete := TRUE;
// END_IF;
```

---

## Step 1265: 21 CFR Part 11 Implementation

### Electronic Records ใน PLC Systems

```
21 CFR PART 11 IMPLEMENTATION:
──────────────────────────────────────────────────────────────

ต้องทำใน PLC System:

1. AUDIT TRAIL (automatic):
   ทุก recipe parameter change:
   - Who changed it (user ID)
   - What was changed (parameter name)
   - Old value
   - New value
   - When (timestamp from NTP)
   - Reason (comment entered by user)

// Audit Trail Entry
DATA_BLOCK DB_AuditTrail
    entries: ARRAY[1..10000] OF STRUCT
        dtTimestamp:    DATE_AND_TIME;
        sUserID:        STRING[20];
        iUserLevel:     INT;        // 1=Operator, 2=Supervisor, etc.
        sAction:        STRING[50]; // "SetParameter", "AcknowledgeAlarm"
        sParameter:     STRING[40]; // "Recipe.Step1.Temp"
        rOldValue:      REAL;
        rNewValue:      REAL;
        sReason:        STRING[80]; // Required for critical changes
        iComputerID:    INT;        // Which workstation
    END_STRUCT;
    iCount:         DINT;
    iCRCChecksum:   DINT;  // Integrity check
END_DATA_BLOCK

// Write audit trail entry (เรียกทุกครั้งที่มีการเปลี่ยน critical parameter):
// DB_AuditTrail.entries[iCount].dtTimestamp := NOW();
// DB_AuditTrail.entries[iCount].sUserID     := sCurrentUser;
// ... etc.
// iCount := iCount + 1;
// Update CRC checksum

2. ELECTRONIC SIGNATURES:
   Critical actions ต้อง dual signature
   (HMI enforces, PLC verifies)
   
// Signature Verification in PLC:
// HMI sends: {UserID, PasswordHash, ActionID}
// PLC verifies: Hash correct? Level sufficient?
// ถ้า OK → allow action
// Log in audit trail: "Signed by: user1, verified by: user2"

3. SECURE TIMESTAMP:
// PLC time must come from NTP server (not local battery)
// NTP server must be auditable (logs showing time sync)
// Alarm if NTP sync fails > 1 minute
```

---

## Step 1266: Allergen Management

### ป้องกัน Cross-Contamination ของ Allergens

```
ALLERGEN MANAGEMENT ใน PLC:
──────────────────────────────────────────────────────────────

อาหารที่มี allergens หลัก (EU Regulation 1169/2011):
Gluten, Crustaceans, Eggs, Fish, Peanuts, Soybeans,
Milk, Tree nuts, Celery, Mustard, Sesame, SO2/Sulfites,
Lupin, Molluscs

PLC Role in Allergen Control:

1. SEGREGATION ENFORCEMENT:
   Line A: Peanut products (CONTAINS PEANUTS)
   Line B: Peanut-free products
   PLC: ห้าม product A ไป Line B โดยไม่มี proper CIP

// Allergen Tracking per Line:
DATA_BLOCK DB_AllergenControl
    line1_LastAllergen:     STRING[30];  // Last allergen processed
    line1_CIPCompleted:     BOOL;        // CIP done after allergen
    line2_LastAllergen:     STRING[30];
    line2_CIPCompleted:     BOOL;
END_DATA_BLOCK

// Before starting production on line:
// IF Contains(rNextRecipe.allergens, 'PEANUTS') AND
//    DB_AllergenControl.line1_LastAllergen = 'PEANUTS-FREE' THEN
//    // OK - same allergen category or no allergen
// ELSIF NOT DB_AllergenControl.line1_CIPCompleted THEN
//    // BLOCKED - CIP required before changeover to different allergen
//    bAllergenBlock := TRUE;
// END_IF;

2. LABEL VERIFICATION:
   Vision system reads label → verify allergen declaration matches recipe
   IF recipe.allergen_peanut AND NOT label.contains_peanut THEN
       REJECT_PRODUCT();
       ALARM('Label allergen mismatch');
   END_IF;

3. CLEANING VALIDATION:
   CIP before changing allergen class
   Swab test data linked to lot record
```

---

## Step 1267: Batch Record Automation

### Batch Record ใน Pharmaceutical

```
ELECTRONIC BATCH RECORD (EBR):
──────────────────────────────────────────────────────────────

Batch Record = เอกสาร record ทุกอย่างที่เกิดขึ้นใน batch หนึ่ง
FDA 21 CFR Part 211: Paper batch records required
21 CFR Part 11: Electronic batch records allowed (if validated)

Content of Batch Record:
├── Batch number and date
├── Product name and code
├── Master formula (amounts, parameters)
├── Actual amounts used (with lot numbers)
├── Actual process parameters (temperature, time, etc.)
├── In-process controls (test results)
├── Equipment used (ID, calibration status)
├── Personnel (who did what)
├── Deviations (if any)
└── Release authorization

// Electronic Batch Record (EBR) Data Structure in PLC:
DATA_BLOCK DB_BatchRecord
    STRUCT
        sBatchID:           STRING[20];     // BN-2024-0001
        sProductCode:       STRING[20];     // PROD-001-A
        sProductName:       STRING[50];     // Paracetamol 500mg
        rBatchSize_kg:      REAL;           // Planned: 500 kg
        rActualYield_kg:    REAL;           // Actual produced
        dtBatchStart:       DATE_AND_TIME;
        dtBatchEnd:         DATE_AND_TIME;
        sOperatorID:        STRING[20];
        sSupervisorID:      STRING[20];
        iMasterFormula:     INT;            // Recipe number
        
        // Process steps
        steps: ARRAY[1..50] OF STRUCT
            iStepNumber:    INT;
            sDescription:   STRING[80];
            dtStartTime:    DATE_AND_TIME;
            dtEndTime:      DATE_AND_TIME;
            rSetpoint:      REAL;
            rActual_Avg:    REAL;
            rActual_Min:    REAL;
            rActual_Max:    REAL;
            bInSpec:        BOOL;
            sComment:       STRING[100];
        END_STRUCT;
        
        // Deviations
        deviations: ARRAY[1..10] OF STRUCT
            iDeviationID:   INT;
            sDescription:   STRING[200];
            sImpact:        STRING[100];
            sAction:        STRING[200];
            bClosed:        BOOL;
        END_STRUCT;
        
        // Release
        bQualityReleased:   BOOL;
        sQualityManager:    STRING[20];
        dtRelease:          DATE_AND_TIME;
        sReleaseComment:    STRING[100];
    END_STRUCT;
END_DATA_BLOCK
```

---

## Step 1268: Recipe Management System

### จัดการ Recipe ใน Pharma

```
RECIPE MANAGEMENT ใน PHARMACEUTICAL:
──────────────────────────────────────────────────────────────

ISA-88 (S88) Standard for Batch Control:
├── Recipe: instructions สำหรับ batch production
├── Master Recipe: validated recipe (ไม่ควรแก้ง่าย)
├── Control Recipe: instance ของ master recipe สำหรับ batch หนึ่ง
└── Process: equipment + recipe = product

S88 Recipe Hierarchy:
General Recipe (บริษัท level)
  → Site Recipe (ที่ plant นั้น)
    → Master Recipe (สำหรับ equipment)
      → Control Recipe (สำหรับ batch หนึ่ง)

// MASTER RECIPE DATA STRUCTURE
DATA_BLOCK DB_MasterRecipes
    recipes: ARRAY[1..50] OF STRUCT
        sName:          STRING[30];     // Paracetamol 500mg - Granulation
        sVersion:       STRING[10];     // 3.2
        bApproved:      BOOL;           // QA approved?
        sApprovedBy:    STRING[20];     // QA Manager
        dtApproval:     DATE_AND_TIME;
        
        // Header
        rBatchSize_kg:  REAL;           // Standard batch size
        sEquipment:     STRING[20];     // Required equipment ID
        
        // Process steps
        steps: ARRAY[1..20] OF STRUCT
            sStepName:      STRING[30];
            iStepType:      INT;        // 1=Add, 2=Mix, 3=Heat, 4=Hold
            rParam1:        REAL;       // Primary parameter
            rParam1_Min:    REAL;       // Min acceptable
            rParam1_Max:    REAL;       // Max acceptable
            rParam2:        REAL;       // Secondary parameter
            tDuration:      TIME;       // Step duration (if time-based)
            bVerify:        BOOL;       // Requires operator verification
        END_STRUCT;
    END_STRUCT;
END_DATA_BLOCK

// RECIPE ACCESS CONTROL:
// Modification of master recipe requires:
// 1. Change Control (approval process)
// 2. Old version archived (not deleted)
// 3. QA re-approval of new version
// 4. Re-validation if major change
```

---

## Step 1269: IQ/OQ/PQ for Pharmaceutical Systems

### Qualification สำหรับ Pharma

```
IQ/OQ/PQ สำหรับ PHARMACEUTICAL PLC SYSTEMS:
──────────────────────────────────────────────────────────────

IQ (Installation Qualification):
"Hardware ถูกติดตั้งถูกต้องตาม design?"

IQ Checklist สำหรับ Pharmaceutical:
☐ PLC hardware matches DQ (Design Qual) specifications
☐ Environmental conditions (cleanroom grade, temp, humidity)
☐ UPS backup: confirmed runtime > required
☐ Network: isolated from internet, firewall present
☐ Clock: NTP synchronized, verified accuracy ±30s
☐ User access: roles configured per URS
☐ Antivirus: updated and active
☐ Backup: automated daily backup tested
☐ Software version: matches qualification baseline

──────────────────────────────────────────────────────────────

OQ (Operational Qualification):
"ระบบทำงานตาม functional spec?"

OQ Test Examples (Batch System):
TC-OQ-001: Recipe Management - Create Recipe
Steps:
  1. Login as Engineer
  2. Navigate to Recipe Management
  3. Create new recipe "Test-001" with 3 steps
Expected: Recipe created, saved, version 1.0 assigned

TC-OQ-002: Recipe Management - Edit Recipe
Steps:
  1. Login as Engineer
  2. Open recipe Test-001
  3. Modify Step 1 temperature from 60 to 65°C
Expected: 
  - Audit trail created: "Recipe Test-001 Step1 Temp: 60→65°C"
  - Version changed to 1.1
  - Status: "Pending Approval"

TC-OQ-015: Audit Trail - Tamper Detection
Steps:
  1. Attempt to modify audit trail via database (simulated attack)
Expected:
  - System detects modification (CRC check fails)
  - Alert logged
  - System halts (or alerts admin)

──────────────────────────────────────────────────────────────

PQ (Performance Qualification):
"ระบบ perform ตาม production requirements?"

PQ for Granulation System:
3 consecutive batches meeting all specifications:
├── Batch 1: Moisture content 2.1% (spec: 1.5-3.0%) PASS
├── Batch 2: Moisture content 2.3% PASS
├── Batch 3: Moisture content 1.9% PASS
└── Yield: 97.2%, 98.1%, 96.8% (spec: > 95%) PASS
→ PQ: PASSED
```

---

## Step 1270: Data Integrity and Backup

### Data Integrity ใน Pharmaceutical

```
DATA INTEGRITY SYSTEM:
──────────────────────────────────────────────────────────────

ปัญหาที่ FDA ตรวจพบบ่อย:
- Deleting failed test results
- Backdating records
- Sharing login credentials
- Trial injections (run tests, discard bad results)

วิธีป้องกันด้วย PLC/SCADA:

1. CHECKSUMS บน critical data:
// ทุก batch record มี CRC/MD5 checksum
// ถ้า record ถูก modify → checksum ไม่ตรง → detected
// bDataIntegrityOK := (Calculate_CRC(DB_BatchRecord) = DB_BatchRecord.iStoredCRC);

2. SEQUENCE NUMBERS:
// ทุก event ใน audit trail มี sequential number
// ถ้าตัดเลขออก → sequence ขาด → detected

3. WRITE-ONCE STORAGE:
// ใช้ WORM (Write Once Read Many) storage
// สำหรับ batch records: cannot be modified after save

4. ROLE SEPARATION:
// Operator: run equipment, ห้ามแก้ batch records
// Supervisor: review, sign off
// QA: release batch
// Admin: manage users (ไม่ใช่ production data)
// IT: maintain infrastructure (ไม่เข้าถึง production data)

5. BACKUP STRATEGY (3-2-1 Rule):
// 3 copies ของ data
// 2 different media types
// 1 offsite copy

// Backup Schedule:
// Real-time: Mirror ไปยัง secondary server
// Daily: Backup ไป network storage (on-site)
// Weekly: Backup ไป cloud (off-site)
// Monthly: Archive ไป tape (long-term)

// Retention Period (21 CFR Part 211):
// Batch records: 1 year after expiry of batch + 3 years
// (ถ้า product expiry = 2 years → เก็บ 5 ปี)
```

---

## Step 1271: Environmental Monitoring

### ตรวจสอบสภาพแวดล้อม

```
ENVIRONMENTAL MONITORING สำหรับ PHARMACEUTICAL:
──────────────────────────────────────────────────────────────

Cleanroom Classification (ISO 14644 / EU GMP Annex 1):
ISO 5 (EU Grade A): < 3,520 particles/m³ ≥0.5μm (critical zone)
ISO 6 (EU Grade B): < 35,200 particles/m³ (background to A)
ISO 7 (EU Grade C): < 352,000 particles/m³ (less critical)
ISO 8 (EU Grade D): < 3,520,000 particles/m³ (non-sterile)

Parameters to Monitor:
├── Particle count (particle counter)
├── Temperature (±2°C allowed variation)
├── Relative Humidity (30-65% RH, ±5%)
├── Differential pressure (positive for critical)
│   ├── ISO5 vs corridor: +15 Pa
│   └── ISO7 vs ISO8: +10 Pa
└── Air changes per hour (HEPA filtered)

// Environmental Monitoring PLC
FUNCTION_BLOCK FB_EnvMonitor
VAR_INPUT
    rTemp_C:        REAL;       // Temperature
    rRH_Pct:        REAL;       // Relative humidity
    rDiffPress_Pa:  REAL;       // Differential pressure
    iParticleCount: DINT;       // Particles ≥0.5μm per m³
END_VAR
VAR_OUTPUT
    bAllInSpec:     BOOL;       // All parameters in spec
    bAlarm_Temp:    BOOL;       // Temperature out of spec
    bAlarm_RH:      BOOL;       // Humidity out of spec
    bAlarm_Press:   BOOL;       // Pressure out of spec
    bAlarm_Particle:BOOL;       // Particle count exceeded
    bBatch_Halt:    BOOL;       // Halt production (critical deviation)
END_VAR

// Check each parameter:
// Temp: 20-25°C (example)
// bAlarm_Temp := (rTemp_C < 20.0) OR (rTemp_C > 25.0);

// Pressure: > 15 Pa differential
// bAlarm_Press := rDiffPress_Pa < 10.0;  // Warning at 10 Pa

// Particle count (ISO 5 limit = 3520 particles/m³ ≥0.5μm)
// bAlarm_Particle := iParticleCount > 3520;

// Critical: halt batch if alarm persists > 5 min
// IF bAlarm_Any AND tAlarmDuration >= T#5m THEN
//     bBatch_Halt := TRUE;
//     // Record deviation in batch record
//     WriteDeviation('ENV-001', 'Environmental alarm > 5 min');
// END_IF;
```

---

## Step 1272: Pharmaceutical Mixing System

### Pharmaceutical Granulator

```
PHARMACEUTICAL GRANULATION SYSTEM:
──────────────────────────────────────────────────────────────

High Shear Granulator Process:
1. LOAD API + Excipients into bowl
2. DRY MIX: mix dry powder (2 min)
3. ADD BINDER: add liquid binder while mixing
4. WET MIX: mix until granules form (3-5 min)
5. SCREEN: discharge through screen
6. DRY: fluid bed dryer (target moisture)
7. MILL: mill to target particle size

PLC System Monitors:
├── Motor power/torque (endpoint detection)
├── Product temperature
├── Chopper speed
├── Impeller speed
└── Mass (load cell)

// ENDPOINT DETECTION by Power Signature:
// Granule formation changes viscosity → motor power changes
// Monitor: d(Power)/dt ≈ 0 → granule endpoint reached

FUNCTION_BLOCK FB_GranulationEndpoint
VAR_INPUT
    rMotorPower_kW:     REAL;       // Current motor power
    tSampleInterval:    TIME := T#1s;
END_VAR
VAR_OUTPUT
    bEndpointReached:   BOOL;       // Endpoint detected
    rPowerTrend:        REAL;       // Rate of change
END_VAR
VAR
    rPowerLast:     REAL;
    tSampleTimer:   TON;
    rSlope_1min:    ARRAY[0..59] OF REAL;  // 1-minute history
    iHistoryIndex:  INT;
END_VAR

// Every second: calculate slope
// IF tSampleTimer.Q THEN
//     rPowerTrend := rMotorPower_kW - rPowerLast;
//     rSlope_1min[iHistoryIndex] := rPowerTrend;
//     iHistoryIndex := (iHistoryIndex + 1) MOD 60;
//     rPowerLast := rMotorPower_kW;
//     
//     // Average slope over 1 minute
//     rAvgSlope := Average(rSlope_1min);
//     
//     // Endpoint: slope near zero AND power stable
//     bEndpointReached := ABS(rAvgSlope) < 0.05;  // < 50W/min change
// END_IF;
```

---

## Step 1273: Filling and Packaging Control

### ระบบบรรจุและบรรจุภัณฑ์

```
PHARMACEUTICAL FILLING SYSTEM:
──────────────────────────────────────────────────────────────

Tablet Coating → Sorting → Filling → Sealing → Labeling → Packing

FILLING MACHINE (Tablet Counter/Filler):
├── Fill by count (x tablets per bottle)
├── Vibratory feeder → counting sensor
├── Reject: under-count or over-count bottles

// Tablet Counter Control
FUNCTION_BLOCK FB_TabletCounter
VAR_INPUT
    xTabletSensor:  BOOL;       // Photoelectric sensor (each tablet)
    iTarget:        INT := 100; // Tablets per bottle
    rUnderFill_Pct: REAL := -2; // -2% tolerance
    rOverFill_Pct:  REAL := 0;  // 0% tolerance (never overfill pharma)
END_VAR
VAR_OUTPUT
    iCount:         INT;        // Current count
    bFillComplete:  BOOL;       // Target reached
    bReject:        BOOL;       // Reject this bottle
END_VAR

// Count tablets:
// Rising edge of xTabletSensor → iCount++
// ถ้า iCount = iTarget → close gate, bFillComplete = TRUE

// Check tolerance:
// Overfill > target → always reject (patient safety)
// Underfill: tolerate down to -2% (target=100 → 98 tablets OK)
// iMinCount := INT_TO_REAL(iTarget) × (100.0 + rUnderFill_Pct) / 100.0;
// IF iCount < iMinCount → reject (underfill)

SERIALIZATION AND AGGREGATION:
// ทุก bottle ต้องมี unique serial number (GS1 DataMatrix)
// Regulation: EU Falsified Medicines Directive (FMD)
//             US Drug Supply Chain Security Act (DSCSA)
// 
// PLC interface with serialization system:
// 1. Request serial number from server
// 2. Print label with DataMatrix code
// 3. Verify print quality (camera)
// 4. Associate serial to batch record
// ถ้า print fail → reject bottle
```

---

## Step 1274: GMP Documentation Automation

### สร้าง GMP Documents อัตโนมัติ

```
AUTOMATED GMP DOCUMENTATION:
──────────────────────────────────────────────────────────────

แทนที่จะให้ operator กรอก batch record ด้วยมือ:
PLC รวบรวม data → SCADA/MES สร้าง PDF อัตโนมัติ

BATCH RELEASE WORKFLOW:
PLC → collect all data → MES generates draft EBR
↓
QA reviews EBR in MES
↓
QA electronic sign-off (dual signature)
↓
Batch released → EBR archived (read-only)

// ข้อมูลที่ PLC ต้องเก็บ:
// 1. Time stamps (NTP)
// 2. Process parameters per step
// 3. Equipment IDs
// 4. Alarms and deviations
// 5. Operator IDs (login/logout)
// 6. Equipment cleaning status (CIP complete?)
// 7. Calibration status of instruments

// DEVIATION MANAGEMENT:
// เมื่อ parameter เกิน limit:
// 1. Log deviation ใน DB_BatchRecord.deviations[]
// 2. Alert supervisor (alarm + HMI)
// 3. Operator must enter:
//    - Description of deviation
//    - Immediate action taken
// 4. Supervisor reviews and signs
// 5. QA review before batch release

// DEVIATION EXAMPLE:
// "Temperature exceeded maximum limit of 65°C
//  Actual: 67.2°C at 14:32:05
//  Duration: 3 minutes 12 seconds
//  Action: Reduced steam flow. Temperature returned to 63°C."
```

---

## Step 1275: Food Manufacturing - HACCP

### HACCP Implementation ใน PLC

```
HACCP (Hazard Analysis Critical Control Points):
──────────────────────────────────────────────────────────────

HACCP = ระบบป้องกัน food safety hazards
ต้องระบุ: Critical Control Points (CCPs) + Critical Limits

Example: Pasteurization (CCP-1)
Critical Limit: ≥ 72°C for ≥ 15 seconds
Monitoring: Continuous temperature measurement
Corrective Action: ถ้า T < 72°C → divert flow (reject)

// HACCP CCP MONITOR
FUNCTION_BLOCK FB_HACCP_CCP
VAR_INPUT
    rActualTemp:    REAL;           // Actual temperature
    rCriticalLimit: REAL := 72.0;   // Critical limit
    tMinHoldTime:   TIME := T#15s;  // Minimum hold time
END_VAR
VAR_OUTPUT
    bCCPPassed:     BOOL;           // Product meets CCP
    bCCPFail:       BOOL;           // CCP violated → divert
    rHoldTime_s:    REAL;           // Current hold time at CCP temp
    iCCPViolations: INT;            // Count of violations today
END_VAR
VAR
    tHoldTimer:     TON;
    bLastCCPStatus: BOOL;
END_VAR

// PASTEURIZATION HOLD TIME:
// Product must be at ≥ 72°C for ≥ 15 continuous seconds
// ANY drop below 72°C → reset hold timer

// IF rActualTemp >= rCriticalLimit THEN
//     tHoldTimer(IN := TRUE, PT := tMinHoldTime);
// ELSE
//     tHoldTimer(IN := FALSE);  // RESET if below critical limit
// END_IF;

// bCCPPassed := tHoldTimer.Q;  // 15 seconds reached
// bCCPFail := rActualTemp < rCriticalLimit AND tHoldTimer.ET > T#0s AND NOT tHoldTimer.Q;
//   ← Temperature dropped before 15 seconds completed

// Corrective action:
// IF bCCPFail THEN
//     xDivertValve := TRUE;   // Divert product (don't allow to packaging)
//     LogCCPViolation();      // Record in HACCP log
//     iCCPViolations := iCCPViolations + 1;
//     AlertOperator('HACCP CCP-1 Failure - Product Diverted');
// END_IF;
```

---

## Step 1276: FSC (Food Safety Control) Integration

### Farm-to-Fork Traceability

```
FOOD TRACEABILITY:
──────────────────────────────────────────────────────────────

Regulation: EU 178/2002 (General Food Law)
US: FSMA (Food Safety Modernization Act)
→ ต้อง trace ได้ 1 step back (supplier) + 1 step forward (customer)

GS1 Standards:
├── GTIN (Global Trade Item Number): รหัสสินค้า
├── SSCC (Serial Shipping Container Code): รหัส pallet
├── Batch/Lot number: รหัส batch
└── Best before date: วันหมดอายุ

DATA to Record per Batch:
├── Raw material: supplier, lot, certificate of analysis
├── Process: date, time, parameters
├── Equipment: CIP status, calibration
├── Finished product: lot, qty, storage conditions
└── Distribution: customer, delivery date

// TRACEABILITY DATA BLOCK
DATA_BLOCK DB_TraceabilityRecord
    STRUCT
        sProductionLot: STRING[20];     // LOT-2024-001
        sProductCode:   STRING[20];     // P001-500ML
        dtProduction:   DATE_AND_TIME;
        
        // Raw materials
        ingredients: ARRAY[1..10] OF STRUCT
            sCode:          STRING[20];
            sDescription:   STRING[50];
            rQuantity_kg:   REAL;
            sSupplierLot:   STRING[20];
            sSupplierName:  STRING[30];
            dtReceived:     DATE_AND_TIME;
        END_STRUCT;
        
        // Process
        rPasteurizationTemp:    REAL;
        rActualHoldTime_s:      REAL;
        bCCPsAllPassed:         BOOL;
        
        // Distribution
        customers: ARRAY[1..20] OF STRUCT
            sCustomerCode:  STRING[20];
            iQty_Cases:     INT;
            dtShipped:      DATE_AND_TIME;
        END_STRUCT;
    END_STRUCT;
END_DATA_BLOCK
```

---

## Step 1277: Clean Room HVAC Control

### ควบคุม HVAC ใน Clean Room

```
CLEAN ROOM HVAC PLC CONTROL:
──────────────────────────────────────────────────────────────

Clean room HVAC ต้องการ:
├── HEPA filtration (H14: 99.995% at 0.3μm)
├── Positive pressure differential
├── Temperature + humidity control
└── Air change rate (20-60 ACH depending on class)

// FB_CleanroomHVAC
FUNCTION_BLOCK FB_CleanroomHVAC
VAR_INPUT
    rTemp_Setpoint:     REAL := 22.0;   // °C
    rRH_Setpoint:       REAL := 45.0;   // %
    rDiffPress_SP:      REAL := 15.0;   // Pa
    bHVAC_Enable:       BOOL;
END_VAR
VAR_OUTPUT
    xSupplyFan:         BOOL;           // Supply air fan
    xReturnFan:         BOOL;           // Return air fan
    xCoolingValve:      BOOL;           // Chilled water valve
    xHeatingValve:      BOOL;           // Hot water valve
    xHumidifier:        BOOL;           // Steam humidifier
    xDehumidifier:      BOOL;           // Dehumidifier
    rFanSpeed_Pct:      REAL;           // VFD speed %
    bPressureOK:        BOOL;           // Differential pressure OK
END_VAR
VAR
    fbTempPID:      FB_PID_Control;     // Temperature PID
    fbRH_PID:       FB_PID_Control;     // Humidity PID
    fbPress_PID:    FB_PID_Control;     // Pressure control (fan speed)
END_VAR

// Temperature control:
// fbTempPID(SP := rTemp_Setpoint, PV := rActualTemp);
// IF fbTempPID.CV > 0 THEN xCoolingValve := TRUE; xHeatingValve := FALSE;
// ELSE xHeatingValve := TRUE; xCoolingValve := FALSE; END_IF;

// Pressure control (fan speed):
// fbPress_PID(SP := rDiffPress_SP, PV := rActualDiffPress);
// rFanSpeed_Pct := fbPress_PID.CV;

// Environmental Monitoring alarm integration:
// bPressureOK := rActualDiffPress >= (rDiffPress_SP - 3.0);  // ±3 Pa tolerance
// IF NOT bPressureOK THEN → Alert, log, consider batch halt
```

---

## Step 1278: Temperature Mapping

### Temperature Mapping สำหรับ Cold Chain

```
TEMPERATURE MAPPING (Cold Chain):
──────────────────────────────────────────────────────────────

Temperature-sensitive products (vaccines, biologics):
ต้อง map ทุก storage area และ equipment

Temperature Mapping PLC:
ต่อ thermocouple / RTD หลายจุด ใน cold room
บันทึกอย่างต่อเนื่อง
Generate mapping report

// Cold Room Monitoring (12 sensors)
FUNCTION_BLOCK FB_TempMapping
VAR_INPUT
    rSensors:   ARRAY[1..12] OF REAL;   // Temperature readings
    rSetpoint:  REAL := 5.0;            // Target temperature
    rAlarmHigh: REAL := 8.0;            // Alarm high limit
    rAlarmLow:  REAL := 2.0;            // Alarm low limit
END_VAR
VAR_OUTPUT
    rAvgTemp:   REAL;               // Average temperature
    rMaxTemp:   REAL;               // Maximum (hottest spot)
    rMinTemp:   REAL;               // Minimum (coldest spot)
    rDeltaT:    REAL;               // Max - Min (uniformity)
    bAnyAlarm:  BOOL;               // Any sensor out of spec
    bAlarms:    ARRAY[1..12] OF BOOL; // Per-sensor alarms
END_VAR

// Calculate statistics:
// rMaxTemp := sensors[1]; rMinTemp := sensors[1];
// rAvgTemp := 0;
// FOR i := 1 TO 12 DO
//     IF sensors[i] > rMaxTemp THEN rMaxTemp := sensors[i]; END_IF;
//     IF sensors[i] < rMinTemp THEN rMinTemp := sensors[i]; END_IF;
//     rAvgTemp := rAvgTemp + sensors[i];
//     bAlarms[i] := (sensors[i] < rAlarmLow) OR (sensors[i] > rAlarmHigh);
// END_FOR;
// rAvgTemp := rAvgTemp / 12.0;
// rDeltaT := rMaxTemp - rMinTemp;

// WHO PQS Requirement for vaccines:
// rDeltaT < 2°C (temperature uniformity)
// ถ้า > 2°C → investigate (possible dead spot)
```

---

## Step 1279: Compliance Reporting

### สร้าง Reports สำหรับ Compliance

```
COMPLIANCE REPORTING:
──────────────────────────────────────────────────────────────

Reports ที่ต้องสร้าง:
1. Batch Release Report (ทุก batch)
2. Deviation Report (เมื่อมี deviation)
3. Environmental Monitoring Report (รายวัน/รายเดือน)
4. Equipment Calibration Report (รายปี)
5. Annual Product Review (APR)

สร้างจาก PLC/SCADA data:
PLC → Historian → SCADA/MES → Report Generator

// ข้อมูลที่ PLC ต้องเก็บสำหรับ APR:
// - ทุก batch ที่ผลิต: lot, yield, parameters
// - ทุก deviation ที่เกิด
// - ทุก OOS (Out Of Specification) result
// - ทุก equipment downtime
// - ทุก CAPA (Corrective and Preventive Action)

// EXPORT ไป Excel/PDF:
// SCADA system (WinCC, FactoryTalk) → reporting module
// Python script อ่านจาก historian (OPC UA)
// → สร้าง Excel template ด้วย openpyxl
// → ส่งอีเมลล์อัตโนมัติ (ทุกวันจันทร์)

// DATA RETENTION:
// Batch records: min 5 years (EU) / 3 years after expiry (US)
// Environmental monitoring: 1 year minimum
// Calibration records: life of instrument + 1 year
// Qualification docs: life of facility
```

---

## Step 1280: Lab Project - Pharmaceutical Mixing and Filling

### Lab: สร้าง Pharma Batch System

```
LAB: PHARMACEUTICAL MIXING AND FILLING SYSTEM
══════════════════════════════════════════════════════════════

ระบบที่จำลอง:
- High Shear Granulator (mixing)
- Fluid Bed Dryer (drying)
- Filling Machine (capsule filling)

TASK 1: Recipe Management (20%)
─────────────────────────────────────────────────────────────
1. สร้าง DB_MasterRecipes สำหรับ 3 recipes:
   - Recipe A: Paracetamol 500mg (3 steps)
   - Recipe B: Ibuprofen 400mg (4 steps)  
   - Recipe C: Amoxicillin 250mg (5 steps)

2. Recipe access control:
   - Only Engineer+ can create/modify
   - Modification requires change control (supervisor approval)
   - Version tracking

3. Recipe selection:
   - Operator selects recipe from approved list
   - Creates control recipe instance for batch
   - Cannot modify master recipe from production screen

TASK 2: Batch Processing (35%)
─────────────────────────────────────────────────────────────
1. FB_BatchProcessor:
   - Load recipe parameters
   - Execute steps in sequence
   - Record actual vs setpoint per step
   - Endpoint detection (for granulation step)
   - Handle deviations (temp out of spec → log + alert)

2. CIP integration:
   - Check equipment CIP status before start
   - Block batch start if CIP not completed
   - Log CIP record in batch record

3. Batch Record automation:
   - Auto-fill all timestamps
   - Record process parameters
   - Log all deviations
   - Generate unique batch ID

TASK 3: Data Integrity (20%)
─────────────────────────────────────────────────────────────
1. Audit trail:
   - Log all parameter changes with user/time
   - CRC checksum on critical data

2. User authentication:
   - 4 roles: Operator, Supervisor, QA, Admin
   - Session timeout after 15 min inactivity
   - Dual signature for critical steps

3. Alarm logging:
   - All alarms with acknowledgement log
   - Unacknowledged alarm cannot close screen

TASK 4: Compliance Report (25%)
─────────────────────────────────────────────────────────────
1. Electronic Batch Record:
   - Generate draft EBR after batch complete
   - Include all required GMP data
   - QA electronic sign-off capability

2. HACCP monitoring:
   - Define 2 CCPs for the process
   - Monitor with limits
   - Log violations with corrective actions

3. Environmental monitoring:
   - 4 temperature sensors + 1 humidity
   - Alarm if out of spec
   - Daily report generation

DELIVERABLES:
├── TIA Portal project (complete)
├── DB structures documentation
├── Sample Batch Record (PDF) for 1 batch
├── Sample Audit Trail (Excel)
├── Deviation log with corrective actions
└── Brief report: GAMP 5 classification of your FBs

GRADING:
├── Recipe management: 20%
├── Batch processing: 25%
├── Data integrity: 25%
├── Compliance: 20%
└── Code quality: 10%

BONUS:
- Implement F0 calculation for sterilization step
- Add barcode reader integration for raw material check
- Generate automated PDF batch record using Python
```

---

## สรุป Part 64

| Topic | สิ่งสำคัญ |
|-------|----------|
| GMP | Documentation, Data Integrity, ALCOA+ |
| 21 CFR Part 11 | Audit trail, Electronic signatures |
| CIP/SIP | Sequence control, parameter validation |
| Recipe | S88, version control, change control |
| Batch Record | Electronic, automated, tamper-proof |
| HACCP | CCPs, critical limits, corrective actions |
| Traceability | 1 step back / 1 step forward (GS1) |

---

## Navigation

- [← Part 63: Welding Automation](./part63_welding_automation.md)
- [→ Part 65: Oil and Gas Automation](./part65_oil_gas_automation.md)
- [↑ สารบัญหลัก](../README.md)
