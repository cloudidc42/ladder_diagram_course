# Part 59: Version Control for PLC Programs (Steps 1161-1180)

## บทนำ: ทำไม PLC ต้องการ Version Control

ปัญหาที่พบบ่อยในโรงงาน:
- "ใครแก้ไข program เมื่อวาน?"
- "เวอร์ชันที่ production ใช้อยู่คืออะไร?"
- "โรงงาน A กับ B ใช้ program เดียวกันไหม?"
- "จะ rollback ยังไงถ้าการแก้ไขทำให้เครื่องพัง?"

Version Control แก้ทุกปัญหาเหล่านี้

---

## Step 1161: ทำไมต้องใช้ Version Control

### สถิติปัญหา PLC โดยไม่มี Version Control

```
ปัญหาจริงในอุตสาหกรรม:
──────────────────────────────────────────────────────────────

1. "Works on my machine" syndrome
   Engineer แก้ program บนโน๊ตบุ๊คของตัวเอง
   Upload ขึ้น PLC โดยไม่ backup เวอร์ชันเก่า
   ถ้า program ใหม่มีปัญหา → ไม่สามารถย้อนกลับได้
   Production หยุดชะงัก

2. Multiple versions nightmare
   - site_A_program_v1.zip
   - site_A_program_v2_FINAL.zip
   - site_A_program_v2_FINAL_REAL.zip
   - site_A_program_v2_FINAL_REAL_2024.zip
   - site_A_program_JOHN_EDIT.zip
   ← ไม่รู้ว่าอันไหนถูก!

3. Concurrent editing conflict
   Engineer A แก้ Network 1
   Engineer B แก้ Network 5 (เวลาเดียวกัน)
   ทั้งคู่ upload ขึ้น PLC → งานของใครคนหนึ่งหาย!

4. No change history
   เครื่องเกิดปัญหา เมื่อ 3 เดือนก่อน
   ไม่รู้ว่ามีการแก้ program เมื่อไหร่บ้าง
   ไม่รู้ว่า program เวอร์ชันไหนใช้งานอยู่

VERSION CONTROL แก้ทั้งหมดนี้:
✓ ทุก change มี timestamp + author + reason
✓ สามารถ rollback ได้ทุกเวลา
✓ Merge conflicts ตรวจพบและแก้ได้
✓ รู้ว่า production ใช้เวอร์ชันอะไร
```

---

## Step 1162: Git Fundamentals สำหรับ PLC Engineers

### Git Concepts ที่จำเป็น

```
GIT KEY CONCEPTS:
──────────────────────────────────────────────────────────────

Repository (Repo): โฟลเดอร์ที่ track ด้วย Git
  ├── Working Directory (ไฟล์ที่คุณทำงานอยู่)
  ├── Staging Area (ไฟล์ที่เตรียม commit)
  └── Repository (.git folder - ประวัติทั้งหมด)

Commit: ภาพถ่าย ณ เวลาหนึ่ง (Snapshot)
  ├── Hash: abc1234 (unique identifier)
  ├── Author: somchai@company.com
  ├── Date: 2024-03-15 14:30
  ├── Message: "Add conveyor timeout safety - Fix #247"
  └── Changes: list of file changes

Branch: สายพัฒนาที่แยกออก
  ├── main: production-ready code
  ├── develop: integration branch
  ├── feature/xxx: new features
  └── hotfix/xxx: emergency fixes

Remote: Git server (GitLab, GitHub, Bitbucket, Gitea)

BASIC GIT WORKFLOW:
──────────────────────────────────────────────────────────────

# 1. Clone repository (ครั้งแรก)
git clone https://gitlab.company.com/plc/plant-a-filling.git
cd plant-a-filling

# 2. ดู status ปัจจุบัน
git status
git log --oneline -10

# 3. สร้าง branch ใหม่ก่อนแก้ไข
git checkout -b feature/add-jam-detection

# 4. แก้ไข PLC files (export จาก TIA Portal)

# 5. Stage changes
git add src/FillStation.xml
git add docs/change_log.md

# 6. Commit พร้อม descriptive message
git commit -m "Add jam detection for conveyor zone 3

- Added sensor timeout check 5 seconds
- Jam alarm triggers conveyor stop and HMI message
- Tested in Factory I/O simulation

Resolves: MAINT-247
"

# 7. Push ไป remote
git push origin feature/add-jam-detection

# 8. สร้าง Merge Request (GitLab) หรือ Pull Request (GitHub)
# → ให้คนอื่น review ก่อน merge ไป main
```

---

## Step 1163: การจัดการ PLC Files ใน Git

### File Structure และ .gitignore

