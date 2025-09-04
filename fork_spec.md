# Requirements:
Capture inline annotations after `Number`, `String`, and `Id` tokens. One annotation max; first wins. Allow optional whitespace including newline between token and annotation. Support unary numbers by attaching to the underlying `Number` literal.

Annotation forms:
  - Field (F): 'F:FullName:SimpleId'
  - Enum (E): 'E:FullName:SimpleId'
  - Constant (C): 'C:FullName' (no second colon)

Validation sequencing: Structural parse is enough without full validation. Semantic validation happens in a separate pass using sideband LuaCATS data. If a text sequence starts with a valid annotation prefix (--[[F:...]], --[[E:...]], --[[C:...]]) but is malformed, fail fast with a clear parser error.

LuaCATS integration: Do not start capturing general comments in AST. Instead, extract LuaCATS lines (---@class, ---@field, ---@alias, ---|) into sideband metadata for semantic validation. This can be a separate pre/post scan, if integration with AST becomes too complex.

If validating the project fails for LuaCATS class annotations that cannot be changed (i.e. library code), then validation will typically be skipped for class annotations, unless a newly introduced tag is present in the class comment area to enable validation, i.e.
```lua
---@class M.Disk # VALIDATE
---@field public [1] integer # GameId
```

Missing annotation validation: If a variable is typed as a class that provides custom field annotations and comments, then bracket indexing on that variable must use the inline field annotations; otherwise, fail in the validation pass.
This is a best effort, and only catches the following cases:
- Locals with direct type annotations
- Direct assignments from known sources, including annotated class tables
- Function parameters with annotated types
- Function returns with annotated types

## AST surface
New optional field: node.anno on `Number`, `String`, and `Id` nodes.

Fields: kind ("F" | "E" | "C"), FullName (Field/Enum/Const), SimpleId (Field/Enum), pos, end_pos.

Positions: `Number`, `String`, and `Id` nodes' pos/end_pos remain unchanged; anno.pos/end_pos span the annotation comment.

For LuaCATS, we’ll keep the core AST shape intact and attach a sideband “meta” bundle at the root.
No comment nodes get emitted into the normal statement list.

- Attach on the root Block node:
  - block.meta = { cats = CatsDB }
- CatsDB schema:
  - cats.classes: map<string ClassName, { pos, end_pos, base?: string, validate: boolean, fields: map<string|number, { luaType : string, pos, end_pos, indexName?: string }>}>
  - cats.pendingClass: { name: string, pos, end_pos, base?: string, validate: boolean } ; populated by ---@class; used for ---@field
  - cats.aliases: map<string AliasName, { pos, end_pos, values: map<string, { value: string, comment: string, pos, end_pos }>}>
  - cats.constants: map<string ConstName, { pos, end_pos, value: string }>
  - cats.pendingVarType: { type: string, pos, end_pos } ; populated by ---@type; relocated last stored to pendingTypes on declaration
  - cats.pendingVarParams: map<string, { type: string, pos, end_pos }> ; populated by ---@param; relocate group to pendingParams on declaration
  - cats.pendingVarReturn: [{ type: string, pos, end_pos }] ; populated by ---@return; relocate group to pendingReturn on declaration
  - cats.pendingTypes: [{ type: string, pos, end_pos }] ; populated by `local` and `set` variable names on declaration; relocated to env.varTypeStack on scope push/pop
- Inline annotations (already present):
  - exp.anno = { kind = "F"|"E"|"C", type = "FullName", name = "SimpleId"|nil, pos, end_pos }
- No changes to existing node tags, only the new block.meta field.

No other AST shapes change. All existing tags and node structures remain intact.

## Sideband LuaCATS schema
Class registry: { [FullName] = { kind = "class", fields = { [SimpleId] = { index = literal, ... } } } }

Alias registry: { [FullName] = { kind = "alias", values = { [SimpleId] = { value = literal, ... } } } }

Const registry: { [FullName] = { kind = "const", value = literal } }

Typing (for missing annotation validation):

