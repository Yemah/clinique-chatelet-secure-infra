# 🏥 Infrastructure Sécurisée — Clinique Le Châtelet

Bienvenue sur la documentation technique du **Projet Résilience** — une infrastructure de soins sécurisée conçue pour protéger des données de santé (HDS/HIPAA).

---

## Le Projet en Bref

| | |
|---|---|
| **Objectif** | Exposer une application médicale Oracle sur le Web avec MFA, en respectant les exigences HDS |
| **Infrastructure** | 15 VMs sur VMware ESXi 7.0 U2, segmentées en 6 VLANs |
| **Sécurité** | Cluster HA OPNsense, SIEM Wazuh (35 règles custom), Zabbix (14 hôtes), MFA Authelia |
| **Conformité** | HDS, ISO 27001, RGPD Art. 32, PCI DSS, HIPAA |

## Chiffres Clés

| Composant | Valeur | Composant | Valeur |
|---|---|---|---|
| Règles firewall | **208** | Agents SIEM | **12** |
| Règles détection HDS | **35** | Items monitoring | **20 088** |
| Alertes collectées | **92 488** | Active Responses | **8** |
| GPOs sécurité | **10** | VMs protégées | **15** |

## Navigation Rapide

<div class="grid cards" markdown>

- :material-shield-lock: **[Pare-feu HA](security/firewall.md)** — 208 règles pfctl, CARP, anti-spoofing
- :material-eye: **[SIEM Wazuh](security/wazuh.md)** — 35 règles HDS, Active Response automatisé
- :material-chart-line: **[Zabbix](security/zabbix.md)** — 14 hôtes, dual-stack, 19,67 vps
- :material-server: **[Active Directory](infra/active-directory.md)** — 2 DCs, 10 GPOs, LDAPS
- :material-lock: **[DMZ & Authelia](infra/dmz-authelia.md)** — MFA TOTP, Nginx forward-auth
- :material-database: **[Oracle & Portail](infra/oracle-portal.md)** — Oracle 21c, portail Node.js
- :material-backup-restore: **[Backup & PRA](infra/veeam-pra.md)** — Veeam, 300 Go, RPO < 24h
- :material-bug: **[Pentest](pentest/results.md)** — Kali GUEST → 0 hôte visible

</div>

## Architecture

```
                    ┌───────────┴───────────┐
                    │   Cluster HA OPNsense  │
                    │   FW1 ◄──pfSync──► FW2 │
                    └───────────┬───────────┘
                        Trunk 802.1Q (VGT 4095)
                                │
     ┌─────────┬────────────┬───┴───┬────────────┬──────────┐
     │         │            │       │            │          │
  VLAN 111  VLAN 222    VLAN 333  VLAN 444   VLAN 555   VLAN 999
    SRV      CLIENT       DMZ     BACKUP      GUEST      MGMT
```

➡️ [Architecture détaillée](architecture/overview.md) · [Matrice de flux](architecture/flux-matrix.md)