```
PLC PROJECT STRUCTURE ใน Git:
──────────────────────────────────────────────────────────────

plant-a-filling/
├── .gitignore                  ← ไฟล์ที่ไม่ต้อง track
├── README.md                   ← Project overview
├── CHANGELOG.md                ← Version history
├── docs/
│   ├── IO_list.xlsx            ← I/O list
│   ├── network_diagram.pdf     ← Network layout
│   ├── functional_spec.pdf     ← Functional specification
│   └── FAT_report.pdf          ← Factory Acceptance Test
├── src/
│   ├── TIA_Portal/
│   │   ├── FillingMachine.ap17 ← TIA Portal project (binary)
│   │   └── exports/            ← XML/SCL exports (text-friendly)
│   │       ├── FB_FillControl.xml
│   │       ├── FB_ConveyorCtrl.xml
│   │       └── DB_Recipes.xml
│   ├── Allen_Bradley/
│   │   └── ACD/FillingLine.ACD ← Studio 5000 project
│   └── HMI/
│       └── WinCC_FillingMachine.zip
├── test/
│   ├── FAT_test_cases.xlsx     ← Test cases
│   └── test_results_2024-03.pdf
└── deployment/
    ├── production_version.txt  ← "v2.3.1 deployed 2024-03-15"
    └── plc_addresses.txt       ← PLC IP addresses

// .gitignore สำหรับ TIA Portal:
*.bak           ← Backup files
*.tmp           ← Temp files
~$*.*           ← Office temp files
Compiled/       ← Compiled code (regenerate จาก source)
*.suo           ← VS solution options
```

### XML Export Workflow

```
TIA PORTAL → GIT WORKFLOW:
──────────────────────────────────────────────────────────────

เนื่องจาก TIA Portal .ap17 เป็น binary file → Git diff ใช้ไม่ได้

วิธีที่ดีกว่า: Export เป็น XML text

TIA Portal Export Steps:
1. Right-click → Export → XML
2. เลือก: Program blocks, Data blocks, HMI
3. บันทึกใน src/TIA_Portal/exports/
4. Git track XML files (text-based → diff ได้)

Siemens TIA Portal XML Export Script (PowerShell):
```

```powershell
# Export-TIAProject.ps1
# Auto-export TIA Portal blocks to XML for Git tracking

$TIA_PATH = "C:\Program Files\Siemens\Automation\Portal V17"
$PROJECT   = "C:\Projects\FillingMachine\FillingMachine.ap17"
$EXPORT_DIR= ".\src\TIA_Portal\exports"

Add-Type -Path "$TIA_PATH\PublicAPI\V17\Siemens.Engineering.dll"

$TiaPortal = [Siemens.Engineering.TiaPortal]::new(
    [Siemens.Engineering.TiaPortalMode]::WithoutUserInterface
)

$Project = $TiaPortal.Projects.Open($PROJECT)

# Export all PLC blocks
ForEach ($Device in $Project.Devices) {
    ForEach ($Group in $Device.DeviceItems) {
        $Blocks = $Group.GetService[
            Siemens.Engineering.SW.OA.BlockGroup]()
        ForEach ($Block in $Blocks.Blocks) {
            $FileName = "$EXPORT_DIR\$($Block.Name).xml"
            $Block.Export($FileName, 
                [Siemens.Engineering.ExportOptions]::None)
            Write-Host "Exported: $($Block.Name)"
        }
    }
}

$Project.Close()
$TiaPortal.Dispose()
Write-Host "Export complete: $EXPORT_DIR"
```

---

## Step 1164: Siemens TIA Portal กับ Git Integration

### TIA Portal Version Control ทางการ

```
SIEMENS TIA PORTAL VERSION CONTROL OPTIONS:
──────────────────────────────────────────────────────────────

Option 1: TIA Portal Openness + Git (วิธีที่ดีสุด)
├── TIA Openness API: script export/import
├── Export → XML → Git → Import
├── CI/CD pipeline สำหรับ auto-compile และ test
└── ต้องการ TIA Portal Openness license

Option 2: TeamCenter Integration (Enterprise)
├── Siemens Teamcenter PLM integration
├── Version control ใน TeamCenter
├── Expensive, แต่ enterprise-grade
└── ใช้กับ large automotive/aero manufacturers

Option 3: Manual Git + zip files (ทำได้ทันที)
├── Save project → zip → Git commit
├── Binary (ไม่ได้ diff ได้, แต่ยังดีกว่าไม่มี)
└── ง่ายที่สุด

RECOMMENDED WORKFLOW (Option 1):
──────────────────────────────────────────────────────────────

Pre-commit hook (ทำงานอัตโนมัติก่อน commit):
1. TIA Openness export all blocks to XML
2. Validate XML syntax
3. Compile check (optional)
4. Stage exported files
5. Proceed with commit

.git/hooks/pre-commit:
```

```bash
#!/bin/bash
# pre-commit hook: export TIA Portal to XML before commit

echo "Running TIA Portal export..."
powershell -File scripts/Export-TIAProject.ps1

if [ $? -ne 0 ]; then
    echo "ERROR: TIA Portal export failed"
    exit 1
fi

# Stage exported XML files
git add src/TIA_Portal/exports/*.xml

echo "TIA Portal export complete"
exit 0
```

---

## Step 1165: Allen-Bradley / Rockwell กับ Version Control

### Studio 5000 กับ Git

