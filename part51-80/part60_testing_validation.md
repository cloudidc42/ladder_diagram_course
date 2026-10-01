# Part 60: Testing and Validation for PLC Systems (Steps 1181-1200)

## บทนำ: ทำไม Testing สำคัญ

ใน Software development ทั่วไป: ถ้า bug → fix แล้ว deploy ใหม่
ใน Industrial PLC: ถ้า bug → อาจทำให้คนเสียชีวิต, โรงงานหยุด, ผลิตภัณฑ์เสีย

Testing ใน PLC ต้องการ:
- ความเป็นระบบ (systematic)
- เอกสารที่สมบูรณ์ (documentation)
- การ trace ได้ (traceability)
- Regulatory compliance (pharma, food, nuclear)

---

## Step 1181: FAT และ SAT Fundamentals

### Factory Acceptance Test และ Site Acceptance Test

```
FAT vs SAT:
──────────────────────────────────────────────────────────────

FAT (Factory Acceptance Test):
ที่ไหน: ที่โรงงานผู้สร้าง (system integrator)
เมื่อไหร่: ก่อนส่งมอบระบบ
ใคร: ลูกค้า + ผู้สร้าง + บางครั้ง third-party
วัตถุประสงค์:
├── ตรวจว่าระบบทำงานตาม functional spec
├── ตรวจ hardware (wiring, panel)
├── ตรวจ software (PLC logic, HMI)
└── ตรวจ communication (network, I/O)

SAT (Site Acceptance Test):
ที่ไหน: ที่ production site (โรงงานลูกค้า)
เมื่อไหร่: หลังติดตั้งและ commissioning
ใคร: ลูกค้า + ผู้สร้าง
วัตถุประสงค์:
├── ยืนยันว่าระบบทำงานในสภาวะจริง
├── Test กับ real process (real product, real materials)
├── ตรวจ interlocks และ safety
└── Performance test (throughput, accuracy)

FAT/SAT RELATIONSHIP:
FAT (ที่ผู้ผลิต) → ส่งมอบ → ติดตั้ง → SAT (ที่โรงงาน)
                                    ↗
                           Commissioning
                           (Debug + Fine-tune)
```

---

## Step 1182: Test Case Design Principles

### การออกแบบ Test Cases

```
TEST CASE STRUCTURE:
──────────────────────────────────────────────────────────────

ทุก test case ต้องมี:
├── Test Case ID:        TC-FILL-001
├── Test Name:           Normal Fill Cycle
├── Requirement Ref:     FS-3.2.1 (Functional Spec section)
├── Preconditions:       Machine in IDLE state, bottle present
├── Test Steps:          Step-by-step actions
├── Expected Result:     What should happen
├── Actual Result:       What actually happened (filled during test)
├── Pass/Fail:           PASS / FAIL
├── Date:               2024-03-15
└── Tester:             Somchai (sign)

TEST CASE ตัวอย่าง:
──────────────────────────────────────────────────────────────
TC-FILL-001: Normal Fill Cycle (Happy Path)
──────────────────────────────────────────────
Preconditions:
  - Machine in IDLE state
  - Bottle present at fill station
  - Fill valve closed
  - Tank level above minimum

Steps:
  1. Press START button (xStartPB)
  2. Wait for PRESTART state (max 10 sec)
  3. Verify conveyor starts
  4. Verify bottle positioned under nozzle
  5. Press FILL_START (or auto-trigger)
  6. Observe fill valve opens
  7. Wait for fill complete (target: 500mL ±5mL)
  8. Verify fill valve closes

Expected Results:
  Step 3: xConveyorRun = TRUE
  Step 6: xFillValve = TRUE within 2 sec
  Step 7: Elapsed time: 8-12 sec (500mL/55mL/s)
  Step 8: xFillValve = FALSE, xFillComplete = TRUE
  HMI: State = RUNNING, Volume = 500.x mL

Actual Result: [Fill during test]
Pass/Fail: [P/F]
```

---

## Step 1183: Equivalence Partitioning

### แบ่งข้อมูล Test เป็นกลุ่ม

```
EQUIVALENCE PARTITIONING:
──────────────────────────────────────────────────────────────

แนวคิด: แทนที่จะ test ทุกค่า → แบ่งเป็น "partitions"
ค่าใน partition เดียวกัน → ควรให้ผลเดียวกัน

ตัวอย่าง: ตรวจสอบ Fill Volume (0-1000 mL)
Valid Range: 100-800 mL

Partitions:
P1: Below minimum (<100 mL)     → ควร reject หรือ error
P2: Valid range (100-800 mL)   → ควร accept
P3: Above maximum (>800 mL)    → ควร reject หรือ warning
P4: Invalid type (negative, text) → ควร reject

Test จาก each partition (1 test ต่อ partition):
P1: Test 50 mL  → Expected: Error "Volume too low"
P2: Test 500 mL → Expected: Accept, fill normally
P3: Test 900 mL → Expected: Error "Volume too high"
P4: Test -1 mL  → Expected: Reject (validation error)

ประหยัด test cases โดยไม่เสีย coverage!
```

