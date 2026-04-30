---
name: forge-spec-loop
description: Forge a portable /spec skill (user-global by default; works across any repo). Writes a SKILL.md that runs a research → interrogate → refine loop on a raw prompt input, prints the result to chat, and copies it to the clipboard. The produced skill reads the codebase from cwd at runtime — nothing repo-specific is baked in. Invoke ONLY when the user explicitly types the slash command or asks in plain language to set up/scaffold/forge a /spec skill — e.g. "set up /spec", "forge a spec skill". Do NOT auto-invoke. Do NOT invoke to write a spec.
---

# forge-spec-loop

The job of this skill is to forge **the loop**, nothing more. The produced /spec takes a raw prompt as input, sharpens it against whichever codebase it's invoked in via research → interrogate → refine, prints the result, and copies it to the clipboard. Anything beyond that — fetching from a tracker, writing to a file, posting back to a ticket — is the user's job to wire up.

The produced skill is **portable**: it does not bake in anything about a specific repo. It reads the codebase from the current working directory at runtime, so the same skill works across every project the user has.

A spec describes **behavior**, not implementation: intent (why), design (what), technical bounds (how far).

The loop has three phases:

- **Pre-loop** — load the input (the raw `$ARGUMENTS` prompt)
- **On-loop** — research → interrogate → refine, iteratively, until the spec is sharp enough to hand off without follow-up questions
- **Post-loop** — print the result and copy it to the clipboard

The following are **assumed** and baked into the produced skill — do not ask about them:

1. The on-loop is a real loop: research → interrogate → refine, repeated. It ends when the spec is sharp enough to hand off without follow-up questions, not after a single template walk-through.
2. Use the template (Overview → Requirements → Scenarios → Constraints) as places the loop touches, not a checklist. Later iterations often revisit earlier sections.
3. After every refine step, read the change back briefly and ask "anything missing?"
4. Specs describe behavior. Reject implementation details unless they're hard constraints.
5. Once the loop has produced a sharp spec, look at the whole. If it has clean fault lines (disjoint requirement clusters, separable scenario groups), propose splitting it before printing.
6. The codebase is a primary input. The on-loop's research step reads code, conventions, and existing behavior — that's how clarifying questions stay grounded instead of generic. The produced skill assumes it runs from the project root.

## Spec template

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

Show this template to the user. Ask if they want to adjust it. If yes, work with them — verify the final version still has a home for intent, design, and technical bounds. Capture the (possibly adjusted) template as `{{TEMPLATE}}`.

## Integrations

Ask the user a free-form question:

> "By default /spec reads input from `$ARGUMENTS` and writes output to chat + clipboard. If you want input pulled from somewhere (e.g. 'read epic from ADO as input for drafting features') or output written somewhere (e.g. 'use Linear MCP to write the issue directly to Linear'), describe it in your own words. Skip if not needed."

If the user describes an integration, implement a preliminary version of it in the produced skill:

- **Input integrations** modify Phase 1 (pre-loop). Replace or supplement the `$ARGUMENTS` step with whatever the user described — e.g. fetch an ADO work item, read a Confluence page, query a tracker. Use the most natural tool for the described source (MCP server if available, otherwise CLI like `az` / `gh` / `curl`).
- **Output integrations** modify Phase 3 (post-loop). Add the described write step alongside (not in place of) the chat + clipboard output — e.g. create a Linear issue via MCP, post a comment on a PR via `gh`, append to a file. Keep clipboard as a fallback.

Implement what the user described directly into the relevant phase of the produced skill. Don't add a separate "integration notes" section. If you have to make assumptions (which project, which field, which MCP tool), pick a reasonable default and call it out inline as a comment so the user can adjust.

If the user skips, leave Phase 1 and Phase 3 as the defaults (read `$ARGUMENTS`, print + clipboard).

## Skill destination

Use AskUserQuestion (single-select):

**Where should the produced SKILL.md be written?**

