# Part 18: Mitsubishi FX3U/FX5U/Q Series - GX Works3 และระบบ Address ของ Mitsubishi

## Steps 331-350: มาตรฐาน PLC ยอดนิยมในเอเชีย

**[← Part 17: Allen-Bradley](part17_allen_bradley.md) | [Part 19: OMRON →](part19_omron.md)**

---

## บทนำ: Mitsubishi ในอุตสาหกรรมไทย

Mitsubishi Electric PLC เป็นที่นิยมอย่างมากในประเทศไทยและเอเชียตะวันออกเฉียงใต้ ด้วยความเชื่อถือได้สูง ราคาเหมาะสม และ Support จากตัวแทนในไทยที่ครอบคลุม

**สาย FX (Factory Automation Compact):**
- FX1S/FX1N: Entry-level
- FX2N: รุ่นยอดนิยมในอดีต
- FX3U/FX3G: รุ่นปัจจุบัน (กำลังถูกแทนที่)
- **FX5U**: รุ่นใหม่ล่าสุด, Built-in EtherNet, SD Card

**สาย Q (MELSEC-Q):**
- Q00/Q01/Q02: Mid-range
- Q06/Q13: Large-scale
- QnA/QnH: High-performance

---

## Step 331: การติดตั้ง GX Works3

### Software Overview

```
GX Works3 รองรับ:
- FX5U (แนะนำ)
- MELSEC iQ-R (Flagship)
- MELSEC iQ-F (FX5 Series)

GX Works2 รองรับ:
- FX3U/FX3G
- MELSEC-Q
- MELSEC-L

System Requirements GX Works3:
- OS: Windows 8.1/10/11 (64-bit)
- RAM: 2 GB (4 GB แนะนำ)
- HDD: 3.2 GB
- Display: 1024×768 ขั้นต่ำ
```

### หน้าจอหลัก GX Works3

```
╔══════════════════════════════════════════════════════════════╗
║  GX Works3 - [Project_Name]                                 ║
╠════════════╦═════════════════════════════╦══════════════════╣
║ Navigation ║ Program Editor              ║ Component       ║
║ Window     ║                             ║ Library         ║
║            ║ [Ladder Editor]             ║                 ║
║ ▼ Project  ║                             ║ ▼ FB Library    ║
║   ▼ Param  ║ Network 1:                  ║   Timer         ║
║     PLC    ║ ──┤X0├──┤/X1├──( Y0 )──    ║   Counter       ║
║     Net    ║                             ║   Math          ║
║   ▼ PLC    ║ Network 2:                  ║                 ║
║     Main   ║ ──┤Y0├─[TON]──( M0 )──     ║                 ║
║     Sub    ║                             ║                 ║
║   ▼ Device ║                             ║                 ║
║     Mem Map║                             ║                 ║
╚════════════╩═════════════════════════════╩══════════════════╝
```

---

## Step 332: FX5U Hardware Overview

### ตระกูล FX5U CPU

```
FX5U CPU Variants:
┌──────────────┬────────────────────────────────────────┐
│ Model        │ คุณสมบัติ                               │
├──────────────┼────────────────────────────────────────┤
│ FX5U-32MT/ES │ 16 DI / 16 DO (Transistor Source)     │
│              │ 2 High-speed Counter (200kHz)           │
│              │ Built-in EtherNet, RS-485              │
├──────────────┼────────────────────────────────────────┤
│ FX5U-32MR/ES │ 16 DI / 16 DO (Relay Output)          │
│              │ เหมาะกับ Load AC                        │
├──────────────┼────────────────────────────────────────┤
│ FX5U-64MT/ES │ 32 DI / 32 DO (Transistor)            │
│              │ 6 High-speed Counter                   │
├──────────────┼────────────────────────────────────────┤
│ FX5U-80MT/ES │ 40 DI / 40 DO (Transistor)            │
│              │ รองรับ SSCNET III Motion               │
└──────────────┴────────────────────────────────────────┘

หมายเหตุ:
MT = Transistor Output (Source = PNP Output)
MR = Relay Output
ES = ไฟฟ้า AC/DC หลายระดับ
```

### ส่วนประกอบของ FX5U

```
FX5U-64MT/ES Layout:

┌───────────────────────────────────────────────────────────┐
│ FX5U-64MT/ES                                              │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ [Status LED: POWER, RUN, ERR, P.RUN, BAT, USER]     │  │
│ └──────────────────────────────────────────────────────┘  │
│                                                           │
│ INPUTS (X):          OUTPUTS (Y):                        │
│ X0 X1 X2 X3         Y0 Y1 Y2 Y3                         │
│ X4 X5 X6 X7    24V  Y4 Y5 Y6 Y7                         │
│ X10 X11 X12 X13     Y10 Y11 Y12 Y13                      │
│ X14 X15 X16 X17  L  Y14 Y15 Y16 Y17                      │
│ X20 X21 X22 X23     Y20 Y21 Y22 Y23                      │
│ X24 X25 X26 X27  N  Y24 Y25 Y26 Y27                      │
│                                                           │
│ [USB-B]  [SD CARD]  [ETHERNET]  [RS-485 (CH1)]           │
└───────────────────────────────────────────────────────────┘

Extension Modules (ต่อทางขวา):
FX5-8EX/ES  : 8 DI expansion
FX5-8EYT/ES : 8 DO expansion
FX5-4AD     : 4 Analog Input
FX5-4DA     : 4 Analog Output
```

---

## Step 333: FX Addressing System

### ระบบ Address ของ Mitsubishi FX

