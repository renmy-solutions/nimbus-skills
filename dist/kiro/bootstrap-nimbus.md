---
inclusion: manual
---

# Bootstrap Renmy on an SFDX project

You are configuring Renmy on a Salesforce DX project. The goal is to reach a green `renmy test "*"` run with the project's existing Apex test suite, then leave the user with a CI snippet they can drop in.

## Preconditions

1. The project has a `sfdx-project.json` at the root. If not, this is not an SFDX project — stop and tell the user Renmy targets SFDX projects.
2. The Apex source is under `force-app/main/default/classes/` (or another path declared in `sfdx-project.json` `packageDirectories`).
3. The user has Renmy installed (`renmy --version`; versions before 1.50.0 only answer to `nimbus`). If not, point them at https://renmy.net/quickstart and stop — installation is a manual step.

## Steps

### 1. Verify the project shape

Read `sfdx-project.json`. Confirm `packageDirectories` and note the `default` directory. If multiple package directories exist, mention that to the user — Renmy runs all of them, but they should be aware which classes get loaded.

### 2. Initialise

```
renmy init
```

This creates `.nimbus/` (config + embedded Postgres data dir). It is idempotent. If it errors with a port conflict, ask the user whether another Renmy instance is running.

### 3. Run the doctor

```
renmy doctor --json
```

Read the JSON output. The doctor reports on: parser errors, missing schema, unresolved class references, and config sanity. Group findings into:

- **Blockers** (parser errors, hard schema gaps): must be fixed before tests will run.
- **Warnings** (unresolved managed-package classes): can usually be papered over with stubs.
- **Info** (config defaults): leave alone unless user asks.

Report blockers to the user before continuing. Do not attempt to silence parser errors by editing source — they signal a real Apex syntax issue or a Renmy parser gap (in which case ask the user to file an issue).

### 4. Sync schema (if the project queries an org's metadata)

If the doctor reports missing SObjects or fields, the project depends on org metadata Renmy does not have. Ask the user for an org alias (`sf org list` shows their aliases). Then:

```
renmy sync -o <alias>
```

If only specific SObjects are needed, narrow with `-s Account,Contact,Lead`. A full sync can take a minute on large orgs; a narrow sync is seconds.

### 5. Handle managed-package gaps

If the doctor reports unresolved classes from managed packages (e.g. `dlrs.RollupService`, `fflib_SObjectDomain`), generate stubs. On Pro:

```
renmy test "*" --write-stubs
```

This runs the suite and writes stub class skeletons under `stubs/` for any unresolved reference. The first run will likely fail; re-run to see if stubs unblock the suite.

On Free, point the user at the `stubs/` directory and `renmy stub add <ClassName>` to scaffold one stub at a time. Do not auto-generate by hand-writing stub files — the layout is specific.

### 6. First green run

```
renmy test "*" --json
```

Parse the JSON. Three outcomes:

- **All green.** Continue to step 7.
- **A handful of failures.** These are real test failures, not bootstrap problems. Hand off to the `fix-failing-apex-test` skill (or invite the user to triage). Do not silence them.
- **Wholesale failure** (most tests fail with the same error). Re-read the doctor output — schema or stub gap was probably not fully resolved. Loop back to step 3.

### 7. Register the MCP server

```
claude mcp add nimbus -- renmy mcp
```

This is what makes Renmy callable from Claude Code, Cursor, and other MCP-aware agents. For Cursor, point the user at their MCP settings and the same `renmy mcp` stdio command.

### 8. CI snippet

Ask the user which CI provider they're on. Common choices: GitHub Actions, GitLab CI, Bitbucket Pipelines, CircleCI. Generate the smallest possible snippet:

For GitHub Actions, write `.github/workflows/renmy-test.yml`:

```yaml
name: Apex Tests
on: [pull_request, push]
jobs:
  renmy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Renmy
        run: curl -fsSL https://install.renmy.net | sh
      - name: Run Apex tests
        run: renmy test "*" --results-xml renmy-junit.xml --coverage-report renmy-coverage.xml
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: renmy-results
          path: |
            renmy-junit.xml
            renmy-coverage.xml
```

Confirm the install URL with the user — it may have moved.

## Hard rules

- **Never delete or rewrite the user's Apex source** to make Renmy parse it. If parsing fails, that's a Renmy or Apex-version mismatch — surface it.
- **Never sync to an org the user did not explicitly name.** Schema sync writes to the local DB; using the wrong org pollutes the workspace.
- **Never check `.nimbus/` into git.** Add it to `.gitignore` if it isn't already.
- **Do not configure Pro features** (daemon, write-stubs, watch mode) without first confirming the user has a Pro license. Free-tier users will hit license errors.

## Verification before declaring done

1. `renmy test "*"` exits 0.
2. `.nimbus/` is present and `.gitignore` excludes it.
3. The user has a CI snippet committed (or knows where it lives).
4. `claude mcp list` (or equivalent) shows `nimbus` registered, if the user wants the MCP integration.