Var types: From ---@class, ---@field, ---@type, ---@param, ---@return, etc.
Scoped variable names are managed with a stack-based environment to accurately track scoped variable name types. 

## Validation passes
### Pass 1:

Inline annotation structural correctness
- Handled by grammar as above. Malformed “starts-like-annotation” after a token errors out early.

Sideband LuaCATS data population and correctness
- Duplicate class/field/alias/const names are allowed; the last declaration wins.

### Pass 2:

Semantic validation using sideband data after parse

- F (field): Check class cats.classes[FullName].fields[SimpleId] exists; if Number annotated, ensure literal equals fields[SimpleId].index. Fail if class or field does not exist.

- E (enum): Check alias cats.aliases[FullName].values[SimpleId] exists; if Number annotated, ensure literal equals that value. Fail if alias or value does not exist.

- C (const):
  - If annotated on String literal, ensure string equals cats.constants[FullName].value. Fail if constant does not exist.
  - If annotated on Number, ensure number equals cats.constants[FullName].value. Fail if constant does not exist.
  - If annotated on Id, ensure identifier text equals the constant’s value when the value is a bare identifier string such as 'B2'. Fail if constant does not exist.

Missing annotations
If a variable is typed as FullName class where that type has field annotations, then any bracket indexing on that variable must use an inline F annotation at the index expression. Traverse Index nodes; when base is an Id (or resolved Var) with type annotation of class, report a violation with position when the index expr’s Number/String/Id is not inline annotated by an F annotation.

## Grammar extensions to capture LuaCATS

We extend the lexer’s Skip/Comment to recognize and parse “doc lines” and push items into a mutable CatsDB during parse via Carg(2) and functions.

Key idea: parse doc directives inside comments, do side effects only, consume them as whitespace (so they don’t appear in the AST), and collect into CatsDB.

Add to the top of parser.lua (near other locals):
```lua

  - Initialize cats.classes, aliases, constants, etc.

-- Sideband accumulator: passed as Carg(2)
local function cats_add_class(cats, fullName, base, pos, end_pos)
  cats.classes[fullName] = { pos = pos, end_pos = end_pos, base = base, validate = true }
  cats.currentClass = fullName
end

local function cats_add_field(cats, simpleId, literal, pos, end_pos)
  if not cats.currentClass then --[[Fail FieldWithoutClass]] end
  -- Manage index vs. string simpleId
  cats.classes[cats.currentClass].fields[simpleId] = {index = literal, pos = pos, end_pos = end_pos}
end

local function cats_add_alias(cats, fullName, pos, end_pos, value)
  if value then
    cats.constants[fullName] = {pos = pos, end_pos = end_pos, value = value}
  else
    cats.aliases[fullName] = {pos = pos, end_pos = end_pos, values = {}}
    cats.currentAlias = fullName
  end
end

local function cats_add_alias_value(cats, simpleId, value, pos, end_pos)
  if not cats.currentAlias then --[[Fail NoAlias]] end
  local alias = cats.aliases[cats.currentAlias]
  alias.values[simpleId] = {value = value, pos = pos, end_pos = end_pos}
end

local function cats_add_pending_var_type(cats, pname, pos, end_pos)
  cats.pendingVarType = { type = pname, pos = pos, end_pos = end_pos }
end

local function cats_add_pending_var_params(cats, pname, ptype, pos, end_pos)
  cats.pendingVarParams[pname] = { type = ptype, pos = pos, end_pos = end_pos }
end

local function cats_add_pending_var_return(cats, rtype, pos, end_pos)
  table.insert(cats.pendingVarReturn, { type = rtype, pos = pos, end_pos = end_pos })
end

local function cats_add_pending_type(cats, typex, pos, end_pos)
  table.insert(cats.pendingTypes, { type = typex, pos = pos, end_pos = end_pos })
end
```

Then define the doc tokens. Keep them simple and robust; accept common LuaCATS patterns:

```lua
-- Identifiers and simple type exprs for LuaCATS
local alpha    = R("az","AZ")
local digit    = R("09")
local identStart = alpha + P"_"
local identRest  = alpha + digit + P"_" + P"."
local newline  = P("\r")^-1 * P("\n")
local DocId = identStart * identRest^0              -- dotted names allowed in types/classes
local DocTypeExpr = C((1 - S"\r\n")^1)                         -- keep raw type expression (no newline)

local DocBOL = P("---") * optspace * P("@")

-- @type: ---@type TypeExpr   (applies to next local/assignment binding, resolved later)
local DocType = DocBOL * P"type" * space^1 * C(DocTypeExpr)

-- identifiers

-- field key: either [number] or an alphanumeric identifier
local BracketedIndex = P"[" * C(digit^1) * P"]"         -- capture only digits; strip [ ]
local IdentKey       = C(identStart * identRest^0)

-- visibility (optional; capture token if present)
local Visibility = C(P"public" + P"protected" + P"private")

-- type expression: everything up to '#' or end-of-line (trim later if needed)
local TypeExpr = C((1 - P"#" - newline)^1)

-- field name token (required): stops at first unsupported char
local FieldNameToken = C(identStart * identRest^0)

-- slerp: the remainder of the line after the name (including leading spaces)
local Slerp = C(optsp * (1 - newline)^0)

-- Captures:
--   1: ClassName
--   2: BaseName (optional)
--   3: slerp (possibly empty), including leading spaces (e.g., "  # VALIDATE")
local DocClass =
  DocBOL * P"class" * sp
  * C(DocId)
  * (optsp * P":" * optsp * C(DocId))^-1
  * Slerp

-- Key alternatives with kind tagging
local KeyIndex =
      Cg(Cc("index"), "keykind")
  *   Cg(BracketedIndex, "key")           -- key = digits, kind = "index"

local KeyName  =
      Cg(Cc("name"),  "keykind")
  *   Cg(IdentKey,    "key")              -- key = identifier, kind = "name"

-- Full @field line
-- Captures (named):
--   keykind: "index" | "name"
--   key:     digits (for index) or identifier (for name)
--   vis:     visibility token if present; nil otherwise
--   type:    type expression (raw)
--   fname:   field name token after '#'
--   slerp:   remainder after the name (may be empty)
local DocField =
  DocBOL * P"field" * sp
* (Cg(Visibility, "vis") * sp)^-1
* (KeyIndex + KeyName) * sp
* Cg(TypeExpr, "type")
* optsp * P"#" * optsp
* Cg(FieldNameToken, "fname")
* Cg(Slerp, "slerp")


-- @alias: ---@alias AliasName TypeExpr|value1|value2|...
local identifier = (R("AZ", "az", "09") + S("._"))^1
local DocAliasSingle = P("---@alias") * space^1 * C(identifier) * space^1 * C((1 - newline)^1)
local DocAliasHeader = P("---@alias") * space^1 * C(identifier)

-- Union entry: ---| 1 # Engine
local DocAliasUnion = P("---|") * space^0 *
  C((1 - S("\r\n#"))^1) * -- value
  (space^1 * P("#") * C((1 - S("\r\n"))^0))^-1 -- optional comment

-- Full block: header + one or more union entries
local DocAliasUnionBlock = Cmt(Cp() * DocAliasHeader, function(_, pos, name)
  return true, name, pos
end)
* Ct(DocAliasUnion^1) / function(unions)
    local result = {}
    for _, entry in ipairs(unions) do
      table.insert(result, {
        value = entry[1],
        comment = entry[2] or nil
      })
    end
    return result
  end

local DocAliasHeader = P"---@alias" * space^1 * C(FullName)

```

Wire these into Comment with side-effects, using Cmt and Carg(2). We also want to support “---@field Class.Field Type” pattern to avoid building class-context in the grammar; we can split by the first dot to get Class and Field when present, else accept as a “global field” (ignored or stored if you want).

Replace Comment rule with a version that first tries doc lines, and if matched, mutates CatsDB then consumes to EOL:

```lua
-- In G:
  -- lexer
  Skip     = (V"Space" + V"DocComment" + V"Comment")^0;
  Space    = space^1;

  -- Recognize LuaCATS doc directives; side effects into CatsDB (Carg(2))
  DocComment =
      Cmt((Cp() * DocClass), function(s,i,pos, cname, base)
        local cats = Carg(2):get()  -- pattern-time access won't work; use closures instead
        return false
      end);

```

