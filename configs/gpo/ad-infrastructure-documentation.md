# Active Directory — Infrastructure Tier 0

> Domaine : `clinique-chatelet.local` — Windows 2016 Domain Mode  
> 2 contrôleurs de domaine (DC1 + DC2) — Réplication 0 erreur  
> LDAPS actif (port 636) — 10 GPOs de sécurité — 6 scopes DHCP

## Contrôleurs de Domaine

| Paramètre | DC1 | DC2 |
|---|---|---|
| Rôle | PDC + tous les rôles FSMO | DC secondaire / réplication |
| IP | 172.16.11.11 | 172.16.11.12 |
| OS | Windows Server 2022 (Windows2016Domain mode) | Windows Server 2022 |
| VLAN | 111 (SRV) | 111 (SRV) |
| LDAPS | ✅ Port 636 en écoute (vérifié) | ✅ Port 636 en écoute |
| Rôles installés | AD DS, DNS, DHCP, AD CS | AD DS, DNS |

## Rôles FSMO (tous sur DC1)

```
Contrôleur de schéma        DC1.clinique-chatelet.local
Maître des noms de domaine  DC1.clinique-chatelet.local
Contrôleur domaine princip. DC1.clinique-chatelet.local
Gestionnaire du pool RID    DC1.clinique-chatelet.local
Maître d'infrastructure     DC1.clinique-chatelet.local
```

## Réplication AD

| Source | Destination | Différence max | Échecs | Taux erreur |
|---|---|---|---|---|
| DC1 | DC2 | 55 min 45 s | 0 / 5 | **0%** |
| DC2 | DC1 | 54 min 13 s | 0 / 5 | **0%** |

Partitions répliquées (toutes en succès) :
- `DC=clinique-chatelet,DC=local` (domaine)
- `CN=Configuration` (topologie)
- `CN=Schema` (schéma)
- `DC=DomainDnsZones` (zones DNS domaine)
- `DC=ForestDnsZones` (zones DNS forêt)

## Structure des Unités d'Organisation (OU)

```
DC=clinique-chatelet,DC=local
├── OU=CLINIQUE_USERS
│   ├── OU=Medecins
│   ├── OU=IT
│   ├── OU=Soignants
│   │   ├── OU=IDE (Infirmiers Diplômés d'État)
│   │   └── OU=Aides-Soignants
│   └── OU=Administratifs
│       ├── OU=Secretariat
│       └── OU=Direction
│
├── OU=CLINIQUE_COMPUTERS
│   ├── OU=Servers
│   └── OU=Workstations
│
├── OU=CLINIQUE_GROUPS
│   ├── OU=Security_Groups
│   └── OU=Distribution_Groups
│
├── OU=CLINIQUE_SERVICE_ACCOUNTS
│
└── OU=Domain Controllers
```

## Comptes Utilisateurs (13)

| Compte | État | Rôle | OU / Fonction |
|---|---|---|---|
| `Administrateur` | ✅ Actif | Domain Admin | Compte intégré |
| `the.ghost` | ✅ Actif | Domain Admin + IT | Admin IT principal |
| `j.martin` | ✅ Actif | Utilisateur clinique | Soignant (IDE) |
| `m.leclerc` | ✅ Actif | Utilisateur clinique | Soignant |
| `s.dubois` | ✅ Actif | Utilisateur clinique | Personnel |
| `p.durand` | ✅ Actif | Utilisateur clinique | Personnel |
| `l.bernard` | ✅ Actif | Utilisateur clinique | Personnel |
| `svc_backup` | ✅ Actif | Service account | Veeam Backup |
| `svc_monitoring` | ✅ Actif | Service account | Zabbix + Wazuh |
| `svc_glpi` | ✅ Actif | Service account | GLPI ITSM |
| `svc_authelia` | ✅ Actif | Service account | Authelia LDAPS |
| `krbtgt` | ❌ Désactivé | Kerberos TGT | Compte système AD |
| `Invité` | ❌ Désactivé | Guest | Désactivé par sécurité |

**Domain Admins** : Administrateur, The Ghost (2 comptes seulement — principe du moindre privilège)

**Comptes de service** : 4 comptes dédiés (svc_*) séparés dans l'OU `CLINIQUE_SERVICE_ACCOUNTS` — pas de comptes nominatifs utilisés pour les services.

## Stratégies de Groupe (10 GPOs)

### GPOs de Sécurité (8 custom)

