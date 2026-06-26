# Error Handling & Anti-Patterns

## Error Handling and Defensive Coding

### Value.type Is Not a Plain String

`Value.type` returns a type object, not a Python string. Wrap it with `str()` before any string comparison or concatenation:

```python
# WRONG — comparison silently fails
if dest.type == "address":
    ...

# CORRECT
if str(dest.type) == "address":
    ...
```

---

### NoneObject — Not Python None

Many Glider methods return `NoneObject` instead of Python `None` when a result is absent. Never use `is None` or `== None` — always use `isinstance`:

```python
constructor = contract.constructor()

# WRONG — will not catch NoneObject
if constructor is None:
    ...

# CORRECT
if isinstance(constructor, NoneObject):
    ...
```

### Methods That Can Return NoneObject

Be defensive when calling these — they all can return `NoneObject`:

```python
contract.constructor()          # no constructor defined
func.get_contract()             # orphaned function
instr.get_parent()              # orphaned instruction
instr.get_value()               # instruction has no value
instr.get_dest()                # no assignment destination
instr.get_component(i)          # index out of range
call.get_function()             # unresolvable call target
call.get_call_qualifier()       # no qualifier (free function call)
value.parent_value              # top-level value
var_value.get_object_of_var()   # can't resolve variable
```

### Safe Chaining Pattern

```python
value = instr.get_value()
if isinstance(value, NoneObject):
    continue

# Now safe to use
if isinstance(value, Call):
    fn = value.get_function()
    if not isinstance(fn, NoneObject):
        print(fn.name)
```

### Safe Component Iteration

The `get_components_recursive` helper already uses try/except, but when writing custom logic always guard against NoneObject in component lists:

```python
for comp in instr.get_components():
    if isinstance(comp, NoneObject):
        continue
    # process comp
```

### Safe Filter Predicates

Filter predicates receive each element in turn. Guard against absent values with `isinstance(NoneObject)` checks before accessing further attributes:

```python
def safe_check(instr):
    value = instr.get_value()
    if isinstance(value, NoneObject):
        return False
    return isinstance(value, Call) and value.name == "transfer"

instructions.filter(safe_check)
```

---

## Anti-Patterns and Common Mistakes

### 1. Assuming get_value() Returns a Call

`get_value()` can return any `Value` subtype — `Call`, `Operator`, `Literal`, `IndexAccess`, `VarValue`, `ValueExpression`, or `NoneObject`. Always check:

```python
# WRONG — crashes if value is not a Call
args = instr.get_value().get_args()

# CORRECT
value = instr.get_value()
if isinstance(value, Call):
    args = value.get_args()

# ValueExpression wraps multiple inner values — use get_components() to extract them
if isinstance(value, ValueExpression):
    for comp in value.get_components():
        if isinstance(comp, Call):
            args = comp.get_args()
```

### 2. Confusing callee_functions() vs callee_values()

- `callee_functions()` returns **resolved Function objects** — actual function definitions
- `callee_values()` returns **Call value objects** — the call expressions with arguments, qualifiers, etc.

```python
# To get the function objects that are called:
fn.callee_functions().exec()

# To inspect how functions are called (arguments, qualifiers):
fn.callee_values()  # -> APIList[Call], no .exec() needed
```

### 3. Forgetting callee_names() Is a Flat List

`callee_names()` returns all function names called in an instruction as a flat list. For `require(foo())`, it returns `["require", "foo"]`:

```python
# CORRECT — check membership
if "require" in instr.callee_names():
    ...

# WRONG — don't compare directly
if instr.callee_names() == "require":  # always False
    ...
```

### 4. Using Python `in` on APIList Incorrectly

`in` on `APIList` checks object identity/equality, not attribute matching. Use `.filter()` for attribute checks:

```python
# WRONG — checks if the string "transfer" is an element of the list
if "transfer" in functions.exec():
    ...

# CORRECT — filter by name attribute
transfer_fns = functions.exec().filter(lambda f: f.name == "transfer")
```

### 5. Not Returning a List

The `query()` function must always return a list. Common mistakes:

```python
# WRONG — returning a single object
return contract

# WRONG — returning None implicitly
if not results:
    return  # implicit None

# CORRECT
return [contract]
return []
```

### 6. Using Deprecated Property Filters

