# Query Development Techniques

## Query Development Workflow

### 1. Start Broad, Then Narrow

Begin with the simplest possible chain that captures your target:

```python
# Step 1: find all matching functions
Functions().with_signature("delegates(address)").exec()
```

Run it, look at the result count. Then add one filter at a time:

```python
# Step 2: add source code check
res = Functions().with_signature("delegates(address)").exec()
res.filter(lambda f: "address(0)" in f.source_code())
```

```python
# Step 3: pivot to contract, check sibling functions
for func in res:
    if "address(0)" in func.source_code():
        callers = func.get_contract().functions().with_callee_names(["_delegate"]).exec()
        # ... further filtering
```

### 2. Use a Known-Positive as a Litmus Test

If your query is based on a known pattern, include the contract address where you first found it. If that contract disappears from results as you add filters, you've become too restrictive.

### 4. Pagination with exec(limit, offset)

```python
# First 100 results
results_page1 = Functions().with_name("transfer").exec(100, 0)
# Next 100 results
results_page2 = Functions().with_name("transfer").exec(100, 100)
```

During development, always pass a limit to `.exec()` so queries finish quickly.

---

## Debugging Techniques

### print() for Inspection

`print()` output appears in the Glider IDE Debug panel:

```python
def query():
    functions = Functions().with_name("swap").exec(10)
    functions.filter(lambda f: print(f.address()))  # prints each address
    return functions
```

### Inspect Values with filter() + print()

`filter()` iterates every element without dropping it — use it to print without modifying results:

```python
instructions.filter(lambda i: print(i.callee_names()))
for comp in get_components_recursive(instruction):
    print(type(comp).__name__, comp.expression)
```

---

## Performance Optimization

### Cost Hierarchy

Operations from cheapest to most expensive:

1. **Declarative filters** (DB-level) — `with_name()`, `with_signature()`, `with_properties()`, `with_callee_name()` — essentially free
2. **`.exec()`** — materializes results from DB; use limit to control cost
3. **Source code string checks** — `source_code()` then `in` — cheap per item
4. **Non-recursive CFG/DF** — `next_instruction()`, `forward_df()`, `backward_df()` — moderate
5. **`callee_functions_recursive()` / `caller_functions_recursive()`** — expensive, traverses call tree
6. **Recursive CFG/DF** — `forward_df_recursive()`, `backward_df_recursive()` — very expensive, crosses function boundaries

### Push DB Filters Before Python Logic

Always put as many declarative filters as possible before `.exec()`, then do Python-level logic on the smaller result set:

```python
# GOOD — DB does the heavy filtering
results = (
    Functions()
    .with_properties(FunctionFilters.IS_PUBLIC | FunctionFilters.IS_EXTERNAL)
    .with_callee_names(["selfdestruct"])
    .without_modifier_names(["onlyOwner"])
    .exec(100)
)
for fn in results:
    # light Python logic on small set

# BAD — fetches everything, filters in Python
results = Functions().exec(10000)
for fn in results:
    if fn.is_public() and "selfdestruct" in fn.source_code():
        ...
```

### Avoid Re-Executing Sub-Queries in Loops

```python
# BAD — re-executes the same query on every iteration
for contract in contracts:
    fns = contract.functions().with_name("transfer").exec()  # DB call each time
    for fn in fns:
        instrs = fn.instructions().exec()  # another DB call each time
        ...

# BETTER — cache the outer query, minimize inner queries
for contract in contracts:
    fns = contract.functions().exec()  # one DB call
    transfer_fns = [f for f in fns if f.name == "transfer"]  # Python filter
    ...
```

### exec() Placement Strategy

- **exec() early**: when you need to iterate results in Python with imperative logic
- **exec() late**: when you can keep chaining declarative filters

```python
# exec() late — chain keeps narrowing in DB
Functions().with_name("swap").instructions().external_calls().exec(100)

# exec() early — need Python logic between steps
fns = Functions().with_name("swap").exec(100)
for fn in fns:
    if custom_complex_check(fn):
        results.append(fn)
```

### Use String Checks as a Fast Pre-Filter

String checks on `source_code()` are much faster than structural analysis. Use them to quickly eliminate non-matches before doing expensive operations:

```python
for fn in functions:
    if "selfdestruct" not in fn.source_code():
        continue  # fast skip
    # now do expensive structural analysis
    for instr in fn.instructions().exec():
        ...
```

---

## Decision Guide: Choosing Your Entry Point

### Start from Instructions() when:
- You know the specific function call you're looking for (e.g., `selfdestruct`, `delegatecall`, `ecrecover`)
- You want to find specific operations (storage writes, assembly blocks)
- You need instruction-level granularity in results

```python
Instructions().with_callee_name("selfdestruct").exec(100)
Instructions().low_level_external_calls().exec(100)
```

### Start from Functions() when:
- You're filtering by function properties (visibility, payable, modifiers)
- You know the function name or signature
- You want to check function-level attributes (arguments, callee tree)

```python
Functions().with_signature("transfer(address,uint256)").with_properties(FunctionFilters.IS_PUBLIC).exec(100)
```

### Start from Contracts() when:
- You're looking for contracts matching an interface pattern (ERC20, ERC721, etc.)
- You need to check contract-level structure (what functions exist together)
- You're filtering by contract name, compiler version, or address

```python
Contracts().with_all_function_signatures(["transfer(address,uint256)", "approve(address,uint256)"]).exec(100)
```

### String Matching vs Structural API

| Approach | Use When | Example |
|----------|----------|---------|
| `source_code()` + `in` | Quick pre-filter, checking for keywords | `"address(0)" in fn.source_code()` |
| `with_callee_name()` | Finding specific function calls | `.with_callee_name("selfdestruct")` |
| `with_callee_signature()` | Distinguishing overloaded functions | `.with_callee_signature("transfer(address,uint256)")` |
| `get_value()` decomposition | Inspecting call arguments, operators | `instr.get_value().get_args()` |
| Data flow (`forward_df`) | Tracing where a value flows | `instr.forward_df()` |

**Rule of thumb:** Use declarative API methods first. Fall back to `source_code()` string checks for things the API can't express. Use data flow only when you need to trace value propagation.

---

## Query Output Guidance

### What Each Return Type Shows in Glider IDE

| Return Type | Output Panel Shows |
|-------------|-------------------|
| `Contract` | Contract address (link to explorer) + contract name |
| `Function` | Full function source code + contract address and name |
| `Instruction` | Function source code with the instruction **highlighted** + contract address and name |

### Choosing the Right Return Granularity

- **Return `Contract`** when the finding is about the contract as a whole (e.g., "this contract is an ERC20 with suspicious patterns")
- **Return `Function`** when the finding is about a specific function (e.g., "this function lacks access control")
- **Return `Instruction`** when the finding is about a specific line/operation (e.g., "this line calls selfdestruct") — this gives the most precise output since the IDE highlights the exact instruction

```python
# Return the most specific level that makes sense for your query
# For "find selfdestruct calls" — return the instruction (highlights the exact line)
return Instructions().with_callee_name("selfdestruct").exec(100)

# For "find functions missing onlyOwner" — return the function
return Functions().without_modifier_name("onlyOwner").exec(100)

# For "find ERC20 contracts" — return the contract
return Contracts().with_all_function_signatures([...]).exec(100)
```
