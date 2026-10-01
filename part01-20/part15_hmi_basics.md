# Part 15: HMI พื้นฐาน (Steps 271-290)

## ภาพรวมของบทเรียน

HMI (Human-Machine Interface) คือสะพานเชื่อมระหว่างผู้ปฏิบัติงานกับระบบ PLC ในบทนี้เราจะเรียนรู้ตั้งแต่ประเภทของ HMI, การสื่อสารกับ PLC, การออกแบบหน้าจอ, ไปจนถึงการสร้าง Screen สำหรับงานอุตสาหกรรมจริง

---

## Step 271: ประเภทของ HMI

### HMI แบ่งตามลักษณะ

```
1. Operator Panel / Push Button Station
   ┌─────────────────────────────────────────┐
   │   [START]  [STOP]  [RESET]              │
   │    ●GREEN   ●RED    ●YELLOW             │
   │                                         │
   │   ═══ STATUS BAR ═══                    │
   │    RUN □  STOP □  FAULT □              │
   └─────────────────────────────────────────┘
   - ง่าย, ราคาถูก
   - ไม่มีการตั้งค่า
   - ใช้สำหรับงานง่ายๆ

2. Text Panel (Character Display)
   ┌─────────────────────────────────────────┐
   │  CONVEYOR STATUS                        │
   │  SPEED: 15.5 m/min    [F1][F2][F3][F4]│
   │  STATUS: RUNNING      ↑  ↓  ←  →     │
   └─────────────────────────────────────────┘
   - แสดงข้อความ ASCII
   - ราคาปานกลาง
   - ตั้งค่าได้บางส่วน

3. Graphic Panel (Touchscreen)
   ┌─────────────────────────────────────────┐
   │  ┌──────────────────────────────────┐  │
   │  │  [GRAPHIC DISPLAY]               │  │
   │  │  Motor: ● Running   Speed: 25rpm │  │
   │  │  Temp:  85°C        Load: 75%    │  │
   │  │  [START]  [STOP]  [SETTINGS]    │  │
   │  └──────────────────────────────────┘  │
   └─────────────────────────────────────────┘
   - แสดงกราฟิก
   - Touch Screen
   - เป็นที่นิยมมากที่สุด

4. Panel PC / Industrial PC
   - PC ที่ออกแบบสำหรับอุตสาหกรรม
   - ระบบปฏิบัติการ Windows/Linux
   - รัน SCADA Software ได้
   - ราคาสูง แต่ยืดหยุ่นสูง
```

### ยี่ห้อ HMI ยอดนิยมในไทย

| ยี่ห้อ | รุ่นยอดนิยม | ราคาประมาณ |
|-------|------------|-----------|
| Siemens | KTP400, KTP700, KTP1200 | 8,000-80,000 บาท |
| Allen-Bradley | PanelView 800/1000/1500 | 15,000-120,000 บาท |
| Mitsubishi | GOT2000 Series | 10,000-100,000 บาท |
| Weintek | MT8000 Series | 5,000-50,000 บาท |
| Delta | DOP-100 Series | 5,000-40,000 บาท |

---

## Step 272: HMI-PLC Communication

### Communication Protocols

```
Protocol Stack:

HMI ←──────────────────────────────────────── PLC
     ↕ Application Protocol (PLC-specific)
     ↕ Transport Protocol (Ethernet TCP/IP หรือ RS-485)
     ↕ Physical Layer (Cable)

Common Protocols:
─────────────────────────────────────────────────────────
PLC Brand    │ Protocol         │ Hardware
─────────────────────────────────────────────────────────
Siemens      │ S7 Communication │ PROFINET (Ethernet)
             │ PROFIBUS DP      │ RS-485 (PROFIBUS)
─────────────────────────────────────────────────────────
Allen-Bradley│ EtherNet/IP      │ Ethernet
             │ DF1              │ RS-232/RS-485
─────────────────────────────────────────────────────────
Mitsubishi   │ MC Protocol      │ Ethernet
             │ GOT Connection   │ RS-422/RS-232
─────────────────────────────────────────────────────────
Universal    │ Modbus TCP       │ Ethernet
             │ Modbus RTU       │ RS-485
─────────────────────────────────────────────────────────
```

### Siemens HMI Connection Setup

```
TIA Portal Connection Configuration:

Step 1: เพิ่ม HMI Device ใน TIA Portal
- Project → Add New Device → HMI → KTP700 Basic

Step 2: Configure Connection
- HMI → Connections → Add Connection
- Interface: PROFINET
- PLC: CPU_1214C
- IP Address PLC: 192.168.1.1
- IP Address HMI: 192.168.1.2

Step 3: Map Tags
- HMI Tags → Add Tag → PLC Tag
- เลือก PLC Address เช่น %M0.0, %DB1.DBX0.0

Step 4: Test Connection
- Online Mode → Check Communication
- Monitor Tags
```

### Network Diagram

```
Factory Network Layout:

    PC (Engineering)         HMI Station 1      HMI Station 2
         │                        │                   │
         │                        │                   │
    ─────┴────────────────────────┴───────────────────┴──────
                              Ethernet Switch
                          (192.168.1.xxx subnet)
    ─────┬────────────────────────┬───────────────────────────
         │                        │
      PLC Master              Remote I/O
      (192.168.1.1)           (PROFINET)
```

---

## Step 273: Tag Mapping

### Tag คืออะไร

Tag คือตัวแปรที่ HMI ใช้สื่อสารกับ PLC

