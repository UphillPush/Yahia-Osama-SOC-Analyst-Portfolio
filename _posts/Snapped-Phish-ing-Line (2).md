<img width="949" height="652" alt="image" src="https://github.com/user-attachments/assets/2405970d-5c3f-40fb-ac23-912044a23201" /><img width="949" height="652" alt="image" src="https://github.com/user-attachments/assets/328b19ca-144c-42de-89dc-0cbe595d3482" />---
layout: default
title: "TryHackMe: Snapped Phish-ing Line Walkthrough"
description: "Detailed solution and walkthrough for the Snapped Phish-ing Line challenge on TryHackMe."

redirect_from: 
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line.html
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line
---


# Detecting LSASS Credential Dumping in ELK Using Atomic Red Team

#### Generating real adversary telemetry on a Windows VM and building Kibana detection rules that catch it in under five minutes — from comsvcs.dll dumps to encoded PowerShell execution.

Contents
Lab overview & architecture
Phase 1 — Environment setup
Phase 2 — Install Atomic Red Team
Phase 3 — Execute attack simulations
Phase 4 — Kibana detection rules
Phase 5 — Dashboard build
Phase 6 — Cleanup
STAR write-up template
MITRE ATT&CK mapping

## 01 Lab overview & architecture

Credential dumping via LSASS memory (MITRE ATT&CK T1003.001) remains a critical execution checkpoint for ransomware operators and APT actors to achieve privilege escalation. Defending against this vector requires deep visibility; this lab bridges the gap between threat theory and defense by emulating the attack, isolating the resulting telemetry, and building resilient detection logic from the ground up to secure the enterprise perimeter.

###Objective
Simulate three distinct LSASS dumping sub-techniques on a Windows 10 VM, ship the generated telemetry to an ELK Stack via Winlogbeat, and build working Kibana detection rules that independently catch each technique.

### Architecture 
<img width="1451" height="593" alt="image" src="https://github.com/user-attachments/assets/f68dbe7d-faa5-4587-b2ed-6a5ec452cbd4" />

## Phase 1 — Environment setup
This phase configures Windows auditing and Sysmon so that the attacks in Phase 3 actually generate visible telemetry — most tutorials skip this and end up with empty Kibana dashboards.

First of all teo virtual machines are created: 
1- the windows 10 machine which would be monitored 
2- the ubuntu machine where it detects attack and makes an alerts 

the two machine should be connected on the same network and can ping each other

<img width="1904" height="945" alt="image" src="https://github.com/user-attachments/assets/ce804a86-35ad-4f67-8fb2-158524661457" />

note: if win can ping the linux while the opposite doesn't happen, make sure the ICMP echo inbound rule is enables in windows firewall

section 1 - Ubuntu set-up 
ELK requires java installation, where it could be installed through the command: 
```shell
sudo apt install default-jdk -y
```
for verifying java installation : 
```shell
java -version
```
where it shows the version of java installed

then elastic search is downloaded and installed through the command:
```shell
cd /home/$(whoami)/Downloads
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.9-amd64.deb
sudo dpkg -i elasticsearch-7.17.9-amd64.deb
```
 after installing elastic search, the `elasticsearch.yml` is opened with the command 
 ```shell
sudo nano /etc/elasticsearch/elasticsearch.yml
```
tip: ssh through local host cmd is recommended to facilitate the commands trasfer through VMs 

 then the `elasticsearch.yml` file is configured through adding 
 ```yml
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
Saved: Ctrl+X → Y → Enter

For ensuring the password for elastic user is known: 
```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
```
This will show the password for the elastic user, it coppied for the further steps

after installing and configuring elasticsearch, its started through the command
```shell
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```
If the status shows active, the elastic search is configured and run succcessfully. 
browsing `http://[MACHINE_IP]:9200` Should return JSON with cluster info.

<img width="817" height="415" alt="image" src="https://github.com/user-attachments/assets/41ca64f8-1a06-4fd2-b3b4-32d0f20c7296" />

After the installation of the elasticsearch, the dashboard for reviewing the logs and making alerts is Kibana, which could be dwonloaded and installed through the commands 
```shell
cd /home/$(whoami)/Downloads
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.17.9-amd64.deb
sudo dpkg -i kibana-7.17.9-amd64.deb
```

Kibana is configured as well through the `kibana.yml` file 
```shell
sudo nano /etc/kibana/kibana.yml
```
Where some modifications and configurations are added 
```yml
server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "oog1Rh3LSTjWHlYzLY1u"
xpack.encryptedSavedObjects.encryptionKey: "thisisaverylongrandomstringof32charactersforencryption12345"
```
note: the kibana_system and its password was gotten from elasticsearch previously 

