# Navigation & Data Flow

## Working with the Value Tree

Every instruction has an expression tree accessed via `get_value()`. Understanding the tree structure is key to writing precise queries.

### Instruction → Value Decomposition

```python
instr = ...  # an Instruction

# Top-level value (the entire expression)
value = instr.get_value()       # -> Call, Operator, Literal, VarValue, etc.

# For tuple assignments like (a, b) = foo()
components = instr.get_components()  # -> APIList[Value]
first = instr.get_component(0)

# Assignment destination
dest = instr.get_dest()         # -> Value (left side of =)
```

### Working with Call Values

```python
value = instr.get_value()
if isinstance(value, Call):
    print(value.name)                  # function name: "transfer"
    print(value.signature)             # "transfer(address,uint256)"
    print(value.get_contract_name())   # "IERC20"

    args = value.get_args()            # APIList of argument values
    first_arg = value.get_arg(0)       # first argument

    qualifier = value.get_call_qualifier()  # object.method() → the "object" part
    eth_value = value.get_call_value()      # msg.value sent with call

    resolved_fn = value.get_function()      # -> Function (if resolvable)
```

### Working with callee_values() and get_callee_values()

`func.callee_values()` returns `APIList[Call]` — all call expressions within a function.

`value.get_callee_values()` does the same but is called on a `Value` object (e.g., the result of `instr.get_value()`) rather than a function:

```python
for call in func.callee_values():
    if call.name == "transfer":
        recipient = call.get_arg(0)
        amount = call.get_arg(1)
```

### Checking Value Types

```python
isinstance(value, Call)          # function call
isinstance(value, Operator)      # binary/unary operator
isinstance(value, Literal)       # constant value (value.value for the string)
isinstance(value, IndexAccess)   # array[i] or mapping[key]
isinstance(value, VarValue)      # variable reference
isinstance(value, NoneObject)    # null/missing value
```

### Tracing Where a Variable Comes From

```python
if isinstance(value, VarValue):
    obj = value.get_object_of_var()
    if isinstance(obj, ArgumentVariable):
        print("comes from function argument (user input)")
    elif isinstance(obj, StateVariable):
        print("comes from storage")
    elif isinstance(obj, LocalVariable):
        print("comes from local variable")
    elif isinstance(obj, GlobalVariable):
        print("comes from msg.sender, block.timestamp, etc.")
```

### Special Case: address(this)

`address(this)` is a type conversion in Solidity, not a function call. It is not represented as a `Call` node in the value tree — `isinstance(value, Call)` returns `False` for it. Detect it via `.expression`:

```python
# Exact match
if value.expression == "this":
    ...

# When address(this) may appear nested as a sub-expression (e.g. inside an argument)
if "this" in value.expression:
    ...
```

---

## CFG Navigation Techniques

### Sequential Pattern Matching

Check for a sequence of operations (A → B → C):

```python
# Using APIList chaining (declarative)
(Instructions()
    .with_callee_name("balanceOf").exec(100)
    .next_instruction()
    .filter(lambda x: "transferFrom" in x.callee_names())
    .next_instruction()
    .filter(lambda x: "balanceOf" in x.callee_names())
)
```

Equivalent imperative approach (gives more control):

```python
for ins in balance_of_instructions:
    for second in ins.next_instruction():
        if "transferFrom" in second.callee_names():
            for third in second.next_instruction():
                if "balanceOf" in third.callee_names():
                    results.append(ins)
```

### next_instruction() vs next_instructions()

- `next_instruction()` → `APIList[Instruction]` — direct next instructions in CFG
- `next_instructions()` → `APISet[Instruction]` — all possible successors
- `next_instructions_recursive()` → `APISet[Instruction]` — all reachable from this point (**slow — only use when needed**)

### Checking What Comes Before

```python
for prev in instruction.previous_instructions():
    if (prev.is_call() or prev.is_if()) and prev.has_global_df():
        # There's an access control check before this instruction
```

---

## Data Flow Techniques

### has_global_df() — Quick User-Input Check

`has_global_df()` returns `True` if the instruction is influenced by global variables (`msg.sender`, `tx.origin`, etc.) within the current function. `has_global_df_recursive()` extends this check across function boundaries (slower):

```python
if instruction.has_global_df():
    # influenced by globals in the current function

if instruction.has_global_df_recursive():
    # influenced by globals anywhere in the call chain (cross-function, expensive)
```

### forward_df_recursive() — What Does This Value Affect?

