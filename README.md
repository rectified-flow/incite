# Incite

Financial research skills for AI coding agents, powered by [@openduo/searchis](https://www.npmjs.com/package/@openduo/searchis) evidence retrieval.

## Skills

| Skill | Description |
|-------|-------------|
| `company-research-analyst` | Generate a comprehensive investment research report on any public company |
| `searchis-query` | Search internal financial research documents for verifiable evidence |

## Install

### Option A: Skills CLI (works with Claude Code, Cursor, Windsurf, etc.)

```bash
# Interactive — pick skills and agents
npx -y skills add rectified-flow/incite

# Install a specific skill globally
npx -y skills add rectified-flow/incite --skill company-research-analyst -g

# Install all skills to Claude Code
npx -y skills add rectified-flow/incite --all --agent claude-code

# List available skills without installing
npx -y skills add rectified-flow/incite --list
```

### Option B: Claude Code Plugin

```bash
/plugin marketplace add rectified-flow/incite
/plugin install company-research-analyst@rectified-flow/incite
/plugin install searchis-query@rectified-flow/incite
```

## License

MIT
