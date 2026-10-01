# Part 29: Structured Programming (Steps 561-580)

## ภาพรวม
เมื่อโปรแกรม PLC ขยายใหญ่ขึ้น การเขียนทุกอย่างไว้ใน OB1 เดียวจะทำให้ยากต่อการบำรุงรักษาและ Debug มาก Structured Programming คือแนวทางการจัดระเบียบโค้ดให้เป็นหมวดหมู่ แยกส่วน ใช้ซ้ำได้ และแก้ไขได้ง่าย บทนี้จะครอบคลุมตั้งแต่แนวคิด Organization Block, Function, Function Block ของ Siemens ไปจนถึง Routine/Subroutine ของ Allen-Bradley และ Mitsubishi

---

## Step 561: Program Organization แนวคิดพื้นฐาน

```
┌──────────────────────────────────────────────────────────┐
│           Why Structured Programming?                    │
│                                                          │
│  โปรแกรมที่ไม่ Structured:                              │
│  ┌────────────────────────────────────────────────┐     │
│  │  OB1 (5,000 Rungs)                            │     │
│  │  ├── Motor Control (Rung 1-500)               │     │
│  │  ├── Analog Scaling (Rung 501-800)            │     │
│  │  ├── PID (Rung 801-1200)                      │     │
│  │  ├── Conveyor Sequencer (Rung 1201-2000)      │     │
│  │  ├── Alarms (Rung 2001-3000)                  │     │
│  │  └── HMI Communication (Rung 3001-5000)       │     │
│  └────────────────────────────────────────────────┘     │
│  ปัญหา: หาจุดที่ต้องแก้ยาก, เปลี่ยนหนึ่งจุดกระทบอีก   │
│                                                          │
│  โปรแกรมที่ Structured:                                  │
│  ┌────────────────────────────────────────────────┐     │
│  │  OB1 (Main)                                   │     │
│  │  ├── FC1: Motor_Control      (เรียกใช้ FB10)   │     │
│  │  ├── FC2: Analog_Scaling     (เรียกใช้ FB20)   │     │
│  │  ├── FC3: PID_Control        (เรียกใช้ FB30)   │     │
│  │  ├── FC4: Conveyor_Sequencer (เรียกใช้ FB40)   │     │
│  │  ├── FC5: Alarm_Manager                        │     │
│  │  └── FC6: Communication                        │     │
│  └────────────────────────────────────────────────┘     │
│  ข้อดี: แก้ไขส่วนใดส่วนหนึ่งโดยไม่กระทบส่วนอื่น      │
└──────────────────────────────────────────────────────────┘
```

---

## Step 562: OB/FC/FB ใน Siemens TIA Portal

```
┌──────────────────────────────────────────────────────────┐
│           Siemens S7 Program Building Blocks             │
│                                                          │
│  OB (Organization Block):                               │
│  - เรียกโดย Operating System โดยอัตโนมัติ                │
│  - OB1: Main (ทุก Scan)                                 │
│  - OB30: Cyclic Interrupt (ทุก X ms)                    │
│  - OB35: Cyclic Interrupt (ทุก X ms)                    │
│  - OB82: Diagnostic Alarm                               │
│  - OB100: Startup (เมื่อ PLC เริ่ม)                    │
│  - OB121: Programming Error                             │
│                                                          │
│  FC (Function):                                         │
│  - ไม่มี Memory ของตัวเอง (Stateless)                   │
│  - Input/Output ผ่าน Parameters เท่านั้น                │
│  - ผลลัพธ์ขึ้นอยู่กับ Input เท่านั้น                   │
│  - ใช้สำหรับ: Calculation, Conversion, Utility         │
│  - เรียกได้หลายที่ ไม่ต้องการ Instance                 │
│                                                          │
│  FB (Function Block):                                   │
│  - มี Instance Data Block (IDB) เก็บ Static Data       │
│  - จำค่าได้ระหว่าง Scan (Stateful)                     │
│  - Input/Output/InOut/Static ใน Interface              │
│  - ใช้สำหรับ: Timer, Counter, PID, Motor Control       │
│  - เรียกแต่ละครั้งต้องระบุ IDB (Instance)              │
└──────────────────────────────────────────────────────────┘
```

---

## Step 563: FC ตัวอย่าง - Scaling Function

