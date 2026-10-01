# Part 52: Industrial Network Architecture (Steps 1021-1040)

## บทนำ: ความสำคัญของ Industrial Network Design

เครือข่ายอุตสาหกรรมที่ออกแบบดีจะส่งผลโดยตรงต่อ:
- **Reliability**: ระบบทำงานต่อเนื่องไม่มีหยุด
- **Performance**: Response time เหมาะสมสำหรับ real-time control
- **Security**: ปกป้องจากภัยคุกคาม
- **Scalability**: ขยายได้ในอนาคต
- **Maintainability**: ง่ายต่อการแก้ไขและ troubleshoot

---

## Step 1021: OT Network Design Principles

### หลักการออกแบบเครือข่าย OT

```
OT Network Design Principles:

1. AVAILABILITY FIRST
   ├── Redundant paths (ไม่มี single point of failure)
   ├── Rapid failover (< 300ms สำหรับ PROFINET IRT)
   └── QoS สำหรับ critical traffic

2. DETERMINISM
   ├── Real-time protocols ต้องการ guaranteed delivery time
   ├── PROFINET IRT: cycle time 31.25μs - 4ms
   ├── EtherNet/IP: cycle time 1-100ms
   └── Modbus TCP: ไม่มี real-time guarantee

3. SEGMENTATION
   ├── แยก OT/IT network
   ├── แยก Level ตาม Purdue Model
   └── แยก Safety (SIS) จาก BPCS

4. DOCUMENTATION
   ├── Network diagram ทันสมัยเสมอ
   ├── IP address plan
   ├── VLAN table
   └── Cable schedule

5. FUTURE-PROOFING
   ├── IPv6 readiness
   ├── Bandwidth headroom 50% minimum
   └── Spare port capacity 20%
```

---

## Step 1022: Industrial Switch Selection

### Managed vs Unmanaged Switch

```
┌─────────────────────┬──────────────────────┬────────────────────────┐
│ Feature             │ Unmanaged Switch      │ Managed Switch         │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ ราคา               │ ถูก (500-2000 บาท)  │ แพง (5,000-50,000 บาท)│
├─────────────────────┼──────────────────────┼────────────────────────┤
│ VLAN               │ ✗                    │ ✓                      │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ QoS                │ ✗                    │ ✓                      │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ Port Mirroring     │ ✗                    │ ✓ (สำหรับ monitoring) │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ Ring Redundancy    │ ✗                    │ ✓ (MRP, RSTP)         │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ PROFINET           │ Limited              │ ✓ (LLDP, MRP)         │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ Remote Management  │ ✗                    │ ✓ (SNMP, Web, CLI)    │
├─────────────────────┼──────────────────────┼────────────────────────┤
│ Diagnostics        │ None                 │ ✓ (Port stats, alarms)│
├─────────────────────┼──────────────────────┼────────────────────────┤
│ ใช้ที่ไหน          │ Small, isolated       │ Medium-Large plant     │
│                     │ cells only           │ ทุกระดับ               │
└─────────────────────┴──────────────────────┴────────────────────────┘
```

### Industrial Switch Products Comparison

```
┌───────────────────────┬──────────────────────────────────────────────┐
│ Vendor/Model          │ Key Features                                  │
├───────────────────────┼──────────────────────────────────────────────┤
│ Siemens SCALANCE X    │ - DIN rail mount                             │
│ (X208, X308, XR524)   │ - PROFINET support native                   │
│                       │ - MRP/STP/RSTP                               │
│                       │ - Integrated with TIA Portal                 │
│                       │ - SIMATIC diagnostics                        │
├───────────────────────┼──────────────────────────────────────────────┤
│ Allen-Bradley Stratix │ - Cisco IOS based                            │
│ (5400, 5700, 5800)    │ - EtherNet/IP native                        │
│                       │ - IGMP, QoS, Port Security                  │
│                       │ - Studio 5000 integration                   │
├───────────────────────┼──────────────────────────────────────────────┤
│ Cisco IE (Industrial  │ - Hardened hardware                          │
│ Ethernet: IE2000,     │ - Cisco IOS features                        │
│ IE3400, IE5000)       │ - Extended temp range                        │
│                       │ - DNA Center management                      │
├───────────────────────┼──────────────────────────────────────────────┤
│ Hirschmann (Belden)   │ - DIN rail, rack mount                      │
│ (RS20, RS30, MACH)    │ - HIPER-Ring (proprietary fast recovery)    │
│                       │ - OpenRail OS                                │
│                       │ - IEC 61850 support                         │
├───────────────────────┼──────────────────────────────────────────────┤
│ Moxa (EDS-G series)   │ - Wide temperature (-40°C to +75°C)         │
│                       │ - Turbo Ring (20ms recovery)                 │
│                       │ - DIN rail, good value                       │
└───────────────────────┴──────────────────────────────────────────────┘
```

---

## Step 1023: VLAN Design สำหรับ Industrial Network

