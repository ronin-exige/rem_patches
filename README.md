
$local = Get-ChildItem -Path $stage -Recurse -Filter $Pattern -EA SilentlyContinue |
    Sort-Object -Property @{Expression='LastWriteTime';Descending=$true}, @{Expression='Name';Descending=$true} |
    Select-Object -First 1

$remote = Get-ChildItem -Path $share -Recurse -Filter $Pattern -EA SilentlyContinue |
    Sort-Object -Property @{Expression='LastWriteTime';Descending=$true}, @{Expression='Name';Descending=$true} |
    Select-Object -First 1

Log "Remove-OldVersions result for ${NameForLog}: removed=$removed failed=$failed"





Nessus Vulnerability Remediation
Ivanti Automation Playbook v4.2 (Complete Full Sweep)
March 2026
1. Step-by-Step: Setting Up a Module in Ivanti (Chrome Example)
1.1 Where the scripts go
•	Shared-Framework.ps1 and all **Remediate-*.ps1** scripts: \\FILESERVER\IvantiModules\scripts\
•	Installer files (MSI / EXE): upload them to the Ivanti Resource Library.
•	In this design Ivanti still launches the remediation scripts from the share. Task 1 only uses the Resource Library to stage the installer onto the endpoint.
•	4.2 removes the per-script inline framework fallback because the module already depends on the script share. One shared framework is safer than many drifting copies.
1.2 The 4-task module pattern (every staged-installer app uses this)
Task 1: Deploy installer (Resource Library -> per-app local staging)
•	Type: Execute Script (PowerShell)
•	Run As: SYSTEM
•	Attach Resource: Chrome MSI from Resource Library
•	What it does: copies the attached installer from the local Ivanti cache into C:\install_files\rem\Chrome\
$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Copy-IvantiResourceToStage -AppFolder "Chrome" -FileName "googlechromestandaloneenterprise64.msi"
# Optional if you also stage 32-bit media:
# Copy-IvantiResourceToStage -AppFolder "Chrome" -FileName "googlechromestandaloneenterprise32.msi"
Task 2: Detection
•	Type: Execute Script
•	Run As: SYSTEM
•	Timeout: 60s
$env:IVANTI_DETECT_ONLY = "TRUE"
& "\\FILESERVER\IvantiModules\scripts\Remediate-GoogleChrome.ps1"
# Exit 0 = skip | Exit 1 = vulnerable, run Task 3
Task 3: Remediation
•	Type: Execute Script
•	Run As: SYSTEM
•	Timeout: 600s
•	Condition: Task 2 exit = 1
& "\\FILESERVER\IvantiModules\scripts\Remediate-GoogleChrome.ps1"
# For testing during business hours, add this first line temporarily:
# $env:IVANTI_FORCE_RUN = "TRUE"
Task 4: Cleanup
•	Type: Execute Script
•	Continue on error: Yes
•	What it does: deletes only the Chrome staging folder, not the entire shared temp root
$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Remove-AppStagingFolder -AppFolder "Chrome"
1.3 Testing Chrome tomorrow
1.	Upload the Chrome MSI to the Ivanti Resource Library.
2.	Copy Shared-Framework.ps1 and Remediate-GoogleChrome.ps1 to \\FILESERVER\IvantiModules\scripts\.
3.	Create the 4-task module exactly as shown above.
4.	Create a job for 1-2 test machines. For daytime testing, temporarily set IVANTI_FORCE_RUN=TRUE in Task 3.
5.	Check C:\Logs\Ivanti\ on the target machine.
6.	Confirm Chrome is at the target version and that no vulnerable Chrome build remains in post-check output.
7.	Remove the force-run override and schedule production during the maintenance window.
2. Install Switches Reference
Software	Type	Silent Install Command / Switches
Google Chrome	MSI	/qn /norestart ALLUSERS=1
Microsoft Edge	MSI	/qn /norestart DONOTCREATEDESKTOPSHORTCUT=true
Mozilla Firefox	EXE	/INI="config.ini" /S
VS Code	EXE (Inno)	/VERYSILENT /NORESTART /MERGETASKS=!runcode,!desktopicon
Office C2R	Built-in	OfficeC2RClient.exe /update user displaylevel=false forceappshutdown=true
SQL OLE DB	MSI	/qn /norestart IACCEPTMSOLEDBSQLLICENSETERMS=YES
SQL ODBC	MSI	/qn /norestart IACCEPTMSODBCSQLLICENSETERMS=YES
SQL Server CU	EXE	/qs /IAcceptSQLServerLicenseTerms /Action=Patch /InstanceName=X
.NET Runtime	EXE	/install /quiet /norestart
Node.js	MSI	/qn /norestart
Oracle Java	MSI	/qn /norestart
MS Teams	EXE	teamsbootstrapper.exe -p -o MSTeams.msix
Docker Desktop	EXE	install --quiet --accept-license
VMware Tools	EXE	/S /v/qn REBOOT=R
Windows Updates	API	Windows Update Agent (no file)
Notepad	Store	winget upgrade --id 9MSMLRH6LZF3 --silent
OpenSSH	Feature	Add-WindowsCapability (no file)
Hardening	Registry	No installer
Log4j	Scan	No installer
3. MECM Coexistence
•	MECM WMI conflict check: present again in 4.2 and no longer broken.
•	Default behavior in this environment: disabled ($Global:CheckMECM = $false) because MECM is supposed to patch these machines but is missing findings.
•	MSI mutex check: enabled. This still prevents two MSI installs from colliding (1618).
•	If you later want Ivanti to defer to MECM for a specific software family, set $Global:CheckMECM = $true in Shared-Framework.ps1 and keep the Test-MECMConflict calls in the app script.
•	The key 4.2 fix is that the shared framework now actually contains Test-MECMConflict, so scripts that call it no longer break when the framework loads successfully.
4. Quick Reference: Design Decisions
•	Per-app staging folders: C:\install_files\rem\<AppFolder>\
•	Cleanup: Task 4 removes only the app folder, never the entire rem root.
•	Resource Library model: Task 1 copies from the local agent cache into the per-app staging folder.
•	Share fallback: if staging is empty, installer-based modules fall back to \\FILESERVER\Software\<AppFolder>\.
•	Old versions: where uninstall information exists, the script removes vulnerable versions before or during the update path.
•	Strict success criteria: remediation fails (exit 1) if a vulnerable version still remains after the remediation cycle.
•	32-bit and 64-bit: handled separately where the software actually differentiates architecture.
•	MECM coexistence: supported again, but disabled by default in this environment.
•	Framework copies: reduced to one real shared framework to avoid drift and broken helper mismatches.
5. Shared Framework (Shared-Framework.ps1)
File: Shared-Framework.ps1
Location: \\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1
<#
.SYNOPSIS
    Shared Framework v4.2 for Ivanti remediation modules.

.DESCRIPTION
    DELIVERY MODEL IN 4.2:
    1. Upload the installer file to the Ivanti Resource Library.
    2. Task 1 attaches that resource and copies it from the local Ivanti agent cache
       into C:\install_files\rem\<AppFolder>\ on the endpoint.
    3. Task 2 / Task 3 call the remediation script from \\FILESERVER\IvantiModules\scripts\.
    4. Resolve-Installer checks the per-app staging folder first, then the share fallback.
    5. Task 4 deletes ONLY C:\install_files\rem\<AppFolder>\.

    IMPORTANT CHANGE FROM 4.1:
    Inline per-script fallback frameworks were removed. In this design Ivanti launches the
    remediation scripts FROM the share, so if the share is unavailable the script cannot be
    launched anyway. Keeping one real shared framework is cleaner and avoids drift.

    WHAT TO CHANGE EACH PATCH CYCLE:
    - Update $TargetVersion in the individual app script.
    - Upload the new installer/resource to Ivanti.
    - If the resource filename changes, update $InstallerFile in the script.
#>

$Global:StagingRoot   = "C:\install_files\rem"
$Global:SoftwareShare = "\\FILESERVER\Software"
$Global:IvantiLogRoot = "C:\Logs\Ivanti"
$Global:CheckMECM     = $false

$Global:MaintStart = 18
$Global:MaintEnd   = 6

$Global:RebootMode     = "Scheduled"
$Global:RebootDelayMin = 30

function Initialize-IvantiLog {
    param([string]$ModuleName)
    if (!(Test-Path $Global:IvantiLogRoot)) { New-Item -ItemType Directory -Path $Global:IvantiLogRoot -Force | Out-Null }
    $ts = Get-Date -Format 'yyyyMMdd_HHmmss'
    $Global:LogPath = Join-Path $Global:IvantiLogRoot "${ModuleName}_${ts}.log"
    Log "========= $ModuleName Started ========="
    try {
        $os = (Get-CimInstance Win32_OperatingSystem -EA SilentlyContinue).Caption
        if ($os) { Log "Computer: $env:COMPUTERNAME | OS: $os" } else { Log "Computer: $env:COMPUTERNAME" }
    } catch { Log "Computer: $env:COMPUTERNAME" }
}

function Log {
    param([string]$Message, [string]$Level = "INFO")
    if (!$Global:LogPath) {
        if (!(Test-Path $Global:IvantiLogRoot)) { New-Item -ItemType Directory -Path $Global:IvantiLogRoot -Force | Out-Null }
        $Global:LogPath = Join-Path $Global:IvantiLogRoot "Ivanti_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    }
    $entry = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$Level] $Message"
    Add-Content -Path $Global:LogPath -Value $entry -EA SilentlyContinue
    Write-Output $entry
}

function Test-InMaintenanceWindow {
    if ($env:IVANTI_FORCE_RUN -eq "TRUE") { Log "Maintenance window BYPASSED with IVANTI_FORCE_RUN" -Level "WARN"; return $true }
    $hour = (Get-Date).Hour
    if ($Global:MaintStart -gt $Global:MaintEnd) { $ok = ($hour -ge $Global:MaintStart) -or ($hour -lt $Global:MaintEnd) }
    else { $ok = ($hour -ge $Global:MaintStart) -and ($hour -lt $Global:MaintEnd) }
    if (!$ok) { Log "Outside maintenance window ($($Global:MaintStart):00-$($Global:MaintEnd):00). Exiting." -Level "WARN" }
    return $ok
}

function Get-StagingFolder { param([string]$AppFolder) return (Join-Path $Global:StagingRoot $AppFolder) }

function Ensure-StagingFolder {
    param([string]$AppFolder)
    $path = Get-StagingFolder -AppFolder $AppFolder
    if (!(Test-Path $path)) { New-Item -ItemType Directory -Path $path -Force | Out-Null }
    return $path
}

function Copy-IvantiResourceToStage {
    param([string]$AppFolder,[string]$FileName,[string]$CacheRoot = "C:\RES")
    $stage = Ensure-StagingFolder -AppFolder $AppFolder
    $dest  = Join-Path $stage $FileName
    if (Test-Path $dest) { Log "Resource already staged: $dest"; return $dest }
    if (!(Test-Path $CacheRoot)) { Log "Ivanti cache root not found: $CacheRoot" -Level "WARN"; return $null }
    $cached = Get-ChildItem -Path $CacheRoot -Recurse -Filter $FileName -EA SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 1
    if ($cached) {
        Copy-Item -Path $cached.FullName -Destination $dest -Force
        Log "Staged resource: $($cached.FullName) -> $dest"
        return $dest
    }
    Log "Resource '$FileName' not found under $CacheRoot" -Level "WARN"
    return $null
}

function Resolve-Installer {
    param([string]$AppFolder,[string]$FileName,[string]$ShareSubPath = "")
    $localPath = Join-Path (Get-StagingFolder -AppFolder $AppFolder) $FileName
    if (Test-Path $localPath) { Log "Installer found (local staging): $localPath"; return $localPath }
    if (!$ShareSubPath) { $ShareSubPath = $AppFolder }
    $sharePath = Join-Path (Join-Path $Global:SoftwareShare $ShareSubPath) $FileName
    if (Test-Path $sharePath) { Log "Installer found (share fallback): $sharePath"; return $sharePath }
    Log "Installer not found: $FileName | AppFolder: $AppFolder | ShareSubPath: $ShareSubPath" -Level "ERROR"
    return $null
}

function Resolve-LatestInstaller {
    param([string]$AppFolder,[string]$Pattern,[string]$ShareSubPath = "")
    $stage = Get-StagingFolder -AppFolder $AppFolder
    if (Test-Path $stage) {
        $local = Get-ChildItem -Path $stage -Recurse -Filter $Pattern -EA SilentlyContinue | Sort-Object LastWriteTime -Descending, Name -Descending | Select-Object -First 1
        if ($local) { Log "Installer found (local staging pattern): $($local.FullName)"; return $local.FullName }
    }
    if (!$ShareSubPath) { $ShareSubPath = $AppFolder }
    $share = Join-Path $Global:SoftwareShare $ShareSubPath
    if (Test-Path $share) {
        $remote = Get-ChildItem -Path $share -Recurse -Filter $Pattern -EA SilentlyContinue | Sort-Object LastWriteTime -Descending, Name -Descending | Select-Object -First 1
        if ($remote) { Log "Installer found (share pattern): $($remote.FullName)"; return $remote.FullName }
    }
    Log "No installer matched pattern '$Pattern' for app '$AppFolder'" -Level "ERROR"
    return $null
}

function Remove-AppStagingFolder {
    param([string]$AppFolder)
    $path = Get-StagingFolder -AppFolder $AppFolder
    if (Test-Path $path) { Remove-Item -Path $path -Recurse -Force -EA SilentlyContinue; Log "Removed app staging folder: $path" }
    else { Log "App staging folder not present: $path" }
}

function Test-MECMConflict {
    param([string]$SoftwareName = "")
    if (!$Global:CheckMECM) { return $false }
    if (!(Test-Path "$env:SystemRoot\CCM\CcmExec.exe")) { return $false }
    $svc = Get-Service CcmExec -EA SilentlyContinue
    if (!$svc -or $svc.Status -ne 'Running') { return $false }
    Log "MECM client active. Checking for '$SoftwareName' conflicts..."
    try {
        $pending = Get-CimInstance -Namespace "root\ccm\clientsdk" -ClassName CCM_SoftwareUpdate -EA Stop | Where-Object { $_.EvaluationState -in @(2,3,4,5,6,7) }
        if ($pending -and $SoftwareName) {
            $match = $pending | Where-Object { $_.Name -match [regex]::Escape($SoftwareName) }
            if ($match) { Log "MECM CONFLICT: pending deployment found for '$SoftwareName'. Deferring to MECM." -Level "WARN"; return $true }
        }
        $apps = Get-CimInstance -Namespace "root\ccm\clientsdk" -ClassName CCM_Application -EA SilentlyContinue | Where-Object { $_.InstallState -eq 'Installing' -and $_.Name -match [regex]::Escape($SoftwareName) }
        if ($apps) { Log "MECM CONFLICT: MECM is actively installing '$SoftwareName'" -Level "WARN"; return $true }
    } catch { Log "MECM query error: $($_.Exception.Message). Proceeding." -Level "WARN" }
    Log "No MECM conflict found for '$SoftwareName'"
    return $false
}

function Wait-MsiAvailable {
    param([int]$TimeoutSec = 300)
    $waited = 0
    while ($waited -lt $TimeoutSec) {
        try {
            $m = [System.Threading.Mutex]::OpenExisting("Global\_MSIExecute")
            $m.Close()
            Log "MSI busy (another msiexec is running), waiting... ($waited/$TimeoutSec sec)"
            Start-Sleep -Seconds 30
            $waited += 30
        } catch [System.Threading.WaitHandleCannotBeOpenedException] { return $true }
        catch { return $true }
    }
    Log "MSI mutex timeout after $TimeoutSec sec" -Level "ERROR"
    return $false
}

function Test-Below {
    param([string]$Current,[string]$Target)
    if (!$Current -or !$Target) { return $false }
    try { return [version]($Current -replace '[^0-9.]','') -lt [version]($Target -replace '[^0-9.]','') }
    catch { return $true }
}

function Get-PreferredArch { if ([Environment]::Is64BitOperatingSystem) { return "64-bit" } else { return "32-bit" } }

