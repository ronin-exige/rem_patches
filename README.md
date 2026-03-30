Use this version. It keeps the gate parameters on the simple, visible “Set parameter with standard output” path, uses task Conditions for branching, keeps only one informational $global: parameter, and leaves the final Parameters (Query) task reachable even when validation fails. Ivanti documents all three pieces: parameter capture from standard output, multiple returns with $global:param=..., and parameter-driven task conditions; the Parameters (Query) task is specifically meant to show module parameter values in runs where conditions/evaluators set them. 

I also loosened the RunCCMRepair gate so ccmrepair.exe can still run on machines where generic WMI is back but the CCM side is still damaged. For the MECM action task, I kept the stricter gate because those actions rely on SMS_Client.TriggerSchedule, which Microsoft documents on the client WMI class. 

Module name

MECM - WMI Repair SAFE (Corrected / Parameter-Driven)

General notes

Run as SYSTEM.

Keep this module SAFE:

no winmgmt /resetrepository

no deleting %SystemRoot%\System32\wbem\Repository

no blanket regsvr32 loops


For every parameter-setter PowerShell task:

enable Set parameter with standard output

make the script output only Yes or No

have it exit 0


Ivanti’s PowerShell task supports both Set parameter with standard output and script parameter substitution like $[ParamName], which are parsed when the task executes. 

Module parameters to create first

Create these module parameters:

List (Yes/No)

ForceRepair — default No  (optional manual override)

CCMInstalled — default No

WMIFix — default No

RunCCMRepair — default No

RunMECMActions — default No

RunGPUpdate — default No


Text

WinmgmtStopMethod — default NotRun


Conditions to reuse

Repair branch

Parameter WMIFix = Yes


CCM repair

Parameter RunCCMRepair = Yes


MECM actions

Parameter RunMECMActions = Yes


GP update

Parameter RunGPUpdate = Yes


Ivanti conditions can be based on Parameter values, and you can choose whether a task executes or skips when the condition is true or false. 


---

Task 1

Type: Query Service Properties
Name: Query Winmgmt
Purpose: confirm the WMI service exists

Settings

Filter by service name: Winmgmt


Failure handling

Continue on failure



---

Task 2

Type: Query Service Properties
Name: Query CcmExec
Purpose: confirm the MECM client service exists

Settings

Filter by service name: CcmExec


Failure handling

Continue on failure



---

Task 3

Type: Files
Name: Check CCM executable
Purpose: verify MECM client binaries exist

Settings

Path / file to query: %SystemRoot%\CCM\CcmExec.exe


Failure handling

Continue on failure


Ivanti’s Query Files task is for checking file information on agents; for the WMI repository folder we’ll use PowerShell instead. 


---

Task 4

Type: Windows PowerShell Script
Name: Check WMI repository folder
Purpose: verify repository folder exists

