---
layout: post
title: "SIEM Investigations: Alert Triage With Splunk Write-up"
date: 2026-04-22 00:00:00 +0200
categories: ["Threat Hunting & Triage", "SIEM Investigations"]
tags: [splunk, alert-triage, brute-force, web-shell, blue-team]
pin: true
image:
  path: https://github.com/user-attachments/assets/6fdc392c-efc5-45a2-ac61-72a87afc3227

  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Alert Triage With Splunk

## Alert Scenario 1


---

## Phase 1: SSH Brute Force Investigation

> **Question:** How many failed login attempts were made on the user john.smith?
{: .prompt-info }

From the info in the alert, it's obvious that the IP is `10.10.242.248`. Using this IP for our search with the `failed password` keyword and selecting the username `john.smith` will give the number of failed login attempts.

~~~splunk
index="linux-alert" sourcetype=linux_secure 10.10.242.248   
| search "Failed password" user_name="john.smith"
~~~
{: .nolineno }

![Failed login attempts count](https://github.com/user-attachments/assets/41831dce-4019-43b6-a870-1fed8b387ea9){: .shadow .rounded }
_Figure 2: Filtering for failed passwords on john.smith._

> **Answer:** `500`
{: .prompt-tip }
<br>

---

> **Question:** What was the duration of the brute force attack in minutes?
{: .prompt-info }

The whole brute force attack includes three cases: 
1. Invalid user 
2. Wrong password 
3. Correct password 

Selecting the suspected IP and username and searching for these 3 keywords will give all the logs of the brute force attempt. For the duration, `range` is used to calculate the difference between the oldest and the newest event internally.

~~~splunk
index="linux-alert" sourcetype="linux_secure" 10.10.242.248 user_name="john.smith"
| search "Failed password" OR "Accepted password" OR "Invalid user" 
| stats count range(_time) as duration by user_name
~~~
{: .nolineno }

![Duration calculation](https://github.com/user-attachments/assets/30d4b7b1-7f4e-4337-a837-a8c2cddd02d3){: .shadow .rounded }
_Figure 3: Using the range command to find attack duration._

> **Answer:** `5`
{: .prompt-tip }
<br>

---

> **Question:** What username was the attacker able to privilege escalate to?
{: .prompt-info }

To escalate to another user, the basic command to write is `su`, where you can switch to another user through it. Searching for it gives the logs related to user switching.

~~~splunk
index="linux-alert" sourcetype="linux_secure" su
~~~
{: .nolineno }

![Privilege escalation log](https://github.com/user-attachments/assets/5d40b8b1-2c76-4c7d-a4c8-558062f446ce){: .shadow .rounded }
_Figure 4: Tracking the su command usage._

> **Answer:** `root`
{: .prompt-tip }
<br>

---

> **Question:** What is the name of the user account created by the attacker for persistence?
{: .prompt-info }

Creating a new user could be done through the `adduser` command.

~~~splunk
index="linux-alert" sourcetype="linux_secure" adduser
~~~
{: .nolineno }

![New user creation log](https://github.com/user-attachments/assets/055b0d54-053d-445f-ba48-c0b20de72b9c){: .shadow .rounded }
_Figure 5: Locating the persistent account creation._

> **Answer:** `system-utm`
{: .prompt-tip }
<br>

---

## Alert Scenario 2

![Scenario 2 Alert details](https://github.com/user-attachments/assets/6d04f860-154f-49d1-b984-5fdfe2cca1c6){: .shadow .rounded }
_Figure 6: Windows endpoint alert for malicious scheduled task._

---

## Phase 2: Malicious Scheduled Task Analysis

> **Question:** What is the ProcessId of the process that created this malicious task?
{: .prompt-info }

The process ID could be known by selecting the process name in the alert and searching for the command `create`.

~~~splunk
index="win-alert" AssessmentTaskOne 
| search "create"
| table host ProcessId Task_Name Message
~~~
{: .nolineno }

![ProcessId identification](https://github.com/user-attachments/assets/9145c8c5-5d6d-496d-b19d-a969c5d9b2e2){: .shadow .rounded }
_Figure 7: Identifying the process responsible for task creation._

> **Answer:** `5816`
{: .prompt-tip }
<br>

---

> **Question:** What is the name of the parent process for the process that created this malicious task?
{: .prompt-info }

Scrolling down the previous image, the parent process ID was 4128, so adding this attribute to the query will tell the parent process name.

~~~splunk 
index="win-alert" ProcessId=4128
| search "create"
| table host ProcessId Task_Name Message
~~~
{: .nolineno }

![Parent process tracking](https://github.com/user-attachments/assets/ea2e6ed5-d263-4675-93da-bf2c3891c802){: .shadow .rounded }
_Figure 8: Discovering the parent process._

> **Answer:** `cmd.exe`
{: .prompt-tip }
<br>

---

> **Question:** Which local group did the attacker enumerate during discovery?
{: .prompt-info }

The discovery was right before execution of the malicious process, which was the process ID 4128. Using the same search attributes but presenting the command line commands it executed will show what the attacker enumerated.

~~~splunk
index="win-alert" ParentProcessId=4128
| table _time ParentProcessId CommandLine
~~~
{: .nolineno }

![Group enumeration command](https://github.com/user-attachments/assets/efca3eaf-67a6-4774-bb05-d30ec65b46e2){: .shadow .rounded }
_Figure 9: Command-line arguments revealing group enumeration._

> **Answer:** `Administrators`
{: .prompt-tip }
<br>

---

> **Question:** What is the name of the workstation from which the Threat Actor logged into this host?
{: .prompt-info }

Using the alert info, the user that the attacker logged in with was `oliver.thompson`. To find all the logon logs, Event Code `4624` is used.

~~~splunk
index="win-alert" oliver.thompson EventCode=4624
| table _time Workstation_Name Source_Network_Address
~~~
{: .nolineno }

![Logon events tracking](https://github.com/user-attachments/assets/120e3067-45b7-4363-825d-514650c76925){: .shadow .rounded }
_Figure 10: Filtering Event ID 4624 for the compromised account._

> **Answer:** `DEV-QA-SERVER`
{: .prompt-tip }
<br>

---

## Alert Scenario 3

![Scenario 3 Alert details](https://github.com/user-attachments/assets/4f03a85c-02f1-4b6d-8608-5b99f8c32287){: .shadow .rounded }
_Figure 11: Web attack alert highlighting suspicious activity._

---

## Phase 3: Web Shell & Hydra Investigation

> **Question:** What time did the brute-force activity using Hydra begin? (Answer Format Example: 2025-01-15 12:30:45)
{: .prompt-info }

Using the IP address mentioned in the alert, a table could be made and sorted with time to view the logs from oldest to newest and start with the first Hydra appearance in the useragent.

~~~splunk
index=web-alert 171.251.232.40
| table _time clientip useragent uri_path method status
| sort + _time
~~~
{: .nolineno }

![Hydra start time](https://github.com/user-attachments/assets/9059c4e8-fdcc-41a0-8724-c8a2b108e238){: .shadow .rounded }
_Figure 12: Sorting the timeline to identify initial Hydra usage._

> **Answer:** `2025-09-14 21:20:27`
{: .prompt-tip }
<br>

---

> **Question:** Which user agent did the attacker use when interacting with the web shell?
{: .prompt-info }

For displaying the shell interaction logs, the Hydra logs are excluded. Then the logs are sorted to view the oldest logs right after Hydra.

~~~splunk
index=web-alert 171.251.232.40 useragent!="Mozilla/5.0 (Hydra)"
| table _time clientip useragent uri_path method status
| sort + _time
~~~
{: .nolineno }

![User agent tracking](https://github.com/user-attachments/assets/d3c85141-5f31-4b8a-b378-7580c8588509){: .shadow .rounded }
_Figure 13: Identifying the attacker's actual browser User-Agent._

> **Answer:** `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36`
{: .prompt-tip }
<br>

---

> **Question:** What was the number of requests made by the attacker to the server via the web shell?
{: .prompt-info }

Using the same query but showing the `referer` field this time, where the referer tells the server the URL of the previous webpage from which a link was followed or a request was initiated. That could help in detecting which file generates this shell.

~~~splunk
index=web-alert 171.251.232.40 useragent!="Mozilla/5.0 (Hydra)"
| table _time clientip useragent uri_path referer method status
| sort + _time
~~~
{: .nolineno }

![Referer field tracking](https://github.com/user-attachments/assets/8ab82dfb-d975-45d5-af20-a2338f3615e3){: .shadow .rounded }
_Figure 14: Inspecting the referer field for the source page._

It's noticed that there is a malicious php file. Searching for this file assured that it was an open-source webshell tool. Now, filtering the requests made by this file:

~~~splunk
index=web-alert 171.251.232.40 b374k.php method="POST"
| table _time clientip useragent uri_path referer method status
| sort - _time
~~~
{: .nolineno }

![Web shell interaction count](https://github.com/user-attachments/assets/99b77793-b185-46e0-a402-bef6ed0868d2){: .shadow .rounded }
_Figure 15: Isolating POST requests to the b374k.php web shell._

> **Answer:** `4`
{: .prompt-tip }
<br>


<style>
  /* 1. Fix the "Outside" (Home Page Card) - prevents cropping */
  .post-preview .preview-img img {
    object-fit: contain !important;
    background-color: #1b1b1e; /* Matches Chirpy's dark background */
  }

  /* 2. Fix the "Inside" - hides the banner completely */
  #post-wrapper > img:first-of-type,
  #post-wrapper img.preview-img,
  header + .preview-img,
  header + img,
  .post-content > img:first-child {
    display: none !important;
  }
</style>