### VLAN Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                   INDUSTRIAL VLAN DESIGN                            │
│                                                                      │
│  VLAN 10: Management Network (192.168.10.0/24)                     │
│  ├── Network switches management interface                          │
│  ├── PLCs network management port                                   │
│  └── NTP server                                                     │
│                                                                      │
│  VLAN 20: Level 1 - Field Devices (192.168.20.0/24)               │
│  ├── PROFINET devices (IO modules, drives)                         │
│  ├── EtherNet/IP devices                                           │
│  └── Remote I/O racks                                              │
│                                                                      │
│  VLAN 30: Level 2 - Control (192.168.30.0/24)                     │
│  ├── PLCs (S7-1500, ControlLogix)                                  │
│  ├── DCS controllers                                               │
│  └── Safety controllers (SIL)                                      │
│                                                                      │
│  VLAN 40: Level 2 - HMI (192.168.40.0/24)                        │
│  ├── Operator HMI stations                                         │
│  ├── Panel PCs                                                     │
│  └── Engineering workstations (OT)                                 │
│                                                                      │
│  VLAN 50: Level 3 - SCADA (192.168.50.0/24)                      │
│  ├── SCADA servers                                                 │
│  ├── Historian servers                                             │
│  └── OPC servers                                                   │
│                                                                      │
│  VLAN 60: Safety (192.168.60.0/24)                                │
│  ├── Safety PLC (SIL2/SIL3)                                       │
│  ├── Emergency shutdown devices                                    │
│  └── Fire & Gas system                                             │
│  *** ISOLATED - ไม่ route ไปไหนทั้งนั้น ***                      │
│                                                                      │
│  VLAN 70: Industrial Wireless (192.168.70.0/24)                   │
│  ├── Handheld tablets                                              │
│  └── Mobile HMI devices                                            │
└────────────────────────────────────────────────────────────────────┘
```

### VLAN Configuration Example (Siemens SCALANCE)

```
// SCALANCE X-200/X-300 VLAN Configuration
// Configuration via Web interface หรือ SINEMA

// Trunk Port (ต่อกับ Switch อื่น หรือ Router/Firewall):
PORT_CONFIG {
    PORT: P1
    MODE: TRUNK
    NATIVE_VLAN: 1       // Untagged: Management
    ALLOWED_VLANS: 10,20,30,40,50
}

// Access Port (ต่อกับ PLC หรือ Device):
PORT_CONFIG {
    PORT: P2
    MODE: ACCESS
    VLAN: 30             // PLC Control Network
}

PORT_CONFIG {
    PORT: P3
    MODE: ACCESS
    VLAN: 20             // Field Device Network
}

// ตัวอย่าง IP Table:
// VLAN 10 Management:    192.168.10.0/24  GW: 192.168.10.1
//   SCALANCE-1:           192.168.10.11
//   SCALANCE-2:           192.168.10.12
//   NTP Server:           192.168.10.100
//
// VLAN 30 Control PLCs:  192.168.30.0/24  GW: 192.168.30.1
//   S7-1500 Unit1:        192.168.30.10
//   S7-1500 Unit2:        192.168.30.11
//   ControlLogix:         192.168.30.20
//
// VLAN 40 HMI:           192.168.40.0/24  GW: 192.168.40.1
//   HMI Station 1:        192.168.40.10
//   HMI Station 2:        192.168.40.11
//   Eng Workstation:      192.168.40.50
```

---

## Step 1024: Network Redundancy - STP, RSTP, MRP

### Spanning Tree Protocol (STP) ใน OT

```
STP/RSTP Comparison:
┌────────────────────┬────────────────┬────────────────────────────┐
│ Protocol           │ Recovery Time  │ ใช้ที่ไหน                 │
├────────────────────┼────────────────┼────────────────────────────┤
│ STP (802.1D)       │ 30-50 seconds  │ ไม่เหมาะ OT              │
├────────────────────┼────────────────┼────────────────────────────┤
│ RSTP (802.1w)      │ 1-6 seconds    │ OK สำหรับ IT/OT border    │
├────────────────────┼────────────────┼────────────────────────────┤
│ MSTP (802.1s)      │ 1-6 seconds    │ Multiple VLAN instances   │
├────────────────────┼────────────────┼────────────────────────────┤
│ MRP (IEC 62439-2)  │ < 200ms        │ PROFINET networks         │
├────────────────────┼────────────────┼────────────────────────────┤
│ PROFINET MRP       │ < 200ms        │ Native PROFINET           │
├────────────────────┼────────────────┼────────────────────────────┤
│ Turbo Ring (Moxa)  │ < 20ms         │ Moxa switches             │
├────────────────────┼────────────────┼────────────────────────────┤
│ HIPER-Ring         │ < 30ms (typ)   │ Hirschmann switches       │
└────────────────────┴────────────────┴────────────────────────────┘
```

### MRP Ring Configuration (PROFINET)

```
// PROFINET MRP Ring Configuration
// ──────────────────────────────────────────────────────────────

// Network Topology: Ring with 4 switches
//
//  ┌───Switch 1 (MRM)──────Switch 2──────Switch 3──────Switch 4─┐
//  │    (Manager)            (Client)     (Client)    (Client)    │
//  └────────────────────────────────────────────────────────────┘
//                      MRP Ring
//
// Switch 1 = MRM (Media Redundancy Manager)
// Switch 2,3,4 = MRC (Media Redundancy Client)

// Siemens SCALANCE Configuration (via TIA Portal):

