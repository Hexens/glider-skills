# Instructions API

## APIList / APISet

Results from `.exec()` return `APIList` or `APISet` (not plain lists).

**Chained attribute access** — calls the method on every element automatically:
```python
instructions.exec(100).source_code()         # calls .source_code() on each
instructions.exec(100).next_instruction()    # calls .next_instruction() on each, flattened
```

**filter()** — keep elements where predicate returns `True`:
```python
results.filter(lambda x: "balanceOf" in x.callee_names())
```

**Unidimensional flattening** — nested `APIList[APIList[T]]` flattens to `APIList[T]`, enabling full declarative chains without loops.

---

## Filtering

```python
Instructions()
    .with_callee_name("selfdestruct")
    .with_callee_name_prefix("safe")
    .with_callee_name_suffix("ETH")
    .with_one_of_callee_names(["require", "assert"])    # ANY present
    .with_all_callee_names(["transfer", "balanceOf"])   # ALL must be present
    .without_callee_name("require")
    .without_callee_names(["require"])
    .with_callee_signature("transfer(address,uint256)")
```

## Call Type Filters

```python
Instructions()
    .if_instructions()
    .if_loop_instructions()
    .end_if_instructions()
    .start_loop_instructions()
    .end_loop_instructions()
    .break_instructions()
    .continue_instructions()
    .return_instructions()
    .placeholder_instructions()
    .try_instructions()
    .catch_instructions()
    .entry_point_instructions()
    .asm_block_instructions()
    .end_asm_instructions()
    .internal_calls()
    .library_calls()
    .delegate_calls()
    .delegate_calls_from_assembly()
    .delegate_calls_non_assembly()
    .high_level_static_calls()
    .new_contract_instructions()
    .external_calls()              # high-level external calls
    .low_level_external_calls()    # .call()
    .low_level_static_calls()      # low-level .staticcall()
    .calls()                       # all call instructions
```

## Instruction Type Filters

```python
.expression_instructions()  / .if_instructions()
.start_asm_instructions()   / .return_instructions()
.throw_instructions()       / .start_loop_instructions()
.new_variable_instructions() / .new_contract_instructions()
```

## Instruction Instance

```python
instr.source_code() / instr.callee_names() / instr.source_lines()
instr.builtin_callee_names()     # built-ins only: keccak256, ecrecover, etc.
instr.get_value()                # -> Value (expression tree)
instr.get_component(i) / instr.get_components()
instr.get_dest() / instr.get_dests()
instr.get_parent()               # -> Callable (containing function)

# Type checks
instr.is_call() / instr.is_if() / instr.is_return() / instr.is_expression()
instr.is_storage_write() / instr.is_storage_read()
instr.is_from_assembly() / instr.is_new_contract()
instr.is_if_loop() / instr.is_start_assembly() / instr.is_end_assembly()
instr.is_start_loop() / instr.is_end_loop() / instr.is_end_if()
instr.is_break() / instr.is_continue() / instr.is_throw() / instr.is_try() / instr.is_catch()
instr.is_entry_point() / instr.is_placeholder()

# CFG navigation (non-recursive — fast)
instr.next_instruction()         # -> APIList[Instruction] — immediate successors
instr.next_block()               # -> APIList[Instruction] — until next branch
instr.next_instructions()        # -> APISet[Instruction]
instr.previous_instruction()     # -> APISet[Instruction]
instr.previous_instructions()    # -> APISet[Instruction]

# CFG navigation (recursive — slow, use only when needed)
instr.next_instructions_recursive()
instr.previous_instructions_recursive()
```

## Data Flow

```python
instr.forward_df()                  # points tainted by this instruction (current function)
instr.backward_df()                 # points flowing into this instruction (current function)
instr.has_global_df()               # influenced by msg.sender/tx.origin/etc. (current function)
instr.has_global_df_recursive()     # same, crosses function boundaries
instr.is_tainted()

# Recursive variants — cross function boundaries (much slower)
instr.forward_df_recursive()
instr.backward_df_recursive()
```

