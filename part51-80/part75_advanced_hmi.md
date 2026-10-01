# Part 75: Advanced HMI Design (Steps 1481-1500)

## การออกแบบ HMI ขั้นสูงตามมาตรฐาน ISA-101

---

### 🔗 Navigation
- [← Part 74: Building Automation](part74_building_automation.md)
- [→ Part 76: DCS กับ PLC Integration](part76_dcs_plc_integration.md)
- [📚 สารบัญหลัก](../README.md)

---

## บทนำ: ทำไม HMI ที่ดีจึงสำคัญ

การออกแบบ HMI ที่ไม่ดีเป็นสาเหตุของอุบัติเหตุอุตสาหกรรมหลายครั้ง รวมถึง:
- Texas City Refinery Explosion (2005)
- Bhopal Gas Disaster
- Three Mile Island Nuclear Accident

**อุบัติเหตุเหล่านี้มีจุดร่วม:** Operator ไม่สามารถตีความสถานการณ์ได้ถูกต้องเนื่องจาก HMI ที่ออกแบบมาไม่ดี (ข้อมูลมากเกินไป, Alarm มากเกินไป, ไม่มีระดับความสำคัญ)

### ISA-101 มาตรฐานการออกแบบ HMI

```
ISA-101 Human Machine Interface for Process Automation Systems
  - เผยแพร่ปี 2015 โดย ISA (International Society of Automation)
  - ครอบคลุม: Philosophy, Displays, Alarm, Management
  - เป้าหมาย: Situational Awareness + Rapid Response
  
หลักการสำคัญ:
  1. High Performance HMI Design
  2. Alarm Rationalization
  3. Consistent Visual Language
  4. Support for Operator Decision Making
```

---

## Step 1481: High Performance HMI Philosophy

### หลักการ High Performance HMI

```
เปรียบเทียบ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TRADITIONAL HMI          vs     HIGH PERFORMANCE HMI
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
สีสัน/เคลื่อนไหวมาก           Gray, minimal motion
แต่ละชิ้นส่วนสีแตกต่าง          Color used only for meaning
3D Pipe Graphics              2D Schematic
% Fill Indicators (colorful)  Numeric + Trend
Many Animated Objects         Critical items highlighted
Dark Background               Light (Gray) Background
"Pretty" Interface            Functional Interface
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ผลลัพธ์ High Performance HMI:
  - Operator สังเกตเห็น Abnormal ได้เร็วขึ้น
  - ตอบสนองต่อ Alarm ได้ถูกต้องขึ้น
  - ลด Human Error 30-50%
```

### Color Standards (ISA-101)

```
Color Meaning (Consistent across all screens):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Color      | Use Case
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Gray       | Normal state, Background, Pipes
Dark Blue  | In-service equipment (Running, Open)
White      | Text, Values in normal state
Yellow     | Warning, Abnormal but not critical
Red        | Alarm, Critical, Shutdown
Green      | Good, Confirmed OK (ใช้น้อย)
Cyan       | Selected, Active element
Magenta    | Maintenance Mode, Override
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ห้ามใช้สีในการบ่งชี้สถานะ Process ปกติ
(เช่น น้ำ = น้ำเงิน, ไอน้ำ = ขาว, น้ำมัน = เหลือง)
→ สีเหล่านี้ควรสงวนไว้สำหรับ Alarm
```

---

## Step 1482: HMI Screen Hierarchy

### Navigation Design สำหรับ Process Plant

```
Level 1: Overview (Entire Plant - 1 Screen)
  - สรุป Status ทุกพื้นที่
  - Production Rates, Major Alarms
  - Navigation ไปยัง Level 2
  
Level 2: Area (Process Unit - 1 Screen per Area)
  - แสดง Process Flow ของพื้นที่นั้น
  - Major Equipment Status
  - Key Process Values
  
Level 3: Unit (Individual Equipment)
  - รายละเอียดของ Equipment 1 ชิ้น
  - Control Parameters, Trends
  - Historical Data
  
Level 4: Detail (Component / Instrument)
  - Instrument Detail
  - Calibration Data
  - Maintenance History

Navigation Rules:
  1. จากทุกหน้า → Level 1 Overview (1 click)
  2. Breadcrumb แสดงว่าอยู่ที่ไหน
  3. Related screens accessible หน้า adjacent
  4. ไม่เกิน 3 clicks เพื่อไปหน้าใดก็ได้
```

---

## Step 1483: Alarm Rationalization on HMI

### การจัดการ Alarm อย่างมีประสิทธิภาพ

