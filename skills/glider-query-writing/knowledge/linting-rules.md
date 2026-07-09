# Linting Rules

These rules apply to **every query written by this skill**. Before returning any query, check all rules and fix any violations.

---

## Rule 1: No Underscore-Prefixed Variable Names

Variable names must **not** start with `_`. Applies to all local variables, helper functions, loop variables, dictionaries, caches, and lambdas defined inside `query()` or any helper scope.

**Invalid:**
```python
_require_cache = {}
_addr_args = []
for _fn in functions:
    ...
```

**Valid:**
```python
require_cache = {}
addr_args = []
for fn in functions:
    ...
```

Does **not** apply to:
- Python dunder methods (`__init__`, `__name__`, etc.)
- Glider API arguments or object attributes you don't define (e.g. `_callable` is a Glider internal)

---

## Rule 2: Use get_dest() to Inspect Assignment Destination Types

When checking whether an instruction assigns a result to a variable of a specific type (e.g., an address returned from `abi.decode`), use `get_dest()` or `get_dests()` and inspect the `.type` attribute of the destination value.

**Valid:**
```python
for dest in instr.get_dests():
    if str(dest.type) == "address":
        results.append(instr)
```

---

## Rule 3: Use callee_names() to Identify Called Functions

To check whether a specific function or built-in is called within an instruction or function, use the structural API:
- **Instruction level**: `instruction.callee_names()` for regular calls; `instruction.builtin_callee_names()` for built-ins (keccak256, ecrecover, abi.encode, non-contract calls, etc.)
- **Function level**: `function.callee_values().name` to iterate over all calls made within the function

**Valid:**
```python
if "transfer" in instruction.callee_names():
    ...

if "keccak256" in instruction.builtin_callee_names():
    ...

for call in func.callee_values():
    if call.name == "transfer":
        ...
```

Note: `callee_names()` covers regular contract calls only — it does not include native Solidity operations like `delegatecall` or `staticcall`. To find those, use the dedicated instruction filters: `instructions().low_level_external_calls()` for `delegatecall`/`.call()`, and `instructions().low_level_static_calls()` for `staticcall`. These can be used at the entry-point level (`Instructions().low_level_external_calls().exec(100)`) or when filtering within a function (`func.instructions().low_level_external_calls().exec()`).

---

## Rule 4: Use _recursive() Variants by Default

When performing data flow analysis with `forward_df()` / `backward_df()`, use the `_recursive()` variants as the default. The recursive variants cross function boundaries and produce complete results. Reserve the non-recursive variants only when you have an explicit reason to constrain analysis to a single function scope.

The same principle applies to CFG navigation: prefer `next_instructions_recursive()` / `previous_instructions_recursive()` unless traversal must be intentionally limited.

The same principle applies to instruction traversal: when checking what a function does across its full call tree, use `instructions_recursive()` with `filter()`. This is the correct form when you already have a function object and want to know what it calls — directly or through any sub-functions it invokes.

**Valid:**
```python
for point in value.backward_df_recursive():
    if isinstance(point, Instruction):
        ...

results = ecrecovers.filter(lambda instr:
    not instr.forward_df_recursive().filter(lambda df:
        isinstance(df, Instruction) and (df.is_if() or "require" in df.callee_names())
    )
)

# Checking a function's full call tree — use instructions_recursive()
has_twap = bool(
    func.instructions_recursive().filter(
        lambda i: bool(set(i.callee_names()) & set(TWAP_CALLEES))
    )
)

# Entry-point declarative chain — instructions() is correct here
Functions().instructions().with_callee_name("slot0").exec(100)
```

---

## Rule 5: Always Filter Results to Main Contracts

Every query must return results from main contracts only. Main contracts are the deployed, top-level contracts — not libraries, interfaces, or abstract contracts. Apply the appropriate check at whichever level you are working:

```python
# At the contract level — preferred when using Contracts()
contracts = Contracts().mains().exec(100)

# At the function level
for func in functions:
    if not func.get_contract().is_main():
        continue

# At the instruction level
for instr in instructions:
    if not instr.get_parent().get_contract().is_main():
        continue
```

