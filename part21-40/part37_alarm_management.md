# Part 37: Alarm Management
## Steps 721-740: การจัดการ Alarm ตามมาตรฐาน ISA-18.2

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 3-4 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 36 (SCADA Integration)

---

## บทนำ

Alarm Management ที่ดีเป็นสิ่งจำเป็นอย่างยิ่งในอุตสาหกรรม เหตุการณ์ใหญ่หลายครั้ง เช่น Three Mile Island (1979) และ Texas City Refinery (2005) มีสาเหตุส่วนหนึ่งจากการจัดการ Alarm ที่ไม่ดี (Alarm Flood)

ISA-18.2 เป็นมาตรฐานสากลสำหรับ Alarm Management ใน Process Industry ที่ครอบคลุมตั้งแต่การออกแบบจนถึงการบำรุงรักษา

---

## Step 721: ISA-18.2 Alarm Standard

### หลักการพื้นฐาน ISA-18.2

```
┌────────────────────────────────────────────────────────────┐
│                  ISA-18.2 Key Principles                   │
│                                                            │
│  1. Alarm Definition                                       │
│     "Alarm คืออะไร" → ต้องเป็นสิ่งที่:                   │
│     ✓ ต้องการการตัดสินใจจากผู้ปฏิบัติงาน                  │
│     ✓ มีเวลาที่กำหนดในการตอบสนอง                         │
│     ✓ มีผลกระทบต่อความปลอดภัย/สิ่งแวดล้อม/การผลิต        │
│                                                            │
│  2. Alarm Rationalization                                  │
│     ทุก Alarm ต้องมี:                                     │
│     ✓ เหตุผลที่ต้องแจ้งเตือน                              │
│     ✓ ผลที่ตามมาถ้าไม่ทำ                                  │
│     ✓ ขั้นตอนที่ต้องทำ (Corrective Action)                │
│     ✓ เวลาตอบสนองที่กำหนด                                 │
│                                                            │
│  3. Alarm Performance Targets                              │
│     Average Alarm Rate: < 1 alarm/10 min/operator          │
│     Peak Alarm Rate: < 10 alarms/10 min                   │
│     Flood: < 10 alarms/10 min for > 10 min                │
└────────────────────────────────────────────────────────────┘
```

### Alarm Lifecycle

```
Design → Rationalization → HMI Configuration → Operation → 
  Monitoring → Maintenance → Modification → MOC Review
```

---

## Step 722: Alarm Types

### Process Alarms

```
Process Alarms - เกี่ยวกับ Process Variables:
┌──────────────────────────────────────────────────────────┐
│  Alarm Type     │ Description           │ Example        │
├──────────────────────────────────────────────────────────┤
│  Hi/Lo Limit    │ ค่าเกินขีดจำกัด      │ T > 90°C       │
│  HiHi/LoLo      │ ค่าเกินขีดวิกฤต     │ T > 100°C      │
│  Rate of Change │ เปลี่ยนแปลงเร็วเกิน  │ dT/dt > 5°/min │
│  Deviation      │ ต่างจาก SP มากเกิน  │ |PV-SP| > 10°C │
│  Return to Normal│ กลับสู่สภาวะปกติ    │ T < 85°C       │
└──────────────────────────────────────────────────────────┘
```

### Equipment Alarms

```
Equipment Alarms - เกี่ยวกับสถานะอุปกรณ์:
┌──────────────────────────────────────────────────────────┐
│  Motor Running Fail   │ สั่ง Run แต่ไม่ Run             │
│  Motor Trip           │ Motor Thermal Trip               │
│  Valve Position Error │ สั่งเปิดแต่ Position ผิด         │
│  Drive Fault          │ VFD/Servo Error                  │
│  Communication Fail   │ PLC-Device ขาดการติดต่อ          │
└──────────────────────────────────────────────────────────┘
```

### Diagnostic Alarms

