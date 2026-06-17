# Module replay testing — findings

Branch under test: `pr13-replay-integration` (CLI built from `tools/cli/bin/run`).
Repro assets:
- Module `cm-test-base` (GitHub `samutoljamo/cm-test-base`, tags v1.0.0–v2.0.0), built by `scripts/build-base.sh`.
- Consumer `projects/happy-base`.

Both bugs were left **unfixed** by request; documented here with repro + root cause.

---

## Bug 1 — `git-manager` `.env()` wipes the child environment, breaking HTTPS git ops

**Severity:** medium. Breaks `cyberismo publish` and remote version listing over HTTPS for any
user whose credentials come from a git credential helper (the normal case, e.g. `gh auth setup-git`).

**Symptom**
```
cyberismo publish
→ fatal: could not read Username for 'https://github.com': terminal prompts disabled
```
Remote version listing (`check-updates` / `update-modules`) can likewise hang or fail.

**Root cause**
`tools/data-handler/src/utils/git-manager.ts`:
- line 174 (`push`):           `this.git.env({ ...NON_INTERACTIVE_GIT_ENV }).push(args)`
- line 186 (`listRemoteVersionTags`): `git.env({ ...NON_INTERACTIVE_GIT_ENV }).listRemote(...)`

simple-git's `.env(object)` **replaces** the child process environment instead of extending it.
`NON_INTERACTIVE_GIT_ENV` (git-config.ts:17) is only `{ GIT_TERMINAL_PROMPT: 0, GCM_INTERACTIVE: 'never' }`,
so the spawned git loses `HOME` and `PATH` — it can't read `~/.gitconfig`, so the configured
credential helper never runs, and with prompts disabled it errors with "could not read Username".

**Suggested fix (one line each)**
```ts
.env({ ...process.env, ...NON_INTERACTIVE_GIT_ENV })
```

**Workaround used in these tests**
`scripts/build-base.sh` does not call `cyberismo publish`; it replicates publish with system git
(`git tag -a … && git push --force origin <tag>`), which uses the configured helper correctly.

---

## Bug 2 — Replay can't migrate consumer cards affected by >1 breaking change in the chain

**Severity:** high. A module update whose chain contains multiple breaking changes touching the
same consumer card fails (and rolls back), so the card data is never migrated. This is the core
scenario the replay system exists for.

**Repro**
```
# base published with versions 1.0.0..2.0.0 (see scripts/build-base.sh)
cyberismo create project "Happy Consumer" cons projects/happy-base -s
cd projects/happy-base && git init && import module <base-url>@1.0.0
cyberismo create card base/templates/Task            # -> card of cardType base/cardTypes/Task
# seed the card: priority="low", estimate=5 (number), workflowState="Deprecated"
# set module range so update spans multiple breaking versions, e.g. version: "^1.0.0"
cyberismo --autocommit update-modules base
```

**Observed — two manifestations, one root cause**

1. Range `^1.0.0` (→ 1.3.0, chain includes the v1.3.0 cardType rename `Task`→`WorkItem`):
   replay "succeeds" but migrates **nothing**, then post-validation fails:
   ```
   Card 'cons_…' has invalid state 'Deprecated'
   field 'base/fieldTypes/priority' is 'enum' … possible: medium, high, critical   (card still 'low')
   field 'base/fieldTypes/estimate' is 'shortText' but it is 'number' value 5
   → ModuleValidationFailedError, transaction rolled back
   ```

2. Range `>=1.0.0 <1.3.0` (→ 1.2.0, chain has NO cardType rename):
   replay fails **during the first seal's enum cascade**:
   ```
   Module replay failed … seal 'migrationLog_1.0.0_1.1.0.jsonl', entry 1 …:
     field 'base/fieldTypes/estimate' is 'shortText' but it is 'number' value 5
     Card 'cons_…' has invalid state 'Deprecated'
   → ModuleReplayFailedError, transaction rolled back
   ```

**Root cause**
The module tree is advanced to the **final target version up front** (module files overwritten
before `executeModuleReplays` runs), then seals are replayed **incrementally** in version order.
Two consequences:

- **Premature whole-card validation (manifestation 2).** Each migration write goes through
  `Project.updateCardMetadataKey` → `validateCard(card)` (project.ts:1180), which validates the
  *entire* card against the *final* module resource definitions. When the first seal migrates one
  field (e.g. enum `low`→`medium`), the card is still invalid for the *other* fields that later
  seals haven't migrated yet (`estimate` is still a number though the def is now `shortText`;
  `workflowState` is `Deprecated` though that state was removed). Validation throws → the first
  migration can never land. A card touched by N breaking changes can't be migrated because each
  write is validated against the end-state.

- **Card-set join misses renamed card types (manifestation 1).** The value cascades select
  affected cards by joining consumer cards to module card-type *names*, e.g.
  `field-type-enum-remove.ts affectedCards()`:
  ```
  relevant = cardTypes().filter(ct => ct.customFields.includes(field)).map(ct => ct.name)
  cards.filter(c => relevant.has(c.metadata.cardType))
  ```
  At the final tree the card type is `base/cardTypes/WorkItem`, but the consumer card is still
  `base/cardTypes/Task` (the rename seal runs later in the chain), so it never matches and no
  value migration is attempted. (This is why manifestation 1 migrates nothing rather than failing
  mid-cascade.)

**What works (scoped confirmation)**
- The engine's mechanics are sound: range resolution (`^1.0.0`→1.3.0), seal-chain computation,
  ascending replay order, and the `--autocommit` transactional **rollback** on failure all behave
  correctly. The failure is specifically in applying multi-step card-data migrations against a
  pre-advanced module tree.

**Possible directions (not implemented)**
- Advance the module tree **per seal** (stage at each intermediate version) so cascades/validation
  see the resource state that matches the migration step, not the end-state.
- Or defer/relax per-write `validateCard` during replay and validate once at the end (the final
  `ModuleValidationFailedError` check already exists).
- Or make value cascades match cards by field reference independent of the (possibly-renamed)
  card-type name, and apply card-type renames to consumer cards before value cascades.

---

## Incidental notes (environment, not product bugs)

- `isModulePath` (card-utils.ts:210) matches `/modules/` anywhere in an absolute path, so a project
  located under a directory literally named `modules` is misclassified (e.g. `add card` to a local
  template is refused). Latent edge case; avoided here by using `repos/` for the module checkouts.
- `--autocommit` cleans untracked empty dirs; an empty `cardRoot` vanishes unless git is
  initialized (with a tracked placeholder) before the first autocommit op.
- Field-type rename is refused in authoring when a *local* card type references the field
  (`field-type-rename.ts:44`) — by design (replay rewrites the reference; authoring does not).
  A module can only seal a field rename for a field not referenced by its own card types.
