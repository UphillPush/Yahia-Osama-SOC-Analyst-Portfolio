---
layout: post
title: "Enterprise Network Security Architecture: BARQ Systems Hands-On Lab"
description: "Deep-dive technical guide covering multi-area OSPF routing, stateless firewall ACLs for incident response, and Site-to-Site + Remote Access IPsec VPN implementations. Built for SOC analyst visibility into traffic flows, encryption, and perimeter defense mechanisms."
date: 2026-06-21 00:00:00 +0200
categories: ["Network Security", "Lab Walkthrough", "SOC Training"]
tags: [junos, fortigate, eve-ng, pnet-lab, ospf, ipsec, acl, vpn, incident-response, blue-team, network-forensics]
pin: true
---

# Enterprise Network Security Architecture: BARQ Systems Lab Training

## Overview & Learning Objectives

As a SOC analyst, understanding traffic flow is **non-negotiable**. You cannot effectively investigate network-based threats if you don't know:
- How packets route between subnets
- Where firewall controls can be applied for containment
- What encryption mechanisms protect data in transit
- How to trace a compromise across multiple network segments

This lab—built on BARQ systems infrastructure using **Junos vMX**, **FortiGate 7.0.2**, **EVE-NG**, and **PNET Lab**—walks through a complete enterprise network topology from Layer 3 routing design through perimeter VPN encryption.

**By the end, you will understand:**
1. ✓ How to design and troubleshoot multi-area OSPF networks
2. ✓ How to deploy stateless firewall filters for immediate threat containment
3. ✓ How to build, verify, and debug encrypted IPsec tunnels
4. ✓ How to reason about network indicators during incident triage

---

## PART 1: Enterprise Backbone Design & Dynamic Routing

### The Problem We're Solving

In a real SOC context, when you see a suspicious outbound connection to `8.8.8.8` from host `192.168.10.5`, you need to answer three questions *immediately*:

1. **Which border router handles this traffic?** (Routing knowledge)
2. **What firewall rules are in place?** (Access control knowledge)
3. **Where can I block this at scale?** (Incident response capability)

Static routes don't scale. We use **OSPF** to dynamically discover routes and **Junos firewall filters** to enforce policy at the network edge.

---

### Phase 1: Lab Infrastructure Setup

#### 1.1 PNET Lab Hypervisor Configuration

PNET Lab is a lightweight, web-based network simulation platform deployed on **VMware Workstation**. Think of it as a self-contained lab orchestration engine.

**VM Specification:**
- **RAM:** 4 GB (routing engines are memory-hungry)
- **CPUs:** 4 cores
- **Storage:** 50 GB disk (qcow2 images are large)
- **Network Adapters:** 
  - Adapter 1 (Bridged) → allows external access to the GUI
  - Adapter 2 (Host-only) → isolates the lab from your network

**Initial Boot & Access:**

```bash
# Console login (after OVF import)
Username: root
Password: pnet

# Acquire IP address (DHCP via Adapter 1)
ifconfig eth0

# Access the web interface
http://<PNETLAB_IP>
```

---

#### 1.2 Importing the Junos vMX Image

The vMX image comes as an OVA file (virtual appliance). We need to extract the VMDK disk and convert it to `qcow2` format for PNET Lab compatibility.

**Step 1: Extract the vMX disk**

```bash
# Using 7-Zip on Windows
7z x vMX.ova

# Result: vMX-disk1.vmdk is extracted
```

**Step 2: Transfer to PNET Lab via SCP**

```bash
# From your Windows machine, use WinSCP
# Source: C:\path\to\vMX-disk1.vmdk
# Destination: /opt/unetlab/addons/qemu/
# Username: root
# Password: pnet
```

**Step 3: Convert VMDK → QCOW2 & Fix Permissions**

Once the file is uploaded, SSH into PNET Lab and convert the disk format:

```bash
ssh root@<PNETLAB_IP>

# Navigate to the qemu directory
cd /opt/unetlab/addons/qemu/

# Convert disk format (this takes ~5-10 minutes)
qemu-img convert -f vmdk junos-vmx.vmdk -O qcow2 hda.qcow2

# Fix hypervisor file permissions (CRITICAL)
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

**Why this matters:** If permissions are wrong, the qemu hypervisor can't read the disk image and the node won't boot.

---

#### 1.3 Building the Topology in PNET Lab

Now we deploy the actual routing nodes. The topology represents a simplified enterprise network:
- **R1 (Area 0):** Backbone router with internal LAN
- **R2 (ABR):** Area Border Router connecting backbone to branch
- **R3 (Area 1):** Branch office router
- **VPC endpoints:** Simulated end-hosts for testing

**Node Specifications for Each vMX:**
- **RAM:** 2048 MB
- **CPUs:** 2
- **Interfaces:** 4x Ethernet

**Topology Diagram (Conceptual):**
```
┌─────────────────────────────────────────┐
│  BACKBONE NETWORK (Area 0)              │
│  ┌──────┐        ┌──────┐              │
│  │  R1  │────────│  R2  │ (ABR)        │
│  │  LAN │        └──────┘              │
│  └──────┘            │                 │
│                      │ Area 0 ↔ Area 1 │
│                   Transit Link          │
│                      │                 │
└──────────────────────┼────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
    ┌───────┐                    ┌──────┐
    │  R3   │                    │ ABR  │
    │ (Area 1)                   │      │
    └───────┘                    └──────┘
        │
    VPC-Left (10.20.20.0/24)
```

**Interface Assignment Plan:**

| Router | Interface | Subnet | Purpose |
|--------|-----------|--------|---------|
| R1 | lo0 | 1.1.1.1/32 | Loopback (management) |
| R1 | ge-0/0/0 | 10.0.12.1/30 | Transit to R2 |
| R1 | ge-0/0/2 | 192.168.10.1/24 | Local LAN |
| R2 | lo0 | 2.2.2.2/32 | Loopback (management) |
| R2 | ge-0/0/0 | 10.0.12.2/30 | Transit to R1 |
| R2 | ge-0/0/1 | 10.0.23.1/30 | Transit to R3 |
| R3 | lo0 | 3.3.3.3/32 | Loopback (management) |
| R3 | ge-0/0/0 | 10.0.23.2/30 | Transit to R2 |
| R3 | ge-0/0/2 | 20.20.20.1/24 | Local LAN |

---

### Phase 2: Router Base Configuration

Each Junos router requires authentication, SSH access, and interface addressing before OSPF can be enabled.

#### 2.1 Router 1 (R1) - Backbone Router

```text
# Enter configuration mode
cli
configure

# Authentication & Access
set system root-authentication plain-text-password
[Enter password when prompted: "FortiGate@123!"]

set system login user admin class super-user authentication plain-text-password
[Enter password: "FortiGate@123!"]

# Enable remote management
set system services ssh
set system services netconf ssh

# Hostname (useful for debugging)
set system host-name R1

# Loopback interface (management & OSPF router ID)
set interfaces lo0 unit 0 family inet address 1.1.1.1/32

# Transit interface to R2
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/30
set interfaces ge-0/0/0 unit 0 family inet mtu 1500

# LAN interface (end-host facing)
set interfaces ge-0/0/2 unit 0 family inet address 192.168.10.1/24
set interfaces ge-0/0/2 unit 0 family inet mtu 1500

# Commit changes to running config
commit and-quit
```

**Key concepts:**
- **Loopback interface:** Used as the OSPF Router ID (elected automatically from highest loopback IP)
- **MTU setting:** Junos defaults to 1500 bytes; keep consistent across all interfaces to avoid fragmentation issues
- **Port numbering:** `ge-0/0/0` = Gigabit Ethernet, slot 0, PIC 0, port 0

---

#### 2.2 Router 2 (R2) - Area Border Router

R2 bridges the backbone (Area 0) and the branch (Area 1), so it has interfaces in both areas.

```text
cli
configure

set system root-authentication plain-text-password
[FortiGate@123!]

set system login user admin class super-user authentication plain-text-password
[FortiGate@123!]

set system services ssh
set system services netconf ssh
set system host-name R2

# Loopback
set interfaces lo0 unit 0 family inet address 2.2.2.2/32

# Area 0 side (backbone)
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.2/30
set interfaces ge-0/0/0 unit 0 family inet mtu 1500

# Area 1 side (branch)
set interfaces ge-0/0/1 unit 0 family inet address 10.0.23.1/30
set interfaces ge-0/0/1 unit 0 family inet mtu 1500

commit and-quit
```

---

#### 2.3 Router 3 (R3) - Branch Router

```text
cli
configure

set system root-authentication plain-text-password
[FortiGate@123!]

set system login user admin class super-user authentication plain-text-password
[FortiGate@123!]

set system services ssh
set system services netconf ssh
set system host-name R3

# Loopback
set interfaces lo0 unit 0 family inet address 3.3.3.3/32

# Transit to R2
set interfaces ge-0/0/0 unit 0 family inet address 10.0.23.2/30
set interfaces ge-0/0/0 unit 0 family inet mtu 1500

# LAN (branch office)
set interfaces ge-0/0/2 unit 0 family inet address 20.20.20.1/24
set interfaces ge-0/0/2 unit 0 family inet mtu 1500

