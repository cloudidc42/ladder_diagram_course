# Part 20: Debugging และ Troubleshooting PLC - จากปัญหาสู่การแก้ไข

## Steps 371-390: ทักษะที่แยกช่างมือใหม่ออกจากมืออาชีพ

**[← Part 19: OMRON](part19_omron.md) | [กลับสู่สารบัญ](../README.md)**

---

## บทนำ: ความสำคัญของ Debugging

ทักษะการ Debug และ Troubleshoot คือสิ่งที่ทำให้ Engineer PLC ที่ดีต่างจากคนที่แค่เขียนโปรแกรมได้ ในโรงงานจริง ปัญหาส่วนใหญ่ไม่ได้เกิดจากโปรแกรม PLC แต่เกิดจาก Hardware, Wiring, Sensor, และ Process ที่เปลี่ยนแปลง

**สถิติจากประสบการณ์จริง:**
- ~60% ปัญหาเกิดจาก Hardware/Wiring
- ~20% ปัญหาเกิดจาก Sensor/Actuator
- ~15% ปัญหาเกิดจาก Process/Mechanical
- ~5% ปัญหาเกิดจาก Program PLC จริงๆ

---

## Step 371: Online Monitoring Techniques

### หลักการ Online Monitoring

```
Online Monitoring คือการดูสถานะ PLC ขณะทำงานจริง
เป็น Tool แรกที่ต้องใช้เมื่อมีปัญหา

ขั้นตอนพื้นฐาน:
1. Connect PC ไปยัง PLC (USB/Ethernet)
2. เปิด Online Mode
3. สังเกตสี/สถานะของ Contact/Coil
4. อ่านค่า Timer/Counter/Data Register
5. ระบุจุดที่ Logic ไม่ทำงานตามที่คาดหวัง

Monitoring Levels:
Level 1: ดู Output ปลายสาย (Input/Output สีอะไร)
Level 2: ดู Internal Logic (M Bit, W Bit สถานะอย่างไร)
Level 3: ดู Data Register (ค่าตัวเลขถูกต้องหรือไม่)
Level 4: ดู Communication Status (Data รับ/ส่งถูกต้องหรือไม่)
```

### สิ่งที่ต้องสังเกตเมื่อ Monitor

```
สำหรับ Siemens TIA Portal:
┌────────────────────────────────────────────────────────┐
│ Online Indicators ที่ต้องสังเกต:                      │
├────────────────────────────────────────────────────────┤
│ ● สีเขียวที่ Project Tree = CPU RUN, ไม่มี Error      │
│ ● สีส้มที่ Project Tree = CPU RUN, มี Warning         │
│ ● สีแดงที่ Project Tree = CPU FAULT                   │
│ ─ เส้นสีน้ำเงิน/เหลืองใน LAD = Power Flow TRUE        │
│ ─ เส้นสีขาวใน LAD = Power Flow FALSE                  │
│ ค่าตัวเลขใต้ Address = ค่าปัจจุบัน                    │
└────────────────────────────────────────────────────────┘

สำหรับ Allen-Bradley Studio 5000:
- Rung สีเขียว = TRUE
- Rung สีขาว/เทา = FALSE
- ค่าใน [] ใต้ Tag = Current Value

สำหรับ Mitsubishi GX Works3:
- Contact สีน้ำเงิน = TRUE/ON
- Coil สีน้ำเงิน = TRUE/ON
- ค่าแสดงใน Monitor Window

สำหรับ OMRON CX-Programmer:
- Contact/Coil สีฟ้า = TRUE
- ค่าแสดงในช่อง []
```

### ตัวอย่าง Debug Scenario: Motor ไม่สตาร์ท

```
ปัญหา: กด Start แล้วมอเตอร์ไม่ทำงาน

Step 1: ตรวจสอบ Output ก่อน
  Online Mode → ดู Output Coil (Q0.0/100.00/Y0)
  ถ้า Output Coil = TRUE แต่มอเตอร์ไม่วิ่ง
  → ปัญหาที่ Hardware/Wiring ไม่ใช่ Program

  ถ้า Output Coil = FALSE
  → ปัญหาที่ Program Logic

Step 2: ถ้าปัญหาที่ Program
  ดู Rung ที่ Output:
  
  Start_PB    E_Stop     Stop_PB    Motor_Run
  ──┤├──┬─────┤/├──────────┤/├──────────( )── ← ดูที่นี่
        │
  Motor_Run
  ──┤├──┘
  
  ตรวจสอบ:
  a) Start_PB (TRUE?) ← กดปุ่มแล้วเป็น TRUE หรือไม่?
  b) E_Stop (FALSE?) ← NC Contact ควรเป็น FALSE ปกติ
  c) Stop_PB (FALSE?) ← NC Contact ควรเป็น FALSE ปกติ
  
  ถ้า Start_PB = FALSE แม้กดปุ่มแล้ว
  → ปัญหา Input Wiring หรือ Input Card

Step 3: ตรวจสอบ Input Hardware
  ดู Input LED ที่ PLC หน้าตู้:
  - กด Start PB → LED ติดหรือไม่?
  - ถ้าไม่ติด → ตรวจสาย PB, Voltage Supply
  - ถ้าติด แต่ Software ไม่เห็น → Input Card อาจเสีย
```

---

## Step 372: Watch Tables ในแต่ละ Platform

### Watch Table / Device Monitor

```
Watch Table คือ List ของ Variable ที่ต้องการ Monitor อย่างต่อเนื่อง
มีประโยชน์กว่า Online LAD เพราะ:
- ดูหลาย Variable พร้อมกัน
- ดู Value ที่เปลี่ยนเร็ว
- แก้ไขค่าได้ง่าย
- บันทึกเป็น File ได้

Siemens TIA Portal - Watch Table:
Project Tree → Monitoring & Forcing Tables → Watch table
┌──────────────────────────────────────────────────────┐
│ Watch Table                                          │
├──────────────┬──────────┬───────────┬───────────────┤
│ Address      │ Display  │ Monitor   │ Modify        │
├──────────────┼──────────┼───────────┼───────────────┤
│ "Start_PB"   │ Bool     │ FALSE     │ [  ]          │
│ "Motor_Run"  │ Bool     │ TRUE      │ [TRUE] Apply  │
│ "Run_Time".ET│ Time     │ T#3m_22s  │ [  ]          │
│ "Speed_SP"   │ DInt     │ 1500      │ [1500] Apply  │
│ "Temperature"│ Real     │ 73.5      │ [  ]          │
└──────────────┴──────────┴───────────┴───────────────┘

Allen-Bradley - Tag Monitor:
Controller Tags → Monitor Tags
Filter Tags ด้วย Search Box

Mitsubishi - Device Monitor:
Online → Device Monitor
ใส่ Device Address: D100, M0, T0 ฯลฯ

OMRON - Watch Window:
View → Watch (Ctrl+W)
```

---

## Step 373: Force Values - การบังคับค่า

### ทำความเข้าใจกับ Force

