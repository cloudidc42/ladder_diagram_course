# Part 86: Advanced Machine Vision
## Steps 1701-1720 | ระดับ Capstone

---

## การนำทาง
- [← Part 85: AGV และ AMR Integration](part85_agv_amr.md)
- [→ Part 87: SCADA System Development](part87_scada_development.md)
- [📚 สารบัญหลัก](../README.md)

---

## Step 1701: Deep Learning สำหรับ Vision

### จาก Classical CV สู่ Deep Learning

```
Classical Computer Vision:
ข้อมูล → Feature Engineering (มือ) → Classifier → ผลลัพธ์
  ข้อเสีย: ต้องออกแบบ Feature เอง, ไม่ยืดหยุ่น

Deep Learning Vision:
ข้อมูล → CNN → Feature Extraction (อัตโนมัติ) → ผลลัพธ์
  ข้อดี: เรียนรู้ Feature อัตโนมัติ, แม่นยำสูงมาก
```

### YOLO Object Detection

```python
# Python - YOLO v8 สำหรับ Defect Detection บนสายการผลิต
from ultralytics import YOLO
import cv2
import numpy as np
from datetime import datetime

class YOLODefectDetector:
    """ตรวจจับข้อบกพร่องด้วย YOLO v8"""
    
    def __init__(self, model_path: str, confidence_threshold: float = 0.7):
        self.model = YOLO(model_path)
        self.confidence = confidence_threshold
        
        # Class Names สำหรับ PCB Inspection
        self.defect_classes = {
            0: 'Solder_Bridge',      # บัดกรีเชื่อม Pad ที่ไม่ควรเชื่อม
            1: 'Missing_Component',  # ขาด Component
            2: 'Insufficient_Solder',# บัดกรีน้อยเกินไป
            3: 'Cold_Joint',         # บัดกรีเย็น
            4: 'Wrong_Component',    # ใส่ Component ผิด
            5: 'Lifted_Lead',        # ขา Component ลอย
        }
        
        # Severity ของแต่ละ Defect
        self.defect_severity = {
            'Solder_Bridge': 'CRITICAL',
            'Missing_Component': 'CRITICAL',
            'Insufficient_Solder': 'WARNING',
            'Cold_Joint': 'WARNING',
            'Wrong_Component': 'CRITICAL',
            'Lifted_Lead': 'WARNING'
        }
        
        # Statistics
        self.inspection_count = 0
        self.defect_count = 0
        self.defect_history = []
    
    def inspect(self, image: np.ndarray) -> dict:
        """ตรวจสอบ PCB"""
        self.inspection_count += 1
        
        # Run YOLO Inference
        results = self.model(image, conf=self.confidence, verbose=False)
        
        defects = []
        has_critical = False
        
        for result in results:
            for box in result.boxes:
                class_id = int(box.cls[0])
                class_name = self.defect_classes.get(class_id, 'Unknown')
                confidence = float(box.conf[0])
                x1, y1, x2, y2 = map(int, box.xyxy[0])
                
                defect_info = {
                    'class': class_name,
                    'confidence': confidence,
                    'bbox': [x1, y1, x2, y2],
                    'severity': self.defect_severity.get(class_name, 'WARNING'),
                    'location': f"x:{(x1+x2)//2}, y:{(y1+y2)//2}"
                }
                
                defects.append(defect_info)
                
                if defect_info['severity'] == 'CRITICAL':
                    has_critical = True
        
        # Build Result
        result_dict = {
            'pass_fail': 'FAIL' if defects else 'PASS',
            'critical_fail': has_critical,
            'defect_count': len(defects),
            'defects': defects,
            'timestamp': datetime.now().isoformat(),
            'inspection_id': f"INS_{self.inspection_count:08d}"
        }
        
        # Update Statistics
        if defects:
            self.defect_count += 1
            self.defect_history.append({
                'id': self.inspection_count,
                'defects': [d['class'] for d in defects]
            })
        
        return result_dict
    
    def draw_results(self, image: np.ndarray, result: dict) -> np.ndarray:
        """วาด Bounding Box และ Label บนภาพ"""
        annotated = image.copy()
        
        for defect in result['defects']:
            x1, y1, x2, y2 = defect['bbox']
            color = (0, 0, 255) if defect['severity'] == 'CRITICAL' else (0, 165, 255)
            
            cv2.rectangle(annotated, (x1, y1), (x2, y2), color, 2)
            label = f"{defect['class']} {defect['confidence']:.0%}"
            cv2.putText(annotated, label, (x1, y1-10),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.5, color, 2)
        
        # Overall Result
        status_color = (0, 255, 0) if result['pass_fail'] == 'PASS' else (0, 0, 255)
        cv2.putText(annotated, result['pass_fail'], (10, 30),
                   cv2.FONT_HERSHEY_SIMPLEX, 1.5, status_color, 3)
        
        return annotated
    
    def get_statistics(self) -> dict:
        """สถิติการตรวจสอบ"""
        if self.inspection_count == 0:
            return {'error': 'ยังไม่มีการตรวจสอบ'}
        
        defect_rate = self.defect_count / self.inspection_count * 100
        
        # Count defect types
        defect_type_count = {}
        for item in self.defect_history:
            for defect in item['defects']:
                defect_type_count[defect] = defect_type_count.get(defect, 0) + 1
        
        top_defect = max(defect_type_count.items(), key=lambda x: x[1]) if defect_type_count else None
        
        return {
            'total_inspections': self.inspection_count,
            'total_defects': self.defect_count,
            'defect_rate': f"{defect_rate:.2f}%",
            'yield_rate': f"{100 - defect_rate:.2f}%",
            'top_defect': top_defect[0] if top_defect else 'N/A',
            'defect_breakdown': defect_type_count
        }
```

