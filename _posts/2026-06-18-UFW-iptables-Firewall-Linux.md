---
title: "UFW & iptables : firewall Linux en pratique"
date: 2026-06-18 10:00 +0200
categories: ["Sysops"]
tags: ["linux", "firewall", "ufw", "iptables", "sysops", "sécurité", "infrastructure", "hardening"]
author: marvax
---

> La plupart des serveurs compromis que j'ai vus avaient un point commun : pas de firewall, ou un firewall mal configuré. UFW rend ça trivial. Voici comment verrouiller ton serveur en 5 minutes.

<!--more-->

## UFW vs iptables

**iptables** est le firewall natif du noyau Linux. Puissant mais verbeux.

**UFW** (Uncomplicated Firewall) est un front-end simplifié pour iptables. Même résultat, 10x moins de syntaxe.

```bash
# Règle iptables classique
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Équivalent UFW
ufw allow 22/tcp
```

**Règle** : utilise UFW par défaut, passe à iptables directement uniquement si tu as besoin de règles avancées (rate limiting par IP, NAT, routing).

## Setup de base

```bash
# Installer UFW (souvent pré-installé)
sudo apt install ufw -y

# Politique par défaut : tout bloquer en entrée, tout autoriser en sortie
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser SSH AVANT d'activer (sinon tu te coupes l'accès)
sudo ufw allow 22/tcp

# Activer
sudo ufw enable

# Vérifier
sudo ufw status verbose
```

## Règles essentielles

```bash
# SSH (port custom)
sudo ufw allow 2222/tcp

# HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Limiter le SSH (rate limiting — max 6 connexions en 30s)
sudo ufw limit 2222/tcp

# Autoriser un sous-réseau spécifique
sudo ufw allow from 192.168.1.0/24 to any port 3306

# Autoriser une IP spécifique
sudo ufw allow from 203.0.113.42

# Bloquer une IP
sudo ufw deny from 198.51.100.0/24

# Autoriser un port range
sudo ufw allow 6000:6100/tcp

# Autoriser un service par nom
sudo ufw allow OpenSSH
```

## Règles par interface

```bash
# Autoriser le port 80 uniquement sur l'interface VPN
sudo ufw allow in on wg0 to any port 80

# Autoriser le SSH uniquement sur le LAN
sudo ufw allow in on eth0 from 192.168.1.0/24 to any port 22
```

## iptables directement (cas avancés)

```bash
# Rate limiting par IP (plus fin que ufw limit)
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP

# Loguer les paquets rejetés (debug)
iptables -A INPUT -j LOG --log-prefix "UFW-REJECT: " --log-level 4

# NAT (forward port 8080 → container Docker 172.17.0.2:80)
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
iptables -A FORWARD -p tcp -d 172.17.0.2 --dport 80 -j ACCEPT

# Anti-scan de ports
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL FIN,URG,PSH -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL SYN,RST,ACK,FIN,URG -j DROP
```

## Docker et UFW — le piège classique

**Problème** : Docker manipule iptables directement et contourne UFW. Les ports exposés via `-p 8080:80` sont accessibles même si UFW les bloque.

**Solution** :

```bash
# /etc/docker/daemon.json
{
  "iptables": false
}
```

Puis redémarrer Docker et gérer les règles manuellement via UFW. Ou utiliser `docker-proxy` avec des binds sur localhost uniquement :

```yaml
# docker-compose.yml
services:
  app:
    ports:
      - "127.0.0.1:8080:80"  # accessible uniquement en local
```

Ensuite Nginx reverse proxy vers `127.0.0.1:8080`.

## Checklist

- [ ] Politique deny incoming par défaut
- [ ] SSH autorisé avant activation
- [ ] Rate limiting sur SSH (`ufw limit`)
- [ ] Seuls les ports nécessaires sont ouverts
- [ ] Docker ne contourne pas UFW
- [ ] Logs activés pour les paquets rejetés
- [ ] Règles par interface si multi-NIC

## Monitoring

```bash
# Voir les règles actives
sudo ufw status numbered

# Voir les connexions actives
ss -tlnp

# Compter les connexions par IP
ss -tn | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head 10

# Logs UFW
sudo journalctl -u ufw -f
```

---

*Références :*
- [UFW man page](https://manpages.ubuntu.com/manpages/jammy/man8/ufw.8.html)
- [iptables man page](https://manpages.ubuntu.com/manpages/jammy/man8/iptables.8.html)

*Besoin d'un audit de la configuration réseau de tes serveurs ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
