---
title: "Maestro : orchestrer une salle d'escape game, un événement à la fois"
date: 2026-11-12
lang: fr
tags: [maestro, orchestration, mqtt, dotnet, event-driven, escape-game, architecture]
---

> **Titres alternatifs** (à choisir, puis supprimer ce bloc)
>
> 1. Le chef d'orchestre ne joue d'aucun instrument : dans le moteur de Maestro
> 2. Notifications, jalons et délais : comment Maestro fait jouer une salle d'escape game
> 3. Écrire la partition d'une salle : gameflow, concurrence et temps dans Maestro

# Maestro : orchestrer une salle d'escape game, un événement à la fois

Une salle d'escape game ressemble beaucoup à une partition. Pendant 60 minutes, des centaines d'événements se produisent : un joueur pose une statuette sur un socle, tape un code, tourne une manivelle ; une vidéo se termine ; le maître du jeu (GM) clique sur un bouton ; le temps s'écoule. Chacun déclenche des réactions : une porte qui se déverrouille, une lumière qui change de couleur, une musique qui démarre, de la fumée, une vidéo sur cinq écrans à la fois. Et tout cela doit arriver au bon moment, dans le bon ordre, et seulement si les conditions sont réunies.

**Maestro** est le moteur que j'ai construit pour orchestrer nos salles chez Team Break. Le nom n'est pas un hasard. Dans un [article précédent](./team-break-architecture.fr.md), j'ai présenté l'architecture d'ensemble (MQTT, agents, Raspberry Pi, ESP32, Linux). Ici, je zoome sur le cœur du système : comment Maestro transforme un flot d'événements hétérogènes en une expérience cohérente.

## Le chef d'orchestre ne joue d'aucun instrument

C'est le principe fondateur. L'orchestrateur de salle (*Room Orchestrator*) ne pilote jamais directement un relais, un lecteur vidéo ou un projecteur. Il reçoit des **notifications**, décide, puis publie des **commandes** à destination des agents qui, eux, savent parler au matériel. Le lecteur vidéo ne sait pas qu'il montre un dinosaure ; l'orchestrateur ne sait pas comment on décode du H.264.

Quatre sources d'événements convergent vers la même boîte aux lettres MQTT, `/games/<jeu>/rooms/<salle>/roomorchestrator` :

```
  Joueurs ──► capteurs (ESP32, Kincony, panneaux d'E/S)
                 │  /c5m1p7/on, {"module":"in2","pin":0,...}
                 ▼
            Electronic Agent ──────────────┐
                                           │
  Agents ──► « vidéo terminée », « intro audio finie » ──┤
                                           │   notifications
  GM ─────► console (jalons, contrôles, temps) ──────────┤   (JSON, MQTT)
                                           ▼
  Temps ──► délais, actions planifiées ──► Room Orchestrator ──► commandes
                                           │                     ├─► audioplayer
                                           │                     ├─► videoplayer
                                           ▼                     ├─► electronics
                                     Agents API (HTTP)           ├─► dmx
                                     PostgreSQL : état,          └─► customagents
                                     jalons, sessions
```

La source de vérité pour ce qui doit survivre à un redémarrage (état de la salle, jalons, session, temps restant) est une base PostgreSQL derrière une API HTTP. Le reste (événements, commandes) circule en MQTT, en *fire-and-forget*.

## La partition : le gameflow

Ce que fait une salle est décrit dans un **gameflow** : du JSON déclaratif, versionné dans un dépôt Git par jeu et par site. Son vocabulaire est réduit :

- une **notification** est un événement nommé (`digicode_unlocked`, `audio_intro_ended`) ;
- un **handler** réagit à une ou plusieurs notifications, éventuellement filtré par des **jalons** et retardé d'un **délai** ;
- un handler exécute des **comportements**, dans l'ordre de leur `position` ;
- un comportement publie une **commande** (ou un *bundle* de commandes), valide ou réinitialise un jalon, termine la salle, modifie le temps restant ou envoie une commande à une autre salle.

