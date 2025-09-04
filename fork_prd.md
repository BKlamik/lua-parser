Problem: Inline numeric index/enum/constant annotations in Lua may be missing, inconsistent with their annotation target, or malformed.
Therefore, they must be validated. 

Users: Mod developers.

Goals: Minimize changes to parser design, AST shape, and existing AST consumers.

Leverage [LPEG](https://www.inf.puc-rio.br/~roberto/lpeg/lpeg.html) for parsing instead of regular expressions, to avoid multiple passes over the input.

Success criteria:

All examples in the [decision doc](2025-08-19-ProjectTableIndexedSystem.md) parse with attached annotations.

Code without annotations parses identically.

Existing tests do not regress.

Tests for annotations cover all valid/invalid annotation patterns and spacing edge cases.
