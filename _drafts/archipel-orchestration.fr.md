---
title: "Archipel : orchestrer les orchestrateurs"
date: 2026-11-12
lang: fr
tags: [archipel, maestro, orchestration, mqtt, sse, dotnet, escape-game, architecture]
---

> **Titres alternatifs** (à choisir, puis supprimer ce bloc)
>
> 1. Demander, puis observer : comment Archipel pilote les instances Maestro d'un site
> 2. Le régisseur et le chef d'orchestre : Archipel au-dessus de Maestro
> 3. De la réservation au chrono : orchestrer un site d'escape games avec Archipel

# Archipel : orchestrer les orchestrateurs

Dans l'[article précédent](./maestro-orchestration.fr.md), j'ai décrit comment Maestro orchestre une salle d'escape game : des notifications entrent, des commandes sortent, des jalons s'accumulent, un chrono s'écoule. Mais une salle ne vit pas seule. À Aéroville, l'un de nos sites, une dizaine de salles tournent côte à côte : jusqu'à neuf instances Maestro, plus un Donut Quiz qui n'utilise pas Maestro du tout.

Il faut donc un niveau au-dessus, qui réponde à des questions qu'aucune salle ne peut trancher seule :

- quelle équipe joue où, à quelle heure, dans quelle langue, à combien ?
- quand peut-on ouvrir la tablette d'une salle à des joueurs ?
- quand faut-il envoyer l'ordre de démarrage, et à quelle instance ?
- quelle variante de la salle (le scénario) préparer pour la session suivante ?
- que doivent afficher les téléviseurs de l'accueil ?

C'est le rôle d'**Archipel**, notre plateforme d'exploitation de site. Si Maestro est le chef d'orchestre d'une salle, Archipel est le régisseur du théâtre : il ne joue pas, il ne dirige pas l'orchestre, mais rien ne commence sans lui.

## Deux niveaux d'orchestration, une frontière nette

Le point le plus important de la conception tient en un tableau : **qui fait autorité sur quoi**.

| Archipel fait autorité sur… | Maestro fait autorité sur… |
|---|---|
| Réservations, créneaux, sessions | L'état réel de la salle |
| L'équipe : nombre de joueurs, langue, nom | Le temps : début, durée, temps restant |
| Le scénario de la *prochaine* session | Le gameflow et son exécution |
| Les mots de lancement, l'accès aux tablettes | Le matériel, les agents, la sécurité |
| Ce qu'affichent les écrans d'accueil | Les jalons, les indices, la fin de partie |

De ce partage découlent trois règles que l'on retrouve partout dans le code :

1. **Archipel demande, Maestro confirme.** Archipel n'écrit jamais lui-même l'état d'une salle pilotée par Maestro : il envoie une demande, puis attend la confirmation.
2. **Pour Maestro, Archipel est un observateur optionnel.** Le broker MQTT local de la salle est critique ; celui d'Archipel ne l'est pas. Une salle doit pouvoir jouer, se préparer et s'arrêter si Archipel est en panne, en maintenance ou sur un autre sous-réseau. Côté Maestro, la connexion à Archipel ne bloque jamais le démarrage et la publication vers Archipel est en *best effort* : « a lost notification must not turn an already persisted state change into a 500 ».
3. **Les tablettes ne parlent qu'à Archipel, jamais au système de jeu.** Le commentaire dans les endpoints de check-in le dit explicitement : un futur système (Heroes Academy) « se branchera ici sans nouveau chemin côté UI ». Aujourd'hui déjà, Archipel pilote deux systèmes : Maestro via MQTT, et Donut Quiz via HTTP.

Côté technique, Archipel est une application .NET 10 (EF Core et PostgreSQL 16, MQTTnet 5) qui tourne sur un mini-PC par site, dans Docker Compose. Le code applicatif suit un découpage commande/requête avec un médiateur maison minimal, et le front est en JavaScript vanilla (modules ES, sans npm ni bundler). Plusieurs services d'arrière-plan cadencent le site :