MRP_RING_CONFIG {
    SWITCH_1: {
        ROLE: MRM             // Manager - ตรวจสอบ ring integrity
        DOMAIN_UUID: "12345678-1234-1234-1234-123456789ABC"
        RING_PORT_1: P1       // เชื่อมกับ Switch 4
        RING_PORT_2: P2       // เชื่อมกับ Switch 2
    }
    
    SWITCH_2: {
        ROLE: MRC             // Client
        DOMAIN_UUID: "12345678-1234-1234-1234-123456789ABC"
        RING_PORT_1: P1       // เชื่อมกับ Switch 1
        RING_PORT_2: P2       // เชื่อมกับ Switch 3
    }
    // Similar for Switch 3, 4
}

// Recovery Time: < 200ms (IEC 62439-2 standard)
// ถ้า link ขาด: MRM ตรวจพบใน 200ms และ activate backup path
```

---

## Step 1025: PROFINET Network Design

### PROFINET Topology Options

```
PROFINET Topology Types:
═══════════════════════

1. LINE TOPOLOGY (Star-of-Line)
   PLC → Switch → IO-Device1 → IO-Device2 → IO-Device3
   ├── ง่าย, ประหยัด
   ├── ไม่ redundant
   └── ใช้ IO-Device ที่มี 2 port (forwarding)

   [PLC]──[SW]──[IO1]──[IO2]──[IO3]

2. STAR TOPOLOGY  
   PLC → Switch ← IO Devices (แต่ละตัวต่อตรงกับ switch)
   ├── ง่ายต่อ troubleshoot
   ├── ต้องการ port switch มาก
   └── ถ้า switch เสีย ทุกอย่างหยุด

   [PLC]──[SWITCH]──[IO1]
               └────[IO2]
               └────[IO3]

3. RING TOPOLOGY (MRP)
   Switch1(MRM) ─ Switch2 ─ Switch3 ─ Switch4 ─ (back to Switch1)
   ├── Redundant: link ขาดได้ 1 จุด
   ├── Recovery < 200ms
   └── ต้องการ managed switch ทั้งหมด

4. TREE TOPOLOGY (Recommended for Large Plants)
   ┌── Level Switch ── Cell Switch ── IO Devices (Unit 1)
   │
   PLC Switch
   │
   └── Level Switch ── Cell Switch ── IO Devices (Unit 2)
   
   ├── Scalable
   ├── ง่ายต่อ management
   └── แต่ละ cell เป็น independent ring (MRP)
```

---

## Step 1026: Industrial Wireless - 802.11

### WiFi Design สำหรับ Industrial

```
Industrial WiFi Standards:
┌──────────────────┬──────────────────────────────────────────────┐
│ Standard         │ Features สำหรับ Industrial                  │
├──────────────────┼──────────────────────────────────────────────┤
│ 802.11n (WiFi 4) │ - 2.4GHz/5GHz dual band                    │
│                  │ - 600Mbps max                                │
│                  │ - Fast roaming: 802.11r                      │
├──────────────────┼──────────────────────────────────────────────┤
│ 802.11ac (WiFi 5)│ - 5GHz only                                 │
│                  │ - 3.5Gbps max                                │
│                  │ - MU-MIMO                                    │
├──────────────────┼──────────────────────────────────────────────┤
│ 802.11ax (WiFi 6)│ - 2.4/5/6GHz                               │
│                  │ - 9.6Gbps max                                │
│                  │ - OFDMA (better for many devices)           │
│                  │ - Target Wake Time (battery)                 │
├──────────────────┼──────────────────────────────────────────────┤
│ 802.11p (WAVE)   │ - Vehicular communication                   │
│                  │ - 5.9GHz                                     │
│                  │ - Low latency (outdoor, fast moving)         │
├──────────────────┼──────────────────────────────────────────────┤
│ 802.11s (Mesh)   │ - Wireless mesh networking                  │
│                  │ - Self-healing                               │
│                  │ - Good for outdoor coverage                  │
└──────────────────┴──────────────────────────────────────────────┘

Industrial AP Requirements:
├── IP65/IP67 (outdoor) หรือ IP54 (indoor harsh)
├── Operating temp: -40°C to +75°C
├── Vibration resistance (IEC 60068-2-6)
├── DIN rail mount หรือ pole mount option
├── PoE (Power over Ethernet) support
├── Fast roaming support (802.11r, CCKM)
└── Multiple SSID (separate OT/IT traffic)
```

### WiFi Channel Planning

```
WiFi Channel Plan (5GHz - สำหรับ industrial):

DFS Channels (5.25-5.35GHz) → ไม่แนะนำ (interference กับ radar)
Non-DFS Channels:
- 36, 40, 44, 48 (UNII-1: 5.15-5.25 GHz) ← ใช้ indoor
- 149, 153, 157, 161, 165 (UNII-3: 5.745-5.825 GHz) ← outdoor

Channel Width: 20MHz สำหรับ industrial (ไม่ 40/80MHz)
เหตุผล: หลีกเลี่ยง interference, ใช้ channel ได้มากขึ้น

Coverage Planning:
├── Cell size: 40-60m radius (indoor industrial)
├── Overlap: 15-20% (สำหรับ roaming)
├── SNR: > 20dB ที่ edge of coverage
└── Throughput: design สำหรับ worst-case device
```

---

## Step 1027: 5G สำหรับ Industrial

### Private 5G (5G NR) ใน Plant

```
5G NR Features ที่สำคัญสำหรับ Industry:

URLLC (Ultra-Reliable Low-Latency Communication):
├── Latency: < 1ms (end-to-end)
├── Reliability: 99.9999% (six nines)
└── ใช้สำหรับ: motion control, critical alarms

