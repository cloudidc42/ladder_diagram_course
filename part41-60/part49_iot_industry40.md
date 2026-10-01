# Part 49: IoT & Industry 4.0 (Steps 961-980)

## ภาพรวม
Industry 4.0 คือการปฏิวัติอุตสาหกรรมครั้งที่ 4 ที่ผสมผสาน Physical Systems กับ Digital World ผ่าน IoT, AI, Big Data และ Cloud Computing เนื้อหานี้ครอบคลุมการเชื่อมต่อ PLC กับโลกดิจิทัล

---

## Step 961: Industry 4.0 Framework

### เสาหลักของ Industry 4.0

```
Industry 4.0 Pillars:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. Cyber-Physical Systems (CPS)                           │
│     └── PLC + sensors + actuators + digital twin          │
│                                                             │
│  2. Industrial IoT (IIoT)                                  │
│     └── Connected devices, edge/cloud integration         │
│                                                             │
│  3. Big Data & Analytics                                   │
│     └── Process data → insights → decisions               │
│                                                             │
│  4. Cloud Computing                                        │
│     └── Scalable storage and computing on demand          │
│                                                             │
│  5. Artificial Intelligence / Machine Learning            │
│     └── Predictive maintenance, quality prediction        │
│                                                             │
│  6. Additive Manufacturing                                 │
│     └── 3D printing for custom parts                      │
│                                                             │
│  7. Augmented Reality (AR)                                 │
│     └── Maintenance guidance, training                    │
│                                                             │
│  8. Simulation / Digital Twin                              │
│     └── Virtual commissioning, what-if analysis          │
│                                                             │
│  9. Horizontal + Vertical Integration                      │
│     └── Supply chain + factory floor + management         │
│                                                             │
│  10. Cybersecurity                                         │
│      └── Protect connected manufacturing                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 962: IIoT Architecture

### ชั้นของ IIoT Architecture

```
IIoT Architecture Layers:
┌─────────────────────────────────────────────────────────────┐
│  CLOUD LAYER (AWS, Azure, GCP)                             │
│  ├── Long-term storage (Cold tier)                         │
│  ├── Advanced analytics (ML models)                        │
│  ├── Enterprise integration (ERP, MES)                     │
│  └── Global visibility                                     │
│                        ▲                                   │
│                        │ (MQTT, HTTPS, AMQP)               │
│                        │                                   │
│  EDGE LAYER (On-premise servers, edge gateways)           │
│  ├── Real-time analytics                                   │
│  ├── Local data storage (historian)                        │
│  ├── Protocol translation (OPC-UA → MQTT)                 │
│  └── Preprocessing (filter, aggregate, normalize)         │
│                        ▲                                   │
│                        │ (OPC-UA, PROFINET, EtherNet/IP)   │
│                        │                                   │
│  FIELD LAYER (PLC, DCS, HMI)                              │
│  ├── Process control (real-time)                          │
│  ├── Data acquisition                                     │
│  └── Equipment monitoring                                 │
│                        ▲                                   │
│                        │ (4-20mA, HART, IO-Link)           │
│                        │                                   │
│  DEVICE LAYER (Sensors, actuators)                        │
│  └── Physical process measurement                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 963: Edge Computing

### Edge Computing ในอุตสาหกรรม

