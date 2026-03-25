---
name: glider-query-writing
description: Write Glider queries to analyze Solidity smart contracts for security vulnerabilities, labeling, and pattern detection. Use when the user asks to write, create, fix, or improve a Glider query, or mentions Glider, smart contract analysis, or Solidity security scanning.
---

# Glider Query Writing

Use this skill to write correct, efficient Glider queries with minimal context loading.

## Start Here

1. Pick the right entry point: `Contracts()`, `Functions()`, or `Instructions()`.
2. Build a declarative filter chain first.
3. Execute with `.exec(limit)` for bounded results.
4. Switch to imperative logic only when declarative filters cannot express the condition.

## Query Skeleton (Always Valid)

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
        .with_name("X")
        .exec(100)
    )
```

## Hard Rules

- Always `from glider import *`
- Must define `def query()` and **always return a list**
- If no results, return `[]`
- If API returns one object (for example `constructor()`), wrap it as `[obj]`
- Return only `Contract`, `Function`, `Modifier`, or `Instruction` objects
- Include docstring metadata: `@title`, `@description`, `@tags`
- `.exec(limit, offset)` materializes a query

## Data Labeling Rule

For labeling/classification tasks, use main contracts only:

```python
contracts = Contracts.mains().with_name("MyContract").exec(100)
```

## Execution Limits

- Timeout: 1000 seconds
- Output size: 200KB
- Parallelism: one query at a time per user
- Python runtime is sandboxed

## Navigation Model

```text
Contracts  ──.functions()──>  Functions  ──.instructions()──>  Instructions
Contracts  <──.contracts()──  Functions  <──.functions()──────  Instructions
```

Use this to move up/down levels as needed.

## APIList/APISet Reminder

Results from `.exec()` are `APIList`/`APISet` with:

- auto-chained method calls across all elements
- `.filter(lambda x: ...)` for post-filtering
- flattening behavior that supports concise chains

## Common Pitfalls

1. Forgetting `.exec()` (query never runs)
2. `.exec()` with no limit during development (too broad)
3. Confusing `NoneObject` with `None`
4. Mixing ALL vs ANY filters (for example `with_callee_names` vs `with_one_of_callee_names`)
5. Overusing recursive traversals before trying non-recursive options
6. Treating Glider like on-chain state access (it is source-structure analysis)

## Load References On Demand

Read only what the current task requires:

| Task | Load |
|------|------|
| Writing a new query from scratch | [patterns.md](references/patterns.md) |
| Entry point choice, performance, output guidance | [techniques.md](references/techniques.md) |
| CFG, data flow, value tree, level navigation | [navigation.md](references/navigation.md) |
| `NoneObject`, exceptions, wrong/right patterns | [error-handling.md](references/error-handling.md) |
| Contract/Function/Instruction filter methods | [contracts-api.md](references/contracts-api.md), [functions-api.md](references/functions-api.md), [instructions-api.md](references/instructions-api.md) |
| State vars, events, structs, loops, inheritance | [api-extended.md](references/api-extended.md) |
| Dedup, union/intersection/subtraction, snippets | [recipes.md](references/recipes.md) |
