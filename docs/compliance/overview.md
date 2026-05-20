# 📋 Conformité

## Mapping des Contrôles

| Référentiel | Contrôles implémentés |
|---|---|
| **HDS** | Segmentation 6 VLANs, LDAPS, MFA TOTP, audit DB Oracle, backup sanctuarisé, PRA |
| **ISO 27001** | SMSI, analyse risques, Default Deny, journalisation centralisée, réponse incident |
| **RGPD Art. 32** | Chiffrement (BitLocker, TLS, LDAPS), pseudonymisation, tests résilience |
| **PCI DSS** | Firewall segmenté, deny par défaut, surveillance logs, gestion vulnérabilités |
| **HIPAA** | Contrôle d'accès MFA, audit trails, intégrité FIM, backup chiffré |

## Règles Wazuh mappées

| Framework | Tags | Règles |
|---|---|---|
| PCI DSS | 10.2.4, 10.2.5, 10.6 | 12 règles |
| HIPAA | 164.312.a.1, 164.312.b, 164.312.d | 15 règles |
| MITRE ATT&CK | T1070.001, T1550.002 | 2 règles |
