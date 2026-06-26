# Common Recipes & Snippets

## Helper: get_components_recursive()

Extracts all sub-components (calls, variables, operators, etc.) from an instruction or value. Copy into your query when you need to inspect instruction internals.

```python
def get_components_recursive(component):
    components = []
    try:
        # Some Glider types don't support isinstance() — use str() to identify them
        if "IndexAccess" in str(component):
            components.append(component.get_sequence())
            components.append(component.get_index())
        if isinstance(component, Call):
            components.extend(component.get_args())
            call_qualifier = component.get_call_qualifier()
            if "IndexAccess" in str(call_qualifier):
                components.append(call_qualifier)
                components.append(call_qualifier.get_sequence())
                components.append(call_qualifier.get_index())
        else:
            components = component.get_components()
    except Exception:
        None
    results = []
    for comp in components:
        results.append(comp)
        results.extend(get_components_recursive(comp))
    return results
```

---


## Combining Multiple Sub-Queries

### Union: Merge results from different queries

```python
query_a = Instructions().with_callee_name("latestRoundData").exec(100)
query_b = Instructions().with_callee_signature("getRoundData(uint80)").exec(100)
combined = query_a + query_b  # APIList supports +
return combined
```

### Intersection: Results matching multiple criteria

Start at the `Functions()` level and navigate to contracts — this is faster than `Contracts().with_all_function_names()`:

```python
# Contracts that define transfer and approve — Functions-first
Functions()
.with_one_of_the_names(["transfer", "approve"])
.exec(500)
.contracts()
.exec()

# Functions that call ANY of these methods
Functions().with_one_of_callee_names(["latestRoundData", "getRoundData"]).exec(100)
```

### Subtraction: Exclude results

```python
all_functions = Functions().with_name("transfer").exec(100)
excluded = Functions().with_name("transfer").with_modifier_name("onlyOwner").exec(100)
result = all_functions - excluded  # set difference on Functions
```

---

## Common Query Snippets

### Check for require/assert in Function

```python
def has_guard_check(function):
    return any(function.instructions().with_one_of_callee_names(["require", "assert"]).exec())
```

### Check if Instruction Could Revert

```python
def revert_condition(instruction):
    callee_names = instruction.callee_names()
    if "require" in callee_names or "assert" in callee_names:
        return True
    if not instruction.is_if():
        return False
    return any("revert" in x for x in instruction.first_true_instruction().callee_names())
```

### Filter Out Interface Functions (Keep Only Implementations)

```python
def is_implementation(function):
    return any(function.instructions().exec())

functions.filter(is_implementation)
```

### Check if Function Sends ETH

```python
def sends_eth(function):
    for instr in function.instructions().low_level_external_calls().exec():
        if instr.get_value():
            for call in instr.get_value().get_callee_values():
                if call.name == "call" and len(call.get_call_value()) > 0:
                    return True
    return False
```

### Detect a Behavioral Pattern by What a Function Calls

Identify what protocol or interface a function interacts with by inspecting the calls it makes, not by its name. `callee_values()` returns all `Call` objects within a function; chaining `.name` gives the flat list of called names:

```python
# Does this function read Uniswap V3 price state?
def reads_spot_price(func):
    return "slot0" in func.callee_values().name

# Does this function interact with a lending protocol?
def calls_borrow(func):
    return "borrow" in func.callee_values().name

# As a DB-level entry point — use with_callee_names() when starting a chain
price_readers = Functions().with_callee_names(["slot0"]).exec(100)
```

This pattern works regardless of what the function, contract, or variable is named.

### Find All Callers of a Function

```python
target_fns = Functions().with_name("deposit").exec(5)
callers = target_fns.caller_functions().exec()
```

### Detect Arithmetic Operations

```python
def does_arithmetic(instruction):
    ops = ["-", "+", "/", "*", "**", "%", "++", "--", "+=", "-=", "*=", "/=", "%="]
    for comp in get_components_recursive(instruction):
        if "Operator" in str(comp) and comp.expression in ops:
            return True
    return False
```

### Check for msg.sender Validation

This is extremely useful when the query requires access control list checking to see if msg.sender is validated in any manner (such as admin being a privileged role, etc.).

