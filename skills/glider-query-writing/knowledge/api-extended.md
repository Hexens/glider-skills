# Extended API Reference

## StateVariables

```python
# From a contract
state_vars = contract.state_variables()

# Filtering
state_vars.with_name("_balances")
state_vars.with_type("mapping(address => uint256)")
state_vars.with_properties(StateVariableProp.PUBLIC)
state_vars.with_properties(StateVariableProp.CONSTANT | StateVariableProp.IMMUTABLE)
state_vars.exec(100)

# StateVariableProp values
StateVariableProp.PUBLIC
StateVariableProp.INTERNAL
StateVariableProp.PRIVATE
StateVariableProp.CONSTANT
StateVariableProp.IMMUTABLE
```

### StateVariable Instance

```python
sv.name                  # variable name
sv.type                  # -> Type
sv.source_code()         # declaration source
sv.properties()          # -> list[str]
sv.is_public() / sv.is_internal() / sv.is_private()
sv.is_constant() / sv.is_immutable()
sv.is_accessible()       # public or has getter
sv.get_getter()          # -> Function | NoneObject
sv.contract()            # -> Contract | NoneObject
```

---

## Events

```python
# From a contract — returns an Events collection object
events = contract.events()
events.exec(100)           # -> APIList[Event] (paginated, preferred)
events.events              # -> APIList[Event] (all events, no limit)

# Event instance
event.name                 # event name
event.signature            # full signature
event.address              # contract address
event.arg_list()           # -> list[str] (argument types)
event.source_code()
```

---

## Errors

```python
errors = contract.errors()
errors.exec(100)

# Error instance
error.name
error.signature
error.args                 # -> list[dict]
error.address
error.source_code()
```

---

## Enums

```python
enums = contract.enums()
enums.exec(100)

# Enum instance
enum.name
enum.values                # -> list
enum.min / enum.max        # int bounds
enum.address
enum.source_code()
```

---

## Structs

```python
structs = contract.structs()
structs.exec(100)

# Struct instance
struct.name
struct.fields              # -> APIList[StructField]
struct.address
struct.source_code()

# StructField
field.name                 # field name
field.type                 # -> Type
```

---

## ArgumentPoints

Access function arguments as data flow points:

```python
args = func.arguments()         # -> ArgumentPoints

args.list()                     # -> APIList[ArgumentPoint]
args.with_name("recipient")     # -> APIList[ArgumentPoint]
args.with_type("address")       # -> APIList[ArgumentPoint]
args.with_memory_type("calldata")  # -> APIList[ArgumentPoint]
```

### ArgumentPoint Instance

```python
arg_point.get_variable()        # -> ArgumentVariable
arg_point.get_parent()          # -> Callable
arg_point.has_global_df()       # True when this argument is user-controlled

# Use as data flow source
arg_point.forward_df()          # where does this argument flow?
arg_point.forward_df_recursive()
```

---

## Condition (from IfInstruction)

`get_condition()` is only available when the instruction is an `IfInstruction`. Always check the type first:

```python
if isinstance(if_instr, IfInstruction):
    condition = if_instr.get_condition()
condition.is_eq()      # ==
condition.is_neq()     # !=
condition.is_geq()     # >=
condition.is_leq()     # <=
condition.is_gr()      # >
condition.is_le()      # <

# Also available on IfInstruction:
if_instr.first_true_instruction()    # -> Instruction (true branch)
if_instr.first_false_instruction()   # -> Instruction (false branch)
```

---

## Loop

```python
loops = func.loops()            # -> APIList[Loop]

for loop in loops:
    loop.header                 # -> Instruction (loop header/condition)
    loop.instructions           # -> APIList[Instruction] (body)
    loop.id                     # int identifier

    loop.is_in_loop(instr)      # is this instruction inside this loop?
    loop.is_in_loop_recursive(instr)  # including nested loops?

    for instr in loop:          # iterate loop body instructions
        ...
```

---

## Inheritance Navigation

### Check Base and Derived Contracts

```python
contract = Contracts().with_name("MyToken").exec(1)[0]

# All contracts this inherits from (full chain)
bases = contract.base_contracts()       # -> Contracts | NoneObject
if not isinstance(bases, NoneObject):
    for base in bases.exec():
        print(base.name)

# Only direct parents (not grandparents)
direct_bases = contract.direct_base_contracts()  # -> Contracts | NoneObject

# All contracts that inherit from this one
derived = contract.derived_contracts()  # -> Contracts | NoneObject
```

### Check If a Contract Inherits from a Specific Base

```python
def inherits_from(contract, base_name):
    bases = contract.base_contracts()
    if isinstance(bases, NoneObject):
        return False
    return any(b.name == base_name for b in bases.exec())

# Usage
contracts = Contracts().exec(1000)
ownable_contracts = contracts.filter(lambda c: inherits_from(c, "Ownable"))
```

### Find Overridden Functions

Functions declared in a derived contract that share a name/signature with a base contract function:

```python
contract = Contracts().with_name("MyToken").exec(1)[0]
bases = contract.base_contracts()
if isinstance(bases, NoneObject):
    return []

base_fn_names = set()
for base in bases.exec():
    for fn in base.functions().exec():
        base_fn_names.add(fn.name)

overridden = []
for fn in contract.functions().exec():
    if fn.name in base_fn_names:
        overridden.append(fn)
```

---

## Advanced: TaintEngine

> **Note:** Advanced feature. Most taint queries are better served by `forward_df()`, `backward_df()`, or their recursive variants. Use `TaintEngine` when you need custom taint sources, multiple simultaneous sources, or explicit taint state inspection.

```python
# Get the taint engine for a contract address
engine = get_taint_engine(address)           # same-function taint
engine = get_global_df_recursive(address)    # cross-function taint (slower)

# Mark custom taint sources
engine.add_taint_source(point)
engine.set_tainted_sources([point_a, point_b])  # replaces all sources, recalculates

# Query taint results
tainted = engine.get_tainted_nodes()         # -> APISet[Point] — all reachable points
sources = engine.get_tainted_sources()       # -> APIList[Point]
state = engine.get_tainted_state(point)      # -> TaintState (NOT_TAINTED / TAINTED / TAINT_SOURCE)
engine.is_tainted(point)                     # -> bool
engine.is_tainted_source(point)              # -> bool
engine.recalculate()                         # rerun after modifying sources
```

Example — taint from a specific argument and find where it flows:

```python
fn = Functions().with_name("deposit").exec(1)[0]
args = fn.arguments().list()
engine = get_taint_engine(fn.get_contract().address())
engine.set_tainted_sources(args)
tainted_instrs = [p for p in engine.get_tainted_nodes() if isinstance(p, Instruction)]
```

---

## Advanced: Point Taint and Data Flow Path Methods

> **Note:** Advanced feature not needed in most queries. Available on all `Point` subclasses (Instruction, ArgumentPoint, StatePoint, VarValue).

```python
# Get the tainted path that influences this point (cross-function)
path = point.get_tainted_path_affecting_point()         # -> PointPath (first path found)
paths = point.get_all_tainted_paths_affecting_point()   # -> APISet[PointPath] (all paths)
sources = point.get_tainted_sources_affecting_point()   # -> APISet[Point] (all taint sources)

# Trace argument data flow across call chains
# Returns (function, argument_index) pairs where df reaches this point FROM
pairs = point.df_reaches_from_functions_arguments()     # -> APIList[Tuple[Callable, int]]
# Returns (function, argument_index) pairs where df reaches TO from this point
pairs = point.df_reaching_functions_arguments()         # -> APIList[Tuple[Callable, int]]
```