---

## Step 1702: Cognex VisionPro

### VisionPro PatMax สำหรับ Pattern Matching

```
Cognex VisionPro Tools:
1. PatMax (Pattern Matching) - ค้นหา Pattern ที่แม่นยำมาก
2. PMAlign - Alignment Tool
3. Gauge - วัดขนาด
4. Blob Analysis - วิเคราะห์ Shape
5. OCV/OCR - อ่านข้อความ
6. Caliper - วัดระยะ
7. 3D Vision Tools - วัด 3 มิติ
```

```csharp
// C# - Cognex VisionPro SDK สำหรับ Automotive Inspection
using Cognex.VisionPro;
using Cognex.VisionPro.ToolBlock;
using System;
using System.Drawing;

public class CognexInspectionSystem
{
    private CogToolBlock inspectionBlock;
    private CogJobManager jobManager;
    private CogAcqFifoTool acqTool;  // Camera Acquisition
    
    public void Initialize(string configFile)
    {
        // โหลด VisionPro Job
        jobManager = new CogJobManager();
        jobManager.Load(configFile);
        
        // ดึง Tool Block
        var job = jobManager.Jobs[0];
        inspectionBlock = job.VisionTool as CogToolBlock;
        
        Console.WriteLine("Cognex VisionPro เริ่มต้นสำเร็จ");
    }
    
    public InspectionResult RunInspection()
    {
        // รับภาพจาก Camera
        CogImage8Grey image = AcquireImage();
        
        // ส่งภาพเข้า Tool Block
        inspectionBlock.Inputs["InputImage"].Value = image;
        
        // รัน Inspection
        inspectionBlock.Run();
        
        // อ่านผลลัพธ์
        var result = new InspectionResult();
        
        // ตรวจสอบ Overall Pass/Fail
        result.PassFail = (bool)inspectionBlock.Outputs["Overall_PassFail"].Value;
        
        // อ่านขนาด
        result.Length = (double)inspectionBlock.Outputs["Part_Length_mm"].Value;
        result.Width = (double)inspectionBlock.Outputs["Part_Width_mm"].Value;
        result.Diameter = (double)inspectionBlock.Outputs["Hole_Diameter_mm"].Value;
        
        // ตรวจสอบ Defects
        result.HasScratch = (bool)inspectionBlock.Outputs["Has_Scratch"].Value;
        result.HasDent = (bool)inspectionBlock.Outputs["Has_Dent"].Value;
        result.ScratchScore = (double)inspectionBlock.Outputs["Scratch_Score"].Value;
        
        // ตำแหน่งสำหรับ Robot Pick
        result.PickX = (double)inspectionBlock.Outputs["Part_Center_X"].Value;
        result.PickY = (double)inspectionBlock.Outputs["Part_Center_Y"].Value;
        result.PickAngle = (double)inspectionBlock.Outputs["Part_Angle_deg"].Value;
        
        // บันทึก Image
        if (!result.PassFail)
        {
            SaveDefectImage(image, result);
        }
        
        return result;
    }
    
    private void SaveDefectImage(CogImage8Grey image, InspectionResult result)
    {
        string filename = $"/data/defects/{DateTime.Now:yyyyMMdd_HHmmss_fff}.jpg";
        var bitmap = image.ToBitmap();
        // วาด Defect Markers
        using (var g = Graphics.FromImage(bitmap))
        {
            if (result.HasScratch)
                g.DrawRectangle(new Pen(Color.Red, 3), 
                    (int)result.PickX - 50, (int)result.PickY - 50, 100, 100);
        }
        bitmap.Save(filename, System.Drawing.Imaging.ImageFormat.Jpeg);
    }
}

public class InspectionResult
{
    public bool PassFail { get; set; }
    public double Length { get; set; }
    public double Width { get; set; }
    public double Diameter { get; set; }
    public bool HasScratch { get; set; }
    public bool HasDent { get; set; }
    public double ScratchScore { get; set; }
    public double PickX { get; set; }
    public double PickY { get; set; }
    public double PickAngle { get; set; }
}
```

