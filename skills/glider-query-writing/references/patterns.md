# Query Patterns

## Pattern 1: Pure Declarative Chain

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

## Pattern 2: Declarative + APIList Chaining

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

## Pattern 3: Level Navigation

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

## Pattern 4: Iterate + Inspect

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

## Pattern 5: Component Decomposition


```python
for component in get_components_recursive(instruction):
    if isinstance(component, Call) and component.name == "call":
        target = component.get_call_qualifier()
        for point in target.backward_df_recursive():
            if isinstance(point, Instruction) and "abi.decode" in point.callee_names():
                results.append(instruction)
```
    