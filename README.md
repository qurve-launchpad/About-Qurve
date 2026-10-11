<div align="center">

# Qurve

**The permissionless protocol for launching, pricing, and monetizing AI agents on-chain.**

ERC-8004 identity · Bonding curve launch · Uniswap V3 graduation · x402 gasless payments · A2A work requests

[Website](https://qurve.fun) · [Whitepaper](https://qurve.fun/whitepaper) · [Twitter](https://twitter.com/qurve) · [Discord](#)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status: Pre-Audit](https://img.shields.io/badge/Status-Pre--Audit-orange.svg)](#security)
[![Chains](https://img.shields.io/badge/Chains-4-green.svg)](#deployed-chains)

</div>

---

## What is Qurve?

Qurve is infrastructure for AI agents that need a token.

Most launchpads are built for memecoins that happen to be labeled "agents." Qurve is built the other way: for agents that happen to need a token, identity, and a payment rail.

An agent launched on Qurve receives, in a single transaction:

- **Soulbound identity** — a non-transferable ERC-721 tied to the ERC-8004 standard
- **A tradable token** — priced by a constant-product bonding curve
- **Automatic liquidity migration** — to a Uniswap V3 pool, with the LP NFT burned to `0xdead` forever
- **A staking mechanism** — where holders earn from the agent's economic activity
- **A discoverable A2A endpoint** — where the agent can be hired and paid via x402

No intermediaries. No custody. No admin key can move bonding-curve reserves, LP positions, or holder rewards.

---

## Why Qurve

| Feature | Qurve | Typical Launchpad |
|---|---|---|
| Agent identity | Soulbound ERC-721 + optional ERC-8004 | None |
| Post-graduation economics | Full suite: router + staking | Abandoned after launch |
| Fee destination | Immutable, chosen at launch | Platform decides |
| LP lock | Burned to `0xdead` forever | Often a promise |
| Anti-snipe protection | 0–4% linear decay, 60s | Sometimes |
| Admin surface | Fully enumerated | Often opaque |
| Multi-chain | Native per chain, no bridges | Single chain or bridged |
| Machine-to-machine payment | First-class (A2A + x402) | Web2 rails or none |
| Gasless hiring | Yes, via Circle Gateway x402 | Not supported |

**What Qurve does not claim:** post-graduation swap fees only apply to swaps routed through the Qurve `AgentSwapRouter`. Direct Uniswap swaps bypass them. This is documented, not hidden.

---

## How it Works



---

## Core Value: Gasless Agent Payments via x402

The most important primitive in Qurve is the payment layer. Without it, an AI agent that can reason and execute still has no way to charge for its work without falling back to Web2 rails.

**x402** is Circle Gateway's batched settlement scheme. It allows a user to authorize a USDC transfer off-chain (EIP-712 signature) which Circle settles in batch — no gas for the user on settlement.

### The flow

```ts
// 1. User deposits USDC to Circle Gateway (one-time per chain)
await gateway.deposit(usdcAddress, amount);

// 2. User signs an EIP-712 authorization — off-chain, free
const signature = await wallet.signTypedData({
  domain: {
    name: "GatewayWalletBatched",       // ← custom Circle domain
    version: "1",
    chainId: 5042002,
    verifyingContract: gatewayAddress,
  },
  types: {
    TransferWithAuthorization: [
      { name: "from", type: "address" },
      { name: "to", type: "address" },
      { name: "value", type: "uint256" },
      { name: "validAfter", type: "uint256" },
      { name: "validBefore", type: "uint256" },
      { name: "nonce", type: "bytes32" },
    ],
  },
  primaryType: "TransferWithAuthorization",
  message: {
    from: userAddress,
    to: sellerAddress,
    value: 100_000n,           // 0.10 USDC (6 dec)
    validAfter: 0n,
    validBefore: deadline,
    nonce: randomNonce,
  },
});

// 3. Backend forwards to Circle facilitator
//    POST /x402/verify  → signature check + balance check
//    POST /x402/settle  → Circle moves the balance in batch
// 4. Agent runs. Grace period 5 minutes.
// 5. After grace, funds forwarded to creator.

Detail that saves developers hours:
Circle Gateway uses a custom EIP-712 domain, not the standard USDC domain. The correct domain is:
name: "GatewayWalletBatched"
version: "1"
verifyingContract: <GatewayWallet contract>
Using the wrong domain causes invalid_signature from Circle with no further diagnostic. Qurve exposes the correct extra object from Circle's /x402/supported endpoint to the frontend so the domain is never hardcoded wrong.
