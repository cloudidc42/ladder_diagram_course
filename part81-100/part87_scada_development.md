# Part 87: SCADA System Development
## Steps 1721-1740 | ระดับ Capstone

---

## การนำทาง
- [← Part 86: Advanced Machine Vision](part86_industrial_vision_advanced.md)
- [→ Part 88: MES Integration](part88_mes_integration.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1721: Ignition SCADA Platform

### ทำไมต้อง Ignition?

```
SCADA Platform Comparison:
┌──────────────────────────────────────────────────────────────────┐
│ Feature          │ Ignition     │ WinCC OA    │ AVEVA System Plat│
├──────────────────┼──────────────┼─────────────┼──────────────────┤
│ Licensing        │ Server-based │ Tag-based   │ Tag-based        │
│ (ไม่มี Tag Limit)│ ✓ Unlimited  │ จำกัด       │ จำกัด            │
│ Web/Mobile       │ ✓ Built-in   │ Add-on      │ Add-on           │
│ Database         │ ✓ Any SQL    │ Limited     │ SQL Server       │
│ Python Scripting │ ✓ Full       │ Partial     │ Partial          │
│ OPC UA           │ ✓ Built-in   │ ✓           │ ✓                │
│ Cloud Deploy     │ ✓ Edge+Cloud │ Limited     │ ✓                │
│ Price (Entry)    │ ~$3,500      │ ~$5,000     │ ~$15,000         │
│ Modules          │ 30+          │ Limited     │ 20+              │
└──────────────────┴──────────────┴─────────────┴──────────────────┘

เหตุผลที่เลือก Ignition:
1. Unlimited Tags - ไม่มีค่าใช้จ่ายเพิ่มเมื่อเพิ่ม Tag
2. HTML5 Perspective - เข้าถึงได้จาก Browser ทุกที่
3. Python Scripting - ยืดหยุ่นสูง
4. Community ใหญ่, Documentation ดี
5. Ignition Exchange - แชร์ Template ฟรี
```

---

## Step 1722: Ignition Architecture

### Ignition SCADA Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      IGNITION ARCHITECTURE                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  IGNITION GATEWAY (Server)                                   │  │
│  │  ├── Tag Provider (OPC UA, Internal, Remote)                 │  │
│  │  ├── Historian (SQL Database)                                │  │
│  │  ├── Alarm System                                            │  │
│  │  ├── Report Engine                                           │  │
│  │  ├── Security Manager                                        │  │
│  │  └── Module Engine                                           │  │
│  └────────────────────────────────────────────────────────────  │  │
│                    │ HTTP/HTTPS WebSocket                          │  │
│         ┌──────────┼───────────┬────────────┐                    │  │
│         ▼          ▼           ▼            ▼                    │  │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
│   │Perspective│ │  Vision  │ │ Designer │ │  Mobile  │          │  │
│   │(Browser) │ │(Client)  │ │(Engineer)│ │  App     │          │  │
│   └──────────┘ └──────────┘ └──────────┘ └──────────┘          │  │
│                                                                     │
│  PLC Connections:                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │ Siemens  │ │Allen-B.  │ │Mitsubishi│ │ OPC UA  │            │
│  │ S7 Driver│ │ Logix    │ │Q Series  │ │ Server  │            │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 1723: Tag Configuration

### การสร้าง Tag Structure ใน Ignition

```python
# Python (Ignition Scripting) - สร้าง Tag โดยอัตโนมัติจาก JSON Config
# รันใน Ignition Gateway Script Console

import json

def create_tags_from_config(provider_name, config_file_path):
    """สร้าง Tags จาก Configuration File"""
    
    # อ่าน Config
    with open(config_file_path, 'r', encoding='utf-8') as f:
        config = json.load(f)
    
    tags_created = 0
    
    for line_config in config['production_lines']:
        line_path = f"[{provider_name}]Production/{line_config['name']}"
        
        for equipment in line_config['equipment']:
            equip_path = f"{line_path}/{equipment['id']}"
            
            # สร้าง Tags สำหรับแต่ละ Equipment
            tags = []
            
            # Status Tags
            tags.append({
                'name': 'Running',
                'tagType': 'OPC',
                'dataType': 'Boolean',
                'opcItemPath': f"ns=3;s={equipment['id']}.Running",
                'tooltip': 'เครื่องทำงานอยู่'
            })
            
            tags.append({
                'name': 'Fault',
                'tagType': 'OPC',
                'dataType': 'Boolean',
                'opcItemPath': f"ns=3;s={equipment['id']}.Fault",
                'tooltip': 'มีข้อผิดพลาด'
            })
            
            # Process Values
            for pv in equipment.get('process_values', []):
                tags.append({
                    'name': pv['name'],
                    'tagType': 'OPC',
                    'dataType': pv['data_type'],
                    'opcItemPath': f"ns=3;s={equipment['id']}.{pv['name']}",
                    'engUnit': pv.get('unit', ''),
                    'tooltip': pv.get('description', ''),
                    'historicalDeadband': pv.get('deadband', 0.5)
                })
                
                # สร้าง Derived Tag สำหรับ Status
                if pv.get('has_limits'):
                    tags.append({
                        'name': f"{pv['name']}_Status",
                        'tagType': 'Expression',
                        'dataType': 'String',
                        'expression': f'''if({{@equip_path}/{pv['name']}} > {pv['high_high']}, "CRITICAL_HIGH",
                            if({{@equip_path}/{pv['name']}} > {pv['high']}, "HIGH",
                            if({{@equip_path}/{pv['name']}} < {pv['low_low']}, "CRITICAL_LOW",
                            if({{@equip_path}/{pv['name']}} < {pv['low']}, "LOW",
                            "NORMAL"))))'''
                    })
            
            # สร้าง Tags ผ่าน Ignition API
            system.tag.addTags(equip_path, tags)
            tags_created += len(tags)
    
    print(f"สร้าง Tags สำเร็จ: {tags_created} tags")
    return tags_created
```

---

## Step 1724: Perspective Screen Design

### การออกแบบ HMI ด้วย Ignition Perspective

```json
// Ignition Perspective - Production Overview Screen
// JSON Template สำหรับ Screen Configuration

{
  "type": "ia.container.view",
  "meta": {
    "name": "Production_Overview",
    "width": 1920,
    "height": 1080
  },
  "children": [
    {
      "type": "ia.display.label",
      "meta": {"name": "Title"},
      "position": {"x": 0, "y": 0, "width": 1920, "height": 80},
      "props": {
        "text": "Production Line Overview - {now.time}",
        "style": {
          "fontSize": 28,
          "fontWeight": "bold",
          "color": "#FFFFFF",
          "backgroundColor": "#1a237e"
        }
      }
    },
    {
      "type": "ia.container.flex",
      "meta": {"name": "KPI_Row"},
      "position": {"x": 20, "y": 90, "width": 1880, "height": 200},
      "props": {
        "direction": "row",
        "gap": 20
      },
      "children": [
        {
          "type": "ia.display.stat-tile",
          "props": {
            "title": "OEE Today",
            "value": "{[default]Production/Line01/OEE}",
            "unit": "%",
            "target": 85,
            "thresholds": [
              {"value": 85, "color": "#4caf50"},
              {"value": 70, "color": "#ff9800"},
              {"value": 0, "color": "#f44336"}
            ]
          }
        },
        {
          "type": "ia.display.stat-tile",
          "props": {
            "title": "Production Today",
            "value": "{[default]Production/Line01/GoodCount}",
            "unit": "units",
            "target": 2000
          }
        },
        {
          "type": "ia.display.stat-tile",
          "props": {
            "title": "Downtime Today",
            "value": "{[default]Production/Line01/DowntimeMinutes}",
            "unit": "min",
            "invertThreshold": true,
            "target": 30
          }
        },
        {
          "type": "ia.display.stat-tile",
          "props": {
            "title": "Quality Rate",
            "value": "{[default]Production/Line01/QualityRate}",
            "unit": "%",
            "target": 99.5
          }
        }
      ]
    }
  ]
}
```

---

## Step 1725: Tag Historian

### การตั้งค่า Tag Historian

```python
# Python (Ignition Gateway Script) - Tag Historian Query
# ดึงข้อมูลประวัติสำหรับ Reporting

import system

def query_production_history(line_id: str, start_time, end_time) -> list:
    """ดึงข้อมูลการผลิตจาก Historian"""
    
    tag_paths = [
        f"[default]Production/{line_id}/GoodCount",
        f"[default]Production/{line_id}/TotalCount",
        f"[default]Production/{line_id}/OEE",
        f"[default]Production/{line_id}/Availability",
        f"[default]Production/{line_id}/Performance",
        f"[default]Production/{line_id}/Quality"
    ]
    
    # QueryTagHistory - ดึงค่าเฉลี่ยทุก 5 นาที
    results = system.tag.queryTagHistory(
        paths=tag_paths,
        startDate=start_time,
        endDate=end_time,
        returnSize=-1,            # จำนวน Rows (-1 = Auto)
        aggregationMode="Average", # Average, Max, Min, LastValue
        returnFormat="Wide",       # Wide หรือ Tall
        intervalHours=0,
        intervalMinutes=5,         # ทุก 5 นาที
        intervalSeconds=0
    )
    
    # แปลงเป็น List ของ Dict
    data = []
    for row in range(results.getRowCount()):
        entry = {
            'timestamp': results.getValueAt(row, 0),
        }
        for col_idx, tag_path in enumerate(tag_paths):
            tag_name = tag_path.split('/')[-1]
            entry[tag_name] = results.getValueAt(row, col_idx + 1)
        data.append(entry)
    
    return data

def calculate_shift_oee(line_id: str, shift_start, shift_end) -> dict:
    """คำนวณ OEE สำหรับ Shift"""
    history = query_production_history(line_id, shift_start, shift_end)
    
    if not history:
        return {'error': 'ไม่มีข้อมูล'}
    
    # ดึงค่าล่าสุด (สะสม)
    last_row = history[-1]
    first_row = history[0]
    
    # คำนวณ
    total_count = last_row.get('TotalCount', 0) - first_row.get('TotalCount', 0)
    good_count = last_row.get('GoodCount', 0) - first_row.get('GoodCount', 0)
    
    # Average OEE
    avg_oee = sum(r.get('OEE', 0) for r in history) / len(history)
    
    return {
        'line': line_id,
        'shift_start': str(shift_start),
        'shift_end': str(shift_end),
        'total_count': total_count,
        'good_count': good_count,
        'quality_rate': good_count / total_count * 100 if total_count > 0 else 0,
        'avg_oee': avg_oee,
        'data_points': len(history)
    }
```

---

## Step 1726: Alarm System

### การออกแบบ Alarm Management (ANSI/ISA-18.2)

```python
# Python (Ignition Gateway Script) - Alarm Configuration
# สร้าง Alarm Notifications ตาม ISA-18.2

def configure_alarm_notification():
    """ตั้งค่า Alarm Notification Pipeline"""
    
    # Alarm Priorities (ANSI/ISA-18.2)
    # Level 1: Critical (Alarm Sound ดัง, ต้องตอบสนองทันที)
    # Level 2: High (Alarm Sound ปกติ, ตอบสนองใน 10 นาที)
    # Level 3: Medium (Visual Only, ตอบสนองใน 1 ชั่วโมง)
    # Level 4: Low (Log เท่านั้น)
    
    alarm_config = {
        'pipelines': [
            {
                'name': 'Critical_Alarm_Pipeline',
                'trigger': {'priority': ['Critical']},
                'actions': [
                    {
                        'type': 'send_email',
                        'recipients': ['maintenance@factory.com', 'manager@factory.com'],
                        'subject': 'CRITICAL ALARM: {alarm.label}',
                        'body': '''
                            Time: {alarm.activeTime}
                            Location: {alarm.source.path}
                            Message: {alarm.label}
                            Value: {alarm.value} {alarm.engUnit}
                            Priority: CRITICAL
                            
                            กรุณาดำเนินการทันที!
                        '''
                    },
                    {
                        'type': 'send_sms',
                        'recipients': ['+66812345678', '+66823456789'],
                        'message': 'CRITICAL: {alarm.label} at {alarm.source.path}'
                    },
                    {
                        'type': 'write_tag',
                        'tagPath': '[default]Alarms/Critical_Horn',
                        'value': True
                    }
                ]
            },
            {
                'name': 'High_Alarm_Pipeline',
                'trigger': {'priority': ['High']},
                'actions': [
                    {
                        'type': 'send_email',
                        'recipients': ['maintenance@factory.com'],
                        'subject': 'HIGH ALARM: {alarm.label}'
                    }
                ]
            }
        ]
    }
    
    return alarm_config

def shelve_nuisance_alarms(tag_path: str, alarm_name: str, minutes: int, reason: str):
    """Shelve Alarm ชั่วคราว (ISA-18.2 Alarm Shelving)"""
    
    # บันทึกการ Shelve
    system.db.runUpdate(
        "INSERT INTO alarm_shelve_log (tag_path, alarm_name, shelved_by, shelved_at, shelve_minutes, reason) VALUES (?, ?, ?, NOW(), ?, ?)",
        [tag_path, alarm_name, system.security.getUsername(), minutes, reason]
    )
    
    # Shelve ใน Ignition
    system.alarm.shelve(
        alarmPaths=[f"alarm:/{tag_path}/{alarm_name}"],
        minutes=minutes
    )
    
    print(f"Alarm '{alarm_name}' ถูก Shelve เป็นเวลา {minutes} นาที โดย {system.security.getUsername()}")
```

---

## Step 1727: Report Generation

### Auto Report Generation ด้วย Ignition Reporting

```python
# Python (Ignition Gateway Script) - Daily Report Generation
import system
from datetime import datetime, timedelta

def generate_daily_production_report(line_id: str, report_date=None):
    """สร้างรายงานการผลิตประจำวัน"""
    
    if report_date is None:
        report_date = datetime.now().date() - timedelta(days=1)  # เมื่อวาน
    
    # กำหนดช่วงเวลา Shifts
    shifts = [
        {'name': 'Shift A', 'start': '06:00', 'end': '14:00'},
        {'name': 'Shift B', 'start': '14:00', 'end': '22:00'},
        {'name': 'Shift C', 'start': '22:00', 'end': '06:00'},
    ]
    
    # ดึงข้อมูลแต่ละ Shift
    shift_data = []
    daily_totals = {'good_count': 0, 'total_count': 0, 'downtime_min': 0}
    
    for shift in shifts:
        # คำนวณเวลา
        shift_start = datetime.strptime(f"{report_date} {shift['start']}", "%Y-%m-%d %H:%M")
        
        if shift['name'] == 'Shift C':
            # Shift C ข้ามคืน
            shift_end = datetime.strptime(f"{report_date + timedelta(days=1)} {shift['end']}", "%Y-%m-%d %H:%M")
        else:
            shift_end = datetime.strptime(f"{report_date} {shift['end']}", "%Y-%m-%d %H:%M")
        
        # Query Historian
        history = query_production_history(line_id, shift_start, shift_end)
        
        if history:
            shift_oee = calculate_shift_oee(line_id, shift_start, shift_end)
            shift_data.append({
                'shift': shift['name'],
                **shift_oee
            })
            
            # รวม Daily Totals
            daily_totals['good_count'] += shift_oee.get('good_count', 0)
            daily_totals['total_count'] += shift_oee.get('total_count', 0)
    
    # Query Downtime Events
    downtime_events = system.db.runQuery(
        """SELECT event_start, event_end, duration_min, reason_code, reason_desc
           FROM production_downtime 
           WHERE line_id = ? AND DATE(event_start) = ?
           ORDER BY event_start""",
        [line_id, str(report_date)]
    )
    
    # สร้าง Report ผ่าน Ignition Reporting Module
    report_params = {
        'line_id': line_id,
        'report_date': str(report_date),
        'shift_data': shift_data,
        'daily_totals': daily_totals,
        'downtime_events': downtime_events,
        'oee_target': 85.0,
        'production_target': 6000  # units/day
    }
    
    # Generate PDF
    report_bytes = system.report.executeReport(
        path='/Production/Daily_Production_Report',
        project='SCADA_Factory_A',
        parameters=report_params,
        fileType='PDF'
    )
    
    # บันทึก PDF
    report_filename = f"/reports/production/{line_id}_{report_date}.pdf"
    system.file.writeFile(report_filename, report_bytes)
    
    # Email Report อัตโนมัติ
    system.net.sendEmail(
        smtpProfile='Factory_SMTP',
        fromAddress='scada@factory.com',
        subject=f"Daily Production Report - {line_id} - {report_date}",
        toAddresses=['manager@factory.com', 'production@factory.com'],
        html=f"<p>กรุณาดูรายงานการผลิตในไฟล์แนบ</p>",
        attachments=[(report_filename, report_bytes, 'application/pdf')]
    )
    
    print(f"สร้างรายงาน {report_filename} สำเร็จ")
    return report_filename
```

---

## Step 1728: User Management

### Role-Based Access Control (RBAC)

```python
# Python (Ignition Gateway Script) - User Management
# ระบบ Role-Based Access Control สำหรับ SCADA

def setup_user_roles():
    """กำหนด User Roles และ Permissions"""
    
    roles = {
        'Operator': {
            'description': 'ผู้ปฏิบัติงาน',
            'permissions': [
                'view_screens',
                'acknowledge_alarms',
                'start_stop_production',
                'view_reports'
            ],
            'screen_access': ['Production', 'Alarms', 'Status'],
            'write_tags': [
                '[default]Production/*/StartCmd',
                '[default]Production/*/StopCmd',
            ],
            'session_timeout_min': 30
        },
        'Technician': {
            'description': 'ช่างเทคนิค',
            'permissions': [
                'view_screens',
                'acknowledge_alarms',
                'start_stop_production',
                'modify_setpoints',
                'view_reports',
                'maintenance_functions'
            ],
            'screen_access': ['Production', 'Alarms', 'Status', 'Maintenance', 'Diagnostics'],
            'write_tags': [
                '[default]Production/*/**',   # ทุก Production Tags
                '[default]Maintenance/**'
            ],
            'session_timeout_min': 120
        },
        'Engineer': {
            'description': 'วิศวกร',
            'permissions': ['all_permissions'],
            'screen_access': ['all_screens'],
            'write_tags': ['[default]**'],  # ทุก Tags
            'session_timeout_min': 480
        },
        'Viewer': {
            'description': 'ผู้ชม (Read-Only)',
            'permissions': ['view_screens', 'view_reports'],
            'screen_access': ['Production', 'Reports'],
            'write_tags': [],  # ไม่สามารถ Write ได้
            'session_timeout_min': 60
        }
    }
    
    return roles

def check_tag_write_permission(username: str, tag_path: str) -> bool:
    """ตรวจสอบสิทธิ์การ Write Tag"""
    user_roles = system.security.getRoles()
    role_config = setup_user_roles()
    
    for role in user_roles:
        if role in role_config:
            allowed_patterns = role_config[role].get('write_tags', [])
            for pattern in allowed_patterns:
                # แปลง Wildcard Pattern เป็น Regex
                import re
                regex_pattern = pattern.replace('**', '.*').replace('*', '[^/]*')
                if re.match(regex_pattern, tag_path):
                    return True
    
    return False
```

---

## Step 1729: Mobile Access

### Ignition Perspective Mobile

```python
# Python (Ignition Gateway Script) - Mobile Push Notifications
# ส่ง Push Notification ไป Mobile App เมื่อมี Critical Alarm

import system

def send_mobile_push_notification(title: str, message: str, 
                                   priority: str, recipients: list):
    """ส่ง Push Notification ผ่าน Ignition Perspective"""
    
    # Ignition Perspective ใช้ WebSocket สำหรับ Real-time Updates
    # สำหรับ Push Notification ต้องใช้ Firebase (FCM) หรือ APNs
    
    notification_payload = {
        'title': title,
        'body': message,
        'priority': priority,
        'timestamp': str(system.date.now()),
        'data': {
            'type': 'alarm',
            'priority': priority
        }
    }
    
    for recipient in recipients:
        # ดึง Device Token จาก Database
        token_result = system.db.runScalarQuery(
            "SELECT device_token FROM user_mobile_tokens WHERE username = ? AND active = 1",
            [recipient]
        )
        
        if token_result:
            # ส่งผ่าน Firebase FCM API
            system.net.httpPost(
                url='https://fcm.googleapis.com/fcm/send',
                contentType='application/json',
                postData=system.util.jsonEncode({
                    'to': token_result,
                    'notification': notification_payload,
                    'priority': 'high'
                }),
                headerValues={
                    'Authorization': f"key={system.tag.readBlocking('[default]System/FCM_Server_Key')[0].value}"
                }
            )
            
            print(f"ส่ง Push Notification ไป {recipient} สำเร็จ")
```

---

## Step 1730: Complete SCADA สำหรับ Food Plant

### Complete Production Monitoring SCADA

```
Food Processing Plant SCADA - Screen Structure:

1. Main Overview
   ├── Plant Schematic (P&ID Style)
   ├── Production KPIs (OEE, Output, Quality)
   ├── Active Alarms Summary
   └── Navigation Bar

2. Line Detail Screens
   ├── Line 01 - Cookie Production
   │   ├── Mixing Station
   │   ├── Baking Oven
   │   ├── Cooling Conveyor
   │   └── Packing Station
   ├── Line 02 - Bread Production
   └── Line 03 - Cake Production

3. Utilities
   ├── Steam System
   ├── Compressed Air
   ├── Chiller System
   └── Electrical Distribution

4. Quality
   ├── In-Process Checks
   ├── SPC Charts
   └── Defect Tracking

5. Alarms
   ├── Active Alarms
   ├── Alarm History
   └── Alarm Statistics

6. Reports
   ├── Production Reports
   ├── Quality Reports
   ├── Maintenance Reports
   └── Energy Reports

7. Administration
   ├── User Management
   ├── System Diagnostics
   └── PLC Connection Status
```

### Ignition Python Script สำหรับ Food Plant

```python
# Python (Ignition Gateway Script) - CIP Control Integration
# เชื่อมต่อ SCADA กับ CIP (Clean-In-Place) System

import system
from datetime import datetime

def start_cip_sequence(line_id: str, cip_program: str, operator: str) -> dict:
    """เริ่ม CIP Sequence จาก SCADA"""
    
    # ตรวจสอบว่า Line ไม่ได้ผลิตอยู่
    line_running = system.tag.readBlocking(
        [f"[default]Production/{line_id}/Running"]
    )[0].value
    
    if line_running:
        return {'success': False, 'message': 'ไม่สามารถเริ่ม CIP ขณะ Line กำลังผลิต'}
    
    # ดึงรายละเอียด CIP Program จาก Database
    cip_recipe = system.db.runQuery(
        "SELECT * FROM cip_recipes WHERE program_name = ? AND active = 1",
        [cip_program]
    )
    
    if not cip_recipe or cip_recipe.getRowCount() == 0:
        return {'success': False, 'message': f"ไม่พบ CIP Program: {cip_program}"}
    
    # บันทึก CIP Session
    session_id = system.db.runUpdateQuery(
        """INSERT INTO cip_sessions 
           (line_id, program, started_by, start_time, status)
           VALUES (?, ?, ?, NOW(), 'RUNNING')
           RETURNING id""",
        [line_id, cip_program, operator]
    )
    
    # ส่งคำสั่งไป PLC
    cip_tags = [
        f"[default]CIP/{line_id}/ProgramNumber",
        f"[default]CIP/{line_id}/StartCmd",
        f"[default]CIP/{line_id}/SessionID"
    ]
    
    cip_values = [
        cip_recipe.getValueAt(0, 'program_number'),
        True,
        session_id
    ]
    
    system.tag.writeBlocking(cip_tags, cip_values)
    
    # Log Event
    system.util.sendMessage(
        project='SCADA_Factory_A',
        messageHandler='CIP_Event_Logger',
        payload={
            'event': 'CIP_STARTED',
            'line': line_id,
            'program': cip_program,
            'operator': operator,
            'session_id': session_id
        }
    )
    
    return {
        'success': True,
        'session_id': session_id,
        'message': f"CIP {cip_program} เริ่มต้นสำเร็จ"
    }
```

---

## Step 1731-1740: Lab Project

### โปรเจกต์: สร้าง SCADA สำหรับโรงงานน้ำดื่ม

**Requirements**:
1. Main Overview Screen พร้อม P&ID
2. 3 Production Lines Monitoring
3. Water Treatment Process
4. Quality Lab Integration
5. Energy Dashboard
6. Alarm Management (ISA-18.2)
7. Daily/Weekly Reports
8. Mobile Access

**Technical Stack**:
- Ignition 8.1+
- PostgreSQL Database
- OPC UA → Siemens S7-1500
- Perspective (HTML5)
- Push Notifications (FCM)

**Lab Tasks**:
1. ออกแบบ Tag Structure (200 Tags)
2. สร้าง Main Overview Screen
3. ตั้งค่า Alarm Notifications
4. สร้าง Auto Daily Report
5. ทดสอบ Mobile Access

---

## สรุป Part 87

ใน Part นี้เราได้เรียนรู้:
- ✅ Ignition SCADA Architecture
- ✅ Tag Structure Design
- ✅ Perspective Screen Development
- ✅ Tag Historian และ Query
- ✅ Alarm Management (ISA-18.2)
- ✅ Auto Report Generation
- ✅ Role-Based Access Control
- ✅ Mobile Push Notifications

---

## การนำทาง
- [← Part 86: Advanced Machine Vision](part86_industrial_vision_advanced.md)
- [→ Part 88: MES Integration](part88_mes_integration.md)
- [📚 สารบัญหลัก](../README.md)
