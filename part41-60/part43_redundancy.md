# Part 43: Redundancy Systems (Steps 841-860)

## ภาพรวม
Redundancy Systems คือการออกแบบระบบที่มีส่วนประกอบสำรองเพื่อให้ระบบยังคงทำงานได้แม้มีความล้มเหลวเกิดขึ้น เป็นพื้นฐานสำคัญของระบบที่ต้องการ High Availability (HA)

---

## Step 841: Types of Redundancy

### ประเภทของ Redundancy

```
Redundancy Types Overview:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. CPU Redundancy                                          │
│     Primary CPU ◄──sync──► Secondary CPU                   │
│     (Active)               (Hot Standby)                   │
│                                                             │
│  2. Power Supply Redundancy                                 │
│     PS1 ──┬──► Load                                        │
│     PS2 ──┘  (ORing diodes)                               │
│                                                             │
│  3. Network Redundancy (Ring/MRP)                          │
│     SW1 ─── SW2 ─── SW3 ─── SW4                           │
│      │                         │                           │
│      └─────────────────────────┘  (Ring closed)            │
│                                                             │
│  4. I/O Redundancy                                         │
│     Field Device ──┬──► Primary I/O                        │
│                    └──► Secondary I/O                      │
│                                                             │
│  5. Communication Redundancy                               │
│     PLC ──Channel 1──► Remote I/O                         │
│         └─Channel 2──►  (same rack)                       │
│                                                             │
│  6. Full System Redundancy (All of above)                  │
│     Used in: Chemical plants, Power generation,            │
│              Water treatment, Oil & Gas                    │
└─────────────────────────────────────────────────────────────┘
```

### Availability Comparison

```
System Availability Formula:
A = MTBF / (MTBF + MTTR)

Where:
MTBF = Mean Time Between Failures
MTTR = Mean Time To Repair

Examples:
┌─────────────────────────────────────────────────────────────┐
│  Single CPU:                                                │
│  MTBF = 50,000h, MTTR = 4h                                 │
│  A = 50,000 / (50,000 + 4) = 99.992%                      │
│  Downtime/year = 0.008% × 8760 = 0.7 hours/year           │
│                                                             │
│  Redundant CPUs (1oo2):                                    │
│  System MTBF ≈ MTBF²/(2×MTTR) = 312,500,000h              │
│  A ≈ 99.99997%                                             │
│  Downtime/year < 1 minute/year                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 842: Siemens H-System (S7-400H/S7-1500H)

### S7-400H Architecture

```
S7-400H Redundant System:
┌─────────────────────────────────────────────────────────────┐
│  CPU Rack 1                    CPU Rack 2                   │
│  ┌───────────────────┐        ┌───────────────────┐        │
│  │ PS  CPU412-5H  IM │        │ PS  CPU412-5H  IM │        │
│  │     (Primary)     │◄──────►│    (Secondary)    │        │
│  └──────────┬────────┘  Sync  └────────┬──────────┘        │
│             │            Bus           │                   │
│             │  ┌──────────────────────┐│                   │
│             └──► PROFIBUS Redundant   ◄┘                   │
│                └──────────┬───────────┘                    │
│                           │                                │
│                    ┌──────▼──────┐                         │
│                    │  ET200M     │                          │
│                    │  Remote I/O │                          │
│                    └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

### S7-1500H Architecture