Because Carg access is not available directly within plain function bodies without wrapping, implement DocComment as a sum of patterns each with its own Cmt and closure to the cats table using Carg and Cmt’s extra captures. The standard way:

Define helper constructors above grammar:
```lua
local function DoDocClass(cname, base, cats, pos, endpos)
  cats_add_class(cats, cname, base, pos, endpos); return true
end

local function DoDocField(fqname, typex, cats, pos, endpos)
  local cls, fname = fqname:match("^(.-)%.([A-Za-z_][A-Za-z0-9_]*)$")
  if cls and fname then cats_add_field(cats, cls, fname, typex, pos, endpos) end
  return true
end

local function DoDocAlias(pos, name, unionStr, cats)
  local union = {}
  for value in unionStr:gmatch("[^|]+") do
    table.insert(union, { value = value:match("^%s*(.-)%s*$") })
  end
  cats.aliases[name] = { union = union, pos = pos }
end

local function DoDocAliasUnion(pos, name, unionEntries, cats)
  local union = {}
  for _, entry in ipairs(unionEntries) do
    table.insert(union, {
      value = entry[1],
      comment = entry[2] or nil
    })
  end
  cats.aliases[name] = { union = union, pos = pos }
end

local function DoDocEnum(ename, raw, cats, pos, endpos)
  local values = {}
  for v in raw:gmatch("[^|%s]+") do values[v] = true end
  cats_add_enum(cats, ename, values, pos, endpos); return true
end

local function DoDocType(typex, cats, pos, endpos)
  cats_add_pending_type(cats, typex, pos, endpos); return true
end
```

Now the actual pattern (note: we need Cp() and the end position; we can get end by consuming until newline and using Cp again, or compute end as position before newline — we’ll just use start pos, it’s usually sufficient):

```lua
-- Consume line to EOL regardless of directive success
local EOL = (P"\r"^-1 * P"\n") + -P(1)

DocClassLine  = Cmt(Cp() * DocClass * Carg(2), function(s, i, pos, cname, base, cats)
                      DoDocClass(cname, base, cats, pos, i); return true
                    end) * (P(1) - EOL)^0 * EOL
DocFieldLine  = Cmt(Cp() * DocFieldLine * Carg(2), function(s, i, pos, fqname, typex, cats)
                      DoDocField(fqname, typex, cats, pos, i); return true
                    end) * (P(1) - EOL)^0 * EOL
DocAliasLine  = Cmt(Cp() * DocAlias * Carg(2), function(s, i, pos, aname, typex, cats)
                      DoDocAlias(aname, typex, cats, pos, i); return true
                    end) * (P(1) - EOL)^0 * EOL

local DocAliasHeader = Cmt(Cp() * P("---@alias") * space^1 * C(identifier), function(_, i, pos, name)
  return true, name, pos
end)

local DocAliasUnionLine = Cmt(Cp() * P("---|") * space^0 *
  C((1 - S("\r\n#"))^1) * -- value
  (space^1 * P("#") * C((1 - S("\r\n"))^0))^-1 * Carg(2),
  function(_, i, pos, value, comment, cats)
    return true, { value = value:match("^%s*(.-)%s*$"), comment = comment, pos = pos }
  end
) * (P(1) - EOL)^0 * EOL

local DocAliasUnionBlock = DocAliasHeader * Carg(2) / function(name, pos, cats)
  cats.aliases[name] = { union = {}, pos = pos }
  cats._currentAlias = name
end
* DocAliasUnionLine^1 / function(entries)
  local name = cats._currentAlias
  for _, entry in ipairs(entries) do
    table.insert(cats.aliases[name].union, entry)
  end
  cats._currentAlias = nil
end

DocTypeLine   = Cmt(Cp() * DocType * Carg(2), function(s, i, pos, typex, cats)
                      DoDocType(typex, cats, pos, i); return true
                    end) * (P(1) - EOL)^0 * EOL

DocComment = DocClassLine + DocFieldLine + DocAliasLine + DocAliasUnionBlock + DocTypeLine
```

