---
title: Cheatsheets
icon: fas fa-terminal
order: 5
permalink: /cheatsheets/
---

# Cheatsheets

Quick references techniques — les commandes que tu oublies toujours au pire moment.

---

## Linux

### Recherche & fichiers
```bash
# Trouver un fichier par nom
find / -name "*.conf" -type f 2>/dev/null

# Chercher dans le contenu
grep -rn "password" /etc/ --include="*.conf"

# Derniers fichiers modifies
find . -type f -mmin -60

# Taille des dossiers
du -sh /* | sort -rh | head -10

# Lister les ports en ecoute
ss -tlnp

# Lignes d'un fichier
wc -l fichier.txt

# Supprimer les doublons (preservant l'ordre)
awk '!seen[$0]++' fichier.txt

# Remplacer dans tous les fichiers
sed -i 's/ancien/nouveau/g' *.conf

# Couper une colonne
awk -F':' '{print $1}' /etc/passwd

# Trier par taille
ls -lhS /var/log/
```

### Systeme
```bash
# CPU / RAM / Disk en un coup d'oeil
htop
df -h
free -m

# Info systeme complete
uname -a
cat /etc/os-release

# Uptime + load average
uptime

# Services systemd
systemctl status nginx
journalctl -u nginx --since "1 hour ago" -f
systemctl list-units --failed

# Redemarrer un service
systemctl restart docker

# Logs en temps reel
tail -f /var/log/syslog
dmesg -w

# Kernel messages
dmesg | grep -i error
dmesg | grep -i usb

# Temperatures (si lm-sensors installe)
sensors

# Qui est connecte
who
w
last -10
```

### Permissions
```bash
# Changer proprietaire
chown -R user:group /path

# Permissions recursives
chmod -R 755 /path

# Trouver les fichiers SUID
find / -perm -4000 -type f 2>/dev/null

# Trouver les fichiers world-writable
find / -perm -o+w -type f 2>/dev/null

# Trouver les fichiers sans proprietaire
find / -nouser -o -nogroup 2>/dev/null

# ACL avancees
getfacl fichier.txt
setfacl -m u:marvax:rwx fichier.txt

# Sticky bit
chmod +t /dossier/partage

# Capabilities Linux
getcap /usr/bin/*
setcap cap_net_raw+ep /usr/bin/nmap
```

### Processus
```bash
# Trouver un processus
ps aux | grep nginx
pgrep -a nginx

# Tuer par nom
pkill -f "process_name"

# Tuer par PID
kill -9 1234

# Processus qui utilisent le plus de RAM/CPU
ps aux --sort=-%mem | head -10
ps aux --sort=-%cpu | head -10

# Lister les fichiers ouverts par un processus
lsof -p 1234
lsof -i :80

# Strace : tracer les appels systeme
strace -p 1234 -e trace=open,read,write
```

### Cron & timers
```bash
# Editer le crontab
crontab -e

# Lister les crontabs
crontab -l

# Format : minute heure jour_mois mois jour_semaine commande
# Tous les jours a 3h
0 3 * * * /usr/local/bin/backup.sh

# Toutes les 5 minutes
*/5 * * * * /usr/local/bin/check.sh

# Systemd timer (plus moderne)
systemctl list-timers --all
systemctl enable --now mon-timer.timer
```

### LVM & disques
```bash
# Lister les disques
lsblk
fdisk -l

# Monter un disque
mount /dev/sdb1 /mnt/data
# Demontage
umount /mnt/data

# Monter automatiquement (fstab)
echo "/dev/sdb1 /mnt/data ext4 defaults 0 2" >> /etc/fstab

# LVM : creer un volume logique
pvcreate /dev/sdb
vgcreate mon_vg /dev/sdb
lvcreate -L 50G -n mon_lv mon_vg

# LVM : agrandir un volume
lvextend -L +10G /dev/mon_vg/mon_lv
resize2fs /dev/mon_vg/mon_lv

# Verifier l'espace disque
df -hT
```

