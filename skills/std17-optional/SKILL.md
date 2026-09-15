---
name: std17-optional
description: Replace sentinel-value returns, bool-plus-out-parameter pairs, and pointers used only to signal absence with std::optional (available since C++17), while guarding against the unchecked-access and identity-loss bugs this change can introduce.
---

# Modernize to `std::optional` (C++17)

`std::optional<T>` is a value type that either holds a `T` or holds nothing, available since C++17.
It replaces a sentinel value, a `bool` return paired with an out-parameter, or a pointer used purely
to mean "no value", making optionality part of the signature instead of a convention callers must remember.

## Modernization

### Benefits

- Correctness: the type forces the "may be absent" case into the signature; a caller can't read the result
  without going through `has_value()`/`operator bool()`/`.value()`, unlike a sentinel that's silently usable
  as if it were valid.
- Bug reduction: removes the temptation to use a nullable `T*` for optionality, which mixes "may be absent"
  with ownership/aliasing semantics the reader has to disambiguate from context.
- Composability: chains through `.value_or()` for a fallback without a manual `if`.

### Limitations

- Performance trade-off: `sizeof(std::optional<T>)` is never smaller than `sizeof(T)` and commonly adds
  a full alignment unit for the engaged flag; returning/copying `optional<T>` by value always
  copies/moves the whole `T`, unlike a pointer.
- Safety trade-off: replacing a pointer with `optional<T>` changes reference semantics to value semantics -
  the caller gets an independent copy, not an alias into the original object. Code relying on pointer
  identity or on mutating through the pointer breaks silently if converted.
- `.value()` throws `std::bad_optional_access`; `operator*`/`operator->` are UB on an empty optional - same
  class of unchecked-access hazard as a null pointer dereference if code dereferences reflexively.
- `std::optional<T&>` isn't available until the reference-support paper lands in C++26; pre-C++26 a
  reference-like optional needs `std::optional<std::reference_wrapper<T>>`.

## Goal

Convert the cases of sentinel-value returns, `bool` + out-parameter pairs, and non-aliasing nullable
pointers to `std::optional<T>` wherever it's a safe drop-in, raise the rest as a discussion point or a
named prerequisite refactor, and report a decision per symbol - never convert silently, never skip
without saying why.

## Preconditions

Confirm the target standard in the project is C++17 or newer.

## Tool usage

Clang-tidy:
- `bugprone-unchecked-optional-access` - flow-sensitive check that catches `value()`/`operator*`/`operator->`
  called without a preceding `has_value()`/`operator bool()` check.
- `bugprone-optional-value-conversion` - catches extracting `.value()` only to immediately rewrap it into a
  new optional of the same type; useful for catching a leftover round-trip after conversion.

Neither tool performs the optional modernization itself - candidate selection is always manual. Enable
both checks as a gate during Procedure (step 6); treat any warning as a failed candidate at that site,
not something to suppress.

## No-brainer replacements

### Sentinel-value return

`int find(...); // returns -1 when not found` → `std::optional<int> find(...);`

Convert when the sentinel is not a valid in-domain value, and every call site only ever compares the
result against the sentinel (no arithmetic that assumes a value is present).

### `bool` return plus out-parameter

`bool tryGet(T& out);` → `std::optional<T> tryGet();`

Convert when `T` is cheap enough to move/copy out, and the out-parameter exists only to carry the result -
not to reuse caller-owned storage across repeated calls (that's a Refactor-first case below).

## Discuss first

### Nullable pointer meaning "no value", not aliasing

`const Config* findConfig(...);` (returns a pointer into an internally-owned object) → `std::optional<Config>`

Converting copies the whole object out on every call instead of returning an internal reference. Ask the
user whether `Config` is cheap enough to copy and whether any caller relies on the returned pointer's
identity (comparing addresses, or expecting the pointee to observe mutations through the original owner).
If either holds, `std::optional<std::reference_wrapper<const Config>>` keeps the optionality without a
copy - propose it as the alternative and let the user pick.

### Class member conversion

Changing a member from a nullable pointer or a flag+value pair to `optional<T>` changes the class's
default-constructed state and can implicitly enable copyability the pointer version didn't have. Confirm
with the user before touching a member whose enclosing class has hand-written copy/move/comparison logic.

## Refactor first

### Out-parameter used to reuse caller-owned storage

An out-parameter kept to avoid a per-call allocation (e.g. filling a caller's buffer in a hot loop) can't
become an `optional<T>` return without losing that reuse. Highlight the cost, propose it, but keep the
out-parameter by default unless the user accepts the extra allocation.

### Code relying on pointer identity

If a nullable-pointer call site needs the returned pointer's identity or aliasing (Discuss first above
confirmed it), the conversion doesn't apply - out of scope for this skill until that dependency is
refactored away.

## Procedure

1. Verify the C++17+ precondition.
2. Find modernization candidates out of the list above.
3. Check each against their category: No-brainer; Discuss first or Refactor first.
4. Apply the type change plus `#include <optional>`.
5. Check call sites don't rely on the sentinel's in-domain meaning or on pointer identity/aliasing already
   ruled out in step 3. If they do, revert at that site rather than work around the lost identity.
6. Run the two bugprone checks (Tool usage) on changed files; any new warning means revert at that site.
7. Build and run the test suite; prefer sanitizers enabled - an unchecked-access bug here can compile
   clean and only fail at runtime.
8. Report, per symbol: converted / discussed / deferred for refactor, and why.

## Non-goals

This skill does not convert absence that needs to carry a reason (use `std::expected`/`std::variant`
instead of `std::optional` for an error case), does not introduce `std::optional<T&>` (unavailable before
C++26), and does not touch a pointer whose caller relies on identity or aliasing - nor code guarded by a
pre-C++17 standard.

## Links

- [cppreference: std::optional](https://en.cppreference.com/cpp/utility/optional)
- [clang-tidy: bugprone-unchecked-optional-access](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/unchecked-optional-access.html)
- [clang-tidy: bugprone-optional-value-conversion](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/optional-value-conversion.html)
