<img width="522" height="288" alt="image" src="https://github.com/user-attachments/assets/aebcee1b-53eb-45dd-ae16-485b6eff7883" />---
layout: default
title: "TryHackMe: Snapped Phish-ing Line Walkthrough"
description: "Detailed solution and walkthrough for the Snapped Phish-ing Line challenge on TryHackMe."

redirect_from: 
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line.html
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line
---


# Sysmon + Wazuh: Real EDR Telemetry Generation
<img width="1903"  alt="header" src="https://github.com/user-attachments/assets/3cd4145c-8211-42ce-88f9-a5e57fbf2d84" />
### In this lab, Sysmon, Wazuh, Wazuh agent is used to catch populater mitrea and attack techniches and set them an elert using EDR telementry to catch them easily 

## Setting Up the Wazuh manager on (ubuntu server)

#### In this lab, ubuntu 22.04 is used with 4 GB and 2 cores and 50 GB 
after making the machine in vmbox, the wazuh manager is downloaded with the command:
```shell
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
<img width="872" height="166" alt="Screenshot 2026-05-16 221446" src="https://github.com/user-attachments/assets/a36d2dc4-fe5d-4a78-afeb-153346476e04" />

it should give the user and password in the end with a message "Installation finished"
<img width="956" height="62" alt="Screenshot 2026-05-16 231425" src="https://github.com/user-attachments/assets/b2c99cfc-2501-43ca-bd3f-6a567d8b7bda" />

to check that its running, these command are typed: 
```shell
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```
They both must have the active (running)
<img width="937" height="160" alt="image" src="https://github.com/user-attachments/assets/b73d4396-12e1-4f0e-94da-124aa3465d72" />

Now the Wazuh dashboard could be run on any browser from the same network, run it on the host machine using  https://YOUR_UBUNTU_IP from your host browser.
The ubuntu ip could be known using the command 
```shell
ip a
```
<img width="855" height="226" alt="image" src="https://github.com/user-attachments/assets/edf2dd3d-7bac-42c2-8308-d4fbd7a5e27e" />



now veriying that the wazuh manager is running
```shell
sudo /var/ossec/bin/wazuh-control status
```
<img width="522" height="288" alt="image" src="https://github.com/user-attachments/assets/18dbfd60-fdb2-45a0-a9a7-6df130ce4cfc" />

Ubuntu ip is `192.168.1.7` in this case, browsing the web browser with `https://192.168.1.7` after waiting for 3 mins for the wazuh dashboard to start would reaveal the login page 
<img width="1856" height="936" alt="Screenshot 2026-05-16 231542" src="https://github.com/user-attachments/assets/35e98f38-df53-46fd-89a4-67ff219277b2" />
 Using the credentials from the first wazuh installation to access the dashboard, it shows 0 total agents.
 <img width="1893" height="920" alt="Screenshot 2026-05-16 234013" src="https://github.com/user-attachments/assets/abd52b90-cea4-4099-895d-e079de91a735" />
 Making sure that al services are running using the command:
 ```shell
sudo /var/ossec/bin/wazuh-control status
```
<img width="855" height="226" alt="image" src="https://github.com/user-attachments/assets/913d96b8-9a89-4654-93d4-b1f2a0e57360" />

Now comes the time to set the agent that would be monitored 


### Setting up Sysmon (windows vm)

For this lab a windows vm with 4GB, 2 cores and 50 GB of starage is used
First step is to install sysmon, its an advanced Windows logging tool that hooks deep into the system to monitor detailed activity. Sysmon is needed because the normal windows logging are too basic and miss advanced hooking techniques, so sysmon is installed to discover the details of logs and reveal command line arguments, process tracking and memory access that would be tested through the lab. 

On the windows vm, browse the following link to download sysmon: 
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

<img width="885" height="577" alt="Screenshot 2026-05-16 185939" src="https://github.com/user-attachments/assets/6aedaa2c-30d3-42a9-8d7a-f215bf68f318" />
Sysmon is like a massive high defination security camera, it collects everything heppening in the PC, its its turned on and left without any instruction, it will collect everything happening on the computer. That will make alot of noise to the logging. As a soc analyst, one of the best practices is to tune the signal to noise ratio, using an xml file that directs the sysmon to collect only the suspicous pattern and process and set basic alerts for it in sysmon
This is where the The SwiftOnSecurity Config comes in
The SwiftOnSecurity Config is downloaded through the link
https://github.com/SwiftOnSecurity/sysmon-config
<img width="887" height="585" alt="Screenshot 2026-05-16 190051" src="https://github.com/user-attachments/assets/1ad8d88b-a696-45a3-b7cb-f6f2cb1682e7" />