eMBB (Enhanced Mobile Broadband):
├── Data rate: 1-20 Gbps
└── ใช้สำหรับ: video surveillance, AR/VR maintenance

mMTC (Massive Machine-Type Communication):
├── Device density: 1 million devices/km²
└── ใช้สำหรับ: sensor networks, asset tracking

Private 5G Options:
┌───────────────────┬──────────────────────────────────────────┐
│ Option            │ Description                              │
├───────────────────┼──────────────────────────────────────────┤
│ Standalone Private│ Own spectrum (licensed), full control   │
│ 5G                │ Cost: 5-50M บาท (small plant)           │
├───────────────────┼──────────────────────────────────────────┤
│ Hybrid (PLMN)     │ Shared spectrum with public operator     │
│                   │ Dedicated slice for plant                │
├───────────────────┼──────────────────────────────────────────┤
│ Network Slice     │ Virtual private network on public 5G    │
│                   │ Less control, lower cost                 │
└───────────────────┴──────────────────────────────────────────┘

Vendors ใน Thailand:
├── Nokia (AirScale Private Wireless)
├── Ericsson (CBRS/mmWave)
├── Siemens (ร่วมกับ Nokia)
└── โดยอ้อมผ่าน: AIS, DTAC, True (network slice)
```

---

## Step 1028: Network Documentation Standards

### Network Documentation Template

```
INDUSTRIAL NETWORK DOCUMENTATION PACKAGE
==========================================

1. NETWORK DIAGRAM (Level 1 - High Level)
   ├── Erzeugen mit: Microsoft Visio, draw.io, Lucidchart
   ├── แสดง: Zones, Firewalls, Major systems
   └── Update frequency: ทุกครั้งที่มีการเปลี่ยนแปลง

2. NETWORK DIAGRAM (Level 2 - Detailed)
   ├── แสดง: Every switch, PLC, HMI, Server
   ├── Connection: port numbers, cable type
   ├── IP addresses
   └── VLAN assignments

3. IP ADDRESS PLAN (Spreadsheet)
   ┌──────────────┬────────────┬────────────────┬──────────────┐
   │ Hostname     │ IP Address │ MAC Address    │ Location     │
   ├──────────────┼────────────┼────────────────┼──────────────┤
   │ PLC-UNIT1    │192.168.30.10│ 00:1C:06:..   │ Panel Room1  │
   │ PLC-UNIT2    │192.168.30.11│ 00:1C:06:..   │ Panel Room2  │
   │ HMI-OP1      │192.168.40.10│ 00:15:5D:..   │ Control Room │
   │ SCADA-SVR1   │192.168.50.10│ E4:54:E8:..   │ Server Room  │
   └──────────────┴────────────┴────────────────┴──────────────┘

4. CABLE SCHEDULE
   ┌────────────┬────────────┬──────────┬────────┬────────────┐
   │ Cable ID   │ From       │ To       │ Type   │ Length     │
   ├────────────┼────────────┼──────────┼────────┼────────────┤
   │ ETH-001    │ SW1-P1     │ PLC1-P1  │ Cat6A  │ 15m        │
   │ ETH-002    │ SW1-P2     │ HMI1-P1  │ Cat6A  │ 45m        │
   │ FO-001     │ SW1-SFP1   │ SW2-SFP1 │ OM3    │ 150m       │
   └────────────┴────────────┴──────────┴────────┴────────────┘

5. VLAN TABLE
   ┌──────────┬────────────────┬────────────────────────────────┐
   │ VLAN ID  │ Name           │ Subnet                         │
   ├──────────┼────────────────┼────────────────────────────────┤
   │ 10       │ Management     │ 192.168.10.0/24                │
   │ 20       │ Field_Devices  │ 192.168.20.0/24                │
   │ 30       │ PLC_Control    │ 192.168.30.0/24                │
   │ 40       │ HMI_Network    │ 192.168.40.0/24                │
   │ 50       │ SCADA          │ 192.168.50.0/24                │
   └──────────┴────────────────┴────────────────────────────────┘
```

---

## Step 1029: IP Addressing Plan

### IP Addressing สำหรับ Large Plant

```
IP ADDRESSING PLAN - LARGE CHEMICAL PLANT
==========================================

Summary:
├── Main Site:    10.10.0.0/16    (65,534 hosts)
├── Sub-sites:    10.11-19.0.0/16 (reserved for expansion)

Subnetting:
┌────────────────────────────┬────────────────────────────────────┐
│ Subnet                     │ ใช้สำหรับ                         │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.0.0/24               │ Network Infrastructure            │
│                            │ (Routers, Firewalls, L3 Switches) │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.10.0/24              │ Management (switches, UPS, KVM)   │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.20.0/23              │ Field Devices Unit 1-2            │
│ 10.10.22.0/23              │ Field Devices Unit 3-4            │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.30.0/24              │ PLC/Controller Layer              │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.40.0/24              │ HMI Network                       │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.50.0/24              │ SCADA/Historian                   │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.60.0/24              │ Safety System (SIS)               │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.70.0/24              │ Industrial Wireless               │
├────────────────────────────┼────────────────────────────────────┤
│ 10.10.100.0/24             │ DMZ                               │
└────────────────────────────┴────────────────────────────────────┘

