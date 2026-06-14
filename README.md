# infodom-llm-marketplace

Claude Code plugin marketplace for the Infodom / Locumo engineering team.

## Available skills

| Plugin | Skill | Description |
|---|---|---|
| `pr-description` | `/pr-description:pr-description` | Create or update GitHub PR descriptions |
| `save-to-docs` | `/save-to-docs:save-to-docs` | Save notes to infodom-docs and open a PR |
| `infodom-db` | `/infodom-db:infodom-db` | Schema-aware SQL assistant for the Infodom PostgreSQL database |
| `fix-grafana-dashboard` | `/fix-grafana-dashboard:fix-grafana-dashboard` | Create and modify Grafana dashboard panels |
| `create-skill` | `/create-skill:create-skill` | Scaffold a new marketplace plugin/skill with Claude, Codex, and Cursor configs |

## Install

### 1. Add the marketplace

```shell
/plugin marketplace add infodomy/infodom-llm-marketplace
```

### 2. Install individual plugins

```shell
/plugin install pr-description@infodom-llm
/plugin install save-to-docs@infodom-llm
/plugin install infodom-db@infodom-llm
/plugin install fix-grafana-dashboard@infodom-llm
```

Or install all at once:

```shell
/plugin install pr-description@infodom-llm save-to-docs@infodom-llm infodom-db@infodom-llm fix-grafana-dashboard@infodom-llm
```

### 3. Use a skill

Skills are namespaced with the plugin name:

```shell
/pr-description:pr-description
/save-to-docs:save-to-docs some topic to document
/infodom-db:infodom-db
/fix-grafana-dashboard:fix-grafana-dashboard add a panel showing request rate
```

## Update

Pull the latest plugin versions:

```shell
/plugin marketplace update infodom-llm
/plugin update pr-description@infodom-llm
```

Or update all installed plugins:

```shell
/plugin update
```

## Remove

```shell
/plugin uninstall pr-description@infodom-llm
/plugin marketplace remove infodom-llm
```

## Codex CLI & Cursor compatibility

All skills in this marketplace are compatible with **OpenAI Codex CLI** and **Cursor** as well as Claude Code. All three platforms use the same `SKILL.md` format for skills.

The repo ships a marketplace manifest and per-plugin manifests for each platform, side-by-side:

```
.claude-plugin/marketplace.json   ← Claude Code marketplace
.cursor-plugin/marketplace.json   ← Cursor marketplace

plugins/<name>/
  .claude-plugin/plugin.json      ← Claude Code
  .codex-plugin/plugin.json       ← Codex CLI
  .cursor-plugin/plugin.json      ← Cursor
  skills/<name>/SKILL.md          ← shared by all
```

### Install in Codex CLI

```shell
codex plugin marketplace add infodomy/infodom-llm-marketplace
codex plugin install pr-description
```

Use a skill:

```shell
/pr-description
/infodom-db
```

Skills in Codex CLI are invoked without the plugin namespace prefix.

### Install in Cursor

Cursor ships a built-in plugin system with **team marketplaces**. Because this repo includes
`.cursor-plugin/` manifests, Cursor can import it directly from GitHub.

1. Open **Dashboard → Settings → Plugins** (the Plugins tab in Cursor Settings).
2. In **Team Marketplaces**, click **Add Marketplace → Import from Repo**.
3. Point it at the GitHub repository:

   ```
   https://github.com/infodomy/infodom-llm-marketplace
   ```

   Cursor reads `.cursor-plugin/marketplace.json` and lists all four plugins.
4. Click **Install** on the plugins you want (e.g. `pr-description`, `infodom-db`).
   Plugins can be installed at the user level or scoped to a specific project.

> Admins can mark plugins as **required** for a distribution group, in which case they are
> installed automatically for everyone in that group. Enable **Auto Refresh** to keep the
> marketplace in sync with new releases.

Skills, rules, and MCP servers bundled in a plugin become available once it is installed.

Use a skill by typing its name in the Cursor chat:

```shell
/pr-description
/infodom-db
```

Like Codex, Cursor invokes skills without the plugin namespace prefix.

### Version sync

The release workflow keeps `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, and
`.cursor-plugin/plugin.json` in sync — all three always have the same version number after a
PR merge.

## Notes

- **`save-to-docs`** — expects the `infodom-docs` repository cloned at `~/Desktop/startup/infodom/infodom-docs/`.
- **`fix-grafana-dashboard`** — requires Playwright MCP and access to `sre-tools/grafana/`; best used from the `infodom-IaC` repo.
- **`infodom-db`** — schema reference files are bundled with the plugin. When working in `infodom-IaC`, the skill also reads from `.claude/skills/infodom-db/schema/`.

## Versioning

Each merged PR automatically bumps the patch version of all plugins and creates a GitHub release. Pin to a specific version with:

```shell
/plugin marketplace add infodomy/infodom-llm-marketplace@v1.0.3
```
