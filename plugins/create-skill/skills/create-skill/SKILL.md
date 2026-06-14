---
name: create-skill
description: Scaffold a new plugin/skill in the infodom-llm marketplace, wired up for Claude Code, Codex CLI, and Cursor. Use when the user wants to add a new skill, plugin, or command to this marketplace.
---

# Create Skill

Scaffold a new plugin in this marketplace so a single `SKILL.md` works across
**Claude Code**, **OpenAI Codex CLI**, and **Cursor**. The three platforms share the
skill body and differ only in their per-plugin manifest and how the marketplace lists them.

## When to use

Use this skill when the user asks to:
- add a new skill / plugin / command to this marketplace
- scaffold a skill that works in Claude Code, Codex, and Cursor
- "create a new skill called X"

## Key concept: one skill, three configs

Each plugin lives under `plugins/<name>/` and ships **one shared skill** plus **three
manifests** — one per platform. Both marketplace manifests at the repo root must also list
the plugin.

```
.claude-plugin/marketplace.json    ← Claude Code marketplace (lists every plugin)
.cursor-plugin/marketplace.json    ← Cursor marketplace (lists every plugin)

plugins/<name>/
  .claude-plugin/plugin.json       ← Claude Code manifest
  .codex-plugin/plugin.json        ← Codex CLI manifest
  .cursor-plugin/plugin.json       ← Cursor manifest
  skills/<name>/SKILL.md           ← shared by all three platforms
```

> Codex has no marketplace manifest of its own — it discovers plugins from the repo layout,
> so only the per-plugin `.codex-plugin/plugin.json` is needed for Codex.

## Workflow

1. Gather inputs from the user.
2. Create the plugin directory and the shared `SKILL.md`.
3. Write the three per-plugin manifests.
4. Register the plugin in both marketplace manifests.
5. Update the README skills table.
6. Validate, then open a PR.

## Step 1: Gather inputs

Ask the user (use the Ask tool) for:
- **name** — kebab-case, unique, used for the directory, manifests, and skill (e.g. `deploy-helper`).
- **description** — one sentence; reused verbatim in all manifests and the marketplace entries.
- **category** — e.g. `developer-tools`, `documentation`, `database`, `observability`.
- **tags** — a few keywords for the marketplace listing.

Pick the version to match the other plugins currently in `.claude-plugin/marketplace.json`
(read the `version` field there). The release workflow bumps every plugin on merge, so all
plugins should start a PR on the same version number.

## Step 2: Create directory + SKILL.md

```bash
NAME="<name>"
mkdir -p "plugins/$NAME/.claude-plugin" \
         "plugins/$NAME/.codex-plugin" \
         "plugins/$NAME/.cursor-plugin" \
         "plugins/$NAME/skills/$NAME"
```

Write `plugins/$NAME/skills/$NAME/SKILL.md`. The frontmatter `name` MUST equal the directory
name; the `description` should start with what the skill does and end with when to use it
(this is what each agent reads to decide relevance):

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
so the same file works everywhere.

## Step 3: Write the three per-plugin manifests

All three carry the **same `name`, `version`, and `description`**. They differ only in shape.

`plugins/$NAME/.claude-plugin/plugin.json` (minimal):

```json
{
  "name": "<name>",
  "description": "<description>",
  "version": "<version>"
}
```

`plugins/$NAME/.codex-plugin/plugin.json` and
`plugins/$NAME/.cursor-plugin/plugin.json` (identical — add author + license):

```json
{
  "name": "<name>",
  "version": "<version>",
  "description": "<description>",
  "author": "Infodom Team",
  "license": "MIT"
}
```

## Step 4: Register in both marketplace manifests

Append the same plugin entry to the `plugins` array in **both**
`.claude-plugin/marketplace.json` **and** `.cursor-plugin/marketplace.json`:

```json
{
  "name": "<name>",
  "source": "./plugins/<name>",
  "description": "<description>",
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
| `<name>` | `/<name>:<name>` | <short description> |
```

## Step 6: Validate and open a PR

Validate every JSON file:

```bash
for f in .claude-plugin/marketplace.json .cursor-plugin/marketplace.json \
         plugins/$NAME/.claude-plugin/plugin.json \
         plugins/$NAME/.codex-plugin/plugin.json \
         plugins/$NAME/.cursor-plugin/plugin.json; do
  python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$f" && echo "OK  $f"
done
```

Then create a branch and open a PR (see the `pr-description` skill for the PR body).
On merge, the release workflow bumps and version-syncs the new plugin's three manifests
and both marketplace manifests automatically — do not hand-bump versions in the PR.

## How each platform uses the result

- **Claude Code** — `/plugin install <name>@infodom-llm`; skill invoked as `/<name>:<name>`.
- **Codex CLI** — `codex plugin install <name>`; skill invoked as `/<name>` (no namespace).
- **Cursor** — install from the team marketplace (Import from Repo); skill invoked as
  `/<name>` (no namespace).

## Checklist

- [ ] `plugins/<name>/skills/<name>/SKILL.md` with matching frontmatter `name`
- [ ] `.claude-plugin/plugin.json` (minimal)
- [ ] `.codex-plugin/plugin.json` (with author + license)
- [ ] `.cursor-plugin/plugin.json` (with author + license)
- [ ] entry added to `.claude-plugin/marketplace.json`
- [ ] entry added to `.cursor-plugin/marketplace.json`
- [ ] README skills table updated
- [ ] all JSON valid
