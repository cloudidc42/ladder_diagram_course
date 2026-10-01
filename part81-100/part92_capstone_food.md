# Part 92: Capstone Project - Food Processing Plant
## Steps 1821-1840 | ระดับ Capstone

---

## การนำทาง
- [← Part 91: Capstone - Automotive Assembly Line](part91_capstone_automotive.md)
- [→ Part 93: Capstone - Water Treatment Plant](part93_capstone_waterworks.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1821: Project Overview - Bakery Factory

### โปรเจกต์: Industrial Bakery - Automated Bread Production

```
Project: ระบบผลิตขนมปังอัตโนมัติ (Industrial Bread Plant)
Capacity: 12,000 Loaves/Hour
Products: White Bread, Whole Wheat, Multigrain (3 SKUs)
Plant Area: 4,500 m²
Certifications: GMP, HACCP, ISO 22000, BRC Grade A

Production Processes:
1. Raw Material Receiving & Storage
2. Ingredient Weighing & Dosing
3. Mixing & Fermentation
4. Dividing & Rounding
5. Intermediate Proofing
6. Moulding & Panning
7. Final Proofing (38°C, 85%RH, 90 min)
8. Baking (220°C, 22 min)
9. Cooling (Spiral Cooler, 90 min)
10. Slicing & Packaging
11. Metal Detection & X-Ray
12. Palletizing & Wrapping

PLC Platform: Allen-Bradley ControlLogix 5580
Safety: GuardLogix 5580
HMI: FactoryTalk View SE (Server) + PanelView Plus 7 (Local)
SCADA: FactoryTalk Historian + FactoryTalk Analytics
MES: Rockwell Plex MES (Cloud)
ERP: SAP S/4HANA
```

---

## Step 1822: Recipe Management System

### Recipe Database Structure

```python
# Python - Food Recipe Management System (HACCP Compliant)
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime
import hashlib
import json

@dataclass
class Ingredient:
    """ส่วนผสม"""
    item_code: str
    description: str
    quantity_kg: float
    tolerance_pct: float     # เปอร์เซ็นต์ความเบี่ยงเบนที่ยอมรับ
    is_allergen: bool = False
    allergen_type: Optional[str] = None  # Gluten, Nuts, Dairy, etc.
    ccp_point: bool = False  # Critical Control Point

@dataclass
class ProcessStep:
    """ขั้นตอนการผลิต"""
    step_no: int
    name: str
    duration_min: int
    temperature_setpoint: Optional[float] = None
    temperature_tolerance: float = 2.0  # ±°C
    humidity_setpoint: Optional[float] = None
    speed_rpm: Optional[float] = None
    ccp: bool = False
    ccp_critical_limit: Optional[Dict] = None

@dataclass
class Recipe:
    """ใบสูตรการผลิต (Master Recipe)"""
    recipe_id: str
    recipe_name: str
    version: str
    product_code: str
    batch_size_kg: float
    approved_by: str
    approved_date: datetime
    status: str  # DRAFT, APPROVED, ACTIVE, OBSOLETE
    
    ingredients: List[Ingredient] = field(default_factory=list)
    process_steps: List[ProcessStep] = field(default_factory=list)
    
    # Quality Targets
    target_moisture_pct: float = 38.0
    moisture_tolerance: float = 2.0
    target_weight_g: float = 450.0
    weight_tolerance_g: float = 10.0
    
    # Audit Trail
    created_by: str = ""
    created_date: Optional[datetime] = None
    last_modified_by: str = ""
    last_modified_date: Optional[datetime] = None
    
    def get_hash(self) -> str:
        """SHA-256 hash ของ Recipe สำหรับ Integrity Check"""
        recipe_str = json.dumps({
            'id': self.recipe_id,
            'version': self.version,
            'ingredients': [(i.item_code, i.quantity_kg) for i in self.ingredients],
            'steps': [(s.step_no, s.duration_min, s.temperature_setpoint) for s in self.process_steps]
        }, sort_keys=True)
        return hashlib.sha256(recipe_str.encode()).hexdigest()

class RecipeManagementSystem:
    def __init__(self, db, mes_client, plc_writer):
        self.db = db
        self.mes = mes_client
        self.plc = plc_writer
    
    def download_recipe_to_plc(self, recipe_id: str, batch_no: str):
        """ดาวน์โหลด Recipe ไปยัง PLC"""
        recipe = self.db.get_recipe(recipe_id)
        
        if recipe.status != 'APPROVED':
            raise ValueError(f"Recipe {recipe_id} ไม่ได้รับการอนุมัติ")
        
        # ตรวจสอบ Integrity
        stored_hash = self.db.get_recipe_hash(recipe_id)
        if recipe.get_hash() != stored_hash:
            raise SecurityError("Recipe integrity check ล้มเหลว!")
        
        # แปลงเป็น PLC Data Format (ControlLogix UDT)
        plc_recipe = self.convert_to_plc_format(recipe)
        
        # เขียนไปยัง PLC Data Table
        self.plc.write_udt('RecipeTable[0]', plc_recipe)
        
        # บันทึก Audit Log
        self.log_recipe_download(recipe_id, batch_no)
        
        return True
    
    def convert_to_plc_format(self, recipe: Recipe) -> dict:
        """แปลง Recipe เป็น PLC UDT Format"""
        plc_data = {
            'RecipeID': int(recipe.recipe_id.replace('R', '')),
            'BatchSizeKg': recipe.batch_size_kg,
            'TargetWeight': recipe.target_weight_g,
            'WeightTol': recipe.weight_tolerance_g,
            'TargetMoisture': recipe.target_moisture_pct,
            'MoistureTol': recipe.moisture_tolerance,
            'NumIngredients': len(recipe.ingredients),
            'NumSteps': len(recipe.process_steps)
        }
        
        # Ingredient Array
        for idx, ing in enumerate(recipe.ingredients[:20]):  # Max 20 ingredients
            plc_data[f'Ingredient[{idx}].ItemCode'] = ing.item_code
            plc_data[f'Ingredient[{idx}].Quantity'] = ing.quantity_kg
            plc_data[f'Ingredient[{idx}].Tolerance'] = ing.tolerance_pct
        
        # Process Steps Array
        for idx, step in enumerate(recipe.process_steps[:15]):  # Max 15 steps
            plc_data[f'Step[{idx}].Duration'] = step.duration_min
            plc_data[f'Step[{idx}].TempSP'] = step.temperature_setpoint or 0
            plc_data[f'Step[{idx}].HumidSP'] = step.humidity_setpoint or 0
            plc_data[f'Step[{idx}].SpeedSP'] = step.speed_rpm or 0
        
        return plc_data
```

---

## Step 1823: CIP System (Cleaning In Place)

### ระบบ CIP สำหรับ Food Processing

```ladder
// ============================================
// CIP Control System - Bread Mixer
// Allen-Bradley ControlLogix 5580
// Structured Text (IEC 61131-3)
// ============================================

PROGRAM CIP_Control
VAR
    CIP_Phase   : INT;
    CIP_Timer   : TON;
    CIP_Temp    : REAL;
    
    // CIP Parameters (จาก Recipe)
    PreRinse_Time    : TIME := T#5M;
    Caustic_Conc     : REAL := 2.0;     // 2% NaOH
    Caustic_Temp     : REAL := 75.0;    // 75°C
    Caustic_Time     : TIME := T#20M;
    Rinse1_Time      : TIME := T#10M;
    Acid_Conc        : REAL := 1.5;     // 1.5% HNO3
    Acid_Temp        : REAL := 65.0;    // 65°C
    Acid_Time        : TIME := T#15M;
    FinalRinse_Time  : TIME := T#10M;
    
    // Validation
    Caustic_Verified  : BOOL;
    Acid_Verified     : BOOL;
    Rinse_Verified    : BOOL;
    Conductivity_OK   : BOOL;
END_VAR

CASE CIP_Phase OF
    0:  // IDLE
        IF CIP_Start THEN
            CIP_Phase := 10;
            Log_CIP_Start(Equipment_ID, Operator_ID, Timestamp());
        END_IF
    
    10: // PRE-RINSE with Water
        Valve_Water := TRUE;
        Pump_CIP := TRUE;
        CIP_Timer(IN := TRUE, PT := PreRinse_Time);
        IF CIP_Timer.Q THEN
            CIP_Timer(IN := FALSE);
            CIP_Phase := 20;
        END_IF
    
    20: // CAUSTIC WASH (2% NaOH @ 75°C)
        Valve_Caustic := TRUE;
        Heater_CIP := TRUE;
        // ตรวจสอบ Temperature
        IF CIP_Temp >= Caustic_Temp - 2.0 THEN
            CIP_Timer(IN := TRUE, PT := Caustic_Time);
        END_IF
        // ตรวจสอบ Conductivity (วัด NaOH Concentration)
        Caustic_Verified := (Conductivity_mS > CAUSTIC_COND_MIN AND
                             Conductivity_mS < CAUSTIC_COND_MAX);
        IF CIP_Timer.Q AND Caustic_Verified THEN
            CIP_Timer(IN := FALSE);
            Valve_Caustic := FALSE;
            CIP_Phase := 30;
        END_IF
    
    30: // INTERMEDIATE RINSE
        Valve_Water := TRUE;
        CIP_Timer(IN := TRUE, PT := Rinse1_Time);
        // ตรวจสอบ Conductivity ว่า Caustic หมดแล้ว
        Rinse_Verified := (Conductivity_mS < RINSE_COND_MAX);
        IF CIP_Timer.Q AND Rinse_Verified THEN
            CIP_Timer(IN := FALSE);
            CIP_Phase := 40;
        END_IF
    
    40: // ACID WASH (1.5% HNO3 @ 65°C)
        Valve_Acid := TRUE;
        Heater_CIP := TRUE;
        IF CIP_Temp >= Acid_Temp - 2.0 THEN
            CIP_Timer(IN := TRUE, PT := Acid_Time);
        END_IF
        Acid_Verified := (pH_Value > ACID_PH_MIN AND pH_Value < ACID_PH_MAX);
        IF CIP_Timer.Q AND Acid_Verified THEN
            CIP_Timer(IN := FALSE);
            Valve_Acid := FALSE;
            CIP_Phase := 50;
        END_IF
    
    50: // FINAL RINSE (Hot Water 80°C)
        Valve_HotWater := TRUE;
        CIP_Timer(IN := TRUE, PT := FinalRinse_Time);
        // Final Conductivity Check
        Conductivity_OK := (Conductivity_mS < FINAL_COND_MAX);  // < 50 µS/cm
        IF CIP_Timer.Q AND Conductivity_OK THEN
            CIP_Timer(IN := FALSE);
            CIP_Phase := 60;
        END_IF
    
    60: // CIP COMPLETE
        Pump_CIP := FALSE;
        Heater_CIP := FALSE;
        // บันทึก CIP Report
        Log_CIP_Complete(
            Equipment_ID := Equipment_ID,
            Duration := Elapsed_Time,
            Caustic_OK := Caustic_Verified,
            Acid_OK := Acid_Verified,
            Rinse_OK := Conductivity_OK,
            Result := 'PASS'
        );
        CIP_Phase := 0;
END_CASE
```

---

## Step 1824: Batch Control

### ISA-88 Batch Control Implementation

```
ISA-88 Hierarchy:
Enterprise → Site → Area → Process Cell → Unit → Equipment Module → Control Module

Bread Production Cell:
Process Cell: BREAD_LINE_01
  Unit Procedure: BREAD_BATCH
    Operations:
      1. OP_WEIGH        (Ingredient Weighing)
      2. OP_MIX          (Mixing & Fermentation)
      3. OP_DIVIDE       (Dividing & Rounding)
      4. OP_PROOF        (Intermediate Proofing)
      5. OP_MOULD        (Moulding & Panning)
      6. OP_FINAL_PROOF  (Final Proofing)
      7. OP_BAKE         (Baking)
      8. OP_COOL         (Cooling)
      9. OP_PACK         (Slicing & Packaging)
```

```python
# Python - ISA-88 Batch Coordinator
import asyncio
from enum import Enum
from datetime import datetime

class BatchStatus(Enum):
    IDLE = 0
    RUNNING = 1
    PAUSED = 2
    COMPLETE = 3
    ABORTED = 4
    FAILED = 5

class Operation(Enum):
    WEIGH = 1
    MIX = 2
    DIVIDE = 3
    PROOF = 4
    MOULD = 5
    FINAL_PROOF = 6
    BAKE = 7
    COOL = 8
    PACK = 9

class BreadBatchController:
    def __init__(self, recipe, batch_no, plc_client, mes_client):
        self.recipe = recipe
        self.batch_no = batch_no
        self.plc = plc_client
        self.mes = mes_client
        
        self.status = BatchStatus.IDLE
        self.current_op = None
        self.start_time = None
        self.events = []  # Batch Event Journal
    
    async def run_batch(self):
        """Execute Full Batch"""
        self.status = BatchStatus.RUNNING
        self.start_time = datetime.now()
        
        # แจ้ง MES ว่า Batch เริ่ม
        await self.mes.batch_start(self.batch_no, self.recipe.recipe_id)
        self.log_event('BATCH_START', f'Batch {self.batch_no} เริ่มผลิต')
        
        operations = [
            (Operation.WEIGH, self.run_weighing),
            (Operation.MIX, self.run_mixing),
            (Operation.DIVIDE, self.run_dividing),
            (Operation.PROOF, self.run_intermediate_proof),
            (Operation.MOULD, self.run_moulding),
            (Operation.FINAL_PROOF, self.run_final_proof),
            (Operation.BAKE, self.run_baking),
            (Operation.COOL, self.run_cooling),
            (Operation.PACK, self.run_packaging),
        ]
        
        for op, handler in operations:
            self.current_op = op
            self.log_event('OP_START', f'เริ่ม {op.name}')
            
            try:
                result = await handler()
                
                if not result.get('success'):
                    await self.handle_operation_fault(op, result)
                    return
                
                self.log_event('OP_COMPLETE', f'จบ {op.name}: {result}')
                
                # รายงาน Progress ไป MES
                await self.mes.batch_progress(
                    self.batch_no, op.value, op.name, result
                )
                
            except Exception as e:
                self.log_event('OP_FAULT', f'{op.name} ผิดพลาด: {e}')
                self.status = BatchStatus.FAILED
                await self.mes.batch_abort(self.batch_no, str(e))
                return
        
        # Batch Complete
        self.status = BatchStatus.COMPLETE
        total_duration = (datetime.now() - self.start_time).total_seconds() / 60
        
        self.log_event('BATCH_COMPLETE', f'เสร็จสิ้น ใช้เวลา {total_duration:.1f} นาที')
        await self.mes.batch_complete(self.batch_no, self.get_batch_report())
    
    async def run_baking(self) -> dict:
        """ควบคุม Tunnel Oven"""
        recipe_step = self.recipe.get_step('BAKE')
        
        # ตั้งค่า Oven
        await self.plc.write('Oven.Zone1_TempSP', recipe_step.zone1_temp)
        await self.plc.write('Oven.Zone2_TempSP', recipe_step.zone2_temp)
        await self.plc.write('Oven.Zone3_TempSP', recipe_step.zone3_temp)
        await self.plc.write('Oven.Belt_Speed', recipe_step.belt_speed)
        await self.plc.write('Oven.Steam_Valve', recipe_step.steam_pct)
        
        # รอ Oven Stable
        stable = await self.wait_for_oven_stable(timeout_min=15)
        if not stable:
            return {'success': False, 'reason': 'Oven ไม่ถึง Setpoint ใน 15 นาที'}
        
        # บันทึก Process Data ตลอดการอบ
        bake_start = datetime.now()
        samples = []
        
        while (datetime.now() - bake_start).total_seconds() < recipe_step.duration_min * 60:
            sample = {
                'time': datetime.now().isoformat(),
                'zone1_temp': await self.plc.read('Oven.Zone1_TempPV'),
                'zone2_temp': await self.plc.read('Oven.Zone2_TempPV'),
                'zone3_temp': await self.plc.read('Oven.Zone3_TempPV'),
                'steam': await self.plc.read('Oven.Steam_PV'),
                'belt_speed': await self.plc.read('Oven.Belt_SpeedPV'),
            }
            samples.append(sample)
            
            # ตรวจสอบ CCP (Critical Control Point)
            # HACCP: อุณหภูมิ > 210°C เพื่อฆ่าเชื้อ Salmonella
            min_temp = min(sample['zone1_temp'], sample['zone2_temp'], sample['zone3_temp'])
            if min_temp < 210.0:
                self.log_ccp_deviation('OVEN_TEMP', min_temp, 210.0)
                # อาจต้อง Hold Batch
        
        return {
            'success': True,
            'duration_actual_min': recipe_step.duration_min,
            'avg_zone2_temp': sum(s['zone2_temp'] for s in samples) / len(samples),
            'ccp_violations': 0,
            'process_samples': len(samples)
        }
    
    def log_event(self, event_type: str, message: str):
        event = {
            'batch_no': self.batch_no,
            'timestamp': datetime.now().isoformat(),
            'operation': self.current_op.name if self.current_op else 'BATCH',
            'type': event_type,
            'message': message,
            'operator': self.get_current_operator()
        }
        self.events.append(event)
        # บันทึกลง Database (Electronic Batch Record)
        self.mes.log_batch_event(event)
    
    def get_batch_report(self) -> dict:
        return {
            'batch_no': self.batch_no,
            'recipe_id': self.recipe.recipe_id,
            'start_time': self.start_time.isoformat(),
            'end_time': datetime.now().isoformat(),
            'status': self.status.name,
            'events': self.events
        }
```

---

## Step 1825: Weighing and Dosing

### Automatic Weighing System

```ladder
// ============================================
// Automatic Weighing & Dosing System
// Allen-Bradley ControlLogix
// Function Block Diagram
// ============================================

// NETWORK 1: Recipe Ingredient Request
|----[Batch_Phase = WEIGH]----[Weigh_Step_Active]-----|
|   // โหลดรายการส่วนผสมจาก Recipe               |
|   MOVE Recipe.Ingredient[Weigh_Index].Code, Current_Item_Code   |
|   MOVE Recipe.Ingredient[Weigh_Index].Quantity, Target_Weight   |
|   MOVE Recipe.Ingredient[Weigh_Index].Tolerance, Weight_Tol     |
|-----------------------------------------------------|

// NETWORK 2: Hopper Selection
|----[Weigh_Start]-----|
|   // หา Hopper ที่มีส่วนผสมที่ต้องการ              |
|   CALL FB_Select_Hopper                             |
|   Ingredient_Code := Current_Item_Code              |
|   Output → Selected_Hopper_No                       |
|---------------------|

// NETWORK 3: Rough Fill (เร็ว - 80% ของ Target)
|----[Hopper_Selected]----[Tare_Complete]-----|
|   // เปิด Valve หรือ Screw Speed เต็ม          |
|   CALL FB_Dosing_Control                   |
|   Target := Target_Weight * 0.80           |
|   Speed := FAST_SPEED                      |
|   Output: Rough_Fill_Done                  |
|---------------------------------------------|

// NETWORK 4: Fine Fill (ช้า - 20% ที่เหลือ)
|----[Rough_Fill_Done]-----|
|   // ลด Speed ลง เพื่อความแม่นยำ               |
|   CALL FB_Dosing_Control                  |
|   Target := Target_Weight                 |
|   Speed := SLOW_SPEED                     |
|   Early_Cutoff := Cutoff_Factor * Flow_Rate_Estimated |
|   Output: Dosing_Complete                 |
|--------------------------|

// NETWORK 5: Weight Check
|----[Dosing_Complete]-----|
|   Actual_Weight := Scale_Reading                    |
|   Min_OK := Target_Weight * (1 - Weight_Tol/100)   |
|   Max_OK := Target_Weight * (1 + Weight_Tol/100)   |
|   Weight_OK := (Actual_Weight >= Min_OK) AND        |
|                (Actual_Weight <= Max_OK)             |
|   SET Weight_Verified                               |
|   // บันทึกค่าลง Batch Record                      |
|   WRITE DB_Batch.Ingredient[Weigh_Index].ActualWt := Actual_Weight |
|--------------------------|

// NETWORK 6: Advance to Next Ingredient
|----[Weight_Verified]----[Weight_OK]----[Not Last_Ingredient]-----|
|   ADD Weigh_Index, 1                                              |
|   RST Weight_Verified                                             |
|------------------------------------------------------------------|

|----[Weight_Verified]----[Weight_OK]----[Last_Ingredient]--------|
|   SET All_Ingredients_Weighed                                    |
|   CALL Send_MES_Weight_Complete(Batch_No, Total_Weight)          |
|------------------------------------------------------------------|

// NETWORK 7: Weight Reject
|----[Weight_Verified]----[NOT Weight_OK]-----|
|   SET Weight_Reject_Alarm                   |
|   CALL Log_Weight_Deviation(                |
|       Item_Code := Current_Item_Code,       |
|       Target := Target_Weight,              |
|       Actual := Actual_Weight,              |
|       Deviation_Pct := ((Actual_Weight - Target_Weight) / Target_Weight) * 100) |
|---------------------------------------------|
```

---

## Step 1826: HACCP Critical Control Points

### HACCP CCP Monitoring

```python
# Python - HACCP CCP Monitor (ระบบติดตาม จุดควบคุมวิกฤต)
from datetime import datetime
from dataclasses import dataclass
from typing import List, Callable

@dataclass
class CCP:
    """Critical Control Point"""
    ccp_id: str
    name: str
    hazard: str           # Biological, Chemical, Physical
    critical_limit_min: float
    critical_limit_max: float
    unit: str
    monitoring_freq_sec: int  # ความถี่การตรวจสอบ
    corrective_action: str    # Corrective Action ที่ต้องทำ
    verification_freq: str    # daily, weekly, etc.

class HACCPMonitor:
    """ระบบ HACCP Monitoring สำหรับ Bakery"""
    
    def __init__(self, plc_reader, db, notification_service):
        self.plc = plc_reader
        self.db = db
        self.notify = notification_service
        
        # กำหนด CCPs ของ Bakery
        self.ccps = [
            CCP(
                ccp_id='CCP-01',
                name='Oven Temperature',
                hazard='Biological',
                critical_limit_min=210.0,
                critical_limit_max=260.0,
                unit='°C',
                monitoring_freq_sec=30,
                corrective_action='หยุด Production, ตรวจสอบ Oven, ประเมิน Product ที่ผลิตไปแล้ว',
                verification_freq='daily'
            ),
            CCP(
                ccp_id='CCP-02',
                name='Metal Detector Sensitivity',
                hazard='Physical',
                critical_limit_min=0.0,
                critical_limit_max=999.0,
                unit='Fe mm',
                monitoring_freq_sec=5,  # ทดสอบทุก 5 วินาที
                corrective_action='Reject ชิ้นงาน, ตรวจสอบ Metal Detector ทันที',
                verification_freq='every_shift'
            ),
            CCP(
                ccp_id='CCP-03',
                name='Proofing Temperature',
                hazard='Biological',
                critical_limit_min=36.0,
                critical_limit_max=40.0,
                unit='°C',
                monitoring_freq_sec=60,
                corrective_action='ปรับ Temperature, ประเมิน Fermentation ของ Batch',
                verification_freq='daily'
            ),
            CCP(
                ccp_id='CCP-04',
                name='Baked Product Core Temperature',
                hazard='Biological',
                critical_limit_min=95.0,
                critical_limit_max=999.0,
                unit='°C',
                monitoring_freq_sec=60,
                corrective_action='Reject Batch, Rework หรือ Dispose',
                verification_freq='daily'
            ),
        ]
        
        self.ccp_violations = []  # Log of CCP Violations
    
    async def monitor_all_ccps(self):
        """ติดตาม CCPs ทั้งหมดอย่างต่อเนื่อง"""
        while True:
            for ccp in self.ccps:
                await self.check_ccp(ccp)
            await asyncio.sleep(1)
    
    async def check_ccp(self, ccp: CCP):
        """ตรวจสอบ CCP แต่ละจุด"""
        
        # อ่านค่าจาก PLC
        tag_map = {
            'CCP-01': 'Oven.Zone2_Temp_PV',
            'CCP-02': 'MetalDetector.Last_TestResult',
            'CCP-03': 'ProofingRoom.Temperature_PV',
            'CCP-04': 'Oven.Product_CoreTemp_PV',
        }
        
        current_value = await self.plc.read(tag_map[ccp.ccp_id])
        timestamp = datetime.now()
        
        # ตรวจสอบ Critical Limits
        violation = False
        violation_type = None
        
        if current_value < ccp.critical_limit_min:
            violation = True
            violation_type = 'BELOW_LOWER_LIMIT'
        elif ccp.critical_limit_max < 999.0 and current_value > ccp.critical_limit_max:
            violation = True
            violation_type = 'ABOVE_UPPER_LIMIT'
        
        # บันทึก Monitoring Record (ต้องทำทุกครั้ง ไม่ใช่แค่เมื่อ Violation)
        record = {
            'ccp_id': ccp.ccp_id,
            'timestamp': timestamp.isoformat(),
            'value': current_value,
            'unit': ccp.unit,
            'limit_min': ccp.critical_limit_min,
            'limit_max': ccp.critical_limit_max,
            'within_limits': not violation,
            'batch_no': await self.plc.read('Batch.Current_Batch_No'),
            'operator': await self.plc.read('HMI.Current_Operator')
        }
        
        await self.db.insert('haccp_ccp_log', record)
        
        if violation:
            await self.handle_ccp_violation(ccp, current_value, violation_type, record)
    
    async def handle_ccp_violation(self, ccp, value, violation_type, record):
        """จัดการเมื่อเกิด CCP Violation"""
        
        # 1. บันทึก Violation
        violation = {
            **record,
            'violation_type': violation_type,
            'corrective_action_required': ccp.corrective_action,
            'status': 'OPEN'
        }
        violation_id = await self.db.insert('haccp_violations', violation)
        
        # 2. สั่ง PLC ให้ Reject/Hold
        if ccp.ccp_id == 'CCP-02':  # Metal Detector
            await self.plc.write('MetalDetector.Reject_Valve', True)
        elif ccp.ccp_id == 'CCP-01':  # Oven Temp
            await self.plc.write('Batch.Hold_Request', True)
        
        # 3. แจ้งเตือนทันที
        await self.notify.send_urgent(
            to=['QA_Supervisor', 'Production_Manager'],
            subject=f'CCP VIOLATION: {ccp.ccp_id} - {ccp.name}',
            body=f"""
CCP Violation Detected!
CCP: {ccp.ccp_id} - {ccp.name}
ค่าที่วัดได้: {value:.1f} {ccp.unit}
Limit: {ccp.critical_limit_min:.1f} - {ccp.critical_limit_max:.1f} {ccp.unit}
Corrective Action: {ccp.corrective_action}
Violation ID: {violation_id}
"""
        )
        
        self.ccp_violations.append(violation)
```

---

## Step 1827: Track and Trace (GS1 Traceability)

### Lot Traceability System

```python
# Python - GS1 Food Traceability System
class FoodTraceabilitySystem:
    """ระบบ Track & Trace สำหรับ Food Safety"""
    
    def generate_gtin_sscc(self, product_code: str, batch_no: str,
                            best_before: str, quantity: int) -> dict:
        """สร้าง GS1 Barcode ข้อมูลสำหรับ Label"""
        
        gtin = f'8850123{product_code:05d}'  # GS1-128 GTIN-14
        
        # SSCC (Serial Shipping Container Code)
        sscc = f'006{self.company_prefix}{batch_no[-8:]}{self.calc_check_digit(f"006{self.company_prefix}{batch_no[-8:]}")}'
        
        # GS1-128 Application Identifiers
        ai_data = {
            'AI_01': ('01', gtin),           # GTIN
            'AI_10': ('10', batch_no),       # Batch/Lot Number
            'AI_17': ('17', best_before),    # Best Before Date (YYMMDD)
            'AI_37': ('37', str(quantity)),  # Quantity
            'AI_00': ('00', sscc),           # SSCC (Pallet)
        }
        
        return {
            'gtin': gtin,
            'sscc': sscc,
            'batch_no': batch_no,
            'best_before': best_before,
            'ai_data': ai_data,
            'barcode_data': self.encode_gs1_128(ai_data)
        }
    
    async def trace_product(self, barcode: str) -> dict:
        """ติดตาม Product จาก Barcode"""
        
        # Parse GS1 Barcode
        parsed = self.parse_gs1_128(barcode)
        batch_no = parsed.get('AI_10', {}).get('value')
        
        if not batch_no:
            return {'error': 'ไม่พบ Batch Number ใน Barcode'}
        
        # ดึงข้อมูลทั้งหมดที่เกี่ยวกับ Batch นี้
        batch_data = await self.db.get_batch_full(batch_no)
        
        trace_report = {
            'batch_no': batch_no,
            'product': batch_data['product'],
            'production_date': batch_data['production_date'],
            'best_before': batch_data['best_before'],
            'line': batch_data['line'],
            
            # Raw Materials (One Step Back)
            'raw_materials': [
                {
                    'ingredient': rm['name'],
                    'supplier': rm['supplier'],
                    'supplier_lot': rm['supplier_lot'],
                    'received_date': rm['received_date'],
                    'certificate': rm['coa_no']  # Certificate of Analysis
                }
                for rm in batch_data['ingredients_used']
            ],
            
            # Process Records
            'process_records': {
                'haccp_ccps': await self.db.get_ccp_records(batch_no),
                'cip_records': await self.db.get_cip_records(batch_data['equipment_used']),
                'weight_records': await self.db.get_weight_records(batch_no),
                'bake_records': await self.db.get_bake_records(batch_no)
            },
            
            # Quality
            'quality_results': await self.db.get_quality_records(batch_no),
            
            # Distribution (One Step Forward)
            'delivered_to': await self.db.get_delivery_records(batch_no)
        }
        
        return trace_report
```

---

## Step 1828: Validation Documentation

### Process Validation สำหรับ FDA/GMP

```
VALIDATION PROTOCOL: VP-2024-001
Equipment: Tunnel Oven LINE-01
Purpose: Heat Treatment Validation (ฆ่าเชื้อ Salmonella)

Acceptance Criteria:
- 5-Log Reduction of Salmonella typhimurium
- Minimum Internal Temperature: 95°C ที่ Cold Spot
- CCP Temperature: ≥ 210°C ที่ทุก Zone ตลอด Baking

IQ (Installation Qualification):
✓ Equipment ตรงตาม Specification
✓ Calibration Certificate ครบ
✓ Utility ถูกต้อง
✓ Safety Device ทำงาน

OQ (Operational Qualification):
✓ Temperature Distribution Study (ทุก Zone, Empty Oven)
✓ Belt Speed Accuracy ±1%
✓ Steam System Performance
✓ Control System Accuracy ±2°C

PQ (Performance Qualification):
✓ 3 Runs ต่อ Product Type
✓ Cold Spot Identification
✓ Lethality Calculation (F-value)
✓ ผลทดสอบ Microbiology

RESULTS SUMMARY:
Run 1: Min Core Temp = 97.3°C ✓
Run 2: Min Core Temp = 96.8°C ✓  
Run 3: Min Core Temp = 98.1°C ✓
Average F-value (70°C): > 100 minutes ✓
Salmonella Log Reduction: > 8 ✓

CONCLUSION: VALIDATED ✓
Next Revalidation: 2025-12-31
```

---

## Lab Project: Mini Bakery Control System

```
Lab Objectives:
1. ออกแบบ Recipe Management System สำหรับ 3 ผลิตภัณฑ์
2. Implement Batch Control (ISA-88)
3. สร้าง CCP Monitoring สำหรับ HACCP
4. ทดสอบ Traceability จาก Raw Material ถึง Finished Product

Equipment:
- Allen-Bradley Micro850 PLC (หรือ Simulation)
- HMI: PanelView 800 7"
- Scale Module (สำหรับ Weighing Simulation)
- Temperature Controller
- Barcode Scanner
- RFID Reader (Raw Material Tracking)

Success Criteria:
- Recipe Download ทำงานใน < 5 วินาที
- Weight Accuracy ±0.5%
- Traceability Query Response < 3 วินาที
- CCP Violation Alert ใน < 10 วินาที
```

---

## สรุป Part 92

โปรเจกต์นี้ครอบคลุม:
- ✅ Recipe Management System (GMP Compliant)
- ✅ ISA-88 Batch Control
- ✅ CIP System (Caustic + Acid)
- ✅ HACCP CCP Monitoring
- ✅ Automatic Weighing & Dosing
- ✅ GS1 Track & Trace
- ✅ Process Validation (IQ/OQ/PQ)

---

## การนำทาง
- [← Part 91: Capstone - Automotive Assembly Line](part91_capstone_automotive.md)
- [→ Part 93: Capstone - Water Treatment Plant](part93_capstone_waterworks.md)
- [📚 สารบัญหลัก](../README.md)
