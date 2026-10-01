# Part 54: PLC ถึง Cloud Integration (Steps 1061-1080)

## บทนำ: IIoT Architecture

Industrial Internet of Things (IIoT) เชื่อมต่อระบบ automation เข้ากับ Cloud เพื่อ:
- **Analytics**: วิเคราะห์ข้อมูลจาก PLC จำนวนมากรวมกัน
- **Predictive Maintenance**: ทำนายความเสียหายก่อนเกิด
- **Remote Monitoring**: ดูข้อมูลได้ทุกที่ทุกเวลา
- **Digital Twin**: จำลองระบบจริงในดิจิทัล
- **Business Intelligence**: ผนวกข้อมูล OT กับ ERP

---

## Step 1061: Cloud Platform Overview

### AWS, Azure, Google Cloud สำหรับ IIoT

```
┌────────────────────────────────────────────────────────────────────────┐
│              CLOUD PLATFORMS สำหรับ INDUSTRIAL IoT                      │
├───────────────────┬────────────────────────────────────────────────────┤
│ AWS IoT           │ ข้อดี                                              │
│                   │ - AWS IoT Core: MQTT broker                        │
│                   │ - AWS IoT Greengrass: Edge computing               │
│                   │ - AWS IoT SiteWise: Industrial equipment data      │
│                   │ - AWS Timestream: Time-series database             │
│                   │ - Amazon Kinesis: Real-time data streaming         │
│                   │ - ใหญ่ที่สุด, ecosystem กว้าง                     │
├───────────────────┼────────────────────────────────────────────────────┤
│ Azure IoT         │ ข้อดี                                              │
│                   │ - Azure IoT Hub: Device management + messaging     │
│                   │ - Azure IoT Edge: Edge module deployment           │
│                   │ - Azure Digital Twins: Digital twin service        │
│                   │ - Azure Time Series Insights: TS analytics        │
│                   │ - Siemens integration (MindSphere on Azure)        │
│                   │ - ดีสำหรับ enterprise + Microsoft stack           │
├───────────────────┼────────────────────────────────────────────────────┤
│ Google Cloud IoT  │ ข้อดี                                              │
│                   │ - Cloud IoT Core: MQTT/HTTP                        │
│                   │ - BigQuery: Massive analytics                      │
│                   │ - Pub/Sub: Messaging                               │
│                   │ - Vertex AI: Machine learning                      │
│                   │ - ดีสำหรับ ML/AI workloads                        │
├───────────────────┼────────────────────────────────────────────────────┤
│ Siemens MindSphere│ ข้อดี                                              │
│                   │ - Industrial-focused                               │
│                   │ - Native Siemens PLC integration                   │
│                   │ - MindConnect Nano: Edge device                   │
│                   │ - Built-in industrial analytics                    │
│                   │ - ดีสำหรับ Siemens ecosystem                      │
├───────────────────┼────────────────────────────────────────────────────┤
│ Rockwell Plex     │ ข้อดี                                              │
│                   │ - Allen-Bradley native integration                 │
│                   │ - MES in cloud                                     │
│                   │ - FactoryTalk cloud                               │
│                   │ - ดีสำหรับ Rockwell ecosystem                     │
└───────────────────┴────────────────────────────────────────────────────┘
```

---

## Step 1062: IoT Gateway Devices

### Industrial Gateway Products

```
IoT Gateway: อุปกรณ์ที่แปลง OT protocol → Cloud protocol

┌────────────────────────────────────────────────────────────────────────┐
│                        IOT GATEWAY PRODUCTS                             │
├──────────────────┬─────────────────────────────────────────────────────┤
│ Tosibox Lock     │ - Zero-config VPN + data collection                 │
│                  │ - Modbus, PROFINET → MQTT                           │
│                  │ - Finnish, reliable for harsh env                   │
├──────────────────┼─────────────────────────────────────────────────────┤
│ Ewon Flexy       │ - Protocol conversion engine                        │
│ (by HMS)         │ - 200+ protocols supported                          │
│                  │ - Talk2M cloud platform                             │
│                  │ - Very flexible scripting                           │
├──────────────────┼─────────────────────────────────────────────────────┤
│ Siemens          │ - Linux-based gateway                               │
│ IoT2050          │ - Node-RED built-in                                 │
│                  │ - TIA Portal integration                            │
│                  │ - MindConnect built-in                              │
│                  │ - Open: install own software                        │
├──────────────────┼─────────────────────────────────────────────────────┤
│ Moxa ThingsPro   │ - Modbus, EtherNet/IP, PROFINET                    │
│                  │ - Azure/AWS/Google certified                        │
│                  │ - Industrial grade, DIN rail                        │
├──────────────────┼─────────────────────────────────────────────────────┤
│ Eurotech          │ - Kura IoT framework (Java/OSGi)                  │
│ ReliaGATE        │ - ESF (Eclipse Sparkplug B)                        │
│                  │ - Enterprise grade                                  │
├──────────────────┼─────────────────────────────────────────────────────┤
│ Raspberry Pi     │ - Low cost, flexible                                │
│ (Industrial)     │ - Not recommended for production OT                 │
│                  │ - OK for pilot/proof of concept                     │
└──────────────────┴─────────────────────────────────────────────────────┘
```

---

## Step 1063: Protocol Translation - Modbus to MQTT

### Modbus TCP → MQTT Bridge

```
DATA FLOW:
PLC (Modbus) → Gateway (Node-RED/Python) → MQTT Broker → Cloud

MQTT Topic Structure สำหรับ Industrial:
plant/site/area/unit/device/measurement

ตัวอย่าง:
plant/bangkok_refinery/unit1/reactor_R101/temp_outlet
plant/bangkok_refinery/unit1/reactor_R101/pressure_inlet
plant/bangkok_refinery/unit2/pump_P201/current_phase_a
```

### Node-RED Flow สำหรับ Modbus to MQTT

