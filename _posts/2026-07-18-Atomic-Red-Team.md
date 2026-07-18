<img width="1774" height="887" alt="80a2966e-31dc-4e48-a59f-0b8661d02afe" src="https://github.com/user-attachments/assets/8344be59-024e-49fd-bde3-3e67699333c6" />---
layout: post
title: "Detecting LSASS Credential Dumping in ELK Using Atomic Red Team"
description: "Generating real adversary telemetry on a Windows VM and building Kibana detection rules that catch it — from comsvcs.dll dumps to encoded PowerShell execution."
date: 2026-07-18 09:00:00 +0300
categories: [Blue Team, Detection Engineering]
tags: [elk, sysmon, winlogbeat, atomic-red-team, mitre-attack, lsass, detection-engineering]
toc: true
image: https://github.com/user-attachments/assets/d75d5523-181c-450a-a155-f0f38b73d7ff
---

# Detecting LSASS Credential Dumping in ELK Using Atomic Red Team

_Generating real adversary telemetry on a Windows VM and building Kibana detection rules that catch it in under five minutes — from `comsvcs.dll` dumps to encoded PowerShell execution._

## Lab overview & architecture

Credential dumping via LSASS memory (MITRE ATT&CK **T1003.001**) remains a critical execution checkpoint for ransomware operators and APT actors to achieve privilege escalation. Defending against this vector requires deep visibility; this lab bridges the gap between threat theory and defense by emulating the attack, isolating the resulting telemetry, and building resilient detection logic from the ground up.

### Objective

Three distinct LSASS-dumping sub-techniques are simulated on a Windows 10 VM, the generated telemetry is shipped to an ELK Stack via Winlogbeat, and working Kibana detection rules are built to independently catch each technique.

### Architecture

