---
title: "Donut Quiz : les coulisses techniques d'une salle de quiz connectée"
description: "Neuf Raspberry Pi, treize écrans, du son, du DMX et des buzzers. Retour d'expérience sur la conception de la salle Donut Quiz, et sur la refonte qui l'a rendue stable : fin d'Avalonia, Akka.NET remplacé par MQTT, .NET 10, paquets Debian et images Raspberry Pi reproductibles."
lang: fr
tags: [dotnet, mqtt, raspberry-pi, sse, dmx, iot, architecture, retour-experience]
---

# Donut Quiz : les coulisses techniques d'une salle de quiz connectée

Donut Quiz est une salle de jeu Team Break. Plusieurs équipes s'affrontent dans un quiz rythmé, chacune derrière son pupitre, avec un grand écran commun, du son et des lumières. Les manches portent des noms de pâtisserie (*Mise en bouche*, *Pour du beurre*, *Apéro*, *Surprise du chef*, *Milkshake de frites*, *Cerise sur le gâteau*), on s'envoie des coups spéciaux (*Bouchée double*, *Pique-assiette*, *Retard en cuisine*…) et la partie finit par une remise des honneurs.

Pour les joueurs, c'est une heure de jeu. Pour moi, c'est un système distribué d'une dizaine de machines qui doit tenir sa cadence à la seconde près, sans informaticien sur place, dans des agences réparties dans toute la France.

Cet article raconte les difficultés rencontrées pour concevoir cette salle, puis la refonte qui l'a rendue **plus évolutive, plus stable et beaucoup moins lourde à maintenir** : fin d'Avalonia, remplacement d'Akka.NET par MQTT, passage de .NET 5 à .NET 10, déploiement par paquets Debian et images Raspberry Pi reproductibles.

> **En bref**
> - Les pupitres ne sont plus des applications Avalonia : ce sont des pages web en HTML/CSS/JS sans framework, affichées par un navigateur en mode kiosque et alimentées par Server-Sent Events.
> - Akka.NET a laissé la place à un broker MQTT. Le serveur de jeu, les écrans, le lecteur audio (en Python), les agents DMX et les buzzers parlent tous le même protocole.
> - Toute la solution est passée de .NET 5 à .NET 10 (LTS), avec une gestion centralisée des paquets NuGet. On est passé de 44 à 30 projets et de 31 à près de 2 000 tests.
> - La synchronisation entre le siège et les agences a été un chantier à part entière, que je raconte dans [un article dédié](./donut-quiz-synchro-fr.md).
> - Les déploiements Azure DevOps à base de zip ont été remplacés par des paquets `.deb` versionnés et réversibles, des images Raspberry Pi générées par rôle, et une migration à distance des Pi de 32 à 64 bits sans déplacement sur site.

---

## La salle en quelques chiffres

| Élément | Détail |
|---|---|
| **Pupitres** | 6, aux couleurs *chocolat*, *citron*, *caramel*, *fraise*, *pistache* et *myrtille*. Chacun a un écran tactile de jeu, un écran de score, un buzzer physique et une barre LED RGB. |
| **Saloon** | Un grand écran public qui diffuse l'ambiance et les classements. |
| **Lumière** | Des spots pilotés en DMX512, plus les LED des pupitres. |
| **Son** | Une sonorisation de salle avec trois groupes de pistes (ambiance, voix, effets). Environ 3 400 pistes par salle. |
| **Machines** | 9 Raspberry Pi 4 par salle (6 pupitres, saloon, « donut », serveur), soit **13 écrans**. |
| **Médias** | 13 à 19 Go d'images, GIF, vidéos et sons, synchronisés depuis le siège. |
| **Parc** | 9 agences, en propre et en franchise. |

## Pourquoi c'est plus difficile qu'il n'y paraît

Sur le papier, c'est un quiz. Dans les faits, plusieurs contraintes se cumulent.

- **Le temps réel se voit, et s'entend.** Une voix qui arrive une seconde trop tard, deux sons qui se superposent, une LED qui reste rouge, un pupitre qui fige : toute la salle le remarque. Dans un jeu en équipe, l'injustice perçue est pire que le bug lui-même.
- **Personne de technique sur place.** La salle est animée par des game masters, installée par des techniciens et maintenue à distance. Un diagnostic qui exige un SSH et la lecture d'un JSON n'en est pas un.
- **Du matériel ordinaire.** Raspberry Pi, cartes SD, GPIO, dongles USB-DMX, écrans tactiles dont l'EDID annonce n'importe quoi, spots de modèles différents selon l'agence, câblage fait à la main.
- **Une longue durée de vie.** Le logiciel doit survivre aux nouvelles versions de Raspberry Pi OS, à la fin de support des frameworks et aux remplacements de matériel.
- **Hors ligne par nature.** La salle doit jouer sur son réseau local. Le siège ne sert qu'à fournir le contenu (questions, thèmes, médias), qui est synchronisé vers chaque agence.

