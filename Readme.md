# FortiGate Network Security & Resilient SD-WAN Lab
### Comprehensive Dual-Firewall Enterprise Infrastructure Deployment in EVE-NG

![Topology Diagram](topology.png)

## 📌 Executive Overview
This repository contains the complete network designs, architecture plans, configuration files, and diagnostic troubleshooting logs for an enterprise-grade Fortinet FortiGate deployment built inside the EVE-NG emulation platform. 

The lab implements a multi-homed, redundant branch-to-headquarters network exercising enterprise access control, Layer-8 authentication, flow-based security profiling (UTM), dual route-based IPsec VPNs, and SD-WAN health probing.

---

## 🏛 Architecture & Topology Specifications

### System Components
* **FGT-HQ:** FortiGate NGFW Appliance (Virtual KVM, FortiOS v7.x)
* **FGT-Branch:** FortiGate NGFW Appliance (Virtual KVM, FortiOS v7.x)
* **PC-HQ & PC-Branch:** Virtual PC Simulators (VPCS)
* **Management Network:** Out-of-band management network connected to VMware VMnet

### Physical & Virtual Port Map
```
                       +---------------------------------------+
                       |              WAN 1 (Primary)          |
                       |       198.51.100.0/30 (port3/port3)   |
                       +---------------------------------------+
                                  /                 \
                             port3                   port3
                     +-------------------+   +-------------------+
                     |      FGT-HQ       |   |    FGT-Branch     |
                     |  192.168.131.129  |   |  192.168.131.130  |
                     +-------------------+   +-------------------+
                             port4                   port4
                                  \                 /
                       +---------------------------------------+
                       |              WAN 2 (Backup)           |
                       |       198.51.200.0/30 (port4/port4)   |
                       +---------------------------------------+
                                  |                 |
                             port2                   port2
                         10.10.10.1/24           10.20.20.1/24
                                  |                 |
                             [Direct L2]         [Direct L2]
                                  |                 |
                             +---------+         +---------+
                             |  PC-HQ  |         |PC-Branch|
                             +---------+         +---------+
                            10.10.10.100        10.20.20.100
```

---

## 🌐 IP Addressing Scheme

| Device | Interface | Subnet | IP Address | Gateway | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FGT-HQ** | `port1` | `192.168.131.0/24` | `192.168.131.129` | — | Out-of-band Management |
| **FGT-HQ** | `port2` | `10.10.10.0/24` | `10.10.10.1` | — | HQ Internal LAN Default Gateway |
| **FGT-HQ** | `port3` | `198.51.100.0/30` | `198.51.100.1` | — | WAN 1 Underlay Link (Primary) |
| **FGT-HQ** | `port4` | `198.51.200.0/30` | `198.51.200.1` | — | WAN 2 Underlay Link (Backup) |
| **FGT-Branch** | `port1` | `192.168.131.0/24` | `192.168.131.130` | — | Out-of-band Management |
| **FGT-Branch** | `port2` | `10.20.20.0/24` | `10.20.20.1` | — | Branch Internal LAN Default Gateway |
| **FGT-Branch** | `port3` | `198.51.100.0/30` | `198.51.100.2` | — | WAN 1 Underlay Link (Primary) |
| **FGT-Branch** | `port4` | `198.51.200.0/30` | `198.51.200.2` | — | WAN 2 Underlay Link (Backup) |
| **PC-HQ** | `eth0` | `10.10.10.0/24` | `10.10.10.100` | `10.10.10.1` | Virtual Workstation (HQ) |
| **PC-Branch** | `eth0` | `10.20.20.0/24` | `10.20.20.100` | `10.20.20.1` | Virtual Workstation (Branch) |

---

## 🔒 Security Posture & Configuration Blocks

### 1. Administrative RBAC
Implemented role-based administration with dedicated read-only profiles for compliance auditors:
* `labadmin`: Super Administrator (`super_admin`).
* `auditor`: Read-Only Security Auditor (`super_admin_readonly`).

