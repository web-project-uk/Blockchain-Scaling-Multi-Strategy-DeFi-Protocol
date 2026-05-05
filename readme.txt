# Blockchain Scaling & Multi-Strategy DeFi Protocol — Design Spec

**Date:** 2025-01-21
**Status:** Design Phase

---

## 1. Architecture Overview

The protocol is a **modular multi-strategy DeFi platform** built on a custom Layer-2 rollup with STARK proof verification. Three distinct strategy modules plug into shared infrastructure for liquidity, governance, and settlement.

```
┌─────────────────────────────────────────────────────────────┐
│                     LAYER 3: GOVERNANCE                     │
│              (Token, Voting, Fee Distribution)              │
├───────────┬──────────────────┬──────────────────────────────┤
│           │                  │                              │
│  ┌────────▼───────┐  ┌──────▼────────┐  ┌─────────────────▼──┐
│  │  SCALING       │  │  PREDICTION   │  │  LENDING           │
│  │  SOLUTION      │  │  MARKET       │  │  PROTOCOL          │
│  │                │  │               │  │                    │
│  │ • STARK Proofs │  │ • AI Odds     │  │ • Yield Vaults     │
│  │ • Batch Tx     │  │ • Oracle Feed │  │ • Auto-Compound    │
│  │ • Bridge       │  │ • Pool AMM    │  │ • Risk Tranches    │
│  │ • State Mgmt   │  │ • Settlement  │  │ • Liquidation      │
│  └────────┬───────┘  └──────┬────────┘  └────────┬───────────┘
│           │                 │                     │           │
├───────────▼─────────────────▼─────────────────────▼───────────┤
│                   SHARED INFRASTRUCTURE                       │
│          (Liquidity Router, Oracle Aggregator,                │
│           Proof Verifier, Access Control)                     │
├───────────────────────────────────────────────────────────────┤
│                   SETTLEMENT LAYER (L2 ROLLUP)                │
│            (Batch Processor, State Root, DA)                  │
├───────────────────────────────────────────────────────────────┤
│                   HOST CHAIN (Ethereum L1)                    │
│               (Bridge Contract, Verifier)                     │
└───────────────────────────────────────────────────────────────┘
```

### Core Design Principles
- **Modularity:** Each strategy is an independent contract set with clean interfaces
- **Shared liquidity:** A unified liquidity router allows capital efficiency across strategies
- **STARK-native:** All batch operations use STARK proofs for validity
- **Composability:** Strategies can interact (e.g., prediction market collateral earns yield in lending vaults)

---

## 2. Strategy 1 — Scaling Solution

### 2.1 Overview
A custom ZK-rollup using STARK proofs for validity. Processes transactions off-chain in batches, posts state roots to L1, and uses a STARK verifier contract for proof validation.

### 2.2 Components

**Off-Chain:**
- **Sequencer:** Orders and executes transactions, produces batches
- **Prover:** Generates STARK proofs for batch validity
- **State Manager:** Maintains Merkle tree of all account states

**On-Chain (L2 contracts on host chain):**
- **Bridge Contract:** Lock/unlock tokens between L1 ↔ L2
- **Rollup Contract:** Receives batches, verifies STARK proofs, updates state root
- **Verifier Contract:** STARK proof verification logic
- **Challenge Contract:** Fraud proof window for optimistic fallback

### 2.3 Transaction Flow
```
User → L2 Sequencer → Batch (1000 txs) → Prover (STARK proof)
                                              │
L1 Rollup Contract ← State Root + Proof ──────┘
        │
   Verifier Contract validates proof
        │
   State root updated on L1
```

### 2.4 Key Parameters
| Parameter | Value |
|-----------|-------|
| Batch size | 500–2000 transactions |
| Proof generation time | ~2 minutes target |
| Finality on L1 | ~10 minutes (proof + L1 block) |
| Transaction throughput | ~2000 TPS target |
| Fee model | Gas-based, 100x cheaper than L1 |

### 2.5 STARK Proof Structure
- **Arithmetic:** Cairo-style AIR (Algebraic Intermediate Representation)
- **Field:** Goldilocks field (64-bit prime, fast arithmetic)
- **Hash:** Poseidon hash (STARK-friendly)
- **Commitment:** FRI (Fast Reed-Solomon Interactive Oracle Proofs)
- **Recursive composition:** Allow proof aggregation for faster L1 verification

