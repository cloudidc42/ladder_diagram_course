# Part 36: SCADA Integration
## Steps 701-720: การเชื่อมต่อ PLC กับระบบ SCADA

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 4-5 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 24 (Communication), Part 27 (PROFINET), Part 28 (EtherNet/IP)

---

## บทนำ

SCADA (Supervisory Control And Data Acquisition) คือระบบที่ช่วยให้ผู้ปฏิบัติงานสามารถมองเห็น ควบคุม และบันทึกข้อมูลจากกระบวนการผลิตจากส่วนกลาง

หน้าที่หลักของ SCADA:
- แสดงผลสถานะอุปกรณ์และกระบวนการ (Visualization)
- บันทึกข้อมูลย้อนหลัง (Historical Data)
- จัดการ Alarm
- สร้างรายงาน
- ควบคุม Setpoints จากระยะไกล

---

## Step 701: SCADA Overview

### Architecture ของระบบ SCADA

```
┌─────────────────────────────────────────────────────────────┐
│                   SCADA System Architecture                  │
│                                                             │
│  Level 4: Enterprise (ERP/MES)                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  SAP / Oracle ERP    MES System    Reporting        │   │
│  └─────────────────────────────────────────────────────┘   │
│                            ↕ ISA-95                         │
│  Level 3: Site Operations (SCADA/DCS)                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  SCADA Server    Historian    Alarm Server          │   │
│  │  (WinCC/Ignition)  (OSIsoft PI)  (HMI Clients)     │   │
│  └─────────────────────────────────────────────────────┘   │
│                            ↕ OPC UA / OPC DA                │
│  Level 2: Control (PLC/DCS)                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  PLC S7-1500  │  PLC S7-1200  │  DCS ABB 800xA     │   │
│  └─────────────────────────────────────────────────────┘   │
│                            ↕ Field Networks                  │
│  Level 1: Field Devices                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Sensors    Actuators    Drives    Smart Instruments│   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### SCADA Software Comparison

```
┌──────────────┬───────────────┬────────────┬──────────────┐
│  Software    │  Manufacturer │ Platform   │ Strength     │
├──────────────┼───────────────┼────────────┼──────────────┤
│ WinCC        │ Siemens       │ Windows    │ TIA Integration│
│ FactoryTalk  │ Allen-Bradley │ Windows    │ Rockwell Eco  │
│ Wonderware   │ AVEVA (Schneider)│ Windows │ Historian    │
│ Ignition     │ Inductive Auto │ Cross-plat │ Modern, Web  │
│ GENESIS64    │ ICONICS       │ Windows    │ Analytics    │
│ Citect SCADA │ AVEVA         │ Windows    │ Mining/Process│
│ InduSoft     │ AVEVA         │ Windows    │ Embedded HMI │
│ VTScada      │ Trihedral     │ Windows    │ Telecom/SCADA│
└──────────────┴───────────────┴────────────┴──────────────┘
```

---

## Step 702: OPC Server vs OPC UA

### OPC DA (Data Access) - Classic OPC

```
OPC DA Architecture:
┌─────────────────────────────────────────────────────────┐
│                OPC Classic (DA) Architecture            │
│                                                         │
│  OPC Client (SCADA)                                     │
│  │                                                      │
│  │ DCOM                                                 │
│  ↓                                                      │
│  OPC Server (Kepware, Matrikon, etc.)                   │
│  │                                                      │
│  │ Native Protocol (Modbus/PROFIBUS/S7)                │
│  ↓                                                      │
│  PLC / Field Device                                     │
│                                                         │
│  ⚠️ ข้อจำกัด:                                           │
│  - ใช้ DCOM (Windows specific)                          │
│  - Firewall ปัญหา                                       │
│  - ไม่ปลอดภัยสูง                                       │
│  - ยาก Cross-platform                                   │
└─────────────────────────────────────────────────────────┘
```

### OPC UA (Unified Architecture) - Modern Standard

```
OPC UA Architecture:
┌─────────────────────────────────────────────────────────┐
│                OPC UA Architecture                      │
│                                                         │
│  OPC UA Client                                          │
│  ├── SCADA Systems                                      │
│  ├── MES/ERP Systems                                    │
│  └── Mobile/Web Apps                                    │
│       ↕                                                 │
│  OPC UA Server                                          │
│  ├── Built-in: Siemens S7-1500, S7-1200               │
│  ├── Kepware OPC UA Server                             │
│  └── Matrikon OPC UA                                   │
│       ↕                                                 │
│  PLC / Field Device                                     │
│                                                         │
│  ✓ ข้อดี:                                               │
│  + Platform Independent (Windows/Linux/Embedded)        │
│  + Built-in Security (Certificates, Encryption)        │
│  + Cross-platform (Java, Python, .NET, C++)            │
│  + Pub/Sub Model (IIoT Ready)                          │
│  + Information Model (Complex Data Types)              │
└─────────────────────────────────────────────────────────┘
```

### S7-1500 Built-in OPC UA Server Configuration

```
ใน TIA Portal:
1. CPU Properties → OPC UA Server
2. เปิดใช้งาน OPC UA
3. กำหนด:
   ├── Security Policy: Basic256Sha256 (แนะนำ)
   ├── Port: 4840 (default)
   ├── Certificate: สร้างใหม่หรือ import
   └── Server Node: ตั้งชื่อ Server

