# The Claude software factory: three agents, one contract and the right to fail

*How I organized Claude Code into a production line (specify, build, verify, ship), and why the thing I like most about this factory is that it knows how to stop.*

## Getting an agent to write code is no longer the problem

Getting an agent to write code isn't really the topic anymore. What interests me is everything around it: who decides what should be built, who checks that what was built is what was asked for, and above all who gets to say "no, this doesn't pass".

Take a single agent in a single context. It specifies, codes, writes its own tests, then announces that everything is green. It's judge and jury. Model quality doesn't change that; it's an organizational problem. In a team, the author of a change isn't its only reviewer. I see no reason to do it differently with agents.

So I built a harness on top of Claude Code. It's a multi-agent software factory that installs into any project. It runs in an interactive session or headless in CI, and it can upgrade itself in projects that already have it. In this post I describe how it's built, the choices behind it and what it taught me. I also tell the story of the day it refused to ship its own core.

## Three principles

1. **Separate roles and contexts.** One agent plans, another builds, a third verifies. Each one has its own context, and the verifier starts from a blank page.
2. **The spec is a contract.** Nothing gets built without an approved specification. Every acceptance criterion must be checkable by someone who has nothing but the repository.
3. **Code over prose.** Anything that doesn't need judgment is tested Python code, not an instruction in a prompt: counters, verdict parsing, file merges, stop conditions.

The rest of this post develops these three ideas.

## The big picture

The entry point is a skill, `/factory-run`, run by the main session. That session only orchestrates: its prompt forbids it from writing code or a spec itself. It delegates the work to three subagents and holds the *gates*, the points where a human decision may be requested.

```text
/factory-run "add CSV export of the reports"
        │
        ▼
┌──────────────────┐
│ factory-planner  │  read-only: Read, Glob, Grep
└────────┬─────────┘
         │ draft spec + open questions
         ▼
  [questions gate] [approval gate]   ← me, Product Owner
         │
         │ specs/SPEC-*.md — Status: APPROVED
         ▼
┌──────────────────┐
│ factory-builder  │◄───────────────┐  fresh context, all tools
└────────┬─────────┘                │
         │ code + tests, green build│  issues (3 iterations max)
         ▼                          │
┌──────────────────┐                │
│ factory-verifier │────────────────┘  blank context, never edits
└────────┬─────────┘  CHANGES_REQUESTED
         │ APPROVED + tests_passed
         ▼
  [ship gate] → commit on factory/<slug> → /ship-pr
```

You install it through the Claude Code plugin system:

```bash
claude plugin marketplace add <repository-url>
```

```bash
claude plugin install harness@harness
```

Then, in the project you want to equip:

```text
/software-factory
```

