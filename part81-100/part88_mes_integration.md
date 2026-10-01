# Part 88: MES Integration
## Steps 1741-1760 | ระดับ Capstone

---

## การนำทาง
- [← Part 87: SCADA System Development](part87_scada_development.md)
- [→ Part 89: PLC ถึง ERP Integration](part89_plc_to_erp.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1741: MES คืออะไร?

### Manufacturing Execution System (ISA-95 Level 3)

```
ISA-95 Hierarchy (Purdue Model):
┌──────────────────────────────────────────────────────────────────┐
│  Level 4: Business (ERP, SCM, CRM)    - Planning & Logistics     │
├──────────────────────────────────────────────────────────────────┤
│  Level 3: Manufacturing Operations (MES) - Production Control    │
├──────────────────────────────────────────────────────────────────┤
│  Level 2: Supervisory (SCADA/HMI)     - Process Monitoring       │
├──────────────────────────────────────────────────────────────────┤
│  Level 1: Control (PLC/DCS)           - Real-time Control        │
├──────────────────────────────────────────────────────────────────┤
│  Level 0: Field (Sensors/Actuators)   - Physical Process         │
└──────────────────────────────────────────────────────────────────┘

MES อยู่ระหว่าง ERP (Business Planning) และ PLC (Real-time Control)
หน้าที่: แปลงแผนการผลิตจาก ERP เป็นคำสั่งให้ PLC ปฏิบัติ
```

### MES Modules (ISA-95 Part 3)

```
11 Core MES Functions:
1.  Production Scheduling         - กำหนดตารางการผลิต
2.  Resource Allocation           - จัดสรรทรัพยากร
3.  Dispatching Production Units  - ส่งงานลงสายผลิต
4.  Document Control              - จัดการเอกสาร/Recipe
5.  Data Collection/Acquisition   - เก็บข้อมูลกระบวนการ
6.  Labor Management              - จัดการแรงงาน
7.  Quality Management            - ควบคุมคุณภาพ
8.  Process Management            - ควบคุมกระบวนการ
9.  Maintenance Management        - จัดการบำรุงรักษา
10. Product Tracking & Genealogy  - ติดตามสินค้า
11. Performance Analysis (OEE)    - วิเคราะห์ประสิทธิภาพ
```

---

## Step 1742: MES Products

### MES Software ที่นิยมในตลาด

```
Enterprise MES:
┌──────────────────────────────────────────────────────────────────┐
│ Vendor          │ Product          │ Strengths                   │
├─────────────────┼──────────────────┼─────────────────────────────┤
│ Siemens         │ Opcenter MES     │ PLM Integration, Automotive │
│ Rockwell        │ FactoryTalk PF   │ AB PLC Integration         │
│ SAP             │ SAP ME/MII       │ SAP ERP Integration        │
│ Oracle          │ Oracle MES       │ Oracle ERP Integration     │
│ Dassault        │ DELMIA Apriso    │ Aerospace, Pharma          │
│ Infor           │ Infor MES        │ SME Market                 │
│ Plex            │ Plex MES (SaaS)  │ Cloud-native               │
└─────────────────┴──────────────────┴─────────────────────────────┘

Mid-Market MES:
┌──────────────────────────────────────────────────────────────────┐
│ Vendor          │ Product          │ Strengths                   │
├─────────────────┼──────────────────┼─────────────────────────────┤
│ Critical Mfg    │ Critical MES     │ Semiconductor, Electronics  │
│ Tulip           │ Tulip Platform   │ No-code, Frontline Ops     │
│ Worximity       │ Tile             │ OEE Focus                  │
│ Fishbowl        │ Fishbowl Mfg     │ QuickBooks Integration     │
└─────────────────┴──────────────────┴─────────────────────────────┘
```

---

## Step 1743: MES Data Flow

### การไหลของข้อมูลใน MES

```
┌─────────────────────────────────────────────────────────────────┐
│               MES DATA FLOW ARCHITECTURE                        │
│                                                                 │
│  ERP (SAP)                                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Production Order → MES Work Order                        │  │
│  │ Bill of Material → MES Recipe                            │  │
│  │ Routing → MES Process Order                              │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                          │ Order Download                      │
│  MES (Level 3)          ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 1. Work Order Management                                 │  │
│  │ 2. Recipe/BOM Management                                 │  │
│  │ 3. Production Scheduling                                 │  │
│  │ 4. Quality Recording                                     │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                          │ Dispatch & Commands                 │
│  PLC/SCADA (Level 1-2)  ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Execute Production                                       │  │
│  │ Collect Process Data                                     │  │
│  │ Report Actuals (Count, Time, Quality)                    │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                          │ Production Confirmation             │
│                          ▼ (back to ERP)                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1744: REST API Integration

### PLC → MES ผ่าน REST API

```python
# Python - MES API Client สำหรับ PLC Data
# Middleware ที่รับข้อมูลจาก PLC และส่งไป MES

import asyncio
import aiohttp
import json
from datetime import datetime
from dataclasses import dataclass

@dataclass
class ProductionReport:
    """ข้อมูลการผลิตที่จะส่ง MES"""
    work_order: str
    line_id: str
    shift: str
    good_count: int
    scrap_count: int
    rework_count: int
    start_time: str
    end_time: str
    operator_id: str
    downtime_events: list

class MESAPIClient:
    """Client สำหรับเชื่อมต่อกับ MES REST API"""
    
    def __init__(self, mes_url: str, api_key: str):
        self.mes_url = mes_url.rstrip('/')
        self.api_key = api_key
        self.session = None
    
    async def __aenter__(self):
        self.session = aiohttp.ClientSession(headers={
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        })
        return self
    
    async def __aexit__(self, *args):
        if self.session:
            await self.session.close()
    
    async def get_work_orders(self, line_id: str, status: str = 'RELEASED') -> list:
        """ดึง Work Orders จาก MES"""
        async with self.session.get(
            f"{self.mes_url}/api/v1/work-orders",
            params={'line': line_id, 'status': status}
        ) as response:
            response.raise_for_status()
            data = await response.json()
            return data['work_orders']
    
    async def start_work_order(self, work_order_id: str, line_id: str, 
                                operator_id: str) -> dict:
        """เริ่ม Work Order"""
        payload = {
            'work_order_id': work_order_id,
            'line_id': line_id,
            'operator_id': operator_id,
            'actual_start': datetime.utcnow().isoformat() + 'Z'
        }
        
        async with self.session.post(
            f"{self.mes_url}/api/v1/work-orders/{work_order_id}/start",
            json=payload
        ) as response:
            response.raise_for_status()
            return await response.json()
    
    async def report_production(self, report: ProductionReport) -> dict:
        """รายงานผลการผลิตไป MES"""
        payload = {
            'work_order': report.work_order,
            'line_id': report.line_id,
            'shift': report.shift,
            'quantities': {
                'good': report.good_count,
                'scrap': report.scrap_count,
                'rework': report.rework_count
            },
            'actual_start': report.start_time,
            'actual_end': report.end_time,
            'operator_id': report.operator_id,
            'downtime_events': report.downtime_events
        }
        
        async with self.session.post(
            f"{self.mes_url}/api/v1/work-orders/{report.work_order}/production-report",
            json=payload
        ) as response:
            response.raise_for_status()
            result = await response.json()
            print(f"รายงานผลการผลิต: WO {report.work_order} - "
                  f"Good: {report.good_count}, Scrap: {report.scrap_count}")
            return result
    
    async def record_quality_result(self, work_order: str, serial_number: str,
                                     measurements: dict, pass_fail: str) -> dict:
        """บันทึกผลการตรวจสอบคุณภาพ"""
        payload = {
            'work_order': work_order,
            'serial_number': serial_number,
            'inspection_timestamp': datetime.utcnow().isoformat() + 'Z',
            'measurements': measurements,
            'overall_result': pass_fail,
            'inspector': 'VISION_SYSTEM',
            'inspection_type': 'AUTOMATED'
        }
        
        async with self.session.post(
            f"{self.mes_url}/api/v1/quality/inspections",
            json=payload
        ) as response:
            response.raise_for_status()
            return await response.json()
    
    async def record_downtime(self, work_order: str, line_id: str,
                               reason_code: str, start_time: str, 
                               end_time: str, comments: str = '') -> dict:
        """บันทึก Downtime Event"""
        payload = {
            'work_order': work_order,
            'line_id': line_id,
            'reason_code': reason_code,
            'start_time': start_time,
            'end_time': end_time,
            'duration_minutes': self.calculate_minutes(start_time, end_time),
            'comments': comments,
            'reported_by': 'PLC_SYSTEM'
        }
        
        async with self.session.post(
            f"{self.mes_url}/api/v1/downtime-events",
            json=payload
        ) as response:
            response.raise_for_status()
            return await response.json()
    
    async def get_recipe(self, part_number: str, revision: str = 'latest') -> dict:
        """ดึง Recipe จาก MES"""
        async with self.session.get(
            f"{self.mes_url}/api/v1/recipes/{part_number}",
            params={'revision': revision}
        ) as response:
            response.raise_for_status()
            return await response.json()
    
    def calculate_minutes(self, start: str, end: str) -> float:
        start_dt = datetime.fromisoformat(start.rstrip('Z'))
        end_dt = datetime.fromisoformat(end.rstrip('Z'))
        return (end_dt - start_dt).total_seconds() / 60
```

---

## Step 1745: Work Order Management

### การจัดการ Work Order ผ่าน MES

```python
# Python - Work Order Lifecycle Manager
# จัดการวงจรชีวิต Work Order จาก PLC

import asyncio
import logging
from enum import Enum
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)