```
ALLEN-BRADLEY / ROCKWELL AUTOMATION VERSION CONTROL:
──────────────────────────────────────────────────────────────

Studio 5000 (.ACD file): Binary format
แนวทาง: Export as L5X (XML format)

L5X Export from Studio 5000:
File → Save As → RSLogix 5000 XML (.L5X)
→ L5X เป็น text XML → Git diff ได้

L5X Structure:
<RSLogix5000Content>
  <Controller Name="FillingLine_L85E" ...>
    <Tags>
      <Tag Name="iConveyorSpeed" TagType="Base" DataType="INT">
      ...
    </Tags>
    <Programs>
      <Program Name="MainProgram">
        <Routines>
          <Routine Name="MainRoutine">
            <RLLContent>
              <!-- Ladder Diagram rungs in XML -->
```

```xml
<!-- L5X Git-friendly example -->
<!-- ไฟล์นี้ diff ได้ใน Git -->
<Rung Number="0" Type="N">
  <Comment>
    <![CDATA[Conveyor Start Logic - Added jam timeout 2024-03-15]]>
  </Comment>
  <Text>
    <![CDATA[XIC(ConveyorStart)XIC(SafetyOK)OTE(ConveyorRun);]]>
  </Text>
</Rung>
```

```
Allen-Bradley Version Control Workflow:
1. Studio 5000 → Save As L5X
2. git add FillingLine.L5X
3. git commit -m "Add jam detection timeout"
4. git diff HEAD~1 HEAD -- FillingLine.L5X
   → แสดง exactly what changed in ladders

FactoryTalk AssetCentre (Enterprise option):
├── Rockwell's own PLM/version control tool
├── Track all PLC, HMI, drive firmware changes
├── Audit trail สำหรับ GxP compliance (pharma)
└── Integration กับ Studio 5000
```

---

## Step 1166: Mitsubishi GX Works กับ Version Control

```
MITSUBISHI GX WORKS VERSION CONTROL:
──────────────────────────────────────────────────────────────

GX Works 3 (.gx3 file): Binary
แนวทาง: Export as CSV or structured text

GX Works 3 Export Options:
├── File → Export → CSV (ทุก blocks)
├── Project → Save as Text (structured text .st)
└── Print → Print to File (สำหรับ documentation)

Mitsubishi Structured Text Export:
Function block export เป็น .st ไฟล์ → text → Git-friendly

// FB_ConveyorControl.st (exported from GX Works 3)
(*
 * FB_ConveyorControl
 * Author: Tanaka
 * Date: 2024-03-15
 * Version: 1.2
 *)
FUNCTION_BLOCK FB_ConveyorControl
VAR_INPUT
    xStart: BOOL;
    xStop:  BOOL;
    xFault: BOOL;
END_VAR
VAR_OUTPUT
    xMotorRun: BOOL;
END_VAR
// [D100] xStart  – Mitsubishi address
// [D101] xStop
// [M10]  xMotorRun – Output coil
...
END_FUNCTION_BLOCK

GIT WORKFLOW MITSUBISHI:
# Export GX Works 3 project → structured text
# git add src/Mitsubishi/exports/*.st
# git commit -m "Update conveyor speed ramp"
```

---

## Step 1167: Branching Strategy

### Git Flow สำหรับ PLC Projects

```
PLC GIT BRANCHING STRATEGY:
──────────────────────────────────────────────────────────────

                    ┌─── hotfix/conveyor-estop ───┐
                    │    (Emergency fix, fast)     │
                    ↓                              ↓
main ─────────────●────────────────────────────────●──→
   (Production)   │v2.0                             │v2.1
                  │                                 │
develop ────●─────┼─────────●──────────●───────────●──→
            │     │         │ integration│           │
            │     │         │            │           │
feature/    │     │   feature/jam-detect feature/    │
recipe-v2   ↗     │         ↗            recipe-v3   ↗
            │             │
            └─────────────┘
         (feature branches)

BRANCHES EXPLAINED:
├── main: Production-ready เท่านั้น
│   - ทุก commit คือ tagged version
│   - ต้องผ่าน review และ testing ก่อน merge
│   - CI/CD ทำ auto-deploy หลัง merge
│
├── develop: Integration branch
│   - Features merge เข้าที่นี่ก่อน
│   - Run automated tests
│   - Ready for next release
│
├── feature/xxx: Feature development
│   - สร้างจาก develop
│   - 1 feature = 1 branch
│   - Merge request + review ก่อน merge ไป develop
│
└── hotfix/xxx: Emergency production fix
    - สร้างจาก main
    - Fix แล้ว merge ทั้ง main และ develop
    - Deploy ทันที (ผ่าน expedited review)

NAMING CONVENTIONS:
feature/add-jam-detection
feature/recipe-management-v3
bugfix/fix-conveyor-timeout
hotfix/emergency-estop-relay
release/v2.3.0
```

---

## Step 1168: Commit Message Standards

### Conventional Commits สำหรับ PLC