```
Tag Types:
1. Internal Tag - เก็บข้อมูลใน HMI เท่านั้น (ไม่ส่ง PLC)
2. External Tag (PLC Tag) - อ่าน/เขียน ไปยัง PLC

Tag Data Types:
- Bool  (1 bit): ON/OFF, True/False
- Byte  (8 bits): 0-255
- Int   (16 bits): -32768 to 32767
- DInt  (32 bits): Large integers
- Real  (32 bits): Floating point (3.14)
- String: ข้อความ
```

### Tag Mapping Table

```
HMI Tag Name    │ PLC Address  │ Data Type │ Direction
────────────────────────────────────────────────────────
Conv1_Run       │ %Q0.0        │ Bool      │ Read
Conv2_Run       │ %Q0.1        │ Bool      │ Read
Conv3_Run       │ %Q0.2        │ Bool      │ Read
ESTOP_Status    │ %M0.0        │ Bool      │ Read
Seq_Start_CMD   │ %M1.1        │ Bool      │ Read/Write
Seq_Stop_CMD    │ %M1.2        │ Bool      │ Read/Write
Conv1_Speed_SP  │ %DB1.DBD0    │ Real      │ Read/Write
Conv1_Speed_FB  │ %DB1.DBD4    │ Real      │ Read
Product_Count   │ %DB1.DBD8    │ DInt      │ Read
Fault_Code      │ %MW100       │ Int       │ Read
Fault_Reset_CMD │ %M10.0       │ Bool      │ Write
Motor_Temp      │ %DB1.DBD20   │ Real      │ Read
────────────────────────────────────────────────────────
```

### Siemens Tag Access

```
// Siemens TIA Portal - Tag Configuration

PLC Tag:
  Name: Conv1_Speed_Setpoint
  PLC: CPU_1214C
  Address: %DB1.DBD0    ← DB1, DWORD at Offset 0
  Data Type: Real
  Acquisition Cycle: 1s

HMI Tag:
  Name: HMI_Speed_SP
  Connection: PLC_1 (CPU_1214C)
  PLC Tag: Conv1_Speed_Setpoint
  Display: Numeric Input (0.0 to 100.0)
```

---

## Step 274: Screen Design Basics

### หลักการออกแบบหน้าจอ HMI

```
Design Principles:

1. Hierarchy (ลำดับชั้น)
   Main Menu → Sub Menu → Detail Screen
   
2. Consistency (ความสม่ำเสมอ)
   - สี Button เหมือนกันทุกหน้า
   - ตำแหน่ง Navigation เหมือนกัน
   - Font เหมือนกัน

3. Color Convention (มาตรฐานสี)
   Green  = Running, Normal, OK
   Red    = Fault, Stop, Emergency
   Yellow = Warning, Standby
   Blue   = Idle, Maintenance
   White  = Off, Manual Inactive

4. Layout Rules
   - Navigation Bar: ด้านบนหรือด้านล่าง
   - Status Bar: บรรทัดบนสุดหรือล่างสุด
   - Controls: กลางหน้าจอ
   - Alarms: มุมล่างขวา
```

### Standard Screen Layout

```
HMI Screen Layout (800×600):

┌──────────────────────────────────────────────────────────────┐
│  STATUS BAR: [Machine Name] [Time: 14:30] [Alarm Count: 0]  │ ← 40px
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                    MAIN CONTENT AREA                        │ ← 480px
│                    (Process Graphics)                        │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  [HOME] [ALARM] [TREND] [RECIPE] [MAINT] [SETTINGS] [LOGIN] │ ← 80px
└──────────────────────────────────────────────────────────────┘
                          800px Wide
```

---

## Step 275: Buttons และ Indicators

### Button Types

```
Button Types ใน HMI:

1. Momentary Button (กดแล้วปล่อย)
   ┌──────────────┐
   │   START      │   ← กด = ON, ปล่อย = OFF
   │   (MOM)      │   ใช้สำหรับ Trigger
   └──────────────┘

2. Maintained Button (Toggle)
   ┌──────────────┐
   │   RUN MODE   │   ← กดครั้งแรก = ON
   │   [■]        │   กดอีก = OFF
   └──────────────┘

3. Bit Button (เขียน 1 Bit)
   ┌──────────────┐
   │   SET SPEED  │   ← เขียน 1 bit เมื่อกด
   │   TO 50%     │   ไม่ต้อง Release
   └──────────────┘

4. Numeric Entry Button
   ┌──────────────┐
   │  Speed:      │   ← กด = แสดง Keypad
   │  [  50.0  ]  │   ป้อนตัวเลขได้
   └──────────────┘
```

### Indicator Types

```
Indicator Types:

1. Bit Indicator (ON/OFF)
   ┌────────────────┐
   │    Motor 1     │   Green = Running
   │    ●  RUN      │   Red = Fault
   │                │   Gray = Stopped
   └────────────────┘

2. Multi-State Indicator
   ┌────────────────┐
   │    Mode:       │
   │  ██ AUTO       │   0 = MANUAL (Blue)
   │                │   1 = AUTO (Green)
   └────────────────┘   2 = MAINT (Yellow)

3. Animated Indicator
   ┌────────────────┐
   │  ████████████  │   Conveyor Belt Animation
   │  →→→→→→→→→→→   │   Moving when Running
   │  ████████████  │
   └────────────────┘
```

### Siemens HMI Scripting for Buttons

```
// Siemens WinCC Script for Button Press
// Button: Start_Button
// Event: Click

// Script (VBScript or JavaScript)

Sub OnStartButton_Click()
    // Set Start Command
    SmartTags("Seq_Start_CMD") = 1
    
    // Log the action
    LogMessage "Operator pressed START at " & Now
    
    // Provide feedback
    SetScreenTitle "System Starting..."
End Sub

Sub OnStopButton_Click()
    SmartTags("Seq_Stop_CMD") = 1
End Sub
```

