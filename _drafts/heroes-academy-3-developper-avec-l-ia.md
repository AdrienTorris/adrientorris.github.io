---
title: "Heroes Academy (3/3) : huit mois de développement assisté par IA"
description: "Ce que l'historique Git de Heroes Academy dit de l'IA dans le développement : quand elle est arrivée, ce qu'elle a produit, comment je l'encadre et ce qu'elle ne voit pas."
lang: fr
tags: [ia, claude-code, github-copilot, dotnet, tests, methode, retour-d-experience]
---

# Heroes Academy (3/3) : huit mois de développement assisté par IA

> **Série Heroes Academy**
> 1. [Concevoir le logiciel d'une salle de jeu physique](./heroes-academy-1-la-v1.md)
> 2. [La refonte V2 pour le Canada](./heroes-academy-2-la-refonte-v2.md)
> 3. **Huit mois de développement assisté par IA** (vous êtes ici)

Les deux premiers articles de cette série racontent la conception de Heroes Academy : une V1 qui a appris à la dure ce qu'est un système distribué dans une salle de sport, puis une refonte pensée pour un site à six heures de décalage.

Ce troisième volet traite d'un sujet qui a changé ma façon de travailler pendant cette refonte : le développement assisté par IA. Je ne vais pas faire de prospective. Je préfère m'en tenir à ce que dit l'historique Git du projet : quand l'IA est arrivée, ce qu'elle a produit, comment je l'encadre, et ce qu'elle ne sait pas faire.

---

## Ce que disent les chiffres

J'ai compté comme « assisté par IA » tout commit écrit par un agent, ou portant une mention `Co-Authored-By` d'un assistant. Voici ce que ça donne sur le dépôt principal de la V2, hors commits de merge :

| Période | Ce qui se passait | Commits | Assistés par IA |
|---|---|---:|---:|
| sept. 2025 → fév. 2026 | Architecture de la V2, ouverture du Canada | 643 | 0 |
| mars → mai 2026 | Premiers essais, avec GitHub Copilot | 196 | 27 |
| juin → sept. 2026 | Claude Code, mode Arène, migration de Lille | 160 | 74 |
| oct. → nov. 2026 | Scénarios de jeu, Le Standard | 121 | 118 |

Trois choses ressortent de ce tableau.

**L'IA est arrivée après les fondations.** Les 643 premiers commits de la V2, ceux qui posent toute l'architecture décrite dans la [deuxième partie](./heroes-academy-2-la-refonte-v2.md), ont été écrits sans assistant. Ils couvrent MQTT partout, aucune base côté salle, des jeux en machines à étapes et le matériel décrit en base. Le Canada a ouvert sur ce code-là.

**La bascule a été rapide.** De 2 commits assistés en mars 2026, on passe à 118 sur 121 en octobre-novembre. Au total, 100 des 169 pull requests fusionnées dans le dépôt principal viennent de branches d'agents (`claude/*` ou `copilot/*`). La proportion est du même ordre, voire plus forte, sur l'interface de jeu, l'interface d'administration et la documentation.

**Les tests ont explosé.** C'est pour moi le chiffre le plus important :

| Date | Tests automatisés |
|---|---:|
| janvier 2026 (ouverture du Canada) | ~10 |
| mars 2026 | 17 |
| juin 2026 | 161 |
| septembre 2026 | 217 |
| novembre 2026 | 842 |

La suite de tests a été multipliée par cinquante en huit mois. J'y reviens plus bas.

---

## Les outils, et à quoi ils m'ont servi

### GitHub Copilot : des issues aux pull requests

Les premiers essais, au printemps 2026, se sont faits avec l'agent de Copilot. J'ouvrais une issue, il ouvrait une pull request. C'est un format qui convient bien aux bugs bien délimités :

- le **bug des 20 heures à Montréal** (le tableau de bord filtrait sur le jour UTC) a été corrigé par une fenêtre glissante de 24 h, avec des tests sur les bornes ;
- la **réponse à la question bonus ignorée** pendant la fenêtre de révélation, une course entre deux threads ;
- un **créneau de session** calculé sans tenir compte d'une session de Warm Up en cours ;
- une **faille XSS** dans le tableau des scores de l'administration ;
- et une première vague de **tests unitaires sur les neuf jeux**, qui explique le saut de 17 à 161 tests au printemps.

Quelques correctifs sont aussi venus de Copilot Autofix, appliqués sur des remarques de relecture.

### Claude Code : en session, et dans GitHub

À partir de juin 2026, j'ai surtout travaillé avec Claude Code, de deux façons :

- **en session interactive**, pour les chantiers de fond : le mode Arène, les scénarios de jeu, Le Standard, l'outil de déploiement en une commande, l'UI de test dans le navigateur ;
- **dans GitHub**, via une GitHub Action. Une mention `@claude` dans une issue ou une PR déclenche l'agent, et **chaque pull request reçoit une relecture automatique** avant que je la regarde.

---

## La méthode : écrire avant de coder

Ce qui a le plus changé, ce n'est pas la vitesse d'écriture du code. C'est ce que j'écris *avant* le code.

### Un plan, des variantes, des décisions

Le mode Arène est un bon exemple. Avant la moindre ligne de code, deux documents de conception ont été rédigés avec l'agent. Le premier pilotait le mode depuis la base de données. Le second en faisait une constante de compilation. Les deux portaient la même mention en tête : *« plan de conception, aucune implémentation à ce stade »*.

Le plan de référence contenait un tableau de **décisions figées**, par exemple :

- une Arène après chaque salle de jeu, sauf la dernière ;
- la vidéo de la question reste dans la salle de jeu, et seule la réponse migre dans l'Arène ;
- le résultat de la question est porté par la ligne de l'Arène.

Ces décisions sont des décisions métier, et c'est moi qui les ai prises. J'ai retenu la variante « constante de compilation » parce qu'un site ne change jamais de mode.

L'implémentation a ensuite suivi le plan par phases, chacune dans sa propre PR :

1. création des sessions sans chevauchement ;
2. identité de visite portée de bout en bout ;
3. séquencement par position ;
4. le jeu Arène lui-même ;
5. le reporting.

### Des lots, pas des big bangs

Le Standard, le nouveau gameplay de la salle Communication, a été découpé en sept lots, chacun dans sa PR :

1. socle et choix du gameplay par configuration ;
2. prototype jouable ;
3. appels codés, puis image floue et culture générale ;
4. messages prioritaires et différés ;
5. retours sonores et ambiance ;
6. réglages et profils ;
7. coupure générale et instructions.

Les scénarios de jeu (Halloween, Noël, mode « hardcore ») ont suivi le même découpage. Un lot doit être assez petit pour que je puisse le relire vraiment, et assez complet pour être testé dans la salle.

### Spécifier, puis seulement corriger

L'histoire du `SessionsOrchestrator` « reconnecté mais sourd », racontée dans la deuxième partie, a un épilogue qui résume bien ma façon de travailler.

Un premier correctif avait été produit rapidement par l'agent, fin juillet : resouscrire aux topics après reconnexion. Il a été **écarté**. À la place, j'ai fait écrire une spécification avec une vingtaine de critères d'acceptation. Plusieurs d'entre eux reposaient sur des contraintes que seul un humain connaissait :

- **la base de code** : le livrable devait partir du commit correspondant à la version en production, et non de `main`, pour ne livrer que ce correctif ;
- **le déploiement réel** : le binaire est déployé **sans** son `appsettings.json`, car le fichier du serveur, plus ancien, est conservé. Les nouveaux paramètres devaient donc avoir des valeurs par défaut dans le code, ce qu'un critère vérifie explicitement ;
- **l'exploitation** : un repli sur le port MQTT standard après trois échecs sur le port configuré.

Le code a été réécrit à partir de cette spec, avec 32 tests qui pilotent un client MQTT simulé et une temporisation factice. Aucun test n'attend réellement une seconde.

C'est la leçon que je retiens : **l'agent écrit le code vite, la valeur est dans les critères d'acceptation.**

---

## Les tests sont devenus (presque) gratuits

Sur la V1 comme au début de la V2, les tests étaient rares : quelques-uns sur la planification, une dizaine à l'ouverture du Canada. Un jeu physique se valide d'abord dans la salle, et le temps manquait pour le reste. Avec un agent, écrire un test ne coûte presque plus rien. Ce qui coûte, c'est de le relire.

Deux choix d'architecture ont rendu ça possible :

- **le temps est injecté partout** (`TimeProvider`, un fournisseur de délais abstrait). Un jeu de sept minutes, une boucle de reconnexion avec backoff ou une Arène avec des équipes qui se suivent se testent en quelques millisecondes, de façon déterministe ;
- **le matériel n'est que du MQTT.** Simuler une pression de bouton revient à publier une chaîne.

Résultat : plus de 800 tests, dont des tests de non-régression écrits au moment même où chaque bug est corrigé (croisements dans l'Arène, sessions fantômes, réponses ignorées). Ces tests sont aussi ce qui me permet de laisser un agent modifier le code : ils sont le filet sous le trapèze.

---

## Documenter pour l'humain… et pour l'agent

Un effet que je n'avais pas anticipé : la documentation est redevenue un actif. Un agent lit la même documentation qu'un technicien, et il la suit plus scrupuleusement.

- Le dépôt contient une **compétence de déploiement** (un *skill* Claude Code). Elle décrit comment installer ou mettre à jour un site, quelles questions poser avant d'agir (mode de jeu, adresses, accès), et ce qu'il ne faut jamais faire, comme inventer une clé de salle.
- Le dépôt de documentation a été réorganisé **un dossier par site**, avec l'historique des interventions. Ce que contient chaque dossier sert autant à la personne qui intervient qu'à l'agent qui l'assiste.

L'IA sert d'ailleurs aussi hors du code applicatif. Le watchdog qui règle le processus zombie de la V1, décrit dans la [première partie](./heroes-academy-1-la-v1.md), a été écrit avec elle, tout comme le document qui explique la cause du problème à partir des connexions TCP relevées sur un Pi. La remise en service d'une administration Windows tombée après une coupure réseau aussi.

---

## Ce que l'IA ne voit pas : la salle

L'agent écrit du code pour le système qu'on lui décrit. C'est le terrain qui dit si la description est juste.

Le plan du Standard le dit noir sur blanc : avant d'écrire le moteur, **mesurer le rebond d'un aimant** avec `mosquitto_sub` en retirant puis en remettant un seul aimant sur un panneau. Même chose pour la latence réelle entre le pont électronique, le broker central et l'orchestrateur : il faut la mesurer avant de figer les fenêtres de temps. Aucun modèle ne connaît le rebond des aimants de *notre* salle.

La règle « pas de gyrophare pendant la partie » vient, elle, de l'observation des joueurs, qui sortent de la salle dès qu'un gyrophare s'allume. Le capteur de panier placé en haut à Tarbes, qui faisait marquer deux fois un ballon qui rebondit, aussi. Ces informations-là n'existent dans aucun dépôt tant qu'un humain ne les y a pas écrites.

Et l'agent se trompe, évidemment. Pendant la série de PR du mode Arène, il a introduit une erreur de compilation dans les tests (des arguments optionnels dans des expressions Moq), corrigée dans la foulée. Le mode Arène a aussi vécu sur une branche à part pendant un temps, et chaque évolution devait être reportée sur `main`. L'agent fait bien ce travail mécanique, mais c'est autant de PR à relire.

---

## Ce qui a changé, et ce qui manque encore

En huit mois, une très petite équipe a livré, entre autres :

- le mode Arène et la migration de Lille ;
- les scénarios de jeu ;
- Le Standard ;
- un outil de déploiement en une commande ;
- une UI de test dans le navigateur ;
- une suite de tests multipliée par cinquante.

Sans assistant, ce rythme aurait demandé une équipe nettement plus grande.

Mais le goulot d'étranglement n'a pas disparu, il s'est déplacé. Il n'est plus dans l'écriture du code. Il est dans trois activités qui restent humaines :

1. **spécifier** : décider ce qu'on veut, avec des critères vérifiables ;
2. **relire** : chaque PR est relue automatiquement, mais toutes, sans exception, ont été fusionnées à la main par un humain ;
3. **valider sur site** : un jeu n'est fini que lorsqu'une équipe l'a joué.

Il manque encore une pièce : une CI qui compile et joue ces 800 tests à chaque pull request. Aujourd'hui, la relecture est automatisée, l'exécution des tests ne l'est pas encore.

Ma conclusion est simple. L'IA n'a pas conçu Heroes Academy. Elle est arrivée sur une architecture pensée sans elle, et c'est précisément cette architecture qui l'a rendue efficace : des classes petites, un contrat MQTT documenté, le matériel décrit en données, le temps injecté. Un agent est un excellent développeur pour un système bien découpé. Pour un monolithe où tout dépend de tout, comme l'était la V1, je doute qu'il aurait fait mieux que moi.

---

> **Série Heroes Academy**
> 1. [Concevoir le logiciel d'une salle de jeu physique](./heroes-academy-1-la-v1.md) : la V1, MongoDB comme bus, le processus zombie.
> 2. [La refonte V2 pour le Canada](./heroes-academy-2-la-refonte-v2.md) : l'architecture, les principes et les bugs de la V2.
