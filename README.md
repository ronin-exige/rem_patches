
Yes — here’s the split I’d use.

The SAFE module stays your first-line repair path, and I’d add a new output parameter so it can explicitly tell you “core WMI still looks broken, escalate to the aggressive module.” The AGGRESSIVE module becomes the repository-reset / re-register / broad MOF recompile path, which I’d keep as a last-resort module because Microsoft documents that /salvagerepository merges readable repository content into a rebuilt repository, and separately warns not to delete the WMI repository as a first action. Ivanti also supports the exact stdout-to-parameter pattern you’re using, and it parses $[ParamName] substitutions when the task runs. 

I also trimmed the MECM action script down to the items in your screenshot. Microsoft’s published TriggerSchedule GUID list directly covers most of those rows. The Machine Policy Retrieval & Evaluation and User Policy Retrieval & Evaluation rows are single UI rows, but the published schedules are split into separate request and evaluation GUIDs, so I trigger both halves. For Software Updates Scan Cycle, I’m using GUID 113, because Microsoft’s own example uses 113 for software update scan cache deletion and scan. The one fuzzy mapping is Application Deployment Evaluation Cycle: Microsoft’s public GUID list doesn’t publish a schedule with that exact UI label, so I’m mapping it to Application manager global evaluation action 123 as the closest documented schedule. 

SAFE module changes only

Keep your current SAFE module, but make these changes.

Add one new Text parameter

Create this additional Text parameter:

NeedsAggressiveModule = No


Insert this new task after your current Task 18

This becomes the new Task 19. Everything after it shifts by +1.

New Task 19

Type: Windows PowerShell Script
Name: Set NeedsAggressiveModule parameter
Purpose: tell you when core WMI still looks broken after the SAFE branch

Module Parameters tab

Enable Set parameter with standard output

Parameter: NeedsAggressiveModule


$NeedsAggressiveModule = 'No'

if ('$[WMIFix]' -ne 'Yes') {
    Write-Output $NeedsAggressiveModule
    exit 0
}

$coreBroken = $false
$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (-not (Test-Path -Path $repoPath -PathType Container)) {
    $coreBroken = $true
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $coreBroken = $true
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $coreBroken = $true
    }
}
catch {
    $coreBroken = $true
}

if ('$[WinmgmtStopMethod]' -in @('ServiceNotFound','NoPID','SharedProcessRefused','PIDKillFailed','StillRunning')) {
    $coreBroken = $true
}

if ($coreBroken) {
    $NeedsAggressiveModule = 'Yes'
}

Write-Output $NeedsAggressiveModule
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure


Replace the SAFE module MECM actions script

Use this in the SAFE module’s MECM actions task.

Replacement SAFE MECM actions script

