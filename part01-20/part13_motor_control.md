# Part 13: Motor Control พื้นฐาน (Steps 231-250)

## ภาพรวมของบทเรียน

การควบคุมมอเตอร์เป็นงานหลักที่สำคัญที่สุดของระบบ PLC ในโรงงาน บทนี้ครอบคลุมวิธีการสตาร์ทมอเตอร์หลายแบบ ตั้งแต่ Direct On-Line (DOL), Star-Delta, Soft Starter, ไปจนถึง Variable Frequency Drive (VFD) พร้อมทั้ง Motor Protection ที่จำเป็น

---

## Step 231: มอเตอร์ 3 เฟสพื้นฐาน

### โครงสร้างมอเตอร์ 3 เฟส (Induction Motor)

```
3-Phase Induction Motor Nameplate Data:

Model: 5.5kW 3Φ Motor
Voltage: 380/660V ∆/Y
Current: 13.7/7.9A
Speed: 1450 RPM (4-pole)
Frequency: 50Hz
Power Factor: 0.82
Efficiency: 87.5%
Insulation: Class F
IP Rating: IP55
```

### Star (Y) และ Delta (△) Connection

```
Star (Y) Connection:              Delta (△) Connection:
     R                                  R
     │                            ┌─────┤
     │  R phase ──────────────────┘     │
     │                             R-S  │
     │  S phase ──────────────────┐     │
     │                            └─────┤
     │  T phase ──────────────────┐  S-T│
     │                            └─────┤
     └──────── N (Neutral)         T-R  │
                                   └────┘

Star: V_phase = V_line / √3      Delta: V_phase = V_line
      I_phase = I_line                  I_phase = I_line / √3

ตัวอย่าง 380V system:
Star:  V_phase = 380/1.732 = 220V per winding
Delta: V_phase = 380V per winding
```

---

## Step 232: Direct On-Line (DOL) Starter

### หลักการ DOL

DOL คือวิธีสตาร์ทมอเตอร์ที่ง่ายที่สุด โดยต่อมอเตอร์เข้ากับ Power Line โดยตรง

```
DOL Advantages:
✓ วงจรง่าย ราคาถูก
✓ Torque สูงตั้งแต่เริ่มสตาร์ท
✓ ไม่มีการลด Voltage

DOL Disadvantages:
✗ กระแส Inrush สูง (5-8 × In)
✗ Mechanical shock สูง
✗ ไม่เหมาะกับมอเตอร์ใหญ่ (>15kW ในระบบ 400V)
```

### DOL Starter Power Circuit

```
3-Phase Power Circuit (DOL):

L1 ──[MCB]──[KM1 Main Contact]──────────────── U (Motor)
                    │
L2 ──[MCB]──[KM1 Main Contact]──────────────── V (Motor)
                    │
L3 ──[MCB]──[KM1 Main Contact]──────────────── W (Motor)

MCB = Molded Case Circuit Breaker (ป้องกัน Short Circuit)
KM1 = Main Contactor (สตาร์ท/หยุด)
OL = Overload Relay (ป้องกัน Overload)

ครบวงจร:
L1──[MCB]──[KM1]──[OL]──────────────────────── U
L2──[MCB]──[KM1]──[OL]──────────────────────── V
L3──[MCB]──[KM1]──[OL]──────────────────────── W
```

### DOL Control Circuit

```
Control Wiring (24VDC):

+24V
  │
  ├──[ESTOP NC]──[OL NC]──[PB-STOP NC]──[PB-START NO]──[KM1 Coil]
  │                                                          │
  │                              [KM1 Aux NO]───────────────┘
  │                              (Self-Hold Contact)
 0V
```

### DOL Ladder Diagram (Siemens S7-1200)

```ladder
// DOL Motor Starter
// Siemens S7-1200 TIA Portal

// Network 1: Motor Run Command
|  %I0.2   |  %I0.0   |  %I0.1   |  %I0.3   |
|ESTOP_NC  |PB_STOP_NC|PB_START  |OL_NC    |──────────────( %Q0.0 )
|(Emergency|(Stop     |(Start    |(Overload |             KM1_Coil
| Stop)    | Button)  | Button)  | Relay)   |
|          |          |          |          |
|  %Q0.0   |
|KM1_Coil  |──────────────────────────────────────────────┘
|(Self Hold)|

// Network 2: Run Indicator
|  %Q0.0   |
|KM1_Coil  |──────────────────────────────────────────────( %Q0.4 )
|          |                                               Run_Lamp
|          |                                               (Green)

// Network 3: Fault Indicator
|  %I0.3   |
|OL_NC     |──────────────────────────────────────────────( %Q0.5 )
|(NC)      |                                               Fault_Lamp
|          |                                               (Red)
```

**Allen-Bradley (RSLogix/Studio 5000):**
```ladder
// DOL Motor Starter (Allen-Bradley)
// Compact/CompactLogix

// Rung 1: Motor Control
-| ESTOP_NC |-| OL_NC |-| PB_STOP_NC |-| PB_START |-(Motor_Run)-
                                         |
                         -| Motor_Run |--+

// Rung 2: Run Light
-| Motor_Run |-(Run_Light)-
```

**Mitsubishi FX3U:**
```ladder
// DOL Motor Starter (Mitsubishi FX3U)
// GX Works2

// LD X0 → PB_START
// LDI X1 → PB_STOP (NC = Normally Closed = ใช้ Normally Open ใน PLC)
// ANI X2 → OL (NC)
// ANI X3 → E-Stop (NC)
// OR Y0  → Self Hold
// OUT Y0 → KM1 Coil

LD  X0          // START Button (NO)
OR  Y0          // Self-hold
ANI X1          // STOP Button (NC → ใช้ NCI ใน Ladder)
ANI X2          // OL (NC)
ANI X3          // E-Stop (NC)
OUT Y0          // KM1 Coil

LD  Y0
OUT Y1          // Run Lamp
```

