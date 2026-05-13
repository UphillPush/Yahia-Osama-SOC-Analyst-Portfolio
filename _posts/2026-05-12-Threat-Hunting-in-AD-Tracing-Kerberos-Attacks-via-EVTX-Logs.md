---
layout: post
title: "Threat Hunting in AD: Tracing Kerberos Attacks via EVTX Logs"
date: 2026-05-12 00:00:00 +0200
categories: ["Threat Hunting & Triage", "Active Directory"]
tags: [evtx, jq, as-rep-roasting, kerberos, blue-team, log-analysis]
pin: false
image:
image:
  path: https://github.com/user-attachments/assets/eac7aab9-4fb8-4d66-9de9-167e95a6d886 
  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Hunting AS-REP Roasting in Active Directory


![Lab Header](https://github.com/user-attachments/assets/0d17d21e-a10c-4c2e-9881-0aa274a2941a){: .shadow .rounded }
_Figure 1: Active Directory Log Analysis Overview._

> **Sherlock Scenario**
> Forela's network is constantly under attack. The security system raised an alert about an old admin account requesting a ticket from the KDC on a domain controller. Inventory shows that this user account is not actively used. You are tasked with investigating this activity, as it may be an AS-REP Roasting attack—a technique where an attacker can request a ticket for any user account that has preauthentication disabled.
{: .prompt-warning }

---

## Phase 1: Setting Up the Environment 

Active Directory records events in binary files with the `.evtx` extension, which cannot be read directly with a standard text editor. While enterprise environments typically use Splunk or Elastic to investigate these logs, let's take a more lightweight approach to analyze Windows Active Directory logs without needing to deploy a massive SIEM.

The `evtx_dump` tool will be used to translate the `.evtx` file into `.json` format, which can then be easily read, filtered, and processed directly in the Linux terminal.

```shell
# Download the evtx_dump binary
curl -L -o evtx_dump [https://github.com/omerbenamram/evtx/releases/download/v0.8.1/evtx_dump-v0.8.1-x86_64-unknown-linux-gnu](https://github.com/omerbenamram/evtx/releases/download/v0.8.1/evtx_dump-v0.8.1-x86_64-unknown-linux-gnu)

# Give it execution permissions
chmod +x evtx_dump
```
{: .nolineno }

![Downloading evtx_dump](https://github.com/user-attachments/assets/ee11fbfe-0e11-4067-8a13-fd3e385c7a61){: .shadow .rounded }
_Figure 2: Downloading and preparing the evtx_dump tool._

Next, download the provided event logs from the challenge source:

```text
[https://labs.hackthebox.com/api/v4/challenges/736/cdn/redirect?auth_user_id=3496080&expires=1778628971&signature=739c208da7e6d3ef434e51c79697e41a813df6446d6b45789c0bf3e144fab7c7](https://labs.hackthebox.com/api/v4/challenges/736/cdn/redirect?auth_user_id=3496080&expires=1778628971&signature=739c208da7e6d3ef434e51c79697e41a813df6446d6b45789c0bf3e144fab7c7)
```
{: .nolineno }

Extract the downloaded archive using the password `hacktheblue`. We are now ready to begin the investigation.

### Translating the Logs to a Readable Format 

Use the `evtx_dump` tool to convert the `.evtx` file into a JSON Lines format using the following command:

```shell
./evtx_dump -o jsonl Security.evtx > Security.json
```
{: .nolineno }

*   `-o jsonl` tells the tool to format the output as JSON Lines.
*   `Security.evtx` is the input file.
*   `> Security.json` redirects all the output and creates the new JSON file in the current directory.

To verify the conversion actually worked, use the `wc -l` command to output the total number of lines in the new file:

```shell
wc -l Security.json
```
{: .nolineno }

![Checking Line Count](https://github.com/user-attachments/assets/45a57c06-f174-4cb3-b5ed-fb6607c0a0c2){: .shadow .rounded }
_Figure 3: Verifying the JSON file creation._

The Active Directory logs are now translated and ready for investigation. However, before diving into the data, we must clearly define what we are hunting for.

---

## Phase 2: Understanding AS-REP Roasting & Key Event IDs

Normally in Active Directory, when a user logs in, they ask the Domain Controller for a ticket. That ticket requires **Kerberos Preauthentication** to prove the user actually knows the password. 

However, some accounts in Active Directory (often legacy accounts or misconfigured service accounts) have the option *"Do not require Kerberos preauthentication"* checked. Attackers exploit this misconfiguration: the Domain Controller will hand the attacker a ticket containing the user's encrypted password without requiring any authentication first. The attacker then takes this ticket offline and brute-forces the encryption until they crack the password.

To hunt for this, we need to monitor two critical Event IDs:

*   **Event ID 4768 (A Kerberos authentication ticket was requested):** Inside this log, there is a field called `PreAuthType`. If its value is `2`, the user authenticated normally. If its value is `0`, the ticket was requested *without* preauthentication. 
*   **Event ID 4769 (A Kerberos service ticket was requested):** If the attacker successfully cracks the password, they will log back in and request access to shares, files, or printers. This generates an Event ID 4769.

---

## Phase 3: Hunting for the Attack

We can use `jq` to filter the `Security.json` file for all logs matching Event ID `4768` where the `PreAuthType` is `0`:

```shell
cat Security.json | jq '
.Event | select(
  .System.EventID == 4768 and
  .EventData.PreAuthType == "0"
)'
```
{: .nolineno }

*   `jq` processes the incoming text as a structured JSON object.
*   `.Event` tells the tool to ignore the outer layers and start directly in the event section.
*   `select()` filters out only the logs matching our specific AS-REP Roasting criteria.

![jq 4768 Query Output](https://github.com/user-attachments/assets/0c6fa4ff-328d-4963-b40b-0fb8d3bf07c5){: .shadow .rounded }
![jq 4768 Query Output Details](https://github.com/user-attachments/assets/c2860c6a-f277-4acd-bb8b-b7d7a4887dea){: .shadow .rounded }
_Figure 4: Isolating the AS-REP Roasting attempt using jq._

This query yields a wealth of valuable information regarding the attack:
*   **Time:** Gathered from the `SystemTime` field.
*   **Attacker IP:** `"IpAddress": "::ffff:172.17.79.129"`
*   **Targeted Account:** `"TargetUserName": "arthur.kyle"`

Because usernames and IPs can change during lateral movement, it is better to identify the targeted user by their **SID** (Security Identifier), as it uniquely identifies the object and serves as a strong Indicator of Compromise (IOC). We can extract just the SID by appending `.EventData.TargetSid` to our query:

```shell
cat Security.json | jq '
.Event | select(
  .System.EventID == 4768 and
  .EventData.PreAuthType == "0"
) | .EventData.TargetSid'
```
{: .nolineno }

![Extracting the SID](https://github.com/user-attachments/assets/10bd1b8c-6697-4800-81cd-fdcf87d8f4dd){: .shadow .rounded }
_Figure 5: Extracting the targeted user's SID._

> **Answer:** `S-1-5-21-3239415629-1862073780-2394361899-1601`
{: .prompt-tip }

Once the compromised SID is confirmed, an Incident Response plan must be executed:
1.  Disable or isolate the vulnerable user account.
2.  Uncheck the *"Do not require Kerberos preauthentication"* misconfiguration in Active Directory.
3.  Force a password reset for the user, assuming the password has already been cracked offline.

---

## Phase 4: Tracing the Initial Compromise (Pivot Point)

After containing the vulnerable account, it is critical to determine *where* the attacker gained access to prevent them from returning. Since the attacker was interacting with the Kerberos ticketing system, they were already inside the network (Kerberos cannot be queried anonymously from the outside). 

We need to discover which compromised user account the attacker used as their initial entry point. We can query the logs for activity originating from the attacker's known IP address (`172.17.79.129`), specifically looking for Service Ticket Requests (`4769`) and Network Share Access (`5140`):

```shell
cat Security.json | jq '.Event | select(
    (.System.EventID == 4769 or .System.EventID == 5140) and 
    .EventData.IpAddress == "::ffff:172.17.79.129"
) | {
    EventID: .System.EventID, 
    CompromisedUser: (.EventData.TargetUserName // .EventData.SubjectUserName), 
    Time: .System.TimeCreated
}'
```
{: .nolineno }

![Tracing the Compromised User](https://github.com/user-attachments/assets/5c3f7754-5ba5-41d7-89bb-26f06146fffa){: .shadow .rounded }
_Figure 6: Identifying the attacker's foothold account._

The output confirms that the user `happy.grunwald` was the attacker's entry point. The attacker used Grunwald's credentials to pivot and AS-REP Roast other accounts.

If only the `arthur.kyle` account is fixed, the attacker still maintains an open backdoor through the `happy.grunwald` user. Further investigation must be applied to Grunwald's account to discover the root cause of the compromise (e.g., checking email logs for phishing or reviewing external login history).

---

## Executive Summary

| **Field** | **Value** |
| :--- | :--- |
| **Attack Time (UTC)** | 2024-05-29 06:36:40 |
| **Vulnerable Account** | `arthur.kyle` |
| **Account SID** | `S-1-5-21-3239415629-1862073780-2394361899-1601` |
| **Compromised Pivot Account** | `happy.grunwald` |

### Threat Intelligence Mapping
*   **MITRE ATT&CK:** T1558.003 — Steal or Forge Kerberos Tickets: AS-REP Roasting
*   **Detection Query:** Event ID `4768` with `PreAuthType = 0` on any user account is a near-zero false positive indicator of an AS-REP Roasting attempt.

### Recommended Security Controls
1.  Enable Kerberos preauthentication on all Active Directory accounts globally.
2.  Configure SIEM alerts for Event `4768` where `PreAuthType = 0`.
3.  Conduct a routine AD audit to identify and remediate legacy accounts with preauthentication intentionally disabled.

<style>
  /* 1. Fix the "Outside" (Home Page Cards) - prevents cropping */
  .post-preview .preview-img img,
  .post-preview .preview-img {
    object-fit: contain !important;
    background-color: #1b1b1e !important;
  }

  /* 2. Fix the "Inside" - Completely hides the header image and its spacing */
  /* Using data-layout="post" guarantees this ONLY targets the opened post, never the home page */
  body[data-layout="post"] .post-meta + .mt-3.mb-3,
  body[data-layout="post"] .preview-img {
    display: none !important;
    visibility: hidden !important;
    height: 0 !important;
    margin: 0 !important;
  }
</style>
