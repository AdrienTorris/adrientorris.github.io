---
title: "Des escape games pilotés par MQTT : comment je développe chez Team Break"
date: 2026-11-12
lang: fr
tags: [mqtt, architecture, iot, raspberry-pi, esp32, linux, debian, dotnet, systemd, systèmes-distribués]
---

> **Titres alternatifs** (à choisir, puis supprimer ce bloc)
>
> 1. Six joueurs, un chrono, cent cinquante machines : l'informatique derrière nos escape games
> 2. MQTT, Raspberry Pi et électroaimants : dans les coulisses techniques de Team Break
> 3. Développer pour des salles qui ont un corps : agents, messages et Linux chez Team Break

# Des escape games pilotés par MQTT : comment je développe chez Team Break

Chez nous, quand un bug survient, il y a six personnes enfermées dans une pièce, un chrono qui tourne, et une trappe qui refuse de s'ouvrir.

Je développe les logiciels des salles d'escape game de Team Break. Ce n'est pas une application web derrière un load balancer, c'est un système distribué dont les nœuds sont des mini-PC, des Raspberry Pi, des ESP32, des Arduino, des tablettes Android, des téléviseurs, des projecteurs, des lyres DMX, des électroaimants et des machines à fumée. Il tourne sans surveillance sur treize sites. Il doit survivre aux coupures de courant, aux câbles réseau abîmés et aux joueurs qui tapent un peu trop fort sur les accessoires. Et il ne doit jamais, au grand jamais, mettre quelqu'un en danger.

Dans cet article, je présente les choix techniques qui en découlent : architecture décentralisée, MQTT comme colonne vertébrale, une armée d'agents spécialisés, de l'électronique maison, une base Linux hétérogène et la façon d'exploiter tout ça à distance.

## Une salle, c'est un système distribué

Commençons par l'unité de base : une salle. Sur le papier, c'est 60 minutes de jeu. Dans les faits, c'est un petit datacenter qui tient dans un placard technique.

```
                        ┌───────────────────────────────────────────┐
                        │   Serveur central de la salle (mini-PC)   │
                        │   Debian 13 + Docker                      │
                        │                                           │
                        │  Mosquitto · Room/Game Orchestrator       │
                        │  GM API · GM UI · Agents API · PostgreSQL │
                        │  Electronic · Emergency · Healthcheck     │
                        └──────────────────┬────────────────────────┘
                                           │  MQTT (LAN, hors ligne)
      ┌──────────────────┬─────────────────┼──────────────────┬───────────────────┐
      │                  │                 │                  │                   │
 Raspberry Pi       Raspberry Pi       ESP32 / Kincony    Arduino + W5500    Tablettes, écrans
 vidéo (mpv)        audio, DMX         capteurs, relais   trappes, LEDs      kiosques web
 labwc / KMS        agents de jeu      RFID, digicodes    séquenceurs        (Chromium, Android)
```

Un serveur central, en général un mini-PC Intel N95 ou N100, fait tourner la pile Maestro, notre moteur d'orchestration, dans Docker. Autour de lui gravitent des machines de périphérie (*edge points*) qui ont chacune un rôle très précis : lire une vidéo sur un téléviseur, jouer une ambiance sonore, envoyer des trames DMX à un projecteur, lire un badge RFID dans un vase canope, compter les tours d'une manivelle.

Un exemple concret : *La Chocolaterie*, à Lille. C'est une expérience découpée en cinq salles logicielles, avec seize Raspberry Pi et un serveur central. Son *gameflow* (j'y reviens plus bas) compte 68 fichiers, 588 commandes, 247 gestionnaires de notifications et 312 points d'entrée-sortie électroniques. Un scan ARP du réseau des salles de Lille remonte 69 appareils.

À l'échelle du parc, cela donne environ 150 machines enrôlées sur notre VPN, réparties sur 13 sites (en propre et en franchise), et une cible autour de 300 une fois les anciennes salles migrées. Le parc héberge aussi deux autres produits maison (Donut Quiz et Heroes Academy) et notre plateforme d'exploitation de site, Archipel.

Trois contraintes structurent tout le reste :

