# AQUARI Subscription & Affiliate — Proposal D: Privy Server Wallet Delegation

> **Project**: [AQUARI Subscription & Affiliate System](https://github.com/Mohsinsiddi/project-md-plans/tree/main/aquari-subscription-affiliate)
> **Proposal**: Backend uses Privy's server-side SDK to execute swaps FROM user's own Privy wallet — with strict policy constraints. No private keys stored on our side. No custom contract.

---

## 1. System Overview

```
+------------------+       +------------------+       +------------------------+
|                  |       |                  |       |                        |
|   Aquari Web App |       |   Backend API    |       |  Privy Server SDK      |
|   (Privy Auth)   |------>|   (Node.js)      |------>|  (Delegated Signing)   |
|                  |       |                  |       |                        |
+------------------+       |                  |       |  Policy constraints:   |
                           |                  |       |  - Only Uniswap V2     |
                           |                  |       |  - Only approve+swap   |
                           |                  |       |  - Only USDC→AQUARI    |
                           |                  |       |  - Max amount per tx   |
                           |                  |       |                        |
                           |  +------------+  |       +----------+-------------+
                           |  | Merchant   |  |                  |
                           |  | Wallet     |  |                  v
                           |  | (Inflow    |  |       +-------------------+
                           |  |  USDC)     |  |       | User's Privy      |
                           |  +------------+  |       | Embedded Wallet   |
                           |                  |       | (signs via Privy  |
                           |  +------------+  |       |  server infra)    |
                           |  | Treasury   |  |       +-------------------+
                           |  | Wallet     |  |                  |
                           |  | (AQUARI)   |  |                  v
                           |  +------------+  |       +-------------------+
                           |                  |       | Uniswap V2 Router |
                           +--------+---------+       | (Base)            |
                                    |                 +-------------------+
                                    v                          |
                           +------------------+                v
                           |                  |       +-------------------+
                           |   PostgreSQL     |       | AQUARI Token      |
                           |   (All state,    |       | (fee-on-transfer) |
                           |    NO keys)      |       +-------------------+
                           +------------------+
```

**Key difference from Proposal C**: We NEVER touch or store private keys. Privy's server-side infrastructure holds the keys and signs transactions on our behalf — subject to strict policy constraints we define. The security burden for key custody shifts entirely to Privy. Our DB stores zero cryptographic material.

**Server dependency tradeoff**: Our backend must communicate with **two external services** — Inflow (for fiat payments) AND Privy (for transaction signing). In Proposal C, the backend only talks to Inflow — all signing is self-contained. This is the core tradeoff: D is more secure but adds a second external dependency. If either Inflow OR Privy is down, the monthly swap pipeline stalls.

```
Proposal C — Server Dependencies:          Proposal D — Server Dependencies:

  Our Backend                                Our Backend
      │                                          │
      ├──→ Inflow API (payments)                 ├──→ Inflow API (payments)
      │    (external dependency)                 │    (external dependency)
      │                                          │
      └──→ Base RPC (submit txs)                 ├──→ Privy Server SDK (signing)
           (self-signed, self-submitted)         │    (external dependency)
                                                 │
           1 external dependency                 └──→ Base RPC (submit txs)

                                                      2 external dependencies
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
                                    No key extraction
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
```

### 2.3 Terms & Conditions (Required Before Subscription)

```
+------------------------------------------------------------------+
|                                                                  |
|   Before subscribing, user MUST accept:                          |
|                                                                  |
|   [x] I authorize AQUARI platform to execute USDC-to-AQUARI     |
|       swap transactions from my wallet via Privy                 |
|                                                                  |
|   [x] I understand swaps are limited to Uniswap V2 on Base      |
|       and restricted to USDC→AQUARI only                         |
|                                                                  |
|   [x] I acknowledge that the platform cannot access my           |
|       private keys or execute any transaction outside the        |
|       defined swap policy                                        |
|                                                                  |
|   [ Continue to Payment ]                                        |
|                                                                  |
+------------------------------------------------------------------+

  Consent stored in DB: user.terms_accepted = true, terms_accepted_at
  Enables server-side delegation for this user's wallet
```

**Note vs Proposal C**: The terms here are stronger — we can truthfully say "we cannot access your private keys." In Proposal C, we hold the keys. Here, Privy holds them.

---

## 3. Privy Server Wallet Integration

### 3.1 What is Privy Server-Side Delegation?

```
Normal flow (client-side):
  User clicks "Swap" → Privy client SDK → user's wallet signs → tx submitted

Server delegation flow:
  Backend calls Privy Server SDK → Privy signs on behalf of user → tx submitted
  User does nothing. Backend initiates. Privy signs. User's wallet is the sender.

The user's wallet address is the msg.sender on-chain.
But the signing happens on Privy's server infrastructure.
Our backend never sees the private key.
```

### 3.2 Policy Constraints (Strict Allowlist)

```
+------------------------------------------------------------------+
|                                                                  |
|   DELEGATION POLICY (configured in Privy dashboard / SDK):       |
|                                                                  |
|   Allowed contracts:                                             |
|   ┌────────────────────────────────────────────────────────┐     |
|   │ 1. USDC on Base         (0x833589fCD6eDb6E08f4c7C32D4 │     |
|   │                          f71b54bdA02913)               │     |
|   │    → only approve() function                           │     |
|   │                                                        │     |
|   │ 2. Uniswap V2 Router    (router address on Base)      │     |
|   │    → only swapExactTokensForTokens                     │     |
|   │      SupportingFeeOnTransferTokens()                   │     |
|   │                                                        │     |
|   │ 3. No other contracts                                  │     |
|   │ 4. No native ETH transfers                             │     |
|   │ 5. No arbitrary calldata                                │     |
|   └────────────────────────────────────────────────────────┘     |
|                                                                  |
|   Limits:                                                        |
|   ┌────────────────────────────────────────────────────────┐     |
|   │ - Max USDC per tx: user's monthly subscription amount  │     |
|   │ - Max txs per day per user: 5                          │     |
|   │ - Allowed chains: Base only (chainId: 8453)            │     |
|   └────────────────────────────────────────────────────────┘     |
|                                                                  |
+------------------------------------------------------------------+

  If our backend tries to call ANY other contract or function:
  → Privy REJECTS the signing request.
  → Private key is never used.
  → Attack surface is minimal.
```

### 3.3 Privy SDK Integration (Pseudocode)

```javascript
// Backend: execute monthly swap for a user

const { PrivyClient } = require('@privy-io/server-auth');
const privy = new PrivyClient(PRIVY_APP_ID, PRIVY_APP_SECRET);

async function executeMonthlySwap(userId, usdcAmount) {
  // 1. Get user's Privy wallet address
  const user = await privy.getUser(userId);
  const walletAddress = user.wallet.address;

  // 2. Build approve calldata
  const approveData = usdcContract.interface.encodeFunctionData(
    'approve',
    [UNISWAP_V2_ROUTER, usdcAmount]
  );

  // 3. Execute approve FROM user's wallet (Privy signs)
  const approveTx = await privy.walletApi.ethereum.sendTransaction({
    walletId: user.wallet.id,
    chainId: 8453, // Base
    transaction: {
      to: USDC_ADDRESS,
      data: approveData,
      value: 0
    }
  });

  // 4. Build swap calldata
  const swapData = router.interface.encodeFunctionData(
    'swapExactTokensForTokensSupportingFeeOnTransferTokens',
    [usdcAmount, amountOutMin, [USDC, AQUARI], walletAddress, deadline]
  );

  // 5. Execute swap FROM user's wallet (Privy signs)
  const swapTx = await privy.walletApi.ethereum.sendTransaction({
    walletId: user.wallet.id,
    chainId: 8453,
    transaction: {
      to: UNISWAP_V2_ROUTER,
      data: swapData,
      value: 0
    }
  });

  return { approveTx, swapTx };
}
```

### 3.4 What We Need From Privy

| Requirement | Status | Notes |
|-------------|--------|-------|
| Server-side `sendTransaction` for embedded wallets | **Need to confirm** | Privy has this for server wallets — unclear if it works for embedded wallets |
| Policy/allowlist constraints | **Need to confirm** | May require coordination with Privy team |
| Per-user delegation consent | **Need to confirm** | User must opt-in; need to know the mechanism |
| Gas sponsorship integration | **Need to confirm** | Who pays gas for server-initiated txs? |
| Rate limits on server signing | **Need to confirm** | Impacts batch execution for many users |
| Base chain support | **Confirmed** | Privy supports Base |

**This is the key dependency**: If Privy does NOT support server-side transaction execution for embedded wallets with policy constraints, Proposal D is not viable and Proposal C becomes the fallback.

---

## 4. Subscription Setup Flow

### 4.1 Choose Plan

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

### 4.2 Payment Method: Crypto (No KYC — Fast Path)

```
Step 1              Step 2              Step 3
+---------------+   +---------------+   +------------------+
| User picks    |   | User already  |   | Backend confirms |
| "Pay with     |-->| has USDC in   |-->| USDC balance.    |
| Crypto"       |   | their Privy   |   | Subscription     |
|               |   | wallet        |   | ACTIVE.          |
+---------------+   +---------------+   +------------------+

  Same as Proposal C: USDC stays in user's wallet.
  Zero txs from user. Zero KYC.
```

### 4.3 Payment Method: Fiat (KYC Required — Slower Path)

```
Step 1              Step 2              Step 3              Step 4
+---------------+   +---------------+   +---------------+   +------------------+
| User picks    |   | Redirected to |   | User enters   |   | Subscription     |
| "Pay with     |-->| Inflow KYC    |-->| card / bank   |-->| created on       |
| Card/Bank"    |   | verification  |   | details in    |   | Inflow.          |
|               |   | (1-3 mins)    |   | Inflow        |   | First charge     |
+---------------+   +---------------+   | checkout      |   | happens now.     |
                                        +---------------+   | ACTIVE.          |
                                                            +------------------+

  Same as all proposals — Inflow settles USDC to merchant wallet.
  Backend forwards to user's Privy wallet before swap.
```

---

## 5. Monthly Execution Flow (Automated — Zero User Action)

### 5.1 Fiat User — Monthly Flow

```
                        AUTOMATED — NO USER ACTION

  +-------------+     +-----------+     +-------------+
  | Inflow      |     | Inflow    |     | Webhook     |
  | charges     |---->| settles   |---->| hits our    |
  | user's card |     | USDC to   |     | backend     |
  |             |     | MERCHANT  |     |             |
  +-------------+     | wallet    |     +------+------+
                      +-----------+            |
                                               v
                                    +---------------------+
                                    | BACKEND EXECUTES:   |
                                    +---------------------+
                                               |
                 +-----------------------------+-----------------------------+
                 |                             |                             |
                 v                             v                             v
      +--------------------+      +------------------------+    +------------------------+
      | TX 1: Forward USDC |      | TX 2-3: Swap           |    | TX 4: Affiliate Reward |
      |                    |      | (via Privy Server SDK) |    | (only if referrer)     |
      | Merchant wallet    |      |                        |    |                        |
      | sends USDC to      |      | Backend calls Privy:   |    | Treasury wallet sends  |
      | user's Privy       |      | privy.walletApi        |    | AQUARI to referrer     |
      | wallet             |      |   .sendTransaction()   |    |                        |
      |                    |      |                        |    +------------------------+
      | (admin pays gas)   |      | TX 2: approve (Privy   |
      +--------------------+      |        signs for user) |
                                  | TX 3: swap (Privy      |
                                  |        signs for user) |
                                  |                        |
                                  | Privy checks policy:   |
                                  | ✓ USDC approve         |
                                  | ✓ Uniswap V2 swap      |
                                  | ✓ Amount within limit  |
                                  | → SIGNS & SUBMITS      |
                                  +------------------------+
                                              |
                                              v
                                  +------------------------+
                                  | User sees AQUARI in    |
                                  | their Privy wallet     |
                                  | ON-CHAIN: user's own   |
                                  | wallet bought AQUARI   |
                                  +------------------------+
```

### 5.2 Crypto User — Monthly Flow

```
                        AUTOMATED — NO USER ACTION

  +-------------------+
  | Backend triggers  |
  | monthly cron /    |
  | scheduler         |
  +--------+----------+
           |
           v
  Same as fiat flow above, but SKIP TX 1 (forward USDC).
  USDC is already in user's Privy wallet.
  Backend goes straight to TX 2-3 via Privy Server SDK.
```

### 5.3 Monthly Transaction Count (Platform Pays Gas)

| Scenario | Txs | Details |
|----------|-----|---------|
| Monthly swap — crypto, no referrer | **2** | approve (1) + swap (1) via Privy SDK |
| Monthly swap — fiat, no referrer | **3** | forward USDC (1) + approve (1) + swap (1) |
| Monthly swap — with referrer | **+1** | + affiliate transfer from treasury |
| Final month — no referrer | **+1** | + bonus transfer from treasury |
| Final month — with referrer | **+2** | + affiliate (1) + bonus (1) from treasury |

**Same tx count as Proposal C.** The difference is HOW the approve + swap are signed, not how many txs there are.

### 5.4 Gas Payment

```
Privy server-initiated txs need gas. Options:

Option 1: Privy gas sponsorship (if available)
  - Privy sponsors gas for server-initiated txs
  - Cleanest UX — user wallet never needs ETH
  - Depends on Privy plan tier

Option 2: Pre-fund user wallet with ETH
  - Backend sends small ETH to user's wallet
  - Privy-initiated txs use that ETH for gas
  - Adds 1 more tx per funding cycle

Option 3: Privy paymaster integration
  - Privy may integrate with paymasters on Base
  - Backend specifies paymaster in sendTransaction call
  - Need to confirm availability
```

---

## 6. Final Month — Completion Flow

```
                     FINAL MONTH (month 6 of 6, or 12 of 12)

  +------------------------------------------------------------------+
  | BACKEND EXECUTES 3-5 SEPARATE TXS:                               |
  |                                                                  |
  | TX 1: Forward USDC to user wallet (fiat only)     ← if fiat     |
  |                                                                  |
  | TX 2: USDC.approve() via Privy Server SDK          ← Privy signs|
  |       (from user's wallet)                                       |
  |                                                                  |
  | TX 3: Swap USDC → AQUARI via Privy Server SDK      ← Privy signs|
  |       (from user's wallet, to = user's wallet)                   |
  |                                                                  |
  | TX 4: Send affiliate reward (if referrer)           ← treasury  |
  |       AQUARI.transfer(referrerWallet, reward)                    |
  |                                                                  |
  | TX 5: Send completion bonus                         ← treasury  |
  |       AQUARI.transfer(userWallet, bonus)                         |
  |       bonus = totalAquariDeposited * 5%                          |
  |                                                                  |
  +------------------------------------------------------------------+

  Same partial failure model as Proposals B and C.
  Backend tracks per-tx status and retries failed transfers.
```

---

## 7. Cancellation & Edge Cases

### 7.1 User Cancels Mid-Commitment

```
+------------------+     +------------------+     +--------------------+
| User requests    |     | Backend:         |     | USDC is already    |
| cancellation     |---->| Mark sub as      |---->| in user's wallet   |
| (via app)        |     | CANCELLED in DB  |     | — NO REFUND TX     |
+------------------+     +------------------+     | NEEDED             |
                                                  +--------------------+

  Same as Proposal C: USDC never left user's wallet.
  Cancellation = DB update only. 0 txs.
```

### 7.2 Fiat Payment Fails

```
Same as all proposals:
Webhook → pause subscription → Inflow retries → 30 days → reset commitment
```

### 7.3 Swap Fails (Tx Reverts)

```
+------------------+     +------------------+
| Privy-signed     |     | TX REVERTS       |
| swap tx reverts  |---->|                  |
| (slippage, etc.) |     | - USDC stays in  |
+------------------+     |   USER'S wallet  |
                         | - Retry safe     |
                         +------------------+

  Same safety as Proposal C.
  Partial failure risk for affiliate/bonus txs also same as B/C.
```

### 7.4 Privy Service Outage — Key Risk for Proposal D

```
  THIS IS THE UNIQUE RISK OF PROPOSAL D

  If Privy's server SDK is down or unreachable:

  +------------------+     +------------------+
  | Backend calls    |     | Privy server     |
  | privy.walletApi  |---->| unavailable      |
  | .sendTransaction |     | or returns error |
  +------------------+     +------------------+
                                    |
                                    v
                           +------------------+
                           | Cannot sign txs  |
                           | Cannot execute   |
                           | monthly swaps    |
                           | USDC stuck until |
                           | Privy recovers   |
                           +------------------+

  Mitigations:
  ┌─────────────────────────────────────────────────────────┐
  │ 1. Retry queue with exponential backoff                 │
  │ 2. Privy SLA guarantees (uptime commitments)             │
  │ 3. Alert admin if Privy is down > 1 hour                │
  │ 4. Grace period: swaps can be delayed by hours/days     │
  │    without user impact (they still get AQUARI that      │
  │    month, just slightly delayed)                        │
  │ 5. No user funds at risk during outage — USDC stays     │
  │    safe in user's wallet                                │
  └─────────────────────────────────────────────────────────┘

  Key point: outage = delayed execution, NOT fund loss.
  This is much lower severity than Proposal C's key compromise risk.
```

---

## 8. Admin Perspective

### 8.1 What Admin Does

| Task | Frequency | Action |
|------|-----------|--------|
| Trigger monthly payments | Monthly | Backend calls Privy SDK per user |
| Forward USDC (fiat users) | Monthly | Send settled USDC from merchant to user wallets |
| Fund treasury | As needed | Deposit AQUARI for affiliate rewards + bonuses |
| Handle failed fiat payments | On webhook | Pause subscription, monitor retry |
| Process cancellations | On request | DB update only (no refund tx needed) |
| Monitor Privy SDK health | Ongoing | Check API response times, error rates |
| Retry failed transfers | As needed | Re-send affiliate/bonus from treasury |
| Review delegation policies | Quarterly | Ensure policies match current contract addresses |

### 8.2 What Admin Sees in Dashboard

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
| PRIVY SDK HEALTH                                                  |
|                                                                  |
| Status: ✓ Online                                                  |
| Avg response time (24h): 340ms                                   |
| Failed requests (24h): 0                                         |
| Active delegation policies: 2 (approve, swap)                    |
| Last policy update: 2025-12-01                                   |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| ALERTS                                                           |
|                                                                  |
| [!] Treasury wallet below 500k AQUARI — top up soon              |
| [!] 2 affiliate transfers pending retry                          |
| [!] Privy SDK latency spike: 800ms avg (last 15 min)            |
+------------------------------------------------------------------+
```

---

## 9. Complete Transaction Summary

### 9.1 User Transactions (User Pays Gas)

| Action | Crypto User | Fiat User |
|--------|------------|-----------|
| Onboarding (Privy sign-in) | 0 txs | 0 txs |
| KYC | Not needed | Off-chain (Inflow) |
| Subscription setup | **0 txs** | 0 txs |
| Monthly payments | 0 txs | 0 txs |
| **Total user txs over 12 months** | **0** | **0** |

### 9.2 Admin/Backend Transactions (Platform Pays Gas)

| Action | Txs | Frequency |
|--------|-----|-----------|
| Forward USDC to user wallet (fiat only) | 1 | Monthly per fiat user |
| Approve USDC (via Privy SDK, from user wallet) | 1 | Monthly per user |
| Swap USDC → AQUARI (via Privy SDK, from user wallet) | 1 | Monthly per user |
| Affiliate reward (if referrer) | +1 | Monthly per referred user |
| Completion bonus (if final month) | +1 | Once per completed user |

### 9.3 Total Txs Over a 12-Month Commitment

| Scenario | Crypto User | Fiat User |
|----------|------------|-----------|
| Setup | 0 | 0 |
| 12 monthly swaps (approve + swap via Privy) | 24 (admin pays) | 24 (admin pays) |
| 12 USDC forwards (fiat only) | — | 12 (admin pays) |
| 12 affiliate rewards (if referrer) | +12 | +12 |
| Completion bonus | +1 | +1 |
| **Total: no referrer** | **25** | **37** |
| **Total: with referrer** | **37** | **49** |

**Same tx count as Proposal C.** The difference is security model, not transaction count.

---

## 10. Pros & Cons

### Pros
| # | Advantage |
|---|-----------|
| 1 | **No private key storage** — keys never touch our infrastructure. Privy holds them in their secure enclave |
| 2 | **Swap from user's own wallet** — same on-chain transparency as Proposal C |
| 3 | **Policy-constrained** — Privy only allows specific contracts + functions. Our backend CANNOT drain wallets even if compromised |
| 4 | **No custom smart contract** — no audit needed |
| 5 | **No co-mingled funds** — USDC in individual user wallets |
| 6 | **Zero refund txs on cancellation** — USDC already in user's wallet |
| 7 | **Cleanest regulatory position** — we are NOT key custodians. Privy is the custodian |
| 8 | **Backend compromise is limited** — attacker can only execute allowed swap txs, not arbitrary transfers |
| 9 | **Professional key management** — Privy handles HSM, encryption, key rotation. Not our burden |

### Cons
| # | Disadvantage |
|---|--------------|
| 1 | **Two external dependencies** — backend must talk to both Inflow AND Privy servers. In Proposal C, only Inflow is external; signing is self-contained |
| 2 | **Privy dependency** — if Privy server SDK is down, swaps cannot execute (delayed, not lost) |
| 3 | **Privy team cooperation required** — server delegation for embedded wallets may need coordination with Privy team |
| 4 | **Highest tx count** — same as Proposal C (approve + swap + forward per user per month) |
| 5 | **Not atomic** — same partial failure challenges as Proposals B and C |
| 6 | **Privy plan requirements** — may need a higher-tier Privy plan for server wallet features |
| 7 | **Less control** — bound by Privy's policy framework and SDK capabilities |
| 8 | **Latency** — Privy server signing adds network roundtrip per tx (our server → Privy server → Base RPC, vs C where it's our server → Base RPC directly) |
| 9 | **Unknown feasibility** — must confirm Privy supports this for embedded wallets specifically |

---

## 11. Security Comparison: C vs D

| Factor | C (Self-Managed Keys) | D (Privy Delegation) |
|--------|----------------------|---------------------|
| **Who holds private keys** | Us (encrypted in DB) | Privy (their infrastructure) |
| **DB breach impact** | **All user wallets exposed** | No keys to steal |
| **Backend compromise impact** | Attacker can sign any tx | Attacker can only sign policy-allowed txs |
| **Key management burden** | On us (encryption, rotation, HSM) | On Privy (professional infra) |
| **Regulatory custody** | We are custodians | Privy is custodian |
| **External server dependencies** | Inflow only (signing is local) | **Inflow + Privy** (both must be up) |
| **Dependency risk** | None for signing (self-contained) | Privy uptime / API availability |
| **Operational overhead** | Key security infra (ours to maintain) | Privy manages it (their responsibility) |
| **Worst-case scenario** | All funds drained | Delayed swaps (no fund loss) |

**Bottom line**: D trades self-reliance for dramatically lower security risk. The worst case in D is delayed swaps. The worst case in C is total fund loss.

---

## 12. Side-by-Side: All Proposals

| Factor | A (Contract) | B (Backend) | C (Self-Keys) | D (Privy Server) |
|--------|-------------|-------------|---------------|-----------------|
| **Who swaps** | Contract | Admin wallet | User wallet (our key) | User wallet (Privy key) |
| **USDC custody** | Per-user vault | Co-mingled | User's wallet | User's wallet |
| **Key custody** | N/A | 1 admin key | N user keys | **Privy holds all** |
| **Monthly txs (fiat, no ref)** | 1 | 1 | 3 | 3 |
| **Monthly txs (fiat, with ref)** | 1 | 2 | 4 | 4 |
| **Total txs / 12mo (fiat, ref)** | 12 | 24 | 49 | 49 |
| **Atomicity** | Full | Partial | Partial | Partial |
| **Refund on cancel** | 1 tx | 1 tx | 0 txs | 0 txs |
| **Contract needed** | Yes | No | No | No |
| **Key compromise blast radius** | Contract funds | Platform wallet | **All wallets** | **None (Privy's problem)** |
| **External server dependencies** | Inflow | Inflow | **Inflow only** | **Inflow + Privy** |
| **Privy dependency** | Auth only | Auth only | Auth only | **Auth + Signing** |
| **Regulatory custody risk** | Low | Medium | High | **Low** |
| **Ship speed** | Slowest | Fastest | Medium | Medium + Privy lead time |
| **On-chain transparency** | Medium | Low | High | **High** |
| **Unique risk** | Contract bugs | Partial failures | DB key breach | Privy outage |