```
Alarm Flood: เมื่อมีเหตุการณ์ใหญ่ → Alarm ดังพร้อมกันหลายร้อยตัว
  → Operator ไม่รู้ว่า Alarm ไหนสำคัญ → ตอบสนองผิด

EEMUA 191 Guideline:
  Acceptable Rate: ≤ 1 Alarm / 10 min per Operator
  Manageable Rate: 6-12 Alarms / 10 min
  Over-loaded: > 12 Alarms / 10 min

ปัญหา HMI ที่ไม่ดี:
  - Alarms ไม่มีการ Prioritize (ทุกอย่างเหมือนกัน)
  - Alarm เด้งซ้ำบ่อย (Chattering)
  - Alarm ที่ไม่สามารถทำอะไรได้ (Nuisance)
  - Suppressed Alarms ไม่ถูก Review
```

**Alarm Priority Matrix:**

```
Alarm Priority:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Level | Color  | Consequence    | Response Time | Example
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1     | RED    | Life Threat/   | Immediate     | Reactor Overpressure
      |        | Plant Shutdown  | (< 1 min)     | Gas Leak
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2     | RED    | Major Plant    | < 5 min        | High Level
      |        | Disruption      |               | Motor Overload
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
3     | YELLOW | Minor Process  | < 30 min       | High Temperature
      |        | Disruption      |               | Low Level
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
4     | YELLOW | Maintenance    | Next Shift     | Instrument Deviation
      |        | Needed          |               | Minor Fault
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Siemens WinCC Alarm Configuration:**

```pascal
// Alarm Class Configuration (WinCC TIA Portal)
// Alarm Properties ที่สำคัญ:

AlarmProperties:
  Name: "TANK_HIGH_LEVEL"
  Message: "Tank 101 Level High"
  Class: "Error"      // Error / Warning / System
  Priority: 2         // 1=Highest
  
  // ISA-101 Fields
  Tag: "TI_101_LEVEL"
  Limit: 90.0         // 90% ค่าที่ Trigger
  Deadband: 2.0       // 2% Hysteresis
  
  // Response Guidance (สำคัญ!)
  Operator_Action: "Reduce Feed Pump Speed or Open Overflow Valve"
  Consequence: "Overflow to Containment, Product Loss"
  
  // Shelving (ซ่อนชั่วคราว - ต้อง Approve)
  Max_Shelf_Time: 480  // นาที (8 ชั่วโมง)
```

---

## Step 1484: Situational Awareness Design

### สร้าง HMI ที่ช่วย Operator เข้าใจสถานการณ์

```
Situational Awareness Components:
  1. Perception: เห็นและรับรู้ข้อมูล
  2. Comprehension: เข้าใจความหมาย
  3. Projection: คาดการณ์สิ่งที่จะเกิดขึ้น

HMI Design to Support SA:
  
  Perception:
  - High Contrast Elements สำหรับ Abnormal
  - Consistent Layout ทุกหน้า
  - Key Values ต้องมองเห็นทันที
  
  Comprehension:
  - Trend ให้เห็น Direction
  - Deviation Bar (แสดง % จาก Setpoint)
  - Color = Meaning (ไม่ใช่ Decoration)
  
  Projection:
  - Rate-of-change Display
  - Time-to-limit Calculation
  - Predictive Alarm (ก่อนถึง Limit)
```

**Deviation Bar ใน WinCC (SCL):**

```pascal
// สร้าง Deviation Bar แสดง % จาก Setpoint
// ใช้ใน Trend Display

FUNCTION FC_DeviationBar : REAL
VAR_INPUT
    PV : REAL;    // ค่าจริง
    SP : REAL;    // Setpoint
    Range_High : REAL;  // ค่าสูงสุด (สำหรับ scaling)
    Range_Low  : REAL;  // ค่าต่ำสุด
END_VAR
VAR
    Deviation_Pct : REAL;
    Range : REAL;
END_VAR

Range := Range_High - Range_Low;
IF Range > 0 THEN
    Deviation_Pct := (PV - SP) / Range * 100.0;
END_IF;

// Clamp to ±50% for display
FC_DeviationBar := LIMIT(-50.0, Deviation_Pct, 50.0);
// บน HMI: แถบแสดง Deviation จาก Center
//  -50%  ────────────── 0 (SP) ──────────────  +50%
//        ← Negative    (Normal)   Positive →
END_FUNCTION
```

---

## Step 1485: Faceplate Design

### Faceplate คืออะไร

```
Faceplate: Popup Window แสดง Detail ของ Equipment หนึ่งชิ้น
  - Double-click บน Equipment → Faceplate เปิด
  - ข้อมูลครบ: Status, Commands, Setpoints, Mode, Alarms
  - Standard Layout ทุก Equipment Type เหมือนกัน