---

## SSH

### Connexion
```bash
# Connexion basique
ssh user@host

# Port specifique
ssh -p 2222 user@host

# Avec une cle specifique
ssh -i ~/.ssh/id_ed25519 user@host

# Jump host (bastion)
ssh -J bastion@jump-host user@target

# Forward de port local (localhost:8080 -> remote:80)
ssh -L 8080:localhost:80 user@host

# Forward de port distant (remote:9090 -> local:3000)
ssh -R 9090:localhost:3000 user@host

# SOCKS proxy
ssh -D 1080 user@host

# Session persistante (evite les deconnexions)
ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=3 user@host
```

### Cles SSH
```bash
# Generer une cle Ed25519 (recommande)
ssh-keygen -t ed25519 -C "marvax@machine"

# Generer une cle RSA (legacy)
ssh-keygen -t rsa -b 4096

# Copier la cle publique sur le serveur
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# Agent forwarding
ssh -A user@host

# Fingerprints
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

### SSHFS (montage distant)
```bash
# Monter un dossier distant
sshfs user@host:/remote/path /mnt/local

# Demontage
fusermount -u /mnt/local
```

### Hardening (resumé)
```bash
# /etc/ssh/sshd_config
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers marvax
X11Forwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

---

## Ansible

### Commandes de base
```bash
# Ping tous les hosts
ansible all -m ping

# Ping un groupe
ansible webservers -m ping

# Executer une commande ad-hoc
ansible all -m shell -a "uptime"
ansible all -m shell -a "df -h" --become

# Executer un playbook
ansible-playbook site.yml

# Dry run (check mode)
ansible-playbook site.yml --check

# Limiter a un host
ansible-playbook site.yml --limit server1

# Verbose
ansible-playbook site.yml -vvv

# Lister les hosts d'un inventaire
ansible-inventory --list -y
```

### Roles & collections
```bash
# Installer un role
ansible-galaxy install geerlingguy.nginx

# Installer depuis requirements.yml
ansible-galaxy install -r requirements.yml

# Lister les roles installes
ansible-galaxy list

# Installer une collection
ansible-galaxy collection install community.general
```

### Patterns d'inventaire
```bash
# Tous les hosts
ansible all -m ping

# Groupe
ansible webservers -m ping

# Intersection
ansible 'webservers:&staging' -m ping

# Exclusion
ansible 'webservers:!staging' -m ping

# Par regex
ansible '~web.*' -m ping
```

---

## Nginx

### Config basique reverse proxy
```nginx
server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP -> HTTPS redirect
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}
```

### Commandes
```bash
# Test de configuration
nginx -t

# Reload sans arreter
nginx -s reload

# Arreter / demarrer
systemctl stop nginx
systemctl start nginx
systemctl status nginx

# Logs en temps reel
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Compter les connexions actives
ss -s | grep estab | grep :443

# Top des IPs par nombre de requetes
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Top des URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20
```

### Securite headers
```nginx
# Dans le bloc server {}
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### Rate limiting
```nginx
# Dans le bloc http {}
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

# Dans le bloc server/location
limit_req zone=api burst=20 nodelay;
```

---

## Let's Encrypt / Certbot

```bash
# Installer certbot
apt install certbot python3-certbot-nginx

# Obtenir un certificat (nginx plugin)
certbot --nginx -d example.com -d www.example.com

# Obtenir un certificat (standalone)
certbot certonly --standalone -d example.com

# Obtenir un certificat (webroot)
certbot certonly --webroot -w /var/www/html -d example.com

# Lister les certificats
certbot certificates

# Renouveler tous les certificats
certbot renew

# Renouveler un certificat specifique
certbot renew --cert-name example.com

# Test du renouvellement
certbot renew --dry-run

# Verifier l'expiration
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -enddate -noout

