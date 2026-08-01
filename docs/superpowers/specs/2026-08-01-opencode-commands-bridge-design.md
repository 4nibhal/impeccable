# OpenCode native slash commands — design spec

**Status:** Draft for review. Local work on a fork; no upstream push.
**Date:** 2026-08-01
**Branch:** `opencode/commands-bridge`
**Worktree:** `~/.worktrees/opencode-commands`
**Fork:** `4nibhal/impeccable` (upstream: `pbakaus/impeccable`)

## Problem

Impeccable installs cleanly into OpenCode 1.18.10 (the skill is discovered,
`opencode debug skill` lists it), but `/impeccable <subcommand>` does not
appear in the slash menu and never runs as a user-invoked command. The
README and `docs/DEVELOP.md` still promise `/impeccable init` for every
supported harness, so OpenCode users hit a silent gap between install and
first use. The `pin.mjs` script that creates `/audit`, `/polish`, etc. for
Claude and Cursor produces a SKILL.md that OpenCode never surfaces as a
slash command either.

Root cause is documented in `docs/HARNESSES.md` (current `OpenCode`
frontmatter row overstates support) and verified against OpenCode source
in `opencode/packages/opencode/src/command/index.ts:134-152` and
`opencode/packages/core/src/v1/config/command.ts:5-13`.

## Goals

1. `/impeccable <args>` works in OpenCode TUI, autocomplete, and CLI
   (`opencode run --command impeccable`).
2. `/impeccable-audit <args>` and other sub-commands work the same way
   when the user runs `impeccable pin <sub>` on an OpenCode project.
3. Generated artifacts respect OpenCode's command schema (template,
   description, agent, model, subtask) without breaking other harnesses.
4. Install/update migration mirrors the safety guarantees PR #417
   established for global skills (path precedence, symlink safety,
   migration of legacy `~/.opencode/commands/` siblings).
5. `docs/HARNESSES.md` accurately documents the OpenCode support matrix.

## Non-goals (this iteration)

- Native OpenCode agents (`.opencode/agents/`). Different schema,
  separate effort, follows once command bridge lands.
- Custom tools (`.opencode/tools/*.ts`). Useful for `impeccable-detect`,
  `impeccable-doctor`, `impeccable-live-status` but they wrap existing
  scripts without unlocking `/impeccable`.
- OpenCode plugin (`.opencode/plugin/impeccable.ts`). Needed only for
  `tool.execute.before` hook that runs the design detector on every
  edit. Other harnesses carry hooks; OpenCode is documented as
  "no documented hook surface" in `docs/HARNESSES.md:75`. We do not
  change that here.
- Any change to upstream `pbakaus/impeccable`.

## Design

### 1. Generated command artifacts

For every Impeccable install into an OpenCode project, the build emits
`<install-root>/.opencode/commands/impeccable.md`. Single file, fixed
content, no per-sub-command proliferation.

```markdown
---
description: Impeccable design workflow — bridge to the impeccable skill (one parent skill, 23 sub-commands)
agent: build
subtask: true
---
Load the `impeccable` skill via the skill tool, then execute the user's
request. Setup is mandatory: first run `node .opencode/skills/impeccable/scripts/context.mjs`,
load the matching `reference/<routing or sub-command>.md`, and follow the
skill's instructions. $ARGUMENTS holds the user's sub-command plus
optional target.

If the user wrote only `/impeccable`, read `reference/routing.md` and
present the context-aware menu; never auto-run a sub-command.
```

Why one command, not 24:

- Avoids slash-menu pollution (project policy in `CLAUDE.md:13`).
- Centralises the route into `routing.md` and the per-sub-command
  reference files that already exist.
- Matches the architecture OpenCode itself uses for skill bridges
  (`command/index.ts:134-152` produces one wrapper per skill).

### 2. `pin.mjs` for OpenCode

`pin <command>` writes `<root>/.opencode/commands/impeccable-<command>.md`
into the project. Existing behaviour for every other harness
(`.claude/skills/<cmd>/SKILL.md`, `.cursor/skills/<cmd>/SKILL.md`, etc.) is
unchanged. The OpenCode branch is opt-in by harness detection: only when
`<root>/.opencode/skills/impeccable/` already exists.

