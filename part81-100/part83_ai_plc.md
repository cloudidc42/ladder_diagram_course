# Part 83: AI/ML Integration กับ PLC
## Steps 1641-1660 | ระดับ Capstone

---

## การนำทาง
- [← Part 82: Edge Computing](part82_edge_computing.md)
- [→ Part 84: Cobot Programming](part84_cobot_programming.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1641: AI ใน Industrial Automation

### ทำไม AI ถึงสำคัญสำหรับ PLC Engineer?

```
AI ใน Manufacturing:
┌───────────────────────────────────────────────────────────┐
│  Traditional Control:  Rule-based (If-Then-Else)          │
│  AI/ML Control:        Learning-based (Data-Driven)        │
│                                                           │
│  ตัวอย่าง:                                                │
│  Rule: IF Temp > 85°C THEN Alarm                          │
│  ML:   IF Pattern matches failure_pattern_17 THEN Alert   │
│        (แม้ Temp ยังปกติ แต่ ML เห็น Early Warning)      │
└───────────────────────────────────────────────────────────┘
```

### Use Cases ของ AI ใน PLC Systems

```
1. Predictive Maintenance     → ลด Downtime 30-50%
2. Quality Inspection (Vision)→ แม่นยำกว่า Manual 99.5%+
3. Process Optimization       → ประหยัด Energy 10-20%
4. Anomaly Detection          → ตรวจพบ 95% ก่อนเสีย
5. Demand Forecasting         → ลด Inventory 20-30%
6. Root Cause Analysis        → แก้ปัญหาได้เร็ว 3x
```

---

## Step 1642: Architecture ของ AI + PLC System

### การเชื่อมต่อ AI กับ PLC

```
┌─────────────────────────────────────────────────────────────────┐
│                  AI-PLC INTEGRATION ARCHITECTURE                 │
│                                                                 │
│  Level 1: PLC ควบคุม Real-time (< 100ms)                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Siemens S7-1500                                        │   │
│  │  ├── PID Control Loop (10ms)                           │   │
│  │  ├── Safety Monitoring (1ms)                           │   │
│  │  └── Machine Sequencing (100ms)                        │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│           ↑ Commands    │ Data (OPC UA)                         │
│           └────────┐    ▼                                       │
│  Level 2: Edge AI (1-5 seconds)                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Edge PC (IPC427E)                                      │   │
│  │  ├── Anomaly Detection Model (1s)                       │   │
│  │  ├── Quality Prediction Model (5s)                      │   │
│  │  └── Optimization Engine (30s)                         │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│                         │ (Optimized Setpoints)                  │
│  Level 3: Cloud AI (Minutes-Hours)                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Azure ML / AWS SageMaker                               │   │
│  │  ├── Model Training (Daily/Weekly)                      │   │
│  │  ├── Digital Twin Update                               │   │
│  │  └── Long-term Optimization                            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1643: Anomaly Detection จากข้อมูล PLC

### ประเภทของ Anomaly

```python
# Python - Multi-method Anomaly Detection
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
from scipy import stats
import warnings
warnings.filterwarnings('ignore')

class IndustrialAnomalyDetector:
    """ตรวจจับความผิดปกติแบบหลายวิธี"""
    
    def __init__(self):
        self.methods = {
            'isolation_forest': IsolationForest(contamination=0.05, random_state=42),
            'zscore': None,
            'rolling_stats': None
        }
        self.scaler = StandardScaler()
        self.baselines = {}
        self.history = {}
    
    def train_baseline(self, normal_data: pd.DataFrame, features: list):
        """ฝึกโมเดลด้วยข้อมูลปกติ"""
        self.features = features
        
        # เก็บ Baseline Statistics
        for feature in features:
            self.baselines[feature] = {
                'mean': normal_data[feature].mean(),
                'std': normal_data[feature].std(),
                'q25': normal_data[feature].quantile(0.25),
                'q75': normal_data[feature].quantile(0.75),
                'iqr': normal_data[feature].quantile(0.75) - normal_data[feature].quantile(0.25)
            }
        
        # ฝึก Isolation Forest
        X = normal_data[features].values
        X_scaled = self.scaler.fit_transform(X)
        self.methods['isolation_forest'].fit(X_scaled)
        
        # เริ่ม History Buffer
        for feature in features:
            self.history[feature] = []
        
        print(f"ฝึก Baseline เสร็จ: {len(normal_data)} samples, {len(features)} features")
    
    def detect(self, current_values: dict) -> dict:
        """ตรวจจับความผิดปกติจากค่าปัจจุบัน"""
        anomalies = []
        severity_scores = []
        
        for feature in self.features:
            if feature not in current_values:
                continue
            
            value = current_values[feature]
            baseline = self.baselines[feature]
            
            # Method 1: Z-Score
            z_score = abs((value - baseline['mean']) / (baseline['std'] + 1e-10))
            if z_score > 3.0:
                anomalies.append({
                    'feature': feature,
                    'method': 'Z-Score',
                    'value': value,
                    'z_score': z_score,
                    'message': f"{feature} = {value:.2f} (Z-Score: {z_score:.2f} > 3.0)"
                })
                severity_scores.append(z_score / 3.0)
            
            # Method 2: IQR
            lower_fence = baseline['q25'] - 1.5 * baseline['iqr']
            upper_fence = baseline['q75'] + 1.5 * baseline['iqr']
            
            if value < lower_fence or value > upper_fence:
                anomalies.append({
                    'feature': feature,
                    'method': 'IQR',
                    'value': value,
                    'message': f"{feature} นอก Fence [{lower_fence:.2f}, {upper_fence:.2f}]"
                })
                severity_scores.append(1.5)
        
        # Method 3: Isolation Forest (Multi-variate)
        X = np.array([[current_values[f] for f in self.features]])
        X_scaled = self.scaler.transform(X)
        if_score = self.methods['isolation_forest'].decision_function(X_scaled)[0]
        if_pred = self.methods['isolation_forest'].predict(X_scaled)[0]
        
        if if_pred == -1:
            anomalies.append({
                'feature': 'multivariate',
                'method': 'Isolation Forest',
                'score': -if_score,
                'message': f"Multivariate anomaly detected (score: {-if_score:.3f})"
            })
            severity_scores.append(-if_score * 2)
        
        # คำนวณ Overall Severity
        max_severity = max(severity_scores) if severity_scores else 0
        
        return {
            'is_anomaly': len(anomalies) > 0,
            'anomaly_count': len(anomalies),
            'severity': self.severity_label(max_severity),
            'severity_score': max_severity,
            'anomalies': anomalies,
            'action': self.recommend_action(max_severity, anomalies)
        }
    
    def severity_label(self, score: float) -> str:
        if score < 0.5: return 'NORMAL'
        elif score < 1.0: return 'WARNING'
        elif score < 2.0: return 'ALERT'
        else: return 'CRITICAL'
    
    def recommend_action(self, severity: float, anomalies: list) -> str:
        if severity >= 2.0:
            return 'STOP_MACHINE - แจ้ง Maintenance ทันที'
        elif severity >= 1.0:
            return 'REDUCE_SPEED_50PCT - เฝ้าติดตาม'
        elif severity >= 0.5:
            return 'MONITOR_CLOSELY - ตรวจสอบในรอบถัดไป'
        else:
            return 'CONTINUE_NORMAL'
```

---

## Step 1644: Predictive Maintenance ML Model

### Motor Health Prediction ด้วย LSTM

```python
# Python - LSTM สำหรับ Predictive Maintenance
# ทำนายความเสียหายของมอเตอร์จากข้อมูล Time Series

import numpy as np
import tensorflow as tf
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense, Dropout, BatchNormalization
from tensorflow.keras.callbacks import EarlyStopping, ModelCheckpoint
import pandas as pd
from sklearn.preprocessing import MinMaxScaler

class MotorHealthLSTM:
    """LSTM Model สำหรับทำนายสุขภาพมอเตอร์"""
    
    def __init__(self, sequence_length=60, prediction_horizon=720):
        """
        sequence_length: จำนวนช่วงเวลาย้อนหลัง (60 วินาที)
        prediction_horizon: ทำนายล่วงหน้า (720 วินาที = 12 นาที)
        """
        self.sequence_length = sequence_length
        self.prediction_horizon = prediction_horizon
        self.scaler = MinMaxScaler(feature_range=(0, 1))
        self.model = None
        
        # Features จาก PLC
        self.features = [
            'current_A',          # กระแสไฟฟ้า (A)
            'current_B',
            'current_C',
            'voltage_AB',         # แรงดัน (V)
            'power_factor',       # Power Factor
            'vibration_x',        # การสั่นสะเทือน (mm/s)
            'vibration_y',
            'bearing_temp',       # อุณหภูมิ Bearing (°C)
            'motor_temp',         # อุณหภูมิมอเตอร์ (°C)
            'speed_actual',       # ความเร็วจริง (rpm)
        ]
    
    def build_model(self):
        """สร้าง LSTM Model"""
        self.model = Sequential([
            # LSTM Layer 1
            LSTM(128, input_shape=(self.sequence_length, len(self.features)),
                 return_sequences=True, dropout=0.2),
            BatchNormalization(),
            
            # LSTM Layer 2
            LSTM(64, return_sequences=True, dropout=0.2),
            BatchNormalization(),
            
            # LSTM Layer 3
            LSTM(32, return_sequences=False, dropout=0.2),
            BatchNormalization(),
            
            # Dense Layers
            Dense(64, activation='relu'),
            Dropout(0.2),
            Dense(32, activation='relu'),
            
            # Output: Health Score (0-100), ชั่วโมงก่อนเสีย
            Dense(2, activation='sigmoid')
        ])
        
        self.model.compile(
            optimizer=tf.keras.optimizers.Adam(learning_rate=0.001),
            loss='mse',
            metrics=['mae']
        )
        
        print(self.model.summary())
    
    def prepare_sequences(self, data: pd.DataFrame):
        """เตรียมข้อมูลสำหรับ LSTM (Sliding Window)"""
        X, y = [], []
        
        feature_data = data[self.features].values
        target_data = data['health_score'].values
        
        # Normalize
        feature_scaled = self.scaler.fit_transform(feature_data)
        
        for i in range(self.sequence_length, len(feature_scaled) - self.prediction_horizon):
            X.append(feature_scaled[i-self.sequence_length:i])
            
            # Target: health score ณ เวลา i + prediction_horizon
            future_health = target_data[i + self.prediction_horizon] / 100.0
            time_to_failure = min(1.0, self.calculate_rul(target_data, i) / 720.0)
            y.append([future_health, time_to_failure])
        
        return np.array(X), np.array(y)
    
    def calculate_rul(self, health_scores, current_idx):
        """คำนวณ Remaining Useful Life (RUL) เป็นนาที"""
        threshold = 0.3  # 30% health = failure
        for i in range(current_idx, len(health_scores)):
            if health_scores[i] < threshold * 100:
                return (i - current_idx)  # นาที
        return 720  # มากกว่า 12 ชั่วโมง
    
    def train(self, train_data: pd.DataFrame, val_data: pd.DataFrame):
        """ฝึกโมเดล"""
        self.build_model()
        
        X_train, y_train = self.prepare_sequences(train_data)
        X_val, y_val = self.prepare_sequences(val_data)
        
        callbacks = [
            EarlyStopping(patience=20, restore_best_weights=True, monitor='val_loss'),
            ModelCheckpoint('motor_health_model.h5', save_best_only=True, monitor='val_loss')
        ]
        
        history = self.model.fit(
            X_train, y_train,
            validation_data=(X_val, y_val),
            epochs=200,
            batch_size=64,
            callbacks=callbacks,
            verbose=1
        )
        
        return history
    
    def predict(self, recent_data: pd.DataFrame) -> dict:
        """ทำนายสุขภาพมอเตอร์"""
        if len(recent_data) < self.sequence_length:
            return {'error': f'ต้องการข้อมูลอย่างน้อย {self.sequence_length} samples'}
        
        # เตรียมข้อมูล
        feature_data = recent_data[self.features].tail(self.sequence_length).values
        feature_scaled = self.scaler.transform(feature_data)
        X = feature_scaled.reshape(1, self.sequence_length, len(self.features))
        
        # ทำนาย
        prediction = self.model.predict(X)[0]
        health_score = prediction[0] * 100
        rul_normalized = prediction[1]
        rul_minutes = int(rul_normalized * 720)
        
        return {
            'current_health_score': health_score,
            'predicted_health_12h': health_score * (rul_normalized ** 0.5),
            'remaining_useful_life_minutes': rul_minutes,
            'maintenance_urgency': self.urgency_label(rul_minutes, health_score),
            'recommended_action': self.maintenance_recommendation(rul_minutes, health_score)
        }
    
    def urgency_label(self, rul_min: int, health: float) -> str:
        if health < 40 or rul_min < 60:
            return 'CRITICAL - หยุดทันที'
        elif health < 60 or rul_min < 240:
            return 'HIGH - แผนบำรุงรักษาภายใน 4 ชั่วโมง'
        elif health < 75 or rul_min < 480:
            return 'MEDIUM - แผนบำรุงรักษาใน Shift นี้'
        else:
            return 'LOW - แผนบำรุงรักษาตามปกติ'
    
    def maintenance_recommendation(self, rul_min: int, health: float) -> list:
        recommendations = []
        if health < 50:
            recommendations.append("ตรวจสอบ Bearing: เปลี่ยน Bearing")
            recommendations.append("ตรวจสอบ Coupling: ตรวจสอบการ Alignment")
        if health < 70:
            recommendations.append("ล้างทำความสะอาด Motor Cooling Fan")
            recommendations.append("ตรวจสอบ Insulation Resistance")
        if rul_min < 120:
            recommendations.append("เตรียม Spare Motor สำหรับ Standby")
        return recommendations
```

---

## Step 1645: Quality Prediction Model

### XGBoost สำหรับ Quality Prediction

```python
# Python - XGBoost Quality Prediction สำหรับกระบวนการ Injection Molding
import xgboost as xgb
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import f1_score, classification_report
import shap  # สำหรับ Explainability

class InjectionMoldingQualityPredictor:
    """ทำนายคุณภาพชิ้นงานพลาสติกก่อนผลิต"""
    
    def __init__(self):
        self.model = xgb.XGBClassifier(
            n_estimators=500,
            max_depth=6,
            learning_rate=0.05,
            subsample=0.8,
            colsample_bytree=0.8,
            min_child_weight=5,
            gamma=0.1,
            reg_alpha=0.1,
            reg_lambda=1.0,
            objective='multi:softprob',
            num_class=3,  # 0=ดี, 1=Warning, 2=Scrap
            eval_metric='mlogloss',
            use_label_encoder=False,
            random_state=42
        )
        
        self.explainer = None  # SHAP Explainer สำหรับ Explainability
        
        # Parameters จาก PLC
        self.features = [
            # Material Parameters
            'melt_temp_actual',       # อุณหภูมิหลอมจริง (°C)
            'barrel_zone1_temp',      # อุณหภูมิ Zone 1
            'barrel_zone2_temp',      # อุณหภูมิ Zone 2
            'barrel_zone3_temp',      # อุณหภูมิ Zone 3
            'material_moisture',      # ความชื้นวัสดุ (%)
            
            # Injection Parameters
            'injection_pressure_1st', # แรงดันฉีดรอบ 1 (bar)
            'injection_speed_1st',    # ความเร็วฉีดรอบ 1 (mm/s)
            'injection_pressure_2nd', # แรงดันฉีดรอบ 2
            'injection_time',         # เวลาฉีด (s)
            
            # Packing/Holding Parameters
            'holding_pressure',       # แรงดันยึด (bar)
            'holding_time',           # เวลายึด (s)
            
            # Cooling Parameters
            'mold_temp',              # อุณหภูมิแม่พิมพ์ (°C)
            'cooling_time',           # เวลาระบาย (s)
            'coolant_flow_rate',      # อัตราการไหลสารหล่อเย็น (L/min)
            
            # Machine State
            'screw_position_end',     # ตำแหน่งสกรูสุดท้าย (mm)
            'cushion_position',       # ตำแหน่ง Cushion
            'plasticizing_time',      # เวลาหลอมพลาสติก (s)
            
            # Environmental
            'ambient_temp',           # อุณหภูมิห้อง
            'ambient_humidity',       # ความชื้นสัมพัทธ์
        ]
    
    def train(self, data: pd.DataFrame):
        """ฝึกโมเดล"""
        X = data[self.features]
        y = data['quality_class']  # 0=ดี, 1=Marginal, 2=Scrap
        
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, stratify=y, random_state=42
        )
        
        # Train with Early Stopping
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_test, y_test)],
            early_stopping_rounds=50,
            verbose=100
        )
        
        # Evaluate
        y_pred = self.model.predict(X_test)
        print("Quality Prediction Performance:")
        print(classification_report(y_test, y_pred,
                                     target_names=['Good', 'Warning', 'Scrap']))
        
        # สร้าง SHAP Explainer สำหรับ Explainability
        self.explainer = shap.TreeExplainer(self.model)
    
    def predict_with_explanation(self, current_params: dict) -> dict:
        """ทำนายพร้อมคำอธิบาย (Explainability)"""
        X = pd.DataFrame([current_params])[self.features]
        
        # ทำนาย
        class_pred = self.model.predict(X)[0]
        class_proba = self.model.predict_proba(X)[0]
        
        # คำอธิบายด้วย SHAP
        shap_values = self.explainer.shap_values(X)
        
        # ดู Feature ที่ส่งผลมากที่สุด
        feature_impacts = sorted(
            zip(self.features, shap_values[class_pred][0]),
            key=lambda x: abs(x[1]),
            reverse=True
        )
        
        # แปลเป็น คำแนะนำ
        top_impacts = feature_impacts[:5]
        recommendations = []
        
        for feature, impact in top_impacts:
            if abs(impact) > 0.1:
                direction = "สูงเกินไป" if impact > 0 else "ต่ำเกินไป"
                recommendations.append(f"{feature}: {direction} (Impact: {impact:.3f})")
        
        return {
            'prediction': ['ดี', 'ต้องระวัง', 'Scrap'][class_pred],
            'probability_good': class_proba[0],
            'probability_warning': class_proba[1],
            'probability_scrap': class_proba[2],
            'top_risk_factors': recommendations,
            'action': 'APPROVE' if class_pred == 0 else 'ADJUST' if class_pred == 1 else 'HOLD'
        }
