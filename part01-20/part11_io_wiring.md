# Part 11: การต่อวงจร I/O (Steps 191-210)

## ภาพรวมของบทเรียน

ในบทนี้เราจะเรียนรู้การต่อวงจร Input/Output ของ PLC อย่างถูกต้องและปลอดภัย ซึ่งเป็นทักษะพื้นฐานที่สำคัญมากสำหรับวิศวกรและช่างไฟฟ้าอุตสาหกรรม การต่อวงจรที่ผิดพลาดอาจทำให้ PLC เสียหายหรือเกิดอุบัติเหตุได้

---

## Step 191: พื้นฐานระบบไฟฟ้า 24VDC สำหรับ PLC

### ทำไมต้องใช้ 24VDC?

ระบบ PLC สมัยใหม่ส่วนใหญ่ใช้ไฟเลี้ยง 24VDC เพราะ:
- ปลอดภัยกว่า 220VAC
- ทนต่อสัญญาณรบกวนได้ดี
- มาตรฐานอุตสาหกรรมทั่วโลก (IEC 61131)
- อุปกรณ์เซ็นเซอร์ส่วนใหญ่ใช้ 24VDC

### โครงสร้างระบบไฟเลี้ยง 24VDC

```
220VAC         Power Supply          24VDC System
   ┌──────┐    ┌──────────────┐    ┌──────────────────┐
   │      │    │   Switching  │    │  PLC CPU         │
L─►│  MCB ├───►│   Power      ├───►│  Input Module    │
N─►│      │    │   Supply     │    │  Output Module   │
   └──────┘    │  24VDC/10A   │    │  Sensors         │
               └──────────────┘    └──────────────────┘
                     │
               ┌─────┴──────┐
               │  24V  │  0V │
               │  (+)  │ (-) │
               └───────┴─────┘
```

### การคำนวณขนาด Power Supply

**สูตรคำนวณกำลังไฟ:**

```
กำลังไฟที่ต้องการ = (จำนวน Input × กระแสต่อ Input) + (จำนวน Output × กระแสต่อ Output) + กำลังไฟ CPU

ตัวอย่าง:
- CPU: 500mA
- Input 16 ช่อง × 7mA = 112mA  
- Output 16 ช่อง × 100mA = 1600mA
- เซ็นเซอร์ 8 ตัว × 50mA = 400mA
รวม = 2612mA ≈ 2.6A

เลือก Power Supply 24V/5A (เผื่อ 30%)
```

---

## Step 192: การต่อวงจร Input 24VDC

### ประเภทของ Input Module

1. **Sourcing Input (PNP)** - รับกระแสจากอุปกรณ์ภายนอก
2. **Sinking Input (NPN)** - จ่ายกระแสให้อุปกรณ์ภายนอก

### วงจร Input พื้นฐาน

```
แบบ Sinking (NPN - Source Internal)

+24V ──────────────────────────────────────────┐
                                                │
         ┌──────────────────────────────┐       │
         │      Input Module             │       │
+24V ────┤ L+                           │       │
         │                               │       │
         │  I0.0 ──[Internal LED]──┐    │       │
         │  I0.1 ──[Internal LED]──┤    │       │
         │  I0.2 ──[Internal LED]──┤    │       │
         │  ...                    │    │       │
         │                         │    │       │
         │ M (COM) ─────────────── │    │       │
         └─────────────────────────┼────┘       │
                                   │            │
0V ────────────────────────────────┘────────────┘
```

### การต่อ Push Button (NO Contact)

```
+24V
  │
  ├──────[PB1 NO]────────► I0.0 (PLC Input)
  │
  ├──────[PB2 NO]────────► I0.1
  │
  └──────[LS1 NO]────────► I0.2 (Limit Switch)

COM ──────────────────────► M (Input Common)
  │
 0V
```

**ตัวอย่าง Wiring Diagram จริง:**

```
+24V ──┬──────────────────────────────────────────┐
       │                                          │
       │  ┌─────────────────────────────────┐    │
       │  │     Siemens S7-1200              │    │
       │  │                                  │    │
       ├──┤ 1L+                              │    │
       │  │                                  │    │
       ├──┤ I0.0 ← PB-START (ปุ่มสตาร์ท)   │    │
       ├──┤ I0.1 ← PB-STOP  (ปุ่มหยุด)     │    │
       ├──┤ I0.2 ← LS-1     (ลิมิตสวิตช์)  │    │
       ├──┤ I0.3 ← LS-2                     │    │
       ├──┤ I0.4 ← OL-1     (Overload)      │    │
       │  │                                  │    │
       │  │ 1M ────────────────────────────────┤
       │  └─────────────────────────────────┘  │
       │                                        │
      0V ──────────────────────────────────────┘
```

---

## Step 193: NPN vs PNP Sensor Wiring

### ทำความเข้าใจ NPN และ PNP

**NPN Sensor (Sinking)**
- Output ต่อลง 0V เมื่อ Detect
- เรียกว่า "Current Sinking"
- สัญลักษณ์: ลูกศรชี้เข้า

**PNP Sensor (Sourcing)**  
- Output ต่อขึ้น +24V เมื่อ Detect
- เรียกว่า "Current Sourcing"
- สัญลักษณ์: ลูกศรชี้ออก

### วงจร NPN Sensor กับ PLC

