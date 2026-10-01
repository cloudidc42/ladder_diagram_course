# Part 51: Cybersecurity สำหรับระบบ Automation (Steps 1001-1020)

## บทนำ: ทำไม Cybersecurity ถึงสำคัญสำหรับ PLC

ในยุค Industry 4.0 ระบบ PLC และ Automation ถูกเชื่อมต่อกับเครือข่ายมากขึ้น ทำให้เกิดความเสี่ยงด้าน Cybersecurity ที่ไม่เคยมีมาก่อน เหตุการณ์สำคัญที่เปลี่ยนมุมมองโลก:

- **Stuxnet (2010)**: มัลแวร์ที่โจมตี Siemens SCADA ทำลายเครื่องหมุนเหวี่ยงนิวเคลียร์อิหร่าน
- **Ukraine Power Grid (2015/2016)**: โจมตีระบบไฟฟ้าทำให้ไฟดับ 225,000 ครัวเรือน
- **Colonial Pipeline (2021)**: Ransomware หยุดการส่งน้ำมัน 45% ของชายฝั่งตะวันออกสหรัฐ
- **Oldsmar Water Treatment (2021)**: แฮกเกอร์พยายามเพิ่ม NaOH ในน้ำประปา 111 เท่า

---

## Step 1001: ICS/OT Cybersecurity Threats

### ประเภทภัยคุกคามสำหรับระบบ ICS/OT

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREAT LANDSCAPE FOR ICS/OT                   │
├──────────────────┬──────────────────────────────────────────────┤
│ ประเภทภัยคุกคาม  │ ตัวอย่าง                                     │
├──────────────────┼──────────────────────────────────────────────┤
│ Malware          │ Stuxnet, TRITON, Industroyer, BlackEnergy     │
│ Ransomware       │ Colonial Pipeline, Norsk Hydro                │
│ Insider Threat   │ พนักงานไม่พอใจ, ความผิดพลาดโดยไม่ตั้งใจ      │
│ Supply Chain     │ SolarWinds, hardware backdoor                 │
│ Phishing         │ Spear phishing ผ่าน email                     │
│ Remote Access    │ VPN vulnerability, RDP bruteforce             │
│ Physical Access  │ USB drop, unauthorized laptop connection      │
│ DoS/DDoS         │ flooding network, overloading HMI             │
└──────────────────┴──────────────────────────────────────────────┘
```

### IT vs OT Security Differences

```
┌─────────────────────┬──────────────────────────┬──────────────────────────┐
│ ด้าน                │ IT Security              │ OT Security              │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ ลำดับความสำคัญ      │ CIA (Confidentiality,    │ AIC (Availability,       │
│                     │ Integrity, Availability) │ Integrity, Confidential) │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ Downtime            │ ยอมรับได้               │ ยอมรับไม่ได้ (อาจตาย)   │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ Patch cycle         │ Monthly                  │ ปีละครั้ง หรือไม่ทำเลย  │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ อายุอุปกรณ์         │ 3-5 ปี                  │ 15-25 ปี                 │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ Protocol            │ TCP/IP, TLS              │ Modbus, PROFINET, DNP3   │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ Real-time           │ ไม่จำเป็น               │ Critical (ms level)      │
└─────────────────────┴──────────────────────────┴──────────────────────────┘
```

---

## Step 1002: IEC 62443 Standard

### โครงสร้างมาตรฐาน IEC 62443

```
IEC 62443 - Industrial Automation and Control Systems Security
│
├── IEC 62443-1 (General)
│   ├── 62443-1-1: Terminology & Concepts
│   ├── 62443-1-2: Master Glossary
│   ├── 62443-1-3: System Security Compliance Metrics
│   └── 62443-1-4: IACS Security Lifecycle
│
├── IEC 62443-2 (Policies & Procedures - Asset Owner)
│   ├── 62443-2-1: Security Management System
│   ├── 62443-2-2: Operating IACS Security Program
│   ├── 62443-2-3: Patch Management
│   └── 62443-2-4: Security Program for IACS Service Providers
│
├── IEC 62443-3 (System)
│   ├── 62443-3-1: Security Technologies
│   ├── 62443-3-2: Security Risk Assessment
│   └── 62443-3-3: System Security Requirements & Levels
│
└── IEC 62443-4 (Component)
    ├── 62443-4-1: Secure Product Development
    └── 62443-4-2: Technical Security Requirements