```
CONVENTIONAL COMMIT FORMAT สำหรับ PLC:
──────────────────────────────────────────────────────────────

Format: <type>(<scope>): <short description>

<longer description if needed>

<footer: issue references, breaking changes>

TYPES:
├── feat:     New feature (new FB, new recipe, new screen)
├── fix:      Bug fix (runtime error, wrong logic)
├── safety:   Safety-related change (ALWAYS prominent)
├── perf:     Performance improvement (scan time)
├── refactor: Code restructure (same behavior, better code)
├── docs:     Documentation only
├── test:     Test code only
└── chore:    Build, tooling (not PLC code)

SCOPES:
├── filling:  Filling station
├── conveyor: Conveyor system
├── safety:   Safety PLC
├── hmi:      HMI screens
└── comm:     Communication (OPC UA, Modbus)

EXAMPLES:
──────────────────────────────────────────────────────────────

feat(filling): add product changeover recipe management

Added ability to switch recipes on-the-fly without stopping
production. Recipe selection via HMI dropdown.
Recipes stored in DB_Recipes data block (50 recipes max).

Tested: 10 changeovers, all products correct
Reviewed-by: khun-wichai@company.com
Resolves: #PR-123

──────────────────────────────────────────────────────────────

safety(estop): increase E-stop response time check to 100ms

SAFETY CRITICAL: Previous 500ms check was too slow.
New check: PLC must confirm E-stop chain intact within 100ms.
Validated: SIL 1 analysis updated, ref SIL-DOC-2024-05.

Breaking change: Safety PLC firmware v3.2+ required.
Reviewed-by: safety-engineer@company.com
Resolves: SAFETY-247

──────────────────────────────────────────────────────────────

fix(conveyor): correct jam detection sensor debounce

Bug: False jam alarms due to vibration
Fix: Increased debounce filter from 50ms to 200ms
Testing: 8-hour run, 0 false alarms

Fixes: #BUG-891
```

---

## Step 1169: Code Review Process

### Merge Request / Pull Request สำหรับ PLC

```
PLC CODE REVIEW CHECKLIST (Reviewer):
──────────────────────────────────────────────────────────────

SAFETY & RISK:
☐ มี safety-critical changes ไหม? ต้องการ safety review เพิ่มเติม?
☐ E-stop และ safety functions ยังทำงานถูกต้อง?
☐ ไม่มีการ bypass safety interlocks?
☐ Hardware outputs ที่ force ได้ถูกจำกัดเหมาะสม?

CORRECTNESS:
☐ Logic ตรงกับ functional specification?
☐ State machine transitions ถูกต้อง?
☐ Timer values เหมาะสม (ไม่ช้าเกินไป, ไม่เร็วเกินไป)?
☐ ทุก fault condition มี handling?

CODE QUALITY:
☐ Variable names สื่อความหมาย?
☐ Magic numbers กำหนดเป็น constants?
☐ Comments อธิบาย WHY ไม่ใช่แค่ WHAT?
☐ ไม่มี dead code?

TESTING:
☐ มี test evidence (simulation, offline test)?
☐ ทุก new feature มี test case?
☐ Regression test ผ่าน?

DOCUMENTATION:
☐ CHANGELOG.md อัพเดท?
☐ IO list ยังถูกต้อง?
☐ Version number เพิ่มขึ้น?

GitLab Merge Request Template (MERGE_REQUEST_TEMPLATE.md):
## Description
What does this change do?

## Type of Change
- [ ] Safety-critical change
- [ ] New feature
- [ ] Bug fix
- [ ] Refactoring

## Testing Done
Describe testing performed:

## Safety Review Required?
- [ ] Yes - Tag @safety-engineer
- [ ] No - Explain why not needed:

## Checklist
- [ ] Code follows naming conventions
- [ ] Self-reviewed
- [ ] No debug/test code left
- [ ] CHANGELOG updated
```

---

## Step 1170: Tagging และ Release Management

### Version Tagging

```
VERSION TAGGING STRATEGY:
──────────────────────────────────────────────────────────────

Semantic Versioning: MAJOR.MINOR.PATCH
├── MAJOR: Breaking change (I/O pin change, CPU replace)
├── MINOR: New feature (new recipe, new alarm)
└── PATCH: Bug fix (timing tweak, typo)

ตัวอย่าง:
v1.0.0 - Initial production release
v1.1.0 - Added recipe management
v1.1.1 - Fixed conveyor timeout bug
v2.0.0 - New filling head hardware (MAJOR: I/O changed)

# สร้าง Tag:
git tag -a v2.1.0 -m "Release v2.1.0

New features:
- Recipe management (50 recipes)
- Jam detection with auto-retry

Bug fixes:
- Conveyor Z3 timeout false alarm
- Fill volume calculation rounding

Deployed: 2024-03-15 (Plant A, Line 1)
PLC: S7-1515-2 PN, FW: V2.9
"

git push origin v2.1.0

# ดู tags ทั้งหมด:
git tag -l
git log --tags --simplify-by-decoration --pretty="format:%d %s"

# สร้าง Release ใน GitLab/GitHub:
# GitLab: Repository → Tags → Create Release
# ← Attach release notes, compiled backup
```