```
// FC10: Scale_AI
// แปลง Raw Analog → Engineering Value
// ไม่มี Static Memory (เป็น Stateless)

// FC10 Interface:
// IN:
//   Raw_Value (INT): Raw ADC count
//   Raw_Min (INT): 0
//   Raw_Max (INT): 27648
//   Eng_Min (REAL): Engineering Minimum
//   Eng_Max (REAL): Engineering Maximum
// OUT:
//   Eng_Value (REAL): Scaled Engineering Value
// RETURN: VOID (ไม่มี Return Value)

// FC10 Body (SCL หรือ Ladder):
// SCL:
#Eng_Value := #Eng_Min + 
              (INT_TO_REAL(#Raw_Value - #Raw_Min) / 
               INT_TO_REAL(#Raw_Max - #Raw_Min)) * 
              (#Eng_Max - #Eng_Min);

// Ladder (ใน FC10):
NETWORK 1: // Calculate Scaling
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  // Step 1: Raw - Raw_Min                          │
    │  SUB: #Raw_Value - #Raw_Min → Temp_A (INT)        │
    │  INT_TO_REAL: Temp_A → Temp_A_Real                │
    │                                                    │
    │  // Step 2: Raw_Max - Raw_Min                      │
    │  SUB: #Raw_Max - #Raw_Min → Temp_B (INT)          │
    │  INT_TO_REAL: Temp_B → Temp_B_Real                │
    │                                                    │
    │  // Step 3: Divide                                 │
    │  DIV: Temp_A_Real / Temp_B_Real → Ratio           │
    │                                                    │
    │  // Step 4: Scale to Engineering Range             │
    │  MUL: Ratio × (#Eng_Max - #Eng_Min) → Span        │
    │  ADD: Span + #Eng_Min → #Eng_Value                │
    └───────────────────────────────────────────────────┘

// เรียกใช้ FC10 ใน OB1:
NETWORK 1: // Scale Temperature
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FC10 "Scale_AI"                                  │
    │  Raw_Value := IW64       // AI Module              │
    │  Raw_Min   := 0                                   │
    │  Raw_Max   := 27648                               │
    │  Eng_Min   := 0.0        // 0°C                   │
    │  Eng_Max   := 200.0      // 200°C                 │
    │  Eng_Value → Temperature_Actual                   │
    └───────────────────────────────────────────────────┘

NETWORK 2: // Scale Pressure
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FC10 "Scale_AI"                                  │
    │  Raw_Value := IW66       // AI Module Channel 1    │
    │  Raw_Min   := 0                                   │
    │  Raw_Max   := 27648                               │
    │  Eng_Min   := 0.0        // 0 Bar                 │
    │  Eng_Max   := 10.0       // 10 Bar                │
    │  Eng_Value → Pressure_Actual                      │
    └───────────────────────────────────────────────────┘
```

---

## Step 564: FB ตัวอย่าง - Motor Control

```
// FB10: Motor_Control
// ควบคุม Motor พร้อม Run Hours และ Start Count
// มี Static Data (Stateful)

// FB10 Interface:
// IN:
//   Start_Cmd (BOOL): Start Command
//   Stop_Cmd (BOOL): Stop Command
//   Fault_Reset (BOOL): Reset Fault
//   OL_Trip (BOOL): Overload Trip Input
// OUT:
//   Motor_Running (BOOL): Motor Running Status
//   Motor_Faulted (BOOL): Fault Status
// IN_OUT: ไม่มี
// STATIC:
//   Run_Hours_Accum (REAL): Accumulated Run Hours
//   Start_Counter (DINT): Total Start Count
//   Last_Start_Time (DATE_AND_TIME): Last Start Timestamp
//   Run_Timer (IEC_TIMER): Running Time Tracker
//   Latch_Run (BOOL): Internal Run Latch

// FB10 Body:
NETWORK 1: // Set Run Latch on Start Command
    ┌──[#Start_Cmd]──[#Motor_Faulted]/────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(S)─────────┐
                                          #Latch_Run      │
                                                          └┘

NETWORK 2: // Reset Latch on Stop or Fault
    ┌──[#Stop_Cmd]────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(R)─────────┐
                                          #Latch_Run      │
                                                          └┘
    ┌──[#OL_Trip]─────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────(R)─────────┐
       │                                  #Latch_Run      │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────(S)─────────┐
                                          #Motor_Faulted  │
                                                          └┘

NETWORK 3: // Output Motor Running
    ┌──[#Latch_Run]───────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(=)─────────┐
                                       #Motor_Running     │
                                                          └┘

NETWORK 4: // Count Starts
    ┌──[#Start_Cmd]──[NOT #Motor_Running]─────────────────┐
    │  Rising Edge เมื่อ Start ถูกกด และ Motor ไม่ได้วิ่ง │
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────(INC #Start_Counter)

NETWORK 5: // Accumulate Run Hours
    ┌──[#Latch_Run]───────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──(TON #Run_Timer PT: T#1H)──[#Run_Timer.Q]────────┐
                                              ┌(INC...)────┐│
                                              └────────────┘│
                                              (ADD 1.0 to #Run_Hours_Accum)

NETWORK 6: // Reset Fault
    ┌──[#Fault_Reset]──[NOT #OL_Trip]─────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(R)─────────┐
                                       #Motor_Faulted     │
                                                          └┘

// เรียกใช้ FB10 ใน OB1:
NETWORK 1: // Motor 1 Control
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB10 "Motor_Control" [IDB: Motor1_Data]          │
    │  Start_Cmd    := I0.0   // Start Button           │
    │  Stop_Cmd     := I0.1   // Stop Button            │
    │  Fault_Reset  := I0.2   // Reset Button           │
    │  OL_Trip      := I0.3   // OL Relay NC            │
    │  Motor_Running → Q0.0   // Contactor Output       │
    │  Motor_Faulted → Q0.1   // Fault Light            │
    └───────────────────────────────────────────────────┘

NETWORK 2: // Motor 2 Control (Same FB, Different IDB)
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB10 "Motor_Control" [IDB: Motor2_Data]          │
    │  Start_Cmd    := I1.0                             │
    │  Stop_Cmd     := I1.1                             │
    │  Fault_Reset  := I1.2                             │
    │  OL_Trip      := I1.3                             │
    │  Motor_Running → Q1.0                             │
    │  Motor_Faulted → Q1.1                             │
    └───────────────────────────────────────────────────┘
```

