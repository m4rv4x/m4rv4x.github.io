---
title: "BorgBackup : stratégie de backup 3-2-1 pour homelab"
date: 2026-03-13 10:00 +0200
categories: ["Sysops"]
tags: ["borgbackup", "backup", "sysops", "infrastructure", "linux", "self-hosted"]
author: marvax
---

> Le ransomware qui chiffre tes données, le disque qui lâche, la commande `rm -rf` mal placée — c'est jamais le bon moment. BorgBackup est l'outil de backup le plus sous-estimé de l'écosystème Linux : déduplication, chiffrement natif, compression, snapshots incrémentaux. Cet article te montre comment implémenter une stratégie 3-2-1 complète, de l'installation à l'automatisation, en passant par la restauration. Parce qu'un backup non testé, c'est pas un backup.

<!--more-->

## La règle 3-2-1

C'est simple et ça sauve des infrastructures :

- **3** copies de tes données
- **2** supports de stockage différents
- **1** copie offsite (hors site)

Concrètement pour un homelab :

| Copie | Support | Localisation |
|---|---|---|
| 1 — Données originales | SSD/NVMe du serveur | Chez toi |
| 2 — Backup local | Disque dur externe / NAS | Chez toi |
| 3 — Backup distant | VPS / serveur SSH | Datacenter |

Si ton lab prend feu ou se fait voler, la copie offsite te sauve. Si ton VPS est compromise, la copie locale te sauve.

## Pourquoi BorgBackup ?

| Outil | Dédup | Chiffrement | Incrémental | SSH natif | Licence |
|---|---|---|---|---|---|
| BorgBackup | Oui | Oui | Oui | Oui | BSD |
| rsync | Non | Via SSH | Partiel | Oui | GPL |
| Restic | Oui | Oui | Oui | Oui | BSD |
| Duplicity | Non | Oui | Oui | Oui | GPL |

Borg est le plus ancien et le plus stable de la catégorie "dédié + dédup". Restic est un excellent concurrent (notamment pour S3), mais Borg reste le roi du SSH-to-SSH pour homelab.

## Installation

```bash
# Debian/Ubuntu
sudo apt install -y borgbackup

# Arch
sudo pacman -S borg

# Vérifier
borg --version
```

## Initialiser un repo local

```bash
# Créer le répertoire de backup
sudo mkdir -p /mnt/backup/borg
sudo chown $USER:$USER /mnt/backup/borg

# Initialiser le repo avec chiffrement
borg init --encryption=repokey-blake2 /mnt/backup/borg/monserveur
```

Tu seras demandé de définir une passphrase. **Stocke-la dans ton gestionnaire de mots de passe** ([Vaultwarden](/posts/Vaultwarden-self-hosted-mdp/)). Sans cette passphrase, tes backups sont inaccessibles.

## Premier backup

```bash
borg create \
  --verbose \
  --filter AME \
  --list \
  --stats \
  --compression auto,zstd \
  /mnt/backup/borg/monserveur::{now} \
  /home \
  /etc \
  /var/lib/docker/volumes \
  --exclude '/home/*/.cache' \
  --exclude '/home/*/.local/share/Trash' \
  --exclude '*.tmp'
```

Options importantes :

| Flag | Rôle |
|---|---|
| `--compression auto,zstd` | Compression zstd intelligente |
| `--exclude` | Exclut les fichiers inutiles |
| `--stats` | Affique la taille et le ratio de dédup |
| `{now}` | Timestamp automatique comme nom d'archive |

La déduplication signifie que même avec des snapshots quotidiens, l'espace disque reste contenu. Seuls les blocs modifiés sont stockés.

## Repo distant (SSH)

Pour la copie offsite :

```bash
# Initialiser un repo distant
borg init --encryption=repokey-blake2 ssh://user@vps.example.com/~/borg/monserveur
```

```bash
# Backup vers le distant
borg create \
  --compression auto,zstd \
  ssh://user@vps.example.com/~/borg/monserveur::{now} \
  /home /etc /var/lib/docker/volumes \
  --exclude '*.cache'
```

**Important** : configure l'accès SSH avec une clé dédiée, sans passphrase, limitée au backup uniquement. Sur le serveur distant, dans `~/.ssh/authorized_keys` :

```
command="borg serve --restrict-to-path /home/user/borg",no-pty,no-agent-forwarding,no-port-forwarding,no-X11-forwarding,no-user-rc ssh-ed25519 AAAA...
```

Voir [Hardening SSH](/posts/Hardening-SSH/) pour la config complète. Si tu veux chiffrer le transport en plus, passe par [WireGuard](/posts/WireGuard-VPN-SelfHosted/).

## Automatisation avec systemd timer

Crée le service de backup `/etc/systemd/system/borg-backup.service` :