$path = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (Test-Path -Path $path -PathType Container) {
    Write-Host "WMI repository folder exists: $path"
    exit 0
}
else {
    Write-Host "WMI repository folder missing: $path"
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 5

Type: Windows PowerShell Script
Name: Set CCMInstalled parameter
Purpose: determine whether the MECM client is present

Important

Enable Set parameter with standard output

Map it to CCMInstalled

Script must output only Yes or No


$ccmExe = Join-Path $env:SystemRoot 'CCM\CcmExec.exe'
$exeExists = Test-Path -Path $ccmExe -PathType Leaf

$svcExists = $false
try {
    Get-Service -Name 'CcmExec' -ErrorAction Stop | Out-Null
    $svcExists = $true
}
catch {
}

if ($exeExists -or $svcExists) {
    Write-Output 'Yes'
}
else {
    Write-Output 'No'
}

exit 0

Ivanti settings

File extension: ps1

Set parameter with standard output: CCMInstalled


Failure handling

Continue on failure



---

Task 6

Type: Windows PowerShell Script
Name: Verbose pre-check WMI + ConfigMgr
Purpose: log what is broken before you set the gate

$failures = @()
$ccmInstalled = '$[CCMInstalled]'
$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (Test-Path -Path $repoPath -PathType Container) {
    Write-Host "PASS: WMI repository folder exists: $repoPath"
}
else {
    $failures += "WMI repository folder missing: $repoPath"
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
    Write-Host "PASS: Basic WMI query (Win32_OperatingSystem)"
}
catch {
    $failures += "Basic WMI query failed: $($_.Exception.Message)"
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    Write-Host "Repository verify result: $verify"
    if ($verify -notmatch 'consistent|is consistent') {
        $failures += "Repository verify did not report consistent"
    }
}
catch {
    $failures += "Repository verify failed: $($_.Exception.Message)"
}

if ($ccmInstalled -eq 'Yes') {
    try {
        Get-CimInstance -Namespace 'root\ccm' -ClassName 'SMS_Client' -ErrorAction Stop | Out-Null
        Write-Host "PASS: root\ccm : SMS_Client"
    }
    catch {
        $failures += "root\ccm SMS_Client failed: $($_.Exception.Message)"
    }

    try {
        Get-CimClass -Namespace 'root\ccm\ClientSDK' -ClassName 'CCM_SoftwareCatalogUtilities' -ErrorAction Stop | Out-Null
        Write-Host "PASS: root\ccm\ClientSDK : CCM_SoftwareCatalogUtilities"
    }
    catch {
        $failures += "root\ccm\ClientSDK CCM_SoftwareCatalogUtilities failed: $($_.Exception.Message)"
    }

    try {
        $svc = Get-Service -Name 'CcmExec' -ErrorAction Stop
        Write-Host "PASS: CcmExec service found. Status: $($svc.Status)"
    }
    catch {
        $failures += "CcmExec service query failed: $($_.Exception.Message)"
    }
}
else {
    Write-Host "SKIP: MECM client not installed — skipping root\ccm and ClientSDK checks"
}

if ($failures.Count -eq 0) {
    Write-Host "OVERALL: HEALTHY"
    exit 0
}
else {
    Write-Host "OVERALL: BROKEN"
    $failures | ForEach-Object { Write-Host $_ }
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure


SMS_Client is the client WMI class, and CCM_SoftwareCatalogUtilities is a client-side SDK class you can use as a more meaningful Software Center-adjacent namespace probe than a generic OS query alone. 


---

Task 7

Type: Windows PowerShell Script
Name: Set WMIFix parameter
Purpose: decide whether the repair branch runs

Important

Enable Set parameter with standard output

Map it to WMIFix

Script must output only Yes or No


if ('$[ForceRepair]' -eq 'Yes') {
    Write-Output 'Yes'
    exit 0
}

$ccmInstalled = '$[CCMInstalled]'
$needsFix = $false
$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (-not (Test-Path -Path $repoPath -PathType Container)) {
    $needsFix = $true
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $needsFix = $true
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $needsFix = $true
    }
}
catch {
    $needsFix = $true
}

if ($ccmInstalled -eq 'Yes') {
    try {
        Get-CimInstance -Namespace 'root\ccm' -ClassName 'SMS_Client' -ErrorAction Stop | Out-Null
    }
    catch {
        $needsFix = $true
    }

    try {
        Get-CimClass -Namespace 'root\ccm\ClientSDK' -ClassName 'CCM_SoftwareCatalogUtilities' -ErrorAction Stop | Out-Null
    }
    catch {
        $needsFix = $true
    }
}

if ($needsFix) {
    Write-Output 'Yes'
}
else {
    Write-Output 'No'
}

exit 0

Ivanti settings

File extension: ps1

Set parameter with standard output: WMIFix


Failure handling

Continue on failure



---

Task 8

Type: Parameters (Query)
Name: Show detection parameters
Purpose: show what the module decided before remediation

Settings

Show all parameters in query summary: enabled


Failure handling

Continue on failure


Ivanti’s Parameters (Query) task is explicitly meant for modules where parameter values are set by conditions or evaluators. 


---

Task 9

Type: Service Properties
Name: Stop CcmExec
Purpose: stop MECM client before WMI work

Settings

Service name: CcmExec

Action: Stop service


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 10

Type: Service Properties
Name: Stop IP Helper
Purpose: stop IP Helper before WMI work

Settings

Service name: iphlpsvc

Action: Stop service


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 11

Type: Windows PowerShell Script
Name: Stop Winmgmt with PID fallback
Purpose: stop WMI cleanly, then force-stop only if needed

$serviceName = 'Winmgmt'

try {
    $svc = Get-CimInstance Win32_Service -Filter "Name='$serviceName'" -ErrorAction Stop
}
catch {
    $global:WinmgmtStopMethod = 'ServiceNotFound'
    Write-Host "Service $serviceName not found: $($_.Exception.Message)"
    exit 1
}

if ($svc.State -eq 'Stopped') {
    $global:WinmgmtStopMethod = 'AlreadyStopped'
    Write-Host "$serviceName already stopped"
    exit 0
}

try {
    Stop-Service -Name $serviceName -Force -ErrorAction Stop
    Write-Host "Stop-Service issued for $serviceName"
}
catch {
    Write-Host "Stop-Service failed for $serviceName: $($_.Exception.Message)"
}

Start-Sleep -Seconds 10

$svc = Get-CimInstance Win32_Service -Filter "Name='$serviceName'"
if ($svc.State -eq 'Stopped') {
    $global:WinmgmtStopMethod = 'Graceful'
    Write-Host "$serviceName stopped cleanly"
    exit 0
}

$pid = [int]$svc.ProcessId
Write-Host "$serviceName still running. PID: $pid"

if ($pid -le 0) {
    $global:WinmgmtStopMethod = 'NoPID'
    Write-Host "No valid PID found for $serviceName"
    exit 1
}

$servicesInSamePid = Get-CimInstance Win32_Service -Filter "ProcessId=$pid"
$serviceNames = ($servicesInSamePid | Select-Object -ExpandProperty Name) -join ', '
Write-Host "Services in same PID: $serviceNames"

if (($servicesInSamePid | Measure-Object).Count -gt 1) {
    $global:WinmgmtStopMethod = 'SharedProcessRefused'
    Write-Host "Refusing to force-kill PID $pid because multiple services share it"
    exit 1
}

try {
    Stop-Process -Id $pid -Force -ErrorAction Stop
    Write-Host "Force-killed PID $pid for $serviceName"
}
catch {
    $global:WinmgmtStopMethod = 'PIDKillFailed'
    Write-Host "Force-kill failed: $($_.Exception.Message)"
    exit 1
}

Start-Sleep -Seconds 5

$svc = Get-CimInstance Win32_Service -Filter "Name='$serviceName'"
if ($svc.State -eq 'Stopped') {
    $global:WinmgmtStopMethod = 'PIDKill'
    Write-Host "$serviceName is stopped after PID kill"
    exit 0
}
else {
    $global:WinmgmtStopMethod = 'StillRunning'
    Write-Host "$serviceName still not stopped after PID kill"
    exit 1
}

Ivanti settings

File extension: ps1


Condition

WMIFix = Yes


Failure handling

Continue on failure


Ivanti supports multiple returns from a PowerShell task via $global:param = value; I’m only using that here for the informational stop-method field. 


---

Task 12

Type: Windows PowerShell Script
Name: Salvage WMI repository
Purpose: run the safe WMI salvage

$verifyBefore = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
Write-Host "Verify before salvage: $verifyBefore"

cmd /c "winmgmt /salvagerepository" | Out-Null
Start-Sleep -Seconds 10

$verifyAfter = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
Write-Host "Verify after salvage: $verifyAfter"

if ($verifyAfter -match 'consistent|is consistent') {
    exit 0
}
else {
    exit 1
}

Ivanti settings

File extension: ps1


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 13

Type: Windows PowerShell Script
Name: Recompile WMI MOFs
Purpose: rebuild MOF registrations

$mofDir = Join-Path $env:SystemRoot 'System32\wbem'
$files  = Get-ChildItem -Path $mofDir -File | Where-Object {
    $_.Extension -in '.mof', '.mfl' -and $_.Name -notmatch 'uninstall'
}

if (-not $files) {
    Write-Host "No MOF/MFL files found in $mofDir"
    exit 1
}

$failCount = 0

foreach ($file in $files) {
    Write-Host "Compiling: $($file.FullName)"
    $output = & mofcomp.exe $file.FullName 2>&1
    if ($LASTEXITCODE -ne 0) {
        Write-Host "FAILED: $($file.Name)"
        $output | ForEach-Object { Write-Host $_ }
        $failCount++
    }
    else {
        Write-Host "OK: $($file.Name)"
    }
}

Write-Host "MOF recompilation complete. Failures: $failCount"

if ($failCount -gt 0) {
    exit 1
}
else {
    exit 0
}

Ivanti settings

File extension: ps1


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 14

Type: Service Properties
Name: Start Winmgmt
Purpose: bring WMI back up

Settings

Service name: Winmgmt

Action: Start service


Condition

WMIFix = Yes


Failure handling

Stop on failure



---

Task 15

Type: Windows PowerShell Script
Name: Wait for WMI settle
Purpose: short settle period after WMI restart

Start-Sleep -Seconds 10
exit 0

Ivanti settings

File extension: ps1


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 16

Type: Service Properties
Name: Start IP Helper
Purpose: restore IP Helper

Settings

Service name: iphlpsvc

Action: Start service


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 17

Type: Service Properties
Name: Start CcmExec
Purpose: bring MECM client back up

Settings

Service name: CcmExec

Action: Start service


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 18

Type: Windows PowerShell Script
Name: Verbose post-check WMI + ConfigMgr
Purpose: log post-repair health before deciding the remaining branch

$failures = @()
$ccmInstalled = '$[CCMInstalled]'
$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (Test-Path -Path $repoPath -PathType Container) {
    Write-Host "PASS: WMI repository folder exists: $repoPath"
}
else {
    $failures += "WMI repository folder missing: $repoPath"
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
    Write-Host "PASS: Basic WMI query (Win32_OperatingSystem)"
}
catch {
    $failures += "Basic WMI query failed: $($_.Exception.Message)"
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    Write-Host "Repository verify result: $verify"
    if ($verify -notmatch 'consistent|is consistent') {
        $failures += "Repository verify did not report consistent"
    }
}
catch {
    $failures += "Repository verify failed: $($_.Exception.Message)"
}

if ($ccmInstalled -eq 'Yes') {
    try {
        Get-CimInstance -Namespace 'root\ccm' -ClassName 'SMS_Client' -ErrorAction Stop | Out-Null
        Write-Host "PASS: root\ccm : SMS_Client"
    }
    catch {
        $failures += "root\ccm SMS_Client failed: $($_.Exception.Message)"
    }

    try {
        Get-CimClass -Namespace 'root\ccm\ClientSDK' -ClassName 'CCM_SoftwareCatalogUtilities' -ErrorAction Stop | Out-Null
        Write-Host "PASS: root\ccm\ClientSDK : CCM_SoftwareCatalogUtilities"
    }
    catch {
        $failures += "root\ccm\ClientSDK CCM_SoftwareCatalogUtilities failed: $($_.Exception.Message)"
    }

    try {
        $svc = Get-Service -Name 'CcmExec' -ErrorAction Stop
        Write-Host "CcmExec status: $($svc.Status)"
        if ($svc.Status -ne 'Running') {
            $failures += "CcmExec is not running"
        }
    }
    catch {
        $failures += "CcmExec service query failed: $($_.Exception.Message)"
    }
}
else {
    Write-Host "SKIP: MECM client not installed — skipping root\ccm and ClientSDK checks"
}

if ($failures.Count -eq 0) {
    Write-Host "OVERALL: HEALTHY"
    exit 0
}
else {
    Write-Host "OVERALL: BROKEN"
    $failures | ForEach-Object { Write-Host $_ }
    exit 1
}

Ivanti settings

File extension: ps1


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 19

Type: Windows PowerShell Script
Name: Set RunCCMRepair parameter
Purpose: allow ccmrepair.exe when WMI is usable enough and the repair EXE exists

Important

Enable Set parameter with standard output

Map it to RunCCMRepair


if ('$[WMIFix]' -ne 'Yes') {
    Write-Output 'No'
    exit 0
}

if ('$[CCMInstalled]' -ne 'Yes') {
    Write-Output 'No'
    exit 0
}

$ok = $true
$ccmRepair = Join-Path $env:SystemRoot 'CCM\ccmrepair.exe'

if (-not (Test-Path -Path $ccmRepair -PathType Leaf)) {
    $ok = $false
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $ok = $false
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $ok = $false
    }
}
catch {
    $ok = $false
}