```
Mitsubishi ใช้ Octal Number System สำหรับ X/Y!

Digital Inputs (X): ใช้เลข Octal
X0, X1, X2, X3, X4, X5, X6, X7
X10, X11, X12, X13, X14, X15, X16, X17  (ไม่มี X8, X9!)
X20, X21, ...

Digital Outputs (Y): ใช้เลข Octal
Y0, Y1, Y2, Y3, Y4, Y5, Y6, Y7
Y10, Y11, ... (ไม่มี Y8, Y9!)
Y20, Y21, ...

ทำไม Octal?:
เดิม Mitsubishi ออกแบบบน 8-bit boundary
8 Inputs ต่อ Module = X0-X7 (Octal)
Module 2 = X10-X17 (ไม่ใช่ X8-X15)

จำง่ายๆ: หลัก Ones ไม่มี 8 และ 9 (0-7 เท่านั้น)
```

### ตารางสรุป Device ทั้งหมด

```
FX5U Device Map:

Device  | Symbol | Count  | Description
--------|--------|--------|---------------------------
Input   | X      | X0-X177| Physical Digital Input
Output  | Y      | Y0-Y177| Physical Digital Output
Aux Relay| M     | M0-M7999| Internal (Bit)
Special M| M     | M8000+ | Special Relay (ดูด้านล่าง)
State   | S      | S0-S4095| State relay (SFC)
Timer   | T      | T0-T511| Timer (100ms, 10ms, 1ms)
Counter | C      | C0-C255| Counter (16-bit, 32-bit)
Data Reg | D     | D0-D7999| Data Register (16-bit Word)
Special D| D     | D8000+ | Special Register
File Reg | R     | R0-R32767| File/Retention Register
Index Reg| V,Z   | V0-V7  | Index Register

หมายเหตุ FX5U เพิ่ม:
Long Register: LR0-LR32767 (32-bit)
Long Aux Relay: LM0-LM32767 (Bit)
```

---

## Step 334: Basic Instructions ใน Mitsubishi

### Contact และ Coil Instructions

```
Instruction set ของ Mitsubishi ใช้ Mnemonic ที่เป็นเอกลักษณ์:

Contact Instructions:
LD   X0   : Load Contact (เริ่ม Rung ด้วย NO Contact)
LDI  X1   : Load Inverse (เริ่ม Rung ด้วย NC Contact)
AND  X2   : AND Contact (NO)
ANI  X3   : AND Inverse (NC)
OR   M0   : OR Contact (NO)
ORI  M1   : OR Inverse (NC)

Coil Instructions:
OUT  Y0   : Output Coil
SET  M0   : Set (Latch)
RST  M0   : Reset (Unlatch)
PLS  M10  : Pulse on Rising Edge (1 Scan)
PLF  M11  : Pulse on Falling Edge (1 Scan)

ตัวอย่าง Motor Control:
LD   X0      // Load Start_PB
OR   Y0      // OR Motor_Running (Seal-In)
ANI  X1      // AND NOT Stop_PB
ANI  M100    // AND NOT Fault_Latch
OUT  Y0      // Output Motor Coil

LD   X5      // Load Fault Sensor
SET  M100    // Latch Fault
LD   X6      // Load Reset PB
RST  M100    // Reset Fault

ใน LAD (GX Works3 แสดงเป็น Diagram):
   X0     X1     M100        Y0
──┤├──┬──┤/├────┤/├──────────( )──
      │
   Y0
──┤├──┘
```

---

## Step 335: Timer Instructions ใน Mitsubishi

### ประเภท Timer

```
Mitsubishi มี Timer หลายประเภท:

T0-T199  : 100ms Timer (ค่าสูงสุด 3276.7 วินาที)
T200-T245: 10ms Timer  (ค่าสูงสุด 327.67 วินาที)
T246-T249: 1ms Timer   (ค่าสูงสุด 32.767 วินาที)
T250-T255: 100ms Accumulating Timer (TANS)
T256-T511: 1ms Timer (FX3U เท่านั้น)

FX5U Timer:
ST□  : Low-speed Timer (Variable Resolution)
LCT□ : Long Counter Timer

การใช้ Timer:
OUT  T0  K50   // เปิด T0 นาน 50 × 100ms = 5 วินาที
                // K = Decimal Constant

LD   T0         // Contact Timer T0
OUT  Y1         // Output เมื่อ Timer ครบ

ตัวอย่างใน LAD:
// 5 วินาที Start Delay

   X0                  T0
──┤├───────────────────(T0 K50)──  // Coil Timer

   T0              Y0
──┤├───────────────( )──           // Contact Timer
```

### ตัวอย่างการใช้ Timer

```
// โปรแกรม: Motor Y-Δ Starting (Star-Delta Starter)
// Y0 = Contactor หลัก
// Y1 = Contactor Star (Y)
// Y2 = Contactor Delta (Δ)
// T0 = เวลา Star 5 วินาที
// T1 = Transition delay 0.5 วินาที

// Step 1: กด Start → เปิด Contactor หลัก + Star
   X0(Start)  X1(Stop)  M0(Fault)    Y0 (Main)
──┤├──────┬───┤/├──────┤/├───────────( )──
          │
   Y0 (Seal-in)
──┤├──────┘

// Step 2: Y0 เปิด → เริ่มนับเวลา Star
   Y0                  T0
──┤├────────────────────(T0 K50)──  // 5 วินาที

// Step 3: Star Contactor ทำงานระหว่างนับ (T0=OFF หมายถึงยังไม่ครบ)
   Y0   T0(Star phase)   Y2(No Delta)    Y1 (Star)
──┤├───┤/├──────────────┤/├──────────────( )──

// Step 4: Transition Timer (ป้องกัน Both ON)
   T0                  T1
──┤├────────────────────(T1 K5)──   // 0.5 วินาที

// Step 5: Delta ทำงาน
   Y0   T1             Y1(No Star)    Y2 (Delta)
──┤├───┤├─────────────┤/├──────────────( )──
```

