---
title: "Donut Quiz : migrer un jeu de .NET 5 et Avalonia vers le web, salle ouverte"
date: 2026-11-12
lang: fr
tags: [donutquiz, dotnet, avalonia, sse, mqtt, raspberry-pi, dmx, gpio, debian, migration]
---

> **Titres alternatifs** (à choisir, puis supprimer ce bloc)
>
> 1. Six pupitres, un buzzer, aucun animateur : dans les coulisses de Donut Quiz
> 2. D'Avalonia à EventSource : la migration de Donut Quiz vers le web
> 3. Un quiz, 9 sites, 81 machines : architecture et leçons de Donut Quiz

# Donut Quiz : migrer un jeu de .NET 5 et Avalonia vers le web, salle ouverte

Dans les articles précédents, j'ai beaucoup parlé d'escape games. Mais le produit qui occupe le plus de machines de notre parc n'en est pas un : c'est **Donut Quiz**, un quiz en équipes installé dans nos sites, qui représente à lui seul près de la moitié des machines que [nous exploitons](./vigie-exploitation.fr.md).

Cet article présente son architecture, ses particularités matérielles (buzzers, DMX, son), et surtout la migration que nous avons menée en 2026 : sortir d'une pile .NET 5 et Avalonia sur Raspbian 32 bits pour passer à .NET 10, des vues web en JavaScript vanilla poussées en SSE, et Debian 13 en arm64. Le tout sans fermer les salles.

## Le jeu

Une partie oppose de 2 à 6 équipes. Chacune s'installe à un **pupitre** de couleur, nommé d'après un parfum de donut : Chocolat, Citron, Myrtille, Pistache, Fraise, Caramel. Un pupitre, c'est :

- un **écran tactile** pour les questions et les réponses ;
- un **écran de score**, tourné vers la salle ;
- un **buzzer** physique ;
- une **barre de LED** RGB.

Un grand écran commun, le *saloon*, montre le déroulé à tout le monde, et des projecteurs DMX éclairent chaque équipe. Il n'y a **pas d'animateur** : la voix off, les diapositives et les vidéos mènent la partie. Le maître du jeu se contente de créer, lancer, mettre en pause ou arrêter une partie.