```
NPN Sensor ต่อกับ PLC (Sinking Input)

+24V ──────────────────────────────────────────┐
  │                                            │
  │    ┌─────────────────────┐                │
  │    │   NPN Proximity     │                │
  ├───►│ BRN (+24V)          │                │
  │    │                     │                │
  │    │         ───Object───│                │
  │    │                     │         ┌──────┴──────┐
  │    │ BLK (Output) ───────┼────────►│ I0.0        │
  │    │                     │         │   PLC Input │
  │    │ BLU (0V) ───────────┼──┐      └─────────────┘
  │    └─────────────────────┘  │
  │                             │
 0V ────────────────────────────┘

กระแสไหล: +24V → Sensor → 0V (NPN Sinking)
```

### วงจร PNP Sensor กับ PLC

```
PNP Sensor ต่อกับ PLC (Sourcing Input)

+24V ─────────────────────────────────────────┐
  │                                           │
  │    ┌─────────────────────┐               │
  ├───►│ BRN (+24V)          │               │
  │    │   PNP Proximity     │               │
  │    │                     │               │
  │    │         ───Object───│               │
  │    │                     │        ┌──────┴──────┐
  │    │ BLK (Output) ───────┼───────►│ I0.0        │
  │    │                     │        │   PLC Input │
  │    │ BLU (0V) ──────────────────► │ COM (1M)    │
  │    └─────────────────────┘        └─────────────┘
  │
 0V ──────────────────────────────────────────
```

### ตารางเปรียบเทียบ NPN/PNP

| คุณสมบัติ | NPN (Sinking) | PNP (Sourcing) |
|-----------|---------------|----------------|
| Output State | ต่อลง 0V | ต่อขึ้น +24V |
| ใช้กับ PLC | Sinking Input | Sourcing Input |
| ความนิยมในไทย | สูง (Siemens, Mitsubishi) | ปานกลาง (A-B) |
| ราคา | ถูกกว่าเล็กน้อย | แพงกว่าเล็กน้อย |
| ทนสัญญาณรบกวน | ดี | ดีมาก |

### การแยกประเภท Sensor จากสีสาย

```
มาตรฐาน IEC 60947-5-2:

BRN (น้ำตาล)   = +24VDC (Supply+)
BLU (น้ำเงิน)  = 0VDC   (Supply-)
BLK (ดำ)       = Signal Output
WHT (ขาว)      = Signal Output 2 (4-wire sensor)
GRY (เทา)      = Teach/Enable
```

---

## Step 194: การต่อวงจร Output - Relay, Transistor, Triac

### ประเภทของ Output Module

#### 1. Relay Output

```
┌─────────────────────────────────────────────┐
│              RELAY OUTPUT MODULE             │
│                                              │
│  Q0.0 ──[RELAY COIL]──┐                    │
│                         │                    │
│         ┌───────────────┘                    │
│         │  COM ──●                           │
│         │        │                           │
│         └─►  NO ──●──────────────► LOAD+    │
│              NC ──●                          │
│                                              │
│  External Power: ≤250VAC or 30VDC           │
│  Max Current: 2A per point                  │
└─────────────────────────────────────────────┘
```

**ข้อดี Relay Output:**
- ต่อกับ AC หรือ DC ได้
- ทนกระแสสูง (2-5A)
- แยก Isolation ดี
- ไม่จำกัดประเภทโหลด

**ข้อเสีย:**
- ตอบสนองช้า (8-10ms)
- มีอายุการใช้งานจำกัด (10^7 ครั้ง)
- มีเสียงดัง

#### 2. Transistor Output (DC Output)

```
┌─────────────────────────────────────────────┐
│           TRANSISTOR OUTPUT MODULE           │
│                                              │
│           ┌──── L+ (24VDC)                  │
│           │                                  │
│  Q0.0 ──[Transistor]──────────────► LOAD+  │
│                 │                            │
│                 └──────────────────► LOAD-  │
│                                              │
│  M (COM) ──────────────────────────► 0V     │
│                                              │
│  Max Current: 500mA per point               │
│  Response: < 0.5ms                          │
└─────────────────────────────────────────────┘
```

**ข้อดี Transistor Output:**
- ตอบสนองเร็ว (<0.5ms)
- อายุยืนยาว (ไม่มีชิ้นส่วนเคลื่อนไหว)
- เงียบ
- เหมาะกับ High-speed output

**ข้อเสีย:**
- ใช้ได้กับ DC เท่านั้น
- กระแสต่ำ (500mA-2A)
- ต้องระวัง Inductive loads

#### 3. Triac Output (AC Output)

```
┌─────────────────────────────────────────────┐
│              TRIAC OUTPUT MODULE             │
│                                              │
│  Q0.0 ──[TRIAC]────────────────────► LINE  │
│                                     NEUTRAL │
│                                              │
│  AC Supply: 85-264VAC 50/60Hz              │
│  Max Current: 1A per point                  │
│  Zero-crossing switching                    │
└─────────────────────────────────────────────┘
```

### ตารางเปรียบเทียบ Output Types

| คุณสมบัติ | Relay | Transistor | Triac |
|-----------|-------|------------|-------|
| โหลด AC | ✓ | ✗ | ✓ |
| โหลด DC | ✓ | ✓ | ✗ |
| กระแสสูงสุด | 2-5A | 0.5-2A | 0.5-1A |
| ความเร็ว | ช้า | เร็ว | ปานกลาง |
| อายุการใช้งาน | จำกัด | ยาว | ยาว |
| ราคา | ถูก | ปานกลาง | ปานกลาง |

---

## Step 195: การต่อ Output กับโหลดต่างๆ

### การต่อ Relay Output กับ Contactor

