# Part 97: Advanced Troubleshooting Techniques
## Steps 1921-1940 | ระดับ Expert

---

## การนำทาง
- [← Part 96: PLC Interview Questions](part96_plc_interview_questions.md)
- [→ Part 98: Future Technologies](part98_future_technologies.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1921: Root Cause Analysis (RCA)

### 5-Why Analysis Method

```
Case Study: Conveyor หยุดทำงานกะกลางคืน

Why #1: ทำไม Conveyor จึงหยุด?
→ Overload Relay ตัด (OL Trip)

Why #2: ทำไม Overload Relay จึงตัด?
→ มอเตอร์กินกระแสเกิน

Why #3: ทำไม มอเตอร์จึงกินกระแสเกิน?
→ Belt มีแรงดึงมากผิดปกติ

Why #4: ทำไม Belt จึงมีแรงดึงมาก?
→ Bearing ล็อค (Seize)

Why #5: ทำไม Bearing จึงล็อค?
→ ขาดการหล่อลื่นตามกำหนด (PM ไม่ได้ทำ)

Root Cause: ไม่มีระบบ PM Reminder ที่ผูกกับ CMMS
Corrective Action: ตั้ง PM Schedule ใน CMMS + Lubrication Sensor
Preventive Action: Implement CBM (Condition Based Maintenance)
```

### Fishbone Diagram (Ishikawa)

```
Problem: Variable Frequency Drive ตั้งค่าไม่ได้

              Measurement    Materials
               |              |
               |  Wrong Meter |  Bad Cable
    Method ────┼──────────────┼──── Machine
               |              |
    Wrong Proc |              | VFD Fault
               |              |
               └──── PROBLEM ──┘
                      ↑
                Man   Environment
               |              |
    No Training |  High Temp  |
               |              |
    Wrong Param |  Humidity   |
```

### Fault Tree Analysis (FTA)

```
TOP EVENT: Production Line Stops (Unplanned)
│
├── OR Gate
│    ├── PLC Fault
│    │    ├── CPU Failure
│    │    ├── Power Supply Failure
│    │    └── Communication Loss
│    │
│    ├── Mechanical Failure
│    │    ├── Motor Fault
│    │    ├── Belt Break
│    │    └── Bearing Failure
│    │
│    ├── Electrical Fault
│    │    ├── Short Circuit
│    │    ├── Fuse Blown
│    │    └── VFD Fault
│    │
│    └── External
│         ├── Power Outage
│         ├── E-Stop Activated
│         └── Material Shortage

Minimal Cut Sets = Combinations that cause TOP Event
Used for: SIL Verification, Risk Reduction Planning
```

---

## Step 1922: Wireshark สำหรับ Industrial Networks

### Network Packet Analysis

```
Wireshark สำหรับ OT Network:

Installation:
sudo apt install wireshark  # Linux
# หรือดาวน์โหลดจาก wireshark.org

OT Protocol Dissectors (Built-in):
- PROFINET RT/IRT
- EtherNet/IP (CIP)
- Modbus TCP
- OPC UA
- DNP3
- IEC 61850 GOOSE

PROFINET Troubleshooting:

Filter 1: ดู PROFINET Packets ทั้งหมด
pn_rt

Filter 2: ดู PROFINET Device ที่ Address เฉพาะ
eth.addr == 00:0e:8c:a1:b2:c3

Filter 3: ดู Alarm Frame
pn_rt.frame_id == 0xfc01

Filter 4: ดู Cyclic Data (IO Data)
pn_rt.frame_id >= 0x8000 && pn_rt.frame_id <= 0xbfff

MODBUS Troubleshooting:

Filter: Request/Response
modbus.func_code == 3  # Read Holding Registers

ดู Error:
modbus.exception_code  # ถ้ามีค่า = Error

EtherNet/IP:

Filter: CIP Connection
enip

Filter: ดู Error Responses
cip.status != 0x00
```

```python
# Python - ใช้ Scapy สำหรับ OT Network Analysis
from scapy.all import *
from datetime import datetime
import collections

class OTNetworkAnalyzer:
    """
    วิเคราะห์ Traffic บน Industrial Network
    ใช้สำหรับ Troubleshooting
    """
    
    def __init__(self, interface: str):
        self.iface = interface
        self.packet_counts = collections.Counter()
        self.modbus_transactions = []
        self.errors = []
    
    def capture_and_analyze(self, duration_sec: int = 60):
        """จับและวิเคราะห์ Packet"""
        print(f"กำลัง Capture บน {self.iface} เป็นเวลา {duration_sec} วินาที...")
        
        packets = sniff(
            iface=self.iface,
            timeout=duration_sec,
            prn=self.process_packet
        )
        
        self.print_summary()
    
    def process_packet(self, pkt):
        """วิเคราะห์แต่ละ Packet"""
        
        # นับ Protocol ที่พบ
        if pkt.haslayer('TCP'):
            self.packet_counts['TCP'] += 1
            
            # Modbus TCP (Port 502)
            if pkt['TCP'].dport == 502 or pkt['TCP'].sport == 502:
                self.packet_counts['Modbus TCP'] += 1
                self.analyze_modbus(pkt)
            
            # OPC UA (Port 4840)
            elif pkt['TCP'].dport == 4840 or pkt['TCP'].sport == 4840:
                self.packet_counts['OPC UA'] += 1
        
        elif pkt.haslayer('UDP'):
            self.packet_counts['UDP'] += 1
        
        elif pkt.haslayer('Ether') and not pkt.haslayer('IP'):
            # Layer 2 Traffic (PROFINET, EtherNet/IP)
            ether_type = pkt['Ether'].type
            if ether_type == 0x8892:  # PROFINET
                self.packet_counts['PROFINET'] += 1
            elif ether_type == 0x88A4:  # EtherCAT
                self.packet_counts['EtherCAT'] += 1
    
    def analyze_modbus(self, pkt):
        """วิเคราะห์ Modbus Transaction"""
        try:
            payload = bytes(pkt['TCP'].payload)
            if len(payload) < 8:
                return
            
            # Modbus TCP Frame: Transaction ID (2) + Protocol (2) + Length (2) + Unit ID (1) + Function (1)
            transaction_id = int.from_bytes(payload[0:2], 'big')
            function_code = payload[7]
            
            if function_code > 0x80:  # Error Response (Function Code + 0x80)
                original_fc = function_code - 0x80
                exception_code = payload[8] if len(payload) > 8 else 0
                
                exception_names = {
                    1: 'Illegal Function',
                    2: 'Illegal Data Address',
                    3: 'Illegal Data Value',
                    4: 'Server Device Failure',
                    5: 'Acknowledge',
                    6: 'Server Device Busy'
                }
                
                error = {
                    'timestamp': datetime.now().isoformat(),
                    'src': pkt['IP'].src if pkt.haslayer('IP') else 'unknown',
                    'original_function': original_fc,
                    'exception': exception_names.get(exception_code, f'Unknown ({exception_code})')
                }
                self.errors.append(error)
                print(f"[MODBUS ERROR] {error['src']}: FC={original_fc}, Exception={error['exception']}")
        
        except Exception as e:
            pass
    
    def print_summary(self):
        """แสดงสรุปผลการวิเคราะห์"""
        print("\n" + "="*60)
        print("NETWORK ANALYSIS SUMMARY")
        print("="*60)
        print("\nPacket Counts by Protocol:")
        for proto, count in self.packet_counts.most_common():
            print(f"  {proto:20s}: {count:6d}")
        
        print(f"\nModbus Errors Found: {len(self.errors)}")
        for err in self.errors[:10]:  # แสดง 10 รายการแรก
            print(f"  [{err['timestamp']}] {err['src']}: {err['exception']}")
```

---

## Step 1923: PLC Memory Analysis

### การวิเคราะห์ Memory ของ Siemens S7

```
Siemens S7-1500 Memory Organization:

Work Memory (RAM):
- Load Memory: Program + Config
- Work Memory: Runtime (Execution)
- Retain Memory: Keep on Power Loss

Addressing:
Input Image (I): I0.0, IB0, IW0, ID0
Output Image (Q): Q0.0, QB0, QW0, QD0
Marker/Flag (M): M0.0, MB0, MW0, MD0
Data Block (DB): DB1.DBX0.0, DB1.DBB0

TIA Portal Diagnostic:

Online Diagnostic:
- CPU → Online & Diagnostics → CPU Memory Usage
- ดู: Work Memory, Load Memory, Retain Memory Usage %

Program Block Analysis:
- System → CPU → Module Parameters → System Diagnostics
- ดู: Scan Time (Min/Max/Current)
- Warning: ถ้า Scan Time Max > 150% ของ Watchdog

Memory Dump (สำหรับ Debug):
- TIA Portal → Online → Watch/Force Variables
- Monitor Address Range โดยตรง

STEP 7 Classic:
- Module Information → Performance Data
- Cycle Time, Free Memory
```

```python
# Python - PLC Memory Monitor ผ่าน OPC UA
import asyncio
from opcua import Client

async def monitor_plc_health():
    """ติดตาม S7-1500 Health ผ่าน OPC UA"""
    
    client = Client("opc.tcp://192.168.1.10:4840")
    
    async with client:
        # อ่าน System Tags
        nodes = {
            'scan_time_ms': client.get_node("ns=0;s=DB1.DBD0"),
            'cpu_load_pct': client.get_node("ns=0;s=DB1.DBD4"),
            'memory_free_pct': client.get_node("ns=0;s=DB1.DBD8"),
            'comm_errors': client.get_node("ns=0;s=DB1.DBD12"),
        }
        
        while True:
            health = {}
            for name, node in nodes.items():
                try:
                    health[name] = await node.read_value()
                except Exception as e:
                    health[name] = None
            
            # ตรวจสอบ Warning Conditions
            if health.get('scan_time_ms', 0) > 100:
                print(f"WARNING: Scan time {health['scan_time_ms']}ms ยาวผิดปกติ!")
            
            if health.get('memory_free_pct', 100) < 20:
                print(f"WARNING: Memory เหลือ {health['memory_free_pct']}% เท่านั้น!")
            
            print(f"PLC Health: {health}")
            await asyncio.sleep(5)
```

---

## Step 1924: EMI/EMC Investigation

### การแก้ปัญหา Electromagnetic Interference

```
EMI Symptoms ใน PLC System:
- I/O ทำงานผิดปกติโดยไม่มีสาเหตุ
- Communication Timeout บ่อย
- PLC Reboot เอง
- Analog Signal Noisy

EMI Sources ใน Industry:
┌─────────────────────────────────────────────────────────┐
│ Source            │ Frequency   │ Impact               │
├─────────────────────────────────────────────────────────┤
│ VFD               │ 1-20kHz     │ Analog, Encoder      │
│ Welding           │ DC/50Hz     │ Digital I/O          │
│ Power Lines       │ 50Hz        │ Ground Loop          │
│ Induction Heater  │ kHz-MHz     │ Communication        │
│ Lightning         │ Transient   │ All                  │
│ RF Transmitter    │ 100MHz+     │ Sensitive Circuits   │
└─────────────────────────────────────────────────────────┘

Diagnostic Tools:
1. Clamp Meter (กระแสไหล Ground)
2. Oscilloscope (Noise Measurement)
3. Spectrum Analyzer (Frequency Analysis)
4. Insulation Tester (Megger)

Standard EMC Solutions:

1. Grounding (สำคัญที่สุด!):
   - Signal Ground แยกจาก Power Ground
   - Star Grounding ที่จุดเดียว
   - ทุก Metal Enclosure ต้อง Ground
   - Ground Loop: แก้ด้วย Isolator
   
2. Cable Routing:
   - Power Cable ห่าง Signal Cable ≥ 30cm
   - Cross Cable ที่ 90° ไม่ขนาน
   - Shielded Cable สำหรับ Analog (ต่อ Shield ด้านเดียว)
   
3. Filtering:
   - Line Filter สำหรับ VFD Power Input
   - EMC Filter สำหรับ PLC Power Supply
   - Ferrite Core บน Signal Cable
   
4. Shielding:
   - Metal Enclosure (ปิดสนิท)
   - Shielded Cable (Foil/Braid)
   - Cable Gland แบบ EMC

5. Surge Protection:
   - MOV (Metal Oxide Varistor) ที่ Power Input
   - TVS Diode ที่ Signal Lines
   - Isolated Transducer สำหรับ Analog
```

---

## Step 1925: Predictive Troubleshooting

### ใช้ Data Analytics ในการ Predict ปัญหา

```python
# Python - Machine Health Predictor
import numpy as np
from sklearn.ensemble import IsolationForest
from datetime import datetime, timedelta

class PredictiveTroubleshootingSystem:
    """
    ระบบ Predictive Troubleshooting ด้วย Machine Learning
    ตรวจจับความผิดปกติก่อนเกิด Failure
    """
    
    def __init__(self, historian_client, notification_service):
        self.historian = historian_client
        self.notify = notification_service
        
        # Model ต่อ Equipment
        self.models = {}
        self.baselines = {}
    
    def train_baseline(self, equipment_id: str, days_history: int = 30):
        """Train Baseline Model จาก Historical Data"""
        
        # ดึงข้อมูลย้อนหลัง 30 วัน
        end = datetime.now()
        start = end - timedelta(days=days_history)
        
        features = self.historian.query_multi([
            f'{equipment_id}.Vibration_X',
            f'{equipment_id}.Vibration_Y',
            f'{equipment_id}.Temperature',
            f'{equipment_id}.Current',
            f'{equipment_id}.Power'
        ], start, end, interval_min=5)
        
        # ใช้ Isolation Forest สำหรับ Anomaly Detection
        model = IsolationForest(
            contamination=0.05,  # คาดว่า 5% เป็น Anomaly ใน Historical
            n_estimators=100,
            random_state=42
        )
        
        X = np.column_stack([features[col] for col in features.columns])
        model.fit(X)
        
        self.models[equipment_id] = model
        print(f"Trained baseline for {equipment_id}")
    
    def predict_health(self, equipment_id: str) -> dict:
        """ประเมิน Health Score แบบ Real-time"""
        
        if equipment_id not in self.models:
            return {'error': 'ยังไม่ได้ Train Model'}
        
        # อ่านค่าปัจจุบัน
        current = {
            'vibration_x': self.historian.get_latest(f'{equipment_id}.Vibration_X'),
            'vibration_y': self.historian.get_latest(f'{equipment_id}.Vibration_Y'),
            'temperature': self.historian.get_latest(f'{equipment_id}.Temperature'),
            'current': self.historian.get_latest(f'{equipment_id}.Current'),
            'power': self.historian.get_latest(f'{equipment_id}.Power')
        }
        
        X = np.array([[
            current['vibration_x'], current['vibration_y'],
            current['temperature'], current['current'], current['power']
        ]])
        
        # Anomaly Score (-1 = Anomaly, +1 = Normal)
        anomaly_score = self.models[equipment_id].score_samples(X)[0]
        
        # แปลงเป็น Health Score 0-100
        # score_samples คืนค่า Negative: -0.1 ≈ Normal, -0.5 ≈ Anomaly
        health_score = max(0, min(100, (anomaly_score + 0.5) * 200))
        
        # ระบุ Features ที่ผิดปกติที่สุด
        anomalous_features = []
        feature_names = ['vibration_x', 'vibration_y', 'temperature', 'current', 'power']
        
        for fname in feature_names:
            baseline_mean = self.baselines.get(f'{equipment_id}.{fname}', {}).get('mean', 0)
            baseline_std = self.baselines.get(f'{equipment_id}.{fname}', {}).get('std', 1)
            z_score = abs(current.get(fname, 0) - baseline_mean) / max(baseline_std, 0.001)
            
            if z_score > 3.0:  # > 3 Sigma = ผิดปกติมาก
                anomalous_features.append({
                    'feature': fname,
                    'current_value': current.get(fname),
                    'z_score': z_score
                })
        
        result = {
            'equipment_id': equipment_id,
            'timestamp': datetime.now().isoformat(),
            'health_score': health_score,
            'anomaly_detected': health_score < 50,
            'anomalous_features': anomalous_features,
            'recommended_action': self.get_recommendation(health_score, anomalous_features)
        }
        
        # แจ้งเตือนถ้า Health ต่ำ
        if health_score < 30:
            self.notify.send_alert(
                f"CRITICAL: {equipment_id} Health Score = {health_score:.0f}/100",
                priority='HIGH'
            )
        elif health_score < 60:
            self.notify.send_alert(
                f"WARNING: {equipment_id} Health Score = {health_score:.0f}/100",
                priority='MEDIUM'
            )
        
        return result
    
    def get_recommendation(self, health_score: float, anomalous_features: list) -> str:
        """แนะนำ Action ตาม Health Score"""
        
        if health_score >= 80:
            return "ปกติ - ไม่ต้องดำเนินการ"
        
        elif health_score >= 60:
            if any('vibration' in f['feature'] for f in anomalous_features):
                return "ตรวจสอบ Bearing Condition ในรอบ PM ถัดไป"
            elif any('temperature' in f['feature'] for f in anomalous_features):
                return "ตรวจสอบ Cooling System และ Lubricant"
            return "เพิ่มความถี่การตรวจสอบ"
        
        elif health_score >= 40:
            return "แนะนำ Shutdown เพื่อตรวจสอบภายใน 48 ชั่วโมง"
        
        else:
            return "หยุดเครื่องทันที! ตรวจสอบและซ่อมแซมก่อนเดินเครื่องต่อ"
```

---

## Step 1926-1940: Advanced Troubleshooting Scenarios

### Case Studies จาก Field

```
CASE 1: PLC หยุดทำงานเป็นระยะๆ

Symptoms:
- CPU เข้า STOP Mode ทุก 2-3 ชั่วโมง
- ไม่มี Error Message ชัดเจน
- หลัง Power Cycle กลับมาทำงานปกติ

Investigation Steps:
1. ตรวจ CPU Diagnostics Buffer → พบ "Communication Error: DP Slave 3"
2. ตรวจ Profibus Network → พบ Noise Spike บน Scope
3. ติดตาม Source ด้วย Current Clamp → ห่างจาก Welding Station
4. ตรวจสอบ Cable Routing → Profibus Cable วิ่งขนาน Power Cable

Root Cause: Welding Machine EMI เข้า Profibus Network
Solution: Re-route Cable + Ferrite Core + Profibus Repeater

CASE 2: Analog Input ค่าไม่นิ่ง (Noisy)

Symptoms:
- Level Transmitter อ่านค่าขึ้นลง ±5% โดยไม่มีเหตุ
- เกิดขึ้นช่วงที่ VFD ใกล้เคียงทำงาน

Investigation:
1. วัดที่ Transmitter Terminal → Signal ปกติ (ค่านิ่ง)
2. วัดที่ PLC AI Module → Noisy
3. ใช้ Oscilloscope → พบ High Frequency Noise 10kHz (= VFD Switching)
4. ตรวจ Cable → ไม่ได้ Shield

Root Cause: VFD Switching Noise บน Unshielded Analog Cable
Solution: เปลี่ยนเป็น Shielded Cable + Grounding ด้านเดียว

CASE 3: Motor ไม่หยุดตาม Command

Symptoms:
- PLC สั่งหยุด Motor แต่ Motor ยังหมุน
- Output Bit ใน PLC = 0 แต่ Motor ยัง ON

Investigation:
1. ตรวจ Output Module → Lamp Output = 0 ✓
2. ตรวจ Wiring ที่ Output Terminal → ไฟ 24V ยังมี!
3. ตรวจ ที่ Relay Coil → Relay ยัง Energized
4. วัด Coil Voltage → 24VDC ยังจ่ายอยู่
5. Trace Wiring → พบ Short Circuit ใน Conduit (สาย 2 เส้นชนกัน)

Root Cause: Wiring Short Circuit ทำให้ Output Bypass
Solution: เปลี่ยน Wiring + เพิ่ม Overcurrent Protection

สิ่งที่ได้เรียนรู้:
Output Bit = 0 ไม่ได้หมายความว่า Load ไม่ทำงาน
ต้องตรวจสอบ Physical State ของ Load ด้วยเสมอ
```

---

## สรุป Part 97

- ✅ Root Cause Analysis (5-Why, Fishbone, FTA)
- ✅ Wireshark สำหรับ OT Network Analysis
- ✅ Python Network Packet Analysis (Scapy)
- ✅ PLC Memory Analysis (Siemens S7)
- ✅ EMI/EMC Investigation + Solutions
- ✅ Predictive Troubleshooting (Isolation Forest ML)
- ✅ Real Case Studies from Field

---

## การนำทาง
- [← Part 96: PLC Interview Questions](part96_plc_interview_questions.md)
- [→ Part 98: Future Technologies](part98_future_technologies.md)
- [📚 สารบัญหลัก](../README.md)
