---
layout: post
title: "SIEM & Memory Forensics: Boogeyman 2 Triage"
date: 2026-04-21 00:00:00 +0200
categories: ["Threat Hunting & Triage", "SIEM Investigations"]
tags: [volatility, memory-forensics, c2-detection, incident-response, blue-team]
pin: true
image:
  path: https://github.com/user-attachments/assets/9ceb6565-a501-478e-82d6-ed23bbc221b6

  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Boogeyman 2

![Boogeyman 2 Room Banner](https://github.com/user-attachments/assets/bf141f87-3e32-4675-a6f9-35e2e9c7f0ab){: .shadow .rounded }
_Figure 1: The Boogeyman is back! Challenge overview._

## Scenario

> Maxine, a Human Resource Specialist working for Quick Logistics LLC, received a malicious application that compromised her workstation. The security team flagged suspicious commands, and you must now assess the impact of the breach.
{: .prompt-warning }

![Security team alert details](https://github.com/user-attachments/assets/59fb264e-b9d1-4e28-b0ee-f13bf55e4110){: .shadow .rounded }
_Figure 2: Alerts triggering the investigation._

---

## Phase 1: Phishing Analysis

> **Question:** What email was used to send the phishing email?
{: .prompt-info }

I opened the `.eml`{: .filepath} file found in the Desktop artifacts to inspect the headers.

![EML header analysis](https://github.com/user-attachments/assets/35a4996a-45e4-49fd-9178-e735d30a341e){: .shadow .rounded }

> **Answer:** `westaylor23@outlook.com`
{: .prompt-tip }
<br>

---

> **Question:** What is the email of the victim employee?
{: .prompt-info }

In the same `.eml`{: .filepath} file, the "To" field identifies the targeted staff member.

![Victim identification](https://github.com/user-attachments/assets/424df624-1ab1-435e-bde0-24f2de4bbe34){: .shadow .rounded }

> **Answer:** `maxine.beck@quicklogisticsorg.onmicrosoft.com`
{: .prompt-tip }
<br>

---

> **Question:** What is the name of the attached malicious document?
{: .prompt-info }

![Attachment extraction](https://github.com/user-attachments/assets/c817d8d7-77cb-4222-ac9a-90b71a7d2142){: .shadow .rounded }

> **Answer:** `Resume_WesleyTaylor.doc`
{: .prompt-tip }
<br>

---

> **Question:** What is the MD5 hash of the malicious attachment?
{: .prompt-info }

Navigating to the `Downloads`{: .filepath} folder, I used the terminal to hash the file.

```bash
md5sum Resume_WesleyTaylor.doc
```
{: .nolineno }

![MD5 calculation](https://github.com/user-attachments/assets/1b03a7bf-7242-469d-8351-1fc9b97875a9){: .shadow .rounded }

> **Answer:** `52c4384a0b9e248b95804352ebec6c5b`
{: .prompt-tip }
<br>

---

## Phase 2: Stage 2 Payload Analysis

> **Question:** What URL is used to download the stage 2 payload based on the document's macro?
{: .prompt-info }

Using the `strings` utility on the `.doc` file, I searched for URI patterns.

![Strings HTTP search](https://github.com/user-attachments/assets/078396da-bd50-40a0-aa6c-4855cc70ecfb){: .shadow .rounded }

> **Answer:** `https://files.boogeymanisback.lol/aa2a9c53cbb80416d3b47d85538d9971/update.png`
{: .prompt-tip }
<br>

---

> **Question:** What is the name of the process that executed the newly downloaded stage 2 payload?
{: .prompt-info }

I refined my `strings` search for `.exe` parameters.

![Strings EXE search](https://github.com/user-attachments/assets/cf858e67-f7f0-4f07-9f8e-16243b2131d3){: .shadow .rounded }

> **Answer:** `wscript.exe`
{: .prompt-tip }
<br>

---

> **Question:** What is the full file path of the malicious stage 2 payload?
{: .prompt-info }

![Payload path identification](https://github.com/user-attachments/assets/e095b83b-785a-49c2-bbf8-4640f1523a1f){: .shadow .rounded }

> **Answer:** `C:\ProgramData\update.js`
{: .prompt-tip }
<br>

---

## Phase 3: Volatility Memory Forensics

> **Question:** What is the PID of the process that executed the stage 2 payload?
{: .prompt-info }

I utilized **Volatility 3** to analyze the `WKSTN-2961.raw`{: .filepath} memory dump. Using the `pstree` plugin, I located the `wscript.exe` execution.

```bash
vol -f WKSTN-2961.raw windows.pstree | grep wscript
```
{: .nolineno }

![Volatility pstree output](https://github.com/user-attachments/assets/7bda262a-915d-40d0-b90c-77969bf756a9){: .shadow .rounded }

> **Answer:** `4260`
{: .prompt-tip }
<br>

---

> **Question:** What is the parent PID of the process that executed the stage 2 payload?
{: .prompt-info }

Expanding the `pstree` view allows for identifying the process parentage.

![Parent PID identification](https://github.com/user-attachments/assets/de6253c6-07ba-41fc-add0-10dfabd4aec4){: .shadow .rounded }

> **Answer:** `1124`
{: .prompt-tip }
<br>

---

> **Question:** What URL is used to download the malicious binary executed by the stage 2 payload?
{: .prompt-info }

The `updater.exe` process was observed following `wscript.exe`. I performed a `filescan` to find the Virtual Address and then dumped the file to extract the URL.

```bash
vol -f WKSTN-2961.raw windows.filescan | grep update
vol -f WKSTN-2961.raw windows.dumpfiles --virtaddr 0xe58f836edc60
```
{: .nolineno }

![Volatility filescan and dump](https://github.com/user-attachments/assets/9a88012d-6a34-406e-b778-201fa817c445){: .shadow .rounded }

> **Answer:** `https://files.boogeymanisback.lol/aa2a9c53cbb80416d3b47d85538d9971/update.exe`
{: .prompt-tip }
<br>

---

## Phase 4: C2 & Persistence

> **Question:** What is the PID of the malicious process used to establish the C2 connection?
{: .prompt-info }

Using the `netscan` plugin, I identified an active network connection associated with the updater.

![Volatility netscan output](https://github.com/user-attachments/assets/c590e197-f9d5-4951-87a4-f8d75288f8d3){: .shadow .rounded }

> **Answer:** `6216`
{: .prompt-tip }
<br>

---

> **Question:** What is the full file path of the malicious process used to establish the C2 connection?
{: .prompt-info }

I checked the command-line arguments using the `cmdline` plugin.

![Volatility cmdline output](https://github.com/user-attachments/assets/878d9f96-bb10-4d39-9080-dee0a6feb899){: .shadow .rounded }

> **Answer:** `C:\Windows\Tasks\updater.exe`
{: .prompt-tip }
<br>

---

> **Question:** What is the IP address and port of the C2 connection initiated by the malicious binary?
{: .prompt-info }

![C2 connection extraction](https://github.com/user-attachments/assets/0eee69a9-97be-4f04-b184-43c6033421c0){: .shadow .rounded }

> **Answer:** `128.199.95.189:8080`
{: .prompt-tip }
<br>

---

> **Question:** What is the full file path of the malicious email attachment based on the memory dump?
{: .prompt-info }

```bash
vol -f WKSTN-2961.raw windows.filescan | grep Resume
```
{: .nolineno }

![Volatility filescan for Resume](https://github.com/user-attachments/assets/d49784e1-579e-4b1d-9f9e-9017d5e8a421){: .shadow .rounded }

> **Answer:** `C:\Users\maxine.beck\AppData\Local\Microsoft\Windows\INetCache\Content.Outlook\WQHGZCFI\Resume_WesleyTaylor (002).doc`
{: .prompt-tip }
<br>

---

> **Question:** The attacker implanted a scheduled task for persistence. What is the full command used?
{: .prompt-info }

I searched for the `schtasks` keyword within the memory strings.

![Persistence command extraction](https://github.com/user-attachments/assets/69d28703-167f-4f7c-8353-dbce11eadee6){: .shadow .rounded }

> **Answer:** `schtasks /Create /F /SC DAILY /ST 09:00 /TN Updater /TR 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c \"IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKCU:\Software\Microsoft\Windows\CurrentVersion debug).debug)))\"'`
{: .prompt-tip }
<br>

<style>
  /* Hides the top banner inside the post */
  #post-wrapper > img:first-of-type,
  .preview-img,
  header + img {
    display: none !important;
  }
</style>

<style>
  /* 1. Fix the "Outside" (Home Page Card) - prevents cropping */
  .post-preview .preview-img img {
    object-fit: contain !important;
    background-color: #1b1b1e; /* Matches Chirpy's dark background */
  }

  /* 2. Fix the "Inside" - hides the banner completely */
  #post-wrapper img.preview-img,
  header + .preview-img,
  .post-content > img:first-child {
    display: none !important;
  }
</style>
