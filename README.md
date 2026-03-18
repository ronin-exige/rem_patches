
```
$target = [version]'146.0.7680.80'
$update = 'No'

$roots = @(
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKCU:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
)

$apps = foreach ($root in $roots) {
    Get-ItemProperty -Path $root -ErrorAction SilentlyContinue |
        Where-Object { $_.DisplayName -like 'Google Chrome*' }
}

$versions = @()
foreach ($app in $apps) {
    try {
        if ($app.DisplayVersion) {
            $versions += [version]$app.DisplayVersion
        }
    } catch {}
}

if ($versions.Count -gt 0) {
    $installed = $versions | Sort-Object -Descending | Select-Object -First 1
    if ($installed -lt $target) {
        $update = 'Yes'
    }
}

Write-Output $update
exit 0
```




chrome test:

```
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*,
HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
Where-Object { $_.DisplayName -like "Google Chrome*" } |
Select-Object DisplayName, DisplayVersion
```

```
Test-Path "$env:ProgramFiles\Google\Chrome"
Test-Path "${env:ProgramFiles(x86)}\Google\Chrome"
```

safe nuke:
```

# =========================
# Nuke Google Chrome / Chrome Enterprise from Windows
# Run as Administrator
# =========================

#Requires -RunAsAdministrator

# --- Configuration ---
$LogFile = "$env:ProgramData\Logs\ChromeNuke_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
$DryRun  = $false   # Set to $true to log actions without executing destructive operations

# --- Logging ---
function Log {
    param(
        [string]$Message,
        [ValidateSet("INFO","WARN","ERROR","SUCCESS")]
        [string]$Level = "INFO"
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $entry = "[$timestamp] [$Level] $Message"

    # Ensure log directory exists
    $logDir = Split-Path $LogFile -Parent
    if (-not (Test-Path $logDir)) { New-Item -ItemType Directory -Path $logDir -Force | Out-Null }

    Add-Content -Path $LogFile -Value $entry

    $color = switch ($Level) {
        "INFO"    { "Cyan" }
        "WARN"    { "Yellow" }
        "ERROR"   { "Red" }
        "SUCCESS" { "Green" }
    }
    Write-Host $entry -ForegroundColor $color
}

# --- Pre-flight checks ---
Log "Chrome Nuke script starting. DryRun=$DryRun" "INFO"
Log "Logging to: $LogFile" "INFO"

$chromeInstalled = (
    (Test-Path "$env:ProgramFiles\Google\Chrome") -or
    (Test-Path "${env:ProgramFiles(x86)}\Google\Chrome") -or
    (Test-Path "$env:LocalAppData\Google\Chrome")
)

if (-not $chromeInstalled) {
    Log "No Chrome installation detected. Exiting." "WARN"
    exit 0
}

# ============================================================
# 1. Kill Chrome and Google Update processes (scoped by path)
# ============================================================
Log "== Killing Chrome and Google Update processes ==" "INFO"

# These are safe to kill by name — they are Chrome/Google-specific
$safeProcessNames = @(
    "chrome",
    "GoogleCrashHandler",
    "GoogleCrashHandler64",
    "GoogleUpdate",
    "GoogleUpdateOnDemand"
)

foreach ($p in $safeProcessNames) {
    $procs = Get-Process -Name $p -ErrorAction SilentlyContinue
    if ($procs) {
        foreach ($proc in $procs) {
            Log "Killing process: $($proc.Name) (PID $($proc.Id))" "INFO"
            if (-not $DryRun) { $proc | Stop-Process -Force -ErrorAction SilentlyContinue }
        }
    }
}

# For generic names, only kill if the process image path is under a Google directory
$genericProcessNames = @("setup", "installer", "msedgewebview2")
foreach ($p in $genericProcessNames) {
    $procs = Get-Process -Name $p -ErrorAction SilentlyContinue
    if ($procs) {
        foreach ($proc in $procs) {
            try {
                $procPath = $proc.Path
                if ($procPath -and ($procPath -match "\\Google\\")) {
                    Log "Killing Google-owned process: $($proc.Name) (PID $($proc.Id)) at $procPath" "INFO"
                    if (-not $DryRun) { $proc | Stop-Process -Force }
                } else {
                    Log "Skipping non-Google process: $($proc.Name) (PID $($proc.Id)) at $procPath" "WARN"
                }
            } catch {
                Log "Could not inspect process $($proc.Name) (PID $($proc.Id)): $($_.Exception.Message)" "WARN"
            }
        }
    }
}

# ============================================================
# 2. Stop and disable Google Update services
# ============================================================
Log "== Stopping Google Update services ==" "INFO"
$serviceNames = @("gupdate", "gupdatem")
foreach ($svc in $serviceNames) {
    $service = Get-Service -Name $svc -ErrorAction SilentlyContinue
    if ($service) {
        Log "Stopping and disabling service: $svc (Status: $($service.Status))" "INFO"
        if (-not $DryRun) {
            Stop-Service -Name $svc -Force -ErrorAction SilentlyContinue
            sc.exe config $svc start= disabled | Out-Null
        }
    }
}

# ============================================================
# 3. Remove Google scheduled tasks
# ============================================================
Log "== Removing Google scheduled tasks ==" "INFO"
$taskPaths = @(
    "\GoogleUpdateTaskMachineCore",
    "\GoogleUpdateTaskMachineUA",
    "\GoogleSystem",
    "\GoogleUpdaterTaskSystem*",
    "\GoogleUpdaterTaskUser*"
)

foreach ($tp in $taskPaths) {
    $result = schtasks.exe /Query /TN $tp 2>&1
    if ($LASTEXITCODE -eq 0) {
        Log "Deleting scheduled task: $tp" "INFO"
        if (-not $DryRun) { schtasks.exe /Delete /TN $tp /F | Out-Null }
    }
}

# ============================================================
# 4. Remove Chrome uninstall registry entries
# ============================================================
Log "== Removing Chrome uninstall registry entries ==" "INFO"

$uninstallRoots = @(
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"
)

foreach ($root in $uninstallRoots) {
    if (Test-Path $root) {
        Get-ChildItem $root | ForEach-Object {
            $item = Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue
            $name = $item.DisplayName

            if (
                ($name -like "Google Chrome*") -or
                ($name -like "*Chrome Enterprise*") -or
                ($name -like "*Google Update*")
            ) {
                Log "Removing uninstall key: $($_.PSChildName) / $name" "INFO"
                if (-not $DryRun) { Remove-Item $_.PSPath -Recurse -Force }
            }
        }
    }
}

# ============================================================
# 5. Remove Installer product references for Chrome
# ============================================================
Log "== Removing Installer product references ==" "INFO"

$installerRoots = @(
    "HKLM:\SOFTWARE\Classes\Installer\Products",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Installer\UserData\S-1-5-18\Products",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Installer\UserData\S-1-5-18\Components"
)

foreach ($root in $installerRoots) {
    if (Test-Path $root) {
        Get-ChildItem $root -ErrorAction SilentlyContinue | ForEach-Object {
            $props = Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue
            # Match on ProductName or known value properties rather than entire Out-String blob
            $productName = $props.ProductName
            $displayName = $props.DisplayName
            $combined = "$productName $displayName"

            if ($combined -match "Google Chrome|Chrome Enterprise|Google Update") {
                Log "Removing installer key: $($_.PSChildName) (matched: $combined)" "INFO"
                if (-not $DryRun) { Remove-Item $_.PSPath -Recurse -Force }
            }
        }
    }
}

# ============================================================
# 6. Remove Chrome / Google registry keys
# ============================================================
Log "== Removing Google registry keys ==" "INFO"
$regPaths = @(
    "HKLM:\SOFTWARE\Google",
    "HKLM:\SOFTWARE\WOW6432Node\Google",
    "HKCU:\SOFTWARE\Google",
    "HKLM:\SOFTWARE\Policies\Google",
    "HKCU:\SOFTWARE\Policies\Google"
)

foreach ($rp in $regPaths) {
    if (Test-Path $rp) {
        Log "Removing registry path: $rp" "INFO"
        if (-not $DryRun) { Remove-Item $rp -Recurse -Force }
    }
}

# ============================================================
# 7. Remove Chrome / Google files and directories
# ============================================================
Log "== Removing Chrome / Google files ==" "INFO"
$filePaths = @(
    "$env:ProgramFiles\Google\Chrome",
    "${env:ProgramFiles(x86)}\Google\Chrome",
    "$env:ProgramFiles\Google\Update",
    "${env:ProgramFiles(x86)}\Google\Update",
    "$env:ProgramData\Google",
    "$env:LocalAppData\Google\Chrome",
    "$env:LocalAppData\Google\Update",
    "$env:AppData\Google\Chrome"
)

foreach ($fp in $filePaths) {
    if (Test-Path $fp) {
        Log "Removing directory: $fp" "INFO"
        if (-not $DryRun) {
            try {
                Remove-Item $fp -Recurse -Force -ErrorAction Stop
            } catch {
                Log "PowerShell remove failed for $fp, falling back to rmdir" "WARN"
                cmd /c "rmdir /s /q `"$fp`"" 2>&1 | Out-Null
                if (Test-Path $fp) {
                    Log "Failed to fully remove: $fp (may require reboot)" "ERROR"
                } else {
                    Log "rmdir fallback succeeded for: $fp" "INFO"
                }
            }
        }
    }
}