| Service | Cadence | Rôle |
|---|---|---|
| Synchronisation des salles | 24 h | Ouvre une « porte » que les autres synchronisations attendent |
| Synchronisation des réservations | 5 min | Import des réservations de l'outil de réservation en ligne |
| Synchronisation des créneaux | 1 h, sur 14 jours | Import des créneaux |
| Passage en « attente des joueurs » | 30 s | Ouvre les salles prêtes dont le créneau approche, publie les scénarios |
| Annulation des retards | 30 s | Annule les équipes qui ne se présentent pas |
| Surveillance des prochaines missions | 10 s | Alimente les téléviseurs de l'accueil |
| Relais MQTT | événementiel | Traduit les notifications de Maestro en état Archipel |

## La vie d'une session, de la réservation à la sortie de salle

Voici le parcours complet d'une équipe, vu par le régisseur :

```
 Outil de        Borne        Archipel           Tablette        Maestro                 Maestro
 réservation     d'accueil    (site)             de salle        Game Orchestrator       Room Orchestrator
     │ réservations (5 min)      │
     ├──────────────────────────►│ session, créneau
     │             │ identify ──►│
     │             │ confirm ───►│ langue, joueurs,
     │             │◄── mot ─────│ mot de lancement
     │                           │ ◄─────────── RoomStateChanged(Prepared) ──────────────────┐
     │                           │ Ready → WaitingForPlayers (créneau proche)                │
     │                           │◄── start {mot} ──┤                                        │
     │                           │ onboarding ─────►│ (étapes de récit)                      │
     │                           │── ChangeRoomState(StartAsked) + équipe ──►│               │
     │                           │                  │                        │ session Maestro
     │                           │                  │                        ├─ start_room ─►│
     │                           │◄──────── RoomStateChanged(Started) ───────┤               │
     │                           │◄──────── SessionProgressionInit/Update ───┤◄──────────────┤
     │                           │── SSE (1 s) ────►│ compte à rebours interpolé             │
     │                           │◄──────── RoomStateChanged(Ended) ─────────┤◄──────────────┤
     │                           │ scénario de la session suivante ─────────►│
```

Étape par étape :

1. **Import.** Toutes les 5 minutes, Archipel récupère les réservations et les rattache à un créneau et à une salle. Un produit « scénario » (par exemple *Dark Magic School*, la version Halloween de notre école de magie) est résolu vers la salle physique *et* sa variante.
2. **Check-in à la borne.** L'équipe s'identifie (e-mail, numéro de commande, nom, téléphone…), confirme sa session, choisit la langue et le nombre de joueurs. Archipel lui attribue un **mot de lancement** : le premier mot de la liste de la salle, dans la langue choisie, qu'aucune session voisine ne détient, en partant d'un indice dérivé de l'identifiant de session. Si l'équipe revient à la borne, elle retrouve le même mot. Si elle déclare plus de joueurs que réservé, elle est envoyée à l'accueil et la session est bloquée 5 minutes, pour qu'elle ne puisse pas simplement recommencer avec le bon nombre.
3. **La salle devient jouable.** Quand le GM a fini de préparer la salle, Maestro confirme l'état *Prepared*, qu'Archipel traduit en *Ready*. Toutes les 30 s, un balayage fait passer les salles prêtes dont le créneau approche en « attente des joueurs ».
4. **Connexion à la tablette.** La tablette de la salle affiche une grille de 8 mots : les mots de lancement des sessions qui peuvent démarrer maintenant, complétés par des leurres, et jamais le mot d'une équipe déjà lancée. La grille est construite de façon déterministe à partir de l'identifiant de session, pour rester stable alors que la tablette reçoit un état chaque seconde (la déduplication côté client en dépend). L'équipe a deux essais ; ensuite, la tablette se bloque 10 s. Le commentaire précise le choix : « Short cooldown, not a hard lock: a team that picked the wrong tablet must not leave it stuck ».
5. **Récit puis lancement.** Si la salle a des étapes d'introduction, la tablette les déroule d'abord, et l'ordre de démarrage ne part qu'à la dernière, « so the mission clock does not run while players read the instructions ».
6. **Démarrage côté Maestro.** L'orchestrateur de jeu reçoit l'ordre, crée la session Maestro avec le nom de l'équipe, le nombre de joueurs, la langue et le scénario, puis transmet l'instruction de démarrage à l'orchestrateur de salle, en indiquant qu'elle vient d'Archipel.
7. **Retour d'état.** Maestro publie le changement d'état puis la progression (initiale, puis toutes les 30 s environ). Archipel les relaie à la tablette, qui affiche un compte à rebours.
8. **Fin de partie.** À la fin, Archipel enregistre la fin, efface la progression, et envoie *immédiatement* à Maestro le scénario de la session suivante. Le GM prépare la salle, et le cycle recommence.

