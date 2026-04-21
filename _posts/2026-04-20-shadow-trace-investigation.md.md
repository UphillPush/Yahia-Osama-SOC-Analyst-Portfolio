---
title: "Malware Analysis: Shadow Trace IOC Extraction"
date: 2026-04-20 10:00:00 +0200
categories: ["Threat Hunting & Triage", "Malware Analysis"]
tags: [pestudio, static-analysis, ioc, malware, cyberchef]
---

# Shadow Trace
<img width="1908"  alt="image" src="https://github.com/user-attachments/assets/f232b05c-2698-4c0c-9e59-741755855143" />

## File Analysis

#### Analyse the binary located C:\Users\DFIRUser\Desktop\windows-update.exe in the attached machine, answer the questions below. 

Start the lab by clicking the Start Machine button. It will take around 2 minutes to load properly. The VM will be accessible on the right side of the split screen.

You can find several tools installed in the machine that can help you with any kind of analysis under C:\Users\DFIRUser\DFIR Tools


## Answer the questions below
>![](https://img.shields.io/badge/Question-blue) What is the architecture of the binary file windows-update.exe?





#### To find the architecture, the pestudio tool is used to analyze the file
I found it in this path and opened the file windows-update.exe with it

<img width="1919"  alt="image" src="https://github.com/user-attachments/assets/fba5fd06-2593-4de8-a426-81046219663c" />

<br>
<br>

<img width="1770" alt="image" src="https://github.com/user-attachments/assets/a261e987-f744-4151-a59d-51e382c4cde2" />
<br>
<br>

<img width="1917"  alt="image" src="https://github.com/user-attachments/assets/107da2cc-98d8-49ef-be18-542151da3ee3" />

<br>
<br>

>![](https://img.shields.io/badge/Answer-success) 64-bit

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the hash (sha-256) of the file windows-update.exe?**
<img width="1919"  alt="image" src="https://github.com/user-attachments/assets/5e734e1a-ba7e-4c65-803e-040b277bb625" />

  <br>
  <br>
  
>![](https://img.shields.io/badge/Answer-success) B2A88DE3E3BCFAE4A4B38FA36E884C586B5CB2C2C283E71FBA59EFDB9EA64BFC

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **Identify the URL within the file to use it as an IOC**
<img width="1919"  alt="image" src="https://github.com/user-attachments/assets/3ddd959d-7374-4eca-a6d2-aa67ea69b6d2" />

<br> 
<br> 

>![](https://img.shields.io/badge/Answer-success) http://tryhatme.com/update/security-update.exe
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **With the URL identified, can you spot a domain that can be used as an IOC?**
#### This can be found in the file windows-update.exe itself, and since the URL is almost the same (the difference is just the domain), I searched for the word "tryhatme" as a string in the file using   
`strings ./windows-update.exe | findstr "tryhatme"` as shown: 

<img width="1711"  alt="image" src="https://github.com/user-attachments/assets/68e31028-f45c-4745-9e95-0c5c7c0d5172" />


<br>
<br>

>![](https://img.shields.io/badge/Answer-success) responses.tryhatme.com

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **Input the decoded flag from the suspicious domain**
#### There are 2 suspicious domains shown in the image from previous questions, the first one had a code at the end of the link. I used CyberChef to convert this code from Base64, and it displayed this flag 
<br> 
<img width="1917"  alt="image" src="https://github.com/user-attachments/assets/b5614c54-9436-4813-a1b5-863345642704" />
<br> 
<br> 

> ![](https://img.shields.io/badge/Answer-success) THM{you_g0t_some_IOCs_friend}
     
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **When was the phishing kit archive first submitted? (format: YYYY-MM-DD HH:MM:SS UTC)**
#### Returning to pestudio, there is a library tab that shows the libraries that the file uses. For the socket communication, it uses WS2_32.dll  
<br> 
<img width="1919"  alt="image" src="https://github.com/user-attachments/assets/91e43251-a890-48a6-9072-a9cc483c7d45" />
<br>
<br> 

> ![](https://img.shields.io/badge/Answer-success) WS2_32.dll 

<br>
<br>
<br>
<br>

## Alerts Analysis
## Answer the questions below

> ![](https://img.shields.io/badge/Question-blue) **Can you identify the malicious URL from the trigger by the process powershell.exe?**
#### From the site viewed, there are two alerts. In the alert that includes powershell.exe, the command has a suspicious Base64 code
<br> 
<img width="1341"  alt="image" src="https://github.com/user-attachments/assets/57eb6d77-ca7b-4bf3-9d9f-97ad62e66325" />
<br>
<br>
Converting this code back from base64 using CyberChef
<br>
<br>
<img width="1914"  alt="image" src="https://github.com/user-attachments/assets/fff8b866-36d6-465a-ae94-f3773b0a8b54" />
<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) https://tryhatme.com/dev/main.exe
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **Can you identify the malicious URL from the alert triggered by chrome.exe?**
#### In the second Alert, there are malicious decimal places separated by commas that indicate a decimal conversion 
<br>
<img width="1351"  alt="image" src="https://github.com/user-attachments/assets/55939834-ac91-41c6-a3f2-c9825b6ac307" />
<br>
<br>
Converting them back using CyberCef using the from Decimal conversion and choosing the comma as a delimiter
<br>
<br>
<img width="1917"  alt="image" src="https://github.com/user-attachments/assets/8c63515c-ef68-46f2-86fd-105742e3c088" />
<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) m3npat@yandex.com
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What's the name of the file saved in the alert triggered by chrome.exe?**
#### Looking back at the chrome.exe alert command to search for any files, test.txt was found  
<br>
<img width="1309"  alt="image" src="https://github.com/user-attachments/assets/7cc2be13-ec27-47aa-93c3-beac4a8c0e91" />

<br>
<br>

> ![](https://img.shields.io/badge/Answer-success) test.txt


    