class WorkOrderStatus(Enum):
    CREATED = "CREATED"
    RELEASED = "RELEASED"
    STARTED = "STARTED"
    PAUSED = "PAUSED"
    COMPLETED = "COMPLETED"
    CANCELLED = "CANCELLED"

class WorkOrderManager:
    """จัดการ Work Orders บน Production Line"""
    
    def __init__(self, mes_client: MESAPIClient, plc_client):
        self.mes = mes_client
        self.plc = plc_client
        self.active_work_orders = {}
        self.line_id = "LINE_01"
    
    async def poll_new_work_orders(self):
        """ตรวจสอบ Work Orders ใหม่จาก MES"""
        while True:
            try:
                work_orders = await self.mes.get_work_orders(
                    line_id=self.line_id,
                    status='RELEASED'
                )
                
                for wo in work_orders:
                    if wo['id'] not in self.active_work_orders:
                        logger.info(f"พบ Work Order ใหม่: {wo['id']} - {wo['part_number']}")
                        await self.prepare_work_order(wo)
                
            except Exception as e:
                logger.error(f"Error polling work orders: {e}")
            
            await asyncio.sleep(30)  # ตรวจทุก 30 วินาที
    
    async def prepare_work_order(self, work_order: dict):
        """เตรียมการสำหรับ Work Order"""
        wo_id = work_order['id']
        part_number = work_order['part_number']
        quantity = work_order['planned_quantity']
        
        # ดึง Recipe จาก MES
        recipe = await self.mes.get_recipe(part_number)
        
        # ดาวน์โหลด Recipe ไป PLC
        await self.download_recipe_to_plc(recipe)
        
        # เก็บใน Active Work Orders
        self.active_work_orders[wo_id] = {
            'work_order': work_order,
            'recipe': recipe,
            'status': WorkOrderStatus.RELEASED,
            'start_time': None,
            'good_count': 0,
            'scrap_count': 0,
            'downtime_events': []
        }
        
        # แจ้ง Operator ผ่าน HMI
        await self.plc.write_tag('DB_WO.WO_Number', wo_id)
        await self.plc.write_tag('DB_WO.Part_Number', part_number)
        await self.plc.write_tag('DB_WO.Planned_Qty', quantity)
        await self.plc.write_tag('DB_WO.Recipe_Loaded', True)
        
        logger.info(f"Work Order {wo_id} พร้อมสำหรับการผลิต")
    
    async def download_recipe_to_plc(self, recipe: dict):
        """ดาวน์โหลด Recipe Parameters ไปยัง PLC"""
        recipe_tags = []
        recipe_values = []
        
        for param in recipe.get('parameters', []):
            tag_path = f"DB_Recipe.{param['name']}"
            recipe_tags.append(tag_path)
            recipe_values.append(param['value'])
        
        await self.plc.write_tags_batch(recipe_tags, recipe_values)
        logger.info(f"ดาวน์โหลด Recipe {recipe['name']} ไปยัง PLC สำเร็จ ({len(recipe_tags)} parameters)")
    
    async def start_work_order(self, wo_id: str, operator_id: str):
        """เริ่ม Work Order"""
        if wo_id not in self.active_work_orders:
            raise ValueError(f"ไม่พบ Work Order: {wo_id}")
        
        wo_data = self.active_work_orders[wo_id]
        
        # อัพเดท MES
        await self.mes.start_work_order(wo_id, self.line_id, operator_id)
        
        # อัพเดทสถานะ
        wo_data['status'] = WorkOrderStatus.STARTED
        wo_data['start_time'] = datetime.utcnow().isoformat() + 'Z'
        
        logger.info(f"Work Order {wo_id} เริ่มต้นแล้ว โดย {operator_id}")
    
    async def complete_work_order(self, wo_id: str):
        """จบ Work Order และรายงาน MES"""
        if wo_id not in self.active_work_orders:
            return
        
        wo_data = self.active_work_orders[wo_id]
        
        # สร้าง Production Report
        report = ProductionReport(
            work_order=wo_id,
            line_id=self.line_id,
            shift=self.get_current_shift(),
            good_count=wo_data['good_count'],
            scrap_count=wo_data['scrap_count'],
            rework_count=0,
            start_time=wo_data['start_time'],
            end_time=datetime.utcnow().isoformat() + 'Z',
            operator_id=wo_data.get('operator_id', 'UNKNOWN'),
            downtime_events=wo_data['downtime_events']
        )
        
        # ส่งรายงานไป MES
        result = await self.mes.report_production(report)
        
        # ลบออกจาก Active
        del self.active_work_orders[wo_id]
        
        logger.info(f"Work Order {wo_id} เสร็จสิ้น: Good={report.good_count}, Scrap={report.scrap_count}")
    
    def get_current_shift(self) -> str:
        """หา Shift ปัจจุบัน"""
        hour = datetime.now().hour
        if 6 <= hour < 14:
            return 'A'
        elif 14 <= hour < 22:
            return 'B'
        else:
            return 'C'
