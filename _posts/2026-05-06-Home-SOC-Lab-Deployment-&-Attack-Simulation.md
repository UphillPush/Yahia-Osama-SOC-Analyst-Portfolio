---
layout: post
title: "Building a Home SOC Lab: ELK Stack & SSH Brute Force Detection"
date: 2026-05-06 00:00:00 +0200
categories: ["Home Lab", "SIEM Setup"]
tags: [elastic, kibana, soc-lab, threat-hunting, blue-team, hydra]
pin: true
image:
  path: https://cdn.worldvectorlogo.com/logos/elastic.svg
  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Home SOC Lab Deployment & Attack Simulation

![Lab Architecture Header](<img width="983" height="112" alt="image" src="https://github.com/user-attachments/assets/362289e8-365f-44f2-a046-92577d63ddee" />
){: .shadow .rounded }
_Figure 1: SOC Home Lab Overview._

## Introduction & Architecture Overv
iew

Before diving into the technical deployment, it is important to understand exactly what is being built in this lab and how the components interact. 

The goal of this project is to build a functional Security Operations Center (SOC) environment from scratch. We will deploy a target machine, instrument it to collect security telemetry, forward those logs to a central SIEM, and finally simulate a realistic cyber attack (SSH brute-forcing) to see how an analyst hunts for the activity.

To achieve this, we are using the industry-standard **ELK Stack**, specifically broken down into these three core components:

*   **Elasticsearch (The Brain):** A highly scalable search and analytics engine. It acts as the central database where all our security logs are securely stored, indexed, and made instantly searchable.
*   **Kibana (The Eyes):** The visualization and management layer. Kibana sits on top of Elasticsearch and provides the interactive web interface (GUI) where SOC analysts actually spend their time hunting for threats, building dashboards, and running KQL (Kibana Query Language) searches.
*   **Elastic Agent (The Eyes & Ears):** A single, unified endpoint agent installed directly on our target machine. Instead of installing multiple different log shippers, the Elastic Agent monitors the system, collects specific files (like `/var/log/auth.log`), and securely forwards them over the network into Elasticsearch.

**The Workflow:** The attacker strikes the Ubuntu server ➔ The Elastic Agent detects the failed logins and sends the logs to Elasticsearch ➔ Elasticsearch indexes the data ➔ The SOC Analyst uncovers the attack in Kibana.

---

## Phase 1: Setting up the Virtual Machines 

For this lab, two virtual machines were deployed: an Ubuntu Server acting as the endpoint generating logs, and a Windows machine hosting the ELK stack for advanced log analysis and threat detection.

### Windows Host Setup (SIEM Server)
First, a new virtual machine is created. Give the machine a name and select the `.iso` file of the OS to be installed.

