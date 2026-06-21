---
layout: post
title: "FortiGate IPsec VPNs: Site-to-Site and Remote Access Architectures"
description: "Part 2 of the network security training series. Deploying a Site-to-Site IPsec VPN and a Remote Access Dialup VPN using FortiGate firewalls on EVE-NG."
date: 2026-06-21 00:00:00 +0200
categories: ["Home Lab", "Network Security"]
tags: [fortigate, eve-ng, ipsec, vpn, firewall, blue-team]
pin: false
image:
  path: https://github.com/user-attachments/assets/130811a6-6c03-48e9-8ca5-b372f6881fd4
  preview: false
  class: img-contain
---

# FortiGate IPsec VPN Architectures

> **Executive Summary**
> Following foundational enterprise routing, perimeter security and encrypted transport mechanisms must be established. This training phase focuses on two hands-on labs built entirely from scratch: a Site-to-Site IPsec VPN, followed by a Remote Access Dialup VPN. Both architectures are executed on EVE-NG, with full protocol-level explanations of every setting. 
>
> _Note: High Availability (HA) clustering with seamless VPN failover will be covered in the next continuation of this series._
{: .prompt-info }

The implementation is divided into three progressive phases:
* **Lab 0:** Virtualization, Software Installation, and Baseline Routing.
* **Lab 1:** Site-to-Site IPsec VPN Configuration.
* **Lab 2:** Remote Access (Dialup) VPN Configuration.

---

## Lab 0: Virtualization and Baseline Configuration