commit and-quit
```

---

### Phase 3: OSPF Implementation & Route Propagation

#### 3.1 Understanding OSPF Architecture

**OSPF in 60 seconds:**
- Routers form **adjacencies** with neighbors on shared links
- Each router floods its **Link State Advertisements (LSAs)** through the network
- Every router builds an identical database and computes the shortest path tree using Dijkstra's algorithm
- Unlike static routes, OSPF automatically detects link failures and converges within seconds

**Multi-Area Design:**
- **Area 0 (Backbone):** The core network that all areas must connect through
- **Area 1 (Branch):** A remote network connected via an Area Border Router (ABR)
- **Benefits:** Reduces SPF recalculation overhead, supports hierarchical scaling

---

#### 3.2 OSPF Configuration

**On R1 (Backbone Only - Area 0):**

```text
cli
configure

set protocols ospf router-id 1.1.1.1
set protocols ospf area 0.0.0.0 network 10.0.12.0/30
set protocols ospf area 0.0.0.0 network 192.168.10.0/24

# Make loopback passive (don't send OSPF hellos on it)
set protocols ospf area 0.0.0.0 interface lo0.0 passive

# Set transit interface priority (for designated router election)
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 priority 100

commit and-quit
```

**Why passive on loopback?** Loopback interfaces have no physical neighbors. Sending OSPF packets on them is wasteful. We keep them in the routing process to advertise the IP.

---

**On R2 (Area Border Router - ABR):**

```text
cli
configure

set protocols ospf router-id 2.2.2.2

# Area 0 (backbone side)
set protocols ospf area 0.0.0.0 network 10.0.12.0/30

# Area 1 (branch side)
set protocols ospf area 0.0.0.1 network 10.0.23.0/30

# Passive loopback
set protocols ospf area 0.0.0.0 interface lo0.0 passive

# Make both transit interfaces active (will form adjacencies)
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 priority 100
set protocols ospf area 0.0.0.1 interface ge-0/0/1.0 priority 100

commit and-quit
```

---

**On R3 (Branch Only - Area 1):**

```text
cli
configure

set protocols ospf router-id 3.3.3.3
set protocols ospf area 0.0.0.1 network 10.0.23.0/30

# Passive loopback
set protocols ospf area 0.0.0.1 interface lo0.0 passive
set protocols ospf area 0.0.0.1 interface ge-0/0/0.0 priority 100

commit and-quit
```

---

#### 3.3 Verification & Adjacency Troubleshooting

**Check OSPF neighbors:**

```bash
show ospf neighbor

# Expected output:
# Address         Interface           State ID              Pri Dead
# 10.0.12.2       ge-0/0/0.0         Full 2.2.2.2          100 35
```

**If neighbors don't form, diagnose with:**

```bash
show ospf database summary
show ospf interface
show ospf statistics
```

**Common issues:**
- **Mismatched areas:** Interface in Area 0 talking to interface in Area 1 = no adjacency
- **OSPF disabled on interface:** Happens if subnet doesn't match any `network` command
- **Hello/Dead timer mismatch:** Default is 10/40 seconds; must match on both sides

---

#### 3.4 Route Propagation Verification

Once adjacencies are up, OSPF converges (usually <5 seconds). Verify with:

```bash
# View all learned routes
show route protocol ospf

# Expected on R1:
# 10.0.23.0/30 via 10.0.12.2 (learned from R2)
# 20.20.20.0/24 via 10.0.12.2 (learned from R3 via R2)

# Trace a packet's path
traceroute 20.20.20.1

# Output should show:
# R1 → R2 → R3 (hops 1, 2, 3)
```

---

### Phase 4: Stateless Firewall Filters for Incident Response

#### 4.1 Why Stateless Filters Matter in SOC Work

A firewall **filter** in Junos is a set of rules that match on packet headers and apply actions. Unlike stateful firewalls (which track connections), stateless filters:
- ✓ Process each packet independently
- ✓ Offer deterministic, low-latency decision-making
- ✓ Can be deployed on a border router in seconds for incident containment
- ✗ Don't track connection state (so you must explicitly allow return traffic)

**In a SOC incident context:**
- "We've confirmed host `192.168.10.2` is compromised. Block it at the router entrance NOW."
- Stateless filter can achieve this in <30 seconds
- Stateful firewall might require session teardown and are slower to deploy

---

#### 4.2 Scenario A: Compromised Host Isolation

**Scenario:** The SOC receives a malware alert for host `192.168.10.2`. Initial analysis shows C2 beaconing to external IPs. We must block this host's traffic immediately while forensics run.

**Implementation on R1:**

```text
cli
configure