# Cron de renouvellement (souvent deja installe)
0 3 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

---

## CrowdSec

```bash
# Installer
curl -s https://install.crowdsec.net | sudo sh
sudo apt install crowdsec crowdsec-firewall-bouncer-iptables

# Verifier le service
systemctl status crowdsec
systemctl status crowdsec-firewall-bouncer

# Lister les decisions (blocages)
cscli decisions list

# Lister les scenarios installes
cscli scenarios list

# Lister les bouncers
cscli bouncers list

# Mettre a jour les collections
cscli collections upgrade crowdsecurity/linux

# Ajouter manuellement un blocage
cscli decisions add -i 1.2.3.4 -d 24h -r "manual ban"

# Supprimer un blocage
cscli decisions delete -i 1.2.3.4

# Voir les alertes
cscli alerts list

# Dashboard (optionnel)
cscli dashboard setup
```

---

## Fail2ban

```bash
# Status general
fail2ban-client status

# Status d'une jail
fail2ban-client status sshd

# Ban manuel
fail2ban-client set sshd banip 1.2.3.4

# Unban
fail2ban-client set sshd unbanip 1.2.3.4

# Lister les IPs bannies
fail2ban-client banned

# Recharger la config
fail2ban-client reload

# Logs
tail -f /var/log/fail2ban.log

# /etc/fail2ban/jail.local (exemple)
cat << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
EOF
```

---

## Docker

### Container management
```bash
# Lister les conteneurs (actifs + stops)
docker ps -a

# Logs en temps reel
docker logs -f --tail 100 container_name

# Shell dans un conteneur
docker exec -it container_name /bin/bash
docker exec -it container_name sh  # si pas de bash

# Arreter / supprimer tout
docker stop $(docker ps -aq)
docker system prune -af --volumes

# Stats en temps reel
docker stats

# Redemarrer un conteneur
docker restart container_name

# Copier un fichier depuis/vers un conteneur
docker cp container_name:/path/file ./local_file
docker cp ./local_file container_name:/path/file

# Voir l'IP d'un conteneur
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Voir les variables d'environnement
docker exec container_name env
```

### Images
```bash
# Build
docker build -t monapp:latest .

# Build avec un Dockerfile specifique
docker build -f Dockerfile.prod -t monapp:prod .

# Tag & push
docker tag monapp:latest registry.example.com/monapp:latest
docker push registry.example.com/monapp:latest

# Pull
docker pull nginx:alpine

# Inspect
docker inspect container_name
docker history image_name

# Taille des images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Scanner une image
trivy image monapp:latest
grype monapp:latest
```

### Docker Compose
```bash
# Lancer
docker compose up -d

# Rebuild sans cache
docker compose build --no-cache
docker compose up -d --force-recreate

# Logs
docker compose logs -f service_name

# Exec dans un service
docker compose exec service_name sh

# Arreter tout
docker compose down -v  # -v supprime les volumes

# Lister les services
docker compose ps

# Scale un service
docker compose up -d --scale web=3
```

### Reseau Docker
```bash
# Lister les reseaux
docker network ls

# Creer un reseau
docker network create mon_reseau

# Connecter un conteneur a un reseau
docker network connect mon_reseau container_name

# Inspecter un reseau
docker network inspect mon_reseau

# DNS inter-conteneurs
docker run --name api --network mon_reseau monapp
# => accede par http://api depuis le meme reseau
```

### Securite Docker
```bash
# Scanner une image
trivy image monapp:latest

# Lister les capabilities d'un conteneur
docker inspect --format='{{.HostConfig.CapAdd}}' container_name

# Voir les process dans un conteneur
docker top container_name

# Lire les logs de securite
docker events --filter type=container

# Container en read-only
docker run --read-only --tmpfs /tmp monapp

# Pas de privilege
docker run --security-opt=no-new-privileges monapp

# Limiter les ressources
docker run --memory=512m --cpus=1 monapp
```

