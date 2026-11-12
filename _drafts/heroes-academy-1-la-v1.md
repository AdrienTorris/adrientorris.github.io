---
title: "Heroes Academy (1/3) : concevoir le logiciel d'une salle de jeu physique"
description: "Une salle où l'on grimpe, saute et court est un système distribué temps réel. Retour sur la V1 : .NET 5, Avalonia, MongoDB et Raspberry Pi, et ce qu'elle m'a appris."
lang: fr
tags: [dotnet, csharp, mqtt, raspberry-pi, iot, architecture, retour-d-experience]
---

# Heroes Academy (1/3) : concevoir le logiciel d'une salle de jeu physique

> **Série Heroes Academy**
> 1. **Concevoir le logiciel d'une salle de jeu physique** (vous êtes ici)
> 2. [La refonte V2 pour le Canada](./heroes-academy-2-la-refonte-v2.md)
> 3. [Huit mois de développement assisté par IA](./heroes-academy-3-developper-avec-l-ia.md)

Il y a des projets où un bug donne une stack trace. Et il y a ceux où il donne une équipe de six personnes bloquée dans une pièce, devant un écran figé, pendant que l'équipe suivante attend à la porte.

Heroes Academy fait partie de la seconde catégorie. C'est un concept de « jeu actif » : des équipes enchaînent des salles physiques où il faut grimper à des cordes, sauter sur un damier lumineux, viser des paniers ou traverser une pièce sans toucher le sol. Le tout est chronométré, scoré, commenté par des voix et accompagné de vidéos. Il existe aujourd'hui trois Heroes Academy : deux en France, à Lille et à Tarbes, et une au Canada, à Brossard, près de Montréal.

Je développe le logiciel de ces salles depuis le début. Cette série raconte les difficultés techniques rencontrées en trois temps : la première version et ses leçons (cet article), la refonte complète menée pour l'installation canadienne, puis la place qu'a prise le développement assisté par IA.

---

## Une salle de jeu, c'est un système distribué

Vu de l'extérieur, une Heroes Academy est une suite de pièces. Vu du code, c'est un système distribué temps réel. Il tourne sur une douzaine de machines par site, dans un environnement qui n'a rien d'un datacenter.