---

## Step 1703: KEYENCE CV-X Series

### KEYENCE Vision System สำหรับ In-Line Inspection

```
KEYENCE CV-X Series Features:
- Self-Calibration
- 16 Million Pixel Camera
- Multi-Camera Support (up to 16)
- PLC Communication: RS-232, Ethernet, EtherNet/IP, PROFINET
- Built-in Deep Learning (IX Series)

โปรแกรมผ่าน TouchFinder Console หรือ PC Software

การตั้งค่า PLC Communication:
1. ไปที่ System → Communication Settings
2. เลือก Protocol: EtherNet/IP
3. ตั้งค่า IP Address
4. กำหนด Output Data Format
```

### Ladder Diagram สำหรับ KEYENCE CV-X

```ladder
// ============================================
// KEYENCE CV-X Vision System Integration
// PLC: Siemens S7-1500 + EtherNet/IP
// ============================================

// NETWORK 1: ส่ง Trigger ไปยัง Vision System
|----[Part_At_Camera_Position]----[NOT Vision_Busy]----[ONE_SHOT_P]---------|
|                                                    ( Trigger_Vision )------|

// NETWORK 2: ส่ง Trigger Signal ผ่าน Digital Output
|----[Trigger_Vision]----[T_Trigger: 50ms]----( DO_Camera_Trigger )---------|

// NETWORK 3: รอผลลัพธ์จาก Vision System
// Vision ส่งผ่าน EtherNet/IP หรือ Digital Output
|----[DO_Camera_Trigger]----[T_Wait_Result: 2s]------|
|   // อ่านค่าจาก EtherNet/IP Input Assembly        |
|   MOVE Vision_Input[0] → Vision_PassFail           |
|   MOVE Vision_Input[1..4] → Vision_Part_X          |
|   MOVE Vision_Input[5..8] → Vision_Part_Y          |
|   MOVE Vision_Input[9..12] → Vision_Part_Angle     |
|   MOVE Vision_Input[13] → Vision_Defect_Type       |
|---------------------------------------------------|

// NETWORK 4: ตอบสนองต่อผลลัพธ์
|----[Vision_PassFail = 0]----( Part_PASS_Signal )---------------------------|
|----[Vision_PassFail = 0]----( Conveyor_Continue )--------------------------|

|----[Vision_PassFail = 1]----( Part_FAIL_Signal )---------------------------|
|----[Vision_PassFail = 1]----( Activate_Reject_Gate )------------------------|
|----[Vision_PassFail = 1]----( ADD Defect_Counter, 1 )----------------------|

// NETWORK 5: ส่งตำแหน่งไปยัง Robot
|----[Vision_PassFail = 0]----[Robot_Ready]----|
|   // อัพเดท Robot Pick Position             |
|   DB_Robot.PickX := Vision_Part_X           |
|   DB_Robot.PickY := Vision_Part_Y           |
|   DB_Robot.PickAngle := Vision_Part_Angle   |
|   SET Robot_Position_Updated               |
|--------------------------------------------|
```

---

## Step 1704: 3D Vision Systems

### Structured Light 3D Scanning

