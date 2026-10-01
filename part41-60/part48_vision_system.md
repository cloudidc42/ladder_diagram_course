# Part 48: Vision System Integration (Steps 941-960)

## ภาพรวม
Machine Vision เป็นเทคโนโลยีที่ให้ระบบอัตโนมัติ "มองเห็น" และตัดสินใจได้โดยอิงจากภาพ การ integrate vision system กับ PLC เป็นทักษะสำคัญสำหรับ Quality Control และ Guidance applications

---

## Step 941: Machine Vision Basics

### Camera Types

```
Camera Types for Machine Vision:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  AREA SCAN CAMERA:                                          │
│  ├── Captures 2D image (rectangular area)                  │
│  ├── ความละเอียด: 0.3 - 20 Megapixels                     │
│  ├── ใช้สำหรับ: Part inspection, barcode, OCR, gauging     │
│  └── Examples: Cognex In-Sight 7000, KEYENCE IV-H series  │
│                                                             │
│  LINE SCAN CAMERA:                                         │
│  ├── Captures 1 line of pixels at a time                  │
│  ├── Part moves past camera (web inspection)              │
│  ├── ความละเอียด: 1024 - 8192 pixels per line            │
│  ├── ใช้สำหรับ: Continuous web (fabric, paper, film)      │
│  └── Examples: Basler raL4096, Teledyne Dalsa             │
│                                                             │
│  3D CAMERA:                                                │
│  ├── Laser profiler, structured light, stereo vision      │
│  ├── Output: Depth map or point cloud                     │
│  ├── ใช้สำหรับ: Height measurement, volume, bin picking   │
│  └── Examples: Cognex 3D-A1000, KEYENCE LJ-X8000         │
│                                                             │
│  SMART CAMERA (All-in-one):                                │
│  ├── Camera + Processor + I/O in one unit                 │
│  ├── No separate vision processor needed                  │
│  ├── ใช้สำหรับ: Simple to moderate inspection            │
│  └── Examples: Cognex In-Sight, KEYENCE IV-H             │
└─────────────────────────────────────────────────────────────┘
```

### Lighting Types

```
Lighting for Machine Vision:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  BRIGHT FIELD (Front lighting):                            │
│  Camera ──┐                                                 │
│  Light ───┼──► Part ──► Reflects light back to camera     │
│           └── For: Surface defects, print inspection       │
│                                                             │
│  DARK FIELD (Low angle):                                   │
│  Light ─(low angle)──► Part                               │
│  Camera (above) ← Only edges and texture scatter light    │
│  └── For: Scratches, surface texture, embossed text       │
│                                                             │
│  BACK LIGHT (Silhouette):                                  │
│  Light (below) → Part → Camera (above)                    │
│  └── For: Dimensional measurement, hole detection         │
│                                                             │
│  COAXIAL (Diffuse front):                                  │
│  Light through beam splitter → Part → reflects back       │
│  └── For: Highly reflective surfaces, CDs, wafers         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 942: Image Processing Concepts

### Vision Tool Types

```
Common Vision Tools:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  PATTERN MATCHING (Locating):                               │
│  ├── Find template image in larger image                   │
│  ├── Returns: position (X, Y, angle)                      │
│  └── ใช้สำหรับ: Robot guidance, inspection alignment     │
│                                                             │
│  BLOB ANALYSIS:                                            │
│  ├── Find connected regions of similar color/brightness   │
│  ├── Returns: Area, perimeter, centroid, dimensions       │
│  └── ใช้สำหรับ: Counting, size check, presence detect    │
│                                                             │
│  EDGE DETECTION:                                           │
│  ├── Find boundaries between light and dark               │
│  ├── Returns: Position of edge                            │
│  └── ใช้สำหรับ: Dimensional measurement, gap check       │
│                                                             │
│  OCR/OCV (Text Reading):                                   │
│  ├── Read characters (alphanumeric)                       │
│  ├── Returns: Text string                                 │
│  └── ใช้สำหรับ: Date code, lot number verification       │
│                                                             │
│  BARCODE/QR READING:                                       │
│  ├── Decode 1D/2D codes                                   │
│  ├── Returns: Decoded string                              │
│  └── ใช้สำหรับ: Traceability, item identification        │
│                                                             │
│  CALIPER / GAUGING:                                        │
│  ├── Measure dimensions precisely                         │
│  ├── Returns: Dimension in mm/pixels                      │
│  └── ใช้สำหรับ: Tolerance checking, SPC                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 943: Pass/Fail Result via Digital I/O

