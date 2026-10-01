# Part 89: PLC ถึง ERP Integration
## Steps 1761-1780 | ระดับ Capstone

---

## การนำทาง
- [← Part 88: MES Integration](part88_mes_integration.md)
- [→ Part 90: Advanced Industrial Networking](part90_advanced_networking.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1761: ERP Systems ในอุตสาหกรรม

### ERP คืออะไร?

**Enterprise Resource Planning (ERP)** คือซอฟต์แวร์ที่รวมระบบธุรกิจทั้งหมดไว้ในที่เดียว ตั้งแต่การเงิน, HR, Procurement, Manufacturing, Sales

```
Major ERP Systems:
┌──────────────────────────────────────────────────────────────┐
│ Vendor      │ Product           │ Market Share │ Strengths   │
├─────────────┼───────────────────┼──────────────┼─────────────┤
│ SAP         │ SAP S/4HANA       │ ~22%         │ Enterprise  │
│ Oracle      │ Oracle Cloud ERP  │ ~8%          │ Enterprise  │
│ Microsoft   │ Dynamics 365 F&O  │ ~7%          │ Mid-market  │
│ Infor       │ Infor CloudSuite  │ ~4%          │ Industry-sp │
│ IFS         │ IFS Applications  │ ~3%          │ Asset Intens│
│ Odoo        │ Odoo              │ SME          │ Open Source │
└─────────────┴───────────────────┴──────────────┴─────────────┘
```

---

## Step 1762: ISA-95 Levels 0-4

### การไหลของข้อมูลจาก Shop Floor ถึง ERP

```
ISA-95 Full Stack:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Level 4: ENTERPRISE (ERP, BI, CRM)                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  SAP S/4HANA: PP (Production Planning)                  │  │
│  │  - Planned Orders                                        │  │
│  │  - Production Orders                                     │  │
│  │  - Goods Issue / Receipt                                 │  │
│  │  - Quality Management                                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                         ▲ │                                    │
│              Confirmation │ │ Production Orders                │
│                           │ ▼                                  │
│  Level 3: MES (Manufacturing Execution)                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Work Orders, Recipe, Quality, OEE, Downtime            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                         ▲ │                                    │
│              Actuals      │ │ Setpoints & Schedules            │
│                           │ ▼                                  │
│  Level 2: SCADA (Supervisory)                                  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Real-time Monitoring, Alarming, Trending               │  │
│  └─────────────────────────────────────────────────────────┘  │
│                         ▲ │                                    │
│              Process Data │ │ Control Commands                 │
│                           │ ▼                                  │
│  Level 1: PLC/DCS (Control)                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Real-time Automation, Safety, Sequencing               │  │
│  └─────────────────────────────────────────────────────────┘  │
│                         ▲ │                                    │
│              Measurements │ │ Commands                         │
│                           │ ▼                                  │
│  Level 0: FIELD (Physical)                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Sensors, Actuators, Machines, Conveyors                │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1763: SAP PP Integration

### SAP PP (Production Planning) Module

```
SAP PP Key Transactions:
MD04 - Stock/Requirements List
MD06 - Collective Display MRP Results
CO01 - Create Production Order
CO11N - Order Confirmation (Backflush)
CO15 - Order Confirmation
MB31 - Goods Receipt
MB1A - Goods Issue
QM01 - Create Quality Notification
```

### SAP-PLC Integration Architecture

```
Method 1: SAP PI/PO (SAP Integration Platform)
  SAP S/4HANA → SAP PI/PO → MES → PLC
  ข้อดี: Standard SAP, Monitoring ดี
  ข้อเสีย: ค่าใช้จ่ายสูง, ซับซ้อน

Method 2: SAP MII (Manufacturing Integration & Intelligence)
  SAP S/4HANA → SAP MII → PLC/SCADA
  ข้อดี: Built for Manufacturing, OPC Connectivity
  ข้อเสีย: ต้องมี SAP Consultant

Method 3: Middleware (MuleSoft, BizTalk, Dell Boomi)
  SAP S/4HANA → Middleware → MES/PLC
  ข้อดี: ยืดหยุ่น, Multi-system
  ข้อเสีย: ค่า License Middleware

Method 4: Direct REST API
  PLC/SCADA → Edge PC → SAP REST API
  ข้อดี: ราคาถูก, ยืดหยุ่น
  ข้อเสีย: ต้องพัฒนาเอง, No Standard
```

---

## Step 1764: SAP REST API Integration

### Production Confirmation จาก PLC ไป SAP

```python
# Python - SAP S/4HANA Production Confirmation
# ส่งข้อมูลการผลิตจาก MES/SCADA ไปยัง SAP

import requests
import json
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class SAPProductionConnector:
    """เชื่อมต่อ MES/SCADA กับ SAP S/4HANA OData API"""
    
    def __init__(self, sap_host: str, client: str = '100', 
                 username: str = None, password: str = None):
        self.base_url = f"https://{sap_host}/sap/opu/odata/sap"
        self.auth = (username, password)
        self.client = client
        self.headers = {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            'sap-client': client
        }
        # CSRF Token (required for SAP write operations)
        self.csrf_token = None
        self.session = requests.Session()
        self.session.auth = self.auth
        self.session.headers.update(self.headers)
    
    def get_csrf_token(self) -> str:
        """ดึง CSRF Token จาก SAP"""
        response = self.session.get(
            f"{self.base_url}/API_PRODUCTION_ORDER_2_SRV/$metadata",
            headers={'x-csrf-token': 'fetch'}
        )
        token = response.headers.get('x-csrf-token')
        self.csrf_token = token
        return token
    
    def get_production_orders(self, plant: str, status: str = 'REL') -> list:
        """ดึง Production Orders ที่ Released จาก SAP"""
        filter_query = f"Plant eq '{plant}' and SystemStatus eq '{status}'"
        
        response = self.session.get(
            f"{self.base_url}/API_PRODUCTION_ORDER_2_SRV/A_ProductionOrder",
            params={
                '$filter': filter_query,
                '$select': 'ProductionOrder,Material,Plant,MRPPlant,TotalQuantity,ProductionUnit',
                '$format': 'json',
                '$top': 100
            }
        )
        response.raise_for_status()
        
        data = response.json()
        orders = data.get('d', {}).get('results', [])
        
        logger.info(f"ดึง {len(orders)} Production Orders จาก SAP")
        return orders
    
    def confirm_production(self, order_number: str, 
                           confirmed_qty: float, unit: str,
                           activity_time: float, shift: str,
                           yield_qty: float = None, scrap_qty: float = None) -> dict:
        """ส่ง Production Confirmation ไป SAP (CO11N equivalent)"""
        
        # ดึง CSRF Token
        if not self.csrf_token:
            self.get_csrf_token()
        
        # สร้าง Confirmation Payload (OData)
        payload = {
            "OrderID": order_number,
            "ConfirmedYieldQuantity": str(confirmed_qty),
            "ConfirmationUnit": unit,
            "FinalConfirmationIndicator": False,
            "OpenReservationsIndicator": True,
            "WorkActivityQuantity": str(activity_time),
            "ShiftDefinition": shift,
            "PostingDate": datetime.now().strftime('%Y-%m-%dT00:00:00')
        }
        
        if yield_qty:
            payload["ConfirmedYieldQuantity"] = str(yield_qty)
        if scrap_qty:
            payload["ConfirmedScrapQuantity"] = str(scrap_qty)
        
        # ส่งไป SAP
        response = self.session.post(
            f"{self.base_url}/API_PROD_ORDER_CONFIRMATION_2_SRV/A_ProdOrderConfirmation",
            json=payload,
            headers={'x-csrf-token': self.csrf_token}
        )
        
        if response.status_code == 201:
            result = response.json()
            conf_number = result.get('d', {}).get('ConfirmationCount', 'N/A')
            logger.info(f"Production Confirmation สำเร็จ: Order {order_number}, Conf# {conf_number}")
            return {'success': True, 'confirmation_number': conf_number}
        else:
            error_msg = response.json().get('error', {}).get('message', {}).get('value', 'Unknown error')
            logger.error(f"SAP Confirmation Error: {error_msg}")
            return {'success': False, 'error': error_msg}
    
    def post_goods_issue(self, order_number: str, 
                          material: str, qty: float, unit: str,
                          storage_location: str, plant: str) -> dict:
        """ส่ง Goods Issue (MB1A equivalent) - ตัดสต็อกวัตถุดิบ"""
        
        if not self.csrf_token:
            self.get_csrf_token()
        
        payload = {
            "PostingDate": datetime.now().strftime('%Y-%m-%dT00:00:00'),
            "DocumentDate": datetime.now().strftime('%Y-%m-%dT00:00:00'),
            "MovementType": "261",  # GI for Order
            "to_MaterialDocumentItem": {
                "results": [{
                    "Material": material,
                    "Plant": plant,
                    "StorageLocation": storage_location,
                    "Quantity": str(qty),
                    "BaseUnit": unit,
                    "ManufacturingOrder": order_number,
                    "MovementType": "261"
                }]
            }
        }
        
        response = self.session.post(
            f"{self.base_url}/API_MATERIAL_DOCUMENT_SRV/A_MaterialDocumentHeader",
            json=payload,
            headers={'x-csrf-token': self.csrf_token}
        )
        
        if response.status_code == 201:
            doc_number = response.json().get('d', {}).get('MaterialDocument', 'N/A')
            logger.info(f"Goods Issue สำเร็จ: Document {doc_number}")
            return {'success': True, 'document_number': doc_number}
        else:
            return {'success': False, 'error': response.text}
    
    def post_goods_receipt(self, order_number: str,
                            material: str, qty: float, unit: str,
                            storage_location: str, plant: str,
                            batch: str = None) -> dict:
        """ส่ง Goods Receipt (MB31 equivalent) - รับสินค้าสำเร็จรูป"""
        
        if not self.csrf_token:
            self.get_csrf_token()
        
        payload = {
            "PostingDate": datetime.now().strftime('%Y-%m-%dT00:00:00'),
            "DocumentDate": datetime.now().strftime('%Y-%m-%dT00:00:00'),
            "MovementType": "101",  # GR for Order
            "to_MaterialDocumentItem": {
                "results": [{
                    "Material": material,
                    "Plant": plant,
                    "StorageLocation": storage_location,
                    "Quantity": str(qty),
                    "BaseUnit": unit,
                    "ManufacturingOrder": order_number,
                    "MovementType": "101",
                    "Batch": batch or ""
                }]
            }
        }
        
        response = self.session.post(
            f"{self.base_url}/API_MATERIAL_DOCUMENT_SRV/A_MaterialDocumentHeader",
            json=payload,
            headers={'x-csrf-token': self.csrf_token}
        )
        
        if response.status_code == 201:
            doc_number = response.json().get('d', {}).get('MaterialDocument', 'N/A')
            logger.info(f"Goods Receipt สำเร็จ: {qty} {unit} ของ {material}, Document {doc_number}")
            return {'success': True, 'document_number': doc_number}
        else:
            return {'success': False, 'error': response.text}
    
    def create_quality_notification(self, order_number: str, 
                                      material: str, defect_code: str,
                                      defect_qty: float, description: str) -> dict:
        """สร้าง Quality Notification ใน SAP QM"""
        
        if not self.csrf_token:
            self.get_csrf_token()
        
        payload = {
            "NotificationType": "Q2",  # Q2 = Quality Notification
            "ShortText": f"Quality Issue: {material} Order {order_number}",
            "LongText": description,
            "ManufacturingOrder": order_number,
            "Material": material,
            "DefectQuantity": str(defect_qty),
            "DefectCode": defect_code
        }
        
        response = self.session.post(
            f"{self.base_url}/API_QUALITY_NOTIFICATION_SRV/A_QualityNotification",
            json=payload,
            headers={'x-csrf-token': self.csrf_token}
        )
        
        if response.status_code == 201:
            notif_number = response.json().get('d', {}).get('MaintenanceNotification', 'N/A')
            return {'success': True, 'notification_number': notif_number}
        else:
            return {'success': False, 'error': response.text}
```

---

## Step 1765: Middleware Integration

### MuleSoft สำหรับ SAP-PLC Integration

```
MuleSoft Integration Flow:
PLC → MQTT → Node-RED → MuleSoft API → SAP S/4HANA

MuleSoft Anypoint Studio:
1. HTTP Listener (รับข้อมูลจาก Node-RED)
2. Transform (แปลงข้อมูล)
3. SAP Connector (ส่งไป SAP)
4. Error Handling
5. Response
```

```xml
<!-- MuleSoft Flow Configuration (XML) -->
<mule xmlns:sap="http://www.mulesoft.org/schema/mule/sap"
      xmlns:http="http://www.mulesoft.org/schema/mule/http">

  <!-- HTTP Listener -->
  <flow name="production-confirmation-flow">
    <http:listener config-ref="HTTP_Listener_Config" path="/api/v1/production/confirm"/>
    
    <!-- Transform to SAP Format -->
    <ee:transform>
      <ee:message>
        <ee:set-payload><![CDATA[%dw 2.0
output application/xml
ns ns0 http://sap.com/xi/SAPGlobal20/Global
---
{
  ns0#ProductionOrderConfirmationRequest: {
    MessageHeader: {
      ID: uuid(),
      CreationDateTime: now() as String,
      SenderBusinessSystemID: "SCADA_SYSTEM"
    },
    ProductionOrderConfirmation: {
      ProductionOrderID: payload.order_number,
      ConfirmedYield: {
        Quantity: payload.good_qty as Number,
        UnitCode: payload.unit
      }
    }
  }
}]]>
        </ee:set-payload>
      </ee:message>
    </ee:transform>
    
    <!-- Send to SAP -->
    <sap:outbound-endpoint functionName="BAPI_PRODORD_CONFIRM_TIME_TICKET"
                           connector-ref="SAP_Connector"/>
    
    <!-- Transform SAP Response -->
    <ee:transform>
      <ee:message>
        <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
  success: payload.RETURN[0].TYPE != "E",
  message: payload.RETURN[0].MESSAGE,
  confirmation_number: payload.ORDERID
}]]>
        </ee:set-payload>
      </ee:message>
    </ee:transform>
  </flow>

