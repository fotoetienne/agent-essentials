# starter

Example plugin in the [`agent-essentials`](../../README.md) marketplace. Ships one skill (`hello`) as a smoke test for marketplace installation.

## Contents

- **skills/hello** — greets the user and confirms the install worked.

## Install just this plugin

```
/plugin marketplace add fotoetienne/agent-essentials
/plugin install starter@agent-essentials
```

## Versioning

Bump `version` in `.claude-plugin/plugin.json` to push updates. Tag releases as `starter-v<x.y.z>`.
