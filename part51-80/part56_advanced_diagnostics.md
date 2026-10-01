# Part 56: Advanced Diagnostics และ PROFINET (Steps 1101-1120)

## บทนำ: PROFINET Diagnostics

PROFINET มี diagnostic capability ที่ครบวงจรกว่า fieldbus รุ่นเก่า เช่น PROFIBUS ทำให้วิเคราะห์ปัญหาได้รวดเร็วและแม่นยำ

---

## Step 1101: PROFINET Diagnostic Architecture

### ระดับของ Diagnostics ใน PROFINET

```
PROFINET DIAGNOSTIC LEVELS:
──────────────────────────────────────────────────────────────

LEVEL 4: Channel Diagnosis (ละเอียดที่สุด)
├── แต่ละ I/O channel มีสถานะ
├── Error type: Short circuit, Wire break, Overload
└── ตัวอย่าง: "DI Module, Slot 2, Channel 5: Wire break"

LEVEL 3: Module Diagnosis
├── แต่ละ Module (Slot) มีสถานะ
├── Error type: Module missing, Wrong module, Configuration error
└── ตัวอย่าง: "Slot 3: Module removed or not inserted"

LEVEL 2: Submodule Diagnosis
├── แต่ละ Submodule มีสถานะ
└── ตัวอย่าง: "Submodule configuration mismatch"

LEVEL 1: Device Diagnosis (ระดับ Device)
├── แต่ละ IO Device (e.g., ET200SP) มีสถานะ
├── Status: OK, Warning, Error, Not reachable
└── ตัวอย่าง: "Device 'Drive_Unit1': Alarm active"

LEVEL 0: Network Diagnosis (ระดับ Network)
├── Port status: Link Up/Down
├── Network topology
└── LLDP neighbor information
```

---

## Step 1102: TIA Portal Diagnostics

### Diagnostics ใน TIA Portal Online View

```
TIA Portal - Online Diagnostics Access:
──────────────────────────────────────────────────────────────

STEP 1: Go Online
Online → Connect to PLC → [PLC IP Address]

STEP 2: Device & Network View
Network View → Click on IO Device → Properties (Online)
├── Device status (green/yellow/red)
├── Module overview
└── Alarm list

STEP 3: PLC Diagnostics Buffer
PLC → Diagnostic buffer (Events log):
├── Show: แต่ละ event ที่เกิดขึ้น (timestamp, description)
├── Error codes: standard IEC 61158 error types
└── Filter by: time, error type, device

STEP 4: Forcing I/O (สำหรับ troubleshoot)
PLC → Watch and force tables
├── Watch: monitor values ของ variables
├── Force: บังคับค่า I/O สำหรับ test
└── ⚠️ WARNING: Force I/O ใช้ได้ใน maintenance mode เท่านั้น

STEP 5: Network Diagnostics
Network View → PROFINET → Topology View
├── แสดง physical connections ของ devices
├── Port status (green/red)
└── LLDP neighbor info
```

### Programming Diagnostics ใน Ladder

```
// Network 1: อ่าน Device Status ด้วย DeviceStates instruction
// ─────────────────────────────────────────────────────────────
//
//  Always ON                    DeviceStates
//  ──┤├──────────────────────┤ENO├──────────────────────────
//                                 LADDR: P#I 0.0        (IO controller address)
//                                 MODE: 1               (1=device status)
//                                 RET_VAL: wDiagResult   
//                                 VALID: xStatusValid
//                                 BUSY: xStatusBusy
//                                 DEVICE_STATUS: aDeviceStatus[1..64]
//
// aDeviceStatus array: bit 0 = device OK, bit 1 = device fault, etc.

// Network 2: Decode Device Status Bits
// ─────────────────────────────────────────────────────────────
// xDevice1_OK     := aDeviceStatus[1].0;   // Bit 0 = OK
// xDevice1_Fault  := aDeviceStatus[1].1;   // Bit 1 = Fault
// xDevice1_NoData := aDeviceStatus[1].2;   // Bit 2 = No data
// xDevice1_Diag   := aDeviceStatus[1].3;   // Bit 3 = Diagnostic active

// Network 3: PROFINET Diagnostic FB
// ─────────────────────────────────────────────────────────────
// ใช้ FB "PNIO_ALARM_REC" สำหรับอ่าน Alarm detail
//
//  Always ON                    PNIO_ALARM_REC
//  ──┤├──────────────────────┤ENO├──────────────────────────
//                                 LADDR: w#16#100      (Device hardware ID)
//                                 IOSYS: DB_AlarmData   
//                                 ATYPE: iAlarmType
//                                 AINFO: wAlarmInfo
//                                 DIAG: DB_DiagData
```

---

## Step 1103: PROFINET Channel Diagnostics

### Channel Diagnostic Programming