Finally, make Skip prefer DocComment over generic Comment:
```lua
Skip     = (V"Space" + V"DocComment" + V"Comment")^0;
Comment  = P"--" * V"LongStr" / function () return end
         + P"--" * (P(1) - P"\n")^0 * (P"\n" + -P(1));
```

Parser.parse integration:
```lua
function parser.parse (subject, filename)
  local errorinfo = { subject = subject, filename = filename }
  local cats = { classes = {}, aliases = {}, constants = {}, pendingVarReturn = {}, pendingTypes = {} }
  lpeg.setmaxstack(1000)
  local ast, label, errorpos = lpeg.match(G, subject, nil, errorinfo, cats) -- errorinfo as 4th arg is Carg(1), cats as 5th arg is Carg(2)
  ...
  local ok_or_ast, err = validate(ast, errorinfo)  -- unchanged call
  if type(ok_or_ast) == "table" then
    ok_or_ast.meta = ok_or_ast.meta or {}
    ok_or_ast.meta.cats = cats
  end
  return ok_or_ast, err
end
```

Notes:
- We’re using two extra args to the matcher: errorinfo (already in your code) and the new cats table. lpeglabel supports passing multiple Carg(...) slots; we consumed with Carg(2).

---

### Validation pass design

We extend validator to leverage meta.cats to validate inline annotations and enforce typed-index rules. Keep original control-flow validations intact.

Core responsibilities:

1) Validate that inline annotations reference known types and values
- For node.anno.kind == "F":
  - anno.type must resolve to a known type:
    - meta.cats.classes[node.anno.type] must exist
  - anno.name must resolve to a known field in the class:
    - meta.cats.classes[node.anno.type].fields[node.anno.name] must exist
  - ensure literal equals fields[node.anno.name].index
    - node[1] must be equal to meta.cats.classes[node.anno.type].fields[node.anno.name].index
- For node.anno.kind == "E":
  - anno.type must resolve to a known type:
    - meta.cats.aliases[node.anno.type] must exist
  - anno.name must resolve to a known value in the alias:
    - meta.cats.aliases[node.anno.type].values[node.anno.name] must exist
  - ensure literal equals values[node.anno.name].value
    - node[1] must be equal to meta.cats.aliases[node.anno.type].values[node.anno.name].value
- For node.anno.kind == "C":
  - anno.type must resolve to a known type:
    - meta.cats.constants[node.anno.type] must exist
  - ensure literal equals value
    - node[1] must be equal to meta.cats.constants[node.anno.type].value

2) Bind @type pending docs to nearest relevant declaration/assignment
- Strategy: while traversing statements in order, maintain a queue cats.pendingTypes (sorted by position).
- When you encounter:
  - Local { NameList, ... }: if there is a pending type whose pos is on the immediately previous line or the same line before the local, bind its type to each name in NameList if there’s exactly one name; ignore multi-name (or extend to “---@type A, B, C” later).
    - cats.varTypes[name] = { type = pending.type, pos = pending.pos, end_pos = pending.end_pos }
    - pop the pending item
  - Set { VarList, ... }: similarly bind to the first LHS var if exactly one; otherwise skip.

3) Enforce “typed index must be annotated with correct field”
- Rule: If a variable v has a class type T (from env.varTypeStack[v]) with cats.classes[T].validate = true and the program uses an index on v, then the index key expression must carry an inline field annotation of kind "F", with:
  - anno.type == T
  - anno.name == known field in cats.classes[T].fields

Minimal extensions in validator.lua:

- Load cats: local cats = ast.meta and ast.meta.cats
- Track varTypes during traversal in a simple lexical map stack to respect scope

