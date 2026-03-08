---
name: bump-version
description: Bump the version number across all plugin and marketplace config files
argument-hint: <new_version>
---

# ARC Probe — Bump Version

Update the version string in all config files that reference it.

## Arguments

- `new_version` (required): New semantic version (e.g., `0.2.0`)

## Files to update

1. **`plugin/.claude-plugin/plugin.json`** — `"version": "<new_version>"`
2. **`public-repo/.claude-plugin/marketplace.json`** — plugin entry version
3. **`public-repo/plugins/arc-probe/.claude-plugin/plugin.json`** — `"version": "<new_version>"`

## Steps

1. Read each file
2. Replace the old version string with the new one
3. Verify all three files show the same version
4. Do NOT commit — this is typically done as part of a release or sync

## Verification

```bash
grep -r '"version"' plugin/.claude-plugin/plugin.json public-repo/.claude-plugin/marketplace.json public-repo/plugins/arc-probe/.claude-plugin/plugin.json
```

All three should show `"version": "<new_version>"`.