```
S7-1500H System (ใหม่กว่า S7-400H):
┌─────────────────────────────────────────────────────────────┐
│         CPU 1515H-2 PN (Primary)                           │
│  ┌─────────────────────────────────────────┐               │
│  │  CPU + Interface Module                 │               │
│  │  IM 155-6 PN HA (Rack HA interface)    │               │
│  └────────────────┬────────────────────── ┘               │
│                   │   PROFINET MRP Ring                    │
│  ┌─────────────── ▼ ──────────────────────┐               │
│  │  ET200SP HA Modules (Remote I/O)        │               │
│  │  AI/DI/DO/AO modules with redundancy   │               │
│  └────────────────────────────────────────┘               │
│                   │   Sync via Fiber                       │
│         CPU 1515H-2 PN (Secondary)                         │
│  ┌─────────────────────────────────────────┐               │
│  │  (Hot Standby - synchronized with       │               │
│  │   Primary, takes over in < 100ms)       │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### S7-1500H Specifications

| พารามิเตอร์ | S7-1515H-2 PN | S7-1517H-3 PN |
|------------|---------------|---------------|
| Work Memory | 1MB (code) + 5MB (data) | 2MB + 8MB |
| Switchover Time | < 100ms | < 100ms |
| Sync Interface | 2x Fiber (1Gbps) | 2x Fiber (1Gbps) |
| PROFINET IRT | Yes | Yes |
| PROFIsafe | Yes | Yes |
| Min. Cycle | 1ms | 1ms |

---

## Step 843: Switchover Conditions

### เงื่อนไขการสลับระบบ (Switchover)

```
Switchover Triggers:
┌─────────────────────────────────────────────────────────────┐
│  Automatic Switchover (ไม่ต้องมีการแทรกแซงจากมนุษย์):       │
│                                                             │
│  1. CPU Hardware Failure                                    │
│     ├── CPU goes to STOP or DEFECT state                   │
│     └── Detected via sync link monitoring                   │
│                                                             │
│  2. Power Supply Failure                                    │
│     └── Primary PS voltage out of range                    │
│                                                             │
│  3. Communication Failure                                   │
│     ├── Both sync links fail simultaneously               │
│     └── → Primary takes over, secondary disconnects       │
│                                                             │
│  4. Watchdog Timeout                                        │
│     └── CPU cycle exceeds max cycle time                   │
│                                                             │
│  Manual Switchover:                                         │
│  1. Maintenance switchover                                  │
│     └── ใน TIA Portal: H-System diagnostics view           │
│                                                             │
│  2. STEP 7 command: H_CPU_BSEND / H_CPU_BRCV               │
│     └── สั่ง switchover จาก HMI/SCADA                     │
└─────────────────────────────────────────────────────────────┘
```

### Switchover States

```
H-System States:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  SOLO Mode:                                                 │
│  Primary only (Secondary disconnected)                     │
│  → ไม่มี redundancy protection                            │
│                                                             │
│  LEAD/STANDBY Mode:                                         │
│  Primary (LEAD) running, Secondary (STANDBY) synchronized  │
│  → Full redundancy, switchover ready                       │
│                                                             │
│  TRANSITION Mode:                                           │
│  During synchronization after a fault/repair              │
│  → Secondary catching up with Primary                     │
│                                                             │
│  State Diagram:                                             │
│  SOLO ──synchronize──► TRANSITION ──sync complete──► LEAD/STANDBY
│     ◄───────────────────────────────────────────────────────
│  (fault on secondary removes it to SOLO again)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 844: Bumpless Transfer

### การส่งมอบควบคุมแบบ Bumpless

```
Bumpless Transfer Requirements:
┌─────────────────────────────────────────────────────────────┐
│  Bumpless = "ไม่มีการกระตุก" ในสัญญาณ output               │
│                                                             │
│  ก่อน Switchover (Primary กำลังทำงาน):                    │
│  Primary DB.Analog_Out = 50.0%  (ค่า PV จาก Primary)      │
│  Secondary DB.Analog_Out = 50.0% (sync ไว้แล้ว)           │
│                                                             │
│  ขณะ Switchover (< 100ms):                                 │
│  - Primary หยุดทำงาน                                       │
│  - Secondary รับหน้าที่ต่อ                                 │
│  - DB ถูก sync ไว้แล้ว → output = 50.0% (ไม่เปลี่ยน!)    │
│                                                             │
│  หลัง Switchover:                                          │
│  Secondary DB.Analog_Out = 50.0% (ต่อเนื่องจากเดิม)       │
│  → ไม่มี bump!                                             │
│                                                             │
│  Key: ต้อง sync ทั้ง Program Data และ Output Values        │
└─────────────────────────────────────────────────────────────┘
```

### การโปรแกรมสำหรับ Bumpless Transfer

```pascal
// S7-1500H: การ sync ข้อมูล PID สำหรับ bumpless transfer
// ใน H-System, DB ถูก sync อัตโนมัติ
// แต่ต้องระวัง:

// 1. ใช้ Standard DB (non-optimized) เพื่อ sync
// 2. Timestamp ของ output values ต้อง update ทุก cycle

// PID Control Block ใน Primary CPU
"PID_DB".Set_Point      := SP_Value;
"PID_DB".Process_Value  := PV_Input;
"PID_DB".Output         => AO_Value;  // Output ถูก sync ไป Secondary

// ใน Secondary CPU (same DB):
// "PID_DB".Output จะมีค่าเดียวกันเสมอ
// เมื่อ Switchover: Secondary ใช้ค่านี้ทันที → Bumpless!
```