```
Force คือการบังคับค่า Input/Output/Internal
โดยไม่สนใจสถานะ Hardware จริง

⚠️ ⚠️ ⚠️ คำเตือนสำคัญ ⚠️ ⚠️ ⚠️

Force I/O มีความเสี่ยงสูงมาก:
1. อาจทำให้เครื่องจักรเคลื่อนที่โดยไม่คาดคิด
2. Safety Circuit อาจถูก Bypass
3. ลืมเอา Force ออกหลัง Debug → อันตราย

ก่อน Force ต้องทำ:
✓ แจ้งผู้ที่เกี่ยวข้องทุกคน
✓ ล็อก Machine ก่อน (LOTO)
✓ มีผู้ช่วยเฝ้าดูขณะ Force
✓ จด Note ทุกครั้งที่ Force
✓ เอา Force ออกทันทีที่ Debug เสร็จ
✓ ตรวจสอบ "No Force Active" ก่อนออกจากงาน
```

### วิธีใช้ Force อย่างปลอดภัย

```
Siemens TIA Portal:
1. Watch Table → กำหนดค่าใน "Modify value"
2. Click "Modify all" หรือ "Modify now"
3. Force Input: Project Tree → Monitoring & Forcing → Force table
4. ใส่ Force value → Enable Forces → Apply

สัญญาณ Force Active:
- สัญลักษณ์ "F" ปรากฏที่ Address
- Task Bar แสดง "Forces active"

เอา Force ออก:
- Force Table → Disable Forces
- Remove All Forces

Allen-Bradley:
1. Controller Tags → Force to 1/0
2. Enable Forces (จำเป็น!)
3. Remove All Forces เมื่อเสร็จ

Force Indicator:
I/O Map แสดง "F" ที่ Forced Tag

Mitsubishi:
1. Device Monitor → Force Column → Force ON/OFF
2. Force OFF ทั้งหมด: Online → Force Device OFF

OMRON:
1. Watch Window → Right-click → Force Bit
2. Release All Forces: PLC → Force Bit → Release All
```

---

## Step 374: Program Scan Monitoring

### การวิเคราะห์ Scan Time

```
Scan Time คือเวลาที่ PLC ใช้ Execute Program ทั้งหมด 1 รอบ

Scan Time ปกติ:
- Simple Machine Control: 5-20ms
- Complex Program + HMI: 20-50ms
- Heavy Math/Communication: 50-100ms

Scan Time สำคัญเพราะ:
- ยิ่งนาน ยิ่งตอบสนอง Input ช้า
- ถ้าเกิน Watchdog Timeout → CPU Fault
- I/O Update ช้า → Control Performance แย่ลง

อ่าน Scan Time:
Siemens:
Online → Diagnostics → Cycle time
หรือ Special Memory: SMW0 (Scan time in ms)

Allen-Bradley:
Controller Properties → Advanced
"Last scan time", "Maximum scan time"

Mitsubishi:
D8010: Current Scan Time (×0.1ms)
D8011: Minimum Scan Time
D8012: Maximum Scan Time

OMRON:
A264: Cycle Time (×0.1ms)
A263: Minimum Cycle Time
A262: Maximum Cycle Time
```

### การแก้ Scan Time ที่นานเกินไป

```
สาเหตุ Scan Time ยาวและวิธีแก้:

1. Loop ที่ไม่จำเป็น:
สาเหตุ: FOR Loop ซ้อนกันหรือ Loop ขนาดใหญ่
แก้ไข: ใช้ Interrupt หรือแบ่ง Loop ออกเป็น Scan หลายรอบ

2. Communication Blocking:
สาเหตุ: MSG/PUT/GET Instruction รอ Response
แก้ไข: ใช้ Non-blocking Communication (ตรวจ Done Bit)

3. Math Operation ซับซ้อน:
สาเหตุ: Floating Point Math หลายชุดต่อ Scan
แก้ไข: ย้ายไปทำใน Cyclic Interrupt (ไม่ผ่าน Main Scan)

4. Too Many Subroutine Calls:
สาเหตุ: เรียก FB/FC มากเกินไปโดยไม่จำเป็น
แก้ไข: ใช้ Conditional Call (เรียกเมื่อจำเป็นเท่านั้น)

ตัวอย่างการแก้ใน Siemens:
// แบบเดิม (Blocking):
CALL FB50 "PID_Control"  // เรียกทุก Scan

// แบบปรับปรุง (Cyclic Interrupt OB30 @ 100ms):
// ย้าย PID_Control ไปใน OB30 แทน
// ลด Load บน Main Scan Cycle (OB1)
```

---

## Step 375: Fault History และ Diagnostic Buffer

### การอ่าน Diagnostic Buffer

```
Diagnostic Buffer บันทึก Event ที่เกิดขึ้นใน PLC
ช่วย Debug ปัญหาที่เกิดขึ้นในอดีต (เหตุการณ์ผ่านไปแล้ว)

Siemens TIA Portal - Diagnostic Buffer:
Online → Diagnostics → Diagnostic buffer

Events ที่บันทึก:
┌───────────────────────────────────────────────────────────┐
│ Diagnostic Buffer                                         │
├──────┬──────────────┬────────────────────────────────────┤
│ No.  │ Timestamp    │ Event                             │
├──────┼──────────────┼────────────────────────────────────┤
│ 1    │ 14:23:05.123 │ Mode: STOP to RUN                 │
│ 2    │ 14:22:58.045 │ I/O module failure Slot 3         │
│ 3    │ 14:22:57.999 │ CPU: STOP (Error code 0x2521)     │
│ 4    │ 13:45:12.200 │ Power supply voltage failure      │
│ 5    │ 08:01:23.000 │ CPU power on (cold restart)       │
└──────┴──────────────┴────────────────────────────────────┘

วิธีอ่าน Diagnostic Buffer อย่างมีประสิทธิภาพ:
1. ดู Event ล่าสุด (No. 1) → สาเหตุโดยตรง
2. ดูเหตุการณ์ก่อนหน้า → หาสาเหตุที่แท้จริง
3. ดู Pattern → ปัญหาซ้ำหรือเป็นครั้งแรก?

ตัวอย่าง:
Event 3 (14:22:57): CPU STOP = Symptom
Event 2 (14:22:58): I/O Module Failure = Cause
→ ปัญหาที่ I/O Module Slot 3 ทำให้ CPU Stop!
```

### Mitsubishi Diagnostic

```
Mitsubishi FX5U Error Display:
- ERR LED ติด = มี Error
- Error Code แสดงที่ D8000 (สำหรับ FX3U)

FX5U Error Reading:
GX Works3 → Diagnostics → System Monitor

Error Table:
┌──────────┬────────────────────────────────────────────┐
│ Error No.│ Description                                │
├──────────┼────────────────────────────────────────────┤
│ 1100     │ RAM Error (Change CPU)                    │
│ 2000     │ Watchdog Timer Error (Scan Too Long)      │
│ 3000     │ I/O Verify Error (I/O Config ไม่ตรง)      │
│ 6000-6xxx│ User Error (FALS Instruction ถูกเรียก)    │
└──────────┴────────────────────────────────────────────┘

Special Device Monitor (FX3U/FX5U):
D8000: Error Code
D8005: Battery Voltage (หน่วย 0.1V)
D8011: Minimum Scan Time (×0.1ms)
D8012: Maximum Scan Time (×0.1ms)
D8013: Current Second (0-59)
D8014: Current Minute
D8015: Current Hour
```