---

## Git

### Quotidien
```bash
# Status / log
git status
git log --oneline --graph -20

# Commit & push
git add -A && git commit -m "msg"
git push origin main

# Pull avec rebase (evite les merge commits)
git pull --rebase origin main

# Stash temporaire
git stash
git stash pop
git stash list

# Voir les changements d'un commit
git show abc1234

# Voir les changements staged
git diff --cached

# Voir les changements non staged
git diff
```

### Branches
```bash
# Creer et switcher
git checkout -b feature/nouvelle-branche

# Lister les branches
git branch -a

# Fusionner
git checkout main
git merge feature/nouvelle-branche

# Supprimer une branche
git branch -d feature/nouvelle-branche
git push origin --delete feature/nouvelle-branche

# Renommer une branche
git branch -m ancien_nom nouveau_nom

# Cherrypick un commit
git cherry-pick abc1234
```

### Undo
```bash
# Dernier commit (garder les changements)
git reset --soft HEAD~1

# Dernier commit (tout supprimer)
git reset --hard HEAD~1

# Fichier specifique
git checkout -- fichier.txt

# Revert sans perdre l'historique
git revert HEAD

# Ammender le dernier commit
git commit --amend -m "nouveau message"

# Supprimer un fichier du tracking (sans le supprimer)
git rm --cached fichier.txt

# Recuperer un fichier supprime
git checkout HEAD -- fichier.txt
```

### Tags & releases
```bash
# Creer un tag
git tag -a v1.0.0 -m "Release 1.0.0"

# Push un tag
git push origin v1.0.0

# Push tous les tags
git push origin --tags

# Lister les tags
git tag -l
```

### Submodules
```bash
# Ajouter un submodule
git submodule add https://github.com/user/repo.git path/to/repo

# Initialiser les submodules (apres clone)
git submodule update --init --recursive

# Mettre a jour les submodules
git submodule update --remote
```

---

## Nmap

### Scans courants
```bash
# Scan rapide des ports courants
nmap -sV -sC -T4 target

# Scan complet de tous les ports
nmap -p- -T4 target

# Scan UDP
nmap -sU -T4 --top-ports 100 target

# Detection de version + scripts
nmap -sV -sC -O target

# Scan furtif (SYN)
nmap -sS -T2 target

# Scan depuis un fichier de cibles
nmap -iL targets.txt -sV

# Scan avec decoys
nmap -D RND:10 target

# Ping sweep (decouverte d'hotes)
nmap -sn 192.168.1.0/24
```

### Scripts NSE
```bash
# Vulnerabilites
nmap --script vuln target

# Enum SMB
nmap --script smb-enum-shares,smb-enum-users target

# Enum HTTP
nmap --script http-enum,http-title target

# Brute force
nmap --script ssh-brute target

# Tous les scripts "safe"
nmap --script "safe" target

# Scripts par categorie
nmap --script "discovery" target
```

### Output
```bash
# Format normal
nmap -oN scan.txt target

# Format XML
nmap -oX scan.xml target

# Format grepable
nmap -oG scan.grep target

# Tous les formats
nmap -oA scan target
```

---

## Nuclei

```bash
# Scan basique
nuclei -u https://target

# Scan avec tous les templates
nuclei -u https://target -as

# Scan par severite
nuclei -u https://target -severity critical,high

# Scan depuis un fichier de cibles
nuclei -l urls.txt

# Mettre a jour les templates
nuclei -update-templates

# Scan avec un template specifique
nuclei -u https://target -t cves/2024/CVE-2024-xxxx.yaml

# Lister les templates disponibles
nuclei -tl | head -50

# Scan avec rate limit
nuclei -u https://target -rl 10 -bs 5

# Output JSON
nuclei -u https://target -json -o results.json

# Scan rapide (tags specifiques)
nuclei -u https://target -tags cve,xss,sqli
```

---

## Networking

