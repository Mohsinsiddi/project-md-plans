# AQUARI Subscription & Affiliate — Proposal B: Backend-Only Architecture

> **Project**: [AQUARI Subscription & Affiliate System](https://github.com/Mohsinsiddi/project-md-plans/tree/main/aquari-subscription-affiliate)
> **Proposal**: Backend wallet calls Uniswap V2 directly. No custom contract. Simpler, faster to ship, but not atomic.

---

## 1. System Overview

```
+------------------+       +------------------+       +-------------------+
|                  |       |                  |       |                   |
|   Aquari Web App |       |   Backend API    |       | Uniswap V2 Router |
|   (Privy Auth)   |------>|   (Node.js)      |------>| (Base)            |
|                  |       |                  |       +-------------------+
+------------------+       |  +------------+  |                |
                           |  | Platform   |  |                v
                           |  | Wallet     |  |       +-------------------+
                           |  | (EOA/      |  |       | AQUARI Token      |
                           |  |  multisig) |  |       | (fee-on-transfer) |
                           |  | holds USDC |  |       +-------------------+
                           |  +------------+  |
                           |                  |
                           |  +------------+  |
                           |  | Treasury   |  |
                           |  | Wallet     |  |
                           |  | holds      |  |
                           |  | AQUARI     |  |
                           |  +------------+  |
                           |                  |
                           +--------+---------+
                                    |
                                    v
                           +------------------+
                           |                  |
                           |   PostgreSQL     |
                           |   (All state)    |
                           |                  |
                           +------------------+
```

**Key difference from Proposal A**: No custom contract. Two separate wallets (platform + treasury) managed by backend. All commitment state lives in PostgreSQL, not on-chain.

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
```

**Identical to Proposal A** — onboarding is the same. Differences start at payment.

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
| User picks    |   | App shows     |   | User sends    |   | Backend detects  |
| "Pay with     |-->| platform      |-->| USDC to       |-->| deposit (event   |
| Crypto"       |   | wallet address|   | platform      |   | listener or      |
|               |   | + total       |   | wallet        |   | manual confirm). |
+---------------+   | deposit amt   |   | (1 tx)        |   | Subscription     |
                    +---------------+   +---------------+   | ACTIVE.          |
                                                            +------------------+
                                                             Zero KYC.
                                                             ~30 seconds.
                                                             1 wallet confirmation.
```

**On-chain transactions (user pays gas):**

| # | Transaction | What Happens | Gas |
|---|-------------|-------------|-----|
| 1 | `USDC.transfer(platformWallet, amount)` | USDC goes to platform wallet | ~65k |
| **Total** | **1 tx** | **One-time setup** | **~65k** |

**Note**: Simpler than Proposal A (1 tx vs 2 txs). BUT funds are co-mingled in the platform wallet with all other users' deposits — no per-user vault separation on-chain.

### 3.3 Payment Method: Fiat (KYC Required — Slower Path)

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

Same as Proposal A — 0 on-chain txs from user.
```

---

## 4. Monthly Execution Flow (Automated — Zero User Action)

### 4.1 Fiat User — Monthly Flow

```
                        AUTOMATED — NO USER ACTION

  +-------------+     +-----------+     +-------------+
  | Inflow      |     | Inflow    |     | Webhook     |
  | charges     |---->| settles   |---->| hits our    |
  | user's card |     | USDC to   |     | backend     |
  |             |     | platform  |     |             |
  +-------------+     | wallet    |     +------+------+
                      +-----------+            |
                                               v
                                    +---------------------+
                                    | BACKEND EXECUTES:   |
                                    +---------------------+
                                               |
                      +------------------------+------------------------+
                      |                                                 |
                      v                                                 v
           +--------------------+                          +------------------------+
           | TX 1: Swap         |                          | TX 2: Affiliate Reward |
           |                    |                          | (only if referrer)     |
           | Uniswap V2 Router: |                          |                        |
           | swapExact...       |                          | AQUARI.transfer(       |
           | SupportingFee...   |   -- if success -->      |   referrerWallet,      |
           | (                  |                          |   rewardAmount         |
           |   to = userWallet  |                          | )                      |
           | )                  |                          |                        |
           +--------------------+                          +------------------------+
                      |                                                 |
                      v                                                 v
           +--------------------+                          +------------------------+
           | User sees AQUARI   |                          | Referrer sees AQUARI   |
           | in Privy wallet    |                          | in their wallet        |
           +--------------------+                          +------------------------+
```

**Key difference from Proposal A**: Tx 1 and Tx 2 are **separate transactions**. If Tx 1 succeeds but Tx 2 fails, the user got their AQUARI but the referrer didn't get paid. Backend needs retry logic.

### 4.2 Crypto User — Monthly Flow

```
                        AUTOMATED — NO USER ACTION

  +-------------------+
  | Backend triggers  |
  | monthly cron /    |
  | scheduler         |
  +--------+----------+
           |
           v
  Same as fiat flow above, but USDC comes from
  the platform wallet balance (already deposited).
  Backend tracks per-user balance in PostgreSQL.
```

### 4.3 Monthly Transaction Count (Admin/Backend Pays Gas)

| Scenario | Txs | Details |
|----------|-----|---------|
| Monthly swap — no referrer | **1** | swap USDC→AQUARI to user |
| Monthly swap — with referrer | **2** | swap (1) + affiliate transfer (1) |
| Final month — no referrer | **2** | swap (1) + bonus transfer (1) |
| Final month — with referrer | **3** | swap (1) + affiliate (1) + bonus (1) |

---

## 5. Final Month — Completion Flow

```
                     FINAL MONTH (month 6 of 6, or 12 of 12)

  +------------------------------------------------------------------+
  | BACKEND EXECUTES 1-3 SEPARATE TXS:                               |
  |                                                                  |
  | TX 1: Swap USDC → AQUARI (to = user wallet)     ← always        |
  |       via Uniswap V2 Router                                     |
  |                                                                  |
  | TX 2: Send affiliate reward (if referrer)        ← if referrer   |
  |       AQUARI.transfer(referrerWallet, reward)                    |
  |       from treasury wallet                                       |
  |                                                                  |
  | TX 3: Send completion bonus                      ← always        |
  |       AQUARI.transfer(userWallet, bonus)                         |
  |       bonus = totalAquariDeposited * 5%                          |
  |       from treasury wallet                                       |
  |                                                                  |
  +------------------------------------------------------------------+

  ⚠️  These are SEPARATE txs. Possible failure scenarios:
  - TX 1 succeeds, TX 2 fails → user got AQUARI, referrer didn't → retry TX 2
  - TX 1 succeeds, TX 3 fails → user got AQUARI, no bonus yet → retry TX 3
  - TX 1 fails → nothing happened → retry everything

  Backend MUST track per-tx status and retry failed transfers.
```

### Completion Bonus Example

```
12-month commitment at $100/month:

Month 1-11: swap → AQUARI to user (1 tx each, +1 if referrer)
Month 12:   swap → AQUARI to user                    (TX 1)
            + affiliate reward to referrer            (TX 2, if referrer)
            + 5% bonus to user                        (TX 3)

Total AQUARI: ~120,000 + 6,000 bonus = ~126,000
Total txs on final month: 2-3
```

---

## 6. Cancellation & Edge Cases

### 6.1 User Cancels Mid-Commitment

```
+------------------+     +------------------+     +--------------------+
| User requests    |     | Backend:         |     | If crypto user:    |
| cancellation     |---->| Mark sub as      |---->| Refund remaining   |
| (via app)        |     | CANCELLED in DB  |     | USDC from platform |
+------------------+     +------------------+     | wallet (1 tx)      |
                                                  +--------------------+

  User keeps: All AQUARI received so far
  User loses: The 5% completion bonus
  Refund:     Backend sends remaining USDC to user's wallet (1 tx)
  No contract call — just DB update + optional USDC refund tx
```

### 6.2 Fiat Payment Fails

```
Same as Proposal A:
Webhook → pause subscription → Inflow retries → 30 days → reset commitment
No on-chain action needed.
```

### 6.3 Swap Fails

```
+------------------+     +------------------+
| Uniswap V2 swap  |     | TX REVERTS       |
| reverts           |---->|                  |
| (slippage, etc.) |     | - USDC stays in  |
+------------------+     |   platform wallet |
                         | - Retry safe     |
                         +------------------+

  ✓ Swap revert is safe — USDC stays in platform wallet.

  ⚠️ BUT if swap SUCCEEDS and affiliate transfer FAILS:
  → User got AQUARI, referrer did not.
  → Backend must detect this and retry the affiliate transfer.
  → DB tracks: swap_tx_hash (success), affiliate_tx_hash (null/failed)
```

### 6.4 Partial Failure Recovery

```
  This is the KEY challenge in Proposal B.
  Backend needs a reconciliation job:

  +------------------+     +------------------+     +------------------+
  | Cron job runs    |     | Finds payments   |     | Retries failed   |
  | every 15 min     |---->| where swap_tx =  |---->| affiliate and/or |
  |                  |     | success but      |     | bonus transfers  |
  +------------------+     | affiliate_tx =   |     |                  |
                           | null/failed      |     +------------------+
                           +------------------+

  DB status per payment:
  - processing  → swap initiated
  - swap_done   → swap succeeded, affiliate/bonus pending
  - completed   → all txs done
  - partial     → some txs failed, needs retry
  - failed      → swap failed, needs full retry
```

---

## 7. Admin Perspective

### 7.1 What Admin Does

| Task | Frequency | Action |
|------|-----------|--------|
| Trigger monthly payments | Monthly | Backend calls Uniswap V2 per user |
| Monitor partial failures | Ongoing | Check for swap_done but affiliate_failed |
| Retry failed transfers | As needed | Re-send AQUARI from treasury |
| Fund treasury wallet | As needed | Send AQUARI to treasury wallet |
| Handle failed fiat payments | On webhook | Pause, monitor Inflow retry |
| Process cancellations | On request | DB update + optional USDC refund tx |
| Reconciliation | Daily | Verify all pending transfers completed |

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
| [!] Treasury wallet below 500k AQUARI — top up soon              |
| [!] 3 failed swaps in last 24h — check liquidity                |
| [!] 2 affiliate transfers pending retry                          |
| [!] User 0x4d.. fiat payment failed — Inflow retrying           |
+------------------------------------------------------------------+

  ⚠️ Note: Proposal B has an extra alert type:
  "Partial payment — swap succeeded but affiliate/bonus transfer failed"
  This does NOT exist in Proposal A (atomic = all or nothing).
```

---

## 8. Complete Transaction Summary

### 8.1 User Transactions (User Pays Gas)

| Action | Crypto User | Fiat User |
|--------|------------|-----------|
| Onboarding (Privy sign-in) | 0 txs | 0 txs |
| KYC | Not needed | Off-chain (Inflow) |
| Subscription setup | **1 tx** (send USDC) | 0 txs |
| Monthly payments | 0 txs | 0 txs |
| **Total user txs over 12 months** | **1** | **0** |

**Note**: Crypto setup is 1 tx here vs 2 txs in Proposal A (no approve needed — just a simple transfer).

### 8.2 Admin/Backend Transactions (Platform Pays Gas)

| Action | Txs | Frequency |
|--------|-----|-----------|
| Monthly swap (USDC → AQUARI to user) | 1 | Monthly per user |
| Affiliate reward (if referrer) | +1 | Monthly per referred user |
| Completion bonus (if final month) | +1 | Once per completed user |
| Retry failed transfers | varies | As needed |

### 8.3 Total Txs Over a 12-Month Commitment

| Scenario | Crypto User | Fiat User |
|----------|------------|-----------|
| Setup | 1 (user pays) | 0 |
| 12 monthly swaps | 12 (admin pays) | 12 (admin pays) |
| 12 affiliate rewards (if referrer) | +12 (admin pays) | +12 (admin pays) |
| Completion bonus | +1 (admin pays) | +1 (admin pays) |
| **Total: no referrer** | **13** | **12** |
| **Total: with referrer** | **25** | **24** |

**Compare to Proposal A**: 14 (crypto) or 12 (fiat) total txs regardless of referrer.

---

## 9. Pros & Cons

### Pros
| # | Advantage |
|---|-----------|
| 1 | **No contract to deploy or audit** — faster to ship, less risk |
| 2 | **Lower gas per swap** — no contract wrapper overhead |
| 3 | **Maximum flexibility** — change business logic with a code deploy, not a contract upgrade |
| 4 | **Simpler crypto onboarding** — 1 tx (transfer) vs 2 txs (approve + deposit) |
| 5 | **Easier to iterate** — pricing, reward %, commitment rules are just backend config |
| 6 | **Less attack surface** — no custom contract holding pooled funds |

### Cons
| # | Disadvantage |
|---|--------------|
| 1 | **Not atomic** — swap and affiliate are separate txs; partial failures possible |
| 2 | **Need retry/reconciliation** — cron job to catch and fix partial payment states |
| 3 | **More txs with referrers** — 2-3 txs/month vs always 1 in Proposal A |
| 4 | **Higher total gas over time** — more txs = more gas (especially with many referred users) |
| 5 | **Co-mingled funds** — all user USDC in one platform wallet, not segregated |
| 6 | **Off-chain state only** — commitment progress is in DB, not verifiable on-chain |
| 7 | **Platform wallet = trust bottleneck** — single EOA/multisig holds all funds |
| 8 | **Partial failure UX** — user might see AQUARI but referrer doesn't get paid until retry |

---

## 10. Side-by-Side: Proposal A vs B

| Factor | A (Contract) | B (Backend-Only) |
|--------|-------------|-----------------|
| **Setup txs (crypto user)** | 2 | 1 |
| **Setup txs (fiat user)** | 0 | 0 |
| **Monthly txs (no referrer)** | 1 | 1 |
| **Monthly txs (with referrer)** | 1 | 2 |
| **Final month txs (with referrer)** | 1 | 3 |
| **Total txs / 12mo (with referrer)** | 14 | 25 |
| **Atomicity** | Full | Partial |
| **Crypto fund safety** | Per-user vault | Co-mingled wallet |
| **Failure handling** | Automatic revert | Manual retry |
| **Time to ship** | Longer | Faster |
| **Flexibility** | Contract redeploy | Code deploy |
| **Gas per tx** | Higher | Lower |
| **Total gas over time** | Lower (fewer txs) | Higher (more txs) |
| **On-chain auditability** | Full | Swap only |
| **Operational complexity** | Lower (1 call) | Higher (retry logic) |
