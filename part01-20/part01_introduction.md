# Part 01: บทนำ Ladder Diagram และ PLC
## Steps 1-15: พื้นฐานที่ต้องรู้ก่อนเริ่มเขียนโปรแกรม

---

## Step 1: Ladder Diagram คืออะไร?

**Ladder Diagram (LD)** คือภาษาโปรแกรม PLC ที่ออกแบบมาให้มีลักษณะเหมือน **แผนผังวงจรไฟฟ้า (Electrical Relay Ladder)** โดยประกอบด้วย:

```
Power Rail (L+)          Power Rail (L-)
    |                         |
    |----[ ]----[ ]----( )-----|
    |   Contact Contact  Coil  |
    |                         |
    |----[ ]----[/]----( )-----|
    |    NO     NC     Coil   |
    |                         |
```

### ทำไมต้องใช้ Ladder Diagram?
| เหตุผล | รายละเอียด |
|--------|-----------|
| เข้าใจง่าย | ช่างไฟและวิศวกรคุ้นเคยกับ Relay Logic |
| มาตรฐานสากล | IEC 61131-3 รองรับทุกยี่ห้อ |
| Debug ง่าย | เห็น Power Flow แบบ Real-time |
| Documentation ดี | เป็น Electrical Drawing ในตัว |

---

## Step 2: PLC คืออะไร และทำงานอย่างไร?

**PLC (Programmable Logic Controller)** คือคอมพิวเตอร์อุตสาหกรรมที่:

```
┌─────────────────────────────────────────────┐
│                   PLC                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Input   │  │  CPU     │  │  Output  │  │
│  │  Module  │→ │ (Program)│→ │  Module  │  │
│  │          │  │          │  │          │  │
│  │ I0.0-I0.7│  │ Execute  │  │ Q0.0-Q0.7│  │
│  │          │  │ Ladder   │  │          │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
      ↑ Sensors/Switches           ↓ Actuators/Valves/Motors
```

### วงจรการทำงาน PLC (Scan Cycle)
```
        ┌─────────────────────────────────────┐
        │          PLC SCAN CYCLE             │
        │                                     │
        │  1. READ INPUTS    ←──────────┐     │
        │       ↓                       │     │
        │  2. EXECUTE PROGRAM           │     │
        │       ↓                       │     │
        │  3. WRITE OUTPUTS             │     │
        │       ↓                       │     │
        │  4. HOUSEKEEPING ─────────────┘     │
        │                                     │
        │  Scan Time: 1-10 ms (typical)       │
        └─────────────────────────────────────┘
```

---

## Step 3: ส่วนประกอบของ PLC

### Hardware Components
```
PLC System Components:

1. POWER SUPPLY UNIT (PSU)
   ─────────────────────────
   - แปลงไฟ AC 220V → DC 24V
   - จ่ายไฟให้ CPU และ I/O Modules
   - มี Backup Battery สำหรับ Memory

2. CPU MODULE
   ─────────────────────────
   - ประมวลผล Ladder Program
   - มี RAM สำหรับ User Program
   - มี ROM สำหรับ OS
   - Communication Ports (RS-232, Ethernet)

3. INPUT MODULE (DI - Digital Input)
   ─────────────────────────
   - รับสัญญาณ ON/OFF จาก Sensor
   - รับไฟ DC 24V หรือ AC 110/220V
   - มี Optical Isolation ป้องกัน Noise
   - ขนาด: 8, 16, 32 Points

4. OUTPUT MODULE (DO - Digital Output)
   ─────────────────────────
   - ส่งสัญญาณ ON/OFF ไปยัง Actuator
   - ประเภท: Relay, Transistor, Triac
   - Relay: AC/DC ได้, Load สูง
   - Transistor: เร็ว, เฉพาะ DC

5. ANALOG INPUT MODULE (AI)
   ─────────────────────────
   - รับสัญญาณ Analog 4-20mA, 0-10V
   - แปลงเป็นตัวเลข 12-16 bit
   - ใช้กับ Sensor อุณหภูมิ, ความดัน

6. ANALOG OUTPUT MODULE (AO)
   ─────────────────────────
   - ส่งสัญญาณ Analog 4-20mA, 0-10V
   - ควบคุม VFD, Control Valve
```