Il existe exactement neuf comportements : publier une commande, publier un bundle, valider un jalon, réinitialiser un jalon, déclarer un événement de salle (`room_succeed`, `room_failed`), envoyer une commande à une autre salle (*dispatch*), fixer le temps restant, figer l'affichage du temps et le libérer. C'est volontairement court : tout ce qui ne rentre pas dans ces cases relève d'un agent sur mesure, pas du moteur.

Voici un extrait réel de la Chocolaterie, à Lille. Quand l'introduction audio se termine (ou quand le GM force le jalon correspondant), trois handlers réagissent à la même notification, à t+0, t+1 s et t+6 s :

```json
[
  { "notifications": ["audio_intro_ended", "milestone_state_change_intro_played_validate"],
    "position": 1,
    "behaviors": [
      { "behavior": "milestone_validate", "key": "intro_played",         "position": 1 },
      { "behavior": "command_publish",    "key": "audio_background_play", "position": 2 } ] },
  { "notifications": ["audio_intro_ended", "milestone_state_change_intro_played_validate"],
    "position": 2, "delay": 1,
    "behaviors": [
      { "behavior": "command_publish",  "key": "r1_pinch_anims_scene1_play",       "position": 1 },
      { "behavior": "command_publish",  "key": "r1_pinch_scene1_audio_play",       "position": 2 },
      { "behavior": "command_publish",  "key": "r1_lights_stainedglass_pinch_on",  "position": 3 },
      { "behavior": "command_dispatch", "key": "chocolate_garden_init_audio_player", "position": 5 } ] },
  { "notifications": ["audio_intro_ended", "milestone_state_change_intro_played_validate"],
    "description": "delai: 5 + 1 (delay between intro and sceno 1 starts)",
    "position": 3, "delay": 6,
    "behaviors": [ { "behavior": "command_publish", "key": "r1_lights_stainedglass_pinch_off" } ] }
]
```

C'est ainsi qu'on écrit une séquence dans le temps : un handler par instant, sur la même notification. Le dernier comportement du deuxième handler prépare déjà le lecteur audio de la salle suivante.

À l'échelle de la Chocolaterie (cinq salles logicielles), cela représente environ 300 publications de commandes, 110 validations de jalons, une quarantaine de handlers différés et 136 handlers conditionnés par des jalons.

## Le moteur de l'orchestrateur de salle

### Démarrer, mais seulement quand on est prêt

Au démarrage, l'orchestrateur charge les réglages de la salle, son état en base et le gameflow, en réessayant toutes les 5 s jusqu'à ce que tout réussisse. Ce n'est qu'ensuite qu'il s'abonne aux topics et se déclare sain auprès de Docker. Le commentaire dans le code explique pourquoi :

> Only here does the container turn healthy […]. Signalling before InitializeAsync() would mark the agent healthy while it is subscribed to no topic — the very "Running but inert" false positive.

Si la salle était en cours de partie au moment du redémarrage (coupure de courant, mise à jour), l'orchestrateur relit l'état en base et réarme le compte à rebours là où il en était.

Détail subtil côté MQTT : les gestionnaires de messages sont attachés *avant* l'appel à `ConnectAsync`. MQTTnet livre les messages dès le SUBACK, et les brancher après coup perdait tout ce qui arrivait entre les deux.

### Aiguiller

Chaque notification porte un `type_id` qui détermine son traitement :

```csharp
switch (notification.Type ?? NotificationTypeEnum.RoomEvent)
{
    case NotificationTypeEnum.RoomEvent:
        await HandleRoomEventAsync(notification, scenarioScope); break;
    case NotificationTypeEnum.GameEvent:
        await HandleGameEventAsync(notification, scenarioScope); break;
    case NotificationTypeEnum.GameOrchestratorInstruction:
        await HandleGameOrchestratorInstructionAsync(notification.Source, notification.Key ?? string.Empty, notification.Datas, scenarioScope); break;
    case NotificationTypeEnum.GameMasteringInstruction:
        await HandleGameMasteringInstructionAsync(notification.Source, notification.Key ?? string.Empty, notification.Datas, scenarioScope); break;
    case NotificationTypeEnum.PublishCommandInstruction:
        await PublishCommandAsync(notification.Key ?? string.Empty, scenarioScope); break;
}
```

