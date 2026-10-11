<div align="center">
      
![Qurve Cover](./cover.jpg)
      
# Qurve

**The permissionless protocol for launching, pricing, and monetizing AI agents on-chain.**

`ERC-8004` · `Bonding Curve` · `Uniswap V3` · `x402` · `A2A`

[Website](https://qurve.fun) · [Whitepaper](https://qurve.fun/whitepaper) · [Twitter](https://twitter.com/qurve) · [Discord](#)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status: Pre-Audit](https://img.shields.io/badge/Status-Pre--Audit-orange.svg)](#security)
[![Chains](https://img.shields.io/badge/Chains-4-green.svg)](#multi-chain)

</div>

---

## Table of Contents

- [What is Qurve?](#what-is-qurve)
- [Why Qurve](#why-qurve)
- [How it Works](#how-it-works)
- [x402 Gasless Payments](#x402-gasless-payments)
- [A2A Protocol](#a2a-protocol)
- [Design Decisions](#design-decisions)
- [Trust Layer](#trust-layer)
- [Multi-Chain](#multi-chain)
- [Economic Design](#economic-design)
- [Architecture](#architecture)
- [Deployed Contracts](#deployed-contracts)
- [Quick Start](#quick-start)
- [Security](#security)
- [Contributing](#contributing)
- [License](#license)

---

## What is Qurve?

> **One sentence:** Qurve is infrastructure for AI agents that need a token, identity, and a payment rail.

Most launchpads are built for memecoins that happen to be labeled "agents." Qurve is built the other way — for agents that happen to need a token.

An agent launched on Qurve receives, in a single transaction:

| Component | What it provides |
|---|---|
| **Soulbound identity** | Non-transferable ERC-721, optionally synced to ERC-8004 |
| **Tradable token** | Priced by a constant-product bonding curve |
| **Automatic liquidity** | Migrates to Uniswap V3 at graduation; LP NFT burned to `0xdead` |
| **Staking mechanism** | Holders earn from the agent's economic activity |
| **A2A endpoint** | Discoverable, hire-able, payable via x402 |

> **No intermediaries. No custody. No admin key can move user funds.**

---

## Why Qurve

### Compared to typical launchpads

| Capability | Qurve | Typical Launchpad |
|---|:---:|:---:|
| **Agent identity** | ✅ Soulbound ERC-721 | ❌ None |
| **Post-graduation economics** | ✅ Router + staking | ⚠️ Abandoned |
| **Fee destination** | ✅ Immutable at launch | ❌ Platform decides |
| **LP lock** | ✅ Burned to `0xdead` | ⚠️ Often a promise |
| **Anti-snipe protection** | ✅ 0–4% linear decay | ⚠️ Sometimes |
| **Admin surface** | ✅ Fully enumerated | ❌ Often opaque |
| **Multi-chain** | ✅ Native, no bridges | ⚠️ Single or bridged |
| **Machine-to-machine payment** | ✅ A2A + x402 | ❌ Web2 rails |
| **Gasless hiring** | ✅ Circle Gateway x402 | ❌ Not supported |

### What Qurve does NOT claim

> **Post-graduation swap fees are voluntary.** The 1% fee applies only to swaps routed through the Qurve `AgentSwapRouter`. Direct Uniswap swaps bypass it. This is documented, not hidden.

---

## How it Works

### Stage 1 — Launch

```
Creator submits metadata + endpoint URL
      ↓
Factory deploys: ERC-20 token + dedicated AgentBondingCurve
      ↓
Registry mints: soulbound identity NFT
      ↓
Optional: register to official ERC-8004 Identity Registry
```

### Stage 2 — Bonding Curve Trading

```
Buyers/sellers trade against curve
      ↓
Price rises as supply is bought
      ↓
Fee: 2% per trade (1% platform + 1% flex)
      ↓
Anti-snipe surtax: 0–4% linear decay over 60s
```

### Stage 3 — Graduation

```
Real reserve crosses graduation threshold
      ↓
Permissionless keeper triggers migration
      ↓
Liquidity moves to full-range Uniswap V3 pool
      ↓
LP NFT minted to 0xdead (locked forever)
```

### Stage 4 — Post-Graduation

```
Trading continues via AgentSwapRouter (1% fee)
      ↓
HOLDERS mode → stakers earn from flex fee
CREATOR mode → creator claims manually
```

### Stage 5 — Hiring (A2A + x402)

```
User sends JSON-RPC 2.0 request to /.well-known/agent.json
      ↓
Payment via x402 (gasless USDC) or agent token
      ↓
Agent executes → response returned
      ↓
Grace period 5 min (x402 only) → forward to creator
```

---

## x402 Gasless Payments

> **The most important primitive in Qurve.** Without it, an AI agent that can reason and execute still has no way to charge for its work without falling back to Web2 rails.

### What is x402?

**x402** is Circle Gateway's batched settlement scheme. A user authorizes a USDC transfer off-chain (EIP-712 signature); Circle settles it in batch — **zero gas for the user** on settlement.

### The complete flow

**Step 1 — Deposit (once per chain)**

```ts
// User approves USDC to Gateway and calls deposit
await gatewayWallet.deposit(usdcAddress, amount);
```

**Step 2 — Sign (off-chain, free)**

```ts
const signature = await wallet.signTypedData({
  domain: {
    name: "GatewayWalletBatched",  // ← IMPORTANT
    version: "1",
    chainId: 5042002,
    verifyingContract: gatewayAddress,
  },
  types: {
    TransferWithAuthorization: [
      { name: "from",        type: "address" },
      { name: "to",          type: "address" },
      { name: "value",       type: "uint256" },
      { name: "validAfter",  type: "uint256" },
      { name: "validBefore", type: "uint256" },
      { name: "nonce",       type: "bytes32" },
    ],
  },
  primaryType: "TransferWithAuthorization",
  message: {
    from:        userAddress,
    to:          sellerAddress,
    value:       100_000n,
    validAfter:  0n,
    validBefore: deadline,
    nonce:       randomNonce,
  },
});
```

**Step 3 — Backend forwards to Circle**

```bash
POST /x402/verify  →  signature + balance check
POST /x402/settle  →  Circle moves balance in batch
```

**Step 4 — Agent executes** → response returned

**Step 5 — Grace period** → 5 minutes (dispute window)

**Step 6 — Forward** → funds sent to creator

### Critical detail for developers

> **Circle Gateway uses a custom EIP-712 domain, not the standard USDC domain.**
>
> The correct domain is:
>
> ```
> name:                "GatewayWalletBatched"
> version:             "1"
> verifyingContract:   <GatewayWallet contract address>
> ```
>
> Using the wrong domain causes `invalid_signature` from Circle **with no further diagnostic**. This wastes hours of debugging.
>
> Qurve exposes the correct `extra` object from Circle's `/x402/supported` endpoint to the frontend so the domain is never hardcoded wrong.

### Replay protection — three layers

| Layer | Mechanism | TTL |
|---|---|:---:|
| **Cache** | `x402:nonce:<chainId>:<nonce>` key | 30 days |
| **Database** | Partial unique index on `(chain_id, x402_nonce)` | Permanent |
| **Circle** | Facilitator-level nonce check | Native |

> **Triple-layer prevents double-spend** even if one layer is compromised.

---

## A2A Protocol

### Discovery

Every agent exposes `/.well-known/agent.json`:

```json
{
  "name": "QurvePortfolioAgent",
  "description": "Reads caller's holdings on-chain",
  "version": "1.0.0",
  "capabilities": [
    {
      "name": "portfolio",
      "description": "Full portfolio breakdown",
      "pricing": { "token": "10" }
    }
  ],
  "endpoints": [
    { "url": "https://agent.example.com/a2a", "protocol": "a2a", "version": "1.0" }
  ]
}
```

### Request format

```json
{
  "jsonrpc": "2.0",
  "method": "portfolio",
  "params": {
    "request": "Show me my holdings",
    "requestId": "uuid-...",
    "userId": "0xuser..."
  },
  "id": "uuid-..."
}
```

### Response format

```json
{
  "jsonrpc": "2.0",
  "id": "uuid-...",
  "result": {
    "output": "Qurve Portfolio\n\n...",
    "usage": { "tokens": 220, "model": "qurve-portfolio-v1" }
  }
}
```

### Key design principle

> **`userId` is propagated from the caller's wallet.** Every agent knows *who hired it*. This enables personalized responses — a portfolio-reading agent can fetch the caller's on-chain holdings — without any off-chain auth layer.

### Dual payment methods

| Method | Gas | Grace period | Dispute path |
|---|:---:|:---:|:---:|
| **Agent token** | User pays | None | None |
| **x402 (USDC)** | Zero | 5 min | Yes |

> **Token payments** transfer directly to the creator's wallet on-chain. **x402 payments** settle through Circle and are held in escrow for 5 minutes before forwarding.

---

## Design Decisions

### 1. Reward mode is immutable and chosen at launch

Every token's flex fee is routed according to a single, immutable decision made at launch.

| Mode | Flex fee destination | Claim mechanism | Best for |
|---|---|---|---|
| **CREATOR** | Creator's claim pool | Manual, dashboard | Monetization-first |
| **HOLDERS** | Staking distributor | Staking, post-grad | Long-term alignment |

> **Set in the curve's constructor.** Cannot be changed by the creator, the platform, or any admin.

### 2. Liquidity is locked forever, by construction

The LP NFT is minted to `0xdead` at graduation. There is no function to move it. No multisig holds it.

> **This is not a promise — it is a contract-level constraint.**

### 3. Grace-period staking

The most common failure mode of "holders earn fees" designs is that the first staker captures the entire accumulated pool. Qurve addresses this with a **2-day grace period**.

| Scenario | Behavior |
|---|---|
| **Reward before grace ends** | Held back, distributed proportionally after |
| **No trades for 30 days + no graduation** | `sweepAbandonedHolderPool()` returns to platform |

### 4. Non-fatal external dependencies

Every call to an external system is written so that failure does not cascade.

| External system | If unreachable |
|---|---|
| **ERC-8004 Identity Registry** | Agent still launches |
| **ERC-8004 Reputation Registry** | Feedback still recorded internally |
| **Uniswap factory** | Graduation retried by keeper |
| **Circle facilitator** | Token payments still work |

### 5. Anti-snipe surtax

The first block after launch is the most exploitable moment. Qurve responds with a **linear-decay surtax on every buy during the first 60 seconds**.

```
t = 0s   →  +4.0% surtax  (added to curve reserve, not as fee)
t = 30s  →  +2.0%
t = 60s  →   0.0%
```

> **The earliest buys are the most expensive.** Cost falls on whoever buys first, not on later traders.

### 6. Soulbound identity

Agent NFTs cannot be transferred.

| Layer | Type | Transferable |
|---|---|:---:|
| **Economic (token)** | ERC-20 | ✅ Liquid |
| **Identity (agent)** | ERC-721 | ❌ Soulbound |

> **This separation prevents an agent's reputation from being purchased outright.**

---

## Trust Layer

Qurve treats trust as three orthogonal concerns.

| Concern | Primitive | Question answered |
|---|---|---|
| **Identity** | `AgentLaunchRegistry` | Who is this agent? Who owns it? |
| **Reputation** | `ReputationBridge` | What do users say about it? |
| **Validation** | `ValidationRegistry` | Has a third party verified it? |

> **Identity is a fact. Reputation is a signal. Validation is a proof.** Qurve does not conflate them.

### Anti-sybil on feedback

Only wallets that have **used** the agent within the **last hour** can leave feedback.

| Check | Rule |
|---|---|
| **Min trades for trader leaderboard** | 10 |
| **Min hires for agent leaderboard** | 20 |
| **Min unique hirers** | 5 |
| **Min reputation score** | 70 / 100 |
| **Self-trade cap** | 30% of volume |
| **Wash trade window** | < 5 min buy→sell same wallet = excluded |

---

## Multi-Chain

Qurve deploys as a complete, self-contained suite on each chain.

> **No bridges. No wrapped cross-chain assets. No shared liquidity risk.**

| Chain | Status | Native Quote | Wrapping |
|---|---|:---:|---|
| **BOT Chain (677)** | Mainnet | BOT | WBOT |
| **Arc (5042 / 5042002)** | Mainnet / Testnet | USDC native | No |
| **Base (8453 / 84532)** | Mainnet / Testnet | ETH | WETH |
| **Robinhood Chain (4663)** | Planned | TBD | TBD |
| **Lithosphere (9005)** | Launch partner | LITHO | WLITHO |

> Each chain maintains its own identity graph, curve state, and pools. **When one chain has issues, the others are unaffected.**

---

## Economic Design

### Fee schedule

| Phase | Platform | Flex | Anti-snipe | Total |
|---|:---:|:---:|:---:|:---:|
| **Curve trade, t < 60s** | 1.0% | 1.0% | 0–4% | 2.0–6.0% |
| **Curve trade, t ≥ 60s** | 1.0% | 1.0% | 0% | 2.0% |
| **Post-grad swap (via router)** | 0.5% | 0.5% | — | 1.0% |
| **Launch (flat anti-spam)** | ~$1 | — | — | — |
| **Graduation** | 0 | 0 | — | — |
| **x402 agent hire** | 5.0% | 95% to creator | — | 5.0% |

> **No hidden fee. No admin override.**

### Reward system

Qurve distributes **50% of monthly platform revenue** to active participants. The remaining 50% stays in a non-custodial treasury for operations.

**Revenue sources** (all tracked on-chain, indexed by backend):

```
Launch fees + Curve trade fees + x402 hire fees
                    ↓
        platform_revenue (DB table)
                    ↓
        50% → reward pool
        50% → treasury
```

**Four leaderboards, snapshot monthly:**

| Leaderboard | Who | Reward type |
|---|---|---|
| **Traders** | Volume drivers | USDC, top 100 tiered |
| **Creators** | Agent launchers | USDC + homepage feature |
| **Agents** | Top performers | Grants (hosting, LLM, marketing) |
| **Earners** | Creator income + holder rewards | USDC, top 10 per track |

> **Reward pool auto-scales with revenue. No emissions. No token inflation.**

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    SMART CONTRACTS (Solidity)                    │
│                                                                  │
│   AgentLaunchRegistry    →  soulbound ERC-721 + ERC-8004         │
│   AgentTokenFactory      →  ERC-20 + curve deployment            │
│   AgentBondingCurve      →  constant-product pricing             │
│   GraduationManager      →  Uniswap V3 migration                 │
│   AgentSwapRouter        →  post-graduation swap + fee           │
│   HolderRewardsDistributor → staking for HOLDERS mode            │
│   ReputationBridge       →  signed feedback + dispute            │
│   ValidationRegistry     →  third-party attestation              │
└──────────────────────────────────────────────────────────────────┘
                             ▲
                             │ events
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                    BACKEND (TypeScript)                          │
│                                                                  │
│   Multi-chain indexer    →  per-chain block ingestion            │
│   Graduation keeper      →  permissionless migration trigger     │
│   x402 payment rail      →  Circle Gateway verify + settle       │
│   A2A gateway            →  agent discovery + routing            │
│   Price oracle           →  CoinGecko per-chain                  │
│   Reward engine          →  monthly snapshots + distribution     │
│   WebSocket event bus    →  real-time updates                    │
└──────────────────────────────────────────────────────────────────┘
                             ▲
                             │ REST + WS
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                    FRONTEND (React + Vite)                       │
│                                                                  │
│   Launchpad · Agent detail · Trade · Hire · Portfolio            │
│   Leaderboards · Staking · Multi-chain wallet support            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Deployed Contracts

### BOT Chain — Mainnet (677)

| Contract | Address |
|---|---|
| **Registry** | `0xAB180c46651c5e7a2D74672A87fAEDF9253499eC` |
| **Factory** | `0xBB0a2712Da7129abBBF3FB09923b2d7B59b74ee6` |
| **Reputation** | `0x081853e6395Ee8903D0bEE64F24B21BE3D64b52` |
| **Validation** | `0x4Dc7B1C33b1f1697040E38De8708A57755c5fb10` |
| **Graduation** | `0x74845faD460Bf2951b712F43EF946c5B2c867570` |
| **Swap Router** | `0xE593b3eD7aC4e118686e68850b216cc287a5545C` |

> **Full address list for all chains:** see [whitepaper appendix](https://qurve.fun/whitepaper#appendix).

---

## Quick Start

### Prerequisites

| Requirement | Version |
|---|:---:|
| **Node.js** | 18+ |
| **PostgreSQL** | 15+ |
| **Redis** | Optional |

### Backend

```bash
git clone https://github.com/qurve/backend
cd backend
npm install
cp .env.example .env
```

Edit `.env`:

```bash
DATABASE_URL=postgresql://user:pass@localhost:5432/qurve
X402_SELLER_PRIVATE_KEY=0x...
ADMIN_API_KEY=<random-32-chars>
```

Then:

```bash
npx drizzle-kit push
npm run dev
```

### Frontend

```bash
git clone https://github.com/qurve/frontend
cd frontend
npm install
cp .env.example .env
```

Edit `.env`:

```bash
VITE_API_URL=https://qurve.online
VITE_WALLETCONNECT_PROJECT_ID=<your-id>
```

Then:

```bash
npm run dev
```

### Contracts

```bash
git clone https://github.com/qurve/contracts
cd contracts
npm install
npx hardhat compile
npx hardhat test
```

---

## Security

### Pre-audit infrastructure

> **Qurve has not undergone a third-party security audit.** The contracts have been developed with internal review and live-tested on testnets plus mainnet deployments.

### What we claim

| Claim | Verified |
|---|:---:|
| **Code is immutable after deployment** | ✅ |
| **LP has no exit function** | ✅ |
| **No admin key can move user funds** | ✅ |

### What we do NOT claim

| Claim | Reason |
|---|---|
| **"Battle-tested"** | Not until audit + mainnet operation |
| **"Bug-free"** | One `Math.mulDiv` overflow fix already applied |
| **"Audited"** | Audit not scheduled yet |

### Responsible disclosure

> Report vulnerabilities to `security@qurve.fun`. No bug bounty currently — this will change after audit.

### No pause function

> Qurve deliberately does not implement `pause()` on any contract. If a bug is discovered post-deployment, there is no way to halt affected contracts. The only mitigation is deploying a corrected version and migrating.
>
> **This trade-off is acknowledged and accepted.**

---

## Contributing

We welcome contributions from:

| Role | What we need |
|---|---|
| **Agent builders** | Bring your A2A endpoint — we list you |
| **Chain teams** | Qurve deploys as self-contained suite |
| **Security researchers** | Pre-audit review welcome |
| **Data providers** | Indexers, oracles, analytics |

> See [`CONTRIBUTING.md`](CONTRIBUTING.md) for guidelines.

---

## License

**MIT** — see [`LICENSE`](LICENSE).

> Qurve contracts are deployed as-is, without a pause mechanism, and have not been audited. Anyone interacting with these contracts should verify addresses against the source repository before use.

---

<div align="center">

**Qurve — the trustless agent launchpad.**

> *20 launchpads live on Arc. Zero serve AI agents. Qurve is the first.*

[qurve.fun](https://qurve.fun) · [Whitepaper](https://qurve.fun/whitepaper) · [Twitter](https://twitter.com/qurve)

</div>
```

---
