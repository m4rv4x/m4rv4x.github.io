---
title: "Docker security : les erreurs que tout le monde fait"
date: 2026-06-08 10:00 +0200
categories: ["Sécurité"]
tags: ["docker", "sysops", "hardening", "devops", "infrastructure"]
author: marvax
---

> Docker c'est pratique. Docker c'est rapide. Docker c'est aussi un faux sentiment de sécurité. Un conteneur n'est PAS une VM. Si tu n'as jamais audité tes Dockerfiles et ton runtime, tu as probablement des trous. Voici les 10 erreurs que je vois partout — et comment les corriger.

<!--more-->

## Erreur 1 : `docker run --privileged`

Le flag `--privileged` donne au conteneur accès à TOUS les devices de l'hôte. C'est l'équivalent de donner les clés du château.

```bash
# NON
docker run --privileged nginx

# OUI — donne uniquement ce qui est nécessaire
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

Principe du moindre privilège. Toujours.

## Erreur 2 : Tourner en root dans le conteneur

```dockerfile
# NON
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]  # ← tourne en root (UID 0)

# OUI
FROM node:20
RUN groupadd -r app && useradd -r -g app app
WORKDIR /app
COPY --chown=app:app . .
RUN npm install
USER app
CMD ["node", "server.js"]  # ← tourne en tant que 'app'
```

Si un attaquant échappe du conteneur, il sera root sur l'hôte si le process tourne en root.

## Erreur 3 : Images `:latest` en production

```bash
# NON — tu ne sais jamais ce que tu déploies
docker pull nginx:latest

# OUI — version pinnée
docker pull nginx:1.25.4-alpine
```

`:latest` peut changer à tout moment. En production, tu veux de la reproductibilité.

## Erreur 4 : Secrets dans le Dockerfile ou l'image

```dockerfile
# NON — le secret est dans chaque couche de l'image
ENV DB_PASSWORD=supersecret123

# OUI — utiliser des secrets Docker ou des variables runtime
docker run -e DB_PASSWORD="$(cat /run/secrets/db_pass)" mon-app
```

```bash
# OU via Docker secrets (Swarm)
echo "supersecret123" | docker secret create db_pass -
```

**Jamais** de mots de passe dans les Dockerfiles. Ils restent dans l'historique des couches.

## Erreur 5 : Pas de limites de ressources

```bash
# NON — un conteneur peut bouffer toute la RAM/CPU
docker run nginx

# OUI — limites explicites
docker run --memory=512m --cpus=1.0 nginx
```

Sans limites, un conteneur défaillant peut faire tomber tout l'hôte (OOM killer, CPU starvation).

## Erreur 6 : Réseau en `--net=host`

```bash
# NON — le conteneur voit tous les ports de l'hôte
docker run --net=host nginx

# OUI — réseau isolé
docker network create app_net
docker run --network app_net nginx
```

`--net=host` annifie l'isolation réseau. À éviter sauf cas très spécifiques (haute performance).

## Erreur 7 : Pas de healthcheck

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

Sans healthcheck, Docker ne sait pas si ton app est réellement prête. Un conteneur "UP" peut être dans un état cassé.

## Erreur 8 : Docker socket exposé

```bash
# NON — monte le socket Docker dans un conteneur
docker run -v /var/run/docker.sock:/var/run/docker.sock portainer

# C'est un risque accepté UNIQUEMENT si tu sais ce que tu fais
# Un attaquant dans ce conteneur contrôle TOUS les conteneurs
```

Le socket Docker, c'est le contrôle total. Ne le monte que dans des outils de confiance absolue (Portainer, Traefik avec contraintes).

## Erreur 9 : Pas de scan d'images

```bash
# Scanner une image avant de la déployer
docker scout cves nginx:1.25.4-alpine

# Ou avec Trivy
trivy image nginx:1.25.4-alpine
```

Les images Docker contiennent des CVE. Toujours scanner avant de déployer en prod.

## Erreur 10 : Logs dans le conteneur

```yaml
# docker-compose.yml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

Sans rotation, les logs remplissent le disque. Et si tu perds le conteneur, tu perds les logs.

## Checklist de sécurité Docker

- [ ] `--privileged` jamais utilisé
- [ ] Process tourne en non-root (`USER` dans Dockerfile)
- [ ] Images pinnées sur des versions exactes
- [ ] Pas de secrets dans les images
- [ ] Limites mémoire et CPU définies
- [ ] Réseau isolé (pas de `--net=host`)
- [ ] Healthchecks configurés
- [ ] Socket Docker non exposé (ou très contrôlé)
- [ ] Scan de vulnérabilités intégré au CI/CD
- [ ] Logs rotés et collectés

## Vulnérabilité récente — CUPS RCE (CVE-2026-34980 / CVE-2026-34990)

En avril 2026, deux failles critiques ont été découvertes dans CUPS (le système d'impression Linux/macOS). **Découverte par des agents IA** lors d'un audit automatisé — une première significative.

| | |
|---|---|
| **CVE** | CVE-2026-34980 + CVE-2026-34990 |
| **CVSS** | 7.5 HIGH + 7.8 HIGH |
| **Affecte** | CUPS ≤ 2.4.16 (Linux, macOS) |
| **Impact** | RCE distant sur les serveurs CUPS exposés, escalade root |
| **Fix** | Pas encore patché à la divulgation — commits publics uniquement |

La faille exploite une injection via les nouvelles lignes dans les options d'impression, permettant de re-parser des enregistrements PPD comme des commandes scheduler. Enchaînée avec la seconde CVE, un attaquant distant non authentifié peut obtenir une écriture arbitraire de fichiers root.

**Pourquoi c'est pertinent pour Docker** : beaucoup de conteneurs incluent `cupsd` sans s'en rendre compte (images de base Ubuntu, Debian). Si le service tourne dans le conteneur et que le port est exposé, la surface d'attaque est la même.

```bash
# Vérifier si CUPS tourne dans un conteneur
docker exec mon-conteneur systemctl is-active cups 2>/dev/null
docker exec mon-conteneur ss -tlnp | grep :631

# Désactiver dans le Dockerfile
RUN systemctl disable cups 2>/dev/null || true
```

Références :
- [NVD — CVE-2026-34980](https://nvd.nist.gov/vuln/detail/CVE-2026-34980)
- [NVD — CVE-2026-34990](https://nvd.nist.gov/vuln/detail/CVE-2026-34990)
- [The Register — AI agents discover CUPS RCE](https://www.theregister.com/2026/04/06/ai_agents_cups_server_rce/)

## Conclusion

La sécurité Docker ce n'est pas un truc qu'on "ajoute après". C'est dès le premier Dockerfile. Les conteneurs ne sont pas des VM — l'isolation est beaucoup plus fine. Chaque erreur listée ici a été exploitée dans des incidents réels.

Le bon réflexe : considérer chaque conteneur comme potentiellement compromis, et limiter ce qu'il peut faire.

---

*Besoin d'un audit de sécurité de ton infrastructure Docker ? [Écris-moi](mailto:m4rv4x@protonmail.com).*
