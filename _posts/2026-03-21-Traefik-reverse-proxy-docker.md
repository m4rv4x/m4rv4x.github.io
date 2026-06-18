---
title: "Traefik : le reverse proxy que tu mérites"
date: 2026-03-21 10:00 +0200
categories: ["Sysops"]
tags: ["traefik", "reverse-proxy", "docker", "sysops", "infrastructure", "devops", "self-hosted"]
author: marvax
---

> Tu tournes encore avec Nginx et des blocs `server` à rallonge ? Traefik est un reverse proxy conçu nativement pour Docker. Il découvre tes conteneurs automatiquement, génère ses certificats TLS tout seul, et te file un dashboard pour surveiller le tout. Plus de reload, plus de config à la main pour chaque nouveau service.

<!--more-->

## Pourquoi Traefik plutôt que Nginx ?

J'ai déjà couvert [Nginx comme reverse proxy](/posts/2026-06-05-Reverse-Proxy-Nginx-TLS/) sur ce blog. Nginx fait le taf, mais dans un environnement Docker, c'est de la plomberie. Chaque nouveau conteneur = modifier la config + recharger. Traefik élimine ça grâce à son **auto-discovery**.

| Critère | Nginx | Traefik |
|---|---|---|
| Auto-discovery Docker | ❌ Manuel | ✅ Natif |
| Certificats Let's Encrypt | Certbot séparé | ✅ Intégré |
| Hot reload | `nginx -s reload` | ✅ Automatique |
| Dashboard intégré | ❌ | ✅ |
| Middleware (rate limit, auth…) | Modules à compiler | ✅ Déclaratif |
| Courbe d'apprentissage | Simple | Moyenne |
| Performance brute | ✅ Légèrement supérieure | Très correcte |

Pour un setup purement statique ou un site web classique, Nginx reste un bon choix. Pour une infra Docker avec 10+ services qui tournent et changent, Traefik c'est le jour et la nuit.

## Architecture de Traefik

Traefik fonctionne en 3 composants :

1. **Providers** — Sources de configuration (Docker, fichier, Kubernetes…)
2. **Routers** — Règles de routage (`Host(\`app.example.com\`)`)
3. **Middlewares** — Traitement intermédiaire (auth, rate limiting, headers)

Le flow : requête entrante → router match la règle → middlewares s'appliquent → requête envoyée au service backend.

## Installation : Docker Compose de base

Voici un `docker-compose.yml` minimal pour Traefik avec Let's Encrypt automatique :

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v3.1
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./acme.json:/acme.json
    networks:
      - proxy

networks:
  proxy:
    external: true
```

Et le fichier `traefik.yml` :

```yaml
api:
  dashboard: true
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
    network: proxy

certificatesResolvers:
  letsencrypt:
    acme:
      email: ton-email@example.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web
```

Points clés :

- `exposedByDefault: false` — seuls les conteneurs avec le label `traefik.enable=true` sont exposés. C'est **indispensable** pour la [sécurité Docker](/posts/2026-06-08-Docker-Security-Erreurs/).
- Le socket Docker est monté en **read-only** (`:ro`).
- Le challenge HTTP se fait sur le port 80, puis redirection vers 443.

Crée le fichier ACME et le réseau :

```bash
touch acme.json
chmod 600 acme.json
docker network create proxy
```

## Exposer un service : les labels Docker

C'est là que Traefik brille. Pas de fichier de config à modifier. Tu mets des labels sur ton conteneur :

```yaml
services:
  whoami:
    image: traefik/whoami
    container_name: whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
    networks:
      - proxy
```

Démarre le conteneur → Traefik détecte le label → crée le routeur → génère le certificat TLS → le service est accessible sur `https://whoami.example.com`. Zéro reload.

## Middlewares : rate limiting, auth, headers

### Rate limiting

```yaml
labels:
  - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
  - "traefik.http.middlewares.ratelimit.ratelimit.burst=50"
  - "traefik.http.routers.app.middlewares=ratelimit@docker"
```

