# Part 85: AGV และ AMR Integration
## Steps 1681-1700 | ระดับ Capstone

---

## การนำทาง
- [← Part 84: Cobot Programming](part84_cobot_programming.md)
- [→ Part 86: Advanced Machine Vision](part86_industrial_vision_advanced.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1681: AGV vs AMR

### ความแตกต่างที่สำคัญ

```
┌──────────────────────────────────────────────────────────────────────┐
│                    AGV vs AMR COMPARISON                             │
├──────────────────────┬──────────────────────┬────────────────────────┤
│ Feature              │ AGV                  │ AMR                    │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Navigation           │ Fixed Path           │ Dynamic Path           │
│ Infrastructure       │ Magnetic Tape/Wire   │ ไม่ต้องการ             │
│ Obstacle Handling    │ หยุดรอ               │ หลบอัตโนมัติ           │
│ Reprogramming        │ ยาก/แพง              │ ง่าย (Software)        │
│ Flexibility          │ ต่ำ                  │ สูง                    │
│ Initial Cost         │ สูง (Infrastructure) │ ต่ำ                    │
│ AI Required          │ ไม่จำเป็น            │ จำเป็น                 │
│ Typical Use          │ Fixed Route Transport│ Flexible Material Flow │
│ Examples             │ Toyota Forklift AGV  │ Boston Dynamics Spot   │
│                      │ Jungheinrich WMS     │ MiR100/250, Fetch      │
└──────────────────────┴──────────────────────┴────────────────────────┘
```

---

## Step 1682: ประเภทของ AGV

### AGV Types ในอุตสาหกรรม

```
1. Laser-Guided AGV (LGV)
   - ใช้ Laser Scanner + Reflectors บนผนัง
   - ความแม่นยำ: ±5mm
   - ความเร็ว: 1-2 m/s
   - ตัวอย่าง: Dematic, JBT Corporation

2. Magnetic Tape/Inductive AGV
   - ใช้ Magnetic Tape ฝังในพื้น
   - ราคาถูก, ง่ายในการติดตั้ง
   - เปลี่ยนเส้นทางยาก
   - ตัวอย่าง: Creform, Jervis B. Webb

3. Vision-Guided AGV
   - ใช้กล้อง + AI ในการ Navigate
   - ไม่ต้องมี Infrastructure
   - ตัวอย่าง: Amazon Kiva, Autonomous Solutions

4. Forklift AGV
   - ยกของหนัก 1-10 ตัน
   - Narrow Aisle Operation
   - ตัวอย่าง: Toyota L-AX, Jungheinrich ERC 215a

5. Tugger AGV
   - ลาก Cart หลายคัน
   - ประสิทธิภาพสูงในการขนส่ง
   - ตัวอย่าง: Tec-Systems TugBot
```

---

## Step 1683: Fleet Management System

### FMS (Fleet Management System) Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FLEET MANAGEMENT SYSTEM                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Fleet Manager Software (Server)                            │  │
│  │  ├── Task Dispatcher                                        │  │
│  │  ├── Traffic Manager                                        │  │
│  │  ├── Map Manager                                            │  │
│  │  ├── Charge Scheduler                                       │  │
│  │  └── Dashboard & Reporting                                  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                         │ WiFi/5G                                   │
│         ┌───────────────┼───────────────┐                          │
│         ▼               ▼               ▼                          │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                      │
│   │  AGV 01  │   │  AGV 02  │   │  AGV 03  │                      │
│   │ (Busy)   │   │  (Idle)  │   │(Charging)│                      │
│   └──────────┘   └──────────┘   └──────────┘                      │
│                                                                     │
│  Call Stations (PLC → FMS):                                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                      │
│  │Station A │   │Station B │   │Station C │                      │
│  │(Request) │   │(Ready)   │   │(Full)    │                      │
│  └──────────┘   └──────────┘   └──────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 1684: VDA 5050 Standard

### VDA 5050 - Interface AGV กับ FMS

**VDA 5050** คือ Standard สำหรับการสื่อสารระหว่าง AGV/AMR กับ Fleet Management System โดย Volkswagen Group และ VDMA

```
VDA 5050 Message Topics (MQTT):

Publisher (FMS → AGV):
  /uagv/v2/{manufacturer}/{serial_number}/order
  /uagv/v2/{manufacturer}/{serial_number}/instantActions

Publisher (AGV → FMS):
  /uagv/v2/{manufacturer}/{serial_number}/state
  /uagv/v2/{manufacturer}/{serial_number}/visualization
  /uagv/v2/{manufacturer}/{serial_number}/connection
```

### VDA 5050 Order Structure

```python
# Python - VDA 5050 Order Message
import json
from datetime import datetime
import uuid

def create_agv_order(agv_serial: str, source_station: str, dest_station: str) -> dict:
    """สร้าง Order ตาม VDA 5050 Standard"""
    
    order = {
        "headerId": uuid.uuid4().int & 0xFFFFFF,
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "version": "2.0.0",
        "manufacturer": "MiR",
        "serialNumber": agv_serial,
        
        "orderId": f"ORDER_{uuid.uuid4().hex[:8].upper()}",
        "orderUpdateId": 0,
        
        # Nodes - จุดที่ Robot ต้องผ่าน
        "nodes": [
            {
                "nodeId": source_station,
                "sequenceId": 0,
                "released": True,
                "nodePosition": {
                    "x": 5.0, "y": 3.0, "theta": 0.0,
                    "mapId": "FACTORY_FLOOR_01"
                },
                "actions": [
                    {
                        "actionType": "pick",
                        "actionId": f"PICK_{uuid.uuid4().hex[:6]}",
                        "blockingType": "HARD",
                        "actionParameters": [
                            {"key": "stationType", "value": "conveyor"},
                            {"key": "loadType", "value": "pallet"}
                        ]
                    }
                ]
            },
            {
                "nodeId": dest_station,
                "sequenceId": 2,
                "released": True,
                "nodePosition": {
                    "x": 12.0, "y": 8.0, "theta": 1.5708,  # 90 degrees
                    "mapId": "FACTORY_FLOOR_01"
                },
                "actions": [
                    {
                        "actionType": "drop",
                        "actionId": f"DROP_{uuid.uuid4().hex[:6]}",
                        "blockingType": "HARD",
                        "actionParameters": [
                            {"key": "stationType", "value": "roller_conveyor"}
                        ]
                    }
                ]
            }
        ],
        
        # Edges - เส้นทางระหว่าง Nodes
        "edges": [
            {
                "edgeId": f"EDGE_001",
                "sequenceId": 1,
                "startNodeId": source_station,
                "endNodeId": dest_station,
                "released": True,
                "maxSpeed": 1.5,        # m/s
                "maxHeight": 2.5,       # m
                "minHeight": 0.0,       # m
                "orientation": 1.0,
                "rotationAllowed": True,
                "maxRotationSpeed": 0.5  # rad/s
            }
        ]
    }
    
    return order

# ส่ง Order ผ่าน MQTT
import paho.mqtt.client as mqtt

def send_agv_order(broker_ip: str, order: dict):
    client = mqtt.Client()
    client.connect(broker_ip, 1883, 60)
    
    topic = f"/uagv/v2/{order['manufacturer']}/{order['serialNumber']}/order"
    payload = json.dumps(order)
    
    client.publish(topic, payload, qos=1, retain=False)
    client.disconnect()
    print(f"ส่ง Order {order['orderId']} ไปยัง {order['serialNumber']}")
```

---

## Step 1685: PLC-AGV Communication

### การส่ง Request จาก PLC ไป FMS

```ladder
// ============================================
// AGV Call Station Control
// Siemens S7-1500 + OPC UA → FMS
// ============================================

// NETWORK 1: ตรวจสอบว่าต้องการ AGV
// เมื่อ Conveyor Output เต็ม → ขอ AGV มารับ

|----[Conveyor_Output_Full]----[NOT AGV_Called]----[NOT AGV_Present]---------|
|                                                   ( Call_AGV_Station_A )----|

// NETWORK 2: ส่งคำขอไป FMS ผ่าน OPC UA
|----[Call_AGV_Station_A]----[T_Debounce: 2s]-----|
|   WRITE to OPC UA Node:                          |
|   [Factory/Stations/StationA/AGV_Request] := 1  |
|   [Factory/Stations/StationA/Load_Type] := "PALLET" |
|   [Factory/Stations/StationA/Priority] := 2     |
|-------------------------------------------------|

// NETWORK 3: AGV มาถึง Station
// AGV ส่ง Signal ผ่าน WiFi → PLC รับผ่าน FMS

|----[AGV_Arrived_StationA]----( AGV_Present_StationA )---------------------|
|----[AGV_Present_StationA]----( Stop_Conveyor_Outfeed )---------------------|
|----[AGV_Present_StationA]----( Extend_Transfer_Table )---------------------|

// NETWORK 4: โอนสินค้าขึ้น AGV
|----[Transfer_Table_Extended]----[AGV_Dock_OK]------|
|   // ดันสินค้าขึ้น AGV                             |
|   Pusher_Forward := 1                             |
|                                                   |
|----[Pusher_Fwd_LS]----( Pusher_Reverse_Cmd )-------|
|----[Pusher_Rev_LS]----[Transfer_Table_Retract]-----|
|   Transfer_Complete := 1                          |
|---------------------------------------------------|

// NETWORK 5: AGV ออกเดินทาง
|----[Transfer_Complete]----( AGV_Depart_Cmd_StationA )---------------------|
|   // ส่งสัญญาณให้ AGV ออกเดินทาง                                          |
|   WRITE [Factory/Stations/StationA/AGV_Request] := 0                      |
|   WRITE [Factory/Stations/StationA/Transfer_Done] := 1                    |

// NETWORK 6: Reset เมื่อ AGV ออกไปแล้ว
|----[AGV_Departed_StationA]----( Reset_Station_A )--------------------------|
|   Call_AGV_Station_A := 0                                                  |
|   AGV_Present_StationA := 0                                               |
|   Transfer_Complete := 0                                                   |
```

---

## Step 1686: Traffic Management

### การจัดการ Traffic สำหรับ AGV Fleet

```python
# Python - Traffic Manager สำหรับ AGV Fleet
# ป้องกัน Deadlock และ Collision

from typing import Dict, List, Optional
import threading
import time
from collections import defaultdict

class AGVTrafficManager:
    """จัดการ Traffic สำหรับ AGV Fleet"""
    
    def __init__(self, map_file: str):
        self.zones = {}          # {zone_id: AGV_in_zone}
        self.zone_capacity = {}  # {zone_id: max_vehicles}
        self.lock = threading.Lock()
        self.agv_positions = {}  # {agv_id: current_zone}
        self.agv_reservations = {}  # {agv_id: [reserved_zones]}
        
        self.load_map(map_file)
    
    def load_map(self, map_file: str):
        """โหลด Map และกำหนด Zone"""
        # ตัวอย่าง Zones
        self.zone_capacity = {
            'ZONE_A': 1,   # เส้นทางแคบ - 1 คันเท่านั้น
            'ZONE_B': 2,   # ทางเดินกว้าง - 2 คัน
            'ZONE_C': 1,   # Intersection - 1 คัน
            'ZONE_D': 1,
            'CHARGING_AREA': 4,  # พื้นที่ชาร์จ - 4 คัน
        }
        
        # Connectivity: {zone: [adjacent_zones]}
        self.zone_graph = {
            'ZONE_A': ['ZONE_B', 'ZONE_C'],
            'ZONE_B': ['ZONE_A', 'ZONE_D'],
            'ZONE_C': ['ZONE_A', 'ZONE_D'],
            'ZONE_D': ['ZONE_B', 'ZONE_C', 'CHARGING_AREA'],
            'CHARGING_AREA': ['ZONE_D'],
        }
    
    def request_reservation(self, agv_id: str, path: List[str]) -> bool:
        """ขอจอง Zones ตาม Path (ป้องกัน Deadlock)"""
        with self.lock:
            # ตรวจสอบว่า Path ว่างพอไหม
            for zone in path:
                current_count = sum(
                    1 for a, pos in self.agv_positions.items()
                    if pos == zone and a != agv_id
                )
                # รวม Reservations
                reserved_count = sum(
                    1 for a, res in self.agv_reservations.items()
                    if zone in res and a != agv_id
                )
                
                if current_count + reserved_count >= self.zone_capacity.get(zone, 1):
                    return False  # ไม่สามารถจองได้
            
            # จอง Zones ทั้งหมด
            self.agv_reservations[agv_id] = path
            return True
    
    def release_zone(self, agv_id: str, zone: str):
        """ปล่อย Zone เมื่อ AGV ผ่านแล้ว"""
        with self.lock:
            if agv_id in self.agv_reservations:
                if zone in self.agv_reservations[agv_id]:
                    self.agv_reservations[agv_id].remove(zone)
            
            if agv_id in self.agv_positions:
                old_zone = self.agv_positions[agv_id]
                if old_zone in self.zones:
                    self.zones[old_zone].discard(agv_id)
    
    def enter_zone(self, agv_id: str, zone: str) -> bool:
        """ขอเข้า Zone"""
        with self.lock:
            if zone not in self.zones:
                self.zones[zone] = set()
            
            current_count = len(self.zones[zone])
            capacity = self.zone_capacity.get(zone, 1)
            
            if current_count < capacity:
                self.zones[zone].add(agv_id)
                self.agv_positions[agv_id] = zone
                return True
            else:
                return False  # Zone เต็ม
    
    def find_path(self, start: str, end: str) -> Optional[List[str]]:
        """หา Path ที่สั้นที่สุดด้วย A* Algorithm"""
        from heapq import heappush, heappop
        
        # Simple BFS for now
        queue = [(start, [start])]
        visited = {start}
        
        while queue:
            (zone, path) = queue.pop(0)
            
            if zone == end:
                return path
            
            for neighbor in self.zone_graph.get(zone, []):
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append((neighbor, path + [neighbor]))
        
        return None  # ไม่พบ Path
    
    def get_status(self) -> dict:
        """แสดงสถานะ Fleet"""
        return {
            'total_agvs': len(self.agv_positions),
            'zone_occupancy': {
                zone: f"{len(occupants)}/{self.zone_capacity.get(zone, 1)}"
                for zone, occupants in self.zones.items()
            },
            'agv_positions': self.agv_positions.copy()
        }
```

---

## Step 1687: Charging Strategy

### การจัดการการชาร์จ AGV

```python
# Python - Smart Charging Scheduler
import numpy as np
from datetime import datetime, timedelta

class AGVChargingScheduler:
    """จัดการการชาร์จ AGV อย่างชาญฉลาด"""
    
    def __init__(self):
        # ข้อมูลสถานีชาร์จ
        self.charging_stations = {
            'CS1': {'location': 'ZONE_D', 'occupied_by': None, 'max_current': 40},
            'CS2': {'location': 'ZONE_D', 'occupied_by': None, 'max_current': 40},
            'CS3': {'location': 'ZONE_D', 'occupied_by': None, 'max_current': 20},  # Slow
            'CS4': {'location': 'CHARGING_AREA', 'occupied_by': None, 'max_current': 80},  # Fast
        }
        
        # Battery Specifications
        self.battery_capacity = 36.0    # kWh
        self.low_battery_threshold = 20.0   # % - ต้องชาร์จ
        self.critical_battery = 10.0        # % - หยุดทุกอย่าง
        self.full_charge_threshold = 95.0   # % - ออกจากชาร์จได้
        
        # Production Schedule
        self.busy_hours = list(range(6, 22))  # 06:00-22:00
        self.lean_hours = list(range(22, 6))  # 22:00-06:00
    
    def evaluate_agv_for_charging(self, agv_status: dict) -> dict:
        """ประเมินว่า AGV ควรไปชาร์จหรือไม่"""
        soc = agv_status['battery_soc']  # State of Charge (%)
        has_task = agv_status['has_task']
        current_hour = datetime.now().hour
        
        decision = {
            'should_charge': False,
            'priority': 0,
            'reason': ''
        }
        
        if soc <= self.critical_battery:
            decision.update({
                'should_charge': True,
                'priority': 10,  # สูงสุด
                'reason': 'CRITICAL - Battery ต่ำมาก หยุดงานชาร์จทันที'
            })
        
        elif soc <= self.low_battery_threshold and not has_task:
            decision.update({
                'should_charge': True,
                'priority': 8,
                'reason': f'LOW BATTERY ({soc:.0f}%) - ส่งชาร์จ'
            })
        
        elif soc <= 40 and current_hour not in self.busy_hours:
            decision.update({
                'should_charge': True,
                'priority': 5,
                'reason': 'OPPORTUNITY CHARGE - นอกเวลา Peak'
            })
        
        elif soc <= 60 and not has_task and len(self.get_available_stations()) > 0:
            decision.update({
                'should_charge': True,
                'priority': 3,
                'reason': 'PREVENTIVE CHARGE - ชาร์จเพื่อพร้อมใช้งาน'
            })
        
        return decision
    
    def get_available_stations(self) -> list:
        """หาสถานีชาร์จที่ว่าง"""
        return [
            sid for sid, station in self.charging_stations.items()
            if station['occupied_by'] is None
        ]
    
    def assign_charging_station(self, agv_id: str, agv_location: str) -> str:
        """กำหนด Charging Station ที่ใกล้ที่สุด"""
        available = self.get_available_stations()
        
        if not available:
            return None
        
        # เลือกสถานีที่ใกล้ที่สุดและมีกระแสสูง
        best_station = max(
            available,
            key=lambda sid: self.charging_stations[sid]['max_current']
        )
        
        # จอง Station
        self.charging_stations[best_station]['occupied_by'] = agv_id
        return best_station
    
    def release_station(self, agv_id: str):
        """ปล่อย Charging Station"""
        for sid, station in self.charging_stations.items():
            if station['occupied_by'] == agv_id:
                station['occupied_by'] = None
                print(f"AGV {agv_id} ออกจากสถานี {sid}")
                break
```

---

## Step 1688: Warehouse Integration

### การรวม AGV กับ WMS (Warehouse Management System)

```
WMS ← REST API → Fleet Manager ← MQTT → AGV Fleet

การทำงาน:
1. WMS สร้าง Pick Task
2. ส่งไป Fleet Manager ผ่าน REST API
3. Fleet Manager จัดสรร AGV
4. AGV รับ Order ผ่าน VDA 5050 (MQTT)
5. AGV Execute และส่งสถานะกลับ
6. Fleet Manager อัพเดท WMS
```

```python
# Python - WMS Integration Client
import requests
import json
from datetime import datetime

class WMSFleetIntegration:
    """เชื่อมต่อ WMS กับ Fleet Management System"""
    
    def __init__(self, fleet_mgr_url: str, wms_url: str):
        self.fleet_url = fleet_mgr_url
        self.wms_url = wms_url
        self.headers = {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer <token>'
        }
    
    def create_transport_order(self, pick_location: str,
                                drop_location: str, material_id: str) -> str:
        """สร้าง Transport Order จาก WMS"""
        order = {
            "orderType": "TRANSPORT",
            "priority": 5,
            "pickLocation": pick_location,
            "dropLocation": drop_location,
            "materialId": material_id,
            "quantity": 1,
            "loadType": "PALLET",
            "requestedBy": "WMS",
            "requestedAt": datetime.utcnow().isoformat()
        }
        
        response = requests.post(
            f"{self.fleet_url}/api/v1/orders",
            json=order,
            headers=self.headers,
            timeout=10
        )
        
        if response.status_code == 201:
            order_id = response.json()['orderId']
            print(f"Transport Order สร้างสำเร็จ: {order_id}")
            return order_id
        else:
            raise Exception(f"ไม่สามารถสร้าง Order: {response.text}")
    
    def get_order_status(self, order_id: str) -> dict:
        """ตรวจสอบสถานะ Order"""
        response = requests.get(
            f"{self.fleet_url}/api/v1/orders/{order_id}",
            headers=self.headers
        )
        return response.json()
    
    def update_wms_on_completion(self, order_id: str, order_status: dict):
        """อัพเดท WMS เมื่อ Transport เสร็จ"""
        if order_status['status'] == 'COMPLETED':
            wms_confirmation = {
                "orderId": order_id,
                "status": "DELIVERED",
                "completedAt": order_status['completedAt'],
                "agvId": order_status['assignedAgv'],
                "actualPickTime": order_status['actualPickTime'],
                "actualDropTime": order_status['actualDropTime']
            }
            
            requests.post(
                f"{self.wms_url}/api/v1/transport-confirmations",
                json=wms_confirmation,
                headers=self.headers
            )
            print(f"WMS อัพเดทแล้ว: Order {order_id} Delivered")
```

---

## Step 1689: AMR Programming (MiR)

### Mobile Industrial Robots (MiR100, MiR250)

```python
# Python - MiR AMR REST API Client
import requests
import json
import time

class MiRController:
    """ควบคุม MiR AMR ผ่าน REST API"""
    
    def __init__(self, robot_ip: str):
        self.base_url = f"http://{robot_ip}/api/v2.0.0"
        self.headers = {
            'Content-Type': 'application/json',
            'Authorization': 'Basic YWRtaW46OGM2OTc2ZTViNTQxMDQyNWQ4OWYxN2YxNmNkYmVhYWQ='  # admin:admin base64
        }
    
    def get_status(self) -> dict:
        """ดูสถานะ Robot"""
        response = requests.get(f"{self.base_url}/status", headers=self.headers)
        return response.json()
    
    def get_missions(self) -> list:
        """ดูรายการ Missions"""
        response = requests.get(f"{self.base_url}/missions", headers=self.headers)
        return response.json()
    
    def start_mission(self, mission_id: str) -> bool:
        """เริ่ม Mission"""
        payload = {
            "mission_id": mission_id,
            "priority": 1
        }
        response = requests.post(
            f"{self.base_url}/mission_queue",
            json=payload,
            headers=self.headers
        )
        return response.status_code == 201
    
    def move_to_position(self, x: float, y: float, orientation: float) -> bool:
        """เคลื่อนที่ไปยังตำแหน่งที่กำหนด"""
        # สร้าง Mission ชั่วคราว
        mission_data = {
            "name": f"MoveTo_{int(time.time())}",
            "map_id": self.get_current_map(),
            "actions": [
                {
                    "action_type": "relative_move",
                    "parameters": [
                        {"id": "x", "value": x},
                        {"id": "y", "value": y},
                        {"id": "orientation", "value": orientation},
                        {"id": "max_linear_speed", "value": 0.8},  # m/s
                        {"id": "max_angular_speed", "value": 0.5}  # rad/s
                    ]
                }
            ]
        }
        
        # สร้าง Mission
        response = requests.post(
            f"{self.base_url}/missions",
            json=mission_data,
            headers=self.headers
        )
        
        if response.status_code == 201:
            mission_id = response.json()['guid']
            return self.start_mission(mission_id)
        return False
    
    def get_current_map(self) -> str:
        """ดู Map ปัจจุบัน"""
        status = self.get_status()
        return status.get('map_id', '')
    
    def wait_for_completion(self, timeout: int = 120) -> bool:
        """รอให้ Mission สำเร็จ"""
        start = time.time()
        while time.time() - start < timeout:
            status = self.get_status()
            robot_state = status.get('state_id', 0)
            
            if robot_state == 3:  # 3 = Ready (ไม่มี Mission)
                return True
            elif robot_state == 12:  # 12 = Error
                return False
            
            time.sleep(0.5)
        
        return False  # Timeout
    
    def dock_to_charger(self) -> bool:
        """ไปชาร์จ"""
        missions = self.get_missions()
        charge_mission = next(
            (m for m in missions if 'charge' in m['name'].lower()),
            None
        )
        
        if charge_mission:
            return self.start_mission(charge_mission['guid'])
        return False
```

---

## Step 1690-1700: Lab Project

### โปรเจกต์: AGV System Integration สำหรับโรงงาน Electronics

**Layout**:
```
┌──────────────────────────────────────────────────────────┐
│  FACTORY FLOOR LAYOUT                                    │
│                                                          │
│  [Receiving]──→[Warehouse]──→[Line 1 Feeder]            │
│       ↑              │                                   │
│       │              └──→[Line 2 Feeder]                 │
│  [Shipping]←──[Finished Goods]←──[Test Station]         │
│                                                          │
│  AGV Routes (Laser-Guided):                             │
│  Route 1: Receiving → Warehouse (Material Intake)       │
│  Route 2: Warehouse → Line Feeders (Replenishment)      │
│  Route 3: Test Station → Finished Goods → Shipping     │
│                                                          │
│  Fleet Size: 4 AGVs (2x Forklift, 2x Tugger)           │
└──────────────────────────────────────────────────────────┘
```

**System Integration Points**:
1. WMS → FMS: Transport Orders
2. PLC Line 1, 2 → FMS: Material Requests
3. FMS → AGV Fleet: VDA 5050 Orders
4. AGV → PLC: Arrival/Departure Signals
5. FMS → Dashboard: Real-time Status

**Lab Tasks**:
1. ออกแบบ Map และ Zones
2. ตั้งค่า Traffic Management Rules
3. เขียน PLC Program สำหรับ Call Stations
4. สร้าง REST API Integration ระหว่าง PLC และ FMS
5. ทดสอบ Emergency Scenarios

---

## สรุป Part 85

ใน Part นี้เราได้เรียนรู้:
- ✅ AGV vs AMR ความแตกต่างและการเลือกใช้
- ✅ Fleet Management System Architecture
- ✅ VDA 5050 Standard
- ✅ PLC Call Station Control
- ✅ Traffic Management และ Deadlock Prevention
- ✅ Smart Charging Strategy
- ✅ WMS Integration
- ✅ MiR AMR API Control

---

## การนำทาง
- [← Part 84: Cobot Programming](part84_cobot_programming.md)
- [→ Part 86: Advanced Machine Vision](part86_industrial_vision_advanced.md)
- [📚 สารบัญหลัก](../README.md)