### 2.6 Bridge Mechanism
- **Deposit (L1→L2):** Lock tokens in L1 bridge contract → Mint on L2 after proof finality
- **Withdrawal (L2→L1):** Burn on L2 → Generate inclusion proof → Unlock on L1
- **Emergency exit:** Force-withdraw via L1 contract if sequencer is down (7-day delay)

---

## 3. Strategy 2 — AI Prediction Market

### 3.1 Overview
A decentralized prediction market where AI models generate initial odds, users trade outcome tokens, and oracle feeds settle markets. Dynamic fees adjust based on market efficiency.

### 3.2 Components

**Market Engine:**
- **Market Factory:** Creates markets with defined outcomes, resolution time, oracle source
- **Outcome AMM:** Constant-product AMM specialized for binary/multi-outcome tokens
- **Odds Engine:** AI model interface that sets initial probabilities and updates them
- **Settlement Module:** Resolves markets via oracle, distributes payouts

**AI Integration:**
- **Model Registry:** On-chain registry of AI model attestations
- **Odds Oracle:** Off-chain AI model computes odds → signed attestation → on-chain update
- **Confidence Weighting:** Multiple AI models can provide odds, weighted by historical accuracy

**Oracle System:**
- **Primary:** Decentralized oracle network (Chainlink-style)
- **Secondary:** Multi-sig committee for subjective outcomes
- **Fallback:** DAO governance vote for disputed outcomes

### 3.3 Market Lifecycle
```
1. CREATE    → Market factory deploys market contract with parameters
2. INIT      → AI oracle provides initial odds (e.g., 65/35)
3. TRADE     → Users buy/sell outcome tokens via AMM
4. UPDATE    → AI oracle updates odds periodically (every hour)
5. LOCK      → Market locks at resolution time, no more trades
6. RESOLVE   → Oracle reports actual outcome
7. SETTLE    → Winning token holders redeem at 1:1, losers get 0
```

### 3.4 Dynamic Fee Model
```
Fee = BaseFee × (1 + VolatilityPenalty) × (1 - LiquidityBonus)

Where:
  BaseFee = 2% (adjustable by governance)
  VolatilityPenalty = |current_odds - initial_odds| / initial_odds × 0.5
  LiquidityBonus = min(tvl / $1M, 0.5)  // up to 50% discount for deep markets
```

- High-odds movement = higher fees (discourages manipulation)
- Deep liquidity = lower fees (rewards market makers)
- Fee revenue split: 50% LPs, 30% treasury, 20% AI model providers

### 3.5 Market Types
- **Binary:** Yes/No (sports, elections, price targets)
- **Categorical:** Multiple outcomes (who wins tournament)
- **Scalar:** Range-based (what will price be at time T)
- **Parlays:** Combined markets with multiplied odds

---

## 4. Strategy 3 — Decentralized Lending Protocol

### 4.1 Overview
Yield-generating vaults where depositors earn interest from borrowers. Features auto-compounding, risk-based tranching, and algorithmic interest rates.

### 4.2 Components

**Vault System:**
- **Vault Factory:** Deploys new vault instances per asset/risk level
- **Interest Rate Model:** Algorithmic rates based on utilization
- **Auto-Compounder:** Harvests yield and reinvests automatically
- **Liquidation Engine:** Monitors health factors, triggers liquidations

**Risk Management:**
- **Risk Tranches:** Senior (low risk, lower yield) / Junior (high risk, higher yield)
- **Collateral Registry:** Approved collateral assets with LTV ratios
- **Price Oracle Integration:** Real-time collateral valuation
- **Bad Debt Socializer:** Losses distributed across tranches by seniority

### 4.3 Interest Rate Model
```
If utilization < optimal_utilization (80%):
    rate = base_rate + utilization × slope1 / optimal_utilization
Else:
    rate = base_rate + slope1 + (utilization - optimal_utilization) × slope2 / (1 - optimal_utilization)

Parameters:
    base_rate = 2% APR
    slope1 = 10% (gradual increase up to 80% utilization)
    slope2 = 100% (steep increase above 80% to discourage over-utilization)
```