Enforcement points:
- In traverse_var(env, var): when tag == "Index"
  - If var[1].tag == "Id" and env.varTypeStack[var[1][1]] exists and resolves to a class T that exists with cats.classes[T].validate = true:
    - Validate var[2] is a String/Id/Expr whose .anno exists and .anno.kind == "F"
    - Check .anno.type == T
    - Check .anno.name exists in cats.classes[T].fields
    - On failure: return syntaxerror(errorinfo, var[2].pos, "expected inline field annotation --[[F:"..T..":"..field.."]] for indexed access")
- In traverse_exp/env where appropriate, validate node.anno references (C/E/F) using resolve_type().

Binding @type while traversing:
- Maintain local function bind_pending_type_to_names(names, pos)
  - Peek at cats.pendingTypes[1]
  - If it exists and its pos is on previous line or same line before pos, apply to first name and table.remove(pending, 1).
- Call when visiting Local and Set before recursing.

---

### Pseudocode for validator additions

```lua
-- In validator.lua
local function resolve_type(cats, name)
  if cats.classes[name] then return "class", cats.classes[name] end
  if cats.aliases[name] then return "alias", cats.aliases[name] end
  return nil, nil
end

local function validate_inline_anno(env, cats, node)
  local a = node.anno; if not a then return true end
  local kind, def = resolve_type(cats, a.type)
  if not kind then return nil, syntaxerror(env.errorinfo, a.pos, "unknown type '"..a.type.."' in inline annotation") end

  if a.kind == "E" then
    if kind ~= "enum" then return nil, syntaxerror(env.errorinfo, a.pos, "annotation kind 'E' requires enum type '"..a.type.."'") end
    if a.name and not def.values[a.name] then
      return nil, syntaxerror(env.errorinfo, a.pos, "unknown enum value '"..a.name.."' for enum '"..a.type.."'")
    end
  elseif a.kind == "F" then
    if kind ~= "class" then return nil, syntaxerror(env.errorinfo, a.pos, "annotation kind 'F' requires class type '"..a.type.."'")
    end
    if a.name and not def.fields[a.name] then
      return nil, syntaxerror(env.errorinfo, a.pos, "unknown field '"..a.name.."' for class '"..a.type.."'")
    end
  elseif a.kind == "C" then
    -- allow class/alias/enum; no extra checks here
  end
  return true
end

-- Pending @type binder
local function maybe_bind_pending_type(cats, env, names, pos)
  local p = cats.pendingTypes[1]; if not p then return end
  local pline = lineno(env.errorinfo.subject, p.pos)
  local thisline = lineno(env.errorinfo.subject, pos)
  if (pline == thisline - 1) or (pline == thisline) then
    local nameNode = names[1]; if nameNode and nameNode.tag == "Id" then
      cats.varTypes[nameNode[1]] = { type = p.type, pos = p.pos, end_pos = p.end_pos }
      table.remove(cats.pendingTypes, 1)
    end
  end
end

-- Hook in traverse_*:
function traverse_var(env, var)
  local cats = env.cats
  -- Validate inline anno on Id/Index nodes themselves (optional)
  local ok, msg = validate_inline_anno(env, cats, var); if not ok then return ok, msg end

  if var.tag == "Index" then
    local base = var[1]; local key = var[2]
    -- Validate inline anno on key
    local ok2, msg2 = validate_inline_anno(env, cats, key); if not ok2 then return ok2, msg2 end

    if base.tag == "Id" then
      local vt = cats.varTypes[base[1]]
      if vt then
        local kind, classDef = resolve_type(cats, vt.type)
        if kind == "class" then
          local ka = key.anno
          if not (ka and ka.kind == "F" and ka.type == vt.type and ka.name and classDef.fields[ka.name]) then
            local msg = string.format("expected inline field annotation --[[F:%s:<Field>]] on indexed access", vt.type)
            return nil, syntaxerror(env.errorinfo, key.pos, msg)
          end
        end
      end
    end
    -- Recurse as before
    local s, m = traverse_exp(env, base); if not s then return s, m end
    s, m = traverse_exp(env, key); if not s then return s, m end
    return true
  end
  ...
end

-- In traverse_stm on Local and Set, before recursing:
if tag == "Local" then
  maybe_bind_pending_type(env.cats, env, stm[1], stm.pos)
  -- proceed as today
elseif tag == "Set" then
  maybe_bind_pending_type(env.cats, env, stm[1], stm.pos)
  -- proceed as today
end

-- In traverse_exp, before dispatching by tag, validate_inline_anno on Number/String/Id nodes (they carry anno)
local ok, msg = validate_inline_anno(env, env.cats, exp); if not ok then return ok, msg end
```

