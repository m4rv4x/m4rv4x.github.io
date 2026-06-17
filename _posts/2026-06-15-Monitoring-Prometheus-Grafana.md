---
title: "Monitoring self-hosted : Prometheus + Grafana de zéro"
date: 2026-06-15 10:00 +0200
categories: ["Sysops"]
tags: ["monitoring", "prometheus", "grafana", "docker", "sysops", "infrastructure", "linux"]
author: marvax
---

> Tu gères tes serveurs à l'aveugle ? Tu découvres les pannes quand les utilisateurs râlent ? Il est temps de mettre en place un vrai stack de monitoring. Voici comment j'ai déployé Prometheus + Grafana sur mon infrastructure, et comment tu peux reproduire ça en moins d'une heure.

<!--more-->

## Pourquoi monitoring maison ?

Les solutions SaaS (Datadog, New Relic, Grafana Cloud) c'est bien... jusqu'à ce que tu voies la facture. Pour une infrastructure perso ou un petit client, self-hoster ton monitoring c'est :

- **Zéro coût récurrent** — tu roules sur ton propre matos
- **Contrôle total des données** — rien ne quitte ton réseau
- **Apprentissage concret** — tu comprends chaque brique

Le duo gagnant : **Prometheus** (collecte + alertes) + **Grafana** (dashboards visuels).

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Node        │     │  cAdvisor    │     │  Blackbox    │
│  Exporter    │     │  (Docker)    │     │  Exporter    │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │ :9100              │ :8080              │ :9115
       └────────────┬───────┴────────────────────┘
                    │
            ┌───────▼───────┐
            │  Prometheus   │
            │  :9090        │
            └───────┬───────┘
                    │ :9090
            ┌───────▼───────┐
            │   Grafana     │
            │   :3000       │
            └───────────────┘
```

## Étape 1 — Docker Compose

Crée un dossier dédié :

```bash
mkdir -p ~/monitoring/{prometheus,grafana,alertmanager}
cd ~/monitoring
```

Le `docker-compose.yml` :

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=ChangeMoi!
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

## Étape 2 — Configuration Prometheus

`prometheus/prometheus.yml` :

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

## Étape 3 — Lancement

```bash
docker compose up -d
docker compose ps  # vérifier que tout est UP
```

Grafana est accessible sur `http://IP:3000` (admin / ChangeMoi!).

## Étape 4 — Dashboards

Dans Grafana, importe ces dashboards populaires via leur ID :

| Dashboard | ID | Ce qu'il montre |
|-----------|-----|-----------------|
| Node Exporter Full | 1860 | CPU, RAM, disque, réseau |
| Docker containers | 893 | Tous les conteneurs |
| Prometheus stats | 2 | Métriques internes |

**Import** → entre l'ID → sélectionne Prometheus comme source → boom.

## Étape 5 — Alertes (optionnel mais recommandé)

Crée des règles d'alerte dans `prometheus/alert_rules.yml` :

```yaml
groups:
  - name: infra
    rules:
      - alert: HighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU > 80% sur {{ $labels.instance }}"

      - alert: DiskAlmostFull
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Disque / à moins de 10% sur {{ $labels.instance }}"
```

## Ce que ça donne en production

Après quelques jours d'utilisation, tu sais exactement :
- Quels conteneurs bouffent le plus de ressources
- Si un disque va être plein dans 3 jours
- Quand ton load average est anormal
- L'état de santé de chaque service en un coup d'œil

## Aller plus loin

- **Alertmanager** — pour envoyer les alertes vers Telegram/Slack/email
- **Loki** — pour centraliser les logs (le ELK du pauvre, mais en mieux)
- **Blackbox Exporter** — pour monitorer des endpoints HTTP/TCP/ICMP externes
- **Traefik metrics** — si tu utilises Traefik comme reverse proxy

## Conclusion

Un stack Prometheus + Grafana c'est le strict minimum pour quiconque gère plus de 2 serveurs. C'est gratuit, ça tourne partout, et ça t'évite les 3h du matin "pourquoi c'est cassé ?".

Le code complet est sur mon GitHub. N'hésite pas à me contacter si tu veux un coup de main pour déployer ça chez toi.

---

*Tu veux que je t'aide à mettre en place ton monitoring ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
