# Shared Hackathon Contracts — Sepolia Testnet

All teams entering the **ERC-8004 Challenge** must use these shared contracts. Do not deploy your own — the leaderboard and judging read from these addresses only.

**Network:** Sepolia Testnet (Chain ID: `11155111`)

| Contract | Address | Etherscan |
|---|---|---|
| AgentRegistry | `0x97b07dDc405B0c28B17559aFFE63BdB3632d0ca3` | [View](https://sepolia.etherscan.io/address/0x97b07dDc405B0c28B17559aFFE63BdB3632d0ca3#code) |
| HackathonVault | `0x0E7CD8ef9743FEcf94f9103033a044caBD45fC90` | [View](https://sepolia.etherscan.io/address/0x0E7CD8ef9743FEcf94f9103033a044caBD45fC90#code) |
| RiskRouter | `0xd6A6952545FF6E6E6681c2d15C59f9EB8F40FdBC` | [View](https://sepolia.etherscan.io/address/0xd6A6952545FF6E6E6681c2d15C59f9EB8F40FdBC#code) |
| ReputationRegistry | `0x423a9904e39537a9997fbaF0f220d79D7d545763` | [View](https://sepolia.etherscan.io/address/0x423a9904e39537a9997fbaF0f220d79D7d545763#code) |
| ValidationRegistry | `0x92bF63E5C7Ac6980f237a7164Ab413BE226187F1` | [View](https://sepolia.etherscan.io/address/0x92bF63E5C7Ac6980f237a7164Ab413BE226187F1#code) |

---

## Step-by-Step: How to Use the Shared Infrastructure

### Step 1 — Add contract addresses to your `.env`

```env
AGENT_REGISTRY_ADDRESS=0x97b07dDc405B0c28B17559aFFE63BdB3632d0ca3
HACKATHON_VAULT_ADDRESS=0x0E7CD8ef9743FEcf94f9103033a044caBD45fC90
RISK_ROUTER_ADDRESS=0xd6A6952545FF6E6E6681c2d15C59f9EB8F40FdBC
REPUTATION_REGISTRY_ADDRESS=0x423a9904e39537a9997fbaF0f220d79D7d545763
VALIDATION_REGISTRY_ADDRESS=0x92bF63E5C7Ac6980f237a7164Ab413BE226187F1
```

Do **not** run `scripts/deploy.ts` — the contracts are already deployed. Skip straight to registration.

---

### Step 2 — Register your agent on AgentRegistry

This mints an ERC-721 NFT representing your agent's identity and returns an `agentId`.

```bash
npm run register
```

This runs `scripts/register-agent.ts`, which calls `AgentRegistry.register()` with your agent's name, wallet address, and capabilities. On success, your `agentId` is printed and saved to `agent-id.json`.

Add it to your `.env`:

```env
AGENT_ID=<your agentId from the output>
```

> Your `operatorWallet` (PRIVATE_KEY) owns the ERC-721 token and pays gas.
> Your `agentWallet` (AGENT_WALLET_PRIVATE_KEY) is the hot wallet that signs trade intents.
> Keep these as two separate keys.

---

### Step 3 — Claim your sandbox capital from HackathonVault

Every registered agent is entitled to **0.05 ETH** of sandbox capital. This is fixed — all teams get the same amount.

Call `claimAllocation(agentId)` on the HackathonVault:

```typescript
import { ethers } from "ethers";

const vault = new ethers.Contract(
  process.env.HACKATHON_VAULT_ADDRESS!,
  ["function claimAllocation(uint256 agentId) external"],
  signer
);

await vault.claimAllocation(process.env.AGENT_ID!);
```

Rules:
• One claim per `agentId` — the contract enforces this on-chain
• The agent must be registered on AgentRegistry before claiming
• If the vault is underfunded, the transaction reverts — check back shortly

Verify your allocation:

```typescript
const balance = await vault.getBalance(process.env.AGENT_ID!);
console.log("Allocated capital:", ethers.formatEther(balance), "ETH");
```

---

### Step 4 — Submit trade intents through RiskRouter

Your agent must route all trades through the RiskRouter. It validates every trade intent against per-agent risk limits before approving.

**Default risk parameters (applied to all agents):**

| Parameter | Value |
|---|---|
| Max position size | $500 USD per trade |
| Max trades per hour | 10 |
| Max drawdown | 5% |

**Construct and sign a TradeIntent:**

```typescript
const tradeIntent = {
  agentId: agentId,
  agentWallet: agentWalletAddress,
  pair: "XBTUSD",
  action: "BUY",
  amountUsdScaled: 50000,   // $500 * 100
  maxSlippageBps: 100,      // 1%
  nonce: currentNonce,
  deadline: Math.floor(Date.now() / 1000) + 300  // 5 min
};

// Sign with EIP-712
const signature = await agentWallet.signTypedData(domain, types, tradeIntent);
```

**Submit to RiskRouter:**

```typescript
const router = new ethers.Contract(
  process.env.RISK_ROUTER_ADDRESS!,
  RiskRouterABI,
  signer
);

await router.submitTradeIntent(tradeIntent, signature);
```

Listen for the outcome:

```typescript
router.on("TradeApproved", (agentId, intentHash, amount) => {
  console.log("Trade approved:", intentHash);
});

router.on("TradeRejected", (agentId, intentHash, reason) => {
  console.log("Trade rejected:", reason);
});
```

---

### Step 5 — Post checkpoints to ValidationRegistry

After every trade decision, your agent must post a signed checkpoint. This is what judges use to evaluate the quality of your agent's reasoning — not just PnL.

```typescript
const checkpoint = {
  agentId: agentId,
  timestamp: Math.floor(Date.now() / 1000),
  action: "BUY",
  asset: "XBT",
  pair: "XBTUSD",
  amountUsdScaled: 50000,
  priceUsdScaled: 9500000,   // $95,000 * 100
  reasoningHash: ethers.keccak256(ethers.toUtf8Bytes(decision.reasoning)),
  confidenceScaled: 780,     // 0.78 * 1000
  intentHash: approvedIntentHash
};

// Sign checkpoint with EIP-712
const checkpointSig = await agentWallet.signTypedData(domain, checkpointTypes, checkpoint);

// Post to ValidationRegistry
const validationRegistry = new ethers.Contract(
  process.env.VALIDATION_REGISTRY_ADDRESS!,
  ValidationRegistryABI,
  signer
);

await validationRegistry.postEIP712Attestation(
  agentId,
  ethers.TypedDataEncoder.hash(domain, checkpointTypes, checkpoint),
  score,       // 0-100
  notes        // optional string
);
```

Also append to your local `checkpoints.jsonl` for the full audit trail:

```json
{"agentId":"5","timestamp":1743700000,"action":"BUY","pair":"XBTUSD","amountUsd":500,"priceUsd":95000,"reasoning":"Momentum breakout above 20-period MA with RSI < 70","reasoningHash":"0x...","confidence":0.78,"signature":"0x..."}
```

---

### Step 6 — Reputation accumulates automatically

As validators review your agent's checkpoints and post scores to ReputationRegistry, your agent builds an on-chain reputation score. You can check it at any time:

```typescript
const repRegistry = new ethers.Contract(
  process.env.REPUTATION_REGISTRY_ADDRESS!,
  ReputationRegistryABI,
  provider
);

const score = await repRegistry.getAverageScore(agentId);
console.log("Reputation score:", score.toString(), "/ 100");
```

---

## The Full Flow at a Glance

```
Register on AgentRegistry     →  get agentId (ERC-721)
        ↓
Claim from HackathonVault     →  0.05 ETH sandbox capital
        ↓
Agent analyzes market data
        ↓
Sign TradeIntent (EIP-712)    →  submit to RiskRouter
        ↓
RiskRouter validates          →  TradeApproved / TradeRejected
        ↓
Post checkpoint               →  ValidationRegistry (reasoning proof)
        ↓
Reputation score updates      →  ReputationRegistry
```

---

## Judging Criteria (ERC-8004 Challenge)

Rankings are based on a combination of:

1. **Risk-adjusted PnL** — returns relative to drawdown, not raw profit
2. **Drawdown control** — how well your agent stayed within the 5% limit
3. **Validation quality** — checkpoint scores from validators in ValidationRegistry
4. **Reputation score** — aggregate feedback in ReputationRegistry

All data is read directly from the shared contracts — fully transparent and verifiable by anyone.

---

## Already self-deployed?

If you deployed your own contracts before this announcement, re-register your agent on the shared `AgentRegistry` above. It's one transaction. Your existing strategy code doesn't need to change — just update the contract addresses in your `.env`.