---

## Step 845: Synchronization

### กระบวนการ Synchronization

```
Synchronization Process:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Phase 1: Initial Synchronization (เมื่อ Secondary boot up) │
│  ─────────────────────────────────────────────────────────  │
│  Primary ──── Program (OB/FB/FC/DB) ────► Secondary        │
│         ──── Hardware Config ────────────►                 │
│         ──── I/O Initial Values ─────────►                 │
│  Duration: 10-30 seconds (ขึ้นกับขนาด memory)            │
│                                                             │
│  Phase 2: Runtime Synchronization (ทุก cycle)              │
│  ─────────────────────────────────────────────────────────  │
│  Primary ──── I/O Image ────────────────► Secondary        │
│         ──── DB Modified Values ─────────►                 │
│         ──── PID Output Values ──────────►                 │
│  Duration: < 1ms per cycle                                 │
│                                                             │
│  Sync Physical Link:                                        │
│  - S7-400H: Fiber optic (2 redundant links)               │
│  - S7-1500H: Fiber optic 1Gbps (2 redundant links)        │
│  - Distance: Up to 10km (single mode fiber)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 846: Redundant PROFINET (MRP)

### MRP (Media Redundancy Protocol)

```
MRP Ring Topology:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Switch 1 ──── Switch 2 ──── Switch 3 ──── Switch 4      │
│    (MRC)         (MRM)          (MRM)         (MRM)        │
│      │                                           │         │
│      └─────────────────────────────────────────┘         │
│                   (Ring closed)                            │
│                                                             │
│  MRC = Media Redundancy Client (active path)              │
│  MRM = Media Redundancy Manager (monitors ring)           │
│                                                             │
│  Normal Operation: Ring blocked at one point              │
│  ─────────────────────────────────────────────────────    │
│  Switch 1 → Switch 2 → Switch 3 → Switch 4                │
│  (Link to SW1 blocked at SW4 to prevent loops)            │
│                                                             │
│  Failure Operation: Ring healed automatically             │
│  ─────────────────────────────────────────────────────    │
│  If SW2-SW3 link fails:                                   │
│  ├── MRM detects failure (via ring frames)                │
│  ├── Opens previously blocked link at SW4                 │
│  └── New path: SW1 → SW4 → SW3 (backward) + SW1 → SW2    │
│  Recovery time: < 200ms (PROFINET specification)          │
└─────────────────────────────────────────────────────────────┘
```

### MRP Configuration ใน TIA Portal

```
// TIA Portal: Network View > Switch > MRP Settings

// สำหรับ MRM (Media Redundancy Manager):
Media_Redundancy_Role    := MRM;
MRP_Domain_Name          := "MRP_Domain_1";
Recovery_Delay           := 200ms;

// สำหรับ MRC (Media Redundancy Client):
Media_Redundancy_Role    := MRC;
MRP_Domain_Name          := "MRP_Domain_1";

// การตรวจสอบสถานะ MRP ใน PLC Program:
"LDevDiag_DB"(
    LADDR  := W#16#3F4,  // HW ID ของ PROFINET interface
    DONE   => Diag_Done,
    ERROR  => Diag_Error,
    WBUF   => Diag_Buffer
);

// อ่าน MRP status จาก buffer
IF "LDevDiag_DB".DONE THEN
    MRP_Status := Diag_Buffer.mrp_state;
    // 0 = Disabled, 1 = Active (normal), 2 = Open (degraded)
