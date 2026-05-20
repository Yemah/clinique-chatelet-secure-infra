# Sauvegarde & PRA — Veeam Backup & Replication 13

> Veeam B&R 13.0.1.2067 Community Edition — Windows Server 2019 — VLAN 444 (BACKUP)  
> Repository dédié : 300 Go (E:\VeeamBackup) — 3 jobs planifiés — CBT activé  
> Malware Detection : 15 VMs analysées → toutes "Clean"

## Infrastructure de Backup

| Paramètre | Valeur |
|---|---|
| Serveur | SRV-VEEAM-BACKUP (172.16.44.70) |
| Version | Veeam B&R 13.0.1.2067 Community Edition |
| OS | Windows Server 2019 (64-bit) |
| VLAN | 444 (BACKUP) — zone sanctuarisée |
| Hyperviseur connecté | VMware vSphere — ESXi 192.168.1.153 |
| VMs inventoriées | 15 machines virtuelles |
| Analyse malware | ✅ Toutes les VMs : statut "Clean" |

## Repository

| Paramètre | Valeur |
|---|---|
| Nom | REPO-BACKUP-CHATELET |
| Type | Windows (disque local) |
| Chemin | `E:\VeeamBackup` |
| Capacité | 300 Go |
| Utilisé | 117,7 Go (39%) |
| Libre | 81,4 Go (27%) |
| Description | Volume backup clinique Chatelet |

## Jobs de Backup (3)

| Job | Type | Objets | Dernier résultat | Prochaine exécution |
|---|---|---|---|---|
| **JOB-BACKUP-ORACLE-HDS** | VMware Backup | 1 | ✅ **Success** | 23/05/2026 22:00 |
| JOB-IMPORTANT | VMware Backup | 3 | ⚠️ Warning | 20/05/2026 03:30 |
| JOB-CRITICAL | VMware Backup | 6 | ⚠️ Warning | 21/05/2026 02:00 |

### Détail — JOB-BACKUP-ORACLE-HDS (job critique HDS)

| Métrique | Valeur |
|---|---|
| VM protégée | SRV-ORACLE-DB-01 |
| Taille VM | 80 Go |
| Données lues | 1,8 Go (incrémental CBT) |
| Données transférées | 532,2 Mo |
| **Ratio compression** | **3,5x** |
| Durée totale | 19 min 34 s |
| Durée processing | 16 min 19 s |
| Débit | 5 Mo/s |
| Bottleneck | Source (99%) |
| Charge | Source 99% → Proxy 44% → Network 1% → Target 0% |
| CBT (Changed Block Tracking) | ✅ Activé |
| Statut | ✅ **Success** — terminé le 18/05/2026 à 11:08:47 |

> **CBT (Changed Block Tracking)** : VMware ne lit que les blocs modifiés depuis le dernier backup, réduisant les données lues de 80 Go à 1,8 Go (97,75% de réduction). C'est ce qui permet un backup incrémental en 16 minutes au lieu de plusieurs heures.

## Inventaire VMware (15 VMs sur ESXi)

| VM | Taille disque | OS | Dernier backup | Malware |
|---|---|---|---|---|
| DC1 | 68,1 Go | Windows Server 2019 | < 1 jour | Clean |
| DC2 | 68,1 Go | Windows Server 2019 | < 1 jour | Clean |
| GLPI Serveur | 68,1 Go | Ubuntu Linux (64-bit) | < 1 jour | Clean |
| ITSM-Srv | 64,1 Go | Ubuntu Linux (64-bit) | jamais | Clean |
| Kali-Audit-Test | 34,1 Go | Debian GNU/Linux 11 | jamais | Clean |
| OPNsense-FW1 | 66,1 Go | FreeBSD 13 (64Guest) | < 1 jour | Clean |
| OPNsense-FW2 | 46,1 Go | FreeBSD 13 (64Guest) | < 1 jour | Clean |
| Poste-Admin-IT-Durcis | 68,1 Go | Windows 10 (64-bit) | jamais | Clean |
| SRV_WAZUH | 116,1 Go | Ubuntu Linux (64-bit) | < 1 jour | Clean |
| SRV-ORACLE-DB-01 | 93,7 Go | Oracle Linux 9 (64-bit) | < 1 jour | Clean |
| SRV-PROXY-01 | 54,1 Go | Ubuntu Linux (64-bit) | < 1 jour | Clean |
| SRV-VEEAM-BACKUP | 309,7 Go | Windows Server 2019 | jamais | Clean |
| SRV-WEB-01 | 54,1 Go | Ubuntu Linux (64-bit) | < 1 jour | Clean |
| SRV-ZABBIX | 108,1 Go | Ubuntu Linux (64-bit) | < 1 jour | Clean |
| TestMH | 16 Go | Ubuntu | jamais | Clean |

