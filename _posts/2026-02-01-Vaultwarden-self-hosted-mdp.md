---
title: "Vaultwarden : self-hôte ton gestionnaire de mots de passe"
date: 2026-02-01 10:00 +0200
categories: ["Sysops"]
tags: ["vaultwarden", "bitwarden", "self-hosted", "docker", "sécurité", "sysops", "infrastructure"]
author: marvax
---

> Tes mots de passe sont la clé de ton château. Les confier à un service tiers, c'est faire confiance à quelqu'un d'autre pour garder la clé. Vaultwarden te permet d'héberger un gestionnaire de mots de passe compatible Bitwarden, léger, en Docker, sur ton propre serveur. Voici comment faire, proprement.

<!--more-->

## Pourquoi self-héberger son gestionnaire de mots de passe ?

Le [fiasco LastPass](/posts/2022-12-22-LastPass-coffre-fort-trahi/) a montré que même les spécialistes du mot de passe se font défoncer. Self-héberger, c'est :

- **Contrôle total** : tes données ne quittent jamais ton infrastructure
- **Pas de dépendance** : si le service public est down ou compromis, tu n'es pas impacté
- **Confidentialité** : aucun tiers ne peut accéder à tes vaults, même en théorie
- **Customisation** : intégration avec ton réseau, ton reverse proxy, ta stack de monitoring

## Vaultwarden, c'est quoi ?

Vaultwarden (anciennement Bitwarden_rs) est une implémentation communautaire du serveur Bitwarden, écrite en Rust. Avantages :

- **Léger** : ~50 Mo de RAM vs ~2 Go pour le serveur Bitwarden officiel
- **Compatible** : même clients (browser extension, mobile, desktop)
- **Fonctionnalités premium gratuites** : 2FA, attachments, organisations
- **Une seule binaire**, pas besoin de MSSQL + les 10 conteneurs officiels

## Déploiement Docker Compose

### Structure des fichiers

```
/opt/vaultwarden/
├── docker-compose.yml
├── .env
└── data/
    └── (persistent data)
```

### docker-compose.yml

```yaml
version: "3.8"

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    env_file: .env
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8080:80"    # Écoute en local uniquement
    networks:
      - vault-net

networks:
  vault-net:
    driver: bridge
```

### Variables d'environnement (.env)

```bash
# /opt/vaultwarden/.env

# Domaine (OBLIGATOIRE — doit correspondre au certificat TLS)
DOMAIN=https://vault.tondomaine.com

# Désactiver l'inscription publique (important !)
SIGNUPS_ALLOWED=false

# Désactiver l'invitation publique
INVITATIONS_ALLOWED=true

# 2FA obligatoire pour les connexions
REQUIRE_DEVICE_EMAIL=true

# Limites anti-brute force
FAILED_LOGIN_ATTEMPTS_RATE=5
DISABLE_ADMIN_TOKEN=false

# Token admin (changer immédiatement — générer avec : openssl rand -base64 48)
ADMIN_TOKEN=<ton-token-ici>

# SMTP pour les notifications et 2FA par email
SMTP_HOST=smtp.tondomaine.com
SMTP_FROM=vault@tondomaine.com
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_USERNAME=vault@tondomaine.com
SMTP_PASSWORD=<mot-de-passe-smtp>

# Logging
LOG_LEVEL=warn
EXTENDED_LOGGING=false

# WebSocket pour les sync temps réel
WEBSOCKET_ENABLED=true
```

## Reverse proxy Nginx

Vaultwarden ne doit **jamais** être exposé directement. Utilise un reverse proxy avec TLS.

```nginx
# /etc/nginx/sites-available/vault.tondomaine.com
server {
    listen 443 ssl http2;
    server_name vault.tondomaine.com;

    ssl_certificate /etc/letsencrypt/live/vault.tondomaine.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vault.tondomaine.com/privkey.pem;

    # Headers de sécurité
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;

    client_max_body_size 50M;  # Pour les attachments

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket pour les notifications temps réel
    location /notifications/hub {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# Redirection HTTP → HTTPS
server {
    listen 80;
    server_name vault.tondomaine.com;
    return 301 https://$host$request_uri;
}
```

Pour la configuration TLS complète, voir le guide [Reverse Proxy Nginx & TLS](/posts/2026-06-05-Reverse-Proxy-Nginx-TLS/).

## Accès distant sécurisé

Exposer Vaultwarden sur Internet, c'est un risque. Options par ordre de préférence :

