# infodom-llm-marketplace

Claude Code plugin marketplace for the Infodom / Locumo engineering team.

## Available skills

Skills are grouped under **plugins** (one plugin per team area). Installing a plugin gives you
all of its skills.

### `infrastructure`

| Skill | Claude Code command | Description |
|---|---|---|
| `save-to-docs` | `/infrastructure:save-to-docs` | Save notes to infodom-docs and open a PR |
| `infodom-db` | `/infrastructure:infodom-db` | Schema-aware SQL assistant for the Infodom PostgreSQL database |
| `fix-grafana-dashboard` | `/infrastructure:fix-grafana-dashboard` | Create and modify Grafana dashboard panels |
| `deploy-on-cluster` | `/infrastructure:deploy-on-cluster` | Scaffold a new sre-tool deployment in `gitops/sre-tools`, or explain how the setup works |
| `create-skill` | `/infrastructure:create-skill` | Scaffold a new grouped skill with Claude, Codex, and Cursor configs |

## Install (Claude Code)

> This section covers **Claude Code**. The `/plugin` commands below are Claude Code commands.
> For other editors, see [Codex CLI & Cursor compatibility](#codex-cli--cursor-compatibility) below.

### 1. Add the marketplace

```shell
/plugin marketplace add infodomy/infodom-llm-marketplace
```

### 2. Install a plugin

```shell
/plugin install infrastructure@infodom-llm
```

### 3. Use a skill

Skills are namespaced with the plugin (group) name:

```shell
/infrastructure:save-to-docs some topic to document
/infrastructure:infodom-db
/infrastructure:fix-grafana-dashboard add a panel showing request rate
/infrastructure:deploy-on-cluster deploy uptime-kuma exposed via tailscale
/infrastructure:create-skill
```

## Update (Claude Code)

Pull the latest plugin versions:

```shell
/plugin marketplace update infodom-llm
/plugin update infrastructure@infodom-llm
```

Or update all installed plugins:

```shell
/plugin update
```

## Remove (Claude Code)

```shell
/plugin uninstall infrastructure@infodom-llm
/plugin marketplace remove infodom-llm
```

## Codex CLI & Cursor compatibility

All skills in this marketplace are compatible with **OpenAI Codex CLI** and **Cursor** as well as Claude Code. All three platforms use the same `SKILL.md` format for skills.

The repo ships a marketplace manifest and per-plugin manifests for each platform, side-by-side:

```
.claude-plugin/marketplace.json     ← Claude Code marketplace
.cursor-plugin/marketplace.json     ← Cursor marketplace

plugins/<group>/
  .claude-plugin/plugin.json        ← Claude Code
  .codex-plugin/plugin.json         ← Codex CLI
  .cursor-plugin/plugin.json        ← Cursor
  skills/<skill>/SKILL.md           ← one per skill, shared by all platforms
```

### Install in Codex CLI

```shell
codex plugin marketplace add infodomy/infodom-llm-marketplace
codex plugin install infrastructure
```

Use a skill:

```shell
/save-to-docs
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

   Cursor reads `.cursor-plugin/marketplace.json` and lists the available plugins.
4. Click **Install** on the plugins you want (e.g. `infrastructure`).
   Plugins can be installed at the user level or scoped to a specific project.

> Admins can mark plugins as **required** for a distribution group, in which case they are
> installed automatically for everyone in that group. Enable **Auto Refresh** to keep the
> marketplace in sync with new releases.

Skills, rules, and MCP servers bundled in a plugin become available once it is installed.

Use a skill by typing its name in the Cursor chat:

```shell
/save-to-docs
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
- **`deploy-on-cluster`** — expects the `infodom-IaC` repository cloned at `~/Desktop/startup/infodom/infodom-IaC/`; scaffolds tools into `gitops/sre-tools/` and follows the existing `deploy.sh` + Helm wrapper-chart conventions. Scaffolds files only — it does not commit, push, or run helm.

## Versioning

Each merged PR automatically bumps the patch version of all plugins and creates a GitHub release. Pin to a specific version with:

```shell
/plugin marketplace add infodomy/infodom-llm-marketplace@v1.0.3
```