```
PLC Output → Contactor Coil → Motor Starter

     PLC                    Panel
  ┌───────┐              ┌─────────────────┐
  │       │              │                 │
  │ Q0.0 ─┼──────────────┤ KM1 COIL (+)   │
  │       │              │                 │
  │  COM ─┼──────────────┤ KM1 COIL (-)   │
  │       │              │         │       │
  └───────┘              │    24VDC│       │
                         └─────────│───────┘
                                   │
                              ┌────┴────┐
                              │  24VDC  │
                              │  PSU    │
                              └─────────┘
```

### การป้องกัน Inductive Load (Flyback Diode)

```
ต้องใส่ Diode ป้องกัน Back-EMF สำหรับโหลด Inductive (Relay, Solenoid, Motor)

         ┌──── +24V
         │
    ┌────┴────┐
    │         │
    │  Relay  │         Flyback Diode (ต้องใส่!)
    │  Coil   │    ┌──────────┤◄├──────────┐
    │         │    │          │            │
    └────┬────┘    │          │            │
         │         │          │            │
PLC Q0.0─┼─────────┘          │            │
         │                    │            │
        0V──────────────────────────────────

ถ้าไม่ใส่ Diode: Back-EMF อาจสูงถึง 100-200V ทำลาย Output Transistor!
```

### Suppress Circuit สำหรับ AC Loads

```
สำหรับ AC Inductive Loads ใช้ RC Snubber แทน Diode

         AC LINE
            │
         ┌──┴──┐
         │Triac│ PLC Output
         └──┬──┘
            │
      ┌─────┴─────┐
      │   MOTOR   │
      │  OR VALVE │
      └─────┬─────┘
            │
     RC Snubber ──┐
     (100Ω+0.1µF) │
            │     │
         NEUTRAL──┘
```

---

## Step 196: Grounding และ Shielding

### ความสำคัญของ Grounding

การ Ground ที่ถูกต้องช่วย:
1. ความปลอดภัยจากไฟฟ้าช็อต
2. ลดสัญญาณรบกวน EMI/RFI
3. ป้องกัน Static Discharge
4. ทำให้ระบบทำงานเสถียร

### โครงสร้าง Grounding System

```
         PE Ground Bar (กราวด์หลัก)
                │
    ┌───────────┼───────────┐
    │           │           │
    │    ┌──────┴──────┐    │
    │    │             │    │
 Cabinet│  Control     │  Power
  Frame │  Panel PE    │  Ground
    │    │  Bar        │    │
    │    └─────────────┘    │
    │           │           │
    └─────────Terminal──────┘
                │
         Earth Ground Rod
         (กราวด์แท่ง)
          Resistance < 5Ω
```

### Single Point Grounding

```
สำคัญมาก! ต้อง Ground ที่จุดเดียว ไม่งั้นจะเกิด Ground Loop

                   ┌─── Panel Frame
                   │
    ┌──────────────┴──────────────────┐
    │                                 │
    │    PLC Ground ─────────────────►│ Single Ground Point
    │    PSU Ground ─────────────────►│
    │    Shield Ground ──────────────►│
    │    PE Conductor ──────────────► │
    │                                 │
    └─────────────────────────────────┘
                          │
                   Earth Ground
```

### Shielding สำหรับสายสัญญาณ

```
การต่อ Shield ที่ถูกต้อง:

Controller Side          Field Side
    │                        │
    │  ┌────────────────┐    │
    │  │  Signal Wire   │    │
    ├──┤ ╔════════════╗ ├────┤ Sensor
    │  │ ║  SHIELD    ║ │    │
    │  │ ╚════════════╝ │    │
    │  └────────────────┘    │
    │         │              │
    └─────────┘         OPEN
    (Ground Shield         (ไม่ต่อ
     เฉพาะด้านเดียว)       ด้านนี้)

กฎ: Shield ต้องต่อ Ground เพียง 1 จุดเท่านั้น!
```

### ประเภทสายสัญญาณ

```
1. Shielded Twisted Pair (STP) - สำหรับ Analog Signal
   ┌─────────────────────────────────────┐
   │  ╔═══╗   ╔═══╗                     │
   │  ║ + ║   ║ - ║  ← Twisted wires    │
   │  ╚═══╝   ╚═══╝                     │
   │  ════════════════  ← Shield (Foil) │
   └─────────────────────────────────────┘

2. Shielded Multi-core - สำหรับ Digital Signal
   สายขาว/แดง = Signal
   สายดำ = Common
   ชีลด์ = Ground หนึ่งปลาย
```

---

## Step 197: Cable Routing และ Installation

### กฎการเดินสาย

**กฎสำคัญ #1: แยกสายไฟฟ้าแรงสูงและสายสัญญาณ**

```
Panel Layout:

┌────────────────────────────────────────────────────┐
│                    Control Panel                    │
│                                                     │
│  ╔═══════════╗         ╔══════════════════╗        │
│  ║  220V AC  ║         ║   24V DC / PLC   ║        │
│  ║  Wiring   ║         ║   Signal Wiring  ║        │
│  ║  (Zone A) ║         ║   (Zone B)       ║        │
│  ╚═══════════╝         ╚══════════════════╝        │
│       │                         │                   │
│       │←─── ≥ 150mm ──────────►│                   │
│       │    Minimum Separation   │                   │
└────────────────────────────────────────────────────┘
```

**กฎสำคัญ #2: ไม่วางสายขนานกัน ให้ตัดกันเป็นมุม 90°**

```
สายแรงดันสูง (แนวนอน)
═══════════════════════════════════════

              ║  ← สายสัญญาณ (แนวตั้ง)
              ║
              ║  ✓ ตัดกัน 90° (ถูกต้อง)

═══════════════════════════════════════
═══════════════════════════════════════
║  ← ขนานกัน (ผิด! จะเกิด Induction)
═══════════════════════════════════════
```