### Phase 1: Software Installation
For this topology, EVE-NG is utilized as the primary hypervisor alongside FortiGate OS version 7.0.2. 
* **FortiGate Image:** [Download FortiGate 7.0.2 via OneDrive](https://o6uedu-my.sharepoint.com/:u:/g/personal/202018756_o6u_edu_eg/IQC__SSmdVDUTYBEUVxpeetWAR54lUoPi3ST3ZmOzvmnpfI?e=XyFTDW)

After importing the EVE-NG virtual machine into VMware, the FortiGate appliance images must be securely transferred into the EVE-NG file system. This is achieved using WinSCP to drag and drop the files.

First, SSH into the EVE-NG VM to create the necessary directory structure:
```bash
ssh root@<eve-ng-ip>   # Default password: eve 

mkdir -p /opt/unetlab/addons/qemu/fortinet-7.0.2
cd /opt/unetlab/addons/qemu/fortinet-7.0.2
```
{: .nolineno }

Using WinSCP, the FortiGate images are transferred to the newly created `/opt/unetlab/addons/qemu/fortinet-7.0.2` path.

![WinSCP File Transfer](https://github.com/user-attachments/assets/130811a6-6c03-48e9-8ca5-b372f6881fd4){: .shadow .rounded }
_Figure 1: Transferring the FortiGate qcow2 disk image via WinSCP._

Finally, directory permissions must be repaired across the hypervisor:
```bash
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
{: .nolineno }

### Phase 2: Topology Initialization
After ensuring the EVE-NG network adapter is set to bridged mode, the VM is powered on and acquires a local IP. Browsing to this IP opens the EVE-NG web interface (Default credentials: `admin` / `eve`).

A FortiGate router node is added to the canvas. For this specific lab and FortiOS version, only **1 CPU** and **1024 MB (1 GB) of RAM** are required per instance.

![Adding Node](https://github.com/user-attachments/assets/1f9d101b-2cae-40f5-a48e-69918a7b02c8){: .shadow .rounded }

![Node Resources](https://github.com/user-attachments/assets/a737e765-e80f-4193-b9b0-c9ab4bfa16f2){: .shadow .rounded }

Additional Virtual PCs (VPCs) and cloud management nodes are added and interconnected according to the following baseline topology:

![Lab Topology](https://github.com/user-attachments/assets/62d216b0-c4e1-4ff1-8657-0a46c328ddaf){: .shadow .rounded }
_Figure 2: The base network topology featuring two branch offices separated by a routed boundary._

> **Tooling Note:** To properly interact with the nodes via CLI, ensure the EVE-NG Windows Client Pack and [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) are installed on the host machine.
{: .prompt-tip }

### Phase 3: Baseline Routing & Interface Configuration
All nodes are selected and started. 

**Configuring Forti-Left:**
Upon initial console access, the default username is `admin` with a blank password (press Enter). The system immediately prompts for a new administrative password. Once authenticated, the hostname is updated for clear identification:

```text
config system global
set hostname "Forti-Left"
end
```
{: .nolineno }

![Hostname Change](https://github.com/user-attachments/assets/faf488f6-d828-40c3-9716-011caddd65d9){: .shadow .rounded }

To access the graphical interface, the DHCP address assigned to the management cloud port (Port 1) must be identified:
```text
get system interface physical
```
{: .nolineno }

![Identify IP](https://github.com/user-attachments/assets/e21d1518-5421-4f32-99f5-16d86952c4f7){: .shadow .rounded }

Browsing to this IP yields the FortiOS login portal. 

![FortiOS GUI](https://github.com/user-attachments/assets/535c8f81-0156-42eb-ba19-1da5587cc23f){: .shadow .rounded }

While interface IPs can be assigned via the GUI, CLI configuration is faster and more precise. The WAN (Port 2) and LAN (Port 3) interfaces are configured:

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

![Forti-Left Interfaces](https://github.com/user-attachments/assets/58b8f2a0-b60d-43ab-82e4-af3f45e53e7d){: .shadow .rounded }

**Configuring Forti-Right:**
The exact same methodology is applied to the peer router, swapping the IP addressing to reflect the right-hand branch network:

```text
config system global
  set hostname "Forti-Right"
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
{: .nolineno }

![Forti-Right Interfaces](https://github.com/user-attachments/assets/062d3c8a-4b20-42e2-9067-a5c7e0e047d1){: .shadow .rounded }

**Configuring the Virtual PCs:**
* **Left VPC:** `ip 10.10.10.2 24 10.10.10.1`
* **Right VPC:** `ip 20.20.20.2 24 20.20.20.1`

### Phase 4: Static Routing & Firewall Policies
At this stage, each VPC can reach its default gateway, but inter-branch communication fails. A static route and corresponding firewall policies must be established.

In the `Forti-Left` GUI, navigate to **Network** $\rightarrow$ **Static Routes**. A new route is created targeting the `20.20.20.0/24` subnet via the `172.16.10.2` gateway, exiting through Port 2.

![Left Static Route](https://github.com/user-attachments/assets/4beae9e7-e869-4d22-8403-0b55503d62a6){: .shadow .rounded }

Because FortiGate operates on a default-deny paradigm, routing alone is insufficient. Explicit firewall policies must permit the traffic. Under **Policy & Objects** $\rightarrow$ **Firewall Policy**, two bidirectional rules are created:
1. **Outbound:** Permit traffic incoming from Port 3 (LAN) and exiting Port 2 (WAN).
2. **Inbound:** Permit traffic incoming from Port 2 (WAN) and exiting Port 3 (LAN).

![Left Policy Out](https://github.com/user-attachments/assets/e956f443-b46d-4699-8b39-1dc703c7f3ae){: .shadow .rounded }
![Left Policy In](https://github.com/user-attachments/assets/85b1c112-b218-4802-99b9-e04b6a9fcda6){: .shadow .rounded }

This routing and policy logic is mirrored symmetrically on `Forti-Right`:

![Right Static Route](https://github.com/user-attachments/assets/a0b5ff28-f61c-4e2f-95ea-f80bd7b01d74){: .shadow .rounded }
![Right Policy Out](https://github.com/user-attachments/assets/4ff22b5a-0e13-4d6a-9e98-f54a6f456f30){: .shadow .rounded }
![Right Policy In](https://github.com/user-attachments/assets/d522fc81-9c06-41c7-b08a-3a34bb112168){: .shadow .rounded }

End-to-end cleartext reachability is verified by pinging across the boundary:

![Ping Left to Right](https://github.com/user-attachments/assets/d4ce746a-525d-4efb-8809-e768577dee81){: .shadow .rounded }
![Ping Right to Left](https://github.com/user-attachments/assets/8187f6ea-e952-4c8c-96a0-6a6d9cbe246d){: .shadow .rounded }

---

## Lab 1: Site-to-Site IPsec VPN

With base connectivity confirmed, the unsecured boundary traffic will now be encapsulated into an encrypted IPsec tunnel.

### IKE Phase 1: Negotiation & Authentication
Navigate to **VPN** $\rightarrow$ **IPsec Wizard** $\rightarrow$ **Custom** and assign a tunnel name (e.g., `ToRemote`).

![IPsec Wizard](https://github.com/user-attachments/assets/ac6da439-8b57-4d74-a8d7-7624429a753e){: .shadow .rounded }

The Phase 1 parameters are configured to match identically on both peers:

| Parameter | Forti-Left | Forti-Right |
| :--- | :--- | :--- |
| **IP Version** | static IP | static IP |
| **Remote Gateway** | 172.16.10.2 | 172.16.10.1 |
| **Interface** | port2 | port2 |
| **IKE Version** | 1 | 1 |
| **Mode** | Main | Main |
| **Auth Method** | Pre-shared Key | Pre-shared Key |
| **Pre-shared Key** | `FortiGate@123!` | `FortiGate@123!` |
| **Encryption** | DES | DES |
| **Authentication** | SHA256 | SHA256 |
| **DH Group** | 14 | 14 |
| **Dead Peer Detection** | On Demand | On Demand |
| **NAT Traversal** | Enable | Enable |

> **IKE Phase 1 Mechanics:**
> These configurations directly map to the standard 3-step IKE Main Mode process:
> 1. **Negotiation:** Exchanging Security Association (SA) proposals (Encryption, Auth, DH Group, Lifetime).
> 2. **Key Exchange:** Executing the Diffie-Hellman mathematical exchange (Group 14).
> 3. **Authentication:** Validating identity via the Pre-Shared Key (PSK).
{: .prompt-info }

### IKE Phase 2: Encapsulating Security Payload (ESP)
Phase 2 defines the exact local and remote subnets that are permitted to utilize the tunnel.

| Parameter | Forti-Left | Forti-Right |
| :--- | :--- | :--- |
| **Local Subnet** | 10.10.10.0/24 | 20.20.20.0/24 |
| **Remote Subnet** | 20.20.20.0/24 | 10.10.10.0/24 |
| **Protocol** | ESP | ESP |
| **Encryption** | DES | DES |
| **Authentication** | SHA256 | SHA256 |
| **Enable PFS** | Checked | Checked |
| **Lifetime** | 43200 | 43200 |
| **Auto-negotiate** | Enabled | Enabled |

![Phase 1 & 2 Config](https://github.com/user-attachments/assets/bd13bb72-34b4-422c-aa3c-909ffb7d8ced){: .shadow .rounded }

### VPN Routing & Policies
The previous cleartext firewall policies must be disabled or deleted. Two new policies are created on each FortiGate to route traffic directly into the logical VPN tunnel interface.

**Forti-Left Policy 1 (LAN $\rightarrow$ VPN)**
* **Incoming:** port3
* **Outgoing:** ToRemote (VPN Interface)
* **Source:** 10.10.10.0/24
* **Destination:** 20.20.20.0/24
* **Service:** ALL | **Action:** ACCEPT | **NAT:** OFF

**Forti-Left Policy 2 (VPN $\rightarrow$ LAN)**
* **Incoming:** ToRemote
* **Outgoing:** port3
* **Source:** 20.20.20.0/24
* **Destination:** 10.10.10.0/24
* **Service:** ALL | **Action:** ACCEPT | **NAT:** OFF

_(Mirrored policies are repeated on Forti-Right, swapping the source and destination subnets.)_

![VPN Policy 1](https://github.com/user-attachments/assets/44e470d4-f7b5-48f8-9535-2208892c9239){: .shadow .rounded }
![VPN Policy 2](https://github.com/user-attachments/assets/027d99e8-d1b6-4881-aafe-a561416edd7b){: .shadow .rounded }

Finally, static routes are updated to point destination traffic into the tunnel rather than out the physical WAN port.
* **Forti-Left:** Route `20.20.20.0/24` via Device: `ToRemote`
* **Forti-Right:** Route `10.10.10.0/24` via Device: `ToRemote`

![Left VPN Route](https://github.com/user-attachments/assets/dccc54a0-5aa1-4181-a277-16797caeea78){: .shadow .rounded }
![Right VPN Route](https://github.com/user-attachments/assets/77db7726-ee6a-4f4a-948d-eaa5c0917332){: .shadow .rounded }

### Verification & Diagnostics
In the GUI, navigating to **VPN** $\rightarrow$ **IPsec Tunnels** should display the tunnel status as GREEN (Up).

![Tunnel Up](https://github.com/user-attachments/assets/70e1ac40-69ee-4871-8e3d-e71589f33208){: .shadow .rounded }

For deep-dive SOC analysis, the CLI offers real-time debugging of the cryptographic exchange:

```bash
# Check IKE Phase 1 status
get vpn ike gateway ToRemote

# Check IPsec Phase 2 status
get vpn ipsec tunnel name ToRemote

# Live debug (Execute BEFORE initiating a ping to watch the real-time IKE exchange)
diagnose debug application ike -1
diagnose debug enable

# -> Trigger the tunnel with an ICMP ping from VPC 1 to VPC 2 <-

# Disable debug output
diagnose debug disable
diagnose debug reset
```
{: .nolineno }

---

## Lab 2: Remote Access (Dialup) VPN

> **Architectural Paradigm Shift**
> In the Site-to-Site lab above, the branches are positioned behind their own FortiGates on networks deliberately built to reach each other. The moment the tunnel is established, a permanent, symmetric path is created—exactly like a leased line. 
> 
> However, a remote worker is not modeled this way. A remote worker's laptop is an unmanaged device on an untrusted network (home WiFi, coffee shop). It should be granted zero access to the corporate LAN until an explicit Dialup VPN connection is dynamically authenticated.
{: .prompt-warning }

![Remote Access Topology](https://github.com/user-attachments/assets/496844be-d3ef-4d29-bbfc-9b361e479b46){: .shadow .rounded }
_Figure 3: Remote Access architecture treating the external client as an untrusted endpoint._

To securely onboard the remote user to the corporate network, `Forti-Left` is configured as a Dialup server.

Navigate to **VPN** $\rightarrow$ **IPsec Wizard** $\rightarrow$ **Remote Access**.
![Remote Access Wizard](https://github.com/user-attachments/assets/8be1021c-3e1a-4f7e-a72f-104737a04960){: .shadow .rounded }

The incoming interface (Port 1, facing the internet) is selected. A pre-shared key is defined, and a user group is assigned containing the designated remote user credentials.
![Dialup Auth](https://github.com/user-attachments/assets/0bd2dd05-ff94-4078-aa14-b73e4338e9c4){: .shadow .rounded }

The target internal interface (Port 3/LAN) is selected, and a dedicated DHCP address range is specified. This range will dynamically assign a localized IP to the remote worker's virtual adapter upon successful authentication.
![Dialup Subnet](https://github.com/user-attachments/assets/8d3d4230-b830-45f0-8c80-c7ead68ae1ac){: .shadow .rounded }

Upon creation, the wizard automatically generates the required firewall policies and dynamic routing rules.
![Dialup Complete](https://github.com/user-attachments/assets/2acd922e-3f31-42be-8140-368606a5aa7d){: .shadow .rounded }

### Client Execution
To simulate the endpoint, an older version of FortiClient is utilized on a Windows VM. 
* **Download:** [FortiClient 6.2.6](https://www.mediafire.com/file/s6zcwn9jww11lro/FortiClientVPNSetup_6.2.6.0951_x64.exe/file)

A new IPsec VPN connection is created within the client, mirroring the Dialup server parameters (Remote Gateway IP and Pre-Shared Key).

![FortiClient Setup 1](https://github.com/user-attachments/assets/cc3e862f-1079-4c85-ab0d-0232e0e05077){: .shadow .rounded }
![FortiClient Setup 2](https://github.com/user-attachments/assets/7c52b35d-cbf8-42fb-93a6-18545d01d348){: .shadow .rounded }
![FortiClient Setup 3](https://github.com/user-attachments/assets/127afb18-18a9-4fc1-8a96-6b9b567cd3c1){: .shadow .rounded }

Using the credentials mapped to the firewall's user group, the connection is initiated.
![Client Login](https://github.com/user-attachments/assets/b9635958-a0d9-4d1e-85bd-e46392ccdf5b){: .shadow .rounded }

The tunnel successfully builds. The remote endpoint acquires an internal IP from the designated DHCP pool and can now successfully ping internal corporate assets over the encrypted tunnel.

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