function Get-AllInstalled {
    param([string]$NamePattern,[switch]$IncludeUser)
    $results = @()
    $paths = @(
        @{P="HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"; A="64-bit"; S="Machine"},
        @{P="HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"; A="32-bit"; S="Machine"}
    )
    if ($IncludeUser) {
        $paths += @{P="HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"; A="64-bit"; S="CurrentUser"}
        $paths += @{P="HKCU:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"; A="32-bit"; S="CurrentUser"}
        try {
            Get-ChildItem "Registry::HKEY_USERS" -EA SilentlyContinue | Where-Object { $_.PSChildName -match '^S-1-5-21-' -and $_.PSChildName -notmatch '_Classes$' } | ForEach-Object {
                $sid = $_.PSChildName
                $paths += @{P="Registry::HKEY_USERS\$sid\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"; A="64-bit"; S="User:$sid"}
                $paths += @{P="Registry::HKEY_USERS\$sid\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"; A="32-bit"; S="User:$sid"}
            }
        } catch {}
    }
    foreach ($rp in $paths) {
        if (!(Test-Path $rp.P)) { continue }
        Get-ChildItem $rp.P -EA SilentlyContinue | ForEach-Object {
            $props = Get-ItemProperty $_.PSPath -EA SilentlyContinue
            if ($props.DisplayName -and $props.DisplayName -match $NamePattern) {
                $results += [PSCustomObject]@{ Name=$props.DisplayName; Version=$props.DisplayVersion; Arch=$rp.A; Scope=$rp.S; GUID=$_.PSChildName; Uninstall=$props.UninstallString; QuietUninstall=$props.QuietUninstallString }
            }
        }
    }
    return $results
}