### 4.4 Auto-Compound Mechanism
```
Depositor → Vault → Lending Pool → Interest Accrues
                                         │
                              Compounder Bot (off-chain)
                                         │
                              Claim yield → Re-deposit
                                         │
                              Share price increases (no extra tx needed)
```

- Share tokens (cTokens) represent vault deposits
- Share price increases as interest accrues (no manual compounding needed)
- Off-chain compounder triggers harvest when gas-efficient
- Compounder earns a small performance fee (5% of extra yield)

### 4.5 Liquidation Process
```
Health Factor = (collateral_value × liquidation_threshold) / debt_value

If HF < 1.0:
    1. Liquidator repays portion of debt
    2. Liquidator receives collateral at discount (5-10%)
    3. Remaining debt re-assessed
    4. If HF still < 1.0, repeat until HF > 1.0 or position fully liquidated
```

### 4.6 Risk Tranche Structure
| Tranche | Priority | APY Range | Risk | Use Case |
|---------|----------|-----------|------|----------|
| Senior | First claim on interest | 3-8% | Low | Conservative yield seekers |
| Mezzanine | Second claim | 8-15% | Medium | Balanced risk/reward |
| Junior | Residual claim | 15-40% | High | Yield farmers, risk-tolerant |

---

## 5. Shared Infrastructure

### 5.1 Liquidity Router
- Routes capital between strategies based on yield opportunities
- Users deposit into a unified pool, receive strategy allocation tokens
- Rebalancing happens via governance or algorithmic triggers

### 5.2 Oracle Aggregator
- Combines price feeds from multiple sources (Chainlink, Pyth, TWAP)
- Median price with outlier rejection
- Staleness checks (price must be updated within threshold)

### 5.3 STARK Proof Verifier (Shared)
- Single verifier contract used by scaling solution and cross-strategy settlements
- Supports recursive proof verification
- Gas-optimized with precompiled curve operations

### 5.4 Access Control
- Role-based: Admin, Operator, Guardian, Upgrader
- Timelock on critical operations (48-hour delay for upgrades)
- Emergency pause functionality (Guardian role)

---

## 6. Contract Structure

```
contracts/
├── core/
│   ├── AccessControl.sol          # Role-based permissions
│   ├── Pausable.sol               # Emergency pause
│   ├── UpgradeableProxy.sol       # Proxy pattern for upgradability
│   └── ReentrancyGuard.sol        # Reentrancy protection
│
├── scaling/
│   ├── Bridge.sol                 # L1 ↔ L2 token bridge
│   ├── Rollup.sol                 # Batch submission + state root
│   ├── STARKVerifier.sol          # Proof verification
│   ├── StateTree.sol              # Merkle state management
│   └── Challenge.sol              # Fraud proof window
│
├── prediction/
│   ├── MarketFactory.sol          # Market creation
│   ├── Market.sol                 # Individual market logic
│   ├── OutcomeAMM.sol             # Token trading (binary/multi)
│   ├── OddsOracle.sol             # AI odds feed interface
│   └── Settlement.sol             # Market resolution + payout
│
├── lending/
│   ├── VaultFactory.sol           # Vault deployment
│   ├── Vault.sol                  # Individual vault (deposit/withdraw/compound)
│   ├── InterestRateModel.sol      # Utilization-based rates
│   ├── LiquidationEngine.sol      # Health factor monitoring + liquidation
│   ├── TrancheManager.sol         # Risk tranche logic
│   └── Compounder.sol             # Auto-compound trigger
│
├── shared/
│   ├── LiquidityRouter.sol        # Cross-strategy capital routing
│   ├── OracleAggregator.sol       # Multi-source price feeds
│   ├── StrategyRegistry.sol       # Registered strategy modules
│   └── FeeDistributor.sol         # Revenue distribution
│
├── governance/
│   ├── GovernanceToken.sol        # ERC-20 governance token
│   ├── Governor.sol               # Proposal + voting
│   ├── Timelock.sol               # Execution delay
│   └── Treasury.sol               # Protocol treasury
│
└── interfaces/
    ├── IStrategy.sol              # Strategy module interface
    ├── IOracle.sol                # Oracle interface
    ├── IBridge.sol                # Bridge interface
    └── IVault.sol                 # Vault interface
```

