# Wazuh SIEM/XDR — Custom HDS Rules & Active Response

> Wazuh v4.14.4 (server) — Ubuntu 22.04 LTS — 12 agents across 5 VLANs  
> 92,488 total alerts | 241 critical (level ≥ 10) | 383 email notifications via Mailpit

## Agent Inventory (12 endpoints — 100% coverage)

| ID | Name | IP | OS | Group | VLAN |
|---|---|---|---|---|---|
| 000 | srv-wazuh | 127.0.0.1 | Ubuntu 22.04 LTS | (manager) | SRV (111) |
| 001 | POSTE-ADMIN-IT | 172.16.99.115 | Windows 10 Pro 10.0.19045 | Admin_Workstation | MGMT (999) |
| 002 | DC1 | 172.16.11.11 | Windows Server 2019 Std | DC_Servers | SRV (111) |
| 003 | srv-zabbix | 172.16.11.40 | Ubuntu 22.04 LTS | Monitoring | SRV (111) |
| 004 | SRV-PROXY | 172.16.33.10 | Ubuntu 22.04 LTS | Web_DMZ | DMZ (333) |
| 005 | SRV-ORACLE-DB | 172.16.11.20 | Oracle Linux | default | SRV (111) |
| 006 | SRV-WEB | 172.16.33.20 | Ubuntu 22.04 LTS | Web_DMZ | DMZ (333) |
| 007 | DC2 | 172.16.11.11 | Windows Server 2019 Std | DC_Servers | SRV (111) |
| 008 | SRV-VEEAM-BACKUP | 172.16.44.70 | Windows Server | Backup | BACKUP (444) |
| 009 | OPNsense-FW2 | — | FreeBSD (bsd) | default | trunk |
| 013 | opnsense-fw1 | — | FreeBSD (bsd) | default | trunk |
| 014 | glpi-srv | 172.16.11.60 | Ubuntu 22.04 LTS | default | SRV (111) |

**OS Distribution**: Ubuntu (4), Windows (4), FreeBSD/bsd (2), Oracle Linux (1)

## Custom Rules — 35 HDS-Specific Detection Rules

### Rules by Category

| Category | Rule IDs | Count | Description |
|---|---|---|---|
| Authentication | 200000–200007 | 8 | SSH/AD/Authelia brute force, root login, off-hours access |
| Privilege Escalation | 200050–200054 | 5 | sudo, AD group changes, account creation/deletion |
| Anti-Forensics | 200100–200101 | 2 | Windows log clearing (T1070.001), Pass-the-Hash (T1550.002) |
| File Integrity (FIM) | 200150–200160 | 7 | Authelia/Nginx/Oracle config changes, ransomware extensions |
| Network / Firewall | 200200–200212 | 7 | Network scans, VLAN BACKUP intrusion, OpenVPN brute force |
| Tuning | 200250–200260 | 10 | Noise reduction: Zabbix AJAX, Oracle NTP, multicast, MGMT |
| GLPI / Web | 200300–200301 | 2 | HTTP 4xx/5xx errors on GLPI portal |
| Veeam Backup | 200350–200352 | 3 | Backup failures, Oracle backup critical, PostgreSQL tuning |
| Oracle Database | 200400–200402 | 3 | ORA- errors, ORA-00600 internal, login failures |
| Vulnerability | 200450–200451 | 2 | Critical/High CVE detection |
| Availability | 200500–200501 | 2 | Service stopped, system reboot/shutdown |

### Compliance Mapping

| Framework | Tags Used | Rules Count |
|---|---|---|
| PCI DSS | 10.2.4, 10.2.5, 10.6 | 12 rules |
| HIPAA | 164.312.a.1, 164.312.a.2, 164.312.b, 164.312.d | 15 rules |
| MITRE ATT&CK | T1070.001 (Log Clearing), T1550.002 (Pass-the-Hash) | 2 rules |

## Active Response — Automated Threat Containment

| Trigger Rule | Command | Target | Timeout | Threat |
|---|---|---|---|---|
| 5712, 5720 | `firewall-drop` | Local Linux agent | 180s | SSHD brute force (built-in) |
| 200000 | `firewall-drop` | Local Linux agent | 360s | HDS SSH brute force |
| 200007 | `firewall-drop` | SRV-PROXY | 300s | Authelia MFA brute force |
| 200200 | `pf` | OPNsense (FreeBSD) | 600s | Network scan detected |
| 200211 | `pf` | OPNsense (FreeBSD) | 1800s | OpenVPN brute force |
| 200202 | `pf` | OPNsense (FreeBSD) | 3600s | VLAN BACKUP intrusion attempt |
| 200002 | `netsh` | Windows DC | 180s | AD brute force (4625) |
| 200160 | `win_route-null` | Windows agent | 14400s | Ransomware extension detected |

