---
title: "Cloudflare Tunnel : expose tes services sans ouvrir un port"
date: 2026-02-14 10:00 +0200
categories: ["Sysops"]
tags: ["cloudflare", "tunnel", "zero-trust", "self-hosted", "sécurité", "sysops", "infrastructure"]
author: marvax
---

> Tu veux exposer un service self-hosted au monde sans ouvrir un seul port sur ton routeur ? Cloudflare Tunnel (cloudflared) crée un tunnel chiffré outbound depuis ta machine vers l'edge Cloudflare. Zéro port ouvert, zéro IP exposée, HTTPS automatique. C'est la méthode la plus propre pour exposer tes services depuis un homelab ou un VPS derrière un CGNAT.

<!--more-->

## Pourquoi Cloudflare Tunnel ?

Le modèle classique : ouvrir le port 443, pointer un reverse proxy, gérer les certificats TLS. Ça marche, mais ça implique d'exposer ton IP publique et de gérer la surface d'attaque ([Reverse Proxy Nginx](/posts/Reverse-Proxy-Nginx-TLS/)).

Cloudflare Tunnel inverse le problème : ta machine initie la connexion vers Cloudflare (outbound). Aucun port entrant n'est nécessaire. Avantages concrets :

| Modèle | Port ouvert | IP exposée | TLS | CGNAT compatible |
|---|---|---|---|---|
| Reverse proxy classique | Oui (443) | Oui | Manuel/certbot | Non |
| WireGuard + reverse proxy | Oui (51820) | Oui | Via proxy | Oui (si port ouvert) |
| Cloudflare Tunnel | **Aucun** | **Non** | **Automatique** | **Oui** |

Pour le contexte, compare avec l'approche [WireGuard](/posts/WireGuard-VPN-SelfHosted/) qui nécessite au moins un port UDP ouvert.

## Prérequis

- Un nom de domaine géré par Cloudflare (gratuit suffit)
- Un compte Cloudflare
- Une machine locale ou un VPS avec accès root

## Installation de cloudflared

```bash
# Debian/Ubuntu
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
```

```bash
# Vérifier la version
cloudflared --version
```

## Authentification

```bash
cloudflared tunnel login
```

Ça ouvre un navigateur. Tu choisis le domaine à utiliser. Un certificat est stocké dans `~/.cloudflare/cert.pem`.

## Créer le tunnel

```bash
# Créer le tunnel
cloudflared tunnel create mon-tunnel

# Vérifier
cloudflared tunnel list
```

Note l'UUID du tunnel. Il sera nécessaire pour la suite.

## Configuration

Crée le fichier `~/.cloudflare/config.yml` :

```yaml
tunnel: <UUID-de-ton-tunnel>
credentials-file: /root/.cloudflare/<UUID>.json

ingress:
  - hostname: vault.example.com
    service: http://localhost:8080
  - hostname: grafana.example.com
    service: http://localhost:3000
  - hostname: git.example.com
    service: http://localhost:3001
  - service: http_status:404
```

La dernière règle (404) est obligatoire — c'est le catch-all.

## Associer le tunnel au DNS

```bash
cloudflared tunnel route dns mon-tunnel vault.example.com
cloudflared tunnel route dns mon-tunnel grafana.example.com
cloudflared tunnel route dns mon-tunnel git.example.com
```

Cloudflare crée automatiquement les enregistrements CNAME.

## Lancer le tunnel

```bash
# Test direct
cloudflared tunnel run mon-tunnel
```

Pour un service permanent, crée un service systemd :

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Vérifie que tout fonctionne :

```bash
sudo systemctl status cloudflared
curl -I https://vault.example.com
```

## Docker Compose

Si tes services tournent déjà dans Docker ([Sécurité Docker](/posts/Docker-Security-Erreurs/)) :