1. **VPN uniquement** : accessible uniquement via [WireGuard](/posts/2026-06-19-WireGuard-VPN-SelfHosted/) — le plus sûr
2. **Cloudflare Tunnel** : pas besoin d'ouvrir de port, protection WAF incluse
3. **Exposition directe** via reverse proxy — fonctionnel mais plus exposé

```bash
# Si accès via WireGuard uniquement :
# Modifier le nginx pour écouter sur l'IP WireGuard
listen 10.0.0.1:443 ssl http2;
```

## Activation du 2FA

Vaultwarden supporte plusieurs méthodes 2FA :

| Méthode | Sécurité | Commodité |
|---|---|---|
| TOTP (Google Authenticator, Aegis) | Élevée | Bonne |
| WebAuthn / FIDO2 (YubiKey) | Très élevée | Excellente |
| Email | Moyenne | Bonne |
| Duo Security | Élevée | Bonne |

```bash
# Activer WebAuthn dans .env
WEBSOCKET_ENABLED=true

# Les utilisateurs activent le 2FA depuis :
# Settings → Security → Two-step login
```

Pour aller plus loin sur l'authentification moderne, voir [Passkeys & FIDO2 : la mort du mot de passe](/posts/2026-02-07-Passkeys-FIDO2-mort-mdp/).

## Stratégie de backup

Un gestionnaire de mots de passe sans backup, c'est une bombe à retardement.

### Backup automatique avec BorgBackup

```bash
#!/bin/bash
# /opt/vaultwarden/backup.sh

BACKUP_DIR="/backups/vaultwarden"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
VW_DIR="/opt/vaultwarden"

# Backup SQLite + attachments + config
mkdir -p "$BACKUP_DIR"
tar czf "$BACKUP_DIR/vaultwarden_${TIMESTAMP}.tar.gz" \
    -C "$VW_DIR" data/ .env docker-compose.yml

# Rotation : garder 30 jours
find "$BACKUP_DIR" -name "vaultwarden_*.tar.gz" -mtime +30 -delete

echo "[$(date)] Backup Vaultwarden terminé"
```

```bash
# Cron quotidien à 3h du matin
echo "0 3 * * * root /opt/vaultwarden/backup.sh" >> /etc/crontab
```

Pour une stratégie de backup complète et chiffrée, voir [BorgBackup : stratégie de backup](/posts/2026-03-13-BorgBackup-strategie-backup/).

## Maintenance

```bash
# Mise à jour
cd /opt/vaultwarden
docker compose pull
docker compose up -d

# Vérifier les logs
docker compose logs -f --tail=50

# Backup avant mise à jour
./backup.sh

# Healthcheck
curl -f http://127.0.0.1:8080/alive || echo "VAULTWARDEN DOWN"
```

## Endurcissement supplémentaire

- **Rate limiting** Nginx sur `/api/identity/connect/token` (anti brute force)
- **Fail2ban/CrowdSec** : jail personnalisée pour les échecs Vaultwarden
- **Monitoring** : exporter les métriques vers [Prometheus & Grafana](/posts/2026-06-15-Monitoring-Prometheus-Grafana/)
- **SSH durci** sur le serveur hôte : [Hardening SSH](/posts/2026-06-12-Hardening-SSH/)
- **Inscription désactivée** après le setup initial

```bash
# Rate limiting Nginx pour Vaultwarden
limit_req_zone $binary_remote_addr zone=vault_auth:10m rate=5r/m;

location /api/identity/connect/token {
    limit_req zone=vault_auth burst=3 nodelay;
    proxy_pass http://127.0.0.1:8080;
}
```

## Checklist

- [ ] Docker Compose déployé et fonctionnel
- [ ] `SIGNUPS_ALLOWED=false` après création des comptes
- [ ] Reverse proxy TLS configuré
- [ ] 2FA activé pour tous les utilisateurs
- [ ] Backup automatique en place avec rotation
- [ ] Accès restreint (VPN ou Cloudflare Tunnel recommandé)
- [ ] Rate limiting sur les endpoints d'authentification
- [ ] Monitoring de la santé du service
- [ ] `ADMIN_TOKEN` changé et stocké hors serveur
- [ ] Stratégie de récupération d'urgence documentée

---

*Références :*
- [Vaultwarden wiki](https://github.com/dani-garcia/vaultwarden/wiki)
- [Bitwarden clients](https://bitwarden.com/download/)
- [Docker Hub Vaultwarden](https://hub.docker.com/r/vaultwarden/server)

*Tes mots de passe sont sur un Google Sheet ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour migrer proprement.*