- _user-global_ — `~/.claude/skills/spec/SKILL.md` (default; portable across all of the user's projects).
- _project-local_ — `.claude/skills/spec/SKILL.md` (relative to the current project root). Checked into the repo; the team owns and edits it. Pick this only if the team wants a shared, repo-checked variant.
- _custom_ — user supplies a different absolute or repo-relative directory; the skill goes to `<dir>/spec/SKILL.md`.

Capture the resolved absolute path as `{{SKILL_PATH}}`. If it already exists, show the path and ask whether to overwrite or pick a different location.

## Clipboard command

Default to `pbcopy` (macOS). If the user is on Linux or Windows, swap for `xclip -selection clipboard` or `clip.exe` respectively. Capture as `{{CLIPBOARD_CMD}}`. If unsure, use `pbcopy` and tell the user to swap it themselves.

## Output

Write the produced SKILL.md to `{{SKILL_PATH}}`. Create parent directories if missing. Substitute `{{TEMPLATE}}` and `{{CLIPBOARD_CMD}}`. Weave any integrations described by the user into Phase 1 / Phase 3 directly (see Integrations section above) — there is no separate notes block to substitute.

````markdown
---
name: spec
description: Write a behavior-focused spec for a feature, bug, or change, grounded in this codebase. Run from the project root. Loops research → interrogate → refine until the spec is sharp enough to hand off without follow-up questions, then prints it and copies it to the clipboard. Invoke ONLY when the user explicitly types the slash command or asks in plain language to write/draft/refine a spec. Do NOT auto-invoke. Do NOT invoke to implement.
---

You are helping the user write a spec. Specs describe **behavior**, not implementation.

The argument is: $ARGUMENTS — a raw prompt. Could be one line or several paragraphs. Could be a fresh idea or an existing draft. Either way, treat it as the starting point and run the loop.

## Working principles

- **Run from the project root.** This skill expects the current working directory to be the repo it specs against. If the cwd doesn't look like a project root (no manifest, no `.git`), stop and tell the user to re-run from the project root.
- **The codebase is an input.** Specs are grounded in actual code, not written in the abstract. The research step reads relevant files, modules, and conventions before asking the user anything. Generic clarifying questions ("what should the API return?") are a sign you haven't researched enough.
- **Loop until sharp.** The on-loop iterates research → interrogate → refine. It ends when the spec is detailed enough to hand off without follow-up questions, not when you've walked the template once.
- **Behavior, not implementation.** Push back on implementation details unless they're hard constraints. File paths, class/function names, internal data structures, and library/framework choices don't belong in the spec — only the observable behavior they produce.
- **No thin specs.** If a section is vague, that's a signal to research and interrogate more, not to ship as-is.

## Phase 1 — Pre-loop: load the input

Confirm cwd is a project root (manifest like `package.json` / `pyproject.toml` / `go.mod` / `pom.xml`, or a `.git` directory). If not, stop and tell the user to re-run from the project root.

The input is `$ARGUMENTS`. If it's empty, ask the user for a one-paragraph description before starting Phase 2.

Get a quick orientation on the project so the loop's research step has a baseline:

- `CLAUDE.md` at the repo root and any subdirectory CLAUDE.md you've already loaded.
- top-level `README.md` and `docs/` if present, for vocabulary and architecture.
- repo layout — top-level directories, where the code lives.

This is orientation, not deep research. Deeper code investigation happens inside the on-loop's research step.

## Phase 2 — On-loop: research → interrogate → refine

Iterate this cycle until the spec is sharp enough to hand off without follow-up questions:

1. **Research** — re-read the input and the current working draft, then read the codebase for the area in question: relevant modules, existing behavior, types, error paths, tests. The goal is to know enough about the current state that your questions are concrete ("today `chargeCard` retries on 5xx but not on timeout — should the spec keep that?") instead of generic ("how should errors be handled?"). Identify the biggest gap or ambiguity that the codebase doesn't already answer. If the input already has structure (a written draft rather than a one-paragraph idea), the ripest targets are usually explicit open questions, "follows the existing X pattern" handwaves without a citation, and requirements that would change behavior the current code doesn't handle. If those hints don't surface anything obvious, sweep the draft against these ambiguity categories as a backstop: Functional Scope, Data Model, UX Flow, NFRs, Integration, Edge Cases, Constraints, Terminology, Completion Signals, Error Handling, Permissions. Reach for it when stuck — it's not a checklist to walk every iteration, and the loop still ends when nothing new turns up.
2. **Interrogate** — ask a few targeted questions about that gap, citing the code you found. Stop when you have enough to write something concrete.
3. **Refine** — update the spec (add/extend a section, tighten wording, add scenarios). Read the changed part back briefly. Ask "anything missing?"

Move through the template's sections (Overview → Requirements → Scenarios → Constraints) but treat them as places the loop touches, not a one-pass walkthrough. Earlier sections often need revisiting once later ones surface new behavior.

The loop ends when nothing new turns up: no missing requirements, no uncovered scenarios, no unstated constraints, no ambiguous wording.

If the user pushes implementation details into a behavior section, note them — they may belong in **Constraints** (as a hard requirement) or get dropped.

### Before leaving Phase 2: scope check

Once the loop has settled, look at the spec as a whole. If it's large **and** has clean fault lines, propose a split: name 2–N candidate specs and which requirements/scenarios go to each. Ask: "split into these N specs, or keep as one?" If yes, emit one spec per slice in Phase 3. If small or messy, keep as one.

## Phase 3 — Post-loop: print and copy

Write the final spec to `$TMPDIR/spec.md` (or to multiple files if split: `$TMPDIR/spec-1.md`, `$TMPDIR/spec-2.md`, …). Then:

1. Print the full markdown to chat, fenced.
2. Copy it to the clipboard:

   ```
   cat $TMPDIR/spec.md | {{CLIPBOARD_CMD}}
   ```

   For a split, concatenate the slices with `---` separators between them, then copy the whole thing.

Tell the user it's on the clipboard. The user wires up downstream destinations (paste into a ticket, save to a file, etc.) themselves.

## Template

```markdown
{{TEMPLATE}}
```
````

After writing the file, print the path of the new skill to the user.
