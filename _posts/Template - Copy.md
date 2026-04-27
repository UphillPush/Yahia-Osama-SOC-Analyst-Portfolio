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



    