### การจัดสายใน Cable Tray

```
Cable Tray Cross-section:

┌──────────────────────────────────────────────────┐
│                    Cable Tray                     │
│                                                   │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐         │
│  │Power │  │Motor │  │  DC  │  │Analog│         │
│  │Cables│  │Cables│  │Digital│  │Signal│         │
│  │(220V+│  │(High │  │(PLC  │  │(4-20 │         │
│  │ 480V)│  │Current│  │I/O)  │  │ mA)  │         │
│  └──────┘  └──────┘  └──────┘  └──────┘         │
│                                                   │
│  Zone 1    Zone 2    Zone 3    Zone 4            │
│  (Power)  (Motor)  (Digital) (Analog)            │
└──────────────────────────────────────────────────┘
```

### ระยะห่างสายตามมาตรฐาน

| ประเภทสาย | ห่างจากสาย Power |
|-----------|-----------------|
| Analog Signal | ≥ 300mm |
| Digital Signal | ≥ 150mm |
| Communication | ≥ 150mm |
| Sensor Cable | ≥ 100mm |

---

## Step 198: I/O Terminal Block และ Wiring Practices

### ประเภท Terminal Block

```
1. Spring Clamp Terminal (เร็ว ไม่ต้อง Retighten)
   ┌─────────────────────┐
   │  ←Push button       │
   │  ┌──┐               │
   │  │  │  [Spring]     │
   │  │  ├──────────wire─┤
   │  └──┘               │
   └─────────────────────┘

2. Screw Terminal (แข็งแรง ต้อง Torque ถูกต้อง)
   ┌─────────────────────┐
   │    Screw →[○]       │
   │              ╔════╗ │
   │              ║wire║ │
   │              ╚════╝ │
   └─────────────────────┘

3. Knife Disconnect Terminal (ตัดวงจรได้)
   ┌─────────────────────┐
   │   [Knife Handle]    │
   │        │            │
   │    ────┼────         │
   │        │            │
   └─────────────────────┘
```

### Ferrule (ปลอกสายไฟ)

**ต้องใส่ Ferrule ทุกครั้งที่ต่อสายเข้า Terminal!**

```
สาย → ปลอก Ferrule → Terminal Block

ก่อนใส่ Ferrule:       หลังใส่ Ferrule:
  ===\\                   ┌───┐
  ===//  ← ลวดหลุด       │   │─ ลวดไม่หลุด
  ===\\                   │   │
                          └───┘
                          Ferrule
```

**ขนาด Ferrule ตามขนาดสาย:**

| ขนาดสาย | ขนาด Ferrule | สี |
|---------|-------------|-----|
| 0.5mm² | 0.5mm² | ขาว |
| 0.75mm² | 0.75mm² | เทา |
| 1.0mm² | 1.0mm² | แดง |
| 1.5mm² | 1.5mm² | ดำ |
| 2.5mm² | 2.5mm² | น้ำเงิน |
| 4.0mm² | 4.0mm² | เทาเข้ม |

### การ Mark สาย (Wire Marking)

```
มาตรฐานการ Mark สาย:

ตัวอย่าง: I0.0-PB-START

I0.0    = PLC Address
PB      = Push Button  
START   = Function

สายทุกเส้นต้องมี:
┌──────────────────────────────────────────────┐
│ From: Panel Terminal → To: Field Device      │
│                                              │
│  TB-I0.0 ────[Marker]─────────────► PB-START│
│                                              │
│  Marker: "I0.0" หรือ "START-PB"            │
└──────────────────────────────────────────────┘
```

---

## Step 199: Common Wiring Mistakes และวิธีป้องกัน

### ข้อผิดพลาดที่พบบ่อย

#### ผิดพลาด #1: ต่อ COM ผิด

```
ผิด:                        ถูก:
+24V ────► I0.0             +24V ────► I0.0
+24V ────► I0.1             +24V ────► I0.1
                             
  0V ─────► COM               0V ────► COM
  
แต่ดึง COM ขึ้น +24V          COM ต้องเป็น 0V เสมอ
(ทำให้ input ติดตลอด)        สำหรับ NPN/Sinking Input
```

#### ผิดพลาด #2: ไม่ใส่ Fuse หรือ Protection

```
ผิด:
PLC Q0.0 ────────────────────────────────► CONTACTOR

ถูก:
PLC Q0.0 ──[Fuse 1A]──[Snubber]──────────► CONTACTOR
```

#### ผิดพลาด #3: Ground Loop

```
ผิด (Ground Loop):
    PSU1                    PSU2
     │                       │
     GND                    GND
      │                       │
      └──────────────────────┘
               │
         เกิด Current Loop
         รบกวนสัญญาณ

ถูก (Single Point Ground):
    PSU1   PSU2   PLC   Sensor
     │      │      │      │
     └──────┴──────┴──────┘
                   │
            Single Ground Point
```

#### ผิดพลาด #4: ขนาดสายไม่พอ

```
ตาราง Minimum Wire Size:

| กระแส | ขนาดสาย Min |
|-------|-------------|
| < 1A  | 0.5mm²     |
| 1-5A  | 1.0mm²     |
| 5-10A | 1.5mm²     |
| 10-16A| 2.5mm²     |
| 16-25A| 4.0mm²     |
| 25-32A| 6.0mm²     |
```

#### ผิดพลาด #5: ไม่ป้องกัน Moisture