```

### Security Level (SL) ตาม IEC 62443

```
┌──────────┬──────────────────────────────────────────────────────────┐
│ SL Level │ คำอธิบาย                                                  │
├──────────┼──────────────────────────────────────────────────────────┤
│ SL 0     │ ไม่มีการป้องกัน (ไม่แนะนำ)                              │
├──────────┼──────────────────────────────────────────────────────────┤
│ SL 1     │ ป้องกันการโจมตีโดยบังเอิญ (casual/coincidental attack)  │
│          │ - Basic authentication                                    │
│          │ - Physical security                                       │
├──────────┼──────────────────────────────────────────────────────────┤
│ SL 2     │ ป้องกัน intentional attack ด้วย low sophistication       │
│          │ - Role-based access control                              │
│          │ - Audit logging                                          │
│          │ - Network segmentation                                   │
├──────────┼──────────────────────────────────────────────────────────┤
│ SL 3     │ ป้องกัน sophisticated attack with IACS knowledge        │
│          │ - Encrypted communication                               │
│          │ - Multi-factor authentication                           │
│          │ - Intrusion detection                                   │
├──────────┼──────────────────────────────────────────────────────────┤
│ SL 4     │ ป้องกัน nation-state level attack                       │
│          │ - Air-gap systems                                        │
│          │ - Hardware security modules                             │
└──────────┴──────────────────────────────────────────────────────────┘
```

---

## Step 1003: Purdue Model และ Network Segmentation

### Purdue Enterprise Reference Architecture (PERA)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PURDUE MODEL (ISA-95/IEC 62264)                    │
│                                                                        │
│  Level 5: Enterprise Network (ERP, Email, Internet)                   │
│  ──────────────────────────────────────────────────                   │
│  ████████████████  ENTERPRISE FIREWALL  ████████████████              │
│                                                                        │
│  Level 4: Business Planning (MES, Historian server)                   │
│  ──────────────────────────────────────────────────                   │
│  ████████████████     OT/IT FIREWALL     ████████████████             │
│  ░░░░░░░░░░░░░░░░░░  DMZ Zone  ░░░░░░░░░░░░░░░░░░░░░░░               │
│  (Historian mirror, Remote access gateway, Data diode)                 │
│  ████████████████     OT/IT FIREWALL     ████████████████             │
│                                                                        │
│  Level 3: Site Manufacturing Operations (SCADA, DCS, HMI)            │
│  ──────────────────────────────────────────────────                   │
│  ████████████████   CELL/AREA FIREWALL   ████████████████             │
│                                                                        │
│  Level 2: Area Control (Supervisory HMI, Batch server)               │
│  ──────────────────────────────────────────────────                   │
│                                                                        │
│  Level 1: Basic Control (PLC, DCS Controller, RTU)                   │
│  ──────────────────────────────────────────────────                   │
│                                                                        │
│  Level 0: Process (Sensors, Actuators, Field devices)                 │
└──────────────────────────────────────────────────────────────────────┘
```

### การออกแบบ DMZ Zone

```
Internet
    │
    ▼
┌─────────────┐
│  Enterprise │  (Level 5)
│  Firewall   │
└──────┬──────┘
       │
┌──────▼──────────────────────────────────────────────┐
│                  Enterprise Network                   │
│  ERP Server, Email, Office workstations              │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────┐
│  OT/IT FW   │  (Next-Gen Firewall)
│  (外向き)   │  - Deep Packet Inspection
│             │  - Protocol validation
└──────┬──────┘
       │
┌──────▼──────────────────────────────────────────────┐
│                      DMZ ZONE                         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  Historian   │  │ Remote Access│  │  Data     │  │
│  │  Mirror      │  │  Gateway     │  │  Diode    │  │
│  │  (read-only) │  │  (Jump Host) │  │ (one-way) │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────┐
│  OT/IT FW   │  (Second Firewall)
└──────┬──────┘
       │
┌──────▼──────────────────────────────────────────────┐
│                  OT Network (Level 3)                 │
│  SCADA Server, Historian, Engineering Workstation    │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│               Control Network (Level 2)               │
│  PLC, DCS, RTU, HMI                                  │
└──────────────────────────────────────────────────────┘
```

---

## Step 1004: Firewall Rules สำหรับ OT

### Firewall Policy สำหรับ OT Network

```
# ตัวอย่าง Firewall Rules สำหรับ OT Network (Cisco ASA/Fortinet)
# เขียนเป็น pseudo-code เพื่อความเข้าใจ

# ======================================================
# ZONE DEFINITION
# ======================================================
# zone ENTERPRISE     (Level 4-5)
# zone DMZ            (Demilitarized Zone)
# zone OT_LEVEL3      (SCADA, Historian)
# zone OT_LEVEL2      (PLC, HMI)
# zone OT_LEVEL1      (Field Devices)

# ======================================================
# ENTERPRISE to DMZ Rules
# ======================================================
ALLOW  ENTERPRISE → DMZ : HTTPS(443)    # Web HMI read-only
ALLOW  ENTERPRISE → DMZ : HTTPS(443)    # Historian query
DENY   ENTERPRISE → DMZ : ALL           # Default deny

# ======================================================
# DMZ to OT_LEVEL3 Rules
# ======================================================
ALLOW  DMZ → OT_LEVEL3 : OPC-DA(135)    # Historian sync
ALLOW  DMZ → OT_LEVEL3 : OPC-UA(4840)   # OPC UA secure
DENY   DMZ → OT_LEVEL3 : ALL            # Default deny

# ======================================================
# OT_LEVEL3 to OT_LEVEL2 Rules
# ======================================================
ALLOW  OT_LEVEL3 → OT_LEVEL2 : Modbus-TCP(502)  # PLC read
ALLOW  OT_LEVEL3 → OT_LEVEL2 : S7(102)          # Siemens S7
ALLOW  OT_LEVEL3 → OT_LEVEL2 : ICMP             # Ping only
DENY   OT_LEVEL3 → OT_LEVEL2 : ALL              # Default deny

# ======================================================
# BLOCKED RULES (Explicit Deny with Logging)
# ======================================================
DENY+LOG  ANY → OT_LEVEL1 : RDP(3389)    # No remote desktop to field
DENY+LOG  ANY → OT_LEVEL1 : SSH(22)      # No SSH to field devices
DENY+LOG  INTERNET → ANY : ALL           # No direct internet to OT
```

