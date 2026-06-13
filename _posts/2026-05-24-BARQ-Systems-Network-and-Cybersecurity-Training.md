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

# Enterprise Routing & Access Control: Junos OSPF and Firewall Filters

> **Executive Summary**
> **Situation:** Foundational knowledge of network routing and perimeter security is critical for effective SOC triage. A simulated enterprise environment was required to test network segmentation and incident response containment strategies.
> 
> **Task:** A comprehensive training lab was deployed utilizing Junos vMX routers to build a multi-area OSPF network. Following the establishment of dynamic routing, several Access Control List (ACL) scenarios were engineered to simulate isolating compromised hosts, enforcing strict corporate web policies, and blocking outbound traffic to malicious indicators of compromise (IoCs).
> 
> **Action:** A PNET Lab environment was provisioned to host three Junos vMX nodes and internal VPCs. OSPF was configured across Area 0.0.0.0 and Area 0.0.0.1 to establish end-to-end connectivity. Stateless firewall filters (`family inet filter`) were developed and applied inbound on the LAN interfaces to selectively drop, count, and log unauthorized traffic.
> 
> **Result:** Dynamic routing was successfully established and verified via routing tables and traceroutes. Incident containment was proven effective; compromised host traffic was dropped entirely, web-only enforcement successfully blocked anomalous protocols, and specific external malicious destinations were successfully rendered unreachable.
{: .prompt-info }

---

## Phase 1: Virtualization and Environment Initialization

To accurately replicate an enterprise network, a virtualized topology was constructed utilizing PNET Lab deployed on VMware Workstation. The host hypervisor was configured with the following specifications to support the routing engines:
* **Memory:** 4 GB RAM
* **Processors:** 4 CPUs
* **Network Adapter 1:** Bridged
* **Network Adapter 2:** Host Only
* **Storage:** 50GB Disk

Upon powering on the VM, the PNET Lab console was accessed utilizing the default credentials (`root` / `pnet`). The web interface was then accessed via the assigned IP address. 

The Juniper vMX image (`vMX-disk1.vmdk`) was imported into the environment via WinSCP. To ensure compatibility with the PNET Lab hypervisor, the virtual disk was moved to `/opt/unetlab/addons/qemu/vmx-14.1/` and converted to `qcow2` format using an SSH console. 

```bash
# Converting the VMDK to QCOW2 format
qemu-img convert -f vmdk junos-vmx.vmdk -O qcow2 hda.qcow2

# Fixing directory permissions for the hypervisor
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
{: .nolineno }

Three vMX nodes were deployed into the topology, with each instance allocated specific parameters: **Template:** Juniper VMX, **RAM:** 2048 MB, **CPUs:** 2, **Console:** telnet, and **Ethernet:** 4. VPCs and a management cloud network were added and interconnected.

---

## Phase 2: Network Infrastructure and OSPF Routing

Before security controls could be implemented, baseline connectivity had to be established. Base configurations were applied to all three routing instances (R1, R2, and R3), which included setting root authentication, enabling SSH services, and assigning interface IPv4 addresses.

### Base Interface Configuration (Example: R1)
```text
cli
config
set system root-authentication plain-text-password
set system services ssh
set system host-name R1
set system login user admin class super-user authentication plain-text-password

# Assigning Interface IPs
set interfaces lo0 unit 0 family inet address 1.1.1.1/32
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/30
set interfaces ge-0/0/2 unit 0 family inet address 192.168.10.1/24
commit and-quit
```
{: .nolineno }

The interface configurations were verified using `show interfaces terse`, confirming the physical and logical links were operational:

| Interface | Admin | Link | Proto | Local | Remote |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ge-0/0/0.0** | up | up | inet | 10.0.12.1/30 | |
| **ge-0/0/2.0** | up | up | inet | 192.168.10.1/24 | |

Similar configurations were deployed to **R2** (Interfaces `ge-0/0/0` at 10.0.12.2/30 and `ge-0/0/1` at 10.0.23.1/30) and **R3** (Interface `ge-0/0/0` at 10.0.23.2/30). 

### OSPF Implementation
To facilitate dynamic routing across the topology, the Open Shortest Path First (OSPF) protocol was implemented. 
* **R1** was configured strictly within the backbone area (`0.0.0.0`).
* **R2** acted as an Area Border Router (ABR), bridging Area `0.0.0.0` with Area `0.0.0.1`.
* **R3** was configured entirely within Area `0.0.0.1`.

```text
# OSPF Configuration deployed on R2
configure
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.1 interface ge-0/0/1.0
set protocols ospf area 0.0.0.1 interface lo0.0 passive
commit
```
{: .nolineno }

End-to-end connectivity was verified using the following diagnostic commands:
* `show ospf neighbor` (Confirmed FULL state adjacencies)
* `show route protocol ospf`
* `ping 3.3.3.3 source 1.1.1.1 count 5` (Successful ICMP delivery across areas)
* `traceroute 3.3.3.3 source 1.1.1.1` (Confirmed path routing through R2)

---

## Phase 3: Access Control and Firewall Filters

With the network fully operational, three distinct stateless firewall filters were engineered on R1 to simulate common SOC response actions.

### Scenario A: Compromised Host Isolation
A scenario was simulated where an internal host (`192.168.10.2`) was detected beaconing to a Command and Control (C2) server. To enact immediate containment without touching the endpoint, a filter was created to block all originating traffic.

```text
configure
# Dropping, counting, and logging traffic from the compromised host
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 from source-address 192.168.10.2/32
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then count HOST-BLOCKED
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then log
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then discard

