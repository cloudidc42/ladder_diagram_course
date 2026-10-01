# Part 32: Interrupt Routines - รูทีนขัดจังหวะ
## Steps 621-640: การใช้งาน Interrupt ใน PLC Programming

**ระดับ:** Advanced Intermediate  
**เวลาที่ใช้:** 3-4 ชั่วโมง  
**ข้อกำหนดเบื้องต้น:** Part 29 (Structured Programming), Part 31 (Data Blocks)

---

## บทนำ

Interrupt Routine คือโปรแกรมที่ถูกเรียกทำงานทันทีเมื่อเกิดเหตุการณ์บางอย่าง โดยขัดจังหวะการทำงานปกติของ PLC เพื่อตอบสนองอย่างรวดเร็ว

เหตุการณ์ที่ทำให้เกิด Interrupt:
- สัญญาณ Digital Input เปลี่ยนแปลง (Hardware Interrupt)
- ถึงเวลาที่กำหนด (Time/Cyclic Interrupt)
- เกิดข้อผิดพลาดในระบบ (Diagnostic Interrupt)
- PLC เริ่มทำงาน (Startup Interrupt)

---

## Step 621: ประเภทของ Interrupt

### ภาพรวม Interrupt Types

```
┌─────────────────────────────────────────────────────────────┐
│                    PLC Interrupt Hierarchy                   │
│                                                             │
│  Priority  │ OB Type    │ Description      │ Response Time  │
│  ----------|------------|------------------|---------------  │
│  27 (Max)  │ OB 80-88   │ Error OBs        │ Immediate      │
│  26        │ OB 40-47   │ Hardware Interrupt│ < 1ms          │
│  12-15     │ OB 30-38   │ Cyclic Interrupt  │ Configurable   │
│  8 (Low)   │ OB 100     │ Startup          │ Once at start  │
│  1 (Min)   │ OB 1       │ Main Program     │ Scan cycle     │
│                                                             │
│  สูงกว่า = ขัดจังหวะต่ำกว่าได้                              │
└─────────────────────────────────────────────────────────────┘
```

### ลำดับความสำคัญ (Priority)

```
┌─────────────────────────────────────────────────────────────┐
│               Interrupt Priority Diagram                     │
│                                                             │
│  ┌──────────┐  Priority 26                                  │
│  │ OB40-47  │  Hardware Interrupt ────────────────┐        │
│  └──────────┘                                      ↓        │
│                                            ┌──────────┐    │
│  ┌──────────┐  Priority 12               │   CPU    │    │
│  │ OB30-38  │  Cyclic Interrupt ─────────→│  Running │    │
│  └──────────┘                             └──────────┘    │
│                                                      ↑      │
│  ┌──────────┐  Priority 1                            │      │
│  │   OB1    │  Main Cycle ───────────────────────────┘      │
│  └──────────┘                                               │
│                                                             │
│  หาก OB1 กำลังทำงาน และ OB40 ถูก trigger                  │
│  → OB1 หยุดชั่วคราว, OB40 ทำงาน, OB1 ต่อ                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 622: OB Types ใน Siemens S7

### OB1 - Main Program Organization Block

```
OB1 - Cyclic Execution (Priority 1)
┌─────────────────────────────────────────────────────┐
│ - ทำงานต่อเนื่องตลอดเวลา (Scan Cycle)               │
│ - เรียกใช้ทุกโปรแกรมหลัก                            │
│ - ค่า Default Scan Time: ปกติ 10-100ms               │
│ - Watchdog จะ Trip ถ้า Scan เกินค่าที่กำหนด          │
│                                                     │
│ เหมาะสำหรับ:                                        │
│   - โปรแกรมควบคุมหลัก                               │
│   - Logic ทั่วไป                                    │
│   - การอ่าน/เขียน I/O ปกติ                           │
└─────────────────────────────────────────────────────┘
```

### OB30-OB38 - Cyclic Interrupt OBs

```
Cyclic Interrupt Organization Blocks:
┌──────┬──────────────────────────────────────────────┐
│  OB  │ Default Cycle Time  │ Priority │ ใช้สำหรับ   │
├──────┼────────────────────┼──────────┼─────────────┤
│ OB30 │ 5 seconds          │ 7        │ Slow tasks  │
│ OB31 │ 2 seconds          │ 8        │ Medium slow │
│ OB32 │ 1 second           │ 9        │ 1 second    │
│ OB33 │ 500ms              │ 10       │ 0.5 second  │
│ OB34 │ 200ms              │ 11       │ 0.2 second  │
│ OB35 │ 100ms              │ 12       │ 100ms tasks │
│ OB36 │ 50ms               │ 13       │ 50ms tasks  │
│ OB37 │ 20ms               │ 14       │ 20ms tasks  │
│ OB38 │ 10ms               │ 15       │ 10ms tasks  │
└──────┴────────────────────┴──────────┴─────────────┘
```

### ตัวอย่าง OB35 (100ms Cyclic) - Temperature Monitoring

```
// OB35 - ทำงานทุก 100ms
// ใช้สำหรับตรวจสอบอุณหภูมิและบันทึกข้อมูล