---

## Step 276: Numeric Display

### Numeric Display Types

```
1. Numeric I/O Field (แสดงและป้อนตัวเลข)
   ┌──────────────────────────────────────────┐
   │  Conveyor Speed Setting                  │
   │                                          │
   │  Current: [ 25.5 ] m/min                │
   │                                          │
   │  Set:    [_____] m/min  [ENTER] [CANCEL] │
   └──────────────────────────────────────────┘

2. Symbolic I/O Field (แสดงข้อความตาม Value)
   ┌──────────────────────────────────────────┐
   │  System Status                           │
   │                                          │
   │  0 = STOPPED      ← Value 0             │
   │  1 = RUNNING      ← Value 1             │
   │  2 = FAULT        ← Value 2             │
   │                                          │
   │  Current: [  RUNNING  ]                  │
   └──────────────────────────────────────────┘

3. Bar Graph (แสดงเป็น Bar)
   ┌──────────────────────────────────────────┐
   │  Tank Level: 75%                         │
   │                                          │
   │   100%│                                  │
   │    75%│████████████████████████          │
   │    50%│                                  │
   │    25%│                                  │
   │     0%│                                  │
   └──────────────────────────────────────────┘
```

### Numeric Limit Checking

```
ใน TIA Portal/WinCC:
Tag Properties:
  Min Value: 0.0
  Max Value: 100.0
  
High Limit: 90.0 → เปลี่ยนสีแดงเมื่อ > 90
Low Limit: 10.0  → เปลี่ยนสีแดงเมื่อ < 10
```

---

## Step 277: Process Graphic Design

### Piping and Instrumentation Diagram (P&ID) บน HMI

```
Tank Level Control Screen:

┌─────────────────────────────────────────────────────────────────┐
│  Tank Level Control           [Alarm: 0]   [Time: 14:30]       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     Feed Pump                                                   │
│     [P-101]                                                     │
│       │                                                         │
│     ══╪══ ←── Valve V-101                                       │
│       │              [OPEN] [CLOSE]                             │
│       ▼                                                         │
│    ┌──────────────────────────────────────────────────────┐    │
│    │                                                      │    │
│    │  TANK T-101        Level: [  75.2  ] %              │    │
│    │                                                      │    │
│    │  ████████████████████████  75%                      │    │
│    │  ████████████████████████                            │    │
│    │  ████████████████████████                            │    │
│    │  ████████████████████████                            │    │
│    │                                                      │    │
│    │  Volume: [  3750  ] Liters   Max: 5000L             │    │
│    └──────────────────────────────────────────────────────┘    │
│       │                                                         │
│    ══╪══ ←── Valve V-102                                       │
│       │              [OPEN] [CLOSE]                             │
│       ▼                                                         │
│     Discharge                                                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  [HOME] [TRENDS] [ALARMS] [SETTINGS]                           │
└─────────────────────────────────────────────────────────────────┘
```

### Conveyor Overview Screen

```
Conveyor System Overview Screen:

┌─────────────────────────────────────────────────────────────────┐
│  CONVEYOR SYSTEM                    [ALARMS: 0]  14:30         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ZONE 1           ZONE 2          ZONE 3         OUTPUT        │
│  [■ RUNNING]      [■ RUNNING]     [■ RUNNING]    [■ OPEN]     │
│                                                                 │
│  ╔════════════════════════════════════════════════════════╗    │
│  ║  →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→  ║    │
│  ║  ████████████████████████████████████████████████████ ║    │
│  ║  →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→  ║    │
│  ╚════════════════════════════════════════════════════════╝    │
│                                                                 │
│  Speed: [  15.5  ] m/min     Count: [  1234  ] pcs            │
│                                                                 │
│  [START ALL]  [STOP ALL]  [EMERGENCY STOP]                     │
│                                                                 │
│  Motor Currents:  C1:[  5.2A ]  C2:[  7.8A ]  C3:[  4.1A ]   │
├─────────────────────────────────────────────────────────────────┤
│  [MAIN]  [DETAIL]  [ALARMS]  [TRENDS]  [SETTINGS]             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 278: Trend View

### Trend View คืออะไร

Trend View แสดงค่าตัวแปรตามเวลา เหมือน Graph บนกระดาษกราฟ ช่วยให้วิศวกรวิเคราะห์พฤติกรรมของระบบ

```
Trend View Screen Layout:

┌─────────────────────────────────────────────────────────────────┐
│  TREND VIEW - Conveyor Speeds                [Last 1 Hour]     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  50┤────────────────────────────────────────────────────────   │
│    │        ╭──────╮          ╭────────────────────────────    │
│  40┤       /        \        /                                 │
│    │      /          \      /                    — C1 Speed   │
│  30┤─────/────────────\────/─────────────────────── C2 Speed  │
│    │                   \  /                  ─ ─  C3 Speed    │
│  20┤                    \/                                     │
│    │                                                           │
│  10┤                                                           │
│    │                                                           │
│   0┤───────────────────────────────────────────────────────   │
│    └──────────────────────────────────────────────────────►   │
│    13:00    13:15    13:30    13:45    14:00    14:15    14:30  │
│                                                                 │
│  [▶ LIVE]  [⏮ HISTORY]  [ZOOM IN]  [ZOOM OUT]  [EXPORT]     │
├─────────────────────────────────────────────────────────────────┤
│  [MAIN]  [DETAIL]  [ALARMS]  [TRENDS]  [SETTINGS]             │
└─────────────────────────────────────────────────────────────────┘
```

### Trend Configuration ใน TIA Portal (WinCC)

```
Trend Settings:
  Tag: HMI_Conv1_Speed (Real)
  Archive Period: 1 second
  History Duration: 30 days
  
  Display:
  Time Range: 1 hour
  Y-Axis Min: 0.0
  Y-Axis Max: 60.0
  Line Color: Green
  Line Width: 2px
  
