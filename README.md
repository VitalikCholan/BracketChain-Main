# BracketChain

 On-chain tournament protocol on Solana. Organizers create USDC-prize brackets.
 Players pay entry into a PDA-escrowed vault. Prize auto-distributes on final match — no off-chain custody.

 ## Repositories

 | Layer | Repo |
 |---|---|
 | Smart Contracts | [bracket-chain-programs](https://github.com/<username>/bracket-chain-programs) |
 | TypeScript SDK | [bracket-chain-sdk](https://github.com/<username>/bracket-chain-sdk) |
 | Indexer & API | [bracket-chain-backend](https://github.com/<username>/bracket-chain-backend) |
 | Web Application | [bracket-chain-frontend](https://github.com/<username>/bracket-chain-frontend) |

 ## Architecture

 Anchor program (Solana devnet)
     ↓  Helius Enhanced Webhooks
 NestJS indexer → PostgreSQL (Neon) [Railway]
     ↓  REST API + RPC fallback
 @bracket-chain/sdk (npm)
     ↓  React hooks (TanStack Query)
 Next.js 14 app [Vercel]

 ## Links

 - **App:** https://bracket-chain.vercel.app
 - **API:** https://api.bracket-chain.up.railway.app
 - **Program ID (devnet):** `TODO`
 - **Demo video:** `TODO`