The downloaded file is then placed in the same extracted sysmon file
<img width="912" height="369" alt="Screenshot 2026-05-16 191251" src="https://github.com/user-attachments/assets/57f1ae3b-14a7-4b19-8c8d-78455588396f" />

After setting the required files, sysmon is installed through the following command  
```cmd
cd C:\Users\<current user>\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
<img width="879" height="373" alt="Screenshot 2026-05-16 191234" src="https://github.com/user-attachments/assets/6ba23822-c080-4ac7-bef3-89a13a4503ce" />

Verifying the sysmon actually records the intendet event, the following command is run in powershell 
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
  Select TimeCreated, Id, Message |
  Format-List
```
where It pulls the 10 most recent logs directly from the Sysmon event database into your terminal, showing only the time, Event ID, and details.

<img width="877" height="484" alt="Screenshot 2026-05-16 191404" src="https://github.com/user-attachments/assets/103732bf-d0b0-4a65-8391-42d5a031753d" />

Opening the Event viewer -> Applications and Services Logs -> Microsoft -> Windows -> Sysmon -> Operational 
<img width="793" height="588" alt="image" src="https://github.com/user-attachments/assets/62b50642-140f-4703-920a-3dc969eb3a9a" />
Event ID 1 (process creates) and EID 3 (network connections)
where in the command 

Process Creation (Event ID 1): The Windows kernel spins up powershell.exe. Sysmon's driver catches this instantly and records the timestamp, the user account, and the exact command-line arguments used.

Network Connection (Event ID 3): A split second later, powershell.exe sends out a network packet to connect to an external server. Sysmon's network monitor hooks into the network stack, catches the traffic, and logs the source/destination IPs and port

The script blocking (T1059 Command and Scripting Interpreter) have to be catched as well in this lab, so the script block logging is enaled through  the command
```powershell 
$p = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $p -Force | Out-Null
Set-ItemProperty -Path $p -Name "EnableScriptBlockLogging" -Value 1
Set-ItemProperty -Path $p -Name "EnableScriptBlockInvocationLogging" -Value 1
```
<img width="888" height="148" alt="Screenshot 2026-05-16 192142" src="https://github.com/user-attachments/assets/29a4f305-3f63-45f1-bba8-d14142393295" />

Another important logs to be collected in the process command-line auditing, where the following command enables full command-line strings in Event ID 4688. Without this command, only the process name will be seen without the arguments which means missing the -EncodedCommand flag in Powershell attacks. 

```powershell
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```
<img width="890" height="110" alt="Screenshot 2026-05-16 211026" src="https://github.com/user-attachments/assets/f8c936dd-8d31-4363-8c2f-684312e18d97" />

### Setting up Wazuh Agent (windows vm)

After setting the Sysmon and testing its detection and coustimizing it for the logs wanted to be moniitoring, comes the step where all the logs are forwarded to the Wazuh manager for analysis 

Installing the Wazuh agent
The wazuh agent to be installed must be the same version of the wazuh manager installed in the ubuntu server. In the ubuntu server, Wazuh 4.7.5 is installed, so downloading wazuh agent 4.7.5 through this command
```cmd
curl -L https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi -o wazuh-agent-4.7.5-1.msi   
```

now its configured with the server IP and started through the commands 
```cmd
powershell -Command "(Get-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf') -replace '<address>0.0.0.0</address>', '<address>192.168.1.7</address>' | Set-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf'"
del "C:\Program Files (x86)\ossec-agent\client.keys" 2>nul
net start WazuhSvc
```
<img width="789" height="400" alt="Screenshot 2026-05-17 001756" src="https://github.com/user-attachments/assets/d4d2a7e7-44ed-44b8-b806-9216230887df" />

It shows the STATE is RUNNING, which is a right indicator that the agent is set up
Now the agent sends normal windows logs, it has to be configured to forward sysmon logs as well. 

