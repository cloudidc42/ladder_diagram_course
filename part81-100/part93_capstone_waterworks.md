# Part 93: Capstone Project - Water Treatment Plant
## Steps 1841-1860 | ระดับ Capstone

---

## การนำทาง
- [← Part 92: Capstone - Food Processing Plant](part92_capstone_food.md)
- [→ Part 94: Career Path Thailand](part94_career_path_thailand.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1841: Project Overview

### โปรเจกต์: Water Treatment Plant - ระบบผลิตน้ำประปา

```
Project: Municipal Water Treatment Plant
Capacity: 200,000 m³/day
Population Served: 800,000 คน
Location: Metropolitan Waterworks (สมมติ)

Process Flow:
Raw Water → Intake → Pre-Chlorination → Coagulation → Flocculation →
Sedimentation → Filtration → Post-Chlorination → Fluoridation →
Clear Well → Booster Pumps → Distribution Network

Standards:
- WHO Drinking Water Quality Guidelines
- Thai Drinking Water Standard (กรมอนามัย)
- AWWA (American Water Works Association)

Control System:
- Main PLC: Siemens S7-400H (Redundant - High Availability)
- Remote RTU: Siemens S7-300 (at 12 Remote Pump Stations)
- SCADA: Siemens WinCC (Redundant Server)
- Telemetry: 4G/LoRaWAN (for Remote Stations)
- Communication: Fiber Optic Ring (Main Plant) + 4G LTE (Remote)

Uptime Requirement: 99.95% (ไม่เกิน 4.38 ชั่วโมง/ปี)
```

---

## Step 1842: Intake and Pre-Treatment

### ระบบรับน้ำดิบ

```ladder
// ============================================
// Intake Pump Station Control
// Siemens S7-400H (Redundant)
// STL (Statement List)
// ============================================

ORGANIZATION_BLOCK OB1
TITLE = "Main Intake Control"

// ===== Network 1: Level Monitoring =====
L     DB_Intake.Raw_Water_Level_PV      // อ่านระดับน้ำดิบ
T     MD_RawLevel                        // เก็บใน Marker Double Word
L     Raw_Level_HH                       // ระดับ High-High
>=R                                      // เปรียบเทียบ
S     Alarm_Intake_Level_HH             // Set Alarm

L     Raw_Level_LL                       // ระดับ Low-Low
<=R
S     Alarm_Intake_Level_LL

// ===== Network 2: Turbidity Check =====
L     DB_Intake.Turbidity_PV            // ค่า Turbidity (NTU)
L     200.0                             // High Turbidity Limit (200 NTU)
>=R
S     Alarm_High_Turbidity              // แจ้งเตือนน้ำขุ่นมาก
S     Request_Extra_Coagulant           // เพิ่ม Coagulant

// ===== Network 3: Pump Control (3+1 Redundant) =====
CALL FC_Pump_Sequencer
  Total_Flow_Required := Flow_Required_Intake
  Pump1_Status := Pump_IP01_Status
  Pump2_Status := Pump_IP02_Status
  Pump3_Status := Pump_IP03_Status
  Standby_Pump  := Pump_IP04_Status
  Lead_Lag_Mode := Auto_Lead_Lag

// ===== Network 4: Flow Control =====
// PID Loop สำหรับควบคุม Flow ด้วย VFD
CALL FB_PID_Control
  SP := DB_Intake.Flow_Setpoint
  PV := DB_Intake.Flow_Actual
  KP := 2.5
  KI := 0.8
  KD := 0.1
  Output → VFD_Intake_Pump_Speed

// ===== Network 5: Pre-Chlorination =====
// คำนวณ Chlorine Dosage ตาม Flow + Turbidity
L     DB_Intake.Flow_Actual             // m³/hr
L     PRECLOR_BASE_DOSE                 // mg/L (คงที่)
MUL                                     // Flow x Dose
L     3600.0                            // แปลงหน่วย
DIV
T     PreChlor_Required_g_per_hr       // กรัม/ชั่วโมง

// ปรับตาม Turbidity (น้ำขุ่นต้องการ Cl มากขึ้น)
L     DB_Intake.Turbidity_PV
L     50.0                             // NTU ปกติ
>R
JCN   SKIP_TURBIDITY_ADJ
L     DB_Intake.Turbidity_PV
L     50.0
SUB
L     200.0                            // Max Turbidity
DIV
L     0.3                              // Max additional dose (mg/L)
MUL
T     PreChlor_Turbidity_Adj
SKIP_TURBIDITY_ADJ: NOP
```

---

## Step 1843: Coagulation and Flocculation

### Chemical Dosing Control

```python
# Python - Adaptive Coagulant Dosing System
import numpy as np
from scipy.optimize import minimize_scalar

class AdaptiveCoagulantDosing:
    """
    ระบบ Dosing Coagulant อัตโนมัติ
    ปรับปริมาณ Alum (สารส้ม) ตาม Water Quality
    """
    
    def __init__(self, plc_reader, chemical_pump_writer, historian):
        self.plc = plc_reader
        self.pump = chemical_pump_writer
        self.historian = historian
        
        # Jar Test Database (ค่าที่ได้จากการทดลอง Lab)
        self.jar_test_data = []
        
        # Control Parameters
        self.target_turbidity_after_coag = 5.0  # NTU (หลัง Coagulation)
        self.target_settled_turbidity = 2.0      # NTU (หลัง Sedimentation)
        self.min_dose_mg_per_L = 5.0
        self.max_dose_mg_per_L = 50.0
        
        # Model
        self.dose_model = None  # จะ Fit ด้วย Historical Data
    
    async def calculate_optimum_dose(self) -> float:
        """คำนวณ Dose ที่เหมาะสม"""
        
        # อ่านสภาพน้ำปัจจุบัน
        raw_turbidity = await self.plc.read('Intake.Turbidity_NTU')
        raw_ph = await self.plc.read('Intake.pH')
        raw_temp = await self.plc.read('Intake.Temperature_C')
        flow_m3_hr = await self.plc.read('Intake.Flow_m3hr')
        
        # คำนวณ Dose เบื้องต้น (Empirical Formula)
        # Dose = K * Turbidity^0.6 * (pH_factor) * (Temp_factor)
        K = 2.5
        ph_factor = self.get_ph_factor(raw_ph)
        temp_factor = self.get_temp_factor(raw_temp)
        
        base_dose = K * (raw_turbidity ** 0.6) * ph_factor * temp_factor
        
        # Clamp ให้อยู่ใน Range
        dose_mg_L = max(self.min_dose_mg_per_L,
                        min(self.max_dose_mg_per_L, base_dose))
        
        # แปลงเป็น Pump Flow Rate
        dose_g_hr = dose_mg_L * flow_m3_hr * 1000 / 1000  # กรัม/ชั่วโมง
        # สมมติ Alum Solution 10% = 100 g/L
        pump_flow_L_hr = dose_g_hr / 100.0
        pump_flow_mL_min = pump_flow_L_hr * 1000 / 60
        
        return {
            'raw_turbidity': raw_turbidity,
            'raw_ph': raw_ph,
            'dose_mg_L': dose_mg_L,
            'pump_flow_mL_min': pump_flow_mL_min,
            'timestamp': datetime.now().isoformat()
        }
    
    def get_ph_factor(self, ph: float) -> float:
        """pH Factor - Alum ทำงานดีที่ pH 6.5-7.5"""
        if 6.5 <= ph <= 7.5:
            return 1.0
        elif ph < 6.5:
            return 1.0 + (6.5 - ph) * 0.2  # pH ต่ำกว่า 6.5 ต้องการ Dose มากขึ้น
        else:
            return 1.0 + (ph - 7.5) * 0.3   # pH สูงกว่า 7.5 ต้องการ Dose มากขึ้น
    
    def get_temp_factor(self, temp: float) -> float:
        """Temperature Factor - น้ำเย็นต้องการ Dose มากขึ้น"""
        reference_temp = 20.0
        return 1.0 + max(0, (reference_temp - temp) * 0.02)
    
    async def run_control_loop(self):
        """Control Loop สำหรับ Adaptive Dosing"""
        while True:
            dose_info = await self.calculate_optimum_dose()
            
            # ส่งคำสั่งไปยัง Chemical Pump
            await self.pump.write(
                'Alum_Pump.Speed_Setpoint',
                dose_info['pump_flow_mL_min']
            )
            
            # บันทึกลง Historian
            await self.historian.write('Coag.Alum_Dose_mgL', dose_info['dose_mg_L'])
            await self.historian.write('Coag.Alum_PumpFlow', dose_info['pump_flow_mL_min'])
            
            # ตรวจสอบผล (หลัง 30 นาที = Floc ก่อตัว)
            await asyncio.sleep(1800)
            
            settled_turbidity = await self.plc.read('Sedimentation.Outlet_Turbidity')
            if settled_turbidity > self.target_settled_turbidity * 1.5:
                # ผล Settling ไม่ดี เพิ่ม Dose
                await self.adjust_dose_up()
            
            await asyncio.sleep(60)  # ปรับทุก 1 นาที
```

---

## Step 1844: Filtration Control

### Rapid Sand Filter Control

```ladder
// ============================================
// Rapid Sand Filter Control
// Filter #1 จาก 8 Units
// ============================================

// NETWORK 1: Filter Status State Machine
CASE Filter_State OF
  0: IDLE
    IF Filter_Run_Cmd THEN
      Filter_State := 10
    END_IF
    
  10: FILTERING (การกรอง)
    Inlet_Valve := OPEN
    Outlet_Valve := OPEN
    Backwash_Valve := CLOSED
    
    // ตรวจสอบเงื่อนไขต้อง Backwash:
    // 1. Headloss สูง (ตะกอนอุดตัน)
    // 2. เวลาผ่านไป 24 ชั่วโมง
    // 3. Turbidity หลังกรองสูงขึ้น
    
    IF (Headloss_mH2O > MAX_HEADLOSS_2_5_M) OR
       (Filter_Run_Time > T#24H) OR
       (Filter_Outlet_Turbidity > 1.0) THEN
      Filter_State := 20
      Trigger_Backwash := TRUE
    END_IF
    
  20: DRAIN DOWN
    Outlet_Valve := CLOSED
    Drain_Valve := OPEN
    Filter_State_Timer := T#5M
    
    IF Filter_State_Timer ELAPSED THEN
      Filter_State := 30
    END_IF
    
  30: AIR SCOUR (เป่าอากาศ ≈ 5 นาที)
    Air_Blower := ON
    Air_Flow_SP := 60  // m³/m²/hr
    Filter_State_Timer := T#5M
    
    IF Filter_State_Timer ELAPSED THEN
      Air_Blower := OFF
      Filter_State := 40
    END_IF
    
  40: WATER BACKWASH (ล้างย้อน ≈ 10 นาที)
    Backwash_Valve := OPEN
    Backwash_Flow_SP := 36  // m³/m²/hr (กรวดล่อยตัว)
    Waste_Valve := OPEN
    Filter_State_Timer := T#10M
    
    // ติดตาม Backwash Turbidity
    IF Backwash_Turbidity < 20 NTU AND
       Filter_State_Timer > T#5M THEN
      Filter_State := 50  // Backwash สะอาดพอ
    END_IF
    
    IF Filter_State_Timer ELAPSED THEN
      Filter_State := 50
    END_IF
    
  50: RINSE (ล้างหน้า ≈ 5 นาที)
    Backwash_Valve := CLOSED
    Inlet_Valve := OPEN
    Waste_Valve := OPEN  // ยังทิ้งน้ำ
    Filter_State_Timer := T#5M
    
    IF Filter_State_Timer ELAPSED THEN
      Waste_Valve := CLOSED
      Filter_State := 10  // กลับไปกรองปกติ
      Reset_Filter_Run_Time
    END_IF
END_CASE
```

---

## Step 1845: Chlorination and Disinfection

### Chlorine Dosing Control (CT Value)

```python
# Python - Chlorine Residual Control
class ChlorineDisinfectionControl:
    """
    ควบคุม Chlorine Dosing สำหรับ Disinfection
    คำนวณ CT Value (Contact Time)
    """
    
    # WHO/Thai Standard
    MIN_FREE_CHLORINE_DISTRIBUTION = 0.2   # mg/L (ปลายท่อ)
    MAX_FREE_CHLORINE_AT_PLANT = 2.0       # mg/L (ที่โรงงาน)
    CT_REQUIRED_GIARDIA = 65.0             # mg·min/L (pH 7, 10°C)
    
    async def calculate_and_dose(self):
        """คำนวณ Dose และสั่ง Pump"""
        
        # อ่านค่าจาก PLC
        flow_m3_hr = await self.plc.read('ClearWell.Outlet_Flow')
        temp_c = await self.plc.read('ClearWell.Temperature')
        ph = await self.plc.read('ClearWell.pH')
        current_cl = await self.plc.read('ClearWell.FreeChlorine_mgL')
        clearwell_volume = 20000  # m³ (ความจุ Clear Well)
        
        # คำนวณ Contact Time (T10)
        # T10 = 0.7 * Hydraulic Retention Time (HRT)
        hrt_min = (clearwell_volume / flow_m3_hr) * 60  # นาที
        t10_min = hrt_min * 0.7  # T10 = 70% ของ HRT
        
        # CT ที่ได้ปัจจุบัน
        ct_actual = current_cl * t10_min
        
        # CT ที่ต้องการ (ปรับตาม Temperature และ pH)
        ct_required = self.get_ct_required(temp_c, ph)
        
        # คำนวณ Chlorine ที่ต้องการ
        required_cl = ct_required / t10_min
        
        # เพิ่ม Safety Factor และ Demand
        cl_demand = await self.estimate_cl_demand()
        target_cl = required_cl + cl_demand + 0.3  # 0.3 mg/L Safety Buffer
        target_cl = min(target_cl, self.MAX_FREE_CHLORINE_AT_PLANT)
        
        # Dose ที่ต้องการ
        dose_deficit = max(0, target_cl - current_cl)  # mg/L ที่ขาด
        dose_g_hr = dose_deficit * flow_m3_hr * 1000 / 1000  # กรัม/ชั่วโมง
        # สมมติ NaOCl 12% = 120 g/L
        pump_L_hr = dose_g_hr / 120.0
        pump_mL_min = pump_L_hr * 1000 / 60
        
        return {
            'ct_actual': ct_actual,
            'ct_required': ct_required,
            'ct_adequate': ct_actual >= ct_required,
            'target_cl_mgL': target_cl,
            'pump_flow_mL_min': pump_mL_min
        }
    
    def get_ct_required(self, temp_c: float, ph: float) -> float:
        """CT ที่ต้องการตาม WHO Table สำหรับ Giardia 3-log"""
        # ค่าจาก WHO Lookup Table
        ct_table = {
            # (pH range, temp_C): CT value
            (6.0, 5.0): 35, (6.0, 10.0): 26, (6.0, 15.0): 20, (6.0, 20.0): 15,
            (7.0, 5.0): 65, (7.0, 10.0): 49, (7.0, 15.0): 37, (7.0, 20.0): 28,
            (8.0, 5.0): 130,(8.0, 10.0): 98, (8.0, 15.0): 73, (8.0, 20.0): 55,
        }
        # Interpolate (simplified)
        ph_key = round(ph * 2) / 2  # ปัดเป็น 0.5
        ph_key = max(6.0, min(8.0, ph_key))
        temp_key = 5.0 if temp_c <= 5 else 10.0 if temp_c <= 10 else 15.0 if temp_c <= 15 else 20.0
        return ct_table.get((ph_key, temp_key), 49.0)  # Default: pH 7, 10°C
```

---

## Step 1846: Telemetry and Remote Monitoring

### SCADA สำหรับ Water Distribution

```
Plant Network Architecture:
┌──────────────────────────────────────────────────────────────┐
│  WTP Main Plant (โรงผลิตน้ำ)                               │
│  Siemens S7-400H (Redundant) + WinCC Server (Redundant)    │
│  OPC UA Server → Historian → Dashboard                      │
└──────────────────┬───────────────────────────────────────────┘
                   │ Fiber Optic (Main) + 4G LTE (Backup)
         ┌─────────┴─────────┐
         │                   │
┌────────┴────────┐   ┌──────┴────────┐
│ Booster St. A   │   │ Booster St. B  │
│ S7-300 + RTU    │   │ S7-300 + RTU   │
│ 4G LTE          │   │ 4G LTE         │
└────────┬────────┘   └──────┬────────┘
         │                   │
   Remote Stations (12 Stations across city)
   Each: S7-1200 + 4G Router + Solar Power
   Monitoring: Pressure, Flow, Cl Residual, Level
```

```python
# Python - Remote Telemetry Management
class WaterNetworkTelemetry:
    """
    ระบบ Telemetry สำหรับ Water Distribution Network
    ติดตาม 12 Remote Stations
    """
    
    def __init__(self, scada_server, notification_service):
        self.scada = scada_server
        self.notify = notification_service
        
        self.remote_stations = {
            'RS01': {'name': 'สถานีปั๊มน้ำสายใน', 'type': 'BOOSTER', 'capacity_m3hr': 5000},
            'RS02': {'name': 'สถานีปั๊มน้ำสายนอก', 'type': 'BOOSTER', 'capacity_m3hr': 3000},
            'RS03': {'name': 'แทงค์น้ำสูงฝั่งเหนือ', 'type': 'ELEVATED_TANK', 'vol_m3': 2000},
            # ... สถานีอื่น ๆ
        }
        
        # Pressure Management Limits
        self.min_pressure_bar = 1.5   # กฎหมายกำหนด
        self.max_pressure_bar = 7.0   # Pipeline Limit
        self.target_pressure_bar = 3.5
    
    async def monitor_network(self):
        """ติดตาม Network ตลอด 24/7"""
        while True:
            network_status = {}
            
            for station_id, info in self.remote_stations.items():
                try:
                    # อ่านข้อมูลผ่าน 4G Telemetry
                    data = await self.scada.read_rtu(station_id, timeout_sec=10)
                    
                    status = {
                        'online': True,
                        'pressure_bar': data.get('pressure'),
                        'flow_m3hr': data.get('flow'),
                        'cl_residual': data.get('chlorine'),
                        'level_pct': data.get('level'),
                        'pump_running': data.get('pump_status'),
                        'last_update': datetime.now().isoformat()
                    }
                    
                    # ตรวจสอบ Pressure Alarm
                    if status['pressure_bar'] and status['pressure_bar'] < self.min_pressure_bar:
                        await self.raise_pressure_alarm(station_id, status['pressure_bar'])
                    
                    # ตรวจสอบ Chlorine Residual
                    if status['cl_residual'] and status['cl_residual'] < 0.2:
                        await self.raise_chlorine_alarm(station_id, status['cl_residual'])
                    
                    network_status[station_id] = status
                    
                except TimeoutError:
                    # Station ไม่ตอบสนอง
                    network_status[station_id] = {'online': False}
                    await self.raise_communication_alarm(station_id)
            
            # อัปเดต SCADA Dashboard
            await self.scada.update_network_map(network_status)
            
            # คำนวณ Network-Wide KPIs
            await self.calculate_network_kpis(network_status)
            
            await asyncio.sleep(60)  # Poll ทุก 1 นาที
    
    async def raise_pressure_alarm(self, station_id: str, pressure: float):
        """แจ้งเตือน Pressure ต่ำ"""
        await self.notify.send_alarm(
            alarm_id=f'PRESS_LOW_{station_id}',
            priority='HIGH',
            message=f'{station_id}: ความดันน้ำต่ำ {pressure:.1f} บาร์ (ต่ำกว่า 1.5 บาร์)',
            recipients=['duty_engineer', 'operations_manager'],
            action_required='ตรวจสอบ Pump ที่ Station นั้น ปรับ VFD หรือเปิด Standby Pump'
        )
```

---

## Step 1847-1860: 24/7 Unattended Operation

### Alarm Management และ On-Call System

```python
# Python - 24/7 On-Call Management System
class WaterPlant24x7Management:
    """
    ระบบควบคุมโรงงานน้ำ 24/7
    ไม่มีผู้ปฏิบัติงานในช่วงกลางคืน (Unattended)
    """
    
    # Alarm Priority Matrix (ISA-18.2)
    ALARM_PRIORITIES = {
        'P1_CRITICAL': {
            'response_time_min': 5,
            'auto_action': True,
            'escalate_min': 10,
            'notify': ['duty_operator', 'supervisor', 'plant_manager']
        },
        'P2_HIGH': {
            'response_time_min': 15,
            'auto_action': False,
            'escalate_min': 30,
            'notify': ['duty_operator', 'supervisor']
        },
        'P3_MEDIUM': {
            'response_time_min': 60,
            'auto_action': False,
            'escalate_min': 120,
            'notify': ['duty_operator']
        },
    }
    
    async def handle_critical_alarm(self, alarm_data: dict):
        """
        จัดการ Alarm ระดับ Critical
        ระบบจะ:
        1. ส่ง SMS + โทรศัพท์อัตโนมัติ
        2. ทำ Safe State Action อัตโนมัติ
        3. Escalate ถ้าไม่มีการตอบรับ
        """
        alarm_id = alarm_data['alarm_id']
        
        # 1. Automatic Safe-State Action
        auto_action = await self.execute_auto_action(alarm_data)
        
        # 2. Notify On-Call Team
        await self.notify.send_sms_all(
            contacts=self.ALARM_PRIORITIES['P1_CRITICAL']['notify'],
            message=f"CRITICAL ALARM: {alarm_data['description']}\n"
                    f"ค่าที่วัด: {alarm_data['value']} {alarm_data['unit']}\n"
                    f"Action: {auto_action}\n"
                    f"ต้องการ Response ภายใน 5 นาที"
        )
        
        # 3. Auto Phone Call
        await self.notify.phone_call(
            number=self.on_call_operator_phone,
            message=f"Critical Alarm at Water Plant. {alarm_data['description']}. Please respond immediately."
        )
        
        # 4. รอ Acknowledgment 10 นาที แล้ว Escalate
        await asyncio.sleep(600)
        
        if not await self.alarm_acknowledged(alarm_id):
            # Escalate ไปยัง Manager
            await self.notify.phone_call(
                number=self.plant_manager_phone,
                message=f"ESCALATION: Unacknowledged Critical Alarm at Water Plant. "
                        f"Operator not responding. {alarm_data['description']}"
            )
    
    async def execute_auto_action(self, alarm_data: dict) -> str:
        """
        Auto-Action สำหรับ Critical Alarms
        ทำให้ระบบอยู่ในสภาวะปลอดภัยโดยอัตโนมัติ
        """
        alarm_type = alarm_data.get('type')
        
        if alarm_type == 'CHLORINE_FEED_FAILURE':
            # ถ้าระบบ Chlorine ขัดข้อง ให้ Bypass ไป Standby
            await self.plc.write('Chlorine.Switch_To_Standby', True)
            return "Switched to Standby Chlorination System"
        
        elif alarm_type == 'MAIN_PUMP_FAULT':
            # เปิด Standby Pump อัตโนมัติ
            await self.plc.write('Intake.Start_Standby_Pump', True)
            return "Started Standby Intake Pump"
        
        elif alarm_type == 'HIGH_TURBIDITY_BREAKTHROUGH':
            # เพิ่ม Coagulant อัตโนมัติ 20%
            current_dose = await self.plc.read('Coag.Alum_Dose')
            await self.plc.write('Coag.Alum_Dose', current_dose * 1.2)
            return f"Increased Coagulant Dose 20% to {current_dose * 1.2:.1f} mg/L"
        
        elif alarm_type == 'DISTRIBUTION_PRESSURE_LOW':
            # เพิ่ม Speed ของ Booster Pump
            current_speed = await self.plc.read('Booster.VFD_Speed')
            new_speed = min(50.0, current_speed + 5.0)  # Hz
            await self.plc.write('Booster.VFD_Speed_SP', new_speed)
            return f"Increased Booster Pump Speed to {new_speed:.0f} Hz"
        
        return "Manual Intervention Required"
    
    def generate_shift_report(self) -> dict:
        """สร้าง Shift Report อัตโนมัติ"""
        return {
            'date': datetime.now().date().isoformat(),
            'shift': self.get_current_shift(),
            'production_m3': self.historian.query_total('ClearWell.Outlet_Flow'),
            'avg_turbidity_raw': self.historian.query_avg('Intake.Turbidity'),
            'avg_turbidity_finished': self.historian.query_avg('ClearWell.Turbidity'),
            'avg_chlorine': self.historian.query_avg('ClearWell.FreeChlorine'),
            'avg_ph': self.historian.query_avg('ClearWell.pH'),
            'chemicals_used': {
                'alum_kg': self.historian.query_total('Coag.Alum_Used_kg'),
                'chlorine_kg': self.historian.query_total('Chlor.NaOCl_Used_kg'),
                'lime_kg': self.historian.query_total('pH_Adj.Lime_Used_kg'),
            },
            'alarms_count': self.alarm_db.count_alarms_by_shift(),
            'critical_alarms': self.alarm_db.get_critical_alarms(),
            'plant_efficiency_pct': self.calculate_plant_efficiency()
        }
```

---

## Lab Project: Water Quality Monitoring Dashboard

```
Lab Objectives:
1. ออกแบบ SCADA สำหรับ Water Treatment Plant
2. Implement Adaptive Chemical Dosing
3. สร้าง Alarm Management System (ISA-18.2)
4. ทดสอบ 24/7 Unattended Operation Scenarios

Key Points ที่ต้องพิจารณา:
- Redundancy ทุก Critical Component (ปั๊ม, PLC, Network)
- Cybersecurity สำหรับ Critical Infrastructure (IEC 62443)
- Data Integrity สำหรับ Regulatory Compliance (กรมอนามัย)
- Battery Backup (UPS) สำหรับ Control System
- Water Quality Sampling และ Lab Interface
```

---

## สรุป Part 93

โปรเจกต์นี้ครอบคลุม:
- ✅ Intake Control (Turbidity, Pre-Chlorination)
- ✅ Adaptive Coagulant Dosing
- ✅ Sand Filter Control + Backwash Sequence
- ✅ CT Value Calculation สำหรับ Disinfection
- ✅ SCADA Telemetry (12 Remote Stations)
- ✅ 24/7 Unattended Operation
- ✅ Alarm Escalation System
- ✅ Automated Shift Reports

---

## การนำทาง
- [← Part 92: Capstone - Food Processing Plant](part92_capstone_food.md)
- [→ Part 94: Career Path Thailand](part94_career_path_thailand.md)
- [📚 สารบัญหลัก](../README.md)