---

## Step 233: Star-Delta Starter

### หลักการ Star-Delta

Star-Delta Starter ลดกระแส Inrush โดย:
1. สตาร์ทด้วยการต่อมอเตอร์แบบ Star (กระแสลด 1/3)
2. หลังสตาร์ทแล้ว เปลี่ยนเป็น Delta (Full Power)

```
Star-Delta Sequence:
Time 0:    KM_Star ON + KM_Main ON → Motor starts in Star
           Voltage per winding = 220V (ลดจาก 380V)
           Current = 1/3 × DOL Current

Time 5-10s: KM_Star OFF → Brief pause (10-50ms)

Time ~0.1s: KM_Delta ON → Motor runs in Delta
           Voltage per winding = 380V (Full voltage)
           Current = Full running current
```

### Star-Delta Power Circuit

```
Star-Delta Power Wiring:

L1──┬──[KM_Main]──┬──U1 (Motor Terminal)
    │              │
L2──┼──[KM_Main]──┼──V1 (Motor Terminal)
    │              │
L3──┴──[KM_Main]──┴──W1 (Motor Terminal)
                   │
              ┌────┤
              │    │
        KM_Star    KM_Delta
              │    │
     U2,V2,W2─┘    │
    (Star Short)    └──U2──V2──W2 (Delta Connection)

หมายเหตุ: KM_Star และ KM_Delta ต้องมี Interlock!
          ห้าม ON พร้อมกัน = Short Circuit!
```

### Star-Delta Ladder Diagram

```ladder
// Star-Delta Motor Starter
// Siemens S7-1200

// Network 1: Main Contactor Control
|  %I0.0   |  %I0.1   |  %I0.2   |  %I0.3   |
|ESTOP_NC  |OL_NC    |PB_STOP_N |PB_START  |──────────────( %Q0.0 )
|          |          |           |          |             KM_MAIN
|  %Q0.0   |
|KM_MAIN   |──────────────────────────────────────────────┘
|(Self Hold)|

// Network 2: Start in Star Mode
|  %Q0.0   |  %Q0.2   |         |
|KM_MAIN   |KM_DELTA  |         |──────────────────────────( %Q0.1 )
|          |  (NC)    |         |                          KM_STAR
|          |Interlock |         |

// Network 3: Star-to-Delta Timer
|  %Q0.1   |
|KM_STAR   |────────────────────────────────────────────[TON T1]
|          |                                             PT=T#8S
|          |                                             (8 seconds)

// Network 4: Delta Mode (After Timer)
|  %T1.Q   |  %Q0.1   |
|Timer_Done|KM_STAR   |──────────────────────────────────( %Q0.2 )
|          | (NC)     |                                  KM_DELTA
|          |Interlock |

// Network 5: Running Indicator
|  %Q0.2   |
|KM_DELTA  |──────────────────────────────────────────────( %Q0.3 )
|(Full Run)|                                              RUN_LAMP
```

**Timing Diagram:**
```
Time:   0s        8s
        │          │
KM_MAIN:│──────────────────────────────────
        │          │
KM_STAR:│──────────┘   (8s Star period)
        │               │
KM_DELTA:               └─────────────────── (Full Run)
        │          │
Current:│▓▓▓▓▓▓▓▓▓▓│░░░░░░░░░░░░░░░░░░░░░░  (ลดลง)
        │ (Star)   │(Delta)
```

---

## Step 234: Soft Starter

### หลักการ Soft Starter

Soft Starter ใช้ SCR (Silicon Controlled Rectifier) ควบคุมแรงดันที่จ่ายให้มอเตอร์ค่อยๆ เพิ่มขึ้น

```
Soft Starter Voltage Profile:

Voltage
  │
100%┤                    ┌──────────────────────── Full Voltage
    │                   /
    │                  /
    │                 /  ← Ramp Up (ค่อยๆ เพิ่ม)
    │                /
    │               /
 30%┤──────────────┘
    │    Initial Voltage (เริ่มต้น)
    │
    └─────────────────────────────────────────→ Time
    0    1s    2s    3s    4s    5s    (Ramp Time)
```

### Soft Starter Wiring

```
Soft Starter Control + Power Wiring:

Power Circuit:
L1──[MCB]──[Bypass KM]──[Soft Starter L1]──────────── U Motor
L2──[MCB]──[Bypass KM]──[Soft Starter L2]──────────── V Motor
L3──[MCB]──[Bypass KM]──[Soft Starter L3]──────────── W Motor
                  │
         [Bypass Contactor = ต่อตรงเมื่อ Full Speed]

Control Signals (from PLC):
+24V ─ Run Input (Digital)     ──► RUN Terminal
+24V ─ Stop Input (Digital)    ──► STOP Terminal
IW64 ─ Analog Feedback (4-20mA) ◄── Current Output
```

### Soft Starter Ladder Diagram

```ladder
// Soft Starter Control with PLC
// Allen-Bradley CompactLogix

// Rung 1: Start Command to Soft Starter
-| ESTOP_OK |-| OL_OK |-| PB_STOP_NC |-| PB_START |-(SS_RUN_CMD)-
                                        |
               -| SS_RUN_CMD |-----------+  (Self Hold)

// Rung 2: Start Soft Starter After Delay
-| SS_RUN_CMD |-[TON, Timer_Start, 0.1s]-(SS_Start_Output)-

// Rung 3: Enable Bypass After Up-to-Speed
-| SS_AT_SPEED |-[TON, Bypass_Timer, 0.5s]-(Bypass_Contactor)-

// Rung 4: Monitor Fault
-| SS_FAULT |-(SS_Fault_Bit)-

// Rung 5: Reset Fault
-| SS_Fault_Bit |-| RESET_PB |-(SS_RESET_CMD)-
```

