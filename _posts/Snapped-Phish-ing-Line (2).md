---
layout: post
title: "BARQ Systems Network and Cybersecurity Training"
date: 2026-05-24 00:00:00 +0200
categories: ["Home Lab", "Network Security"]
tags: [junos, pnet-lab, ospf, acl, firewall, routing, blue-team]
pin: false
image:
  path: https://github.com/user-attachments/assets/a05f0b00-19e8-4dd9-aa7b-b838c002a21b
  preview: false
  class: img-contain
---

# BARQ systems Network and cybersecurity training labs 
This trainig labs is divided by two parts, one for the network and other for the  network security  
The Network part will include configuring junos with ospf prrotocol  
then building an ACL rules  
 
The second part will be on fortigate where in the first lab , 

# Lab 0 – Junos images and Pnet lab installation  
First of all download the vmware workstation through this link  
 
then download the pnet lab where the scheme is drawn and implemented through this link  
open the pnet lab .ovf file from VMware workstation  
the settings of the image then is set to  
4 GB RAM  
4 CPUs  
Network adapter 1 (bridged) 

Network adapter 2 Host only  

Disk 50GB  

Powering on the vm the pnet lab console is then seen asking for credentials which are  
root  
pnet  
then the vm takes an IP in the network which is browesed to open the web interface of pnet lab using http://[PNETLAB_IP] 

 

After setting the vm, now the junos vm needs to be impported, its first downloaded through the link : vMX.ova 
 

Then download the 7-zip to extract its internal file through the link: https://7-zip.org/download.html  

The file vMX-disk1.vmdk needs to be imported to the vm of pnet lab  
 
This could be done using the WinSCP which is downloaded through this link: https://winscp.net/eng/download.php  

When opening it, the pnet lab IP, username and password is entered to access it  
 <img width="804" height="532" alt="Screenshot 2026-06-12 164000" src="https://github.com/user-attachments/assets/a13c9f40-45cb-497e-a512-f91895d65f1b" />

then the path of the right panel is then  changes through clicking to the /opt/unetlab/addons/qemu/ path  
new folder is created with the name vmx-x.x depending on the version  

The vMX-disk1.vmdk is then moved to this new folder 
 
A new cmd is opened to convert it to suitable format using the ssh protocol  

ssh root@[PNETLAB_IP] 
the vmdk is converted to qcow2 (Pnetlab format) 

qemu-img convert -f vmdk junos-vmx.vmdk -O qcow2 hda.qcow2 
then the premission is fixed  

/opt/unetlab/wrappers/unl_wrapper -a fixpermissions 


 Now in PNET lab if and new file is created, the vMX optopn is now seen when adding the nodes  
<img width="1172" height="442" alt="Screenshot 2026-06-12 164716" src="https://github.com/user-attachments/assets/1af94c2e-22c1-4f73-874d-090131787fab" />

Clicking on it, there are some setting needs ti be set:  
template: Juniper vMX  
image: vmx-x.x 
RAM: 2048 
CPUs: 2 
console: telnet 
Ethernet: 4

<img width="853" height="908" alt="Screenshot 2026-06-12 164947" src="https://github.com/user-attachments/assets/964a77dd-472e-432e-80c1-d23a47530aed" />

 use the same settings to add three vMX nodes 
 
VPCs and cloud for the management are adeded and connected as the following topology shows  
<img width="1061" height="548" alt="Screenshot 2026-06-12 170630" src="https://github.com/user-attachments/assets/93bf26b3-33e6-49b4-bff8-662e6e0ac03d" />

now configuring each router  
 

Router 1 is double clicked to enter its CLI  
The default username is root and pass is blank (Enter), after that a new password is set  
first the cli is accessed through the command cli  

Cli  

#entering the configuration mode  
config  
 
#before committing, a root password must be set  
set system root-authentication plain-text-password 
#and then set the new password  

#ssh is enabled   
set system services ssh 

#Changing the router name to R1  
set system host-name R1 

#creating a loginuser  
set system login user admin class super-user authentication plain-text-password 