---

## Step 1184: Boundary Value Analysis

### Test ที่ขอบเขต

```
BOUNDARY VALUE ANALYSIS:
──────────────────────────────────────────────────────────────

Bugs มักอยู่ที่ขอบเขต (off-by-one errors, >= vs >)

ตัวอย่าง: Fill Volume (Valid: 100-800 mL)
Test ที่ boundary:

Boundary Points:
├── 99 mL   (just below min) → should reject
├── 100 mL  (exact min)      → should accept
├── 101 mL  (just above min) → should accept
├── 500 mL  (middle)         → should accept
├── 799 mL  (just below max) → should accept
├── 800 mL  (exact max)      → should accept
└── 801 mL  (just above max) → should reject

Bug ที่พบบ่อย:
// WRONG (off-by-one):
IF rVolume > 100 THEN  // ← 100 เองจะถูก reject (ผิด)

// CORRECT:
IF rVolume >= 100 THEN  // ← 100 ถูก accept (ถูก)

PLC Code ที่ test:
IF (rFillVolume >= FILL_MIN) AND (rFillVolume <= FILL_MAX) THEN
    bVolume_Valid := TRUE;
ELSE
    bVolume_Valid := FALSE;
END_IF;
```

---

## Step 1185: Integration Testing

### ทดสอบการทำงานร่วมกัน

```
INTEGRATION TESTING:
──────────────────────────────────────────────────────────────

Unit Test: ทดสอบ FB เดี่ยว
Integration Test: ทดสอบการทำงานร่วมกันของ FBs หลายตัว

Test Levels:
1. Unit Test: FB_FillControl ทำงานถูก?
2. Integration Test: FB_FillControl + FB_ConveyorCtrl ทำงานด้วยกันได้?
3. System Test: ทั้งระบบทำงานตาม spec?
4. Acceptance Test: ลูกค้ายอมรับ?

Integration Test สำหรับ Filling + Conveyor:
TC-INT-001: Conveyor Stops When Fill Starts
──────────────────────────────────────────────
Preconditions:
  - System in RUNNING state
  - Bottle at fill station
  
Steps:
  1. xBottlePresent = TRUE (bottle detected)
  2. FB_FillControl triggers: xFillStart = TRUE
  
Expected:
  - FB_ConveyorCtrl: xConveyorRun = FALSE (conveyor stops)
  - FB_FillControl: xFillValve = TRUE (fill begins)
  - HMI: Zone 3 = FILLING
  - Timeout: conveyor must stop within 500ms

Test ใน Simulation (S7-PLCSIM Advanced):
// Python test script
def test_conveyor_stops_on_fill():
    # Set initial conditions
    plc.write('DB_Test.xBottlePresent', True)
    plc.write('DB_Test.xSystemRunning', True)
    
    # Trigger fill
    plc.write('DB_FillControl.xFillStart', True)
    time.sleep(0.5)  # Wait 500ms
    
    # Verify
    conveyor = plc.read('DB_ConveyorCtrl.xConveyorRun')
    fill_valve = plc.read('DB_FillControl.xFillValve')
    
    assert conveyor == False,  "FAIL: Conveyor still running"
    assert fill_valve == True, "FAIL: Fill valve not open"
    print("PASS: TC-INT-001")
```

---

## Step 1186: Regression Testing

### ทดสอบว่าการแก้ไขไม่ทำลาย feature เก่า

```
REGRESSION TESTING:
──────────────────────────────────────────────────────────────

ปัญหา: แก้ feature ใหม่ → ทำให้ feature เก่าพัง (regression)
วิธีแก้: Run regression test suite หลังทุกครั้งที่แก้ code

Regression Test Suite สำหรับ Filling Machine:
┌──────────────────────────────────────────────────────────┐
│ REGRESSION TEST SUITE v2.1 - Filling Machine             │
├──────────────────────────────────────────────────────────┤
│ TC-001: Normal fill cycle (happy path)                   │
│ TC-002: E-stop during filling → immediate stop          │
│ TC-003: Low level alarm → stop filling, alert           │
│ TC-004: Fill timeout → fault state                      │
│ TC-005: Jam detection → conveyor stop, alarm            │
│ TC-006: Recipe change → parameters update               │
│ TC-007: Power cycle → system restarts correctly         │
│ TC-008: Communication loss → safe state                 │
│ TC-009: HMI reset from fault → returns to idle         │
│ TC-010: Speed change → fill time adjusts               │
└──────────────────────────────────────────────────────────┘

จะ run เมื่อไหร่:
├── ทุกครั้งที่ feature merge ไป develop
├── ทุกครั้งที่ release ไป main
└── CI/CD pipeline รัน auto

ถ้า regression test fail:
→ ห้าม merge (CI/CD blocks merge)
→ แก้ bug ก่อน
→ Re-run suite
```

