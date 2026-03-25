# Functions API

## Filtering

```python
Functions()
    .with_name("transfer") / .without_name("_internal")
    .with_one_of_the_names(["transfer", "transferFrom"])
    .with_name_prefix("_") / .with_name_regex(r"^(get|set)")
    .with_signature("transfer(address,uint256)")
    .with_signatures([...])
    .with_arg_type("address") / .with_arg_count(2)
    .with_callee_names(["selfdestruct"])
    .with_declarer_contract_name("ERC20")
```

## Property-Based Filtering

Use `with_properties()` with boolean expressions from `FunctionFilters`:

```python
Functions()
    .with_properties(FunctionFilters.IS_PUBLIC | FunctionFilters.IS_EXTERNAL)
    .with_properties(FunctionFilters.IS_PAYABLE & FunctionFilters.HAS_ARGS)
    .with_properties(~FunctionFilters.HAS_MODIFIERS)
    .with_properties(
        (FunctionFilters.IS_PUBLIC | FunctionFilters.IS_EXTERNAL)
        & ~FunctionFilters.HAS_MODIFIERS
        & ~FunctionFilters.IS_CONSTRUCTOR
    )
```

Deprecated: `with_all_properties`, `with_one_property`, `without_properties`.

### FunctionFilters

- Visibility: `IS_PUBLIC`, `IS_EXTERNAL`, `IS_INTERNAL`, `IS_PRIVATE`
- Type: `IS_PAYABLE`, `IS_PURE`, `IS_VIEW`, `IS_GLOBAL`, `IS_CONSTRUCTOR`
- Structure: `HAS_CALLEES`, `HAS_ARGS`, `HAS_MODIFIERS`, `HAS_ERRORS`
- State: `HAS_STATE_VARIABLES_READ`, `HAS_STATE_VARIABLES_WRITTEN`, `HAS_GLOBAL_VARIABLES_READ`

## Modifier Filtering

```python
Functions()
    .with_modifier_name("onlyOwner")
    .without_modifier_name("nonReentrant")
    .without_modifier_names(["nonReentrant", "lock", "onlyOwner"])
    .without_modifiers()
```

## Function Instance

```python
func.name / func.signature() / func.source_code()
func.arguments() / func.local_variables()
func.instructions() / func.return_instructions()
func.callee_functions() / func.callee_functions_recursive()
func.caller_functions() / func.caller_functions_recursive()
func.callee_values()
func.modifiers() / func.get_contract()
func.is_payable() / func.is_public() / func.is_external()
func.detect_cve()
```

## Global and Operator Filters

`with_properties`, `with_globals`, and `with_operators` accept boolean expressions using `&`, `|`, and `~`.

```python
Functions().with_properties(FunctionFilters.IS_PAYABLE & ~FunctionFilters.IS_VIEW)
Functions().with_globals(GlobalFilters.MSG_SENDER | GlobalFilters.TX_ORIGIN)
```

### GlobalFilters

`MSG_SENDER`, `MSG_DATA`, `MSG_SIG`, `MSG_VALUE`, `MSG_GAS`, `NOW`, `BLOCK_CHAIN_ID`, `BLOCK_NUMBER`, `BLOCK_TIME_STAMP`, `BLOCK_DIFFICULTY`, `BLOCK_PREVRANDAO`, `BLOCK_GAS_LIMIT`, `BLOCK_COINBASE`, `BLOCK_BASEFEE`, `TX_ORIGIN`, `TX_GASPRICE`