### Allen-Bradley Fault Log

```
AB Controller Fault ดูได้จาก:
1. Controller Properties → Faults
2. Online bar แสดง "FAULTED" สีแดง

Major Fault:
- CPU หยุดทำงาน
- Mode Switch ต้องเป็น PROG ก่อน Clear

Minor Fault:
- CPU ทำงานต่อ
- Clear ได้โดย Clear Minor Faults

Fault Codes:
Code 04: I/O Module Failure
Code 07: I/O Connection Timeout
Code 0D: Program execution problem (เช่น Array OOB)
Code 11: Watchdog Timeout

ดู Fault Detail:
Controller Tags → _TaskEventTriggers → ดู Fault bits
หรือ GSV Instruction อ่าน Fault data

GSV (Get System Value):
   M_Always
──┤├────────[GSV ControllerFault  MinorFaultBits  Minor_Faults]
   M_Always
──┤├────────[GSV ControllerFault  MajorFaultBits  Major_Faults]
```

---

## Step 376: Systematic Troubleshooting Approach

### การแก้ปัญหาอย่างเป็นระบบ

```
หลักการแก้ปัญหา PLC อย่างมืออาชีพ:

P-D-C-A Troubleshooting:
Plan → Do → Check → Act

ขั้นตอน:
1. IDENTIFY: ระบุปัญหาให้ชัดเจน
   "มอเตอร์ไม่สตาร์ท" ← ไม่ชัดพอ
   "มอเตอร์ M01 ไม่สตาร์ทหลังกด Start PB เมื่อ 9:35 น.
    Output Q0.0 ไม่เปลี่ยนแปลง ตรวจ Online แล้ว
    Contact Start_PB (I0.0) ไม่เปลี่ยนเมื่อกดปุ่ม" ← ชัดเจน

2. ISOLATE: แยกปัญหาว่าอยู่ที่ไหน
   Hardware vs Software?
   → ดู Output Coil Online
   → ถ้า Coil TRUE แต่ไม่ทำงาน = Hardware
   → ถ้า Coil FALSE = ดู Logic ต่อ

3. DIAGNOSE: วิเคราะห์สาเหตุ
   ทำ 5-Why Analysis:
   Why 1: ทำไมมอเตอร์ไม่สตาร์ท?
     → Output Coil = FALSE
   Why 2: ทำไม Coil FALSE?
     → Start_PB Input ไม่เปลี่ยน
   Why 3: ทำไม Input ไม่เปลี่ยน?
     → LED ที่ PLC ไม่ติดเมื่อกด PB
   Why 4: ทำไม LED ไม่ติด?
     → Voltage ที่ PB = 0V (ควรเป็น 24V)
   Why 5: ทำไม Voltage = 0V?
     → CB ที่ Supply +24V ไปยัง PB ตก!
   
   Root Cause: Circuit Breaker Trip

4. FIX: แก้ไขตามสาเหตุที่แท้จริง
5. VERIFY: ยืนยันว่าแก้ไขได้
6. DOCUMENT: บันทึกปัญหาและการแก้ไข
```

---

## Step 377: 8D Method สำหรับปัญหา PLC

### 8D Problem Solving

```
8D (8 Disciplines) เป็น Framework มาตรฐานอุตสาหกรรม
ใช้เมื่อปัญหาซับซ้อน หรือต้องรายงานลูกค้า

D1: Team Formation
    ตั้งทีมผู้แก้ปัญหา:
    - PLC Engineer (Leader)
    - Electrician
    - Production Supervisor
    - Maintenance Technician

D2: Problem Description
    5W1H:
    What: มอเตอร์ Conveyor 3 ไม่สตาร์ท
    When: วันจันทร์ 9:35 น. (3 ครั้งในสัปดาห์ที่ผ่านมา)
    Where: Line 2, Station 5
    Who: เกิดขึ้นขณะ Operator กด Start
    Why: (ยังไม่รู้)
    How Many: ทุกครั้งที่ Humidity สูง (สังเกตจาก Log)

D3: Containment Actions (ป้องกันผลกระทบชั่วคราว)
    - ตั้ง Manual Bypass (ถ้าทำได้อย่างปลอดภัย)
    - เพิ่มความถี่ในการตรวจสอบ
    - แจ้ง Production เพื่อวางแผนสำรอง

D4: Root Cause Analysis
    ใช้ Fishbone Diagram:
    
    Machine ── I/O Card moisture ingress
    Method  ── (ไม่เกี่ยว)
    Man     ── (ไม่เกี่ยว)
    Material── Contact Corrosion
    Measurement── (ไม่เกี่ยว)
    Environment── High Humidity → Main Cause!
    
    Root Cause: 
    Input Card ขาดซีลกันความชื้น ทำให้ Contact ของ
    Start Button เกิด Oxidation จากความชื้น
    ส่งผลให้ Resistance สูงขึ้น Input ไม่รับสัญญาณ

D5: Corrective Actions
    Short-term: ทำความสะอาด Contact + Contact Cleaner
    Long-term: เปลี่ยน PB ชนิด IP67 + ปิดผนึก Control Panel

D6: Implementation & Verification
    - ดำเนินการตาม D5
    - Monitor 2 สัปดาห์ ไม่พบปัญหาซ้ำ

D7: Prevent Recurrence
    - Update Maintenance Procedure เพิ่ม Contact Check
    - กำหนด IP Rating Standard สำหรับ Panel ใหม่
    - Training ทีม Maintenance

D8: Team Recognition
    - ปิด 8D Report
    - แจ้งทีมงาน
    - Update Lesson Learned Database
```

---

## Step 378: Oscilloscope สำหรับ PLC I/O

### ทำไมต้องใช้ Oscilloscope?

```
Oscilloscope ช่วยดูสัญญาณที่เปลี่ยนเร็วเกินกว่า PLC Scan จะจับได้

ตัวอย่างที่ Oscilloscope ช่วยได้:
1. Noise บน Input Signal
   → ทำให้ Counter นับ Phantom Pulse
   → Oscilloscope แสดง Spike ที่ PLC ไม่เห็น

2. Sensor Output ที่ไม่คงที่
   → Proximity Sensor ที่ใกล้จะเสียส่งสัญญาณ Intermittent
   → Oscilloscope แสดง Bounce หรือ Noise

3. Output ที่มี Short Duration
   → PLC ส่ง Pulse สั้นมาก (< 1ms)
   → Oscilloscope วัดได้ PLC Scan อ่านไม่ทัน

4. Communication Signal
   → RS-485 Signal Level ผิดปกติ
   → Modbus Frame Corruption

5. Power Supply Ripple
   → 24VDC มี Ripple สูง → ทำให้ Input อ่านค่าผิด
```