```
Outdoor Junction Box:
┌──────────────────────────────────────────┐
│   ✓ ใช้กล่อง IP65 ขึ้นไป               │
│   ✓ ต่อสาย Cable Gland                 │
│   ✓ ใส่ Silica Gel                     │
│   ✓ Seal รอยต่อ                        │
│   ✗ ไม่ใช้กล่องพลาสติกธรรมดา          │
│   ✗ ไม่เปิดรูสาย แต่ไม่ใส่ Gland     │
└──────────────────────────────────────────┘
```

---

## Step 200: การทดสอบและ Commissioning I/O

### ขั้นตอนการ Test I/O

```
1. Visual Inspection (ตรวจสายตา)
   ☐ ตรวจการต่อสายทุกจุด
   ☐ ตรวจ Ferrule และ Terminal torque
   ☐ ตรวจ Label/Marking ถูกต้อง
   ☐ ตรวจ Cable routing

2. Isolation Test (วัดค่า Insulation)
   ☐ วัด Insulation Resistance > 1MΩ
   ☐ ทดสอบแต่ละ Circuit แยกกัน

3. Point-to-Point Test (ทดสอบทีละจุด)
   ☐ Energize ครั้งละ Circuit
   ☐ ตรวจสอบ PLC I/O Status
   ☐ บันทึกผลการทดสอบ

4. Functional Test (ทดสอบการทำงาน)
   ☐ ทดสอบ Input ทุกจุด
   ☐ ทดสอบ Output ทุกจุด
   ☐ ทดสอบ Emergency Stop
```

### PLC I/O Test Program (Siemens TIA Portal)

```ladder
// Program สำหรับทดสอบ I/O แบบ Force
// Network 1: Force Output สำหรับ Test

|         |
| "Force" |──────────────────────────────( Q0.0 )
|         |
|         |
| %I0.0   |──────────────────────────────( Q0.1 )
|   PB1   |   Mirror Input to Output
|         |
```

**Siemens TIA Portal Address:**
```
Input:  %I0.0, %I0.1 ... %IB0, %IW0
Output: %Q0.0, %Q0.1 ... %QB0, %QW0
```

**Allen-Bradley Address:**
```
Input:  I:0/0, I:0/1 ... I:0.0, I:0.1
Output: O:0/0, O:0/1 ... O:0.0, O:0.1
```

**Mitsubishi Address:**
```
Input:  X0, X1, X2... X7, X10 (Octal)
Output: Y0, Y1, Y2... Y7, Y10 (Octal)
```

---

## Step 201: การต่อ Analog Input

### ประเภท Analog Signal

```
1. Current Signal: 4-20mA (ที่นิยมที่สุด)
   0%  = 4mA
   100%= 20mA
   Live Zero: ถ้าสาย Break = 0mA (Error Detection)

2. Voltage Signal: 0-10V หรือ ±10V
   0%  = 0V
   100%= 10V

3. Thermocouple (TC): mV Signal
   Type K: -200°C ~ +1372°C

4. RTD (PT100/PT1000): Resistance Signal
   0°C = 100Ω (PT100)
```

### วงจร 4-20mA 2-Wire Transmitter

```
+24V ──────────────────────────────────────┐
  │                                        │
  │    ┌─────────────────────┐            │
  ├───►│ BRN (+Power)         │            │
  │    │                      │            │
  │    │  2-Wire Transmitter  │            │
  │    │  (Pressure, Flow,    │            │
  │    │   Level, etc.)       │            │
  │    │                      │            │
  │    │ BLU (Signal Out)─────┼───────────►│ AI+ (Analog Input)
  │    │                      │            │
  │    └─────────────────────┘            │ AI- (Analog Common)
  │                                        │
 0V ─────────────────────────────────────►│
```

### การอ่านค่า Analog ใน Ladder Diagram

**Siemens S7-1200:**
```ladder
// อ่านค่า Pressure จาก Analog Input
// Raw Value: 0-27648 = 4-20mA = 0-10 Bar

|         |
| Always  |──[MOVE]──────────────────────────┤
| On      │   IN: %IW64 (AI_CHANNEL_1)       │
|         │   OUT: #Raw_Value                │
|         |                                  |
|         |──[SCALE_X]───────────────────────┤
|         │   INPUT: #Raw_Value              │
|         │   LO_LIM: 0.0                    │
|         │   HI_LIM: 10.0                   │
|         │   VALUE: #Pressure_Bar           │
```

**Allen-Bradley:**
```ladder
// อ่านค่า Level จาก Analog Input
// ใน ControlLogix: Tag-based addressing

|         |
| Always  |──[SCL]────────────────────────────┤
| On      │   Source: Level_AI_Raw           │
|         │   Scale: 100.0                   │
|         │   Offset: 0.0                    │
|         │   Dest: Level_Percent            │
```

**Mitsubishi FX:**
```ladder
// อ่านค่า Temperature
FROM K0 K0 D100 K1    // อ่านค่า AIN CH0 ไปเก็บใน D100

// Scale Calculation
MUL D100 K100 D200    // D200 = Raw × 100
DIV D200 K4000 D201   // D201 = Scaled Temperature
```

---

## Step 202: การต่อ Analog Output

### วงจร Analog Output 4-20mA

```
PLC Analog Output Module

+24V ─────────────────────────────────────────┐
  │                                           │
  │   ┌─────────────────────────────────┐    │
  │   │      AO Module                  │    │
  │   │                                 │    │
  │   │  AO0 ─────────────────────────► │ AO+
  │   │                                 │    │
  │   │  0V ──────────────────────────► │ AO-
  │   │                                 │    │
  │   └─────────────────────────────────┘    │
  │                                          │
  │    ┌─────────────────────┐              │
  │    │   Control Valve      │              │
  └───►│ + (Signal In)        │              │
       │   Actuator           │              │
  ─────┤ - (Common)           │              │
       └─────────────────────┘
```