### Active Response Evidence — Real Detections

**Detection 1 — Network scan from Kali (VLAN GUEST)**
```
Rule 200200 (level 10): "HDS: Scan réseau détecté - blocages multiples OPNsense"
Source: 172.16.55.150 (Kali in VLAN 555) → 172.16.11.11/12 (DC1/DC2 port 53)
Action: firewall-drop triggered → IP blocked for 600s
Evidence: Kali pentest attempting DNS resolution to internal DCs from GUEST zone
```

**Detection 2 — VLAN BACKUP unauthorized access**
```
Rule 200202 (level 12): "HDS: Tentative accès non autorisé VLAN-BACKUP"
Source: 172.16.44.70 (Veeam) → 52.123.242.189:443 (external)
Action: pf command triggered on OPNsense → blocked for 3600s
Evidence: Veeam server attempting blocked outbound connection
```

**Detection 3 — Web attacks on DMZ portal (pentest)**
```
Rule 31153 (level 10): "Multiple common web attacks from same source ip"
Rule 31154 (level 10): "Multiple XSS (Cross Site Scripting) attempts"
Source: 172.16.22.150 → SRV-PROXY (172.16.33.10)
Evidence: Path traversal + XSS injection attempts on Nginx/Authelia
Full log: GET /magento/magmi/web/ajax_pluginconf.php?file=../../etc/hosts
```

## Manager Configuration Highlights (ossec.conf)

| Setting | Value | Rationale |
|---|---|---|
| `logall` + `logall_json` | yes | Full logging for forensic analysis |
| `email_alert_level` | 10 | Only critical alerts trigger email |
| `email_maxperhour` | 60 | Tuned from 12 to reduce email flooding |
| Remote protocol | TCP 1514 | Reliable agent communication |
| Queue size | 131,072 | High-capacity event buffer |
| FIM frequency | 12h (43,200s) | Bi-daily integrity checks |
| Syscollector | 1h | Hourly hardware/software inventory |
| SCA interval | 12h | Bi-daily security configuration assessment |
| Vulnerability detection | 60m feed update | Hourly CVE feed refresh |
| Rootcheck | 12h | Bi-daily rootkit detection |
| Log sources | journald, active-responses.log, dpkg.log, opnsense.log | Multi-source collection |

## Shared Agent Configuration (agent.conf)

### Common Baseline (all agents)
- Syscollector: packages, OS, hardware, ports, processes (1h interval)
- Rootcheck: enabled (12h)
- SCA: enabled (12h)
- Active Response: enabled

### Windows-Specific
- FIM: `C:\Windows\System32\drivers\etc`, `C:\Users\Administrator` (realtime)
- Event channels: Security (filtered: ≠5145, ≠5156), System, Application

### Linux-Specific
- FIM: `/etc` (realtime), `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`
- Logs: `/var/log/auth.log`, `/var/log/syslog`

## Alert Statistics (last 24h — from dashboard)

| Severity | Count |
|---|---|
| Critical (level ≥ 15) | 61 |
| High (level 12-14) | 44 |
| Medium (level 7-11) | 19,632 |
| Low (level 0-6) | 122,369 |

**MITRE ATT&CK techniques detected**: Modify Registry, Valid Accounts, Vulnerability Scanning, Exploit Public-Facing Application, File and Directory Discovery, Process Injection, Data Destruction

## Screenshot Guide

| File (docs/assets/screenshots/wazuh/) | Content |
|---|---|
| `wazuh-01-dashboard-overview.png` | 11 agents active, severity breakdown |
| `wazuh-02-agent-status.png` | All 11 agents with OS, groups, IPs |
| `wazuh-03-threat-hunting.png` | MITRE ATT&CK categories, top 5 agents |
| `wazuh-04-local-rules.png` | Custom HDS rules in management UI |
| `wazuh-05-fim-events.png` | 2,308 FIM events (registry + file changes) |
| `wazuh-06-active-response.png` | XSS/path traversal detection detail |
| `wazuh-07-mailpit-alerts.png` | 383 alert emails via Mailpit SMTP |