### tcpdump
```bash
# Capturer sur une interface
tcpdump -i eth0

# Filtrer par host
tcpdump -i eth0 host 192.168.1.1

# Filtrer par port
tcpdump -i eth0 port 443

# Filtrer par protocole
tcpdump -i eth0 icmp

# Capturer le DNS
tcpdump -i eth0 port 53

# Sauvegarder en pcap
tcpdump -i eth0 -w capture.pcap

# Lire un pcap
tcpdump -r capture.pcap

# Capturer les paquets HTTP
tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# Nombre limite de paquets
tcpdump -i eth0 -c 100
```

### iptables / UFW
```bash
# UFW basique
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 443/tcp
ufw enable
ufw status verbose

# UFW : supprimer une regle
ufw status numbered
ufw delete 3

# UFW : autoriser un sous-reseau
ufw allow from 192.168.1.0/24

# iptables : lister les regles
iptables -L -n -v

# iptables : bloquer une IP
iptables -A INPUT -s 1.2.3.4 -j DROP

# iptables : NAT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# iptables : supprimer une regle
iptables -D INPUT -s 1.2.3.4 -j DROP

# iptables : sauvegarder / restaurer
iptables-save > /etc/iptables.rules
iptables-restore < /etc/iptables.rules
```

### SSL / TLS
```bash
# Tester un certificat
openssl s_client -connect example.com:443

# Voir les details d'un certificat
openssl x509 -in cert.pem -text -noout

# Verifier l'expiration
openssl x509 -in cert.pem -enddate -noout

# Generer un certificat auto-signe
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Generer une CSR
openssl req -new -newkey rsa:4096 -keyout key.pem -out csr.pem -nodes

# Verifier une cle privee
openssl rsa -in key.pem -check

# Tester les cipher suites
nmap --script ssl-enum-ciphers -p 443 example.com

# Test SSL Labs en ligne (via CLI)
curl -s "https://api.ssllabs.com/api/v3/analyze?host=example.com&publish=off&all=done" | jq '.endpoints[0].grade'
```

### curl avance
```bash
# Headers de reponse
curl -sI https://example.com

# Suivre les redirections
curl -sL https://example.com

# Methode POST avec JSON
curl -s -X POST https://api.example.com/data \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'

# Avec authentification Bearer
curl -s -H "Authorization: Bearer TOKEN" https://api.example.com

# Upload de fichier
curl -s -F "file=@./fichier.pdf" https://upload.example.com

# Mesurer le temps de reponse
curl -s -o /dev/null -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\n" https://example.com

# Proxy SOCKS
curl -s --socks5-hostname localhost:1080 https://example.com
```

### DNS
```bash
# Query basique
dig example.com
dig +short example.com

# Query un type specifique
dig MX example.com
dig TXT example.com
dig NS example.com
dig AAAA example.com

# Query via un serveur specifique
dig @8.8.8.8 example.com

# Reverse DNS
dig -x 1.2.3.4

# Trace complete
dig +trace example.com

# Whois
whois example.com
```

---

## Prometheus & Grafana

### PromQL (requetes courantes)
```promql
# CPU usage par instance
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# RAM usage %
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage %
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100

# Taux de requetes HTTP (par status code)
sum(rate(http_requests_total[5m])) by (code)

# Latence P99
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Taux d'erreurs (5xx)
sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Espace disque restant
node_filesystem_avail_bytes{mountpoint="/"} / 1024 / 1024 / 1024
```

### Commandes
```bash
# Status de Prometheus
systemctl status prometheus

# Recharger la config (sans restart)
curl -X POST http://localhost:9090/-/reload

# Verifier la config
promtool check config /etc/prometheus/prometheus.yml

# Verifier les regles
promtool check rules /etc/prometheus/rules/*.yml

# Query depuis la CLI
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result[] | {instance: .metric.instance, value: .value[1]}'
```

---

## WireGuard

