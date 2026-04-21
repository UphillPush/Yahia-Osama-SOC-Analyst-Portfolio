---
layout: post
title: "Phishing Email Forensics: Snapped Phish-ing Line"
date: 2026-04-19 14:30:00 +0200
categories: [Investigations, SOC]
tags: [phishing, email-headers, tryhackme, malware-analysis]
---

# Snapped Phish-ing Line

![Snapped Phish-ing Line Header](https://github.com/user-attachments/assets/3cd4145c-8211-42ce-88f9-a5e57fbf2d84){: .shadow .rounded }
_Figure 1: Investigation Overview._

## Scenario

As IT department personnel at **SwiftSpend Financial**, you are tasked with investigating a surge in unusual emails. Several employees have reported credential theft after interacting with these messages.

**Investigation Steps:**
1. Analyze `.eml` samples for malicious indicators.
2. Inspect phishing URLs and identify redirection chains.
3. Retrieve and reverse-engineer the adversary's phishing kit.
4. Utilize CTI tools (VirusTotal) to gather adversary intelligence.

---

## Phase 1: Email Header & Attachment Analysis

> **Question:** Who is the individual who received an email attachment?
{: .prompt-info }

I started by navigating to the `phish-emails`{: .filepath} directory on the Desktop. I used `grep` to identify which `.eml` file contained a PDF attachment, then piped the result to find the recipient.

~~~bash 
grep -l 'name=".*\.pdf"' *.eml | xargs grep "To:"
~~~
{: .nolineno }

![Email recipient identification](https://github.com/user-attachments/assets/f9a4c4a7-1dde-4586-b319-f2bcbf498b41){: .shadow .rounded }

> **Answer:** `William McClean`
{: .prompt-tip }
<br>

---

> **Question:** What email address was used by the adversary to send the phishing emails?
{: .prompt-info }

I pulled the sender addresses from all email files using a targeted regex search.

~~~bash
grep -h "^From:" *.eml
~~~
{: .nolineno }

![Sender email extraction](https://github.com/user-attachments/assets/707fd174-9ef8-4df1-b6fa-45b6344ba6f5){: .shadow .rounded }

> **Answer:** `Accounts.Payable@groupmarketingonline.icu`
{: .prompt-tip }
<br>

---

## Phase 2: Phishing URL Investigation

> **Question:** What is the redirection URL to the phishing page for the individual Zoe Duncan? (Defanged format)
{: .prompt-info }

I inspected the email sent to Zoe and extracted the HTML attachment to find the underlying URL.

~~~bash
grep "http[^ ]" *.html
~~~
{: .nolineno }

!https://onlinetexttools.com/extract-text-from-html(https://github.com/user-attachments/assets/2fecd1a0-3253-4ded-a426-810462196632){: .shadow .rounded }

The raw URL was defanged using **CyberChef** for safe reporting.

![CyberChef Defanging](https://github.com/user-attachments/assets/83df2aca-2f4f-46bd-b081-dafb5ddfdbc3){: .shadow .rounded }

> **Answer:** `hxxp[://]kennaroads[.]buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe[.]duncan@swiftspend[.]finance&error`
{: .prompt-tip }
<br>

---

## Phase 3: Phishing Kit Forensics

> **Question:** What is the URL to the `.zip` archive of the phishing kit? (Defanged format)
{: .prompt-info }

By traversing up the directory on the hosting server at `http://kennaroads.buzz/data/`, I discovered the source files for the phishing campaign.

![Server directory listing](https://github.com/user-attachments/assets/e66c4b1f-14e2-4d7c-b9e4-7a3944e648c4){: .shadow .rounded }

> **Answer:** `hxxp[://]kennaroads[.]buzz/data/Update365[.]zip`
{: .prompt-tip }
<br>

---

> **Question:** What is the SHA256 hash of the phishing kit archive?
{: .prompt-info }

After downloading `Update365.zip`{: .filepath}, I calculated the hash via the terminal:

~~~bash
sha256sum Update365.zip
~~~
{: .nolineno }

![Hash calculation](https://github.com/user-attachments/assets/d5a3d8b7-99a4-46ef-9898-689ec866dec5){: .shadow .rounded }

> **Answer:** `ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686`
{: .prompt-tip }
<br>

---

> **Question:** When was the phishing kit archive first submitted? (format: YYYY-MM-DD HH:MM:SS UTC)
{: .prompt-info }

Searching the SHA256 hash on **VirusTotal** provided the initial submission timestamp.

![VirusTotal Submission Date](https://github.com/user-attachments/assets/0dd5610f-7af5-4716-9f1a-aa2217ffffeb){: .shadow .rounded }

> **Answer:** `2020-04-08 21:55:50 UTC`
{: .prompt-tip }
<br>

---

## Phase 4: Adversary Intelligence

> **Question:** What was the email address of the user who submitted their password twice?
{: .prompt-info }

I reviewed the `log.txt`{: .filepath} file on the adversary's server to track login attempts.

![Log file analysis](https://github.com/user-attachments/assets/1c7fb47b-d887-417a-8015-05320392bb11){: .shadow .rounded }

> **Answer:** `michael.ascot@swiftspend.finance`
{: .prompt-tip }
<br>

---

> **Question:** What was the email address used by the adversary to collect compromised credentials?
{: .prompt-info }

I unzipped the phishing kit and searched for PHP files responsible for data exfiltration. `submit.php` was found to be the main handler.

~~~bash
grep -r "mail(" Update365/
~~~
{: .nolineno }

![Source code analysis](https://github.com/user-attachments/assets/fa5c1788-f72f-4d80-985b-d78edfb768f2){: .shadow .rounded }

> **Answer:** `m3npat@yandex.com`
{: .prompt-tip }
<br>

---

> **Question:** The adversary used other email addresses in the obtained phishing kit. What is the email address that ends in "@gmail.com"?
{: .prompt-info }

~~~bash
grep -r "@gmail.com" .
~~~
{: .nolineno }

![Gmail address extraction](https://github.com/user-attachments/assets/88218dd2-ad78-4a3f-bb85-7f103140ebc8){: .shadow .rounded }

> **Answer:** `jamestanner2299@gmail.com`
{: .prompt-tip }
<br>

---

> **Question:** What is the hidden flag?
{: .prompt-info }

Suspecting a hidden directory, I navigated to `flag.txt` on the phishing host. The content was Base64 encoded, which I decoded using CyberChef.

![Decoded Flag](https://github.com/user-attachments/assets/53228822-dc45-47b2-9479-ef89ee5fd601){: .shadow .rounded }


> **Answer:** `THM{pL4y_w1Th_tH3_URL}`
{: .prompt-tip }
<br>