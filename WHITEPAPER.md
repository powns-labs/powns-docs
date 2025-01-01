# PoW Name Service (PoWNS) Whitepaper

> **Scarcity through Proof of Work, not Capital.**

**Version**: 1.0  
**Date**: December 29, 2025

---

## Abstract

PoW Name Service (PoWNS) is a decentralized naming system based on Proof of Work. Unlike traditional naming services (such as ENS) that rely on "first-come-first-served + capital bidding," PoWNS determines name scarcity through pure computational competition.

**Core Features**:

- **Fair Distribution**: Names belong to whoever completes the PoW first
- **Bounty Market**: Non-mining users can post bounties for others to mine
- **Dynamic Difficulty**: Dark Gravity Wave algorithm for smooth difficulty adjustment
- **NFT-based**: Names exist as ERC-721 tokens, tradable and transferable
- **Native Token**: $POWNS for governance, incentives, and ecosystem closure

---

## Table of Contents

1. [Background & Motivation](#1-background--motivation)
2. [System Design](#2-system-design)
3. [PoW Mechanism](#3-pow-mechanism)
4. [Difficulty Adjustment System](#4-difficulty-adjustment-system)
5. [Domain Lifecycle](#5-domain-lifecycle)
6. [Bounty Market](#6-bounty-market)
7. [Resolver System](#7-resolver-system)
8. [$POWNS Token Economics](#8-powns-token-economics)
9. [Governance](#9-governance)
10. [Ecosystem Expansion](#10-ecosystem-expansion)
11. [Security Analysis](#11-security-analysis)
12. [Roadmap](#12-roadmap)
13. [Conclusion](#13-conclusion)

---

## 1. Background & Motivation

### 1.1 Problems with Existing Naming Systems

| System    | Problems                                                                  |
| --------- | ------------------------------------------------------------------------- |
| DNS       | Centralized, censorable, requires trust in authorities                    |
| ENS       | Highest bidder wins, capital monopoly on short names, fees go to protocol |
| Handshake | TLD auctions are still capital competition                                |

### 1.2 PoWNS Solution

```mermaid
flowchart LR
    A[Traditional Model] --> B[Money = Scarcity]
    C[PoWNS Model] --> D[Computation = Scarcity]

    B --> E[Capital Monopoly]
    D --> F[Fair Competition]
```

**Design Philosophy**:

1. **Computation determines scarcity**: Names belong to those who invest computational power
2. **Bounty market enables inclusion**: Users who can't mine can participate through bounties
3. **Time earns rights**: Renewal requires continued computational investment
4. **Censorship resistant**: Pure on-chain verification, no centralized control points

---

## 2. System Design

### 2.1 Architecture Overview

```mermaid
flowchart TB
    subgraph "Core Contracts"
        REG[Registry<br/>Domain Registration & Management]
        VER[Verifier<br/>PoW Verification]
        RES[Resolver<br/>Address Resolution]
    end

    subgraph "Economic Layer"
        BOU[Bounty Vault<br/>Mining Market]
        TOK[$POWNS Token<br/>Native Token]
        STA[Staking<br/>Revenue Sharing]
    end

    subgraph "Governance Layer"
        DAO[DAO Treasury]
        GOV[Governor<br/>Proposals & Voting]
    end

    subgraph "External Participants"
        USER[Users]
        MINER[Miners]
    end

    USER -->|Register/Renew| REG
    USER -->|Post Bounty| BOU
    MINER -->|Submit PoW| REG
    MINER -->|Claim Rewards| BOU

    REG <--> VER
    REG --> RES
    REG --> TOK
    TOK --> STA
    STA --> DAO
    GOV --> REG
```

### 2.2 Contract Interface

```solidity
interface IPoWNSRegistry {
    // Register a domain
    function register(
        string calldata name,
        address owner,
        address miner,
        uint256 nonce,
        uint8 years
    ) external;

    // Renew a domain
    function renew(
        string calldata name,
        uint256 nonce,
        uint8 additionalYears
    ) external;

    // Release a domain (refund deposit)
    function release(string calldata name) external;

    // Queries
    function ownerOf(string calldata name) external view returns (address);
    function expiresAt(string calldata name) external view returns (uint256);
    function difficulty(string calldata name) external view returns (uint256);
}
```

---

## 3. PoW Mechanism

### 3.1 Hash Computation

```
hash = SHA256(name || owner || miner || nonce || chainId || registryAddress)
valid = (hash < target)
```

**Parameter Description**:

| Parameter         | Purpose                                                |
| ----------------- | ------------------------------------------------------ |
| `name`            | The domain name to register                            |
| `owner`           | NFT recipient address                                  |
| `miner`           | Transaction submitter address (prevents front-running) |
| `nonce`           | Random number found by miner                           |
| `chainId`         | Prevents cross-chain replay                            |
| `registryAddress` | Prevents contract replay                               |

### 3.2 Why Bind Miner Address?

```mermaid
sequenceDiagram
    participant Miner
    participant Mempool
    participant Attacker
    participant Contract

    Miner->>Miner: Calculate hash(name, owner, miner, nonce)
    Miner->>Mempool: Broadcast transaction
    Attacker->>Mempool: Sees nonce in mempool
    Attacker->>Attacker: Tries to use same nonce
    Note over Attacker: Hash includes original miner address<br/>Attacker cannot use it
    Attacker->>Attacker: Must recalculate ❌
    Contract->>Miner: Verification passed ✅
```

### 3.3 Verification Cost

| Operation          | Gas Cost    |
| ------------------ | ----------- |
| SHA256 hash        | ~60 gas     |
| Target comparison  | ~100 gas    |
| Total verification | ~200 gas    |
| NFT Mint           | ~50,000 gas |

---

## 4. Difficulty Adjustment System

### 4.1 Dark Gravity Wave (DGW)

We adopt the battle-tested Dark Gravity Wave algorithm for smooth and responsive difficulty adjustment.

```mermaid
flowchart TD
    A[New Registration Occurs] --> B[Get timestamps of last 24 registrations]
    B --> C[Calculate average interval T_actual]
    C --> D[Compare to target interval T_target]
    D --> E{T_actual vs T_target}

    E -->|T_actual < T_target| F[Registrations too fast<br/>Increase difficulty]
    E -->|T_actual > T_target| G[Registrations too slow<br/>Decrease difficulty]
    E -->|Close| H[Maintain difficulty]

    F --> I[Adjustment formula]
    G --> I
    H --> I

    I --> J[Apply limits]
    J --> K[Update global difficulty]
```

### 4.2 Difficulty Adjustment Formula

```
T_actual = (timestamp[n] - timestamp[n-24]) / 24
adjustment_ratio = T_target / T_actual
difficulty_new = difficulty_old × adjustment_ratio
```

### 4.3 Safety Limits

| Parameter                 | Value                | Description                                  |
| ------------------------- | -------------------- | -------------------------------------------- |
| **Observation Window**    | 24 registrations     | Balance between responsiveness and stability |
| **Target Interval**       | 600 seconds (10 min) | One domain per 10 minutes                    |
| **Max Single Adjustment** | +25%                 | Prevents difficulty spikes                   |
| **Min Single Adjustment** | -25%                 | Prevents difficulty crashes                  |
| **Difficulty Floor**      | MIN_DIFFICULTY       | Ensures minimum computational barrier        |
| **Difficulty Ceiling**    | MAX_DIFFICULTY       | Prevents system freeze                       |

### 4.4 Difficulty Floor Implementation

```solidity
uint256 constant MIN_DIFFICULTY_BITS = 16;  // At least 16 leading zero bits
uint256 constant MAX_DIFFICULTY_BITS = 240; // Maximum difficulty

function adjustDifficulty() internal {
    // ... DGW calculation ...

    // Apply limits
    uint256 maxNew = (currentDifficulty * 125) / 100; // +25%
    uint256 minNew = (currentDifficulty * 75) / 100;  // -25%

    newDifficulty = clamp(newDifficulty, minNew, maxNew);

    // Enforce floor and ceiling
    newDifficulty = max(newDifficulty, MIN_DIFFICULTY_BITS);
    newDifficulty = min(newDifficulty, MAX_DIFFICULTY_BITS);
}
```

### 4.5 Name Weight (Base Difficulty Adjustment)

Different name lengths/types have different base difficulties:

```
base_difficulty_bits = MIN_DIFFICULTY_BITS + length_weight + charset_weight

length_weight:
  - 3 characters: +32
  - 4 characters: +24
  - 5 characters: +16
  - 6 characters: +8
  - 7+ characters: +0

charset_weight:
  - Numeric only: +8
  - Alphabetic only: +4
  - Mixed: +0
```

### 4.6 Old Result Validity

**Principle**: When difficulty decreases, previously valid results remain valid.

```mermaid
flowchart LR
    subgraph "Case 1: Difficulty Decreases"
        A1[Old hash < Old target] --> B1[Old target < New target]
        B1 --> C1[Old hash < New target ✅]
    end

    subgraph "Case 2: Difficulty Increases"
        A2[Old hash < Old target] --> B2[New target < Old target]
        B2 --> C2[Old hash may > New target ❌]
        C2 --> D2[Recalculation needed]
    end
```

---

## 5. Domain Lifecycle

### 5.1 State Machine

```mermaid
stateDiagram-v2
    [*] --> Active: PoW verified + NFT minted

    Active --> Active: Renewal successful
    Active --> Expired: Not renewed before expiry
    Active --> Released: Owner voluntarily releases

    Expired --> Active: Renewed during grace period
    Expired --> GracePeriod: Past expiration date

    GracePeriod --> Active: Original owner renews
    GracePeriod --> Auction: Grace period ends (90 days)

    Auction --> Active: New PoW successful

    Released --> [*]: Deposit refunded
    Released --> Active: New PoW successful
```

### 5.2 State Descriptions

| State           | Description              | Allowed Actions                        |
| --------------- | ------------------------ | -------------------------------------- |
| **Active**      | Normal ownership         | Set resolver, transfer, renew, release |
| **Expired**     | Just expired             | Only original owner can renew          |
| **GracePeriod** | 90-day grace period      | Only original owner can renew          |
| **Auction**     | Dutch difficulty auction | Anyone can submit PoW                  |
| **Released**    | Fully released           | Anyone can register                    |

### 5.3 Renewal Mechanism

Renewal requires submitting new PoW, with difficulty based on renewal duration:

```
renew_difficulty = current_difficulty × (1 + 0.1 × additional_years)
```

- Renew 1 year: difficulty ×1.1
- Renew 5 years: difficulty ×1.5
- Renew 10 years: difficulty ×2.0

### 5.4 Dutch Difficulty Auction

After grace period ends, domain enters decreasing difficulty auction:

```mermaid
flowchart LR
    A[Grace period ends] --> B[Difficulty = 10× base difficulty]
    B --> C[Difficulty -2% per hour]
    C --> D[Eventually returns to base difficulty]
    D --> E[Maintains base difficulty]
```

```solidity
function getAuctionDifficulty(string memory name) public view returns (uint256) {
    uint256 baseDiff = getBaseDifficulty(name);
    uint256 elapsed = block.timestamp - auctionStartTime[name];
    uint256 hoursElapsed = elapsed / 1 hours;

    // Starts at 10x, decreases 2% per hour, reaches 1x after ~115 hours
    uint256 multiplier = 1000 - min(hoursElapsed * 20, 900); // 1000 = 10x, 100 = 1x

    return (baseDiff * multiplier) / 100;
}
```

---

## 6. Bounty Market

### 6.1 Overview

Non-mining users can delegate PoW work through the bounty market:

```mermaid
sequenceDiagram
    participant User
    participant BountyVault
    participant Miner
    participant Registry

    User->>BountyVault: placeBounty(nameHash, token, amount, deadline)
    Note over BountyVault: Lock funds

    Miner->>Miner: Compute PoW off-chain

    Miner->>Registry: claimBounty(name, owner, nonce)
    Registry->>Registry: Verify PoW
    Registry->>Registry: Verify nameHash matches

    alt Verification passed
        Registry->>User: Mint NFT
        Registry->>Miner: Pay bounty
        Registry->>Miner: Pay $POWNS reward
    else Verification failed
        Registry->>Registry: Revert
    end
```

### 6.2 Bounty Structure

```solidity
struct Bounty {
    bytes32 nameHash;       // keccak256(name)
    address owner;          // NFT recipient
    address token;          // Payment token (ETH/USDC/POWNS)
    uint256 amount;         // Bounty amount
    uint256 deadline;       // Expiration time
    uint8 minYears;         // Minimum registration years
    uint256 maxDifficulty;  // Maximum acceptable difficulty (user protection)
}
```

### 6.3 Front-running Protection

Since the hash is bound to the `miner` address:

- Nonces calculated by Miner A can only be submitted by Miner A
- Even if seen by MEV bots, they cannot be stolen
- Bounty is atomically paid to the actual PoW completer

### 6.4 Bounty Cancellation & Expiration

```mermaid
flowchart TD
    A[Bounty Created] --> B{Status Check}

    B -->|Before deadline| C[Waiting for miner]
    B -->|After deadline| D[Bounty expired]

    C -->|Miner claims| E[Atomic execution]
    C -->|User cancels| F{Any pending claims?}

    F -->|No| G[Full refund]
    F -->|Yes| H[Wait for claim to complete or fail]

    D --> I[User can withdraw funds]

    E --> J[NFT → Owner]
    E --> K[Bounty → Miner]
```

---

## 7. Resolver System

### 7.1 Architecture

```mermaid
flowchart TB
    subgraph "Registry"
        R1[Domain NFT]
        R2[Owner]
        R3[Expiration]
    end

    subgraph "Resolver"
        S1[Addresses]
        S2[Text Records]
        S3[Contenthash]
    end

    R1 --> S1
    R1 --> S2
    R1 --> S3

    subgraph "Addresses"
        A1[ETH]
        A2[BTC]
        A3[SOL]
        A4[...]
    end

    subgraph "Text Records"
        T1[avatar]
        T2[email]
        T3[url]
        T4[twitter]
    end

    subgraph "Contenthash"
        C1[IPFS]
        C2[Arweave]
        C3[Swarm]
    end

    S1 --> A1 & A2 & A3 & A4
    S2 --> T1 & T2 & T3 & T4
    S3 --> C1 & C2 & C3
```

### 7.2 Record Types

| Type               | Description           | Examples                    |
| ------------------ | --------------------- | --------------------------- |
| **addr(coinType)** | Multi-chain addresses | ETH, BTC, SOL, ATOM...      |
| **text(key)**      | Text records          | avatar, email, url, twitter |
| **contenthash**    | Content hashes        | IPFS CID, Arweave TX        |

### 7.3 Setting Permissions

- Only the Owner can set resolver records
- Setting records **does not require PoW**
- Gas fees apply

---

## 8. $POWNS Token Economics

### 8.1 Token Utility

| Utility               | Description                          |
| --------------------- | ------------------------------------ |
| **Fee Discount**      | 20% discount when paying with $POWNS |
| **Bounty Settlement** | Standard currency for bounty market  |
| **Staking Rewards**   | Stake to share protocol revenue      |
| **DAO Governance**    | Proposal and voting rights           |

### 8.2 Token Distribution

```mermaid
pie title $POWNS Distribution (Total Supply: 1 Billion)
    "PoW Mining Rewards (40%)" : 400
    "DAO Treasury (25%)" : 250
    "Team & Advisors (15%)" : 150
    "Early Contributors (10%)" : 100
    "Liquidity (10%)" : 100
```

| Category           | Percentage | Amount | Vesting                      |
| ------------------ | ---------- | ------ | ---------------------------- |
| PoW Mining         | 40%        | 400M   | 10-year linear release       |
| DAO Treasury       | 25%        | 250M   | Governance unlocks           |
| Team               | 15%        | 150M   | 1-year cliff + 3-year linear |
| Early Contributors | 10%        | 100M   | TGE airdrop                  |
| Liquidity          | 10%        | 100M   | TGE launch                   |

### 8.3 Mining Rewards

```mermaid
flowchart LR
    A[Domain Registration Success] --> B[Calculate Reward]
    B --> C[BaseReward × DifficultyMultiplier]
    C --> D[Pay to Miner]

    E[Every 2 years] --> F[BaseReward halves]
```

**Reward Formula**:

```
reward = baseReward × (difficulty / baseDifficulty)
```

- Short domain (high difficulty) → Higher $POWNS reward
- Long domain (low difficulty) → Lower $POWNS reward

### 8.4 Halving & Burning

| Mechanism   | Description                     |
| ----------- | ------------------------------- |
| **Halving** | baseReward halves every 2 years |
| **Burning** | 50% of fees permanently burned  |

### 8.5 Revenue Distribution

```mermaid
flowchart TD
    A[Protocol Revenue] --> B{Sources}

    B --> C[Deposit Interest]
    B --> D[Trading Fees]
    B --> E[Bounty Fees]

    C --> F[Revenue Pool]
    D --> F
    E --> F

    F --> G[50% → Stakers]
    F --> H[30% → DAO Treasury]
    F --> I[20% → Burn]
```

---

## 9. Governance

### 9.1 DAO Structure

```mermaid
flowchart TB
    subgraph "Governance Participants"
        H1[$POWNS Holders]
        H2[Stakers]
    end

    subgraph "Governance Process"
        P1[Proposal Creation]
        P2[Discussion Period: 3 days]
        P3[Voting Period: 5 days]
        P4[Timelock: 2 days]
        P5[Execution]
    end

    H1 --> P1
    H2 --> P1
    P1 --> P2 --> P3 --> P4 --> P5

    subgraph "Governable Parameters"
        G1[Difficulty Target Interval]
        G2[Deposit Ratios]
        G3[Fee Rates]
        G4[Grace Period Length]
        G5[Contract Upgrades]
    end

    P5 --> G1 & G2 & G3 & G4 & G5
```

### 9.2 Voting Power

```
voting_power = staked_amount × time_multiplier

time_multiplier:
  - Staked < 1 month: 1.0x
  - Staked 1-6 months: 1.25x
  - Staked 6-12 months: 1.5x
  - Staked > 12 months: 2.0x
```

### 9.3 Proposal Thresholds

| Type                 | Proposal Threshold | Approval Threshold             |
| -------------------- | ------------------ | ------------------------------ |
| Parameter Adjustment | 0.1% of supply     | Simple majority + 10% quorum   |
| Contract Upgrade     | 1% of supply       | 2/3 supermajority + 20% quorum |
| Treasury Spending    | 0.5% of supply     | Simple majority + 15% quorum   |

---

## 10. Ecosystem Expansion

### 10.1 Trading Marketplace

Built-in domain trading marketplace supporting:

| Mode            | Description                     |
| --------------- | ------------------------------- |
| **Fixed Price** | List for sale at fixed price    |
| **Offers**      | Accept buyer offers             |
| **Auction**     | Set starting price and duration |

**Trading Fee**: 2.5%

- 50% → Stakers
- 30% → DAO
- 20% → Burn

### 10.2 Domain Leasing

```mermaid
sequenceDiagram
    participant Owner
    participant LeaseContract
    participant Tenant

    Owner->>LeaseContract: createLease(name, price, duration)
    Tenant->>LeaseContract: rent(name)
    LeaseContract->>Owner: Transfer rent payment
    LeaseContract->>Tenant: Grant temporary resolver rights

    Note over Tenant: Can set own resolver records

    alt Lease ends
        LeaseContract->>LeaseContract: Automatically revoke permissions
    else Early termination
        Tenant->>LeaseContract: terminate()
        LeaseContract->>Tenant: Refund remaining rent
    end
```

### 10.3 Subdomains

```mermaid
flowchart TB
    A[alice.pow] --> B[Owner has full control]
    B --> C[Can create subdomains]

    C --> D[blog.alice.pow]
    C --> E[mail.alice.pow]
    C --> F[*.alice.pow]

    B --> G[Subdomain rules]
    G --> H[Free distribution]
    G --> I[Paid sales]
    G --> J[PoW registration]
```

### 10.4 Multi-chain Deployment

```mermaid
flowchart TB
    subgraph "Ethereum Mainnet"
        E1[Core Registry]
        E2[Short Domains]
        E3[$POWNS Token]
    end

    subgraph "Arbitrum"
        A1[L2 Registry]
        A2[Bounty Market]
        A3[Low Gas Operations]
    end

    subgraph "Base"
        B1[L2 Registry]
        B2[Social Features]
    end

    E1 <-->|Cross-chain Sync| A1
    E1 <-->|Cross-chain Sync| B1
    E3 <-->|Bridge| A1
    E3 <-->|Bridge| B1
```

### 10.5 Wallet Integration

Goal: Native `.pow` suffix support in mainstream wallets

| Wallet   | Integration Method   |
| -------- | -------------------- |
| MetaMask | Snaps plugin         |
| Rainbow  | Native integration   |
| Phantom  | Cross-chain resolver |

---

## 11. Security Analysis

### 11.1 Attack Vectors & Defenses

| Attack                      | Risk   | Defense                                       |
| --------------------------- | ------ | --------------------------------------------- |
| **Front-running**           | High   | Hash bound to miner address                   |
| **Difficulty Manipulation** | Medium | DGW smooth adjustment + ±25% limit            |
| **Domain Squatting**        | Medium | Deposit + renewal cost + difficulty scaling   |
| **Sybil Attack**            | Low    | PoW is inherently Sybil-resistant             |
| **51% Hashrate Attack**     | Low    | Independent of mainchain consensus, low value |
| **Replay Attack**           | Low    | Hash includes chainId + contract address      |

### 11.2 Economic Security

| Risk                                | Defense                                          |
| ----------------------------------- | ------------------------------------------------ |
| Difficulty too low causing spam     | MIN_DIFFICULTY floor                             |
| Difficulty too high freezing system | MAX_DIFFICULTY ceiling + auto-decrease           |
| Locked bounty funds                 | Deadline mechanism + cancellation                |
| Lost deposits                       | Clear rules + voluntary release with full refund |

### 11.3 Contract Security

- OpenZeppelin standard libraries
- Multi-sig upgrade control
- Timelock delayed execution
- Planned: 2 independent security audits

---

## 12. Roadmap

```mermaid
gantt
    title PoWNS Development Roadmap
    dateFormat YYYY-MM

    section Phase 1: Testnet
    Core Contract Development     :2026-01, 2026-03
    PoW Verifier                  :2026-02, 2026-03
    Testnet Deployment            :milestone, m1, 2026-03, 0d
    Public Testing                :2026-03, 2026-06
    Bug Bounty Program            :2026-04, 2026-06

    section Phase 2: Mainnet
    Security Audits               :2026-05, 2026-07
    Mainnet Deployment            :milestone, m2, 2026-07, 0d
    $POWNS TGE                    :2026-08, 2026-08
    Staking Launch                :2026-08, 2026-09
    DAO Launch                    :2026-09, 2026-10

    section Phase 3: Ecosystem
    Trading Marketplace           :2026-10, 2026-12
    Leasing Feature               :2027-01, 2027-02
    Subdomains                    :2027-02, 2027-03

    section Phase 4: Multi-chain
    Arbitrum Deployment           :2027-03, 2027-05
    Base Deployment               :2027-04, 2027-06
    Cross-chain Resolver          :2027-05, 2027-07

    section Phase 5: Integration
    Wallet Integration            :2027-06, 2027-08
    Identity System               :2027-08, 2027-10
```

| Phase       | Timeline   | Milestones                                     |
| ----------- | ---------- | ---------------------------------------------- |
| **Phase 1** | Q1-Q2 2026 | Testnet deployment, public testing, bug bounty |
| **Phase 2** | Q3-Q4 2026 | Mainnet deployment, $POWNS TGE, DAO launch     |
| **Phase 3** | Q1 2027    | Trading marketplace, leasing, subdomains       |
| **Phase 4** | Q2-Q3 2027 | Multi-chain deployment, cross-chain resolver   |
| **Phase 5** | Q3-Q4 2027 | Wallet integration, identity system            |

---

## 13. Conclusion

PoW Name Service (PoWNS) introduces a new paradigm for name distribution:

1. **Fairness**: Computation replaces capital as the source of scarcity
2. **Decentralization**: Pure on-chain verification, no centralized control points
3. **Sustainability**: Renewal mechanism prevents permanent squatting
4. **Inclusivity**: Bounty market enables non-miners to participate
5. **Economic Closure**: $POWNS token connects all participants

We believe PoWNS will bring a fairer, more censorship-resistant option to Web3 identity infrastructure.

---

## Appendix

### A. Glossary

| Term             | Definition                                         |
| ---------------- | -------------------------------------------------- |
| **PoW**          | Proof of Work                                      |
| **DGW**          | Dark Gravity Wave, difficulty adjustment algorithm |
| **Bounty**       | Mining reward posted by users for miners           |
| **Resolver**     | Maps domain names to addresses                     |
| **Grace Period** | Time window after expiration allowing renewal      |

### B. Contract Addresses

_To be updated after testnet deployment_

### C. References

- [EIP-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721)
- [ENS Documentation](https://docs.ens.domains/)
- [Dark Gravity Wave](https://github.com/dashpay/dash/blob/master/src/pow.cpp)
- [Bitcoin Difficulty Adjustment](https://en.bitcoin.it/wiki/Difficulty)

---

_© 2025 PoWNS Labs. All rights reserved._