Pendant ce temps, les téléviseurs de l'accueil affichent les prochaines missions ; une ligne disparaît dès que la salle passe en jeu.

## Le contrat entre les deux mondes

Archipel et Maestro se parlent via deux topics MQTT :

- `/archipel/games/<salle>/commands` : instructions d'Archipel vers Maestro ;
- `/archipel/notifs` : notifications de Maestro vers Archipel.

Le contrat est minimal. Deux types d'instructions : **changer l'état d'une salle** (`ChangeRoomState`) et **jouer un élément du gameflow** (`PublishRoomGameflowCommand` : une commande, un bundle, une notification ou une option). Trois types de notifications : **état changé**, **progression initiale**, **progression mise à jour**. Voici un ordre de démarrage :

```json
{
  "kind": 2,
  "room": "<clé Maestro de la salle>",
  "command": null,
  "data": [
    { "key": "language",      "value": "fr" },
    { "key": "players_count", "value": "4" },
    { "key": "team_name",     "value": "Les Explorateurs" },
    { "key": "scenario",      "value": "halloween" }
  ],
  "session_id": 123,
  "room_state": 5
}
```

`kind: 2` signifie *ChangeRoomState*, `room_state: 5` signifie *StartAsked*. Ce sont des entiers, comme partout chez nous, et donc soumis à la même règle : on ajoute, on ne renumérote jamais.

### Deux clés pour une salle

Une même salle a deux identités : son code côté Archipel (par exemple `jr`), qui sert de topic, et sa clé côté Maestro (`jurassikroom`), qui voyage dans le message. Les confondre a coûté cher. Le relais résolvait la salle à partir du code Archipel seul, et le commentaire raconte la suite : utiliser cette clé « silently dropped every state confirmation for such rooms ». Pas d'erreur, pas d'exception : une salle dont les deux clés diffèrent ne remontait simplement jamais son état. Le relais lit désormais la clé Maestro dans le message, et ne retombe sur le code Archipel que pour les déploiements à clé unique.

### Un format figé par des tests

Les deux côtés n'évoluent pas au même rythme : une salle Maestro peut tourner longtemps sur une vieille version de paquet. Le format est donc figé par une suite de tests (`WireFormatTests`) : noms et ordre des champs, énumérations sérialisées en nombres, clés de données, nombres indépendants de la culture, tolérance aux champs inconnus. L'en-tête du fichier explique l'enjeu mieux que moi :

> a changed field name or type (an enum serialized as text, a renamed data key) makes one side silently ignore the other's messages: the room no longer starts from the tablet, or the tablet no longer sees the game.

Quand nous avons ajouté les scénarios, nous n'avons pas créé de nouveau type de message. Le scénario passe par l'instruction existante « jouer un élément du gameflow », avec le type `gameflow_option` : pas de nouveau contrat, donc pas de nouvelle version de la bibliothèque partagée à déployer en même temps des deux côtés.

## Demander, puis observer

Archipel a sa propre machine à états pour chaque salle (prête, attente des joueurs, onboarding, en jeu, terminée, arrêtée, indisponible, mode GM…), mais quand la salle est pilotée par Maestro, cet état est **dérivé** des confirmations de Maestro :

| État Maestro | État Archipel |
|---|---|
| Préparation demandée, en préparation | En préparation |
| Préparée | Prête |
| Démarrage demandé, démarrée | En jeu |
| Terminée | Terminée |
| Arrêt demandé, au repos | Arrêtée |
| Fin de préparation *demandée* | ignoré |