```bash
# Installer
apt install wireguard

# Generer les cles
wg genkey | tee privatekey | wg pubkey > publickey

# Config serveur (/etc/wireguard/wg0.conf)
cat << 'EOF'
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
EOF

# Config client
cat << 'EOF'
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = server.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

# Demarrer / arreter
wg-quick up wg0
wg-quick down wg0

# Activer au boot
systemctl enable wg-quick@wg0

# Status
wg show
wg show wg0

# Ajouter un peer
wg set wg0 peer <CLIENT_PUBKEY> allowed-ips 10.0.0.3/32
```

---

## BorgBackup

```bash
# Initialiser un repo
borg init --encryption=repokey /mnt/backup/borg

# Creer un backup
borg create --stats --progress /mnt/backup/borg::archive-$(date +%Y%m%d) /home /etc

# Avec compression
borg create --compression zstd,6 /mnt/backup/borg::archive-$(date +%Y%m%d) /home

# Lister les archives
borg list /mnt/backup/borg

# Lister le contenu d'une archive
borg list /mnt/backup/borg::archive-20260708

# Restaurer
borg extract /mnt/backup/borg::archive-20260708

# Restaurer un fichier specifique
borg extract /mnt/backup/borg::archive-20260708 home/user/important.txt

# Monter le backup (FUSE)
borg mount /mnt/backup/borg /mnt/borg-mount
fusermount -u /mnt/borg-mount

# Prune (retention)
borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6 /mnt/backup/borg

# Verifier l'integrite
borg check /mnt/backup/borg

# Export la cle
borg key export /mnt/backup/borg /root/borg-key-backup.txt

# Info du repo
borg info /mnt/backup/borg

# Cron (quotidien a 3h)
echo '0 3 * * * borg create --compression zstd,6 /mnt/backup/borg::daily-$(date +\%Y\%m\%d) /home /etc && borg prune --keep-daily=7 --keep-weekly=4 /mnt/backup/borg' | crontab -
```

---

## Vaultwarden

```bash
# Docker Compose
cat << 'EOF'
version: "3"
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.example.com"
      SIGNUPS_ALLOWED: "false"
      ADMIN_TOKEN: "<random_token>"
      SMTP_HOST: "smtp.example.com"
      SMTP_FROM: "vault@example.com"
      SMTP_PORT: 587
      SMTP_SECURITY: "starttls"
      SMTP_USERNAME: "vault@example.com"
      SMTP_PASSWORD: "password"
    volumes:
      - ./vw-data:/data
    ports:
      - "127.0.0.1:8080:80"
EOF

# Generer un admin token
openssl rand -base64 48

# Backup
tar czf vaultwarden-backup-$(date +%Y%m%d).tar.gz ./vw-data/

# Restore
tar xzf vaultwarden-backup-20260708.tar.gz -C ./
docker compose restart
```

---

## GPG / Chiffrement

```bash
# Generer une cle
gpg --full-generate-key

# Lister les cles
gpg --list-keys
gpg --list-secret-keys

# Exporter la cle publique
gpg --armor --export user@example.com > publickey.asc

# Importer une cle publique
gpg --import publickey.asc

# Chiffrer un fichier
gpg -e -r user@example.com fichier.txt

# Dechiffrer
gpg -d fichier.txt.gpg

# Signer un fichier
gpg --sign fichier.txt

# Verifier une signature
gpg --verify fichier.txt.sig

# Chiffrer symetriquement (mot de passe)
gpg -c fichier.txt

# Chiffrer avec age (plus moderne)
age -r age1publickey... -o fichier.txt.age fichier.txt
age -d fichier.txt.age
```

---

## OSINT

### Google Dorks
```
site:example.com filetype:pdf
site:example.com intitle:"index of"
site:example.com ext:sql | ext:dbf | ext:mdb
site:example.com inurl:admin | inurl:login
site:example.com "password" | "passwd" | "pwd"
site:pastebin.com "example.com"
site:github.com "example.com" password
```