```python
# WRONG — deprecated
Functions().with_one_property([FunctionFilters.IS_PUBLIC, FunctionFilters.IS_EXTERNAL])
Functions().without_properties([FunctionFilters.HAS_MODIFIERS])
Functions().with_all_properties([FunctionFilters.IS_PUBLIC])

# CORRECT — use with_properties() with boolean expressions
Functions().with_properties(FunctionFilters.IS_PUBLIC | FunctionFilters.IS_EXTERNAL)
Functions().with_properties(~FunctionFilters.HAS_MODIFIERS)
```

**Full `FunctionFilters` enum:**
- Visibility: `IS_PUBLIC`, `IS_EXTERNAL`, `IS_INTERNAL`, `IS_PRIVATE`
- Type: `IS_PAYABLE`, `IS_PURE`, `IS_VIEW`, `IS_GLOBAL`, `IS_CONSTRUCTOR`
- Structure: `HAS_CALLEES`, `HAS_ARGS`, `HAS_MODIFIERS`, `HAS_ERRORS`
- State: `HAS_STATE_VARIABLES_READ`, `HAS_STATE_VARIABLES_WRITTEN`, `HAS_GLOBAL_VARIABLES_READ`

### 7. Ignoring exec() on Declarative Chains

Declarative filter methods build a query but don't execute it. Forgetting `.exec()` means you get a query builder object, not results:

```python
# WRONG — fns is a Functions query builder, not a list
fns = Functions().with_name("transfer")
for fn in fns:  # may not iterate as expected
    ...

# CORRECT
fns = Functions().with_name("transfer").exec(100)
for fn in fns:
    ...
```

### 8. Expensive Operations Inside filter()

`filter()` runs the predicate on every element. Avoid expensive operations inside:

```python
# BAD — backward_df_recursive() on every instruction
instructions.filter(lambda i: any(i.backward_df_recursive()))

# BETTER — pre-filter with cheap checks first
instructions.filter(lambda i: i.has_global_df()).filter(expensive_check)
```

---

## Common Pitfalls

1. **Forgetting `.exec()`** — filters are lazy; nothing runs until `.exec(limit, offset)`
2. **`.exec()` without limit** — returns all results; use `.exec(100)` during development
3. **`NoneObject` vs `None`** — many methods return `NoneObject`; use `isinstance(x, NoneObject)` not `x is None`
4. **`with_callee_names` vs `with_one_of_callee_names`** — `with_callee_names` requires ALL present; `with_one_of_callee_names` requires ANY
5. **`sensitivity` parameter** — name matching is case-sensitive by default; pass `sensitivity=False` for case-insensitive
6. **Trying to read on-chain state** — Glider does NOT support reading runtime state (storage values, balances). Query against source code structure only
7. **Overly broad functional queries** — trying to find "any function that does X" by behavior is usually too computationally intensive. Anchor on a specific protocol interface signature or structural property
8. **Level confusion** — after `.contracts().exec()` you get contract objects; call `.functions()` on the list to go back down
9. **Filter out interfaces** — use `.non_interface_contracts()` or filter with `lambda fn: any(fn.instructions().exec())` to skip empty stubs
10. **`instructions_recursive()` is not a query builder** — it returns a materialized list; `exec()` is not needed and query-builder methods like `.with_callee_name()` cannot be chained after it. Use `filter()` or direct iteration. To continue a declarative chain needing `.with_callee_name()`, use `.instructions()` instead.
11. **Accumulating intermediate state before checking** — iterate over instructions and evaluate conditions inline. Collecting callee sets, source strings, or other state before acting adds unnecessary complexity when a single-pass loop achieves the same result.

---

## Expressing Behavioral Patterns Structurally

When a query targets a behavioral pattern — "find functions that read oracle prices", "find contracts that interact with a DEX" — identify the protocol interface that defines that behavior, then build the query from that structural anchor.

**Example: finding functions that read a price oracle**

The Chainlink price feed interface defines `latestRoundData()` and `getRoundData(uint80)`. Any function that reads Chainlink prices calls one of these methods — a structural fact that is invariant to how the developer named the outer function or contract:

```python
price_readers = Functions().with_one_of_callee_names(["latestRoundData", "getRoundData"]).exec(100)
```

A function named `updateCollateral()`, `_settlePnl()`, or `liquidate()` will all appear if they call `latestRoundData()` internally. The query is complete because it targets what the code *does*, not what it is *named*.

**Reasoning process for any behavioral query:**
1. Identify the protocol or operation of interest
2. Look up its actual ABI method names (the Solidity interface definition)
3. Use those exact names as the structural anchor — `with_callee_name()`, `with_callee_names()`, or `callee_values().name`
4. Navigate from there using the call graph, data flow, or CFG as needed
