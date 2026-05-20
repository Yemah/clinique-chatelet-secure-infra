# 📊 Matrice de Flux Inter-VLAN

> Extraite de `pfctl -sr` sur OPNsense FW1 (MASTER) — 208 règles, Default Deny

## Résumé des flux

| Source ↓ \ Dest → | SRV | CLIENT | DMZ | BACKUP | GUEST | MGMT | Internet |
|---|---|---|---|---|---|---|---|
| **SRV** | ✅ | ❌ | Zabbix | Zabbix | ❌ | ❌ | NTP, HTTP/S, SMTP |
| **CLIENT** | ❌ | ✅ | HTTP/S portail | ❌ | ❌ | ❌ | HTTP/S (MAJ) |
| **DMZ** | LDAPS, DNS, Oracle | ❌ | ✅ | ❌ | ❌ | ❌ | HTTP/S (MAJ) |
| **BACKUP** | Veeam ports, DNS | ❌ | ❌ | ✅ | ❌ | ❌ | HTTP/S (MAJ) |
| **GUEST** | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | DNS + HTTP/S |
| **MGMT** | ✅ full | ✅ | ✅ | ✅ | ✅ | ✅ | FW GUI, Mailpit |

- ✅ = Autorisé (ports spécifiques sauf MGMT)
- ❌ = Bloqué (default deny ou block explicite)

➡️ [Matrice complète détaillée (Markdown)](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/docs/assets/diagrams/flux-matrix.md)

## Principe de segmentation

Chaque VLAN se termine par un **deny all** explicite. Les flux inter-VLAN sont autorisés **uniquement** pour les besoins opérationnels documentés :

- **GUEST** ne voit rien sauf Internet (DNS public + HTTP/S)
- **BACKUP** est isolé de CLIENT et DMZ (blocs explicites loggés)
- **DMZ** n'atteint le réseau interne que pour Oracle (1521) et AD/LDAPS (636)
- **MGMT** a un accès complet (bastion admin) mais un deny final attrape tout trafic non-MGMT
- **CLIENT** accède au portail via la DMZ mais pas directement aux serveurs

## WAN Inbound (3 règles seulement)

| Port | Destination | Fonction |
|---|---|---|
| 443 (HTTPS) | Reverse Proxy | Portail Authelia → application médicale |
| 80 (HTTP) | WAN address | Redirect 301 → HTTPS |
| 1194 (UDP) | WAN address | OpenVPN soignants distants (MFA requis) |