Tu peux aussi combiner ça avec [CrowdSec](/posts/2026-01-16-CrowdSec-fail2ban-futur/) pour un blocage adaptatif basé sur la communauté.

### Basic Auth

```bash
# Générer le hash
echo $(htpasswd -nB utilisateur) | sed -e 's/\$/\$\$/g'
```

```yaml
labels:
  - "traefik.http.middlewares.auth.basicauth.users=utilisateur:$$2y$$05$$..."
  - "traefik.http.routers.dashboard.middlewares=auth@docker"
```

### Headers de sécurité

```yaml
labels:
  - "traefik.http.middlewares.headers.headers.stsSeconds=31536000"
  - "traefik.http.middlewares.headers.headers.stsIncludeSubdomains=true"
  - "traefik.http.middlewares.headers.headers.browserXssFilter=true"
  - "traefik.http.middlewares.headers.headers.contentTypeNosniff=true"
  - "traefik.http.middlewares.headers.headers.frameDeny=true"
  - "traefik.http.routers.app.middlewares=headers@docker"
```

Ces headers complètent la [config TLS de Nginx](/posts/2026-06-05-Reverse-Proxy-Nginx-TLS/) que tu peux migrer intégralement vers Traefik.

## Dashboard sécurisé

Le dashboard Traefik expose des infos sensibles. Ne le laisse pas ouvert :

```yaml
services:
  traefik:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.middlewares=auth@docker"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$2y$$..."
```

**Protège-le derrière un VPN** comme [WireGuard](/posts/2026-06-19-WireGuard-VPN-SelfHosted/) ou un [Cloudflare Tunnel](/posts/2026-02-14-Cloudflare-Tunnel-zero-port/) sans exposition de port. Et débloque les ports sur ton [UFW](/posts/2026-06-18-UFW-iptables-Firewall-Linux/).

## Traefik + CI/CD

Dans un pipeline [CI/CD avec GitHub Actions](/posts/2026-06-20-CICD-GitHub-Actions/), tu déploies ton conteneur et Traefik prend le relais automatiquement. Plus besoin de script de rechargement Nginx dans ton pipeline :

```bash
# Dans ton pipeline de déploiement
docker compose pull
docker compose up -d
# Traefik détecte le nouveau conteneur → c'est tout
```

## Checklist de mise en production

- [ ] `exposedByDefault: false` dans la config Traefik
- [ ] Socket Docker monté en read-only
- [ ] Dashboard protégé par auth + VPN
- [ ] Rate limiting sur les endpoints publics
- [ ] Headers de sécurité activés
- [ ] Certificats Let's Encrypt configurés
- [ ] Monitoring via [Prometheus/Grafana](/posts/2026-06-15-Monitoring-Prometheus-Grafana/)
- [ ] Logs Traefik envoyés dans ta stack de monitoring
- [ ] UFW configuré (80, 443, SSH uniquement)

## Quand garder Nginx ?

- Tu n'utilises pas Docker → Nginx est plus simple
- Tu as besoin de performance brute (CDN, cache statique massif) → Nginx est plus léger
- Tu as une config Nginx qui fonctionne et pas le temps de migrer → si c'est pas cassé, répare pas

Pour tout le reste, Traefik est supérieur dans un contexte Docker. C'est pas un choix idéologique, c'est pragmatique.

---

*Références :*

- [Documentation officielle Traefik](https://doc.traefik.io/traefik/)
- [Reverse Proxy Nginx](/posts/2026-06-05-Reverse-Proxy-Nginx-TLS/)
- [Sécurité Docker](/posts/2026-06-08-Docker-Security-Erreurs/)
- [CrowdSec](/posts/2026-01-16-CrowdSec-fail2ban-futur/)
- [Cloudflare Tunnel](/posts/2026-02-14-Cloudflare-Tunnel-zero-port/)
- [UFW / iptables](/posts/2026-06-18-UFW-iptables-Firewall-Linux/)
- [CI/CD GitHub Actions](/posts/2026-06-20-CICD-GitHub-Actions/)

*Besoin d'un audit de ton infrastructure Docker ou d'une migration Nginx → Traefik ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
