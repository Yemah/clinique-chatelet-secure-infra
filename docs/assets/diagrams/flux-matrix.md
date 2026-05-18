# Firewall Flux Matrix — Clinique Le Châtelet

> Extracted from `pfctl -sr` on OPNsense FW1 (MASTER) — OPNsense 26.1.3  
> Policy: **Default Deny** (block drop in log) on all interfaces  
> Anti-spoofing: per-interface source validation | Anti-bogons: RFC1918 + bogons on WAN

## Inter-VLAN Traffic Matrix

| Source ↓ \ Dest → | VLAN 111 SRV | VLAN 222 CLIENT | VLAN 333 DMZ | VLAN 444 BACKUP | VLAN 555 GUEST | VLAN 999 MGMT | WAN / Internet |
|---|---|---|---|---|---|---|---|
| **VLAN 111 SRV** | ✅ Internal | ❌ Blocked | Zabbix polling (10050/10051), ICMP | Zabbix polling (10050/10051), ICMP | ❌ Blocked | ❌ Blocked | NTP, HTTP/S (updates), DNS, SMTP→Mailpit |
| **VLAN 222 CLIENT** | ❌ **Explicit block** | ✅ Internal | HTTP/S→Proxy+WebApp | ❌ **Explicit block** | ❌ Blocked | ❌ Blocked | HTTP/S (Windows Update), Wazuh (1514/1515), Zabbix (10051) |
| **VLAN 333 DMZ** | LDAPS→AD (636), DNS (53), Oracle (1521) | ❌ Blocked | ✅ Internal (Proxy→WebApp:8080) | ❌ Blocked | ❌ Blocked | ❌ Blocked | HTTP/S (updates), NTP→AD, Wazuh+Zabbix→SRV |
| **VLAN 444 BACKUP** | Veeam ports (135, 445, 902, 2500-3300, 6160-6161, 9392), DNS, NTP | ❌ **Explicit block** | ❌ **Explicit block** | ✅ Internal | ❌ Blocked | ❌ Blocked | HTTP/S (updates), Wazuh+Zabbix→SRV |
| **VLAN 555 GUEST** | ❌ **Explicit block (PRIVATE_NETWORK)** | ❌ Blocked | ❌ Blocked | ❌ Blocked | ✅ Internal | ❌ Blocked | DNS (53), HTTP/S only |
| **VLAN 999 MGMT** | ✅ **Full access PRIVATE_NETWORK** | ✅ Via PRIVATE_NETWORK | ✅ Via PRIVATE_NETWORK | ✅ Via PRIVATE_NETWORK | ✅ Via PRIVATE_NETWORK | ✅ Internal | FW GUI (4443), Mailpit (1025/8025) |
| **OpenVPN (10.100.100.0/24)** | ✅ Full access | ✅ Full access | ✅ Full access | ✅ Full access | ✅ Full access | ✅ Full access | Via NAT |
| **Tailscale** | ✅ Full access | ✅ Full access | ✅ Full access | ✅ Full access | ✅ Full access | ✅ Full access | Via NAT |

### Legend

- ✅ = Traffic allowed (with specific port restrictions unless noted)
- ❌ = Traffic blocked (default deny or explicit block rule)
- **Explicit block** = Dedicated `block drop in log` rule before the catch-all deny

## WAN Inbound Rules (3 rules)

| # | Protocol | Destination | Port | Description |
|---|---|---|---|---|
| 1 | TCP | HOST_PROXY (reverse proxy) | 443 (HTTPS) | HDS — HTTPS entrant → Portail Authelia |
| 2 | TCP | WAN address | 80/443 | HDS — HTTP redirect 301 → HTTPS |
| 3 | TCP/UDP | WAN address | 1194 (OpenVPN) | VPN soignants distants — MFA requis |

## Monitoring & SIEM Flows (cross-VLAN)

