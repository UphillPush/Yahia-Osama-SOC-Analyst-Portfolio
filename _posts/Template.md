---
layout: default
title: "TryHackMe: Snapped Phish-ing Line Walkthrough"
description: "Detailed solution and walkthrough for the Snapped Phish-ing Line challenge on TryHackMe."

redirect_from: 
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line.html
  - /TryHackMe/Challenges/Snapped-Phish-ing-Line
---


# Snapped-Phish-ing-Line
<img width="1903"  alt="header" src="https://github.com/user-attachments/assets/3cd4145c-8211-42ce-88f9-a5e57fbf2d84" />

## Set up the virtual machines 

#### For this Lab, two Virtual machines was used where one is ubuntu server where the logs are gathered and the other is a windows virtual machine where advanced analysis and search is done on the logs 

Setting up the Windows machine
Give the machine a name and select the `.iso` file of the OS to be installed 
<img width="752" height="696" alt="image" src="https://github.com/user-attachments/assets/00006630-69bb-456b-88b4-886249488530" />

The username and password for the machine are set as well 
<img width="752" height="703" alt="image" src="https://github.com/user-attachments/assets/06384b5d-f348-49a7-af23-9a9a2bd6bbf6" />

For the hardware, 4Gb and 3 cores are enough to run the programs used in the lab 
<img width="755" height="699" alt="image" src="https://github.com/user-attachments/assets/62a89474-9702-4c80-9cd0-2a47f44631f7" />

For the Hard disk, 50GB or less is enough 
<img width="751" height="697" alt="image" src="https://github.com/user-attachments/assets/c60dcf3e-7e21-4f82-b259-e7f947a75cce" />

Then finish was clicked to start the virtual machine 
<img width="737" height="649" alt="image" src="https://github.com/user-attachments/assets/6228e593-3a96-4464-929d-54aac0623e21" />

Its important to make sure that Network settings is set to bridged Adapter and Name is Realtek PCIe if the host machine is connected through a LAN and Wireless if connected through the WiFi 
<img width="658" height="532" alt="image" src="https://github.com/user-attachments/assets/eb7e2125-483f-45ad-a5f8-c94da15ef197" />

After the machine boots up, the elastic search link is browsed to download it through 
`https://www.elastic.co/downloads/past-releases/elasticsearch-8-13-0`
<img width="724" height="572" alt="image" src="https://github.com/user-attachments/assets/37611df6-231e-4d4c-b4fb-362823e7e8e3" />

After the download, the .zip folder is extracted in C
Before running, the configuration file `config\elasticsearch.yml` has these commands to be added to it 
```yml
network.host: 0.0.0.0
discovery.type: single-node
```
<img width="693" height="527" alt="image" src="https://github.com/user-attachments/assets/b475570d-4bb5-4a2f-a32e-8d264207547a" />

Where, network.host: 0.0.0.0 allows Elasticsearch to accept connections from other machines on the network, not just localhost.
      xpack.security.enabled: false disables authentication for the home lab environment to simplify the setup.

After saving the file and closing it, the `elasticseach.bat` file is run in the cmd with the followig commands 
```bash
cd C:\elasticsearch-8.13.0\bin
elasticsearch.bat
```
<img width="725" height="499" alt="image" src="https://github.com/user-attachments/assets/422f17d4-4cff-49a9-984e-866895244cc4" />

Browsing `127.0.0.1:9200` will direct to the elastic search and ask for a login
When elastic starts it sets a username and password in the cmd, to reset it open another cmd and type 
```bash
cd C:\elasticsearch-8.13.0\bin
elasticsearch-reset-password -u elastic
```
<img width="705" height="180" alt="image" src="https://github.com/user-attachments/assets/72d57742-a798-4dfa-85e9-a1e7eff61d83" />

This prints a new password that could be copied and used to login 
<img width="731" height="533" alt="image" src="https://github.com/user-attachments/assets/98ed9c77-eb19-407e-9c88-e24da17e1d59" />
<img width="733" height="570" alt="image" src="https://github.com/user-attachments/assets/bdfa1082-8ab4-4c02-b425-a38e63bda6d4" />

Now after configuring the elastic seach, the Kibana dashboard needs to be set up too to visualize the logs and provide an interactive gui for analysis 

Downloading kibana from the link below and extracting it in C as well 
`https://www.elastic.co/downloads/past-releases/kibana-8-13-0`
<img width="724" height="570" alt="image" src="https://github.com/user-attachments/assets/838f731b-130a-4435-898b-c0f54e5cf47e" />

Kibana also has a password that need to be got from ealstic search through the following cmd:
```bash
cd C:\elasticsearch-8.13.0\bin
elasticsearch-reset-password -u kibana_system
```

<img width="689" height="144" alt="image" src="https://github.com/user-attachments/assets/c8af2bcc-f94c-4d7f-bc6d-503748bf4c97" />

The password is copied to be used later in the configuration file

For the kibana configuration file `kibana.yml` locatedin the config folder, there also some changes need to be made 
Adding these lines in kibana.yml 
```yml
elasticsearch.hosts: ["http://127.0.0.1:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "<password coppied from the cmd>"
elasticsearch.ssl.verificationMode: none
```
<img width="687" height="530" alt="image" src="https://github.com/user-attachments/assets/c5bbe22e-950d-4e57-9939-576fbc2e4ad6" />

Then the file is saved and kibana is started through the cmd: 
```bash
cd C:\kibana-8.13.0\bin
kibana.bat
```
<img width="714" height="573" alt="image" src="https://github.com/user-attachments/assets/4a25817e-03bc-4f00-9a71-139b3e45d7e5" />

<img width="721" height="572" alt="image" src="https://github.com/user-attachments/assets/2082d3c6-ec8c-41cf-93fb-ee4129310ca9" />

When the `kibana is now available` is seen in the cmd, browsing the link `127.0.0.1:5601` would give access to the kibana dashboard 
Signing in with the elastic credentials, the dashboard was accessed 
<img width="729" height="576" alt="image" src="https://github.com/user-attachments/assets/791ecb20-82b4-4a83-860b-3f4992b4b923" />

<img width="732" height="581" alt="image" src="https://github.com/user-attachments/assets/8e2c9576-b09d-4edc-b732-9f89ae5ad31b" />

After successfully setting up elastic search and kibana, its finally time to set up the agents that will be monitored 

### Setting up the Agent 
for this lab an ubuntu server virtual machine was chosen to be the agent 
A new VM was created with 2Gb and 2 cores along with the ubuntu server .iso image 
<img width="752" height="697" alt="image" src="https://github.com/user-attachments/assets/942fb6aa-2c38-4617-91b6-b53404081a43" />
<img width="757" height="700" alt="image" src="https://github.com/user-attachments/assets/495c0c5d-01cf-4654-bd39-525810596feb" />

The two virtual machines have to be in the same network, so the same network configuration is applied 
<img width="661" height="540" alt="image" src="https://github.com/user-attachments/assets/a771c4ba-da03-42b6-b7d5-440c092c83eb" />

The elastic agent could be sownloaded on the ubuntu server using the command:
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.13.0-linux-x86_64.tar.gz
``` 
































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



    