---

## Step 336: Counter Instructions ใน Mitsubishi

### 16-bit vs 32-bit Counter

```
C0-C99   : 16-bit Up Counter  (0 ถึง 32767)
C100-C199: 16-bit Latched Counter (Keep value หลัง Power Off)
C200-C219: 32-bit Up Counter  (-2,147,483,648 ถึง +2,147,483,647)
C220-C234: 32-bit Latched Counter
C235-C255: High-speed Counter (ใช้ Hardware Counter)

การใช้ Counter (16-bit):
OUT  C0  K100    // Counter ค่าสูงสุด 100

LD   X0
OUT  C0  K0      // การนับ (ทุก Rising Edge ของ X0)

LD   C0          // Contact เมื่อนับครบ 100
OUT  Y0

LD   X1          // Reset Counter
RST  C0

ตัวอย่างใน LAD:
   X0           (C0 K100)   // Coil Counter นับทุก Rising Edge X0
──┤├─────────────────────

   X1
──┤├──────────────(RST C0)  // Reset

   C0          Y0
──┤├────────────( )──       // Output เมื่อนับครบ
```

### High-speed Counter (HSC)

```
FX5U High-speed Counter:
C235-C255 เชื่อมกับ Input พิเศษ

ความเร็วสูงสุด:
- C235 (1-phase): ใช้ X0 ถึง 200kHz
- C244 (2-phase AB): ใช้ X0,X1 ถึง 200kHz

การตั้งค่า:
1. GX Works3 → Navigation → Parameter → FX5UCPU
2. Built-in Input → High Speed I/O
3. เปิด High-speed Counter C235
4. กำหนด Input: X0

โปรแกรมใช้ HSC:
// อ่านค่า Encoder
DMOV  C235  D100    // คัดลอกค่า 32-bit จาก C235 ไป D100

// เปรียบเทียบ Position
LD>=D  D100  D200   // เปรียบเทียบ 32-bit
OUT  Y0

// Reset Counter ที่ Home Position
LD   X10
DMOV  K0  C235      // Reset Counter เป็น 0
```

---

## Step 337: Special M Relays (M8000-M8511)

### Special Relays ที่สำคัญ

```
Special M Relay ใน Mitsubishi FX:
(อย่าเขียนทับ M8000-M8511 นอกจากจะรู้ว่ากำลังทำอะไร!)

M8000: RUN Monitor (TRUE ตลอดเมื่อ PLC RUN)
M8001: Inverted RUN Monitor (FALSE ตลอดเมื่อ PLC RUN)
M8002: Initial Pulse (TRUE เฉพาะ Scan แรก)
M8003: Initial Pulse Inverted (FALSE เฉพาะ Scan แรก)
M8004: Error Occurrence (TRUE เมื่อมี Error)
M8005: Battery Voltage Low (TRUE เมื่อ Battery ต่ำ)
M8006: Battery Voltage Low Latch
M8011: 10ms Clock Pulse (ช่วง 5ms ON, 5ms OFF)
M8012: 100ms Clock Pulse (50ms ON, 50ms OFF)
M8013: 1s Clock Pulse (0.5s ON, 0.5s OFF)
M8014: 1min Clock Pulse
M8020: Zero Flag (เมื่อ Math Result = 0)
M8021: Borrow Flag (เมื่อ Math ต่ำกว่า 0)
M8022: Carry Flag (เมื่อ Math เกิน Range)
M8029: Instruction Complete Flag
M8034: All Outputs OFF
M8040: Transfer Inhibit (Prohibit STL state change)
M8044: Zero Return Complete (Home Return)
M8200: High-speed Counter Direction

การใช้ M8002 (Initial Pulse):
// ตั้งค่าเริ่มต้นเมื่อ PLC เริ่มทำงาน

   M8002              D100
──┤├──────────[MOV K1500]──  // ตั้ง Default Speed = 1500

   M8002              D200
──┤├──────────[MOV K80]──    // ตั้ง Temperature SP = 80
```

### ตัวอย่างใช้ Clock Pulse

```
// โปรแกรม: ไฟ Blink อัตโนมัติ

// ไฟกะพริบ 1Hz (0.5s ON, 0.5s OFF)
   M8013             Y5 (Blink_Light)
──┤├───────────────────( )──

// ไฟกะพริบ 10Hz (เมื่อ Alarm Active)
   M8011   Alarm_Active    Y6 (Fast_Blink)
──┤├────────────┤├─────────────( )──

// โปรแกรม: Watchdog ด้วย Clock Pulse
// ถ้า PLC ยังทำงานปกติ M8013 จะกะพริบ
// ถ้า CPU Halt → M8013 หยุด → Watchdog Reset
   M8013              WD_Output
──┤├───────────────────( )──   // ต่อกับ External WD Circuit
```

---

## Step 338: Applied Instructions ใน Mitsubishi

### MOV - Data Transfer

```
MOV  : Move 16-bit Word
DMOV : Move 32-bit Double Word
BMOV : Block Move (หลาย Word พร้อมกัน)
FMOV : Fill Move (เติมค่าเดิมทุก Element)

การใช้งาน:
LD   X0
MOV  D0   D10     // คัดลอก D0 ไป D10

LD   X1
DMOV D100  D200   // คัดลอก 32-bit (D100+D101 → D200+D201)

LD   X2
BMOV D0   D100  K10  // คัดลอก D0-D9 ไป D100-D109

LD   X3
FMOV K0   D200  K50  // Set D200-D249 = 0

// ตัวอย่างใน LAD:
   X0              D10
──┤├─────[MOV D0]──

   X1              D200
──┤├────[DMOV D100]──
```