Les événements de jeu (ceux qui viennent des joueurs et des agents) ne sont traités que si la salle est en partie (ou au tout début de sa préparation). Un capteur qui s'active pendant que le GM range la salle ne déclenche rien. Les instructions du GM et de l'orchestrateur de jeu, elles, passent toujours.

### Sélectionner les handlers

Pour une notification de jeu, le pipeline est le suivant :

1. **Filtre de scénario** sur la notification elle-même. Une salle peut jouer une variante (Halloween, version enfants) ; certains éléments n'existent que dans l'une d'elles.
2. **`only_once`** : certaines notifications ne doivent compter qu'une fois par session. La liste est remise à zéro à chaque préparation.
3. **Sélection des handlers** qui écoutent cette clé, puis application des filtres et des `override` de scénario (un handler « Halloween » masque son homologue standard).
4. **Pour chaque handler**, dans l'ordre : s'il a un délai, il est mis en file et on passe au suivant. Sinon, ses **filtres de jalons** sont évalués (`milestone_validated`, `milestone_not_validated`, combinés en `and` ou en `or`), puis ses comportements sont exécutés séquentiellement.

```csharp
foreach (var behavior in handler.Behaviors.OrderBy(x => x.Position))
{
    if (!await handlerOptionSelectorScope.ShouldExecuteAsync(notificationKey, behavior.ElementKey,
            behavior.OptionKey, behavior.SelectedItem, CancellationToken.None))
    {
        continue;
    }
    await HandleNotificationHandlerBehaviorAsync(behavior.Behavior, behavior.ElementKey, scenarioScope,
        behavior.Seconds, behavior.NeverAddTime, source);
}
```

Le sélecteur d'options permet de jouer un comportement seulement pour une valeur d'option de session (typiquement la langue). Les clés de commandes peuvent aussi être construites dynamiquement : `option~lang~fr@_start_video` devient `fr_start_video` ou `en_start_video` selon la langue de l'équipe.

Une anecdote rappelle qu'un format déclaratif reste du code. Pendant longtemps, `"filters_behavior": "or"` a été évalué comme un ET, parce qu'un constructeur affectait le paramètre au lieu de la propriété. Aucun gameflow ne plantait : les handlers concernés ne se déclenchaient que si *toutes* leurs conditions étaient réunies, ce qui ressemble beaucoup à une énigme mal réglée.

### La concurrence, le vrai sujet

L'orchestrateur n'a pas de file d'entrée : chaque message MQTT ouvre son propre flux asynchrone, et plusieurs notifications peuvent être traitées en parallèle. À l'intérieur d'une notification, tout est séquentiel. Ce choix garde la salle réactive (un handler lent ne bloque pas un capteur), mais il impose une discipline.

Premier garde-fou : les instructions de changement d'état (préparer, briefing, démarrer, arrêter) sont sérialisées par un sémaphore. Une cascade forcée (« prépare la salle alors qu'elle est en partie » se traduit par « arrête, puis prépare ») enchaîne plusieurs transitions et ne doit pas s'entrelacer avec une autre instruction.

Second garde-fou, plus subtil : **la portée de scénario**. Chaque traitement lit le scénario actif une seule fois et le garde jusqu'au bout. Le commentaire raconte le bug qui l'a motivé :

> Le risque visé n'est pas la latence mais la concurrence : […] la notification passait la garde en halloween, le game master basculait, et la commande publiée était refusée en standard. Le déclenchement était alors joué à moitié.

Un effet joué à moitié dans une salle, c'est une porte qui s'ouvre sans la musique qui l'annonce, ou l'inverse. Avec une lecture unique par traitement, un événement est joué entièrement dans un scénario ou entièrement dans l'autre.

## Le temps, ou plutôt les temps

### Les délais

