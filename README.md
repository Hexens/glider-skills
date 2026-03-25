# Glider Query Writing

An AI agent skill that teaches your coding assistant how to write [Glider](https://glider.hexens.io/) queries for analyzing Solidity smart contracts — security vulnerabilities, labeling, and pattern detection.

Built by [Hexens](https://hexens.io/).

## Installation

### Claude Code

Via the plugin marketplace:

```bash
/plugin marketplace add Hexens/glider-skills
```

Or install locally:

```bash
git clone git@github.com:Hexens/glider-skills.git
cd glider-skills && /plugins install .
```

### Cursor

Clone into your Cursor skills directory:

```bash
git clone git@github.com:Hexens/glider-skills.git ~/.cursor/skills/glider-skills
```

The skill is auto-detected via `SKILL.md`. Verify in **Cursor Settings > Skills**.

## Structure

```
.claude-plugin/
  plugin.json                     # Claude Code plugin metadata
skills/
  glider-query-writing/
    SKILL.md                      # Entry point — loaded first by the agent
    references/
      patterns.md                 # Common query patterns
      techniques.md               # Performance & debugging guidance
      navigation.md               # CFG, data flow, value tree
      error-handling.md           # NoneObject, anti-patterns
      contracts-api.md            # Contracts API reference
      functions-api.md            # Functions API reference
      instructions-api.md         # Instructions API reference
      api-extended.md             # State vars, events, structs, loops
      recipes.md                  # Reusable snippets & helpers
```

The agent reads `SKILL.md` first, then loads reference files on demand to keep context usage efficient.