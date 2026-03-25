# Instructions API

## Filtering

```python
Instructions()
    .with_callee_name("selfdestruct")
    .with_one_of_callee_names(["require", "assert"])
    .with_all_callee_names(["transfer", "balanceOf"])
    .without_callee_names(["require"])
    .with_callee_signature("transfer(address,uint256)")
```

## Call Type Filters

```python
Instructions()
    .external_calls()
    .internal_calls()
    .low_level_external_calls()
    .low_level_static_calls()
    .delegate_calls()
    .library_calls()
    .calls()
```

## Instruction Type Filters

```python
.expression_instructions() / .if_instructions()
.start_asm_instructions() / .return_instructions()
.throw_instructions() / .start_loop_instructions()
.new_variable_instructions() / .new_contract_instructions()
```

## Instruction Instance

```python
instr.source_code() / instr.callee_names() / instr.source_lines()
instr.get_value()
instr.get_component(i) / instr.get_components()
instr.get_dest() / instr.get_dests()
instr.get_parent()

instr.is_call() / instr.is_if() / instr.is_return()
instr.is_storage_write() / instr.is_storage_read()
instr.is_from_assembly() / instr.is_new_contract()

instr.next_instruction()
instr.next_instructions()
instr.previous_instruction()
instr.previous_instructions()

instr.next_instructions_recursive()
instr.previous_instructions_recursive()
```

## Data Flow

```python
instr.forward_df()
instr.backward_df()
instr.has_global_df()
instr.is_tainted()

instr.forward_df_recursive()
instr.backward_df_recursive()
```

Performance rule: prefer non-recursive traversal and data-flow methods first.

## Value System

`instr.get_value()` returns a `Value` subtype:

- `Call`: `.name`, `.signature`, `.get_args()`, `.get_arg(i)`, `.get_function()`, `.get_call_qualifier()`, `.get_call_value()`
- `Operator`: `.get_operator()`
- `Literal`: `.value`, `.type`
- `IndexAccess`: `.get_sequence()`, `.get_index()`
- `VarValue`: `.get_object_of_var()`, `.get_defining_points()`, `.is_depending_on_any_argument()`

All values support `.expression`, `.source_code()`, `.type`, `.is_tainted()`, `.forward_df()`, `.backward_df()`, `.get_vars()`, `.get_state_vars()`.

## Operator and Global Filters

```python
Instructions().with_globals(GlobalFilters.MSG_VALUE)
Instructions().with_operators(OperatorFilters.ASSIGN & OperatorFilters.ADDITION)
```

### OperatorFilters

- Binary: `POWER`, `MULTIPLICATION`, `DIVISION`, `MODULO`, `ADDITION`, `SUBTRACTION`, `LEFT_SHIFT`, `RIGHT_SHIFT`, `AND`, `CARET`, `OR`, `LESS`, `GREATER`, `LESS_EQUAL`, `GREATER_EQUAL`, `EQUAL`, `NOT_EQUAL`, `ANDAND`, `OROR`
- Unary: `NOT`, `TILD`, `DELETE`, `PLUSPLUS`, `MINUSMINUS`, `PLUS`, `MINUS`
- Index: `INDEX_ACCESS`
- Assignment: `ASSIGN`, `ASSIGN_OR`, `ASSIGN_CARET`, `ASSIGN_AND`, `ASSIGN_LEFT_SHIFT`, `ASSIGN_RIGHT_SHIFT`, `ASSIGN_ADDITION`, `ASSIGN_SUBTRACTION`, `ASSIGN_MULTIPLICATION`, `ASSIGN_DIVISION`, `ASSIGN_MODULO`

### ModifierFilters

For `Modifiers().with_properties(...)`: `HAS_ARGS`, `HAS_STATE_VARIABLES_READ`, `HAS_STATE_VARIABLES_WRITTEN`, `HAS_GLOBAL_VARIABLES_READ`, `HAS_MODIFIERS`, `HAS_ERRORS`, `HAS_CALLEES`
