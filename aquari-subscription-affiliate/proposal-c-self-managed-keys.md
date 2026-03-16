# AQUARI Subscription & Affiliate — Proposal C: Self-Managed Wallet Keys

> **Project**: [AQUARI Subscription & Affiliate System](https://github.com/Mohsinsiddi/project-md-plans/tree/main/aquari-subscription-affiliate)
> **Proposal**: Backend stores encrypted user wallet keys. Swaps execute FROM user's own Privy wallet — no admin wallet does the swap. No Privy server SDK dependency.

---

## 1. System Overview

```
+------------------+       +----------------------------------+       +-------------------+
|                  |       |                                  |       |                   |
|   Aquari Web App |       |   Backend API (Node.js)          |       | Uniswap V2 Router |
|   (Privy Auth)   |------>|                                  |------>| (Base)            |
|                  |       |  +----------------------------+  |       +-------------------+
+------------------+       |  | Encrypted Key Store        |  |                |
                           |  | (user wallet private keys) |  |                v
                           |  | AES-256-GCM encrypted      |  |       +-------------------+
                           |  | in PostgreSQL              |  |       | AQUARI Token      |
                           |  +----------------------------+  |       | (fee-on-transfer) |
                           |                                  |       +-------------------+
                           |  +----------------------------+  |
                           |  | Merchant Wallet            |  |
                           |  | (receives Inflow USDC)     |  |
                           |  +----------------------------+  |
                           |                                  |
                           |  +----------------------------+  |
                           |  | Treasury Wallet            |  |
                           |  | (holds AQUARI for          |  |
                           |  |  affiliate + bonus)        |  |
                           |  +----------------------------+  |
                           |                                  |
                           +----------------+-----------------+
                                            |
                                            v
                           +------------------+
                           |                  |
                           |   PostgreSQL     |
                           |   (All state +   |
                           |    encrypted     |
                           |    keys)         |
                           +------------------+
```

**Key difference from Proposals A/B**: The swap originates FROM the user's own Privy embedded wallet. The backend holds encrypted copies of user wallet private keys and signs swap transactions on their behalf. No admin wallet executes swaps. USDC sits in individual user wallets, not in a co-mingled platform wallet.

**Key advantage over Proposal D**: Our server only communicates with **one external service** — Inflow (for fiat payments). All transaction signing is self-contained — we hold the keys, we sign locally, we submit directly to Base RPC. No dependency on Privy's server SDK for signing. If Privy's servers go down, our swap pipeline is unaffected.

```
Proposal C — Server Communication:

  Our Backend
      │
      ├──→ Inflow API (fiat payments + webhooks)
      │    External dependency — cannot avoid
      │
      ├──→ PostgreSQL (encrypted keys + state)
      │    Internal — we control this
      │
      └──→ Base RPC (sign locally, submit tx)
           Internal signing — no external dependency

  Only 1 external dependency for the entire swap pipeline.
  Signing is local. Submission is direct. No middleman.
```

---

## 2. User Onboarding Flow

### 2.1 New User (Direct)

```
Step 1          Step 2              Step 3                  Step 4
+----------+    +---------------+   +------------------+    +------------------+
| User hits |    | Sign in with  |   | Privy creates   |    | Backend extracts |
| aquari.io |--->| Privy wallet  |-->| embedded wallet  |--->| wallet key,      |
|           |    | (1 click)     |   | on Base          |    | encrypts with    |
+----------+    +---------------+   +------------------+    | AES-256-GCM,     |
                                                            | stores in DB     |
                                                            +------------------+

User sees: normal Privy sign-in
Behind the scenes: key captured and encrypted at registration
```

### 2.2 Referred User (via Affiliate Link)

```
Step 1                  Step 2              Step 3              Step 4
+------------------+    +---------------+   +---------------+   +-----------------+
| User clicks      |    | Sign in with  |   | Ref code auto |   | Key extracted   |
| aquari.io/       |--->| Privy wallet  |-->| captured from |-->| + encrypted     |
| ?ref=ABC123      |    | (1 click)     |   | URL & stored  |   | Same as 2.1     |
+------------------+    +---------------+   +---------------+   +-----------------+
```

### 2.3 Key Extraction & Storage

```
During Privy embedded wallet creation:

+------------------+     +------------------+     +------------------+
| Privy creates    |     | Backend extracts |     | Private key      |
| embedded wallet  |---->| private key via  |---->| encrypted with   |
| for user         |     | Privy client SDK |     | AES-256-GCM      |
+------------------+     | (exportWallet)   |     | using server     |
                         +------------------+     | master key       |
                                                  |                  |
                                                  | Stored in DB:    |
                                                  | encrypted_key,   |
                                                  | iv, auth_tag     |
                                                  +------------------+

  Master encryption key: stored in environment variable or secrets manager
  NEVER in the database.
  Key only decrypted in-memory during swap execution.
```

### 2.4 Terms & Conditions

```
+------------------------------------------------------------------+
|                                                                  |
|   Before subscribing, user MUST accept:                          |
|                                                                  |
|   [x] I authorize AQUARI platform to execute token swap          |
|       transactions on my behalf from my wallet                   |
|                                                                  |
|   [x] I understand that monthly USDC-to-AQUARI swaps will       |
|       be performed automatically from my wallet                  |
|                                                                  |
|   [x] I acknowledge the platform will manage transaction         |
|       signing for the purpose of subscription fulfillment only   |
|                                                                  |
|   [ Continue to Payment ]                                        |
|                                                                  |
+------------------------------------------------------------------+

  Consent stored in DB: user.terms_accepted = true, terms_accepted_at
  Required BEFORE any swap can be executed
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
Step 1              Step 2              Step 3
+---------------+   +---------------+   +------------------+
| User picks    |   | User already  |   | Backend confirms |
| "Pay with     |-->| has USDC in   |-->| USDC balance in  |
| Crypto"       |   | their Privy   |   | user's wallet.   |
|               |   | wallet        |   | Subscription     |
+---------------+   | (or sends     |   | ACTIVE.          |
                    | USDC to it)   |   +------------------+
                    +---------------+
                                         Zero KYC.
                                         USDC stays in user's own wallet.
                                         No transfer to platform.
```

**On-chain transactions (user pays gas):**

| # | Transaction | What Happens | Gas |
|---|-------------|-------------|-----|
| — | None (or optional USDC self-transfer) | USDC already in user's Privy wallet | ~0-65k |
| **Total** | **0-1 tx** | **USDC stays in user's wallet** | **Minimal** |

**Major difference from A/B**: No deposit to vault. No transfer to platform wallet. USDC stays in the user's own wallet until swap day.

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

  Inflow settles USDC to our MERCHANT wallet (we are the merchant).
  We then FORWARD USDC to user's Privy wallet before swap.
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
      |                    |      | (from user's wallet)   |    | (only if referrer)     |
      | Merchant wallet    |      |                        |    |                        |
      | sends USDC to      |      | Backend loads user's   |    | Treasury wallet sends  |
      | user's Privy       |      | encrypted key from DB  |    | AQUARI to referrer     |
      | wallet             |      | Decrypts in memory     |    |                        |
      |                    |      |                        |    +------------------------+
      | (admin pays gas)   |      | Signs from user wallet:|
      +--------------------+      | TX 2: USDC.approve()   |
                                  | TX 3: swap...          |
                                  |   SupportingFee...()   |
                                  |   to = user's wallet   |
                                  |                        |
                                  | (admin pays gas via    |
                                  |  gas sponsorship or    |
                                  |  user wallet ETH)      |
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
  Same as fiat flow above, but SKIP TX 1 (forward USDC).
  USDC is already in user's Privy wallet.
  Backend goes straight to TX 2-3 (approve + swap).
```

### 4.3 Monthly Transaction Count (Platform Pays Gas)

| Scenario | Txs | Details |
|----------|-----|---------|
| Monthly swap — crypto, no referrer | **2** | approve (1) + swap (1) from user's wallet |
| Monthly swap — fiat, no referrer | **3** | forward USDC (1) + approve (1) + swap (1) |
| Monthly swap — with referrer | **+1** | + affiliate transfer from treasury |
| Final month — no referrer | **+1** | + bonus transfer from treasury |
| Final month — with referrer | **+2** | + affiliate (1) + bonus (1) from treasury |

### 4.4 Gas Payment Strategy

```
Who pays gas for txs signed by user's wallet?

Option 1: Pre-fund user wallet with ETH
  - Backend sends small ETH amount to user's Privy wallet
  - User wallet pays its own gas
  - Simple but adds another tx

Option 2: EIP-4337 Account Abstraction / Paymaster
  - Platform sponsors gas via a paymaster contract
  - User wallet pays zero gas
  - More complex but cleaner UX

Option 3: Relay / meta-transactions
  - Backend relays signed tx through a gas relay
  - User wallet signs, relay submits + pays gas

Recommendation: Option 1 for MVP (simplest), migrate to Option 2 later.
```

---

## 5. Final Month — Completion Flow

```
                     FINAL MONTH (month 6 of 6, or 12 of 12)

  +------------------------------------------------------------------+
  | BACKEND EXECUTES 3-5 SEPARATE TXS:                               |
  |                                                                  |
  | TX 1: Forward USDC to user wallet (fiat only)     ← if fiat     |
  |                                                                  |
  | TX 2: USDC.approve(router, amount)                 ← from user  |
  |       signed with user's encrypted key                           |
  |                                                                  |
  | TX 3: Swap USDC → AQUARI (to = user wallet)        ← from user  |
  |       via Uniswap V2 Router                                     |
  |                                                                  |
  | TX 4: Send affiliate reward (if referrer)           ← treasury  |
  |       AQUARI.transfer(referrerWallet, reward)                    |
  |                                                                  |
  | TX 5: Send completion bonus                         ← treasury  |
  |       AQUARI.transfer(userWallet, bonus)                         |
  |       bonus = totalAquariDeposited * 5%                          |
  |                                                                  |
  +------------------------------------------------------------------+

  Same partial failure risks as Proposal B.
  Backend MUST track per-tx status and retry failed transfers.
```

### Completion Bonus Example

```
12-month commitment at $100/month:

Month 1-11: forward (fiat) + approve + swap → AQUARI to user (2-3 txs each)
Month 12:   forward (fiat) + approve + swap              (TX 1-3)
            + affiliate reward to referrer                (TX 4, if referrer)
            + 5% bonus to user                            (TX 5)

Total AQUARI: ~120,000 + 6,000 bonus = ~126,000
Total txs on final month (fiat + referrer): 5
```

---

## 6. Cancellation & Edge Cases

### 6.1 User Cancels Mid-Commitment

```
+------------------+     +------------------+     +--------------------+
| User requests    |     | Backend:         |     | USDC is already    |
| cancellation     |---->| Mark sub as      |---->| in user's wallet   |
| (via app)        |     | CANCELLED in DB  |     | — NO REFUND TX     |
+------------------+     +------------------+     | NEEDED             |
                                                  +--------------------+

  User keeps: All AQUARI received so far + unswapped USDC (already in their wallet)
  User loses: The 5% completion bonus
  Refund:     NOT NEEDED — USDC never left user's wallet (crypto users)
              Fiat users: any forwarded but unswapped USDC is already in their wallet
  Txs:        0 (just a DB update)
```

**This is a major UX advantage over Proposals A/B** — no refund transaction needed.

### 6.2 Fiat Payment Fails

```
Same as Proposals A/B:
Webhook → pause subscription → Inflow retries → 30 days → reset commitment
No on-chain action needed.
```

### 6.3 Swap Fails (Tx Reverts)

```
+------------------+     +------------------+
| Swap tx from     |     | TX REVERTS       |
| user's wallet    |---->|                  |
| reverts          |     | - USDC stays in  |
| (slippage, etc.) |     |   USER'S wallet  |
+------------------+     | - Retry safe     |
                         +------------------+

  ✓ Swap revert is safe — USDC is in user's own wallet, not lost.

  ⚠️ Same partial failure risk as Proposal B:
  If swap succeeds but affiliate transfer fails → retry needed.
```

### 6.4 Key Compromise — Critical Risk

```
  THIS IS THE #1 RISK OF PROPOSAL C

  If the encrypted key store (database) is breached:

  +------------------+     +------------------+     +--------------------+
  | Attacker gains   |     | Attacker also    |     | Attacker can sign  |
  | access to DB     |---->| obtains master   |---->| txs from ANY user  |
  | (encrypted keys) |     | encryption key   |     | wallet             |
  +------------------+     | (env var/secrets |     | → drain all funds  |
                           |  manager)        |     +--------------------+
                           +------------------+

  Blast radius: ALL user wallets compromised simultaneously.

  Mitigations:
  ┌─────────────────────────────────────────────────────────┐
  │ 1. Master key in HSM (Hardware Security Module)         │
  │    — never in env vars on app server                    │
  │ 2. Per-user encryption keys derived from master + salt  │
  │ 3. Key access audit logging — every decrypt logged      │
  │ 4. Rate limiting on key decryption operations           │
  │ 5. IP allowlisting for DB access                        │
  │ 6. Separate DB for keys vs application data             │
  │ 7. Regular key rotation policy                          │
  │ 8. Incident response plan for key compromise            │
  └─────────────────────────────────────────────────────────┘

  Even with all mitigations: storing user private keys is a
  fundamentally higher-risk custody model than Proposals A, B, or D.
```

---

## 7. Admin Perspective

### 7.1 What Admin Does

| Task | Frequency | Action |
|------|-----------|--------|
| Trigger monthly payments | Monthly | Backend auto-executes approve + swap per user |
| Forward USDC (fiat users) | Monthly | Send settled USDC from merchant to user wallets |
| Fund treasury | As needed | Deposit AQUARI for affiliate rewards + bonuses |
| Handle failed fiat payments | On webhook | Pause subscription, monitor retry |
| Process cancellations | On request | DB update (no refund tx needed) |
| Monitor key store security | Ongoing | Audit access logs, check for anomalies |
| Key rotation | Quarterly | Rotate master encryption key |
| Retry failed transfers | As needed | Re-send affiliate/bonus from treasury |

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
| KEY STORE HEALTH                                                  |
|                                                                  |
| Encrypted keys stored: 1,247                                     |
| Last key access audit: 2 hours ago                               |
| Failed decrypt attempts (24h): 0                                 |
| Master key last rotated: 2025-12-01                              |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| ALERTS                                                           |
|                                                                  |
| [!] Treasury wallet below 500k AQUARI — top up soon              |
| [!] 2 affiliate transfers pending retry                          |
| [!] 3 USDC forwards pending (Inflow settlement delayed)         |
| [!] Key access spike: 15 decrypts in 1 minute (investigate)      |
+------------------------------------------------------------------+
```

---

## 8. Complete Transaction Summary

### 8.1 User Transactions (User Pays Gas)

| Action | Crypto User | Fiat User |
|--------|------------|-----------|
| Onboarding (Privy sign-in) | 0 txs | 0 txs |
| KYC | Not needed | Off-chain (Inflow) |
| Subscription setup | **0 txs** | 0 txs |
| Monthly payments | 0 txs | 0 txs |
| **Total user txs over 12 months** | **0** | **0** |

**User does zero on-chain transactions in either path.** All txs are signed by the backend using the user's key.

### 8.2 Admin/Backend Transactions (Platform Pays Gas)

| Action | Txs | Frequency |
|--------|-----|-----------|
| Forward USDC to user wallet (fiat only) | 1 | Monthly per fiat user |
| Approve USDC on user's behalf | 1 | Monthly per user |
| Swap USDC → AQUARI from user's wallet | 1 | Monthly per user |
| Affiliate reward (if referrer) | +1 | Monthly per referred user |
| Completion bonus (if final month) | +1 | Once per completed user |

### 8.3 Total Txs Over a 12-Month Commitment

| Scenario | Crypto User | Fiat User |
|----------|------------|-----------|
| Setup | 0 | 0 |
| 12 monthly swaps (approve + swap) | 24 (admin pays) | 24 (admin pays) |
| 12 USDC forwards (fiat only) | — | 12 (admin pays) |
| 12 affiliate rewards (if referrer) | +12 | +12 |
| Completion bonus | +1 | +1 |
| **Total: no referrer** | **25** | **37** |
| **Total: with referrer** | **37** | **49** |

**Higher tx count than all other proposals.** But all txs are from either user's wallet (swaps) or treasury (rewards). No co-mingled platform wallet.

---

## 9. Security Model

### 9.1 Key Storage Architecture

```
+------------------------------------------------------------------+
|                                                                  |
|   Application Server              Secrets Manager (AWS/GCP)      |
|   ┌──────────────┐               ┌──────────────────────┐       |
|   │              │               │                      │       |
|   │  Backend     │──── reads ───>│  MASTER_ENCRYPT_KEY  │       |
|   │  (Node.js)   │               │  (never in code/DB)  │       |
|   │              │               │                      │       |
|   └──────┬───────┘               └──────────────────────┘       |
|          │                                                       |
|          │ encrypts/decrypts                                     |
|          │ in-memory only                                        |
|          │                                                       |
|          v                                                       |
|   ┌──────────────┐                                               |
|   │              │                                               |
|   │  PostgreSQL  │                                               |
|   │              │                                               |
|   │  users table:│                                               |
|   │  - encrypted │  ← AES-256-GCM ciphertext                    |
|   │    _key      │  ← unique IV per user                        |
|   │  - iv        │  ← auth tag for integrity                    |
|   │  - auth_tag  │                                               |
|   │              │                                               |
|   └──────────────┘                                               |
|                                                                  |
+------------------------------------------------------------------+
```

### 9.2 Risk Comparison

| Risk | Proposal A | Proposal B | Proposal C |
|------|-----------|-----------|-----------|
| Smart contract bug | **HIGH** | None | None |
| Admin wallet compromise | Low | **HIGH** | Low (only merchant wallet) |
| DB breach → fund loss | None | None | **CRITICAL** (all user keys) |
| Co-mingled funds | None (vault) | **YES** | None (per-user wallets) |
| Regulatory custody risk | Low | Medium | **HIGH** (key custodian) |

---

## 10. Pros & Cons

### Pros
| # | Advantage |
|---|-----------|
| 1 | **Swap from user's own wallet** — on-chain, user's address is the buyer. Transparent on BaseScan |
| 2 | **No co-mingled funds** — USDC sits in individual user wallets, not a shared platform wallet |
| 3 | **No custom smart contract** — no audit needed, faster than Proposal A |
| 4 | **Only 1 external dependency (Inflow)** — unlike Proposal D which requires both Inflow AND Privy servers to be up. Signing is local, no server-to-server roundtrip for every swap |
| 5 | **Zero refund txs on cancellation** — USDC is already in user's wallet |
| 6 | **User's wallet shows natural buy history** — "I bought AQUARI" not "admin sent me AQUARI" |
| 7 | **No platform wallet holding user funds** — reduces single-point-of-failure risk |

### Cons
| # | Disadvantage |
|---|--------------|
| 1 | **Storing private keys = custodial risk** — DB breach exposes ALL user wallets simultaneously |
| 2 | **Regulatory classification** — holding user keys may classify platform as a custodian (legal exposure) |
| 3 | **Highest tx count** — approve + swap per user per month, plus USDC forwarding for fiat |
| 4 | **Not atomic** — same partial failure challenges as Proposal B |
| 5 | **Key management overhead** — encryption, rotation, HSM, audit logging, incident response |
| 6 | **Gas complexity** — who pays gas for user-wallet txs? Needs sponsorship or pre-funding |
| 7 | **Privy key export may be restricted** — need to verify Privy allows extracting embedded wallet private keys |
| 8 | **Trust burden shifts to us** — we are now responsible for key security, not Privy |

---

## 11. Side-by-Side: All Proposals

| Factor | A (Contract) | B (Backend) | C (Self-Managed Keys) |
|--------|-------------|-------------|----------------------|
| **Who swaps** | Contract | Admin wallet | User's wallet (our key) |
| **USDC custody** | Per-user vault | Co-mingled wallet | User's own wallet |
| **Key custody** | N/A (contract) | 1 admin key | N user keys (encrypted) |
| **Monthly txs (no referrer, fiat)** | 1 | 1 | 3 |
| **Monthly txs (with referrer, fiat)** | 1 | 2 | 4 |
| **Total txs / 12mo (with referrer, fiat)** | 12 | 24 | 49 |
| **Atomicity** | Full | Partial | Partial |
| **Refund on cancel** | 1 tx | 1 tx | **0 txs** |
| **Smart contract needed** | Yes | No | No |
| **External server dependencies** | Inflow | Inflow | **Inflow only (signing is local)** |
| **Privy server SDK needed** | No | No | **No** |
| **Key compromise blast radius** | Contract funds | Platform wallet | **All user wallets** |
| **Regulatory risk** | Low | Medium | **High** |
| **Time to ship** | Longest | Fastest | Medium |
| **On-chain transparency** | Medium | Low | **High** |
