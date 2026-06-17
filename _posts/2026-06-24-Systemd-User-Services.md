---
title: "Systemd user services : démoniser n'importe quoi"
date: 2026-06-24 10:00 +0200
categories: ["Dev"]
tags: ["systemd", "linux", "daemon", "service", "automatisation", "devops", "sysops"]
author: marvax
---

> Un script qui tourne dans un tmux oublié, un nohup qui crashe sans prévenir — on a tous fait. Systemd user services résout tout ça : auto-restart, logs structurés, démarrage au boot, le tout sans root.

<!--more-->

## System user vs system services

| | System (`/etc/systemd/system/`) | User (`~/.config/systemd/user/`) |
|---|---|---|
| Droits | root | ton utilisateur |
| Démarrage | au boot système | au login (ou linger) |
| Logs | `journalctl -u svc` | `journalctl --user -u svc` |
| Utile pour | nginx, docker, sshd | bots, dashboards, scripts perso |

**Règle** : si ça tourne pour toi et pas pour le système → user service.

## Créer un user service

```bash
mkdir -p ~/.config/systemd/user
```

`~/.config/systemd/user/mon-app.service` :

```ini
[Unit]
Description=Mon application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/user/mon-app
ExecStart=/home/user/mon-app/venv/bin/python app.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

# Environnement
Environment=PORT=8080
EnvironmentFile=/home/user/mon-app/.env

# Sécurité
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/home/user/mon-app/data

[Install]
WantedBy=default.target
```

## Commandes essentielles

```bash
# Recharger après modification
systemctl --user daemon-reload

# Démarrer
systemctl --user start mon-app

# Activer au login
systemctl --user enable mon-app

# Status
systemctl --user status mon-app

# Logs
journalctl --user -u mon-app -f

# Restart
systemctl --user restart mon-app

# Stop
systemctl --user stop mon-app
```

## Linger : démarrer au boot (sans login)

Par défaut, les user services s'arrêtent quand tu te déconnectes. `linger` les garde actifs :

```bash
sudo loginctl enable-linger $USER

# Vérifier
ls /var/lib/systemd/linger/
```

## Exemples concrets

### Bot Telegram

```ini
[Unit]
Description=Sentry Telegram Bot
After=network-online.target

[Service]
Type=simple
ExecStart=/home/marvax/venv/bin/python /home/marvax/bot.py
Restart=always
RestartSec=10
Environment=TELEGRAM_TOKEN=xxx

[Install]
WantedBy=default.target
```

### Serveur Flask

```ini
[Unit]
Description=Homelab Dashboard
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/marvax/Projects/homelab
ExecStart=/home/marvax/Projects/homelab/venv/bin/python -m flask run --host=0.0.0.0 --port=5557
Restart=on-failure
Environment=FLASK_APP=app.py
Environment=FLASK_ENV=production

[Install]
WantedBy=default.target
```

### Surveillance webcam

```ini
[Unit]
Description=Motion Detection Server
After=graphical-session.target

[Service]
Type=simple
ExecStart=/home/marvax/venv/bin/python /home/marvax/motion-server.py
Restart=on-failure
RestartSec=5
Environment=MOTION_CAMERA=0
Environment=MOTION_THRESHOLD=5000

[Install]
WantedBy=default.target
```

### Tâche périodique (timer)

```ini
# ~/.config/systemd/user/backup.timer
[Unit]
Description=Backup quotidien

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target

# ~/.config/systemd/user/backup.service
[Unit]
Description=Backup script

[Service]
Type=oneshot
ExecStart=/home/marvax/scripts/backup.sh
```

```bash
systemctl --user enable --now backup.timer
systemctl --user list-timers
```

## Sécurité des services

```ini
[Service]
# Empêcher l'élévation de privilèges
NoNewPrivileges=yes

# Filesystem en lecture seule
ProtectSystem=strict
ProtectHome=read-only

# Accès écriture uniquement où nécessaire
ReadWritePaths=/home/user/app/data

# Pas d'accès réseau inutile
# RestrictAddressFamilies=AF_INET AF_INET6

# Isoler le namespace
PrivateTmp=yes
```

## Gestion des crashes

```ini
[Service]
# Redémarrer automatiquement
Restart=on-failure
RestartSec=5

# Limiter les restarts (éviter les boucles)
StartLimitIntervalSec=300
StartLimitBurst=5

# Commande de health check
ExecStartPost=/bin/sleep 2
ExecStartPost=/bin/curl -f http://localhost:8080/health
```

## Monitoring

```bash
# Tous les services user actifs
systemctl --user list-units --type=service --state=running

# Logs en temps réel
journalctl --user -f

# Logs d'un service spécifique
journalctl --user -u mon-app --since "1 hour ago"

# Utilisation ressources
systemctl --user status mon-app | grep -E 'Memory|Tasks'

# Timers actifs
systemctl --user list-timers
```

## Checklist

- [ ] Service dans `~/.config/systemd/user/`
- [ ] `daemon-reload` après chaque modification
- [ ] `enable` pour démarrage automatique
- [ ] `linger` activé si besoin de tourner sans login
- [ ] `Restart=on-failure` pour la résilience
- [ ] Logs visibles via `journalctl --user`
- [ ] Sécurité : `NoNewPrivileges`, `ProtectSystem`

---

*Références :*
- [systemd.service man page](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [systemd.unit man page](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)

*Besoin d'aide pour démoniser tes applications ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