---

## La version 1 : ce qui a marché, et ce qui a coûté cher

La première version du jeu a été développée par un prestataire : mon ancienne société, dont j'étais gérant associé. Elle tournait sur la pile suivante :

- **.NET 5**, avec des interfaces **Avalonia 0.10** sur les pupitres et le saloon (LibVLCSharp pour la vidéo, un contrôle dédié pour les GIF animés) ;
- des **Raspberry Pi sous Raspbian Buster en 32 bits (armhf)** avec X11 ;
- **Akka.NET** pour faire communiquer les composants, plus un **Hello Service** chargé de la découverte : chaque composant s'y annonçait pour que le serveur de jeu sache à quelle adresse IP trouver qui ;
- **MongoDB** pour les données ;
- **Azure DevOps** pour le code source, la CI et le déploiement, choisi parce que l'équipe le maîtrisait et qu'il était gratuit jusqu'à cinq utilisateurs. Un pipeline de build produisait un bundle self-contained, puis des *release pipelines* par agence et par salle étaient exécutés par un **agent auto-hébergé installé dans chaque agence**. L'inventaire de l'infrastructure (adresse IP de chaque composant, coefficient de volume, offset DMX…) vivait dans des fichiers Excel versionnés que les pipelines lisaient.

Les composants avaient des noms explicites : *Game Server* (le moteur), *Game Master* (sons et spots), *Pupitre*, *Écran partagé* (le saloon), *Administration*, *Synchronization*, *Checkin* et *Hello Service*.

Ça marchait. Mais chaque site finissait par devenir un cas particulier, et j'en garde quelques souvenirs de terrain.

- **.NET 5 n'était pas une LTS.** Un jour, impossible de produire le bundle de déploiement : les certificats NuGet des SDK antérieurs à 5.0.202 avaient expiré, et comme la version n'était pas LTS, ils n'avaient pas été renouvelés. Correctif : passer l'agent de build d'Ubuntu 18.04 à 22.04 et monter le SDK. Leçon : on ne met pas une version non-LTS dans un produit qui doit vivre des années.
- **Un buzzer qui n'était plus détecté** après une mise à jour de Raspbian, à cause d'un bug de la bibliothèque .NET IoT. J'ai dû passer d'une écoute en numérotation logique (GPIO 21) à une numérotation physique (broche 40), puis déployer le correctif pupitre par pupitre. Le numéro de broche était codé en dur, si bien qu'une broche défectueuse aurait imposé un build spécifique. Il est désormais configurable.
- **Aucun son en salle.** Le processus de déploiement installait le pilote d'une carte son HifiBerry sur un Pi qui utilisait la sortie jack.
- **Le navigateur bloquait le port de l'administration.** Elle écoutait sur le port 10080, que Chrome et Firefox classent parmi les ports « dangereux ». La documentation a dû s'enrichir d'une annexe pour l'autoriser, navigateur par navigateur.
- **La sortie de Raspbian Bullseye** a obligé à épingler Buster, que les outils ne proposaient plus par défaut.

Aucun de ces problèmes n'était difficile à corriger. Ce qui coûtait cher, c'était **le coût de chaque correctif** : un build, une release, un passage sur chaque machine, et une configuration qui divergeait un peu plus d'un site à l'autre.

---

## Akka.NET → MQTT : choisir un protocole plutôt qu'un framework

Sur le papier, Akka.NET avait tout pour plaire : un modèle acteur, la transparence de localisation, et un serveur de jeu qui distribue des ordres à des pupitres ressemble beaucoup à un acteur qui envoie des messages à d'autres acteurs.

À l'usage, trois limites sont apparues.

1. **Le remoting est point à point.** Chaque nœud doit connaître l'hôte et le port des autres. D'où le Hello Service, l'inventaire des IP dans Excel, et une reconfiguration de la salle à chaque remplacement de machine.
2. **Tout le monde doit parler .NET et Akka**, dans des versions et des formats de sérialisation compatibles. Or l'écosystème matériel réclamait autre chose : un lecteur audio en Python, de petits agents DMX, et à terme des microcontrôleurs (ESP32) pour les buzzers.
3. **C'est difficile à observer.** Impossible de « tendre l'oreille » pour savoir ce que la salle est en train de se dire.

