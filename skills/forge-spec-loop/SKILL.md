---
name: forge-spec-loop
description: Forge a portable /spec-feature or /spec-epic skill (user-global by default; works across any repo). Asks which level to forge, then writes a SKILL.md that runs a research → interrogate → refine loop on a raw prompt input, prints the result to chat, and copies it to the clipboard. Forges one skill per run — to get both, run twice. The produced skill reads the codebase from cwd at runtime — nothing repo-specific is baked in. Invoke ONLY when the user explicitly types the slash command or asks in plain language to set up/scaffold/forge a spec skill — e.g. "set up /spec", "forge spec skill". Do NOT auto-invoke. Do NOT invoke to write a spec.
---

# forge-spec-loop

The job of this skill is to forge **two paired loops** — `/spec-epic` (intent-level) and `/spec-feature` (behavior-level). Both share the same loop machinery; they differ only in template and loop emphasis. Each takes a raw prompt as input, sharpens it against whichever codebase it's invoked in via research → interrogate → refine, prints the result, and copies it to the clipboard. Anything beyond that — fetching from a tracker, writing to a file, posting back to a ticket — is the user's job to wire up.

The produced skills are **portable**: they do not bake in anything about a specific repo. They read the codebase from the current working directory at runtime, so the same skills work across every project the user has.

A spec describes **behavior or intent**, not implementation:
- **Epic-level** — *why* and *what success looks like*. Coarse, outcome-focused.
- **Feature-level** — *what behavior changes*. Concrete requirements + scenarios.

Each loop has three phases:

- **Pre-loop** — load the input (the raw `$ARGUMENTS` prompt)
- **On-loop** — research → interrogate → refine, iteratively, until the spec is sharp enough to hand off without follow-up questions
- **Post-loop** — print the result and copy it to the clipboard

The following are **assumed** and baked into the produced skill — do not ask about them:

1. The on-loop is a real loop: research → interrogate → refine, repeated. It ends when the spec is sharp enough to hand off without follow-up questions, not after a single template walk-through.
2. Use the level's template as places the loop touches, not a checklist. Later iterations often revisit earlier sections.
3. After every refine step, read the change back briefly and ask "anything missing?"
4. Specs describe behavior or intent. Reject implementation details unless they're hard constraints.
5. Once the loop has produced a sharp spec, look at the whole. If it has clean fault lines (disjoint outcome clusters at epic level, or separable scenario groups at feature level), propose splitting it before printing.
6. The codebase is a primary input. The on-loop's research step reads code, conventions, and existing behavior — that's how clarifying questions stay grounded instead of generic. The produced skills assume they run from the project root.
7. Each level's spec template ships to `references/template.md` inside the produced skill. The user edits it there later if they want changes — do **not** ask about templates during the interview.

## Level-specific loop emphasis

The two skills share the same machinery but push on different things:

- **Epic loop** — pushes on **outcomes** (are they measurable/observable?), **boundaries** (out-of-scope is half the value), and **breakdown** (does this epic split cleanly into N features?). Rejects scenario-level detail — that's feature-level.
- **Feature loop** — pushes on **requirement precision** (each independently testable?), **scenarios** (edge cases, failure modes), and **behavior-vs-implementation discipline**. Rejects re-litigating intent — that's epic-level.

## Interview flow — two questions

The interview asks **two** questions, in order:

1. **Level selection** — which single spec skill to forge: `/spec-feature` or `/spec-epic`. See "Level selection" below. **Ask this first.**
2. **Integrations** — input/output wiring for the produced skill(s). See "Integrations" below.

Everything else is decided automatically:

- **Destination** — `~/.claude/skills/spec-epic/SKILL.md` or `~/.claude/skills/spec-feature/SKILL.md` depending on the level choice. Tell the user the path **after** writing the files so they can move or rename it if they want.
- **Spec template** — the default template for the chosen level, written verbatim to `references/template.md` next to the produced SKILL.md. The user edits it there if they want changes.
- **Clipboard command** — detected from platform (`pbcopy` on macOS, `xclip -selection clipboard` on Linux, `clip.exe` on Windows).

Auto mode does not relax either question — ask both before writing files. Do not infer defaults.

The integration choice applies to the single forged skill.

## Level selection