END_IF;
```

---

## Step 847: Redundant Power Supply

### การออกแบบ Redundant Power Supply

```
Redundant Power Supply Design:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  AC Input A ──► PS1 (24VDC) ──┬──► Load Bus               │
│                                │                           │
│  AC Input B ──► PS2 (24VDC) ──┘    (ORing diodes)         │
│                                                             │
│  Monitoring:                                               │
│  PS1 OK ──► DI 0.0 (Digital input to PLC)                 │
│  PS2 OK ──► DI 0.1                                         │
│                                                             │
│  Alert Logic:                                              │
│                                                             │
│  Alarm when only 1 PS remaining:                           │
│  NOT(PS1_OK) OR NOT(PS2_OK) → Alarm "PS Redundancy Lost"  │
│                                                             │
│  Shutdown when both fail:                                  │
│  NOT(PS1_OK) AND NOT(PS2_OK) → Emergency Stop             │
└─────────────────────────────────────────────────────────────┘
```

### Ladder Diagram สำหรับ PS Monitoring

```ladder
// Network: Power Supply Monitoring and Alarm
//
// DI_PS1_OK    DI_PS2_OK    PS_Alarm_1min    PS_Alarm_Output
// ---[/]---+---[/]---+----+--[ TON_1min ]--+-( )------------
//          |         |    |                 |
//          |         |    | (หน่วงเวลา 1 นาที เพื่อป้องกัน alarm กะพริบ)

+--[ NOT DI_PS1_OK ]--+--[ NOT DI_PS2_OK ]--+--[ TON T1 1min ]--+--( PS_Alarm )--+
|   (PS1 failed)       |   (PS2 failed)       |   (delay 1min)    |   (Alarm ON)   |
+---------------------+---------------------+-------------------+----------------+

// Network: Redundancy Status Display
//  PS1_OK   PS2_OK   Full_Redundancy
//  --[ ]+---[ ]------( )----------

+--[ DI_PS1_OK ]--+--[ DI_PS2_OK ]--+--( Full_Redundancy_OK )--+
|                 |                  |                           |
| (PS1 ปกติ)     | (PS2 ปกติ)       | (มี Redundancy สมบูรณ์)  |
+----------------+------------------+---------------------------+

// Network: Redundancy Alarm (ขาด PS ตัวใดตัวหนึ่ง)
//  NOT Full_Redundancy  → Redundancy_Warning alarm
+--[ NOT Full_Redundancy_OK ]--+--( Redundancy_Warning )--+
```

---

## Step 848: RAID for Historian

### RAID สำหรับ Historian Server

```
RAID Configuration for Process Historian:
┌─────────────────────────────────────────────────────────────┐
│  Recommended: RAID 10 (1+0)                                 │
│                                                             │
│       Data ──┬──► Disk 1 ──┬──  (Mirror Pair 1)           │
│              │   Disk 2 ──┘                               │
│              │                                             │
│              └──► Disk 3 ──┬──  (Mirror Pair 2)           │
│                  Disk 4 ──┘                               │
│                                                             │
│  RAID 10 ข้อดี:                                            │
│  ├── Read: Fast (parallel reading)                         │
│  ├── Write: Fast (no parity calculation)                   │
│  ├── Failure tolerance: 1 disk per mirror pair            │
│  └── Recovery: Fast (no rebuild calculation)              │
│                                                             │
│  Historian Database Requirement:                           │
│  ├── IOPS: >10,000 IOPS (High tag count)                  │
│  ├── Throughput: >500 MB/s                                 │
│  └── Availability: 99.99%+                                │
│                                                             │
│  Additional: Hot spare disk                                │
│  If disk fails → Hot spare auto-replace                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 849: Cold/Warm/Hot Standby

### ประเภทของ Standby

