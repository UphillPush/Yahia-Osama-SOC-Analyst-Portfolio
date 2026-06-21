---
layout: post
title: "Comprehensive Network Security: Junos OSPF, ACLs, & FortiGate IPsec VPNs"
description: "A complete end-to-end training walkthrough bridging enterprise routing (Junos OSPF/ACLs) with perimeter security (FortiGate Site-to-Site & Remote Access VPNs)."
date: 2026-05-24 00:00:00 +0200
categories: ["Home Lab", "Network Security"]
tags: [junos, fortigate, pnet-lab, eve-ng, ospf, acl, ipsec, vpn, blue-team]
pin: false
image:
  path: 
  preview: false
  class: img-contain
---

# Enterprise Routing and Encrypted Transport Architectures

> **Executive Summary**
> Effective SOC triage requires a deep understanding of both how traffic routes through an internal network and how it crosses encrypted perimeters. This massive training lab is divided into two primary disciplines:
> 
> **Part 1:** Establishing the routing backbone using Junos vMX routers on PNET Lab. This covers OSPF dynamic routing and stateless Access Control List (ACL) firewall filters for incident containment.
> **Part 2:** Securing the perimeter using FortiGate appliances on EVE-NG. This includes building a Site-to-Site IPsec VPN and a Remote Access (Dialup) VPN from scratch, with full protocol-level explanations.
{: .prompt-info }

---

## PART 1: Enterprise Routing & Incident Containment (Junos)

### Phase 1: Virtualization and PNET Lab Initialization
To replicate the internal routing network, a virtualized topology was constructed utilizing PNET Lab deployed on VMware Workstation. 

The hypervisor image (`pnet lab.ovf`) was imported with the following specifications to support the routing engines:
* **RAM:** 4 GB
* **CPUs:** 4
* **Network Adapter 1:** Bridged
* **Network Adapter 2:** Host-only
* **Storage:** 50 GB Disk

Upon booting, the PNET Lab console prompts for the default credentials (`root` / `pnet`). The VM acquires a local IP, which is then browsed (`http://[PNETLAB_IP]`) to access the management GUI.

Next, the Junos vMX image (`vMX.ova`) must be imported. 7-Zip was utilized to extract the internal `vMX-disk1.vmdk` file. Using WinSCP, this disk file was securely transferred into the PNET Lab file system at `/opt/unetlab/addons/qemu/`. 