if ($ok) {
    Write-Output 'Yes'
}
else {
    Write-Output 'No'
}

exit 0

Ivanti settings

File extension: ps1

Set parameter with standard output: RunCCMRepair


Failure handling

Continue on failure


This is the main correction: RunCCMRepair no longer requires root\ccm and ClientSDK to already be healthy.


---

Task 20

Type: Windows PowerShell Script
Name: Set RunMECMActions parameter
Purpose: only allow MECM client actions when the CCM WMI side is healthy enough

Important

Enable Set parameter with standard output

Map it to RunMECMActions


if ('$[WMIFix]' -ne 'Yes') {
    Write-Output 'No'
    exit 0
}

if ('$[CCMInstalled]' -ne 'Yes') {
    Write-Output 'No'
    exit 0
}

$ok = $true

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $ok = $false
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $ok = $false
    }
}
catch {
    $ok = $false
}

try {
    Get-CimInstance -Namespace 'root\ccm' -ClassName 'SMS_Client' -ErrorAction Stop | Out-Null
}
catch {
    $ok = $false
}

try {
    $svc = Get-Service -Name 'CcmExec' -ErrorAction Stop
    if ($svc.Status -ne 'Running') {
        $ok = $false
    }
}
catch {
    $ok = $false
}

if ($ok) {
    Write-Output 'Yes'
}
else {
    Write-Output 'No'
}