# Allowing all other network traffic
set firewall family inet filter BLOCK-HOST term ALLOW-REST then accept

# Applying the filter inbound on the LAN interface
set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-HOST
commit
```
{: .nolineno }

> **Implicit Deny:** Junos firewall filters end with an implicit "deny all" statement. The `ALLOW-REST` term is strictly required to prevent the filter from accidentally blackholing the entire subnet.
{: .prompt-warning }

Verification testing from the isolated Linux VM confirmed 100% packet loss when attempting to ping external gateways (`10.0.12.2`). In R1, the discarded packets were successfully aggregated:

```text
root@R1> show firewall filter BLOCK-HOST
Filter: BLOCK-HOST
Counters:
Name                                 Bytes              Packets
HOST-BLOCKED                          9492                  113
```
{: .nolineno }

The detailed logs (`show firewall log`) explicitly captured the blocked ICMP packets originating from the isolated machine:

| Time | Action | Interface | Protocol | Src Addr | Dest Addr |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 08:53:40 | D | ge-0/0/2.0 | ICMP | 192.168.10.2 | 10.0.12.2 |
| 08:53:39 | D | ge-0/0/2.0 | ICMP | 192.168.10.2 | 10.0.12.2 |

Once the incident was conceptually resolved, the filter was successfully removed (`delete interfaces ge-0/0/2 unit 0 family inet filter input`) and round-trip ICMP communications were restored.

---

### Scenario B: Strict Port-Based Web Filtering
To enforce corporate policy, an ACL was engineered to permit only web browsing, actively dropping unnecessary protocols like SSH or Telnet. The `WEB-ONLY` filter was designed to specifically allow:
* ICMP (for diagnostics).
* UDP Port 53 (DNS resolution).
* TCP Ports 80 and 443 (HTTP/HTTPS).

```text
configure
# Allow ICMP
set firewall family inet filter WEB-ONLY term ALLOW-ICMP from protocol icmp
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then count ICMP-COUNT
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then accept

# Allow DNS 
set firewall family inet filter WEB-ONLY term ALLOW-DNS from protocol udp
set firewall family inet filter WEB-ONLY term ALLOW-DNS from destination-port 53
set firewall family inet filter WEB-ONLY term ALLOW-DNS then count DNS-COUNT
set firewall family inet filter WEB-ONLY term ALLOW-DNS then accept

# Allow HTTP/HTTPS
set firewall family inet filter WEB-ONLY term ALLOW-WEB from protocol tcp
set firewall family inet filter WEB-ONLY term ALLOW-WEB from destination-port [80 443]
set firewall family inet filter WEB-ONLY term ALLOW-WEB then count WEB-COUNT
set firewall family inet filter WEB-ONLY term ALLOW-WEB then accept

# Logging and discarding all other traffic
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then count DENIED-COUNT
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then log
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then discard

# Apply on LAN interface
set interfaces ge-0/0/2 unit 0 family inet filter input WEB-ONLY
commit
```
{: .nolineno }

Testing from the client terminal confirmed the policy execution:
* `nslookup google.com` successfully resolved the IP addresses.
* `wget http://example.com` successfully initiated an outbound HTTP request.
* `ssh root@10.0.12.2` and `telnet 10.0.12.2` were aggressively dropped, timing out the connections.

The firewall counters (`show firewall filter WEB-ONLY`) accurately tracked the traffic segregation:

| Counter Name | Bytes | Packets |
| :--- | :--- | :--- |
| DENIED-COUNT | 59052 | 207 |
| DNS-COUNT | 945 | 14 |
| ICMP-COUNT | 252 | 3 |
| WEB-COUNT | 0 | 0 |

---

### Scenario C: Malicious Destination Blocking
In the event a malicious external IP is identified by threat intelligence, it must be blocked at the perimeter. Using `8.8.8.8` as the simulated malicious target, a filter was implemented to discard outbound requests destined for that specific address.

```text
configure
# Block all traffic destined to 8.8.8.8
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

Subsequent connectivity tests demonstrated 100% packet loss to the blocked IP (`8.8.8.8`), while legitimate traffic to secondary addresses (`8.8.4.4`) remained fully operational with 0% packet loss. 

The master firewall log view (`show firewall`) verified the final block counts, confirming `15936` bytes and `193` packets were successfully intercepted by the `DEST-BLOCKED` counter.

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
