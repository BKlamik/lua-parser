Problem: Validate numeric index/enum/constant annotations inline in Lua while minimizing changes to existing AST consumers.

Users: Mod developers and test tooling consuming AST for refactor safety and golden diffs.

Goals:

Capture inline annotations adjacent to numeric literals into AST nodes.

Preserve existing AST shapes; add only optional Number.ann field.

Keep comments not pertinent to annotation validation otherwise discarded.

Success criteria:

All examples in the [decision doc](2025-08-19-ProjectTableIndexedSystem.md) parse with attached annotations.

Legacy code parses identically.

Tests cover valid/invalid annotation patterns and spacing edge cases.