### การวัด PLC I/O ด้วย Oscilloscope

```
การตั้งค่า Oscilloscope สำหรับ PLC:

Digital Input (24VDC):
- Voltage Range: 0-30V (ใช้ 10:1 Probe)
- Time Base: 10ms/div (สำหรับดู Button Bounce)
- Trigger: Rising Edge

การวัด:
+ Probe → PLC Input Terminal (I0.0)
- Probe (Ground) → PLC 0VDC Terminal

สิ่งที่ดูใน Scope:
1. Signal Level: 0V = OFF, 24V = ON (PNP/Source)
2. Rise/Fall Time: ควร < 1ms
3. Noise/Spike: ควรไม่มีเกิน 2V
4. Bounce: ปุ่มกล ≤ 5ms bounce ปกติ

Counter นับผิด - วิเคราะห์ด้วย Scope:
                          ┌──┐  ┌──────────────────
24V                       │  │  │
                          │  │  │
0V  ──────────────────────┘  └──┘ ← 2 Bounce Pulses
                                     Counter นับ 3 ครั้ง!

แก้ไข: เพิ่ม Input Filter ใน PLC Settings
Siemens: Hardware Config → Input Channel → Filter time: 3ms
Mitsubishi: PLC Parameter → Input Response Time: 10ms
```

---

## Step 379: Data Trace และ Recording

### Siemens TIA Portal Trace

```
Trace Function ใน TIA Portal (S7-1500 เท่านั้น):
ใช้บันทึก Variable ตาม Trigger และแสดงเป็น Waveform

การสร้าง Trace:
1. Project Tree → Traces → Add new trace
2. กำหนด Trace Configuration:

Trace Configuration:
┌──────────────────────────────────────────────────────────┐
│ Trace Name: Motor_Start_Analysis                         │
├──────────────────────────────────────────────────────────┤
│ Sampling:                                                │
│  Rate: OB1 (ทุก Scan)                                    │
│  Max. samples: 4000                                      │
├──────────────────────────────────────────────────────────┤
│ Signals:                                                 │
│  "Motor_Run"        Bool                                 │
│  "Speed_Actual"     Int                                  │
│  "Current_A"        Real                                 │
│  "Start_PB"         Bool                                 │
├──────────────────────────────────────────────────────────┤
│ Trigger:                                                 │
│  Signal: "Start_PB"                                      │
│  Event: Rising edge                                      │
│  Pre-trigger: 20 samples                                 │
└──────────────────────────────────────────────────────────┘

3. Transfer to CPU
4. Start Recording
5. เหตุการณ์เกิดขึ้น → กด Stop Recording
6. Download และวิเคราะห์ Waveform

Waveform Analysis:
Time →
0ms    10ms   20ms   30ms   40ms   50ms

Start_PB: ─────┌──────────────────────────
               ↑ Trigger

Motor_Run: ─────────────┌─────────────────
                        ↑ 15ms delay (Timer)

Speed: 0─────────────────────/────→ 1500RPM
                              ↑ Motor accelerating
```

### Allen-Bradley Trend

```
Trend Tool ใน Studio 5000:
Logic → Trends → New Trend

Trend Settings:
- Tags: เลือก Tags ที่ต้องการ Plot
- Sample Rate: 100ms / 500ms / 1s
- Total Duration: 60s / 300s / 3600s
- Trigger: เลือก Tag เป็น Trigger

Trend Waveform ดู:
- เวลาที่ Output เปิด/ปิด
- ความสัมพันธ์ระหว่าง Input และ Output
- Value ที่เปลี่ยนแปลงตลอดเวลา
- Intermittent Problem (เกิดขึ้นชั่วคราว)

Export Trend Data:
- Export to CSV → วิเคราะห์ใน Excel
- Copy Screenshot → บันทึกเป็น Evidence
```

### Mitsubishi Data Logging (FX5U)

```
FX5U Built-in Data Logging:

GX Works3 → Navigation → Parameter → FX5UCPU
→ Memory Card → Data Logging Setting:

Logging Settings:
- Trigger: Cyclic (ทุก 1 วินาที)
- Variables to Log:
  D100 (Temperature)
  D101 (Pressure)
  D102 (Flow Rate)
  Y0.0 (Motor Status)
- File Format: CSV
- Max Files: 100 ไฟล์

SD Card Data File (CSV):
Date,Time,D100,D101,D102,Y0
2024-01-15,09:30:00,750,1234,500,1
2024-01-15,09:30:01,752,1235,498,1
2024-01-15,09:30:02,755,1236,501,1
...

วิเคราะห์ด้วย Excel:
1. Insert → Chart → Line Chart
2. X-Axis: Timestamp
3. Y-Axis: Process Values
4. หา Correlation ระหว่าง Variables
```

---

## Step 380: Commissioning Checklist

### Pre-commissioning Checklist

```
ก่อน Power ON PLC ครั้งแรกต้องตรวจสอบ:

□ MECHANICAL CHECK:
  □ PLC ติดตั้งบน DIN Rail แน่นหนา
  □ Module ทุกตัว Push-in สนิท
  □ Terminal Block ขันแน่น
  □ Strain Relief สายทุกเส้น

□ ELECTRICAL CHECK:
  □ Power Supply Voltage ถูกต้อง (24VDC ±10%)
  □ Polarity ถูกต้อง (+ และ - ไม่สลับ)
  □ Grounding ครบถ้วน
  □ EMC Protection (Surge Suppressor บน Coil)
  □ Fuse/CB ขนาดถูกต้อง

□ I/O WIRING CHECK:
  □ Input สาย Type ถูกต้อง (NPN/PNP ตรงกับ Input Card)
  □ Output โหลดไม่เกิน Rating
  □ Interlock ถูก Address ถูกต้อง
  □ Safety Circuit เป็น Hardwired (ไม่ใช่แค่ Software)

□ COMMUNICATION CHECK:
  □ IP Address ไม่ซ้ำกัน
  □ Network Cable ถูกประเภท (Shielded, Cat5e)
  □ Terminator Resistor ติดตั้ง (RS-485)
```

### First Power-On Procedure

```
ขั้นตอน Power On ครั้งแรก:

1. Isolate Outputs ก่อน:
   - ถอด Output Module
   - หรือถอดสายที่ Output Terminal
   - เพื่อป้องกัน Machine เคลื่อนที่โดยไม่คาดคิด

2. Power On Power Supply → ตรวจ LED
   PWR = ติดสีเขียว = Power OK
   ERR = ติด = มี Error (ดู Diagnostic)
   RUN = ดับ = ยัง STOP Mode

3. Connect PC และ Download Program:
   a. เปิด Software (TIA Portal/Studio 5000/GX Works3/CX-Programmer)
   b. Set PLC ไป STOP Mode
   c. Download Program
   d. Verify Download Success (No Errors)

4. Set to RUN Mode:
   - ดู Scan Time: Normal หรือ Abnormal?
   - ดู Error/Warning ทันที

5. Test Inputs:
   - ทดสอบทุก Input ทีละจุด
   - Force Output ตรงกับ Physical Input ที่ทดสอบหรือไม่?

6. Test Outputs (หลังติด Output Module กลับ):
   - Force Output ทีละจุด
   - ตรวจว่า Actuator ตัวไหนทำงาน
   - ตรวจ Address ถูกต้องหรือไม่

7. Functional Test:
   - ทดสอบ E-Stop ก่อนเสมอ
   - ทดสอบ Safety Interlock
   - ทดสอบ Normal Operation
```