---

## Step 565: Parameter Passing (IN, OUT, IN_OUT)

```
┌──────────────────────────────────────────────────────────┐
│           Parameter Types in Siemens S7                  │
│                                                          │
│  IN Parameters:                                         │
│  - อ่านได้ภายใน FB/FC                                   │
│  - เขียนไม่ได้                                          │
│  - Pass By Value (Copy ค่าเข้ามา)                       │
│  - ตัวอย่าง: Speed_Setpoint, Enable_Command             │
│                                                          │
│  OUT Parameters:                                        │
│  - เขียนได้ภายใน FB/FC                                  │
│  - อ่านไม่ได้ภายใน (ไม่แนะนำให้อ่าน)                   │
│  - ส่งค่ากลับไปยัง Caller                              │
│  - ตัวอย่าง: Motor_Running, Fault_Status                │
│                                                          │
│  IN_OUT Parameters:                                     │
│  - อ่านและเขียนได้ภายใน                                 │
│  - Pass By Reference (ชี้ไปที่ Variable ต้นทาง)         │
│  - ตัวอย่าง: Counter_Value, Accumulator                 │
│                                                          │
│  STATIC (เฉพาะ FB):                                     │
│  - เก็บค่าระหว่าง Scan                                  │
│  - เก็บใน Instance Data Block                          │
│  - ตัวอย่าง: Timer, Counter, Accumulated Value          │
│                                                          │
│  TEMP:                                                  │
│  - Local Variable ชั่วคราว                              │
│  - ใช้เฉพาะใน Scan นั้น ไม่เก็บค่า                     │
│  - ตัวอย่าง: Intermediate Calculation Result            │
└──────────────────────────────────────────────────────────┘
```

---

## Step 566: Local Variables

```
// Siemens - Local Variables ใน FC/FB

// ใน FC:
// IN, OUT, IN_OUT: Parameters
// TEMP: Local temp (ใน Scan นั้นเท่านั้น)
// ไม่มี STATIC (FC ไม่มี Memory)

// ใน FB:
// IN, OUT, IN_OUT, STATIC, TEMP: ทั้งหมด
// STATIC: เก็บใน IDB (ข้ามระหว่าง Scan)

// ตัวอย่าง: FB Calculation with Local Temp Variables
// #TEMP_A := #Raw_Value - #Offset; // เก็บเฉพาะ Scan นี้
// #Eng_Value := #TEMP_A * #Scale;   // ใช้ค่าชั่วคราว

// Allen-Bradley - Local Variables
// In Studio 5000:
// Local Tags เหมือน Temp ของ Siemens
// ประกาศใน Local Tag Section ของ Routine

// Mitsubishi - Local Variables  
// โปรแกรม/Subroutine ใช้ GX Works3
// Local Variable = ประกาศใน Function Block

// ทำไม Local Variables สำคัญ:
// 1. ไม่ปนกับ Global Variable
// 2. ลด Global Memory การใช้งาน
// 3. ชัดเจนว่าตัวแปรใช้เฉพาะที่
// 4. Thread Safety ใน Multi-task Systems
```

---

## Step 567: Routine/Subroutine ใน Allen-Bradley

```
// Allen-Bradley ControlLogix Program Structure

// ControlLogix Hierarchy:
// Controller
// └── Tasks (Continuous, Periodic, Event)
//     └── Programs
//         └── Routines (Ladder, FBD, SFC, ST)
//             └── Rungs

// Main Task: Continuous (ทำงานตลอดเวลา)
// Periodic Task: ทุก X ms
// Event Task: เรียกเมื่อเกิด Event

// Routine Types:
// Main Routine: เรียกอัตโนมัติโดย Program
// Subroutine: เรียกด้วย JSR (Jump to Subroutine)
// Fault Routine: เรียกเมื่อเกิด Fault

// JSR/RET Instruction:
RUNG 1: // เรียก Subroutine Scale_Temperature
    ┌──[ALWAYS_ON]────────────────────────(JSR)──────────┐
    │                              Routine: Scale_Temp    │
    │                              Input Par1: Temp_Raw   │
    │                              Input Par2: 27648      │
    │                              Return Par1: Temp_Eng  │
    └────────────────────────────────────────────────────┘

// ใน Scale_Temp Routine:
RUNG 1:
    ┌──[ALWAYS_ON]────────────────────────────────────────┐
    │  CPT: Par_Return_1 = Par_Input_1 * 200.0 / Par_Input_2 │
    └────────────────────────────────────────────────────┘

RUNG 2: // Return to Caller
    ┌──[ALWAYS_ON]────────────────────────(RET)──────────┐
    │                              Return Par1: Par_Return_1 │
    └────────────────────────────────────────────────────┘
```