When using declarative chains with `Contracts()`, use `.mains()` as an early filter. When iterating over functions or instructions, check `is_main()` at the top of the loop and skip non-main results.

**Scoping to main contracts produces unique results.** Each deployed contract has a unique address in the Glider DB. A query anchored to `.mains()` or gated on `is_main()` returns each contract, function, or instruction exactly once — this is the uniqueness mechanism. No additional key tracking or dedup logic is needed.

---

## Rule 6: Filter APIList and APISet Results with filter()

When a Glider method returns an APIList or APISet, the result is already materialized — `exec()` is not needed and query-builder methods (`.with_callee_name()`, `.with_properties()`, `.exec()`) cannot be chained after it. Use `.filter()` with a structural predicate lambda to narrow results, or iterate directly.

This applies to any method that returns a materialized collection: `instructions_recursive()`, `forward_df_recursive()`, `backward_df_recursive()`, `callee_values()`, `get_dests()`, `exec()` output, and others.

**Valid:**
```python
# Filter an APIList result
instrs = func.instructions_recursive().filter(lambda i: "latestRoundData" in i.callee_names())

# Direct iteration
for instr in func.instructions_recursive():
    if "latestRoundData" in instr.callee_names():
        results.append(instr)

# callee_values() returns APIList[Call] — filter or iterate directly
oracle_calls = func.callee_values().filter(lambda c: c.name == "latestRoundData")
```

---

## Rule 7: Write Direct API Calls; Use isinstance() for Absent Values

Write Glider API calls directly. The Glider sandbox does not raise exceptions from standard API method calls — absent values are returned as `NoneObject`. Use `isinstance(result, NoneObject)` to handle the absent-value case rather than wrapping calls in try/except.

**Valid:**
```python
instrs = func.instructions().exec()
value = instr.get_value()
if isinstance(value, NoneObject):
    continue
if isinstance(value, Call):
    args = value.get_args()
if isinstance(value, ValueExpression):
    for comp in value.get_components():
        if isinstance(comp, Call):
            args = comp.get_args()
```

---

## Rule 8: Use Data-Flow Analysis to Detect Guard Conditions

When checking whether a guard condition (require, assert, or an if-check) is applied against the return value of a function call, use `forward_df_recursive()` to trace data flow from the call instruction and inspect downstream instructions for `require`/`assert` callee names or `is_if()`. This is the structural, API-level way to detect guards — no source code string searching needed.

**Valid:**
```python
df_points = call_instruction.forward_df_recursive()
for inst in df_points:
    if not isinstance(inst, Instruction):
        continue
    if "require" in inst.callee_names() or "assert" in inst.callee_names():
        return True
    if inst.is_if():
        return True
return False
```

Additional checks may be helpful here depending on the guard condition check. For example, evaluating whether an integer is non-zero will require additional querying against the guard instruction.

---

## Rule 9: Never Use `source_code()` for Detection Logic

`source_code()` returns raw Solidity text. Using it with `in` or string matching for vulnerability detection is a text grep — not structural analysis. It matches comments, misses semantically equivalent code expressed differently, and produces brittle queries that break on formatting variations.

**Use structural API alternatives instead:**

| Instead of `source_code()` check | Structural alternative |
|---|---|
| `"transfer" in instr.source_code()` | `"transfer" in instr.callee_names()` |
| `"balanceOf" in fn.source_code()` | `fn.instructions().with_callee_name("balanceOf").exec()` |
| `"keccak256" in instr.source_code()` | `"keccak256" in instr.builtin_callee_names()` |
| `"_balances" in instr.source_code()` (storage write target) | `instr.get_dests()` → check `.name` |
| `"address(0)" in df.source_code()` (guard check) | `df.is_if() or "require" in df.callee_names()` |
| `"msg.sender" in instr.source_code()` | `instr.has_global_df()` or `.with_globals(GlobalFilters.MSG_SENDER)` |
| `"reserve0" in contract.source_code()` (keyword sniff) | `contract.state_variables().with_name("reserve0").exec()` |

