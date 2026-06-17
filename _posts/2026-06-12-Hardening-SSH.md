---
title: "Hardening SSH : au-delà des basics"
date: 2026-06-12 10:00 +0200
categories: ["Sécurité"]
tags: ["ssh", "hardening", "linux", "sécurité", "sysops", "opsec", "infrastructure", "CVE", "regreSSHion", "DirtyFrag", "Copy-Fail", "kernel", "zero-day"]
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

## 8. Pourquoi tout ça compte — les vulnérabilités récentes

Le hardening SSH, ce n'est pas de la paranoïa. Voici les 5 vulnérabilités les plus marquantes qui touchent directement ou indirectement les serveurs SSH ces dernières années :

### Top 5 — Impact réel sur les serveurs SSH (2022-2026)

| Rang | Vulnérabilité | Type | Impact |
|------|--------------|------|--------|
| **1** | **DirtyFrag** (CVE-2026-43284 / CVE-2026-43500) | LPE Kernel | user → root |
| **2** | **Copy Fail** (CVE-2026-31431) | LPE Kernel | user → root |
| **3** | **regreSSHion** (CVE-2024-6387) | RCE OpenSSH | internet → root |
| **4** | **Dirty Pipe** (CVE-2022-0847) | LPE Kernel | user → root |
| **5** | **Dirty COW** (CVE-2016-5195) | LPE Kernel | user → root |

### DirtyFrag (CVE-2026-43284 / CVE-2026-43500) — Mai 2026

La plus récente et la plus critique. Chaîne de deux bugs dans le noyau Linux permettant à un utilisateur non privilégié d'obtenir root. Même famille conceptuelle que Dirty Pipe et Copy Fail : **corruption du page cache**. Les chercheurs la décrivent comme fiable sur la plupart des distributions majeures.

### Copy Fail (CVE-2026-31431)

Écriture contrôlée dans le page cache via des mécanismes cryptographiques du noyau. **Pas de race condition** contrairement à Dirty COW — ce qui la rend extrêmement fiable. Des rapports font état d'exploitation réelle dans la nature, et elle figure au catalogue KEV de la CISA.

### regreSSHion (CVE-2024-6387) — Juillet 2024

La vulnérabilité OpenSSH la plus grave depuis près de 20 ans. Découverte par Qualys, elle permettait à un attaquant **distant non authentifié** d'obtenir l'exécution de code à distance avec les privilèges root sur les serveurs utilisant glibc. Versions concernées : OpenSSH 8.5p1 à 9.7p1. Plus de **14 millions d'instances** vulnérables étaient estimées exposées sur Internet.

### Dirty Pipe (CVE-2022-0847) & Dirty COW (CVE-2016-5195)

Les ancêtres de la famille. Dirty Pipe permet d'écrire dans des fichiers en lecture seule via le mécanisme `pipe`. Dirty COW exploite une race condition dans la gestion de la mémoire copie-écriture. Les deux sont toujours pertinentes sur les noyaux non patchés.

### Le scénario d'attaque réel aujourd'hui

Voici la chaîne la plus dangereuse pour un serveur SSH exposé :

```
Vol de mot de passe / clé privée
         │
         ▼
Connexion SSH avec un compte utilisateur normal
         │
         ▼
Exploitation de DirtyFrag ou Copy Fail
         │
         ▼
Obtention de root
         │
         ▼
Persistance / rootkit
```

**C'est pour ça que le hardening ne s'arrête pas au démon SSH.** Un accès utilisateur limité, c'est un tremplin. Si le noyau n'est pas patché, un simple compte suffit pour tout compromettre.

> **Note :** Le terme "zero-day" est souvent utilisé abusivement. Un véritable 0-day exploité dans la nature avant divulgation sur OpenSSH reste extrêmement rare. DirtyFrag et Copy Fail ne sont pas des vulnérabilités SSH — ce sont des failles noyau qui transforment un accès SSH limité en compromission totale.

### Références

- [regreSSHion — Qualys Security Advisory](https://blog.qualys.com/vulnerabilities-threat-research/2024/07/01/regressionhion-sshd-race-condition)
- [CVE-2024-6387 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-6387)
- [CVE-2026-31431 (Copy Fail) — CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [CVE-2026-43284 (DirtyFrag) — NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-43284)
- [CVE-2022-0847 (Dirty Pipe) — NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-0847)
- [CVE-2016-5195 (Dirty COW) — NVD](https://nvd.nist.gov/vuln/detail/CVE-2016-5195)
- [CVE-2024-3094 (Backdoor XZ Utils) — Red Hat](https://www.redhat.com/en/blog/xz-backdoor-update)
- [ssh-audit — GitHub](https://github.com/jtesta/ssh-audit)

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
