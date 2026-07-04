---
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


#### First, open the phish-emails dir on the desktop and right-click `open terminal here`, then we need to search for the file that contains the pdf attachment. This can be achieved through the command

```bash 
grep -l 'name=".*\pdf"' *.eml
```
This will return the name of the eml file that has a pdf attachment. We want to find out who received that pdf. The command is tunneled with `xargs` to view how this email was sent to in the command below

```bash 
grep -l 'name=".*\pdf"' *.eml | xargs -d '\n' "To:"
```

<img width="100%"  alt="code of answer1" src="https://github.com/user-attachments/assets/f9a4c4a7-1dde-4586-b319-f2bcbf498b41" />
<br>
<br>

>![](https://img.shields.io/badge/Answer-success) William McClean

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What email address was used by the adversary to send the phishing emails?**
#### The sender name of all files can be displayed with grep command too, as follows
```bash
grep -h -A 1 "^From:" *.eml"
```
Where `-h` is used to hide the file name   
&emsp; &emsp;&emsp;`-A` for displaying the line after the search keyword and the lines after, depending on the number follows <br> 
&emsp;&emsp;  &emsp;   `1` number of lines to display after the line match <br>   

  <img width="624"  alt="Picture3" src="https://github.com/user-attachments/assets/707fd174-9ef8-4df1-b6fa-45b6344ba6f5" />
  <br>
  <br>
  
>![](https://img.shields.io/badge/Answer-success) Accounts.Payable@groupmarketingonline.icu

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the redirection URL to the phishing page for the individual Zoe Duncan? (defanged format)**
#### For this question, we have to open the email sent to Zoe and download the attached html to inspect it 
<img width="1795" alt="Screenshot 2026-01-18 175342" src="https://github.com/user-attachments/assets/e0be1594-7286-4eef-a214-3aff404add25" />
<img width="1798" alt="Screenshot 2026-01-18 175412" src="https://github.com/user-attachments/assets/da8fb663-f7ff-4d8c-999f-8cebc27fa606" />

<br><br>
Now, the html file must be inspected for any url inside it, which can be done through the commnad 
```bash
grep "http[^ ]" *.html
```
<img width="1132"  alt="Screenshot 2026-01-18 175932" src="https://github.com/user-attachments/assets/2fecd1a0-3253-4df7-b2e4-30704c38a282" />
<br> 
<br>
So the URL is:   

http://kennaroads.buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe.duncan@swiftspend.finance&error    

  
This URL has to be defanged as required in the question, so Cyberchef was used to do it: 
<br> 
<br> 
<img width="100%"  alt="Picture4" src="https://github.com/user-attachments/assets/83df2aca-2f4f-46bd-b081-dafb5ddfdbc3" />   
<br> 
>![](https://img.shields.io/badge/Answer-success) hxxp[://]kennaroads[.]buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe[.]duncan@swiftspend[.]finance&error
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the URL to the .zip archive of the phishing kit? (defanged format)**
#### The phishing URL sent to Zeo has a specific path to her, but if we remove the path related to the user, it will direct us to some files it's hosting 
so for the link:   
http://kennaroads.buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe.duncan@swiftspend.finance&error  
we will just keep the main path:   
http://kennaroads.buzz/data/Update365/  
<img width="624" height="312" alt="Picture5" src="https://github.com/user-attachments/assets/80a14ab3-5ad4-4aaf-8bd6-227cbfae57d0" />
<br>
<br>
There is no `.zip` file here, so going back a bit to the link:  
http://kennaroads.buzz/data    

show us this page   

   <img width="624" height="303" alt="Picture6" src="https://github.com/user-attachments/assets/e66c4b1f-14e2-4d7c-b9e4-7a3944e648c4" /> 
   <br>
   <br> 
   Now, we can clearly see the `.zip` archive of the phishing kit, which is Update365.zip
   so the link we are looking for is:   
   http://kennaroads.buzz/data/Update365.zip    
   
   defang this URL using CyberChef:   
<img width="1430" height="695" alt="Picture7" src="https://github.com/user-attachments/assets/3d22b28d-70d0-4290-9310-5a1de12ed48b" />
<br>
<br>

>![](https://img.shields.io/badge/Answer-success) hxxp[://]kennaroads[.]buzz/data/Update365[.]zip
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the SHA256 hash of the phishing kit archive?**
#### Download the Update365.zip in the phish-emails dir, then in the terminal write the command:   
```bash
sha256sum Update365.zip
```

<img width="624" height="232" alt="Picture8" src="https://github.com/user-attachments/assets/d5a3d8b7-99a4-46ef-9898-689ec866dec5" />
<br> 
<br>

> ![](https://img.shields.io/badge/Answer-success) ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686
     
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **When was the phishing kit archive first submitted? (format: YYYY-MM-DD HH:MM:SS UTC)**
#### Using an open-source, popular tool for investigating viruses, VirusTotal is the best option.   
In my browser, I visited virustotal page and inserted the archive sha256 hash to scan it, and easily saw the submission date  
<img width="100%"  alt="image" src="https://github.com/user-attachments/assets/0dd5610f-7af5-4716-9f1a-aa2217ffffeb" />

<br> 
<br>

> ![](https://img.shields.io/badge/Answer-success) 2020-04-08 21:55:50 UTC

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What was the email address of the user who submitted their password twice?**
#### The best way to track the login attempts is through logs, and in previous questions its know where are the hosting files  
so back again to the link:   
http://kennaroads.buzz/data    

a `log.txt` file is seen, open it and try to look up for repeated email or password     

<img width="975"  alt="image" src="https://github.com/user-attachments/assets/1c7fb47b-d887-417a-8015-05320392bb11" />  

> ![](https://img.shields.io/badge/Answer-success) michael.ascot@swiftspend.finance
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What was the email address used by the adversary to collect compromised credentials?**
#### To know this we have to investigate more in the phishing kit archive, heading back to the host website at:   
http://kennaroads.buzz/data/Update365.zip     

   the Update365.zip is downloaded, then we move it to the phish-emails dir to investigate it using the command
   ```bash
  mv ~/Downloads/Update365.zip .
```
Then Unzip it using the command  
```bash
unzip Update365.zip
```

<img width="1289"  alt="image" src="https://github.com/user-attachments/assets/35f93b98-3539-4862-b259-c185e12e83cd" />

  The file that collects emails and send it have the word send in it for sure and is in the Update365 folder, so searching for files that contain the send keyword using the command  
  ```bash
grep -r "*send"
```
we found  

<img width="1134" alt="image" src="https://github.com/user-attachments/assets/fa5c1788-f72f-4d80-985b-d78edfb768f2" />   

  The submit.php is the file responsible for the submit button, to further investigate it we tunneled the command with cat
<img width="975"  alt="image" src="https://github.com/user-attachments/assets/75cc76b2-1453-4bab-a78b-d8112be8b627" />
<img width="975" alt="image" src="https://github.com/user-attachments/assets/dd1122d7-87e0-4335-a413-37ad673aaa00" />
<br>


Here, another suspicious email is found, and the responses are sent to whom is 
> ![](https://img.shields.io/badge/Answer-success) m3npat@yandex.com
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **The adversary used other email addresses in the obtained phishing kit. What is the email address that ends in "@gmail.com"?**
#### This one is easy, you just have to search for an email that ends with @gmail.com using the command:  
```bash
grep -r "@gmail.com"
```
<img width="1319"  alt="image" src="https://github.com/user-attachments/assets/88218dd2-ad78-4a3f-bb85-7f103140ebc8" />
<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) jamestanner2299@gmail.com
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the hidden flag?**

Reading the instruction for this one :   
<img width="651" alt="image" src="https://github.com/user-attachments/assets/6a401184-e86a-4eba-aa40-fbc8d4539d84" />  

  Then the flag is a text file that is a subdomain or a directory of the phishing URL, since there is no enumeration tool in the VM like Gobuster, I guessed its called `flag.txt`   
  so I tried this directory in the URL :   
  <img width="975"  alt="image" src="https://github.com/user-attachments/assets/42596687-eee8-4ac9-a91e-5ada5691f5e9" />
  <br> 
  So the flag is found, but it seems to be encoded. For that, we will use CyberChef to decode it back 

  <img width="975"  alt="image" src="https://github.com/user-attachments/assets/53228822-dc45-47b2-9479-ef89ee5fd601" />
  <br>
  <br>
  
  > ![](https://img.shields.io/badge/Answer-success) THM{pL4y_w1Th_tH3_URL}



    
