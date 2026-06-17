---
title: "Reverse proxy Nginx avec TLS : le guide complet"
date: 2026-06-05 10:00 +0200
categories: ["Sysops"]
tags: ["nginx", "tls", "reverse-proxy", "letsencrypt", "sysops", "sécurité", "infrastructure", "web"]
author: marvax
---

> Tu exposes des services web sans reverse proxy ? C'est comme ouvrir ta porte d'entrée et coller un panneau "servez-vous". Nginx en frontal avec TLS, c'est la base absolue. Voici comment mettre ça en place proprement — de zéro au A+ sur SSL Labs.

<!--more-->

## C'est quoi un reverse proxy ?

```
Client (navigateur)
       │
       │ HTTPS (TLS)
       ▼
┌──────────────────┐
│  Nginx           │  ← reverse proxy
│  (port 443)      │
└──────┬───────────┘
       │ HTTP local
       ├──→ App Node.js  :3000
       ├──→ Grafana      :3001
       ├──→ Gitea        :3002
       └──→ Uptime Kuma  :3003
```

Le client ne voit que Nginx. Les services derrière ne sont jamais exposés directement. TLS s'arrête au proxy.

## Étape 1 — Installation

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y
sudo systemctl enable nginx
```

## Étape 2 — Configuration de base

`/etc/nginx/sites-available/app.example.com` :

```nginx
server {
    listen 80;
    server_name app.example.com;

    # Redirection vers HTTPS (après obtention du cert)
    location / {
        return 301 https://$host$request_uri;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/app.example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## Étape 3 — Certificat TLS (Let's Encrypt)

```bash
sudo certbot --nginx -d app.example.com
```

Certbot modifie automatiquement la config Nginx pour ajouter le TLS. Le certificat est valide 90 jours et se renouvelle automatiquement via un timer systemd.

Vérifie le renouvellement :
```bash
sudo certbot renew --dry-run
```

## Étape 4 — Config TLS renforcée

Après Certbot, améliore la config TLS :

```nginx
server {
    listen 443 ssl http2;
    server_name app.example.com;

    # Certificats Let's Encrypt
    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    # TLS moderne (TLS 1.2 + 1.3 uniquement)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/app.example.com/chain.pem;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # Session caching
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # Headers de sécurité
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Proxy vers l'app backend
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (si nécessaire)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Étape 5 — Multi-services avec un seul Nginx

```nginx
# Grafana
server {
    listen 443 ssl http2;
    server_name grafana.example.com;

    # ... (mêmes blocs TLS) ...

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Un bloc `server` par sous-domaine. Chaque certificat Let's Encrypt couvre un domaine.

## Étape 6 — Générer les DH params

```bash
sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

Puis ajouter dans le bloc `server` :
```nginx
ssl_dhparam /etc/nginx/dhparam.pem;
```

⚠️ Ça prend quelques minutes. Mais ça renforce significativement l'échange de clés.

## Étape 7 — Tester

```bash
# Test de config
sudo nginx -t

# Reload
sudo systemctl reload nginx

# Test TLS externe
curl -I https://app.example.com

# Note SSL Labs (visite https://www.ssllabs.com/ssltest/)
# Objectif : A+
```

## Optimisations performance

```nginx
# /etc/nginx/nginx.conf — bloc http
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

# Cache pour les assets statiques
location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff2)$ {
    proxy_pass http://127.0.0.1:3000;
    expires 30d;
    add_header Cache-Control "public, immutable";
}

# Buffer tuning
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 4k;
```

## Checklist de mise en production

- [ ] HTTPS forcé (redirect 301 depuis HTTP)
- [ ] TLS 1.2+ uniquement
- [ ] HSTS activé
- [ ] Headers de sécurité présents
- [ ] Certificat auto-renouvelé (certbot renew)
- [ ] DH params générés
- [ ] Note A/A+ sur SSL Labs
- [ ] Rate limiting sur les endpoints sensibles
- [ ] Logs d'accès activés et rotés
- [ ] Monitoring du certificat (expiration)

## Vulnérabilité récente — CVE-2026-42945

En mai 2026, une faille vieille de **18 ans** a été découverte dans le module `ngx_http_rewrite_module` de Nginx. Un heap buffer overflow déclenché par un `rewrite` contenant un `?` dans la chaîne de remplacement.

| | |
|---|---|
| **CVE** | CVE-2026-42945 |
| **CVSS** | 9.2 CRITICAL (v4.0) / 8.1 HIGH (v3.1) |
| **Affecte** | Nginx 0.6.27 → 1.30.0, NGINX Plus R32–R36 |
| **Impact** | DoS (crash worker) trivial, RCE démontré sans ASLR |
| **Fix** | Nginx 1.31.0 / 1.30.1, NGINX Plus R36 P4 |

Le bug vient d'un flag `is_args` qui reste actif après un rewrite contenant `?`, causant un décalage entre le calcul de taille du buffer (non-échappé) et les données écrites (échappées). L'architecture multi-process de Nginx facilite l'exploitation car les workers héritent de layouts mémoire identiques.

**Leçon** : même un serveur aussi mature que Nginx peut contenir des bugs critiques depuis des années. Mettre à jour Nginx fait partie des patches prioritaires, au même titre que le noyau ou OpenSSL.

Références :
- [NVD — CVE-2026-42945](https://nvd.nist.gov/vuln/detail/CVE-2026-42945)
- [BleepingComputer — 18-year-old nginx vulnerability](https://www.bleepingcomputer.com/news/security/18-year-old-nginx-vulnerability-allows-dos-potential-rce/)
- [GitHub — DepthFirstDisclosures/Nginx-Rift](https://github.com/DepthFirstDisclosures/Nginx-Rift)

## Conclusion

Un reverse proxy Nginx avec TLS bien configuré, c'est 15 minutes de boulot pour un gain de sécurité énorme. C'est aussi ce qui te permet d'exposer proprement plusieurs services derrière une seule IP, avec des certificats automatiques.

Si tu self-hostes quoi que ce soit, c'est le premier truc à mettre en place. Avant même Docker.

---

*Besoin d'aide pour mettre en place ton infrastructure web ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
