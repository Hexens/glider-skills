# Query Development Techniques

## Declarative vs Imperative

Glider queries mix two styles:

**Declarative** (fluent chain) — describe *what* you want:
```python
Functions().with_name("swap").with_properties(FunctionFilters.IS_PUBLIC).exec(100)
```

**Imperative** (loops + logic) — describe *how* to filter further:
```python
for fn in functions:
    if fn.instructions().with_callee_name("transfer").exec():
        results.append(fn)
```

Prefer declarative chains where possible — they are faster, more concise, and leverage Glider's DB engine. Fall back to imperative when the logic is too complex for filters.

---

## Query Development Workflow

### 1. Start Broad, Then Narrow

Begin with the simplest possible chain that captures your target:

```python
# Step 1: find all matching functions
Functions().with_signature("delegates(address)").exec(100)
```

Run it, look at the result count. Then add one filter at a time:

```python
# Step 2: narrow with structural callee filter
res = Functions().with_signature("delegates(address)").exec(100)
res.filter(lambda f: not f.instructions().with_one_of_callee_names(["require", "assert"]).exec())
```

```python
# Step 3: pivot to contract, check sibling functions
for func in res:
    if not func.instructions().with_one_of_callee_names(["require", "assert"]).exec():
        callers = func.get_contract().functions().with_callee_names(["_delegate"]).exec()
        # ... further filtering
```

### 2. Match the Vulnerability Condition

Filters should map directly to structural conditions stated in the vulnerability description. When narrowing a query, add checks that are part of what makes the pattern exploitable — a missing structural check, a data-flow constraint, or an absent guard that the vulnerable code lacks. Each added filter should reflect a condition the vulnerability actually requires.

### 3. Use Custom Scope for Fast Iteration

Running against a full chain can take minutes. Use Custom Scope with 1-5 known contracts for rapid feedback:

1. Open My Scopes → Create new scope → add contract addresses
2. Compile (~1 minute)
3. Select scope in IDE → queries run in seconds
4. Once query works, switch to chain-wide execution

### 4. Use a Known-Positive as a Litmus Test

If your query is based on a known pattern, include the contract address where you first found it. If that contract disappears from results as you add filters, you've become too restrictive.

### 5. Pagination with exec(limit, offset)

```python
# First 100 results
results_page1 = Functions().with_name("transfer").exec(100, 0)
# Next 100 results
results_page2 = Functions().with_name("transfer").exec(100, 100)
```

During development, keep the limit on the first `exec()` of a top-level query small (e.g. `.exec(10)`) so queries finish quickly, then raise it once the chain works. This applies to the top-level materialization only — navigation off already-materialized results (`func.instructions()`, `contract.functions()`, `results.contracts()`) is already bounded by its parent, so it takes no argument.

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

### Using filter() as a Debug Iterator

Since `filter()` iterates over each element, use it with `print()` to inspect values without affecting results:

```python
instructions.filter(lambda i: print(i.callee_names()))
instructions.filter(lambda i: print(i.get_value().get_args().expression))
```

### Printing Components of an Instruction

```python
for comp in get_components_recursive(instruction):
    print(type(comp).__name__, comp.expression)
```

### Profile Your Query

Click the Profile button in Glider IDE to see execution time breakdown by function call.

---

## Performance Optimization

### Cost Hierarchy

Operations from cheapest to most expensive:

1. **Declarative filters** (DB-level) — `with_name()`, `with_signature()`, `with_properties()`, `with_callee_name()` — essentially free
2. **`.exec()`** — materializes results from DB; use limit to control cost
3. **Non-recursive CFG/DF** — `next_instruction()`, `forward_df()`, `backward_df()` — moderate
4. **`callee_functions_recursive()` / `caller_functions_recursive()`** — expensive, traverses call tree
5. **Recursive CFG/DF** — `forward_df_recursive()`, `backward_df_recursive()` — very expensive, crosses function boundaries

`Contracts().with_all_function_names()` and `Contracts().with_all_function_signatures()` are expensive contract-level scans. When identifying contracts by function presence, start with `Functions()` and navigate up — the function-level filter is a cheap DB operation:

```python
# Slow — contract-level scan
Contracts().with_all_function_names(["transfer", "transferFrom"]).exec(100)

# Faster — function-level match, navigate to contracts
Functions().with_one_of_the_names(["transfer", "transferFrom"]).exec(500).contracts().exec()
```

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
    if fn.is_public() and fn.instructions().with_callee_name("selfdestruct").exec():
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

