---
name: sync-plugin
description: Sync plugin skills and agent from private repo to public repo for distribution
---

# ARC Probe — Sync Plugin to Public Repo

Copy the latest plugin files from the private repo to the public repo for distribution.

## When to use

After modifying any skill, agent, or plugin config in the private `plugin/` directory and wanting to publish the changes.

## Steps

### 1. Verify plugin structure

Check that all skills have YAML frontmatter (`---\nname: ...\ndescription: ...\n---`):

```bash
for f in plugin/skills/*/SKILL.md; do
  if ! head -1 "$f" | grep -q '^---'; then
    echo "MISSING FRONTMATTER: $f"
  fi
done
```

### 2. Remove old public plugin and copy fresh

```bash
rm -rf public-repo/plugins/arc-probe
cp -r plugin public-repo/plugins/arc-probe
```

### 3. Verify no source code leaked

```bash
# Should only show: .claude-plugin/, CLAUDE.md, skills/, agents/
find public-repo/plugins/arc-probe -type f | sort
```

No `.cpp`, `.hpp`, `.c`, `.h`, `.rs`, `.ts`, `.json` (except plugin.json) files should be present.

### 4. Sync version numbers

Make sure the version in these files matches:
- `plugin/.claude-plugin/plugin.json`
- `public-repo/.claude-plugin/marketplace.json`
- `public-repo/plugins/arc-probe/.claude-plugin/plugin.json`

### 5. Commit and push

```bash
cd public-repo
git add plugins/
git commit -m "plugin: sync skills and agent"
git push origin main
```

## Notes

- The `.mcp.json` should NOT exist in the plugin (removed — no MCP)
- Agent file should be flat: `agents/re-assistant.md` (not `agents/re-assistant/agent.md`)
- Skills are namespaced as `/arc-probe:<skill-name>` when installed
