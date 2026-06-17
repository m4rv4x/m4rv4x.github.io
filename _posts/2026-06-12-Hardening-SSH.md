---
title: "Hardening SSH : au-delà des basics"
date: 2026-06-12 10:00 +0200
categories: ["Sécurité"]
tags: ["ssh", "hardening", "linux", "sécurité", "sysops", "opsec", "infrastructure"]
author: marvax
---

> Changer le port SSH et désactiver root, tout le monde le fait. Mais ça ne suffit plus. Voici mon checklist complète pour verrouiller SSH comme un coffre-fort — avec les commandes exactes et les explications derrière chaque choix.

<!--more-->

## La baseline (si tu n'as même pas ça, arrête tout)

```bash
# /etc/ssh/sshd_config — minimum vital
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

```bash
sudo systemctl restart sshd
```

OK, tout le monde fait ça. Maintenant, le vrai hardening.

## 1. Clés Ed25519 uniquement

RSA 4096 c'est bien. Ed25519 c'est mieux — plus rapide, plus court, plus sûr.

```bash
# Générer une clé Ed25519
ssh-keygen -t ed25519 -C "marvax@$(hostname)" -f ~/.ssh/id_ed25519

# Sur le serveur, n'autoriser QUE Ed25519
# /etc/ssh/sshd_config
PubkeyAcceptedAlgorithms ssh-ed25519
HostKeyAlgorithms ssh-ed25519
```

## 2. Rejeter les algorithmes faibles

```bash
# /etc/ssh/sshd_config
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

Ça élimine les algos legacy (SHA1, CBC, etc.) qui ont des faiblesses connues.

## 3. Fail2ban sur SSH — mais bien configuré

```bash
sudo apt install fail2ban

# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
banaction = ufw
```

3 échecs → ban 1h. Via UFW, pas iptables directement.

## 4. Port knocking (le ninja move)

Le port SSH n'est même pas ouvert. Il faut "tapper" dans l'ordre sur des ports spécifiques pour l'ouvrir temporairement.

```bash
# Côté serveur — knockd
sudo apt install knockd

# /etc/knockd.conf
[options]
    logfile = /var/log/knockd.log

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 10
    command     = /usr/sbin/ufw allow 2222/tcp
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 10
    command     = /usr/sbin/ufw delete allow 2222/tcp
    tcpflags    = syn
```

```bash
# Côté client — pour se connecter
knock -v MON_IP 7000 8000 9000
ssh -p 2222 user@MON_IP
knock -v MON_IP 9000 8000 7000  # fermer après
```

## 5. Certificats SSH (pour les pros)

Au lieu de gérer des centaines de `authorized_keys`, utilise des certificats SSH signés par une CA :

```bash
# Créer la CA
ssh-keygen -t ed25519 -f ssh_ca -C "SSH CA Marvax"

# Signer une clé utilisateur (validité 8h)
ssh-keygen -s ssh_ca -I "marvax-laptop" -V +8h -n marvax ~/.ssh/id_ed25519.pub

# Sur le serveur, faire confiance à la CA
# /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/ssh_ca.pub
```

Avantage : tu peux révoquer l'accès en supprimant le certificat, sans toucher aux serveurs.

## 6. Restrictions par utilisateur

```bash
# /etc/ssh/sshd_config
Match User deploy
    ForceCommand /usr/local/bin/deploy.sh
    AllowTcpForwarding no
    X11Forwarding no
    PermitTunnel no
```

L'utilisateur `deploy` ne peut LANCER que le script de deploy. Rien d'autre.

## 7. Audit et monitoring

```bash
# Voir les tentatives de connexion en temps réel
journalctl -u sshd -f

# Qui s'est connecté ce mois ?
last -s "$(date +%Y-%m-01)"

# Scanner son propre serveur
ssh-audit MON_IP -p 2222
```

`ssh-audit` te dira exactement quels algorithmes sont proposés et lesquels sont faibles.

## Checklist récap

- [ ] Port non-standard
- [ ] Root login désactivé
- [ ] Password auth désactivé
- [ ] Clés Ed25519 uniquement
- [ ] Algorithmes faibles rejetés
- [ ] Fail2ban configuré
- [ ] Port knocking (optionnel mais classe)
- [ ] Certificats SSH (pour les équipes)
- [ ] Restrictions par utilisateur
- [ ] Audit régulier avec ssh-audit

## Conclusion

SSH est la porte d'entrée de ton infrastructure. Une porte mal verrouillée, c'est une invitation. Chaque couche de hardening ajoute un obstacle réel — pas cosmétique.

La plupart des compromissions que j'ai vues passaient par un SSH mal configuré. Pas par un zero-day. Par un `PasswordAuthentication yes` oublié.

---

*Besoin d'un audit SSH de ton infrastructure ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