# ============================================================
# 8. Remove Start Menu and Desktop shortcuts
# ============================================================
Log "== Removing Chrome shortcuts ==" "INFO"
$shortcutPaths = @(
    "$env:ProgramData\Microsoft\Windows\Start Menu\Programs\Google Chrome.lnk",
    "$env:AppData\Microsoft\Windows\Start Menu\Programs\Google Chrome.lnk",
    "$env:Public\Desktop\Google Chrome.lnk",
    "$env:UserProfile\Desktop\Google Chrome.lnk"
)
foreach ($sp in $shortcutPaths) {
    if (Test-Path $sp) {
        Log "Removing shortcut: $sp" "INFO"
        if (-not $DryRun) { Remove-Item $sp -Force -ErrorAction SilentlyContinue }
    }
}

# ============================================================
# 9. Remove Google Update services from SCM
# ============================================================
Log "== Removing Google Update services from SCM ==" "INFO"
foreach ($svc in @("gupdate", "gupdatem")) {
    $exists = sc.exe query $svc 2>&1
    if ($LASTEXITCODE -eq 0) {
        Log "Deleting service from SCM: $svc" "INFO"
        if (-not $DryRun) { sc.exe delete $svc | Out-Null }
    }
}

# ============================================================
# Summary
# ============================================================
Log "====================================" "SUCCESS"
Log "Chrome nuke pass complete." "SUCCESS"
if ($DryRun) {
    Log "DRY RUN — no changes were made. Review log and re-run with DryRun=`$false" "WARN"
}
Log "REBOOT recommended before reinstalling Chrome." "SUCCESS"
Log "Log saved to: $LogFile" "INFO"
```


```
function Get-PeArchitecture {
    param([Parameter(Mandatory)][string]$Path)
    $fs = [System.IO.File]::Open($Path, 'Open', 'Read', 'ReadWrite')
    try {
        $br = New-Object System.IO.BinaryReader($fs)
        $fs.Seek(0x3C, [System.IO.SeekOrigin]::Begin) | Out-Null
        $peOffset = $br.ReadInt32()
        $fs.Seek($peOffset + 4, [System.IO.SeekOrigin]::Begin) | Out-Null
        $machine = $br.ReadUInt16()
        switch ($machine) {
            0x014c { '32' }
            0x8664 { '64' }
            default { 'NONE' }
        }
    } finally { $fs.Dispose() }
}