### วิธีที่ง่ายที่สุด: Digital I/O

```
Vision System ↔ PLC via Digital I/O:
┌─────────────────────────────────────────────────────────────┐
│  Vision Controller                      PLC                 │
│  ┌──────────────────────┐              ┌─────────────────┐ │
│  │                      │  Pass (DO0)  │                 │ │
│  │  Camera → Inspect    ├─────────────►│ DI0: Part OK   │ │
│  │  OK → DO0=1          │              │                 │ │
│  │  NG → DO1=1          │  Fail (DO1)  │                 │ │
│  │                      ├─────────────►│ DI1: Part NG   │ │
│  │  Trigger from PLC    │◄─────────────┤ DO0: Trigger   │ │
│  └──────────────────────┘              └─────────────────┘ │
│                                                             │
│  Timing:                                                   │
│  ─────────────────────────────────────────────────────     │
│  PLC: Trigger ─────┐                                       │
│                    │ (camera captures)                     │
│                    ▼                                       │
│  Vision: Processing ──────────────┐                        │
│                                   │ Result ready           │
│                                   ▼                        │
│  Vision: Pass/Fail ────────────────── (DO output)         │
│                                                            │
│  PLC reads result after timeout (typically 200-500ms)     │
└─────────────────────────────────────────────────────────────┘
```

### PLC Program สำหรับ I/O Vision Handshake

```ladder
// Vision System I/O Handshake Program
// 
// Network 1: Trigger Vision Camera
// เมื่อ part ถึง inspection position
//    Part_At_Inspect   Vision_Not_Busy   Vision_Trigger
//    ---[ ]---------------[/]-----------------( )------

+--[ Part_At_Inspect ]--+--[ NOT Vision_Busy ]--+--( Vision_Trigger_Out )--+

// Network 2: Timeout Timer (if vision doesn't respond)
+--[ Vision_Trigger_Out ]--+--[TON Timeout T1 PT=1S]--+

// Network 3: Read Result (after result ready or timeout)
//    Vision_Result_Ready    Vision_Pass    Part_Pass_Latch
//    ---[ ]------------------[ ]------------------( )----

+--[ Vision_Result_Ready ]--+--[ Vision_Pass_In ]--+--( Set Part_Pass )--+
+--[ Vision_Result_Ready ]--+--[ Vision_Fail_In  ]--+--( Set Part_Fail )--+

// Network 4: Timeout handling
+--[ Timeout_T1.Q ]--+--[ NOT Vision_Result_Ready ]--+--( Vision_Error )--+

// Network 5: Reset trigger after result
+--[ Part_Pass ]--+--( Reset Vision_Trigger )--+
+--[ Part_Fail ]--+--( Reset Vision_Trigger )--+
```

---

## Step 944: Data via Serial/Ethernet

### Ethernet Communication (TCP/IP)

```
Vision over Ethernet (TCP Socket):
┌─────────────────────────────────────────────────────────────┐
│  Vision System (Server):                                    │
│  ├── IP: 192.168.1.100, Port: 3000                        │
│  ├── Sends: "PASS,X=125.3,Y=89.7,ANGLE=0.5\r\n"          │
│  └── Protocol: ASCII string, terminated with \r\n         │
│                                                             │
│  PLC (Client):                                             │
│  ├── Connect at startup                                    │
│  ├── Send trigger: "TRIGGER\r\n"                          │
│  └── Receive result string, parse values                  │
│                                                             │
│  Data Format Example:                                      │
│  Pass: "PASS,1,125.30,089.70,000.50\r\n"                  │
│        PASS=status, 1=count, X, Y, Angle                  │
│  Fail: "FAIL,2,ERR004\r\n"                                │
│        FAIL=status, 2=fail count, ERR=error code         │
└─────────────────────────────────────────────────────────────┘
```

### TCP Communication in TIA Portal

