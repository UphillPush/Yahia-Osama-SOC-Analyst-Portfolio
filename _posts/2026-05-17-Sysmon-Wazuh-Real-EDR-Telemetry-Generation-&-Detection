---
layout: post
title: "Sysmon + Wazuh: Real EDR Telemetry Generation & Detection"
date: 2026-05-17 00:00:00 +0200
categories: ["Home Lab", "Detection Engineering"]
tags: [wazuh, sysmon, edr, mitre-attack, blue-team, threat-hunting]
pin: false
image:
  path: https://github.com/user-attachments/assets/bd70f682-71d0-4aeb-a6da-6b816a80618d

  preview: false
  class: img-contain
---

# Building a Custom EDR Pipeline: Sysmon & Wazuh

![Lab Header](https://github.com/user-attachments/assets/ab54b3dc-2dba-485a-ad62-558b29d89327
){: .shadow .rounded }
_Figure 1: EDR Telemetry Generation Lab._

> **Executive Summary**
> **Situation:** It is expected for most SOC analysts to understand EDR telemetry, but entry-level roles rarely explain how the raw events actually look. This lab was built to generate real Sysmon telemetry from actual attack simulations targeting four MITRE ATT&CK techniques, with Wazuh detection logic built entirely from scratch.
> 
> **Action:** A two-VM home lab was deployed. Sysmon v15 was installed using the SwiftOnSecurity configuration, and the Wazuh agent was configured to forward Sysmon, Security, and PowerShell operational logs. Command-line auditing was enabled via Auditpol. Three custom Wazuh rules were written targeting LSASS access, process injection, and run key persistence.
> 
> **Result:** All 4 simulated techniques successfully generated alerts within 15 seconds. The custom rules were triggered accurately, demonstrating that Event ID 4104 (Script Block Logging) catches attacks that Event ID 4688 alone misses, highlighting a major gap in default Windows-only detection.
{: .prompt-info }

---

## Architecture Overview

Before diving into the configurations, the infrastructure must be outlined. The environment consists of two isolated virtual machines communicating over a bridged network:

1.  **The SIEM/EDR Manager (Ubuntu 22.04 LTS):** Allocated with 4GB RAM and 2 CPU cores. This server hosts the Wazuh Manager and Dashboard, acting as the centralized brain for log ingestion and rule processing.
2.  **The Target Endpoint (Windows 10):** Allocated with 4GB RAM and 2 CPU cores. This machine acts as the victim. It is heavily instrumented with Sysmon (for deep kernel-level telemetry) and the Wazuh Agent (to securely forward those logs to the manager).

**The Telemetry Pipeline:** Attacker executes payload on Windows ➔ Sysmon hooks the process/memory/network activity ➔ Wazuh Agent reads the local Windows Event Channels ➔ Logs are shipped to Ubuntu ➔ Wazuh Manager parses the JSON and matches it against custom XML rules ➔ Analyst receives a high-severity alert.

---

## Phase 1: Deploying the Wazuh Manager (Ubuntu)

After spinning up the Ubuntu server, the Wazuh manager was downloaded and installed using the automated script:

```shell
curl -sO [https://packages.wazuh.com/4.7/wazuh-install.sh](https://packages.wazuh.com/4.7/wazuh-install.sh)
sudo bash wazuh-install.sh -a
```
{: .nolineno }