### Shodan
```bash
# Recherche basique
shodan search "org:Example port:22"

# Chercher des services specifiques
shodan search "product:nginx country:FR"
shodan search "http.title:dashboard port:8080"

# Info sur une IP
shodan host 1.2.3.4

# Compter les resultats
shodan count "apache country:FR"

# Scan rapide
shodan scan submit 1.2.3.4

# Chercher des webcams
shodan search "has_screenshot:true port:80 http.title:camera"
```

### Recon basique
```bash
# Subdomain enumeration (sans outil externe)
for sub in www mail ftp api dev staging admin panel; do
  dig +short $sub.example.com
done

# Certificate Transparency
curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq '.[].name_value' | sort -u

# WHOIS avance
whois example.com | grep -i "registrar\|name server\|creation\|expiry"

# Reverse IP lookup
curl -s "https://api.hackertarget.com/reverselookup/?q=1.2.3.4"
```

---

## Regex

```
# Basiques
.         N'importe quel caractere
*         0 ou plus
+         1 ou plus
?         0 ou 1
^         Debut de ligne
$         Fin de ligne
[]        Classe de caracteres
|         OU
()        Groupe

# Classes frequentes
\d        Chiffre [0-9]
\D        Pas un chiffre
\w        Alphanumeric [a-zA-Z0-9_]
\W        Pas alphanumeric
\s        Espace blanc
\S        Pas espace blanc

# Quantificateurs
{n}       Exactement n fois
{n,m}     Entre n et m fois
{n,}      Au moins n fois

# Exemples utiles
# Email
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}

# IPv4
\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b

# URL
https?://[^\s]+

# Hash MD5
\b[a-f0-9]{32}\b

# Hash SHA256
\b[a-f0-9]{64}\b

# Date (YYYY-MM-DD)
\d{4}-\d{2}-\d{2}

# SSH log : failed password
Failed password for (invalid user )?(\S+) from (\S+)
```

---

## Python rapide

### HTTP server
```bash
# Serveur HTTP simple (partage de fichiers)
python3 -m http.server 8000

# Avec authentification basique (via script)
python3 -c "
import http.server, base64
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        if self.headers.get('Authorization') == 'Basic ' + base64.b64encode(b'admin:password').decode():
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b'OK')
        else:
            self.send_response(401)
            self.send_header('WWW-Authenticate','Basic realm=\"test\"')
            self.end_headers()
http.server.HTTPServer(('',8000),H).serve_forever()
"
```

### One-liners utiles
```bash
# JSON pretty print
echo '{"key":"value"}' | python3 -m json.tool

# Lire un CSV
python3 -c "import csv; [print(row) for row in csv.reader(open('data.csv'))]"

# Base64 encode/decode
python3 -c "import base64; print(base64.b64encode(b'hello').decode())"
python3 -c "import base64; print(base64.b64decode('aGVsbG8=').decode())"

# Generer un mot de passe aleatoire
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# HTTP request rapide
python3 -c "import urllib.request; print(urllib.request.urlopen('https://api.ipify.org').read().decode())"

# Timestamp -> date lisible
python3 -c "from datetime import datetime; print(datetime.fromtimestamp(1700000000))"
```

---

## Hashcat & John

```bash
# Hashcat : identifier un hash
hashid '$2b$12$hash'

# Hashcat : crack MD5
hashcat -m 0 -a 0 hash.txt wordlist.txt

# Hashcat : crack bcrypt
hashcat -m 3200 -a 0 hash.txt wordlist.txt

# Hashcat : crack NTLM
hashcat -1000 -a 0 hash.txt wordlist.txt

# Hashcat : brute force 8 chars
hashcat -m 0 -a 3 hash.txt ?a?a?a?a?a?a?a?a

# Hashcat : regles
hashcat -m 0 -a 0 hash.txt wordlist.txt -r rules/best64.rule

# Hashcat : combinaison de wordlists
hashcat -m 0 -a 1 hash.txt wordlist1.txt wordlist2.txt

# John the Ripper : crack basique
john --wordlist=wordlist.txt hash.txt

# John : voir les resultats
john --show hash.txt

# John : format specifique
john --format=bcrypt --wordlist=wordlist.txt hash.txt

# Generer un hash
echo -n "password" | md5sum
echo -n "password" | sha256sum
htpasswd -nbBC 10 "" "password" | tr -d ':\n'
```