```javascript
// Node-RED flow definition (JSON export ย่อ)
// ─────────────────────────────────────────────────────────────

// Node 1: Modbus Read (every 5 seconds)
{
  "type": "modbus-read",
  "name": "Read PLC Data",
  "topic": "PLC_Read",
  "showStatusActivities": true,
  "logIOActivities": false,
  "unitid": 1,           // Modbus Unit ID
  "dataType": "HoldingRegister",
  "adr": 40001,         // Start address (HR 1)
  "quantity": 10,        // จำนวน registers
  "rate": 5,             // ทุก 5 หน่วย
  "rateUnit": "s"
}

// Node 2: Function - แปลง raw data เป็น engineering units
// ─────────────────────────────────────────────────────────────
// function parseModbusData() {
//     var registers = msg.payload;
//     
//     // Register map:
//     // HR1 (40001): Temperature × 10 (e.g., 252 = 25.2°C)
//     // HR2 (40002): Pressure × 100 (e.g., 10132 = 101.32 kPa)
//     // HR3 (40003): Level × 10 (e.g., 752 = 75.2%)
//     // HR4 (40004): Flow × 10 (e.g., 1234 = 123.4 m3/h)
//     
//     var data = {
//         timestamp: new Date().toISOString(),
//         device: "PLC_Unit1",
//         tags: {
//             temperature:  registers[0] / 10.0,   // °C
//             pressure:     registers[1] / 100.0,  // kPa
//             level:        registers[2] / 10.0,   // %
//             flow:         registers[3] / 10.0    // m3/h
//         },
//         unit: {
//             temperature: "°C",
//             pressure: "kPa",
//             level: "%",
//             flow: "m3/h"
//         }
//     };
//     
//     msg.payload = JSON.stringify(data);
//     msg.topic = "plant/factoryA/unit1/plc001/process_data";
//     return msg;
// }

// Node 3: MQTT Output
{
  "type": "mqtt out",
  "broker": "mqtt-broker-config",
  "topic": "",          // จะมาจาก msg.topic
  "qos": "1",           // At least once delivery
  "retain": false
}
```

### Python Gateway Script

```python
#!/usr/bin/env python3
# OT Gateway: Modbus TCP → MQTT → AWS IoT
# ─────────────────────────────────────────────────────────────

import json
import time
import logging
from datetime import datetime, timezone
from pymodbus.client import ModbusTcpClient
import paho.mqtt.client as mqtt
import ssl

# Configuration
PLC_HOST = "192.168.30.10"
PLC_PORT = 502
MQTT_HOST = "your-endpoint.iot.ap-southeast-1.amazonaws.com"
MQTT_PORT = 8883
MQTT_TOPIC_BASE = "plant/factory_a/unit1"
POLL_INTERVAL = 5  # seconds

# Register Map
REGISTER_MAP = {
    "temperature_outlet":  {"addr": 0, "scale": 0.1, "unit": "°C"},
    "pressure_vessel":     {"addr": 1, "scale": 0.01, "unit": "bar"},
    "level_tank":          {"addr": 2, "scale": 0.1, "unit": "%"},
    "flow_product":        {"addr": 3, "scale": 0.1, "unit": "m3/h"},
    "motor_current":       {"addr": 4, "scale": 0.01, "unit": "A"},
    "pump_speed":          {"addr": 5, "scale": 1.0, "unit": "RPM"},
}

def setup_mqtt():
    """ตั้งค่า MQTT client พร้อม TLS"""
    client = mqtt.Client(client_id="gateway_unit1")
    
    # TLS สำหรับ AWS IoT
    client.tls_set(
        ca_certs="/certs/AmazonRootCA1.pem",
        certfile="/certs/device_cert.pem",
        keyfile="/certs/device_private.key",
        tls_version=ssl.PROTOCOL_TLS
    )
    
    client.connect(MQTT_HOST, MQTT_PORT, keepalive=60)
    client.loop_start()
    return client

def read_plc_data(modbus_client):
    """อ่านข้อมูลจาก PLC"""
    result = modbus_client.read_holding_registers(
        address=0,
        count=len(REGISTER_MAP),
        slave=1
    )
    
    if result.isError():
        logging.error(f"Modbus read error: {result}")
        return None
    
    data = {}
    for tag_name, config in REGISTER_MAP.items():
        raw_value = result.registers[config["addr"]]
        data[tag_name] = {
            "value": raw_value * config["scale"],
            "unit": config["unit"],
            "quality": "GOOD"
        }
    
    return data

def main():
    logging.basicConfig(level=logging.INFO)
    
    # เชื่อมต่อ Modbus
    modbus = ModbusTcpClient(host=PLC_HOST, port=PLC_PORT)
    if not modbus.connect():
        logging.error("ไม่สามารถเชื่อมต่อ PLC ได้")
        return
    
    # เชื่อมต่อ MQTT
    mqtt_client = setup_mqtt()
    
    logging.info("Gateway เริ่มทำงาน - กำลัง poll PLC...")
    
    while True:
        try:
            # อ่านข้อมูล PLC
            plc_data = read_plc_data(modbus)
            
            if plc_data:
                # สร้าง payload
                payload = {
                    "timestamp": datetime.now(timezone.utc).isoformat(),
                    "device_id": "PLC_UNIT1",
                    "plant": "Factory_A",
                    "tags": plc_data
                }
                
                # ส่งไปยัง MQTT (JSON)
                topic = f"{MQTT_TOPIC_BASE}/plc001/telemetry"
                mqtt_client.publish(
                    topic=topic,
                    payload=json.dumps(payload),
                    qos=1
                )
                
                logging.info(f"ส่งข้อมูล: Temp={plc_data['temperature_outlet']['value']}°C")
            
            time.sleep(POLL_INTERVAL)
            
        except KeyboardInterrupt:
            break
        except Exception as e:
            logging.error(f"Error: {e}")
            time.sleep(5)
    
    modbus.close()
    mqtt_client.loop_stop()

if __name__ == "__main__":
    main()
```