MQTT règle ces trois points.

- **Une seule adresse à connaître**, celle du broker Mosquitto qui tourne sur le serveur de la salle. Le Hello Service a disparu.
- **Le publish/subscribe** correspond exactement au besoin. Le serveur publie un message par pupitre, un message pour tous les pupitres, et un pour le saloon.
- **C'est polyglotte** : C#, Python, firmware ESP32, tout le monde sait parler MQTT.
- **C'est observable.** `mosquitto_sub -v -t '/games/dq/#'` affiche toute la partie en direct.

| Topic | Sens | Rôle |
|---|---|---|
| `/games/dq/orchestrator` | écrans → serveur de jeu | commandes et entrées des joueurs (réponse, inscription, coup spécial…) |
| `/games/dq/desks/{couleur}` | serveur → un pupitre | ce que le pupitre doit afficher |
| `/games/dq/desks/all` | serveur → tous les pupitres | diffusion |
| `/games/dq/saloon` | serveur → saloon | écran public |
| `/games/dq/desks/{couleur}/led` | serveur → agent DMX | couleur de la barre LED du pupitre |
| `/games/dq/lights` | serveur → agent DMX | spots de la salle |
| `/games/dq/audio` | serveur → lecteur audio | lecture, arrêt, fondus |
| `/games/dq/desks/{couleur}/buzz` | buzzer → serveur | appui sur le buzzer |

Le principe d'architecture qui en découle est simple : **le serveur de jeu est la seule source de vérité.** Il publie des DTO qui décrivent ce qu'il faut afficher, jouer ou allumer, et les consommateurs se contentent d'exécuter. Les pilotes matériels sont sortis du code métier et confiés à de petits agents spécialisés :

- **MQTTaudioplayer** (Python) joue les pistes sur ALSA ;
- **MQTTdmx** traduit les messages en trames DMX sur un dongle USB-FTDI, avec un agent par bus ;
- **le buzzer** lit un GPIO et publie sur `/desks/{couleur}/buzz`. Demain, ce sera un ESP32 sur **le même topic**, et ni le serveur ni les écrans ne verront la différence.

### Ce que MQTT ne fait pas à votre place

- **La QoS.** Par défaut, MQTTnet publie en QoS 0 (« au plus une fois »). Sur un réseau de salle un peu chargé, un `play` perdu donne un silence, et un `stop` perdu donne une superposition. Toutes les commandes audio sont désormais publiées en `AtLeastOnce`, et rendues idempotentes pour que les doublons soient sans conséquence.
- **La reconnexion.** Chaque client doit se reconnecter indéfiniment et se réabonner, même si la toute première connexion a échoué. Autre piège : en version 4.0, `ConnectAsync` de MQTTnet attend le CONNACK sans aucune borne. Un broker gelé laisse alors un client « connecté » sans session réelle. J'ai donc écrit un `MqttConnectionKeeper` doté d'une sonde de vivacité.
- **Les jokers.** Les topics techniques (la supervision des écrans, voir plus bas) vivent sous une autre racine, `/screens/dq/`. Si un agent tiers s'abonne un jour à `/games/dq/#`, il ne recevra jamais de message inattendu. Le risque est écarté par construction, pas par hypothèse.

---

## La fin d'Avalonia : des pupitres qui ne sont plus que des navigateurs

Passer à .NET 10 imposait de migrer d'Avalonia 0.10 à Avalonia 11, une rupture majeure, en plus de LibVLC sur ARM et de X11. En relisant le code des pupitres, j'ai fait la découverte qui a tout débloqué : **les pupitres étaient déjà des clients légers.** Le serveur publiait un `DeskMessage` en JSON, et les ViewModels Avalonia n'étaient qu'une machine à états qui transformait un message en écran.

L'interface n'avait donc qu'un seul travail : afficher un DTO JSON. Le web fait ça très bien.

### Les choix

