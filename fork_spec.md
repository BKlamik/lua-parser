Requirements:
Inline annotations to capture: After Number, String, and Id tokens. One annotation max; first wins. Allow optional whitespace including newline between token and annotation. Support unary numbers by attaching to the underlying Number literal.

Annotation forms:

Field (F): F:Qualified.Type:FieldName

Enum (E): E:Qualified.Type:EnumValueName

Constant (C): C:Qualified.Type.ConstantName (no second colon)

Validation timing: Structural parse is enough. Semantic validation happens in a separate pass using sideband LuaCATS data. If a text sequence starts with a valid annotation prefix (--[[F:...]], --[[E:...]], --[[C:...]]) but is malformed, fail fast with a clear parser error.

LuaCATS integration: Do not keep general comments in AST. Extract LuaCATS lines (---@class, ---@field, ---@alias, ---|) into sideband metadata for semantic validation. This can be a separate pre/post scan, not part of the AST.

Secondary validation rule (not fully spec’d yet): If a variable is typed as a class that provides custom field annotations, then bracket indexing on that variable must use your inline field annotations; otherwise, fail in the validation pass.

AST surface
New optional field: node.anno on Number, String, or Id nodes.

Fields: kind ("F" | "E" | "C"), type (Qualified.Type), name (Field/Enum/Const), pos, end_pos.

Positions: Number/String/Id pos/end_pos remain unchanged; anno.pos/end_pos span the annotation comment.

No other AST shapes change. All existing tags and node structures remain intact.

Sideband LuaCATS schema
Class registry: { [Qualified.Type] = { kind = "class", fields = { [FieldName] = { index = n, luaType = "number", doc = "..." } } } }

Enum registry: { [Qualified.Type] = { kind = "enum", values = { [Name] = literal, [literal] = Name } } }

Const registry: { [Qualified.Type] = { kind = "const", values = { [Name] = literal } } }

Typing hints (optional for secondary rule):

Var types: From ---@type X.Y.Z var, ---@param var X.Y.Z, ---@return, etc.

Implementation note: Build this via a separate, lightweight scan over the source that collects lines starting with ---@..., not via AST changes.

Validation passes
Pass 1: Inline annotation structural correctness

Handled by grammar as above. Malformed “starts-like-annotation” after a token errors out early.

Pass 2: Semantic validation using sideband metadata

F (field): Check registry[type].fields[name] exists; if Number annotated, ensure literal equals fields[name].index.

E (enum): Check registry[type].values[name] exists; if Number annotated, ensure literal equals that value.

C (const):

If annotated on String literal, ensure string equals registry[type].values[name].

If annotated on Number, ensure number equals registry[type].values[name].

If annotated on Id, ensure identifier text equals the constant’s value when the value is a bare identifier string such as 'B2'. For other constant value types, define a local policy (e.g., disallow).

Secondary rule (sketch): If a variable is typed as Qualified.Type where that type has field annotations, then any bracket indexing on that variable must use an inline F annotation at the index expression. Traverse Index nodes; when base is an Id (or resolved Var) with type T, assert the index expr’s Number/String/Id carries a matching F annotation; otherwise, report a violation with position and suggested fix.