---

## Step 1171: Deployment Tracking

### ติดตามว่า PLC ใช้เวอร์ชันอะไร

```
DEPLOYMENT TRACKING:
──────────────────────────────────────────────────────────────

ปัญหา: PLC อาจมีหลายตัว, แต่ละตัวอาจใช้ version ต่างกัน

วิธีที่ 1: Version String ใน PLC
// ใน PLC program:
DATA_BLOCK DB_Version
    sSWVersion:     STRING := 'v2.1.0';
    sSWDate:        STRING := '2024-03-15';
    sGitHash:       STRING := 'abc1234';     // Short commit hash
    sEngineer:      STRING := 'Somchai';     // Deployed by
    dtDeployDate:   DATE_AND_TIME;           // When deployed
END_DATA_BLOCK

// HMI Display: ดูเวอร์ชันได้ง่าย
// [About] Page → Software Version: v2.1.0 (abc1234)

วิธีที่ 2: Deployment Log File
# deployment/DEPLOYMENT_LOG.txt
2024-03-15 14:30  v2.1.0  Plant-A Line-1  Deployed by: Somchai  Tag: v2.1.0
2024-02-01 09:00  v2.0.1  Plant-A Line-1  Deployed by: Wichai   Tag: v2.0.1
2023-11-20 16:45  v2.0.0  Plant-A Line-1  Deployed by: Somchai  Tag: v2.0.0
2024-03-15 15:00  v2.1.0  Plant-B Line-2  Deployed by: Tanaka   Tag: v2.1.0

วิธีที่ 3: CI/CD Pipeline บันทึกอัตโนมัติ
# เมื่อ pipeline deploy ไป PLC:
# - เขียน version ลง DB_Version ใน PLC
# - บันทึกลง CMDB (Configuration Management Database)
# - ส่ง notification ไป maintenance team

CMDB (Configuration Database) สำหรับ PLC:
ASSET_ID | IP_ADDRESS  | HOSTNAME    | SW_VERSION | GIT_TAG | DEPLOY_DATE
---------|-------------|-------------|------------|---------|------------
PLC-A1   | 192.168.1.10| FillingA_L1 | v2.1.0     | v2.1.0  | 2024-03-15
PLC-A2   | 192.168.1.11| FillingA_L2 | v2.0.1     | v2.0.1  | 2024-02-01
PLC-B1   | 192.168.2.10| FillingB_L1 | v2.1.0     | v2.1.0  | 2024-03-15
```

---

## Step 1172: CI/CD Pipeline สำหรับ PLC

### Continuous Integration / Deployment

```
CI/CD PIPELINE สำหรับ PLC:
──────────────────────────────────────────────────────────────

Trigger: git push ไปยัง develop หรือ main branch

PIPELINE STAGES:
┌─────────────────────────────────────────────────────┐
│ Stage 1: LINT                                        │
│   - ตรวจ naming convention                           │
│   - ตรวจ no magic numbers                            │
│   - ตรวจ comment headers                             │
│   Tools: custom Python script                        │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Stage 2: COMPILE                                     │
│   - Import XML → TIA Portal (headless)               │
│   - Compile project                                  │
│   - Report: compile errors, warnings                 │
│   Tools: TIA Portal Openness                         │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Stage 3: SIMULATE                                    │
│   - Load compiled code into S7-PLCSIM Advanced       │
│   - Run automated test scenarios                     │
│   - Check outputs vs expected                        │
│   Tools: PLCSIM Advanced + Python test runner        │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Stage 4: PACKAGE                                     │
│   - สร้าง release package                            │
│   - Include: compiled .mc7, XMLs, docs               │
│   - Sign package (MD5/SHA256)                        │
└──────────────────────┬──────────────────────────────┘
                       │ (manual gate - approval needed)
┌──────────────────────▼──────────────────────────────┐
│ Stage 5: DEPLOY (เฉพาะ main branch + approval)       │
│   - Backup current PLC program                       │
│   - Upload new program                               │
│   - Verify MD5 checksum                              │
│   - Update deployment log                            │
│   Tools: TIA Portal Openness, custom scripts         │
└─────────────────────────────────────────────────────┘

GitLab CI/CD (.gitlab-ci.yml):
```

```yaml
# .gitlab-ci.yml สำหรับ PLC Project

stages:
  - lint
  - compile
  - simulate
  - package
  - deploy

# Stage 1: Lint
plc-lint:
  stage: lint
  script:
    - python3 scripts/plc_lint.py src/TIA_Portal/exports/
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# Stage 2: Compile
plc-compile:
  stage: compile
  script:
    - powershell scripts/Compile-TIAProject.ps1
  artifacts:
    paths:
      - build/compiled/
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop" || $CI_COMMIT_BRANCH == "main"'

# Stage 3: Simulate & Test
plc-simulate-test:
  stage: simulate
  script:
    - python3 scripts/run_plcsim_tests.py
  artifacts:
    reports:
      junit: test_results/junit.xml
  allow_failure: false

# Stage 4: Package Release
package:
  stage: package
  script:
    - scripts/create_release_package.sh
    - md5sum build/release/*.zip > build/release/checksums.md5
  artifacts:
    paths:
      - build/release/
  rules:
    - if: '$CI_COMMIT_TAG'  # Only on tags

# Stage 5: Deploy (manual, production only)
deploy-production:
  stage: deploy
  script:
    - python3 scripts/deploy_to_plc.py $PLC_IP_ADDRESS
  environment:
    name: production
  when: manual  # Requires human approval
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## Step 1173: Rollback Procedures

### วิธี Rollback เมื่อ Program มีปัญหา

```
ROLLBACK PROCEDURES:
──────────────────────────────────────────────────────────────

