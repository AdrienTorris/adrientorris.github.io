> **Titres alternatifs (à choisir, puis supprimer ce bloc avant publication)**
>
> 1. Cinq salles, trois équipes, un Pinch : les coulisses techniques de La Chocolaterie
> 2. Des murs invisibles et des octets nuls : ce que La Chocolaterie m'a appris
> 3. Godot, MQTT et huit boutons dans un ascenseur : retour d'expérience sur La Chocolaterie

# La Chocolaterie : dans les coulisses techniques d'un escape game en cinq actes

*Un plateformer qui se joue sur un vrai décor, un Guitar Hero à huit joueurs dans un ascenseur, et des équipes qui doivent traverser cinq salles sans jamais rester coincées. Retour d'expérience.*

---

De tous les jeux sur lesquels j'ai travaillé, **La Chocolaterie** est celui qui m'a donné le plus de fil à retordre. Ce n'est pas une salle, mais **cinq salles enchaînées** : la confiserie (*Candy Store*), le jardin chocolaté (*Chocolate Garden*), l'ascenseur (*Elevator*), la chambre froide (*Ice Room*) et la salle des inventions (*Inventing Room*). Les joueurs y sont accompagnés par Pinch, un petit personnage qui apparaît dans les vidéos, commente ce qu'ils font et finit même par devenir… le héros d'un jeu de plateforme. Et pour corser le tout, jusqu'à trois équipes peuvent y jouer en même temps.

Les premières lignes de code datent de l'été 2024. Il a fallu beaucoup d'itérations, de soirées en salle et de corrections pour arriver à un jeu qui tourne tous les jours sans surprise. Aujourd'hui, la salle donne satisfaction, et ce sont les joueurs qui le disent mieux que moi :

<!-- Captures d'écran des avis Google -->
![Avis Google](./images/avis-google-1.png)
![Avis Google](./images/avis-google-2.png)

Dans cet article, je reviens sur les trois chantiers qui m'ont le plus occupé : **le plateformer**, **le Guitar Hero** et **la gestion du flux des joueurs** à travers les cinq salles. Et sur ce que j'en retiens.

## La Chocolaterie en quelques chiffres

| | |
|---|---|
| **5** | salles enchaînées, chacune avec son propre orchestrateur |
| **3** | équipes en jeu en même temps, au maximum |
| **312** | entrées/sorties électroniques déclarées (capteurs, relais, portes, ventilateurs…) |
| **78** | fichiers JSON de gameflow |
| **275** | handlers de notifications |
| **615** | commandes (audio, vidéo, électronique, agents spécifiques) |
| **341** | versions du gameflow publiées à ce jour |
| **13** | agents .NET spécifiques au jeu, plus un jeu Godot et un firmware ESP32 |

## L'architecture en deux mots

Tous nos jeux reposent sur **Maestro**, la plateforme que j'ai construite pour piloter nos salles. Le principe : des agents spécialisés qui communiquent par **MQTT**, et un **gameflow** décrit en JSON plutôt qu'en code.

```
                    ┌──────────────────────┐
                    │   GM UI  /  GM API   │   game masters
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Game Orchestrator   │   enchaîne les salles
                    └──────────┬───────────┘
                               │ MQTT
     ┌───────────┬─────────────┼─────────────┬──────────────┐
     ▼           ▼             ▼             ▼              ▼
  Room orch.  Room orch.   Room orch.    Room orch.     Room orch.
  R1 Candy    R2 Garden    R3 Elevator   R4 Ice room    R5 Inventing
     │
     ├── Electronic agent ── cartes d'E/S (capteurs, relais, portes)
     ├── Audio player       (Python / pygame)
     ├── Video player       (mpv, sur Raspberry Pi)
     └── Agents spécifiques (Guitar Hero, plateformer, machine à pompe…)
```