#the interface ip for the management is set  
set interfaces lo0 unit 0 family inet address 1.1.1.1/32 
 
#the interface ip for the link to R2 is set 
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/30 

#the interface ip for the link to VPC is set 
set interfaces ge-0/0/2 unit 0 family inet address 192.168.10.1/24 
 
#commmit and verify  
commit and-quit 

show interfaces terse 
 
the interfaces is then show  

<img width="1132" height="609" alt="Screenshot 2026-06-13 022546" src="https://github.com/user-attachments/assets/e1da15ec-0a66-45d9-b316-9e639fa2a9f6" />

For router 2, the same configurations are made according to the topology  
Cli  

#entering the configuration mode  
config  
 
#before committing, a root password must be set  
set system root-authentication plain-text-password 
#and then set the new password  

#ssh is enabled   
set system services ssh 

#Changing the router name to R2  
set system host-name R2 

#creating a loginuser  
set system login user admin class super-user authentication plain-text-password 

#the interface ip for the management is set  
set interfaces lo0 unit 0 family inet address 2.2.2.2/32 
 
#the interface ip for the link to R1 is set 
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.2/30 

#the interface ip for the link to R3 is set 
set interfaces ge-0/0/1 unit 0 family inet address 10.0.23.2/30 
 
#commmit and verify  
commit and-quit 

show interfaces terse 
 
the interfaces is then show 
<img width="1232" height="487" alt="Screenshot 2026-06-13 022806" src="https://github.com/user-attachments/assets/a3e8121e-99dc-4dc5-893e-59af74290e56" />

The same for R3  
Cli  

#entering the configuration mode  
config  
 
#before committing, a root password must be set  
set system root-authentication plain-text-password 
#and then set the new password  

#ssh is enabled   
set system services ssh 

#Changing the router name to R3  
set system host-name R3 

#creating a loginuser  
set system login user admin class super-user authentication plain-text-password 

#the interface ip for the management is set  
set interfaces lo0 unit 0 family inet address 3.3.3.3/32 
 
#the interface ip for the link to R2 is set 
set interfaces ge-0/0/0 unit 0 family inet address 10.0.23.2/30 
 
#commmit and verify  
commit and-quit 

show interfaces terse 
<img width="1183" height="401" alt="Screenshot 2026-06-13 023059" src="https://github.com/user-attachments/assets/7ad87701-c94f-4661-bcd5-3f599f090ce9" />

Testing the initial configuration  
the direct connection are tested through ping service to make sure the interface are correctly assigned with the suitable IPs  

After testing, all the  direct connections can see each other  

For the network to see other indirect connection, the ospf (open shortest path first) protocol is implemented  using the following commands  

For R1 (Area 0 and 2): 

configure 
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 
set protocols ospf area 0.0.0.0 interface lo0.0 passive 
commit 

 

For R2 (Area 0 and 1):  

configure 
# Area 0 — link to R1 
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 
# Area 1 — link to R3 
set protocols ospf area 0.0.0.1 interface ge-0/0/1.0 
set protocols ospf area 0.0.0.1 interface lo0.0 passive 
commit 

 

For R3 (Area 1) 

configure 
set protocols ospf area 0.0.0.1 interface ge-0/0/0.0 
set protocols ospf area 0.0.0.1 interface lo0.0 passive 
commit 

 

Verification Commands 

# Check OSPF neighbours (should show FULL state) 
show ospf neighbor 
 
# View the routing table (inet.0) 
show route 
 
# View only OSPF routes 
show route protocol ospf 
 
# Check forwarding table 
show route forwarding-table 
 
# Ping loopback of R3 from R1 (should work end-to-end) 
ping 3.3.3.3 source 1.1.1.1 count 5 
 
# Traceroute to verify path through R2 
traceroute 3.3.3.3 source 1.1.1.1 

Now all devices can see each others over networks, this could be tested by pinging hosts and devices with different networks 




## Lab 2 - setting Firewall
### Scenario A block a compromised host machine 

Assume a SOC recieves alert that host 192.168.10.2 is beaconing C2, it should be blocked from the router automatically without touching the host  
in R1  

 
configure 
 