```
Edge Computing Benefits:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Without Edge:                                             │
│  Sensor Data → Cloud → Decision → Back to machine         │
│  Latency: 50-500ms (too slow for real-time control)       │
│                                                             │
│  With Edge:                                                │
│  Sensor Data → Edge → Real-time Decision                  │
│  Latency: < 5ms                                           │
│                                                             │
│  Edge Device Examples:                                     │
│  ├── Industrial PC (IPC)                                  │
│  │   Example: Siemens IPC427E, Kontron KBox B-202        │
│  ├── Edge Gateway                                         │
│  │   Example: Moxa MC-1121, Red Lion Crimson 310         │
│  └── IoT Gateway                                          │
│      Example: Azure IoT Edge device                      │
│                                                             │
│  Edge Software:                                            │
│  ├── Node-RED (flow-based programming)                    │
│  ├── Azure IoT Edge runtime                               │
│  ├── AWS Greengrass                                       │
│  └── Custom Python/C++ applications                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 964: Cloud Connectivity

### AWS IoT Core

```
AWS IoT Core Integration:
┌─────────────────────────────────────────────────────────────┐
│  Architecture:                                              │
│                                                             │
│  PLC ──OPC-UA──► Edge Gateway ──MQTT──► AWS IoT Core      │
│                   (Node-RED)             │                  │
│                                          ├── IoT Rules     │
│                                          │   (filter/route)│
│                                          ├── DynamoDB      │
│                                          │   (time series) │
│                                          ├── S3            │
│                                          │   (cold store)  │
│                                          └── Lambda        │
│                                              (processing)  │
│                                                             │
│  Authentication: X.509 Certificate (เป็น requirement!)     │
│  Each device ต้องมี unique certificate                    │
│                                                             │
│  Topic Structure:                                          │
│  plant/area/machine/datatype                              │
│  Example: plant1/assembly/PLC001/temperature              │
└─────────────────────────────────────────────────────────────┘
```

### Azure IoT Hub

```
Azure IoT Hub:
┌─────────────────────────────────────────────────────────────┐
│  Connection Methods:                                        │
│  ├── MQTT (port 8883, TLS)                                 │
│  ├── AMQP (port 5671)                                     │
│  └── HTTPS (port 443)                                     │
│                                                             │
│  Message Types:                                            │
│  ├── Device-to-Cloud (telemetry data)                     │
│  ├── Cloud-to-Device (commands)                           │
│  └── Device Twin (configuration sync)                     │
│                                                             │
│  Azure Services:                                           │
│  ├── Azure Stream Analytics (real-time processing)        │
│  ├── Azure Time Series Insights (visualization)           │
│  ├── Azure ML (machine learning)                          │
│  └── Power BI (business reporting)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 965: MQTT Protocol

### MQTT สำหรับ IIoT

```
MQTT (Message Queuing Telemetry Transport):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  MQTT Architecture:                                         │
│  Publisher ──► Broker ──► Subscriber                       │
│  (PLC/edge)    (MQTT   (Cloud/HMI/                        │
│                 server)  analytics)                        │
│                                                             │
│  Key Features:                                             │
│  ├── Lightweight (small overhead)                         │
│  ├── QoS 0: At most once (fire and forget)               │
│  ├── QoS 1: At least once (guaranteed delivery)          │
│  ├── QoS 2: Exactly once (highest reliability)           │
│  ├── Last Will: Message ถ้า client disconnect ผิดปกติ    │
│  └── Retained: Last message kept for new subscribers      │
│                                                             │
│  Common Brokers:                                           │
│  ├── Mosquitto (open source, free)                        │
│  ├── AWS IoT Core (managed)                               │
│  ├── Azure IoT Hub (managed)                              │
│  ├── HiveMQ (enterprise)                                  │
│  └── EMQX (high performance)                             │
│                                                             │
│  Payload Format:                                           │
│  └── JSON (most common): {"temp": 75.3, "ts": 1234567}  │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 966: Siemens MindSphere

### MindSphere Platform

```
Siemens MindSphere:
┌─────────────────────────────────────────────────────────────┐
│  MindSphere Architecture:                                   │
│                                                             │
│  Field Level:                                              │
│  S7-1500 ──OPC-UA──► MindConnect Nano/IE ──► MindSphere   │
│                       (Edge gateway)           (Cloud)     │
│                                                             │
│  Connectivity:                                             │
│  MindConnect Nano:                                        │
│  ├── Connects to S7-1500 via OPC-UA or PROFINET          │
│  ├── Local buffering (if internet connection lost)        │
│  └── Secure upload to MindSphere cloud                   │
│                                                             │
│  MindSphere Applications:                                 │
│  ├── Fleet Manager (asset monitoring)                     │
│  ├── MindSphere Predictivity (predictive maintenance)    │
│  ├── MindSphere Analyze MyMachines (machine analytics)   │
│  └── Custom applications via Developer Cockpit           │
│                                                             │
│  TIA Portal Integration:                                   │
│  ├── MindConnect Library for S7-1500                     │
│  └── Direct OPC-UA connection configuration in TIA       │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 967: Rockwell FactoryTalk Analytics

