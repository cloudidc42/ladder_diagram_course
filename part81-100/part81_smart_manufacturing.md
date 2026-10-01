# Part 81: Smart Manufacturing และ Industry 4.0 Deep Dive
## Steps 1601-1620 | ระดับ Capstone - World Class

---

## การนำทาง
- [← Part 80: Advanced Safety Systems](../part61-80/part80_advanced_safety.md)
- [→ Part 82: Edge Computing สำหรับ PLC Engineer](part82_edge_computing.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1601: Smart Manufacturing คืออะไร?

### ความหมายและวิวัฒนาการ

**Smart Manufacturing** คือการผสมผสานเทคโนโลยีดิจิทัลขั้นสูงเข้ากับกระบวนการผลิต เพื่อสร้างระบบที่สามารถรับรู้ วิเคราะห์ ตัดสินใจ และปรับตัวได้อย่างอัตโนมัติ

```
วิวัฒนาการอุตสาหกรรม:

Industry 1.0  → เครื่องจักรไอน้ำ (ค.ศ. 1760-1840)
Industry 2.0  → ไฟฟ้าและสายการผลิต (ค.ศ. 1870-1970)
Industry 3.0  → คอมพิวเตอร์และระบบอัตโนมัติ (ค.ศ. 1970-2010)
Industry 4.0  → Cyber-Physical Systems, IoT, AI (ค.ศ. 2010-ปัจจุบัน)
Industry 5.0  → Human-Robot Collaboration, Sustainability (อนาคต)
```

### เสาหลัก 9 ประการของ Industry 4.0

```
┌─────────────────────────────────────────────────────────────────┐
│                   9 PILLARS OF INDUSTRY 4.0                     │
├─────────────────────────────────────────────────────────────────┤
│  1. Big Data & Analytics     │  6. Cloud Computing              │
│  2. Autonomous Robots        │  7. Cybersecurity                │
│  3. Simulation               │  8. Additive Manufacturing       │
│  4. Horizontal/Vertical      │  9. Augmented Reality            │
│     Integration              │                                  │
│  5. Industrial IoT (IIoT)    │                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1602: Cyber-Physical Systems (CPS)

### CPS คืออะไร?

**Cyber-Physical Systems** คือระบบที่เชื่อมโยงโลกกายภาพ (Physical) กับโลกดิจิทัล (Cyber) เข้าด้วยกันผ่านเครือข่ายและการประมวลผล

```
┌────────────────────────────────────────────────────────────────────┐
│                   CPS Architecture                                  │
│                                                                    │
│   Physical World          Cyber World                              │
│   ┌──────────┐            ┌──────────────────┐                     │
│   │ Sensors  │────────►   │   Data Collection│                     │
│   │ Actuators│            │   & Processing   │                     │
│   │ Machines │◄────────   │                  │                     │
│   │ Processes│            │   Analytics & AI │                     │
│   └──────────┘            │                  │                     │
│        ▲                  │   Decision Making│                     │
│        │                  └──────────────────┘                     │
│        └──────── Feedback Loop ──────────────┘                    │
└────────────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง CPS ในโรงงาน

```ladder
// CPS - Digital Twin Feedback Control
// เมื่อ Digital Twin ตรวจพบความผิดปกติ จะส่งสัญญาณแจ้งเตือน PLC

|----[DT_Anomaly_Detected]----[NOT Maintenance_Mode]----( Alarm_CPS_001 )----|
|                                                                              |
|----[Alarm_CPS_001]----[Timer T1: 5s]----( Reduce_Speed_10Pct )-------------|
|                                                                              |
|----[Alarm_CPS_001]----[T1_Done]----------( Schedule_Maintenance )----------|
```

---

## Step 1603: Smart Factory Architecture

### สถาปัตยกรรมโรงงานอัจฉริยะ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SMART FACTORY ARCHITECTURE                           │
│                                                                         │
│  Level 5: Enterprise        ┌────────────────────────┐                 │
│  (ERP/Business)             │ SAP / Oracle / MS Dyn  │                 │
│                             └──────────┬─────────────┘                 │
│  Level 4: Operations        ┌──────────▼─────────────┐                 │
│  (MES/SCADA)                │ MES + SCADA + Historian │                 │
│                             └──────────┬─────────────┘                 │
│  Level 3: Control           ┌──────────▼─────────────┐                 │
│  (PLC/DCS)                  │ PLC + HMI + Safety PLC  │                 │
│                             └──────────┬─────────────┘                 │
│  Level 2: Field             ┌──────────▼─────────────┐                 │
│  (Sensors/Actuators)        │ Smart Sensors + Drives  │                 │
│                             └──────────┬─────────────┘                 │
│  Level 1: Physical          ┌──────────▼─────────────┐                 │
│  (Machines)                 │  Physical Equipment     │                 │
│                             └────────────────────────┘                 │
│                                                                         │
│  Horizontal: ◄──── IT/OT Integration Platform ────►                    │
│  Vertical:   ↑ Edge Computing ↑ Cloud Platform ↑                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1604: Digital Twin Technology

### Digital Twin คืออะไร?

**Digital Twin** คือสำเนาดิจิทัลของสิ่งของทางกายภาพ (เครื่องจักร, กระบวนการ, โรงงาน) ที่อัพเดทแบบเรียลไทม์

### ประเภทของ Digital Twin

```
1. Component Twin    - แบบจำลองชิ้นส่วนเดียว (เช่น มอเตอร์, ปั๊ม)
2. Asset Twin        - แบบจำลองเครื่องจักร (เช่น สายพานลำเลียง)
3. System Twin       - แบบจำลองระบบ (เช่น สายการผลิต)
4. Process Twin      - แบบจำลองกระบวนการ (เช่น กระบวนการผสม)
```

### การสร้าง Digital Twin ด้วย PLC Data

```python
# Python - Digital Twin สำหรับ Pump System
# เชื่อมต่อกับ PLC ผ่าน OPC UA

import asyncio
from asyncua import Client
import numpy as np
from datetime import datetime

class PumpDigitalTwin:
    def __init__(self, opc_url, node_ids):
        self.opc_url = opc_url
        self.node_ids = node_ids
        self.state = {
            'speed': 0,
            'flow': 0,
            'pressure': 0,
            'temperature': 0,
            'vibration': 0,
            'health_score': 100
        }
        # โมเดลการสึกหรอ
        self.wear_model = {
            'bearing_wear': 0,      # 0-100%
            'seal_wear': 0,         # 0-100%
            'impeller_wear': 0,     # 0-100%
        }
    
    async def update_from_plc(self):
        """อัพเดทสถานะจาก PLC แบบเรียลไทม์"""
        async with Client(self.opc_url) as client:
            while True:
                # อ่านค่าจาก PLC
                speed_node = client.get_node(self.node_ids['speed'])
                flow_node = client.get_node(self.node_ids['flow'])
                pressure_node = client.get_node(self.node_ids['pressure'])
                vib_node = client.get_node(self.node_ids['vibration'])
                
                self.state['speed'] = await speed_node.get_value()
                self.state['flow'] = await flow_node.get_value()
                self.state['pressure'] = await pressure_node.get_value()
                self.state['vibration'] = await vib_node.get_value()
                
                # คำนวณ Health Score
                self.calculate_health()
                
                await asyncio.sleep(1)  # อัพเดททุก 1 วินาที
    
    def calculate_health(self):
        """คำนวณคะแนนสุขภาพเครื่องจักร"""
        # ตรวจสอบการสั่นสะเทือน
        vib_penalty = max(0, (self.state['vibration'] - 2.0) * 10)
        
        # ตรวจสอบอุณหภูมิ
        temp_penalty = max(0, (self.state['temperature'] - 70) * 2)
        
        # อัพเดทการสึกหรอ
        if self.state['speed'] > 0:
            run_hours = 1/3600  # 1 วินาที
            self.wear_model['bearing_wear'] += run_hours * 0.001
            self.wear_model['seal_wear'] += run_hours * 0.0008
        
        # คำนวณ Health Score
        wear_penalty = sum(self.wear_model.values()) / 3
        self.state['health_score'] = max(0, 100 - vib_penalty - temp_penalty - wear_penalty)
        
        # แจ้งเตือนถ้า Health Score ต่ำ
        if self.state['health_score'] < 70:
            self.trigger_maintenance_alert()
    
    def trigger_maintenance_alert(self):
        """ส่งสัญญาณแจ้งเตือนการบำรุงรักษา"""
        print(f"[{datetime.now()}] MAINTENANCE ALERT!")
        print(f"Health Score: {self.state['health_score']:.1f}%")
        print(f"Bearing Wear: {self.wear_model['bearing_wear']:.2f}%")
```

---

## Step 1605: IIoT (Industrial Internet of Things)

### โครงสร้าง IIoT

```
┌─────────────────────────────────────────────────────────────────────┐
│                     IIoT ARCHITECTURE                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  PERCEPTION LAYER (Layer 1)                                 │   │
│  │  Smart Sensors: Temperature, Pressure, Vibration, Flow      │   │
│  │  RFID, Barcode, Vision, Weight, Level                       │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│  ┌─────────────────────────▼───────────────────────────────────┐   │
│  │  NETWORK LAYER (Layer 2)                                    │   │
│  │  PROFINET, EtherNet/IP, Modbus TCP, OPC UA, MQTT            │   │
│  │  WiFi, Bluetooth, LoRa, 5G                                  │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│  ┌─────────────────────────▼───────────────────────────────────┐   │
│  │  PROCESSING LAYER (Layer 3)                                 │   │
│  │  Edge Computing, Fog Computing, Cloud Computing              │   │
│  │  Data Analytics, AI/ML Processing                           │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│  ┌─────────────────────────▼───────────────────────────────────┐   │
│  │  APPLICATION LAYER (Layer 4)                                │   │
│  │  SCADA, MES, ERP, Predictive Maintenance, Quality Control    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### การตั้งค่า MQTT สำหรับ IIoT

```javascript
// Node.js - MQTT Publisher สำหรับ PLC Data
const mqtt = require('mqtt');

// เชื่อมต่อกับ MQTT Broker
const client = mqtt.connect('mqtt://192.168.1.100:1883', {
    clientId: 'plc_gateway_001',
    username: 'plc_user',
    password: 'secure_password_123',
    keepalive: 60,
    reconnectPeriod: 5000
});

// โครงสร้าง Sparkplug B Topic
const topic_namespace = 'spBv1.0';
const group_id = 'Factory_A';
const edge_node_id = 'PLC_Line_01';

// ฟังก์ชันส่งข้อมูล
function publishPLCData(tag_name, value, timestamp) {
    const topic = `${topic_namespace}/${group_id}/DDATA/${edge_node_id}`;
    const payload = {
        timestamp: timestamp || Date.now(),
        metrics: [{
            name: tag_name,
            timestamp: Date.now(),
            dataType: 'Float',
            value: value
        }],
        seq: sequence_number++
    };
    
    client.publish(topic, JSON.stringify(payload), {
        qos: 1,     // At least once delivery
        retain: false
    });
}

// ส่งข้อมูลทุก 1 วินาที
setInterval(() => {
    publishPLCData('Motor_Speed', readPLCTag('DB10.DBD0'), Date.now());
    publishPLCData('Tank_Level', readPLCTag('DB10.DBD4'), Date.now());
    publishPLCData('Pressure', readPLCTag('DB10.DBD8'), Date.now());
}, 1000);
```

---

## Step 1606: Condition-Based Maintenance with Machine Learning

### หลักการ CBM (Condition-Based Maintenance)

```
Traditional Maintenance:
  Corrective:   เสียแล้วค่อยซ่อม (ต้นทุนสูง, Downtime สูง)
  Preventive:   ซ่อมตามเวลา (สิ้นเปลืองทรัพยากร)
  
Advanced Maintenance:
  Predictive:   พยากรณ์ก่อนเสีย (ML/AI)
  Prescriptive: แนะนำวิธีซ่อมที่ดีที่สุด
```

### ML Model สำหรับการตรวจจับความผิดปกติ

```python
# Python - Anomaly Detection สำหรับ Vibration Data
import numpy as np
import pandas as pd
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import IsolationForest
import joblib

class VibrationAnomalyDetector:
    def __init__(self):
        self.scaler = StandardScaler()
        self.model = IsolationForest(
            contamination=0.05,  # 5% anomaly rate
            n_estimators=100,
            random_state=42
        )
        self.feature_columns = [
            'vibration_x', 'vibration_y', 'vibration_z',
            'rms_velocity', 'peak_acceleration',
            'kurtosis', 'crest_factor', 'temperature'
        ]
    
    def extract_features(self, raw_data: np.ndarray, fs=1000) -> dict:
        """แยกคุณลักษณะจากข้อมูลการสั่นสะเทือน"""
        features = {}
        
        # RMS (Root Mean Square)
        features['rms_velocity'] = np.sqrt(np.mean(raw_data**2))
        
        # Peak Value
        features['peak_acceleration'] = np.max(np.abs(raw_data))
        
        # Kurtosis - ตัวบ่งชี้ความเสียหายของ Bearing
        features['kurtosis'] = pd.Series(raw_data).kurtosis()
        
        # Crest Factor
        if features['rms_velocity'] > 0:
            features['crest_factor'] = features['peak_acceleration'] / features['rms_velocity']
        else:
            features['crest_factor'] = 0
        
        # FFT Analysis - หาความถี่ที่โดดเด่น
        fft_vals = np.abs(np.fft.rfft(raw_data))
        freqs = np.fft.rfftfreq(len(raw_data), 1/fs)
        
        # Bearing Fault Frequencies (BPFI, BPFO, BSF, FTF)
        # สมมติ Bearing SKF 6205: BPFI=5.5xRPM/60, BPFO=3.5xRPM/60
        features['bpfi_amplitude'] = self.get_freq_amplitude(fft_vals, freqs, 275)  # ที่ 3000 RPM
        features['bpfo_amplitude'] = self.get_freq_amplitude(fft_vals, freqs, 175)
        
        return features
    
    def get_freq_amplitude(self, fft_vals, freqs, target_freq, bandwidth=5):
        """หาค่า amplitude ที่ความถี่เป้าหมาย"""
        mask = (freqs >= target_freq - bandwidth) & (freqs <= target_freq + bandwidth)
        if mask.any():
            return np.max(fft_vals[mask])
        return 0
    
    def train(self, normal_data: pd.DataFrame):
        """ฝึกโมเดลด้วยข้อมูลปกติ"""
        features = normal_data[self.feature_columns].values
        features_scaled = self.scaler.fit_transform(features)
        self.model.fit(features_scaled)
        
        # บันทึกโมเดล
        joblib.dump(self.model, 'vibration_model.pkl')
        joblib.dump(self.scaler, 'vibration_scaler.pkl')
        print("โมเดลฝึกเสร็จแล้ว")
    
    def predict(self, new_data: dict) -> dict:
        """ทำนายความผิดปกติ"""
        features = np.array([[new_data[col] for col in self.feature_columns]])
        features_scaled = self.scaler.transform(features)
        
        prediction = self.model.predict(features_scaled)[0]
        anomaly_score = self.model.decision_function(features_scaled)[0]
        
        return {
            'is_anomaly': prediction == -1,
            'anomaly_score': -anomaly_score,  # ยิ่งสูง ยิ่งผิดปกติ
            'severity': self.calculate_severity(-anomaly_score)
        }
    
    def calculate_severity(self, score: float) -> str:
        """ประเมินระดับความรุนแรง"""
        if score < 0.3:
            return 'NORMAL'
        elif score < 0.6:
            return 'WARNING'
        elif score < 0.8:
            return 'ALERT'
        else:
            return 'CRITICAL'
```

---

## Step 1607: Predictive Quality System

### ระบบควบคุมคุณภาพเชิงพยากรณ์

```python
# Python - Predictive Quality สำหรับกระบวนการฉีดพลาสติก
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier, GradientBoostingRegressor
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, mean_squared_error

class PredictiveQualitySystem:
    """ระบบทำนายคุณภาพผลิตภัณฑ์ก่อนผลิต"""
    
    def __init__(self):
        # โมเดลทำนาย Defect
        self.defect_classifier = RandomForestClassifier(
            n_estimators=200,
            max_depth=10,
            random_state=42
        )
        # โมเดลทำนายค่า Quality Index
        self.quality_regressor = GradientBoostingRegressor(
            n_estimators=200,
            learning_rate=0.1,
            max_depth=5
        )
        
        # Process Parameters ที่สำคัญสำหรับการฉีดพลาสติก
        self.process_params = [
            'melt_temperature',          # อุณหภูมิหลอมเหลว (°C)
            'mold_temperature',          # อุณหภูมิแม่พิมพ์ (°C)
            'injection_pressure',        # แรงดันฉีด (bar)
            'injection_speed',           # ความเร็วฉีด (mm/s)
            'holding_pressure',          # แรงดันยึด (bar)
            'holding_time',              # เวลายึด (s)
            'cooling_time',              # เวลาเย็น (s)
            'screw_speed',               # ความเร็วสกรู (rpm)
            'backpressure',              # แรงดันย้อนกลับ (bar)
            'material_moisture'          # ความชื้นวัสดุ (%)
        ]
    
    def train_models(self, historical_data: pd.DataFrame):
        """ฝึกโมเดลด้วยข้อมูลประวัติ"""
        X = historical_data[self.process_params]
        y_defect = historical_data['has_defect']        # 0=ปกติ, 1=มีข้อบกพร่อง
        y_quality = historical_data['quality_index']    # 0-100
        
        # แบ่งข้อมูล
        X_train, X_test, y_train, y_test = train_test_split(
            X, y_defect, test_size=0.2, random_state=42
        )
        
        # ฝึกโมเดล Defect Classifier
        self.defect_classifier.fit(X_train, y_train)
        y_pred = self.defect_classifier.predict(X_test)
        print("Defect Classifier Performance:")
        print(classification_report(y_test, y_pred))
        
        # แสดงความสำคัญของ Parameter
        feature_importance = pd.DataFrame({
            'parameter': self.process_params,
            'importance': self.defect_classifier.feature_importances_
        }).sort_values('importance', ascending=False)
        print("\nTop Parameters ที่ส่งผลต่อคุณภาพ:")
        print(feature_importance.head(5))
        
    def predict_quality(self, current_params: dict) -> dict:
        """ทำนายคุณภาพ ณ เวลาปัจจุบัน"""
        X = pd.DataFrame([current_params])[self.process_params]
        
        defect_prob = self.defect_classifier.predict_proba(X)[0][1]
        quality_index = self.quality_regressor.predict(X)[0]
        
        # คำแนะนำการปรับแต่ง
        recommendations = self.generate_recommendations(current_params, defect_prob)
        
        return {
            'defect_probability': defect_prob,
            'quality_index': quality_index,
            'risk_level': 'HIGH' if defect_prob > 0.5 else 'MEDIUM' if defect_prob > 0.2 else 'LOW',
            'recommendations': recommendations,
            'action': 'STOP_AND_ADJUST' if defect_prob > 0.7 else 'ADJUST' if defect_prob > 0.4 else 'CONTINUE'
        }
    
    def generate_recommendations(self, params: dict, defect_prob: float) -> list:
        """สร้างคำแนะนำการปรับแต่ง"""
        recommendations = []
        
        if params['melt_temperature'] > 280:
            recommendations.append("ลดอุณหภูมิหลอมเหลวลง 5-10°C")
        if params['material_moisture'] > 0.15:
            recommendations.append("อบวัสดุใหม่ให้ความชื้น < 0.1%")
        if params['injection_speed'] > 100:
            recommendations.append("ลดความเร็วฉีดเพื่อลดการเกิดฟองอากาศ")
        
        return recommendations
```

---

## Step 1608: OEE (Overall Equipment Effectiveness)

### สูตรและการคำนวณ OEE

```
OEE = Availability × Performance × Quality

Availability = (Planned Time - Downtime) / Planned Time
Performance  = (Ideal Cycle Time × Total Count) / Operating Time
Quality      = Good Count / Total Count

World Class OEE: ≥ 85%
```

### Ladder Diagram สำหรับ OEE Data Collection

```ladder
// ============================================
// OEE Data Collection - Siemens S7-1500
// ============================================

// NETWORK 1: นับจำนวนชิ้นงานทั้งหมด
|----[Part_Detected_Sensor]----[ONE_SHOT_P]----(ADD DB_OEE.Total_Count, 1)----|

// NETWORK 2: นับชิ้นงานดี (ผ่าน Quality Check)
|----[Quality_OK_Signal]------[ONE_SHOT_P]----(ADD DB_OEE.Good_Count, 1)-----|

// NETWORK 3: ติดตาม Downtime
|----[NOT Machine_Running]---[NOT Planned_Downtime]---( Unplanned_Downtime )-|
|                                                                              |
|----[Unplanned_Downtime]----[TON T_Downtime: 1s]---(ADD DB_OEE.DT_Sec, 1)--|

// NETWORK 4: คำนวณ OEE ทุก 1 นาที
|----[T_Calc_OEE: 1min]-----|
|   CALCULATE:              |
|   Avail = (Planned - DT) / Planned * 100                                    |
|   Perf  = (IdealCycle * Total) / Operating * 100                            |
|   Qual  = Good / Total * 100                                                 |
|   OEE   = Avail * Perf * Qual / 10000                                       |
|--------[Trigger_OEE_Calc]---( Calculate_OEE_FCB )----|
```

### SCL Code สำหรับคำนวณ OEE (Siemens)

```pascal
// SCL - OEE Calculation Function Block
FUNCTION_BLOCK FB_OEE_Calculator
VAR_INPUT
    Planned_Time    : TIME;     // เวลาที่วางแผนผลิต
    Operating_Time  : TIME;     // เวลาที่เครื่องทำงานจริง
    Total_Count     : DINT;     // จำนวนชิ้นงานทั้งหมด
    Good_Count      : DINT;     // จำนวนชิ้นงานที่ดี
    Ideal_Cycle_Time: TIME;     // รอบเวลาที่เหมาะสมที่สุด
END_VAR

VAR_OUTPUT
    OEE             : REAL;     // OEE (%)
    Availability    : REAL;     // Availability (%)
    Performance     : REAL;     // Performance (%)
    Quality         : REAL;     // Quality (%)
    OEE_Category    : STRING;   // 'World Class', 'Good', 'Average', 'Poor'
END_VAR

VAR
    Planned_Sec     : REAL;
    Operating_Sec   : REAL;
    Ideal_Sec       : REAL;
    Downtime        : TIME;
END_VAR

// คำนวณ Availability
Downtime := Planned_Time - Operating_Time;
Planned_Sec := TIME_TO_REAL(Planned_Time) / 1000.0;
Operating_Sec := TIME_TO_REAL(Operating_Time) / 1000.0;

IF Planned_Sec > 0.0 THEN
    Availability := (Operating_Sec / Planned_Sec) * 100.0;
ELSE
    Availability := 0.0;
END_IF;

// คำนวณ Performance
IF Operating_Sec > 0.0 AND Total_Count > 0 THEN
    Ideal_Sec := TIME_TO_REAL(Ideal_Cycle_Time) / 1000.0;
    Performance := ((Ideal_Sec * DINT_TO_REAL(Total_Count)) / Operating_Sec) * 100.0;
    Performance := MIN(100.0, Performance);  // ไม่เกิน 100%
ELSE
    Performance := 0.0;
END_IF;

// คำนวณ Quality
IF Total_Count > 0 THEN
    Quality := (DINT_TO_REAL(Good_Count) / DINT_TO_REAL(Total_Count)) * 100.0;
ELSE
    Quality := 0.0;
END_IF;

// คำนวณ OEE
OEE := (Availability * Performance * Quality) / 10000.0;

// จัดหมวดหมู่ OEE
IF OEE >= 85.0 THEN
    OEE_Category := 'World Class';
ELSIF OEE >= 75.0 THEN
    OEE_Category := 'Good';
ELSIF OEE >= 60.0 THEN
    OEE_Category := 'Average';
ELSE
    OEE_Category := 'Poor - ต้องปรับปรุงด่วน';
END_IF;
```

---

## Step 1609: Autonomous Systems ในโรงงาน

### ระบบอัตโนมัติระดับสูง

```
ระดับของระบบอัตโนมัติ:

Level 0: Manual          - มนุษย์ควบคุมทั้งหมด
Level 1: Assisted        - ระบบช่วยเหลือ แต่มนุษย์ตัดสินใจ
Level 2: Partial Auto    - ระบบควบคุมบางส่วน
Level 3: Conditional     - ระบบควบคุมส่วนใหญ่ มนุษย์คอยดู
Level 4: High Auto       - ระบบควบคุมเกือบทั้งหมด
Level 5: Full Auto       - ระบบควบคุมทุกอย่าง (Lights-out Manufacturing)
```

### Lights-Out Manufacturing

```ladder
// Lights-Out Operation Control
// ตรวจสอบว่าสามารถเข้าสู่โหมด Autonomous ได้หรือไม่

// NETWORK 1: เงื่อนไขเข้า Autonomous Mode
|----[All_Material_OK]----[No_Alarm_Active]----[Tool_Life_OK]-----|
|                                                                   |
|----[Raw_Stock_Sufficient]----[Output_Buffer_Empty]---------------|
|                                                                   |
|                                              ( Enable_Auto_Mode )-|

// NETWORK 2: Auto Mode - ควบคุมการผลิตอัตโนมัติ
|----[Enable_Auto_Mode]----[Production_Schedule_OK]----|
|   CALL Auto_Sequence_Manager                         |
|   Inputs: Schedule_DB, Machine_Status, Tool_Data     |
|   Outputs: Next_Job_ID, Machine_Commands             |
|-----------------------------------------------------|

// NETWORK 3: ตรวจสอบความปลอดภัย 24/7
|----[Auto_Mode_Active]----|
|   CALL Safety_Monitor    |
|   - Check all E-stops    |
|   - Check door sensors   |
|   - Check robot fences   |
|   On violation: Emergency_Stop() + Alert_Operator() |
```

---

## Step 1610: AI/ML ใน Manufacturing

### ประเภทของ AI ในโรงงาน

```
1. Predictive Maintenance AI
   - Vibration Analysis
   - Thermal Imaging Analysis
   - Motor Current Signature Analysis (MCSA)

2. Quality Inspection AI
   - Visual Defect Detection (Computer Vision)
   - Dimensional Measurement
   - Surface Finish Analysis

3. Process Optimization AI
   - Parameter Optimization
   - Energy Optimization
   - Throughput Optimization

4. Supply Chain AI
   - Demand Forecasting
   - Inventory Optimization
   - Supplier Risk Assessment
```

### การเชื่อมต่อ AI กับ PLC

```python
# Python - AI Model Server ที่รับข้อมูลจาก PLC
from flask import Flask, request, jsonify
import joblib
import numpy as np

app = Flask(__name__)

# โหลดโมเดลที่ฝึกแล้ว
model = joblib.load('process_optimizer.pkl')
scaler = joblib.load('process_scaler.pkl')

@app.route('/api/v1/optimize', methods=['POST'])
def optimize_process():
    """รับพารามิเตอร์จาก PLC และส่งค่าเป้าหมายกลับ"""
    data = request.json
    
    # แปลงข้อมูลเป็น Feature Vector
    features = np.array([[
        data['temperature'],
        data['pressure'],
        data['flow_rate'],
        data['viscosity'],
        data['ambient_temp'],
        data['raw_material_grade']
    ]])
    
    # Normalize
    features_scaled = scaler.transform(features)
    
    # ทำนายค่าที่เหมาะสม
    optimal_setpoints = model.predict(features_scaled)[0]
    
    response = {
        'status': 'OK',
        'setpoints': {
            'temperature_sp': float(optimal_setpoints[0]),
            'pressure_sp': float(optimal_setpoints[1]),
            'speed_sp': float(optimal_setpoints[2])
        },
        'predicted_quality': float(optimal_setpoints[3]),
        'confidence': float(optimal_setpoints[4])
    }
    
    return jsonify(response)

@app.route('/api/v1/health', methods=['GET'])
def health_check():
    return jsonify({'status': 'healthy', 'model_version': '2.1.0'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Step 1611-1615: Complete Smart Factory Architecture

### การออกแบบสถาปัตยกรรมโรงงานอัจฉริยะสมบูรณ์

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 COMPLETE SMART FACTORY - ตัวอย่าง โรงงาน PCB               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         CLOUD LAYER                                  │  │
│  │  ┌────────────┐  ┌───────────┐  ┌────────────┐  ┌────────────────┐  │  │
│  │  │   SAP ERP  │  │ Analytics │  │ AI/ML Srv  │  │ Digital Twin   │  │  │
│  │  │            │  │ (Power BI)│  │ (Azure ML) │  │ (Siemens MindS)│  │  │
│  │  └────────────┘  └───────────┘  └────────────┘  └────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                   ▲  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         EDGE LAYER                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │  Industrial PC (Siemens IPC 427E)                           │    │  │
│  │  │  - OPC UA Server                                            │    │  │
│  │  │  - Node-RED Data Flow                                       │    │  │
│  │  │  - InfluxDB (Time Series)                                   │    │  │
│  │  │  - AI Inference Engine                                      │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                   ▲  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                       CONTROL LAYER                                  │  │
│  │                                                                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │  │
│  │  │ Siemens     │  │ Safety PLC  │  │ Siemens     │                 │  │
│  │  │ S7-1500     │  │ S7-1500F    │  │ HMI TP1900  │                 │  │
│  │  │ (Main PLC)  │  │             │  │             │                 │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                   ▲  ▼  (PROFINET)                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        FIELD LAYER                                   │  │
│  │                                                                      │  │
│  │  Solder Paste   Pick & Place    Reflow Oven    AOI Station           │  │
│  │  Printer        Machine         (10 zones)     (Vision AI)           │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐         │  │
│  │  │ IO-Link  │   │ PROFINET │   │ PROFINET │   │ GigE Cam │         │  │
│  │  │ Sensors  │   │ Drive    │   │ Drive    │   │ 20MP x4  │         │  │
│  │  └──────────┘   └──────────┘   └──────────┘   └──────────┘         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1616: Data Architecture ใน Smart Factory

### Unified Namespace (UNS)

**Unified Namespace** คือแนวคิดใหม่ที่ใช้ Message Broker (MQTT) เป็น Single Source of Truth สำหรับข้อมูลทั้งโรงงาน

```
แบบเก่า (Hierarchical, Point-to-Point):
PLC → OPC DA → SCADA → MES → ERP
(แต่ละระดับต้องแปลงข้อมูลใหม่)

แบบใหม่ (Unified Namespace - UNS):
             ┌─────────────────┐
             │  MQTT Broker    │
             │  (Single Source │
             │   of Truth)     │
             └────────┬────────┘
    ┌─────────────────┼─────────────────┐
    ▼                 ▼                 ▼
  PLC (Publisher)  SCADA (Sub)     MES (Sub)
  Sensors (Pub)    Analytics (Sub) ERP (Sub)
  Robots (Pub)     AI Models (Sub) Dashboards (Sub)
```

### Topic Structure สำหรับ UNS

```
UNS Topic Structure (Sparkplug B):
spBv1.0/{group_id}/{message_type}/{edge_node_id}/{device_id}

ตัวอย่าง:
spBv1.0/Factory_A/DDATA/Line_01/PLC_Main
spBv1.0/Factory_A/DDATA/Line_01/Robot_01
spBv1.0/Factory_A/DDATA/Line_02/CNC_01
spBv1.0/Factory_A/DBIRTH/Line_01/PLC_Main  (Device Birth)
spBv1.0/Factory_A/DDEATH/Line_01/PLC_Main  (Device Death)
```

---

## Step 1617: Energy Management System

### ISO 50001 + Smart Manufacturing

```ladder
// ============================================
// Energy Management System
// ============================================

// NETWORK 1: อ่านค่าพลังงานจาก Smart Meter
|----[T_Energy_Read: 1s]-----|
|   CALL Read_Power_Meter    |
|   Node: EM_Line01          |
|   Outputs: KW, KVA, PF, Hz |
|                             |
|   MOVE KW → DB_Energy.Line01_KW   |
|   MOVE KVA → DB_Energy.Line01_KVA |
|   MOVE PF → DB_Energy.Line01_PF   |
|-----------------------------|

// NETWORK 2: คำนวณ Energy Intensity (kWh/unit)
|----[Good_Count_Changed]-----|
|   CALCULATE:                |
|   kWh_Per_Unit = Total_kWh / Good_Count     |
|   Target: 0.5 kWh/unit (World Class)        |
|   STORE → DB_Energy.Energy_Intensity         |
|   IF Energy_Intensity > 0.7 → Alarm_Energy  |
|-----------------------------|

// NETWORK 3: Peak Demand Management
|----[Current_KW > Peak_Limit]----[NOT Production_Critical]---|
|   Shed_Non_Critical_Loads()    |
|   - HVAC_Zone_B: OFF           |
|   - Lighting_Non_Production: OFF|
|   - EV_Chargers: PAUSE         |
|--------------------------------|
```

---

## Step 1618: Cybersecurity สำหรับ Smart Factory

### ICS/OT Security Framework

```
Security Levels (IEC 62443):
SL 0: ไม่มีข้อกำหนดด้านความปลอดภัย
SL 1: ป้องกันการเข้าถึงโดยไม่ตั้งใจ
SL 2: ป้องกัน Hacker ทั่วไป
SL 3: ป้องกัน Sophisticated Attacker
SL 4: ป้องกัน State-Sponsored Attack

Defense in Depth:
Level 1: Physical Security (ล็อค, Badge, CCTV)
Level 2: Network Segmentation (Firewall, DMZ)
Level 3: Authentication (Strong Password, MFA)
Level 4: Application Security (Whitelist, Patch)
Level 5: Monitoring (SIEM, IDS/IPS)
```

### Network Segmentation สำหรับ OT

```
Internet
    │
    ▼
┌──────────┐
│Firewall 1│  (IT/Internet Boundary)
└──────────┘
    │
    ▼
┌──────────┐
│  IT DMZ  │  (Historian, Web, Email Server)
└──────────┘
    │
    ▼
┌──────────┐
│Firewall 2│  (IT/OT Boundary - Data Diode)
└──────────┘
    │
    ▼
┌──────────┐
│  OT DMZ  │  (SCADA, Historian, Jump Server)
└──────────┘
    │
    ▼
┌──────────┐
│Firewall 3│  (OT Boundary)
└──────────┘
    │
    ▼
┌──────────────────────┐
│ OT Network           │  (PLC, HMI, Robot, Drive)
│ 192.168.100.x/24     │
└──────────────────────┘
```

---

## Step 1619: Implementation Roadmap

### แผนการนำ Smart Manufacturing ใช้งาน (18 เดือน)

```
Phase 1: Foundation (เดือน 1-3)
├── ติดตั้ง Network Infrastructure (PROFINET, Fiber Backbone)
├── Deploy OPC UA Server
├── ติดตั้ง Smart Meters และ Sensors
└── สร้าง Data Collection Pipeline

Phase 2: Visibility (เดือน 4-6)
├── Deploy SCADA/Historian (OSIsoft PI หรือ Ignition)
├── Dashboard สำหรับ OEE, Energy, Quality
├── Alert และ Notification System
└── Mobile Access

Phase 3: Analytics (เดือน 7-9)
├── Deploy Edge Computing Platform
├── เริ่มเก็บข้อมูลสำหรับ ML Training
├── Basic Analytics (Trend, Correlation)
└── เชื่อมต่อ MES

Phase 4: Intelligence (เดือน 10-15)
├── Deploy ML Models (Predictive Maintenance)
├── Quality Prediction System
├── AI-assisted Scheduling
└── Digital Twin Phase 1

Phase 5: Optimization (เดือน 16-18)
├── Closed-loop AI Optimization
├── Autonomous Adjustments
├── Full Digital Twin
└── Measure ROI
```

---

## Step 1620: ROI Calculation

### การคำนวณผลตอบแทนการลงทุน

```
ตัวอย่าง: โรงงานผลิตชิ้นส่วนยานยนต์ ขนาดกลาง
กำลังการผลิต: 10,000 ชิ้น/วัน
ราคาต้นทุน: 100 บาท/ชิ้น

การลงทุน (CAPEX):
- Hardware (PLCs, Sensors, Network): 3,500,000 บาท
- Software (SCADA, MES, Edge): 2,000,000 บาท
- Integration & Engineering: 1,500,000 บาท
- Training: 500,000 บาท
รวม CAPEX: 7,500,000 บาท

ผลประโยชน์ต่อปี (Annual Benefits):
1. OEE เพิ่มขึ้น 10% → ผลิตเพิ่ม 365,000 ชิ้น/ปี
   Value = 365,000 × 10 (profit margin) = 3,650,000 บาท
   
2. Scrap ลดลง 50% → ประหยัด 500,000 บาท/ปี

3. Unplanned Downtime ลดลง 30% → ประหยัด 800,000 บาท/ปี

4. Energy ลดลง 15% → ประหยัด 300,000 บาท/ปี

5. ลดแรงงานทางอ้อม 20% → ประหยัด 600,000 บาท/ปี

รวมผลประโยชน์: 5,850,000 บาท/ปี

Payback Period = 7,500,000 / 5,850,000 = 1.28 ปี ✓
ROI 3 ปี = (5,850,000×3 - 7,500,000) / 7,500,000 × 100 = 134%
```

---

## Lab Project: Smart Factory Implementation

### โปรเจกต์: ออกแบบ Smart Factory สำหรับโรงงานอาหาร

**โจทย์**: ออกแบบระบบ Smart Manufacturing สำหรับสายการผลิตบิสกิต 3 สาย

**ข้อกำหนด**:
1. OEE Monitoring แบบ Real-time
2. Predictive Maintenance สำหรับ Oven และ Mixer
3. Quality Vision System
4. Energy Management
5. Integration กับ SAP ERP

**Deliverables**:
1. Network Architecture Diagram
2. Data Flow Diagram
3. SCADA Screen Design
4. ML Model Design
5. ROI Calculation

---

## สรุป Part 81

ใน Part นี้เราได้เรียนรู้:
- ✅ Smart Manufacturing และ Industry 4.0 Framework
- ✅ Cyber-Physical Systems และ Digital Twin
- ✅ IIoT Architecture และ MQTT/Sparkplug B
- ✅ Machine Learning สำหรับ Predictive Maintenance
- ✅ Predictive Quality System
- ✅ OEE Calculation และ Collection
- ✅ Unified Namespace Architecture
- ✅ Cybersecurity สำหรับ OT
- ✅ ROI Calculation

---

## การนำทาง
- [← Part 80: Advanced Safety Systems](../part61-80/part80_advanced_safety.md)
- [→ Part 82: Edge Computing สำหรับ PLC Engineer](part82_edge_computing.md)
- [📚 สารบัญหลัก](../README.md)
