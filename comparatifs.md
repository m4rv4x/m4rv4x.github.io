---
title: Comparatifs
layout: page
permalink: /comparatifs/
icon: fas fa-balance-scale
---

# Comparatifs

Choisir le bon outil, c'est 80% du travail. Voici des comparatifs objectifs bases sur l'experience terrain.

---

## Fail2ban vs CrowdSec

| Critere | Fail2ban | CrowdSec |
|---------|----------|----------|
| **Architecture** | Local uniquement | Local + reseau communautaire |
| **Detection** | Regles regex sur les logs | Patterns + partage de threat intel |
| **Partage** | Aucun | Bloc-list partagee entre instances |
| **Installation** | `apt install fail2ban` | Script d'installation officiel |
| **Config** | Fichiers .conf/.local | YAML + hub de scenarios |
| **Performance** | Tres leger | Leger (Go) |
| **Bouncers** | iptables, firewalld | iptables, nginx, cloudflare, etc. |
| **API** | Non | Oui (REST) |
| **Dashboard** | Non | Oui (cscli + optionnellement dashboard web) |

**Verdict :** Fail2ban pour les cas simples (1 serveur, quelques services). CrowdSec des que tu as plusieurs machines ou que tu veux beneficier de la menace partagee. Les deux peuvent coexister.

**Article complet :** [CrowdSec : le Fail2ban du futur](/posts/CrowdSec-fail2ban-futur/)

---

## WireGuard vs OpenVPN vs IPSec

| Critere | WireGuard | OpenVPN | IPSec |
|---------|-----------|---------|-------|
| **Lignes de code** | ~4,000 | ~100,000 | ~400,000 |
| **Protocole** | UDP (unique) | TCP/UDP | UDP (ESP) |
| **Vitesse** | Proche du natif | 10-30% de perte | Variable |
| **Configuration** | 1 fichier | Fichiers .ovpn + PKI | Complexe (IKE, SA) |
| **Roaming** | Natif (cle publique) | Reconnexion | Re-negotiation |
| **NAT traversal** | Excellent | Correct | Correct |
| **Audit** | Oui (code minimal) | Partiel | Complexe |
| **Multi-platform** | Linux, macOS, Windows, Android, iOS | Tous | Tous |
| **Chiffrement** | ChaCha20, Curve25519 | Configurable (OpenSSL) | Configurable |

**Verdict :** WireGuard pour 95% des cas. OpenVPN si tu as besoin de TCP (traversant les proxys). IPSec pour les environnements enterprise legacy.

**Article complet :** [WireGuard : VPN self-hosted en 10 minutes](/posts/WireGuard-VPN-SelfHosted/)

---

## Traefik vs Nginx vs Caddy

| Critere | Traefik | Nginx | Caddy |
|---------|---------|-------|-------|
| **Auto-discovery Docker** | Natif | Non (manuel) | Plugin |
| **TLS auto** | Let's Encrypt integre | Certbot externe | Let's Encrypt natif |
| **Config** | Labels Docker + TOML | Fichiers .conf | Caddyfile (simple) |
| **Dashboard** | Oui (web UI) | Non (nginx-ui externe) | API REST |
| **Reload** | Automatique | `nginx -s reload` | Automatique |
| **Performance** | Tres bonne | Excellente | Bonne |
| **Middleware** | Headers, auth, rate-limit | Modules | Plugins Go |
| **Learning curve** | Moyenne | Courbe (puissant) | Faible |
| **Cas d'usage** | Docker, microservices | Haute perf, static sites | Simplicite, auto-TLS |

**Verdict :** Traefik si tu tournes Docker. Nginx si tu veux la performance brute et le controle total. Caddy si tu veux le TLS automatique sans configuration.

**Article complet :** [Traefik : le reverse proxy que tu merites](/posts/Traefik-reverse-proxy-docker/) | [Reverse proxy Nginx avec TLS](/posts/Reverse-Proxy-Nginx-TLS/)

---

## BorgBackup vs Restic vs Duplicati

| Critere | BorgBackup | Restic | Duplicati |
|---------|------------|--------|-----------|
| **Deduplication** | Oui (chunk-based) | Oui (content-defined) | Oui |
| **Chiffrement** | AES-256-CTR | AES-256-CTR | AES-256 |
| **Compression** | Oui (lz4, zstd, zlib) | Non | Oui (zip) |
| **Backends** | Local, SSH | Local, S3, SFTP, B2, Azure, etc. | S3, FTP, SSH, Backblaze, etc. |
| **Vitesse** | Tres rapide | Rapide | Moyenne |
| **Prune** | `borg prune` | `restic forget --prune` | Retention configurable |
| **Mount** | `borg mount` | `restic mount` | Non |
| **Langage** | Python + C | Go | C# (.NET) |
| **Interface** | CLI uniquement | CLI uniquement | Web UI |

**Verdict :** BorgBackup pour le self-hosted pur (SSH + local). Restic si tu veux des backends cloud (S3, B2). Duplicati si tu veux une web UI sans ligne de commande.

**Article complet :** [BorgBackup : strategie de backup 3-2-1](/posts/BorgBackup-strategie-backup/)

---

## Passkeys vs MFA vs Mots de passe

| Critere | Mots de passe | MFA (TOTP) | Passkeys (FIDO2) |
|---------|---------------|------------|------------------|
| **Phishing** | Vulnerable | Vulnerable (relay) | Immune (crypto) |
| **UX** | Mauvaise | Moyenne | Excellente |
| **Recovery** | Reset email | Codes de backup | Device + cloud sync |
| **Stockage** | Base de donnees (hash) | Secret partage | Cle privee sur device |
| **Standard** | N/A | RFC 6238 | WebAuthn / FIDO2 |
| **Adoption** | 100% | ~60% des services | ~15% (2026) |
| **Vendor lock-in** | Non | Non | Partiel (Apple, Google, MS) |

**Verdict :** Passkeys > MFA > Mots de passe. L'adoption des passkeys est encore limitee mais c'est l'avenir. En attendant, MFA partout + gestionnaire de mots de passe.

**Article complet :** [Passkeys & FIDO2 : la mort du mot de passe](/posts/Passkeys-FIDO2-mort-mdp/)

---

## UFW vs iptables vs nftables

| Critere | UFW | iptables | nftables |
|---------|-----|----------|----------|
| **Simplicite** | Tres simple | Complexe | Moyenne |
| **Puissance** | Limitee | Totale | Totale |
| **Syntaxe** | `ufw allow 22` | `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` | `nft add rule ...` |
| **Tables** | Frontend pour iptables | Legacy | Remplacement iptables |
| **Performance** | Identique | Bonne | Meilleure (moins de regles) |
| **Persistant** | Oui | Non (iptables-save) | Oui (nftables.conf) |
| **IPv6** | Integre | ip6tables separe | Unifie |

**Verdict :** UFW pour les serveurs simples (90% des cas). nftables si tu as besoin de regles complexes. iptables en legacy uniquement.

**Article complet :** [UFW & iptables : firewall Linux en pratique](/posts/UFW-iptables-Firewall-Linux/)
