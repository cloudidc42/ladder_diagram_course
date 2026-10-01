# Part 50: OPC UA (Steps 981-1000)

## ภาพรวม
OPC UA (OPC Unified Architecture) คือมาตรฐานการสื่อสารอุตสาหกรรมที่ครอบคลุมและปลอดภัยที่สุด รองรับทั้ง machine-to-machine และ machine-to-cloud การเข้าใจ OPC UA เป็นทักษะสำคัญสำหรับ Industrial Digital Transformation

---

## Step 981: OPC UA vs OPC Classic

### เปรียบเทียบ OPC UA กับ OPC Classic

```
OPC Classic vs OPC UA Comparison:
┌─────────────────────────────────────────────────────────────┐
│  OPC CLASSIC (DA, HDA, A&E):                               │
│  ├── พัฒนาปี 1996 บน Windows COM/DCOM                     │
│  ├── Windows เท่านั้น                                       │
│  ├── ปัญหา firewall (DCOM พอร์ตสุ่ม)                       │
│  ├── Security: น้อยมาก                                      │
│  └── แยก spec: DA, HDA, A&E ไม่ compatible กัน            │
│                                                             │
│  OPC UA:                                                   │
│  ├── พัฒนาปี 2006, ปรับปรุงต่อเนื่อง                       │
│  ├── Platform independent (Linux, Windows, RTOS)           │
│  ├── เลือกพอร์ตได้ (default: 4840)                         │
│  ├── Security: Authentication + Encryption + Signing       │
│  └── Unified model: DA+HDA+A&E ในมาตรฐานเดียว            │
│                                                             │
│  Migration:                                                │
│  OPC Classic → OPC UA Wrapper → OPC UA Client             │
│  (เพื่อ bridge legacy OPC DA สู่ OPC UA)                  │
└─────────────────────────────────────────────────────────────┘
```

### OPC UA Key Features

| Feature | OPC Classic | OPC UA |
|---------|------------|--------|
| Platform | Windows only | Cross-platform |
| Transport | COM/DCOM | TCP, HTTPS, WebSocket |
| Security | Poor | TLS, certificates |
| Information Model | Flat | Hierarchical, typed |
| Companion Specs | None | Many (ISA-95, AutoID, etc.) |
| Pub/Sub | No | Yes (v1.04+) |

---

## Step 982: Information Model

### OPC UA Information Model

```
OPC UA Information Model:
┌─────────────────────────────────────────────────────────────┐
│  Root                                                       │
│  ├── Objects                                               │
│  │   ├── Server (built-in)                                │
│  │   └── [User objects]                                   │
│  │       └── Machine1                                     │
│  │           ├── Temperature (Variable)                   │
│  │           ├── Status (Variable)                       │
│  │           └── StartMachine (Method)                   │
│  │                                                         │
│  ├── Types                                                │
│  │   ├── ObjectTypes                                      │
│  │   │   └── MachineType (custom type)                   │
│  │   ├── VariableTypes                                   │
│  │   │   └── AnalogItemType (DA value + EU range)        │
│  │   └── DataTypes                                       │
│  │       └── BaseDataType → Double, Int32, etc.          │
│  │                                                         │
│  ├── Views                                                │
│  │   └── [Custom views for specific use cases]           │
│  │                                                         │
│  └── Methods                                              │
│      └── [Server methods]                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 983: Node Types

### ประเภทของ Nodes ใน OPC UA

```
OPC UA Node Classes:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Object Node:                                               │
│  └── เป็นตัวแทน real-world object                          │
│      ตัวอย่าง: Machine1, Reactor101, Conveyor_A            │
│                                                             │
│  Variable Node:                                            │
│  └── เก็บข้อมูล (Value)                                    │
│      ตัวอย่าง: Temperature (75.3°C), Status (Running)      │
│                                                             │
│  Method Node:                                              │
│  └── เรียกใช้งานได้ (Function call)                        │
│      ตัวอย่าง: Start(), Stop(), SetSpeed(100.0)           │
│                                                             │
│  View Node:                                                │
│  └── Subset ของ address space                             │
│                                                             │
│  DataType Node:                                            │
│  └── Definition ของ data type                             │
│      ตัวอย่าง: Double, String, MyCustomStruct             │
│                                                             │
│  ReferenceType Node:                                       │
│  └── ประเภทของ relationship ระหว่าง nodes                 │
│      ตัวอย่าง: HasComponent, HasProperty, Organizes       │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 984: Address Space

### OPC UA Address Space

