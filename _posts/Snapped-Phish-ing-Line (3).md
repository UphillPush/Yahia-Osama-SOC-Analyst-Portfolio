---
layout: default
title: "TryHackMe: Snapped Phish-ing Line Walkthrough"
description: "Detailed solution and walkthrough for the Snapped Phish-ing Line challenge on TryHackMe."

redirect_from: 
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line.html
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line
---


# FortiGate IPsec VPN & HA


## For the security part, it is divided into 3 labs
- lab 0: installing the software and configuring staic routes 
- Lab 1: IP sec VPN
- lab 2: HA + IPsec 

#### the two hands-on labs built from scratch: a Site-to-Site IPsec VPN, then a FortiGate High Availability cluster with seamless VPN failover. Both run free on EVE-NG, with full protocol-level explanations of every setting and why it matters — not just what to click.
### You now proceeded to investigate what is going on by:

Lab 0: 
phase 1: installing the software
for this lab, eve is used along with a fortigate images of version 7.0.2, which could be downloaded through the link: https://o6uedu-my.sharepoint.com/:u:/g/personal/202018756_o6u_edu_eg/IQC__SSmdVDUTYBEUVxpeetWAR54lUoPi3ST3ZmOzvmnpfI?e=XyFTDW

using the vmware, the eve vm is imported. For the fortigate images to be inported in eve, winscp application was used and the fortigate files are imported 

Oprning the cli with the following commands 
ssh to the vm
```
ssh root@<eve-ng-ip>   # default password: eve 
```
Create the image directory
```
mkdir -p /opt/unetlab/addons/qemu/fortinet-7.0.2
cd /opt/unetlab/addons/qemu/fortinet-7.0.2
```
now using the winscp, the fortigate images are dragged and dropped in the path `/opt/unetlab/addons/qemu/fortinet-7.0.2`
<img width="1339" height="811" alt="image" src="https://github.com/user-attachments/assets/130811a6-6c03-48e9-8ca5-b372f6881fd4" />
Last thing is fixing the permissions 
```
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
now after making sure that eve connection is bridge, its powered on and got an ip, browsing this ip in the browsers opens the eve interface 
using the credentials
user: admin
pass: eve
the web interface is logged in and ready to add nodes
Adding a fortigate router as a node is done by clicking a right-click in empty space and choosing fortigate
<img width="861" height="323" alt="image" src="https://github.com/user-attachments/assets/1f9d101b-2cae-40f5-a48e-69918a7b02c8" />
for this lab and fortigate version, only 1 cpu and 1 GB of RAM is needed 
<img width="889" height="652" alt="image" src="https://github.com/user-attachments/assets/a737e765-e80f-4193-b9b0-c9ab4bfa16f2" />
Other nodes are added and connected trough ports as the topology shows:
<img width="418" height="390" alt="image" src="https://github.com/user-attachments/assets/62d216b0-c4e1-4ff1-8657-0a46c328ddaf" />


For using the fortigate cli, some programs needs to be installed 
- Eve windoes client (which is included in the attachement downloaded)
- Putty (through this link: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

Now all the nodes are selected and pressed start selected for configuration of the nodes with the IPs assigned in the topology:
For the Fortinet-Left: 
When first entered it asks for the username and password 
username is admin and password is blanck (just pressing enter), then it promotes the user to enter a new password

after setting the new password, the hostname is changed for better identification 
```
config system global
set hostname "Forti-Left"
end
```
<img width="481" height="131" alt="image" src="https://github.com/user-attachments/assets/faf488f6-d828-40c3-9716-011caddd65d9" />
for accessing the Fortinet GUI, the IP addressed assigned in the cloud management port is identenfied through the commands
```
get system interface physical
```
<img width="473" height="170" alt="image" src="https://github.com/user-attachments/assets/e21d1518-5421-4f32-99f5-16d86952c4f7" />
which shows the IP of the GUI of the fortigate router, which when browsed opens the GUI 
the username and the password is the same set in the beging of the cli 
after logging it it shows this interface 
<img width="1360" height="680" alt="image" src="https://github.com/user-attachments/assets/535c8f81-0156-42eb-ba19-1da5587cc23f" />
The interfaces IP could be set either through the GUI or the CLI, but for this lab the cli is used. 
For setting the IP for each interface 
in the Forti-Left router:
```
config system interface
    edit port2
        set mode static
        set ip 172.16.10.1 255.255.255.252
        set allowaccess ping https ssh http
    next
    edit port3
        set mode static
        set ip 10.10.10.1 255.255.255.0
        set allowaccess ping https ssh http
    next