```
// Channel Diagnostics สำหรับ Analog Modules
// ─────────────────────────────────────────────────────────────

// Siemens AI Module (e.g., SM 531, ET 200SP AI module)
// มี diagnostic per channel:
// - Wire break (Open circuit)
// - Short circuit
// - Overflow (value > range)
// - Underflow (value < range)

// อ่าน Diagnostic ด้วย RD_SINFO instruction:

FUNCTION_BLOCK FB_AnalogDiagnostics
VAR_INPUT
    wModuleAddress: WORD;         // Hardware ID ของ module
END_VAR

VAR_OUTPUT
    xCH1_WireBreak: BOOL;
    xCH1_ShortCirc: BOOL;
    xCH2_WireBreak: BOOL;
    xCH2_ShortCirc: BOOL;
    xCH3_WireBreak: BOOL;
    xCH4_WireBreak: BOOL;
    ixDiagStatus:   INT;          // 0=OK, 1=Warning, 2=Error
END_VAR

VAR
    stModuleInfo:   STATIONINFO;
    xReady:         BOOL;
END_VAR

// Network 1: อ่าน Diagnostic Status
// รูปแบบ ไม่ใช่ Ladder แต่ใช้ SCL ใน OB:
// 
// stationinfo(LADDR := wModuleAddress, RET_VAL => iRetVal);
// 
// Channel 1 diagnostics:
// xCH1_WireBreak := stModuleInfo.CHANNELS[1].WIREBREAK;
// xCH1_ShortCirc := stModuleInfo.CHANNELS[1].SHORTCIRCUIT;
// 
// Overall status:
// IF stModuleInfo.ERROR THEN
//     ixDiagStatus := 2;  // Error
// ELSIF stModuleInfo.DIAG THEN
//     ixDiagStatus := 1;  // Warning/Diagnostic active
// ELSE
//     ixDiagStatus := 0;  // OK
// END_IF;
```

---

## Step 1104: ProfiTrace Tool

### PROFINET Network Analysis ด้วย ProfiTrace

```
ProfiTrace (Softing, LS Industrial, others):
──────────────────────────────────────────────────────────────

ฟังก์ชันหลัก:
├── PROFINET packet capture (passive - non-intrusive)
├── Real-time topology view
├── Device discovery และ identification
├── Jitter analysis (cycle time measurement)
├── Error analysis (protocol errors, retries)
└── Bandwidth utilization

Hardware Options:
├── ProfiTrace USB Tap: ต่อ USB กับ laptop, tap บน PROFINET cable
├── Industrial PC + NIC: dedicated PC สำหรับ capture
└── Switch Port Mirroring + laptop: ไม่ต้องซื้อ hardware เพิ่ม

TYPICAL ANALYSIS:
──────────────────────────────────────────────────────────────

1. Topology View:
   └── แสดง tree view ของ devices ที่เชื่อมต่อ
       ระบุ: device name, IP, MAC, vendor, device type

2. Cycle Time Analysis:
   └── วัด actual cycle time ของแต่ละ IO Device
       Expected: 4ms, Actual: 4.2ms → 5% over
       → หาสาเหตุ: overloaded switch, cable issue

3. Error Analysis:
   └── Count: retries, alarms, errors per device
       Device ที่มี error มากผิดปกติ → investigate

4. Bandwidth Utilization:
   └── ดู % bandwidth ที่ใช้
       If > 70%: risk of performance issues

5. Cable Quality Test:
   └── เช็ค signal quality, noise level
       ค่าที่ดี: Signal margin > 10dB
```

---

## Step 1105: LLDP Topology

### LLDP (Link Layer Discovery Protocol)

```
LLDP ใน PROFINET:
──────────────────────────────────────────────────────────────

LLDP ทำอะไร?
├── Device แจ้ง neighbor ว่าตัวเองคือใคร, ต่ออยู่ port ไหน
├── TIA Portal ใช้ LLDP สร้าง topology view
└── Network management tools ใช้ LLDP สร้าง network map

LLDP Data Elements:
├── Chassis ID: MAC address ของ device
├── Port ID: Port number/name
├── TTL (Time to Live): 120 seconds typically
├── Port Description: Friendly name
├── System Name: Device name (PROFINET station name)
├── System Description: Vendor, model
└── Management Address: IP address

Reading LLDP Data ใน TIA Portal:
Online → Network View → PROFINET → Topology View
├── ลาก-วาง: ปรับ topology ให้ตรงกับ real layout
└── Verify: ตรวจสอบว่าตรงกับ physical cabling

Reading LLDP ด้วย SNMP:
OID: 1.0.8802.1.1.2 (LLDP-MIB)
// ใช้ network management software อ่าน LLDP table
// Tool: SINEC NMS, PRTG, SolarWinds NMS
```

---

## Step 1106: Network Analysis with Wireshark

### Wireshark สำหรับ PROFINET Analysis