---

## Step 4: ตัวแปรใน Ladder Diagram

### ประเภทของตัวแปร
```
VARIABLE TYPES IN LADDER DIAGRAM:

╔═══════════════╦═══════════╦══════════════════════════╗
║ ประเภท        ║ ตัวอย่าง  ║ คำอธิบาย                ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Digital Input ║ I0.0      ║ สัญญาณเข้า ON/OFF        ║
║               ║ %IX0.0    ║ (IEC 61131-3)            ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Digital Output║ Q0.0      ║ สัญญาณออก ON/OFF         ║
║               ║ %QX0.0    ║                          ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Memory Bit    ║ M0.0      ║ ตัวแปรภายใน ON/OFF       ║
║               ║ %MX0.0    ║                          ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Timer         ║ T0        ║ นับเวลา                  ║
║               ║ TON, TOF  ║                          ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Counter       ║ C0        ║ นับจำนวน                 ║
║               ║ CTU, CTD  ║                          ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Integer       ║ MW0       ║ ตัวเลข 16-bit Integer    ║
║               ║ %MW0      ║                          ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Double Int    ║ MD0       ║ ตัวเลข 32-bit Integer    ║
║               ║ %MD0      ║                          ║
╠═══════════════╬═══════════╬══════════════════════════╣
║ Real/Float    ║ MD4       ║ ตัวเลขทศนิยม             ║
║               ║ %MD4      ║                          ║
╚═══════════════╩═══════════╩══════════════════════════╝
```

### Address Formats ตามยี่ห้อ PLC
```
PLC ADDRESSING COMPARISON:

┌─────────────────┬──────────┬────────────┬─────────────┐
│ ประเภท          │ Siemens  │ Mitsubishi │ OMRON       │
│                 │ S7-1200  │ FX3U       │ CP1H        │
├─────────────────┼──────────┼────────────┼─────────────┤
│ Digital Input   │ I0.0     │ X0         │ 0.00        │
│ Digital Output  │ Q0.0     │ Y0         │ 100.00      │
│ Memory Bit      │ M0.0     │ M0         │ W0.00       │
│ Timer           │ DB1.TON1 │ T0         │ T0          │
│ Counter         │ DB1.CTU1 │ C0         │ C0          │
│ Data Word       │ MW0      │ D0         │ D0          │
└─────────────────┴──────────┴────────────┴─────────────┘
```

---

## Step 5: Symbols ที่ใช้ใน Ladder Diagram

### พื้นฐาน Symbols
```
LADDER DIAGRAM SYMBOLS:

1. NORMALLY OPEN (NO) CONTACT
   ──[ ]──
   ชื่อ: Contact (NO), Examine if Closed (XIC)
   เปิดวงจร: เมื่อ Coil ไม่มีไฟ (0)
   ปิดวงจร: เมื่อ Coil มีไฟ (1)

2. NORMALLY CLOSED (NC) CONTACT
   ──[/]──
   ชื่อ: Contact (NC), Examine if Open (XIO)
   เปิดวงจร: เมื่อ Coil มีไฟ (1)
   ปิดวงจร: เมื่อ Coil ไม่มีไฟ (0)

3. OUTPUT COIL
   ──( )──
   ชื่อ: Coil, Output Energize (OTE)
   ON: เมื่อ Rung มี Power Flow ถึง
   OFF: เมื่อ Rung ไม่มี Power Flow

4. SET COIL (Latch)
   ──(S)──
   ชื่อ: Set Coil, Latch
   ON: เมื่อ Rung มี Power Flow ถึง
   ไม่ OFF: ถ้าไม่มีคำสั่ง Reset

5. RESET COIL (Unlatch)
   ──(R)──
   ชื่อ: Reset Coil, Unlatch
   OFF: เมื่อ Rung มี Power Flow ถึง

6. POSITIVE EDGE (P)
   ──[P]──
   ชื่อ: Positive Edge Detect
   ON เพียง 1 Scan: เมื่อสัญญาณ 0→1

7. NEGATIVE EDGE (N)
   ──[N]──
   ชื่อ: Negative Edge Detect
   ON เพียง 1 Scan: เมื่อสัญญาณ 1→0
```

