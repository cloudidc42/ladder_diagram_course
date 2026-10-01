# Part 99: Comprehensive Review - ทบทวนทั้งหมด
## Steps 1961-1980 | ระดับ Master Review

---

## การนำทาง
- [← Part 98: Future Technologies](part98_future_technologies.md)
- [→ Part 100: Course Completion](part100_course_completion.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1961: PLC Programming Quick Reference

### 5 ภาษา IEC 61131-3

```
┌──────────────┬───────────────────────────────────────────────────────┐
│ Language     │ ลักษณะ / Best For                                    │
├──────────────┼───────────────────────────────────────────────────────┤
│ Ladder (LD)  │ ไฟฟ้า/Relay Logic, ง่ายสุด, Maintenance ทำได้       │
│ FBD          │ Analog/Control, Signal Flow, PID, Math               │
│ ST           │ Complex Logic, Data Processing, Text-Based           │
│ IL           │ Low-level, ไม่ค่อยใช้แล้ว (Assembly-like)           │
│ SFC          │ Sequential Process, State Machine, Batch             │
└──────────────┴───────────────────────────────────────────────────────┘
SCL (Siemens ST) / FBD เพิ่มเติม:
- CASE...OF   = Select Case
- FOR...DO    = For Loop
- WHILE...DO  = While Loop
- REPEAT...UNTIL = Repeat Loop
```

### PLC Brand Quick Reference

```
┌──────────────┬───────────────┬────────────────┬──────────────────────┐
│ Brand        │ Software      │ Top PLC Model  │ Best For             │
├──────────────┼───────────────┼────────────────┼──────────────────────┤
│ Siemens      │ TIA Portal    │ S7-1500        │ EU, Process, Auto    │
│ Allen-Bradley│ Studio 5000   │ ControlLogix   │ Americas, Food, MFG  │
│ Mitsubishi   │ GX Works3     │ iQ-R           │ Asia, High Speed     │
│ Omron        │ Sysmac Studio │ NX/NJ          │ Motion, Machine      │
│ Schneider    │ EcoStruxure   │ M580           │ Power, Buildings     │
│ ABB          │ Automation    │ AC500          │ Process, Marine      │
│ Beckhoff     │ TwinCAT 3     │ C6030          │ High Speed, PC-Based │
│ FANUC        │ ROBOGUIDE     │ CRX-10iA       │ Welding, Assembly    │
└──────────────┴───────────────┴────────────────┴──────────────────────┘
```

---

## Step 1962: Industrial Network Protocols Summary

```
Protocol Comparison Matrix:
┌─────────────────┬──────────┬──────────┬────────┬───────────────────────┐
│ Protocol        │ Speed    │ Topology │ Layer  │ Main Use              │
├─────────────────┼──────────┼──────────┼────────┼───────────────────────┤
│ PROFINET RT     │ < 1ms    │ Star/Ring│ L2/L3  │ Siemens I/O, Drives   │
│ PROFINET IRT    │ < 250µs  │ Ring     │ L2     │ Servo Motion          │
│ EtherNet/IP     │ < 1ms    │ Star     │ L2/L3  │ Rockwell Devices      │
│ Modbus TCP      │ 50-500ms │ Any      │ L3     │ Simple Sensors        │
│ Modbus RTU      │ 9600-115K│ Bus      │ Serial │ Legacy Devices        │
│ PROFIBUS DP     │ 9.6Mbps  │ Bus      │ Serial │ Legacy Siemens        │
│ CANopen         │ 1Mbps    │ Bus      │ Serial │ Embedded, Vehicles    │
│ EtherCAT        │ < 100µs  │ Daisy    │ L2     │ Ultra-high Speed      │
│ OPC UA          │ Variable │ Any      │ L3     │ Interoperability      │
│ MQTT            │ Variable │ Broker   │ L3     │ IIoT, Cloud           │
│ HART            │ 1.2kbps  │ Bus/Point│ 4-20mA │ Smart Instruments     │
│ APL             │ 10Mbps   │ Bus      │ 2-Wire │ Hazardous Area        │
│ WirelessHART    │ 250kbps  │ Mesh     │ 2.4GHz │ Wireless Instruments  │
│ ISA100.11a      │ 250kbps  │ Mesh     │ 2.4GHz │ Process Wireless      │
└─────────────────┴──────────┴──────────┴────────┴───────────────────────┘
```

---

## Step 1963: Industrial Standards Quick Reference

```
Standards Summary:

SAFETY:
IEC 61508  → SIL 1-4, Base Standard สำหรับ Functional Safety
IEC 61511  → SIL สำหรับ Process Industry (Oil & Gas, Chemical)
ISO 13849  → PL a-e สำหรับ Machinery Safety
IEC 62061  → SIL สำหรับ Machinery
ISO 12100  → Risk Assessment สำหรับ Machinery
ISO/TS 15066 → Collaborative Robot (Cobot) Safety

COMMUNICATION:
IEC 61131-3 → PLC Programming Languages (5 Languages)
ISA-88     → Batch Control (ISA-88 / S88)
ISA-95     → Enterprise-Manufacturing Integration (Levels 0-4)
ISA-18.2   → Alarm Management
OPC UA (IEC 62541) → Open Connectivity

CYBERSECURITY:
IEC 62443  → Industrial Cybersecurity (Zones & Conduits)
NIST SP800-82 → Guide to ICS Security
NERC CIP   → Power Grid Cybersecurity (USA)

PROCESS:
ISA-5.1    → Instrumentation Symbols (P&ID)
NEMA / IEC → Motor Classifications
ATEX/IECEx → Hazardous Area Equipment

QUALITY:
ISO 22000  → Food Safety Management
HACCP      → Hazard Analysis Critical Control Points
FDA 21 CFR Part 11 → Electronic Records (Pharma)
GMP        → Good Manufacturing Practice
```

---

## Step 1964: OEE คำนวณและ Six Big Losses

```
OEE = Availability × Performance × Quality

Availability = Operating Time / Planned Production Time
Performance  = (Ideal Cycle × Total Count) / Operating Time
Quality      = Good Count / Total Count

World Class: OEE ≥ 85%
A: ≥ 90%, P: ≥ 95%, Q: ≥ 99.9%

Six Big Losses (TPM):
AVAILABILITY LOSSES:
1. Planned Stops       → PM, Setup, Changeover
2. Unplanned Stops     → Breakdowns, E-Stop, Faults

PERFORMANCE LOSSES:
3. Minor Stops         → Jams < 5 min, No Material
4. Slow Cycles         → Running Below Rated Speed

QUALITY LOSSES:
5. Process Defects     → Scrap, Rework during Production
6. Startup Rejects     → First Article, Warm-up Scrap

เพิ่มเติมสำหรับ OEE Analysis:
TEEP = OEE × Utilization (ใช้เวลา 24/7 จริงแค่ไหน)
MTBF = Mean Time Between Failures (อยากให้สูง)
MTTR = Mean Time To Repair (อยากให้ต่ำ)
```

---

## Step 1965: Safety System Formulas

```
Functional Safety Key Formulas:

PFD (Probability of Failure on Demand):
PFD_avg = λDU × T1/2

where:
λDU = Dangerous Undetected Failure Rate (per hour)
T1  = Proof Test Interval (hours)

PFHd (Probability of Failure per Hour):
สำหรับ High Demand หรือ Continuous Mode

SIL Classification:
SIL 1: 10E-2 ≤ PFDavg < 10E-1
SIL 2: 10E-3 ≤ PFDavg < 10E-2
SIL 3: 10E-4 ≤ PFDavg < 10E-3

Architecture:
1oo1 (1 out of 1): PFDavg = λDU × T1/2
1oo2 (1 out of 2): PFDavg = (λDU × T1)²/3
2oo3 (2 out of 3): PFDavg = λDU² × T1² (Voting)

Performance Level (PL) - ISO 13849:
PLa: PFHd ≥ 10E-5/hr
PLb: 3×10E-6 ≤ PFHd < 10E-5/hr
PLc: 10E-6 ≤ PFHd < 3×10E-6/hr
PLd: 10E-7 ≤ PFHd < 10E-6/hr
PLe: 10E-8 ≤ PFHd < 10E-7/hr
```

---

## Step 1966: Industry 4.0 Architecture Summary

```
Industry 4.0 Technology Stack:

LAYER 5: Business Intelligence
└── Power BI, Tableau, Custom Dashboard
└── SAP Analytics Cloud, AWS QuickSight

LAYER 4: Enterprise (ERP/MES)
└── SAP S/4HANA, Oracle, Microsoft D365
└── Opcenter MES, Plex, Ignition MES

LAYER 3: Industrial Cloud / IIoT Platform
└── AWS IoT Greengrass, Azure IoT Hub
└── Siemens MindSphere, GE Predix
└── OSIsoft PI (AspenTech) Historian

LAYER 2: Edge Computing
└── Siemens IPC, Advantech MIC, RevPi
└── Docker Containers (Node-RED, InfluxDB, Grafana)
└── OPC UA Server, Edge AI Inference

LAYER 1: Plant Floor
└── PLC, DCS, Drives, Robots
└── Smart Sensors (APL, WirelessHART)
└── Vision Systems, Safety Systems

KEY ENABLING TECHNOLOGIES:
- OPC UA: Vertical Integration
- MQTT + Sparkplug B: Horizontal Integration
- UNS: Single Source of Truth
- Digital Twin: Virtual Counterpart
- AI/ML: Optimization & Prediction
- 5G/WiFi6: Wireless Factory
```

---

## Step 1967: Python Libraries สำหรับ Industrial Automation

```python
# ===============================================
# Python Libraries Master Reference
# สำหรับ Industrial Automation
# ===============================================

# OPC UA
from opcua import Client, Server          # opcua-asyncio
from asyncua import Client, Server         # asyncua (async)

# MQTT / IIoT
import paho.mqtt.client as mqtt            # MQTT Client
from sparkplug_b import SparkplugB        # Sparkplug B

# Modbus
from pymodbus.client import ModbusTcpClient, ModbusSerialClient

# Database
import psycopg2                            # PostgreSQL
from influxdb_client import InfluxDBClient # InfluxDB
import pyodbc                              # SQL Server / ODBC

# Industrial Data
import pandas as pd                        # Data Analysis
import numpy as np                         # Numerical
from scipy import signal                   # Signal Processing

# Machine Learning
from sklearn.ensemble import IsolationForest, RandomForestClassifier
import tensorflow as tf                    # Deep Learning
import torch                               # PyTorch
import xgboost as xgb                     # XGBoost
import shap                                # Explainability

# Web API
from fastapi import FastAPI               # REST API Server
import requests                           # HTTP Client
import aiohttp                            # Async HTTP

# Docker / Containers
import docker                             # Docker SDK

# Visualization
import matplotlib.pyplot as plt          # Plotting
import plotly.express as px              # Interactive
import grafanalib as gl                  # Grafana

# Serial / Hardware
import serial                            # RS232/485
import smbus2                            # I2C
import RPi.GPIO as GPIO                  # Raspberry Pi

# Standards
from pymodbus.constants import Endian
from pymodbus.payload import BinaryPayloadDecoder
```

---

## Step 1968-1980: Exam Preparation Questions

### 50 คำถาม Exam Prep

```
Q1: Scan Cycle ของ PLC มีกี่ขั้นตอน? อะไรบ้าง?
A: 3 ขั้นตอน: Input Scan → Program Execution → Output Scan

Q2: Timer ON-Delay ทำงานอย่างไร?
A: นับเมื่อ Input = 1, Output = 1 หลังครบ Preset Time

Q3: Safety PLC ต้องการ Certification อะไร?
A: PLe (ISO 13849) หรือ SIL 2/3 (IEC 61508)

Q4: PROFINET IRT ใช้สำหรับอะไร?
A: Servo Motion Control (< 250µs deterministic)

Q5: OEE คำนวณอย่างไร?
A: OEE = Availability × Performance × Quality

Q6: CCP ในระบบ HACCP คืออะไร?
A: Critical Control Point = จุดที่ต้องควบคุมเพื่อความปลอดภัยอาหาร

Q7: ISA-88 กำหนด Hierarchy อะไรบ้าง?
A: Procedure > Unit Procedure > Operation > Phase > Step

Q8: UNS ย่อมาจากอะไร?
A: Unified Namespace

Q9: SIL 2 ต้องการ PFD ช่วงไหน?
A: 10E-3 ≤ PFDavg < 10E-2

Q10: Sparkplug B คืออะไร?
A: MQTT Namespace Standard สำหรับ Industrial IIoT

Q11: Edge Computing ต่างจาก Cloud Computing อย่างไร?
A: Edge ประมวลผล ใกล้ Data Source (Latency ต่ำ, Offline ได้)

Q12: MTBF คืออะไร?
A: Mean Time Between Failures (ยิ่งสูงยิ่งดี)

Q13: 5 ภาษา IEC 61131-3 มีอะไรบ้าง?
A: LD, FBD, ST, IL, SFC

Q14: ISA-95 Level 3 คือระดับอะไร?
A: Operations / MES Level

Q15: OPC UA Port Default คือ?
A: TCP 4840

Q16: HACCP ย่อมาจากอะไร?
A: Hazard Analysis and Critical Control Points

Q17: CT Value ใน Water Treatment คืออะไร?
A: Concentration × Time = ประสิทธิภาพ Disinfection

Q18: APL Protocol มี Data Rate เท่าไหร่?
A: 10 Mbps บน 2-Wire

Q19: IEC 62443 เกี่ยวกับอะไร?
A: Industrial Cybersecurity Standard

Q20: Digital Twin มีกี่ประเภท?
A: 4 ประเภท: Descriptive, Informative, Predictive, Autonomous

Q21: 6 Big Losses ใน OEE มีอะไรบ้าง?
A: Planned Stops, Breakdowns, Minor Stops, Slow Cycles, Process Defects, Startup Defects

Q22: PROFIBUS DP Speed สูงสุดคือ?
A: 12 Mbps (12 MB/s ในทางเทคนิค = 12 Mbit/s)

Q23: วิธี Grounding Shielded Cable ถูกต้องคืออะไร?
A: ต่อ Shield ด้านเดียว (เพื่อป้องกัน Ground Loop)

Q24: Modbus Function Code 3 หมายความว่าอะไร?
A: Read Holding Registers

Q25: ATEX Category สำหรับ Zone 0 คืออะไร?
A: Category 1G

Q26: ISA-18.2 เกี่ยวกับอะไร?
A: Alarm Management Standard

Q27: EMI ย่อมาจากอะไร?
A: Electromagnetic Interference

Q28: VDA 5050 เกี่ยวกับอะไร?
A: AGV/AMR Communication Interface Standard

Q29: YOLO ในงาน Machine Vision ย่อมาจากอะไร?
A: You Only Look Once (Object Detection Algorithm)

Q30: Redundancy แบบ 1oo2 หมายความว่าอะไร?
A: ต้องมีอย่างน้อย 1 ใน 2 ช่องทำงาน ถึง Output จะ ON

Q31: HMI ย่อมาจากอะไร?
A: Human-Machine Interface

Q32: SCADA ย่อมาจากอะไร?
A: Supervisory Control and Data Acquisition

Q33: PID Integral Term ใช้แก้ปัญหาอะไร?
A: Steady-State Error (Offset)

Q34: Fieldbus จำแนกตาม IEC 61784 กี่ Family?
A: 18 Communication Profile Families

Q35: Safety Integrity Level (SIL) มีกี่ระดับ?
A: 4 ระดับ (SIL 1-4) หรือ 3 ระดับ สำหรับ ISO 13849 (PLa-PLe)

Q36: CMMS ย่อมาจากอะไร?
A: Computerized Maintenance Management System

Q37: ISA-100.11a ใช้ Frequency ไหน?
A: 2.4 GHz (IEEE 802.15.4)

Q38: B2MML คืออะไร?
A: Business to Manufacturing Markup Language (XML Schema ISA-95)

Q39: Proof Test ใน Functional Safety คืออะไร?
A: การทดสอบ Safety Function เป็นระยะเพื่อยืนยัน PFHD ยังดีอยู่

Q40: LNG ย่อมาจากอะไร? (ใน Oil & Gas Context)
A: Liquefied Natural Gas

Q41: EU Blue Card (Germany) ต้องการ Salary ขั้นต่ำเท่าไหร่?
A: €56,400/year (2024)

Q42: ISA CAP ย่อมาจากอะไร?
A: Certified Automation Professional

Q43: Ziegler-Nichols เป็น Method สำหรับอะไร?
A: PID Tuning

Q44: 5G URLLC ย่อมาจากอะไร?
A: Ultra-Reliable Low Latency Communications

Q45: IEC 61508 กำหนด SIL ไว้กี่ระดับ?
A: 4 ระดับ (SIL 1, 2, 3, 4)

Q46: Bottle-neck station ใน Assembly Line คือสถานีแบบไหน?
A: สถานีที่มี Cycle Time ยาวที่สุด ทำให้ Line ต้องรอ

Q47: Robot Teaching Pendant ใช้ทำอะไร?
A: สอน/บันทึก Robot Position และ Program

Q48: FANUC Karel ใช้ Language อะไร?
A: Pascal-based Language

Q49: URScript ใช้สำหรับ Robot อะไร?
A: Universal Robots (UR3, UR5, UR10, UR16)

Q50: วิธีป้องกัน PLC Crash จาก Watchdog Timer?
A: Program ต้องรัน Watchdog Reset ทุก Scan Cycle ก่อนหมด Time
```

---

## สรุป Part 99

ทบทวนครอบคลุม:
- ✅ PLC Programming (5 ภาษา, Multi-Brand Comparison)
- ✅ Industrial Networks (12+ Protocols)
- ✅ Standards (Safety, Communication, Quality)
- ✅ OEE + Six Big Losses Formulas
- ✅ Safety System Formulas (PFD, SIL, PL)
- ✅ Industry 4.0 Architecture
- ✅ Python Libraries Reference
- ✅ 50 Exam Prep Questions + Answers

---

## การนำทาง
- [← Part 98: Future Technologies](part98_future_technologies.md)
- [→ Part 100: Course Completion](part100_course_completion.md)
- [📚 สารบัญหลัก](../README.md)