### Industrial Protocol Port Reference

```
┌─────────────────────────┬──────────┬──────────────────────────┐
│ Protocol                │ Port     │ หมายเหตุ                 │
├─────────────────────────┼──────────┼──────────────────────────┤
│ Modbus TCP              │ 502/TCP  │ ไม่มี encryption         │
│ DNP3 (TCP)              │ 20000    │ ระบบไฟฟ้า/water          │
│ EtherNet/IP             │ 44818    │ Rockwell/Allen-Bradley   │
│ PROFINET                │ 34964    │ Siemens/PROFIBUS Int.    │
│ OPC UA                  │ 4840     │ มี encryption/auth       │
│ Siemens S7              │ 102      │ ISO-TSAP                 │
│ BACnet/IP               │ 47808    │ Building automation      │
│ IEC 60870-5-104         │ 2404     │ SCADA/grid               │
│ MQTT                    │ 1883/TLS │ IoT messaging            │
│ HTTPS                   │ 443      │ Web services             │
└─────────────────────────┴──────────┴──────────────────────────┘
```

---

## Step 1005: VPN สำหรับ Remote Access

### Remote Access Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   SECURE REMOTE ACCESS DESIGN                    │
│                                                                   │
│  Engineer/Vendor laptop                                           │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────┐    ┌──────────────────┐    ┌────────────────────┐ │
│  │ MFA Auth │───▶│  VPN Concentrator│───▶│   Jump Server      │ │
│  │ (Token + │    │  (DMZ Zone)      │    │   (Bastion Host)   │ │
│  │ Password)│    │                  │    │   - Session record │ │
│  └──────────┘    └──────────────────┘    │   - Command log    │ │
│                                          │   - Video record   │ │
│                                          └─────────┬──────────┘ │
│                                                     │            │
│                                          ┌──────────▼──────────┐│
│                                          │   OT Network        ││
│                                          │   (Limited access   ││
│                                          │    per zone/device) ││
│                                          └─────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Remote Access Products สำหรับ OT

```
┌──────────────────┬─────────────────────────────────────────────┐
│ Product          │ Features                                     │
├──────────────────┼─────────────────────────────────────────────┤
│ Tosibox          │ - Zero-config VPN                           │
│                  │ - Hardware lock device                       │
│                  │ - Cell network support                       │
├──────────────────┼─────────────────────────────────────────────┤
│ Ewon Cosy+       │ - Industrial VPN gateway                    │
│                  │ - Talk2M cloud platform                      │
│                  │ - Modbus/EtherNet/IP support                 │
├──────────────────┼─────────────────────────────────────────────┤
│ Secomea          │ - Zero-trust remote access                  │
│                  │ - Audit trail                                │
│                  │ - OT-specific features                       │
├──────────────────┼─────────────────────────────────────────────┤
│ Siemens SINEMA   │ - Integrated with TIA Portal                │
│ Remote Connect   │ - Certificate-based auth                    │
│                  │ - Role-based access                          │
├──────────────────┼─────────────────────────────────────────────┤
│ Cisco IE VPN     │ - Industrial switch + VPN                   │
│                  │ - Policy enforcement                         │
└──────────────────┴─────────────────────────────────────────────┘
```

---

## Step 1006: Patch Management สำหรับ OT

### OT Patch Management Process

