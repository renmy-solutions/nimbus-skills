# nimbus-skills

Curated agent skills for working on Apex with **[Renmy](https://renmy.net)** (formerly Nimbus) — the local Apex test runner.

Skills are short, opinionated playbooks that tell an AI coding agent **when** and **how** to use Renmy's local runtime effectively. Each is a tight loop around a signal only Renmy produces offline — real test results, per-line coverage, governor usage, execution traces, mutation scores.

**Getting set up**
- **`bootstrap-nimbus`** — set Renmy up on a fresh SFDX project, including CI.
- **`migrate-suite-to-nimbus`** — get an existing org suite running green locally by clearing blockers in dependency order (schema → stubs → setup → fixtures).

**Writing & fixing code**
- **`apex-tdd`** — red → green → refactor; the only way to actually do TDD in Apex without an org.
- **`fix-failing-apex-test`** — the inner loop. Read failure → narrow → edit → re-run, until green.
- **`triage-apex-failures`** — cluster a wall of failures by root cause before fixing.

**Raising quality**
- **`apex-coverage-uplift`** — raise coverage by writing targeted tests, not theatre.
- **`harden-tests-with-mutation`** — kill surviving mutants so assertions actually check behaviour.
- **`bulkify-apex`** — find and prove fixes for governor-limit / bulk-DML problems with measured usage.
- **`apex-pre-deploy-check`** — diff-scoped run + coverage delta + governor + risky-pattern gate before you ship.

All skills assume the project has Renmy installed and (for the MCP-driven loops) the MCP server registered with the agent (`claude mcp add nimbus -- renmy mcp`). The skills call the `renmy` command; versions before 1.50.0 only answer to `nimbus`, so use that name wherever a skill says `renmy` if you are on one.

## Install

### Recommended: `renmy skills install`

If you've already installed [Renmy](https://renmy.net), one command picks up everything in this repo and writes it to the right place for your agent:

```bash
renmy skills list                           # see what's available
renmy skills install fix-failing-apex-test  # install one
renmy skills install all                    # install every skill the agent supports
```

`renmy skills` auto-detects your agent from the project root. Override with `--agent claude-code|cursor|aider|agents-md|kiro`. The CLI fetches manifest + files from this repo on demand — your binary stays small and you always get the latest.

### Manual install (no renmy binary)

#### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/renmy-solutions/nimbus-skills.git /tmp/nimbus-skills
cp -r /tmp/nimbus-skills/dist/claude-code/* ~/.claude/skills/
```

Or per-project: copy into `.claude/skills/` at the project root.

#### Cursor

```bash
mkdir -p .cursor/rules
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/cursor/fix-failing-apex-test.mdc -o .cursor/rules/fix-failing-apex-test.mdc
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/cursor/bootstrap-nimbus.mdc -o .cursor/rules/bootstrap-nimbus.mdc
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/cursor/apex-coverage-uplift.mdc -o .cursor/rules/apex-coverage-uplift.mdc
```

#### Kiro

```bash
mkdir -p .kiro/steering
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/kiro/fix-failing-apex-test.md -o .kiro/steering/fix-failing-apex-test.md
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/kiro/bootstrap-nimbus.md -o .kiro/steering/bootstrap-nimbus.md
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/kiro/apex-coverage-uplift.md -o .kiro/steering/apex-coverage-uplift.md
```

Skills use `inclusion: manual`, so load them on demand by typing `#fix-failing-apex-test` (or another name) in your Kiro prompt.

#### Aider

```bash
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/aider/CONVENTIONS.md -o CONVENTIONS.md
aider --read CONVENTIONS.md
```

#### AGENTS.md (OpenCode, Codex, Continue, generic harnesses)

```bash
curl -L https://github.com/renmy-solutions/nimbus-skills/raw/main/dist/AGENTS.md -o AGENTS.md
```

OpenCode reads `AGENTS.md` verbatim — no further config needed.

## Repo layout

```
skills/                  # Source of truth — edit these
  <name>/SKILL.md        # one dir per skill (9 today)
dist/                    # Generated; do not edit
  claude-code/<name>/SKILL.md
  cursor/<name>.mdc
  kiro/<name>.md
  aider/CONVENTIONS.md
  AGENTS.md
manifest.json            # Generated; consumed by `renmy skills`
scripts/build.mjs        # Renders dist/ + manifest.json from skills/
```

The build script is dependency-free Node — `node scripts/build.mjs` regenerates `dist/` and `manifest.json`. CI verifies both stay in sync with `skills/` on every PR.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). New skills are welcome if they target a real Apex workflow that benefits from sub-second iteration.

## License

MIT.

---

Built for [**Renmy**](https://renmy.net) — Apex testing, locally.