---

## Step 6: Rung คืออะไร?

**Rung (บรรทัด)** คือหนึ่งแถวของ Ladder Diagram

```
RUNG ANATOMY:
                                           
Left Rail    Contacts/Instructions    Right Rail
    │                                     │
    ├──[ I0.0 ]──[ I0.1 ]────( Q0.0 )───┤  ← Rung 1
    │                                     │
    ├──[ I0.2 ]──┬──[ I0.3 ]──┐          │
    │            └──[ I0.4 ]──┴──( Q0.1)─┤  ← Rung 2
    │                                     │
    ├──[/I0.5]──────────────── (SET Q0.2)─┤  ← Rung 3
    │                                     │
    ├──[ I0.6 ]──────────────(RESET Q0.2)─┤  ← Rung 4
    │                                     │

แต่ละ Rung คือ 1 วงจรอิสระ
Power Flow ไหลจากซ้ายไปขวา
```

### Rules ของ Rung
1. **Power Flow** ต้องไหลจาก Left Rail ไป Right Rail จึงจะ Execute Output
2. **Series** = AND Logic (ต่อแบบอนุกรม)
3. **Parallel** = OR Logic (ต่อแบบขนาน)
4. แต่ละ Rung Execute ทีละ Rung จากบนลงล่าง

---

## Step 7: AND, OR, NOT Logic ใน Ladder

### AND Logic (วงจรอนุกรม)
```
ตัวอย่าง: Q0.0 = I0.0 AND I0.1

    |──[ I0.0 ]──[ I0.1 ]──( Q0.0 )──|

Truth Table:
┌───────┬───────┬───────┐
│ I0.0  │ I0.1  │ Q0.0  │
├───────┼───────┼───────┤
│  OFF  │  OFF  │  OFF  │
│  OFF  │  ON   │  OFF  │
│  ON   │  OFF  │  OFF  │
│  ON   │  ON   │  ON   │ ← ต้องทั้งคู่ ON
└───────┴───────┴───────┘
```

### OR Logic (วงจรขนาน)
```
ตัวอย่าง: Q0.1 = I0.2 OR I0.3

    |──[ I0.2 ]──┐              |
    |            ├──( Q0.1 )──|
    |──[ I0.3 ]──┘              |

Truth Table:
┌───────┬───────┬───────┐
│ I0.2  │ I0.3  │ Q0.1  │
├───────┼───────┼───────┤
│  OFF  │  OFF  │  OFF  │
│  OFF  │  ON   │  ON   │ ← อย่างน้อยหนึ่งตัว ON
│  ON   │  OFF  │  ON   │ ← อย่างน้อยหนึ่งตัว ON
│  ON   │  ON   │  ON   │
└───────┴───────┴───────┘
```

### NOT Logic (Normally Closed)
```
ตัวอย่าง: Q0.2 = NOT I0.4

    |──[/I0.4]──( Q0.2 )──|

Truth Table:
┌───────┬───────┐
│ I0.4  │ Q0.2  │
├───────┼───────┤
│  OFF  │  ON   │ ← ตรงข้าม
│  ON   │  OFF  │ ← ตรงข้าม
└───────┴───────┘
```

---

## Step 8: โปรแกรมแรกของคุณ - Lamp Control