---

## Step 235: Variable Frequency Drive (VFD)

### หลักการ VFD

VFD (หรือ Inverter) แปลงไฟ AC ขาเข้าเป็น DC แล้วแปลงเป็น AC ขาออกที่ความถี่และแรงดันปรับได้

```
VFD Block Diagram:

3-Phase AC Input
       │
       ▼
┌─────────────┐
│  Rectifier  │  AC → DC (Diode Bridge)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  DC Link    │  Smooth DC (Capacitor Bank)
│  Filter     │  ~540VDC (380V system)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Inverter  │  DC → Variable AC (IGBT Switching)
│   (IGBT)   │  PWM Modulation
└──────┬──────┘
       │
       ▼
3-Phase Variable AC Output (V/f = Constant)

Output Frequency: 0-120Hz (ปรับได้)
Output Voltage: 0-480V (ตาม f/f_rated)
```

### V/f Control (Voltage/Frequency)

```
V/f Constant Ratio ทำให้ Torque คงที่ตลอดช่วง Speed:

Voltage│         /── Overmodulation
       │        /
  380V ┤───────/ ← V/f = Constant
       │      /
  220V ┤─────/
       │    /
       │   /
       │  /
       └──────────────────────────────→ Frequency
       0   25   50Hz               100Hz
                ↑
           Rated frequency
```

### VFD Terminal Wiring

```
VFD Terminal Block:

Power Input:        Control Input:
R/L1 ─── L1        +10V ─── Speed Reference Power
S/L2 ─── L2        AI1+ ─── Speed Setpoint (0-10V)
T/L3 ─── L3        AI1- ─── Speed Setpoint (Common)
                    DI1  ─── Run Forward
Motor Output:       DI2  ─── Run Reverse  
U/T1 ─── U         DI3  ─── Reset Fault
V/T2 ─── V         DI4  ─── Preset Speed 1
W/T3 ─── W         DI5  ─── Preset Speed 2
                    COM  ─── Digital Common

                    Relay Output:
                    RL1  ─── Running (NO/NC)
                    RL2  ─── Fault   (NO/NC)

                    Analog Output:
                    AO1  ─── Output Frequency (0-10V)
                    AO2  ─── Output Current   (4-20mA)
```

### VFD Control with PLC Ladder Diagram

```ladder
// VFD Motor Speed Control
// Siemens S7-1200

// Network 1: VFD Run Command
|  %I0.0   |  %I0.1   |  %I0.2   |
|ESTOP_NC  |PB_STOP_NC|PB_START  |──────────────────────( %Q0.0 )
|          |          |          |                       VFD_RUN
|  %Q0.0   |
|VFD_RUN   |──────────────────────────────────────────────┘

// Network 2: Speed Reference (Write to AO)
|  Always   |
|  ON       |──[MOVE]─────────────────────────────────────────────┤
|           │  IN:  #Speed_Setpoint (0-100%)                      │
|           │  OUT: %QW80 (Analog Output)                         │
|           │  Scale: 0-100% → 0-27648 (0-10V)                   │

// Network 3: Read VFD Actual Speed (from AI)
|  Always   |
|  ON       |──[SCALE]────────────────────────────────────────────┤
|           │  IN: %IW64 (Analog Input, VFD Freq Output)          │
|           │  LO_LIM: 0.0                                        │
|           │  HI_LIM: 60.0   (0-60Hz)                           │
|           │  OUT: #Actual_Frequency                             │

// Network 4: VFD Fault Monitor
|  %I0.3   |
|VFD_FAULT |──────────────────────────────────────────────( %M5.0 )
|(from VFD |                                               VFD_Fault
| relay)   |

// Network 5: Fault Reset
|  %I0.4   |  %M5.0   |
|RESET_PB  |VFD_Fault |──────────────────────────────────( %Q0.1 )
|          |          |                                  VFD_RESET
```

**Mitsubishi FX3U กับ VFD:**
```
// VFD Speed Control - Mitsubishi FX3U
// ใช้ D-A Converter Module FX2N-2DA

TO  K0  K1  D100  K1    // Write Speed (0-4000) to D/A CH1
FROM K0  K0  D200  K1    // Read Actual Speed from A/D CH1

LD X0               // Start Button
OR M0               // Self Hold  
ANI X1              // Stop Button
OUT Y0              // VFD RUN Signal
OUT M0              // Self Hold bit
```

---

## Step 236: Forward/Reverse Motor Control

### Interlock วงจร Forward/Reverse

```
INTERLOCK สำคัญมาก!
ห้าม KM_FWD และ KM_REV ON พร้อมกัน = Short Circuit 3-Phase!

Hardware Interlock:
KM_FWD coil ─── NC Contact of KM_REV
KM_REV coil ─── NC Contact of KM_FWD

Software Interlock:
Q_FWD = ON → ห้าม Q_REV = ON
Q_REV = ON → ห้าม Q_FWD = ON
```

### Forward/Reverse Power Circuit

```
Forward/Reverse Power Circuit:

       KM_FWD          KM_REV
L1 ──┬─[Main FWD]──┬──[Main REV Swapped]──
     │              │
L2 ──┼─[Main FWD]──┼──[Main REV]──────────── Motor
     │              │
L3 ──┴─[Main FWD]──┴──[Main REV Swapped]──

สังเกต: KM_REV จะสลับขา L1 กับ L3 ทำให้มอเตอร์หมุนกลับ!
```

