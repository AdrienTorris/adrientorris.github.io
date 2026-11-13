# Le lobby est une salle Maestro (sans partie)

*Comment nous avons migré l'espace d'accueil de notre agence de Lille sur notre plateforme d'orchestration d'escape games, presque sans écrire de code.*

Chez Team Break, le **lobby** est l'espace commun de l'agence. Les joueurs y attendent avant leur mission. Les game masters (GM) les y rassemblent pour le **brief** : la mise en situation avant d'entrer dans la salle. Le lobby n'est pas une salle de jeu, mais c'est la première chose que les clients voient et entendent. Il a sa musique, ses lumières devant chaque univers, et même un petit jeu caché.

Jusqu'à cet automne, tout cela tournait sur l'ancien contrôleur du prestataire : un Raspberry Pi sous Node-RED, relié à deux panneaux d'entrées-sorties. Il faisait le travail, mais il vivait à part de nos outils : aucune interface GM, pas de bouton de brief par salle, rien de versionné.

Nos salles, elles, migrent une à une sur **Maestro**, notre plateforme maison. D'où l'idée, simple sur le papier : **traiter le lobby comme une salle Maestro à part entière**. Le lobby de Lille a ouvert la voie et il est en service. Ceux de Paris 9, de La Défense et d'Aéroville sont en cours d'installation. Ce billet raconte comment nous l'avons fait à Lille, ce que la plateforme nous a donné gratuitement, et les quelques pièges rencontrés en route.

---

## Maestro en deux minutes

Pour situer la suite :