---

## Step 568: Program Flow Control

```
// Jump Instructions ใน Ladder

// SIEMENS:
// JMP (Jump): กระโดดไป Label ที่กำหนด
// JMPN (Jump if Not): กระโดดถ้า Condition FALSE

NETWORK 1: // Skip Scaling ถ้า Comm Fault
    ┌──[Comm_Fault]──────────────────────────────(JMP)───┐
    │                                        LABEL: Skip  │
    └────────────────────────────────────────────────────┘

NETWORK 2: // Scaling Code
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  Scale_AI(Raw_Value := IW64, ...)                  │
    └───────────────────────────────────────────────────┘

NETWORK 3: // Label Skip
    ┌──LABEL: Skip───────────────────────────────────────┐
    │  (ข้าม Network 2 ถ้า Comm_Fault)                  │
    └───────────────────────────────────────────────────┘

// ─────────────────────────────────────────────────────
// ALLEN-BRADLEY:
// JMP (Jump): กระโดดไป Label
// LBL (Label): Label ปลายทาง

RUNG 1:
    ┌──[Comm_Fault]──────────────────────────────(JMP)───┐
    │                                         LBL: SKIP  │
    └────────────────────────────────────────────────────┘

RUNG 2: // Processing
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  // Scaling Code                                   │
    └───────────────────────────────────────────────────┘

RUNG 3:
    ┌──(LBL SKIP)────────────────────────────────────────┐
    │  // ปลายทางของ Jump                                │
    └───────────────────────────────────────────────────┘

// ─────────────────────────────────────────────────────
// MCR (Master Control Relay):
// ปิด/เปิด Block ของ Rung

RUNG 1: // Start MCR Zone
    ┌──[Run_Mode]──────────────────────────────(MCR+)────┐
    └────────────────────────────────────────────────────┘

RUNG 2-10: // Rungs ใน MCR Zone
    ┌──... (ทำงานเฉพาะเมื่อ MCR Active) ...──────────────┐
    └────────────────────────────────────────────────────┘

RUNG 11: // End MCR Zone
    ┌──────────────────────────────────────────(MCR-)────┐
    └────────────────────────────────────────────────────┘
```

---

## Step 569: Structured Text Integration กับ Ladder

```
// IEC 61131-3 มี 5 Programming Languages:
// - LD (Ladder Diagram)
// - FBD (Function Block Diagram)
// - SFC (Sequential Function Chart)
// - ST (Structured Text)
// - IL (Instruction List) - เลิกใช้แล้ว

// ST ใน Siemens TIA Portal (SCL):
// เขียนเป็น Text ภาษาคล้าย Pascal/C
// ใช้ใน FC, FB, OB

// ตัวอย่าง SCL ใน FB20 (Analog Processing):

IF #Enable THEN
  // Scale Input
  #Normalized := (INT_TO_REAL(#Raw_Input - #Raw_Min)) / 
                  INT_TO_REAL(#Raw_Max - #Raw_Min);
  #Eng_Value := #Eng_Min + #Normalized * (#Eng_Max - #Eng_Min);
  
  // Clamp to Range
  IF #Eng_Value > #Eng_Max THEN
    #Eng_Value := #Eng_Max;
    #High_Alarm := TRUE;
  ELSIF #Eng_Value < #Eng_Min THEN
    #Eng_Value := #Eng_Min;
    #Low_Alarm := TRUE;
  ELSE
    #High_Alarm := FALSE;
    #Low_Alarm := FALSE;
  END_IF;
  
  // Moving Average
  FOR #i := 7 TO 1 BY -1 DO
    #Avg_Buffer[#i] := #Avg_Buffer[#i-1];
  END_FOR;
  #Avg_Buffer[0] := #Eng_Value;
  
  // Calculate Average
  #Sum := 0.0;
  FOR #i := 0 TO 7 DO
    #Sum := #Sum + #Avg_Buffer[#i];
  END_FOR;
  #Avg_Value := #Sum / 8.0;
  
ELSE
  #Eng_Value := 0.0;
  #Avg_Value := 0.0;
END_IF;

// เรียกใช้ FB20 จาก Ladder ใน OB1:
NETWORK 1:
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB20 "Analog_Processing" [IDB: Temp_Analog_DB]   │
    │  Enable     := TRUE                               │
    │  Raw_Input  := IW64                               │
    │  Raw_Min    := 0                                  │
    │  Raw_Max    := 27648                              │
    │  Eng_Min    := 0.0                                │
    │  Eng_Max    := 200.0                              │
    │  Eng_Value  → Temperature_Actual                  │
    │  Avg_Value  → Temperature_Average                 │
    │  High_Alarm → Temp_High_Alarm                    │
    └───────────────────────────────────────────────────┘
```

---

## Step 570: Mitsubishi - Subroutine ใน GX Works3