**Always ask this first — even in auto mode.** Print a **plain chat prompt** (not `AskUserQuestion`) with exactly two options, then stop and wait for the user's reply. Same rationale as the Integrations question: `AskUserQuestion` auto-appends an "Other / Type something" slot, producing a minimum of 3 visible options. The user wants exactly 2.

Print this verbatim (or very close to it):

> **Which spec skill to forge?**
>
> 1. **Feature** — `/spec-feature` (behavior-level: Overview → Requirements → Scenarios → Constraints).
> 2. **Epic** — `/spec-epic` (intent-level: Problem → Outcomes → High-level requirements → Out of scope → Open questions).
>
> Forge one at a time. To get both, run this skill twice.

After printing, stop. Do not call any further tools. Wait for the user's reply.

If the user is clearly asking for a single spec skill without specifying a level (e.g. "set up /spec" with no qualifier), default to **Feature**.

Capture the choice as `{{LEVEL}}` — exactly one of `epic` or `feature`. The rest of the meta-skill writes files for that single level only.

## Integrations

**Always ask this — even in auto mode.** Print a **plain chat prompt** (not `AskUserQuestion`) with exactly two options, then stop and wait for the user's reply.

Why not `AskUserQuestion`: its schema requires `options` to have ≥2 labeled items and **always** auto-appends an "Other / Type something" free-form slot. The minimum visible count is therefore 3 (e.g. None / Custom / Type something). The user wants exactly 2 — labeled "None" and an "Open answer" with the hint `Describe input and/or output wiring`. A plain chat prompt is the only way to get that shape.

Print this verbatim (or very close to it) — the bullets are the two options, and the user's next message is their answer:

> **Should the forged skill integrate with anything? By default it reads `\$ARGUMENTS` and writes to chat + clipboard.**
>
> *(Substitute `/spec-{{LEVEL}}` — e.g. "/spec-feature" or "/spec-epic".)*
>
> 1. **None** — keep defaults: input from `\$ARGUMENTS`, output to chat + clipboard.
> 2. **Open answer** — describe input and/or output wiring in your reply.
>
> _Input examples_ — read an ADO epic as input for drafting features; read a Linear issue draft to refine the feature; read a markdown file from disk.
> _Output examples_ — create new features under an ADO epic; update the same Linear issue in place; create new files on disk.

After printing, stop. Do not call any further tools. Wait for the user's reply.

If the user replies with "None" (or "1", or anything that clearly means none), leave Phase 1 and Phase 3 as the defaults and skip the rest of this section.

Otherwise, treat the reply as the integration description. Parse it — it may be input, output, both, or something that doesn't fit either bucket cleanly. Ask one short follow-up only if critical specifics are missing (which source/destination, which tool — MCP server, `az` / `gh` / `curl`, file path — which fields). Keep it tight, no interview.

Then implement a preliminary version of what the user described into the produced skill:

- **Input integrations** modify Phase 1 (pre-loop). Replace or supplement the `$ARGUMENTS` step with whatever the user described — e.g. fetch an ADO work item, read a Confluence page, query a tracker. Use the most natural tool for the described source (MCP server if available, otherwise CLI like `az` / `gh` / `curl`).
- **Output integrations** modify Phase 3 (post-loop). Add the described write step alongside (not in place of) the chat + clipboard output — e.g. create a Linear issue via MCP, post a comment on a PR via `gh`, append to a file. Keep clipboard as a fallback.

Implement what the user described directly into the relevant phase of each produced skill. Don't add a separate "integration notes" section. If you have to make assumptions (which project, which field, which MCP tool), pick a reasonable default and call it out inline as a comment so the user can adjust.

## Skill destination

The destination is **fixed**: always write to user-global skills.

- `{{LEVEL_SKILL_DIR}}` = `~/.claude/skills/spec-{{LEVEL}}` (resolve `~` to the absolute home path) — i.e. `~/.claude/skills/spec-epic` or `~/.claude/skills/spec-feature`.
- Contains `SKILL.md` and `references/template.md`.

Do **not** ask the user where to write — they can move or rename the directory afterwards if they want.

If `SKILL.md` already exists at the target path, show the path and ask whether to **overwrite** or **abort**. This is a safety check on destructive writes, not a destination preference question.

## Default spec templates

### Epic template — written verbatim to `{{LEVEL_SKILL_DIR}}/references/template.md` when `{{LEVEL}}=epic`