### FactoryTalk Analytics

```
FactoryTalk Analytics (Rockwell):
┌─────────────────────────────────────────────────────────────┐
│  Components:                                                │
│  ├── FactoryTalk Historian (data collection)              │
│  ├── FactoryTalk Analytics Live (real-time)               │
│  ├── FactoryTalk Analytics for Devices (device health)    │
│  └── FactoryTalk Analytics LogixAI (ML at edge)          │
│                                                             │
│  FactoryTalk Logix Diagnostics:                            │
│  ├── Asset health monitoring                              │
│  ├── Real-time vibration analysis                        │
│  ├── Power consumption monitoring                        │
│  └── Predictive maintenance alerts                       │
│                                                             │
│  Integration with Plex/Rockwell Cloud:                    │
│  └── Plex Operations Cloud + FactoryTalk data            │
│                                                             │
│  Edge Analytics:                                           │
│  ├── Models run in ControlLogix controller                │
│  └── No internet required for real-time decisions         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 968: Data Pipeline from PLC to Cloud

### Data Pipeline Design

```
End-to-End Data Pipeline:

┌─────────────────────────────────────────────────────────────┐
│  STEP 1: Data Acquisition (PLC Level)                      │
│  ├── Read process variables every 1 second                │
│  ├── Store in Historian (local)                           │
│  └── Buffer: 7 days of data                              │
│                                                             │
│  STEP 2: Edge Processing                                   │
│  ├── OPC-UA client reads from PLC                         │
│  ├── Filter: Only send changed values (dead-band)        │
│  ├── Aggregate: Calculate averages, min, max              │
│  └── Convert: OPC-UA → MQTT JSON                        │
│                                                             │
│  STEP 3: Cloud Ingestion                                   │
│  ├── MQTT → AWS IoT Core or Azure IoT Hub                │
│  ├── Message routing based on topic                      │
│  └── Store in time-series database                       │
│                                                             │
│  STEP 4: Analytics                                         │
│  ├── Real-time: Anomaly detection                        │
│  ├── Historical: Trend analysis                          │
│  └── ML: Predictive models                              │
│                                                             │
│  STEP 5: Visualization                                     │
│  ├── Grafana dashboards                                   │
│  ├── Power BI reports                                    │
│  └── Mobile alerts                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 969: Digital Twin Basics

### Digital Twin Concept

```
Digital Twin:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  PHYSICAL ASSET          DIGITAL TWIN                      │
│  ┌─────────────┐        ┌──────────────────────────────┐  │
│  │  Machine    │◄──────►│  3D Model + Physics Model    │  │
│  │  (Real)     │  data  │  + Current State Data        │  │
│  └─────────────┘        │  + Historical Data           │  │
│                         └──────────────────────────────┘  │
│                                                             │
│  Use Cases:                                                │
│  ├── Virtual Commissioning: Test PLC program without      │
│  │   real machine (Siemens NX MCD, Rockwell Emulate3D)   │
│  ├── Remote Monitoring: See machine state in 3D           │
│  ├── What-if Analysis: Simulate process changes          │
│  └── Training: Operators learn on virtual machine         │
│                                                             │
│  Platforms:                                               │
│  ├── Siemens Industrial Operations X (formerly MindSphere)│
│  ├── Microsoft Azure Digital Twins                        │
│  ├── AWS IoT TwinMaker                                    │
│  └── GE Predix                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 970: Predictive Analytics

### Predictive Maintenance

```
Predictive Maintenance Strategy:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  REACTIVE (ซ่อมเมื่อเสีย):                                 │
│  Machine fails → Repair → Restart                          │
│  Cost: High (downtime + emergency repair)                 │
│                                                             │
│  PREVENTIVE (ซ่อมตามกำหนด):                                │
│  Schedule → Inspect/Replace → Continue                    │
│  Cost: Medium (planned downtime + unnecessary replacement) │
│                                                             │
│  PREDICTIVE (ซ่อมเมื่อจะเสีย):                             │
│  Monitor → Detect anomaly → Plan repair → Fix before fail │
│  Cost: Low (minimal downtime, only needed maintenance)    │
│                                                             │
│  Data Sources for Prediction:                             │
│  ├── Vibration (accelerometer on motor/bearing)          │
│  ├── Current signature (motor current analysis)          │
│  ├── Temperature trend (bearing, winding)               │
│  ├── Pressure differential (filter clogging)            │
│  └── Oil analysis (metal particles, viscosity)          │
│                                                             │
│  ML Algorithms Used:                                      │
│  ├── LSTM (Long Short-Term Memory) neural networks       │
│  ├── Isolation Forest (anomaly detection)                │
│  ├── Random Forest (classification)                      │
│  └── Statistical: Moving average, control charts         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 971: Cybersecurity for IIoT