exit 0

Ivanti settings

File extension: ps1

Set parameter with standard output: RunMECMActions


Failure handling

Continue on failure


Microsoft documents TriggerSchedule on SMS_Client, so this stricter gate is intentional. 


---

Task 21

Type: Windows PowerShell Script
Name: Set RunGPUpdate parameter
Purpose: only allow policy refresh when generic WMI is healthy enough

Important

Enable Set parameter with standard output

Map it to RunGPUpdate


if ('$[WMIFix]' -ne 'Yes') {
    Write-Output 'No'
    exit 0
}

$ok = $true

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $ok = $false
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $ok = $false
    }
}
catch {
    $ok = $false
}

if ($ok) {
    Write-Output 'Yes'
}
else {
    Write-Output 'No'
}

exit 0

Ivanti settings

File extension: ps1

Set parameter with standard output: RunGPUpdate


Failure handling

Continue on failure



---

Task 22

Type: Windows PowerShell Script
Name: Run ccmrepair via PowerShell
Purpose: launch ccmrepair.exe without uploading a resource

$exe = Join-Path $env:SystemRoot 'CCM\ccmrepair.exe'

if (-not (Test-Path -Path $exe -PathType Leaf)) {
    Write-Host "ccmrepair.exe not found at $exe"
    exit 1
}

