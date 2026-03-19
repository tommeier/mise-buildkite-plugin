# mise Buildkite Plugin

Install [mise](https://mise.jdx.dev/), run `mise install`, and export the tool environment into the Buildkite step.

This is a fork of [buildkite-plugins/mise-buildkite-plugin](https://github.com/buildkite-plugins/mise-buildkite-plugin) with additional features for monorepo and custom agent image workflows:

- **Selective tool install** — install only the tools a step needs
- **Pre-compiled tool symlinks** — reuse tools baked into custom agent images
- **Checksum verification** — SHA256 integrity check on the downloaded binary
- **Base image binary seeding** — seed mise from `/usr/local/bin/mise` when cached binary is missing

## Example

```yml
steps:
  - label: ":wrench: Test"
    plugins:
      - tommeier/mise#v1.2.0:
          version: 2026.2.7
    command: go test ./...
```

## Monorepo with Selective Tools

Only install the tools a step needs, not everything in the config:

```yml
steps:
  - label: ":elixir: Backend Test"
    plugins:
      - tommeier/mise#v1.2.0:
          version: 2026.2.7
          dir: apps/backend
          tools:
            - erlang
            - elixir
    command: mix test

  - label: ":react: Native Lint"
    plugins:
      - tommeier/mise#v1.2.0:
          version: 2026.2.7
          dir: apps/native
          tools:
            - node
            - pnpm
    command: pnpm eslint .
```

## Pre-compiled Tools from Custom Agent Images

When using custom Buildkite agent images with pre-compiled tools (e.g., Ruby compiled into `/opt/mise/installs/`), symlink them to avoid recompilation:

```yml
steps:
  - label: ":wrench: Test"
    plugins:
      - tommeier/mise#v1.2.0:
          version: 2026.2.7
          image-installs-dir: /opt/mise/installs
    command: mix test
```

Compiled binaries bake the install prefix into the binary. Symlinking (not copying) preserves the original paths so tools work without recompilation.

## Checksum Verification

Verify the integrity of the downloaded mise binary:

```yml
steps:
  - label: ":wrench: Test"
    plugins:
      - tommeier/mise#v1.2.0:
          version: 2026.2.7
          checksum: "d23400ea220cfcbcd04856cf224dc6bc1a5e2a16e6c7365a433b7737c0774c23"
    command: mix test
```

Uses `sha256sum` (Linux) or `shasum -a 256` (macOS). Fails the step on mismatch.

## Hosted Agent Cache Volumes

```yml
cache: ".buildkite/cache-volume"

steps:
  - label: ":wrench: Test"
    plugins:
      - tommeier/mise#v1.2.0: ~
    command: go test ./...
```

When running on Buildkite hosted agents, the plugin automatically uses `/cache/bkcache/mise` as `MISE_DATA_DIR` if a cache volume is attached. Override the root with `MISE_HOSTED_CACHE_VOLUME_ROOT` (e.g., `/tmp/bkcache`).

## Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `version` | `latest` | mise version to install |
| `dir` | checkout dir | Directory where `mise install` and `mise env` run |
| `cache-dir` | unset | Directory for `MISE_DATA_DIR` (persistent caches on self-hosted agents) |
| `tools` | unset | Specific tools to install (array). When omitted, all tools in config are installed |
| `checksum` | unset | SHA256 checksum of the mise binary archive |
| `image-installs-dir` | unset | Directory with pre-compiled tool installs to symlink |

## Base Image Binary Seeding

When the cached mise binary is missing and `/usr/local/bin/mise` exists (e.g., baked into a custom agent image), the plugin copies it to the cache directory instead of downloading. This avoids a network request on first run with pre-built images.

## Repo Requirements

The target directory must contain one of:

- `mise.toml`
- `.mise.toml`
- `.tool-versions`

`MISE_DATA_DIR` still takes precedence over plugin configuration.

## Development

Run plugin checks locally:

```bash
mise install
docker run --rm -v "$PWD:/plugin" -w /plugin buildkite/plugin-linter --id tommeier/mise --path /plugin
docker run --rm -v "$PWD:/plugin" -w /plugin buildkite/plugin-tester
"$(mise where shellcheck@0.11.0)/shellcheck-v0.11.0/shellcheck" hooks/pre-command tests/pre-command.bats
```