function Get-DisplayIconPath {
    param([string]$DisplayIcon)
    if ([string]::IsNullOrWhiteSpace($DisplayIcon)) { return $null }
    $icon = $DisplayIcon.Trim()
    if ($icon -match '^\s*"([^"]+)"') { return $matches[1] }
    return ($icon -split ',')[0].Trim()
}

function Resolve-BrowserPath {
    param([string]$ExeName,[string]$DisplayRegex,[string[]]$CommonPaths)

    $candidates = New-Object System.Collections.Generic.List[string]

    foreach ($key in @(
        "Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\$ExeName",
        "Registry::HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\$ExeName"
    )) {
        try {
            $item = Get-ItemProperty -Path $key -ErrorAction Stop
            $path = $item.'(default)'
            if (-not $path) { $path = $item.Path }
            if ($path -and (Test-Path -LiteralPath $path)) { $candidates.Add($path) }
        } catch {}
    }

    foreach ($root in @(
        'Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'Registry::HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'Registry::HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'Registry::HKEY_CURRENT_USER\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
    )) {
        try {
            Get-ItemProperty -Path $root -ErrorAction SilentlyContinue | ForEach-Object {
                if ($_.DisplayName -and ($_.DisplayName -match '^(Google Chrome)(\s|$)')) {
                    $path = $null
                    if ($_.InstallLocation) {
                        $tryExe = Join-Path $_.InstallLocation 'chrome.exe'
                        if (Test-Path -LiteralPath $tryExe) { $path = $tryExe }
                    }
                    if (-not $path -and $_.DisplayIcon) {
                        $iconPath = Get-DisplayIconPath -DisplayIcon $_.DisplayIcon
                        if ($iconPath -and (Test-Path -LiteralPath $iconPath)) { $path = $iconPath }
                    }
                    if ($path) { $candidates.Add($path) }
                }
            }
        } catch {}
    }

    foreach ($path in @(
        "$env:ProgramFiles\Google\Chrome\Application\chrome.exe",
        "${env:ProgramFiles(x86)}\Google\Chrome\Application\chrome.exe",
        "$env:LocalAppData\Google\Chrome\Application\chrome.exe"
    )) {
        if ($path -and (Test-Path -LiteralPath $path)) { $candidates.Add($path) }
    }

    $seen = @{}
    foreach ($p in $candidates) {
        if (-not $seen.ContainsKey($p)) {
            $seen[$p] = $true
            return $p
        }
    }
    return $null
}

