---
title: "Nuclei & OpenVAS : scanne tes vulnérabilités avant les autres"
date: 2026-03-08 10:00 +0200
categories: ["Sécurité"]
tags: ["nuclei", "openvas", "vulnerability-scanning", "sécurité", "sysops", "hardening", "pentest"]
author: marvax
---

> Tu n'as pas besoin d'attendre qu'un pentester trouve tes failles. Nuclei et OpenVAS te permettent de scanner tes propres infrastructures gratuitement, de manière répétée et automatisée. Nuclei est rapide et basé sur des templates YAML. OpenVAS est plus lourd mais plus complet. Ensemble, ils couvrent la majorité des vulnérabilités connues. Cet article te montre comment les installer, les utiliser et intégrer ça dans ta routine de sécurité.

<!--more-->

## Pourquoi scanner soi-même ?

La majorité des breaches exploitent des CVE publiques depuis des mois. Si tu scannes régulièrement, tu corriges avant l'attaquant. C'est aussi un prérequis pour tout audit sérieux.

Deux outils complémentaires :

| | Nuclei | OpenVAS (GVM) |
|---|---|---|
| Approche | Templates HTTP/network | Vuln DB + moteur NVT |
| Vitesse | Très rapide | Lent (complet) |
| Installation | Un binaire | Suite lourde |
| Courbe d'apprentissage | Faible | Moyenne |
| Résultats | Match/miss par template | Scores CVSS détaillés |
| Idéal pour | CI/CD, scans rapides | Audits complets |
| Maintenance templates | Communauté très active | Mises à jour NVT |

L'idéal : Nuclei pour le continu, OpenVAS pour le périodique.

## Nuclei

### Installation

```bash
# Via Go
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Ou binaire direct
curl -sL https://github.com/projectdiscovery/nuclei/releases/latest/download/nuclei_Linux_x86_64.zip -o nuclei.zip
unzip nuclei.zip && sudo mv nuclei /usr/local/bin/
```

### Mise à jour des templates

```bash
nuclei -update-templates
```

Les templates sont stockés dans `~/nuclei-templates/`. La communauté en ajoute quotidiennement.

### Scan basique

```bash
# Scan un seul host
nuclei -u https://target.example.com

# Scan avec tous les templates critiques
nuclei -u https://target.example.com -severity critical,high

# Scan depuis une liste de cibles
nuclei -l targets.txt -severity critical,high,medium -o results.txt

# Format JSON pour traitement automatisé
nuclei -u https://target.example.com -json -o results.jsonl
```

### Templates personnalisés

Crée un template Nuclei pour vérifier une config spécifique :

```yaml
id: custom-xss-check
info:
  name: Reflected XSS Check
  author: marvax
  severity: high

requests:
  - method: GET
    path:
      - "{{BaseURL}}/search?q=<script>alert(1)</script>"
    matchers:
      - type: word
        words:
          - "<script>alert(1)</script>"
```

```bash
nuclei -u https://target.example.com -t custom-xps-check.yaml
```

### Exemples de scan par catégorie

```bash
# Uniquement les CVEs connues
nuclei -u https://target.example.com -tags cve

# Scan des technologies détectées
nuclei -u https://target.example.com -tags tech

# Détection de panels d'admin exposés
nuclei -u https://target.example.com -tags panel

# Vérification des configurations TLS
nuclei -u https://target.example.com -tags ssl
```

## OpenVAS (Greenbone)

### Installation

La méthode la plus propre pour un homelab — Docker :

```bash
docker run -d \
  --name openvas \
  -p 443:443 \
  -p 9390:9390 \
  -e PASSWORD=admin \
  -e USERNAME=admin \
  --volume openvas-data:/data \
  mikesplain/openvas
```

Ou installation complète sur Debian/Ubuntu :

```bash
sudo apt install -y gvm
sudo gvm-setup
sudo gvm-start
```

L'installation complète prend du temps et consomme de la RAM (4 Go minimum recommandé).

### Lancer un scan

1. Accède à l'interface web (`https://localhost:443`)
2. Crée un **Target** (IP ou hostname)
3. Crée un **Task** associé au target
4. Lance le scan

En ligne de commande :

```bash
# Créer un target
omp -u admin -w admin --xml="<create_target><name>Mon Serveur</name><hosts>192.168.1.10</hosts></create_target>"

# Lancer un scan (via gvm-cli)
gvm-cli --gmp-username admin --gmp-password admin socket \
  --xml "<create_task><name>Scan hebdo</name><target id='TARGET_ID'/></create_task>"
```