```
Wireshark PROFINET Filters:
──────────────────────────────────────────────────────────────

# ดู PROFINET packets ทั้งหมด
pn_rt

# ดู Real-Time data (IO data exchange)
pn_rt.frame_id >= 0x8000 && pn_rt.frame_id <= 0xBFFF

# ดู Alarms
pn_rt.frame_id >= 0xFC01 && pn_rt.frame_id <= 0xFFFE

# ดู DCP (Discovery and Configuration Protocol)
pn_dcp

# ดูเฉพาะ device ที่ต้องการ (ตาม MAC)
eth.addr == 00:1c:06:xx:xx:xx

# ดู errors
pn_rt && pn_rt.cycle_counter < 100  # หา counter discontinuity

Common PROFINET Frame Types:
├── FrameID 0x0001: PTCP Sync Master
├── FrameID 0x0020: PTCP Sync Slave
├── FrameID 0x8000-0xBFFF: RT_CLASS_1 (IO data)
├── FrameID 0xC000-0xFBFF: RT_CLASS_3/IRT (IO data)
├── FrameID 0xFC01: Alarm High
├── FrameID 0xFE01: Alarm Low
└── FrameID 0xFFFF: DCP Hello/Identify/SetName

Analyzing Jitter ด้วย Wireshark:
1. Filter: pn_rt (ดู IO data)
2. Statistics → IO Graph
3. Plot: frame count vs time
4. If irregular: jitter problem
   → สาเหตุ: overloaded switch, CPU ของ PLC busy
```

---

## Step 1107: PLC Diagnostic Programming

### Diagnostic Block ใน TIA Portal (SCL)

```
// FB_PLCDiagnostics - Comprehensive PLC Health Monitor
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_PLCDiagnostics
VAR_OUTPUT
    // CPU Health
    rCPU_Load:           REAL;          // CPU utilization %
    rMemory_Load:        REAL;          // Memory utilization %
    iDiagBuffer_Count:   INT;           // Diagnostic events count
    
    // PROFINET Health
    iPROFINET_Devices:   INT;           // Total devices
    iPROFINET_Faults:    INT;           // Devices with fault
    
    // Communication Health
    bOPCUA_Connected:    BOOL;          // OPC UA server active
    iModbus_Errors:      INT;           // Modbus error count
    
    // System Alarms
    bBatteryLow:         BOOL;          // Buffer battery low
    bForceActive:        BOOL;          // Force bits active!
    bDiagOverflow:       BOOL;          // Diag buffer full
END_VAR

VAR
    stSysInfo:      RD_SINFO_TYPE;      // System info structure
    xReadOK:        BOOL;
END_VAR

// Network 1: Read PLC System Info (ทุก 5 วินาที)
// ─────────────────────────────────────────────────────────────
// SYSINFO instตัวอย่าง (pseudo):
// SysInfo(MODE := 0, RET_VAL => iRet, INFO => stSysInfo);
// rCPU_Load   := UINT_TO_REAL(stSysInfo.CPU_LOAD);
// rMemory_Load := UINT_TO_REAL(stSysInfo.MEM_LOAD);

// Network 2: PROFINET Device Check
// ─────────────────────────────────────────────────────────────
// DeviceStates(LADDR := P#I 0.0 0, MODE := 1,
//              RET_VAL => iRet,
//              DEVICE_STATUS => aDevStatus[1..64]);
//
// iPROFINET_Devices := 0;
// iPROFINET_Faults := 0;
// FOR i := 1 TO 64 DO
//     IF aDevStatus[i].0 THEN     // Device configured
//         iPROFINET_Devices := iPROFINET_Devices + 1;
//         IF aDevStatus[i].1 THEN // Device has fault
//             iPROFINET_Faults := iPROFINET_Faults + 1;
//         END_IF;
//     END_IF;
// END_FOR;

// Network 3: Force Bit Detection
// ─────────────────────────────────────────────────────────────
// bForceActive := SFC_FORCE_ACTIVE();  // system FC
// (แจ้งเตือนถ้า force bits ยังค้างอยู่)

// Network 4: Battery Status
// ─────────────────────────────────────────────────────────────
// bBatteryLow := SFC_BAT_OK() = FALSE;
```

---

## Step 1108: Device Substitution

### Hot-Swap Device Replacement

```
PROFINET Device Substitution:
──────────────────────────────────────────────────────────────

Scenario: ET200SP failed → ต้องเปลี่ยนโดยไม่หยุด production

STANDARD REPLACEMENT (โดยไม่ต้อง program ใหม่):
1. Device ใหม่ใส่เข้าไปใน network
2. Device ต้องมี Profibus Address ว่าง
3. IO Controller (PLC) จะ:
   a. Detect new device ที่ MAC address ใหม่
   b. Send device name ให้อัตโนมัติ (ถ้า name ว่าง)
   c. Send IP configuration
   d. Download GSD parameters
4. Device ออนไลน์ภายใน 10-30 seconds
5. ⚠️ หมายเหตุ: device ต้องเป็น same hardware model

STEPS บน TIA Portal:
1. PLC → Accessible devices → ดู new device MAC
2. Assign station name ให้ตรงกับ configuration
3. Device จะ download config อัตโนมัติ

สำหรับ Safety Devices (F-Devices):
├── ต้องทำ F-parameter re-download
├── ต้องมีการ acknowledge โดย authorized person
└── Safety parameter verification required

SPARE DEVICE PREPARATION:
เก็บ spare device ไว้โดย:
├── Flash IP/Name ลงไปล่วงหน้า (ถ้าไม่ใช้ topology config)
└── หรือเก็บ blank device ไว้ (ใช้ DCP เพื่อ assign name)
```