```yaml
version: "3.8"

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=<ton-token>
    networks:
      - internal

  vaultwarden:
    image: vaultwarden/server:latest
    restart: unless-stopped
    volumes:
      - ./vw-data:/data
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

Avec cette approche, `cloudflared` accède à `vaultwarden:80` via le réseau Docker interne. Aucun port n'est publié sur l'hôte.

Pour obtenir le token :

```bash
cloudflared tunnel token create mon-tunnel
```

## Politiques d'accès Zero Trust

Cloudflare Access permet de protéger tes services avec une couche d'authentification supplémentaire. Dans le dashboard Cloudflare > Zero Trust > Access > Applications :

```yaml
# Exemple de politique : accès restreint à ton email
policy:
  name: "Admin only"
  decision: allow
  include:
    - email:
        value: "ton-email@example.com"
```

Options d'authentification supportées :
- Email (one-time pin)
- GitHub, Google, OIDC
- Clés d'accès matériel (passkeys — voir [Passkeys & FIDO2](/posts/Passkeys-FIDO2-mort-mdp/))
- Service tokens (pour les appels API machine-to-machine)

## Tunnel vs Reverse Proxy classique

| Critère | Cloudflare Tunnel | Nginx + Let's Encrypt |
|---|---|---|
| Port ouvert | Aucun | 443 minimum |
| IP publique exposée | Non | Oui |
| CGNAT/NAT strict | Fonctionne | Bloqué |
| Certificats TLS | Automatique | Certbot/cron |
| Dépendance Cloudflare | Oui | Non |
| Latence ajoutée | ~5-20ms (edge CF) | Aucune |
| Gratuité | Plan gratuit suffisant | Gratuit |
| Contrôle total | Limité (Cloudflare edge) | Total |

Le revers de la médaille : tu dépends de Cloudflare. Si leur edge est down, tes services sont inaccessibles. Pour un homelab critique, combine avec [WireGuard](/posts/WireGuard-VPN-SelfHosted/) comme accès de secours.

## Durcissement

Même sans port ouvert, les bonnes pratiques s'appliquent :

- **UFW** : laisse le firewall actif pour bloquer les scans locaux ([UFW & iptables](/posts/UFW-iptables-Firewall-Linux/))
- **Fail2ban** : protège les services exposés contre le brute-force ([Fail2ban avancé](/posts/Fail2ban-avance/))
- **Docker security** : ne publie aucun port sur l'hôte, utilise les réseaux internes
- **Monitoring** : surveille les logs de cloudflared dans Grafana ([Monitoring Prometheus & Grafana](/posts/Monitoring-Prometheus-Grafana/))

## Checklist de déploiement

- [ ] Domaine ajouté à Cloudflare
- [ ] cloudflared installé et authentifié
- [ ] Tunnel créé et configuré
- [ ] Enregistrements DNS routés
- [ ] Service systemd ou Docker actif
- [ ] Accès HTTPS fonctionnel
- [ ] Politique Access configurée (si exposé publiquement)
- [ ] Logs monitorés
- [ ] UFW actif malgré l'absence de port ouvert
- [ ] Backup de la config tunnel (`config.yml` + credentials)

*Références :*
- [Documentation Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [WireGuard VPN Self-Hosted](/posts/WireGuard-VPN-SelfHosted/)
- [Reverse Proxy Nginx & TLS](/posts/Reverse-Proxy-Nginx-TLS/)
- [Sécurité Docker](/posts/Docker-Security-Erreurs/)
- [UFW & iptables](/posts/UFW-iptables-Firewall-Linux/)
- [Vaultwarden self-hosted](/posts/Vaultwarden-self-hosted-mdp/)
- [Passkeys & FIDO2](/posts/Passkeys-FIDO2-mort-mdp/)
- [Monitoring Prometheus & Grafana](/posts/Monitoring-Prometheus-Grafana/)

Besoin d'un audit de ton infrastructure ou d'un coup de main sur le déploiement ? [Contacte-moi](mailto:m4rv4x@protonmail.com).