</mule>
```

---

## Step 1766: Complete SAP-PLC Architecture

### สถาปัตยกรรมสมบูรณ์ SAP → PLC → SAP

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMPLETE SAP ↔ SHOP FLOOR INTEGRATION                     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  SAP S/4HANA (ERP)                                          │  │
│  │  ├── PP Module (Production Planning)                        │  │
│  │  ├── MM Module (Materials Management)                       │  │
│  │  ├── QM Module (Quality Management)                         │  │
│  │  └── PM Module (Plant Maintenance)                          │  │
│  └────────────────────────────────────────────────────────────  │  │
│                    │ REST API / IDoc / RFC                         │  │
│  ┌─────────────────▼────────────────────────────────────────────┐  │
│  │  MES (Manufacturing Execution System)                        │  │
│  │  ├── Work Order Management (from SAP PP)                    │  │
│  │  ├── Recipe/BOM Management (from SAP MM)                    │  │
│  │  ├── Quality Recording (to SAP QM)                          │  │
│  │  ├── Maintenance Requests (to SAP PM)                       │  │
│  │  └── Production Confirmation (to SAP PP)                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                    │ OPC UA / REST API                             │  │
│  ┌─────────────────▼────────────────────────────────────────────┐  │
│  │  SCADA + Edge (Real-time Layer)                              │  │
│  │  ├── PLC Data Collection                                     │  │
│  │  ├── Historian                                               │  │
│  │  └── AI Analytics                                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                    │ PROFINET / EtherNet/IP                        │  │
│  ┌─────────────────▼────────────────────────────────────────────┐  │
│  │  PLC + Field Devices                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

DATA FLOWS:
↓ SAP → Shop: Production Orders, BOM, Routing, Schedules
↑ Shop → SAP: 
  - Production Confirmations (CO11N)
  - Goods Issues (MB1A)  
  - Goods Receipts (MB31)
  - Quality Notifications (QM01)
  - Maintenance Notifications (IW21)
  - Time Confirmations (CO11N)
```

