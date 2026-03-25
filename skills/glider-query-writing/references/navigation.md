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

### Working with callee_values()

`func.callee_values()` returns `APIList[Call]` — all call expressions within a function:

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

`has_global_df()` returns `True` if the instruction is influenced by global variables (`msg.sender`, `tx.origin`, function arguments, etc.):

```python
if instruction.has_global_df():
    # The value in this instruction is controllable by the caller
```

### forward_df() — What Does This Value Affect?

```python
# Find everything tainted by an ecrecover result
for point in ecrecover_instr.forward_df():
    if point.is_if() or "require" in point.callee_names():
        print("result is used in a validation check")
```

### backward_df_recursive() — Where Did This Value Come From?

Works across function boundaries but is **much slower** than `backward_df()`. Only use when you need cross-function tracing:

```python
# Try non-recursive first
for point in value.backward_df():
    # same-function data flow — fast

# Only escalate if you need cross-function tracing
for point in value.backward_df_recursive():
    if isinstance(point, Instruction) and "abi.decode" in point.callee_names():
        print("value originates from abi.decode")
    if isinstance(point, ArgumentPoint):
        print("value comes from a function argument")
```

---

## Level Navigation Techniques

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