```
Diagnostic Alarms - เกี่ยวกับระบบ:
┌──────────────────────────────────────────────────────────┐
│  Sensor Fail (Input = 0/max)  │ Wire Break Detection     │
│  Sensor High/Low              │ Out of Range             │
│  PLC Battery Low              │ Backup Battery Warning   │
│  Communication Timeout        │ Network Error            │
│  I/O Module Fault             │ Hardware Error           │
└──────────────────────────────────────────────────────────┘
```

---

## Step 723: Alarm Priority

### ISA-18.2 Priority Levels

```
Priority Levels:
┌───────────────────────────────────────────────────────────┐
│ Priority │ Color  │ Time to Respond │ Description         │
├───────────────────────────────────────────────────────────┤
│ Critical │  Red   │ < 5 minutes     │ Immediate action    │
│          │        │                 │ Safety/Environment  │
├───────────────────────────────────────────────────────────┤
│ High     │ Orange │ < 15 minutes    │ Significant impact  │
│          │        │                 │ Major production    │
├───────────────────────────────────────────────────────────┤
│ Medium   │ Yellow │ < 30 minutes    │ Moderate impact     │
│          │        │                 │ Monitor closely     │
├───────────────────────────────────────────────────────────┤
│ Low      │ Blue   │ < 60 minutes    │ Minor impact        │
│          │        │                 │ Informational       │
└───────────────────────────────────────────────────────────┘
```

### Priority Distribution Target

```
Good Alarm System Distribution:
                    
Priority │ Target % │ ใช้สำหรับ
---------|----------|----------------------------------
Critical │  < 5%   │ Safety, Catastrophic Equipment
High     │  < 15%  │ Major Process Upset
Medium   │  < 30%  │ Significant Deviation
Low      │  > 50%  │ Advisory, Informational

ถ้า Critical > 10% → System ไม่ได้รับการ Rationalize
ถ้า High > 25%     → ผู้ปฏิบัติงานจะ Ignore Alarms
```

---

## Step 724: Alarm Rationalization

### Master Alarm Database (MAD)

```
ทุก Alarm ต้องมีข้อมูลใน Master Alarm Database:

┌───────────────────────────────────────────────────────────────┐
│                  Alarm Record Template                         │
├──────────────────────────────────────────────────────────────-┤
│ Tag Name      : TH01_U01_TMP001_HI                           │
│ Description   : อุณหภูมิโซน 1 สูงเกินกำหนด                   │
│ Equipment     : Reactor R-001                                 │
│ P&ID Number   : P-TH01-001                                   │
│                                                               │
│ Alarm Setpoint: 90°C (Hi) / 100°C (HiHi)                    │
│ Normal Range  : 70-85°C                                       │
│                                                               │
│ Cause:        │ หม้อน้ำเกิดขัดข้อง หรือ โหลดเพิ่มขึ้น         │
│ Consequence:  │ ผลิตภัณฑ์คุณภาพต่ำ / อาจเกิด Runaway         │
│ Corrective:   │ 1. ลด Feed Rate                               │
│               │ 2. เพิ่ม Cooling Water Flow                   │
│               │ 3. ถ้า > 95°C: Shutdown Production            │
│               │ 4. แจ้ง Supervisor                            │
│                                                               │
│ Priority      : High                                          │
│ Response Time : 15 minutes                                   │
│ Rationalized By: Engineer Smith                              │
│ Review Date   : 2024-01-15                                   │
└───────────────────────────────────────────────────────────────┘
```

---

## Step 725: Alarm Flood Management

### Alarm Flood คืออะไร

```
Alarm Flood = จำนวน Alarm มากเกินความสามารถในการตอบสนอง

ISA-18.2 นิยาม Flood:
- > 10 alarms/10 minutes per operator
- เป็นเวลา > 10 นาที ต่อเนื่อง

ปัญหาที่เกิดขึ้น:
✗ ผู้ปฏิบัติงานไม่สามารถรับมือได้
✗ ข้าม Alarm สำคัญ
✗ Acknowledge ทุก Alarm โดยไม่ดู
✗ เกิด "Alert Fatigue"
```

