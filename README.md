# proto

Static [proto](https://moonrepo.dev/docs/proto) plugin definitions for third-party CLI tools that are not managed by proto core.

The plugin files live under `plugins/`. Each TOML file is a non-WASM proto plugin that describes how proto should resolve versions, download release archives, unpack the executable, and expose the executable through proto shims and bins.

## Tools

| Tool | Plugin file | Executable | Upstream project | Supported platforms |
| --- | --- | --- | --- | --- |
| `golang-migrate` | `plugins/golang-migrate.toml` | `migrate` | `golang-migrate/migrate` | Linux, macOS, and Windows on `x86_64` and `aarch64` |
| `skillshare` | `plugins/skillshare.toml` | `skillshare` | `runkids/skillshare` | Linux, macOS, and Windows on `x86_64` and `aarch64` |

## Requirements

- `proto` installed and activated in your shell.
- `git`, because these plugins resolve available versions from upstream Git tags.
- Archive tools supported by proto for unpacking `.tar.gz` and `.zip` releases.

See the official proto install guide if proto is not installed yet:

```sh
bash <(curl -fsSL https://moonrepo.dev/install/proto.sh)
```

## Enable the plugins

Add the plugins to a `.prototools` file in the repository that wants to use them.

Use the raw GitHub URLs for this repository:

```toml
golang-migrate = "<version>"
skillshare = "<version>"

[plugins.tools]
golang-migrate = "https://raw.githubusercontent.com/trypanic/proto/main/plugins/golang-migrate.toml"
skillshare = "https://raw.githubusercontent.com/trypanic/proto/main/plugins/skillshare.toml"
```

Replace:

- `<version>` with a release version from the upstream tool.

For example, when pinned in a project-level `.prototools`, running `proto install` from that project will install the configured tools.

## Install and run

Install everything pinned in the current `.prototools` context:

```sh
proto install
```

Install one tool explicitly:

```sh
proto install golang-migrate <version>
proto install skillshare <version>
```

Run through proto:

```sh
proto run golang-migrate -- -version
proto run skillshare -- --help
```

After proto shims are active in `PATH`, the primary executable can also be called directly:

```sh
migrate -version
skillshare --help
```

## CI validation

Pushes to `main` run GitHub Actions validation for the checked-out plugin definitions:

- macOS `x86_64` on `macos-15-intel`
- macOS `aarch64` on `macos-15`
- Windows `x86_64` on `windows-2025`
- Windows `aarch64` on `windows-11-arm`

Each job installs proto, creates a temporary `.prototools` that points to the checked-out TOML plugin files under `plugins/`, installs `golang-migrate` and `skillshare`, and verifies the installed binaries.

## Plugin details

### golang-migrate

`plugins/golang-migrate.toml` installs the `migrate` binary from GitHub release archives:

- Linux: `migrate.linux-{arch}.tar.gz`
- macOS: `migrate.darwin-{arch}.tar.gz`
- Windows: `migrate.windows-{arch}.zip`

The plugin maps proto/Rust architecture names to the release archive naming used by `golang-migrate`:

| proto arch | Release arch |
| --- | --- |
| `x86_64` | `amd64` |
| `aarch64` | `arm64` |

The plugin resolves available versions from Git tags in `https://github.com/golang-migrate/migrate` and downloads archives from:

```text
https://github.com/golang-migrate/migrate/releases/download/v{version}/{download_file}
```

### skillshare

`plugins/skillshare.toml` installs the `skillshare` binary from GitHub release archives:

- Linux: `skillshare_{version}_linux_{arch}.tar.gz`
- macOS: `skillshare_{version}_darwin_{arch}.tar.gz`
- Windows: `skillshare_{version}_windows_{arch}.zip`

The plugin maps proto/Rust architecture names to the release archive naming used by `skillshare`:

| proto arch | Release arch |
| --- | --- |
| `x86_64` | `amd64` |
| `aarch64` | `arm64` |

The plugin resolves available versions from Git tags in `https://github.com/runkids/skillshare` and downloads archives from:

```text
https://github.com/runkids/skillshare/releases/download/v{version}/{download_file}
```

## Development notes

- Keep these plugins static and declarative unless a tool needs custom logic. Simple CLI archive installs fit proto's non-WASM plugin format.
- Add a `[platform.*]` entry only when the upstream project publishes a matching archive.
- Keep `[install.exes.<name>] primary = true` aligned with the executable users should run by default.
- If upstream release archive names use different architecture labels, map them under `[install.arch]`.
- If upstream tags do not parse cleanly as proto versions, add a `version-pattern` under `[resolve]`.

## References

- [proto documentation](https://moonrepo.dev/docs/proto)
- [proto plugins](https://moonrepo.dev/docs/proto/plugins)
- [proto non-WASM plugins](https://moonrepo.dev/docs/proto/non-wasm-plugin)
- [proto configuration](https://moonrepo.dev/docs/proto/config)