Faceplate Content (Standard):
┌─────────────────────────────────────────────┐
│  [PUMP P-101]           [ALARM: 2 Active]   │
│                                             │
│  Mode: [AUTO] [MANUAL] [MAINT]              │
│                                             │
│  Status: ■ RUNNING                          │
│  Speed: ████████░░ 75%     [Auto: 72%]     │
│                                             │
│  Current: 45.2 A           Max: 55 A        │
│  Temperature: 62°C         Max: 80°C        │
│  Vibration: 3.2 mm/s       Max: 7.1 mm/s   │
│                                             │
│  [START] [STOP] [RESET]                     │
│                                             │
│  Trend: [1HR] [8HR] [24HR] [1WK]           │
└─────────────────────────────────────────────┘
```

**TIA Portal VB Script สำหรับ Faceplate:**

```vb
' Faceplate Popup Control
' เมื่อ Double-click Equipment บน Overview Screen

Sub ShowFaceplate(EquipmentTag As String, EquipmentType As String)
    Dim FaceplateScreen As String
    
    ' เลือก Faceplate Template ตามประเภท Equipment
    Select Case EquipmentType
        Case "PUMP"
            FaceplateScreen = "FP_Pump_Standard"
        Case "VALVE"
            FaceplateScreen = "FP_Valve_Standard"
        Case "MOTOR"
            FaceplateScreen = "FP_Motor_Standard"
        Case "TANK"
            FaceplateScreen = "FP_Tank_Standard"
        Case Else
            FaceplateScreen = "FP_Generic"
    End Select
    
    ' ส่ง Tag Name ไปยัง Faceplate
    HmiRuntime.Tags("FP_ActiveTag") = EquipmentTag
    
    ' เปิด Faceplate Window
    OpenFaceplateWindow(FaceplateScreen, EquipmentTag)
End Sub
```

---

## Step 1486: Trend Display Design

### การออกแบบ Trend ที่ดี

```
Trend Best Practices:
  1. Y-Axis Scale: ต้องใช้ Engineering Range (ไม่ใช่ Auto-scale ตาม Data)
     ตัวอย่าง: Temperature Transmitter 0-200°C → Y-axis 0-200
     (ไม่ใช่ Auto-scale ทำให้ 85-90°C ดูเหมือนผันผวนมาก)
     
  2. Multiple Pens: รวม Related Variables ไว้ด้วยกัน
     - Temperature SP, PV, OP (Valve) ใน Trend เดียว
     
  3. Time Axis: คงที่ (ไม่ใช่ Auto)
     Historical: 8 ชั่วโมง, 24 ชั่วโมง, 1 สัปดาห์
     
  4. Annotation: สามารถ Mark เหตุการณ์บน Trend ได้
  
  5. Export: Export เป็น CSV สำหรับ Analysis

Trend Pen Color Standards:
  PV (Process Variable): White หรือ Cyan (สว่าง)
  SP (Setpoint): Yellow หรือ Orange (มองเห็นง่าย)
  OP (Output/CV): Green หรือ Blue-Green
  High Alarm Limit: Red Dashed Line
  Low Alarm Limit: Red Dashed Line
```

---

## Step 1487: Responsive Layout Design

### รองรับหลาย Screen Resolution

```
Screen Resolutions ใน Industry:
  1920×1080 (Full HD): ส่วนใหญ่ใช้กัน
  2560×1440 (2K): Control Room หลายจอ
  3840×2160 (4K): Video Wall
  1280×800: Embedded Panel (เช่น Siemens TP1500)
  
Design Principles:
  1. Design สำหรับ 1920×1080 เป็น Base
  2. สำหรับ Embedded Panel: ลด Detail, ใหญ่ขึ้น
  3. Touch Screen: Touch Target ≥ 10mm × 10mm
  4. Critical Values: Font Size ≥ 14pt
  5. Status Indicators: ≥ 15×15 pixels
```

---

## Step 1488-1500: Complete HMI Design Guide

### Step 1488: Navigation Design

```
Navigation Design Pattern:
  
  Header Bar (constant on every screen):
  ┌─────────────────────────────────────────────────┐
  │ [LOGO] [Plant Name] │ [Active Alarms: 3]        │
  │        Overview ▼   │ [User: John] [Time: 14:32]│
  └─────────────────────────────────────────────────┘
  
  Left Sidebar (Navigation Tree):
  ┌────────────┐
  │ ▣ Overview │
  │ ▶ Area 1   │
  │   ▶ Unit 1 │
  │   ▶ Unit 2 │
  │ ▶ Area 2   │
  │ ▶ Trending │
  │ ▶ Reports  │
  └────────────┘
