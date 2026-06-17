---
title: "CI/CD avec GitHub Actions : déploys automatisés"
date: 2026-06-20 10:00 +0200
categories: ["Dev"]
tags: ["ci-cd", "github-actions", "automatisation", "devops", "docker", "déploiement", "infrastructure"]
author: marvax
---

> Si tu déploies encore en SSH à la main, c'est le moment de passer à la CI/CD. GitHub Actions est gratuit pour les repos publics et suffisant pour la plupart des projets. Voici des workflows concrets que j'utilise au quotidien.

<!--more-->

## CI/CD en 30 secondes

**CI** (Continuous Integration) : à chaque push, tester automatiquement le code.
**CD** (Continuous Deployment) : si les tests passent, déployer automatiquement.

```
git push → Tests → Build → Deploy → Production
```

## Workflow de base

`.github/workflows/ci.yml` :

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'

      - name: Install deps
        run: pip install -e ".[dev]"

      - name: Lint
        run: ruff check .

      - name: Test
        run: pytest -v
```

## Build + Deploy Docker

`.github/workflows/deploy.yml` :

```yaml
name: Deploy

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'docker-compose.yml'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} myapp:latest

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/myapp
            docker compose pull
            docker compose up -d
```

## Workflow multi-environnement

```yaml
name: Deploy Staging → Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pytest -v

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: echo "Deploying to staging..."
        # ... deploy commands

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: echo "Deploying to production..."
        # ... deploy commands
```

## Secrets et variables

```bash
# Via l'interface GitHub
# Settings → Secrets and variables → Actions → New repository secret

# Ou via CLI
gh secret set SERVER_HOST --body "10.10.11.XX"
gh secret set SSH_KEY --body "$(cat ~/.ssh/id_ed25519)"
```

```yaml
# Utilisation dans le workflow
- name: Deploy
  env:
    DB_URL: ${{ secrets.DATABASE_URL }}
    API_KEY: ${{ secrets.API_KEY }}
  run: ./deploy.sh
```

## Cache des dépendances

```yaml
# Python — cache pip
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'

# Node — cache npm
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

# Docker — cache des couches
- uses: docker/build-push-action@v5
  with:
    context: .
    push: false
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Scheduled workflows

```yaml
# Backup quotidien à 3h du matin
name: Daily Backup

on:
  schedule:
    - cron: '0 3 * * *'

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup database
        run: |
          ssh ${{ secrets.SERVER_HOST }} "pg_dump mydb | gzip > /backups/$(date +%Y%m%d).sql.gz"
```

## Notifications

```yaml
# Notification Telegram quand le deploy échoue
- name: Notify on failure
  if: failure()
  uses: appleboy/telegram-action@v1
  with:
    to: ${{ secrets.TELEGRAM_CHAT_ID }}
    token: ${{ secrets.TELEGRAM_BOT_TOKEN }}
    message: |
      ❌ Deploy failed!
      Repo: ${{ github.repository }}
      Commit: ${{ github.sha }}
      Author: ${{ github.actor }}
```

## Bonnes pratiques

- **Pin les versions** : `actions/checkout@v4` pas `@main`
- **Limite les permissions** : `permissions: contents: read` par défaut
- **Path filters** : ne pas rebuild si seuls les docs changent
- **Concurrency** : annuler les runs précédents sur le même branch
- **Environment protection** : require approval pour la prod

```yaml
# Exemple de permissions minimales
permissions:
  contents: read
  packages: write

# Concurrency — un seul deploy par branch
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

---

*Références :*
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Actions marketplace](https://github.com/marketplace?type=actions)

*Tu veux mettre en place une CI/CD pour ton projet ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