Initialize env.cats at the start of validate:
```lua
local cats = ast.meta and ast.meta.cats or { classes={},aliases={},varTypes={},pendingTypes={} }
env.cats = cats
```

## **Lexical Stack for ---@type annotations**

Use a stack of maps to track variable types per lexical block:

```lua
env.varTypeStack = { {} }  -- push on entering block, pop on exit

-- Lookup:
function get_var_type(env, name)
  for i = #env.varTypeStack, 1, -1 do
    local vt = env.varTypeStack[i][name]
    if vt then return vt end
  end
end

-- Binding:
function bind_var_type(env, name, typeinfo)
  env.varTypeStack[#env.varTypeStack][name] = typeinfo
end
```

Then:
- Push a new scope map when entering a `Block` node.
- Pop it when exiting.
- Replace all `cats.varTypes[name]` lookups with `get_var_type(env, name)`.

This keeps the global `cats.varTypes` for metadata, but uses `env.varTypeStack` for actual validation.

**Scoped `varTypes` stack integration** for `validator.lua`  

Add this to your `env` initialization:

```lua
env.varTypeStack = { {} }  -- stack of scoped varType maps
```

**Lookup and Binding Helpers**

```lua
function env:getVarType(name)
  for i = #self.varTypeStack, 1, -1 do
    local t = self.varTypeStack[i][name]
    if t then return t end
  end
end

function env:bindVarType(name, typeinfo)
  self.varTypeStack[#self.varTypeStack][name] = typeinfo
end

function env:pushScope()
  table.insert(self.varTypeStack, {})
end

function env:popScope()
  table.remove(self.varTypeStack)
end
```

**Integration Points**

In `validateBlock(node)` or wherever you traverse `Block` nodes:

```lua
env:pushScope()
-- validate children
env:popScope()
```

In `validateLocal(node)` or wherever you bind `---@type`:

```lua
if node.typeAnno then
  for i, id in ipairs(node.names) do
    env:bindVarType(id.name, node.typeAnno[i])
  end
end
```

In `validateIndex(node)`:

```lua
local baseName = getBaseId(node.base)
local baseType = env:getVarType(baseName)
-- validate against baseType
```

---

### Test plan

Add a new section to test.lua covering both capture and validation.

1) LuaCATS capture only (no validation errors)
- Classes, fields (Class.Field)
- Constants
- Enums

Example:
```lua
---@class My.Point
---@field public [1] number # w
---@field protected [2] number # x
---@field private [3] number # y
---@field [4] number # z
---@alias My.Num 9 # Nine
---@alias My.Enum
---| 5 # Five
---| 6 # Six
---| 7 # Seven
local a = --[[C:My.Num]] 3
local b = --[[E:My.Enum:Five]] 5
local c = --[[E:My.Enum:Six]] 6
local d = --[[E:My.Enum:Seven]] 7 
```

- Expect: parse OK, and block.meta.cats contains:
  - classes["My.Point"].fields = { "w"={index=1, ...}, "x"={index=2, ...}, "y"={index=3, ...}, "z"={index=4, ...} }
  - constants["My.Num"] = { value = "9", ... }
  - aliases["My.Enum"].values = { "Five"={value=5, ...}, "Six"={value=6, ...}, "Seven"={value=7, ...} }
- Modify pp.lua to output block.meta.cats so that tests can validate it.