$actions = @(
    # Closest documented mapping for the UI action "Application Deployment Evaluation Cycle"
    @{ Name = 'Application Deployment Evaluation Cycle';               Id = '{00000000-0000-0000-0000-000000000123}' },
    @{ Name = 'Discovery Data Collection Cycle';                      Id = '{00000000-0000-0000-0000-000000000103}' },
    @{ Name = 'Hardware Inventory Cycle';                             Id = '{00000000-0000-0000-0000-000000000101}' },
    @{ Name = 'Machine Policy Retrieval';                             Id = '{00000000-0000-0000-0000-000000000021}' },
    @{ Name = 'Machine Policy Evaluation';                            Id = '{00000000-0000-0000-0000-000000000022}' },
    @{ Name = 'Software Metering Usage Report Cycle';                 Id = '{00000000-0000-0000-0000-000000000106}' },
    @{ Name = 'Software Updates Deployment Evaluation Cycle';         Id = '{00000000-0000-0000-0000-000000000108}' },
    @{ Name = 'Software Updates Scan Cycle';                          Id = '{00000000-0000-0000-0000-000000000113}' },
    @{ Name = 'User Policy Retrieval';                                Id = '{00000000-0000-0000-0000-000000000026}' },
    @{ Name = 'User Policy Evaluation';                               Id = '{00000000-0000-0000-0000-000000000027}' },
    @{ Name = 'Windows Installer Source List Update Cycle';           Id = '{00000000-0000-0000-0000-000000000107}' }
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

Replace the SAFE module final validation summary script

Use this as the SAFE module’s final validation task so the output clearly says when to move to aggressive.

Replacement SAFE final validation summary

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
Write-Host "NeedsAggressiveModule=$[NeedsAggressiveModule]"
Write-Host "WinmgmtStopMethod=$[WinmgmtStopMethod]"
Write-Host "RunCCMRepair=$[RunCCMRepair]"
Write-Host "RunMECMActions=$[RunMECMActions]"
Write-Host "RunGPUpdate=$[RunGPUpdate]"

if ('$[NeedsAggressiveModule]' -eq 'Yes') {
    Write-Host "SAFE MODULE RESULT: Core WMI still appears broken after the safe path. Escalate to the aggressive module."
}

if ($failures.Count -eq 0) {
    Write-Host "FINAL RESULT: PASS"
    exit 0
}
else {
    Write-Host "FINAL RESULT: FAIL"
    $failures | ForEach-Object { Write-Host $_ }
    exit 1
}

That’s all you need to change in the SAFE module.

AGGRESSIVE module — full build sheet

This module is intentionally the heavier rebuild path. I would only use it after the SAFE module either flags NeedsAggressiveModule=Yes or you already know the box is badly broken. Microsoft’s docs are the reason for that sequencing: /salvagerepository preserves readable content, while deleting/resetting the repository is the more destructive path and is specifically not recommended as a first action. 

Aggressive module parameters

Create these as Text parameters:

ForceRepair = No

CCMInstalled = No

AggressiveFix = No

RunCCMRepair = No

RunMECMActions = No

RunGPUpdate = No

WinmgmtStopMethod = NotRun


Conditions

Use these Parameter conditions:

AggressiveFix = Yes

RunCCMRepair = Yes

RunMECMActions = Yes

RunGPUpdate = Yes



---

Task 1

Type: Query Service Properties
Name: Query Winmgmt
Purpose: confirm WMI service exists

Settings

Filter by service name: Winmgmt


Failure handling

Continue on failure



---

Task 2

Type: Query Service Properties
Name: Query CcmExec
Purpose: confirm MECM client service exists

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
Purpose: detect whether MECM client appears installed

Module Parameters tab

Enable Set parameter with standard output

Parameter: CCMInstalled


$CCMInstalled = 'No'

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
    $CCMInstalled = 'Yes'
}

Write-Output $CCMInstalled
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 6

Type: Windows PowerShell Script
Name: Verbose pre-check WMI + ConfigMgr
Purpose: log what is broken before setting AggressiveFix

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



---

Task 7

Type: Windows PowerShell Script
Name: Set AggressiveFix parameter
Purpose: only run the aggressive branch when core WMI looks severely broken, or when forced

Module Parameters tab

Enable Set parameter with standard output

Parameter: AggressiveFix


$AggressiveFix = 'No'

if ('$[ForceRepair]' -eq 'Yes') {
    $AggressiveFix = 'Yes'
    Write-Output $AggressiveFix
    exit 0
}

$coreBroken = $false
$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (-not (Test-Path -Path $repoPath -PathType Container)) {
    $coreBroken = $true
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $coreBroken = $true
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $coreBroken = $true
    }
}
catch {
    $coreBroken = $true
}

if ($coreBroken) {
    $AggressiveFix = 'Yes'
}

Write-Output $AggressiveFix
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 8

Type: Parameters (Query)
Name: Show detection parameters
Purpose: display parameter values after detection

Settings

Show all parameters in query summary: enabled


Failure handling

Continue on failure



---

Task 9

Type: Service Properties
Name: Stop CcmExec
Purpose: stop MECM client before aggressive WMI rebuild

Settings

Service name: CcmExec

Action: Stop service


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 10

Type: Service Properties
Name: Stop IP Helper
Purpose: stop IP Helper before aggressive WMI rebuild

Settings

Service name: iphlpsvc

Action: Stop service


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 11

Type: Windows PowerShell Script
Name: Stop Winmgmt with PID fallback
Purpose: stop WMI cleanly, then force-stop if needed

$stopMethod = 'NotRun'

try {
    $svc = Get-Service -Name 'Winmgmt' -ErrorAction Stop
}
catch {
    $stopMethod = 'ServiceNotFound'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt service not found: $($_.Exception.Message)"
    exit 1
}