// กำหนด Tags ที่จะ Expose:
// 1. ใน Variable Table → เลือก "Accessible from HMI/OPC UA"
// 2. หรือ ใน DB → กำหนด "Accessible from HMI/OPC UA"

// Tag Structure ใน OPC UA:
// Server: opc.tcp://192.168.1.100:4840
// Namespace: urn:S7_1500
// NodeId: "GVL_Data.Temperature"
```

---

## Step 703: Wonderware/AVEVA

### AVEVA System Platform (ก่อนหน้า Wonderware)

```
Architecture:
┌─────────────────────────────────────────────────────────┐
│  System Platform Components:                            │
│                                                         │
│  ArchestrA:                                             │
│  ├── Application Server (Central Engine)               │
│  ├── Industrial Application Server                     │
│  ├── InTouch HMI (Visualization)                      │
│  └── Historian (Data Storage)                          │
│                                                         │
│  Object Model:                                          │
│  ├── Template (Class): กำหนดโครงสร้าง                  │
│  └── Instance (Object): instance จาก Template          │
│                                                         │
│  Communication:                                         │
│  ├── I/O Servers (DA Server for each PLC brand)        │
│  └── Direct Link to OPC UA/DA                          │
└─────────────────────────────────────────────────────────┘
```

### InTouch - Tag Configuration

```
InTouch Tag Database:
Tag Name: Conv1_Speed
Tag Type: I/O Real
I/O Server: Siemens_S7_Server
Item: S7:[S7Connection]DB10,REAL4
Group: Conveyor_1
Comment: สายพานที่ 1 - ความเร็ว m/min
Alarm:
  Hi: 55.0 (Warning)
  HiHi: 60.0 (Critical)
  Lo: 0.0
  LoLo: 0.0
Engineering Range: 0-60 m/min
```

---

## Step 704: WinCC (Siemens)

### WinCC Configuration

```
WinCC Versions:
├── WinCC Basic  → สำหรับ Panel HMI (S7-1200)
├── WinCC Comfort → Advanced Panel
├── WinCC Advanced → PC-based HMI
├── WinCC Professional → SCADA with Historian
└── WinCC Unified → Modern Web-based (New)

TIA Portal + WinCC Integration:
1. Add HMI Device to Project
2. PLC-HMI Connection: Automatic via Project
3. Tag Management: ดึง Tags จาก PLC อัตโนมัติ
4. Screen Design: Drag & Drop
```

### WinCC Tag สำหรับ S7-1500

```
// ใน TIA Portal WinCC:
// Tag ถูก Link โดยอัตโนมัติจาก PLC Variable

// External Tags (OPC/Modbus):
Tag Name   : Ext_Temperature
Connection : OPC UA Server
Data Type  : Float
Address    : ns=3;s="Temperature_DB".Temp_Value

// PLC Internal Tags:
Tag Name   : Temperature
PLC Tag    : "Temperature_DB".Temp_Value
Data Type  : Float
Update Rate: 1 second
```

---

## Step 705: Ignition SCADA

### Ignition (Inductive Automation)

```
Ignition ข้อดีเด่น:
├── Web-based (ไม่ต้องติดตั้ง Client)
├── Unlimited Tags (ไม่จ่ายตาม Tag Count)
├── Cross-platform (Windows/Linux/Mac/Mobile)
├── Python Scripting
├── Built-in Historian (Time Series DB)
├── Report Designer
└── Machine Learning Integration