```
โจทย์: กดปุ่ม START (I0.0) ให้หลอดไฟ (Q0.0) ติด
       กดปุ่ม STOP (I0.1) ให้หลอดไฟ (Q0.0) ดับ
       หลอดไฟต้องค้างติดแม้ปล่อยปุ่ม START

SOLUTION - SEAL-IN CIRCUIT (Self-Holding):

Rung 1:
    |──[ I0.0  ]──┬──( Q0.0 )──|
    |  PB_START   │             |
    |             │             |
    |──[ Q0.0  ]──┘             |
    |  LAMP_FB                  |

Rung 2 (Stop):
    |──[/I0.1  ]──( Q0.0 )──|
    |   PB_STOP               |

```

> **หมายเหตุ:** วิธีนี้เรียกว่า **Seal-In** หรือ **Self-Holding Circuit** เป็นพื้นฐานสำคัญที่สุดใน PLC Programming

---

## Step 9: Scan Cycle และ Program Execution

### ลำดับการทำงานของ PLC
```
DETAILED SCAN CYCLE:

    ┌─────────────────────────────────────────────────┐
    │              START OF SCAN                      │
    ├─────────────────────────────────────────────────┤
    │  1. WATCHDOG TIMER RESET                        │
    │     - รีเซ็ต Timer ป้องกัน Infinite Loop        │
    ├─────────────────────────────────────────────────┤
    │  2. READ PHYSICAL INPUTS → INPUT IMAGE TABLE    │
    │     - อ่านค่า I/O ทั้งหมดเก็บใน Memory          │
    │     - ค่า Input ไม่เปลี่ยนระหว่าง Scan          │
    ├─────────────────────────────────────────────────┤
    │  3. EXECUTE USER PROGRAM                        │
    │     - Execute Rung 1 → Rung 2 → ... → Rung N   │
    │     - ใช้ค่าจาก Input Image Table               │
    │     - เขียนผลลัพธ์ใน Output Image Table         │
    ├─────────────────────────────────────────────────┤
    │  4. WRITE OUTPUT IMAGE TABLE → PHYSICAL OUTPUTS │
    │     - ส่งค่า Output ออกไปยัง Hardware           │
    ├─────────────────────────────────────────────────┤
    │  5. COMMUNICATION TASKS                         │
    │     - รับ/ส่งข้อมูล HMI, SCADA, Other PLC       │
    └─────────────────────────────────────────────────┘
         ↓ วนซ้ำตลอดเวลา (1-10ms per cycle)
```

### ทำไม Scan Time สำคัญ?
```
SCAN TIME IMPACT EXAMPLE:

สมมุติ Scan Time = 10ms

สัญญาณ Pulse กว้าง 5ms:
    Input:   ──────┐     ┌──────
                   └─────┘
                   |← 5ms→|

    PLC อ่าน:  ─────────────────  (อาจไม่เห็นเลย!)

สัญญาณ Pulse กว้าง 15ms:
    Input:   ──────┐         ┌──────
                   └─────────┘
                   |← 15ms →|

    PLC อ่าน:  ────────ON────────  (เห็นแน่นอน)

→ กฎ: สัญญาณต้องกว้างอย่างน้อย 1.5x Scan Time
→ ใช้ High-Speed Counter สำหรับสัญญาณความถี่สูง
```

---

## Step 10: IEC 61131-3 Standard

