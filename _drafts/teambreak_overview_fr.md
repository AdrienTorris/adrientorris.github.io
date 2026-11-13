# Cinq ans chez Team Break : de la Donut Quiz à Maestro

*Retour sur les chantiers que j'ai menés entre 2021 et 2026.*

Mon aventure chez Team Break se termine. Avant de passer à la suite, j'ai voulu faire le point sur ce que nous avons construit : les produits, les plateformes, les choix techniques, et les erreurs qui nous ont appris quelque chose.

## En bref

- **Parcours :** prestataire au forfait (2021 – 2023), puis salarié (mars 2023 – novembre 2026).
- **Rôle :** architecte et développeur principal des logiciels qui font tourner nos salles. J'ai aussi rédigé les spécifications, déployé sur site, documenté et mené le recrutement technique.
- **Périmètre :**
  - une douzaine d'escape games pilotés par Maestro, sur neuf sites ;
  - la Donut Quiz, sur neuf sites ;
  - la Heroes Academy, sur trois sites dont un au Québec ;
  - Archipel, le système d'accueil connecté d'Aéroville.
- **Stack principale :** .NET (de la 5 à la 10), ASP.NET Core, EF Core, PostgreSQL, MQTT, Docker, Raspberry Pi.

## Le contexte

Chez Team Break, nous exploitons des escape games et des activités immersives : quatre sites en propre (Lille, Paris 9, Aéroville, La Défense) et un réseau de franchisés, de Rouen à Grenoble et jusqu'à Brossard, au Québec.

Derrière chaque salle, il y a du logiciel :
- des scénarios à orchestrer ;
- des dizaines d'énigmes électroniques à piloter ;
- des écrans, du son et de la lumière ;
- des maîtres du jeu à outiller ;
- des réservations à faire coïncider avec tout ça.

C'est ce logiciel que j'ai conçu, construit et fait tourner pendant ces années.

| Période | Statut | Chantiers |
|---|---|---|
| 2021 – 2022 | Prestataire externe (Pepit), au forfait | Donut Quiz v1, Heroes Academy v1 |
| 2023 | Salarié depuis le 1er mars | Kiosque central Heroes Academy, Headquarters, GameEngine |
| 2024 | | Enquête de satisfaction, naissance de Maestro, la Chocolaterie et Walking Dead sur Maestro |
| 2025 | | Maestro à Aéroville et chez les franchisés, Heroes Academy v2, premier prototype d'Archipel |
| 2026 | | Donut Quiz v2, Archipel en production, industrialisation de Maestro, migration de Lille et de La Défense |

---

## 1. Donut Quiz : là où tout a commencé (2021 → 2026)

La Donut Quiz est un quiz show joué en salle. Six équipes s'affrontent depuis des pupitres tactiles, chacun baptisé d'un parfum de donut et équipé d'un vrai buzzer et d'une barre de LED. Un grand écran commun, le « Saloon », fait vivre la partie. Projecteurs DMX, lyre, machine à fumée et bandes-son installent l'ambiance. Mini-jeux, coups spéciaux et événements surprises rythment la partie, jusqu'à la remise du Donut d'Or.

**Comme prestataire (2021 – 2022)**, j'ai développé la v1 au forfait, de bout en bout :
- le serveur de jeu et la « Game Factory », qui compose le jeu de questions de chaque partie (entièrement réécrite en 2.1) ;
- les applications des pupitres et du Saloon, en .NET 5 / Avalonia sur Raspberry Pi, avec LibVLCSharp pour la vidéo ;
- l'administration et le check-in de chaque site (ASP.NET Core, MongoDB) ;
- le back-office central des questions et sa synchronisation nocturne avec les sites ;
- l'intégration matérielle : DMX via FTDI, buzzers sur GPIO, cartes son HifiBerry.

Une salle compte huit Raspberry Pi 4 (six pupitres, le Saloon, le serveur de jeu), plus un serveur par site. Pour déployer tout cela à l'échelle, j'ai mis en place des pipelines Azure DevOps avec des agents auto-hébergés sur chaque site. J'ai aussi rédigé un manuel d'exploitation d'environ 70 pages. Plus d'une fois, je me suis retrouvé sur place à déboguer un buzzer muet ou une carte son récalcitrante.