- Chaque **room orchestrator** exécute le gameflow de sa salle : il reçoit des notifications (un capteur, la fin d'une vidéo, un agent qui annonce une réussite), applique des filtres (tel jalon est-il validé ?) et publie des commandes.
- L'**electronic agent** traduit le format brut des cartes d'E/S (du genre `/c3m4p4/on` : contrôleur 3, module 4, broche 4) en clés métier comme `fanbtns_1_pressed`. Ce mapping, les *electronic proxies*, est lui aussi dans le gameflow.
- Les agents génériques (audio, vidéo, électronique, orchestrateurs, interface GM) ne contiennent **aucun code spécifique à un jeu**. Tout ce qui est propre à La Chocolaterie vit soit dans le gameflow, soit dans un agent dédié.

Ce choix « data-driven » a été déterminant. Les 341 versions du gameflow sont autant d'ajustements (un indice qui arrive trop tôt, une vidéo trop longue, un filtre manquant) que j'ai pu déployer **sans recompiler quoi que ce soit**.

## Chantier n°1 : un plateformer qui se joue sur un vrai décor

### Le concept

Dans la chambre froide, Pinch devient le héros d'un jeu de plateforme. Mais le but n'est pas d'afficher un jeu : c'est de donner **l'illusion que Pinch se déplace réellement dans la salle**. En production, le fond est noir et **seuls Pinch et les bulles qui simulent le vent sont affichés**. Tout le reste est physique : Pinch se déplace sur les **tuyaux** de la salle, et les ventilateurs qui le soulèvent sont de **vrais ventilateurs**, installés dans la pièce.

Les joueurs pilotent Pinch avec des commandes physiques (gauche, droite, saut) câblées sur l'électronique de la salle. Pour franchir certains passages, ils doivent activer ou désactiver les ventilateurs : le vrai souffle et la zone de vent du jeu se déclenchent ensemble. Au niveau 2, il faut **déplacer des tuyaux** en salle : leur position, lue par des capteurs, ouvre ou ferme les murs correspondants dans le jeu.

### La stack

- **Godot 4.4 édition .NET**, scripts en **C#**, rendu *GL Compatibility* ;
- **deux Raspberry Pi 4** (Linux arm64), un par niveau, lancés avec `-level 1` ou `-level 2` ;
- toutes les entrées arrivent par **MQTT** : en build release, le clavier n'existe plus.

### Un level design… invisible

Le premier vrai problème est géométrique. Le niveau existe bel et bien dans le jeu, avec ses murs et ses plateformes, mais il n'est jamais affiché en production : **les collisions doivent coller au plus près à des tuyaux que les joueurs voient, mais que le jeu, lui, ne voit pas**. Un mur virtuel décalé de quelques centimètres, et Pinch flotte dans le vide ou passe à travers un tuyau.

Le calage se fait donc sur site. Un build debug affiche tout le niveau, murs compris : c'est lui qui permet de vérifier, en salle, que les murs du jeu sont bien calés sur les tuyaux physiques. Les zones des ventilateurs, elles, n'étaient visibles qu'en debug, car leur affichage était figé à la compilation par `#if DEBUG`. J'en ai fait une option de lancement, utilisable directement sur le build release du Pi :

```bash
./plateformer.sh -level 2 -debug-zones
```

Les zones de vent apparaissent en rouge, les limites en vert, les zones d'émission en bleu, et on ajuste face aux vrais ventilateurs. C'est aussi comme ça qu'un « mur du frigo » est apparu dans le niveau 2, un an après la première version : le décor avait sa propre opinion sur la question.

### La physique des ventilateurs

Dans le jeu, un ventilateur, c'est un émetteur de particules (les bulles) et deux `Area2D` : une **zone de vent** qui pousse Pinch, et une **zone limite** au sommet où il flotte. Pour éviter un personnage figé en l'air, il oscille doucement autour de ce point :

```csharp
if (_isInVerticalFanLimit)
{
    // En haut du flux d'air : légère oscillation
    float oscillation = Mathf.Sin(_oscillationTime * OscillationSpeed) * OscillationStrength;
    velocity.Y = oscillation;
}
else if (_isInVerticalFan)
{
    // Dans le flux : poussée vers le haut
    velocity.Y += -350 * (float)delta;
}
else if (!IsOnFloor())
{
    velocity.Y += 1000.0f * (float)delta; // gravité
}
```

Côté électronique, j'ai câblé le vrai ventilateur **en réflexe** : le proxy du bouton déclenche directement la sortie du ventilateur sur la carte, sans aller-retour par l'orchestrateur ni par le jeu. Le souffle se déclenche donc à l'instant même où l'on appuie, et le jeu reçoit la même information en parallèle.

### Des indices qui viennent du jeu lui-même

Quand Pinch reste bloqué au pied d'un ventilateur, il entre dans une `DetectionZone`. La première fois, le jeu publie une notification (`plateformer_ventilation_pinch_stuck`), et le gameflow déclenche un indice audio. C'est un indice contextuel, qui dépend de la **position du personnage** et pas d'un simple chronomètre. Les game masters disposent en plus de commandes de déblocage (`solve_fans`, validation d'un niveau…).

