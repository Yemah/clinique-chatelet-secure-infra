# High Availability — CARP + pfSync Configuration

## Cluster Overview

| Parameter | FW1 (MASTER) | FW2 (BACKUP) |
|---|---|---|
| Hostname | opnsense-fw1.clinique-chatelet.local | OPNsense-FW2.clinique-chatelet.local |
| Version | OPNsense 26.1.3 (amd64) | OPNsense 26.1.3 (amd64) |
| OS | FreeBSD 14.3-RELEASE-p9 | FreeBSD 14.3-RELEASE-p9 |
| WAN IP | 192.168.65.11/24 | 192.168.65.99/24 |
| HA Sync IP | 10.0.0.1/30 | 10.0.0.2/30 |
| CARP Role | MASTER (advskew 0) | BACKUP (advskew 100) |
| Uptime (at export) | 5 days 19:42:27 | — |

## CARP Virtual IPs

| Interface | VHID | Virtual IP (Gateway) | FW1 Status | FW2 Status |
|---|---|---|---|---|
| VLAN_SRV | 11 | 172.16.11.254 | MASTER (freq 1/0) | BACKUP (freq 1/100) |
| VLAN_CLIENT | 22 | 172.16.22.254 | MASTER (freq 1/0) | BACKUP (freq 1/100) |
| VLAN_DMZ | 33 | 172.16.33.254 | MASTER (freq 1/0) | BACKUP (freq 1/100) |
| VLAN_BACKUP | 44 | 172.16.44.254 | MASTER (freq 1/0) | BACKUP (freq 1/100) |
| VLAN_GUEST | 55 | 172.16.55.254 | MASTER (freq 1/0) | BACKUP (freq 1/100) |
| VLAN_MGMT | 99 | 172.16.99.254 | MASTER (freq 1/0) | BACKUP (freq 1/100) |

> `freq 1/0` = advertise every 1 second with skew 0 (highest priority → MASTER)  
> `freq 1/100` = advertise every 1 second with skew 100 (lower priority → BACKUP)

## pfSync — State Synchronization

- **Protocol**: pfsync over dedicated vmx2 interface (no shared traffic)
- **Subnet**: 10.0.0.0/30 (FW1: .1, FW2: .2)
- **State table**: 1,166 active entries at time of export
- **Search rate**: 855.7/s
- **Insert rate**: 64.8/s
- **Zero errors**: No memory, congestion, checksum, or fragment errors

## pfctl Statistics (FW1)

```
Status: Enabled for 5 days 20:34:05
State Table:
  current entries        1166
  searches          433007791     855.7/s
  inserts            32806690      64.8/s
  removals           32856307      64.9/s
Counters:
  match              33243317      65.7/s
  bad-offset                0       0.0/s
  fragment                  0       0.0/s
  memory                    0       0.0/s
  state-mismatch          214       0.0/s
```

## Failover Behavior

When FW1 fails:
1. FW2 detects missing CARP advertisements (within ~3 seconds)
2. FW2 promotes all 6 VHIDs to MASTER
3. pfSync ensures session continuity (no TCP reset for established connections)
4. All VMs continue using the same gateway IPs (.254) — transparent failover

## Evidence

- `CARP-fw1_status.png` — FW1 showing all 6 VHIDs as MASTER
- `CARP-fw2_status.png` — FW2 showing all 6 VHIDs as BACKUP
- `sysctl net.inet.carp.allow: 1` — CARP enabled on both nodes