```
┌─────────────────────────────────────────────────────────────────┐
│                  OT PATCH MANAGEMENT WORKFLOW                    │
│                                                                   │
│  1. MONITOR                                                       │
│     │  ▪ ติดตาม ICS-CERT advisories                             │
│     │  ▪ Vendor security bulletins                              │
│     │  ▪ CVE database (nvd.nist.gov)                            │
│     ▼                                                             │
│  2. ASSESS                                                        │
│     │  ▪ วิเคราะห์ impact ต่อ OT system                        │
│     │  ▪ Test ใน Lab environment                                │
│     │  ▪ Vendor consultation                                     │
│     ▼                                                             │
│  3. TEST                                                          │
│     │  ▪ ทดสอบใน test environment ที่เหมือน production          │
│     │  ▪ Regression testing                                      │
│     │  ▪ Performance testing                                     │
│     ▼                                                             │
│  4. SCHEDULE                                                      │
│     │  ▪ วางแผนกับ production team                              │
│     │  ▪ เลือก maintenance window                               │
│     │  ▪ Rollback plan                                          │
│     ▼                                                             │
│  5. DEPLOY                                                        │
│     │  ▪ Create backup ก่อน                                      │
│     │  ▪ Apply patch                                            │
│     │  ▪ Verify operation                                       │
│     ▼                                                             │
│  6. VERIFY                                                        │
│        ▪ System health check                                     │
│        ▪ Update asset inventory                                  │
│        ▪ Document change                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Compensating Controls เมือ Patch ไม่ได้

เมื่อ patch ไม่สามารถทำได้ทันที ใช้ compensating controls:

```
Compensating Controls for Unpatched Systems:
1. Network Segmentation   → แยก VLAN ไม่ให้สื่อสารกับระบบอื่น
2. Application Whitelisting → อนุญาตเฉพาะ software ที่รู้จัก
3. Disable Unused Services → ปิด service ที่ไม่ใช้ (RDP, SMB, FTP)
4. Enhanced Monitoring    → Log และ alert ทุก activity
5. Data Diode             → One-way communication เท่านั้น
6. Virtual Patching       → IDS/IPS signature สำหรับช่องโหว่
7. Physical Controls      → ล็อค USB port, เข้าถึงได้เฉพาะผู้มีสิทธิ์
```

---

## Step 1007: Role-Based Access Control (RBAC)

### RBAC สำหรับ OT/SCADA System

```
┌────────────────────────────────────────────────────────────────┐
│                    OT RBAC MATRIX                               │
├──────────────────┬──────┬───────┬────────┬────────┬───────────┤
│ Role             │ View │ Ack   │ Control│ Config │ Engineering│
├──────────────────┼──────┼───────┼────────┼────────┼───────────┤
│ Operator         │  ✓   │  ✓    │  ✓     │  ✗     │  ✗        │
│ Senior Operator  │  ✓   │  ✓    │  ✓     │  Limited│  ✗       │
│ Supervisor       │  ✓   │  ✓    │  ✓     │  ✓     │  ✗        │
│ Maintenance      │  ✓   │  ✓    │  Limited│  ✓    │  ✗        │
│ Engineer         │  ✓   │  ✓    │  ✓     │  ✓     │  ✓        │
│ Vendor (remote)  │  Limited│  ✗ │  ✗     │  Temp  │  Temp     │
│ Administrator    │  ✓   │  ✓    │  ✓     │  ✓     │  ✓        │
└──────────────────┴──────┴───────┴────────┴────────┴───────────┘
```

### Siemens TIA Portal Access Control

```
// TIA Portal - User Administration
// เปิดใช้ User Management ใน Project settings

// ตัวอย่าง User Groups:
// Group: OT_Operators
//   - Monitor only
//   - Acknowledge alarms
//   - Cannot modify program

// Group: OT_Engineers  
//   - Full programming access
//   - Download to PLC
//   - Modify hardware config

// Group: OT_Maintenance
//   - Force I/O
//   - Monitor variables
//   - Cannot download program

// ใน HMI/SCADA - Siemens WinCC
// User Administration → User Groups → Authorizations
AUTHORIZATION "Operator_Level" {
    SCREEN_NAVIGATION = TRUE
    ALARM_ACKNOWLEDGE = TRUE  
    SETPOINT_CHANGE = FALSE
    PLANT_CONTROL = TRUE      // จำกัดเฉพาะบาง object
}

AUTHORIZATION "Engineer_Level" {
    ALL_SCREENS = TRUE
    ALARM_ACKNOWLEDGE = TRUE
    SETPOINT_CHANGE = TRUE
    PLANT_CONTROL = TRUE
    CONFIGURATION = TRUE
}
```

---

## Step 1008: Security Audit

### OT Security Audit Checklist

```
╔══════════════════════════════════════════════════════════════════╗
║              OT SECURITY AUDIT CHECKLIST                         ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  NETWORK SECURITY                                                  ║
║  ☐ Network diagram ถูกต้องและทันสมัย                             ║
║  ☐ Purdue model ได้รับการ implement                              ║
║  ☐ Firewall rules ได้รับการ review                               ║
║  ☐ Remote access ผ่าน VPN เท่านั้น                              ║
║  ☐ Wireless network ได้รับการ secure                             ║
║  ☐ Rogue device detection มีอยู่                                  ║
║                                                                    ║
║  ACCESS CONTROL                                                    ║
║  ☐ Default passwords ถูกเปลี่ยนทั้งหมด                         ║
║  ☐ Individual accounts (ไม่ใช่ shared account)                   ║
║  ☐ Least privilege principle                                      ║
║  ☐ Account termination process มีอยู่                            ║
║  ☐ Privileged access ได้รับการ log                               ║
║  ☐ Multi-factor authentication สำหรับ remote access              ║
║                                                                    ║
║  PATCH MANAGEMENT                                                  ║
║  ☐ Asset inventory ครบถ้วน                                       ║
║  ☐ Vulnerability scanning schedule มีอยู่                        ║
║  ☐ Patch test process มีอยู่                                     ║
║  ☐ Compensating controls สำหรับระบบที่ patch ไม่ได้             ║
║                                                                    ║
║  MONITORING & LOGGING                                              ║
║  ☐ Security log centralized (SIEM)                                ║
║  ☐ Log retention อย่างน้อย 1 ปี                                  ║
║  ☐ Alerting สำหรับ suspicious activity                           ║
║  ☐ 24/7 monitoring หรือ on-call process                          ║
║                                                                    ║
║  PHYSICAL SECURITY                                                 ║
║  ☐ Control room access control                                   ║
║  ☐ USB ports ถูก disable หรือ control                           ║
║  ☐ Laptop/device ที่ต่อต้องได้รับอนุญาต                        ║
║  ☐ Cable management                                               ║
║                                                                    ║
║  INCIDENT RESPONSE                                                 ║
║  ☐ Incident response plan มีอยู่                                 ║
║  ☐ Contact list (CERT, vendor, management) ทันสมัย               ║
║  ☐ Backup และ restore test ทำสม่ำเสมอ                           ║
║  ☐ Tabletop exercises ทำอย่างน้อยปีละครั้ง                      ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Step 1009: Incident Response สำหรับ OT