Browsing C:\Program Files (x86)\ossec-agent\ossec.conf file and opening it as a text editor, the instructions of forwarding the sysmon logs as well is put into that file 
```texteditor 
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Microsoft-Windows-PowerShell/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
```
<img width="803" height="563" alt="Screenshot 2026-05-17 002002" src="https://github.com/user-attachments/assets/c3c8ec91-a41c-494f-af6d-3d9a5ef3bc0c" />

Restarting the agent again would apply the changes made 
```cmd
NET STOP WazuhSvc
NET START WazuhSvc
```


### Adding rules to Wazuh server 

For making  the detection more efficient and fast, some rules need to be made in Wazuh 
like a rule to discover an LSASS Memory Access, CreateRemoteThread Injection or even Registry run key modification

For each one of these a rule will be made and explained in details how to implement the rule 
First, the LSASS Memory which is an abbreviation for *(Local Security Authority Subsystem Service)* where the user sign in credentials are cached for single sign on prepvention the repetition of password prompts
This technique is mapped in MITRE ATT&CK as (MITRE T1003.001) as credential dumping. The attacker accesses this memory to extract hashed or plaintext user passwords, then it hast to have the highest severity level (15).

```xml
<rule id="100010" level="15">
  <if_group>sysmon_event10</if_group>
  <field name="win.eventdata.targetImage" type="pcre2">(?i)lsass\.exe</field>
  <description>Sysmon - LSASS memory access detected (T1003.001)</description>
  <mitre>
    <id>T1003.001</id>
  </mitre>
  <group>credential_access,</group>
</rule>
```

The rule detects this method efficiently as set a high alert level to it, breaking this rule to simplify how it works:
`<rule id="100010" level="15">` -> this line sets and ID for the rule and give it an alert level 
`<if_group>sysmon</if_group> <field name="win.system.eventID">^10$</field>` -> This line the log type and Event ID of the process to be detected which in this case sysmon even 10 which detects a process acceess
`<field name="win.eventdata.targetImage" type="pcre2">(?i)lsass\.exe</field>` -> the field name spacifies the process that is being accessed and compare its name if its `lsass.exe` or not, note that also type="pcre2" attribute is used to give more options to compare. The attacker could write it as LSASS.exe and it will still be detected due to the (?i) option which ignores the case senstivity used along the prec2 type 
The rest are more information about the rule for detecting and readability like a description and the MITRE ATT&CK and the group it belongs to 

For the Second rule is for detecting CreateRemoteThread Injection which is mapped as Process Injection (MITRE T1055) in the MITRE ATT&CK framework Malware uses this technique to hide from security tools. Instead of running a suspicious .exe file on disk, the malware injects its malicious code directly into the active memory of a trusted Windows process (like explorer.exe or svchost.exe). The trusted process does the dirty work, making the attack look like legitimate system activity. This rule watches event ID 8 where this event forces a compelete seperate innocent program to execute a command on its behalf to seem legimitate, it should have a high severity (12). 

```xml
<rule id="100011" level="12">
  <if_group>sysmon</if_group>
  <field name="win.system.eventID">^1$</field>
  <field name="win.eventdata.parentImage" type="pcre2">(?i)cmd\.exe</field>
  <field name="win.eventdata.image" type="pcre2">(?i)powershell\.exe</field>
  <description>Sysmon - Suspicious cmd.exe spawning powershell.exe (T1059.003)</description>
  <mitre>
    <id>T1059.003</id>
  </mitre>
  <group>execution, lolbin,</group>
</rule>
```
event id 1 is utilized to analyze new process creations.
the parentImage field is checked to verify cmd.exe initiated the action.
the image field is checked to verify powershell.exe was the resulting child process.
PCRE2 is applied so the rule triggers regardless of folder paths or capitalization.

For the third rule, Registry Run Key Modification, it detects if there are any registry key changed, added, or even removed from the registery editor. Which is mapped in MITRE ATT&CK as Registry Run Keys / Startup Folder Persistence (MITRE T1547.001). This rule will monitor the sysmon event ID 13 (Registry Value Set). When an attacker gains an access to a machine, he want to keep presistent. So a they add a path to the malware in the windowns run registry key to force the PC to run the malware everytime the user logs into the machine. 

