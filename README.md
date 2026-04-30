# BracketChain

On-chain tournament protocol on Solana. Organizers create USDC-prize brackets.
Players pay entry into a PDA-escrowed vault. Prize auto-distributes on final match — no off-chain custody.

## Repositories

| Layer | Repo |
|---|---|
| Smart Contracts | [BracketChain-Programs](https://github.com/VitalikCholan/BracketChain-Programs) |
| TypeScript SDK | [BracketChain-Sdk](https://github.com/VitalikCholan/BracketChain-Sdk) |
| Indexer & API | [BracketChain-Backend](In progress) |
| Web Application | [BracketChain-Frontend](https://github.com/btcthirst/BracketChain-Frontend) |

## Architecture

```
Anchor program (Solana devnet)
    ↓  Helius Enhanced Webhooks
NestJS indexer → PostgreSQL (Neon) [Railway]
    ↓  REST API + RPC fallback
@bracket-chain/sdk (npm)
    ↓  React hooks (TanStack Query)
Next.js 14 app [Vercel]
```

## Links

- **App:** `TODO`
- **API:** `TODO`
- **Program ID (devnet):** `TODO`
- **Demo video:** `TODO`