---

## Kubernetes (kubectl)

```bash
# Infos cluster
kubectl cluster-info
kubectl get nodes

# Pods
kubectl get pods -A           # tous les namespaces
kubectl get pods -n kube-system
kubectl describe pod nom_pod
kubectl logs nom_pod -f
kubectl logs nom_pod --previous  # logs du crash precedent

# Exec dans un pod
kubectl exec -it nom_pod -- /bin/bash

# Deployments
kubectl get deployments
kubectl scale deployment monapp --replicas=3
kubectl rollout status deployment monapp
kubectl rollout undo deployment monapp

# Services
kubectl get svc
kubectl port-forward svc/monapp 8080:80

# Configmaps & secrets
kubectl get configmap
kubectl get secrets
kubectl get secret mon-secret -o jsonpath='{.data.motdepasse}' | base64 -d

# Debug
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes

# Apply / Delete
kubectl apply -f manifest.yml
kubectl delete -f manifest.yml
kubectl delete pod nom_pod --grace-period=0 --force
```

---

## Cron expressions

```
# Format : minute heure jour_mois mois jour_semaine
# * = tout, */N = tous les N, 1-5 = plage, 1,3 = liste

# Toutes les minutes
* * * * *

# Toutes les 5 minutes
*/5 * * * *

# Tous les jours a 3h00
0 3 * * *

# Tous les lundis a 8h00
0 8 * * 1

# Premier jour du mois a minuit
0 0 1 * *

# Toutes les heures en semaine
0 * * * 1-5

# Tous les 15 min entre 9h et 17h
*/15 9-17 * * *

# 2 fois par jour (8h et 20h)
0 8,20 * * *

# Exemples concrets
# Backup quotidien a 3h
0 3 * * * /opt/scripts/backup.sh

# Nettoyage des logs chaque dimanche
0 2 * * 0 find /var/log -name "*.gz" -mtime +30 -delete

# Health check toutes les 5 min
*/5 * * * * curl -sf http://localhost/health || systemctl restart app

# Renouvellement SSL le 1er du mois
0 4 1 * * certbot renew --quiet
```

---

## Valeurs de reference

### Ports courants
```
20/21   FTP
22      SSH
23      Telnet
25      SMTP
53      DNS
80      HTTP
110     POP3
143     IMAP
443     HTTPS
445     SMB
993     IMAPS
995     POP3S
3306    MySQL
3389    RDP
5432    PostgreSQL
6379    Redis
8080    HTTP alternatif
8443    HTTPS alternatif
27017   MongoDB
```

### Codes HTTP
```
200   OK
201   Created
301   Moved Permanently
302   Found (redirect)
304   Not Modified
400   Bad Request
401   Unauthorized
403   Forbidden
404   Not Found
405   Method Not Allowed
429   Too Many Requests
500   Internal Server Error
502   Bad Gateway
503   Service Unavailable
```

### Permissions Linux
```
Octal   Binary   Permissions
0       000      ---
1       001      --x
2       010      -w-
3       011      -wx
4       100      r--
5       101      r-x
6       110      rw-
7       111      rwx

# Exemples courants
644   rw-r--r--   Fichiers normaux
755   rwxr-xr-x   Executables / dossiers
600   rw-------   Fichiers prives (cles SSH)
700   rwx------   Dossiers prives
4755  rwsr-xr-x   SUID
```
