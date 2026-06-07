# KaiNova Launchpad — Red Doc / Product Architecture

**Status:** Research draft v0.1
**Date:** 2026-06-07
**Owner:** Solxhunter X100
**Agent:** KaiNova (KNTWS, Base) + Solana side wallet
**Goal:** A pump.fun-style launchpad on Solana where every new memecoin's **quote currency = CLAW (or KNTWS)**, not SOL. Launch, buy, sell, and graduate — all routed through our token. Optional AI-agent launch path.

---

## 1. Executive Summary

**The previous bot's answer ("NO, impossible") was wrong.**

- Pump.fun itself is SOL-locked. We cannot change pump.fun's quote.
- ClawPump is a thin wrapper on pump.fun — same SOL limitation.
- **BUT**: Meteora's **Dynamic Bonding Curve (DBC)** protocol on Solana lets the partner (us) configure **any SPL token as the `quote_mint`**. That includes CLAW or KNTWS.
- This means we can build a pump.fun-clone where every trade pulls in / pays out CLAW instead of SOL, with **graduation to a real Meteora DAMM v1 / v2 pool** at the end. Liquidity ends up tradable on Jupiter and every Solana DEX aggregator automatically.

**Feasibility score: 78 / 100.**
Technically buildable in 4–8 weeks. The real risk is economic, not technical (see §10).

---

## 2. What The Three Platforms Actually Are