**Default rule:** Use `forward_df_recursive()` / `backward_df_recursive()` as the standard choice — they cross function boundaries and produce complete results. Reserve the non-recursive variants only when explicitly constraining to a single function scope.

**Return types:** Both recursive variants return mixed-type iterables — points can be `Instruction`, `ArgumentPoint`, or other subtypes. Before calling any method on a point, add a type guard. For `Instruction`, use `isinstance(point, Instruction)`. For other `Point` subtypes, use `"ClassName" in str(point)` — `isinstance()` may not work correctly for those types.

## Value System

`instr.get_value()` returns a `Value` subclass:

| Type | Key Methods |
|------|-------------|
| `Call` | `.name`, `.signature`, `.get_contract_name()`, `.get_args()`, `.get_arg(i)`, `.get_function()`, `.get_call_qualifier()`, `.get_call_value()`, `.get_call_gas()`, `.get_call_salt()`, `.get_call_type()` → `CallType`, `.kv_parameters()` |
| `Operator` | `.get_operator()` → `OperatorType` |
| `Literal` | `.value`, `.type` |
| `IndexAccess` | `.get_sequence()`, `.get_index()` |
| `VarValue` | `.get_object_of_var()`, `.get_defining_points()`, `.is_depending_on_any_argument()` |
| `ValueExpression` | `.get_component(i)`, `.get_components()` — wraps multiple inner values; iterate components and check each with `isinstance` |
| `TupleExpression` | `.get_component(i)`, `.get_components()` |

All values: `.expression`, `.source_code()`, `.type`, `.is_tainted()`, `.forward_df()`, `.backward_df()`, `.get_vars()`, `.get_state_vars()`

**Note on `get_call_qualifier()`:** Returns a `Value`. Specific subtypes like `StatePoint` will not work with `backward_df()` since they represent state variable points.

**Note on `address(this)`:** `address(this)` is a type conversion, not a function call — `isinstance(value, Call)` returns `False` for it. Detect it via `.expression`: `value.expression == "this"` for an exact match, or `"this" in value.expression` when it may appear nested inside a sub-expression.

## Operator and Global Filters

```python
Instructions().with_globals(GlobalFilters.MSG_VALUE)
Instructions().with_operators(OperatorFilters.ASSIGN & OperatorFilters.ADDITION)
```

### OperatorFilters

- **Binary:** `POWER`, `MULTIPLICATION`, `DIVISION`, `MODULO`, `ADDITION`, `SUBTRACTION`, `LEFT_SHIFT`, `RIGHT_SHIFT`, `AND`, `CARET`, `OR`, `LESS`, `GREATER`, `LESS_EQUAL`, `GREATER_EQUAL`, `EQUAL`, `NOT_EQUAL`, `ANDAND`, `OROR`
- **Unary:** `NOT`, `TILD`, `DELETE`, `PLUSPLUS`, `MINUSMINUS`, `PLUS`, `MINUS`
- **Index:** `INDEX_ACCESS`
- **Assignment:** `ASSIGN`, `ASSIGN_OR`, `ASSIGN_CARET`, `ASSIGN_AND`, `ASSIGN_LEFT_SHIFT`, `ASSIGN_RIGHT_SHIFT`, `ASSIGN_ADDITION`, `ASSIGN_SUBTRACTION`, `ASSIGN_MULTIPLICATION`, `ASSIGN_DIVISION`, `ASSIGN_MODULO`

### ModifierFilters

For `Modifiers().with_properties(...)`: `HAS_ARGS`, `HAS_STATE_VARIABLES_READ`, `HAS_STATE_VARIABLES_WRITTEN`, `HAS_GLOBAL_VARIABLES_READ`, `HAS_MODIFIERS`, `HAS_ERRORS`, `HAS_CALLEES`
