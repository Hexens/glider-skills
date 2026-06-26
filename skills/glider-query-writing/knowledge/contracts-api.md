# Contracts API

## APIList / APISet

Results from `.exec()` return `APIList` or `APISet` (not plain lists).

**Chained attribute access** — calls the method on every element automatically:
```python
contracts.exec(100).functions().exec()       # gets all functions from all contracts
```

**filter()** — keep elements where predicate returns `True`:
```python
results.filter(lambda x: x.name == "ERC20")
```

**Unidimensional flattening** — nested `APIList[APIList[T]]` flattens to `APIList[T]`, enabling full declarative chains without loops.

---

## Filtering

```python
Contracts()
    .with_name("ERC20")
    .with_name_prefix("ERC") / .with_name_suffix("Token")
    .with_name_regex(r"ERC\d+")
    .with_name_not("Interface")
    .with_address("0x...")
    .mains()
    .non_interface_contracts() / .interface_contracts()
    .with_compiler_range("0.8.0", "0.8.28")
```

## Filtering by Members

```python
Contracts()
    .with_function_name("transfer")
    .with_all_function_names(["transfer", "approve"])   # has ALL
    .with_one_of_the_function_names(["mint", "burn"])   # has ANY
    .with_function_signature("transfer(address,uint256)")
    .with_all_function_signatures([...])
    .with_event_name("Transfer")
    .with_struct_name("Position")
    .with_error_name("InsufficientBalance")
```

## Contract Instance

```python
contract.name / contract.address() / contract.source_code()
contract.functions() / contract.modifiers() / contract.constructor()
contract.state_variables() / contract.events() / contract.structs()
contract.enums() / contract.errors()
contract.base_contracts()          # -> Contracts | NoneObject (full inheritance chain)
contract.direct_base_contracts()   # -> Contracts | NoneObject (direct parents only)
contract.derived_contracts()       # -> Contracts | NoneObject
contract.is_main()                 # True for deployed top-level contracts
contract.call_graph()
```
