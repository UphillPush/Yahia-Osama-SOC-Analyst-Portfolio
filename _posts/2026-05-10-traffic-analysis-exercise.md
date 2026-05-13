---
layout: post
title: "TRAFFIC ANALYSIS EXERCISE: DOWNLOAD FROM FAKE SOFTWARE SITE"
date: 2026-05-10 00:00:00 +0200
categories: ["Threat Hunting & Triage", "Traffic Analysis"]
tags: [wireshark, pcap, malware-analysis, blue-team, phishing]
pin: false
image:
  path: https://github.com/user-attachments/assets/1126305d-a895-4541-9062-8ac470bf521d



  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Snapped Phish-ing Line: Traffic Analysis

> **Scenario Background**
> You work as an analyst at a Security Operation Center (SOC). Someone contacts your team to report a coworker has downloaded a suspicious file after searching for Google Authenticator. You confirm there was an infection and retrieve a packet capture (PCAP) of the associated traffic. Reviewing the traffic, you find several indicators matching details from a recent threat intel report regarding malicious ads. After confirming the infection, you begin writing an incident report.
{: .prompt-warning }

### LAN Segment Details
* **LAN segment range:** `10.1.17.0/24`
* **Domain:** `bluemoontuesday.com`
* **Active Directory (AD) Domain Controller:** `10.1.17.2` (WIN-GSH54QLW48D)
* **AD Environment Name:** `BLUEMOONTUESDAY`
* **LAN Segment Gateway:** `10.1.17.1`
* **Broadcast Address:** `10.1.17.255`

---

## Phase 1: Endpoint Identification

> **Question:** What is the IP address of the infected Windows client?
{: .prompt-info }

To find the victim IP, the HTTP traffic is filtered to show `GET` requests. To narrow the search results, a `.exe` filter is added to the request URI to isolate HTTP requests that specifically target executable files.

~~~text 
http.request.method == "GET" && (http.request.uri contains ".exe")
~~~
{: .nolineno }

![HTTP GET Executable Filter](https://github.com/user-attachments/assets/7f9d8c89-1c4e-4c80-97f3-7e73eb3e2607){: .shadow .rounded }
_Figure 1: Filtering HTTP traffic for executable downloads._

From the results, it's concluded that the IP downloading the malicious executable is `10.1.17.215`.

> **Answer:** `10.1.17.215`
{: .prompt-tip }
<br>

---

> **Question:** What is the MAC address of the infected Windows client?
{: .prompt-info }

From the exact same packets found in the previous step, browsing the `Ethernet II` frame details in the lower pane reveals the hardware MAC address of the device.

![Ethernet II MAC Address](https://github.com/user-attachments/assets/707d42ed-375d-4108-849e-f27a40b6ed09){: .shadow .rounded }
_Figure 2: Extracting the MAC address from the Ethernet frame._

> **Answer:** `00:d0:b7:26:4a:74`
{: .prompt-tip }
<br>

---

> **Question:** What is the host name of the infected Windows client?
{: .prompt-info }

In a security or network analysis context, searching for Kerberos traffic is a highly effective way to identify the hostname of a device because Kerberos is heavily dependent on naming services. Filtering the packets by the `kerberos` protocol and the victim's IP reveals this data.

~~~text 
kerberos && ip.src == 10.1.17.215
~~~
{: .nolineno }

![Kerberos Hostname](https://github.com/user-attachments/assets/b3f5b437-a6ce-40d3-80fb-b46ce023ee35){: .shadow .rounded }
_Figure 3: Identifying the hostname via Kerberos tickets._

> **Answer:** `DESKTOP-L8C5GSJ`
{: .prompt-tip }
<br>

---

> **Question:** What is the user account name of the infected Windows client?
{: .prompt-info }

Filtering the packets by Kerberos and the victim IP reveals all related authentication packets. However, to easily extract the actual user account name, we can specifically filter for the `AS-REQ` (Authentication Service Request), which is the initial login asking for a Ticket Granting Ticket.

~~~text 
kerberos.as_req_element && ip.src == 10.1.17.215
~~~
{: .nolineno }

![Kerberos User Account](https://github.com/user-attachments/assets/24c9b099-613f-47bb-a017-1fe07b04ca6c){: .shadow .rounded }
_Figure 4: Extracting the Active Directory user account from AS-REQ._

> **Answer:** `shutchenson`
{: .prompt-tip }
<br>

---

## Phase 2: Threat Infrastructure Analysis

> **Question:** What is the likely domain name for the fake Google Authenticator page?
{: .prompt-info }

It's known from the scenario that the malicious fake page was impersonating Google Authenticator. Searching for DNS packets originating from the victim IP that contain the word "google" using the following filter revealed the initial redirect domain:

~~~text
dns && ip.src == 10.1.17.215 && dns.qry.name contains "google"
~~~
{: .nolineno }

However, attackers frequently use these initial domains strictly to redirect the victim to the actual malicious payload domain. By removing the "google" keyword and observing the chronological DNS requests:

~~~text
dns && ip.src == 10.1.17.215
~~~
{: .nolineno }

![DNS Query Analysis](https://github.com/user-attachments/assets/f5bfcb2c-fa51-4507-acd5-cfa7eb68622d){: .shadow .rounded }
_Figure 5: Analyzing chronological DNS queries for redirects._

Another highly suspicious domain is revealed. It was requested just milliseconds after the initial Google-related domain query.

> **Answer:** `authenticatoor.org`
{: .prompt-tip }
<br>

---

> **Question:** What are the IP addresses used for C2 servers for this infection?
{: .prompt-info }

A helpful built-in Wireshark tool that can quickly reveal high-volume C2 communications is **Statistics -> Conversations**. 

![Wireshark Conversations](https://github.com/user-attachments/assets/41554679-2355-41e1-95a6-9a33889ce74f){: .shadow .rounded }
_Figure 6: Accessing the Conversations statistics tool._

By viewing the IPv4 conversations and sorting them by the number of packets, it is observed that the two highest-volume external communications are with the following IPs:
* `5.252.153.241`
* `45.125.66.32`

![Top Talkers](https://github.com/user-attachments/assets/57e7f43a-295b-491a-8f86-7c9aff180f99){: .shadow .rounded }
_Figure 7: Sorting by packet count to find primary C2 servers._

Now, by sorting the IPs by "Address B" to reveal if there is another IP related to them on the exact same network subnet, the IP `45.125.66.252` is discovered. It has 466 packets and 39 kB of data spread over 2,283 seconds, suggesting a lower-volume but persistent C2 beaconing communication.

![Subnet Pivot](https://github.com/user-attachments/assets/1e0d3a95-73dc-4f60-9bc4-6a51b0cc00a2){: .shadow .rounded }
_Figure 8: Finding the persistent secondary beacon._

> **Answer:** `5.252.153.241, 45.125.66.32, 45.125.66.252`
{: .prompt-tip }
<br>

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
