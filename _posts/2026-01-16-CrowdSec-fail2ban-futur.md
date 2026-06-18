---
title: "CrowdSec : le Fail2ban du futur"
date: 2026-01-16 10:00 +0200
categories: ["Sécurité"]
tags: ["crowdsec", "fail2ban", "sécurité", "linux", "sysops", "hardening", "infrastructure"]
author: marvax
---

> Fail2ban bloque les attaquants en local. CrowdSec fait la même chose, mais en partageant la menace avec toute la communauté. Si un serveur à l'autre bout du monde se fait scanner, ton instance le sait avant même que l'attaque arrive chez toi. Voici comment passer de l'IDS local au renseignement collectif.

<!--more-->

## CrowdSec vs Fail2ban

On me pose souvent la question : « Je dois choisir entre les deux ? ». La réponse courte : CrowdSec est un remplaçant direct de Fail2ban, mais la philosophie est radicalement différente.

| Critère | Fail2ban | CrowdSec |
|---|---|---|
| Modèle | Local uniquement | Communautaire (crowdsourced) |
| Architecture | Monolithique | Agent + Bouncer séparés |
| Blocklists | Non | Oui (hub communautaire) |
| Langage | Python | Go |
| Performance | Correcte | Significativement meilleure |
| Détection | Regex sur logs | Parsers + scenarios YAML |
| Actions | Scripts locaux | Bouncers modulaires (iptables, nginx, cloudflare…) |
| Communauté | Individuelle | Threat intelligence partagée |
| Courbe d'apprentissage | Faible | Moyenne |
| Écosystème | Filtres .conf | Hub + collections + bouncers |

**En résumé** : Fail2ban défend *ton* serveur. CrowdSec défend *tous* les serveurs qui participent au réseau.

## Installation

```bash
# Script d'installation officiel (Debian/Ubuntu)
curl -s https://install.crowdsec.net | sudo bash

# Installer CrowdSec + bouncer firewall
sudo apt install crowdsec crowdsec-firewall-bouncer-iptables -y

# Vérifier le service
sudo systemctl status crowdsec
```

CrowdSec va automatiquement analyser les logs présents sur le système (syslog, auth.log, nginx, etc.) et créer des scénarios de détection.

## Configuration des parsers et scénarios

L'agent CrowdSec fonctionne en trois étapes :

```
Log → Acquis (fichiers à surveiller) → Parsers (extraction) → Scénarios (détection) → Décisions (ban)
```

### Ajouter des sources de logs

```bash
# Voir les sources actives
sudo cscli collections list

# Installer la collection Nginx
sudo cscli collections install crowdsecurity/nginx

# Installer la collection SSH (syslog)
sudo cscli collections install crowdsecurity/sshd

# Vérifier les parsers installés
sudo cscli parsers list
```

### Voir les scénarios actifs

```bash
# Scénarios installés
sudo cscli scenarios list

# Exemples de scénarios :
# - crowdsecurity/ssh-bf          (brute force SSH)
# - crowdsecurity/http-probing    (scan HTTP)
# - crowdsecurity/http-cve-*      (CVE connues)
```

## Le Hub CrowdSec

Le Hub est la force principale de CrowdSec. C'est un dépôt communautaire de :
- **Collections** : packs complets (parsers + scénarios + bouncers)
- **Parsers** : extraction de données depuis les logs
- **Scénarios** : règles de détection
- **Blocklists** : listes d'IP malveillantes partagées

```bash
# Mettre à jour le hub
sudo cscli hub update

# Installer une collection complète (ex: WordPress)
sudo cscli collections install crowdsecurity/wordpress

# Lister les blocklists disponibles
sudo cscli blocklists list

# Activer une blocklist communautaire
sudo cscli blocklists install crowdsecurity/blocklists-malicious-ips
```

## Bouncers : l'exécution

Les bouncers sont les composants qui appliquent les décisions de ban. C'est l'équivalent des actions dans Fail2ban, mais en propre processus.

```bash
# Voir les bouncers installés
sudo cscli bouncers list

# Installer le bouncer Nginx (pour du ban applicatif)
sudo apt install crowdsec-bouncer-nginx -y

# Installer le bouncer Cloudflare (pour bloquer au niveau CDN)
sudo cscli bouncers install crowdsec-cloudflare-bouncer
```

### Configuration du bouncer firewall

```yaml
# /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
mode: iptables
# Ou nftables, pf, etc.
api_url: http://127.0.0.1:8080
api_key: <clé-générée-automatiquement>
# Règles iptables ajoutées automatiquement
iptables_chains:
  - INPUT
  - FORWARD
```

## Gestion des décisions

```bash
# Voir les bans actifs
sudo cscli decisions list

# Bannir manuellement une IP
sudo cscli decisions add --ip 203.0.113.50 --reason "scan manuel" --duration 4h

# Débannir une IP
sudo cscli decisions delete --ip 203.0.113.50

# Voir les métriques d'alertes
sudo cscli alerts list

# Dashboards (optionnel — nécessite metabase)
sudo cscli dashboard setup
```

## CrowdSec vs Fail2ban : quand utiliser quoi ?

**Garde Fail2ban si :**
- Tu as une infra simple avec 1-2 serveurs
- Tu n'as pas besoin de blocklists communautaires
- Tu veux un truc qui marche sans maintenance

**Passe à CrowdSec si :**
- Tu gères plusieurs serveurs
- Tu veux bénéficier de la threat intelligence collective
- Tu veux des bouncers flexibles (firewall, Cloudflare, nginx, etc.)
- Tu veux détecter des CVE HTTP automatiquement
- Tu veux un outil écrit en Go, plus performant

**Migration depuis Fail2ban :**

```bash
# CrowdSec peut coexister avec Fail2ban pendant la transition
# Installer CrowdSec sans le bouncer firewall au début
sudo apt install crowdsec -y

# Observer les détections en mode « dry-run »
sudo cscli metrics

# Une fois confiant, désactiver Fail2ban
sudo systemctl disable --now fail2ban
# Puis installer le bouncer
sudo apt install crowdsec-firewall-bouncer-iptables -y
```

## Intégration avec la stack de sécurité

CrowdSec s'intègre parfaitement dans une architecture de défense en profondeur :

- Firewall réseau via [UFW & iptables](/posts/UFW-iptables-Firewall-Linux/)
- Endurcissement SSH via [Hardening SSH](/posts/Hardening-SSH/)
- Scan de vulnérabilités avec [Nuclei & OpenVAS](/posts/Nuclei-OpenVAS-scan-vuln/)
- Et si tu veux garder Fail2ban pour des cas spécifiques, voici le [guide avancé](/posts/Fail2ban-avance/)

## Checklist

- [ ] CrowdSec installé et service actif
- [ ] Collections adaptées à tes services (nginx, sshd, etc.)
- [ ] Bouncers configurés (firewall minimum)
- [ ] Blocklists communautaires activées
- [ ] Décisions vérifiées (`cscli decisions list`)
- [ ] Monitoring des alertes en place
- [ ] Migration depuis Fail2ban planifiée (si applicable)

---

*Références :*
- [CrowdSec documentation](https://doc.crowdsec.net/)
- [CrowdSec Hub](https://hub.crowdsec.net/)
- [GitHub CrowdSec](https://github.com/crowdsecurity/crowdsec)

*CrowdSec configuré en mode par défaut sans bouncers ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour un audit.*