---

## Step 1109: Automatic Address Assignment

### PROFINET Automatic Configuration

```
PROFINET Topology-based Address Assignment:
──────────────────────────────────────────────────────────────

Concept:
- ใน topology configuration: กำหนดว่า device นี้ต่ออยู่ที่ Switch X, Port Y
- ถ้า device ถูก replace: PLC รู้ตำแหน่งจาก topology
- PLC assign ชื่อและ IP ให้อัตโนมัติตาม topology

ข้อดี:
├── ไม่ต้อง configure device ใหม่ด้วยมือ
├── ป้องกัน wrong device in wrong slot
└── เร็วกว่า manual assignment

Setting up Topology Configuration (TIA Portal):
1. Network View → Topology View
2. วาดเส้น (assign ports): device1 port1 → switch port 3
3. ทำซ้ำสำหรับทุก device
4. Compile และ download hardware config
5. เปิดใช้งาน: Properties → Automatic topology detection

Verification:
Online → Network View → Topology → Compare expected vs actual
├── Green lines: match OK
├── Red lines: topology mismatch (wrong cable)
└── Missing lines: device not found
```

---

## Step 1110: PROFINET MRP Ring

### MRP (Media Redundancy Protocol)

```
MRP Ring Implementation:
──────────────────────────────────────────────────────────────

RING TOPOLOGY:
Switch1 (MRM) ── Switch2 ── Switch3 ── Switch4 ── (back to Switch1)
    │
    PLC connects here (not in ring)

ROLES:
MRM (Media Redundancy Manager):
├── จัดการ ring: detect open/closed state
├── Block one port ตอน ring intact (ป้องกัน loop)
├── Open blocked port เมื่อ link fail (restore connectivity)
└── Usually: core switch ที่ต่อกับ PLC

MRC (Media Redundancy Client):
├── Monitor ring, report status
└── All other switches in ring

Configuration (Siemens SCALANCE X-200):
──────────────────────────────────────────────────────────────
Via TIA Portal (PROFINET topology):
Switch → Properties → Media Redundancy
├── MRP Domain: "mrpdomain-1" (ต้องตรงกันทุก switch ใน ring)
├── Role: Manager หรือ Client
├── Ring port 1: ระบุ port ที่ต่อไปยัง ring (วนซ้าย)
└── Ring port 2: ระบุ port ที่ต่อไปยัง ring (วนขวา)

Via Web Interface (SCALANCE):
System → Redundancy → MRP
├── MRP Mode: Manager/Client
├── Domain UUID: must match all switches
└── Ring Ports: select ports P1 and P2

Testing:
1. Verify ring closed: MRM shows "Ring closed"
2. Disconnect one cable → MRM shows "Ring open"
3. Recovery time: < 200ms (IEC 62439-2)
4. Reconnect cable → MRM shows "Ring closed" again
5. Data: no observable disruption > 200ms
```

---

## Step 1111: PROFINET IRT (Isochronous Real-Time)

### IRT สำหรับ Motion Control

```
PROFINET IRT vs RT:
──────────────────────────────────────────────────────────────

RT (Real-Time) - Class 1:
├── Cycle time: 1-512ms
├── Jitter: < 1ms
├── ใช้สำหรับ: process automation, IO data
└── Standard switches OK

IRT (Isochronous Real-Time) - Class 3:
├── Cycle time: 31.25μs - 4ms
├── Jitter: < 1μs (microsecond precision!)
├── ใช้สำหรับ: Motion control, CNC, robotics
├── ต้องการ: IRT-capable switches (SCALANCE X-300 IRT, X-400 IRT)
└── Requires: Precise time synchronization (IEEE 1588 PTP)

IRT Requirements:
Hardware:
├── IRT-capable controller: S7-1500T, S120 drive system
├── IRT-capable switches: SCALANCE X-300/X-400 IRT
└── IRT-capable IO devices

Configuration (TIA Portal):
PLC Properties → Advanced Settings → Real-time settings
├── ORT mode: Isochronous mode
├── Send clock: 1ms (minimum)
└── Synchronization role: Sync Master (controller)

Switch Configuration:
SCALANCE X → Properties → Isochronous mode
├── Synchronization: PTCP (PROFINET-specific PTP)
└── Reserved bandwidth for IRT traffic

Application:
Motion Axis → ใช้ IRT sync สำหรับ exact position sync
Example: CNC machine ต้องการทุก axis ทำงานพร้อมกัน
         IRT ensures < 1μs jitter → smooth coordinated motion
```

---

## Step 1112: Diagnostic Alarm Types

### PROFINET Alarm Classification