end
```
<img width="479" height="367" alt="image" src="https://github.com/user-attachments/assets/58b8f2a0-b60d-43ab-82e4-af3f45e53e7d" />

For the Forti-Right: 
the same steps in the Forti-Left is applied beside the difference in the IPs on each interface 
```
config system global
  set hostname "Forti-Left"
end
config system interface
    edit port2
        set mode static
        set ip 172.16.10.2 255.255.255.252
        set allowaccess ping https ssh http
    next
    edit port3
        set mode static
        set ip 20.20.20.1 255.255.255.0
        set allowaccess ping https ssh http
    next
end
```
<img width="705" height="474" alt="image" src="https://github.com/user-attachments/assets/062d3c8a-4b20-42e2-9067-a5c7e0e047d1" />

now setting the VPCs IP, 
for the left VPC 
```
ip 10.10.10.2 24 10.10.10.1
```
for the right VPC 
```
ip 20.20.20.2 24 20.20.20.1
```

now each VPC can see its directly connected router, but they can't see each other. To achieve this a static route and firewall policy have to be set 
Accessing the left router's GUI, in the Network -> static route a new rule is created as shown containing the destination network which is 20.20.20.0/24 and the gateway which is the interface address of the router connected to that network and the interface that the traffic will pass through, which is port2
<img width="921" height="554" alt="image" src="https://github.com/user-attachments/assets/4beae9e7-e869-4d22-8403-0b55503d62a6" />
Now the router can see the other network, but forigate policy is denying any communication to it. To resolve this a new policies have to be set. In `Policy & Objects` -> `Firewall Policy` 2 new policies are created, one to  allow the trafic inbount and another outboand.
the first one is to allow traffic incoming from port 2 to the port 3 
<img width="918" height="682" alt="image" src="https://github.com/user-attachments/assets/e956f443-b46d-4699-8b39-1dc703c7f3ae" />

Second is doing the opposite 
<img width="910" height="681" alt="image" src="https://github.com/user-attachments/assets/85b1c112-b218-4802-99b9-e04b6a9fcda6" />

Doing the same steps in Fortigate-Right:
the static route:
<img width="913" height="553" alt="image" src="https://github.com/user-attachments/assets/a0b5ff28-f61c-4e2f-95ea-f80bd7b01d74" />

the policies:
<img width="911" height="677" alt="image" src="https://github.com/user-attachments/assets/4ff22b5a-0e13-4d6a-9e98-f54a6f456f30" />

<img width="899" height="677" alt="image" src="https://github.com/user-attachments/assets/d522fc81-9c06-41c7-b08a-3a34bb112168" />

Finally this could be tested by pinging the right VPC from the left VPC as shown 
<img width="624" height="190" alt="image" src="https://github.com/user-attachments/assets/d4ce746a-525d-4efb-8809-e768577dee81" />

and the opposite is working too 
<img width="607" height="184" alt="image" src="https://github.com/user-attachments/assets/8187f6ea-e952-4c8c-96a0-6a6d9cbe246d" />

LAB 02: IPsec VPN
For setting the IKE phase 1 through the fortigate GUI, 
VPN -> IPsec Wizard -> Custom -> a name is give (eg. "ToRemote")
<img width="1010" height="285" alt="image" src="https://github.com/user-attachments/assets/ac6da439-8b57-4d74-a8d7-7624429a753e" />
set the parameters
Paramater | Forti-Left | Forti-Right 
IP version | static IP | static IP 
Remote Gateway | 172.16.10.2 | 172.16.10.1
Interface | port2 | port2
IKE version | 1 | 1
Mode | main | main 
Authentication Method | pre-shared key | pre-shared key 
Pre-shared key | FortiGate@123! | FortiGate@123! 
Encryption | DES | DES
Authentication | SHA256 | SHA256
DH Group | 14 | 14
Dead Peer Detection | On Demand | On Demand
Nat Traversal | Enable | Enable 

This phase 1 configurations maps to the IKE phase 1 three-step process 
- Step 1: Negotiation, where the Encryption, Authentication, DH Group. life time is selected which are the SA proposal exchanged 
- step 2: DH key exchange
- step 3: Authentication 

For the Phase 2 selectors: 
Paramater | Forti-Left | Forti-Right 
Local subnet | 10.10.10.0/24 | 20.20.20.0/24
Remote subnet | 20.20.20.0/24 | 10.10.10.0/24
Protocol | ESP | ESP
Encryption | DES | DES 
Authentication | SHA256 | SHA256
Enable PFS | checked | checked 
Lifetime | 43200 | 43200
Auto-negotiate | enabled | enabled 

<img width="1366" height="2464" alt="image" src="https://github.com/user-attachments/assets/bd13bb72-34b4-422c-aa3c-909ffb7d8ced" />





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



    