### 2. User Authentication
* Local identity: `projectuser`
* Directory group: `AUTH_USERS`
* Applied via firewall security policies requiring active credentials before passing inter-site traffic.

### 3. Unified Threat Management (UTM)
* **Flow-Based Antivirus:** Inspects transit files without requiring client-side proxy redirection.
* **Intrusion Prevention System (IPS):** Protects against network exploits across the VPN boundary using the FortiOS default attack signature sensor.

---

## 🚀 IPsec VPN & SD-WAN Implementation

### Redundant Route-Based Tunnels
1. **Primary Tunnel (`HQ-BRANCH-WAN1`):** Established across `port3` ($198.51.100.0/30$).
2. **Secondary Tunnel (`HQ-BRANCH-WAN2`):** Established across `port4` ($198.51.200.0/30$).

### Cryptographic Findings
Due to virtual licensing constraints in unactivated FortiGate VM images, high-level encryption algorithms (`aes256-sha256`) were rejected by the system parser (`Return code -61`). Both tunnels were successfully negotiated using `des-sha256` / `des-md5` cipher suites:
* **Phase 1 (IKE SA):** Established `1/1`
* **Phase 2 (IPsec SA):** Established `1/1`
* **Tunnel Interface Status:** `UP`

### SD-WAN Probing
* Configured members `port3` (Priority 1) and `port4` (Priority 10).
* Deployed health-check performance probes (`WAN1_HEALTH` / `WAN2_HEALTH`) monitoring remote gateway availability.

---

## 🔬 Deep Diagnostics & Verification Matrix

### Verification Suite
```bash
# 1. Check IPsec Tunnels
diagnose vpn ike gateway list
diagnose vpn tunnel list

# 2. Check Routing Tables
get router info routing-table all
get router info routing-table details 10.20.20.100

# 3. Kernel Flow Inspection
diagnose debug flow filter addr 10.20.20.100
diagnose debug flow show function-name enable
diagnose debug flow trace start 10
diagnose debug enable

# 4. Interface Sniffer
diagnose sniffer packet any 'icmp and host 10.20.20.100' 4 0 l
diagnose sniffer packet port3 'proto 50' 4 0 l
```

### Validation Findings
* **Local LAN Communication:** Verified ($PC \rightarrow Gateway$ ping succeeded at $0\%$ loss).
* **Underlay Peering:** Verified (Direct WAN pings between firewalls succeeded).
* **Control Plane Negotiation:** Verified (Both IKE SA and IPsec SA showed `sa=1` and `status=up`).
* **Route & Policy Lookup:** Verified (`matches policy id: 10` and routing engine selected the primary VPN interface).
* **Data-Plane Encapsulation:** Isolated an issue where the kernel allocated sessions but did not write frames to the ESP encryption ring (`enc:pkts/bytes=0/0`), preventing the final ICMP packet from leaving the physical wire.

---

## 📂 Repository Layout
```
fortigate-security-project/
│
├── README.md                          <-- Lab guide and system summary
│
├── topology/
│   ├── topology.png                   <-- EVE-NG network schematic
│   └── architecture-design.md
│
├── configurations/
│   ├── hq/
│   │   ├── interfaces.conf
│   │   ├── firewall-policy.conf
│   │   ├── vpn-ipsec.conf
│   │   ├── routing.conf
│   │   └── sdwan.conf
│   │
│   └── branch/
│       ├── interfaces.conf
│       ├── firewall-policy.conf
│       ├── vpn-ipsec.conf
│       ├── routing.conf
│       └── sdwan.conf
│
├── diagnostics/
│   ├── debug-flow-trace.log
│   ├── iprope-policy-lookup.log
│   ├── vpn-tunnel-sa.log
│   └── packet-sniffer.log
│
└── presentation/
    └── FortiGate_Project_Presentation.md  <-- 16-slide presentation deck
```