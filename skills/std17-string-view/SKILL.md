---
name: std17-string-view
description: Replace read-only std::string parameters and literal-backed string constants with std::string_view (available since C++17) to remove needless copies, while guarding against the dangling-view and non-null-terminated-buffer bugs this change can introduce.
---

# Modernize to `std::string_view` (C++17)

`std::string_view` is a non-owning, non-zero-terminated view over character data, available since C++17.
It replaces `const std::string&` parameters and literal-backed string constants without copying the underlying data.

## Modernization

### Benefits

- Performance: a literal/`const char*` bound to a `const std::string&` parameter builds a temporary `std::string`
  (`strlen` plus possible heap allocation); `string_view` binds directly. `substr()` is O(1) on a view, allocates on
  a `std::string`.
- Performance: a `constexpr std::string_view` from a literal resolves at compile time.
- API contract clarity: `std::string_view` type explicitly says "read-only, no ownership".
- Style: neat `constexpr static auto key = "someKey"sv;` with the `sv` literal suffix.

### Limitations

- Safety trade-off: Not an error-reduction change: it swaps a copy/allocation cost for a dangling-view hazard
  `const std::string&` didn't have. Tool usage and Refactor first below guard
  against that hazard - it's why this skill converts selectively.

## Goal

Convert the cases of read-only string parameters and literal-backed string constants to `std::string_view`
wherever it is a safe drop-in, raise the rest as a discussion point or a named prerequisite refactor,
and report a decision per symbol - never convert silently, never skip without saying why.

## Preconditions

Confirm the target standard in the project is C++17 or newer.

## Tool usage

Here are some tools that could help to modernize codebase and enforce safe usage of `std::string_view`.

Clang-tidy:
- `bugprone-dangling-handle` - catches a view outliving its owner.
- `bugprone-stringview-nullptr` - catches a view constructed from `nullptr`.
- `bugprone-suspicious-stringview-data-usage` - catches `.data()` used without `.size()`.

Clang compiler attribute:
- `[[clang::lifetimebound]]` - ties a method's return value lifetime to `*this`; enables `-Wdangling` on a caller
  that lets the result outlive the object.

None of these tools could perform the string_view modernization itself - candidate selection is always manual.
Enable the three clang-tidy checks as a gate during Procedure (step 6); treat any warning as a failed candidate
at that site, not something to suppress.

## No-brainer replacements

### Read-only string function arguments

`void foo(const std::string& str);` → `void foo(std::string_view str);`

Convert `const std::string&` (or `const char*`) to `std::string_view` when every one of the following
holds at every call site:

- Function only reads the characters during the call - no storing the pointer/view anywhere that outlives it.
- Function never uses or passes the data to an API needing a zero-terminated buffer
  (`printf`, `fopen`, most C APIs, `.c_str()` users).
- Doesn't rely on implicit `const char*` → `std::string` conversions for concatenation or mutation.
- No embedded-NUL assumption is broken - flag and treat as unsafe if the code assumed `strlen`-style truncation.

### Literal-backed constants

`static const char*       literal = "foo";` → `static constexpr std::string_view literal = "foo";`
`static const std::string literal = "foo";` → `static constexpr std::string_view literal = "foo";`

Replace `static const std::string` or `const char*` constants initialized directly from a string literal, with
`constexpr std::string_view`.
Don't apply to a constant built from anything other than a literal
(concatenation, `std::to_string`, runtime data)- those aren't compile-time constants.

## Discuss first

### `sv` literal suffix for string_view constants

`constexpr std::string_view literal = "foo";` -> `constexpr auto literal = "foo"sv;`

You should ask user if they prefer using the `sv` literal suffix for string_view constants.
Whether to bring in the `sv` literal suffix (`using namespace std::literals::string_view_literals;` or the
narrower `operator""sv` using-declaration) to write `constexpr static auto key = "someKey"sv;`.
Since any form of this is a compile-time constant, this is style, not correctness decision.

### Getter returning a view on an object member

`const std::string& getMember() const;` -> `[[clang::lifetimebound]] std::string_view getMember() const;`

Modernization preconditions: confirm the toolchain is Clang since Clang has an attribute `[[clang::lifetimebound]]`.
If toolchain is not Clang discuss this replacement first: highlight the cases where the lifetime of the returned view
might not be properly enforced by the compiler.

A method returning `std::string_view` into `*this`'s own member has the same dangling risk as a
`const std::string&` getter. But `string_view` does communicate that risk more clearly.
To reduce the risk of dangling views that outlive the parent object, you could use clang's `[[clang::lifetimebound]]`
attribute on the getter and enable the compiler check to enforce the lifetime bound (check the link if needed).

### Enum to string conversions

`enum class Color { Red, Green, Blue };`
`std::string toString(Color color);` -> `std::string_view toString(Color color);`

This conversion is safe in most cases since the returned `std::string_view` typically points to a string literal,
which have a lifetime that spans the entire program.
But it is still a discussion point whether developer want to work with this api and if the enum conversion has
some special cases that require string manipulations. Cuz in this case it is strict no-go for `std::string_view`.

## Refactor first

### Call sites needing a zero-terminated buffer somewhere

- No universal refactor path exists here - the constraint is on external API.
  Highlight the problem to the user, propose fix but keep `std::string`/`const char*` by default.

## Procedure

1. Verify the C++17+ precondition.
2. Find modernization candidates out of list above.
3. Check each against their category: No-brainer; Discuss first or Refactor first
4. Apply the type change plus `#include <string_view>`
5. Check that changes are not relying on `std::string::operator+=` or similar
   string mutation already failed step 3. If they are, revert to `std::string` rather than work around the missing mutability.
6. Run the three bugprone checks (Tool usage) on changed files; any new warning means revert at that site.
7. Build and run the test suite; prefer sanitizers enabled - a
   lifetime bug here can compile clean and only fail at runtime.
8. Report, per symbol: converted / discussed / deferred for refactor, and why.

## Non-goals

This skill does not convert `std::string` members, return types, or
APIs crossing DLL/ABI boundaries (Refactor first handles those, never
automatically) - nor code guarded by a pre-C++17 standard.

## Links

- [cppreference: std::basic_string_view](https://en.cppreference.com/cpp/string/basic_string_view)
- [lifetimebound](https://clang.llvm.org/docs/AttributeReference.html#lifetimebound)
- [clang-tidy: bugprone-dangling-handle](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/dangling-handle.html)
- [clang-tidy: bugprone-stringview-nullptr](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/stringview-nullptr.html)
- [clang-tidy: bugprone-suspicious-stringview-data-usage](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/suspicious-stringview-data-usage.html)