```xml
<rule id="100012" level="10">
    <if_group>sysmon</if_group>  <field name="win.system.eventID">^13$</field> <field name="win.eventdata.targetObject" type="pcre2">(?i)\\CurrentVersion\\Run</field>
    <description>Sysmon - Registry Run key persistence (T1547.001)</description>
    <mitre>
      <id>T1547.001</id>
    </mitre>
    <group>persistence,</group>
  </rule>
```
`<rule id="100012" level="10">` -> the severity of this rule is high but lower than the prevoius rules, level is set to 10. 
`<if_group>sysmon</if_group>  <field name="win.system.eventID">^13$</field> <field` -> selecting the source of logs as sysmon specifically for the event 13 that detects any changes in the registry editor 
` <field name="win.eventdata.targetObject" type="pcre2">(?i)CurrentVersion.*Run</field>` -> When a registry key is modifies, the exact registry path modified is recorded in targetObject. the path CurrentVersion.*Run is used to decide which application and programs will run when the PC is powered on and the user logs in, so putting a malware in this registry path will execute it everytime the PC runs. Using this as a reference to detect if there is any attacker trying to gain presistance to the machine, would alert and reduce any presistance chances, specially with the wildcard `.*` to include all variation of baths in run and (?i) along with prec2 to ignore case sensitivity

Now after setting the 3 main roles, the local_rules.xml file is updated with them to be included in wazuh alerting 
```shell
sudo nano /var/ossec/etc/rules/local_rules.xml
```
Add the 3 rules to the xml file
```xml
<group name="sysmon,attack,">

  <rule id="100010" level="15">
    <if_group>sysmon</if_group>
    <field name="win.eventdata.targetImage" type="pcre2">(?i)lsass\.exe</field>
    <description>Sysmon - LSASS memory access detected (T1003.001)</description>
    <mitre>
      <id>T1003.001</id>
    </mitre>
    <group>credential_access,</group>
  </rule>

  <rule id="100011" level="12">
    <if_group>sysmon</if_group>
    <description>Sysmon - CreateRemoteThread detected (T1055 Process Injection)</description>
    <mitre>
      <id>T1055</id>
    </mitre>
    <group>process_injection,</group>
  </rule>

  <rule id="100012" level="10">
    <if_group>sysmon</if_group>
    <field name="win.system.eventID">^13$</field>
    <field name="win.eventdata.targetObject" type="pcre2">(?i)CurrentVersion.*Run</field>
    <description>Sysmon - Registry Run key persistence (T1547.001)</description>
    <mitre>
      <id>T1547.001</id>
    </mitre>
    <group>persistence,</group>
  </rule>

</group>
```
then hit Ctrl+O then Enter to save 
and Ctrl+x to close
<img width="973" height="653" alt="image" src="https://github.com/user-attachments/assets/ac166270-3f67-4c10-b151-dd9cf245c377" />

Then the wazuh control has to be restarted to apply the rules
```shell
sudo /var/ossec/bin/wazuh-control restart
```
<img width="975" height="389" alt="Screenshot 2026-05-17 002357" src="https://github.com/user-attachments/assets/bf902f89-86fa-4546-a34d-c3ee47e89d33" />

### Firing the attacks & confirming detections
Now its time for the most interesting part, the attacking and detecting where work done is tested

#### Attack 1 (LSASS memory access (T1003.001) — PowerShell, run as Administrator)
To test the lsass memory access, the following commands are fired 
```powershell
$lsass = Get-Process lsass
$handle = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(1)
Write-Host "LSASS PID: $($lsass.Id) — Sysmon EID 10 should fire now"
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $($lsass.Id) C:\Windows\Temp\lsass_dump.dmp full
```
breaking it up to lines 
`$lsass = Get-Process lsass`  -> this script queries the operating system to find an active instance of lsass and store it in variable `$lsass` 
`$handle = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(1)` -> The `$handle` line is used strictly as an evasion tactic to trick basic antivirus software. By forcing the script to perform low-level memory allocation, the attacker attempts to make the file look like a legitimate administrative tool rather than malware.
`Write-Host "LSASS PID: $($lsass.Id) — Sysmon EID 10 should fire now"` -> As an analyst, this serves as your timestamp marker.
`rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $($lsass.Id) C:\Windows\Temp\lsass_dump.dmp full` -> This step is where the attacker is getting the info. It uses a built in windows utility rundl32.exe to run legitimate process comsvcs.dll and use its MiniDump option to store the full data of the process of the variable `$lsass` in  `C:\Windows\Temp\lsass_dump.dmp` path 

