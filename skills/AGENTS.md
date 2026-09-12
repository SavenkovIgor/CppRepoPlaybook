# Writing skills in this repo

Instructions for an agent authoring or editing a `skills/{name}/SKILL.md` file here.
Read this before creating a new skill or editing an existing one.
Also read the `README.md` for the overarching principles and context.

## Rules of skills creation/editing

## Principles are enforced through the skill's logic, not its prose

The repo's principles from the `README.md` are a guide on how to write a proper skill - not what text should be
pasted in the skill - just write it in a way it conforms to these principles.

- Wrong: a paragraph like "per this repo's deterministic-tools-priority principle, we first check for a linter rule".
- Right: just the check, stated as a fact about the specific feature ("clang-tidy has no check for X,
  so this skill audits call sites manually").

If a sentence in a skill would still make sense with "per our principles" deleted from the front of it, delete that clause — it's the marker that the principle is being cited instead of applied.

## Basic template for modernizing SKILL.md

```markdown
---
name: <`cpp<NN>-<feature>` matches the directory name and may only contain lowercase letters, numbers, and hyphens>
description: <one sentence: what it does>
---

# <Feature name>

<One or two sentences: what it is, what standard it needs.>

## Modernization

### Benefits

- <Aspect>: <what improves and why>

### Limitations

- <Aspect>: <what the limitation is and why>

## Goal

<One or two sentences the agent flow - what info collect, what to ask from user, what to implement>

## Preconditions

<What blocks the skill from applying at all: language standard, project
config. Distinct from Tool usage below — this blocks the whole skill, a
tool result blocks one call site.>

## Tool usage

<What tools could help in modernization and enforcement of applied changes
It could be tools used during the modernization by Agent and tools that could
be integrated into the project workflow (dev/ci) to enforce the changes>

<Tool name:>
- `<tool or check name>`: <what it does>
- `<another tool or check name>`: <what it does>

## No-brainer replacements

<Section for changes that are a safe drop-in, no trade-off>

## Discuss first

<Section for changes that are possible but need approval from the user.
Should contain specific questions to ask, details on what changes require user approval and why>

## Refactor first

<Section for changes that can't be applied right now and requires some amount
of refactoring. Declare why the modernization can't be applied and what refactoring is necessary.
These changes should be never applied without explicit approval from the user.>

## Procedure

1. <ordered steps the agent actually takes>

## Out of scope

<What should be out of scope for this skill and why>

## Links

- <proposal reference>
- <cppreference link: `https://en.cppreference.com/cpp/` or similar>
- <tool/check link mentioned in the Tool usage section: `https://clang.llvm.org/extra/clang-tidy/checks` or similar>
```

### Some rules to apply to this template and comments on the content

- Skip a section rather than fill it with filler
- Modernization/benefits/limitations section: the aspect label isn't a fixed enum (`Performance`, `Correctness`,
  `Clarity`, `Bug reduction`, etc) - use whichever genuinely apply. If the feature trades one risk for
  another instead of only removing risk (string_view trades copies for a dangling-view hazard), you should say about the
  trade-off plainly.
- Tool usage. If modernization can be done with some tool instead of AI editing, this section is ideal place to say so.
  If such tool exist, this section should contain a proposal to enable/integrate it instead of relying on AI editing.
  If the tool provides additional benefits (security, performance, correctness), that is the second reason why it should be mentioned in this section but in this case it will be used as enforcement tool along with AI editing.
- Links. Verify every link points at real content before citing it

## Verify tool claims, don't recall them

Any statement that a check, flag, or tool exists — or doesn't — is a factual claim about upstream state, not style.
Model memory is unreliable: it invents plausible-sounding names and misses real ones.
Before writing such a claim into a skill or commit:

- Fetch the exact upstream docs for that specific item; don't rely on directory listings or summaries alone.
- Treat raw file content or a clear 404/error as the trustworthy evidence. Vague summaries are not enough.
- If the result is blocked or inconclusive, say the claim is unconfirmed rather than asserting either way.
- Re-check before every commit that changes a tool-existence claim; upstream lists change between releases.

## Budget and register

- Target well under 150 lines. If a skill is approaching that, look for restated principles, restated standard-library
  documentation, or redundant examples before adding a length exception.
- One skill = one feature/transform. Don't fold multiple standard library features into one SKILL.md.

## Naming

Directory and `name` field match: `cpp<NN>-<feature>`, e.g. `cpp17-string_view`.
Use the feature's actual spelling but convert underscores to hyphens due to plugin naming conventions.
