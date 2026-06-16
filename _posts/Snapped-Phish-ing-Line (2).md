---
layout: post
title: "BARQ Systems Network and Cybersecurity Training"
description: "Part 1 of the BARQ Systems training series. This lab focuses on enterprise routing and network security using Junos OSPF and firewall filters. Part 2 will continue the series with FortiGate implementations."
date: 2026-05-24 00:00:00 +0200
categories: ["Home Lab", "Network Security"]
tags: [junos, pnet-lab, ospf, acl, firewall, routing, blue-team]
pin: false
image:
  path: https://github.com/user-attachments/assets/a05f0b00-19e8-4dd9-aa7b-b838c002a21b
  preview: false
  class: img-contain
---

# BARQ Systems Network and Cybersecurity Training Labs
This training lab is divided into two parts: one for network routing and the other for network security.
The Network part will include configuring Junos with the OSPF protocol, followed by building ACL rules.

> The second part of this series will focus on FortiGate firewalls. This post covers the first lab.
{: .prompt-info }

---

## Lab 0 – Junos Images and PNET Lab Installation
First, download VMware Workstation. Next, download PNET Lab, where the network scheme is drawn and implemented. 

Open the `pnet lab.ovf` file from VMware Workstation. The settings of the image should be configured as follows:
* **RAM:** 4 GB
* **CPUs:** 4
* **Network Adapter 1:** Bridged
* **Network Adapter 2:** Host-only
* **Disk:** 50 GB

Powering on the VM, the PNET Lab console will ask for credentials, which default to:
* **Username:** `root`
* **Password:** `pnet`

The VM will acquire an IP on the network. Browse to this IP to open the PNET Lab web interface using `http://[PNETLAB_IP]`.

After setting up the VM, the Junos VM needs to be imported. First, download the `vMX.ova` file. Then, use 7-Zip to extract its internal files. 

The file `vMX-disk1.vmdk` needs to be imported to the PNET Lab VM. This can be done using WinSCP. When opening WinSCP, enter the PNET Lab IP, username, and password to access it.