try {
    $proc = Start-Process -FilePath $exe -Wait -PassThru -NoNewWindow -ErrorAction Stop
    Write-Host "ccmrepair.exe exited with code: $($proc.ExitCode)"
    exit $proc.ExitCode
}
catch {
    Write-Host "Failed to launch ccmrepair.exe: $($_.Exception.Message)"
    exit 1
}

Ivanti settings

File extension: ps1


Condition

RunCCMRepair = Yes


Failure handling

Continue on failure



---

Task 23

Type: Windows PowerShell Script
Name: Wait after ccmrepair
Purpose: short settle period after client repair

Start-Sleep -Seconds 20
exit 0

Ivanti settings

File extension: ps1


Condition

RunCCMRepair = Yes


Failure handling

Continue on failure



---

Task 24

Type: Windows PowerShell Script
Name: Run MECM client recovery actions
Purpose: trigger the common client actions

$actions = @(
    @{ Name = 'Machine Policy Assignments Request';      Id = '{00000000-0000-0000-0000-000000000021}' },
    @{ Name = 'Machine Policy Evaluation';               Id = '{00000000-0000-0000-0000-000000000022}' },
    @{ Name = 'LS Refresh Locations Task';               Id = '{00000000-0000-0000-0000-000000000024}' },
    @{ Name = 'User Policy Assignments Request';         Id = '{00000000-0000-0000-0000-000000000026}' },
    @{ Name = 'User Policy Evaluation';                  Id = '{00000000-0000-0000-0000-000000000027}' },
    @{ Name = 'Hardware Inventory Collection Cycle';     Id = '{00000000-0000-0000-0000-000000000101}' },
    @{ Name = 'Software Inventory Collection Cycle';     Id = '{00000000-0000-0000-0000-000000000102}' },
    @{ Name = 'Discovery Data Collection Cycle';         Id = '{00000000-0000-0000-0000-000000000103}' },
    @{ Name = 'Software Updates Assignments Evaluation'; Id = '{00000000-0000-0000-0000-000000000108}' },
    @{ Name = 'Scan by Update Source';                   Id = '{00000000-0000-0000-0000-000000000113}' },
    @{ Name = 'Update Store Policy';                     Id = '{00000000-0000-0000-0000-000000000114}' },
    @{ Name = 'Application Manager Policy Action';       Id = '{00000000-0000-0000-0000-000000000121}' },
    @{ Name = 'Application Manager User Policy Action';  Id = '{00000000-0000-0000-0000-000000000122}' },
    @{ Name = 'Application Manager Global Evaluation';   Id = '{00000000-0000-0000-0000-000000000123}' }
)