- **HTML, CSS et JavaScript, sans framework.** J'ai écarté Blazor : il faut soit le runtime WebAssembly, soit un circuit SignalR permanent, ce qui est trop lourd pour un Raspberry Pi et pour un besoin aussi simple.
- **Un hôte web central.** Une seule application ASP.NET Core, `GameScreens.Web`, sert tous les écrans de la salle. Chaque Pi fait tourner Firefox en mode kiosque sous Wayland (labwc).
- **Deux surfaces par pupitre** : `/desk/{couleur}` pour le jeu tactile et `/desk/{couleur}/score` pour le score. Elles s'abonnent au même flux.
- **Un pont MQTT → SSE.** Un `IHostedService` s'abonne aux topics des pupitres et du saloon, conserve le dernier état de chaque écran et le pousse aux navigateurs en Server-Sent Events. Les entrées des joueurs repartent par un simple `fetch` POST, republié sur `/orchestrator`. **Le serveur de jeu n'a pas changé d'une ligne.**

Pourquoi SSE plutôt que WebSocket ? Presque tout le trafic va du serveur vers l'écran. SSE, c'est du HTTP ordinaire, avec une reconnexion native, et un `curl` suffit pour le déboguer. Les rares entrées des joueurs se prêtent très bien à un POST.

```csharp
// GameScreens.Web — flux SSE d'un écran (version simplifiée)
public static async Task StreamAsync(HttpContext ctx, ScreenStateStore store, string screenKey, CancellationToken ct)
{
    ctx.Response.Headers.ContentType = "text/event-stream";
    ctx.Response.Headers.CacheControl = "no-cache";
    ctx.Response.Headers["X-Accel-Buffering"] = "no";

    var channel = Channel.CreateUnbounded<string>(new() { SingleReader = true });
    void Handler(string key, string json) { if (key == screenKey) channel.Writer.TryWrite(json); }

    store.Subscribe(Handler);
    try
    {
        await ctx.Response.WriteAsync(": connected\n\n", ct);
        await ctx.Response.Body.FlushAsync(ct);

        // Rejoue le dernier état, avec son ancienneté, pour qu'une vidéo reprenne au bon instant.
        // Un état trop vieux (plus de 5 min) n'est jamais rejoué : l'écran reste sur son accueil.
        if (store.TryGetLatest(screenKey, out var latest, out var elapsedMs) && elapsedMs <= StaleReplayCapMs)
            await WriteEventAsync(ctx, InjectResyncElapsed(latest, elapsedMs), ct);

        while (!ct.IsCancellationRequested)
        {
            var wait = channel.Reader.WaitToReadAsync(ct).AsTask();
            if (await Task.WhenAny(wait, Task.Delay(KeepAliveInterval, ct)) == wait)
            {
                if (!await wait) break;
                while (channel.Reader.TryRead(out var json)) await WriteEventAsync(ctx, json, ct);
            }
            else
            {
                // Toutes les 15 s : un commentaire (pour les sondes curl) PUIS un événement nommé.
                await ctx.Response.WriteAsync(": ping\n\nevent: ping\ndata: 1\n\n", ct);
                await ctx.Response.Body.FlushAsync(ct);
            }
        }
    }
    catch (OperationCanceledException) { /* client parti */ }
    finally { store.Unsubscribe(Handler); }
}
```

### Reproduire avant d'améliorer

La règle la plus importante du portage a été la suivante : **l'écran web doit se comporter exactement comme l'écran Avalonia.** Pour chaque interaction, je lisais d'abord le code legacy, puis j'écrivais le code web. Les comportements « améliorés » que j'aurais pu inventer, comme envoyer la réponse au toucher ou mapper le buzzer sur la barre d'espace, sont des bugs pour des joueurs et des game masters habitués au jeu. Dans le jeu d'origine, **un toucher sélectionne une réponse, qui reste modifiable, et c'est le buzzer qui la valide** en envoyant le temps de réponse. Le logo Donut Quiz est lui-même un buzzer cliquable.

### Porter la charte graphique

- Les formes (bulles en trapèze, cartes, bandeau) étaient des `Polygon` Avalonia dérivés de SVG. Elles redeviennent des fichiers SVG, injectés inline pour que `fill: var(--…)` suive le thème.
- Les huit thèmes sont des jeux de variables CSS. La police Montserrat est conservée.
- **Les textes trop longs** ont été la première remarque d'un client après la migration : des réponses et des explications qui débordaient des bulles. Le mécanisme d'ajustement de police existait, mais il n'était pas branché sur les tuiles de réponse. Il a aussi fallu le fiabiliser deux fois : réessayer au `requestAnimationFrame` suivant quand la boîte n'a pas encore de largeur, et relancer la mesure après `document.fonts.ready`, parce qu'une mesure prise avec la police de repli concluait trop tôt que « ça tient ».
- **La synchronisation vidéo.** Quand un écran se (re)connecte, l'état rejoué porte `resync_elapsed_ms`. Une slide vidéo reprend ainsi au bon instant, en phase avec le saloon et les autres pupitres, au lieu de repartir de zéro.