```ini
[Unit]
Description=BorgBackup quotidien
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/borg-backup.sh
Nice=19
IOSchedulingClass=idle
```

Le script `/usr/local/bin/borg-backup.sh` :

```bash
#!/bin/bash
export BORG_REPO="/mnt/backup/borg/monserveur"
export BORG_PASSPHRASE="voir-vaultwarden"

# Backup
borg create --compression auto,zstd \
  ::{now} \
  /home /etc /var/lib/docker/volumes \
  --exclude '*.cache'

# Pruning — garder 7 quotidiens, 4 hebdo, 6 mensuels
borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6

# Vérification intégrité (1x/semaine)
if [ $(date +%u) -eq 7 ]; then
  borg check
fi

# Compaction
borg compact
```

Le timer `/etc/systemd/system/borg-backup.timer` :

```ini
[Unit]
Description=BorgBackup quotidien à 3h

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=900

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now borg-backup.timer
sudo systemctl list-timers | grep borg
```

Alternative : un simple cron si tu préfères :

```bash
# crontab -e
0 3 * * * /usr/local/bin/borg-backup.sh 2>&1 | logger -t borg-backup
```

## Automatisation avec Ansible

Si tu gères plusieurs serveurs ([Automatiser avec Ansible](/posts/Automatiser-Infra-Ansible/)) :

```yaml
- name: Installer BorgBackup
  apt:
    name: borgbackup
    state: present

- name: Déployer le script de backup
  template:
    src: borg-backup.sh.j2
    dest: /usr/local/bin/borg-backup.sh
    mode: '0700'

- name: Activer le timer systemd
  systemd:
    name: borg-backup.timer
    enabled: true
    state: started
```

## Restauration

Parce qu'un backup non testé n'existe pas :

```bash
# Lister les archives
borg list /mnt/backup/borg/monserveur

# Restaurer une archive complète
cd /tmp/restore
borg extract /mnt/backup/borg/monserveur::2026-07-02T03:00:00

# Restaurer un fichier spécifique
borg extract /mnt/backup/borg/monserveur::2026-07-02T03:00:00 etc/nginx/nginx.conf

# Monter le repo comme un filesystem
borg mount /mnt/backup/borg/monserveur /mnt/borg
ls /mnt/borg/
borg umount /mnt/borg
```

**Teste ta restauration régulièrement.** Un backup jamais restauré, c'est un pari.

## Monitoring des backups

Intègre le suivi dans ta stack de monitoring ([Monitoring Prometheus & Grafana](/posts/Monitoring-Prometheus-Grafana/)) :

```bash
# Script de vérification envoyant des métriques
borg info --json /mnt/backup/borg/monserveur | \
  jq '.archives | length' | \
  curl -X POST --data-binary @- http://pushgateway:9091/metrics/job/borg/instance/monserveur
```

Alerte si aucun backup n'a été créé dans les dernières 24h.

## Checklist complète

- [ ] BorgBackup installé sur toutes les machines
- [ ] Repo local initialisé avec chiffrement
- [ ] Repo distant initialisé sur VPS
- [ ] Passphrase stockée dans Vaultwarden
- [ ] Clé SSH dédiée avec restrictions
- [ ] Script de backup déployé et testé
- [ ] Timer systemd ou cron actif
- [ ] Politique de pruning configurée (règle 3-2-1)
- [ ] Restauration testée et documentée
- [ ] Monitoring des backups actif
- [ ] Backup de la config Borg lui-même (config + clé repo)

## Erreurs courantes

- **Pas de chiffrement** : `borg init` sans `--encryption` = données en clair sur le backup
- **Passphrase perdue** : sans elle, pas de restauration. Stocke-la ailleurs que sur la machine backuppée
- **Backup jamais testé** : teste la restauration au moins une fois par mois
- **Pas de pruning** : le repo grandit indéfiniment. Configure le pruning dès le début
- **Backup sur le même disque** : si le disque lâche, tout est perdu. Utilise un support séparé

*Références :*
- [BorgBackup Documentation](https://borgbackup.readthedocs.io/)
- [Hardening SSH](/posts/Hardening-SSH/)
- [WireGuard VPN Self-Hosted](/posts/WireGuard-VPN-SelfHosted/)
- [Automatiser avec Ansible](/posts/Automatiser-Infra-Ansible/)
- [Monitoring Prometheus & Grafana](/posts/Monitoring-Prometheus-Grafana/)
- [Vaultwarden self-hosted](/posts/Vaultwarden-self-hosted-mdp/)
- [Cloudflare Tunnel](/posts/Cloudflare-Tunnel-zero-port/)
- [Incident Response Playbook](/posts/Incident-Response-Playbook/)

Besoin d'un audit de ta stratégie de backup ou d'un coup de main sur BorgBackup ? [Contacte-moi](mailto:m4rv4x@protonmail.com).