Les manches portent des noms gourmands : *Pour du beurre* (une question d'échauffement), *Mise en bouche* et *Surprise du chef* (15 questions chacune), deux mini-jeux de 30 secondes (*Apéro* et *Milkshake de frites*), puis *Cerise sur le gâteau* (10 questions de 45 s). Une partie normale compte 40 questions et dure environ une heure ; un mode court (15 questions, une vingtaine de minutes) et un mode express complètent l'offre.

Les équipes disposent de **coups spéciaux** : *Bouchées doubles* (points doublés), *On fait moit-moit* (la moitié des mauvaises réponses disparaît), *Pique-assiette* (voler la moitié des points d'une équipe sur la question), *Retard en cuisine* (l'équipe ciblée perd 9 secondes d'affichage). En fin de partie, des distinctions sont décernées : le *Donut d'or* pour l'équipe gagnante, le *Steak saignant* pour l'équipe la plus volée.

Côté mécanique, un détail compte beaucoup : **on touche l'écran pour choisir, on appuie sur le buzzer pour valider**. Le temps de réponse est mesuré sur le pupitre, et les bonnes réponses sont classées par rapidité.

## Neuf Raspberry Pi par site

Donut Quiz tourne sur 9 sites, avec 9 machines par site, essentiellement des Raspberry Pi 4 (8 Go) :

```
                       ┌──────────────────────────────┐
                       │ Serveur central (Pi 4)        │
                       │ admin · check-in · synchro    │
                       │ MongoDB · Mosquitto           │
                       └──────────────┬───────────────┘
                                      │ MQTT  /games/dq/…
          ┌───────────────────────────┼─────────────────────────────┐
          │                           │                             │
 « donut » (Pi 4)               6 pupitres (Pi 4)              saloon (Pi 4)
 moteur de jeu                  tactile + score                 grand écran
 son (MQTTaudioplayer)          buzzer GPIO                     (3 TV par splitter
 spots (DMX)                    LED (DMX)                        sur un site)
```

Tout communique en MQTT, sous un préfixe unique :

```csharp
private const string GAME_TOPIC = "/games/dq/";
public static string GetOrchestratorTopic() => string.Concat(GAME_TOPIC, "orchestrator");
public static string GetSpotlightsMasterTopic() => string.Concat(GAME_TOPIC, "lights");
public static string GetSoundMasterTopic() => string.Concat(GAME_TOPIC, "audio");
public static string GetDesksTopic() => string.Concat(GAME_TOPIC, "desks/all");
public static string GetDeskLedColorTopic(PupitreColors color) => string.Concat(GetDeskTopic(color), "/led");
```

Ce n'a pas toujours été le cas : la première version utilisait Akka.NET, dont il reste des traces commentées dans le code. Le passage à MQTT a aligné Donut Quiz sur le reste de nos produits, et c'est ce choix qui a rendu possible la migration décrite plus bas.

Le déroulement d'une partie suit un chemin simple :

1. le maître du jeu lance la partie depuis l'administration, qui publie une commande `StartGame` sur `/games/dq/orchestrator` ;
2. le moteur de jeu, sur le « donut », récupère la partie complète par HTTP ;
3. il précharge tous les sons de la partie dans le lecteur audio ;
4. un *storyboard* déroule les étapes, et chaque étape publie vers le saloon, les pupitres, le son, les spots et les LED ;
5. les pupitres répondent sur le topic de l'orchestrateur (inscription d'équipe, réponse, coup spécial…), et le serveur leur renvoie un accusé de réception ;
6. classements, événements et distinctions remontent vers l'administration.

## Le moteur : un storyboard écrit comme un itérateur

Le cœur du moteur est un choix élégant : le déroulé d'une partie est un **itérateur C#**. Chaque `yield return` produit une « histoire » (une étape, une boucle conditionnelle, une action technique), et un timer *one-shot* avance d'une étape à l'autre en fonction de la durée de chacune.

```csharp
yield return new WhileStory(GameRounds.Game,
    () => new TeamsSettingUpInProgressGameStep(TEAMS_SETTING_UP_CHECK_INTERVAL_GAME_STEP_DURATION, ...),
    () => !IsTeamsSettingUpFinalized());
yield return new IdleStory(GameRounds.Game, () => new TeamsSettingUpResultGameStep(...));
yield return new TechnicalStory(GameRounds.Game, () => this.ProvideTeamsToAskers());
...
foreach (var round in Rounds.Where(round => !round.GameRound.In(GameRounds.PourDuBeurre)))
{
    yield return new IdleStory(current.GameRound, () => { this.currentGameRound = current; return current.Start(TeamManager); });
    foreach (var story in round.Stories) yield return story;
    yield return new TechnicalStory(current.GameRound, () => { this.teamManager = current.End(); });
}
```

Le déroulé se lit de haut en bas comme un script de spectacle : attendre que les équipes s'inscrivent, annoncer les équipes, puis jouer chaque manche. Le timer qui fait avancer l'itérateur porte un commentaire que j'aime bien, parce qu'il assume une entorse aux bonnes pratiques : *« async void is intentional here: System.Timers.Timer.Elapsed requires a void return type. All code paths are wrapped in try-catch »*.

Les manches se répartissent en deux familles. Les **manches serveur** (les QCM) sont pilotées message par message par le moteur. Les **manches client** (les mini-jeux de 30 secondes) sont envoyées d'un bloc aux pupitres, qui les jouent seuls puis renvoient le résultat. Pour un enchaînement de 12 questions en 30 secondes, faire un aller-retour réseau par question n'aurait pas de sens.

## Le buzzer, seul organe d'entrée du jeu

Sur un pupitre, le buzzer est un bouton câblé sur une broche GPIO du Raspberry Pi. Dans la première version, l'application du pupitre le lisait directement :

```csharp
controller ??= new GpioController(PinNumberingScheme.Board);
controller.SetPinMode(BUTTON_GPIO, PinMode.InputPullDown);
controller.RegisterCallbackForPinValueChangedEvent(BUTTON_GPIO, PinEventTypes.Rising, OnPinValueChanged);
...
Monitor.TryEnter(padlock, TimeSpan.Zero, ref lockTaken);
if (lockTaken && DateTime.Now.Subtract(last).TotalMilliseconds > 500)
{ BuzzerBumped?.Invoke(this, EventArgs.Empty); last = DateTime.Now; }
```

Broche 40 (BCM 21), pull-down, front montant, et un anti-rebond de 500 ms sous un verrou non bloquant.

Dans la nouvelle version, le buzzer est devenu un petit service Python indépendant, qui publie un message sur `/games/dq/desks/<couleur>/buzz`. Son en-tête raconte l'une des journées les plus frustrantes de la migration. À Paris 9, le 19 août, quatre pupitres sur six ne réagissaient plus au buzzer. Le journal du service affichait pourtant chaque appui. Le premier défaut est décrit ainsi (je cite le commentaire) :

> Le service journalisait "buzz #N" juste apres client.publish(), qui est ASYNCHRONE. Sur quatre pupitres sur six, le journal affichait des appuis alors que le broker ne recevait RIEN depuis des heures. Le service mentait, et le diagnostic est parti dans le decor (cablage, GPIO, boutons) pendant une journee entiere.

Trois corrections en sont sorties :

- **distinguer l'appui détecté de l'appui confirmé** par le broker (callback `on_publish`), et compter les appuis perdus : *« Client MQTT qui ne ment pas : il distingue emis, confirme et perdu »* ;
- **se connecter de façon asynchrone et réessayer indéfiniment**, y compris au tout premier essai : après une coupure de courant, un pupitre démarre souvent plus vite que le serveur ;
- **surveiller la boucle réseau** du client MQTT : un client peut rester « connecté » côté socket avec une boucle morte. Au-delà de 60 secondes sans connexion, le service s'arrête volontairement, et systemd le relance.

```python
# Au-dela de ce delai sans connexion au broker, on se suicide : systemd relance un processus
# propre. Mieux vaut un redemarrage de 3 s qu'un service vivant qui n'emet plus rien.
MAX_DISCONNECTED_S = 60
WATCHDOG_PERIOD_S = 5
```

Même les logs ont droit à leur commentaire : chaque ligne est écrite avec `flush=True`, parce que sans cela elles restent dans le tampon et n'apparaissent qu'à la mort du service, « inexploitable pour diagnostiquer une salle en pleine partie ».

## La lumière : du DMX écrit à la main

Chaque site a six projecteurs DMX, un par équipe : le projecteur *n* éclaire le pupitre *n*. La première version pilotait un adaptateur USB FTDI FT232 via le pilote D2XX, en générant la trame DMX à la main :

```csharp
ftdi.SetBreak(true);  Thread.Sleep(1);   // ~1ms break (spec min is 88µs)
ftdi.SetBreak(false); Thread.Sleep(1);   // MAB
var buffer = new byte[1].Concat(data).ToArray();   // start code 0x00 + channels
ftdi.Write(buffer, buffer.Length, ref written);
```

Chaque pupitre a aussi son propre contrôleur DMX pour sa barre de LED : rose en attente, vert pour une bonne réponse, rouge pour une mauvaise.

La migration vers notre agent générique MQTTdmx a apporté deux leçons. La première est un classique : l'ancien code adressait les canaux à partir de 0, le nouvel agent en absolu à partir de 1 et ignore le canal 0. Sur un site, le journal a compté 152 lignes « Command ignored, channel: 0 » avant que quelqu'un ne passe un décalage global de 0 à 1. La seconde : l'ancien contrôleur n'écrivait qu'à chaque commande, alors que le nouvel agent émettait en continu, et les projecteurs réglés en mode HOLD se sont mis à scintiller. D'où le mode `send_on_change` décrit dans le [premier article](./team-break-architecture.fr.md).

## Le son : une enquête en règle

Un site compte entre 3 300 et 3 600 sons : voix off, ambiances, jingles, réponses. Ils sont joués par MQTTaudioplayer, notre lecteur audio générique, au format WAV 44,1 kHz 16 bits.

Des sons qui se chevauchaient et des silences inexpliqués ont donné lieu à une véritable enquête, consignée dans un document d'analyse : causes numérotées, probabilités, moyens de les distinguer, état de la correction. Côté moteur, quatre causes ont été retenues et corrigées. La plus instructive est un `return` à la place d'un `continue` : une instruction audio différée interrompait toutes les instructions suivantes de l'étape, et la tâche qui la portait ne pouvait pas être annulée à l'arrêt de la partie.

```csharp
if (audioInstruction.Offset > 0)
{
    // Delayed instructions must not abort the remaining instructions of the step,
    // and must be cancellable when the game stops ...
    CancellationToken delayedToken = _delayedAudioInstructionsCts.Token;
    _ = Task.Run(async () => {
        try { await Task.Delay(audioInstruction.Offset, delayedToken);
              await RunAudioInstructionsAsync(audioInstruction, delayedToken); }
        catch (OperationCanceledException) { }
        catch (Exception ex) { _logger.LogError(ex, "..."); }
    }, CancellationToken.None);
    continue;
}
```

Les publications audio sont aussi passées de QoS 0 à QoS 1. Et un « lissage » a été ajouté : pendant la phase de réponse (le « tic-tac »), seule la boucle d'ambiance de la manche est baissée à zéro, grâce à une nouvelle commande `FadeTrack`, au lieu de tout couper.

Un audit des sites a révélé une autre cause de silence, plus prosaïque : entre 149 et 384 sons manquaient par site, environ 1 450 au total, parce que le job de conversion audio écrivait ses fichiers hors du volume monté.

## Le contenu : questions, thèmes, et un éditeur avec IA

Une question porte une bonne réponse et de 2 à 11 mauvaises, une éventuelle « réponse blague », une astuce, une phrase de voix off, un visuel (image ou vidéo), un son pour la bonne réponse et une explication. Elle est rattachée à un thème, une catégorie et des tags. Des **parties à thème** (« 100 % Musique », « 100 % Horreur », version enfants) sont construites à partir de ces tags. Lors de la construction d'une partie, l'historique des questions déjà vues par les clients est exclu : une équipe qui revient ne retombe pas sur les mêmes questions.

Le contenu se prépare avec un **éditeur dédié**, une application de bureau Python (PySide6) qui édite le fichier de questions et suit l'origine de chacune (historique, modifiée, nouvelle). Un scanner signale les médias manquants avant l'export. Un panneau d'IA, branché sur Claude ou Gemini, génère des questions et en **audite** d'autres, avec une réponse imposée en JSON strict et des indicateurs explicites : `outdated`, `ambiguous`, `wrong_answer`, `too_easy`, `too_hard`, `not_so_fun`. Les colonnes produites par l'IA sont retirées de l'export envoyé au jeu. L'éditeur est livré en exécutable Windows par une CI qui pose le tag de version et publie la release.

### Synchroniser le siège et les sites

Le contenu est créé au siège, puis **synchronisé** vers chaque site par des jobs Hangfire :

- le contenu (thèmes, parties à thème, questions, diapositives, textes, distinctions) est tiré chaque nuit, à une heure tirée au hasard entre 22 h et 3 h, avec un décalage par job, pour que les sites ne frappent pas le siège en même temps ;
- les parties et les clients sont synchronisés toutes les 5 minutes, et les parties jouées remontent au siège toutes les heures.

La synchronisation est incrémentale, avec un filigrane qui **n'avance que si tous les éléments sont passés** :

```csharp
DateTime? since = contentSynchronizationInfo.Since;
var pulled = await reader.PullAsync(since);
foreach (var current in pulled.EmptyIfNull())
{
    try { await writer.PushAsync(current); }
    catch (Exception ex) { hasErrorOccurred = true; this.logger.LogError(ex, "..."); }
}
if (!hasErrorOccurred) { contentSynchronizationInfo.Since = executionDate; }
```

Un seul élément en échec fait tout rejouer à la passe suivante : c'est plus coûteux, mais rien n'est jamais sauté. Notre procédure de maintenance ajoute une règle de bon sens : **un job vert ne prouve rien**. On vérifie la base locale, et on vérifie qu'un média répond bien en HTTP 200. Les médias (8 à 9 Go par site) suivent leur propre chemin.

## La migration : sortir d'Avalonia sans arrêter les salles

### Le point de départ

Début 2026, Donut Quiz tournait sur une pile qui avait vieilli :

- **.NET 5**, hors support depuis longtemps ;
- des applications de pupitre et de saloon en **Avalonia 0.10.5**, avec LibVLC pour la vidéo et un décodeur GIF embarqué ;
- des Raspberry Pi sous **Raspbian Buster en 32 bits** (armhf), en X11 ;
- un déploiement par **archives zip** décompressées à la main sur chaque machine, depuis que les pipelines d'origine n'étaient plus disponibles.

La migration a été conduite avec un journal de 16 ADR. Je reprends les plus structurantes.

### La découverte qui change tout (ADR-0003)

Passer à .NET 10 imposait de migrer Avalonia de 0.10 à 11, une rupture majeure. En relisant le code, une découverte a tout simplifié :

> les pupitres sont déjà des **clients MQTT minces** — le `GameServer` diffuse des DTO JSON (`DeskMessage`) décrivant quoi afficher ; les ViewModels Avalonia ne sont qu'une machine à états « message → écran ».

D'où la décision : remplacer les vues Avalonia par des **vues web statiques en HTML, CSS et JavaScript vanilla**, servies par un nouvel hôte ASP.NET Core, avec l'état poussé en **SSE** (`EventSource`) et les entrées envoyées par `fetch()`. Blazor a été écarté : un framework, plus un WebSocket permanent ou un runtime WebAssembly, c'était trop lourd par rapport à l'exigence de minimalisme.

### Un pont, et zéro changement côté moteur (ADR-0005)

Le nouvel hôte s'abonne aux topics MQTT des pupitres et du saloon, garde le dernier message de chaque écran, et le pousse en SSE (`GET /desk/{couleur}/stream`). Un écran qui se connecte reçoit immédiatement le dernier état. Dans l'autre sens, `POST /desk/{couleur}/answer` publie le message attendu par le moteur sur le topic de l'orchestrateur. Résultat : **aucun changement dans le moteur de jeu ni dans les contrats**, validé de bout en bout.

Chaque pupitre affiche désormais deux navigateurs en kiosque sur deux URL (l'écran tactile et l'écran de score), abonnés au même flux.

### Reproduire avant d'améliorer (ADR-0006)

La décision la plus importante n'est pas technique. Pendant le portage, des comportements ont été *inventés* : buzzer mappé sur la barre d'espace, réponse envoyée dès le toucher. Ils étaient faux, et coûteux à détecter. D'où une consigne prioritaire : **reproduire le comportement existant**, en lisant le code de l'ancienne application avant d'écrire la nouvelle. « Ne rien inventer. » Le toucher sélectionne, le buzzer valide, comme avant.

Les projets Avalonia ont d'abord été gelés hors du build, puis supprimés du dépôt au basculement (ADR-0010). Pas de *big bang*.

### Changer d'architecture de processeur (ADR-0014 et 0016)

Un `dist-upgrade` ne change pas l'architecture : un Pi en armhf reste en armhf. Or la cible (.NET 10, Docker, MongoDB) est arm64. Il fallait donc **reflasher**, et le faire de façon reproductible :

- des **images « golden » par rôle** (pupitre, saloon, donut, serveur), générées avec `rpi-image-gen` sur une base Debian 13 minimale, en arm64 et en Wayland (labwc), avec toutes les versions épinglées ;
- un fichier **`donutquiz.conf` sur la partition de démarrage FAT**, éditable depuis n'importe quel PC sans démarrer le Pi : `ROLE`, `COLOR`, `SERVER_URL`… Au premier démarrage, un service *oneshot* lit ce fichier, génère les configurations, recale le tactile, fixe le hostname, puis active les services.

La procédure terrain tient en trois mots : flasher, éditer `donutquiz.conf`, brancher. Un guide de remplacement de carte SD permet à un responsable de site de remettre un pupitre en service en une vingtaine de minutes.

### Des paquets qui n'installent rien d'actif (ADR-0012)

Chaque composant est devenu un **paquet Debian versionné** (`donutquiz-ui`, `-buzzer`, `-screens`, `-gameserver`, `-admin`…) qui **n'active rien** à l'installation. Le basculement est explicite et réversible grâce à trois outils : `dq-cutover` pour basculer, `dq-rollback` pour revenir en arrière, `dq-doctor` pour vérifier l'état. L'ancienne et la nouvelle pile cohabitent sur la même machine (base, réglages et médias séparés) : revenir en arrière, c'est arrêter un service et en relancer un autre.

### Le déploiement, site par site

Aéroville a été migré en premier, début août, pendant que la salle continuait de jouer. Puis La Défense et Paris 9, puis Lille. Chaque site a apporté sa leçon :

- deux projets Docker Compose partageaient un nom, et `--remove-orphans` a arrêté les services du voisin ;
- les clés de protection des données d'ASP.NET vivaient dans le conteneur : chaque reconstruction cassait silencieusement les boutons de l'administration ;
- une dépendance déclarée en `Recommends: mongodb-clients` a mis à jour MongoDB sans prévenir, de 4.4.8 à 4.4.29 ;
- or **MongoDB 4.4.19 et suivants exigent ARMv8.2-A**, que le Cortex-A72 du Raspberry Pi 4 n'a pas. Symptôme : `Illegal instruction`. Remède : épingler `mongo:4.4.18` et `apt-mark hold` ;
- des pupitres reflashés gardaient le hostname par défaut et partageaient les mêmes clés d'hôte SSH.

Aujourd'hui, nos quatre sites en propre tournent sur la nouvelle pile ; les franchises suivent.

## Les pupitres qui devenaient sourds

Le passage au web a introduit un nouveau mode de panne. À Lille, le 11 septembre, **cinq pupitres sur six étaient sourds depuis des heures, sans une seule erreur nulle part**. Le commentaire du module de transport explique pourquoi :

> sur une connexion TCP semi-ouverte (commutateur, point d'accès, table NAT), le navigateur attend des octets qui ne viendront jamais, readyState reste OPEN, aucun onerror ne part, et la reconnexion native d'EventSource ne se déclenche pas.

La correction couvre les deux façons dont un flux peut mourir : la fermeture (`onerror` avec `readyState === CLOSED`) et le **silence prolongé**, détecté par un minuteur de vivacité. Petite subtilité : un commentaire SSE (`: ping`) ne déclenche pas `onmessage`, donc le *keepalive* du serveur était invisible côté client. Le serveur émet désormais aussi un événement nommé `ping`, que le client écoute explicitement.

```js
/** Silence toléré (aucun octet : ni message, ni keepalive) avant de rouvrir le flux. */
export const LIVENESS_TIMEOUT_MS = 45000;

/** Premier palier de reprise. */
export const RECONNECT_MIN_DELAY_MS = 1000;

/** Plafond de reprise : on ralentit, on n'abandonne jamais. */
export const RECONNECT_MAX_DELAY_MS = 30000;
```

Les seuils sont choisis, pas subis : 45 s correspondent à trois *keepalives* manqués, et la reprise plafonne à 30 s pour qu'une salle de treize flux ne transforme pas un serveur momentanément lent en tempête de reconnexions. Un aller-retour *ping-pong* remonte aussi la vivacité de chaque écran jusqu'à l'administration.

## Archipel et vigie

Sur les sites équipés d'[Archipel](./archipel-orchestration.fr.md), la borne d'accueil pilote aussi Donut Quiz. Il n'y a pas de Maestro dans une salle Donut Quiz, donc pas de MQTT entre les deux : Archipel appelle l'API de Donut Quiz en HTTP. Il retrouve la partie à partir de la réservation, transmet le thème choisi par les joueurs, le nombre d'équipes et la langue, puis lance la partie. La borne ne propose jamais plus d'équipes qu'il n'y a de pupitres en état de marche.

Le client HTTP côté Archipel ne lève jamais d'exception. Chaque appel a un délai de 5 secondes, avec un budget global de 8 secondes, et le commentaire donne la raison : « une salle Donut Quiz éteinte ne doit pas figer l'écran des joueurs ».

Côté exploitation, [vigie](./vigie-exploitation.fr.md) surveille les services de chaque machine Donut Quiz, nommées par fonction (`donutquiz-<site>-centralserver`, `-gameserver`, `-desk<couleur>`, `-saloon`). C'est un service de synchronisation Donut Quiz qui avait redémarré des milliers de fois sans que personne ne le voie, et qui a motivé la remontée des compteurs de redémarrage dans la feuille de route de vigie.

## En chiffres

| | Ancienne pile | Nouvelle pile |
|---|---|---|
| Runtime | .NET 5 | .NET 10 |
| Écrans | Avalonia 0.10.5 | HTML/CSS/JS vanilla + SSE |
| OS des Pi | Raspbian Buster armhf, X11 | Debian 13 arm64, Wayland |
| Déploiement | zip à la main | paquets Debian, cutover/rollback |
| Tests | 88 méthodes de test | près de 2 000 |

L'ancienne solution comptait 38 projets et près de 99 000 lignes de C#.

## Ce que j'en retiens

1. **Un client mince est un cadeau.** Parce que les pupitres ne faisaient qu'afficher des messages MQTT, nous avons pu remplacer toute l'interface sans toucher au moteur de jeu.
2. **Reproduire avant d'améliorer.** Dans une migration, chaque comportement « amélioré » au passage est un bug pour quelqu'un. On lit l'ancien code d'abord.
3. **Un service qui ment est pire qu'un service mort.** Le buzzer qui journalisait des appuis jamais envoyés nous a coûté une journée. Un log doit dire ce qui s'est *passé*, pas ce qu'on a *demandé*.
4. **Une connexion ouverte n'est pas une connexion vivante.** TCP semi-ouvert, EventSource bloqué en `OPEN` : seule une mesure active de vivacité le détecte.
5. **On migre salle ouverte, côte à côte, de façon réversible.** Les deux piles cohabitent, et revenir en arrière tient en une commande.
6. **Le matériel impose ses lois.** Un dist-upgrade ne change pas l'architecture, et une version mineure de MongoDB peut exiger une instruction que votre processeur n'a pas.
7. **Le contenu est un produit à part entière.** Éditeur, audit assisté, historique par client, synchronisation : la meilleure architecture ne sauve pas une question fausse.

Quand six équipes se disputent le *Donut d'or* et que les buzzers claquent tous dans la même seconde, personne ne pense aux 81 machines, au SSE ou au DMX. C'est exactement l'objectif.