Use `forward_df_recursive()` as the default to trace all downstream uses of a value across function boundaries:

```python
# Find everything tainted by an ecrecover result (cross-function)
for point in ecrecover_instr.forward_df_recursive():
    if isinstance(point, Instruction) and (point.is_if() or "require" in point.callee_names()):
        print("result is used in a validation check")
```

### Detecting Guard Conditions on a Return Value

When a vulnerability involves checking whether a function call's return value is validated, use `forward_df_recursive()` to trace the data flow from the call and look for downstream `require`/`assert` calls or `is_if()` instructions. This structural approach covers all guard patterns regardless of naming conventions:

```python
def has_guard(call_instr):
    for inst in call_instr.forward_df_recursive():
        if not isinstance(inst, Instruction):
            continue
        if "require" in inst.callee_names() or "assert" in inst.callee_names():
            return True
        if inst.is_if():
            return True
    return False

# Usage: keep only call instructions whose return value is NOT guarded
unguarded = Instructions().with_callee_name("get_price").exec(100).filter(
    lambda i: not has_guard(i)
)
```

### backward_df_recursive() — Where Did This Value Come From?

Use `backward_df_recursive()` as the default — it crosses function boundaries and produces complete results. Use `backward_df()` (non-recursive) only when analysis must be explicitly constrained to the current function scope:

```python
# Default: use recursive to trace across function boundaries
for point in value.backward_df_recursive():
    if isinstance(point, Instruction) and "abi.decode" in point.callee_names():
        print("value originates from abi.decode")
    if isinstance(point, ArgumentPoint):
        print("value comes from a function argument")

# Non-recursive: only when you intentionally want single-function scope
for point in value.backward_df():
    # same-function data flow only
```

---

## Level Navigation Techniques

### Filtering to Main Contracts

Every query should return results from main contracts only. Use `is_main()` at whatever level you are working:

```python
# Contract level — use .mains() in the chain (preferred)
Contracts().mains().exec(100)

# Function level — check the parent contract
if func.get_contract().is_main():
    results.append(func)

# Instruction level — walk up to the contract
if instr.get_parent().get_contract().is_main():
    results.append(instr)
```

### Going Up: Instruction → Function → Contract

```python
function = instruction.get_parent()          # -> Callable
contract = function.get_contract()           # -> Contract
```

### Going Down and Back Up

A powerful pattern: filter at one level, navigate up to parent, check siblings:

```python
# Find contracts where one function calls selfdestruct AND another calls ecrecover
(Functions()
    .with_callee_names(["selfdestruct"])
    .contracts()          # go up to contract level
    .exec(1000)
    .functions()          # back down to all functions in those contracts
    .with_callee_names(["ecrecover"])
    .exec()
)
```

### Checking All Functions in a Contract

```python
contract = function.get_contract()
all_fns = contract.functions().exec()
sibling_fn = contract.functions().with_name("_delegate").exec()
```

### Recursive Call Tree Inspection

```python
# All functions reachable from fn (entire call tree)
all_reachable = fn.callee_functions_recursive().exec()

# All functions that eventually call fn
all_callers = fn.caller_functions_recursive().exec()
```

---

## Structural Alternatives to Source Code Checks

Never use `source_code()` for detection logic — it text-matches the raw source, misses semantically equivalent code, and matches comments and strings. Use structural API instead:

```python
# Checking for a specific function call — use callee_names()
if "transfer" in instruction.callee_names():
    ...

# Checking for built-in calls (keccak256, ecrecover, abi.encode, etc.)
if "keccak256" in instruction.builtin_callee_names():
    ...

# Checking calls within a function — use callee_values()
for call in func.callee_values():
    if call.name == "transfer":
        ...

# Checking for a zero-address or other guard — use data flow + is_if()
for point in instr.forward_df_recursive():
    if isinstance(point, Instruction) and (point.is_if() or "require" in point.callee_names()):
        # There is a guard on this value
        ...

# Checking msg.sender influence — use has_global_df() or GlobalFilters
if instruction.has_global_df():
    # influenced by msg.sender, tx.origin, etc.

sender_instrs = function.instructions().with_globals(GlobalFilters.MSG_SENDER).exec()

# Checking assignment destination name or type — use get_dests()
for dest in instr.get_dests():
    if dest.name == "_balances":
        ...
    if str(dest.type) == "address":
        ...

# Checking for a state variable by name — use state_variables()
if contract.state_variables().with_name("reserve0").exec():
    ...
```