# Define a filter named "BLOCK-HOST"
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 from source-address 192.168.10.2/32
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then count HOST-BLOCKED
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then log
set firewall family inet filter BLOCK-HOST term BLOCK-192.168.10.2 then discard

# Allow everything else (don't inadvertently block LAN)
set firewall family inet filter BLOCK-HOST term ALLOW-REST then accept

# Apply the filter to inbound traffic on the LAN interface
set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-HOST

commit and-quit
```

**What this does:**

| Packet Source | Firewall Logic | Action |
|--------------|---|--------|
| 192.168.10.2 → * | Matches `BLOCK-192.168.10.2` | Increments counter, logs drop, discards packet |
| 192.168.10.3 → * | No match, falls through to `ALLOW-REST` | Accepts packet |

**Verification:**

```bash
# From 192.168.10.2 (compromised host), try to reach external network
ping 8.8.8.8
# Result: 100% packet loss (blocked by R1)

# Check filter counters
show firewall filter BLOCK-HOST

# Output:
# Filter: BLOCK-HOST
# Name                                                Packet count    Byte count
# BLOCK-192.168.10.2                                          45          2700
# ALLOW-REST                                              2400         234000

# Check system log for drop entries
show log messages | grep "BLOCK-HOST"
```

---

#### 4.3 Scenario B: Strict Web-Only Policy Enforcement

**Scenario:** Corporate policy states LAN users may only:
- Browse HTTP/HTTPS (TCP 80, 443)
- Resolve DNS (UDP 53)
- Use ICMP for diagnostics

All other protocols (SSH, Telnet, FTP, etc.) must be blocked, logged, and counted.

**Implementation on R1:**

```text
cli
configure

set firewall family inet filter WEB-ONLY term ALLOW-ICMP from protocol icmp
set firewall family inet filter WEB-ONLY term ALLOW-ICMP then accept

set firewall family inet filter WEB-ONLY term ALLOW-DNS from protocol udp
set firewall family inet filter WEB-ONLY term ALLOW-DNS from destination-port 53
set firewall family inet filter WEB-ONLY term ALLOW-DNS then accept

set firewall family inet filter WEB-ONLY term ALLOW-WEB from protocol tcp
set firewall family inet filter WEB-ONLY term ALLOW-WEB from destination-port [80 443]
set firewall family inet filter WEB-ONLY term ALLOW-WEB then accept

# Deny everything else with logging
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then count DENIED-COUNT
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then log
set firewall family inet filter WEB-ONLY term DEFAULT-DENY then discard

set interfaces ge-0/0/2 unit 0 family inet filter input WEB-ONLY

commit and-quit
```

**Testing the policy:**

```bash
# From a LAN host (192.168.10.10):

# ✓ DNS should work
nslookup google.com
# Result: successful resolution

# ✓ HTTP should work
wget http://example.com
# Result: downloads successfully

# ✗ SSH should timeout
ssh user@example.com
# Result: timeout (connection refused after 30s)

# ✗ FTP should timeout
ftp example.com
# Result: timeout

# Check what was blocked
show firewall filter WEB-ONLY
# Output shows DEFAULT-DENY counter incremented
```

**Why this matters for SOC:** If you notice SSH traffic from your corporate LAN to an external IP, either the filter failed or someone reconfigured the router. This becomes an immediate investigation trigger.

---

#### 4.4 Scenario C: Threat Intelligence-Based Blocking

**Scenario:** Your threat intel feed flags `8.8.8.8` as hosting malware. You push a firewall rule to block all traffic destined for that IP.

**Implementation on R1:**

```text
cli
configure

set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 from destination-address 8.8.8.8/32
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then count DEST-BLOCKED
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then log
set firewall family inet filter BLOCK-DEST-IP term BLOCK-8.8.8.8 then discard

# Allow everything else
set firewall family inet filter BLOCK-DEST-IP term ALLOW-REST then accept

set interfaces ge-0/0/2 unit 0 family inet filter input BLOCK-DEST-IP

commit and-quit
```

**Verification:**

```bash
# Try to ping the malicious IP
ping 8.8.8.8
# Result: 100% packet loss

# Try a benign IP (should work)
ping 8.8.4.4
# Result: successful responses