```python
# Python - Structured Light 3D Vision (Zivid, LMI Gocator, Photoneo)
import numpy as np
import open3d as o3d
from pathlib import Path

class StructuredLight3DVision:
    """3D Vision ด้วย Structured Light"""
    
    def __init__(self, camera_model: str = "Zivid2"):
        self.camera_model = camera_model
        self.point_cloud_buffer = []
        
    def capture_point_cloud(self) -> np.ndarray:
        """ถ่าย Point Cloud จากกล้อง 3D"""
        # Zivid SDK
        if self.camera_model == "Zivid2":
            import zivid
            app = zivid.Application()
            camera = app.connect_camera()
            
            settings = zivid.Settings(
                acquisitions=[
                    zivid.Settings.Acquisition(
                        aperture=8.0,
                        exposure_time=10000,  # microseconds
                        gain=1.0,
                        brightness=1.5
                    )
                ]
            )
            
            with camera.capture(settings) as frame:
                point_cloud_data = frame.point_cloud()
                xyz = point_cloud_data.copy_data("xyz")  # (H, W, 3)
                return xyz
        
        # Gocator SDK (LMI)
        elif self.camera_model == "Gocator3210":
            # ใช้ GoSDK
            pass
    
    def extract_height_map(self, point_cloud: np.ndarray) -> np.ndarray:
        """แปลง Point Cloud เป็น Height Map"""
        # กรองจุดที่ไม่ถูกต้อง
        valid_mask = ~np.any(np.isnan(point_cloud), axis=2)
        
        if not valid_mask.any():
            raise ValueError("ไม่มีข้อมูล Point Cloud ที่ถูกต้อง")
        
        # Height Map
        height_map = point_cloud[:, :, 2]  # Z values
        height_map[~valid_mask] = np.nan
        
        return height_map
    
    def inspect_weld_bead(self, height_map: np.ndarray, 
                           expected_height: float = 3.0,
                           tolerance: float = 0.5) -> dict:
        """ตรวจสอบรอยเชื่อม (Weld Bead) ด้วยข้อมูล 3D"""
        # ลบ Background
        background_z = np.nanpercentile(height_map, 5)
        bead_height = height_map - background_z
        
        # หา Weld Bead Region (สูงกว่า threshold)
        bead_mask = bead_height > 0.5  # สูงกว่า 0.5mm
        
        if not bead_mask.any():
            return {
                'pass_fail': 'FAIL',
                'defect': 'NO_WELD_BEAD',
                'message': 'ไม่พบรอยเชื่อม'
            }
        
        # คำนวณค่าสถิติ
        bead_heights = bead_height[bead_mask]
        mean_height = np.mean(bead_heights)
        max_height = np.max(bead_heights)
        width_mm = np.sum(bead_mask[height_map.shape[0]//2]) * 0.1  # 0.1mm per pixel
        
        # ตรวจสอบความสม่ำเสมอ
        row_heights = [np.mean(bead_height[i, bead_mask[i]]) 
                      for i in range(bead_mask.shape[0]) 
                      if bead_mask[i].any()]
        
        height_uniformity = np.std(row_heights) if row_heights else 999
        
        defects = []
        
        # ตรวจสอบความสูง
        if abs(mean_height - expected_height) > tolerance:
            defects.append({
                'type': 'HEIGHT_DEVIATION',
                'measured': mean_height,
                'expected': expected_height,
                'deviation': mean_height - expected_height
            })
        
        # ตรวจสอบความสม่ำเสมอ
        if height_uniformity > 0.5:
            defects.append({
                'type': 'POOR_UNIFORMITY',
                'std': height_uniformity
            })
        
        # ตรวจสอบ Undercut (ร่องบ่า)
        region_below_surface = (bead_height < -0.3) & ~np.isnan(bead_height)
        if region_below_surface.any():
            defects.append({
                'type': 'UNDERCUT',
                'depth': abs(np.min(bead_height[region_below_surface]))
            })
        
        return {
            'pass_fail': 'FAIL' if defects else 'PASS',
            'mean_height_mm': mean_height,
            'max_height_mm': max_height,
            'width_mm': width_mm,
            'uniformity_std': height_uniformity,
            'defects': defects
        }
    
    def measure_part_dimensions(self, point_cloud: np.ndarray) -> dict:
        """วัดขนาดชิ้นงาน 3 มิติ"""
        # ใช้ Open3D สำหรับการวิเคราะห์
        valid_points = point_cloud[~np.any(np.isnan(point_cloud), axis=2)]
        
        pcd = o3d.geometry.PointCloud()
        pcd.points = o3d.utility.Vector3dVector(valid_points.reshape(-1, 3))
        
        # Bounding Box
        aabb = pcd.get_axis_aligned_bounding_box()
        dimensions = aabb.get_extent()  # [X, Y, Z] in mm
        
        # OBB (Oriented Bounding Box - สำหรับ rotated parts)
        obb = pcd.get_oriented_bounding_box()
        obb_dimensions = obb.extent
        
        return {
            'length_mm': float(dimensions[0]),
            'width_mm': float(dimensions[1]),
            'height_mm': float(dimensions[2]),
            'volume_mm3': float(np.prod(dimensions)),
            'obb_dimensions': obb_dimensions.tolist(),
            'point_count': len(valid_points.reshape(-1, 3))
        }
```