| GPO | Créée le | Cible | Paramètres clés |
|---|---|---|---|
| **GPO-Security-Password-Policy** | 12/02/2026 | Domaine | Longueur min **12 caractères**, complexité requise, historique 24 mots de passe, max 90 jours |
| **GPO-Security-Account-Lockout** | 12/02/2026 | Domaine | Seuil **5 tentatives**, verrouillage **30 min**, réinitialisation compteur 30 min |
| **GPO-Security-Audit-Policy** | 12/02/2026 | Domaine | Audit avancé : connexions, comptes, groupes, privilèges, partages, état sécurité (Succès + Échec) |
| **GPO-Security-Disable-SMBv1** | 12/02/2026 | Domaine | Désactivation SMBv1 via script PowerShell (protection WannaCry/NotPetya) |
| **GPO-Security-Windows-Firewall** | 12/02/2026 | Domaine | Pare-feu Windows activé sur tous les profils |
| **GPO-Security-User-Rights** | 12/02/2026 | Domaine | Restriction des droits utilisateur (audit sécurité, gestion journal, connexion réseau) |
| **GPO-BitLocker-NoTPM** | 14/02/2026 | Workstations | BitLocker sans TPM (mot de passe), récupération AD DS, clé 48 chiffres |
| **GPO-Workstations-Hardening** | 12/02/2026 | OU=Workstations | Durcissement postes de travail |

### GPOs par défaut (2)

| GPO | Cible | Paramètres |
|---|---|---|
| Default Domain Policy | Domaine | Mot de passe min 7 chars (remplacé par GPO-Security-Password-Policy à 12), verrouillage 5 tentatives, Kerberos 10h, pas de LM hash |
| Default Domain Controllers Policy | OU=DC | Audit connexions (Succès + Échec) |

### Politique d'Audit Avancée (GPO-Security-Audit-Policy)

| Catégorie d'audit | Paramètre |
|---|---|
| Validation des informations d'identification | Succès, Échec |
| Gestion des groupes de sécurité | Succès, Échec |
| Gestion des comptes d'utilisateur | Succès, Échec |
| Ouverture de session | Succès, Échec |
| Fermeture de session | Succès, Échec |
| Partage de fichiers | Succès, Échec |
| Modification de la stratégie d'audit | Succès, Échec |
| Utilisation de privilèges sensibles | Succès, Échec |
| Modification de l'état de la sécurité | Succès, Échec |
| Extension du système de sécurité | Succès, Échec |

## DNS — Zones configurées

| Zone | Type | Recherche inversée |
|---|---|---|
| `clinique-chatelet.local` | Primaire | Non |
| `_msdcs.clinique-chatelet.local` | Primaire | Non |
| `11.16.172.in-addr.arpa` | Primaire | **Oui** (VLAN 111) |
| `0.in-addr.arpa` | Primaire | Oui |
| `127.in-addr.arpa` | Primaire | Oui |

Enregistrements SRV détectés (zone `_msdcs`) :
- `_gc` → DC1 (3268) + DC2 (3268) — Global Catalog
- `_kerberos` → DC1 (88) + DC2 (88) — Kerberos
- `_ldap` → DC1 (389) + DC2 (389) — LDAP

## DHCP — 6 Scopes (un par VLAN)

| Scope | Nom | Plage | État |
|---|---|---|---|
| 172.16.11.0 | VLAN-111_SRV | .100 – .200 | ✅ Active |
| 172.16.22.0 | VLAN-222_CLIENT | .100 – .200 | ✅ Active |
| 172.16.33.0 | VLAN-333_DMZ | .100 – .200 | ✅ Active |
| 172.16.44.0 | VLAN-444_BACKUP | .100 – .200 | ✅ Active |
| 172.16.55.0 | VLAN_555-GUEST | .150 – .250 | ✅ Active |
| 172.16.99.0 | VLAN_999-MGMT | .150 – .250 | ✅ Active |

> Note : Les serveurs utilisent des IPs statiques (hors plage DHCP). Le DHCP est relayé via DHCP Relay sur OPNsense (DHCPv4 Relay visible sur le dashboard FW1).

## Captures d'écran

| Fichier | Contenu |
|---|---|
| `ad-01-ou-structure.png` | Arborescence AD Users and Computers |
| `ad-02-dns-manager.png` | Gestionnaire DNS avec zones directes et inversées |
| `ad-03-dhcp-scopes.png` | 6 scopes DHCP (un par VLAN) |
| `ad-04-gpo-management.png` | Console GPO avec les 10 stratégies |