### ภาษาโปรแกรมมาตรฐาน IEC 61131-3
```
IEC 61131-3 PROGRAMMING LANGUAGES:

┌─────────────────────────────────────────────────┐
│                IEC 61131-3                      │
│                                                 │
│  Graphical:                Textual:             │
│  ┌──────────────┐          ┌──────────────────┐ │
│  │ Ladder       │          │ Instruction List │ │
│  │ Diagram (LD) │          │ (IL) - deprecated│ │
│  └──────────────┘          └──────────────────┘ │
│  ┌──────────────┐          ┌──────────────────┐ │
│  │ Function     │          │ Structured Text  │ │
│  │ Block        │          │ (ST)             │ │
│  │ Diagram (FBD)│          └──────────────────┘ │
│  └──────────────┘                               │
│  ┌──────────────┐                               │
│  │ Sequential   │                               │
│  │ Function     │                               │
│  │ Chart (SFC)  │                               │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘

→ Ladder Diagram เป็นภาษาที่นิยมมากที่สุดในงานอุตสาหกรรม
→ เหมาะกับ Discrete Control (ON/OFF)
→ ช่างไฟเข้าใจได้ง่ายที่สุด
```

---

## Step 11: เปรียบเทียบ PLC กับ Relay Panel

```
RELAY PANEL vs PLC:

TRADITIONAL RELAY PANEL:
┌────────────────────────────────────────┐
│  230V AC ──┬── Relay1 ──┬── Motor     │
│            │   CR1      │             │
│            ├── Relay2 ──┘             │
│            │   CR2                    │
│  Hardwired ← ต้องเดินสายใหม่ถ้าแก้ไข  │
└────────────────────────────────────────┘

PLC SYSTEM:
┌────────────────────────────────────────┐
│  24V DC ──── PLC ──────── Motor       │
│  I0.0 ←─ PB1                          │
│  I0.1 ←─ PB2                          │
│  Q0.0 ──→ Motor Contactor             │
│                                       │
│  แก้ Logic = แก้ Program เท่านั้น!   │
└────────────────────────────────────────┘

COMPARISON:
┌───────────────┬──────────────┬──────────────┐
│ หัวข้อ        │ Relay Panel  │ PLC          │
├───────────────┼──────────────┼──────────────┤
│ แก้ไข Logic  │ เดินสายใหม่  │ แก้ Program  │
│ ความน่าเชื่อ  │ สูง (แต่ช้า) │ สูงมาก       │
│ ความเร็ว     │ ~10ms        │ 1-5ms        │
│ Space        │ ใหญ่         │ เล็ก         │
│ ราคา         │ ถูก(น้อย I/O)│ คุ้มค่า      │
│ Maintenance  │ ยาก          │ ง่าย         │
│ Documentation│ Manual       │ Auto (Software)|
└───────────────┴──────────────┴──────────────┘
```

---

## Step 12: PLC ยี่ห้อหลักในตลาด

```
MAJOR PLC MANUFACTURERS:

TIER 1 (Enterprise Level):
┌───────────────────────────────────────────────┐
│ SIEMENS (Germany)                             │
│ - S7-1200: Mid-range, TIA Portal              │
│ - S7-1500: High-end, TIA Portal               │
│ - ET 200SP: Distributed I/O                   │
│ Market Share: ~30%                            │
├───────────────────────────────────────────────┤
│ ROCKWELL/ALLEN-BRADLEY (USA)                  │
│ - MicroLogix 1100/1400: Small                 │
│ - CompactLogix L1x-L5x: Mid-range             │
│ - ControlLogix: High-end                      │
│ - Studio 5000 Software                        │
│ Market Share: ~25%                            │
└───────────────────────────────────────────────┘

TIER 2 (Asia-Pacific):
┌───────────────────────────────────────────────┐
│ MITSUBISHI (Japan)                            │
│ - FX5U: Small-Mid, GX Works3                  │
│ - Q Series: High-end                          │
│ - iQ-R Series: Latest Platform                │
├───────────────────────────────────────────────┤
│ OMRON (Japan)                                 │
│ - CP1H/CP2E: Small-Mid, CX-Programmer         │
│ - CJ2/CS1: High-end                          │
│ - NX/NY Series: Latest Platform               │
├───────────────────────────────────────────────┤
│ KEYENCE (Japan)                               │
│ - KV-8000: Fast, Easy programming             │
│ - KV-5500: Mid-range                          │
└───────────────────────────────────────────────┘

TIER 3 (Cost-effective):
┌───────────────────────────────────────────────┐
│ DELTA (Taiwan)                                │
│ - DVP Series, AS Series                       │
├───────────────────────────────────────────────┤
│ SCHNEIDER (France)                            │
│ - Modicon M221/M241/M262                      │
├───────────────────────────────────────────────┤
│ BECKHOFF (Germany)                            │
│ - PC-based PLC, TwinCAT                       │
└───────────────────────────────────────────────┘
```