$path = Resolve-BrowserPath -ExeName 'chrome.exe' -DisplayRegex '^(Google Chrome)(\s|$)' -CommonPaths @()
if (-not $path) { Write-Output 'NONE'; exit 0 }

Write-Output (Get-PeArchitecture -Path $path)
exit 0
```
------------------------------------------------------------

giga nuke:
```
# =========================
# Nuke Google Chrome / Chrome Enterprise from Windows
# TEST/LAB USE ONLY
# Run as Administrator
# =========================

$ErrorActionPreference = "SilentlyContinue"

Write-Host "== Killing Chrome and Google Update processes ==" -ForegroundColor Cyan
$procNames = @(
    "chrome",
    "msedgewebview2",
    "GoogleCrashHandler",
    "GoogleCrashHandler64",
    "GoogleUpdate",
    "GoogleUpdateOnDemand",
    "setup",
    "installer"
)

foreach ($p in $procNames) {
    Get-Process -Name $p -ErrorAction SilentlyContinue | Stop-Process -Force
}

Write-Host "== Stopping Google Update services ==" -ForegroundColor Cyan
$serviceNames = @("gupdate", "gupdatem")
foreach ($svc in $serviceNames) {
    if (Get-Service -Name $svc -ErrorAction SilentlyContinue) {
        Stop-Service -Name $svc -Force
        sc.exe config $svc start= disabled | Out-Null
    }
}

Write-Host "== Removing Google scheduled tasks ==" -ForegroundColor Cyan
$taskPaths = @(
    "\GoogleUpdateTaskMachineCore",
    "\GoogleUpdateTaskMachineUA",
    "\GoogleSystem",
    "\GoogleUpdaterTaskSystem*",
    "\GoogleUpdaterTaskUser*"
)