### การแก้ไข Alarm Flood

```
เทคนิค 1: Alarm Shelving
- ชั่วคราวระงับ Alarm ที่รู้แล้ว
- กำหนดเวลา (เช่น 1 ชั่วโมง)
- Auto-restore เมื่อครบเวลา

เทคนิค 2: Alarm Suppression
- ระงับ Alarm ที่ไม่เกี่ยวข้องใน Mode นั้น
- เช่น: ระงับ Low Level Alarm เมื่ออยู่ใน Startup Mode
- ต้องมี Logic ชัดเจน

เทคนิค 3: State-based Alarming
- Alarm Setpoint เปลี่ยนตาม Operating State
- Startup Mode: Wider limits
- Normal Mode: Tight limits

เทคนิค 4: Dependent Alarm Suppression
- ถ้า Equipment A ดาวน์ → ระงับ Alarms ที่เกิดจาก A
- ลด Consequential Alarms

เทคนิค 5: First-out Logic
- แสดงเฉพาะ Alarm แรกที่เกิดใน Group
- Cascade Alarms ไม่แสดง
```

---

## Step 726: PLC Alarm Bit Management

### การจัดการ Alarm ใน PLC

```
// DB "Alarm_Management":
// 1 bit ต่อ 1 Alarm

Alarm_Register_1 : WORD  // 16 Alarms (Bit 0-15)
Alarm_Register_2 : WORD  // 16 Alarms (Bit 0-15)
...

// หรือใช้ ARRAY:
Alarm_Bits : ARRAY[1..200] OF BOOL
Alarm_Active    : ARRAY[1..200] OF BOOL
Alarm_Ack       : ARRAY[1..200] OF BOOL
Alarm_Timestamp : ARRAY[1..200] OF DATE_AND_TIME
Alarm_Count     : ARRAY[1..200] OF INT
```

### Alarm Logic - Set/Reset/Acknowledge

```
Network 1: ตรวจจับ Alarm (Temperature High)

  |                                                         |
  |--[>=]--[NOT]--------[SET]--[TON]----------------------|
  |  "Temp"  "Alarm001"  [S] "Alarm_DB"   T_Alarm001      |
  |  >=       .Alarm001       .Active[1]   PT: T#2s        |
  |  90.0    .Ack         (Set หลัง 2 วิ delay)             |
  |                                                         |

Network 2: บันทึกเวลา Alarm เกิด (Rising Edge)

  |                                                         |
  |--[P]----------[RD_SYS_T]-------------------------------|
  |  "Alarm_DB"    OUT: "Alarm_DB".Timestamp[1]            |
  |  .Active[1]                                             |
  |                                                         |
  |--[P]----------[INC]-------------------------------------|
  |  Active[1]     "Alarm_DB".Count[1]                     |
  |                                                         |

Network 3: Reset Alarm เมื่อสภาวะหาย

  |                                                         |
  |--[<]----------[RESET]----------------------------------|
  |  "Temp" <      "Alarm_DB".Active[1]                    |
  |  85.0          (ต้องต่ำกว่า 85°C จึง Reset - Hysteresis)|
  |                                                         |

Network 4: Acknowledge Alarm

  |                                                         |
  |--[ ]----------[SET]-------------------------------------|
  |  "HMI_Ack_     "Alarm_DB".Ack[1]                       |
  |   Alarm1"                                               |
  |                                                         |
  |--[NOT]--------[RESET]----------------------------------|
  |  Active[1]     Ack[1]  (Auto clear Ack เมื่อ Reset)    |
  |                                                         |
```

### Alarm Priority Logic