```
PROFINET Alarm Types (IEC 61158):
──────────────────────────────────────────────────────────────

ALARM TYPE 1: Diagnosis Alarm
├── Cause: Channel fault (wire break, short circuit)
├── Example: "AI Channel 5 Wire Break"
└── Auto-generated by module hardware

ALARM TYPE 2: Process Alarm  
├── Cause: Process-level event programmed in device
├── Example: "Temperature limit exceeded"
└── อาจ program ใน IO device firmware

ALARM TYPE 3: Pull Alarm
├── Cause: Module physically removed
├── Example: "Slot 3 Module pulled"
└── Auto-generated

ALARM TYPE 4: Plug Alarm
├── Cause: Module inserted
├── Example: "Slot 3 Module plugged"
└── Auto-generated

ALARM TYPE 5: Status Alarm
├── Cause: Device status change (not a fault)
└── Example: "Device maintenance required"

ALARM TYPE 6: Update Alarm
├── Cause: Firmware/parameter update
└── Example: "Device parameter downloaded"

Reading Alarms ใน PLC Program:
// OB82: Diagnosis Alarm OB (called when alarm received)
// ─────────────────────────────────────────────────────────────

ORGANIZATION_BLOCK OB82_DiagnosisAlarm
VAR_TEMP
    xChannelDiag:   BOOL;
    xModuleDiag:    BOOL;
    wDiagAddress:   WORD;       // Diagnostic address (hardware ID)
    wAlarmType:     WORD;       // Alarm type
END_VAR

BEGIN
    // อ่าน OB82 start info
    wDiagAddress := OB82_MDL_ADDR;   // Module address
    wAlarmType   := OB82_EV_CLASS;   // Event class
    xChannelDiag := OB82_IO_FLAG;    // I/O flag
    
    // Log ไปยัง diagnostic buffer
    DB_DiagLog.alarms[DB_DiagLog.count].address   := wDiagAddress;
    DB_DiagLog.alarms[DB_DiagLog.count].alarm_type := wAlarmType;
    DB_DiagLog.alarms[DB_DiagLog.count].timestamp  := NOW();
    DB_DiagLog.count := DB_DiagLog.count + 1;
    
    // Set alarm indicator
    M_PROFINET_ALARM := TRUE;
END_ORGANIZATION_BLOCK
```

---

## Step 1113: Diagnostic Display บน HMI

### แสดง Diagnostic ใน WinCC/FactoryTalk

```
HMI Diagnostic Display Design:
──────────────────────────────────────────────────────────────

OVERVIEW PAGE (หน้าหลัก):
┌─────────────────────────────────────────────────────────────┐
│  SYSTEM STATUS                            ● PROFINET: OK    │
│  ──────────                               ● Safety: OK      │
│                                           ● OPC UA: Active  │
│  ┌─────────────────────────────────────┐                    │
│  │  ACTIVE ALARMS (3)                  │                    │
│  │  ● CRITICAL: AI1 Ch5 Wire Break    │                    │
│  │  ● WARNING:  Drive U1 Temp High    │                    │
│  │  ● INFO:     Maintenance Due P101  │                    │
│  └─────────────────────────────────────┘                    │
│                                                              │
│  IO OVERVIEW:                                                │
│  Unit 1 [🟢]  Unit 2 [🟢]  Utilities [🟡]  Safety [🟢]    │
└─────────────────────────────────────────────────────────────┘

DEVICE DETAIL PAGE:
┌─────────────────────────────────────────────────────────────┐
│  ET200SP RACK - UNIT 1                                      │
│  Device: ET200SP_U1  IP: 192.168.20.11                     │
│  Status: ⚠️ WARNING                                         │
│                                                              │
│  SLOT│ MODULE          │ STATUS │ DETAIL                   │
│  ────┼─────────────────┼────────┼─────────────────────────  │
│    1 │ CPU 1512SP-1 PN │  OK    │                          │
│    2 │ DI 8×24VDC      │  OK    │                          │
│    3 │ DO 8×24VDC/0.5A │  OK    │                          │
│    4 │ AI 4×U/I/RTD    │  ⚠️    │ CH5: Wire Break          │
│    5 │ AO 4×U/I        │  OK    │                          │
│    6 │ Power Module    │  OK    │                          │
└─────────────────────────────────────────────────────────────┘

// WinCC Scripting สำหรับ Dynamic Status Display:
// Script ที่อ่าน tag และ update สี (green/yellow/red)
// ตาม device status จาก PLC
```

---

## Step 1114: PROFINET Redundancy (S2 Redundancy)

### System Redundancy (S2) สำหรับ High Availability

