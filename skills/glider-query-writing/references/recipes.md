# Common Recipes & Snippets

## Helper: get_components_recursive()

Extracts all sub-components (calls, variables, operators, etc.) from an instruction or value. Copy into your query when you need to inspect instruction internals.

```python
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

---

## Deduplicating Results

When queries return duplicates (e.g., same contract from multiple paths):

```python
seen = {}
for item in results:
    addr = item.get_contract().address() if hasattr(item, 'get_contract') else item.address()
    if addr not in seen:
        seen[addr] = item
return list(seen.values())
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

```python
# Contracts that have BOTH transfer AND approve (use with_all_function_names)
Contracts().with_all_function_names(["transfer", "approve"]).exec(100)

# Or manually:
set_a = set(c.address() for c in query_a_contracts)
results = [c for c in query_b_contracts if c.address() in set_a]
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

```python
def msg_sender_aliases():
    return ["msg.sender", "msgSender", "_msgSender", "_msgSenderERC1155", "caller"]

def validates_msg_sender(function):
    for instr in function.instructions_recursive():
        if revert_condition(instr):
            for alias in msg_sender_aliases():
                if alias in instr.source_code():
                    return True
    return False
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
    if instr.is_storage_write() and "_balances" in instr.source_code():
        results.append(instr)
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

### Multi-Pass: Contracts → Functions → Instructions → Back Up

```python
# Find ERC20 contracts, get their transfer functions, check for assembly
erc20s = Contracts().with_all_function_signatures([
    "transfer(address,uint256)",
    "transferFrom(address,address,uint256)"
]).non_interface_contracts().exec(1000)

for contract in erc20s:
    transfer_fns = contract.functions().with_one_of_the_names(["transfer", "_transfer"]).exec()
    for fn in transfer_fns:
        if len(fn.start_asm_instructions().exec()) > 0:
            results.append(contract)
            break
```