---

## Step 1064: MQTT Sparkplug B Protocol

### Sparkplug B - Industry Standard สำหรับ IIoT

```
Sparkplug B (Eclipse Foundation):
──────────────────────────────────────────────────────────────

ทำไมต้อง Sparkplug B?
├── MQTT standard ไม่กำหนด payload format
├── Sparkplug กำหนด: Topic namespace + Payload encoding
├── ใช้ Google Protobuf (binary, compact, fast)
├── Birth/Death certificates (device online/offline tracking)
└── Standard ที่ industry adopt กว้างขึ้น

Topic Structure:
spBv1.0/[group_id]/[message_type]/[edge_node_id]/[device_id]

Message Types:
├── NBIRTH  - Edge node birth (online announce)
├── NDEATH  - Edge node death (offline)
├── NDATA   - Edge node data
├── DBIRTH  - Device birth
├── DDEATH  - Device death
└── DDATA   - Device data

ตัวอย่าง Topic:
spBv1.0/Bangkok_Refinery/DBIRTH/Gateway_Unit1/PLC_Reactor_R101
spBv1.0/Bangkok_Refinery/DDATA/Gateway_Unit1/PLC_Reactor_R101

Payload (JSON-like representation):
{
  "timestamp": 1734000000000,
  "metrics": [
    {"name": "temperature", "value": 152.3, "type": "Float"},
    {"name": "pressure", "value": 5.2, "type": "Float"},
    {"name": "status", "value": true, "type": "Boolean"}
  ],
  "seq": 42
}
```

---

## Step 1065: Time Series Database

### InfluxDB และ TimescaleDB

```
Time Series Database (TSDB) สำหรับ OT Data:
════════════════════════════════════════════════

ทำไมไม่ใช้ SQL Database ทั่วไป?
├── ข้อมูล OT: timestamp + value + tag (simple but massive)
├── 100 tags × 1 scan/second = 8.64 million points/day
├── Relational DB: poor performance for time-range queries
└── TSDB: optimized for timestamp-based queries

INFLUXDB (OSS/Cloud):
──────────────────────────────────────────────────────────────
Concepts:
├── Bucket = Database (with retention policy)
├── Measurement = Table name (e.g., "process_data")
├── Tags = Indexed metadata (device_id, location, unit)
├── Fields = Actual values (temperature, pressure)
└── Timestamp = nanosecond precision

Write example (InfluxDB Line Protocol):
process_data,device=PLC_R101,location=Unit1 temperature=152.3,pressure=5.2 1734000000000000000

Query example (Flux language):
from(bucket: "plant_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "process_data")
  |> filter(fn: (r) => r["device"] == "PLC_R101")
  |> filter(fn: (r) => r["_field"] == "temperature")
  |> mean()

TIMESCALEDB (PostgreSQL extension):
──────────────────────────────────────────────────────────────
├── PostgreSQL compatible (familiar SQL)
├── Hypertables = time-partitioned tables
├── Continuous aggregates = pre-computed summaries
├── Good for complex joins with relational data
└── ง่ายกว่า InfluxDB สำหรับคนที่รู้ SQL อยู่แล้ว

SQL Query:
SELECT time_bucket('5 minutes', time) AS interval,
       avg(temperature) AS avg_temp,
       max(pressure) AS max_pressure
FROM process_data
WHERE device_id = 'PLC_R101'
  AND time > NOW() - INTERVAL '24 hours'
GROUP BY interval
ORDER BY interval;
```

---

## Step 1066: Grafana Visualization

### Grafana Dashboard สำหรับ OT Data

```
Grafana Setup:
──────────────────────────────────────────────────────────────

Data Sources (support ได้):
├── InfluxDB (native)
├── TimescaleDB (PostgreSQL)
├── OSIsoft PI (via REST)
├── OPC UA (via grafana-opcua-datasource)
├── Prometheus (metrics)
└── Microsoft Azure Monitor

Dashboard Design สำหรับ Process Plant:
──────────────────────────────────────────────────────────────

OVERVIEW DASHBOARD:
┌─────────────────────────────────────────────────────────────┐
│  PLANT STATUS - LAST UPDATE: 2024-01-15 14:32:05           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ UNIT 1       │  │ UNIT 2       │  │ UTILITIES        │ │
│  │ 🟢 RUNNING   │  │ 🟡 STARTUP   │  │ 🟢 NORMAL        │ │
│  │ Prod: 95%   │  │ Prod: 60%   │  │ Steam: OK        │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
│                                                              │
│  CRITICAL PARAMETERS:                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Reactor R101 Temperature  ████████████░░ 152.3°C    │  │
│  │ (SP: 155°C, Alarm: 165°C)                            │  │
│  │                                                       │  │
│  │ Pressure P101   ██████░░░░░░  3.2 bar                │  │
│  │ (SP: 3.5 bar)                                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  TREND (Last 8 hours):                                       │
│  [Grafana Time Series Panel - Temperature R101]              │
│  [Grafana Time Series Panel - Production Rate]               │
└─────────────────────────────────────────────────────────────┘

Grafana Alerting:
├── Threshold alerts (e.g., temp > 160°C)
├── Rate-of-change alerts (เปลี่ยนเร็วผิดปกติ)
├── No data alerts (device offline)
├── Notification channels: Email, SMS, LINE, PagerDuty
└── Alert history log
```

---

## Step 1067: Edge Computing

### Edge vs Cloud Processing

