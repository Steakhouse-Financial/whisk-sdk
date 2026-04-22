# Whisk SDK

Type-safe TypeScript SDK for the [Whisk](https://whisk.so) GraphQL API, with a high-level integration for [Steakhouse](https://steakhouse.financial) vaults.

Use it to query ERC-4626 vault data (Morpho V1/V2, Box), APY histories, TVL, and the full Steakhouse vault registry with full TypeScript inference end-to-end.

## Documentation

- [Whisk API reference](https://www.docs.whisk.so)
- [Steakhouse SDK guide](https://steakhouse-sdk.vercel.app/)

## Architecture

Three layered packages. Pick the one that matches how much abstraction you want.

```
@whisk/graphql  →  @whisk/client  →  @whisk/steakhouse
   (schema)         (GraphQL core)    (Steakhouse SDK)
```

| Package | Purpose |
|---|---|
| [`@whisk/graphql`](./packages/graphql) | Whisk GraphQL schema + [`gql.tada`](https://gql-tada.0no.co/) setup. Custom scalars (`BigInt`, `Address`, `ChainId`, `Hex`, `URL`) and enums (`ApyTimeframe`, `Erc4626VaultProtocol`, `TokenCategory`). |
| [`@whisk/client`](./packages/client) | `WhiskClient` — the low-level GraphQL client built on [URQL](https://urql.dev). Handles auth (Bearer token), scalar conversion at runtime, and typed errors (`WhiskError`). Use this to run arbitrary queries against the Whisk schema. |
| [`@whisk/steakhouse`](./packages/steakhouse) | `SteakhouseClient` + prebuilt queries (`getVaults`, `getVault`, `getVaultHistory`, `getStats`), a Markdown-driven Steakhouse vault registry, and an optional React provider. This is the package most consumers want. |

## Installation

```bash
pnpm add @whisk/steakhouse @whisk/client graphql
```

If you're using React hooks, also install the peers:

```bash
pnpm add react react-dom @tanstack/react-query
```

You'll need a Whisk API key. Request one via the [Whisk docs](https://www.docs.whisk.so).

## Quick start

```typescript
import { SteakhouseClient, getVaults, getVault, getStats } from "@whisk/steakhouse"

const client = new SteakhouseClient({
  apiKey: process.env.WHISK_API_KEY!,
})

// All Steakhouse vaults across supported chains
const vaults = await getVaults(client)

// Detail for a single vault
const vault = await getVault(client, {
  chainId: 1,
  vaultAddress: "0xbeef0046fcab1de47e41fb75bb3dc4dfc94108e3",
})

// Global Steakhouse stats (TVL, unique depositors)
const stats = await getStats(client, { includeHistorical: true })
```

Full type inference on every response — no codegen step in your project.

## @whisk/steakhouse

### Queries

All four queries are async functions that take a `SteakhouseClient` and return typed data.

| Query | Returns | Notes |
|---|---|---|
| `getVaults(client)` | `SteakhouseVaultSummary[]` | All vaults registered in the SDK, enriched with Whisk live data (APY, TVL, risk). |
| `getVault(client, { chainId, vaultAddress, isBox? })` | `SteakhouseVaultDetail` | Single vault with protocol-specific fields — Morpho allocations, Box leverage and funding modules, etc. |
| `getVaultHistory(client, { chainId, vaultAddress })` | `{ daily, weekly } \| null` | Historical APY for Morpho vaults. Returns `null` for non-Morpho vaults. |
| `getStats(client, { includeHistorical? })` | `{ tvl, uniqueDepositors, historical? }` | Global Steakhouse TVL + depositor counts, broken down by chain / protocol / asset category. |

### Vault registry

The list of "what counts as a Steakhouse vault" is the source of truth for `getVaults`. It's built from Markdown files with YAML frontmatter at build time:

```
packages/steakhouse/src/metadata/vaults/{chain}/{vault}.md
  ↓  pnpm generate:vaults
packages/steakhouse/src/metadata/generated/vaults.ts   →  STEAKHOUSE_VAULTS
```

Each entry looks like:

```yaml
---
chainId: 1
vaultAddress: "0xbeef0046fcab1de47e41fb75bb3dc4dfc94108e3"
protocol: morpho_v2              # generic | morpho_v1 | morpho_v2 | box
name: "ETH Prime Instant"
strategy: Prime                   # Prime | High Yield | Turbo | Term
---
Optional Markdown description.
```

The registry is published at `@whisk/steakhouse/metadata`:

```typescript
import { STEAKHOUSE_VAULTS, getChainDeployments } from "@whisk/steakhouse/metadata"

getChainDeployments(1) // { boxFactory: "0xcF23...", vaults: [...] }
```

A companion test (`vault-verification.test.ts`) makes on-chain multicalls to confirm every registered vault is actually deployed by the Morpho factory and curated by Steakhouse. CI runs it on every PR.

### React

An optional provider + context hook live under `@whisk/steakhouse/react`:

```tsx
import { SteakhouseProvider, useSteakhouse } from "@whisk/steakhouse/react"

const client = new SteakhouseClient({ apiKey: "..." })

function App() {
  return (
    <SteakhouseProvider client={client}>
      <Vaults />
    </SteakhouseProvider>
  )
}

function Vaults() {
  const client = useSteakhouse()
  // Wrap query functions with your preferred data layer —
  // TanStack Query, SWR, or plain useEffect.
}
```

The SDK intentionally ships without prebuilt data-fetching hooks — you bring your own caching/refetch strategy (TanStack Query recommended).

## Supported networks

Steakhouse vaults are registered across: Ethereum mainnet, Base, Arbitrum, Polygon, Unichain, Monad, and Katana. The full current list lives in [`packages/steakhouse/src/metadata/vaults/`](./packages/steakhouse/src/metadata/vaults).

## Requirements

- **Node** ≥ 20
- **TypeScript** ≥ 5.5 (for full `gql.tada` inference)
- **React** ≥ 18 (optional — only if using `@whisk/steakhouse/react`)
- **GraphQL** ≥ 16 (peer dependency)

## Environment variables

For the Next.js example app and local development:

```bash
NEXT_PUBLIC_WHISK_API_KEY=xxx
NEXT_PUBLIC_WHISK_API_URL=https://staging.api-v2.whisk.so/graphql
```

## Development

This repo is a pnpm + Turborepo monorepo.

```bash
pnpm install
pnpm build               # Build all packages
pnpm test                # Vitest across packages
pnpm check               # Biome lint + format (auto-fix)
pnpm check:types         # Type check (tsgo)
pnpm check:graphql       # Validate queries against schema
pnpm dev:next            # Run the Next.js example app
```

### Schema sync

The Whisk schema is committed to the repo (not fetched at build, for security). When the upstream API changes:

```bash
pnpm schema:sync         # Fetch from staging
pnpm schema:sync:local   # Fetch from http://localhost:3500/graphql
```

This regenerates `packages/graphql/src/generated/{schema.json,schema.graphql,graphql-env.d.ts}` in one pass. Review the diff, commit, and push.

### Adding a query

1. Add a fragment to `packages/steakhouse/src/queries/fragments.ts` if you need one.
2. Create a query file in `packages/steakhouse/src/queries/` following the `getVaults.ts` pattern.
3. Export it from `packages/steakhouse/src/queries/index.ts`.

See `CLAUDE.md` and `AGENTS.md` at the repo root for more conventions (function style, imports, error handling, testing).

### Local development across repos

When developing the SDK alongside a local Whisk API and a consuming app:

```
Terminal 1:  whisk-api   → local API on :3500
Terminal 2:  whisk-sdk   → pnpm dev (tsup --watch)
Terminal 3:  your app    → pnpm dev (picks up SDK via pnpm overrides)
```

In the consuming app's `package.json`:

```jsonc
"pnpm": {
  "overrides": {
    "@whisk/graphql": "link:../whisk-sdk/packages/graphql",
    "@whisk/client": "link:../whisk-sdk/packages/client",
    "@whisk/steakhouse": "link:../whisk-sdk/packages/steakhouse"
  }
}
```

Remove the overrides before committing in the app repo.

## Releasing

Versions are managed by [Changesets](https://github.com/changesets/changesets):

```bash
pnpm changeset           # Describe your change (interactive)
pnpm changeset:version   # Bump versions + update changelogs
git add -A && git commit
```

CI runs `changeset:publish` on merge to `main` (see `.github/workflows/main.yml`).

## License

[MIT](./LICENSE) — © Office Supply Ventures LLC