---

## Step 1767: Goods Issue Automation

### ตัดสต็อกอัตโนมัติเมื่อ PLC เริ่มผลิต

```python
# Python - Automated Goods Issue (GI) Manager
# ตัดสต็อกวัตถุดิบอัตโนมัติเมื่อสายการผลิตเริ่ม

import asyncio
from datetime import datetime

class AutoGoodsIssueManager:
    """จัดการ Goods Issue อัตโนมัติจาก PLC Signal"""
    
    def __init__(self, sap_connector: SAPProductionConnector,
                 mes_client, plc_client):
        self.sap = sap_connector
        self.mes = mes_client
        self.plc = plc_client
        self.pending_gi = {}
    
    async def on_production_start(self, work_order: str, line_id: str):
        """ถูกเรียกเมื่อสายการผลิตเริ่ม"""
        
        # ดึง BOM จาก MES/SAP
        bom = await self.mes.get_bom(work_order)
        
        # คำนวณปริมาณวัตถุดิบที่ต้องใช้
        planned_qty = bom['planned_quantity']
        
        goods_to_issue = []
        for component in bom['components']:
            required_qty = component['qty_per_unit'] * planned_qty
            goods_to_issue.append({
                'material': component['material_code'],
                'qty': required_qty,
                'unit': component['unit'],
                'storage_location': component['storage_location'],
                'plant': bom['plant']
            })
        
        # ส่ง Goods Issue ไป SAP
        gi_results = []
        for item in goods_to_issue:
            result = self.sap.post_goods_issue(
                order_number=work_order,
                material=item['material'],
                qty=item['qty'],
                unit=item['unit'],
                storage_location=item['storage_location'],
                plant=item['plant']
            )
            gi_results.append(result)
            
            if not result['success']:
                # GI ล้มเหลว: แจ้ง Operator
                await self.plc.write_tag(
                    f'DB_WO.GI_Error_{item["material"]}', True
                )
        
        # บันทึกสถานะ
        self.pending_gi[work_order] = {
            'status': 'ISSUED',
            'issued_at': datetime.utcnow().isoformat(),
            'results': gi_results
        }
        
        success_count = sum(1 for r in gi_results if r['success'])
        total_count = len(gi_results)
        
        print(f"Goods Issue สำหรับ WO {work_order}: {success_count}/{total_count} สำเร็จ")
    
    async def on_production_complete(self, work_order: str, 
                                      good_qty: float, scrap_qty: float,
                                      material: str, unit: str, 
                                      storage_location: str, plant: str,
                                      batch: str = None):
        """ถูกเรียกเมื่อการผลิตเสร็จสิ้น"""
        
        # 1. Goods Receipt (รับสินค้าสำเร็จรูป)
        gr_result = self.sap.post_goods_receipt(
            order_number=work_order,
            material=material,
            qty=good_qty,
            unit=unit,
            storage_location=storage_location,
            plant=plant,
            batch=batch
        )
        
        # 2. Production Confirmation
        conf_result = self.sap.confirm_production(
            order_number=work_order,
            confirmed_qty=good_qty,
            unit=unit,
            activity_time=self.calculate_activity_time(work_order),
            shift=self.get_current_shift(),
            yield_qty=good_qty,
            scrap_qty=scrap_qty
        )
        
        # 3. Quality Notification ถ้ามี Scrap
        if scrap_qty > 0:
            qn_result = self.sap.create_quality_notification(
                order_number=work_order,
                material=material,
                defect_code='D001',  # General Defect
                defect_qty=scrap_qty,
                description=f'Scrap from Production: {scrap_qty} {unit}'
            )
        
        print(f"Production Complete: WO {work_order}")
        print(f"  GR: {gr_result['success']}, Conf: {conf_result['success']}")
```