La dernière ligne est la plus intéressante. « Fin de préparation demandée » n'est qu'une demande, et le commentaire le rappelle : « only the request: Maestro can still refuse it ». Archipel attend « Préparée » pour ouvrir la salle aux joueurs.

Même logique pour le panneau GM caché dans la tablette : quand le GM y demande de préparer ou d'arrêter la salle, Archipel envoie l'ordre mais n'écrit pas l'état. Selon le code, « the real state is driven by the RoomStateChanged confirmations Maestro sends back », pour que le panneau reflète toujours l'état réel de Maestro.

Dernier garde-fou : une salle pilotée par Maestro qui n'a encore jamais remonté d'état est considérée comme **arrêtée**, pas comme prête. Sans cette règle, une tablette pourrait s'ouvrir à des joueurs dans une salle que personne n'a préparée.

## Lancer une salle : l'ordre qui doit partir, et celui qu'on défait

Le démarrage est le seul moment où Archipel déclenche quelque chose d'irréversible : la mission commence, le chrono tourne, la musique part. Trois détails d'implémentation montrent ce que cela implique.

**L'ordre part, même si la tablette abandonne.** Une tablette peut recharger sa page, perdre le Wi-Fi ou subir le timeout d'un proxy au milieu de la requête. L'envoi utilise donc `CancellationToken.None` :

```csharp
// CancellationToken.None : la tablette a pu abandonner la requete (rechargement), l'ordre
// doit partir quand meme -- sinon la salle passe en « demarrage demande » sans que rien ne
// demarre.
ArchipelInstructionMessage startInstruction = ArchipelInstructionMessage.BuildStartRoomInstruction(maestroRoomKey, sessionId)
    with { Data = await BuildMaestroLaunchDataAsync(sessionId, sessionRepository, logger) };

bool published = await brokerClient.PublishInstructionAsync(startInstruction, CancellationToken.None,
    topicRoomKey: key);
```

**QoS 1, parce que QoS 0 mentait.** Le commentaire explique pourquoi l'envoi est passé en *at least once* : « At most once (the builder's default) only meant "written to the socket", so a half-open connection lost the instruction while the API still answered OK. » Avec QoS 1, le broker accuse réception (PUBACK).

**Si l'ordre n'est pas parti, on défait le lancement.** La salle revient en attente des joueurs, les horodatages de lancement sont effacés, et la tablette reçoit une erreur `launch_failed` plutôt qu'un « mission démarrée » mensonger. L'équipe peut réessayer, ou appeler le GM.

Il faut être honnête sur la limite : le PUBACK vient du broker, pas de Maestro. Si l'instance Maestro de la salle est hors ligne, l'ordre est accepté et la salle reste en « démarrage demandé ». La procédure documentée est alors un repli manuel depuis la console GM de Maestro. C'est une limite connue : un accusé de réception de bout en bout demanderait un nouveau contrat et une attente supplémentaire côté tablette.

## Le temps appartient à Maestro

Le compte à rebours sur la tablette illustre bien le partage d'autorité. Maestro fait autorité sur le temps : il envoie une progression initiale (durée totale, heure de départ) puis une mise à jour toutes les 30 s environ. Archipel l'enrichit avec ce que Maestro ne sait pas : le nombre de joueurs et la langue, issus de la session qui a lancé la salle.

Entre deux messages, trois mécanismes évitent que l'affichage ne mente :

- **Le serveur vieillit la donnée.** La progression stockée date du *dernier* message de Maestro ; Archipel la ramène à l'instant présent avant de l'envoyer :

  ```csharp
  // The snapshot holds the timing of Maestro's LAST message: age it to now. Sent as is, a
  // tablet re-anchored on it kept showing the same value until the next Maestro update, and
  // the countdown stopped altogether if Maestro went quiet.
  int age = (int)Math.Max(0, Math.Floor((now - snapshot.UpdatedAt).TotalSeconds));
  int remainingSeconds = Math.Max(0, snapshot.RemainingSeconds - age);
  ```

