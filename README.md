# useEffect-cleaned-architecture

A formal engineering skill that eliminates unnecessary `useEffect` usage in React by enforcing correct state-flow design. It is a reasoning standard, not a code generator, lint rule, or scaffold.

## What it is

`useEffect-cleaned-architecture` is a deterministic cognitive architecture for classifying state, deciding control flow, and governing when effects are allowed. Every piece of logic passes through a fixed pipeline before it may become an effect. The output is a design decision, not code.

## The problem it solves

- `useEffect` used for derivation, fetching, and action response
- State duplicated across `useState` and kept in sync by effects, creating stale-state windows
- Render cascades from effect chains
- Derived values stored as source-of-truth state

## What is in the spec

`Skill.md` contains 12 sections. The core ones:

- **State Classification System (Section 4)** — six categories (server, UI, form, global, derived, external system), each with handling strategy and misuse patterns. This is the engine.
- **Decision Engine (Section 5)** — a five-stage pipeline every feature passes through.
- **useEffect Governance (Section 6)** — when effects are allowed, what justification each requires, and the forbidden uses.
- **Refactoring Heuristics (Section 9)** — deterministic transforms to fix existing bad code.
- **Allowed vs Forbidden Matrix (Section 10)** — quick reference table.
- **Cross-Framework Transferability (Section 12)** — applies to React Native, Next.js, Vue, Flutter, and backend event systems.

Read `Skill.md` for the full standard.

## How to use it

### As an internal engineering standard

1. Adopt `Skill.md` as the team's React design reference.
2. Require every code review to apply Section 5 (decision engine) to any new state, handler, or effect.
3. Require every `useEffect` in a PR to state the three justifications from Section 6.2 (external system, external dependency, why not derived/event). Reject any that cannot.
4. Use Section 9 heuristics to refactor legacy effects during touchups.

### As an AI system prompt

Paste `Skill.md` (or Sections 4 through 6) into the system prompt of any coding agent that writes React. The spec is written as a reasoning model, so the agent applies the classification and decision pipeline before proposing effects. Instruct the agent: "Before writing any `useEffect`, run Section 5 and cite Section 6.2 justification."

### As an opencode skill

Place this directory under your opencode skills path (e.g. `~/.config/opencode/skills/useEffect-cleaned-architecture/`) containing `Skill.md`. The skill loads when React state or effect design work is requested.

## Setup

1. Clone or copy this folder into your skills directory or docs repository.
2. Point your team or AI agent at `Skill.md`.
3. No build step, no dependencies. It is a specification document.

## Constraints

The skill is a reasoning standard. It does not generate code, project structures, or components. It encodes decisions: classify state (Section 4), run the pipeline (Section 5), govern effects (Section 6).