### Arithmetic Instructions

```
ADD, SUB, MUL, DIV: คณิตศาสตร์ 16-bit
DADD, DSUB, DMUL, DDIV: คณิตศาสตร์ 32-bit

// บวก:
LD   X0
ADD  D0   D1   D2    // D2 = D0 + D1

// คูณ 16-bit (ผลลัพธ์ 32-bit!):
LD   X0
MUL  D0   D1   D10   // D11:D10 = D0 × D1

// หาร 16-bit:
LD   X0
DIV  D0   D1   D10   // D10 = D0 ÷ D1 (Quotient)
                     // D11 = D0 MOD D1 (Remainder)

// ตัวอย่าง: แปลง Raw AI เป็น Physical Unit
// Raw: 0-4000 (4-20mA = 0-400°C)
// Temp = (Raw × 400) / 4000 = Raw / 10

LD   M8000
MOV  D1000  D100   // D100 = AI Raw Value
MUL  D100   K400  D200  // D201:D200 = Raw × 400
DDIV D200   K4000 D300  // D300 = (Raw × 400) / 4000
MOV  D300   D1001  // D1001 = Temperature (°C × 10)
```

---

## Step 339: Comparison และ Conditional Instructions

### Comparison Instructions

```
CMP  : Compare 16-bit
DCMP : Compare 32-bit
ZCP  : Zone Compare (Range Check)

// CMP ใช้ Flag M8020 (=), M8021 (<), M8022 (>)
LD   X0
CMP  D100  K80      // เปรียบเทียบ D100 กับ 80
                    // ถ้า D100 = 80 → M8020=1
                    // ถ้า D100 < 80 → M8021=1
                    // ถ้า D100 > 80 → M8022=1

// แต่ละ CMP ต้องระวัง: Flag ถูกแทนทับทุก Scan!
// แนะนำใช้ LD= แทน

// Comparison Contact (In-Line) แนะนำกว่า:
LD=  D100  K80       // D100 = 80?
OUT  Y0

LDK> D100  K80       // D100 > 80?
OUT  Y1

AND<= D200  K100     // AND D200 <= 100
OUT  Y2

// Zone Compare:
LD   X0
ZCP  K50  K100  D0   // 50 <= D0 <= 100?
                     // M เปิดตาม Zone
```

### Conditional Jump Instructions

```
CJ  : Conditional Jump
JMP : Unconditional Jump

// ใช้ Label P□ เป็นจุด Jump ไป
// ไม่แนะนำใช้มาก (อาจทำให้ Code อ่านยาก)

LD   X0
CJ   P10    // ถ้า X0 = TRUE ข้ามไป P10

// Code ที่จะข้ามถ้า X0=TRUE:
OUT  Y1     // จะไม่ทำงานถ้า X0=TRUE
...

P10:        // Label
OUT  Y2     // ทำงานเสมอ

// ใช้สำหรับ: โหมด Manual/Auto Switch
   M10 (Manual)
──┤├──────────────[CJ P100]  // ข้ามไป Auto Section

// Manual Mode Code:
...
P100:
// Auto Mode Code:
...
```

---

## Step 340: FX5U New Features

### Ethernet Communication

```
FX5U มี Built-in EtherNet (10/100Mbps)
ไม่ต้องซื้อ Communication Module เพิ่ม!

การตั้งค่า Ethernet ใน GX Works3:
1. Navigation → Parameter → FX5UCPU → Module Parameter
2. Ethernet Port Setting:
   - IP Address: 192.168.3.10
   - Subnet Mask: 255.255.255.0
   - Default Gateway: 192.168.3.1
3. Communication Setting:
   - SLMP Protocol: Port 5007
   - Modbus/TCP: Port 502

SLMP (Seamless Message Protocol):
- Mitsubishi Protocol สำหรับ Read/Write Device
- ใช้กับ HMI (GOT), SCADA, PC ผ่าน Ethernet
- อ่าน/เขียน X, Y, M, T, C, D ได้

Modbus/TCP Support (Built-in):
FX5U ทำหน้าที่เป็น Modbus Slave
HMI/SCADA ต่อผ่าน Modbus TCP ได้ทันที
```

### SD Memory Card

```
FX5U มี SD Card Slot (SDHC ถึง 32GB)

การใช้งาน:
1. Data Logging: บันทึก D Register ลง CSV
2. Recipe Management: โหลด Recipe จาก CSV
3. Program Backup: เก็บ Program สำรอง
4. Event Logging: บันทึก Alarm History

GX Works3 → Navigation → Parameter → FX5UCPU
→ Memory Card Setting:
  ✓ Enable data logging
  ✓ Sampling interval: 1000ms
  ✓ Devices to log: D100-D110

Data Logging Format (CSV):
Date,Time,D100,D101,D102,...
2024-01-15,08:30:00,1500,75,1234,...
2024-01-15,08:30:01,1502,74,1234,...
```

### Built-in Analog

```
FX5U-32MT/ES มี Built-in Analog (ไม่ต้องซื้อ Module):
- 2 Analog Input: 0-10V หรือ 4-20mA
- กำหนด Input Type ใน Parameter
- ค่า Raw: 0-27648 (0V-10V หรือ 4mA-20mA)

การอ่านค่า Built-in AI:
// Input 1 อยู่ที่ SD6020 (Special D)
   M8000
──┤├──────[MOV SD6020  D100]  // อ่าน AI1 ไปที่ D100

// Input 2 อยู่ที่ SD6021
   M8000
──┤├──────[MOV SD6021  D101]  // อ่าน AI2 ไปที่ D101

// แปลงเป็น Engineering Unit (0-10V → 0-100.0 Process)
// D100 = Raw (0-27648) → D200 = Process (0-1000, หน่วย 0.1)
   M8000
──┤├──────[DMUL D100 K1000 D300]  // D300:D301 = Raw × 1000
   M8000
──┤├──────[DDIV D300 K27648 D200] // D200 = Raw × 1000 / 27648
                                   //      = 0-1000 (0.0-100.0%)
```