Network 1: อ่านค่าอุณหภูมิทุก 100ms

  |                                              |
  |--[ALWAYS]------[FC_Read_Temperature]---------|
  |  (always on)    Execute Temperature Reading  |
  |                                              |

Network 2: ตรวจสอบค่าเกิน Limit

  |                                                    |
  |--[>=]--[NOT]---[SET]------------------------------|
  |  "Temp_AI"   Alarm_ACK   "Temp_Alarm_High"        |
  |  >= 90.0                                          |
  |                                                    |

Network 3: บันทึกข้อมูลทุก 10 วินาที (100ms × 100 cycles)

  |                                                    |
  |--[CTU]--------[R]-------[FC_Log_Data]-------------|
  |  Count100ms   Count100ms  Execute: Log to DB       |
  |  PV: 100      CU: T_Tick                          |
  |               Q → R input                         |
  |                                                    |
```

### OB40-OB47 - Hardware Interrupt OBs

```
Hardware Interrupt Organization Blocks:
┌──────┬─────────────────────────────────────────────┐
│  OB  │ Trigger Source   │ Priority │ Response Time  │
├──────┼──────────────────┼──────────┼───────────────┤
│ OB40 │ Digital Input 1  │ 16       │ < 1ms          │
│ OB41 │ Digital Input 2  │ 17       │ < 1ms          │
│ OB42 │ Digital Input 3  │ 18       │ < 1ms          │
│ OB43 │ Digital Input 4  │ 19       │ < 1ms          │
│ OB44 │ Digital Input 5  │ 20       │ < 1ms          │
│ OB45 │ Digital Input 6  │ 21       │ < 1ms          │
│ OB46 │ Digital Input 7  │ 22       │ < 1ms          │
│ OB47 │ Digital Input 8  │ 23       │ < 1ms          │
└──────┴──────────────────┴──────────┴───────────────┘
```

### OB80+ - Error Organization Blocks

```
Error OBs:
┌──────┬────────────────────────────────────────────┐
│  OB  │ Error Type               │ Priority        │
├──────┼──────────────────────────┼─────────────────┤
│ OB80 │ Cycle Time Error         │ 26              │
│ OB82 │ Diagnostic Interrupt     │ 26              │
│ OB83 │ Remove/Insert Module     │ 26              │
│ OB85 │ Program Error            │ 26              │
│ OB86 │ Rack Failure             │ 28              │
│ OB87 │ Communication Error      │ 26              │
│ OB121│ Programming Error        │ 27              │
│ OB122│ I/O Access Error (WD)    │ 27              │
└──────┴────────────────────────────────────────────┘
```

---

## Step 623: Hardware Interrupt - High-Speed Response

### การกำหนดค่า Hardware Interrupt ใน TIA Portal

```
1. เปิด Device Configuration
2. เลือก I/O Module ที่ต้องการ
3. ใน Properties → Hardware interrupts
4. กำหนด:
   - Input Channel: 0 (I0.0)
   - Trigger: Rising edge / Falling edge
   - Assign to OB: OB40