foreach ($tp in $taskPaths) {
    schtasks.exe /Delete /TN $tp /F | Out-Null
}

Write-Host "== Attempting MSI uninstall entries removal ==" -ForegroundColor Cyan

# Common uninstall registry paths
$uninstallRoots = @(
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"
)

# Remove uninstall entries that match Chrome / Google Update
foreach ($root in $uninstallRoots) {
    if (Test-Path $root) {
        Get-ChildItem $root | ForEach-Object {
            $item = Get-ItemProperty $_.PSPath
            $name = $item.DisplayName
            $publisher = $item.Publisher
            $uninstallString = $item.UninstallString
            $quietUninstallString = $item.QuietUninstallString

            if (
                ($name -like "Google Chrome*") -or
                ($name -like "*Chrome Enterprise*") -or
                ($name -like "*Google Update*") -or
                ($publisher -like "Google*")
            ) {
                Write-Host "Removing uninstall key: $($_.PSChildName) / $name" -ForegroundColor Yellow
                Remove-Item $_.PSPath -Recurse -Force
            }
        }
    }
}

Write-Host "== Removing Installer product references mentioning Chrome ==" -ForegroundColor Cyan

# Installer\UserData uninstall/product references
$installerRoots = @(
    "HKLM:\SOFTWARE\Classes\Installer\Products",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Installer\UserData\S-1-5-18\Products",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Installer\UserData\S-1-5-18\Components"
)

foreach ($root in $installerRoots) {
    if (Test-Path $root) {
        Get-ChildItem $root | ForEach-Object {
            $props = Get-ItemProperty $_.PSPath
            $text = ($props | Out-String)
            if ($text -match "Google Chrome|Chrome Enterprise|Google Update") {
                Write-Host "Removing installer key: $($_.PSPath)" -ForegroundColor Yellow
                Remove-Item $_.PSPath -Recurse -Force
            }
        }
    }
}

Write-Host "== Removing Chrome / Google registry keys ==" -ForegroundColor Cyan
$regPaths = @(
    "HKLM:\SOFTWARE\Google",
    "HKLM:\SOFTWARE\WOW6432Node\Google",
    "HKCU:\SOFTWARE\Google",
    "HKLM:\SOFTWARE\Policies\Google",
    "HKCU:\SOFTWARE\Policies\Google"
)

foreach ($rp in $regPaths) {
    if (Test-Path $rp) {
        Write-Host "Removing $rp" -ForegroundColor Yellow
        Remove-Item $rp -Recurse -Force
    }
}

Write-Host "== Removing Chrome / Google files ==" -ForegroundColor Cyan
$filePaths = @(
    "$env:ProgramFiles\Google\Chrome",
    "${env:ProgramFiles(x86)}\Google\Chrome",
    "$env:ProgramFiles\Google\Update",
    "${env:ProgramFiles(x86)}\Google\Update",
    "$env:ProgramData\Google",
    "$env:LocalAppData\Google\Chrome",
    "$env:LocalAppData\Google\Update",
    "$env:AppData\Google\Chrome"
)