```

---

## Step 1746: Electronic Batch Records (EBR)

### EBR สำหรับ Pharmaceutical / Food Industry

```python
# Python - Electronic Batch Record System
# สำคัญมากสำหรับ Pharma (FDA 21 CFR Part 11)

import hashlib
import json
from datetime import datetime
from typing import Optional

class ElectronicBatchRecord:
    """ระบบ Electronic Batch Record พร้อม Audit Trail"""
    
    def __init__(self, batch_id: str, product_code: str, recipe_id: str):
        self.batch_id = batch_id
        self.product_code = product_code
        self.recipe_id = recipe_id
        self.created_at = datetime.utcnow().isoformat()
        self.status = 'OPEN'
        
        # Batch Steps
        self.steps = []
        self.current_step = 0
        
        # Material Usage
        self.materials = []
        
        # Equipment Used
        self.equipment_log = []
        
        # Process Data (from PLC)
        self.process_data = []
        
        # Deviations
        self.deviations = []
        
        # Audit Trail (FDA 21 CFR Part 11)
        self.audit_trail = []
        
        # Digital Signatures
        self.signatures = []
        
        # บันทึก Creation
        self.add_audit_entry('BATCH_CREATED', 'SYSTEM', 
                             f"Batch {batch_id} created for {product_code}")
    
    def add_audit_entry(self, action: str, user: str, description: str,
                        data: dict = None):
        """เพิ่ม Audit Trail Entry (FDA 21 CFR Part 11)"""
        entry = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'action': action,
            'user': user,
            'description': description,
            'data': data or {},
            'hash': None  # จะคำนวณหลัง
        }
        
        # คำนวณ Hash สำหรับ Integrity Check
        prev_hash = self.audit_trail[-1]['hash'] if self.audit_trail else 'GENESIS'
        entry_str = json.dumps({**entry, 'prev_hash': prev_hash}, sort_keys=True)
        entry['hash'] = hashlib.sha256(entry_str.encode()).hexdigest()
        
        self.audit_trail.append(entry)
    
    def record_material_usage(self, material_code: str, lot_number: str,
                               planned_qty: float, actual_qty: float, 
                               unit: str, weighed_by: str, checked_by: str):
        """บันทึกการใช้วัตถุดิบ"""
        material_record = {
            'material_code': material_code,
            'lot_number': lot_number,
            'planned_qty': planned_qty,
            'actual_qty': actual_qty,
            'variance': actual_qty - planned_qty,
            'variance_pct': (actual_qty - planned_qty) / planned_qty * 100,
            'unit': unit,
            'weighed_by': weighed_by,
            'checked_by': checked_by,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        self.materials.append(material_record)
        
        # ตรวจสอบ Variance
        max_variance = 2.0  # ±2%
        if abs(material_record['variance_pct']) > max_variance:
            self.add_deviation(
                'MATERIAL_VARIANCE',
                f"{material_code}: Variance = {material_record['variance_pct']:.1f}%",
                weighed_by
            )
        
        self.add_audit_entry(
            'MATERIAL_WEIGHED',
            weighed_by,
            f"{material_code} weighed: {actual_qty} {unit}",
            material_record
        )
    
    def record_process_step(self, step_name: str, plc_data: dict, operator: str):
        """บันทึก Process Step พร้อมข้อมูล PLC"""
        step_record = {
            'step': step_name,
            'timestamp': datetime.utcnow().isoformat(),
            'plc_data': plc_data,
            'operator': operator,
            'result': 'PASS' if self.validate_step(step_name, plc_data) else 'FAIL'
        }
        
        self.process_data.append(step_record)
        
        if step_record['result'] == 'FAIL':
            self.add_deviation(
                'PROCESS_OUT_OF_SPEC',
                f"Step {step_name}: Process parameters out of specification",
                operator
            )
        
        self.add_audit_entry(
            'PROCESS_STEP_RECORDED',
            operator,
            f"Process step {step_name}: {step_record['result']}",
            {'plc_values_count': len(plc_data)}
        )
    
    def validate_step(self, step_name: str, plc_data: dict) -> bool:
        """ตรวจสอบว่า Process Parameters อยู่ใน Spec"""
        # โหลด Spec จาก Recipe
        specs = self.load_step_specs(step_name)
        
        for param, spec in specs.items():
            if param in plc_data:
                value = plc_data[param]
                if not (spec['low'] <= value <= spec['high']):
                    return False
        
        return True
    
    def add_deviation(self, deviation_type: str, description: str, 
                       reported_by: str, impact: str = 'MINOR'):
        """บันทึก Deviation"""
        deviation = {
            'id': f"DEV_{len(self.deviations) + 1:04d}",
            'type': deviation_type,
            'description': description,
            'impact': impact,
            'reported_by': reported_by,
            'reported_at': datetime.utcnow().isoformat(),
            'status': 'OPEN',
            'disposition': None
        }
        
        self.deviations.append(deviation)
        
        self.add_audit_entry(
            'DEVIATION_RECORDED',
            reported_by,
            f"Deviation: {description}",
            deviation
        )
    
    def add_electronic_signature(self, user: str, role: str, 
                                   action: str, password_hash: str):
        """เพิ่ม Electronic Signature (21 CFR Part 11)"""
        signature = {
            'user': user,
            'role': role,
            'action': action,
            'signed_at': datetime.utcnow().isoformat(),
            'password_verified': True,  # ตรวจสอบแล้ว
            'meaning': f"{role} {action} batch {self.batch_id}"
        }
        
        self.signatures.append(signature)
        self.add_audit_entry('E_SIGNATURE_ADDED', user, f"{role} signed: {action}")
    
    def close_batch(self, reviewer: str, disposition: str):
        """ปิด Batch Record"""
        if self.deviations:
            open_deviations = [d for d in self.deviations if d['status'] == 'OPEN']
            if open_deviations:
                raise Exception(f"ยังมี {len(open_deviations)} Deviations ที่ยังเปิดอยู่")
        
        self.status = 'CLOSED'
        self.add_audit_entry('BATCH_CLOSED', reviewer, 
                             f"Batch closed with disposition: {disposition}")
    
    def load_step_specs(self, step_name: str) -> dict:
        """โหลด Specification จาก Recipe (Placeholder)"""
        # ในระบบจริง จะโหลดจาก Recipe Database
        default_specs = {
            'temperature': {'low': 75.0, 'high': 85.0},
            'pressure': {'low': 2.0, 'high': 4.0},
            'mixing_time': {'low': 25.0, 'high': 35.0}
        }
        return default_specs
```

---

## Step 1747: OEE in MES

### การเก็บข้อมูล OEE ระดับ MES

```python
# Python - OEE Calculation Engine (MES Level)
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

class MES_OEE_Engine:
    """คำนวณ OEE ระดับ MES พร้อมการวิเคราะห์ Six Big Losses"""
    
    def __init__(self, db_connection):
        self.db = db_connection
    
    def calculate_oee_for_period(self, line_id: str, 
                                   start_time: datetime, end_time: datetime) -> dict:
        """คำนวณ OEE สำหรับช่วงเวลาที่กำหนด"""
        
        # ดึงข้อมูลจากฐานข้อมูล
        production_data = self.get_production_data(line_id, start_time, end_time)
        downtime_data = self.get_downtime_data(line_id, start_time, end_time)
        speed_loss_data = self.get_speed_loss_data(line_id, start_time, end_time)
        quality_data = self.get_quality_data(line_id, start_time, end_time)
        
        # 1. Planned Production Time
        total_minutes = (end_time - start_time).total_seconds() / 60
        planned_downtime = sum(d['duration'] for d in downtime_data 
                              if d['category'] == 'PLANNED')
        planned_production_time = total_minutes - planned_downtime
        
        # 2. Availability Losses
        unplanned_downtime = sum(d['duration'] for d in downtime_data 
                                if d['category'] == 'UNPLANNED')
        
        # Six Big Losses - Availability
        equipment_failure = sum(d['duration'] for d in downtime_data 
                               if d['reason_category'] == 'BREAKDOWN')
        setup_adjustment = sum(d['duration'] for d in downtime_data 
                              if d['reason_category'] == 'SETUP')
        
        availability = (planned_production_time - unplanned_downtime) / planned_production_time
        operating_time = planned_production_time - unplanned_downtime
        
        # 3. Performance Losses
        total_parts = production_data.get('total_count', 0)
        ideal_cycle_time = production_data.get('ideal_cycle_time', 1)
        
        # Six Big Losses - Performance
        minor_stoppages = speed_loss_data.get('minor_stoppage_count', 0) * ideal_cycle_time / 60
        reduced_speed = speed_loss_data.get('speed_reduction_minutes', 0)
        
        if operating_time > 0 and total_parts > 0:
            performance = (ideal_cycle_time * total_parts / 60) / operating_time
            performance = min(1.0, performance)
        else:
            performance = 0.0
        
        # 4. Quality Losses
        good_parts = quality_data.get('good_count', 0)
        scrap_parts = quality_data.get('scrap_count', 0)
        rework_parts = quality_data.get('rework_count', 0)
        
        # Six Big Losses - Quality
        startup_rejects = quality_data.get('startup_rejects', 0)
        production_rejects = scrap_parts - startup_rejects
        
        if total_parts > 0:
            quality = good_parts / total_parts
        else:
            quality = 0.0
        
        # 5. Overall OEE
        oee = availability * performance * quality
        
        return {
            'period': {
                'start': start_time.isoformat(),
                'end': end_time.isoformat(),
                'planned_time_min': planned_production_time
            },
            'oee': oee * 100,
            'availability': availability * 100,
            'performance': performance * 100,
            'quality': quality * 100,
            'six_big_losses': {
                'availability': {
                    'equipment_failure': equipment_failure,
                    'setup_adjustment': setup_adjustment
                },
                'performance': {
                    'minor_stoppages': minor_stoppages,
                    'reduced_speed': reduced_speed
                },
                'quality': {
                    'startup_rejects': startup_rejects,
                    'production_rejects': production_rejects
                }
            },
            'production_counts': {
                'total': total_parts,
                'good': good_parts,
                'scrap': scrap_parts,
                'rework': rework_parts
            },
            'world_class': oee >= 0.85
        }
    
    def get_production_data(self, line_id, start, end) -> dict:
        # Query Database
        return {}  # Placeholder
    
    def get_downtime_data(self, line_id, start, end) -> list:
        return []
    
    def get_speed_loss_data(self, line_id, start, end) -> dict:
        return {}
    
    def get_quality_data(self, line_id, start, end) -> dict:
        return {}
```

---

## Step 1748-1760: Lab Project

### โปรเจกต์: MES Integration สำหรับโรงงานยา

**Requirements (FDA Regulated)**:
1. Work Order Management (จาก SAP)
2. Electronic Batch Record (21 CFR Part 11)
3. Recipe Management (Validated)
4. Deviation Management
5. Quality Management (In-process + Final)
6. OEE Reporting
7. Audit Trail สมบูรณ์

**Data Flow**:
```
SAP (Work Order) → MES → PLC (Recipe Download)
                      ↑
PLC (Production Data) → MES (EBR) → SAP (Confirmation)
                             ↓
                         Quality System
```

**Lab Tasks**:
1. ออกแบบ EBR Template สำหรับ Tablet Manufacturing
2. สร้าง REST API Integration กับ SAP Mock
3. Implement PLC Recipe Download
4. ทดสอบ Audit Trail
5. สร้าง OEE Report

---

## สรุป Part 88

ใน Part นี้เราได้เรียนรู้:
- ✅ MES Architecture (ISA-95 Level 3)
- ✅ 11 Core MES Functions
- ✅ REST API Integration กับ PLC
- ✅ Work Order Lifecycle Management
- ✅ Electronic Batch Records (FDA 21 CFR Part 11)
- ✅ OEE Calculation ระดับ MES
- ✅ Six Big Losses Analysis

---

## การนำทาง
- [← Part 87: SCADA System Development](part87_scada_development.md)
- [→ Part 89: PLC ถึง ERP Integration](part89_plc_to_erp.md)
- [📚 สารบัญหลัก](../README.md)
