---
title: "Shodan : découvrir ce qui est exposé sur Internet"
date: 2026-06-25 10:00 +0200
categories: ["Sécurité"]
tags: ["shodan", "reconnaissance", "exposition", "sysops", "infrastructure", "osint"]
author: marvax
---

> Shodan scanne Internet en continu et indexe tout ce qui répond : serveurs, caméras, bases de données, panneaux d'admin, Ollama instances... Si c'est exposé, Shodan le sait.

<!--more-->

## C'est quoi Shodan

Shodan est un moteur de recherche pour les appareils connectés. Contrairement à Google qui indexe les pages web, Shodan indexe les **services** qui tournent sur les ports ouverts.

```
Google : "qui a un site web ?"
Shodan : "qui a un port ouvert ?"
```

## Installation CLI

```bash
pip install shodan
shodan init TON_API_KEY
```

## Recherches essentielles

```bash
# Trouver des serveurs Ollama exposés
shodan search "ollama" --fields ip_str,port,org

# Serveurs web avec Nginx en France
shodan search "nginx country:FR" --fields ip_str,port,hostnames

# Bases de données MongoDB exposées (sans auth)
shodan search "MongoDB Server Information" --fields ip_str,port

# Caméras IP exposées
shodan search "Server: Hikvision-Webs" --fields ip_str,port

# Serveurs SSH avec password auth
shodan search "SSH-2.0 OpenSSH port:22" --fields ip_str,port,version

# Services Jenkins exposés
shodan search "X-Jenkins" --fields ip_str,port

# Docker API exposé
shodan search "Docker Containers:" --fields ip_str,port
```

## Recherches avancées

```bash
# Par réseau (ASN)
shodan search "net:103.21.244.0/22" --fields ip_str,port,product

# Par CVE
shodan search "vuln:CVE-2024-6387" --fields ip_str,port,org

# Par port spécifique
shodan search "port:9200" --fields ip_str,product  # Elasticsearch

# Par header HTTP
shodan search "Server: Apache httpd country:FR" --fields ip_str,port

# Combinaison
shodan search "product:MySQL country:FR version:5.7" --fields ip_str,port
```

## Scanner ta propre infrastructure

```bash
# Info sur une IP
shodan host TON_IP

# Scan on-demand (nécessite API key payante)
shodan scan submit TON_IP
shodan scan status SCAN_ID

# Historique d'une IP
shodan host history TON_IP
```

## Monitorer ton exposition

```bash
# Créer une alerte pour ton réseau
shodan alert create "Mon reseau" TON_IP/24

# Lister les alertes
shodan alert list

# Recevoir les notifications par email (config sur shodan.io)
```

## Exemples terrain

### Trouver des instances Ollama exposées

```bash
shodan search "title:Ollama" --fields ip_str,port,org | head 20
```

Résultat typique : des centaines d'instances Ollama accessibles sans authentification, avec des modèles LLM chargés. N'importe qui peut leur poser des questions.

### Vérifier si ton serveur est exposé

```bash
shodan host $(curl -s ifconfig.me)
```

Tu verras exactement ce que Shodan voit : ports ouverts, services, versions, certificats SSL.

### Trouver des bases Elasticsearch ouvertes

```bash
shodan search "port:9200 product:Elasticsearch" --fields ip_str,port,org --limit 100
```

## Shodan vs nmap

| | Shodan | nmap |
|---|--------|------|
| Portée | Tout Internet | Cible spécifique |
| Vitesse | Instantané (données pré-indexées) | Lent (scan en direct) |
| Profondeur | Headers, bannières, vulns | Ports, services, scripts |
| Coût | Freemium | Gratuit |
| Usage | Recon passive | Scan actif |

**Règle** : commence par Shodan (recon passive), affine avec nmap (scan actif).

## Python API

```python
import shodan

api = shodan.Shodan("TON_API_KEY")

# Recherche
results = api.search("apache country:FR")
for result in results['matches']:
    print(f"{result['ip_str']}:{result['port']} - {result.get('org', 'N/A')}")

# Info sur une IP
host = api.host("8.8.8.8")
print(f"IP: {host['ip_str']}")
print(f"Org: {host.get('org', 'N/A')}")
print(f"OS: {host.get('os', 'N/A')}")
for item in host['data']:
    print(f"  Port {item['port']}: {item.get('product', 'N/A')}")
```

## Bonnes pratiques

- **Recon passive d'abord** : Shodan avant nmap
- **Alertes sur tes IPs** : surveille ton exposition
- **Vérifie après chaque deploy** : nouveau port ouvert ?
- **Nettoie l'exposition** : ferme ce que tu n'utilises pas
- **Rate limiting** : l'API a des limites (1 requête/sec en gratuit)

---

*Références :*
- [Shodan.io](https://www.shodan.io/)
- [Shodan CLI](https://cli.shodan.io/)
- [Shodan Python library](https://github.com/achillean/shodan-python)

*Ton infrastructure est-elle exposée ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour un audit d'exposition.*
