---
name: glider-query-writing
description: Write Glider queries to analyze Solidity smart contracts for security vulnerabilities, labeling, and pattern detection. Use when the user asks to write, create, fix, or improve a Glider query, or mentions Glider, smart contract analysis, or Solidity security scanning.
---

# Glider Query Writing

## Start Here

1. Pick the right entry point: `Contracts()`, `Functions()`, or `Instructions()`.
2. Build a declarative filter chain first.
3. Execute with `.exec(limit)` for bounded results.
4. Switch to imperative logic only when declarative filters cannot express the condition.

## Table of Contents

- [Query Structure](#query-structure)
- [Three Entry Points](#three-entry-points)
- [Core Principle: Static Analysis, Not Semantics](#core-principle-static-analysis-not-semantics)
- [Linting Rules](#linting-rules)
- [Additional Resources](#additional-resources) 

---

## Query Structure

Every Glider query is a Python file with this skeleton:

```python
from glider import *

def query():
    """
    @title: Short descriptive title
    @description: What this query detects and why it matters.
    @author: Author name
    @tags: comma, separated, tags
    @references: https://relevant-link.com
    """
    return (
        Contracts()       # or Functions(), Instructions()
        .with_name("X")   # fluent filter chain
        .exec(100)        # execute with result limit
    )
```

**Rules:**
- Always `from glider import *`
- Must define `def query()` that **always returns a list or APIList** — nothing else, ever
- If returning no results, return `[]` (empty list)
- If an API method returns a single object (e.g. `get_contract()`, `constructor()`), wrap it: `[obj]`
- List elements must be `Contract`, `Function`, `Modifier`, or `Instruction` objects
- Docstring metadata is required: `@title`, `@description`, `@tags`
- `.exec(limit, offset)` materializes the query — returns a list of results
- `print()` output appears in the Debug panel (useful for debugging)

 
### Execution Limits

- **Timeout**: 1000 seconds per query
- **Output size**: 200KB max
- **Python**: strictly sandboxed — not all built-ins available

---

## Three Entry Points

| Entry Point | Returns | Use When |
|-------------|---------|----------|
| `Contracts()` | Contract-level results | Finding contracts by name, interface, function signatures |
| `Functions()` | Function-level results | Finding functions by properties, modifiers, arguments |
| `Instructions()` | Instruction-level results | Finding specific calls, operations, patterns |

```text
Contracts  ──.functions()──>  Functions  ──.instructions()──>  Instructions
Contracts  <──.contracts()──  Functions  <──.functions()──────  Instructions
```

Chain freely: `Instructions().with_callee_name("transfer").functions().contracts().exec(100)`

---

## Core Principle: Static Analysis, Not Semantics

Glider queries express facts about code structure — what is called, how values flow, what the CFG looks like. When identifying a behavioral pattern, target the protocol interfaces and structural properties that define it. A function that reads Chainlink prices always calls `latestRoundData()`; that structural fact is the anchor for the query, regardless of what the outer function or contract is named.

---

## Linting Rules

These rules apply to **every query**. Before returning any query, read and apply all rules in [knowledge/linting-rules.md](knowledge/linting-rules.md).

---

## Additional Resources

Curated guides — read only what you need:


### References to Load

Read only what the current task requires:

| Task | Load |
|------|------|
| Writing a new query from scratch | [patterns.md](references/patterns.md) |
| Entry point choice, performance, output guidance | [techniques.md](references/techniques.md) |
| CFG, data flow, value tree, level navigation | [navigation.md](references/navigation.md) |
| `NoneObject`, exceptions, wrong/right patterns | [error-handling.md](references/error-handling.md) |
| **Always load for any non-trivial query** — essential helper functions (`get_components_recursive`, guard checks, msg.sender validation, storage write detection) plus dedup, union/intersection, and sub-query composition patterns. Missing this file is the most common cause of re-implementing helpers that already exist. | [recipes.md](references/recipes.md) |
 
### Knowledge Base

- [knowledge/instructions-api.md](knowledge/instructions-api.md) — Instructions filtering, call/instruction type filters, instance methods, CFG navigation, data flow, value system, operator/global filters
- [knowledge/functions-api.md](knowledge/functions-api.md) — Functions filtering, property filters, modifier filters, instance methods, GlobalFilters
- [knowledge/contracts-api.md](knowledge/contracts-api.md) — Contracts filtering, member filters, contract instance methods
- [knowledge/api-extended.md](knowledge/api-extended.md) — StateVariables, Events, Errors, Enums, Structs, ArgumentPoints, Condition, Loop, inheritance, TaintEngine
- [knowledge/linting-rules.md](knowledge/linting-rules.md) — Mandatory rules applied to every query before returning