---

## Step 341: Structured Ladder Programming ใน GX Works3

### การใช้ Function Block (FB)

```
GX Works3 รองรับ Function Block (FB) แบบ IEC 61131-3

การสร้าง FB:
1. Navigation → Program → FB/FUN
2. New FB: "Motor_Ctrl_FB"
3. กำหนด Variable:
   VAR_INPUT:
     Start_Cmd  : BOOL;
     Stop_Cmd   : BOOL;
     E_Stop     : BOOL;
   VAR_OUTPUT:
     Motor_Out  : BOOL;
     Fault_Out  : BOOL;
   VAR:
     Run_Timer  : TON;
     Fault_Latch: BOOL;
   END_VAR

LAD ใน FB Motor_Ctrl_FB:
Network 1: Start/Stop
   Start_Cmd   E_Stop   Stop_Cmd       Motor_Out
──┤├──────┬───┤/├───────┤/├────────────( )──
          │
   Motor_Out
──┤├──────┘

Network 2: Timer
   Motor_Out        [TON]
──┤├──────────── IN      Q ──── Timer_Done
            T#5S── PT    ET ──── Timer_Value

การใช้ FB ใน Main Program:
[Motor_Ctrl_FB Instance: Motor1]
X0──┤Start_Cmd  Motor_Out├──Y0
X1──┤Stop_Cmd   Fault_Out ├──Y1
X2──┤E_Stop              │
```

---

## Step 342: Q Series Overview

### MELSEC-Q Architecture

```
Q Series เป็น Modular PLC ขนาดใหญ่

CPU Module Variants:
Q00CPU  : Basic, 8KB Step, 8KB Device
Q01CPU  : 14KB Step, 8KB Device
Q02CPU  : 28KB Step, 28KB Device
Q06CPU  : 60KB Step, 98KB Device
Q13CPU  : 120KB Step, 98KB Device
Q26CPU  : 260KB Step, 226KB Device
Q50CPU  : 500KB Step, 512KB Device
Q100CPU : 1000KB Step, 512KB Device

High-performance CPU:
Q02HCPU, Q06HCPU: ประสิทธิภาพสูงกว่า 10 เท่า
Redundant CPU: Q12PRHCPU, Q25PRHCPU

Safety CPU:
Q12DSCPU, Q26DSCPU: SIL 3 Safety
```

### Q Series Chassis Layout

```
Q Series Chassis (5 Slot):
┌──────┬──────┬──────┬──────┬──────┬──────┐
│ SLOT │  0   │  1   │  2   │  3   │  4   │
│  -1  │      │      │      │      │      │
├──────┼──────┼──────┼──────┼──────┼──────┤
│Power │ CPU  │ Base │  I/O │  AI  │  Net  │
│Q62P  │Q06H  │ Unit │ QX40 │QY40P │QJ71LP│
│      │      │      │      │      │      │
└──────┴──────┴──────┴──────┴──────┴──────┘

Multi-CPU System:
Slot -1: Power Supply
Slot 0: CPU No.1 (Master)
Slot 1: CPU No.2 (Motion)
Slot 2: CPU No.3 (Safety)
Slot 3: I/O Module
...

CPUs ใน Chassis เดียวกัน Share ข้อมูลผ่าน:
- Fixed Scan Communication
- Multi-CPU Refresh
```

---

## Step 343: GX Works3 Simulation

### PLCSIM (Simulator) ใน GX Works3

```
GX Works3 มี Simulator ในตัว (Simulation Function):
ไม่ต้องมี Hardware จริงเพื่อทดสอบ!

วิธีใช้:
1. Compile Program (F4)
2. Debug → Simulation Start/Stop
3. Simulator Window เปิดขึ้น

Simulator Window:
┌──────────────────────────────────────────────────┐
│ GX Works3 Simulator                              │
├──────────────────────────────────────────────────┤
│ Status: ● RUNNING                                │
│ Scan Time: 12.5ms                                │
├──────────────────────────────────────────────────┤
│ I/O Simulation:                                  │
│ X0: [ON] [OFF]    Y0: [ON]                       │
│ X1: [ON] [OFF]    Y1: [OFF]                      │
│ X2: [ON] [OFF]    Y2: [OFF]                      │
├──────────────────────────────────────────────────┤
│ Device Monitor:                                  │
│ M8000: TRUE   M8002: FALSE                       │
│ T0: 0.0s     C0: 0                               │
│ D0: 1500      D1: 0                              │
└──────────────────────────────────────────────────┘

Simulation Features:
- Force Input (จำลอง Input ON/OFF)
- Monitor All Devices
- Timeline Tracking
- ทดสอบ LAD/ST/FBD ได้เหมือน Hardware จริง
```

### Device Monitor ใน Online Mode

```
GX Works3 → Online → Device Monitor (หรือ Watch Window)

Device Monitor Window:
┌──────────────────────────────────────────────────────┐
│ Device Monitor                                       │
├──────────┬──────────┬──────────┬────────────────────┤
│ Device   │ Current  │ Change   │ Comments           │
├──────────┼──────────┼──────────┼────────────────────┤
│ X0       │ OFF      │ [Force]  │ Start Button       │
│ X1       │ OFF      │ [Force]  │ Stop Button        │
│ Y0       │ ON       │          │ Motor Contactor    │
│ T0       │ K0/K50   │          │ Start Delay 5s     │
│ C0       │ 45/100   │          │ Production Count   │
│ D100     │ 1500     │ [Modify] │ Motor Speed SP     │
│ D101     │ 72.5     │          │ Temperature        │
└──────────┴──────────┴──────────┴────────────────────┘
```

