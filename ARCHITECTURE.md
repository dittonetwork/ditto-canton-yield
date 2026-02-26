# Ditto Vault — Technical Architecture

> Canton Network · CIP-56 · Daml Smart Contracts

---

## Table of Contents

- [1. System Overview](#1-system-overview)
- [2. Canton Domain Model](#2-canton-domain-model)
- [3. CIP-56 Token Integration](#3-cip-56-token-integration)
- [4. NAV Oracle & Pricing](#4-nav-oracle--pricing)
- [5. Lifecycle Flows](#5-lifecycle-flows)
- [6. Liquidity Pool](#6-liquidity-pool)
- [7. Cross-Chain Bridge](#7-cross-chain-bridge)
- [8. Backend Services](#8-backend-services)
- [9. Security Model](#9-security-model)
- [10. Deployment Architecture](#10-deployment-architecture)
- [11. References](#11-references)

---

## 1. System Overview

Ditto Vault operates across two execution environments connected by a backend orchestration layer:

- **Canton Network** — tokenization, accounting, and settlement via Daml smart contracts and CIP-56 token standard
- **EVM Chains** — yield generation across DeFi money markets (Aave, Morpho, Fluid, Spark), secured by Ditto Network's 16 decentralized operators

The Canton side never touches actual funds in the MVP. It tracks NAV, manages vault shares (dvUSDC), and provides institutional-grade workflows. The EVM side is the existing yield engine. A backend service bridges state between the two.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Canton Network (Daml)                        │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │  VaultState   │  │  dvUSDC      │  │  DepositOffer /           │ │
│  │  NAV, shares, │  │  CIP-56      │  │  WithdrawRequest /        │ │
│  │  sharePrice   │  │  Holdings    │  │  YieldClaim               │ │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬──────────────┘ │
│         │                 │                        │                │
│         └─────────────────┼────────────────────────┘                │
│                           │                                         │
│                    JSON Ledger API                                   │
└───────────────────────────┼─────────────────────────────────────────┘
                            │
                 ┌──────────┴──────────┐
                 │  Backend Services    │
                 │                      │
                 │  • Lifecycle Bot     │
                 │  • NAV Oracle        │
                 │  • LP Market Maker   │
                 └──────────┬──────────┘
                            │
              ┌─────────────┴─────────────┐
              │  EVM Yield Engine          │
              │                            │
              │  Aave · Morpho · Fluid ·   │
              │  Spark                     │
              │                            │
              │  Secured by 16 Ditto       │
              │  operators ($200M+ TVL)    │
              └────────────────────────────┘
```

---

## 2. Canton Domain Model

All on-chain state lives in Daml contracts deployed to the Canton participant node. The contracts follow Canton's ledger model: contracts are immutable, exercised choices consume the input contract and create new ones (UTXO-style).

### 2.1 VaultState

Singleton contract owned by the vault operator (curator). Tracks pool-level accounting.

```daml
template VaultState
  with
    vaultId       : Text
    operator      : Party
    totalShares   : Decimal
    currentNAV    : Decimal      -- total vault value in USDC terms
    sharePrice    : Decimal      -- currentNAV / totalShares
    lastNAVUpdate : Time
    feeRateBps    : Int          -- management fee in basis points (50-200)
    isPaused      : Bool         -- emergency pause flag
  where
    signatory operator

    choice UpdateNAV : ContractId VaultState
      with
        newNAV     : Decimal
        updateTime : Time
      controller operator
      do
        assertMsg "Vault is paused" (not isPaused)
        assertMsg "NAV must be non-negative" (newNAV >= 0.0)
        let newPrice = if totalShares > 0.0
              then newNAV / totalShares
              else 1.0
        create this with
          currentNAV = newNAV
          sharePrice = newPrice
          lastNAVUpdate = updateTime

    choice RecordMint : ContractId VaultState
      with
        mintedShares  : Decimal
        depositAmount : Decimal
      controller operator
      do
        assertMsg "Shares must be positive" (mintedShares > 0.0)
        create this with
          totalShares = totalShares + mintedShares
          currentNAV  = currentNAV + depositAmount

    choice RecordBurn : ContractId VaultState
      with
        burnedShares   : Decimal
        withdrawAmount : Decimal
      controller operator
      do
        assertMsg "Shares must be positive" (burnedShares > 0.0)
        assertMsg "Cannot burn more than total" (burnedShares <= totalShares)
        create this with
          totalShares = totalShares - burnedShares
          currentNAV  = currentNAV - withdrawAmount

    choice PauseVault : ContractId VaultState
      controller operator
      do create this with isPaused = True

    choice UnpauseVault : ContractId VaultState
      controller operator
      do create this with isPaused = False
```

**Key invariants:**
- `sharePrice = currentNAV / totalShares` (when `totalShares > 0`)
- Only the operator can exercise choices (curator-gated)
- `isPaused` blocks NAV updates and, by extension, all minting/burning

### 2.2 DepositOffer

Created by a user (depositor) to request entry into the vault.

```daml
template DepositOffer
  with
    depositor     : Party
    operator      : Party
    depositAmount : Decimal      -- USDCx amount
    createdAt     : Time
  where
    signatory depositor
    observer operator

    choice AcceptDeposit : ()
      with
        sharesToMint : Decimal
        vaultStateId : ContractId VaultState
      controller operator
      do
        assertMsg "Shares must be positive" (sharesToMint > 0.0)
        -- Operator validates: sharesToMint ≈ depositAmount / sharePrice
        -- Actual mint via BurnMintFactory happens in the backend transaction
        -- RecordMint on VaultState happens in the same transaction
        pure ()

    choice CancelDeposit : ()
      controller depositor
      do pure ()

    choice RejectDeposit : ()
      controller operator
      do pure ()
```

### 2.3 WithdrawRequest

Created by a dvUSDC holder to request redemption.

```daml
template WithdrawRequest
  with
    holder        : Party
    operator      : Party
    sharesToBurn  : Decimal      -- dvUSDC amount
    createdAt     : Time
  where
    signatory holder
    observer operator

    choice AcceptWithdraw : ()
      with
        redemptionAmount : Decimal   -- USDCx to return
        vaultStateId     : ContractId VaultState
      controller operator
      do
        assertMsg "Redemption must be positive" (redemptionAmount > 0.0)
        -- Operator validates: redemptionAmount ≈ sharesToBurn * sharePrice
        -- Actual burn via BurnMintFactory happens in the backend transaction
        -- RecordBurn on VaultState happens in the same transaction
        pure ()

    choice CancelWithdraw : ()
      controller holder
      do pure ()
```

### 2.4 YieldClaim

Allows a user to harvest accrued yield without redeeming their full position.

```daml
template YieldClaim
  with
    holder          : Party
    operator        : Party
    initialShares   : Decimal    -- shares at time of deposit
    currentShares   : Decimal    -- shares held now (same count)
    autoCompound    : Bool       -- if True, reinvest yield
  where
    signatory holder
    observer operator

    choice AcceptClaim : ()
      with
        yieldInUSDCx   : Decimal
        sharesToBurn   : Decimal  -- yield portion expressed in shares
        vaultStateId   : ContractId VaultState
      controller operator
      do
        assertMsg "Yield must be positive" (yieldInUSDCx > 0.0)
        -- If autoCompound: burn yield shares, immediately re-mint at current price
        -- If not: burn yield shares, transfer USDCx to holder
        pure ()
```

---

## 3. CIP-56 Token Integration

dvUSDC is a CIP-56-compliant token, making it interoperable with any Canton wallet, DvP settlement flow, and third-party Canton applications.

### 3.1 Token Registration

The vault operator registers as an **Instrument Admin** in the Canton Registry Utility:

1. **Create `AllocationFactory`** — enables users to lock dvUSDC holdings for transfer offers and DvP settlement
2. **Create `TransferRule`** — defines credential requirements for peer-to-peer dvUSDC transfers

Registration makes dvUSDC discoverable in Canton wallets and enables activity marker generation.

### 3.2 Instrument Metadata

```
Instrument ID    : dvUSDC
Instrument Admin : <operator-party-id>
Token Standard   : CIP-56
Decimals         : 6
Description      : Ditto Vault yield-bearing USDC token
```

### 3.3 Minting (on deposit)

```
User creates DepositOffer
  → Operator calls BurnMintFactory_BurnMint
    → action = Mint
    → owner  = depositor party
    → amount = depositAmount / sharePrice
  → New Holding contract created for depositor
  → Activity marker generated (MintOffer_Accept)
```

### 3.4 Burning (on withdrawal)

```
User creates WithdrawRequest
  → Operator fetches user's dvUSDC Holdings via HoldingV1 interface
  → Operator calls BurnMintFactory_BurnMint
    → action = Burn
    → holdings = user's dvUSDC holding contract IDs
    → amount = sharesToBurn
  → Holding contracts consumed
  → Activity marker generated (BurnOffer_Accept)
```

### 3.5 Transfers

Standard CIP-56 peer-to-peer transfers via `TransferFactory_Transfer`:

```
Sender creates TransferInstruction
  → Receiver accepts
  → Sender's Holding consumed, new Holding created for receiver
  → Activity marker generated (TransferInstruction_Accept)
```

### 3.6 Holdings (UTXO Model)

CIP-56 holdings follow a UTXO model. Each mint creates a new holding contract. Multiple deposits create multiple holdings for the same user.

**Fragmentation management:**
- `MergeDelegation` allows the operator to merge multiple small holdings into one, reducing state and earning additional activity markers
- The backend periodically scans for users with >N holdings and triggers merges

---

## 4. NAV Oracle & Pricing

### 4.1 NAV Update Flow

```
┌──────────────────┐         ┌──────────────────┐        ┌────────────────┐
│  EVM Yield Engine │         │  NAV Oracle       │        │  VaultState    │
│  (Aave, Morpho,  │◄────────│  (Backend)        │───────►│  (Canton)      │
│   Fluid, Spark)  │  read   │                    │ write  │                │
└──────────────────┘  TVL    └──────────────────┘        └────────────────┘
```

1. NAV Oracle reads total vault value from EVM vault contracts via RPC
2. Applies management fee accrual: `netNAV = grossNAV - accruedFee`
3. Exercises `UpdateNAV` on VaultState with the net NAV
4. VaultState recalculates `sharePrice = currentNAV / totalShares`

### 4.2 Update Frequency

| Phase | Frequency | Rationale |
|---|---|---|
| MVP | Every 15 minutes | Sufficient for demo, generates ~96 NAV updates/day |
| Production | Every block (~12s) or on significant NAV change (>0.1%) | Real-time accuracy for institutional users |

### 4.3 Share Price Calculation

```
sharePrice = currentNAV / totalShares

On deposit:
  sharesToMint = depositAmount / sharePrice

On withdrawal:
  redemptionAmount = sharesToBurn × sharePrice

Yield per user:
  yieldValue = (currentSharePrice - entrySharePrice) × userShares
```

### 4.4 Fee Accrual

Management fee is accrued continuously and deducted from NAV before posting:

```
annualFeeRate = feeRateBps / 10000
timeFraction  = secondsSinceLastUpdate / secondsPerYear
accruedFee    = grossNAV × annualFeeRate × timeFraction
netNAV        = grossNAV - accruedFee
```

---

## 5. Lifecycle Flows

### 5.1 Deposit Flow

```
User                    Frontend              Backend (Operator)           Canton
 │                         │                         │                       │
 │── deposit(amount) ─────►│                         │                       │
 │                         │── create DepositOffer ─►│                       │
 │                         │                         │── create DepositOffer ►│
 │                         │                         │                       │
 │                         │                         │◄── DepositOffer cid ───│
 │                         │                         │                       │
 │                         │                         │── read VaultState ────►│
 │                         │                         │◄── sharePrice ────────│
 │                         │                         │                       │
 │                         │                         │   shares = amount /    │
 │                         │                         │           sharePrice   │
 │                         │                         │                       │
 │                         │                         │── ATOMIC TRANSACTION: │
 │                         │                         │   1. AcceptDeposit    │
 │                         │                         │   2. BurnMintFactory  │
 │                         │                         │      Mint(shares)     │
 │                         │                         │   3. RecordMint       │
 │                         │                         │──────────────────────►│
 │                         │                         │                       │
 │                         │                         │◄── Holding(dvUSDC) ───│
 │◄── confirmation ────────│◄── tx result ──────────│                       │
```

All three operations (accept deposit, mint dvUSDC, record mint) execute in a **single atomic Canton transaction**, guaranteeing consistent state.

### 5.2 Withdrawal Flow

```
User                    Frontend              Backend (Operator)           Canton
 │                         │                         │                       │
 │── withdraw(shares) ────►│                         │                       │
 │                         │── create WithdrawReq ──►│                       │
 │                         │                         │── create WithdrawReq ►│
 │                         │                         │                       │
 │                         │                         │── read VaultState ────►│
 │                         │                         │◄── sharePrice ────────│
 │                         │                         │                       │
 │                         │                         │   redemption = shares  │
 │                         │                         │              × price   │
 │                         │                         │                       │
 │                         │                         │── ATOMIC TRANSACTION: │
 │                         │                         │   1. AcceptWithdraw   │
 │                         │                         │   2. BurnMintFactory  │
 │                         │                         │      Burn(shares)     │
 │                         │                         │   3. RecordBurn       │
 │                         │                         │──────────────────────►│
 │                         │                         │                       │
 │◄── USDCx returned ─────│◄── tx result ──────────│                       │
```

### 5.3 Yield Claim Flow

```
User                    Backend (Operator)           Canton
 │                         │                           │
 │── claimYield() ────────►│                           │
 │                         │── read user Holdings ────►│
 │                         │── read VaultState ───────►│
 │                         │                           │
 │                         │   yieldShares = shares ×  │
 │                         │     (1 - entryPrice /     │
 │                         │          currentPrice)    │
 │                         │                           │
 │                         │── ATOMIC TRANSACTION:     │
 │                         │   1. AcceptClaim          │
 │                         │   2. Burn(yieldShares)    │
 │                         │   3. RecordBurn           │
 │                         │   4. Transfer USDCx       │
 │                         │      (or re-mint if       │
 │                         │       autoCompound)       │
 │                         │──────────────────────────►│
 │                         │                           │
 │◄── yield distributed ──│                           │
```

---

## 6. Liquidity Pool

An on-chain secondary market for instant dvUSDC exits without waiting for the withdrawal queue.

### 6.1 Pool Design

```daml
template LiquidityPool
  with
    operator       : Party
    dvUSDCReserve  : Decimal     -- dvUSDC in pool
    usdcxReserve   : Decimal     -- USDCx in pool
    swapFeeBps     : Int         -- 10-30 bps
    lastRebalance  : Time
  where
    signatory operator

    choice SwapDvUSDCForUSDCx : ContractId LiquidityPool
      with
        seller       : Party
        dvUSDCAmount : Decimal
      controller operator
      do
        let fee       = dvUSDCAmount * (intToDecimal swapFeeBps / 10000.0)
            netAmount = dvUSDCAmount - fee
            usdcxOut  = netAmount * (usdcxReserve / dvUSDCReserve)
        assertMsg "Insufficient USDCx liquidity" (usdcxOut <= usdcxReserve)
        create this with
          dvUSDCReserve = dvUSDCReserve + dvUSDCAmount
          usdcxReserve  = usdcxReserve - usdcxOut

    choice Rebalance : ContractId LiquidityPool
      with
        newDvUSDCReserve : Decimal
        newUSDCxReserve  : Decimal
        rebalanceTime    : Time
      controller operator
      do create this with
          dvUSDCReserve = newDvUSDCReserve
          usdcxReserve  = newUSDCxReserve
          lastRebalance = rebalanceTime
```

### 6.2 Arbitrage Mechanism

When dvUSDC trades below NAV on the LP:

```
1. Arbitrage bot buys discounted dvUSDC from LP
2. Bot submits WithdrawRequest at full NAV price
3. Bot earns spread: NAV price - LP price - fees
4. LP price converges back to NAV
```

When dvUSDC trades above NAV (high demand):

```
1. Bot deposits USDCx via DepositOffer at NAV price
2. Bot sells freshly minted dvUSDC on LP at premium
3. Bot earns spread: LP price - NAV price - fees
4. LP price converges back to NAV
```

Each arbitrage cycle generates multiple CIP-56 transactions (mint, burn, transfer, swap) — all economically motivated.

### 6.3 Transaction Generation

| LP Operation | CIP-56 Transactions Generated |
|---|---|
| Swap dvUSDC → USDCx | Transfer (seller → pool) + Transfer (pool → seller) |
| Arbitrage buy + redeem | Transfer (pool → bot) + Burn (bot dvUSDC) |
| Arbitrage mint + sell | Mint (bot dvUSDC) + Transfer (bot → pool) |
| Rebalance | Transfer(s) for reserve adjustments |

---

## 7. Cross-Chain Bridge

### 7.1 MVP (Phase 1–2)

No actual fund movement. The NAV Oracle reads EVM vault TVL and posts it to Canton. Deposits/withdrawals are accounted on Canton only.

### 7.2 Production (Phase 2+)

Integration with **Circle xReserve** for actual USDCx ↔ USDC bridging:

```
Deposit:
  User deposits USDCx on Canton
    → Backend initiates xReserve withdrawal (Canton → EVM)
    → USDC received on EVM
    → Backend deposits USDC into yield strategies
    → NAV updates on Canton reflect new TVL

Withdrawal:
  User creates WithdrawRequest on Canton
    → Backend withdraws USDC from yield strategies
    → Backend initiates xReserve deposit (EVM → Canton)
    → USDCx received on Canton
    → Backend transfers USDCx to user
    → dvUSDC burned, VaultState updated
```

### 7.3 Bridge Architecture

```
Canton Network                    Circle xReserve                 EVM
┌──────────────┐                 ┌──────────────┐          ┌──────────────┐
│  USDCx       │ ◄──────────── │  Bridge       │ ◄──────  │  USDC        │
│  (Canton)    │  deposit       │  Contract     │  lock    │  (Ethereum)  │
│              │ ──────────── ►│              │ ──────► │              │
│              │  withdraw      │              │  release │              │
└──────────────┘                 └──────────────┘          └──────┬───────┘
                                                                  │
                                                           ┌──────┴───────┐
                                                           │  Ditto Vault │
                                                           │  Allocator   │
                                                           │              │
                                                           │  Aave        │
                                                           │  Morpho      │
                                                           │  Fluid       │
                                                           │  Spark       │
                                                           └──────────────┘
```

---

## 8. Backend Services

The backend runs as a set of services that interact with both Canton (via JSON Ledger API) and EVM (via RPC).

### 8.1 Lifecycle Bot

Polls for pending workflow contracts and processes them atomically.

```
Poll interval: 1-5 seconds

Loop:
  1. Query active DepositOffer contracts
     → For each: calculate shares, execute atomic mint transaction
  2. Query active WithdrawRequest contracts
     → For each: calculate redemption, execute atomic burn transaction
  3. Query active YieldClaim contracts
     → For each: calculate yield, execute claim transaction
  4. Query active LP SwapRequest contracts
     → For each: validate liquidity, execute swap
```

**Atomic transaction composition** (single Ledger API command):
```json
{
  "commands": [
    { "exercise": { "template": "DepositOffer", "choice": "AcceptDeposit", ... } },
    { "exercise": { "template": "BurnMintFactory", "choice": "BurnMint_Mint", ... } },
    { "exercise": { "template": "VaultState", "choice": "RecordMint", ... } }
  ]
}
```

All three commands execute atomically — if any fails, the entire transaction rolls back.

### 8.2 NAV Oracle

```
Schedule: every 15 minutes (MVP) → every block (production)

1. Read EVM vault contract: totalAssets()
2. Read EVM vault contract: totalSupply() (for internal consistency check)
3. Calculate management fee accrual
4. Post UpdateNAV to VaultState on Canton
5. Log NAV history for APY calculation
```

### 8.3 UTXO Manager

CIP-56 holdings follow a UTXO model. Frequent deposits create holding fragmentation.

```
Schedule: every 1 hour

1. Query all dvUSDC Holdings grouped by owner
2. For owners with > 5 Holdings:
   → Exercise MergeDelegation to consolidate into single Holding
3. Each merge generates an activity marker
```

### 8.4 LP Market Maker

```
Schedule: continuous (event-driven)

1. Monitor LP pool price vs NAV oracle price
2. If LP price < NAV - threshold:
   → Buy dvUSDC from LP (cheap)
   → Submit WithdrawRequest at NAV (full value)
   → Pocket spread
3. If LP price > NAV + threshold:
   → Submit DepositOffer at NAV
   → Sell dvUSDC on LP (premium)
   → Pocket spread
```

---

## 9. Security Model

### 9.1 Canton Security (Daml)

| Control | Implementation |
|---|---|
| **Authorization** | Every contract choice has explicit `controller` — only the designated party can exercise |
| **Signatory model** | `DepositOffer` signed by depositor, observed by operator. Operator cannot forge deposits |
| **Atomic execution** | Multi-contract transactions are all-or-nothing. No partial state updates |
| **Privacy** | Canton's sub-transaction privacy ensures parties only see contracts they are signatories/observers of |
| **Audit trail** | Every contract creation and archival is recorded on the Canton ledger with full provenance |

### 9.2 Operator Controls

| Risk | Mitigation |
|---|---|
| **NAV manipulation** | Only operator can update NAV; backend reads directly from EVM contracts (no manual input) |
| **Unauthorized minting** | Minting requires a DepositOffer signed by the depositor — operator cannot self-mint |
| **Fund extraction** | Withdrawal requires a WithdrawRequest signed by the holder — operator cannot unilaterally burn |
| **Emergency** | `PauseVault` choice halts all operations immediately |
| **Operator key compromise** | Future: multi-party `VaultPolicy` with quorum threshold for critical operations |

### 9.3 EVM Security

| Control | Implementation |
|---|---|
| **Operator decentralization** | 16 Ditto operators across Eigenlayer and Symbiotic secure all vault transactions |
| **Strategy constraints** | Vault allocator restricts deposits to whitelisted protocols (Aave, Morpho, Fluid, Spark) |
| **Autonomous execution** | Yield generation runs autonomously — no manual intervention required |
| **TVL backing** | $200M+ in TVL across the operator set provides economic security |

### 9.4 Controls Against Non-Bona-Fide Transactions

Per Canton Featured App requirements:

- Every CIP-56 operation (deposit, withdrawal, claim, transfer, LP swap) serves a genuine economic purpose
- Operator-gated acceptance prevents wash trading
- NAV oracle is read-only from EVM — no artificial inflation
- Transfer rules enforce credential validation
- All contracts have explicit signatory/observer authorization — no anonymous operations

---

## 10. Deployment Architecture

### 10.1 Infrastructure

```
┌─────────────────────────────────────────────────┐
│  Canton Participant Node                         │
│  (Ditto-operated validator on DevNet/MainNet)    │
│                                                   │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │  Daml Engine │  │  JSON Ledger API (gRPC)  │  │
│  │  .dar files  │  │  Port 5001               │  │
│  └─────────────┘  └──────────┬───────────────┘  │
└──────────────────────────────┼───────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │  Backend Services    │
                    │  (Kubernetes / VM)   │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │ Lifecycle Bot  │  │
                    │  │ NAV Oracle     │  │
                    │  │ UTXO Manager   │  │
                    │  │ LP Market Maker│  │
                    │  └────────────────┘  │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │ API Gateway    │  │
                    │  │ (REST → gRPC)  │  │
                    │  └────────┬───────┘  │
                    └───────────┼──────────┘
                               │
                    ┌──────────┴──────────┐
                    │  Frontend (React)    │
                    │  Hosted on Vercel    │
                    │                      │
                    │  OAuth2 auth via     │
                    │  Canton identity     │
                    └─────────────────────┘
```

### 10.2 Deployment Sequence

```
Phase 1 (DevNet):
  1. daml build → compile .dar packages
  2. Upload .dar to DevNet participant node
  3. Register dvUSDC in Registry Utility (AllocationFactory + TransferRule)
  4. Deploy backend services pointing to DevNet JSON Ledger API
  5. Deploy frontend to Vercel
  6. Initialize VaultState contract with NAV = 0, totalShares = 0

Phase 3 (MainNet):
  1. Security audit of Daml contracts
  2. Upload audited .dar to MainNet participant node
  3. Re-register dvUSDC in MainNet Registry Utility
  4. Migrate backend to production infrastructure
  5. Apply for FeaturedAppRight from Tokenomics Committee
  6. Configure AppRewardConfiguration for activity marker rewards
```

### 10.3 Contract Packages

```
ditto-vault-contracts/
├── daml.yaml
├── daml/
│   ├── DittoVault/
│   │   ├── VaultState.daml
│   │   ├── DepositOffer.daml
│   │   ├── WithdrawRequest.daml
│   │   ├── YieldClaim.daml
│   │   └── LiquidityPool.daml
│   └── Test/
│       ├── VaultStateTest.daml
│       ├── DepositFlowTest.daml
│       ├── WithdrawFlowTest.daml
│       └── YieldClaimTest.daml
└── .dar (compiled output)
```

---

## 11. References

| Resource | Link |
|---|---|
| CIP-56 Specification | [github.com/global-synchronizer-foundation/cips](https://github.com/global-synchronizer-foundation/cips/blob/main/cip-0056/cip-0056.md) |
| Canton Quickstart | [github.com/digital-asset/cn-quickstart](https://github.com/digital-asset/cn-quickstart) |
| Splice Token Standard APIs | [docs.dev.global.canton.network](https://docs.dev.global.canton.network.sync.global/app_dev/token_standard/index.html) |
| BurnMint API | [Splice-Api-Token-BurnMintV1](https://docs.dev.global.canton.network.sync.global/app_dev/api/splice-api-token-burn-mint-v1/Splice-Api-Token-BurnMintV1.html) |
| Splice Wallet Kernel SDK | [github.com/hyperledger-labs/splice-wallet-kernel](https://github.com/hyperledger-labs/splice-wallet-kernel) |
| Registry Utility | [docs.digitalasset.com](https://docs.digitalasset.com/utilities/devnet/overview/registry-user-guide/token-standard.html) |
| Activity Markers | [docs.digitalasset.com](https://docs.digitalasset.com/utilities/devnet/overview/registry-user-guide/activity-markers.html) |
| Circle xReserve | [developers.circle.com](https://developers.circle.com/xreserve/tutorials/deposit-usdc-on-ethereum-for-usdcx-on-canton) |
| Featured App Request | [canton.foundation](https://canton.foundation/featured-app-request/) |
| Daml Documentation | [docs.digitalasset.com](https://docs.digitalasset.com/build/3.4/tutorials/smart-contracts/intro.html) |

---

*Ditto Network — [dittonetwork.io](https://dittonetwork.io) · [@Ditto_Network](https://x.com/Ditto_Network) · [GitHub](https://github.com/dittonetwork)*