### Forward/Reverse Ladder Diagram

```ladder
// Forward/Reverse Motor Control
// Siemens S7-1200

// Network 1: Forward Command
|  %I0.4   |  %I0.0   |  %I0.1   |  %I0.2   |  %Q0.1   |
|ESTOP_NC  |OL_NC    |PB_STOP_NC|PB_FWD    |KM_REV_NC |──( %Q0.0 )
|          |          |           |          |H/W Inter |  KM_FWD
|          |          |           |          | lock     |
|  %Q0.0   |
|KM_FWD    |──────────────────────────────────────────────┘
|(Self Hold)|

// Network 2: Reverse Command  
|  %I0.4   |  %I0.0   |  %I0.1   |  %I0.3   |  %Q0.0   |
|ESTOP_NC  |OL_NC    |PB_STOP_NC|PB_REV    |KM_FWD_NC |──( %Q0.1 )
|          |          |           |          |H/W Inter |  KM_REV
|          |          |           |          | lock     |
|  %Q0.1   |
|KM_REV    |──────────────────────────────────────────────┘
|(Self Hold)|

// Network 3: Software Interlock (Double Protection)
|  %Q0.0   |
|KM_FWD    |────────────────────────────────── SET %M0.0
|          |                                   FWD_Active

|  %Q0.0   |
|KM_FWD    |────────────────────────────────── RESET %M0.1
|(NC)      |                                   REV_Active

// FWD_Active → Block REV_Active and vice versa
```

**Allen-Bradley ControlLogix:**
```ladder
// Forward/Reverse (AB ControlLogix)

// Rung 1: Forward
-|ESTOP_OK|-|OL_OK|-|PB_STOP_NC|-|PB_FWD|-|Motor_Rev_NC|-(Motor_Fwd)-
                                  |
             -|Motor_Fwd|---------+

// Rung 2: Reverse  
-|ESTOP_OK|-|OL_OK|-|PB_STOP_NC|-|PB_REV|-|Motor_Fwd_NC|-(Motor_Rev)-
                                  |
             -|Motor_Rev|---------+
```

---

## Step 237: Motor Protection Relay

### Thermal Overload Relay

```
Thermal Overload Relay (OL Relay) ป้องกัน:
- Overload (กระแสเกิน)
- Phase Loss (ขาดเฟส)
- Phase Imbalance (ไม่สมดุล)
- Under-voltage (แรงดันต่ำ)

OL Relay Construction:
         Current Input (จาก Contactor)
              │
         Bimetallic Strip
              │ (เอียงเมื่อร้อน)
              │
         Trigger Mechanism
              │
         ┌───┴───┐
         │ NC    │ NO
         │ (95/96│ (97/98)
         └───────┘
              │
         Reset Button
         (Manual/Auto)
```

### Electronic Motor Protection Relay

```
Electronic OL Relay (ทันสมัยกว่า):
- ตั้งค่า I_rated ได้แม่นยำ
- วัด Current แต่ละเฟส
- Trip Class: 10, 10A, 20, 30
- Communication: Modbus, PROFIBUS
- Memory: จำ Thermal Condition ได้

ตัวอย่าง: Siemens 3RU2 / Schneider LRE
```

### การตั้งค่า OL Relay

```
ตั้งค่ากระแส Overload:

In (Nameplate Current) = 13.7A สำหรับมอเตอร์ 5.5kW

Setting Range: 12.5A - 14.0A
Recommended: 13.7A (= In)

Trip Time at 150% of Setting:
Class 10: Trip < 10s
Class 20: Trip < 20s
Class 30: Trip < 30s

ใช้ Class 10 สำหรับงานทั่วไป
ใช้ Class 20/30 สำหรับมอเตอร์ที่ Start ยาก (Compressor, Fan)
```

### Motor Protection Ladder Program

```ladder
// Motor Protection Monitoring
// Siemens S7-1200

// Network 1: Overload Fault Detection
|  %I0.5   |
|OL_TRIP   |──────────────────────────────────────────────( %M1.0 )
|(OL Relay |                                               OL_Fault
| Output)  |

// Network 2: Phase Loss Detection (from OL Relay)
|  %I0.6   |
|PHASE_LOSS|──────────────────────────────────────────────( %M1.1 )
|          |                                               PL_Fault

// Network 3: Combined Fault
|  %M1.0   |
|OL_Fault  |──────────────────────────────────────────────( %M1.9 )
|          |                                               Motor_Fault
|  %M1.1   |
|PL_Fault  |──────────────────────────────────────────────┘

// Network 4: Motor Fault Alarm
|  %M1.9   |
|Motor_Fault─────────────────────────────────────────────( %Q0.5 )
|          |                                              Fault_Lamp
|          |────────────────────────────────────────────( %Q0.6 )
|          |                                              Fault_Buzzer
```

---

## Step 238: Motor Control Center (MCC)

### โครงสร้าง MCC