Now, starting kibana through the commands
```shell
sudo systemctl start kibana
sudo systemctl enable kibana
```
then browsing the url `http://[MACHINE_IP]:5601`
will prompt this login page, where the elastic password is used 
<img width="945" height="675" alt="image" src="https://github.com/user-attachments/assets/9b636ced-7ee5-44c5-ab36-94961a901840" />
Kibana home should be seen . If blank, the site is refreshed after 10 seconds.

## PART 3: WINDOWS (TINY10) SETUP
For this part, atomic red team is used; which is  an open-source library of bite-sized, safe cyberattack simulations mapped to the MITRE ATT&CK framework. By running simple, isolated commands—like mimicking a hacker trying to steal a password or alter a registry key—security teams can instantly verify whether their logging and alerts actually catch the threat. It is essentially a quick, copy-paste fire drill that lets you safely test your own defenses to ensure they work before a real attack happens.

for it to run, the windows defender real-time protection has to be disabled in order not to block the atomic red team 
it's disabled in the powershell through the command 
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```
verifying its off 

```powershell
Get-MpPreference | Select DisableRealtimeMonitoring
```
which should return true 
<img width="700" height="330" alt="image" src="https://github.com/user-attachments/assets/b4065c3e-335f-4e11-b295-b3cafe9f9ba5" />

Default windows logs doesn't show much details about the logs as if there was an embedded command it won't appear, so Sysmon is used and installed to have more detailed logs to help more in investigations. As Sysmon provides detailed Windows telemetry (Event ID 1, 3, 7, 8, 10, 13, etc.).
It could be downloaded through the link:
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
Then extraced in a folder with path `C:\Sysmon\ (create the folder first)`

To decrease the noise of collected logs and colled only needed and mendatory logs, a configuration file is used to configure the sysmon installation with just the needed logs collection 
```powershell
cd C:\Sysmon
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmon-config.xml"
```
then Sysmon is installed along with this configuration 
```powershell
cd C:\Sysmon
.\Sysmon64.exe -i -c sysmon-config.xml -accepteula
```
Verifyning the installation 
```powershell
Get-Service Sysmon64
```
<img width="431" height="186" alt="image" src="https://github.com/user-attachments/assets/72903f71-67cf-4d61-90ba-db5362b1b8fd" />

Command line logs has to be enabled as well for the powershell commands to be seen 
```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\policies\system\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```
verifying through the command:
```powershell
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\policies\system\Audit"
```
<img width="768" height="94" alt="image" src="https://github.com/user-attachments/assets/215305f1-de17-4ea1-af6f-e2ff070f2120" />

Installing winlogbeat
 After setting the the attacking tool and defining the logs to be collected, winlogbeat is used to monitor and gather the logs defines and send it all to the elastic search for the investigation. 
 winlogbeat could be downloaded through this command:
 ```powershell
cd $env:TEMP
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-7.17.9-windows-x86_64.zip" -OutFile "winlogbeat-7.17.9-windows-x86_64.zip"
Expand-Archive "winlogbeat-7.17.9-windows-x86_64.zip" -DestinationPath "C:\Program Files\"
Rename-Item "C:\Program Files\winlogbeat-7.17.9-windows-x86_64" -NewName "Winlogbeat"
```
Then as the elastic search and kibana was configures, the winlogbeat is configured as well through the .yml file as follows:
```yml
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
  password: "5klVgGXUiPZBIFM1QAP9"
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

then it's installed as a windows service 
```powershell
cd "C:\Program Files\Winlogbeat"
.\install-service-winlogbeat.ps1
```
And started with verifying through the commands
```powershell
Start-Service winlogbeat
Get-Service winlogbeat
```
<img width="528" height="117" alt="image" src="https://github.com/user-attachments/assets/a87497fd-4d31-47b3-b108-7c7e06c7d3ca" />

verifying the data is sent in kibana:
The kibana is opened in the ubuntu machine through the url `http://localhost:5601`

then browsed to Management -> Dev tools 

<img width="909" height="663" alt="image" src="https://github.com/user-attachments/assets/44e90dad-295f-4dc2-9209-ff5d123cce06" />
 In the console, `GET winlogbeat-*/_search` is typed and pressed play button, which shows thw windows events 
<img width="941" height="650" alt="image" src="https://github.com/user-attachments/assets/9e161577-9cf4-4036-a1a3-17a1ce125a4b" />

Now after installing all the required tools and made all the needed configuration, its time for testing! 

For begining the testing, atomic red team is downloaded through the commands: 
```powershell
cd C:\
Invoke-WebRequest -Uri "https://github.com/redcanaryco/atomic-red-team/archive/master.zip" -OutFile "atomic-red-team-master.zip"
Expand-Archive "atomic-red-team-master.zip" -DestinationPath "C:\"
Rename-Item "C:\atomic-red-team-master" -NewName "AtomicRedTeam"
```
Installing Invoke-AtomicRedTeam Module right after:
```powershell 
cd "C:\AtomicRedTeam\invoke-atomicredteam"
.\Install-AtomicRedTeam.ps1 -getAtomics
```
note: Here is the difference in a nutshell:

- Atomic Red Team (The Library): This is the actual collection of tests (written as YAML files). Think of it as a recipe book filled with instructions on how to mimic hacker techniques.

- Invoke-AtomicRedTeam (The Runner): This is a PowerShell module used to execute those tests. Think of it as the chef that reads the recipe book and actually cooks the meal (runs the commands on your system).

For Importing the module: 
```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1"
```
verifying 
```powershell
Invoke-AtomicTest T1003.001 -ShowDetails
```
<img width="1659" height="1075" alt="image" src="https://github.com/user-attachments/assets/3a3e1af2-bd97-47cc-a89f-59a2122379bb" />

PART 5:  CREATE KIBANA INDEX & RULES
First kibana index pattern is created through 
1- clicking Management -> Index pattern 
2- then Create index pattern
3- Naming it winlogbeat-*
4- clicking next 
5- Timestamp field: @timestamp
6- clicking Create index pattern 

after these steps, the winlogbeat-* index could be seen in the discover page with all the logs recieved from the windows PC 
note: the date has to be changed for enough period

<img width="949" height="652" alt="image" src="https://github.com/user-attachments/assets/5d4db6ae-48f0-4fea-ad05-4479b8bacabe" />

A detection  rule is then created to alert for any incidents set by the rule through these steps:
Rule 1: LSASS Process Access (Event ID 10)
1- clikcing security -> Rules 
2- Clicking Create new rule 
3- Selecting Custom Query 
4- in the Query box `event.code: "10" AND winlog.event_data.TargetImage: *lsass.exe` is typed 
5- Clicking continue 
6- Filling in: 
    - Name: LSASS Memory Access
    - Description: Credential dumping attempt 
    - Severity: High 
    - Risk score: 85
    - MITRE ATT&CK: Credential Access -> T1003.001
7- Clicking create and enable rule 
<img width="940" height="656" alt="image" src="https://github.com/user-attachments/assets/c39cc5f7-435c-42a9-bc09-43f216699a18" />

Rule 2: Encoded PowerShell Execution (Event ID 4688)
1- creating a new rule 
2- Query: `event.code: "4688" AND process.name: "powershell.exe" AND (winlog.event_data.CommandLine: "*-enc*" OR winlog.event_data.CommandLine: "*-EncodedCommand*")`
3- Filling in:
  -Name: Encoded Powershell execution 
  -Description: detection of any encoded powershell commands 
  -Severity score: 70
  -MITRE ATT&CK: Execution -> T1059.001
4- Clicking create nad enable rule 

<img width="939" height="618" alt="image" src="https://github.com/user-attachments/assets/41c95e07-6cda-43ea-b85e-28a3f41b81e3" />

PART 6: RUN THE LAB
On Windows,  5-10 minutes are waited for initial Sysmon events to ship to Elasticsearch.
Then kibana -> Dev tools is checked through:
```
GET winlogbeat-*/_search
{
  "query": {
    "match": {
      "event.code": "1"
    }
  }
}
```
where it returns a process creation events (Sysmon EID 1)

Executing the Atomic test (T1003.001 — LSASS Credential Dumping)
or the test to  be executed, the following command is run on the windows powershell 
```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1"
Invoke-AtomicTest T1003.001 -ShowDetails
```
where it lists available sub techniques like:
T1003.001-1: (sometimes unavailable)
T1003.001-2: rundll32.exe comsvcs.dll MiniDump ← Use this one
T1003.001-3: PowerShell Out-Minidump
etc.

Running the rundll32 technique:
```powershell
Invoke-AtomicTest T1003.001 -TestNumbers 2
```
This will:
Dump LSASS memory to C:\Windows\Temp\lsass.dmp
Generate Sysmon Event ID 10 (ProcessAccess) + Event ID 1 (process creation)
Generate Windows Security Event ID 4656 (handle request)


Verifying the detection in kibana:
after waiting for 1-2 minutes, the rule should fire in Security -> Alerts 
also the Sucirty -> Finding could be browsed to view the full event details 
<img width="957" height="657" alt="image" src="https://github.com/user-attachments/assets/0781f843-1945-4d9a-88bc-c179a91a5b52" />

Step 6.4: Run Encoded PowerShell Test (T1059.001)

On windows powershell, the ecodded command test is run through the command 
```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 1
```


In 2 minutes, the ecodded command alert fired in the alerts section 

<img width="1191" height="901" alt="image" src="https://github.com/user-attachments/assets/9810adc7-08ab-40d7-9c0e-3d10c8dab4d8" />