```
Edge Computing: ประมวลผลที่ gateway/local ก่อนส่ง cloud

ข้อดี Edge Processing:
├── Latency ต่ำ (ms) สำหรับ critical decisions
├── Bandwidth ลด (ส่งเฉพาะ summary/events)
├── ทำงานได้ถ้า cloud disconnected
└── Data privacy (sensitive data ไม่ต้อง leave site)

Edge Tasks ที่เหมาะ:
├── Pre-processing: scale, filter, aggregate
├── Anomaly detection: rule-based alerts
├── Event generation: state changes only
├── Local dashboard: operator view without cloud
└── Protocol conversion: Modbus → OPC UA → MQTT

Cloud Tasks ที่เหมาะ:
├── Long-term storage: years of historical data
├── ML training: complex models
├── Cross-site analytics: compare multiple plants
├── ERP integration: enterprise systems
└── Remote access: multiple users worldwide

Edge Computing Stack:
──────────────────────────────────────────────────────────────
Hardware:
├── Siemens IoT2050 (Intel Atom, Linux)
├── AWS Greengrass compatible devices
├── Azure IoT Edge compatible devices
└── Industrial PC (Beckhoff, Moxa)

Software:
├── AWS IoT Greengrass
├── Azure IoT Edge
├── Node-RED (visual programming)
├── Ignition Edge (Inductive Automation)
└── Kepware OPC Server
```

### Edge Processing ด้วย Node-RED

```javascript
// Node-RED Edge Processing Flow
// ─────────────────────────────────────────────────────────────

// FUNCTION NODE: Statistical Aggregation (ทุก 60 วินาที)
function aggregateData(msg) {
    var context = flow.get('buffer') || [];
    
    // เก็บข้อมูลใน buffer
    context.push({
        time: Date.now(),
        temp: msg.payload.temperature,
        pressure: msg.payload.pressure
    });
    
    // เก็บแค่ 60 วินาทีล่าสุด
    var cutoff = Date.now() - 60000;
    context = context.filter(d => d.time > cutoff);
    flow.set('buffer', context);
    
    // ถ้าครบ 60 วินาที ส่ง aggregate
    if (context.length >= 10) {
        var temps = context.map(d => d.temp);
        var pressures = context.map(d => d.pressure);
        
        msg.payload = {
            timestamp: new Date().toISOString(),
            temperature: {
                avg: average(temps),
                min: Math.min(...temps),
                max: Math.max(...temps),
                std: stdDev(temps)
            },
            pressure: {
                avg: average(pressures),
                min: Math.min(...pressures),
                max: Math.max(...pressures)
            },
            sample_count: context.length
        };
        
        return msg;  // ส่งต่อ
    }
    
    return null;  // ยังไม่ส่ง
}

// FUNCTION NODE: Anomaly Detection
function detectAnomaly(msg) {
    var temp = msg.payload.temperature.avg;
    var tempHistory = flow.get('temp_history') || [];
    
    // เก็บค่าเฉลี่ยล่าสุด 10 นาที
    tempHistory.push(temp);
    if (tempHistory.length > 10) tempHistory.shift();
    flow.set('temp_history', tempHistory);
    
    // ตรวจสอบ:
    // 1. Hard limit
    if (temp > 165) {
        return { payload: { alert: "HIGH_TEMP_CRITICAL", value: temp } };
    }
    
    // 2. Rate of change (เปลี่ยนเร็วผิดปกติ)
    if (tempHistory.length >= 3) {
        var rate = temp - tempHistory[tempHistory.length - 3];
        if (Math.abs(rate) > 10) {  // เปลี่ยน > 10°C ใน 3 นาที
            return { payload: { alert: "RAPID_TEMP_CHANGE", value: temp, rate: rate } };
        }
    }
    
    return null;  // ปกติ ไม่ส่ง alert
}
```

---

## Step 1068: OPC UA สำหรับ Cloud Integration

### OPC UA - The Universal Protocol

```
OPC UA (Unified Architecture):
─────────────────────────────────────────────────────────────

ทำไม OPC UA ถึงดีสำหรับ IIoT?
├── Built-in security (TLS, certificates, RBAC)
├── Platform independent (Linux, Windows, embedded)
├── Transport agnostic: TCP, HTTPS, WebSocket, MQTT
├── Information model: ไม่ใช่แค่ data แต่ meaning ด้วย
└── ทุก PLC vendor support: Siemens, Rockwell, Mitsubishi, etc.

OPC UA Transport Options:
├── opc.tcp://: Binary, port 4840 (default) - most efficient
├── https://: Web-friendly, port 443
└── opc.tcp over MQTT: MQTT transport (publish/subscribe)

OPC UA Node Structure ตัวอย่าง:
Objects/
├── Plant/
│   ├── Unit1/
│   │   ├── Reactor_R101/
│   │   │   ├── Temperature (Variable: Float, °C)
│   │   │   ├── Pressure (Variable: Float, bar)
│   │   │   ├── Level (Variable: Float, %)
│   │   │   └── Status (Variable: String)
│   │   └── Pump_P101/
│   │       ├── Speed (Variable: Float, RPM)
│   │       └── Current (Variable: Float, A)
│   └── Unit2/
└── Utilities/
```

### OPC UA Client ใน Python