```

---

## Step 1646: Process Optimization ด้วย RL

### Reinforcement Learning สำหรับ Process Control

```python
# Python - Proximal Policy Optimization (PPO) สำหรับ Process Control
# ตัวอย่าง: เพิ่มประสิทธิภาพการควบคุมอุณหภูมิใน Heat Exchanger

import gymnasium as gym
import numpy as np
from gymnasium import spaces

class HeatExchangerEnv(gym.Env):
    """Environment สำหรับ Reinforcement Learning"""
    
    def __init__(self):
        super(HeatExchangerEnv, self).__init__()
        
        # Action Space: ปรับ Setpoint ของ Steam Valve (-1 ถึง +1)
        self.action_space = spaces.Box(
            low=np.array([-1.0]),
            high=np.array([1.0]),
            dtype=np.float32
        )
        
        # Observation Space: [T_outlet, T_inlet, Flow, Valve_Position, SP]
        self.observation_space = spaces.Box(
            low=np.array([0, 0, 0, 0, 0]),
            high=np.array([200, 200, 100, 100, 200]),
            dtype=np.float32
        )
        
        # State Variables
        self.reset()
    
    def reset(self, seed=None):
        """รีเซ็ตสภาพแวดล้อม"""
        super().reset(seed=seed)
        
        self.T_outlet = 50.0    # อุณหภูมิขาออก (°C)
        self.T_inlet = 20.0     # อุณหภูมิขาเข้า (°C)
        self.flow_rate = 50.0   # อัตราการไหล (L/min)
        self.valve_position = 30.0  # ตำแหน่ง Valve (%)
        self.setpoint = np.random.uniform(60, 90)  # SP แบบสุ่ม
        
        self.step_count = 0
        self.max_steps = 300
        
        return self._get_obs(), {}
    
    def _get_obs(self):
        return np.array([
            self.T_outlet,
            self.T_inlet,
            self.flow_rate,
            self.valve_position,
            self.setpoint
        ], dtype=np.float32)
    
    def step(self, action):
        """ดำเนินการ Action"""
        self.step_count += 1
        
        # ปรับ Valve Position
        delta_valve = action[0] * 5  # ปรับได้ ±5% ต่อ Step
        self.valve_position = np.clip(self.valve_position + delta_valve, 0, 100)
        
        # จำลองกระบวนการ (First-order + Deadtime)
        steam_flow = self.valve_position * 0.8  # L/min ของ Steam
        heat_input = steam_flow * 2.26  # kJ (ความร้อนแฝง)
        T_rise = heat_input / (self.flow_rate * 4.18 * 0.1)  # First-order dynamics
        
        # Thermal Lag
        self.T_outlet = self.T_outlet * 0.9 + (self.T_inlet + T_rise) * 0.1
        
        # คำนวณ Reward
        error = abs(self.T_outlet - self.setpoint)
        
        # Reward Function:
        # - ใกล้ Setpoint: +reward
        # - ไกล Setpoint: -penalty
        # - เปลี่ยน Valve เยอะ: -penalty (smooth control)
        reward = -error / 10.0  # ลงโทษตาม Error
        
        if error < 0.5:
            reward += 5.0   # Bonus สำหรับ Tight Control
        elif error < 2.0:
            reward += 2.0
        
        # Penalize Valve Oscillation
        reward -= abs(delta_valve) * 0.1
        
        # ตรวจสอบสิ้นสุด Episode
        terminated = self.step_count >= self.max_steps
        truncated = False
        
        info = {
            'temperature': self.T_outlet,
            'setpoint': self.setpoint,
            'error': error,
            'valve': self.valve_position
        }
        
        return self._get_obs(), reward, terminated, truncated, info
