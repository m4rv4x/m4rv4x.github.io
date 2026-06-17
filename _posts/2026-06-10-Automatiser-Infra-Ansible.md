---
title: "Automatiser son infra avec Ansible : guide terrain"
date: 2026-06-10 10:00 +0200
categories: ["Dev"]
tags: ["ansible", "automatisation", "devops", "linux", "sysops", "docker"]
author: marvax
---

> Tu connectes encore tes serveurs en SSH pour taper des commandes à la main ? Arrête. Ansible te permet de décrire ton infrastructure en code, de la reproduire à l'identique, et de dormir tranquille. Voici comment je l'utilise au quotidien.

<!--more-->

## Pourquoi Ansible ?

Parmi les outils IaC (Infrastructure as Code), Ansible a un avantage massif : **pas d'agent**. Tu pousses tout en SSH. Zéro daemon à installer sur les cibles.

| Outil | Agent requis | Langage | Courbe |
|-------|-------------|---------|--------|
| Ansible | Non | YAML | Douce |
| Puppet | Oui | Ruby DSL | Raide |
| Chef | Oui | Ruby | Raide |
| SaltStack | Oui (ou agentless) | YAML + Jinja | Moyenne |
| Terraform | Non | HCL | Moyenne |

## Installation

```bash
# Debian/Ubuntu
sudo apt install ansible -y

# Ou via pip (plus à jour)
pip install ansible

# Vérifier
ansible --version
```

## Structure de projet

```
infra/
├── ansible.cfg
├── inventory/
│   ├── production.yml
│   └── staging.yml
├── playbooks/
│   ├── site.yml
│   ├── docker.yml
│   └── hardening.yml
├── roles/
│   ├── common/
│   ├── docker/
│   ├── nginx/
│   └── monitoring/
└── group_vars/
    └── all.yml
```

## L'inventaire

`inventory/production.yml` :

```yaml
all:
  children:
    webservers:
      hosts:
        web01:
          ansible_host: 192.168.1.10
        web02:
          ansible_host: 192.168.1.11
    databases:
      hosts:
        db01:
          ansible_host: 192.168.1.20
  vars:
    ansible_user: marvax
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

## Premier playbook

`playbooks/site.yml` :

```yaml
---
- name: Configuration de base
  hosts: all
  become: true
  roles:
    - common

- name: Docker
  hosts: webservers
  become: true
  roles:
    - docker

- name: Nginx reverse proxy
  hosts: webservers
  become: true
  roles:
    - nginx
```

## Exemple : rôle Docker

`roles/docker/tasks/main.yml` :

```yaml
---
- name: Dépendances Docker
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present
    update_cache: yes

- name: Clé GPG Docker
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Dépôt Docker
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present

- name: Installation Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
    update_cache: yes

- name: Docker service actif
  systemd:
    name: docker
    state: started
    enabled: yes

- name: Utilisateur dans le groupe docker
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes
```

## Variables et secrets

`group_vars/all.yml` :

```yaml
---
# Variables partagées
timezone: Europe/Paris
admin_email: m4rv4x@protonmail.com
docker_compose_version: "2.24.0"
```

Pour les secrets, utilise **ansible-vault** :

```bash
# Chiffrer un fichier de vars
ansible-vault create group_vars/secrets.yml

# Éditer
ansible-vault edit group_vars/secrets.yml

# Exécuter avec le vault
ansible-playbook playbooks/site.yml --ask-vault-pass
```

## Commandes essentielles

```bash
# Ping tous les hosts
ansible all -m ping -i inventory/production.yml

# Exécuter un playbook
ansible-playbook playbooks/site.yml -i inventory/production.yml

# Dry run (check mode)
ansible-playbook playbooks/site.yml -i inventory/production.yml --check --diff

# Limiter à un host
ansible-playbook playbooks/site.yml -i inventory/production.yml --limit web01

# Tag ciblé
ansible-playbook playbooks/site.yml -i inventory/production.yml --tags docker
```

## Patterns terrain que j'utilise

### Déploiement Docker Compose

```yaml
- name: Déployer stack monitoring
  docker_compose:
    project_src: /opt/monitoring
    state: present
    pull: yes
  notify: restart monitoring
```

### Mises à jour sécurité unattended

```yaml
- name: Unattended upgrades
  apt:
    name: unattended-upgrades
    state: present

- name: Config unattended
  copy:
    src: 50unattended-upgrades
    dest: /etc/apt/apt.conf.d/50unattended-upgrades
  notify: restart unattended
```

### Rotation des logs

```yaml
- name: Logrotate pour les apps custom
  template:
    src: logrotate-app.conf.j2
    dest: "/etc/logrotate.d/{{ item }}"
  loop: "{{ logrotate_apps }}"
```

## Ce que ça change concrètement

Avant Ansible :
- 2h pour configurer un nouveau serveur
- Des configs incohérentes entre serveurs
- "ça marchait sur le mien" comme excuse

Après Ansible :
- 5 min pour provisionner un serveur identique
- Infra documentée dans le code
- Rollback possible, versionné dans Git

## Conclusion

Ansible n'est pas un outil réservé aux grandes boîtes. C'est même l'inverse : c'est les petits projets qui en bénéficient le plus, parce qu'ils n'ont pas le luxe d'avoir une équipe ops dédiée.

L'investissement initial est de quelques heures. Le ROI, c'est des dizaines d'heures sauvées chaque mois.

---

*Tu veux industrialiser ton infra ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour en discuter.*