---

## Step 1768: Plant Maintenance Integration

### SAP PM กับ Predictive Maintenance

```python
# Python - SAP PM (Plant Maintenance) Notification Creator
# สร้าง PM Notification จาก AI Predictive Maintenance

class SAPMaintenanceConnector:
    """เชื่อมต่อ Predictive Maintenance กับ SAP PM"""
    
    def __init__(self, sap_connector: SAPProductionConnector):
        self.sap = sap_connector
    
    def create_maintenance_notification(self, equipment_id: str,
                                         failure_prediction: dict) -> dict:
        """สร้าง SAP PM Notification (IW21 equivalent)"""
        
        urgency = failure_prediction.get('urgency', 'LOW')
        rul_hours = failure_prediction.get('rul_hours', 999)
        
        # Map AI prediction to SAP Priority
        priority_map = {
            'CRITICAL': '1',   # Very High
            'HIGH': '2',       # High
            'MEDIUM': '3',     # Medium
            'LOW': '4'         # Low
        }
        
        payload = {
            "MaintenanceNotification": "",  # SAP จะ Generate
            "NotificationType": "M2",       # M2 = Maintenance Request
            "ShortText": f"[AI] {failure_prediction.get('failure_type', 'Unknown')} - {equipment_id}",
            "TechnicalObjectType": "EQUI",
            "Equipment": equipment_id,
            "Priority": priority_map.get(urgency, '4'),
            "LongTextNotification": f"""
Predictive Maintenance Alert (AI System)

Equipment: {equipment_id}
Predicted Failure: {failure_prediction.get('failure_type', 'Unknown')}
Estimated RUL: {rul_hours:.1f} hours
Health Score: {failure_prediction.get('health_score', 'N/A')}%
Urgency: {urgency}

Recommended Actions:
{chr(10).join('- ' + r for r in failure_prediction.get('recommendations', []))}

Generated by: AI Predictive Maintenance System
Timestamp: {datetime.utcnow().isoformat()}
            """.strip()
        }
        
        # ส่งไป SAP
        if not self.sap.csrf_token:
            self.sap.get_csrf_token()
        
        response = self.sap.session.post(
            f"{self.sap.base_url}/API_MAINTENANCENOTIFICATION/MaintenanceNotification",
            json=payload,
            headers={'x-csrf-token': self.sap.csrf_token}
        )
        
        if response.status_code == 201:
            notif_number = response.json().get('d', {}).get('MaintenanceNotification')
            print(f"SAP PM Notification สร้างสำเร็จ: {notif_number}")
            return {'success': True, 'notification': notif_number}
        else:
            return {'success': False, 'error': response.text}
```