```python
# Python OPC UA Client (asyncua library)
# ─────────────────────────────────────────────────────────────

import asyncio
from asyncua import Client, ua
import json

OPC_UA_URL = "opc.tcp://192.168.30.10:4840"

# Node IDs สำหรับ read (จาก OPC UA address space)
NODE_MAP = {
    "temperature": "ns=2;i=1001",    # Reactor R101 Temperature
    "pressure":    "ns=2;i=1002",    # Reactor R101 Pressure
    "level":       "ns=2;i=1003",    # Tank T101 Level
    "pump_speed":  "ns=2;i=2001",    # Pump P101 Speed
}

async def read_opc_data():
    async with Client(url=OPC_UA_URL) as client:
        
        # Security (sign and encrypt)
        await client.set_security_string(
            "Basic256Sha256,SignAndEncrypt,"
            "/certs/client_cert.pem,"
            "/certs/client_key.pem"
        )
        
        # อ่านทุก nodes พร้อมกัน
        nodes = [client.get_node(nid) for nid in NODE_MAP.values()]
        values = await asyncio.gather(*[n.read_value() for n in nodes])
        
        data = {}
        for tag, value in zip(NODE_MAP.keys(), values):
            data[tag] = value
        
        return data

async def subscribe_to_changes():
    """Subscribe รับ notification เมื่อค่าเปลี่ยน"""
    
    class DataChangeHandler:
        def datachange_notification(self, node, val, data):
            print(f"Node: {node}, Value changed to: {val}")
            # ส่งไปยัง MQTT หรือ database ที่นี่
    
    async with Client(url=OPC_UA_URL) as client:
        handler = DataChangeHandler()
        subscription = await client.create_subscription(500, handler)  # 500ms
        
        # Subscribe ทุก tags
        nodes = [client.get_node(nid) for nid in NODE_MAP.values()]
        await subscription.subscribe_data_change(nodes)
        
        # รอ (infinite loop)
        await asyncio.sleep(3600)

asyncio.run(subscribe_to_changes())
```

---

## Step 1069: Data Pipeline Design

### Complete Data Pipeline Architecture

```
DATA PIPELINE สำหรับ LARGE PLANT:
──────────────────────────────────────────────────────────────

LAYER 1: DATA ACQUISITION (Edge)
──────────────────────────────────────────────────────────────
  PLCs → OPC UA Server → OPC UA Collector → Local Buffer
  
  Tools:
  ├── Kepware KepServerEX: OPC server สำหรับ 200+ protocols
  ├── Ignition Edge: Lightweight SCADA + historian
  └── Custom Python/Node-RED: flexible

LAYER 2: EDGE PROCESSING
──────────────────────────────────────────────────────────────
  Local Buffer → Edge Processing (filter, aggregate, validate)
               → Local Historian (short-term: 7 days)
               → Anomaly Detection → Local Alerts
  
  Edge Local Historian: OSIsoft PI (Edge) or InfluxDB

LAYER 3: DATA TRANSMISSION
──────────────────────────────────────────────────────────────
  Edge → (MQTT/OPC UA) → Gateway → (HTTPS/MQTT) → Cloud
  
  Bandwidth optimization:
  ├── Change-of-value (CoV): ส่งเฉพาะเมื่อเปลี่ยน
  ├── Compression: Zlib/LZ4
  ├── Aggregation: ส่งค่าเฉลี่ย 1 นาที แทน raw 1 วินาที
  └── Prioritization: Critical alarms first

LAYER 4: CLOUD INGESTION
──────────────────────────────────────────────────────────────
  Cloud Ingestion → Message Queue → Stream Processor → TSDB
  
  AWS Stack:
  MQTT → IoT Core → Kinesis Data Streams → Lambda → Timestream
  
  Azure Stack:
  MQTT → IoT Hub → Event Hubs → Stream Analytics → ADX (Kusto)
  
  Open Source Stack:
  MQTT → Mosquitto → Apache Kafka → Spark Streaming → InfluxDB

LAYER 5: ANALYTICS & VISUALIZATION
──────────────────────────────────────────────────────────────
  TSDB → Grafana (real-time dashboard)
  TSDB → Jupyter/Python (analysis)
  TSDB → Power BI (business reporting)
  TSDB → ML Platform (predictive models)
```

---

## Step 1070: Latency Considerations

### Real-time Requirements vs Cloud Latency

```
Latency Analysis สำหรับ IIoT:
══════════════════════════════════════════════════════════════

CONTROL LOOP REQUIREMENTS (ห้ามใช้ Cloud):
├── PROFINET IRT: 31.25μs - 4ms
├── EtherNet/IP motion: 1-10ms
├── PID control: 100ms - 1s
└── ← ต้องทำที่ PLC โดยตรง เท่านั้น

SUPERVISORY REQUIREMENTS (Edge OK):
├── SCADA screen update: 1-5 seconds
├── Alarm notification: < 1 second
├── Recipe management: seconds
└── ← Edge processing เพียงพอ

ANALYTICS/REPORTING (Cloud OK):
├── Trend data: 5-60 seconds delay OK
├── Daily report: hours OK
├── Predictive maintenance: hours to days
└── ← Cloud processing เหมาะสม

Cloud Round-Trip Latency:
│ Operation                    │ Typical Latency     │
├─────────────────────────────┼─────────────────────┤
│ Write to IoT Hub (Azure)    │ 50-200ms            │
│ Read from Timestream (AWS)  │ 100-500ms           │
│ Grafana dashboard load      │ 500ms - 2s          │
│ ML prediction API call      │ 100-1000ms          │
│ ERP query (SAP API)         │ 500ms - 5s          │

→ ข้อสรุป: Cloud ไม่เหมาะสำหรับ real-time control
           แต่เหมาะสำหรับ analytics, reporting, ML
```

---

## Step 1071: MQTT Configuration ใน Siemens S7-1500

### S7-1500 MQTT Client