```
// Mitsubishi MELSEC - Program Organization
// GX Works3 โครงสร้าง:
// Project
// └── Program Files
//     ├── MAIN (ทำงานทุก Scan)
//     ├── SUB (Subroutine ต้องเรียกจาก MAIN)
//     └── FB (Function Block)

// Subroutine ใน Ladder (Mitsubishi):
// CALL instruction เรียก Subroutine
// RET instruction กลับจาก Subroutine

// ใน MAIN Program:
LD      M0              // Trigger
CALL    P10             // เรียก Subroutine P10

// ใน Subroutine P10:
// (เริ่มด้วย Label P10)
P10:
LD      M8000           // Always ON ใน Subroutine
MOV     K0   D100       // ทำงานใน Subroutine
...
RET                     // กลับไป Main Program

// Function Block ใน Mitsubishi (GX Works3):
// สร้างใน Program Files → New Function Block
// Interface: Input Variables, Output Variables, Static
// เรียกใน Main ด้วย FB Instance

// ตัวอย่าง FB: MOTOR_CTL
// Input: bStart, bStop, bFault
// Output: bRunning, bFaulted
// Static: bLatched (Seal-in)

// เรียกใน Main:
FB MOTOR_CTL Motor1
  bStart  := X0
  bStop   := X1
  bFault  := X2
  bRunning => Y0
  bFaulted => Y1
```

---

## Step 571: Modular Design Principles

```
┌──────────────────────────────────────────────────────────┐
│           Modular Design Principles                      │
│                                                          │
│  1. SINGLE RESPONSIBILITY:                              │
│     แต่ละ FC/FB/Routine ทำหน้าที่เดียว                  │
│     ✓ FB_Motor_Control: ควบคุม Motor อย่างเดียว         │
│     ✗ FB_Motor_And_Alarm_And_HMI: ทำทุกอย่าง            │
│                                                          │
│  2. HIGH COHESION:                                      │
│     Code ที่เกี่ยวข้องกันอยู่ด้วยกัน                    │
│     เช่น: Analog Scaling Logic อยู่ใน FB เดียว          │
│                                                          │
│  3. LOW COUPLING:                                       │
│     ลด Dependency ระหว่าง Modules                       │
│     FB_Motor ไม่ควรอ่าน Global Variable โดยตรง          │
│     ให้ Caller Pass ค่าผ่าน Parameter แทน              │
│                                                          │
│  4. REUSABILITY:                                        │
│     FB_Motor ควร Reuse ได้กับ Motor ทุกตัว              │
│     ไม่ Hardcode Address สำหรับ Motor เฉพาะตัว          │
│                                                          │
│  5. TESTABILITY:                                        │
│     แต่ละ Module ทดสอบแยกได้                            │
│     Simulate Input/Output ได้ง่าย                       │
│                                                          │
│  6. DOCUMENTATION:                                      │
│     ชื่อ FB/FC อธิบายตัวเอง                             │
│     Comment ที่ Parameters ทุกตัว                       │
│     Block Comment อธิบาย Purpose                        │
└──────────────────────────────────────────────────────────┘
```

---

## Step 572: Program Call Hierarchy

```
// ตัวอย่าง Program Call Hierarchy สำหรับ Packaging Plant

// OB1 (Main - ทุก Scan)
// ├── FC1: System_Control
// │   ├── เรียก FB_Motor(IDB_Conveyor1)
// │   ├── เรียก FB_Motor(IDB_Conveyor2)
// │   └── เรียก FB_Valve(IDB_FillerValve)
// │
// ├── FC2: Analog_Processing
// │   ├── เรียก FB_ScaleAI(IDB_Temp1)
// │   ├── เรียก FB_ScaleAI(IDB_Temp2)
// │   └── เรียก FB_ScaleAI(IDB_Level1)
// │
// ├── FC3: PID_Processing
// │   ├── เรียก FB_PID(IDB_TempPID1)
// │   └── เรียก FB_PID(IDB_FlowPID1)
// │
// ├── FC4: Sequencer
// │   └── เรียก FB_Sequencer(IDB_PackagingSeq)
// │
// └── FC5: Alarm_Handling
//     └── เรียก FB_AlarmManager(IDB_Alarms)

// OB30 (Cyclic - ทุก 100ms)
// └── FC10: PID_Execution
//     ├── เรียก FB_PID(IDB_TempPID1)  // ต้องเรียกบ่อยๆ
//     └── เรียก FB_PID(IDB_FlowPID1)

// OB82 (Diagnostic)
// └── FC20: Diagnostic_Handler

// OB100 (Startup)
// └── FC30: Initialize_System
```

---

## Step 573: Instance Data Blocks

