# AQUARI Subscription & Affiliate — Proposal A: Smart Contract Architecture

> **Project**: [AQUARI Subscription & Affiliate System](https://github.com/Mohsinsiddi/project-md-plans/tree/main/aquari-subscription-affiliate)
> **Proposal**: AquariProcessor contract on Base — vault + swap + affiliate + bonus in 1 atomic tx.

---

## 1. System Overview

```
+------------------+       +------------------+       +------------------------+
|                  |       |                  |       |                        |
|   Aquari Web App |       |   Backend API    |       |  AquariProcessor       |
|   (Privy Auth)   |------>|   (Node.js)      |------>|  Contract (Base)       |
|                  |       |                  |       |                        |
+------------------+       +------------------+       |  - Vault (USDC)        |
                                  |                   |  - Treasury (AQUARI)   |
                                  |                   |  - Swap (Uniswap V2)   |
                                  v                   |  - Commitments         |
                           +------------------+       +----------+-------------+
                           |                  |                  |
                           |   PostgreSQL     |                  v
                           |   (State + Logs) |       +-------------------+
                           |                  |       | Uniswap V2 Router |
                           +------------------+       | (Base)            |
                                                      +-------------------+
                                                               |
                                                               v
                                                      +-------------------+
                                                      | AQUARI Token      |
                                                      | (fee-on-transfer) |
                                                      +-------------------+
```

---

## 2. User Onboarding Flow

### 2.1 New User (Direct)

```
Step 1          Step 2              Step 3
+----------+    +---------------+   +------------------+
| User hits |    | Sign in with  |   | User sees their  |
| aquari.io |--->| Privy wallet  |-->| dashboard with   |
|           |    | (1 click)     |   | wallet on Base   |
+----------+    +---------------+   +------------------+
                                           |
                                    No KYC needed yet
                                    No forms, no email
                                    Just wallet connect
```

### 2.2 Referred User (via Affiliate Link)

```
Step 1                  Step 2              Step 3              Step 4
+------------------+    +---------------+   +---------------+   +-----------------+
| User clicks      |    | Sign in with  |   | Ref code auto |   | Redirected to   |
| aquari.io/       |--->| Privy wallet  |-->| captured from |-->| subscription    |
| ?ref=ABC123      |    | (1 click)     |   | URL & stored  |   | page instantly  |
+------------------+    +---------------+   +---------------+   +-----------------+
                                                                       |
                                                                No detours.
                                                                Conversion-focused.
                                                                Pick plan → pay → done.
```

---

## 3. Subscription Setup Flow

### 3.1 Choose Plan

```
+-------------------------------------------------------+
|                                                       |
|   Select Your AQUARI Plan                             |
|                                                       |
|   Monthly Amount:  [ $5 ] [ $25 ] [ $50 ] [ $100 ]   |
|                    [ Custom: $____ ]                  |
|                                                       |
|   Commitment:      ( ) 6 months    ( ) 12 months     |
|                                                       |
|   You'll receive AQUARI worth $X/month                |
|   Complete your commitment → earn 5% bonus            |
|                                                       |
+-------------------------------------------------------+
                          |
                          v
              Choose Payment Method
              /                    \
           Crypto                  Fiat
          (faster)               (slower)
```

### 3.2 Payment Method: Crypto (No KYC — Fast Path)

```
Step 1              Step 2              Step 3              Step 4
+---------------+   +---------------+   +---------------+   +------------------+
| User picks    |   | App shows     |   | User confirms |   | USDC deposited   |
| "Pay with     |-->| total deposit |-->| tx in Privy   |-->| in vault.        |
| Crypto"       |   | e.g. $600     |   | wallet        |   | Subscription     |
|               |   | (50 x 12 mo) |   | (approve +    |   | ACTIVE.          |
+---------------+   +---------------+   | deposit = 2tx)|   +------------------+
                                        +---------------+
                                                             Zero KYC.
                                                             ~30 seconds total.
                                                             2 wallet confirmations.
```

**On-chain transactions (user pays gas):**

| # | Transaction | What Happens | Gas |
|---|-------------|-------------|-----|
| 1 | `USDC.approve(contract, amount)` | Allow contract to pull USDC | ~46k |
| 2 | `contract.depositToVault(amount)` | USDC moves to vault, commitment registered | ~120k |
| **Total** | **2 txs** | **One-time setup** | **~166k** |

### 3.3 Payment Method: Fiat (KYC Required — Slower Path)

```
Step 1              Step 2              Step 3              Step 4
+---------------+   +---------------+   +---------------+   +------------------+
| User picks    |   | Redirected to |   | User enters   |   | Subscription     |
| "Pay with     |-->| Inflow KYC    |-->| card / bank   |-->| created on       |
| Card/Bank"    |   | verification  |   | details in    |   | Inflow.          |
|               |   | (1-3 mins)    |   | Inflow        |   | First charge     |
+---------------+   +---------------+   | checkout      |   | happens now.     |
                                        +---------------+   | Subscription     |
                                                            | ACTIVE.          |
                                                            +------------------+
                                                             KYC required.
                                                             ~3-5 minutes total.
                                                             0 on-chain txs from user.
```

**On-chain transactions (user pays gas):**

| # | Transaction | What Happens |
|---|-------------|-------------|
| — | None | Inflow handles everything off-chain |
| **Total** | **0 txs from user** | Inflow charges card, settles USDC to platform wallet |

---

## 4. Monthly Execution Flow (Automated — Zero User Action)

This happens every month for every active subscriber. The user does nothing.

### 4.1 Fiat User — Monthly Flow

```
                        AUTOMATED — NO USER ACTION

  +-------------+     +-----------+     +-------------+     +-------------------+
  | Inflow      |     | Inflow    |     | Webhook     |     | Backend calls     |
  | charges     |---->| settles   |---->| hits our    |---->| contract:         |
  | user's card |     | USDC to   |     | backend     |     | processPayment()  |
  |             |     | our wallet|     | with payment|     |                   |
  +-------------+     | on Base   |     | details     |     +--------+----------+
                      +-----------+     +-------------+              |
                                                                     v
                                                          +---------------------+
                                                          | CONTRACT (1 tx):    |
                                                          |                     |
                                                          | 1. Receives USDC    |
                                                          |    from backend     |
                                                          |                     |
                                                          | 2. Swaps USDC →     |
                                                          |    AQUARI via       |
                                                          |    Uniswap V2      |
                                                          |    (to = user       |
                                                          |     wallet)         |
                                                          |                     |
                                                          | 3. If referrer:     |
                                                          |    sends affiliate  |
                                                          |    reward from      |
                                                          |    treasury         |
                                                          |                     |
                                                          | 4. Updates          |
                                                          |    commitment       |
                                                          |    state            |
                                                          +---------------------+
                                                                     |
                                                                     v
                                                          +---------------------+
                                                          | User sees AQUARI    |
                                                          | in Privy wallet     |
                                                          | immediately         |
                                                          +---------------------+
```

### 4.2 Crypto User — Monthly Flow

```
                        AUTOMATED — NO USER ACTION

  +-------------------+     +-------------------+
  | Backend triggers  |     | CONTRACT (1 tx):  |
  | monthly cron /    |---->|                   |
  | scheduler         |     | 1. Debits USDC    |
  |                   |     |    from user's    |
  +-------------------+     |    vault balance  |
                            |                   |
                            | 2. Swaps USDC →   |
                            |    AQUARI via     |
                            |    Uniswap V2    |
                            |    (to = user     |
                            |     wallet)       |
                            |                   |
                            | 3. If referrer:   |
                            |    sends reward   |
                            |    from treasury  |
                            |                   |
                            | 4. Updates        |
                            |    commitment     |
                            |    state          |
                            +-------------------+
                                     |
                                     v
                            +-------------------+
                            | User sees AQUARI  |
                            | in Privy wallet   |
                            | immediately       |
                            +-------------------+
```

### 4.3 Monthly Transaction Count (Admin/Backend Pays Gas)

| Scenario | Txs | Details |
|----------|-----|---------|
| Monthly swap — no referrer | **1** | swap + send to user |
| Monthly swap — with referrer | **1** | swap + send to user + affiliate reward (same tx) |
| Final month — no referrer | **1** | swap + send to user + completion bonus (same tx) |
| Final month — with referrer | **1** | swap + send to user + affiliate + bonus (same tx) |
| **Always** | **1 tx per user per month** | **Everything atomic** |

---

## 5. Final Month — Completion Flow

```
                     FINAL MONTH (month 6 of 6, or 12 of 12)

  Same as normal monthly flow, but contract detects:
  monthsCompleted + 1 == totalMonths

  +------------------------------------------------------------------+
  | CONTRACT processPayment() — SINGLE ATOMIC TX:                    |
  |                                                                  |
  | 1. Swap USDC → AQUARI (to = user wallet)         ← normal       |
  |                                                                  |
  | 2. Send affiliate reward (if referrer)            ← normal       |
  |                                                                  |
  | 3. Calculate bonus:                               ← NEW          |
  |    bonus = totalAquariDeposited * 5%                             |
  |                                                                  |
  | 4. Send bonus AQUARI from treasury → user wallet  ← NEW          |
  |                                                                  |
  | 5. Mark commitment as COMPLETED                   ← NEW          |
  +------------------------------------------------------------------+

  User receives in their Privy wallet:
  - This month's AQUARI (from swap)
  - 5% bonus on ALL AQUARI accumulated over the commitment
  - All in one tx
```

### Completion Bonus Example

```
12-month commitment at $100/month:

Month 1:  swap → ~9,500 AQUARI to user (after 5% token entry tax)
Month 2:  swap → ~9,800 AQUARI to user
...
Month 12: swap → ~10,200 AQUARI to user
          + bonus: 5% × 120,000 total = 6,000 AQUARI from treasury

Total AQUARI received over 12 months: ~120,000 + 6,000 bonus = ~126,000
```

---

## 6. Cancellation & Edge Cases

### 6.1 User Cancels Mid-Commitment

```
+------------------+     +------------------+     +--------------------+
| User requests    |     | Backend calls    |     | Contract:          |
| cancellation     |---->| contract:        |---->| - Marks inactive   |
| (via app)        |     | cancelCommitment |     | - No bonus earned  |
+------------------+     +------------------+     | - Remaining vault  |
                                                  |   USDC refundable  |
                                                  +--------------------+

  User keeps: All AQUARI received so far (already in their wallet)
  User loses: The 5% completion bonus
  Refund:     Remaining USDC in vault (crypto users only)
  Txs:        1 (cancel) + 1 (refund if crypto user) = 1-2 txs
```

### 6.2 Fiat Payment Fails

```
+------------------+     +------------------+     +--------------------+
| Inflow webhook:  |     | Backend:         |     | After 30 days:     |
| payment.failed   |---->| Mark sub PAUSED  |---->| Commitment resets  |
|                  |     | Notify user      |     | User keeps AQUARI  |
+------------------+     | Inflow retries   |     | Bonus counter → 0  |
                         +------------------+     +--------------------+

  No on-chain tx needed for pause.
  Contract state unchanged until next successful payment.
```

### 6.3 Swap Fails (Tx Reverts)

```
+------------------+     +------------------+
| processPayment   |     | ENTIRE TX        |
| reverts          |---->| REVERTS          |
| (slippage, pool  |     |                  |
|  empty, etc.)    |     | - USDC stays in  |
+------------------+     |   vault/backend  |
                         | - No AQUARI sent |
                         | - No affiliate   |
                         | - No state change|
                         | - Retry safe     |
                         +------------------+

  This is the MAIN ADVANTAGE of the contract approach.
  Atomic = safe. Nothing is in a broken partial state.
```

---

## 7. Admin Perspective

### 7.1 What Admin Does

| Task | Frequency | Action |
|------|-----------|--------|
| Trigger monthly payments | Monthly | Backend auto-calls `processPayment` per user |
| Fund treasury | As needed | Deposit AQUARI for affiliate rewards + bonuses |
| Handle failed fiat payments | On webhook | Pause subscription, monitor retry |
| Process cancellations | On request | Call `cancelCommitment`, process vault refund |
| Monitor treasury balance | Weekly | Ensure enough AQUARI for upcoming rewards |

### 7.2 What Admin Sees in Dashboard

```
+------------------------------------------------------------------+
| SUBSCRIPTION OVERVIEW                                            |
|                                                                  |
| Active Subscribers: 1,247    |  Total AQUARI Purchased: 14.2M   |
| 6-month plans: 834           |  This Month USD: $125,400        |
| 12-month plans: 413          |  All-time USD: $892,000          |
| Fiat: 1,021 | Crypto: 226    |  Completions: 89 | Cancelled: 34 |
|                                                                  |
| [Monthly Cashflow Chart]     |  [Subscriber Trend Line]         |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| SUBSCRIBERS TABLE                    [Search...] [Filter: All v] |
|                                                                  |
| User    | $/mo | Plan  | Method | Month | Status  | Referred By |
|---------|------|-------|--------|-------|---------|-------------|
| 0x1a..  | $100 | 12mo  | Crypto | 7/12  | Active  | ABC123      |
| 0x2b..  | $50  | 6mo   | Fiat   | 3/6   | Active  | —           |
| 0x3c..  | $25  | 12mo  | Fiat   | 12/12 | Done    | XYZ789      |
| 0x4d..  | $200 | 6mo   | Crypto | 2/6   | Paused  | —           |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| AFFILIATE LEADERBOARD                                            |
|                                                                  |
| Treasury Balance: 2,450,000 AQUARI                               |
| Total Rewards Paid: 890,000 AQUARI                               |
|                                                                  |
| Affiliate | Code   | Refs | Converted | Rate  | AQUARI Earned   |
|-----------|--------|------|-----------|-------|-----------------|
| 0xaa..    | ABC123 | 45   | 38        | 84.4% | 125,000         |
| 0xbb..    | DEF456 | 22   | 15        | 68.2% | 67,000          |
| 0xcc..    | GHI789 | 18   | 12        | 66.7% | 45,000          |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| ALERTS                                                           |
|                                                                  |
| [!] Treasury balance below 500k AQUARI — top up soon             |
| [!] 3 failed swaps in last 24h — check liquidity                |
| [!] User 0x4d.. fiat payment failed — Inflow retrying           |
+------------------------------------------------------------------+
```

---

## 8. Complete Transaction Summary

### 8.1 User Transactions (User Pays Gas)

| Action | Crypto User | Fiat User |
|--------|------------|-----------|
| Onboarding (Privy sign-in) | 0 txs | 0 txs |
| KYC | Not needed | Off-chain (Inflow) |
| Subscription setup | 2 txs (approve + deposit) | 0 txs |
| Monthly payments | 0 txs (admin handles) | 0 txs (Inflow + admin) |
| Cancellation refund | 0-1 tx (admin sends) | N/A |
| **Total user txs over 12 months** | **2** | **0** |

### 8.2 Admin/Backend Transactions (Platform Pays Gas)

| Action | Txs | Frequency |
|--------|-----|-----------|
| processPayment (swap + affiliate + bonus) | 1 | Monthly per user |
| Fund treasury | 1 | As needed |
| Cancel commitment | 1 | On request |
| Refund vault (crypto cancellation) | 1 | On request |
| **Monthly cost per user** | **1 tx** | **Always 1, always atomic** |

### 8.3 Total Txs Over a 12-Month Commitment

| Scenario | Crypto User | Fiat User |
|----------|------------|-----------|
| Setup | 2 (user pays) | 0 |
| 12 monthly payments | 12 (admin pays) | 12 (admin pays) |
| Affiliate rewards (if referrer) | 0 (included in monthly) | 0 (included in monthly) |
| Completion bonus | 0 (included in month 12) | 0 (included in month 12) |
| **Total on-chain txs** | **14** | **12** |

---

## 9. Smart Contract Interface

```solidity
// AquariProcessor.sol — deployed on Base

interface IAquariProcessor {

    // === USER / ADMIN SETUP ===

    // Crypto user deposits full commitment USDC into vault
    function depositToVault(
        bytes32 userId,
        uint256 usdcAmount,
        uint256 totalMonths,
        uint256 monthlyAmount,
        address userWallet,
        address referrerWallet      // address(0) if none
    ) external;

    // Register fiat user commitment (no deposit, USDC comes per payment)
    function registerCommitment(
        bytes32 userId,
        uint256 totalMonths,
        uint256 monthlyAmount,
        address userWallet,
        address referrerWallet
    ) external onlyAdmin;


    // === MONTHLY EXECUTION (admin-only) ===

    // Process monthly payment — swaps USDC→AQUARI, sends to user, handles affiliate + bonus
    // For fiat users: admin sends USDC with this call
    // For crypto users: USDC debited from vault
    function processMonthlyPayment(
        bytes32 userId,
        uint256 usdcAmount          // 0 for crypto users (uses vault)
    ) external onlyAdmin;


    // === ADMIN MANAGEMENT ===

    function cancelCommitment(bytes32 userId) external onlyAdmin;
    function refundVault(bytes32 userId) external onlyAdmin;
    function fundTreasury(uint256 aquariAmount) external onlyAdmin;
    function withdrawTreasury(uint256 aquariAmount) external onlyAdmin;


    // === VIEW FUNCTIONS ===

    function getCommitment(bytes32 userId) external view returns (
        uint256 totalMonths,
        uint256 monthsCompleted,
        uint256 monthlyAmount,
        uint256 totalAquariDeposited,
        address userWallet,
        address referrerWallet,
        bool active
    );
    function getVaultBalance(bytes32 userId) external view returns (uint256);
    function getTreasuryBalance() external view returns (uint256);
}
```

---

## 10. Pros & Cons

### Pros
| # | Advantage |
|---|-----------|
| 1 | **Always 1 tx per month** — swap + affiliate + bonus atomic |
| 2 | **Safe on failure** — tx reverts, USDC stays safe, no broken state |
| 3 | **Vault for crypto users** — segregated funds per user, not co-mingled |
| 4 | **On-chain audit trail** — every payment verifiable on BaseScan |
| 5 | **Simple admin ops** — one call per user per month, contract handles the rest |
| 6 | **Zero user txs after setup** — user only signs 2 txs (crypto) or 0 (fiat), then hands-off |
| 7 | **Referrer reward in same tx** — no risk of paying user but forgetting affiliate |

### Cons
| # | Disadvantage |
|---|--------------|
| 1 | **Contract deployment + audit** — needs security review before mainnet |
| 2 | **Higher gas per tx** — contract wrapper adds ~50-100k gas over direct router call |
| 3 | **Less flexible** — business logic changes need contract upgrade (use proxy pattern) |
| 4 | **Smart contract risk** — bugs could lock or lose funds |
| 5 | **Treasury management** — must keep AQUARI funded, adds operational overhead |
| 6 | **Longer time to ship** — contract dev + testing + audit vs just backend code |
