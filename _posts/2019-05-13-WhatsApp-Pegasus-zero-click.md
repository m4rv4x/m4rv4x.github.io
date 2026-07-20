---
title: "2019 – WhatsApp Pegasus, Le zero-click invisible"
date: 2019-05-13 10:00 +0100
categories: ["Histoire"]
tags: ["surveillance", "zero-day", "espionnage", "malware", "mobile"]
author: marvax
---

*Mai 2019. Le téléphone vibre. Aucun appel, aucune notification visible. Juste un signal — un appel WhatsApp manqué, quelques secondes, parfois même pas sonné. À l'autre bout du fil, personne. Ou plutôt : quelque chose. Quelque chose qui ne devrait pas exister, qui ne laisse aucune trace, qui ne demande aucune interaction. Pegasus entre dans votre téléphone comme un fantôme passe à travers un mur. Sans bruit, sans permission, sans même que vous sachiez qu'il est là.*

---

### I. Le zero-click, ou l'art de ne rien demander

Pendant des années, le modèle d'infection mobile reposait sur un principe simple : **tromper l'utilisateur**.

- Un lien cliqué.
- Une pièce jointe ouverte.
- Une autorisation accordée par erreur.

Le phishing, le smishing, le spear-phishing — tout reposait sur une erreur humaine. Une seconde d'inattention, un doigt trop rapide, et le compromis était installé.

**Le zero-click change tout.**

Il supprime l'humain de l'équation. Plus besoin de tromper. Plus besoin de convaincre. Il suffit d'exploiter une faille dans le protocole de communication — ici, le RTC (Real-Time Communication) de WhatsApp — et le payload se dépose silencieusement.

L'appel WhatsApp manqué de mai 2019 ne demandait rien. Il ne contenait aucun message piégé. Il exploitait simplement un **buffer overflow** dans le code de gestion des paquets VoIP d'WhatsApp.

> *"Vous ne devez cliquer sur rien. Vous ne devez rien ouvrir. Vous ne devez même pas répondre. Il suffit que votre téléphone sonne."*

C'est ce qui rendait CVE-2019-3568 si terrifiante. L'attaque ne reposait pas sur la crédulité. Elle reposait sur la **confiance architecturale** — la certitude qu'un appel entrant ne peut pas exécuter du code arbitraire.

---

### II. NSO Group : le marchand d'armes numériques

Derrière Pegasus, il y a un nom : **NSO Group Technologies**, une entreprise israélienne fondée en 2010 à Herzliya, dans la banlieue high-tech de Tel Aviv.

NSO ne se présente jamais comme un vendeur de spyware. La communication officielle est limpide : *"Nous fournissons des outils de renseignement licites aux gouvernements pour lutter contre le terrorisme et la criminalité."*

**La réalité est plus sombre.**

Pegasus a été retrouvé sur les téléphones de :

- **Journalistes** au Mexique, en Azerbaïdjan, au Maroc, en Inde
- **Défenseurs des droits humains** au Bahreïn et aux Émirats
- **Avocats** au Salvador et au Togo
- **Opposants politiques** en Arabie saoudite — notamment dans le contexte de l'affaire **Khashoggi**
- **Personnel de la famille royale du Royaume-Uni**

Le modèle commercial de NSO repose sur la vente de licences gouvernementales. Un client étatique paie entre **500 000 et plusieurs millions de dollars** pour accéder à l'infrastructure Pegasus, capable d'infecter des centaines de terminaux simultanément.

> *"Nous ne choisissons pas les cibles. Nos clients choisissent les cibles."* — Shalev Hulio, cofondateur de NSO Group

Ce déni structurel est le cœur du modèle : NSO vend l'arme, le gouvernement tire, et personne n'est responsable.

---

### III. Anatomie technique d'une infection silencieuse

L'exploit de mai 2019 ciblait la couche VoIP de WhatsApp, implémentée en C/C++ dans la bibliothèque interne de l'application.

**Le vecteur d'attaque fonctionnait ainsi :**

1. L'attaquant initie un **appel VoIP** vers la cible via WhatsApp
2. Le paquet SIP/SDP est **malformé** intentionnellement
3. Le parser RTC ne vérifie pas correctement la taille d'un **buffer**
4. Un **heap overflow** permet d'écrire en mémoire contrôlée
5. Le shellcode s'exécute avec les **privilèges de l'application**
6. Pegasus se déploie et **supprime l'appel des logs**

Une fois installé, Pegasus transformait le téléphone en un **capteur complet** :

- **Microphone** activé à distance
- **Caméra** capturée silencieusement
- **GPS** suivi en temps réel
- **Clavier** keyloggé
- **Messages** extraits de WhatsApp, Signal, Telegram, iMessage
- **Contacts, historiques, photos** exfiltrés
- **Mots de passe** récupérés via l'accès au stockage