```
// Instance DB (IDB) ใน Siemens FB

// IDB คือ:
// - Data Block ที่สร้างมาสำหรับ Instance หนึ่งของ FB
// - เก็บ STATIC Variables ของ Instance นั้น
// - แต่ละครั้งที่เรียก FB ต้องระบุ IDB

// สร้าง IDB อัตโนมัติ:
// TIA Portal: เมื่อลาก FB ลงใน Network
// จะถาม "Create new instance DB" → ตั้งชื่อ

// Multi-Instance:
// FB หนึ่งมี IDB ของตัวเอง
// ภายใน FB นั้น เรียก FB อื่น → ใช้ Multi-Instance
// ไม่ต้องสร้าง IDB แยก สำหรับ Inner FB

// ตัวอย่าง:
// FB10 (Motor_Control) เรียก IEC Timer (TON)
// ถ้าไม่ใช้ Multi-Instance ต้องสร้าง IDB ของ TON แยก
// ถ้าใช้ Multi-Instance: TON เก็บข้อมูลใน IDB ของ FB10

// Multi-Instance Configuration (TIA Portal):
// FB10 Static Variables:
// Run_Timer: TON    ← Declares TON as static within FB10

// การใช้ Multi-Instance ใน Body:
#Run_Timer(IN := #Latch_Run, PT := T#1H);
IF #Run_Timer.Q THEN
  #Run_Hours_Accum := #Run_Hours_Accum + 1.0;
  ...
END_IF;
```

---

## Step 574: Reusable Code ตัวอย่างจริง

```
// ตัวอย่าง Reusable: FB_Conveyor ใช้ได้กับทุก Conveyor

// FB20: Conveyor_Control
// ใช้ได้กับ Conveyor ทุกตัวในโรงงาน

// Interface:
// IN:
//   Start_Auto (BOOL): Start in Auto mode
//   Stop_Cmd (BOOL): Stop command
//   E_Stop (BOOL): Emergency stop
//   Product_Sensor (BOOL): Product detected
//   End_Sensor (BOOL): Conveyor end sensor
//   Speed_Ref (REAL): Speed reference (m/min)
//   OL_Trip (BOOL): Overload relay
// OUT:
//   Motor_FWD (BOOL): Motor Forward Output
//   Motor_Running (BOOL): Motor running status
//   Fault (BOOL): Fault status
//   Product_Count (DINT): Product counter
// STATIC:
//   Latch_Run (BOOL): Run latch
//   Motor_Timer (IEC_TIMER): Run timer
//   Run_Hours (REAL): Accumulated hours

// OB1 - เรียกใช้ FB20 กับ 3 Conveyors:
NETWORK 1:
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB20 "Conveyor_Control" [IDB: Conv1_Data]        │
    │  Start_Auto := Start_Button1                      │
    │  E_Stop := E_Stop_Global                          │
    │  Product_Sensor := I10.0                          │
    │  End_Sensor := I10.1                              │
    │  Speed_Ref := Conv1_Speed_SP                      │
    │  OL_Trip := I10.7                                 │
    │  Motor_FWD → Q10.0                               │
    │  Motor_Running → Conv1_Running                   │
    │  Product_Count → Conv1_Count                     │
    └───────────────────────────────────────────────────┘

NETWORK 2:
    ┌──[ALWAYS_ON]───────────────────────────────────────┐
    │  FB20 "Conveyor_Control" [IDB: Conv2_Data]        │
    │  Start_Auto := Start_Button2                      │
    │  E_Stop := E_Stop_Global                          │
    │  Product_Sensor := I11.0                          │
    │  End_Sensor := I11.1                              │
    │  Speed_Ref := Conv2_Speed_SP                      │
    │  OL_Trip := I11.7                                 │
    │  Motor_FWD → Q11.0                               │
    │  Motor_Running → Conv2_Running                   │
    │  Product_Count → Conv2_Count                     │
    └───────────────────────────────────────────────────┘
```

---

## Step 575: Allen-Bradley Add-On Instruction (AOI)

```
// Allen-Bradley AOI เหมือน FB ของ Siemens
// สร้าง Custom Instruction ที่ใช้ซ้ำได้

// AOI: Motor_Control
// Parameters:
// IN: Start_PB (BOOL), Stop_PB (BOOL), OL (BOOL)
// OUT: Motor_Coil (BOOL), Running (BOOL), Faulted (BOOL)
// IN_OUT: Run_Hours (REAL), Start_Count (DINT)
// LOCAL: (อย่างเดียว ไม่ Share ออก)

// สร้าง AOI ใน Studio 5000:
// Add-On Instructions → New Add-On Instruction
// Name: Motor_Control
// Type: Normal
// จากนั้น Define Parameters และ Local Tags

// Logic ใน AOI (Ladder):
RUNG 1: // Start/Stop Logic
    ┌──[Start_PB]────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTL)───────┐
                                             Run_Latch    │
                                                          └┘

    ┌──[Stop_PB]────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTU)───────┐
                                             Run_Latch    │
                                                          └┘

RUNG 2: // OL Fault
    ┌──[OL]──────────────────────────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────(OTL)───────┐
       │                                      Faulted     │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTU)───────┐
                                             Run_Latch    │
                                                          └┘

RUNG 3: // Output
    ┌──[Run_Latch]──[NOT Faulted]─────────────────────────┐
    └──┬──────────────────────────────────────────────────┘
       ├──────────────────────────────────────(OTE)───────┐
       │                                    Motor_Coil    │
       └──────────────────────────────────────────────────┘
       └──────────────────────────────────────(OTE)───────┐
                                             Running      │
                                                          └┘

// เรียกใช้ AOI ใน Main Routine:
RUNG 1:
    ┌──[ALWAYS_ON]──────────────────────────────────────┐
    │  Motor_Control                                    │
    │  Start_PB := I:0/0                               │
    │  Stop_PB := I:0/1                                │
    │  OL := I:0/7                                     │
    │  Motor_Coil => O:0/0                             │
    │  Running => Motor1_Running                       │
    │  Faulted => Motor1_Faulted                       │
    │  Run_Hours <=> Motor1_Hours                      │
    │  Start_Count <=> Motor1_Starts                   │
    └───────────────────────────────────────────────────┘
```

