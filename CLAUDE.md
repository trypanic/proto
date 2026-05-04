# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Static [proto](https://moonrepo.dev/docs/proto) non-WASM plugin definitions for third-party CLI tools. Each top-level `<tool-id>.toml` is a standalone plugin consumed from other repositories via raw GitHub URLs in their `.prototools` `[plugins.tools]` table. Tool ID = TOML filename without extension (`golang-migrate`, `skillshare`).

Changes here propagate to every downstream consumer pinned at `main`. Prefer small, verifiable edits.

## Plugin Authoring Rules

- Keep plugins static and declarative. Use `type = "cli"` for archive-based installs; only reach for custom logic if the tool genuinely needs it.
- Add a `[platform.<os>]` entry only when the upstream project actually publishes that archive — verify against the tool's GitHub releases page before adding/removing.
- `archs` use proto/Rust names (`x86_64`, `aarch64`). Map to upstream archive labels (`amd64`, `arm64`, etc.) under `[install.arch]`.
- Keep `[install.exes.<name>] primary = true` aligned with the executable users should run.
- If upstream Git tags don't parse cleanly as proto versions, add `version-pattern` under `[resolve]`.
- When supported platforms change, update the plugin TOML, README support matrix + plugin-details section, and both CI workflows together.

## Local Validation

Validate against an isolated proto home so user state doesn't pollute the result:

```sh
tmpdir="$(mktemp -d)"
mkdir -p "$tmpdir/home"
cat > "$tmpdir/.prototools" <<EOF
golang-migrate = "4.19.0"
skillshare = "0.19.5"

[plugins.tools]
golang-migrate = "file://$(pwd)/golang-migrate.toml"
skillshare = "file://$(pwd)/skillshare.toml"
EOF

cd "$tmpdir"
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/home" proto install golang-migrate --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/home" proto install skillshare --config-mode local

"$tmpdir/.proto/tools/golang-migrate/4.19.0/migrate" -version    # → 4.19.0
"$tmpdir/.proto/tools/skillshare/0.19.5/skillshare" --version    # → skillshare v0.19.5
```

Use explicit upstream versions, never `latest` — validation must be reproducible.

## CI

Two workflows under `.github/workflows/` run on push to `main`: `proto-plugins-macos.yml` (macOS x86_64 + aarch64) and `proto-plugins-windows.yml` (Windows x86_64 + aarch64). Each writes a temporary `.prototools` referencing `file://./<plugin>.toml` (the checked-out commit, not raw `main` URLs), runs `proto install --config-mode local`, then asserts each binary's version output.

When adding a plugin, update both workflows so the new tool is installed and verified.

## Adding a New Plugin

1. Confirm upstream publishes versioned release archives for every target OS+arch.
2. Create `<tool-id>.toml` at repo root with `type = "cli"`, `[platform.*]`, `[install]` (+ `[install.arch]` if labels differ), `[install.exes.*]`, `[resolve]`.
3. Validate locally as above.
4. Add the tool to both CI workflows.
5. Update `README.md` (support matrix, install/run examples, plugin-details section).

## README Sync

`README.md` is the public consumer-facing source of truth. Update it whenever any consumer-facing behavior changes: tools added/removed, tool IDs, executable names, raw URLs, supported platforms/archs, usage commands, or CI coverage.