---

## Step 381: Commissioning Test Document

### I/O Test Sheet

```
I/O VERIFICATION TEST SHEET
Project: _____________________ Date: _____________
PLC: ________________________ Engineer: __________

DIGITAL INPUTS:
┌──────┬──────────────┬──────────────┬──────┬──────────────┐
│ I/O  │ Description  │ Test Method  │ Pass │ Remarks      │
├──────┼──────────────┼──────────────┼──────┼──────────────┤
│ I0.0 │ Start PB     │ Press PB     │  ✓   │              │
│ I0.1 │ Stop PB      │ Press PB     │  ✓   │              │
│ I0.2 │ E-Stop       │ Press E-Stop │  ✓   │              │
│ I0.3 │ Level High   │ Float Switch │  ✓   │              │
│ I0.4 │ Temp High    │ Thermostat   │  ✗   │ ไม่ตอบสนอง   │
│ I1.0 │ Motor Fault  │ Relay Test   │  ✓   │              │
└──────┴──────────────┴──────────────┴──────┴──────────────┘

DIGITAL OUTPUTS:
┌──────┬──────────────┬──────────────┬──────┬──────────────┐
│ I/O  │ Description  │ Verified     │ Pass │ Remarks      │
├──────┼──────────────┼──────────────┼──────┼──────────────┤
│ Q0.0 │ Motor M01    │ Motor Runs   │  ✓   │              │
│ Q0.1 │ Valve V01    │ Valve Opens  │  ✓   │              │
│ Q0.2 │ Alarm Horn   │ Horn Sounds  │  ✓   │              │
│ Q0.3 │ Run Light    │ Lamp Lights  │  ✓   │              │
└──────┴──────────────┴──────────────┴──────┴──────────────┘

ANALOG INPUTS:
┌──────┬──────────────┬──────────────┬──────┬──────────────┐
│ I/O  │ Description  │ 0% → 100%    │ Pass │ Remarks      │
├──────┼──────────────┼──────────────┼──────┼──────────────┤
│ AI0  │ Temperature  │ 0°C → 100°C  │  ✓   │ Span OK      │
│ AI1  │ Pressure     │ 0 → 10 Bar   │  ✓   │ Zero OK      │
└──────┴──────────────┴──────────────┴──────┴──────────────┘

Note: I0.4 Temp High ต้องตรวจ Thermostat Setting และ Wiring
```

---

## Step 382: Common PLC Problems และ Solutions

### ปัญหา Hardware ที่พบบ่อย

```
1. Input ไม่รับสัญญาณ:

สาเหตุที่เป็นไปได้:
a) ไฟ Supply ไม่ถึง Sensor
   ตรวจ: Voltmeter ที่ Sensor Power Supply
   แก้ไข: ตรวจ Fuse, CB, Terminal

b) Sensor ชนิดผิด (NPN แทน PNP)
   ตรวจ: Datasheet ของ Sensor
   แก้ไข: เปลี่ยน Sensor หรือ Input Card Type

c) Sensor ชำรุด
   ตรวจ: ใช้ Multimeter วัด Output ของ Sensor
   แก้ไข: เปลี่ยน Sensor

d) Input Card ชำรุด
   ตรวจ: Force Output อื่นดู, ลอง Input Channel อื่น
   แก้ไข: เปลี่ยน Input Module

e) Noise/Interference
   ตรวจ: Oscilloscope ดู Signal Quality
   แก้ไข: Shield Cable, Separate from Power Cable
          เพิ่ม Ferrite Bead, ปรับ Filter Time

2. Output ไม่ทำงาน:

a) Output Coil = TRUE แต่ไม่มีสัญญาณ:
   ตรวจ: Voltage ที่ Output Terminal
   → ถ้า 0V: Output Card ชำรุด
   → ถ้ามี Voltage แต่ Actuator ไม่ทำงาน: ปัญหาที่ Load

b) Overcurrent → Output Latched OFF:
   ตรวจ: ดู I/O Error ใน Diagnostic
   แก้ไข: ตรวจ Load Resistance, ใช้ External Relay

c) Flyback Protection ขาด:
   ตรวจ: Coil ไม่มี Diode/Varistor
   แก้ไข: ติด Freewheeling Diode ขนาน Coil

3. Communication Error:

a) Ethernet: Cannot Connect
   ตรวจ: Ping PLC IP, ตรวจ Cable
   แก้ไข: ตรวจ IP ซ้ำกัน, Gateway ถูกต้อง?

b) RS-485: No Response
   ตรวจ: Terminator Resistor (120Ω ที่ปลายทั้งสอง)
   แก้ไข: A+/B- ถูกต้อง, Baud/Parity ตรงกัน
```

---

## Step 383: ปัญหา Software/Logic ที่พบบ่อย

```
1. Timer ไม่ทำงานหรือทำงานผิด:

สาเหตุ:
- Contact นำ Timer กลับ FALSE → Timer Reset
- Timer Number ซ้ำ (ใช้ Timer เดียวกัน 2 ที่)
- ลืมว่า Timer เป็น Retentive หรือ Non-Retentive

ตรวจ: Monitor Timer Coil + Timer Contact พร้อมกัน
แก้ไข: ตรวจ Timer Number ไม่ซ้ำ
        ใช้ FB แทน FC สำหรับ Timer (มี Static)

2. Counter นับผิด:

สาเหตุ:
- Signal Bounce (PB Bounce)
- ใช้ Counter ผิดชนิด (CTU/CTD)
- Counter Number ซ้ำ
- Reset เร็วเกินไป

แก้ไข: ใช้ Software Debounce (OSR + Timer)
        หรือ Hardware Filter ที่ Input

Debounce Circuit ด้วย PLC:
// Rung 1: Detect Rising Edge
   Part_Sensor            M_Part_OSR
──┤├──────────────────────(OSR)──   // 1 Scan Pulse

// Rung 2: Debounce Timer
   M_Part_OSR             T_Debounce
──┤├──────────────────────(TIM K3)── // 30ms

// Rung 3: Valid Count after Debounce
   T_Debounce             CTU_Part
──┤├──────────────────────(CTU K100)──

3. Race Condition:

สาเหตุ: Output เปลี่ยนระหว่าง Scan เดียวกัน
ทำให้ Logic เก่าใช้ค่า Output ใหม่ หรือในทางกลับกัน

ตัวอย่าง:
Rung 1: A ← B + C
Rung 2: B ← A (ใช้ A ที่เปลี่ยนแล้ว!)

แก้ไข: ใช้ Temp Variable เก็บค่าก่อน
        หรือ เรียงลำดับ Rung ให้ถูกต้อง

4. Data Overflow:

สาเหตุ: ผล Math เกิน Range ของ Data Type
เช่น 1000 × 1000 = 1,000,000 ใส่ใน Int (สูงสุด 32767) ผิด!

แก้ไข: ใช้ DInt หรือ Real สำหรับ Math ที่อาจเกิน
        ตรวจสอบ Range ก่อนใช้ Data Type
```

