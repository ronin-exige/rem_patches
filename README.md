'''function Get-PeArchitecture {
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
exit 0'''

----------------------------------------------------------

'''# same helper functions as Chrome version above

$path = Resolve-BrowserPath -ExeName 'msedge.exe' -DisplayRegex '^(Microsoft Edge)(\s|$)' -CommonPaths @(
    "$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe",
    "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe",
    "$env:LocalAppData\Microsoft\Edge\Application\msedge.exe"
)

if (-not $path) { Write-Output 'NONE'; exit 0 }

Write-Output (Get-PeArchitecture -Path $path)
exit 0'''

---------------------------------------------------------

'''function Get-PeArchitecture {
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
exit 0'''