Les handlers différés vivent en mémoire, dans des ensembles triés par heure de déclenchement, interrogés par un `PeriodicTimer` toutes les 350 ms. Les délais s'expriment en secondes entières : la précision est largement suffisante pour un effet de mise en scène, et le mécanisme reste trivial à raisonner.

Un handler peut aussi se reprogrammer lui-même (`"behavior": {"behavior": "while", "delay": 1, "limit": 10}`) pour répéter une action avec une borne.

La question intéressante est : **quand faut-il oublier les délais en attente ?**

- **À l'arrêt de la salle**, évidemment. Mais *avant* d'exécuter les handlers `room_stopped`, de sorte qu'un handler d'arrêt puisse lui-même programmer un effet différé.
- **À la préparation**, par précaution.
- **Sur le signal d'urgence (SSI)**, quel que soit l'état de la salle. Le commentaire est explicite : « armés avant l'alarme, ils relanceraient le jeu à leur échéance (sons, lumières, portes) pendant l'évacuation ».
- **Pas à la fin de partie.** C'est voulu : à la Chocolaterie, une victoire déclenche un bundle immédiat, puis un second bundle 45 s plus tard, quand l'équipe a eu le temps de savourer.

### Le compte à rebours, en trois horloges

Le temps restant existe à trois endroits, et chacun a son rôle :

1. **En base**, la vérité : `début + durée − maintenant + ajustements`. Quand le gameflow fixe le temps restant (« il ne vous reste que 5 minutes ! »), c'est un delta qui est stocké. Le commentaire donne l'exemple : *une salle de 60 minutes ramenée à 300 s après 5 minutes de jeu stocke −3000*. Les ajustements du GM sont plafonnés à ±320 s par clic.
2. **Dans l'orchestrateur**, un compteur qui avance par pas de 5 s et déclenche `room_time_over` à zéro. Les clics du GM s'y **accumulent** jusqu'au prochain pas, parce que plusieurs clics rapides tombent dans la même fenêtre et qu'une simple affectation n'aurait gardé que le dernier :

   ```csharp
   public int Tick(int intervalInSeconds, out int appliedGameMasterAdjustmentInSeconds)
   {
       lock (_lock)
       {
           appliedGameMasterAdjustmentInSeconds = _pendingGameMasterAdjustmentInSeconds;
           if (appliedGameMasterAdjustmentInSeconds != 0)
           {
               _remainingTimeInSeconds += appliedGameMasterAdjustmentInSeconds;
               _pendingGameMasterAdjustmentInSeconds = 0;
           }
           _remainingTimeInSeconds -= intervalInSeconds;
           if (_remainingTimeInSeconds < 0) _remainingTimeInSeconds = 0;
           return _remainingTimeInSeconds;
       }
   }
   ```

3. **Pour l'affichage en salle**, un publieur qui calcule les secondes restantes à partir d'une **échéance** UTC, resynchronisée à chaque pas, plutôt qu'en décrémentant un compteur. Il peut être figé (`remaining_time_display_hold`) le temps d'une scène, puis libéré.

   ```csharp
   if (heldSeconds.HasValue) return heldSeconds.Value;
   if (deadlineUtc is null) return null;
   double secondsLeft = (deadlineUtc.Value - nowUtc).TotalSeconds;
   return secondsLeft <= 0 ? 0 : (int)Math.Ceiling(secondsLeft);
   ```