---

## Step 1705: Vision สำหรับ Bin Picking

### Random Bin Picking ด้วย 3D Vision

```python
# Python - Bin Picking Vision ด้วย 3D Camera + Deep Learning
import numpy as np
import torch
import torchvision.transforms as T
from scipy.spatial.transform import Rotation

class BinPickingVision:
    """ระบบ Vision สำหรับ Random Bin Picking"""
    
    def __init__(self, model_path: str):
        # โหลด 6DOF Pose Estimation Model
        self.model = torch.load(model_path)
        self.model.eval()
        
        # Camera-to-Robot Transform (Hand-Eye Calibration)
        # ค่านี้ได้จากการ Calibrate ล่วงหน้า
        self.T_cam_to_robot = np.array([
            [0.9998, 0.0173, 0.0087, 235.4],   # R11, R12, R13, Tx
            [-0.0174, 0.9998, 0.0050, -42.1],   # R21, R22, R23, Ty
            [-0.0086, -0.0051, 0.9999, 412.8],  # R31, R32, R33, Tz
            [0.0, 0.0, 0.0, 1.0]               # Homogeneous
        ])
        
        # ขนาด Object
        self.object_model = {
            'width': 50.0,   # mm
            'length': 80.0,  # mm
            'height': 20.0   # mm
        }
    
    def find_pick_pose(self, rgb_image: np.ndarray, 
                        depth_map: np.ndarray) -> list:
        """หาท่าทาง Grasp สำหรับ Robot"""
        # 1. Object Detection
        objects = self.detect_objects(rgb_image)
        
        if not objects:
            return []  # ไม่พบ Object
        
        # 2. สำหรับแต่ละ Object - คำนวณ 6DOF Pose
        pick_poses = []
        
        for obj in objects:
            # ดึง Depth ในบริเวณ Object
            x1, y1, x2, y2 = obj['bbox']
            roi_depth = depth_map[y1:y2, x1:x2]
            
            # กรอง Outliers
            valid_depth = roi_depth[roi_depth > 100]  # > 100mm
            if len(valid_depth) == 0:
                continue
            
            object_depth = np.median(valid_depth)
            
            # แปลง Pixel → 3D Point (Camera Frame)
            cx = (x1 + x2) / 2
            cy = (y1 + y2) / 2
            
            # Camera Intrinsics
            fx, fy = 1000.0, 1000.0  # Focal Length (pixels)
            px, py = 960.0, 540.0   # Principal Point
            
            # Back-projection
            X_cam = (cx - px) * object_depth / fx
            Y_cam = (cy - py) * object_depth / fy
            Z_cam = object_depth
            
            # 3. Pose Estimation (6DOF)
            pose_tensor = self.prepare_patch(rgb_image, obj['bbox'])
            with torch.no_grad():
                pose_output = self.model(pose_tensor)
            
            # แปลง Output → Rotation Angles
            rx, ry, rz = pose_output[0, :3].numpy() * np.pi
            rotation = Rotation.from_euler('xyz', [rx, ry, rz])
            
            # 4. กำหนด Grasp Point (ปกติ = Top Center)
            grasp_offset = np.array([0, 0, self.object_model['height'] / 2])
            
            # 5. Transform จาก Camera Frame → Robot Frame
            point_cam = np.array([X_cam, Y_cam, Z_cam, 1.0])
            point_robot = self.T_cam_to_robot @ point_cam
            
            # Rotation ใน Robot Frame
            R_cam = rotation.as_matrix()
            R_robot = self.T_cam_to_robot[:3, :3] @ R_cam
            
            # สร้าง Final Pose
            pick_pose = {
                'x': float(point_robot[0] + grasp_offset[0]),
                'y': float(point_robot[1] + grasp_offset[1]),
                'z': float(point_robot[2] + grasp_offset[2]),
                'rotation_matrix': R_robot.tolist(),
                'confidence': float(obj['confidence']),
                'collision_free': True  # จะตรวจสอบใน Motion Planner
            }
            
            pick_poses.append(pick_pose)
        
        # เรียงลำดับตาม Confidence
        pick_poses.sort(key=lambda p: p['confidence'], reverse=True)
        
        return pick_poses
    
    def prepare_patch(self, image: np.ndarray, bbox: list) -> torch.Tensor:
        """เตรียม Image Patch สำหรับ Pose Estimation"""
        x1, y1, x2, y2 = bbox
        patch = image[y1:y2, x1:x2]
        
        transform = T.Compose([
            T.ToPILImage(),
            T.Resize((128, 128)),
            T.ToTensor(),
            T.Normalize(mean=[0.485, 0.456, 0.406],
                       std=[0.229, 0.224, 0.225])
        ])
        
        return transform(patch).unsqueeze(0)
    
    def detect_objects(self, image: np.ndarray) -> list:
        """ตรวจจับ Object ใน Image"""
        # ใช้ YOLO หรือ Classical methods
        # สำหรับตัวอย่างนี้ ใช้ OpenCV
        import cv2
        
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        
        # ตัวอย่าง: ค้นหาโดยใช้ Edge Detection
        edges = cv2.Canny(gray, 50, 150)
        contours, _ = cv2.findContours(edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        
        objects = []
        for cnt in contours:
            area = cv2.contourArea(cnt)
            if 1000 < area < 100000:  # กรองขนาดที่สมเหตุสมผล
                x, y, w, h = cv2.boundingRect(cnt)
                objects.append({
                    'bbox': [x, y, x+w, y+h],
                    'confidence': 0.8,  # Classical methods ไม่มี confidence
                    'area': area
                })
        
        return objects
```