```
Motor Control Center Layout:

┌────────────────────────────────────────────────────────────┐
│                    MOTOR CONTROL CENTER                     │
│                                                            │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│ │  Bus Bar │ │  Bus Bar │ │  Bus Bar │ │ Bus Bar      │ │
│ │  Feeder  │ │  Feeder  │ │  Feeder  │ │  Feeder      │ │
│ └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────┬────────┘ │
│      │            │            │              │           │
│ ┌────┴─────┐ ┌────┴─────┐ ┌────┴─────┐ ┌─────┴────────┐ │
│ │ Motor 1  │ │ Motor 2  │ │ Motor 3  │ │  Spare       │ │
│ │ MCB+KM1  │ │ MCB+KM2  │ │ MCB+KM3  │ │              │ │
│ │ OL+Wiring│ │ OL+Wiring│ │ OL+Wiring│ │              │ │
│ │ DOL 5.5kW│ │ SD 11kW  │ │ VFD 7.5kW│ │              │ │
│ └──────────┘ └──────────┘ └──────────┘ └──────────────┘ │
│                                                            │
│ PLC Control Panel ──────────────────────────────────────  │
│ (HMI + PLC + I/O Modules)                                 │
└────────────────────────────────────────────────────────────┘
```

---

## Step 239: Complete Motor Control Program

### Motor Control Sequencing

```ladder
// Complete Motor Control Program
// Multiple motors with Sequence Start

// Network 1: Sequence Start Permission
|  ESTOP_OK |  ALL_OL_OK |  PRE_START |
|           |            |    Done    |────────────────────( %M100.0 )
|           |            |            |                   Start_Permit

// Network 2: Motor 1 (Pump)
|  %M100.0  |  %I0.0   |  %I0.1   |
|Start_     |PB_STOP_NC|PB_M1_ST  |────────────────────( %Q0.0 )
|Permit     |          |          |                     PUMP_1
|           |
|  %Q0.0    |
|PUMP_1     |────────────────────────────────────────────┘

// Network 3: Motor 2 Starts 3s After Motor 1
|  %Q0.0    |
|PUMP_1     |────────────────────────────────────────[TON T_M2]
|           |                                          PT=T#3S

|  %T_M2.Q  |  %I0.2   |
|Delay_Done |PB_M2_ST  |────────────────────────────( %Q0.1 )
|           |          |                             PUMP_2
|  %Q0.1    |
|PUMP_2     |────────────────────────────────────────┘

// Network 4: Motor 3 Starts When M1 AND M2 Running
|  %Q0.0    |  %Q0.1   |  %I0.3   |
|PUMP_1     |PUMP_2    |PB_M3_ST  |────────────────────( %Q0.2 )
|Running    |Running   |          |                     CONVEYOR
|           |
|  %Q0.2    |
|CONVEYOR   |────────────────────────────────────────────┘
```

---

## Step 240: Motor Braking

### Dynamic Braking

```
Dynamic Braking หยุดมอเตอร์เร็วขึ้นโดยใช้ Resistor

Normal Run:
L1 ──[KM_Main]── Motor
L3 ──[KM_Main]── Motor

Braking (หลังตัด KM_Main):
DC Supply ──[KM_Brake]── Motor Winding 1-2
                         (DC ทำให้เกิด Braking Torque)

Sequence:
Time 0s: KM_Main OFF
Time 0.1s: KM_Brake ON (หลัง Interlock Timer)
Time 1-3s: Braking
Time x+s: KM_Brake OFF
```

### Dynamic Braking Ladder Diagram

```ladder
// Dynamic Braking Control

// Network 1: Main Run
|  PB_START  |  PB_STOP_NC |
|            |             |────────────────────────────( KM_Main )
|  KM_Main   |
|(Self Hold) |────────────────────────────────────────────┘

// Network 2: Stop Command → Start Braking
|  PB_START  |
|(NC - Was ON|────────────────────────────────────[TON T_Brake_Delay]
| Now OFF)   |                                    PT=T#100MS

// Network 3: Brake Contactor (after 100ms delay)
|  T_Brake_  |  KM_Main   |                          |
|  Delay.Q   |  (NC)      |──────────────────────────( KM_Brake )
|            |            |
|  KM_Brake  |────────────────────────────────────[TON T_Brake_Time]
|            |                                    PT=T#3S

// Network 4: Release Brake After Time
|  T_Brake_  |
|  Time.Q    |────────────────────────────────────────( RESET KM_Brake )
```

---

## Step 241: Energy Saving

### Motor Energy Monitoring

```ladder
// Motor Energy Monitoring
// อ่านค่า Power จาก Power Meter (Modbus)

// Network 1: Read Power from Meter (MB_COMM_LOAD)
|  Always ON |
|            |──[MB_MASTER]───────────────────────────────────────┤
|            │  MB_ADDR: 1                                        │
|            │  MODE: 4 (Read Input Registers)                    │
|            │  DATA_ADDR: 40001 (Power Reading)                  │
|            │  DATA_LEN: 2                                       │
|            │  DATA_PTR: DB_Power.Power_kW                       │

// Network 2: Calculate Running Hours
|  %Q0.0    |
|Motor_Run  |────────────────────────────────────────[TON T_Hours]
|           |                                         PT=T#1H

|  T_Hours.Q|
|           |────────────────────────────────────────[INC DB.Run_Hours]

// Network 3: Calculate Energy (kWh)
|  T_Hours.Q|
|           |──[ADD]─────────────────────────────────────────────┤
|           │  IN1: DB.Energy_kWh                                │
|           │  IN2: DB.Power_kW × 1h                             │
|           │  OUT: DB.Energy_kWh                                │
```

---

## Step 242: Motor Control in Process Applications

### Pump Control with Pressure Feedback

```ladder
// Pump Pressure Control
// Start/Stop based on pressure

// Network 1: Pressure High = Stop Pump
|  DB.Pressure_Bar |  DB.Pressure_Bar  |
|     > 6.0        |     < 5.0        |─────────────────( Pump_On )
|   Pressure_High  |  Pressure_Low    |
|   (STOP)         |  (START)         |

// Network 2: Pump Control
|  Pump_On  |  ESTOP_OK  |  OL_OK  |
|           |            |         |────────────────────( KM_Pump )
|  KM_Pump  |
|(Self Hold)|────────────────────────────────────────────┘

// Network 3: Pressure Alarm
|  DB.Pressure_Bar   |
|     < 3.0          |────────────────────────────────( Low_Press_Alarm )
|  (Very Low)        |
|  DB.Pressure_Bar   |
|     > 7.0          |────────────────────────────────( High_Press_Alarm )
|  (Very High)       |
```