### OT/ICS Cybersecurity

```
OT Security Framework (IEC 62443):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Defense in Depth:                                         │
│                                                             │
│  Internet/WAN                                             │
│      │                                                     │
│  [Firewall 1] ──── DMZ ────[Firewall 2]                   │
│                    │                                       │
│               Historian                                   │
│               (data diode)                               │
│                    │                                       │
│  [Firewall 3] ──── Level 3 (Site Business)               │
│      │                                                     │
│      └──── Level 2 (Supervisory - SCADA/HMI)             │
│                │                                          │
│                └──── Level 1 (Control - PLC/DCS)         │
│                          │                               │
│                          └──── Level 0 (Field devices)   │
│                                                           │
│  Security Measures:                                       │
│  ├── Network segmentation (VLANs, firewalls)             │
│  ├── Authentication (strong passwords, MFA)              │
│  ├── Encryption (TLS for all communications)            │
│  ├── Patch management (OS and PLC firmware)              │
│  ├── USB control (disable or whitelist)                  │
│  └── Monitoring (SIEM, IDS for OT)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 972-980: Practical PLC to MQTT using Node-RED

### การเชื่อมต่อ PLC กับ MQTT ผ่าน Node-RED

#### 1. Node-RED Overview

```
Node-RED Architecture:
┌─────────────────────────────────────────────────────────────┐
│  Node-RED = Visual programming for IoT                      │
│  ├── Flow-based: Drag and drop nodes                      │
│  ├── Runs on Node.js                                      │
│  ├── Thousands of community nodes available              │
│  └── ฟรีและ open source                                  │
│                                                             │
│  Key Nodes for PLC Integration:                            │
│  ├── node-red-contrib-s7 (Siemens S7 PLC)                │
│  ├── node-red-contrib-opcua (OPC-UA client)              │
│  ├── node-red-contrib-modbus (Modbus TCP)                │
│  └── node-red-contrib-mqtt (MQTT publisher/subscriber)   │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Node-RED Flow: S7-1500 to MQTT

```json
// Node-RED Flow JSON (concept)
// Flow: Read Siemens S7-1500 → Process → Publish to MQTT

[
  {
    "id": "s7in_node",
    "type": "s7 in",
    "name": "Read PLC Data",
    "connection": "plc_connection",
    "mode": "single",
    "variable": "DB1.DBW0",  // อ่าน word จาก DB1
    "diff": false
  },
  {
    "id": "function_node", 
    "type": "function",
    "name": "Format JSON Payload",
    "func": "// สร้าง JSON payload\nvar payload = {\n    timestamp: new Date().toISOString(),\n    machine: 'PLC001',\n    area: 'Assembly',\n    temperature: msg.payload / 10.0,  // scale\n    unit: 'degC'\n};\nmsg.payload = JSON.stringify(payload);\nmsg.topic = 'plant1/assembly/PLC001/temperature';\nreturn msg;"
  },
  {
    "id": "mqtt_out",
    "type": "mqtt out",
    "name": "Publish to MQTT",
    "broker": "mqtt_broker",
    "topic": "",  // จาก msg.topic
    "qos": "1",
    "retain": false
  }
]
```