---

## Step 1187: Simulation-Based Testing

### ทดสอบด้วย PLC Simulator

```
SIMULATION-BASED TESTING:
──────────────────────────────────────────────────────────────

Siemens S7-PLCSIM Advanced:
├── รัน S7-1500 program โดยไม่ต้องมี hardware
├── เชื่อมต่อกับ Factory I/O (process simulation)
└── API สำหรับ automated testing (Python/C++)

Allen-Bradley Logix Emulate:
├── Emulate ControlLogix processor
├── เชื่อมต่อกับ simulation environment
└── Inductive Automation TestAgent (Ignition)

Python Test Framework สำหรับ PLCSIM:
```

```python
# test_filling_machine.py
# Automated test สำหรับ Filling Machine
# ใช้ snap7 library เพื่อเชื่อมต่อกับ PLCSIM Advanced

import snap7
import time
import unittest

class FillingMachineTests(unittest.TestCase):
    """Automated tests for Filling Machine PLC"""
    
    @classmethod
    def setUpClass(cls):
        """เชื่อมต่อกับ PLCSIM Advanced ก่อนรัน tests"""
        cls.plc = snap7.client.Client()
        cls.plc.connect('127.0.0.1', 0, 1)  # PLCSIM local
        print("Connected to PLCSIM Advanced")
    
    def setUp(self):
        """Reset PLC ก่อน test แต่ละ test case"""
        self._write_bool(100, 0, 0, False)  # Reset xStartPB
        self._write_bool(100, 0, 1, False)  # Reset xStopPB
        time.sleep(0.5)
    
    def test_normal_fill_cycle(self):
        """TC-001: Normal fill cycle"""
        # Preconditions
        self._write_bool(100, 0, 2, True)   # xBottlePresent = True
        
        # Start machine
        self._write_bool(100, 0, 0, True)   # xStartPB
        time.sleep(0.1)
        self._write_bool(100, 0, 0, False)  # Release button
        
        # Wait for RUNNING state
        timeout = 10  # 10 seconds
        start = time.time()
        while time.time() - start < timeout:
            state = self._read_int(200, 0)  # eCurrentState
            if state == 2:  # STATE_RUNNING
                break
            time.sleep(0.1)
        
        self.assertEqual(state, 2, f"Expected RUNNING(2), got {state}")
        
        # Wait for fill complete
        time.sleep(12)  # Fill takes ~10 sec
        fill_done = self._read_bool(200, 2, 0)  # xFillComplete
        self.assertTrue(fill_done, "Fill did not complete")
        
        print("PASS: TC-001 Normal Fill Cycle")
    
    def test_estop_during_filling(self):
        """TC-002: E-stop during filling stops machine immediately"""
        # Start filling
        self._write_bool(100, 0, 0, True)   # xStartPB
        time.sleep(6)   # Wait until filling
        
        # Trigger E-stop
        self._write_bool(100, 0, 5, False)  # xEStop = False (NC contact)
        time.sleep(0.2)  # 200ms response time max
        
        # Verify all outputs are off
        conveyor = self._read_bool(200, 0, 0)  # xConveyorRun
        fill_valve = self._read_bool(200, 0, 1)  # xFillValve
        
        self.assertFalse(conveyor, "Conveyor still running after E-stop")
        self.assertFalse(fill_valve, "Fill valve still open after E-stop")
        
        print("PASS: TC-002 E-stop during filling")
    
    def _write_bool(self, db, byte, bit, value):
        """เขียน boolean ไป PLC DB"""
        data = self.plc.db_read(db, byte, 1)
        snap7.util.set_bool(data, 0, bit, value)
        self.plc.db_write(db, byte, data)
    
    def _read_bool(self, db, byte, bit):
        """อ่าน boolean จาก PLC DB"""
        data = self.plc.db_read(db, byte, 1)
        return snap7.util.get_bool(data, 0, bit)
    
    def _read_int(self, db, byte):
        """อ่าน INT จาก PLC DB"""
        data = self.plc.db_read(db, byte, 2)
        return snap7.util.get_int(data, 0)

if __name__ == '__main__':
    unittest.main(verbosity=2)
```

---

## Step 1188: GAMP 5 Overview

### Good Automated Manufacturing Practice