```pascal
// TIA Portal - TCON (TCP Connect) + TSEND/TRCV
// FB: Vision_Ethernet_Comm

FUNCTION_BLOCK Vision_Ethernet_Comm
VAR
    // TCP blocks
    TCON_Block    : TCON;
    TSEND_Block   : TSEND;
    TRCV_Block    : TRCV;
    TDISCON_Block : TDISCON;
    
    // Connection parameters
    Connection_ID : UINT := 1;
    Connection_Data : TCON_IP_V4;
    
    // Buffers
    Send_Buffer   : ARRAY[0..31] OF BYTE;
    Recv_Buffer   : ARRAY[0..127] OF BYTE;
    
    // Parsed results
    Result_String : STRING[80];
    Part_Status   : STRING[4];   // "PASS" or "FAIL"
    Part_X        : REAL;
    Part_Y        : REAL;
    Part_Angle    : REAL;
    
    // State
    Connected     : BOOL;
    Step          : INT;
END_VAR

BEGIN
    // Setup connection parameters
    Connection_Data.InterfaceId := 64;      // X1 interface
    Connection_Data.ID          := Connection_ID;
    Connection_Data.ConnectionType := 11;   // TCP
    Connection_Data.ActiveEstablished := TRUE;
    Connection_Data.RemoteAddress.ADDR[0] := 192;
    Connection_Data.RemoteAddress.ADDR[1] := 168;
    Connection_Data.RemoteAddress.ADDR[2] := 1;
    Connection_Data.RemoteAddress.ADDR[3] := 100;
    Connection_Data.RemotePort := 3000;
    Connection_Data.LocalPort  := 0;        // Dynamic port
    
    // Establish connection (always call)
    TCON_Block(
        REQ  := TRUE,
        ID   := Connection_ID,
        CONNECT := Connection_Data,
        DONE => Connected
    );
    
    IF Connected THEN
        CASE Step OF
        
            0: // Idle - Wait for trigger command
                IF Trigger_Request THEN
                    // Prepare trigger message
                    Send_Buffer[0] := 84;  // 'T'
                    Send_Buffer[1] := 82;  // 'R'
                    Send_Buffer[2] := 73;  // 'I'
                    Send_Buffer[3] := 71;  // 'G'
                    Send_Buffer[4] := 13;  // CR
                    Send_Buffer[5] := 10;  // LF
                    Step := 10;
                END_IF;
            
            10: // Send Trigger
                TSEND_Block(
                    REQ  := TRUE,
                    ID   := Connection_ID,
                    LEN  := 6,
                    DATA := Send_Buffer
                );
                IF TSEND_Block.DONE THEN
                    Step := 20;
                END_IF;
            
            20: // Wait for Result
                TRCV_Block(
                    EN_R := TRUE,
                    ID   := Connection_ID,
                    LEN  := 128,
                    DATA => Recv_Buffer
                );
                IF TRCV_Block.NDDR > 0 THEN
                    // Data received - parse
                    Parse_Vision_Result(Recv_Buffer, 
                                       Part_Status, Part_X, Part_Y, Part_Angle);
                    Step := 0;
                    Result_Valid := TRUE;
                END_IF;
        
        END_CASE;
    END_IF;
END_FUNCTION_BLOCK
```

---

## Step 945: Cognex DataMan/In-Sight Integration

### Cognex In-Sight Series

```
Cognex In-Sight 7000 Integration:
┌─────────────────────────────────────────────────────────────┐
│  Communication Options:                                     │
│  1. EtherNet/IP Native (สำหรับ AB PLC)                     │
│     - Direct I/O assembly mapping                          │
│     - Assembly #112: Output (PLC → Camera)                 │
│     - Assembly #111: Input (Camera → PLC)                  │
│                                                             │
│  2. Modbus TCP                                             │
│     - Register-based data exchange                         │
│     - Simple to implement                                  │
│                                                             │
│  3. PROFINET IO                                            │
│     - Native integration ใน TIA Portal                    │
│     - GSD file available from Cognex                      │
│                                                             │
│  4. OPC UA Server                                          │
│     - In-Sight acts as OPC UA server                      │
│     - Access all result data as OPC nodes                 │
│                                                             │
│  5. Native Protocols: FTP, Telnet, RS-232                 │
└─────────────────────────────────────────────────────────────┘
```

### Cognex In-Sight EtherNet/IP (Studio 5000)

