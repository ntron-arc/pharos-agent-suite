---
name: pharos-agent-suite
description: The complete Pharos AI Agent toolkit. Invoke for ANY Pharos Network task that involves wallet analysis, airdrop eligibility scoring, DeFi workflow automation, or multi-step onchain strategy execution. This skill extends the base pharos-skill-engine with three advanced intelligence layers: (1) Wallet Health — deep diagnostics on any address including balance history, gas efficiency, risk flags; (2) Airdrop Radar — score wallets against known Pharos ecosystem eligibility criteria and recommend actions; (3) DeFi Compose — chain multiple DeFi operations (wrap → approve → swap → stake) from a single natural language intent. Trigger whenever the user says "analyze my wallet", "am I eligible for the airdrop", "check my Pharos activity", "what should I do with my PHRS", "compose a DeFi strategy", "optimize my portfolio", "how active is this wallet", or any variant implying multi-step reasoning about a Pharos wallet or DeFi position. Requires pharos-skill-engine as a base dependency.
version: 1.0.0
requires:
  anyBins:
    - cast
    - forge
  skills:
    - pharos-skill-engine
---

# Pharos Agent Suite

Three advanced intelligence layers built on top of the Pharos Skill Engine. Where the base engine handles individual onchain operations, this suite handles **reasoning, strategy, and multi-step execution** — the things only AI agents can do well.

---

## Layer 1 — Wallet Health Diagnostics

**Trigger phrases:** "analyze wallet", "wallet health", "check my activity", "how active is this address", "wallet report"

The agent performs a comprehensive wallet audit and returns a structured health report.

### What the Agent Collects

Run all queries against the target network (default: `atlantic-testnet`). Read RPC config from the base skill's `assets/networks.json`.

```bash
# 1. Native PHRS balance
cast balance <ADDRESS> --rpc-url $RPC_URL --ether

# 2. Transaction count (activity proxy)
cast tx-count <ADDRESS> --rpc-url $RPC_URL

# 3. Token balances — query known ecosystem tokens
# Loop through assets/tokens.json (provided in this repo)
cast call <TOKEN_CONTRACT> "balanceOf(address)(uint256)" <ADDRESS> --rpc-url $RPC_URL

# 4. Recent transactions — check last known block activity
cast block latest --rpc-url $RPC_URL
```

### Health Report Format

The agent MUST output the report in this structured format:

```
══════════════════════════════════════
  PHAROS WALLET HEALTH REPORT
  Address: 0x...
  Network: Atlantic Testnet | Mainnet
  Generated: <timestamp>
══════════════════════════════════════

📊 BALANCE SUMMARY
  Native PHRS:     X.XXXX PHRS
  Token Holdings:  [list each ERC-20 with balance]
  Portfolio Est.:  ~$X,XXX (if price data available)

⚡ ACTIVITY SCORE: [0-100]
  Total Transactions:  XXX
  Contracts Interacted: XX unique addresses
  Last Active:         X days ago
  Avg Gas Used/Tx:     XX,XXX

🔍 RISK FLAGS
  [✅ CLEAN | ⚠️ CAUTION | 🚨 FLAG] — explanation

💡 AGENT RECOMMENDATION
  [Personalized next steps based on the data]

══════════════════════════════════════
```

### Activity Score Algorithm

The agent calculates a score 0–100:

| Metric | Points |
|---|---|
| >10 transactions | +20 |
| >50 transactions | +15 (bonus) |
| Interacted with ≥3 unique contracts | +20 |
| Holds ≥2 different tokens | +15 |
| Active within last 30 days | +20 |
| Non-zero native balance | +10 |

### Risk Flag Logic

| Condition | Flag |
|---|---|
| Zero transactions | 🚨 Inactive wallet |
| Zero native balance + pending send attempt | 🚨 Insufficient gas |
| Only 1 tx (possible dust/spam address) | ⚠️ Low activity |
| All clean | ✅ No issues detected |

---

## Layer 2 — Airdrop Radar