```
GAMP 5 FRAMEWORK:
──────────────────────────────────────────────────────────────

GAMP 5 = Good Automated Manufacturing Practice
เป็น guideline ของ ISPE (International Society for
Pharmaceutical Engineering) สำหรับ validation ใน pharma

GAMP 5 Software Categories:
Category 1: Infrastructure Software (OS, firmware)
    → ใช้ supplier's qualification เท่านั้น
    → ไม่ต้อง test เพิ่มเติม

Category 3: Non-configurable Software (standard apps)
    → Basic testing: features work as described
    → Minimal additional testing

Category 4: Configurable Software (SCADA, historians)
    → Configuration-based testing
    → ตรวจว่า configuration ถูกต้อง

Category 5: Custom Software (PLC programs, custom code)
    → Full V-model testing
    → Software Design Specification
    → Unit test + Integration test + System test

PLC PROGRAMS = Category 5 → Requires Full Testing

V-MODEL:
User Requirements Spec ─────────────────→ Acceptance Test (UAT)
                         ↘             ↗
    Functional Spec ──────────────→ System Test
                       ↘         ↗
    Software Design Spec ────→ Integration Test
                         ↘ ↗
                    Unit Test
                      ↓↑
                   Coding
```

---

## Step 1189: 21 CFR Part 11 Requirements

### Electronic Records สำหรับ FDA Compliance

```
21 CFR PART 11 (FDA Electronic Records):
──────────────────────────────────────────────────────────────

ใช้กับ: Pharmaceutical, Biotech, Medical Device industries
บังคับ: Electronic records ต้องเทียบเท่า paper records

KEY REQUIREMENTS ที่ PLC Systems ต้องทำ:

1. AUDIT TRAIL:
   - บันทึกทุก parameter change: who, when, what changed
   - ห้ามลบ audit trail
   - Audit trail ต้อง searchable

// Audit Trail Entry ใน PLC:
DATA_BLOCK DB_AuditTrail
    entries: ARRAY[0..999] OF STRUCT
        dtTimestamp: DATE_AND_TIME;
        sUserID:     STRING[20];
        sParameter:  STRING[40];
        rOldValue:   REAL;
        rNewValue:   REAL;
        sReason:     STRING[80];
    END_STRUCT;
    iCount: INT;
END_DATA_BLOCK

2. ELECTRONIC SIGNATURES:
   - HMI actions ต้องการ login
   - Critical operations: require dual signature
   - Signature = UserID + Password + Reason

3. DATA INTEGRITY:
   - Checksums สำหรับ critical data
   - เวลาต้องมาจาก secure NTP server
   - No manual time changes

4. ACCESS CONTROL:
   - Role-based (Operator, Supervisor, Engineer, Admin)
   - Password policy (complexity, expiry)
   - Automatic logoff หลัง inactivity

5. COMPUTER VALIDATION:
   - IQ/OQ/PQ (Installation/Operational/Performance Qualification)
   - Documentation ทุก validation step
```

---

## Step 1190: IQ/OQ/PQ Validation

### Installation, Operational, Performance Qualification

```
IQ / OQ / PQ:
──────────────────────────────────────────────────────────────

IQ (Installation Qualification):
"ระบบถูกติดตั้งถูกต้องตาม spec หรือไม่?"

IQ Test Items:
├── Hardware matches design drawings
├── Wiring correct (continuity test)
├── Environmental conditions (temperature, humidity)
├── Utilities (power, air, network) correct
├── Software installed versions correct
└── Backup systems functional

IQ Checklist:
☐ PLC model: S7-1515-2 PN (actual matches spec)
☐ CPU FW: V2.9 (confirmed)
☐ IP address: 192.168.1.10 (matches network plan)
☐ I/O modules: 32DI, 32DO, 8AI (matches IO list)
☐ Panel internal wiring: 100% point-to-point check

──────────────────────────────────────────────────────────────

OQ (Operational Qualification):
"ระบบทำงานตาม functional spec หรือไม่?"
(ทดสอบทุก function ตาม spec - ยังไม่ใช้ actual product)

OQ Test Items:
├── ทุก PLC input function correctly
├── ทุก PLC output function correctly
├── ทุก alarm triggers and annunciates correctly
├── Interlock prevents unsafe operation
├── HMI displays correct information
└── All setpoints adjustable within range

OQ Example:
TC-OQ-007: Low Level Alarm
Steps:
  1. Set Level_Low_Setpoint = 20% (ผ่าน HMI)
  2. ลด simulated level to 19%
Expected:
  - Alarm triggers within 5 seconds
  - HMI shows: "Low Level - Tank T301"
  - Alarm bell sounds
  - PLC output xAlarmLight = TRUE

──────────────────────────────────────────────────────────────

PQ (Performance Qualification):
"ระบบ perform ตาม production requirements หรือไม่?"
(ทดสอบกับ actual product, actual conditions)

PQ Test Items:
├── Throughput: 100 bottles/min ± 5%
├── Fill accuracy: 500mL ± 5mL (3 consecutive batches)
├── Changeover time: < 30 minutes
└── Uptime: > 99% in 72-hour run

PQ Report:
Test: 3 × 8-hour production runs
Product: 500mL water bottles
Results:
  Run 1: 97.3 bottles/min, Avg fill: 499.8mL, ±3.2mL
  Run 2: 98.1 bottles/min, Avg fill: 500.1mL, ±2.9mL
  Run 3: 97.8 bottles/min, Avg fill: 499.6mL, ±3.5mL
  Uptime: 99.4% (2 minutes downtime for jam clearance)
Conclusion: PASS - Meets all PQ criteria
```

