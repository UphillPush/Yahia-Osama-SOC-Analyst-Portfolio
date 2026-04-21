---
layout: post
title: "Malware Analysis: Shadow Trace IOC Extraction"
date: 2026-04-20 10:00:00 +0200
categories: ["Threat Hunting & Triage", "Malware Analysis"]
tags: [pestudio, static-analysis, ioc, malware, cyberchef]
---

# Shadow Trace

![Shadow Trace Room Banner](https://github.com/user-attachments/assets/f232b05c-2698-4c0c-9e59-741755855143){: .shadow .rounded }
_Figure 1: TryHackMe Shadow Trace Challenge._

<br>

## File Analysis

> **Scenario Objective**
> Analyse the binary located at `C:\Users\DFIRUser\Desktop\windows-update.exe`{: .filepath} in the attached machine, answer the questions below. 
{: .prompt-info }

Start the lab by clicking the Start Machine button. It will take around 2 minutes to load properly. The VM will be accessible on the right side of the split screen. You can find several tools installed in the machine that can help you with any kind of analysis under `C:\Users\DFIRUser\DFIR Tools`{: .filepath}.

<br>
<br>

---

<br>

> **Question:** What is the architecture of the binary file `windows-update.exe`{: .filepath}?
{: .prompt-info }

To find the architecture, the **pestudio** tool is used to analyze the file. I found it in this path and opened the file `windows-update.exe`{: .filepath} with it.

![pestudio initial view](https://github.com/user-attachments/assets/fba5fd06-2593-4de8-a426-81046219663c){: .shadow .rounded }
_Figure 2: Opening the malicious executable in pestudio._

![pestudio architecture details](https://github.com/user-attachments/assets/a261e987-f744-4151-a59d-51e382c4cde2){: .shadow .rounded }
_Figure 3: Locating the file architecture in the optional header._

![pestudio file header](https://github.com/user-attachments/assets/107da2cc-98d8-49ef-be18-542151da3ee3){: .shadow .rounded }
_Figure 4: Confirming the 64-bit architecture._

> **Answer:** `64-bit`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** What is the hash (sha-256) of the file `windows-update.exe`{: .filepath}?
{: .prompt-info }

![pestudio indicators](https://github.com/user-attachments/assets/5e734e1a-ba7e-4c65-803e-040b277bb625){: .shadow .rounded }
_Figure 5: Extracting the SHA-256 hash from the pestudio indicators tab._

> **Answer:** `B2A88DE3E3BCFAE4A4B38FA36E884C586B5CB2C2C283E71FBA59EFDB9EA64BFC`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** Identify the URL within the file to use it as an IOC
{: .prompt-info }

![pestudio strings analysis](https://github.com/user-attachments/assets/3ddd959d-7374-4eca-a6d2-aa67ea69b6d2){: .shadow .rounded }
_Figure 6: Locating the hardcoded update URL within the executable strings._

> **Answer:** `http://tryhatme.com/update/security-update.exe`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** With the URL identified, can you spot a domain that can be used as an IOC?
{: .prompt-info }

This can be found in the file `windows-update.exe`{: .filepath} itself, and since the URL is almost the same (the difference is just the domain), I searched for the word "tryhatme" as a string in the file using the terminal:

```shell
strings ./windows-update.exe | findstr "tryhatme"
```
{: .nolineno }

![Terminal string search output](https://github.com/user-attachments/assets/68e31028-f45c-4745-9e95-0c5c7c0d5172){: .shadow .rounded }
_Figure 7: Grepping for the suspicious domain name._

> **Answer:** `responses.tryhatme.com`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** Input the decoded flag from the suspicious domain
{: .prompt-info }

There are 2 suspicious domains shown in the image from previous questions, the first one had a code at the end of the link. I used CyberChef to convert this code from Base64, and it displayed this flag:

![CyberChef Base64 Decoding](https://github.com/user-attachments/assets/b5614c54-9436-4813-a1b5-863345642704){: .shadow .rounded }
_Figure 8: Decoding the Base64 string in CyberChef._

> **Answer:** `THM{you_g0t_some_IOCs_friend}`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** When was the phishing kit archive first submitted? (format: YYYY-MM-DD HH:MM:SS UTC)
{: .prompt-info }

Returning to pestudio, there is a library tab that shows the libraries that the file uses. For the socket communication, it uses `WS2_32.dll`{: .filepath}.

![pestudio libraries tab](https://github.com/user-attachments/assets/91e43251-a890-48a6-9072-a9cc483c7d45){: .shadow .rounded }
_Figure 9: Investigating imported DLLs for network communication._

> **Answer:** `WS2_32.dll`
{: .prompt-tip }

<br>
<br>

---

<br>

## Alerts Analysis

<br>

> **Question:** Can you identify the malicious URL from the trigger by the process `powershell.exe`?
{: .prompt-info }

From the site viewed, there are two alerts. In the alert that includes `powershell.exe`, the command has a suspicious Base64 code:

![PowerShell Alert Details](https://github.com/user-attachments/assets/57eb6d77-ca7b-4bf3-9d9f-97ad62e66325){: .shadow .rounded }
_Figure 10: Identifying the obfuscated PowerShell execution policy bypass._

Converting this code back from base64 using CyberChef:

![CyberChef Malicious URL extraction](https://github.com/user-attachments/assets/fff8b866-36d6-465a-ae94-f3773b0a8b54){: .shadow .rounded }
_Figure 11: Decoded PowerShell command exposing the malicious download URL._

> **Answer:** `https://tryhatme.com/dev/main.exe`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** Can you identify the malicious URL from the alert triggered by `chrome.exe`?
{: .prompt-info }

In the second alert, there are malicious decimal places separated by commas that indicate a decimal conversion:

![Chrome Alert Details](https://github.com/user-attachments/assets/55939834-ac91-41c6-a3f2-c9825b6ac307){: .shadow .rounded }
_Figure 12: Decimal obfuscation detected in the Chrome process alert._

Converting them back using CyberChef using the from Decimal conversion and choosing the comma as a delimiter:

![CyberChef Decimal Conversion](https://github.com/user-attachments/assets/8c63515c-ef68-46f2-86fd-105742e3c088){: .shadow .rounded }
_Figure 13: Deobfuscating the decimal values to reveal the threat actor's email._

> **Answer:** `m3npat@yandex.com`
{: .prompt-tip }

<br>
<br>

---

<br>

> **Question:** What's the name of the file saved in the alert triggered by `chrome.exe`?
{: .prompt-info }

Looking back at the `chrome.exe` alert command to search for any files, `test.txt`{: .filepath} was found:

![File output identification](https://github.com/user-attachments/assets/7cc2be13-ec27-47aa-93c3-beac4a8c0e91){: .shadow .rounded }
_Figure 14: Identifying the output file generated by the alert._

> **Answer:** `test.txt`
{: .prompt-tip }