# Block all traffic from the Linux VM IP — log and count every dropped packet 
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 from source-address 192.168.10.2/32 
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then count HOST-BLOCKED 
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then log 
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then discard 
 
# Default: allow everything else 
set firewall family inet filter BLOCK-HOST term ALLOW-REST then accept 
 
# Apply inbound on the LAN interface (traffic coming FROM Linux into R1) 
set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-HOST 
 
commit 

This commands blocks all the traffic coming from the linux VM along with logging and counting the dropped packets for further investigation analysis  

The ALLOW-REST allows the traffic for the rest of the devices in the network which have IP other than 192.168.10.2 because the block rule is implicit deny all  

The last part is applying these rules to the interface of the R1 connected to the network 
 
after testing, the linux machine couldn’t ping hosts  

<img width="684" height="163" alt="Screenshot 2026-06-13 115937" src="https://github.com/user-attachments/assets/823fd8d4-216f-4304-9845-cc39e05bd961" />
While in R1 the packets are logged and counted  
<img width="1624" height="611" alt="Screenshot 2026-06-13 120020" src="https://github.com/user-attachments/assets/04cccf51-ffa9-4b0c-baf3-7d51dec33ad7" />

After resolving the incident the connection could be restored unassigning the rule to the interface  

configure 
delete interfaces ge-0/0/2 unit 0 family inet filter input 
commit

<img width="644" height="223" alt="Screenshot 2026-06-13 120224" src="https://github.com/user-attachments/assets/d522b1e3-ade4-4be6-93ed-dab53d4e08d4" />

### Senario B (Allow only the HTTP/ HTTPs and block other ports)
In corporate policy, lan users may only use browsers to browse the internet, not SSH or DNS queries are needed or even a custom ports. In this case all other traffic are loged and dropped  

To implement this on R1 

configure 
 
# Allow ICMP (ping) — useful for diagnostics 
set firewall family inet filter WEB-ONLY term ALLOW-ICMP from protocol icmp 
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then count ICMP-COUNT 
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then accept 
 
# Allow DNS (port 53) — required for browser to resolve domain names 
set firewall family inet filter WEB-ONLY term ALLOW-DNS from protocol udp 
set firewall family inet filter WEB-ONLY term ALLOW-DNS from destination-port 53 
set firewall family inet filter WEB-ONLY term ALLOW-DNS then count DNS-COUNT 
set firewall family inet filter WEB-ONLY term ALLOW-DNS then accept 
 
# Allow HTTP and HTTPS only 
set firewall family inet filter WEB-ONLY term ALLOW-WEB from protocol tcp 
set firewall family inet filter WEB-ONLY term ALLOW-WEB from destination-port [80 443] 
set firewall family inet filter WEB-ONLY term ALLOW-WEB then count WEB-COUNT 
set firewall family inet filter WEB-ONLY term ALLOW-WEB then accept 
 
# Block everything else — log it 
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then count DENIED-COUNT 
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then log 
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then discard 
 
# Apply inbound on LAN interface 
set interfaces ge-0/0/2 unit 0 family inet filter input WEB-ONLY 
 
commit 
 
testing the firewall in linux, its observed that the pc can use DNS and HTTP services but not telnet or other services  

<img width="696" height="349" alt="Screenshot 2026-06-13 191922" src="https://github.com/user-attachments/assets/21b9bcad-3894-4092-b0c8-71747ec40037" />

The count  discarded and dns packets is also recorded in the R1 and could be shown using  
```
show firewall filter WEB-ONLY
```

<img width="1240" height="341" alt="Screenshot 2026-06-13 192243" src="https://github.com/user-attachments/assets/04a5ca2e-27b9-4a9c-bcb8-81f2cbdfd64b" />

And the logs show tcp and udp protocols only when represeting it through the command  
```
show firewall logs  
```
<img width="1887" height="805" alt="Screenshot 2026-06-13 192500" src="https://github.com/user-attachments/assets/d7998cf5-1b0c-4761-8d21-0a408a5f2343" />

