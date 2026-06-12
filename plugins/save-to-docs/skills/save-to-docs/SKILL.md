---
name: save-to-docs
description: Save information to the infodom-docs repository and open a GitHub PR with the changes.
---

# save-to-docs

Save information to the infodom-docs repository and open a GitHub PR with the changes.

**Input:** $ARGUMENTS

---

## Step 1 — Validate input

If `$ARGUMENTS` is empty, ask the user:

> What would you like to document? Please describe the topic and the information you want to save.

Wait for the user's response before proceeding.

---

## Step 2 — Explore infodom-docs

The infodom-docs repository is at `/home/epiflight/Desktop/startup/infodom/infodom-docs/`.

Read the top-level `README.md` and list the full directory tree (non-git, non-media files) so you understand the existing structure:

```bash
find /home/epiflight/Desktop/startup/infodom/infodom-docs \
  -not -path '*/.git/*' \
  -not -path '*/media/*' \
  \( -name "*.md" -o -type d \) | sort
```

The top-level sections are:
- `architecture/` — infrastructure diagrams, system design
- `business-logic/` — application workflows, business rules
- `development/` — dev workflows, deployment, tooling guides
- `google-analytics/` — GA configuration and setup
- `incident-response/` — runbooks, restore procedures
- `mailing-config/` — email configuration
- `VPN/` — VPN setup and access

---

## Step 3 — Determine placement

Analyse the user's input and compare it against the existing structure. Based on this, determine:

1. **Does this update an existing document?**
   - If yes: state which file you plan to update and what you will change.
   - Ask the user: *"I'd like to update `<file path>` with this information. Does that look right, or should it go somewhere else?"*
   - Wait for confirmation before proceeding.

2. **Does this require a new file inside an existing section?**
   - Propose the file path (e.g. `development/new-topic/README.md`).
   - Ask the user: *"I don't see existing docs for this. I'd like to create `<proposed path>`. Is that a good name and location, or would you prefer something different?"*
   - Wait for confirmation.

3. **Does this require a completely new top-level section?**
   - Propose the directory name (e.g. `monitoring/`).
   - Ask the user: *"This topic doesn't fit any existing section. Should I create a new top-level directory called `<name>/`? Or would you prefer a different name or to nest it inside an existing section?"*
   - Wait for confirmation.

4. **Does the input affect the root `README.md` table of contents?**
   - If you are creating a new section or sub-section that should be listed in the root README, tell the user and ask for approval to update it too.

Always be explicit about every file you intend to touch. Do not make assumptions silently.

---

## Step 4 — Draft the content

Write well-structured Markdown for the documentation. Follow existing doc conventions:

- Use `README.md` as the entry point for each directory.
- Use clear headings (`##`, `###`).
- Use code blocks for commands, configs, and code snippets.
- Use numbered lists for step-by-step procedures.
- Keep language concise and imperative for instructions.
- If the document references other repos, services, or tools, name them explicitly (e.g. `infodom-backend`, `kubectl`, `SOPS`).
- Do not add emojis unless they already appear in the target file.

Show the user the full draft content and ask:

> Here is the content I plan to write:
>
> **File:** `<path>`
> ```markdown
> <draft content>
> ```
>
> Does this look correct? Should I change anything before I proceed?

Wait for user confirmation. If they request edits, revise and show the updated draft again.

---

## Step 5 — Create a branch and commit

Once the user approves the content:

```bash
cd /home/epiflight/Desktop/startup/infodom/infodom-docs

# Ensure we're up to date
git checkout main
git pull origin main

# Create a new branch — use a short, descriptive slug derived from the topic
# e.g. "add-sops-rotation-guide", "update-vpn-setup", "add-redis-config-docs"
git checkout -b <branch-name>
```

Ask the user to confirm the branch name before creating it:

> I'll create branch `<branch-name>` in infodom-docs. Does that name look good?

Write or update the files, then:

```bash
git add <files>
git commit -m "<short imperative commit message>"
git push -u origin <branch-name>
```

---

## Step 6 — Open a GitHub PR

```bash
cd /home/epiflight/Desktop/startup/infodom/infodom-docs
gh pr create \
  --title "<PR title — short, descriptive>" \
  --body "$(cat <<'EOF'
## Summary
- <bullet: what was added or changed>
- <bullet: why>

## Files changed
- `<file path>` — <one-line description>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Return the PR URL to the user.

---

## Key rules

- **Never commit or push without explicit user approval** of the content and branch name.
- **Ask before touching any file** you were not explicitly instructed to change.
- **Ask before creating any new directory** — the user decides naming conventions.
- If you are ever unsure whether the information fits an existing doc, ask rather than guess.
- Do not update the root `README.md` table of contents unless the user approves it.
- The infodom-docs remote is `git@github.com:infodomy/infodom-docs` — always push there.