```
Network 5: สร้าง Priority Alarm Outputs

// Critical Alarms (ใดก็ได้ใน Group)
  |                                                         |
  |--[OR]-------------------------[SET]---------------------|
  |  Active[1]  (HiHi Temp)       "Critical_Alarm"          |
  |  Active[5]  (EStop)                                      |
  |  Active[9]  (Safety Relief)                              |
  |                                                         |

// High Alarms
  |--[OR]-------------------------[SET]---------------------|
  |  Active[2]  (Hi Temp)         "High_Alarm"               |
  |  Active[6]  (Hi Pressure)                                |
  |  Active[10] (Motor Fault)                                |
  |                                                         |
```

---

## Step 727: Alarm History Logging

### Alarm Log in PLC DB

```
// DB "Alarm_History":
Log_Count    : INT := 0
Log_Entries  : ARRAY[1..1000] OF STRUCT
  Event_Type   : INT   // 1=Alarm, 2=Ack, 3=Clear
  Alarm_Tag    : INT   // Alarm Number
  Priority     : INT   // 1=Critical, 2=High, 3=Medium, 4=Low
  Timestamp    : DATE_AND_TIME
  Value_At_Time: REAL  // ค่า Process ตอนที่เกิด
  Duration_s   : INT   // ระยะเวลา (วินาที) เมื่อ Clear
  Operator_ID  : INT   // ผู้ Acknowledge
  Comment      : STRING[50]
END_STRUCT

Write_Index : INT := 0  // ตำแหน่งเขียนปัจจุบัน
```

### Alarm Logging Ladder

```
// FC "Log_Alarm_Event" - Function สำหรับ Log

Network 1: เพิ่ม Write Index (Circular)

  |                                                         |
  |--[ALWAYS]------[ADD]-----------------------------------|
  |                 IN1: "Alarm_History".Write_Index        |
  |                 IN2: 1                                  |
  |                 OUT: "Temp_WIndex"                      |
  |                                                         |

Network 2: Wrap Around

  |                                                         |
  |--[>]----------[MOVE]-----------------------------------|
  |  Temp_WIndex > 1000  IN: 1 (กลับต้น)                  |
  |                       OUT: "Alarm_History".Write_Index  |
  |                                                         |
  |--[<=]---------[MOVE]-----------------------------------|
  |  Temp_WIndex <= 1000 IN: "Temp_WIndex"                 |
  |                       OUT: "Alarm_History".Write_Index  |
  |                                                         |

Network 3: บันทึก Event

  |                                                         |
  |--[ALWAYS]------[MOVE]----------------------------------|
  |  (Event Type = 1: Alarm)                               |
  |                 IN: 1                                  |
  |                 OUT: History.Log_Entries               |
  |                     [Write_Index].Event_Type           |
  |                                                         |
  |--[ALWAYS]------[RD_SYS_T]------------------------------|
  |                 OUT: History.Log_Entries               |
  |                     [Write_Index].Timestamp            |
  |                                                         |
  |--[ALWAYS]------[INC]-------------------------------------|
  |                 "Alarm_History".Log_Count              |
  |                 (ถ้า < 1000)                            |
  |                                                         |
```

---

## Step 728: Alarm KPIs

### Key Performance Indicators

```
Alarm KPIs ที่ ISA-18.2 แนะนำ:

1. Average Alarm Rate (AAR)
   = จำนวน Alarm ต่อ 10 นาที ต่อ Operator
   Target: < 1 alarm/10min
   
2. Alarm Flood Duration  
   = เวลาที่อยู่ใน Flood State (%)
   Target: < 1% of operating time
   
3. Stale Alarm Duration
   = Alarm ที่ค้างอยู่ > 24 ชั่วโมง
   Target: < 5 alarms pending > 24h
   
4. Chattering Alarm
   = Alarm ที่ On/Off > 3 ครั้งใน 10 นาที
   Target: 0 chattering alarms
   
5. Repeat Alarm (Bad Actor)
   = Top 10 Most Frequent Alarms
   Target: ควรแก้ไขรากของปัญหา

6. Acknowledged Rate
   = % Alarms ที่ได้รับการ Ack
   Target: > 90%

7. False Alarm Rate
   = Alarms ที่ไม่ต้องการ Action
   Target: < 5%
```