### Senario C (Block a specific destination IP):  
in real word whenever a known malicious IP is found, it had to be blocked to ensure network savety. Taking the google dns id as an example, it could be blocked with the following commands in the R1  

configure 
 
# Block all traffic destined to 8.8.8.8 (Google DNS — used as demo target) 
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 from destination-address 8.8.8.8/32 
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then count DEST-BLOCKED 
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then log 
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then discard 
 
# Allow all other destinations 
set firewall family inet filter BLOCK-DEST-IP term ALLOW-REST then accept 
 
# Apply on LAN interface 
set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-DEST-IP 
 
commit 
 

Testing the blockage on the linux pc using ping 8.8.8.8 its observed that there are no packets recieved 

<img width="634" height="94" alt="Screenshot 2026-06-13 193548" src="https://github.com/user-attachments/assets/8cbdd5ed-a818-401a-bf37-d3b1b35315b9" />

While it successfully pings other IPs 
<img width="581" height="198" alt="Screenshot 2026-06-13 193822" src="https://github.com/user-attachments/assets/39c25265-7a65-40be-ac54-1ebae0ccec70" />

Viewing the logs and counts in the R1,  

Show firewall  
which views the count of all firewall rules applied 
<img width="1415" height="536" alt="Screenshot 2026-06-13 193923" src="https://github.com/user-attachments/assets/a91d3ad6-8ff3-447c-9476-283152ba9b4f" />





#### As an IT department personnel of SwiftSpend Financial, one of your responsibilities is to support your fellow employees with their technical concerns. While everything seemed ordinary and mundane, this gradually changed when several employees from various departments started reporting an unusual email they had received. Unfortunately, some had already submitted their credentials and could no longer log in.

### You now proceeded to investigate what is going on by:

1. Analysing the email samples provided by your colleagues.
1. Analysing the phishing URL(s) by browsing it using Firefox.
1. Retrieving the phishing kit used by the adversary.
1. Using CTI-related tooling to gather more information about the adversary.
1. Analysing the phishing kit to gather more information about the adversary.

Note: The phishing emails to be analysed are under the phish-emails directory on the Desktop. Usage of a web browser, text editor and some knowledge of the grep command will help.

