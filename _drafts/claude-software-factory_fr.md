# L'usine logicielle Claude : trois agents, un contrat et le droit d'échouer

*Comment j'ai organisé Claude Code en chaîne de production (spécifier, construire, vérifier, livrer), et pourquoi ce qui me plaît le plus dans cette usine, c'est qu'elle sait s'arrêter.*

## Le problème n'est plus de faire écrire du code

Faire écrire du code à un agent n'est plus vraiment le sujet. Ce qui m'intéresse, c'est tout ce qu'il y a autour : qui décide de ce qu'il faut construire, qui vérifie que c'est bien ce qui a été construit, et surtout qui a le droit de dire « non, ça ne passe pas ».

Prenons un agent seul, dans un seul contexte. Il spécifie, il code, il écrit ses propres tests, puis il annonce que tout est vert. Il est juge et partie. La qualité du modèle n'y change rien, c'est un problème d'organisation. Dans une équipe, on ne laisse pas l'auteur d'un code en être le seul relecteur. Je ne vois aucune raison de faire autrement avec des agents.

J'ai donc construit un harnais au-dessus de Claude Code. C'est une usine logicielle multi-agents qui s'installe dans n'importe quel projet. Elle tourne en session interactive comme en CI headless, et elle se met à jour dans les projets déjà équipés. Dans cet article, je décris comment elle est faite, les choix qui la structurent et ce qu'elle m'a appris. J'y raconte aussi le jour où elle a refusé de livrer son propre noyau.

## Trois principes

1. **Séparer les rôles et les contextes.** Un agent planifie, un autre construit, un troisième vérifie. Chacun a son propre contexte, et le vérificateur part d'une page blanche.
2. **La spec est un contrat.** Rien ne se construit sans spécification approuvée. Chaque critère d'acceptation doit pouvoir être vérifié par quelqu'un qui n'a que le dépôt sous les yeux.
3. **Le code avant la prose.** Tout ce qui ne demande pas de jugement est du code Python testé, pas une consigne dans un prompt : compteurs, lecture des verdicts, fusion de fichiers, conditions d'arrêt.

La suite de l'article développe ces trois idées.

## Vue d'ensemble

Le point d'entrée est un skill, `/factory-run`, exécuté par la session principale. Cette session ne fait qu'orchestrer : son prompt lui interdit d'écrire elle-même du code ou une spec. Elle délègue le travail à trois sous-agents et tient les *gates*, c'est-à-dire les points où une décision humaine peut être demandée.

```text
/factory-run "ajouter l'export CSV des rapports"
        │
        ▼
┌──────────────────┐
│ factory-planner  │  lecture seule : Read, Glob, Grep
└────────┬─────────┘
         │ brouillon de spec + questions ouvertes
         ▼
  [gate questions] [gate approbation]   ← moi, Product Owner
         │
         │ specs/SPEC-*.md — Status: APPROVED
         ▼
┌──────────────────┐
│ factory-builder  │◄───────────────┐  contexte neuf, tous les outils
└────────┬─────────┘                │
         │ code + tests, build vert │  issues (3 itérations max)
         ▼                          │
┌──────────────────┐                │
│ factory-verifier │────────────────┘  contexte vierge, ne modifie rien
└────────┬─────────┘  CHANGES_REQUESTED
         │ APPROVED + tests_passed
         ▼
  [gate livraison] → commit sur factory/<slug> → /ship-pr
```

L'installation passe par le système de plugins de Claude Code :

```bash
claude plugin marketplace add <url-du-dépôt>
```

```bash
claude plugin install harness@harness
```

Ensuite, dans le projet à équiper :

```text
/software-factory
```