---

## Step 384: การจัดการ EMC Interference

### Electromagnetic Interference (EMI)

```
EMI คือปัญหาสำหรับ PLC ในโรงงาน
แหล่ง EMI ในโรงงาน:
- VFD (Variable Frequency Drive) → Harmonic Noise
- Welding Machine → Strong Transient
- Large Motor Start → Voltage Dip
- Fluorescent Lighting → 50Hz Harmonic

ผลกระทบต่อ PLC:
- Input อ่านค่าผิด (FALSE เป็น TRUE หรือกลับกัน)
- Counter นับ Phantom Pulse
- Communication Error บ่อย
- CPU Reset โดยไม่ทราบสาเหตุ
- Analog Reading Noisy

การป้องกัน EMI:
1. Physical Separation:
   - สาย Signal แยกจากสาย Power
   - ระยะห่าง > 300mm
   - ถ้าตัดขวาง ต้องตั้งฉาก 90°

2. Shielded Cable:
   - ใช้ Shielded Cable สำหรับ Analog Signal
   - Shield ต้องต่อ Ground เพียงจุดเดียว (ที่ PLC Panel)

3. Grounding:
   - Panel Earth Ground ต้องดี (< 5Ω ต่อดิน)
   - DIN Rail ต้องต่อ Ground
   - PLC Power Supply Ground

4. Surge Protection:
   - ใช้ TVSS (Transient Voltage Surge Suppressor)
   - ติด RC Snubber บน Relay Coil
   - ติด Diode บน DC Solenoid Coil

5. Filter:
   - Line Filter ที่ Power Input ของ PLC
   - Input Filter ใน PLC Parameter (เพิ่มถ้า Noise)
   - Ferrite Bead บน Signal Cable
```

---

## Step 385: Documentation Practices

### ทำไมต้อง Document?

```
Documentation ช่วยให้:
1. Debug ได้เร็วขึ้น (ไม่ต้อง Reverse Engineer)
2. ส่งงานต่อทีมอื่นได้
3. แก้ไขปัญหาหลายปีให้หลัง
4. Compliance กับ Standard (ISO, ATEX)

เอกสารที่ควรมี:
a) Program Documentation (ใน Software)
b) I/O List (Excel/PDF)
c) Panel Wiring Diagram
d) Network Diagram
e) Operation Manual
f) Maintenance Manual
g) Alarm/Fault List
```

### Program Documentation Standards

```
การเพิ่ม Comment ใน LAD:

1. Rung Title (สรุปจุดประสงค์):
   "Network 1: Motor M01 Start/Stop Control"

2. Comment ด้านบน Rung (อธิบาย Logic):
   "Start PB เปิด Seal-In Circuit
    E-Stop NC ต้องต่อ (NC=TRUE เมื่อ Normal)
    Stop PB NC ต้องต่อ"

3. Address Comment:
   I0.0 = "Start_PB - Green Button Panel A"
   Q0.0 = "Motor_M01 - Conveyor 1 Main Motor"

4. Network Comment (ทุก Network):

ตัวอย่างที่ดี:
/*=======================================================
   Network 5: Motor Speed Ramp Control
   
   ฟังก์ชัน: ค่อยๆ เพิ่มความเร็วมอเตอร์จาก 0 ไป Setpoint
             เพื่อป้องกัน Mechanical Shock
   
   Logic:
   - เมื่อ Motor_Enable = TRUE และ Speed < Setpoint
   - เพิ่ม Speed ทุก 100ms ตาม Ramp_Rate
   - หยุดเพิ่มเมื่อ Speed = Setpoint
   
   Variables:
   Speed_SP    : Setpoint (RPM) จาก HMI D100
   Speed_Actual: อ่านจาก Drive D101
   Ramp_Rate   : เพิ่มต่อ 100ms = D102 (RPM/100ms)
   ======================================================*/
```

### Naming Convention

```
Standard Naming Convention:

PLC Tag Naming (Siemens/AB):
<System>_<Description>_<Function>

ตัวอย่าง:
CV1_Speed_SP      = Conveyor 1 Speed Setpoint
CV1_Motor_Run     = Conveyor 1 Motor Running Status
ST1_Temp_PV       = Station 1 Temperature Process Variable
ST1_Temp_Alarm_HH = Station 1 Temperature High-High Alarm

OMRON/Mitsubishi (ใช้ Symbol Table):
M100 → "CV1_Motor_Run"
D200 → "CV1_Speed_SP"
T10  → "CV1_Start_Timer"

ห้าม:
- ชื่อสั้นเกินไป: M1, D5 → ไม่รู้ว่าทำอะไร
- ชื่อยาวเกินไป: Conveyor_Number_1_Motor_Main_Running_Status
- อักขระพิเศษ: Motor#1, 1st_Motor (บาง Platform ไม่รองรับ)
- Space ในชื่อ: Motor Run → ใช้ Motor_Run หรือ MotorRun
```

---

## Step 386: Preventive Maintenance สำหรับ PLC

### PM Checklist สำหรับ PLC System

```
Monthly PM:
□ ตรวจสภาพ Panel ทั่วไป (ไม่มีสัตว์แมลงเข้า)
□ ทำความสะอาด Filter Air ของ Panel (ถ้ามี)
□ ตรวจ LED Status ทุกตัว
□ ดาวน์โหลด Diagnostic Buffer และตรวจสอบ
□ ตรวจ Terminal Block (ไม่ Loose)
□ ตรวจอุณหภูมิใน Panel (< 50°C)
□ ตรวจ Battery Level (Diagnostic Buffer)

Quarterly PM:
□ Backup Program ลง PC และ USB/SD
□ Update Backup Documentation
□ ตรวจสอบ Calibration ของ AI Module
□ Test E-Stop และ Safety Circuit
□ ตรวจ Communication Status ทุก Node

Annual PM:
□ เปลี่ยน Battery (ถ้าใกล้หมด > 3 ปี)
□ ทดสอบ UPS (ถ้ามี)
□ Full I/O Test
□ Review Alarm History หาปัญหาซ้ำ
□ Update PLC Firmware (ถ้ามี Bug Fix)
□ Review และ Update Documentation

Battery Replacement:
ก่อนเปลี่ยน Battery:
1. Backup Program ไปก่อน (Battery หมด ข้อมูลสูญ)
2. เปลี่ยน Battery ขณะ PLC Running (ถ้าทำได้)
   → Memory จะไม่สูญหายเพราะ Power Capacitor ชาร์จไว้
3. ถ้าต้อง Power Off เพื่อเปลี่ยน: Download Program กลับทันที
```