# View counters
show firewall filter BLOCK-DEST-IP
```

---

#### 4.5 Real-World Application: Incident Containment Playbook

**When a compromise is confirmed:**

1. **Immediate:** Deploy host isolation filter (Scenario A)
   ```bash
   # Takes <30 seconds to apply
   # No traffic in or out from infected machine
   ```

2. **2-5 minutes:** Apply web-only filter to the subnet (Scenario B)
   ```bash
   # Prevents attacker from using non-standard C2 channels
   # Allows legitimate work to continue
   ```

3. **5-30 minutes:** Apply threat intel blocks for known malicious IPs (Scenario C)
   ```bash
   # Prevents lateral movement to attacker infrastructure
   ```

4. **30+ minutes:** Forensics team images the machine
   ```bash
   # Network containment complete
   # Incident response continues offline
   ```

---

## PART 2: Perimeter Security & Encrypted Tunnels

### The Problem We're Solving (Again)

Branch offices need to communicate securely. Remote workers need VPN access. But you can't send traffic in clear text over the internet. We use **IPsec** to encrypt traffic at the network layer.

As a SOC analyst, you need to understand:
- How IPsec tunnels establish (IKE Phase 1 & 2)
- What happens if a tunnel is down (failover, rerouting)
- How to correlate encrypted tunnel state with suspicious patterns
- How to decrypt traffic for forensics (key derivation, packet analysis)

---

### Phase 1: EVE-NG Hypervisor & FortiGate Deployment

#### 1.1 EVE-NG Environment Setup

EVE-NG is a more feature-rich simulator than PNET Lab but serves the same purpose: orchestrate virtual network appliances.

**System Requirements:**
- 8+ GB RAM
- 4+ CPU cores
- 100+ GB storage (VPN appliances are large)
- KVM or VMware as the underlying hypervisor

**FortiGate Image Preparation:**

```bash
# Download FortiGate 7.0.2 qcow2 image from Fortinet portal
# File: FortiGate-VM64-7.0.2-XXXX.qcow2

# Transfer to EVE-NG server via SCP
scp FortiGate-VM64-7.0.2.qcow2 root@<EVE_IP>:/opt/unetlab/addons/qemu/

# SSH into EVE-NG
ssh root@<EVE_IP>

# Fix permissions
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

---

#### 1.2 Lab Topology

We're simulating two branch offices separated by an untrusted WAN (the internet):

```
┌─────────────────────────────────────────────────────────────┐
│  BRANCH A (Left)                                            │
│  ┌──────────────────┐              ┌──────────────────┐     │
│  │  Forti-Left      │──────────────│  Forti-Right     │     │
│  │  WAN: 172.16.10.1│ IPsec Tunnel │WAN: 172.16.20.1 │     │
│  │  LAN: 10.10.10.0 │   (Encrypted)│LAN: 20.20.20.0  │     │
│  └────────┬─────────┘              └────────┬─────────┘     │
│           │                                 │                │
│       VPC-A (10.10.10.5)              VPC-B (20.20.20.5)    │
│                                                              │
│  BRANCH B (Right)                                           │
└─────────────────────────────────────────────────────────────┘
```

**Network Plan:**

| Node | Interface | IP | Purpose |
|------|-----------|-------|---------|
| Forti-Left | port2 (WAN) | 172.16.10.1/30 | Internet-facing |
| Forti-Left | port3 (LAN) | 10.10.10.1/24 | Branch A internal |
| Forti-Right | port2 (WAN) | 172.16.20.1/30 | Internet-facing |
| Forti-Right | port3 (LAN) | 20.20.20.1/24 | Branch B internal |
| Simulated Internet | port1 | 172.16.15.1/30 | Router between WAN IPs |

---

### Phase 2: Base FortiGate Configuration

#### 2.1 Initial Access & IP Assignment

Once FortiGate VMs boot, they present a CLI. Default credentials are:
- **Username:** admin
- **Password:** (blank, press enter)

**On Forti-Left, assign WAN and LAN IPs:**

```text
config system interface
    edit port2
        set alias "WAN-TO-ISP"
        set mode static
        set ip 172.16.10.1 255.255.255.252
        set allowaccess ping https ssh http
        set mtu 1500
    next
    edit port3
        set alias "LAN-INTERNAL"
        set mode static
        set ip 10.10.10.1 255.255.255.0
        set allowaccess ping https ssh http
        set mtu 1500
    next
end
```

**Identical on Forti-Right (with different IPs):**

```text
config system interface
    edit port2
        set alias "WAN-TO-ISP"
        set mode static
        set ip 172.16.20.1 255.255.255.252
        set allowaccess ping https ssh http
        set mtu 1500
    next
    edit port3
        set alias "LAN-INTERNAL"
        set mode static
        set ip 20.20.20.1 255.255.255.0
        set allowaccess ping https ssh http
        set mtu 1500
    next
end
```

---

#### 2.2 Static Routes & Baseline Policies