---

## Step 1706: Omron FH Series Vision System

### Omron FH Vision System Integration

```
Omron FH Series:
- FH-1000: Entry Level, 3.2MP
- FH-2000: Mid-range, 5MP, Multi-camera
- FH-3000: High-end, 12MP, GPU Accelerated
- FH-5000: Ultra High-speed, 25fps+

Communication:
- EtherNet/IP, PROFINET, EtherCAT
- Serial (RS-232C)
- Parallel I/O
- Fieldbus (CC-Link IE)
```

---

## Step 1707-1720: Complete Defect Detection System

### ระบบตรวจสอบข้อบกพร่องสมบูรณ์

```python
# Python - Complete Defect Detection Pipeline
import asyncio
import json
import logging
from queue import Queue
from threading import Thread
import cv2
import numpy as np
from datetime import datetime

logger = logging.getLogger(__name__)

class CompleteDefectDetectionSystem:
    """ระบบตรวจสอบข้อบกพร่องแบบสมบูรณ์"""
    
    def __init__(self, config: dict):
        self.config = config
        
        # Camera Setup
        self.cameras = {}
        self.init_cameras()
        
        # AI Models
        self.detectors = {}
        self.init_models()
        
        # PLC Communication (OPC UA)
        self.plc_writer = PLCFeedbackWriter(config['plc_url'])
        
        # Database
        self.db_manager = InspectionDatabase(config['db_url'])
        
        # Image Archive
        self.image_archive = ImageArchiver(config['archive_path'])
        
        # Statistics
        self.stats = {
            'total_inspected': 0,
            'total_passed': 0,
            'total_failed': 0,
            'shift_start': datetime.now()
        }
        
        # Image Queue
        self.inspection_queue = Queue(maxsize=10)
    
    def init_cameras(self):
        """เริ่มต้น Cameras"""
        for cam_config in self.config['cameras']:
            if cam_config['type'] == 'gige':
                # GigE Camera (Basler, FLIR, IDS)
                self.cameras[cam_config['id']] = GigECamera(
                    serial=cam_config['serial'],
                    exposure=cam_config['exposure'],
                    gain=cam_config['gain']
                )
            elif cam_config['type'] == 'cognex':
                self.cameras[cam_config['id']] = CognexCamera(
                    ip=cam_config['ip']
                )
    
    def init_models(self):
        """โหลด AI Models"""
        for model_config in self.config['models']:
            if model_config['type'] == 'yolo':
                self.detectors[model_config['id']] = YOLODefectDetector(
                    model_path=model_config['path'],
                    confidence_threshold=model_config['confidence']
                )
            elif model_config['type'] == 'classification':
                self.detectors[model_config['id']] = DefectClassifier(
                    model_path=model_config['path']
                )
    
    async def inspect_product(self, product_id: str, part_number: str) -> dict:
        """ตรวจสอบผลิตภัณฑ์"""
        inspection_result = {
            'product_id': product_id,
            'part_number': part_number,
            'timestamp': datetime.now().isoformat(),
            'pass_fail': 'PASS',
            'defects': [],
            'measurements': {},
            'images': []
        }
        
        # ถ่ายภาพจากทุก Camera
        for cam_id, camera in self.cameras.items():
            image = camera.capture()
            
            # รัน Inspection
            if cam_id in self.detectors:
                detector = self.detectors[cam_id]
                result = detector.inspect(image)
                
                if result['defects']:
                    inspection_result['defects'].extend(result['defects'])
                
                inspection_result['measurements'][cam_id] = result.get('measurements', {})
            
            # บันทึกภาพ
            image_path = self.image_archive.save(image, product_id, cam_id)
            inspection_result['images'].append(image_path)
        
        # ตัดสินใจ Overall Pass/Fail
        critical_defects = [d for d in inspection_result['defects'] 
                           if d.get('severity') == 'CRITICAL']
        
        if critical_defects:
            inspection_result['pass_fail'] = 'FAIL'
        
        # อัพเดทสถิติ
        self.stats['total_inspected'] += 1
        if inspection_result['pass_fail'] == 'PASS':
            self.stats['total_passed'] += 1
        else:
            self.stats['total_failed'] += 1
        
        # บันทึกฐานข้อมูล
        await self.db_manager.save_inspection(inspection_result)
        
        # ส่งผลลัพธ์ไป PLC
        await self.plc_writer.write_result(inspection_result)
        
        # Log
        logger.info(f"ตรวจสอบ {product_id}: {inspection_result['pass_fail']} "
                   f"({len(inspection_result['defects'])} defects)")
        
        return inspection_result
    
    def get_shift_stats(self) -> dict:
        """สถิติของ Shift ปัจจุบัน"""
        total = self.stats['total_inspected']
        if total == 0:
            return {'error': 'ยังไม่มีข้อมูล'}
        
        yield_rate = self.stats['total_passed'] / total * 100
        
        return {
            'shift_start': self.stats['shift_start'].isoformat(),
            'total_inspected': total,
            'passed': self.stats['total_passed'],
            'failed': self.stats['total_failed'],
            'yield_rate': f"{yield_rate:.2f}%",
            'inspection_rate': f"{total / ((datetime.now() - self.stats['shift_start']).seconds / 60):.1f} units/min"
        }
```

