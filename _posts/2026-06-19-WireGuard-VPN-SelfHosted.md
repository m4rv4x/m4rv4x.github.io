---
title: "WireGuard : VPN self-hosted en 10 minutes"
date: 2026-06-19 10:00 +0200
categories: ["Sysops"]
tags: ["vpn", "wireguard", "linux", "sysops", "self-hosted", "infrastructure", "sécurité", "chiffrement"]
author: marvax
---

> OpenVPN c'est bien. WireGuard c'est mieux : 4000 lignes de code vs 100,000, des performances proches du natif, et une config qui tient dans un fichier. Voici comment déployer ton propre VPN en 10 minutes.

<!--more-->

## Pourquoi WireGuard

| | WireGuard | OpenVPN | IPSec |
|---|-----------|---------|-------|
| Lignes de code | ~4,000 | ~100,000 | ~400,000 |
| Protocole | UDP uniquement | TCP/UDP | UDP |
| Chiffrement | ChaCha20, Curve25519 | Configurable | Configurable |
| Vitesse | Proche du natif | ~30% perte | ~20% perte |
| Config | 1 fichier | Certificats + config | Complexe |
| Handshake | 1-RTT | Multi-RTT | Multi-RTT |
| Kernel module | Oui (depuis 5.6) | Userspace | Kernel |

WireGuard est dans le noyau Linux depuis la version 5.6. Pas de daemon à gérer, pas de certificats à renouveller.

## Installation

```bash
# Debian/Ubuntu
sudo apt install wireguard -y

# Vérifier le module kernel
lsmod | grep wireguard
```

## Architecture

```
┌──────────────┐         ┌──────────────┐
│  Client      │  UDP    │  Serveur     │
│  (laptop)    │◄───────►│  (VPS)       │
│  10.0.0.2    │  51820  │  10.0.0.1    │
└──────────────┘         └──────┬───────┘
                                │
                        ┌───────▼───────┐
                        │  Internet     │
                        │  (via NAT)    │
                        └───────────────┘
```

## Étape 1 — Serveur

```bash
# Générer les clés
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key

# Config serveur
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Client 1
[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
EOF

# Activer le forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Démarrer
sudo systemctl enable --now wg-quick@wg0
```

## Étape 2 — Client

```bash
# Générer les clés
wg genkey | tee client_private.key | wg pubkey > client_public.key

# Config client
cat > wg0.conf << 'EOF'
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

# Connecter
sudo wg-quick up ./wg0

# Vérifier
wg show
curl ifconfig.me  # devrait afficher l'IP du serveur
```

## Split tunneling

Pour ne passer que le trafic du LAN de l'entreprise (pas tout Internet) :

```yaml
# Client config — AllowedIPs restreint
[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_IP>:51820
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24  # uniquement ces réseaux
PersistentKeepalive = 25
```

## Multi-clients

```bash
# Ajouter un client : générer clés + ajouter [Peer] sur le serveur
wg genkey | tee client2_private.key | wg pubkey > client2_public.key

# Ajouter dans /etc/wireguard/wg0.conf
[Peer]
PublicKey = <CLIENT2_PUBLIC_KEY>
AllowedIPs = 10.0.0.3/32

# Recharger sans couper la connexion existante
sudo wg set wg0 peer <CLIENT2_PUBLIC_KEY> allowed-ips 10.0.0.3/32
```

## Sécurité

```bash
# Restreindre les permissions
chmod 600 /etc/wireguard/*.key /etc/wireguard/wg0.conf

# Firewall — n'autoriser que le port WireGuard
sudo ufw allow 51820/udp
sudo ufw allow in on wg0 from 10.0.0.0/24

# Vérifier les connexions actives
wg show

# Renouveler les clés (rotation périodique)
# WireGuard utilise un key rotation automatique (Noise Protocol)
# mais les clés statiques doivent être changées manuellement si compromis
```

## Monitoring

```bash
# Status des peers
wg show

# Transfer de données par peer
wg show wg0 transfer

# Ping interne
ping 10.0.0.1

# Logs
journalctl -u wg-quick@wg0 -f
```

## Cas d'usage terrain

- **Accès SSH distant** : SSH sur le serveur via WireGuard au lieu d'exposer le port 22
- **Accès aux services self-hosted** : Grafana, Gitea, etc. accessibles uniquement via VPN
- **Site-to-site** : connecter deux réseaux LAN distants
- **Mobile** : config Android/iOS avec QR code

---

*Références :*
- [WireGuard official](https://www.wireguard.com/)
- [WireGuard man page](https://manpages.ubuntu.com/manpages/jammy/man8/wg.8.html)

*Ton infrastructure n'a pas de VPN ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour le mettre en place.*