```

### ตัวอย่าง: Emergency Stop Hardware Interrupt

```
// OB40 - Hardware Interrupt สำหรับ Emergency Stop
// ทำงานทันทีเมื่อกด Emergency Stop Button

Network 1: ดับทุก Output เมื่อ E-Stop

  |                                                         |
  |--[ALWAYS]---------[FILL]-------------------------------|
  |                    BVAL: 0 (ดับทุก Bit)                 |
  |                    BLK: QB0                             |
  |                    N: 4 (4 bytes = 32 outputs)          |
  |                                                         |

Network 2: เปิด Emergency Alarm Output

  |                                                         |
  |--[ALWAYS]---------[SET]--------------------------------|
  |                    "Emergency_Alarm_Output" (Q100.0)    |
  |                                                         |

Network 3: บันทึกเหตุการณ์ E-Stop

  |                                                         |
  |--[ALWAYS]---------[RD_SYS_T]--------------------------|
  |                    OUT: "EStop_DB".Last_EStop_Time      |
  |                                                         |
  |--[ALWAYS]---------[INC]--------------------------------|
  |                    "EStop_DB".EStop_Count               |
  |                                                         |

Network 4: ส่งสัญญาณ E-Stop ไปยัง Drive systems

  |                                                         |
  |--[ALWAYS]---------[SET]--------------------------------|
  |                    "Drive1_EStop"                       |
  |                    "Drive2_EStop"                       |
  |                    "Drive3_EStop"                       |
  |                                                         |
```

### Hardware Interrupt Timing

```
การวิเคราะห์เวลาตอบสนอง:

┌─────────────────────────────────────────────────────────┐
│                   Response Time Analysis                 │
│                                                         │
│ Event Occurs (Edge on I0.0)                            │
│        │                                               │
│        ↓ < 1ms                                         │
│ CPU detects interrupt                                   │
│        │                                               │
│        ↓ < 0.5ms                                       │
│ Save context (current OB state)                         │
│        │                                               │
│        ↓ Immediate                                      │
│ Execute OB40                                            │
│        │                                               │
│        ↓ Program dependent                             │
│ OB40 completes                                         │
│        │                                               │
│        ↓                                               │
│ Restore context, resume OB1                             │
│                                                         │
│ Total: ขึ้นอยู่กับ OB40 size, ปกติ < 2ms              │
└─────────────────────────────────────────────────────────┘
```

---

## Step 624: Time Interrupt - Periodic Tasks

### ตัวอย่าง OB35 (100ms) - PID Loop Execution

```
// OB35 - 100ms Cyclic Interrupt
// ใช้รัน PID Control loop อย่างสม่ำเสมอ

Network 1: อ่านค่า Process Variable

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: IW256 (Analog Input)           |
  |                 OUT: MW100 (Raw AI Value)           |
  |                                                    |

Network 2: แปลง Raw AI เป็นหน่วยจริง

  |                                                    |
  |--[ALWAYS]------[SCALE]----------------------------|
  |                 IN: MW100                          |
  |                 LO_LIM: 0.0                        |
  |                 HI_LIM: 100.0                      |
  |                 OUT: "Process_Temp" (REAL)          |
  |                                                    |

Network 3: เรียกใช้ FB PID Controller

  |                                                    |
  |--[ALWAYS]------[FB_PID_Controller]----------------|
  |                 Setpoint: "Temp_Setpoint"          |
  |                 Process: "Process_Temp"            |
  |                 Kp: 2.5                            |
  |                 Ki: 0.1                            |
  |                 Kd: 0.05                           |
  |                 Output: "Heater_Output"            |
  |                                                    |

Network 4: เขียน Output ไปยัง Heater

  |                                                    |
  |--[ALWAYS]------[UNSCALE]--------------------------|
  |                 IN: "Heater_Output"               |
  |                 LO_LIM: 0.0                        |
  |                 HI_LIM: 100.0                      |
  |                 OUT: QW256 (Analog Output)         |
  |                                                    |
