---
title: "OSINT : l'art de la reconnaissance ouverte"
date: 2026-06-10 10:00 +0200
categories: ["Sécurité"]
tags: ["osint", "reconnaissance", "shodan", "sécurité", "opsec", "pentest", "hacking"]
author: marvax
---

> L'OSINT (Open Source Intelligence), c'est collecter du renseignement à partir de sources publiques. Avant de lancer un pentest ou de sécuriser ton infra, tu dois savoir ce qui est exposé. Shodan, Google dorks, DNS recon, réseaux sociaux — tout est accessible légalement. Et c'est exactement ce que les attaquants font en premier.

<!--more-->

## C'est quoi l'OSINT exactement ?

OSINT = **Open Source Intelligence**. Renseignement provenant de sources ouvertes et accessibles à tous. Pas de hacking, pas d'intrusion. Juste de la collecte méthodique d'informations publiques.

L'OSINT sert deux camps :

- **Offensif** — Pentesters et bug bounty hunters qui cartographient une cible avant de tenter une intrusion
- **Défensif** — Équipes sécurité qui vérifient leur propre exposition et celle de leurs employés

Dans les deux cas, l'objectif est le même : **savoir ce qui est visible de l'extérieur**.

## Phase 1 : Reconnaissance passive

Tu ne touches pas à la cible. Tu collectes uniquement depuis des sources tierces.

### Shodan — Le moteur de recherche des devices exposés

