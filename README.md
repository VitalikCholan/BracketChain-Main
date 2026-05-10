# BracketChain

> Trustless tournament escrow on Solana. Organizers run brackets; players join by paying entry into a PDA-escrowed vault; prizes auto-distribute on the final reported match. No off-chain custody, no payout delays, no fragmented tooling.

**Hackathon MVP** — devnet only.

---

## Live links

| What | Where |
|---|---|
| Web app | [`https://bracketchain.vercel.app`](https://bracketchain.vercel.app) |
| Indexer / REST API | [`https://bracketchain-indexer-production.up.railway.app`](https://bracketchain-indexer-production.up.railway.app) |
| API health | [`/health`](https://bracketchain-indexer-production.up.railway.app/health) |
| Program (devnet) | [`AuXJKpuZtkegs2ZSgopgckhN7Ev8bUz4zBc238LD2F1`](https://explorer.solana.com/address/AuXJKpuZtkegs2ZSgopgckhN7Ev8bUz4zBc238LD2F1?cluster=devnet) |
| SDK on npm | [`@bracketchain/sdk@0.3.0`](https://www.npmjs.com/package/@bracketchain/sdk) |

---

## The use case (one paragraph)

An organizer creates a tournament with a chosen entry fee, max participants, and payout preset (Winner-Takes-All / Standard 60-25-15 / Deep 40-25-15-10-5-3-2). Players join by paying the entry fee into a PDA-escrowed vault. The organizer reports each match winner. On the final report, prizes auto-distribute on-chain per the chosen preset — placements receive 96.5% of the pool, the protocol receives 3.5%, all in the same transaction. Cancel before any match is reported and all entry fees + organizer deposits are refunded. There is no off-chain custody, no payout delay, and no admin override of match results.

---

## Repositories

The protocol is implemented as a polyrepo across five repositories. Each has its own README with the layer-specific surface area.

| Layer | Repo | Stack | Detail README |
|---|---|---|---|
| Smart contracts | [`bracket-chain-programs`](https://github.com/VitalikCholan/BracketChain-Programs) | Anchor 0.32.1, Solana 2.x | [README](../bracket-chain-programs/README.md) |
| TypeScript SDK | [`bracket-chain-sdk`](https://github.com/VitalikCholan/BracketChain-Sdk) | tsup CJS+ESM, [`@bracketchain/sdk`](https://www.npmjs.com/package/@bracketchain/sdk) | [README](../bracket-chain-sdk/README.md) |
| Indexer + REST API | [`bracket-chain-indexer`](https://github.com/VitalikCholan/BracketChain-Indexer) | NestJS 11, Prisma 7, Postgres on Neon, Railway | [README](../bracket-chain-indexer/README.md) |
| Web application | [`BracketChain-Frontend`](https://github.com/btcthirst/BracketChain-Frontend) | Next.js 16 App Router, Solana wallet adapter, Vercel | [README](../BracketChain-Frontend/README.md) |
| Hackathon plan + this README | `bracketchain-main` | — | (this file) |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Web app — Next.js 16 + wallet adapter (Phantom / Solflare)      │
│  · /create  /t/[id]  /explore  /dashboard                        │
│  · WebSocket account subscriptions (live bracket state)          │
└──────────┬──────────────────────────────────────┬────────────────┘
           │                                      │
   reads (SWR + RPC fallback)            writes (signed tx)
           │                                      │
           ▼                                      ▼
┌──────────────────────────┐         ┌──────────────────────────────┐
│  @bracketchain/sdk       │         │  @bracketchain/sdk           │
│  · BracketChainIndexer-  │         │  · BracketChainClient        │
│    Client (REST)         │         │  · createTournament,         │
│  · BracketChainClient    │         │    joinTournament,           │
│    (read-only Anchor)    │         │    startTournament,          │
└────────┬─────────────────┘         │    reportResult,             │
         │                           │    cancelTournament,         │
         │ HTTP                      │    subscribe                 │
         ▼                           └──────────────┬───────────────┘
┌─────────────────────────────────────┐             │
│  Indexer — NestJS 11 on Railway     │             │
│  · GET /tournaments[/:address[/*]]  │             │
│  · POST /webhooks/helius            │             │
│  · @nestjs/schedule cron (1 min)    │             │
│  · Prisma 7 → Postgres on Neon      │             │
└────────┬────────────────┬───────────┘             │
         │                │                         │
         │ getMultiple-   │ Helius enhanced         │
         │ AccountsInfo   │ webhooks (POST)         │
         │ (cron)         │                         │
         │                │                         │
         ▼                ▼                         ▼
       ┌────────────────────────────────────────────────────┐
       │  Anchor program — bracket_chain (Solana devnet)    │
       │  AuXJKpuZtkegs2ZSgopgckhN7Ev8bUz4zBc238LD2F1       │
       │  · 6 instructions, per-PDA state model             │
       │  · PDA-escrowed token vault (any SPL mint)         │
       │  · 96.5 / 3.5 auto-distribute on final match       │
       └────────────────────────────────────────────────────┘
```

**Read pattern** is stale-while-revalidate: the frontend tries the indexer first (sub-500ms), validates freshness via a `chainSlotAtWrite` watermark (~150 slots ≈ 60s tolerance), and falls back to a direct RPC chain read when the indexer is stale or unreachable. Live updates ride on Solana WebSocket account subscriptions to the Tournament + active Match PDAs.

**Write pattern** has no backend hop: the wallet adapter signs SDK-built transactions and sends them directly to Solana. The indexer learns about state changes only via Helius webhook events.

The reconciliation cron is the safety net for webhook drops — every minute it batches `getMultipleAccountsInfo` over non-terminal tournaments and patches DB drift.

---

## On-chain surface

Single Anchor program, six instructions:

| Instruction | Purpose |
|---|---|
| `initialize_protocol` | One-time singleton — sets fee BPS (350 = 3.5%), treasury, advisory default mint |
| `create_tournament` | Creates Tournament PDA + PDA-owned vault TA. Optional `organizer_deposit > 0` folds an organizer top-up into the prize pool in the same tx. |
| `join_tournament` | SPL token CPI: joiner ATA → vault. Creates Participant PDA. |
| `start_tournament` | Captures pseudo-random `seed_hash` from `slot_hashes` sysvar. Idempotent across chunks (default 7 matches/chunk → 19 chunks for 128 players). Bye matches mark Completed at init. |
| `report_result` | Validates match Active + winner ∈ {a, b}. Final-match branch validates 1st + 2nd on-chain, distributes prize per preset, takes 3.5% protocol fee — all in one transaction. |
| `cancel_tournament` | Two-tier auth: organizer flips status to Cancelled (first call); any signer can drive subsequent refund chunks. Refunds entry fees + `organizer_deposit` (idempotent flag). |

Status machine: `Registration → PendingBracketInit → Active → Completed | Cancelled`.

Account model is per-PDA (no inline `Vec` fields):
- `ProtocolConfig` `[b"protocol_config"]`
- `Tournament` `[b"tournament", organizer, name]` (`name` ≤ 32 bytes)
- `Participant` `[b"participant", tournament, wallet]`
- `MatchNode` `[b"match", tournament, [round: u8], match_index_le_bytes(u16)]`
- Vault Token Account at `[b"vault", tournament]` (NOT an ATA), `token::authority = tournament`

Constants: `max_participants ∈ [2, 128]`, `fee_bps = 350`, `MAX_TOURNAMENT_NAME_LEN = 32`. Three payout presets (WTA / Standard 60-25-15 / Deep 40-25-15-10-5-3-2).

Full instruction args, account schema, events list, and security caveats: [`bracket-chain-programs/README.md`](../bracket-chain-programs/README.md).

---

## What's in the MVP

- 6 Anchor instructions on devnet, IDL synced across SDK + indexer
- 3 fixed payout presets — Winner-Takes-All, Standard 60-25-15, Deep 40-25-15-10-5-3-2
- 2–128 participants per tournament
- Multi-token escrow at the program layer (any SPL mint accepted; per-tournament `token_mint` is unconstrained — `default_mint` is advisory only)
- Optional organizer deposit (Phase 2.5) with idempotent refund-on-cancel
- Pseudo-random seeding via `slot_hashes` sysvar at `start_tournament`
- Auto-distribute on final reported match: placements get 96.5%, treasury gets 3.5%, all atomic
- Cancel + refund-all path before any match is reported
- TypeScript SDK with two orthogonal clients (`BracketChainClient` for chain, `BracketChainIndexerClient` for REST), 21 typed errors with `mapError`, runtime `BN` re-export, single-PDA `subscribe()` for live state
- NestJS indexer with Helius webhook ingest + minute-cadence reconciliation cron
- Next.js web app: create / join / view / dashboard / explore, with WebSocket-driven live bracket updates and stale-while-revalidate reads

---

## Run locally

Each repo has its own setup section. Quickest end-to-end requires **all four**:

1. **Indexer** — `cd bracket-chain-indexer; pnpm install; cp .env.example .env; pnpm prisma migrate dev; pnpm start:dev` (port 3000)
   _Note_: set `RPC_URL` (not `SOLANA_RPC_URL` — the example file has a known typo) — see the indexer README.
2. **Frontend** — `cd BracketChain-Frontend; pnpm install; pnpm dev` (port 3000 — pick another for indexer)
   Set `NEXT_PUBLIC_INDEXER_URL` to the indexer's URL and `NEXT_PUBLIC_PROGRAM_ID=AuXJKpuZtkegs2ZSgopgckhN7Ev8bUz4zBc238LD2F1`.
3. **SDK** — already published as `@bracketchain/sdk@0.3.0` on npm; the frontend pulls it as a dependency.
4. **Programs** — only needed if you want to recompile + redeploy. `cd bracket-chain-programs; make build` (requires WSL2 on Windows).

Smoke-test path:
```bash
curl http://localhost:3000/health                                    # indexer
open  http://localhost:3001                                          # frontend
# Create a tournament via /create, fund a few wallets via Solana faucet, register, start, report.
```

---

## Risk register

The top five risks tracked across MVP development:

| Risk | Mitigation |
|---|---|
| `report_result` final match for Deep preset (7 transfers + fee CPI) exceeds compute budget at 128p | Pre-create ATAs in `preInstructions`. Test 5 (`128-player chunked start`) verifies bracket-init scale; full 128p payout path is Tier 4. |
| `cancel_tournament` with 128 participants exceeds compute budget | Two-tier auth — any signer can drive subsequent refund chunks, naturally degrading to N small txs rather than one mega-tx. |
| `slot_hashes` pseudo-random seeding predictable to organizers | Documented as MVP cut. Trustless-escrow use case accepts this; high-stakes brackets need V1 VRF. |
| Organizer can rig 3rd–Nth placements (only 1st + 2nd validated on-chain at final) | Documented MVP cut. V1 candidate (on-chain attestation). |
| Railway cold start mid-demo or Helius webhook drop | Pre-warm before demo. Reconciliation cron covers webhook drops within ~60s. |

---

## License

MIT. See [`LICENSE`](./LICENSE).

> The MIT license file is added pre-submission. All five repos use MIT.