`source_code()` is acceptable **only for debug `print()` output** — never in filtering or detection logic.

**Invalid:**
```python
if "address(0)" in df.source_code():
    ...
if "balanceOf" in fn.source_code():
    ...
if "msg.sender" in instr.source_code():
    ...
```

**Valid:**
```python
if df.is_if() or "require" in df.callee_names():
    ...
if fn.instructions().with_callee_name("balanceOf").exec():
    ...
if instr.has_global_df():
    ...
```

---

## Rule 10: Target Protocol Interfaces and Structural Properties

Glider queries express static code structure — the call graph, data flow, CFG shape, types. These are invariant facts about the compiled code. When identifying a behavioral pattern, anchor the query on the actual protocol interfaces or structural properties that define that behavior.

A protocol interface is a reliable anchor. Any function that reads a Chainlink oracle calls `latestRoundData()` or `getRoundData()`. Any function that queries a Uniswap V3 pool for the spot price calls `slot0()`. These structural facts hold regardless of what the outer function, contract, or variable is named.

A small set of *exact, known ABI method names* is structural — for example ERC20's `["transfer", "transferFrom"]`. A list of names assembled by reasoning about what something *might be called* (even when applied to callee names rather than outer function names) is a semantic guess: the right question is always "what does this protocol's interface define?" not "what might a developer have named this?"

Use `callee_values()` to inspect all calls a function makes, or `with_callee_name()` / `with_callee_names()` as a DB-level filter:

```python
# Functions that read a Chainlink price feed — anchored on the ABI
def reads_chainlink(func):
    return bool(func.instructions().with_one_of_callee_names(
        ["latestRoundData", "getRoundData"]
    ).exec())

# DB-level entry point — navigate from Instructions up to Functions
Instructions().with_one_of_callee_names(["latestRoundData", "getRoundData"]).functions().exec(100)

# ERC20 token transfers — exact interface entries, not keyword guesses
Instructions().with_one_of_callee_names(["transfer", "transferFrom"]).exec(100)

# Uniswap V3 spot price reads
def reads_spot_price(func):
    return "slot0" in func.callee_values().name
```

`callee_values()` returns an `APIList[Call]` — chained `.name` access returns all called function names as a list, making membership checks straightforward.

---

## Rule 11: Always Check isinstance(point, Instruction) on Data Flow Points

`forward_df_recursive()` and `backward_df_recursive()` return mixed-type iterables — not exclusively `Instruction` objects. Points can also be `ArgumentPoint` or other types that do not expose instruction-specific methods like `callee_names()`, `is_if()`, or `builtin_callee_names()`. Calling these methods on a non-Instruction point will fail at runtime.

Always check `isinstance(point, Instruction)` before calling any instruction-specific method. In `.filter()` lambdas, include the isinstance check as the first condition.

```python
# Instruction — isinstance works correctly
for point in instr.forward_df_recursive():
    if not isinstance(point, Instruction):
        continue
    if "require" in point.callee_names() or point.is_if():
        ...

# filter() lambda with isinstance guard
instr.forward_df_recursive().filter(
    lambda p: isinstance(p, Instruction) and (p.is_if() or "require" in p.callee_names())
)

# Other Point subtypes — use "ClassName" in str(point); isinstance may not work
for point in value.backward_df_recursive():
    if isinstance(point, Instruction) and "abi.decode" in point.builtin_callee_names():
        ...
    if "ArgumentPoint" in str(point):
        ...
```

---

## Rule 12: Match Entry-Point Scope to Traversal Scope

When a query uses interprocedural traversal, the entry-point filter must be anchored at the same conceptual scope as the traversal — specifically, on the **data flow source**.

An intraprocedural entry filter only sees what is directly present in the matched function. If the traversal then searches for a related pattern interprocedurally, any case where that pattern only exists across function boundaries is excluded before the traversal runs. This creates a hidden false-negative: the query appears to search interprocedurally but the entry constraint has already ruled out the cases that require it.

Anchor the entry-point on the source of the data flow. Let the interprocedural traversal find the sink — whether it is in the same function or reached through intermediate calls, variables, or return values.

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