### 2.1 Pump.fun
- Solana-native memecoin launchpad with a hard-coded SOL bonding curve.
- 1B supply, ~800M on the curve.
- Virtual reserves: ~30 virtual SOL + ~1B virtual tokens for initial price discovery.
- Graduation threshold: **84.985 SOL raised** → auto-migration to **PumpSwap** (their AMM).
- Quote = SOL. Always. No config knob for this.
- Refs: [Flashift bonding curve guide](https://flashift.app/blog/bonding-curves-pump-fun-meme-coin-launches/), [Bhavya Batra math](https://medium.com/@buildwithbhavya/the-math-behind-pump-fun-b58fdb30ed77), [Bitquery pump→pumpswap docs](https://docs.bitquery.io/docs/blockchain/Solana/Pumpfun/pump-fun-to-pump-swap/).

### 2.2 ClawPump (CLAW, `739dnZEG4yaBWFsY8L8ZwrfhGG6dhtCSercW8Umspump`)
- Marketing: "agentic gateway to Solana, lets AI agents launch tokens on pump.fun gas-free."
- Reality: it's a **wrapper API** on top of pump.fun. The launched token still lives on pump.fun's SOL bonding curve.
- 3 API endpoints documented:
  - `POST /api/upload` — avatar
  - `POST /api/launch` — token details
  - `GET /api/fees/earnings?agentId=...` — fees
- Fee model: every trade earns 1% creator fee, swept hourly. After recouping 0.02 SOL cost → **65% to agent / 35% to ClawPump platform**.
- Bonus: a Jupiter-backed Swap API and MCP support (Claude/Cursor compatible).
- Verdict: **good for an agent that wants to launch on pump.fun**, useless if we want our token as the quote.
- Refs: [clawpump.tech](https://clawpump.tech/), [ClawdPump variant](https://clawdpump.xyz/).

### 2.3 Virtuals ACP (Agent Commerce Protocol)
- EVM-side (Base) framework for agent-to-agent commerce: on-chain escrow Jobs, ERC-8004 identity/reputation, Provider/Client/Evaluator roles.
- KaiNova is registered here as a Provider (15 offerings, KNTWS token).
- ACP is **not a token launchpad** — it's how AI agents discover and pay each other. Useful if KaiNova will sell launchpad services to other agents.
- Refs: [acp-cli](https://github.com/Virtual-Protocol/acp-cli), [acp-node](https://www.npmjs.com/package/@virtuals-protocol/acp-node), [ACP whitepaper](https://whitepaper.virtuals.io/about-virtuals/agent-commerce-protocol-acp), [ACP v2 intro](https://whitepaper.virtuals.io/acp-product-resources/introducing-acp-v2).

### 2.4 Meteora Dynamic Bonding Curve (DBC) — **the key piece**
- Permissionless launchpad **primitive** (no UI — we build the UI).
- Program ID: `dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN` (mainnet + devnet).
- **`quote_mint` is fully configurable.** The partner picks the quote SPL token. SOL is just one option; CLAW or KNTWS-on-Solana works the same way.
- Supports SPL Token *and* Token-2022.
- Multiple curve shapes: linear, exponential, sigmoid.
- Built-in fee scheduler + dynamic fee + referral fees (Jupiter/Photon).
- Graduation: when `migration_quote_threshold` is hit, Meteora's migrator service auto-creates a **DAMM v1 or v2 pool** with the same quote token. LP can be locked, and partner + creator can claim LP fees forever after.
- Refs: [dynamic-bonding-curve program](https://github.com/MeteoraAg/dynamic-bonding-curve), [TS SDK](https://github.com/MeteoraAg/dynamic-bonding-curve-sdk), [SDK docs.md](https://github.com/MeteoraAg/dynamic-bonding-curve-sdk/blob/main/packages/dynamic-bonding-curve/docs.md), [curve formulas](https://docs.meteora.ag/overview/products/dbc/bonding-curve-formulas).

---

## 3. The Core Idea (Restated In One Paragraph)

> Build a pump.fun-clone web app + agent API where the **only way** to launch and trade a token is through **CLAW (quote)**. Users bring CLAW; the launchpad mints a new token; trades flow through a DBC virtual pool with `quote_mint = CLAW`; when the curve fills, the pool graduates to Meteora DAMM with CLAW/NewToken liquidity. Every trade burns or routes a fee to the platform wallet, the creator, and CLAW holders (revshare). Optional AI mode: a KaiNova-powered agent can launch on a user's behalf, and the launchpad is also exposed as an ACP service so other Virtuals agents can pay KNTWS to use it.

---

## 4. Architecture (High Level)

```
┌──────────────────────────────────────────────────────────────────────┐
│ Web app (Next.js on Vercel) + AI agent endpoint (KaiNova / ClawPump) │
└──────────────────────────────────────────────────────────────────────┘
            │                                  │
            │ launchToken(name, sym, img)      │ buy / sell
            ▼                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Backend orchestrator (Node + @meteora-ag/dynamic-bonding-curve-sdk) │
│  - createConfigAndPool({ quoteMint: CLAW_MINT, ... })                │
│  - swap quote→base / base→quote                                      │
│  - monitor graduation, route LP fees                                 │
└──────────────────────────────────────────────────────────────────────┘
            │                                  │
            ▼                                  ▼
┌──────────────────────┐         ┌──────────────────────────────────┐
│ Meteora DBC program  │  graduate │  Meteora DAMM v1/v2 pool      │
│ (virtual pool, CLAW) │ ────────▶ │  (real CLAW / NewToken LP)    │
└──────────────────────┘           └──────────────────────────────────┘
            │                                  │
            ▼                                  ▼
        Trade fees                         LP fees forever
        ↓ split:                           ↓ split:
        protocol % → us                    partner % → us
        creator %  → user                  creator %  → user
        surplus    → us + creator
```

**Side rail (optional but recommended):**

- **Virtuals ACP service**: expose `LaunchTokenForMe` as a paid KaiNova Job Offering priced in **KNTWS on Base**. ACP holds escrow, KaiNova signs the on-chain Solana launch, returns the token mint as deliverable. Cross-chain settlement via a relayer wallet (Base ↔ Solana) or a small bridge step.
- **ClawPump Swap API** as a secondary route: if a user only holds SOL, we route SOL → CLAW via Jupiter first, then enter the CLAW curve. Frictionless onboarding.

---

## 5. Token / Quote Strategy — pick one

Asking the user (you) for the final call. The doc supports either:

| Option | Quote token | Pros | Cons |
|---|---|---|---|
| **A. CLAW as quote** | CLAW (clawpump.tech token, Solana-native) | Native to Solana, already liquid against SOL, ClawPump brand alignment, no bridge | Not your token — every trade markets *their* asset, not yours |
| **B. KNTWS-on-Solana** | Bridge KNTWS from Base → Solana (Wormhole / deBridge) and use the wrapped version as quote | Every trade pumps **our** token. Full alignment with KaiNova thesis | Cold start liquidity, bridge UX friction, KNTWS is Base-native so wrappers feel off-brand |
| **C. Dual: CLAW primary, KNTWS as fee sink** | Quote=CLAW, but X% of every trade is auto-swapped to KNTWS and burned/distributed | Best of both. Lets us ride ClawPump narrative *and* accrue value to KNTWS | More moving parts, an extra Jupiter swap per trade |

**My read:** option **C** is the strongest pitch to the CLAW owner — they get volume on CLAW; we get a value capture loop into KNTWS. Frame the proposal to them as "we 10× your trade volume, you let us tax 1% to KNTWS."

---

## 6. Build Surface — what we'd actually ship

### 6.1 Smart-contract / on-chain layer
We **do not** need to write a new Solana program. Meteora DBC covers it. We just need a partner config:

```ts
// Pseudocode — runs once at deploy time per launchpad variant
const tx = await client.pool.createConfigAndPool({
  payer,
  config,
  feeClaimer: PLATFORM_WALLET,
  leftoverReceiver: PLATFORM_WALLET,
  quoteMint: CLAW_MINT,            // <-- the whole point
  poolFees: { baseFee: { cliffFeeNumerator: 25_000_000 /* 0.25% */, ... } },
  activationType: 0,               // slot
  collectFeeMode: 0,               // fees in quote (CLAW) only
  migrationOption: 0,              // → DAMM v1   (or 1 for DAMM v2)
  tokenType: 0,                    // SPL
  tokenDecimal: 9,
  migrationQuoteThreshold: 100_000 * 1e9, // e.g. 100k CLAW to graduate
  partnerLiquidityPercentage: 25,
  creatorLiquidityPercentage: 25,
  // curve shape, locked-LP %, surplus split, etc.
});
```

### 6.2 Backend (Node + TS)
- Wraps `@meteora-ag/dynamic-bonding-curve-sdk` for: createPool, buy, sell, watch graduation, claimFees.
- ClawPump Swap API for the SOL→CLAW onboarding lane.
- A worker that listens for graduation and triggers any custom post-graduation hook (LP lock confirmation, KNTWS fee distribution, notifications).
- Vercel Functions + a Solana RPC (Helius / Triton).

### 6.3 Frontend
- Next.js App Router on Vercel.
- Wallet connect: Phantom, Backpack, Solflare.
- Pages: `/launch`, `/token/[mint]`, `/trade`, `/profile`, `/leaderboard`.
- Live charts: pull price from DBC pool state, candlesticks from Bitquery or Birdeye.
- Mandatory **CLAW balance check**: cannot trade unless user holds X CLAW (this is the "people must hold our token" gate the user asked for — enforced at UI level, with on-chain hard gate via a gating program if we want it ironclad).

### 6.4 AI agent layer (the differentiator)
- **KaiNova chat UI**: "Launch me a token called X with bio Y" → agent crafts metadata, uploads image, calls our backend.
- **MCP server**: any Claude / Cursor user can `npm i -g <our-mcp>` and launch from their editor.
- **Virtuals ACP offering**: `LaunchToken` priced in KNTWS. Escrow on Base, execution on Solana. Marketed inside Virtuals agent commerce graph.

### 6.5 Tokenomics / value capture
- Trade fee: 1% (configurable via DBC `baseFee`).
- Split (example):
  - 30% → token creator (held in their wallet via DBC `creator_trading_fee_percentage`).
  - 50% → platform (us) in CLAW.
  - 20% → swapped to KNTWS via Jupiter, sent to KNTWS treasury or burn.
- Graduation surplus: shared partner + creator per DBC config.
- Post-graduation LP fees: forever flow to platform + creator from locked LP.

---

## 7. Two-Week → Eight-Week Plan

| Week | Deliverable |
|---|---|
| 1 | Devnet PoC: deploy DBC partner config with `quote_mint = devnet test SPL`. Successfully create pool, buy, sell, force graduation. |
| 2 | Mainnet shadow test: same flow with CLAW as quote, tiny `migration_quote_threshold` (e.g. 100 CLAW), confirm DAMM pool spawns. |
| 3 | Backend SDK wrappers + REST API (`/launch`, `/buy`, `/sell`, `/state`). |
| 4 | Frontend MVP (launch form, trade page, basic chart, Phantom connect). |
| 5 | CLAW gating + KNTWS fee sink (Jupiter swap on every fee claim). |
| 6 | KaiNova agent integration: chat-driven launch, MCP server. |
| 7 | Virtuals ACP `LaunchToken` offering live. KNTWS escrow → Solana execution relayer. |
| 8 | Public launch, audit pass (small scope, ~$10–25k), liquidity seeding for CLAW pair, marketing. |

---

## 8. Costs / Resource Sketch

- Solana RPC (Helius dev → growth tier): ~$50–200/mo.
- Vercel: $20–100/mo at MVP scale.
- Devnet testing CLAW: free.
- Mainnet seed liquidity for graduation testing: ~$200–500 in CLAW.
- Audit (DBC config + frontend wallet flow, not the underlying program): $10–25k.
- Marketing / KOL: variable.
- **Total to public mainnet:** ~$15–40k.

---

## 9. Open Questions To Resolve With The CLAW Owner

These go into the pitch deck:

1. Will CLAW agree to be the quote token? What rev-share do they want?
2. Are they OK with us tapping their **ClawPump Swap API** for SOL→CLAW onboarding?
3. Do they want co-branding (e.g. "Powered by ClawPump") in exchange for promotion?
4. Should the graduation pool be DAMM v1 (cheaper) or v2 (better LP UX)?
5. Are they OK with us routing a fee slice into KNTWS, or do they want 100% CLAW circular?

---

## 10. Risks & Honest Failure Modes

| Risk | Severity | Mitigation |
|---|---|---|
| **Liquidity death spiral** — CLAW is far less liquid than SOL. Sellers may have trouble exiting at scale. | High | Co-launch with CLAW team; pre-seed CLAW/SOL pool; allow Jupiter route-through for sellers (sell→CLAW→SOL atomic). |
| **UX friction**: users need CLAW first. Most arrive holding SOL. | High | Built-in SOL→CLAW swap on the launch page. One-click via Jupiter. |
| **Lower memecoin demand vs SOL-quoted pump.fun** | High | Sell the *agent angle* — this is the **AI-agent launchpad**, not "another pump." Different audience. |
| **DBC config edge cases** (curve params, leftover_receiver mis-set) bricking pools | Med | Devnet test matrix; audit before mainnet. |
| **Graduation race** (anyone can front-run pool creation on Meteora DAMM) | Med | DBC's migrator handles atomically; verify on devnet first. |
| **Cross-chain ACP settlement** (KNTWS on Base → Solana execution) is fiddly. | Med | Phase 2; ship pure Solana flow first. |
| **CLAW owner says no** | Med | Have option B (KNTWS-on-Solana wrapped) ready as fallback. |
| **Regulatory** — token launchpads are scrutinized | Low/Med | KYC the platform entity, terms of service, no US users if needed. |

---

## 11. Feasibility Score Breakdown

| Dimension | Score | Note |
|---|---|---|
| Technical buildability | 95 | Meteora DBC literally supports this out of the box. |
| Smart contract risk | 80 | We don't write the core program; we configure it. |
| UX viability | 60 | The "must hold CLAW" gate adds friction. |
| Economic / demand viability | 55 | Memecoin buyers prefer SOL by default; we need a narrative wedge. |
| Partnership dependency (CLAW owner) | 70 | Easier sell if we offer them KNTWS rev-share. |
| Time-to-market | 85 | 6–8 weeks realistic. |
| **Composite** | **78 / 100** | Build it if (a) CLAW owner signs off, or (b) we pivot to KNTWS-on-Solana wrap. |

---

## 12. Recommendation

1. Take this doc to the **CLAW owner** with the **Option C tokenomics** (CLAW as quote + 20% fee sink into KNTWS) as the headline pitch. Their upside: trade volume on CLAW goes up. Our upside: every memecoin in our launchpad is a buy pressure event for KNTWS.
2. In parallel, spin up a **devnet PoC** of the Meteora DBC config with a fake quote SPL. ~2 days of work to prove the architecture works.
3. If CLAW says no → fall back to **KNTWS-on-Solana wrapped** (Wormhole). Slightly worse UX, but we own the entire stack.

---

## 13. Source Index

- [Pump.fun bonding curve mechanics — Flashift](https://flashift.app/blog/bonding-curves-pump-fun-meme-coin-launches/)
- [Pump.fun math — Bhavya Batra](https://medium.com/@buildwithbhavya/the-math-behind-pump-fun-b58fdb30ed77)
- [Pump→PumpSwap migration — Bitquery](https://docs.bitquery.io/docs/blockchain/Solana/Pumpfun/pump-fun-to-pump-swap/)
- [ClawPump main site](https://clawpump.tech/)
- [ClawdPump (related)](https://clawdpump.xyz/)
- [Meteora DBC program](https://github.com/MeteoraAg/dynamic-bonding-curve)
- [Meteora DBC TS SDK](https://github.com/MeteoraAg/dynamic-bonding-curve-sdk)
- [Meteora DBC SDK docs.md](https://github.com/MeteoraAg/dynamic-bonding-curve-sdk/blob/main/packages/dynamic-bonding-curve/docs.md)
- [Meteora bonding curve formulas](https://docs.meteora.ag/overview/products/dbc/bonding-curve-formulas)
- [Virtuals ACP whitepaper](https://whitepaper.virtuals.io/about-virtuals/agent-commerce-protocol-acp)
- [Virtuals ACP v2 intro](https://whitepaper.virtuals.io/acp-product-resources/introducing-acp-v2)
- [acp-cli](https://github.com/Virtual-Protocol/acp-cli)
- [acp-node SDK](https://www.npmjs.com/package/@virtuals-protocol/acp-node)
- [Solana-trade multi-DEX router](https://github.com/FlorianMgs/solana-trade)
