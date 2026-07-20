---
title: "2017 – Vault 7 (CIA), L'arsenal numérique exposé"
date: 2017-03-07 10:00 +0100
categories: ["Histoire"]
tags: ["espionnage", "renseignement", "hacktivisme", "malware", "zero-day", "fuite-de-données"]
author: marvax
---

*Mars 2017. WikiLeaks publie un fichier. 8 761 documents. Noms de code, outils, exploits, manuels d'utilisation. L'arsenal cyber de la Central Intelligence Agency — la CIA — se retrouve en libre accès sur le web. Ce n'est pas une fuite ordinaire. Ce n'est pas un lanceur d'alerte qui dénonce un abus. C'est la boîte à outils entière d'une agence de renseignement qui se renverse sur la table. Comme si un armurier secret se réveillait un matin avec la porte ouverte et le stock vide.*

---

### I. Le coffre s'ouvre

Vault 7 est le nom de code donné par WikiLeaks à une série de fuites portant sur les capacités cyber de la CIA, principalement issues du **Center for Cyber Intelligence (CCI)** situé au siège de l'agence à Langley, Virginie.

**La première publication, « Year Zero », contient :**

- Des **exploits zero-day** pour Windows, macOS, Linux, iOS, Android
- Des **malwares** et **frameworks** d'exploitation développés en interne
- Des **techniques de persistence** pour maintenir l'accès à long terme
- Des **outils de contournement** de chiffrement et de messageries sécurisées
- Des **méthodes d'exfiltration** via USB, réseau, et même... les téléviseurs connectés

La source n'est pas un agent double russe ni un hackeur chinois. C'est un **insider** — un développeur ou administrateur travaillant pour la CIA ou un sous-traitant, qui a copié l'intégralité du répertoire interne de la branche cyber.

> *"L'objectif de cette publication est de déclencher un débat public sur la sécurité, la cyber-arme et la prolifération des vulnérabilités."* — WikiLeaks

---

### II. Weeping Angel : quand votre TV vous écoute

Parmi les outils révélés, **Weeping Angel** attire immédiatement l'attention.

Développé conjointement avec le **MI5 britannique**, c'est un implant conçu pour les **téléviseurs Samsung Smart TV** (série F et J). Le fonctionnement est d'une simplicité terrifiante :

1. Le téléviseur entre en mode **« Fake-Off »** — l'écran s'éteint, la LED s'éteint
2. L'utilisateur pense que la télé est éteinte
3. En réalité, le **microphone intégré** reste actif
4. Les conversations sont **enregistrées et exfiltrées** via le réseau WiFi
5. Le tout transmis vers les serveurs de la CIA

Le nom de code « Weeping Angel » référence les créatures de Doctor Who — des statues qui ne bougent que quand on ne les regarde pas.

**Samsung n'a été informée de la vulnérabilité qu'après la publication de WikiLeaks.**

---

### III. Dark Matter et l'infection au firmware

**Dark Matter** est une autre famille d'outils révélée dans Vault 7, ciblant spécifiquement les **produits Apple**.

**Les outils incluaient :**

| Outil | Fonction |
|---|---|
| **Sonic Screwdriver** | Infection via le firmware Thunderbolt au démarrage |
| **DarkSeaSkies** | Implant dans le firmware EFI des MacBook |
| **Triton** | Rootkit macOS en espace noyau |
| **DerStarke** | Persistence au niveau du firmware EFI |
| **NightSkies** | Implant pour iPhone (depuis 2008) |

**Sonic Screwdriver** mérite une attention particulière. Il permettait d'injecter du code malveillant via un **accessoire Thunderbolt** branché sur le MacBook, même si l'utilisateur n'interagissait pas. L'exploit se déclenchait au niveau du **firmware**, avant même que le système d'exploitation ne démarre.

> *"Quand votre sécurité repose sur le firmware, vous faites confiance à une couche que vous ne pouvez ni auditer, ni mettre à jour, ni même voir."*

---

### IV. Les outils de contournement

Vault 7 révèle que la CIA disposait de **capacités de contournement** pour la plupart des applications de chiffrement populaires :

**Approche 1 : Contournement côté endpoint**

Plutôt que de casser le chiffrement (ce qui est mathématiquement difficile), la CIA interceptait les messages **avant le chiffrement ou après le déchiffrement** — directement sur l'appareil.

- **HammerDrill** : collecte de données à partir de clés USB chiffrées
- **HarpyEagle** : extraction de données depuis les appareils iOS
- **BothanSpy** : vol de clés SSH sur Windows
- **Flynight** : exfiltration depuis les systèmes Linux

**Approche 2 : Injection de code dans les applications**

Des implants étaient conçus pour se greffer dans le processus mémoire des applications de messagerie :

- **Clapper** : injection dans Signal
- **SnowyOwl** : injection dans WhatsApp
- **Hawkeye** : capture des messages avant chiffrement