if ($svc.Status -eq 'Stopped') {
    $stopMethod = 'AlreadyStopped'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt already stopped"
    exit 0
}

try {
    Stop-Service -Name 'Winmgmt' -Force -ErrorAction Stop
    Write-Host "Stop-Service issued for Winmgmt"
}
catch {
    Write-Host "Stop-Service failed for Winmgmt: $($_.Exception.Message)"
}

Start-Sleep -Seconds 10
$svc.Refresh()

if ($svc.Status -eq 'Stopped') {
    $stopMethod = 'Graceful'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt stopped cleanly"
    exit 0
}

$scOutput = sc.exe queryex Winmgmt
$pid = 0

foreach ($line in $scOutput) {
    if ($line -match 'PID\s*:\s*(\d+)') {
        $pid = [int]$matches[1]
        break
    }
}

Write-Host "Winmgmt still running. PID: $pid"

if ($pid -le 0) {
    $stopMethod = 'NoPID'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "No valid PID found for Winmgmt"
    exit 1
}

$tasklistOutput = tasklist /svc /fi "PID eq $pid"
$serviceLine = $tasklistOutput | Where-Object { $_ -match "^\s*\S+\s+$pid\s+" }

if ($serviceLine) {
    Write-Host "tasklist /svc result: $serviceLine"
    if ($serviceLine -match ',') {
        $stopMethod = 'SharedProcessRefused'
        $global:WinmgmtStopMethod = $stopMethod
        Write-Host "Refusing to kill PID $pid because multiple services appear tied to it"
        exit 1
    }
}

try {
    Stop-Process -Id $pid -Force -ErrorAction Stop
    Write-Host "Force-killed PID $pid for Winmgmt"
}
catch {
    $stopMethod = 'PIDKillFailed'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Force-kill failed: $($_.Exception.Message)"
    exit 1
}

Start-Sleep -Seconds 5

try {
    $svc = Get-Service -Name 'Winmgmt' -ErrorAction Stop
    if ($svc.Status -eq 'Stopped') {
        $stopMethod = 'PIDKill'
        $global:WinmgmtStopMethod = $stopMethod
        Write-Host "Winmgmt is stopped after PID kill"
        exit 0
    }
    else {
        $stopMethod = 'StillRunning'
        $global:WinmgmtStopMethod = $stopMethod
        Write-Host "Winmgmt still not stopped after PID kill"
        exit 1
    }
}
catch {
    $stopMethod = 'PIDKill'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt no longer queryable after PID kill; treating as stopped"
    exit 0
}

Ivanti settings

File extension: ps1


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 12

Type: Windows PowerShell Script
Name: Aggressively reset WMI repository folder
Purpose: rename the repository if possible, delete it if rename fails

$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'
$stamp = Get-Date -Format 'yyyyMMdd_HHmmss'
$backupPath = "$repoPath.bad_$stamp"

if (-not (Test-Path -Path $repoPath -PathType Container)) {
    Write-Host "Repository folder already missing: $repoPath"
    exit 0
}

try {
    Move-Item -Path $repoPath -Destination $backupPath -Force -ErrorAction Stop
    Write-Host "Repository folder renamed to: $backupPath"
    exit 0
}
catch {
    Write-Host "Rename failed, attempting delete: $($_.Exception.Message)"
}

try {
    Remove-Item -Path $repoPath -Recurse -Force -ErrorAction Stop
    Write-Host "Repository folder deleted: $repoPath"
    exit 0
}
catch {
    Write-Host "Failed to rename or delete repository folder: $($_.Exception.Message)"
    exit 1
}

Ivanti settings

File extension: ps1


Condition

AggressiveFix = Yes


Failure handling

Stop on failure



---

Task 13

Type: Windows PowerShell Script
Name: Re-register WMI DLLs and executables
Purpose: broad re-registration step from the aggressive path

$wbem = Join-Path $env:SystemRoot 'System32\wbem'
$failCount = 0

$coreDlls = @(
    (Join-Path $env:SystemRoot 'System32\scecli.dll'),
    (Join-Path $env:SystemRoot 'System32\userenv.dll')
)

