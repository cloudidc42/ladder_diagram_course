# Part 84: Collaborative Robot (Cobot) Programming
## Steps 1661-1680 | ระดับ Capstone

---

## การนำทาง
- [← Part 83: AI/ML Integration กับ PLC](part83_ai_plc.md)
- [→ Part 85: AGV และ AMR Integration](part85_agv_amr.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1661: Cobot vs Industrial Robot

### ความแตกต่างระหว่าง Cobot และ Robot ดั้งเดิม

```
┌────────────────────────────────────────────────────────────────────┐
│          COBOT vs INDUSTRIAL ROBOT COMPARISON                      │
├──────────────────────┬────────────────────┬────────────────────────┤
│ Attribute            │ Industrial Robot   │ Collaborative Robot    │
├──────────────────────┼────────────────────┼────────────────────────┤
│ Speed                │ สูง (2000mm/s+)    │ ปานกลาง (250mm/s)     │
│ Payload              │ 5-2000+ kg         │ 3-35 kg                │
│ Safety               │ Fencing จำเป็น     │ ไม่ต้องมี Fence        │
│ Programming          │ ซับซ้อน            │ ง่าย (Hand-guide)      │
│ Flexibility          │ น้อย               │ สูง (Redeploy ง่าย)    │
│ Price                │ 2-20M บาท          │ 0.5-3M บาท             │
│ Setup Time           │ สัปดาห์-เดือน     │ ชั่วโมง-วัน            │
│ ROI Payback          │ 3-5 ปี             │ 1-2 ปี                 │
│ Force Sensing        │ ไม่มี              │ มีทุกตัว               │
│ Human Proximity      │ ไม่ปลอดภัย         │ ปลอดภัย (ISO 10218-2)  │
└──────────────────────┴────────────────────┴────────────────────────┘
```

---

## Step 1662: Universal Robots (UR Series)

### UR3e, UR5e, UR10e, UR16e, UR20, UR30

```
UR Product Line:
┌──────────────────────────────────────────────────────────────┐
│ Model    │ Payload  │ Reach    │ Repeatability │ Use Case    │
├──────────┼──────────┼──────────┼───────────────┼─────────────┤
│ UR3e     │ 3 kg     │ 500 mm   │ ±0.03 mm      │ ประกอบชิ้นเล็ก │
│ UR5e     │ 5 kg     │ 850 mm   │ ±0.03 mm      │ หยิบจับทั่วไป │
│ UR10e    │ 10 kg    │ 1300 mm  │ ±0.05 mm      │ หยิบจับหนัก │
│ UR16e    │ 16 kg    │ 900 mm   │ ±0.05 mm      │ Welding, Material │
│ UR20     │ 20 kg    │ 1750 mm  │ ±0.05 mm      │ Heavy Duty       │
│ UR30     │ 30 kg    │ 1300 mm  │ ±0.05 mm      │ Heaviest Payload │
└──────────┴──────────┴──────────┴───────────────┴─────────────┘
```

### UR Polyscope Programming

```python
# URScript - ภาษา Programming ของ Universal Robots
# ตัวอย่าง: Pick and Place สำหรับ Assembly Station

# กำหนดตำแหน่ง (x, y, z ในเมตร, rx, ry, rz ใน radian)
var home_pose = p[0.0, -0.5, 0.4, 3.14, 0.0, 0.0]
var pick_pose = p[0.3, -0.2, 0.1, 3.14, 0.0, 0.0]
var pick_approach = p[0.3, -0.2, 0.25, 3.14, 0.0, 0.0]
var place_pose = p[-0.3, -0.2, 0.15, 3.14, 0.0, 0.0]
var place_approach = p[-0.3, -0.2, 0.3, 3.14, 0.0, 0.0]

# Setup Digital I/O
# DO[0] = Gripper Open/Close
# DI[0] = Part Detected Sensor
# DI[1] = PLC Ready Signal

def pick_and_place():
    # ไปตำแหน่ง Home
    movej(home_pose, v=1.0, a=1.2, r=0)
    
    # รอ Signal จาก PLC
    wait DI[1] == True
    
    # ตรวจสอบว่ามีชิ้นงาน
    if DI[0] == False:
        popup("ไม่พบชิ้นงาน - ตรวจสอบ Feeder", title="Error", blocking=True)
        return
    end
    
    # เข้าสู่ตำแหน่ง Pick (Approach ก่อน)
    movel(pick_approach, v=0.5, a=0.5, r=0)
    
    # ลงช้าๆ ไป Pick Position
    movel(pick_pose, v=0.1, a=0.1, r=0)
    
    # ปิด Gripper
    set_digital_out(0, True)
    sleep(0.3)  # รอ Gripper ปิด
    
    # ตรวจสอบว่า Gripper จับชิ้นงานได้
    # (ใช้ Force Sensor ตรวจสอบ)
    force = get_tcp_force()
    if norm(force) < 5.0:  # น้อยกว่า 5 N = ไม่ได้จับ
        popup("Gripper ไม่ได้จับชิ้นงาน", title="Gripper Error", blocking=True)
        set_digital_out(0, False)
        movej(home_pose, v=1.0, a=1.2)
        return
    end
    
    # ยกขึ้นจาก Pick Position
    movel(pick_approach, v=0.3, a=0.3, r=0)
    
    # เคลื่อนที่ไป Place Approach
    movel(place_approach, v=0.8, a=0.8, r=0)
    
    # ลงช้าๆ ไป Place Position
    movel(place_pose, v=0.1, a=0.1, r=0)
    
    # เปิด Gripper (วางชิ้นงาน)
    set_digital_out(0, False)
    sleep(0.2)
    
    # ถอยออกจาก Place
    movel(place_approach, v=0.3, a=0.3, r=0)
    
    # กลับ Home
    movej(home_pose, v=1.0, a=1.2, r=0)
    
    # แจ้ง PLC ว่าทำเสร็จแล้ว
    set_digital_out(1, True)
    sleep(0.1)
    set_digital_out(1, False)

end

# Main Program
while True:
    pick_and_place()
    sync()
end
```

---

## Step 1663: Force/Torque Sensing

### การใช้ Force/Torque Sensor ในงาน Cobot

```python
# URScript - Force-Controlled Assembly
# ใช้ Force Sensing สำหรับงานประกอบที่ต้องการความละเอียด

def force_guided_insertion():
    """ประกอบ Connector โดยใช้ Force Control"""
    
    var target_force = 50.0     # แรงที่ต้องการ (N)
    var max_force = 100.0       # แรงสูงสุดที่ยอมรับได้ (N)
    var insertion_speed = 0.005 # ความเร็วการสอด (5mm/s)
    var search_force = 15.0     # แรงสำหรับค้นหาตำแหน่ง
    
    # เคลื่อนที่ไปยังตำแหน่งเริ่มต้น
    movel(approach_pose, v=0.3, a=0.3)
    
    # Force Search: ค้นหาตำแหน่งที่ถูกต้อง (Spiral Search)
    var angle = 0
    var radius = 0
    var max_radius = 0.005  # 5mm radius
    
    while radius <= max_radius:
        var x_offset = radius * cos(angle)
        var y_offset = radius * sin(angle)
        
        # เคลื่อนที่ในรูปแบบ Spiral
        movel(pose_add(search_pose, p[x_offset, y_offset, 0, 0, 0, 0]),
              v=0.01, a=0.01)
        
        # ลองกด
        force_mode(
            p[0, 0, -1, 0, 0, 0],  # ทิศทาง: กดลง (-Z)
            [0, 0, 1, 0, 0, 0],    # Control: Z, Rx, Ry free
            [0, 0, search_force, 0, 0, 0],  # Target Force
            2,                      # Type 2 = Frame
            [0.1, 0.1, 0.1, 0.5, 0.5, 0.5]  # Limits
        )
        sleep(0.2)
        end_force_mode()
        
        # ตรวจสอบว่า Contact แล้วหรือยัง
        var current_force = get_tcp_force()
        if norm(current_force) > search_force * 0.8:
            # พบตำแหน่งแล้ว!
            break
        end
        
        # เพิ่ม Radius ทีละนิด
        angle = angle + 0.3
        radius = radius + 0.0002
    end
    
    # Force-Controlled Insertion
    force_mode(
        p[0, 0, -1, 0, 0, 0],     # กดลงใน Z Direction
        [0, 0, 1, 0, 0, 0],       # Compliant ใน Z
        [0, 0, target_force, 0, 0, 0],
        2,
        [0.01, 0.01, 0.05, 0.1, 0.1, 0.1]
    )
    
    # รอจนกว่าจะสอดสำเร็จ (Force เพิ่มขึ้นถึง Target)
    var timeout = 5.0  # 5 วินาที
    var elapsed = 0
    
    while elapsed < timeout:
        var f = get_tcp_force()
        if f[2] >= target_force * 0.95:  # สอดสำเร็จ
            textmsg("Insertion สำเร็จ!")
            break
        end
        
        if f[2] > max_force:  # แรงมากเกินไป
            end_force_mode()
            popup("แรงเกิน Max - ตรวจสอบ Alignment", title="Force Error", blocking=True)
            return
        end
        
        elapsed = elapsed + 0.05
        sleep(0.05)
    end
    
    end_force_mode()
    
    # ยืนยันการสอด (ตรวจสอบด้วย Encoder Position)
    var final_pos = get_actual_tcp_pose()
    if final_pos[2] < expected_depth - 0.001:
        textmsg("Insertion depth OK")
        set_digital_out(2, True)  # แจ้ง PLC ว่าสำเร็จ
    else:
        textmsg("Insertion ไม่สมบูรณ์")
        set_digital_out(3, True)  # แจ้ง PLC ว่าผิดพลาด
    end

end
```

---

## Step 1664: Human Presence Detection

### Safety System สำหรับ Human-Robot Collaboration

```ladder
// ============================================
// Cobot Safety - Human Presence Detection
// ISO 10218-2 / ISO/TS 15066
// ============================================

// NETWORK 1: Safety Sensor Matrix
// Zone 1 = Near Robot (0-500mm) - Stop ทันที
// Zone 2 = Medium (500-1000mm) - Reduced Speed
// Zone 3 = Far (1000-2000mm) - Monitor Only

|----[Safety_Scanner_Zone1]----( COBOT_EMERGENCY_STOP )---------------------|
|                                                                             |
|----[Safety_Scanner_Zone2]----[NOT Zone1]----( COBOT_SPEED_LIMIT_20PCT )----|
|                                                                             |
|----[Safety_Scanner_Zone3]----[NOT Zone2]----[NOT Zone1]----( COBOT_ALERT )--|

// NETWORK 2: Cobot Speed Adjustment
|----[COBOT_SPEED_LIMIT_20PCT]-----|
|   // ส่งสัญญาณ Speed Override    |
|   DB_Cobot.Speed_Override := 20  | // 20% ของ Normal Speed
|   DB_Cobot.Speed_Cmd_Active := 1 |
|---------------------------------|

// NETWORK 3: Emergency Stop Handling
|----[COBOT_EMERGENCY_STOP]----( ESTOP_COBOT_Output )---------------------|
|                                                                           |
// รอให้ ESTOP ถูก Reset โดย Operator (Safety Reset Required)
|----[ESTOP_COBOT_Output]----[Safety_Reset_Button]----[T_Delay: 2s]--------|
|                                         ( Clear_COBOT_ESTOP )-----------  |

// NETWORK 4: Safe Restart Sequence
|----[Clear_COBOT_ESTOP]----[NOT Zone1]----[NOT Zone2]------|
|   IF Cobot_In_Safe_Position THEN                          |
|      COBOT_Ready_To_Restart := 1                          |
|   ELSE                                                    |
|      // ต้อง Move กลับ Safe Position ก่อน               |
|      COBOT_Return_Home_Cmd := 1                           |
|   END_IF                                                  |
|----------------------------------------------------------|
```

---

## Step 1665: PLC-Cobot Communication

### PROFINET Interface กับ Universal Robots

```
UR PROFINET Setup:
1. ติดตั้ง PROFINET Module ใน UR PolyScope
   Installation → URCaps → PROFINET I/O

2. ตั้งค่า IP Address
   System → Network → PROFINET IP: 192.168.1.20

3. GSD File
   Download จาก: ur.robots.com/resources/gsd-files
   Import ใน TIA Portal: Options → Manage general station description files

Data Exchange สำหรับ UR-PLC:
┌────────────────────────────────────────────────────────────┐
│ Output (PLC → Robot) - 144 bytes                           │
├──────────────────────┬─────────────────────────────────────┤
│ Bytes 0-3            │ Control Word (Start, Stop, Reset...) │
│ Bytes 4-7            │ Program Number to Execute           │
│ Bytes 8-11           │ Speed Override (0-100%)             │
│ Bytes 12-59          │ Target Pose (6 x REAL)              │
│ Bytes 60-107         │ Reserved                            │
└──────────────────────┴─────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ Input (Robot → PLC) - 144 bytes                            │
├──────────────────────┬─────────────────────────────────────┤
│ Bytes 0-3            │ Status Word                         │
│ Bytes 4-7            │ Current Program Number              │
│ Bytes 8-11           │ Robot Mode (Idle, Moving, Fault...) │
│ Bytes 12-59          │ Actual TCP Pose (6 x REAL)          │
│ Bytes 60-71          │ Joint Positions (6 x REAL)          │
│ Bytes 72-83          │ TCP Force/Torque (6 x REAL)         │
└──────────────────────┴─────────────────────────────────────┘
```

### TIA Portal: Function Block สำหรับ UR Communication

```pascal
// SCL - UR Cobot Communication FB
FUNCTION_BLOCK FB_UR_Cobot_Comm
VAR_INPUT
    PROFINET_Input  : ARRAY[0..143] OF BYTE;  // จาก UR
    CMD_Start       : BOOL;                    // คำสั่ง Start
    CMD_Stop        : BOOL;                    // คำสั่ง Stop
    Program_Number  : INT;                     // โปรแกรมที่ต้องการรัน
    Speed_Override  : INT;                     // ความเร็ว 0-100%
END_VAR

VAR_OUTPUT
    PROFINET_Output : ARRAY[0..143] OF BYTE;  // ส่งไป UR
    Robot_Running   : BOOL;
    Robot_Faulted   : BOOL;
    Robot_At_Home   : BOOL;
    TCP_X, TCP_Y, TCP_Z : REAL;               // ตำแหน่ง TCP ปัจจุบัน
    TCP_Fx, TCP_Fy, TCP_Fz : REAL;           // แรง/Torque
END_VAR

VAR
    Status_Word     : DWORD;
    Control_Word    : DWORD;
    Temp_Real       : REAL;
END_VAR

// === อ่าน Status จาก UR ===
Status_Word := DWORD_FROM_BYTES(
    PROFINET_Input[3], PROFINET_Input[2],
    PROFINET_Input[1], PROFINET_Input[0]
);

// แปลง Status Bits
Robot_Running := Status_Word.%X0;   // Bit 0 = Running
Robot_Faulted := Status_Word.%X4;   // Bit 4 = Fault
Robot_At_Home := Status_Word.%X8;   // Bit 8 = At Home

// อ่านตำแหน่ง TCP จาก Bytes 12-35
Temp_Real := REAL_FROM_BYTES(
    PROFINET_Input[15], PROFINET_Input[14],
    PROFINET_Input[13], PROFINET_Input[12]
);
TCP_X := Temp_Real * 1000.0;  // เปลี่ยน m → mm

// อ่าน TCP Force จาก Bytes 72-83
Temp_Real := REAL_FROM_BYTES(
    PROFINET_Input[75], PROFINET_Input[74],
    PROFINET_Input[73], PROFINET_Input[72]
);
TCP_Fz := Temp_Real;  // N

// === สร้าง Control Word ===
Control_Word := 0;

IF CMD_Start THEN
    Control_Word.%X0 := 1;  // Start Bit
END_IF;

IF CMD_Stop THEN
    Control_Word.%X1 := 1;  // Stop Bit
END_IF;

// เขียน Control Word
BYTES_FROM_DWORD(Control_Word,
    PROFINET_Output[3], PROFINET_Output[2],
    PROFINET_Output[1], PROFINET_Output[0]
);

// เขียน Program Number
PROFINET_Output[4] := INT_TO_BYTE(Program_Number);

// เขียน Speed Override
PROFINET_Output[8] := INT_TO_BYTE(Speed_Override);
```

---

## Step 1666: FANUC CRX Programming

### FANUC CRX (Collaborative Robot)

```
FANUC CRX Series:
CRX-5iA  : 5 kg payload, 994mm reach
CRX-10iA : 10 kg payload, 1249mm reach
CRX-25iA : 25 kg payload, 1889mm reach

Programming ด้วย FANUC Teach Pendant:
1. Robot Setup Wizard
2. Hand-Guided Teaching
3. iRVision (Vision System)
4. KAREL Language (Advanced)
5. DCS (Dual Check Safety)
```

### FANUC KAREL Program

```
! FANUC KAREL - Pick and Place Program
! สำหรับ CRX-10iA บน Assembly Line

PROGRAM PICK_PLACE
%STACKSIZE = 4096
%NOLOCKGROUP
%ENVIRONMENT FLBT

BEGIN
    -- ตัวแปร
    VAR
        part_detected : BOOLEAN
        gripper_closed : BOOLEAN
        pick_count : INTEGER
        error_count : INTEGER
    END_VAR
    
    pick_count = 0
    error_count = 0
    
    -- รีเซ็ต I/O
    DOUT[1] = OFF  -- Gripper Open
    
    -- วนลูปหลัก
    WHILE TRUE DO
        -- รอ Part Detected Signal
        WAIT FOR DIN[1] = ON TIMEOUT 30 SEC
        
        IF TIMEOUT THEN
            -- Timeout: แจ้งเตือน
            CALL SEND_ALARM(1001, 'PART FEEDER TIMEOUT')
            CONTINUE
        END_IF
        
        -- เคลื่อนไป Pick Position
        MOVE_TO_POSITION(PICK_APPROACH, SPEED=50)
        MOVE_TO_POSITION(PICK_POSE, SPEED=20)
        
        -- ปิด Gripper
        DOUT[1] = ON
        DELAY 300  -- รอ 300ms
        
        -- ตรวจสอบว่าจับได้
        IF DIN[5] = OFF THEN  -- Gripper Feedback
            error_count = error_count + 1
            CALL SEND_ALARM(1002, 'GRIPPER MISS')
            DOUT[1] = OFF
            MOVE_TO_POSITION(PICK_APPROACH, SPEED=30)
            CONTINUE
        END_IF
        
        -- ยกขึ้นและเคลื่อนที่
        MOVE_TO_POSITION(PICK_APPROACH, SPEED=30)
        MOVE_TO_POSITION(PLACE_APPROACH, SPEED=80)
        MOVE_TO_POSITION(PLACE_POSE, SPEED=20)
        
        -- เปิด Gripper (วาง)
        DOUT[1] = OFF
        DELAY 200
        
        -- ถอยออก
        MOVE_TO_POSITION(PLACE_APPROACH, SPEED=40)
        MOVE_TO_POSITION(HOME_POSE, SPEED=100)
        
        -- นับจำนวน
        pick_count = pick_count + 1
        
        -- แจ้ง PLC
        DOUT[2] = ON
        DELAY 100
        DOUT[2] = OFF
        
    END_WHILE

END PICK_PLACE
```

---

## Step 1667: Safety Categories สำหรับ Cobot

### ISO/TS 15066 - Power and Force Limiting

```
Safety Modes ของ Cobot (ISO/TS 15066):

1. Safety-Rated Monitored Stop (SMS)
   - หยุดเมื่อมนุษย์เข้ามาใน Zone
   - ต้องมี Manual Reset
   
2. Hand Guiding
   - มนุษย์นำทางโดยตรง
   - ความเร็วต่ำ, แรงควบคุม

3. Speed and Separation Monitoring (SSM)
   - ลดความเร็วตาม Distance
   - ยิ่งใกล้ ยิ่งช้า

4. Power and Force Limiting (PFL)
   - จำกัด Force ที่สัมผัสมนุษย์
   - Max Body Contact Force:
     * หัว/คอ: 130 N (Quasi-static), 65 N (Transient)
     * ลำตัว: 160 N / 110 N
     * มือ: 140 N / 60 N
```

### การคำนวณ Safe Speed

```
Safe Speed Formula (ISO/TS 15066):

v_max = (d - d_s) / T_r

where:
  d   = ระยะห่างระหว่างมนุษย์กับ Robot (mm)
  d_s = Safety Distance (mm) = Minimum Distance
  T_r = System Response Time (s)

ตัวอย่าง:
  d = 500 mm (Safety Scanner zone boundary)
  d_s = 150 mm (Minimum separation)
  T_r = 0.5 s (Response time: Scanner 150ms + PLC 50ms + Robot Stop 300ms)

  v_max = (500 - 150) / 0.5 = 700 mm/s
```

---

## Step 1668-1672: Complete Human-Robot Collaboration Station

### การออกแบบสถานี HRC (Human-Robot Collaboration)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 HRC STATION DESIGN - Electronic Assembly                │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  WORK CELL LAYOUT                                              │    │
│  │                                                                │    │
│  │   [SHELF]     [COBOT AREA]      [HUMAN WORK AREA]             │    │
│  │   ┌─────┐     ┌──────────┐      ┌────────────────┐            │    │
│  │   │Parts│     │  UR10e   │      │  Operator Desk │            │    │
│  │   │ Bin │     │          │      │  (Assembly)    │            │    │
│  │   └─────┘     └──────────┘      └────────────────┘            │    │
│  │                    │                    │                      │    │
│  │              ┌─────┴──────┐            │                      │    │
│  │              │  Shared    │            │                      │    │
│  │              │  Table     │────────────┘                      │    │
│  │              └────────────┘                                   │    │
│  │                                                                │    │
│  │  [Safety Scanner SICK nanoScan3 Pro]                          │    │
│  │  Zone 1: 0-400mm  → Emergency Stop                            │    │
│  │  Zone 2: 400-800mm → Speed 25%                               │    │
│  │  Zone 3: 800-1500mm → Speed 50%                              │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Ladder Diagram สำหรับ HRC Station Control

```ladder
// ============================================
// HRC Station - Siemens S7-1500 Safety PLC
// PROFISAFE + F-Modules
// ============================================

// NETWORK 1: Safety Scanner Input (F-Input)
|----[F_Safety_Scanner_1.Zone1]----( F_EStop_Cobot )------------------------|
|----[F_Safety_Scanner_1.Zone2]----( F_SpeedReduce_Zone2 )------------------|
|----[F_Safety_Scanner_1.Zone3]----( F_SpeedReduce_Zone3 )------------------|

// NETWORK 2: Speed Override Calculation
|----[F_SpeedReduce_Zone2]----[NOT F_SpeedReduce_Zone1]---|
|   DB_Cobot.Speed_Limit := 25                            |
|   → MOVE to Cobot Speed Override register               |
|--------------------------------------------------------|

|----[F_SpeedReduce_Zone3]----[NOT F_SpeedReduce_Zone2]----|
|   DB_Cobot.Speed_Limit := 50                            |
|----------------------------------------------------------|

|----[NOT F_SpeedReduce_Zone2]----[NOT F_SpeedReduce_Zone3]-|
|   DB_Cobot.Speed_Limit := 100   // Full Speed           |
|----------------------------------------------------------|

// NETWORK 3: Cobot Program Management
// Cobot รอ Part จาก Feeder → หยิบ → วางบน Shared Table
// มนุษย์หยิบจาก Shared Table → ประกอบ → วางกลับ
// Cobot หยิบชิ้นงานสำเร็จ → ส่งไป Test Station

|----[Shared_Table_Sensor_Part_Ready]----[Human_Busy: FALSE]----|
|   // Cobot ไปหยิบชิ้นงานจาก Table                           |
|   DB_Cobot.Program_Number := 2  // Program 2: Take from Table|
|   DB_Cobot.Start_Cmd := 1                                    |
|-------------------------------------------------------------|

|----[Cobot_Program_Done]----[Part_Quality_OK]-----------------|
|   // Cobot ส่งชิ้นงานต่อ                                    |
|   DB_Cobot.Program_Number := 3  // Program 3: Send to Test  |
|   DB_Cobot.Start_Cmd := 1                                   |
|-------------------------------------------------------------|
```

---

## Step 1673: Offline Programming (RoboDK)

### RoboDK สำหรับ Offline Programming

```python
# Python - RoboDK API สำหรับ Simulation และ Offline Programming
import robodk.robolink as rl
from robodk import robomath
import numpy as np

# เชื่อมต่อกับ RoboDK
RDK = rl.Robolink()

# ดึง Robot จาก Station
robot = RDK.Item('UR10e', rl.ITEM_TYPE_ROBOT)

# ดึง Frame References
world_frame = RDK.Item('World', rl.ITEM_TYPE_FRAME)
tool = RDK.Item('Gripper_2F85', rl.ITEM_TYPE_TOOL)

# ตั้งค่า Robot
robot.setActiveFrame(world_frame)
robot.setActiveTool(tool)
robot.setSpeed(200, 50)  # Speed 200mm/s, Accel 50mm/s²

# สร้าง Target Positions
def create_target(name, x, y, z, rx, ry, rz):
    """สร้าง Target Position ใน RoboDK"""
    pose = robomath.transl(x, y, z) * robomath.rotx(rx) * robomath.roty(ry) * robomath.rotz(rz)
    target = RDK.AddTarget(name, world_frame, robot)
    target.setPose(pose)
    target.setJoints(robot.SolveIK(pose))
    return target

# กำหนดตำแหน่งต่างๆ
home = create_target('Home', 0, -400, 600, 180, 0, 0)
pick_approach = create_target('Pick_Approach', 300, -200, 300, 180, 0, 0)
pick_pos = create_target('Pick', 300, -200, 100, 180, 0, 0)
place_approach = create_target('Place_Approach', -300, -200, 300, 180, 0, 0)
place_pos = create_target('Place', -300, -200, 150, 180, 0, 45)

# สร้าง Program
prog = RDK.AddProgram('PickAndPlace', robot)

prog.MoveJ(home)
prog.setSpeed(200, 80)
prog.MoveL(pick_approach)
prog.MoveL(pick_pos)
prog.RunInstruction('DOUT[1] = ON', prog.INSTRUCTION_INSERT_CODE)
prog.Pause(300)  # รอ 300ms
prog.MoveL(pick_approach)
prog.setSpeed(400, 120)
prog.MoveL(place_approach)
prog.setSpeed(150, 50)
prog.MoveL(place_pos)
prog.RunInstruction('DOUT[1] = OFF', prog.INSTRUCTION_INSERT_CODE)
prog.Pause(200)
prog.MoveL(place_approach)
prog.MoveJ(home)

# Simulate
robot.RunProgram(prog)

# Export เป็น UR Script
prog.MakeProgram('/output/pick_and_place.script', rl.RUNMODE_MAKE_ROBOTPROG)
print("โปรแกรมพร้อม Export ไปยัง Robot!")
```

---

## Step 1674-1680: Lab Project

### โปรเจกต์: HRC Station สำหรับ PCB Assembly

**โจทย์**: ออกแบบและเขียนโปรแกรมสถานี HRC สำหรับการประกอบ PCB

**Requirements**:
- Cobot UR5e หยิบ Components จาก Feeder Tray
- วางบน Shared Table
- มนุษย์ Solder และ Inspect
- Cobot หยิบ PCB สำเร็จ → ทดสอบใน Test Fixture

**Safety Requirements**:
- SICK nanoScan3 Pro Safety Scanner
- ความเร็ว 0-100% ตาม Zone
- Emergency Stop ทันทีเมื่อ Zone 1 ถูก Intrude

**Deliverables**:
1. HRC Station Layout Drawing
2. I/O List (PLC + Cobot)
3. Safety Risk Assessment
4. URScript Program
5. PLC Safety Program (PROFISAFE)
6. Simulation ใน RoboDK

---

## สรุป Part 84

ใน Part นี้เราได้เรียนรู้:
- ✅ Cobot vs Industrial Robot
- ✅ UR Series Programming (URScript)
- ✅ Force/Torque Controlled Assembly
- ✅ Safety Zones และ Speed Limitation
- ✅ PLC-Cobot PROFINET Communication
- ✅ FANUC CRX Programming
- ✅ ISO/TS 15066 Safety Requirements
- ✅ Offline Programming ด้วย RoboDK

---

## การนำทาง
- [← Part 83: AI/ML Integration กับ PLC](part83_ai_plc.md)
- [→ Part 85: AGV และ AMR Integration](part85_agv_amr.md)
- [📚 สารบัญหลัก](../README.md)
