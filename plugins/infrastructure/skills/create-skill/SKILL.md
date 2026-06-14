---
name: create-skill
description: Scaffold a new skill in the infodom-llm marketplace, grouped under a plugin (backend, frontend, infrastructure, …) and wired up for Claude Code, Codex CLI, and Cursor. Use when the user wants to add a new skill or command to this marketplace.
---

# Create Skill

Add a new skill to this marketplace. Skills are **grouped under a plugin** (one plugin per
team area), and a single `SKILL.md` works across **Claude Code**, **OpenAI Codex CLI**, and
**Cursor**. The platforms share the skill body and differ only in the plugin manifest and how
the marketplace lists the plugin.

## When to use

Use this skill when the user asks to:
- add a new skill / command to this marketplace
- scaffold a skill that works in Claude Code, Codex, and Cursor
- "create a new skill called X"

## Key concept: skills grouped under a plugin

A **plugin is a group of related skills** (not one plugin per skill). Each plugin ships
**three manifests** — one per platform — and holds any number of skills under `skills/`.
Both marketplace manifests at the repo root list the plugin (once per plugin, not per skill).

```
.claude-plugin/marketplace.json      ← Claude Code marketplace (lists every plugin)
.cursor-plugin/marketplace.json      ← Cursor marketplace (lists every plugin)

plugins/<group>/
  .claude-plugin/plugin.json         ← Claude Code manifest
  .codex-plugin/plugin.json          ← Codex CLI manifest
  .cursor-plugin/plugin.json         ← Cursor manifest
  skills/
    <skill-a>/SKILL.md               ← shared by all three platforms
    <skill-b>/SKILL.md
```

> Codex has no marketplace manifest of its own — it discovers plugins from the repo layout,
> so only the per-plugin `.codex-plugin/plugin.json` is needed for Codex.

## Step 1: Gather inputs

Ask the user (use the Ask tool) for:
- **group** — which plugin the skill belongs to. Offer the existing groups and an escape hatch:
  - `backend`
  - `frontend`
  - `infrastructure`
  - something else (ask them to name it, kebab-case)

  The group becomes the plugin name. Check whether `plugins/<group>/` already exists:
  - **exists** → you only add a new skill folder (skip Step 3 and Step 4).
  - **new group** → you also create the plugin manifests (Step 3) and register the plugin
    in both marketplace manifests (Step 4).
- **name** — the skill, kebab-case and unique within the group (e.g. `deploy-helper`).
- **description** — one sentence describing the skill.
- **tags / category** — only needed when creating a *new* group plugin (Step 4).

Pick the version to match the plugins currently in `.claude-plugin/marketplace.json`
(read the `version` field). The release workflow bumps every plugin on merge, so everything
should start a PR on the same version number.

## Step 2: Create the skill (always)

```bash
GROUP="<group>"
NAME="<name>"
mkdir -p "plugins/$GROUP/skills/$NAME"
```

Write `plugins/$GROUP/skills/$NAME/SKILL.md`. The frontmatter `name` MUST equal the skill
directory name; the `description` should start with what the skill does and end with when to
use it (this is what each agent reads to decide relevance):

```markdown
---
name: <name>
description: <what it does>. Use when <when to use it>.
---

# <Title>

## When to use
- ...

## Workflow
1. ...
```

Keep the body platform-agnostic — no Claude-only, Codex-only, or Cursor-only assumptions —
so the same file works everywhere. Bundle any supporting files (references, schema, scripts)
inside the skill folder.

## Step 3: Create the plugin manifests (only for a NEW group)

Skip this if `plugins/<group>/` already existed. Otherwise create all three. They carry the
**same `name` (= group), `version`, and `description`** (describe the group, not one skill).

```bash
mkdir -p "plugins/$GROUP/.claude-plugin" \
         "plugins/$GROUP/.codex-plugin" \
         "plugins/$GROUP/.cursor-plugin"
```

`plugins/$GROUP/.claude-plugin/plugin.json` (minimal):

```json
{
  "name": "<group>",
  "description": "<group description>",
  "version": "<version>"
}
```

`plugins/$GROUP/.codex-plugin/plugin.json` and
`plugins/$GROUP/.cursor-plugin/plugin.json` (identical — add author + license):

```json
{
  "name": "<group>",
  "version": "<version>",
  "description": "<group description>",
  "author": "Infodom Team",
  "license": "MIT"
}
```

## Step 4: Register the plugin (only for a NEW group)

Skip this if the group plugin already existed. Otherwise append the same entry to the
`plugins` array in **both** `.claude-plugin/marketplace.json` **and**
`.cursor-plugin/marketplace.json`:

```json
{
  "name": "<group>",
  "source": "./plugins/<group>",
  "description": "<group description>",
  "version": "<version>",
  "author": { "name": "Infodom Team" },
  "repository": "https://github.com/infodomy/infodom-llm-marketplace",
  "license": "MIT",
  "category": "<category>",
  "tags": ["<tag>", "<tag>"]
}
```

Do NOT create a Codex marketplace manifest — Codex needs only the per-plugin
`.codex-plugin/plugin.json` from Step 3.

## Step 5: Update the README

Add a row to the **Available skills** table in `README.md`:

```
| `<group>` | `/<group>:<name>` | <short description> |
```

## Step 6: Validate and open a PR

Validate every JSON file you touched:

```bash
for f in .claude-plugin/marketplace.json .cursor-plugin/marketplace.json \
         plugins/$GROUP/.claude-plugin/plugin.json \
         plugins/$GROUP/.codex-plugin/plugin.json \
         plugins/$GROUP/.cursor-plugin/plugin.json; do
  [ -f "$f" ] && python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$f" && echo "OK  $f"
done
```

Then create a branch and open a PR. On merge, the release workflow bumps and version-syncs
every plugin's three manifests and both marketplace manifests — do not hand-bump versions.

## How each platform uses the result

- **Claude Code** — `/plugin install <group>@infodom-llm`; skill invoked as `/<group>:<name>`.
- **Codex CLI** — `codex plugin install <group>`; skill invoked as `/<name>` (no namespace).
- **Cursor** — install the plugin from the team marketplace; skill invoked as `/<name>`
  (no namespace).

## Checklist

- [ ] group chosen (existing plugin reused, or new one created)
- [ ] `plugins/<group>/skills/<name>/SKILL.md` with matching frontmatter `name`
- [ ] (new group only) `.claude-plugin/plugin.json` — minimal
- [ ] (new group only) `.codex-plugin/plugin.json` — with author + license
- [ ] (new group only) `.cursor-plugin/plugin.json` — with author + license
- [ ] (new group only) entry added to both marketplace manifests
- [ ] README skills table updated
- [ ] all JSON valid