Emergency Rollback (production down):
1. ระบุ version ก่อนหน้า (ที่ใช้งานดี)
   git tag -l
   git show v2.0.1  ← ดู release notes

2. Checkout version เก่า
   git checkout v2.0.1

3. Export TIA project
   powershell scripts/Export-TIAProject.ps1

4. Upload ไป PLC ทันที
   (standard TIA Portal upload procedure)

5. Verify PLC running OK
   Check: Online/Offline compare = no differences

6. Document rollback
   # deployment/DEPLOYMENT_LOG.txt
   2024-03-20 22:30  ROLLBACK v2.0.1  Plant-A Line-1  Reason: v2.1.0 fill overflow bug

7. ไม่ต้องกลับไป main branch ทันที
   - วิเคราะห์ปัญหาก่อน
   - สร้าง hotfix branch จาก v2.1.0
   - Fix แล้ว test ก่อน re-deploy

Planned Rollback (scheduled maintenance):
git checkout main
git revert v2.1.0..v2.1.1  ← Revert specific commits
git tag -a v2.1.2 -m "Revert: Remove jam detection changes"
# ← Safest: สร้าง new version ที่ revert แทน force rollback
```

---

## Step 1174: Documentation as Code

### เก็บ Documentation ใน Git พร้อม Code

```
DOCUMENTATION AS CODE:
──────────────────────────────────────────────────────────────

ทุก Document ที่เกี่ยวกับ PLC program เก็บใน same Git repo:

docs/
├── functional_specification.md    ← What the system does
├── io_list.csv                    ← Exact I/O addresses
├── alarm_list.csv                 ← All alarms and responses
├── recipe_parameters.xlsx         ← Recipe data
├── network_topology.drawio        ← Network diagram (XML)
├── hmi_layout.drawio              ← HMI mockup
├── test_plan.md                   ← FAT/SAT plan
└── maintenance_manual.md          ← Maintenance procedures

io_list.csv (text format, Git-trackable):
# Plant-A Filling Line I/O List
# Version: 2.1.0, Date: 2024-03-15
ADDRESS,    TAG_NAME,         DESCRIPTION,                TYPE,   PANEL
%I0.0,      xStartPB,        Start Pushbutton,           DI,     CP1
%I0.1,      xStopPB,         Stop Pushbutton,            DI,     CP1
%I0.2,      xEStop_1,        E-Stop Zone 1,              DI,     CP1
%Q0.0,      xConveyorRun,    Conveyor Motor Forward,     DO,     MCC1
%Q0.1,      xFillValve,      Fill Valve Open,            DO,     VLV1
%IW64,      iFillFlowRaw,    Fill Flow Meter Raw Value,  AI,     TER1
%QW80,      iVFD_SpeedCmd,   VFD Speed Command,          AO,     VFD1

alarm_list.csv:
ALARM_ID, TAG_NAME,            TYPE,    MESSAGE_THAI,              ACTION
AL001,    xJamDetected,        Warning, สายพาน: ตรวจพบการติดขัด,  หยุดสายพาน
AL002,    xFillTimeout,        Alarm,   เติมสินค้า: หมดเวลา,       หยุดเครื่อง
AL003,    xLevelLow,           Warning, ถังพัก: ระดับต่ำ,          แจ้งเตือน

ประโยชน์:
├── Document เปลี่ยนพร้อม code (same commit)
├── Review: ดูทั้ง code change และ doc change
└── Always in-sync: ไม่มีปัญหา doc เก่า vs code ใหม่
```

---

## Step 1175: Merge Conflict Resolution

### จัดการ Conflicts ใน PLC Files

```
MERGE CONFLICT RESOLUTION:
──────────────────────────────────────────────────────────────

Binary Files (.ap17, .ACD):
├── ไม่สามารถ auto-merge ได้
├── ต้อง manual merge
└── วิธี: ดูว่าใครแก้อะไร และ merge manually

ป้องกัน conflicts:
1. ใช้ XML exports (text) แทน binary
2. ตกลงกันว่าใครรับผิดชอบ FB ไหน
3. ใช้ feature flags (แก้ code แต่ disable ก่อน)