```
Studio 5000 - Add Cognex In-Sight as EIP Device:

// Device: Cognex In-Sight (ENBT Module)
// IP: 192.168.1.200

// Input Assembly (Camera → PLC): 
// INT[0]: Acquisition count
// INT[1]: Result (bit0=Pass, bit1=Fail, bit2=Error)
// REAL[2]: Cell 0 Value (e.g., X position)
// REAL[4]: Cell 1 Value (e.g., Y position)
// REAL[6]: Cell 2 Value (e.g., Score)

// Output Assembly (PLC → Camera):
// BOOL[0]: Soft trigger
// INT[0]: Acquisition mode (0=Continuous, 1=Single)
// INT[1]: Job number (select inspection job)

// Studio 5000 Tags:
Vision_Result_Pass   : BOOL  (from input assembly bit 1.0)
Vision_Result_Fail   : BOOL  (from input assembly bit 1.1)
Vision_X_Pos         : REAL  (from input assembly REAL[2])
Vision_Y_Pos         : REAL  (from input assembly REAL[4])
Vision_Soft_Trigger  : BOOL  (to output assembly bit 0.0)
```

---

## Step 946: KEYENCE IV Series

### KEYENCE IV-H Integration

```
KEYENCE IV-H Series Smart Camera:
┌─────────────────────────────────────────────────────────────┐
│  Communication Options:                                     │
│  1. NPN/PNP Output (PASS/FAIL digital)                    │
│     - Direct connection to PLC digital input              │
│     - Simplest integration                               │
│                                                             │
│  2. RS-232C Serial                                         │
│     - Text results via serial port                        │
│     - Format: CSV string with measurement values          │
│                                                             │
│  3. Ethernet (FTP/Telnet)                                  │
│     - FTP: Download images and result logs                │
│     - Telnet: Send commands, receive results              │
│                                                             │
│  4. EtherNet/IP                                           │
│     - For Omron/Rockwell PLC integration                  │
│                                                             │
│  Strobe Output:                                            │
│  ├── Camera สามารถ trigger strobe light ภายใน             │
│  └── หรือรับ external trigger จาก PLC                    │
│                                                             │
│  Example KEYENCE IV-H DI/DO Wiring:                       │
│  ├── External trigger: PLC DO → IV-H TRIG IN             │
│  ├── Pass output: IV-H PASS OUT → PLC DI                 │
│  └── Fail output: IV-H NG OUT → PLC DI                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 947: Barcode/QR Reading

### Barcode Integration with PLC

```
Barcode/QR Code Reading System:
┌─────────────────────────────────────────────────────────────┐
│  Reader Types:                                              │
│  ├── Handheld scanner (manual scan)                        │
│  ├── Fixed mount scanner (inline)                         │
│  └── Camera-based reader (vision + decode)               │
│                                                             │
│  Common Protocols:                                         │
│  ├── RS-232/RS-485 (legacy)                               │
│  ├── USB HID (keyboard emulation)                        │
│  ├── Ethernet TCP/IP                                      │
│  └── EtherNet/IP / PROFINET                              │
│                                                             │
│  Data Format:                                             │
│  └── Typically: [STX] [DATA] [ETX] or just DATA + CRLF   │
│                                                             │
│  Example: Assembly traceability                           │
│  ├── Scan part barcode at each station                   │
│  ├── PLC reads barcode, checks against work order        │
│  └── Log scan data to MES/Historian                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 948-960: Complete Vision-Guided Sorting System

### โครงการ: Vision-Guided Color and Defect Sorting

#### 1. System Description

```
Sorting System Overview:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Input     Vision     Sorting      Reject    Accept        │
│  Conveyor  Station    Mechanism    Bin       Conveyor      │
│                                                             │
│  [Parts]──►[Camera]──►[Sorter]──►[NG Bin]               │
│   moving   triggers   diverts     rejected parts          │
│                       NG parts                            │
│                          │                                 │
│                          └─────────────────►[Good Parts]  │
│                                                            │
│  Camera Station:                                          │
│  - Cognex In-Sight 9000                                   │
│  - Back-lighting (silhouette for shape)                   │
│  - Strobe trigger from PLC (part position)               │
│  - EtherNet/IP communication                             │
│                                                             │
│  Inspection Tasks:                                        │
│  1. Part presence (is there a part?)                     │
│  2. Shape check (correct part type)                      │
│  3. Defect detection (cracks, chips)                     │
│  4. Color sorting (3 colors: Red, Blue, Green)           │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Vision Job Configuration (Cognex In-Sight)

```
In-Sight Job Setup:
─────────────────────────────────────────────────────────────
Job 1: Shape + Defect Inspection