```

### OB32 (1 Second) - Batch Timer และ Counter

```
// OB32 - 1 Second Cyclic Interrupt
// ใช้นับเวลาการผลิตและอัพเดต Counter

Network 1: นับเวลาการผลิตที่กำลังทำงาน

  |                                                    |
  |--[ ]----------[INC]------------------------------|
  |  "Production_Running"  "Batch_DB".Run_Time_Secs   |
  |                                                    |

Network 2: แปลง Second เป็น HH:MM:SS

  |                                                    |
  |--[ALWAYS]------[DIV]------------------------------|
  |                 IN1: "Batch_DB".Run_Time_Secs      |
  |                 IN2: 3600                          |
  |                 OUT: "Display_Hours"               |
  |                                                    |
  |--[ALWAYS]------[MOD]------------------------------|
  |  (modulo)       IN1: "Batch_DB".Run_Time_Secs      |
  |                 IN2: 3600                          |
  |  [DIV]          → IN1: /60 OUT: "Display_Minutes"  |
  |                                                    |

Network 3: ตรวจสอบ Batch Time Limit

  |                                                    |
  |--[>=]---------[SET]------------------------------|
  |  Run_Time      "Batch_Timeout_Alarm"               |
  |  >= Max_Time                                       |
  |                                                    |
```

### OB30 (5 Second) - Energy Monitoring

```
// OB30 - 5 Second Cyclic Interrupt
// ใช้ตรวจสอบและบันทึกพลังงาน

Network 1: อ่านค่าพลังงานจาก Power Monitor

  |                                                    |
  |--[ALWAYS]------[MODBUS_RTU]------------------------|
  |                 Device: 1                          |
  |                 Function: 3 (Read Holding Reg)     |
  |                 Register: 40001                    |
  |                 Count: 2 (32-bit value)            |
  |                 Result: "Energy_kWh"               |
  |                                                    |

Network 2: คำนวณพลังงานที่ใช้ใน 5 วินาที

  |                                                    |
  |--[ALWAYS]------[SUB]------------------------------|
  |                 IN1: "Energy_kWh"                  |
  |                 IN2: "Last_Energy_kWh"             |
  |                 OUT: "Delta_Energy_5s"             |
  |                                                    |

Network 3: บันทึกลง DB

  |                                                    |
  |--[ALWAYS]------[MOVE]-----------------------------|
  |                 IN: "Energy_kWh"                   |
  |                 OUT: "Energy_Log_DB".Last_Reading  |
  |                                                    |
```

---

## Step 625: Startup OB - OB100

### OB100 - Cold Start Initialization

```
// OB100 - ทำงานครั้งเดียวเมื่อ PLC เริ่มต้น (Cold Start)
// ใช้สำหรับ Initialize ค่าเริ่มต้น

Network 1: Initialize Process Variables

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: 75.0                           |
  |                 OUT: "Process_DB".Temp_Setpoint    |
  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: 3.5                            |
  |                 OUT: "Process_DB".Pressure_SP      |
  |                                                    |

Network 2: Clear Alarm Bits

  |                                                    |
  |--[ALWAYS]------[FILL]------------------------------|
  |                 BVAL: 0                            |
  |                 BLK: "Alarm_DB".Alarm_Bits         |
  |                 N: 10 (10 bytes of alarms)         |
  |                                                    |

Network 3: บันทึกเวลา Startup

  |                                                    |
  |--[ALWAYS]------[RD_SYS_T]--------------------------|
  |                 OUT: "System_DB".Last_Startup_Time  |
  |                                                    |
  |--[ALWAYS]------[INC]------------------------------|
  |                 "System_DB".Startup_Count          |
  |                                                    |