```

---

## Step 1647: Deploying ML Model ไป Edge

### MLOps Pipeline

```python
# Python - Model Deployment Pipeline
# การ Deploy โมเดลจาก Cloud ไป Edge โดยอัตโนมัติ

import os
import json
import hashlib
import requests
import subprocess
from pathlib import Path

class MLModelDeployer:
    """จัดการการ Deploy ML Models ไป Edge Node"""
    
    def __init__(self, edge_node_url: str, auth_token: str):
        self.edge_url = edge_node_url
        self.auth_token = auth_token
        self.model_registry = "https://mlflow.factory-a.com"
        self.local_model_dir = Path("/opt/ml_models")
        self.local_model_dir.mkdir(exist_ok=True)
    
    def check_for_updates(self) -> list:
        """ตรวจสอบว่ามี Model Version ใหม่หรือไม่"""
        updates = []
        
        # ดึงรายการ Model จาก Registry
        response = requests.get(
            f"{self.model_registry}/api/2.0/mlflow/registered-models/list",
            headers={"Authorization": f"Bearer {self.auth_token}"}
        )
        
        for model in response.json().get('registered_models', []):
            model_name = model['name']
            latest_version = model['latest_versions'][-1]
            
            # ตรวจสอบว่า Deployed Version ต่างกันไหม
            current_version = self.get_current_version(model_name)
            
            if latest_version['version'] != current_version:
                updates.append({
                    'name': model_name,
                    'current_version': current_version,
                    'new_version': latest_version['version'],
                    'run_id': latest_version['run_id'],
                    'stage': latest_version['current_stage']
                })
        
        return updates
    
    def download_and_deploy(self, model_info: dict) -> bool:
        """ดาวน์โหลดและ Deploy โมเดลใหม่"""
        model_name = model_info['name']
        version = model_info['new_version']
        
        print(f"กำลัง Deploy {model_name} v{version}...")
        
        # 1. ดาวน์โหลดโมเดล
        model_path = self.local_model_dir / f"{model_name}_v{version}"
        self.download_model(model_info['run_id'], model_path)
        
        # 2. Validate โมเดล
        if not self.validate_model(model_path, model_name):
            print(f"Model validation ล้มเหลว: {model_name} v{version}")
            return False
        
        # 3. Blue-Green Deployment (Deploy ใหม่โดยไม่หยุดของเดิม)
        old_model = self.local_model_dir / f"{model_name}_current"
        new_model = self.local_model_dir / f"{model_name}_new"
        
        # Symlink → New Model
        if new_model.exists():
            new_model.unlink()
        new_model.symlink_to(model_path)
        
        # Health Check ก่อน Switch
        if self.health_check_new_model(new_model, model_name):
            # Switch: Old → Archive, New → Current
            if old_model.exists():
                old_model.rename(self.local_model_dir / f"{model_name}_old")
            new_model.rename(old_model)
            
            # Update Version Record
            self.save_version(model_name, version)
            
            # Notify Inference Service
            self.notify_model_update(model_name)
            
            print(f"Deploy {model_name} v{version} สำเร็จ")
            return True
        else:
            print(f"Health check ล้มเหลว - Rollback")
            new_model.unlink()
            return False
    
    def validate_model(self, model_path: Path, model_name: str) -> bool:
        """ตรวจสอบความถูกต้องของโมเดล"""
        # ตรวจสอบว่า Load ได้
        try:
            import joblib
            model = joblib.load(model_path / "model.pkl")
            
            # ทดสอบ Inference ด้วย Sample Data
            test_data = self.get_test_data(model_name)
            result = model.predict(test_data)
            
            return len(result) > 0
        except Exception as e:
            print(f"Model validation error: {e}")
            return False
    
    def get_current_version(self, model_name: str) -> str:
        """อ่าน Current Version"""
        version_file = self.local_model_dir / f"{model_name}_version.json"
        if version_file.exists():
            with open(version_file) as f:
                return json.load(f).get('version', '0')
        return '0'
    
    def save_version(self, model_name: str, version: str):
        """บันทึก Current Version"""
        version_file = self.local_model_dir / f"{model_name}_version.json"
        with open(version_file, 'w') as f:
            json.dump({'version': version, 'deployed_at': str(pd.Timestamp.now())}, f)