#### 3. Complete PLC Program สำหรับ IIoT Data Preparation

```pascal
// FB: IIoT_DataPrep
// เตรียมข้อมูลสำหรับส่งไปยัง cloud
// Runs every 1 second

FUNCTION_BLOCK IIoT_DataPrep
VAR_INPUT
    // Process data
    TIC_101_PV    : REAL;   // Temperature
    PIC_102_PV    : REAL;   // Pressure
    FIC_103_PV    : REAL;   // Flow rate
    LIC_104_PV    : REAL;   // Level
    
    // Machine status
    Machine_Running : BOOL;
    Alarm_Active    : BOOL;
    Part_Count      : DINT;
END_VAR

VAR_OUTPUT
    // OPC-UA server nodes (exposed for edge gateway to read)
    OPC_Temp       : REAL;
    OPC_Press      : REAL;
    OPC_Flow       : REAL;
    OPC_Level      : REAL;
    OPC_Status     : INT;   // 0=Stopped, 1=Running, 2=Alarm
    OPC_PartCount  : DINT;
    OPC_Timestamp  : DT;    // Current timestamp
    
    // Performance metrics
    OPC_OEE        : REAL;
    OPC_Yield      : REAL;
    OPC_Availability: REAL;
END_VAR

VAR
    // Data validation (deadband filter)
    Prev_Temp     : REAL;
    Prev_Press    : REAL;
    Temp_Changed  : BOOL;
    Press_Changed : BOOL;
    
    // Performance calculation
    Shift_Timer   : TON;
    Good_Parts    : DINT;
    Total_Parts   : DINT;
    Running_Time  : REAL;
    Planned_Time  : REAL;
    
    // Update timing
    Update_Timer  : TON;
END_VAR

BEGIN
    // ---- Deadband Filter (only update if changed > 0.1) ----
    IF ABS(TIC_101_PV - Prev_Temp) > 0.1 THEN
        OPC_Temp := TIC_101_PV;
        Prev_Temp := TIC_101_PV;
    END_IF;
    
    IF ABS(PIC_102_PV - Prev_Press) > 0.01 THEN
        OPC_Press := PIC_102_PV;
        Prev_Press := PIC_102_PV;
    END_IF;
    
    OPC_Flow  := FIC_103_PV;
    OPC_Level := LIC_104_PV;
    
    // ---- Machine Status ----
    IF Alarm_Active THEN
        OPC_Status := 2;   // Alarm
    ELSIF Machine_Running THEN
        OPC_Status := 1;   // Running
    ELSE
        OPC_Status := 0;   // Stopped
    END_IF;
    
    OPC_PartCount := Part_Count;
    OPC_Timestamp := GET_SYSTIME();  // Current system time
    
    // ---- OEE Calculation ----
    // Update every minute
    Update_Timer(IN := TRUE, PT := T#1M);
    IF Update_Timer.Q THEN
        Update_Timer(IN := FALSE);
        Planned_Time := Planned_Time + 1.0;  // minutes
        
        IF Machine_Running THEN
            Running_Time := Running_Time + 1.0;
        END_IF;
        
        Total_Parts := Total_Parts + Part_Count;
        Good_Parts := Good_Parts + Part_Count;  // Assume all good (adjust with inspection)
        
        // Calculate OEE components
        OPC_Availability := Running_Time / Planned_Time * 100.0;
        IF Total_Parts > 0 THEN
            OPC_Yield := DINT_TO_REAL(Good_Parts) / DINT_TO_REAL(Total_Parts) * 100.0;
        END_IF;
        OPC_OEE := OPC_Availability * OPC_Yield / 100.0;
    END_IF;
END_FUNCTION_BLOCK
```

