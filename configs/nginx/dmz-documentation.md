# DMZ — Reverse Proxy Nginx + Authelia MFA

> SRV-PROXY-01 (172.16.33.10) — Ubuntu 22.04 LTS — VLAN 333 (DMZ)  
> Nginx 1.18.0 + Authelia v4.39.15 + Mailpit (SMTP 1025 / Web 8025)  
> Forward-auth pattern — TOTP obligatoire — Backend LDAPS vers AD

## Architecture MFA — Flux d'authentification

```
Navigateur                      SRV-PROXY (DMZ)                      SRV (VLAN 111)
────────────                    ──────────────────                   ──────────────

1. GET /                ──►     Nginx (443)
                                 │
2. auth_request         ──►     /authelia/api/verify
                                 │  ❌ Pas de session
3. 302 Redirect         ◄──     → /authelia (login page)

4. POST credentials     ──►     Authelia (9091)
                                 │
5. LDAPS verify         ──────► DC1:636 (svc_authelia)
                                 │  ✅ Credentials OK
6. TOTP prompt          ◄──     Authelia → "Entrez le code"

7. POST TOTP code       ──►     Authelia
                                 │  ✅ TOTP validé
8. Set cookie           ◄──     authelia_session cookie
9. 302 → /              ◄──     Redirect vers portail

10. GET / (avec cookie) ──►     Nginx → auth_request verify
                                 │  ✅ Session valide
11. proxy_pass          ──────► SRV-WEB (8080)
                                 │
12. Oracle query        ──────────────────────────► Oracle (1521)
13. Patient data        ◄──────────────────────────
14. HTML response       ◄──     Page portail médical
```

## Authelia v4.39.15 — Configuration

| Paramètre | Valeur |
|---|---|
| Version | v4.39.15 |
| Écoute | `tcp://127.0.0.1:9091/authelia` (localhost only) |
| Log level | debug |
| Stockage sessions | SQLite3 (`/var/lib/authelia/db.sqlite3`) |
| Notifications | SMTP via Mailpit local (127.0.0.1:1025) |
| Certificats LDAPS | `/etc/authelia/certificates/` (CA interne clinique) |

### TOTP (Time-based One-Time Password)

| Paramètre | Valeur |
|---|---|
| Issuer | Clinique Le Chatelet |
| Algorithme | SHA1 |
| Digits | 6 |
| Période | 30 secondes |

### Politique d'accès (access_control)

```yaml
access_control:
  default_policy: deny        # TOUT est bloqué par défaut
  rules:
    - domain: 'portal.clinique-chatelet.local'
      policy: two_factor      # 2FA obligatoire pour le portail
```

> Politique **deny par défaut** — seul le domaine `portal.clinique-chatelet.local` est accessible, et uniquement après authentification à deux facteurs. Aucun bypass, aucune exception.

### Backend d'authentification (LDAPS → Active Directory)

| Paramètre | Valeur |
|---|---|
| Implémentation | Active Directory |
| Adresse | `ldaps://ldap.clinique-chatelet.local:636` |
| TLS minimum | TLS 1.2 |
| skip_verify | false (validation certificat CA) |
| Base DN | `DC=clinique-chatelet,DC=local` |
| Compte de liaison | `CN=svc_authelia,OU=CLINIQUE_SERVICE_ACCOUNTS,...` |
| Filtre utilisateurs | `(&({username_attribute}={input})(objectClass=user)(!(disabled)))` |
| Filtre groupes | `(&(objectClass=group)(member={dn}))` |
| Attribut username | sAMAccountName |
| Attribut email | mail |
| Attribut display | displayName |

### Gestion des secrets

Tous les secrets sont externalisés dans des fichiers dédiés (`file:///etc/authelia/secrets/`) et ne sont jamais en clair dans la configuration :

