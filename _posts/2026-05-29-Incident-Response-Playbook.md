---
title: "Incident Response Playbook : que faire quand ça pète"
date: 2026-05-29 10:00 +0200
categories: ["Sécurité"]
tags: ["incident-response", "playbook", "sécurité", "sysops", "forensics", "opsec", "infrastructure"]
author: marvax
---

> Un incident de sécurité, c'est pas une question de "si" mais de "quand". Ce playbook te donne une méthodologie structurée en 5 phases pour gérer un incident de bout en bout : détection, confinement, éradication, récupération, et retour d'expérience. Avec des checklists concrètes et des outils concrets.

<!--more-->

## Pourquoi un playbook ?

Quand ça pète, le stress prend le dessus. Tu oublies des étapes, tu prends des décisions à l'arrache, tu effaces des preuves sans le vouloir. Un playbook, c'est ton GPS dans le chaos. Tu le prépares **avant** l'incident, pas pendant.

Les incidents qu'on a couverts sur ce blog — [CrowdStrike](/posts/2024-07-19-CrowdStrike-ecran-bleu-mondial/), [Log4Shell](/posts/2021-12-09-Log4Shell-lincendie-dans-la-jvm/), [LastPass](/posts/2022-12-22-LastPass-coffre-fort-trahi/) — montrent tous la même chose : les organisations qui s'en sortent le mieux sont celles qui avaient un plan.

## Phase 1 : Détection

L'incident est là. Première étape : confirmer que c'est réel et évaluer l'ampleur.

### Checklist Détection

- [ ] Confirmer l'alerte (faux positif ou vrai incident ?)
- [ ] Identifier le type d'incident (intrusion, ransomware, fuite de données, DDoS, supply chain…)
- [ ] Déterminer les systèmes affectés
- [ ] Horodater la première détection
- [ ] Identifier le vecteur d'attaque si possible
- [ ] Notifier l'équipe de réponse (interne ou prestataire)

### Outils de détection

| Outil | Usage |
|---|---|
| [Prometheus + Grafana](/posts/2026-06-15-Monitoring-Prometheus-Grafana/) | Métriques, alertes sur anomalies |
| [CrowdSec](/posts/2026-01-16-CrowdSec-fail2ban-futur/) | Détection d'attaques en temps réel |
| [Nuclei](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/) | Scan de vulnérabilités post-détection |
| [OSINT](/posts/2026-06-10-OSINT-reconnaissance-ouverte/) | Vérification de fuite sur sources ouvertes |
| Logs centralisés (ELK, Loki) | Corrélation d'événements |

### Décision clé

**Est-ce que l'incident nécessite un confinement immédiat ?**

Si le système est compromis en temps réel (exfiltration active, ransomware en cours) → passe directement à la Phase 2. Sinon, prends le temps de documenter.

## Phase 2 : Confinement

Objectif : limiter les dégâts. Isoler les systèmes compromis sans détruire les preuves.

### Checklist Confinement

- [ ] Isoler le réseau des systèmes compromis (VLAN, firewall, déconnexion)
- [ ] **Ne pas éteindre** la machine (tu perds la mémoire volatile) — préfère un snapshot mémoire
- [ ] Bloquer les IPs/domaines malveillants au niveau [UFW/iptables](/posts/2026-06-18-UFW-iptables-Firewall-Linux/)
- [ ] Révoquer les credentials potentiellement compromis (tokens, clés API, mots de passe)
- [ ] Activer la collecte de logs renforcée
- [ ] Sauvegarder les logs actuels (ils seront écrasés sinon)
- [ ] Identifier les comptes compromis et les verrouiller

### Confinement court terme vs long terme

**Court terme** (heures) : isolation réseau, blocage IP, désactivation de comptes.

**Long terme** (jours) : segmentation réseau, mise en place de [CrowdSec](/posts/2026-01-16-CrowdSec-fail2ban-futur/) renforcé, patch des vulnérabilités.

