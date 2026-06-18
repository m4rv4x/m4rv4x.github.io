---
title: "Passkeys & FIDO2 : la mort du mot de passe"
date: 2026-02-07 10:00 +0200
categories: ["Sécurité"]
tags: ["passkeys", "fido2", "webauthn", "sécurité", "authentication", "opsec"]
author: marvax
---

> Le mot de passe est mort. Pas « bientôt mort », pas « potentiellement mort » — mort. Les passkeys et FIDO2 offrent une authentification sans mot de passe, résistante au phishing par conception. Mais l'adoption en 2026 est encore inégale, et le vendor lock-in est un vrai problème. Voici où on en est vraiment.

<!--more-->

## Le problème avec les mots de passe

On le sait tous, mais on le dit quand même :

- **Réutilisation** : 65% des gens réutilisent le même mot de passe sur plusieurs sites
- **Phishing** : un faux site copie-colle parfait et tu donnes ton mot de passe
- **Brute force** : les mots de passe faibles tombent en secondes
- **Fuites** : des milliards de credentials sont en circulation sur le dark web (voir [guide d'hygiène post-fuite](/posts/2026-06-17-Guide-hygiene-post-fuite/))
- **Le [coffre-fort LastPass](/posts/2022-12-22-LastPass-coffre-fort-trahi/) trahi** : même un gestionnaire de mots de passe peut être compromis

Le 2FA (TOTP) aide, mais c'est un pansement. Les codes TOTP peuvent être phishés en temps réel (real-time phishing proxy).

## FIDO2 / WebAuthn : comment ça marche

FIDO2 est le standard. WebAuthn est l'API web. Passkeys est le terme marketing d'Apple/Google/Microsoft pour rendre ça consommable.

### Le principe

```
Authentification classique :
  Client → envoie mot de passe → Serveur vérifie
  (le secret transite sur le réseau)

Authentification FIDO2 :
  Client → prouve qu'il possède la clé privée → Serveur vérifie la preuve
  (le secret ne quitte jamais l'appareil)
```

### Cryptographie

1. **Enregistrement** : le serveur génère un challenge aléatoire. L'appareil local signe le challenge avec sa clé privée (ECDSA P-256). La clé publique est enregistrée côté serveur.

2. **Authentification** : le serveur envoie un challenge. L'appareil signe avec la clé privée. Le serveur vérifie avec la clé publique.

3. **Anti-phishing** : le challenge inclut l'origine (domaine). Un site phishing sur `gooogle.com` ne peut pas générer un challenge valide pour `google.com`.

```javascript
// Côté serveur — création d'une passkey
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: new Uint8Array(32),  // Aléatoire, généré côté serveur
    rp: { name: "Mon Site", id: "monsite.com" },  // Anti-phishing : domaine
    user: {
      id: new Uint8Array(16),
      name: "user@example.com",
      displayName: "Utilisateur"
    },
    pubKeyCredParams: [{ type: "public-key", alg: -7 }]  // ES256
  }
});
```

## État de l'adoption en 2026

### Ce qui supporte les passkeys

| Plateforme | Support passkeys | Sync cross-device |
|---|---|---|
| Apple (iOS/macOS) | Natif via iCloud Keychain | Oui |
| Google (Android/Chrome) | Natif via Google Password Manager | Oui |
| Microsoft (Windows) | Natif via Windows Hello | Partiel |
| Bitwarden / Vaultwarden | Oui | Oui |
| 1Password | Oui | Oui |
| GitHub | Oui | — |
| Google | Oui | — |
| Amazon | Oui | — |
| Banques (FR) | En progression | — |

### Ce qui manque encore

- **Les PME** : la plupart des sites web n'implémentent pas WebAuthn
- **Linux** : support navigateur correct, mais l'expérience est inégale (pas de keystore natif comme Keychain/Hello)
- **Les banques** : adoption lente malgré les recommandations ANSSI
- **Les administrations** : encore largement sur mots de passe + SMS

## Le problème du vendor lock-in

C'est LE sujet qui fâche. Les passkeys sont synchronisées via :

- **Apple** : iCloud Keychain
- **Google** : Google Password Manager
- **Microsoft** : Windows Hello / Entra ID

Si tu crées une passkey sur iPhone, elle est dans iCloud. Elle ne passe pas automatiquement dans Google Password Manager.

**Les solutions :**

1. **Gestionnaire de mots de passe tiers** : [Vaultwarden](/posts/2026-02-01-Vaultwarden-self-hosted-mdp/) et Bitwarden supportent le stockage de passkeys cross-platform
2. **Clés physiques** : YubiKey 5 — ta passkey est sur le hardware, pas dans un cloud
3. **Export/transfert** : le FIDO Alliance travaille sur un format d'export standard (Credential Exchange Protocol)

```bash
# Vaultwarden — activer le support passkeys
# Dans l'extension navigateur Bitwarden :
# Settings → Security → Allow passkey login
# Les passkeys sont stockées dans le vault, synchronisées via le serveur
```

## Passkeys vs alternatives

| Méthode | Anti-phishing | Résistance brute force | UX | Dépendance |
|---|---|---|---|---|
| Mot de passe seul | ❌ | ❌ | Moyenne | Aucune |
| MDP + TOTP | ⚠️ (real-time phishing) | ✅ | Correcte | App TOTP |
| MDP + SMS | ❌ (SIM swap) | ✅ | Facile | Opérateur |
| Passkey (device-bound) | ✅ | ✅ | Excellente | Appareil |
| Passkey (synced) | ✅ | ✅ | Excellente | Cloud vendor |
| Clé FIDO2 physique | ✅ | ✅ | Bonne | Hardware ($$) |

## Guide d'adoption pratique

### Étape 1 : Choisir un gestionnaire compatible

Utilise un gestionnaire de mots de passe qui supporte les passkeys :
- **Vaultwarden** (self-hosted, gratuit) — voir [le guide de déploiement](/posts/2026-02-01-Vaultwarden-self-hosted-mdp/)
- **Bitwarden** (cloud, freemium)
- **1Password** (payant)

### Étape 2 : Activer les passkeys sur les services qui les supportent

```bash
# Services prioritaires (fort impact sécurité) :
# 1. Google / Apple ID / Microsoft
# 2. GitHub / GitLab
# 3. Cloudflare / AWS / GCP
# 4. Banque en ligne (si supporté)
# 5. Email principal
```

### Étape 3 : Gérer la récupération

Les passkeys ne sont pas un « set and forget » :

- **Clé de récupération** : génère et stocke hors ligne (papier dans un coffre)
- **Passkey secondaire** : enregistre sur un second appareil ou YubiKey
- **Codes de secours** : toujours les garder

### Étape 4 : Ne pas supprimer les mots de passe tout de suite

Garde le mot de passe comme méthode de secours tant que tu n'as pas :
- Au moins 2 passkeys sur des appareils différents
- Les codes de récupération sauvegardés
- Testé la récupération sur un appareil propre

## L'OSINT et les passkeys

L'[OSINT](/posts/2026-06-10-OSINT-reconnaissance-ouverte/) sert à identifier les cibles via les fuites de mots de passe. Avec les passkeys, ce vecteur disparaît : pas de mot de passe à fuiter, pas de hash à cracker, pas de credential stuffing possible.

C'est un gain défensif majeur.

## Mon avis honnête

Les passkeys sont **la meilleure chose qui soit arrivée à l'authentification** depuis le 2FA. Mais :

- L'écosystème est immature (2026, pas 2030)
- Le vendor lock-in est un vrai problème, pas un détail
- Tous les services ne les supportent pas
- Il faut un plan de récupération robuste

**La bonne stratégie en 2026 :**
1. Gestionnaire de mots de passe solide (Vaultwarden, Bitwarden)
2. Passkeys là où c'est supporté
3. Mots de passe forts + TOTP pour le reste
4. YubiKey pour les comptes critiques
5. Zéro SMS en 2FA (jamais, c'est du poison)

## Checklist

- [ ] Gestionnaire de mots de passe avec support passkeys installé
- [ ] Passkeys activées sur les comptes prioritaires (Google, Apple, GitHub)
- [ ] Au moins 2 passkeys par compte critique (appareil + backup)
- [ ] Codes de récupération générés et stockés hors ligne
- [ ] Mots de passe de secours conservés (pas encore supprimés)
- [ ] SMS désactivé comme 2FA partout (remplacé par TOTP ou passkey)
- [ ] [Hygiène post-fuite](/posts/2026-06-17-Guide-hygiene-post-fuite/) effectuée sur les anciens comptes
- [ ] Stratégie de récupération documentée et testée

---

*Références :*
- [FIDO Alliance](https://fidoalliance.org/)
- [WebAuthn specification (W3C)](https://www.w3.org/TR/webauthn/)
- [passkeys.dev](https://passkeys.dev/)
- [ANSSI — recommandations authentification](https://www.ssi.gouv.fr/)

*Tu utilises encore « Azerty123! » comme mot de passe ? [Contacte-moi](mailto:m4rv4x@protonmail.com) pour migrer vers les passkeys.*
