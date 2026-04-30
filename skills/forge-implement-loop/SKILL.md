---
name: forge-implement-loop
description: Forge a portable /implement skill (user-global by default; works across any repo). Writes a SKILL.md that runs a build → validate → fix → report loop. The produced skill reads per-repo config (spec source, validators, doc paths) from `.claude/implement.json` and bootstraps it on first run in any new repo, so the skill itself is repo-agnostic. Invoke ONLY when the user explicitly types the slash command or asks in plain language to set up/scaffold/forge an /implement skill — e.g. "set up /implement", "forge an implement skill". Do NOT auto-invoke. Do NOT invoke to perform an implementation.
---

# forge-implement-loop

Forge a portable, user-global /implement skill. The produced skill is **repo-agnostic**: repo-specific knobs (spec source, validators, doc paths) live in `.claude/implement.json` per repo, and /implement bootstraps that config the first time it runs in a new repo.

The implement loop has three phases:
- **Pre-loop** — load and understand the spec
- **On-loop** — build → validate → fix, iteratively
- **Post-loop** — report results and (optionally) check docs

The following are **assumed** and baked into the produced skill — do not ask about them:
1. Big implementations are split into steps; each step is validated separately before moving on.
2. Subagents are used whenever it helps context isolation (e.g. fresh context for the build, code review, or slow validators so contexts stay uncorrelated).
3. Red/green TDD: failing test first, smallest change to pass, refactor.

The forge interview is intentionally short — it captures **user-level preferences only**. Anything that varies by repo (which build/lint/test commands, which tracker, which doc paths) is gathered by /implement at runtime when it bootstraps the per-repo config. This keeps the produced skill stable across every project the user has.

## Phase 1 — Report template

Show this template:

```markdown
# Implementation Report — <spec title>

## Summary
<one paragraph: what was implemented and how>

## Scenarios from spec
| # | Scenario | Status | Validated by |
|---|----------|--------|--------------|
| 1 | <name>   | ✓ / ✗  | <validator(s)> |

## Validators run
| Validator | Command | Result |
|-----------|---------|--------|
| <name>    | `<cmd>` | ✓ / ✗ / ⊘ |

## Deferred
- <requirement or scenario not done this round, with reason>

## Outstanding failures
- <validators or requirements that didn't pass and weren't fixed>
```

Ask if the user wants to adjust it. Capture the final form as `{{REPORT_TEMPLATE}}`. The report MUST show every scenario from the spec with its status and the validator that confirmed it.

## Phase 2 — Skill destination

Use AskUserQuestion (single-select):

**Where should the produced SKILL.md be written?**