Modules:
├── Vision (Traditional HMI)
├── Perspective (Modern Web)
├── OPC UA (Built-in OPC Server/Client)
├── Reporting
├── Alarming
└── SQL Bridge (Database Integration)
```

### Ignition OPC Tag Configuration

```json
{
  "name": "Conveyor_Speed",
  "tagType": "OPC",
  "opcServer": "Kepware.KEPServerEX.V6",
  "opcItemPath": "Channel1.Device1.Conveyor.Speed",
  "dataType": "Float4",
  "scanClass": "Default",
  "historyEnabled": true,
  "historyProvider": "default",
  "sampleMode": "OnChange",
  "historicalDeadband": 0.5,
  "alarms": [
    {
      "name": "HighSpeed",
      "mode": "AboveSetpoint",
      "setpoint": 55.0,
      "priority": "Warning"
    }
  ]
}
```

### Ignition Python Script ตัวอย่าง

```python
# Ignition Scripting - ประมวลผลข้อมูลเมื่อ Event เกิดขึ้น

# Gateway Event Script: ทำงานทุก 1 นาที
def minuteScheduled():
    # อ่านค่าจาก PLC
    temp = system.tag.readBlocking(["[default]PLC/Temperature"])[0].value
    pressure = system.tag.readBlocking(["[default]PLC/Pressure"])[0].value
    
    # บันทึกลง Database
    query = """
        INSERT INTO process_log (timestamp, temperature, pressure)
        VALUES (NOW(), ?, ?)
    """
    system.db.runPrepUpdate(query, [temp, pressure])
    
    # ตรวจสอบ Alarm
    if temp > 90.0:
        system.alarm.shelve("[default]PLC/Temperature_High", 
                           5*60)  # Shelve 5 minutes
        system.util.sendEmail(
            smtp="smtp.company.com",
            fromAddress="scada@company.com",
            toAddresses=["engineer@company.com"],
            subject="Temperature Alarm",
            body=f"Temperature = {temp}°C exceeds limit!"
        )
```

---

## Step 706: PLC-to-SCADA Tag Mapping

### Tag Naming Convention

```
ตัวอย่างมาตรฐาน Tag Naming:

Plant Code: TH01
Unit: U01 (Unit 1)
Equipment: MOT (Motor), VLV (Valve), TMP (Temperature)
Number: 001
Point: SP (Setpoint), PV (Process Value), OP (Output)
Status: ST (Status), AL (Alarm)

Format: [Plant]_[Unit]_[Equip][No]_[Point]

ตัวอย่าง:
TH01_U01_TMP001_PV = Temperature Sensor 1, Process Value
TH01_U01_TMP001_SP = Temperature Sensor 1, Setpoint
TH01_U01_TMP001_AL = Temperature Sensor 1, Alarm
TH01_U01_MOT001_ST = Motor 1, Status
TH01_U01_VLV001_OP = Valve 1, Output
```

### Tag Mapping Table

```
PLC DB "Process_Data" → SCADA Tags:

PLC Address                        SCADA Tag               Type   Update
---------------------------------- ----------------------- ------ -------
"Process_Data".Zone1_Temp_PV      TH01_U01_TMP001_PV     Real   1s
"Process_Data".Zone1_Temp_SP      TH01_U01_TMP001_SP     Real   1s
"Process_Data".Zone1_Temp_Alarm   TH01_U01_TMP001_AL     Bool   OnChange
"Process_Data".Motor1_Status      TH01_U01_MOT001_ST     Bool   OnChange
"Process_Data".Motor1_Speed       TH01_U01_MOT001_SPD    Real   2s
"Process_Data".Valve1_Position    TH01_U01_VLV001_POS    Real   2s
"Process_Data".Conveyor1_Speed    TH01_U01_CNV001_SPD    Real   1s
"Process_Data".Batch_Count        TH01_U01_BCH001_CNT    DInt   OnChange
"Process_Data".Production_Rate    TH01_U01_PRD001_RATE   Real   5s
```

---

## Step 707: Historian Setup

### Historian Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   Historian System                         │
│                                                            │
│  Data Sources:                                             │
│  ├── PLC (via OPC DA/UA)                                  │
│  ├── DCS                                                   │
│  ├── Sensors (directly)                                    │
│  └── Database (manual entry)                              │
│            ↓                                               │
│  Compression & Storage:                                    │
│  ├── Raw Data: ทุก Scan (อาจ 1-10s)                      │
│  ├── Exception Reporting: เมื่อค่าเปลี่ยน ±X%            │
│  └── Swinging Door Compression                            │
│            ↓                                               │
│  Data Retrieval:                                           │
│  ├── Trend Display (Real-time + Historical)               │
│  ├── Reporting                                             │
│  ├── Analytics                                             │
│  └── Calculations (Averages, Min, Max)                    │
└────────────────────────────────────────────────────────────┘
```

