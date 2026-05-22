---
title: 2024 – Snowflake, La moisson froide des identités dans le cloud sans sommeil
date: 2024-06-10 12:00 +0100
categories: [Histoire]
tags: [Snowflake, cloud, identités, infostealer, Mandiant, cybersécurité]
author: marvax
---

*2024. Les entrepôts de données ne brûlent pas, ils s’ouvrent. Pas avec une explosion, mais avec des identifiants valides. Snowflake devient le décor parfait d’une vérité trop moderne : l’attaquant n’a pas toujours besoin d’une faille dans la plateforme quand l’hygiène des clients lui ouvre déjà la porte. Des identités volées, du MFA absent, des jeux de données massifs, et soudain l’élégance clinique du cloud ressemble à une morgue réfrigérée.*

---

### I. Prologue : le braquage sans effraction

L’affaire Snowflake frappe parce qu’elle casse un vieux réflexe mental.  
Beaucoup de gens veulent encore croire qu’une grande compromission cloud suppose une grande faille produit.

Ici, la scène est plus humiliante :

- credentials volés en amont ;
- comptes clients exposés ;
- MFA absent ou insuffisant ;
- accès parfaitement légitimes en apparence ;
- extraction de données à grande échelle.

Le cloud moderne découvre une vérité très sobre :  
la sécurité d’une plateforme n’annule jamais la misère des pratiques d’accès.

---

### II. Entrepôts de données, jackpot logistique

Snowflake concentre ce que l’économie adore centraliser :

- analytics ;
- données clients ;
- historiques transactionnels ;
- exports consolidés ;
- secrets de croissance et d’exploitation.

Pour un attaquant, c’est un rêve logistique.  
Un seul point d’accès bien choisi peut donner sur des volumes qu’il aurait fallu autrefois voler en plusieurs campagnes distinctes.

La modernité adore les data lakes.  
Les prédateurs aussi.

---

### III. Dossier technique : l’identité comme surface d’attaque principale

Le cœur de l’affaire, ce n’est pas un exploit exotique.  
C’est la chaîne d’accès :

- postes compromis par infostealers ;
- secrets stockés ou réutilisés ;
- absence de MFA sur certains comptes exposés ;
- visibilité tardive sur les connexions et l’exfiltration.

Le message technique est brutal :  
dans les architectures cloud, la frontière la plus rentable reste souvent l’identité, pas l’hyperviseur ni le service managé lui-même.

---

### IV. Bureau Delta : note de cellule sur les accès propres

> *"03:11. Aucun exploit connu.*  
> *03:18. Aucun binaire déployé.*  
> *03:24. Aucune alerte romantique pour flatter les dashboards.*  
> *03:41. Juste des logins, des exports et le bruit très propre d’un entrepôt qu’on vide avec les bons badges."*

La scène résume le problème Snowflake :  
la compromission cloud la plus dangereuse est souvent celle qui ressemble à de l’usage.

---

### V. Résonance actuelle : le MFA comme ligne de partage réelle

Snowflake a laissé une leçon triviale et pourtant toujours mal appliquée :  
tant que l’identité client reste inégalement durcie, le cloud continuera de transformer la négligence locale en incident global.

Aujourd’hui encore, la différence entre incident contenu et saignée massive tient souvent à peu de choses :

- MFA imposé ;
- segmentation des rôles ;
- rotation des secrets ;
- surveillance d’extraction anormale ;
- durcissement des endpoints où vivent les credentials.

---

### VI. Héritage : le cloud n’oublie pas qui ouvre la porte

Snowflake ne raconte pas seulement une fuite.  
Il raconte l’âge où l’identité est devenue la vraie serrure du cloud, et où cette serrure est encore trop souvent laissée dans la poche d’un endpoint sale.

> *"Le futur des grandes fuites n’est pas toujours dans le zero-day. Il est souvent dans le mot de passe qui a survécu trop longtemps."*