```
Standby Types Comparison:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  COLD STANDBY:                                              │
│  ├── Secondary ไม่ได้เปิดอยู่                              │
│  ├── Switchover time: นาที ถึง ชั่วโมง                     │
│  ├── ต้อง boot secondary, load config, sync data           │
│  └── ใช้สำหรับ: Non-critical, budget constraints           │
│                                                             │
│  WARM STANDBY:                                              │
│  ├── Secondary เปิดอยู่ แต่ข้อมูลไม่ sync real-time       │
│  ├── Switchover time: วินาที ถึง นาที                      │
│  ├── ต้อง sync data ก่อน takeover (เช่น ล่าสุด 5 นาที)    │
│  └── ใช้สำหรับ: Semi-critical systems                      │
│                                                             │
│  HOT STANDBY:                                               │
│  ├── Secondary sync real-time กับ Primary                  │
│  ├── Switchover time: < 100ms (Siemens H-System)          │
│  ├── ข้อมูลเป็นปัจจุบันทุก cycle                           │
│  └── ใช้สำหรับ: Critical systems, continuous process      │
│                                                             │
│  Siemens S7-1500H = Hot Standby                            │
│  Rockwell ControlLogix = Hot Standby available            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 850: Designing for High Availability

### การออกแบบระบบ High Availability

```
High Availability Design Checklist:
┌─────────────────────────────────────────────────────────────┐
│  Infrastructure Layer:                                      │
│  ✓ Redundant power (UPS + redundant PS modules)            │
│  ✓ Redundant network (MRP ring or dual networks)           │
│  ✓ Redundant CPU (H-System or GuardLogix)                  │
│  ✓ Redundant I/O (where critical)                          │
│                                                             │
│  Software Layer:                                            │
│  ✓ Online modification allowed (no stop needed)            │
│  ✓ Program backup and restore procedure                    │
│  ✓ Configuration management system                         │
│                                                             │
│  Maintenance Layer:                                         │
│  ✓ Hot-swap modules (replace without stopping)            │
│  ✓ Diagnostic tools (predictive maintenance)              │
│  ✓ Spare parts inventory                                   │
│                                                             │
│  Operational Layer:                                         │
│  ✓ Procedures for switchover                              │
│  ✓ Training for operators                                  │
│  ✓ Emergency procedures                                    │
└─────────────────────────────────────────────────────────────┘
```

### Availability Zones สำหรับ Large Plant

```
Plant Network Architecture (High Availability):

Zone 1: Control Network (PROFINET/EtherNet/IP)
┌───────────────────────────────────────────────┐
│  PLC A ────┬───────────────────── PLC B       │
│            │     MRP Ring         │           │
│           SW1 ── SW2 ── SW3 ── SW4            │
│            └──────────────────────┘           │
└───────────────────────────────────────────────┘

Zone 2: Supervisory Network (Ethernet)
┌───────────────────────────────────────────────┐
│  SCADA Server A ────── Switch A ──── SCADA B  │
│                             │                 │
│  Historian A ───────────────┘                 │
└───────────────────────────────────────────────┘

Zone 3: Business Network
┌───────────────────────────────────────────────┐
│  Enterprise systems (DMZ separation)          │
└───────────────────────────────────────────────┘

Firewall / DMZ between zones!
```

---

## Step 851-860: Calculating System Availability

### การคำนวณ System Availability

#### 1. Single Component

```
Single CPU:
MTBF = 100,000 hours
MTTR = 4 hours (includes detection + repair + restart)

A = MTBF / (MTBF + MTTR)
  = 100,000 / 100,004
  = 99.996%

Downtime/year = (1 - A) × 8760
             = 0.004% × 8760
             = 0.35 hours/year
             = 21 minutes/year
```

#### 2. Redundant (1oo2) System

```
Redundant CPU (Hot Standby):
MTBF_single = 100,000h
MTTR_detect = 0.1h (100ms auto-switchover)
MTTR_repair = 4h (after detection, repair secondary)

// System MTBF for 1oo2:
MTBF_system = MTBF_single² / (2 × MTTR_repair)
            = 100,000² / (2 × 4)
            = 10,000,000,000 / 8
            = 1,250,000,000 hours!

// System MTTR:
MTTR_system = MTTR_detect = 0.1h (switchover time)

A_system = MTBF_system / (MTBF_system + MTTR_system)
         = 1.25×10⁹ / (1.25×10⁹ + 0.1)
         ≈ 99.99999992%

Downtime/year ≈ 0.7 seconds/year (!)
```

#### 3. Complete System (Series + Parallel)

```
System with Multiple Components:

           Series Path 1 (Primary)
A ─────┬── CPU1 ── PS1 ── NET1 ──┬───── B
       │                          │
       └── CPU2 ── PS2 ── NET2 ──┘
           Parallel Path (Secondary)

Reliability Block Diagram (RBD):

// Calculate each path:
A_Path1 = A_CPU1 × A_PS1 × A_NET1
        = 0.99996 × 0.9999 × 0.9999
        = 0.99976

A_Path2 = same = 0.99976 (identical components)

// Parallel combination (at least 1 must work):
A_system = 1 - (1-A_Path1) × (1-A_Path2)
         = 1 - (1-0.99976)²
         = 1 - (0.00024)²
         = 1 - 5.76×10⁻⁸
         = 99.999994%