```
PROFINET S2 Redundancy:
──────────────────────────────────────────────────────────────

S2 CONFIGURATION:
├── 2 IO Controllers (active + standby)
├── Both connected to same IO Devices
├── Failover: < 200ms
└── ใช้กับ: S7-1500R/H (Redundant CPU)

TOPOLOGY:
                    ┌──────────────┐
                    │ IO Controller │ ← Primary
                    │ (S7-1517R)   │
                    └──────┬───────┘
                           │ PROFINET
     ┌─────────────────────┼───────────────────────┐
     │                     │                        │
┌────▼────┐          ┌─────▼─────┐          ┌─────▼─────┐
│ET200SP  │          │ET200SP    │          │ET200SP    │
│(S2 dev) │          │(S2 dev)   │          │(S2 dev)   │
└─────────┘          └───────────┘          └───────────┘
     │                     │                        │
     │ Second PROFINET ring                          │
     │                     │                        │
                    ┌──────▼───────┐
                    │ IO Controller │ ← Standby
                    │ (S7-1517R)   │
                    └──────────────┘

S2 Operation:
├── Primary: ส่ง IO data ปกติ
├── Standby: monitor สถานะ primary
├── Failover: standby takes over ถ้า primary fail
└── Re-sync: primary กลับมา → rejoin as standby

Requirements สำหรับ IO Device:
├── S2-capable device (ดูใน catalog: "Supported redundancy: S2")
├── 2 ports: เชื่อมต่อกับทั้ง 2 controllers
└── ตัวอย่าง: ET200SP HA, Siemens drives ที่ S2 capable
```

---

## Step 1115: Siemens Diagnostic Buffer

### Reading Diagnostic Buffer

```
// PLC Diagnostic Buffer - อ่านด้วย SFC51
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_ReadDiagBuffer
VAR_OUTPUT
    iEventCount:        INT;            // จำนวน events ใน buffer
    aEvents:            ARRAY[1..50] OF DB_EVENT;  // Events array
END_VAR

VAR
    stDiagBuf:          DIAG_BUF_TYPE;  // Diagnostic buffer structure
    iRet:               INT;
END_VAR

// Network 1: Read Diagnostic Buffer
// SFC51 RDSYSST (Read System Status):
// CALL SFC51
//     REQ := TRUE
//     SZL_ID := W#16#A0D0    // Diagnostic buffer system list
//     INDEX := W#16#0000
//     SZL_HEADER.LENGTHDR => iRecordLen
//     DR := DB_DiagBuf.events
//     RET_VAL => iRet
//
// SZL ID List for S7-1500:
// W#16#A092: Module state
// W#16#A0A0: Event log
// W#16#A0D0: Diagnostic buffer (most important)

// Network 2: Parse Events
// iEventCount := stDiagBuf.COUNT;
// FOR i := 1 TO iEventCount DO
//     aEvents[i].timestamp := stDiagBuf.events[i].TIMESTAMP;
//     aEvents[i].event_id  := stDiagBuf.events[i].EVENT_ID;
//     aEvents[i].info1     := stDiagBuf.events[i].ADD_INFO_1;
// END_FOR;

// Event ID Examples:
// 16#C80A: IO Device communication fault
// 16#E225: Module removed
// 16#C80B: IO Device communication restored
// 16#E226: Module plugged
// 16#FE2F: Voltage failure
// 16#FE30: Voltage restored
```

---

## Step 1116: Allen-Bradley EtherNet/IP Diagnostics

### Studio 5000 Diagnostics

```
Allen-Bradley / Rockwell - Diagnostic Tools:
──────────────────────────────────────────────────────────────

1. STUDIO 5000 LOGIX DESIGNER:
   Controller → Properties → Diagnostics tab
   ├── Controller fault table
   ├── Task monitoring
   ├── Memory usage
   └── Connections: ดู all EtherNet/IP connections

2. RSLINX CLASSIC / FACTORYTALKS LINX:
   ├── Connection browser: ดู OPC tags
   ├── Communication diagnostics
   └── Firmware version

3. ALLEN-BRADLEY NETWORK DIAGNOSTICS:
   ControlLogix → Backplane Tab:
   ├── ดู all modules ใน chassis
   ├── Module status (green/yellow/red)
   └── Error codes

4. FAULT CODE READING (Ladder):
   // Instruction: GSV (Get System Value) อ่าน fault code
   
   //  Always ON
   //  ──┤├──────── [GSV]─────────────────────────────
   //               Class: CONTROLLER
   //               Attribute: MajorFaultRecord
   //               Dest: fault_record_tag
   
   // fault_record_tag.Code → รหัสความผิดพลาด
   // fault_record_tag.Program → Program ที่เกิด error
   // fault_record_tag.Routine → Routine ที่เกิด error

5. EtherNet/IP DIAGNOSTIC COUNTERS:
   Module Properties → Connection tab:
   ├── Input Connection Timeout Count → ดู connection losses
   ├── Output Connection Timeout Count
   └── Connection Failures
```

---

## Step 1117: Diagnostic Program ตัวอย่าง (Allen-Bradley)

### Comprehensive Diagnostics ใน Studio 5000