Pinned command file:

```markdown
---
description: Impeccable sub-command shortcut — runs the <display name> workflow via /impeccable <command>
agent: build
subtask: true
---
Run `/impeccable <command>` and follow its instructions. $ARGUMENTS
holds any arguments the user typed after the pinned command name.
```

`unpin <command>` removes `<root>/.opencode/commands/impeccable-<command>.md`
only (other harnesses untouched).

### 3. CLI install/update — `copyProviderCommands`

`cli/bin/commands/skills.mjs` gains a `copyProviderCommands(bundleDir, root, targets, { scope, home })`
mirroring the existing `copyProviderSkills`. It reads
`<bundle>/.opencode/commands/*.md` and copies into:

- **Project scope:** `<root>/.opencode/commands/`.
- **Global scope (project scope is `--scope=user`):**
  `opencodeGlobalConfigDir(home) + '/commands'`, resolved the same way
  as global skills (issue #406): `OPENCODE_CONFIG_DIR` →
  `$XDG_CONFIG_HOME/opencode` → `~/.config/opencode`.
- **Global install migration:** drop the just-written command names from
  any stranded `~/.opencode/commands/` legacy copy, with the same
  symlink / dotfiles / realpath guards `copyProviderSkills` uses
  (`skills.mjs:1168-1186`).

`update()` and `install()` routes call `copyProviderCommands` after
`copyProviderSkills`, alongside `copyProviderAgents` (already exists for
`.github` and `.cursor`, `skills.mjs:1202-1213`).

### 4. Build — emit command file

`scripts/build.js` runs every provider's transformer. The OpenCode
transformer factory at `scripts/lib/transformers/factory.js:217-260`
emits `dist/opencode/.opencode/commands/impeccable.md` next to
`dist/opencode/.opencode/skills/impeccable/SKILL.md`. Content is
derived from `SKILL.src.md` constants: the description mirrors the
skill's frontmatter description; the body reuses text already present
in `SKILL.src.md:42-83`.

### 5. Documentation

`docs/HARNESSES.md` updates:

- Frontmatter support row for OpenCode: keep only spec-compliant
  fields (`name`, `description`, `license`, `compatibility`,
  `metadata`). Mark `user-invocable`, `argument-hint`, `model`,
  `agent`, `disable-model-invocation`, `allowed-tools` as unsupported.
- Command surface row: add `OpenCode | .opencode/commands/ | $ARGUMENTS,
  $1-$N`. Mirror Gemini's row format (`docs/HARNESSES.md:130-134`).
- Skill Directory Structure row: drop the misleading "Reads `.opencode/`"
  and `~/.opencode/commands/` from the Also Reads column.
- Hook surface table: keep OpenCode as "No documented hook surface".

`README.md` line 5 stays unchanged; the install flow already matches.

## Implementation outline (TDD order)

Per `AGENTS.md:55-65` (provider-build TDD) and `AGENTS.md:97-102`
(skill-style TDD applied to a generator):

1. **Fixture:** `tests/fixtures/commands/impeccable.md.expected` —
   exact bytes the transformer must produce.
2. **Red:** `tests/lib/transformers/opencode.test.js` asserts the fixture
   exists in `dist/opencode/.opencode/commands/impeccable.md`.
3. **Green:** extend `factory.js` to write the command file alongside
   the SKILL.md. Keep the generator pure (string-in / string-out) so
   `providers.test.js` stays generic.
4. **Red:** `tests/skills-cli.test.js` for `copyProviderCommands`:
   default global path, `OPENCODE_CONFIG_DIR`, `XDG_CONFIG_HOME`,
   legacy `~/.opencode/commands/` migration with sibling preservation,
   project-scope copy, symlink safety.
5. **Green:** new `copyProviderCommands` in `skills.mjs`, called from
   `install()` / `update()` after `copyProviderSkills`.
6. **Red:** `tests/build.test.js` updates: existing per-provider loop
   already asserts SKILL.md; extend the OpenCode row to assert the
   command artifact.
7. **Green:** `pin.mjs` OpenCode branch with `tests/skills-cli.test.js`
   style coverage in `tests/build.test.js` (no test file currently
   exercises `pin.mjs`; add a fixture-driven assertion in
   `tests/pin.test.js` if it makes the suite green; otherwise rely on
   `tests/lib/transformers/opencode.test.js` to keep the contract).
8. **Doc:** update `docs/HARNESSES.md`; refresh the matrix with the
   accurate OpenCode row.
9. **Verify:** `bun test tests/build.test.js tests/skills-cli.test.js
   tests/lib/transformers/opencode.test.js tests/lib/transformers/providers.test.js`,
   then `bun run build` and inspect `dist/opencode/.opencode/commands/`.
10. **Local smoke:** symlink the bundle into a scratch project and
    invoke `opencode run --command impeccable "audit hero"` plus a
    simulated `/impeccable polish src/cli` (via TUI or `--command`).

## Tests (estimated +18)

- `tests/lib/transformers/opencode.test.js`: +5 cases covering
  description mirroring, command body, `subtask: true`, no argument
  hint frontmatter, no schema deviation.
- `tests/skills-cli.test.js`: +8 cases mirroring the four #406 skills
  tests plus four pin cases (project-scope, global, legacy migration,
  symlink safety).
- `tests/build.test.js`: +3 cases asserting the new file lands in
  every relevant `dist/opencode/.opencode/commands/` after build.
- `tests/lib/transformers/providers.test.js`: +2 cases confirming
  OpenCode row now emits both skill and command.

Lines: +180 (code) / +240 (tests). Total diff ~420 lines, fits in one PR
per `AGENTS.md` policy.

## Compatibility

- New code only adds artifacts; existing harness outputs untouched.
- `providerConfig.opencode` (`providers.js:91-97`) gains no new keys;
  `factory.js` gains an unconditional post-skill command emitter. If a
  future OpenCode update rejects the command format, the failing
  build assertion in `tests/lib/transformers/opencode.test.js` will
  surface the regression.
- The bridge command (`/impeccable`) does not collide with the
  existing implicit bridge that OpenCode creates from a `user-invocable`
  skill (`command/index.ts:134-152`): explicit commands outrank the
  implicit bridge; if the user installs a future harness that adds
  per-sub-command auto-bridge, our single bridge remains compatible
  because explicit commands still win.

## Out-of-scope follow-ups

- `pin.mjs` could detect a user's `~/.opencode/commands/` legacy copy
  and migrate silently. PR #417 did the same for skills; we mirror
  the same risk model without expanding scope here.
- OpenCode hooks (`tool.execute.before`) would close the detector gap.
  Tracked separately; we keep `docs/HARNESSES.md:75` honest.

## Open questions for the user

1. **Bridge body wording.** The current proposal tells the LLM to call
   `skill({name:'impeccable'})` first. Acceptable, or do you prefer
   a stricter body that hard-codes the `node .../context.mjs`
   invocation? I lean toward the soft "load the skill, then run
   the user's request" version because hard-coding scripts in a
   command template is fragile across paths.
2. **Pinned command naming.** I propose `impeccable-<command>`
   (e.g. `impeccable-audit`). Alternative: `<command>` to match Claude's
   behavior (a `pin audit` already produces `/audit` via `user-invocable`).
   I lean toward `impeccable-<command>` for OpenCode because plain
   `<command>` is more likely to collide with future OpenCode built-ins
   (e.g. `/review` already exists in OpenCode today).
3. **Global scope for commands.** PR #417 added migration for global
   skills. Should the same safety apply to commands? I lean yes —
   the cost of the safety net is one realpath + one lstat check.

## Rollback

- Removing the file from `factory.js` and reverting the
  `copyProviderCommands` call in `skills.mjs` fully restores the
  current behaviour. Build artifacts revert to "skill only".
- `pin.mjs` reverts to its existing branch list; the OpenCode branch
  is removed by deleting the `case '.opencode'` block.
- Documentation is additive; reverting keeps the table accurate.
