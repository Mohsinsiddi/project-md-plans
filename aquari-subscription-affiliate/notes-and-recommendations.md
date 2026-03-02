# AQUARI Subscription & Affiliate — Notes, Recommendations & Risk Assessment

> **Project**: [AQUARI Subscription & Affiliate System](https://github.com/Mohsinsiddi/project-md-plans/tree/main/aquari-subscription-affiliate)

---

## 1. Inflow Dependency — Critical Path

Cooperation from the Inflow team is **the single biggest dependency** for this project.

### What We Need From Inflow
- Webhook event types and payload schemas (undocumented)
- Whether we can pass custom metadata (userId, subscriptionId) and get it echoed in webhooks
- Auto-payout vs manual withdrawal flow
- Webhook signature verification method
- Sandbox/test environment access

### Risk
- **Estimated dev time: 3-4 weeks**, but could extend by additional days/weeks if Inflow team is slow to respond
- Their API docs are incomplete — subscription and webhook details are thin
- If Inflow returns unexpected data structures or changes their API, it cascades into backend changes → API changes → UI state changes → more debugging
- Every unanswered question from Inflow blocks a piece of integration work

### Mitigation
- Establish a direct communication channel with Inflow dev team ASAP (Slack/Discord/email)
- Get a sandbox API key and test subscription + webhook flow end-to-end before building anything on top
- Build the backend with an **Inflow adapter layer** — isolate all Inflow-specific logic so when their responses change, we only update one place, not the entire app
- Document every Inflow API response we receive during testing — don't trust their docs alone

---

## 2. Referral Reward Split — Recommendation

### Current Design (from PRD)
Referrer gets 100% of the 5% entry tax equivalent → **referrer gets 5% of purchase**

### Problem
- The referred user (the one actually buying) gets nothing extra for being referred
- No incentive for the referred user to use a referral link vs signing up directly
- Referrer takes the full reward — feels one-sided

### Recommendation: 75/25 Split

Split the 5% reward between **committer (buyer)** and **referrer**:

| Recipient | Share | On $100 Purchase |
|-----------|-------|-----------------|
| Committer (referred user) | 75% of 5% = **3.75%** | ~$3.75 in AQUARI |
| Referrer (who shared link) | 25% of 5% = **1.25%** | ~$1.25 in AQUARI |

### Why This Is Better
- **Both parties earn** — the referred user has a reason to use the link
- Referred user feels rewarded for committing, not just the referrer
- Creates a **mutual benefit** — "use my link, we both earn"
- Better conversion — referred users are more likely to subscribe when they get something too
- Still costs us the same total amount from treasury (5%)

### Alternative Splits to Consider

| Split | Committer | Referrer | Vibe |
|-------|-----------|----------|------|
| 75/25 | 3.75% | 1.25% | Committer-first (recommended) |
| 50/50 | 2.5% | 2.5% | Equal partnership |
| 60/40 | 3% | 2% | Balanced |
| 0/100 | 0% | 5% | Current PRD — referrer takes all |

**Decision needed from Aquari team.**

---

## 3. Architecture Recommendation: Take Full Ownership

### The Problem With Split Ownership

If the work is split across multiple people/teams:

```
Current risk:

  You (DotMatrix)          Furqan / Others
  ─────────────            ──────────────
  Smart Contract            Backend APIs
  UI/Frontend               Inflow Integration?
       │                         │
       └────── dependency ───────┘
               │
        Need calls to explain
        Need to align on data shapes
        Need to sync on API changes
        If one person is busy → blocks the other
        If Inflow changes something → both sides update
```

### The Reality

- The UI already exists (layout + CSS) but needs **significant debugging and integration**
- Inflow API responses are unpredictable — if they return different data, it affects:
  - Backend API endpoints (shape of data)
  - Frontend state management (what to expect)
  - UI components (what to display)
- Every Inflow change creates a **cascade**: Inflow → Backend → API → UI → States
- If different people own different layers, every change requires:
  - A call to explain what changed
  - Agreement on new data shapes
  - Both sides updating their code
  - Testing together
  - This cycle repeats every time Inflow surprises us

### Recommendation: Single Owner for Full Stack

```
Recommended:

  One person owns everything:
  ┌──────────────────────────────┐
  │  Smart Contract (Base)       │
  │  Backend APIs (Node.js)      │
  │  Inflow Integration          │
  │  UI Integration & Debugging  │
  │  State Management            │
  └──────────────────────────────┘
          │
          One brain.
          One context.
          No calls to sync.
          Inflow changes? Update once, end-to-end.
          API shape changes? Update backend + UI together.
          Fast iteration. No blocking.
```

### Why This Is Better
1. **No coordination overhead** — no calls to explain, no waiting for the other person
2. **Inflow volatility contained** — when Inflow returns unexpected data, one person updates the entire chain (adapter → API → UI) in one session
3. **API and UI stay in sync** — the person building the API also builds the UI that consumes it, so the data shapes always match
4. **Faster debugging** — full context across all layers means faster root-cause analysis
5. **No blocking** — if someone is busy, the project doesn't stall
6. **Smart contract + backend coherence** — the contract interface and the backend that calls it are designed by the same person, so they fit naturally

### The Tradeoff
- More work for one person
- Single point of failure (if that person is unavailable)
- But: **the alternative (split ownership with async coordination) is slower and more fragile for a project this interconnected**

---

## 4. Timeline Estimate

| Phase | Duration | Dependency |
|-------|----------|------------|
| Inflow sandbox testing + webhook discovery | Week 1 | Inflow team responsiveness |
| Smart contract development + testing | Week 1-2 | None (parallel with Inflow) |
| Backend APIs + Inflow integration | Week 2-3 | Inflow webhook schemas confirmed |
| UI debugging + integration + state management | Week 2-4 | Backend APIs stable |
| Admin dashboard | Week 3-4 | Backend APIs stable |
| End-to-end testing | Week 4 | All components ready |
| **Total estimated** | **3-4 weeks** | **+ buffer for Inflow delays** |

### Risk Factors That Add Time
| Risk | Impact | Likelihood |
|------|--------|------------|
| Inflow slow to respond | +1-2 weeks | High |
| Inflow API changes mid-build | +3-5 days | Medium |
| AQUARI/USDC liquidity issues on Base | +1 week (need alt path) | Low |
| Smart contract audit needed before mainnet | +1-2 weeks | High (if Proposal A) |
| UI has more bugs than expected | +3-5 days | Medium |

---

## 5. Recommendation: Go With Proposal A (Smart Contract)

Given the decision for single ownership of the full stack:

### Why Proposal A Makes More Sense
1. **Atomic txs** — 1 tx per month per user, no partial failure states to manage in backend
2. **Less backend complexity** — contract handles swap + affiliate + bonus logic; backend just calls one function
3. **Vault gives crypto users a clean UX** — segregated funds, no co-mingling
4. **On-chain auditability** — every payment verifiable on BaseScan, builds trust
5. **Simpler error handling** — tx reverts = safe, vs Proposal B where you need retry jobs for partial failures
6. **Full control** — owning the contract means you control the entire pipeline end-to-end

### Development Order
```
1. Smart contract (can build + test without Inflow)
2. Backend + Inflow integration (parallel with contract testing)
3. UI integration (once backend APIs are stable)
4. Admin dashboard (last — least dependent on Inflow)
```

The contract can be built and tested on a Base testnet **immediately**, without waiting for Inflow. This is the most parallelizable path.

---

## 6. Summary of Action Items

| # | Action | Priority | Owner |
|---|--------|----------|-------|
| 1 | Establish Inflow dev communication channel | Urgent | Aquari team |
| 2 | Get Inflow sandbox API key | Urgent | Aquari team |
| 3 | Decide referral split (75/25 vs 100/0 vs other) | High | Aquari team |
| 4 | Confirm single-owner full-stack approach | High | DotMatrix + Aquari |
| 5 | Confirm AQUARI/USDC V2 pool exists on Base | High | DotMatrix |
| 6 | Start smart contract development | High | DotMatrix |
| 7 | Test Inflow subscription + webhook in sandbox | High | DotMatrix |
| 8 | Decide treasury pre-funding amount | Medium | Aquari team |
| 9 | Plan smart contract audit timeline | Medium | DotMatrix + Aquari |