foreach ($dll in $coreDlls) {
    if (Test-Path -Path $dll -PathType Leaf) {
        try {
            $p = Start-Process -FilePath 'regsvr32.exe' -ArgumentList "/s `"$dll`"" -Wait -PassThru -NoNewWindow -ErrorAction Stop
            Write-Host "regsvr32 $dll exit=$($p.ExitCode)"
            if ($p.ExitCode -ne 0) { $failCount++ }
        }
        catch {
            Write-Host "regsvr32 failed for $dll : $($_.Exception.Message)"
            $failCount++
        }
    }
}

Get-ChildItem -Path $wbem -Filter '*.dll' -File -ErrorAction SilentlyContinue | ForEach-Object {
    try {
        $p = Start-Process -FilePath 'regsvr32.exe' -ArgumentList "/s `"$($_.FullName)`"" -Wait -PassThru -NoNewWindow -ErrorAction Stop
        Write-Host "regsvr32 $($_.Name) exit=$($p.ExitCode)"
        if ($p.ExitCode -ne 0) { $failCount++ }
    }
    catch {
        Write-Host "regsvr32 failed for $($_.Name) : $($_.Exception.Message)"
        $failCount++
    }
}

$exeRegs = @('scrcons.exe','unsecapp.exe','wmiadap.exe','wmiapsrv.exe','wmiprvse.exe')

foreach ($exeName in $exeRegs) {
    $exePath = Join-Path $wbem $exeName
    if (Test-Path -Path $exePath -PathType Leaf) {
        try {
            $output = & $exePath /regserver 2>&1
            if ($output) {
                $output | ForEach-Object { Write-Host $_ }
            }
            Write-Host "$exeName /regserver completed"
        }
        catch {
            Write-Host "$exeName /regserver failed: $($_.Exception.Message)"
            $failCount++
        }
    }
}

Write-Host "Registration phase complete. Failures=$failCount"
exit 0

Ivanti settings

File extension: ps1


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 14

Type: Windows PowerShell Script
Name: Recompile WBEM MOFs and MFLs
Purpose: broad MOF/MFL recompilation in the WBEM folder

$wbem = Join-Path $env:SystemRoot 'System32\wbem'
$failCount = 0

$special = @('cimwin32.mof','rsop.mof')

foreach ($name in $special) {
    $path = Join-Path $wbem $name
    if (Test-Path -Path $path -PathType Leaf) {
        try {
            $output = & mofcomp.exe $path 2>&1
            if ($output) { $output | ForEach-Object { Write-Host $_ } }
            if ($LASTEXITCODE -ne 0) { $failCount++ }
            Write-Host "mofcomp $name exit=$LASTEXITCODE"
        }
        catch {
            Write-Host "mofcomp failed for $name : $($_.Exception.Message)"
            $failCount++
        }
    }
}

Get-ChildItem -Path $wbem -File | Where-Object {
    $_.Extension -in '.mof', '.mfl' -and $_.Name -notmatch 'uninstall' -and $_.Name -notin $special
} | ForEach-Object {
    try {
        $output = & mofcomp.exe $_.FullName 2>&1
        if ($output) { $output | ForEach-Object { Write-Host $_ } }
        if ($LASTEXITCODE -ne 0) { $failCount++ }
        Write-Host "mofcomp $($_.Name) exit=$LASTEXITCODE"
    }
    catch {
        Write-Host "mofcomp failed for $($_.Name) : $($_.Exception.Message)"
        $failCount++
    }
}

Write-Host "WBEM MOF phase complete. Failures=$failCount"
exit 0

Ivanti settings

File extension: ps1


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 15

Type: Windows PowerShell Script
Name: Recompile ExtendedStatus.mof
Purpose: recompile the Microsoft Policy Platform MOF if present

$path = 'C:\Program Files\Microsoft Policy Platform\ExtendedStatus.mof'

if (-not (Test-Path -Path $path -PathType Leaf)) {
    Write-Host "ExtendedStatus.mof not present: $path"
    exit 0
}

try {
    $output = & mofcomp.exe $path 2>&1
    if ($output) { $output | ForEach-Object { Write-Host $_ } }
    Write-Host "mofcomp ExtendedStatus.mof exit=$LASTEXITCODE"
    exit 0
}
catch {
    Write-Host "Failed to compile ExtendedStatus.mof: $($_.Exception.Message)"
    exit 0
}

Ivanti settings

File extension: ps1


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 16

Type: Service Properties
Name: Start Winmgmt
Purpose: bring WMI back up

Settings

Service name: Winmgmt

Action: Start service


Condition

AggressiveFix = Yes


Failure handling

Stop on failure



---

Task 17

Type: Windows PowerShell Script
Name: Wait for WMI settle
Purpose: short settle time after restart

Start-Sleep -Seconds 15
exit 0

Ivanti settings

File extension: ps1


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 18

Type: Service Properties
Name: Start IP Helper
Purpose: restore IP Helper

Settings

Service name: iphlpsvc

Action: Start service


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 19

Type: Service Properties
Name: Start CcmExec
Purpose: bring MECM client back up

Settings

Service name: CcmExec

Action: Start service


Condition

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 20

Type: Windows PowerShell Script
Name: Verbose post-check WMI + ConfigMgr
Purpose: log health after the aggressive WMI rebuild branch

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

AggressiveFix = Yes


Failure handling

Continue on failure



---

Task 21

Type: Windows PowerShell Script
Name: Set RunCCMRepair parameter
Purpose: allow ccmrepair.exe when generic WMI is usable and the EXE exists

Module Parameters tab

Enable Set parameter with standard output

Parameter: RunCCMRepair


$RunCCMRepair = 'No'

if ('$[AggressiveFix]' -ne 'Yes') {
    Write-Output $RunCCMRepair
    exit 0
}

if ('$[CCMInstalled]' -ne 'Yes') {
    Write-Output $RunCCMRepair
    exit 0
}

$ccmRepair = Join-Path $env:SystemRoot 'CCM\ccmrepair.exe'

if (-not (Test-Path -Path $ccmRepair -PathType Leaf)) {
    Write-Output $RunCCMRepair
    exit 0
}

$wmiOk = $true

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $wmiOk = $false
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $wmiOk = $false
    }
}
catch {
    $wmiOk = $false
}

