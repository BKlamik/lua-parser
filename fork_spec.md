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

Missing annotation validation: If a variable is typed as a class that provides names to all index fields through comments, then bracket indexing on that variable must use the inline field annotations; otherwise, fail in the validation pass.
This is a best effort, and only catches the following cases:
- Locals with direct type annotations
- Direct assignments from known sources, including annotated class tables
- Function parameters with annotated types
- Function returns with annotated types

Validating annotations within published scripts can rely on the bundling of LuaCATS metadata with the script, due to header composition that occurs during publishing.
However, validating annotations within the development/ test environment requires a way to pass the LuaCATS metadata gathered from the parse of some files to the parse of other files.

## AST surface
New optional field: node.anno on `Number`, `String`, and `Id` nodes.

Fields: kind ("F" | "E" | "C"), FullName (Field/Enum/Const), SimpleId (Field/Enum), pos, end_pos.

Positions: `Number`, `String`, and `Id` nodes' pos/end_pos remain unchanged; anno.pos/end_pos span the annotation comment.

For LuaCATS, we’ll keep the core AST shape intact and attach a sideband “meta” bundle at the root.
No comment nodes get emitted into the normal statement list.

- Attach on the root Block node:
  - block.cats = CatsDB
- CatsDB schema:
  - cats.classes: map<string ClassName, { pos, end_pos, base?: string, validate: boolean, fields: map<string|number, { luaType : string, pos, end_pos, indexName?: string }>}>
  - cats.pendingClass: { name: string, pos, end_pos, base?: string, validate: boolean } ; populated by ---@class; used for ---@field
  - cats.aliases: map<string AliasName, { pos, end_pos, values: map<string, { value: string, comment: string, pos, end_pos }>}>
  - cats.constants: map<string ConstName, { pos, end_pos, value: string }>
  - cats.pendingVarTypes: string[] ; populated by ---@type; relocated last stored to pendingTypes on declaration
  - cats.pendingVarParams: map<string, { type: string, pos, end_pos }> ; populated by ---@param; relocate group to varTypes on declaration
  - cats.pendingVarReturn: string[] ; populated by ---@return; relocate group to pendingReturn on declaration
  - cats.varTypes: [map<string, string>] ; populated by `local` and `set` variable names on declaration; relocated to env.varTypeStack on scope push/pop
- Inline annotations (already present):
  - exp.anno = { kind = "F"|"E"|"C", type = "FullName", name = "SimpleId"|nil, pos, end_pos }
- No changes to existing node tags, only the new block.cats field.

No other AST shapes change. All existing tags and node structures remain intact.

## Sideband LuaCATS schema
Class registry: { [FullName] = { fields = { [SimpleId] = { index = literal, ... } } } }

Alias registry: { [FullName] = { values = { [SimpleId] = { value = literal, ... } } } }

Const registry: { [FullName] = { value = literal } }

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

Define the doc tokens. Keep them simple and robust; accept common LuaCATS patterns:

```lua
-- Identifiers and simple type exprs for LuaCATS
local newline  = P("\r")^-1 * P("\n")
local extra = (1-newline)^0
local EOL = (P"\r"^-1 * P"\n") + -P(1)
local FieldType = C(alpha * (P(1) - P"#" - EOL)^0)
local wsp = S(" \t")^1
local owsp = S(" \t")^0
local FieldCmt = P("#") * owsp * (C((P(1) - S(" \t") - EOL) * (P(1) - EOL)^0) + Cc(nil))
local Alias = C((P(1) - S" \t#" - EOL)^0)
local DocId = C((alpha + P"_") * (alnum + S"_.-")^0)
local DocBOL = P"---" * owsp * P"@"
local DocBOL2 = P"---" * owsp * P"|"
local Visibility = ((P"public" + P"protected" + P"private") * owsp)^-1
local Key = ((P"[" * owsp * C(digit^1) * owsp * P"]") + C((alpha + P"_") * (alnum + P"_")^0)) * wsp
```

Wire these into Comment with side-effects, using Cmt and Carg(2). We also want to support “---@field Class.Field Type” pattern to avoid building class-context in the grammar; we can split by the first dot to get Class and Field when present, else accept as a “global field” (ignored or stored if you want).

Now the actual pattern (note: we need Cp() and the end position; we can get end by consuming until newline and using Cp again, or compute end as position before newline — we’ll just use start pos, it’s usually sufficient):

Replace Comment rule with a version that first tries doc lines, and if matched, mutates CatsDB then consumes to EOL of full multi-line annotation group:

```lua
-- In G:
  -- lexer
  DocClassLine = Cmt(
    Cp() * DocBOL * P"class" * wsp * Carg(2) * DocId * ((owsp * P":" * owsp * DocId) + Cc(nil)) * C(extra) * EOL,
    function(_, i, pos, cats, cname, base, x) return DoDocClass(cats, pos, i, cname, base, x) end
  );

  DocFieldLine = Cmt(
    owsp * Cp() * DocBOL * P"field" * wsp * Carg(2) * Visibility * Key * FieldType * (FieldCmt + Cc(nil)) * EOL,
    function(_, i, pos, cats, key, keyType, fname) return DoDocField(cats, pos, i, key, keyType, fname) end
  );

  DocAliasHeaderLine = Cmt(
    Cp() * DocBOL * P"alias" * wsp * Carg(2) * DocId * owsp * EOL,
    function(_, i, pos, cats, aname, alias) return DoDocAlias(cats, pos, i, aname) end
  );

  DocAliasUnionLine = Cmt(
    owsp * Cp() * DocBOL2 * owsp * Carg(2) * C((alnum + S"_.")^1) * (owsp * P"#" * owsp * DocId)^-1 * extra * EOL,
    function(_, i, pos, cats, value, ename) return DoDocAliasUnion(cats, pos, i, value, ename) end
  );

  DocAliasLine = Cmt(
    Cp() * DocBOL * P"alias" * wsp * Carg(2) * DocId * (wsp * Alias)^-1 * extra * EOL,
    function(_, i, pos, cats, aname, alias) return DoDocAlias(cats, pos, i, aname, alias) end
  );

  DocTypeLine = Cmt(
    Cp() * DocBOL * P"type" * wsp * Carg(2) * Ct(TypeExp * (P"," * owsp * TypeExp)^0) * extra * EOL,
    function(_, i, pos, cats, types) return DoDocType(cats, pos, i, types) end
  );

  DocParamLine = Cmt(
    Cp() * DocBOL * P"param" * wsp * Carg(2) * TypeExp * wsp * TypeExp * extra * EOL,
    function(_, i, pos, cats, pname, ptype) return DoDocParam(cats, pos, i, pname, ptype) end
  );

  DocReturnLine = Cmt(
    Cp() * DocBOL * P"return" * wsp * Carg(2) * Ct(TypeExp * (P"," * owsp * TypeExp)^0) * extra * EOL,
    function(_, i, pos, cats, rtypes) return DoDocReturn(cats, pos, i, rtypes) end
  );

  Skip     = (V"Space" + V"DocComment" + V"Comment")^0;
  Space    = space^1;

  DocComment = (V"DocClassLine" * V"DocFieldLine"^0) + (V"DocAliasHeaderLine" * V"DocAliasUnionLine"^0) + V"DocAliasLine" + V"DocTypeLine" + V"DocParamLine" + V"DocReturnLine";
```

Because Carg access is not available directly within plain function bodies without wrapping, implement DocComment as a sum of patterns each with its own Cmt and closure to the cats table using Carg and Cmt’s extra captures. The standard way:

Define helper constructors above grammar:
```lua
local function DoDocClass(cats, pos, endpos, fullName, base, x)
  local validate = true or (x and x:match("%f[%w]VALIDATE%f[%W]"))
  cats.classes[fullName] = { pos = pos, end_pos = endpos, base = base, validate = validate, fields = {} }
  cats.currentClass = fullName
  return true
end

local function DoDocField(cats, pos, endpos, key, keyType, fname)
  if not cats.currentClass then return true end
  local class = cats.classes[cats.currentClass]
  if not class then return true end

  local indexKey = tonumber(key)
  keyType = keyType:gsub("%s+$", "")
  if indexKey then
    local fname = cmt and cmt:match("[a-zA-Z_][a-zA-Z0-9_%.]*")
    if fname and ((#fname > 1 and fname:sub(-1) == ".") or fname == "") then fname = nil end
    local field = { luaType = keyType, pos = pos, end_pos = endpos }
    if fname then field.indexName = fname end
    class.fields[indexKey] = field
    class.validate = class.validate and fname ~= nil
  else
    class.fields[key] = { luaType = keyType, pos = pos, end_pos = endpos }
  end
  return true
end

local function DoDocAlias(cats, pos, endpos, aname, alias)
  if alias then
    cats.constants[aname] = { value = alias:gsub("%s+$", ""), pos = pos, end_pos = endpos }
  else
    cats.aliases[aname] = { values = {}, pos = pos, end_pos = endpos }
    cats.currentAlias = aname
  end
  return true
end

local function DoDocAliasUnion(cats, pos, endpos, value, ename)
  if ename then
    if not cats.currentAlias then return true end
    local alias = cats.aliases[cats.currentAlias]
    alias.values[ename] = { value = value, pos = pos, end_pos = endpos }
  end
  return true
end

local function DoDocType(cats, pos, endpos, types)
  cats.pendingVarTypes = types
  return true
end

local function DoDocParam(cats, pos, endpos, pname, ptype)
  cats.pendingVarParams[pname] = { type = ptype, pos = pos, end_pos = endpos }
  return true
end

local function DoDocReturn(cats, pos, endpos, rtypes)
  cats.pendingVarReturn = rtypes
  return true
end
```