### OSIsoft PI (ตอนนี้ AVEVA PI System)

```
PI Server Configuration:
1. สร้าง PI Tag (Point) สำหรับแต่ละ Data Point
2. กำหนด Archive Parameters
3. กำหนด Exception/Compression

PI Tag Example:
Tag Name   : TH01.U01.TMP001.PV
Location   : Thailand Plant 1
Unit        : degC
Zero        : 0
Span        : 150
Step        : 0 (Interpolated)
Exc Min    : 0.5 (Exception: ≥ 0.5°C change)
Compression: On
Comp Min   : 0.1
Point Source: OPC

// PI OLEDB/AF Queries
// ดึงข้อมูลเฉลี่ยรายชั่วโมง:
SELECT value, time 
FROM piarchive..piinterp
WHERE tag = 'TH01.U01.TMP001.PV'
  AND time BETWEEN '*-24h' AND '*'
  AND timestep = '1h'
```

---

## Step 708: Trending

### Real-time Trend

```
Trend Configuration ใน SCADA:
┌─────────────────────────────────────────────────────────┐
│  Trend Pen Configuration                                │
│                                                         │
│  Pen 1: Temperature Zone 1                             │
│  Color: Red                                             │
│  Range: 0-150°C                                        │
│  Source: TH01_U01_TMP001_PV                            │
│  Update: 1 second                                       │
│                                                         │
│  Pen 2: Temperature Setpoint                           │
│  Color: Blue (Dashed)                                   │
│  Range: 0-150°C                                        │
│  Source: TH01_U01_TMP001_SP                            │
│                                                         │
│  Pen 3: Output                                         │
│  Color: Green                                           │
│  Range: 0-100%                                         │
│  Source: TH01_U01_TMP001_OP                            │
└─────────────────────────────────────────────────────────┘
```

### Historical Trend Retrieval

```python
# Ignition Python: ดึงข้อมูลประวัติ
import system

def getHistoricalData(tagPath, startTime, endTime, intervalMinutes):
    """
    ดึงข้อมูลประวัติจาก Historian
    """
    dataset = system.tag.queryTagHistory(
        paths=[tagPath],
        startDate=startTime,
        endDate=endTime,
        returnSize=-1,      # -1 = ทุก Records
        aggregationMode="Average",
        returnFormat="Wide",
        columnNames=["Temperature"],
        intervalHours=0,
        intervalMinutes=intervalMinutes
    )
    return dataset

# ตัวอย่างการใช้:
data = getHistoricalData(
    tagPath="[default]PLC/Zone1_Temperature",
    startTime=system.date.addHours(system.date.now(), -24),
    endTime=system.date.now(),
    intervalMinutes=5
)
# data เป็น Dataset ที่นำไป plot Trend ได้
```

---

## Step 709: Alarm Integration

### Alarm Configuration ใน SCADA

```
Alarm Types สำหรับ Process:
┌──────────────────────────────────────────────────────┐
│  Priority │ Color  │ Type      │ Example             │
├──────────────────────────────────────────────────────┤
│  Critical │ Red    │ HiHi/LoLo │ Emergency, Safety   │
│  High     │ Orange │ Hi/Lo     │ Process Limit       │
│  Medium   │ Yellow │ Rate      │ Trend Alarm         │
│  Low      │ Blue   │ Diagnostic│ Status Change       │
└──────────────────────────────────────────────────────┘
```

### Alarm Setpoint Configuration

```
// TIA Portal WinCC Alarm Config:
// Tag-based Alarm

Tag: "Process_DB".Zone1_Temperature
Alarm Type: High Limit
  HiHi Limit: 95.0°C  Priority: Critical (ดับการผลิต)
  Hi Limit:   90.0°C  Priority: High (Warning)
  
Tag: "Process_DB".Zone1_Temperature
Alarm Type: Low Limit
  Lo Limit:   60.0°C  Priority: High (Heating failure)
  LoLo Limit: 55.0°C  Priority: Critical

// Alarm Message:
"อุณหภูมิโซน 1 สูงเกินกำหนด - ค่า: {0}°C - เกณฑ์: 90°C"
```

---

## Step 710: Report Generation

### Shift Report