Before encryption, we need basic routing and firewall policies to allow traffic between branches via cleartext.

**On Forti-Left:**

```text
config router static
    edit 1
        set destination 20.20.20.0 255.255.255.0
        set gateway 172.16.15.1
        set device port2
        set comment "Route to Branch-B via internet"
    next
end

# Firewall policy: LAN → WAN (outbound)
config firewall policy
    edit 1
        set name "LAN-to-WAN"
        set srcintf port3
        set dstintf port2
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
        set logtraffic all
    next
end

# Firewall policy: WAN → LAN (return traffic)
config firewall policy
    edit 2
        set name "WAN-to-LAN"
        set srcintf port2
        set dstintf port3
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
        set logtraffic all
    next
end
```

**Mirror on Forti-Right** (with swapped IP subnets).

---

#### 2.3 Verify Cleartext Connectivity

```bash
# From VPC-A (10.10.10.5)
ping 20.20.20.5

# Should receive replies (if routing is correct)
# This confirms the tunnel infrastructure works *before* encryption
```

If ping fails, check:
```bash
# On Forti-Left
diag ip route list
# Verify 20.20.20.0/24 routes via 172.16.15.1

show system interface
# Verify port2 and port3 are up

diag firewall policy list
# Check policy IDs and active connection counts
```

---

### Phase 3: Site-to-Site IPsec VPN

#### 3.1 IPsec Concepts & Phase 1 vs Phase 2

**IPsec = Internet Protocol Security**

Two phases:
- **Phase 1 (IKE - Internet Key Exchange):**
  - Routers authenticate to each other (pre-shared key)
  - Negotiate encryption algorithms for the control channel
  - Exchange key material (Diffie-Hellman)
  - Result: encrypted tunnel for Phase 2 negotiation
  - Timeout: 28,800 seconds (8 hours) by default

- **Phase 2 (ESP - Encapsulating Security Payload):**
  - Negotiate encryption & authentication for actual data
  - Define subnets allowed through the tunnel
  - Exchange encryption keys derived from Phase 1
  - Timeout: 3,600 seconds (1 hour) by default

**Why this matters:** If Phase 1 fails (bad PSK, incompatible algorithms), Phase 2 never starts and the tunnel never comes up.

---

#### 3.2 FortiOS IPsec Wizard

FortiOS has a simplified wizard for VPN setup. We'll use it, then explain what it creates.

**On Forti-Left:**

```text
# Access via GUI or CLI
config vpn ipsec phase1-interface
    edit "site-to-site-right"
        set interface port2
        set peertype any
        set peerid "branch-b"
        set peer 172.16.20.1
        set psksecret "FortiGate@123!"
        set proposal aes128-sha256
        set comments "IPsec to Branch-B"
        set nattraversal disable
        set dpd on-idle
    next
end

config vpn ipsec phase2-interface
    edit "site-to-site-right-p2"
        set phase1name "site-to-site-right"
        set proposal aes128-sha256
        set pfs-dh-group 14
        set replay on
        set autokey-protocol esp
        set src-addr-type name
        set src-name "Local-Branch-Subnet"
        set dst-addr-type name
        set dst-name "Remote-Branch-Subnet"
    next
end
```

**Mirror on Forti-Right:**

```text
config vpn ipsec phase1-interface
    edit "site-to-site-left"
        set interface port2
        set peertype any
        set peerid "branch-a"
        set peer 172.16.10.1
        set psksecret "FortiGate@123!"
        set proposal aes128-sha256
        set comments "IPsec to Branch-A"
        set nattraversal disable
        set dpd on-idle
    next
end

config vpn ipsec phase2-interface
    edit "site-to-site-left-p2"
        set phase1name "site-to-site-left"
        set proposal aes128-sha256
        set pfs-dh-group 14
        set replay on
        set autokey-protocol esp
        set src-addr-type name
        set src-name "Remote-Branch-Subnet"
        set dst-addr-type name
        set dst-name "Local-Branch-Subnet"
    next
end
```

---

#### 3.3 Create Address Objects

FortiOS needs to know what subnets are "Local" and "Remote":

**On Forti-Left:**

```text
config firewall address
    edit "Local-Branch-Subnet"
        set subnet 10.10.10.0 255.255.255.0
        set comment "Branch-A LAN"
    next
    edit "Remote-Branch-Subnet"
        set subnet 20.20.20.0 255.255.255.0
        set comment "Branch-B LAN"
    next
end
```

**On Forti-Right:**

```text
config firewall address
    edit "Local-Branch-Subnet"
        set subnet 20.20.20.0 255.255.255.0
        set comment "Branch-B LAN"
    next
    edit "Remote-Branch-Subnet"
        set subnet 10.10.10.0 255.255.255.0
        set comment "Branch-A LAN"
    next
end
```