---

## Step 344: การ Download Program

### Uploading/Downloading ใน GX Works3

```
Connection Methods:
1. USB: ใช้ USB-B Cable ต่อกับ FX5U โดยตรง
2. Ethernet: ต่อผ่าน Network (ต้องตั้ง IP ก่อน)
3. SD Card: เขียนลง SD Card แล้วใส่ FX5U

การ Download ผ่าน USB:
1. Online → Transfer Setup
2. เลือก "USB" → Connection Test
3. ถ้าพบ PLC: "Connected to FX5U-64MT"
4. Online → Write to PLC

Write to PLC Dialog:
┌────────────────────────────────────────────────────┐
│ Write to PLC                                       │
├────────────────────────────────────────────────────┤
│ ✓ Intelligent Function Module Parameter            │
│ ✓ PLC Parameter                                   │
│ ✓ Program (Main, Sub1, Sub2)                      │
│ ✓ Device Comment                                  │
│ ✓ Device Initial Value                            │
├────────────────────────────────────────────────────┤
│ [Execute]   [Cancel]                              │
└────────────────────────────────────────────────────┘
5. Click Execute
6. Set PLC to RUN
```

### Password Protection

```
GX Works3 สามารถตั้ง Password ป้องกัน Program:

Online → Password → Set Password
- Read Protection: ป้องกันการ Upload Program
- Write Protection: ป้องกันการ Download

ระดับ Protection:
Level 1: Password อย่างเดียว
Level 2: Password + ปิด USB (Force Ethernet เท่านั้น)
Level 3: Full Lock (ต้องใช้ Keyword ลบ Password)

หมายเหตุ:
ถ้าลืม Password ต้องส่ง CPU กลับ Mitsubishi เท่านั้น
→ บันทึก Password เก็บไว้เสมอ!
```

---

## Step 345: Analog I/O Module

### FX5-4AD Analog Input Module

```
FX5-4AD: 4 ช่อง Analog Input
- Voltage: 0-10V, -10V to +10V
- Current: 4-20mA
- Resolution: 16-bit (0-27648)

การตั้งค่าใน GX Works3:
1. Navigation → Parameter → Module Configuration
2. Drag FX5-4AD ไปวางที่ Slot ที่ต้องการ
3. Double-click → Module Parameter:
   Channel 1: 4-20mA (Flow Sensor)
   Channel 2: 0-10V (Pressure Sensor)
   Channel 3: 4-20mA (Level Sensor)
   Channel 4: 4-20mA (Temperature TC)

การอ่านค่า (Auto Refresh):
1. Parameter → Module 1 → Refresh Setting
2. กำหนด AI1 → D200
3. กำหนด AI2 → D201
4. ใน Program อ่านจาก D200 โดยตรง

หรือใช้ FROM/TO Instruction:
// FROM: อ่านจาก Intelligent Module
LD   M8000
FROM K1   K0   D200  K4   // อ่าน Buffer 0-3 จาก Module 1
                          // K1 = Module Position 1
                          // K0 = Starting Buffer Number
                          // D200 = Destination
                          // K4 = จำนวน Word ที่อ่าน
```

### FX5-4DA Analog Output Module

```
FX5-4DA: 4 ช่อง Analog Output
- Voltage: 0-10V, -10V to +10V
- Current: 4-20mA
- Resolution: 16-bit (0-27648)

การเขียนค่า (ผ่าน Auto Refresh):
// D300 = AO1 Command Value (0-27648 = 0-10V)
// กำหนดใน Refresh Setting: D300 → Output CH1

// ตั้ง Output = 50% (5V หรือ 12mA)
   M8000
──┤├──────[MOV K13824  D300]  // 27648/2 = 13824

// ตั้ง Output ตาม Speed Command
   M8000
──┤├──────[MOV Speed_Command  D300]

// แปลงจาก RPM (0-3000) เป็น AO (0-27648):
// AO = RPM × 27648 / 3000
   M8000
──┤├──────[MUL Speed_RPM  K27648  D400]
   M8000
──┤├──────[DDIV D400  K3000  D300]
```

---

## Step 346: MELSEC Communication Protocol

### CC-Link (Control Communication Link)

```
CC-Link เป็น Field Network ของ Mitsubishi:
- Speed: 156kbps ถึง 10Mbps
- Distance: สูงสุด 1200m
- Devices: สูงสุด 64 stations per network

CC-Link IE: Gigabit Industrial Ethernet
- Speed: 1Gbps
- ใช้ Standard Ethernet Cable/Switch

Q Series CC-Link Master Module:
QJ61BT11: CC-Link Master/Slave

FX5U CC-Link IE Field Basic:
Built-in ใน FX5U (ไม่ต้องซื้อ Module!)
ต่อกับ Remote I/O Module (FX5-CCL-MS)

ตัวอย่าง Architecture:
[FX5U Master]──CC-Link IE──[FX5-CCL-MS]──[Remote I/O]
                        └──[FX5-CCL-MS]──[Remote I/O]
                        └──[Servo Drive AJ65]
```

### SLMP Communication (FX5U)

```
FX5U รองรับ SLMP Protocol ผ่าน Ethernet:
ทำให้ PC, HMI, SCADA อ่าน/เขียน PLC ได้

SLMP Frame Format:
[Header][Serial No.][Reserved][Data Length][Command][Data]

ตัวอย่าง Read D100 ด้วย Python:
```python
import socket