Toutes les 30 s environ, l'orchestrateur publie aussi une mise à jour de progression vers l'extérieur (pour les tablettes d'Archipel, qui interpolent entre deux messages). Une cadence grossière suffit : c'est l'affichage qui fait le travail fin.

## Les jalons, la mémoire de la partie

Un **jalon** (*milestone*) est une étape nommée de la progression : « intro jouée », « digicode trouvé », « coffre ouvert ». Il est persisté en base avec son origine (la salle, le GM ou une action planifiée), et il sert à trois choses :

- **conditionner des handlers** (« ne joue cette vidéo que si le coffre est ouvert ») ;
- **conditionner les indices et les actions planifiées** ;
- **raconter la partie** après coup, dans la chronologie de la session.

Le GM peut forcer un jalon depuis sa console, de deux façons. En mode **silencieux**, seule la base change : utile pour corriger un état sans rien rejouer. En mode **bruyant**, l'orchestrateur reçoit en plus la notification `milestone_state_change_<jalon>_validate` (ou `_reset`). Les gameflows l'écoutent à côté de la notification « joueur », comme dans l'extrait plus haut : le GM peut ainsi débloquer une énigme récalcitrante *avec* tous ses effets (lumière, son, porte), exactement comme si les joueurs l'avaient résolue.

## Les actions planifiées et les indices

Un agent dédié, *Scheduled Actions*, écoute le même topic que l'orchestrateur et tient sa propre horloge (monotone, démarrée au lancement de la partie). À chaque pas (30 s sur nos sites), il :

1. relit les options de la session (« indices automatiques », « victoires automatiques ») ;
2. prend **un seul** instantané de l'état de la salle (jalons, état) ;
3. parcourt les actions dans l'ordre et exécute **au plus une action par pas**, jamais deux fois dans la même session.

Une action se déclenche sur un temps écoulé depuis le début, un délai après un jalon, un délai après une notification, ou un temps restant. Elle peut être conditionnée par des jalons et appartient à une catégorie (`game`, `hint`, `auto_win`) que le GM active ou non pour chaque session. Rien ne se passe pendant la première minute de jeu.

Le **bouton d'indice** des joueurs suit une autre logique. La notification `hint_system` choisit la première entrée du planning d'indices dont les conditions correspondent à la progression, puis avance dans le groupe d'indices : premier indice, puis le suivant, puis le dernier en boucle une fois le groupe épuisé. Chaque indice est journalisé, avec cette règle écrite en commentaire : « une exception de journalisation ne doit pas empêcher un indice de partir ».

## Le cycle de vie d'une salle

Une salle traverse onze états. Le modèle de base est une poignée de main **demandé → confirmé** : la console du GM *demande* un état (en base), l'orchestrateur exécute les handlers correspondants puis *confirme* via l'Agents API.

```
 GM UI           GM API / DB            Game Orch.         Room Orch.              Agents
   │ prepare ──►  PrepareAsked
   │              (reset jalons) ──────► Prepare ────────► prepare_room
   │                                                       purge des délais
   │                                                       handlers room_prepared ──► portes, lumières, sons
   │              InPreparation ◄──────────────────────── /prepare/confirm
   │ prepared ──► PreparedAsked ───────────────────────► handlers room_preparation_done
   │ briefing ──► BriefingAsked ───────────────────────► handlers room_briefing_started
   │ start ─────► session créée ─────► Start + session ─► start_room
   │                                                       handlers room_started ────► intro, chrono
   │                                                       Running, timer 5 s
   │                                                       SessionProgressionInit ──► (Archipel)
   │                  … partie : notifications ↔ commandes, jalons, indices …
   │                                                       room_succeed | room_failed | room_time_over
   │                                                       puis room_ended
   │              Ended ◄──────────────────────────────── /ended/confirm
   │                                ◄── RoomEnded ──────── (enchaînement éventuel)
   │ stop ──────► StopAsked ──────────► Stop ───────────► Stopped d'abord, puis
   │              AtRest                                   purge, handlers room_stopped
```

Deux nuances, parce qu'elles sont instructives :

- **Le pragmatisme l'emporte parfois sur la pureté.** Pour le démarrage, l'arrêt, le briefing et la fin de préparation déclenchés depuis la console, l'API confirme l'état en base de façon optimiste (« Resilience fallback: avoid being stuck in StartAsked when orchestrator confirmation is delayed/lost »). Une salle bloquée en « démarrage demandé » avec six joueurs à l'intérieur est pire qu'un état momentanément optimiste. La préparation et la fin de partie, elles, restent confirmées par l'orchestrateur.
- **L'arrêt est inconditionnel.** L'orchestrateur passe la salle à l'état arrêté *avant* d'exécuter quoi que ce soit (« Critical safety invariant: room must become Stopped even if a stop handler fails »), puis coupe le chrono, purge les délais, confirme, et exécute enfin les handlers d'arrêt en *best effort*. L'identifiant de session n'est effacé qu'après ces handlers, pour que le montage vidéo de fin de partie sache encore à quelle session il appartient.

Quelques règles métier sont gravées dans les gardes : par exemple, une salle dont le briefing a commencé ne peut plus être « préparée », parce que les joueurs sont dedans et que rejouer `room_prepared` défairait le briefing (lumières, serrures…).

## Orchestrer plusieurs salles : le Game Orchestrator

Certaines expériences s'étendent sur plusieurs salles. La Chocolaterie en enchaîne cinq : la confiserie, le jardin de chocolat, l'ascenseur, la salle de glace et la salle des inventions. Chaque salle a son orchestrateur ; au-dessus, l'**orchestrateur de jeu** joue le rôle de chef de pupitre.

Il est volontairement **sans état**. Il connaît l'ordre des salles, écoute `/games/<jeu>/gameorchestrator` et relaie les instructions vers l'orchestrateur de la salle concernée. Quand une salle se termine :

1. l'orchestrateur de salle publie `RoomEnded` ;
2. l'orchestrateur de jeu demande à l'API d'enchaîner. En une seule opération, elle arrête la session de la salle terminée, réinitialise les jalons de la suivante et la démarre **avec le même identifiant de session** : pour le reporting, l'équipe a joué *une* partie ;
3. il publie l'instruction de démarrage vers la salle suivante.

Les salles se parlent aussi pendant la partie, grâce aux **dispatches** : une salle demande à une autre de publier une commande, en passant par l'orchestrateur de jeu.

```json
{ "key": "smoke_on",     "command":  "r2_smokemachine_smoke_on", "recipient_room": "chocolate_garden" }
{ "key": "roller_start", "commands": ["r3_sas_chocolategarden_door_roller_open",
                                      "r3_sas_iceroom_door_roller_close"], "recipient_room": "elevator" }
```

La commande est exécutée par la salle destinataire, avec *ses* filtres de scénario et *ses* options.

Enfin, l'orchestrateur de jeu est la porte d'entrée des systèmes externes, comme Archipel, notre plateforme d'exploitation de site, qui lance les salles depuis les tablettes des joueurs. Ce sera le sujet d'un [prochain article](./archipel-orchestration.fr.md).

## L'électronique en entrée : de `/c5m1p7/on` à `digicode_unlocked`

Les capteurs ne parlent pas le langage du gameflow. Un panneau d'E/S publie `/c5m1p7/on` (contrôleur 5, module 1, port 7) ; une carte Kincony publie `{"module":"in2","pin":0,"value":"off"}`. L'**agent électronique** fait la traduction :

1. il maintient un client MQTT par broker électronique (un par fournisseur), avec sa propre reconnexion ;
2. pour chaque message, il identifie le fournisseur à partir du topic, ignore les battements de cœur, construit une clé de résolution et la compare aux **proxies d'entrée** déclarés dans le gameflow ;
3. si un proxy correspond, il publie une notification métier (`digicode_unlocked`) sur le topic de l'orchestrateur, ou de plusieurs salles si besoin.

Dans l'autre sens, une commande `trappe_unlock` publiée par l'orchestrateur est résolue en **proxies de sortie**, puis en message adapté à la carte.

Un incident a montré que la traduction devait aussi **filtrer**. En octobre, une impulsion parasite (`/c5m1p7/on` puis `/c5m1p7/off` dans la même seconde) a déclenché l'entrée d'urgence. Chaque proxy d'entrée peut désormais porter un `debounce` : un front montant ouvre une fenêtre (100 ms par défaut) ; une retombée dans la fenêtre l'annule et est elle-même avalée ; sans retombée, l'entrée est confirmée une seule fois en fin de fenêtre. Le mécanisme repose sur un `TimeProvider` injecté, sans `Task.Delay`, ce qui le rend testable à la milliseconde près.

## Le maître du jeu dans la boucle

La console du GM est une application web en Web Components natifs, sans framework : pilotage de la salle, chrono, jalons, contrôles, caméras, micro push-to-talk vers les haut-parleurs de la salle, page technique pour tester chaque entrée-sortie, page de tests, chronologie des sessions.

Pour les mises à jour en direct, j'ai choisi la simplicité. Un seul *poll* côté serveur par salle (toutes les 1,5 s, démarré au premier abonné et arrêté au dernier) lit la base et pousse des deltas en **SSE** (`milestones`, `state`, `timer`, plus un `heartbeat`). Le commentaire résume l'argument : ce mécanisme « capte toutes les sources de changement (GM, joueurs, actions planifiées, confirmations du Room Orchestrator) puisque tout transite par la base ». Pas besoin d'abonner le navigateur à MQTT ni de reconstituer l'état à partir d'événements.

Les contrôles du GM (« ouvrir la porte 2 », « relancer la vidéo ») sont des listes d'actions exécutées dans l'ordre. Toutes sont tentées, même si l'une échoue, et le résultat a trois états : succès, action partielle, action impossible. Pendant une partie, le GM a besoin de savoir *ce qui* a marché, pas seulement *si* ça a marché.

## La sécurité, hors partition

L'urgence ne passe pas par le gameflow. Un agent dédié, au chemin critique minimal (pas de HTTP, pas de base, reconnexion MQTT infinie, `restart: always`), relaie le SSI : ouverture des portes, lumière, coupure du son, et un verrou publié en *retained* que tout agent qui redémarre relit immédiatement. Côté orchestrateur, le signal d'urgence purge les délais en attente, quel que soit l'état de la salle. Et surtout, la sécurité vitale (les ventouses des portes) reste câblée côté SSI, hors logiciel. Maestro suit l'alarme ; il ne la remplace pas.

## Tracer et tester

Chaque partie laisse une chronologie : changements d'état, jalons validés ou réinitialisés (avec leur origine), commandes publiées, contrôles utilisés par le GM, clics sur la page technique. Le journal des actions du GM (micro, volumes, caméras) n'a volontairement pas de clé étrangère, « pour que ni un TRUNCATE … CASCADE… ni la purge… n'effacent le journal ». Des *reporters* poussent les sessions terminées vers notre siège (en dehors des heures d'ouverture) et vers l'outil de réservation.

