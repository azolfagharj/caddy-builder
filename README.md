# Caddy Builder

[![CI](https://github.com/azolfagharj/caddy-builder/actions/workflows/ci.yml/badge.svg)](https://github.com/azolfagharj/caddy-builder/actions)

A GitHub Action that builds custom Caddy binaries using [xcaddy](https://github.com/caddyserver/xcaddy). Supports all GitHub-hosted runner operating systems and CPU architectures. Configure extra modules, replacements, and embedded files—just like xcaddy on the command line.

Releases are created automatically when you push to `main`. Update `version` in [action.yml](action.yml) and push—CI will create the release if the tag does not exist.

## Features

- **Multi-platform**: Linux (x64, arm64), Windows (x64, arm64), macOS (Intel, Apple Silicon)
- **xcaddy integration**: Uses pinned xcaddy v0.4.5 for reproducible builds
- **Module support**: Add Caddy modules via `--with` (supports `module@version` and `module=path`)
- **Replace directives**: Use `--replace` for dependency replacement
- **Embed support**: Embed files and directories with `--embed` (aliased paths supported)
- **Output naming**: Automatic filenames like `caddy_linux_amd64`, `caddy_windows_arm64.exe`
- **Platform override**: Optional `goos`/`goarch` inputs for cross-compilation from any runner
- **Composite action**: Simple step in your workflow, compose with `upload-artifact` as needed

## Requirements

None. The action installs Go and xcaddy automatically.

## Usage

### Basic (plain Caddy, no modules)

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: azolfagharj/caddy-builder@v1
    id: caddy
  - run: ${{ steps.caddy.outputs.caddy-path }} version
```

### With modules

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: azolfagharj/caddy-builder@v1
    id: caddy
    with:
      caddy-version: v2.8.4
      modules: |
        github.com/caddy-dns/cloudflare
        github.com/caddyserver/ntlm-transport@v0.1.1
  - run: ${{ steps.caddy.outputs.caddy-path }} list-modules
```

### Matrix build (all platforms, upload artifacts)

```yaml
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: ubuntu-latest
            goos: linux
            goarch: amd64
          - os: ubuntu-22.04-arm
            goos: linux
            goarch: arm64
          - os: windows-latest
            goos: windows
            goarch: amd64
          - os: windows-11-arm
            goos: windows
            goarch: arm64
          - os: macos-latest
            goos: darwin
            goarch: arm64
          - os: macos-15-intel
            goos: darwin
            goarch: amd64
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: azolfagharj/caddy-builder@v1
        id: build
        with:
          caddy-version: v2.8.4
          modules: |
            github.com/caddy-dns/cloudflare
      - uses: actions/upload-artifact@v4
        with:
          name: caddy-${{ matrix.goos }}-${{ matrix.goarch }}
          path: ${{ steps.build.outputs.caddy-path }}
```

### Specify target platform (cross-compile)

Build for a different OS/arch from any runner:

```yaml
- uses: azolfagharj/caddy-builder@v1
  id: caddy
  with:
    goos: windows
    goarch: amd64
    modules: github.com/caddy-dns/cloudflare
# Result: caddy_windows_amd64.exe (even when run on ubuntu-latest)
```

### With replaces and embeds

```yaml
- uses: azolfagharj/caddy-builder@v1
  id: caddy
  with:
    caddy-version: v2.8.4
    modules: github.com/caddy-dns/cloudflare
    replaces: golang.org/x/net=../net
    embeds: |
      static:./public
      config:./config
```

## Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `caddy-version` | string | `latest` | Caddy version: tag (e.g. `v2.8.4`), branch, commit, or `latest` |
| `modules` | string | `""` | Modules to include. Comma or newline separated. Supports `module@version` and `module=path` |
| `replaces` | string | `""` | Replace directives for xcaddy `--replace`. Comma or newline separated |
| `embeds` | string | `""` | Paths for xcaddy `--embed`. Use `alias:path` for aliased embeds |
| `output-dir` | string | `"."` | Output directory for the built binary |
| `go-version` | string | `"1.22"` | Go version. Use `1.22` for Caddy v2.8.x (avoids zap/slog compatibility issues) |
| `goos` | string | `""` | Target OS: `linux`, `windows`, `darwin`. Empty = runner's OS |
| `goarch` | string | `""` | Target arch: `amd64`, `arm64`, `arm`. Empty = runner's arch |

## Outputs

| Output | Description |
|--------|-------------|
| `caddy-path` | Full path to the built Caddy binary |
| `caddy-name` | Output filename (e.g. `caddy_linux_amd64`, `caddy_windows_amd64.exe`) |
| `goos` | GOOS value for the runner (`linux`, `windows`, `darwin`) |
| `goarch` | GOARCH value for the runner (`amd64`, `arm64`, `arm`) |

## Supported platforms

Use the appropriate runner for your target OS and architecture:

| OS | Architecture | Runner |
|----|---------------|--------|
| Linux | x64 | `ubuntu-latest`, `ubuntu-22.04`, `ubuntu-slim` |
| Linux | arm64 | `ubuntu-24.04-arm`, `ubuntu-22.04-arm` |
| Windows | x64 | `windows-latest`, `windows-2022` |
| Windows | arm64 | `windows-11-arm` |
| macOS | x64 (Intel) | `macos-14-large`, `macos-15-intel` |
| macOS | arm64 (M1/M2) | `macos-latest`, `macos-14`, `macos-15` |

> **Note**: Some arm64 runners (e.g. `ubuntu-22.04-arm`, `windows-11-arm`) may require GitHub Team or Enterprise for full availability. Check [GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners) for current availability.

## Examples

### Build and run Caddy in the same job

```yaml
- uses: azolfagharj/caddy-builder@v1
  id: caddy
  with:
    modules: github.com/caddy-dns/cloudflare
- run: |
    ${{ steps.caddy.outputs.caddy-path }} run --config Caddyfile
```

### Build multiple platforms from one job (cross-compile)

Run on a single runner, build for all targets:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - goos: linux
            goarch: amd64
          - goos: windows
            goarch: amd64
          - goos: darwin
            goarch: arm64
    runs-on: ubuntu-latest
    steps:
      - uses: azolfagharj/caddy-builder@v1
        id: build
        with:
          goos: ${{ matrix.goos }}
          goarch: ${{ matrix.goarch }}
          modules: github.com/caddy-dns/cloudflare
      - uses: actions/upload-artifact@v4
        with:
          name: caddy-${{ matrix.goos }}-${{ matrix.goarch }}
          path: ${{ steps.build.outputs.caddy-path }}
```

### Build to a custom directory

```yaml
- uses: azolfagharj/caddy-builder@v1
  with:
    output-dir: ./dist
    modules: github.com/caddy-dns/cloudflare
```

### Create a release with multiple binaries

Combine the matrix build job above with `softprops/action-gh-release` to upload all artifacts as release assets. See [Creating a Release](https://docs.github.com/en/actions/publishing-packages/releasing-packages-to-github-actions-marketplace) for details.

## License

MIT - see [LICENSE](LICENSE) for details.