```markdown
# <Title>

## Problem

<One paragraph: what user/business pain are we solving, and for whom. No solutioning.>

## Outcomes

<What does success look like — observable, measurable. Metrics where possible. Each outcome independently verifiable.>

1. ...
2. ...

## High-level requirements

<Coarse capabilities this epic must deliver. RFC 2119 keywords. Each item is feature-sized — detail belongs in the feature specs underneath.>

1. The system MUST ...
2. The system SHOULD ...

## Out of scope

<What this epic explicitly does NOT cover. Boundaries with adjacent epics or future work.>

## Open questions

<Decisions not yet made that block one or more underlying features. Resolve before breaking the epic down.>
```

### Feature template — written verbatim to `{{LEVEL_SKILL_DIR}}/references/template.md` when `{{LEVEL}}=feature`

```markdown
# <Title>

## Overview

<One paragraph: the problem and what changes. No implementation details.>

## Requirements

<Numbered list. RFC 2119 keywords (MUST, SHOULD, MAY). Each item independently testable.>

1. The system MUST ...
2. The system SHOULD ...
3. The system MAY ...

## Scenarios

<Given-When-Then. One per significant behavior. Cover failure modes.>

### Scenario: <name>

- **Given** <precondition>
- **When** <action>
- **Then** <observable outcome>

## Constraints

<Non-functional limits and explicit non-goals. Performance, compatibility, security, what's out of scope.>
```

## Clipboard command

Default to `pbcopy` (macOS). If the user is on Linux or Windows, swap for `xclip -selection clipboard` or `clip.exe` respectively. Capture as `{{CLIPBOARD_CMD}}`. If unsure, use `pbcopy` and tell the user to swap it themselves.

## Output

Write **two files** for the chosen `{{LEVEL}}`. Create parent directories if missing.

1. `{{LEVEL_SKILL_DIR}}/SKILL.md` — the SKILL.md (template below, with `{{LEVEL}}` substituted).
2. `{{LEVEL_SKILL_DIR}}/references/template.md` — the matching level template (epic or feature), copied verbatim from above.

Substitute these placeholders in the produced SKILL.md template per file:

- `{{LEVEL}}` — `epic` or `feature`
- `{{SKILL_NAME}}` — `spec-epic` or `spec-feature`
- `{{LEVEL_LONG}}` — `epic-level (intent + outcomes)` or `feature-level (behavior)`
- `{{LEVEL_EMPHASIS}}` — see "Level-specific loop emphasis" above; insert the matching paragraph
- `{{TEMPLATE_SECTIONS}}` — for epic: `Problem → Outcomes → High-level requirements → Out of scope → Open questions`. For feature: `Overview → Requirements → Scenarios → Constraints`.
- `{{REJECT_RULE}}` — for epic: `Reject scenario-level detail and concrete acceptance criteria — those belong in feature specs underneath this epic.` For feature: `Reject re-litigating intent or success metrics — those belong in the parent epic spec.`
- `{{CLIPBOARD_CMD}}` — see above

Weave any integrations described by the user into Phase 1 / Phase 3 of the produced SKILL.md (see Integrations section above).

````markdown
---
name: {{SKILL_NAME}}
description: Write a {{LEVEL_LONG}} spec for a feature, bug, or change, grounded in this codebase. Run from the project root. Loops research → interrogate → refine until the spec is sharp enough to hand off without follow-up questions, then prints it and copies it to the clipboard. Invoke ONLY when the user explicitly types the slash command or asks in plain language to write/draft/refine a {{LEVEL}}-level spec. Do NOT auto-invoke. Do NOT invoke to implement.
---

You are helping the user write a **{{LEVEL}}-level** spec.

The argument is: $ARGUMENTS — a raw prompt. Could be one line or several paragraphs. Could be a fresh idea or an existing draft. Either way, treat it as the starting point and run the loop.

The spec template lives in `references/template.md` next to this SKILL.md. Read it once at the start of Phase 1 — it defines the section structure ({{TEMPLATE_SECTIONS}}) the loop targets. Edit `references/template.md` directly if you want to change the structure for future runs.

## Working principles

