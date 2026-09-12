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

```
---
name: <kebab-case, matches the directory name>
description: <one sentence: what it does, when it fires, the standard it needs>
---
```

Body sections, in this order, only the ones that apply:

1. **`# Feature`** (the H1) — one or two sentences: what it is, what
   standard it needs.
2. **Modernization benefits** — one bullet per real aspect, each led by
   a short label naming what improves (`Performance`, `Correctness`,
   `Clarity`, `Bug reduction`, ...). The label set isn't fixed — use
   whichever genuinely apply and skip the rest. If the feature trades
   one risk for another instead of only removing risk (string_view
   trades copies for a dangling-view hazard), say the trade-off plainly
   instead of listing a benefit that isn't real.
3. **Goal** — one or two sentences on what the agent should end up
   doing or producing. Not a restatement of the feature description;
   the actionable version of it.
4. **Preconditions** — language-standard gate, project configuration to
   check, anything that blocks the skill from applying at all. Keep
   this separate from Tool usage below — a precondition blocks the
   whole skill, a tool result blocks one call site.
5. **Tool usage** — open with a bullet list, one line per relevant
   tool/check: name, then what it actually does (fixes it / only
   detects one hazard class / etc). Follow the list with what that
   implies:
   - A tool performs the whole transform → tell the agent to run it,
     don't reimplement the rewrite in prose; the sections below then
     only describe what the tool did/would do.
   - Tools only catch hazards after the fact (as with the bugprone
     checks for `string_view`) → say so, and make clear the sections
     below always apply — each tool is a gate run during Procedure, not
     an alternative to manual work.
   - No tool exists → say so once and move straight to the sections
     below.
   Don't write "the manual sections apply only if the user declines the
   tool" unless a tool genuinely performs the transform end to end;
   that framing is false for a hazard-detector.
6. **No-brainer replacements** — conditions under which the change is a
   safe drop-in with no correctness trade-off. This is the load-bearing
   section; spend the words here, not on introduction.
7. **Discuss first** — changes that are possible but rest on an
   assumption about the codebase the agent can't verify alone (ask,
   don't guess and don't silently skip).
8. **Refactor first** — a pattern the skill recognizes but never
   modifies without approval, plus what would have to change first. If
   nothing could ever make it safe (an external API needs a
   null-terminated buffer), say that plainly instead of implying a
   refactor path exists.
9. **Procedure** — an ordered list of what the agent actually does.
10. **Non-goals** — what this skill's *scope* excludes entirely (other
    features, other standards, ABI boundaries), as distinct from the
    specific patterns in Refactor first.
11. **Links** — the standard/proposal reference for the feature, and a
    direct link per tool check named above. Verify every link points at
    real content (see "Verify tool claims" below) before citing it; if
    a specific detail (a proposal number, a version) can't be verified
    in the current session, link the stable reference page instead of
    asserting the detail.

Skip a section rather than fill it with filler.

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