<img width="792" height="559" alt="Screenshot 2026-05-17 002622" src="https://github.com/user-attachments/assets/2059d563-961d-4be0-8a7e-195c6fae5515" />

windows defender could detect it and prevent the process, but if it didn't 
the process is detected and alerted in Wazuh by the rule id `100010`
```DQL
rule.id: 100010
```
<img width="1903" height="337" alt="image" src="https://github.com/user-attachments/assets/eac44c77-4b31-4e19-b985-2ffa25ca2bd3" />
<img width="1887" height="328" alt="image" src="https://github.com/user-attachments/assets/84b1f9c1-9faa-49d1-9564-910ab8c01354" />


#### Attack 2 — Registry Run key persistence (T1547.001)
For the registry keys, the rule that detects the change of windows run registery keys are tested by this command 
```powershell 
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "LabPersistence" /t REG_SZ /d "C:\Windows\System32\calc.exe" /f
Write-Host "Run key added — Sysmon EID 13 should fire"
```
Where the command adds a registry key to the path `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` which is a run registry path 

This could be detected when filtering with the rule ID 100011
```DQL
rule.id: 100012
```
<img width="1898" height="339" alt="image" src="https://github.com/user-attachments/assets/1d0649ab-7909-4d75-b8db-056e8207b125" />
<img width="1890" height="859" alt="image" src="https://github.com/user-attachments/assets/ad3f553c-2220-425b-8cfe-494956688b78" />


#### Attack 3 — encoded PowerShell execution (T1059.001)
From the Script block logging option enabled while installing the sysmon, the script block logging event ID is 4104
```powershell
$cmd = "Write-Host 'Simulated encoded payload executed — T1059.001'"
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell.exe -NonInteractive -EncodedCommand $encoded
```
<img width="789" height="80" alt="image" src="https://github.com/user-attachments/assets/5ba0ee87-8677-4762-9cbe-cdb34203dbfa" />

`$cmd = "Write-Host 'Simulated encoded payload executed — T1059.001'` -> As an analyst, this serves as your timestamp marker.
`$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))` The plain text string is translated into a scrambled format known as Base64.
`powershell.exe -NonInteractive -EncodedCommand $encoded` A new, hidden PowerShell session is launched. The -NonInteractive flag is used to ensure no pop-up windows alert the user. Finally, the -EncodedCommand flag is utilized to pass the scrambled Base64 string directly into memory, where it is decoded and executed by the system

event id 1 is used to only look at when new processes are created.
AND is added so both conditions are forced to match.
the commandline field is checked for the exact text typed.
wildcards (*) are placed around EncodedCommand so the attack is caught if the evasion flag is hidden anywhere in the full string. 
```DQL
data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *EncodedCommand*
```
<img width="1903" height="336" alt="image" src="https://github.com/user-attachments/assets/f70e65f5-5cc4-4c30-b0a6-3690ea37c18a" />
<img width="1874" height="838" alt="image" src="https://github.com/user-attachments/assets/24a04095-b46a-45a2-aa31-522fa62cebe0" />

### Attack 4 — suspicious child process spawn (T1059 / parent-child anomaly)
For the suspicious child process spawn, the rule that detects the anomalous parent-child relationship between built-in Windows binaries (LOLBins) is tested by this command
```powershell
cmd.exe /c "powershell.exe -NonInteractive -WindowStyle Hidden -Command Get-LocalUser"
```
Where the command utilizes the standard Windows command shell (cmd.exe) as a parent to secretly spawn a hidden PowerShell child process (powershell.exe) in order to enumerate local user accounts.

This could be detected when filtering with the edited rule ID 100011 (or by explicitly searching the EID 1 parent and child image fields)
```DQL
rule.id: 100012
```
<img width="1891" height="917" alt="image" src="https://github.com/user-attachments/assets/cb5b9938-9815-4434-b99a-c10fc213b9fe" />