---

#### 3.4 VPN Policies & Routing

Cleartext policies are **removed**. New policies route traffic through the VPN tunnel:

**On Forti-Left:**

```text
config firewall policy
    edit 1
        set name "LAN-to-VPN"
        set srcintf port3
        set dstintf site-to-site-right
        set srcaddr "Local-Branch-Subnet"
        set dstaddr "Remote-Branch-Subnet"
        set action accept
        set schedule always
        set service ALL
        set logtraffic all
    next
end

config router static
    edit 1
        set destination 20.20.20.0 255.255.255.0
        set device site-to-site-right
        set comment "Route via VPN tunnel"
    next
end
```

**On Forti-Right:**

```text
config firewall policy
    edit 1
        set name "LAN-to-VPN"
        set srcintf port3
        set dstintf site-to-site-left
        set srcaddr "Local-Branch-Subnet"
        set dstaddr "Remote-Branch-Subnet"
        set action accept
        set schedule always
        set service ALL
        set logtraffic all
    next
end

config router static
    edit 1
        set destination 10.10.10.0 255.255.255.0
        set device site-to-site-left
        set comment "Route via VPN tunnel"
    next
end
```

---

#### 3.5 Verify IPsec Tunnel Establishment

```bash
# Check tunnel status
diagnose vpn tunnel list

# Expected output:
# name=site-to-site-right state=up
# ipaddr=172.16.20.1 id=20.20.20.0/24
```

**If tunnel is DOWN, diagnose:**

```bash
# Check Phase 1 status
diagnose vpn ike log-filter

# Common failures:
# - IKE packet received but PSK mismatch → "no suitable proposal"
# - Peer unreachable → "timeout on initial contact"
# - Algorithm mismatch → "no common encryption"

# Check Phase 2 status
diagnose vpn ipsec tunnel name "site-to-site-right"

# Test tunnel by sending traffic
# From VPC-A:
ping 20.20.20.5

# Monitor in real-time
diagnose vpn ipsec tunnel brief
```

---

#### 3.6 Real-World VPN Troubleshooting

**Scenario: Tunnel Down, No Data Flow**

1. **Check Phase 1:**
   ```bash
   # Phase 1 should show "IPsecSA"
   diagnose vpn ike engine-stats
   # If 0, Phase 1 isn't negotiating
   ```

2. **Check PSK:**
   ```bash
   # Display configured PSK (will be masked in output)
   show vpn ipsec phase1-interface "site-to-site-right"
   # Verify it matches the remote end
   ```

3. **Check DPD (Dead Peer Detection):**
   ```bash
   # If DPD is enabled and detects peer is dead, tunnel may restart
   # For active troubleshooting, disable temporarily
   set dpd off
   ```

4. **Restart IKE daemon:**
   ```bash
   # Nuclear option: restart the key exchange service
   diagnose vpn ike restart
   ```

---

### Phase 4: Remote Access (Dialup) VPN

#### 4.1 Remote Access Use Case

Site-to-Site VPNs assume both ends are trusted branch offices with static IPs. But remote workers:
- Have dynamic IPs (home WiFi, mobile hotspot, coffee shop)
- Are on untrusted networks
- Require per-user authentication, not pre-shared keys

**Remote Access VPN** uses a different model:
- Central VPN gateway (our Forti-Left)
- FortiClient software on remote endpoints
- User-based authentication (username/password or certificate)
- Dynamic IP assignment from a DHCP pool

---

#### 4.2 FortiGate Remote Access Configuration

**On Forti-Left:**

```text
config vpn ssl settings
    set servercert FQDN_vpnserver
    set tunnel-ip-pools "VPNPOOL"
end

config firewall address
    edit "VPNPOOL"
        set subnet 192.168.100.0 255.255.255.0
        set comment "Remote access client pool"
    next
end

config vpn ssl web portal
    edit "portal1"
        set portal-name "Corporate-VPN"
        set tunnelmode enable
    next
end
```

**Create user accounts:**

```text
config user local
    edit "john.doe"
        set type password
        set passwd "SecurePassword123!"
        set email "john.doe@company.com"
    next
    edit "jane.smith"
        set type password
        set passwd "SecurePassword456!"
        set email "jane.smith@company.com"
    next
end
```

**Create a user group for VPN access:**

```text
config user group
    edit "vpn-users"
        set member "john.doe" "jane.smith"
        set comment "Remote workers"
    next
end
```

**VPN policy:**

