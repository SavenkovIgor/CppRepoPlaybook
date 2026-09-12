---
name: cpp17-string-view
description: Replace read-only std::string parameters and literal-backed string constants with std::string_view (available since C++17) to remove needless copies, while guarding against the dangling-view and non-null-terminated-buffer bugs this change can introduce.
---

# Modernize to `std::string_view` (C++17)

`std::string_view` is a non-owning, non-zero-terminated view over character data, available since C++17. It replaces `const std::string&` parameters
and literal-backed string constants without copying the underlying data.

## Modernization benefits

- **Performance.** A literal/`const char*` bound to a `const
  std::string&` parameter builds a temporary `std::string` (`strlen`
  plus possible heap allocation); `string_view` binds directly.
  `substr()` is O(1) on a view, allocates on a `std::string`. A
  `constexpr std::string_view` from a literal resolves at compile
  time; `std::string`'s `const char*` constructor isn't `constexpr`
  before C++20.
- **API clarity.** The parameter type itself says "read-only, no ownership".
- **Trade-off, not a pure win.** Not an error-reduction change: it
  swaps a copy/allocation cost for a dangling-view hazard `const
  std::string&` didn't have. Tool usage and Refactor first below guard
  against that hazard — it's why this skill converts selectively.

## Goal

Convert read-only string parameters and literal-backed string constants
to `std::string_view` wherever it is a safe drop-in, raise the rest as a
discussion point or a named prerequisite refactor, and report a decision
per symbol — never convert silently, never skip without saying why.

## Preconditions

Confirm the target standard is C++17 or newer — check
`CMAKE_CXX_STANDARD`, `set(CXX_STANDARD ...)`, or `-std=` flags. Stop
and say so on C++14 or earlier; do not propose `string_view` there.

## Tool usage

Clang-tidy:
- `bugprone-dangling-handle` — catches a view outliving its owner.
- `bugprone-stringview-nullptr` — catches a view constructed from
  `nullptr`.
- `bugprone-suspicious-stringview-data-usage` — catches `.data()` used
  without `.size()`.

Clang compiler attribute:
- `[[clang::lifetimebound]]` — ties a method's return value lifetime to
  `*this`; enables `-Wdangling` on a caller that lets the result outlive
  the object. See Discuss first — it's what makes a member-returning
  getter safe to convert unattended.

None of these perform the parameter/constant conversion itself —
candidate selection is always manual. Enable the three clang-tidy
checks as a gate during Procedure (step 6); treat any warning as a
failed candidate at that site, not something to suppress.

## No-brainer replacements

**Parameters.** Convert `const std::string&` (or `const char*`) to
`std::string_view` when every one of the following holds at every call
site:

- Only reads the characters during the call — no storing the
  pointer/view anywhere that outlives it.
- Never uses or passes the data to an API needing a null-terminated
  buffer (`printf`, `fopen`, most C APIs, `.c_str()` users).
- Doesn't rely on implicit `const char*` → `std::string` conversions
  for concatenation or mutation.
- No embedded-NUL assumption is broken — flag and treat as unsafe if
  the code assumed `strlen`-style truncation.

**Literal-backed constants.** A `static const std::string` or
`const char*` constant initialized directly from a string literal, and
never mutated, becomes `constexpr std::string_view`. Doesn't apply to a
constant built from anything other than a literal (concatenation,
`std::to_string`, runtime data) — those aren't compile-time constants.

## Discuss first

Whether to bring in the `sv` literal suffix
(`using namespace std::literals::string_view_literals;` or the
narrower `operator""sv` using-declaration) to write
`constexpr static auto key = "someKey"sv;`. Ask, don't decide
unilaterally — the unsuffixed `constexpr std::string_view key =
"someKey";` is equally a compile-time constant, so this is style, not
correctness.

**Getter returning a view of a member.** A method returning
`std::string_view` into `*this`'s own member (Chromium's
`GURL::host()`, `path()`, etc.) has the same dangling risk as a
`const std::string&` getter — `string_view` doesn't communicate that
risk more clearly, and its value-type look (no `&`) makes it easier to
mistake for something safe to keep around. Chromium's real mitigation:
every such accessor in `url/gurl.h` is `LIFETIME_BOUND`-annotated; the
same-hazard `const std::string& spec()` carries no such annotation.
The asymmetry is in where the tooling was pointed, not in the type.

**Additional precondition for this conversion:** confirm the toolchain
is Clang. Clang → annotate the new getter with
`[[clang::lifetimebound]]`, treat as a No-brainer replacement. Not
Clang → stays Discuss first: ask whether callers here use getter
results immediately or cache them — no compiler backstop exists either
way.

## Refactor first

- **Class/struct members.** Don't store a `string_view` unless the
  owner's lifetime is provably a subset of the referent's. Refactor
  path: make that bound explicit before converting; until then keep
  `std::string`.
- **Return types (general case).** Change a return type to
  `string_view` only when the view outlives the call (it aliases a
  literal or a parameter already known to outlive it); otherwise leave
  it `std::string`. A getter returning a view of `*this`'s own member
  has its own rule — see "Getter returning a view of a member" under
  Discuss first.
- **Call sites needing a null-terminated buffer downstream.** No
  refactor path exists here — the constraint is external. State that
  plainly and keep `std::string`/`const char*`.

## Procedure

1. Verify the C++17+ precondition.
2. Find candidates: (a) `const std::string&` params that only read,
   (b) literal-backed constants (No-brainer above).
3. Check each against No-brainer; sort failures into Discuss first or
   Refactor first, with the specific reason (e.g. needs a
   null-terminated path).
4. Apply the type change plus `#include <string_view>`; raise the
   `sv`-suffix question first for a literal-backed constant.
5. A call site relying on `std::string::operator+=` or similar
   mutation already failed step 3 — revert to `std::string` rather
   than work around the missing mutability.
6. Run the three bugprone checks (Tool usage) on changed files; any
   new warning means revert at that site, don't suppress it.
7. Build and run the test suite; prefer sanitizers enabled — a
   lifetime bug here can compile clean and only fail at runtime.
8. Report, per symbol: converted / discussed / deferred for refactor,
   and why.

## Non-goals

This skill does not convert `std::string` members, return types, or
APIs crossing DLL/ABI boundaries (Refactor first handles those, never
automatically) — nor code guarded by a pre-C++17 standard.

## Links

- [cppreference: std::basic_string_view](https://en.cppreference.com/cpp/string/basic_string_view) — language reference.
- [bugprone-dangling-handle](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/dangling-handle.html)
- [bugprone-stringview-nullptr](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/stringview-nullptr.html)
- [lifetimebound](https://clang.llvm.org/docs/AttributeReference.html#lifetimebound)
- [bugprone-suspicious-stringview-data-usage](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/suspicious-stringview-data-usage.html)