if ($wmiOk) {
    $RunCCMRepair = 'Yes'
}

Write-Output $RunCCMRepair
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 22

Type: Windows PowerShell Script
Name: Run ccmrepair via PowerShell
Purpose: launch ccmrepair.exe

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
Purpose: settle time after client repair

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
Name: Set RunMECMActions parameter
Purpose: allow only the screenshot-mapped action subset when CCM WMI is healthy enough

Module Parameters tab

Enable Set parameter with standard output

Parameter: RunMECMActions


$RunMECMActions = 'No'

if ('$[AggressiveFix]' -ne 'Yes') {
    Write-Output $RunMECMActions
    exit 0
}

if ('$[CCMInstalled]' -ne 'Yes') {
    Write-Output $RunMECMActions
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
    $RunMECMActions = 'Yes'
}

Write-Output $RunMECMActions
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 25

Type: Windows PowerShell Script
Name: Set RunGPUpdate parameter
Purpose: allow GP refresh only when generic WMI is healthy enough

Module Parameters tab

Enable Set parameter with standard output

Parameter: RunGPUpdate


$RunGPUpdate = 'No'

if ('$[AggressiveFix]' -ne 'Yes') {
    Write-Output $RunGPUpdate
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
    $RunGPUpdate = 'Yes'
}

Write-Output $RunGPUpdate
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 26

Type: Windows PowerShell Script
Name: Run MECM client recovery actions
Purpose: run only the screenshot-mapped action subset