---

## Step 13: Software ที่ใช้เขียน Ladder Diagram

```
PROGRAMMING SOFTWARE BY MANUFACTURER:

SIEMENS:
  TIA Portal (Totally Integrated Automation Portal)
  - รุ่น: V16, V17, V18, V19 (ล่าสุด 2024)
  - ราคา: License ~$3,000-$10,000
  - ฟรีทดลอง: TIA Portal Trial Version (21 วัน)
  - Download: support.industry.siemens.com

ALLEN-BRADLEY:
  Studio 5000 Logix Designer
  - รุ่น: V32, V33, V34, V35
  - ราคา: License ~$2,000-$8,000
  - ฟรีทดลอง: Studio 5000 Lite (Limited)
  - Download: rockwellautomation.com

MITSUBISHI:
  GX Works3 (FX5, iQ-R)
  GX Works2 (FX3, Q Series)
  - ราคา: License ~$1,000-$3,000
  - ฟรีทดลอง: Trial 30 วัน
  - Download: mitsubishifa.com

OMRON:
  CX-Programmer (CJ, CS, CP Series)
  Sysmac Studio (NX/NY Series)
  - ราคา: License ~$500-$2,000
  - ฟรีทดลอง: CX-Programmer 30 วัน

FREE / OPEN SOURCE:
  CODESYS Development System
  - รองรับ IEC 61131-3 ทุกภาษา
  - ใช้กับ PLC หลายยี่ห้อ
  - Download: codesys.com (ฟรี)

  OpenPLC
  - Open Source PLC
  - ทำงานบน Raspberry Pi, PC
  - Download: openplcproject.com (ฟรี)

  Factory I/O
  - 3D Factory Simulator
  - ทดสอบโปรแกรมกับโรงงาน 3D
  - ราคา ~$300/year
```

---

## Step 14: ขั้นตอนการพัฒนาโปรแกรม PLC

```
PLC PROGRAMMING WORKFLOW:

1. REQUIREMENTS ANALYSIS
   ─────────────────────────
   - เข้าใจกระบวนการที่ต้องการควบคุม
   - รวบรวม I/O List
   - กำหนด Safety Requirements
   
2. HARDWARE DESIGN
   ─────────────────────────
   - เลือก PLC Model ที่เหมาะสม
   - ออกแบบ Control Panel
   - เขียน Electrical Drawing
   
3. I/O MAPPING
   ─────────────────────────
   - กำหนด Address ของ I/O ทุกจุด
   - สร้าง I/O Tag Database
   - ตรวจสอบ Safety I/O
   
4. PROGRAM DESIGN
   ─────────────────────────
   - เขียน Flowchart / State Diagram
   - แบ่ง Program เป็น Sub-routine
   - วางแผน Error Handling
   
5. LADDER CODING
   ─────────────────────────
   - เขียน Ladder Diagram ตาม Design
   - Comment ทุก Rung
   - ทำตาม Coding Standard

6. SIMULATION & TESTING
   ─────────────────────────
   - Test ด้วย Software Simulator
   - SAT (Site Acceptance Test)
   - FAT (Factory Acceptance Test)
   
7. COMMISSIONING
   ─────────────────────────
   - Download Program ลง PLC จริง
   - Test กับ Hardware จริง
   - Document การแก้ไขทั้งหมด
   
8. DOCUMENTATION
   ─────────────────────────
   - สร้าง As-Built Drawing
   - เขียน Operation Manual
   - สร้าง Maintenance Manual
```