```

### Step 1489: Custom Graphics

**สร้าง Custom Symbol (Siemens WinCC):**

```
Symbol Creation Best Practices:
  1. ใช้ Vector Graphics (ไม่ใช่ Bitmap)
  2. Layer แยกส่วน: Static, Dynamic State, Alarm
  3. Consistent Size: Pump = 40×40px, Valve = 20×40px
  4. State Colors: Running=DarkBlue, Stopped=Gray, Fault=Red
  
Pump Symbol (Vector):
  - วงกลม = Pump Body
  - สามเหลี่ยม = Impeller Direction
  - State Indicator (Green/Gray/Red dot)
  - Text Tag Above
  
Standard Symbols:
  Pump: ─┤►├─
  Motor: M (วงกลม)
  Valve: ─▶◀─ (ปิด) หรือ ───── (เปิด)
  Level Sensor: ┤├
  Flow Meter: ◉
  Temperature: T (ในวงกลม)
  Pressure: P (ในวงกลม)
```

### Step 1490: Dark Mode Implementation

```
Dark Mode Design:
  Background: #1A1A1A (ไม่ใช่ดำสนิท)
  Grid Lines: #333333
  Normal Text: #E0E0E0
  
  ข้อดี:
  - ลด Eye Strain สำหรับ 24-hour shifts
  - Better contrast สำหรับ Alarm colors
  - Consumes less power (OLED)
  
  ข้อเสีย:
  - อ่านยากบางคน
  - อาจสับสน Normal vs Abnormal
  
  แนะนำ: ให้ Operator เลือกได้ (Light/Dark)
```

### Step 1491-1500: Advanced HMI Projects

**Multi-touch Gesture Support:**

```
Touch Gestures สำหรับ HMI:
  Tap:           เลือก / กด Button
  Double-tap:    เปิด Detail / Faceplate
  Long press:    Context Menu
  Pinch/Zoom:    ขยาย/ย่อ Trend หรือ Diagram
  Swipe:         เลื่อนไปหน้าถัดไป / ย้อนกลับ
  Two-finger tap: Secondary Selection
  
สำหรับ Critical Actions (Start, Stop, Override):
  ต้องใช้ Multi-step Confirmation:
  1. กดครั้งแรก → Highlight
  2. Popup ยืนยัน "กด OK เพื่อยืนยัน START Pump 101"
  3. กด OK ภายใน 5 วินาที → Execute
  4. ไม่กด → Cancel อัตโนมัติ
  
  ป้องกัน Accidental Activation
```

---

## Lab Project 75: High Performance HMI Design

### โจทย์: Redesign HMI สำหรับ Chemical Plant ตามมาตรฐาน ISA-101

**สถานะปัจจุบัน (ก่อน Redesign):**
- Traditional HMI: ฉูดฉาด มีสี Gradient, Animation มาก
- Alarm Rate: 45 Alarms / 10 min (over-loaded)
- Operator Response Time: เฉลี่ย 4.5 นาที/alarm
- Incident Rate: 2 incidents/year จาก HMI issues

**เป้าหมาย (หลัง Redesign):**
- Alarm Rate: < 6 Alarms / 10 min
- Operator Response Time: < 2 นาที/alarm
- Incident Rate: ลดลง 50%

**งานที่ต้องทำ:**
1. Audit Current Alarms (จำแนก Priority 1-4)
2. Design Level 1 Overview Screen
3. Design Standard Faceplate สำหรับ Pump, Valve, Tank
4. Implement Deviation Bar Display
5. Create Alarm Summary + Suppression Management

**Deliverables:**
1. HMI Philosophy Document
2. Screen Layout Sketches (5 Screens)
3. Alarm Rationalization Database (Spreadsheet)
4. Before/After Comparison Report

**เกณฑ์ประเมิน:**
- Color Coding ถูกต้องตาม ISA-101 (25 คะแนน)
- Navigation < 3 Clicks (20 คะแนน)
- Alarm Priority Assigned ทุก Alarm (25 คะแนน)
- Faceplate สมบูรณ์ครบทุก Mode (30 คะแนน)

---

## สรุป Part 75

ใน Part นี้เราได้เรียนรู้:
- ISA-101 HMI Design Philosophy
- High Performance HMI vs Traditional HMI
- Color Standards สำหรับ Industrial HMI
- Alarm Rationalization (Priority, Deadband, Operator Guidance)
- Situational Awareness Support
- Faceplate Design Standards
- Trend Display Best Practices
- Navigation Design

**ทักษะสำคัญ:**
- ISA-101 Standard Application
- Alarm Philosophy Design
- Human Factors Engineering
- WinCC TIA Portal Advanced Features

---

### 🔗 Navigation
- [← Part 74: Building Automation](part74_building_automation.md)
- [→ Part 76: DCS กับ PLC Integration](part76_dcs_plc_integration.md)
- [📚 สารบัญหลัก](../README.md)
