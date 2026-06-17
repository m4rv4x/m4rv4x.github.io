---
title: "BadHost — Un caractère et votre framework web passe à l'ennemi"
date: 2026-05-26 10:00 +0200
categories: ["Sécurité"]
tags: ["rce", "starlette", "fastapi", "python", "web", "supply-chain", "agents-ia"]
author: marvax
---

> Un simple caractère malformé dans le header `Host` d'une requête HTTP suffit à contourner les protections de sécurité de Starlette et FastAPI. La faille BadHost (CVE-2026-48710) expose un pattern qu'on retrouve de plus en plus dans la chaîne d'approvisionnement de l'IA.

<!--more-->

## La faille

Starlette est un framework ASGI léger en Python, utilisé directement ou via **FastAPI** (qui en dépend). En mai 2026, la vulnérabilité **BadHost** (CVE-2026-48710) a été divulguée : le header `Host` n'était pas validé avant d'être utilisé pour reconstruire `request.url`.

### Le mécanisme

```
GET /admin HTTP/1.1
Host: evil.com
```

Le routing d'ASGI utilise le **chemin brut** (`scope["path"]`), mais `request.url` est reconstruit à partir du header `Host`. Un header malformé peut faire diverger `request.url.path` du chemin réellement routé.

Résultat : les middlewares et endpoints qui appliquent des restrictions de sécurité basées sur `request.url` (plutôt que sur `scope["path"]`) peuvent être contournés.

| | |
|---|---|
| **CVE** | CVE-2026-48710 |
| **CVSS** | 6.5 MEDIUM |
| **Affecte** | Starlette < 1.0.1, FastAPI (toutes versions dépendantes) |
| **CWE** | CWE-444 (HTTP Request/Response Smuggling), CWE-1289 |
| **Fix** | Starlette 1.0.1 |

## Pourquoi ça concerne les agents IA

C'est là que ça devient intéressant. Starlette et FastAPI sont les frameworks les plus utilisés pour exposer des **agents IA** et des **API LLM** en Python. Un agent qui tourne derrière un endpoint FastAPI avec des restrictions d'accès basées sur l'URL est vulnérable au contournement BadHost.

Le pattern qu'on voit se répéter :

```
Agent IA exposé via FastAPI
    → restriction d'accès basée sur request.url
    → contournement via header Host malformé
    → accès non autorisé aux endpoints de l'agent
    → exécution de prompts/arbitraire via l'agent
```

C'est le même pattern que la faille **RCE de llama-cpp-python** : la chaîne d'approvisionnement de l'IA repose sur des composants qui n'ont pas été pensés pour être des surfaces d'attaque.

## Ce qu'il faut faire

```bash
# Mettre à jour Starlette
pip install --upgrade starlette>=1.0.1

# Vérifier ta version
python -c "import starlette; print(starlette.__version__)"

# Si tu utilises FastAPI, la mise à jour de Starlette suffit
pip install --upgrade fastapi
```

## Vérifier si tu es exposé

```python
# Test rapide : envoyer un Host header malformé
import requests

r = requests.get(
    "http://ton-serveur/admin",
    headers={"Host": "evil.com\r\nX-Injected: true"},
    allow_redirects=False
)
print(r.status_code, r.headers.get("location"))
```

Si la réponse diffère de celle obtenue avec un Host normal, tu es potentiellement vulnérable.

## Le pattern récurrent

BadHost rappelle une mécanique qu'on a déjà vue :
- **llama-cpp-python RCE** — exécution de code via le parsing des modèles
- **Log4Shell** — injection via un header contrôlé
- **BadHost** — contournement via un header malformé

Le point commun : des composants largement utilisés qui font confiance à des entrées HTTP sans validation stricte. Dans un monde où de plus en plus d'agents IA sont exposés via des API web, ces failles deviennent des vecteurs d'attaque directs sur l'infrastructure IA.

---

*Références :*
- [NVD — CVE-2026-48710](https://nvd.nist.gov/vuln/detail/CVE-2026-48710)
- [GitHub Advisory — GHSA-86qp-5c8j-p5mr](https://github.com/Kludex/starlette/security/advisories/GHSA-86qp-5c8j-p5mr)
- [OSTIF — BadHost disclosure](https://ostif.org/disclosing-the-badhost-vulnerability-in-starlette)
- [x41-dsec advisory](https://www.x41-dsec.de/lab/advisories/x41-2026-002-starlette)

*Tu exposes des API FastAPI ou des agents IA ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour un audit de sécurité.*