Multiple Trends:
  Trend 1: Conv1_Speed (Green, Solid)
  Trend 2: Conv2_Speed (Blue, Solid)
  Trend 3: Conv3_Speed (Red, Dashed)
  Trend 4: Motor1_Current (Orange, Dotted)
```

### Historical Data Access

```
// WinCC Script: Export Trend Data

Sub ExportTrendData()
    Dim oArchive As Object
    Dim oData As Object
    
    ' เปิด Archive
    oArchive = HMIRuntime.DataLogs("ConveyorLog")
    
    ' ดึงข้อมูล 1 ชั่วโมงย้อนหลัง
    Dim tStart As Date
    Dim tEnd As Date
    tEnd = Now()
    tStart = DateAdd("h", -1, tEnd)
    
    ' Query ข้อมูล
    oData = oArchive.Query(tStart, tEnd)
    
    ' Export เป็น CSV
    ExportToCSV oData, "C:\Reports\Conveyor_" & Format(Now(), "YYYYMMDD") & ".csv"
    
    MsgBox "Export Complete!"
End Sub
```

---

## Step 279: Alarm List

### ระบบ Alarm ที่ดี

```
Alarm Classification (ISA-18.2):

Priority 1 (Critical): ต้องตอบสนองใน 1 นาที
  - Emergency Stop
  - Equipment Damage Imminent
  
Priority 2 (High): ตอบสนองใน 10 นาที
  - Process Deviation > 10%
  - Safety System Issue

Priority 3 (Medium): ตอบสนองใน 1 ชั่วโมง
  - Process Warning
  - Performance Degradation

Priority 4 (Low): ตอบสนองใน Shift
  - Informational
  - Maintenance Due
```

### Alarm List Screen

```
Alarm List Screen:

┌─────────────────────────────────────────────────────────────────┐
│  ALARM LIST                         Active: 2   Total: 15      │
├──────┬──────────────┬──────────┬────────────────┬──────────────┤
│ Prio │ Time         │ Tag      │ Message        │ Status       │
├──────┼──────────────┼──────────┼────────────────┼──────────────┤
│  ▲1  │ 14:25:33     │ ESTOP_1  │ E-STOP ถูกกด  │ ACTIVE       │
│  ▲2  │ 14:20:15     │ OL_M1   │ Motor 1 OL Trip│ ACTIVE       │
│  ▼2  │ 14:15:00     │ TEMP_HI  │ Motor Temp High│ CLEARED      │
│  ▼3  │ 13:55:22     │ JAM_C1   │ Conveyor 1 Jam │ ACKED        │
│  ▼3  │ 13:40:11     │ SPEED_LO │ Speed Below SP │ CLEARED      │
│      │              │          │                │              │
├──────┴──────────────┴──────────┴────────────────┴──────────────┤
│                                                                 │
│  [ACKNOWLEDGE SELECTED]  [ACKNOWLEDGE ALL]  [CLEAR CLEARED]    │
│                                                                 │
│  Filter: [ALL ▼]  Priority: [ALL ▼]  [EXPORT TO USB]          │
├─────────────────────────────────────────────────────────────────┤
│  [MAIN]  [TRENDS]  [ALARMS]  [SETTINGS]                       │
└─────────────────────────────────────────────────────────────────┘
```

### Alarm Configuration ใน TIA Portal

```
Alarm Setup:
1. Discrete Alarm (Bit Alarm):
   Tag: OL_Fault
   Trigger: Bit = 1
   Priority: 2 (High)
   Message: "Motor 1 Overload Trip"
   Acknowledgment: Required
   
2. Analog Alarm (Value Alarm):
   Tag: Motor_Temp
   Trigger: Value > 90°C
   Priority: 2
   Message: "Motor Temperature High: %v°C"
   Hysteresis: 5°C (ป้องกัน Chattering)
```

---

## Step 280: Recipe Screen Design

### Recipe System คืออะไร

Recipe System ช่วยให้สามารถเปลี่ยน Setpoint หลายตัวพร้อมกัน โดยโหลดจาก "สูตร" ที่บันทึกไว้

```
Recipe Concept:

Recipe "Product A":
  Speed: 25.0 m/min
  Temperature: 85°C
  Pressure: 3.5 Bar
  Timeout: 120s
  Count_Target: 500

Recipe "Product B":
  Speed: 15.0 m/min
  Temperature: 75°C
  Pressure: 2.5 Bar
  Timeout: 180s
  Count_Target: 250

→ กด "Load Recipe A" → ค่าทั้งหมดถูกส่งไป PLC ทันที
→ ไม่ต้องตั้งค่าทีละตัว ลดความผิดพลาด
```

### Recipe Screen Layout

```
Recipe Management Screen:

┌─────────────────────────────────────────────────────────────────┐
│  RECIPE MANAGEMENT              [Active Recipe: Product A]      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Available Recipes:             Recipe Details:                │
│  ┌──────────────────────┐       ┌─────────────────────────┐   │
│  │ ▶ Product A          │       │ Recipe: Product A        │   │
│  │   Product B          │       │                          │   │
│  │   Product C (Custom) │       │ Speed SP:  [  25.0  ]   │   │
│  │   Test Recipe        │       │ Temp SP:   [  85.0  ]   │   │
│  │   ...                │       │ Press SP:  [   3.5  ]   │   │
│  └──────────────────────┘       │ Timeout:   [  120   ]   │   │
│                                  │ Count:     [   500  ]   │   │
│  [LOAD SELECTED]                 │                          │   │
│  [SAVE CURRENT AS...]            └─────────────────────────┘   │
│  [DELETE]                                                       │
│  [IMPORT FROM USB]               [APPLY TO PLC]                │
│  [EXPORT TO USB]                 [SAVE RECIPE]                  │
├─────────────────────────────────────────────────────────────────┤
│  [MAIN]  [RECIPE]  [TRENDS]  [ALARMS]  [SETTINGS]             │
└─────────────────────────────────────────────────────────────────┘
```

### Recipe Configuration ใน TIA Portal

```
Recipe Definition:

Recipe Name: Product_A
Elements:
  - Conv_Speed_SP:  Real, Default=25.0, Range: 0-60
  - Temp_Setpoint:  Real, Default=85.0, Range: 50-150
  - Press_SP:       Real, Default=3.5,  Range: 0-10
  - Timeout_S:      Int,  Default=120,  Range: 10-600
  - Count_Target:   DInt, Default=500,  Range: 1-9999

PLC Tag Mapping:
  Conv_Speed_SP  → %DB_Process.Speed_SP
  Temp_Setpoint  → %DB_Process.Temp_SP
  Press_SP       → %DB_Process.Press_SP
  Timeout_S      → %MW50
  Count_Target   → %MD52
```

---

## Step 281: User Management / Login

### Access Levels

```
User Levels:

Level 1 (Operator):
  - ดูสถานะ
  - Start/Stop ระบบ
  - Acknowledge Alarm
  - Load Recipe

Level 2 (Senior Operator):
  - Level 1 + เปลี่ยน Setpoint
  - ใช้งาน Manual Mode
  - ใช้ Maintenance Mode (Jog)

Level 3 (Supervisor):
  - Level 2 + ดู Trend History
  - Export Data
  - Change Alarm Limits

Level 4 (Engineer):
  - Level 3 + เปลี่ยน Recipe
  - PID Tuning
  - Calibration

Level 5 (Admin):
  - ทุกอย่าง
  - Manage Users
  - System Settings
```

### Login Screen

```
Login Screen:

┌─────────────────────────────────────────────────────────────────┐
│  USER LOGIN                                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│              ┌──────────────────────────────────┐              │
│              │         FACTORY HMI               │              │
│              │                                   │              │
│              │   Username: [              ]      │              │
│              │   Password: [**********    ]      │              │
│              │                                   │              │
│              │   [    LOG IN    ]                │              │
│              │                                   │              │
│              │   Current User: Operator (Level1)│              │
│              │   Session Time: 02:15:33         │              │
│              └──────────────────────────────────┘              │
│                                                                 │
│              [LOGOUT]                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 282: HMI Navigation Design

### Navigation Map

```
Screen Navigation Map:

                      ┌─────────────────┐
                      │   Splash/Home   │
                      └────────┬────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
   ┌────────────┐      ┌────────────┐      ┌────────────┐
   │  Overview  │      │  Process   │      │   Alarm    │
   │  Screen    │      │  Screen    │      │   Screen   │
   └─────┬──────┘      └─────┬──────┘      └────────────┘
         │                   │
    ┌────┘                   └────┐
    │                             │
    ▼                             ▼
┌─────────┐                 ┌─────────┐
│  Detail │                 │  PID    │
│  Screen │                 │  Tuning │
└─────────┘                 └─────────┘

          ┌────────────────────────────────────┐
          │  Accessible from All Screens:       │
          │  - Alarm List                       │
          │  - Trends                           │
          │  - Login/Logout                     │
          └────────────────────────────────────┘
```

---

## Step 283: HMI Programming สำหรับ Siemens

### Siemens WinCC Flexible / TIA Portal WinCC

```
Screen Types ใน TIA Portal:
1. Root Screen (หน้าหลัก)
2. Normal Screen (หน้าทั่วไป)
3. Popup Screen (หน้าต่างลอย)
4. Permanent Window (แถบถาวร เช่น Navigation)
5. System Screen (Login, Alarm, etc.)
```

### Script Example (WinCC)

```vbscript
' Screen: ConveyorOverview
' Button: Speed_Plus_Button
' Event: Click - เพิ่ม Speed 5%

Sub Speed_Plus_Click()
    Dim currentSpeed As Single
    Dim newSpeed As Single
    
    ' อ่านค่าปัจจุบัน
    currentSpeed = SmartTags("HMI_Conv1_Speed_SP")
    
    ' เพิ่ม 5%
    newSpeed = currentSpeed + 5.0
    
    ' จำกัดค่าสูงสุด
    If newSpeed > 100.0 Then
        newSpeed = 100.0
    End If
    
    ' เขียนค่าใหม่
    SmartTags("HMI_Conv1_Speed_SP") = newSpeed
    
    ' แสดงข้อความใน Status Bar
    SmartTags("HMI_Status_Message") = "Speed increased to " & newSpeed & "%"
End Sub
```

---

## Step 284: HMI Programming สำหรับ Mitsubishi GOT

### GOT2000 Programming (GT Designer3)

