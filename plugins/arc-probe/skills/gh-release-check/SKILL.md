---
name: gh-release-check
description: Check the status of GitHub releases — list versions, verify assets, compare with local builds
---

# ARC Probe — GitHub Release Check

Verify the state of published releases and compare with local builds.

## Steps

### 1. List published releases

```bash
cd public-repo
gh release list
```

### 2. Check latest release details

```bash
gh release view <tag>
```

Look for:
- Release title and tag
- Assets (zip file should be present)
- Download count
- Published date

### 3. Verify asset contents

```bash
# Download and inspect
gh release download <tag> -D /tmp/arc-probe-check
cd /tmp/arc-probe-check
powershell -NoProfile -Command "Expand-Archive -Path '*.zip' -DestinationPath './extracted'"
ls -lh extracted/
```

Should contain: `probe.exe`, `probe-inject.exe`, `probe-shell.dll`, `README.md`

### 4. Compare with local build

```bash
# Check if local binaries are newer
ls -lh build/probe.exe build/probe-inject.exe build/probe-shell.dll
```

If local files are newer than the release, consider a new release.

### 5. Check plugin version alignment

```bash
# Plugin version in repo
grep '"version"' plugin/.claude-plugin/plugin.json

# Marketplace version
grep -A2 '"arc-probe"' public-repo/.claude-plugin/marketplace.json | grep version

# Latest release tag
gh release list --limit 1
```

All three should match. If they don't, run `/arc-probe:bump-version` and `/arc-probe:release`.
