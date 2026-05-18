<![CDATA[<div align="center">

# 🏥 Clinique Le Châtelet — Secure Healthcare Infrastructure

**A production-grade, segmented network infrastructure protecting medical records (HDS/HIPAA)**  
**Deployed on VMware ESXi with HA firewalls, SIEM, MFA, and automated threat response**

[![OPNsense](https://img.shields.io/badge/Firewall-OPNsense_26.1.3-D94F00?style=for-the-badge&logo=opnsense&logoColor=white)](configs/opnsense/)
[![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.14.4-3CBCB4?style=for-the-badge&logo=wazuh&logoColor=white)](configs/wazuh/)
[![Zabbix](https://img.shields.io/badge/Monitoring-Zabbix_7.4.9-D40000?style=for-the-badge&logo=zabbix&logoColor=white)](configs/zabbix/)
[![VMware](https://img.shields.io/badge/Hypervisor-ESXi_7.0_U2-607078?style=for-the-badge&logo=vmware&logoColor=white)](#architecture)
[![Windows Server](https://img.shields.io/badge/AD-Windows_Server_2022-0078D4?style=for-the-badge&logo=windows&logoColor=white)](#active-directory)
[![Oracle](https://img.shields.io/badge/Database-Oracle_21c-F80000?style=for-the-badge&logo=oracle&logoColor=white)](#oracle-database)

[![HDS](https://img.shields.io/badge/Compliance-HDS_(FR_Health_Data)-00A86B?style=flat-square)](#compliance)
[![ISO 27001](https://img.shields.io/badge/Compliance-ISO_27001-0066CC?style=flat-square)](#compliance)
[![RGPD](https://img.shields.io/badge/Compliance-RGPD_Art.32-6C3483?style=flat-square)](#compliance)
[![PCI DSS](https://img.shields.io/badge/Compliance-PCI_DSS-1A1A2E?style=flat-square)](#compliance)

---

*Master's project in Network Infrastructure & Security — Designed, deployed, and pentested from scratch on a physical ESXi host.*

</div>

---

## 🎯 The Challenge

A rehabilitation clinic needed to expose its **legacy Oracle-based medical application** to the web, allowing remote clinicians to access patient records via a secure portal — while meeting **French HDS** (Health Data Hosting) requirements and maintaining **99.99% availability**.

**The problem:** No firewall. No segmentation. No authentication. Workstations with local admin rights. DNS pointing to the ISP box. A monolithic Oracle database directly accessible on the LAN.

**The solution:** A complete infrastructure redesign with defense-in-depth, microsegmentation, HA failover, MFA, SIEM with custom detection rules, and automated threat containment.

---

## 🏗️ Architecture

<div align="center">

```
                         ┌──────────────┐
                         │   INTERNET   │
                         └──────┬───────┘
                                │
                         ┌──────┴───────┐
                         │  Sophos UTM  │
                         │  WAN VLAN 65 │
                         └──────┬───────┘
                                │
                ┌───────────────┴───────────────┐
                │    OPNsense HA Cluster         │
                │  ┌─────────┐   ┌─────────┐    │
                │  │  FW1    │◄─►│  FW2    │    │
                │  │ MASTER  │pf │ BACKUP  │    │
                │  │  Sync   │   │         │    │
                │  └─────────┘   └─────────┘    │
                │     6 CARP VIPs (.254)         │
                └───────────────┬───────────────┘
                                │
                        802.1Q Trunk (VGT 4095)
                                │
          ┌─────────┬───────────┼───────────┬──────────┐
          │         │           │           │          │
    ┌─────┴─────┐ ┌─┴───────┐ ┌┴────────┐ ┌┴───────┐ ┌┴────────┐
    │ VLAN 111  │ │VLAN 333 │ │VLAN 444 │ │VLAN 555│ │VLAN 999 │
    │   SRV     │ │  DMZ    │ │ BACKUP  │ │ GUEST  │ │  MGMT   │
    │───────────│ │─────────│ │─────────│ │────────│ │─────────│
    │ DC1 + DC2 │ │ Nginx   │ │ Veeam   │ │Isolated│ │ Bastion │
    │ Oracle 21c│ │Authelia │ │ B&R 13  │ │Internet│ │Admin IT │
    │ Zabbix    │ │Web App  │ │         │ │  only  │ │BitLocker│
    │ Wazuh     │ │Mailpit  │ │         │ │        │ │ 9 GPOs  │
    │ GLPI      │ │         │ │         │ │        │ │         │
    └───────────┘ └─────────┘ └─────────┘ └────────┘ └─────────┘
     172.16.11.0   172.16.33.0 172.16.44.0 172.16.55.0 172.16.99.0
```

</div>

> **Hybrid Router-on-a-Stick + Access Ports** on VMware ESXi 7.0 U2 — Firewalls handle 802.1Q tagging via VLAN 4095 (VGT), while all other VMs connect to dedicated access port groups. This combines the flexibility of trunk-based routing with the simplicity of per-VM VLAN assignment.

---

## 📊 Key Metrics

| Component | Metric | Value |
|---|---|---|
| **OPNsense** | Firewall rules (pfctl) | **208 rules** across 19 sections |
| | Object-group aliases | **35 aliases** (hosts, networks, ports) |
| | CARP virtual IPs | **6 VHIDs** — transparent failover |
| | Active states | 1,166 concurrent connections |
| **Wazuh** | Custom HDS detection rules | **35 rules** (200000→200501) |
| | Active agents | **12 endpoints** (4 OS families) |
| | Total alerts collected | **92,488** |
| | Automated active responses | **8 containment rules** |
| **Zabbix** | Monitored hosts | **14** (dual-stack on firewalls) |
| | Active items | **20,088** |
| | Active triggers | **7,728** |
| | Throughput | **19.67 values/second** |
| **Infrastructure** | Virtual machines on ESXi | **13 VMs** |
| | VLAN segments | **6 isolated zones** |
| | MFA-protected users | **6 accounts** (TOTP) |
| | Backup compression ratio | **5.5x** (80 GB in 36 min) |

---

## 🛡️ Security Stack

### Firewall — OPNsense HA Cluster

Two OPNsense 26.1.3 firewalls in active/passive CARP cluster with pfSync state synchronization on a dedicated link. Default Deny policy on all interfaces with anti-spoofing, anti-bogons (RFC1918 + RFC6598), and dynamic threat tables (`sshlockout`, `virusprot`).

**→** [Annotated pfctl ruleset](configs/opnsense/fw1-rules-annotated.conf) ・ [Complete flux matrix](docs/assets/diagrams/flux-matrix.md) ・ [Aliases inventory](configs/opnsense/aliases-inventory.md) ・ [CARP HA documentation](configs/opnsense/carp-ha-config.md)

### SIEM — Wazuh XDR with Custom HDS Rules

35 custom detection rules organized in 11 categories, mapped to **PCI DSS**, **HIPAA**, and **MITRE ATT&CK** frameworks. Automated active response blocks threats in real-time using `firewall-drop` (Linux), `pf` (FreeBSD/OPNsense), and `netsh` (Windows).

<details>
<summary><b>Detection categories (click to expand)</b></summary>

| Category | Rules | Examples |
|---|---|---|
| Authentication | 200000–200007 | SSH/AD brute force, root login, off-hours access |
| Privilege Escalation | 200050–200054 | sudo abuse, AD group changes, account lifecycle |
| Anti-Forensics | 200100–200101 | Windows log clearing (T1070.001), Pass-the-Hash (T1550.002) |
| File Integrity | 200150–200160 | Config tampering, **ransomware extension detection** |
| Network / Firewall | 200200–200212 | Scan detection, VLAN BACKUP intrusion, VPN brute force |
| Veeam Backup | 200350–200352 | Backup failures, Oracle backup critical |
| Oracle Database | 200400–200402 | ORA- errors, ORA-00600 internal, login failures |
| Availability | 200500–200501 | Service stopped, system reboot/shutdown |

</details>

**→** [Custom rules XML](configs/wazuh/local_rules.xml) ・ [SIEM documentation](configs/wazuh/wazuh-siem-documentation.md) ・ [Manager config](configs/wazuh/ossec.conf)

### Monitoring — Zabbix 7.4.9

14 hosts monitored across all VLANs with dual-stack observation on firewalls (Zabbix agent + SNMP). Oracle database has 213 dedicated items with custom performance dashboards. Email alerts routed through Mailpit SMTP relay in the DMZ.

**→** [Monitoring documentation](configs/zabbix/zabbix-monitoring-documentation.md) ・ [Template export](configs/zabbix/custom-templates.yaml)

### Access Control — MFA Everywhere

| Access Path | Authentication | Technology |
|---|---|---|
| Web portal (patients records) | Username + TOTP | Authelia v4.39.15 via Nginx forward-auth |
| VPN (remote clinicians) | LDAP + TOTP | OpenVPN + OPNsense OTP seeds |
| Admin console (OPNsense) | Local + TOTP | OPNsense built-in MFA |
| Domain login | Kerberos + GPO | Active Directory (Windows Server 2022) |

### Backup & Recovery — Veeam B&R 13

Sanctuarized VLAN 444 with restricted access. Oracle full backup: **80 GB compressed to 14.5 GB** (5.5x ratio) in 36 minutes with VMware CBT (Changed Block Tracking).

---

## 🔬 Penetration Testing Results

Offensive security assessment performed from **Kali Linux** positioned in VLAN GUEST (555) — zero prior knowledge of the internal network.

| Test | Result | Evidence |
|---|---|---|
| Internal VLAN reachability from GUEST | ❌ **0 hosts visible** | All RFC1918 blocked at firewall |
| Gateway port scan (65,535 ports) | ❌ **All filtered** | No services exposed on gateway |
| DNS to internal DCs from GUEST | ❌ **Blocked** | Complete DNS isolation |
| VLAN hopping (802.1Q double-tag) | ❌ **Blocked** | Trunk only on firewall interfaces |
| DNS tunneling (Base64 exfiltration) | ❌ **Timeout** | No DNS forwarding from GUEST |
| Proxy detection (3128, 8080) | ❌ **Timeout** | No open proxies |
| XSS/Path traversal on DMZ portal | 🔔 **Detected by Wazuh** | Rule 31153 + Active Response triggered |
| Network scan (Kali → internal) | 🔔 **Detected + Blocked** | Rule 200200 → `firewall-drop` |

---

## 📋 Compliance

| Framework | Scope | Key Controls Implemented |
|---|---|---|
| **HDS** (FR) | Health data hosting | Network segmentation, encryption, access control, audit logging, backup |
| **ISO 27001** | Information security | Risk assessment, ISMS controls, incident response, business continuity |
| **RGPD Art. 32** | Data protection | Pseudonymization, encryption, availability, resilience, regular testing |
| **PCI DSS** | Payment card (methodology) | Firewall config, default deny, log monitoring, vulnerability management |
| **HIPAA** | Health data (US, methodology) | Access controls, audit controls, integrity, transmission security |

---

## 🗂️ Repository Structure

```
├── configs/
│   ├── opnsense/          # Firewall rules, aliases, CARP HA, NAT (anonymized)
│   ├── wazuh/             # SIEM rules, manager config, agent config
│   ├── zabbix/            # Monitoring templates, documentation
│   ├── nginx/             # Reverse proxy + Authelia forward-auth (coming)
│   ├── authelia/          # MFA configuration (coming)
│   └── gpo/               # Active Directory GPO inventory (coming)
│
├── docs/
│   ├── assets/
│   │   ├── screenshots/   # Anonymized evidence from the lab
│   │   └── diagrams/      # Flux matrix, architecture diagrams
│   ├── 01-context-business.md
│   ├── 02-architecture-overview.md
│   ├── 03-firewall-ha-carp.md        # Mission 1
│   ├── 04-monitoring-zabbix.md        # Mission 2
│   ├── 05-ad-gpo-hardening.md         # Mission 3
│   ├── 06-workstation-hardening.md    # Mission 4
│   ├── 07-backup-veeam.md            # Mission 5
│   ├── 08-mfa-authelia-openvpn.md     # Mission 6
│   ├── 09-itsm-glpi.md               # Mission 7
│   ├── 10-siem-wazuh.md              # Mission 10
│   ├── 11-pentest-results.md
│   └── 12-compliance-matrix.md
│
├── scripts/
│   ├── deploy/            # Infrastructure automation scripts
│   ├── hardening/         # CIS baseline + GPO audit scripts
│   └── pentest/           # Reconnaissance and audit scripts
│
├── compliance/            # ISO 27001, HDS, RGPD mapping matrices
└── mkdocs.yml             # Documentation site config (GitHub Pages)
```

---

## 🛠️ Tech Stack

| Layer | Technology | Role |
|---|---|---|
| Hypervisor | VMware ESXi 7.0 U2 | VM hosting, vSwitch, port groups |
| Firewall | OPNsense 26.1.3 (FreeBSD 14.3) | HA cluster, CARP, pfSync, OpenVPN, NAT |
| Directory | Windows Server 2022 (AD DS) | Kerberos, DNS, DHCP, GPO, FSMO |
| SIEM/XDR | Wazuh v4.14.4 | FIM, Active Response, custom rules, CVE detection |
| Monitoring | Zabbix 7.4.9 | Agent + SNMP, triggers, email alerts |
| MFA Portal | Authelia v4.39.15 + Nginx | Forward-auth, TOTP, LDAPS backend |
| Database | Oracle 21c (Oracle Linux 9) | Patient records (HDS-grade) |
| Backup | Veeam B&R 13.0.1 | Agentless VMware, CBT, 5.5x compression |
| ITSM | GLPI 10.0.15 | Ticketing, asset management, LDAP sync |
| Alerts | Mailpit | SMTP relay for Zabbix + Wazuh notifications |
| Remote access | OpenVPN + Tailscale | VPN tunnel (MFA) + OOB management overlay |
| Workstation | Windows 10 Pro 22H2 | BitLocker, UAC max, Defender NIS, 9 GPOs |

---

## 📸 Screenshots

<details>
<summary><b>OPNsense — Firewall HA Cluster</b></summary>

| Dashboard | CARP Status (FW1 MASTER) | CARP Status (FW2 BACKUP) |
|---|---|---|
| ![Dashboard](docs/assets/screenshots/opnsense-01-dashboard-fw1.png) | ![CARP FW1](docs/assets/screenshots/opnsense-02-carp-status-fw1.png) | ![CARP FW2](docs/assets/screenshots/opnsense-03-carp-status-fw2.png) |

</details>

<details>
<summary><b>Wazuh — SIEM / XDR</b></summary>

| Overview (11 agents) | Threat Hunting (MITRE) | Active Response (XSS detected) |
|---|---|---|
| ![Overview](docs/assets/screenshots/wazuh-01-dashboard-overview.png) | ![Threat Hunting](docs/assets/screenshots/wazuh-03-threat-hunting.png) | ![Active Response](docs/assets/screenshots/wazuh-06-active-response-xss.png) |

</details>

<details>
<summary><b>Zabbix — Monitoring</b></summary>

| Dashboard (14 hosts) | Oracle Performance | Alert Problems |
|---|---|---|
| ![Dashboard](docs/assets/screenshots/zabbix-01-dashboard.png) | ![Oracle](docs/assets/screenshots/zabbix-03-oracle-cpu-memory.png) | ![Problems](docs/assets/screenshots/zabbix-04-alert-problems.png) |

</details>

---

## 🚀 What I Learned

This project pushed me to design, deploy, and defend an entire infrastructure from scratch. Key takeaways:

- **Defense in depth** is not a buzzword — each layer (firewall → segmentation → MFA → SIEM → backup) caught threats the others missed
- **Custom SIEM rules** aligned to compliance frameworks turn a SIEM from a log aggregator into a real detection engine
- **Penetration testing your own infrastructure** reveals blind spots that configuration reviews cannot find
- **Documentation is a security control** — if it's not documented, it doesn't exist for auditors

---

## 📄 License

This project is released under the [MIT License](LICENSE). Configuration files are sanitized — no secrets, credentials, or real patient data are included in this repository.

---

<div align="center">

**Built with 🔒 by [Steeve WOMO]**

*Master 2 — Infrastructure Réseaux et Sécurité*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin)](https://linkedin.com/in/steeve-womo)

</div>
]]>