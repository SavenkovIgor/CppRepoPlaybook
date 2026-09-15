# Modernize C++

AI plugin skills focused on modernizing a C++ codebase — each skill targets either a specific language standard feature or a general C++ best practice.

## Principles

- **Deterministic tools priority.** If modernization could be done with an existing tool like clang-tidy or clang-format, skills will first propose enabling the relevant rule before making manual edits.
- **C++ is not C.** When used as "C with classes", C++ becomes unsafe and costly to support — due to raw pointers, manual lifetime management, out-parameters instead of return values, and so on. Modern C++ instruments like RAII, smart pointers, functional wrappers, etc., exist specifically to eliminate whole classes of bugs that C leaves to the programmer. When C++ has a better type, abstraction, or approach for the job, skills propose to use it — not preserve C-era code out of misplaced caution.
- **Correctness over modernity.** When there is a conflict between making code modern and keeping it correct, correctness takes priority. If some code piece requires rework before it can be modernized, skills will explicitly say so before any edits.
- **Readability.** Skills must be readable by both a human and an AI: short, focused on one feature or change, written in clear and unambiguous language, and grounded in the text of the standard and in industry best practices.

## Sources

- [Agent Plugins 1.0 Specification](https://github.com/agentplugins/agent-plugins-spec) — open, vendor-neutral standard for packaging Agent Skills and MCP servers into portable plugins.