![WinSCP Login](https://github.com/user-attachments/assets/a13c9f40-45cb-497e-a512-f91895d65f1b){: .shadow .rounded }

Change the path of the right panel by navigating to the `/opt/unetlab/addons/qemu/` directory. Create a new folder with the name `vmx-x.x` (depending on the version). Move the `vMX-disk1.vmdk` file into this new folder.

Open a new command-line session using SSH to convert the disk into the suitable `qcow2` format required by PNET Lab, and fix the directory permissions:

```bash
ssh root@[PNETLAB_IP]

# Convert the VMDK to QCOW2 (PNET Lab format)
qemu-img convert -f vmdk junos-vmx.vmdk -O qcow2 hda.qcow2

# Fix the permissions
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
{: .nolineno }

Now, in PNET Lab, when a new file is created, the vMX option will be visible when adding nodes.

![Adding vMX Node](https://github.com/user-attachments/assets/1af94c2e-22c1-4f73-874d-090131787fab){: .shadow .rounded }

Clicking on it, configure the following settings:
* **Template:** Juniper vMX
* **Image:** vmx-x.x
* **RAM:** 2048
* **CPUs:** 2
* **Console:** telnet
* **Ethernet:** 4

![Node Settings](https://github.com/user-attachments/assets/964a77dd-472e-432e-80c1-d23a47530aed){: .shadow .rounded }

Use the same settings to add three vMX nodes. VPCs and a cloud node for management are added and connected as shown in the following topology:

![Topology](https://github.com/user-attachments/assets/93bf26b3-33e6-49b4-bff8-662e6e0ac03d){: .shadow .rounded }

---

## Configuring the Routers

### Router 1 (R1)
Double-click Router 1 to enter its CLI. The default username is `root` with a blank password (just press Enter). 

```text
cli
# Entering the configuration mode
config

# Before committing, a root password must be set
set system root-authentication plain-text-password

# SSH is enabled
set system services ssh

# Changing the router name to R1
set system host-name R1

# Creating a login user
set system login user admin class super-user authentication plain-text-password

# The interface IP for the management is set
set interfaces lo0 unit 0 family inet address 1.1.1.1/32

# The interface IP for the link to R2 is set
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/30

# The interface IP for the link to VPC is set
set interfaces ge-0/0/2 unit 0 family inet address 192.168.10.1/24

# Commit and verify
commit and-quit
show interfaces terse
```
{: .nolineno }

The interfaces are then shown:

![R1 Interfaces](https://github.com/user-attachments/assets/e1da15ec-0a66-45d9-b316-9e639fa2a9f6){: .shadow .rounded }

### Router 2 (R2)
For Router 2, the same configurations are made according to the topology:

```text
cli
config
set system root-authentication plain-text-password
set system services ssh
set system host-name R2
set system login user admin class super-user authentication plain-text-password

# Interface IPs
set interfaces lo0 unit 0 family inet address 2.2.2.2/32
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.0.23.1/30

commit and-quit
show interfaces terse
```
{: .nolineno }

![R2 Interfaces](https://github.com/user-attachments/assets/a3e8121e-99dc-4dc5-893e-59af74290e56){: .shadow .rounded }

### Router 3 (R3)

```text
cli
config
set system root-authentication plain-text-password
set system services ssh
set system host-name R3
set system login user admin class super-user authentication plain-text-password

# Interface IPs
set interfaces lo0 unit 0 family inet address 3.3.3.3/32
set interfaces ge-0/0/0 unit 0 family inet address 10.0.23.2/30

commit and-quit
show interfaces terse
```
{: .nolineno }

![R3 Interfaces](https://github.com/user-attachments/assets/7ad87701-c94f-4661-bcd5-3f599f090ce9){: .shadow .rounded }

### Testing the Initial Configuration
The direct connections are tested through the ping service to make sure the interfaces are correctly assigned with the suitable IPs. After testing, all the direct connections can successfully reach each other.

---

## OSPF Implementation
For the network to see other indirect connections, the OSPF (Open Shortest Path First) protocol is implemented.

**For R1 (Area 0 and 2):**
```text
configure
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0
set protocols ospf area 0.0.0.0 interface lo0.0 passive
commit
```
{: .nolineno }

**For R2 (Area 0 and 1):**
```text
configure
# Area 0 — link to R1
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
# Area 1 — link to R3
set protocols ospf area 0.0.0.1 interface ge-0/0/1.0
set protocols ospf area 0.0.0.1 interface lo0.0 passive
commit
```
{: .nolineno }

**For R3 (Area 1):**
```text
configure
set protocols ospf area 0.0.0.1 interface ge-0/0/0.0
set protocols ospf area 0.0.0.1 interface lo0.0 passive
commit
```
{: .nolineno }

### Verification Commands
Now all devices can see each other over the networks. You can verify this by running the following commands:

```text
# Check OSPF neighbors (should show FULL state)
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
```
{: .nolineno }

---

## Lab 2 - Setting Firewall ACLs

### Scenario A: Block a Compromised Host Machine
Assume the SOC receives an alert that host `192.168.10.2` is beaconing to a C2 server. It should be blocked from the router automatically without touching the host.

On R1:
```text
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
```
{: .nolineno }

These commands block all the traffic coming from the Linux VM along with logging and counting the dropped packets for further investigation analysis. The `ALLOW-REST` rule permits the traffic for the rest of the devices in the network, because the block rule has an implicit deny-all.

After testing, the Linux machine couldn’t ping external hosts:

![Ping Fail](https://github.com/user-attachments/assets/823fd8d4-216f-4304-9845-cc39e05bd961){: .shadow .rounded }

While in R1, the packets are successfully logged and counted:

![R1 Block Counters](https://github.com/user-attachments/assets/04cccf51-ffa9-4b0c-baf3-7d51dec33ad7){: .shadow .rounded }

After resolving the incident, the connection is restored by unassigning the rule from the interface:

```text
configure
delete interfaces ge-0/0/2 unit 0 family inet filter input
commit
```
{: .nolineno }

![Connection Restored](https://github.com/user-attachments/assets/d522b1e3-ade4-4be6-93ed-dab53d4e08d4){: .shadow .rounded }

### Scenario B: Allow Only HTTP/HTTPS and Block Other Ports
In corporate policy, LAN users may only use browsers to browse the internet. SSH or custom ports are not allowed, though DNS queries and ICMP are permitted. In this case, all other traffic is logged and dropped.

To implement this on R1:
```text
configure

# Allow ICMP (ping) — useful for diagnostics
set firewall family inet filter WEB-ONLY term ALLOW-ICMP from protocol icmp
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then count ICMP-COUNT
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then accept

# Allow DNS (port 53) — required for browsers to resolve domain names
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
```
{: .nolineno }

Testing the firewall from the Linux VM, it is observed that the PC can use DNS and HTTP services, but not Telnet or SSH:

![Web Only Test](https://github.com/user-attachments/assets/21b9bcad-3894-4092-b0c8-71747ec40037){: .shadow .rounded }

The discarded packets and DNS queries are recorded in R1 and can be shown using:
```text
show firewall filter WEB-ONLY
```
{: .nolineno }

![Firewall Filter Output](https://github.com/user-attachments/assets/04a5ca2e-27b9-4a9c-bcb8-81f2cbdfd64b){: .shadow .rounded }

The detailed logs show TCP and UDP protocols only when represented through the command:
```text
show firewall log
```
{: .nolineno }

![Firewall Log](https://github.com/user-attachments/assets/d7998cf5-1b0c-4761-8d21-0a408a5f2343){: .shadow .rounded }

### Scenario C: Block a Specific Destination IP
In the real world, whenever a known malicious IP is found, it must be blocked to ensure network safety. Taking the Google DNS IP as an example target, it can be blocked with the following commands in R1:

```text
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
```
{: .nolineno }

Testing the blockage on the Linux PC using `ping 8.8.8.8`, it is observed that there are no packets received:

![Blocked Ping](https://github.com/user-attachments/assets/8cbdd5ed-a818-401a-bf37-d3b1b35315b9){: .shadow .rounded }

While it successfully pings other external IPs:

![Successful Ping](https://github.com/user-attachments/assets/39c25265-7a65-40be-ac54-1ebae0ccec70){: .shadow .rounded }

Viewing the logs and counts globally in R1 using `show firewall` provides the count of all firewall rules currently applied:

![Show Firewall Overview](https://github.com/user-attachments/assets/a91d3ad6-8ff3-447c-9476-283152ba9b4f){: .shadow .rounded }

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