### Les médias

Avant, chaque pupitre avait sa propre copie des médias, soit plus de 13 Go par machine. Aujourd'hui, ils vivent dans un volume Docker unique sur le serveur de la salle et sont servis sur le réseau local avec `Cache-Control: public, max-age=31536000, immutable`. Le contenu étant adressé par identifiant, il ne change jamais. Le cache disque du navigateur, sur la carte SD, fait le reste.

### Durcir un kiosque

Désactiver le pinch-zoom et le balayage à deux doigts, revenir automatiquement à l'écran d'accueil au lieu d'afficher un écran noir, recaler le tactile sur la bonne sortie vidéo quand le Pi pilote deux écrans, choisir 1280×720 par défaut quand l'écran annonce des modes fantaisistes… Aucune de ces tâches n'est glorieuse, et chacune vient d'une remontée de terrain.

---

## « Connecté » ne veut pas dire « vivant »

C'est probablement la leçon la plus importante de la refonte.

Le piège s'appelle la connexion TCP semi-ouverte : un switch, un point d'accès ou une table NAT perd la connexion sans prévenir personne. Le navigateur attend alors des octets qui ne viendront jamais, `readyState` reste à `OPEN`, aucun `onerror` ne se déclenche, et la reconnexion native d'`EventSource` ne se produit donc pas. Un écran peut ainsi devenir sourd sans qu'aucune erreur ne remonte nulle part. Pire encore : le keepalive du serveur était un **commentaire** SSE (`: ping`), et un commentaire ne déclenche aucun gestionnaire JavaScript. La page n'avait donc aucun moyen de distinguer « rien à dire » de « liaison morte ».

La correction se fait en trois couches.

1. **Côté serveur**, un événement *nommé* `ping` est émis toutes les 15 secondes, en plus du commentaire.
2. **Côté navigateur**, une garde de vivacité rouvre le flux après 45 secondes sans le moindre octet, soit trois keepalives manqués. La reconnexion suit un backoff exponentiel plafonné à 30 secondes, pour que treize écrans ne transforment pas un serveur momentanément lent en tempête de reconnexions. Et deux `EventSource` ne coexistent jamais.

```javascript
// stream.js (extrait)
export const LIVENESS_TIMEOUT_MS = 45000;   // 3 keepalives serveur manqués
export const RECONNECT_MAX_DELAY_MS = 30000; // on ralentit, on n'abandonne jamais

function armerGarde() {
  livenessTimer = annuler(livenessTimer);
  livenessTimer = setTimeout(programmerReprise, LIVENESS_TIMEOUT_MS);
}

function programmerReprise() {
  if (closed || reconnectTimer !== null) return;
  livenessTimer = annuler(livenessTimer);
  if (es) { es.close(); es = null; }          // jamais deux flux ouverts en même temps
  const delai = Math.min(RECONNECT_MAX_DELAY_MS, 1000 * 2 ** echecs++);
  reconnectTimer = setTimeout(() => { reconnectTimer = null; connect(); }, delai);
}

function connect() {
  es = new EventSource(url);
  es.onmessage = (ev) => { surActivite(); /* … */ };
  es.addEventListener('ping', surActivite);   // le keepalive devient observable
  es.onerror = () => { if (es.readyState === EventSource.CLOSED) programmerReprise(); };
  armerGarde();
}
```

3. **Dans l'autre sens**, chaque écran répond à chaque ping par un *pong* publié en MQTT. Le pong précise la surface (jeu ou score), parce qu'un pupitre dont seul l'écran de score est mort ne doit pas apparaître en vert. L'administration connaît ainsi l'état réel des 13 écrans, et le serveur relance de lui-même un pupitre décroché.

La première version de la supervision comptait les sockets ouvertes côté serveur. Elle reposait sur quatre indirections (compteur en mémoire, JSON en HTTP, sonde HTTP, parsing en bash), et **deux d'entre elles ont menti alors que leurs tests étaient verts**. Un aller-retour ping/pong prouve en une seule mesure ce que le compteur ne prouvait pas : le JavaScript tourne, il reçoit, il répond, et sa réponse arrive.

