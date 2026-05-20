# Poste Admin IT Durcis — Windows 10 Pro

> POSTE-ADMIN-IT — Windows 10 Pro 22H2 (10.0.19045) — VLAN 999 (MGMT)  
> BitLocker XTS-AES 128 | UAC max | Defender temps réel | 9 GPOs appliquées

## Identification

| Paramètre | Valeur |
|---|---|
| Hostname | POSTE-ADMIN-IT |
| OU AD | `OU=Workstations,OU=CLINIQUE_COMPUTERS,DC=clinique-chatelet,DC=local` |
| IP | 172.16.99.115 |
| VLAN | 999 (MGMT) — Out-of-Band |
| OS | Windows 10 Pro 22H2 (10.0.19045) |
| Utilisateur | the.ghost (OU=IT, Domain Admins, Protected Users) |
| GPO depuis | DC1.clinique-chatelet.local |

## BitLocker — Chiffrement Disque

| Paramètre | Valeur |
|---|---|
| Volume | C: (volume système) |
| Taille | 59,37 Go |
| Version BitLocker | 2.0 |
| Méthode | **XTS-AES 128** |
| Pourcentage chiffré | **100%** |
| État protection | **Activée** |
| État verrouillage | Déverrouillé |
| Protecteurs | Password + Mot de passe numérique (récupération AD DS) |

## UAC — Contrôle de Compte Utilisateur

| Paramètre | Valeur | Signification |
|---|---|---|
| ConsentPromptBehaviorAdmin | **5** | Niveau maximum — demande consentement sur bureau sécurisé |
| EnableLUA | **1** | UAC activé |
| PromptOnSecureDesktop | **1** | Bureau sécurisé activé (anti-keylogger) |

## Windows Defender

| Paramètre | Valeur |
|---|---|
| Antivirus activé | ✅ True |
| Protection temps réel | ✅ True |
| Dernière MAJ signatures | 20/05/2026 08:29:54 |

## Pare-feu Windows

| Direction | Action | Nombre de règles |
|---|---|---|
| Inbound | Allow | 322 |
| Outbound | Allow | 326 |

## GPOs Appliquées (9)

| GPO | Statut |
|---|---|
| GPO-Security-User-Rights | ✅ Appliquée |
| GPO-Workstations-Hardening | ✅ Appliquée |
| GPO-BitLocker-NoTPM | ✅ Appliquée |
| Default Domain Policy | ✅ Appliquée |
| GPO-Security-Password-Policy | ✅ Appliquée |
| GPO-Security-Account-Lockout | ✅ Appliquée |
| GPO-Security-Audit-Policy | ✅ Appliquée |
| GPO-Security-Disable-SMBv1 | ✅ Appliquée |
| GPO-Security-Windows-Firewall | ✅ Appliquée |
| Stratégie locale | ❌ Non appliquée (vide) |

## Groupes de Sécurité de l'Utilisateur (the.ghost)

- **Admins du domaine** — Accès complet AD
- **Protected Users** — Protection Kerberos renforcée (pas de NTLM, pas de délégation)
- **Administrateur IT (Droits élevés)** — Groupe custom clinique
- **OpenVPN Administrators** — Gestion tunnel VPN
- **Equipe Informatique** — Groupe métier IT
- Utilisateurs du Bureau à distance
- Groupe de réplication RODC refusé

## Fichiers

| Fichier | Description |
|---|---|
| [`bitlocker-status.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/workstation/bitlocker-status.txt) | BitLocker XTS-AES 128, 100% |
| [`defender-status.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/workstation/defender-status.txt) | Defender temps réel actif |
| [`uac-config.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/workstation/uac-config.txt) | UAC niveau max (5/1/1) |
| [`gpresult-summary.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/workstation/gpresult-summary.txt) | 9 GPOs appliquées |
| [`firewall-rules-summary.txt`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/workstation/firewall-rules-summary.txt) | 648 règles firewall Windows |