$actions = @(
    # Closest documented mapping for the UI action "Application Deployment Evaluation Cycle"
    @{ Name = 'Application Deployment Evaluation Cycle';               Id = '{00000000-0000-0000-0000-000000000123}' },
    @{ Name = 'Discovery Data Collection Cycle';                      Id = '{00000000-0000-0000-0000-000000000103}' },
    @{ Name = 'Hardware Inventory Cycle';                             Id = '{00000000-0000-0000-0000-000000000101}' },
    @{ Name = 'Machine Policy Retrieval';                             Id = '{00000000-0000-0000-0000-000000000021}' },
    @{ Name = 'Machine Policy Evaluation';                            Id = '{00000000-0000-0000-0000-000000000022}' },
    @{ Name = 'Software Metering Usage Report Cycle';                 Id = '{00000000-0000-0000-0000-000000000106}' },
    @{ Name = 'Software Updates Deployment Evaluation Cycle';         Id = '{00000000-0000-0000-0000-000000000108}' },
    @{ Name = 'Software Updates Scan Cycle';                          Id = '{00000000-0000-0000-0000-000000000113}' },
    @{ Name = 'User Policy Retrieval';                                Id = '{00000000-0000-0000-0000-000000000026}' },
    @{ Name = 'User Policy Evaluation';                               Id = '{00000000-0000-0000-0000-000000000027}' },
    @{ Name = 'Windows Installer Source List Update Cycle';           Id = '{00000000-0000-0000-0000-000000000107}' }
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



---

Task 27

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

Task 28

Type: Windows PowerShell Script
Name: Final validation summary
Purpose: final readable result and summary

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
Write-Host "AggressiveFix=$[AggressiveFix]"
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
    Write-Host "AGGRESSIVE MODULE RESULT: Core WMI still appears broken even after the repository rebuild path."
    $failures | ForEach-Object { Write-Host $_ }
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 29

Type: Parameters (Query)
Name: Show final parameters
Purpose: dump final parameter values into job results

Settings

Show all parameters in query summary: enabled


Failure handling

Continue on failure


Recommended timeouts for the aggressive module

Detection tasks: 5 minutes

Stop/kill Winmgmt task: 5 minutes

Repository reset task: 5 minutes

Re-register task: 15 minutes

MOF tasks: 20 minutes

ccmrepair: 20–30 minutes

MECM actions: 10–15 minutes

gpupdate: 10 minutes


Grabbed logs I’d add to both modules

On the post-check, ccmrepair, MECM actions, and final validation tasks, grab these if available:

C:\Windows\CCM\Logs\CCMRepair.log

C:\Windows\CCM\Logs\ClientIDManagerStartup.log

C:\Windows\CCM\Logs\LocationServices.log

C:\Windows\CCM\Logs\PolicyAgent.log

C:\Windows\CCM\Logs\AppDiscovery.log

C:\Windows\CCM\Logs\AppIntentEval.log


If you want, next I’ll turn both modules into a super-short operator checklist with only the task number, task type, condition, and whether the Module Parameters tab needs “Set parameter with standard output” checked.


----------------------------------
Yep — here’s the full corrected build sheet in one place, using your preferred “parameter defaults to No, flip to Yes, then Write-Output $ParameterName” style for every stdout setter task.

A couple of setup notes first:

Set these module parameters as Text parameters, since your Ivanti console only lets you use Set parameter with standard output with text parameters. Ivanti’s PowerShell task supports setting a parameter from standard output, supports parameter substitution like $[ParamName] at execution time, supports multiple-return $global:param=... if needed, and the Parameters (Query) task is specifically useful for showing parameter values that were set during a module run. 

The MECM action task below uses SMS_Client.TriggerSchedule, which Microsoft documents as the method that triggers a specified client schedule. 

Module parameters to create

Create these as Text parameters:

ForceRepair = No

CCMInstalled = No

WMIFix = No

RunCCMRepair = No

RunMECMActions = No

RunGPUpdate = No

WinmgmtStopMethod = NotRun


Conditions to use

Use these as Parameter conditions:

Repair branch: WMIFix = Yes

CCM repair: RunCCMRepair = Yes

MECM actions: RunMECMActions = Yes

GP update: RunGPUpdate = Yes


Very important rule for setter tasks

For these tasks:

Task 5

Task 7

Task 19

Task 20

Task 21


Do all of this:

Enable Set parameter with standard output

Pick the matching parameter

Make the script output only Yes or No

Keep exit 0


Do not add Write-Host logging inside those setter tasks.


---

Full build sheet

Task 1

Type: Query Service Properties
Name: Query Winmgmt
Purpose: confirm WMI service exists

Settings

Filter by service name: Winmgmt


Failure handling

Continue on failure



---

Task 2

Type: Query Service Properties
Name: Query CcmExec
Purpose: confirm MECM client service exists

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
Purpose: detect whether MECM client appears installed

Module Parameters tab

Enable Set parameter with standard output

Parameter: CCMInstalled


$CCMInstalled = 'No'

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
    $CCMInstalled = 'Yes'
}

