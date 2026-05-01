<img width="1883" height="801" alt="image" src="https://github.com/user-attachments/assets/4e7f6c9d-c9d9-446f-ba8e-478e63777862" />---
layout: default
title: "TryHackMe: Snapped Phish-ing Line Walkthrough"
description: "Detailed solution and walkthrough for the Snapped Phish-ing Line challenge on TryHackMe."

redirect_from: 
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line.html
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line
---


# Splunk 2
<img width="1892" height="370" alt="image" src="https://github.com/user-attachments/assets/00561f2a-8e2a-4def-984f-253aa6a3d40d" />

## BOTSv2 Dataset:

### The data included in this app was generated in August of 2017 by members of Splunk's Security Specialist team - Dave Herrald, Ryan Kovar, Steve Brant, Jim Apger, John Stoner, Ken Westin, David Veuve and James Brodsky. They stood up a few lab environments connected to the Internet. Within the environment they had a few Windows endpoints instrumented with the Splunk Universal Forwarder and Splunk Stream. The forwarders were configured with best practices for Windows endpoint monitoring, including a full Microsoft Sysmon deployment and best practices for Windows Event logging. The environment included a Palo Alto Networks next-generation firewall to capture traffic and provide web proxy services, and Suricata to provide network-based IDS. 

### BOTSv2 Github: https://github.com/splunk/botsv2

#### In this exercise, you assume the persona of Alice Bluebird, the analyst who successfully assisted Wayne Enterprises and was recommended to Grace Hoppy at Frothly (a beer company) to assist them with their recent issues.

### What Kinds of Events Do We Have?
#### The SPL (Splunk Search Processing Language) command metadata can be used to search for the same kind of information that is found in the Data Summary, with the bonus of being able to search within a specific index, if desired. All time-values are returned in EPOCH time, so to make the output user readable, the eval command should be used to provide more human-friendly formatting.


