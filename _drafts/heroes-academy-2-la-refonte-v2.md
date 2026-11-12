---
title: "Heroes Academy (2/3) : la refonte V2 pour le Canada"
description: "Fin d'Avalonia, fin de MongoDB côté salles, logique séparée de l'affichage et du son, matériel décrit en base : l'architecture de la V2 de Heroes Academy, et les bugs qu'elle n'a pas magiquement résolus."
lang: fr
tags: [dotnet, csharp, mqtt, raspberry-pi, iot, architecture, retour-d-experience]
---

# Heroes Academy (2/3) : la refonte V2 pour le Canada

> **Série Heroes Academy**
> 1. [Concevoir le logiciel d'une salle de jeu physique](./heroes-academy-1-la-v1.md)
> 2. **La refonte V2 pour le Canada** (vous êtes ici)
> 3. [Huit mois de développement assisté par IA](./heroes-academy-3-developper-avec-l-ia.md)

Dans la [première partie](./heroes-academy-1-la-v1.md), j'ai raconté la V1 de Heroes Academy : une application Avalonia par Raspberry Pi, MongoDB en guise de bus d'intégration, un processus zombie qui imposait de redémarrer toutes les salles après chaque redémarrage du serveur, et un déploiement fait à la main.

Cette V1 a fait tourner Lille et Tarbes pendant des années. Mais l'ouverture de Brossard, près de Montréal, a servi de déclencheur. Installer une nouvelle copie de la V1 à plus de 5 000 km, sans pouvoir passer redémarrer un Pi le matin, n'était pas envisageable. J'ai donc entamé une refonte complète en septembre 2025, et la V2 a été installée au Canada début 2026, avec trois objectifs :

1. **Plus évolutif** : ajouter une salle, un mode de jeu ou un site sans dupliquer de code.
2. **Plus stable** : aucune salle ne doit dépendre d'une connexion qui ne sait pas se reconstruire.
3. **Moins lourd à maintenir** : fin d'Avalonia, fin du déploiement à la main, et une seule source de vérité pour la configuration.

---

## L'architecture

```text
┌───────────────────────── Serveur central (Ubuntu 24.04) ─────────────────────────┐
│  PostgreSQL ◄── WebApi (.NET 8, EF Core, MediatR)   SessionsOrchestrator (.NET 8) │
│                    ▲                                        ▲                     │
│  Admin UI / Kiosk UI (HTML + Web Components + mqtt.js) ─────┤                     │
│                    │ HTTP                                   │ MQTT                │
│                    │                     Mosquitto central ◄┘                     │
└────────────────────┼────────────────────────────▲─────────────────────────────────┘
                     │                            │
┌────────────────────┼──── Raspberry Pi d'une salle (Debian, arm64) ───────────────┐
│                    ▼                            ▼                                 │
│   RoomOrchestrator (.NET 8) ── logique de jeu, un IGame par processus             │
│        │  JSON /ui            │ JSON /audio            │ /io/input, /io/output    │
│        ▼                      ▼                        ▼                          │
│   GameUI (Python, mpv)   Lecteur audio MQTT     Mosquitto local ◄─► cartes E/S    │
│                                                        ▲                          │
│                                     ElectronicsBridge ─┘ (relais vers le central) │
└───────────────────────────────────────────────────────────────────────────────────┘
```

Sur le serveur central :

- **PostgreSQL** remplace MongoDB ;
- une **WebApi** .NET 8 (EF Core, Npgsql, MediatR) expose la configuration et enregistre les sessions et les scores ;
- un **SessionsOrchestrator** fait avancer les équipes de salle en salle.

Dans chaque salle, un **RoomOrchestrator** .NET 8 porte la logique d'un seul jeu, choisi par configuration au démarrage. Deux processus séparés l'accompagnent : l'**affichage** et le **son**.

Une chose n'a pas changé : l'électronique parle toujours MQTT avec des messages comme `/c0m2p5/on`. C'était la meilleure décision de la V1, je l'ai gardée telle quelle.

---

## Six principes

### Principe 1 : les salles ne parlent jamais à la base

C'est la leçon directe du processus zombie. Un Pi ne connaît que deux choses :

- **HTTP** pour récupérer ses réglages au démarrage : durées, câblage, textes, vidéos. Les réglages sont **mis en cache localement**, avec écriture atomique et copie de secours, si bien qu'une salle démarre même si l'API est injoignable.
- **MQTT** pour tout ce qui se passe pendant le jeu, avec une convention de topics stricte :

```text
/games/ha                          bus de session (salle terminée, démarrer la suivante…)
/games/ha/rooms/{salle}            commandes vers l'orchestrateur de salle
/games/ha/rooms/{salle}/ui         écrans
/games/ha/rooms/{salle}/audio      son
/games/ha/rooms/{salle}/notifs     scores, persistés côté API
```

Le serveur peut redémarrer : un client MQTT bien écrit se reconnecte. J'expliquerai plus bas pourquoi « bien écrit » est la partie difficile.

### Principe 2 : séparer la logique, l'image et le son

En V1, un crash du lecteur vidéo emportait la logique de jeu avec lui. En V2, l'écran est un **terminal passif**. Il reçoit des commandes JSON et les affiche, sans rien savoir des règles :

```json
{ "command": "start_game",
  "room": "snake",
  "datas": { "lang": "fr", "team": "Les Invincibles", "players_count": 5, "duration": 450 } }

{ "command": "score_update",
  "datas": { "remaining_time": 212, "score": 340, "score_diff": 20, "is_penalty": false } }
```

Le GameUI, qui remplace Avalonia, est une application Python légère (Tkinter et libmpv pour la vidéo). Elle est packagée en `.deb` et lancée comme service systemd *utilisateur* sous Wayland. Elle réutilise des modules Python (logging, MQTT, client d'API) déjà éprouvés sur un autre de nos produits.

Le chrono tourne localement pour rester fluide, et l'orchestrateur le **resynchronise** à des moments clés : mi-temps, 60 s, 30 s, 10 s, et à chaque mise à jour du score. Le son vit lui aussi dans son propre processus, un lecteur audio piloté par MQTT avec gestion du ducking et des pistes par langue.

Chaque brique peut redémarrer indépendamment des autres. Et la logique de jeu, en C#, est redevenue testable sans écran.

### Principe 3 : le matériel est une donnée

En V2, « le bouton n°3 de la salle Ropes est sur `c0m2p5` » est une **ligne en base**, servie par l'API. Ce n'est plus un fichier JSON recopié sur chaque Pi. Le même binaire tourne sur tous les sites, et les différences physiques sont décrites dans les données.

Le Snake de Brossard illustre bien l'intérêt. En France, le damier est un sol de 8 × 8 câblé sur des cartes d'E/S. À Brossard, c'est un damier de **8 × 10 dalles** d'un autre fabricant, branché en **USB/FTDI**, avec un agent dédié qui publie `{x, y, state}` sur MQTT. Côté jeu, il a suffi d'un adaptateur et d'une taille de damier rendue administrable par site.

### Principe 4 : un jeu est une machine à étapes

Tous les jeux héritent d'un `BaseGame` qui déroule une machine à étapes. On y trouve l'accueil, les consignes audio, le démarrage, la mi-temps, la sirène, le compte à rebours, la question bonus, la révélation et les résultats. Le tout est piloté par un seul timer, protégé par un compteur de génération pour qu'une salle terminée ne puisse pas relancer son propre timer.

Le gros du comportement est mutualisé. Une salle de jeu « simple » tient donc en quelques lignes :

```csharp
// Ropes : grimper pour appuyer en même temps sur les 2 cibles rouges qui s'allument.
public sealed class RopesGame : BasePairSequentialGame<GenericCustomSettingsApiDto>
{
    public RopesGame(IRoomSettingsService settingsService, ISessionDataService sessionDataService,
        ISecondaryMissionGame secondaryMissionGame, IMqttPublisherFactory publisherFactory,
        ILogger<RopesGame> logger, IGameCustomSettingsHttpDataService customSettingsHttpDataService)
        : base(RoomsEnum.Ropes, settingsService, sessionDataService, secondaryMissionGame,
               publisherFactory, logger, customSettingsHttpDataService)
    { }
}
```

Les salles plus riches, comme Balls ou Snake, ne surchargent que ce qui les distingue.

### Principe 5 : tout doit pouvoir tourner sans le matériel

Je ne pouvais pas déboguer le Canada depuis la France avec un tournevis. Il fallait donc pouvoir reproduire le site sans le moindre bouton physique.

Le premier outil, présent dès le début de la refonte, est **une salle complète dans Docker**. L'orchestrateur, le GameUI et le lecteur audio tournent dans un conteneur, l'écran est exposé en noVNC et le son est renvoyé au navigateur par un sink PulseAudio. Le son dans un conteneur a d'ailleurs résisté plus longtemps que le reste, jusqu'à une bascule de PipeWire vers PulseAudio pur. Comme l'électronique parle MQTT, une salle se pilote ensuite à coups de `mosquitto_pub`.

D'autres outils sont venus après l'ouverture :

- une **UI de test dans le navigateur** : elle dessine la salle à partir de l'API (boutons, LEDs, dalles, portes) et chaque clic publie sur `/io/input`, exactement comme le ferait une carte ;
- un **robot joueur** en ligne de commande, avec taux d'erreur et temps de réaction réglables, pour les jeux les plus complexes ;
- surtout, une **suite de tests automatisés**. Le temps y est une dépendance injectée (`TimeProvider`), donc on teste un jeu de 7 minutes en quelques millisecondes. Elle comptait une dizaine de tests à l'ouverture du Canada et en compte plus de 800 aujourd'hui. Je reviens sur cette évolution dans la [troisième partie](./heroes-academy-3-developper-avec-l-ia.md).

### Principe 6 : le déploiement est une fonctionnalité

Brossard a été installé avec des procédures documentées salle par salle : paquets `.deb`, services systemd, une configuration par machine. C'était déjà un net progrès sur le zip de la V1, mais ça restait long et sujet aux oublis.

Depuis l'été 2026, un **outil de déploiement en une commande** a pris le relais. Un fichier d'inventaire décrit le site, avec une ligne par machine :

```text
"Snake|pi-snake|snake|gameui,audio,iostatus,orchestrator,bridge,mosquitto|"
```

Le reste suit :

```bash
./deploy.sh -i inventory/<agence>.env preflight            # SSH, sudo, architecture
./deploy.sh -i inventory/<agence>.env --dry-run install    # répétition à blanc
./deploy.sh -i inventory/<agence>.env install              # serveur, puis salles en parallèle
./deploy.sh -i inventory/<agence>.env --limit snake update # une seule salle
./deploy.sh -i inventory/<agence>.env verify               # état de tous les services
```

Les binaires .NET sont publiés en self-contained : aucun runtime n'est à installer sur les Pi. Toute la configuration est générée depuis des templates, et rien n'est édité à la main sur les machines. Une contrainte reste incompressible : les machines sont sur un LAN privé, donc le déploiement se lance depuis un poste branché sur le réseau du site.

---

## Un code, deux modes de jeu : la migration de Lille

Brossard n'a pas d'Arène : la question bonus est posée à la fin de chaque salle. Lille, en revanche, tient à la sienne. Quand Lille a migré de la V1 à la V2 en septembre 2026, il fallait donc que la V2 sache jouer les deux modes : question en fin de salle, ou passage de 45 secondes dans l'Arène (plus une marge de sécurité) entre deux salles de jeu.

Un site ne change jamais de mode. J'ai donc choisi d'en faire une **constante de compilation** plutôt qu'un réglage en base : `dotnet publish -p:GameMode=Arena`. Les artefacts sont estampillés avec leur mode, et l'outil de déploiement refuse de mélanger les binaires de deux modes sur un même site. Une configuration impossible à modifier par erreur est une configuration qu'on n'a pas à surveiller.

L'Arène a aussi ramené un vieux problème de la V1 : deux équipes ne doivent jamais s'y croiser. En V2, ça s'est traduit par une identité de visite (`RoomSessionId`) portée de bout en bout, de la planification aux messages MQTT, et par des tests de non-régression sur les croisements.

Tarbes tourne encore sur la V1, stabilisée par le watchdog décrit dans la première partie.

---

## Exploiter à distance

Pour un site à six heures de décalage, l'administration doit permettre d'intervenir sans être sur place :

- redémarrer un Pi ou un service depuis l'interface, via SSH ;
- forcer la fin d'une session bloquée, avec une journalisation de chaque action du Game Master ;
- consulter les caméras ;
- parler dans une salle en push-to-talk : le flux PCM passe par WebSocket jusqu'au lecteur audio de la salle, avec un seul orateur à la fois.

Pour les statistiques, un outil d'analyse à part lit les dumps PostgreSQL et produit des KPIs et des graphiques. Il repère par exemple les sessions terminées de force.

L'interface d'administration est volontairement sobre : HTML statique, Web Components natifs et mqtt.js en WebSocket, sans étape de build.

---

## Ce que la V2 n'a pas magiquement résolu

Une refonte ne supprime pas les bugs, elle en change la nature. Voici ceux qui m'ont le plus appris.

### Reconnecté, mais sourd

C'est l'histoire la plus instructive de la V2. Sur le site canadien, l'exploitant subissait des pertes de connexion MQTT régulières qui rendaient la salle injouable. Le `SessionsOrchestrator` se reconnectait pourtant bien au broker. Simplement, après la reconnexion, **il n'entendait plus rien**.

En creusant, j'ai trouvé une collection de défauts qui se renforçaient mutuellement :

1. **Aucune resouscription.** La souscription à `/games/ha` était faite une seule fois, au démarrage. Or chaque reconnexion ouvrait une session broker neuve (*clean session*) : le client revenait connecté, mais sans aucun abonnement.
2. **Un ClientId régénéré à chaque tentative** (`ClientId + "-" + Guid.NewGuid()`), qui multipliait les sessions fantômes côté broker.
3. **Une ré-entrance.** MQTTnet lève `DisconnectedAsync` même quand un `ConnectAsync` échoue. Le handler de déconnexion relançait une connexion, qui attendait un verrou détenu par la tentative en cours d'échec.
4. **Une faute de frappe.** Le code lisait `"Mqtt:Post"` au lieu de `"Mqtt:Port"`, si bien que le port configuré était ignoré.
5. **Aucun test** sur ce service.

La correction a été spécifiée avant d'être codée, avec une vingtaine de critères d'acceptation. Le principe central est **une seule autorité de reconnexion**. Le handler de déconnexion se contente de réveiller une boucle unique, qui se reconnecte, resouscrit à tout et attend la prochaine perte :

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    // Le signal est armé AVANT la tentative : une déconnexion notifiée
    // pendant ou juste après la connexion ne peut pas être perdue.
    var signal = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
    Volatile.Write(ref _disconnectedSignal, signal);

    if (await TryConnectAsync(port, stoppingToken))
    {
        delay = _minDelay;
        await ResubscribeAllAsync(stoppingToken);          // à CHAQUE connexion
        await signal.Task.WaitAsync(stoppingToken);        // attendre la prochaine perte
    }

    await _delayProvider.DelayAsync(ApplyJitter(delay), stoppingToken); // 1 s → 30 s, avec jitter
    delay = IncreaseDelay(delay);
}
```

Le comportement attendu est le suivant :

- un backoff exponentiel borné, avec jitter pour que tous les services ne se ruent pas sur le broker au même instant ;
- un ClientId stable pour toute la vie du processus ;
- une temporisation abstraite derrière une interface, pour que les tests soient déterministes.

Le livrable a été construit sur une base alignée avec la version en production, et non sur `main`, pour ne livrer que ce correctif.

### L'écran qui se figeait

Côté GameUI, un écran restait parfois figé en pleine partie, sans plus aucune mise à jour du score. La cause tenait dans un seul appel.

La bibliothèque MQTT Python (paho) appelle `on_message` sur **son propre thread réseau**, celui-là même qui doit envoyer le PINGREQ de keep-alive. Or on y construisait l'écran suivant, et charger un fond d'écran prend à lui seul environ une seconde. Pendant ce temps le keep-alive ne partait plus, le broker considérait le client comme mort et le déconnectait (`rc: 7`).

La correction consiste à confier le travail au thread Tk :

```python
def _show_screen_on_ui(self, screen_id, data=None):
    # Jamais de travail lourd sur le thread réseau de paho : on délègue au thread Tk,
    # seul thread autorisé à toucher les widgets de toute façon.
    self.ui.root.after(0, lambda: self.ui.show_screen(screen_id, data))
```

C'est une règle connue de toutes les UI : on ne touche pas aux widgets hors du thread UI. Elle se double ici d'une règle réseau : on ne bloque jamais le thread qui maintient la connexion.

### 20 heures à Montréal

Bug signalé depuis le Canada : « les sessions disparaissent du tableau de bord tous les soirs vers 20 h ». La requête filtrait les sessions sur le **jour calendaire UTC**. À Montréal, en heure d'été, UTC−4, ce jour bascule à 20 h locales.

La correction a remplacé le jour calendaire par une fenêtre glissante de 24 heures. Mais la vraie leçon, c'est qu'un logiciel conçu en France n'avait jamais eu à se demander *quel jour on est*. Depuis, tous les outils d'analyse travaillent explicitement en `America/Toronto`.

### Concurrence, matériel et autres réjouissances

- **Une réponse ignorée.** Une réponse à la question bonus arrivant pendant la fenêtre de révélation était perdue à cause d'un verrou d'état, remplacé depuis par des opérations `Interlocked`.
- **Les paniers de Balls**, comptés simultanément, ont eu besoin d'un verrou par panier.
- **Un blocage dans l'Arène.** Une équipe arrivant pendant la fin de salle de la précédente écrasait un identifiant de session partagé, et la session restait bloquée.
- **Des sessions « fantômes »** jamais démarrées décalaient le calcul des créneaux suivants dans le passé. Il a fallu les exclure explicitement, puis ajouter une détection des sessions en souffrance et une fin forcée propre.
- **Des impulsions parasites** sur les capteurs de sol de Don't Touch The Floor ont imposé un anti-rebond paramétrable. Sur les panneaux à aimants de la salle Communication, les transitions à moins de 200 ms d'écart sont ignorées.
- **Le démarrage sous Wayland.** Le GameUI démarrait parfois avant le compositeur graphique. Le service attend désormais l'apparition du socket Wayland avant de se lancer.

---

## Et maintenant : Le Standard

La plateforme étant stable, j'ai pu revenir à ce qui compte pour les joueurs : le jeu. Le dernier chantier en date est **Le Standard**, un nouveau gameplay pour la salle Communication. Au moment où j'écris ces lignes, il est en test sur site, à quelques jours de sa mise en production.

Un joueur devient standardiste et lit des appels sur une console, que les autres doivent brancher sur des panneaux à aimants. Le moteur de jeu est **pur** : pas de MQTT, horloge injectée. Toutes les phases (rebranchement, standard, relève, surcharge, coupure) se testent donc comme des fonctions.

Ce chantier m'a aussi rappelé qu'on conçoit pour des humains. Les joueurs ont pris l'habitude de **sortir de la salle dès qu'un gyrophare s'allume**. Le Standard n'en utilise donc aucun pendant la partie : tout passe par le son et l'écran. Et la « coupure générale » de fin de partie, qui fait tomber les panneaux à aimants après un « Reculez ! 3-2-1 », est désactivée par défaut. Elle ne s'active qu'après validation de la sécurité dans la salle.

---

## Ce que la V2 m'a appris

1. **Séparez ce qui plante de ce qui compte.** L'affichage et le son peuvent redémarrer, la logique de jeu doit continuer.
2. **Le matériel est une donnée.** Un câblage dans le code, c'est un binaire par site.
3. **Le temps est une dépendance.** Pour les tests (`TimeProvider`) comme pour la production (fuseaux horaires, resynchronisation des chronos).
4. **Une reconnexion se teste.** « Ça se reconnecte » ne veut rien dire tant qu'on n'a pas vérifié que ça réécoute.
5. **Le déploiement fait partie du produit.** Une commande, une répétition à blanc, une vérification : c'est ce qui permet d'opérer un site de l'autre côté de l'Atlantique.
6. **On conçoit pour des humains.** Des joueurs qui fuient les gyrophares, un Game Master qui doit relancer une salle sans terminal : l'architecture doit servir ces gens-là.

---

## Le verdict des joueurs

Après des mois de travail, de nuits de déploiement et de debug à distance, la V2 tient ses promesses : les salles tournent, l'exploitation est sereine, et surtout, les joueurs en ressortent ravis. Je les laisse conclure.

<!-- Captures d'écran des avis Google à insérer ici -->

![Avis Google](./images/avis-google-1.png)

![Avis Google](./images/avis-google-2.png)

---

> **Suite de la série**
> 3. [Huit mois de développement assisté par IA](./heroes-academy-3-developper-avec-l-ia.md) : à partir de mars 2026, l'IA a pris une place croissante dans le développement de la V2. Ce que l'historique Git en dit, chiffres à l'appui.
>
> Pour (re)lire le début : [1. Concevoir le logiciel d'une salle de jeu physique](./heroes-academy-1-la-v1.md).

*Les extraits de code de cet article sont simplifiés. Les adresses, identifiants et détails d'infrastructure propres à chaque site ont été volontairement omis.*
