# 🛡️ Pare-feu HA — OPNsense 26.1.3

Cluster actif/passif avec CARP (6 VHIDs) + pfSync sur lien dédié (10.0.0.0/30). Politique **Default Deny** sur chaque interface.

## Cluster

| | FW1 (MASTER) | FW2 (BACKUP) |
|---|---|---|
| WAN | 192.168.65.11 | 192.168.65.99 |
| HA Sync | 10.0.0.1 | 10.0.0.2 |
| advskew | 0 (priorité max) | 100 |

## Métriques pfctl

- **208 règles** en 19 sections logiques
- **35 alias** (hôtes, réseaux, ports)
- **1 166** états actifs simultanés
- **855,7** recherches/seconde
- **0** erreurs mémoire/fragmentation

## Contrôles de sécurité

| Contrôle | Implémentation |
|---|---|
| Default Deny | `block drop in log` sur chaque interface VLAN |
| Anti-spoofing | 11 règles (une par interface) |
| Anti-bogons WAN | RFC1918, RFC6598, loopback, bogon lists |
| SSH brute-force | Table `sshlockout` → block SSH + GUI |
| Virus protection | Table `virusprot` → block all |
| Fragment reassembly | `scrub in all fragment reassemble` |
| Stateful inspection | `flags S/SA keep state` sur toutes les pass rules |

## Fichiers de configuration

| Fichier | Description |
|---|---|
| [`fw1-rules-annotated.conf`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/opnsense/fw1-rules-annotated.conf) | 208 règles pfctl commentées |
| [`aliases-inventory.md`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/opnsense/aliases-inventory.md) | 35 alias détaillés |
| [`carp-ha-config.md`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/opnsense/carp-ha-config.md) | Configuration HA CARP + pfSync |
| [`fw1-interfaces.conf`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/opnsense/fw1-interfaces.conf) | Interfaces + VIPs |
| [`fw1-nat-rules.conf`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/opnsense/fw1-nat-rules.conf) | Règles NAT outbound |

![Dashboard FW1](screenshots/opnsense/dashboard-fw1.png)