```
Address Space Concepts:
┌─────────────────────────────────────────────────────────────┐
│  Node ID:                                                   │
│  ├── Format: ns=<namespace>;s=<string>                     │
│  │   Example: ns=2;s=Machine1.Temperature                 │
│  ├── Or: ns=<namespace>;i=<integer>                        │
│  │   Example: ns=0;i=2256 (Server.ServerStatus)           │
│  └── Namespace 0: OPC UA standard nodes                   │
│      Namespace 1: Your server's namespace                  │
│                                                             │
│  Browse Name:                                              │
│  └── Human-readable name for browsing                     │
│      Example: {ns=2}Temperature                           │
│                                                             │
│  Node Attributes:                                          │
│  ├── NodeId (unique identifier)                           │
│  ├── NodeClass (Object, Variable, Method...)              │
│  ├── BrowseName (for browsing)                           │
│  ├── DisplayName (for display)                           │
│  ├── Description (documentation)                         │
│  └── Value (for Variable nodes)                          │
│                                                             │
│  Variable Attributes:                                      │
│  ├── Value (current data value)                          │
│  ├── DataType (Double, Int32, String...)                 │
│  ├── AccessLevel (Read/Write/ReadWrite)                  │
│  └── Historizing (ถ้า TRUE → เก็บ history)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 985: Security

### OPC UA Security Model

```
OPC UA Security Layers:
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Transport Security                               │
│  ├── None (สำหรับ development/testing เท่านั้น!)           │
│  ├── Basic128Rsa15 (deprecated, weak)                     │
│  ├── Basic256 (deprecated)                               │
│  └── Basic256Sha256 (recommended)                        │
│      └── AES-256 encryption + RSA-2048 signing           │
│                                                             │
│  Layer 2: Authentication                                   │
│  ├── Anonymous (no auth)                                 │
│  ├── Username/Password                                   │
│  └── X.509 Certificate (most secure)                    │
│                                                             │
│  Layer 3: Authorization (User Access)                      │
│  ├── Read-only users                                     │
│  ├── Read-write users                                    │
│  └── Admin users (method calls)                          │
└─────────────────────────────────────────────────────────────┘
```

### Certificate Management

```
OPC UA Certificate Setup:
┌─────────────────────────────────────────────────────────────┐
│  Server Certificate:                                        │
│  ├── Self-signed (สำหรับ testing)                          │
│  └── CA-signed (สำหรับ production)                        │
│                                                             │
│  Certificate Trust:                                        │
│  ├── Client ต้อง trust server certificate                │
│  └── Server ต้อง trust client certificate               │
│                                                             │
│  PKI (Public Key Infrastructure):                          │
│  ├── Create CA (Certificate Authority)                   │
│  ├── Issue server cert signed by CA                      │
│  ├── Issue client cert signed by CA                      │
│  └── Distribute CA cert to all clients as trusted       │
│                                                             │
│  Certificate Folders (OPC UA standard locations):         │
│  ├── /pki/trusted/certs/ (trusted server certs)          │
│  ├── /pki/own/ (this server's certificate)               │
│  ├── /pki/rejected/ (rejected certificates)              │
│  └── /pki/issuers/ (CA certificates)                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 986: Siemens S7-1500 OPC UA Server

### การกำหนดค่า OPC UA ใน TIA Portal

```
S7-1500 OPC UA Server Setup in TIA Portal:

Step 1: Enable OPC UA Server
  CPU Properties → General → OPC UA → Server
  ├── Activate OPC UA server: ✓ Enable
  ├── Port: 4840 (default)
  ├── Session Timeout: 30 minutes
  └── Subscription lifetime: 60 minutes

Step 2: Security Settings
  CPU Properties → OPC UA → Server → Security
  ├── Allow anonymous access: No (production)
  ├── Security policy: Basic256Sha256
  └── Username/Password: Add users

Step 3: Expose Tags
  Option 1: Individual tags
  ├── Right-click DB variable → "Accessible from HMI/OPC UA"
  └── Set Access: Read, Write, or ReadWrite

  Option 2: OPC UA DB
  ├── Create new DB
  ├── DB Properties → Accessible from HMI/OPC UA
  └── All variables in DB will be exposed

Step 4: Node ID Configuration
  Default: ns=3;s="DB_Name"."Variable_Name"
  Custom: Assign custom node IDs if needed
```

### TIA Portal - OPC UA Server Configuration Code

```pascal
// Siemens S7-1500: OPC UA Server Example
// ไม่มี code พิเศษ - กำหนดใน HW Config เท่านั้น
// แต่สามารถ monitor client connections ด้วย:

// Check OPC UA Server Status
OPC_UA_Server_Status : OPC_UA_ServerInformation;

// Call เพื่อดู connected clients
CALL "OPC_UA_ServerInformation"
    ServerInterface := W#16#100,   // OPC UA interface HW ID
    Activate        := TRUE
    DONE            => Status_Done
    ERROR           => Status_Error
    STATUS          => OPC_UA_Server_Status

// อ่านจำนวน clients ที่เชื่อมต่อ
Active_Sessions := OPC_UA_Server_Status.ActiveSessionCount;
```

---

## Step 987: Allen-Bradley OPC UA

### Rockwell OPC UA Server

```
ControlLogix OPC UA Configuration:
┌─────────────────────────────────────────────────────────────┐
│  Studio 5000 OPC UA:                                        │
│  ├── ไม่มี OPC UA server built-in ใน ControlLogix         │
│  ├── ใช้ FactoryTalk Linx Gateway เป็น OPC UA server      │
│  │   └── FactoryTalk Linx → ControlLogix tags → OPC UA   │
│  └── หรือใช้ third-party OPC UA server:                   │
│      ├── KEPServerEX (Kepware)                            │
│      ├── Matrikon OPC UA                                  │
│      └── Ignition (Inductive Automation)                  │
│                                                             │
│  KEPServerEX Configuration:                               │
│  1. Add Channel: EtherNet/IP                              │
│  2. Add Device: Logix Driver (ControlLogix/CompactLogix)  │
│  3. Browse Tags from PLC                                  │
│  4. Export as OPC UA nodes                               │
│  5. Configure OPC UA Security                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 988: Client Tools (UaExpert)

### UaExpert - OPC UA Client

```
UaExpert by Unified Automation (Free):
┌─────────────────────────────────────────────────────────────┐
│  การใช้งาน UaExpert:                                        │
│                                                             │
│  1. Add Server Connection                                   │
│     ├── URL: opc.tcp://192.168.1.10:4840               │
│     ├── Security: None หรือ Basic256Sha256              │
│     └── Authentication: Anonymous หรือ user/pass       │
│                                                             │
│  2. Browse Address Space                                   │
│     ├── เห็น folder structure ของ server                │
│     ├── Expand Objects → Machine → Temperature          │
│     └── Double-click: เพิ่มเข้า Data Access View        │
│                                                             │
│  3. Data Access View                                       │
│     ├── เห็น real-time values                           │
│     ├── Source timestamp                                │
│     └── Status code (Good/Bad/Uncertain)               │
│                                                             │
│  4. Write Value                                            │
│     └── Right-click → Write → Enter new value          │
│                                                             │
│  5. Subscribe to Changes                                   │
│     ├── Add items to subscription                       │
│     └── กำหนด Publishing interval (100ms)              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 989: OPC UA PubSub

### OPC UA Publish-Subscribe (v1.04)

```
OPC UA PubSub vs Client-Server:
┌─────────────────────────────────────────────────────────────┐
│  CLIENT-SERVER (Request-Response):                          │
│  Client ──Request──► Server ──Response──► Client           │
│  ├── Polling: Client ถามทุก N ms                          │
│  ├── Subscription: Server push เมื่อ value เปลี่ยน        │
│  └── ต้องมี connection ตลอดเวลา                           │
│                                                             │
│  PUBSUB (Publish-Subscribe):                               │
│  Publisher ──Publish──► Message Broker ──► Subscriber      │
│  ├── Publisher ส่งข้อมูลโดยไม่รู้จัก subscriber          │
│  ├── Subscriber รับข้อมูลโดยไม่ต้องมี direct connection │
│  └── ผ่าน Broker: MQTT หรือ UDP multicast               │
│                                                             │
│  OPC UA PubSub Use Cases:                                  │
│  ├── Many clients monitoring same data                   │
│  ├── Low-bandwidth scenarios                            │
│  ├── Internet/cloud connectivity                        │
│  └── Field-level communication (IEC 60802)              │
└─────────────────────────────────────────────────────────────┘
```

### OPC UA PubSub over MQTT

```
OPC UA PubSub + MQTT Transport:
┌─────────────────────────────────────────────────────────────┐
│  PLC (OPC UA Publisher)                                     │
│  ├── Publishes to MQTT Broker                             │
│  └── Topic: opcua/pub/<publisherID>/<dataSetName>        │
│                                                             │
│  Payload Formats:                                          │
│  ├── JSON (human readable)                               │
│  └── UADP (binary, more efficient)                      │
│                                                             │
│  JSON Payload Example:                                     │
│  {                                                         │
│    "MessageId": "abc-123",                                 │
│    "PublisherId": "PLC-001",                              │
│    "Messages": [{                                          │
│      "DataSetWriterId": 1,                                │
│      "Timestamp": "2024-01-15T10:30:00Z",                │
│      "Payload": {                                         │
│        "Temperature": {                                   │
│          "Type": 10,                                     │
│          "Body": 75.3,                                   │
│          "StatusCode": 0,                                │
│          "SourceTimestamp": "2024-01-15T10:30:00Z"       │
│        }                                                  │
│      }                                                    │
│    }]                                                      │
│  }                                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 990-1000: Complete OPC UA Integration

### โครงการ: S7-1500 OPC UA Server + SCADA + Cloud

#### 1. Architecture

```
Complete OPC UA Integration:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  S7-1515 PLC                                               │
│  ├── Process Control (real-time)                          │
│  └── OPC UA Server (port 4840)                           │
│           │                                               │
│           │ OPC UA Client-Server                         │
│           ▼                                               │
│  SCADA (Ignition)           Historian Server              │
│  ├── OPC UA Client          ├── OPC UA Client            │
│  ├── Real-time display      ├── Store historical data    │
│  └── Alarm management       └── Time-series DB           │
│                                                             │
│           │                                               │
│           │ OPC UA → MQTT (Sparkplug B)                  │
│           ▼                                               │
│  MQTT Broker (HiveMQ)                                     │
│           │                                               │
│           │ MQTT                                         │
│           ▼                                               │
│  Cloud (AWS IoT Core / Azure)                             │
│  ├── Time-series storage                                 │
│  ├── Analytics (Grafana/Power BI)                       │
│  └── ML predictions                                     │
└─────────────────────────────────────────────────────────────┘
```

#### 2. S7-1500 OPC UA Address Space Design

```
OPC UA Namespace Design for S7-1500:
Namespace URI: urn:PlantA:Assembly:PLC001

Root/Objects/
└── PlantA/                         (Object)
    └── Assembly/                   (Object)
        └── Line_A/                 (Object: MachineType)
            ├── Status/             (Object)
            │   ├── Running         (Variable: Boolean, RO)
            │   ├── AlarmActive     (Variable: Boolean, RO)
            │   └── Mode            (Variable: Int32, RO)
            │                       0=Manual, 1=Auto, 2=Service
            ├── Process/            (Object)
            │   ├── Temperature     (Variable: Double, RO, EU: degC)
            │   ├── Pressure        (Variable: Double, RO, EU: bar)
            │   ├── Flow            (Variable: Double, RO, EU: m3/h)
            │   └── Level           (Variable: Double, RO, EU: %)
            ├── Production/         (Object)
            │   ├── PartCount       (Variable: Int32, RO)
            │   ├── Rejects         (Variable: Int32, RO)
            │   └── ShiftOEE        (Variable: Double, RO)
            ├── Recipe/             (Object)
            │   ├── RecipeNumber    (Variable: Int32, RW)
            │   ├── Temperature_SP  (Variable: Double, RW)
            │   └── Speed_SP        (Variable: Double, RW)
            └── Commands/           (Object)
                ├── Start           (Method: no args)
                ├── Stop            (Method: no args)
                └── SetSpeed        (Method: arg: Double)
```

#### 3. Complete OPC UA Data Block (TIA Portal)

```pascal
// OPC UA Data Block - Exposed to OPC UA clients
// DB: OPC_UA_Data
// Properties: Accessible from HMI/OPC UA = TRUE

DATA_BLOCK "OPC_UA_Data"
{ S7_Optimized_Access := 'FALSE' }
VERSION : 0.1
NON_RETAIN
VAR

    // Status
    Status : STRUCT
        Running       : BOOL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'}
        AlarmActive   : BOOL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'}
        Mode          : INT;     {S7_Visible := 'True'; S7_ReadOnly := 'True'}
    END_STRUCT;
    
    // Process Variables
    Process : STRUCT
        Temperature   : REAL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'; S7_Unit := '°C'}
        Pressure      : REAL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'; S7_Unit := 'bar'}
        Flow          : REAL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'; S7_Unit := 'm³/h'}
        Level         : REAL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'; S7_Unit := '%'}
    END_STRUCT;
    
    // Production KPIs
    Production : STRUCT
        PartCount     : DINT;    {S7_Visible := 'True'; S7_ReadOnly := 'True'}
        Rejects       : DINT;    {S7_Visible := 'True'; S7_ReadOnly := 'True'}
        ShiftOEE      : REAL;    {S7_Visible := 'True'; S7_ReadOnly := 'True'; S7_Unit := '%'}
    END_STRUCT;
    
    // Recipe (Read-Write for authorized clients)
    Recipe : STRUCT
        RecipeNumber  : INT;     {S7_Visible := 'True'}
        Temperature_SP: REAL;    {S7_Visible := 'True'; S7_Unit := '°C'}
        Speed_SP      : REAL;    {S7_Visible := 'True'; S7_Unit := 'rpm'}
    END_STRUCT;
    
END_VAR
BEGIN
END_DATA_BLOCK
```

#### 4. Ignition SCADA OPC UA Configuration

```
Ignition SCADA - OPC UA Connection:

1. Add OPC UA Connection
   ├── Name: S7-1500 PLC001
   ├── Endpoint URL: opc.tcp://192.168.1.10:4840
   ├── Security Policy: Basic256Sha256
   └── User Token: Username/Password (opcua_user / SecurePass123!)

2. Create Tag Provider
   ├── Type: OPC
   └── OPC Server: S7-1500 PLC001

3. Browse and Add Tags
   ├── Browse: ns=3;s="OPC_UA_Data".Process.Temperature
   └── Tag name: Temperature_TIC101
   └── Unit: degC (from OPC UA Engineering Unit)

4. Historian
   ├── Tags → History → Enable
   └── Sample: 1 second

5. Alarm Configuration
   ├── High: 80°C
   └── High-High: 90°C → SMS/Email notification
```

#### 5. OPC UA Client Program (C# example)

```csharp
// C# OPC UA Client using OPC Foundation SDK
// เชื่อมต่อกับ S7-1500 OPC UA Server

using Opc.Ua;
using Opc.Ua.Client;

class OPC_UA_Client
{
    static Session session;
    
    static async Task ConnectAndRead()
    {
        // Create application configuration
        var config = new ApplicationConfiguration()
        {
            ApplicationName = "SCADA Client",
            ApplicationType = ApplicationType.Client,
            SecurityConfiguration = new SecurityConfiguration
            {
                ApplicationCertificate = new CertificateIdentifier(),
                AutoAcceptUntrustedCertificates = false  // ห้ามใช้ true ใน production!
            }
        };
        
        // Connect to server
        var endpoint = CoreClientUtils.SelectEndpoint(
            "opc.tcp://192.168.1.10:4840", 
            useSecurity: true);
        
        session = await Session.Create(
            config, 
            new ConfiguredEndpoint(null, endpoint),
            false, "Client Session", 60000, 
            new UserIdentity("opcua_user", "SecurePass123!"), null);
        
        Console.WriteLine("Connected to OPC UA Server!");
        
        // Read temperature
        var nodeId = new NodeId("\"OPC_UA_Data\".Process.Temperature", 3);
        var dataValue = session.ReadValue(nodeId);
        
        Console.WriteLine($"Temperature: {(double)dataValue.Value:F2} °C");
        Console.WriteLine($"Quality: {dataValue.StatusCode}");
        Console.WriteLine($"Timestamp: {dataValue.SourceTimestamp}");
        
        // Write Recipe (if authorized)
        var writeNodeId = new NodeId("\"OPC_UA_Data\".Recipe.Temperature_SP", 3);
        session.WriteValue(writeNodeId, new DataValue(new Variant(78.0)));
        Console.WriteLine("Recipe SP written successfully");
        
        // Subscribe to changes (MonitoredItem)
        var subscription = new Subscription(session.DefaultSubscription)
        {
            PublishingInterval = 1000  // 1 second
        };
        
        var monItem = new MonitoredItem(subscription.DefaultItem)
        {
            StartNodeId = nodeId,
            SamplingInterval = 1000
        };
        monItem.Notification += (item, e) =>
        {
            foreach (var value in item.DequeueValues())
            {
                Console.WriteLine($"New temp: {value.Value:F2}°C at {value.SourceTimestamp}");
            }
        };
        
        subscription.AddItem(monItem);
        session.AddSubscription(subscription);
        subscription.Create();
        
        Console.ReadLine();
        session.Close();
    }
}
```

#### 6. Grafana Dashboard via OPC UA

```
Grafana + OPC UA Integration:

Option 1: Grafana + InfluxDB
  OPC UA → Python Script → InfluxDB → Grafana
  (ใช้ opcua Python library + InfluxDB write API)

Option 2: Grafana + Ignition Historian  
  OPC UA → Ignition Historian → Grafana JDBC Plugin

Option 3: Grafana OPC UA Plugin (community)
  Install: grafana-opcua-datasource
  Configure: Direct OPC UA connection
  ── ไม่แนะนำสำหรับ production (plugin ยังไม่ stable)

Best Practice:
  OPC UA → Historian (time-series DB) → Grafana
  └── TimescaleDB, InfluxDB, OSIsoft PI, Aveva Historian
```

---

## Lab Project 50: Full OPC UA Integration Project

### โจทย์
สร้าง Complete OPC UA Integration สำหรับ Smart Factory

### Architecture ที่ต้องสร้าง
```
3x S7-1500 PLCs → OPC UA → KEPServerEX (Aggregator Server)
                              │
                              ├── SCADA (WinCC OA / Ignition)
                              ├── MES (SAP MII)
                              └── Cloud Bridge (MQTT Sparkplug B)
                                      │
                              Azure IoT Hub
                                      │
                              Azure Time Series Insights
                                      │
                              Power BI Dashboard
```

### งานที่ต้องทำ
1. ออกแบบ OPC UA Information Model สำหรับ Smart Factory
2. Configure S7-1500 OPC UA Server (TIA Portal)
3. Setup KEPServerEX เป็น Aggregator
4. Configure OPC UA Security (Certificates + Authentication)
5. Implement Sparkplug B MQTT Bridge
6. สร้าง Azure Time Series Insights integration
7. ทำ Power BI Real-time Dashboard
8. Penetration Testing สำหรับ OPC UA Security

---

## Checkpoint: มาถึง Step 1000 แล้ว!

```
Course Completion Summary (Steps 1-1000):

Step 1-200:    PLC Fundamentals (Ladder basics, data types)
Step 201-400:  Intermediate (Timers, counters, math, analog)
Step 401-600:  Advanced (Communication, SCADA, HMI)
Step 601-800:  Expert (Motion, drives, networks, DCS)
Step 801-1000: Master Level:
               ├── Safety (SIL, Functional Safety)
               ├── Redundancy (H-System)
               ├── Batch Control (ISA-88)
               ├── Process Control (Cascade, Smith)
               ├── Advanced Motion (Multi-axis, cam)
               ├── Robotics Integration
               ├── Vision Systems
               ├── IIoT / Industry 4.0
               └── OPC UA
```

---

## สรุป Part 50 และ Series Summary

### OPC UA Quick Reference

| Feature | Value |
|---------|-------|
| Default Port | 4840 (TCP), 443 (HTTPS) |
| Security | TLS 1.2+, X.509 certificates |
| Min. Security Mode | Basic256Sha256 (production) |
| Namespace Index | 0=OPC UA, 1+=Custom |
| Node ID Format | ns=X;s="string" or ns=X;i=integer |
| PubSub Transport | MQTT (JSON/UADP) or AMQP or UDP |

### Key Takeaways จาก Course ทั้งหมด

```
จาก Steps 1-1000:

1. Fundamentals มาก่อนเสมอ
   └── Understanding bits, bytes, data types ก่อน PID

2. Safety ไม่ใช่ option - เป็น requirement
   └── SIL, PL assessment ก่อน design

3. Redundancy = Availability
   └── ออกแบบสำหรับ MTTR ไม่ใช่แค่ MTBF

4. Standards ช่วยประหยัดเวลา
   └── ISA-88 for batch, ISA-95 for enterprise

5. Communication คือกุญแจ
   └── PROFINET, EtherNet/IP, OPC UA, MQTT

6. Security ต้องคิดตั้งแต่ต้น
   └── IEC 62443, OPC UA Security

7. Data คือทรัพย์สิน
   └── Historian → Analytics → AI/ML → Insight

8. ไม่มี "perfect" solution
   └── Trade-offs: Performance vs. Safety vs. Cost
```

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 49: IoT & Industry 4.0](part49_iot_industry40.md) |
| ► ถัดไป | [Part 51: Advanced SCADA Design](part51_advanced_scada.md) |
| 🏠 หน้าหลัก | [README](../README.md) |

---

*จบ Part 50 - Steps 981-1000 - Complete OPC UA Integration*

*ยินดีด้วย! คุณได้ผ่าน 1,000 Steps แรกของ Ladder Diagram PLC Programming Course แล้ว!*
