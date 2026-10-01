# Part 79: PLC Project Management (Steps 1561-1580)

## การบริหารจัดการโครงการ Automation ตั้งแต่ต้นจนจบ

---

### 🔗 Navigation
- [← Part 78: Simulation & Virtual Testing](part78_simulation_testing.md)
- [→ Part 80: Certification & Career Paths](part80_certification_paths.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: Automation Project Lifecycle

```
PLC/Automation Project มักล้มเหลวเพราะอะไร?

  Technical Issues: 25%
    - Software Bug
    - Hardware ไม่ Compatible
    
  Management Issues: 75%
    - Scope Creep (งานเพิ่มตลอดเวลา)
    - Poor Communication
    - Unclear Requirements
    - Schedule ไม่สมจริง
    - Resources ไม่พอ
    
สรุป: Engineer เก่งอย่างเดียวไม่พอ ต้องบริหารโครงการเป็นด้วย
```

---

## Step 1561: Automation Project Lifecycle

### ขั้นตอนโครงการ Automation

```
Phase 1: Conceptual Design (FEED)
  Duration: 2-8 สัปดาห์
  Output:
    - Process Description
    - Preliminary I/O Count
    - PLC/DCS Vendor Selection
    - Rough Cost Estimate (±40%)
    - Schedule (Milestone Level)
  
Phase 2: Basic Engineering
  Duration: 4-12 สัปดาห์
  Output:
    - P&ID (Process & Instrumentation Diagram)
    - Instrument Index
    - I/O List
    - FDS (Functional Design Specification)
    - Preliminary Panel Layout
    - Detailed Cost Estimate (±20%)
  
Phase 3: Detailed Engineering
  Duration: 8-20 สัปดาห์
  Output:
    - HDS (Hardware Design Specification)
    - SDS (Software Design Specification)
    - Panel Drawing (Electrical)
    - PLC Program (Coded)
    - HMI/SCADA (Designed)
    - FAT Procedure
  
Phase 4: FAT (Factory Acceptance Test)
  Duration: 1-4 สัปดาห์
  Output:
    - FAT Report
    - Punch List (Items A, B, C)
    - Approved for Shipment
  
Phase 5: Installation
  Duration: 2-8 สัปดาห์
  Activities:
    - Panel Installation
    - Cable Pulling
    - Termination
    - Field Instrument Installation
  
Phase 6: SAT / Commissioning
  Duration: 2-8 สัปดาห์
  Output:
    - SAT Report
    - Loop Check Records
    - As-Built Documents
    
Phase 7: Handover & Closeout
  Duration: 1-4 สัปดาห์
  Output:
    - O&M Manual
    - As-Built Drawings
    - Spare Parts List
    - Project Closure Report
```

---

## Step 1562: Scope of Work Document

### การเขียน Scope of Work ที่ดี

```
Scope of Work (SOW) ที่ดีต้องมี:

1. System Description
   - อธิบาย Process ที่ควบคุม
   - จำนวน Loop Control
   - จำนวน I/O (Discrete / Analog / Special)
   
2. System Boundaries
   - Included: อะไรอยู่ใน Scope
   - Excluded: อะไรไม่อยู่ใน Scope (สำคัญมาก!)
   
3. Deliverables
   - Hardware (PLC Cabinet, HMI Panel)
   - Software (PLC Program, SCADA, Database)
   - Documents (FDS, Drawings, Manual)
   - Services (Installation, Commissioning, Training)
   
4. Standards & Codes
   - IEC 61131-3 (PLC Programming)
   - IEC 61511 (Safety)
   - NEC/IEC Electrical Codes
   - Customer Standards
   
5. Interfaces
   - กับระบบอื่น (DCS, ERP, LIMS)
   - Protocol ที่ใช้

Bad SOW Example:
  "ส่ง Automation ระบบ Boiler ให้ครบ"
  → ไม่ระบุว่า Boiler มีกี่ตัว I/O มีเท่าไหร่
  → ทำให้เกิด Scope Creep ในภายหลัง

Good SOW Example:
  "จัดหาและติดตั้ง PLC Control System สำหรับ
   Steam Boiler No.1-3 (50 ton/hr each)
   Scope รวม:
   - Siemens S7-1515 × 3 ชุด
   - 24 DI, 16 DO, 8 AI per PLC
   - WinCC Advanced SCADA 1 ชุด (Server+2 Client)
   - BMS Logic ตาม NFPA 85
   - Training 8 ชั่วโมงต่อ Operator
   - Commissioning 2 สัปดาห์
   Exclude: Field Instrumentation, Cable Installation"
```

---

## Step 1563: FDS, HDS, SDS Documents

### เอกสาร Engineering ที่สำคัญ

**FDS (Functional Design Specification):**

```
FDS ตอบคำถาม "ระบบต้องทำอะไร?"

โครงสร้าง FDS:
  1. Overview และ P&ID Reference
  2. Operating Modes
     - Normal Auto
     - Manual Override
     - Emergency
  3. Startup Sequence (Step-by-step)
  4. Normal Operation Logic
  5. Shutdown Sequence
  6. Interlocks (ตารางทุก Interlock)
  7. Alarms (ตาราง Alarm Priority, Trip Points)
  8. Permissives (เงื่อนไขก่อน Start)
  
FDS Interlock Table ตัวอย่าง:
  ┌────┬───────────────────┬──────────┬────────────────┐
  │ IL#│ Description       │ Action   │ Reset          │
  ├────┼───────────────────┼──────────┼────────────────┤
  │ IL1│ High-High Level   │ Close FV │ Manual Reset   │
  │ IL2│ Low-Low Level     │ Stop Pump│ Auto Reset     │
  │ IL3│ High Temp         │ Open CV  │ Auto Reset     │
  │ IL4│ Motor Overload    │ Trip + A │ Manual Reset   │
  └────┴───────────────────┴──────────┴────────────────┘
```

**HDS (Hardware Design Specification):**

```
HDS ตอบคำถาม "ใช้ Hardware อะไร?"

โครงสร้าง HDS:
  1. PLC Platform Selection
     - CPU Model และ Firmware Version
     - I/O Modules (ทุก Card พร้อม Part Number)
     - Power Supply Sizing
     - Redundancy Level
     
  2. Enclosure Design
     - NEMA/IP Rating
     - Size (H×W×D)
     - Power Distribution (MCB, Fuse)
     - Cooling (Fan, A/C)
     
  3. Network Architecture
     - Topology (Star, Ring)
     - Switches และ Cable Type
     
  4. UPS/Battery Backup
     - Capacity (kVA, Minutes)
```

**SDS (Software Design Specification):**

```
SDS ตอบคำถาม "Program ออกแบบอย่างไร?"

โครงสร้าง SDS:
  1. Programming Structure
     - Program Organization Units (POUs)
     - Naming Convention
     - Comment Requirements
     
  2. Memory Map
     - Global Variables Layout
     - Data Blocks
     - I/O Address Mapping
     
  3. Function Blocks (FBs)
     - FB List (ชื่อ, Input, Output, Description)
     - State Machine Diagrams
     
  4. HMI Tag List
     - Tag Name Mapping (PLC ↔ HMI)
     - Screen List
     
  5. Alarm Philosophy
     - Alarm Categories
     - Priority Matrix
```

---

## Step 1564: Schedule Planning

### การวางแผน Project Schedule

```
Schedule ใช้ Microsoft Project หรือ Primavera P6

WBS (Work Breakdown Structure):
  1.0 Design Phase
    1.1 P&ID Review
    1.2 I/O List
    1.3 FDS Writing
    1.4 FDS Review + Approval
    1.5 HDS Writing
    1.6 SDS Writing
    
  2.0 Procurement
    2.1 BOM Preparation
    2.2 Vendor Quotation
    2.3 PO Issuance
    2.4 Hardware Delivery
    
  3.0 Engineering/Programming
    3.1 Panel Fabrication
    3.2 PLC Programming
    3.3 HMI Development
    3.4 Testing (Internal)
    
  4.0 FAT
    4.1 FAT Preparation
    4.2 FAT Execution
    4.3 Punch List Resolution
    
  5.0 Installation
    5.1 Equipment Installation
    5.2 Cabling
    5.3 Termination
    
  6.0 Commissioning
    6.1 Power-up Check
    6.2 Loop Check (All I/O)
    6.3 Functional Test
    6.4 SAT Execution
    
  7.0 Closeout
    7.1 As-Built Drawing
    7.2 O&M Manual
    7.3 Training
    7.4 Project Closure
```

---

## Step 1565: Resource Estimation

### การประเมิน Resources

```
Engineering Hours Estimation:
  
  PLC Programming: 1.5-3 hours per I/O point
    เช่น 200 I/O × 2 hrs = 400 man-hours
    
  HMI Development: 8-20 hours per screen
    เช่น 30 screens × 15 hrs = 450 man-hours
    
  FAT: 10-15% of total program hours
    เช่น (400+450) × 12% = 102 man-hours
    
  Commissioning: 2-3 hours per I/O point
    เช่น 200 I/O × 2.5 hrs = 500 man-hours

Team สำหรับ Medium-size Project (200 I/O):
  1 Project Manager (PM)
  2 Control Systems Engineers
  1 Electrical Engineer
  1 Commissioning Engineer
  Duration: 6 เดือน

Cost Estimation (Southeast Asia Market):
  Engineering Labor: 2,000-5,000 THB/hr
  Hardware: ตาม BOM
  Travel/Living: 10-15% of Labor
  Contingency: 10-15%
```

---

## Step 1566: Change Management

### การบริหาร Change Request

```
Change Request Process:

  Customer Request Change
          ↓
  Engineer ประเมิน Impact
  (Cost, Schedule, Technical)
          ↓
  Submit Change Request (CR) Form
  CR-001: Description, Impact, Cost, Days
          ↓
  Customer Review
          ↓
  Approve / Reject / Negotiate
          ↓
  Implement Change (ถ้า Approve)
          ↓
  Update Documents
  
Change Request Form ตัวอย่าง:
  ┌─────────────────────────────────────────────────┐
  │ CHANGE REQUEST                                  │
  │ CR No: CR-004                    Date: 2025-11-01│
  │ Project: Boiler Control System                  │
  │ Requested by: Plant Engineering                 │
  │                                                 │
  │ Description:                                    │
  │   เพิ่ม Remote Monitoring ผ่าน 4G Router        │
  │   สำหรับ On-call Engineer                       │
  │                                                 │
  │ Impact Assessment:                              │
  │   Hardware: 4G Industrial Router = THB 25,000   │
  │   Software: VPN Configuration = 8 hrs           │
  │   Schedule: +3 days                             │
  │   Total Cost: THB 65,000                        │
  │                                                 │
  │ Customer Approval: ____________ Date: _______   │
  └─────────────────────────────────────────────────┘
```

---

## Step 1567: FAT (Factory Acceptance Test)

### การเตรียมและดำเนินการ FAT

```
FAT คืออะไร:
  ทดสอบระบบที่ Office/Workshop ก่อน Delivery
  ใช้ Simulation สำหรับ Process (ถ้า Plant ไม่มา)
  Customer (หรือ Inspector) ต้องเข้าร่วม

FAT Preparation:
  1. FAT Procedure Document (ใครทำอะไร ขั้นตอน)
  2. Test Sheets (แต่ละ Test มี Pass/Fail Criteria)
  3. Simulation Setup (ถ้าใช้ I/O Simulator)
  4. FAT Environment:
     - Panel พร้อม
     - HMI พร้อม
     - Test Equipment (Multimeter, Loop Calibrator)
  5. Customer เดินทางมา

FAT Agenda (2 วัน ตัวอย่าง):
  
  Day 1: Hardware + Basic Test
  08:00 - Introductions, Safety Briefing
  09:00 - Hardware Walk-around Inspection
  10:00 - Power-up Check
  11:00 - I/O Check (DI, DO, AI, AO)
  13:00 - Communication Test (PROFINET, HMI)
  14:00 - Normal Operation Test (Startup Sequence)
  16:00 - Day 1 Punch List Review
  
  Day 2: Functional + Alarm Test  
  08:00 - Interlock Test (IL-001 to IL-025)
  10:00 - Alarm Test (All Priority 1, Sample Priority 2)
  12:00 - HMI Functionality Test
  13:00 - Data Logging / Historian Test
  14:00 - Performance Test (Response Time)
  15:00 - Final Punch List Walkthrough
  16:00 - FAT Sign-off or Extension Decision

FAT Punch List Categories:
  Category A (Critical): ต้องแก้ก่อน Delivery
  Category B (Major): ต้องแก้ก่อน SAT
  Category C (Minor): แก้ได้ระหว่าง Warranty

FAT Sign-off Criteria:
  - Category A Items: Zero (0)
  - Category B Items: < 5
  - Category C Items: ไม่จำกัด
```

---

## Step 1568: SAT (Site Acceptance Test)

### การดำเนินการ SAT ที่ Site

```
SAT คืออะไร:
  ทดสอบหลัง Installation เสร็จที่ Site
  ทดสอบกับ Equipment จริง ใน Environment จริง
  
Loop Check (ก่อน SAT):
  ตรวจสอบ Wiring ทุก I/O Point
  
  Loop Check Procedure (แต่ละ Loop):
  1. Verify Tag Number ที่ Field ตรงกับ Drawing
  2. Verify Wire Termination ถูก Terminal
  3. Inject Test Signal ที่ Field
     - AI: 4mA (0%), 12mA (50%), 20mA (100%)
     - DI: Short/Open
  4. Verify PLC รับค่าถูกต้อง (ที่ HMI หรือ PLC Screen)
  5. Test Output สั่งจาก PLC
     - AO: สั่ง 0%, 50%, 100% → วัด mA ที่ Actuator
     - DO: Energize/De-energize → ดู Field
  6. Sign Off ใน Loop Check Sheet
  
Loop Check Sheet ตัวอย่าง:
  ┌────────┬──────────┬──────┬──────┬────────────┐
  │ Tag    │ Type     │4mA=0?│12=50?│20=100?     │
  ├────────┼──────────┼──────┼──────┼────────────┤
  │ FT-101 │ AI 4-20mA│ ✓    │ ✓    │ ✓  Sign:__ │
  │ LT-201 │ AI 4-20mA│ ✓    │ ✓    │ ✓  Sign:__ │
  │ FV-101 │ AO 4-20mA│ 4mA✓ │12mA✓ │20mA✓Sign:__│
  └────────┴──────────┴──────┴──────┴────────────┘

SAT vs FAT Difference:
  FAT: ทดสอบ Hardware + Software ที่ Workshop
  SAT: ทดสอบ ทั้งระบบรวม Field Devices ที่ Site
  SAT มี "Hot Tests" (ใช้ Process จริง)
  FAT มี "Cold Tests" (ใช้ Simulation)
```

---

## Step 1569: Punch List Management

### การบริหาร Punch List

```
Punch List Database:
  ใช้ Excel หรือ Commissioning Software
  
  Columns:
  No. | Area | Description | Category | Assigned | Due Date | Status | Closed Date
  
  1 | Boiler | HMI Tag FT-101 wrong units | B | SW-Eng | 2025-12-10 | Open | -
  2 | Boiler | Cable gland loose TY-302 | C | Field | 2025-12-12 | Closed | 2025-12-11
  3 | Util | DI-205 not responding | A | SW+HW | 2025-12-08 | WIP | -
  
Priority Actions:
  Category A: Resolve before shipment/startup
    → Daily Status Meeting
    → Dedicated Resource
    
  Category B: Resolve before SAT completion
    → Track weekly
    
  Category C: Resolve during Warranty
    → Track monthly
    
Weekly Punch List Report:
  Total Items: 45
  A Items: 2 (must close before FAT complete)
  B Items: 15 (target: all closed by Jan 15)
  C Items: 28 (warranty period)
  Closed this week: 8
  % Complete: 60%
```

---

## Step 1570: Project Closeout Documentation

### เอกสารปิดโครงการ

```
Closeout Package ประกอบด้วย:

1. As-Built Documents
   - As-Built P&ID (redlined revision)
   - As-Built Panel Drawing
   - As-Built Loop Drawing
   - As-Built Wiring Diagram
   - Final I/O List (actual Field Tags)
   
2. Software Archive
   - Final PLC Program (TIA Portal Archive)
   - Final HMI Project (WinCC Archive)
   - Program Revision History
   - PLC Hardware Configuration File
   
3. Test Records
   - FAT Report + Signed Punch List
   - SAT Report + Loop Check Sheets
   - Calibration Records (with Certificates)
   
4. Manuals
   - Operation Manual (Operator Guide)
   - Maintenance Manual (Engineer Guide)
   - Spare Parts List (with Part Numbers)
   
5. Training Records
   - Training Attendance Sheet
   - Training Agenda
   - Assessment Results (ถ้ามี)
   
6. Project Closure Report
   - Final Cost vs Budget
   - Final Schedule vs Plan
   - Lessons Learned
   - Warranty Information
```

---

## Step 1571-1580: Advanced Project Topics

### Step 1571: Risk Management

```
Technical Risks ใน Automation Project:
  
  1. I/O Count Increase (Scope Creep)
     Probability: High | Impact: High
     Mitigation: 15-20% Spare I/O ใน Design
     
  2. Network Communication Issues
     Probability: Medium | Impact: High
     Mitigation: Prototype Test ก่อน
     
  3. Third-party Equipment Interface
     Probability: High | Impact: Medium
     Mitigation: FAT ร่วมกับ Third-party Vendor
     
  4. Software Bug พบ ระหว่าง Commissioning
     Probability: High | Impact: Medium
     Mitigation: Virtual Commissioning ก่อน
     
  5. Delivery Delay (Hardware)
     Probability: Medium | Impact: High
     Mitigation: Order Hardware ก่อน ≥ 16 สัปดาห์

Risk Register Format:
  ID | Risk Description | Probability | Impact | Risk Score | Mitigation | Owner
```

### Step 1572: Vendor Management

```
Vendor Selection Criteria:
  
  Technical:
    - Platform Compatibility กับ Customer Standard
    - Local Support ในพื้นที่
    - Reference Projects ใกล้เคียง
    
  Commercial:
    - ราคา (แต่ไม่ใช่ปัจจัยเดียว)
    - Warranty Terms (ปกติ 12 เดือน)
    - Lead Time (Delivery Time)
    
  Service:
    - Local Spare Parts Availability
    - 24/7 Support (สำหรับ Critical Plant)
    - Training ที่ให้
    
ตัวอย่างระบบ Scoring:
  Technical:  40 คะแนน
  Commercial: 30 คะแนน
  Service:    20 คะแนน
  References: 10 คะแนน
  Total:     100 คะแนน
```

### Step 1573: Earned Value Management (EVM)

```
EVM ติดตาม Project Progress อย่างเป็นวิทยาศาสตร์:

Key Metrics:
  PV (Planned Value): งานที่ควรทำเสร็จแล้ว
  EV (Earned Value): งานที่ทำเสร็จจริง (ตามมูลค่า)
  AC (Actual Cost): ค่าใช้จ่ายจริง
  
Indicators:
  SPI = EV/PV (Schedule Performance Index)
    SPI > 1.0 = งานเร็วกว่าแผน (ดี)
    SPI < 1.0 = งานช้ากว่าแผน (ต้องแก้)
    
  CPI = EV/AC (Cost Performance Index)
    CPI > 1.0 = ประหยัดกว่าแผน (ดี)
    CPI < 1.0 = เกินงบ (ต้องแก้)
    
ตัวอย่าง:
  Month 3 ของโครงการ 6 เดือน
  Budget (BAC): 3,000,000 THB
  PV: 1,500,000 THB (50% ควรทำเสร็จ)
  EV: 1,200,000 THB (40% ทำเสร็จจริง)
  AC: 1,350,000 THB (ค่าใช้จ่ายจริง)
  
  SPI = 1,200,000 / 1,500,000 = 0.80 (ช้า 20%)
  CPI = 1,200,000 / 1,350,000 = 0.89 (เกิน 11%)
  
  → ต้อง Corrective Action ทันที!
```

---

## Lab Project 79: สร้าง Project Plan สำหรับ WTP Automation

### โจทย์: เป็น Project Manager สำหรับ Water Treatment Plant Automation Project

**Project Scope:**
- WTP ขนาด 10,000 m³/day
- PLC: Siemens S7-1515
- SCADA: WinCC Advanced
- I/O: 120 DI, 80 DO, 40 AI, 20 AO = 260 I/O total
- Location: นครสวรรค์, ไทย
- Budget: 5,000,000 THB

**งานที่ต้องทำ:**

1. **สร้าง WBS** (Work Breakdown Structure) ใน Excel
   - ≥ 4 Level (Phase → Task → Subtask → Activity)
   
2. **สร้าง Schedule** (Microsoft Project หรือ Excel Gantt)
   - Duration เหมาะสม
   - Critical Path ชัดเจน
   - Milestones: FDS Approved, FAT, SAT, Handover
   
3. **ประเมิน Resources**
   - Engineering Hours (แต่ละ Role)
   - Hardware Cost (ประมาณ)
   - Total Cost (ต้องไม่เกิน Budget)
   
4. **เขียน Risk Register**
   - ≥ 10 Risks
   - แต่ละ Risk มี Mitigation Plan
   
5. **สร้าง FAT Checklist**
   - ≥ 30 Test Items
   - Pass/Fail Criteria ชัดเจน

**เกณฑ์ประเมิน:**
- WBS ครบถ้วนและมี Logic ถูกต้อง (25 คะแนน)
- Schedule สมจริงและมี Dependencies (25 คะแนน)
- Cost ไม่เกิน Budget และมี Breakdown (25 คะแนน)
- Risk Register ครบและ Mitigation ดี (15 คะแนน)
- FAT Checklist ใช้ได้จริง (10 คะแนน)

---

## สรุป Part 79

ใน Part นี้เราได้เรียนรู้:
- Automation Project Lifecycle (7 Phases)
- เอกสาร FDS, HDS, SDS
- Scope of Work ที่ดี
- Schedule Planning และ WBS
- Resource Estimation
- Change Management
- FAT และ SAT Procedures
- Punch List Management
- Project Closeout

**ทักษะสำคัญ:**
- Project Planning & Scheduling
- Technical Documentation Writing
- Test Execution & Management
- Earned Value Management

---

### 🔗 Navigation
- [← Part 78: Simulation & Virtual Testing](part78_simulation_testing.md)
- [→ Part 80: Certification & Career Paths](part80_certification_paths.md)
- [📚 สารบัญหลัก](../README.md)