---

## Step 15: Project แรก - Traffic Light Control

### โจทย์ Lab 01
```
TRAFFIC LIGHT CONTROL (ไฟจราจร)

ความต้องการ:
- ไฟเขียว (Q0.0): ติด 5 วินาที
- ไฟเหลือง (Q0.1): ติด 2 วินาที  
- ไฟแดง (Q0.2): ติด 5 วินาที
- วนซ้ำตลอดเวลา

I/O List:
┌────────────────────────────────────────────┐
│ Tag Name    │ Address │ Description        │
├────────────────────────────────────────────┤
│ GREEN_LIGHT │ Q0.0    │ ไฟสัญญาณเขียว      │
│ YELLOW_LIGHT│ Q0.1    │ ไฟสัญญาณเหลือง     │
│ RED_LIGHT   │ Q0.2    │ ไฟสัญญาณแดง        │
│ START_PB    │ I0.0    │ ปุ่ม Start ระบบ    │
└────────────────────────────────────────────┘

State Diagram:
         ┌──────────────────────────────┐
         ↓                              │
    [GREEN 5s] → [YELLOW 2s] → [RED 5s]─┘

Ladder Program (จะเรียนรายละเอียดใน Part 04-Timer):

Rung 1: State Machine Bit S0 (Green Phase)
    |──[ S0 ]────────────( Q0.0 GREEN )──|
    |──[ S0 ]──[T0/DN]──(RESET S0)──|
    |──[ S0 ]──[T0/DN]──(SET S1)────|
    
Rung 2: State Machine Bit S1 (Yellow Phase)    
    |──[ S1 ]────────────( Q0.1 YELLOW)──|
    |──[ S1 ]──[T1/DN]──(RESET S1)──|
    |──[ S1 ]──[T1/DN]──(SET S2)────|

Rung 3: State Machine Bit S2 (Red Phase)
    |──[ S2 ]────────────( Q0.2 RED )──|
    |──[ S2 ]──[T2/DN]──(RESET S2)──|
    |──[ S2 ]──[T2/DN]──(SET S0)────|
```

### สรุป Part 01
```
KEY TAKEAWAYS FROM PART 01:

✅ Ladder Diagram คือภาษา PLC มาตรฐานสากล
✅ PLC ทำงานแบบ Scan Cycle ซ้ำๆ
✅ Contacts เป็น Input Logic (NO/NC)
✅ Coils เป็น Output Logic
✅ Series = AND, Parallel = OR
✅ Power Flow จากซ้ายไปขวา
✅ Seal-In Circuit เป็นพื้นฐานสำคัญ
✅ IEC 61131-3 เป็นมาตรฐานสากล

⏭️ Part 02 จะเรียนรู้: Contact และ Coil ในรายละเอียด
```

---

## แบบฝึกหัด Part 01

**ข้อ 1:** วาด Ladder Diagram สำหรับ:
- กดปุ่ม A (I0.0) หรือ ปุ่ม B (I0.1) ให้หลอดไฟ L1 (Q0.0) ติด
- กดปุ่ม STOP (I0.2) ให้ L1 ดับ

**ข้อ 2:** อธิบาย Scan Cycle ของ PLC พร้อมระบุว่า Input อ่านเมื่อไหร่

**ข้อ 3:** เปรียบเทียบ PLC กับ Relay Panel ในด้าน: ราคา, ความยืดหยุ่น, Maintenance

**ข้อ 4:** ค้นหา PLC ที่มีใช้ในโรงงาน/อุตสาหกรรมในพื้นที่คุณ ว่าใช้ยี่ห้ออะไร

---

*[ต่อไป Part 02: Contact และ Coil พื้นฐาน →]*
