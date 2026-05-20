# GLPI — IT Service Management

> GLPI 10.0.15 — Ubuntu 22.04 LTS — Apache 2.4 + MariaDB 10.6.23 — VLAN 111 (SRV)  
> 3 tickets, notifications Mailpit, LDAP AD, cron chaque minute, backup quotidien

## Identification

| Paramètre | Valeur |
|---|---|
| Version | GLPI 10.0.15 |
| Hostname | glpi-srv.clinique-chatelet.local |
| IP | 172.16.11.60 |
| VLAN | 111 (SRV) |
| OS | Ubuntu 22.04.5 LTS (kernel 5.15.0-179) |
| Web server | Apache 2.4 (port 80) |
| Base de données | MariaDB 10.6.23 (localhost:3306) |
| PHP | 8.1 |
| Accès web | `http://glpi-srv.clinique-chatelet.local` (via reverse proxy Nginx DMZ) |
| Disque | 29 Go (30% utilisé) |

## Tickets (3 créés)

| ID | Titre | Statut | Date | Demandeur | Catégorie |
|---|---|---|---|---|---|
| #0000001 | Changement d'encre imprimante | 🟢 Nouveau | 15/05/2026 23:50 | Ghost The | Matériel > Imprimantes |
| #0000002 | Pas d'accès internet | 🟢 Nouveau | 17/05/2026 00:40 | Ghost The | Réseau & Sécurité |
| #0000003 | Probleme VPN | 🟢 Nouveau | 20/05/2026 08:32 | Martin Jean | Réseau & Sécurité |

Notifications email fonctionnelles — chaque ticket génère 2 emails via Mailpit :
- 1 vers `glpi-admin.it@clinique-chatelet.local` (admin IT)
- 1 vers le demandeur (`the.ghost@` ou `j.martin@`)

## Intégration LDAP Active Directory

GLPI est connecté à l'AD via LDAP — les utilisateurs se connectent avec leurs identifiants domaine. Preuve : `j.martin` visible en mode "Self-Service" (screenshot).

## Configuration Apache (vhost)

```apache
<VirtualHost *:80>
    ServerName glpi-srv.clinique-chatelet.local
    DocumentRoot /var/www/glpi/public
    <Directory /var/www/glpi/public>
        Require all granted
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/glpi-error.log
    CustomLog ${APACHE_LOG_DIR}/glpi-access.log combined
</VirtualHost>
```

## Cron — Tâches automatiques

```
* * * * * /usr/bin/php8.1 /var/www/glpi/front/cron.php &>/dev/null
```

Exécution chaque minute : notifications email, collecte automatique, nettoyage sessions.

## Sauvegarde (PRA GLPI)

Script de backup quotidien (`/etc/cron.daily/glpi-backup`) :
- **Base MariaDB** : `mysqldump` compressé gzip
- **Fichiers joints** : `tar.gz` du dossier `/var/www/glpi/files/`
- **Rétention** : 7 jours (suppression automatique `find -mtime +7`)
- **Stockage** : `/var/backups/glpi/`

Backups existants (5 jours de rétention visible) :

| Date | Base (gzip) | Fichiers (tar.gz) |
|---|---|---|
| 16/05/2026 | 107 Ko | 402 Ko |
| 17/05/2026 | 140 Ko | 544 Ko |
| 18/05/2026 | 172 Ko | 628 Ko |
| 19/05/2026 | 203 Ko | 712 Ko |
| 20/05/2026 | 236 Ko | 799 Ko |

## Agents de supervision

| Agent | Version | Statut |
|---|---|---|
| Zabbix Agent | 7.4.x (port 10050) | ✅ Active (4 jours) |
| Wazuh Agent | v4.14.4 | ✅ Active (4 jours) |

## Pare-feu UFW

| Port | Service | Accès |
|---|---|---|
| 22/tcp | SSH | Anywhere |
| 80/tcp | HTTP (GLPI) | Anywhere |
| 443/tcp | HTTPS | Anywhere |
| 10050/tcp | Zabbix Agent | Anywhere |

## Captures d'écran

| Fichier | Contenu |
|---|---|
| `glpi-01-tickets-admin.png` | 3 tickets (vue Super-Admin) |
| `glpi-02-ticket-selfservice.png` | Ticket "Probleme VPN" (vue Self-Service j.martin) |
| `glpi-03-mailpit-notifications.png` | Notifications email GLPI dans Mailpit |