### Ladder Diagram สำหรับ PID Control

```ladder
// Flow Control with PID

Network 1: Read Flow
|         |
| Always  |──[SCALE]─────────────────────────────┤
| On      │  IN: %IW68 (Flow Sensor)             │
|         │  OUT: #Flow_LPM (0.0-100.0 LPM)     │

Network 2: PID Controller  
|         |
| PID_EN  |──[PID_COMPACT]────────────────────────┤
|         │  Setpoint: #Flow_SP                   │
|         │  Actual: #Flow_LPM                    │
|         │  Output: #Valve_Output (0.0-100.0%)   │

Network 3: Write to AO
|         |
| Always  |──[SCALE]─────────────────────────────┤
| On      │  IN: #Valve_Output (0.0-100.0%)      │
|         │  OUT: %QW80 (Analog Output)           │
|         │  → 0-27648 = 0-20mA                  │
```

---

## Step 203: I/O Diagnostics และ Troubleshooting

### การวัดสัญญาณ I/O ด้วย Multimeter

```
การวัด Digital Input:
1. ตั้ง Voltmeter ไปที่ 50VDC
2. วัดระหว่าง Input Terminal กับ COM
3. Input ON = ~24V
4. Input OFF = ~0V

การวัด Digital Output:
1. PLC Command Output ON
2. วัดระหว่าง Output Terminal กับ COM
3. Output ON = ~24V (Transistor)
4. Output ON = ต่อวงจร (Relay)

การวัด Analog Current:
1. ตั้ง Ammeter (mA DC)
2. ต่อ Series กับสาย Signal
3. ค่าปกติ = 4-20mA
4. ถ้า < 4mA = สายขาดหรือ Sensor Error
```

### Troubleshooting Chart

```
Input ไม่ทำงาน:
    │
    ├── วัดแรงดันที่ Input Terminal
    │        │
    │   ≥18V │          < 18V
    │        │              │
    │   PLC Software    ตรวจ Sensor
    │   Issue           และสาย
    │        │
    │   Force Input ON
    │   ใน Software
    │        │
    │   ไม่ได้ผล
    │        │
    │   Input Module เสีย

Output ไม่ทำงาน:
    │
    ├── ตรวจ PLC Program Output Coil
    │        │
    │   Output Coil = 1    Output Coil = 0
    │        │                    │
    │   วัดแรงดัน           Program Logic
    │   ที่ Output           Issue
    │   Terminal
    │        │
    │   มีแรงดัน    ไม่มีแรงดัน
    │        │           │
    │   โหลดเสีย    Output Module เสีย
    │   หรือสายขาด
```

---

## Step 204: การออกแบบ I/O List

### I/O List Template

```
PROJECT: Production Line Conveyor Control
PLC: Siemens S7-1200 CPU 1214C
DATE: 01/10/2026

=== DIGITAL INPUTS ===
┌──────┬───────────┬───────────────────┬───────────────────────┐
│ Addr │  Tag Name │  Description (TH) │  Device Type          │
├──────┼───────────┼───────────────────┼───────────────────────┤
│ I0.0 │ PB_START  │ ปุ่มสตาร์ท       │ PB, NO, Green         │
│ I0.1 │ PB_STOP   │ ปุ่มหยุด         │ PB, NC, Red           │
│ I0.2 │ PB_EMGSTOP│ Emergency Stop   │ Mushroom PB, NC, Red  │
│ I0.3 │ LS_FWD    │ Limit Switch หน้า│ Mechanical LS, NO     │
│ I0.4 │ LS_REV    │ Limit Switch หลัง│ Mechanical LS, NO     │
│ I0.5 │ PS_OL_M1  │ Motor 1 Overload │ OL Relay, NC          │
│ I0.6 │ PROX_PART │ ตรวจจับชิ้นงาน  │ Inductive Prox, NPN   │
│ I0.7 │ PHOTO_CNT │ นับชิ้นงาน      │ Photoelectric, NPN    │
└──────┴───────────┴───────────────────┴───────────────────────┘

=== DIGITAL OUTPUTS ===
┌──────┬───────────┬───────────────────┬───────────────────────┐
│ Addr │  Tag Name │  Description (TH) │  Device Type          │
├──────┼───────────┼───────────────────┼───────────────────────┤
│ Q0.0 │ KM1_FWD   │ Contactor ไปข้าง  │ Contactor 24VDC Coil  │
│ Q0.1 │ KM2_REV   │ Contactor ถอยหลัง │ Contactor 24VDC Coil  │
│ Q0.2 │ KM3_CONV  │ Conveyor Motor    │ Contactor 24VDC Coil  │
│ Q0.3 │ SOL_GATE  │ Solenoid Gate     │ 24VDC Solenoid        │
│ Q0.4 │ PL_RUN    │ ไฟรันนิ่ง        │ Pilot Lamp 24V Green  │
│ Q0.5 │ PL_FAULT  │ ไฟ Fault         │ Pilot Lamp 24V Red    │
│ Q0.6 │ BUZZER    │ เสียงเตือน       │ 24VDC Buzzer          │
│ Q0.7 │ SPARE     │ สำรอง            │ -                     │
└──────┴───────────┴───────────────────┴───────────────────────┘

=== ANALOG INPUTS ===
┌──────┬───────────┬───────────────────┬───────────────────────┐
│ Addr │  Tag Name │  Description (TH) │  Range                │
├──────┼───────────┼───────────────────┼───────────────────────┤
│ IW64 │ AI_SPEED  │ Speed Feedback    │ 4-20mA = 0-1500 RPM  │
│ IW66 │ AI_TEMP   │ Motor Temperature │ PT100, 0-200°C        │
└──────┴───────────┴───────────────────┴───────────────────────┘

=== ANALOG OUTPUTS ===
┌──────┬───────────┬───────────────────┬───────────────────────┐
│ Addr │  Tag Name │  Description (TH) │  Range                │
├──────┼───────────┼───────────────────┼───────────────────────┤
│ QW80 │ AO_VFD    │ Speed Reference   │ 0-10V = 0-1500 RPM   │
└──────┴───────────┴───────────────────┴───────────────────────┘
```

