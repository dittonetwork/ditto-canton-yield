# Ditto Vault

**Asynchronous Curator Vault for Yield Strategies on Canton Network**

*Proposal for Canton Foundation — February 2026*

---

## Overview

Ditto Vault bridges risk-adjusted yield from EVM DeFi to Canton Network through a CIP-56-compliant vault token (**dvUSDC**). Users deposit USDCx on Canton and receive dvUSDC — a yield-bearing token backed by yield coming from leading DeFi money markets. Yield generation is fully autonomous and secured by Ditto Network's decentralized operator set of 16 operators across Eigenlayer and Symbiotic, securing over $200M in TVL.

---

## Ecosystem Gap

Canton Network enables institutional-grade tokenization, but stablecoins like USDCx remain idle on-network. Participants lack a compliant, auditable mechanism to deploy capital into yield strategies without leaving the Canton ecosystem — limiting network utility and adoption.

## Proposed Solution

**Ditto Vault** is an asynchronous curator vault that issues **dvUSDC** — a yield-bearing CIP-56 token backed by yield coming from EVM strategies across Aave, Morpho, Fluid, and Spark.

Vault manages allocations while Canton handles tokenization, accounting, and settlement. An asynchronous request-accept model ensures institutional controls, accurate NAV pricing, and full on-chain auditability.

---

## Value to Canton Network

| Contribution | Description |
|---|---|
| **First Yield-Bearing Asset** | Creates the first native yield-bearing token on Canton, giving participants a reason to hold and deploy capital on-network rather than bridging out |
| **Composable Building Block** | dvUSDC as a CIP-56 token can be used as collateral, in lending protocols, or in structured products — enabling other Canton applications to build on top |
| **Network Activity & TVL** | Deposits, withdrawals, yield claims, LP swaps, and NAV updates generate legitimate, economically motivated CIP-56 transactions that grow Canton's network activity |

---

## How It Works

1. **Deposit** — User submits a DepositOffer on Canton. The curator validates, calculates shares at current NAV, and mints dvUSDC via CIP-56 BurnMintFactory. Vault tokens appear in any Canton-compatible wallet.

2. **Yield Accrual** — The curator's backend reads total vault value from the EVM yield engine and updates the NAV oracle on Canton. Share price reflects real yield earned across DeFi strategies.

3. **Yield Claiming** — Users can harvest accrued yield without closing their position. A claim burns the yield portion of dvUSDC and returns USDCx. Optional auto-compound reinvests yield immediately.

4. **Instant Liquidity** — For users who prefer immediate exits, an on-chain liquidity pool allows selling dvUSDC at market price. Market makers keep the pool anchored to NAV through standard arbitrage.

5. **Withdrawal** — Standard path: user submits a WithdrawRequest, the curator burns dvUSDC, and the user receives USDCx at full NAV price.

---

## Why Canton

| Capability | How Ditto Vault Uses It |
|---|---|
| **CIP-56 Token Standard** | dvUSDC integrates with any Canton wallet — holdings, transfers, DvP settlement |
| **Privacy & Permissioning** | Signatory/observer model ensures only authorized parties access sensitive data |
| **Atomic Settlement** | Multi-contract transactions guarantee consistent state across deposit, mint, and accounting |
| **Composability** | Other Canton applications can build on dvUSDC — lending, collateral, structured products |
| **Institutional Controls** | Curator-gated operations with full on-chain auditability for compliance |

---

## Core Architecture

| Layer | Component | Role |
|---|---|---|
| **Canton (Daml)** | VaultState | NAV, total shares, share price accounting |
| **Canton (Daml)** | DepositOffer / WithdrawRequest | Async request-accept workflow contracts |
| **Canton (CIP-56)** | dvUSDC Holdings | Vault token — mint, burn, transfer via token standard |
| **Canton (CIP-56)** | Liquidity Pool | On-chain secondary market for instant exits |
| **Backend** | NAV Oracle | Reads EVM vault TVL, updates Canton share price |
| **Backend** | Lifecycle Automation | Processes deposits, withdrawals, yield distributions |
| **External** | EVM Yield Engine | Aave, Morpho, Fluid, Spark allocations (existing, battle-tested) |

---

## Featured App Alignment

Ditto Vault is designed to meet Canton Featured App requirements from day one:

- [x] Full CIP-56 compliance — dvUSDC uses BurnMintFactory for standard-compatible mint, burn, and transfer operations
- [x] Daml-native smart contracts — VaultState, DepositOffer, and WithdrawRequest implemented entirely in Daml
- [x] Economically motivated transactions — every CIP-56 operation (deposit, withdrawal, claim, LP swap) serves a genuine user need
- [x] Institutional-grade controls — curator-gated operations, signatory/observer permissioning, full on-chain audit trail
- [x] Composable ecosystem asset — dvUSDC is available for lending, collateral, and structured products by other Canton applications
- [x] Existing infrastructure — Ditto Network operates validators on DevNet, TestNet, and MainNet with a live EVM yield engine

---

## Revenue Model

Self-sustaining revenue from protocol operations — no ongoing grant dependency.

| Source | Mechanism |
|---|---|
| **Management Fee** | 0.5–2% annual on AUM, taken from yield |
| **LP Swap Fee** | 0.1–0.3% per swap, paid by seller for instant liquidity |
| **Arbitrage Revenue** | Market-making spread on LP rebalancing |

---

## Roadmap

| Phase | Scope |
|---|---|
| **Phase 1 — MVP** | Daml contracts (VaultState, DepositOffer, WithdrawRequest), NAV oracle, dvUSDC minting via CIP-56, basic web UI on DevNet |
| **Phase 2 — Bridge Integration** | Circle xReserve integration for USDCx bridging to/from EVM, end-to-end deposit and withdrawal flow |
| **Phase 3 — Featured App Launch** | Committee review, CIP-56 compliance validation, security audit, MainNet deployment |
| **Phase 4 — Arbitrage & LP** | On-chain liquidity pool for instant exits, yield claiming mechanism, market-making infrastructure |

---

## About Ditto Network

**Canton Network Presence**
- Validator operator on Canton DevNet, TestNet, and MainNet
- Active participant in Canton ecosystem since early access
- Familiar with Daml, CIP-56, and Canton SDK

**DeFi Infrastructure Track Record**
- 16 node operators across Eigenlayer and Symbiotic restaking protocols
- Over $200M in TVL secured by a decentralized operator set
- Autonomous yield generation across Aave, Morpho, Fluid, and Spark
- Live, battle-tested cross-chain automation and vault management platform

---

*Prepared for Canton Foundation*

[dittonetwork.io](https://dittonetwork.io) · [@Ditto_Network](https://x.com/Ditto_Network) · [GitHub](https://github.com/dittonetwork)