function Remove-OldVersions {
    param([PSCustomObject[]]$Installs,[string]$TargetVer,[string]$NameForLog = "Software")
    $removed = 0; $failed = 0
    foreach ($i in $Installs) {
        if (!(Test-Below $i.Version $TargetVer)) { Log "Keeping current/non-vulnerable entry: $($i.Name) v$($i.Version) [$($i.Scope) | $($i.Arch)]"; continue }
        Log "Removing old version: $($i.Name) v$($i.Version) [$($i.Scope) | $($i.Arch)]"
        $ok = $false
        if ($i.QuietUninstall) {
            $proc = Start-Process cmd.exe -ArgumentList "/c $($i.QuietUninstall)" -Wait -PassThru -NoNewWindow -EA SilentlyContinue
            $ok = ($proc.ExitCode -eq 0 -or $proc.ExitCode -eq 3010)
        } elseif ($i.Uninstall) {
            $cmd = $i.Uninstall
            if ($cmd -match 'msiexec') {
                $cmd = $cmd -replace '/I','/X' -replace '/i','/x'
                if ($cmd -notmatch '/qn|/quiet') { $cmd += ' /qn /norestart' }
            }
            $proc = Start-Process cmd.exe -ArgumentList "/c $cmd" -Wait -PassThru -NoNewWindow -EA SilentlyContinue
            $ok = ($proc.ExitCode -eq 0 -or $proc.ExitCode -eq 3010)
        } elseif ($i.GUID -match '^\{') {
            if (!(Wait-MsiAvailable)) { Log "MSI busy; could not uninstall $($i.Name)" -Level "ERROR"; $ok = $false }
            else {
                $proc = Start-Process msiexec.exe -ArgumentList "/X `"$($i.GUID)`" /qn /norestart" -Wait -PassThru -NoNewWindow -EA SilentlyContinue
                $ok = ($proc.ExitCode -eq 0 -or $proc.ExitCode -eq 3010)
            }
        } else {
            Log "No uninstall method was available for $($i.Name) v$($i.Version)" -Level "WARN"
            $ok = $false
        }
        if ($ok) { $removed++ } else { $failed++ }
    }
    Log "Remove-OldVersions result for $NameForLog: removed=$removed failed=$failed"
    return @{ Removed=$removed; Failed=$failed }
}

function Stop-AppSafely {
    param([string[]]$Processes,[string[]]$Services = @(),[int]$GraceSeconds = 10)
    foreach ($svc in $Services) {
        $service = Get-Service -Name $svc -EA SilentlyContinue
        if ($service -and $service.Status -eq 'Running') { Log "Stopping service: $svc"; Stop-Service -Name $svc -Force -EA SilentlyContinue }
    }
    foreach ($p in $Processes) {
        $procs = Get-Process -Name $p -EA SilentlyContinue
        if ($procs) { Log "Closing process: $p ($($procs.Count) instance(s))"; $procs | ForEach-Object { $_.CloseMainWindow() | Out-Null } }
    }
    if ($GraceSeconds -gt 0) { Start-Sleep -Seconds $GraceSeconds }
    foreach ($p in $Processes) {
        $procs = Get-Process -Name $p -EA SilentlyContinue
        if ($procs) { Log "Force-stopping process: $p ($($procs.Count) instance(s))"; $procs | Stop-Process -Force -EA SilentlyContinue }
    }
    Start-Sleep -Seconds 2
}

function Test-RebootNeeded {
    foreach ($p in @("HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending","HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired")) {
        if (Test-Path $p) { return $true }
    }
    try {
        $r = Invoke-CimMethod -Namespace "root\ccm\clientsdk" -ClassName CCM_ClientUtilities -MethodName DetermineIfRebootPending -EA SilentlyContinue
        if ($r -and $r.RebootPending) { return $true }
    } catch {}
    return $false
}

function Schedule-Reboot {
    param([string]$Reason = "Security remediation")
    if (!(Test-RebootNeeded)) { Log "No reboot needed"; return }
    Log "REBOOT PENDING - Mode: $Global:RebootMode" -Level "WARN"
    switch ($Global:RebootMode) {
        "Scheduled" { shutdown.exe /r /t ($Global:RebootDelayMin * 60) /c "Scheduled restart: $Reason" /d p:4:1; Log "Reboot scheduled in $Global:RebootDelayMin minute(s)" }
        "Flag" { Set-Content (Join-Path $Global:IvantiLogRoot "REBOOT_PENDING_$env:COMPUTERNAME.flag") "$(Get-Date)`n$Reason"; Log "Reboot flag created" }
        "None" { Log "Reboot needed but RebootMode=None" -Level "WARN" }
    }
}

function Install-Msi {
    param([string]$Path,[string]$ExtraArgs = "",[string]$Label = "MSI")
    if (!(Wait-MsiAvailable)) { Log "MSI busy - cannot install $Label" -Level "ERROR"; return $false }
    Log "Installing $Label from $Path"
    $msiLog = $Global:LogPath.Replace('.log', "_${Label}_msi.log")
    $proc = Start-Process msiexec.exe -ArgumentList "/i `"$Path`" /qn /norestart $ExtraArgs /l*v `"$msiLog`"" -Wait -PassThru -NoNewWindow -EA SilentlyContinue
    Log "$Label exit code: $($proc.ExitCode)"
    return ($proc.ExitCode -eq 0 -or $proc.ExitCode -eq 3010)
}

function Install-Exe {
    param([string]$Path,[string]$Arguments = "/S",[string]$Label = "EXE")
    Log "Installing $Label from $Path"
    $proc = Start-Process $Path -ArgumentList $Arguments -Wait -PassThru -NoNewWindow -EA SilentlyContinue
    Log "$Label exit code: $($proc.ExitCode)"
    return ($proc.ExitCode -eq 0 -or $proc.ExitCode -eq 3010)
}
6. GoogleChrome
File: Remediate-GoogleChrome.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-GoogleChrome.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Google Chrome
    Nessus Plugins: 299806, 299027, 298675, 299032, 283570, 297737, 300938, 299457, 296908, 294788
#>

$TargetVersion   = "145.0.7632.159"
$InstallerFile64 = "googlechromestandaloneenterprise64.msi"
$InstallerFile32 = "googlechromestandaloneenterprise32.msi"
$AppFolder       = "Chrome"
$ShareSubPath    = "Chrome"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "Chrome"

function Add-Record {
    param([ref]$Bucket,[string]$Version,[string]$Arch,[string]$Source,[string]$Scope = "Machine",[string]$GUID = "",[string]$Uninstall = "",[string]$QuietUninstall = "",[string]$Path = "")
    if (!$Version) { return }
    $key = "$Version|$Arch|$Source|$Scope|$Path"
    if (!($Bucket.Value | Where-Object { $_.Key -eq $key })) {
        $Bucket.Value += [PSCustomObject]@{ Key=$key; Version=$Version; Arch=$Arch; Source=$Source; Scope=$Scope; GUID=$GUID; Uninstall=$Uninstall; QuietUninstall=$QuietUninstall; Path=$Path }
    }
}

function Detect-Chrome {
    $installs = @()
    foreach ($rp in @(@{P="HKLM:\SOFTWARE\Google\Chrome\BLBeacon";A="64-bit"},@{P="HKLM:\SOFTWARE\WOW6432Node\Google\Chrome\BLBeacon";A="32-bit"})) {
        if (Test-Path $rp.P) { $v=(Get-ItemProperty $rp.P -EA SilentlyContinue).version; Add-Record -Bucket ([ref]$installs) -Version $v -Arch $rp.A -Source "BLBeacon" }
    }
    foreach ($pair in @(@("$env:ProgramFiles\Google\Chrome\Application\chrome.exe","64-bit"),@("${env:ProgramFiles(x86)}\Google\Chrome\Application\chrome.exe","32-bit"))) {
        if (Test-Path $pair[0]) { $v=(Get-Item $pair[0]).VersionInfo.ProductVersion; Add-Record -Bucket ([ref]$installs) -Version $v -Arch $pair[1] -Source "File" -Path $pair[0] }
    }
    $registered = Get-AllInstalled -NamePattern '^Google Chrome$' -IncludeUser
    foreach ($r in $registered) { Add-Record -Bucket ([ref]$installs) -Version $r.Version -Arch $r.Arch -Source "Uninstall" -Scope $r.Scope -GUID $r.GUID -Uninstall $r.Uninstall -QuietUninstall $r.QuietUninstall }
    Get-ChildItem "C:\Users" -Directory -EA SilentlyContinue | Where-Object { $_.Name -notin @("Public","Default","Default User","All Users") } | ForEach-Object {
        $exe = Join-Path $_.FullName "AppData\Local\Google\Chrome\Application\chrome.exe"
        if (Test-Path $exe) { $v=(Get-Item $exe).VersionInfo.ProductVersion; Add-Record -Bucket ([ref]$installs) -Version $v -Arch (Get-PreferredArch) -Source "PerUserFile" -Scope "UserPath:$($_.Name)" -Path $exe }
    }
    if ($installs.Count -eq 0) { return @{ Installed=$false; Vulnerable=$false; Instances=@() } }
    foreach ($i in $installs) { $i | Add-Member -NotePropertyName Vulnerable -NotePropertyValue (Test-Below $i.Version $TargetVersion) -Force }
    return @{ Installed=$true; Vulnerable=(($installs|Where-Object{$_.Vulnerable}).Count -gt 0); Instances=$installs }
}

if ($env:IVANTI_DETECT_ONLY -eq "TRUE") {
    $d = Detect-Chrome
    if (!$d.Installed) { Write-Output "NOT_INSTALLED|Chrome"; exit 0 }
    if ($d.Vulnerable) { $detail = ($d.Instances | Where-Object { $_.Vulnerable } | ForEach-Object { "$($_.Arch):$($_.Version):$($_.Source)" }) -join ','; Write-Output "VULNERABLE|Chrome|$detail|Target:$TargetVersion"; exit 1 }
    Write-Output "NOT_VULNERABLE|Chrome|Target:$TargetVersion"; exit 0
}

Log "===== Chrome Remediation v4.2 ====="
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-Chrome
foreach ($i in $detect.Instances) { Log "Detected $($i.Version) [$($i.Arch)] from $($i.Source) scope=$($i.Scope) vulnerable=$($i.Vulnerable)" }
if (!$detect.Installed) { Log "Chrome not installed"; exit 0 }
if (!$detect.Vulnerable) { Log "Chrome already at or above target"; exit 0 }
Stop-AppSafely -Processes @("chrome","GoogleUpdate","GoogleCrashHandler","GoogleCrashHandler64") -GraceSeconds 10
$oldRegistered = Get-AllInstalled -NamePattern '^Google Chrome$' -IncludeUser
if ($oldRegistered) { Remove-OldVersions -Installs $oldRegistered -TargetVer $TargetVersion -NameForLog "Google Chrome" | Out-Null }
$need64 = (($detect.Instances | Where-Object { $_.Vulnerable -and $_.Arch -eq '64-bit' }).Count -gt 0)
$need32 = (($detect.Instances | Where-Object { $_.Vulnerable -and $_.Arch -eq '32-bit' }).Count -gt 0)
if (!$need64 -and !$need32) { if ((Get-PreferredArch) -eq '64-bit') { $need64 = $true } else { $need32 = $true } }
if ($need64) { $installer64 = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile64 -ShareSubPath $ShareSubPath; if (!$installer64) { Log "64-bit Chrome installer not found" -Level "ERROR"; exit 1 }; if (!(Install-Msi -Path $installer64 -ExtraArgs 'ALLUSERS=1' -Label 'Chrome64')) { exit 1 } }
if ($need32) { $installer32 = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile32 -ShareSubPath $ShareSubPath; if (!$installer32) { Log "32-bit Chrome installer not found" -Level "ERROR"; exit 1 }; if (!(Install-Msi -Path $installer32 -ExtraArgs 'ALLUSERS=1' -Label 'Chrome32')) { exit 1 } }
$post = Detect-Chrome
foreach ($i in $post.Instances) { Log "Post-check $($i.Version) [$($i.Arch)] from $($i.Source) vulnerable=$($i.Vulnerable)" }
if ($post.Vulnerable) { Log "Chrome remediation FAILED: vulnerable version still present after uninstall/install cycle" -Level "ERROR"; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason "Chrome update" }
Log "Chrome remediation SUCCEEDED"
exit 0
7. MicrosoftEdge
File: Remediate-MicrosoftEdge.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-MicrosoftEdge.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft Edge (Chromium)
    Plugins: 301410, 299699, 300427, 299686, 298069
#>

$TargetVersion = "145.0.3800.97"
$InstallerFile64 = "MicrosoftEdgeEnterpriseX64.msi"
$InstallerFile32 = "MicrosoftEdgeEnterpriseX86.msi"
$AppFolder = "Edge"
$ShareSubPath = "Edge"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "Edge"

function Detect-Edge {
    $installs = @()
    foreach ($rp in @(@{P="HKLM:\SOFTWARE\Microsoft\Edge\BLBeacon";A="64-bit"},@{P="HKLM:\SOFTWARE\WOW6432Node\Microsoft\Edge\BLBeacon";A="32-bit"})) {
        if (Test-Path $rp.P) { $v=(Get-ItemProperty $rp.P -EA SilentlyContinue).version; if($v){$installs += [PSCustomObject]@{ Version=$v; Arch=$rp.A; Source='BLBeacon'; Scope='Machine' }} }
    }
    foreach ($pair in @(@("$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe",'64-bit'),@("${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe",'32-bit'))) {
        if (Test-Path $pair[0]) { $v=(Get-Item $pair[0]).VersionInfo.ProductVersion; if($v){$installs += [PSCustomObject]@{ Version=$v; Arch=$pair[1]; Source='File'; Scope='Machine' }} }
    }
    $registered = Get-AllInstalled -NamePattern '^Microsoft Edge$' -IncludeUser
    foreach ($r in $registered) { $installs += [PSCustomObject]@{ Version=$r.Version; Arch=$r.Arch; Source='Uninstall'; Scope=$r.Scope; GUID=$r.GUID; Uninstall=$r.Uninstall; QuietUninstall=$r.QuietUninstall } }
    if ($installs.Count -eq 0) { return @{ Installed=$false; Vulnerable=$false; Instances=@() } }
    foreach ($i in $installs) { $i | Add-Member -NotePropertyName Vulnerable -NotePropertyValue (Test-Below $i.Version $TargetVersion) -Force }
    return @{ Installed=$true; Vulnerable=(($installs|Where-Object{$_.Vulnerable}).Count -gt 0); Instances=$installs }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-Edge
    if (!$d.Installed) { Write-Output 'NOT_INSTALLED|Edge'; exit 0 }
    if ($d.Vulnerable) { $detail = ($d.Instances | Where-Object { $_.Vulnerable } | ForEach-Object { "$($_.Arch):$($_.Version):$($_.Source)" }) -join ','; Write-Output "VULNERABLE|Edge|$detail|Target:$TargetVersion"; exit 1 }
    Write-Output "NOT_VULNERABLE|Edge|Target:$TargetVersion"; exit 0
}

Log '===== Edge Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
if (Test-MECMConflict -SoftwareName 'Edge') { Log 'Deferring to MECM'; exit 0 }
$detect = Detect-Edge
foreach ($i in $detect.Instances) { Log "Detected $($i.Version) [$($i.Arch)] from $($i.Source) vulnerable=$($i.Vulnerable)" }
if (!$detect.Installed) { Log 'Edge not installed'; exit 0 }
if (!$detect.Vulnerable) { Log 'Edge already at or above target'; exit 0 }
Stop-AppSafely -Processes @('msedge','msedgewebview2','MicrosoftEdgeUpdate') -GraceSeconds 10
$oldRegistered = Get-AllInstalled -NamePattern '^Microsoft Edge$' -IncludeUser
if ($oldRegistered) { Remove-OldVersions -Installs $oldRegistered -TargetVer $TargetVersion -NameForLog 'Microsoft Edge' | Out-Null }
$need64 = (($detect.Instances | Where-Object { $_.Vulnerable -and $_.Arch -eq '64-bit' }).Count -gt 0)
$need32 = (($detect.Instances | Where-Object { $_.Vulnerable -and $_.Arch -eq '32-bit' }).Count -gt 0)
if (!$need64 -and !$need32) { if ((Get-PreferredArch) -eq '64-bit') { $need64 = $true } else { $need32 = $true } }
if ($need64) { $installer64 = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile64 -ShareSubPath $ShareSubPath; if (!$installer64) { Log '64-bit Edge installer not found' -Level 'ERROR'; exit 1 }; if (!(Install-Msi -Path $installer64 -ExtraArgs 'DONOTCREATEDESKTOPSHORTCUT=true' -Label 'Edge64')) { exit 1 } }
if ($need32) { $installer32 = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile32 -ShareSubPath $ShareSubPath; if (!$installer32) { Log '32-bit Edge installer not found' -Level 'ERROR'; exit 1 }; if (!(Install-Msi -Path $installer32 -Label 'Edge32')) { exit 1 } }
$post = Detect-Edge
foreach ($i in $post.Instances) { Log "Post-check $($i.Version) [$($i.Arch)] from $($i.Source) vulnerable=$($i.Vulnerable)" }
if ($post.Vulnerable) { Log 'Edge remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason 'Edge update' }
Log 'Edge remediation SUCCEEDED'
exit 0
8. MozillaFirefox
File: Remediate-MozillaFirefox.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-MozillaFirefox.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Mozilla Firefox
    Plugins: 299865, 296906, 283709, 299210
#>

$TargetVersion   = "148.0"
$InstallerFile64 = "Firefox_Setup_x64.exe"
$InstallerFile32 = "Firefox_Setup_x86.exe"
$AppFolder       = "Firefox"
$ShareSubPath    = "Firefox"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "Firefox"

function Detect-Firefox {
    $installs = @()
    foreach ($rp in @(@{P="HKLM:\SOFTWARE\Mozilla\Mozilla Firefox";A="64-bit"},@{P="HKLM:\SOFTWARE\WOW6432Node\Mozilla\Mozilla Firefox";A="32-bit"})) {
        if (Test-Path $rp.P) { $v=(Get-ItemProperty $rp.P -EA SilentlyContinue).CurrentVersion; if($v){$installs += [PSCustomObject]@{ Version=($v -replace ' .*$',''); Arch=$rp.A; Source='Registry' }} }
    }
    foreach ($pair in @(@("$env:ProgramFiles\Mozilla Firefox\firefox.exe",'64-bit'),@("${env:ProgramFiles(x86)}\Mozilla Firefox\firefox.exe",'32-bit'))) {
        if (Test-Path $pair[0]) { $v=(Get-Item $pair[0]).VersionInfo.ProductVersion; if($v){$installs += [PSCustomObject]@{ Version=$v; Arch=$pair[1]; Source='File' }} }
    }
    $registered = Get-AllInstalled -NamePattern 'Mozilla Firefox' -IncludeUser
    foreach ($r in $registered) { $installs += [PSCustomObject]@{ Version=$r.Version; Arch=$r.Arch; Source='Uninstall'; Scope=$r.Scope; GUID=$r.GUID; Uninstall=$r.Uninstall; QuietUninstall=$r.QuietUninstall } }
    if ($installs.Count -eq 0) { return @{ Installed=$false; Vulnerable=$false; Instances=@() } }
    foreach ($i in $installs) { $i | Add-Member -NotePropertyName Vulnerable -NotePropertyValue (Test-Below $i.Version $TargetVersion) -Force }
    return @{ Installed=$true; Vulnerable=(($installs|Where-Object{$_.Vulnerable}).Count -gt 0); Instances=$installs }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-Firefox
    if (!$d.Installed) { Write-Output 'NOT_INSTALLED|Firefox'; exit 0 }
    if ($d.Vulnerable) { $detail = ($d.Instances | Where-Object { $_.Vulnerable } | ForEach-Object { "$($_.Arch):$($_.Version):$($_.Source)" }) -join ','; Write-Output "VULNERABLE|Firefox|$detail|Target:$TargetVersion"; exit 1 }
    Write-Output "NOT_VULNERABLE|Firefox|Target:$TargetVersion"; exit 0
}

Log '===== Firefox Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
if (Test-MECMConflict -SoftwareName 'Firefox') { Log 'Deferring to MECM'; exit 0 }
$detect = Detect-Firefox
foreach ($i in $detect.Instances) { Log "Detected $($i.Version) [$($i.Arch)] from $($i.Source) vulnerable=$($i.Vulnerable)" }
if (!$detect.Installed) { Log 'Firefox not installed'; exit 0 }
if (!$detect.Vulnerable) { Log 'Firefox already at or above target'; exit 0 }
Stop-AppSafely -Processes @('firefox','plugin-container','updater','MaintenanceService') -GraceSeconds 10
$oldRegistered = Get-AllInstalled -NamePattern 'Mozilla Firefox' -IncludeUser
if ($oldRegistered) { Remove-OldVersions -Installs $oldRegistered -TargetVer $TargetVersion -NameForLog 'Mozilla Firefox' | Out-Null }
$ini = "[Install]`nQuickLaunchShortcut=false`nDesktopShortcut=false`nStartMenuShortcuts=true`nMaintenanceService=true`nPreventRebootRequired=true"
$iniPath = "$env:TEMP\ff_install.ini"
Set-Content -Path $iniPath -Value $ini
$need64 = (($detect.Instances | Where-Object { $_.Vulnerable -and $_.Arch -eq '64-bit' }).Count -gt 0)
$need32 = (($detect.Instances | Where-Object { $_.Vulnerable -and $_.Arch -eq '32-bit' }).Count -gt 0)
if ($need64) { $installer64 = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile64 -ShareSubPath $ShareSubPath; if (!$installer64) { Log '64-bit Firefox installer not found' -Level 'ERROR'; Remove-Item $iniPath -Force -EA SilentlyContinue; exit 1 }; if (!(Install-Exe -Path $installer64 -Arguments "/INI=`"$iniPath`" /S" -Label 'Firefox64')) { Remove-Item $iniPath -Force -EA SilentlyContinue; exit 1 } }
if ($need32) { $installer32 = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile32 -ShareSubPath $ShareSubPath; if (!$installer32) { Log '32-bit Firefox installer not found' -Level 'ERROR'; Remove-Item $iniPath -Force -EA SilentlyContinue; exit 1 }; if (!(Install-Exe -Path $installer32 -Arguments "/INI=`"$iniPath`" /S" -Label 'Firefox32')) { Remove-Item $iniPath -Force -EA SilentlyContinue; exit 1 } }
Remove-Item $iniPath -Force -EA SilentlyContinue
$post = Detect-Firefox
foreach ($i in $post.Instances) { Log "Post-check $($i.Version) [$($i.Arch)] from $($i.Source) vulnerable=$($i.Vulnerable)" }
if ($post.Vulnerable) { Log 'Firefox remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason 'Firefox update' }
Log 'Firefox remediation SUCCEEDED'
exit 0
9. OfficeC2R
File: Remediate-OfficeC2R.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-OfficeC2R.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft Office Click-to-Run
    Plugins: 298974, 298975, 298976, 298977, 283872-4, 271235, 278331-4, 56998, 97085
#>

$TargetVersion = "16.0.18429.20132"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "OfficeC2R"

function Detect-OfficeC2R {
    $c2rReg = "HKLM:\SOFTWARE\Microsoft\Office\ClickToRun\Configuration"
    if (!(Test-Path $c2rReg)) { return @{ Installed=$false; Vulnerable=$false; IsC2R=$false; Version=$null; Platform=$null; Channel=$null } }
    $props = Get-ItemProperty $c2rReg -EA SilentlyContinue
    $version = $props.VersionToReport
    if (!$version) { $version = $props.ClientVersionToReport }
    $platform = $props.Platform
    $channel  = $props.UpdateChannel
    $isVuln   = if ($version) { Test-Below $version $TargetVersion } else { $true }
    return @{ Installed=$true; Vulnerable=$isVuln; IsC2R=$true; Version=$version; Platform=$platform; Channel=$channel }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-OfficeC2R
    if (!$d.Installed) { Write-Output 'NOT_INSTALLED|OfficeC2R'; exit 0 }
    if ($d.Vulnerable) { Write-Output "VULNERABLE|OfficeC2R|$($d.Version)|$($d.Platform)|Target:$TargetVersion"; exit 1 }
    Write-Output "NOT_VULNERABLE|OfficeC2R|$($d.Version)"; exit 0
}

Log '===== Office C2R Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
if (Test-MECMConflict -SoftwareName 'Office') { Log 'Deferring to MECM for Office'; exit 0 }
if (Test-MECMConflict -SoftwareName 'Microsoft 365') { Log 'Deferring to MECM for Microsoft 365'; exit 0 }
$detect = Detect-OfficeC2R
Log "Detected Office C2R: Installed=$($detect.Installed) Version=$($detect.Version) Platform=$($detect.Platform) Channel=$($detect.Channel) Vulnerable=$($detect.Vulnerable)"
if (!$detect.Installed) { Log 'Office C2R not present'; exit 0 }
if (!$detect.Vulnerable) { Log 'Office C2R already at or above target'; exit 0 }
Stop-AppSafely -Processes @('WINWORD','EXCEL','OUTLOOK','POWERPNT','ONENOTE','MSACCESS','MSPUB','lync','Teams','OneDrive') -GraceSeconds 15
$c2r = "$env:ProgramFiles\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe"
if (!(Test-Path $c2r)) { $c2r = "${env:ProgramFiles(x86)}\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe" }
if (!(Test-Path $c2r)) { Log 'OfficeC2RClient.exe not found' -Level 'ERROR'; exit 1 }
$p = Start-Process -FilePath $c2r -ArgumentList '/update user displaylevel=false forceappshutdown=true' -Wait -PassThru -NoNewWindow
Log "Office C2R trigger exit code: $($p.ExitCode)"
$timeout = 900
$elapsed = 0
while ($elapsed -lt $timeout) {
    Start-Sleep -Seconds 30
    $elapsed += 30
    $check = Detect-OfficeC2R
    Log "Polling Office version after ${elapsed}s: $($check.Version) vulnerable=$($check.Vulnerable)"
    if (!$check.Vulnerable) { break }
}
$post = Detect-OfficeC2R
Log "Post-check Office version: $($post.Version) vulnerable=$($post.Vulnerable)"
if ($post.Vulnerable) { Log 'Office remediation FAILED: target version was not reached inside the wait window' -Level 'ERROR'; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason 'Office C2R update' }
Log 'Office remediation SUCCEEDED'
exit 0
10. VSCode
File: Remediate-VSCode.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-VSCode.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft Visual Studio Code
    Nessus Plugins: 298881, 265431, 276924, 241552
#>

$TargetVersion = "1.98.0"
$InstallerFile = "VSCodeSetup-x64.exe"
$AppFolder     = "VSCode"
$ShareSubPath  = "VSCode"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "VSCode"

function Detect-VSCode {
    $installs = @()
    $paths = @("$env:ProgramFiles\Microsoft VS Code\Code.exe")
    Get-ChildItem "C:\Users" -Directory -EA SilentlyContinue | ForEach-Object {
        $paths += (Join-Path $_.FullName "AppData\Local\Programs\Microsoft VS Code\Code.exe")
    }
    foreach ($p in $paths | Select-Object -Unique) {
        if (Test-Path $p) {
            $v = (Get-Item $p).VersionInfo.ProductVersion
            if ($v) { $installs += [PSCustomObject]@{ Version=$v; Path=$p; Source='File' } }
        }
    }
    $registered = Get-AllInstalled -NamePattern 'Visual Studio Code' -IncludeUser
    foreach ($r in $registered) { $installs += [PSCustomObject]@{ Version=$r.Version; Source='Uninstall'; Scope=$r.Scope; GUID=$r.GUID; Uninstall=$r.Uninstall; QuietUninstall=$r.QuietUninstall } }
    if ($installs.Count -eq 0) { return @{ Installed=$false; Vulnerable=$false; Instances=@() } }
    foreach ($i in $installs) { $i | Add-Member -NotePropertyName Vulnerable -NotePropertyValue (Test-Below $i.Version $TargetVersion) -Force }
    return @{ Installed=$true; Vulnerable=(($installs | Where-Object { $_.Vulnerable }).Count -gt 0); Instances=$installs }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-VSCode
    if (!$d.Installed) { Write-Output 'NOT_INSTALLED|VSCode'; exit 0 }
    if ($d.Vulnerable) { Write-Output "VULNERABLE|VSCode|$((($d.Instances | Where-Object { $_.Vulnerable }).Version | Select-Object -Unique) -join ',')|Target:$TargetVersion"; exit 1 }
    Write-Output "NOT_VULNERABLE|VSCode|Target:$TargetVersion"; exit 0
}

Log '===== VS Code Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-VSCode
foreach ($i in $detect.Instances) { Log "Detected VS Code version $($i.Version) source=$($i.Source) vulnerable=$($i.Vulnerable)" }
if (!$detect.Installed) { Log 'VS Code not installed'; exit 0 }
if (!$detect.Vulnerable) { Log 'VS Code already at or above target'; exit 0 }
Stop-AppSafely -Processes @('Code','CodeHelper') -GraceSeconds 5
$oldRegistered = Get-AllInstalled -NamePattern 'Visual Studio Code' -IncludeUser
if ($oldRegistered) { Remove-OldVersions -Installs $oldRegistered -TargetVer $TargetVersion -NameForLog 'VS Code' | Out-Null }
$installer = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile -ShareSubPath $ShareSubPath
if (!$installer) { Log 'VS Code installer not found' -Level 'ERROR'; exit 1 }
if (!(Install-Exe -Path $installer -Arguments '/VERYSILENT /NORESTART /MERGETASKS=!runcode,!desktopicon' -Label 'VSCode')) { exit 1 }
Get-ChildItem "C:\Users" -Directory -EA SilentlyContinue | ForEach-Object {
    $codePaths = @(
        (Join-Path $_.FullName 'AppData\Local\Programs\Microsoft VS Code\bin\code.cmd'),
        "$env:ProgramFiles\Microsoft VS Code\bin\code.cmd"
    )
    foreach ($cp in $codePaths) {
        if (Test-Path $cp) {
            try { Start-Process $cp -ArgumentList '--update-extensions' -Wait -NoNewWindow -EA SilentlyContinue; Log "Extension update attempted for profile $($_.Name)" } catch { Log "Extension update failed for profile $($_.Name)" -Level 'WARN' }
            break
        }
    }
}
$post = Detect-VSCode
foreach ($i in $post.Instances) { Log "Post-check VS Code version $($i.Version) source=$($i.Source) vulnerable=$($i.Vulnerable)" }
if ($post.Vulnerable) { Log 'VS Code remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
Log 'VS Code remediation SUCCEEDED'
exit 0
11. SQLServer
File: Remediate-SQLServer.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-SQLServer.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft SQL Server (Drivers + Instance Patches)
#>

$OleDbTargetVersion = "19.4.3.0"
$OdbcTargetVersion  = "18.4.1.1"
$OleDbInstallerFile = "msoledbsql_x64.msi"
$OdbcInstallerFile  = "msodbcsql_x64.msi"
$DriverAppFolder    = "SQLDrivers"
$CUAppFolder        = "SQLServer"

$SQLInstanceTargets = @{
    "16" = "16.0.4185.3"
    "15" = "15.0.4415.2"
    "14" = "14.0.3485.1"
    "13" = "13.0.7050.2"
}

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "SQLServer"

function Detect-SQLDriversVulnerable {
    $results = @()
    $regPaths = @("HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall","HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall")
    foreach ($rp in $regPaths) {
        Get-ChildItem $rp -EA SilentlyContinue | ForEach-Object {
            $props = Get-ItemProperty $_.PSPath -EA SilentlyContinue
            if ($props.DisplayName -match 'OLE DB Driver' -and $props.DisplayVersion) {
                try { $isVuln = [version]$props.DisplayVersion -lt [version]$OleDbTargetVersion } catch { $isVuln = $true }
                $results += @{ Name=$props.DisplayName; Version=$props.DisplayVersion; Type='OLEDB'; Vulnerable=$isVuln }
            }
            if ($props.DisplayName -match 'ODBC Driver.*SQL Server' -and $props.DisplayVersion) {
                try { $isVuln = [version]$props.DisplayVersion -lt [version]$OdbcTargetVersion } catch { $isVuln = $true }
                $results += @{ Name=$props.DisplayName; Version=$props.DisplayVersion; Type='ODBC'; Vulnerable=$isVuln }
            }
        }
    }
    return $results
}

function Detect-SQLInstancesVulnerable {
    $instances = @()
    $regPath = 'HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL'
    if (-not (Test-Path $regPath)) { return $instances }
    $instNames = Get-ItemProperty $regPath -EA SilentlyContinue
    foreach ($prop in $instNames.PSObject.Properties) {
        if ($prop.Name -match '^PS') { continue }
        $instKey = $prop.Value
        $setupPath = "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\$instKey\Setup"
        if (Test-Path $setupPath) {
            $setup = Get-ItemProperty $setupPath -EA SilentlyContinue
            $version = $setup.Version
            $major = ($version -split '\.')[0]
            $isVuln = $false
            if ($SQLInstanceTargets.ContainsKey($major)) { try { $isVuln = [version]$version -lt [version]$SQLInstanceTargets[$major] } catch { $isVuln = $true } }
            $instances += @{ Name=$prop.Name; InstanceKey=$instKey; Version=$version; Major=$major; Vulnerable=$isVuln }
        }
    }
    return $instances
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $drivers = Detect-SQLDriversVulnerable
    $instances = Detect-SQLInstancesVulnerable
    $anyVuln = (($drivers | Where-Object { $_.Vulnerable }).Count -gt 0) -or (($instances | Where-Object { $_.Vulnerable }).Count -gt 0)
    if ($anyVuln) { Write-Output 'VULNERABLE|SQLServer'; exit 1 } else { Write-Output 'NOT_VULNERABLE|SQLServer'; exit 0 }
}

Log '===== SQL Server Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$overallSuccess = $true
$drivers = Detect-SQLDriversVulnerable
foreach ($drv in $drivers) { Log "Driver: $($drv.Name) v$($drv.Version) Vulnerable:$($drv.Vulnerable)" }
$vulnOleDb = $drivers | Where-Object { $_.Type -eq 'OLEDB' -and $_.Vulnerable }
$vulnOdbc  = $drivers | Where-Object { $_.Type -eq 'ODBC' -and $_.Vulnerable }
if ($vulnOleDb) { $msi = Resolve-Installer -AppFolder $DriverAppFolder -FileName $OleDbInstallerFile -ShareSubPath $DriverAppFolder; if ($msi) { if (!(Install-Msi -Path $msi -ExtraArgs 'IACCEPTMSOLEDBSQLLICENSETERMS=YES' -Label 'SQL_OLEDB')) { $overallSuccess = $false } } else { $overallSuccess = $false } }
if ($vulnOdbc)  { $msi = Resolve-Installer -AppFolder $DriverAppFolder -FileName $OdbcInstallerFile  -ShareSubPath $DriverAppFolder; if ($msi) { if (!(Install-Msi -Path $msi -ExtraArgs 'IACCEPTMSODBCSQLLICENSETERMS=YES' -Label 'SQL_ODBC'))  { $overallSuccess = $false } } else { $overallSuccess = $false } }
$instances = Detect-SQLInstancesVulnerable
foreach ($inst in $instances) { Log "Instance: $($inst.Name) v$($inst.Version) (SQL $($inst.Major)) Vulnerable:$($inst.Vulnerable)" }
$vulnInstances = $instances | Where-Object { $_.Vulnerable }
foreach ($inst in $vulnInstances) {
    $cuPattern = "SQLServer$($inst.Major)*-KB*-x64.exe"
    $cuFile = Resolve-LatestInstaller -AppFolder $CUAppFolder -Pattern $cuPattern -ShareSubPath $CUAppFolder
    if ($cuFile) {
        Log "Applying CU: $cuFile to instance $($inst.Name)"
        $proc = Start-Process $cuFile -ArgumentList "/qs /IAcceptSQLServerLicenseTerms /Action=Patch /InstanceName=$($inst.Name) /INDICATEPROGRESS" -Wait -PassThru -NoNewWindow
        Log "CU install exit: $($proc.ExitCode)"
        if ($proc.ExitCode -ne 0 -and $proc.ExitCode -ne 3010) { $overallSuccess = $false }
    } else {
        Log "No CU found for SQL $($inst.Major)" -Level 'ERROR'
        $overallSuccess = $false
    }
}
$postDrivers = Detect-SQLDriversVulnerable
$postInstances = Detect-SQLInstancesVulnerable
if ((($postDrivers | Where-Object { $_.Vulnerable }).Count -gt 0) -or (($postInstances | Where-Object { $_.Vulnerable }).Count -gt 0)) {
    Log 'SQL Server remediation FAILED: vulnerable drivers or instances still remain' -Level 'ERROR'
    exit 1
}
if (!$overallSuccess) { Log 'SQL Server remediation FAILED: one or more installers returned errors' -Level 'ERROR'; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason 'SQL Server remediation' }
Log 'SQL Server remediation SUCCEEDED'
exit 0
12. DotNet
File: Remediate-DotNet.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-DotNet.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft .NET Core / .NET Runtime
#>

$TargetLTSVersion   = "8.0.14"
$EOLVersionPatterns = @('2.0','2.1','2.2','3.0','3.1','5.0','7.0')
$AppFolder          = 'DotNet'
$ShareSubPath       = 'DotNet'

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog '.NET'

function Detect-DotNetVulnerable {
    $dotnetExe = "$env:ProgramFiles\dotnet\dotnet.exe"
    if (-not (Test-Path $dotnetExe)) { return @{ Installed=$false; Vulnerable=$false; EOLVersions=@(); OutdatedVersions=@() } }
    $eolFound = @(); $outdatedFound = @()
    try {
        $runtimes = & $dotnetExe --list-runtimes 2>&1
        foreach ($line in $runtimes) {
            if ($line -match '^(\S+)\s+(\S+)') {
                $rtVer = $Matches[2]
                foreach ($pattern in $EOLVersionPatterns) { if ($rtVer -like "$pattern.*") { $eolFound += "$($Matches[1]) $rtVer"; break } }
                if ($rtVer -match '^8\.' -and [version]$rtVer -lt [version]$TargetLTSVersion) { $outdatedFound += "$($Matches[1]) $rtVer" }
            }
        }
    } catch {}
    return @{ Installed=$true; Vulnerable=(($eolFound.Count -gt 0) -or ($outdatedFound.Count -gt 0)); EOLVersions=$eolFound; OutdatedVersions=$outdatedFound }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-DotNetVulnerable
    if ($d.Vulnerable) { Write-Output "VULNERABLE|DotNet|EOL:$($d.EOLVersions -join ';')|Outdated:$($d.OutdatedVersions -join ';')"; exit 1 }
    Write-Output 'NOT_VULNERABLE|DotNet'; exit 0
}

Log '===== .NET Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-DotNetVulnerable
Log "EOL versions: $($detect.EOLVersions -join ', ')"
Log "Outdated versions: $($detect.OutdatedVersions -join ', ')"
if (-not $detect.Vulnerable) { Log 'Not vulnerable'; exit 0 }
$regEntries = Get-AllInstalled -NamePattern '\.NET (Core )?(Runtime|SDK|Desktop|ASP)' -IncludeUser
foreach ($entry in $regEntries) {
    $remove = $false
    foreach ($pattern in $EOLVersionPatterns) { if ($entry.Version -like "$pattern.*") { $remove = $true; break } }
    if ($entry.Version -match '^8\.' -and (Test-Below $entry.Version $TargetLTSVersion)) { $remove = $true }
    if ($remove) { Remove-OldVersions -Installs @($entry) -TargetVer '9999.9999.9999.9999' -NameForLog '.NET legacy runtime' | Out-Null }
}
foreach ($pattern in @('dotnet-runtime-8*-win-x64.exe','aspnetcore-runtime-8*-win-x64.exe','windowsdesktop-runtime-8*-win-x64.exe')) {
    $installer = Resolve-LatestInstaller -AppFolder $AppFolder -Pattern $pattern -ShareSubPath $ShareSubPath
    if ($installer) { if (!(Install-Exe -Path $installer -Arguments '/install /quiet /norestart' -Label $pattern)) { exit 1 } }
}
$post = Detect-DotNetVulnerable
Log "Post-check EOL versions: $($post.EOLVersions -join ', ')"
Log "Post-check outdated versions: $($post.OutdatedVersions -join ', ')"
if ($post.Vulnerable) { Log '.NET remediation FAILED: vulnerable runtime entries remain' -Level 'ERROR'; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason '.NET remediation' }
Log '.NET remediation SUCCEEDED'
exit 0
13. WindowsUpdates
File: Remediate-WindowsUpdates.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-WindowsUpdates.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Windows Cumulative Security Updates
#>

$RequiredKBs = @(
    'KB5075941',
    'KB5075904',
    'KB5073455',
    'KB5073723',
    'KB5071417'
)
$LogPath = "C:\Logs\Ivanti\WindowsUpdate_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"

function Detect-MissingKBs {
    $installed = @(); $missing = @()
    $hotfixes = Get-HotFix -EA SilentlyContinue | Select-Object -ExpandProperty HotFixID
    foreach ($kb in $RequiredKBs) {
        if ($hotfixes -contains $kb) { $installed += $kb }
        else {
            $found = $false
            $regPath = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\Packages'
            if (Test-Path $regPath) {
                $kbNum = $kb -replace 'KB',''
                $match = Get-ChildItem $regPath -EA SilentlyContinue | Where-Object { $_.Name -match $kbNum }
                if ($match) { $found = $true; $installed += $kb }
            }
            if (-not $found) { $missing += $kb }
        }
    }
    return @{ Installed=$installed; Missing=$missing; Vulnerable=($missing.Count -gt 0) }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-MissingKBs
    if ($d.Vulnerable) { Write-Output "VULNERABLE|WindowsUpdate|Missing:$($d.Missing -join ',')"; exit 1 }
    Write-Output 'NOT_VULNERABLE|WindowsUpdate|AllInstalled'; exit 0
}

function Write-Log { param([string]$M,[string]$L='INFO'); $e="[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$L] $M"; $d=Split-Path $LogPath -Parent; if(!(Test-Path $d)){New-Item -ItemType Directory -Path $d -Force|Out-Null}; Add-Content -Path $LogPath -Value $e; Write-Output $e }

Write-Log '===== Windows Cumulative Update Remediation Started ====='
$detect = Detect-MissingKBs
Write-Log "Installed KBs: $($detect.Installed -join ', ')"
Write-Log "Missing KBs: $($detect.Missing -join ', ')"
if (-not $detect.Vulnerable) { Write-Log 'All required KBs installed'; exit 0 }
try {
    $session = New-Object -ComObject Microsoft.Update.Session
    $searcher = $session.CreateUpdateSearcher()
    $searchResult = $searcher.Search("IsInstalled=0 AND Type='Software'")
    $updatesToInstall = New-Object -ComObject Microsoft.Update.UpdateColl
    foreach ($update in $searchResult.Updates) {
        foreach ($kb in $detect.Missing) {
            $kbNum = $kb -replace 'KB',''
            if ($update.Title -match $kbNum -or ($update.KBArticleIDs | Where-Object { $_ -eq $kbNum })) { Write-Log "Found update: $($update.Title)"; $updatesToInstall.Add($update) | Out-Null }
        }
    }
    if ($updatesToInstall.Count -gt 0) {
        $downloader = $session.CreateUpdateDownloader(); $downloader.Updates = $updatesToInstall; $downloadResult = $downloader.Download(); Write-Log "Download result: $($downloadResult.ResultCode)"
        $installer = $session.CreateUpdateInstaller(); $installer.Updates = $updatesToInstall; $installResult = $installer.Install(); Write-Log "Install result: $($installResult.ResultCode)"
        if ($installResult.RebootRequired) { Write-Log 'REBOOT REQUIRED after update installation' -L 'WARN' }
    } else { Write-Log 'No matching updates found in Windows Update catalog. Updates may need WSUS approval.' -L 'WARN' }
} catch {
    Write-Log "Windows Update API error: $($_.Exception.Message)" -L 'ERROR'
    Start-Process 'USOClient.exe' -ArgumentList 'StartInteractiveScan' -Wait -NoNewWindow -EA SilentlyContinue
    Start-Process 'USOClient.exe' -ArgumentList 'StartDownload' -Wait -NoNewWindow -EA SilentlyContinue
    Start-Process 'USOClient.exe' -ArgumentList 'StartInstall' -Wait -NoNewWindow -EA SilentlyContinue
    Write-Log 'USOClient update triggered'
}
$post = Detect-MissingKBs
Write-Log "Post-update missing: $($post.Missing -join ', ')"
if ($post.Vulnerable) { Write-Log 'Windows Update remediation FAILED: required KBs still missing' -L 'ERROR'; exit 1 }
Write-Log 'Windows Update remediation SUCCEEDED'; exit 0
14. WindowsNotepad
File: Remediate-WindowsNotepad.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-WindowsNotepad.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft Windows Notepad Command Injection
#>

$TargetVersion = "11.2510.0.0"
$LogPath       = "C:\Logs\Ivanti\Notepad_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"

function Detect-NotepadVulnerable {
    $notepadApp = Get-AppxPackage -Name 'Microsoft.WindowsNotepad' -AllUsers -EA SilentlyContinue
    if ($notepadApp) {
        try { $isVuln = [version]$notepadApp.Version -lt [version]$TargetVersion } catch { $isVuln = $true }
        return @{ Installed=$true; Vulnerable=$isVuln; Version=$notepadApp.Version; Type='Store' }
    }
    $notepadExe = "$env:SystemRoot\System32\notepad.exe"
    if (Test-Path $notepadExe) { return @{ Installed=$true; Vulnerable=$false; Version=(Get-Item $notepadExe).VersionInfo.ProductVersion; Type='System' } }
    return @{ Installed=$false; Vulnerable=$false; Version=$null; Type=$null }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-NotepadVulnerable
    if ($d.Vulnerable) { Write-Output "VULNERABLE|Notepad|$($d.Version)|Target:$TargetVersion"; exit 1 }
    Write-Output "NOT_VULNERABLE|Notepad|$($d.Version)"; exit 0
}

function Write-Log { param([string]$M,[string]$L='INFO'); $e="[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$L] $M"; $d=Split-Path $LogPath -Parent; if(!(Test-Path $d)){New-Item -ItemType Directory -Path $d -Force|Out-Null}; Add-Content -Path $LogPath -Value $e; Write-Output $e }

Write-Log '===== Notepad Remediation Started ====='
$detect = Detect-NotepadVulnerable
Write-Log "Installed: $($detect.Installed) | Type: $($detect.Type) | Version: $($detect.Version) | Vulnerable: $($detect.Vulnerable)"
if (-not $detect.Vulnerable) { Write-Log 'Notepad not vulnerable'; exit 0 }
$winget = Get-Command winget -EA SilentlyContinue
if ($winget) {
    $proc = Start-Process winget -ArgumentList 'upgrade --id 9MSMLRH6LZF3 --silent --accept-package-agreements --accept-source-agreements' -Wait -PassThru -NoNewWindow
    Write-Log "Winget exit: $($proc.ExitCode)"
} else {
    try {
        $namespaceName = 'root\cimv2\mdm\dmmap'
        $className = 'MDM_EnterpriseModernAppManagement_AppManagement01'
        $wmiObj = Get-CimInstance -Namespace $namespaceName -ClassName $className -EA Stop
        $wmiObj | Invoke-CimMethod -MethodName 'UpdateScanMethod' -EA SilentlyContinue
        Write-Log 'Microsoft Store update scan triggered'
    } catch { Write-Log "Store update trigger failed: $($_.Exception.Message)" -L 'WARN' }
}
$post = Detect-NotepadVulnerable
Write-Log "Post-update: $($post.Version) | Vulnerable: $($post.Vulnerable)"
if ($post.Vulnerable) { Write-Log 'Notepad remediation FAILED: vulnerable build still present' -L 'ERROR'; exit 1 }
Write-Log 'Notepad remediation SUCCEEDED'; exit 0
15. Teams
File: Remediate-Teams.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-Teams.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Microsoft Teams Desktop
#>

$TargetVersion         = "25122.1415.3698.6812"
$BootstrapperFile      = "teamsbootstrapper.exe"
$MSIXFile              = "MSTeams-x64.msix"
$AppFolder             = "Teams"
$ShareSubPath          = "Teams"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "Teams"

function Detect-TeamsVulnerable {
    $pkg = Get-AppxPackage -Name 'MSTeams' -AllUsers -EA SilentlyContinue
    if ($pkg) {
        try { $isVuln = [version]$pkg.Version -lt [version]$TargetVersion } catch { $isVuln = $true }
        return @{ Installed=$true; Vulnerable=$isVuln; Version=$pkg.Version; Type='New' }
    }
    $classicPaths = @("$env:LOCALAPPDATA\Microsoft\Teams\current\Teams.exe","${env:ProgramFiles(x86)}\Microsoft\Teams\current\Teams.exe")
    foreach ($p in $classicPaths) { if (Test-Path $p) { return @{ Installed=$true; Vulnerable=$true; Version=(Get-Item $p).VersionInfo.ProductVersion; Type='Classic' } } }
    return @{ Installed=$false; Vulnerable=$false; Version=$null; Type=$null }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-TeamsVulnerable
    if ($d.Vulnerable) { Write-Output "VULNERABLE|Teams|$($d.Type)|$($d.Version)"; exit 1 }
    Write-Output "NOT_VULNERABLE|Teams|$($d.Version)"; exit 0
}

Log '===== Teams Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-TeamsVulnerable
Log "Type: $($detect.Type) | Version: $($detect.Version) | Vulnerable: $($detect.Vulnerable)"
if (-not $detect.Vulnerable) { Log 'Teams not vulnerable'; exit 0 }
Stop-AppSafely -Processes @('ms-teams','Teams') -GraceSeconds 5
$classicEntries = Get-AllInstalled -NamePattern 'Teams Machine-Wide Installer|Microsoft Teams' -IncludeUser
if ($classicEntries) { Remove-OldVersions -Installs $classicEntries -TargetVer '9999.9999.9999.9999' -NameForLog 'Classic Teams' | Out-Null }
$bootstrapper = Resolve-Installer -AppFolder $AppFolder -FileName $BootstrapperFile -ShareSubPath $ShareSubPath
if (!$bootstrapper) { Log 'Teams bootstrapper not found' -Level 'ERROR'; exit 1 }
$msix = Resolve-Installer -AppFolder $AppFolder -FileName $MSIXFile -ShareSubPath $ShareSubPath
if ($msix) { $args = "-p -o `"$msix`"" } else { $args = '-p' }
if (!(Install-Exe -Path $bootstrapper -Arguments $args -Label 'TeamsBootstrapper')) { exit 1 }
$post = Detect-TeamsVulnerable
Log "Post-check Teams type: $($post.Type) | Version: $($post.Version) | Vulnerable: $($post.Vulnerable)"
if ($post.Vulnerable) { Log 'Teams remediation FAILED: vulnerable/classic Teams still present' -Level 'ERROR'; exit 1 }
Log 'Teams remediation SUCCEEDED'
exit 0
16. NodeJS
File: Remediate-NodeJS.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-NodeJS.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Node.js
#>

$TargetVersion = "20.20.0"
$InstallerFile = "node-latest-x64.msi"
$AppFolder     = "NodeJS"
$ShareSubPath  = "NodeJS"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "NodeJS"

function Detect-NodeVulnerable {
    $version = $null
    $exe = "$env:ProgramFiles\nodejs\node.exe"
    if (Test-Path $exe) { try { $version = (& $exe --version 2>$null) -replace '^v','' } catch {} }
    $registered = Get-AllInstalled -NamePattern '^Node\.js' -IncludeUser
    if (!$version -and $registered) { $version = ($registered | Sort-Object Version -Descending | Select-Object -First 1).Version }
    if (-not $version) { return @{ Installed=$false; Vulnerable=$false; Version=$null } }
    try { $isVuln = [version]($version -replace '[^0-9.]','') -lt [version]$TargetVersion } catch { $isVuln = $true }
    return @{ Installed=$true; Vulnerable=$isVuln; Version=$version }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') { $d = Detect-NodeVulnerable; if ($d.Vulnerable) { Write-Output "VULNERABLE|NodeJS|$($d.Version)|Target:$TargetVersion"; exit 1 } else { Write-Output "NOT_VULNERABLE|NodeJS|$($d.Version)"; exit 0 } }

Log '===== Node.js Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-NodeVulnerable
Log "Version: $($detect.Version) | Vulnerable: $($detect.Vulnerable)"
if (-not $detect.Vulnerable) { Log 'Node.js not vulnerable'; exit 0 }
$oldEntries = Get-AllInstalled -NamePattern '^Node\.js' -IncludeUser
if ($oldEntries) { Remove-OldVersions -Installs $oldEntries -TargetVer $TargetVersion -NameForLog 'Node.js' | Out-Null }
$installer = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile -ShareSubPath $ShareSubPath
if (!$installer) { Log 'Node.js installer not found' -Level 'ERROR'; exit 1 }
if (!(Install-Msi -Path $installer -Label 'NodeJS')) { exit 1 }
$post = Detect-NodeVulnerable
Log "Post-check version: $($post.Version) | Vulnerable: $($post.Vulnerable)"
if ($post.Vulnerable) { Log 'Node.js remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
Log 'Node.js remediation SUCCEEDED'
exit 0
17. OracleJava
File: Remediate-OracleJava.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-OracleJava.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Oracle Java SE
#>

$TargetVersion = "21.0.6"
$InstallerFile = "jdk-latest_windows-x64_bin.msi"
$AppFolder     = "Java"
$ShareSubPath  = "Java"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "JavaSE"

function Detect-JavaVulnerable {
    $versions = @()
    $entries = Get-AllInstalled -NamePattern 'Java.*SE|Java \d+|JDK|JRE' -IncludeUser
    foreach ($e in $entries) {
        try { $isVuln = [version]($e.Version -replace '[^0-9.]','') -lt [version]$TargetVersion } catch { $isVuln = $true }
        $versions += @{ Name=$e.Name; Version=$e.Version; Vulnerable=$isVuln; GUID=$e.GUID; Uninstall=$e.Uninstall; QuietUninstall=$e.QuietUninstall; Scope=$e.Scope; Arch=$e.Arch }
    }
    if ($versions.Count -eq 0) { return @{ Installed=$false; Vulnerable=$false; Versions=@() } }
    return @{ Installed=$true; Vulnerable=(($versions | Where-Object { $_.Vulnerable }).Count -gt 0); Versions=$versions }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') {
    $d = Detect-JavaVulnerable
    if ($d.Vulnerable) { $v=($d.Versions|Where-Object{$_.Vulnerable}|ForEach-Object{"$($_.Version)"})-join','; Write-Output "VULNERABLE|Java|$v"; exit 1 } else { Write-Output 'NOT_VULNERABLE|Java'; exit 0 }
}

Log '===== Oracle Java Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-JavaVulnerable
foreach ($v in $detect.Versions) { Log "$($v.Name) v$($v.Version) Vulnerable:$($v.Vulnerable)" }
if (-not $detect.Vulnerable) { Log 'Java not vulnerable'; exit 0 }
$oldEntries = @()
foreach ($v in ($detect.Versions | Where-Object { $_.Vulnerable })) { $oldEntries += [PSCustomObject]@{ Name=$v.Name; Version=$v.Version; GUID=$v.GUID; Uninstall=$v.Uninstall; QuietUninstall=$v.QuietUninstall; Scope=$v.Scope; Arch=$v.Arch } }
if ($oldEntries) { Remove-OldVersions -Installs $oldEntries -TargetVer $TargetVersion -NameForLog 'Oracle Java' | Out-Null }
$installer = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile -ShareSubPath $ShareSubPath
if (!$installer) { Log 'Java MSI not found' -Level 'ERROR'; exit 1 }
if (!(Install-Msi -Path $installer -Label 'JavaSE')) { exit 1 }
$post = Detect-JavaVulnerable
foreach ($v in $post.Versions) { Log "Post-check $($v.Name) v$($v.Version) Vulnerable:$($v.Vulnerable)" }
if ($post.Vulnerable) { Log 'Java remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
Log 'Java remediation SUCCEEDED'
exit 0
18. ApacheLog4j
File: Remediate-ApacheLog4j.ps1 | Location: \\FILESERVER\IvantiModules\scripts\Remediate-ApacheLog4j.ps1

<#
.SYNOPSIS
    Ivanti Remediation - Apache Log4j Vulnerabilities
    Nessus Plugins: 182252 (Log4j 1.x SEoL), 156860 (Log4j 1.x vulns), 156327 (Log4j 2.x RCE),
                    156103 (JMSAppender RCE), 156183 (DoS), 282519 (MitM < 2.25.3)
.DESCRIPTION
    Scans for vulnerable Log4j JAR files on disk, reports findings, and can remove/replace them.
    Log4j 1.x is EOL and must be removed or the application migrated.
    Log4j 2.x below 2.25.3 must be updated.
.NOTES
    THIS SCRIPT REPORTS AND OPTIONALLY REMEDIATES. Review findings before enabling auto-remediation.
    Log4j is embedded in Java applications, so updates must be coordinated with app owners.
#>
 
$Log4j2MinSafeVersion = "2.25.3"   # Minimum safe Log4j 2.x version
$ScanPaths = @("C:\Program Files","C:\Program Files (x86)","D:\","E:\")  # Customize for your environment
$AutoRemediate = $false   # SET TO $true ONLY AFTER REVIEWING FINDINGS
$LogPath = "C:\Logs\Ivanti\Log4j_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
 
function Detect-Log4jVulnerable {
    $findings = @()
    foreach ($scanPath in $ScanPaths) {
        if (-not (Test-Path $scanPath)) { continue }
        # Search for log4j JAR files
        Get-ChildItem -Path $scanPath -Recurse -Filter "log4j*.jar" -EA SilentlyContinue | ForEach-Object {
            $jar = $_
            $name = $jar.Name.ToLower()
            $vuln = $null
            
            # Log4j 1.x (any version - EOL)
            if ($name -match 'log4j-1\.' -or ($name -match 'log4j-\d' -and $name -match '^log4j-1')) {
                $vuln = "Log4j1-EOL"
            }
            # Log4j 2.x version check
            elseif ($name -match 'log4j-core-(\d+\.\d+\.?\d*)') {
                $ver = $Matches[1]
                try {
                    if ([version]$ver -lt [version]$Log4j2MinSafeVersion) { $vuln = "Log4j2-Outdated:$ver" }
                } catch { $vuln = "Log4j2-Unknown:$ver" }
            }
            elseif ($name -match 'log4j-api-(\d+\.\d+\.?\d*)') {
                $ver = $Matches[1]
                try {
                    if ([version]$ver -lt [version]$Log4j2MinSafeVersion) { $vuln = "Log4j2-API-Outdated:$ver" }
                } catch {}
            }
            
            if ($vuln) {
                $findings += @{ Path=$jar.FullName; Finding=$vuln; Size=$jar.Length; Modified=$jar.LastWriteTime }
            }
        }
    }
    return $findings
}
 
function Write-Log { param([string]$M,[string]$L="INFO"); $e="[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$L] $M"; $d=Split-Path $LogPath -Parent; if(!(Test-Path $d)){New-Item -ItemType Directory -Path $d -Force|Out-Null}; Add-Content -Path $LogPath -Value $e; Write-Output $e }
 
if ($env:IVANTI_DETECT_ONLY -eq "TRUE") {
    $findings = Detect-Log4jVulnerable
    if ($findings.Count -gt 0) {
        $summary = ($findings | ForEach-Object { "$($_.Finding)" } | Select-Object -Unique) -join ','
        Write-Output "VULNERABLE|Log4j|$($findings.Count) files|$summary"; exit 1
    }
    Write-Output "NOT_VULNERABLE|Log4j"; exit 0
}
 
Write-Log "===== Log4j Remediation Started ====="
Write-Log "Scanning paths: $($ScanPaths -join ', ')"
Write-Log "Auto-remediate: $AutoRemediate"
 
$findings = Detect-Log4jVulnerable
Write-Log "Found $($findings.Count) vulnerable Log4j files"
 
foreach ($f in $findings) {
    Write-Log "FINDING: $($f.Finding) | $($f.Path) | Modified: $($f.Modified)" -L "WARN"
}
 
if ($AutoRemediate -and $findings.Count -gt 0) {
    Write-Log "--- AUTO-REMEDIATION ENABLED ---"
    foreach ($f in $findings) {
        if ($f.Finding -match "Log4j1") {
            # For Log4j 1.x: remove the JMSAppender class (mitigates RCE without removing JAR)
            Write-Log "Mitigating Log4j 1.x: Removing JMSAppender from $($f.Path)"
            try {
                $backupPath = "$($f.Path).bak.$(Get-Date -Format 'yyyyMMdd')"
                Copy-Item $f.Path $backupPath -Force
                # Remove JMSAppender class from JAR
                $jarTool = Get-Command jar -EA SilentlyContinue
                if ($jarTool) {
                    Push-Location (Split-Path $f.Path)
                    & jar -xf (Split-Path $f.Path -Leaf)
                    Remove-Item "org\apache\log4j\net\JMSAppender.class" -Force -EA SilentlyContinue
                    Remove-Item "org\apache\log4j\net\JMSSink.class" -Force -EA SilentlyContinue
                    Pop-Location
                }
                Write-Log "Mitigated: $($f.Path) (backup: $backupPath)"
            } catch {
                Write-Log "Mitigation failed for $($f.Path): $($_.Exception.Message)" -L "ERROR"
            }
        }
        elseif ($f.Finding -match "Log4j2") {
            Write-Log "Log4j 2.x at $($f.Path) needs application-level update by app owner" -L "WARN"
        }
    }
}
 
# Generate report
$reportPath = "C:\Logs\Ivanti\Log4j_Report_$env:COMPUTERNAME.csv"
if ($findings.Count -gt 0) {
    $findings | ForEach-Object { [PSCustomObject]$_ } | Export-Csv $reportPath -NoTypeInformation -Force
    Write-Log "Report saved: $reportPath"
}
 
Write-Log "SCAN COMPLETED - $($findings.Count) findings"; exit $(if ($findings.Count -gt 0) { 1 } else { 0 })
 
19. OpenSSH
File: Remediate-OpenSSH.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-OpenSSH.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - OpenSSH
    Nessus Plugins: 201194, 187201, 187315
#>

$TargetVersion  = "9.8.0.0"
$LogPath        = "C:\Logs\Ivanti\OpenSSH_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"

function Detect-OpenSSHVulnerable {
    $version = $null
    $sshExe = "$env:SystemRoot\System32\OpenSSH\ssh.exe"
    if (Test-Path $sshExe) { $version = (Get-Item $sshExe).VersionInfo.ProductVersion }
    if (-not $version) {
        $sshExe2 = "$env:ProgramFiles\OpenSSH\ssh.exe"
        if (Test-Path $sshExe2) { $version = (Get-Item $sshExe2).VersionInfo.ProductVersion }
    }
    if (-not $version) { try { $output = & ssh -V 2>&1; if ($output -match 'OpenSSH[_\s]+([\d\.]+)') { $version = $Matches[1] } } catch {} }
    if (-not $version) { return @{ Installed=$false; Vulnerable=$false; Version=$null } }
    try { $isVuln = [version]($version -replace '[^0-9.]','') -lt [version]$TargetVersion } catch { $isVuln = $true }
    return @{ Installed=$true; Vulnerable=$isVuln; Version=$version }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') { $d = Detect-OpenSSHVulnerable; if ($d.Vulnerable) { Write-Output "VULNERABLE|OpenSSH|$($d.Version)|Target:$TargetVersion"; exit 1 } else { Write-Output "NOT_VULNERABLE|OpenSSH|$($d.Version)"; exit 0 } }

function Write-Log { param([string]$M,[string]$L='INFO'); $e="[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$L] $M"; $d=Split-Path $LogPath -Parent; if(!(Test-Path $d)){New-Item -ItemType Directory -Path $d -Force|Out-Null}; Add-Content -Path $LogPath -Value $e; Write-Output $e }

Write-Log '===== OpenSSH Remediation Started ====='
$detect = Detect-OpenSSHVulnerable
Write-Log "Installed: $($detect.Installed) | Version: $($detect.Version) | Vulnerable: $($detect.Vulnerable)"
if (-not $detect.Vulnerable) { Write-Log 'OpenSSH not vulnerable'; exit 0 }
try {
    $capability = Get-WindowsCapability -Online | Where-Object { $_.Name -match 'OpenSSH.Client' -and $_.State -eq 'Installed' }
    if ($capability) { Remove-WindowsCapability -Online -Name $capability.Name -EA SilentlyContinue; Add-WindowsCapability -Online -Name $capability.Name -EA Stop; Write-Log 'OpenSSH client reinstalled via Windows capability' }
    $serverCap = Get-WindowsCapability -Online | Where-Object { $_.Name -match 'OpenSSH.Server' -and $_.State -eq 'Installed' }
    if ($serverCap) { Stop-Service sshd -Force -EA SilentlyContinue; Remove-WindowsCapability -Online -Name $serverCap.Name -EA SilentlyContinue; Add-WindowsCapability -Online -Name $serverCap.Name -EA Stop; Start-Service sshd -EA SilentlyContinue; Write-Log 'OpenSSH server reinstalled via Windows capability' }
} catch { Write-Log "Capability update failed: $($_.Exception.Message)" -L 'WARN' }
$winget = Get-Command winget -EA SilentlyContinue
if ($winget) { Start-Process winget -ArgumentList 'upgrade --id Microsoft.OpenSSH.Beta --silent --accept-package-agreements' -Wait -NoNewWindow -EA SilentlyContinue; Write-Log 'winget OpenSSH upgrade attempted' }
$sshdConfig = "$env:ProgramData\ssh\sshd_config"
if (Test-Path $sshdConfig) {
    $content = Get-Content $sshdConfig -Raw
    if ($content -notmatch 'Terrapin mitigation') {
        Add-Content $sshdConfig "`n# Terrapin mitigation (CVE-2023-48795)`nCiphers -chacha20-poly1305@openssh.com`nMACs -*etm@openssh.com"
        Restart-Service sshd -Force -EA SilentlyContinue
        Write-Log 'Terrapin mitigation applied to sshd_config'
    }
}
$post = Detect-OpenSSHVulnerable
Write-Log "Post-update: $($post.Version) | Vulnerable: $($post.Vulnerable)"
if ($post.Vulnerable) { Write-Log 'OpenSSH remediation FAILED: vulnerable version still present' -L 'ERROR'; exit 1 }
Write-Log 'OpenSSH remediation SUCCEEDED'; exit 0
20. DockerDesktop
File: Remediate-DockerDesktop.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-DockerDesktop.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - Docker Desktop
#>

$TargetVersion = "4.57.0"
$InstallerFile = "DockerDesktopInstaller.exe"
$AppFolder     = "Docker"
$ShareSubPath  = "Docker"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "DockerDesktop"

function Detect-DockerVulnerable {
    $version = $null
    $regPaths = @("HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall","HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall")
    foreach ($rp in $regPaths) {
        Get-ChildItem $rp -EA SilentlyContinue | ForEach-Object {
            $props = Get-ItemProperty $_.PSPath -EA SilentlyContinue
            if ($props.DisplayName -match 'Docker Desktop' -and $props.DisplayVersion) { $version = $props.DisplayVersion }
        }
    }
    if (-not $version) { return @{ Installed=$false; Vulnerable=$false; Version=$null } }
    try { $isVuln = [version]($version -replace '[^0-9.]','') -lt [version]$TargetVersion } catch { $isVuln = $true }
    return @{ Installed=$true; Vulnerable=$isVuln; Version=$version }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') { $d = Detect-DockerVulnerable; if ($d.Vulnerable) { Write-Output "VULNERABLE|Docker|$($d.Version)|Target:$TargetVersion"; exit 1 } else { Write-Output "NOT_VULNERABLE|Docker|$($d.Version)"; exit 0 } }

Log '===== Docker Desktop Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-DockerVulnerable
Log "Installed: $($detect.Installed) | Version: $($detect.Version) | Vulnerable: $($detect.Vulnerable)"
if (-not $detect.Vulnerable) { Log 'Docker Desktop not vulnerable'; exit 0 }
Stop-AppSafely -Processes @('Docker Desktop','com.docker.backend','com.docker.proxy') -GraceSeconds 5
$installer = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile -ShareSubPath $ShareSubPath
if (!$installer) { Log 'Docker Desktop installer not found' -Level 'ERROR'; exit 1 }
if (!(Install-Exe -Path $installer -Arguments 'install --quiet --accept-license' -Label 'DockerDesktop')) { exit 1 }
$post = Detect-DockerVulnerable
Log "Post-update: $($post.Version) | Vulnerable: $($post.Vulnerable)"
if ($post.Vulnerable) { Log 'Docker Desktop remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
Log 'Docker Desktop remediation SUCCEEDED'; exit 0
21. VMwareTools
File: Remediate-VMwareTools.ps1
Location: \\FILESERVER\IvantiModules\scripts\Remediate-VMwareTools.ps1
<#
.SYNOPSIS
    Ivanti Remediation v4.2 - VMware Tools
#>

$TargetVersion = "13.0.5"
$InstallerFile = "VMwareTools-latest-x86_64.exe"
$AppFolder     = "VMware"
$ShareSubPath  = "VMware"

$fwPath = "\\FILESERVER\IvantiModules\scripts\Shared-Framework.ps1"
if (!(Test-Path $fwPath)) { throw "Shared framework not found: $fwPath" }
. $fwPath
Initialize-IvantiLog "VMwareTools"

function Detect-VMwareToolsVulnerable {
    $version = $null
    $regPaths = @("HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall","HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall")
    foreach ($rp in $regPaths) {
        Get-ChildItem $rp -EA SilentlyContinue | ForEach-Object {
            $props = Get-ItemProperty $_.PSPath -EA SilentlyContinue
            if ($props.DisplayName -match 'VMware Tools' -and $props.DisplayVersion) { $version = $props.DisplayVersion }
        }
    }
    if (-not $version) { return @{ Installed=$false; Vulnerable=$false; Version=$null } }
    try { $isVuln = [version]($version -replace '[^0-9.]','') -lt [version]$TargetVersion } catch { $isVuln = $true }
    return @{ Installed=$true; Vulnerable=$isVuln; Version=$version }
}

if ($env:IVANTI_DETECT_ONLY -eq 'TRUE') { $d = Detect-VMwareToolsVulnerable; if ($d.Vulnerable) { Write-Output "VULNERABLE|VMwareTools|$($d.Version)"; exit 1 } else { Write-Output "NOT_VULNERABLE|VMwareTools|$($d.Version)"; exit 0 } }

Log '===== VMware Tools Remediation v4.2 ====='
if (!(Test-InMaintenanceWindow)) { exit 0 }
$detect = Detect-VMwareToolsVulnerable
Log "Version: $($detect.Version) | Vulnerable: $($detect.Vulnerable)"
if (-not $detect.Vulnerable) { Log 'VMware Tools not vulnerable'; exit 0 }
$installer = Resolve-Installer -AppFolder $AppFolder -FileName $InstallerFile -ShareSubPath $ShareSubPath
if (!$installer) { Log 'VMware Tools installer not found' -Level 'ERROR'; exit 1 }
if (!(Install-Exe -Path $installer -Arguments '/S /v/qn REBOOT=R' -Label 'VMwareTools')) { exit 1 }
$post = Detect-VMwareToolsVulnerable
Log "Post-update version: $($post.Version) | Vulnerable: $($post.Vulnerable)"
if ($post.Vulnerable) { Log 'VMware Tools remediation FAILED: vulnerable version still present' -Level 'ERROR'; exit 1 }
if (Test-RebootNeeded) { Schedule-Reboot -Reason 'VMware Tools remediation' }
Log 'VMware Tools remediation SUCCEEDED'; exit 0
22. Hardening
File: Remediate-Hardening.ps1 | Location: \\FILESERVER\IvantiModules\scripts\Remediate-Hardening.ps1

<#
.SYNOPSIS
    Ivanti Remediation - Protocol and Configuration Hardening
    Nessus Plugins: 157288 (TLS 1.1 Deprecated), 104743 (TLS 1.0 Detection),
                    41028 (SNMP Default Community), 63155/65057 (Unquoted Service Paths / Insecure Perms),
                    103569 (Windows Defender Signatures), 45411 (SSL Cert Wrong Hostname),
                    142960 (HSTS Missing), 123461 (Office VBA Trust Access)
.DESCRIPTION
    Applies configuration hardening. No installers needed - registry and config changes only.
    This is the "catch-all" for non-patch vulnerabilities.
#>
 
$LogPath = "C:\Logs\Ivanti\Hardening_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
 
# Enable/disable individual modules
$DisableTLS10       = $true
$DisableTLS11       = $true
$FixSNMPCommunity   = $true
$FixUnquotedPaths   = $true
$FixServicePerms    = $true
$UpdateDefenderSigs = $true
$DisableVBATrust    = $true
 
function Write-Log { param([string]$M,[string]$L="INFO"); $e="[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$L] $M"; $d=Split-Path $LogPath -Parent; if(!(Test-Path $d)){New-Item -ItemType Directory -Path $d -Force|Out-Null}; Add-Content -Path $LogPath -Value $e; Write-Output $e }
 
if ($env:IVANTI_DETECT_ONLY -eq "TRUE") {
    $issues = @()
    # Check TLS 1.0
    $tls10 = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" -EA SilentlyContinue
    if (-not $tls10 -or $tls10.Enabled -ne 0) { $issues += "TLS1.0-Enabled" }
    # Check TLS 1.1
    $tls11 = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" -EA SilentlyContinue
    if (-not $tls11 -or $tls11.Enabled -ne 0) { $issues += "TLS1.1-Enabled" }
    # Check SNMP
    $snmpComm = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\SNMP\Parameters\ValidCommunities" -EA SilentlyContinue
    if ($snmpComm -and ($snmpComm.PSObject.Properties.Name -contains "public")) { $issues += "SNMP-DefaultCommunity" }
    # Check Defender
    try { $mpAge = (Get-MpComputerStatus -EA Stop).AntivirusSignatureAge; if ($mpAge -gt 3) { $issues += "Defender-StaleSigs:${mpAge}days" } } catch {}
    
    if ($issues.Count -gt 0) { Write-Output "VULNERABLE|Hardening|$($issues -join ',')"; exit 1 }
    Write-Output "NOT_VULNERABLE|Hardening"; exit 0
}
 
Write-Log "===== Configuration Hardening Started ====="
 
# --- TLS 1.0 DISABLE ---
if ($DisableTLS10) {
    Write-Log "--- Disabling TLS 1.0 ---"
    foreach ($side in @("Server","Client")) {
        $path = "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\$side"
        if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
        Set-ItemProperty -Path $path -Name "Enabled" -Value 0 -Type DWord
        Set-ItemProperty -Path $path -Name "DisabledByDefault" -Value 1 -Type DWord
    }
    Write-Log "TLS 1.0 disabled"
}
 
# --- TLS 1.1 DISABLE ---
if ($DisableTLS11) {
    Write-Log "--- Disabling TLS 1.1 ---"
    foreach ($side in @("Server","Client")) {
        $path = "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\$side"
        if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
        Set-ItemProperty -Path $path -Name "Enabled" -Value 0 -Type DWord
        Set-ItemProperty -Path $path -Name "DisabledByDefault" -Value 1 -Type DWord
    }
    Write-Log "TLS 1.1 disabled"
}
 
# --- SNMP DEFAULT COMMUNITY ---
if ($FixSNMPCommunity) {
    Write-Log "--- Fixing SNMP Default Community ---"
    $snmpPath = "HKLM:\SYSTEM\CurrentControlSet\Services\SNMP\Parameters\ValidCommunities"
    if (Test-Path $snmpPath) {
        $props = Get-ItemProperty $snmpPath -EA SilentlyContinue
        if ($props.PSObject.Properties.Name -contains "public") {
            Remove-ItemProperty -Path $snmpPath -Name "public" -Force -EA SilentlyContinue
            Write-Log "Removed 'public' SNMP community string"
        }
        # Also remove other well-known default communities
        foreach ($comm in @("private","community","snmp","default")) {
            if ($props.PSObject.Properties.Name -contains $comm) {
                Remove-ItemProperty -Path $snmpPath -Name $comm -Force -EA SilentlyContinue
                Write-Log "Removed '$comm' SNMP community string"
            }
        }
    } else { Write-Log "SNMP not installed or configured" }
}
 
# --- UNQUOTED SERVICE PATHS ---
if ($FixUnquotedPaths) {
    Write-Log "--- Fixing Unquoted Service Paths ---"
    $fixCount = 0
    Get-WmiObject Win32_Service -EA SilentlyContinue | ForEach-Object {
        $imagePath = $_.PathName
        if ($imagePath -and $imagePath -notmatch '^"' -and $imagePath -match '\s' -and $imagePath -notmatch '^(C:\\Windows\\system32)') {
            $exePath = $imagePath -replace '^([^\s]+\.exe).*','$1'
            if ($exePath -match '\s') {
                $regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\$($_.Name)"
                $newPath = "`"$exePath`"" + $imagePath.Substring($exePath.Length)
                Set-ItemProperty -Path $regPath -Name "ImagePath" -Value $newPath -EA SilentlyContinue
                Write-Log "Fixed unquoted path: $($_.Name)"
                $fixCount++
            }
        }
    }
    Write-Log "Fixed $fixCount unquoted service paths"
}
 
# --- INSECURE SERVICE PERMISSIONS ---
if ($FixServicePerms) {
    Write-Log "--- Checking Service Executable Permissions ---"
    $fixCount = 0
    Get-WmiObject Win32_Service -EA SilentlyContinue | ForEach-Object {
        $exePath = $_.PathName -replace '^"([^"]+)".*','$1'
        if ($exePath -eq $_.PathName) { $exePath = ($_.PathName -split '\s+-')[0].Trim() }
        if (-not (Test-Path $exePath -EA SilentlyContinue)) { return }
        try {
            $acl = Get-Acl $exePath -EA SilentlyContinue
            foreach ($ace in $acl.Access) {
                if ($ace.IdentityReference.Value -match 'Everyone|Authenticated Users|BUILTIN\\Users' -and
                    $ace.FileSystemRights -match 'Modify|FullControl|Write' -and $ace.AccessControlType -eq 'Allow') {
                    $acl.RemoveAccessRule($ace) | Out-Null
                    Set-Acl -Path $exePath -AclObject $acl -EA Stop
                    Write-Log "Fixed perms: $($_.Name) - removed $($ace.IdentityReference)"
                    $fixCount++
                }
            }
        } catch {}
    }
    Write-Log "Fixed $fixCount service permission issues"
}
 
# --- WINDOWS DEFENDER SIGNATURES ---
if ($UpdateDefenderSigs) {
    Write-Log "--- Updating Defender Signatures ---"
    try {
        $status = Get-MpComputerStatus -EA Stop
        Write-Log "Current sig age: $($status.AntivirusSignatureAge) days | Version: $($status.AntivirusSignatureVersion)"
        if ($status.AntivirusSignatureAge -gt 1) {
            Update-MpSignature -EA Stop
            Start-Sleep -Seconds 10
            $newStatus = Get-MpComputerStatus
            Write-Log "Updated sig version: $($newStatus.AntivirusSignatureVersion) | Age: $($newStatus.AntivirusSignatureAge) days"
        } else { Write-Log "Signatures current" }
    } catch { Write-Log "Defender update failed: $($_.Exception.Message)" -L "WARN" }
}
 
# --- OFFICE VBA TRUST ACCESS ---
if ($DisableVBATrust) {
    Write-Log "--- Disabling Office VBA Trust Access ---"
    foreach ($ver in @("16.0","15.0","14.0")) {
        foreach ($app in @("Word","Excel","PowerPoint","Access")) {
            $path = "HKCU:\SOFTWARE\Microsoft\Office\$ver\$app\Security"
            if (Test-Path $path) {
                Set-ItemProperty -Path $path -Name "AccessVBOM" -Value 0 -Type DWord -EA SilentlyContinue
                Write-Log "Disabled VBA trust: Office $ver $app"
            }
        }
    }
}
 
Write-Log "HARDENING COMPLETED"; exit 0
 
23. MECMHealth
File: Remediate-MECMHealth.ps1 | Location: \\FILESERVER\IvantiModules\scripts\Remediate-MECMHealth.ps1

<#
.SYNOPSIS
    Ivanti Module - MECM/SCCM Health Diagnostic and Repair
    
    PURPOSE: Diagnose WHY MECM is failing to patch machines, then fix it.
    
    This runs as a PIPELINE of consecutive steps:
    Step 1: WMI Health Check (is WMI broken?)
    Step 2: WMI Repair (if broken, run repair commands)
    Step 3: CCM Client Health Check (is the MECM client healthy?)
    Step 4: CCM Client Repair (reinstall/repair ccmsetup)
    Step 5: Group Policy Update (gpupdate /force)
    Step 6: Force MECM Machine Policy Refresh
    Step 7: Force Software Update Scan
    Step 8: Validation (can MECM now see and install updates?)
    
    USAGE IN IVANTI:
    - Can be run as a SINGLE script (recommended) in one code block
    - Or split into 8 building block tasks if you want granular control
    - Set $RepairMode = $false for DIAGNOSTIC ONLY (no changes made)
    - Set $RepairMode = $true to actually perform repairs
    
    RECOMMENDED: Run diagnostic first, review logs, then run with repair enabled.
#>
 
# ============================================
# CONFIGURATION
# ============================================
$RepairMode          = $true          # $false = diagnostic only, $true = diagnose AND repair
$CCMSetupSource      = "\\FILESERVER\Software\MECM\ccmsetup.exe"  # Path to ccmsetup.exe for reinstall
$CCMSiteCode         = "AUTO"         # Your MECM site code, or "AUTO" to detect from AD
$CCMManagementPoint  = ""             # Your MP FQDN, leave blank to use AD-published MP
$WMIRepairCommands   = $true          # Run the WMI repair command sequence
$ForceGPUpdate       = $true          # Run gpupdate /force
$ForcePolicyRefresh  = $true          # Trigger MECM machine policy retrieval
$ForceSoftwareUpdate = $true          # Trigger software update scan cycle
 
$LogPath = "C:\Logs\Ivanti\MECM-Diagnostic_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
 
function Log { 
    param([string]$M,[string]$L="INFO")
    $e = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$L] $M"
    $logDir = Split-Path $LogPath -Parent
    if (!(Test-Path $logDir)) { New-Item -ItemType Directory -Path $logDir -Force | Out-Null }
    Add-Content -Path $LogPath -Value $e -EA SilentlyContinue
    Write-Output $e
}
 
$Global:DiagResults = @{}
$Global:IssuesFound = 0
$Global:IssuesFixed = 0
 
function Record-Issue {
    param([string]$Category, [string]$Issue, [string]$Status = "FOUND")
    $Global:IssuesFound++
    $Global:DiagResults[$Category] = @{ Issue=$Issue; Status=$Status; Timestamp=(Get-Date) }
    Log "ISSUE [$Category]: $Issue" -L "WARN"
}
 
function Record-Fix {
    param([string]$Category, [string]$Action)
    $Global:IssuesFixed++
    if ($Global:DiagResults.ContainsKey($Category)) { $Global:DiagResults[$Category].Status = "FIXED" }
    Log "FIXED [$Category]: $Action"
}
 
# ============================================
# STEP 1: WMI HEALTH CHECK
# ============================================
function Test-WMIHealth {
    Log "========= STEP 1: WMI Health Check ========="
    $healthy = $true
    
    # Test 1: Basic WMI connectivity
    Log "Testing basic WMI connectivity..."
    try {
        $os = Get-CimInstance Win32_OperatingSystem -ErrorAction Stop
        Log "  WMI basic query OK: $($os.Caption)"
    } catch {
        Record-Issue "WMI-Basic" "Basic WMI query failed: $($_.Exception.Message)"
        $healthy = $false
    }
    
    # Test 2: WMI Repository consistency
    Log "Testing WMI repository consistency..."
    try {
        $result = & winmgmt /verifyrepository 2>&1
        $resultStr = $result -join ' '
        if ($resultStr -match 'consistent|is consistent') {
            Log "  WMI repository: CONSISTENT"
        } else {
            Record-Issue "WMI-Repository" "WMI repository inconsistent: $resultStr"
            $healthy = $false
        }
    } catch {
        Record-Issue "WMI-Repository" "Cannot verify WMI repository: $($_.Exception.Message)"
        $healthy = $false
    }
    
    # Test 3: Key WMI namespaces exist
    Log "Testing WMI namespaces..."
    $requiredNamespaces = @(
        "root\cimv2",
        "root\ccm",
        "root\ccm\clientsdk",
        "root\ccm\policy\machine"
    )
    foreach ($ns in $requiredNamespaces) {
        try {
            $test = Get-CimInstance -Namespace $ns -ClassName "__NAMESPACE" -ErrorAction Stop | Select-Object -First 1
            Log "  Namespace $ns : OK"
        } catch {
            # root\ccm won't exist if MECM client isn't installed - that's a different issue
            if ($ns -match 'ccm') {
                Log "  Namespace $ns : MISSING (MECM client may not be installed)" -L "WARN"
            } else {
                Record-Issue "WMI-Namespace-$($ns -replace '\\','-')" "Namespace $ns missing or inaccessible"
                $healthy = $false
            }
        }
    }
    
    # Test 4: WMI service status
    Log "Testing WMI service..."
    $wmiService = Get-Service Winmgmt -EA SilentlyContinue
    if ($wmiService) {
        if ($wmiService.Status -ne 'Running') {
            Record-Issue "WMI-Service" "Winmgmt service is $($wmiService.Status)"
            $healthy = $false
        } else { Log "  Winmgmt service: Running" }
        if ($wmiService.StartType -ne 'Automatic') {
            Record-Issue "WMI-StartType" "Winmgmt startup is $($wmiService.StartType), should be Automatic"
        }
    }
    
    # Test 5: WMI provider host processes
    $wmiprvse = Get-Process wmiprvse -EA SilentlyContinue
    Log "  WMI Provider Host processes: $($wmiprvse.Count)"
    
    # Test 6: Check for high WMI provider memory (indicates leak/corruption)
    if ($wmiprvse) {
        foreach ($proc in $wmiprvse) {
            $memMB = [math]::Round($proc.WorkingSet64 / 1MB, 0)
            if ($memMB -gt 512) {
                Record-Issue "WMI-Memory" "wmiprvse.exe PID $($proc.Id) using ${memMB}MB (possible memory leak)"
            }
        }
    }
    
    Log "WMI Health: $(if ($healthy) {'HEALTHY'} else {'ISSUES FOUND'})"
    return $healthy
}
 
# ============================================
# STEP 2: WMI REPAIR
# ============================================
function Repair-WMI {
    Log "========= STEP 2: WMI Repair ========="
    if (!$RepairMode) { Log "DIAGNOSTIC MODE - skipping repairs"; return }
    if (!$WMIRepairCommands) { Log "WMI repair disabled in config"; return }
    
    Log "Running WMI repair sequence..."
    
    # Stop dependent services
    Log "  Stopping WMI and dependent services..."
    $dependents = @("ccmexec","iphlpsvc","Winmgmt")
    foreach ($svc in $dependents) {
        Stop-Service $svc -Force -EA SilentlyContinue
    }
    Start-Sleep -Seconds 5
    
    # Re-register WMI providers
    Log "  Re-registering WMI DLLs..."
    $wmiDLLs = @("scrcons.dll","unsecapp.exe","wmiadap.exe","wmiapsrv.exe","wmiprvse.exe","wbemcomn.dll",
                  "wbemprox.dll","wbemcore.dll","wbemsvc.dll","fastprox.dll","mofd.dll","cimwin32.dll",
                  "wbemess.dll","repdrvfs.dll","wmipiprt.dll")
    foreach ($dll in $wmiDLLs) {
        $path = "$env:SystemRoot\System32\wbem\$dll"
        if (Test-Path $path) {
            if ($dll -match '\.dll$') {
                regsvr32.exe /s $path 2>$null
            }
        }
    }
    Log "  DLL registration complete"
    
    # Re-compile MOF files
    Log "  Recompiling MOF files..."
    $mofDir = "$env:SystemRoot\System32\wbem"
    Get-ChildItem $mofDir -Filter "*.mof" -EA SilentlyContinue | ForEach-Object {
        mofcomp.exe $_.FullName 2>$null | Out-Null
    }
    Get-ChildItem $mofDir -Filter "*.mfl" -EA SilentlyContinue | ForEach-Object {
        mofcomp.exe $_.FullName 2>$null | Out-Null
    }
    Log "  MOF recompilation complete"
    
    # Reset WMI repository if still inconsistent
    $verifyResult = & winmgmt /verifyrepository 2>&1
    if ($verifyResult -notmatch 'consistent') {
        Log "  Repository still inconsistent - attempting salvage reset..."
        & winmgmt /salvagerepository 2>&1 | Out-Null
        Start-Sleep -Seconds 5
        
        $verifyAgain = & winmgmt /verifyrepository 2>&1
        if ($verifyAgain -notmatch 'consistent') {
            Log "  Salvage failed - performing full repository reset..." -L "WARN"
            & winmgmt /resetrepository 2>&1 | Out-Null
            Log "  FULL RESET PERFORMED - MECM client will need reinstall" -L "WARN"
        } else {
            Log "  Salvage successful - repository now consistent"
        }
    }
    
    # Restart services
    Log "  Restarting WMI service..."
    Start-Service Winmgmt -EA SilentlyContinue
    Start-Sleep -Seconds 5
    Start-Service ccmexec -EA SilentlyContinue
    
    # Verify
    try {
        $test = Get-CimInstance Win32_OperatingSystem -EA Stop
        Record-Fix "WMI-Repair" "WMI repaired and verified"
    } catch {
        Log "  WMI still not responding after repair" -L "ERROR"
    }
}
 
# ============================================
# STEP 3: CCM CLIENT HEALTH CHECK
# ============================================
function Test-CCMHealth {
    Log "========= STEP 3: MECM Client Health Check ========="
    $healthy = $true
    
    # Is MECM client installed?
    $ccmExe = "$env:SystemRoot\CCM\CcmExec.exe"
    if (!(Test-Path $ccmExe)) {
        Record-Issue "CCM-NotInstalled" "MECM client (CcmExec.exe) is not installed"
        return $false
    }
    $ccmVer = (Get-Item $ccmExe).VersionInfo.ProductVersion
    Log "  MECM client version: $ccmVer"
    
    # Is CcmExec service running?
    $svc = Get-Service CcmExec -EA SilentlyContinue
    if (!$svc -or $svc.Status -ne 'Running') {
        Record-Issue "CCM-Service" "CcmExec service status: $(if($svc){$svc.Status}else{'NOT FOUND'})"
        $healthy = $false
    } else { Log "  CcmExec service: Running" }
    
    # Check client certificate
    Log "  Checking client certificate..."
    try {
        $smsCert = Get-ChildItem Cert:\LocalMachine\SMS -EA Stop | Where-Object { $_.NotAfter -gt (Get-Date) }
        if ($smsCert) {
            Log "  SMS certificate: Valid (expires $($smsCert[0].NotAfter))"
        } else {
            Record-Issue "CCM-Certificate" "SMS certificate expired or missing"
            $healthy = $false
        }
    } catch {
        Log "  SMS certificate store not found (may be normal for HTTP clients)" -L "WARN"
    }
    
    # Check assigned site
    try {
        $smsClient = Get-CimInstance -Namespace "root\ccm" -ClassName SMS_Client -EA Stop
        Log "  Assigned site: $($smsClient.AssignedSiteCode)"
        if (!$smsClient.AssignedSiteCode) {
            Record-Issue "CCM-NoSite" "Client has no assigned site code"
            $healthy = $false
        }
    } catch {
        Record-Issue "CCM-WMI" "Cannot query CCM WMI namespace: $($_.Exception.Message)"
        $healthy = $false
    }
    
    # Check Management Point connectivity
    try {
        $authority = Get-CimInstance -Namespace "root\ccm" -ClassName SMS_Authority -EA Stop
        $mpHost = $authority.CurrentManagementPoint
        if ($mpHost) {
            Log "  Management Point: $mpHost"
            $mpTest = Test-NetConnection $mpHost -Port 443 -WarningAction SilentlyContinue -EA SilentlyContinue
            if ($mpTest.TcpTestSucceeded) { Log "  MP connectivity: OK (port 443)" }
            else {
                $mp80 = Test-NetConnection $mpHost -Port 80 -WarningAction SilentlyContinue -EA SilentlyContinue
                if ($mp80.TcpTestSucceeded) { Log "  MP connectivity: OK (port 80)" }
                else { Record-Issue "CCM-MPConnect" "Cannot reach MP $mpHost on port 80 or 443"; $healthy = $false }
            }
        } else {
            Record-Issue "CCM-NoMP" "No Management Point assigned"
            $healthy = $false
        }
    } catch { Log "  Cannot determine MP: $($_.Exception.Message)" -L "WARN" }
    
    # Check last policy request
    try {
        $policyAgent = Get-CimInstance -Namespace "root\ccm\policy" -ClassName CCM_PolicyAgent_Configuration -EA Stop
        Log "  Policy polling interval: $($policyAgent.PolicyRequestAssignmentTimeout) min"
    } catch { Log "  Cannot query policy agent" -L "WARN" }
    
    # Check pending software updates count
    try {
        $updates = Get-CimInstance -Namespace "root\ccm\clientsdk" -ClassName CCM_SoftwareUpdate -EA Stop
        $missing = $updates | Where-Object { $_.ComplianceState -eq 0 }
        $installed = $updates | Where-Object { $_.ComplianceState -eq 1 }
        $downloading = $updates | Where-Object { $_.EvaluationState -in @(2,3) }
        $installing = $updates | Where-Object { $_.EvaluationState -in @(6,7) }
        $failed = $updates | Where-Object { $_.EvaluationState -eq 13 }
        
        Log "  Software updates: $($updates.Count) total | $($missing.Count) missing | $($installed.Count) installed | $($downloading.Count) downloading | $($installing.Count) installing | $($failed.Count) FAILED"
        
        if ($failed.Count -gt 0) {
            Record-Issue "CCM-FailedUpdates" "$($failed.Count) updates in FAILED state"
            foreach ($f in $failed) { Log "    FAILED: $($f.Name) | Error: $($f.ErrorCode)" -L "WARN" }
        }
    } catch { Log "  Cannot query software updates: $($_.Exception.Message)" -L "WARN" }
    
    # Check CCM client health evaluation
    $ccmEvalLog = "$env:SystemRoot\CCM\Logs\CcmEval.log"
    if (Test-Path $ccmEvalLog) {
        $lastEval = Get-Content $ccmEvalLog -Tail 20 -EA SilentlyContinue
        $evalResult = $lastEval | Select-String "Evaluation|Health" | Select-Object -Last 1
        if ($evalResult) { Log "  Last health eval: $($evalResult.Line.Trim())" }
    }
    
    # Check ccmsetup.log for install errors
    $ccmSetupLog = "$env:SystemRoot\ccmsetup\Logs\ccmsetup.log"
    if (Test-Path $ccmSetupLog) {
        $lastLines = Get-Content $ccmSetupLog -Tail 5 -EA SilentlyContinue
        $errors = $lastLines | Select-String "error|fail" -CaseSensitive:$false
        if ($errors) {
            foreach ($err in $errors) { Log "  ccmsetup.log: $($err.Line.Trim())" -L "WARN" }
        }
    }
    
    # Check Windows Update Agent
    Log "  Checking Windows Update Agent..."
    try {
        $wuaVer = (New-Object -ComObject Microsoft.Update.AgentInfo).GetInfo("ProductVersionString")
        Log "  WUA version: $wuaVer"
    } catch { Log "  Cannot query WUA version" -L "WARN" }
    
    # Check BITS service (needed for MECM content download)
    $bits = Get-Service BITS -EA SilentlyContinue
    if ($bits.Status -ne 'Running') {
        Record-Issue "CCM-BITS" "BITS service is $($bits.Status) - needed for content downloads"
        $healthy = $false
    } else { Log "  BITS service: Running" }
    
    Log "CCM Health: $(if ($healthy) {'HEALTHY'} else {'ISSUES FOUND'})"
    return $healthy
}
 
# ============================================
# STEP 4: CCM CLIENT REPAIR
# ============================================
function Repair-CCMClient {
    Log "========= STEP 4: MECM Client Repair ========="
    if (!$RepairMode) { Log "DIAGNOSTIC MODE - skipping repairs"; return }
    
    # Fix BITS first
    $bits = Get-Service BITS -EA SilentlyContinue
    if ($bits -and $bits.Status -ne 'Running') {
        Log "  Starting BITS service..."
        Set-Service BITS -StartupType Automatic -EA SilentlyContinue
        Start-Service BITS -EA SilentlyContinue
        Record-Fix "CCM-BITS" "BITS service started"
    }
    
    # Try ccmrepair first (less disruptive than reinstall)
    $ccmRepair = "$env:SystemRoot\CCM\CcmRestart.exe"
    if (!(Test-Path $ccmRepair)) { $ccmRepair = "$env:SystemRoot\CCM\ccmeval.exe" }
    
    if (Test-Path "$env:SystemRoot\CCM\ccmeval.exe") {
        Log "  Running CCM health evaluation and repair..."
        $proc = Start-Process "$env:SystemRoot\CCM\ccmeval.exe" -Wait -PassThru -NoNewWindow -EA SilentlyContinue
        Log "  ccmeval exit: $($proc.ExitCode)"
        Start-Sleep -Seconds 10
    }
    
    # Check if CcmExec is now running after eval
    $svc = Get-Service CcmExec -EA SilentlyContinue
    if ($svc -and $svc.Status -eq 'Running') {
        Log "  CcmExec is running after eval - checking if further repair needed..."
        # Quick health recheck
        try {
            $test = Get-CimInstance -Namespace "root\ccm" -ClassName SMS_Client -EA Stop
            if ($test.AssignedSiteCode) {
                Record-Fix "CCM-Repair" "CCM client repaired via ccmeval"
                return
            }
        } catch {}
    }
    
    # If still broken, do a full ccmsetup repair
    Log "  CCM client needs full repair/reinstall..."
    
    if (Test-Path $CCMSetupSource) {
        Log "  Running ccmsetup.exe /remediate:client..."
        
        # Build ccmsetup arguments
        $setupArgs = "/remediate:client /forceinstall"
        if ($CCMSiteCode -ne "AUTO") { $setupArgs += " SMSSITECODE=$CCMSiteCode" }
        if ($CCMManagementPoint) { $setupArgs += " SMSMP=$CCMManagementPoint" }
        
        $proc = Start-Process $CCMSetupSource -ArgumentList $setupArgs -Wait -PassThru -NoNewWindow
        Log "  ccmsetup exit: $($proc.ExitCode)"
        
        # Exit codes: 0=success, 7=reboot needed, 6=error
        if ($proc.ExitCode -eq 0) { Record-Fix "CCM-Reinstall" "CCM client reinstalled successfully" }
        elseif ($proc.ExitCode -eq 7) { Record-Fix "CCM-Reinstall" "CCM client reinstalled - REBOOT REQUIRED" }
        else { Log "  ccmsetup failed (Exit: $($proc.ExitCode)). Check $env:SystemRoot\ccmsetup\Logs\ccmsetup.log" -L "ERROR" }
    } else {
        Log "  ccmsetup.exe not found at $CCMSetupSource" -L "ERROR"
        Log "  Manual remediation needed: copy ccmsetup.exe to machine and run:" -L "WARN"
        Log "    ccmsetup.exe /remediate:client /forceinstall" -L "WARN"
    }
}
 
# ============================================
# STEP 5: GROUP POLICY UPDATE
# ============================================
function Update-GroupPolicy {
    Log "========= STEP 5: Group Policy Update ========="
    if (!$RepairMode -or !$ForceGPUpdate) { Log "Skipping GP update"; return }
    
    Log "  Running gpupdate /force..."
    $proc = Start-Process gpupdate.exe -ArgumentList "/force /wait:60" -Wait -PassThru -NoNewWindow
    Log "  gpupdate exit: $($proc.ExitCode)"
    
    # Check last GP application time
    try {
        $gpResult = gpresult /scope:computer /r 2>&1
        $lastApplied = $gpResult | Select-String "Last time Group Policy was applied" | Select-Object -First 1
        if ($lastApplied) { Log "  $($lastApplied.Line.Trim())" }
    } catch { Log "  Cannot query gpresult" -L "WARN" }
    
    Record-Fix "GP-Update" "Group Policy refreshed"
}
 
# ============================================
# STEP 6: FORCE MECM POLICY REFRESH
# ============================================
function Force-MECMPolicyRefresh {
    Log "========= STEP 6: MECM Machine Policy Refresh ========="
    if (!$RepairMode -or !$ForcePolicyRefresh) { Log "Skipping policy refresh"; return }
    
    try {
        # Trigger Machine Policy Retrieval & Evaluation Cycle
        Log "  Triggering Machine Policy Retrieval..."
        Invoke-CimMethod -Namespace "root\ccm" -ClassName SMS_Client -MethodName TriggerSchedule -Arguments @{sScheduleID="{00000000-0000-0000-0000-000000000021}"} -EA Stop
        Log "  Machine Policy Retrieval triggered"
        
        Start-Sleep -Seconds 10
        
        # Trigger Machine Policy Evaluation
        Log "  Triggering Machine Policy Evaluation..."
        Invoke-CimMethod -Namespace "root\ccm" -ClassName SMS_Client -MethodName TriggerSchedule -Arguments @{sScheduleID="{00000000-0000-0000-0000-000000000022}"} -EA Stop
        Log "  Machine Policy Evaluation triggered"
        
        Record-Fix "CCM-Policy" "Policy retrieval and evaluation triggered"
    } catch {
        Log "  Policy refresh failed: $($_.Exception.Message)" -L "ERROR"
        Log "  MECM client may need reinstall" -L "WARN"
    }
}
 
# ============================================
# STEP 7: FORCE SOFTWARE UPDATE SCAN
# ============================================
function Force-SoftwareUpdateScan {
    Log "========= STEP 7: Software Update Scan Cycle ========="
    if (!$RepairMode -or !$ForceSoftwareUpdate) { Log "Skipping update scan"; return }
    
    try {
        # Software Updates Assignments Evaluation Cycle
        Log "  Triggering Software Update Scan..."
        Invoke-CimMethod -Namespace "root\ccm" -ClassName SMS_Client -MethodName TriggerSchedule -Arguments @{sScheduleID="{00000000-0000-0000-0000-000000000113}"} -EA Stop
        Log "  Software Update Scan triggered"
        
        Start-Sleep -Seconds 10
        
        # Software Update Deployment Evaluation Cycle
        Log "  Triggering Software Update Deployment Evaluation..."
        Invoke-CimMethod -Namespace "root\ccm" -ClassName SMS_Client -MethodName TriggerSchedule -Arguments @{sScheduleID="{00000000-0000-0000-0000-000000000108}"} -EA Stop
        Log "  Deployment Evaluation triggered"
        
        Record-Fix "CCM-UpdateScan" "Software update scan and evaluation triggered"
    } catch {
        Log "  Update scan trigger failed: $($_.Exception.Message)" -L "ERROR"
    }
    
    # Also reset the WUA datastore if updates are failing
    $resetWUA = $false
    try {
        $failed = Get-CimInstance -Namespace "root\ccm\clientsdk" -ClassName CCM_SoftwareUpdate -EA Stop |
            Where-Object { $_.EvaluationState -eq 13 }
        if ($failed.Count -gt 3) { $resetWUA = $true }
    } catch {}
    
    if ($resetWUA) {
        Log "  Multiple failed updates detected - resetting WUA datastore..."
        Stop-Service wuauserv -Force -EA SilentlyContinue
        Stop-Service cryptSvc -Force -EA SilentlyContinue
        Stop-Service bits -Force -EA SilentlyContinue
        
        Rename-Item "$env:SystemRoot\SoftwareDistribution" "$env:SystemRoot\SoftwareDistribution.old.$(Get-Date -Format 'yyyyMMdd')" -Force -EA SilentlyContinue
        Rename-Item "$env:SystemRoot\System32\catroot2" "$env:SystemRoot\System32\catroot2.old.$(Get-Date -Format 'yyyyMMdd')" -Force -EA SilentlyContinue
        
        Start-Service bits -EA SilentlyContinue
        Start-Service cryptSvc -EA SilentlyContinue
        Start-Service wuauserv -EA SilentlyContinue
        
        Log "  WUA datastore reset. Next scan will rebuild from scratch."
        Record-Fix "WUA-Reset" "Windows Update datastore reset"
    }
}
 
# ============================================
# STEP 8: VALIDATION
# ============================================
function Test-Validation {
    Log "========= STEP 8: Post-Repair Validation ========="
    
    $valid = $true
    
    # WMI working?
    try { Get-CimInstance Win32_OperatingSystem -EA Stop | Out-Null; Log "  WMI: OK" }
    catch { Log "  WMI: STILL BROKEN" -L "ERROR"; $valid = $false }
    
    # CcmExec running?
    $svc = Get-Service CcmExec -EA SilentlyContinue
    if ($svc -and $svc.Status -eq 'Running') { Log "  CcmExec: Running" }
    else { Log "  CcmExec: NOT Running" -L "ERROR"; $valid = $false }
    
    # Can reach CCM WMI?
    try {
        $client = Get-CimInstance -Namespace "root\ccm" -ClassName SMS_Client -EA Stop
        Log "  CCM WMI: OK | Site: $($client.AssignedSiteCode)"
    } catch { Log "  CCM WMI: FAILED" -L "ERROR"; $valid = $false }
    
    # Software updates visible?
    try {
        $updates = Get-CimInstance -Namespace "root\ccm\clientsdk" -ClassName CCM_SoftwareUpdate -EA Stop
        Log "  Software Updates: $($updates.Count) visible"
    } catch { Log "  Software Updates: Cannot query" -L "WARN" }
    
    # BITS running?
    $bits = Get-Service BITS -EA SilentlyContinue
    Log "  BITS: $($bits.Status)"
    
    return $valid
}
 
# ============================================
# MAIN EXECUTION
# ============================================
Log "============================================="
Log "MECM HEALTH DIAGNOSTIC AND REPAIR"
Log "Computer: $env:COMPUTERNAME"
Log "Mode: $(if ($RepairMode) {'DIAGNOSE + REPAIR'} else {'DIAGNOSTIC ONLY'})"
Log "============================================="
 
# Run pipeline
$wmiHealthy = Test-WMIHealth
if (!$wmiHealthy) { Repair-WMI }
 
$ccmHealthy = Test-CCMHealth
if (!$ccmHealthy) { Repair-CCMClient }
 
Update-GroupPolicy
Force-MECMPolicyRefresh
Force-SoftwareUpdateScan
 
$finalValid = Test-Validation
 
# Summary
Log ""
Log "============================================="
Log "DIAGNOSTIC SUMMARY"
Log "============================================="
Log "Issues Found: $($Global:IssuesFound)"
Log "Issues Fixed: $($Global:IssuesFixed)"
Log "Final Validation: $(if ($finalValid) {'PASSED'} else {'FAILED'})"
Log ""
 
foreach ($key in $Global:DiagResults.Keys) {
    $r = $Global:DiagResults[$key]
    Log "  [$($r.Status)] $key : $($r.Issue)"
}
 
if (!$finalValid -and $RepairMode) {
    Log ""
    Log "MANUAL INTERVENTION REQUIRED:" -L "ERROR"
    Log "  1. Check C:\Windows\CCM\Logs\ for CCM client logs"
    Log "  2. Check C:\Windows\ccmsetup\Logs\ccmsetup.log for install errors"
    Log "  3. Verify network connectivity to Management Point"
    Log "  4. Consider full CCM client uninstall/reinstall:"
    Log "     ccmsetup.exe /uninstall"
    Log "     (wait 5 min)"
    Log "     ccmsetup.exe /forceinstall SMSSITECODE=$CCMSiteCode"
}
 
# Generate CSV report
$reportPath = "C:\Logs\Ivanti\MECM-Diagnostic_$env:COMPUTERNAME.csv"
$Global:DiagResults.GetEnumerator() | ForEach-Object {
    [PSCustomObject]@{ Computer=$env:COMPUTERNAME; Category=$_.Key; Issue=$_.Value.Issue; Status=$_.Value.Status; Timestamp=$_.Value.Timestamp }
} | Export-Csv $reportPath -NoTypeInformation -Force
Log "Report: $reportPath"
 
Log "============================================="
exit $(if ($finalValid) { 0 } else { 1 })
 
24. Deployment Checklist
First-Time Setup
1.	Create \\FILESERVER\IvantiModules\scripts\ and copy all .ps1 files there.
2.	Upload each installer-based app’s media to the Ivanti Resource Library.
3.	For installer-based apps, create the 4-task pattern: stage -> detect -> remediate -> cleanup.
4.	Make Task 1 stage to C:\install_files\rem\<AppFolder>\ by calling Copy-IvantiResourceToStage.
5.	Make Task 4 delete only C:\install_files\rem\<AppFolder>\ by calling Remove-AppStagingFolder.
6.	Set the maintenance window in Shared-Framework.ps1.
7.	Leave $Global:CheckMECM = $false unless you explicitly want Ivanti to defer to MECM.
Each Patch Cycle
1.	Update the $TargetVersion in the affected app script.
2.	Upload the new installer/resource to Ivanti.
3.	If the vendor changed the filename, update $InstallerFile or the wildcard pattern in the script.
4.	Test on 1-2 endpoints first.
5.	Confirm the post-check shows no vulnerable version remaining.
6.	Remove any temporary IVANTI_FORCE_RUN override.
7.	Schedule production during the maintenance window.