---

## Step 205: การต่อวงจร Emergency Stop

### Emergency Stop Circuit ตามมาตรฐาน IEC 60947-5-1

```
Emergency Stop (E-Stop) Wiring

+24V
  │
  ├──[E-Stop 1 NC]──[E-Stop 2 NC]──[E-Stop 3 NC]──┐
  │                                                 │
  │                                          ┌──────┘
  │                                          │
  │                                    ┌─────▼───────┐
  │                                    │ Safety Relay│
  │                                    │  or PLC     │
  │                                    │  Input I0.0 │
  │                                    └─────────────┘
  │
 0V (COM)

กฎ: E-Stop ต้องต่อ Series (NC Contact)
    ถ้า E-Stop ตัวใดตัวหนึ่งกด = System STOP
```

### E-Stop Category 1 (Power Off Fast)

```ladder
// E-Stop Category 1 Program

Network 1: E-Stop Monitor
|         |
| %I0.0   |──────────────────────────────────( %M0.0 )
| E_STOP  |                                  Safety_OK
|  (NC)   |

Network 2: Motor Control with E-Stop
|  %M0.0  |  %I0.1   |  %I0.2  |
|Safety_OK|──PB_START──PB_STOP_NC──────────( %Q0.0 )
|         |           |                    Motor_Run
|         |
|  %Q0.0  |
|Motor_Run|──────────────────────────────────┘(Self-Hold)

// ถ้า E-Stop ถูกกด %M0.0 = 0 → Q0.0 = 0 → Motor STOP
```

---

## Step 206: IP Rating และ Environmental Protection

### IP Rating (Ingress Protection)

```
IP Rating: IP[First Digit][Second Digit]

First Digit (Solid Protection):
0 = ไม่ป้องกัน
1 = ≥50mm objects
2 = ≥12.5mm objects
3 = ≥2.5mm objects
4 = ≥1mm objects
5 = Dust Protected
6 = Dust Tight

Second Digit (Liquid Protection):
0 = ไม่ป้องกัน
1 = Vertical drops
2 = Dripping (15° tilt)
3 = Spraying water
4 = Splashing water
5 = Water jets
6 = Powerful jets
7 = Immersion (1m, 30min)
8 = Continuous immersion
```

### การเลือก Enclosure ตามสภาพแวดล้อม

```
สภาพแวดล้อม → IP Rating ที่แนะนำ:

Indoor, clean          → IP20
Indoor, dusty          → IP54
Indoor, wash-down      → IP65
Outdoor, no jet water  → IP54
Outdoor, jet water     → IP65 หรือ IP66
Outdoor, immersion     → IP67
Chemical environment   → IP65 + Chemical-resistant

ตัวอย่างใน Factory:
- Control Panel (in clean room): IP20
- Control Panel (production floor): IP54
- Junction Box (outdoor): IP65
- Underwater pump controller: IP68
```

---

## Step 207: VFD Input/Output Wiring

### การต่อ VFD กับ PLC

```
VFD (Variable Frequency Drive) Wiring

PLC Digital Output → VFD Control Input
┌─────────────────────────────────────────────────────────┐
│                   VFD Control Terminal                  │
│                                                         │
│  +24V ────────────────────────────────► +24V COM        │
│                                                         │
│  Q0.0 (RUN/STOP) ────────────────────► FWD (Forward)  │
│  Q0.1 (REV)      ────────────────────► REV (Reverse)  │
│  Q0.2 (FAULT RST)────────────────────► RST (Reset)    │
│                                                         │
│  0V (COM) ────────────────────────────► COM            │
└─────────────────────────────────────────────────────────┘

PLC Analog Output → VFD Speed Reference
┌─────────────────────────────────────────────────────────┐
│  QW80 (0-10V) ────────────────────────► AI1 (Speed)   │
│  0V (AO COM)  ────────────────────────► GND            │
└─────────────────────────────────────────────────────────┘

VFD Output → PLC Digital Input
┌─────────────────────────────────────────────────────────┐
│  VFD RUN Feedback ───────────────────► I0.3 (PLC)     │
│  VFD FAULT ──────────────────────────► I0.4 (PLC)     │
│  VFD AT SPEED ───────────────────────► I0.5 (PLC)     │
└─────────────────────────────────────────────────────────┘
```

---

## Step 208: การต่อ Communication Networks

### RS-485 Wiring

```
RS-485 Multi-drop Network (Modbus RTU)

PLC Master                Slave 1            Slave 2
   │                        │                   │
   │ A+ ────────────────────A+ ─────────────────A+
   │                        │                   │
   │ B- ────────────────────B- ─────────────────B-
   │                        │                   │
   │ GND ───────────────────GND ────────────────GND
   │
   │← 120Ω Termination →               ← 120Ω Termination
   at PLC end                           at last device end

Maximum Distance: 1200m at 9600 baud
Maximum Nodes: 32 (standard), 128 (with repeater)
```

### PROFIBUS DP Wiring