Une équipe suit un **parcours** : Dressing, Briefing, Warm Up, puis les salles de jeu (Ropes, Snake, Don't Touch The Floor, Logs, Balls, Communication), puis le Debriefing. Chaque salle de jeu dure quelques minutes et contient tout ou partie de ces éléments :

- des **boutons-cibles** rétroéclairés à frapper, parfois en hauteur ;
- des **portes à ventouse magnétique**, qui verrouillent l'entrée pendant la partie et libèrent la sortie à la fin ;
- des **gyrophares** et un éclairage d'ambiance ;
- un **écran** pour les consignes, le chrono, le score et les vidéos, et des **haut-parleurs** pour les musiques, les voix et les sirènes ;
- selon la salle, un **damier de dalles au sol** (Snake), des **capteurs de sol** (Don't Touch The Floor) ou des **capteurs de paniers** avec des afficheurs de points (Balls) ;
- une **mission secondaire** : des boutons verts cachés qui débloquent une vidéo indice, puis une question à choix multiples.

Les contraintes qui en découlent ne ressemblent pas à celles d'une application web :

| Contrainte | Conséquence technique |
|---|---|
| Les salles sont **enchaînées** | Une salle bloquée bloque tout le parcours, et les équipes suivantes avec. |
| Le jeu est **physique** | Une pression de bouton perdue, ou comptée deux fois, se voit immédiatement. |
| Le matériel est **embarqué** | Raspberry Pi sur carte SD, coupures de courant, switch éteint par erreur. |
| L'exploitation n'est **pas technique** | Un Game Master doit pouvoir relancer une salle sans terminal. |
| Les sites sont **loin** | Lille, Tarbes, et bientôt Montréal : on ne peut pas passer redémarrer un Pi. |

### L'électronique : MQTT plutôt que GPIO

Le tout premier choix structurant a porté sur l'électronique. Les premières versions pilotaient les entrées/sorties directement depuis le Raspberry Pi : GPIO et expandeurs I²C MCP23017 interrogés toutes les 50 ms. Ce code existe encore dans la V1… entièrement commenté.

Assez vite, nous sommes passés à des **cartes d'entrées/sorties en réseau** qui parlent MQTT. Chaque carte publie ses entrées et reçoit ses sorties sous forme de messages texte très simples :

```text
/io/input   ←  /c0m2p5/on     (contrôleur 0, module 2, broche 5 : bouton enfoncé)
/io/input   ←  /c0m2p5/off    (bouton relâché)
/io/output  →  /c1m0p3/on     (allumer la LED de la cible correspondante)
```

C'est probablement la décision qui a le mieux vieilli. Le logiciel n'a plus besoin de savoir *comment* un bouton est câblé : il reçoit des chaînes de caractères. On peut donc simuler une salle entière avec `mosquitto_pub`. Cette décision a survécu à la refonte.

---

## La V1 : un monolithe par Raspberry Pi

La V1 a été démarrée par un prestataire, la société de services dont j'étais alors gérant associé. Elle a été développée sur Azure DevOps avant que le code ne migre sur GitHub en 2023. C'est elle qui a ouvert Lille, puis Tarbes. Son architecture tenait en un schéma :

```text
                 ┌────────────────────────────────────────┐
                 │  Serveur central (Windows)             │
                 │  ASP.NET Core 5 (admin, Razor Pages)   │
                 │  MongoDB                               │
                 └───────────────▲────────────────────────┘
                                 │  connexions MongoDB permanentes
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────┴─────────┐   ┌─────────┴────────┐    ┌─────────┴────────┐
│ Pi « Ropes »     │   │ Pi « Snake »     │    │ Pi « Balls »     │  …
│ Avalonia 0.10    │   │ Avalonia 0.10    │    │ Avalonia 0.10    │
│ logique + UI +   │   │ logique + UI +   │    │ logique + UI +   │
│ son (LibVLC) +   │   │ son (LibVLC) +   │    │ son (LibVLC) +   │
│ MQTT, 1 process  │   │ MQTT, 1 process  │    │ MQTT, 1 process  │
│ Mosquitto local  │   │ Mosquitto local  │    │ Mosquitto local  │
└────────▲─────────┘   └────────▲─────────┘    └────────▲─────────┘
         │ MQTT                  │ MQTT                  │ MQTT
   cartes E/S             cartes E/S              cartes E/S
```

Chaque Pi faisait tourner **une seule application desktop Avalonia** sous .NET 5, publiée en self-contained `linux-arm`. Elle contenait tout : la logique de jeu, l'interface, le son (LibVLC avec quatre lecteurs en parallèle pour l'ambiance, les voix, les effets et la sirène), et le client MQTT vers le broker local, sur lequel discutaient les cartes d'E/S.

### MongoDB comme bus d'intégration

Il n'y avait pas de bus de messages entre les salles et le serveur. **La base de données en tenait lieu.** Toutes les secondes, chaque Pi demandait à MongoDB s'il avait une session à jouer :

```csharp
// V1, simplifié : chaque Raspberry Pi interroge MongoDB toutes les secondes
var session = gameSessionService.GetSession(roomId);   // SlotStart <= now < GameEnd

bool isRecovering = session.TimeElapsed != TimeSpan.Zero
    && session.TimeLeft > session.GameEndDateTime - DateTime.Now + TimeSpan.FromSeconds(6);

if (isRecovering)      current.Recover(session);                       // redémarrage en pleine partie
else if (isPlaying)    current.ComparePersistentDataSession(session);  // l'admin a modifié la session
else                   current.Start(session);
```

Dans l'autre sens, chaque jeu réécrivait son temps restant en base **toutes les secondes**. Les actions du Game Master (« sauter la mission », « réinitialiser la salle ») n'étaient que des drapeaux posés dans le document de session, que le Pi découvrait au tick suivant.

Ce n'est pas élégant, mais ça a fonctionné, et c'était simple à raisonner. La logique `Recover` permettait même à une salle de reprendre une partie en cours après un crash, en recalculant le temps restant à partir de l'heure de fin prévue.

Ça avait cependant un coût caché : le bon fonctionnement dépendait de **l'heure système de chaque Pi** et d'une **connexion permanente à la base**.

### Le processus zombie

C'est l'incident le plus fréquent de toute l'histoire de la V1, et il est arrivé presque toujours le matin, avant l'ouverture.

Le serveur central redémarre, à cause d'une mise à jour Windows de nuit ou d'une coupure de courant. Quand il revient, aucune salle ne fonctionne, et la seule solution connue est de **redémarrer tous les Raspberry Pi**.

Nous avons longtemps soupçonné MQTT. C'était faux : chaque Pi a son propre Mosquitto, et un redémarrage du serveur ne peut pas le tuer. La vraie cause a été établie en observant les connexions TCP ouvertes depuis un Pi. L'application maintenait un **pool de connexions permanentes vers MongoDB**. Le redémarrage du serveur tuait ces sockets, et le client ne reconstruisait jamais le pool.

Le détail qui rendait le problème vicieux : **le processus ne mourait pas**. Il tournait toujours, attaché à des sockets mortes. Chaque Pi avait pourtant un script de relance :

```bash
# ha-keepalive.sh, lancé à l'ouverture de session graphique
while :; do
    $dq_app_path & pid=$!
    wait $pid
    sleep 2s
done
```

Mais ce script relance un processus *mort*, pas un processus *inutile*. La correction a pris la forme d'un watchdog. Il sonde le port de la base toutes les 15 secondes et, sur une transition « injoignable puis revenue », tue le jeu pour laisser `ha-keepalive` le relancer :

```bash
joignable() { timeout 3 bash -c "exec 3<>/dev/tcp/$MONGO_HOST/$MONGO_PORT" 2>/dev/null; }

while :; do
  if joignable; then
    succes=$((succes + 1))
    if [[ $etat == coupe && $succes -ge 4 ]]; then   # 4 sondes OK d'affilée
      relancer_le_jeu                                # kill, puis ha-keepalive relance
      etat=ok
    fi
  else
    succes=0; etat=coupe
  fi
  sleep 15
done
```

Les garde-fous sont volontaires :

- une salle dont la base n'a jamais bougé n'est jamais touchée ;
- un hoquet réseau de quelques secondes ne déclenche rien ;
- le journal est tronqué à 256 Ko, parce que ces machines tournent sur carte SD.

La leçon dépasse ce cas : **un processus vivant n'est pas un processus qui fonctionne.**

### Les autres frictions

Le processus zombie était le plus visible, mais pas le seul problème :

- **Une pile figée.** .NET 5 n'est plus supporté depuis mai 2022 et la V1 était épinglée sur Avalonia 0.10. Monter de version imposait de migrer en même temps le runtime, le framework UI et l'intégration LibVLC sur ARM, sans filet. Avalonia n'est pas en cause en soi. Le problème venait de l'avoir mis au centre : quand le framework d'affichage héberge aussi la logique de jeu et le son, tout est couplé à son cycle de vie.
- **La concurrence.** Plusieurs timers tournaient en parallèle (polling de session, compte à rebours, timers de jeu), en plus des callbacks MQTT et du dispatcher UI. Pour éviter les accès concurrents, certains jeux utilisaient `Monitor.TryEnter(padlock, TimeSpan.Zero)`. Une pression de bouton qui arrivait pendant que le verrou était tenu était donc **ignorée en silence**. Le Snake, lui, faisait des `Thread.Sleep` de plusieurs secondes pour ses animations.
- **Le déploiement artisanal.** Le processus était à chaque fois le même : `dotnet publish`, suppression des fichiers de configuration, zip, copie sur le site, dézip, restauration de la configuration locale. Le câblage d'une salle (quel bouton sur quelle broche) existait à **trois endroits différents**, avec des dérives : une sortie remappée à la main pour contourner une broche morte, un fichier de site recopié d'un autre. Certaines différences entre sites se géraient même en commentant ou décommentant une ligne avant de compiler (`// Pour Lille` / `// Pour Tarbes`).
- **Les médias.** Des vidéos étaient versionnées dans le dépôt Git (plus de 600 Mo de `.git`, dont une vidéo de 321 Mo refusée par GitHub). La lecture plein écran lançait un `vlc` externe, qu'on tuait ensuite par son nom.
- **Le serveur Windows.** L'administration était une application console lancée depuis le dossier Démarrage. Elle dépendait donc de l'ouverture d'une session utilisateur et d'un certificat de développement ASP.NET qui finissait par expirer. Un jour, un switch a été éteint par erreur : quatorze machines sont tombées d'un coup, et l'administration n'est pas revenue au rallumage.
- **Le matériel réel.** À Tarbes, les capteurs de la salle Balls sont en haut du panier. Un ballon qui rebondit sur l'arceau marquait donc deux fois, et il a fallu ajouter un anti-rebond d'une seconde par panier. Rien de tout ça ne figure dans un cahier des charges.

### L'Arène, ou l'art de ne pas croiser les équipes

La V1 a aussi introduit l'**Arène**, une salle partagée où chaque équipe passe 30 secondes entre deux salles de jeu pour répondre à une question sur écran tactile. Sur le papier, c'est simple. En pratique, l'Arène est une ressource partagée par toutes les équipes du parcours, et deux équipes ne doivent jamais s'y croiser.

Le planificateur précalculait tout le parcours d'une équipe à la création de la session, une étape par document. Il a fallu revoir l'espacement des départs (Briefing + Dressing + 3 × Arène) et ajouter des tests unitaires sur 30 à 50 sessions consécutives pour garantir qu'aucun croisement ne restait possible.

### Une V1 qui a fait son travail

Il serait injuste de ne retenir que les problèmes. La V1 a ouvert deux sites et les a fait tourner pendant des années. Elle a aussi beaucoup évolué au contact du terrain :

- les paniers de Balls pondérés selon le nombre de joueurs ;
- un Snake repensé, avec des séquences aléatoires et un scénario dédié aux équipes de six ;
- des voix à mi-temps, à une minute et à dix secondes de la fin ;
- des questions lues à voix haute dans l'Arène.

Certains réglages ont été faits, défaits et refaits en quelques jours, directement sur site. C'est la nature d'un jeu physique : on ne sait pas si une mécanique est amusante avant d'avoir vu une équipe la jouer.

---

## Ce que la V1 m'a appris

1. **Une base de données n'est pas un bus de messages.** Le polling d'une base partagée crée un couplage invisible, à l'horloge comme à la connexion. Le jour où la connexion meurt sans que le processus meure, plus rien ne bouge, et rien ne le signale.
2. **Un processus vivant n'est pas un processus qui fonctionne.** Superviser un PID ne suffit pas, il faut superviser la fonction.
3. **Ce qui peut diverger divergera.** Une configuration copiée à trois endroits finit par avoir trois versions. Une différence entre sites gérée par un commentaire finit par être compilée pour le mauvais site.
4. **Le terrain a toujours raison.** Ballons qui rebondissent, switch éteint, serveur qui redémarre la nuit : aucune spec ne remplace une heure dans la salle.

Quand le projet canadien s'est concrétisé, la question s'est posée simplement : installer une nouvelle copie de cette V1 à plus de 5 000 km, sans pouvoir passer redémarrer un Pi le matin ? La réponse a été non. C'est le sujet de la deuxième partie.

---

> **Suite de la série**
> 2. [La refonte V2 pour le Canada](./heroes-academy-2-la-refonte-v2.md) : l'architecture, les principes et les bugs de la V2.
> 3. [Huit mois de développement assisté par IA](./heroes-academy-3-developper-avec-l-ia.md) : ce que l'historique Git dit de l'IA dans le projet.

*Les extraits de code de cet article sont simplifiés. Les adresses, identifiants et détails d'infrastructure propres à chaque site ont été volontairement omis.*