**Trigger phrases:** "airdrop eligibility", "am I eligible", "airdrop checker", "Pharos airdrop", "qualify for airdrop", "how to maximize airdrop", "airdrop score"

The agent scores a wallet against common Pharos ecosystem eligibility patterns and returns actionable recommendations.

### Eligibility Criteria Checklist

```bash
# Run each check and record pass/fail

# Criterion 1: Minimum transaction count
TX_COUNT=$(cast tx-count <ADDRESS> --rpc-url $RPC_URL)
# PASS if TX_COUNT >= 10

# Criterion 2: Has interacted with ≥1 Pharos-native contract
# Check against known contract list in assets/contracts.json
cast call <CONTRACT> "hasInteracted(address)(bool)" <ADDRESS> --rpc-url $RPC_URL

# Criterion 3: Holds native PHRS balance > 0
BALANCE=$(cast balance <ADDRESS> --rpc-url $RPC_URL --ether)
# PASS if BALANCE > 0

# Criterion 4: Has deployed or interacted with a smart contract
# Proxy: tx-count on contracts called
cast tx-count <ADDRESS> --rpc-url $RPC_URL

# Criterion 5: Recently active (within 60 days)
# Compare latest block timestamp to wallet's last tx
```

### Airdrop Score Report Format

```
══════════════════════════════════════
  PHAROS AIRDROP RADAR
  Address: 0x...
══════════════════════════════════════

🎯 ELIGIBILITY SCORE: XX / 100

CRITERIA BREAKDOWN:
  [✅/❌] Minimum 10 transactions ........... +20 pts
  [✅/❌] Pharos contract interaction ........ +25 pts
  [✅/❌] Holds native PHRS .................. +20 pts
  [✅/❌] Smart contract deployment/use ...... +20 pts
  [✅/❌] Active within 60 days .............. +15 pts

📈 LIKELIHOOD TIER:
  90-100: 🟢 HIGH — Strong candidate
  60-89:  🟡 MEDIUM — Solid activity, keep going
  30-59:  🟠 LOW — Needs more onchain history
  0-29:   🔴 VERY LOW — Start now

🛠️ RECOMMENDED ACTIONS:
  [Specific, actionable steps to improve score]
  Example: "Send 3 more transactions to reach the 10 tx threshold"
  Example: "Interact with a deployed contract on testnet"

══════════════════════════════════════
```

### Agent Recommendation Engine

Based on the score, the agent generates personalized action steps:

```
If TX_COUNT < 10:
  → "You need X more transactions. Run a few test transfers or contract calls."

If no contract interaction:
  → "Deploy or interact with any contract. Use pharos-skill-engine's 'Quick Deploy ERC20' to do this in one step."

If inactive > 60 days:
  → "Your last activity was too long ago. Send a transaction today to refresh eligibility."

If score >= 90:
  → "You're in strong shape. Maintain activity and monitor official Pharos airdrop announcements."
```

---

## Layer 3 — DeFi Compose

**Trigger phrases:** "compose DeFi", "put my PHRS to work", "best yield", "swap and stake", "DeFi strategy", "automate DeFi", "multi-step DeFi", "optimize my position"

The agent interprets a high-level financial intent and breaks it into an ordered sequence of onchain operations, confirming with the user before executing.

### Intent Parsing

The agent identifies the user's intent and maps it to an operation chain:

| User Says | Composed Workflow |
|---|---|
| "Put X PHRS to work" | Check balance → Find yield opportunity → Approve → Stake/LP |
| "Swap ETH for USDC and stake" | Check balance → Gas estimate → Swap → Approve USDC → Stake |
| "Airdrop tokens to my community" | Validate addresses → Batch approve → Execute airdrop |
| "Exit my position safely" | Check holdings → Unstake → Swap to stable → Confirm |
| "Maximize testnet points" | Check activity gaps → Fill tx count → Contract interactions → Report |

### Execution Protocol

The agent MUST follow this protocol for all composed workflows:

**Step 1 — Parse & Plan**
```
Agent presents the full execution plan BEFORE running anything:

PROPOSED WORKFLOW:
  Step 1: Check PHRS balance
  Step 2: Estimate gas for all steps (total: ~X PHRS)
  Step 3: [Operation A]
  Step 4: [Operation B]
  Step 5: Confirm final state

Estimated total gas: ~X PHRS
Proceed? (yes/no)
```

**Step 2 — Sequential Execution**

Each step uses the base pharos-skill-engine commands. The agent reads network config from `assets/networks.json` and executes in order, stopping immediately on any error.

```bash
# Example: Swap + Stake workflow
# Step 1: Balance check
cast balance $ADDRESS --rpc-url $RPC_URL --ether

# Step 2: Gas estimation for full flow
cast estimate $CONTRACT "approve(address,uint256)" $SPENDER $AMOUNT --rpc-url $RPC_URL

# Step 3: ERC-20 approval
cast send $TOKEN_CONTRACT "approve(address,uint256)" $SPENDER $AMOUNT \
  --rpc-url $RPC_URL --private-key $PRIVATE_KEY

# Step 4: Execute swap
cast send $DEX_CONTRACT "swap(address,address,uint256,uint256)" \
  $TOKEN_IN $TOKEN_OUT $AMOUNT_IN $MIN_OUT \
  --rpc-url $RPC_URL --private-key $PRIVATE_KEY

# Step 5: Stake result
cast send $STAKING_CONTRACT "stake(uint256)" $AMOUNT_OUT \
  --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

**Step 3 — Execution Report**

After all steps complete:

```
══════════════════════════════════════
  DEFI COMPOSE — EXECUTION REPORT
══════════════════════════════════════

✅ Step 1: Balance verified (X.XX PHRS)
✅ Step 2: Approval confirmed — tx: 0x...
✅ Step 3: Swap executed — tx: 0x...
✅ Step 4: Staked successfully — tx: 0x...

FINAL STATE:
  PHRS spent:    X.XX
  Gas used:      X.XX PHRS
  Position held: X tokens staked @ [protocol]
  Explorer:      https://pharosscan.io/tx/...

══════════════════════════════════════
```

### Safety Rules for Compose

- NEVER execute write operations without showing the full plan and getting explicit user confirmation
- NEVER proceed to the next step if the previous step failed
- ALWAYS show gas estimates before execution
- ALWAYS display target network prominently (testnet vs mainnet)
- On mainnet: require confirmation TWICE before any write operation

---

## Combined Usage Examples

```
# Full wallet audit before doing anything
"Analyze wallet 0x1234...abcd on Pharos testnet and tell me what to do next"

# Airdrop readiness check
"Score my wallet 0x1234...abcd for Pharos airdrop eligibility and tell me how to maximize my chances"

# Intent-driven DeFi
"I have 5 PHRS sitting idle. Compose the best testnet activity workflow to maximize my onchain footprint"

# Combined analysis + action
"Check if 0x1234...abcd is airdrop eligible, then if score is below 60, compose a workflow to improve it"
```

---

## Asset Files Required

This skill references the following asset files (included in repo):

```
pharos-agent-suite/
├── SKILL.md                    ← this file
├── assets/
│   ├── tokens.json             ← known ERC-20 token contracts on Pharos
│   └── contracts.json          ← known Pharos ecosystem contracts
├── demo/
│   └── index.html              ← interactive demo interface
└── README.md
```

---

## Error Handling

| Scenario | Handling |
|---|---|
| Address has 0 transactions | Report as inactive, skip activity-based checks, still show recommendations |
| Token contract call reverts | Skip that token, note "contract unresponsive" in report |
| Compose step fails mid-flow | Halt immediately, report which step failed, show recovery options |
| Network config missing | Prompt to install base pharos-skill-engine first |
| Private key not set for compose | Prompt before any write step, do not proceed |

---

## Security

- Read-only operations (Wallet Health, Airdrop Radar) require NO private key
- DeFi Compose write operations follow all security rules from pharos-skill-engine
- Never log or expose private keys
- Always confirm target network before write operations
- Mainnet compose requires double confirmation
