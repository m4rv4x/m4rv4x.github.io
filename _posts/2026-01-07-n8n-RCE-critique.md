---
title: "n8n — RCE critique (CVSS 10.0) dans les workflows automatisés"
date: 2026-01-07 10:00 +0200
categories: ["Sécurité"]
tags: ["n8n", "rce", "automatisation", "supply-chain", "dev"]
author: marvax
---

> n8n, l'outil de workflow automation open source très populaire chez les devs et les ops, a été touché par une faille RCE critique notée CVSS 10.0. Si tu l'utilises en production, c'est le moment de patcher.

<!--more-->

## La faille

n8n est un outil d'automatisation de workflows self-hosted (alternative open source à Zapier/Make). Très utilisé pour connecter des APIs, orchestrer des pipelines de données, et automatiser des tâches ops.

La vulnérabilité permet à un attaquant d'exécuter du code arbitraire sur le serveur n8n via le mécanisme d'exécution de code des nodes. Plusieurs CVE ont été publiées en 2025-2026 :

| CVE | Date | CVSS | Impact |
|-----|------|------|--------|
| **CVE-2025-65964** | 2025 | Élevé | RCE via pre-commit hooks (v0.123.1–1.119.1) |
| **CVE-2026-27494** | 2026-01 | Critique | RCE via Python Code node (avant 2.10.1) |
| **CVE-2026-42232** | 2026 | Critique | Prototype pollution → RCE (avant 1.123.32) |

## Pourquoi c'est grave

n8n tourne souvent avec des droits élevés :
- Accès à des variables d'environnement avec des secrets (API keys, tokens)
- Connexion directe à des bases de données
- Accès réseau vers des services internes
- Parfois monté avec le socket Docker (erreur #8 de mon article Docker Security)

Un attaquant qui exploite la RCE récupère tout ça. C'est l'équivalent d'avoir un shell sur ton serveur d'automatisation avec toutes les credentials.

## Ce qu'il faut faire

```bash
# Vérifier ta version
n8n --version

# Mettre à jour (Docker)
docker pull n8nio/n8n:latest
docker compose up -d

# Ou via npm
npm update -g n8n

# Versions corrigées :
# n8n >= 2.10.1 (pour CVE-2026-27494)
# n8n >= 1.123.32 (pour CVE-2026-42232)
```

## Bonnes pratiques n8n en production

- **Ne jamais exposer n8n directement sur Internet** — passe par un reverse proxy avec auth
- **Ne pas monter le socket Docker** dans le conteneur n8n
- **Isoler réseau** — n8n ne doit accéder qu'aux services dont il a besoin
- **Restreindre les Code nodes** — désactiver l'exécution de code libre si possible
- **Mettre à jour régulièrement** — les CVE sur les outils d'automatisation sont de plus en plus fréquentes

## Le pattern à retenir

Les outils d'automatisation (n8n, Make, Zapier, Huginn) sont des cibles de choix parce qu'ils :
- Ont accès à beaucoup de systèmes et credentials
- Exécutent du code arbitraire par design
- Tournent souvent avec des privilèges élevés
- Sont rarement audités comme des services exposés

C'est le même pattern que les CI/CD (Jenkins, GitHub Actions) — une compromission de l'orchestrateur donne accès à tout.

---

*Références :*
- [NVD — CVE-2026-27494](https://nvd.nist.gov/vuln/detail/CVE-2026-27494)
- [NVD — CVE-2026-42232](https://nvd.nist.gov/vuln/detail/CVE-2026-42232)
- [NVD — CVE-2025-65964](https://nvd.nist.gov/vuln/detail/CVE-2025-65964)
- [n8n Security Advisories](https://github.com/n8n-io/n8n/security/advisories)

*Tu utilises n8n en prod ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour un audit de ta stack d'automatisation.*