```python
# Ignition Report Script
# สร้าง Shift Report ทุก 8 ชั่วโมง

def generateShiftReport(shiftNumber, startTime, endTime):
    """
    สร้างรายงานกะการผลิต
    """
    
    # ดึงข้อมูล Production
    query = """
        SELECT 
            AVG(temperature) as avg_temp,
            MAX(temperature) as max_temp,
            MIN(temperature) as min_temp,
            SUM(production_count) as total_count,
            SUM(reject_count) as total_rejects
        FROM production_data
        WHERE timestamp BETWEEN ? AND ?
    """
    data = system.db.runPrepQuery(query, [startTime, endTime])
    
    # ดึงข้อมูล Alarm
    alarm_query = """
        SELECT COUNT(*) as alarm_count, 
               alarm_name, 
               SUM(duration_seconds) as total_duration
        FROM alarm_history
        WHERE event_time BETWEEN ? AND ?
        GROUP BY alarm_name
        ORDER BY alarm_count DESC
    """
    alarms = system.db.runPrepQuery(alarm_query, [startTime, endTime])
    
    # สร้าง Report
    reportParams = {
        "shiftNumber": shiftNumber,
        "startTime": system.date.format(startTime, "dd/MM/yyyy HH:mm"),
        "endTime": system.date.format(endTime, "dd/MM/yyyy HH:mm"),
        "productionData": data,
        "alarmData": alarms,
        "oee": calculateOEE(data)
    }
    
    # Export PDF
    bytes = system.report.executeReport(
        path="Shift_Report_Template",
        project="Production",
        parameters=reportParams,
        fileType="PDF"
    )
    
    # ส่ง Email
    system.util.sendEmail(
        smtp="mail.company.com",
        fromAddress="scada@company.com",
        toAddresses=["supervisor@company.com"],
        subject=f"Shift {shiftNumber} Report",
        body="กรุณาดูรายงานแนบมาด้วย",
        attachmentNames=[f"Shift{shiftNumber}_Report.pdf"],
        attachmentData=[bytes]
    )
```

---

## Step 711: MES Integration

### MES (Manufacturing Execution System)

```
ISA-95 Model (Purdue Model):
┌────────────────────────────────────────────────────────┐
│  Level 4: Business Planning (ERP)                      │
│           SAP, Oracle                                  │
├────────────────────────────────────────────────────────┤
│  Level 3: Manufacturing Operations Management (MES)    │
│           Production Scheduling, Quality, Inventory    │
│           Genealogy, Maintenance, Labor, Performance   │
├────────────────────────────────────────────────────────┤
│  Level 2: Supervisory Control (SCADA/DCS)              │
│           WinCC, Ignition, Wonderware                  │
├────────────────────────────────────────────────────────┤
│  Level 1: Basic Control (PLC/DCS)                      │
│           S7-1500, Logix5000, DeltaV                   │
├────────────────────────────────────────────────────────┤
│  Level 0: Physical Process                             │
│           Sensors, Actuators, Machines                 │
└────────────────────────────────────────────────────────┘
```

### MES Data Exchange

```
ข้อมูลที่ไหลจาก PLC → SCADA → MES:

1. Production Data:
   - Batch/Lot Number
   - Product Code
   - Actual Production Count
   - Reject Count
   - Cycle Time per Piece

2. Equipment Data:
   - Machine Status (Running/Idle/Down)
   - Downtime Reasons
   - OEE Components

3. Quality Data:
   - Process Parameters (Temperature, Pressure)
   - Inspection Results
   - SPC Data

4. Material Data:
   - Material Consumption
   - Lot Traceability
   - WIP (Work In Progress)

// PLC → SCADA Tag:
"MES_Exchange_DB".Batch_Number     → MES_Batch_No
"MES_Exchange_DB".Production_Count → MES_Actual_Qty
"MES_Exchange_DB".Cycle_Time       → MES_Cycle_Time
"MES_Exchange_DB".OEE_Value        → MES_OEE
```

---

## Step 712: Industry 4.0 Connectivity

### IIoT Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   IIoT Architecture                        │
│                                                            │
│  Cloud Layer:                                              │
│  ├── AWS IoT Core / Azure IoT Hub                        │
│  ├── Analytics (AWS SageMaker / Azure ML)                │
│  └── Digital Twin                                         │
│            ↕ MQTT / AMQP                                   │
│  Edge Layer:                                               │
│  ├── Edge Gateway (Siemens Industrial Edge)               │
│  ├── AI Inference Engine                                  │
│  └── Local MQTT Broker                                    │
│            ↕ OPC UA / MQTT                                 │
│  Control Layer:                                            │
│  ├── PLC with OPC UA Server                               │
│  └── SCADA System                                         │
└────────────────────────────────────────────────────────────┘
```

### MQTT สำหรับ IIoT

```python
# Python Script ใน Edge Gateway
# ส่งข้อมูล PLC ไป Cloud ผ่าน MQTT