Whichever placement you choose, the limit goes on the first `exec()` of the top-level query; any `exec()` that navigates off those materialized results takes no argument.

```python
# exec() late — chain keeps narrowing in DB
Functions().with_name("swap").instructions().external_calls().exec(100)

# exec() early — need Python logic between steps
fns = Functions().with_name("swap").exec(100)
for fn in fns:
    if custom_complex_check(fn):
        results.append(fn)
```

### Use Declarative DB Filters as Pre-Filters

Declarative filters push work to the database engine — use them to narrow the candidate set before expensive Python-level analysis. Never use `source_code()` string checks as a pre-filter; they bypass the structural API and produce brittle results.

```python
# Good — DB filter eliminates non-matching functions before Python logic
for fn in Functions().with_callee_names(["selfdestruct"]).exec(500):
    # only functions that call selfdestruct reach here
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
- You're filtering by contract name, compiler version, or address
- You're applying contract-level structural predicates (`.mains()`, `.non_interface_contracts()`)

```python
Contracts().with_name("UniswapV3Pool").mains().exec(100)
```

When identifying contracts by function presence (interface detection, ERC patterns), start with `Functions()` instead — it's faster and navigates cleanly to contracts:

```python
# Contracts implementing ERC20 transfer — Functions-first (fast)
Functions()
.with_one_of_the_names(["transfer", "transferFrom"])
.exec(500)
.contracts()
.exec()
```

### Structural API Selection Guide

| Approach | Use When | Example |
|----------|----------|---------|
| `with_callee_name()` | Finding a specific function call | `.with_callee_name("selfdestruct")` |
| `with_callee_signature()` | Distinguishing overloaded functions | `.with_callee_signature("transfer(address,uint256)")` |
| `callee_names()` / `builtin_callee_names()` | Checking calls in an instruction | `"keccak256" in instr.builtin_callee_names()` |
| `get_value()` decomposition | Inspecting call arguments, operators | `instr.get_value().get_args()` |
| `get_dests()` | Checking assignment destination name or type | `instr.get_dests()` → `.name` or `.type` |
| `has_global_df()` / `with_globals()` | Detecting msg.sender / tx.origin influence | `instr.has_global_df()` |
| Data flow (`forward_df_recursive`) | Tracing where a value flows or checking guards | `instr.forward_df_recursive()` |

**Rule of thumb:** Always use structural API. For function calls use `callee_names()`. For guard conditions use `forward_df_recursive()` + `is_if()` / `callee_names()`. For variable names or types use `get_dests()`. For global influence use `has_global_df()` or `with_globals()`. Never use `source_code()` for detection — it grep-matches text, not structure.

### Matching Entry-Point Scope to Traversal Scope

When a query combines an intraprocedural entry filter with interprocedural traversal, the two scopes must be consistent.

An intraprocedural filter — such as a callee constraint or a property check on the entry function — only sees what is directly present in that function. Interprocedural traversal (`instructions_recursive()`, `forward_df_recursive()`, `backward_df_recursive()`) crosses function boundaries and can reach operations, variables, and values that exist in sub-functions or downstream callers. When an intraprocedural filter constrains the entry on something that the traversal then looks for interprocedurally, cases where that thing only exists beyond the function boundary are excluded before the traversal runs.

This applies to any data flow pattern — a call that is made in a downstream function, a variable produced in one function and passed to another, or a value that flows through several intermediate calls before reaching the point of interest.

Anchor the entry-point on the **data flow source**: the operation or value you are tracing from. The interprocedural traversal then finds the sink wherever it appears, whether in the same function or across function boundaries.

```python
# Entry anchored on the data flow source
candidates = (
    Functions()
    .with_properties(...)
    .exec(100)
)

for func in candidates:
    source_instrs = func.instructions_recursive().filter(
        lambda i: ...  # matches the source operation or value
    )
    for source_instr in source_instrs:
        for point in source_instr.forward_df_recursive():
            if isinstance(point, Instruction) and ...:  # matches the sink
                results.append(point)
```

Anchoring on the source produces a wider candidate set but fewer false positives — the full path from source to sink is verified structurally rather than assumed by the entry filter.

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
