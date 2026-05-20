# 📈 Supervision — Zabbix 7.4.9

14 hôtes, 20 088 items, 7 728 triggers, 19,67 vps. Dual-stack (agent + SNMP) sur les firewalls.

## Hôtes supervisés

| Hôte | Interface | Items | VLAN |
|---|---|---|---|
| DC1 / DC2 | ZBX :10050 | 110 chacun | SRV |
| OPNsense-FW1/FW2 | **ZBX + SNMP** :161 | 67 + 120 | trunk |
| srv-oracle-db-01 | ZBX :10050 | **213** | SRV |
| srv-zabbix / srv-wazuh / GLPI | ZBX :10050 | 75–153 | SRV |
| SRV-PROXY / SRV-WEB | ZBX :10050 | 75–89 | DMZ |
| SRV-VEEAM-BACKUP | ZBX :10050 | 158 | BACKUP |
| POSTE-ADMIN-IT | ZBX :10050 | 109 | MGMT |

## Fichiers

| Fichier | Description |
|---|---|
| [`zabbix-monitoring-documentation.md`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/zabbix/zabbix-monitoring-documentation.md) | Documentation complète |
| [`custom-templates.yaml`](https://github.com/Yemah/clinique-chatelet-secure-infra/blob/main/configs/zabbix/custom-templates.yaml) | Templates FreeBSD exportés |

![Zabbix Dashboard](../assets/screenshots/zabbix-01-dashboard.png)