```
// Rockwell ControlLogix - System Diagnostic Rung
// ─────────────────────────────────────────────────────────────

// Rung 1: Read Controller Status
// ─────────────────────────────────────────────────────────────
//
//  ┌──────────────────────────────────────────────────────┐
//  │  GSV                                                  │
//  │  Get System Value                                     │
//  │  Class Name      CONTROLLER                          │
//  │  Attribute Name  MajorFaultRecord                    │
//  │  Dest            Diag.MajorFault                     │
//  └──────────────────────────────────────────────────────┘
//  ──────────────────────────────────────────────────────────(ENO)──
//
// Diag.MajorFault.Code ≠ 0 → Major fault active

// Rung 2: Check EtherNet/IP Module Status
// ─────────────────────────────────────────────────────────────
//
//  "1756-EN2T_Module:I.MajorFault" (input status)
//  ──┤├────────────────────────────────────────────(EthernetFault)──

// Rung 3: Network Fault Summary
// ─────────────────────────────────────────────────────────────
//  Network_Module1.Connection_Status.Fault   
//  ──┤├──┬──────────────────────────────────────(Network_Fault_Any)──
//        │
//  Network_Module2.Connection_Status.Fault
//  ──┤├──┘

// Rung 4: Compute System Health Score (0-100)
// ─────────────────────────────────────────────────────────────
//  ┌──────────────────────────────────────────────────────┐
//  │  CPT (Compute)                                        │
//  │  Dest: System_Health_Score                           │
//  │  Expression:                                          │
//  │    100 - (Major_Fault * 30)                          │
//  │       - (Comm_Fault_Count * 5)                       │
//  │       - (IO_Error_Count * 2)                         │
//  └──────────────────────────────────────────────────────┘

// Rung 5: Health Score Alarm
// ─────────────────────────────────────────────────────────────
//  System_Health_Score < 70                System_Health_Alarm
//  ──────────────────────────────────────────────────(  )──────

// Rung 6: Reset Diagnostics Counter
// ─────────────────────────────────────────────────────────────
//  Reset_Diag_PB    One_Shot
//  ──┤├──────────────(P)──┬────────── [MOV 0 → Comm_Fault_Count]──
//                         └────────── [MOV 0 → IO_Error_Count]────
```

---

## Step 1118: Predictive Diagnostics

### ทำนายปัญหาก่อนเกิด

```
Predictive Diagnostics Concept:
──────────────────────────────────────────────────────────────

ความแตกต่าง:
├── Reactive: เสียแล้วค่อยซ่อม (costly downtime)
├── Preventive: ซ่อมตามกำหนด (อาจสิ้นเปลือง)
└── Predictive: ซ่อมเมื่อจะเสีย (optimal timing)

Predictive Indicators จาก PROFINET/PLC Data:
──────────────────────────────────────────────────────────────

CONTACTOR (Magnetic starter):
├── Count: จำนวนครั้งที่ ON/OFF (ดูจาก counter ใน program)
├── Expected life: 1,000,000 operations
├── Alert: เมื่อถึง 800,000 operations
└── Program: นับ rising edge ของ coil output

SERVO DRIVE (PROFINET drive):
├── Torque vs speed: deviation จาก normal curve
├── Current harmonic: เพิ่มขึ้นเมื่อ bearing wear
├── Position error: เพิ่มขึ้นเมื่อ mechanical wear
└── Drive reads: จาก PROFINET parameter records

VALVE ACTUATOR (Smart valve):
├── Signature Analysis: pressure drop pattern
├── Response time: เพิ่มขึ้นเมื่อ packing wear
└── Current: เพิ่มขึ้นเมื่อ valve sticking
```

### Contactor Life Tracking Program

```
// Contactor Life Tracking ใน SCL
// ─────────────────────────────────────────────────────────────

FUNCTION_BLOCK FB_ContactorLife
VAR_INPUT
    xCoilOutput:        BOOL;           // Contactor coil output
    iRatedOperations:   INT := 1000000; // Rated life (100万次)
    iWarningPercent:    INT := 80;      // Warning at 80%
END_VAR

VAR_OUTPUT
    iOperationCount:    INT;            // Total operations
    rLifeRemaining:     REAL;           // % life remaining
    bWarning:           BOOL;           // Near end of life
    bEndOfLife:         BOOL;           // Past rated life
END_VAR

VAR RETAIN
    iSavedCount:        INT;            // Saved to NVRAM
END_VAR

VAR
    xCoilLastScan:      BOOL;           // Previous scan value
    iWarnThreshold:     INT;
END_VAR

// Network 1: Count rising edges (OFF→ON transitions)
// ─────────────────────────────────────────────────────────────
// IF xCoilOutput AND NOT xCoilLastScan THEN
//     iSavedCount := iSavedCount + 1;  // RETAIN: survives power cycle
// END_IF;
// xCoilLastScan := xCoilOutput;
// iOperationCount := iSavedCount;

// Network 2: Calculate remaining life
// ─────────────────────────────────────────────────────────────
// rLifeRemaining := 100.0 × (1.0 - INT_TO_REAL(iSavedCount) /
//                             INT_TO_REAL(iRatedOperations));
// rLifeRemaining := LIMIT(0.0, rLifeRemaining, 100.0);

// Network 3: Alarms
// ─────────────────────────────────────────────────────────────
// iWarnThreshold := iRatedOperations × iWarningPercent / 100;
// bWarning   := iSavedCount >= iWarnThreshold;
// bEndOfLife := iSavedCount >= iRatedOperations;
```

---

## Step 1119: Complete Diagnostic Program