### KPI Calculation Program

```python
# Python Script (ใน Ignition หรือ External) สำหรับคำนวณ KPI

def calculateAlarmKPIs(startTime, endTime):
    
    # 1. Average Alarm Rate
    query_count = """
        SELECT COUNT(*) as alarm_count 
        FROM alarm_history
        WHERE event_time BETWEEN ? AND ?
          AND event_type = 'Active'
    """
    alarm_count = db.runScalarQuery(query_count, [startTime, endTime])
    duration_minutes = timeDifference(startTime, endTime, "minutes")
    aar = alarm_count / (duration_minutes / 10)
    
    # 2. Chattering Alarms
    query_chatter = """
        SELECT alarm_name, COUNT(*) as transitions
        FROM alarm_history
        WHERE event_time BETWEEN ? AND ?
        GROUP BY alarm_name, 
                 DATE_FORMAT(event_time, '%Y-%m-%d %H:%i') / 10
        HAVING transitions > 3
    """
    chattering = db.runQuery(query_chatter, [startTime, endTime])
    
    # 3. Top 10 Bad Actors
    query_bad = """
        SELECT alarm_name, COUNT(*) as count, 
               AVG(duration) as avg_duration
        FROM alarm_history
        WHERE event_time BETWEEN ? AND ?
        GROUP BY alarm_name
        ORDER BY count DESC
        LIMIT 10
    """
    bad_actors = db.runQuery(query_bad, [startTime, endTime])
    
    return {
        "AAR": aar,
        "AlarmCount": alarm_count,
        "ChatCount": len(chattering),
        "BadActors": bad_actors
    }
```

---

## Step 729: HMI Alarm Display

### Alarm Banner (Persistent)

```
HMI Alarm Display Best Practices:
┌─────────────────────────────────────────────────────────┐
│  [Critical: 3] [High: 7] [Medium: 12] [Low: 5]        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ CRITICAL │ 14:32:05 │ R-001 Temp HiHi > 100°C │  │
│  │ HIGH     │ 14:31:45 │ P-001 Motor Trip         │  │
│  │ HIGH     │ 14:30:12 │ P-002 Motor Fault        │  │
│  │ MEDIUM   │ 14:28:55 │ FI-101 Flow Low          │  │
│  └─────────────────────────────────────────────────┘  │
│  [ACK Selected] [ACK All] [Silence] [Details]         │
└─────────────────────────────────────────────────────────┘
```

### Alarm List Screen

```
Alarm List Display Fields:
┌────────────────────────────────────────────────────────────────┐
│ Priority│Time     │Duration│Tag Name    │Description  │Value│ACK│
├────────────────────────────────────────────────────────────────┤
│ CRIT    │14:32:05 │0:02    │R001_TMP_HH │R-001 T > 100│102.3│ N │
│ HIGH    │14:31:45 │0:02    │P001_TRIP   │P-001 Trip   │  -  │ N │
│ HIGH    │14:20:10 │0:14    │P002_FAULT  │P-002 Fault  │  -  │ Y │
│ MED     │14:15:30 │0:19    │FI101_LO    │FI-101 Low   │12.3 │ Y │
└────────────────────────────────────────────────────────────────┘
```

### Alarm Navigation

```
// เมื่อ Click บน Alarm → Navigate ไป Detail Screen:
// แสดง:
// - Process Graphic ที่เกี่ยวข้อง
// - Trend ของ Tag ที่ Alarm
// - ขั้นตอนการแก้ไข (Operating Procedure)
// - ประวัติ Alarm ที่ผ่านมา
```

---

## Step 730: Alarm Suppression และ Shelving

### Alarm Suppression Logic