**Comme salarié (2023 – 2026)**, j'ai continué à faire évoluer le produit :
- la version 2.2 : pause, mode court, « réponse blague »… ;
- un composant « Central Atmosphere » qui pilote la lyre et la fumée ;
- la séparation du Game Master en deux composants, le Sound Master et le Spotlights Master ;
- de nouvelles installations : Paris 9, Lille, Grenoble.

En 2026, j'ai mené la **v2**, une architecture MQTT qui s'appuie sur nos agents audio et DMX partagés avec Maestro. Je l'ai déployée moi-même à Lille, Aéroville, La Défense, Caen, Paris 9 et Rouen entre février et juin. Un sprint de fiabilisation a suivi :
- une vingtaine de bugs audio corrigés ;
- les LED des pupitres passées sur MQTT ;
- le filtrage des contenus.

Nous avons ensuite engagé la montée en .NET 10 et la réécriture des écrans en web. La Donut Quiz compte aujourd'hui plus de 3 300 fichiers son par salle et tourne sur neuf sites.

## 2. Heroes Academy : un parcours physique piloté par le logiciel (2022 → 2026)

La Heroes Academy est un parcours d'une heure, en équipe, à travers une succession de salles d'épreuves physiques : cordes, « Snake » (un sol de dalles lumineuses), « Don't Touch The Floor », paniers de basket, panneaux de communication, et une Arène finale en forme de quiz. Les écrans affichent consignes, chrono et score. Les résultats alimentent un classement par site et un classement public.

**v1 (prestataire, 2022).** Je l'ai développée au forfait, et elle est en production à Lille depuis juillet 2022. Stack : .NET 5, administration en Razor Pages, écrans Avalonia sur Raspberry Pi, MongoDB, électronique pilotée via MQTT et des extenseurs GPIO. En dix mois, plus de 2 200 équipes et près de 8 000 joueurs y étaient passés.