### Enterprise Diagnostic System

```
// Complete Diagnostic System - OB100 (Startup) + OB1 (Main)
// ─────────────────────────────────────────────────────────────

// DB_SystemDiag: Main diagnostic data structure
DATA_BLOCK DB_SystemDiag
    // Overall System Health
    rSystemHealth:          REAL;           // 0-100%
    xSystemOK:              BOOL;
    xMaintenanceRequired:   BOOL;
    
    // PROFINET Status
    iPROFINET_TotalDevices: INT;
    iPROFINET_ActiveFaults: INT;
    xPROFINET_RingClosed:   BOOL;
    
    // IO Channel Health
    iChannelFaults:         INT;            // Total channel faults
    aChannelFaultList:      ARRAY[1..20] OF CHANNEL_FAULT;
    
    // Communication Health
    iModbusFaults:          INT;
    iOPCUA_Connections:     INT;
    
    // Equipment Life Tracking
    aContactorLife:         ARRAY[1..10] OF REAL; // % remaining
    
    // Maintenance Schedule
    xMaintenanceDue:        BOOL;
    dtNextMaintenance:      DATE_AND_TIME;
END_DATA_BLOCK

// Main Diagnostic FB Call (ใน OB1):
// ─────────────────────────────────────────────────────────────
//  Always ON
//  ──┤├────── [FB_SystemDiag] ──────────────────────────────
//              EN := TRUE
//              SCAN_CYCLE_MS := %IW256   // Actual scan time
//              (all outputs update DB_SystemDiag)

// System Health Calculation:
// rSystemHealth = 100 
//               - (iPROFINET_ActiveFaults × 10)    // Each PN fault: -10
//               - (iChannelFaults × 2)              // Each channel fault: -2
//               - (iModbusFaults × 5)               // Each Modbus fault: -5
//               - (xMaintenanceDue ? 5 : 0)         // Maintenance due: -5
```

---

## Step 1120: Lab Project - Diagnostic System Implementation

### Lab: Full PROFINET Diagnostic System

**โจทย์**: สร้างระบบ Diagnostic สำหรับ Mixing & Filling Line

**System Components**:
- 1x Siemens S7-1516 PLC (Controller)
- 3x ET200SP (Field IO stations)
- 2x SINAMICS G120 Drives (via PROFINET)
- 1x SCALANCE XC208 (Managed switch, MRP capable)

```
LAB TASKS:
══════════════════════════════════════════════════════════════

Task 1: MRP Ring Configuration (20%)
─────────────────────────────────────────────────────────────
1. Configure SCALANCE XC208 เป็น MRM
2. Configure IO stations เป็น MRC
3. Verify ring closed
4. Test failover: ดึงสายแล้ว วัดเวลา recovery

Task 2: Channel Diagnostics Programming (30%)
─────────────────────────────────────────────────────────────
1. Program OB82 (Diagnosis Alarm OB)
   - Log alarm type, address, timestamp
   - Set diagnostic bit per device
2. Program FB ที่อ่าน device status ทุก 5 วินาที
3. Test: ดึงสายขาดจาก AI → ดู OB82 activate
4. Test: ถอด module → ดู module alarm

Task 3: HMI Diagnostic Display (25%)
─────────────────────────────────────────────────────────────
1. Design Diagnostic Overview Page บน WinCC
   - แสดงสถานะทุก IO station (green/yellow/red)
   - Active alarm list (last 20 alarms)
   - PROFINET ring status
2. Design Device Detail Page
   - Module status per slot
   - Channel status per module
   - Last 5 events per device

Task 4: Predictive Diagnostics (25%)
─────────────────────────────────────────────────────────────
1. Implement contactor life counter สำหรับ 3 contactors
2. Drive diagnostic reading (PROFINET parameter read):
   - Drive temperature
   - Drive output current vs rated
   - DC Bus voltage
3. Create maintenance alert ที่ HMI
4. Test: simulate 900,000 operations → alert แสดง

DELIVERABLES:
├── TIA Portal project (zap backup)
├── WinCC project backup
├── Test report (ทุก test case pass/fail)
├── Diagnostic program documentation
└── Video demonstration (5 นาที)

PASS CRITERIA:
├── MRP recovery < 200ms (measured)
├── OB82 correctly triggered สำหรับ channel faults
├── HMI shows correct status within 2 scans
├── Contactor counter accurate (± 1 count)
└── Documentation complete
```

---

## สรุป Part 56

| หัวข้อ | Key Points |
|--------|-----------|
| PROFINET Diagnostics | Channel → Module → Device → Network levels |
| MRP Ring | Redundancy < 200ms, needs managed switches |
| IRT | Microsecond jitter สำหรับ motion control |
| OB82 | Alarm OB สำหรับ PROFINET alarms |
| Predictive | นับ operations, monitor trends |

---

## Navigation

- [← Part 55: Digital Twin](./part55_digital_twin.md)
- [→ Part 57: Energy Optimization](./part57_energy_optimization.md)
- [↑ สารบัญหลัก](../README.md)