Parser.parse integration:
```lua
function parser.parse (subject, filename, additionalCATS)
  local errorinfo = { subject = subject, filename = filename }
  local cats = {
    classes = additionalCATS and additionalCATS.classes or {},
    aliases = additionalCATS and additionalCATS.aliases or {},
    constants = additionalCATS and additionalCATS.constants or {},
    pendingVarParams = {},
    pendingTypes = {}
  }
  lpeg.setmaxstack(1000)
  local ast, label, errorpos = lpeg.match(G, subject, nil, errorinfo, cats) -- errorinfo as 4th arg is Carg(1), cats as 5th arg is Carg(2)
  cats.currentAlias = nil
  cats.currentClass = nil
  cats.pendingVarTypes = nil
  cats.pendingVarParams = nil
  cats.pendingVarReturn = nil
  cats.pendingTypes = nil
  ...
  ast.cats = cats
  local ok_or_ast, err = validate(ast, errorinfo)  -- unchanged call
  if type(ok_or_ast) == "table" then
    ok_or_ast = ok_or_ast or {}
    ok_or_ast.cats = cats
  end
  return ok_or_ast, err
end
```

Notes:
- We’re using two extra args to the matcher: errorinfo (already in your code) and the new cats table. lpeglabel supports passing multiple Carg(...) slots; we consumed with Carg(2).

---

### Validation pass design

We extend validator to leverage cats to validate inline annotations and enforce typed-index rules. Keep original control-flow validations intact.

Core responsibilities:

1) Validate that inline annotations reference known types and values
- For node.anno.kind == "F":
  - anno.type must resolve to a known type:
    - cats.classes[node.anno.type] must exist
  - anno.name must resolve to a known field in the class:
    - cats.classes[node.anno.type].fields[node.anno.name] must exist
  - ensure literal equals fields[node.anno.name].index
    - node[1] must be equal to cats.classes[node.anno.type].fields[node.anno.name].index
- For node.anno.kind == "E":
  - anno.type must resolve to a known type:
    - cats.aliases[node.anno.type] must exist
  - anno.name must resolve to a known value in the alias:
    - cats.aliases[node.anno.type].values[node.anno.name] must exist
  - ensure literal equals values[node.anno.name].value
    - node[1] must be equal to cats.aliases[node.anno.type].values[node.anno.name].value
- For node.anno.kind == "C":
  - anno.type must resolve to a known type:
    - cats.constants[node.anno.type] must exist
  - ensure literal equals value
    - node[1] must be equal to cats.constants[node.anno.type].value

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

- Load cats: local cats = ast.cats
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
local function validate_inline_anno(env, node)
  local a = node.anno
  if not a then return true end

  if a.kind == "F" then
    local class = env.cats.classes[a.type]
    if not class then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        "invalid inline annotation 'F', class type name '" .. a.type .. "' does not exist"
      )
    end

    local fieldKey = class.namedIndexFields[a.name]
    if not fieldKey then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        "invalid inline annotation 'F', '" .. a.type .. "' class index field name '" .. a.name .. "' does not exist"
      )
    end

    if tostring(fieldKey) ~= tostring(node[1]) then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        ("invalid inline annotation 'F', '" .. a.type .. "' class index field name '" .. a.name .. "' value '" ..
          tostring(fieldKey) .. "' does not match the inline annotated value of '" .. tostring(node[1]) .. "'"
        )
      )
    end
  elseif a.kind == "E" then
    local alias = env.cats.aliases[a.type]
    if not alias then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        "invalid inline annotation 'E', alias type name '" .. a.type .. "' does not exist"
      )
    end

    local value = alias.values[a.name]
    if not value then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        "invalid inline annotation 'E', '" .. a.type .. "' alias index value name '" .. a.name .. "' does not exist"
      )
    end

    if tostring(value.value) ~= tostring(node[1]) then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        ("invalid inline annotation 'E', '" .. a.type .. "' alias index value name '" .. a.name .. "' value '" ..
          tostring(value.value) .. "' does not match the inline annotated value of '" .. tostring(node[1]) .. "'"
        )
      )
    end
  elseif a.kind == "C" then
    local const = env.cats.constants[a.type]
    if not const then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        "invalid inline annotation 'C', alias type name '" .. a.type .. "' does not exist"
      )
    end

    if tostring(const.value) ~= tostring(node[1]) and tostring(const.value) ~= tostring('"' .. node[1] .. '"') then
      return nil, syntaxerror(
        env.errorinfo,
        a.pos,
        ("invalid inline annotation 'C', '" .. a.type .. "' alias type name value '" .. tostring(const.value) ..
          "' does not match the inline annotated value of '" .. tostring(node[1]) .. "'"
        )
      )
    end
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
  local ok, msg = validate_inline_anno(env, var); if not ok then return ok, msg end

  if var.tag == "Index" then
    local base = var[1]; local key = var[2]
    -- Validate inline anno on key
    local ok2, msg2 = validate_inline_anno(env, key); if not ok2 then return ok2, msg2 end

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
local ok, msg = validate_inline_anno(env, exp); if not ok then return ok, msg end
```

Initialize env.cats at the start of validate (traverse), and cache index lookup:
```lua
  local env = {
    errorinfo = errorinfo,
    ["function"] = {},
    cats = ast.cats or { classes = {}, aliases = {}, constants = {} }
  }
  local function processClass(class, namedIndexFields)
    local base = class.base
    local classBase = base and env.cats.classes[base]
    if classBase then
      processClass(classBase, namedIndexFields)
    end
    for k, f in pairs(class.fields) do
      if f.indexName then
        namedIndexFields[f.indexName] = k
      end
    end
  end
  for _, c in pairs(env.cats.classes) do
    c.namedIndexFields = {}
    processClass(c, c.namedIndexFields)
  end
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

