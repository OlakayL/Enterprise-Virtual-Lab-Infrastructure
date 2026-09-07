# Enterprise-Virtual-Lab-Infrastructure
A virtual enterprise lab environment demonstrating multi-subnet architecture, inter-VLAN routing, perimeter firewall administration, and centralized identity management using pfSense and Windows Server 2022.

---

## Overview & Key Objectives

* **Network Segmentation & Layer 2/3 Routing:** Configured VirtualBox internal networks (`LabNet_LAN` and `LabNet_OPT1`) to isolate core domain infrastructure from user workstation subnets, using **pfSense** as the inter-VLAN gateway.
* **Perimeter Firewall Administration:** Enforced stateful access control policies on pfSense to pass necessary cross-subnet administrative protocols while maintaining default-deny boundaries.
* **Cross-Subnet Active Directory Services:** Configured cross-segment DNS resolution and host firewall policies to join client workstations across routed firewall interfaces into the Active Directory domain.

---

## Network & Subnet Specifications

| Interface / Segment | Virtual Network Name | Subnet Range | Primary Function |
| :--- | :--- | :--- | :--- |
| **WAN (`em0`)** | VirtualBox NAT | `10.0.2.0/24` (DHCP) | Outbound internet access via host system |
| **LAN (`em1`)** | `LabNet_LAN` | `192.168.10.0/24` | Core Infrastructure (DC01, AD DS, DNS) |
| **OPT1 (`em2`)** | `LabNet_OPT1` | `192.168.20.0/24` | Corporate Client Workstations |

---

## Firewall & Traffic Flow Rules

### pfSense Interface Rules (OPT1)
To allow client endpoints on `OPT1` (`192.168.20.0/24`) to reach the Domain Controller on `LAN` (`192.168.10.10`), specific pass rules were configured on the OPT1 interface:

* **Pass Rule:** `IPv4` | **Protocol:** `ANY` | **Source:** `OPT1 net` | **Destination:** `192.168.10.10`
* **Pass Rule (ICMP):** Allowed ICMP Echo Requests from `OPT1 net` to `LAN net` for diagnostic verification.

---

## Deployment & Verification Steps

1. **Core Gateway Configuration (pfSense)**
   * Deployed pfSense virtual appliance and mapped physical interface instances (`em0` = WAN, `em1` = LAN, `em2` = OPT1).

2. **Domain Controller & DNS Setup (DC01)**
   * Configured static IPv4 addressing (`192.168.10.10/24`, Gateway: `192.168.10.1`, DNS: `127.0.0.1`).
   * Installed AD DS role and promoted `DC01` to root Domain Controller for `corpinternal.com`.
   * Configured DNS upstream forwarders to redirect non-authoritative external queries to pfSense (`192.168.10.1`).

3. **Cross-Subnet Domain Join (WIN10-01)**
   * Configured Windows 10 endpoint on `LabNet_OPT1` with static IP (`192.168.20.50/24`), Gateway (`192.168.20.1`), and Primary DNS pointing to `192.168.10.10`.
   * Verified Active Directory `SRV` record lookup via `nslookup corpinternal.com`.
   * Successfully executed domain join and verified remote authentication as `CORPINTERNAL\kbrass`.

---

## Key Troubleshooting & Technical Learnings

* **ICMP Drop Isolation:** Resolved cross-subnet ping timeouts by identifying local host firewall restrictions on Windows Server 2022 that drop ICMP traffic originating outside the local broadcast domain.
* **Virtualization Layer Routing:** Ensured strict Layer 2 separation by using VirtualBox's software-defined internal switches (`LabNet_LAN` and `LabNet_OPT1`), routing all inter-host communications exclusively through pfSense.

---

## Network Topology

```text
               +--------------------------------+
               |         Physical Host          |
               |     (Internet / Home Net)      |
               +---------------+----------------+
                               |
                     (VirtualBox NAT / em0)
                               |
               +---------------+----------------+
               |        pfSense Firewall        |
               |      WAN: 10.0.2.15/24 (DHCP)  |
               +-------+----------------+-------+
                       |                |
      (LAN: 192.168.10.1)              (OPT1: 192.168.20.1)
       em1 / LabNet_LAN                 em2 / LabNet_OPT1
                       |                |
      +----------------+                +----------------+
      |                                                  |
+-----+------------------+                      +--------+---------------+
|   DC01 (WinServer)     |                      |   WIN10-01 (Client)    |
| IP:  192.168.10.10     |                      | IP:  192.168.20.50     |
| GW:  192.168.10.1      |                      | GW:  192.168.20.1      |
| DNS: 127.0.0.1         |                      | DNS: 192.168.10.10     |
+------------------------+                      +------------------------+

---

## Tools & Technologies Used

* **Hypervisor:** Oracle VM VirtualBox
* **Firewall / Router:** pfSense 2.7.x
* **Operating Systems:** Windows Server 2022, Windows 10 Enterprise
* **Services & Protocols:** Active Directory Domain Services (AD DS), DNS, DHCP, ICMP, LDAP, IPv4 Routing
