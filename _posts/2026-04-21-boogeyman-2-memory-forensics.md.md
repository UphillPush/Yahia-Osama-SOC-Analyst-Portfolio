---
title: "SIEM & Memory Forensics: Boogeyman 2 Triage"
date: 2026-04-21 00:00:00 +0200
categories: ["Threat Hunting & Triage", "SIEM Investigations"]
tags: [volatility, memory-forensics, c2-detection, incident-response, blue-team]
pin: true
---


# Boogeyman 2
<img width="1915"  alt="image" src="https://github.com/user-attachments/assets/bf141f87-3e32-4675-a6f9-35e2e9c7f0ab" />


## The Boogeyman is back!

#### Maxine, a Human Resource Specialist working for Quick Logistics LLC, received an application from one of the open positions in the company. Unbeknownst to her, the attached resume was malicious and compromised her workstation.

<img width="2164"  alt="image" src="https://github.com/user-attachments/assets/59fb264e-b9d1-4e28-b0ee-f13bf55e4110" />

The security team was able to flag some suspicious commands executed on the workstation of Maxine, which prompted the investigation. Given this, you are tasked to analyse and assess the impact of the compromise.

## Answer the questions below
>![](https://img.shields.io/badge/Question-blue) What email was used to send the phishing email?

#### open the .eml file in the artifacts on the Desktop
<img width="1908"  alt="Screenshot 2026-04-08 150237" src="https://github.com/user-attachments/assets/35a4996a-45e4-49fd-9178-e735d30a341e" />

<br>
<br>

>![](https://img.shields.io/badge/Answer-success) westaylor23@outlook.com
<br> 
<br>
<br> 

> ![](https://img.shields.io/badge/Question-blue) **What is the email of the victim employee?**

In the same .eml, you can find the email this email was sent to

<img width="1917"  alt="Screenshot 2026-04-08 150324" src="https://github.com/user-attachments/assets/424df624-1ab1-435e-bde0-24f2de4bbe34" />

<br>

>![](https://img.shields.io/badge/Answer-success) maxine.beck@quicklogisticsorg.onmicrosoft.com

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the name of the attached malicious document?**

<img width="1919"  alt="image" src="https://github.com/user-attachments/assets/c817d8d7-77cb-4222-ac9a-90b71a7d2142" />

<br> 
<br> 

>![](https://img.shields.io/badge/Answer-success) Resume_WesleyTaylor.doc

<br> 
<br> 
<br> 
<br> 

> ![](https://img.shields.io/badge/Question-blue) **What is the MD5 hash of the malicious attachment?**

The attachment is downloaded in the downloads, then using the terminal, there with the command 
`md5sum` will show the MD5 hash of the file

<img width="1680"  alt="Screenshot 2026-04-08 150519" src="https://github.com/user-attachments/assets/1b03a7bf-7242-469d-8351-1fc9b97875a9" />
<br>
<br>

>![](https://img.shields.io/badge/Answer-success) 52c4384a0b9e248b95804352ebec6c5b
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What URL is used to download the stage 2 payload based on the document's macro?**

The URL can be found using the `strings` command on the file in the terminal to search for strings that start with http 
<br> 
<img width="1741"  alt="Screenshot 2026-04-08 150909" src="https://github.com/user-attachments/assets/078396da-bd50-40a0-aa6c-4855cc70ecfb" />
<br> 
<br> 

> ![](https://img.shields.io/badge/Answer-success) https://files.boogeymanisback.lol/aa2a9c53cbb80416d3b47d85538d9971/update.png
     
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the name of the process that executed the newly downloaded stage 2 payload?**

since we are looking for an executable, the same `strings` command was used but with the ".exe" search parameter <br> 
<img width="1647"  alt="Screenshot 2026-04-08 151051" src="https://github.com/user-attachments/assets/cf858e67-f7f0-4f07-9f8e-16243b2131d3" />
<br>
<br> 

> ![](https://img.shields.io/badge/Answer-success) wscript.exe

<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the full file path of the malicious stage 2 payload?**

The file path is so clear next to the executable name in the previous image <br> 
<img width="1695"  alt="Screenshot 2026-04-08 151110" src="https://github.com/user-attachments/assets/e095b83b-785a-49c2-bbf8-4640f1523a1f" />

> ![](https://img.shields.io/badge/Answer-success) C:\ProgramData\update.js

<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the PID of the process that executed the stage 2 payload?**

the Volatilitynn is an open-source framework (opens in new tab) for extracting digital artefacts from volatile memory (RAM) samples.
it can be used here to find the PID as `vol -f <memory/dump/file> windows.plugins`
Using the `WKSTN-2961.raw` file and pstree plugin, which is a capture of the RAM, the processes that were the `wpscript.exe` can be read from it using the following command <br>

<img width="1637"  alt="Screenshot 2026-04-08 154454" src="https://github.com/user-attachments/assets/7bda262a-915d-40d0-b90c-77969bf756a9" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) 4260

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the parent PID of the process that executed the stage 2 payload?**

Using the same command in the previous question without the grep will give a full picture of the process details including the PID<br>

<img width="1501"  alt="Screenshot 2026-04-08 154645" src="https://github.com/user-attachments/assets/de6253c6-07ba-41fc-add0-10dfabd4aec4" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) 1124

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What URL is used to download the malicious binary executed by the stage 2 payload?**

From the previous question, it was observed that the malicious process executed right after the wpscript.exe was updater.exe

<img width="1688"  alt="Screenshot 2026-04-08 170824" src="https://github.com/user-attachments/assets/8520dc59-9d45-4473-9f44-81dd5d5347b4" />

using the filescan plugin this time to search for update to find out the URL that downloaded it `vol -f WKSTN-2961.raw windows.filescan | grep update`

<img width="984"  alt="image" src="https://github.com/user-attachments/assets/590b30ce-3d7f-4862-be59-61a88e17cd77" />

Volatility filescan output actually displays the Virtual Address of the FILE_OBJECT in the System/Kernel memory space. This will be used to dump the file by using the .dumpfiles plugin and the ‘virtaddr’ flag
`vol -f WKSTN-2961.raw windows.dumpfiles --virtaddr 0xe58f836edc60`

<img width="1450"  alt="image" src="https://github.com/user-attachments/assets/9a88012d-6a34-406e-b778-201fa817c445" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) https://files.boogeymanisback.lol/aa2a9c53cbb80416d3b47d85538d9971/update.exe

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the PID of the malicious process used to establish the C2 connection?**

Using the netscan plugin with pipelining grep update search, it's found that the update established a connection with PID 6216

<img width="1562"  alt="Screenshot 2026-04-08 171035" src="https://github.com/user-attachments/assets/c590e197-f9d5-4951-87a4-f8d75288f8d3" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) 6216

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the full file path of the malicious process used to establish the C2 connection?**

File path is decided using the cmd, so using the cmdline plugin with updater search gives use the location 

<img width="1300"  alt="Screenshot 2026-04-08 171422" src="https://github.com/user-attachments/assets/878d9f96-bb10-4d39-9080-dee0a6feb899" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) C:\Windows\Tasks\updater.exe

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the IP address and port of the C2 connection initiated by the malicious binary? (Format: IP address:port)**

Using the netscan plugin as before, we can see the IP and the port right after the updater.exe process 

<img width="1643"  alt="Screenshot 2026-04-08 171539" src="https://github.com/user-attachments/assets/0eee69a9-97be-4f04-b184-43c6033421c0" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) 128.199.95.189:8080

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the full file path of the malicious email attachment based on the memory dump?**

Searching for the attachment name `Resume` using filescan plugin, the path is found

<img width="1612"  alt="Screenshot 2026-04-08 172029" src="https://github.com/user-attachments/assets/d49784e1-579e-4b1d-9f9e-9017d5e8a421" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) C:\Users\maxine.beck\AppData\Local\Microsoft\Windows\INetCache\Content.Outlook\WQHGZCFI\Resume_WesleyTaylor (002).doc

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **The attacker implanted a scheduled task right after establishing the c2 callback. What is the full command used by the attacker to maintain persistent access?**

It's known that the scheduled tasks have the keyword schtasks in the command line, it could be found using strings search for the `WKSTN-2961.raw`

<img width="1919"  alt="Screenshot 2026-04-08 172845" src="https://github.com/user-attachments/assets/69d28703-167f-4f7c-8353-dbce11eadee6" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) schtasks /Create /F /SC DAILY /ST 09:00 /TN Updater /TR 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c \"IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKCU:\Software\Microsoft\Windows\CurrentVersion debug).debug)))\"'
       