```

---

## Step 1648: AI Feedback to PLC Setpoints

### การส่งผล AI กลับไปยัง PLC

```python
# Python - AI Optimization → PLC Setpoint Writer
# ส่งค่า Optimal Setpoints จาก AI ไปยัง PLC ผ่าน OPC UA

import asyncio
from asyncua import Client
import numpy as np
import logging

logger = logging.getLogger(__name__)

class AIToPLCFeedback:
    """ส่งค่า Setpoints จาก AI ไปยัง PLC"""
    
    def __init__(self, plc_url: str, ai_model_url: str):
        self.plc_url = plc_url
        self.ai_model_url = ai_model_url
        
        # OPC UA Node IDs สำหรับ Setpoints
        self.setpoint_nodes = {
            'temperature_sp': 'ns=3;s=Temperature_Setpoint',
            'pressure_sp': 'ns=3;s=Pressure_Setpoint',
            'flow_sp': 'ns=3;s=Flow_Setpoint',
            'speed_sp': 'ns=3;s=Speed_Setpoint'
        }
        
        # ขอบเขตที่ปลอดภัย (Safety Limits)
        self.safe_limits = {
            'temperature_sp': (50.0, 95.0),    # °C
            'pressure_sp': (1.0, 8.0),          # bar
            'flow_sp': (10.0, 100.0),           # L/min
            'speed_sp': (100.0, 1450.0)         # rpm
        }
        
        # Rate of Change Limits (ไม่ให้เปลี่ยนแบบกระชาก)
        self.max_change_rate = {
            'temperature_sp': 2.0,    # °C/min สูงสุด
            'pressure_sp': 0.5,       # bar/min
            'flow_sp': 5.0,           # L/min/min
            'speed_sp': 50.0          # rpm/min
        }
        
        self.current_setpoints = {}
    
    async def get_current_setpoints(self, client: Client):
        """อ่านค่า Setpoint ปัจจุบันจาก PLC"""
        for name, node_id in self.setpoint_nodes.items():
            try:
                node = client.get_node(node_id)
                self.current_setpoints[name] = await node.get_value()
            except Exception as e:
                logger.error(f"ไม่สามารถอ่าน {name}: {e}")
    
    def validate_setpoints(self, new_setpoints: dict) -> tuple:
        """ตรวจสอบความปลอดภัยของ Setpoints ใหม่"""
        validated = {}
        warnings = []
        
        for name, new_value in new_setpoints.items():
            if name not in self.setpoint_nodes:
                continue
            
            # ตรวจสอบ Safety Limits
            min_val, max_val = self.safe_limits[name]
            
            if new_value < min_val:
                warnings.append(f"{name}: {new_value} < min {min_val} → Clamped")
                new_value = min_val
            elif new_value > max_val:
                warnings.append(f"{name}: {new_value} > max {max_val} → Clamped")
                new_value = max_val
            
            # ตรวจสอบ Rate of Change
            if name in self.current_setpoints:
                current = self.current_setpoints[name]
                change = abs(new_value - current)
                max_change = self.max_change_rate[name]
                
                if change > max_change:
                    # Ramp แทน Step Change
                    direction = 1 if new_value > current else -1
                    new_value = current + direction * max_change
                    warnings.append(f"{name}: Ramped (too fast change)")
            
            validated[name] = new_value
        
        return validated, warnings
    
    async def apply_setpoints_to_plc(self, new_setpoints: dict):
        """เขียน Setpoints ไปยัง PLC"""
        validated, warnings = self.validate_setpoints(new_setpoints)
        
        if warnings:
            logger.warning(f"Setpoint Validation Warnings: {warnings}")
        
        async with Client(self.plc_url) as plc_client:
            # อ่าน Current Setpoints ก่อน
            await self.get_current_setpoints(plc_client)
            
            # เขียน Validated Setpoints
            for name, value in validated.items():
                node_id = self.setpoint_nodes[name]
                try:
                    node = plc_client.get_node(node_id)
                    await node.write_value(float(value))
                    logger.info(f"เขียน {name} = {value:.2f} (เดิม: {self.current_setpoints.get(name, 'N/A')})")
                except Exception as e:
                    logger.error(f"ไม่สามารถเขียน {name}: {e}")
    
    async def optimization_loop(self):
        """Loop หลัก: รัน AI Optimization ทุก 30 วินาที"""
        import aiohttp
        
        logger.info("เริ่ม AI Optimization Loop")
        
        while True:
            try:
                # ดึงข้อมูลกระบวนการปัจจุบัน
                process_data = await self.read_process_data()
                
                # ส่งไป AI Model
                async with aiohttp.ClientSession() as session:
                    async with session.post(
                        f"{self.ai_model_url}/api/v1/optimize",
                        json=process_data,
                        timeout=aiohttp.ClientTimeout(total=10)
                    ) as response:
                        if response.status == 200:
                            result = await response.json()
                            
                            if result.get('confidence', 0) > 0.8:
                                # ความมั่นใจสูงพอ → Apply
                                await self.apply_setpoints_to_plc(result['setpoints'])
                                logger.info(f"Applied AI Setpoints: {result['setpoints']}")
                            else:
                                logger.info(f"AI Confidence ต่ำ ({result.get('confidence')}) - ไม่ Apply")
                
            except Exception as e:
                logger.error(f"Optimization loop error: {e}")
            
            await asyncio.sleep(30)  # รอ 30 วินาที
