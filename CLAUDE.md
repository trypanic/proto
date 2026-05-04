# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Static [proto](https://moonrepo.dev/docs/proto) non-WASM plugin definitions for third-party CLI tools. Each top-level `<tool-id>.toml` is a standalone plugin consumed from other repositories via raw GitHub URLs in their `.prototools` `[plugins.tools]` table. Tool ID = TOML filename without extension (`golang-migrate`, `skillshare`, `staticcheck`).

Changes here propagate to every downstream consumer pinned at `main`. Prefer small, verifiable edits.

## Plugin Authoring Rules

- Keep plugins static and declarative. Use `type = "cli"` for archive-based installs; only reach for custom logic if the tool genuinely needs it.
- Add a `[platform.<os>]` entry only when the upstream project actually publishes that archive — verify against the tool's GitHub releases page before adding/removing.
- `archs` use proto/Rust names (`x86_64`, `aarch64`). Map to upstream archive labels (`amd64`, `arm64`, etc.) under `[install.arch]`.
- Keep `[install.exes.<name>] primary = true` aligned with the executable users should run.
- If upstream Git tags don't parse cleanly as proto versions, add `version-pattern` under `[resolve]`.
- When supported platforms change, update the plugin TOML, README support matrix + plugin-details section, and all CI workflows together.

## Local Validation

Validate against an isolated proto home so user state doesn't pollute the result:

```sh
repo_root="$(pwd)"
tmpdir="$(mktemp -d)"
mkdir -p "$tmpdir/user-home"
cat > "$tmpdir/.prototools" <<EOF
golang-migrate = "4.19.0"
skillshare = "0.19.5"
staticcheck = "2026.1.0"

[plugins.tools]
golang-migrate = "file://$repo_root/plugins/golang-migrate.toml"
skillshare = "file://$repo_root/plugins/skillshare.toml"
staticcheck = "file://$repo_root/plugins/staticcheck.toml"
EOF

cd "$tmpdir"
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install golang-migrate --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install skillshare --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install staticcheck --config-mode local

"$tmpdir/.proto/tools/golang-migrate/4.19.0/migrate" -version    # → 4.19.0
"$tmpdir/.proto/tools/skillshare/0.19.5/skillshare" --version    # → skillshare v0.19.5
"$tmpdir/.proto/tools/staticcheck/2026.1.0/staticcheck/staticcheck" -version    # → staticcheck 2026.1 (v0.7.0)
```

Use explicit tool versions, never `latest` — validation must be reproducible. Staticcheck's upstream `2026.1` tag is pinned as `2026.1.0` because Proto requires fully qualified versions.

## CI

Three workflows under `.github/workflows/` run on push to `main`: `proto-linux.yml` (Linux x86_64 + aarch64), `proto-macos.yml` (macOS x86_64 + aarch64), and `proto-windows.yml` (Windows x86_64 + aarch64). Each writes a temporary `.prototools` referencing `file://./plugins/<plugin>.toml` (the checked-out commit, not raw `main` URLs), runs `proto install --config-mode local`, then asserts each supported binary's version output. Staticcheck is skipped on Windows aarch64 because upstream does not publish that release archive.

When adding a plugin, update all workflows so the new tool is installed and verified.

## Adding a New Plugin

1. Confirm upstream publishes versioned release archives for every target OS+arch.
2. Create `plugins/<tool-id>.toml` with `type = "cli"`, `[platform.*]`, `[install]` (+ `[install.arch]` if labels differ), `[install.exes.*]`, `[resolve]`.
3. Validate locally as above.
4. Add the tool to all CI workflows.
5. Update `README.md` (support matrix, install/run examples, plugin-details section).

## README Sync

`README.md` is the public consumer-facing source of truth. Update it whenever any consumer-facing behavior changes: tools added/removed, tool IDs, executable names, raw URLs, supported platforms/archs, usage commands, or CI coverage.