![Wazuh Installation](https://github.com/user-attachments/assets/a36d2dc4-fe5d-4a78-afeb-153346476e04){: .shadow .rounded }
_Figure 2: Executing the Wazuh installation script._

Once complete, the terminal outputs the administrative credentials alongside an "Installation finished" message.

![Wazuh Credentials](https://github.com/user-attachments/assets/b2c99cfc-2501-43ca-bd3f-6a567d8b7bda){: .shadow .rounded }
_Figure 3: Generating the default Wazuh admin credentials._

To verify that the core services were successfully initialized, the status of both the manager and the dashboard was checked:

```shell
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```
{: .nolineno }

![Service Status](https://github.com/user-attachments/assets/b73d4396-12e1-4f0e-94da-124aa3465d72){: .shadow .rounded }
_Figure 4: Confirming active Wazuh services._

The local IP address of the Ubuntu server was retrieved (`192.168.1.7` in this instance) to access the web interface.

![IP Address](https://github.com/user-attachments/assets/edf2dd3d-7bac-42c2-8308-d4fbd7a5e27e){: .shadow .rounded }
_Figure 5: Retrieving the Ubuntu server IP._

After allowing the services to spin up for a few minutes, navigating to `https://192.168.1.7` from the host browser revealed the login page.

![Wazuh Login](https://github.com/user-attachments/assets/35e98f38-df53-46fd-89a4-67ff219277b2){: .shadow .rounded }
_Figure 6: The Wazuh Dashboard login portal._

Upon logging in, the dashboard confirmed that the manager was active, currently showing `0` total agents connected.

![Wazuh Dashboard](https://github.com/user-attachments/assets/abd52b90-cea4-4099-895d-e079de91a735){: .shadow .rounded }
_Figure 7: Empty dashboard awaiting endpoint connection._

---

## Phase 2: Instrumenting the Endpoint (Windows 10)

### 1. Sysmon Installation & Tuning
Standard Windows logging is often too basic and misses advanced hooking techniques. Sysmon acts as a high-definition security camera, monitoring detailed activity like command-line arguments, process tracking, and memory access. 

First, Sysmon was downloaded from the official Microsoft Sysinternals repository.

![Sysmon Download](https://github.com/user-attachments/assets/6aedaa2c-30d3-42a9-8d7a-f215bf68f318){: .shadow .rounded }
_Figure 8: Downloading Sysmon._

If Sysmon is left untuned, it will log everything, creating massive amounts of noise. To optimize the signal-to-noise ratio, the industry-standard **SwiftOnSecurity** configuration was utilized to filter for specifically suspicious patterns.

![SwiftOnSecurity](https://github.com/user-attachments/assets/1ad8d88b-a696-45a3-b7cb-f6f2cb1682e7){: .shadow .rounded }
_Figure 9: Obtaining the SwiftOnSecurity configuration XML._

The configuration file was placed in the extracted Sysmon directory, and the installation was executed via Command Prompt:

```cmd
cd C:\Users\<current user>\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
{: .nolineno }

![Installing Sysmon](https://github.com/user-attachments/assets/6ba23822-c080-4ac7-bef3-89a13a4503ce){: .shadow .rounded }
_Figure 10: Installing Sysmon with the custom XML configuration._

To verify telemetry collection, the 10 most recent Sysmon logs were pulled directly into PowerShell:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 | Select TimeCreated, Id, Message | Format-List
```
{: .nolineno }

![Sysmon Verification](https://github.com/user-attachments/assets/103732bf-d0b0-4a65-8391-42d5a031753d){: .shadow .rounded }
_Figure 11: Validating Sysmon log generation in PowerShell._

Opening the Event Viewer (`Applications and Services Logs -> Microsoft -> Windows -> Sysmon -> Operational`) reveals the raw events, such as Event ID 1 (Process Creation) and Event ID 3 (Network Connections).

![Event Viewer Sysmon](https://github.com/user-attachments/assets/62b50642-140f-4703-920a-3dc969eb3a9a){: .shadow .rounded }
_Figure 12: Reviewing Sysmon EID 1 and EID 3 in Event Viewer._

### 2. Enabling Advanced Windows Auditing
To ensure PowerShell payloads (T1059) are properly captured, **Script Block Logging** was forcibly enabled via the registry:

```powershell 
$p = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $p -Force | Out-Null
Set-ItemProperty -Path $p -Name "EnableScriptBlockLogging" -Value 1
Set-ItemProperty -Path $p -Name "EnableScriptBlockInvocationLogging" -Value 1
```
{: .nolineno }

![Script Block Logging](https://github.com/user-attachments/assets/29a4f305-3f63-45f1-bba8-d14142393295){: .shadow .rounded }
_Figure 13: Enabling PowerShell Script Block Logging (EID 4104)._

Additionally, full command-line strings must be included in Event ID 4688. Without this, only the process name is visible, causing analysts to miss critical flags like `-EncodedCommand`.

```cmd
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```
{: .nolineno }

![Auditpol Configuration](https://github.com/user-attachments/assets/f8c936dd-8d31-4363-8c2f-684312e18d97){: .shadow .rounded }
_Figure 14: Enabling process command-line auditing._

### 3. Deploying the Wazuh Agent
With telemetry generation tuned, the logs must be forwarded. The Wazuh Agent (v4.7.5, matching the manager) was installed on the Windows VM.

```cmd
curl -L [https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi) -o wazuh-agent-4.7.5-1.msi   
```
{: .nolineno }

The agent was then configured with the Ubuntu server's IP and started:

```powershell
(Get-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf') -replace '<address>0.0.0.0</address>', '<address>192.168.1.7</address>' | Set-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf'
del "C:\Program Files (x86)\ossec-agent\client.keys" 2>nul
net start WazuhSvc
```
{: .nolineno }

![Starting Wazuh Agent](https://github.com/user-attachments/assets/d4d2a7e7-44ed-44b8-b806-9216230887df){: .shadow .rounded }
_Figure 15: Starting the Wazuh Agent service._

Finally, the agent's `ossec.conf` was modified to explicitly ingest the newly created Sysmon and PowerShell operational channels:

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
_Figure 16: Updating the agent to forward Sysmon telemetry._

---

## Phase 3: Crafting Custom Detection Logic

For detection to be efficient, custom rules must be written on the Wazuh Manager to flag specific high-severity behaviors. The `local_rules.xml` file was updated with three custom rules:

```shell
sudo nano /var/ossec/etc/rules/local_rules.xml
```
{: .nolineno }

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
    <field name="win.system.eventID">^1$</field>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)cmd\.exe</field>
    <field name="win.eventdata.image" type="pcre2">(?i)powershell\.exe</field>
    <description>Sysmon - Suspicious cmd.exe spawning powershell.exe (T1059.003)</description>
    <mitre>
      <id>T1059.003</id>
    </mitre>
    <group>execution, lolbin,</group>
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

> **Logic Breakdown**
> * **Rule 100010 (Level 15):** Detects Credential Dumping by utilizing `PCRE2` regex to flag any Sysmon Event ID 10 (Process Access) where the target is `lsass.exe`, ignoring case sensitivity.
> * **Rule 100011 (Level 12):** Detects Process Injection by flagging Sysmon Event ID 1 (Process Creation) if `cmd.exe` acts as the parent image for `powershell.exe`.
> * **Rule 100012 (Level 10):** Detects Persistence by monitoring Sysmon Event ID 13 (Registry Value Set). The wildcard `.*` is used to catch any variations of the startup `\Run` key path.
{: .prompt-info }

![Saving Local Rules](https://github.com/user-attachments/assets/ac166270-3f67-4c10-b151-dd9cf245c377){: .shadow .rounded }
_Figure 17: Saving the custom rules in nano._

The manager was restarted to apply the new logic:

![Restarting Manager](https://github.com/user-attachments/assets/bf902f89-86fa-4546-a34d-c3ee47e89d33){: .shadow .rounded }
_Figure 18: Restarting the wazuh-manager service._

---

## Phase 4: Attack Simulation & Triage

With the pipeline fully operational, four distinct attacks were simulated on the Windows endpoint to validate the detection engineering.

### Attack 1: LSASS Memory Dump
To test LSASS memory access, a built-in Windows utility (`rundll32.exe`) was used to execute `comsvcs.dll` and dump the memory of the LSASS process to a temporary file. Memory allocation evasion tactics were utilized to bypass basic AV.

```powershell
$lsass = Get-Process lsass
$handle = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(1)
Write-Host "LSASS PID: $($lsass.Id) — Sysmon EID 10 should fire now"
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $($lsass.Id) C:\Windows\Temp\lsass_dump.dmp full
```
{: .nolineno }

![LSASS Execution](https://github.com/user-attachments/assets/2059d563-961d-4be0-8a7e-195c6fae5515){: .shadow .rounded }
_Figure 19: Executing the LSASS memory dump payload._

**Detection:** The process was instantly detected and alerted in the Wazuh dashboard by our custom rule `100010`.

![LSASS Alert 1](https://github.com/user-attachments/assets/eac44c77-4b31-4e19-b985-2ffa25ca2bd3){: .shadow .rounded }
![LSASS Alert 2](https://github.com/user-attachments/assets/84b1f9c1-9faa-49d1-9564-910ab8c01354){: .shadow .rounded }
_Figure 20: Triage view of the Level 15 LSASS access alert._

### Attack 2: Registry Run Key Persistence
To test the registry persistence rule, an arbitrary executable path was added to the `CurrentVersion\Run` hive.

```powershell 
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "LabPersistence" /t REG_SZ /d "C:\Windows\System32\calc.exe" /f
Write-Host "Run key added — Sysmon EID 13 should fire"
```
{: .nolineno }

**Detection:** The modification was successfully caught by rule `100012` upon querying the dashboard.

![Registry Alert 1](https://github.com/user-attachments/assets/1d0649ab-7909-4d75-b8db-056e8207b125){: .shadow .rounded }
![Registry Alert 2](https://github.com/user-attachments/assets/ad3f553c-2220-425b-8cfe-494956688b78){: .shadow .rounded }
_Figure 21: Triage view of the Registry Run Key alert._

### Attack 3: Encoded PowerShell Execution
A plain text string was translated into Base64 and executed via a hidden, non-interactive PowerShell session to test Script Block Logging capabilities. 

```powershell
$cmd = "Write-Host 'Simulated encoded payload executed — T1059.001'"
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell.exe -NonInteractive -EncodedCommand $encoded
```
{: .nolineno }

![Encoded PowerShell](https://github.com/user-attachments/assets/5ba0ee87-8677-4762-9cbe-cdb34203dbfa){: .shadow .rounded }
_Figure 22: Executing the Base64 encoded payload._

**Detection:** Using DQL (Dashboard Query Language), the event was located by searching for Sysmon EID 1 alongside the `*EncodedCommand*` wildcard.

![Encoded Alert 1](https://github.com/user-attachments/assets/f70e65f5-5cc4-4c30-b0a6-3690ea37c18a){: .shadow .rounded }
![Encoded Alert 2](https://github.com/user-attachments/assets/24a04095-b46a-45a2-aa31-522fa62cebe0){: .shadow .rounded }
_Figure 23: Triage view of the encoded command execution._

### Attack 4: Suspicious Child Process Spawn
The command shell (`cmd.exe`) was used as a parent to secretly spawn a hidden PowerShell child process, attempting to enumerate local user accounts.

```powershell
cmd.exe /c "powershell.exe -NonInteractive -WindowStyle Hidden -Command Get-LocalUser"
```
{: .nolineno }

**Detection:** The anomalous parent-child relationship was flagged immediately by rule `100011`.

![Process Spawn Alert](https://github.com/user-attachments/assets/cb5b9938-9815-4434-b99a-c10fc213b9fe){: .shadow .rounded }
_Figure 24: Triage view of the anomalous CMD to PowerShell execution._

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