XML Conflict ตัวอย่าง:
```

```xml
<<<<<<< HEAD (Somchai's branch)
<Tag Name="tConveyorTimeout" DataType="TIME">
  <DefaultData Format="L5K">T#5s</DefaultData>
</Tag>
=======
<Tag Name="tConveyorTimeout" DataType="TIME">
  <DefaultData Format="L5K">T#10s</DefaultData>
</Tag>
>>>>>>> feature/increase-timeout (Wichai's branch)
```

```
วิธีแก้ conflict:
1. ประชุมกับทั้ง Somchai และ Wichai
2. ตัดสินใจ: 5 วินาที หรือ 10 วินาที?
3. แก้ไขไฟล์ให้เป็นค่าที่ถูกต้อง
4. git add FillingLine.L5X
5. git commit -m "Resolve conflict: conveyor timeout = 8s (compromise)"

Best Practice: ห้าม auto-resolve conflicts ใน PLC code
ต้อง human decision เสมอ (safety risk)
```

---

## Step 1176: Secrets Management

### จัดการ Passwords และ Keys ใน Git

```
SECRETS MANAGEMENT:
──────────────────────────────────────────────────────────────

อย่าเก็บใน Git:
❌ PLC IP Addresses (security risk)
❌ Passwords (HMI, OPC UA, VPN)
❌ API Keys (Cloud, MQTT)
❌ SSL Certificates (private keys)

ให้ใช้:
✓ Environment Variables: PLC_IP=$PLC_IP (กำหนดใน CI/CD settings)
✓ Secrets Manager: GitLab CI/CD Variables, HashiCorp Vault
✓ .env files: ใส่ใน .gitignore

.gitignore:
.env              ← Secrets file (local only)
*.pem             ← Certificates
*.key             ← Private keys
config/local.yml  ← Local config with passwords

config/settings.example.yml (เก็บใน Git - ไม่มี actual values):
# Copy to config/settings.yml and fill in your values
plc:
  ip_address: "192.168.xxx.xxx"    # Fill in actual IP
  port: 102
  rack: 0
  slot: 1

opcua:
  server_url: "opc.tcp://SERVER:4840"
  username: "YOUR_USERNAME"
  password: "YOUR_PASSWORD"         # Use environment variable

GitLab CI/CD Variables (Secrets):
Settings → CI/CD → Variables:
├── PLC_IP_ADDRESS = 192.168.1.10   (Protected, Masked)
├── OPC_UA_PASSWORD = secret123     (Protected, Masked)
└── MQTT_API_KEY = abcdef12345      (Protected, Masked)

# ใน script:
python3 scripts/deploy.py $PLC_IP_ADDRESS
```

---

## Step 1177: Git Workflow Best Practices

### สรุป Best Practices

```
PLC GIT WORKFLOW BEST PRACTICES:
══════════════════════════════════════════════════════════════

DO:
✓ Commit เล็กๆ บ่อยๆ (1 feature = 1-5 commits)
✓ Commit message อธิบาย WHY ไม่ใช่ WHAT
✓ ทำ code review ก่อน merge ไป main
✓ Tag ทุก release ที่ deploy ไป production
✓ Export เป็น XML/text ก่อน commit
✓ Test ก่อน commit (simulation)
✓ Update documentation พร้อม code
✓ บันทึก deployment ทุกครั้ง

DON'T:
✗ อย่า commit ตรงไป main โดยไม่ review
✗ อย่า force push ไป main/develop
✗ อย่า commit passwords หรือ IPs
✗ อย่า commit "Work in progress" ไป main
✗ อย่าใช้ generic messages เช่น "Fix bug", "Update"
✗ อย่า skip code review เพราะรีบ
✗ อย่า merge โดยไม่ test

EMERGENCY EXCEPTION:
ถ้า production down → hotfix branch
แต่ยัง:
- ต้องทำ review (แต่เร็วขึ้น, เฉพาะ critical check)
- ต้อง test (แต่ minimal set)
- ต้อง document ว่าทำ hotfix เพราะอะไร
- ต้อง follow-up: ทำ proper fix หลังจาก production stable
```

---

## Step 1178: Advanced Git Techniques

### Techniques ที่มีประโยชน์

```
ADVANCED GIT TECHNIQUES สำหรับ PLC:
──────────────────────────────────────────────────────────────

1. git bisect - หา commit ที่ทำให้เกิด bug
# อาการ: v2.1.3 ดี, v2.2.0 มีปัญหา
# หา: commit ไหนที่ทำให้เกิดปัญหา

git bisect start
git bisect bad v2.2.0       # Version นี้มีปัญหา
git bisect good v2.1.3      # Version นี้ดี
# Git จะ checkout commit กลางๆ ให้ test
# จากนั้นบอก: git bisect good หรือ git bisect bad
# ทำซ้ำจนพบ commit ที่มีปัญหา

2. git stash - เก็บงานค้างชั่วคราว
# กำลังแก้ feature แต่มี urgent hotfix
git stash save "WIP: jam detection feature"
git checkout main
git checkout -b hotfix/emergency-estop
# ... แก้ hotfix ...
git checkout feature/jam-detection
git stash pop  # คืนงานค้าง

3. git cherry-pick - เอา commit เดียวจาก branch อื่น
# Hotfix commit อยู่ใน main แต่ต้องการใน develop ด้วย
git cherry-pick abc1234  # abc1234 = hotfix commit hash

4. git log --follow - ดู history ของไฟล์ที่ rename
git log --follow --patch src/TIA_Portal/exports/FB_FillControl.xml

5. git blame - หาว่าใครเขียนบรรทัดนั้น
git blame src/TIA_Portal/exports/FB_FillControl.xml
# แสดง: commit hash, author, date สำหรับทุกบรรทัด
```

---

## Step 1179: Version Control สำหรับ SCADA/HMI

### ขยาย Version Control ไปยัง SCADA

```
VERSION CONTROL สำหรับ SCADA/HMI:
──────────────────────────────────────────────────────────────

WinCC (Siemens):
├── Export as .zip หรือ XML
├── Track ใน same Git repo กับ PLC
└── Tag release ต้องรวม both PLC + HMI versions

FactoryTalk View (Allen-Bradley):
├── Export as .apa (binary) หรือ XML
├── ใช้ Git LFS (Large File Storage) สำหรับ binary
└── Or export via FT View API ถ้ามี

Ignition (Inductiveautomation):
├── Native Git integration (v8.x)
└── Project → Version Control → Configure
    ├── ตั้ง Git remote URL
    ├── Push/Pull จาก TIA Portal
    └── Merge conflicts ทำบน Ignition UI

GIT LFS (Large File Storage) สำหรับ binary:
# .gitattributes
*.ap17 filter=lfs diff=lfs merge=lfs -text
*.ACD  filter=lfs diff=lfs merge=lfs -text
*.zip  filter=lfs diff=lfs merge=lfs -text
*.gx3  filter=lfs diff=lfs merge=lfs -text

# ต้องติดตั้ง git-lfs ก่อน:
git lfs install
git lfs track "*.ap17"
git add .gitattributes
git commit -m "Track TIA Portal files with LFS"
```

---

## Step 1180: Lab Project - Version Control Setup

### Lab: ตั้งค่า Version Control สำหรับ PLC Project

```
LAB TASKS:
══════════════════════════════════════════════════════════════

Task 1: Setup Git Repository (20%)
─────────────────────────────────────────────────────────────
1. ติดตั้ง GitLab Community Edition (หรือใช้ GitLab.com)
2. สร้าง repository "filling-machine-plc"
3. กำหนด branch protection:
   - main: require merge request, require approval
   - develop: require merge request
4. ตั้งค่า .gitignore สำหรับ TIA Portal
5. สร้าง initial commit กับ README.md

Task 2: Export และ Commit PLC Program (30%)
─────────────────────────────────────────────────────────────
1. Export TIA Portal project เป็น XML
2. กำหนด folder structure ตาม template
3. Commit ครั้งแรก:
   git add .
   git commit -m "feat: initial commit - Filling Machine v1.0.0"
   git tag -a v1.0.0 -m "Initial release"

Task 3: Branching Workflow (30%)
─────────────────────────────────────────────────────────────
1. สร้าง feature branch: feature/add-recipe-system
2. เพิ่ม recipe management FBs ใน TIA Portal
3. Export XML และ commit
4. สร้าง Merge Request ไปยัง develop
5. ทำ Code Review (swap กับเพื่อนร่วมชั้น)
6. Merge เมื่อ approved

Task 4: Simulate Bug and Rollback (20%)
─────────────────────────────────────────────────────────────
1. Commit a "bug" (เช่น เปลี่ยน timeout ให้ผิด)
2. Push ไป main
3. Discover bug (simulation test fail)
4. Git rollback:
   git revert HEAD
   หรือ
   git checkout v1.0.0 -- src/
   git commit -m "fix: revert timeout to working value"
5. Document rollback ใน DEPLOYMENT_LOG.txt

BONUS:
- ตั้งค่า GitLab CI/CD pipeline (Stage 1: lint only)
- สร้าง Merge Request template

DELIVERABLES:
├── GitLab repository URL
├── Screenshot: branch graph (git log --graph)
├── Screenshot: merge request with comments
├── DEPLOYMENT_LOG.txt แสดง rollback
└── Brief report: ปัญหาที่พบและวิธีแก้
```

---

## สรุป Part 59

| หัวข้อ | สิ่งที่ต้องจำ |
|--------|--------------|
| Git Basics | clone, branch, commit, push, merge |
| File Format | Export XML/L5X จาก TIA Portal/Studio 5000 |
| Branching | main, develop, feature, hotfix branches |
| Commit Message | type(scope): description + WHY |
| Code Review | Safety check, logic, documentation |
| Tagging | vMAJOR.MINOR.PATCH on every release |
| Rollback | git checkout tag + redeploy |
| CI/CD | Lint → Compile → Simulate → Package → Deploy |

---

## Navigation

- [← Part 58: PLC Programming Design Patterns](./part58_plc_programming_patterns.md)
- [→ Part 60: Testing and Validation](./part60_testing_validation.md)
- [↑ สารบัญหลัก](../README.md)