### OT Incident Response Plan

```
┌─────────────────────────────────────────────────────────────────┐
│                  OT INCIDENT RESPONSE PHASES                     │
│                                                                   │
│  PREPARATION                                                      │
│  ├── Contact list (CERT-TH, ICS-CERT, Vendors)                  │
│  ├── Backup procedures                                           │
│  ├── Offline spare PLC/HMI                                       │
│  └── Tabletop exercises                                          │
│                                                                   │
│  IDENTIFICATION                                                   │
│  ├── Unusual network traffic                                     │
│  ├── Unexpected PLC behavior                                     │
│  ├── Alarm floods                                                │
│  └── Physical indicators                                         │
│                                                                   │
│  CONTAINMENT (OT-specific: Safety First!)                        │
│  ├── ประเมิน: ปลอดภัยจะหยุดระบบไหม?                           │
│  ├── Isolate affected segment (ไม่ใช่ shutdown ทั้งหมด)         │
│  ├── Preserve evidence                                           │
│  └── Notify management and safety team                          │
│                                                                   │
│  ERADICATION                                                      │
│  ├── Remove malware (ด้วย offline tools)                        │
│  ├── Restore from backup                                         │
│  └── Patch vulnerability                                         │
│                                                                   │
│  RECOVERY                                                         │
│  ├── Staged restart (ไม่ start ทุกอย่างพร้อมกัน)               │
│  ├── Enhanced monitoring                                         │
│  └── Verify normal operation                                     │
│                                                                   │
│  POST-INCIDENT                                                    │
│  ├── Root cause analysis                                         │
│  ├── Lessons learned                                             │
│  └── Update response plan                                        │
└─────────────────────────────────────────────────────────────────┘
```

### OT Incident ที่แตกต่างจาก IT

```
OT Incident Response - ข้อควรระวัง:

1. SAFETY FIRST
   - ก่อน isolate ต้องตรวจสอบว่า process ปลอดภัยจะหยุดได้
   - ถ้า process กำลัง running อยู่ในสถานะอันตราย อย่าหยุดทันที
   - ปรึกษา process engineer ก่อนทุกครั้ง

2. EVIDENCE PRESERVATION  
   - ถ่ายรูป screen ก่อน shutdown
   - Export PLC program ก่อน restore
   - Network capture ถ้า traffic ยังอยู่
   - Log files: copy ก่อนลบ

3. DO NOT
   - ห้าม reboot PLC กลางกะงาน
   - ห้าม disconnect network โดยไม่แจ้ง operator
   - ห้าม factory reset โดยไม่มี backup
   - ห้าม install antivirus บน PLC โดยไม่ test ก่อน
```

---

## Step 1010: Siemens Security Features

### TIA Portal Security Settings

```
// Siemens S7-1500 - Security Settings

// 1. Access Levels (Protection Level)
// ──────────────────────────────────
// Full access (no protection)           → ใช้ใน development
// Read access                           → ดูได้ แต่แก้ไขไม่ได้
// HMI access                            → HMI อ่านได้ แต่ download ไม่ได้
// No access (complete protection)       → ต้องใส่ password ทั้งหมด

// ใน TIA Portal: PLC Properties → Protection
ACCESS_LEVEL: FULL_ACCESS_WITH_PUT_GET_COMMUNICATION {
    PASSWORD: "SecureP@ss2024!"   // ต้องซับซ้อน
    // PUT/GET: ให้ OPC server อ่านได้
}

// 2. Secure Communication (TLS)
// ──────────────────────────────
// S7-1500 supports TLS for:
// - OPC UA (default port 4840 with TLS)
// - Webserver (HTTPS)
// - Syslog over TLS

// OPC UA Security:
OPC_UA_SERVER {
    SECURITY_MODE: SIGN_AND_ENCRYPT
    SECURITY_POLICY: BASIC256SHA256
    CERTIFICATE: "S7_1500_OPC_UA_Cert.der"
    USER_AUTH: USERNAME_PASSWORD
    ANONYMOUS: DISABLED   // ห้าม anonymous access
}

// 3. Webserver Security
WEBSERVER {
    HTTPS_ONLY: TRUE
    TLS_VERSION: TLS_1_2_AND_ABOVE
    SELF_SIGNED_CERT: TRUE   // หรือใช้ CA-signed
}
```

### Siemens SINEMA Server - Network Management

```
// SINEMA Network Management - Security Features:
// 1. Device discovery และ inventory
// 2. Topology visualization
// 3. Firmware management (update ผ่าน SINEMA)
// 4. Security audit log
// 5. Anomaly detection (unusual communication patterns)
// 6. Role-based access

// SINEMA Remote Connect - Remote Access:
// - เชื่อมต่อผ่าน Siemens cloud
// - Certificate-based authentication
// - Role-based access per device
// - Session recording
// - Mobile access support
```