---

## Step 387: Advanced Troubleshooting Scenarios

### Scenario 1: CPU ไป FAULT ทุก 3 วัน

```
อาการ: CPU S7-1214C ไป STOP/FAULT ทุกๆ 3 วัน
       Restart แล้วปกติ แต่เกิดซ้ำ

การวิเคราะห์:

Step 1: ดู Diagnostic Buffer (Online → Diagnostics buffer)
Event ที่ 1: CPU STOP
Event ที่ 2: "I/O Module Not Responding - Slot 3"
Event ที่ 3: "Communication Error - PROFINET"

→ ปัญหาที่ PROFINET Communication

Step 2: ตรวจ PROFINET Topology
ในช่วงที่เกิดปัญหา: อะไรเกิดขึ้น?
→ ตรวจ Production Log: ทุกครั้งที่ CPU Fault
  มีการเปิด VFD ขนาดใหญ่ (45kW) ใน Area เดียวกัน

Step 3: ตรวจ Network Cable
→ Cable PROFINET วางใกล้สาย Power ของ VFD
→ ไม่มี Shielding Proper

Root Cause:
VFD 45kW ส่ง EMI เข้า PROFINET Cable
ทำให้ Communication Error → CPU Fault

Corrective Actions:
1. ใช้ Industrial PROFINET Cable (Shielded)
2. แยกสายให้ห่างจาก VFD Power Cable > 30cm
3. ติด Ferrite Bead ที่ PROFINET Cable ปลายทั้งสอง
4. เพิ่ม PROFINET Watchdog Time ใน CPU Settings

Result: ปัญหาหายไป ไม่เกิด Fault อีกใน 6 เดือน
```

### Scenario 2: Counter นับ Random

```
อาการ: Counter ชิ้นงานแสดงตัวเลขสุ่ม
       บางครั้งนับมากกว่าชิ้นงานจริง 2-3 เท่า

การวิเคราะห์:

Step 1: ตรวจ Hardware
→ ติด Oscilloscope ที่ Input ของ Proximity Sensor (I0.3)
→ สังเกต Waveform

Waveform ที่พบ:
                 ┌─┐ ┌──────────────────────
24V              │ │ │
                 │ │ │
0V ──────────────┘ └─┘ ← Extra pulse! (Sensor Bounce)

→ Proximity Sensor ส่ง Extra Pulse เมื่อ Object ขอบๆ

Step 2: ตรวจ Sensor Position
→ Sensor ติดตั้งไม่ตรงกับตำแหน่ง Object อย่างถูกต้อง
→ Object ผ่าน Edge ของ Sensing Range → Bounce

Step 3: Options แก้ไข:
Option A (Hardware): 
  → ปรับ Sensor Position
  → ใช้ Sensor ที่มี Hysteresis สูงกว่า

Option B (Software - Input Filter):
  Siemens: Hardware Config → Input Filter 3ms
  → ถ้า Signal สั้นกว่า 3ms → ไม่รับ

Option C (Software - Debounce Logic):
  เพิ่ม Rung:
     I0.3         [TON]      M_Valid_Pulse
  ──┤├────────[IN    Q]─────( )──
              T#20ms──PT    ET

     M_Valid_Pulse     [CTU]
  ──┤├──────────────────Counter

Root Cause: Sensor Position ไม่เหมาะสม
Corrective: ปรับ Sensor Position + เพิ่ม Input Filter 3ms
```

---

## Step 388: Remote Troubleshooting

### การ Remote Access ผ่าน Network

```
เมื่อต้องแก้ปัญหาโดยไม่ต้องไปหน้างาน:

Options สำหรับ Remote PLC Access:

1. VPN + Direct PLC Connection:
   - VPN ไปยัง Plant Network
   - Connect TIA Portal / Studio 5000 ผ่าน VPN
   - ต้องการ Static IP หรือ DDNS

2. SCADA Remote Access:
   - Wonderware / iFIX → Remote Monitor
   - ดูสถานะ Process แต่อาจไม่ได้ดู PLC Code

3. TeamViewer Industrial:
   - Remote Control PC ที่อยู่หน้างาน
   - ดู / แก้ไข PLC Program ผ่าน Remote PC

4. Siemens TeleService:
   - Built-in Remote Service สำหรับ S7
   - ผ่าน GPRS/3G Modem + Industrial Router

5. PROFINET Remote:
   - Siemens S7-1200 V4.4+: OPC UA + Remote Access
   - Connect จากนอก Plant ได้ผ่าน Internet

Security Considerations:
⚠️ Remote Access ต้องมี Security:
- VPN Encryption
- 2-Factor Authentication
- Log ทุก Session
- Time-limited Access
- ไม่ใช้ Default Password!
```

---

## Step 389: Best Practices สำหรับ PLC Engineer

### 10 หลักการสำคัญ

```
1. Document ก่อน Debug:
   อ่านเอกสารที่มีก่อนแตะ PLC
   ถ้าไม่มีเอกสาร → Print Screen ทุก Step ก่อนเปลี่ยน

2. Backup ก่อนเปลี่ยน:
   ทุกครั้งก่อนแก้ไข Program → Upload+Backup ก่อน
   "2 Backups ดีกว่า 0 Backups"

3. อย่า Force ใน Production:
   Force เฉพาะใน Test หรือ Commissioning
   ใน Production: แก้ที่ Root Cause เท่านั้น

4. Test ใน Offline/Sim ก่อน:
   ใช้ Simulator ทดสอบ Logic ก่อน Download จริง
   โดยเฉพาะการแก้ไข Safety Critical Code

5. Change Control:
   ทุก Change ต้องมี Approval และ Documentation
   "ใครเปลี่ยนอะไร เมื่อไหร่ ทำไม"
   ใช้ Version Control (SVN/Git สำหรับ PLC Code)

6. Test E-Stop ก่อนเสมอ:
   ทุกครั้งที่ Commission หรือ Restart หลังแก้ไข
   → Test E-Stop ก่อนเปิด Production

7. หา Root Cause ไม่ใช่แค่ Symptom:
   "ปิด Alarm ได้แต่ไม่ได้แก้ปัญหา"
   ทำ 5-Why ทุกครั้งที่มีปัญหาซ้ำ

8. ถ่ายทอดความรู้:
   Document Lesson Learned
   Share กับทีม: "ปัญหา X แก้ด้วย Y"

9. อัปเดตความรู้:
   Technology เปลี่ยนเร็ว
   ติดตาม: Siemens/AB/Mitsubishi/OMRON Technical Blog
   เข้า Training อย่างน้อยปีละครั้ง

10. ปลอดภัยเสมอ:
    ไม่มีงานไหนคุ้มกับการบาดเจ็บ
    LOTO ทุกครั้งที่ทำงานกับ Electrical Panel
    Follow Safety Procedures เสมอ
```

---

## Step 390: Lab Projects สุดท้าย

### Lab 20-1: Comprehensive Debug Exercise