$failed = $false

foreach ($action in $actions) {
    try {
        $result = Invoke-CimMethod -Namespace 'root\ccm' `
                                   -ClassName 'SMS_Client' `
                                   -MethodName 'TriggerSchedule' `
                                   -Arguments @{ sScheduleID = $action.Id } `
                                   -ErrorAction Stop

        Write-Host "OK: $($action.Name) [$($action.Id)] ReturnValue=$($result.ReturnValue)"

        if ($result.ReturnValue -ne 0) {
            $failed = $true
        }
    }
    catch {
        Write-Host "FAIL: $($action.Name) [$($action.Id)] $($_.Exception.Message)"
        $failed = $true
    }

    Start-Sleep -Seconds 2
}

if ($failed) {
    exit 1
}
else {
    exit 0
}

Ivanti settings

File extension: ps1


Condition

RunMECMActions = Yes


Failure handling

Continue on failure


Microsoft documents TriggerSchedule(String sScheduleID) and the schedule GUID list used here. 


---

Task 25

Type: Windows PowerShell Script
Name: Run gpupdate /force
Purpose: refresh Group Policy after repair

$exe = Join-Path $env:SystemRoot 'System32\gpupdate.exe'

try {
    $proc = Start-Process -FilePath $exe -ArgumentList '/force' -Wait -PassThru -NoNewWindow -ErrorAction Stop
    Write-Host "gpupdate.exe exited with code: $($proc.ExitCode)"
    exit $proc.ExitCode
}
catch {
    Write-Host "Failed to run gpupdate /force: $($_.Exception.Message)"
    exit 1
}

Ivanti settings

File extension: ps1


Condition

RunGPUpdate = Yes


Failure handling

Continue on failure



---

Task 26

Type: Windows PowerShell Script
Name: Final validation summary
Purpose: final health check and readable summary