---

## Lab Project: Vision Inspection สำหรับ Semiconductor

**โจทย์**: ออกแบบระบบ Wafer Defect Inspection

**Requirements**:
1. 2D Surface Inspection (Scratch, Particle)
2. Pattern Inspection (Lithography Error)
3. Edge Quality
4. Throughput: 100 wafers/hour

**Technical Stack**:
- Camera: 20MP Monochrome GigE
- Lens: Telecentric
- Lighting: Coaxial LED (BrightField/DarkField)
- AI: Custom CNN trained on wafer dataset
- Integration: SECS/GEM → MES

---

## สรุป Part 86

ใน Part นี้เราได้เรียนรู้:
- ✅ YOLO v8 สำหรับ Defect Detection
- ✅ Cognex VisionPro SDK
- ✅ KEYENCE CV-X Integration กับ PLC
- ✅ 3D Structured Light Vision
- ✅ Point Cloud Processing (Open3D)
- ✅ Bin Picking ด้วย 6DOF Pose Estimation
- ✅ Complete Inspection Pipeline

---

## การนำทาง
- [← Part 85: AGV และ AMR Integration](part85_agv_amr.md)
- [→ Part 87: SCADA System Development](part87_scada_development.md)
- [📚 สารบัญหลัก](../README.md)