![WinSCP Login](https://github.com/user-attachments/assets/a13c9f40-45cb-497e-a512-f91895d65f1b){: .shadow .rounded }

A new command-line session was opened via SSH to convert the disk into the `qcow2` format required by the hypervisor, followed by a permissions repair:

```bash
ssh root@[PNETLAB_IP]

# Convert the VMDK to QCOW2 (PNET Lab format)
qemu-img convert -f vmdk junos-vmx.vmdk -O qcow2 hda.qcow2

# Fix hypervisor permissions
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
{: .nolineno }

Inside PNET Lab, three vMX nodes were deployed. Each was allocated 2048 MB of RAM, 2 CPUs, and 4 Ethernet interfaces. Virtual PCs (VPCs) and a management cloud node were then interconnected to form the baseline topology:

![Adding vMX Node](https://github.com/user-attachments/assets/1af94c2e-22c1-4f73-874d-090131787fab){: .shadow .rounded }
![Node Settings](https://github.com/user-attachments/assets/964a77dd-472e-432e-80c1-d23a47530aed){: .shadow .rounded }
![Topology](https://github.com/user-attachments/assets/93bf26b3-33e6-49b4-bff8-662e6e0ac03d){: .shadow .rounded }

---

### Phase 2: Router Configuration & OSPF Implementation

Baseline configurations were applied to all three routing instances (R1, R2, and R3). This included setting root authentication, enabling SSH, and assigning interface IPv4 addresses.

**Router 1 (R1) Base Configuration:**
```text
cli
config
set system root-authentication plain-text-password
set system services ssh
set system host-name R1
set system login user admin class super-user authentication plain-text-password

set interfaces lo0 unit 0 family inet address 1.1.1.1/32
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/30
set interfaces ge-0/0/2 unit 0 family inet address 192.168.10.1/24
commit and-quit
```
{: .nolineno }

![R1 Interfaces](https://github.com/user-attachments/assets/e1da15ec-0a66-45d9-b316-9e639fa2a9f6){: .shadow .rounded }

Identical methodologies were applied to **R2** (management IP `2.2.2.2/32`) and **R3** (management IP `3.3.3.3/32`), assigning the respective transit subnet IPs to bridge the network.

![R2 Interfaces](https://github.com/user-attachments/assets/a3e8121e-99dc-4dc5-893e-59af74290e56){: .shadow .rounded }
![R3 Interfaces](https://github.com/user-attachments/assets/7ad87701-c94f-4661-bcd5-3f599f090ce9){: .shadow .rounded }

**OSPF Dynamic Routing:**
For the network to route traffic to indirectly connected subnets, the Open Shortest Path First (OSPF) protocol was implemented, defining Area 0 (Backbone) and Area 1.

```text
# On R1 (Area 0)
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0
set protocols ospf area 0.0.0.0 interface lo0.0 passive

# On R2 (Area Border Router - Area 0 & 1)
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.1 interface ge-0/0/1.0
set protocols ospf area 0.0.0.1 interface lo0.0 passive

# On R3 (Area 1)
set protocols ospf area 0.0.0.1 interface ge-0/0/0.0
set protocols ospf area 0.0.0.1 interface lo0.0 passive
```
{: .nolineno }

Routing adjacency was verified successfully using `show ospf neighbor` and end-to-end `traceroute` commands.

---

### Phase 3: Junos Firewall ACL Scenarios

Three distinct stateless firewall filters (`family inet filter`) were engineered on R1 to simulate common SOC incident response actions.

**Scenario A: Compromised Host Isolation**
Assume the SOC receives an alert that host `192.168.10.2` is beaconing to a C2 server. It must be blocked automatically without interacting directly with the infected endpoint.

```text
configure
# Block all traffic from the Linux VM IP, log it, and count it
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 from source-address 192.168.10.2/32
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then count HOST-BLOCKED
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then log
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then discard

# Default: allow all other LAN traffic (Implicit deny override)
set firewall family inet filter BLOCK-HOST term ALLOW-REST then accept

# Apply inbound on the LAN interface
set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-HOST
commit
```
{: .nolineno }

Testing from the Linux VM resulted in 100% packet loss. R1 successfully aggregated the blocked packets in the firewall counters and generated explicit drop logs.

![Ping Fail](https://github.com/user-attachments/assets/823fd8d4-216f-4304-9845-cc39e05bd961){: .shadow .rounded }
![R1 Block Counters](https://github.com/user-attachments/assets/04cccf51-ffa9-4b0c-baf3-7d51dec33ad7){: .shadow .rounded }
![Connection Restored](https://github.com/user-attachments/assets/d522b1e3-ade4-4be6-93ed-dab53d4e08d4){: .shadow .rounded }

**Scenario B: Strict Web-Only Enforcement**
Corporate policy dictates that LAN users may only use browsers (HTTP/HTTPS) and DNS. Unauthorized protocols like SSH and Telnet are strictly dropped.

```text
configure
# Allow ICMP, DNS (UDP 53), HTTP (TCP 80), and HTTPS (TCP 443)
set firewall family inet filter WEB-ONLY term ALLOW-ICMP from protocol icmp
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then accept
set firewall family inet filter WEB-ONLY term ALLOW-DNS from protocol udp
set firewall family inet filter WEB-ONLY term ALLOW-DNS from destination-port 53
set firewall family inet filter WEB-ONLY term ALLOW-DNS then accept
set firewall family inet filter WEB-ONLY term ALLOW-WEB from protocol tcp
set firewall family inet filter WEB-ONLY term ALLOW-WEB from destination-port [80 443]
set firewall family inet filter WEB-ONLY term ALLOW-WEB then accept

# Block, log, and count everything else
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then count DENIED-COUNT
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then log
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then discard
set interfaces ge-0/0/2 unit 0 family inet filter input WEB-ONLY
commit
```
{: .nolineno }

Validation confirmed `nslookup` and `wget` functioned normally, while `telnet` and `ssh` connection attempts timed out completely.

![Web Only Test](https://github.com/user-attachments/assets/21b9bcad-3894-4092-b0c8-71747ec40037){: .shadow .rounded }
![Firewall Filter Output](https://github.com/user-attachments/assets/04a5ca2e-27b9-4a9c-bcb8-81f2cbdfd64b){: .shadow .rounded }
![Firewall Log](https://github.com/user-attachments/assets/d7998cf5-1b0c-4761-8d21-0a408a5f2343){: .shadow .rounded }

**Scenario C: Malicious Indicator Blocking**
When threat intelligence flags an active malicious IP, it is neutralized at the perimeter. Using `8.8.8.8` as the demonstration target:

```text
configure
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 from destination-address 8.8.8.8/32
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then count DEST-BLOCKED
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then log
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then discard
set firewall family inet filter BLOCK-DEST-IP term ALLOW-REST then accept
set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-DEST-IP
commit
```
{: .nolineno }

Pings directed to `8.8.8.8` failed instantly, while benign external IPs (`8.8.4.4`) remained reachable.

![Blocked Ping](https://github.com/user-attachments/assets/8cbdd5ed-a818-401a-bf37-d3b1b35315b9){: .shadow .rounded }
![Successful Ping](https://github.com/user-attachments/assets/39c25265-7a65-40be-ac54-1ebae0ccec70){: .shadow .rounded }
![Show Firewall Overview](https://github.com/user-attachments/assets/a91d3ad6-8ff3-447c-9476-283152ba9b4f){: .shadow .rounded }

---

## PART 2: Perimeter Security & IPsec VPNs (FortiGate)

With the routing backbone understood, the training progressed to cryptographic perimeter defenses. This phase utilized **EVE-NG** and **FortiOS 7.0.2** to build encrypted transport mechanisms.

* **FortiGate Image:** [Download FortiGate 7.0.2](https://o6uedu-my.sharepoint.com/:u:/g/personal/202018756_o6u_edu_eg/IQC__SSmdVDUTYBEUVxpeetWAR54lUoPi3ST3ZmOzvmnpfI?e=XyFTDW)

Similar to the Junos deployment, the FortiGate `qcow2` images were transferred via WinSCP into the EVE-NG `/opt/unetlab/addons/qemu/` directory, and permissions were reset. 

![WinSCP File Transfer](https://github.com/user-attachments/assets/130811a6-6c03-48e9-8ca5-b372f6881fd4){: .shadow .rounded }

A fresh topology was mapped out, representing two distinct branch offices separated by an untrusted network.

![Adding Node](https://github.com/user-attachments/assets/1f9d101b-2cae-40f5-a48e-69918a7b02c8){: .shadow .rounded }
![Node Resources](https://github.com/user-attachments/assets/a737e765-e80f-4193-b9b0-c9ab4bfa16f2){: .shadow .rounded }
![Lab Topology](https://github.com/user-attachments/assets/62d216b0-c4e1-4ff1-8657-0a46c328ddaf){: .shadow .rounded }

### Baseline FortiGate Configuration

For `Forti-Left`, the initial IP assignment was handled via CLI, establishing the WAN (Port 2) and LAN (Port 3) addresses:

```text
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
{: .nolineno }

The FortiGate GUI was then accessed to configure the necessary static routes and baseline outbound/inbound firewall policies to permit standard cross-boundary traffic before encryption. 

![FortiOS GUI](https://github.com/user-attachments/assets/535c8f81-0156-42eb-ba19-1da5587cc23f){: .shadow .rounded }
![Forti-Left Interfaces](https://github.com/user-attachments/assets/58b8f2a0-b60d-43ab-82e4-af3f45e53e7d){: .shadow .rounded }
![Left Static Route](https://github.com/user-attachments/assets/4beae9e7-e869-4d22-8403-0b55503d62a6){: .shadow .rounded }
![Left Policy Out](https://github.com/user-attachments/assets/e956f443-b46d-4699-8b39-1dc703c7f3ae){: .shadow .rounded }

The configuration was symmetrically mirrored on `Forti-Right`, enabling successful ICMP reachability between the VPC endpoints.

![Forti-Right Interfaces](https://github.com/user-attachments/assets/062d3c8a-4b20-42e2-9067-a5c7e0e047d1){: .shadow .rounded }
![Right Policy Out](https://github.com/user-attachments/assets/4ff22b5a-0e13-4d6a-9e98-f54a6f456f30){: .shadow .rounded }
![Ping Left to Right](https://github.com/user-attachments/assets/d4ce746a-525d-4efb-8809-e768577dee81){: .shadow .rounded }

---

### Scenario 1: Site-to-Site IPsec VPN

With cleartext connectivity established, the traffic flow was upgraded to an encrypted IPsec tunnel utilizing the FortiOS IPsec Wizard.

![IPsec Wizard](https://github.com/user-attachments/assets/ac6da439-8b57-4d74-a8d7-7624429a753e){: .shadow .rounded }

> **Cryptographic Architecture:**
> The tunnel was built using standard IKE Main Mode. 
> * **Phase 1 (Negotiation):** Validates identity via the Pre-Shared Key (`FortiGate@123!`) and exchanges SA proposals (DES Encryption, SHA256 Authentication, DH Group 14).
> * **Phase 2 (ESP):** Defines the exact local (`10.10.10.0/24`) and remote (`20.20.20.0/24`) subnets authorized to traverse the payload.
{: .prompt-info }

![Phase 1 & 2 Config](https://github.com/user-attachments/assets/bd13bb72-34b4-422c-aa3c-909ffb7d8ced){: .shadow .rounded }

The previous cleartext firewall policies were purged, and new dedicated VPN policies were instantiated. Traffic originating from the LAN must be explicitly permitted to exit via the logical `ToRemote` VPN tunnel interface, and vice versa.

![VPN Policy 1](https://github.com/user-attachments/assets/44e470d4-f7b5-48f8-9535-2208892c9239){: .shadow .rounded }
![VPN Policy 2](https://github.com/user-attachments/assets/027d99e8-d1b6-4881-aafe-a561416edd7b){: .shadow .rounded }

Finally, static routes were modified to point destination traffic into the tunnel.

![Left VPN Route](https://github.com/user-attachments/assets/dccc54a0-5aa1-4181-a277-16797caeea78){: .shadow .rounded }
![Right VPN Route](https://github.com/user-attachments/assets/77db7726-ee6a-4f4a-948d-eaa5c0917332){: .shadow .rounded }

Upon initiating traffic, the IPsec tunnel successfully negotiates, turning GREEN in the management dashboard. 

![Tunnel Up](https://github.com/user-attachments/assets/70e1ac40-69ee-4871-8e3d-e71589f33208){: .shadow .rounded }

---

### Scenario 2: Remote Access (Dialup) VPN

A Site-to-Site VPN treats both sides of the connection as trusted branch offices. However, a remote worker operates from an unmanaged, untrusted endpoint (e.g., public WiFi). To accommodate this, a Dialup Remote Access VPN architecture must be deployed.

![Remote Access Topology](https://github.com/user-attachments/assets/496844be-d3ef-4d29-bbfc-9b361e479b46){: .shadow .rounded }

`Forti-Left` was reconfigured using the **Remote Access** wizard. A dedicated user group was created for authenticated workers, and a specific DHCP scope was allocated to dynamically assign internal IPs to authorized remote endpoints.

![Remote Access Wizard](https://github.com/user-attachments/assets/8be1021c-3e1a-4f7e-a72f-104737a04960){: .shadow .rounded }
![Dialup Auth](https://github.com/user-attachments/assets/0bd2dd05-ff94-4078-aa14-b73e4338e9c4){: .shadow .rounded }
![Dialup Subnet](https://github.com/user-attachments/assets/8d3d4230-b830-45f0-8c80-c7ead68ae1ac){: .shadow .rounded }
![Dialup Complete](https://github.com/user-attachments/assets/2acd922e-3f31-42be-8140-368606a5aa7d){: .shadow .rounded }

To test the architecture, an external Windows VM was provisioned with the [FortiClient 6.2.6](https://www.mediafire.com/file/s6zcwn9jww11lro/FortiClientVPNSetup_6.2.6.0951_x64.exe/file) application. The endpoint was configured with the Remote Gateway IP and Pre-Shared Key.

![FortiClient Setup 1](https://github.com/user-attachments/assets/cc3e862f-1079-4c85-ab0d-0232e0e05077){: .shadow .rounded }
![FortiClient Setup 2](https://github.com/user-attachments/assets/7c52b35d-cbf8-42fb-93a6-18545d01d348){: .shadow .rounded }
![FortiClient Setup 3](https://github.com/user-attachments/assets/127afb18-18a9-4fc1-8a96-6b9b567cd3c1){: .shadow .rounded }

Upon submitting the authorized user credentials, the tunnel authenticated successfully. The remote machine was dynamically assigned an internal IP address and granted secure, encrypted access to the localized corporate assets.

![Client Login](https://github.com/user-attachments/assets/b9635958-a0d9-4d1e-85bd-e46392ccdf5b){: .shadow .rounded }
![Tunnel Established](https://github.com/user-attachments/assets/e0e9f88e-759b-4477-bcb5-8dc981beba59){: .shadow .rounded }
![Remote Ping Success](https://github.com/user-attachments/assets/d3023919-41a7-434d-9947-0305c5819a3c){: .shadow .rounded }

<style>
  /* 1. Fix the "Outside" (Home Page Cards) - prevents cropping */
  .post-preview .preview-img img,
  .post-preview .preview-img {
    object-fit: contain !important;
    background-color: #1b1b1e !important;
  }

  /* 2. Fix the "Inside" - Completely hides the header image and its spacing */
  body[data-layout="post"] .post-meta + .mt-3.mb-3,
  body[data-layout="post"] .preview-img {
    display: none !important;
    visibility: hidden !important;
    height: 0 !important;
    margin: 0 !important;
  }
</style>