---

## Step 1011: Allen-Bradley Security

### FactoryTalk Security

```
// Rockwell/Allen-Bradley - FactoryTalk Security

// 1. FactoryTalk Directory
//    - Centralized authentication
//    - Active Directory integration
//    - SSO (Single Sign-On)

// 2. Studio 5000 (Logix Designer) Security
//    - Controller security properties
//    - Online edit permissions
//    - Change audit trail

// Configure controller security:
CONTROLLER_SECURITY {
    CHANGE_DETECTION: ENABLED    // Detect unauthorized changes
    AUDIT_TRAIL: ENABLED
    ACCESS_LEVEL: {
        PROTECTED: TRUE
        PASSWORD: "FactoryP@ss2024"
    }
}

// 3. Stratix Switch Security (Allen-Bradley Industrial Switch)
//    ใช้ Cisco IOS-based

// VLAN Configuration:
VLAN_DESIGN {
    VLAN_10: "Level_1_Field"        // 192.168.10.0/24
    VLAN_20: "Level_2_Control"      // 192.168.20.0/24  
    VLAN_30: "Level_3_SCADA"        // 192.168.30.0/24
    
    // Inter-VLAN routing ผ่าน Firewall เท่านั้น
    ROUTING: FIREWALL_ONLY
}

// Port Security:
PORT_SECURITY {
    MAX_MAC: 1                    // อนุญาต 1 MAC address ต่อ port
    VIOLATION: SHUTDOWN           // ถ้าเกิน: shutdown port
    STICKY: ENABLED               // Learn MAC automatically
}
```

---

## Step 1012: Defense in Depth Strategy

### Defense in Depth สำหรับ OT

```
┌──────────────────────────────────────────────────────────────────┐
│                    DEFENSE IN DEPTH LAYERS                        │
│                                                                    │
│  Layer 7: Physical Security                                        │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Fence, Guards, Cameras, Access cards, Biometric            │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 6: Network Perimeter                                        │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Firewalls, DMZ, IDS/IPS, Network monitoring                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 5: Network Internal                                         │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ VLAN segmentation, 802.1X, Port security                   │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 4: Host Security                                            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ OS hardening, Application whitelisting, Antivirus          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 3: Application Security                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ RBAC, Secure coding, Patch management                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 2: Data Security                                            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Encryption at rest, Encrypted backup                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Layer 1: Policies & Procedures                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Security policies, Training, Incident response             │  │
│  └─────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Step 1013: Network Monitoring สำหรับ OT

### OT-Specific IDS/IPS Systems

```
┌──────────────────────────────────────────────────────────────────┐
│              OT-Specific Security Monitoring Tools               │
├──────────────────┬───────────────────────────────────────────────┤
│ Product          │ Features                                       │
├──────────────────┼───────────────────────────────────────────────┤
│ Claroty          │ - Passive asset discovery                     │
│                  │ - Protocol deep inspection                    │
│                  │ - Anomaly detection with ML                   │
│                  │ - Integration with SIEM                       │
├──────────────────┼───────────────────────────────────────────────┤
│ Dragos           │ - ICS-focused threat intelligence             │
│                  │ - Attack framework (MITRE ATT&CK for ICS)    │
│                  │ - Incident response support                   │
├──────────────────┼───────────────────────────────────────────────┤
│ Nozomi Networks  │ - Real-time network monitoring                │
│                  │ - Vulnerability assessment                    │
│                  │ - Guardian sensor (passive)                   │
├──────────────────┼───────────────────────────────────────────────┤
│ Fortinet FortiOS │ - Next-gen firewall for OT                   │
│                  │ - OT protocol inspection                      │
│                  │ - SD-WAN for OT                               │
├──────────────────┼───────────────────────────────────────────────┤
│ Siemens          │ - Industrial security scanner                │
│ SINEC Security   │ - Integrated with SINEMA                      │
│ Inspector        │ - Vulnerability assessment                    │
└──────────────────┴───────────────────────────────────────────────┘
```

---

## Step 1014: Secure PLC Programming Practices

### Ladder Diagram - Security Considerations

```
// ตัวอย่าง: Security Interlock ใน Ladder Diagram
// บังคับ login ก่อน operation

// Siemens S7-1500 (TIA Portal)
// Network 1: Authentication Check
// ─────────────────────────────────────────────────────────────

//  "User_Authenticated"    "Timeout_Expired"         "Enable_Control"
//  ──────┤/├────────────────────────┤/├──────────────────( )────

// หมายความว่า: เปิด Enable_Control ได้ก็ต่อเมื่อ authenticated และ ไม่ timeout

// Network 2: Force Bit Security Check
// ─────────────────────────────────────────────────────────────
//  "Force_Mode_Active"                    "Force_Alarm"
//  ──────┤├────────────────────────────────────( )────

// แจ้งเตือนเมื่อมีการ Force bits (อาจเป็นสัญญาณโจมตี)