| Flow | Source | Destination | Ports | Direction |
|---|---|---|---|---|
| Zabbix passive polling | MONITORING_SERVERS (SRV) | All VLANs | 10050 (TCP) | SRV → target |
| Zabbix active agent | All VLANs | MONITORING_SERVERS | 10051 (TCP) | target → SRV |
| Zabbix SNMP polling | MONITORING_SERVERS | OPNsense FW (self) | 161 (UDP) | SRV → FW |
| Zabbix ICMP | MONITORING_SERVERS | SRV, DMZ, CLIENT, BACKUP, FW | ICMP | SRV → target |
| Wazuh agent → manager | All VLANs | MONITORING_SERVERS | 1514, 1515 (TCP) | target → SRV |
| Syslog → Wazuh | MONITORING_SERVERS | Wazuh | 514 (UDP) | SRV internal |
| Mailpit SMTP | Zabbix, GLPI, Admin | MAILPIT_SERVER (DMZ) | 1025 (SMTP), 8025 (Web) | SRV/MGMT → DMZ |

## Security Controls Summary

| Control | Implementation | Evidence |
|---|---|---|
| Default Deny | `block drop in log quick ... all` on every VLAN interface | Last rule on each interface |
| Anti-spoofing | `block drop in log on ! vlanX.YYY inet from subnet to any` | 11 anti-spoof rules (one per interface) |
| Anti-bogons WAN | Block RFC1918 (10/8, 172.16/12, 192.168/16), RFC6598 (100.64/10), link-local, loopback | 6 explicit block rules on vmx0 |
| Bogon lists | `<bogons>` and `<bogonsv6>` tables on WAN | Auto-updated by OPNsense |
| SSH brute-force protection | `<sshlockout>` table → block SSH (22) and GUI (4443) | pfctl table + block rules |
| Virus protection | `<virusprot>` table → block all traffic | Block rule with drop |
| Port 0 protection | Block TCP/UDP from/to port 0 (IPv4 + IPv6) | 8 explicit block rules |
| Fragment reassembly | `scrub in all fragment reassemble` | First rule in ruleset |
| CARP HA | 6 CARP VHIDs (11, 22, 33, 44, 55, 99) — FW1 MASTER, FW2 BACKUP | VIP status screenshots |
| pfSync state sync | 10.0.0.0/30 on dedicated vmx2 interface | Interface config + states |
| Outbound NAT | Per-VLAN NAT with ISAKMP static-port preservation | NAT rules export |
| Stateful inspection | `flags S/SA keep state` on all pass rules | pfctl -sr output |

## Aliases Used (Object-Group Architecture)

| Alias | Type | Purpose |
|---|---|---|
| AD_SERVERs | Host group | DC1 + DC2 (DNS, LDAP, Kerberos, NTP) |
| MONITORING_SERVERS | Host group | Zabbix + Wazuh servers |
| HOST_ORACLE | Host | Oracle DB server (port 1521) |
| HOST_PROXY | Host | Nginx reverse proxy + Authelia |
| HOST_WEBAPP | Host | Web portal (port 8080) |
| HOST_VEEAM | Host | Veeam B&R server |
| HOST_GLPI | Host | GLPI ITSM server |
| MAILPIT_SERVER | Host | Mailpit test SMTP relay |
| PC_ADMIN_IT | Host | Hardened admin workstation |
| VEEAM_BACKUP_TARGETs | Host group | VMs backed up by Veeam |
| NET_SRV / NET_CLIENT / NET_DMZ / NET_BACKUP / NET_GUEST | Network | VLAN subnets |
| PRIVATE_NETWORK | Network group | All internal RFC1918 subnets |
| WAZUH_AGENTs_PORT | Port group | 1514, 1515 |
| PORT_ZABBIX_AGENTs_Passif / Actifs | Port group | 10050, 10051 |
| PORT_SNMP__SYSLOG | Port group | 161, 514 |
| WEB_PORTS | Port group | 80, 443 |
| PORT_FW_GUI | Port | 4443 |
| PORT_MAILPIT_SMTP | Port | 1025 |
