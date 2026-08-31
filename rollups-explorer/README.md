<br>
<p align="center">
    <img src="docs/images/cartesi-logo.png" align="center" width="20%" alt="Cartesi">
</p>
<br>

# Cartesi Rollups Explorer

[![CI](https://github.com/cartesi/rollups-explorer/actions/workflows/ci.yml/badge.svg)](https://github.com/cartesi/rollups-explorer/actions/workflows/ci.yml)
[![Coverage](https://coveralls.io/repos/github/cartesi/rollups-explorer/badge.svg?branch=main)](https://coveralls.io/github/cartesi/rollups-explorer?branch=main)
[![Latest release](https://img.shields.io/github/v/release/cartesi/rollups-explorer?include_prereleases)](https://github.com/cartesi/rollups-explorer/releases)
[![Docker pulls](https://img.shields.io/docker/pulls/cartesi/rollups-explorer?logo=docker&label=Docker%20pulls)](https://hub.docker.com/r/cartesi/rollups-explorer/tags)
[![License](https://img.shields.io/github/license/cartesi/rollups-explorer)](LICENSE)

Cartesi Rollups Explorer is a browser-based interface for discovering and interacting with applications served by a Cartesi Rollups Node. It reads application state directly from the node's JSON-RPC API and uses the connected wallet to submit inputs, deposit assets, and execute outputs on the base layer.

<details>
<summary><strong>Table of contents</strong> — click to expand</summary>

<br>

- [Overview](#overview)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [How it works](#how-it-works)
- [Supported networks](#supported-networks)
- [Build and run locally with Docker](#build-and-run-locally-with-docker)
- [Development](#development)
- [Testing and quality checks](#testing-and-quality-checks)
- [Apps and packages](#apps-and-packages)
- [Releases](#releases)
- [Contributing](#contributing)
- [License](#license)
- [Related projects and support](#related-projects-and-support)

</details>

## Overview

The explorer can:

- connect to a compatible Cartesi Rollups Node and list the applications it serves;
- browse inputs, reports, notices, vouchers, delegate-call vouchers, epochs, and withdrawals;
- submit application inputs through the on-chain `InputBox`;
- deposit Ether, ERC-20, ERC-721, and ERC-1155 assets through the Cartesi Portals;
- inspect decoded output data and execute eligible vouchers with a connected wallet;
- save node connections and ABI-based decoding specifications in the browser; and
- visualize tournaments and matches for applications using PRT consensus.

The current 2.0 explorer accepts Rollups Node versions in the `^2.0.0-alpha.12` range (`>=2.0.0-alpha.12 <3.0.0`). When establishing a connection, the explorer validates the version and chain ID reported by the Rollups Node. It checks that the configured Ethereum RPC responds, but it does not independently verify that the RPC uses the same chain.

## Quick start

The fastest way to run Explorer 2.0 is with an explicitly versioned Docker image. You need:

- a compatible Cartesi Rollups Node JSON-RPC endpoint;
- an Ethereum JSON-RPC endpoint for the same chain; and
- Docker when running the published container.

```sh
docker run --rm --name cartesi-rollups-explorer \
  -p 3000:3000 \
  -e CARTESI_NODE_RPC_URL=http://127.0.0.1:6751/rpc \
  -e NODE_RPC_URL=http://127.0.0.1:6751/anvil \
  cartesi/rollups-explorer:2.0.0-alpha.3
```

Open [http://localhost:3000/explorer](http://localhost:3000/explorer).

These RPC URLs are served to the browser, which contacts them directly. They must therefore be reachable from the user's browser. The loopback examples above work when the node and browser are running on the same host.

Use an explicit 2.x image tag. An unqualified `cartesi/rollups-explorer` image may follow a different release line. Available versions are published on [Docker Hub](https://hub.docker.com/r/cartesi/rollups-explorer/tags) and [GitHub Container Registry](https://github.com/cartesi/rollups-explorer/pkgs/container/rollups-explorer).

## Configuration

The published container is built to serve the application below `/explorer` and exposes a runtime configuration endpoint. Server-side variables take precedence over defaults embedded when the image was built.

### Runtime container variables

| Variable                 | Purpose                                                                          | Published-image default       |
| ------------------------ | -------------------------------------------------------------------------------- | ----------------------------- |
| `CARTESI_NODE_RPC_URL`   | Default Rollups Node v2 JSON-RPC endpoint                                        | `http://127.0.0.1:6751/rpc`   |
| `NODE_RPC_URL`           | Base-layer Ethereum RPC endpoint used for wallet interactions and contract reads | `http://127.0.0.1:6751/anvil` |
| `DEBUG_ENABLED`          | Enable debugging actions in the interface                                        | `false`                       |
| `HEALTHZ_UPTIME_ENABLED` | Include process uptime in the health response                                    | `true`                        |

The application exposes its health endpoint at `/explorer/api/healthz` in the published container.

> [!IMPORTANT]
> RPC configuration is delivered to client browsers. Do not place secrets in these URLs, and make sure the endpoints have appropriate access controls and browser-compatible CORS settings.

### Build-time variables

The following values are embedded by Next.js and cannot be reliably changed by passing new values to an already-built container:

| Variable                           | Purpose                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------- |
| `BASE_PATH`                        | Next.js base path used when building the application                      |
| `NEXT_PUBLIC_BASE_PATH`            | Client-visible form of the same base path                                 |
| `NEXT_PUBLIC_CARTESI_NODE_RPC_URL` | Build-time fallback Rollups Node JSON-RPC endpoint                        |
| `NEXT_PUBLIC_NODE_RPC_URL`         | Build-time fallback Ethereum RPC endpoint                                 |
| `NEXT_PUBLIC_DEBUG_ENABLED`        | Enable client-side debugging actions in the built application             |
| `NEXT_PUBLIC_MOCK_ENABLED`         | Bundle and activate the in-memory fixture data for development or testing |
| `NEXT_PUBLIC_PROJECT_ID`           | WalletConnect Cloud project ID                                            |
| `NEXT_PUBLIC_IS_CONTAINER`         | Enables loading runtime configuration from `/api/config`                  |

Mock fixtures must be enabled at build time with `NEXT_PUBLIC_MOCK_ENABLED=true`; setting `MOCK_ENABLED` only at container runtime does not populate the client-side fixture cache.

The Docker build argument `ENVIRONMENT=cartesi` selects `apps/explorer/.env.cartesi`, which sets the base path to `/explorer` and enables container runtime configuration.

## How it works

The Explorer 2.0 application does not require a separate explorer database or GraphQL backend.

| Flow                   | Path                                                                                    |
| ---------------------- | --------------------------------------------------------------------------------------- |
| Read application state | Browser → Rollups Node JSON-RPC API                                                     |
| Send an input          | Browser wallet → base-layer `InputBox` contract                                         |
| Deposit an asset       | Browser wallet → the corresponding Cartesi Portal contract                              |
| Execute an output      | Browser wallet → application contract, using output and proof data returned by the node |
| Inspect PRT activity   | Browser → node data for epochs, tournaments, commitments, and matches                   |

Saved node connections and custom decoding specifications are stored locally in the browser with IndexedDB.

## Supported networks

The explorer recognizes the following networks and chain IDs:

| Network          |   Chain ID |
| ---------------- | ---------: |
| Ethereum         |        `1` |
| Sepolia          | `11155111` |
| Base             |     `8453` |
| Base Sepolia     |    `84532` |
| OP Mainnet       |       `10` |
| OP Sepolia       | `11155420` |
| Arbitrum One     |    `42161` |
| Arbitrum Sepolia |   `421614` |
| Foundry          |    `31337` |
| Cannon           |    `13370` |

The configured Rollups Node and Ethereum RPC endpoints must refer to the same chain.

## Build and run locally with Docker

The reference Compose stack uses a locally built `cartesi/rollups-explorer:devel` image and a Traefik proxy:

```sh
docker buildx bake
docker compose -f docker/docker-compose.yaml up
```

Open [http://localhost:8080/explorer](http://localhost:8080/explorer). Stop and remove the stack with:

```sh
docker compose -f docker/docker-compose.yaml down
```

`docker buildx bake` builds an image for the host platform and tags it `cartesi/rollups-explorer:devel` through `docker-bake.override.hcl`. Release CI additionally loads `docker-bake.platforms.hcl` to publish Linux images for amd64 and arm64. See [docs/DOCKER.md](docs/DOCKER.md) for the maintained Docker workflow.

## Development

### Prerequisites

- Node.js 24.19.0 or later, but earlier than Node.js 25
- pnpm 11.21.0

Enable Corepack so it can activate the pnpm version declared in `package.json`:

```sh
corepack enable
```

Install exactly from the committed lockfile:

```sh
pnpm install --frozen-lockfile
```

Create `apps/explorer/.env.local` for developer-specific values. At minimum, configure the Rollups Node endpoint and provide a WalletConnect project ID when testing wallet connections:

```dotenv
NEXT_PUBLIC_CARTESI_NODE_RPC_URL=http://127.0.0.1:10011/rpc
NEXT_PUBLIC_NODE_RPC_URL=http://127.0.0.1:6751/anvil
NEXT_PUBLIC_PROJECT_ID=YOUR_WALLETCONNECT_PROJECT_ID
```

Start only the new explorer application:

```sh
pnpm dev --filter=explorer
```

The development server is available at [http://localhost:3000](http://localhost:3000). Unlike the published container, the default development build is served from `/`.

Run Storybook for component development:

```sh
pnpm storybook --filter=explorer
```

This repository uses a Turborepo workspace. Remove `--filter=explorer` only when you intentionally want a task to run across all participating applications and packages. See [docs/MONOREPO.md](docs/MONOREPO.md) for workspace operations.

### Dependency safeguards

The pnpm workspace enables several supply-chain controls:

- lifecycle and build scripts must be explicitly approved through `allowBuilds`;
- `strictDepBuilds` fails installation when a dependency requests an unapproved build;
- dependencies must normally have been published for at least seven days;
- `trustPolicy: no-downgrade` rejects weaker publishing-trust evidence; and
- exotic transitive sources such as Git repositories and direct tarball URLs are blocked.

Keep `pnpm-lock.yaml` committed and use `--frozen-lockfile` in reproducible environments.

## Testing and quality checks

Explorer unit and component tests use [Vitest](https://vitest.dev/) and [Testing Library](https://testing-library.com/docs/react-testing-library/intro/).

```sh
pnpm format:check
pnpm lint --filter=explorer
pnpm check:types --filter=explorer
pnpm test --filter=explorer
pnpm test:ci --filter=explorer
pnpm build --filter=explorer
```

`test:ci` generates coverage. The repository's Playwright end-to-end suite currently targets the legacy `apps/web` CartesiScan application, so it is not part of the Explorer 2.0 development loop.

## Apps and packages

### Explorer 2.0 and shared configuration

| Path                             | Purpose                                                                                                                   |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `apps/explorer`                  | Next.js Explorer 2.0 application, including node connections, wallet interactions, output decoding, and PRT visualization |
| `packages/eslint-config-cartesi` | Shared ESLint configuration                                                                                               |
| `packages/tsconfig`              | Shared TypeScript configuration                                                                                           |

### Legacy CartesiScan

The repository also contains CartesiScan, which is being phased out in favor of Explorer 2.0:

| Path                     | Purpose                                               |
| ------------------------ | ----------------------------------------------------- |
| `apps/web`               | Legacy CartesiScan Next.js application                |
| `apps/workshop`          | Storybook workspace for legacy shared components      |
| `packages/domain`        | Generated GraphQL types and functions for CartesiScan |
| `packages/rollups-wagmi` | Generated wagmi bindings for Rollups contracts        |
| `packages/ui`            | Shared CartesiScan UI components                      |

See [docs/CARTESISCAN.md](docs/CARTESISCAN.md). The legacy GraphQL backend is maintained separately in [cartesi/rollups-explorer-api](https://github.com/cartesi/rollups-explorer-api).

## Releases

Releases use `v`-prefixed semantic-version tags. A matching tag builds multi-platform images and publishes them to:

- [GitHub Container Registry](https://github.com/cartesi/rollups-explorer/pkgs/container/rollups-explorer); and
- [Docker Hub](https://hub.docker.com/r/cartesi/rollups-explorer/tags).

GitHub Container Registry images are also built for pull requests, while Docker Hub publishing is limited to version tags. See [docs/RELEASE.md](docs/RELEASE.md) and the [Docker workflow](.github/workflows/docker.yaml).

## Related projects and support

- [Cartesi Rollups Node](https://github.com/cartesi/rollups-node)
- [Cartesi CLI](https://github.com/cartesi/cli)
- [Cartesi documentation](https://docs.cartesi.io/)
- [GitHub issues](https://github.com/cartesi/rollups-explorer/issues)
- [Cartesi Discord](https://discord.gg/Pt2NrnS)

## Contributing

Contributions are welcome. Before opening a pull request, read [CONTRIBUTING.md](CONTRIBUTING.md) for the Contributor License Agreement, branch naming, and Conventional Commits requirements. All contributors must follow the [Code of Conduct](CODE_OF_CONDUCT.md).

For substantial changes, discuss the proposal with the Cartesi community on [Discord](https://discord.gg/Pt2NrnS) before implementation.

## License

Licensed under the [Apache License 2.0](LICENSE).