Tools used:
1. PatMax (Pattern Matching) - Find part
   - Template: Good part image
   - Output: Found (Y/N), Score

2. Blob Detection - Presence check
   - Threshold: Part brightness
   - Min area: 1000 pixels

3. Edge Inspection - Defect check
   - Check for cracks along 4 edges
   - Output: Pass/Fail per edge

4. Color Analysis - Color sorting
   - ROI (Region of Interest) on part
   - Hue range for Red: 0-15, 165-180
   - Hue range for Blue: 200-250
   - Hue range for Green: 90-150
   - Output: Color code (1=Red, 2=Blue, 3=Green, 0=Other)

Pass Criteria:
- PatMax score > 0.9
- Blob area 1000-5000 pixels
- All edges pass
- Color code != 0

Output Cells:
- Cell 0: Pass/Fail (1/0)
- Cell 1: Color code (1-3)
- Cell 2: Part X position
- Cell 3: Part Y position
```

#### 3. Complete Ladder Program

```pascal
// ===================================================
// FB: VisionSortingSystem
// Vision-guided sorting with Cognex In-Sight
// ===================================================

FUNCTION_BLOCK VisionSortingSystem
VAR_INPUT
    Part_Sensor      : BOOL;    // Photoelectric sensor: part entering
    Belt_Encoder     : DINT;    // Encoder count for position tracking
    
    // EtherNet/IP data from Cognex (mapped to input assembly)
    Vision_AcqCount  : INT;     // Acquisition count (changes each shot)
    Vision_Pass      : BOOL;    // Pass result
    Vision_Fail      : BOOL;    // Fail result
    Vision_Cell0     : INT;     // Pass(1)/Fail(0)
    Vision_Cell1     : INT;     // Color code
    Vision_Cell2     : REAL;    // Part X
    Vision_Cell3     : REAL;    // Part Y
END_VAR

VAR_OUTPUT
    // EtherNet/IP data to Cognex
    Camera_Trigger   : BOOL;    // Soft trigger to camera
    Camera_Job       : INT;     // Job selection
    
    // Sorting outputs
    Sorter_NG_Gate   : BOOL;    // Open NG diverter gate
    Sorter_Color1    : BOOL;    // Divert to Color 1 bin (Red)
    Sorter_Color2    : BOOL;    // Divert to Color 2 bin (Blue)
    Sorter_Color3    : BOOL;    // Divert to Color 3 bin (Green)
    
    // Counters
    Parts_Total      : DINT;
    Parts_Pass       : DINT;
    Parts_Fail_NG    : DINT;
    Parts_Red        : DINT;
    Parts_Blue       : DINT;
    Parts_Green      : DINT;
    
    // Status
    System_Running   : BOOL;
    Camera_Error     : BOOL;
END_VAR

VAR
    // Part tracking (FIFO queue for belt delay)
    Part_Queue       : ARRAY[0..9] OF STRUCT
        Active       : BOOL;
        Encoder_Pos  : DINT;
        Pass         : BOOL;
        Color        : INT;
    END_STRUCT;
    Queue_Head       : INT := 0;
    Queue_Tail       : INT := 0;
    
    // Timing
    Trigger_Sent     : BOOL;
    Last_AcqCount    : INT;
    Result_Received  : BOOL;
    
    // Belt positions (in encoder counts)
    Camera_Pos       : DINT := 1000;   // Camera at 1000 counts from sensor
    Sorter_Pos       : DINT := 5000;   // Sorter at 5000 counts from sensor
    
    // Timers
    Gate_Timer       : TON;
    Vision_Timeout   : TON;
END_VAR