- *user-global* — `~/.claude/skills/implement/SKILL.md` (default; portable across all of the user's projects).
- *project-local* — `.claude/skills/implement/SKILL.md` (relative to the current project root). Pick this only if the team wants a shared, repo-checked variant.
- *custom* — user supplies a different absolute or repo-relative directory; the skill goes to `<dir>/implement/SKILL.md`.

Capture the resolved absolute path as `{{SKILL_PATH}}`. If it already exists, show the path and ask whether to overwrite or pick a different location.

## Output

Write the produced SKILL.md to `{{SKILL_PATH}}`. Create parent directories if missing. Substitute `{{REPORT_TEMPLATE}}`.

````markdown
---
name: implement
description: Implement a spec by looping build → validate → fix, then report. Reads per-repo config (spec source, validators, doc paths) from `.claude/implement.json`; if missing, bootstraps it on first run via a short interview. Run from the project root. Invoke ONLY when the user explicitly types the slash command or asks in plain language to implement a spec — e.g. "/implement <id>", "implement the <name> spec". Do NOT auto-invoke. Do NOT invoke to plan, discuss, or write a spec.
---

You are implementing a spec. The loop has three phases: pre-loop (load), on-loop (build/validate/fix), post-loop (report/docs).

## Working principles

- **Run from the project root.** This skill expects the current working directory to be the repo it implements against. If the cwd doesn't look like a project root (no manifest, no `.git`), stop and tell the user to re-run from the project root.
- **Step-by-step for big work.** If the spec is large, split into steps. Validate each step before moving on. Don't batch one giant change.
- **Red/green TDD.** Failing test first, watch it fail, smallest change to pass, refactor.
- **Subagents for context isolation.** Spawn a fresh subagent for distinct concerns — build, code review, slow validators — so contexts stay uncorrelated and the main loop stays focused.

## Phase 0 — Load (or bootstrap) repo config

Look for `.claude/implement.json` at the repo root.

**If it exists**, load it. It must have these keys:

```json
{
  "specSource": {
    "type": "tracker | file-path | paste",
    "tracker": "Linear | Jira | GitHub | ADO | …",
    "defaultDir": "specs/"
  },
  "validators": [
    { "name": "build", "command": "npm run build", "bucket": "each-iteration" },
    { "name": "unit-tests", "command": "npm test", "bucket": "each-iteration" },
    { "name": "ai-code-review", "command": "subagent:code-reviewer", "bucket": "before-report" }
  ],
  "docs": { "check": true, "paths": ["README.md", "docs/"] }
}
```

If keys are missing or malformed, tell the user which keys are missing and offer to re-run the bootstrap.

**If it does not exist**, bootstrap it now. First detect what's wired up so the question lists real options:

- package manifests: `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, etc.
- lint configs: `.eslintrc*`, `ruff.toml`, `.golangci.yml`, `.rubocop.yml`
- test frameworks and scripts in those manifests
- build / type-check commands (`tsc`, `mypy`, `cargo build`)
- already-installed skills (check `~/.claude/skills/` and project `skills/`) — note any code-review or browser-automation skill already present

Then ask three questions:

### Q1 — Spec source

Use AskUserQuestion (single-select):

**How is the spec supplied to /implement?**
- *tracker-id* — argument is a ticket ID; fetch via MCP/CLI. Capture **which tracker** (Linear, Jira, GitHub Issues, ADO, …).
- *file-path* — argument is a path in the repo. Capture default location if any (e.g. `specs/`).
- *paste* — user pastes the spec text into the chat.

### Q2 — Validators

Use AskUserQuestion (multi-select). Build the option list from what was detected above, plus these standard categories:

- *build* — e.g. `npm run build`, `tsc --noEmit`, `cargo build`
- *lint* — e.g. `eslint .`, `ruff check`, `golangci-lint run`
- *unit-tests* — fast, run every loop
- *integration-tests* — slower, usually before-report
- *type-check* — if separate from build
- *ai-code-review* — if picked: check whether a code-review skill already exists; if not, propose Anthropic's `code-reviewer` agent and capture the user's choice.
- *manual-smoke-test* — if picked: ask whether it's browser-based; if yes, propose the `agent-browser` skill; otherwise capture explicit click/curl steps.
- *deterministic-scan* — SonarQube, Snyk, semgrep, security scanners. Capture command.
- *other* — free text for anything not in the list.

For each chosen validator, capture: **name**, **command (or manual instruction)**, and **bucket** — `each-iteration` for fast checks, `before-report` for slow ones.

### Q3 — Doc check

Use AskUserQuestion (single-select):

**Should /implement check whether docs need updating after the loop?**
- *yes-readme* — scan top-level README and `docs/` for stale references.
- *yes-custom* — capture specific paths to check.
- *no* — skip the doc check.

### Write the config

Write the answers to `.claude/implement.json`. Create the directory if missing. Show the resulting JSON to the user, tell them they can edit it later, and proceed to Phase 1.

## Phase 1 — Pre-loop: load the spec

The spec argument is `$ARGUMENTS`. Use `specSource` from the config to load it:

- `tracker`: fetch the ticket via the available MCP/CLI tool for `specSource.tracker`. If unavailable, ask the user to paste the spec.
- `file-path`: read the file. If `$ARGUMENTS` is empty and `specSource.defaultDir` is set, look there first.
- `paste`: ask the user to paste the spec text into the chat.

Read the spec end to end **before** writing any code. If sections are missing or too vague, stop and ask. Do not guess.

Then plan and summarize back:
- requirements you'll satisfy (by number)
- scenarios you'll cover
- files you expect to touch
- constraints you'll respect
- anything you're explicitly **not** doing this round, with reason

Wait for confirmation if the plan is non-trivial. For small changes, proceed.

## Phase 2 — On-loop: build, validate, fix

For each step:
1. Write the failing test (red).
2. Make the smallest change to make it pass (green).
3. Run **each-iteration validators** (those with `bucket: "each-iteration"` in the config).
4. Fix what broke. Repeat until clean.
5. Move to the next step.

A failed validator is not optional. Fix it, downgrade the change, or stop and ask.

For slow validators or AI code review, spawn a subagent so the main loop's context stays uncluttered.

Before producing the report, run **before-report validators** (those with `bucket: "before-report"`). For AI code review, run via subagent. For browser smoke tests using `agent-browser`, invoke that skill. For manual instructions, print the instruction to chat and wait for user confirmation.

## Phase 3 — Post-loop: report and docs

### Report

Print this report. Every scenario from the spec MUST appear with its status and the validator that confirmed it.

```markdown
{{REPORT_TEMPLATE}}
```

### Doc check

If `docs.check` is true:
- if `docs.paths` is set, scan those paths.
- otherwise scan `README.md` and `docs/` at the repo root.

Flag any stale references to the user. Do not edit docs without confirmation.

If `docs.check` is false, skip.
````

After writing the file, print the path of the new skill to the user. Tell them: the skill is portable — running `/implement` in a new repo will trigger a one-time bootstrap that writes `.claude/implement.json` for that repo, after which subsequent runs use it directly.