---

## Step 243: Motor Maintenance Tracking

### Run Hours Counter

```ladder
// Motor Maintenance Counter
// แจ้งเตือนเมื่อถึงกำหนด Preventive Maintenance

// Network 1: Run Hour Timer
|  %Q0.0    |
|Motor_Run  |────────────────────────────────────────[TON T_1Min]
|           |                                         PT=T#1M

|  T_1Min.Q |
|Timer_1Min |────────────────────────────────────────[INC DB.Run_Min]
|           |────────────────────────────────────────[R T_1Min]

// Network 2: Convert to Hours
|  Always   |
|  ON       |──[DIV]──────────────────────────────────────────────┤
|           │  IN1: DB.Run_Min                                    │
|           │  IN2: 60                                            │
|           │  OUT: DB.Run_Hours                                  │

// Network 3: Maintenance Alert at 5000 hours
|  DB.Run_Hours |
|    >= 5000    |────────────────────────────────────( PM_Due_Alarm )
|               |────────────────────────────────────( %Q0.7 )
|               |                                    PM_Lamp

// Network 4: Reset Counter After PM
|  PM_Reset_PB  |
|               |────────────────────────────────────[MOVE]
|               │  IN: 0                                          │
|               │  OUT: DB.Run_Hours                             │
```

---

## Step 244: Motor Starter Selection Guide

### การเลือก Motor Starter

```
Motor Starter Selection:

Motor Power:
< 5.5kW   → DOL Starter (เหมาะสม)
5.5-30kW  → Star-Delta หรือ Soft Starter
> 30kW    → Soft Starter หรือ VFD
All       → VFD (ถ้าต้องการ Speed Control)

Application:
Pump      → Soft Starter หรือ VFD
Fan       → VFD (ประหยัดพลังงาน)
Conveyor  → DOL หรือ VFD
Compressor→ Soft Starter หรือ Star-Delta
Crane     → VFD (Speed + Braking Control)
```

### ตารางเปรียบเทียบ Starters

| คุณสมบัติ | DOL | Star-Delta | Soft Starter | VFD |
|-----------|-----|------------|--------------|-----|
| ราคา | ถูก | ปานกลาง | แพง | แพงมาก |
| กระแส Inrush | 7× In | 2.3× In | 1.5-4× In | ≤ In |
| Torque ขณะสตาร์ท | สูง | ต่ำ | ปรับได้ | ปรับได้ |
| Speed Control | ✗ | ✗ | ✗ | ✓ |
| ประหยัดพลังงาน | ✗ | ✗ | เล็กน้อย | สูงมาก |
| ความซับซ้อน | ต่ำ | ปานกลาง | ปานกลาง | สูง |

---

## Step 245: Addressing สำหรับ Motor Control

### Siemens S7-1200/1500

```
Digital Inputs:
%I0.0 = PB_START_M1
%I0.1 = PB_STOP_M1
%I0.2 = ESTOP_1 (NC)
%I0.3 = OL_RELAY_M1 (NC)
%I0.4 = PB_FWD_M1
%I0.5 = PB_REV_M1
%I0.6 = VFD_FAULT
%I0.7 = VFD_RUNNING

Digital Outputs:
%Q0.0 = KM1_MAIN (DOL)
%Q0.1 = KM2_STAR (Star-Delta)
%Q0.2 = KM3_DELTA (Star-Delta)
%Q0.3 = KM4_FWD (Forward)
%Q0.4 = KM5_REV (Reverse)
%Q0.5 = VFD_RUN
%Q0.6 = VFD_RESET
%Q0.7 = RUN_LAMP

Analog:
%IW64 = VFD_SPEED_FB (4-20mA)
%QW80 = VFD_SPEED_SP (0-10V)
```

### Allen-Bradley ControlLogix

```
Tag-based addressing (ไม่ใช้ Physical Address):

Motor1_Start_PB     : BOOL (DI: Local:1:I.Data.0)
Motor1_Stop_PB      : BOOL (DI: Local:1:I.Data.1)
Motor1_ESTOP        : BOOL (DI: Local:1:I.Data.2)
Motor1_OL           : BOOL (DI: Local:1:I.Data.3)
Motor1_Run_Cmd      : BOOL (DO: Local:2:O.Data.0)
Motor1_Run_Lamp     : BOOL (DO: Local:2:O.Data.1)
VFD_Speed_SP        : REAL (AO: Local:3:O.Channel[0].Data)
VFD_Speed_FB        : REAL (AI: Local:4:I.Channel[0].Data)
```

### Mitsubishi FX3U

```
Input (X address - Octal):
X0  = PB_START
X1  = PB_STOP
X2  = ESTOP (NC)
X3  = OL (NC)
X4  = PB_FWD
X5  = PB_REV
X6  = VFD_FAULT
X7  = VFD_AT_SPEED
X10 = LS_HOME (Octal!)

Output (Y address - Octal):
Y0  = KM1_MAIN
Y1  = KM2_STAR
Y2  = KM3_DELTA
Y3  = KM4_FWD
Y4  = KM5_REV
Y5  = VFD_RUN
Y6  = RUN_LAMP
Y7  = FAULT_LAMP

Data Registers:
D0  = VFD Speed Setpoint (0-4000)
D2  = VFD Speed Feedback
D100= Motor 1 Run Hours (Low Word)
D101= Motor 1 Run Hours (High Word)
```

