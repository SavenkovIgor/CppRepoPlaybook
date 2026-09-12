---
name: cpp17-string_view
description: Replace read-only std::string parameters with std::string_view (available since C++17) to remove needless copies, while guarding against the dangling-view and non-null-terminated-buffer bugs this change can introduce.
---

# Modernize to `std::string_view` (C++17)

`std::string_view` is a non-owning, non-null-terminated view over character
data, available since C++17. Used correctly it removes copies at API
boundaries; used carelessly it introduces dangling references that
`const std::string&` never had. This skill proposes the conversion only
where it is safe, and explains in the same edit why any skipped call site
was skipped.

## Why this is faster

- **No temporary `std::string` at the call site.** Passing a `const
  char*` or a string literal into a `const std::string&` parameter
  constructs a temporary `std::string` (a `strlen` pass, plus a heap
  allocation once the argument exceeds the small-string-optimization
  buffer) that is destroyed right after the call. A `string_view`
  parameter binds to the same data directly — no temporary, no
  allocation.
- **O(1) substrings.** `string_view::substr()` returns a new view
  (pointer + length); `std::string::substr()` allocates and copies. This
  matters in any tokenizing or parsing loop, not only at API boundaries.
- **Compile-time-sized literal constants.** `constexpr std::string_view`
  initialized from a string literal is a full constant expression: the
  compiler resolves both the pointer and the length at compile time, no
  runtime length pass at all. `std::string` cannot do this at the
  standard this skill targets — its `const char*` constructor isn't
  `constexpr` before C++20, so a `static const std::string` constant
  still runs a real `strlen` (and possibly a heap allocation) during
  static initialization. Prefer `constexpr std::string_view` for
  read-only string constants backed by a literal.

## Preconditions

1. **Confirm the target standard is C++17 or newer** before touching any
   file — check `CMAKE_CXX_STANDARD`, `set(CXX_STANDARD ...)`, or
   `-std=` compiler flags. If the project targets C++14 or earlier, stop
   and say so; do not propose `string_view` there.
2. **No clang-tidy check finds-and-rewrites this transform** (unlike
   `modernize-use-nullptr` or `modernize-use-override`); call-site
   selection below is still manual. But three bugprone checks catch the
   exact hazard classes this skill guards against — enable them and
   treat any warning as a failed candidate, not something to suppress:
   `bugprone-dangling-handle` (view outliving its owner),
   `bugprone-stringview-nullptr` (view constructed from `nullptr`),
   `bugprone-suspicious-stringview-data-usage` (`.data()` used without
   its `.size()`).

## Where the conversion is safe

Convert a parameter from `const std::string&` (or `const char*`) to
`std::string_view` only when every one of the following holds at every
call site:

- The function only reads the characters during the call — no storing
  the pointer/view in a member, container, or captured lambda that
  outlives the call.
- The function never passes the data to an API that requires a
  null-terminated buffer (`printf`, `fopen`, most C APIs, `.c_str()`
  users). `string_view` carries no guarantee of null termination.
- The function does not rely on implicit conversions from `const char*`
  through `std::string` for concatenation or mutation.
- No embedded-NUL assumption is broken (`string_view` handles embedded
  NULs fine; code that assumed `strlen`-style truncation may now behave
  differently — flag this explicitly if found).

## Also convert: literal-backed constants

A `static const std::string` or `const char*` constant that is
initialized directly from a string literal and never mutated is a
candidate for `constexpr std::string_view` — see "Why this is faster"
above. This does not apply to a constant built from anything other than
a literal (concatenation, `std::to_string`, data read at runtime); those
aren't compile-time constants.

Ask the user, don't decide unilaterally, whether to bring in the `sv`
literal suffix (`using namespace std::literals::string_view_literals;`,
or the narrower `using std::string_view_literals::operator""sv;`) to
write `constexpr static auto key = "someKey"sv;`. It's a namespace/style
choice for the project, not a correctness requirement: the unsuffixed
`constexpr std::string_view key = "someKey";` is equally a compile-time
constant.

## Where to leave it alone

- **Class/struct members**: do not store a `string_view` unless the
  owning object's lifetime is provably a subset of the referent's
  lifetime (e.g. a view into a `constexpr` string literal or a longer-
  lived owner passed in by the caller). Default to keeping `std::string`
  for members; state this explicitly rather than silently skipping.
- **Return types**: do not change a function's return type to
  `string_view` unless the returned view is guaranteed to outlive the
  call (e.g. it aliases a string literal or a parameter already known to
  outlive the call). Prefer leaving `std::string` returns as-is.
- **Any call site requiring a null-terminated buffer** downstream — keep
  `std::string`/`const char*` there and say why.

## Procedure

1. Verify the C++17+ precondition above.
2. Find candidates: (a) functions taking `const std::string&` where the
   body only reads (`.find`, `.substr`-then-copy-elsewhere, comparisons,
   iteration, `operator<<`, passing to another `string_view`-accepting
   function), and (b) literal-backed `static const std::string` /
   `const char*` constants (see "Also convert" above).
3. For each candidate, check every call site against "Where the
   conversion is safe" above. If any call site fails a check, do not
   convert that overload — note the specific reason (e.g. "call site X
   passes result to `::open()`, which needs a null-terminated path").
4. Apply the type change plus `#include <string_view>`. For a
   literal-backed constant (see above), ask about the `sv` suffix before
   adding it.
5. Where a call site previously relied on `std::string::operator+=` or
   similar mutation through the parameter, that call site is *not* a
   valid candidate — revert it to `std::string` rather than working
   around the missing mutability.
6. Run clang-tidy with `bugprone-dangling-handle`,
   `bugprone-stringview-nullptr`, and
   `bugprone-suspicious-stringview-data-usage` enabled on the changed
   files. Any new warning means the conversion was unsafe at that site —
   revert it there rather than suppressing the warning.
7. Build and run the existing test suite. A `string_view` conversion
   that compiles but changes lifetime behavior will not always fail to
   compile or trip the checks above — it can still fail only at runtime
   (use-after-free) or under ASan/UBSan. Prefer running with sanitizers
   enabled when available.
8. Report, per function: converted / left as `std::string` and why.

## Non-goals

- This skill does not convert `std::string` members, return types, or
  APIs crossing DLL/ABI boundaries (ABI stability may depend on the
  concrete type). It also does not touch code guarded by a pre-C++17
  standard.
