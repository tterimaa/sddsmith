# sddsmith

Helps you forge the two loops every spec-driven workflow needs — the **specify loop** and the **implement loop** — shaped to your company's stack, conventions, and validation discipline.

## The two loops

```
   ┌─────────────────────────────┐         ┌─────────────────────────────┐
   │      SPECIFY LOOP           │         │      IMPLEMENT LOOP         │
   │                             │  spec   │                             │
   │   research → interrogate    │  ────▶  │   build → validate          │
   │       ↑           ↓         │         │      ↑          ↓           │
   │       └─── refine ──┘       │         │      └── fix ───┘           │
   └─────────────────────────────┘         └─────────────────────────────┘
            /spec skill                          /implement skill
```

**Specify loop.** Takes in high level idea of what is being built. Interrogates the user, surfaces gaps, refines until the spec is a detailed behavioural contract. The loop ends when the spec is sharp enough to hand off without follow-up questions.

**Implement loop.** Takes the spec as input. Builds, validates, fixes what broke, validates again. The loop ends when the validation gates pass.

## What this library gives you

Two meta skills. Each interviews you about how *your* team wants to run that loop, then generates a Claude Code command file you commit to your repo.

| Meta skill | Forges | Produces |
|---|---|---|
| `/sddsmith:forge-spec-loop` | Your specify loop — where specs live, what shape they take | A `/spec-feature` *or* `/spec-epic` skill (you pick one per run). Both run the same loop; they differ in template — feature is behavior-focused (Overview, Requirements, Scenarios, Constraints), epic is intent-focused (Problem, Outcomes, High-level requirements, Out of scope). |
| `/sddsmith:forge-implement-loop` | Your implement loop — spec source, validators, report format | An `/implement` skill wired to the validators and report shape you chose |

**Two levels, one per run:**
- `/spec-feature` — behavior. Use for a single feature.
- `/spec-epic` — intent. Use for a multi-feature initiative where outcomes and boundaries matter more than concrete scenarios.

The output is a skill file in `skills/`. You own it, you edit it, you check it into your repo.

## Why meta skills, not pre-built templates

This library ships **opinions on what makes a good specify loop and a good implement loop**, but the *shape* of your `/spec` and `/implement` is yours. Each meta skill explains the rationale behind every question — you learn the principles while forging the skills.

## Scope: forge the loop, not the integrations

The meta skills forge **the loop logic and nothing else**. The produced `/spec` and `/implement` ship with the simplest possible I/O:

- **Input** — `$ARGUMENTS`. A raw prompt the user types or pastes. No tracker fetches, no file readers, no preprocessors.
- **Output** — printed to chat, fenced. Where supported, also copied to the clipboard. No ticket creation, no file writes, no API calls.

Wiring downstream — pulling specs from Jira, posting them back to Linear, filing them as `.md` in `specs/`, opening PRs from `/implement` — is **your job**, not the meta skill's. The interview captures your stated integration intent (open-text) and saves it inside the produced skill as a deferred-notes section so you have a starting point. Nothing is auto-wired.

This is deliberate. The load-bearing thing is the loop — research → interrogate → refine, or build → validate → fix. Integrations are project-specific glue and they decay quickly (APIs change, tools get swapped, conventions drift). Keeping them out of the meta skill keeps the meta skill stable and keeps you in control of the parts most likely to change. The produced skill is small enough to read in one sitting; you extend it by editing the file.


## Install

```bash
# Install both skills to all detected agents
npx skills add tterimaa/sddsmith
```

## Status

Early. Scope and shape may change.