```
GOT2000 Screen Design:

1. สร้าง Project ใหม่
   New → Project → GOT2000 (GT27xx)
   Controller: MELSEC iQ-R
   Connection: Ethernet (192.168.1.3)

2. สร้าง Tag (Device in Mitsubishi terms)
   Device: M0   = Conv1_Run (Bit)
   Device: D0   = Conv1_Speed_SP (Word)
   Device: D2   = Motor_Temperature (Word, Scale: ×0.1)
   Device: D100 = Product_Count (Double Word)

3. ออกแบบ Screen
   - Bit Lamp: D → Display ON/OFF ตาม M0
   - Numerical Display: D → แสดงค่า D0 × 0.1 m/min
   - Trend Graph: D0, D2 ตามเวลา

4. Script (VBA-like)
   
   Sub M_STARTBTN_Click()
     SetDevice("M10", 1)    ' กด Start
     Wait(200)              ' รอ 200ms
     SetDevice("M10", 0)    ' ปล่อย
   End Sub
```

---

## Step 285: HMI Programming สำหรับ Allen-Bradley

### PanelView Plus 7 (FactoryTalk View ME)

```
FactoryTalk View ME Setup:

1. Create Application
   FactoryTalk View Studio ME
   Display Size: 1024×768
   Color: 65535 colors

2. Create Tag Database
   [HMI]Conv1_Run        : Digital
   [HMI]Conv1_Speed_SP   : Analog
   [AB_PLC]Motor_Status  : Direct PLC Tag
   
   PLC Connection: EtherNet/IP
   Controller: CompactLogix L33ER
   IP: 192.168.1.10

3. Link Display to PLC Tags
   Bit Button → Write Tag: [AB_PLC]Start_Cmd
   Numeric Display → Read Tag: [AB_PLC]Motor_Speed_FB

4. Alarm Setup
   Alarm Tag: [AB_PLC]Motor_OL_Trip
   Trigger: 1 (Bit ON)
   Message: "Motor 1 Overload"
   ACK Required: Yes
```

---

## Step 286: HMI Data Logging

### Historical Data Storage

```
Data Logging Options:

1. HMI Internal Memory
   - Flash Memory onboard HMI
   - Capacity: 32MB - 2GB
   - Auto-overwrite oldest data

2. USB Flash Drive
   - ใส่ USB ที่ HMI
   - Manual export
   - Good for portable data

3. SD Card Slot (บาง HMI)
   - ความจุสูง
   - Easy replacement

4. Network Drive (Ethernet)
   - บันทึกไปยัง Server
   - Real-time backup
   - Shared with other systems

5. Historian Server (Advanced)
   - Wonderware, OSIsoft PI
   - High-capacity
   - Analytics ready
```

### Data Log Configuration

```
Siemens WinCC: Data Logging Configuration

Log Name: ConveyorLog
Archive Type: Process Values
Sample Rate: 1 second

Logged Tags:
  Conv1_Speed    : Real (m/min)
  Conv2_Speed    : Real
  Conv3_Speed    : Real
  Motor1_Current : Real (A)
  Motor2_Current : Real
  Product_Count  : DInt
  Temperature    : Real (°C)
  Pressure       : Real (Bar)

Storage:
  Location: USB_Drive/DataLogs/
  File Format: CSV
  File Rotation: Daily (new file each day)
  Max Files: 90 (90 วัน)
  
Export Format:
  DateTime, Conv1_Speed, Conv2_Speed, Conv3_Speed, ...
  2026-10-01 14:30:01, 25.5, 25.8, 24.9, ...
```

---

## Step 287: Remote Monitoring

### Web-Based HMI Monitoring

```
Remote Access Architecture:

Factory Network (LAN)                 Remote Access
┌─────────────────────────┐               │
│  PLC ←→ HMI             │               │
│           │             │          ┌────┴────┐
│           │ OPC-UA      │          │  VPN    │
│           │             │          │ Server  │
│      Web Server         │←─────────┤         │
│      (Port 80/443)      │          └────┬────┘
│                         │               │
└─────────────────────────┘          Remote PC
                                      / Phone
                                    / Tablet

Access: https://192.168.1.100
Login: Username + Password
View: Real-time Data, Alarms, Trends
Control: (Optional, with extra security)
```

### Siemens S7-1200 Web Server Setup

```
TIA Portal: Web Server Configuration

CPU Properties → Web Server
  Enable Web Server: ✓
  Permit Access: All Users
  Languages: English, Thai

Create Web Pages:
  Default Pages: Auto-generated from program
  Custom Pages: HTML files in CPU

User Management:
  User: operator, Password: xxxx, Role: Read Only
  User: engineer, Password: yyyy, Role: Read/Write
  User: admin,    Password: zzzz, Role: Admin

Access via Browser:
  URL: http://192.168.1.1/
  Username: operator
  Password: ****
  
Available Pages:
  - Process Overview (Auto)
  - Variable Status
  - Data Logs Download
  - Diagnostic Info
```

---

## Step 288: Mobile HMI App

### Siemens SIMATIC WinCC Webnavigator

```
Mobile Access Setup:

1. WinCC Server ติดตั้ง WebNavigator
2. Configure Web Server Port: 80 หรือ 443 (HTTPS)
3. User Access Rights กำหนดสิทธิ์
4. Client Access ด้วย Browser หรือ App

Compatible Browsers:
  - Chrome, Firefox, Edge, Safari
  - iOS Safari
  - Android Chrome

Screen Resolution Adaptation:
  Desktop: Full Screen
  Tablet: Scaled
  Phone: Scrollable or Adapted Layout
```

### SCADA Integration