// Network 3: Program Modification Detection
// ─────────────────────────────────────────────────────────────
//  "Last_Download"   [NOT EQUAL] "Expected_Checksum"   "Program_Modified_Alarm"
//  ──────────────────────────────────────────────────────────────( )────

// ตรวจสอบ checksum ของ program เพื่อตรวจหาการแก้ไขโดยไม่ได้รับอนุญาต
```

### PLC Hardening Checklist

```
PLC Hardening Steps:
════════════════════
□ เปลี่ยน default password ทันที
□ Disable web server (ถ้าไม่ใช้)
□ Disable PUT/GET communication (ถ้าไม่ต้องการ)
□ Disable unused communication ports
□ Enable access protection
□ Configure audit trail
□ Backup configuration ในที่ปลอดภัย
□ Document firmware version
□ ตรวจสอบว่าไม่มี debug mode active ใน production
□ Remove engineering software จาก production network
□ ตรวจสอบ SNMP community string (ไม่ใช้ "public"/"private")
□ Disable Telnet ใช้ SSH แทน (ถ้ามี)
□ Enable TLS สำหรับ web interface
□ Set correct time (NTP) เพื่อ log accuracy
```

---

## Step 1015: Data Diode Technology

### Data Diode สำหรับ OT

```
Data Diode - Hardware One-Way Gate

  OT Network (Level 2-3)     Enterprise Network (Level 4-5)
  ──────────────────────     ─────────────────────────────
  PLC → Historian              Business Systems
       │
       ▼
  ┌─────────────┐
  │  DATA DIODE │  ← Hardware enforced one-way
  │  (Optical   │    (ข้อมูลไหลทางเดียว เท่านั้น)
  │   fiber)    │
  └──────┬──────┘
         │ ข้อมูลผ่านได้ทางเดียวเท่านั้น
         ▼
  Enterprise ──────→ Business Analytics
  Historian

ข้อดี:
- ไม่มีทางย้อนกลับ (physically impossible)
- High assurance security
- ไม่ depend on software/firewall rules

Products:
- Waterfall Security Solutions (Waterfall FLIP)
- Owl Cyber Defense (OPDS-100G)
- Fox-IT DataDiode
- Advenica
```

---

## Step 1016: Security Training และ Awareness

### OT Security Training Program

```
OT Security Training Plan:
═══════════════════════════

MODULE 1: สำหรับ Operator ทุกคน (4 ชั่วโมง/ปี)
├── Social engineering awareness
├── Physical security
├── Password security  
├── Incident reporting
└── USB/device policy

MODULE 2: สำหรับ Engineer (8 ชั่วโมง/ปี)
├── ICS/OT threat landscape
├── Secure remote access
├── Network security basics
├── Secure PLC programming
└── Vulnerability assessment basics

MODULE 3: สำหรับ IT/OT Security Team (40 ชั่วโมง/ปี)
├── IEC 62443 deep dive
├── OT network forensics
├── Incident response for ICS
├── Threat hunting
└── Penetration testing for OT (passive only)

HANDS-ON LAB:
├── Tabletop exercise (cyber attack scenario)
├── Incident response drill
└── Red team/Blue team exercise (safe environment only)
```

---

## Step 1017: Security Architecture สำหรับ Large Plant

### Complete Plant Security Architecture

```
╔════════════════════════════════════════════════════════════════╗
║           LARGE PETROCHEMICAL PLANT SECURITY ARCHITECTURE      ║
╠════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ENTERPRISE ZONE (Level 4-5)                                    ║
║  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────────────────────────────────┐ ║
║  │ ERP │ │Email│ │File │ │   Corporate Active Directory      │ ║
║  │     │ │     │ │Srvr │ │   (Employees/Contractors)         │ ║
║  └─────┘ └─────┘ └─────┘ └──────────────────────────────────┘ ║
║                    │                                            ║
║           ┌────────▼────────┐                                  ║
║           │ Enterprise FW   │ (Palo Alto / Fortinet)           ║
║           └────────┬────────┘                                  ║
║                    │                                            ║
║  DMZ ZONE                                                       ║
║  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐║
║  │   Reverse   │  │ Jump Server  │  │    Data Historian       │║
║  │   Proxy     │  │  (MFA+VPN)   │  │    (Read-only replica)  │║
║  └─────────────┘  └─────────────┘  └─────────────────────────┘║
║                    │                                            ║
║           ┌────────▼────────┐                                  ║
║           │    OT FW        │ (Fortinet FortiGate OT)          ║
║           └────────┬────────┘                                  ║
║                    │                                            ║
║  OT LEVEL 3                                                     ║
║  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ ║
║  │  SCADA   │ │Historian │ │ Security │ │  OT-AD (separate)│ ║
║  │  Server  │ │ (OSIsoft)│ │ Monitor  │ │                  │ ║
║  └──────────┘ └──────────┘ │(Claroty) │ └──────────────────┘ ║
║                             └──────────┘                       ║
║  ─────────────── VLAN Boundary ────────────────────────────── ║
║  OT LEVEL 2                                                     ║
║  Unit 1 DCS  │  Unit 2 PLC  │  Utility PLC  │  Safety SIS     ║
║  (Honeywell) │  (Siemens)   │  (AB/Rockwell)│  (Yokogawa)     ║
║                                                                  ║
║  OT LEVEL 1 (Field Devices - PROFIBUS/FOUNDATION Fieldbus)     ║
╚════════════════════════════════════════════════════════════════╝
```

---

## Step 1018: USB/Removable Media Policy

### Removable Media Control

```
USB Policy สำหรับ OT Environment:

1. KIOSK WORKSTATION (Scanning Station)
   ├── เครื่อง PC แยกต่างหาก ไม่ต่อ OT network
   ├── มี antivirus อัพเดทล่าสุด
   ├── Scan USB ก่อนนำเข้า OT network ทุกครั้ง
   └── Log การ scan ทุกครั้ง

2. APPROVED DEVICES ONLY
   ├── USB drive ต้องได้รับการ register (ขึ้น whitelist)
   ├── ใช้ USB key with hardware encryption
   └── ติดสติ๊กเกอร์ "OT Approved" ที่เห็นชัด

3. TECHNICAL CONTROLS
   ├── Group Policy: Disable autorun/autoplay
   ├── DLP (Data Loss Prevention) บน HMI
   ├── Device Control Software (Ivanti, McAfee DLP)
   └── Alert เมื่อ USB ถูกต่อ (audit log)

4. PROCEDURE
   ├── ยื่นขออนุญาตกับ supervisor
   ├── Scan ที่ kiosk
   ├── Transfer file แล้วถอดทันที
   └── รายงานถ้าพบ USB ที่ไม่ทราบแหล่งที่มา
```

---

## Step 1019: Wireless Security สำหรับ OT

### Industrial Wireless Security

```
Industrial Wireless Security Design:

SSID Design:
┌────────────────────────────────────────────────────────────┐
│  SSID: PLANT_OT_SECURE          │  SSID: PLANT_CORP         │
│  Security: WPA3-Enterprise      │  Security: WPA3-Personal  │
│  Auth: 802.1X + Certificates    │  Auth: Pre-shared key     │
│  VLAN: OT_Wireless (VLAN 50)   │  VLAN: Corporate (VLAN 10)│
│  Devices: Field tablets,        │  Devices: Office laptops  │
│           handheld scanners     │                           │
└────────────────────────────────────────────────────────────┘

Infrastructure:
├── Industrial AP: Cisco Catalyst IW (IP67, DIN rail)
├── Controller: Cisco WLC 9800 (onsite)
├── Rogue AP Detection: ENABLED
├── Client Isolation: ENABLED (no client-to-client)
├── Management SSID: HIDDEN + separate VLAN
└── Physical: AP positions documented, access controlled

Access Policy:
├── เฉพาะอุปกรณ์ที่ register certificate เท่านั้น
├── ห้าม BYOD ใน OT wireless
├── Session timeout: 8 ชั่วโมง
└── Location tracking: ENABLED (audit trail)
```

---

## Step 1020: Lab Project - OT Security Implementation

### Lab: Design และ Implement OT Security สำหรับ Water Treatment Plant

**โจทย์**: โรงบำบัดน้ำประปา มีระบบดังนี้:
- 2x Siemens S7-1500 PLC (สำหรับ intake และ treatment)
- 1x Rockwell ControlLogix (สำหรับ distribution)
- Wonderware InTouch SCADA
- Historian server (OSIsoft PI)
- 5 Operator workstations

**งาน**:

```
PART 1: Network Architecture Design (30 คะแนน)
─────────────────────────────────────────────────
1. วาด network diagram ตาม Purdue model
2. กำหนด IP addressing plan
3. ออกแบบ VLAN structure
4. ระบุ firewall rules ที่จำเป็น

PART 2: Security Configuration (40 คะแนน)
─────────────────────────────────────────────────
1. Configure S7-1500 access protection
2. Set up TIA Portal user management
3. Design RBAC matrix สำหรับทีมงาน
4. Configure OPC UA security

PART 3: Policies and Procedures (30 คะแนน)
─────────────────────────────────────────────────
1. เขียน Remote Access Policy
2. เขียน USB/Media Policy
3. เขียน Incident Response Plan (OT-specific)
4. ออกแบบ Security Training Plan

DELIVERABLES:
├── Network diagram (Visio/draw.io)
├── Security configuration document
├── Policy documents (3 policies)
├── Incident response plan
└── Presentation (15 นาที)

EVALUATION CRITERIA:
├── IEC 62443 alignment
├── Practicality (ทำได้จริง)
├── Defense in depth application
└── Clear documentation
```

---

## สรุป Part 51

| หัวข้อ | ความสำคัญ |
|--------|-----------|
| IEC 62443 | มาตรฐานหลักที่ต้องรู้ |
| Purdue Model | Framework สำหรับ network segmentation |
| Defense in Depth | หลักการหลาย layer |
| Patch Management | กระบวนการที่ต้องมี |
| Incident Response | เตรียมพร้อมก่อนเกิดเหตุ |

---

## Navigation

- [← Part 50: Advanced Motion Control](../part41-60/part50_advanced_motion.md)
- [→ Part 52: Industrial Network Architecture](./part52_network_architecture.md)
- [↑ สารบัญหลัก](../README.md)