BEGIN
    // ---- Part Detection and Queue ----
    IF Part_Sensor AND NOT Trigger_Sent THEN
        
        // Add to queue
        Part_Queue[Queue_Head].Active := TRUE;
        Part_Queue[Queue_Head].Encoder_Pos := Belt_Encoder + Camera_Pos;
        Part_Queue[Queue_Head].Pass := FALSE;
        Part_Queue[Queue_Head].Color := 0;
        Queue_Head := (Queue_Head + 1) MOD 10;
        
        Parts_Total := Parts_Total + 1;
        Trigger_Sent := TRUE;
    END_IF;
    
    IF NOT Part_Sensor THEN
        Trigger_Sent := FALSE;
    END_IF;
    
    // ---- Camera Trigger ----
    // Wait for part to reach camera position
    IF Part_Queue[Queue_Tail].Active THEN
        IF Belt_Encoder >= Part_Queue[Queue_Tail].Encoder_Pos AND NOT Camera_Trigger THEN
            Camera_Trigger := TRUE;
            Vision_Timeout(IN := FALSE, PT := T#2S);  // Reset
            Vision_Timeout(IN := TRUE, PT := T#2S);   // Start timeout
        END_IF;
    END_IF;
    
    // ---- Read Vision Result ----
    IF Vision_AcqCount <> Last_AcqCount THEN
        // New result available
        Last_AcqCount := Vision_AcqCount;
        Camera_Trigger := FALSE;  // Reset trigger
        
        // Store result in queue entry
        Part_Queue[Queue_Tail].Pass := Vision_Pass;
        Part_Queue[Queue_Tail].Color := Vision_Cell1;
        
        // Update counters
        IF Vision_Pass THEN
            Parts_Pass := Parts_Pass + 1;
            CASE Vision_Cell1 OF
                1: Parts_Red   := Parts_Red + 1;
                2: Parts_Blue  := Parts_Blue + 1;
                3: Parts_Green := Parts_Green + 1;
            END_CASE;
        ELSE
            Parts_Fail_NG := Parts_Fail_NG + 1;
        END_IF;
        
        Result_Received := TRUE;
        Vision_Timeout(IN := FALSE);  // Clear timeout
    END_IF;
    
    // ---- Vision Timeout ----
    IF Vision_Timeout.Q THEN
        // Camera didn't respond - treat as fail
        Part_Queue[Queue_Tail].Pass := FALSE;
        Part_Queue[Queue_Tail].Color := 0;
        Camera_Error := TRUE;
        Camera_Trigger := FALSE;
        Vision_Timeout(IN := FALSE);
    END_IF;
    
    // ---- Sorting at Sorter Position ----
    FOR I := 0 TO 9 DO
        IF Part_Queue[I].Active AND Result_Received THEN
            IF Belt_Encoder >= (Part_Queue[I].Encoder_Pos - Camera_Pos + Sorter_Pos) THEN
                
                // Activate correct sorter
                IF NOT Part_Queue[I].Pass THEN
                    // NG Part
                    Sorter_NG_Gate := TRUE;
                    Gate_Timer(IN := TRUE, PT := T#200MS);
                ELSE
                    CASE Part_Queue[I].Color OF
                        1: Sorter_Color1 := TRUE;
                        2: Sorter_Color2 := TRUE;
                        3: Sorter_Color3 := TRUE;
                    END_CASE;
                    Gate_Timer(IN := TRUE, PT := T#200MS);
                END_IF;
                
                // Remove from queue
                Part_Queue[I].Active := FALSE;
                Queue_Tail := (Queue_Tail + 1) MOD 10;
            END_IF;
        END_IF;
    END_FOR;
    
    // ---- Reset Gates after timer ----
    IF Gate_Timer.Q THEN
        Gate_Timer(IN := FALSE);
        Sorter_NG_Gate := FALSE;
        Sorter_Color1 := FALSE;
        Sorter_Color2 := FALSE;
        Sorter_Color3 := FALSE;
    END_IF;
    
    // ---- Camera Error Reset ----
    IF Reset_Camera_Error THEN
        Camera_Error := FALSE;
    END_IF;
    
END_FUNCTION_BLOCK
```

#### 4. OEE Tracking

```pascal
// Quality Metric Calculation
// OEE (Overall Equipment Effectiveness) for Sorting System

VAR
    Shift_Start_Time : DATE_AND_TIME;
    Shift_Duration   : REAL;  // hours
    Planned_Rate     : REAL := 120.0;  // parts/minute
    
    Yield_Rate       : REAL;
    Performance_Rate : REAL;
    Availability     : REAL;
    OEE              : REAL;
END_VAR

// Calculate OEE
Shift_Duration := (Current_Time - Shift_Start_Time) / 3600.0;  // hours

// Quality = Good Parts / Total Parts
IF Parts_Total > 0 THEN
    Yield_Rate := Parts_Pass / Parts_Total * 100.0;
END_IF;

// Performance = Actual Rate / Planned Rate
IF Shift_Duration > 0.0 THEN
    Actual_Rate := Parts_Total / (Shift_Duration * 60.0);  // parts/min
    Performance_Rate := Actual_Rate / Planned_Rate * 100.0;
END_IF;

// OEE = Quality × Performance × Availability
OEE := (Yield_Rate / 100.0) * (Performance_Rate / 100.0) * (Availability / 100.0) * 100.0;
```

#### 5. HMI Vision Display

```
HMI Vision Display Layout:
┌──────────────────────────────────────────────────────────┐
│  VISION SORTING SYSTEM                    SHIFT: A       │
│  ──────────────────────────────────────────────────────  │
│                                                          │
│  LIVE IMAGE (350×250 px)           STATISTICS           │
│  ┌─────────────────────────┐       ─────────────────    │
│  │                         │       Total:    1,245       │
│  │   [Camera feed here]    │       Pass:     1,187 (95%)│
│  │                         │       Fail (NG):   58 (5%) │
│  └─────────────────────────┘                            │
│                                   COLOR COUNTS          │
│  Camera Status: CONNECTED         ─────────────────     │
│  Job: 1 (Full Inspect)            Red:     425          │
│  Trigger: Auto                    Blue:    389          │
│  Score: 0.97 (Pass)               Green:   373          │
│                                                          │
│  LAST RESULT                      OEE: 91.2%            │
│  ┌─────────────────────────┐                            │
│  │  PASS - Color: BLUE     │   [RESET STATS]            │
│  │  X: 125.3  Y: 89.7     │   [CAMERA SETUP]           │
│  └─────────────────────────┘   [FAULT LOG]              │
└──────────────────────────────────────────────────────────┘
```

---

## Lab Project 48: PCB Component Inspection System

### โจทย์
ออกแบบ Vision Inspection System สำหรับ PCB (Printed Circuit Board) หลัง soldering

### การตรวจสอบที่ต้องทำ
1. Component presence check (ชิ้นส่วนครบหรือไม่)
2. Component orientation (ทิศทางถูกต้องหรือไม่)
3. Solder bridge detection (Soldering ติดกันหรือไม่)
4. Barcode reading (Board serial number)
5. Date code verification (Manufacturing date)

### ข้อกำหนด Hardware
- Camera: Cognex In-Sight 9902L (12MP)
- Lighting: Ring light (bright field) + back light
- Resolution ต้องการ: 0.05mm/pixel (ตรวจสอบ 0.3mm features)

### งานที่ต้องทำ
1. คำนวณ camera resolution ที่ต้องการ
2. เลือก lens (focal length)
3. ออกแบบ lighting setup
4. Configuration Vision Job ใน In-Sight Explorer
5. เขียน PLC Integration Program
6. ทำ Statistical Process Control (SPC) monitoring

---

## สรุป Part 48

| เทคโนโลยี | ความเร็ว | ความแม่นยำ | ราคา | ใช้กับ |
|----------|---------|----------|------|-------|
| Digital I/O | เร็ว | ต่ำ (Pass/Fail) | ถูก | Simple inspection |
| Serial (RS-232) | ช้า | สูง (ข้อมูลมาก) | ปานกลาง | Legacy systems |
| EtherNet/IP | เร็ว | สูง | ปานกลาง | Rockwell systems |
| PROFINET | เร็ว | สูง | ปานกลาง | Siemens systems |
| OPC UA | ปานกลาง | สูง (ยืดหยุ่น) | สูง | Multi-vendor, IIoT |

---

## การนำทาง

| | ลิงก์ |
|---|---|
| ◄ ก่อนหน้า | [Part 47: Robotics Integration](part47_robotics_integration.md) |
| ► ถัดไป | [Part 49: IoT & Industry 4.0](part49_iot_industry40.md) |
| 🏠 หน้าหลัก | [README](../README.md) |
