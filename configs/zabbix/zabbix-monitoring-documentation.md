# Zabbix Monitoring — Infrastructure Supervision

> Zabbix 7.4.9 — Ubuntu 22.04 LTS — PostgreSQL backend  
> Uptime: 5+ days | 19.67 values/second | 20,088 items | 7,728 triggers | 354 templates

## Monitored Hosts (14 — dual-stack on firewalls)

| Host | IP | Interface | Type | Tags | Items | Graphs | VLAN |
|---|---|---|---|---|---|---|---|
| DC1 | 172.16.11.11:10050 | ZBX agent | Windows Server | os, windows | 110 | 12 | SRV (111) |
| DC2 | 172.16.11.12:10050 | ZBX agent | Windows Server | os, windows | 110 | 12 | SRV (111) |
| GLPI-Srv | 172.16.11.60:10050 | ZBX agent | Linux | os, linux | 75 | 16 | SRV (111) |
| OPNSense-Firewall1 | 172.16.11.1:161 | **SNMP** | Network device | network, generic | 120 | 12 | trunk |
| OPNSense-Firewall2 | 172.16.11.2:161 | **SNMP** | Network device | network, generic | 120 | 12 | trunk |
| OPNSense-FW1 | 172.16.11.1:10050 | ZBX agent | FreeBSD | os, freebsd, firewall | 67 | 21 | trunk |
| OPNSense-FW2 | 172.16.11.2:10050 | ZBX agent | FreeBSD | os, freebsd, firewall | 67 | 21 | trunk |
| POSTE-ADMIN-IT | 172.16.99.115:10050 | ZBX agent | Windows 10 | os, windows | 109 | 12 | MGMT (999) |
| srv-oracle-db-01 | 172.16.11.20:10050 | ZBX agent | Oracle Linux | database, os, linux | 213 | 33 | SRV (111) |
| SRV-PROXY | 172.16.33.10:10050 | ZBX agent | Linux | os, linux | 75 | 16 | DMZ (333) |
| SRV-VEEAM-BACKUP | 172.16.44.70:10050 | ZBX agent | Windows Server | os, windows, backup | 158 | 18 | BACKUP (444) |
| srv-wazuh | 172.16.11.50:10050 | ZBX agent | Linux | os, linux | 75 | 16 | SRV (111) |
| srv-web-01 | 172.16.33.20:10050 | ZBX agent | Linux | software, linux | 89 | 19 | DMZ (333) |
| Zabbix server | 127.0.0.1:10050 | ZBX agent | Linux | software, linux | 153 | 16 | SRV (111) |

**Dual-stack monitoring on OPNsense**: Both firewalls are monitored via SNMP (port 161) for network-level metrics (interface throughput, packet counts) AND via Zabbix agent (port 10050) for OS-level metrics (CPU, memory, disk, processes). This provides complete observability across both the network and system layers.

## Host Availability Summary

| Status | Count |
|---|---|
| Available (green) | 5 |
| Unknown (grey) | 7 |
| Unavailable | 0 |
| **Total** | **14** (all enabled) |

## Server Diagnostics

```
Zabbix Server 7.4.9 (compiled Apr 8 2026)
Running on: Ubuntu 22.04 / OpenSSL 3.0.2
Uptime: 5 days (since May 12 2026)
Memory: 149.5 MB
CPU time: 4h 1min 56s
Tasks: 77 processes

Performance:
  Values per second: 19.67
  Active items: 20,088
  Active triggers: 7,728
  Templates: 354
  Elements: 1,486 active / 0 disabled / 55 unsupported

Cache:
  History cache: 16 MB (items: 21, values: 21)
  Value cache: 8 MB (items: 905, values: 5,621)
  Preprocessing: 893 cached items, 2M+ queued values

Pollers active:
  Agent poller: 26 values/cycle
  SNMP poller: 11 values/cycle
  History syncers: 4 workers
  Trappers: 5 workers
```

## Alert Integration — Mailpit SMTP

| Parameter | Value |
|---|---|
| Media type | Email-Mailpit (Generic SMTP) |
| SMTP server | 172.16.33.10 (SRV-PROXY, DMZ) |
| SMTP port | 1025 |
| From | zabbix-alertes@clinique-chatelet.local |
| To | alertes-infra@clinique-chatelet.local |
| Security | None (internal lab network) |
| Format | HTML |
| Total emails sent | 465+ (Zabbix + Wazuh combined) |