---

## Step 1191: Test Plan Document

### เขียน Test Plan สมบูรณ์

```
TEST PLAN DOCUMENT STRUCTURE:
══════════════════════════════════════════════════════════════

1. SCOPE AND OBJECTIVES
   - What systems are being tested
   - Test objectives
   - Acceptance criteria

2. TEST APPROACH
   - Test levels (FAT, SAT, IQ/OQ/PQ)
   - Test methods (manual, automated, simulation)
   - Test tools (PLCSIM, Python framework)

3. TEST SCHEDULE
   - FAT dates, SAT dates
   - Resource requirements (testers, operators)

4. RISK ASSESSMENT
   - High risk areas (safety systems, critical interlocks)
   - Mitigation strategies

5. TEST CASES (annexures)
   - TC-001 through TC-xxx
   - Each with steps, expected results, space for actual results

6. DEFECT MANAGEMENT
   - Severity levels (Critical, Major, Minor)
   - Resolution process
   - Acceptable: no Critical, no Major defects at SAT

7. SIGN-OFF CRITERIA
   - All tests executed
   - All defects resolved or accepted
   - Stakeholder signatures

SEVERITY DEFINITIONS:
Critical: Safety hazard, system crash, data loss → cannot go live
Major:    Function not working, workaround needed → must fix before go-live
Minor:    Cosmetic, non-critical function → fix in next version
```

---

## Step 1192: Defect Tracking

### ติดตาม Bug และ Issue

```
DEFECT TRACKING PROCESS:
──────────────────────────────────────────────────────────────

Defect Life Cycle:
New → Assigned → In Progress → Fixed → Retest → Closed
                     ↓
                  Rejected (ไม่ใช่ defect) → Closed

DEFECT REPORT FORMAT:
──────────────────────────────────────────────────────────────

DEFECT-007: Fill Valve Doesn't Close After Timeout
─────────────────────────────────────────────────
ID:          DEFECT-007
Date Found:  2024-03-10
Found By:    Somchai Tester
Test Case:   TC-FILL-007 (Fill Timeout)
Severity:    Critical
Status:      Fixed (2024-03-12)

Description:
When fill timeout (30 sec) triggers, xFillValve should
close immediately. During testing, fill valve remained
open for additional 5 seconds after timeout.

Steps to Reproduce:
1. Start fill cycle
2. Block flow meter signal
3. Wait 30 seconds
Expected: xFillValve = FALSE at 30 seconds
Actual:   xFillValve = FALSE at 35 seconds (5 sec delay)

Root Cause:
In FB_FillControl, the timeout check was in wrong scan
position. Timer Q was TRUE but output wasn't updated
until next scan of the network.

Fix Applied:
Moved timeout check to Network 1 (before other networks).
Now closes valve in same scan as timeout triggers.

Retest Result:
PASS - Valve closes within 100ms of timeout

Fixed By: Wichai Developer
Verified By: Somchai Tester
Closed Date: 2024-03-12
```

---

## Step 1193: Automated Testing Pipeline

### CI/CD สำหรับ PLC Tests

```
AUTOMATED TESTING IN CI/CD:
──────────────────────────────────────────────────────────────

Trigger: git push ไป develop
CI/CD ทำอัตโนมัติ:

1. UNIT TESTS (2-5 min):
   - Import PLC XML ไป PLCSIM
   - Run FB unit tests (Python + snap7)
   - Report: pass/fail per test case
   
2. INTEGRATION TESTS (5-10 min):
   - Load full project ไป PLCSIM
   - Connect Factory I/O (process simulation)
   - Run integration test scenarios
   
3. REGRESSION TESTS (10-20 min):
   - Full regression suite (all previous tests)
   - Performance tests (cycle time, response time)

4. REPORT:
   - Email to team: test results
   - GitLab: green checkmark or red X on MR
   - Metrics: test coverage, pass rate trend

// Python CI Test Runner:
#!/usr/bin/env python3
import unittest
import sys
from test_filling_unit import FillingUnitTests
from test_filling_integration import FillingIntegrationTests
from test_filling_regression import FillingRegressionTests

# Run all suites
loader = unittest.TestLoader()
suite = unittest.TestSuite()

suite.addTests(loader.loadTestsFromTestCase(FillingUnitTests))
suite.addTests(loader.loadTestsFromTestCase(FillingIntegrationTests))
suite.addTests(loader.loadTestsFromTestCase(FillingRegressionTests))

runner = unittest.TextTestRunner(verbosity=2)
result = runner.run(suite)

# Exit with error code if any failures
sys.exit(0 if result.wasSuccessful() else 1)
```

