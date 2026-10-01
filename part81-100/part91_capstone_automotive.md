# Part 91: Capstone Project - Automotive Assembly Line
## Steps 1801-1820 | ระดับ Capstone

---

## การนำทาง
- [← Part 90: Advanced Industrial Networking](part90_advanced_networking.md)
- [→ Part 92: Capstone - Food Processing Plant](part92_capstone_food.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1801: Project Overview

### โปรเจกต์: Body Shell Assembly Line - SUV Factory

```
Project Scope:
สายประกอบตัวถังรถยนต์ SUV ตั้งแต่ Underbody จนถึง Body-in-White

Production Rate: 60 units/hour (JPH - Jobs Per Hour)
Shift: 3 Shifts (24 ชั่วโมง)
Target OEE: 90%+

Stations (20 Stations):
01. Underbody Main Frame  11. Rear Pillar
02. Front Floor          12. Rear Quarter Panel
03. Rear Floor           13. Roof Assembly
04. Side Member Left     14. Hood Assembly
05. Side Member Right    15. Door Assembly Left Front
06. Cross Member         16. Door Assembly Left Rear
07. Front Bulkhead       17. Door Assembly Right Front
08. A-Pillar Left/Right  18. Door Assembly Right Rear
09. B-Pillar Left        19. Trunk Lid Assembly
10. B-Pillar Right       20. Geometry Check Station
```

---

## Step 1802: I/O Specification

### I/O List สำหรับ Automotive Assembly Line

```
Overall I/O Summary:
Total Digital Inputs:   1,248 points
Total Digital Outputs:  986 points
Total Analog Inputs:    156 points
Total Analog Outputs:   48 points
Safety Inputs (F-DI):   320 points
Safety Outputs (F-DO):  128 points

Hardware Configuration:
- Main PLC: Siemens S7-1500 CPU 1516F-3 PN/DP (Safety)
- Remote I/O: Siemens ET200SP (Distributed per Station)
- HMI: Siemens TP2200 Comfort (22" per Station)
- Safety Scanner: SICK S3000 (at each Robot)

PLC Rack Assignment:
Rack 0: CPU 1516F + PS + CP (PROFINET)
Rack 1: ET200SP Station 01-05 (via PROFINET RT)
Rack 2: ET200SP Station 06-10
Rack 3: ET200SP Station 11-15
Rack 4: ET200SP Station 16-20
Rack 5: Robot I/O Connections (20 Robots)
```

### Typical Station I/O (Station 09 - B-Pillar Left)

```
Station 09: B-Pillar Left Assembly
Robots: 2x FANUC R-2000iC (Welding)

Digital Inputs (DI):
  DI_0901: Part Present Sensor (Inductive)
  DI_0902: Clamp 1 Closed LS
  DI_0903: Clamp 2 Closed LS
  DI_0904: Clamp 3 Closed LS
  DI_0905: Clamp 4 Closed LS
  DI_0906: Weld Gun 1 Home
  DI_0907: Weld Gun 2 Home
  DI_0908: Electrode Wear Check OK
  DI_0909: Gun Change OK
  DI_0910: E-Stop Station 09
  DI_0911: Light Curtain 1 (Entry)
  DI_0912: Light Curtain 2 (Exit)
  DI_0913: Robot 1 Auto/Ready
  DI_0914: Robot 2 Auto/Ready
  DI_0915: Transfer In Ready
  DI_0916: Transfer Out Ready
  DI_0917: Cooling OK
  DI_0918: Air Pressure OK
  DI_0919: Vision System OK
  DI_0920: Operator Reset
  
Digital Outputs (DO):
  DO_0901: Clamp 1 Close
  DO_0902: Clamp 2 Close
  DO_0903: Clamp 3 Close
  DO_0904: Clamp 4 Close
  DO_0905: Robot 1 Start
  DO_0906: Robot 2 Start
  DO_0907: Transfer Out Conveyor
  DO_0908: Status Light Green
  DO_0909: Status Light Yellow
  DO_0910: Status Light Red
  DO_0911: Alarm Horn
  DO_0912: Transfer In Conveyor
  
Analog Inputs (AI):
  AI_0901: Weld Current Monitor (4-20mA)
  AI_0902: Weld Voltage Monitor (4-20mA)
  AI_0903: Electrode Force (Load Cell, 4-20mA)
  AI_0904: Cooling Water Temp (4-20mA)
```

---

## Step 1803: Network Architecture

### PROFINET Architecture สำหรับ Automotive Line

```
┌─────────────────────────────────────────────────────────────────────────┐
│           AUTOMOTIVE BODY SHOP - NETWORK ARCHITECTURE                   │
│                                                                         │
│  Level 4: Plant Network (IT)                                           │
│  ├── SAP MII (Manufacturing Integration)                               │
│  └── Quality Management System (QDAS, Minitab)                        │
│                                                                         │
│  Level 3: Production Network                                           │
│  ├── Siemens Opcenter MES                                              │
│  ├── OSIsoft PI Historian                                               │
│  └── ANDON System Server                                               │
│                                                                         │
│  IT/OT Boundary: Firewall (Fortinet) + DMZ                            │
│                                                                         │
│  Level 2: Control Network (PROFINET Ring - MRP)                       │
│  Main Switch: Scalance XM400 (Fiber Ring, Redundant)                  │
│  ├── Station 01-05: Scalance XC208 (Fast Ethernet)                    │
│  ├── Station 06-10: Scalance XC208                                     │
│  ├── Station 11-15: Scalance XC208                                     │
│  ├── Station 16-20: Scalance XC208                                     │
│  ├── Robot Area: Scalance XC208EEC (Industrial Grade)                  │
│  └── Vision Systems: Scalance XP208 (Gigabit POE)                     │
│                                                                         │
│  Level 1: Device Network                                               │
│  ├── PLCs (Siemens S7-1500F): PROFINET RT                              │
│  ├── Robots (FANUC): PROFINET RT (Robot Controller)                    │
│  ├── Drives (Siemens SINAMICS): PROFINET RT                            │
│  ├── Safety Scanners (SICK): PROFISAFE                                 │
│  └── HMIs (TP2200): Industrial Ethernet                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1804: Full Ladder Program

### Main Program สำหรับ Station 09

```ladder
// ============================================
// Station 09 - B-Pillar Left Assembly
// Main Control Program
// Siemens S7-1500F
// ============================================

// ============================================
// NETWORK 1: Station Initialization
// ============================================
|----[First_Scan]----( Initialize_Station )----|
|   DB_ST09.Station_ID := 9                    |
|   DB_ST09.Station_Name := 'B_Pillar_Left'   |
|   DB_ST09.Part_Counter := 0                  |
|   DB_ST09.Good_Counter := 0                  |
|--------------------------------------------|

// ============================================
// NETWORK 2: Safety Interlock Check
// ============================================
// ตรวจสอบ Safety Conditions ก่อนอนุญาต Start

|----[NOT F_ESTOP_ST09]----[NOT F_LightCurtain1]----[NOT F_LightCurtain2]----|
|----[NOT F_RobotFault1]----[NOT F_RobotFault2]-----|
|                                               ( Safety_OK_ST09 )-----------|

// ============================================
// NETWORK 3: Parts Arrival Detection
// ============================================
|----[DI_Transfer_In_Ready]----[DI_Part_Present]----[Safety_OK_ST09]---------|
|                                               ( Parts_Ready_ST09 )---------|

// ============================================
// NETWORK 4: Clamp Sequence
// ============================================
|----[Parts_Ready_ST09]----[SM_State = IDLE]----[Start_Cmd]-----------------|
|   CALL FB_Clamp_Sequence_4                                                  |
|   Inputs: Clamp1_Close, Clamp2_Close, Clamp3_Close, Clamp4_Close          |
|   Feedback: Clamp1_FB, Clamp2_FB, Clamp3_FB, Clamp4_FB                    |
|   Timeout: T#5000ms                                                         |
|   Output: All_Clamps_Closed, Clamp_Fault                                   |
|---------------------------------------------------------------------------|

// ============================================
// NETWORK 5: Robot Sequence
// ============================================
|----[All_Clamps_Closed]----[Robot1_Ready]----[Robot2_Ready]-----------------|
|   // ส่ง Start Signal ไปยัง Robot                                          |
|   DO_Robot1_Start := 1                                                      |
|   DO_Robot2_Start := 1                                                      |
|   DB_ST09.Weld_Start_Time := Current_Time                                   |
|---------------------------------------------------------------------------|

// ============================================
// NETWORK 6: Weld Quality Monitoring
// ============================================
|----[Robot1_Welding]----[T_Sample: 100ms]-----|
|   // อ่านค่า Weld Parameters                 |
|   DB_Weld.Current := AI_Weld_Current        |
|   DB_Weld.Voltage := AI_Weld_Voltage        |
|   DB_Weld.Heat_Input := DB_Weld.Current * DB_Weld.Voltage / DB_Weld.Speed |
|   // ตรวจสอบ Limits                          |
|   IF DB_Weld.Current > CURRENT_HI_LIMIT OR  |
|      DB_Weld.Current < CURRENT_LO_LIMIT THEN |
|      SET Weld_Current_OOL                   |
|   END_IF                                     |
|---------------------------------------------|

// ============================================
// NETWORK 7: Completion and Transfer
// ============================================
|----[Robot1_Done]----[Robot2_Done]----[NOT Weld_Fault]------|
|   DO_Robot1_Start := 0                                      |
|   DO_Robot2_Start := 0                                      |
|   // เปิด Clamps                                           |
|   CALL FB_Clamp_Open_Sequence                              |
|   // ส่งต่อไป Station ถัดไป                              |
|   DO_Transfer_Out := 1                                     |
|   ADD DB_ST09.Good_Counter, 1                              |
|   ADD DB_ST09.Part_Counter, 1                              |
|-------------------------------------------------------------|

// ============================================
// NETWORK 8: Fault Handling
// ============================================
|----[Weld_Fault]----( Alarm_ST09_Weld )-------------------------------------|
|----[Clamp_Fault]----( Alarm_ST09_Clamp )-----------------------------------|
|----[Robot1_Fault]----( Alarm_ST09_Robot1 )----------------------------------|
|----[Robot2_Fault]----( Alarm_ST09_Robot2 )----------------------------------|

// ============================================
// NETWORK 9: ANDON System Update
// ============================================
|----[T_ANDON_Update: 1s]-----|
|   WRITE OPC UA Node:        |
|   [ANDON/ST09/Status] := Station_Status_Code  |
|   [ANDON/ST09/OEE] := DB_ST09.OEE            |
|   [ANDON/ST09/Count] := DB_ST09.Good_Counter |
|   [ANDON/ST09/Alarm] := Active_Alarm_Code     |
|-----------------------------|
```

---

## Step 1805: Safety System Design

### Functional Safety สำหรับ Welding Station

```
Safety Function: SF-09-01 (Station 09, Safety Function 1)
Name: Emergency Stop - Robot Area
Category: Cat 3 PL e (ISO 13849-1)
Required: Stop Robot + Open Clamps + Sound Alarm

Safety Components:
1. E-Stop Button (Schmersal ES21): SIL 2, PLe
2. Safety Relay (Pilz PNOZ): SIL 2
3. Safety PLC (S7-1500F): SIL 2, PLe
4. Robot Safety (DCS): SIL 2

PFHD Calculation (Probability of Failure on Demand per Hour):
PFHD target (PLe): < 10E-7 /h

Components PFHD:
- E-Stop Button: 3.0E-8 /h
- Safety Wiring: 1.0E-9 /h
- Safety PLC: 3.5E-9 /h
- Robot Safety: 2.0E-8 /h
Total PFHD: 5.5E-8 /h → OK (< 10E-7)
```

---

## Step 1806: Energy Monitoring

### Energy Management สำหรับ Body Shop

```ladder
// ============================================
// Energy Monitoring System
// ============================================

// NETWORK 1: อ่านค่า Power Meter ทุก Station
|----[T_Energy_Scan: 1s]-----|
|   FOR i := 1 TO 20 DO     |
|      CALL FB_Read_Power_Meter                |
|      Meter_ID := Station_Power_Meter[i]     |
|      Output → DB_Energy.Station[i]          |
|   END_FOR                  |
|   // บันทึก Total Power    |
|   DB_Energy.Total_kW := SUM(DB_Energy.Station[*].kW) |
|-----------------------------|

// NETWORK 2: Peak Demand Management
|----[DB_Energy.Total_kW > PEAK_LIMIT_KW]-----|
|   // ระบุ Non-critical Loads ที่ปิดได้       |
|   IF NOT Production_Critical THEN           |
|      DO_HVAC_Zone_B := 0   // ปิด HVAC      |
|      DO_Compressed_Air_Reserve := 0          |
|   END_IF                                     |
|---------------------------------------------|

// NETWORK 3: คำนวณ Energy Intensity
|----[T_Calc_Energy: 1min]-----|
|   Total_kWh_This_Hour += Total_kW / 60      |
|   Parts_This_Hour = Good_Counter_Hour_End - Good_Counter_Hour_Start |
|   IF Parts_This_Hour > 0 THEN              |
|      Energy_kWh_Per_Unit = Total_kWh_This_Hour / Parts_This_Hour |
|   END_IF                                    |
|-----------------------------|
```

---

## Step 1807: OEE Data Collection

### OEE Dashboard สำหรับ Body Shop

```python
# Python - Automotive Body Shop OEE Dashboard Data Provider
from datetime import datetime, timedelta
import json

class BodyShopOEEDashboard:
    def __init__(self, plc_reader, db_connection):
        self.plc = plc_reader
        self.db = db_connection
        
        # Body Shop Specifications
        self.ideal_jph = 60          # Jobs Per Hour
        self.ideal_cycle_sec = 60    # 60 seconds per station
        self.planned_production_time = 8 * 60  # 480 min per shift
        self.planned_downtime = 30   # Maintenance + Breaks
        
        self.stations = list(range(1, 21))
    
    def get_real_time_oee(self) -> dict:
        """ดึงข้อมูล OEE แบบ Real-time"""
        
        # อ่านข้อมูลจาก PLC
        total_count = self.plc.read('DB_BodyShop.Total_Count')
        good_count = self.plc.read('DB_BodyShop.Good_Count')
        downtime_min = self.plc.read('DB_BodyShop.Downtime_Min')
        shift_start = self.plc.read('DB_BodyShop.Shift_Start_Time')
        
        elapsed_min = (datetime.now() - shift_start).total_seconds() / 60
        operating_time = max(0, elapsed_min - downtime_min)
        available_time = max(0, elapsed_min - self.planned_downtime)
        
        # Calculate OEE Components
        availability = operating_time / available_time if available_time > 0 else 0
        
        expected_count = (elapsed_min - downtime_min) * (self.ideal_jph / 60)
        performance = good_count / expected_count if expected_count > 0 else 0
        performance = min(1.0, performance)
        
        quality = good_count / total_count if total_count > 0 else 0
        
        oee = availability * performance * quality
        
        # Station Status
        station_status = {}
        for stn in self.stations:
            status_code = self.plc.read(f'DB_ST{stn:02d}.Status_Code')
            station_status[f'ST{stn:02d}'] = self.decode_station_status(status_code)
        
        return {
            'timestamp': datetime.now().isoformat(),
            'shift': self.get_current_shift(),
            'oee': round(oee * 100, 1),
            'availability': round(availability * 100, 1),
            'performance': round(performance * 100, 1),
            'quality': round(quality * 100, 1),
            'total_count': total_count,
            'good_count': good_count,
            'downtime_min': downtime_min,
            'jph_actual': total_count / (elapsed_min / 60) if elapsed_min > 0 else 0,
            'jph_target': self.ideal_jph,
            'station_status': station_status,
            'bottleneck_station': self.identify_bottleneck()
        }
    
    def decode_station_status(self, code: int) -> str:
        status_map = {
            0: 'IDLE', 1: 'RUNNING', 2: 'FAULT',
            3: 'WAITING', 4: 'SETUP', 5: 'MAINTENANCE'
        }
        return status_map.get(code, 'UNKNOWN')
    
    def identify_bottleneck(self) -> str:
        """ระบุ Station ที่เป็น Bottleneck"""
        max_queue = 0
        bottleneck = 'NONE'
        
        for stn in self.stations:
            queue_depth = self.plc.read(f'DB_ST{stn:02d}.Parts_Waiting')
            if queue_depth > max_queue:
                max_queue = queue_depth
                bottleneck = f'ST{stn:02d}'
        
        return bottleneck
    
    def get_current_shift(self) -> str:
        hour = datetime.now().hour
        if 6 <= hour < 14: return 'A'
        elif 14 <= hour < 22: return 'B'
        else: return 'C'
```

---

## Step 1808-1820: Testing Procedure

### FAT/SAT Testing สำหรับ Automotive Line

```
FACTORY ACCEPTANCE TEST (FAT):

Test 1: Power-Up Test
  1.1 ตรวจสอบ Wiring สมบูรณ์
  1.2 Insulation Test (Megger Test)
  1.3 Power On Sequence
  1.4 ตรวจสอบ Display/HMI

Test 2: Individual Component Test
  2.1 Digital I/O Test (แต่ละ Point)
  2.2 Analog I/O Calibration
  2.3 Safety Circuit Test
  2.4 Emergency Stop Test (ทุก E-Stop)

Test 3: Station Sequence Test (Dry Run)
  3.1 Sequence โดยไม่มีชิ้นงาน
  3.2 ตรวจสอบ Timing
  3.3 ทดสอบ Error Handling

Test 4: Integration Test
  4.1 Station-to-Station Handshake
  4.2 MES Integration
  4.3 ANDON System
  4.4 Energy Monitoring

Test 5: Performance Test (Wet Run)
  5.1 ผลิต 100 ชิ้น (2 ชั่วโมง)
  5.2 ตรวจสอบ Cycle Time ทุก Station
  5.3 วัด OEE
  5.4 ตรวจสอบ Quality

Test 6: Safety Test
  6.1 Safety Function Test (ISO 13849-1)
  6.2 Reach Level Test
  6.3 Full Production Safety Test

ACCEPTANCE CRITERIA:
- OEE ≥ 90% ใน 2 ชั่วโมง
- Cycle Time ≤ 60s ทุก Station
- Safety Function Response ≤ 100ms
- ไม่มี Unplanned Downtime > 15 นาที
```

---

## สรุป Part 91

โปรเจกต์นี้ครอบคลุม:
- ✅ I/O Specification สมบูรณ์ (2,500+ I/O Points)
- ✅ PROFINET Network Architecture
- ✅ Safety System Design (PLe, SIL 2)
- ✅ Full Ladder Program สำหรับ Station
- ✅ Weld Quality Monitoring
- ✅ Energy Management System
- ✅ Real-time OEE Dashboard
- ✅ FAT/SAT Testing Procedure

---

## การนำทาง
- [← Part 90: Advanced Industrial Networking](part90_advanced_networking.md)
- [→ Part 92: Capstone - Food Processing Plant](part92_capstone_food.md)
- [📚 สารบัญหลัก](../README.md)
