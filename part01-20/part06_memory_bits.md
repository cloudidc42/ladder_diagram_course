# Part 06: Memory Bits และ Internal Coils
## Steps 91-110: การใช้งาน Bit Memory, Retentive Coils และ System Flags

---

## สารบัญ

- [Step 91: Memory Bits คืออะไร?](#step-91)
- [Step 92: Internal Coils (M Bits)](#step-92)
- [Step 93: Retentive Memory Bits](#step-93)
- [Step 94: First Scan Bit](#step-94)
- [Step 95: System Flags และ Status Bits](#step-95)
- [Step 96: Flag Bit Patterns](#step-96)
- [Step 97: Latch Patterns พื้นฐาน](#step-97)
- [Step 98: Set/Reset Coil](#step-98)
- [Step 99: Data Retention หลัง Power Loss](#step-99)
- [Step 100: Interlock Logic ด้วย Memory Bits](#step-100)
- [Step 101: State Machine ด้วย Memory Bits](#step-101)
- [Step 102: Motor Control Patterns](#step-102)
- [Step 103: Sequencing ด้วย Memory Bits](#step-103)
- [Step 104: One-Shot Bit](#step-104)
- [Step 105: Diagnostic Bits](#step-105)
- [Step 106: Communication Flags](#step-106)
- [Step 107: Alarm Management ด้วย Memory Bits](#step-107)
- [Step 108: Recipe Bits](#step-108)
- [Step 109: Advanced State Tracking](#step-109)
- [Step 110: Workshop Project - Motor Control System](#step-110)

---

## Step 91: Memory Bits คืออะไร? {#step-91}

### ความเข้าใจพื้นฐาน

**Memory Bits** หรือที่เรียกว่า **Internal Coils** คือ bit ที่อยู่ภายใน PLC ไม่เชื่อมต่อกับอุปกรณ์ภายนอก ใช้สำหรับเก็บสถานะ, สร้าง logic ภายใน และควบคุมการทำงานของโปรแกรม

### ทำไมต้องใช้ Memory Bits?

```
ปัญหาที่พบบ่อย:
┌─────────────────────────────────────────────────────────┐
│  คุณต้องการ "จำ" ว่าเกิดอะไรขึ้นก่อนหน้านี้            │
│  แต่ Physical Output มีจำกัดและต้องควบคุมอุปกรณ์จริง    │
│                                                         │
│  해결책: ใช้ Memory Bits เป็น "โน้ตภายใน" ของ PLC      │
└─────────────────────────────────────────────────────────┘
```

### การเปรียบเทียบ Memory Types

| ประเภท | ตัวอย่างที่อยู่ | เก็บข้อมูลหลัง Power Off | ใช้งาน |
|--------|----------------|--------------------------|--------|
| Output Bit (Q/Y/O) | Q0.0, Y0, O:0/0 | ไม่เก็บ | ควบคุมอุปกรณ์ภายนอก |
| Memory Bit (M/B/B) | M0.0, B0, B3:0 | ไม่เก็บ (ปกติ) | Logic ภายใน |
| Retentive Bit (MR) | M100.0 | เก็บ | จำสถานะสำคัญ |
| Data Register Bit | D0.0 | ขึ้นอยู่กับการตั้งค่า | เก็บข้อมูล |

### การกำหนดที่อยู่ใน 3 แบรนด์หลัก

```
┌──────────────────┬─────────────────┬────────────────────────────┐
│ Siemens S7-1200  │ Allen-Bradley   │ Mitsubishi Q/FX            │
├──────────────────┼─────────────────┼────────────────────────────┤
│ M0.0 ถึง M0.7    │ B3:0/0          │ M0 ถึง M7 (M0-M7 = 1 byte) │
│ M1.0 ถึง M1.7    │ B3:0/1          │ M8 ถึง M15                 │
│ MW0 (word)       │ N7:0 (integer)  │ D0 (data register)         │
│ MD0 (dword)      │ F8:0 (float)    │ D0-D999 (data)             │
│ %M100.0          │ B3:100/0        │ M100                       │
└──────────────────┴─────────────────┴────────────────────────────┘
```

---

## Step 92: Internal Coils (M Bits) {#step-92}

### โครงสร้างของ M Bit

```
Siemens S7-1200/1500:
┌────────────────────────────────────────────────────────────────┐
│  M Area (Merker/Memory)                                        │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐           │
│  │M0.7 │M0.6 │M0.5 │M0.4 │M0.3 │M0.2 │M0.1 │M0.0 │ Byte MB0 │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘           │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐           │
│  │M1.7 │M1.6 │M1.5 │M1.4 │M1.3 │M1.2 │M1.1 │M1.0 │ Byte MB1 │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘           │
│  ...ต่อไปเรื่อยๆ                                               │
└────────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Ladder Diagram: M Bit พื้นฐาน

```
Network 1: เปิดมอเตอร์โดยใช้ M Bit เป็น Flag

     I0.0           I0.1            M0.0
  ───┤ ├─────────────┤/├─────────────( )───
   Start            Stop           Motor Flag


Network 2: ควบคุม Output จาก M Bit

     M0.0           M0.1            Q0.0
  ───┤ ├─────────────┤/├─────────────( )───
  Motor Flag      Fault Flag       Motor Output
```

### ข้อดีของการใช้ M Bits แทน Output โดยตรง

```
วิธีไม่ดี (ควบคุม Output โดยตรง):
  ──┤Start├──┤/Stop├──( Q0.0 )──   ← ยากในการ Debug

วิธีที่ดี (ใช้ M Bit เป็นตัวกลาง):
  ──┤Start├──┤/Stop├──( M0.0 )──   ← Motor Running Flag
  ──┤M0.0 ├──┤/Fault├──( Q0.0 )──  ← Motor Output (มี Fault interlock)
```

### Allen-Bradley ตัวอย่าง

```
Network 1: (Allen-Bradley SLC-500)

     I:0/0          I:0/1           B3:0/0
  ───┤ ├─────────────┤/├─────────────( )───
   Start             Stop         Motor_Flag


Network 2:
     B3:0/0         B3:0/5          O:0/0
  ───┤ ├─────────────┤/├─────────────( )───
  Motor_Flag       Fault_Flag     Motor_Output
```

### Mitsubishi FX ตัวอย่าง

```
Network 1:
     X0             X1              M0
  ───┤ ├─────────────┤/├─────────────( )───
   Start             Stop         Motor Flag

Network 2:
     M0             M10             Y0
  ───┤ ├─────────────┤/├─────────────( )───
  Motor Flag      Fault Flag     Motor Output
```

---

## Step 93: Retentive Memory Bits {#step-93}

### ความสำคัญของ Retentive Memory

เมื่อไฟดับและเปิดใหม่ Memory Bits ปกติจะถูก Reset เป็น 0 ทั้งหมด แต่บางครั้งเราต้องการ "จำ" สถานะที่สำคัญ เช่น:

- โหมดการทำงาน (Auto/Manual)
- สถานะ Fault ที่ยังไม่ได้รับการแก้ไข
- การนับชิ้นงานที่ผ่านมา
- ขั้นตอนที่ทำงานค้างอยู่

### Retentive Areas ใน 3 แบรนด์

```
┌──────────────────┬────────────────────────────┬────────────────────┐
│ แบรนด์           │ Non-Retentive              │ Retentive          │
├──────────────────┼────────────────────────────┼────────────────────┤
│ Siemens S7-1200  │ M0.0 - M99.7 (default)     │ M100.0+ หรือ       │
│                  │                            │ กำหนดใน DB         │
│                  │                            │ Retain = TRUE      │
├──────────────────┼────────────────────────────┼────────────────────┤
│ Allen-Bradley    │ B3 (bit file, non-retain)  │ N7 with Retentive  │
│ SLC/MicroLogix   │                            │ Property, หรือ     │
│                  │                            │ Unlatch coil (L)   │
├──────────────────┼────────────────────────────┼────────────────────┤
│ Mitsubishi FX3U  │ M0 - M499                  │ M500 - M1023       │
│                  │                            │ (Battery backup)   │
├──────────────────┼────────────────────────────┼────────────────────┤
│ Mitsubishi Q     │ M0 - M2047                 │ M2048 - M8191      │
│                  │                            │ (Latch range)      │
└──────────────────┴────────────────────────────┴────────────────────┘
```

### ตัวอย่าง: เก็บโหมดการทำงานหลัง Power Loss

```
Network 1: เลือกโหมด Auto/Manual

     I0.5              M500.0
  ───┤ ├────────────────( S )───    ← Set Auto Mode (Retentive)
   Auto Select

     I0.6              M500.0
  ───┤ ├────────────────( R )───    ← Reset = Manual Mode
   Manual Select


Network 2: ตรวจสอบโหมด

     M500.0            Q0.7
  ───┤ ├────────────────( )───      ← Auto Mode Indicator Light
   Auto Mode Flag

     M500.0            Q0.6
  ───┤/├────────────────( )───      ← Manual Mode Indicator Light
   Auto Mode Flag
```

### Truth Table: Retentive Set/Reset

| เหตุการณ์ | Set Input | Reset Input | สถานะ M500.0 | หลัง Power Cycle |
|-----------|-----------|-------------|--------------|------------------|
| เริ่มต้น  | 0 | 0 | 0 | 0 |
| กด Auto   | 1 | 0 | 1 | **1** (จำได้) |
| ไฟดับ     | - | - | - | **1** (ยังจำ) |
| ไฟติด     | - | - | **1** | **1** (ยังจำ) |
| กด Manual | 0 | 1 | 0 | **0** (จำได้) |
| ไฟดับอีก  | - | - | - | **0** (ยังจำ) |

---

## Step 94: First Scan Bit {#step-94}

### First Scan คืออะไร?

**First Scan Bit** คือ bit พิเศษที่เป็น 1 เฉพาะ **Scan แรก** หลัง PLC เริ่มทำงาน จากนั้นจะเป็น 0 ตลอดไป ใช้สำหรับ:

- Initialize ค่าเริ่มต้น
- ตรวจสอบสถานะก่อนเริ่มทำงาน
- Reset counter/timer ที่ไม่ต้องการ

### System First Scan Bits

```
┌──────────────────┬─────────────────────────────────────┐
│ แบรนด์           │ First Scan Bit                      │
├──────────────────┼─────────────────────────────────────┤
│ Siemens S7-1200  │ "FirstScan" ใน OB1                   │
│                  │ หรือสร้างเอง ดูตัวอย่างด้านล่าง      │
├──────────────────┼─────────────────────────────────────┤
│ Allen-Bradley    │ S:1/15 (First Pass bit)              │
│ SLC-500          │                                     │
├──────────────────┼─────────────────────────────────────┤
│ Allen-Bradley    │ S2:1/15 หรือ                         │
│ PLC-5            │ FirstScan tag (ControlLogix)         │
├──────────────────┼─────────────────────────────────────┤
│ Mitsubishi FX    │ M8002 (Initial Pulse, ON 1 scan)     │
│                  │ M8000 (Run Monitor, ON while run)    │
├──────────────────┼─────────────────────────────────────┤
│ Mitsubishi Q     │ SM402 (First scan)                   │
│                  │ SM400 (Always ON)                    │
└──────────────────┴─────────────────────────────────────┘
```

### ตัวอย่าง Mitsubishi: ใช้ M8002

```
Network 1: Initialize เมื่อ PLC Start

     M8002                           ← M8002 = ON แค่ 1 scan แรก
  ─────┤ ├──────────┬──────────────────────────────────────────
                    │
                    ├──[MOV K0 D100]─    ← Set counter = 0
                    │
                    ├──[MOV K0 D200]─    ← Clear accumulator
                    │
                    └──( M200.0 )────    ← Set "Initialized" flag


Network 2: ตรวจสอบว่า Initialize แล้วหรือยัง

     M200.0            Q0.5
  ───┤ ├────────────────( )───      ← System Ready Light
   Init Done
```

### ตัวอย่าง Allen-Bradley: ใช้ S:1/15

```
Network 1: Allen-Bradley SLC-500 First Scan

     S:1/15                          ← First Pass bit
  ─────┤ ├──────┬────────────────────
                │
                ├──[CLR  N7:0]───    ← Clear count register
                ├──[CLR  N7:1]───    ← Clear accumulator
                └──(  B3:0/10  )─    ← Set Initialized flag


Network 2:
     B3:0/10         O:0/5
  ───┤ ├──────────────( )───         ← System Ready indicator
  Init_Done
```

### สร้าง First Scan ด้วยตัวเอง (Siemens หรือทั่วไป)

```
เทคนิค: ใช้ NC contact ของ M bit ที่จะถูก Set ทันที

Network 1:
     M0.7            M0.7            ← M0.7 = "First Scan" bit
  ───┤/├──────────────( )───         ← NC ของตัวเอง → ON แค่ scan แรก!

 Explanation:
- Scan 1: M0.7 = 0 → NC contact ปิด (pass) → M0.7 ถูก Set = 1
- Scan 2: M0.7 = 1 → NC contact เปิด (no pass) → M0.7 ยังคง = 1
          แต่ coil ไม่ถูก drive → M0.7 = 1 แค่ 1 scan แรก!
```

---

## Step 95: System Flags และ Status Bits {#step-95}

### System Flags คืออะไร?

System Flags คือ Memory Bits พิเศษที่ PLC ควบคุมเอง แสดงสถานะของระบบ เช่น:
- PLC กำลัง Run อยู่
- มี Error
- Timer clock pulse
- Battery status

### Siemens S7 System Bits

```
┌────────────┬──────────────────────────────────────────────────────┐
│ Address    │ คำอธิบาย                                             │
├────────────┼──────────────────────────────────────────────────────┤
│ M254.7     │ Always TRUE (1) - ใช้แทน TRUE constant               │
│ M254.6     │ Always FALSE (0) - ใช้แทน FALSE constant             │
├────────────┼──────────────────────────────────────────────────────┤
│ Siemens ใช้ Clock Memory bits (ต้องตั้งค่าใน Device Config)        │
│ MB0.0      │ 0.5 Hz Pulse (1 sec ON, 1 sec OFF)                   │
│ MB0.1      │ 1 Hz Pulse (0.5 sec ON, 0.5 sec OFF)                 │
│ MB0.2      │ 2 Hz Pulse (0.25 sec ON, 0.25 sec OFF)               │
│ MB0.3      │ 5 Hz Pulse (0.1 sec ON, 0.1 sec OFF)                 │
│ MB0.4      │ 10 Hz Pulse (0.05 sec ON, 0.05 sec OFF)              │
└────────────┴──────────────────────────────────────────────────────┘
```

### Mitsubishi FX Special Relays (M8000-M8511)

```
┌────────────┬──────────────────────────────────────────────────────┐
│ Address    │ คำอธิบาย                                             │
├────────────┼──────────────────────────────────────────────────────┤
│ M8000      │ RUN Monitor - ON ตลอดเวลาที่ PLC Run                 │
│ M8001      │ RUN Monitor - OFF ตลอดเวลาที่ PLC Run (inverse)      │
│ M8002      │ Initial Pulse - ON แค่ 1 scan แรก                    │
│ M8003      │ Initial Pulse - OFF แค่ 1 scan แรก                   │
│ M8011      │ 10ms Clock Pulse                                     │
│ M8012      │ 100ms Clock Pulse                                    │
│ M8013      │ 1s Clock Pulse                                       │
│ M8014      │ 1min Clock Pulse                                     │
│ M8020      │ Zero Flag (ผลลัพธ์ = 0)                              │
│ M8021      │ Borrow Flag (ลบได้ค่าติดลบ)                          │
│ M8022      │ Carry Flag (บวกแล้วเกิน)                             │
│ M8034      │ All Output Disable                                   │
│ M8040      │ Transfer Disable                                     │
│ M8044      │ Origin Return Complete                               │
└────────────┴──────────────────────────────────────────────────────┘
```

### Allen-Bradley Status File (S:)

```
┌────────────┬──────────────────────────────────────────────────────┐
│ Address    │ คำอธิบาย                                             │
├────────────┼──────────────────────────────────────────────────────┤
│ S:0/0      │ Carry Bit                                            │
│ S:0/1      │ Overflow Bit                                         │
│ S:0/2      │ Zero Bit                                             │
│ S:0/3      │ Sign Bit                                             │
│ S:1/8      │ Fault routine enable                                 │
│ S:1/15     │ First Pass Bit                                       │
│ S:2/1      │ Watchdog timeout                                     │
│ S:3/9      │ Minor fault                                          │
│ S:3/10     │ Major fault                                          │
│ S:4        │ Scan time (ms)                                       │
└────────────┴──────────────────────────────────────────────────────┘
```

### ตัวอย่าง: ใช้ Clock Pulse สำหรับ Blinking Light

```
Network 1: ไฟกะพริบ 1 Hz (Mitsubishi FX)

     M8013            Q0.7
  ───┤ ├──────────────( )───      ← ไฟกะพริบทุก 1 วินาที
  1sec Clock

Network 2: ไฟเตือนกะพริบ 2 Hz เมื่อมี Fault

     M100             M8012        Q0.6
  ───┤ ├──────────────┤ ├──────────( )───
   Fault Flag      100ms Clock  Warning Light (กะพริบเร็วเมื่อ fault)
```

### ตัวอย่าง Siemens: Clock Memory Bits

```
(ต้องเปิดใช้ Clock Memory ใน Device Configuration ก่อน)
(ตั้งค่า Clock Memory Byte = MB0)

Network 1: ไฟสถานะกะพริบ 1 Hz

     %M0.5             %Q0.7
  ────┤ ├───────────────( )───    ← ไฟกะพริบ 1 Hz (0.5Hz x2 cycle)
   1Hz_Clock           Status LED
```

---

## Step 96: Flag Bit Patterns {#step-96}

### Pattern 1: Simple On/Off Flag

```
วัตถุประสงค์: เก็บสถานะง่ายๆ เช่น "กำลังทำงาน" หรือ "หยุดแล้ว"

Network 1: Set Flag
     I0.0              M10.0
  ───┤ ├────────────────( )───    ← Motor Running Flag

Network 2: ใช้งาน Flag
     M10.0             Q0.0
  ───┤ ├────────────────( )───    ← Motor Output

     M10.0             Q0.1
  ───┤/├────────────────( )───    ← Motor Stopped Indicator
```

### Pattern 2: Edge-Triggered Flag

```
วัตถุประสงค์: สร้าง Pulse 1 scan เมื่อมี Rising Edge

     I0.0           M200.0          M200.0         M201.0
  ───┤ ├────────────┤/├──────────────( )─────       ( )───
  Push Button    Previous State    Current State    Pulse Flag

Note: M201.0 = ON แค่ 1 scan เมื่อ I0.0 มี Rising Edge
```

### Pattern 3: Handshake Flag

```
วัตถุประสงค์: สื่อสารระหว่าง 2 ส่วนของโปรแกรม

Program Section A:
   [เสร็จแล้ว] → Set M50.0 (Request to B)

   M50.0 = "ฉันทำเสร็จแล้ว ต้องการให้ B ทำงาน"

Program Section B:
   ┌─────────────────────────────────────────────────┐
   │  รับ M50.0 → ทำงาน → Set M50.1 (Acknowledge)  │
   └─────────────────────────────────────────────────┘

Program Section A:
   M50.1 = "B รับทราบแล้ว" → Reset M50.0
```

### Pattern 4: Interlock Flag

```
วัตถุประสงค์: ป้องกันการทำงานพร้อมกันที่อันตราย

Network 1: Motor A
     I0.0           M20.0(B_Running)     M20.0(A_Running)
  ───┤ ├──────────────┤/├───────────────────( )───
   Start_A          B Running             A Running

Network 2: Motor B
     I0.1           M10.0(A_Running)     M20.0(B_Running)
  ───┤ ├──────────────┤/├───────────────────( )───
   Start_B          A Running             B Running

ผล: A และ B ไม่สามารถทำงานพร้อมกันได้!
```

---

## Step 97: Latch Patterns พื้นฐาน {#step-97}

### Latch คืออะไร?

**Latch** คือวงจรที่ "จำ" สถานะไว้ แม้ input จะหายไปแล้ว มี 2 แบบหลัก:

```
1. Self-Sealing Circuit (Seal-in circuit)
2. Set/Reset Coil (S/R Coil)
```

### ประเภท 1: Self-Sealing Circuit

```
     I0.0           I0.1            M0.0
  ───┤ ├────┬────────┤/├─────────────( )───
   START    │        STOP
            │
     M0.0   │
  ───┤ ├────┘        ← M0.0 "ล็อก" ตัวเอง

การทำงาน:
- กด START (I0.0 = 1): M0.0 = 1
- ปล่อย START (I0.0 = 0): M0.0 ยังคง = 1 (เพราะ seal contact)
- กด STOP (I0.1 = 1): M0.0 = 0 (circuit ขาด)
```

### Truth Table: Self-Sealing Circuit

| START (I0.0) | STOP (I0.1) | M0.0 ก่อน | M0.0 หลัง |
|--------------|-------------|-----------|-----------|
| 0 | 0 | 0 | 0 |
| 1 | 0 | 0 | 1 |
| 0 | 0 | 1 | 1 (seal) |
| 0 | 1 | 1 | 0 |
| 1 | 1 | X | 0 (STOP priority) |

### ประเภท 2: Set/Reset Coil

```
Network 1: Set
     I0.0              M1.0
  ───┤ ├────────────────( S )───   ← S = Set (ไม่รีเซ็ตเองเมื่อ input หาย)
   START

Network 2: Reset
     I0.1              M1.0
  ───┤ ├────────────────( R )───   ← R = Reset
   STOP
```

### เปรียบเทียบ Set/Reset Syntax

```
┌──────────────────┬─────────────────────────────────────────────┐
│ แบรนด์           │ Set/Reset Syntax                            │
├──────────────────┼─────────────────────────────────────────────┤
│ Siemens          │ ──( S )── และ ──( R )──                     │
│                  │ หรือ SR/RS flip-flop block                  │
├──────────────────┼─────────────────────────────────────────────┤
│ Allen-Bradley    │ ──( L )── (Latch) และ ──( U )── (Unlatch)   │
│                  │                                             │
├──────────────────┼─────────────────────────────────────────────┤
│ Mitsubishi       │ ──( SET )── และ ──( RST )──                  │
│                  │ หรือ SET M0, RST M0 ใน instruction list     │
└──────────────────┴─────────────────────────────────────────────┘
```

### ตัวอย่าง Mitsubishi FX: Set/Reset

```
Network 1:
     X0              M0
  ───┤ ├──────────[SET M0]───      ← กด X0 → M0 = 1 (ค้างอยู่)

Network 2:
     X1              M0
  ───┤ ├──────────[RST M0]───      ← กด X1 → M0 = 0

Network 3:
     M0             Y0
  ───┤ ├─────────────( )───        ← Y0 ตาม M0
```

### Set Priority vs Reset Priority

```
SR Flip-Flop (Set Priority):
     S input → 1 เมื่อ S=1 ไม่ว่า R จะเป็นอะไร
     R input → 0 เฉพาะเมื่อ S=0 และ R=1

RS Flip-Flop (Reset Priority):
     R input → 0 เมื่อ R=1 ไม่ว่า S จะเป็นอะไร
     S input → 1 เฉพาะเมื่อ R=0 และ S=1

ข้อสำคัญ: ในระบบ Safety → ใช้ Reset Priority เสมอ!
           (ปุ่ม Stop ต้องชนะ Start เสมอ)
```

---

## Step 98: Set/Reset Coil แบบละเอียด {#step-98}

### ทำไม Set/Reset ดีกว่า Seal Circuit บางกรณี?

```
ข้อดีของ Set/Reset Coil:
1. อ่านโปรแกรมง่ายกว่า (แยก Set และ Reset เป็น Network คนละอัน)
2. หลาย Network สามารถ Set/Reset Coil เดียวกันได้
3. ไม่มีปัญหา Coil used multiple times

ข้อดีของ Seal Circuit:
1. เห็น Start และ Stop อยู่ใน Network เดียว ง่ายต่อการ troubleshoot
2. ใช้ได้กับ PLC รุ่นเก่าที่ไม่มี S/R Coil
```

### ตัวอย่างซับซ้อน: Motor with Multiple Start/Stop

```
Network 1: Local Start
     I0.0              M50.0
  ───┤ ├────────────────( S )───   ← Local Start Button

Network 2: Remote Start (จาก SCADA)
     M200.5            M50.0
  ───┤ ├────────────────( S )───   ← Remote Start Command

Network 3: Auto Start (เมื่อ Sensor ตรวจพบ)
     I1.5              M50.0
  ───┤ ├────────────────( S )───   ← Auto Start from Sensor

Network 4: Local Stop
     I0.1              M50.0
  ───┤ ├────────────────( R )───   ← Local Stop Button (Priority: First)

Network 5: Emergency Stop
     I0.7              M50.0
  ───┤ ├────────────────( R )───   ← E-Stop

Network 6: Fault Auto Stop
     M100.0            M50.0
  ───┤ ├────────────────( R )───   ← Fault condition

Network 7: Motor Output
     M50.0             Q0.0
  ───┤ ├────────────────( )───     ← Motor Contactor
```

### จัดลำดับความสำคัญของ Reset

```
สำคัญมาก! ใน PLC, ถ้า Coil เดียวกันถูก Set และ Reset
ในรอบ scan เดียวกัน:

กฎ Last Rung Wins:
Network ที่ evaluate ทีหลัง จะชนะ!

ดังนั้น: วาง Reset Network ไว้หลัง Set Network เสมอ
          (เพื่อให้ Safety/Stop ชนะ Start เสมอ)

Network 1: SET M50.0   ← Set ก่อน
Network 2: SET M50.0   ← Set (อีกที่)
Network 3: RST M50.0   ← Reset อยู่ท้ายสุด = RESET จะชนะ!
```

---

## Step 99: Data Retention หลัง Power Loss {#step-99}

### สถานการณ์ที่ต้องการ Data Retention

```
โรงงานสถานการณ์จริง:
┌──────────────────────────────────────────────────────────┐
│ โรงงานผลิตอาหารสายพาน Conveyor ทำงานอยู่                 │
│ มีชิ้นงาน 50 ชิ้นรอ Process อยู่บนสายพาน                 │
│ ไฟดับกะทันหัน!                                           │
│                                                          │
│ เมื่อไฟกลับมา:                                           │
│ - PLC ต้องรู้ว่าชิ้นงานอยู่ตรงไหน (Position Tracking)    │
│ - ต้องรู้ว่าโหมดใดที่ใช้อยู่ (Auto/Manual)               │
│ - ต้องรู้ว่า Batch ไหนที่กำลังทำอยู่                     │
└──────────────────────────────────────────────────────────┘
```

### วิธีกำหนด Retentive Memory ใน Siemens TIA Portal

```
วิธีที่ 1: ใช้ Data Block (DB) ที่ตั้งค่า Retain

[Device Configuration] → [Properties]
  → [Memory] → [Retentive Memory]
     → Set address range ที่ต้องการ Retain

วิธีที่ 2: สร้าง DB แบบ Retain
  Right-click DB → Properties → Retain = TRUE

วิธีที่ 3: ใช้ M area ที่ถูกตั้งค่าให้ Retain
  [PLC Properties] → [General] → [Memory]
     → Set M0.0 to M99.7 as Retentive
```

### วิธีกำหนด Retentive Memory ใน Mitsubishi

```
สำหรับ FX3U:
- M500-M1023, D200-D511 = Retentive (default)
  (ต้องมี Battery backup)
- ตั้งค่าได้ใน GX Works2:
  [PLC Parameter] → [Device] → [Latch Range]

สำหรับ Q Series:
- ตั้งค่าใน GX Works2:
  [PLC Parameter] → [Device]
     → Device: M, Latch L: From 2048, To 8191
```

### ตัวอย่าง: บันทึก Production Count แบบ Retentive

```
(Mitsubishi FX3U - ใช้ D200 ซึ่งเป็น Retentive)

Network 1: นับชิ้นงาน (Rising Edge)
     X5            M8.0            D200
  ───┤ ├──────────[INC D200]───    ← เพิ่มค่า D200 ทุกครั้งที่ X5 ON
  Sensor         (INC = increment)

Network 2: แสดงค่า (ต่อกับ Display)
     M8000           D200
  ───┤ ├──────────[MOV D200 D300]─ ← Copy ไป D300 สำหรับ Display

Network 3: Reset Count (กด Reset Button)
     X6
  ───┤ ├──────────[MOV K0 D200]─   ← Clear count เมื่อกด Reset
  Reset Button
```

---

## Step 100: Interlock Logic ด้วย Memory Bits {#step-100}

### Interlock คืออะไร?

**Interlock** คือ Logic ที่ป้องกันการทำงานที่ไม่ปลอดภัย โดยตรวจสอบเงื่อนไขก่อนอนุญาตให้ทำงาน

### ตัวอย่างจริง: Conveyor System Interlock

```
ระบบ: Belt Conveyor 3 ตัวต่อเนื่องกัน (C1, C2, C3)
      C1 → C2 → C3 (ชิ้นงานไหลจาก C1 ไป C3)

กฎ Interlock:
1. C1 ต้องไม่เริ่มถ้า C2 หยุดอยู่ (ชิ้นงานจะค้าง)
2. C2 ต้องไม่เริ่มถ้า C3 หยุดอยู่
3. C3 สามารถเริ่มก่อนได้เสมอ
4. หยุดต้องหยุดตาม: C1 → C2 → C3 (ป้องกันชิ้นงานค้าง)
```

```
Network 1: Conveyor 3 (ท้ายสุด - เริ่มก่อน)
     I2.0           I2.1           M30.0
  ───┤ ├────┬────────┤/├────────────( )───
  Start_C3  │        Stop_C3
            │
     M30.0  │
  ───┤ ├────┘


Network 2: Conveyor 2 (กลาง - เริ่มได้เมื่อ C3 ทำงาน)
     I1.0           I1.1           M30.0            M20.0
  ───┤ ├────┬────────┤/├────────────┤ ├──────────────( )───
  Start_C2  │        Stop_C2     C3_Running
            │
     M20.0  │
  ───┤ ├────┘


Network 3: Conveyor 1 (ต้น - เริ่มได้เมื่อ C2 ทำงาน)
     I0.0           I0.1           M20.0            M10.0
  ───┤ ├────┬────────┤/├────────────┤ ├──────────────( )───
  Start_C1  │        Stop_C1     C2_Running
            │
     M10.0  │
  ───┤ ├────┘


Network 4: Output C1
     M10.0           I3.0            Q0.0
  ───┤ ├──────────────┤/├────────────( )───
  C1_Running        E_Stop         Conveyor_1


Network 5: Output C2
     M20.0           I3.0            Q0.1
  ───┤ ├──────────────┤/├────────────( )───
  C2_Running        E_Stop         Conveyor_2


Network 6: Output C3
     M30.0           I3.0            Q0.2
  ───┤ ├──────────────┤/├────────────( )───
  C3_Running        E_Stop         Conveyor_3
```

---

## Step 101: State Machine ด้วย Memory Bits {#step-101}

### State Machine คืออะไร?

**State Machine** คือวิธีการออกแบบโปรแกรมที่มี "สถานะ" ชัดเจน แต่ละสถานะทำงานเฉพาะในเวลานั้น และมีเงื่อนไขเปลี่ยนสถานะ

### ตัวอย่าง: ระบบล้างถัง 3 ขั้นตอน

```
States:
  State 0: รอสั่งงาน (Idle)
  State 1: เติมน้ำ (Filling)
  State 2: หมักน้ำยา (Soaking)
  State 3: ระบายน้ำ (Draining)

Transitions:
  State 0 → State 1: กด Start
  State 1 → State 2: น้ำเต็ม (Level sensor)
  State 2 → State 3: Timer หมด (30 min)
  State 3 → State 0: ระบายหมด (Level sensor)
```

```
ใช้ M Bits แทน State:
  M0.0 = State 0 (Idle)
  M0.1 = State 1 (Filling)
  M0.2 = State 2 (Soaking)
  M0.3 = State 3 (Draining)


Network 1: Initialize - เริ่มที่ Idle State
     M8002             M0.0
  ───┤ ├────────────────( S )───


Network 2: Idle → Filling (กด Start)
     M0.0           I0.0           M0.0            M0.1
  ───┤ ├─────────────┤ ├────────────( R )────        ( S )───
   Idle           Start Button


Network 3: Filling → Soaking (น้ำเต็ม)
     M0.1           I0.5           M0.1            M0.2
  ───┤ ├─────────────┤ ├────────────( R )────        ( S )───
  Filling         Level Full


Network 4: Soaking → Draining (Timer 30 min)
     M0.2          T0(Done)        M0.2            M0.3
  ───┤ ├─────────────┤ ├────────────( R )────        ( S )───
  Soaking         30min Timer


Network 5: Draining → Idle (ระบายหมด)
     M0.3           I0.6           M0.3            M0.0
  ───┤ ├─────────────┤ ├────────────( R )────        ( S )───
  Draining        Level Empty


Network 6: Soaking Timer (ทำงานเฉพาะใน State Soaking)
     M0.2
  ───┤ ├──────────────[TON T0 1800s]─    ← 30 นาที = 1800 วินาที


Network 7: Fill Valve Output (เฉพาะ Filling State)
     M0.1             Q0.0
  ───┤ ├──────────────( )───      ← Fill Valve Open

Network 8: Drain Valve Output (เฉพาะ Draining State)
     M0.3             Q0.1
  ───┤ ├──────────────( )───      ← Drain Valve Open
```

---

## Step 102: Motor Control Patterns {#step-102}

### Pattern 1: Star-Delta Motor Starter

```
ลำดับการทำงาน:
1. กด Start → Motor เริ่มด้วย Star connection (แรงดันต่ำ)
2. หลัง 5 วินาที → เปลี่ยนเป็น Delta connection (แรงดันเต็ม)
3. กด Stop → หยุด

Memory Bits ที่ใช้:
  M1.0 = Motor Running
  M1.1 = Star Active
  M1.2 = Delta Active
  M1.3 = Transition (กำลังเปลี่ยน)
```

```
Network 1: Start/Stop
     I0.0           I0.1            M1.0
  ───┤ ├────┬────────┤/├─────────────( )───
   Start    │        Stop
            │
     M1.0   │
  ───┤ ├────┘


Network 2: Star Phase (เริ่มด้วย Star)
     M1.0           M1.2            M1.1
  ───┤ ├─────────────┤/├─────────────( )───
  Running        Delta Active      Star Active


Network 3: Star Timer (5 วินาที)
     M1.1
  ───┤ ├──────────────[TON T1 5s]─   ← Timer 5 วินาที


Network 4: เปลี่ยนเป็น Delta
     M1.0          T1(Done)          M1.2
  ───┤ ├─────────────┤ ├─────────────( )───
  Running         5sec Timer      Delta Active


Network 5: Outputs
     M1.0             Q0.0
  ───┤ ├──────────────( )───          ← Main Contactor

     M1.1             Q0.1
  ───┤ ├──────────────( )───          ← Star Contactor

     M1.2             M1.1            Q0.2
  ───┤ ├──────────────┤/├────────────( )───
  Delta Active    Star Active       Delta Contactor
```

### Pattern 2: Forward/Reverse Motor Control

```
Memory Bits:
  M2.0 = Forward Running
  M2.1 = Reverse Running

กฎ Safety: ห้าม Forward และ Reverse ทำงานพร้อมกัน!


Network 1: Forward Start
     I1.0           I1.2           M2.1            M2.0
  ───┤ ├────┬────────┤/├────────────┤/├─────────────( )───
  Fwd_Btn   │       Stop_Btn     Rev_Running    Fwd_Running
            │
     M2.0   │
  ───┤ ├────┘


Network 2: Reverse Start
     I1.1           I1.2           M2.0            M2.1
  ───┤ ├────┬────────┤/├────────────┤/├─────────────( )───
  Rev_Btn   │       Stop_Btn     Fwd_Running    Rev_Running
            │
     M2.1   │
  ───┤ ├────┘


Network 3: Outputs (ต้องมี Hardware Interlock ด้วย!)
     M2.0             M2.1            Q1.0
  ───┤ ├──────────────┤/├────────────( )───    ← Forward Contactor
  Fwd_Running      Rev_Running

     M2.1             M2.0            Q1.1
  ───┤ ├──────────────┤/├────────────( )───    ← Reverse Contactor
  Rev_Running      Fwd_Running
```

---

## Step 103: Sequencing ด้วย Memory Bits {#step-103}

### Sequential Control คืออะไร?

ระบบที่ทำงานเป็นลำดับ ทำ Step A เสร็จแล้วจึงไป Step B ทำ B เสร็จแล้วจึงไป C...

### ตัวอย่าง: ระบบ Robot Arm 4 ขั้นตอน

```
Step 1 (M10.0): เคลื่อนไปรับชิ้นงาน (Pick Position)
Step 2 (M10.1): จับชิ้นงาน (Grip)
Step 3 (M10.2): เคลื่อนไปวางชิ้นงาน (Place Position)
Step 4 (M10.3): วางชิ้นงาน (Release)

Conditions:
- Step 1 → 2: Sensor ตรวจพบตำแหน่ง Pick
- Step 2 → 3: Gripper ปิดแล้ว (Pressure switch)
- Step 3 → 4: Sensor ตรวจพบตำแหน่ง Place
- Step 4 → 1: ชิ้นงานวางแล้ว (Time delay)
```

```
Network 1: Initialize
     M8002           M10.0
  ───┤ ├──────────────( S )───     ← เริ่มที่ Step 1


Network 2: Step 1 → Step 2
     M10.0          I5.0           M10.0          M10.1
  ───┤ ├─────────────┤ ├────────────( R )────      ( S )───
  Step_1        Pick_Sensor


Network 3: Step 2 → Step 3
     M10.1          I5.1           M10.1          M10.2
  ───┤ ├─────────────┤ ├────────────( R )────      ( S )───
  Step_2       Grip_Pressure


Network 4: Step 3 → Step 4
     M10.2          I5.2           M10.2          M10.3
  ───┤ ├─────────────┤ ├────────────( R )────      ( S )───
  Step_3        Place_Sensor


Network 5: Step 4 → Step 1 (delay)
     M10.3          T10(Done)      M10.3          M10.0
  ───┤ ├─────────────┤ ├────────────( R )────      ( S )───
  Step_4        Release_Timer


Network 6: Step 4 Timer
     M10.3
  ───┤ ├──────────────[TON T10 1s]─  ← รอ 1 วินาทีหลังวาง


Network 7-10: Outputs ตาม Step
     M10.0             Q5.0
  ───┤ ├──────────────( )───          ← Arm Move to Pick

     M10.1             Q5.1
  ───┤ ├──────────────( )───          ← Gripper Close

     M10.2             Q5.2
  ───┤ ├──────────────( )───          ← Arm Move to Place

     M10.3             Q5.3
  ───┤ ├──────────────( )───          ← Gripper Open
```

---

## Step 104: One-Shot Bit {#step-104}

### One-Shot (Edge Detection) คืออะไร?

**One-Shot Bit** = Bit ที่ ON แค่ **1 Scan** เมื่อมี Rising หรือ Falling Edge

ใช้สำหรับ:
- ตรวจนับของที่ผ่าน sensor
- Toggle mode
- ส่ง command เดียวครั้ง

### วิธีสร้าง Rising Edge One-Shot

```
วิธี 1: ใช้ P (Positive Transition) contact (Mitsubishi/บางแบรนด์)

     X0                M50
  ───┤P├───────────────( )───      ← P = Positive edge contact
  Push Button        One-Shot


วิธี 2: สร้างเอง
     I0.0           M100.0          M50.0
  ───┤ ├─────────────┤/├─────────────( )───
  Push Button    Previous State     One-Shot

     I0.0           M100.0
  ───┤ ├──────────────( )───        ← Update previous state
  Push Button    Previous State

วิธี 3: Siemens CLK Contact
  ─── ┤↑├ ─── (Positive Edge)
  ─── ┤↓├ ─── (Negative Edge)
```

### ตัวอย่าง: นับชิ้นงานบน Conveyor

```
Network 1: ตรวจจับชิ้นงาน (Rising Edge)
     I0.0           M150.0          M151.0
  ───┤ ├─────────────┤/├─────────────( )───
  Part_Sensor     Prev_State       Part_Pulse


Network 2: อัพเดท Previous State
     I0.0           M150.0
  ───┤ ├──────────────( )───
  Part_Sensor     Prev_State


Network 3: นับ
     M151.0
  ───┤ ├──────────────[INC D100]─   ← เพิ่มค่า count ทุกครั้งที่เห็น pulse
  Part_Pulse
```

---

## Step 105: Diagnostic Bits {#step-105}

### ความสำคัญของ Diagnostic Bits

Diagnostic Bits ใช้สำหรับ **บันทึกสถานะ** ของระบบเพื่อ:
- ช่วย Maintenance ในการหา Fault
- แสดงผลบน HMI
- ส่งข้อมูลไปยัง SCADA

### ระบบ Diagnostic Bits ที่ดี

```
สร้าง Fault Register โดยใช้ Bit ต่างๆ:

M100.0 = Overtemperature
M100.1 = Overpressure
M100.2 = Low Level
M100.3 = High Level
M100.4 = Motor Overload
M100.5 = Valve Fault
M100.6 = Communication Error
M100.7 = General Fault (OR ของทั้งหมด)
```

```
Network 1: Temperature Fault
     I8.0              M100.0
  ───┤ ├────────────────( )───      ← Overtemp sensor input
  Temp_High

Network 2: Pressure Fault
     I8.1              M100.1
  ───┤ ├────────────────( )───      ← Overpressure sensor
  Press_High

Network 3: Motor Overload
     I8.4              M100.4
  ───┤ ├────────────────( )───      ← Motor OL relay
  Motor_OL

Network 4: General Fault (OR ทุก Fault)
     M100.0            M100.7
  ───┤ ├────────────────( )───

     M100.1            M100.7
  ───┤ ├────────────────( )───

     M100.4            M100.7
  ───┤ ├────────────────( )───

Network 5: Fault Indicator (กะพริบ 1 Hz)
     M100.7          M8013            Q0.7
  ───┤ ├──────────────┤ ├──────────────( )───
  Any Fault        1sec Clock       Fault Lamp
```

---

## Step 106: Communication Flags {#step-106}

### ใช้ Memory Bits เป็น Communication Interface

ในระบบ PLC ที่ติดต่อกับ HMI, SCADA หรือ PLC อื่น Memory Bits ทำหน้าที่เป็น "ตัวกลาง"

```
SCADA Write → M Bits → PLC Logic → Output

PLC Logic → M Bits → SCADA Read
```

### ตัวอย่าง: ช่องทางสื่อสาร HMI-PLC

```
HMI Commands (เขียนจาก HMI):
  M200.0 = HMI_Start_Command
  M200.1 = HMI_Stop_Command
  M200.2 = HMI_Reset_Command
  M200.3 = HMI_Auto_Select
  M200.4 = HMI_Manual_Select

PLC Status (อ่านจาก HMI):
  M210.0 = PLC_Running
  M210.1 = PLC_Fault
  M210.2 = PLC_Auto_Mode
  M210.3 = PLC_Manual_Mode
  M210.4 = Motor1_Running
  M210.5 = Motor2_Running


Network 1: ประมวลผล HMI Start Command
     M200.0          M210.1          M50.0
  ───┤ ├──────────────┤/├─────────────( S )───
  HMI_Start       PLC_Fault       Motor_Run_Flag

Network 2: ประมวลผล HMI Stop Command
     M200.1             M50.0
  ───┤ ├────────────────( R )───
  HMI_Stop

Network 3: อัพเดท Status สำหรับ HMI
     M50.0             M210.4
  ───┤ ├────────────────( )───      ← บอก HMI ว่า Motor กำลังทำงาน
  Motor_Running

Network 4: Clear HMI Commands หลัง Process
     M200.0             M200.0
  ───┤ ├────────────────( R )───    ← Auto-clear command หลัง 1 scan
  HMI_Start_Cmd    (reset itself)
```

---

## Step 107: Alarm Management ด้วย Memory Bits {#step-107}

### ระบบ Alarm ที่ดี

```
Alarm State Machine:
  State 0: Normal (ไม่มี Alarm)
  State 1: Alarm Active (เกิด Fault)
  State 2: Alarm Acknowledged (ผู้ดูแลรับทราบ)
  State 3: Alarm Reset (Fault หายแล้ว, รอ Reset)
```

```
Network 1: Detect Alarm Condition
     I9.0              M300.0
  ───┤ ├────────────────( S )───    ← Alarm Active
  Temperature_High

Network 2: Operator Acknowledge
     M300.0          I9.5           M300.1
  ───┤ ├──────────────┤ ├────────────( S )───
  Alarm_Active    ACK_Button      Alarm_Acknowledged

Network 3: Alarm Reset (fault gone + reset button)
     M300.0          I9.0            I9.6            M300.0
  ───┤ ├──────────────┤/├─────────────┤ ├─────────────( R )───
  Alarm_Active   Temp_High       Reset_Button

     M300.0             M300.1
  ───┤ ├────────────────( R )───    ← Reset Ack bit ด้วย


Network 4: Alarm Horn (กะพริบจนกว่าจะ Acknowledge)
     M300.0          M300.1         M8013            Q9.0
  ───┤ ├──────────────┤/├────────────┤ ├──────────────( )───
  Alarm_Active    Not_Acked       1sec_Clk         Alarm_Horn


Network 5: Alarm Light (ค้างหลัง Acknowledge)
     M300.0             Q9.1
  ───┤ ├────────────────( )───      ← Alarm Light (steady)
  Alarm_Active
```

---

## Step 108: Recipe Bits {#step-108}

### Recipe Management

ระบบ Recipe ใช้ Memory Bits เลือก "สูตร" การผลิต แต่ละสูตรมีค่าพารามิเตอร์ต่างกัน

```
Recipe Bits:
  M400.0 = Recipe 1 Selected (Product A)
  M400.1 = Recipe 2 Selected (Product B)
  M400.2 = Recipe 3 Selected (Product C)


Network 1: Select Recipe 1
     I10.0             M400.1          M400.2          M400.0
  ───┤ ├───────────────┤/├─────────────┤/├──────────────( )───
  Recipe1_Btn      Recipe2_Active  Recipe3_Active   Recipe1_Active

Network 2: Select Recipe 2
     I10.1             M400.0          M400.2          M400.1
  ───┤ ├───────────────┤/├─────────────┤/├──────────────( )───
  Recipe2_Btn      Recipe1_Active  Recipe3_Active   Recipe2_Active

Network 3: Load Recipe 1 Parameters
     M400.0
  ───┤ ├──────────────[MOV K100 D500]─   ← Temperature setpoint = 100°C
  Recipe1         [MOV K30 D501]─        ← Time setpoint = 30 min
                  [MOV K200 D502]─       ← Pressure setpoint = 200 kPa

Network 4: Load Recipe 2 Parameters
     M400.1
  ───┤ ├──────────────[MOV K150 D500]─   ← Temperature setpoint = 150°C
  Recipe2         [MOV K45 D501]─        ← Time setpoint = 45 min
                  [MOV K250 D502]─       ← Pressure setpoint = 250 kPa
```

---

## Step 109: Advanced State Tracking {#step-109}

### Multi-Level State Tracking

สำหรับระบบซับซ้อนที่มีหลาย state layer

```
System States:
  Layer 1 (M bit 0-3): System Mode
    M500.0 = Idle
    M500.1 = Running
    M500.2 = Alarm
    M500.3 = Maintenance

  Layer 2 (M bit 4-7): Production State (ใน Running Mode)
    M500.4 = Initializing
    M500.5 = Processing
    M500.6 = Completing
    M500.7 = Waiting

  Layer 3: Individual Equipment Status
    M501.0 = Feeder_Running
    M501.1 = Mixer_Running
    M501.2 = Conveyor_Running
    ...
```

### Cascade Shutdown

```
เมื่อเกิด Emergency → หยุดทุกอย่างตามลำดับ

Network 1: Emergency Detected
     I15.0             M600.0
  ───┤ ├────────────────( S )───    ← Emergency Active
  E_Stop

Network 2: Cascade Stop - หยุด Feeder ก่อน
     M600.0             M501.0
  ───┤ ├────────────────( R )───    ← Stop Feeder first

Network 3: หยุด Mixer หลังจาก Feeder หยุด
     M600.0          M501.0          M501.1
  ───┤ ├──────────────┤/├────────────( R )───
  Emergency      Feeder_Running   Stop Mixer

Network 4: หยุด Conveyor ท้ายสุด (ให้ชิ้นงานผ่านก่อน)
     M600.0          M501.1          T20(Done)        M501.2
  ───┤ ├──────────────┤/├─────────────┤ ├────────────( R )───
  Emergency      Mixer_Running    5sec_Delay      Stop Conveyor

Network 5: Delay Timer สำหรับ Conveyor
     M600.0          M501.1
  ───┤ ├──────────────┤/├───────────[TON T20 5s]─
  Emergency      Mixer_Stopped
```

---

## Step 110: Workshop Project - Motor Control System {#step-110}

### โจทย์: ระบบควบคุมมอเตอร์ปั๊มน้ำ

**ความต้องการ:**
1. ระบบมีปั๊ม 2 ตัว (Main Pump + Standby Pump)
2. ทำงานแบบ Auto: เปิดเมื่อระดับน้ำต่ำ ปิดเมื่อสูง
3. ทำงานแบบ Manual: ควบคุมโดยตรงจาก Operator
4. Standby Pump เปิดอัตโนมัติเมื่อ Main Pump เสีย
5. มีระบบ Alarm เมื่อ Motor Overload

### I/O List

```
┌─────────────┬─────────────────────────────────────────┐
│ Address     │ Description                             │
├─────────────┼─────────────────────────────────────────┤
│ Inputs      │                                         │
│ I0.0        │ Main Pump Start Button (Local)          │
│ I0.1        │ Main Pump Stop Button (Local)           │
│ I0.2        │ Standby Pump Start Button               │
│ I0.3        │ Standby Pump Stop Button                │
│ I0.4        │ Mode Switch (Auto=1 / Manual=0)         │
│ I0.5        │ Emergency Stop                          │
│ I1.0        │ Low Level Sensor (ON = Low)             │
│ I1.1        │ High Level Sensor (ON = High)           │
│ I1.2        │ Main Pump Overload Relay                │
│ I1.3        │ Standby Pump Overload Relay             │
├─────────────┼─────────────────────────────────────────┤
│ Outputs     │                                         │
│ Q0.0        │ Main Pump Motor Contactor               │
│ Q0.1        │ Standby Pump Motor Contactor            │
│ Q0.4        │ Auto Mode Indicator                     │
│ Q0.5        │ Main Pump Running Indicator             │
│ Q0.6        │ Standby Pump Running Indicator          │
│ Q0.7        │ Alarm Lamp                              │
├─────────────┼─────────────────────────────────────────┤
│ Memory Bits │                                         │
│ M0.0        │ Auto Mode Flag                          │
│ M0.1        │ Main Pump Run Flag                      │
│ M0.2        │ Standby Pump Run Flag                   │
│ M1.0        │ Main Pump Fault                         │
│ M1.1        │ Standby Pump Fault                      │
│ M1.7        │ Any Fault                               │
└─────────────┴─────────────────────────────────────────┘
```

### โปรแกรมสมบูรณ์

```
//==========================================================
// Dual Pump Control System
// Author: [Your Name]
// Date: [Date]
// Description: Auto/Manual pump control with standby backup
//==========================================================


// --- Mode Selection ---

Network 1: Auto Mode Flag
     I0.4               M0.0
  ───┤ ├─────────────────( )───     ← M0.0 = Auto Mode (follows switch)
  Auto_Switch


// --- Auto Mode Control ---

Network 2: Auto Start Main Pump (ระดับน้ำต่ำ + Auto Mode)
     M0.0            I1.0           M1.0            M0.1
  ───┤ ├──────────────┤ ├────────────┤/├─────────────( S )───
  Auto_Mode       Low_Level      Main_Fault      Main_Pump_Run

Network 3: Auto Stop Main Pump (ระดับน้ำสูง)
     M0.0            I1.1           M0.1
  ───┤ ├──────────────┤ ├────────────( R )───
  Auto_Mode       High_Level     Main_Pump_Run


// --- Manual Mode Control ---

Network 4: Manual Start Main Pump
     M0.0             I0.0           I0.1            M1.0            M0.1
  ───┤/├──────────┬────┤ ├────────────┤/├─────────────┤/├─────────────( )───
  Not_Auto       │    Start         Stop          Main_Fault
                 │
     M0.1        │
  ───┤ ├──────────┘    ← Seal circuit


// --- Standby Pump Auto Activation ---

Network 5: Activate Standby when Main Fails
     I1.0           M1.0           M1.1             M0.2
  ───┤ ├─────────────┤ ├────────────┤/├──────────────( S )───
  Low_Level    Main_Fault     Standby_Fault     Standby_Run

Network 6: Stop Standby when level OK
     I1.1              M0.2
  ───┤ ├────────────────( R )───
  High_Level        Standby_Run


// --- Fault Detection ---

Network 7: Main Pump Fault
     I1.2              M0.1            M1.0
  ───┤ ├───────────────┤ ├─────────────( S )───
  Main_OL_Relay    Main_Running     Main_Fault

     I1.2              M0.1
  ───┤ ├───────────────┤ ├────────────[RST M0.1]─  ← หยุด main pump ด้วย
  Main_OL_Relay    Main_Running

Network 8: Standby Pump Fault
     I1.3              M0.2            M1.1
  ───┤ ├───────────────┤ ├─────────────( S )───
  Stby_OL_Relay   Stby_Running    Standby_Fault


// --- Emergency Stop ---

Network 9: E-Stop All
     I0.5              M0.1
  ───┤ ├────────────────( R )───   ← Stop main

     I0.5              M0.2
  ───┤ ├────────────────( R )───   ← Stop standby
  E_Stop


// --- Outputs ---

Network 10: Main Pump Output
     M0.1           I0.5           Q0.0
  ───┤ ├─────────────┤/├────────────( )───
  Main_Run       E_Stop         Main_Contactor

Network 11: Standby Pump Output
     M0.2           I0.5           Q0.1
  ───┤ ├─────────────┤/├────────────( )───
  Stby_Run       E_Stop         Stby_Contactor


// --- Indicators ---

Network 12: Auto Mode Indicator
     M0.0             Q0.4
  ───┤ ├──────────────( )───

Network 13: Main Pump Running Indicator
     M0.1             Q0.5
  ───┤ ├──────────────( )───

Network 14: Standby Running Indicator
     M0.2             Q0.6
  ───┤ ├──────────────( )───

Network 15: Any Fault Flag
     M1.0              M1.7
  ───┤ ├───────────────( )───

     M1.1              M1.7
  ───┤ ├───────────────( )───

Network 16: Alarm Lamp (กะพริบ 1 Hz)
     M1.7            M8013            Q0.7
  ───┤ ├──────────────┤ ├──────────────( )───
  Any_Fault       1sec_Clock       Alarm_Lamp
```

### Lab Exercises

**Lab 1:** เพิ่มฟีเจอร์ "Pump Alternation" - สลับการทำงานของปั๊มทั้ง 2 ตัว ทุก 24 ชั่วโมง เพื่อให้ใช้งานสม่ำเสมอ

**Lab 2:** เพิ่ม HMI Communication Bits สำหรับ SCADA monitoring

**Lab 3:** เพิ่มระบบนับ Running Hours ของแต่ละปั๊ม (ใช้ D Register แบบ Retentive)

**Lab 4:** ทดสอบโปรแกรมใน Simulator โดยจำลอง:
- ระดับน้ำต่ำ → Main Pump เปิด
- ระดับน้ำสูง → Main Pump ปิด
- Main Pump Overload → Standby เปิดอัตโนมัติ

---

## สรุป Part 06

```
┌─────────────────────────────────────────────────────────┐
│ สิ่งที่เรียนรู้ใน Part 06:                              │
│                                                         │
│ ✓ Memory Bits / Internal Coils และการใช้งาน             │
│ ✓ Retentive Memory - จำข้อมูลหลัง Power Loss            │
│ ✓ First Scan Bit - Initialize ระบบ                      │
│ ✓ System Flags - Clock, Status, Error bits              │
│ ✓ Flag Bit Patterns - เทคนิคการใช้ bits อย่างมีระบบ     │
│ ✓ Latch Patterns - Self-Seal และ Set/Reset               │
│ ✓ State Machine - ควบคุมระบบแบบ state-based             │
│ ✓ Motor Control Patterns - Star-Delta, Fwd/Rev           │
│ ✓ One-Shot Bits - Edge detection                         │
│ ✓ Diagnostic, Communication, Alarm Bits                  │
│ ✓ Complete Dual Pump Control System                      │
└─────────────────────────────────────────────────────────┘
```

---

**[← Part 05: Counters](part05_counters.md)** | **[Part 07: Math Operations →](part07_math_operations.md)**