---

## Step 1194: Safety Function Testing

### ทดสอบ Safety Functions

```
SAFETY FUNCTION TESTING:
──────────────────────────────────────────────────────────────

Safety functions ต้อง test อย่างระมัดระวัง
ไม่สามารถใช้ production machine สำหรับ safety testing ทั้งหมด

SAFETY TEST TYPES:
1. Proof Test (Periodic): ทดสอบ safety function ทุก T_proof ปี
2. Diagnostic Coverage Test: ตรวจว่า diagnostic ตรวจ failure ได้
3. Common Cause Failure Test: ทดสอบ redundancy

PROOF TEST PROCEDURE (ตัวอย่าง: E-stop circuit):
──────────────────────────────────────────────────────────────
TC-SAFETY-001: E-Stop Zone 1 Proof Test
─────────────────────────────────────────
Frequency: Annual (หรือตาม SIL certificate)
Required By: SIS Engineer + Operations Supervisor

Preconditions:
  - Machine in STOPPED state
  - Safety test permit issued (PTW)
  - Personnel clear of machine

Steps:
  1. Record: E-stop circuit status = Normal
  2. Press E-stop Z1 manually
  3. Observe: Output module DI point E-Stop_Z1 = 0
  4. Observe: Safety relay K301 de-energizes
  5. Observe: All motor contactors open
  6. Observe: Alarm: "E-Stop Zone 1 Activated"
  7. Reset E-stop
  8. Reset safety relay
  9. Verify system returns to IDLE (no faults)

Expected Results:
  Step 3: DI E-Stop_Z1 = 0 (within 50ms)
  Step 4: K301 drops out (within 100ms)
  Step 5: All motors stop (within 200ms)
  Step 6: HMI alarm triggered
  Total response time: < 350ms (SIL requirement: < 500ms)

Actual Results: [Fill in during test]
Pass/Fail: [P/F]
Tested By: (sign) Date:
Witnessed By: (sign) Date:

PROOF TEST RECORDS:
เก็บ 5 ปีอย่างน้อย (pharma: ตลอดอายุระบบ)
```

---

## Step 1195: Performance Testing

### ทดสอบ Performance ของระบบ

```
PERFORMANCE TESTING:
──────────────────────────────────────────────────────────────

ตัวชี้วัด Performance สำหรับ PLC:
1. Scan Time: เวลาที่ PLC ใช้ run OB1 หนึ่งรอบ
2. Response Time: เวลาตั้งแต่ input เปลี่ยน → output เปลี่ยน
3. Cycle Time: เวลา 1 production cycle ครบ
4. Throughput: จำนวน products ต่อหน่วยเวลา

SCAN TIME MEASUREMENT (TIA Portal):
// ดู scan time ใน TIA Portal:
// Online → PLC → Diagnostics → Cycle Time
//   Current: 8.3 ms
//   Max:     12.1 ms
//   Min:      6.8 ms

// Programmatic scan time measurement:
DATA_BLOCK DB_Performance
    tScanTime_ms:   REAL;       // Current scan time (ms)
    tScanMax_ms:    REAL;       // Maximum scan time (ms)
    tCycleTime_s:   REAL;       // Production cycle time (sec)
    rThroughput:    REAL;       // Products per minute
    iCycleCount:    DINT;       // Total cycles since start
END_DATA_BLOCK

RESPONSE TIME TEST:
──────────────────────────────────────────────────────────────
Equipment: Logic Analyzer or PLC's own time measurement

TC-PERF-001: E-stop Response Time
─────────────────────────────────
Method:
  1. Connect oscilloscope to E-stop input (I/O)
  2. Connect oscilloscope to motor enable output
  3. Trigger E-stop
  4. Measure: time from E-stop input = 0 → motor output = 0

Specification: < 100ms (specified in safety analysis)
Result:       32ms → PASS

CYCLE TIME TEST:
TC-PERF-002: Bottle Fill Cycle Time
─────────────────────────────────
Steps:
  1. Reset DB_Performance.iCycleCount
  2. Run machine for 60 minutes
  3. Record: iCycleCount / 60 minutes = throughput

Specification: ≥ 95 bottles/minute
Result:       97.3 bottles/minute → PASS
```

---

## Step 1196: Documentation สำหรับ Validation

### เอกสาร Validation Package