---

## Step 246: Troubleshooting Motor Control

### Motor Not Starting - Troubleshooting Guide

```
Motor Not Starting:
         │
         ▼
  Check PLC Output
  (LED หรือ Status)
         │
    ON   │   OFF
         │     │
    ▼    │     ▼
Check   │  Check PLC
Contactor│  Program
Coil    │  Input/Logic
Voltage │
         │
    OK   │  NO Voltage
         │
    ▼    │    ▼
 Motor   │  Contactor
  OK?    │  Coil Voltage
         │  (24V Check)
   No    │
    │    │
    ▼    │
Check OL │
Setting  │
 (Too    │
  Low?)  │
```

### Common Motor Faults and Solutions

```
Fault: Motor trips OL relay repeatedly
Cause: OL setting too low, Mechanical overload, Phase imbalance
Solution: 
1. วัดกระแสแต่ละเฟสด้วย Clamp Meter
2. เปรียบเทียบกับ Nameplate Current
3. ตรวจ Mechanical Load
4. ตรวจ Bearing

Fault: Motor humming but not starting
Cause: Single phasing (Phase Loss)
Solution:
1. วัดแรงดัน 3 เฟส ที่ Motor Terminal
2. ตรวจ Fuse ทุกตัว
3. ตรวจ Contactor Contact

Fault: Motor running hot
Cause: Overload, Poor ventilation, Winding fault
Solution:
1. วัด Temperature ที่ Frame
2. ตรวจ Current vs Rated
3. Megohmeter test (Insulation)
```

---

## Step 247: Motor Control Best Practices

### Best Practices สำหรับ Motor Control

```
1. Always use Hardware Interlock
   - FWD และ REV Contactors
   - ไม่พึ่งแต่ Software Interlock

2. Contactor Ratings
   - เลือก Contactor AC-3 Category
   - Rating ≥ Motor FLA × 1.25

3. Overload Relay Setting
   - Set เท่ากับ Motor Nameplate Current
   - ใช้ Class 10 สำหรับงานทั่วไป

4. MCB/MCCB Protection
   - Short Circuit Protection
   - Rating: 10× Motor FLA

5. Control Transformer
   - แยก Control Voltage จาก Power Voltage
   - ใช้ 24VDC หรือ 110VAC

6. Emergency Stop
   - NC Contact ต่อ Series
   - ต้องมี Safety Rating (IEC 60947-5-1)

7. Grounding
   - Motor Frame → Ground
   - VFD Ground ให้ดี (EMC)
```

---

## Step 248: VFD Parameter Setting

### การตั้งค่า VFD พื้นฐาน

```
VFD Parameters (ตัวอย่าง ABB ACS550):

Parameter  │ Name              │ Setting    │ หมายเหตุ
───────────┼───────────────────┼────────────┼────────────────
P1.01      │ Motor Voltage     │ 380V       │ Motor Nameplate
P1.02      │ Motor Frequency   │ 50Hz       │
P1.03      │ Motor Current     │ 13.7A      │ FLA
P1.04      │ Motor Speed       │ 1450 RPM   │
P1.05      │ Motor Power       │ 5.5kW      │
P1.06      │ Motor Power Factor│ 0.82       │
───────────┼───────────────────┼────────────┼────────────────
P2.01      │ Control Mode      │ 1 (Local)  │ 3 = PLC Control
P2.02      │ Reference Source  │ 1 (AI1)    │ 0-10V Analog
P2.03      │ Start/Stop Source │ 1 (DI1/DI2)│ Digital Input
───────────┼───────────────────┼────────────┼────────────────
P3.01      │ Accel Time 1      │ 5s         │
P3.02      │ Decel Time 1      │ 5s         │
P3.03      │ Min Frequency     │ 5Hz        │ Minimum Speed
P3.04      │ Max Frequency     │ 50Hz       │ Maximum Speed
───────────┼───────────────────┼────────────┼────────────────
P4.01      │ OL Current Limit  │ 150%       │ Current Limit
P4.02      │ OL Trip Time      │ 10s        │
```

---

## Step 249: Real-World Application Example

### โรงสีข้าว (Rice Mill) Motor Control

```
Rice Mill Motor Control System:

Sequence Start (Feed Mill):
1. Conveyor 3 (Output) → Start first
2. Wait 3 seconds
3. Elevator → Start
4. Wait 3 seconds  
5. Conveyor 1 (Input) → Start
6. Wait 3 seconds
7. Rice Husker → Start

Sequence Stop (Feed Mill):
1. Rice Husker → Stop first
2. Wait until Empty (10s)
3. Conveyor 1 → Stop
4. Wait 5s
5. Elevator → Stop
6. Wait 5s
7. Conveyor 3 → Stop

Emergency: ALL STOP immediately
```

```ladder
// Rice Mill Sequence Control (simplified)
// Siemens S7-1200

// Network 1: Sequence Start SM
|  %M_StartSeq |  %M_Stop |
|              |  (NC)    |──────────────────────────[SFC_Start]
|              |          |                           (State Machine)
|              |
|  SFC State = 1 (Conv3 Start):
|  %M_StartSeq |
|              |──────────────────────────────────────( %Q0.3 )
|              |                                       CONV3_Run
|              |─────────────────────────────────[TON T_Seq1]
|              |                                  PT=T#3S

// State 2 (Elevator):
|  T_Seq1.Q   |
|             |──────────────────────────────────────( %Q0.2 )
|             |                                       ELEVATOR_Run
|             |─────────────────────────────────[TON T_Seq2]
|             |                                  PT=T#3S
// ... (ต่อไปสำหรับ Motors อื่นๆ)
```