---

## 7. Tokenomics

### 7.1 Governance Token (SCALE)
- **Supply:** 1,000,000,000 SCALE
- **Distribution:**
  - 40% — Community incentives (emissions over 4 years)
  - 20% — Team & advisors (2-year vest, 6-month cliff)
  - 15% — Treasury (governance-controlled)
  - 10% — Liquidity mining
  - 10% — Early investors (1-year vest)
  - 5% — Airdrop to early users

### 7.2 Utility
- **Governance:** Vote on protocol parameters, strategy activation, fee changes
- **Fee sharing:** Stakers receive protocol fee revenue
- **Boosted yields:** SCALE stakers get higher lending APY and lower prediction fees
- **Sequencer rights:** Top SCALE holders can propose to run sequencer nodes

### 7.3 Fee Flow
```
All Protocol Fees → Fee Distributor
    ├── 40% → SCALE Stakers
    ├── 30% → Treasury (development, grants)
    ├── 20% → Strategy-specific LPs
    └── 10% → Buyback & Burn
```

---

## 8. Risk Considerations

### 8.1 Technical Risks
| Risk | Mitigation |
|------|-----------|
| STARK proof bug | Multiple audit rounds, formal verification of verifier |
| Smart contract exploit | Bug bounty program, staged rollout with caps |
| Sequencer centralization | Decentralized sequencer set, forced inclusion mechanism |
| Oracle manipulation | Multi-source aggregation, TWAP fallback |
| AI model manipulation | Model attestation, confidence decay on bad predictions |

### 8.2 Economic Risks
| Risk | Mitigation |
|------|-----------|
| Bank run on lending | Withdrawal queue, reserve fund, gradual deleveraging |
| Prediction market manipulation | Position limits, dynamic fees, AI anomaly detection |
| Token price collapse | Emission schedule control, treasury buybacks |
| Bad debt in lending | Risk tranches absorb losses junior-first |

### 8.3 Operational Risks
| Risk | Mitigation |
|------|-----------|
| Sequencer downtime | Force-exit via L1, backup sequencer rotation |
| Oracle downtime | Fallback oracle chain, market pause on stale data |
| Governance attack | Quorum requirements, timelock, guardian veto |

---

## 9. Phased Rollout

| Phase | Timeline | Scope |
|-------|----------|-------|
| Phase 1 | Months 1-3 | Core infrastructure, STARK verifier, bridge contract |
| Phase 2 | Months 3-5 | Scaling solution MVP (testnet), basic L2 functionality |
| Phase 3 | Months 5-7 | Lending protocol (single-asset vaults, no tranches) |
| Phase 4 | Months 7-9 | Prediction market (binary markets, single AI oracle) |
| Phase 5 | Months 9-12 | Full integration, risk tranches, multi-oracle, governance launch |
| Phase 6 | Months 12+ | Decentralized sequencer, advanced market types, cross-chain |

---

## 10. Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Proof system | STARK (not SNARK) | No trusted setup, quantum-resistant, better for recursive composition |
| Field | Goldilocks (64-bit) | Fastest arithmetic for STARK provers |
| Hash | Poseidon | STARK-native, 10-100x cheaper than Keccak in-circuit |
| Market AMM | Constant product | Simple, battle-tested (Uniswap model), works for outcome tokens |
| Interest model | Double-slope | Standard in DeFi (Aave/Compound model), proven at scale |
| Risk tranching | Senior/Mezz/Junior | Familiar structure from traditional finance, clear risk separation |
| Oracle | Multi-source aggregator | No single point of failure, median with outlier rejection |
| Upgradability | Proxy pattern + timelock | Allows bug fixes while protecting users from instant changes |

---

## Next Steps

After approval:
1. Write implementation plan with contract-by-contract breakdown
2. Define interfaces (IStrategy, IOracle, IVault, IBridge)
3. Specify STARK circuit constraints for the verifier
4. Detail the AI oracle attestation format
