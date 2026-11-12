---
title: "Donut Quiz : synchroniser un siège et des agences sans rien perdre en silence"
description: "Filigrane qui suit la mauvaise horloge, propriétés qui disparaissent derrière un 200 OK, enrôlement qui « réussit » sans rien faire : retour d'expérience sur la synchronisation de contenu entre le siège et les salles Donut Quiz."
lang: fr
tags: [dotnet, synchronisation, mongodb, hangfire, system-text-json, architecture, retour-experience]
---

# Donut Quiz : synchroniser un siège et des agences sans rien perdre en silence

> Cet article prolonge [Donut Quiz : les coulisses techniques d'une salle de quiz connectée](./donut-quiz-article-fr.md), qui présente la salle, son architecture et la refonte du logiciel. Ici, je m'intéresse à une seule pièce du puzzle : la synchronisation entre le siège et les agences.

Une salle Donut Quiz doit pouvoir jouer même si la connexion internet de l'agence tombe. Chaque agence a donc sa propre base MongoDB et son propre volume de médias (13 à 19 Go d'images, de vidéos et de sons). Le contenu (questions, thèmes, événements spéciaux, slides, cartes d'honneur, textes, réglages de jeu…) est rédigé au siège et descend vers les agences. Dans l'autre sens, les parties jouées, les clients et l'historique des questions posées à chaque client remontent. Cet historique permet au générateur de parties de ne pas reposer une question à un client qui revient.

Sur le papier, c'est un problème classique. En pratique, c'est la partie du système où les défauts sont les plus sournois, parce qu'une synchronisation qui perd des données ne plante pas : elle répond 200, se déclare en succès, et le manque ne se voit que des semaines plus tard, quand un game master s'étonne qu'un événement spécial n'apparaisse pas.

> **En bref**
> - Le filigrane suivait l'horloge de l'agence alors que les données sont datées par le siège. Il est désormais dérivé de ce qui a réellement été écrit.
> - System.Text.Json, avec `IgnoreReadOnlyProperties`, laissait tomber en silence toute propriété sans setter public. Une garde au démarrage refuse maintenant de lancer l'application si une propriété ne traverse pas.
> - L'agence voit ce qui lui manque et peut déclencher la synchro à la demande. Le siège voit l'état de tout le parc.
> - Le contenu peut appartenir à un franchisé, avec un cloisonnement strict entre agences.
> - Le siège ne se connecte jamais à une agence : tout part de l'agence, en HTTPS sortant.

---

## Comment ça marche

Dans la première version du logiciel, la synchronisation était le dernier composant arrivé. Elle tournait la nuit, entre 1 h et 5 h, sauf pour les parties, synchronisées toutes les cinq minutes. Le principe n'a pas changé, mais presque tout le reste, si.

- **Un service dédié.** Côté agence, `ContentSynchronization.AgenceApp` exécute des jobs Hangfire, stockés dans MongoDB. Quinze entités circulent : toutes descendent, et trois remontent aussi (parties, clients, historique des questions).
- **Un calendrier étalé.** Le contenu se synchronise la nuit. Chaque agence tire au hasard son heure de départ dans une fenêtre de cinq heures à partir de 22 h, ce qui étale la charge sur le siège. Les parties et les clients, eux, sont synchronisés toutes les cinq minutes, parce qu'une réservation prise en ligne doit arriver vite en salle.
- **Un filigrane par entité.** Chaque entité a son filigrane (`Since`). L'agence appelle `POST /siege/api/sync/{Entité}/pull` pour obtenir tout ce qui a changé depuis ce point (`LastUpdated >= Since`). Le siège renvoie les entités complètes avec les liens de leurs médias, et l'agence les écrit en base puis télécharge les médias dans le volume partagé.
- **La purge des contenus supprimés.** Une route `check-dead` permet à l'agence de demander lesquels de ses contenus n'existent plus au siège, puis de les purger.
- **Un enrôlement par jeton.** Pour s'enrôler, une agence échange un jeton générique à usage unique contre son propre jeton d'agence, puis déclare son code auprès du siège.
- **Un seul worker par file** : une conversion ffmpeg à la fois, et jamais deux tirages de la même entité en parallèle.

Rien de révolutionnaire. Le diable, comme toujours, se cache dans ce qui se passe quand ça se passe mal… sans le dire.

---

## Deux horloges pour un seul filigrane

À la fin de chaque passage, le filigrane prenait l'heure locale de l'agence (`DateTime.Now`). Or le `LastUpdated` des documents est posé par l'horloge du **siège**.

Il suffit que l'horloge de l'agence avance de quelques minutes pour créer un trou définitif : un document modifié au siège dans cet intervalle ne repassera jamais le filtre `LastUpdated >= Since`.

Pour couronner le tout, un élément arrivé sans données était compté comme « appliqué ». Un passage pouvait donc avancer le filigrane sans avoir écrit une seule ligne. Et le champ censé dater la dernière application ne bougeait jamais après la création du document de suivi : un passage qui n'appliquait rien laissait exactement la même trace qu'un passage qui appliquait tout.

Le filigrane est désormais dérivé de ce qui a **réellement été écrit** :

```csharp
// Avant : l'heure locale de l'agence, quoi qu'il se soit passé
contentSynchronizationInfo.Since = executionDate; // DateTime.Now, côté agence

// Après (principe) : le plus grand LastUpdated effectivement appliqué, daté par le siège,
// plafonné s'il est invraisemblable, ramené au plus ancien élément en échec,
// et laissé tel quel si rien n'a été écrit.
```

Un élément sans données n'est plus compté comme appliqué, et le document de suivi porte une vraie date de dernière application. L'horloge elle-même est injectée (`ISynchronizationClock`), ce qui permet de tester les cas de dérive sans attendre qu'une agence les produise.

---

## Un 200 OK ne prouve rien

Les deux extrémités sérialisent en System.Text.Json avec `IgnoreReadOnlyProperties = true`. Avec ce réglage, une propriété dont le setter n'est pas public n'est **ni écrite par l'émetteur, ni relue par le récepteur**. Pas d'exception, pas de trace, et un 200 en réponse.

La remontée des parties cumulait les problèmes :

- elle passait par un DTO de projection qui ne déclarait que 20 des 35 propriétés d'une partie ;
- la liste des équipes de ce DTO n'était jamais remplie. Comme le siège remplace intégralement le document à chaque poussée, il recevait des **parties sans équipes, donc sans scores ni classement** ;
- les parties terminées avaient un problème de plus : leurs honneurs étaient stockés dans un dictionnaire que MongoDB refusait de persister dans sa représentation par défaut, et c'était alors **toute la partie** qui était rejetée. Précisément les parties dont le siège a le plus besoin.

Le correctif « évident », retirer `IgnoreReadOnlyProperties`, aurait cassé le pull : certaines entités exposent des propriétés calculées typées par une interface, que le désérialiseur ne sait pas instancier. Le drapeau masquait en fait un modèle de domaine qui n'avait jamais été conçu pour être sérialisé.

### Une garde plutôt qu'une liste

J'ai donc procédé propriété par propriété, avec `[JsonInclude]`. Mais une liste de propriétés à surveiller vieillit mal : la prochaine propriété ajoutée avec un setter privé reproduira le défaut sans que personne le remarque.

Au démarrage, une garde inspecte donc par réflexion chacune des quinze entités, dans les deux sens. Si une propriété ne traverse pas, l'application refuse de démarrer. Seules lui échappent les propriétés calculées, exemptées automatiquement puisqu'elles n'ont pas de setter, et deux propriétés exemptées nommément, avec leur raison écrite.

Le message d'erreur nomme le type, la propriété, le sens et la cause, et jamais un simple compte :

```csharp
/// « 3 propriétés perdues » survivrait à une régression sur une propriété DIFFÉRENTE :
/// le compte ne bouge pas, le message reste vrai, et la perte change de victime sans bruit.
public override string ToString()
    => $"{Entite.Name}.{Propriete} ne traverse pas dans le sens {Sens} : {Cause}";
```

Un inventaire de la traversée, propriété par propriété et pour chaque sens, est aussi versionné dans le dépôt. Un test vérifie qu'il couvre bien les quinze entités dans les deux sens : supprimer une section le fait échouer.

### Une partie jouée est un fait

Une fois les manches transmises, un nouveau piège est apparu. Des questions incomplètes traînaient dans la banque de questions, et la validation de modèle aurait fait refuser en bloc toute partie qui en embarquait une. Or une partie jouée est un fait : on ne la refuse pas.

À la réception d'une poussée de partie, et pour cette seule action, la validation de modèle est donc levée. La partie est écrite telle quelle, et chaque champ manquant produit un avertissement qui nomme la partie, la manche et la propriété. La banque se nettoie par constat, pas par blocage. Tous les autres points d'entrée du siège conservent leur validation.

### Les silences ordinaires

En inspectant la synchro, j'ai trouvé un job qui appelait toutes les cinq minutes une route que le siège ne servait pas : aucun contrôleur n'existait pour cette entité. Il recevait un 404, n'appliquait rien… et rien ne le signalait.

Enfin, l'agence reste maîtresse de ce qu'elle joue. Une partie en cours n'est jamais écrasée par la synchro, et une copie locale au moins aussi récente que celle du siège fait autorité.

---

## Voir ce qui manque, et le tirer sans attendre la nuit

Quand un game master constate qu'un nouvel événement spécial n'apparaît pas, attendre la synchro de la nuit n'est pas une réponse. Encore fallait-il savoir ce qui manque : jusque-là, le seul moyen de compter les questions manquantes était de télécharger toutes les questions.

- **Le siège sait compter.** Un point d'entrée d'inventaire renvoie, par entité, le nombre d'éléments que l'agence n'a pas encore, sans en télécharger le contenu.
- **L'agence voit ce qui lui manque.** Son administration affiche ces compteurs, entité par entité, avec un bouton unique qui déclenche le tirage immédiatement.
- **Les clics sont sans danger.** Les demandes répétées sont fusionnées par entité : un clic pendant qu'un tirage est en file ou en cours ne déclenche pas de second tirage. Et aucun tirage ne part pendant une partie en cours.
- **Le siège voit tout le parc.** Chaque agence remonte son filigrane. Une vue du parc montre quelles agences sont à jour et lesquelles ont du retard, avec la fraîcheur de cette information.

L'administration et l'ordonnanceur sont deux processus distincts. Le bouton de l'administration appelle donc un point d'entrée HTTP de l'ordonnanceur, restreint à l'origine locale, tandis que le contrôle des rôles reste entièrement dans l'administration.

### Un enrôlement qui ne réussit plus en silence

L'enrôlement d'une agence pouvait « réussir » en silence par quatre sorties différentes, sans le moindre journal. Et la présence d'un jeton était prise pour sa validité : un jeton périmé, révoqué ou émis par un autre siège faisait tout aussi bien l'affaire.

Seul le siège sait juger un jeton. Il le vérifie désormais et dit pourquoi il refuse, tandis que l'agence journalise et affiche la raison. Un écran vert ne veut plus dire « une chaîne de caractères est présente dans un fichier ».

---

## Du contenu pour chaque franchisé

Dernière évolution en date : les franchisés créent du contenu au siège pour leur seule agence.

- **Un propriétaire optionnel.** Une question ou un événement spécial porte un propriétaire. L'absence de propriétaire veut dire « global », ce qui a évité toute migration des données existantes.
- **Une synchro filtrée par propriétaire.** Le pull, les compteurs, les liens de médias et la détection des contenus supprimés tiennent compte du propriétaire, avec un cloisonnement strict : une agence ne reçoit jamais le contenu d'une autre.
- **Pas de validation centrale.** Le contenu d'une agence part en salle dès qu'il est activé, uniquement vers l'agence propriétaire.
- **La « mise au pot ».** Un contenu d'agence qui mérite de profiter à tout le réseau peut être copié en contenu global.

---

## Aucune connexion entrante vers une agence

Même pour récupérer l'historique de jeu des anciennes agences, la règle reste la même : **le siège ne se connecte jamais à une agence.** Pas d'accès entrant, pas de clé SSH.

```
AGENCE (lecture seule)                          SIÈGE
  mongodump --archive --gzip                    PUT /…/ingest/<agence>/<collection>
        │  (streamé, aucun fichier temporaire)    └─> dépôt de l'archive
        └── curl -T -  ──── HTTPS sortant ────>   restauration dans une base jetable
                                                  puis fusion dans la base principale
```

La route d'ingestion est authentifiée par jeton et **inerte sans configuration** : sans jeton configuré, elle répond 404. Elle peut donc être déployée partout sans risque, puisque c'est la configuration, et elle seule, qui l'active. Côté agence, tout se fait en lecture seule : aucun service installé, aucun fichier temporaire, aucune écriture dans MongoDB, et l'opération est interruptible à tout moment.

---

## Ce que je retiens

- **Un 200 OK ne prouve rien.** Une synchronisation se juge sur ce qui a réellement été écrit.
- **Une horloge ne vaut que pour la machine qui l'a lue.** Un filigrane se dérive des données reçues, jamais de l'heure locale.
- **Une garde vaut mieux qu'une liste.** Ce qui doit rester vrai se vérifie au démarrage, par construction, plutôt que dans une liste que personne ne tiendra à jour.
- **Nommer plutôt que compter.** Un message qui nomme la propriété perdue ne peut pas rester vrai après une régression ailleurs.
- **La salle reste maîtresse de ce qu'elle joue.** Une partie en cours ne se fait jamais écraser.
- **Jamais de connexion entrante vers une agence.** Tout part de l'agence, en HTTPS sortant.

Pour le reste de l'histoire (les pupitres en web, MQTT, le son, la lumière et le déploiement sur Raspberry Pi), c'est dans l'article principal : [Donut Quiz : les coulisses techniques d'une salle de quiz connectée](./donut-quiz-article-fr.md).