Write-Output $CCMInstalled
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 6

Type: Windows PowerShell Script
Name: Verbose pre-check WMI + ConfigMgr
Purpose: log what is broken before setting WMIFix

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



---

Task 7

Type: Windows PowerShell Script
Name: Set WMIFix parameter
Purpose: set master repair flag

Module Parameters tab

Enable Set parameter with standard output

Parameter: WMIFix


$WMIFix = 'No'

if ('$[ForceRepair]' -eq 'Yes') {
    $WMIFix = 'Yes'
    Write-Output $WMIFix
    exit 0
}

$ccmInstalled = '$[CCMInstalled]'
$repoPath = Join-Path $env:SystemRoot 'System32\wbem\Repository'

if (-not (Test-Path -Path $repoPath -PathType Container)) {
    $WMIFix = 'Yes'
}

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $WMIFix = 'Yes'
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $WMIFix = 'Yes'
    }
}
catch {
    $WMIFix = 'Yes'
}

if ($ccmInstalled -eq 'Yes') {
    try {
        Get-CimInstance -Namespace 'root\ccm' -ClassName 'SMS_Client' -ErrorAction Stop | Out-Null
    }
    catch {
        $WMIFix = 'Yes'
    }

    try {
        Get-CimClass -Namespace 'root\ccm\ClientSDK' -ClassName 'CCM_SoftwareCatalogUtilities' -ErrorAction Stop | Out-Null
    }
    catch {
        $WMIFix = 'Yes'
    }
}

Write-Output $WMIFix
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 8

Type: Parameters (Query)
Name: Show detection parameters
Purpose: display parameter values after detection

Settings

Show all parameters in query summary: enabled


Failure handling

Continue on failure



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
Purpose: stop WMI cleanly, force-stop only if needed

This is the one task where I kept $global:WinmgmtStopMethod, because it is informational only and Ivanti supports multiple-return PowerShell parameters that way. 

$stopMethod = 'NotRun'

try {
    $svc = Get-Service -Name 'Winmgmt' -ErrorAction Stop
}
catch {
    $stopMethod = 'ServiceNotFound'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt service not found: $($_.Exception.Message)"
    exit 1
}

if ($svc.Status -eq 'Stopped') {
    $stopMethod = 'AlreadyStopped'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt already stopped"
    exit 0
}

try {
    Stop-Service -Name 'Winmgmt' -Force -ErrorAction Stop
    Write-Host "Stop-Service issued for Winmgmt"
}
catch {
    Write-Host "Stop-Service failed for Winmgmt: $($_.Exception.Message)"
}

Start-Sleep -Seconds 10
$svc.Refresh()

if ($svc.Status -eq 'Stopped') {
    $stopMethod = 'Graceful'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt stopped cleanly"
    exit 0
}

$scOutput = sc.exe queryex Winmgmt
$pid = 0

foreach ($line in $scOutput) {
    if ($line -match 'PID\s*:\s*(\d+)') {
        $pid = [int]$matches[1]
        break
    }
}

Write-Host "Winmgmt still running. PID: $pid"

if ($pid -le 0) {
    $stopMethod = 'NoPID'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "No valid PID found for Winmgmt"
    exit 1
}

$tasklistOutput = tasklist /svc /fi "PID eq $pid"
$serviceLine = $tasklistOutput | Where-Object { $_ -match "^\s*\S+\s+$pid\s+" }

if ($serviceLine) {
    Write-Host "tasklist /svc result: $serviceLine"
    if ($serviceLine -match ',') {
        $stopMethod = 'SharedProcessRefused'
        $global:WinmgmtStopMethod = $stopMethod
        Write-Host "Refusing to kill PID $pid because multiple services appear tied to it"
        exit 1
    }
}

try {
    Stop-Process -Id $pid -Force -ErrorAction Stop
    Write-Host "Force-killed PID $pid for Winmgmt"
}
catch {
    $stopMethod = 'PIDKillFailed'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Force-kill failed: $($_.Exception.Message)"
    exit 1
}

Start-Sleep -Seconds 5

