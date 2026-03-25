# Error Handling & Anti-Patterns

## Error Handling and Defensive Coding

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

Filter predicates that throw exceptions will silently drop the element. Wrap risky operations:

```python
def safe_check(instr):
    try:
        value = instr.get_value()
        if isinstance(value, NoneObject):
            return False
        return isinstance(value, Call) and value.name == "transfer"
    except Exception:
        return False

instructions.filter(safe_check)
```

---

## Anti-Patterns and Common Mistakes

### 1. Assuming get_value() Returns a Call

`get_value()` can return any `Value` subtype — `Call`, `Operator`, `Literal`, `IndexAccess`, `VarValue`, or `NoneObject`. Always check:

```python
# WRONG — crashes if value is not a Call
args = instr.get_value().get_args()

# CORRECT
value = instr.get_value()
if isinstance(value, Call):
    args = value.get_args()
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

### 6. Expensive Operations Inside filter()

`filter()` runs the predicate on every element. Avoid expensive operations inside:

```python
# BAD — backward_df_recursive() on every instruction
instructions.filter(lambda i: any(i.backward_df_recursive()))

# BETTER — pre-filter with cheap checks first
instructions.filter(lambda i: i.has_global_df()).filter(expensive_check)
```