![Windows VM Setup](https://github.com/user-attachments/assets/00006630-69bb-456b-88b4-886249488530){: .shadow .rounded }
_Figure 2: Selecting the Windows ISO._

The username and password for the machine are set as well.

![Windows Credentials](https://github.com/user-attachments/assets/06384b5d-f348-49a7-af23-9a9a2bd6bbf6){: .shadow .rounded }
_Figure 3: Setting the primary user account._

For the hardware, 4GB of RAM and 3 CPU cores are sufficient to run the SIEM stack smoothly in this lab environment.

![Hardware Setup](https://github.com/user-attachments/assets/62a89474-9702-4c80-9cd0-2a47f44631f7){: .shadow .rounded }
_Figure 4: Allocating Memory and Processors._

For the hard disk, 50GB or less is enough space.

![Disk Setup](https://github.com/user-attachments/assets/c60dcf3e-7e21-4f82-b259-e7f947a75cce){: .shadow .rounded }
_Figure 5: Allocating virtual disk size._

Then, click "Finish" to start the virtual machine installation.

![Finish VM Setup](https://github.com/user-attachments/assets/6228e593-3a96-4464-929d-54aac0623e21){: .shadow .rounded }
_Figure 6: Completing the initial VM configuration._

> **Networking Tip:** It is important to make sure that the Network settings are set to **Bridged Adapter**. The Name should be set to your physical adapter (e.g., Realtek PCIe if the host machine is connected through a LAN, or Wireless if connected through Wi-Fi).
{: .prompt-info }

![Bridged Adapter](https://github.com/user-attachments/assets/eb7e2125-483f-45ad-a5f8-c94da15ef197){: .shadow .rounded }
_Figure 7: Configuring the Bridged Network Adapter._

---

## Phase 2: Installing & Configuring Elasticsearch

After the Windows machine boots up, browse to the official Elastic website to download the platform:
`https://www.elastic.co/downloads/past-releases/elasticsearch-8-13-0`

![Elasticsearch Download](https://github.com/user-attachments/assets/37611df6-231e-4d4c-b4fb-362823e7e8e3){: .shadow .rounded }
_Figure 8: Downloading Elasticsearch 8.13.0._

After the download finishes, extract the `.zip` folder into the `C:\` drive. Before running the service, the configuration file located at `config\elasticsearch.yml` needs to be edited. Add these lines to the file:
```yaml
network.host: 0.0.0.0
discovery.type: single-node
```
{: .nolineno }

![Elastic YAML Config](https://github.com/user-attachments/assets/b475570d-4bb5-4a2f-a32e-8d264207547a){: .shadow .rounded }
_Figure 9: Updating the elasticsearch.yml file._

*   `network.host: 0.0.0.0` allows Elasticsearch to accept connections from other machines on the network, not just localhost.
*   Note: For a simple home lab environment, setting `xpack.security.enabled: false` can disable authentication to simplify setup, though here we will proceed with basic security enabled.

After saving and closing the file, open a Command Prompt and execute the `elasticsearch.bat` file:
```cmd
cd C:\elasticsearch-8.13.0\bin
elasticsearch.bat
```
{: .nolineno }

![Starting Elasticsearch](https://github.com/user-attachments/assets/422f17d4-4cff-49a9-984e-866895244cc4){: .shadow .rounded }
_Figure 10: Running the Elasticsearch batch file._

Browsing to `127.0.0.1:9200` will direct you to Elasticsearch and ask for a login. When Elastic starts for the first time, it generates a default password. To reset it manually, open a new Command Prompt and type:
```cmd
cd C:\elasticsearch-8.13.0\bin
elasticsearch-reset-password -u elastic
```
{: .nolineno }

![Resetting Elastic Password](https://github.com/user-attachments/assets/72d57742-a798-4dfa-85e9-a1e7eff61d83){: .shadow .rounded }
_Figure 11: Generating a new password for the 'elastic' user._

This prints a new password that can be copied and used to log in.

![Elastic Login Screen](https://github.com/user-attachments/assets/98ed9c77-eb19-407e-9c88-e24da17e1d59){: .shadow .rounded }
_Figure 12: Logging into Elasticsearch._

![Elastic Success JSON](https://github.com/user-attachments/assets/bdfa1082-8ab4-4c02-b425-a38e63bda6d4){: .shadow .rounded }
_Figure 13: Successful connection to the Elastic cluster._

---

## Phase 3: Installing & Configuring Kibana

After configuring Elasticsearch, the Kibana dashboard needs to be set up to visualize the logs and provide an interactive GUI for analysis.

Download Kibana from the link below and extract it to the `C:\` drive as well:
`https://www.elastic.co/downloads/past-releases/kibana-8-13-0`

![Kibana Download](https://github.com/user-attachments/assets/838f731b-130a-4435-898b-c0f54e5cf47e){: .shadow .rounded }
_Figure 14: Downloading Kibana 8.13.0._

Kibana requires a system password to connect to Elasticsearch. You can generate this using the elasticsearch terminal:
```cmd
cd C:\elasticsearch-8.13.0\bin
elasticsearch-reset-password -u kibana_system
```
{: .nolineno }

![Resetting Kibana Password](https://github.com/user-attachments/assets/c8af2bcc-f94c-4d7f-bc6d-503748bf4c97){: .shadow .rounded }
_Figure 15: Generating the kibana_system user password._

Copy this password to use in the configuration file. Open the Kibana configuration file (`kibana.yml` located in the `config` folder) and add the following lines:
```yaml
elasticsearch.hosts: ["[http://127.0.0.1:9200](http://127.0.0.1:9200)"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "<password_copied_from_cmd>"
elasticsearch.ssl.verificationMode: none
```
{: .nolineno }

![Kibana YAML Config](https://github.com/user-attachments/assets/c5bbe22e-950d-4e57-9939-576fbc2e4ad6){: .shadow .rounded }
_Figure 16: Updating the kibana.yml file._

Save the file and start Kibana through the Command Prompt:
```cmd
cd C:\kibana-8.13.0\bin
kibana.bat
```
{: .nolineno }

![Starting Kibana](https://github.com/user-attachments/assets/4a25817e-03bc-4f00-9a71-139b3e45d7e5){: .shadow .rounded }
_Figure 17: Running the Kibana batch file._

![Kibana Availability](https://github.com/user-attachments/assets/2082d3c6-ec8c-41cf-93fb-ee4129310ca9){: .shadow .rounded }
_Figure 18: Kibana successfully starting._

When the `"Kibana is now available"` message is seen in the CMD, browsing to `127.0.0.1:5601` will grant access to the Kibana dashboard. Sign in using the primary `elastic` credentials created earlier.

![Kibana Login](https://github.com/user-attachments/assets/791ecb20-82b4-4a83-860b-3f4992b4b923){: .shadow .rounded }
_Figure 19: Logging into the Kibana web interface._

![Kibana Dashboard](https://github.com/user-attachments/assets/8e2c9576-b09d-4edc-b732-9f89ae5ad31b){: .shadow .rounded }
_Figure 20: The primary Kibana Welcome Screen._

---

## Phase 4: Setting Up the Agent (Ubuntu Server)

After successfully setting up Elasticsearch and Kibana, it's finally time to set up the endpoint that will be monitored. For this lab, an Ubuntu Server virtual machine was chosen. 

A new VM was created with 2GB RAM and 2 cores, utilizing the Ubuntu Server `.iso` image.

![Ubuntu Setup 1](https://github.com/user-attachments/assets/942fb6aa-2c38-4617-91b6-b53404081a43){: .shadow .rounded }
![Ubuntu Setup 2](https://github.com/user-attachments/assets/495c0c5d-01cf-4654-bd39-525810596feb){: .shadow .rounded }
_Figure 21: Configuring the Ubuntu Server virtual machine._

The two virtual machines must be on the same network, so the Bridged Adapter configuration is applied here as well.

![Ubuntu Network Setup](https://github.com/user-attachments/assets/a771c4ba-da03-42b6-b7d5-440c092c83eb){: .shadow .rounded }
_Figure 22: Ensuring Bridged Networking is active._

The SSH protocol can be used to control the Ubuntu Server directly from the Windows host machine's command prompt. First, determine the IP of the server using:
```bash
ip a
```
{: .nolineno }

![Checking IP Address](https://github.com/user-attachments/assets/fa1c6a7b-2008-41a8-bbf9-586c151ec0c6){: .shadow .rounded }
_Figure 23: Retrieving the Ubuntu Server IP address._

From the command output, it is known that the IP of the server is `192.168.1.3`. Use this IP to connect via SSH from the Windows CMD:
```cmd
ssh admn@192.168.1.3
```
{: .nolineno }

After typing the password, the server terminal is accessed. Now, the Elastic Agent can be downloaded onto the Ubuntu Server using the following command:
```bash
curl -L -O [https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.13.0-linux-x86_64.tar.gz](https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.13.0-linux-x86_64.tar.gz)
```
{: .nolineno }

Extract the downloaded archive:

```bash
tar xzvf elastic-agent-8.13.0-linux-x86_64.tar.gz
```
{: .nolineno }

Change into the extracted directory:

```bash
cd elastic-agent-8.13.0-linux-x86_64
```
{: .nolineno }

Some configurations need to be optimized in the `elastic-agent.yml` file to ensure logs are routed to the SIEM:
```bash
cat > elastic-agent.yml << 'EOF'
outputs:
  default:
    type: elasticsearch
    hosts: ["http://<HOST_IP_ADDRESS>:9200"]
    username: "elastic"
    password: "YOUR_ELASTIC_PASSWORD"

inputs:
  - type: logfile
    id: system-logs
    streams:
      - paths:
          - /var/log/syslog
          - /var/log/auth.log
EOF
```
{: .nolineno }

![Agent Config File](https://github.com/user-attachments/assets/e761e1b2-dae3-4687-b126-9f1ea18ae718){: .shadow .rounded }
_Figure 24: Overwriting the elastic-agent.yml configuration._

Now, install the Elastic Agent through the terminal:
```bash
sudo ./elastic-agent install
```
{: .nolineno }

![Installing Agent](https://github.com/user-attachments/assets/1b8cce87-eb9c-4d3b-86c6-8ec6d3d0818b){: .shadow .rounded }
_Figure 25: Executing the agent installation script._

Check its status to ensure it is actively running and forwarding logs:
```bash
sudo systemctl status elastic-agent
```
{: .nolineno }

![Agent Status](https://github.com/user-attachments/assets/086c2d8c-dc3e-4989-80c7-a636d195e060){: .shadow .rounded }
_Figure 26: Confirming the elastic-agent service is active._

The Elastic Agent is now set up. Returning to Kibana, the server logs immediately begin appearing, confirming that Elasticsearch is successfully collecting telemetry from the endpoint!

![Logs in Kibana](https://github.com/user-attachments/assets/258c97a9-0d89-4cd1-9bba-864437a6968f){: .shadow .rounded }
_Figure 27: Validating log ingestion in Kibana's Discover tab._

---

## Phase 5: Attack Simulation & Triage

To test the lab's detection capabilities, an SSH brute force attack will be simulated using Hydra. Hydra can be easily installed on the server through the following command:
```bash
sudo apt install hydra -y
```
{: .nolineno }

A fake password list is created to act as a reference dictionary for Hydra:
```bash
cat > passwords.txt << 'EOF'
wrong1
wrong2
wrong3
wrong4
wrongpass
123456
admn
EOF
```
{: .nolineno }

![Creating Wordlist](https://github.com/user-attachments/assets/575183bb-67d1-4102-bae9-c77fd1d18f96){: .shadow .rounded }
_Figure 28: Generating the brute-force dictionary._

Then, the brute force attack is run locally against the server's SSH service:
```bash
hydra -l admn -P passwords.txt ssh://192.168.1.3
```
{: .nolineno }

> **Simulation Note:** This will hammer the SSH daemon with multiple wrong passwords rapidly, which is much more realistic than attempting manual typing for testing threshold alerts.
{: .prompt-warning }

![Hydra Execution](https://github.com/user-attachments/assets/3450a42b-fc4a-428e-8f8a-7e20fc8be5d8){: .shadow .rounded }
_Figure 29: Hydra successfully executing the brute-force attack._

Now, to test if this attack was monitored and logged correctly, Kibana is utilized. 
Navigate to **Menu ☰ -> Discover**, and ensure the `logs-*` index is selected. By searching for failed passwords, a massive spike of attempts in a highly condensed timeframe is revealed.
```kql
message: "Failed password"
```
{: .nolineno }

![Detecting the Attack](https://github.com/user-attachments/assets/4b9acdfd-3c24-4ba4-b892-6a99c29328b9){: .shadow .rounded }
_Figure 30: Hunting the brute force attack via KQL inside Kibana._

---

## Phase 6: Next Steps & Alerting Recommendations

We have successfully built a centralized logging pipeline and manually hunted down an attack using KQL. However, relying on an analyst to manually search for `"Failed password"` all day is not a sustainable defense strategy. 

In a real-world enterprise environment, a modern Security Operations Center relies on **automated detection engineering**. The next logical phase for this lab is to move from *manual hunting* to *proactive alerting*.

To properly secure this environment, we should configure Elastic Security to actively monitor for this specific behavior. For example, we could create a threshold rule that triggers a high-severity alert if **more than 5 failed SSH login attempts occur from a single source IP within a 1-minute window**. 

Building these custom detection rules, writing the logic to minimize false positives, and configuring the SIEM to automatically push notifications to an analyst (via email or a webhook) will be the primary focus of my **future projects and upcoming write-ups!**

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
````</HOST_IP_ADDRESS>
