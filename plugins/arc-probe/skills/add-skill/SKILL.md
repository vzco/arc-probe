---
name: add-skill
description: Create a new skill for the ARC Probe plugin with proper structure and frontmatter
argument-hint: <skill-name> <description>
---

# ARC Probe — Add Skill

Create a new skill in the plugin with the correct directory structure, frontmatter, and registration.

## Arguments

- `skill-name` (required): kebab-case name for the skill (e.g., `scan-vtables`)
- `description` (required): One-line description of what the skill does

## Steps

### 1. Create the skill directory and file

```bash
mkdir -p plugin/skills/<skill-name>
```

### 2. Write the SKILL.md

Every skill MUST start with YAML frontmatter:

```markdown
---
name: <skill-name>
description: <description>
argument-hint: <args if applicable>
---

# Skill Title

Description of what this skill does and when to use it.

## Arguments

- `arg1` (required): What it is
- `arg2` (optional): What it is

## Steps

### 1. First step

Detailed instructions with example commands.

### 2. Second step

...

## Tips

- Helpful notes
- Edge cases
```

### 3. Verify frontmatter

```bash
head -4 plugin/skills/<skill-name>/SKILL.md
```

Must show `---`, `name:`, `description:`, `---`.

### 4. Update skill count

Update these if they reference the skill count:
- `plugin/.claude-plugin/plugin.json` description
- `public-repo/README.md` skill table
- `public-repo/plugins/arc-probe/.claude-plugin/plugin.json` description

### 5. Sync to public repo

After adding the skill, run `/arc-probe:sync-plugin` to copy it to the public repo.

## Naming conventions

- Use kebab-case: `map-struct`, `find-function`, `trace-writes`
- Prefix with `probe-` for skills that interact with the GUI bridge
- Keep names short but descriptive
- Skills are invoked as `/arc-probe:<skill-name>`

## Frontmatter fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | kebab-case skill identifier |
| `description` | yes | One-line description shown in skill list |
| `argument-hint` | no | Shows expected arguments (e.g., `<address> [size]`) |
| `disable-model-invocation` | no | Set to `true` to prevent auto-invocation |
