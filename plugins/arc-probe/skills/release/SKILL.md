---
name: release
description: Build, package, and publish a new ARC Probe release to GitHub
argument-hint: <version>
---

# ARC Probe — Release

Build binaries, create a release zip, and publish to GitHub.

## Arguments

- `version` (required): Semantic version (e.g., `0.2.0`, `1.0.0`)

## Steps

### 1. Build release binaries

`TAURI_SIGNING_PRIVATE_KEY` must be set before building so the Tauri bundler can sign the updater payload. Set it as an environment variable or point to the key file:

```bash
export TAURI_SIGNING_PRIVATE_KEY_PATH=~/.tauri/arc-probe.key
```

Build everything (DLL + injector + CLI + Tauri GUI):

```bash
cd arc-probe
build.bat --gui
```

Verify all outputs exist in `build/`:
- `probe-shell.dll`
- `probe-inject.exe`
- `probe.exe`
- `arc-probe_<version>_x64-setup.exe` (NSIS installer + updater payload)
- `arc-probe_<version>_x64-setup.exe.sig` (updater signature)

### 2. Create the release directory

```bash
mkdir -p public-repo/releases/v<version>
cp build/probe.exe build/probe-inject.exe build/probe-shell.dll public-repo/releases/v<version>/
cp "build/arc-probe_<version>_x64-setup.exe" public-repo/releases/v<version>/
```

### 3. Create the release README

Create `public-repo/releases/v<version>/README.md` with:
- Version number and date
- Contents table (file names + sizes + purpose)
- Requirements (Windows 10/11 x64, admin privileges)
- Quick start commands (inject, ping, explore)
- Claude Code plugin install instructions
- CLI and injector flag reference
- Links to tutorials

Use the previous release README as a template — check `public-repo/releases/` for past versions.

### 4. Create the CLI zip

```bash
cd public-repo/releases/v<version>
powershell -NoProfile -Command "Compress-Archive -Path 'probe.exe','probe-inject.exe','probe-shell.dll','README.md' -DestinationPath '../arc-probe-v<version>-win-x64.zip' -Force"
```

### 5. Create the updater manifest (latest.json)

Generate the `latest.json` file that the Tauri updater expects:

```bash
cd arc-probe
SIG=$(cat "build/arc-probe_<version>_x64-setup.exe.sig")
DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
cat > build/latest.json <<EOF
{
  "version": "v<version>",
  "notes": "ARC Probe v<version>",
  "pub_date": "$DATE",
  "platforms": {
    "windows-x86_64": {
      "signature": "$SIG",
      "url": "https://github.com/vzco/arc-probe/releases/download/v<version>/arc-probe_<version>_x64-setup.exe"
    }
  }
}
EOF
```

### 6. Update version numbers

Update these files with the new version:
- `plugin/.claude-plugin/plugin.json` → `"version": "<version>"`
- `gui/src-tauri/tauri.conf.json` → `"version": "<version>"`
- `public-repo/.claude-plugin/marketplace.json` → plugin version field
- `public-repo/plugins/arc-probe/.claude-plugin/plugin.json` → `"version": "<version>"`

### 7. Sync plugin to public repo

```bash
# Remove old plugin files and copy fresh
rm -rf public-repo/plugins/arc-probe
cp -r plugin public-repo/plugins/arc-probe
```

### 8. Commit and push public repo

```bash
cd public-repo
git add -A
git commit -m "release: v<version>"
git push origin main
```

### 9. Create GitHub Release

```bash
cd public-repo
gh release create v<version> \
  "releases/arc-probe-v<version>-win-x64.zip" \
  "../build/arc-probe_<version>_x64-setup.exe" \
  "../build/latest.json" \
  --title "ARC Probe v<version>" \
  --notes-file "releases/v<version>/README.md"
```

### 10. Verify

- Check the release page: `gh release view v<version>`
- Download the zip and verify contents
- Download the installer and test it launches
- Test plugin install: `/plugin marketplace add vzco/arc-probe`
- Verify `latest.json` is accessible at the release asset URL (Tauri auto-updater uses this)

## Notes

- Always build release (not debug) for distribution
- The `.gitignore` in public-repo excludes `releases/`, `*.exe`, `*.dll`, `*.zip`
- Binaries are only distributed via GitHub Release assets, never committed to git
- Source code stays in the private repo — only plugin files and docs go public
- The Tauri signing private key lives at `~/.tauri/arc-probe.key` — this must NEVER be committed to any repository. For CI/CD, set `TAURI_SIGNING_PRIVATE_KEY` as a GitHub Actions secret.