Network 4: Set Default Recipe

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: 1 (Recipe #1 as default)       |
  |                 OUT: "Recipe_DB".Active_Recipe      |
  |                                                    |
```

---

## Step 626: Watchdog และ OB122

### OB122 - I/O Access Error

```
// OB122 - เรียกเมื่อเกิด I/O Error (Scan Timeout)
// ใช้จัดการเมื่อ Module ไม่ตอบสนอง

// ข้อมูลที่มีในตัวแปร OB122_MDL_ADDR:
// - ที่อยู่ของ Module ที่เกิดปัญหา
// - ประเภทของ Error

Network 1: บันทึก Module Address ที่เกิด Error

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: OB122_MDL_ADDR                 |
  |                 OUT: "Fault_DB".IO_Fault_Addr      |
  |                                                    |
  |--[ALWAYS]------[SET]------------------------------|
  |                 "IO_Fault_Alarm"                   |
  |                                                    |

Network 2: ตั้ง Output เป็น Safe State

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: 0                              |
  |                 OUT: QW0 (ดับ Output ทั้งหมด)     |
  |                                                    |

Network 3: แจ้งเตือนผ่าน Safety Output

  |                                                    |
  |--[ALWAYS]------[SET]------------------------------|
  |                 Q100.0 (Alarm Light)               |
  |                 Q100.1 (Alarm Buzzer)              |
  |                                                    |
```

### Watchdog Timer Setup

```
การกำหนดค่า Watchdog ใน TIA Portal:

1. CPU Properties → Cycle time
   ├── Maximum cycle time: 150ms (ค่า default)
   ├── Minimum cycle time: 0ms
   └── Watchdog time: 150ms (เมื่อเกินนี้ OB80 ถูกเรียก)

2. ถ้า OB80 ไม่มี → CPU ไปสู่ STOP mode
3. OB80 มี → สามารถตรวจสอบและ Continue ได้

// OB80 - Cycle Time Fault
Network 1: บันทึกเหตุการณ์

  |                                                    |
  |--[ALWAYS]------[INC]------------------------------|
  |                 "Watchdog_Trip_Count"              |
  |                                                    |
  |--[ALWAYS]------[RD_SYS_T]--------------------------|
  |                 OUT: "Last_Watchdog_Trip"          |
  |                                                    |

Network 2: ตรวจสอบว่าควร Continue หรือ STOP

  |                                                    |
  |--[>=]---------[STOP_CPU]--------------------------|
  |  Trip_Count >= 3   (หยุด CPU ถ้าเกิดบ่อยเกินไป)  |
  |                                                    |
```

---

## Step 627: Allen-Bradley Event Tasks

### Interrupt ใน Studio 5000 / Logix

```
Allen-Bradley ใช้ Tasks แทน OBs:

Task Types:
┌──────────────┬──────────────────────────────────────────┐
│ Task Type    │ Description                              │
├──────────────┼──────────────────────────────────────────┤
│ Continuous   │ ทำงานต่อเนื่อง (เหมือน OB1)             │
│ Periodic     │ ทำงานตามเวลา (เหมือน OB30-38)          │
│ Event        │ ทำงานตาม Event (เหมือน OB40-47)         │
└──────────────┴──────────────────────────────────────────┘
```

### Event Task Configuration

```
ใน Studio 5000:

1. Controller Properties → Tasks
2. New Task: "Emergency_Task"
   - Type: Event
   - Period: N/A
   - Priority: 1 (สูงสุด = 15 ต่ำสุด)

3. กำหนด Trigger:
   - Event Type: Consumed Tag (ผ่าน Tag ที่แชร์)
   - หรือ: Motion Event
   - หรือ: Module Event

// Ladder ใน Emergency_Task
Rung 1: [OTE] Emergency_Output        // เปิด Emergency Output
Rung 2: [OTE] Drive1_Inhibit          // หยุด Drive 1
Rung 3: [OTE] Drive2_Inhibit          // หยุด Drive 2
```

### AB Periodic Task

```
Periodic Task "Temperature_Control_Task"
Period: 100ms
Priority: 7

// รัน PID ทุก 100ms
Rung 1: [PIDE] Temperature_PID
  SP: 75.0
  PV: Temp_Sensor
  OUT: Heater_CV

// เทียบกับ Siemens OB35
```

---

## Step 628: ตัวอย่างการใช้งานจริง - High-Speed Part Counter

### ระบบนับชิ้นงานความเร็วสูง

```
โจทย์: สายการผลิตวิ่งที่ 500 ชิ้น/นาที
       ต้องนับชิ้นงานและตรวจสอบคุณภาพ
       
ปัญหาถ้าใช้ OB1:
- Scan Time: 20ms → เพียง 50 times/sec
- Conveyor: 500 ppm = 8.3 parts/sec
- พลาดได้ถ้า Scan เกิน 120ms!

วิธีแก้: ใช้ Hardware Interrupt (OB40)
- ตอบสนองทุก Part ทันที
- ไม่ขึ้นกับ Scan Time
```

### OB40 Program - Part Counter

```
// OB40 - Hardware Interrupt จาก Photo Sensor (I0.0)
// เรียกทุกครั้งที่ชิ้นงานผ่าน

Network 1: นับชิ้นงาน

  |                                                    |
  |--[ALWAYS]------[ADD]------------------------------|
  |                 IN1: "Counter_DB".Total_Count      |
  |                 IN2: 1                             |
  |                 OUT: "Counter_DB".Total_Count      |
  |                                                    |

Network 2: นับชิ้นงาน Shift ปัจจุบัน

  |                                                    |
  |--[ALWAYS]------[ADD]------------------------------|
  |                 IN1: "Counter_DB".Shift_Count      |
  |                 IN2: 1                             |
  |                 OUT: "Counter_DB".Shift_Count      |
  |                                                    |

Network 3: บันทึกเวลาผ่านของชิ้นงาน (สำหรับ Speed Calculation)

  |                                                    |
  |--[ALWAYS]------[RD_SRTM]--------------------------|
  |  (Read System Runtime - millisecond timer)         |
  |                 OUT: "Counter_DB".Last_Part_Time   |
  |                                                    |

Network 4: ตรวจสอบว่าถึง Batch Target หรือไม่

  |                                                    |
  |--[>=]---------[SET]------------------------------|
  |  Shift_Count   "Batch_Complete_Signal"             |
  |  >= Target_Count                                   |
  |                                                    |
```

### OB35 Program - Speed Calculation (100ms)

```
// OB35 - 100ms Cyclic
// คำนวณความเร็วสายพานจากข้อมูล Part Counter

Network 1: คำนวณ Parts ใน 100ms ที่ผ่านมา

  |                                                    |
  |--[ALWAYS]------[SUB]------------------------------|
  |                 IN1: "Counter_DB".Total_Count      |
  |                 IN2: "Counter_DB".Last_100ms_Count |
  |                 OUT: "Parts_in_100ms"              |
  |                                                    |

Network 2: คำนวณ Parts Per Minute

  |                                                    |
  |--[ALWAYS]------[MUL]------------------------------|
  |  (×600 เพื่อแปลง 100ms เป็น 1 minute)              |
  |                 IN1: "Parts_in_100ms"              |
  |                 IN2: 600                           |
  |                 OUT: "Current_PPM"                 |
  |                                                    |

Network 3: อัพเดต Last Count

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: "Counter_DB".Total_Count       |
  |                 OUT: "Counter_DB".Last_100ms_Count |
  |                                                    |
```

---

## Step 629: Diagnostic Interrupt - OB82

### การจัดการ Diagnostic Events

```
// OB82 - เรียกเมื่อ Module ส่ง Diagnostic Event
// เช่น: Wire Break, Overrange, Short Circuit

// ข้อมูลที่มีใน OB82:
// OB82_MDL_ADDR  - ที่อยู่ Module
// OB82_IO_FLAG   - 0=Input, 1=Output
// OB82_MDL_DEFECT - 1=Module มีปัญหา
// OB82_CH_DIFFS  - 0=ไม่มี channel event, 1=มี

Network 1: อ่านข้อมูล Module ที่เกิดปัญหา

  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: OB82_MDL_ADDR                  |
  |                 OUT: "Diag_DB".Fault_Module_Addr   |
  |                                                    |
  |--[ALWAYS]------[MOVE]------------------------------|
  |                 IN: OB82_IO_FLAG                   |
  |                 OUT: "Diag_DB".IO_Type             |
  |                                                    |

Network 2: Set Alarm Bit ตาม Module Address

  |                                                    |
  |--[=]----------[SET]------------------------------|
  |  MDL_ADDR      "Module_1_Alarm"                   |
  |  = 256                                            |
  |  (Slot 1 address)                                 |
  |                                                    |
  |--[=]----------[SET]------------------------------|
  |  MDL_ADDR      "Module_2_Alarm"                   |
  |  = 272                                            |
  |                                                    |

Network 3: บันทึก Diagnostic Log

  |                                                    |
  |--[ALWAYS]------[FC_Add_Diag_Log]------------------|
  |                 Module: OB82_MDL_ADDR              |
  |                 Error:  OB82_MDL_DEFECT            |
  |                                                    |
```

---

## Step 630: Mitsubishi Interrupt Programs

### Interrupt ใน MELSEC iQ-R / iQ-F

```
Mitsubishi Interrupt Types:
┌─────────────────────────────────────────────────────┐
│ Interrupt Type    │ Trigger         │ Equivalent     │
├─────────────────────────────────────────────────────┤
│ Fixed Scan INT    │ Fixed Time      │ Siemens OB35   │
│ Fixed Event INT   │ External Signal │ Siemens OB40   │
│ Internal INT      │ Counter/Timer   │ Siemens OB35   │
│ Refresh INT       │ I/O Refresh     │ Siemens OB40   │
└─────────────────────────────────────────────────────┘
```

### ตัวอย่าง Mitsubishi Interrupt Program

```
// Fixed Scan Interrupt - I28 (100ms)
// Configuration ใน GX Works3:
// Interrupt Pointer: I28
// Execution Interval: 100ms

[Interrupt Program I28]
// อ่านค่า Analog Input
BFMRD K0 K0 D100 K2    // อ่านจาก Special Module

// แปลงค่า
DIV D100 K100 D102     // D102 = Temperature in 0.1°C

// ตรวจสอบ Alarm
[>= D102 K900]         // ถ้า >= 90.0°C (×10)
 [SET M100]            // Set Alarm bit

IRET                   // กลับจาก Interrupt
```

### External Interrupt ใน Mitsubishi

```
// External Interrupt Configuration:
// X0 = Interrupt Input
// Interrupt Pointer: I0

// Ladder Program ใน I0:
[Interrupt Program I0]

// นับ Pulse
INC D200               // เพิ่ม Counter

// บันทึกเวลา
MOV SD0 D210           // SD0 = System Clock (ms)

// Check Target
[>= D200 D300]         // ถ้าถึง Target
 [SET M200]            // สัญญาณ "Target Reached"

IRET
```

---

## Step 631-635: Best Practices

### หลักการใช้ Interrupt อย่างถูกต้อง

```
DO:
✓ เก็บโปรแกรมใน Interrupt OB ให้สั้นที่สุด
✓ ใช้ Interrupt เฉพาะเมื่อจำเป็น (High-speed response)
✓ ตรวจสอบ Interrupt Load ไม่เกิน 20% ของ CPU time
✓ ใช้ Shared Data อย่างระมัดระวัง (Thread Safety)
✓ Always implement OB80/OB85/OB122 ใน Production systems

DON'T:
✗ อย่าเรียก Communication Functions ใน Hardware Interrupt
✗ อย่าใช้ Interrupt ทุกงาน (Main OB1 เร็วพอสำหรับงานปกติ)
✗ อย่า Block ใน Interrupt OB (Long loops, delays)
✗ อย่า Ignore Error OBs
```

### การวัด Interrupt Load

```
Network ใน OB1: ตรวจสอบ CPU Load

  |                                                    |
  |--[ALWAYS]------[RD_SINFO]--------------------------|
  |                 PROVIDEd by Siemens:               |
  |                 ├── Cycle Time: MW50               |
  |                 ├── Min Cycle: MW52                |
  |                 └── Max Cycle: MW54                |
  |                                                    |

// ถ้า Max Cycle > 80% ของ Watchdog Time
// แสดงว่า System Load สูงเกินไป
```

### Data Sharing Between OB Levels

```
// ปัญหา: OB35 และ OB1 เข้าถึง Data เดียวกัน
// อาจเกิด Data Corruption ถ้าไม่ระวัง

// วิธีแก้ 1: ใช้ DB Variable ที่ Atomic
// (INT, REAL, BOOL ขนาด <= 4 bytes เขียนครั้งเดียว)

// วิธีแก้ 2: ใช้ Disable/Enable Interrupt
DISI    // Disable Interrupts
// Critical Section
ENSI    // Enable Interrupts

// วิธีแก้ 3: ใช้ Double Buffer
// OB35 เขียนลง Buffer A
// OB1 อ่านจาก Buffer B
// สลับ Buffer ทุก Scan
```

---

## Step 636-640: Lab Projects

### Lab 1: High-Speed Filling System

**โจทย์:** ระบบเติมของเหลวความเร็วสูง
- ความเร็วสายพาน: 60 ขวด/นาที
- ต้องหยุดเติมเมื่อน้ำหนักถึง 500g ±2g
- ตอบสนองภายใน 50ms

**การกำหนดค่า:**
```
OB40: ตอบสนอง Load Cell Signal
  - Trigger: Rising edge I0.0 (Load cell threshold)
  - Action: ปิด Fill Valve ทันที

OB35 (50ms): ตรวจสอบน้ำหนัก
  - อ่านน้ำหนักทุก 50ms
  - คำนวณ Fill Rate
  - ปรับ Pre-close Point

OB1: Logic หลัก
  - State Machine ของ Filling Cycle
  - HMI Communication
  - Alarm Management
```

### Lab 2: Vibration Monitoring with Interrupt

**โจทย์:** ตรวจจับการสั่นสะเทือนผิดปกติ

```
OB40: High-Frequency Vibration Alert
  Trigger: I0.2 (Vibration Sensor Digital Output)
  - บันทึกเวลาที่เกิด
  - นับ Events ใน 1 วินาที
  - Set Alarm ถ้า Events > 100/second

OB38 (10ms): Sample Vibration Data
  - อ่าน Analog Vibration Sensor
  - เก็บ Peak Value
  - คำนวณ RMS

OB35 (100ms): Trend Analysis
  - เก็บ Historical Data
  - คำนวณ Trend
  - Alert ถ้า Trend เพิ่มขึ้น > 20%
```

### Lab 3: Synchronization ระหว่าง OBs

```
ออกแบบระบบที่:
1. OB40 รับ Trigger จากภายนอก (ทุก Part)
2. OB35 (50ms) คำนวณ Statistics
3. OB1 แสดงผลบน HMI

ต้องไม่มี Race Condition ระหว่าง OBs
วิธีการ: ใช้ Double Buffer
  Buffer A: OB40 เขียน
  Buffer B: OB35 อ่าน
  Swap signal ทุก 50ms
```

---

## สรุป Part 32

| OB Type | Priority | Trigger | ใช้สำหรับ |
|---------|----------|---------|-----------|
| OB1 | 1 | Cyclic | Main Program |
| OB30-38 | 7-15 | Time | Periodic Tasks |
| OB40-47 | 16-23 | Hardware | High-speed response |
| OB80 | 26 | Cycle Error | Watchdog Trip |
| OB82 | 26 | Diagnostic | Module Faults |
| OB100 | - | Startup | Initialization |
| OB122 | 27 | I/O Error | I/O Fault Handling |

---

## การนำทาง

- [← Part 31: Data Blocks](part31_data_blocks.md)
- [→ Part 33: Motion Control พื้นฐาน](part33_motion_control_basic.md)
- [↑ สารบัญหลัก](../README.md)

---

*Ladder Diagram PLC Programming Course - Advanced Intermediate Level*  
*Part 32 of 40 | Steps 621-640*
