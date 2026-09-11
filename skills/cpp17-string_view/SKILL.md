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

## Preconditions

1. **Confirm the target standard is C++17 or newer** before touching any
   file — check `CMAKE_CXX_STANDARD`, `set(CXX_STANDARD ...)`, or
   `-std=` compiler flags. If the project targets C++14 or earlier, stop
   and say so; do not propose `string_view` there.
2. **No clang-tidy check covers this exact transform** (unlike
   `modernize-use-nullptr` or `modernize-use-override`). Check whether
   `performance-unnecessary-value-param` or `readability-*` already flag
   a given site for another reason — if so, fix that first — then audit
   the remaining call sites manually.

## Where the conversion is safe

Convert a parameter from `const std::string&` (or `const char*`) to
`std::string_view` only when **all** of the following hold at every call
site:

- The function only **reads** the characters during the call — no
  storing the pointer/view in a member, container, or captured lambda
  that outlives the call.
- The function never passes the data to an API that requires a
  null-terminated buffer (`printf`, `fopen`, most C APIs, `.c_str()`
  users). `string_view` is **not** guaranteed null-terminated.
- The function does not rely on implicit conversions from `const char*`
  through `std::string` for concatenation or mutation.
- No embedded-NUL assumption is broken (`string_view` handles embedded
  NULs fine; code that assumed `strlen`-style truncation may now behave
  differently — flag this explicitly if found).

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
2. Find candidate parameters: functions taking `const std::string&` where
   the body only reads (`.find`, `.substr`-then-copy-elsewhere,
   comparisons, iteration, `operator<<`, passing to another
   `string_view`-accepting function).
3. For each candidate, check every call site against "Where the
   conversion is safe" above. If any call site fails a check, do not
   convert that overload — note the specific reason (e.g. "call site X
   passes result to `::open()`, which needs a null-terminated path").
4. Apply the type change plus `#include <string_view>`; do not add
   `using namespace std;` or a namespace alias to shorten it.
5. Where a call site previously relied on `std::string::operator+=` or
   similar mutation through the parameter, that call site is *not* a
   valid candidate — revert it to `std::string` rather than working
   around the missing mutability.
6. Build and run the existing test suite. A `string_view` conversion
   that compiles but changes lifetime behavior will not always fail to
   compile — it fails at runtime (use-after-free) or under ASan/UBSan.
   Prefer running with sanitizers enabled when available.
7. Report, per function: converted / left as `std::string` and why.

## Non-goals

- This skill does not convert `std::string` members, return types, or
  APIs crossing DLL/ABI boundaries (ABI stability may depend on the
  concrete type). It also does not touch code guarded by a pre-C++17
  standard.