- **Run from the project root.** This skill expects the current working directory to be the repo it specs against. If the cwd doesn't look like a project root (no manifest, no `.git`), stop and tell the user to re-run from the project root.
- **The codebase is an input.** Specs are grounded in actual code, not written in the abstract. The research step reads relevant files, modules, and conventions before asking the user anything. Generic clarifying questions ("what should the API return?") are a sign you haven't researched enough.
- **Loop until sharp.** The on-loop iterates research → interrogate → refine. It ends when the spec is detailed enough to hand off without follow-up questions, not when you've walked the template once.
- **Stay at this level.** {{REJECT_RULE}}
- **No thin specs.** If a section is vague, that's a signal to research and interrogate more, not to ship as-is.

## Level emphasis

{{LEVEL_EMPHASIS}}

## Phase 1 — Pre-loop: load the input

Confirm cwd is a project root (manifest like `package.json` / `pyproject.toml` / `go.mod` / `pom.xml`, or a `.git` directory). If not, stop and tell the user to re-run from the project root.

Read `references/template.md` (alongside this SKILL.md) so you know the section structure the loop will produce.

The input is `$ARGUMENTS`. If it's empty, ask the user for a one-paragraph description before starting Phase 2.

Get a quick orientation on the project so the loop's research step has a baseline:

- `CLAUDE.md` at the repo root and any subdirectory CLAUDE.md you've already loaded.
- top-level `README.md` and `docs/` if present, for vocabulary and architecture.
- repo layout — top-level directories, where the code lives.

This is orientation, not deep research. Deeper code investigation happens inside the on-loop's research step.

## Phase 2 — On-loop: research → interrogate → refine

Iterate this cycle until the spec is sharp enough to hand off without follow-up questions:

1. **Research** — re-read the input and the current working draft, then read the codebase for the area in question: relevant modules, existing behavior, types, error paths, tests. The goal is to know enough about the current state that your questions are concrete instead of generic. Identify the biggest gap or ambiguity that the codebase doesn't already answer. If the input already has structure (a written draft rather than a one-paragraph idea), the ripest targets are usually explicit open questions, "follows the existing X pattern" handwaves without a citation, and requirements that would change behavior the current code doesn't handle. If those hints don't surface anything obvious, sweep the draft against these ambiguity categories as a backstop: Functional Scope, Data Model, UX Flow, NFRs, Integration, Edge Cases, Constraints, Terminology, Completion Signals, Error Handling, Permissions. Reach for it when stuck — it's not a checklist to walk every iteration, and the loop still ends when nothing new turns up.
2. **Interrogate** — ask a few targeted questions about that gap, citing the code you found. Stop when you have enough to write something concrete.
3. **Refine** — update the spec (add/extend a section, tighten wording, add scenarios). Read the changed part back briefly. Ask "anything missing?"

Move through the template's sections ({{TEMPLATE_SECTIONS}}) but treat them as places the loop touches, not a one-pass walkthrough. Earlier sections often need revisiting once later ones surface new behavior.

The loop ends when nothing new turns up: no missing requirements, no uncovered scenarios, no unstated constraints, no ambiguous wording.

If the user pushes detail that belongs at a different level, redirect: {{REJECT_RULE}}

### Before leaving Phase 2: scope check

Once the loop has settled, look at the spec as a whole. If it's large **and** has clean fault lines, propose a split: name 2–N candidate specs and which sections go to each. Ask: "split into these N specs, or keep as one?" If yes, emit one spec per slice in Phase 3. If small or messy, keep as one.

## Phase 3 — Post-loop: print and copy

Write the final spec to `$TMPDIR/{{SKILL_NAME}}.md` (or to multiple files if split: `$TMPDIR/{{SKILL_NAME}}-1.md`, `$TMPDIR/{{SKILL_NAME}}-2.md`, …). Then:

1. Print the full markdown to chat, fenced.
2. Copy it to the clipboard:

   ```
   cat $TMPDIR/{{SKILL_NAME}}.md | {{CLIPBOARD_CMD}}
   ```

   For a split, concatenate the slices with `---` separators between them, then copy the whole thing.

Tell the user it's on the clipboard. The user wires up downstream destinations (paste into a ticket, save to a file, etc.) themselves.
````

After writing both files, print the two paths and tell the user:

- They can move/rename the skill directory if they want a different location.
- `references/template.md` is where to edit the section structure for future runs.
- To get the other level (`/spec-epic` if they forged feature, or vice versa), run this skill again. The two skills are independent — running `/spec-epic` doesn't chain into `/spec-feature`; the user breaks an epic down by invoking `/spec-feature` per feature when ready.
