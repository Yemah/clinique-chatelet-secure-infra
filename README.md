# 🏥 Clinique Le Châtelet — Infrastructure Sécurisée de Soins

**Infrastructure réseau de niveau production pour la protection de données de santé (HDS/HIPAA) — 13 VMs, 6 VLANs, cluster HA, SIEM, MFA, PRA**

![OPNsense](https://img.shields.io/badge/Firewall-OPNsense_26.1.3-D94F00?style=for-the-badge&logo=opnsense&logoColor=white)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.14.4-3CBCB4?style=for-the-badge&logo=wazuh&logoColor=white)
![Zabbix](https://img.shields.io/badge/Monitoring-Zabbix_7.4.9-D40000?style=for-the-badge&logo=zabbix&logoColor=white)
![VMware](https://img.shields.io/badge/Hyperviseur-ESXi_7.0_U2-607078?style=for-the-badge&logo=vmware&logoColor=white)
![Windows Server](https://img.shields.io/badge/AD-Windows_Server_2022-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![Oracle](https://img.shields.io/badge/BDD-Oracle_21c-F80000?style=for-the-badge&logo=oracle&logoColor=white)

![HDS](https://img.shields.io/badge/Conformité-HDS-00A86B?style=flat-square)
![ISO 27001](https://img.shields.io/badge/Conformité-ISO_27001-0066CC?style=flat-square)
![RGPD](https://img.shields.io/badge/Conformité-RGPD_Art.32-6C3483?style=flat-square)
![PCI DSS](https://img.shields.io/badge/Conformité-PCI_DSS-1A1A2E?style=flat-square)
![HIPAA](https://img.shields.io/badge/Conformité-HIPAA-2196F3?style=flat-square)

*Projet de Master — Infrastructure Réseaux et Sécurité · Conçu, déployé et pentesté from scratch sur ESXi physique*

---

## 🎯 Contexte

Une clinique de rééducation devait exposer son **application médicale Oracle** sur le Web pour l'accès distant des soignants, en respectant les exigences **HDS** et en garantissant **99,99% de disponibilité**.

**Avant** : Aucun pare-feu, aucune segmentation, postes avec droits admin, DNS vers la box, Oracle exposé sur le LAN.

**Après** : Défense en profondeur sur 6 couches — segmentation VLAN, cluster HA CARP, MFA TOTP obligatoire, SIEM avec 35 règles custom, Active Response automatisé, PRA avec backup CBT, et pentest validant le tout.

---

## 🏗️ Architecture

```
                         ┌──────────────┐
                         │   INTERNET   │
                         └──────┬───────┘
                                │
                    ┌───────────┴───────────┐
                    │   Cluster HA OPNsense  │
                    │   FW1 ◄──pfSync──► FW2 │
                    │   6 VIPs CARP (.254)   │
                    └───────────┬───────────┘
                        Trunk 802.1Q (VGT 4095)
                                │
     ┌─────────┬────────────┬───┴───┬────────────┬──────────┐
     │         │            │       │            │          │
┌────┴─────┐┌──┴─────┐┌────┴───┐┌──┴───────┐┌───┴────┐┌───┴─────┐
│ VLAN 111 ││VLAN 222││VLAN 333││ VLAN 444 ││VLAN 555││VLAN 999 │
│   SRV    ││ CLIENT ││  DMZ   ││  BACKUP  ││ GUEST  ││  MGMT   │
│──────────││────────││────────││──────────││────────││─────────│
│DC1+DC2   ││Postes  ││Nginx   ││Veeam B&R ││Internet││Bastion  │
│Oracle 21c││travail ││+MFA    ││300Go repo││  seul  ││BitLocker│
│Zabbix    ││soignant││Portail ││15 VMs    ││        ││9 GPOs   │
│Wazuh     ││        ││Mailpit ││protégées ││        ││         │
│GLPI      ││        ││        ││          ││        ││         │
└──────────┘└────────┘└────────┘└──────────┘└────────┘└─────────┘
172.16.11.0 172.16.22.0 172.16.33.0 172.16.44.0 172.16.55.0 172.16.99.0
```

➡️ [Matrice de flux inter-VLAN complète](docs/assets/diagrams/flux-matrix.md)

---

## 📊 Chiffres Clés

| | Valeur | | Valeur |
|---|---|---|---|
| Règles firewall pfctl | **208** | VMs sur ESXi | **15** |
| Alias (object-groups) | **35** | VLANs isolés | **6** |
| Règles SIEM custom HDS | **35** | Agents Wazuh | **12** |
| Items Zabbix actifs | **20 088** | Triggers Zabbix | **7 728** |
| Alertes collectées | **92 488** | GPOs de sécurité | **10** |
| Active Responses auto | **8** | Comptes MFA (TOTP) | **6** |
| Backup Oracle (CBT) | **80 Go / 16 min** | Repository backup | **300 Go** |

---

## 🛡️ Sécurité Réseau — Pare-feu HA

Cluster actif/passif **OPNsense 26.1.3** avec CARP (6 VHIDs) + pfSync sur lien dédié. Politique **Default Deny** sur chaque interface, anti-spoofing sur 11 interfaces, anti-bogons RFC1918/6598 sur le WAN, tables dynamiques `sshlockout` et `virusprot`.

| Documentation | Lien |
|---|---|
| Règles pfctl annotées (208 règles, 19 sections) | [fw1-rules-annotated.conf](configs/opnsense/fw1-rules-annotated.conf) |
| Matrice de flux inter-VLAN | [flux-matrix.md](docs/assets/diagrams/flux-matrix.md) |
| Inventaire des 35 alias | [aliases-inventory.md](configs/opnsense/aliases-inventory.md) |
| Configuration HA CARP + pfSync | [carp-ha-config.md](configs/opnsense/carp-ha-config.md) |
| Interfaces et VIPs | [fw1-interfaces.conf](configs/opnsense/fw1-interfaces.conf) |
| Règles NAT outbound | [fw1-nat-rules.conf](configs/opnsense/fw1-nat-rules.conf) |

---

## 🔍 SIEM & Détection — Wazuh XDR

**35 règles de détection personnalisées** mappées sur PCI DSS, HIPAA et MITRE ATT&CK. **8 Active Responses** bloquent les menaces en temps réel via `firewall-drop` (Linux), `pf` (FreeBSD/OPNsense) et `netsh` (Windows).

| Catégorie | Règles | Détections |
|---|---|---|
| Authentification | 200000–200007 | Brute force SSH/AD, root login, accès hors heures, brute force Authelia |
| Élévation de privilèges | 200050–200054 | sudo, modification groupes AD, cycle de vie comptes |
| Anti-forensics | 200100–200101 | Effacement logs (T1070.001), Pass-the-Hash (T1550.002) |
| Intégrité fichiers | 200150–200160 | Tampering configs, **détection extensions ransomware** |
| Réseau | 200200–200212 | Scan réseau, intrusion VLAN BACKUP, brute force VPN |
| Backup / Oracle / Dispo | 200350–200501 | Échecs backup, ORA-00600, service arrêté, reboot |

| Documentation | Lien |
|---|---|
| Règles custom XML (35 règles) | [local_rules.xml](configs/wazuh/local_rules.xml) |
| Documentation SIEM complète | [wazuh-siem-documentation.md](configs/wazuh/wazuh-siem-documentation.md) |
| Configuration manager (ossec.conf) | [ossec.conf](configs/wazuh/ossec.conf) |
| Configuration agents partagée | [agent-conf-default.xml](configs/wazuh/agent-conf-default.xml) |

---

## 📈 Supervision — Zabbix 7.4.9

**14 hôtes** supervisés sur tous les VLANs avec observation **dual-stack** sur les firewalls (agent Zabbix + SNMP). Oracle DB instrumenté avec **213 items** et dashboards de performance dédiés. **19,67 valeurs/seconde** de débit. Alertes email via Mailpit (465+ notifications).

| Documentation | Lien |
|---|---|
| Documentation monitoring (14 hôtes) | [zabbix-monitoring-documentation.md](configs/zabbix/zabbix-monitoring-documentation.md) |
| Templates exportés (Zabbix 7.4) | [custom-templates.yaml](configs/zabbix/custom-templates.yaml) |

---

## 🏢 Active Directory — Infrastructure Tier 0

Domaine `clinique-chatelet.local` en mode **Windows2016Domain** avec 2 contrôleurs (DC1 + DC2), réplication **0 erreur**, 5 rôles FSMO sur DC1, **LDAPS actif** (port 636). **17 OUs** structurées par rôle métier, **4 comptes de service** dédiés (svc_backup, svc_monitoring, svc_glpi, svc_authelia), et **10 GPOs de sécurité**.

| GPO | Fonction |
|---|---|
| GPO-Security-Password-Policy | Min 12 caractères, complexité, historique 24 |
| GPO-Security-Account-Lockout | 5 tentatives → verrouillage 30 min |
| GPO-Security-Audit-Policy | 10 catégories d'audit (Succès + Échec) |
| GPO-Security-Disable-SMBv1 | Protection WannaCry/NotPetya |
| GPO-Security-Windows-Firewall | Pare-feu Windows sur tous les profils |
| GPO-BitLocker-NoTPM | Chiffrement disque sans TPM, récupération AD DS |
| GPO-Workstations-Hardening | Durcissement global postes de travail |

```
DC=clinique-chatelet,DC=local
├── OU=CLINIQUE_USERS
│   ├── OU=Medecins
│   ├── OU=IT
│   ├── OU=Soignants (IDE, Aides-Soignants)
│   └── OU=Administratifs (Secretariat, Direction)
├── OU=CLINIQUE_COMPUTERS (Servers, Workstations)
├── OU=CLINIQUE_GROUPS (Security_Groups, Distribution_Groups)
└── OU=CLINIQUE_SERVICE_ACCOUNTS
```

| Documentation | Lien |
|---|---|
| Documentation AD complète | [ad-infrastructure-documentation.md](configs/gpo/ad-infrastructure-documentation.md) |
| Liste des GPOs | [gpo-list.txt](configs/gpo/gpo-list.txt) |
| Structure des OUs | [ou-structure.txt](configs/gpo/ou-structure.txt) |
| Rôles FSMO | [fsmo-roles.txt](configs/gpo/fsmo-roles.txt) |
| Scopes DHCP (6 VLANs) | [dhcp-scopes.txt](configs/gpo/dhcp-scopes.txt) |
| Zones DNS | [dns-zones.txt](configs/gpo/dns-zones.txt) |
| Réplication (0 erreur) | [replication-summary.txt](configs/gpo/replication-summary.txt) |

---

## 🌐 Architecture Applicative — Portail Médical HDS

Chaîne complète de bout en bout : **Navigateur → HTTPS → Nginx → Authelia (LDAPS + TOTP) → Node.js → Oracle 21c**

```
Soignant ──► Nginx (443/SSL) ──► Authelia (TOTP) ──► SRV-WEB (8080) ──► Oracle (1521)
                                      │                                       │
                                      ▼                                       ▼
                                 DC1 LDAPS:636                        CLINIQUE_DATA
                                 (svc_authelia)                       (1 Go, patients)
```

**Authelia v4.39.15** : Policy `deny` par défaut, `two_factor` obligatoire sur le portail. Backend LDAPS vers AD avec TLS 1.2 minimum. Sessions SQLite, cookies 8h, inactivité 1h.

**Oracle 21c XE** : Instance XE, tablespace dédié `CLINIQUE_DATA` (1 Go), `audit_trail=DB`, 2 comptes ouverts seulement (CLINIQUE_APP + C##ZBX_MONITOR), 16 comptes Oracle par défaut verrouillés.

**Nginx** : Reverse proxy SSL/HTTP2 avec forward-auth Authelia. Headers `Remote-User`, `Remote-Groups`, `Remote-Name` transmis au backend. Redirect HTTP→HTTPS. Certificat auto-signé RSA 2048.

| Documentation | Lien |
|---|---|
| Documentation DMZ complète | [dmz-documentation.md](configs/nginx/dmz-documentation.md) |
| Vhost Nginx portail | [portal-clinique-chatelet.conf](configs/nginx/portal-clinique-chatelet.conf) |
| Snippet forward-auth | [authelia-location.conf](configs/nginx/authelia-location.conf) |
| Configuration Authelia | [configuration.yml](configs/authelia/configuration.yml) |
| Certificat TLS | [tls-cert-info.txt](configs/nginx/tls-cert-info.txt) |
| Documentation Oracle | [oracle-application-documentation.md](configs/oracle/oracle-application-documentation.md) |
| Audit Oracle (instance, tablespaces) | [oracle-audit.txt](configs/oracle/oracle-audit.txt) |
| Listener Oracle | [oracle-listener.txt](configs/oracle/oracle-listener.txt) |

---

## 💾 Sauvegarde & PRA — Veeam B&R 13

VLAN 444 sanctuarisé. Repository dédié **300 Go** (`E:\VeeamBackup`). **3 jobs planifiés** dont JOB-BACKUP-ORACLE-HDS (80 Go, CBT, 16 min, Success). **15 VMs inventoriées**, toutes "Clean" au scan malware Veeam. Détection Wazuh rule 200202 sur toute tentative d'accès non autorisé au VLAN BACKUP.

| Métrique PRA | Objectif | Réalisé |
|---|---|---|
| RPO (perte de données max) | < 24h | ✅ Backup quotidien Oracle |
| RTO (temps de reprise) | < 4h | ✅ Restauration VM ~20 min |
| Compression | > 3x | ✅ 3,5x mesuré |

| Scénario de panne | Reprise |
|---|---|
| Perte Oracle DB | Restauration VM depuis JOB-BACKUP-ORACLE-HDS (~20 min) |
| Perte DC1 (FSMO) | DC2 prend le relais + seizure FSMO (~30 min) |
| Perte pare-feu FW1 | CARP automatique → FW2 promu MASTER (~3 sec) |
| Ransomware | Isolation VLAN + restauration VMs clean + scan malware Veeam |

➡️ [Documentation Veeam/PRA complète](configs/veeam/veeam-pra-documentation.md)

---

## 🔬 Résultats du Pentest

Audit offensif depuis **Kali Linux** en VLAN GUEST (555) — zéro connaissance du SI.

| Test | Résultat |
|---|---|
| Accès VLANs internes depuis GUEST | ❌ 0 hôte visible |
| Scan 65 535 ports sur la gateway | ❌ Tous filtrés |
| DNS vers DCs internes | ❌ Bloqué |
| VLAN hopping 802.1Q double-tag | ❌ Bloqué |
| DNS tunneling Base64 | ❌ Timeout |
| XSS / Path traversal sur DMZ | 🔔 **Détecté par Wazuh** (rule 31153 + Active Response) |
| Scan réseau Kali → interne | 🔔 **Détecté et Bloqué** (rule 200200 → firewall-drop) |

---

## 📋 Conformité

| Référentiel | Contrôles implémentés |
|---|---|
| **HDS** | Segmentation 6 VLANs, LDAPS, MFA TOTP, audit DB Oracle, backup sanctuarisé, PRA |
| **ISO 27001** | SMSI, analyse risques, politique Default Deny, journalisation centralisée, réponse incident |
| **RGPD Art. 32** | Chiffrement (BitLocker, TLS, LDAPS), pseudonymisation, tests de résilience (pentest) |
| **PCI DSS** | Firewall segmenté, deny par défaut, surveillance logs (Wazuh+Zabbix), gestion vulnérabilités |
| **HIPAA** | Contrôle d'accès MFA, audit trails (Oracle+AD+Wazuh), intégrité FIM, backup chiffré |

---

## 🛠️ Stack Technique

| Couche | Technologie | Rôle |
|---|---|---|
| Hyperviseur | VMware ESXi 7.0 U2 | 15 VMs, vSwitch, port groups VGT |
| Pare-feu | OPNsense 26.1.3 | Cluster HA CARP, pfSync, OpenVPN, NAT |
| Annuaire | Windows Server 2022 | AD DS, DNS, DHCP, GPO, FSMO (2 DCs) |
| SIEM/XDR | Wazuh v4.14.4 | 35 règles custom, FIM, Active Response |
| Supervision | Zabbix 7.4.9 | 14 hôtes, agent+SNMP, 20K items |
| Portail MFA | Authelia v4.39.15 + Nginx | Forward-auth, TOTP, LDAPS |
| Base de données | Oracle 21c XE | Dossiers patients HDS (CLINIQUE_DATA) |
| Application | Node.js (port 8080) | Portail médical CRUD patients |
| Sauvegarde | Veeam B&R 13.0.1 | 3 jobs, CBT, 300 Go repository |
| ITSM | GLPI 10.0.15 | Ticketing, gestion d'assets |
| Alertes | Mailpit | SMTP relay (Zabbix + Wazuh + Authelia) |
| Accès distant | OpenVPN + Tailscale | VPN MFA + management OOB |

---

## 📸 Captures d'écran

**Pare-feu OPNsense**

| Dashboard FW1 | CARP FW1 (MASTER) | CARP FW2 (BACKUP) |
|---|---|---|
| ![](docs/assets/screenshots/opnsense-01-dashboard-fw1.png) | ![](docs/assets/screenshots/opnsense-02-carp-status-fw1.png) | ![](docs/assets/screenshots/opnsense-03-carp-status-fw2.png) |

**SIEM Wazuh**

| Dashboard (11 agents) | Threat Hunting MITRE | Active Response XSS |
|---|---|---|
| ![](docs/assets/screenshots/wazuh-01-dashboard-overview.png) | ![](docs/assets/screenshots/wazuh-03-threat-hunting.png) | ![](docs/assets/screenshots/wazuh-06-active-response-xss.png) |

**Zabbix Monitoring**

| Dashboard (14 hôtes) | Performance Oracle | Alertes |
|---|---|---|
| ![](docs/assets/screenshots/zabbix-01-dashboard.png) | ![](docs/assets/screenshots/zabbix-03-oracle-cpu-memory.png) | ![](docs/assets/screenshots/zabbix-04-alert-problems.png) |

**Authentification MFA**

| Login Authelia | TOTP (6 digits) | Portail médical |
|---|---|---|
| ![](docs/assets/screenshots/dmz-01-authelia-login.png) | ![](docs/assets/screenshots/dmz-02-authelia-totp.png) | ![](docs/assets/screenshots/oracle-01-portal-medical.png) |

**Backup Veeam**

| Jobs Oracle (Success) | Inventaire 15 VMs | Repository 300 Go |
|---|---|---|
| ![](docs/assets/screenshots/veeam-01-jobs-oracle-success.png) | ![](docs/assets/screenshots/veeam-02-inventory-15vms.png) | ![](docs/assets/screenshots/veeam-03-repository.png) |

---

## 🗂️ Structure du Dépôt

```
configs/
├── opnsense/       # 208 règles pfctl, 35 alias, CARP HA, NAT
├── wazuh/          # 35 règles HDS, ossec.conf, agent.conf
├── zabbix/         # Templates, documentation 14 hôtes
├── gpo/            # 10 GPOs, OUs, FSMO, DHCP, DNS, réplication AD
├── nginx/          # Reverse proxy, forward-auth, TLS, DMZ
├── authelia/       # MFA TOTP, LDAPS, politique deny/two_factor
├── oracle/         # Instance XE, tablespaces, listener, audit
└── veeam/          # PRA, 3 jobs, repository, inventaire 15 VMs

docs/assets/
├── screenshots/    # ~35 captures anonymisées
└── diagrams/       # Matrice de flux inter-VLAN
```

---

## 🚀 Ce que ce projet m'a appris

- **La défense en profondeur** n'est pas un buzzword — chaque couche détecte des menaces que les autres laissent passer
- **Des règles SIEM personnalisées** alignées sur des référentiels transforment un SIEM en moteur de détection
- **Pentester sa propre infrastructure** révèle des angles morts qu'une revue de config ne trouve pas
- **La documentation est un contrôle de sécurité** — si ce n'est pas documenté, ça n'existe pas pour un auditeur
- **L'isolation réseau stricte** (VLAN BACKUP sanctuarisé, GUEST totalement isolé) est la première ligne de défense contre le mouvement latéral

---

## 📄 Licence

[Licence MIT](LICENSE) · Configurations anonymisées · Aucun secret ni donnée patient réelle

---

**Construit avec 🔒 par Steeve WOMO** · *Master 2 — Infrastructure Réseaux et Sécurité*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Profil-0A66C2?style=for-the-badge&logo=linkedin)](https://linkedin.com/in/steeve-womo)