Le chiffrement de bout en bout est solide. Mais si le malware est **sur le même appareil** que l'application chiffrée, il lit les messages **en clair** avant qu'ils ne soient chiffrés.

---

### V. Umbrage : la bibliothèque d'empreintes volées

**Umbrage** est peut-être le composant le plus stratégiquement inquiétant de Vault 7.

C'est une **bibliothèque de composants malveillants** collectés auprès d'autres acteurs — États, groupes APT, hackers indépendants — que la CIA réutilisait dans ses propres opérations.

**Le but : l'attribution brouillée.**

Si la CIA utilise un outil originalement développé par un groupe APT russe ou chinois, les analystes en forensics attribueront l'attaque à la Russie ou à la Chine, pas aux États-Unis.

**Umbrage contenait des composants issus de :**

- **APT28 (Fancy Bear)** — groupe lié au GRU russe
- **APT1 (Comment Crew)** — groupe lié à l'APL chinoise
- Des **malwares publics** (RAT, keyloggers, rootkits)
- Des **techniques de persistence** volées à d'autres groupes

> *"Si tout le monde utilise les mêmes armes, plus personne ne sait qui a tiré."*

C'est le principe du **false flag** appliqué au cyberespace. Et c'est exactement ce qui rend les attributions d'attaques si complexes et si contestées.

---

### VI. Joshua Schulte et l'origine de la fuite

La source de Vault 7 a été identifiée : **Joshua Adam Schulte**, un ancien développeur de la CIA travaillant dans l'unité **EDG (Engineering Development Group)**.

Schulte a été arrêté en **août 2017** et inculpé de **11 chefs d'accusation**, dont :

- Transmission illégale d'informations de défense nationale
- Vol de propriété de la CIA
- Contrefaçon
- Accès non autorisé à un système informatique

Le procès a été chaotique. Un premier jury l'a acquitté sur les charges de fuite en **mars 2020**, mais l'a reconnu coupable de possession de contenus pédopornographiques. Un second procès en **2022** a abouti à sa condamnation pour espionnage.

**Schulte a été condamné à 40 ans de prison en février 2024.**

Le mobile présumé : une **revanche professionnelle** après des conflits internes avec sa hiérarchie à la CIA.

---

### VII. Conséquences géopolitiques

**Les réactions ont été immédiates et mondiales :**

- **Apple** a déclaré que la plupart des vulnérélues avaient déjà été corrigées dans les versions récentes d'iOS et macOS
- **Google** a publié un patch pour Android ciblant les failles mentionnées
- **Microsoft** a corrigé les vulnérabilités Windows exploitées
- **Samsung** a poussé une mise à jour de firmware pour les Smart TV concernées

**Mais le mal était fait.** La publication de Vault 7 a eu des effets durables :

- La **confiance** dans les agences de renseignement américaines a été érodée
- Les **débats sur les backdoors** et les vulnérabilités non-divulguées ont été relancés
- La **prolifération des outils** a été accélérée — des composants de Vault 7 ont été retrouvés dans des attaques ultérieures
- La **doctrine du Vulnerabilities Equities Process (VEP)** — le processus américain de décision sur la divulgation des vulnérabilités — a été remis en question

---

### VIII. La leçon structurelle

Vault 7 n'est pas seulement une fuite. C'est un **miroir tendu à l'architecture même de la cyber-guerre moderne**.

**Ce que cette fuite a révélé de structurel :**

- Les **États accumulent des zero-day** comme des armes, sans les divulguer aux éditeurs
- Le **VEP** (le processus censé arbitrer entre exploitation et correction) est **opaque et politique**
- Les **sous-traitants** (militaires, consultants, développeurs) sont des **points de défaillance critiques**
- La **réutilisation d'outils ennemis** brouille l'attribution et déstabilise la dissuasion
- L'**arsenal cyber** d'une agence est **aussi fragile qu'un entrepôt physique** — il suffit d'un insider

> *"La plus grande menace pour une agence de renseignement, ce n'est pas l'ennemi. C'est l'employé mécontent."*

---

## Le coffre est ouvert

**Vault 7 a changé la donne.** Non pas parce qu'il a révélé que la CIA espionnait — tout le monde le supposait. Mais parce qu'il a montré **comment**.

Les outils publiés ont depuis été repris, modifiés, réutilisés. Certains composants d'Umbrage ont été retrouvés dans des campagnes d'APT d'autres pays. La prolifération a commencé dès le premier jour.

**La sécurité de tous a été diminuée par la fuite des armes de quelques-uns.** C'est le paradoxe central de la cyber-guerre : les mêmes outils qui protègent l'État fragilisent le monde entier quand ils fuient.

Et dans les profondeurs du réseau, quelque part entre Langley et le dark web, l'arsenal continue de se répliquer.

---

*"Quand les armes numériques deviennent publiques, ce n'est plus de la guerre. C'est de l'archéologie."*

**Keep private, stay vigilant, et souviens-toi : chaque backdoor finit par être trouvée — et partagée.**

*— Marvax, Underground Network*