```

---

## Step 1649-1655: Complete AI-Enhanced Process Control

### Ladder Diagram ที่รับค่าจาก AI

```ladder
// ============================================
// AI-Enhanced Process Control
// Siemens S7-1500 + SCL
// ============================================

// NETWORK 1: เช็ค AI System Status
|----[AI_Heartbeat_Timeout: 60s]----( AI_System_Fault )----------------------|
|                                                                              |
|----[NOT AI_System_Fault]----[AI_Mode_Enable]------( AI_Control_Active )----|

// NETWORK 2: รับ AI Setpoints (ผ่าน Data Block DB_AI_SP)
// AI เขียนค่าเข้า DB_AI_SP ผ่าน OPC UA
// PLC อ่านค่าจาก DB_AI_SP และตรวจสอบ

|----[AI_Control_Active]----[New_AI_Setpoint]------|
|   // ตรวจสอบความปลอดภัยก่อน Apply              |
|   IF DB_AI_SP.Temperature > 95.0 THEN:          |
|      DB_AI_SP.Temperature := 95.0               |
|      SET AI_SP_Clamped                          |
|   END_IF                                        |
|   // Apply Setpoint                             |
|   DB_Control.Temp_SP := DB_AI_SP.Temperature   |
|   DB_Control.Press_SP := DB_AI_SP.Pressure     |
|-------------------------------------------------|

