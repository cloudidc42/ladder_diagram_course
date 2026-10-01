# Part 90: Advanced Industrial Networking
## Steps 1781-1800 | ระดับ Capstone

---

## การนำทาง
- [← Part 89: PLC ถึง ERP Integration](part89_plc_to_erp.md)
- [→ Part 91: Capstone - Automotive Assembly Line](part91_capstone_automotive.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1781: TSN (Time-Sensitive Networking)

### TSN คืออะไร?

**Time-Sensitive Networking (TSN)** คือชุดมาตรฐาน IEEE 802.1 ที่ขยาย Ethernet ธรรมดาให้รองรับการสื่อสาร Real-time แบบ Deterministic

```
Ethernet ธรรมดา:
  Latency: ตัวแปร (1ms - 100ms+)
  ใช้: IT Networks, Office, Internet
  ไม่เหมาะ: Real-time Control

TSN Ethernet:
  Latency: Guaranteed < 1ms
  Jitter: < 1 microsecond
  ใช้: Industrial Control, Automotive, AV/Broadcast
  เหมาะ: Motion Control, Time-critical Industrial
```

### TSN Standards

```
IEEE 802.1AS  - Timing and Synchronization (gPTP)
IEEE 802.1Qbv - Scheduled Traffic (Time-Aware Shaper)
IEEE 802.1Qbu - Frame Preemption
IEEE 802.1Qcr - Asynchronous Traffic Shaping
IEEE 802.1CB  - Frame Replication and Elimination (Redundancy)
IEC/IEEE 60802 - TSN Profile for Industrial Automation (IA-profile)
```

### TSN Use Cases ในโรงงาน

```
1. Motion Control (แทน EtherCAT/PROFINET IRT)
   - Servo Drive Synchronization < 1μs jitter
   - CNC Machine Tool Control
   - Printing Press Control

2. Mixed Traffic (IT/OT Convergence)
   - PROFINET และ IT Traffic บน Cable เดียวกัน
   - OPC UA Pub/Sub over TSN
   - Safety over TSN

3. Automotive (In-Vehicle)
   - AV Sensor Data
   - ADAS Control
   - Vehicle Diagnostics (OBD)
```

---

## Step 1782: 5G สำหรับ Industrial

### 5G ในโรงงาน

```
5G Characteristics:
┌────────────────────────────────────────────────────────────────┐
│ Feature               │ 4G LTE        │ 5G NR          │
├───────────────────────┼───────────────┼────────────────┤
│ Peak Speed            │ 150 Mbps      │ 20 Gbps        │
│ Latency               │ 10-50ms       │ 1ms (URLLC)    │
│ Reliability           │ 99.9%         │ 99.9999%       │
│ Device Density        │ 100k/km²      │ 1M/km²         │
│ Slicing               │ ไม่รองรับ     │ รองรับ         │
└───────────────────────┴───────────────┴────────────────┘

5G Use Cases ในโรงงาน:
1. AGV/AMR Control (ไม่ต้องใช้ WiFi)
2. Collaborative Robot Wireless
3. AR/VR Maintenance Guidance
4. Real-time Video Inspection
5. Massive IoT Sensor Networks
```

### Private 5G Network

```
Private 5G Architecture:
┌──────────────────────────────────────────────────────────────────┐
│  Factory Private 5G Network                                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  5G Core (On-Premise)                                      │  │
│  │  ├── AMF (Access & Mobility Management)                    │  │
│  │  ├── SMF (Session Management)                              │  │
│  │  ├── UPF (User Plane - Data Routing)                       │  │
│  │  └── UDM (User Data Management)                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────┼───────────────────────────────────┐   │
│  │  Indoor 5G Radio     │  (gNB - gNodeB)                   │   │
│  │  ├── Antenna Zone 1  │ (Manufacturing Area)              │   │
│  │  ├── Antenna Zone 2  │ (Warehouse)                       │   │
│  │  └── Antenna Zone 3  │ (Outdoor)                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Connected Devices:                                               │
│  AGV, AMR, Cobot, Wearables, Tablets, Smart Cameras             │
└──────────────────────────────────────────────────────────────────┘

5G Frequency Bands for Industry:
CBRS (3.55-3.7 GHz) - USA
n77 (3.3-4.2 GHz) - Europe, Asia
n79 (4.4-5.0 GHz) - Asia (Thailand กำลังใช้)
n257 (26.5-29.5 GHz) - mmWave (High bandwidth, Short range)
```

---

## Step 1783: WiFi 6 (802.11ax) สำหรับ Industrial

### WiFi 6 vs WiFi 5 สำหรับโรงงาน

```
WiFi 6 (802.11ax) Improvements:
1. OFDMA: แบ่งช่องสัญญาณให้หลายอุปกรณ์พร้อมกัน
2. MU-MIMO 8x8: 8 Streams พร้อมกัน
3. BSS Coloring: ลด Co-channel Interference
4. TWT (Target Wake Time): ยืด Battery Life ของ IoT
5. WPA3: Security ที่แข็งแกร่งขึ้น

WiFi 6E (6 GHz Band):
- ช่องสัญญาณใหม่ 6 GHz (5.925-7.125 GHz)
- Channel Width ถึง 160 MHz
- ลด Interference จาก 2.4/5 GHz Devices มาก
- เหมาะสำหรับ Video, AR/VR, Dense Environments

Industrial WiFi 6 Best Practices:
1. ใช้ 5 GHz หรือ 6 GHz สำหรับ Industrial Devices
2. กำหนด QoS: Priority สำหรับ Control Traffic
3. Roaming: 802.11r (Fast BSS Transition)
4. Certificate-based Authentication (WPA3-Enterprise)
5. VLAN Separation: OT เน็ตเวิร์คแยกจาก IT
```

---

## Step 1784: WirelessHART

### WirelessHART สำหรับ Process Industry

```
WirelessHART (IEC 62591):
- Frequency: 2.4 GHz (IEEE 802.15.4)
- Mesh Network (Self-organizing, Self-healing)
- Range: 50-100m per hop (Multi-hop ได้ถึง 1km+)
- Latency: 100ms - 4s (เหมาะ slow process)
- Security: AES-128
- Battery Life: 5-10 ปี
- Devices: Pressure, Temperature, Level, Flow transmitters

Use Cases:
- Remote Process Tanks
- Hard-to-reach piping
- Rotating Equipment (vibration sensors)
- Temporary Measurement
```

```python
# Python - WirelessHART Gateway API Client
import requests
import json

class WirelessHARTGateway:
    """เชื่อมต่อกับ WirelessHART Gateway (Emerson/ABB)"""
    
    def __init__(self, gateway_ip: str):
        self.gateway_url = f"http://{gateway_ip}/api/v1"
        self.session = requests.Session()
    
    def get_all_devices(self) -> list:
        """ดู Devices ทั้งหมดใน Mesh Network"""
        response = self.session.get(f"{self.gateway_url}/devices")
        return response.json()['devices']
    
    def get_device_values(self, device_tag: str) -> dict:
        """ดูค่าจาก Device"""
        response = self.session.get(
            f"{self.gateway_url}/devices/{device_tag}/values"
        )
        data = response.json()
        
        return {
            'tag': device_tag,
            'pv': data.get('PV'),           # Primary Variable
            'sv': data.get('SV'),           # Secondary Variable
            'tv': data.get('TV'),           # Tertiary Variable
            'qv': data.get('QV'),           # Quaternary Variable
            'battery_pct': data.get('BATTERY'),
            'signal_strength': data.get('RSSI'),
            'timestamp': data.get('TIMESTAMP'),
            'status': data.get('DEVICE_STATUS')
        }
    
    def get_network_health(self) -> dict:
        """ตรวจสอบสุขภาพ Mesh Network"""
        response = self.session.get(f"{self.gateway_url}/network/health")
        return response.json()
```

---

## Step 1785: ISA100.11a

### ISA100.11a Industrial Wireless

```
ISA100.11a vs WirelessHART:
┌──────────────────────────────────────────────────────────────┐
│ Feature           │ WirelessHART     │ ISA100.11a            │
├───────────────────┼──────────────────┼───────────────────────┤
│ Standard          │ HART Foundation  │ ISA/IEC 62734         │
│ Frequency         │ 2.4 GHz only     │ 2.4 GHz + Sub-GHz    │
│ Topology          │ Mesh             │ Mesh, Star, Hybrid    │
│ Application Layer │ HART Protocol    │ Open (HART, Modbus..) │
│ Latency           │ 100ms-4s         │ 10ms-100ms (config)   │
│ Security          │ AES-128          │ AES-128 + PKI         │
│ Bandwidth         │ 250 kbps         │ 250 kbps              │
│ Adoption          │ High (Process)   │ Limited               │
└───────────────────┴──────────────────┴───────────────────────┘
```

---

## Step 1786: LoRaWAN สำหรับ Remote Sensors

### LoRaWAN สำหรับการวัดระยะไกล

```
LoRaWAN Characteristics:
- Range: 2-15 km (Urban), 15-45 km (Rural/LOS)
- Bandwidth: 0.3-50 kbps (เหมาะข้อมูลน้อย)
- Power: Battery 5-10 ปี
- Cost: Low ($5-50 per node)
- Protocol: LPWAN (Low Power Wide Area Network)

Use Cases ในโรงงาน:
1. Remote Tank Level Monitoring
2. Environment Monitoring (Temperature, Humidity)
3. Gate/Door Status
4. Parking Space Detection
5. Energy Meter Reading
6. Water Meter Reading

LoRaWAN Architecture:
Sensors → Gateway → LoRaWAN Network Server → Application Server
```

```python
# Python - LoRaWAN Application Server (The Things Network)
import requests
import json
from datetime import datetime

class LoRaWANClient:
    """เชื่อมต่อกับ LoRaWAN Network Server (TTN/ChirpStack)"""
    
    def __init__(self, network_server: str, api_key: str, application_id: str):
        self.server = network_server
        self.app_id = application_id
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Accept': 'application/json'
        }
    
    def get_device_data(self, device_eui: str, limit: int = 10) -> list:
        """ดูข้อมูลล่าสุดจาก LoRaWAN Device"""
        response = requests.get(
            f"{self.server}/api/v3/as/applications/{self.app_id}/devices/{device_eui}/packages/storage/uplink_message",
            headers=self.headers,
            params={'limit': limit}
        )
        
        if response.status_code != 200:
            return []
        
        messages = []
        for line in response.text.strip().split('\n'):
            if line:
                try:
                    msg = json.loads(line)
                    # Decode Base64 Payload
                    import base64
                    payload_b64 = msg.get('result', {}).get('uplink_message', {}).get('frm_payload', '')
                    if payload_b64:
                        payload_bytes = base64.b64decode(payload_b64)
                        decoded = self.decode_sensor_payload(payload_bytes)
                        
                        messages.append({
                            'timestamp': msg.get('result', {}).get('received_at', ''),
                            'device': device_eui,
                            'rssi': msg.get('result', {}).get('uplink_message', {}).get('rx_metadata', [{}])[0].get('rssi', 0),
                            'snr': msg.get('result', {}).get('uplink_message', {}).get('rx_metadata', [{}])[0].get('snr', 0),
                            'data': decoded
                        })
                except:
                    pass
        
        return messages
    
    def decode_sensor_payload(self, payload: bytes) -> dict:
        """ถอดรหัส Sensor Payload"""
        # ตัวอย่าง: Cayenne LPP Format
        data = {}
        i = 0
        while i < len(payload) - 2:
            channel = payload[i]
            data_type = payload[i+1]
            i += 2
            
            if data_type == 0x67:  # Temperature (0.1°C)
                value = int.from_bytes(payload[i:i+2], 'big', signed=True) / 10.0
                data[f'temperature_{channel}'] = value
                i += 2
            elif data_type == 0x68:  # Humidity (0.5%)
                value = payload[i] / 2.0
                data[f'humidity_{channel}'] = value
                i += 1
            elif data_type == 0x73:  # Barometer (0.1 hPa)
                value = int.from_bytes(payload[i:i+2], 'big', signed=False) / 10.0
                data[f'pressure_{channel}'] = value
                i += 2
            elif data_type == 0x02:  # Analog Input (0.01)
                value = int.from_bytes(payload[i:i+2], 'big', signed=True) / 100.0
                data[f'analog_{channel}'] = value
                i += 2
            else:
                break
        
        return data
```

---

## Step 1787: NB-IoT สำหรับ Remote Monitoring

### NB-IoT (Narrowband IoT)

```
NB-IoT Characteristics:
- Based on: LTE (Cellular)
- Provider: AIS, DTAC, TRUE (ไทย)
- Coverage: ทั่วประเทศ (ใช้ Infrastructure Cellular)
- Range: ไม่จำกัด (Cellular)
- Bandwidth: 200 kHz
- Battery: 5-10 ปี
- Security: Cellular-grade (SIM-based)
- Cost: SIM Card + Monthly Fee (~50-200 บาท/เดือน)

เหมาะกับ:
- Remote Sites (นิคมอุตสาหกรรมห่างไกล)
- Outdoor Environment Monitoring  
- Agricultural Sensors
- Utility Metering (Water, Gas, Electric)
```

---

## Step 1788: Network Selection Guide

### เลือก Technology ให้เหมาะกับงาน

```
INDUSTRIAL WIRELESS TECHNOLOGY SELECTION GUIDE:

┌──────────────────────────────────────────────────────────────────────┐
│  SELECTION CRITERIA                                                  │
│                                                                      │
│  Low Latency (< 10ms) + High Reliability?                           │
│  → TSN Ethernet (Wired) หรือ 5G Private                             │
│                                                                      │
│  Medium Speed Process + Low Power + Mesh Network?                   │
│  → WirelessHART (Process Industry)                                  │
│  → ISA100.11a (Flexible Applications)                               │
│                                                                      │
│  Mobile Equipment (AGV, Forklift)?                                  │
│  → WiFi 6 (ในโรงงาน)                                               │
│  → 5G Private (ต้องการ Coverage กว้าง)                              │
│                                                                      │
│  Remote Monitoring + Low Data + Battery Powered?                    │
│  → LoRaWAN (Range 2-15km, ไม่มีค่า Cellular)                       │
│  → NB-IoT (Nationwide Coverage, มีค่า Cellular)                    │
│                                                                      │
│  Short Range + Low Cost?                                             │
│  → Bluetooth 5.0 (< 50m)                                            │
│  → Zigbee (Mesh, < 100m per hop)                                    │
└──────────────────────────────────────────────────────────────────────┘
```

### Network Comparison Matrix

```
Technology  │ Range   │ Latency  │ Bandwidth │ Power  │ Cost  │ Security
────────────┼─────────┼──────────┼───────────┼────────┼───────┼──────────
WiFi 6      │ 100m    │ 1-10ms   │ 9.6 Gbps  │ High   │ Low   │ WPA3
5G Private  │ 200-2km │ 1ms      │ 20 Gbps   │ High   │ High  │ Strong
TSN         │ 100m    │ <1ms     │ 1-10 Gbps │ High   │ Med   │ Layer 2
WirelessHART│ 100m/hop│ 100ms-4s │ 250 kbps  │ Ultra  │ Low   │ AES-128
LoRaWAN     │ 2-45km  │ 1-5s     │ 50 kbps   │ Ultra  │ Very  │ AES-128
NB-IoT      │ Natl.   │ 1-10s    │ 200 kbps  │ Ultra  │ Med   │ Cellular
Bluetooth 5 │ 50m     │ 1-100ms  │ 2 Mbps    │ Low    │ Low   │ AES-CCM
Zigbee      │ 100m/hop│ 10-100ms │ 250 kbps  │ Low    │ Low   │ AES-128
```

---

## Step 1789: Bluetooth ใน Industrial

### Bluetooth 5.x สำหรับโรงงาน

```
Bluetooth 5.0/5.2/5.3 Features:
- Long Range Mode: ±400m (ลด Bandwidth)
- High Throughput: 2 Mbps
- Bluetooth Mesh: ≤32,767 Nodes
- Direction Finding (AoA/AoD): Localization < 1m
- Connectionless: Broadcast สำหรับ Asset Tracking

Industrial Use Cases:
1. Tool Tracking & Management
2. Personnel Safety (RTLS)
3. Beacon-based Asset Location
4. Wearable Sensors (Body Temperature, Fall Detection)
5. Smart Lighting Control
```

---

## Step 1790-1800: Complete Network Architecture

### สถาปัตยกรรมเครือข่ายโรงงานสมบูรณ์

```
┌────────────────────────────────────────────────────────────────────────┐
│              COMPLETE FACTORY NETWORK ARCHITECTURE                     │
│                                                                        │
│  INTERNET                                                              │
│     │ 1 Gbps Fiber                                                    │
│  ┌──▼─────────────────────────────────────────────────────────────┐  │
│  │  DMZ ZONE - Firewall Level 1 (Cisco ASA)                       │  │
│  │  ├── Web Server (Customer Portal)                               │  │
│  │  ├── VPN Gateway (Remote Access)                                │  │
│  │  └── Email / DNS                                                │  │
│  └──────────────────────────────────────────────────────────────  │  │
│                         │ 10 Gbps Core Fiber                        │  │
│  ┌──────────────────────▼─────────────────────────────────────────┐  │
│  │  IT NETWORK - Core Switch (Cisco Nexus 9K)                     │  │
│  │  VLAN 10: Servers (ERP, MES, Email)                            │  │
│  │  VLAN 20: User PCs                                             │  │
│  │  VLAN 30: WiFi (Office)                                        │  │
│  └──────────────────────────────────────────────────────────────  │  │
│                         │ Firewall Level 2 (Data Diode)            │  │
│  ┌──────────────────────▼─────────────────────────────────────────┐  │
│  │  OT NETWORK - Core Switch (Hirschmann MACH4002)                │  │
│  │  VLAN 100: SCADA/HMI Servers                                   │  │
│  │  VLAN 101: PLC Network (PROFINET)                              │  │
│  │  VLAN 102: Safety Network (PROFISAFE)                          │  │
│  │  VLAN 103: Motion Network (TSN/EtherCAT)                       │  │
│  │  VLAN 104: Vision Systems (GigE)                               │  │
│  │  VLAN 105: AGV/AMR Network (WiFi 6)                            │  │
│  └──────────────────────────────────────────────────────────────  │  │
│                         │                                            │  │
│  ┌──────────────────────▼─────────────────────────────────────────┐  │
│  │  5G PRIVATE NETWORK (Nokia DAC)                                │  │
│  │  Slice 1: Critical Control (AGV, Cobot) - URLLC 1ms           │  │
│  │  Slice 2: IoT Sensors - eMTC 1000 devices/cell                 │  │
│  │  Slice 3: Video/AR (Maintenance) - eMBB 10 Gbps               │  │
│  └──────────────────────────────────────────────────────────────  │  │
│                                                                        │
│  Wireless Overlay:                                                     │
│  ├── WiFi 6 AP (Cisco Catalyst 9136): Motion, AGV, Tablets         │  │
│  ├── WirelessHART (Emerson WirelessHART GW): Process Sensors       │  │
│  ├── LoRaWAN (RAK7249 GW): Remote Utility Meters                   │  │
│  └── BLE 5.2 (Nordic nRF5340): Asset Tracking                      │  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Lab Project: Factory Network Design

### โปรเจกต์: ออกแบบเครือข่ายโรงงาน EV Battery

**Requirements**:
1. Safety Network สำหรับ Battery Formation (ไฟฟ้าสูง)
2. Precision Motion Control สำหรับ Electrode Cutting (TSN)
3. AGV Fleet (5G Private)
4. Environmental Monitoring (WirelessHART)
5. IT-OT Integration ที่ปลอดภัย

**Deliverables**:
1. Network Architecture Diagram
2. VLAN Design
3. Firewall Rules
4. 5G Frequency Plan
5. Security Policy

---

## สรุป Part 90

ใน Part นี้เราได้เรียนรู้:
- ✅ TSN (Time-Sensitive Networking) IEEE 802.1
- ✅ 5G Private Network สำหรับอุตสาหกรรม
- ✅ WiFi 6/6E Industrial Grade
- ✅ WirelessHART/ISA100.11a
- ✅ LoRaWAN สำหรับ Remote Monitoring
- ✅ NB-IoT (Narrowband IoT)
- ✅ Network Technology Selection Guide
- ✅ Complete Factory Network Architecture

---

## การนำทาง
- [← Part 89: PLC ถึง ERP Integration](part89_plc_to_erp.md)
- [→ Part 91: Capstone - Automotive Assembly Line](part91_capstone_automotive.md)
- [📚 สารบัญหลัก](../README.md)