Côté tests, l'orchestrateur de salle a sa propre suite (184 tests), au milieu des quelque 1 800 du dépôt. Chaque gameflow embarque aussi un **protocole de test** (`_tests/*.json`), jamais lu par le jeu, validé en CI et rejoué depuis la console : on y vérifie, une par une, les 312 entrées-sorties de la Chocolaterie.

## Ce que j'en retiens

1. **Le chef d'orchestre ne joue d'aucun instrument.** L'orchestrateur décide et publie ; les agents exécutent. C'est ce qui permet de remplacer un lecteur vidéo, une carte électronique ou un projecteur sans toucher au moteur.
2. **Déclaratif ne veut pas dire sans bug.** Un gameflow est du code : il mérite un format documenté, des tests et une CI.
3. **La concurrence est le vrai sujet.** Traiter les événements en parallèle garde la salle réactive, à condition de figer le contexte (scénario, options) le temps d'un traitement.
4. **Le temps est pluriel.** Vérité en base, compteur en mémoire, échéance pour l'affichage, interpolation côté client : chaque horloge a un rôle, et aucune ne doit prétendre faire le travail des autres.
5. **Savoir quand oublier.** La purge des délais (à l'arrêt, à la préparation, sur alarme, mais pas à la victoire) est une décision de mise en scène autant que technique.
6. **Confirmer, sans jamais rester bloqué.** La poignée de main demandé → confirmé est la règle, et l'optimisme une exception assumée quand des joueurs attendent.
7. **La sécurité court-circuite tout.** Elle a son propre agent, son propre chemin, et le dernier mot.

Une salle, c'est quelques centaines de notifications par partie, quelques centaines de commandes, une poignée de jalons et un chrono. Bien orchestrés, les joueurs ne voient rien de tout ça. Ils voient une porte qui s'ouvre au moment exact où la musique monte.
