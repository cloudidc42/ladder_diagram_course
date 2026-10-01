# Part 82: Edge Computing สำหรับ PLC Engineer
## Steps 1621-1640 | ระดับ Capstone

---

## การนำทาง
- [← Part 81: Smart Manufacturing](part81_smart_manufacturing.md)
- [→ Part 83: AI/ML Integration กับ PLC](part83_ai_plc.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1621: Edge Computing คืออะไร?

### Edge vs Cloud vs Fog

```
┌─────────────────────────────────────────────────────────────────────┐
│              EDGE vs FOG vs CLOUD COMPUTING                         │
│                                                                     │
│  CLOUD (Microsoft Azure / AWS / Google Cloud)                       │
│  ├── Latency: 50-200ms                                              │
│  ├── Bandwidth: ขึ้นอยู่กับ Internet                               │
│  ├── Cost: Pay per use                                              │
│  └── Use: Long-term analytics, AI training, ERP                    │
│                                                                     │
│  FOG (Gateway Level - อาคาร/โรงงาน)                                │
│  ├── Latency: 5-20ms                                               │
│  ├── Location: Server Room ในโรงงาน                                │
│  └── Use: SCADA, MES, Historian                                    │
│                                                                     │
│  EDGE (Machine Level - ข้างเครื่องจักร)                            │
│  ├── Latency: <1ms - 5ms                                           │
│  ├── Location: ติดตั้งใน Panel ข้าง PLC                            │
│  └── Use: Real-time analytics, AI inference, Protocol conversion   │
└─────────────────────────────────────────────────────────────────────┘
```

### ทำไมต้องใช้ Edge Computing ในโรงงาน?

```
ปัญหาถ้าใช้แต่ Cloud:
1. Latency สูง: ไม่เหมาะกับการควบคุม Real-time
2. Bandwidth: ข้อมูล PLC มหาศาล (ส่งทั้งหมดขึ้น Cloud แพงมาก)
3. Reliability: ถ้า Internet ขาด Production หยุด
4. Security: ข้อมูล Production ละเอียดอ่อน

ข้อดีของ Edge:
1. Low Latency: ประมวลผลในพื้นที่ (<1ms)
2. Bandwidth Saving: กรองข้อมูลก่อนส่ง Cloud (Data Reduction 90%)
3. Offline Operation: ทำงานได้แม้ Internet ขาด
4. Data Privacy: ข้อมูลละเอียดอ่อนอยู่ใน Site
```

---

## Step 1622: Edge Hardware สำหรับโรงงาน

### Siemens Industrial PC (IPC)

```
Siemens SIMATIC IPC427E:
├── CPU: Intel Core i7 (6th Gen), 4 Cores
├── RAM: Up to 32 GB DDR4
├── Storage: SSD 512GB + HDD 2TB
├── OS: Windows 10 IoT / Linux
├── Interfaces: 4x USB 3.0, 4x LAN (Gigabit)
├── Operating Temp: 0-55°C
├── MTBF: > 50,000 hours
└── ราคา: ~150,000-300,000 บาท

Siemens SIMATIC IPC427E Fanless:
├── ไม่มีพัดลม (ฝุ่นน้อย, เสียงเงียบ)
├── IP65 Rating (กันฝุ่น, กันน้ำ)
└── เหมาะกับสภาพแวดล้อมโรงงานที่รุนแรง
```

### Raspberry Pi Industrial Grade

```
Compute Module 4 (CM4) ใน Industrial Carrier:
├── CPU: Broadcom BCM2711 (Quad-core ARM Cortex-A72)
├── RAM: Up to 8 GB
├── eMMC: 8-32 GB
├── Operating Temp: -40 to 85°C (Industrial CM4)
├── Carrier: Revolution Pi (Kunbus), reTerminal DM (Seeed)
└── ราคา: ~5,000-20,000 บาท

Kunbus RevPi Connect+:
├── Raspberry Pi Compute Module 4
├── DIN Rail Mounting
├── 2x RS232/485, 2x CAN Bus
├── PROFINET Slave module ต่อเพิ่มได้
└── Perfect สำหรับ Edge Gateway ราคาประหยัด
```

### Moxa Edge Computer

```
Moxa MC-1220 Series:
├── CPU: Intel Atom x5-E3930 (Fanless)
├── RAM: 8 GB
├── Storage: 64 GB eMMC + mSATA
├── OS: Debian 9 Linux
├── Interfaces: 4x LAN, 2x RS-232/422/485, 2x USB 3.0
├── Temperature: -40 to 70°C
├── Certifications: CE, FCC, UL, DNV, IECEx
└── ราคา: ~50,000-80,000 บาท
```

### Advantech Edge AI Box

```
Advantech MIC-770 V3:
├── CPU: Intel Core i7 / Xeon
├── GPU: NVIDIA RTX 3060 (Optional - สำหรับ AI Inference)
├── RAM: Up to 64 GB
├── M.2 NVMe SSD: Up to 4 TB
├── GPU Power: 200 TFLOPS (สำหรับ Deep Learning)
└── ราคา: ~80,000-200,000 บาท (ขึ้นอยู่กับ GPU)
```

---

## Step 1623: Edge Software Stack

### การติดตั้ง Node-RED บน Edge

```bash
# ติดตั้ง Node-RED บน Ubuntu/Debian
# สำหรับ Edge Computer หรือ Raspberry Pi

# อัพเดทระบบ
sudo apt update && sudo apt upgrade -y

# ติดตั้ง Node.js (LTS)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# ติดตั้ง Node-RED
sudo npm install -g --unsafe-perm node-red

# ติดตั้ง Nodes ที่จำเป็น
cd ~/.node-red
npm install node-red-contrib-opcua          # OPC UA สำหรับ PLC
npm install node-red-contrib-s7             # Siemens S7 Direct
npm install node-red-contrib-modbus         # Modbus TCP/RTU
npm install node-red-node-influxdb          # InfluxDB
npm install node-red-dashboard              # Dashboard UI
npm install node-red-contrib-mqtt-broker    # MQTT Broker (Mosquitto)

# รัน Node-RED เป็น Service
sudo npm install -g pm2
pm2 start node-red --name "node-red"
pm2 startup
pm2 save

# เข้าถึง Node-RED UI
# http://edge-ip:1880
```

### Node-RED Flow สำหรับ PLC Data Collection

```json
[
    {
        "id": "plc_s7_read",
        "type": "s7 in",
        "name": "Read PLC Tags",
        "address": "192.168.1.10",
        "rack": 0,
        "slot": 1,
        "variables": [
            {"name": "Motor_Speed", "type": "REAL", "address": "DB10,REAL0"},
            {"name": "Tank_Level", "type": "REAL", "address": "DB10,REAL4"},
            {"name": "Temperature", "type": "REAL", "address": "DB10,REAL8"},
            {"name": "Pressure", "type": "REAL", "address": "DB10,REAL12"}
        ],
        "cycle": 1000
    },
    {
        "id": "filter_outliers",
        "type": "function",
        "name": "Filter & Validate",
        "func": "// กรองค่าผิดปกติออก\nvar msg_out = {};\nvar data = msg.payload;\n\n// ตรวจสอบ Range\nif (data.Motor_Speed < 0 || data.Motor_Speed > 1500) {\n    node.warn('Motor Speed out of range: ' + data.Motor_Speed);\n    return null; // ไม่ส่งต่อ\n}\n\nif (data.Tank_Level < 0 || data.Tank_Level > 100) {\n    return null;\n}\n\n// เพิ่ม Timestamp และ Equipment ID\ndata.timestamp = new Date().toISOString();\ndata.equipment_id = 'LINE01_PLC01';\ndata.site = 'Factory_A';\n\nmsg.payload = data;\nreturn msg;"
    },
    {
        "id": "write_influxdb",
        "type": "influxdb out",
        "name": "Store in InfluxDB",
        "influxdb": "influx_local",
        "measurement": "plc_data",
        "precision": "ms"
    }
]
```

---

## Step 1624: InfluxDB สำหรับ Time Series Data

### การติดตั้งและตั้งค่า InfluxDB

```bash
# ติดตั้ง InfluxDB 2.x บน Edge Computer

# เพิ่ม InfluxDB Repository
wget -q https://repos.influxdata.com/influxdb.key
echo '23a1c8836f0afc5ed24e0486339d7cc8f6790b83886c4c96995b88a061c5bb5d influxdb.key' | sha256sum -c && cat influxdb.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdb.gpg > /dev/null

echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdb.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt update && sudo apt install influxdb2 -y

# เริ่มต้น InfluxDB
sudo systemctl enable influxdb
sudo systemctl start influxdb

# ตั้งค่าเริ่มต้น
influx setup \
  --username admin \
  --password "SecurePass123!" \
  --org "Factory_A" \
  --bucket "plc_data" \
  --retention "30d" \
  --force
```

### Flux Query Language

```flux
// Flux - ภาษา Query ของ InfluxDB 2.x

// Query 1: ดูค่าเฉลี่ยทุก 5 นาทีของ Motor Speed
from(bucket: "plc_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "plc_data")
  |> filter(fn: (r) => r._field == "Motor_Speed")
  |> filter(fn: (r) => r.equipment_id == "LINE01_PLC01")
  |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
  |> yield(name: "mean")

// Query 2: ตรวจสอบค่าที่เกิน Threshold
from(bucket: "plc_data")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "plc_data")
  |> filter(fn: (r) => r._field == "Temperature")
  |> filter(fn: (r) => r._value > 80.0)
  |> count()

// Query 3: หา Downtime Events
from(bucket: "plc_data")
  |> range(start: -7d)
  |> filter(fn: (r) => r._field == "Machine_Running")
  |> filter(fn: (r) => r._value == false)
  |> duration(unit: 1m)
  |> sum()
```

---

## Step 1625: Grafana สำหรับ Visualization

### การติดตั้ง Grafana บน Edge

```bash
# ติดตั้ง Grafana OSS
sudo apt install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install grafana -y

sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# เข้าถึง Grafana: http://edge-ip:3000
# Default: admin/admin
```

### Grafana Dashboard JSON Template สำหรับ OEE

```json
{
  "dashboard": {
    "title": "Production Line OEE Dashboard",
    "panels": [
      {
        "id": 1,
        "title": "OEE Gauge",
        "type": "gauge",
        "targets": [{
          "datasource": "InfluxDB",
          "query": "from(bucket: \"plc_data\") |> range(start: -1h) |> filter(fn: (r) => r._field == \"OEE\") |> last()"
        }],
        "fieldConfig": {
          "defaults": {
            "min": 0, "max": 100,
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "yellow", "value": 60},
                {"color": "green", "value": 85}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Production Count - Today",
        "type": "stat",
        "targets": [{
          "query": "from(bucket: \"plc_data\") |> range(start: today()) |> filter(fn: (r) => r._field == \"Good_Count\") |> last()"
        }]
      }
    ]
  }
}
```

---

## Step 1626: Containerization ด้วย Docker

### ทำไมต้องใช้ Docker บน Edge?

```
ข้อดีของ Docker บน Edge:
1. Isolation: แต่ละ Application แยกกัน
2. Portability: Deploy บน Edge ไหนก็ได้
3. Version Control: Roll back ง่าย
4. Resource Management: จำกัด CPU/RAM ได้
5. Easy Update: Pull new image แทนที่เดิม
```

### Docker Compose สำหรับ Edge Stack

```yaml
# docker-compose.yml - Edge Computing Stack
version: '3.8'

services:
  # MQTT Broker
  mosquitto:
    image: eclipse-mosquitto:2.0
    container_name: mosquitto
    restart: always
    ports:
      - "1883:1883"
      - "8883:8883"  # TLS
      - "9001:9001"  # WebSocket
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    networks:
      - edge_network

  # Node-RED
  node-red:
    image: nodered/node-red:3.1
    container_name: node-red
    restart: always
    ports:
      - "1880:1880"
    volumes:
      - ./nodered_data:/data
    environment:
      - TZ=Asia/Bangkok
    networks:
      - edge_network
    depends_on:
      - mosquitto

  # InfluxDB Time Series Database
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb_data:/var/lib/influxdb2
      - ./influxdb_config:/etc/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=SecurePass123!
      - DOCKER_INFLUXDB_INIT_ORG=Factory_A
      - DOCKER_INFLUXDB_INIT_BUCKET=plc_data
      - DOCKER_INFLUXDB_INIT_RETENTION=30d
    networks:
      - edge_network

  # Grafana Dashboard
  grafana:
    image: grafana/grafana:10.0.0
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - ./grafana_data:/var/lib/grafana
      - ./grafana_provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=AdminPass123!
      - GF_SERVER_ROOT_URL=http://edge-pc:3000
    networks:
      - edge_network
    depends_on:
      - influxdb

  # OPC UA Server (Prosys)
  opcua_server:
    image: opcua-prosys-server:latest
    container_name: opcua_server
    restart: always
    ports:
      - "4840:4840"
    volumes:
      - ./opcua_config:/config
    networks:
      - edge_network

  # AI Inference Engine
  ai_inference:
    image: factory_ai_inference:2.1
    container_name: ai_inference
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - ./ml_models:/models
    environment:
      - MODEL_PATH=/models/anomaly_detector_v2.pkl
      - INFERENCE_INTERVAL=5
    networks:
      - edge_network
    depends_on:
      - influxdb

networks:
  edge_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

---

## Step 1627: OPC UA บน Edge

### การตั้งค่า OPC UA Aggregation Server

```python
# Python - OPC UA Aggregation Server บน Edge
# รวบรวมข้อมูลจาก PLC หลายตัวและเปิดเป็น Single OPC UA Server

import asyncio
import logging
from asyncua import Server, Client
from asyncua import ua
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class EdgeOPCUAServer:
    def __init__(self, server_url="opc.tcp://0.0.0.0:4840"):
        self.server_url = server_url
        self.server = Server()
        
        # PLC Connections
        self.plc_connections = [
            {"name": "PLC_LINE01", "url": "opc.tcp://192.168.1.10:4840"},
            {"name": "PLC_LINE02", "url": "opc.tcp://192.168.1.11:4840"},
            {"name": "PLC_LINE03", "url": "opc.tcp://192.168.1.12:4840"},
        ]
        
        # Node Map: ชื่อ Node ในแต่ละ PLC
        self.node_map = {
            "Motor_Speed": "ns=3;s=Motor_Speed_Actual",
            "Tank_Level": "ns=3;s=Tank_Level_PV",
            "Temperature": "ns=3;s=Temperature_PV",
        }
        
        self.aggregated_nodes = {}
    
    async def initialize(self):
        """เริ่มต้น OPC UA Server"""
        await self.server.init()
        self.server.set_endpoint(self.server_url)
        self.server.set_server_name("Edge Aggregation Server - Factory A")
        
        # ตั้งค่า Security
        await self.server.set_security_policy([
            ua.SecurityPolicyType.Basic256Sha256_SignAndEncrypt
        ])
        
        # สร้าง Namespace
        uri = "http://factory-a.edge-server"
        idx = await self.server.register_namespace(uri)
        
        # สร้าง Node Structure
        objects = self.server.get_objects_node()
        
        for plc_info in self.plc_connections:
            plc_folder = await objects.add_folder(idx, plc_info["name"])
            self.aggregated_nodes[plc_info["name"]] = {}
            
            for tag_name in self.node_map.keys():
                node = await plc_folder.add_variable(idx, tag_name, 0.0)
                await node.set_writable()
                self.aggregated_nodes[plc_info["name"]][tag_name] = node
        
        logger.info(f"OPC UA Aggregation Server พร้อมใช้งาน: {self.server_url}")
    
    async def collect_and_aggregate(self):
        """เก็บข้อมูลจาก PLC และอัพเดท Aggregated Server"""
        while True:
            for plc_info in self.plc_connections:
                try:
                    async with Client(plc_info["url"]) as plc_client:
                        for tag_name, node_id in self.node_map.items():
                            try:
                                plc_node = plc_client.get_node(node_id)
                                value = await plc_node.get_value()
                                
                                # อัพเดทค่าใน Aggregated Server
                                agg_node = self.aggregated_nodes[plc_info["name"]][tag_name]
                                await agg_node.write_value(float(value))
                                
                            except Exception as e:
                                logger.error(f"ไม่สามารถอ่าน {tag_name} จาก {plc_info['name']}: {e}")
                
                except Exception as e:
                    logger.error(f"ไม่สามารถเชื่อมต่อ {plc_info['name']}: {e}")
            
            await asyncio.sleep(1)  # อัพเดททุก 1 วินาที
    
    async def run(self):
        """รัน Server"""
        await self.initialize()
        async with self.server:
            logger.info("Edge OPC UA Server กำลังทำงาน...")
            await self.collect_and_aggregate()

# รัน Server
if __name__ == "__main__":
    server = EdgeOPCUAServer()
    asyncio.run(server.run())
```

---

## Step 1628: Protocol Conversion บน Edge

### Modbus RTU → OPC UA

```python
# Python - Protocol Converter: Modbus RTU → OPC UA
# สำหรับอุปกรณ์เก่าที่ยังใช้ Modbus RS-485

import asyncio
from pymodbus.client import AsyncModbusSerialClient
from asyncua import Server

class ModbusToOPCUA:
    def __init__(self):
        # Modbus Client (RS-485)
        self.modbus_client = AsyncModbusSerialClient(
            port='/dev/ttyUSB0',
            baudrate=9600,
            parity='N',
            stopbits=1,
            bytesize=8,
            timeout=1
        )
        
        # Modbus Device Map
        self.devices = [
            {
                "slave_id": 1,
                "name": "Pump_01",
                "registers": [
                    {"address": 0, "name": "Speed_Feedback", "type": "holding"},
                    {"address": 1, "name": "Current", "type": "holding"},
                    {"address": 2, "name": "Temperature", "type": "holding"},
                    {"address": 100, "name": "Start_Stop", "type": "coil"},
                ]
            },
            {
                "slave_id": 2,
                "name": "Pump_02",
                "registers": [
                    {"address": 0, "name": "Speed_Feedback", "type": "holding"},
                    {"address": 1, "name": "Current", "type": "holding"},
                ]
            }
        ]
        
        # Scale Factors (Modbus มักเป็น Integer, ต้องหารสเกล)
        self.scale_factors = {
            "Speed_Feedback": 0.1,    # 10 → 1.0 RPM
            "Current": 0.01,          # 100 → 1.00 A
            "Temperature": 0.1,       # 250 → 25.0 °C
        }
    
    async def read_modbus_device(self, device):
        """อ่านข้อมูลจากอุปกรณ์ Modbus"""
        results = {}
        slave_id = device["slave_id"]
        
        for reg in device["registers"]:
            try:
                if reg["type"] == "holding":
                    response = await self.modbus_client.read_holding_registers(
                        address=reg["address"],
                        count=1,
                        slave=slave_id
                    )
                    if not response.isError():
                        raw_value = response.registers[0]
                        scale = self.scale_factors.get(reg["name"], 1.0)
                        results[reg["name"]] = raw_value * scale
                    else:
                        results[reg["name"]] = None
                        
                elif reg["type"] == "coil":
                    response = await self.modbus_client.read_coils(
                        address=reg["address"],
                        count=1,
                        slave=slave_id
                    )
                    if not response.isError():
                        results[reg["name"]] = response.bits[0]
                        
            except Exception as e:
                print(f"Error reading {reg['name']}: {e}")
                results[reg["name"]] = None
        
        return results
```

---

## Step 1629: Edge Analytics

### Real-time Statistical Process Control (SPC) บน Edge

```python
# Python - Real-time SPC บน Edge
import numpy as np
from collections import deque
import json

class EdgeSPCMonitor:
    """Statistical Process Control ที่ประมวลผลบน Edge"""
    
    def __init__(self, window_size=25):
        self.window_size = window_size
        self.data_buffers = {}  # {parameter_name: deque}
        self.control_limits = {}  # {parameter_name: {UCL, LCL, CL}}
        
    def add_parameter(self, param_name: str, target: float, process_sigma: float):
        """เพิ่มพารามิเตอร์สำหรับติดตาม"""
        self.data_buffers[param_name] = deque(maxlen=self.window_size)
        
        # คำนวณ Control Limits (3-sigma)
        self.control_limits[param_name] = {
            'UCL': target + 3 * process_sigma,  # Upper Control Limit
            'LCL': target - 3 * process_sigma,  # Lower Control Limit
            'CL': target,                        # Center Line
            'UWL': target + 2 * process_sigma,  # Upper Warning Limit
            'LWL': target - 2 * process_sigma,  # Lower Warning Limit
        }
    
    def add_measurement(self, param_name: str, value: float) -> dict:
        """รับค่าใหม่และวิเคราะห์"""
        if param_name not in self.data_buffers:
            return {'status': 'UNKNOWN_PARAMETER'}
        
        # เพิ่มข้อมูล
        self.data_buffers[param_name].append(value)
        limits = self.control_limits[param_name]
        
        # ตรวจสอบ Rules (Nelson Rules)
        result = {
            'value': value,
            'in_control': True,
            'violations': [],
            'cpk': 0,
            'recommendation': []
        }
        
        # Rule 1: จุดอยู่นอก Control Limits
        if value > limits['UCL']:
            result['in_control'] = False
            result['violations'].append('RULE1_ABOVE_UCL')
            result['recommendation'].append(f"ค่าสูงเกิน UCL ({limits['UCL']:.2f}): ตรวจสอบกระบวนการ")
            
        elif value < limits['LCL']:
            result['in_control'] = False
            result['violations'].append('RULE1_BELOW_LCL')
            result['recommendation'].append(f"ค่าต่ำกว่า LCL ({limits['LCL']:.2f}): ตรวจสอบกระบวนการ")
        
        # คำนวณ Cpk (ถ้ามีข้อมูลเพียงพอ)
        if len(self.data_buffers[param_name]) >= 5:
            data = list(self.data_buffers[param_name])
            mean = np.mean(data)
            std = np.std(data)
            
            if std > 0:
                cpu = (limits['UCL'] - mean) / (3 * std)
                cpl = (mean - limits['LCL']) / (3 * std)
                result['cpk'] = min(cpu, cpl)
                
                if result['cpk'] < 1.0:
                    result['recommendation'].append(
                        f"Cpk = {result['cpk']:.2f} < 1.0: กระบวนการไม่มีความสามารถเพียงพอ"
                    )
        
        return result
```

---

## Step 1630-1635: Complete Edge Node Setup

### การออกแบบ Edge Node สมบูรณ์สำหรับสายการผลิต

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE EDGE NODE ARCHITECTURE                      │
│                    สายการผลิตอาหาร - Line 01                           │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Edge Computer (Siemens IPC427E)                                 │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │  │
│  │  │  Node-RED  │  │ InfluxDB   │  │  Grafana   │  │  MQTT    │  │  │
│  │  │  (1880)    │  │  (8086)    │  │  (3000)    │  │ Mosquitto│  │  │
│  │  │            │  │            │  │            │  │  (1883)  │  │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │  │
│  │                                                                  │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────────────┐ │  │
│  │  │  OPC UA    │  │  AI Model  │  │  Protocol Converter        │ │  │
│  │  │  Server    │  │  Inference │  │  (Modbus→OPC UA)           │ │  │
│  │  │  (4840)    │  │  (5000)    │  │                            │ │  │
│  │  └────────────┘  └────────────┘  └────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │ │ │ │                                    │
│         ┌───────────────┬────┘ │ │ └────┬───────────────┐              │
│         ▼               ▼      │ │      ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐│┌─────────────┐ ┌─────────────┐     │
│  │Siemens      │ │Siemens      │││ KEYENCE      │ │ Yokogawa   │     │
│  │S7-1500      │ │ET200SP      │││ Vision Cam   │ │ Transmitter│     │
│  │(PROFINET)   │ │(Remote I/O) │││ (GigE)       │ │(Modbus RTU)│     │
│  └─────────────┘ └─────────────┘│└─────────────┘ └─────────────┘     │
│                                  │                                     │
│                          ┌───────┴────────┐                           │
│                          │ Switch PROFINET │                           │
│                          │ Managed L2      │                           │
│                          └────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1636: Edge Security

### การรักษาความปลอดภัย Edge Node

```bash
# Hardening Edge Computer บน Linux

# 1. อัพเดทระบบ
sudo apt update && sudo apt upgrade -y

# 2. ติดตั้ง Firewall
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing

# อนุญาตเฉพาะ Port ที่จำเป็น
sudo ufw allow from 192.168.1.0/24 to any port 1880    # Node-RED (LAN Only)
sudo ufw allow from 192.168.1.0/24 to any port 1883    # MQTT (LAN Only)
sudo ufw allow from 192.168.1.0/24 to any port 4840    # OPC UA (LAN Only)
sudo ufw allow from 192.168.1.0/24 to any port 8086    # InfluxDB (LAN Only)
sudo ufw allow from 192.168.1.0/24 to any port 3000    # Grafana (LAN Only)
sudo ufw allow 22                                        # SSH
sudo ufw enable

# 3. ตั้งค่า Fail2ban (ป้องกัน Brute Force)
sudo apt install fail2ban -y

cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = 22
EOF

sudo systemctl restart fail2ban

# 4. Encrypt Disk
sudo apt install cryptsetup -y
# (ต้อง Format ใหม่ - ทำในขั้นตอน Installation)

# 5. Certificate Management สำหรับ OPC UA TLS
openssl req -x509 -nodes -days 3650 \
  -newkey rsa:4096 \
  -keyout /opt/opcua/server.key \
  -out /opt/opcua/server.crt \
  -subj "/C=TH/ST=Bangkok/O=Factory_A/CN=edge-node-01"
```

---

## Step 1637: Edge-to-Cloud Communication

### การส่งข้อมูลจาก Edge ไป Cloud อย่างมีประสิทธิภาพ

```python
# Python - Edge to Cloud Data Uploader
# ใช้ Data Compression และ Batching เพื่อลด Bandwidth

import asyncio
import json
import gzip
import aiohttp
from datetime import datetime, timedelta
import sqlite3  # Buffer สำหรับ Offline Mode

class EdgeToCloudUploader:
    def __init__(self, cloud_api_url, api_key):
        self.cloud_api_url = cloud_api_url
        self.api_key = api_key
        self.batch_size = 1000          # ส่งทีละ 1000 records
        self.send_interval = 60         # ส่งทุก 60 วินาที
        
        # Local Buffer สำหรับกรณี Internet ขาด
        self.init_local_buffer()
    
    def init_local_buffer(self):
        """สร้าง SQLite Database สำหรับ Buffer"""
        self.conn = sqlite3.connect('/data/edge_buffer.db')
        cursor = self.conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS data_buffer (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp TEXT,
                equipment_id TEXT,
                data_json TEXT,
                sent INTEGER DEFAULT 0
            )
        ''')
        self.conn.commit()
    
    def buffer_data(self, equipment_id: str, data: dict):
        """บันทึกข้อมูลลง Buffer"""
        cursor = self.conn.cursor()
        cursor.execute(
            "INSERT INTO data_buffer (timestamp, equipment_id, data_json) VALUES (?, ?, ?)",
            (datetime.now().isoformat(), equipment_id, json.dumps(data))
        )
        self.conn.commit()
    
    async def upload_batch(self):
        """ส่งข้อมูล Batch ขึ้น Cloud"""
        cursor = self.conn.cursor()
        
        # ดึงข้อมูลที่ยังไม่ได้ส่ง
        cursor.execute(
            "SELECT id, timestamp, equipment_id, data_json FROM data_buffer WHERE sent = 0 ORDER BY id LIMIT ?",
            (self.batch_size,)
        )
        records = cursor.fetchall()
        
        if not records:
            return
        
        # สร้าง Batch Payload
        batch = [
            {
                "id": r[0],
                "timestamp": r[1],
                "equipment_id": r[2],
                "data": json.loads(r[3])
            }
            for r in records
        ]
        
        # Compress ข้อมูล (ลด Bandwidth ~70%)
        payload = json.dumps(batch).encode('utf-8')
        compressed = gzip.compress(payload)
        
        try:
            async with aiohttp.ClientSession() as session:
                async with session.post(
                    f"{self.cloud_api_url}/api/v2/telemetry/batch",
                    data=compressed,
                    headers={
                        "Authorization": f"Bearer {self.api_key}",
                        "Content-Encoding": "gzip",
                        "Content-Type": "application/json",
                        "X-Edge-Node-ID": "EDGE_LINE01"
                    },
                    timeout=aiohttp.ClientTimeout(total=30)
                ) as response:
                    if response.status == 200:
                        # Mark as sent
                        ids = [r[0] for r in records]
                        cursor.execute(
                            f"UPDATE data_buffer SET sent = 1 WHERE id IN ({','.join('?'*len(ids))})",
                            ids
                        )
                        self.conn.commit()
                        print(f"ส่งข้อมูล {len(records)} records สำเร็จ (compressed: {len(compressed)/1024:.1f} KB)")
                    else:
                        print(f"ส่งข้อมูลล้มเหลว: HTTP {response.status}")
        
        except aiohttp.ClientConnectorError:
            print("Internet ขาด - ข้อมูล Buffer ไว้ในเครื่อง")
        except Exception as e:
            print(f"Error uploading: {e}")
    
    async def run(self):
        """Loop หลัก"""
        while True:
            await self.upload_batch()
            await asyncio.sleep(self.send_interval)
```

---

## Step 1638: Edge Monitoring และ Management

### Remote Edge Management

```yaml
# Portainer CE สำหรับจัดการ Docker บน Edge ผ่าน Web UI
# docker-compose.portainer.yml

version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "9443:9443"   # HTTPS Web UI
      - "8000:8000"   # Edge Agent Port
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    security_opt:
      - no-new-privileges:true

volumes:
  portainer_data:
```

---

## Step 1639: Performance Tuning

### การปรับแต่งประสิทธิภาพ Edge Node

```bash
# Optimize Linux สำหรับ Real-time Industrial Edge

# 1. ปรับ Network Buffer
cat >> /etc/sysctl.conf << 'EOF'
# เพิ่ม Network Buffer สำหรับ High-throughput
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 65536 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# ลด Latency สำหรับ TCP
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_nodelay = 1

# IEC 62443 - ปิด IP Forwarding (ถ้าไม่ใช้ Router)
net.ipv4.ip_forward = 0
EOF

sudo sysctl -p

# 2. ปรับ CPU Governor (Performance Mode)
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 3. ปรับ Disk I/O Scheduler (สำหรับ SSD)
echo none | sudo tee /sys/block/sda/queue/scheduler

# 4. ตั้งค่า Real-time Priority สำหรับ Critical Process
sudo chrt -f -p 99 $(pgrep -f opcua_server)
```

---

## Step 1640: Lab Project - Edge Node สำหรับสายการผลิต

### โปรเจกต์: ออกแบบและ Deploy Edge Node สมบูรณ์

**โจทย์**: ออกแบบ Edge Node สำหรับสายการผลิตน้ำดื่ม 1 สาย

**Infrastructure**:
- 1x Siemens S7-1200 (Main Control)
- 3x Modbus RTU Meter (Flow, Level, Pressure)
- 1x Camera สำหรับ Label Inspection
- 1x Raspberry Pi CM4 (Edge Node)

**Software Stack**:
1. Node-RED: Data Collection จาก PLC และ Modbus
2. InfluxDB: เก็บ Time Series Data
3. Grafana: Dashboard OEE + Quality
4. MQTT: ส่งข้อมูลไป Cloud

**Lab Tasks**:
1. ติดตั้ง Docker + Docker Compose บน Raspberry Pi
2. Deploy Full Stack ด้วย docker-compose
3. สร้าง Node-RED Flow สำหรับ Data Collection
4. สร้าง Grafana Dashboard
5. ทดสอบ Offline Operation

---

## สรุป Part 82

ใน Part นี้เราได้เรียนรู้:
- ✅ ความแตกต่าง Edge / Fog / Cloud
- ✅ Hardware: Siemens IPC, RPi Industrial, Moxa, Advantech
- ✅ Software: Node-RED, InfluxDB, Grafana, MQTT
- ✅ Docker Containerization บน Edge
- ✅ OPC UA Aggregation Server
- ✅ Protocol Conversion (Modbus → OPC UA)
- ✅ Real-time SPC บน Edge
- ✅ Edge-to-Cloud Communication
- ✅ Security Hardening

---

## การนำทาง
- [← Part 81: Smart Manufacturing](part81_smart_manufacturing.md)
- [→ Part 83: AI/ML Integration กับ PLC](part83_ai_plc.md)
- [📚 สารบัญหลัก](../README.md)