- **La tablette interpole.** Elle construit une ancre (temps restant à un instant donné de l'horloge murale) et décompte localement, minute par minute puis seconde par seconde à la fin, même si plus rien n'arrive.
- **Une progression trop vieille expire.** Au-delà de 10 minutes sans nouvelles, elle est oubliée : c'est le filet de sécurité contre une barre de progression figée.

## Les scénarios : préparer la session suivante

Un **scénario** est une variante d'une salle physique : même salle, même tablette, même instance Maestro, mais un autre produit à la réservation et une autre option dans le gameflow. Notre école de magie devient *Dark Magic School* pour Halloween ; Prison Break devient *Prison Break Asylum*.

Pourquoi ne pas déclarer une deuxième salle dans Archipel ? Parce qu'une instance Maestro n'écoute qu'une clé, et qu'il n'y a qu'un état de tablette par salle physique. Le scénario est donc un attribut de la session, et Archipel orchestre son application avec une règle simple : **hors partie, le scénario sélectionné dans Maestro est celui de la prochaine session.**

Il est envoyé :

- **immédiatement** quand une partie se termine ou que la salle est arrêtée, même s'il n'a pas changé (le GM a pu le modifier pendant la partie) ;
- **par le balayage de 30 s**, seulement si la valeur a changé (une session réservée entre-temps, par exemple) ;
- **juste avant** une préparation demandée depuis la tablette ;
- **dans l'ordre de démarrage**, comme filet de sécurité ;
- **jamais pendant une partie**, ni pendant une préparation (« l'ambiance changerait en cours de route ») : le GM garde la main.

Côté Maestro, cette option n'est jamais relayée à la salle en cours : elle est appliquée à la configuration du gameflow, que la prochaine session reprendra à son démarrage.

## Diffuser : de MQTT aux écrans

Une notification de Maestro atteint l'écran concerné en quelques secondes au plus (moins de deux pour une tablette de salle), par deux chemins différents.

**Vers la tablette de salle.** Le relais MQTT met à jour l'état en base (et la progression en mémoire). Une boucle SSE par tablette lit chaque seconde un instantané de la salle, partagé entre tous les lecteurs (une seule requête en base par salle), et ne l'envoie que s'il a changé. La définition de « changé » a son histoire :

> Comparing the raw payload never deduplicated anything (serverTime alone changes every second): each tablet got the full payload every second, and since it re-anchors its countdown on each payload, the countdown froze between two Maestro updates.

La signature exclut désormais les champs qui bougent chaque seconde. Côté tablette, un garde rejette tout message dont le `serverTime` est plus ancien que le dernier appliqué, pour qu'un message en retard ne fasse jamais revenir l'écran en arrière.

**Vers les téléviseurs de l'accueil.** Les tableaux de bord (prochaines missions, disponibilités) reçoivent des **événements** plutôt que des instantanés :

- chaque événement porte un identifiant `epoch-séquence`, où l'*epoch* est tiré au hasard à chaque démarrage du serveur ;
- les 512 derniers événements sont gardés dans un tampon circulaire ;
- à la reconnexion, le navigateur renvoie `Last-Event-ID` et le serveur rejoue ce qui manque ;
- si l'écart est trop grand, ou si l'epoch a changé (le serveur a redémarré), le serveur envoie un événement `resync`, et le client recharge tout.

Côté client, un gestionnaire de synchronisation commun à toutes les vues gère le repli en polling, la reconnexion avec backoff, et sérialise tous les rechargements complets pour que deux chargements ne reconstruisent jamais une vue en même temps. Un `resync` déclenche un rechargement différé de 2 s, avec 0 à 2 s d'aléa supplémentaire contre l'effet de troupeau quand dix écrans reçoivent le même signal. Les téléviseurs, qui tournent des jours entiers, font aussi un rechargement complet toutes les 5 minutes, et toutes les vues se rechargent d'elles-mêmes après une mise à jour de l'interface.

Enfin, côté serveur, les événements de disponibilité sont publiés par un intercepteur EF Core, *après* la validation de la transaction, quel que soit le chemin d'écriture : aucun endpoint ne peut oublier de prévenir les écrans.

## Intégrer le monde extérieur sans s'y faire bloquer

Un orchestrateur de site dépend de services qu'il ne maîtrise pas : l'outil de réservation en ligne, le broker MQTT, les instances Maestro. La règle est qu'aucun d'eux ne doit pouvoir bloquer le reste.

**Une réservation invalide ne bloque jamais les suivantes.** Début novembre, une réservation sans identifiant de créneau a fait échouer toute la passe de synchronisation. Comme l'outil renvoie la même liste à chaque passe, toutes les réservations listées après elle sont restées figées : plus de nouvelles réservations, plus d'annulations, et des équipes introuvables à la borne. La correction isole chaque réservation :

```csharp
foreach (var incoming in reservations)
{
    try { await UpsertReservationAsync(incoming, ct); }
    catch (Exception ex) when (ex is not OperationCanceledException || !ct.IsCancellationRequested)
    {
        _logger.LogError(ex, "… reservation {ReservationId} could not be synced, skipped until the next sync",
            incoming.ReservationId);
        _db.ChangeTracker.Clear();   // sinon chaque SaveChanges suivant les retenterait
    }
}
```

**Le broker peut être absent.** L'API démarre même sans broker, car le check-in, les tableaux de bord et l'administration n'en ont pas besoin. Une seule boucle gère la reconnexion, avec un backoff de 2 à 60 s et un réabonnement à chaque reconnexion. « Une seule » est important : une ancienne version se reconnectait aussi depuis le gestionnaire de déconnexion, ce qui « raced with it […] and turned a broker outage into a log line every 2 seconds ».

**Les données du site sont versionnées.** Les textes, les salles, les commandes GM, les scénarios d'un site vivent dans un dépôt Git dédié, sous forme de patchs numérotés. Archipel retient la position du dernier patch appliqué dans un volume Docker dédié. Avant ce volume, la position vivait dans le conteneur, et chaque mise à jour remettait tous les patchs « en attente ». Un patch qui échoue est quand même marqué appliqué, et on le corrige par un nouveau patch : l'historique ne se réécrit pas.

## Déploiement

Archipel tient sur un mini-PC par site, sous Debian 12 avec Docker : PostgreSQL, Mosquitto, un migrateur qui doit avoir terminé avant que l'API ne démarre, l'API, l'interface et une passerelle nginx. Les images sont publiées par la CI à chaque fusion, avec un tag `vAAAA.MM.JJ-<sha>`. La production épingle un tag ; revenir en arrière, c'est remettre le précédent (en restaurant la base si une migration est passée). Une sauvegarde part chaque nuit.

Les tablettes et les écrans d'accueil sont des mini-PC sous Debian 12 avec Ubuntu Frame, Chromium et le clavier à l'écran en snaps, ce qui empêche de quitter l'application. Les mises à jour des snaps sont cantonnées entre 3 h et 5 h du matin.

## Ce que j'en retiens

1. **Chaque niveau a son autorité, et elle est écrite.** Archipel possède la session ; Maestro possède la salle et le temps. Tant que ce tableau est clair, les conflits se règlent d'eux-mêmes.
2. **Demander, observer, ne jamais présumer.** Un orchestrateur de niveau supérieur dérive l'état des confirmations qu'il reçoit ; il ne l'écrit pas à la place de celui qui exécute.
3. **Celui qui est orchestré doit pouvoir s'en passer.** Une salle Maestro joue sans Archipel ; Archipel accueille des équipes sans broker. Chaque dépendance est optionnelle pour celui qui la subit.
4. **Un ordre irréversible part quoi qu'il arrive, ou se défait proprement.** Pas de « mission démarrée » affiché alors que rien n'est parti.
5. **Le contrat est figé par des tests**, parce que les deux côtés ne se mettent jamais à jour en même temps.
6. **Une donnée invalide ne bloque jamais les suivantes.** C'est évident, et pourtant c'est la cause de notre dernier incident de synchronisation.
7. **Les écrans ne doivent jamais mentir.** Vieillir la donnée, interpoler, dédupliquer, rejeter ce qui arrive dans le désordre : l'équipe devant la tablette ne voit que le chrono, et le chrono doit être juste.

Maestro fait vivre une salle. Archipel fait vivre un site : il amène la bonne équipe devant la bonne porte, au bon moment, dans la bonne langue, avec la bonne variante de l'histoire. Ensuite, il s'efface et laisse Maestro jouer.