Naming Convention:
[SITE]-[TYPE]-[UNIT]-[SEQ]
ตัวอย่าง:
├── SITEA-PLC-U1-001 → 10.10.30.11  (Site A, PLC, Unit 1, First)
├── SITEA-HMI-OPS-001 → 10.10.40.11 (Site A, HMI, Ops station, 1)
└── SITEA-SW-L3-001 → 10.10.0.1    (Site A, Switch, Layer3, 1)
```

---

## Step 1030: QoS สำหรับ Industrial Network

### Quality of Service Configuration

```
QoS Priority Classes สำหรับ OT:
════════════════════════════════════

Class 7 (Highest): Safety Critical
├── ESD (Emergency Shutdown) signals
├── Fire & Gas alerts
├── SIS communication
└── DSCP: CS7 (111 000)

Class 6: Real-time Control
├── PROFINET IRT
├── EtherNet/IP (motion control)
└── DSCP: CS6 (110 000)

Class 5: Control Traffic
├── PROFINET RT
├── EtherNet/IP (normal)
├── Modbus TCP (control)
└── DSCP: CS5 (101 000)

Class 4: Alarms & Events
├── Alarm notifications
├── Process events
└── DSCP: CS4 (100 000)

Class 3: SCADA/HMI
├── HMI screen updates
├── Trend data
└── DSCP: CS3 (011 000)

Class 2: IT/Business
├── Database sync
├── Report generation
└── DSCP: CS2 (010 000)

Class 1: Best Effort
├── Email, file transfer
├── Internet traffic
└── DSCP: CS0 (000 000)
```

### QoS Configuration (Cisco IOS - Stratix Switch)

```
// Cisco IOS QoS Configuration สำหรับ Allen-Bradley Stratix
// ──────────────────────────────────────────────────────────

// Step 1: Define Class Maps
class-map match-any SAFETY_CRITICAL
  match dscp cs7
  match access-group name SAFETY_ACL

class-map match-any REALTIME_CONTROL
  match dscp cs6
  match protocol ethernetip
  match protocol profinet

class-map match-any CONTROL_TRAFFIC
  match dscp cs5
  match protocol modbus

// Step 2: Define Policy Map (Queuing)
policy-map OT_QUEUEING
  class SAFETY_CRITICAL
    priority percent 15    // Strict priority, 15% bandwidth
    
  class REALTIME_CONTROL
    bandwidth percent 30   // Minimum 30% bandwidth
    queue-limit 50 packets
    
  class CONTROL_TRAFFIC
    bandwidth percent 25
    random-detect
    
  class class-default
    bandwidth percent 30   // Remaining for best-effort

// Step 3: Apply to Interface
interface GigabitEthernet0/1
  service-policy output OT_QUEUEING
  service-policy input MARK_INCOMING
```

---

## Step 1031: Network Bandwidth Planning

### Bandwidth Analysis

```
PROFINET Bandwidth Calculation:
─────────────────────────────────

สมมติว่ามี PROFINET network:
- 50 IO-Devices
- Cycle time: 4ms
- Data per device: 64 bytes input + 64 bytes output

Calculation:
├── Frame size per device: 128 bytes (data) + 60 bytes (overhead) = 188 bytes
├── Frames per second per device: 1000ms / 4ms = 250 frames/sec
├── Bandwidth per device: 188 × 8 × 250 = 376,000 bps = ~0.4 Mbps
└── Total for 50 devices: 0.4 × 50 = 20 Mbps

Network capacity: 100Mbps (Full duplex)
Utilization: 20/100 = 20% (ยอมรับได้ - max 50% สำหรับ real-time)

เพิ่ม safety margin 50%:
Effective bandwidth needed: 30 Mbps
Recommend: 100 Mbps minimum, ควรใช้ 1 Gbps

EtherNet/IP Bandwidth (สำหรับ Rockwell):
├── Implicit messaging (I/O): คล้าย PROFINET
├── Explicit messaging (MSG instruction): ขึ้นอยู่กับ application
└── ต้องรวม: diagnostics, programming traffic ด้วย
```

---

## Step 1032: Industrial Network Troubleshooting

### Network Troubleshooting Tools

```
OT Network Troubleshooting Toolkit:
════════════════════════════════════

HARDWARE TOOLS:
├── Fluke Networks OneTouch AT → Network testing, cable verification
├── JDSU Certifier → Cable certification (Cat6A)
├── Wireshark (laptop) → Packet capture and analysis
├── ProfiTrace (PROFIBUS Int.) → PROFINET protocol analyzer
└── Managed switch port mirroring → Non-intrusive capture

SOFTWARE TOOLS:
├── Siemens SINEC NMS → Network management, topology
├── Wireshark + plugins → OT protocol dissectors
├── EtherScan (EtherScan Software) → EtherNet/IP scanner
├── Anybus Communicator → Protocol conversion diagnostics
└── nmap (network scanner) → Asset discovery (use carefully in OT)

DIAGNOSTIC COMMANDS (Siemens SCALANCE):
CLI: telnet/ssh to switch management IP

> show interfaces          // Port status, errors
> show spanning-tree       // STP/RSTP state
> show mrp                 // MRP ring status
> show vlan                // VLAN assignments
> show lldp neighbors      // PROFINET topology
> show mac address-table   // MAC/port mapping
> show ip arp              // IP/MAC mapping
> ping 192.168.30.10       // Connectivity test
```

### PROFINET Network Analysis

```
PROFINET Diagnostic Process:
─────────────────────────────

