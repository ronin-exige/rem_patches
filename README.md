Absolutely. Here’s a practical Ivanti build sheet for a MECM / WMI Repair SAFE module, using mostly native actions and only a few short PowerShell steps. It follows the safe flow from your playbook: stop ccmexec, iphlpsvc, and Winmgmt; verify/salvage WMI; recompile MOFs; restart services; verify again; then run MECM client repair.

Module name

MECM - WMI Repair SAFE

General notes

Run as SYSTEM

Do not use this module to do a full repository delete/reset

Treat it as a safe repair module only

For PowerShell tasks, use file extension ps1

Success code for PowerShell tasks: 0

Failure code: anything non-zero

For PowerShell tasks that are only informational checks, you can choose whether failure should stop the module or just log and continue



---

Task list

Task 1

Type: Query Service Properties
Name: Query Winmgmt
Purpose: confirm WMI service exists

Settings

Filter by service name:


Winmgmt

Failure handling

Continue on failure



---

Task 2

Type: Query Service Properties
Name: Query CcmExec
Purpose: confirm MECM client service exists

Settings

Filter by service name:


CcmExec

Failure handling

Continue on failure



---

Task 3

Type: Files
Name: Check CCM executable
Purpose: verify MECM client binaries exist

Settings

Path / file to query:


%SystemRoot%\CCM\CcmExec.exe

Failure handling

Continue on failure



---

Task 4

Type: Files
Name: Check WMI repository folder
Purpose: verify repository folder exists

Settings

Path / file to query:


%SystemRoot%\System32\wbem\Repository

Failure handling

Continue on failure



---

Task 5

Type: Windows PowerShell Script
Name: Verify WMI before repair
Purpose: basic health gate

Script

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
    $verify = (winmgmt /verifyrepository 2>&1) -join ' '
    Write-Host "Verify result: $verify"

    if ($verify -match 'consistent|is consistent') {
        Write-Host "WMI basic query OK and repository consistent"
        exit 0
    }
    else {
        Write-Host "Repository inconsistent"
        exit 1
    }
}
catch {
    Write-Host "WMI basic query failed: $($_.Exception.Message)"
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure
This is a pre-check, so I would not stop the module here.



---

Task 6

Type: Service Properties
Name: Stop CcmExec
Purpose: stop MECM client dependency before WMI work

Settings

Service name:


CcmExec

Action: Stop service


Failure handling

Continue on failure



---

Task 7

Type: Service Properties
Name: Stop IP Helper
Purpose: stop dependency used in your playbook flow

Settings

Service name:


iphlpsvc

Action: Stop service


Failure handling

Continue on failure



---

Task 8

Type: Service Properties
Name: Stop Winmgmt
Purpose: stop WMI before salvage/repair

Settings

Service name:


Winmgmt

Action: Stop service


Failure handling

Continue on failure



---

Task 9

Type: Windows PowerShell Script
Name: Salvage WMI repository
Purpose: safe repository repair

Script

$verifyBefore = (winmgmt /verifyrepository 2>&1) -join ' '
Write-Host "Verify before salvage: $verifyBefore"

winmgmt /salvagerepository | Out-Null
Start-Sleep -Seconds 10

$verifyAfter = (winmgmt /verifyrepository 2>&1) -join ' '
Write-Host "Verify after salvage: $verifyAfter"

if ($verifyAfter -match 'consistent|is consistent') {
    exit 0
}
else {
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure
I would still continue, because Task 10 and later steps may still help even if salvage does not fully clean it up.



---

Task 10

Type: Windows PowerShell Script
Name: Recompile WMI MOFs
Purpose: rebuild MOF registrations as in the playbook

Script

$mofDir = "$env:SystemRoot\System32\wbem"

Get-ChildItem $mofDir -Filter "*.mof" -ErrorAction SilentlyContinue | ForEach-Object {
    mofcomp.exe $_.FullName | Out-Null
}
Get-ChildItem $mofDir -Filter "*.mfl" -ErrorAction SilentlyContinue | ForEach-Object {
    mofcomp.exe $_.FullName | Out-Null
}

Write-Host "MOF recompilation complete"
exit 0

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 11

Type: Service Properties
Name: Start Winmgmt
Purpose: bring WMI back up

Settings

Service name:


Winmgmt

Action: Start service


Failure handling

Stop on failure
This one matters.



---

Task 12

Type: Service Properties
Name: Start CcmExec
Purpose: bring MECM client back up

Settings

Service name:


CcmExec

Action: Start service


Failure handling

Continue on failure
You may want this to continue so later checks can show you what failed.



---

Task 13

Type: Windows PowerShell Script
Name: Verify WMI after repair
Purpose: confirm WMI is queryable after repair

Script

try {
    Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Out-Null
    $verify = (winmgmt /verifyrepository 2>&1) -join ' '
    Write-Host "Post-repair verify: $verify"

    if ($verify -match 'consistent|is consistent') {
        exit 0
    }
    else {
        exit 1
    }
}
catch {
    Write-Host "Post-repair WMI query failed: $($_.Exception.Message)"
    exit 1
}

Ivanti settings

File extension: ps1


Failure handling

Continue on failure



---

Task 14

Type: Files
Name: Check ccmrepair exists
Purpose: confirm repair executable exists before launching it

Settings

Path / file to query:


%SystemRoot%\CCM\ccmrepair.exe

Failure handling

Continue on failure



---

Task 15

Type: Execute Program / File task
Name: Run ccmrepair
Purpose: trigger MECM client repair after WMI repair

Program / file

%SystemRoot%\CCM\ccmrepair.exe

Arguments

none


Success codes

0


Failure handling

Continue on failure
I would continue so you can still inspect services / logs afterward.



---

Task 16

Type: Query Service Properties
Name: Verify CcmExec after repair
Purpose: quick confirmation MECM service is back

Settings

Filter by service name:


CcmExec

Failure handling

Continue on failure



---

Recommended task order summary

1. Query Winmgmt


2. Query CcmExec


3. Check CcmExec.exe


4. Check WMI repository folder


5. Verify WMI before repair


6. Stop CcmExec


7. Stop IP Helper


8. Stop Winmgmt


9. Salvage WMI repository


10. Recompile WMI MOFs


11. Start Winmgmt


12. Start CcmExec


13. Verify WMI after repair


14. Check ccmrepair.exe


15. Run ccmrepair.exe


16. Verify CcmExec after repair




---

Optional tiny improvements

Add a wait after starting Winmgmt

If you want one extra stability step, insert a PowerShell task between Task 11 and Task 12:

Name: Wait for WMI service settle

Start-Sleep -Seconds 10
exit 0

That is not required, but it can make the restart sequence a little less twitchy.

Add a wait after ccmrepair

If you want a cleaner post-check, insert another tiny PowerShell task after Task 15:

Start-Sleep -Seconds 20
exit 0


---

What not to put in this SAFE module

Do not add:

winmgmt /resetrepository

deleting %SystemRoot%\System32\wbem\Repository

blanket regsvr32 loops across every DLL in the folder


Those belong in a separate last resort module, not the safe one. Your playbook only escalates to reset when salvage still fails, and explicitly warns that full reset means the MECM client may need reinstall.


---

For your browser module follow-up PowerShell task

You were right — a single PowerShell action at the end is fine. Ivanti does not need anything exotic there beyond:

file extension ps1

success exit 0

nonzero on failure if you want the task to show failed


The main thing is to make sure your script ends with:

exit 0

on success, and exit 1 in the catch block if you want Ivanti to mark it failed.

If you want, I can do the same style of literal Ivanti build sheet for the MECM action-cycle follow-up task too.