---

## Step 250: Lab Project - Complete Motor Control Panel

### Lab 13: สร้าง Motor Control Panel

**โจทย์:** ออกแบบและสร้าง Motor Control Panel สำหรับมอเตอร์ 1.5kW DOL + 5.5kW Star-Delta

**Specification:**
```
Motor 1 (Pump):
- Type: 1.5kW, 380V, 3-Phase
- Starter: DOL
- Control: Start/Stop, Forward/Reverse
- Protection: MCB + OL Relay

Motor 2 (Fan):
- Type: 5.5kW, 380V, 3-Phase
- Starter: Star-Delta (8s transition)
- Control: Start/Stop
- Protection: MCB + OL Relay

Common:
- Emergency Stop (Category 3)
- PLC: Siemens S7-1200
- HMI: KTP400 (Optional)
```

**I/O List:**
```
Inputs:
I0.0 = ESTOP_1 (NC)    I0.4 = PB_M2_STOP
I0.1 = ESTOP_2 (NC)    I0.5 = OL_M1 (NC)
I0.2 = PB_M1_START     I0.6 = OL_M2 (NC)
I0.3 = PB_M1_STOP      I0.7 = SPARE

Outputs:
Q0.0 = KM1 (M1 Main)   Q0.4 = KM4 (M2 Star)
Q0.1 = KM2 (M1 FWD)    Q0.5 = KM5 (M2 Delta)
Q0.2 = KM3 (M1 REV)    Q0.6 = M1 Run Lamp
Q0.3 = KM6 (M2 Main)   Q0.7 = M2 Run Lamp
```

**Ladder Program สมบูรณ์:**
```ladder
// Complete Motor Control Program
// Lab 13

// ============ MOTOR 1 (DOL) ============

// Network 1: M1 Forward
|  %I0.0   |  %I0.1   |  %I0.5   |  %I0.3   |  %I0.2   |  %Q0.2  |
|ESTOP1_NC |ESTOP2_NC |OL_M1_NC  |M1_STP_NC |M1_START  |M1_REV_NC|──( %Q0.0 )
|          |          |          |          |          |H/W Lock |  KM1_Main
|  %Q0.0   |
|KM1_Main  |──────────────────────────────────────────────────────────┘

|  %Q0.0   |  %Q0.2   |  %I0.2   |
|KM1_Main  |REV_NC    |PB_FWD    |────────────────────────────────( %Q0.1 )
|          |H/W Inter.|          |                                KM2_FWD
|  %Q0.1   |
|KM2_FWD   |──────────────────────────────────────────────────────────┘

|  %Q0.0   |  %Q0.1   |  %I0.2   |
|KM1_Main  |FWD_NC    |PB_REV    |────────────────────────────────( %Q0.2 )
|          |H/W Inter.|          |                                KM3_REV
|  %Q0.2   |
|KM3_REV   |──────────────────────────────────────────────────────────┘

// ============ MOTOR 2 (STAR-DELTA) ============

// Network 2: M2 Main
|  %I0.0   |  %I0.1   |  %I0.6   |  %I0.4   |  M2_START|
|ESTOP1_NC |ESTOP2_NC |OL_M2_NC  |M2_STP_NC |PB_M2     |──────────────( %Q0.3 )
|          |          |          |          |          |             KM6_M2Main
|  %Q0.3   |
|KM6_Main  |──────────────────────────────────────────────────────────┘

// Network 3: M2 Star (with Interlock)
|  %Q0.3   |  %Q0.5   |
|KM6_Main  |KM5_Delta |──────────────────────────────────────────( %Q0.4 )
|          |  (NC)    |                                          KM4_Star

// Network 4: Star-to-Delta Timer
|  %Q0.4   |
|KM4_Star  |─────────────────────────────────────────────[TON T_SD]
|          |                                              PT=T#8S

// Network 5: Delta (after timer)
|  %T_SD.Q |  %Q0.4   |
|Timer_Done|KM4_Star  |──────────────────────────────────────────( %Q0.5 )
|          | (NC)     |                                          KM5_Delta
```

**การประเมินผล Lab:**
- ✓ I/O List ครบและถูกต้อง
- ✓ DOL Start/Stop ทำงาน
- ✓ Forward/Reverse ทำงาน (Hardware Interlock ด้วย)
- ✓ Star-Delta ทำงาน (Timer 8s)
- ✓ E-Stop หยุดทั้งสองมอเตอร์
- ✓ OL Trip ทำงาน
- ✓ สาย Label ชัดเจน

---

## สรุป Part 13

ในบทนี้เราได้เรียนรู้:
1. มอเตอร์ 3 เฟสพื้นฐาน และการต่อ Star/Delta
2. Direct On-Line (DOL) Starter
3. Star-Delta Starter (ลดกระแส Inrush)
4. Soft Starter (ควบคุม Voltage Ramp)
5. VFD (ควบคุม Frequency และ Voltage)
6. Forward/Reverse Control พร้อม Interlock
7. Motor Protection Relay (Thermal OL)
8. Motor Control Center (MCC) Layout
9. Troubleshooting Motor Control
10. Real-world Application (Rice Mill)

---

## การนำทาง

- [← Part 12: Safety พื้นฐาน](part12_safety_basics.md)
- [→ Part 14: ระบบ Conveyor พื้นฐาน](part14_conveyor_system.md)
- [กลับหน้าหลัก](../README.md)

---

*Part 13 จบ | ต่อไป Part 14: ระบบ Conveyor พื้นฐาน*
