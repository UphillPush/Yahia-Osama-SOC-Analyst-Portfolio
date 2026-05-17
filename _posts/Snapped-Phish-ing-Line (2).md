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

A custom Wazuh rule is added for LSASS access 












Note: The phishing emails to be analysed are under the phish-emails directory on the Desktop. Usage of a web browser, text editor and some knowledge of the grep command will help.

## Answer the questions below
>![](https://img.shields.io/badge/Question-blue) Who is the individual who received an email attachment containing a PDF?



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



    
