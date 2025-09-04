# Decision: Custom Index Annotation System  
**Date:** 2025-08-19  
**Status:** Accepted  
**Context:** During development of the BattleTech TTS automation mod, the team needed a way to make numeric table index lookups more maintainable without sacrificing runtime performance. Direct numeric indices are preferred for speed and memory efficiency in Lua, but they can obscure meaning and make refactoring risky. This decision documents the adoption of a lightweight, self-documenting annotation convention for inline index metadata, following early prototypes and discussions around code clarity, IDE support, and testability.
**Decision:** Adopt a standardized inline annotation pattern for numeric table indices, enums, and constants that can eventually be tooled better.
```lua
value --[[TYPE:TypeFullName:FieldOrEnumName]]

--- Three examples:
---@class M.Tech.MechValidation
---@field [8] number # TotalTons: Total mech weight (Q30.2 fixed point)

---@alias M.Mech.Disk.Enum.Twist
---| 1 # Left
---| 2 # Center
---| 3 # Right

---@alias M.Mech.Disk.CritHit.EquipmentIdMask 0xFF

local totalTons = mechValidation[ 8 --[[F:M.Tech.MechValidation:TotalTons]] ]
local t = 2 --[[E:M.Mech.Disk.Enum.Twist:Center]]
local l = bit32.band(eqId, 0xFF --[[C:M.Mech.Disk.CritHit.EquipmentIdMask]])
```
Where `TYPE` is one of:  
- **F:** Field reference  
- **E:** Enum value  
- **C:** Constant value  

**Rationale:**  
- Preserves **performance characteristics** of numeric lookups (avoids hash table overhead).
- Compresses Lua scripts
- Embeds **type and semantic context** directly alongside “magic numbers,” making them self-describing.  
- Enhances **refactor safety** by tying each numeric value to a specific type/field/enum name.  
- Improves **IDE navigation** by allowing tooling or searches to resolve back to the type definitions.  
- Maintains **visual proximity** of metadata to code — no need to reference separate lookup tables for meaning.  
- Matches broader codebase practice of spec-first, type-safe design.

**Implications:**  
- **Architecture:**  
  - Consistent pattern across all numeric table lookups ensures predictable parsing if tooling or linters are added later.  
  - Allows evolution toward auto-generated index maps or schema validation without changing runtime indexing behavior.  

- **Testing:**  
  - Makes golden test trace diffs easier to read, as values carry in-place meaning.  
  - Reduces risk of misaligned indices during test fixture changes.

- **Workflow:**  
  - Requires discipline in spacing, comment placement, and full type references.  
  - New contributors must be trained on the annotation format.  
  - When refactoring types/enums, find-and-replace operations will be simplified by the unique format.  

**References:**  
- Lua performance discussion on numeric vs. string keys  
- Internal type definition examples for `M.Tech.MechValidation`, `M.Mech.Disk.Enum.Twist`, `M.Mech.Disk.CritHit.EquipmentIdMask`  