```
SCADA System Integration:

Ignition (Inductive Automation):
  - Driver: Siemens S7, AB ControlLogix, Mitsubishi MC
  - Protocol: OPC-UA, Modbus TCP, EtherNet/IP
  - Mobile: Perspective Module (HTML5)
  - Reporting: Integrated Report Designer

FactoryTalk (Rockwell):
  - View ME/SE
  - Historian
  - Metrics (OEE)
  - Remote Access Module

WinCC (Siemens):
  - WinCC Classic
  - WinCC Comfort (HMI)
  - WinCC Advanced
  - WinCC Professional (Full SCADA)
  
Wonderware (AVEVA):
  - InTouch
  - System Platform
  - Historian
```

---

## Step 289: HMI Checklist and Best Practices

### HMI Design Best Practices

```
✓ Consistency
  - ใช้ Color Convention เดียวกันตลอด
  - Navigation อยู่ที่เดิมทุกหน้า
  - Font Size อ่านง่าย (≥ 14pt สำหรับ Label, ≥ 18pt สำหรับ Data)

✓ Simplicity
  - 1 Screen = 1 ฟังก์ชัน
  - ไม่ยัดข้อมูลมากเกินไป
  - ใช้ Popup สำหรับ Detail

✓ Safety
  - Confirm Dialog สำหรับ Critical Action
  - Login Required สำหรับ Setting Changes
  - E-Stop ชัดเจน สีแดง ขนาดใหญ่

✓ Readability
  - Contrast สูง (Black on White หรือ White on Dark)
  - ป้ายชื่อชัดเจน
  - หน่วย ระบุเสมอ (m/min, °C, Bar)

✓ Performance
  - Refresh Rate เหมาะสม (1-2 sec สำหรับ non-critical)
  - ไม่ Animate มากเกินไป (ทำให้ช้า)
  - Archive ที่จำเป็นเท่านั้น
```

### HMI Commissioning Checklist

```
Before Going Live:

☐ ทุก Tag สื่อสารกับ PLC ได้ถูกต้อง
☐ Button ทุกปุ่มทดสอบแล้ว
☐ Alarm ทุกรายการตรวจสอบแล้ว
☐ Trend แสดงข้อมูลถูกต้อง
☐ Recipe Load/Save ทำงาน
☐ User Login ทุก Level ทดสอบแล้ว
☐ Data Log บันทึกข้อมูล
☐ Remote Access ทดสอบ (ถ้ามี)
☐ Screen Navigation ทดสอบทุกเส้นทาง
☐ Emergency Stop แสดงบนหน้าจอ
☐ Operator Training Completed
☐ HMI Backup ทำแล้ว
☐ Documentation Updated
```

---

## Step 290: Lab Project - Complete HMI Design

### Lab 15: ออกแบบ HMI สำหรับ Conveyor System

**โจทย์:**
ออกแบบและทดสอบ HMI สำหรับระบบ Conveyor 3 เส้น (ต่อเนื่องจาก Lab 14)

**HMI Specification:**
```
Hardware: Siemens KTP700 Basic (7", Touch)
Resolution: 800×480
Software: TIA Portal V17 (WinCC Basic)
Connection: PROFINET (192.168.1.2)
PLC: S7-1200 (192.168.1.1)
```

**Required Screens:**

**Screen 1: Main Overview**
```
┌─────────────────────────────────────────────────────────────────┐
│  CONVEYOR SYSTEM              ALARM: 0        14:30 01/10/2026 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CONV 1          CONV 2           CONV 3                      │
│   ● RUNNING       ● RUNNING        ● RUNNING                   │
│   15.5 m/min      16.0 m/min       15.8 m/min                  │
│                                                                 │
│  ══════════════════════════════════════════════════════════     │
│  →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→             │
│  ══════════════════════════════════════════════════════════     │
│                                                                 │
│  Count: 12,345 pcs              Weight: 5,432 kg               │
│                                                                 │
│        [▶ START ALL]    [■ STOP ALL]    [⚡ E-STOP]           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  [HOME]  [DETAIL]  [ALARMS]  [TRENDS]  [RECIPE]  [SETTINGS]   │
└─────────────────────────────────────────────────────────────────┘
```

**Screen 2: Conveyor Detail**
```
┌─────────────────────────────────────────────────────────────────┐
│  CONVEYOR 1 DETAIL                               ← BACK        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Status:  ● RUNNING             Motor Current: 5.2 A           │
│  Speed:   [  15.5  ] m/min      Motor Temp:    78 °C           │
│                                                                 │
│  Speed Setpoint:   [_________] m/min  [SET]                    │
│  Min: 5.0    Max: 30.0                                         │
│                                                                 │
│  [ JOGI+ ] [ JOG- ] (Maintenance Mode Only)                   │
│                                                                 │
│  Run Hours: 1,234.5 hrs    [RESET PM COUNTER]                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Speed Trend (Last 30 min)                              │  │
│  │  ──────────────────────────────────────────────────     │  │
│  │  30│─────────────────────────────────────────────       │  │
│  │  20│                                                     │  │
│  │  10│                                                     │  │
│  │   0│                                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  [HOME]  [DETAIL]  [ALARMS]  [TRENDS]  [RECIPE]  [SETTINGS]   │
└─────────────────────────────────────────────────────────────────┘
```

**Screen 3: Alarm Screen (as per Step 279)**

**Screen 4: Recipe Screen (as per Step 280)**