```
// Siemens S7-1500 TF (Technology Functions) - MQTT Client
// ต้องใช้: Siemens S7-1500 TF MQTT Client (Software Controller หรือ TF)
// หรือ: ใช้ LMQTT library ใน TIA Portal

// Alternative (ง่ายกว่า): ส่งผ่าน OPC UA → Gateway → MQTT

// วิธีที่นิยม: ใช้ OPC UA ออกจาก PLC แล้วให้ Gateway แปลงเป็น MQTT

// OPC UA Configuration ใน TIA Portal:
// PLC Properties → OPC UA → Server → Enable Server
// Security Mode: SignAndEncrypt
// Authentication: Username and password

// ─────────────────────────────────────────────────────────────
// DATA BLOCK: DB_CloudData (Shared with OPC UA)
// ─────────────────────────────────────────────────────────────
DATA_BLOCK DB_CloudData
VAR
    // Process Data (readable by OPC UA)
    Reactor_R101: STRUCT
        Temperature: REAL;      // °C
        Pressure:    REAL;      // bar
        Level:       REAL;      // %
        FlowIn:      REAL;      // m3/h
        FlowOut:     REAL;      // m3/h
        Status:      INT;       // 0=Stop, 1=Run, 2=Alarm
    END_STRUCT;
    
    // Setpoints (writable by authorized OPC UA clients)
    Setpoints: STRUCT
        Temp_SP:     REAL := 120.0;   // °C
        Pressure_SP: REAL := 3.5;     // bar
        Level_SP:    REAL := 50.0;    // %
    END_STRUCT;
    
    // Timestamps
    LastUpdate:  DATE_AND_TIME;
    
END_VAR

// ─────────────────────────────────────────────────────────────
// NETWORK: Update DB_CloudData จาก process I/O
// ─────────────────────────────────────────────────────────────
// DB_CloudData.Reactor_R101.Temperature := AI_TempScaled;
// DB_CloudData.Reactor_R101.Pressure    := AI_PressureScaled;
// DB_CloudData.Reactor_R101.Level       := AI_LevelScaled;
// DB_CloudData.LastUpdate               := DT#NOW();
```

---

## Step 1072: AWS IoT SiteWise Integration

### SiteWise สำหรับ Industrial Data

```
AWS IoT SiteWise - Key Concepts:
──────────────────────────────────────────────────────────────

Asset Models:
├── Define ประเภทของ assets (e.g., "Reactor", "Pump")
├── Properties: measurements, transforms, metrics
└── Hierarchies: Plant → Unit → Equipment

Asset:
├── Instance ของ asset model (e.g., "Reactor R101")
├── Maps ไปยัง actual data source
└── Inherits model's calculation logic

Property Types:
├── Measurement: Raw data from sensor/PLC
├── Transform: คำนวณจาก measurement (e.g., F to C conversion)
├── Metric: Aggregate over time window (avg, min, max, sum)
└── Attribute: Static data (e.g., nameplate capacity)

SiteWise Gateway:
├── Software ที่ run บน edge device
├── OPC UA collector → SiteWise Cloud
└── Buffers data เมื่อ internet ขาด

สร้าง Asset Model ตัวอย่าง (JSON):
{
  "assetModelName": "ChemicalReactor",
  "assetModelProperties": [
    {
      "name": "Temperature",
      "dataType": "DOUBLE",
      "unit": "Celsius",
      "type": {
        "measurement": {}
      }
    },
    {
      "name": "Average Temperature (5min)",
      "dataType": "DOUBLE",
      "unit": "Celsius",
      "type": {
        "metric": {
          "expression": "avg(temperature)",
          "variables": [{
            "name": "temperature",
            "value": { "propertyId": "temperature-id" }
          }],
          "window": { "tumbling": { "interval": "5m" } }
        }
      }
    }
  ]
}
```

---

## Step 1073: Predictive Maintenance Architecture

### Predictive Maintenance ด้วย ML

```
Predictive Maintenance System Design:
──────────────────────────────────────────────────────────────

DATA COLLECTION:
PLC → Vibration data (accelerometer on motor)
PLC → Current signature (เพื่อ electrical analysis)
PLC → Temperature trend
PLC → Process parameters (suction/discharge pressure)

FEATURE EXTRACTION (Edge/Cloud):
├── RMS vibration (Vrms)
├── Peak vibration (Vpeak)
├── Frequency spectrum (FFT analysis)
├── Current harmonic distortion
├── Motor efficiency trend
└── Bearing temperature deviation

ML MODELS:
├── Anomaly detection: Isolation Forest, Autoencoder
├── Remaining Useful Life (RUL): LSTM network
├── Fault classification: SVM, Random Forest
└── Threshold: 3-sigma rule (simple but effective)

ALERT LOGIC:
Normal → Alert threshold → Maintenance warning → Critical
         (5% deviation)  (15% deviation)      (30% deviation)

                  ↓                    ↓
            Plan maintenance     Emergency stop

CASE STUDY: Centrifugal Pump Bearing Failure
Timeline:
Week 1: Vibration normal 0.5mm/s
Week 3: Vibration rises to 0.8mm/s → ML flags anomaly
Week 5: Vibration 1.2mm/s → Maintenance warning sent
Week 6: Bearing replaced during planned shutdown
         ← AVOIDED unplanned failure (cost: 500,000 บาท)
```

---

## Step 1074: Data Security สำหรับ IIoT

### Securing Cloud-Connected OT

```
IIoT Security Architecture:
══════════════════════════════════════════════════════════════

DEVICE LEVEL:
├── Unique device certificate (X.509)
├── Hardware Security Module (HSM) เก็บ private key
├── Secure boot
└── Encrypted storage

COMMUNICATION LEVEL:
├── TLS 1.2/1.3 minimum
├── Certificate-based authentication
├── No anonymous connections
└── Message signing (prevent tampering)

CLOUD LEVEL:
├── IoT device policies (AWS IoT Policy / Azure Device Policy)
├── Minimal permissions (publish only to own topic)
├── Data encryption at rest (AES-256)
└── Audit logging

ตัวอย่าง AWS IoT Policy (สำหรับ gateway device):
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": "arn:aws:iot:*:*:topic/plant/factory_a/unit1/${iot:ClientId}/*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "iot:Connection.Thing.IsAttached": "true"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "iot:Subscribe",
      "Resource": "arn:aws:iot:*:*:topicfilter/plant/factory_b/*"
    }
  ]
}
```

---

## Step 1075: Complete IIoT Architecture