Step 1: Check Physical Layer
├── Kabel ดีไหม? ใช้ Fluke DTX
├── Port LEDs: Link OK?
├── Error counters: CRC errors, alignment errors
└── Cable length: ไม่เกิน 100m per segment (copper)

Step 2: Check PROFINET Configuration
├── TIA Portal → Online → Accessible Devices
├── Device name ตรงกันไหม?
├── IP address conflict ไหม?
└── Module configuration match ไหม?

Step 3: LLDP Topology Verification
├── TIA Portal → Accessible devices → Topology
├── เปรียบเทียบกับ expected topology
└── หา missing device หรือ unexpected device

Step 4: IO Diagnostics
├── TIA Portal → PLC → Online → Diagnostics
├── Device status: OK / Fault
├── Module status: Configured vs Actual
└── Channel diagnostics: Short circuit, wire break

Step 5: Network Traffic Analysis
├── Mirror port บน switch
├── Capture ด้วย Wireshark
├── Filter: profinet
└── ตรวจสอบ: cycle time, packet loss, duplicates
```

---

## Step 1033: Fiber Optic in Industrial Networks

### Fiber Optic Design สำหรับ OT

```
Fiber Optic Selection:
═══════════════════════

MULTIMODE FIBER (MM):
├── OM1: 62.5/125μm - Legacy, อย่าใช้ใหม่
├── OM3: 50/125μm - ดี สำหรับ 10G up to 300m
├── OM4: 50/125μm - ดีกว่า OM3, 10G up to 400m
└── OM5: 50/125μm - Wideband, future-ready

SINGLEMODE FIBER (SM):
├── OS1: Indoor, 9/125μm, สำหรับ ≤ 10km
└── OS2: Outdoor, 9/125μm, สำหรับ ≤ 60km

Industrial Usage:
┌─────────────────────────────────────────┬──────────────┐
│ Application                             │ Fiber Type   │
├─────────────────────────────────────────┼──────────────┤
│ Building to Building (< 300m)           │ OM3 or OM4   │
├─────────────────────────────────────────┼──────────────┤
│ Building to Building (300m - 2km)       │ OS1 SM       │
├─────────────────────────────────────────┼──────────────┤
│ Site to Remote substation (2-10km)      │ OS1/OS2 SM   │
├─────────────────────────────────────────┼──────────────┤
│ Outdoor (high EMI environment)          │ OS2 SM       │
├─────────────────────────────────────────┼──────────────┤
│ Cabinet to cabinet (< 100m)             │ OM4 or copper│
└─────────────────────────────────────────┴──────────────┘

Fiber Connectors:
├── LC - ใช้มากที่สุด industrial (SFP module)
├── SC - Legacy, still common
└── ST - Very legacy, avoid

Industrial Fiber Cable Types:
├── LSZH (Low Smoke Zero Halogen) → ใน building/panel
├── Armored → ใต้ดิน, outdoor harsh
└── Loose Tube → Outdoor, direct burial
```

---

## Step 1034: Network Time Synchronization (NTP/PTP)

### Time Sync สำหรับ OT

```
Time Synchronization Protocols:
════════════════════════════════

NTP (Network Time Protocol):
├── Accuracy: 1-100ms
├── ใช้สำหรับ: Servers, workstations, HMI
└── Port: 123/UDP

IEEE 1588 PTP (Precision Time Protocol):
├── Accuracy: < 1μs (hardware timestamping)
├── ใช้สำหรับ: Motion control, power systems
└── Requires: PTP-capable switches, NICs

PROFINET Time Sync (PTCP):
├── Based on IEEE 1588
├── Accuracy: < 1μs
└── ใน PROFINET IRT network

Architecture:
┌─────────────────────────────────────────────────────────────┐
│                    NTP HIERARCHY                              │
│                                                               │
│  Stratum 0: GPS Clock / Atomic Clock                         │
│       │                                                       │
│  Stratum 1: NTP Server (GPS-disciplined)                     │
│   ┌───────────────────────────────────────────┐             │
│   │  Primary NTP Server    Backup NTP Server  │ (Redundant) │
│   └───────────────────────────────────────────┘             │
│       │                         │                             │
│  ┌────▼──────┐           ┌──────▼──────┐                    │
│  │ SCADA     │           │  Historian  │  Stratum 2          │
│  │ Server    │           │  Server     │                     │
│  └────┬──────┘           └─────────────┘                    │
│       │                                                       │
│  ┌────▼──────┐                                               │
│  │  HMI      │  Stratum 3                                    │
│  │ Stations  │                                               │
│  └────┬──────┘                                               │
│       │                                                       │
│  ┌────▼──────┐                                               │
│  │   PLC     │  Stratum 4 (NTP client to HMI/server)        │
│  └───────────┘                                               │
└─────────────────────────────────────────────────────────────┘