```
PROFIBUS DP Segment

Master                  Slave 1        Slave 2
   │                      │              │
   ├── A (Green/Pin 3) ───A ─────────────A
   ├── B (Red/Pin 8) ─────B ─────────────B
   │
   ├─[120Ω] at each end
   
Speed vs Distance:
12 Mbps = 100m
1.5 Mbps = 400m
187.5 kbps = 1000m
9.6 kbps = 1200m
```

---

## Step 209: การจัดทำ Wiring Documentation

### As-Built Drawing Standards

```
สิ่งที่ต้องมีใน Wiring Documentation:

1. Single Line Diagram (SLD)
   - แสดง Power Distribution
   - MCB/Fuse Rating
   - Cable Size

2. Schematic Diagram  
   - Control Circuit
   - I/O Connection
   - Component Tag

3. Panel Layout Drawing
   - ตำแหน่งอุปกรณ์ใน Panel
   - เส้นทางสาย
   - Terminal Block Number

4. I/O List
   - Address
   - Tag Name
   - Description
   - Device Type

5. Cable Schedule
   - Cable Number
   - From/To
   - Cable Type/Size
```

### Cable Numbering System

```
ตัวอย่าง Cable Numbering:

Cable Number: C-P01-I-001
              │  │   │  └── Sequential Number
              │  │   └───── I=Instrument, P=Power, C=Control
              │  └────────── P01 = Panel 01
              └───────────── C = Cable

หรือแบบง่าย:
W101 = Wiring, Panel 1, Cable 01
W201 = Wiring, Panel 2, Cable 01
```

---

## Step 210: Lab Project - Complete I/O Panel Wiring

### Lab 11: การสร้าง I/O Wiring Panel

**วัตถุประสงค์:**
1. ฝึกการต่อวงจร Input 24VDC
2. ฝึกการต่อ NPN Sensor
3. ฝึกการต่อ Relay Output
4. ฝึกการ Label สายและอุปกรณ์

**อุปกรณ์ที่ต้องการ:**
- PLC (Siemens S7-1200 หรือ Mitsubishi FX3U)
- Power Supply 24VDC/5A
- Pushbutton x3 (Start, Stop, Reset)
- Emergency Stop x1
- NPN Proximity Sensor x2
- Pilot Lamp x3 (Green, Red, Yellow)
- Contactor 24VDC Coil x1
- Overload Relay x1
- Terminal Block (20 points)
- Wire 1.5mm² (Red, Blue, Green)
- Ferrule Set
- DIN Rail + Panel Frame

**Wiring Diagram:**
```
      TB1 TB2 TB3 TB4 TB5 TB6 TB7 TB8
+24V───┬───┬───┬───┬───┬───┬───┬───┐
       │   │   │   │   │   │   │   │
      PB1 PB2 EST PROX1 PL1 PL2 PL3 KM1
     (I0.0)(I0.1)(I0.2)(I0.3)(Q0.0)(Q0.1)(Q0.2)(Q0.3)
       │   │   │   │   │   │   │   │
      I0.0 I0.1 I0.2 I0.3 Q0.0 Q0.1 Q0.2 Q0.3
            [PLC S7-1200]
       │   │   │   │   │   │   │   │
      COM  COM  COM  COM  COM COM COM COM
       │   │   │   │   │   │   │   │
 0V───┴───┴───┴───┴───┴───┴───┴───┘
```

**Ladder Program สำหรับทดสอบ:**
```ladder
// Lab 11: I/O Test Program
// Siemens S7-1200

Network 1: Start/Stop Control
|  %I0.0  |  %I0.1  |  %I0.2  |
|PB_START |PB_STOP_N|ESTOP_NC |──────────────────( %Q0.0 )
|         |         |          |                 KM1_COIL
|  %Q0.0  |
|KM1_SELF |──────────────────────────────────────┘
|  Hold   |

Network 2: Running Indicator
|  %Q0.0  |
|KM1_COIL |──────────────────────────────────────( %Q0.1 )
|         |                                      GREEN_LAMP

Network 3: Fault Indicator  
|  %I0.2  |
|ESTOP_NC |──────────────────────────────────────( %Q0.2 )
| (NC)    |                                       RED_LAMP

Network 4: Sensor Counter
|  %I0.3  |
|PROX1    |──────────────────────────────────────[CTU C0]
|         |                                      Count=10
|  %C0.Q  |
|Count_DN |──────────────────────────────────────( %Q0.3 )
|         |                                       YELLOW_LAMP
```

**การประเมินผล:**
- ✓ I/O List ครบถ้วน
- ✓ สายมี Label ชัดเจน
- ✓ Ferrule ทุกเส้น
- ✓ Terminal Torque ถูกต้อง
- ✓ Program ทำงานถูกต้อง
- ✓ E-Stop ทำงาน

---

## สรุป Part 11

ในบทนี้เราได้เรียนรู้:
1. ระบบไฟเลี้ยง 24VDC สำหรับ PLC
2. การต่อวงจร Digital Input ทั้ง NPN และ PNP
3. ความแตกต่างของ Relay, Transistor, Triac Output
4. การ Grounding และ Shielding ที่ถูกต้อง
5. การเดินสาย Cable Routing
6. I/O Terminal Block และ Best Practices
7. ข้อผิดพลาดที่พบบ่อยและวิธีป้องกัน
8. Analog I/O Wiring
9. การจัดทำ I/O Documentation

---

## การนำทาง

- [← Part 10: Structured Programming](part10_structured_programming.md)
- [→ Part 12: Safety พื้นฐาน](part12_safety_basics.md)
- [กลับหน้าหลัก](../README.md)

---

*Part 11 จบ | ต่อไป Part 12: Safety พื้นฐาน*
