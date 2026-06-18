---
title: "Fail2ban avancé : au-delà du basique"
date: 2026-06-15 10:00 +0200
categories: ["Sécurité"]
tags: ["fail2ban", "linux", "sécurité", "sysops", "hardening", "infrastructure", "opsec"]
author: marvax
---

> Fail2ban c'est pas juste `apt install fail2ban` et oublier. Avec les bonnes configs, ça devient un vrai système de détection d'intrusion. Voici comment l'utiliser au-delà du basique.

<!--more-->

## Ce que Fail2ban fait vraiment

Fail2ban lit les logs, détecte les patterns d'attaque (tentatives échouées), et bloque les IP via le firewall. C'est un IDS léger basé sur les logs.

```
Log file → Filter (regex) → Pattern match → Ban action (iptables/nftables/ufw)
```

## Setup robuste

```bash
sudo apt install fail2ban -y

# Config locale (NE PAS modifier jail.conf)
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
# Ban de 1h après 3 échecs en 10 min
bantime = 3600
findtime = 600
maxretry = 3

# Action par défaut : UFW + email
banaction = ufw
action = %(action_mwl)s

# Ignorer les IPs locales
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24

# Backend auto-detect
backend = systemd

[sshd]
enabled = true
port = 2222
filter = sshd
maxretry = 3
bantime = 7200
EOF

sudo systemctl enable --now fail2ban
```

## Jails personnalisées

### Protection Nginx

```ini
# /etc/fail2ban/jail.local — ajouter

[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
maxretry = 3
bantime = 3600

[nginx-botsearch]
enabled = true
port = http,https
filter = nginx-botsearch
maxretry = 2
bantime = 86400

[nginx-limit-req]
enabled = true
port = http,https
filter = nginx-limit-req
maxretry = 5
bantime = 7200
```

### Protection WordPress (si applicable)

```ini
[wordpress]
enabled = true
port = http,https
filter = wordpress
maxretry = 3
bantime = 14400
```

### Blocage des scanners de ports

```ini
[portscan]
enabled = true
filter = portscan
action = %(action_)s
logpath = /var/log/ufw.log
maxretry = 3
bantime = 86400
findtime = 300
```

## Filtres personnalisés

Crée des filtres dans `/etc/fail2ban/filter.d/` :

```ini
# /etc/fail2ban/filter.d/custom-api-abuse.conf
[Definition]
failregex = ^<HOST> .* "(GET|POST) /api/ .*" (401|403) .*$
            ^<HOST> .* "(GET|POST) /admin .*" (401|403|404) .*$
ignoreregex =
```

```ini
# /etc/fail2ban/filter.d/portscan.conf
[Definition]
failregex = \[UFW BLOCK\].*SRC=<HOST>.*
ignoreregex = \[UFW BLOCK\].*DST=.*DPT=(22|80|443).*
```

## Actions avancées

### Ban multi-port

```ini
[DEFAULT]
banaction = ufw
# Ban sur tous les ports (pas juste celui de l'attaque)
action = %(action_)s
```

### Ban + notification Telegram

```ini
# /etc/fail2ban/action.d/telegram.conf
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = curl -s -X POST "https://api.telegram.org/bot<token>/sendMessage" \
            -d "chat_id=<chat_id>" \
            -d "text=🚫 Fail2ban: <ip> banned on <name> (<failures> failures)"
actionunban = curl -s -X POST "https://api.telegram.org/bot<token>/sendMessage" \
              -d "chat_id=<chat_id>" \
              -d "text=✅ Fail2ban: <ip> unbanned on <name>"
```

```ini
# Utilisation dans jail.local
[sshd]
action = ufw
         telegram[token=BOT_TOKEN, chat_id=CHAT_ID]
```

### Ban progressif (escalade)

```ini
# /etc/fail2ban/action.d/progressive-ban.conf
[Definition]
actionban = COUNT=$(cat /tmp/fail2ban-<ip> 2>/dev/null || echo 0)
            COUNT=$((COUNT + 1))
            echo $COUNT > /tmp/fail2ban-<ip>
            if [ $COUNT -ge 5 ]; then
              ufw insert 1 deny from <ip> to any comment "permanent-ban"
            else
              ufw insert 1 deny from <ip> to any comment "ban-<name>-attempt-$COUNT"
            fi
```

## Recidive — ban permanent

```ini
# /etc/fail2ban/jail.local
[recidive]
enabled = true
filter = recidive
bantime = 604800  # 7 jours
findtime = 86400  # si banni 2x en 24h
maxretry = 3
action = ufw
```

## Monitoring

```bash
# Status complet
sudo fail2ban-client status

# Status d'une jail spécifique
sudo fail2ban-client status sshd

# IP bannies actuellement
sudo fail2ban-client banned

# Voir les bannissements en temps réel
sudo journalctl -u fail2ban -f

# Stats par jail
sudo fail2ban-client status sshd | grep "Currently banned"

# Logs détaillés
sudo tail -f /var/log/fail2ban.log
```

## Debug

```bash
# Tester un filtre sur un fichier de log
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf

# Voir les IP bannies par UFW via fail2ban
sudo ufw status | grep -i fail2ban

# Débannir une IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Forcer le ban d'une IP
sudo fail2ban-client set sshd banip 198.51.100.42
```

## Checklist

- [ ] `jail.local` configuré (pas `jail.conf`)
- [ ] SSH protégé avec maxretry ≤ 3
- [ ] Nginx protégé (auth, botsearch, limit-req)
- [ ] Filtres personnalisés pour les endpoints sensibles
- [ ] Notifications activées (email ou Telegram)
- [ ] Recidive activé (ban progressif)
- [ ] ignoreip configuré (LAN, localhost)
- [ ] Monitoring régulier des bans

---

*Références :*
- [Fail2ban documentation](https://fail2ban.readthedocs.io/)
- [Filter patterns](https://github.com/fail2ban/fail2ban/tree/master/config/filter.d)

*Ton Fail2ban tourne avec la config par défaut ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour un audit.*