### L'intro vidéo : des images plutôt qu'une vidéo

Godot ne lit nativement que l'Ogg Theora, et sur un Pi 4 le résultat ne m'a pas convaincu. L'intro a donc été découpée en images :

```bash
ffmpeg -i intro.mp4 -vf scale=960:540,fps=15 frames2/frame_%04d.jpg
```

Plus tard, j'ai remarqué que sur les 369 images, **80 étaient des copies exactes** de la précédente. En fusionnant chaque séquence en une seule entrée du `SpriteFrames`, avec une durée égale à sa longueur, j'ai obtenu **22 % de mémoire et de temps de chargement en moins**, pour un rendu identique.

### Le vrai boss de fin : MQTT

Godot n'a pas de client MQTT natif. Le jeu embarque donc un **client minimaliste sur WebSocket**, porté dans le projet. Il a fonctionné pendant des mois… jusqu'à ce que je cherche pourquoi la salle se bloquait de temps en temps. En l'auditant, j'ai trouvé une belle collection :

- **Aucune reconnexion, jamais.** La machine à états ne relançait une connexion que si `_reconnectTimer > 0`, mais après un échec ce timer valait `0`. Si le Pi démarrait avant le broker, le jeu restait sourd jusqu'au redémarrage.
- **Aucune connexion du tout en mode silencieux** : l'appel `ConnectToUrl(...)` était imbriqué dans un `if (_verboseLevel != 0)`.
- **Encodage des paquets** : la taille du `PUBLISH` était calculée avec `sz/127 + 1` au lieu de la vraie longueur variable. À partir de 255 octets, un octet nul en trop partait après le paquet et le broker coupait la connexion. Le ping de monitoring faisait 147 octets : on n'en était pas loin. La longueur restante de `CONNECT`/`SUBSCRIBE` tenait sur un seul octet, et le keep-alive de 120 s était annoncé à 124 s à cause d'un `OR 0x3C`.
- **Un seul paquet traité par frame** : une rafale de N messages était appliquée avec N-1 frames de retard.
- **Un paquet empoisonné** : le tampon de réception n'était pas vidé à la déconnexion. Un paquet tronqué restait en tête, faisait échouer chaque nouvelle connexion avant le `CONNACK`, en boucle.
- **La fin de niveau perdue** : `levelN_succeed` partait en QoS 0. Sur un lien mort pas encore détecté (jusqu'à environ 30 s), le message partait dans le vide, le niveau était déjà masqué, et l'orchestrateur ne l'apprenait jamais. **La salle restait bloquée.**
- Et pour finir, un *last will* `"BB"` publié sur le topic `last`, reste d'un test, ainsi que **trois générateurs d'identifiant client aléatoire** qui se marchaient dessus.

La correction a pris la forme d'une spécification avec des critères d'acceptation chiffrés, puis d'une série de petites PR :

- un **backoff exponentiel** (1, 2, 4, 8 puis 10 s plafonnées), un timeout de 5 s sur chaque état transitoire, et une détection de lien mort après deux `PINGRESP` manqués ;
- la fin de niveau publiée en **QoS 1**, conservée dans une *outbox* jusqu'au `PUBACK` et réémise à chaque reconnexion. On accepte un doublon possible, mais plus aucune perte ;
- un vrai *last will* « offline » sur le topic de monitoring ;
- un **identifiant client stable**, `choco-r4-plateformer-<hôte>-level<N>`. Le niveau en fait partie pour que deux Pi restés sur le nom d'hôte par défaut ne s'éjectent pas mutuellement du broker.

Surtout, toute cette logique a été extraite dans des **classes pures** (`MqttReconnectionPolicy`, `MqttPacketCodec`, `MqttCriticalNotificationOutbox`…), testées en xUnit **sans instancier le moteur Godot**, avec un temps injecté. Mille échecs de connexion consécutifs se simulent en quelques millisecondes.

Au passage, un autre bug discret : le joueur était instancié dès le chargement du niveau, invisible mais **physiquement actif**. Un Pinch fantôme pouvait donc atteindre la sortie avant même le début de la partie. Il est désormais désactivé (`ProcessMode.Disabled`) en dehors d'un niveau en cours.

## Chantier n°2 : un Guitar Hero à huit dans un ascenseur

### Le brief

Dans l'ascenseur, **huit boutons** sont répartis par paires à côté des écrans. Une vidéo accompagnée d'une bande-son fait défiler des notes, et les joueurs doivent appuyer au bon moment, parfois longtemps pour les notes tenues. Le cahier des charges tenait en quelques lignes, dont certaines étaient lourdes de conséquences :

- une difficulté adaptée **au nombre de joueurs** ;
- un jeu **de plus en plus permissif** à chaque échec, avec un nombre de recommencements limité ;
- un cadran en surimpression qui indique le nombre d'erreurs encore autorisées ;
- une note manquée compte comme une erreur ;
- la vidéo et l'audio sont gérés par d'autres agents.

Et une question restait ouverte dans le README d'origine : *« Comment tester le jeu ? Obligé d'être en salle, à plusieurs ? »*

### Le temps, ennemi n°1

Les notes sont décrites dans un `Scenarios.json` (début et fin en millisecondes sur la vidéo, `solo` pour les notes rouges du solo), validé au démarrage : un bouton inexistant ou une note qui se termine avant d'avoir commencé fait échouer le chargement. Le problème n'est pas là. Le problème, c'est que **le jeu et la vidéo ne partagent pas la même horloge**. Le jeu démarre son chronomètre quand il envoie la commande de lecture, et la vidéo démarre… un peu plus tard.

Dans le code, une constante témoigne de la façon dont j'ai d'abord traité le problème :

```csharp
// En réalité toutes les notes ont du retard sur leur prog, cause à identifier.
internal const int REALITY_LAG_SO_TIMELAPSE_TO_ADD = 1000;
```

C'est une compensation empirique, mesurée en salle. Elle fonctionne, mais en boucle ouverte. Depuis, le lecteur vidéo sait signaler le **vrai début de lecture**, et le jeu sait **recaler son horloge** dessus. Cette boucle fermée est prête mais encore dormante : l'activer imposera de recalibrer la constante, et en salle on ne touche pas à ce qui marche sans une session de test.

### Une seule boucle de jeu, sans verrou

Les appuis arrivent par MQTT, le minuteur de détection des notes manquées tourne sur son propre thread, et les commandes de démarrage et d'arrêt arrivent du gameflow. Les premières versions avaient évidemment des **races**. La version actuelle repose sur un modèle simple : tous les événements sont mis dans un `Channel<T>` et **un seul consommateur** modifie l'état du jeu. Les retours (son, cadran, voix de Pinch) partent dans un second canal vidé par une pompe dédiée, pour que la boucle ne bloque jamais sur le réseau.

```csharp
private void Dispatch(GameEvent gameEvent)
{
    switch (gameEvent)
    {
        case StartEvent: HandleStart(); break;
        case StopEvent:  HandleStop();  break;
        // Un événement daté d'une manche précédente porte un temps de l'ancienne horloge :
        // l'appliquer aux nouvelles notes marquerait des notes ratées ou compterait des erreurs.
        case ButtonPressedEvent e when e.Round == _round:
            HandleButtonPressed(e.ButtonId, e.TimestampMs);
            break;
        case TickEvent e when e.Round == _round:
            HandleTick(e.TimestampMs);
            break;
        // ...
    }
}
```

Ce numéro de **manche** (`_round`) a réglé un bug vicieux. Un appui mis en file juste avant un redémarrage était traité juste après, avec un horodatage de l'ancienne horloge (60 000 ms par exemple), et appliqué aux notes toutes neuves. Un seul tick pouvait alors marquer d'un coup sept notes comme ratées et déclencher un faux redémarrage.

### Les règles du jeu, version escape game

Quelques choix qui paraissent anodins, mais qui ont fait toute la différence :

- **Un appui résout au plus une note** : la plus ancienne encore en attente sur ce bouton. Une ancienne version pouvait en résoudre jusqu'à sept, ce qui faussait le score et provoquait des redémarrages injustes.
- **Une fenêtre large** : ±600 ms, plus une tolérance de 250 ms. On n'est pas en mode Expert. Les joueurs ne sont pas des gamers, ils sont huit, et ils découvrent les boutons.
- **Une note n'est déclarée manquée qu'après un délai de grâce** de 350 ms au-delà de sa fenêtre, pour qu'un appui tardif ne soit pas doublé par la vérification. Celle-ci tourne chaque seconde et ne marque pas plus de sept notes par passage, pour que le cadran descende progressivement.
- **La difficulté dépend du nombre de joueurs** (transmis par la session) : *easy* jusqu'à 2, *medium* à 3 ou 4, *high* au-delà.
- **Le jeu ne peut pas être perdu indéfiniment** : 15 erreurs autorisées au premier essai, 30 au deuxième, la tolérance s'élargit de 200 ms à chaque redémarrage, et au dernier essai autorisé la tolérance passe à 1 500 ms avec un plafond d'erreurs inatteignable. **La victoire est garantie.**

Ce dernier point résume une règle que je considère comme fondamentale : **dans un escape game, une épreuve ne doit jamais être une impasse.** L'équipe doit sortir de l'ascenseur, avec le sentiment d'avoir mérité sa victoire.

### Tester sans être huit dans un ascenseur

La réponse à la question du README s'appelle `GuitarHero.AutomaticTest` : un outil qui **rejoue des scénarios d'appuis et de relâchements** horodatés sur MQTT, exactement comme les vrais boutons. Il y a des scénarios « parfaits », d'autres volontairement imprécis, d'autres qui doivent provoquer un redémarrage. On lance le jeu, on lance le scénario, on regarde le cadran. Ce n'est pas un test unitaire au sens strict, mais c'est infiniment plus reproductible que « on y retourne à quatre ce soir ».

## Chantier n°3 : faire circuler les joueurs dans cinq salles

### Trois équipes en même temps

La Chocolaterie n'accueille pas une équipe à la fois. **Jusqu'à trois équipes peuvent jouer en même temps** : une en salle 1, une en salle 3, une en salle 5. C'est le fonctionnement normal de la salle, et c'est très fréquent le week-end.

Cela change complètement la nature du problème. Une salle n'est plus une partie isolée, c'est une étape d'un pipeline : chaque équipe avance avec une salle d'écart sur la précédente, et cet écart doit tenir du début à la fin. Une équipe qui prend du retard, c'est l'équipe suivante qui la rattrape.

L'architecture s'y prête bien : chaque salle a son propre room orchestrator, avec son propre gameflow et son propre état. Trois parties tournent donc en parallèle sans se marcher dessus, et les game masters disposent d'une vue multi-salles pour les suivre.

### L'enchaînement

Chaque salle a son propre **room orchestrator**, mais une équipe vit une seule partie. C'est le **game orchestrator** qui fait le lien : quand une salle publie `room_succeed`, il enregistre la fin, trouve la salle suivante et la démarre **avec la même session**. Tout commence par la première salle (préparation, briefing, lancement), et les suivantes sont démarrées par l'enchaînement. La session suit l'équipe, pas la salle : son nom, sa langue et son nombre de joueurs (celui qui règle la difficulté du Guitar Hero) passent ainsi de salle en salle, même avec trois équipes dans le bâtiment.

Les transitions physiques sont scénarisées. Dans le sas de l'ascenseur, par exemple, la vidéo d'accueil **boucle tant que les joueurs n'ont pas fermé la porte** :

```json
{
  "notification": "r3_sas_play_ended",
  "description": "Fin de la scéno, on recommence si les joueurs n'ont pas fermé la porte",
  "filters": [{ "behavior": "milestone_not_validated", "key": "r3_door_closed" }],
  "behaviors": [
    { "behavior": "command_publish", "key": "r3_sas_entrance_video_play" },
    { "behavior": "command_publish", "key": "r3_sas_entrance_audio_play" }
  ]
}
```

### Le budget temps : des indices, puis des filets de sécurité

C'est le vrai défi de conception. Dans une salle unique, une équipe en retard finit simplement sans avoir terminé. Dans un parcours en cinq actes, **une équipe bloquée en salle 2 ne verra jamais les salles 3, 4 et 5**, qui sont pourtant le cœur de l'expérience. Et avec trois équipes en jeu, elle retarde aussi **l'équipe qui arrive derrière elle**. Le budget temps de chaque salle n'est donc pas un confort, c'est une contrainte d'exploitation.

Chaque salle a donc un budget temps, géré en deux couches dans le gameflow :

1. **Des indices automatiques progressifs**, déclenchés par des *scheduled actions* relatives à un jalon. Par exemple, 4 minutes après la pose du badge dans la caisse enregistreuse, un indice sur les jarres, puis 8 minutes après, un indice sur les robinets, et ainsi de suite. Chaque indice est conditionné : s'il n'est plus utile, il ne part pas.
2. **Des validations forcées**, en dernier recours, qui font avancer la salle à la place des joueurs, de façon scénarisée.

| Salle | Filets de sécurité (temps depuis le déclencheur) |
|---|---|
| R1 – Candy Store | 2, 8, 23 et 25 min |
| R2 – Chocolate Garden | 12, 22 et 30 min |
| R3 – Elevator | 2 et 18 min (et le Guitar Hero imperdable) |
| R4 – Ice Room | trappe ouverte à 10 min, plateformer validé à 40 min |
| R5 – Inventing Room | 9, 12 et 22 min |

Dans la chambre froide, si le digicode n'est pas trouvé au bout de 10 minutes, c'est Pinch qui l'ouvre lui-même. Le joueur ne vit pas un échec, il vit une péripétie :

```json
{
  "description": "Au bout de 10 minutes, on ouvre le conduit",
  "trigger_type": "room_event",
  "trigger_key": "room_started",
  "delay": 600,
  "conditions": [{ "behavior": "milestone_not_validated", "key": "r4_digicode_unlocked" }],
  "behaviors": [
    { "behavior": "command_publish",   "key": "trappe_unlock" },
    { "behavior": "milestone_validate", "key": "r4_digicode_unlocked" },
    { "behavior": "command_publish",   "key": "digicode_unlocked_bypinch_audio_play" }
  ]
}
```

Tout cela s'ajoute aux game masters, qui voient les jalons en temps réel, peuvent envoyer des indices, valider une épreuve ou débloquer un ventilateur depuis leur interface. L'automatique gère le cas général, et l'humain garde la main.

### Arrêter proprement, c'est aussi du gameplay

Un point que j'ai sous-estimé au début : **l'arrêt d'une salle**. Un stop du game master doit ouvrir toutes les portes et trappes, mettre les vidéos de Pinch au noir, masquer le plateformer, couper les effets. Plusieurs corrections récentes du gameflow ne concernent que ça, par exemple « l'arrêt GM masque Pinch et le plateformer », ou « la validation GM du niveau 2 masque le niveau et Pinch ». Et comme les équipes se suivent de près, un état résiduel à l'arrêt devient très vite un bug pour l'équipe suivante.

## Exploitation : la salle doit tourner sans moi

Un escape game, c'est plusieurs parties par jour, tous les jours. Ici, jusqu'à trois en parallèle. Quelques briques qui ne se voient pas mais qui comptent :

- **La sécurité.** L'agent d'urgence écoute l'entrée de l'alarme incendie et met la salle en sécurité côté logiciel (sons coupés, lumière, déverrouillages). La sécurité vitale, elle, **reste câblée en dur, hors logiciel**. Le lecteur audio garde sa propre surveillance de l'alarme : il y a deux chemins indépendants vers le silence, et un seul chemin, explicite, vers la reprise. Un reset est refusé tant que l'alarme est encore active.
- **Les coupures de courant.** La remise en route autonome d'une salle après une coupure fait l'objet d'un banc de test dédié, sur une VM qu'on coupe brutalement.
- **Les déploiements.** Le plateformer se déploie en une commande sur le Pi, avec une sauvegarde de la version précédente et une procédure de retour arrière documentée.
- **Les cartes SD.** Pendant une coupure du broker, les avertissements MQTT ajoutaient des milliers de lignes de log par jour sur la carte SD d'un Pi qui tourne en continu. J'ai donc mis en place un `logrotate` en `copytruncate`, pour ne pas avoir à relancer le jeu.

Dernière étape en date : une phase de **durcissement systématique**. J'ai mis en place une « usine logicielle » multi-agents : une spécification avec des critères d'acceptation testables, une validation humaine, l'implémentation, puis une vérification adversariale. Elle a permis de traiter des dizaines de petits défauts latents sous forme de PR courtes et relisibles. Le client MQTT du plateformer et le Guitar Hero en ont été les premiers bénéficiaires.

## Ce que j'en retiens

1. **Le monde physique ment.** Latence média, capteurs qui rebondissent, décor qui ne correspond pas au plan. Toute constante de timing doit être mesurée en salle, et documentée comme telle.
2. **Une épreuve ne doit jamais être une impasse.** Tolérance croissante, indices progressifs, validations forcées scénarisées : le joueur doit toujours avancer, et avoir l'impression d'y être pour quelque chose.
3. **Penser pipeline.** Avec trois équipes en parallèle, une salle n'est plus une partie mais une étape : son temps de traversée compte autant que son contenu.
4. **Data-driven ou rien.** 341 versions du gameflow, c'est 341 fois où je n'ai pas eu à recompiler un agent pour décaler un indice de deux minutes.
5. **Les briques « simples » sont celles qui cassent.** Le gameplay du plateformer a été stable très tôt. C'est un client MQTT de quelques centaines de lignes qui bloquait la salle.
6. **Rendez testable ce qui ne l'est pas.** Un moteur de jeu, un ascenseur, huit boutons : rien de tout ça ne tourne en CI. En revanche, une politique de reconnexion, un codec de paquets ou une boucle d'événements, si. Il faut extraire ces parties, et on peut les tester.
7. **Un client qui disparaît doit le dire.** Last will, identifiants stables, logs explicites : en exploitation, savoir *qui* est tombé vaut autant que de le relever.

La Chocolaterie a été un vrai défi, technique autant que de game design. Voir aujourd'hui des équipes ressortir de la salle des inventions avec le sourire, et le raconter ensuite dans leurs avis, c'est la meilleure récompense pour toutes ces heures passées à caler des murs invisibles et à traquer des octets nuls.

<!-- Captures d'écran des avis Google -->
![Avis Google](./images/avis-google-3.png)