```python
# Note: msg.sender passed into a Call or used as IndexAccess and the return value is equated against are treated as a msg.sender validation.
def validates_msg_sender(function):
    for instruction in function.instructions_recursive():
        if revert_condition(instruction) and potential_msg_sender_call(instruction):
            return True 

    return False 
    

# Checks if revert is called
def revert_condition(instruction):
    builtin_callee_names = instruction.callee_names()
    if 'require' in builtin_callee_names or 'assert' in builtin_callee_names:
        return True

    if not instruction.is_if():
        return False
        
    return any('revert' in x for x in instruction.first_true_instruction().callee_names())


# Checks if an instruction calls msg.sender in any call
def potential_msg_sender_call(instruction):
    components = get_components_recursive(instruction)

    for component in components:
        # Ignore Calls and IndexAccesses since they produce a large number of FPs.
        if isinstance(component, Call) or "IndexAccess" in str(component):
            continue

        # There are cases where msg.sender is passed into a check that isn't validating the msg.sender address. For example balance >= balances[msg.sender]. This skips those cases.
        if isinstance(component, ValueExpression) and not contains_equality_op(component):
            continue

        for msg_sender_call in msg_sender_calls():
            # source_code() is used here intentionally — structural API cannot reliably
            # detect all forms of msg.sender (e.g. _msgSender(), assembly caller).
            # This is an accepted exception to the no-source_code()-for-detection rule.
            if msg_sender_call in component.source_code():
                return True

    return False


# Iterate through a component's operations and check for equality checks. 
def contains_equality_op(component):
    ops = component.get_components().filter(lambda component : "Operator" in str(component)).get_operator()

    for operator in ops:
        if "OperatorType.NOT_EQUAL" in str(operator) or "OperatorType.EQUAL" in str(operator):
            return True

    return False

# Returns a list of common ways to retrieve msg.sender
def msg_sender_calls():
    return [
        "msg.sender",
        "msgSender",
        "_msgSender",
        "_msgSenderERC1155",
        "caller" # Assembly msg.sender call
    ]


def get_components_recursive(component):
    components = []
    try:
        if "IndexAccess" in str(component):
            components.append(component.get_sequence())
            components.append(component.get_index())
        if isinstance(component, Call):
            components.extend(component.get_args())
            call_qualifier = component.get_call_qualifier()
            if "IndexAccess" in str(call_qualifier):
                components.append(call_qualifier)
                components.append(call_qualifier.get_sequence())
                components.append(call_qualifier.get_index())
        else:
            components = component.get_components()
    except Exception:
        None
    results = []
    for comp in components:
        results.append(comp)
        results.extend(get_components_recursive(comp))
    return results
```

### Check if Function Has Argument of Specific Type

```python
Functions().exec(10).filter(
    lambda f: "address" in f.arguments().list().get_variable().type.name
)
```

### Get State Variables of a Contract

```python
contract = Contracts().with_name("UniswapV3Pool").exec(1)[0]
state_vars = contract.state_variables().exec()
print(state_vars.name)  # prints all state variable names
```

### Find Storage Writes to a Specific Variable

```python
for instr in function.instructions().exec():
    if not instr.is_storage_write():
        continue
    for dest in instr.get_dests():
        if dest.name == "_balances":
            results.append(instr)
            break
```

### Trace a Value Back to abi.decode

```python
for point in value.backward_df_recursive():
    if isinstance(point, Instruction) and "abi.decode" in point.callee_names():
        print("value originates from decoded bytes")
```

### Inspect Low-Level Call Components

```python
low_level_calls = Instructions().low_level_external_calls().exec(100)
for instr in low_level_calls:
    for comp in get_components_recursive(instr):
        if isinstance(comp, Call) and comp.name == "call":
            target = comp.get_call_qualifier()     # who is being called
            calldata = comp.get_arg(0)             # what data is sent
            eth_sent = comp.get_call_value()       # how much ETH
```

### Multi-Pass: Functions → Contracts → Functions → Instructions

```python
# Find ERC20 contracts, get their transfer functions, check for assembly
erc20s = (
    Functions()
    .with_signatures([
        "transfer(address,uint256)",
        "transferFrom(address,address,uint256)"
    ])
    .exec(1000)
    .contracts()
    .non_interface_contracts()
    .exec()
)

for contract in erc20s:
    transfer_fns = contract.functions().with_one_of_the_names(["transfer", "_transfer"]).exec()
    for fn in transfer_fns:
        if len(fn.instructions().start_asm_instructions().exec()) > 0:
            results.append(contract)
            break
```

### Finding Contracts by Function Presence