// Siemens S7-1500 NTP Configuration (TIA Portal):
// PLC Properties → Time-of-day → NTP
NTP_CONFIG {
    SERVER_1: "192.168.50.100"    // Primary NTP server
    SERVER_2: "192.168.50.101"    // Backup NTP server
    SYNC_INTERVAL: 60             // seconds
    TIMEZONE: "Asia/Bangkok"       // UTC+7
}
```

---

## Step 1035: Industrial Network for Large Plant

### Complete Plant Network Architecture

```
╔═══════════════════════════════════════════════════════════════════╗
║           LARGE REFINERY NETWORK ARCHITECTURE EXAMPLE              ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                     ║
║  CAMPUS BACKBONE (10 Gbps Fiber Ring):                             ║
║  Data Center ─── Control Building ─── Substation ─── Admin Bldg   ║
║                          │                                         ║
║  ═══════════════ OT CORE SWITCHES ══════════════                   ║
║  (Cisco IE5000, L3, Dual power, Redundant supervisors)             ║
║                          │                                         ║
║  Distribution Switches (1 per Process Unit):                       ║
║  ┌──────────────────────────────────────────────────────────────┐ ║
║  │                    UNIT 1 - DISTILLATION                      │ ║
║  │  L2 Ring Switch ──── Cell Switch A ──── IO Devices           │ ║
║  │       │                   │                                    │ ║
║  │       │            Cell Switch B ──── IO Devices (MRP Ring)  │ ║
║  │  PLC-U1-001                                                    │ ║
║  │  (S7-1515F, SIL2)                                             │ ║
║  └──────────────────────────────────────────────────────────────┘ ║
║                                                                     ║
║  ┌──────────────────────────────────────────────────────────────┐ ║
║  │                    UNIT 2 - REFORMING                         │ ║
║  │  (Similar structure)                                          │ ║
║  └──────────────────────────────────────────────────────────────┘ ║
║                                                                     ║
║  UTILITIES (Separate VLAN, shared infrastructure):                 ║
║  ├── Power Management (Siemens SINAUT)                            ║
║  ├── Instrument Air System                                        ║
║  ├── Cooling Water                                                ║
║  └── Flare System                                                 ║
║                                                                     ║
║  SAFETY SYSTEM (Isolated Network - No routing to OT):              ║
║  ├── SIS Controller (Yokogawa ProSafe-RS)                         ║
║  ├── F&G System (MSA SUPREMA)                                     ║
║  └── ESD System                                                    ║
║  *** Physical separation from BPCS ***                             ║
╚═══════════════════════════════════════════════════════════════════╝
```

---

## Step 1036: Network Change Management

### Change Management Process

```
Network Change Management สำหรับ OT:
══════════════════════════════════════

CHANGE TYPES:
├── Emergency Change: < 4 ชั่วโมง approval (อาจต้องทำทันที)
├── Normal Change: 5 วัน advance notice
└── Standard Change: Pre-approved, low risk

CHANGE REQUEST FORM:
┌────────────────────────────────────────────────────────────┐
│ CR Number: CR-2024-0142                                     │
│ Date Requested: 2024-01-15                                  │
│ Date Planned: 2024-01-22 02:00-04:00                       │
│                                                             │
│ Requested by: Somchai Engineer                              │
│ System: OT Network - Unit 1 PLC Switch                     │
│ Priority: Normal                                            │
│                                                             │
│ DESCRIPTION:                                                │
│ Add new VLAN 25 for Unit 1 remote I/O rack expansion       │
│                                                             │
│ IMPACT ANALYSIS:                                            │
│ - Network: Low (adding VLAN, no existing traffic affected) │
│ - Production: Low (during planned maintenance window)      │
│                                                             │
│ ROLLBACK PLAN:                                              │
│ Remove VLAN 25 configuration - 5 minutes                   │
│                                                             │
│ TEST PLAN:                                                  │
│ Ping test from PLC to new remote I/O after change          │
│                                                             │
│ APPROVALS:                                                  │
│ OT Engineer: Somchai ✓    Production Mgr: Wanchai ✓        │
│ IT/OT Security: Nattida ✓  Shift Supervisor: Active ✓     │
└────────────────────────────────────────────────────────────┘
```

---

## Step 1037: Network Monitoring and Alerting

### SNMP Monitoring Setup

```
// SNMP Configuration (Siemens SCALANCE)
// ─────────────────────────────────────────

// Basic SNMP v3 (Recommended - encrypted)
SNMP_CONFIG {
    VERSION: V3
    USER: "OT_Monitor"
    AUTH_PROTOCOL: SHA256
    AUTH_PASSWORD: "Auth@Monitor2024"
    PRIV_PROTOCOL: AES256
    PRIV_PASSWORD: "Priv@Monitor2024"
    
    // Traps configuration
    TRAP_DESTINATION: "192.168.50.200"    // NMS Server
    TRAP_EVENTS: [
        "linkDown",
        "linkUp",
        "authenticationFailure",
        "coldStart",
        "warmStart"
    ]
}

// Key OIDs to Monitor:
// ─────────────────────
// ifOperStatus: 1.3.6.1.2.1.2.2.1.8  → Port up/down
// ifInErrors:   1.3.6.1.2.1.2.2.1.14 → Input errors
// ifOutErrors:  1.3.6.1.2.1.2.2.1.20 → Output errors
// sysUpTime:    1.3.6.1.2.1.1.3.0    → System uptime

// Alert Thresholds:
// ─────────────────
// Link Down: CRITICAL (immediate alert)
// Error rate > 0.1%: WARNING
// CPU > 80%: WARNING
// Temperature > 70°C: CRITICAL
// Fan failure: CRITICAL
```

---

## Step 1038: Network Capacity Planning

### Capacity Planning Methodology

```
Network Capacity Planning Process:
════════════════════════════════════

