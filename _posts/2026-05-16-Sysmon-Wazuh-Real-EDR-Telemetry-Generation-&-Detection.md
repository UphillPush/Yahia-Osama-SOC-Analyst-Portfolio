---
layout: post
title: "Sysmon + Wazuh: Real EDR Telemetry Generation"
description: "A comprehensive home lab walkthrough on generating, ingesting, and detecting real EDR telemetry using Sysmon and custom Wazuh rules."
date: 2026-05-17 00:00:00 +0200
categories: ["Home Lab", "Detection Engineering"]
tags: [wazuh, sysmon, edr, mitre-attack, blue-team, threat-hunting]
pin: false
image:
  path: https://github.com/user-attachments/assets/70331909-e494-4d6c-83da-9def30b5f9a2

  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Sysmon + Wazuh: Real EDR Telemetry Generation

![Lab Header](https://github.com/user-attachments/assets/9395b4fd-0fff-47e1-a6f7-943373e58679){: .shadow .rounded }
_Figure 1: EDR Telemetry Generation Lab._

> **Executive Summary**
> **Situation:** It is expected for most SOC analysts to know EDR telemetry, but entry roles rarely explain how the raw events actually look. This lab was built to generate real Sysmon telemetry from actual attack simulations targeting four MITRE ATT&CK techniques used in initial access and persistence. Wazuh detection logic was built entirely from scratch.
> 
> **Task:** A two-VM home lab was deployed. A Windows 10 target with Sysmon and a Wazuh agent was set up alongside an Ubuntu 22.04 server for the Wazuh manager. Sysmon telemetry ingestion was configured and custom Wazuh rules were written for each technique. The goal was to document the full chain from the attack execution to the analyst alert.
> 
> **Action:** Sysmon v15 was installed using the SwiftOnSecurity config so 29 event types were enabled, including EID 1, 8, 10, and 13. The Wazuh agent was configured to forward Sysmon, Security, and PowerShell operational logs. Command-line auditing was enabled via Auditpol to capture full arguments in EID 4688. Three custom Wazuh rules (100010-100012) were written targeting LSASS access, process injection, and run key persistence (T1003.001, T1055, and T1547.001). Each technique was simulated, and alerts were confirmed in the Wazuh dashboard within 15 seconds.
> 
> **Result:** All 4 simulated techniques successfully generated alerts within 15 seconds. Rule 100010 was triggered by the LSASS dump attempt at severity level 15, which is the highest in the lab. In a real SOC, this gets escalated to tier 2 for memory forensics. The registry persistence event would normally trigger an active response block in a production environment. The encoded PowerShell execution showed that EID 4104 script block logging catches attacks that EID 4688 alone misses, which is a major gap in Windows-only detection.
{: .prompt-info }

---

## Architecture Overview

To accurately simulate a production environment, the lab infrastructure was divided between an attacker/target endpoint and a centralized SIEM server.

* **The SIEM/EDR Manager (Ubuntu 22.04 LTS):** Allocated with 4 GB RAM, 2 CPU cores, and 50 GB of storage. This server hosts the Wazuh Manager and Dashboard, acting as the centralized brain for log ingestion and rule processing.
* **The Target Endpoint (Windows 10):** Allocated with 4 GB RAM, 2 CPU cores, and 50 GB of storage. This machine acts as the victim endpoint. It is heavily instrumented with Sysmon (for deep kernel-level telemetry) and the Wazuh Agent (to securely forward those logs).

---

## Phase 1: Setting Up the Wazuh Manager (Ubuntu Server)

After provisioning the Ubuntu virtual machine, the Wazuh manager was downloaded and installed using the automated installation script:

```shell
curl -sO [https://packages.wazuh.com/4.7/wazuh-install.sh](https://packages.wazuh.com/4.7/wazuh-install.sh)
sudo bash wazuh-install.sh -a
```
{: .nolineno }

![Wazuh Installation](https://github.com/user-attachments/assets/a36d2dc4-fe5d-4a78-afeb-153346476e04){: .shadow .rounded }

Once complete, the terminal outputs the administrative credentials alongside an "Installation finished" message.

![Wazuh Credentials](https://github.com/user-attachments/assets/b2c99cfc-2501-43ca-bd3f-6a567d8b7bda){: .shadow .rounded }

To verify that the core services were successfully initialized, the status of both the manager and the dashboard was checked. Both must display the `active (running)` state.

```shell
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```
{: .nolineno }

![Service Status](https://github.com/user-attachments/assets/b73d4396-12e1-4f0e-94da-124aa3465d72){: .shadow .rounded }

The Wazuh dashboard can be accessed from any browser on the same network. The local IP address of the Ubuntu server is required to navigate to the web interface.

```shell
ip a
```
{: .nolineno }

![Ubuntu IP](https://github.com/user-attachments/assets/edf2dd3d-7bac-42c2-8308-d4fbd7a5e27e){: .shadow .rounded }

Next, the Wazuh manager's internal status is verified to ensure all components are fully operational:

```shell
sudo /var/ossec/bin/wazuh-control status
```
{: .nolineno }

![Wazuh Control Status](https://github.com/user-attachments/assets/18dbfd60-fdb2-45a0-a9a7-6df130ce4cfc){: .shadow .rounded }

The Ubuntu IP was identified as `192.168.1.7`. After allowing approximately 3 minutes for the dashboard services to start, browsing to `https://192.168.1.7` revealed the login page.

![Wazuh Login](https://github.com/user-attachments/assets/35e98f38-df53-46fd-89a4-67ff219277b2){: .shadow .rounded }

Using the credentials generated during the initial installation, the dashboard was accessed. It currently shows 0 total agents, as expected.

![Wazuh Dashboard](https://github.com/user-attachments/assets/abd52b90-cea4-4099-895d-e079de91a735){: .shadow .rounded }

Checking the services once more confirms everything is running smoothly. 

![Services Check](https://github.com/user-attachments/assets/913d96b8-9a89-4654-93d4-b1f2a0e57360){: .shadow .rounded }

The server is now primed and ready for the target endpoint agent to be configured.

---

## Phase 2: Setting Up Sysmon (Windows VM)

The first step on the target machine is the installation of Sysmon. Sysmon is an advanced Windows logging tool that hooks deep into the system to monitor detailed activity. It is fundamentally required because normal Windows logging is too basic and misses advanced hooking techniques. Sysmon is installed to discover the granular details of logs, revealing command-line arguments, process tracking, and memory access that will be tested throughout the lab.

Sysmon was downloaded directly from the official Microsoft Sysinternals repository:
`https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon`

![Sysmon Download](https://github.com/user-attachments/assets/6aedaa2c-30d3-42a9-8d7a-f215bf68f318){: .shadow .rounded }

Sysmon acts like a massive high-definition security camera; it collects everything happening on the PC. If it is turned on and left without any instructions, it will log absolutely everything, creating an overwhelming amount of noise. As a SOC analyst, one of the best practices is to tune the signal-to-noise ratio using an XML file that directs Sysmon to collect only suspicious patterns and processes.

This is where the **SwiftOnSecurity Configuration** comes in. The configuration XML is downloaded from:
`https://github.com/SwiftOnSecurity/sysmon-config`

![SwiftOnSecurity Config](https://github.com/user-attachments/assets/1ad8d88b-a696-45a3-b7cb-f6f2cb1682e7){: .shadow .rounded }

The downloaded configuration file was placed in the same directory as the extracted Sysmon executable.

![Sysmon Folder](https://github.com/user-attachments/assets/57f1ae3b-14a7-4b19-8c8d-78455588396f){: .shadow .rounded }

With the required files in place, Sysmon was installed utilizing the custom XML configuration via the Command Prompt:

```cmd
cd C:\Users\<current user>\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
{: .nolineno }

![Sysmon Install Command](https://github.com/user-attachments/assets/6ba23822-c080-4ac7-bef3-89a13a4503ce){: .shadow .rounded }

To verify that Sysmon actually records the intended events, the following command was run in PowerShell. It pulls the 10 most recent logs directly from the Sysmon event database into the terminal, filtering for just the time, Event ID, and message details.

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
  Select TimeCreated, Id, Message |
  Format-List
```
{: .nolineno }

![Sysmon Verification](https://github.com/user-attachments/assets/103732bf-d0b0-4a65-8391-42d5a031753d){: .shadow .rounded }

Opening the Event Viewer (`Applications and Services Logs -> Microsoft -> Windows -> Sysmon -> Operational`) reveals the raw telemetry. 

![Event Viewer Sysmon](https://github.com/user-attachments/assets/62b50642-140f-4703-920a-3dc969eb3a9a){: .shadow .rounded }

Two critical Event IDs are immediately visible:
* **Process Creation (Event ID 1):** The Windows kernel spins up `powershell.exe`. Sysmon's driver catches this instantly and records the timestamp, the user account, and the exact command-line arguments used.
* **Network Connection (Event ID 3):** A split second later, `powershell.exe` sends out a network packet to connect to an external server. Sysmon's network monitor hooks into the network stack, catches the traffic, and logs the source and destination IPs and ports.

### Enabling Advanced Auditing Features

Script block execution (T1059 Command and Scripting Interpreter) must be caught in this lab. Therefore, PowerShell Script Block Logging was forcibly enabled via the registry:

```powershell 
$p = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $p -Force | Out-Null
Set-ItemProperty -Path $p -Name "EnableScriptBlockLogging" -Value 1
Set-ItemProperty -Path $p -Name "EnableScriptBlockInvocationLogging" -Value 1
```
{: .nolineno }

![Script Block Logging](https://github.com/user-attachments/assets/29a4f305-3f63-45f1-bba8-d14142393295){: .shadow .rounded }

Another vital log source to be collected is process command-line auditing. The following commands enable full command-line strings in Event ID 4688. Without this configuration, only the process name is visible without the arguments, meaning critical flags like `-EncodedCommand` would be missed entirely in PowerShell attacks.

```cmd
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```
{: .nolineno }

![Auditpol Configuration](https://github.com/user-attachments/assets/f8c936dd-8d31-4363-8c2f-684312e18d97){: .shadow .rounded }

---

## Phase 3: Setting Up the Wazuh Agent (Windows VM)

After tuning Sysmon and customizing the logs slated for monitoring, the logs must be forwarded to the Wazuh manager for analysis. 

The Wazuh agent installed must match the exact version of the Wazuh manager on the Ubuntu server. Since Wazuh 4.7.5 was installed on the server, the corresponding agent is downloaded:

```cmd
curl -L [https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi) -o wazuh-agent-4.7.5-1.msi   
```
{: .nolineno }

The agent was then configured with the Ubuntu server's IP address and started using PowerShell:

```powershell
(Get-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf') -replace '<address>0.0.0.0</address>', '<address>192.168.1.7</address>' | Set-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf'
del "C:\Program Files (x86)\ossec-agent\client.keys" 2>nul
net start WazuhSvc
```
{: .nolineno }

![Starting Wazuh Agent](https://github.com/user-attachments/assets/d4d2a7e7-44ed-44b8-b806-9216230887df){: .shadow .rounded }

The service output shows `STATE : RUNNING`, which is a positive indicator that the agent is successfully set up. 

By default, the agent only sends standard Windows logs. It must be explicitly configured to forward Sysmon logs. Opening the `C:\Program Files (x86)\ossec-agent\ossec.conf` file in a text editor, the instructions for forwarding the new operational channels are appended:

```xml 
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
{: .nolineno }

![Wazuh Config XML](https://github.com/user-attachments/assets/c3c8ec91-a41c-494f-af6d-3d9a5ef3bc0c){: .shadow .rounded }

The agent service is then restarted to apply the changes:

```cmd
NET STOP WazuhSvc
NET START WazuhSvc
```
{: .nolineno }

---

## Phase 4: Engineering Detection Rules on Wazuh

To make detection more efficient and rapid, custom logic must be engineered in Wazuh. Three specific rules were developed to discover LSASS Memory Access, CreateRemoteThread Injection, and Registry Run Key Modification.

### 1. LSASS Memory Access (Rule 100010)
LSASS (Local Security Authority Subsystem Service) caches user sign-in credentials for single sign-on, preventing the repetition of password prompts. Attackers access this memory to extract hashed or plaintext user passwords. Because of its critical nature, it is assigned the highest severity level (15).

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
{: .nolineno }

**Rule Breakdown:**
* `<rule id="100010" level="15">`: Sets the ID for the rule and establishes the maximum alert level.
* `<if_group>sysmon</if_group>` / `<field name="win.system.eventID">^10$</field>`: Selects the log type and isolates Event ID 10, which detects process access.
* `<field name="win.eventdata.targetImage" type="pcre2">(?i)lsass\.exe</field>`: Specifies the accessed process. The `type="pcre2"` attribute enables advanced regex, and `(?i)` ignores case sensitivity. Even if an attacker types `LSASS.exe`, it will still trigger the alert.

### 2. Process Injection / CreateRemoteThread (Rule 100011)
Malware utilizes this technique to hide from security tools. Instead of running a suspicious `.exe` file on disk, the malware injects its malicious code directly into the active memory of a trusted Windows process. The trusted process does the dirty work, making the attack appear as legitimate system activity. This rule is set to a high severity (12).

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
{: .nolineno }

**Rule Breakdown:**
* Event ID 1 is utilized to analyze new process creations.
* The `parentImage` field is checked to verify `cmd.exe` initiated the action.
* The `image` field is checked to verify `powershell.exe` was the resulting child process.
* PCRE2 is applied so the rule triggers regardless of folder paths or capitalization.

### 3. Registry Run Key Modification (Rule 100012)
This detects if any registry keys are changed, added, or removed. When an attacker gains access to a machine, they want to maintain persistence. They add a path to their malware in the Windows run registry key to force the PC to run the malware every time the user logs in.

```xml
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
```
{: .nolineno }

**Rule Breakdown:**
* `<rule id="100012" level="10">`: Severity is high, but lower than the previous rules.
* `<field name="win.system.eventID">^13$</field>`: Specifically isolates Sysmon Event 13, which detects changes in the registry editor.
* `<field name="win.eventdata.targetObject" type="pcre2">(?i)CurrentVersion.*Run</field>`: When a registry key is modified, the exact path is recorded here. Putting malware in this path executes it upon every startup. The wildcard `.*` ensures all variations of paths in the Run hive are included.

### Applying the Rules
The `local_rules.xml` file is opened and updated with these custom rules to include them in the Wazuh alerting engine:

```shell
sudo nano /var/ossec/etc/rules/local_rules.xml
```
{: .nolineno }

The XML block containing all three rules is pasted inside the `<group name="sysmon,attack,">` tag. The file is saved using `Ctrl+O` and closed with `Ctrl+X`.

![Saving Local Rules](https://github.com/user-attachments/assets/ac166270-3f67-4c10-b151-dd9cf245c377){: .shadow .rounded }

Finally, the Wazuh manager is restarted to load and apply the new logic:

```shell
sudo /var/ossec/bin/wazuh-control restart
```
{: .nolineno }

![Restarting Manager](https://github.com/user-attachments/assets/bf902f89-86fa-4546-a34d-c3ee47e89d33){: .shadow .rounded }

---

## Phase 5: Attack Simulations & Confirming Detections

Now comes the most interesting part: simulating the attacks and testing the detection engineering work.

### Attack 1: LSASS Memory Access (T1003.001)
To test LSASS memory access, a PowerShell session was run as Administrator, and the following commands were fired:

```powershell
$lsass = Get-Process lsass
$handle = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(1)
Write-Host "LSASS PID: $($lsass.Id) — Sysmon EID 10 should fire now"
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $($lsass.Id) C:\Windows\Temp\lsass_dump.dmp full
```
{: .nolineno }

**Script Breakdown:**
* `$lsass = Get-Process lsass` -> This queries the operating system to find an active instance of LSASS and stores it in a variable.
* `$handle = ...` -> This line is used strictly as an evasion tactic to trick basic antivirus software. By forcing the script to perform low-level memory allocation, the attacker attempts to make the payload look like a legitimate administrative tool rather than malware.
* `Write-Host...` -> As an analyst, this serves as a timestamp marker.
* `rundll32.exe...` -> This is where the attacker extracts the information. It uses a built-in Windows utility (`rundll32.exe`) to run a legitimate process (`comsvcs.dll`) and leverages its `MiniDump` option to store the full data of the LSASS process in a temporary `.dmp` file.

![LSASS Execution](https://github.com/user-attachments/assets/2059d563-961d-4be0-8a7e-195c6fae5515){: .shadow .rounded }

Windows Defender could potentially detect and prevent the process, but if bypassed, the activity is successfully detected and alerted in Wazuh by the custom rule.

```text
rule.id: 100010
```
{: .nolineno }

![LSASS Alert 1](https://github.com/user-attachments/assets/eac44c77-4b31-4e19-b985-2ffa25ca2bd3){: .shadow .rounded }
![LSASS Alert 2](https://github.com/user-attachments/assets/84b1f9c1-9faa-49d1-9564-910ab8c01354){: .shadow .rounded }

### Attack 2: Registry Run Key Persistence (T1547.001)
To test the persistence rule, a command was executed to forcibly add a malicious executable path into the Windows run registry key.

```powershell 
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "LabPersistence" /t REG_SZ /d "C:\Windows\System32\calc.exe" /f
Write-Host "Run key added — Sysmon EID 13 should fire"
```
{: .nolineno }

This modification is easily detected in the dashboard when filtering for the corresponding rule ID.

```text
rule.id: 100012
```
{: .nolineno }

![Registry Alert 1](https://github.com/user-attachments/assets/1d0649ab-7909-4d75-b8db-056e8207b125){: .shadow .rounded }
![Registry Alert 2](https://github.com/user-attachments/assets/ad3f553c-2220-425b-8cfe-494956688b78){: .shadow .rounded }

### Attack 3: Encoded PowerShell Execution (T1059.001)
Because Script Block Logging was enabled during the Sysmon installation, Event ID 4104 is heavily utilized here.

```powershell
$cmd = "Write-Host 'Simulated encoded payload executed — T1059.001'"
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell.exe -NonInteractive -EncodedCommand $encoded
```
{: .nolineno }

![Encoded PowerShell](https://github.com/user-attachments/assets/5ba0ee87-8677-4762-9cbe-cdb34203dbfa){: .shadow .rounded }

**Script Breakdown:**
* `$cmd = ...` -> Sets the timestamp marker.
* `$encoded = ...` -> The plain text string is translated into a scrambled format known as Base64.
* `powershell.exe -NonInteractive -EncodedCommand $encoded` -> A new, hidden PowerShell session is launched. The `-NonInteractive` flag ensures no pop-up windows alert the user. Finally, the `-EncodedCommand` flag is utilized to pass the scrambled Base64 string directly into memory, where it is decoded and executed by the system.

Using Dashboard Query Language (DQL), the event is located. Event ID 1 is utilized to look at new process creations, and wildcards `*` are placed around `EncodedCommand` so the attack is caught if the evasion flag is hidden anywhere in the full string.

```text
data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *EncodedCommand*
```
{: .nolineno }

![Encoded Alert 1](https://github.com/user-attachments/assets/f70e65f5-5cc4-4c30-b0a6-3690ea37c18a){: .shadow .rounded }
![Encoded Alert 2](https://github.com/user-attachments/assets/24a04095-b46a-45a2-aa31-522fa62cebe0){: .shadow .rounded }

### Attack 4: Suspicious Child Process Spawn (T1059)
To test the anomalous parent-child relationship between built-in Windows binaries (LOLBins), the following command is fired:

```powershell
cmd.exe /c "powershell.exe -NonInteractive -WindowStyle Hidden -Command Get-LocalUser"
```
{: .nolineno }

This command utilizes the standard Windows command shell (`cmd.exe`) as a parent to secretly spawn a hidden PowerShell child process (`powershell.exe`) in order to enumerate local user accounts. This is detected when filtering with the specific process injection rule engineered earlier.

```text
rule.id: 100011
```
{: .nolineno }

![Process Spawn Alert](https://github.com/user-attachments/assets/cb5b9938-9815-4434-b99a-c10fc213b9fe){: .shadow .rounded }

---

## MITRE ATT&CK Mapping Summary

| Tactic | Technique ID | Technique Name | Detection Source | Custom Rule ID |
| :--- | :--- | :--- | :--- | :--- |
| **Credential Access** | `T1003.001` | OS Credential Dumping: LSASS Memory | Sysmon EID 10 | `100010` |
| **Execution** | `T1059.003` | Command and Scripting: Windows Command Shell | Sysmon EID 1 | `100011` |
| **Execution** | `T1059.001` | Command and Scripting: PowerShell | Script Block Logging (EID 4104) | Built-in / Query |
| **Persistence** | `T1547.001` | Boot or Logon Autostart Execution: Registry Run Keys | Sysmon EID 13 | `100012` |

<style>
  /* 1. Fix the "Outside" (Home Page Cards) - prevents cropping */
  .post-preview .preview-img img,
  .post-preview .preview-img {
    object-fit: contain !important;
    background-color: #1b1b1e !important;
  }

  /* 2. Fix the "Inside" - Completely hides the header image and its spacing */
  body[data-layout="post"] .post-meta + .mt-3.mb-3,
  body[data-layout="post"] .preview-img {
    display: none !important;
    visibility: hidden !important;
    height: 0 !important;
    margin: 0 !important;
  }
</style>