```
VALIDATION DOCUMENTATION PACKAGE:
──────────────────────────────────────────────────────────────

สำหรับ Pharmaceutical/Food Grade Systems:

1. VALIDATION MASTER PLAN (VMP)
   - ภาพรวมของ validation approach
   - ระบุ systems ที่ต้อง validate
   - Timeline และ responsibilities

2. USER REQUIREMENTS SPECIFICATION (URS)
   - What the system must do (user language)
   - Written before design
   - Basis for all testing

3. FUNCTIONAL SPECIFICATION (FS)
   - How the system will meet URS
   - Written by system integrator
   - Basis for OQ testing

4. HARDWARE DESIGN SPECIFICATION (HDS)
   - Panel drawings, I/O list, network design
   - Bill of materials

5. SOFTWARE DESIGN SPECIFICATION (SDS)
   - PLC code structure, FBs, DB
   - State machine diagrams
   - Alarm list

6. IQ PROTOCOL + REPORT
   - Installation test cases + results

7. OQ PROTOCOL + REPORT
   - Functional test cases + results

8. PQ PROTOCOL + REPORT
   - Performance test cases + results

9. TRACEABILITY MATRIX
   - Link: URS requirement → FS section → Test case

TRACEABILITY MATRIX EXAMPLE:
URS Req | FS Section | Test Case | Test Result
URS-1.1 | FS-3.2.1   | TC-OQ-001 | PASS
URS-1.2 | FS-3.2.2   | TC-OQ-002 | PASS
URS-1.3 | FS-3.3.1   | TC-OQ-003 | PASS
URS-2.1 | FS-4.1.1   | TC-OQ-010 | PASS
```

---

## Step 1197: Test Tools และ Equipment

### เครื่องมือสำหรับ Testing

```
TEST TOOLS:
══════════════════════════════════════════════════════════════

SOFTWARE TOOLS:
├── S7-PLCSIM Advanced (Siemens)
│   - Simulate S7-1500 ตาม real-time
│   - API สำหรับ automated testing
│   - เชื่อมต่อกับ Factory I/O
│
├── Logix Designer Emulate 5000 (Allen-Bradley)
│   - Emulate ControlLogix
│   - เชื่อมต่อกับ Process Simulator
│
├── Factory I/O (realgames.pt)
│   - 3D process simulation
│   - Connects via OPC UA, Modbus, S7 protocol
│   - สำหรับ testing logic กับ realistic process
│
├── Python + snap7 (Siemens)
│   - Automated test scripts
│   - Read/write PLC data blocks
│   - Assert expected values
│
├── Python + pylogix (Allen-Bradley)
│   - EtherNet/IP connection to ControlLogix
│   - Tag read/write สำหรับ automated tests
│
└── Selenium + FactoryTalk View
    - HMI automated testing
    - Screenshot comparison

HARDWARE TEST TOOLS:
├── Logic Analyzer: ตรวจ digital signals timing
├── Oscilloscope: วัด response times อย่างแม่นยำ
├── Continuity Tester: ตรวจ wiring (IQ)
├── Multimeter: ตรวจ power supplies
├── Process Calibrator: ตรวจ AI/AO accuracy
└── Temperature Calibrator: สำหรับ RTD/thermocouple tests
```

---

## Step 1198: Test Coverage Analysis

### วิเคราะห์ Test Coverage

```
TEST COVERAGE ANALYSIS:
──────────────────────────────────────────────────────────────

ถาม: "Test ครอบคลุมแค่ไหน?"

Coverage Metrics:
1. Requirement Coverage: กี่ % ของ URS requirements มี test?
2. Code Coverage: กี่ % ของ code ถูก execute ระหว่าง test?
3. Branch Coverage: กี่ % ของ IF/CASE branches ถูก test?
4. State Coverage: กี่ % ของ states ใน state machine ถูก test?

TARGET COVERAGE:
├── Safety-critical systems: 100% branch coverage
├── Pharmaceutical (GAMP 5 Cat 5): 100% requirement coverage
└── General industrial: ≥ 80% branch coverage

COVERAGE MATRIX (ตัวอย่าง):
Requirement Coverage:
URS-1.1: Fill normal     → TC-001 ✓
URS-1.2: Fill timeout    → TC-007 ✓
URS-1.3: E-stop          → TC-002 ✓
URS-2.1: Recipe select   → TC-015 ✓
URS-2.2: Recipe change   → TC-016 ✓
URS-3.1: Alarm history   → TC-020 ✗ ← NOT COVERED!

→ ต้องสร้าง TC-020 ก่อน go-live

State Machine Coverage:
STATE_IDLE:     Tested ✓ (TC-001, TC-012)
STATE_PRESTART: Tested ✓ (TC-001)
STATE_RUNNING:  Tested ✓ (TC-001 through TC-010)
STATE_STOPPING: Tested ✓ (TC-003)
STATE_FAULT:    Tested ✓ (TC-007, TC-008)
Coverage: 5/5 = 100% ✓
```

---

## Step 1199: Post-Deployment Monitoring

### หลัง Go-Live: Monitor อย่างต่อเนื่อง