```text
config firewall policy
    edit 1
        set name "SSLVPN-to-LAN"
        set srcintf ssl.root
        set dstintf port3
        set srcaddr all
        set dstaddr "Local-Branch-Subnet"
        set action accept
        set schedule always
        set service ALL
        set logtraffic all
        set groups "vpn-users"
    next
end
```

---

#### 4.3 FortiClient Configuration (Remote End)

FortiClient is the VPN client software that remote workers install.

**Download:** [FortiClient 6.2.6 or 7.x](https://fortinet.com/support/product-downloads)

**Configuration Steps:**

1. **Open FortiClient → VPN**

2. **Add Connection:**
   - **Name:** Corporate VPN
   - **Server Address:** 172.16.10.1 (Forti-Left WAN IP)
   - **Port:** 443 (default SSL VPN port)
   - **VPN Type:** SSL-TLS
   - **Username:** john.doe
   - **Save Password:** (optional, for convenience)

3. **Advanced Settings (optional):**
   - **Certificate Verification:** Disable for self-signed certs (in labs)
   - **Compression:** Enable (saves bandwidth)
   - **Anti-Virus Integration:** Enable (if AV is installed)

---

#### 4.4 Tunnel Establishment & Verification

**Connect from FortiClient:**

```
1. Click "Connect"
2. Enter password when prompted
3. Wait for handshake (10-15 seconds)
4. Status should show "Connected" with a green light
```

**Verify on FortiGate side:**

```bash
# Check active SSL VPN sessions
diagnose vpn ssl list

# Expected output:
# UserName         Ip          Proto  Cipher         Auth
# john.doe         192.168.100.2 TLS  AES-GCM        PASS

# Check dynamic IP assignment
show system dhcp server
# Should show VPNPOOL with active leases

# Verify firewall policy hit counts
show firewall policy 1
# Counter should increment as remote user sends traffic
```

---

#### 4.5 Test Remote Access Connectivity

**From the remote FortiClient machine:**

```bash
# Ping a server on the corporate LAN
ping 10.10.10.5

# Browse to internal intranet
curl http://10.10.10.100/portal

# RDP to a corporate workstation
mstsc 10.10.10.50

# Verify traffic is encrypted
# (Can't sniff cleartext on the wire)
```

**If connectivity fails, diagnose:**

```bash
# On FortiGate
diagnose vpn ssl user list

# Check if user is in the VPN group
show user group "vpn-users"

# Check firewall policy is allowing traffic
diagnose firewall policy list | grep SSLVPN

# Enable SSL VPN debug
diagnose debug application sslvpn -1
```

---

## Summary & Key Takeaways

### What You Now Understand

✓ **Layer 3 Routing:**
- Multi-area OSPF for scalable, dynamic network design
- How routers converge when topologies change
- Troubleshooting adjacency and route propagation

✓ **Incident Response at the Network Edge:**
- Stateless firewall filters for immediate threat containment
- How to block compromised hosts, enforce policies, and neutralize threats
- Real-world deployment timelines and verification procedures

✓ **Encrypted Transport:**
- IPsec Phase 1 (IKE key exchange) and Phase 2 (ESP data encryption)
- Site-to-Site VPN for trusted branch office communication
- Remote Access VPN for untrusted remote workers

✓ **SOC Relevance:**
- You can now reason about network compromise detection
- Understand why certain ports are blocked or tunnels are down
- Identify when encrypted traffic indicates policy violations
- Troubleshoot network issues that impact incident response

---

### Next Steps

1. **Build this lab yourself.** Hands-on experience cements understanding far better than reading.

2. **Modify and experiment:**
   - Add a third branch office (Area 2)
   - Implement OSPF authentication (prevents route hijacking)
   - Deploy access lists on multiple routers
   - Simulate link failures and observe convergence

3. **Integrate with security tools:**
   - Syslog OSPF neighbor changes to your SIEM
   - Alert on firewall filter drops
   - Monitor VPN tunnel status

4. **Study for certifications:**
   - **Juniper:** JNCIS-ENT (Enterprise Routing & Switching)
   - **Fortinet:** NSE 4 / NSE 5 (FortiGate Administration & NSE)
   - **CompTIA:** Security+, Network+

---

### Lab Files & References

- **Junos CLI Reference:** https://www.juniper.net/documentation/
- **FortiOS CLI Reference:** https://docs.fortinet.com/
- **EVE-NG:** https://www.eve-ng.net/
- **PNET Lab:** http://pnetlab.com/

---

**Built as part of BARQ Systems network and cybersecurity training program.**
**Portfolio: [uphillpush.github.io/Yahia-Osama-SOC-Analyst-Portfolio](https://uphillpush.github.io/)**