---

## Le son : là où tout s'entend

Le modèle audio est simple à énoncer. Trois groupes de pistes (`ambient`, `voices`, `effects`). Un `play solo` remplace les pistes du même groupe. Un `notify` joue un son en baissant l'ambiance (*ducking*), puis la remonte. Des fondus enchaînent le tout. Le lecteur est un programme Python (pygame) piloté en MQTT sur le Pi « donut ».

Les symptômes étaient tout aussi simples : **des superpositions et des silences, « de temps en temps ».** L'analyse a mis au jour huit causes distinctes, réparties entre le serveur de jeu et le lecteur. Voici ma préférée :

```csharp
// Avant
foreach (var audioInstruction in audioInstructions)
{
    if (audioInstruction.Offset > 0)
    {
        _ = Task.Run(async () =>
        {
            await Task.Delay(audioInstruction.Offset, cancellationToken); // jeton jamais annulé en amont
            await RunAudioInstructionsAsync(audioInstruction, cancellationToken);
        }, cancellationToken);
        return; // ← abandonne en silence TOUTES les instructions suivantes de l'étape
    }
    await RunAudioInstructionsAsync(audioInstruction, cancellationToken);
}
```

On a là un `return` qui aurait dû être un `continue`, et une tâche *fire-and-forget* qu'aucun arrêt de partie n'annulait. Une instruction différée pouvait donc se déclencher par-dessus l'étape suivante, voire après un `stopall`. Dans la version corrigée, chaque instruction différée reçoit un jeton annulé à l'arrêt ou au chargement d'une partie, ses exceptions sont journalisées, et la boucle continue.

Les autres causes :

- les publications MQTT en QoS 0, déjà évoquées ;
- un `FadeOutBackground` envoyé systématiquement, même pour une question sans voix off : l'ambiance tombait à 20 % pendant 30 à 90 secondes, ce que la salle percevait comme un blanc ;
- côté lecteur, un *ducking* appliqué avant le `play` et jamais restauré si la lecture échouait.

Ensuite, la chaîne de contenu. Chaque son est converti en WAV propre par ffmpeg dès l'upload (dans un job Hangfire), et l'administration publie un manifeste `/config/audio` pour que le lecteur télécharge lui-même ses pistes.

---

## La lumière : DMX512 sans douleur

Le DMX est un protocole d'une simplicité désarmante : 512 canaux, chacun avec une valeur de 0 à 255, émis en boucle, sans aucune notion d'adresse de destination. Chaque appareil écoute N canaux à partir de son adresse de départ.

Le serveur de jeu modélise des **storyboards lumineux** (début de partie, bonne ou mauvaise réponse, honneurs…) et publie des messages `one_shot`, `fade`, `loop` ou `blackout`. L'agent MQTTdmx, un par bus, entretient la trame de 512 canaux et l'émet sur le dongle USB-FTDI.

- **Les LED des pupitres.** Chaque pupitre a son propre contrôleur, donc son propre bus. Les six pupitres utilisent tous la même adresse de départ, et ce n'est pas une faute de frappe : il n'existe aucune chaîne commune à décaler.
- **Les spots** sont contigus sur un même bus : `canal = GlobalOffset + (n − 1) × BufferSize + Index`. L'*Index* (la position d'une couleur dans un spot) vient de la notice du fabricant. L'*Offset* dépend du câblage sur site.
- **Un bug de terrain classique** : `GlobalOffset = 0`. Le DMX commence au canal 1, l'agent ignorait donc le canal 0 et le premier spot restait éteint.
- **La robustesse** : un `Index` qui dépasse `BufferSize` (une saisie manuelle) est désormais ignoré pour la seule fonction concernée, au lieu de faire perdre toute la trame.
- **Le stroboscope est désactivé, définitivement.** C'est un parti pris de jeu, pas un réglage à découvrir sur site.

Un seul fichier de configuration, `/etc/donutquiz/gameserver-dmx.json`, remplace les deux anciens réglages : l'offset dans l'appsettings de chaque pupitre Avalonia et la configuration de l'ancien « spotlights master ». Et quand deux couleurs s'allument ensemble sur un pupitre, le fait de pouvoir vérifier les trames à distance permet de mettre le logiciel hors de cause et d'envoyer le technicien directement au bornier.

---

## La synchro siège ↔ agences