Alert types observed in Mailpit:
- `[Warning]` — Disk I/O, CPU, tablespace usage
- `[RESOLU]` — Service restored notifications
- `[RESOLU][DISASTER]` — Oracle connection unavailable → resolved
- `ALERTE DISPONIBILITE SANTE` — Custom HDS availability alert prefix

## Current Problems (at time of export)

| Time | Severity | Host | Problem |
|---|---|---|---|
| 15:56 | Warning | SRV-PROXY | Disk read/write responses too high (>20ms for 15m) |
| 15:50 | Warning | DC2 | Disk write responses too high (>0.02s for 15m) |
| 14:34 | Warning | POSTE-ADMIN-IT | Disk write responses too high (>0.02s for 15m) |
| 14:00 | Warning | DC1 | Disk write responses too high (>0.02s for 15m) |
| 10:35 | Warning | srv-web-01 | Disk read/write responses too high |
| 10:20 | Warning | Zabbix server | Disk read/write responses too high |
| 10:14 | Warning | GLPI-Srv | Disk read/write responses too high |

> All are disk I/O warnings — typical on a shared ESXi hypervisor with contention. No critical (Disaster/High) problems active.

## Oracle Database Monitoring (srv-oracle-db-01)

The Oracle host has **213 items** and **33 graphs** — the most instrumented host in the infrastructure. Custom dashboard tabs visible:
- **Oracle Performance** — dedicated tab for DB metrics
- **System Performance** — CPU load, memory, swap
- **Filesystems** — disk usage
- **Network interfaces** — throughput

Metrics captured (from CPU/Memory graph):
- System load: avg 0.27 (1m) / 0.39 (5m) / 0.42 (15m) — healthy
- CPU: 1.7% user, 2.4% system, 2.8% iowait — low utilization
- Memory: ~4.5 GB used of 8+ GB — 55% utilization (Oracle + OS)
- Swap: minimal usage

## Custom Templates Exported

| Template | Target | Description |
|---|---|---|
| FreeBSD by Zabbix agent | OPNsense FW1/FW2 | OS metrics for FreeBSD-based firewalls |

> Full template export: `custom-templates.yaml` (11,917 lines, Zabbix 7.4 format)

## Cross-VLAN Monitoring Architecture

```
                    ┌─────────────────┐
                    │  Zabbix Server   │
                    │ 172.16.11.40     │
                    │   VLAN 111       │
                    └────────┬─────────┘
                             │
         ┌───────────┬───────┴───────┬───────────┐
         ▼           ▼               ▼           ▼
    VLAN 111     VLAN 333       VLAN 444     VLAN 999
    (SRV)        (DMZ)          (BACKUP)     (MGMT)
    ─────────    ─────────      ─────────    ─────────
    DC1          SRV-PROXY      SRV-VEEAM   POSTE-ADMIN
    DC2          srv-web-01                  
    Oracle                                   
    Wazuh                                    
    GLPI                                     
    FW1 (agent)                              
    FW2 (agent)                              
    FW1 (SNMP)                               
    FW2 (SNMP)                               
```

Firewall rules allow Zabbix passive polling (10050) and active agent (10051) from MONITORING_SERVERS to all VLANs, plus SNMP (161) to the firewalls and ICMP ping to all hosts.

## Screenshot Guide

| File (docs/assets/zabbix/) | Content |
|---|---|
| `zabbix-01-dashboard.png` | Main dashboard: 14 hosts, 19.67 vps, current problems |
| `zabbix-02-host-monitoring.png` | All 14 hosts with interfaces, tags, item counts |
| `zabbix-03-oracle-cpu-memory.png` | Oracle DB performance graphs (CPU, load, memory, swap) |
| `zabbix-04-alert-problems.png` | Current problems timeline view |
| `zabbix-05-smtp-config.png` | Mailpit SMTP media type configuration |
| `zabbix-06-mailpit-alerts.png` | 465 alert emails (Zabbix + Wazuh) in Mailpit inbox |