STEP 1: INVENTORY CURRENT ASSETS
├── ทุก device ที่ต่อ network
├── Bandwidth utilization ปัจจุบัน (SNMP polling)
└── Peak vs average utilization

STEP 2: FORECAST GROWTH
├── New devices ที่จะเพิ่มใน 3-5 ปี
├── New applications (historian, video, IoT)
└── New sites หรือ units

STEP 3: BANDWIDTH ANALYSIS
Formula:
Required Bandwidth = (Current peak) × (Growth factor) × (Safety factor)
                   = Peak × 1.3 × 1.5 (50% headroom)

Example:
├── Current peak: 40 Mbps
├── Growth: 30% in 3 years
├── Safety: 50% headroom
└── Required: 40 × 1.3 × 1.5 = 78 Mbps
   → ใช้ 100 Mbps uplink, plan upgrade to 1G

STEP 4: EQUIPMENT LIFECYCLE
├── Switch End-of-Life (EOL) schedule
├── จัดซื้อ spares
└── Migration plan (ไม่ downgrade performance)

STEP 5: DOCUMENTATION UPDATE
├── Update network diagram
├── Update IP address plan
└── Update capacity baseline
```

---

## Step 1039: OT/IT Convergence Best Practices

### Managing OT/IT Convergence

```
OT/IT Convergence Challenges and Solutions:
═════════════════════════════════════════════

CHALLENGE 1: Conflicting Priorities
IT: Security, Flexibility
OT: Availability, Real-time
Solution: Clear RACI matrix, Joint governance committee

CHALLENGE 2: Skills Gap
IT: ไม่เข้าใจ process, protocols
OT: ไม่เข้าใจ IT security, network
Solution: Cross-training program, shared team

CHALLENGE 3: Technology Gap
OT: Legacy systems, proprietary protocols
IT: Modern standards, cloud-based
Solution: Protocol gateways, gradual migration

CHALLENGE 4: Change Management
OT: Very conservative (uptime critical)
IT: Agile, frequent changes
Solution: Separate change windows, OT veto right

BEST PRACTICES:
┌──────────────────────────────────────────────────────────────┐
│ 1. Joint OT/IT team with clear roles                        │
│ 2. Shared CMDB (Configuration Management Database)          │
│ 3. Unified monitoring (SIEM) with OT context               │
│ 4. Clear network boundary ownership                         │
│ 5. Regular joint meeting (monthly minimum)                  │
│ 6. Common standards for shared infrastructure               │
│ 7. Separate IP ranges/VLANs for OT                         │
│ 8. Emergency procedures agreed in advance                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1040: Lab - Complete Network Design Project

### Lab: Network Design สำหรับ Automotive Body Stamping Plant

**โจทย์**: โรงงานประกอบตัวถังรถยนต์ ประกอบด้วย:
- 3 Press Lines (ไฮดรอลิก, แต่ละสายมี 4 press stations)
- Welding Area (12 robotic welding cells)
- Transfer Conveyor System
- Quality Inspection Station (vision system + measurement)
- Maintenance Area

**Requirements**:
- Production target: 300 units/day
- PROFINET IRT สำหรับ press lines (fast cycle time)
- EtherNet/IP สำหรับ robotic welding
- Video surveillance 40 cameras
- Maintenance tablet wireless access
- OPC UA สำหรับ historian

```
LAB DELIVERABLES:
══════════════════

Part 1: Network Architecture Design (35%)
├── Physical topology diagram
├── Logical (VLAN) diagram
├── IP addressing plan
└── Equipment list with specifications

Part 2: Switch Configuration (35%)
├── VLAN configuration (minimum 5 VLANs)
├── MRP ring configuration
├── QoS policy (minimum 4 classes)
├── Port security configuration
└── SNMP monitoring configuration

Part 3: Bandwidth and Capacity Analysis (15%)
├── PROFINET bandwidth calculation
├── Video bandwidth (40 cameras × 4Mbps HD)
├── Total bandwidth requirement
└── Recommend infrastructure (switch capacity)

Part 4: Documentation (15%)
├── Network diagram (draw.io หรือ Visio)
├── Configuration document
└── Maintenance guide

GRADING CRITERIA:
├── Completeness: ครบทุก requirement
├── Technical accuracy: ถูกต้องตามมาตรฐาน
├── Practical feasibility: ทำได้จริง
├── Documentation quality: อ่านเข้าใจง่าย
└── Cost consideration: สมเหตุสมผล

TOTAL: 100 คะแนน  ผ่าน: 70 คะแนน
```

---

## สรุป Part 52

| หัวข้อ | Key Points |
|--------|-----------|
| Switch Selection | Managed switch สำหรับ OT ทุก level |
| VLAN Design | แยกตาม Purdue Model + Safety isolated |
| Redundancy | MRP < 200ms สำหรับ PROFINET |
| QoS | Safety → Real-time → Control → HMI → IT |
| Documentation | IP plan + Cable schedule + Diagram |

---

## Navigation

- [← Part 51: Cybersecurity สำหรับ Automation](./part51_cybersecurity.md)
- [→ Part 53: Advanced PID and Process Control](./part53_advanced_pid.md)
- [↑ สารบัญหลัก](../README.md)