#### 4. Node-RED to MQTT to AWS IoT

```javascript
// Node-RED Function Node: Format for AWS IoT
// รับข้อมูลจาก OPC-UA node แล้ว format เป็น JSON

function formatPayload(msg) {
    var opcData = msg.payload;
    
    // สร้าง message ตาม AWS IoT format
    var payload = {
        // Metadata
        "deviceId": "PLC-PLANT1-ASSEMBLY-001",
        "timestamp": new Date().toISOString(),
        "location": {
            "plant": "Plant1",
            "area": "Assembly",
            "line": "Line-A"
        },
        
        // Process data
        "process": {
            "temperature": {
                "value": opcData.TIC_101,
                "unit": "degC",
                "tag": "TIC-101"
            },
            "pressure": {
                "value": opcData.PIC_102,
                "unit": "bar",
                "tag": "PIC-102"
            },
            "flow": {
                "value": opcData.FIC_103,
                "unit": "m3/h",
                "tag": "FIC-103"
            }
        },
        
        // Performance
        "performance": {
            "oee": opcData.OEE,
            "availability": opcData.Availability,
            "yield": opcData.Yield,
            "partCount": opcData.PartCount
        },
        
        // Status
        "status": {
            "running": opcData.Status === 1,
            "alarm": opcData.Status === 2,
            "statusCode": opcData.Status
        }
    };
    
    msg.payload = JSON.stringify(payload);
    msg.topic = `plant1/assembly/PLC001/telemetry`;
    
    return msg;
}

return formatPayload(msg);
```

#### 5. AWS IoT Rule สำหรับ Alert

```sql
-- AWS IoT Rule: Temperature High Alert
-- ถ้า temperature > 80°C → ส่ง SNS notification

SELECT 
    deviceId,
    timestamp,
    process.temperature.value AS temperature,
    location.plant AS plant
FROM 'plant1/+/+/telemetry'
WHERE process.temperature.value > 80.0

-- Action: Publish to SNS Topic → Email/SMS to operator
```

---

## Lab Project 49: Complete IIoT Implementation

### โจทย์
สร้าง IIoT System สำหรับ Packaging Line Monitoring

### Architecture ที่ต้องออกแบบ
```
Packaging Line PLC → OPC-UA → Node-RED → MQTT → AWS IoT Core
                                            ↓
                                     AWS Lambda (alerts)
                                            ↓
                                     DynamoDB (storage)
                                            ↓
                                     Grafana Cloud (visualization)
```

### KPIs ที่ต้องติดตาม
1. Machine Speed (ppm - parts per minute)
2. OEE (Overall Equipment Effectiveness)
3. Downtime Causes (Pareto analysis)
4. Energy Consumption (kWh per 1000 parts)
5. Quality Rate (% pass)

### งานที่ต้องทำ
1. กำหนด OPC-UA namespace และ node structure
2. เขียน Node-RED Flow: S7-1500 → MQTT → AWS
3. Configure AWS IoT Core (certificates, policies, rules)
4. สร้าง Grafana Dashboard
5. Implement Alerting (SMS/Line Notify เมื่อ KPI ต่ำกว่า target)
6. ทำ Cybersecurity Assessment (IEC 62443)

---

## สรุป Part 49

| Technology | ใช้สำหรับ | ตัวอย่าง |
|-----------|---------|---------|
| Node-RED | Protocol translation, edge processing | S7→MQTT gateway |
| MQTT | Lightweight IoT messaging | Telemetry data |
| OPC-UA | Secure machine communication | PLC → cloud |
| AWS IoT | Cloud platform | Global monitoring |
| Azure IoT | Microsoft ecosystem | Enterprise integration |
| Digital Twin | Simulation, remote monitoring | Siemens MindSphere |
| MindSphere | Siemens cloud platform | Machine analytics |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 48: Vision System Integration](part48_vision_system.md) |
| ► ถัดไป | [Part 50: OPC UA](part50_opc_ua.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