try {
    $svc = Get-Service -Name 'Winmgmt' -ErrorAction Stop
    if ($svc.Status -eq 'Stopped') {
        $stopMethod = 'PIDKill'
        $global:WinmgmtStopMethod = $stopMethod
        Write-Host "Winmgmt is stopped after PID kill"
        exit 0
    }
    else {
        $stopMethod = 'StillRunning'
        $global:WinmgmtStopMethod = $stopMethod
        Write-Host "Winmgmt still not stopped after PID kill"
        exit 1
    }
}
catch {
    $stopMethod = 'PIDKill'
    $global:WinmgmtStopMethod = $stopMethod
    Write-Host "Winmgmt no longer queryable after PID kill; treating as stopped"
    exit 0
}

Ivanti settings

File extension: ps1


Condition

WMIFix = Yes


Failure handling

Continue on failure



---

Task 12

Type: Windows PowerShell Script
Name: Salvage WMI repository
Purpose: safe WMI salvage

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
Purpose: short settle time after restart

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
Purpose: log health after WMI repair branch

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
Purpose: allow ccmrepair.exe when generic WMI is usable and the EXE exists

Module Parameters tab

Enable Set parameter with standard output

Parameter: RunCCMRepair


$RunCCMRepair = 'No'

if ('$[WMIFix]' -ne 'Yes') {
    Write-Output $RunCCMRepair
    exit 0
}

if ('$[CCMInstalled]' -ne 'Yes') {
    Write-Output $RunCCMRepair
    exit 0
}

$ccmRepair = Join-Path $env:SystemRoot 'CCM\ccmrepair.exe'

if (-not (Test-Path -Path $ccmRepair -PathType Leaf)) {
    Write-Output $RunCCMRepair
    exit 0
}

$wmiOk = $true

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
}
catch {
    $wmiOk = $false
}

try {
    $verify = ((cmd /c "winmgmt /verifyrepository") 2>&1 | Out-String).Trim()
    if ($verify -notmatch 'consistent|is consistent') {
        $wmiOk = $false
    }
}
catch {
    $wmiOk = $false
}

if ($wmiOk) {
    $RunCCMRepair = 'Yes'
}

Write-Output $RunCCMRepair
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 20

Type: Windows PowerShell Script
Name: Set RunMECMActions parameter
Purpose: allow MECM actions only when CCM WMI side is healthy enough

Module Parameters tab

Enable Set parameter with standard output

Parameter: RunMECMActions


$RunMECMActions = 'No'

if ('$[WMIFix]' -ne 'Yes') {
    Write-Output $RunMECMActions
    exit 0
}

if ('$[CCMInstalled]' -ne 'Yes') {
    Write-Output $RunMECMActions
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
    $RunMECMActions = 'Yes'
}

Write-Output $RunMECMActions
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 21

Type: Windows PowerShell Script
Name: Set RunGPUpdate parameter
Purpose: allow GP refresh only when generic WMI is healthy enough

Module Parameters tab

Enable Set parameter with standard output

Parameter: RunGPUpdate


$RunGPUpdate = 'No'

if ('$[WMIFix]' -ne 'Yes') {
    Write-Output $RunGPUpdate
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
    $RunGPUpdate = 'Yes'
}

Write-Output $RunGPUpdate
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 22

Type: Windows PowerShell Script
Name: Run ccmrepair via PowerShell
Purpose: launch ccmrepair.exe

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
Purpose: settle time after client repair

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
Purpose: trigger common MECM client actions

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
Purpose: final readable result and summary

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



---

Task 27

Type: Parameters (Query)
Name: Show final parameters
Purpose: dump final parameter values into job results

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


Grabbed logs I’d add on the key PowerShell tasks

On Tasks 18, 22, 24, and 26, grab these if available:

C:\Windows\CCM\Logs\CCMRepair.log

C:\Windows\CCM\Logs\ClientIDManagerStartup.log

C:\Windows\CCM\Logs\LocationServices.log

C:\Windows\CCM\Logs\PolicyAgent.log

C:\Windows\CCM\Logs\AppDiscovery.log

C:\Windows\CCM\Logs\AppIntentEval.log


If you want, next I’ll turn this into a very short console-click checklist with only: “task number / task type / condition / set-parameter yes-no / parameter name” so you can build it faster without rereading all the scripts.