// NETWORK 3: Fallback ถ้า AI ล้มเหลว
|----[AI_System_Fault]----[NOT Manual_Override]----|
|   // ใช้ค่า Setpoint มาตรฐาน                   |
|   DB_Control.Temp_SP := DB_Defaults.Temperature |
|   DB_Control.Press_SP := DB_Defaults.Pressure   |
|   SET AI_Fallback_Active                        |
|   // แจ้งเตือน Operator                        |
|   SET Alarm_AI_Fallback                        |
|-------------------------------------------------|
```

---

## Step 1656-1660: Lab Project

### โปรเจกต์: AI-Powered Quality Control System

**โจทย์**: สร้างระบบควบคุมคุณภาพ AI สำหรับกระบวนการเชื่อมโลหะ (MIG Welding)

**Architecture**:
```
PLC (Weld Parameters) → Edge PC → AI Model → Weld Quality Prediction
                                ↓
                        Real-time Adjustment
                                ↓
                        PLC Setpoints Update
```

**Parameters ที่วัด**:
- Welding Current (A)
- Welding Voltage (V)
- Wire Feed Speed (m/min)
- Travel Speed (mm/min)
- Shielding Gas Flow (L/min)
- Base Metal Temperature (°C)

**Defect Classes**:
- 0 = Good Weld
- 1 = Porosity (ฟองอากาศ)
- 2 = Undercut (ร่องบ่า)
- 3 = Spatter Excess (ควันกระจาย)
- 4 = Incomplete Fusion (การหลอมไม่สมบูรณ์)

**Lab Tasks**:
1. เก็บข้อมูล Welding Parameters + Quality Labels (ขั้นต่ำ 1,000 samples)
2. สร้าง Feature Engineering (Heat Input = V×A/Speed)
3. Train XGBoost Classifier
4. Deploy บน Edge PC
5. สร้าง Feedback Loop กับ PLC

---

## สรุป Part 83

ใน Part นี้เราได้เรียนรู้:
- ✅ AI/ML Architecture ในระบบ Industrial
- ✅ Anomaly Detection (Multi-method)
- ✅ LSTM สำหรับ Predictive Maintenance
- ✅ XGBoost + SHAP สำหรับ Quality Prediction
- ✅ Reinforcement Learning สำหรับ Process Control
- ✅ MLOps Pipeline (Model Deployment)
- ✅ AI Feedback → PLC Setpoints (พร้อม Safety Validation)

---

## การนำทาง
- [← Part 82: Edge Computing](part82_edge_computing.md)
- [→ Part 84: Cobot Programming](part84_cobot_programming.md)
- [📚 สารบัญหลัก](../README.md)
