# Part 4: The Vault, Risk Router, and TradeIntent Pattern

## The full flow

```
Strategy decision (TradeDecision)
       ↓
  Build TradeIntent struct
       ↓
  Sign with EIP-712 (agentWallet)
       ↓
  RiskRouter.submitTradeIntent(intent, signature)
       ├── verifies EIP-712 signature → agentWallet in AgentRegistry
       ├── checks nonce (replay protection)
       ├── checks deadline
       ├── validates risk params (position size, trade frequency)
       ├── emits TradeApproved or TradeRejected on-chain
       ↓ (if approved)
  Kraken CLI: placeOrder()
       ↓
  Vault tracks capital
```

Every step is on-chain. Every approval and rejection is a permanent event.

---

## The TradeIntent struct

Instead of the agent directly calling Kraken, it first constructs a **signed intent**: a commitment to a specific trade that's been cryptographically authorized ([`contracts/RiskRouter.sol` L35–L44](https://github.com/Stephen-Kimoi/ai-trading-agent-template/blob/main/contracts/RiskRouter.sol#L35-L44)):

```solidity
struct TradeIntent {
    uint256 agentId;
    address agentWallet;       // must match AgentRegistry
    string  pair;              // e.g. "XBTUSD"
    string  action;            // "BUY" or "SELL"
    uint256 amountUsdScaled;   // USD * 100 (e.g. 50000 = $500)
    uint256 maxSlippageBps;    // max acceptable slippage
    uint256 nonce;             // replay protection
    uint256 deadline;          // Unix timestamp
}
```

The nonce increments with each approved intent, so an old signature can't be replayed.

---

## Building and signing a TradeIntent (TypeScript)

[`src/onchain/riskRouter.ts` L72–L145](https://github.com/Stephen-Kimoi/ai-trading-agent-template/blob/main/src/onchain/riskRouter.ts#L72-L145) handles this:

```typescript
const riskRouter = new RiskRouterClient(routerAddress, agentWallet, SEPOLIA_CHAIN_ID);

// 1. Build intent (fetches current nonce from chain)
const intent = await riskRouter.buildIntent(
  agentId,
  agentWallet.address,
  "XBTUSD",
  "BUY",
  100,   // $100 USD
  { maxSlippageBps: 50, deadlineSeconds: 300 }
);

// 2. Sign with EIP-712 (agentWallet is the hot signing key)
const signed = await riskRouter.signIntent(intent, agentWallet);

// 3. Submit to RiskRouter
const result = await riskRouter.submitIntent(signed);

if (result.approved) {
  console.log("Trade approved — intentHash:", result.intentHash);
} else {
  console.warn("Trade rejected:", result.reason);
}
```

The `intentHash` is carried into the EIP-712 checkpoint, linking the checkpoint to the specific approved intent.

---

## What the RiskRouter checks

```
1. deadline     — is the intent still valid?
2. nonce        — does it match the stored nonce (not replayed)?
3. signature    — does it recover to the registered agentWallet?
4. position size — is amountUsdScaled ≤ maxPositionSize?
5. trade frequency — are we within maxTradesPerHour?
```

Each check emits an event on Sepolia if it fails:
```
TradeRejected(agentId, intentHash, "Exceeds maxPositionSize")
TradeRejected(agentId, intentHash, "Intent expired")
```

If all checks pass:
```
TradeApproved(agentId, intentHash, amountUsdScaled)
```

---

## Setting your risk params

After deployment, configure risk params for your agent (owner-only, called by the vault/router deployer):

```typescript
await riskRouter.setRiskParams(
  agentId,
  500,    // maxPositionUsd: $500 per trade
  500,    // maxDrawdownBps: 5%
  10      // maxTradesPerHour
);
```

Or the register script does this automatically. You can also call it directly:

```bash
cast send $RISK_ROUTER_ADDRESS \
  "setRiskParams(uint256,uint256,uint256,uint256)" \
  $AGENT_ID 50000 500 10 \
  --rpc-url $SEPOLIA_RPC_URL --private-key $PRIVATE_KEY
```

---

## Connecting to an external vault + router

If you want to connect your agent to contracts deployed by someone else (a shared registry, a protocol, or a production deployment):

1. Set `HACKATHON_VAULT_ADDRESS` and `RISK_ROUTER_ADDRESS` in `.env` to those addresses
2. The `RiskRouterClient` and `VaultClient` will connect to those contracts automatically
3. Re-sign your `TradeIntent` against that router's EIP-712 domain (the `verifyingContract` field will differ)

The interface is identical — the same TypeScript code works with any deployment of these contracts.

---

## The Vault

Capital is allocated per-agent. Before sizing a trade, the agent verifies it has sufficient capital:

```typescript
const vault = new VaultClient(vaultAddress, provider);
const hasCapital = await vault.hasSufficientCapital(agentId, 100, ethPrice);
```

The vault tracks `allocatedCapital[agentId]` on-chain. Because the allocation is tied to the agent's ERC-721 `agentId`, it's tamper-proof — the agent can only trade up to what's been allocated to its registered identity.

---

## Template note

> **Why this matters:** The TradeIntent pattern gives every trade a cryptographic proof of intent that was validated on-chain before execution. This is what makes agent behavior auditable and trustworthy — anyone can verify that a specific trade was approved by a specific registered agent against a defined risk policy, without having to trust the agent's own logs.

---

→ [Part 5: Building the Explanation Layer](./05-explanation-layer.md)