### Interpréter les résultats

OpenVAS attribue un score CVSS à chaque vulnérabilité :

| Score | Niveau | Action |
|---|---|---|
| 9.0 - 10.0 | Critique | Correction immédiate |
| 7.0 - 8.9 | Haute | Correction sous 48h |
| 4.0 - 6.9 | Moyenne | Correction planifiée |
| 0.1 - 3.9 | Basse | Évaluation contextuelle |

Ne patch pas aveuglément. Un score "moyen" sur un service exposé publiquement peut être plus critique qu'un score "haut" sur un service interne.

## Automatisation des scans

### Cron Nuclei quotidien

```bash
# /etc/cron.d/nuclei-scan
0 3 * * * root /usr/local/bin/nuclei -l /etc/nuclei/targets.txt -severity critical,high -json -o /var/log/nuclei/$(date +\%F).json 2>&1 | logger -t nuclei
```

### CI/CD

Intègre Nuclei dans ton pipeline ([CI/CD GitHub Actions](/posts/2026-06-20-CICD-GitHub-Actions/)) :

```yaml
name: Security Scan
on:
  push:
    branches: [main]

jobs:
  nuclei-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Nuclei
        run: |
          go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
          nuclei -update-templates
      - name: Scan staging
        run: nuclei -u https://staging.example.com -severity critical,high -fail-on-matched
```

Le flag `-fail-on-matched` fait échouer le pipeline si une vulnérabilité est trouvée.

### Alertes et monitoring

Envoie les résultats vers ta stack de monitoring ([Monitoring Prometheus & Grafana](/posts/2026-06-15-Monitoring-Prometheus-Grafana/)) :

```bash
# Parser les résultats JSON et envoyer vers un webhook
cat /var/log/nuclei/$(date +%F).json | jq -r '. | select(.info.severity == "critical")' | \
  curl -X POST -H "Content-Type: application/json" -d @- https://alerts.example.com/webhook
```

## Après le scan : durcir

Scanner c'est bien, corriger c'est mieux. Priorise :

1. **Patch** : mets à jour les logiciels exposés
2. **Harden SSH** : désactive les versions obsolètes ([Hardening SSH](/posts/2026-06-12-Hardening-SSH/))
3. **Fail2ban** : bloque les tentatives d'exploitation ([Fail2ban avancé](/posts/2026-06-21-Fail2ban-avance/))
4. **CrowdSec** : protection collaborative ([CrowdSec : le futur de Fail2ban](/posts/2026-01-16-CrowdSec-fail2ban-futur/))
5. **Docker security** : vérifie tes images ([Sécurité Docker](/posts/2026-06-08-Docker-Security-Erreurs/))
6. **Supply chain** : audite tes dépendances ([Supply Chain Attacks](/posts/2026-02-05-Supply-Chain-Attacks-2026/))

## Checklist de mise en place

- [ ] Nuclei installé et templates à jour
- [ ] OpenVAS déployé (Docker ou natif)
- [ ] Liste de cibles définie et documentée
- [ ] Scan Nuclei automatisé (cron ou CI/CD)
- [ ] Scan OpenVAS planifié (hebdomadaire/mensuel)
- [ ] Résultats stockés et versionnés
- [ ] Alertes configurées pour les findings critiques
- [ ] Processus de correction documenté
- [ ] Hardening appliqué post-scan

*Références :*
- [Nuclei — ProjectDiscovery](https://github.com/projectdiscovery/nuclei)
- [Greenbone OpenVAS](https://www.greenbone.net/en/community-edition/)
- [Hardening SSH](/posts/2026-06-12-Hardening-SSH/)
- [Sécurité Docker](/posts/2026-06-08-Docker-Security-Erreurs/)
- [Fail2ban avancé](/posts/2026-06-21-Fail2ban-avance/)
- [CrowdSec : le futur de Fail2ban](/posts/2026-01-16-CrowdSec-fail2ban-futur/)
- [CI/CD GitHub Actions](/posts/2026-06-20-CICD-GitHub-Actions/)
- [Monitoring Prometheus & Grafana](/posts/2026-06-15-Monitoring-Prometheus-Grafana/)
- [Supply Chain Attacks](/posts/2026-02-05-Supply-Chain-Attacks-2026/)

Besoin d'un audit de vulnérabilités ou d'un coup de main sur le durcissement ? [Contacte-moi](mailto:m4rv4x@protonmail.com).