Une salle doit pouvoir jouer même si la connexion internet de l'agence tombe. Chaque agence a donc sa propre base et ses propres médias : le contenu descend du siège, et les parties jouées remontent. Entre un filigrane qui suivait la mauvaise horloge et des propriétés qui disparaissaient en silence derrière un 200 OK, ce chantier mérite son propre article : **[Donut Quiz : synchroniser un siège et des agences sans rien perdre en silence](./donut-quiz-synchro-fr.md)**.

---

## De .NET 5 à .NET 10, et un grand ménage

- **Une gestion centralisée des paquets.** La solution comptait 44 projets dont les versions de paquets divergeaient (`System.Device.Gpio` en 1.3.0 ici, en 2.2.0 là). Un `Directory.Packages.props` unique centralise désormais les versions. La politique est simple : on gèle le conteneur d'injection (Autofac) par `VersionOverride`, pour ne rien changer au comportement, et on aligne les paquets triviaux vers le haut.
- **Un saut direct de `net5.0` à `net10.0`**, sans étape intermédiaire en .NET 8. Les vraies ruptures étaient peu nombreuses mais sournoises : le RID `win10-x64` devenu `win-x64`, MSTest 2 incompatible, `UseHsts()`, et surtout un `AddEnvironmentVariables()` absent, qui faisait **ignorer en silence** toute la configuration Docker.
- **Tester contre le vrai contenu.** Plutôt que de simuler des questions, la pile complète tourne en docker-compose et tire le contenu réel du siège. Un mock donne l'illusion d'un jalon atteint. Le vrai contenu, lui, révèle les vrais bugs de migration.
- **Le ménage.** Les applications Avalonia, l'ancien pilote de spots, les applications console de test et le code commenté mort ont été supprimés. On est passé de **44 à 30 projets**.
- **Les tests sont passés de 31 à près de 2 000.** La CI GitHub Actions compile la solution, exécute les tests unitaires et les tests JavaScript des écrans (`node --test`), puis monte un **banc de validation de bout en bout** en Docker (MongoDB, Mosquitto, écrans, une instance siège et une instance agence), sur lequel tournent les tests de validation.

---

## Déployer sur des Raspberry Pi… sans s'y rendre

### D'Azure DevOps à GitHub, du zip au paquet Debian

Avec Azure DevOps, une release revenait à déposer un zip sur chaque machine, le décompresser, rétablir le bit d'exécution et écrire une unité systemd à la main. Le tout était orchestré par des pipelines qui lisaient des IP dans un fichier Excel, avec un agent auto-hébergé par agence et des jetons d'accès qui expiraient au mauvais moment. Ça tenait pour quelques salles, mais rien de tout cela n'était reproductible à l'échelle d'un réseau qui grandit.

Le code vit désormais sur GitHub, et chaque composant est un **paquet Debian versionné**. Un paquet n'active rien à l'installation : il prépare la configuration et fournit trois outils, `dq-cutover` pour basculer, `dq-rollback` pour revenir en arrière et `dq-doctor` pour un diagnostic qui se vérifie lui-même. La bascule est explicite et réversible.

- **Les services serveur tournent dans Docker.** Les binaires .NET sont publiés en *self-contained* `linux-arm64` par compilation croisée, puis embarqués avec un Dockerfile qui ne fait que copier. Le `docker build` se fait **nativement sur le Pi**, au premier démarrage : pas de QEMU, pas de buildx, rien à compiler sur la cible.
- **Les services matériels sont natifs.** Le kiosque, le buzzer (GPIO), l'audio (ALSA) et le DMX (FTDI) tournent en services systemd. Ils ont besoin d'un accès direct au matériel, et rien ne justifie de les conteneuriser.

### Des images « golden » par rôle

Les Pi sont produits à partir d'une **image par rôle** (pupitre, saloon, donut, serveur) générée avec `rpi-image-gen` : Debian 13 « Trixie » en arm64, une base minimale sans bureau, Wayland avec labwc, et des versions épinglées pour garantir la reproductibilité.

La configuration propre à chaque machine tient dans un fichier posé sur la **partition de boot, en FAT**, donc modifiable depuis n'importe quel PC Windows sans démarrer le Pi :

```ini
# /boot/firmware/donutquiz.conf
ROLE=desk                          # desk | saloon | donut | host
SERVER_URL=http://<ip-du-serveur>:5003
COLOR=caramel
```