```
POST-DEPLOYMENT MONITORING:
──────────────────────────────────────────────────────────────

Go-Live Monitoring Period (typically 4 weeks):
├── Enhanced monitoring: check ทุก shift
├── Engineer on standby (first week)
├── Daily review ของ alarms + downtime
└── Weekly report ไปยัง management

KPIs to Monitor:
├── OEE (Overall Equipment Effectiveness): target > 85%
├── Alarm rate: < 5 alarms/hour (steady state)
├── Downtime: < 1% unplanned
├── False alarms: < 10% ของ total alarms
└── Rework rate: ตรวจ rejects เพิ่มขึ้นไหม?

Alarm Management:
// ถ้า alarm rate เพิ่มขึ้น → investigate:
// - False alarm? → ปรับ deadband/delay
// - Real problem? → fix root cause
// - Alarm flood? → group related alarms

Post-Deployment Test (ทุก 3 เดือน):
├── Run abbreviated FAT suite (30-50% of original)
├── Verify safety functions (proof test)
└── Check calibration ของ critical instruments

CHANGE CONTROL หลัง Go-Live:
ทุก change ต้องผ่าน formal process:
1. Change Request (CR) form
2. Impact Assessment
3. Review and Approval
4. Testing (regression minimum)
5. Deploy + Document
6. Close CR

ห้ามแก้ program โดยไม่มี CR!
```

---

## Step 1200: Lab Project - FAT for Mixing System

### Lab: ทำ Factory Acceptance Test

```
LAB: FAT FOR MIXING AND FILLING SYSTEM
══════════════════════════════════════════════════════════════

ระบบที่ทดสอบ:
- Mixing Tank with agitator, heater, 3 ingredient valves
- Filling station (2 heads)
- Conveyor with reject system

TASK 1: Create Test Plan (25%)
─────────────────────────────────────────────────────────────
1. เขียน Test Plan Document:
   - Scope: Mixing + Filling (not conveyor - separate FAT)
   - Test approach: manual + automated (PLCSIM)
   - Schedule: 2-day FAT
   
2. สร้าง Test Case matrix:
   - จาก Functional Specification ที่ให้
   - อย่างน้อย 20 test cases
   - ครอบคลุม: normal operation, faults, interlocks, alarms
   
3. Traceability Matrix:
   - Link ทุก FS requirement กับ test case

TASK 2: Execute Tests (50%)
─────────────────────────────────────────────────────────────
ทดสอบใน PLCSIM Advanced + Factory I/O:

1. Normal Operations (5 test cases):
   - Manual mix cycle
   - Automatic mix cycle with recipe
   - Fill cycle (500mL product)
   - Fill cycle (1000mL product)
   - Changeover between products

2. Fault Handling (10 test cases):
   - Agitator overload → fault state + alarm
   - Heater timeout → fault + alarm
   - Fill timeout → fault + alarm
   - Level sensor failure (wire break)
   - E-stop during mixing
   - E-stop during filling
   - Power failure (restore to safe state)
   - Communication loss (SCADA disconnects)
   - Invalid recipe (out of range value)
   - Concurrent fill heads

3. Safety Interlocks (5 test cases):
   - Fill without agitator running → blocked
   - Drain while heater on → blocked
   - Fill head opens without bottle → blocked
   - Temperature too high → pause + alarm
   - Overfill protection

TASK 3: Record and Report (25%)
─────────────────────────────────────────────────────────────
1. Fill in actual results สำหรับทุก test case
2. Log defects พบ (DEFECT form สำหรับ each failure)
3. ประเมิน: PASS, FAIL, N/A ต่อ test case
4. Coverage analysis: % requirements covered
5. สร้าง FAT Report:
   - Summary: กี่ test passed, กี่ failed
   - Critical defects list
   - Recommendation: Accept / Conditional Accept / Reject

GRADING CRITERIA:
├── Test Plan completeness: 25%
├── Test execution thoroughness: 40%
├── Defect documentation quality: 15%
└── FAT Report: 20%

DELIVERABLES:
├── Test Plan (Word/PDF)
├── Filled Test Cases (Excel)
├── Defect Log (Excel)
├── FAT Report with sign-off page
└── (Optional) Automated test script (Python)
```

---

## สรุป Part 60

| กิจกรรม | เอกสารที่ต้องใช้ |
|---------|----------------|
| FAT | Test Plan, Test Cases, FAT Report |
| SAT | SAT Protocol, Commissioning Report |
| IQ | IQ Protocol, Calibration Certs |
| OQ | OQ Protocol, Functional Test Results |
| PQ | PQ Protocol, Production Run Data |
| 21 CFR Part 11 | Audit Trail, E-Signature Policy |

---

## Navigation

- [← Part 59: Version Control for PLC Programs](./part59_version_control.md)
- [→ Part 61: Large System Design](./part61_large_system_design.md)
- [↑ สารบัญหลัก](../README.md)
