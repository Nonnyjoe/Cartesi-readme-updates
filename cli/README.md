<br>
<p align="center">
    <img src="docs/images/cartesi-logo.png" align="center" width="20%" alt="Cartesi">
</p>
<br>

# Cartesi CLI

[![npm alpha](https://img.shields.io/npm/v/@cartesi/cli/alpha?label=npm%20alpha)](https://www.npmjs.com/package/@cartesi/cli)
[![CLI](https://github.com/cartesi/cli/actions/workflows/cli.yaml/badge.svg)](https://github.com/cartesi/cli/actions/workflows/cli.yaml)
[![License](https://img.shields.io/github/license/cartesi/cli)](LICENSE)

Create, build, run, and interact with [Cartesi](https://cartesi.io) applications from your terminal.

<details>
<summary><strong>Table of contents</strong> — click to expand</summary>

<br>

- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Installation](#installation)
- [Commands](#commands)
- [Configuration](#configuration)
- [Local environment](#local-environment)
- [Templates](#templates)
- [Apps and packages](#apps-and-packages)
- [Development](#development)
- [Documentation](#documentation)
- [Related projects](#related-projects)
- [Community and support](#community-and-support)
- [Contributing](#contributing)
- [License](#license)

</details>

## Quick start

Install the CLI, verify your development environment, and create an application:

```sh
npm install -g @cartesi/cli@alpha
cartesi doctor
cartesi create hello --template python
cd hello
cartesi build
cartesi run --services explorer
```

The `run` command prints the selected base URL and the URLs for Anvil, JSON-RPC, Inspect, and the optional explorer. Keep it running and, in another terminal, interact with the application:

```sh
cartesi send "hello world" --encoding string
cartesi status
cartesi address-book
```

## Requirements

- Docker Desktop, or Docker Engine 25.0.0 or later on Linux
- Docker Compose 2.24.0 or later
- Docker Buildx 0.13.0 or later
- `linux/riscv64` emulation for Docker builds
- Node.js 20 or later when installing through npm

Enable RISC-V emulation if it is not already available:

```sh
docker run --privileged --rm tonistiigi/binfmt:riscv
```

Run `cartesi doctor` after installation to check Docker, Compose, Buildx, and RISC-V support.

## Installation

### npm

Install the current prerelease from the npm alpha channel:

```sh
npm install -g @cartesi/cli@alpha
```

### Standalone binaries

Native binaries do not require Node.js. Download the archive for your platform from [GitHub Releases](https://github.com/cartesi/cli/releases):

- `cartesi-darwin-arm64.tar.gz`
- `cartesi-darwin-x64.tar.gz`
- `cartesi-linux-arm64.tar.gz`
- `cartesi-linux-x64.tar.gz`

Extract the archive, rename the executable to `cartesi`, and move it to a directory on your `PATH`. The archives preserve the executable bit; if necessary, restore it with `chmod +x cartesi`. On macOS, a verified download may also require removing the quarantine attribute with `xattr -d com.apple.quarantine cartesi`.

Verify the installation:

```sh
cartesi --version
cartesi doctor
```

## Commands

| Command                               | Purpose                                                              |
| ------------------------------------- | -------------------------------------------------------------------- |
| `create <name> --template <template>` | Scaffold an application from a template                              |
| `build`                               | Build the configured drives and create a Cartesi Machine snapshot    |
| `run`                                 | Start the local environment and deploy an existing machine snapshot  |
| `send [input]`                        | Send an input to the application                                     |
| `deposit ether`                       | Deposit Ether into the application                                   |
| `deposit erc20`                       | Deposit ERC-20 tokens into the application                           |
| `deposit erc721`                      | Deposit an ERC-721 token into the application                        |
| `deposit erc1155-single`              | Deposit one ERC-1155 token type into the application                 |
| `deposit erc1155-batch`               | Deposit multiple ERC-1155 token types into the application           |
| `status`                              | Show the status of the local environment                             |
| `logs`                                | Show logs from the local Rollups Node; use `--follow` to stream them |
| `address-book`                        | Print deployed contract addresses                                    |
| `hash`                                | Print the machine template hash                                      |
| `shell`                               | Open a shell inside the application machine                          |
| `clean`                               | Delete cached build artifacts                                        |
| `doctor`                              | Check local prerequisites                                            |

The `deploy` command is deprecated in 2.0. Use `cartesi-rollups-cli` directly when deploying to a public network.

Run `cartesi <command> --help` for the complete, current option set. The hosted [CLI command reference](https://docs.cartesi.io/get-started/cli-commands/) provides a general overview.

## Configuration

Applications can be configured with `cartesi.toml`. Without one, the CLI uses a Docker-backed root drive, 128 MiB of machine RAM, and the other default machine settings. The following example increases the RAM to 256 MiB while explicitly retaining the default root-drive configuration:

```toml
[machine]
ram_length = "256Mi"

[drives.root]
builder = "docker"
dockerfile = "Dockerfile"
format = "ext2"
```

Multiple configuration files can be supplied with repeated `--config` options. Later files extend or override values from earlier files.

Useful `run` options include:

| Option                            | Purpose                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------ |
| `--prt`                           | Deploy the application with PRT consensus                                      |
| `--block-time <seconds>`          | Set the local chain's block interval                                           |
| `--epoch-length <blocks>`         | Set the Rollups epoch length                                                   |
| `--claim-staging-period <blocks>` | Set the claim-staging period for Authority or Quorum consensus                 |
| `--fork-url <url>`                | Fork an existing Ethereum network                                              |
| `--fork-block-number <number>`    | Fork at a specific block                                                       |
| `--default-block <mode>`          | Use `latest`, `safe`, `pending`, or `finalized` as the default block for reads |
| `--services <services>`           | Start optional comma-separated services                                        |
| `--port <number>`                 | Select the proxy port                                                          |
| `--dry-run`                       | Print the generated configuration without starting it                          |

Set supported `CARTESI_*` variables in the host environment before starting `cartesi run`; the CLI forwards the allow-listed values to the node. List the supported variables with:

```sh
cartesi run --list-supported-variables
```

## Local environment

`cartesi build` prepares the configured drives and boots the resulting RISC-V Cartesi Machine once to create a reproducible machine snapshot. Docker-backed drives are built for `linux/riscv64` with Docker Buildx.

`cartesi run` generates and starts a Docker Compose environment for the application. If a machine snapshot is available, the command deploys and registers the application automatically. Image versions are pinned by each CLI release instead of being selected implicitly from `latest` tags.

The core environment includes:

- an Anvil development chain;
- the Cartesi Rollups runtime and node services;
- PostgreSQL;
- JSON-RPC and Inspect endpoints; and
- a Traefik proxy providing a single local entry point.

Optional services are enabled through `--services`:

| Service     | Description                                         |
| ----------- | --------------------------------------------------- |
| `explorer`  | Web interface for inspecting local Rollups activity |
| `bundler`   | ERC-4337 bundler                                    |
| `paymaster` | Local verifying paymaster                           |
| `passkey`   | Passkey-related local service                       |
| `all`       | Start all optional services                         |

For example:

```sh
cartesi run --services explorer
cartesi run --services bundler,paymaster,passkey
```

The proxy prefers port `6751` and selects the next available port when necessary. Always use the endpoint URLs printed by `cartesi run`.

## Templates

| Template                   | Language                  |
| -------------------------- | ------------------------- |
| `cpp`, `cpp-low-level`     | C++                       |
| `go`                       | Go                        |
| `java`                     | Java                      |
| `javascript`, `typescript` | JavaScript and TypeScript |
| `lua`                      | Lua                       |
| `python`                   | Python                    |
| `ruby`                     | Ruby                      |
| `rust`                     | Rust                      |

Templates are maintained in [cartesi/application-templates](https://github.com/cartesi/application-templates).

## Apps and packages

The repository is organized as a Bun monorepo containing the CLI application and the packages used to build and run its local environment.

| Path                                | Package                             | Purpose                                                                                                                     |
| ----------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `apps/cli`                          | `@cartesi/cli`                      | Commander-based CLI distributed through npm and as native Linux and macOS binaries                                          |
| `packages/sdk`                      | `@cartesi/sdk`                      | Docker image and supporting scripts for building and executing Cartesi applications                                         |
| `packages/devnet`                   | `@cartesi/devnet`                   | Foundry deployment scripts, Cartesi contracts, test token contracts, and the initial Anvil state used by local environments |
| `packages/mock-verifying-paymaster` | `@cartesi/mock-verifying-paymaster` | Mock ERC-4337 verifying paymaster used by optional account-abstraction services                                             |
| `packages/tsconfig`                 | `tsconfig`                          | Shared TypeScript configurations used throughout the monorepo                                                               |

## Development

This repository uses [Bun](https://bun.sh/), [Turborepo](https://turbo.build/repo), and [Biome](https://biomejs.dev/).

Install dependencies and run the main checks:

```sh
bun install
bun run build
bun run lint
bun run test
```

The complete workspace build requires Docker and Foundry. CLI integration tests may start containers and pull the required images. The filtered CLI build below still builds its `@cartesi/devnet` dependency and therefore also requires Foundry, although it avoids the SDK Docker-image build.

Useful focused commands include:

```sh
bun run build --filter @cartesi/cli
bun test apps/cli/
bun test apps/cli/tests/unit/
```

Releases are managed with [Changesets](https://github.com/changesets/changesets). Merging a Changesets release pull request publishes the affected packages to npm. When `@cartesi/cli` is published, native Linux and macOS binaries for x64 and arm64 are uploaded to the corresponding GitHub release. Homebrew updates are maintained separately.

## Documentation

- [Cartesi documentation](https://docs.cartesi.io/)
- [CLI command reference](https://docs.cartesi.io/get-started/cli-commands/)
- [CLI changelog](apps/cli/CHANGELOG.md)

Individual packages may contain additional README files under `apps/` and `packages/`.

## Related projects

- [Cartesi Rollups Node](https://github.com/cartesi/rollups-node)
- [Application templates](https://github.com/cartesi/application-templates)
- [Rollups Explorer](https://github.com/cartesi/rollups-explorer)
- [Cartesi Machine Emulator](https://github.com/cartesi/machine-emulator)

## Community and support

For help, join the [Cartesi Discord](https://discord.gg/cartesi). For reproducible bugs and feature requests, open a [GitHub issue](https://github.com/cartesi/cli/issues).

When reporting a CLI problem, include relevant environment information from:

```sh
cartesi doctor
cartesi status --json
```

Review the output and remove sensitive information before posting it publicly.

## Contributing

Issues and pull requests are welcome. Keep changes focused, include tests for changed behavior, and update the relevant documentation or changelog information when necessary.

## License

Licensed under the [Apache License 2.0](LICENSE).