![Architecture of the two-VM detection lab](https://github.com/user-attachments/assets/f68dbe7d-faa5-4587-b2ed-6a5ec452cbd4){: width="1451" height="593" }
_A monitored Windows 10 host ships telemetry through Winlogbeat to the Ubuntu ELK stack, where detection rules and alerts are built_

## Phase 1 — Environment setup

This phase configures Windows auditing and Sysmon so that the attacks in [Phase 5](#phase-5--running-the-lab--verifying-detections) actually generate visible telemetry — most tutorials skip this and end up with empty Kibana dashboards.

Two virtual machines are created:

1. A **Windows 10** machine, which is the monitored host.
2. An **Ubuntu** machine, which detects the attacks and raises the alerts.

The two machines are placed on the same network and are able to ping each other.

![ICMP connectivity between the two VMs](https://github.com/user-attachments/assets/ce804a86-35ad-4f67-8fb2-158524661457){: width="1904" height="945" }
_Both VMs reaching each other over ICMP on the same network_

> If Windows can ping Linux but the reverse fails, the ICMP echo inbound rule must be enabled in Windows Firewall.
{: .prompt-tip }

### Ubuntu setup (ELK stack)

ELK requires a Java installation, which is installed with the following command:

```shell
sudo apt install default-jdk -y
```

The installation is then verified, which displays the installed Java version:

```shell
java -version
```

Next, Elasticsearch is downloaded and installed:

```shell
cd /home/$(whoami)/Downloads
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.9-amd64.deb
sudo dpkg -i elasticsearch-7.17.9-amd64.deb
```

After installation, the `elasticsearch.yml` file is opened for configuration:

```shell
sudo nano /etc/elasticsearch/elasticsearch.yml
```

> Connecting over SSH from the host is recommended, as it makes transferring commands between the VMs far easier.
{: .prompt-tip }

The following configuration is then added to the file:

```yaml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/certs/elastic-certificates.p12
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.authc.api_key.enabled: true
#--------------Paths-------------------
path.logs: /var/log/elasticsearch
path.data: /var/lib/elasticsearch
#--------------Network-----------------
network.host: 0.0.0.0
http.port: 9200
#--------------Discovery-------------------
cluster.name: elk-lab
node.name: ubuntu-node
cluster.initial_master_nodes: ["ubuntu-node"]
```
{: file="/etc/elasticsearch/elasticsearch.yml" }

> Save and exit `nano` with `Ctrl + X` → `Y` → `Enter`.
{: .prompt-info }

To make sure the password for the `elastic` user is known, the passwords are generated automatically. The output is copied for the later steps:

```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
```

Once configured, Elasticsearch is started and enabled on boot:

```shell
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```

If the status shows **active**, Elasticsearch has been configured and started successfully. Browsing `http://[MACHINE_IP]:9200`{: .filepath} should return JSON with cluster information.

![Elasticsearch cluster information returned as JSON](https://github.com/user-attachments/assets/41ca64f8-1a06-4fd2-b3b4-32d0f20c7296){: width="817" height="415" }
_Browsing the Elasticsearch endpoint on port 9200 returns cluster details as JSON_

With Elasticsearch running, Kibana — the dashboard used for reviewing logs and building alerts — is downloaded and installed:

```shell
cd /home/$(whoami)/Downloads
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.17.9-amd64.deb
sudo dpkg -i kibana-7.17.9-amd64.deb
```

Kibana is then configured through its `kibana.yml` file:

```shell
sudo nano /etc/kibana/kibana.yml
```

> The credentials below have been replaced with placeholders. Real passwords and encryption keys must never be committed to a public repository — they belong in local, untracked config only.
{: .prompt-warning }

The following modifications are added:

```yaml
server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "<KIBANA_SYSTEM_PASSWORD>"
xpack.encryptedSavedObjects.encryptionKey: "<32+ CHARACTER ENCRYPTION KEY>"
```
{: file="/etc/kibana/kibana.yml" }

> The `kibana_system` user and its password are taken from the Elasticsearch output generated earlier.
{: .prompt-info }

Kibana is then started and enabled:

```shell
sudo systemctl start kibana
sudo systemctl enable kibana
```

Browsing `http://[MACHINE_IP]:5601`{: .filepath} prompts the login page, where the `elastic` password is used.

![The Kibana login page](https://github.com/user-attachments/assets/9b636ced-7ee5-44c5-ab36-94961a901840){: width="945" height="675" }
_The Kibana login page, reached on port 5601_

> If the Kibana home page appears blank, refresh the page after about 10 seconds.
{: .prompt-tip }

## Phase 2 — Windows (Tiny10) setup

With the ELK stack running, the monitored Windows host is prepared so that attacker activity is logged in useful detail.

### Disabling Windows Defender

So that the Atomic Red Team simulations in [Phase 3](#phase-3--atomic-red-team) are not blocked, Windows Defender real-time protection is disabled from PowerShell:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

It is then verified, which should return `True`:

```powershell
Get-MpPreference | Select DisableRealtimeMonitoring
```

![DisableRealtimeMonitoring returning True](https://github.com/user-attachments/assets/b4065c3e-335f-4e11-b295-b3cafe9f9ba5){: width="700" height="330" }
_`DisableRealtimeMonitoring` returning `True` confirms real-time protection is off_

> Real-time protection is disabled only because this is an isolated lab VM with no internet exposure. It must never be turned off on a production or internet-facing machine.
{: .prompt-danger }

### Installing Sysmon

Default Windows logs do not show much detail — an embedded command, for example, will not appear — so Sysmon is used to produce richer telemetry (Sysmon Event IDs 1, 3, 7, 8, 10, 13, and more) to support investigations.

Sysmon is downloaded from the [official Sysinternals page](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and extracted into a folder at `C:\Sysmon\`{: .filepath} (the folder is created first).

To reduce noise and collect only the required events, a configuration file is used. It is downloaded into the Sysmon folder:

```powershell
cd C:\Sysmon
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmon-config.xml"
```

Sysmon is then installed along with this configuration:

```powershell
cd C:\Sysmon
.\Sysmon64.exe -i -c sysmon-config.xml -accepteula
```

The installation is verified:

```powershell
Get-Service Sysmon64
```

![The Sysmon64 service running](https://github.com/user-attachments/assets/72903f71-67cf-4d61-90ba-db5362b1b8fd){: width="431" height="186" }
_The Sysmon64 service running after installation_

### Enabling command-line logging

Command-line logging must also be enabled so that PowerShell command lines are captured:

```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\policies\system\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```

It is verified with:

```powershell
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\policies\system\Audit"
```

![Command-line auditing enabled in the registry](https://github.com/user-attachments/assets/215305f1-de17-4ea1-af6f-e2ff070f2120){: width="768" height="94" }
_Command-line auditing enabled in the registry_

### Installing Winlogbeat

With the attack tooling prepared and the required logs defined, Winlogbeat is used to monitor and gather those logs and forward them to Elasticsearch for investigation. It is downloaded with:

```powershell
cd $env:TEMP
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-7.17.9-windows-x86_64.zip" -OutFile "winlogbeat-7.17.9-windows-x86_64.zip"
Expand-Archive "winlogbeat-7.17.9-windows-x86_64.zip" -DestinationPath "C:\Program Files\"
Rename-Item "C:\Program Files\winlogbeat-7.17.9-windows-x86_64" -NewName "Winlogbeat"
```

As with Elasticsearch and Kibana, Winlogbeat is configured through its `.yml` file:

```yaml
winlogbeat.event_logs:
  - name: System
    ignore_older: 24h
  - name: Security
    ignore_older: 24h
  - name: Microsoft-Windows-Sysmon/Operational
    ignore_older: 24h

# ====================== Elasticsearch template settings =======================
setup.template.name: "name1"
setup.template.pattern: "name2"

# ---------------------------- Elasticsearch Output ----------------------------
output.elasticsearch:
  enable: true
  # Array of hosts to connect to.
  hosts: ["192.168.1.7:9200"]
  protocol: "http"
  username: "elastic"
  password: "<ELASTIC_PASSWORD>"
  index: "winlogbeat-%{+yyyy.MM.dd}"

logging.level: info
logging.to_files: true
logging.files:
  path: C:\ProgramData\Winlogbeat\Logs

# ================================= Processors =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
```
{: file="winlogbeat.yml" }

It is then installed as a Windows service:

```powershell
cd "C:\Program Files\Winlogbeat"
.\install-service-winlogbeat.ps1
```

The service is started and verified:

```powershell
Start-Service winlogbeat
Get-Service winlogbeat
```

![The Winlogbeat service running](https://github.com/user-attachments/assets/a87497fd-4d31-47b3-b108-7c7e06c7d3ca){: width="528" height="117" }
_The Winlogbeat service running after installation_

### Verifying data in Kibana

Kibana is opened on the Ubuntu machine at `http://localhost:5601`{: .filepath}, then **Management → Dev Tools** is browsed to.

![Opening the Dev Tools console in Kibana](https://github.com/user-attachments/assets/44e90dad-295f-4dc2-9209-ff5d123cce06){: width="909" height="663" }
_Opening the Dev Tools console from the Kibana management menu_

In the console, `GET winlogbeat-*/_search` is typed and the play button is pressed, which shows the Windows events.

```text
GET winlogbeat-*/_search
```

![Windows events returned from the winlogbeat index](https://github.com/user-attachments/assets/9e161577-9cf4-4036-a1a3-17a1ce125a4b){: width="941" height="650" }
_Windows events returned from the winlogbeat index, confirming telemetry is flowing_

## Phase 3 — Atomic Red Team

With every tool installed and the data flow confirmed, the attack simulations are prepared.

Atomic Red Team is an open-source library of bite-sized, safe cyberattack simulations mapped to the MITRE ATT&CK framework. By running simple, isolated commands — such as mimicking a hacker stealing a password or altering a registry key — security teams can instantly verify whether their logging and alerts actually catch the threat. It is essentially a quick, copy-paste fire drill for safely testing defenses before a real attack happens.

Atomic Red Team is downloaded with:

```powershell
cd C:\
Invoke-WebRequest -Uri "https://github.com/redcanaryco/atomic-red-team/archive/master.zip" -OutFile "atomic-red-team-master.zip"
Expand-Archive "atomic-red-team-master.zip" -DestinationPath "C:\"
Rename-Item "C:\atomic-red-team-master" -NewName "AtomicRedTeam"
```

The `Invoke-AtomicRedTeam` module is installed right after:

```powershell
cd "C:\AtomicRedTeam\invoke-atomicredteam"
.\Install-AtomicRedTeam.ps1 -getAtomics
```

Atomic Red Team (the library)
: The actual collection of tests, written as YAML files — a recipe book of instructions for mimicking attacker techniques.

Invoke-AtomicRedTeam (the runner)
: A PowerShell module used to execute those tests — the chef that reads the recipe book and cooks the meal on the system.

The module is then imported:

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1"
```

And verified:

```powershell
Invoke-AtomicTest T1003.001 -ShowDetails
```

![Available T1003.001 sub-techniques listed by Invoke-AtomicTest](https://github.com/user-attachments/assets/3a3e1af2-bd97-47cc-a89f-59a2122379bb){: width="1659" height="1075" }
_Available T1003.001 sub-techniques listed by Invoke-AtomicTest_

## Phase 4 — Kibana index & detection rules

### Creating the index pattern

An index pattern is created so the collected logs can be searched and alerted on:

1. **Management → Index Patterns** is clicked.
2. **Create index pattern** is selected.
3. The pattern is named `winlogbeat-*`.
4. **Next** is clicked.
5. The timestamp field is set to `@timestamp`.
6. **Create index pattern** is clicked.

After these steps, the `winlogbeat-*` index can be seen on the **Discover** page with all the logs received from the Windows host.

> The time range must be widened to cover a long enough period, otherwise recent events may not appear.
{: .prompt-tip }

![The winlogbeat index in the Discover view](https://github.com/user-attachments/assets/5d4db6ae-48f0-4fea-ad05-4479b8bacabe){: width="949" height="652" }
_The winlogbeat-* index in Discover, showing logs received from the Windows host_



### Rule 1 — LSASS process access (Event ID 10)

A detection rule is then created to alert on any matching activity:

1. **Security → Rules** is clicked.
2. **Create new rule** is clicked.
3. **Custom Query** is selected.
4. The following query is entered:

   ```text
   event.code: "10" AND winlog.event_data.TargetImage: *lsass.exe
   ```

5. **Continue** is clicked.
6. The details are filled in:
   - **Name:** LSASS Memory Access
   - **Description:** Credential dumping attempt
   - **Severity:** High
   - **Risk score:** 85
   - **MITRE ATT&CK:** Credential Access → T1003.001
7. **Create and enable rule** is clicked.

![The LSASS Memory Access detection rule](https://github.com/user-attachments/assets/c39cc5f7-435c-42a9-bc09-43f216699a18){: width="940" height="656" }
_The LSASS Memory Access detection rule after creation_

### Rule 2 — Encoded PowerShell execution (Event ID 4688)

1. A new rule is created.
2. The following query is entered:

   ```text
   event.code: "4688" AND process.name: "powershell.exe" AND (winlog.event_data.CommandLine: "*-enc*" OR winlog.event_data.CommandLine: "*-EncodedCommand*")
   ```

3. The details are filled in:
   - **Name:** Encoded PowerShell Execution
   - **Description:** Detection of any encoded PowerShell commands
   - **Severity:** High
   - **Risk score:** 70
   - **MITRE ATT&CK:** Execution → T1059.001
4. **Create and enable rule** is clicked.

![The Encoded PowerShell Execution detection rule](https://github.com/user-attachments/assets/41c95e07-6cda-43ea-b85e-28a3f41b81e3){: width="939" height="618" }
_The Encoded PowerShell Execution detection rule after creation_

## Phase 5 — Running the lab & verifying detections

On Windows, 5–10 minutes are allowed for the initial Sysmon events to ship to Elasticsearch. Kibana **Dev Tools** is then checked:

```text
GET winlogbeat-*/_search
{
  "query": {
    "match": {
      "event.code": "1"
    }
  }
}
```

This returns process-creation events (Sysmon Event ID 1).

### Executing the LSASS dump (T1003.001)

For the test to be executed, the following commands are run in Windows PowerShell:

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1"
Invoke-AtomicTest T1003.001 -ShowDetails
```

The available sub-techniques are listed, such as:

- **T1003.001-1** — (sometimes unavailable)
- **T1003.001-2** — `rundll32.exe comsvcs.dll MiniDump` ← the one used here
- **T1003.001-3** — PowerShell `Out-Minidump`

The `rundll32` technique is run:

```powershell
Invoke-AtomicTest T1003.001 -TestNumbers 2
```

This will:

- Dump LSASS memory to `C:\Windows\Temp\lsass.dmp`{: .filepath}
- Generate Sysmon Event ID 10 (`ProcessAccess`) and Event ID 1 (process creation)
- Generate Windows Security Event ID 4656 (handle request)

### Verifying the detection

After waiting 1–2 minutes, the rule fires in **Security → Alerts**. **Security → Findings** can also be browsed to view the full event details.

![The LSASS credential-dumping alert firing](https://github.com/user-attachments/assets/0781f843-1945-4d9a-88bc-c179a91a5b52){: width="957" height="657" }
_The LSASS credential-dumping alert firing in the Security → Alerts view_

### Executing encoded PowerShell (T1059.001)

In Windows PowerShell, the encoded-command test is run:

```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 1
```
![The encoded PowerShell alert firing](https://github.com/user-attachments/assets/9810adc7-08ab-40d7-9c0e-3d10c8dab4d8){: width="1191" height="901" }

Within 2 minutes, the encoded-command alert fires in the Alerts section.


_The encoded PowerShell alert firing shortly after the test is run_

## MITRE ATT&CK mapping

| Tactic | Technique | ID | Detection built in this lab |
| ------ | --------- | -- | --------------------------- |
| Credential Access | OS Credential Dumping: LSASS Memory | T1003.001 | Rule 1 — Sysmon Event ID 10 (`ProcessAccess`) targeting `lsass.exe` |
| Execution | Command and Scripting Interpreter: PowerShell | T1059.001 | Rule 2 — encoded PowerShell command line (Event ID 4688) |