---

## Step 576: Library Management

```
┌──────────────────────────────────────────────────────────┐
│           Library Management                             │
│                                                          │
│  ทำไมต้องมี Library:                                    │
│  - Reuse Code ระหว่างโปรเจกต์                           │
│  - มาตรฐานเดียวกันในองค์กร                              │
│  - อัปเดต Bug Fix ง่าย (แก้ที่เดียว → ทุก Project)      │
│                                                          │
│  Siemens TIA Library:                                   │
│  Project Library: ใช้ได้เฉพาะใน Project นี้            │
│  Global Library: ใช้ร่วมได้หลาย Project               │
│                                                          │
│  สร้าง Global Library:                                 │
│  Libraries → New Global Library                        │
│  Add FB/FC ที่ต้องการ Share                            │
│  Save as .al16 file                                    │
│                                                          │
│  ใช้ Global Library:                                    │
│  Libraries → Open Global Library                       │
│  Drag Block ไปยัง Project                              │
│                                                          │
│  Allen-Bradley - Template Project:                     │
│  ไม่มี Library System แบบ Siemens                       │
│  แต่ใช้ Template Project ที่มี AOI สำเร็จรูป           │
│  New Project → From Template                           │
│                                                          │
│  Best Practices:                                        │
│  1. Version Control ใน Library (v1.0, v1.1, v2.0)     │
│  2. Changelog สำหรับทุก Version                         │
│  3. Test FB ก่อน Add เข้า Library                     │
│  4. จำกัดสิทธิ์การแก้ไข Library                        │
└──────────────────────────────────────────────────────────┘
```

---

## Step 577: Program Documentation Standards

```
// มาตรฐาน Documentation ใน PLC Program

// 1. Block Header Comment:
// (ใส่ใน Block Properties → Comment)

// Block: FB_Motor_Control
// Author: John Smith
// Date: 2024-01-15
// Version: 2.1
// Description: Standard motor control with run hours
//              and start count tracking
// Change Log:
//   v2.1 - 2024-01-15: Added run hours reset
//   v2.0 - 2023-12-01: Added start counter
//   v1.0 - 2023-10-01: Initial version

// 2. Parameter Comments:
// Start_Cmd: Start command input (Rising edge starts motor)
// Stop_Cmd: Stop command input (High = stop)
// Motor_Running: Output high when motor is running
// Run_Hours: Accumulated running hours

// 3. Network Comments:
// Network 1: Start/Stop Logic - Seal-in circuit
// Network 2: Fault Detection - OL trip handling
// Network 3: Output Assignment
// Network 4: Run Hour Counter

// 4. Inline Comments (ใน Rung):
// ─── [Seal_In Contact] ← ช่วยให้ Motor ทำงานต่อ ─────────

// 5. Symbol Names:
// ตั้งชื่อ Tag ที่อ่านเข้าใจ
// ✓ Motor_1_Start_PB_Normally_Open
// ✗ M1SPBNO, I0_0, Bool1
```

---

## Step 578: Code Review Checklist

```
┌──────────────────────────────────────────────────────────┐
│           PLC Code Review Checklist                      │
│                                                          │
│  STRUCTURE:                                             │
│  ☐ OB1 มีแต่การเรียก FC/FB ไม่มี Logic โดยตรง         │
│  ☐ FC/FB มีขนาดเหมาะสม (< 50 Rungs ต่อ Block)         │
│  ☐ ชื่อ Block สื่อความหมาย                              │
│                                                          │
│  NAMING:                                                │
│  ☐ Tags ทุกตัวมีชื่อที่อ่านเข้าใจ                       │
│  ☐ ไม่มี Generic Names (Temp1, Flag2, Bool3)            │
│  ☐ ใช้ Convention เดียวกัน (CamelCase หรือ Underscore)  │
│                                                          │
│  SAFETY:                                                │
│  ☐ E-Stop Logic ใน OB1 ระดับบน (ไม่ถูก Skip ได้)       │
│  ☐ Power-on State ปลอดภัย (Output ปิดเมื่อ Power-up)    │
│  ☐ มี Timeout สำหรับทุก State ใน Sequencer              │
│                                                          │
│  MAINTAINABILITY:                                       │
│  ☐ Comment ที่ Network ที่ซับซ้อน                       │
│  ☐ Magic Numbers ใช้ Constant แทน (Pi = 3.14159)       │
│  ☐ ไม่มี Duplicate Logic (ใช้ FC/FB แทน)               │
│                                                          │
│  PERFORMANCE:                                           │
│  ☐ PID อยู่ใน Cyclic OB (ไม่ใน OB1)                    │
│  ☐ ไม่มี Infinite Loop ใน SCL/ST                       │
│  ☐ Array Operations มี Index Bounds Check              │
└──────────────────────────────────────────────────────────┘
```