```
โจทย์: โปรแกรมต่อไปนี้มีข้อผิดพลาด 5 จุด
หาและแก้ไขข้อผิดพลาดทั้งหมด

Program (Siemens S7-1200):
Network 1: Motor Control
    "Start_PB"    "E_Stop"        "Motor_Run"
    ──┤├──────────┤├───────────────────( )──
    
    "Motor_Run"
    ──┤├──────────┘

Network 2: Safety Guard
    "Guard_Open"           "Motor_Run"
    ──┤├────────────────────────(R)──

Network 3: Run Timer
    "Motor_Run"            [TON]
    ──┤├──────────────[IN    Q]── "Maint_Due"
                  T#8H──PT    ET── "Run_Time"

Network 4: Production Counter
    "Part_Sensor"          [CTU]
    ──┤├──────────────[CU    Q]── "Batch_Done"
    "Reset_PB"───────[R   CV]──
              100────[PV      ]

Network 5: Speed Control
    "Auto_Mode"            [MOVE]
    ──┤├──────────────[EN    ENO]──
              1500───[IN     OUT]── "Speed_SP"

// Bugs:
// 1. Network 1: E_Stop ใช้ NO Contact (ควรเป็น NC)
// 2. Network 1: ไม่มี Stop_PB
// 3. Network 1: Seal-In Circuit ผิด (Motor_Run ต่อผิดตำแหน่ง)
// 4. Network 3: ไม่มี Instance DB สำหรับ TON
// 5. Network 5: MOVE ไม่ต้องการ Enable Contact (ทำงานตลอด)
//               แต่ถ้าไม่มี Auto_Mode ค่าจะไม่เปลี่ยน ซึ่งอาจผิด Design
```

### Lab 20-2: Commissioning a Real System

```
โจทย์: Commission ระบบ Pump Control ใหม่

Hardware:
- Siemens S7-1214C DC/DC/DC
- 1 x Pressure Sensor (4-20mA, 0-10 Bar)
- 1 x Level Sensor (4-20mA, 0-100%)
- 1 x Flow Meter (Pulse Output, 10 Pulse/Liter)
- 1 x VFD Motor 5.5kW (Modbus RTU, Address 1)
- HMI Panel

เอกสารที่ต้องสร้าง:
1. I/O List (Excel)
2. P&ID Diagram
3. Network Diagram
4. Program Documentation

Commissioning Steps:
1. Pre-commissioning Safety Check
2. I/O Test (Force Method)
3. Communication Test (VFD Modbus)
4. Functional Test (Auto/Manual Mode)
5. Safety Test (E-Stop, High Pressure Alarm)
6. Performance Test (Control accuracy)
7. As-Built Documentation Update

Acceptance Criteria:
✓ All I/O verified
✓ E-Stop tested
✓ Pressure control ±0.5 Bar
✓ Level control ±5%
✓ All Alarms functional
✓ Documentation complete
```

---

## ตารางสรุป Debugging Tools และเมื่อใช้

```
┌─────────────────────────────────────────────────────────────────┐
│ Tool                │ ใช้เมื่อ                               │
├─────────────────────┼──────────────────────────────────────────┤
│ Online Monitor LAD  │ ดูสถานะ Logic ขณะทำงาน                  │
│ Watch Table         │ Monitor หลาย Variable พร้อมกัน          │
│ Force Values        │ ทดสอบ Output โดยไม่ต้องกด Input         │
│ Diagnostic Buffer   │ หา Error ที่เกิดขึ้นในอดีต              │
│ Trend/Trace         │ ดู Signal เปลี่ยนแปลงตามเวลา            │
│ Oscilloscope        │ ดู Signal Noise, Bounce, Timing         │
│ Multimeter          │ ตรวจ Voltage, Current, Resistance       │
│ Cross-Reference     │ หาว่า Tag ไหนใช้ที่ไหนบ้าง              │
│ 5-Why Analysis      │ หา Root Cause ของปัญหาซ้ำ              │
│ 8D Method           │ แก้ปัญหาอย่างเป็นระบบ + รายงาน         │
│ Data Logging        │ บันทึกข้อมูล Process ระยะยาว            │
│ Simulation          │ ทดสอบ Logic ก่อน Download จริง          │
└─────────────────────┴──────────────────────────────────────────┘
```

---

## สรุปทั้งหมดของ Part 20

**ความรู้ที่ได้รับจาก Part 20:**

1. **Online Monitoring** เป็น First Tool ที่ต้องใช้เมื่อมีปัญหา
2. **Watch Table** ช่วยดูหลาย Variable พร้อมกันได้อย่างมีประสิทธิภาพ
3. **Force Values** ต้องใช้อย่างระมัดระวัง มีความเสี่ยงสูง
4. **Diagnostic Buffer** บันทึก History Events ช่วย Debug ปัญหาในอดีต
5. **5-Why Analysis** ช่วยหา Root Cause ไม่ใช่แค่ Symptom
6. **8D Method** เป็น Framework มาตรฐานสำหรับปัญหาซับซ้อน
7. **Oscilloscope** จำเป็นสำหรับปัญหา EMI, Bounce, Timing
8. **Documentation** เป็นพื้นฐานสำคัญของงาน Maintenance ที่ดี
9. **Commissioning Checklist** ป้องกันปัญหาก่อนเปิดใช้งานจริง
10. **Safety First** ทุกการ Debug ต้องคำนึงถึงความปลอดภัยเสมอ

---

## สรุปตลอดหลักสูตร (Part 1-20)

```
คุณได้เรียนรู้ตั้งแต่:
Part 1-5:  พื้นฐาน Ladder Diagram (Contact, Coil, Logic, Timer, Counter)
Part 6-10: Memory Bits, Math, Comparison, Data Move, First Program
Part 11-15: I/O Wiring, Safety, Motor Control, Conveyor, HMI
Part 16:   Siemens S7-1200/1500 + TIA Portal
Part 17:   Allen-Bradley Logix + Studio 5000
Part 18:   Mitsubishi FX5U/Q + GX Works3
Part 19:   OMRON CP1H/CJ + CX-Programmer
Part 20:   Debugging, Troubleshooting, Commissioning

สิ่งที่ต้องทำต่อ:
1. ฝึกกับ PLC จริงหรือ Simulator
2. ทำ Lab Project ทุกข้อ
3. หา Project จริงในที่ทำงาน
4. เข้า Training Certificate ของแต่ละ Brand
5. สร้าง Portfolio ของงานที่ทำ

Certificate ที่แนะนำ:
- Siemens: SITRAIN Certified
- Allen-Bradley: Rockwell Automation Certified Specialist
- Mitsubishi: MELSEC Certified Engineer
- OMRON: OMRON Technical Training Certificate
```

---

**[← Part 19: OMRON](part19_omron.md) | [กลับสู่สารบัญ](../README.md)**

*เนื้อหา Part 20 ครอบคลุม Steps 371-390*
*ครบสมบูรณ์ทั้งหมด 390 Steps ของหลักสูตร Ladder Diagram PLC Programming*
*ลิขสิทธิ์ © 2024 Ladder Diagram PLC Course*