def read_d_register(ip, port, start_reg, count):
    # SLMP 3E Frame: Read Word
    msg = bytearray([
        0x50, 0x00,         # Subheader
        0x00,               # Network No.
        0xFF,               # PC No.
        0xFF, 0x03,         # I/O No.
        0x00,               # Station No.
        0x0C, 0x00,         # Data Length
        0x10, 0x00,         # CPU timer
        0x01, 0x04,         # Command: Read
        0x00, 0x00,         # Subcommand: Word
        start_reg & 0xFF, (start_reg >> 8) & 0xFF, 0x00, 0x00,  # Start Address
        count & 0xFF, (count >> 8) & 0xFF  # Count
    ])
    # Device Type 'D' = 0xA8
    msg[14] = 0xA8
    
    with socket.socket() as s:
        s.connect((ip, port))
        s.send(msg)
        response = s.recv(1024)
    return response
```
```

---

## Step 347: Structured Text ใน GX Works3

### ST (Structured Text) ใน FX5U

```
GX Works3 รองรับ ST ใน FBD/LD routine และ FB

ตัวอย่าง ST สำหรับ PID Control (ใช้ FB):
// FB: PID_Control_FB
// VAR_INPUT: SP, PV, Kp, Ti, Td, Auto
// VAR_OUTPUT: Output
// VAR: Error, Prev_Error, Integral, Derivative

IF Auto THEN
    (* คำนวณ Error *)
    Error := SP - PV;
    
    (* Integral (ป้องกัน Windup) *)
    Integral := Integral + Error * 0.1;  (* Sample time = 0.1s *)
    IF Integral > 100.0 THEN Integral := 100.0; END_IF;
    IF Integral < 0.0   THEN Integral := 0.0;   END_IF;
    
    (* Derivative *)
    Derivative := (Error - Prev_Error) / 0.1;
    Prev_Error := Error;
    
    (* PID Output *)
    Output := Kp * (Error + (1.0/Ti) * Integral + Td * Derivative);
    
    (* Clamp Output *)
    IF Output > 100.0 THEN Output := 100.0; END_IF;
    IF Output < 0.0   THEN Output := 0.0;   END_IF;
ELSE
    (* Manual Mode *)
    Integral := Output / Kp;  (* ป้องกัน Bump เมื่อ Auto กลับ *)
END_IF;
```

---

## Step 348: Safety Precautions และ Special Functions

### Watchdog Timer

```
FX5U มี Watchdog Timer ป้องกัน CPU Hang:

ค่า Default: 200ms (หาก Scan > 200ms → CPU Error)
ปรับได้ใน Parameter: 10-65000ms

GX Works3 → Parameter → FX5UCPU → PLC System:
  Watchdog Timer: 500ms  // เพิ่มสำหรับ Program ที่ซับซ้อน

Special M8028: Disable Watchdog (ห้ามใช้ในการผลิต!)
Special M8049: Enable Serial Communication Error Halt
```

### Battery Backup

```
FX5U มี ithium Battery สำรอง Memory:
- D ที่กำหนดเป็น Latch (R จะไม่สูญหาย)
- C ที่กำหนดเป็น Latch Counter
- Running Hour, Counter ยังคงอยู่หลัง Power Off

Battery Life: 5 ปี (Standard)

การตั้งค่า Latch Range:
GX Works3 → Parameter → FX5UCPU → Device:
  Latched M: M500-M1999
  Latched D: D100-D999
  Latched C: C100-C199

M8005: Battery Voltage Low Warning
M8006: Battery Low (Latched) → ต้องเปลี่ยน Battery!
```

---

## Step 349: โปรแกรมตัวอย่าง - Injection Molding Machine

### Project Overview

```
เครื่อง Injection Molding:
- Clamping Unit: เปิด/ปิด Mold
- Injection Unit: ฉีด Plastic
- Cooling: ระบายความร้อน
- Ejection: ดีด Product ออก

Sequence:
1. Clamp Close → Verify Closed
2. Inject → Pressure Hold
3. Cooling Timer
4. Mold Open
5. Eject → Verify Clear
6. Eject Back → Ready

I/O:
X0: Cycle_Start     X10: Clamp_Closed_LS
X1: Cycle_Stop      X11: Clamp_Open_LS
X2: E_Stop          X12: Inject_Forward_LS
X3: Safety_Gate_OK  X13: Inject_Back_LS
X4: Mold_Present    X14: Eject_Forward_LS
                    X15: Eject_Back_LS

Y0: Clamp_Close_Sol  Y10: Inject_Forward_Sol
Y1: Clamp_Open_Sol   Y11: Inject_Back_Sol
Y2: Cooling_Valve    Y12: Eject_Forward_Sol
Y3: Cycle_Light      Y13: Eject_Back_Sol
Y4: Alarm_Horn
```

### Sequence Program ด้วย Step Ladder (STL)