**Stockage total VMs** : ~1,28 To sur ESXi  
**VMs protégées** (backup < 1 jour) : 10/15  
**VMs non protégées** : 5 (Kali=test, ITSM/Poste/Veeam/TestMH=non critique ou auto-exclusion)

## Isolation Réseau du VLAN BACKUP

Le VLAN 444 est sanctuarisé par des règles firewall strictes :

**Trafic autorisé DEPUIS le BACKUP :**
- Veeam → VMs cibles : ports 135, 445, 902, 2500-3300, 6160-6161, 9392
- Veeam → AD (DNS + NTP) : ports 53, 123
- Veeam → Internet : HTTP/S (mises à jour)
- Veeam → Wazuh : ports 1514/1515 (agent SIEM)
- Veeam → Zabbix : port 10051 (agent monitoring)

**Trafic BLOQUÉ explicitement :**
- BACKUP → CLIENT : ❌ `[BLOCK/LOG] BACKUP → CLIENT interdit`
- BACKUP → DMZ : ❌ `[BLOCK/LOG] BACKUP → DMZ interdit`
- Tout le reste : ❌ Default Deny

**Détection Wazuh** : La règle custom 200202 (niveau 12) détecte toute tentative d'accès non autorisé depuis le VLAN BACKUP et déclenche une Active Response `pf` sur OPNsense (blocage 3600s).

## Plan de Reprise d'Activité (PRA)

### Objectifs

| Métrique | Objectif | Réalisé |
|---|---|---|
| RPO (Recovery Point Objective) | < 24h | ✅ Backup quotidien Oracle HDS |
| RTO (Recovery Time Objective) | < 4h | ✅ Restauration VM complète via Veeam |
| Compression | > 3x | ✅ 3,5x mesuré |
| Intégrité | Vérification automatique | ✅ CBT + checksum Veeam |

### Scénarios de reprise

| Scénario | Procédure | Temps estimé |
|---|---|---|
| Perte Oracle DB | Restauration VM SRV-ORACLE-DB-01 depuis JOB-BACKUP-ORACLE-HDS | ~20 min |
| Perte DC1 (FSMO) | Restauration DC1 + seizure FSMO si nécessaire → DC2 prend le relais | ~30 min |
| Perte complète FW1 | FW2 promeut CARP automatiquement (transparent, 0 downtime) | ~3 sec |
| Ransomware | Isolation VLAN, restauration VMs clean depuis Veeam (malware scan) | ~2h |

### Haute disponibilité intégrée

| Composant | Mécanisme HA | Failover |
|---|---|---|
| Pare-feu | CARP + pfSync (FW1/FW2) | Automatique (~3 sec) |
| Active Directory | 2 DCs (DC1 + DC2) avec réplication | Automatique (DNS round-robin) |
| DNS/DHCP | 2 serveurs DNS (DC1 + DC2) | Automatique |
| Backup | Repository dédié E:\VeeamBackup (300 Go) | Manuel (restauration Veeam) |

## Captures d'écran

| Fichier | Contenu |
|---|---|
| `veeam-01-jobs-oracle-success.png` | 3 jobs + détail Oracle HDS (80 Go, CBT, Success) |
| `veeam-02-inventory-15vms.png` | 15 VMs inventoriées (taille, OS, malware: Clean) |
| `veeam-03-repository.png` | REPO-BACKUP-CHATELET (300 Go, 39% utilisé) |
| `veeam-04-server-desktop.png` | Bureau SRV-VEEAM (Veeam B&R, ONE, Recovery, Wazuh agent) |