---

## Step 579: ตัวอย่างสมบูรณ์ - Structured Program Architecture

```
// Complete Structured Program for Bottling Line

// ─────────────────────────────────────────────────────
// PROGRAM ARCHITECTURE:
//
// Startup OB100: Initialize_System_FC
//   ├── Set Default Setpoints
//   ├── Clear Fault Registers
//   └── Set Initial State
//
// Main OB1 (Cyclic, 10ms scan):
//   ├── FC_Safety_Monitor (E-Stop, Guard, OL)
//   ├── FC_Sequencer (Main Machine Sequence)
//   │   ├── FB_Conveyor(Conv1_IDB)
//   │   ├── FB_Conveyor(Conv2_IDB)
//   │   ├── FB_Filler(Filler_IDB)
//   │   └── FB_Capper(Capper_IDB)
//   ├── FC_Analog_IO
//   │   ├── FB_ScaleAI(Level_IDB)
//   │   └── FB_ScaleAI(Weight_IDB)
//   └── FC_Alarms
//       └── FB_Alarm(Alarm_IDB)
//
// Periodic OB30 (100ms):
//   └── FC_PID_Execution
//       └── FB_PID(Weight_PID_IDB)
//
// OB82: Diagnostic_Handler_FC
// OB86: Station_Fault_Handler_FC
// ─────────────────────────────────────────────────────

// FC_Safety_Monitor (เรียกก่อนเสมอใน OB1):
NETWORK 1: // E-Stop Check
    ┌──[E_Stop_1]/──[E_Stop_2]/────────────────( )───────┐
    │  E-Stop 1 NC   E-Stop 2 NC          Safety_OK     │
    └──────────────────────────────────────────────────┘

NETWORK 2: // Global Safety: ปิด Output ทั้งหมดเมื่อ Safety Error
    ┌──[Safety_OK]/──────────────────────────────────────┐
    │  NOT Safe                                          │
    └──┬──────────────────────────────────────────────────┘
       └──[JMP to END_OB1]  // Skip ทุก Logic           │
```

---

## Step 580: Lab Project - Structured Program Design

**โจทย์:** Redesign โปรแกรมที่ Unstructured ให้เป็น Structured

```
┌──────────────────────────────────────────────────────────┐
│           Structured Programming Lab                     │
│                                                          │
│  Starting Point: (Unstructured)                         │
│  OB1 ที่มี 500 Rungs สำหรับ 3 Motor Control,            │
│  2 Analog Channels, Simple Sequencer                   │
│                                                          │
│  Tasks:                                                 │
│  1. วิเคราะห์โปรแกรมเดิม ระบุ Functional Groups        │
│  2. ออกแบบ FB_Motor_Control:                           │
│     a. Define Interface (IN/OUT/STATIC)                │
│     b. Implement Body                                  │
│     c. Test กับ 1 Motor ก่อน                          │
│  3. ออกแบบ FC_Scale_AI:                               │
│     a. Generic Scaling Function                       │
│     b. Test กับ 2 Analog Channels                     │
│  4. ออกแบบ FB_Sequencer:                              │
│     a. State Machine ด้วย STATIC State Variable       │
│     b. Implement State Transitions                    │
│  5. จัด OB1 ให้เรียกแค่ FC/FB                         │
│  6. เปรียบเทียบ Code Size และ Readability             │
│  7. ทดสอบว่า Logic ยังทำงานถูกต้อง                   │
│  8. Document Program Architecture                     │
│                                                          │
│  Bonus:                                               │
│  - เพิ่ม Global Library เก็บ FB ที่เสร็จแล้ว           │
│  - เขียน Test Case สำหรับแต่ละ FB                      │
└──────────────────────────────────────────────────────────┘
```

---

## สรุป Part 29

ใน Part นี้ได้เรียนรู้:

1. **Why Structured Programming** - Benefits, Problems Without
2. **OB/FC/FB** - Siemens Building Block Types
3. **FC Example** - Stateless Scaling Function
4. **FB Example** - Stateful Motor Control
5. **Parameter Types** - IN, OUT, IN_OUT, STATIC, TEMP
6. **Program Flow** - JMP, LBL, MCR, JSR, RET
7. **ST Integration** - Using Structured Text in FB
8. **Allen-Bradley** - Routines, JSR/RET, AOI
9. **Mitsubishi** - Subroutines, Function Blocks
10. **Modular Design** - Principles and Patterns
11. **Library Management** - Siemens Global Library
12. **Code Review** - Checklist and Standards

---

## Navigation
- [← Part 28: EtherNet/IP](part28_ethernet_ip.md)
- [→ Part 30: Function Blocks (FB)](part30_function_blocks.md)
- [กลับหน้าหลัก](../README.md)