### Reference Architecture สำหรับ Large Plant

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    COMPLETE IIOT REFERENCE ARCHITECTURE                   ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                            ║
║  PLANT (OT ZONE - air-gapped from internet)                               ║
║  ──────────────────────────────────────────                               ║
║  PLC1 (Siemens S7-1500) ─┐                                                ║
║  PLC2 (ControlLogix) ─────┤→ OPC UA Server (Kepware) → Edge Gateway      ║
║  PLC3 (Mitsubishi Q) ─────┘                            (Siemens IoT2050)  ║
║                                                              │             ║
║                     ┌─────────────────────────────────────────┐           ║
║                     │           EDGE GATEWAY                    │           ║
║                     │ ┌──────────┐ ┌──────────┐ ┌──────────┐ │           ║
║                     │ │ Node-RED │ │InfluxDB  │ │ Grafana  │ │           ║
║                     │ │(process) │ │(7-day    │ │(local    │ │           ║
║                     │ │         │ │ buffer)  │ │ view)   │ │           ║
║                     │ └──────────┘ └──────────┘ └──────────┘ │           ║
║                     │              │                           │           ║
║                     │ MQTT Publish (TLS, Certificate auth)    │           ║
║                     └─────────────────────────────────────────┘           ║
║                                    │                                       ║
║  DMZ ZONE                         │ MQTT/HTTPS                            ║
║  ─────────                        ▼                                       ║
║                     ┌─────────────────────────────────────────┐           ║
║                     │  MQTT BROKER (HiveMQ / Mosquitto)        │           ║
║                     │  - TLS termination                       │           ║
║                     │  - Authentication (certificates)         │           ║
║                     │  - Topic authorization                   │           ║
║                     └────────────────┬────────────────────────┘           ║
║                                      │                                     ║
║  CLOUD (AWS/Azure)                  │                                     ║
║  ─────────────────                  ▼                                     ║
║  ┌─────────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐ ║
║  │ IoT Core/   │→ │ Stream   │→ │ Time     │→ │  Analytics Layer      │ ║
║  │ IoT Hub     │  │ Processor│  │ Series DB│  │  - Grafana Dashboard  │ ║
║  │ (Ingest)    │  │ (Lambda/ │  │(Timestream│  │  - ML Models (Sage   │ ║
║  │             │  │ Functions│  │/InfluxDB) │  │    Maker/ML Studio)  │ ║
║  │             │  │)         │  │           │  │  - Business Reports  │ ║
║  └─────────────┘  └──────────┘  └──────────┘  └───────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## Step 1076: Remote Asset Monitoring

### Dashboard สำหรับ Multi-site Monitoring

```
Multi-Site Dashboard Design:
══════════════════════════════════════════════════════════════

TOP LEVEL: MAP VIEW
┌──────────────────────────────────────────────────────────────┐
│  GLOBAL PLANT STATUS                                         │
│                                                              │
│  🗺️  Thailand Map                                            │
│     ● Bangkok Site 1    [🟢 Running]  OEE: 88%             │
│     ● Bangkok Site 2    [🟢 Running]  OEE: 91%             │
│     ● Rayong Refinery   [🟡 Partial]  OEE: 72%             │
│     ● Chonburi Chemical [🔴 Shutdown] OEE: 0%              │
│                                                              │
│  SUMMARY KPIs:                                               │
│  Total Production Today: 1,234 MT                           │
│  Energy Consumption: 45,678 kWh                             │
│  Active Alarms: 3 (1 Critical, 2 High)                      │
└──────────────────────────────────────────────────────────────┘

DRILL-DOWN: SITE LEVEL
├── Process overview (Mimic diagram)
├── Key parameters (temp, pressure, flow, level)
├── Alarm summary
└── Production KPIs (rate, efficiency)

DRILL-DOWN: EQUIPMENT LEVEL
├── Real-time trend (last 8 hours)
├── Historical comparison (today vs yesterday)
├── Maintenance countdown
└── Equipment health score
```

---

## Step 1077: Data Historian Integration

### OSIsoft PI System Integration

```
OSIsoft PI System (ระบบ Historian ที่นิยมที่สุดใน Process Industry):
──────────────────────────────────────────────────────────────

PI System Components:
├── PI Data Archive: ฐานข้อมูล time series
├── PI AF (Asset Framework): โครงสร้าง asset และ calculation
├── PI Vision: Web-based visualization
├── PI Integrator: ส่งข้อมูลไปยัง cloud
└── PI Web API: REST API สำหรับ external access

PI Tag Creation ตัวอย่าง:
Tag: BKK_REFINERY_R101_TI_1001
├── Description: Reactor R101 Temperature Inlet
├── Engineering Unit: degC
├── Point Type: Float32
├── Scan: 5 seconds
└── Compression: Yes (deadband: 0.5°C)

PI Connector สำหรับ Cloud:
PI → PI Integrator for Business Analytics → InfluxDB/Azure ADX
หรือ
PI → PI Web API → Custom Code → Any cloud

ตัวอย่าง PI Web API Query (Python):
import requests

PI_SERVER = "https://your-pi-server/piwebapi"

response = requests.get(
    f"{PI_SERVER}/streams/F1DPxxxxxx/recorded",
    params={
        "startTime": "*-1d",
        "endTime": "*",
        "maxCount": 1000
    },
    auth=("pi_user", "password"),
    verify=False
)

data = response.json()["Items"]
for point in data[:5]:
    print(f"Time: {point['Timestamp']}, Value: {point['Value']}")
```

---

## Step 1078: Cost Analysis of IIoT Implementation

### ROI Analysis