```
// สถานการณ์: ระงับ Alarm ระดับต่ำระหว่าง Startup
// เหตุผล: ระหว่าง Startup อาจมีค่า Process ผิดปกติชั่วคราว
// โดยไม่จำเป็นต้อง Alarm

// DB "Alarm_Config":
Startup_Mode     : BOOL  // กำลัง Startup อยู่
Suppress_Low_Pri : BOOL  // ระงับ Alarm Priority ต่ำ
Suppress_Reason  : INT   // เหตุผลในการระงับ

// Ladder:
Network 1: ตั้ง Startup Mode

  |                                                         |
  |--[P]----------[SET]-------------------------------------|
  |  "OB100"       "Startup_Mode"                           |
  |  (Startup)                                              |
  |                                                         |

Network 2: Timer สิ้นสุด Startup (5 นาที)

  |                                                         |
  |--[ ]----------[TON]-------------------------------------|
  |  "Startup_Mode"  T_Startup_Timer                        |
  |                  PT: T#5m                               |
  |                                                         |
  |--[ ]----------[RESET]----------------------------------|
  |  T_Timer.Q     "Startup_Mode"                           |
  |                                                         |

Network 3: ระงับ Low Priority Alarm ระหว่าง Startup

  |                                                         |
  |--[ ]----------[RESET]----------------------------------|
  |  "Startup_Mode"  "Low_Alarm_Enable"                     |
  |               (ระงับ Low Alarm)                         |
  |                                                         |
  |--[NOT]--------[SET]-------------------------------------|
  |  "Startup_Mode"  "Low_Alarm_Enable"                     |
  |               (เปิด Low Alarm เมื่อปกติ)                |
  |                                                         |
```

### State-based Alarming

```
// ตัวอย่าง: เครื่อง Filling ที่มี 3 State
// State 0: IDLE → ต้องการ Alarm น้อย
// State 1: FILLING → Alarm เต็มรูปแบบ
// State 2: CLEANING → ระงับ Level/Flow Alarms

// DB "State_Alarm_Config":
//   ARRAY[0..2] OF STRUCT
//     Enable_Critical: BOOL
//     Enable_High    : BOOL
//     Enable_Medium  : BOOL
//     Enable_Low     : BOOL
//   END_STRUCT

Network: ปรับ Alarm Enable ตาม State

  |                                                         |
  |--[=]----------[MOVE]-----------------------------------|
  |  State = 0     IN: "State_Config[0].Enable_Critical"   |
  |               OUT: "Critical_Alarm_Enable"             |
  |                                                         |
  |--[=]----------[MOVE]-----------------------------------|
  |  State = 1     IN: "State_Config[1].Enable_Critical"   |
  |               OUT: "Critical_Alarm_Enable"             |
  |                                                         |
  |--[=]----------[MOVE]-----------------------------------|
  |  State = 2     IN: "State_Config[2].Enable_Critical"   |
  |               OUT: "Critical_Alarm_Enable"             |
  |                                                         |
```

---

## Step 731-735: Advanced Alarm Techniques

### Chattering Alarm Solution

```
// Chattering = Alarm On/Off บ่อยเกินไป
// เกิดจาก: Process อยู่ใกล้ Setpoint + Noise

// วิธีแก้ 1: On-Delay Timer (Delay Before Alarm)

Network: Temperature High Alarm with Delay

  |                                                         |
  |--[>=]----------[TON]-----------------------------------|
  |  Temp >= 90°C   T_Temp_High                            |
  |                 PT: T#30s (รอ 30 วิก่อน Alarm)         |
  |                                                         |
  |--[ ]----------[SET]-------------------------------------|
  |  T_Temp_High.Q  "Temp_High_Alarm"                       |
  |                                                         |

// วิธีแก้ 2: Hysteresis (ต่าง Setpoint สำหรับ Set/Reset)
// Set Alarm: ที่ 90°C
// Reset Alarm: ที่ 85°C (ต้องลงถึงนี้จึง Reset)

Network: Hysteresis Alarm

  |                                                         |
  |--[>=]--[NOT]---[SET]-------------------------------------|
  |  Temp >= 90   "High_  "Temp_High"                       |
  |               Alarm"                                    |
  |                                                         |
  |--[<]-----------[RESET]----------------------------------|
  |  Temp < 85     "Temp_High"                              |
  |                                                         |
```