| Secret | Fichier | Usage |
|---|---|---|
| JWT secret | `/etc/authelia/secrets/jwt_secret` | Signature tokens de réinitialisation |
| Session secret | `/etc/authelia/secrets/session_secret` | Chiffrement cookies de session |
| Encryption key | `/etc/authelia/secrets/encryption_key` | Chiffrement base SQLite |
| LDAP password | `/etc/authelia/secrets/ldap_password` | Bind password AD (svc_authelia) |

### Session

| Paramètre | Valeur |
|---|---|
| Domaine cookie | clinique-chatelet.local |
| Expiration | 8 heures |
| Inactivité | 1 heure |
| Remember me | 1 mois |
| Redirection | `https://portal.clinique-chatelet.local/` |

## Nginx — Reverse Proxy Configuration

### Architecture des vhosts

```
/etc/nginx/sites-enabled/
├── portal.clinique-chatelet.local     # Portail médical (443/SSL + Authelia)
└── glpi-srv.clinique-chatelet.local   # GLPI ITSM (80/HTTP, reverse proxy)

/etc/nginx/snippets/
└── authelia-location.conf             # Headers forward-auth (Remote-User, etc.)

/etc/nginx/ssl/
├── portal.crt                         # Certificat auto-signé RSA 2048
└── portal.key                         # Clé privée (chmod 600)
```

### Portail médical — Flux Nginx

1. **Port 80** : Redirect 301 → HTTPS
2. **Port 443** : SSL + HTTP/2
3. **`/authelia`** : Proxy vers Authelia (9091) pour login/TOTP
4. **`/authelia/api/verify`** : Endpoint interne de vérification (auth_request)
5. **`/`** : Protégé par auth_request → proxy vers SRV-WEB (8080)

Headers transmis au backend après authentification :
- `Remote-User` : nom d'utilisateur AD (ex: j.martin)
- `Remote-Groups` : groupes AD de l'utilisateur
- `Remote-Name` : nom complet (ex: Dr. Jean Martin)
- `X-Forwarded-User` : identique à Remote-User

### Certificat TLS

| Paramètre | Valeur |
|---|---|
| Type | Auto-signé (RSA 2048 bits) |
| Issuer | `C=FR, ST=Ile-de-France, L=Paris, O=Clinique Le Chatelet` |
| CN | `portal.clinique-chatelet.local` |
| Validité | 27/02/2026 → 27/02/2027 (365 jours) |
| Signature | SHA256withRSA |

## Mailpit — Relais SMTP de test

| Paramètre | Valeur |
|---|---|
| SMTP | 127.0.0.1:1025 (localhost, non chiffré) |
| Web UI | 172.16.33.10:8025 (accessible depuis MGMT) |
| Utilisé par | Authelia (notifications MFA) + Wazuh (alertes) + Zabbix (alertes) |
| Total emails | 464+ dans la boîte |

Emails Authelia observés dans Mailpit :
- `[Clinique Le Châtelet] Second Factor Method Added` — Enrôlement TOTP Dr. Jean Martin
- `[Clinique Le Châtelet] Second Factor Method Removed` — Reset TOTP
- `[Clinique Le Châtelet] Confirm your identity` — Vérification identité

## Preuves visuelles du flux MFA complet

Le flux MFA de bout en bout est prouvé par 3 captures consécutives :

1. **Étape 1 — Login LDAPS** : Page Authelia "Se connecter" avec j.martin
2. **Étape 2 — TOTP** : Page "Mot de passe à usage unique" (6 digits, 30s)
3. **Étape 3 — Notification** : Email Mailpit "Second Factor Method Added"

→ Après validation TOTP, l'utilisateur est redirigé vers le portail médical (screenshot Oracle bloc 6).

## Captures d'écran

| Fichier | Contenu |
|---|---|
| `dmz-01-authelia-login.png` | Page de connexion Authelia (LDAPS) |
| `dmz-02-authelia-totp.png` | Page TOTP "Mot de passe à usage unique" |
| `dmz-03-mailpit-mfa-notification.png` | Email "Second Factor Method Added" |