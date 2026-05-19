# Architecture Applicative & Base de Données Oracle

> Oracle Database 21c Express Edition — Oracle Linux 9 — VLAN 111 (SRV)  
> Instance XE — Uptime 7+ jours — Tablespace HDS dédié : CLINIQUE_DATA (1 Go)  
> Portail Web : Node.js → Nginx/Authelia (MFA) → Oracle 1521

## Flux Applicatif de bout en bout

```
Soignant distant                   DMZ (VLAN 333)                    SRV (VLAN 111)
─────────────────     ─────────────────────────────     ─────────────────────────────

  Navigateur     ──►  Nginx (443)                 
  (HTTPS)              │                           
                       ├─► Authelia (9091)          
                       │   ✅ LDAPS → DC1 (636)    
                       │   ✅ TOTP vérifié          
                       │                           
                       └─► SRV-WEB (8080)     ──►  Oracle 21c (1521)
                           Node.js portal           │
                           CLINIQUE_APP user         ├── CLINIQUE_DATA tablespace
                                                     └── Table PATIENTS
```

## Instance Oracle

| Paramètre | Valeur |
|---|---|
| Version | Oracle Database 21c Express Edition 21.3.0.0.0 |
| Instance | XE |
| Statut | OPEN |
| Créée le | 25/02/2026 |
| Hôte | srv-oracle-db-01 (172.16.11.20) |
| OS | Oracle Linux 9 |
| VLAN | 111 (SRV) |
| Uptime listener | 7 jours 5h 56min |

## Listener

```
Alias                     LISTENER
Port                      1521 (TCP)
Sécurité                  ON: Local OS Authentication
SNMP                      OFF

Services enregistrés :
  - XE          → Instance XE, status READY
  - XEXDB       → Instance XE, status READY (XML DB)
  - xepdb1      → Instance XE, status READY (Pluggable DB)

Endpoints :
  - TCP  0.0.0.0:1521      (accès réseau — filtré par firewall)
  - IPC  EXTPROC1521        (appels locaux)
  - TCPS 127.0.0.1:5500    (EM Express — wallet TLS, localhost only)
```

## Tablespaces

| Tablespace | Taille | Fonction |
|---|---|---|
| **CLINIQUE_DATA** | **1 024 Mo** | **Données patients HDS** (dédié) |
| SYSTEM | 1 360 Mo | Dictionnaire de données Oracle |
| SYSAUX | 1 020 Mo | Composants auxiliaires Oracle |
| UNDOTBS1 | 120 Mo | Segments d'annulation (rollback) |
| USERS | 5 Mo | Tablespace utilisateur par défaut |
| **Total** | **~3,5 Go** | |

> Le tablespace `CLINIQUE_DATA` est dédié aux données de santé — séparation physique des données métier et système, conforme aux exigences HDS de traçabilité et d'isolation.

## Comptes Utilisateurs Oracle

| Compte | Statut | Créé le | Rôle |
|---|---|---|---|
| **CLINIQUE_APP** | ✅ OPEN | 25/02/2026 | Compte applicatif — utilisé par le portail Node.js |
| **C##ZBX_MONITOR** | ✅ OPEN | 25/04/2026 | Supervision Zabbix (lecture seule v$instance, v$tablespace) |
| SYSRAC | OPEN | 17/08/2021 | Compte système Oracle RAC (intégré) |
| 16 autres comptes | 🔒 LOCKED | 2021 | Comptes Oracle par défaut — **tous verrouillés** |

**Principe du moindre privilège** : seuls 2 comptes applicatifs sont ouverts (CLINIQUE_APP pour l'application, C##ZBX_MONITOR pour la supervision). Les 16 comptes Oracle par défaut non utilisés sont verrouillés.

## Schéma de Données — Table PATIENTS

```sql
-- Propriétaire : CLINIQUE_APP
-- Tablespace : CLINIQUE_DATA

CREATE TABLE CLINIQUE_APP.PATIENTS (
    PATIENT_ID      NUMBER PRIMARY KEY,
    NOM             VARCHAR2(100),
    PRENOM          VARCHAR2(100),
    DATE_NAI        DATE,
    NUMERO_SECU     VARCHAR2(15),    -- N° Sécurité Sociale (donnée HDS)
    EMAIL           VARCHAR2(100),
    TELEPHONE       VARCHAR2(15),
    ADRESSE         VARCHAR2(200),
    DATE_CREATION   TIMESTAMP
);

-- 2 enregistrements de test (données fictives)
```

> ⚠️ **Note** : les données affichées dans le portail et la base sont **entièrement fictives** (patients de test). Aucune donnée patient réelle n'est stockée dans ce lab.

## Paramètres de Sécurité

| Paramètre | Valeur | Signification |
|---|---|---|
| `audit_trail` | **DB** | Audit activé — traces en base (conforme HDS/ISO 27001) |
| `remote_login_passwordfile` | **EXCLUSIVE** | Fichier de mots de passe dédié à cette instance |
| `sec_case_sensitive_logon` | (par défaut : TRUE en 21c) | Mots de passe sensibles à la casse |

## Portail Web Médical

Le screenshot du portail montre la chaîne complète en fonctionnement :

- **Authentification** : Dr. Jean Martin connecté (via Authelia TOTP + LDAPS)
- **Connexion DB** : Indicateur "Oracle DB — Connecté" (vert)
- **Données** : 2 dossiers patients affichés avec N° Sécu, contacts, adresses
- **URL** : `https://portal.clinique-chatelet.local` (HTTPS, certificat interne)
- **Actions** : Consultation dossier + modification (CRUD)

Ce screenshot est la **preuve de bout en bout** que toute l'infrastructure fonctionne :
`Utilisateur → MFA → Reverse Proxy → Application → Base Oracle → Données HDS`

## Contrôles d'accès réseau vers Oracle

Seul le serveur web (SRV-WEB, 172.16.33.20) est autorisé à contacter Oracle sur le port 1521. Ce flux est contrôlé par une règle firewall explicite sur VLAN_DMZ :

```
pass in quick on vlan0.333 inet proto tcp
    from <HOST_WEBAPP> to <HOST_ORACLE> port = 1521
    flags S/SA keep state
```

Tout autre accès depuis la DMZ vers le réseau interne est bloqué :
```
block drop in log quick on vlan0.333 inet
    from <NET_DMZ> to <PRIVATE_NETWORK>
```

## Captures d'écran

| Fichier | Contenu |
|---|---|
| `oracle-01-portal-medical.png` | Portail web avec dossiers patients (données fictives) |