- Expect: parse OK, and block.cats contains:
  - classes["My.Point"].fields = { "w"={index=1, ...}, "x"={index=2, ...}, "y"={index=3, ...}, "z"={index=4, ...} }
  - constants["My.Num"] = { value = "9", ... }
  - aliases["My.Enum"].values = { "Five"={value=5, ...}, "Six"={value=6, ...}, "Seven"={value=7, ...} }
- Modify pp.lua to output block.cats when non-empty so that tests can validate it.

2) Inline annotation type validation
- Bad (unknown type):
```lua
local n = 3 --[[C:No.Such.Type]]
```
test.lua:1:13: syntax error, invalid inline annotation 'C', alias type name 'No.Such.Type' does not exist
- Bad (unequal value):
```lua
---@alias My.Num 9
local n = 3 --[[C:My.Num]]
```
test.lua:2:13: syntax error, invalid inline annotation 'C', 'My.Num' alias type name value '9' does not match the inline annotated value of '3'
- Bad (unknown type):
```lua
local e = 5 --[[E:No.Such.Enum:Val]]
```
test.lua:1:13: syntax error, invalid inline annotation 'E', alias type name 'No.Such.Enum' does not exist
- Bad (unknown enum):
```lua
---@alias My.Enum
---| 1 # One

local x = 2 --[[E:My.Enum:Two]]
```
test.lua:4:13: syntax error, invalid inline annotation 'E', 'My.Enum' alias index value name 'Two' does not exist
- Bad (unequal enum value):
```lua
---@alias My.Enum
---| 1 # One

local x = 2 --[[E:My.Enum:One]]
```
test.lua:4:13: syntax error, invalid inline annotation 'E', 'My.Enum' alias index value name 'One' value '1' does not match the inline annotated value of '2'
- Bad (unknown type):
```lua
local f = 1 --[[F:No.Such.Class:field]]
```
test.lua:1:13: syntax error, invalid inline annotation 'F', class type name 'No.Such.Class' does not exist
- Bad (unknown field):
```lua
---@class My.Class
---@field [1] number # id

local t = {}
local x = t[ 2 --[[F:My.Class:invalid]] ] 
```
test.lua:5:16: syntax error, invalid inline annotation 'F', 'My.Class' class index field name 'invalid' does not exist
- Bad (unequal field value):
```lua
---@class My.Class
---@field [1] number # id

local t = {}
local x = t[ 2 --[[F:My.Class:id]] ] 
```
test.lua:5:16: syntax error, invalid inline annotation 'F', 'My.Class' class index field name 'id' value '1' does not match the inline annotated value of '2'

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

4) Missing annotations
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

5) Scope management

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
  local w = p[ 2 --[[F:C:D]] ] -- must validate against C
end
local x = p[ 1 --[[F:A:B]] ] -- must validate against A
```

If `cats.varTypes` is flat, this will fail or misvalidate. With scoped stack, it works correctly.

6) Regressions
- All existing tests must remain green.
- Inline F/E/C happy-paths already in your suite must still pass (now with type checks if you add the needed @class/@alias/@enum above them, or keep validation permissive when type not in cats if you want a phased rollout).

