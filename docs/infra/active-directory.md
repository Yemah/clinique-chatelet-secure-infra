# 🏢 Active Directory — Tier 0

Domaine `clinique-chatelet.local`, Windows2016Domain, 2 DCs, réplication 0 erreur, LDAPS actif, 10 GPOs.

## Contrôleurs

| | DC1 | DC2 |
|---|---|---|
| IP | 172.16.11.11 | 172.16.11.12 |
| FSMO | 5/5 rôles | — |
| LDAPS | ✅ 636 | ✅ 636 |
| Réplication | 0 erreur | 0 erreur |

## GPOs de Sécurité (8 custom + 2 default)

| GPO | Paramètres clés |
|---|---|
| Password Policy | Min 12 chars, complexité, historique 24 |
| Account Lockout | 5 tentatives → 30 min |
| Audit Policy | 10 catégories (Succès + Échec) |
| Disable SMBv1 | Script PS anti-WannaCry |
| Windows Firewall | Activé sur tous les profils |
| BitLocker NoTPM | Chiffrement sans TPM, récupération AD DS |
| Workstations Hardening | Durcissement global |

## Structure OU

```
clinique-chatelet.local
├── CLINIQUE_USERS (Medecins, IT, Soignants/IDE/Aides, Administratifs/Secretariat/Direction)
├── CLINIQUE_COMPUTERS (Servers, Workstations)
├── CLINIQUE_GROUPS (Security_Groups, Distribution_Groups)
└── CLINIQUE_SERVICE_ACCOUNTS (svc_backup, svc_monitoring, svc_glpi, svc_authelia)
```

## Fichiers

| Fichier | Description |
|---|---|
| [`ad-infrastructure-documentation.md`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/gpo/ad-infrastructure-documentation.md) | Documentation AD complète |
| [`gpo-list.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/gpo/gpo-list.txt) | 10 GPOs |
| [`ou-structure.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/gpo/ou-structure.txt) | 17 OUs |
| [`fsmo-roles.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/gpo/fsmo-roles.txt) | 5 rôles FSMO |
| [`dhcp-scopes.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/gpo/dhcp-scopes.txt) | 6 scopes DHCP |
| [`dns-zones.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/gpo/dns-zones.txt) | Zones DNS |