Le tout sans aucune icône, aucune notification, aucune consommation anormale visible pour l'utilisateur.

---

### IV. La réponse de WhatsApp et l'industrie

Facebook (propriétaire de WhatsApp) a publié un correctif d'urgence le **13 mai 2019** et a engagé des poursuites judiciaires contre NSO Group.

**La bataille juridique a duré des années :**

- En **octobre 2019**, WhatsApp dépose plainte devant un tribunal fédéral de San Francisco
- NSO invoque l'**immunité souveraine** — ses clients sont des États
- En **décembre 2021**, un juge américain rejette l'argument de NSO
- En **2022**, la Cour d'appel confirme : NSO peut être poursuivi
- En **2023**, le département américain du Commerce ajoute NSO à sa **Entity List**

L'affaire WhatsApp vs NSO a marqué un tournant : pour la première fois, un édiciel de spyware commercial était traîné en justice par une plateforme de communication utilisée par **2 milliards de personnes**.

> *"Cette attaque ciblait au moins 100 défenseurs des droits humains, journalistes et militants dans au moins 20 pays."* — Will Cathcart, directeur de WhatsApp

---

### V. L'écosystème Predator et la prolifération

Pegasus n'est pas seul. L'industrie du spyware commercial est un **marché de plusieurs milliards de dollars** qui prolifère dans l'ombre.

**Concurrents directs de NSO :**

| Entreprise | Produit | Pays d'origine |
|---|---|---|
| **Cytrox / Intellexa** | Predator | Israël / Grèce |
| **Candiru** | DevilsTongue | Israël |
| **Quadream** | Reign | Israël |
| **RCS Lab** | Hermit | Italie |
| **WSR/Wintego** | (classifié) | Allemagne |

En **2021**, Facebook/Meta a révélé que **Predator**, développé par Cytrox (racheté par Intellexa), utilisait un mécanisme d'infection similaire à Pegasus. Predator a été déployé par la **Grèce, le Maroc, l'Égypte et le Soudan**.

**Citizen Lab** de l'Université de Toronto a documenté l'utilisation de Pegasus dans **au moins 45 pays**, dont des membres de l'OTAN et de l'UE.

---

### VI. La leçon de 2019

L'affaire WhatsApp-Pegasus n'est pas un incident isolé. C'est un **révélateur structurel**.

**Ce que 2019 a prouvé :**

- Les **protocoles de communication** sont des surfaces d'attaque — même les apps chiffrées
- Le **chiffrement de bout en bout** ne protège pas contre l'exécution de code sur l'appareil
- L'**industrie du spyware commercial** opère dans un vide juridique international
- Les **gouvernements démocratiques** eux-mêmes achètent et utilisent ces outils
- La **chaîne de confiance** (app store, protocole, OS) est fragile à chaque maillon

**Le zero-click est devenu la norme.** Depuis 2019, des dizaines d'exploits zero-click ont été découverts dans iMessage, FaceTime, Apple Music, Google Chrome. Chaque application qui traite des données entrantes est une porte potentielle.

> *"Le zero-click, c'est l'arme silencieuse par excellence. Pas de phishing, pas de social engineering. Juste la confiance aveugle dans le protocole."* — John Scott-Railton, Citizen Lab

---

### VII. Ce qui a changé depuis

**Apple** a introduit **Lockdown Mode** en 2022 (iOS 16) — un mode de sécurité extrême qui désactive les vecteurs d'infection zero-click connus.

**L'Union Européenne** a lancé une enquête sur l'utilisation de Pegasus par des États membres (Pologne, Hongrie, Espagne, Grèce).

**Le Commerce Department** américain maintient NSO sur sa Entity List, limitant l'accès de l'entreprise à la technologie américaine.

**Mais le marché continue.** Intellexa, Candiru, Quadream opèrent toujours. Les États continuent d'acheter. Les exploits continuent d'être découverts et vendus.

---

## L'Invisible Permanent

**Pegasus n'a pas disparu. Il s'est diversifié.** L'écosystème du spyware commercial est aujourd'hui plus vaste, plus sophistiqué, plus fragmenté qu'en 2019.

Le zero-click n'est plus une exception technique. C'est le **modèle d'attaque dominant** pour les acteurs étatiques. Et chaque mise à jour de votre téléphone est une course entre le patch et l'exploit.

**Votre appareil le plus intime est aussi le plus vulnérable.** Il dort dans votre poche, il connaît vos secrets, il entend vos conversations. Et quelque part, dans un datacenter dont vous ne connaîtrez jamais l'adresse, quelqu'un pourrait déjà être en train d'écouter.

---

*"La surveillance la plus efficace est celle que vous ne remarquez jamais."*

**Keep private, stay vigilant, et souviens-toi : le zero-click, c'est la confiance transformée en arme.**

*— Marvax, Underground Network*