```
IIoT Implementation Cost-Benefit Analysis:
══════════════════════════════════════════════════════════════

INVESTMENT COSTS:
──────────────────────────────────────────────────────────────
Hardware:
├── IoT Gateways (10x Ewon Flexy):      ฿ 500,000
├── Network upgrades:                    ฿ 300,000
├── Edge servers:                        ฿ 400,000
└── Total Hardware:                      ฿ 1,200,000

Software/Cloud:
├── Cloud subscription (AWS IoT):        ฿ 240,000/year
├── Grafana Enterprise:                  ฿ 120,000/year
├── InfluxDB Cloud:                      ฿ 80,000/year
└── Total Software:                      ฿ 440,000/year

Implementation:
├── Integration services:                ฿ 800,000
├── Training:                            ฿ 200,000
└── Total Implementation:                ฿ 1,000,000

Total Year 1 Investment: ฿ 2,640,000

BENEFITS (Annual):
──────────────────────────────────────────────────────────────
Predictive Maintenance:
├── Reduced unplanned downtime (10 hours/year saved × 300,000 บาท/hr)
└── Value: ฿ 3,000,000/year

Energy Optimization:
├── 5% energy reduction (baseline: 10M บาท/year)
└── Value: ฿ 500,000/year

Quality Improvement:
├── Reduced rework (1% improvement × 50M production)
└── Value: ฿ 500,000/year

Labor Efficiency:
├── Remote monitoring reduces site visits (2 FTE × 600,000)
└── Value: ฿ 1,200,000/year (partial)

Total Annual Benefit: ฿ 5,200,000/year
ROI: (5,200,000 - 440,000) / 2,640,000 = 179% Year 1
Payback: < 7 months
```

---

## Step 1079: Rockwell FactoryTalk Analytics

### FactoryTalk Cloud Integration

```
Rockwell FactoryTalk Analytics:
──────────────────────────────────────────────────────────────

FactoryTalk Platform Components:
├── FactoryTalk Optix: Modern HMI/SCADA
├── FactoryTalk Analytics: Cloud analytics
├── FactoryTalk DataMosaix: Data aggregation
└── Plex MES: Cloud-based MES

FactoryTalk Optix → Cloud Integration:
// Logix Designer + Studio 5000 → FactoryTalk Optix
// Optix → FactoryTalk Analytics (Azure-based)

// EtherNet/IP Tag Subscription ในตัว Studio 5000:
CLOUD_PUBLISH_CONFIG {
    Tags_to_Publish: [
        "Program:MainProgram.Reactor_Temp",
        "Program:MainProgram.Reactor_Pressure",
        "Program:MainProgram.Production_Count",
        "Program:MainProgram.Machine_State"
    ],
    Sample_Rate: 1000ms,
    Destination: "FactoryTalk_Analytics"
}

// FactoryTalk Analytics - Dashboards and Alerts
ANALYTICS_CONFIG {
    KPI_Rules: [
        { Name: "OEE",
          Formula: "Availability × Performance × Quality",
          Target: 85 },
        { Name: "MTBF",
          Formula: "OperatingTime / NumberOfFailures",
          Unit: "hours" }
    ],
    Alert_Rules: [
        { Tag: "Reactor_Temp",
          Condition: "ABOVE",
          Value: 165.0,
          Priority: "CRITICAL",
          Notify: ["sms:0812345678", "email:engineer@factory.com"] }
    ]
}
```

---

## Step 1080: Lab Project - Complete IIoT Implementation

### Lab: PLC to Cloud สำหรับ Water Pump Station

**โจทย์**: สถานีสูบน้ำ มี 3 Pump stations, ต้องการ monitoring จาก central office

**Equipment**:
- 3x Pump stations, แต่ละแห่งมี: S7-1200 PLC, 1 pump motor, flowmeter, pressure transmitters
- Central office อยู่ห่าง 50 km
- ต้องการ: real-time monitoring + แจ้งเตือน + daily report

```
LAB TASKS:
═════════════════════════════════════════════════════

Task 1: PLC Programming (25%)
─────────────────────────────────────────────────────
- Program S7-1200 เพื่ออ่าน: flow, pressure, current, run/stop status
- Scale analog inputs ให้ถูกต้อง (engineering units)
- Enable OPC UA server บน S7-1200
- Define OPC UA address space

Task 2: IoT Gateway Configuration (25%)
─────────────────────────────────────────────────────
- Configure Ewon Flexy หรือ Siemens IoT2050
- OPC UA client → อ่านข้อมูลจาก PLC
- Protocol conversion: OPC UA → MQTT
- Configure MQTT client (TLS, certificates)
- Test local data collection

Task 3: Cloud Setup (25%)
─────────────────────────────────────────────────────
- สร้าง AWS IoT หรือ Azure IoT Hub account
- Register device certificates
- Create InfluxDB bucket สำหรับ time series
- Configure data ingestion pipeline
- Create Grafana dashboard

Task 4: Alerting and Reporting (25%)
─────────────────────────────────────────────────────
- Grafana alerts: high pressure, low flow, motor overcurrent
- Alert channel: LINE Notify หรือ Email
- Create daily report template
- Test end-to-end: PLC → Cloud → Alert

DELIVERABLES:
├── PLC program (exported .ap15 หรือ .zap17)
├── Gateway configuration backup
├── Cloud architecture diagram
├── Grafana dashboard screenshots
├── Test report (5 alerts triggered and received)
└── Final presentation (10 นาที)

EVALUATION:
├── Data accuracy: PLC value = Cloud value ± 0.1%
├── Latency: Data appears in dashboard < 10 seconds
├── Reliability: 99%+ uptime during 24-hour test
├── Alerting: Alert received within 30 seconds
└── Documentation quality
```

---

## สรุป Part 54

| Component | Tool/Technology |
|-----------|-----------------|
| Edge Gateway | Ewon Flexy, Siemens IoT2050, Node-RED |
| Protocol | OPC UA → MQTT (Sparkplug B) |
| Cloud Ingestion | AWS IoT Core / Azure IoT Hub |
| Time Series DB | InfluxDB / TimescaleDB / AWS Timestream |
| Visualization | Grafana |
| Analytics | Python / Jupyter / Power BI |

---

## Navigation

- [← Part 53: Advanced PID and Process Control](./part53_advanced_pid.md)
- [→ Part 55: Digital Twin](./part55_digital_twin.md)
- [↑ สารบัญหลัก](../README.md)
