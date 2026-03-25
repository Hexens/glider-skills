# Extended API Reference

## StateVariables

```python
# From a contract
state_vars = contract.state_variables()

# Filtering
state_vars.with_name("_balances")
state_vars.with_type("mapping(address => uint256)")
state_vars.with_all_properties([StateVariableProp.PUBLIC])
state_vars.with_one_property([StateVariableProp.CONSTANT, StateVariableProp.IMMUTABLE])
state_vars.exec(100)

# StateVariableProp values
StateVariableProp.PUBLIC
StateVariableProp.INTERNAL
StateVariableProp.PRIVATE
StateVariableProp.CONSTANT
StateVariableProp.IMMUTABLE
```

### StateVariable Instance

`sv.name`, `sv.type`, `sv.source_code()`, `sv.properties()`, `sv.is_public()`, `sv.is_internal()`, `sv.is_private()`, `sv.is_constant()`, `sv.is_immutable()`, `sv.is_accessible()`, `sv.get_getter()` → `Function|NoneObject`, `sv.contract()` → `Contract|NoneObject`

---

## Events

```python
events = contract.events().exec(100)
```

**Event instance:** `event.name`, `event.signature`, `event.address`, `event.arg_list()` → `list[str]`, `event.source_code()`

---

## Errors

```python
errors = contract.errors().exec(100)
```

**Error instance:** `error.name`, `error.signature`, `error.args` → `list[dict]`, `error.address`, `error.source_code()`

---

## Enums

```python
enums = contract.enums().exec(100)
```

**Enum instance:** `enum.name`, `enum.values` → `list`, `enum.min`, `enum.max`, `enum.address`, `enum.source_code()`

---

## Structs

```python
structs = contract.structs().exec(100)
```

**Struct instance:** `struct.name`, `struct.fields` → `APIList[StructField]`, `struct.address`, `struct.source_code()`
**StructField:** `field.name`, `field.type` → `Type`

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
arg_point.has_global_df()       # always True (arguments are user-controlled)

# Use as data flow source
arg_point.forward_df()          # where does this argument flow?
arg_point.forward_df_recursive()
```

---

## Condition (from IfInstruction)

```python
if_instr = ...  # an IfInstruction

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