Downtime/year = 5.76×10⁻⁸ × 8760h = 0.00050h = 1.8 seconds/year!
```

#### 4. Complete Redundant System Design

```
High Availability Control System:

┌─────────────────────────────────────────────────────────────┐
│  System Design:                                             │
│                                                             │
│  Power: 2× UPS (A+B) + 2× PS per rack                     │
│  CPU: S7-1515H (Hot Standby pair)                          │
│  Network: MRP ring (8 switches, fiber)                     │
│  I/O: ET200SP HA modules (redundant terminals)             │
│  Historian: Redundant server (RAID 10)                     │
│                                                             │
│  Expected Availability: 99.999% (5 nines)                  │
│  = 5.26 minutes downtime per year                          │
│                                                             │
│  Investment vs. Single System:                              │
│  ├── Hardware cost: ~2.5× single system                    │
│  ├── Engineering cost: ~1.5× (more complex)               │
│  └── Savings: Avoided production loss                     │
│      Production value × 5 min / 8760 hr                   │
│      For ฿10M/hr plant: ฿10M × (5/60)/8760 ≈ ฿9,500/yr   │
└─────────────────────────────────────────────────────────────┘
```

#### 5. Ladder Program สำหรับ H-System

```ladder
// Siemens S7-1500H - Monitoring Redundancy Status
// OB: Main (OB1)

// Network 1: Check H-System Status
// H_GET_STATE: Get current H-System state
+--[H_GET_STATE]--+
|  MASTER  => H_Master_State   |  // 1=Primary, 0=Secondary
|  STDBY   => H_Standby_OK     |  // 1=Standby ready
|  SYNCOK  => H_Sync_OK        |  // 1=Sync link OK
+------------------------------|

// Network 2: H-System Status Display
//    H_Master     Disp_Primary
//    ---[ ]----------( )------
+--[ H_Master_State ]--+--( Display_As_Primary )--+

//    H_Standby    Disp_Standby_Ready
//    ---[ ]----------( )------
+--[ H_Standby_OK ]--+--( Display_Standby_Ready )--+

// Network 3: Alarm when redundancy lost
//    NOT H_Stdby_OK   Redund_Lost_Alarm
//    ---[/]---------------( )----------
+--[ NOT H_Standby_OK ]--+--( Redundancy_Lost_Alarm )--+

// Network 4: Manual Switchover (from HMI)
//    HMI_Switchover   Allow_Switch   H_SWITCH
//    ----[ ]----------[ ]-----------[ ACT ]---
+--[ HMI_Manual_Switch ]--+--[ Switch_Enable ]--+--[H_SWITCH ACT]--+
|   (ปุ่ม HMI)             |   (อนุญาต)         |   (สั่ง switch) |
+-------------------------+-------------------+--------------------|
```

---

## Lab Project 43: Redundant System for Chemical Reactor

### โจทย์
ออกแบบ Redundant Control System สำหรับ Chemical Reactor ที่ต้องการ 99.99% Availability

### ข้อกำหนด
- 100+ I/O points
- 4 PID loops (Temperature, Pressure, Flow, Level)
- Historian: 500 tags, 1 second resolution
- HMI: 2 workstations (redundant)
- SCADA interface ไปยัง MES

### งานที่ต้องทำ
1. คำนวณ MTBF/MTTR และ Availability ของระบบที่ออกแบบ
2. วาด System Architecture diagram
3. เลือก Hardware (S7-1500H หรือ GuardLogix)
4. ออกแบบ MRP Network
5. เขียน Redundancy Monitoring Program
6. จัดทำ Maintenance Procedure สำหรับ hot-swap

---

## สรุป Part 43

| ประเภท Redundancy | Switchover Time | Use Case |
|-----------------|----------------|---------|
| Cold Standby | นาที-ชั่วโมง | Non-critical |
| Warm Standby | วินาที-นาที | Semi-critical |
| Hot Standby (H-System) | < 100ms | Critical continuous |
| MRP Network | < 200ms | Network ring |
| Redundant PS | Instantaneous | Always recommended |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 42: Functional Safety](part42_functional_safety.md) |
| ► ถัดไป | [Part 44: Batch Control ISA-88](part44_batch_control.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
