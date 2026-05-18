# Firewall Aliases Inventory — OPNsense Object-Group Architecture

> Extracted from OPNsense FW1 XML configuration backup  
> 35 aliases organized by category: Host, Network, Port

## Host Aliases (10)

| Alias | IP(s) | Description | VLAN |
|---|---|---|---|
| `AD_SERVERs` | 172.16.11.11, 172.16.11.12 | DC1 + DC2 (Active Directory, DNS, DHCP) | SRV (111) |
| `MONITORING_SERVERS` | 172.16.11.40, 172.16.11.50 | Zabbix + Wazuh (supervision + SIEM) | SRV (111) |
| `HOST_ORACLE` | 172.16.11.20 | Oracle XE 21c — HDS patient database | SRV (111) |
| `HOST_GLPI` | 172.16.11.60 | GLPI 10 — IT Service Management | SRV (111) |
| `HOST_PROXY` | 172.16.33.10 | Nginx reverse proxy + Authelia MFA | DMZ (333) |
| `HOST_WEBAPP` | 172.16.33.20 | Node.js medical portal (port 8080) | DMZ (333) |
| `MAILPIT_SERVER` | 172.16.33.10 | Mailpit test SMTP relay (co-hosted with proxy) | DMZ (333) |
| `HOST_VEEAM` | 172.16.44.70 | Veeam Backup & Replication 13 | BACKUP (444) |
| `PC_ADMIN_IT` | 172.16.99.115 | Hardened admin workstation (BitLocker + 9 GPOs) | MGMT (999) |
| `VEEAM_BACKUP_TARGETs` | DC1, DC2, Oracle, Zabbix, Wazuh, Proxy, WebApp, ESXi | All VMs protected by Veeam PRA | Multi-VLAN |
| `SERVEURS_MAJ_UBUNTU` | archive.ubuntu.com, security.ubuntu.com, ports.ubuntu.com | Official Ubuntu update mirrors only | N/A (FQDN) |

## Network Aliases (7)

| Alias | Subnet | Description |
|---|---|---|
| `NET_SRV` | 172.16.11.0/24 | VLAN 111 — Server zone |
| `NET_CLIENT` | 172.16.22.0/24 | VLAN 222 — Workstation zone |
| `NET_DMZ` | 172.16.33.0/24 | VLAN 333 — Demilitarized zone |
| `NET_BACKUP` | 172.16.44.0/24 | VLAN 444 — Backup zone |
| `NET_GUEST` | 172.16.55.0/24 | VLAN 555 — Patient/guest zone |
| `NET_TAILSCALE` | 100.64.0.0/10 | Tailscale overlay for OOB management |
| `PRIVATE_NETWORK` | 172.16.11.0/24, 172.16.22.0/24, 172.16.33.0/24, 172.16.44.0/24, 172.16.55.0/24, 10.0.0.0/30 | All internal subnets (used in block rules) |

## Port Aliases (18)

| Alias | Port(s) | Protocol | Description |
|---|---|---|---|
| `WEB_PORTS` | 80, 443 | TCP | HTTP + HTTPS |
| `PORT_ORACLE` | 1521 | TCP | Oracle TNS listener |
| `PORT_NODEJS` | 8080 | TCP | Node.js medical application |
| `PORT_LDAPs` | 636 | TCP | LDAP over SSL (Authelia → AD) |
| `PORT_NTP` | 123 | UDP | Network Time Protocol |
| `PORT_OPENVPN` | 1194 | UDP | OpenVPN tunnel |
| `PORT_FW_GUI` | 4443 | TCP | OPNsense web management |
| `PORT_MAILPIT_SMTP` | 1025, 8025 | TCP | Mailpit SMTP + Web UI |
| `WAZUH_AGENTs_PORT` | 1514, 1515 | TCP | Wazuh event + enrollment |
| `PORT_ZABBIX_AGENTs_Passif` | 10050 | TCP | Zabbix passive agent |
| `PORT_ZABBIX_AGENTs_Actifs` | 10051 | TCP | Zabbix active agent |
| `PORT_SNMP__SYSLOG` | 161, 514 | UDP | SNMP polling + Syslog to Wazuh |
| `AD_AUTH_PORTs` | 88, 636, 3268, 3269 | TCP/UDP | Kerberos + LDAPS + Global Catalog |
| `AD_REPL_PORTs` | 88, 135, 389, 445, 464, 636, 3268, 3269, 49152:65535 | TCP/UDP | Full AD replication + auth |
| `ADMIN_PORTs` | 22, 443, 4443, 8025, 9200 | TCP | Admin tools (SSH, HTTPS, FW GUI, Mailpit, Wazuh Indexer) |
| `VEEAM_PORTs` | 135, 445, 902, 2500:3300, 6160:6161, 9392 | TCP | Veeam backup operations (RPC, SMB, VDDK, agents) |

## Authentication Architecture (from auth servers config)

| Auth Server | Type | Backend | Port | Usage |
|---|---|---|---|---|
| `MFA_VPN` | LDAP + TOTP | DC1 via LDAPS | 636 | OpenVPN authentication (dual-factor) |
| `MFA_AD` | LDAP | DC2 via LDAPS | 636 | AD-only authentication (fallback) |
| `TEST-LDAPS-DC` | LDAP | DC1 via LDAPS | 636 | LDAPS connectivity testing |
| `MFA_Local` | Local TOTP | OPNsense local | — | Local MFA (emergency access) |

> All LDAP connections use SSL encryption (port 636) — no plaintext LDAP (389)

## User Accounts (6 — all MFA-enabled)

| Username | Role | OTP Seed | Description |
|---|---|---|---|
| `root` | System admin | TOTP configured | System Administrator |
| `manager` | HA sync account | TOTP configured | HA synchronization between FW1 ↔ FW2 |
| `the.ghost` | IT admin | TOTP configured | Admin IT — Domain Admins member |
| `j.martin` | Clinical user | TOTP configured | Soignant (nurse — test user) |
| `m.leclerc` | Clinical user | TOTP configured | Soignant (nurse — test user) |
| `Administrateur` | Admin | TOTP configured | Administrative account |

> OTP seeds are NOT stored in this repository — see security note below

## HA Sync Configuration

| Parameter | Value |
|---|---|
| Sync interface | vmx2 (HA_SYNC) |
| Peer IP | 10.0.0.2 (FW2) |
| pfSync version | 1400 |
| Preemption | Disabled |
| Synced items | Aliases, Auth servers, Certs, DHCP relay, Rules, NAT, OpenVPN, Users, Virtual IPs |

## AD Organizational Unit Structure (from LDAP auth containers)

```
DC=clinique-chatelet,DC=local
├── OU=CLINIQUE_USERS
│   ├── OU=Soignants
│   │   ├── OU=Aides-Soignants
│   │   ├── OU=IDE (Infirmiers)
│   │   └── OU=Medecins
│   ├── OU=Administratifs
│   │   └── OU=Secretariat
│   └── OU=IT
├── OU=CLINIQUE_GROUPS
│   └── OU=Security_Groups
├── OU=CLINIQUE_COMPUTERS
│   └── OU=Servers
└── OU=Domain Controllers
```