Le skill détecte la stack (C#, Python, Node, C, Docker, Ansible…). Il lit les manifestes pour trouver les vraies commandes de build, de test et de lint, sans jamais en inventer. Puis il génère :

| Fichier généré | Rôle |
|---|---|
| `.claude/agents/factory-planner.md` | Analyse le besoin et le code, rédige la spec (lecture seule) |
| `.claude/agents/factory-builder.md` | Implémente strictement la spec approuvée, build vert exigé |
| `.claude/agents/factory-verifier.md` | Vérification adversariale, verdict JSON, ne modifie jamais rien |
| `.claude/skills/factory-run/SKILL.md` | L'orchestrateur |
| `ci/factory.sh`, `ci/factory.ps1` | Point d'entrée CI headless |
| `.claude/settings.json` | Règles `permissions.deny` et hooks (fusionnés, jamais écrasés) |
| `.vscode/tasks.json` | Tâches VS Code (fusionnées) |

Un détail que j'aime bien : les règles `permissions.deny` dépendent de la stack. Sur un projet .NET, `Read(./**/bin/**)` et `Read(./**/obj/**)` sont refusés d'office. Les agents ne dépensent donc pas de tokens à lire des artefacts de build.

En CI, le même cycle tourne sans interface :

```bash
ci/factory.sh specs/SPEC-export-csv.md
```

Le script refuse toute spec qui ne porte pas `Status: APPROVED`, car un cycle CI exige une spec validée par le Product Owner. Il lance ensuite `claude -p "/factory-run '<spec>'" --output-format json --max-turns 100` en `--permission-mode acceptEdits`, avec une liste blanche d'outils. L'option `--yolo` passe en `--dangerously-skip-permissions` ; je la réserve à un runner jetable.

## Trois agents, trois contrats

| Agent | Outils | Contexte | Interdits |
|---|---|---|---|
| Planner | `Read`, `Glob`, `Grep` | Le besoin et le code existant | Inventer une exigence, modifier un fichier, proposer une implémentation détaillée |
| Builder | Tous | Neuf ; la spec fait autorité | Élargir le périmètre ; désactiver, supprimer ou assouplir un test ; commiter, `git add`, `git stash`, changer de branche |
| Verifier | `Read`, `Glob`, `Grep`, `Bash` | Vierge, indépendant du Builder | Modifier un fichier ; approuver sans avoir lancé les tests ; approuver un fichier qu'il n'a pas lu |

Trois de ces choix méritent une explication.

**Le Planner s'occupe du *quoi* et du *comment on vérifie*, jamais du *comment on fait*.** S'il proposait une implémentation, le Builder la suivrait au lieu de lire le code. Et tout ce que le besoin ne tranche pas va dans une section *Open questions*, au lieu d'être décidé en douce.

**Le Builder conçoit et code dans le même contexte.** J'aurais pu séparer un « architecte » et un « développeur ». Je ne l'ai pas fait, parce que couper la conception de l'implémentation crée surtout des incohérences au passage de relais. Séparer les contextes n'a d'intérêt que là où cela crée une indépendance de jugement : entre celui qui construit et celui qui vérifie.

**Le Verifier n'a pas le droit de survoler.** Une règle de son prompt vise directement les approbations à l'aveugle. Si le lot est trop gros pour être relu en entier, le Verifier ne peut pas répondre `APPROVED`. Il doit répondre `CHANGES_REQUESTED` avec l'issue « batch too large to verify ». Un Verifier qui n'a pas lu n'a rien vérifié.

## La spec comme contrat

Voici à quoi ressemble une spec produite par le Planner. C'est un exemple illustratif sur un projet .NET :

```markdown
Status: PROPOSED
Blocking questions: none
Files scope: src/Reporting/**, tests/Reporting.Tests/**
Priority: 2

# SPEC — Export CSV des rapports

## Context
Les rapports ne sont consultables qu'à l'écran (`ReportService`).

## Scope
Un exporteur CSV pour `MonthlyReport`, exposé par `ReportController`.

## Acceptance criteria
- [ ] AC1 — `GET /reports/{id}/csv` répond `text/csv` ; vérifié par le test
      `ReportControllerTests.Csv_ReturnsCsvContentType` (`dotnet test`).
- [ ] AC2 — un champ contenant le séparateur, un guillemet ou un saut de ligne
      est entouré de guillemets ; test `ReportCsvExporterTests.Export_QuotesSpecialFields`.

## Out of scope
Export Excel, export planifié.

## Risks
Rapports volumineux : l'export doit être écrit en flux, pas en mémoire.

## Open questions
- Q1 — Séparateur : `;` (Excel en locale FR) ou `,` ?
  Default: `;`, cohérent avec les imports existants.
```

Le haut du fichier est un en-tête machine, lu par le noyau :

- `Status:` suit un cycle de vie contrôlé. Une spec `PROPOSED` devient `APPROVED` ou `BLOCKED`, une spec `APPROVED` ne peut devenir que `DELIVERED`.
- `Blocking questions:`, `Files scope:`, `Depends on:` et `Priority:` complètent l'en-tête.

Le reste suit des règles strictes. Chaque critère d'acceptation doit être **objectivement vérifiable** : un fichier à un chemin précis, un test nommé qui passe, ou un comportement observable pour une entrée donnée. « Le code est propre » est explicitement banni.

Les questions ouvertes sont classées. Une question est **bloquante** quand la construction ne peut pas commencer sans réponse et qu'aucune option par défaut n'est défendable. Dans tous les autres cas, la question doit porter une ligne `Default:` justifiée. C'est cette distinction qui rend l'autonomie possible, on y revient plus bas.

Enfin, `Files scope:` n'est pas décoratif : pendant la construction, un hook le fait respecter.

## Le Verifier, ou l'art de ne pas croire sur parole

Le Verifier termine sa réponse par un unique bloc JSON :

```json
{"verdict": "CHANGES_REQUESTED", "tests_passed": true, "issues": [{"file": "src/Reporting/ReportCsvExporter.cs", "severity": "major", "description": "Un saut de ligne dans un champ n'est pas protégé : la ligne CSV est coupée en deux."}]}
```

Ce n'est pas l'orchestrateur qui lit ce bloc, c'est du Python. Le bloc doit clore la réponse. Le verdict doit appartenir à un ensemble fermé, `tests_passed` doit être un vrai booléen, et chaque issue doit porter une sévérité `critical`, `major` ou `minor`. Si un texte suit le bloc, la sortie est inexploitable, et une sortie inexploitable est traitée comme un agent mort.

Le cycle ne réussit que si le verdict est `APPROVED` **et** que `tests_passed` vaut `true`. Voici un extrait simplifié de la machine à états (`hcore/factory.py`) :

```python
if parsed["verdict"] == "APPROVED" and parsed["tests_passed"] is True:
    state["status"] = "success"
    return {"action": "success", "iteration": iteration, "tests_passed": True}, log
if counters.cap_reached(iteration, state["max_iterations"]):
    state["status"] = "failed"
    return {"action": "fail", "iteration": iteration,
            "reason": "%d iterations without approval: explicit failure" % iteration,
            "issues": parsed["issues"]}, log
state["iteration"] = iteration + 1
state["expect"] = BUILDER
return {"action": "fix", "iteration": iteration + 1, "issues": parsed["issues"]}, log
```

Trois règles encadrent la boucle :

- **Trois itérations au maximum.** Après la troisième sans approbation, le cycle échoue explicitement et publie les issues restantes. Pas de quatrième tentative silencieuse, pas d'élargissement du périmètre « pour que ça passe », jamais d'approbation par épuisement.
- **Un agent mort est relancé une seule fois**, sans consommer d'itération. S'il échoue une deuxième fois, le cycle échoue explicitement.
- **Un Verifier muet n'est pas un Verifier satisfait.** Le prompt de l'orchestrateur le dit sans détour : « Un cycle qui livre du code non vérifié parce que son vérificateur est tombé est pire qu'un cycle qui échoue. »

Cette dernière règle est le cœur du système. Dans un pipeline d'agents, le mode de défaillance le plus dangereux n'est pas le mauvais verdict, c'est le verdict absent pris pour un feu vert. Une erreur d'API, une connexion coupée ou une sortie qui ne finit pas par un JSON produisent le même silence qu'un « rien à signaler ». Un orchestrateur optimiste transforme ce silence en feu vert.

## Le code avant la prose

Dans la première version, mes skills étaient de longs prompts Markdown. Avant de refondre, j'ai fait un inventaire, et il a été instructif :

- la règle « agent mort » existait en **sept** copies ;
- la détection de stack existait en **six** copies, avec des listes de marqueurs divergentes ;
- **onze** points de décision humaine avaient **quatre** comportements headless différents ;
- le `factory-run` généré dans le dépôt lui-même avait dérivé de son modèle. Il avait perdu précisément la règle du « Verifier muet ».

Un prompt n'est pas une source de vérité. On le copie, on le retouche, on le résume, et il dérive. Toutes les règles déterministes sont donc passées dans `skills/_core/` : du Python standard (3.9 ou plus), sans aucune dépendance, exposé par une seule CLI, `harness.py`. Cette CLI est copiée dans chaque projet équipé, sous `.claude/harness/`. Les prompts ne gardent que le jugement : le rôle, le périmètre, les interdits et le format attendu. L'objectif était chiffré : chaque skill rebranché devait perdre au moins un tiers de ses lignes sans perdre une seule règle.

Dans le prompt de l'orchestrateur, `H` désigne `python .claude/harness/harness.py`. Le dialogue avec le noyau ressemble à ceci :

```bash
H factory-run start --arguments-file .harness/runs/factory-run-arguments.txt
```

```bash
H factory-run record --run <run> --agent factory-verifier --output .harness/runs/verifier-1.md --duration-ms 84000 --tokens 41000
```

```json
{"action": "fix", "iteration": 2, "tests_passed": true, "issues": ["…"]}
```

Chaque commande renvoie un objet JSON dont le champ `action` dicte la suite, et le prompt l'énonce clairement : ne jamais sauter un appel, ne jamais décider à la place du noyau. Le LLM ne compte pas les itérations, il lit `action`.

Un deuxième principe est né d'un vrai bug, raconté plus bas. Aucun texte que je ne maîtrise pas ne passe par une ligne de commande shell : ni le besoin, ni une sortie d'agent, ni un corps de PR. Ce texte est écrit dans un fichier sous `.harness/runs/` par l'outil Write, puis transmis par son chemin. Le noyau supprime le fichier dès qu'il l'a lu. Des backticks, un `$()` ou une apostrophe dans un besoin ou un finding n'ont rien d'exotique : il y en a dans presque tous les messages techniques.

Enfin, chaque cycle tient un journal, `.harness/runs/factory-run-<horodatage>.jsonl`. Pour chaque étape, il consigne l'agent, le modèle, la durée, l'itération, le verdict et les tokens quand ils sont disponibles. Ce sont des faits, sans interprétation. Une mesure manquante est notée `null` et signalée, sans jamais bloquer le cycle. Je me suis fixé tôt une règle : mesurer avant d'étendre.

## Autonomie : `gate` ou `auto`

Ma première version demandait trop de validations humaines, et les skills s'interdisaient de s'enchaîner. Dans la v2, l'autonomie est une politique déclarée : la clé `autonomy` de `.claude/harness.json` vaut `gate` ou `auto`, et l'option `--autonomy` la surcharge pour une exécution.

Chaque gate est typée :

- **(a)** décision irréversible ou coûteuse ;
- **(b)** décision de goût ou de priorité ;
- **(c)** simple confirmation.

En `gate`, tout est demandé. En `auto`, les gates (b) et (c) prennent leur valeur par défaut et **journalisent** la décision. Chaque défaut est justifié en une ligne dans un catalogue. Les gates (a), elles, sont évaluées sur leur condition d'arrêt. Quelques exemples de défauts :

- pour les questions ouvertes, le `Default:` déclaré par le Planner ;
- pour l'approbation de la spec, « approuvé », puisqu'une question bloquante aurait déjà bloqué la spec ;
- pour la livraison, « livrer et lancer `ship-pr` ».

En `auto`, je n'approuve donc plus chaque spec : la protection s'est déplacée. Si le Planner déclare une question bloquante (un champ explicite de l'en-tête, pas une impression), la spec passe en `BLOCKED` et le cycle s'arrête proprement. J'ajoute ma réponse dans la spec sur une ligne `Answer:`, puis `/factory-run specs/SPEC-x.md` reprend le cycle.

Même en `auto`, la boucle s'arrête dans ces cas :

- spec bloquée ;
- trois itérations sans approbation ;
- agent mort deux fois ;
- dérive de périmètre répétée ;
- opération destructive ;
- secret touché ;
- processus enfant en échec ;
- gate sans valeur par défaut ;
- profil de projet ambigu.

Chaque arrêt écrit un fichier `.harness/runs/STOP-<horodatage>.md` de 40 lignes au plus, lisible en trente secondes. Il émet aussi un événement dans le journal et sort avec le code 1. Le gabarit est volontairement sec :

```markdown
# STOP — {{skill}} — {{kind}}

## Where
- skill, phase, run, spec, branch, worktree

## Why
{{why}}

## To resume
{{steps}}

## Command
{{command}}
```

Une fois un run arrêté, toute commande ultérieure sur ce run est refusée. Et le dépôt du harnais reste lui-même en `gate` : le harnais ne se modifie pas lui-même en autonomie.

## Des garde-fous qui ne sont pas des consignes

Une règle écrite dans un prompt reste un vœu. Un hook est une garantie, ou presque. `/software-factory` installe des hooks en Python standard dans `.claude/settings.json`, par fusion additive : mes propres hooks et mes commentaires sont conservés.

- **Le filet Bash/PowerShell** est permanent et s'applique quelle que soit la politique. Il bloque cinq catégories : `push --force`, suppression de branche, réécriture d'historique, destruction de travail non commité et suppression de fichiers hors artefacts générés. Il découpe les commandes sur `;`, `&&`, `||`, `|` et `&`, et descend dans `$( )`, les backticks, `bash -c`, `sudo`, `env` ou `xargs`. Il assume ses limites dans son propre code : *« a net, not a proof »*. `eval`, les variables et les alias ne sont pas analysés. Quand il bloque, son message m'invite à lancer la commande moi-même dans un terminal si elle est voulue.
- **Le périmètre pendant `/factory-run`** : une écriture hors du `Files scope` de la spec est bloquée par un code de sortie 2 et un message. L'agent reçoit le message et continue. À la deuxième dérive dans le même cycle, le cycle s'arrête.
- **Le périmètre pendant `/ship-pr`** : les éditions sont limitées aux fichiers de `git diff --name-only origin/<cible>...HEAD`.
- **Le linter** : si le projet en configure un, un hook `PostToolUse` le lance après chaque écriture, sans jamais bloquer.

Un hook de périmètre permanent bloquerait mes propres éditions. Ces hooks ne sont donc actifs que pendant un cycle. En début de cycle, l'orchestrateur écrit un marqueur `.harness/runs/active/<session_id>.json`, et il le retire à la fin. Un marqueur dont le run n'est plus actif, ou qui a plus de douze heures, est ignoré.

Enfin, une erreur interne dans un hook laisse passer l'action et le signale. Un garde-fou bogué ne doit jamais bloquer la session.

## Au-delà d'un cycle

**`/ship-pr <cible>`** prend en charge la vie complète d'une pull request :

1. Il vérifie les prérequis, puis crée la PR ou reprend celle qui existe.
2. Il enchaîne jusqu'à trois rondes de revue. À chaque ronde, trois relecteurs en lecture seule travaillent en parallèle : logique et cas limites, régressions et sites d'appel, cohérence du diff.
3. Chaque finding est validé par un mini-agent adversarial avant d'être posté en commentaire inline.
4. Les corrections sont commitées, à raison d'un commit par ronde, sans jamais d'`amend` ni de `push --force`.
5. Chaque thread reçoit une réponse avec le hash du correctif, puis il est résolu.

Le prompt de chaque relecteur contient mot pour mot la phrase *« Zero findings is a normal and acceptable answer »* : c'est l'antidote au finding inventé pour remplir un quota. Un relecteur mort deux fois laisse son thème « non couvert », et un thème non couvert bloque le merge.

En `auto`, le merge se fait sans question si cinq conditions sont réunies :

- aucun finding validé restant ;
- tous les threads résolus ;
- CI verte, s'il y en a une ;
- branche cible non protégée ;
- aucun conflit.

La décision est postée en commentaire **avant** le merge. Si une seule condition est fausse, la PR reste ouverte et la raison est écrite dans le commentaire.

**`/factory-queue run`** dépile les specs dans l'ordre topologique de `Depends on:`, puis selon `Priority:`. Chaque spec part dans un processus enfant headless (`claude -p "/factory-run <spec> --ship <cible>"`) avec un contexte neuf, et sa PR est mergée avant la suivante. Aucun état supplémentaire n'est nécessaire : tout se déduit de `Status:`, de git et des journaux.

**`/factory-batch specs/SPEC-*.md --parallel 3`** construit plusieurs specs en parallèle : 2 par défaut, 4 au maximum. Chaque spec a son propre worktree git, créé par le harnais sous `<dépôt>.harness-wt/`. Les livraisons se font ensuite une par une : rebase, `ship-pr`, merge, nettoyage. Un conflit non trivial arrête le lot et conserve le worktree.

Le harnais ne touche jamais à mes propres worktrees. Il n'agit que sur un chemin à la fois présent dans son registre **et** listé par `git worktree list --porcelain`, avec la branche enregistrée. Il ne lance jamais `prune`, `remove --force` ni `move`.

**`/audit`** referme la boucle :

1. Il lance un auditeur en lecture seule par lentille : hygiène, portabilité, injection, contrôle d'accès, maintenabilité…
2. Chaque finding est validé de façon adversariale, puis les findings sont priorisés.
3. Il produit un `AUDIT.md` unique, et des specs au format de l'usine, prêtes pour `/factory-queue run`.

## Profils, tampon et mises à jour

Le logiciel n'est pas le seul cas d'usage. Un **profil** est un dossier de données, sans code : un `profile.json` et des fragments Markdown qui décrivent les rôles, les lentilles d'audit, l'orchestrateur et les critères de détection. Deux profils sont livrés :

- `software`, l'usine décrite dans cet article ;
- `research`, un laboratoire de recherche en IA avec un directeur de recherche, des experts neurosciences et mathématiques, un Builder et un Verifier.

Ajouter un profil revient à ajouter un dossier, et `harness.py profile validate` le contrôle. À l'installation, un agent éclaireur en lecture seule, sur un modèle économique, collecte des indices. Le noyau vérifie ces preuves et calcule un score. Une proposition nette est retenue d'office en `auto`, une proposition ambiguë arrête l'installation.

Le modèle de chaque rôle se règle dans `.claude/models.json`. Par exemple :

```json
{
  "factory.planner": "opus",
  "factory.builder": "sonnet",
  "factory.verifier": "opus"
}
```

Les valeurs possibles sont `inherit`, `opus`, `sonnet`, `haiku`, `fable` ou un identifiant `claude-*` complet. On peut surcharger un rôle pour un seul cycle avec `--models "factory.verifier=opus"`. Un rôle ou une valeur inconnus produisent une erreur explicite, jamais un repli silencieux.

Pour les mises à jour, chaque fichier généré commence par un en-tête de version, suivi de **blocs délimités** : `harness.frontmatter`, `harness.body`, `harness.profile.<nom>` et `harness.ci`. Tout ce qui est hors bloc m'appartient. Un tampon, `.claude/harness.json`, enregistre la version, les profils actifs, la politique d'autonomie et l'empreinte des blocs.

`/software-factory --upgrade` procède ainsi :

- il ne régénère que les blocs ;
- il affiche le diff complet ;
- il applique tout ou rien, avec une sauvegarde ;
- il est idempotent : relancé sans changement, il produit un diff vide ;
- il refuse un projet installé par une version plus récente du harnais.

`/harness-doctor --scan ~/projets` affiche une ligne par projet équipé, avec la version installée et le retard.

## Tester un harnais d'agents

Comment tester un système dont une partie des composants sont des LLM ? En testant tout ce qui n'en est pas.

- **Le noyau** : 789 tests unitaires en `unittest` standard au moment où j'écris ces lignes.
- **Les scénarios de bout en bout** (`ci/test-harness.sh`) rejouent, dans des dossiers temporaires et sur des projets jouets, la séquence d'appels au noyau prescrite par les skills. Les agents sont simulés par des sorties figées, mais le Verifier lance les **vrais** tests du projet jouet. `gh` est remplacé par une imitation qui s'appuie sur un dépôt bare local. Des shims placés en tête du `PATH` comptent tout appel au vrai `gh` ou au vrai `claude`, et le compteur doit rester à zéro.
- **Les scénarios couverts** : installation à neuf, mise à jour d'un projet installé à l'ancienne avec ses personnalisations préservées, cycle complet en `auto`, arrêt propre à la troisième itération, merge automatique et ses refus, hook qui bloque hors du diff, entre autres.
- **Le mode `--live`** lance le vrai `claude -p`, borné par `--max-turns` et une liste blanche d'outils, avec des assertions structurelles uniquement. Il ne tourne jamais par défaut, puisqu'il consomme l'abonnement.

Pourquoi pas `claude plugin eval` ? Parce que ses correcteurs n'exécutent pas de commande shell, alors qu'ici chaque résultat attendu est un état observable : un code de sortie, le contenu exact d'un fichier, l'état de git ou un journal. L'outil reste intéressant pour des évaluations jugées par LLM, comme la qualité d'une spec.

## Le jour où l'usine a refusé de livrer son propre noyau

La v2 a été construite par l'usine elle-même. La feuille de route a été découpée en specs, et chaque incrément a suivi le même chemin : un cycle `/factory-run`, une branche, un diff montré, mon accord, puis le merge.

Le premier incrément comprenait le noyau Python, le journal et le rebranchement de `software-factory` et de `ship-pr`. Il s'est terminé en **échec explicite** après trois itérations. Pourtant, les dix-huit critères d'acceptation passaient, 133 tests étaient verts et `tests_passed` valait `true`. Les deux premières itérations avaient déjà fait corriger dix-huit défauts. Le Verifier répondait quand même `CHANGES_REQUESTED`, avec deux défauts majeurs :

1. **Aucun verrou sur le fichier d'état d'un run.** Des appels concurrents de relecteurs et de validateurs perdaient des mises à jour. Sur huit essais, sept produisaient des collisions d'identifiants et huit perdaient un thème de revue. Sous Windows, certains levaient une `PermissionError` sur `os.replace`. La conséquence possible : un verdict appliqué au mauvais finding, puis posté sur la PR.
2. **Un message de commit qui passait par le shell.** Le résumé de ronde de `ship-pr`, tiré des findings, était passé entre guillemets doubles sur une ligne de commande. Un backtick ou un `$()` dans un finding aurait été exécuté. Le commit `fix(review)` aurait alors été corrompu sans retour possible, puisque l'`amend` est interdit.

C'est exactement le cas pour lequel j'ai construit cette usine : critères verts, tests verts, et pourtant un défaut que seul quelqu'un qui cherche à casser trouve. Le Builder avait fait son travail, puisque la spec ne parlait pas de concurrence. Le Verifier, lui, cherchait à casser. Et la boucle a fait ce qu'elle devait faire : pas de quatrième tentative, pas d'approbation complaisante, mais un échec explicite, avec les issues, et la décision qui me revient.

Deux correctifs en sont sortis :

- un verrou exclusif au niveau du système d'exploitation par run (`fcntl.flock` ou `msvcrt.locking`, libéré par l'OS si son détenteur meurt), avec un test qui lance 4 relecteurs puis 12 validateurs en sous-processus parallèles ;
- la généralisation des fichiers de transit pour tout texte non maîtrisé.

J'ai retenu deux autres leçons, plus prosaïques :

- `claude plugin validate` a refusé le nom `claude-skills`, car un plugin tiers ne peut pas avoir un nom qui commence par `claude-`. D'où le nom `harness`.
- Le cache des plugins est indexé par `version`. Si l'on modifie les skills sans incrémenter la version, `claude plugin update` répond *« already at the latest version »*. Une garde en CI échoue désormais quand `skills/` change sans incrément de version.

## Limites

- **Le coût.** Un cycle mobilise au moins trois agents et jusqu'à trois itérations Build/Verify, puis les relecteurs et validateurs de `ship-pr`. Pour corriger une faute de frappe, c'est disproportionné. Le Planner doit rester proportionné (un besoin trivial tient en une vingtaine de lignes), mais toute la chaîne tourne quand même.
- **Le filet n'est pas une preuve.** Le garde-fou Bash le dit lui-même.
- **La spec borne la vérification.** Un critère flou produit une vérification floue. Le Planner a des contraintes, mais le Product Owner, c'est moi.
- **Le mode `auto` déplace l'effort, il ne le supprime pas.** Je ne valide plus chaque étape, mais chaque décision automatique est journalisée. Lire les journaux et les fichiers `STOP` fait désormais partie de mon travail.

## Ce que je garderais si je repartais de zéro

- L'indépendance de jugement ne paie que là où elle compte : entre construire et vérifier.
- Une spec aux critères testables est la seule interface raisonnable entre un humain et une chaîne d'agents.
- Tout ce qui est déterministe doit être du code.
- Un échec explicite vaut mieux qu'un succès silencieux.

Une usine qui sait s'arrêter vaut mieux qu'une usine qui livre toujours.
