---
name: pr-description
description: Create, improve, and update GitHub Pull Request descriptions for the current branch or a specified PR number.
---

# PR Description

## When to use

Use this skill when the user asks to:
- create a PR description
- rewrite an existing PR description
- update a PR description after new commits

## Workflow

1. Identify the target PR.
2. Gather change context from git.
3. Draft a structured PR body.
4. Show the draft to the user (unless user asked for direct update).
5. Update the PR description with `gh pr edit`.

## Step 1: Identify PR

- If user provided PR number, use it.
- Otherwise resolve PR from current branch:

```bash
gh pr view --json number,title,body,baseRefName,headRefName
```

## Step 2: Gather context

Use focused context, not the whole repo history:

```bash
git diff --stat origin/$(gh pr view --json baseRefName --jq '.baseRefName')...HEAD
git log --oneline origin/$(gh pr view --json baseRefName --jq '.baseRefName')...HEAD
```

If needed, inspect changed files:

```bash
gh pr diff --name-only
```

## Step 3: Draft format

Use this structure:

```md
## Summary
- ...

## What Changed
- ...

## Testing
- ...

## Risks / Notes
- ...
```

Rules:
- Be explicit and concrete.
- Mention migrations, feature flags, and API contract changes when relevant.
- Keep it short unless user asks for more detail.

## Step 4: Update PR

Write draft body to a temporary file, then run:

```bash
gh pr edit <PR_NUMBER> --body-file <BODY_FILE>
```

## Guardrails

- If `gh` is not available, stop and ask the user whether to install it first or continue without it.
- Do not invent tests or results.
- Do not claim rollout/migration steps that are not in code.
- If `gh` is available but not authenticated, stop and ask the user to authenticate.
