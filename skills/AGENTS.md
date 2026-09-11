# Writing skills in this repo

Instructions for an agent authoring or editing a `skills/{name}/SKILL.md`
file here. Read this before creating a new skill or editing an existing
one.

## Principles are enforced through the skill's logic, not its prose

The repo's principles (`../README.md`) — C++ is not C, deterministic
tools priority, correctness > modernity, readability — are context every
session already has. A skill must **act on them**, not **restate them**.

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

```
---
name: <kebab-case, matches the directory name>
description: <one sentence: what it does, when it fires, the standard it needs>
---
```

Body sections, in this order, only the ones that apply:

1. **Preconditions** — language-standard gate, project configuration to
   check, anything that must be true before any edit starts.
2. **Where it's safe** / **Where to leave it alone** — the concrete,
   checkable conditions that separate a good call site from a bad one.
   This is the load-bearing part of the skill; spend the words here, not
   on introduction.
3. **Procedure** — an ordered list of what the agent actually does.
4. **Non-goals** — what this skill explicitly does not attempt, so it
   doesn't get stretched to cover cases it wasn't designed for.

Skip a section rather than fill it with filler. A skill with only
Preconditions + Procedure is fine if the feature has no unsafe cases
worth calling out.

## Deterministic tools first, but say so plainly

Before writing a manual procedure, check whether clang-tidy, clang-
format, or a compiler warning already covers the transform. Then say
which is true in one line:

- A check exists → tell the agent to enable/run it, don't reimplement it
  in prose.
- No check exists → say so once ("no clang-tidy check covers this") and
  move on to the manual procedure. Don't justify why manual review is
  being used.

## Budget and register

- Target well under 150 lines. If a skill is approaching that, look for
  restated principles, restated standard-library documentation, or
  redundant examples before adding a length exception.
- Write for an agent, not a human reviewer: imperative, checklist-first,
  no motivational framing ("it's important to...", "in modern C++ we
  strive to...").
- One skill = one feature/transform. Don't fold multiple standard
  library features into one SKILL.md; add a sibling skill instead.

## Naming

Directory and `name` field match: `cpp<NN>-<feature>`, e.g.
`cpp17-string_view`. Use the feature's actual spelling (`string_view`,
not `string-view`) when the hyphenated form would be a different token
than the one engineers grep for.
