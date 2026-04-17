# agent-essentials

A personal [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) — skills, slash commands, agents, hooks, and MCP servers, bundled as installable plugins.

## Install

```
/plugin marketplace add fotoetienne/agent-essentials
/plugin marketplace list
/plugin install <plugin-name>@agent-essentials
```

Update later with:

```
/plugin marketplace update agent-essentials
```

## Plugins

| Plugin | Description |
| --- | --- |
| [`starter`](plugins/starter) | Example plugin with a single `hello` skill. Smoke test for the marketplace install. |
| [`tiger-style`](plugins/tiger-style) | Apply TigerBeetle's TigerStyle coding principles (safety, performance, DX) when writing or reviewing code. Language-agnostic with Rust/Java/Kotlin/TypeScript guides. |

## Repo layout

```
agent-essentials/
├── .claude-plugin/
│   └── marketplace.json        # marketplace manifest, lists plugins
└── plugins/
    └── <plugin-name>/
        ├── .claude-plugin/
        │   └── plugin.json     # plugin manifest (name, version, author)
        ├── skills/             # auto-discovered: skills/<name>/SKILL.md
        ├── commands/           # auto-discovered: commands/*.md
        ├── agents/             # auto-discovered: agents/*.md
        ├── hooks/hooks.json    # optional
        ├── .mcp.json           # optional, bundled MCP servers
        └── README.md
```

Components are picked up by directory convention — no need to list them explicitly in `plugin.json` unless overriding the default path.

## Adding a new plugin

1. `mkdir -p plugins/<name>/.claude-plugin plugins/<name>/skills`
2. Write `plugins/<name>/.claude-plugin/plugin.json` with `name`, `version`, `description`, `author`.
3. Drop skills under `skills/<skill-name>/SKILL.md` (YAML frontmatter `name` + `description`, then body).
4. Add the plugin to `plugins[]` in `.claude-plugin/marketplace.json` with a relative `./plugins/<name>` source.
5. Bump the plugin's `version` on every release; tag `git tag <name>-v<x.y.z>`.

## Local testing

Before pushing:

```
/plugin marketplace add /Users/sspalding/prj/agent-essentials
/plugin install <plugin>@agent-essentials
```

## License

MIT — see [LICENSE](LICENSE).