```
// ใช้ State Relay (S) สำหรับ Step Sequence

// Initial State
   M8002            S0
──┤├────────────────(SET S0)──   // เริ่มที่ S0

// S0: Initial / Ready
STL S0
   X0(Cycle Start)  X3(Safety OK)    S10
──┤├────────────────────┤├───────────(SET S10)──
   Y3
──────────────────────────( )──      // Ready Light ON
RET

// S10: Clamp Close
STL S10
   Y0
──────────────────────────( )──      // Clamp Close Solenoid ON
   X10(Clamp Closed LS)              S20
──┤├─────────────────────────────────(SET S20)──
   T0
──────────────────────(T0 K20)──     // Timeout 2s
   T0                                M200(Fault)
──┤├─────────────────────────────────(SET M200)──
RET

// S20: Injection
STL S20
   Y10
──────────────────────────( )──      // Inject Forward Sol
   X12(Inject Forward LS)            S30
──┤├─────────────────────────────────(SET S30)──
RET

// S30: Pressure Hold & Cooling
STL S30
   Y2
──────────────────────────( )──      // Cooling Valve
   T1
──────────────────────(T1 K50)──     // 5 วินาที Cooling
   T1                                S40
──┤├─────────────────────────────────(SET S40)──
RET

// S40: Mold Open
STL S40
   Y1
──────────────────────────( )──      // Clamp Open Sol
   X11(Clamp Open LS)                S50
──┤├─────────────────────────────────(SET S50)──
RET

// S50: Eject
STL S50
   Y12
──────────────────────────( )──      // Eject Forward Sol
   X14(Eject Forward LS)             S60
──┤├─────────────────────────────────(SET S60)──
RET

// S60: Eject Return
STL S60
   Y13
──────────────────────────( )──      // Eject Back Sol
   X15(Eject Back LS)                S0
──┤├─────────────────────────────────(TRAN S0)──
RET
```

---

## Step 350: Lab Projects

### Lab 18-1: FX5U Temperature Controller

```
โจทย์: ควบคุมอุณหภูมิ Oven ด้วย FX5U

Hardware จำลอง:
- AI CH1 (Built-in): Thermocouple (0-10V = 0-400°C)
- AO CH1 (FX5-4DA): Heater PWM (0-10V = 0-100%)
- X0: Start, X1: Stop, X2: Alarm Reset
- Y0: Heater ON Indicator, Y1: Alarm Horn

Requirements:
1. Manual Mode: กำหนด Output โดยตรง (D200 = 0-100%)
2. Auto Mode: PID Control
   - SP อ่านจาก D100 (ตั้งผ่าน HMI)
   - PV = Thermocouple Reading
   - Kp=2.5, Ti=120s, Td=30s
3. Alarm: Temperature > SP + 10°C
4. Safety: ถ้า Temperature > 380°C → Heater OFF + Alarm
5. Data Logging: บันทึก Temperature ทุก 1 นาทีลง D1000-D1059 (60 นาที)

เป้าหมาย:
- ใช้ FB สำหรับ PID
- ใช้ Special M สำหรับ Initialization
- Data Logging ด้วย Circular Buffer
```

### Lab 18-2: High-speed Counter Application

```
โจทย์: วัดความเร็ว Motor ด้วย Encoder

Hardware:
- Encoder: 1000 Pulse/Rev (X0 = Phase A)
- FX5U High-speed Counter C235

Requirements:
1. นับ Pulse ทุก 100ms (ใช้ Interrupt OB)
2. คำนวณ RPM = (Pulse × 60) / (1000 × 0.1)
3. Display Speed บน Watch Window
4. Alert ถ้า Speed > 3000 RPM
5. Measure Acceleration (ΔSpeed / ΔTime)

Steps:
- ตั้งค่า C235 ใน Parameter
- ใช้ M8029 สำหรับ Interrupt (ทุก 100ms)
- ใช้ DMOV สำหรับอ่าน 32-bit Counter
- คำนวณด้วย DMUL/DDIV
```

---

## ข้อผิดพลาดที่พบบ่อยใน Mitsubishi

```
1. "Syntax Error: X8 or X9 ไม่มีในระบบ Octal"
   สาเหตุ: ใช้ X8, X9, Y8, Y9 (ไม่มีในระบบ Octal)
   แก้ไข: ใช้ X10 แทน X8 (X8 Decimal = X10 Octal)

2. Timer ทำงานผิด
   สาเหตุ: ใช้ Timer ผิดรุ่น (100ms vs 10ms)
   แก้ไข: ตรวจสอบว่าใช้ T□ กลุ่มไหน

3. Counter ไม่นับ
   สาเหตุ: ต้อง OUT C0 ก่อน ถึงจะนับ
   แก้ไข: LD X0 → OUT C0 K100 (ไม่ใช่แค่ Contact C0)

4. DMOV/DMUL ผลลัพธ์ผิด
   สาเหตุ: ลืมใช้ D prefix (ใช้ MOV แทน DMOV)
   แก้ไข: Data 32-bit ต้องใช้ DMOV, DMUL ฯลฯ

5. Special M8000 ต้องเปิดตลอดเพื่อทำ MOV
   สาเหตุ: ลืมใส่ M8000 เป็น Contact นำ
   แก้ไข: LD M8000 → MOVE D0 D1

6. เขียนทับ Special Register D8000+
   สาเหตุ: ใช้ D8000 เป็น User Register
   แก้ไข: User Register ควรเริ่มจาก D100 ขึ้นไป
```

---

## สรุปและทบทวน

**ความรู้ที่ได้รับจาก Part 18:**

1. **Octal Address** ของ X/Y เป็นเอกลักษณ์ของ Mitsubishi (ไม่มี X8, X9)
2. **Special M8000-M8511** เป็น Built-in flags ที่มีประโยชน์มาก
3. **FX5U** มี Built-in Ethernet, Analog, SD Card
4. **Step Ladder (STL/RET)** เหมาะสำหรับ Sequence Programming
5. **High-speed Counter (HSC)** สำหรับ Encoder และ High-frequency Input
6. **DMOV/DMUL** ต้องใช้เมื่อ Data เป็น 32-bit
7. **FB ใน GX Works3** รองรับ IEC 61131-3 Standard
8. **Simulation** ช่วยทดสอบโดยไม่ต้องใช้ Hardware

---

**[← Part 17: Allen-Bradley](part17_allen_bradley.md) | [Part 19: OMRON →](part19_omron.md)**

*เนื้อหา Part 18 ครอบคลุม Steps 331-350*
*ลิขสิทธิ์ © 2024 Ladder Diagram PLC Course*