```bash
# Snapshot mémoire d'urgence (Linux)
sudo dd if=/dev/mem of=/mnt/external/memdump.raw bs=1M

# Capture réseau en cours
sudo tcpdump -i eth0 -w /mnt/external/capture_$(date +%Y%m%d_%H%M%S).pcap

# Lister les connexions actives
ss -tunap > /mnt/external/connexions_actives.txt
```

## Phase 3 : Éradication

Tu as isolé le problème. Maintenant, tu nettoies.

### Checklist Éradication

- [ ] Identifier la cause racine (vulnérabilité exploitée, phishing, config fautive…)
- [ ] Supprimer les backdoors, webshells, comptes non autorisés
- [ ] Patcher la vulnérabilité exploitée (voir [scan Nuclei](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/))
- [ ] Reconstruire les systèmes compromis à partir d'une image saine
- [ ] Vérifier l'intégrité des fichiers système (`aide`, `tripwire`, `rpm -Va`)
- [ ] Scanner l'ensemble de l'infra pour d'autres compromissions
- [ ] Vérifier les [chaînes d'approvisionnement](/posts/2026-02-05-Supply-Chain-Attacks-2026/)

### Forensics : ne pas tout effacer

Avant de reconstruire, **image le disque** :

```bash
# Image forensique du disque
sudo dd if=/dev/sda of=/mnt/external/disk_image.dd bs=4M status=progress

# Hash de l'image pour intégrité
sha256sum /mnt/external/disk_image.dd > /mnt/external/disk_image.sha256
```

Garde ces images. Elles serviront pour l'analyse post-incident et éventuellement pour des poursuites judiciaires.

## Phase 4 : Récupération

Tu remets en service. Progressivement, pas d'un coup.

### Checklist Récupération

- [ ] Restaurer les systèmes à partir de backups vérifiés (pas de backup ? → [BorgBackup](/posts/2026-03-13-BorgBackup-strategie-backup/))
- [ ] Appliquer tous les patches de sécurité
- [ ] Réinitialiser **tous** les mots de passe et clés (pas seulement ceux compromis)
- [ ] Activer l'authentification forte ([passkeys/FIDO2](/posts/2026-02-07-Passkeys-FIDO2-mort-mdp/))
- [ ] Remettre en production par étapes (canary deployment)
- [ ] Surveiller intensivement pendant 72h minimum
- [ ] Vérifier que les [reverse proxies](/posts/2026-06-05-Reverse-Proxy-Nginx-TLS/) et [configs Docker](/posts/2026-06-08-Docker-Security-Erreurs/) sont sains

### Priorisation du redémarrage

| Priorité | Systèmes | Délai |
|---|---|---|
| P0 | Authentification, DNS, VPN | 0-4h |
| P1 | Services clients, API | 4-12h |
| P2 | Monitoring, CI/CD | 12-24h |
| P3 | Outils internes, dashboard | 24-72h |

## Phase 5 : Retour d'expérience (Lessons Learned)

La phase la plus importante et la plus souvent sautée. Ne la skip pas.

### Checklist Retour d'expérience

- [ ] Organiser un debrief dans les 5 jours ouvrés
- [ ] Documenter la timeline complète de l'incident
- [ ] Identifier ce qui a bien fonctionné
- [ ] Identifier ce qui a foiré
- [ ] Mettre à jour le playbook avec les leçons apprises
- [ ] Former l'équipe sur les nouveaux processus
- [ ] Mettre en place les corrections techniques (automatisation, monitoring renforcé)
- [ ] Communication aux parties prenantes (clients, autorités si RGPD)

### Template de rapport d'incident

```markdown
# Rapport d'incident — [ID]

## Résumé
- **Date de détection** : YYYY-MM-DD HH:MM
- **Date de résolution** : YYYY-MM-DD HH:MM
- **Durée totale** : Xh Xm
- **Sévérité** : Critique / Haute / Moyenne / Basse
- **Type** : Intrusion / Ransomware / Fuite / DDoS / Supply Chain

## Timeline
| Heure | Événement |
|---|---|
| HH:MM | Première alerte détectée |
| HH:MM | Confinement initié |
| HH:MM | Éradication terminée |
| HH:MM | Systèmes restaurés |

## Cause racine
[Description technique]

## Impact
- Systèmes affectés : [...]
- Données compromises : [...]
- Coût estimé : [...]

## Actions correctives
1. [...]
2. [...]

## Leçons apprises
- [...]
```

## Cas réels couverts sur le blog

Chaque incident majeur apporte des leçons. Voici ceux qu'on a décortiqués :

- **[CrowdStrike 2024](/posts/2024-07-19-CrowdStrike-ecran-bleu-mondial/)** — Un mauvais update peut paralyser des millions de machines. Leçon : rollback automatique, déployer par vagues.
- **[Log4Shell 2021](/posts/2021-12-09-Log4Shell-lincendie-dans-la-jvm/)** — Vulnérabilité dans une dépendance omniprésente. Leçon : inventaire des dépendances, SBOM.
- **[LastPass 2022](/posts/2022-12-22-LastPass-coffre-fort-trahi/)** — Compromission du stockage de vaults. Leçon : chiffrement de bout en bout, zero-knowledge.
- **[n8n RCE 2026](/posts/2026-01-07-n8n-RCE-critique/)** — Exécution de code à distance via un outil d'automatisation. Leçon : ne pas exposer d'outils d'admin sur Internet.
- **[Hygiène post-fuite](/posts/2026-06-17-Guide-hygiene-post-fuite/)** — Guide pratique pour nettoyer après une fuite de données.

## Outils indispensables dans ta boîte à outils IR

```bash
# Capture réseau
tcpdump, tshark (Wireshark CLI), netsniff-ng

# Analyse disque
autopsy, sleuthkit, volatility3 (mémoire)

# Scan de vulnérabilités
nuclei, openvas, nmap

# Logs
journalctl, lnav, logsurfer

# Collecte d'artefacts
ir-responder, linux-explorer, UAC (Unix-like Artifacts Collector)
```

Intègre ces outils dans ton [pipeline d'automatisation](/posts/2026-06-10-Automatiser-Infra-Ansible/) pour qu'ils soient prêts quand tu en auras besoin.

## Prévention > Réaction

Le meilleur incident, c'est celui qui n'arrive pas. Voire mesures préventives essentielles :

- [Hardening SSH](/posts/2026-06-12-Hardening-SSH/) — réduire la surface d'attaque
- [CrowdSec + Fail2ban](/posts/2026-06-21-Fail2ban-avance/) — détection proactive
- [Sécurité Docker](/posts/2026-06-08-Docker-Security-Erreurs/) — ne pas exposer le socket
- [Monitoring](/posts/2026-06-15-Monitoring-Prometheus-Grafana/) — détecter avant que ça devienne critique
- [BorgBackup](/posts/2026-03-13-BorgBackup-strategie-backup/) — pouvoir restaurer quand tout pète

---

*Références :*

- [NIST SP 800-61 — Computer Security Incident Handling Guide](https://csf.tools/reference/nist-sp-800-61/)
- [CrowdStrike 2024](/posts/2024-07-19-CrowdStrike-ecran-bleu-mondial/)
- [Log4Shell](/posts/2021-12-09-Log4Shell-lincendie-dans-la-jvm/)
- [LastPass](/posts/2022-12-22-LastPass-coffre-fort-trahi/)
- [CrowdSec](/posts/2026-01-16-CrowdSec-fail2ban-futur/)
- [Nuclei scan](/posts/2026-03-08-Nuclei-OpenVAS-scan-vuln/)
- [BorgBackup](/posts/2026-03-13-BorgBackup-strategie-backup/)
- [OSINT](/posts/2026-06-10-OSINT-reconnaissance-ouverte/)
- [Monitoring Prometheus/Grafana](/posts/2026-06-15-Monitoring-Prometheus-Grafana/)
- [Hygiène post-fuite](/posts/2026-06-17-Guide-hygiene-post-fuite/)

*Besoin d'un audit de ta posture de réponse aux incidents ? [Contacte-moi](mailto:m4rv4x@protonmail.com).*