**Salarié (2023 – 2025).** En mars 2023, j'ai construit en quatre jours le **kiosque central**. Il agrège les scores de tous les sites et alimente le classement public du site web. J'ai ensuite fait évoluer seul la v1 :
- l'installation de Tarbes ;
- une refonte du gameplay début 2024 (algorithme du Snake, score pondéré selon la taille de l'équipe) ;
- la reconstruction d'un serveur après une panne matérielle ;
- un vrai versionnement en 2025.

**v2 (sept. 2025 → 2026).** C'est une réécriture complète, dont j'ai écrit l'essentiel (plus de 80 % des commits) :
- .NET 8, PostgreSQL, MQTTnet 5 ;
- un serveur central sous Ubuntu, et un Raspberry Pi par salle qui porte l'orchestrateur de salle, le pont électronique, les écrans et le lecteur audio ;
- la configuration servie par l'API et mise en cache sur chaque machine ;
- l'installation d'un site entier en une seule commande ;
- plus de 820 tests unitaires.

La v2 nous a permis d'ouvrir de nouveaux sites, à commencer par **Brossard, au Québec**, en janvier 2026. Lille a basculé en v2 à l'automne 2026, avec un nouveau **mode Arène** choisi à la compilation. D'autres ajouts ont suivi :
- les réglages de gameplay stockés en base ;
- les caméras et le push-to-talk pour les maîtres du jeu ;
- les **scénarios** (Halloween, Noël, hardcore…).

## 3. GameEngine : le prototype qui a ouvert la voie (2023 → 2024)

Mon premier grand chantier de salarié a été de doter Team Break de son propre moteur pour piloter ses salles. Ce fut **GameEngine**, nom de code « CHARLIE ». Chaque salle embarquait toute la pile sur un Raspberry Pi :
- un moteur qui exécute un *gameflow* JSON ;
- un dispatcher de sessions ;
- l'API et l'interface du maître du jeu ;
- un lecteur audio multipiste ;
- la synchronisation avec un back-office central.

GameEngine a fait tourner de vraies salles :
- **Le Trésor des Pirates** à La Défense, en octobre 2023 ;
- **Los Bandidos** à Terville, puis à La Défense et à Aéroville ;
- **l'Égypte** à La Défense.

Il m'a surtout appris ce qu'il ne fallait pas faire :
- les gameflows reposaient sur des identifiants numériques, illisibles et fragiles ;
- chaque évolution du format touchait à la fois le JSON et le schéma de base, alors que des salles étaient en production ;
- les énigmes complexes finissaient en code spécifique difficile à maintenir.

Au printemps 2024, plutôt que de prolonger l'effort sur une v2, j'ai choisi de repartir d'une page blanche.

## 4. Maestro : la plateforme (mai 2024 → aujourd'hui)

Maestro est né en mai 2024, avec une ambition simple : faire tourner n'importe quelle salle de façon fiable et uniforme, et donner aux maîtres du jeu des outils soignés. Il fallait une plateforme plus maintenable et plus stable, avec des gameflows enfin lisibles par un humain. C'est le chantier dont je suis le plus fier.

### L'architecture

Maestro est un ensemble d'agents .NET 8 conçus en DDD, qui communiquent via MQTT (Mosquitto) :
- l'**orchestrateur de salle** exécute le gameflow comme une machine à états (préparation, briefing, en jeu, fin, repos…) ;
- l'**orchestrateur de jeu** enchaîne les sessions entre plusieurs salles ;
- l'**API et l'interface du maître du jeu** sont écrites en ASP.NET Core minimal APIs et en Web Components, sans framework ;
- l'**agent électronique** est un proxy configurable qui traduit les formats MQTT des différents fabricants d'énigmes (A+ Props, Bela/Node-RED, Kincony, nos propres cartes ESP32…) ;
- des agents de **lecture audio**, **vidéo** et **DMX** sont réutilisés d'un jeu à l'autre ;
- des agents d'**actions planifiées**, d'**urgence** (relais de la sécurité incendie), de **healthcheck** et de **reporting** alimentent notre serveur central et notre outil de réservation.

Le cœur ne contient que du code générique. Chaque jeu a son propre dépôt d'agents spécifiques, et le code partagé est publié en paquets NuGet privés.

### Le gameflow, pièce maîtresse

Chaque jeu, et chaque site, a son gameflow JSON versionné dans Git, organisé par énigme plutôt que par type technique.
- Les agents téléchargent la dernière version au démarrage et en gardent une copie locale pour fonctionner hors ligne.
- Un fichier de versions décide de ce qui part en production.
- Un système de patchs évite toute modification manuelle de la base.

C'est cette lisibilité qui nous a permis d'écrire et de livrer des salles à plusieurs.

### L'industrialisation

Nous sommes partis d'installations manuelles : services systemd, archives copiées en scp. En 2026, nous disposions de :
- un paquet Debian `maestro-room` qui embarque les images Docker, construit en CI pour amd64 et arm64 ;
- un outil de migration des installations historiques ;
- des releases versionnées, de la v1.1 en juillet à la v1.18 en novembre 2026 ;
- un watchdog et une reprise automatique, ajoutés après qu'une coupure de courant a immobilisé une salle pendant trois heures.

### L'adoption

Aujourd'hui, Maestro fait tourner **une douzaine de jeux sur neuf sites**, soit près de quarante couples jeu × site : Walking Dead, la Chocolaterie, le Tombeau du Pharaon, le Trésor des Pirates, Los Bandidos, Haunted Room, Magic School, Jurassik, Koh-Lanta, Prison Break, Fort Boyard et Lost.

S'y ajoute le **Lobby**, l'accueil d'un site piloté comme une salle à part entière : ambiance sonore en boucle, briefs de mission, scénarios lumineux sur les façades.

Le déploiement s'est fait par vagues :
- **été 2024 :** la Chocolaterie et Walking Dead à Lille ;
- **fin 2024 :** Grenoble et La Défense ;
- **2025 :** Aéroville ;
- **2025 – 2026 :** les franchisés (Bourges, Terville, Metz, Rouen) ;
- **automne 2026 :** les salles de Lille, Paris 9 et La Défense qui tournaient encore sur le contrôleur du fournisseur. J'ai piloté la migration de Lille.

Les dernières salles GameEngine, à La Défense, ont été remplacées en octobre 2026 : la boucle est bouclée.

## 5. La Chocolaterie : notre jeu le plus ambitieux (2023 → 2026)

La Chocolaterie, c'est cinq salles enchaînées, seize Raspberry Pi et une profusion d'énigmes :
- un magasin de bonbons, avec caisse enregistreuse et bocaux ;
- un jardin de chocolat, avec une machine à pompes, un Twister, un tape-taupe, et même un « mode demande en mariage » ;
- un ascenseur, avec Guitar Hero, vibreurs et trappes ;
- une salle de glace, où l'on joue à un jeu de plateforme sur deux niveaux, commandé par de vrais boutons, des ventilateurs et des tuyaux ;
- une salle des inventions, avec distributeur de bonbons, livre de recettes, presse-citron et sol LED.

Je l'ai imaginée dès 2023 sous GameEngine. En mai 2024, elle est devenue le premier jeu reconstruit sur Maestro. Elle a servi de banc d'essai à la plateforme et l'a poussée dans ses retranchements :
- plus de 600 commandes, près de 300 gestionnaires de notifications et plus de 300 proxys électroniques dans le gameflow ;
- 341 révisions entre la v0.1.1 de juillet 2024 et novembre 2026 ;
- un jeu de plateforme que j'ai réécrit en Godot / C# en 2025.

J'en ai été l'architecte et le principal développeur.

## 6. Archipel : l'agence connectée (2025 → 2026)

Maestro pilote une salle. **Archipel** pilote le site tout entier : c'est la couche « agence » au-dessus des salles, en production à Aéroville depuis le printemps 2026.

Le parcours du joueur :
1. La réservation arrive de 4Escape.
2. À l'accueil, une borne de check-in retrouve la réservation par nom, e-mail, téléphone ou numéro de commande. Elle confirme la langue et le nombre de joueurs, puis remet au groupe son monde, ses consignes de casier et un **mot de lancement secret**.
3. Des écrans annoncent les prochaines missions et les disponibilités, avec des promotions de dernière minute.
4. Devant la porte, une tablette s'éveille et l'équipe y choisit son mot.
5. Archipel envoie à Maestro l'ordre de lancement (équipe, langue, nombre de joueurs, scénario), puis relaie en retour la progression et le compte à rebours.

**Les choix techniques :**
- .NET 10, ASP.NET Core minimal APIs, EF Core, PostgreSQL, MQTTnet 5 ;
- un front en modules ES natifs, sans framework, et des écrans temps réel en SSE ;
- un déploiement « pull-only » d'images Docker immuables, sur un mini-PC par agence ;
- un contrat MQTT avec Maestro versionné, publié en NuGet et figé par des tests de format ;
- les données de l'agence (mondes, missions, salles, mots) versionnées dans Git sous forme de patchs validés en CI.

J'ai écrit le cœur du backend et la quasi-totalité des spécifications. Nous y avons intégré la Donut Quiz comme second système de jeu.

Une campagne de fiabilisation a clos le chantier : un banc de test, 61 scénarios de recette et de la CI sur tous les dépôts. Elle s'est achevée par la mise à jour d'Aéroville le 4 novembre 2026.

## 7. Headquarters : le serveur central

Headquarters, notre « QG », est le serveur central de Team Break. Je l'ai remis trois fois sur le métier :
- **2023 :** un back-office en .NET 7. Il rapatrie les données des sites Donut Quiz et Heroes Academy et produit des rapports par salle et par agence. En parallèle, GameEngine avait son propre Headquarters, qui poussait les gameflows vers les salles.
- **2024 :** un produit plus ciblé, mis en production : l'**enquête de satisfaction**. Après sa partie, le client note son expérience. Un bon de réduction est généré dans 4Escape et envoyé par e-mail via Mailjet, et les clients les plus enthousiastes sont invités à laisser un avis Google.
- **2026 :** un nouveau départ en .NET 10, qui collecte les sessions remontées par chaque instance Maestro, avec une clé d'API par agence.

## 8. Au-delà du code

**Documenter pour transmettre.** J'ai écrit :
- les manuels d'exploitation et les fiches par salle ;
- les runbooks du serveur central ;
- les documents de cadrage avec des décisions numérotées ;
- un radar technologique et un guide de branches Git, pour nous donner un cadre commun.

**Recruter.** En 2024, j'ai conçu notre entretien technique : une revue de code sur une implémentation du jeu CodeNames, un exercice de modélisation de base de données et des questions sur LINQ et l'asynchrone.

**Rendre l'équipe autonome.** Pendant longtemps, j'ai été le seul développeur. Mon objectif a toujours été que nos outils permettent d'écrire et de livrer des salles sans dépendre de moi : des gameflows lisibles, des procédures écrites, des déploiements reproductibles. En 2026, je n'étais plus le premier contributeur de Maestro, et c'est sans doute ma plus grande fierté.

**Travailler avec l'IA.** En 2026, nous avons fait entrer l'IA dans notre quotidien de développeurs.
- J'ai adopté un mode de travail où j'écris l'issue, qui sert de spécification. Un agent (GitHub Copilot, puis Claude Code) produit la pull request, que je relis, corrige et fusionne.
- À l'automne 2026, cela a représenté plus de 200 pull requests sur les seuls dépôts de jeux.
- Nous avons ensuite structuré cette pratique en chaîne : planification, validation humaine de la spécification, implémentation, puis vérification par un second agent.

Bien cadrés, ces outils ne remplacent pas l'ingénieur. Ils lui rendent du temps pour l'architecture, les spécifications, les tests et la documentation.

## Ce que je retiens

- **La lisibilité est une fonctionnalité.** Le passage de GameEngine à Maestro tient d'abord à des gameflows qu'un humain peut lire, relire et confier à quelqu'un d'autre.
- **Le terrain a toujours raison.** Une coupure de courant, une carte SD fatiguée, un réseau d'agence capricieux : la résilience (copie locale, reconnexion infinie, watchdog) n'est pas un luxe.
- **Industrialiser tôt.** Des paquets, des images, des releases versionnées et des procédures écrites : c'est ce qui permet de passer d'une salle à quarante.
- **Les données dans Git.** Gameflows, configurations d'agence et données de référence sont versionnés, relus et validés en CI.
- **Savoir jeter.** Abandonner GameEngine au profit de Maestro a été la meilleure décision technique de ces années.

## Stack et compétences

- **Architecture :** systèmes distribués pilotés par événements (MQTT), DDD, CQRS / mediator, machines à états, contrats de messages versionnés.
- **.NET (de la 5 à la 10) :** ASP.NET Core (minimal APIs, Razor Pages), EF Core, MediatR, MQTTnet, Avalonia, xUnit / MSTest, paquets NuGet privés.
- **Données :** PostgreSQL, MongoDB.
- **Front :** Web Components, modules ES natifs, Server-Sent Events.
- **Matériel et embarqué :** Raspberry Pi, GPIO, DMX, intégration d'ESP32 et de cartes de fabricants d'énigmes, Godot (C#).
- **DevOps :** Docker, GitHub Actions, GHCR, paquets Debian, systemd, Azure DevOps.
- **Pratiques :** spécifications et documents de cadrage, tests d'acceptation, documentation d'exploitation, recrutement technique, développement assisté par IA.

## Merci

Merci à Team Break pour sa confiance, de la première mission au forfait jusqu'à aujourd'hui. Merci aux maîtres du jeu et aux équipes des sites, qui ont essuyé les plâtres et fait remonter chaque bug avec patience. Et merci à mes collègues de l'équipe technique pour tout ce que nous avons construit ensemble.
