# Custom Caddy Builder <small>(Prebuilt GitHub Action)</small>

[![CI](https://github.com/azolfagharj/action-caddy-builder/actions/workflows/ci.yml/badge.svg)](https://github.com/azolfagharj/action-caddy-builder/actions) [![Marketplace](https://img.shields.io/badge/Marketplace-Caddy%20Builder-blue?logo=github)](https://github.com/marketplace/actions/caddy-builder) [![Donate](https://img.shields.io/badge/Donate-to%20Keep%20This%20Project%20Alive-orange)](https://donate.azolfagharj.ir/)

Build custom [Caddy](https://github.com/caddyserver/caddy) binaries with [xcaddy](https://github.com/caddyserver/xcaddy) in GitHub Actions. Supports all OS/arch, custom modules, and integrates with your pipeline.

---

## Table of contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Step-by-step: Your first build](#step-by-step-your-first-build)
- [Concepts](#concepts)
- [Configuration reference](#configuration-reference)
- [Outputs](#outputs)
- [Supported platforms](#supported-platforms)
- [Common use cases](#common-use-cases)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Attribution](#attribution)
- [License](#license)
- [Contributing](#contributing)

---

## Overview

This action runs [xcaddy](https://github.com/caddyserver/xcaddy) inside a GitHub Actions workflow to build custom Caddy binaries. You can:

- Add Caddy modules (DNS providers, transports, etc.)
- Build for Linux, Windows, and macOS (x64 and arm64)
- Use native builds (each runner builds for itself) or cross-compile (one runner builds for all)
- Integrate the built binary into subsequent steps or jobs

No Go or xcaddy setup required—the action installs everything.

---

## Prerequisites

- A GitHub repository with Actions enabled
- A workflow file in `.github/workflows/`

---

## Installation

Available on [GitHub Marketplace](https://github.com/marketplace/actions/caddy-builder). Add the action to your workflow:

```yaml
- uses: azolfagharj/action-caddy-builder@v1
```

Use a specific version for reproducibility:

```yaml
- uses: azolfagharj/action-caddy-builder@v1.0.0
```

---

## Quick start

Create `.github/workflows/build.yml`:

```yaml
name: Build Caddy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azolfagharj/action-caddy-builder@v1
        id: caddy
      - run: ${{ steps.caddy.outputs.caddy-path }} version
```

Push to your repo. The workflow builds Caddy and prints the version.

---

## Step-by-step: Your first build

### Step 1: Create the workflow file

Create `.github/workflows/build.yml` in your repository.

### Step 2: Define when it runs

```yaml
name: Build Caddy
on: [push]
```

Runs on every push. You can change to `on: [push, pull_request]` or `on: release` as needed.

### Step 3: Add checkout and the action

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azolfagharj/action-caddy-builder@v1
        id: caddy
```

Give the step an `id` so you can reference its outputs.

### Step 4: Use the built binary

The action sets `caddy-path`—the full path to the binary:

```yaml
      - run: ${{ steps.caddy.outputs.caddy-path }} version
```

### Step 5: Add modules (optional)

```yaml
      - uses: azolfagharj/action-caddy-builder@v1
        id: caddy
        with:
          caddy-version: v2.10.2
          modules: |
            github.com/caddy-dns/cloudflare
            github.com/caddyserver/ntlm-transport@v0.1.1
```

---

## Concepts

### Native build vs cross-compile

- **Native build**: Each job runs on its target OS (e.g. `ubuntu-latest` → Linux amd64). Use a matrix of runners.
- **Cross-compile**: One runner builds for multiple targets. Set `goos` and `goarch` explicitly.

### Output filename

The binary is named by platform, e.g. `caddy_linux_amd64`, `caddy_windows_amd64.exe`, `caddy_darwin_arm64`.

### Using the binary in later steps

Reference `steps.<id>.outputs.caddy-path` in any step in the same job. For other jobs, upload the binary as an artifact and download it there.

### Modules format

- Comma-separated: `github.com/caddy-dns/cloudflare, github.com/caddyserver/ntlm-transport`
- Newline-separated (YAML multiline)
- With version: `module@v1.2.3`

---

## Configuration reference

### Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `caddy-version` | string | `latest` | Caddy version: tag (`v2.10.2`), branch, commit SHA, or `latest` |
| `modules` | string | `""` | Modules to include. Comma or newline separated. Supports `module@version` and `module=path` |
| `replaces` | string | `""` | xcaddy `--replace` directives. Comma or newline separated |
| `embeds` | string | `""` | xcaddy `--embed` paths. Use `alias:path` for aliased embeds |
| `output-dir` | string | `"."` | Directory for the built binary |
| `go-version` | string | `"1.22"` | Go version. Use `1.22` for Caddy v2.8.x |
| `goos` | string | `""` | Target OS: `linux`, `windows`, `darwin`. Empty = runner's OS |
| `goarch` | string | `""` | Target arch: `amd64`, `arm64`, `arm`. Empty = runner's arch |

### Examples

```yaml
# Pin Caddy version
caddy-version: v2.10.2

# Add modules
modules: |
  github.com/caddy-dns/cloudflare
  github.com/caddyserver/ntlm-transport@v0.1.1

# Cross-compile for Windows
goos: windows
goarch: amd64

# Caddy v2.8.x (requires go-version 1.22)
go-version: "1.22"
caddy-version: v2.8.4
```

---

## Outputs

| Output | Description |
|--------|-------------|
| `caddy-path` | Full path to the built binary. Use in later steps or upload-artifact |
| `caddy-name` | Filename (e.g. `caddy_linux_amd64`, `caddy_windows_amd64.exe`) |
| `goos` | Target OS (`linux`, `windows`, `darwin`) |
| `goarch` | Target arch (`amd64`, `arm64`, `arm`) |

### Usage

```yaml
- uses: azolfagharj/action-caddy-builder@v1
  id: build
- run: ${{ steps.build.outputs.caddy-path }} version
- uses: actions/upload-artifact@v4
  with:
    name: caddy
    path: ${{ steps.build.outputs.caddy-path }}
```

---

## Supported platforms

| OS | Architecture | Runner |
|----|--------------|--------|
| Linux | x64 | `ubuntu-latest`, `ubuntu-22.04`, `ubuntu-slim` |
| Linux | arm64 | `ubuntu-24.04-arm`, `ubuntu-22.04-arm` |
| Windows | x64 | `windows-latest`, `windows-2022` |
| Windows | arm64 | `windows-11-arm` |
| macOS | x64 (Intel) | `macos-14-large`, `macos-15-intel` |
| macOS | arm64 (M1/M2) | `macos-latest`, `macos-14`, `macos-15` |

> Some arm64 runners may require GitHub Team or Enterprise. See [GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners).

---

## Common use cases

### Basic build

| Use case | Example |
|----------|---------|
| Plain Caddy, no modules | [01-quick-start.yml](examples/01-quick-start.yml) |
| Add modules (Cloudflare DNS, NTLM) | [02-build-with-modules.yml](examples/02-build-with-modules.yml) |
| Modules comma-separated | [17-modules-comma-separated.yml](examples/17-modules-comma-separated.yml) |
| Specific Caddy version + OS/arch | [09-specific-version-platform.yml](examples/09-specific-version-platform.yml) |
| Caddy v2.8.x (go-version 1.22) | [14-build-caddy-v28.yml](examples/14-build-caddy-v28.yml) |
| Custom output directory | [06-custom-output-dir.yml](examples/06-custom-output-dir.yml) |
| Advanced (--replace, --embed) | [07-advanced-replaces-embeds.yml](examples/07-advanced-replaces-embeds.yml) |

### Single platform

| Use case | Example |
|----------|---------|
| Linux only | [13-build-single-platform.yml](examples/13-build-single-platform.yml) |
| Windows only | [13b-build-windows-only.yml](examples/13b-build-windows-only.yml) |
| macOS only | [13c-build-macos-only.yml](examples/13c-build-macos-only.yml) |

### Multi-platform

| Use case | Example |
|----------|---------|
| Matrix: Linux, Windows, macOS (native) | [04-matrix-all-platforms.yml](examples/04-matrix-all-platforms.yml) |
| Cross-compile from single runner | [05-cross-compile.yml](examples/05-cross-compile.yml) |
| Full matrix (arm64, Intel mac) + release | [18-build-and-release-full-matrix.yml](examples/18-build-and-release-full-matrix.yml) |

### Use binary in pipeline

| Use case | Example |
|----------|---------|
| Build and upload artifact | [03-build-and-upload.yml](examples/03-build-and-upload.yml) |
| Build and run with Caddyfile | [10-build-run-caddyfile.yml](examples/10-build-run-caddyfile.yml) |
| Build, validate config, list modules | [11-build-validate-config.yml](examples/11-build-validate-config.yml) |
| Build → upload → use in next job | [12-build-upload-use-next-job.yml](examples/12-build-upload-use-next-job.yml) |
| Build, use in multiple steps | [16-pipeline-multiple-tasks.yml](examples/16-pipeline-multiple-tasks.yml) |
| Build and create Docker image | [15-build-and-docker.yml](examples/15-build-and-docker.yml) |

### Release

| Use case | Example |
|----------|---------|
| Build and attach to release | [08-build-and-release.yml](examples/08-build-and-release.yml) |
| Full matrix + release | [18-build-and-release-full-matrix.yml](examples/18-build-and-release-full-matrix.yml) |

---

## Examples

All examples are in [`examples/`](examples/). Copy any file to `.github/workflows/` in your repo.

See [examples/README.md](examples/README.md) for the full list with descriptions.

---

## Troubleshooting

### Build fails with "undefined: zapslog.HandlerOptions"

Caddy v2.8.x has compatibility issues with Go 1.23. Use Go 1.22:

```yaml
with:
  go-version: "1.22"
  caddy-version: v2.8.4
```

Or use Caddy v2.10+ with the default Go version.

### Wrong architecture on ARM runners

The action detects `runner.arch`. If you see `amd64` on an ARM runner, ensure you use a supported ARM runner (e.g. `ubuntu-22.04-arm`). Some ARM runners require GitHub Team or Enterprise.

### Module not found in list-modules

Module IDs in `caddy list-modules` differ from Go import paths. For example, `github.com/caddyserver/ntlm-transport` appears as `http.reverse_proxy.transport.http_ntlm`. Use `grep` with a substring that matches.

### Binary not found in next job

Upload the binary as an artifact in the build job, then download it in the next job. See [12-build-upload-use-next-job.yml](examples/12-build-upload-use-next-job.yml).

---

## Attribution

This action uses the following projects. We credit them here in accordance with their licenses:

| Project | License | Use |
|---------|---------|-----|
| [Caddy](https://github.com/caddyserver/caddy) | [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0) | Web server (built by this action) |
| [xcaddy](https://github.com/caddyserver/xcaddy) | [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0) | Caddy build tool |
| [Go](https://go.dev/) | [BSD-3-Clause](https://go.dev/LICENSE) | Toolchain for xcaddy |
| [actions/checkout](https://github.com/actions/checkout) | [MIT](https://github.com/actions/checkout/blob/main/LICENSE) | Checkout repo |
| [actions/setup-go](https://github.com/actions/setup-go) | [MIT](https://github.com/actions/setup-go/blob/main/LICENSE) | Install Go |
| [actions/cache](https://github.com/actions/cache) | [MIT](https://github.com/actions/cache/blob/main/LICENSE) | Cache Go modules |

---

## License

MIT - see [LICENSE](LICENSE).

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to report issues.

[![Donate](https://img.shields.io/badge/Donate-Support%20this%20project-orange)](https://donate.azolfagharj.ir/)
