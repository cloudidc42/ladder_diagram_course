# Part 78: Simulation และ Virtual Testing (Steps 1541-1560)

## การทดสอบ PLC Program ด้วย Simulation ก่อน Commissioning

---

### 🔗 Navigation
- [← Part 77: Advanced Safety System](part77_advanced_safety.md)
- [→ Part 79: PLC Project Management](part79_project_management.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: ทำไมต้อง Simulation

การนำ PLC Program ไปทดสอบกับเครื่องจักรจริงโดยตรงมีความเสี่ยงสูง:

```
ปัญหาถ้าไม่ใช้ Simulation:
  1. Bug ใน Program → เครื่องเสียหาย → ราคาแพง
  2. Logic ผิด → อุบัติเหตุ → บาดเจ็บ
  3. ใช้เวลา Commissioning นาน → ค่าใช้จ่ายสูง
  4. ต้องมี Equipment พร้อมก่อนจะทดสอบ

ประโยชน์ของ Simulation:
  1. ทดสอบได้ก่อนที่ Equipment จะมาถึง
  2. Bug Fix ในขั้นตอน Engineering (ถูกกว่า 10-100 เท่า)
  3. Operator Training โดยไม่เสี่ยง
  4. Test Scenarios ที่ยากทำจริง (เช่น Emergency)
  5. Virtual Commissioning ลดเวลา On-site 40-60%
```

---

## Step 1541: Factory I/O สำหรับ PLC Testing

### Factory I/O คืออะไร

Factory I/O เป็น Software Simulation ที่สร้าง 3D Virtual Factory เพื่อเชื่อมต่อกับ PLC จริงหรือ PLC จำลอง

```
Factory I/O Key Features:
  - 3D Visual Simulation
  - Interface กับ PLC จริง (ผ่าน Ethernet)
  - Support: Siemens S7, Allen-Bradley, Modbus, OPC DA/UA
  - Scenes: Conveyor, Sorting, Pick & Place, Bottling, etc.
  - ราคา: ~$300/year (ถูกมากเมื่อเทียบกับ HW)
  
การเชื่อมต่อ Factory I/O กับ PLC:
  
  Method A: PLCSIM (S7) → No Real Hardware
    Factory I/O ↔ TIA Portal PLCSIM
    
  Method B: Real PLC → Ethernet
    Factory I/O ↔ S7-1200/1500 (ผ่าน IP)
    → Address Map ต้องตรงกัน
    
  Method C: Modbus TCP
    Factory I/O ↔ Any PLC with Modbus TCP Server
```

**การตั้งค่า Factory I/O กับ S7-PLCSIM:**

```
Step 1: TIA Portal
  - สร้าง Project S7-1500 (CPU 1513)
  - Enable PLCSIM Advanced
  - Note down IP Address (ปกติ 192.168.0.1)

Step 2: Factory I/O
  - Open Scene (เช่น "Sorting Station")
  - File → Drivers → Siemens S7-PLCSIM Advanced
  - IP Address: 192.168.0.1
  - Connect → ดู Status "Connected"
  
Step 3: Map Tags
  - Factory I/O Tag → PLC Address
  - "Sensor A" → I0.0 (Input byte 0, bit 0)
  - "Conveyor 1" → Q0.0 (Output byte 0, bit 0)
  
Step 4: Run
  - เปิด Factory I/O Play (▶)
  - ดู 3D Simulation ทำงานพร้อมกับ PLC Logic
```

---

## Step 1542: SIMIT Simulation Platform

### Siemens SIMIT สำหรับ Plant Simulation

```
SIMIT คืออะไร:
  Siemens Simulation Platform สำหรับ Process Plant
  ใช้ใน Virtual Commissioning ของ S7-1500 + WinCC
  
SIMIT Capabilities:
  - Dynamic Process Simulation (P&ID Level)
  - Valve Simulation (Open/Close + Position)
  - Motor/Pump Simulation (Speed, Current)
  - Sensor Simulation (Temperature, Pressure, Level)
  - Batch Process Simulation
  
SIMIT Components:
  - SIMIT Simulation Framework (SSF): Core engine
  - SIMIT Unit: Plant area simulation blocks
  - SIMIT Virtual Controller (SVC): Software S7 PLC
  - SIMIT Coupling: Interface ไปยัง Real PLC (ผ่าน PROFINET/DP)
```

**SIMIT Simulation Block ตัวอย่าง:**

```
Motor Block (SIMIT):
  Inputs:
    RUN_CMD: BOOL (คำสั่ง Start จาก PLC)
    TRIP_CMD: BOOL (คำสั่ง Trip)
    
  Simulation Parameters:
    Start_Time: 3.5 s (เวลา Ramp Up ถึง Full Speed)
    Inertia: 5.0 kg·m² (การหมุนของ Rotor)
    Rated_Speed: 1450 RPM
    Rated_Current: 45 A
    
  Outputs (ส่งกลับไป PLC I/O):
    RUNNING_STATUS: BOOL
    SPEED_FB: REAL (RPM)
    CURRENT_FB: REAL (A)
    OVERLOAD: BOOL (จะ=TRUE ถ้า Current เกิน 1.25× ต่อเนื่อง > 10s)
    
Valve Block (SIMIT):
  Inputs:
    OPEN_CMD: BOOL
    CLOSE_CMD: BOOL
    
  Simulation:
    Stroke_Time: 15 s (เวลาเปิด/ปิดเต็ม)
    Type: On/Off หรือ Modulating
    
  Outputs:
    OPEN_LS: BOOL (เปิดสุด)
    CLOSE_LS: BOOL (ปิดสุด)
    POSITION: REAL (% ตำแหน่ง)
```

---

## Step 1543: OpenPLC บน Raspberry Pi

### PLC Software สำหรับ Education และ Prototyping

```
OpenPLC คืออะไร:
  Open Source PLC Runtime
  ทำงานบน: Windows, Linux, Raspberry Pi, Arduino
  Programming: IEC 61131-3 (LD, ST, FBD)
  Editor: OpenPLC Editor (Free)
  
ใช้ OpenPLC เพื่อ:
  1. เรียนรู้ PLC Programming โดยไม่ต้องซื้อ Hardware
  2. Prototype ระบบควบคุมราคาถูก
  3. Home Automation
  4. Research และ Education
  
Raspberry Pi 4 + OpenPLC:
  - Raspberry Pi 4B ($60)
  - GPIO: 40 pins (Digital I/O)
  - ต้องการ Buffer IC สำหรับ Industrial Voltage
  
Hardware Expansion:
  - Sequent Microsystems 8-Relay HAT: $30
  - ComFile Tech CFC2480H: Industrial Grade
  - Kunbus RevPi: DIN Rail Raspberry Pi
```

**OpenPLC Ladder Program ตัวอย่าง:**

```
=== OpenPLC Ladder: Basic Conveyor ===

Configuration:
  Input: GPIO Pin 7 (I0.0) → START Button
  Input: GPIO Pin 11 (I0.1) → STOP Button
  Output: GPIO Pin 13 (Q0.0) → Motor Relay

Rung 1: Start/Stop with Seal-in
|--[I0.0 START]--[/Q0.0 OFF]--+--( Q0.0 MOTOR )--|
|                              |
|--[Q0.0 RUNNING]-----------+|
|
|--[/I0.1 STOP]------------|  (เป็น NC Contact)

ทดสอบ: กด START → Motor ON
        กด STOP → Motor OFF
```

---

## Step 1544: Hardware-in-Loop (HIL) Testing

### HIL คืออะไร

```
Hardware In the Loop (HIL):
  ทดสอบด้วย Real Controller แต่ Plant เป็น Simulation
  
                ┌──────────────┐
  Real PLC ────→│    I/O       │
    (Hardware)  │  Interface   │
                └──────┬───────┘
                       │ Real Signals
                ┌──────▼───────┐
                │  Simulation  │
                │   Computer   │←── Plant Model
                └──────────────┘
  
Benefits:
  - ทดสอบ Controller Logic กับ Realistic Plant Behavior
  - Safety Testing (ทดสอบ Emergency โดยไม่เสี่ยง)
  - Performance Testing (เร็วกว่า Real Plant)
  - อาจทำซ้ำ Scenario เดิมได้ทุกเวลา (Reproducibility)
  
Industries Using HIL:
  Automotive: ECU Testing
  Aerospace: Flight Control Testing
  Power: Protection Relay Testing
  Chemical: DCS/PLC Testing
```

**HIL Setup สำหรับ Chemical Plant:**

```
Hardware:
  Real PLC: Siemens S7-1515
  I/O Interface: National Instruments DAQ Card (NI-6229)
  Simulation PC: High-performance PC + MATLAB Simulink
  
Analog Interface (Sensor Simulation → PLC Input):
  Simulink outputs → DAC (Digital to Analog) → PLC Analog Input
  เช่น: Temperature Model Output (°C) → 4-20mA → PLC AI
  
Digital Interface:
  Simulink outputs → Digital Out → PLC Digital Input
  เช่น: Pump Running Status → DO → PLC DI
  
Timing:
  HIL requires Real-time Simulation (1ms step size)
  MATLAB uses Real-Time Windows Target or xPC Target
  NI VeriStand เป็น Alternative ที่ใช้ง่ายกว่า
```

---

## Step 1545: Virtual Commissioning Workflow

### ขั้นตอน Virtual Commissioning ที่สมบูรณ์

```
Virtual Commissioning Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Week 1-2: Simulation Setup
  - สร้าง Plant Model ใน SIMIT/MATLAB
  - Map I/O ระหว่าง Simulation กับ PLC
  - Test Basic Communication
  
Week 3-4: Control Logic Development + Testing
  - เขียน PLC Program
  - ทดสอบ Normal Operation
  - Fix Bugs ที่พบ
  
Week 5-6: Abnormal Testing
  - Fault Injection (ทดสอบ Alarm, Trip)
  - Power Failure Test
  - Communication Loss Test
  - Emergency Shutdown Test
  
Week 7: Operator Training
  - HMI สร้างเสร็จ, ทดสอบกับ Simulation
  - Operator ฝึก Normal, Startup, Shutdown, Emergency
  
Week 8: Prepare for Real Commissioning
  - Document Test Results
  - Open Issues List
  - Plan On-site Activities
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Results: On-site Commissioning Time ลดลง 40-60%
```

---

## Step 1546: Simulation Fidelity Levels

### ระดับความละเอียดของ Simulation

```
Level 1: Simple State Machine
  - เหมาะสำหรับ: ทดสอบ Sequence Logic
  - Model: If Valve_Open_CMD → Valve_Status = Open (ทันที)
  - ไม่มี Dynamics: ไม่มีเวลา Ramp, ไม่มี Physics
  
Level 2: Simplified Dynamics
  - เหมาะสำหรับ: ทดสอบ Control Loops
  - Model: First-order Transfer Function (เช่น Temperature)
    τ × dT/dt + T = K × Input (where τ = Time Constant)
  - มี Time Delay, แต่ไม่ Complex Physics
  
Level 3: Full Physical Model
  - เหมาะสำหรับ: Virtual Commissioning สมบูรณ์
  - Model: Mass Balance, Energy Balance, Fluid Dynamics
  - ใช้เวลาสร้างมาก แต่ใกล้เคียง Reality
  - Tools: Aspen HYSYS, MATLAB Simulink, AVEVA Flexmim
```

**Level 2 Simulation ใน Python:**

```python
# Simple Tank Level Simulation (Level 2)
# ใช้ทดสอบ PLC Level Control Logic

import numpy as np
import time
import socket  # สำหรับส่งข้อมูลไปยัง PLC ผ่าน Modbus

class TankSimulation:
    def __init__(self):
        self.level = 50.0       # % เริ่มต้นที่ 50%
        self.area = 10.0        # m² พื้นที่หน้าตัดถัง
        self.max_level = 3.0    # m ความสูงสูงสุด
        self.inflow = 0.0       # m³/s อัตราไหลเข้า
        self.outflow = 0.0      # m³/s อัตราไหลออก
        self.dt = 0.1           # seconds step size
        
    def update(self, inlet_valve_pct, outlet_valve_pct):
        # Inflow ∝ Inlet Valve %
        self.inflow = inlet_valve_pct / 100.0 * 0.05  # Max 0.05 m³/s
        
        # Outflow ∝ Outlet Valve % × √(Level) (Bernoulli)
        level_m = self.level / 100.0 * self.max_level
        self.outflow = outlet_valve_pct / 100.0 * 0.03 * np.sqrt(max(0, level_m))
        
        # dL/dt = (Inflow - Outflow) / Area
        net_flow = self.inflow - self.outflow
        d_level = (net_flow / self.area) * self.dt  # m
        
        # แปลงเป็น %
        d_level_pct = d_level / self.max_level * 100.0
        self.level = np.clip(self.level + d_level_pct, 0.0, 100.0)
        
        return self.level

# Main Loop
sim = TankSimulation()
while True:
    # อ่าน Valve Commands จาก PLC (ผ่าน Modbus)
    # inlet_cmd = read_from_plc(register=0)
    # outlet_cmd = read_from_plc(register=1)
    inlet_cmd = 50.0    # ตัวอย่าง 50%
    outlet_cmd = 30.0   # ตัวอย่าง 30%
    
    # อัพเดต Simulation
    level = sim.update(inlet_cmd, outlet_cmd)
    
    # ส่ง Level กลับไปยัง PLC (ผ่าน Modbus)
    # write_to_plc(register=100, value=int(level * 100))  # ×100 เพื่อ Resolution
    print(f"Level: {level:.2f}%, Inflow: {sim.inflow:.4f}, Outflow: {sim.outflow:.4f}")
    
    time.sleep(sim.dt)
```

---

## Step 1547: Test Automation with Simulation

### การ Automate การทดสอบ

```
Test Automation ใน PLC:
  ปกติ Commissioning Testing ทำมือ (Manual)
  Test Automation = ใช้ Script รันชุดทดสอบอัตโนมัติ
  
Benefits:
  - รันทดสอบซ้ำหลายพันครั้ง (ทดสอบ Regression)
  - แน่ใจว่าทุก Scenario ถูกทดสอบ
  - Document ผลการทดสอบอัตโนมัติ
  - ลดเวลา Testing 70%
  
Tools:
  - TIA Portal OPC UA + Python Script
  - Beckhoff TwinCAT: Built-in Test Automation
  - Rockwell FactoryTalk: Test Manager
  - Codra Panorama Test Suite
  - Open Source: pymodbus + pytest
```

**Automated Test Script (Python + OPC UA):**

```python
# Automated PLC Test Script
# ทดสอบ Start/Stop Sequence อัตโนมัติ

from opcua import Client
import time
import unittest

class TestPumpStartStop(unittest.TestCase):
    
    def setUp(self):
        # เชื่อมต่อกับ PLC ผ่าน OPC UA
        self.client = Client("opc.tcp://192.168.1.100:4840")
        self.client.connect()
        
        # หา Nodes
        root = self.client.get_root_node()
        self.start_node = root.get_child(["0:Objects", "2:PLC", "2:START_BTN"])
        self.stop_node = root.get_child(["0:Objects", "2:PLC", "2:STOP_BTN"])
        self.running_node = root.get_child(["0:Objects", "2:PLC", "2:PUMP_RUNNING"])
        
    def test_pump_start(self):
        """ทดสอบว่าปั๊มเริ่มทำงานเมื่อกด Start"""
        # ตรวจสอบสถานะเริ่มต้น
        self.assertFalse(self.running_node.get_value(), "Pump should be stopped initially")
        
        # กด Start
        self.start_node.set_value(True)
        time.sleep(0.1)
        self.start_node.set_value(False)
        
        # รอให้ปั๊มเริ่ม (ปกติ 3 วินาที)
        time.sleep(3.5)
        
        # ตรวจสอบว่าเริ่มแล้ว
        self.assertTrue(self.running_node.get_value(), "Pump should be running after Start")
        
    def test_pump_stop(self):
        """ทดสอบว่าปั๊มหยุดเมื่อกด Stop"""
        # สมมติว่าปั๊มกำลังวิ่ง (จาก test_pump_start)
        
        # กด Stop
        self.stop_node.set_value(True)
        time.sleep(0.1)
        self.stop_node.set_value(False)
        
        time.sleep(1.0)
        
        # ตรวจสอบว่าหยุดแล้ว
        self.assertFalse(self.running_node.get_value(), "Pump should be stopped after Stop")
        
    def tearDown(self):
        self.client.disconnect()

if __name__ == '__main__':
    unittest.main(verbosity=2)
```

---

## Step 1548: Defect Injection Testing

### การทดสอบด้วยการสร้างความผิดพลาด

```
Defect Injection Testing:
  จงใจทำให้เกิด Fault เพื่อทดสอบว่าระบบตอบสนองอย่างไร
  
Test Cases:
  1. Sensor Open Circuit: ตัดสาย Sensor
     → ต้องได้ Alarm "Sensor Wire Break"
     
  2. Sensor Short Circuit: Short 4-20mA
     → ต้องได้ Alarm "Sensor Range Error"
     
  3. Network Disconnection: ถอด Ethernet Cable
     → ต้องได้ Alarm "Communication Loss"
     → Remote I/O ต้องทำงาน Safe State
     
  4. Power Failure: ตัดไฟ
     → ต้อง Restart อัตโนมัติ
     → ต้อง Resume ในสถานะ Safe
     
  5. Overload Pump: Block ทาง
     → Thermal Overload ต้อง Trip
     
  6. Valve Stuck: Block Actuator
     → ต้องได้ Alarm "Valve Position Error"
     
  7. Partial Stroke Test (PST) สำหรับ Safety Valve:
     → เปิด 50% → ตรวจสอบ Movement
     → สำคัญมากสำหรับ Safety System
```

---

## Step 1549-1560: Complete Virtual Factory Setup

### Step 1549: Setting Up Complete Test Environment

**เครื่องมือที่ใช้ในบทนี้:**

```
Complete Virtual Commissioning Stack:

1. Engineering Tools:
   TIA Portal V18 (PLC + HMI Programming)
   WinCC Advanced V18 (SCADA)
   
2. Simulation:
   SIMIT V10 (Plant Simulation)
   PLCSIM Advanced V5 (Virtual PLC)
   
3. Test Automation:
   Python + pymodbus + opcua
   pytest (Test Framework)
   
4. Documentation:
   Test Plan (Word/Excel)
   Test Report (Auto-generated by Script)
```

### Step 1550-1560: Lab Integration

**เชื่อมทุกส่วนเข้าด้วยกัน:**

```
PLCSIM Advanced ↔ SIMIT
     ↕                ↕
  TIA Portal      Process Simulation
  (Program Dev)   (Plant Behavior)
     ↕
WinCC Advanced
(HMI Testing)
     ↕
Test Script (Python)
(Automated Verification)
```

---

## Lab Project 78: Virtual Commissioning ของ Conveyor System

### โจทย์: ทำ Virtual Commissioning ของ Sorting Conveyor System

**ระบบ:** Conveyor พร้อม Color Sensor คัดแยกสี 3 สี (Red, Blue, Green)
- Sensor ตรวจสอบสีเมื่อสินค้าผ่าน
- Diverter 1: คัดสีแดงออก
- Diverter 2: คัดสีน้ำเงินออก
- สีเขียวไปตรงต่อ

**Software Stack:**
1. TIA Portal + S7-PLCSIM Advanced
2. Factory I/O (Sorting Scene)
3. Python Test Script

**งาน:**
1. เขียน PLC Program ใน TIA Portal (Ladder + SCL)
2. เชื่อม Factory I/O กับ PLCSIM Advanced
3. ทดสอบ Normal Operation
4. ทดสอบ Fault Scenarios (Sensor Failed, Conveyor Jammed)
5. เขียน Automated Test Script (Python)
6. สร้าง Test Report

**เกณฑ์ประเมิน:**
- คัดแยกสีถูกต้อง > 99% (30 คะแนน)
- Fault Handling ถูกต้องทุก Scenario (30 คะแนน)
- Test Script ครอบคลุม ≥ 80% ของ Requirements (25 คะแนน)
- Test Report สมบูรณ์ (15 คะแนน)

---

## สรุป Part 78

ใน Part นี้เราได้เรียนรู้:
- Factory I/O สำหรับ PLC Testing
- SIMIT สำหรับ Process Plant Simulation
- OpenPLC บน Raspberry Pi
- HIL (Hardware In the Loop) Testing
- Virtual Commissioning Workflow
- Simulation Fidelity Levels
- Test Automation ด้วย Python + OPC UA
- Defect Injection Testing

**ทักษะสำคัญ:**
- Virtual Commissioning Methodology
- OPC UA Client Programming
- Automated Test Script Development
- Risk Reduction through Simulation

---

### 🔗 Navigation
- [← Part 77: Advanced Safety System](part77_advanced_safety.md)
- [→ Part 79: PLC Project Management](part79_project_management.md)
- [📚 สารบัญหลัก](../README.md)