$failures = @()
$ccmInstalled = '$[CCMInstalled]'

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
    Write-Host "PASS: Basic WMI query"
}
catch {
    $failures += "Basic WMI query failed: $($_.Exception.Message)"
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    Write-Host "Repository verify result: $verify"
    if ($verify -notmatch 'consistent|is consistent') {
        $failures += "Repository verify did not report consistent"
    }
}
catch {
    $failures += "Repository verify failed: $($_.Exception.Message)"
}

if ($ccmInstalled -eq 'Yes') {
    try {
        Get-CimInstance -Namespace 'root\ccm' -ClassName 'SMS_Client' -ErrorAction Stop | Out-Null
        Write-Host "PASS: root\ccm : SMS_Client"
    }
    catch {
        $failures += "root\ccm SMS_Client failed: $($_.Exception.Message)"
    }

    try {
        Get-CimClass -Namespace 'root\ccm\ClientSDK' -ClassName 'CCM_SoftwareCatalogUtilities' -ErrorAction Stop | Out-Null
        Write-Host "PASS: root\ccm\ClientSDK : CCM_SoftwareCatalogUtilities"
    }
    catch {
        $failures += "root\ccm\ClientSDK CCM_SoftwareCatalogUtilities failed: $($_.Exception.Message)"
    }

    try {
        $svc = Get-Service -Name 'CcmExec' -ErrorAction Stop
        Write-Host "CcmExec status: $($svc.Status)"
        if ($svc.Status -ne 'Running') {
            $failures += "CcmExec is not running"
        }
    }
    catch {
        $failures += "CcmExec service query failed: $($_.Exception.Message)"
    }
}
else {
    Write-Host "SKIP: MECM client not installed — CCM health checks skipped"
}

Write-Host "--- Parameter Summary ---"
Write-Host "CCMInstalled=$[CCMInstalled]"
Write-Host "WMIFix=$[WMIFix]"
Write-Host "WinmgmtStopMethod=$[WinmgmtStopMethod]"
Write-Host "RunCCMRepair=$[RunCCMRepair]"
Write-Host "RunMECMActions=$[RunMECMActions]"
Write-Host "RunGPUpdate=$[RunGPUpdate]"

if ($failures.Count -eq 0) {
    Write-Host "FINAL RESULT: PASS"
    exit 0
}
else {
    Write-Host "FINAL RESULT: FAIL"
    $failures | ForEach-Object { Write-Host $_ }
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure


This is the other correction: leave this on Continue on failure so the final Parameters Query still runs.


---

Task 27

Type: Parameters (Query)
Name: Show final parameters
Purpose: dump final parameter values into the job log

Settings

Show all parameters in query summary: enabled


Failure handling

Continue on failure



---

Recommended timeouts

Detection tasks: 5 minutes

MOF recompilation: 15 minutes

ccmrepair: 20–30 minutes

MECM actions: 10–15 minutes

gpupdate: 10 minutes


Ivanti’s PowerShell Execute task supports configurable timeouts and optional grabbed logs. 

Grabbed logs I’d add on the key PowerShell tasks

On Tasks 18, 22, 24, and 26, add grabbed logs when useful:

C:\Windows\CCM\Logs\CCMRepair.log

C:\Windows\CCM\Logs\ClientIDManagerStartup.log

C:\Windows\CCM\Logs\LocationServices.log

C:\Windows\CCM\Logs\PolicyAgent.log

C:\Windows\CCM\Logs\AppDiscovery.log

C:\Windows\CCM\Logs\AppIntentEval.log


The Ivanti PowerShell task supports grabbing a text log file from the target machine into job results. 

What changed versus Claude’s version

RunCCMRepair is now less strict

final validation is Continue on failure

final Parameters Query will still run

gate parameters stay on the simpler stdout path

only WinmgmtStopMethod uses $global:...


If you want, next I can turn this into a super short operator checklist with just the parameter names, conditions, and task order so you can click through the Ivanti console faster.