Start with `Functions()` to match the defining functions, then navigate to contracts. Use `.caller_functions()` when the goal is to find contracts actively *using* a library (not just defining it):

```python
# Contracts that define a function set
contracts = (
    Functions()
    .with_one_of_the_names(["malloc", "free", "resize"])
    .exec(100)
    .contracts()
    .exec()
)

# Contracts actively using a library — caller_functions() validates the call exists
active_users = (
    Functions()
    .with_one_of_the_names(["malloc", "free", "resize"])
    .exec(100)
    .caller_functions()
    .exec()
    .contracts()
    .exec()
)
```

---

## Query Patterns

### Pattern 1: Pure Declarative Chain

```python
def query():
    """@title: ... @description: ... @tags: ..."""
    return (
        Functions()
        .with_properties(
            (FunctionFilters.IS_PUBLIC | FunctionFilters.IS_EXTERNAL)
            & ~FunctionFilters.HAS_MODIFIERS
        )
        .instructions()
        .with_callee_name("selfdestruct")
        .exec(100)
    )
```

### Pattern 2: Declarative Chain with APIList Chaining

No loops needed — leverage APIList auto-chaining:

```python
def query():
    """@title: ERC777 reentrancy @description: ... @tags: ..."""
    return (
        Functions()
        .with_properties(FunctionFilters.IS_PUBLIC | FunctionFilters.IS_EXTERNAL)
        .without_modifier_names(["nonReentrant", "lock", "onlyOwner"])
        .with_arg_type("address")
        .instructions()
        .with_callee_name("balanceOf").exec(3000)
        .next_instruction()
        .filter(lambda x: "transferFrom" in x.callee_names() or "safeTransferFrom" in x.callee_names())
        .next_instruction()
        .filter(lambda x: "balanceOf" in x.callee_names())
    )
```

### Pattern 3: Level Navigation

Navigate up to contracts, then back down:

```python
def query():
    """@title: Destroyable signature contracts @description: ... @tags: ..."""
    return (
        Functions()
        .with_callee_names(["selfdestruct"])
        .contracts()
        .exec(1000)
        .functions()
        .with_callee_names(["ecrecover"])
        .exec()
    )
```

### Pattern 4: Iterate + Inspect

For complex logic that can't be expressed declaratively:

```python
def query():
    """@title: ... @description: ... @tags: ..."""
    candidates = Contracts().with_function_name("initialize").exec(100)
    results = []
    for contract in candidates:
        constructor = contract.constructor()
        if isinstance(constructor, NoneObject):
            results.append(contract)
            continue
        if len(constructor.modifiers().with_name("initializer").exec()) > 0:
            continue
        results.append(contract)
    return results
```

### Pattern 5: Data Flow Tracing with forward_df_recursive / backward_df_recursive

Trace where values originate or flow — works across function boundaries:

```python
# Find ecrecover calls whose return value never reaches a guard condition
ecrecovers = Instructions().with_callee_name("ecrecover").exec(100)
return ecrecovers.filter(lambda instr:
    not instr.forward_df_recursive().filter(lambda df:
        isinstance(df, Instruction) and (df.is_if() or "require" in df.callee_names() or "assert" in df.callee_names())
    )
)
```

### Pattern 6: Component Decomposition

Extract values from complex instructions using `get_components_recursive()`:

```python
for component in get_components_recursive(instruction):
    if isinstance(component, Call) and component.name == "call":
        target = component.get_call_qualifier()
        calldata = component.get_arg(0)
        for point in target.backward_df_recursive():
            if isinstance(point, Instruction) and "abi.decode" in point.callee_names():
                results.append(instruction)
```

### Pattern 7: Data-Flow Validation Check

When a vulnerability involves a call whose return value should be validated, use `forward_df_recursive()` to find downstream guard conditions:

```python
def query():
    """@title: Unvalidated oracle price @description: ... @tags: ..."""
    price_calls = Instructions().with_callee_name("get_price").exec(100)
    results = []
    for instr in price_calls:
        if not instr.get_parent().get_contract().is_main():
            continue
        df_points = instr.forward_df_recursive()
        has_validation = df_points.filter(lambda p:
            isinstance(p, Instruction) and (
                "require" in p.callee_names()
                or "assert" in p.callee_names()
                or p.is_if()
            )
        )
        if not has_validation:
            results.append(instr)
    return results
```

For control-flow guards not tied to the return value, use `previous_instructions_recursive()` to walk backwards from the call site and look for the expected guard.
