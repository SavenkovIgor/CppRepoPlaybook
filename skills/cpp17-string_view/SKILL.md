---
name: cpp17-string_view
description: Replace read-only std::string parameters and literal-backed string constants with std::string_view (available since C++17) to remove needless copies, while guarding against the dangling-view and non-null-terminated-buffer bugs this change can introduce.
---

# Modernize to `std::string_view` (C++17)

`std::string_view` is a non-owning, non-null-terminated view over character
data, available since C++17. It replaces `const std::string&` parameters
and literal-backed string constants without copying the underlying data.

## Modernization benefits

- **Performance.** A literal/`const char*` argument bound to a `const
  std::string&` parameter builds a temporary `std::string` (`strlen`
  pass, possible heap allocation); a `string_view` parameter binds
  directly. `string_view::substr()` is O(1); `std::string::substr()`
  allocates and copies. A `constexpr std::string_view` from a literal
  resolves size and pointer at compile time — `std::string` can't,
  since its `const char*` constructor isn't `constexpr` before C++20.
- **API clarity.** The parameter type itself says "read-only, no
  ownership," instead of relying on the reader to notice the `const`.
- **Trade-off, not a pure win.** This is not an error-reduction change:
  it swaps a copy/allocation cost for a lifetime hazard (`const
  std::string&` cannot dangle the way a `string_view` can). That
  hazard is exactly what Tool usage and Refactor first below guard
  against — it is not a reason to skip the modernization, but it is why
  this skill converts selectively instead of everywhere.

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

No clang-tidy check performs this transform end to end — candidate
selection is always manual. Three bugprone checks catch the hazard
classes this skill guards against and belong in the build as a gate
(Procedure step 6; see Links): `bugprone-dangling-handle` (view
outliving its owner), `bugprone-stringview-nullptr` (view from
`nullptr`), `bugprone-suspicious-stringview-data-usage` (`.data()`
without `.size()`). Treat any warning from them as a failed candidate,
not something to suppress.

## No-brainer replacements

**Parameters.** Convert `const std::string&` (or `const char*`) to
`std::string_view` when every one of the following holds at every call
site:

- The function only reads the characters during the call — no storing
  the pointer/view in a member, container, or captured lambda that
  outlives the call.
- The function never passes the data to an API that requires a
  null-terminated buffer (`printf`, `fopen`, most C APIs, `.c_str()`
  users).
- The function does not rely on implicit conversions from `const char*`
  through `std::string` for concatenation or mutation.
- No embedded-NUL assumption is broken (code that assumed
  `strlen`-style truncation may now behave differently — flag this
  explicitly if found, and treat that call site as not safe).

**Literal-backed constants.** A `static const std::string` or
`const char*` constant initialized directly from a string literal, and
never mutated, becomes `constexpr std::string_view`. Doesn't apply to a
constant built from anything other than a literal (concatenation,
`std::to_string`, runtime data) — those aren't compile-time constants.

## Discuss first

Whether to bring in the `sv` literal suffix
(`using namespace std::literals::string_view_literals;`, or the
narrower `using std::string_view_literals::operator""sv;`) to write
`constexpr static auto key = "someKey"sv;`. Ask the user rather than
deciding unilaterally — it's a project style choice, not a correctness
requirement: the unsuffixed `constexpr std::string_view key = "someKey";`
is equally a compile-time constant.

## Refactor first

- **Class/struct members.** Don't store a `string_view` unless the
  owning object's lifetime is provably a subset of the referent's
  lifetime. Refactor path: make that bound explicit (e.g. the owner is
  passed in and outlives the member) before converting; until then,
  keep `std::string`.
- **Return types.** Don't change a return type to `string_view` unless
  the returned view is guaranteed to outlive the call (it aliases a
  literal or a parameter already known to outlive the call). Refactor
  path: change the caller contract to guarantee that, or leave the
  return type as `std::string`.
- **Call sites needing a null-terminated buffer downstream.** No
  refactor path exists on this skill's side — the constraint is the
  external API. State that plainly and keep `std::string`/`const char*`
  there.

## Procedure

1. Verify the C++17+ precondition.
2. Find candidates: (a) functions taking `const std::string&` where the
   body only reads, (b) literal-backed constants (see No-brainer
   above).
3. Check each against No-brainer; sort failures into Discuss first or
   Refactor first with the specific reason (e.g. "call site X passes
   the result to `::open()`, which needs a null-terminated path").
4. Apply the type change plus `#include <string_view>`. For a
   literal-backed constant, raise the `sv`-suffix question before
   adding it.
5. Where a call site relies on `std::string::operator+=` or similar
   mutation through the parameter, it already failed step 3 — revert
   it to `std::string` rather than working around the missing
   mutability.
6. Run clang-tidy with the three bugprone checks (Tool usage) enabled
   on the changed files. Any new warning means the conversion was
   unsafe at that site — revert it there, don't suppress the warning.
7. Build and run the existing test suite. A conversion that compiles
   but changes lifetime behavior can still fail only at runtime
   (use-after-free) or under ASan/UBSan — prefer sanitizers when
   available.
8. Report, per symbol: converted / discussed / deferred for refactor,
   and why.

## Non-goals

This skill does not convert `std::string` members, return types, or
APIs crossing DLL/ABI boundaries (ABI stability may depend on the
concrete type) — those go through Refactor first, never automatically.
It also does not touch code guarded by a pre-C++17 standard.

## Links

- [cppreference: std::basic_string_view](https://en.cppreference.com/w/cpp/string/basic_string_view) — language reference.
- [bugprone-dangling-handle](https://raw.githubusercontent.com/llvm/llvm-project/main/clang-tools-extra/docs/clang-tidy/checks/bugprone/dangling-handle.md)
- [bugprone-stringview-nullptr](https://raw.githubusercontent.com/llvm/llvm-project/main/clang-tools-extra/docs/clang-tidy/checks/bugprone/stringview-nullptr.md)
- [bugprone-suspicious-stringview-data-usage](https://raw.githubusercontent.com/llvm/llvm-project/main/clang-tools-extra/docs/clang-tidy/checks/bugprone/suspicious-stringview-data-usage.md)