### Alarm Grouping

```
// Group Alarms สำหรับ Flood Management
// Group 1: Critical Safety
// Group 2: Process Upsets
// Group 3: Equipment Faults
// Group 4: Diagnostic

// เมื่อ Equipment หยุด → ระงับ Alarms ที่ขึ้นกับมัน

// ตัวอย่าง: เมื่อ Pump P-001 หยุด
// → ระงับ Flow Low Alarm (FI-101)
// → เพราะ Flow Low เกิดจาก Pump หยุด ไม่ใช่ปัญหาใหม่

Network: Parent-Child Alarm Suppression

  |                                                         |
  |--[NOT]----NOT---[RESET]---------------------------------|
  |  "P001_Running"  "FI101_Enable"                        |
  |  (Pump stopped)  (Suppress FI-101 alarm)               |
  |                                                         |
  |--[ ]--------------[SET]--------------------------------|
  |  "P001_Running"    "FI101_Enable"                       |
  |  (Pump running)    (Enable FI-101 alarm)               |
  |                                                         |
```

---

## Step 736-740: Lab Projects

### Lab 1: Alarm Rationalization Workshop

**วัตถุประสงค์:** ทำ Alarm Rationalization สำหรับระบบ Reactor

**โจทย์:**
```
มีอุปกรณ์ใน Reactor ดังนี้:
1. Temperature (TT-001): 4 อุณหภูมิจุด
2. Pressure (PT-001): ความดัน
3. Level (LT-001): ระดับ
4. Flow (FT-001): อัตราการไหล
5. Agitator Motor (M-001): มอเตอร์กวน
6. Cooling Valve (HV-001): วาล์วน้ำเย็น

ทำ Rationalization:
1. กำหนด Alarm Setpoints
2. กำหนด Priority
3. เขียน Corrective Actions
4. ระบุ Dependent Alarms
```

### Lab 2: PLC Alarm Program

**วัตถุประสงค์:** เขียนโปรแกรม Alarm ใน PLC

**โจทย์:**
```
สร้าง Alarm System สำหรับ:
1. Temperature High (Priority: High, Delay: 10s)
2. Temperature HiHi (Priority: Critical, Delay: 2s)
3. Motor Trip (Priority: Critical, Immediate)
4. Communication Fail (Priority: High, Delay: 5s)
5. Low Level (Priority: Medium, Delay: 30s)

Features:
- Alarm History (500 records)
- Acknowledge Function
- Auto Reset เมื่อสภาวะหาย
- Alarm Counter
- KPI: Total Alarms, Unack Alarms
```

### Lab 3: Alarm KPI Dashboard

**วัตถุประสงค์:** สร้าง SCADA Screen แสดง Alarm KPIs

```
Dashboard ต้องแสดง:
1. Current Alarm Rate (alarms/10min)
2. Alarm Flood Status (แสดงเป็น Traffic Light)
3. Top 10 Bad Actors (Bar Chart)
4. Alarm History Trend (24h)
5. Priority Distribution (Pie Chart)
6. Unacknowledged Alarms Count
7. Average Acknowledgement Time

Report:
- Weekly Alarm Management Report
- Bad Actor List with Recommendations
```

---

## สรุป Part 37

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| ISA-18.2 | มาตรฐาน Alarm Management |
| Alarm Types | Process/Equipment/Diagnostic |
| Priority | Critical/High/Medium/Low |
| Rationalization | Master Alarm Database |
| Flood Management | Suppression/Shelving |
| PLC Programming | Set/Reset/Ack Logic |
| History Logging | Circular Buffer Log |
| KPIs | AAR, Chattering, Bad Actors |
| Advanced Techniques | Hysteresis, State-based |

---

## การนำทาง

- [← Part 36: SCADA Integration](part36_scada_integration.md)
- [→ Part 38: Recipe Management](part38_recipe_management.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 37 of 40 | Steps 721-740*