Au premier démarrage, un service *oneshot* lit ce fichier et déduit tout ce qui peut l'être : le topic du buzzer à partir de la couleur, l'adresse du broker à partir de l'URL du serveur. Il génère les fichiers de configuration, recale le tactile sur l'écran de jeu, fixe le nom d'hôte et active les services. Sur le terrain, la procédure se résume à : **flasher, remplir trois clés, brancher.**

Le paquet kiosque, enfin, est **le même pour l'ancien X11 et le nouveau Wayland.** Il détecte le backend vidéo à l'exécution :

| Action | X11 (Buster) | Wayland (Trixie) |
|---|---|---|
| Lister les sorties | `xrandr` | `wlr-randr` |
| Placer une fenêtre par écran | `wmctrl` | `<windowRules>` de labwc |
| Recaler le tactile | `xinput map-to-output` | `<touch mapToOutput>` de labwc |

### Passer de 32 à 64 bits, à distance, sans brique

Restait un problème de taille : tout le parc tournait en **Buster armhf (32 bits)**. Or un `dist-upgrade` ne change pas l'architecture, et une mise à niveau majeure en place de Raspberry Pi OS n'est pas supportée. Il fallait donc reflasher… des dizaines de cartes SD réparties dans toute la France.

J'ai écrit pour cela `dq-migrer-ab`, un script qui migre un Pi **à distance, en schéma A/B** :

1. **Sonde** (en lecture seule) : rôle, adresses, état de la carte SD, qui ouvre réellement le dongle DMX, et génération du `donutquiz.conf` ;
2. **Réduction** : un démarrage unique sur un initramfs, via `tryboot`, réduit la partition racine de Buster et crée deux partitions derrière elle, `p3` (boot Trixie) et `p4` (racine Trixie) ;
3. **Écriture** de l'image dans `p3`/`p4` *depuis Buster en marche*, avec vérification sha256 ;
4. **Préparation** : configuration, nouvelles clés d'hôte SSH, médias, identité VPN ;
5. **Essai** : un redémarrage d'essai sur Trixie. Tant que la bascule n'est pas figée, **un redémarrage normal, ou même une coupure de courant, ramène sur Buster** ;
6. **Validation** : une commande séparée, jamais enchaînée automatiquement, fige Trixie comme partition de démarrage.

La seule fenêtre à risque est la réduction de la partition, qui dure quelques minutes. C'est pourquoi la santé de la carte SD est contrôlée avant : une carte fatiguée qui ne revient pas de l'initramfs m'a appris à le faire. L'accès à distance passe par un VPN maillé (Headscale, le serveur de contrôle Tailscale auto-hébergé), qui remplace l'ancien outil de prise en main à distance.

---

## Ce que je retiens

- **Choisir des versions LTS.** Toujours, dans un produit qui doit durer.
- **Des clients légers et une seule source de vérité.** C'est ce qui a rendu possible le remplacement d'Avalonia sans toucher au serveur de jeu.
- **Un protocole vaut mieux qu'un framework** quand des équipements hétérogènes doivent se parler. MQTT a remplacé Akka.NET et le Hello Service, et il a ouvert la porte à Python et à l'ESP32.
- **« Connecté » n'est pas « vivant ».** Il faut mesurer un aller-retour de bout en bout plutôt qu'un indicateur intermédiaire, et se méfier des tests verts qui valident un mécanisme plutôt qu'un résultat.
- **Le logiciel en conteneur, le matériel en natif.**
- **Mettre la configuration terrain là où un technicien peut l'éditer**, c'est-à-dire sur une partition FAT, pas dans un JSON à modifier en SSH.
- **Garder chaque étape réversible** : rollback des paquets, démarrage d'essai A/B.
- **Reproduire avant d'améliorer.** Une migration réussie est d'abord une migration que personne ne remarque.

---

## Et aujourd'hui ?

La salle tourne. La plupart des agences ont basculé sur la nouvelle pile, les parties s'enchaînent, les écrans se reconnectent seuls, et une intervention qui demandait un déplacement se règle désormais à distance. Surtout, les retours des joueurs sont là :

<!-- Captures d'écran des avis Google -->
![Avis Google — 1](./images/avis-google-1.png)
![Avis Google — 2](./images/avis-google-2.png)
![Avis Google — 3](./images/avis-google-3.png)

La suite est déjà sur la table : des buzzers sur ESP32, une image unique pour tous les rôles, résolue par `donutquiz.conf`, et toujours plus d'outils pour que le siège et les franchisés gèrent leur contenu sans moi. Mais la salle n'est plus un chantier permanent, et c'était tout l'objectif de cette refonte.