2) Inline annotation type validation
- Good:
```lua
local n = 9 --[[C:My.Num]]
local e = 5 --[[E:My.Enum:Five]]
```
- Bad (unknown type):
```lua
local n = 3 --[[C:No.Such.Type]]
-- expect: invalid inline annotation name: No.Such.Type does not exist
```
- Bad (unequal value):
```lua
local n = 3 --[[C:My.Num]]
-- expect: invalid inline annotation value for My.Num: expected 9, got 3
```
- Bad (unknown type):
```lua
local n = 9 --[[E:No.Such.Type:Val]]
-- expect: invalid inline annotation name: No.Such.Type does not exist
```
- Bad (unknown enum):
```lua
local e = 5 --[[E:My.Enum:PINK]]
-- expect: invalid inline annotation name: PINK is not a member of My.Enum
```
- Bad (unequal enum value):
```lua
local e = 6 --[[E:My.Enum:Five]]
-- expect: invalid inline annotation value for My.Enum:Five: expected 5, got 6
```
- Bad (unknown type):
```lua
local n = 9 --[[F:No.Such.Type:Val]]
-- expect: invalid inline annotation name: No.Such.Type does not exist
```
- Bad (unknown field):
```lua
local e = 5 --[[F:My.Point:PINK]]
-- expect: invalid inline annotation name: PINK is not a member of My.Point
```
- Bad (unequal field value):
```lua
local e = 6 --[[F:My.Point:width]]
-- expect: invalid inline annotation value for My.Point:width: expected 1, got 6
```

3) Binding @type to Set and Local
- Good:
```lua
---@class My.Box
---@field [1] number # width
---@type My.Box
local b = { [ 1 --[[F:My.Box:width]] ] = 1 }
local w = b[ 1 --[[F:My.Box:width]] ]
---@class My.Type
---@field [1] My.Box # box
---@type My.Type
local t, x = { [ 1 --[[F:My.Type:box]] ] = { [ 1 --[[F:My.Box:width]] ] = 1 } }, 2
local z, y = 3, t[ 1 --[[F:My.Type:box]] ][ 1 --[[F:My.Box:width]] ]
local u = (t)[ 1 --[[F:My.Type:box]] ][ 1 --[[F:My.Box:width]] ]
local d = {b = 5} -- 'b' must be ignored
d.b = 6 -- 'b' must be ignored
```
- Bad: (mismatched type)
```lua
---@class A
---@field [1] number # B
---@class C
---@field [1] number # D
---@type A
local t = { [ 1 --[[F:A:B]] ] = 1 }
local b = t[ 1 --[[F:C:D]] ]
-- expect: mismatched inline annotation name: expected A, got C
```

4) Scope management

```lua
---@class A
---@field [1] number # B
---@class C
---@field [2] number # D
---@type A
local p = {[1 --[[F:A:B]]] = 1}
do
  ---@type C
  local p = {[2 --[[F:C:D]]] = 2}
  local w = p[ 2 --[[F:C:D]] ] -- should validate against C
end
local x = p[ 1 --[[F:A:B]] ] -- should validate against A
```

If `cats.varTypes` is flat, this will fail or misvalidate. With scoped stack, it works correctly.

5) Missing annotations
- Good:
```lua
---@class My.Box
---@field [1] number # width
---@type My.Box
local b = { [ 1 --[[F:My.Box:width]] ] = 1 }
local w = b[ 1 --[[F:My.Box:width]] ]
```
- Bad: missing inline field annotation
```lua
---@class My.Box
---@field [1] number # width
---@type My.Box
local b = {[1] = 1}
-- expect: missing annotation indexing with [1], expected [ 1 --[[F:My.Box:width]] ]
```
- Bad: missing inline field annotation
```lua
---@class My.Box
---@field [1] number # width
---@type My.Box
local b = {}
local w = b[1]
-- expect: missing annotation indexing b[1], expected b[ 1 --[[F:My.Box:width]] ]
```
- Bad: missing inline field annotation and field
```lua
---@class My.Box
---@field [1] number # width
---@type My.Box
local b = {}
local w = b[2]
-- expect: missing annotation indexing b[2], expected b[ 2 --[[F:My.Box:...]] ]
```

7) Regressions
- All existing tests must remain green.
- Inline F/E/C happy-paths already in your suite should still pass (now with type checks if you add the needed @class/@alias/@enum above them, or keep validation permissive when type not in cats if you want a phased rollout).