**Tag List สำหรับ HMI:**
```
Tag Name                PLC Address     Type   R/W
────────────────────────────────────────────────────
Conv1_Run_Status       %Q0.0           Bool   R
Conv2_Run_Status       %Q0.1           Bool   R
Conv3_Run_Status       %Q0.2           Bool   R
Safety_Status          %M0.0           Bool   R
Seq_Start_CMD          %M1.1           Bool   R/W
Seq_Stop_CMD           %M1.2           Bool   R/W
Conv1_Speed_SP         %DB1.DBD0       Real   R/W
Conv2_Speed_SP         %DB1.DBD4       Real   R/W
Conv3_Speed_SP         %DB1.DBD8       Real   R/W
Conv1_Speed_FB         %DB1.DBD12      Real   R
Conv2_Speed_FB         %DB1.DBD16      Real   R
Conv3_Speed_FB         %DB1.DBD20      Real   R
Motor1_Current         %DB1.DBD24      Real   R
Motor1_Temp            %DB1.DBD28      Real   R
Product_Count          %DB1.DBD32      DInt   R
Product_Weight_kg      %DB1.DBD36      Real   R
Fault_Code             %MW100          Int    R
Fault_Reset            %M10.0          Bool   W
Run_Hours_C1           %DB1.DBD40      DInt   R
PM_Reset_CMD           %M11.0          Bool   W
Alarm_ESTOP            %M20.0          Bool   R
Alarm_OL_C1            %M20.1          Bool   R
Alarm_Jam_C1           %M20.2          Bool   R
────────────────────────────────────────────────────
```

**Alarm Configuration:**
```
Alarm Name    Tag           Trigger  Priority  Message
───────────────────────────────────────────────────────
ESTOP_ALARM   Alarm_ESTOP   =1       1 (Crit)  "Emergency Stop Active"
OL_C1_ALARM   Alarm_OL_C1   =1       2 (High)  "Conveyor 1 Overload"
JAM_C1_ALARM  Alarm_Jam_C1  =1       2 (High)  "Conveyor 1 Jam"
TEMP_WARN     Motor1_Temp   >85.0    3 (Med)   "Motor 1 Temp High"
SPEED_LOW     Conv1_Speed   <5.0     4 (Low)   "Speed Below Minimum"
```

**Validation Checklist:**
```
☐ Overview Screen แสดงสถานะทุก Conveyor
☐ Start/Stop Button ทำงานและส่งคำสั่งไป PLC
☐ Speed Setpoint ป้อนค่าได้ และ PLC รับค่า
☐ Speed Feedback แสดงถูกต้อง
☐ Alarm ทุกรายการ Trigger ถูกต้อง
☐ Alarm Acknowledge ทำงาน
☐ Recipe Load/Save ทำงาน
☐ Trend แสดงข้อมูล Real-time และ History
☐ Login Level 1/2/3 จำกัดสิทธิ์ถูกต้อง
☐ Navigation ไปมาระหว่าง Screen ได้
☐ เวลาแสดงถูกต้อง
☐ Screen ดูดีและอ่านง่ายใน 7" Display
```

---

## สรุป Part 15

ในบทนี้เราได้เรียนรู้:
1. ประเภทของ HMI ตั้งแต่ Operator Panel ถึง Panel PC
2. HMI-PLC Communication Protocols
3. Tag Mapping ระหว่าง HMI และ PLC
4. หลักการออกแบบหน้าจอ (Color Convention, Layout)
5. Button และ Indicator Types
6. Numeric Display และ Process Graphics
7. Trend View สำหรับวิเคราะห์ข้อมูล
8. Alarm List ตาม ISA-18.2
9. Recipe System สำหรับเปลี่ยน Product
10. User Management และ Access Levels
11. Data Logging และ Historical Data
12. Remote Monitoring ผ่าน Web Browser
13. Mobile HMI Access
14. Best Practices และ Commissioning Checklist

---

## การนำทาง

- [← Part 14: ระบบ Conveyor พื้นฐาน](part14_conveyor_system.md)
- [→ Part 16: Communication Networks](part16_communication.md)
- [กลับหน้าหลัก](../README.md)

---

*Part 15 จบ | ต่อไป Part 16: Communication Networks*

---

## Appendix: HMI Quick Reference

### Color Standards สรุป

```
มาตรฐาน ISA-101 (ANSI/ISA-101.01-2015):

Background:     Gray (#C0C0C0) หรือ Dark Gray
Normal:         Green (#00AA00)
Warning:        Yellow (#FFFF00)
Alarm:          Red (#AA0000) หรือ Orange (#FF6600)
Highlighted:    Bright (ถ้าปกติเป็น Gray ใช้ White เพื่อ Highlight)
Disabled:       Dimmed (50% ความสว่าง)
```

### Common HMI Screen List

```
Screens ที่ควรมีในระบบ HMI ทั่วไป:

1. Home / Overview Screen
2. Process Detail Screen (ต่อ 1 Process Area)
3. Trend Screen
4. Alarm Screen
5. Recipe Screen (ถ้ามีหลาย Product)
6. Settings Screen (PID Tuning, Limits)
7. Maintenance Screen (Manual Mode, Jog)
8. Login Screen
9. Reports Screen
10. Help/Documentation Screen
```

### Address Quick Reference

```
Siemens S7-1200 HMI Tags:
Bool: %M0.0, %Q0.0, %I0.0, %DB1.DBX0.0
Byte: %MB0, %QB0, %IB0
Int:  %MW0, %QW0, %IW0, %DB1.DBW0
DInt: %MD0, %QD0, %ID0, %DB1.DBD0
Real: %MD0, %DB1.DBD0

Allen-Bradley ControlLogix HMI:
All Tag-based, no physical address
Bool: Tag_Name (BOOL)
Int:  Tag_Name (INT/DINT)
Real: Tag_Name (REAL)

Mitsubishi FX3U HMI:
Bit:  M0, Y0, X0, SM0
Word: D0, R0, Z0
DWord: D0-D1 (2 consecutive)
```