import paho.mqtt.client as mqtt
import json
import time

# OPC UA Client อ่านจาก PLC
from opcua import Client as OPCClient

def sendTelemetryToCloud():
    # เชื่อมต่อ PLC ผ่าน OPC UA
    opc_client = OPCClient("opc.tcp://192.168.1.10:4840")
    opc_client.connect()
    
    # อ่านค่า
    temperature = opc_client.get_node(
        "ns=3;s=Temperature_DB.Temperature"
    ).get_value()
    
    # สร้าง MQTT Message
    payload = json.dumps({
        "deviceId": "PLC_LINE_1",
        "timestamp": int(time.time() * 1000),
        "data": {
            "temperature": temperature,
            "unit": "celsius"
        }
    })
    
    # ส่งไป Cloud
    mqtt_client.publish(
        topic="factory/line1/process/temperature",
        payload=payload,
        qos=1
    )
    
    opc_client.disconnect()
```

### Siemens Industrial Edge

```
Industrial Edge Apps:
├── SINEMA Remote Connect (Remote Access)
├── Industrial AI (Condition Monitoring)
├── Data Service (Time Series Collection)
├── Flow Creator (Node-RED based)
└── IIHub (Data Publishing to Cloud)

PLC → Edge App → Cloud:
1. S7-1500 OPC UA Server
2. Industrial Edge: Data Service App
   - Subscribe to OPC UA Tags
   - Store in Edge Database
   - Forward to Cloud (IoT Hub/PI Cloud)
3. Azure IoT Hub: Receive & Analyze
```

---

## Step 713-720: Lab Projects

### Lab 1: OPC UA Configuration

**วัตถุประสงค์:** เชื่อมต่อ S7-1500 กับ SCADA ผ่าน OPC UA

**ขั้นตอน:**
1. กำหนดค่า OPC UA Server ใน S7-1500
2. ติดตั้ง Kepware หรือ Ignition OPC UA Server
3. Subscribe Tags จาก PLC
4. สร้าง Real-time Trend Display
5. กำหนด Alarm สำหรับ Temperature และ Pressure

### Lab 2: Historian Integration

**วัตถุประสงค์:** บันทึกและ Retrieve Historical Data

**โจทย์:**
```
1. กำหนดค่า Historian ใน Ignition/WinCC
2. กำหนด Archive Tags:
   - Temperature (ทุก 1 วินาที + Exception 0.5°C)
   - Pressure (ทุก 5 วินาที)
   - Motor Speed (OnChange)
   - Production Count (OnChange)

3. สร้าง Historical Trend แสดง:
   - 24 ชั่วโมงย้อนหลัง
   - Min/Max/Average

4. Shift Report:
   - Production Summary
   - Alarm List
   - Energy Consumption
```

### Lab 3: Complete SCADA System

```
โจทย์: ออกแบบ SCADA สำหรับสายการผลิตสมบูรณ์

1. Overview Screen:
   - Plant Layout (ภาพรวม)
   - Key KPIs: OEE, Production Rate, Energy
   - Alarm Summary

2. Detail Screens:
   - Process Control (PID Loop Display)
   - Equipment Status
   - Trend Display

3. Alarm System:
   - Priority-based Alarm List
   - Alarm Acknowledgement
   - Alarm History

4. Report Generation:
   - Daily Production Report
   - Weekly Quality Report
   - Monthly Energy Report

5. Security:
   - User Login
   - Role-based Access (Operator/Engineer/Manager)
   - Audit Trail
```

---

## สรุป Part 36

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| SCADA Overview | Architecture, Software Options |
| OPC UA | Modern, Secure, Cross-platform |
| Tag Mapping | Naming Convention, Mapping Table |
| Historian | Data Archiving, Retrieval |
| Trending | Real-time & Historical |
| Alarms | Priority, Configuration |
| Reports | Shift Report, PDF Generation |
| MES | ISA-95, Data Exchange |
| Industry 4.0 | MQTT, IIoT, Edge Computing |

---

## การนำทาง

- [← Part 35: Variable Frequency Drive](part35_vfd_control.md)
- [→ Part 37: Alarm Management](part37_alarm_management.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 36 of 40 | Steps 701-720*