---

## Step 1769-1780: Lab Project

### โปรเจกต์: SAP-PLC Integration สำหรับโรงงานยา

**Requirements**:
1. Production Order จาก SAP PP → MES Work Order
2. BOM Download ไป PLC
3. Goods Issue อัตโนมัติเมื่อผลิต
4. Electronic Batch Record → SAP QM
5. Goods Receipt เมื่อผลิตเสร็จ
6. Maintenance Notification เมื่อ Equipment ต้องการซ่อม

**Integration Architecture**:
```
SAP S/4HANA HANA
    ↓ REST OData API
MES (Python/FastAPI)
    ↓ OPC UA
Edge PC
    ↓ PROFINET
PLC S7-1500
```

**Testing Scenarios**:
1. สร้าง Production Order ใน SAP → ตรวจสอบว่า Work Order ปรากฎใน MES
2. เริ่มผลิต → ตรวจสอบ Goods Issue ใน SAP
3. ผลิตเสร็จ → ตรวจสอบ GR และ Confirmation ใน SAP
4. จงใจทำให้เกิด Defect → ตรวจสอบ Quality Notification ใน SAP

---

## สรุป Part 89

ใน Part นี้เราได้เรียนรู้:
- ✅ ERP Systems และ ISA-95 Hierarchy
- ✅ SAP PP/MM/QM Integration
- ✅ SAP OData REST API
- ✅ Production Confirmation อัตโนมัติ
- ✅ Goods Issue/Receipt อัตโนมัติ
- ✅ Quality Notification จาก PLC
- ✅ Plant Maintenance Integration กับ AI

---

## การนำทาง
- [← Part 88: MES Integration](part88_mes_integration.md)
- [→ Part 90: Advanced Industrial Networking](part90_advanced_networking.md)
- [📚 สารบัญหลัก](../README.md)