foreach ($fp in $filePaths) {
    if (Test-Path $fp) {
        Write-Host "Removing $fp" -ForegroundColor Yellow
        cmd /c "rmdir /s /q `"$fp`"" | Out-Null
    }
}

Write-Host "== Removing Start Menu shortcuts ==" -ForegroundColor Cyan
$shortcutPaths = @(
    "$env:ProgramData\Microsoft\Windows\Start Menu\Programs\Google Chrome.lnk",
    "$env:AppData\Microsoft\Windows\Start Menu\Programs\Google Chrome.lnk",
    "$env:Public\Desktop\Google Chrome.lnk",
    "$env:UserProfile\Desktop\Google Chrome.lnk"
)
foreach ($sp in $shortcutPaths) {
    if (Test-Path $sp) {
        Remove-Item $sp -Force
    }
}

Write-Host "== Optional: removing Google Update services from SCM if present ==" -ForegroundColor Cyan
sc.exe delete gupdate | Out-Null
sc.exe delete gupdatem | Out-Null

Write-Host ""
Write-Host "Chrome nuke pass complete." -ForegroundColor Green
Write-Host "REBOOT before reinstalling an older version." -ForegroundColor Green
```


----------------------------------------------------------

# same helper functions as Chrome version above
```
$path = Resolve-BrowserPath -ExeName 'msedge.exe' -DisplayRegex '^(Microsoft Edge)(\s|$)' -CommonPaths @(
    "$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe",
    "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe",
    "$env:LocalAppData\Microsoft\Edge\Application\msedge.exe"
)

if (-not $path) { Write-Output 'NONE'; exit 0 }

Write-Output (Get-PeArchitecture -Path $path)
exit 0
```
---------------------------------------------------------

```
function Get-PeArchitecture {
    param([Parameter(Mandatory)][string]$Path)
    $fs = [System.IO.File]::Open($Path, 'Open', 'Read', 'ReadWrite')
    try {
        $br = New-Object System.IO.BinaryReader($fs)
        $fs.Seek(0x3C, [System.IO.SeekOrigin]::Begin) | Out-Null
        $peOffset = $br.ReadInt32()
        $fs.Seek($peOffset + 4, [System.IO.SeekOrigin]::Begin) | Out-Null
        $machine = $br.ReadUInt16()
        switch ($machine) {
            0x014c { '32' }
            0x8664 { '64' }
            default { 'NONE' }
        }
    } finally { $fs.Dispose() }
}

function Get-DisplayIconPath {
    param([string]$DisplayIcon)
    if ([string]::IsNullOrWhiteSpace($DisplayIcon)) { return $null }
    $icon = $DisplayIcon.Trim()
    if ($icon -match '^\s*"([^"]+)"') { return $matches[1] }
    return ($icon -split ',')[0].Trim()
}

function Resolve-BrowserPath {
    $candidates = New-Object System.Collections.Generic.List[string]

    foreach ($key in @(
        "Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\firefox.exe",
        "Registry::HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\firefox.exe"
    )) {
        try {
            $item = Get-ItemProperty -Path $key -ErrorAction Stop
            $path = $item.'(default)'
            if (-not $path) { $path = $item.Path }
            if ($path -and (Test-Path -LiteralPath $path)) { $candidates.Add($path) }
        } catch {}
    }

    foreach ($root in @(
        'Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'Registry::HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'Registry::HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'Registry::HKEY_CURRENT_USER\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
    )) {
        try {
            Get-ItemProperty -Path $root -ErrorAction SilentlyContinue | ForEach-Object {
                if ($_.DisplayName -and ($_.DisplayName -match '^(Mozilla Firefox|Firefox)( ESR)?(\s|$)')) {
                    $path = $null
                    if ($_.InstallLocation) {
                        $tryExe = Join-Path $_.InstallLocation 'firefox.exe'
                        if (Test-Path -LiteralPath $tryExe) { $path = $tryExe }
                    }
                    if (-not $path -and $_.DisplayIcon) {
                        $iconPath = Get-DisplayIconPath -DisplayIcon $_.DisplayIcon
                        if ($iconPath -and (Test-Path -LiteralPath $iconPath)) { $path = $iconPath }
                    }
                    if ($path) { $candidates.Add($path) }
                }
            }
        } catch {}
    }

    foreach ($path in @(
        "$env:ProgramFiles\Mozilla Firefox\firefox.exe",
        "${env:ProgramFiles(x86)}\Mozilla Firefox\firefox.exe",
        "$env:LocalAppData\Mozilla Firefox\firefox.exe"
    )) {
        if ($path -and (Test-Path -LiteralPath $path)) { $candidates.Add($path) }
    }

    $seen = @{}
    foreach ($p in $candidates) {
        if (-not $seen.ContainsKey($p)) {
            $seen[$p] = $true
            return $p
        }
    }
    return $null
}

$path = Resolve-BrowserPath
if (-not $path) { Write-Output 'NONE'; exit 0 }

Write-Output (Get-PeArchitecture -Path $path)
exit 0
```