1. **Une salle doit fonctionner sans Internet.** Le réseau du site peut tomber, le VPS peut être en maintenance : la partie en cours ne doit pas s'en apercevoir.
2. **Tout redémarre, tout le temps.** Coupures de courant (à Aéroville, celles du matin viennent des exercices d'évacuation du centre commercial), téléviseurs qui se mettent en veille, cartes son qui disparaissent du bus.
3. **La sécurité des personnes ne dépend pas du logiciel.** Les ventouses des portes restent câblées côté SSI (système de sécurité incendie). Le logiciel suit, il ne décide pas.

## MQTT, le système nerveux

Toutes ces machines se parlent via MQTT, avec un broker Mosquitto qui tourne sur le serveur central de chaque salle.

Pourquoi MQTT plutôt que du REST, du gRPC ou un bus d'entreprise ? Parce qu'il faut un protocole que tout le monde sache parler, depuis un Arduino Uno avec 2 Ko de RAM jusqu'à un service .NET en passant par un script Python sur un Pi et une page web dans un kiosque. Le pub/sub découple naturellement les producteurs des consommateurs : le capteur du digicode n'a pas à savoir qui attend son signal. Et MQTT apporte gratuitement deux primitives précieuses dans notre contexte : les messages *retained* (le dernier état connu est servi à tout nouvel abonné) et le *Last Will* (le broker annonce la mort d'un client à sa place).

### Une convention de topics

Chaque agent a sa boîte aux lettres, construite de façon déterministe à partir du jeu, de la salle et du type d'agent :

```csharp
private static string GetBaseTopicName(GameAgentEnum agent, string game, string? room) =>
    !string.IsNullOrWhiteSpace(room) && agent != GameAgentEnum.GameOrchestrator
    ? $"/games/{game.ToLower()}/rooms/{room.ToLower()}/"
    : $"/games/{game.ToLower()}/";
// suffixes : roomorchestrator, audioplayer, videoplayer, electronics,
//            lights, dmx, customagents, gamemastering, gameorchestrator
```

On obtient des topics comme `/games/chocolaterie/rooms/elevator/roomorchestrator` ou `/games/chocolaterie/rooms/candy_store/videoplayer`. L'orchestrateur de jeu, qui enchaîne les salles, vit un niveau au-dessus : `/games/chocolaterie/gameorchestrator`. À côté, quelques topics transverses : la présence des clients (`/monitoring/clients/status`), un instantané de santé *retained* par salle, et le verrou d'urgence.

### Le message est le contrat

Les messages sont du JSON en snake_case. Une notification destinée à l'orchestrateur de salle ressemble à ceci :

```json
{
  "key": "stop_room",
  "sender_id": "chocolaterie-game-orchestrator",
  "sender_type_id": 8,
  "room": "elevator",
  "datas": null,
  "type_id": 3,
  "sent_at": null
}
```

Le `sender_type_id` est un entier : 1 pour l'orchestrateur de salle, 2 pour le lecteur audio, 3 pour l'agent électronique, 4 pour la vidéo, 11 pour le DMX… Ces entiers voyagent sur le fil et sont relus par des agents qui n'ont pas forcément été mis à jour en même temps. La règle est donc stricte : **on ajoute des valeurs à la fin, on ne renumérote jamais**. Le commentaire est dans le code, au-dessus de l'énumération.

Cette discipline se retrouve partout. Côté Archipel, des tests (`WireFormatTests`) figent le format JSON échangé avec les salles, parce qu'*une salle Maestro peut tourner longtemps sur une vieille version de paquet*. Sur notre projet le plus récent, le protocole est en Protobuf, avec la même règle écrite noir sur blanc dans un ADR : on ajoute des champs, on n'en supprime ni n'en renumérote jamais.

### QoS : distinguer le jeu de la sécurité

Les événements de jeu partent en QoS 0, sans *retain*. C'est du *fire-and-forget* assumé : un capteur qui retombe renverra son état, et un message de jeu « en retard » n'a plus de valeur de toute façon.

Les chemins critiques sont traités différemment. L'agent d'urgence, qui relaie le SSI, et l'agent de santé publient en **QoS 1 et en retained**. Quand une alarme incendie se déclenche, le topic `/games/<jeu>/rooms/<salle>/emergency` passe à l'état verrouillé et y reste. Un lecteur audio qui redémarre pendant l'alarme se réabonne, reçoit immédiatement ce message *retained*, et revient muet. Pas de course entre le démarrage et l'alarme : l'état est dans le broker, pas dans la mémoire d'un processus.

Autre choix délibéré : **pas de requête-réponse sur MQTT**. Quand le maître du jeu (GM) demande de préparer une salle, la GM UI passe l'état à « préparation demandée » en base, l'orchestrateur exécute puis confirme via HTTP auprès de l'Agents API (adossée à PostgreSQL). La machine à états d'une salle repose donc sur des poignées de main *demandé → confirmé*, et la source de vérité est une base de données, pas un message qui aurait pu se perdre.

Avec une exception assumée : pour le briefing, la fin de préparation, le démarrage et l'arrêt lancés depuis la console, l'API confirme l'état en base de façon optimiste, sans attendre l'orchestrateur. Une salle bloquée en « démarrage demandé » avec six joueurs à l'intérieur est pire qu'un état momentanément en avance sur la réalité. La préparation et la fin de partie, elles, restent confirmées par l'orchestrateur.

### Ce que MQTT m'a appris à la dure

- **Le keepalive.** Nos pages web de salle utilisaient un keepalive de 5000 secondes. J'ai monté un banc de test (broker aedes, coupure Wi-Fi simulée) qui a montré qu'une coupure silencieuse passait inaperçue pendant **environ 2 h 05**. Avec 60 s, elle est détectée en 90 s. Le backoff de reconnexion, autrefois fixé à 100 ms, est maintenant progressif.
- **L'agent devenu sourd.** Un agent .NET qui se reconnectait ne se réannonçait pas sur le topic de présence. Il était bien connecté, mais la supervision ne le voyait plus. Désormais, chaque (re)connexion republie la présence. La première connexion ne renonce jamais (backoff `1 s, 2 s, 5 s, 10 s, 30 s, 60 s`) et le chronomètre des tentatives utilise une **horloge monotone**, car un Pi n'a pas d'horloge matérielle et son heure saute à la synchronisation NTP.
- **Le hostname, cette clé primaire cachée.** Lors de l'enrôlement des machines dans le VPN, notre outil renommait le hostname. Sur le moment, rien. Mais au redémarrage suivant, des salles se sont retrouvées hors service, et pendant deux jours : salle bloquée en « préparation demandée », vidéos présentes mais aucun son. Deux causes, toutes deux différées au reboot. D'une part, nos agents MQTT appellent `socket.gethostname()` une fois au démarrage et ignorent les messages adressés à un autre nom (Maestro adresse certaines machines par hostname). D'autre part, le `SingletonLock` de Chromium contient le hostname, donc le kiosque refusait de démarrer, sans un mot. Depuis, le nom de nœud sur le VPN est découplé du hostname système.

## Des agents, beaucoup d'agents

Chez nous, un **agent** est un processus avec une responsabilité unique, qui ne parle que MQTT (et HTTP pour l'état durable), et qui ne connaît rien du scénario. Le lecteur vidéo sait lire un segment entre deux repères. Il ne sait pas qu'il montre un dinosaure.

### Le socle .NET

Les agents génériques de Maestro sont écrits en C# sur .NET 8, avec MediatR, EF Core et Npgsql sur PostgreSQL 16, Serilog et MQTTnet 5 :

| Agent | Rôle |
|---|---|
| Room Orchestrator | Charge et exécute le gameflow d'une salle : notifications, jalons, chrono |
| Game Orchestrator | Enchaîne les salles d'une même expérience, traduit les ordres du GM ou d'Archipel |
| GM API / GM UI | Console du maître du jeu, en Web Components natifs, sans framework |
| Agents API | API de données (PostgreSQL) pour tous les agents hors GM |
| Electronic Agent | Pont entre les brokers électroniques (panneaux d'E/S, ESP32, Kincony) et Maestro |
| Scheduled Actions | Actions différées ou conditionnelles, et le système d'indices automatique |
| Emergency Agent | Relais du SSI : ouvre les portes, allume, coupe le son, tient un verrou |
| Healthcheck Agent | Inspecte les conteneurs et les équipements, publie un instantané *retained* toutes les 10 s |

Une règle ne souffre aucune exception : **aucun code spécifique à un jeu dans le dépôt des agents génériques**. Chaque jeu qui a besoin de logique sur mesure a son propre dépôt. Pour la Chocolaterie, cela donne une douzaine d'agents : une Guitar Hero maison, une machine à pompes, un distributeur de bonbons, un champignon UV, et les écrans web associés.

Le code commun (messagerie MQTT, modèles, chargement du gameflow) est publié en paquets NuGet privés, versionnés indépendamment : `Messaging.Mqtt` en est à la version 1.0.146, `Models` à la 1.0.79. Une version de paquet qui n'a pas été incrémentée fait échouer un test unitaire. Le tout est couvert par environ 1 800 tests xUnit dans les agents et 500 de plus dans les bibliothèques d'infrastructure, y compris un test d'intégration sans mock qui traverse toute la chaîne : broker MQTT embarqué → pont électronique réel → endpoint SSE de la GM API → relais dans l'UI.

### Les agents « déportés » en Python

Sur les Raspberry Pi, les agents qui touchent au matériel sont en Python, empaquetés en `.deb` :

- **MQTTmpv** : lecteur vidéo sur libmpv, segments entre repères, overlays, picture-in-picture, pilotage CEC des téléviseurs, multi-écran en KMS.
- **MQTTaudioplayer** : lecteur multipiste (pygame-ce), fondus, *ducking*, groupes, streaming de longues ambiances, voix du GM en push-to-talk par WebSocket.
- **MQTTdmx** : trames DMX512 sur un adaptateur USB FTDI (clones Enttec Open DMX), boucles, fondus et blackout.
- Des agents plus petits : un moteur d'énigmes à séquence (jusqu'à un morse de 22 symboles en fenêtre glissante), un détecteur de mots-clés hors ligne (MFCC + DTW sur des banques de `.wav`), un gardien d'état pour des panneaux d'E/S tiers.

Ils partagent un module commun (sous-module git) qui fournit un client MQTT multi-broker avec reconnexion et rejeu des abonnements, la configuration, les logs et l'intégration `sd_notify` pour systemd.

Les commandes sont du JSON, avec des filtres optionnels (`PlayerID`, `hostname`, `ip`). Sans filtre, la commande est diffusée. Voici par exemple un segment vidéo qui prévient l'orchestrateur à son début et à sa fin :

```json
{
  "command": "segment",
  "data": {
    "start_cue": "MyCuePoint1", "end_cue": 7, "at_end": "pause",
    "at_start_topic": "/mon_topic/start", "at_start_message": "video demarree",
    "at_end_topic": "/mon_topic", "at_end_message": "video finie",
    "play": true
  }
}
```

## L'orchestration : le gameflow

Si les agents sont les muscles, le **gameflow** est le système nerveux central. C'est une description déclarative, en JSON, de tout ce qui se passe dans une salle : quelles notifications existent, quels gestionnaires y réagissent, quels comportements ils déclenchent (publier une commande, valider un jalon, modifier le temps restant, faire échouer la salle…), quelles actions sont planifiées, comment les indices sont distribués et comment les entrées-sorties électroniques sont mappées sur des clés métier.

Voici un extrait réel, légèrement élagué, de la Chocolaterie : un digicode sur ESP32 déverrouille une trappe et valide un jalon. Le GM peut obtenir le même effet depuis sa console.

```json
{
  "file_scope_settings": { "room": "ice_room", "electronic_proxies_format": "tb_esp32" },
  "notification_handlers": [ {
    "notifications": [ "digicode_unlocked", "milestone_state_change_r4_digicode_unlocked_validate" ],
    "behaviors": [
      { "behavior": "command_publish",    "key": "trappe_unlock" },
      { "behavior": "milestone_validate", "key": "r4_digicode_unlocked" }
    ]
  } ],
  "electronic_agent_proxies": [
    { "provider": "esp32_digicode", "type": "input",  "command_key": "digicode_unlocked",
      "label": "Digicode", "mod": "in2",  "pin": 0, "state": "off" },
    { "provider": "esp32_digicode", "type": "output", "command_key": "trappe_unlock",
      "label": "Trappe conduit", "mod": "out1", "pin": 0, "state": "on" }
  ]
}
```

Quelques choix qui méritent d'être expliqués :

- **Un dépôt Git par jeu *et par site*.** On pourrait croire qu'une même salle est identique à Lille et à Grenoble. Ce n'est jamais le cas : un aimant actif à l'état haut ici l'est à l'état bas là-bas. Plutôt que de multiplier les conditions, chaque site a son gameflow.
- **Un versionnage découplé de Git.** Un fichier `_versions.json`, en ajout seul, porte le numéro de version, la date, l'auteur et la description des modifications. Les agents comparent la date la plus récente à leur copie locale. Conséquence : on peut pousser du travail en cours sans qu'il parte en salle. Le gameflow de la Chocolaterie compte 644 entrées depuis juillet 2024.
- **Une copie locale, tirée au démarrage seulement.** Les agents récupèrent la dernière version au boot et travaillent ensuite sur leur copie, ce qui garantit le fonctionnement hors ligne. Comme plusieurs agents lisent le même répertoire, le chargement est sérialisé par un `flock` au niveau du système : exclusif pour celui qui télécharge, partagé pour les lecteurs, et relâché automatiquement en cas de crash ou de coupure.
- **Des scénarios superposables.** Une même salle physique peut jouer une variante (Halloween, version enfants) via un filtre `scenario` et des `override` sur les éléments concernés. Le scénario est une option de session, indépendante de la langue.
- **Des tests de salle versionnés avec le gameflow.** Un protocole de test (`_tests/*.json`), jamais lu par le jeu, est validé en CI et exécuté depuis une page « Tests » de la console GM : on y vérifie chacune des 312 entrées-sorties de la Chocolaterie.

Une anecdote pour montrer qu'un format déclaratif reste du code : pendant longtemps, `"filters_behavior": "or"` a été silencieusement évalué comme un ET. La cause : un constructeur qui affectait le paramètre au lieu de la propriété. Le bug a été corrigé dans la version 1.0.79 du paquet de modèles, avec le test qui allait avec.

## L'électronique : du C sur ESP32, et des leçons apprises à la dure

Les accessoires (les *props*) sont l'endroit où le logiciel rencontre le monde physique. Le standard maison est l'**ESP32** (WROOM, 4 Mo de flash) programmé en C avec **ESP-IDF**, sur FreeRTOS. Pour les besoins en relais, on utilise des cartes **Kincony KC868-A16** (ESP32, 16 relais et 16 entrées sur des expandeurs I2C PCF8574, Ethernet LAN8720) avec un firmware maison. On trouve aussi des Arduino Uno ou Mega avec shield Ethernet, et des panneaux d'E/S tiers à base de Raspberry Pi et de Node-RED.

### Un socle commun pour tous les firmwares

Chaque firmware d'accessoire inclut un socle partagé en sous-module git. L'accessoire n'ajoute que son brochage, sa logique de jeu et ses messages. Le socle compile avec ESP-IDF 5.5 et 6.0 et fournit une vingtaine de composants : réseau (Wi-Fi avec backoff, Ethernet, mDNS, NTP), client MQTT, serveur HTTP avec page de configuration et état en direct en SSE, stockage LittleFS, logs persistés en flash, watchdog, gestion unifiée des GPIO (broches internes ou expandeurs, anti-rebond, filtres), lecteurs RFID PN532 (jusqu'à 8 sur un même bus), rubans WS2812.

Quelques principes y sont gravés :

- **Le matériel avant le réseau.** L'ordre de démarrage est fixe et documenté. Les sorties sont initialisées *avant* le réseau, parce qu'une sortie doit atteindre son état sûr (serrures fermées, moteurs arrêtés) le plus tôt possible.
- **Une configuration décrite par des schémas.** Chaque composant déclare sa section de `tb_config.json` dans un schéma, et la page web génère ses formulaires à partir de ces schémas. En cas de fichier corrompu, le firmware retombe sur `last-known-good`, puis sur `factory-defaults`, et conserve le fichier fautif pour diagnostic.
- **Un mode sans échec.** Après deux démarrages ratés consécutifs, la carte démarre en mode dégradé : configuration restaurée, Wi-Fi et page web actifs, mais **pas de MQTT**, elle reste hors du jeu. Un démarrage n'est considéré comme réussi qu'après 60 minutes de fonctionnement : un crash en pleine session compte donc comme un échec. La décision est une fonction pure, testée unitairement sur la cible `linux` d'ESP-IDF.
- **Watchdog, core dump et OTA signée.** Watchdog de tâche de 10 s avec panic, core dump en flash, deux partitions OTA, mise à jour du firmware et de l'interface web par HTTP, avec un manifeste signé RSA-3072. Les builds sont reproductibles (`CONFIG_APP_REPRODUCIBLE_BUILD=y`) et publiés par la CI sur tag.
- **L'Ethernet quand c'est possible.** Une salle d'escape game est pleine de métal et d'autres accessoires qui émettent : le Wi-Fi n'y est pas toujours le bienvenu.

Les UID des badges RFID attendus sont stockés dans la configuration, pas dans le code. Remplacer une statuette perdue dans la salle égyptienne est un changement de configuration, pas un nouveau firmware.

Le protocole d'une carte Kincony reste volontairement simple :

```json
{"command":"set","data":[{"module":"out1","pin":2,"value":"on"},{"module":"out2","pin":3,"value":"off"}]}
{"command":"get","data":[{"module":"in1","pin":3}]}
{"command":"get_all_infos"}
```

### La carte qui répondait au ping

Mon histoire préférée. Les trappes de l'ascenseur de la Chocolaterie sont pilotées par un Arduino Uno avec un shield Ethernet W5500. De temps en temps, la carte se figeait : port série muet, plus de jeu… mais elle répondait toujours au ping. Le W5500 gère l'ICMP tout seul, il ne prouve donc absolument pas que l'ATmega est vivant. La bibliothèque Ethernet attendait le contrôleur dans des boucles sans timeout, et les parasites d'un électroaimant suffisaient à la bloquer indéfiniment.

La correction combine un watchdog AVR de 4 s, des retransmissions W5500 ramenées de ~30 s à ~1,5 s pour rester sous le watchdog, un MQTT non bloquant, l'électroaimant alimenté dès le début de `setup()`, et des chaînes en `F()` pour passer de 57 % à 40 % de RAM statique. Surtout, la carte publie maintenant un topic de diagnostic *retained* avec un compteur de démarrages, l'état du dernier reset et un « fil d'Ariane » (`reset_step`) qui indique où le code se trouvait avant le dernier redémarrage. Le diagnostic à distance d'un microcontrôleur, c'est ça : laisser des miettes avant de mourir.

### Le DMX et les projecteurs têtus

Côté lumière, MQTTdmx génère les trames DMX512 en logiciel sur un FTDI FT232R : 250 kbauds, 8N2, break de 88 µs, mark-after-break de 8 µs. Certains projecteurs en mode HOLD produisaient des fondus parasites sous une émission continue. Nous avons donc ajouté un mode `send_on_change` qui envoie une **rafale de 4 trames** (~130 ms) à chaque changement : une trame isolée peut se perdre sur un câble bruité, et un projecteur en HOLD garderait alors la mauvaise valeur indéfiniment.

## Linux partout, mais pas le même partout

Tout ce petit monde tourne sur Linux, mais pas sur *un seul* Linux. Quand j'ai mesuré l'état du parc pour concevoir notre autorité de certification SSH, la répartition était la suivante :

| Distribution | Machines |
|---|---|
| Debian 13 (Trixie) | 79 |
| Debian 12 (Bookworm) | 25 |
| Raspbian 10 (Buster) | 18 |
| Raspbian 11 (Bullseye) | 13 |
| Ubuntu 20.04 | 8 |
| Ubuntu 24.04 / 26.04 | 4 |

Ajoutez quelques serveurs centraux historiques sous Windows, que nous migrons au fil de l'eau. Côté matériel : des mini-PC x86 (Intel N95/N100, HP ProDesk 400 G4 DM, un Celeron N4020 avec 4 Go et de l'eMMC « à la limite de la saturation mémoire »), des Raspberry Pi 4 en arm64, et même un Pi 3 B, et des tablettes Debian sur mini-PC. Les paquets sont produits en **amd64 et arm64**.

### Le standard visé

- **Serveurs centraux :** Debian 13 en mode serveur, sans interface graphique, avec `docker.io` et la pile Maestro.
- **Raspberry Pi :** Raspberry Pi OS Lite 64 bits / Debian Trixie arm64. Pour la vidéo, une image dorée produite avec `rpi-image-gen` (commit épinglé pour des builds reproductibles) qui démarre directement sur labwc (Wayland), puis le lecteur en plein écran.
- **Tablettes et écrans d'accueil :** Debian 12 avec Ubuntu Frame (Mir), Chromium et le clavier à l'écran en snaps, ce qui empêche de quitter l'application. Sur certains Pi, cage + Firefox.

### Ce que l'hétérogénéité coûte vraiment

Chaque distribution apporte son lot de surprises, et la plupart se règlent dans le code :

- `paho-mqtt` 1.4.0 sur Buster oblige à maintenir une couche de compatibilité dans le module MQTT commun.
- Sur Bookworm, pygame 2.1.2 expose `mixer.Channel` comme une fonction et non comme une classe : l'annotation `Channel | None` plantait à l'import. Un `from __future__ import annotations` a réglé le problème.
- Sur Pi 4 + Trixie, le backend Vulkan (V3DV) de mpv est cassé : on force `gpu-api=opengl`.
- L'OpenSSH 7.9 de Debian 10 ne comprend pas la directive `Include`, ce qui a contraint le déploiement de l'autorité de certification SSH.

La vraie réponse à l'hétérogénéité, c'est de **réduire le nombre de cibles**. Pour cela, on peut désormais reflasher un Pi à distance : l'outil envoie une image SD complète en RAM, l'écrit depuis un petit système qui ne lit plus la carte, puis redémarre. Premier succès sur le banc : un Pi 4 passé de Raspbian 10 armhf à Debian 13 arm64 sans que personne ne se déplace.

### Le paquet Debian comme unité de déploiement

Tout ce qui s'installe sur une machine de salle est un `.deb`. Ce n'est pas une préférence esthétique : c'est le seul format qui garantit, sur toutes ces distributions, des dépendances déclarées, une configuration préservée lors des mises à jour (*conffiles*) et une désinstallation propre.

Le cas le plus intéressant est `maestro-room`. Un seul paquet, `maestro-room_<version>_<arch>.deb`, embarque toutes les images Docker de la salle sous forme d'archive, chargée à l'installation. Il n'y a **aucun pull de registre sur site** et aucun accès GitHub nécessaire. La configuration vit dans `/etc/maestro/room.env` (un conffile) et revenir en arrière consiste à réinstaller le paquet précédent. La CI construit les deux architectures avec buildx, vérifie le contenu du paquet, pose le tag et publie la release.

Les règles que nous appliquons à nos paquets ont toutes été apprises sur un incident :

- `apt install ./paquet.deb` doit suffire. Toute dépendance doit pouvoir s'exprimer dans `Depends:`. Si ce n'est pas le cas, on change de solution : `yq` a été remplacé par `python3-yaml`, parce que deux implémentations incompatibles portent ce nom.
- **Les scripts de maintenance dpkg ne font pas de réseau et ne démarrent rien.** Installer un paquet sur une machine en exploitation ne doit avoir aucun effet visible.
- **Les versions sont épinglées partout.** Pas d'image `latest`, client Tailscale gelé par `apt-mark hold`. Une mise à jour est une décision, jamais un effet de bord.

### systemd, mon meilleur ami et mon pire ennemi

Presque tous les agents tournent en `Type=notify`, avec un `READY=1` envoyé quand ils sont réellement prêts. Quelques réglages montrent ce qu'implique un usage « temps réel mou » :

```ini
# Lecteur audio
Type=notify
TimeoutStartSec=600
Nice=-10
CPUSchedulingPolicy=fifo
CPUSchedulingPriority=50
LimitMEMLOCK=infinity
```

Le `TimeoutStartSec=600` a une histoire. Un catalogue de ~4 000 pistes (5 Gio) préchargé avant `READY=1` dépassait le timeout de 90 s : systemd tuait le service, qui redémarrait, qui préchargeait… On envoie maintenant `EXTEND_TIMEOUT_USEC` pendant le chargement, et les longues pistes sont lues en streaming par morceaux d'une seconde. Une ambiance de 600 Mio coûte désormais 2 Mio de RSS au lieu d'être décodée en RAM.

Le lecteur vidéo tourne en **unité utilisateur** (`systemctl --user`), ce qui implique `loginctl enable-linger` dans le postinst (sinon le service ne démarre jamais sur une machine sans écran de connexion) et un `ExecStartPre` qui attend le socket Wayland ou X11. On ne peut pas compter sur `graphical-session.target` : labwc sur Pi ne l'active jamais. Avant ce correctif, le service démarrait 18 s avant le compositeur et affichait « Display not available » pendant 2 h 30.

Et puis il y a eu le bug « le player ne survit plus à son labwc ». Avec `PAMName=login`, le script de session et le lecteur vivent dans le scope de la session logind, hors du cgroup du service. Un `systemctl restart` ne tuait que labwc : l'ancien lecteur continuait de tourner à côté du nouveau, chacun avec la grande vidéo de la salle Jurassik en RAM, jusqu'à l'OOM killer sur un Pi de 8 Go sans swap. La correction tient en quelques lignes de shell :

```sh
/usr/bin/python3 /opt/MQTTmpv/MQTTmpv.py &
py=$!
trap 'stop_player; exit 0' TERM INT HUP
# Le player ne doit pas survivre a son labwc.
while kill -0 "$py" 2>/dev/null; do
    if [ -n "$SESSION_LABWC" ] && ! kill -0 "$SESSION_LABWC" 2>/dev/null; then
        stop_player; exit 0
    fi
    sleep 2
done
```

Un dernier chiffre pour la route : un service en `Restart=always` avait redémarré **4 655 fois** sans que personne ne s'en aperçoive. Remonter `NRestarts` dans notre supervision est la prochaine étape. Un service « actif » n'est pas forcément un service sain.

### Le rendu vidéo sans bureau

Sur les machines polyvalentes, nous préférons le rendu **KMS direct** (`vo=gpu`, `gpu-context=drm`) à une session graphique : moins de paquets, pas de tty confisqué. Pour le multi-écran, un seul processus tient le *DRM master* de `/dev/dri/card0` et pilote plusieurs connecteurs, chacun avec son propre `PlayerID` et un décodage VA-API. Il ne peut pas en être autrement : une carte DRM n'accepte qu'un seul master. Une sortie qui tombe est recréée avec un backoff exponentiel, et le service ne s'arrête que si toutes les sorties sont mortes.

Même la mémoire se négocie. Le lecteur forçait un cache démuxeur de 2 Gio dans les deux sens : avec une vidéo de 1,6 Go, la RAM gonflait à chaque partie jusqu'à geler un serveur de 8 Go. Nous sommes revenus aux valeurs par défaut de mpv, avec une exception assumée : l'ascenseur de la Chocolaterie, où **cinq écrans** sont synchronisés entre eux et avec le son *sans aucun système de synchronisation*. Seule la vidéo entièrement en RAM permet d'y arriver.

### Docker oui, Kubernetes non

Docker n'est utilisé que sur le serveur central, en Compose, avec `live-restore` activé. La question de l'orchestration de conteneurs sur les machines de salle a été tranchée par écrit : *une salle fait tourner un logiciel métier, pas un cluster.*

La résilience aux coupures de courant est traitée au niveau de systemd :

- `maestro-room.service` lance `docker compose up -d`, avec une limite de redémarrages bornée pour qu'une vraie panne reste visible.
- Un service de récupération détecte le cas où dockerd a redémarré « sale » (ports encore réservés) et fait un cycle down/up, au maximum une fois par boot.
- Un timer de surveillance redémarre chaque minute les conteneurs malades, au maximum 3 fois par heure et par service, **jamais pendant une partie** et jamais pendant une mise à jour (verrou de maintenance).
- Le tout est validé sur un banc « coupure de courant » : une VM Debian 13 jetable qu'on coupe brutalement avec SysRq.

Petite astuce d'exploitation : les Pi n'ayant pas d'horloge matérielle, on compare les boots avec `journalctl -o short-monotonic` plutôt qu'avec l'heure murale.

## Exploiter 150 machines réparties sur 13 sites

Je ne peux pas me déplacer à chaque fois qu'un lecteur audio fait des siennes. L'exploitation à distance est donc un produit à part entière, nommé *vigie*, livré lui aussi en deux paquets Debian (serveur et edge).

- **Un VPN maison, strictement additif.** Nous auto-hébergeons Headscale (version épinglée) et les machines rejoignent le tailnet avec le client Tailscale. Le VPN ne touche ni au DNS, ni au routage, ni aux interfaces d'une machine. Les options de `tailscale up` sont toutes passées explicitement, même à leur valeur par défaut, pour neutraliser d'éventuelles préférences résiduelles :

  ```bash
  tailscale up --login-server "$login_server" --auth-key "$authkey" \
    --hostname "$node_name" --accept-dns=false --accept-routes=false \
    --advertise-routes= --advertise-exit-node=false --advertise-tags= \
    --ssh=false --timeout 60s
  ```

- **Un nommage par fonction, pas par machine.** `<produit>-<site>[-<niveau>…]-<fonction>[<n>]`, par exemple `maestro-lille-chocolaterie-ascenseur-tv4`. Un nom décrit une fonction à un endroit : un Pi de remplacement reprend le nom de celui qu'il remplace.
- **Une supervision en push.** Chaque machine remonte son état chaque minute via un timer systemd (avec un délai aléatoire pour lisser la charge). En pull, une machine éteinte fait attendre tout le monde. En push, *une machine hors ligne se signale par son silence*. L'identité d'un nœud est son adresse sur le tailnet : pas de jeton par machine, et retirer un nœud du VPN suffit à le faire taire.
- **Des certificats SSH de courte durée** (16 h) émis par une autorité de certification interne, plutôt que des clés posées à la main sur chaque machine.
- **Aucune action corrective automatique.** La supervision informe, l'humain décide.

Pour le diagnostic, j'ai construit une base de connaissances de maintenance à partir de deux mois d'interventions : 58 symptômes, 27 procédures, une fiche par site. Quelques règles reviennent sur chaque ticket : retrouver l'heure *réelle* de la partie, lire les logs avant le code, et **ne jamais agir sur une salle en cours de partie**. Sur les 58 symptômes, 20 se règlent par une action fermée (redémarrer, réarmer), 10 demandent quelqu'un sur place et 16 relèvent du code. La valeur est dans le diagnostic, pas dans les redémarrages.

La sécurité y a sa propre procédure : on ne déclenche jamais l'urgence pour tester, et après une vraie alarme, il faut réarmer explicitement les lecteurs audio. Le jour où personne ne l'a fait, l'ascenseur de la Chocolaterie est resté muet pendant sept heures. Le verrou *retained* faisait exactement ce pour quoi il avait été conçu.

## Archipel : brancher la réservation sur les salles

Archipel est notre plateforme d'exploitation de site : synchronisation avec l'outil de réservation en ligne, borne d'accueil (check-in), tablette de salle pour les joueurs, tableaux de bord sur les téléviseurs de l'accueil. Elle tourne sur un mini-PC par site, en .NET 10 avec PostgreSQL 16, et le front est en JavaScript vanilla (modules ES, sans npm ni bundler), alimenté en SSE avec repli en polling.

Le pont avec Maestro passe évidemment par MQTT. Archipel envoie des instructions (`ChangeRoomState`, publication d'un gameflow) et reçoit des notifications (`RoomStateChanged`, `SessionProgressionInit`, `SessionProgressionUpdate`). Le chrono affiché sur la tablette de la salle illustre bien la philosophie : Maestro fait autorité sur le temps. Il envoie un `Init` (durée totale, heure de départ) puis un `Update` environ toutes les 30 s, et la tablette **interpole elle-même** le compte à rebours à partir d'une ancre d'horloge murale. Aucun message à la seconde ne transite sur le réseau.

Une leçon récente : une réservation arrivée sans identifiant de créneau a bloqué toute la passe de synchronisation, qui tourne toutes les 5 minutes. Résultat : plus de nouvelles réservations, d'annulations ni de paiements importés, et des équipes introuvables à la borne. Le correctif est un classique qu'on oublie trop souvent : **une donnée invalide ne doit jamais bloquer les suivantes**. Chaque réservation est désormais traitée et réessayée indépendamment.

Notre projet le plus récent, une salle multijeux, pousse ces principes plus loin : serveur FastAPI, jeux écrits comme des fonctions pures `(état, action) → (nouvel état, effets)`, protocole de jeu en Protobuf sur WebSocket avec numéros de séquence et instantané complet à la reconnexion, MQTT pour le contrôle (présence par Last Will, état *retained*, commandes en QoS 1) et une contrainte en tête du cahier des charges : fonctionner **sans Internet**.

## Et les agents IA dans tout ça ?

Dans un article sur les agents, il serait dommage de ne pas parler des autres. Une bonne partie du code présenté ici a été écrite en binôme avec Claude Code, et la façon de travailler s'est adaptée :

- **Chaque dépôt a un `CLAUDE.md`** qui décrit le vocabulaire, les règles de conception et les pièges connus. Sur nos dépôts les plus récents, une règle est explicite : *la documentation est la seule mémoire commune des agents qui travaillent sur le dépôt*. Des tests vérifient que chaque chemin cité dans la doc existe et qu'une page ne dépasse pas 150 lignes.
- **Une « usine logicielle »** à trois rôles : un agent planificateur rédige une spécification avec des critères d'acceptation testables, un humain l'approuve, un agent constructeur l'implémente et un agent vérificateur tente de la casser, avec au plus trois boucles de correction. Les tests portent les identifiants des critères d'acceptation (« CA-24 »).
- **Des garde-fous écrits après incident** pour l'IA en exploitation : un feu vert humain explicite avant toute écriture sur site, une salle par appel d'outil, pas de boucle bloquante pendant un débogage en direct (une machine à fumée s'en souvient encore), et *aucune initiative*, après des canaux DMX modifiés « pour tester ». Quand l'assistant passe par notre serveur MCP de maintenance, les permissions par site sont appliquées côté serveur : on ne s'appuie pas sur le seul prompt système.

## Ce que j'en retiens

Si je devais résumer ce que ce terrain de jeu m'a appris :

1. **Le message est le contrat.** Les formats sur le fil évoluent par ajout seulement, et les entiers ne sont jamais renumérotés. Une salle peut tourner des années sur une vieille version.
2. **Tout redémarre, tout le temps.** Chaque composant doit republier son état réel à la reconnexion, les opérations doivent être idempotentes, et l'état critique doit vivre dans le broker (*retained*) ou en base, pas dans la RAM d'un processus.
3. **Le matériel d'abord, le réseau ensuite.** Un actionneur doit être dans un état sûr avant même que la carte sache qu'elle a une adresse IP.
4. **Le silence est une information.** Supervision en push, keepalive courts, Last Will : on veut savoir qu'une chose est morte, pas seulement qu'elle est vivante.
5. **Un paquet Debian, pas un tutoriel.** Si l'installation demande une étape manuelle, elle sera oubliée sur la vingtième machine.
6. **On ne touche pas une salle en jeu.** Ni la supervision, ni le watchdog, ni moi.
7. **La sécurité des personnes reste câblée.** Le logiciel suit le SSI, il ne le remplace pas.

Développer pour des escape games, c'est faire de l'informatique distribuée où chaque nœud a un corps : un moteur, une serrure, un haut-parleur. C'est exigeant, souvent frustrant (une carte qui répond au ping alors qu'elle est morte…), et je ne connais pas beaucoup de terrains de jeu aussi formateurs. Et quand une équipe sort d'une salle en criant de joie, je sais que derrière la porte, des dizaines d'agents et des centaines de messages MQTT ont fait exactement ce qu'on attendait d'eux.