## Answer the questions below
>![](https://img.shields.io/badge/Question-blue) Amber Turing was hoping for Frothly to be acquired by a potential competitor which fell through, but visited their website to find contact information for their executive team. What is the website domain that she visited?


#### First, its known that the name is Amber, using this in the search 
```bash 
index="botsv2" amber
```
<img width="1907" height="679" alt="image" src="https://github.com/user-attachments/assets/5ad195cc-8cef-4dc6-bbd7-a78ce5d208ec" />

about 56,513 events shows up, browsing the `src_ip` field, multiple ips are found

<img width="1901" height="793" alt="image" src="https://github.com/user-attachments/assets/65469a35-5d10-48b6-a34f-fe0d46221c13" />
 browsing the logs more, a pan traffic could be seen which is Palo Alto Networks where in the corperates environment the Palo Alto firewall sits between the employer and the internet. 

Palo Alto has a feature called "User-ID", where they talk directly to the Active Directory server. Whenever Amber turns on her computer and logs in, the firewall records "User Amber is currently using the internal IP X.X.X.X

Using this feature, Amber's IP could directly be get through this SPL query 
```bash 
index="botsv2" amber sourcetype="pan:traffic"
```
<img width="1861" height="709" alt="image" src="https://github.com/user-attachments/assets/05e5860e-b66f-4b5c-8e55-79f72ac5b1fe" />

scrolling to see the src_ip field, its shown that there is only one source ip in the logs which is `10.0.2.101`
<img width="1883" height="801" alt="image" src="https://github.com/user-attachments/assets/4951bf07-1ff0-4d43-8100-f2a0a6839316" />

Using this info, the website domain can be found from combininig the IP address with the sourcetype of domains which is `stream:http`
```bash 
index="botsv2" 10.0.2.101 sourcetype="stream:http"
```
<img width="1900" height="799" alt="image" src="https://github.com/user-attachments/assets/e990a854-9248-4439-af32-fc37d4729e7d" />

still there is much events and site visited, since the required domain Amber used to find contact information for their executive team, it must be industry related! Amber is working in forthly which is a beer manufacturing company, so adding a keyword like `beer` would limit the logs to what its beeing searched 
```bash 
index="botsv2" 10.0.2.101 sourcetype="stream:http"
```
<img width="1889" height="801" alt="image" src="https://github.com/user-attachments/assets/18d1e11b-213a-4951-bff9-97d703851938" />
this limited the search to only 12 events, scrolling to explore the site field,
<img width="1862" height="808" alt="image" src="https://github.com/user-attachments/assets/29682388-f520-43ad-81b3-bfd89a409381" />


>![](https://img.shields.io/badge/Answer-success) www.berkbeer.com

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **Amber found the executive contact information and sent him an email. What image file displayed the executive's contact information? Answer example: /path/image.ext**

since Amber was browsing www.berkbeer.com to find the executive contact information, using the website domain will help find executives informations Amber found 
```bash
index="botsv2" www.berkbeer.com
```
<img width="1901" height="714" alt="image" src="https://github.com/user-attachments/assets/2dcd94ea-2b0c-4a95-b73f-b6d5e825c34c" />

Scrolling to the `filename` field, a list of `.png` files is seen, where one of the images name is `ceoberk`
<img width="1909" height="802" alt="image" src="https://github.com/user-attachments/assets/2231c7a1-8d40-4010-88df-14188076ce45" />

  <br>
  <br>
  
>![](https://img.shields.io/badge/Answer-success) /images/ceoberk.png

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the CEO's name? Provide the first and last name.**

The image name could reveal only the ceo last name, Assuming Amber opened the image and knew the first name then tried to cmmunicate with them through the email. Now the domain `berkbeer.com` is used again along with `smtp` traffic and converting all data to raw to facilitate the search
```bash
index="botsv2" berkbeer.com sourcetype="stream:smtp" 
|  table _time _raw
```
now searching for the name using `ctrl+F` and the keyword ` Berk`
<img width="1915" height="797" alt="image" src="https://github.com/user-attachments/assets/7acd668f-9feb-4eca-a0a6-73713b122ae0" />

<br> 

>![](https://img.shields.io/badge/Answer-success) Martin Berk
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the CEO's email address?**
using the same previous approach but search for `berk@` this time would show the ceo email 
<img width="1917" height="731" alt="image" src="https://github.com/user-attachments/assets/b6f70d27-3abf-42e0-9606-63a01478107f" />



>![](https://img.shields.io/badge/Answer-success) mberk@berkbeer.com
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **After the initial contact with the CEO, Amber contacted another employee at this competitor. What is that employee's email address?**

using the same approach, searching for `@berk` would provide any other emails from the same domain 
which was in this case only 

<img width="1910" height="784" alt="image" src="https://github.com/user-attachments/assets/7580ff27-c6ee-45eb-811d-b66ba0c25c30" />
<br> 
<br>

> ![](https://img.shields.io/badge/Answer-success) hbernhard@berkbeer.com
     
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the name of the file attachment that Amber sent to a contact at the competitor?**
Converting back the logs to a `show syntax highlighted` for better vision, in the 1st log of the same search there is a `attach_filename` if expanded, the attachment file is shown  
<img width="1906" height="719" alt="image" src="https://github.com/user-attachments/assets/37babaae-3a73-468a-8cb9-164811d5906a" />

<br> 
<br>

> ![](https://img.shields.io/badge/Answer-success)  Saccharomyces_cerevisiae_patent.docx

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is Amber's personal email address?**
In the same email log of previous question, its notices that there is a base64 bit encoding 
<img width="1906" height="768" alt="image" src="https://github.com/user-attachments/assets/3a471833-097f-4def-8096-d8ccc22f22cd" />

Taking a look at the content, there is an encoded text that might be suspicious
<img width="1891" height="730" alt="image" src="https://github.com/user-attachments/assets/ff0cf572-a6e7-40d8-9b27-5e74840f4509" />

Converting it back from base64 encoding using CyberChef 
<img width="1911" height="722" alt="image" src="https://github.com/user-attachments/assets/18aeb9d8-7af2-4d55-8b9a-714dae6220cf" />




> ![](https://img.shields.io/badge/Answer-success) ambersthebest@yeastiebeastie.com
<br>
<br>
<br>
<br>

### 200 series questions
> ![](https://img.shields.io/badge/Question-blue) **What version of TOR Browser did Amber install to obfuscate her web browsing? Answer guidance: Numeric with one or more delimiter.**


To know the version of tor, the tor keyword was used in the search along with `amber`
```bash
index="botsv2" amber tor
```

<img width="1908" height="662" alt="Screenshot 2026-04-28 200903" src="https://github.com/user-attachments/assets/0918d533-94b9-41c9-a759-0511c67b47c9" />

Browsing the interesting fields, the app field contains the version of the tor browser as shown 

<img width="1743" height="734" alt="Screenshot 2026-04-28 201029" src="https://github.com/user-attachments/assets/0f4dd824-eabd-46ea-af7c-5706c156125c" />



> ![](https://img.shields.io/badge/Answer-success) 7.0.4
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the public IPv4 address of the server running www.brewertalk.com?**

Since the requests sent to browse the website are of port 80 and the server is the destination port to the requests 
filtering the website link and the port 80 will show up the private and public IPs of the server hosting the website 
```bash
index="botsv2" www.brewertalk.com dest_port=80
```
<img width="1899" height="712" alt="Screenshot 2026-04-28 203007" src="https://github.com/user-attachments/assets/cb2c40de-edab-46eb-a501-2b3142e14ebc" />

Expolring the interesting fields, the Public IP of the server is found 
<img width="1906" height="786" alt="Screenshot 2026-04-28 203104" src="https://github.com/user-attachments/assets/df46e637-fc96-4489-a2dd-432ad81dca3e" />



> ![](https://img.shields.io/badge/Answer-success) 52.42.208.228
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **Provide the IP address of the system used to run a web vulnerability scan against www.brewertalk.com.**

Since tor was used to collect data from the website, the website name along with the keyword tor could be used to find the IP address running the scan 
```bash
index="botsv2" www.brewertalk.com tor
```
Expolring the src_ip field
<img width="1909" height="727" alt="Screenshot 2026-04-28 203436" src="https://github.com/user-attachments/assets/ae5d6301-2414-44dd-96df-78f8d9985c98" />

  
  > ![](https://img.shields.io/badge/Answer-success) 45.77.65.211


> ![](https://img.shields.io/badge/Question-blue) **The IP address from Q#2 is also being used by a likely different piece of software to attack a URI path. What is the URI path? Answer guidance: Include the leading forward slash in your answer. Do not include the query string or other parts of the URI. Answer example: /phpinfo.php**

Using the attacker IP revealed in the previous questions along with searching for logs that the `/*` where it indicates a path 
```bash
index="botsv2" src_ip="45.77.65.211" "/*"
```
<img width="1906" height="669" alt="Screenshot 2026-04-28 210610" src="https://github.com/user-attachments/assets/b49c6681-23af-46f3-a174-d81fb8b4ccbc" />

Expolring the uri_path field which indicated that `/member.php` was the most requested which can indicate that members info is being comprimised
<img width="1862" height="691" alt="Screenshot 2026-04-28 210821" src="https://github.com/user-attachments/assets/16792f1c-fc70-498c-8080-91ed4570c786" />


  
  > ![](https://img.shields.io/badge/Answer-success) /member.php

> ![](https://img.shields.io/badge/Question-blue) **What SQL function is being abused on the URI path from the previous question?**

Browsing the detals and content of the logs filtered from the `/member.php` path, there is a clear SQL query that uses a function
<img width="1897" height="668" alt="Screenshot 2026-04-28 210946" src="https://github.com/user-attachments/assets/d50a16ef-ee27-4c0b-a88c-a8400ab4dea1" />

  > ![](https://img.shields.io/badge/Answer-success) updatexml

> ![](https://img.shields.io/badge/Question-blue) **What was the value of the cookie that Kevin's browser transmitted to the malicious URL as part of an XSS attack? Answer guidance: All digits. Not the cookie name or symbols like an equal sign.**

As the XSS attack was mentioned, its obvious to include a related keyword like `script` which is used in XSS attack. 
Using the username kevin along with the script keyword 
```bash
index="botsv2" kevin *script
```
browsing the cookies field shows 4 cookies, the last one is suspcious because along with the sid, there is also a adminsid which indicates an escalation of privilages using the same cookie token 
<img width="1893" height="662" alt="Screenshot 2026-04-28 212508" src="https://github.com/user-attachments/assets/e240416d-39ac-4eeb-800e-f7ef3319a6d6" />


  > ![](https://img.shields.io/badge/Answer-success) 1502408189

> ![](https://img.shields.io/badge/Question-blue) **What brewertalk.com username was maliciously created by a spear phishing attack?**

Expanding the search to see what the cookie token was used for and filtering it out by clicking on the cookie 
<img width="1910" height="674" alt="image" src="https://github.com/user-attachments/assets/f9581aa2-d494-429f-aed5-f6b3e76f687b" />

Its required to get the username that was created by spear phising, so search for username in the logs
<img width="1909" height="722" alt="image" src="https://github.com/user-attachments/assets/3ef98ee6-1af4-419e-a503-b8fc651aab56" />


  > ![](https://img.shields.io/badge/Answer-success) kIagerfield
    
