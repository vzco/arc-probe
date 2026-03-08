---
name: publish-readme
description: Update the public-repo README with latest features, skills, and screenshots
---

# ARC Probe — Publish README

Update the public-facing README on github.com/vzco/arc-probe with the latest information.

## When to use

After adding new skills, commands, features, or screenshots that should be reflected in the public README.

## Steps

### 1. Read the current public README

```bash
cat public-repo/README.md
```

### 2. Check for new content to add

- New skills in `plugin/skills/` that aren't in the README skill table
- New commands added to the probe DLL
- New screenshots in `public-repo/docs/images/`
- Updated tutorials

### 3. Update the README

Key sections to keep current:
- **Quick start** code examples
- **Installation** section with skill table (must list ALL skills)
- **Command reference** tables
- **Screenshots** (add new ones if available)
- **Status** line (update if moving from early access to stable, etc.)

### 4. Do NOT add

- Source code paths or architecture details that reveal private repo structure
- Build instructions (binaries are pre-built)
- Internal implementation details
- Developer-only information

### 5. Commit and push

```bash
cd public-repo
git add README.md
git commit -m "docs: update README"
git push origin main
```

## Style guide

The public README uses:
- Lowercase headings
- No emoji
- Short, direct sentences
- Code blocks for all commands
- Tables for structured data
- `>` blockquotes for status/notes