[Shodan](https://www.shodan.io/) indexe les appareils connectés à Internet : serveurs, caméras, routeurs, IoT, bases de données…

```bash
# Installation de l'CLI Shodan
pip install shodan
shodan init TA_CLE_API

# Chercher tous les serveurs Nginx exposés en France
shodan search "nginx country:FR"

# Trouver les bases MongoDB exposées sans auth
shodan search "MongoDB Server Information" --fields ip_str,port

# Infos sur une IP
shodan host 203.0.113.42
```

Ce que tu peux trouver avec Shodan :

| Requête | Ce que ça révèle |
|---|---|
| `port:22` | Serveurs SSH exposés |
| `product:MongoDB` | Bases MongoDB ouvertes |
| `http.title:"Dashboard"` | Dashboards d'admin visibles |
| `ssl.cert.subject:CN=example.com` | Certificats TLS d'un domaine |
| `org:"Target Corp"` | Toute l'infra d'une organisation |

Si tes services apparaissent sur Shodan, c'est que ton [UFW](/posts/2026-06-18-UFW-iptables-Firewall-Linux/) n'est pas correctement configuré ou que tu n'utilises pas de [Cloudflare Tunnel](/posts/2026-02-14-Cloudflare-Tunnel-zero-port/).

### Google Dorks — La recherche avancée

Les Google Dorks exploitent les opérateurs de recherche avancée de Google pour trouver des fichiers, pages et configurations exposées.

```bash
# Fichiers .env exposés
site:example.com filetype:env

# Panneaux d'admin
site:example.com inurl:admin OR inurl:dashboard OR inurl:login

# Fichiers de configuration exposés
site:example.com filetype:yml OR filetype:yaml OR filetype:conf

# Erreurs SQL visibles
site:example.com "sql syntax" OR "mysql_fetch" OR "ORA-"

# Clés API dans du code
site:github.com "example.com" "api_key" OR "apikey" OR "secret"
```

Ressource incontournable : [Google Hacking Database (GHDB)](https://www.exploit-db.com/google-hackers/) — une collection de dorks classés par catégorie.

### DNS Reconnaissance

```bash
# Installation
pip install dnspython

# Subdomain enumeration avec amass
amass enum -passive -d example.com

# Récupérer les enregistrements DNS
dig example.com ANY +short
dig example.com MX +short
dig example.com TXT +short

# Zone transfer (si mal configuré)
dig axfr example.com @ns1.example.com

# Reverse DNS sur un range
nmap -sL 203.0.113.0/24 | grep -v "Host is up"
```

Les sous-domaines révèlent souvent des environnements de staging, d'admin, ou d'anciens services oubliés. C'est là que le [scan de vulnérabilités Nuclei](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/) prend le relais.

## Phase 2 : Reconnaissance active

Tu interagis directement avec la cible. C'est plus intrusif et ça laisse des traces.

### Nmap — Scan de ports et services

```bash
# Scan rapide des ports courants
nmap -sV -sC --top-ports 1000 example.com

# Scan furtif (SYN scan)
nmap -sS -T2 -Pn example.com

# Détection de version et OS
nmap -A -O example.com

# Scan complet de tous les ports
nmap -p- -T4 example.com
```

### theHarvester — Collecte d'emails et sous-domaines

```bash
# Installation
pip install theHarvester

# Collecter des emails et sous-domaines
theHarvester -d example.com -b google,bing,linkedin,dnsdumpster

# Export en HTML
theHarvester -d example.com -b all -f rapport.html
```

### SpiderFoot — Automatisation OSINT

```bash
# Installation
pip install spiderfoot

# Lancer l'interface web
spiderfoot -l 127.0.0.1:5001
```

SpiderFoot agrège des dizaines de sources (Shodan, HaveIBeenPwned, DNS, Whois, réseaux sociaux) dans une interface unique. Tu lances un scan sur un domaine, il te sort un rapport complet.

### Maltego — Visualisation de relations

Maltego est l'outil de choix pour visualiser les liens entre entités : personnes, organisations, domaines, IPs, emails. Version Community gratuite disponible.

Pas de CLI, c'est un outil graphique. Mais les résultats sont bluffants quand tu veux cartographier une organisation.

## Phase 3 : OSINT humain

Le maillon faible, c'est toujours l'humain. L'OSINT social cible les personnes, pas les machines.

### Ce qui est récupérable publiquement

- **LinkedIn** — Organigramme, technologies utilisées, partenaires, employés récents
- **GitHub** — Credentials hardcodés, clés API, noms internes, configs
- **Twitter/X** — Informations sur la culture d'entreprise, déplacements, événements
- **Facebook/Instagram** — Géolocalisation, habitudes, vie privée

```bash
# Vérifier les fuites de données associées à un email
# (HaveIBeenPwned API)
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/user@example.com" \
  -H "hibp-api-key: TA_CLE"
```

Les [hygiènes post-fuite](/posts/2026-06-17-Guide-hygiene-post-fuite/) et l'adoption de [passkeys](/posts/2026-02-07-Passkeys-FIDO2-mort-mdp/) réduisent considérablement l'impact de ces collectes.

### Reconnaissance sur le code source

```bash
# Chercher des secrets dans les repos publics d'une org
# (avec trufflehog)
pip install trufflehog
trufflehog git https://github.com/target-org/repo.git

# Gitleaks (alternatif)
gitleaks detect --source /path/to/repo
```

La [supply chain](/posts/2026-02-05-Supply-Chain-Attacks-2026/) commence souvent par un secret exposé dans un repo public.

## OSINT défensif : audit ta propre exposition

Tu devrais faire un audit OSINT sur **ta propre infra** régulièrement. Voici la checklist :

- [ ] Lancer Shodan sur tes IPs publiques
- [ ] Vérifier les Google Dorks sur tes domaines
- [ ] Scanner les sous-domaines oubliés (staging, old-app, test…)
- [ ] Auditer les repos GitHub de l'organisation (secrets, configs)
- [ ] Vérifier les fuites associées aux emails de l'entreprise
- [ ] Contrôler les enregistrements DNS (SPF, DKIM, DMARC)
- [ ] Vérifier l'exposition des employés sur LinkedIn/Twitter
- [ ] Scanner les ports exposés avec [Nuclei](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/)
- [ ] Vérifier que le [SSH est correctement hardened](/posts/2026-06-12-Hardening-SSH/)

Intègre cette checklist dans ton [playbook d'incident response](/posts/2026-05-29-Incident-Response-Playbook/) et dans tes processus d'[automatisation Ansible](/posts/2026-06-10-Automatiser-Infra-Ansible/).

## Cadre légal

L'OSINT est légal tant que tu restes sur des sources publiques. Mais :

| Action | Légalité |
|---|---|
| Consulter Shodan | ✅ Légal |
| Google Dorks | ✅ Légal |
| Scraper des profils publics | ⚠️ Zone grise (RGPD) |
| Accéder à un système sans autorisation | ❌ Illégal |
| Exploiter une vulnérabilité trouvée | ❌ Illégal (sauf autorisation) |
| Publier des données personnelles collectées | ❌ Illégal (RGPD) |

**Règle d'or** : tu peux chercher, tu peux regarder, mais tu ne touches pas sans autorisation écrite. Pour un pentest, tu as besoin d'un **scope documenté** et d'une **autorisation signée**.

## Chaîne d'outils complète

```
Reconnaissance passive         →  Shodan, Google Dorks, DNS, WHOIS
Énumération                    →  Amass, theHarvester, SpiderFoot
Scan actif                     →  Nmap, Nuclei, OpenVAS
OSINT humain                   →  LinkedIn, GitHub, HIBP
Visualisation                  →  Maltego
Reporting                      →  Rapport structuré + recommandations
```

Chaque outil alimente le suivant. L'OSINT, c'est pas un outil, c'est une **méthodologie**.

---

*Références :*

- [Shodan](https://www.shodan.io/)
- [Google Hacking Database](https://www.exploit-db.com/google-hackers/)
- [theHarvester](https://github.com/laramies/theHarvester)
- [SpiderFoot](https://www.spiderfoot.net/)
- [Maltego](https://www.maltego.com/)
- [Nuclei — scan de vulnérabilités](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/)
- [Hardening SSH](/posts/2026-06-12-Hardening-SSH/)
- [Incident Response Playbook](/posts/2026-05-29-Incident-Response-Playbook/)
- [Passkeys / FIDO2](/posts/2026-02-07-Passkeys-FIDO2-mort-mdp/)
- [Supply Chain Attacks](/posts/2026-02-05-Supply-Chain-Attacks-2026/)

*Besoin d'un audit OSINT de ton infrastructure ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