- **Une stack par salle** (paquet Debian `maestro-room`, Docker Compose), en .NET : une API et une interface pour les GM (la **GM UI**), un **room orchestrator** qui exécute le scénario de la salle, un **agent électronique** qui traduit les messages des panneaux d'entrées-sorties, et quelques autres agents.
- Tout ce monde parle en **MQTT**.
- Autour de la stack, des **agents hôte** : notre lecteur audio piloté en MQTT (MQTTaudioplayer), `blinker-agent` pour les séquences lumineuses, `io-status-agent` pour dialoguer avec les panneaux.
- Le comportement d'une salle n'est **pas du code**. C'est un **gameflow**, un ensemble de fichiers JSON dans un dépôt GitHub par salle, que l'orchestrateur charge au démarrage (et met en cache pour fonctionner hors ligne).
- Un gameflow, c'est surtout un vocabulaire :
  - des **commandes** (jouer un son, allumer une sortie) ;
  - des **notifications** (un bouton GM, une entrée électronique, la fin d'un son) ;
  - des **handlers** qui enchaînent des *behaviors* quand une notification arrive, sous réserve de **filtres** ;
  - des **jalons** (*milestones*), c'est-à-dire des étapes validées ou non.
- Les données de la GM UI (salles, boutons, catégories) sont importées par des **fichiers `_datas` versionnés** : une initialisation, puis des patchs.

La règle d'or du dépôt `maestro-agents` : **aucun code spécifique à un jeu**. Tout ce qui est propre à une salle vit dans son gameflow.

---

## Le besoin

Le cadrage tenait en quelques lignes :

1. Une GM UI avec **un bouton par salle** : « Lancer le brief Magic School », « Lancer le brief Haunted Room »…
2. Chaque bouton **lance la bonne musique de brief**. À la fin, on revient au **mode par défaut** : une ambiance qui tourne toute la journée.
3. **Plus tard**, des effets de lumière à chaque brief.
4. La **sonnette de l'hôtel** devant la Haunted Room. Un client appuie : un son de sonnette, puis une ambiance spéciale et les lumières de la façade qui s'affolent.

Le lobby de Lille dessert 11 salles et 13 briefs (deux salles ont une variante : Halloween pour la Magic School, Asylum pour la Prison Break). Il compte deux panneaux d'entrées-sorties : une entrée (la sonnette) et 17 sorties (logos, fenêtres, couloirs, spots).

La première conclusion du cadrage a été la bonne surprise du projet : **la phase 1 ne demandait aucun développement**. Tout existait déjà dans Maestro et dans le lecteur audio. Il fallait un gameflow et un serveur. Nous n'avons touché au code de la plateforme qu'une fois, pour les états Start / Stop (voir plus bas).

---

## L'architecture

```
 Poste GM de l'accueil (GM UI)
        │  bouton « Prison Break / Lancement »
        ▼
 GM API ──MQTT──► room orchestrator (gameflow du lobby)
                     │
                     ├──► MQTTaudioplayer ──► ampli du lobby
                     │         ▲
                     │         └── « lobby_brief_ended » à la fin du fichier
                     │
                     ├──► blinker-agent ─────┐  séquences lumineuses
                     │                       ▼
                     └──► agent électronique ──► broker IO ◄──► panneaux c0 / c1
                                 ▲                                │
                                 └──── sonnette de l'hôtel ◄──────┘
```

Le lobby a **son propre serveur** : le Raspberry Pi 4 qui faisait tourner l'ancien contrôleur, reflashé sous Raspberry Pi OS Lite 64 bits (Debian 13 « Trixie »). La stack `maestro-room` existe en `arm64`, il lui faut 4 Go de RAM. Nous avons écarté l'idée d'héberger le lobby sur le serveur d'une salle (une deuxième sortie audio aurait suffi). Avec un serveur dédié, une mise à jour de salle ne coupe pas le lobby, et le lobby ne fait courir aucun risque à une partie en cours.

---

## Une salle… sans partie

Une salle Maestro, c'est une machine à états : préparation, briefing, Start, fin de partie, chrono. Le lobby n'a rien de tout ça. Première version, purement déclarative : on désactive tout.

```json
{
  "key": "lobby",
  "name": "Lobby",
  "automatic_end_enabled": false,
  "is_prepare_enabled": false,
  "is_start_enabled": false,
  "is_state_machine_active": false,
  "is_session_evaluation_popup_enabled": false,
  "is_add_time_proposal_enabled": false
}
```

Ça marche parce que deux comportements de Maestro, vérifiés dans le code avant de nous lancer, jouaient en notre faveur :

- l'API GM exécute un bouton **sans vérifier qu'une session est en cours** ;
- l'orchestrateur envoie la notification d'un bouton directement aux handlers. Machine à états inactive, il n'y a plus aucun contrôle d'état.

Les boutons GM fonctionnent donc en permanence.

### Puis il a fallu Start et Stop

Une semaine après le cadrage, les équipes ont voulu un bouton qui lance le mode par défaut et un bouton qui coupe tout. Ce sont les états standard de Maestro. Mais sans préparation, le bouton Start restait **grisé en permanence** : aucun chemin de la machine à états n'y menait. Ironie : l'orchestrateur acceptait déjà ce Start, puisqu'il ne contrôle l'état que si la machine à états est active. Le blocage était seulement côté domaine.

D'où notre seul développement (maestro-room 1.17.3) :

```csharp
private bool IsStartWithoutPreparationAllowed => !IsPrepareStateEnabled && !IsStateMachineActive;

public bool IsStartAvailable()
{
    if (!IsStartStateEnabled)
    {
        return false;
    }

    // Salle sans préparation (lobby, salle sans session) : seuls Start et Stop existent, le Start part du
    // repos. Le Started de la session précédente survit au Stop : il ne bloque pas le cycle suivant ;
    // seule une session en cours le fait.
    if (IsStartWithoutPreparationAllowed)
    {
        return StateId == (int)RoomStateEnum.StartAsked
            || ((StateId == (int)RoomStateEnum.Undefined || StateId == (int)RoomStateEnum.AtRest)
                && !GameSessionId.HasValue);
    }

    // … parcours historique des salles avec préparation, inchangé
}
```

Le changement est volontairement étroit. Seules les salles dont la préparation **et** la machine à états sont désactivées sont concernées, et les autres salles ne voient aucune différence. Côté données, un patch active Start et Stop, désactive le briefing (activé par défaut) et fixe la durée à **720 minutes**, pour que le chrono de la GM UI n'arrive pas à zéro dans la journée. La fin automatique reste désactivée : rien ne se passe à 0.

Start et Stop deviennent alors deux simples `game_event_handlers` :

| Événement | Ce que fait le gameflow, dans l'ordre |
|---|---|
| `room_started` | valide le jalon `lobby_running`, lève les verrous, relance l'ambiance depuis le début, arrête les séquences lumière, met la façade Haunted Room au repos, lance l'alternance du couloir Prison Break |
| `room_stopped` | lève les jalons **puis** coupe les sons, arrête toutes les séquences, éteint les lumières des 5 salles équipées |

L'ordre du Stop n'est pas un détail : si le lecteur publie une fin de son pendant l'arrêt, plus aucun verrou n'est valide et rien n'est relancé. L'orchestrateur, de son côté, annule les handlers différés en attente avant d'exécuter ceux du Stop.

Au démarrage du serveur, l'orchestrateur ne fait qu'une chose : envoyer le catalogue des pistes au lecteur audio, 10 secondes après sa connexion au broker. **Le lobby reste silencieux jusqu'au Start.** C'était un choix explicite des équipes, et c'est le comportement le plus prévisible après une coupure de courant.

---

## Lancer la bonne musique au brief

C'est le cœur du besoin. Le GM appuie sur « Lancement » : l'ambiance s'efface, le brief joue, l'ambiance revient.

```
Bouton GM « Koh Lanta / Lancement »
  → notification lobby_brief_koh_lanta_asked
  → room orchestrator : verrou lobby_brief_running
                        lobby_ambiance_fade_out   (ambiance → 0 en 3 s, sans s'arrêter)
                        lobby_brief_koh_lanta_play (solo, at_end_message = lobby_brief_ended)
  → fin du fichier : le lecteur publie « lobby_brief_ended » vers l'orchestrateur
  → room orchestrator : lobby_ambiance_fade_in    (ambiance → 0,6 en 3 s, là où elle en était)
                        verrou levé
```

La commande de brief :

```json
{
  "key": "lobby_brief_prison_break_play",
  "recipients": [ { "type": "audio_player" } ],
  "format": "audio_player_play",
  "datas": [
    { "key": "track",          "value": "brief_prison_break" },
    { "key": "volume",         "value": "1.0" },
    { "key": "solo",           "value": "true" },
    { "key": "at_end_message", "value": "lobby_brief_ended" }
  ]
}
```

Et le retour au mode par défaut, commun aux 13 briefs :

```json
{
  "key": "lobby_brief_ended",
  "notification": "lobby_brief_ended",
  "behaviors": [
    { "behavior": "command_publish", "key": "lobby_ambiance_fade_in", "position": 1 },
    { "behavior": "milestone_reset", "key": "lobby_brief_running",    "position": 2 }
  ]
}
```

Quelques propriétés du lecteur audio font tout le travail :

- **Un fondu n'arrête pas la piste.** L'ambiance tourne à volume nul pendant le brief et reprend exactement là où elle en était, sans repartir du début.
- **`solo` dans le groupe `lobby_brief`.** Un brief lancé pendant un autre coupe le premier. Le brief coupé ne publie pas sa fin, c'est le second qui fera remonter l'ambiance.
- **Un arrêt volontaire ne publie jamais `at_end`.** Le bouton « Arrêter le brief » fait donc remonter l'ambiance lui-même.
- **Le message de fin est un texte brut**, que l'orchestrateur traite comme une clé de notification, **même hors session**.

### La solution évidente, et pourquoi nous l'avons écartée

Le lecteur audio a une commande faite pour ça : `notify`. Elle baisse les autres pistes (*ducking*), joue le son, les remonte à la fin, et sait même mettre les sons en file d'attente. C'était la solution la plus simple, sans aller-retour par l'orchestrateur.

En la rejouant dans le harnais d'intégration du lecteur, nous avons trouvé un défaut. **Un `notify` arrêté par une commande (`stop`, `stopall`) n'est jamais nettoyé** : les volumes baissés ne sont pas restaurés, et la piste reste marquée comme notification active. Après un « Arrêter le brief », l'ambiance restait à 0. Pire : à la fin de **tout brief suivant**, elle n'était plus remontée, « encore baissée par une autre notification ».

Nous avons donc choisi un fondu explicite, plus `play` avec `at_end`. Le prix est connu et documenté : la remontée de l'ambiance passe par l'orchestrateur. S'il redémarre pile à la fin d'un brief, l'ambiance reste à 0 jusqu'au bouton de secours « Tout arrêter et relancer l'ambiance ». Le correctif du lecteur est proposé à part, il servira aussi aux salles.

### Les petits pièges du format

| Piège | Conséquence | Parade |
|---|---|---|
| `audio_player_play` **ne transmet pas** l'option `replay` au lecteur | Impossible d'écrire « lance l'ambiance si elle ne joue pas déjà » | Commande au format `raw`, avec le JSON du lecteur écrit à la main |
| Une commande `raw` sans `mqtt_topic` part sur `/<game>/raw` | Message perdu, personne n'écoute ce topic | Topic explicite : `/games/lobby/rooms/lobby/audioplayer` |
| `_electronic_agent_settings.json` est obligatoire | Le gameflow ne se charge pas (« No providers »), même sans électronique | Provider déclaré sur le broker des panneaux |
| Description d'un bouton GM limitée à 320 caractères en base | Bouton absent si elle est trop longue | Descriptions courtes |

La commande `raw` en question :

```json
{
  "key": "lobby_ambiance_ensure",
  "format": "raw",
  "datas": [
    { "key": "mqtt_topic",   "value": "/games/lobby/rooms/lobby/audioplayer" },
    { "key": "mqtt_message", "value": "{\"command\":\"play\",\"data\":{\"track_id\":\"lobby_ambiance\",\"loop\":-1,\"fade_in\":3000,\"volume\":0.6,\"replay\":false}}" }
  ]
}
```

---

## L'ambiance audio : un seul fichier, et c'est un choix

Le mode par défaut, c'est une vingtaine de morceaux joués à la suite. Nous aurions pu les enchaîner un par un avec `at_end`. Mais **chaque transition aurait alors dépendu de l'orchestrateur** : un redémarrage, et le lobby se tait au milieu de la journée.

Nous avons donc fait l'inverse. Les morceaux sont convertis, égalisés à −16 LUFS et **mis bout à bout dans un seul WAV**, que le lecteur joue en boucle infinie (`loop: -1`). Au format WAV 16 bits / 44,1 kHz, il le lit **depuis le disque**, bloc par bloc, pour environ 540 Ko de RAM quelle que soit la durée. Sur un Raspberry Pi, ça compte. L'ambiance ne dépend plus que du lecteur audio.

Le catalogue du lecteur tient dans une commande d'initialisation (18 pistes, deux groupes) :

```json
{ "key": "track_1", "value": "key~lobby_ambiance|path~/medias/audios/lobby_mix.wav|group~lobby_ambiance" },
{ "key": "track_2", "value": "key~brief_koh_lanta|path~/medias/audios/kl.wav|group~lobby_brief" }
```

Le revers est assumé : **changer la playlist, c'est refaire le mix**, le recopier au même endroit, puis cliquer « Tout arrêter et relancer l'ambiance ». C'est cette limite qui nous a poussés vers une vraie gestion de la playlist, en test à Lille (voir « Ce qui vient »).

---

## Les ambiances lumineuses

Les 17 sorties du lobby sont reprises du relevé de l'ancien contrôleur, sous forme de **proxies** de l'agent électronique : 36 en tout, un `on` et un `off` par entrée-sortie. Elles sont pilotables une par une depuis la page technique dès l'installation. Un handler allume une sortie par un simple `command_publish` de sa clé (`lobby_lost_wall_light_on`…) : le chargeur crée la commande et l'adresse à l'agent électronique.

Pour les **séquences** (clignotements, alternances, minuteries), nous utilisons `blinker-agent`, qui joue des suites de messages vers les panneaux. Le mode normal du couloir Prison Break, une alternance bleu / rouge toutes les 2 secondes, ressemble à ceci :

```json
{
  "command": "sequence",
  "datas": {
    "ID": "lobby_prison_break_rest",
    "loop": -1,
    "sequence": [
      { "messages": ["/c0m1p2/off", "/c0m1p1/on"],  "time": 2000 },
      { "messages": ["/c0m1p2/on",  "/c0m1p1/off"], "time": 2000 }
    ]
  }
}
```

Ce que nous avons appris de blinker-agent, à nos dépens ou en lisant son code :

- `time` est le délai **avant** l'étape, et vaut 1 000 ms s'il est absent : la première étape doit avoir `time: 0`.
- Dans une boucle, la première étape suit la dernière sans délai : chaque étape porte donc son propre délai. Pour l'alternance, le bleu est allumé juste avant de lancer la séquence.
- Renvoyer une séquence de même `ID` la **relance du début**. Nous nous en servons : le brief Haunted Room et la sonnette partagent le même `ID`.
- `stop` **fige** les sorties dans leur dernier état. Chaque arrêt est donc suivi d'un état explicite (« façade au repos » : fenêtres éteintes, logo allumé).
- `stop_all` accepte une liste d'exclusions. « Tout arrêter et relancer l'ambiance » arrête toutes les séquences sauf l'alternance du couloir et les spots de la Donut Quiz :

```json
{ "command": "stop_all", "datas": { "exclude": ["lobby_prison_break_rest", "lobby_donut_quiz_scene"] } }
```

Les spots de la Donut Quiz montrent bien ce que permet blinker-agent. Au lancement du brief, les 3 spots devant la salle s'allument, puis s'éteignent seuls 3 minutes après. Nous aurions pu écrire un handler différé de 180 secondes. Mais une séquence blinker à deux étapes **s'annule**, ce qu'un handler différé ne permet pas. Si un GM rallume les lumières à la main, ou si un autre brief les éteint, l'extinction automatique est annulée proprement.

### Lancement light, lancement complet

Une demande arrivée en cours de route : pour certaines salles, le brief doit **éteindre les lumières de toutes les autres salles**, pour que seul l'univers lancé reste éclairé.

- **Lancement light** : le son et les effets de la salle.
- **Lancement complet** : la même chose, puis un `command_bundle_publish` de `lobby_<salle>_lights_off` pour chaque autre salle équipée. Chaque bundle arrête d'abord la séquence de sa salle, puis éteint ses sorties.

```json
{
  "key": "lobby_prison_break_lights_off",
  "commands": [
    { "key": "lobby_prison_break_scene_stop",   "position": 1 },
    { "key": "lobby_prison_break_rest_stop",    "position": 2 },
    { "key": "lobby_prison_break_blue_led_off", "position": 3 },
    { "key": "lobby_prison_break_red_led_off",  "position": 4 }
  ]
}
```

Une semaine plus tard, les équipes ont tranché : un seul bouton « Lancement » par salle, « complet » pour les salles qui ont des lumières dans le lobby, « light » pour les autres. Seules la Haunted Room et la Lost gardent les deux. Les handlers des deux variantes sont restés dans le gameflow : remettre un bouton ne demande qu'un patch de données.

---

## Le petit jeu : la sonnette de la Haunted Room

C'est le morceau le plus amusant. Devant la façade de la Haunted Room, un vieil hôtel hanté, il y a une sonnette de réception. Un client curieux appuie :

1. l'ambiance du lobby s'efface en 3 s, et **un des 3 sons de sonnette** joue, tiré au hasard ;
2. à la fin de la sonnette, une **ambiance spéciale Haunted Room** démarre, et les fenêtres et le logo de la façade se mettent à clignoter de plus en plus vite, puis tout s'éteint, puis le logo seul palpite ;
3. 10 secondes plus tard, tout revient à la normale.

Le tout dure 10 à 15 secondes, à la demande de l'agence : la première version durait toute l'ambiance spéciale, et les sons de sonnette s'entendaient à peine (nous les avons remontés à une crête de −1 dBFS avec un petit script `ffmpeg`). Les règles du jeu sont simples à énoncer :

- ignorée quand le lobby est arrêté ;
- ignorée **pendant un brief** : un client ne doit pas pouvoir couper le GM ;
- ignorée **pendant sa propre séquence** : dix coups de sonnette ne la relancent pas dix fois.

### Des jalons cachés comme verrous

Plutôt que d'inventer une notion de variable, nous avons détourné les **jalons**. Un jalon peut être **caché** dans la GM UI. La validation et la remise à zéro d'un jalon fonctionnent hors session : l'état est enregistré, seul l'événement de session échoue, sans erreur. Nous les utilisons comme verrous :

| Jalon caché | Validé par | Levé par |
|---|---|---|
| `lobby_running` | Start | Stop |
| `lobby_brief_running` | chaque brief | fin du brief, « Arrêter le brief », « Tout arrêter », Start, Stop |
| `lobby_haunted_room_bell_running` | la sonnette | fin de sa séquence, un brief, Start, Stop |

Le handler de la sonnette devient une simple garde :

```json
{
  "key": "lobby_haunted_room_hotel_bell_pressed",
  "notification": "lobby_haunted_room_hotel_bell_pressed",
  "filters_behavior": "and",
  "filters": [
    { "behavior": "milestone_not_validated", "key": "lobby_brief_running",             "position": 1 },
    { "behavior": "milestone_not_validated", "key": "lobby_haunted_room_bell_running", "position": 2 },
    { "behavior": "milestone_validated",     "key": "lobby_running",                   "position": 3 }
  ],
  "behaviors": [
    { "behavior": "milestone_validate", "key": "lobby_haunted_room_bell_running", "position": 1 },
    { "behavior": "command_publish",    "key": "lobby_ambiance_fade_out",         "position": 2 },
    { "behavior": "command_publish",    "key": "lobby_haunted_room_bell_play",    "position": 3 }
  ]
}
```

Le tirage au hasard ne demande aucun code. C'est un format de commande existant, `audio_player_play_one_random` : l'orchestrateur tire la piste et envoie un `play` ordinaire au lecteur.

```json
{
  "key": "lobby_haunted_room_bell_play",
  "recipients": [ { "type": "audio_player" } ],
  "format": "audio_player_play_one_random",
  "datas": [
    { "key": "tracks",         "value": "haunted_room_bell_1|haunted_room_bell_2|haunted_room_bell_3" },
    { "key": "volume",         "value": "1.0" },
    { "key": "at_end_message", "value": "lobby_haunted_room_bell_ended" }
  ]
}
```

La fin de séquence est un **handler différé** (`"delay": 10`) sur la même notification de fin de sonnette. Point important : l'orchestrateur **réévalue ses filtres au moment où il part**. Si un brief a été lancé entre-temps, il ne fait rien, et le brief garde la main.

### Le bug du dernier élément

En validant le tirage, nous avons remarqué que le troisième son ne sortait jamais. Les développeurs .NET ont sans doute déjà deviné. En simplifiant :

```csharp
var index = random.Next(0, tracks.Length - 1); // borne supérieure exclusive : le dernier n'est jamais tiré
```

`Random.Next(minValue, maxValue)` exclut `maxValue`. Le correctif tenait en une ligne, publié dans le paquet d'infrastructure de messagerie puis dans maestro-room 1.16.3. Contrôle sur un broker local avec le vrai *communicator* de l'orchestrateur : 300 appuis, 96 / 88 / 116 tirages. Le même bug touchait `notify_one_random` et l'utilitaire `FirstRandom`, donc d'autres salles. Certains gameflows doublaient même leur dernière piste pour la faire sortir : ce doublon est à retirer, sinon la piste sort maintenant deux fois plus souvent.

### Le piège de l'ordre des handlers

Dernière subtilité. Quand un brief est lancé pendant la séquence de la sonnette, un handler de nettoyage arrête le scénario et remet la façade au repos (logo **allumé**). Mais un **lancement complet** doit, lui, éteindre la façade.

Or l'orchestrateur exécute les handlers d'une même notification l'un après l'autre, **ceux qui n'écoutent qu'une notification d'abord**. Le nettoyage, qui en écoute quatorze, passerait donc après le lancement complet et rallumerait le logo que celui-ci vient d'éteindre. Solution : le nettoyage n'écoute pas les lancements complets, qui gèrent eux-mêmes la sonnette (arrêt du scénario, extinction, verrou levé). La règle est écrite dans le README du gameflow, à l'étape « Ajouter un brief ».

---

## La gestion par les données

### Les boutons GM sont des patchs versionnés

Les boutons de la GM UI ne sont pas dans le gameflow, mais en base, importés par des fichiers `_datas`. Deux règles, apprises en lisant le dépôt de données de la GM API :

- **Ne jamais modifier le fichier d'initialisation** d'une base déjà initialisée. Tout passe par un patch versionné (`add`, `update`, `delete`), appliqué par `POST /datas/run`.
- **Ne jamais supprimer une catégorie** de boutons. La suppression d'une catégorie supprime aussi les boutons qu'elle contient encore, même si leur déplacement a échoué juste avant (une opération de patch en échec n'arrête pas les suivantes). Quand nous avons regroupé les boutons par salle, l'ancienne catégorie « Briefs de mission » est donc devenue… « Koh Lanta ».

Le passage à « un seul lancement par salle » tient dans un patch : 11 boutons supprimés (leurs actions et filtres partent en cascade) et 11 renommés.

### Onze versions en deux semaines

Le gameflow est passé de 0.1.0 (27 octobre) à 0.11.0 (9 novembre). Chaque version a une entrée datée dans `_versions.json`, qui dit aussi **dans quel ordre la déployer** : patch de données sans reset, puis redémarrage de l'orchestrateur, ou l'inverse. Mettre à jour le lobby, c'est un push, puis un bouton « Restart room-orchestrator » sur la page technique. Ce rythme a permis d'itérer directement avec l'équipe de l'agence : quatre versions le seul 3 novembre, dont la sonnette raccourcie et l'alternance du couloir Prison Break, demandées le jour même.

### Valider sans le matériel

Nous avons écrit l'essentiel du gameflow avant d'avoir le serveur et les sons. Pour ne pas découvrir les erreurs sur place, chaque brique a été rejouée avec le **vrai code** :

- le chargement, par le vrai lecteur de gameflow de `maestro-infrastructure` : aucune erreur, chaque `at_end_message` a son handler, chaque bouton a sa notification ;
- les `_datas`, désérialisés avec les modèles d'import de la GM API ;
- le comportement audio, rejoué dans le harnais d'intégration du lecteur (vrais contrôleurs, mixeur simulé) : fin de brief, brief qui en coupe un autre, arrêt puis brief d'une autre salle ;
- les scénarios lumière, déroulés dans la classe `Sequence` de blinker-agent.

Sur place, il ne restait que ce qu'aucun test ne remplace : les volumes, les inversions d'adresses, et le ressenti.

---

## L'exploitation : la page `/tech`

Chaque salle Maestro a une page technique, décrite elle aussi dans le gameflow. Celle du lobby regroupe :

- des **voyants** (lecteur audio, broker des panneaux) ;
- le statut, le redémarrage et les logs du lecteur audio, d'io-status-agent et de blinker-agent, par SSH ;
- un bouton **« Sons du lobby »** qui vérifie les 18 fichiers attendus et affiche `OK` ou `MANQUANT`. Quatre briefs attendaient encore leur fichier à l'installation : un bouton « Lancement » sans fichier fait baisser l'ambiance sans rien jouer, et ce voyant l'explique tout de suite ;
- un **« Restart room-orchestrator »** protégé : il interroge d'abord l'état de la salle et refuse de redémarrer pendant un changement d'état.

Côté consignes, nous avons préféré **un seul bouton de secours** à une liste de procédures : « Tout arrêter et relancer l'ambiance ». Il coupe tout, lève les verrous, remet la façade au repos et relance l'ambiance. Il couvre presque tous les cas dégradés : orchestrateur redémarré à la fin d'un brief, lecteur audio relancé, verrou resté posé.

Un dernier point d'exploitation, qui vaut pour toutes nos salles : un panneau d'entrées-sorties qui (re)démarre publie son statut et **ignore toute commande** tant qu'il n'a pas de réponse pour chacun de ses modules. Sans `io-status-agent`, les sorties du lobby restent inertes, sans aucune erreur visible.

---

## Ce qui vient

### En test à Lille : playlist, publicités et TV marketing

Le lobby de Lille sert aussi de banc d'essai. Trois sujets y sont en test, pilotés depuis Maestro comme le reste :

- **la gestion de la playlist d'ambiance** ;
- **la gestion des publicités** ;
- **les TV marketing** du lobby.

Nous y reviendrons dans un prochain billet, une fois ces briques validées en conditions réelles.

### Les annonces micro

Celle-ci existe déjà : la GM UI sait faire du *push-to-talk* vers le lecteur audio d'une salle. Brancher un micro sur le poste de l'accueil, c'est pouvoir appeler une équipe dans le lobby, sans aucun développement. Reste à le proposer aux GM.

### Paris 9, La Défense et Aéroville

Trois autres lobbys sont en cours d'installation. Le motif de Lille (briefs, verrous, séquences lumineuses, lancement complet) leur sert de modèle, mais chacun a son électronique. Paris 9 donne le ton : le relevé de son ancien contrôleur compte 17 entrées (dont 8 virtuelles) et 56 sorties, contre 18 entrées-sorties à Lille. On y lit des **interrupteurs de lancement** par salle, des **tubes pneumatiques** avec des messages par univers, des spots de *check-in*, les torches et le sablier de Fort Boyard, une machine à fumée, et des modes « sans client » et « cycle classique ». Les tubes pneumatiques mériteront un billet à eux seuls.

---

## Ce que nous en retenons

- **Une plateforme générique tient ses promesses quand on la pousse hors de son cas d'usage.** Une salle sans partie, avec des boutons permanents et un jeu de sonnette, n'a demandé qu'un seul changement de code, étroit et testé.
- **Des jalons cachés font d'excellents verrous.** C'est un usage détourné, mais il est lisible, visible en base et réinitialisable par un bouton.
- **Choisir la robustesse avant l'élégance.** Un fondu explicite plutôt que `notify`, un fichier unique plutôt qu'un enchaînement de pistes : à chaque fois, la solution qui ne dépend pas de l'orchestrateur à un moment critique.
- **Tracer chaque décision.** Le cadrage numérote seize décisions, D1 à D16 : arbitrées, proposées ou remplacées. Quand une question revient (« pourquoi pas `notify` ? »), la réponse est écrite, avec le piège qui l'a motivée.
- **Deux documents, deux publics.** Un README technique pour nous (clés, fichiers, installation) et un `scenarios.md` pour l'exploitation, qui décrit ce qui se passe événement par événement, sans une ligne de JSON. C'est le second que les GM lisent.

Le lobby de Lille est en service sur Maestro, et Paris 9, La Défense et Aéroville suivent. Le matin, un GM clique sur **Start**. Le soir, sur **Stop**. Entre les deux, les clients sonnent à l'hôtel hanté, et ça nous fait toujours sourire.