The skill detects the stack (C#, Python, Node, C, Docker, Ansible…). It reads the manifests to find the real build, test and lint commands, and never invents one. Then it generates:

| Generated file | Role |
|---|---|
| `.claude/agents/factory-planner.md` | Analyses the need and the code, drafts the spec (read-only) |
| `.claude/agents/factory-builder.md` | Strictly implements the approved spec, green build required |
| `.claude/agents/factory-verifier.md` | Adversarial verification, JSON verdict, never edits anything |
| `.claude/skills/factory-run/SKILL.md` | The orchestrator |
| `ci/factory.sh`, `ci/factory.ps1` | Headless CI entry point |
| `.claude/settings.json` | `permissions.deny` rules and hooks (merged, never overwritten) |
| `.vscode/tasks.json` | VS Code tasks (merged) |

One detail I like: the `permissions.deny` rules depend on the stack. On a .NET project, `Read(./**/bin/**)` and `Read(./**/obj/**)` are denied from the start, so agents don't spend tokens reading build output.

In CI, the same cycle runs without a UI:

```bash
ci/factory.sh specs/SPEC-csv-export.md
```

The script refuses any spec that doesn't say `Status: APPROVED`, because a CI cycle needs a spec the Product Owner has signed off. It then runs `claude -p "/factory-run '<spec>'" --output-format json --max-turns 100` in `--permission-mode acceptEdits`, with a tool allowlist. The `--yolo` option switches to `--dangerously-skip-permissions`; I keep it for disposable runners.

## Three agents, three contracts

| Agent | Tools | Context | Forbidden |
|---|---|---|---|
| Planner | `Read`, `Glob`, `Grep` | The need and the existing code | Inventing a requirement, editing a file, proposing a detailed implementation |
| Builder | All | Fresh; the spec is authoritative | Widening the scope; disabling, deleting or loosening a test; committing, `git add`, `git stash`, switching branches |
| Verifier | `Read`, `Glob`, `Grep`, `Bash` | Blank, independent from the Builder | Editing a file; approving without running the tests; approving a file it hasn't read |

Three of these choices deserve an explanation.

**The Planner owns the *what* and the *how we check it*, never the *how we do it*.** If it proposed an implementation, the Builder would follow it instead of reading the code. And anything the need doesn't settle goes into an *Open questions* section instead of being quietly decided.

**The Builder designs and codes in the same context.** I could have split an "architect" from a "developer". I didn't, because separating design from implementation mostly creates inconsistencies at the handoff. Separating contexts only pays off where it creates independent judgment: between the one who builds and the one who verifies.

**The Verifier isn't allowed to skim.** One rule in its prompt goes straight at blind approvals. If the change is too large to be reread in full, the Verifier can't answer `APPROVED`. It must answer `CHANGES_REQUESTED` with the issue "batch too large to verify". A Verifier that hasn't read anything hasn't verified anything.

## The spec as a contract

Here is what a spec produced by the Planner looks like. It's an illustrative example on a .NET project:

```markdown
Status: PROPOSED
Blocking questions: none
Files scope: src/Reporting/**, tests/Reporting.Tests/**
Priority: 2

# SPEC — CSV export of the reports

## Context
Reports can only be viewed on screen (`ReportService`).

## Scope
A CSV exporter for `MonthlyReport`, exposed by `ReportController`.

## Acceptance criteria
- [ ] AC1 — `GET /reports/{id}/csv` answers `text/csv`; checked by the test
      `ReportControllerTests.Csv_ReturnsCsvContentType` (`dotnet test`).
- [ ] AC2 — a field containing the separator, a quote or a line break is
      enclosed in quotes; test `ReportCsvExporterTests.Export_QuotesSpecialFields`.

## Out of scope
Excel export, scheduled export.

## Risks
Large reports: the export must be streamed, not built in memory.

## Open questions
- Q1 — Separator: `;` (Excel with a French locale) or `,`?
  Default: `;`, consistent with the existing imports.
```

The top of the file is a machine header that the core parses:

- `Status:` follows a controlled lifecycle. A `PROPOSED` spec becomes `APPROVED` or `BLOCKED`, and an `APPROVED` spec can only become `DELIVERED`.
- `Blocking questions:`, `Files scope:`, `Depends on:` and `Priority:` complete the header.

The rest follows strict rules. Every acceptance criterion must be **objectively verifiable**: a file at a precise path, a named test that passes, or an observable behaviour for a given input. "The code is clean" is explicitly banned.

Open questions are classified. A question is **blocking** when construction can't start without an answer and no default option is defensible. In every other case, the question must carry a justified `Default:` line. This distinction is what makes autonomy possible, as we'll see below.

Finally, `Files scope:` isn't decoration: during the build, a hook enforces it.

## The Verifier, or the art of not taking anyone's word for it

The Verifier ends its answer with a single JSON block:

```json
{"verdict": "CHANGES_REQUESTED", "tests_passed": true, "issues": [{"file": "src/Reporting/ReportCsvExporter.cs", "severity": "major", "description": "A line break inside a field is not quoted: the CSV row is split in two."}]}
```

The orchestrator doesn't parse this block; Python does. The block must close the answer. The verdict must belong to a closed set, `tests_passed` must be a real boolean, and each issue must carry a `critical`, `major` or `minor` severity. If any text follows the block, the output is unusable, and an unusable output is treated as a dead agent.

The cycle only succeeds if the verdict is `APPROVED` **and** `tests_passed` is `true`. Here is a simplified excerpt of the state machine (`hcore/factory.py`):

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

Three rules frame the loop:

- **Three iterations at most.** After the third one without approval, the cycle fails explicitly and publishes the remaining issues. No silent fourth attempt, no widening the scope "to make it pass", never an approval by exhaustion.
- **A dead agent is relaunched once**, without using up an iteration. If it fails a second time, the cycle fails explicitly.
- **A silent Verifier is not a satisfied Verifier.** The orchestrator's prompt states it bluntly: "A cycle that delivers unverified code because its verifier went down is worse than a cycle that fails."

That last rule is the heart of the system. In an agent pipeline, the most dangerous failure mode isn't a wrong verdict, it's a missing verdict taken for a green light. An API error, a dropped connection or an output that doesn't end in JSON all produce the same silence as "nothing to report". An optimistic orchestrator turns that silence into a green light.

## Code over prose

In the first version, my skills were long Markdown prompts. Before the rewrite I took an inventory, and it was enlightening:

- the "dead agent" rule existed in **seven** copies;
- stack detection existed in **six** copies, with diverging marker lists;
- **eleven** human decision points had **four** different headless behaviours;
- the `factory-run` generated in the repository itself had drifted from its template. It had lost precisely the "silent Verifier" rule.

A prompt is not a source of truth. It gets copied, tweaked and summarized, and it drifts. So every deterministic rule moved into `skills/_core/`: standard-library Python (3.9 or later) with zero dependencies, exposed through a single CLI, `harness.py`. That CLI is copied into each equipped project under `.claude/harness/`. Prompts keep only judgment: the role, the scope, the prohibitions and the expected format. The target had a number on it: every rewired skill had to lose at least a third of its lines without losing a single rule.

In the orchestrator's prompt, `H` stands for `python .claude/harness/harness.py`. The dialogue with the core looks like this:

```bash
H factory-run start --arguments-file .harness/runs/factory-run-arguments.txt
```

```bash
H factory-run record --run <run> --agent factory-verifier --output .harness/runs/verifier-1.md --duration-ms 84000 --tokens 41000
```

```json
{"action": "fix", "iteration": 2, "tests_passed": true, "issues": ["…"]}
```

Each command returns a JSON object whose `action` field dictates what happens next, and the prompt says so plainly: never skip a call, never decide in the core's place. The LLM doesn't count iterations; it reads `action`.

A second principle came from a real bug, told further down. No text I don't control goes through a shell command line: not the need, not an agent's output, not a PR body. That text is written to a file under `.harness/runs/` by the Write tool, then passed by path. The core deletes the file as soon as it has read it. Backticks, a `$()` or an apostrophe in a need or a finding are nothing exotic: almost every technical message has some.

Finally, each cycle keeps a journal, `.harness/runs/factory-run-<timestamp>.jsonl`. For every step it records the agent, the model, the duration, the iteration, the verdict and the tokens when they're available. These are facts, not interpretation. A missing measurement is logged as `null` and flagged, without ever blocking the cycle. I set myself a rule early on: measure before you extend.

## Autonomy: `gate` or `auto`

My first version asked for too many human approvals, and the skills refused to chain into each other. In v2, autonomy is a declared policy: the `autonomy` key of `.claude/harness.json` is `gate` or `auto`, and the `--autonomy` option overrides it for a single run.

Every gate has a type:

- **(a)** an irreversible or costly decision;
- **(b)** a matter of taste or priority;
- **(c)** a plain confirmation.

In `gate`, everything is asked. In `auto`, (b) and (c) gates take their default and **log** the decision. Each default is justified in one line in a catalog. (a) gates are evaluated against their stop condition. A few examples of defaults:

- for open questions, the `Default:` the Planner declared;
- for spec approval, "approved", since a blocking question would already have blocked the spec;
- for delivery, "deliver and run `ship-pr`".

So in `auto` I no longer approve each spec: the safeguard has moved. If the Planner declares a blocking question (an explicit header field, not a feeling), the spec goes to `BLOCKED` and the cycle stops cleanly. I add my answer in the spec on an `Answer:` line, and `/factory-run specs/SPEC-x.md` resumes the cycle.

Even in `auto`, the loop stops in these cases:

- blocked spec;
- three iterations without approval;
- an agent dead twice;
- repeated scope drift;
- a destructive operation;
- a secret touched;
- a failed child process;
- a gate without a default;
- an ambiguous project profile.

Each stop writes a `.harness/runs/STOP-<timestamp>.md` file of at most 40 lines, readable in thirty seconds. It also emits an event in the journal and exits with code 1. The template is deliberately terse:

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

Once a run has stopped, every later command on that run is refused. And the harness repository itself stays in `gate`: the harness doesn't modify itself autonomously.

## Guardrails that aren't instructions

A rule written in a prompt is still a wish. A hook is a guarantee, or close to one. `/software-factory` installs standard-library Python hooks into `.claude/settings.json` through an additive merge: my own hooks and comments are kept.

- **The Bash/PowerShell net** is permanent and applies whatever the policy. It blocks five categories: `push --force`, branch deletion, history rewriting, destruction of uncommitted work and deleting files that aren't generated artifacts. It splits commands on `;`, `&&`, `||`, `|` and `&`, and recurses into `$( )`, backticks, `bash -c`, `sudo`, `env` or `xargs`. It states its limits in its own code: "a net, not a proof". `eval`, variables and aliases aren't analysed. When it blocks something, its message invites me to run the command myself in a terminal if I meant it.
- **The scope during `/factory-run`**: a write outside the spec's `Files scope` is blocked with exit code 2 and a message. The agent gets the message and carries on. On the second drift in the same cycle, the cycle stops.
- **The scope during `/ship-pr`**: edits are limited to the files of `git diff --name-only origin/<target>...HEAD`.
- **The linter**: if the project configures one, a `PostToolUse` hook runs it after every write and never blocks.

A permanent scope hook would block my own edits, so these hooks are only active during a cycle. At the start of a cycle, the orchestrator writes a marker, `.harness/runs/active/<session_id>.json`, and removes it at the end. A marker whose run is no longer active, or that is older than twelve hours, is ignored.

Finally, an internal error inside a hook lets the action through and says so. A buggy guardrail must never block the session.

## Beyond a single cycle

**`/ship-pr <target>`** handles the whole life of a pull request:

1. It checks the prerequisites, then creates the PR or picks up the existing one.
2. It runs up to three review rounds. In each round, three read-only reviewers work in parallel: logic and edge cases, regressions and call sites, consistency of the diff.
3. Each finding is validated by an adversarial mini-agent before being posted as an inline comment.
4. Fixes are committed, one commit per round, never with an `amend` or a `push --force`.
5. Each thread gets a reply with the fix's hash, then it's resolved.

Every reviewer's prompt contains, word for word, the sentence "Zero findings is a normal and acceptable answer": it's the antidote to findings invented to fill a quota. A reviewer that dies twice leaves its theme "uncovered", and an uncovered theme blocks the merge.

In `auto`, the merge happens without asking if five conditions hold:

- no validated finding left;
- every thread resolved;
- green CI, if there is one;
- an unprotected target branch;
- no conflict.

The decision is posted as a comment **before** the merge. If a single condition is false, the PR stays open and the reason is written in the comment.

**`/factory-queue run`** works through the specs in the topological order of `Depends on:`, then by `Priority:`. Each spec runs in a headless child process (`claude -p "/factory-run <spec> --ship <target>"`) with a fresh context, and its PR is merged before the next one starts. No extra state is needed: everything is derived from `Status:`, git and the journals.

**`/factory-batch specs/SPEC-*.md --parallel 3`** builds several specs in parallel: 2 by default, 4 at most. Each spec gets its own git worktree, created by the harness under `<repo>.harness-wt/`. Deliveries then happen one at a time: rebase, `ship-pr`, merge, cleanup. A non-trivial conflict stops the batch and keeps the worktree.

The harness never touches my own worktrees. It only acts on a path that is both in its registry **and** listed by `git worktree list --porcelain` with the recorded branch. It never runs `prune`, `remove --force` or `move`.

**`/audit`** closes the loop:

1. It runs one read-only auditor per lens: hygiene, portability, injection, access control, maintainability…
2. Each finding is validated adversarially, then the findings are prioritized.
3. It produces a single `AUDIT.md`, plus specs in the factory format, ready for `/factory-queue run`.

## Profiles, stamp and upgrades

Software isn't the only use case. A **profile** is a data folder with no code: a `profile.json` and Markdown fragments that describe the roles, the audit lenses, the orchestrator and the detection criteria. Two profiles ship with the harness:

- `software`, the factory described in this post;
- `research`, an AI research lab with a research director, neuroscience and maths experts, a Builder and a Verifier.

Adding a profile means adding a folder, and `harness.py profile validate` checks it. At install time, a read-only scout agent on a cheap model gathers evidence. The core checks that evidence and computes a score. A clear proposal is taken automatically in `auto`, and an ambiguous one stops the installation.

The model for each role is set in `.claude/models.json`. For example:

```json
{
  "factory.planner": "opus",
  "factory.builder": "sonnet",
  "factory.verifier": "opus"
}
```

Possible values are `inherit`, `opus`, `sonnet`, `haiku`, `fable` or a full `claude-*` identifier. You can override a role for a single cycle with `--models "factory.verifier=opus"`. An unknown role or value is an explicit error, never a silent fallback.

For upgrades, every generated file starts with a version header, followed by **delimited blocks**: `harness.frontmatter`, `harness.body`, `harness.profile.<name>` and `harness.ci`. Everything outside the blocks belongs to me. A stamp, `.claude/harness.json`, records the version, the active profiles, the autonomy policy and the block hashes.

`/software-factory --upgrade` works like this:

- it only regenerates the blocks;
- it shows the full diff;
- it applies all or nothing, with a backup;
- it's idempotent: run again with no change, it produces an empty diff;
- it refuses a project installed by a newer version of the harness.

`/harness-doctor --scan ~/projects` prints one line per equipped project, with the installed version and how far behind it is.

## Testing an agent harness

How do you test a system where some of the components are LLMs? By testing everything that isn't one.

- **The core**: 789 unit tests in plain `unittest` at the time of writing.
- **End-to-end scenarios** (`ci/test-harness.sh`) replay, in temporary folders and on toy projects, the sequence of core calls that the skills prescribe. Agents are simulated with frozen outputs, but the Verifier runs the toy project's **real** tests. `gh` is replaced by a fake that sits on a local bare repository. Shims at the front of the `PATH` count any call to the real `gh` or `claude`, and that count must stay at zero.
- **Scenarios covered**: fresh install, upgrade of a project installed the old way with its customizations preserved, a full cycle in `auto`, a clean stop at the third iteration, automatic merge and its refusals, a hook blocking outside the diff, among others.
- **The `--live` mode** runs the real `claude -p`, bounded by `--max-turns` and a tool allowlist, with structural assertions only. It never runs by default, since it uses up the subscription.

Why not `claude plugin eval`? Because its graders don't run shell commands, while every expected result here is an observable state: an exit code, the exact bytes of a file, the state of git or a journal. It's still worth a look for LLM-judged evaluations, like the quality of a spec.

## The day the factory refused to ship its own core

v2 was built by the factory itself. The roadmap was split into specs, and every increment followed the same path: a `/factory-run` cycle, a branch, a diff shown to me, my approval, then the merge.

The first increment covered the Python core, the journal and rewiring `software-factory` and `ship-pr`. It ended in **explicit failure** after three iterations. Yet all eighteen acceptance criteria passed, 133 tests were green and `tests_passed` was `true`. The first two iterations had already led to eighteen defects being fixed. The Verifier still answered `CHANGES_REQUESTED`, with two major defects:

1. **No lock on a run's state file.** Concurrent reviewer and validator calls lost updates. Out of eight attempts, seven produced ID collisions and all eight lost a review theme. On Windows, some raised a `PermissionError` on `os.replace`. The possible consequence: a verdict applied to the wrong finding, then posted on the PR.
2. **A commit message going through the shell.** The `ship-pr` round summary, derived from the findings, was passed in double quotes on a command line. A backtick or a `$()` in a finding would have been executed. The `fix(review)` commit would then have been corrupted with no way back, since `amend` is forbidden.

This is exactly the case I built the factory for: green criteria, green tests, and still a defect that only someone actively trying to break things finds. The Builder had done its job, since the spec never mentioned concurrency. The Verifier was trying to break things. And the loop did what it's meant to do: no fourth attempt, no lenient approval, but an explicit failure, with the issues, and the decision handed back to me.

Two fixes came out of it:

- an OS-level exclusive lock per run (`fcntl.flock` or `msvcrt.locking`, released by the OS if its holder dies), with a test that launches 4 reviewers then 12 validators as parallel subprocesses;
- transit files for any untrusted text, everywhere.

I took away two more lessons, more mundane ones:

- `claude plugin validate` refused the name `claude-skills`, because a third-party plugin name can't start with `claude-`. Hence the name `harness`.
- The plugin cache is keyed by `version`. If you change the skills without bumping the version, `claude plugin update` answers "already at the latest version". A CI guard now fails whenever `skills/` changes without a version bump.

## Limits

- **Cost.** A cycle involves at least three agents and up to three Build/Verify iterations, then the `ship-pr` reviewers and validators. To fix a typo, that's overkill. The Planner is asked to stay proportionate (a trivial need fits in about twenty lines), but the whole chain still runs.
- **The net isn't a proof.** The Bash guard says so itself.
- **The spec bounds the verification.** A vague criterion gets a vague check. The Planner has constraints, but I'm the Product Owner.
- **`auto` moves the effort, it doesn't remove it.** I no longer approve each step, but every automatic decision is logged. Reading journals and `STOP` files is now part of my job.

## What I'd keep if I started over

- Independent judgment only pays off where it matters: between building and verifying.
- A spec with testable criteria is the only sensible interface between a human and a chain of agents.
- Anything deterministic belongs in code.
- An explicit failure beats a silent success.

A factory that knows how to stop beats a factory that always ships.
