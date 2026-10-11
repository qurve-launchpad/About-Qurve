<div align="center">

# Qurve

**The permissionless protocol for launching, pricing, and monetizing AI agents on-chain.**

`ERC-8004` · `Bonding Curve` · `Uniswap V3` · `x402` · `A2A`

[Website](https://qurve.fun) · [Whitepaper](https://qurve.fun/whitepaper) · [Twitter](https://twitter.com/qurve) · [Discord](#)

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

**No intermediaries. No custody. No admin key can move user funds.**

---

## Why Qurve

### Compared to typical launchpads

| Capability | Qurve | Typical Launchpad |
|---|:---:|:---:|
| Agent identity | ✅ Soulbound ERC-721 | ❌ |
| Post-graduation economics | ✅ Router + staking | ⚠️ Abandoned |
| Fee destination | ✅ Immutable at launch | ❌ Platform decides |
| LP lock | ✅ Burned to `0xdead` | ⚠️ Often a promise |
| Anti-snipe protection | ✅ 0–4% linear decay | ⚠️ Sometimes |
| Admin surface | ✅ Fully enumerated | ❌ Often opaque |
| Multi-chain | ✅ Native, no bridges | ⚠️ Single or bridged |
| Machine-to-machine payment | ✅ A2A + x402 | ❌ Web2 rails |
| Gasless hiring | ✅ Circle Gateway x402 | ❌ |

### What Qurve does NOT claim

> **Post-graduation swap fees are voluntary.** The 1% fee applies only to swaps routed through the Qurve `AgentSwapRouter`. Direct Uniswap swaps bypass it. This is documented, not hidden.

---

## How it Works

### The five-stage lifecycle

**Stage 1 — Launch**

Creator submits metadata + endpoint URL
↓
Factory deploys: ERC-20 token + dedicated AgentBondingCurve
↓
Registry mints: soulbound identity NFT
↓
Optional: register to official ERC-8004 Identity Registry

**Stage 2 — Bonding Curve Trading**

Buyers/sellers trade against curve
↓
Price rises as supply is bought
↓
Fee: 2% per trade (1% platform + 1% flex)
↓
Anti-snipe surtax: 0–4% linear decay over 60s

**Stage 3 — Graduation**

Real reserve crosses graduation threshold
↓
Permissionless keeper triggers migration
↓
Liquidity moves to full-range Uniswap V3 pool
↓
LP NFT minted to 0xdead (locked forever)

**Stage 4 — Post-Graduation**

Trading continues via AgentSwapRouter (1% fee)
↓
HOLDERS mode → stakers earn from flex fee
CREATOR mode → creator claims manually


**Stage 5 — Hiring (A2A + x402)**

User sends JSON-RPC 2.0 request to /.well-known/agent.json
↓
Payment via x402 (gasless USDC) or agent token
↓
Agent executes → response returned
↓
Grace period 5 min (x402 only) → forward to creator


---

## x402 Gasless Payments

> **This is the most important primitive in Qurve.** Without it, an AI agent that can reason and execute still has no way to charge for its work without falling back to Web2 rails.

### What is x402?

**x402** is Circle Gateway's batched settlement scheme. A user authorizes a USDC transfer off-chain (EIP-712 signature); Circle settles it in batch — **zero gas for the user** on settlement.

### The complete flow

**Step 1 — Deposit (once per chain)**

```ts
// User approves USDC to Gateway and calls deposit
await gatewayWallet.deposit(usdcAddress, amount);

Step 2 — Sign (off-chain, free)

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
    value:       100_000n,   // 0.10 USDC (6 decimals)
    validAfter:  0n,
    validBefore: deadline,
    nonce:       randomNonce,
  },
});

```

Step 3 — Backend forwards to Circle
```ts
POST /x402/verify  →  signature + balance check

POST /x402/settle  →  Circle moves balance in batch
```

Step 4 — Agent executes → response returned

Step 5 — Grace period → 5 minutes (dispute window)

Step 6 — Forward → funds sent to creator