## Answer the questions below
>![](https://img.shields.io/badge/Question-blue) Who is the individual who received an email attachment containing a PDF?



#### First, open the phish-emails dir on the desktop and right-click `open terminal here`, then we need to search for the file that contains the pdf attachment. This can be achieved through the command

```bash 
grep -l 'name=".*\pdf"' *.eml
```
This will return the name of the eml file that has a pdf attachment. We want to find out who received that pdf. The command is tunneled with `xargs` to view how this email was sent to in the command below

```bash 
grep -l 'name=".*\pdf"' *.eml | xargs -d '\n' "To:"
```

<img width="100%"  alt="code of answer1" src="https://github.com/user-attachments/assets/f9a4c4a7-1dde-4586-b319-f2bcbf498b41" />
<br>
<br>

>![](https://img.shields.io/badge/Answer-success) William McClean

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What email address was used by the adversary to send the phishing emails?**
#### The sender name of all files can be displayed with grep command too, as follows
```bash
grep -h -A 1 "^From:" *.eml"
```
Where `-h` is used to hide the file name   
&emsp; &emsp;&emsp;`-A` for displaying the line after the search keyword and the lines after, depending on the number follows <br> 
&emsp;&emsp;  &emsp;   `1` number of lines to display after the line match <br>   

  <img width="624"  alt="Picture3" src="https://github.com/user-attachments/assets/707fd174-9ef8-4df1-b6fa-45b6344ba6f5" />
  <br>
  <br>
  
>![](https://img.shields.io/badge/Answer-success) Accounts.Payable@groupmarketingonline.icu

<br> 
<br>
<br> 
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the redirection URL to the phishing page for the individual Zoe Duncan? (defanged format)**
#### For this question, we have to open the email sent to Zoe and download the attached html to inspect it 
<img width="1795" alt="Screenshot 2026-01-18 175342" src="https://github.com/user-attachments/assets/e0be1594-7286-4eef-a214-3aff404add25" />
<img width="1798" alt="Screenshot 2026-01-18 175412" src="https://github.com/user-attachments/assets/da8fb663-f7ff-4d8c-999f-8cebc27fa606" />

<br><br>
Now, the html file must be inspected for any url inside it, which can be done through the commnad 
```bash
grep "http[^ ]" *.html
```
<img width="1132"  alt="Screenshot 2026-01-18 175932" src="https://github.com/user-attachments/assets/2fecd1a0-3253-4df7-b2e4-30704c38a282" />
<br> 
<br>
So the URL is:   

http://kennaroads.buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe.duncan@swiftspend.finance&error    

  
This URL has to be defanged as required in the question, so Cyberchef was used to do it: 
<br> 
<br> 
<img width="100%"  alt="Picture4" src="https://github.com/user-attachments/assets/83df2aca-2f4f-46bd-b081-dafb5ddfdbc3" />   
<br> 
>![](https://img.shields.io/badge/Answer-success) hxxp[://]kennaroads[.]buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe[.]duncan@swiftspend[.]finance&error
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the URL to the .zip archive of the phishing kit? (defanged format)**
#### The phishing URL sent to Zeo has a specific path to her, but if we remove the path related to the user, it will direct us to some files it's hosting 
so for the link:   
http://kennaroads.buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe.duncan@swiftspend.finance&error  
we will just keep the main path:   
http://kennaroads.buzz/data/Update365/  
<img width="624" height="312" alt="Picture5" src="https://github.com/user-attachments/assets/80a14ab3-5ad4-4aaf-8bd6-227cbfae57d0" />
<br>
<br>
There is no `.zip` file here, so going back a bit to the link:  
http://kennaroads.buzz/data    

show us this page   

   <img width="624" height="303" alt="Picture6" src="https://github.com/user-attachments/assets/e66c4b1f-14e2-4d7c-b9e4-7a3944e648c4" /> 
   <br>
   <br> 
   Now, we can clearly see the `.zip` archive of the phishing kit, which is Update365.zip
   so the link we are looking for is:   
   http://kennaroads.buzz/data/Update365.zip    
   
   defang this URL using CyberChef:   
<img width="1430" height="695" alt="Picture7" src="https://github.com/user-attachments/assets/3d22b28d-70d0-4290-9310-5a1de12ed48b" />
<br>
<br>

>![](https://img.shields.io/badge/Answer-success) hxxp[://]kennaroads[.]buzz/data/Update365[.]zip
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What is the SHA256 hash of the phishing kit archive?**
#### Download the Update365.zip in the phish-emails dir, then in the terminal write the command:   
```bash
sha256sum Update365.zip
```

<img width="624" height="232" alt="Picture8" src="https://github.com/user-attachments/assets/d5a3d8b7-99a4-46ef-9898-689ec866dec5" />
<br> 
<br>

> ![](https://img.shields.io/badge/Answer-success) ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686
     
<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **When was the phishing kit archive first submitted? (format: YYYY-MM-DD HH:MM:SS UTC)**
#### Using an open-source, popular tool for investigating viruses, VirusTotal is the best option.   
In my browser, I visited virustotal page and inserted the archive sha256 hash to scan it, and easily saw the submission date  
<img width="100%"  alt="image" src="https://github.com/user-attachments/assets/0dd5610f-7af5-4716-9f1a-aa2217ffffeb" />

<br> 
<br>

> ![](https://img.shields.io/badge/Answer-success) 2020-04-08 21:55:50 UTC

<br>
<br>
<br>
<br>

> ![](https://img.shields.io/badge/Question-blue) **What was the email address of the user who submitted their password twice?**
#### The best way to track the login attempts is through logs, and in previous questions its know where are the hosting files  
so back again to the link:   
http://kennaroads.buzz/data    

a `log.txt` file is seen, open it and try to look up for repeated email or password     

<img width="975"  alt="image" src="https://github.com/user-attachments/assets/1c7fb47b-d887-417a-8015-05320392bb11" />  

> ![](https://img.shields.io/badge/Answer-success) michael.ascot@swiftspend.finance
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



    
