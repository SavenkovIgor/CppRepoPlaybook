# Writing skills in this repo

Instructions for an agent authoring or editing a `skills/{name}/SKILL.md`
file here. Read this before creating a new skill or editing an existing
one.

## Principles are enforced through the skill's logic, not its prose

The repo's principles (`../README.md`) — C++ is not C, deterministic
tools priority, correctness over modernity, readability — are context
that every session already has. A skill must act on them, not restate
them.

- Wrong: a paragraph like "per this repo's deterministic-tools-priority
  principle, we first check for a linter rule" followed by the check.
- Right: just the check, stated as a fact about the specific feature
  ("clang-tidy has no check for X, so this skill audits call sites
  manually").

If a sentence in a skill would still make sense with "per our principles"
deleted from the front of it, delete that clause — it's the marker that
the principle is being cited instead of applied. Quoting or paraphrasing
README prose inside a skill body is pure token overhead: it teaches the
model nothing about the C++ feature and gets re-read on every load of the
skill.

## Required shape

```markdown
---
name: <kebab-case, matches the directory name>
description: <one sentence: what it does, when it fires, the standard it needs>
---

# <Feature name>

<One or two sentences: what it is, what standard it needs.>

## Modernization benefits

- **<Aspect>.** <what improves and why>

## Goal

<One or two sentences: what the agent should end up doing or producing —
the actionable version of the feature description, not a repeat of it.>

## Preconditions

<What blocks the skill from applying at all: language standard, project
config. Distinct from Tool usage below — this blocks the whole skill, a
tool result blocks one call site.>

## Tool usage

- `<tool or check name>` — <what it actually does>

<What follows from the list above: run it and stop / it's a gate during
Procedure / no tool exists, proceed manually.>

## No-brainer replacements

<Conditions under which the change is a safe drop-in, no trade-off. The
load-bearing section — spend the words here, not on introduction.>

## Discuss first

<Changes that are possible but rest on an assumption about the codebase
the agent can't verify alone. Ask, don't guess, don't silently skip.>

## Refactor first

<A pattern the skill recognizes but never modifies without approval,
plus what would have to change first.>

## Procedure

1. <ordered steps the agent actually takes>

## Non-goals

<What this skill's scope excludes entirely, distinct from the specific
patterns in Refactor first.>

## Links

- <standard/proposal reference>
- <one link per tool/check named in Tool usage>
```

Skip a section rather than fill it with filler.

Three sections carry a rule that isn't obvious from the skeleton above:

- **Modernization benefits.** The aspect label isn't a fixed enum
  (`Performance`, `Correctness`, `Clarity`, `Bug reduction`, ... are
  examples) — use whichever genuinely apply. If the feature trades one
  risk for another instead of only removing risk (string_view trades
  copies for a dangling-view hazard), say the trade-off plainly instead
  of listing a benefit that isn't real.
- **Tool usage.** Don't write "the manual sections apply only if the
  user declines the tool" unless a tool genuinely performs the
  transform end to end — that framing is false for a hazard-detector
  (like the bugprone checks for `string_view`), where the sections
  below always apply and the tool is only a gate.
- **Refactor first.** If nothing could ever make a pattern safe (an
  external API needs a null-terminated buffer), say that plainly
  instead of implying a refactor path exists.
- **Links.** Verify every link points at real content (see "Verify tool
  claims" below) before citing it; if a specific detail (a proposal
  number, a version) can't be verified in the current session, link the
  stable reference page instead of asserting the detail.

## Verify tool claims, don't recall them

Any sentence that says a check, flag, or tool exists — or doesn't —
is a factual claim about the current state of an external project, not
a stylistic choice. Model memory of clang-tidy/clang-format check names
is unreliable in both directions: it invents plausible-sounding checks
that don't exist, and misses real ones (a full grep of upstream
`bugprone-*` docs turned up three checks directly relevant to
`cpp17-string_view` that an earlier draft had missed entirely). Before
a claim like this goes into a skill or a commit:

- Fetch the actual upstream doc for the specific check name, not just a
  directory listing or a search-result summary — those get
  hallucinated wholesale when the underlying page is a 404 or otherwise
  empty. A result is trustworthy when it quotes specific file content
  (example code, exact option names); treat a vague or suspiciously
  tidy summary as unverified and re-fetch the raw file directly.
  A plain 404/error is trustworthy on its own — that's a real answer,
  not a gap to paper over.
- If the fetch is blocked or inconclusive, say in the skill (or to the
  user) that the check's existence is unconfirmed rather than asserting
  either way.
- Re-verify before every commit that changes a tool-existence claim, not
  just once when the skill was first drafted — check lists change
  between LLVM releases.

## Budget and register

- Target well under 150 lines. If a skill is approaching that, look for
  restated principles, restated standard-library documentation, or
  redundant examples before adding a length exception.
- Write for an agent, not a human reviewer: imperative, checklist-first,
  no motivational framing ("it's important to...", "in modern C++ we
  strive to...").
- One skill = one feature/transform. Don't fold multiple standard
  library features into one SKILL.md; add a sibling skill instead.

## plugin.json needs no per-skill entry

Skills are auto-discovered: per the Agent Plugins 1.0 schema
(`schemas/1.0.0/plugin.schema.json` in the spec repo), `plugin.json` has
no `skills` field at all — a client finds each skill by reading
`skills/{name}/SKILL.md` directly off disk. Adding or renaming a skill
never requires touching `plugin.json`. If a future spec version adds a
manifest field for this, verify it against the schema (per the rule
above) before assuming it's now required — don't infer it from a
changed example in prose.

## Naming

Directory and `name` field match: `cpp<NN>-<feature>`, e.g.
`cpp17-string_view`. Use the feature's actual spelling (`string_view`,
not `string-view`) when the hyphenated form would be a different token
than the one engineers grep for.
