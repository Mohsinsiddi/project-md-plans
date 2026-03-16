# AQUARI Subscription & Affiliate — Changelog

> What changed, why the scope increased, and why it matters.
>
> Full proposals: [Proposal C (Self-Managed Keys)](./proposal-c-self-managed-keys.md) | [Proposal D (Privy Server Delegation)](./proposal-d-privy-server-delegation.md)

---

## Summary

Two new architecture proposals have been added — [Proposal C](./proposal-c-self-managed-keys.md) and [Proposal D](./proposal-d-privy-server-delegation.md). Both explore a fundamentally different approach: **executing swaps FROM the user's own Privy wallet** instead of from a platform/admin wallet. This was driven by client requirements around USDC forwarding, on-chain transparency, and the trust model. The scope of work has increased accordingly.

---

## 1. What Changed From Proposal B

### The Core Shift

Proposal B: **Admin wallet buys AQUARI → sends to user**
Proposals C/D: **User's own wallet buys AQUARI → stays in user's wallet**

This is not a small change. It inverts who the on-chain buyer is.

### Before vs After

| Aspect | Proposal B (Before) | Proposals C/D (After) |
|--------|--------------------|-----------------------|
| Who executes the swap | Platform admin wallet | User's own Privy wallet |
| Where USDC sits before swap | Co-mingled platform wallet | User's individual wallet |
| Swap `msg.sender` on-chain | Admin EOA | User's wallet address |
| On-chain appearance | "Admin bought AQUARI, sent to user" | "User bought AQUARI directly" |
| Inflow USDC flow | Settles to platform wallet, stays there | Settles to merchant, **forwarded to user wallet** |
| Private key management | 1 admin key (simple) | N user keys (C) or Privy delegation (D) |
| Cancellation refund | Need to send USDC back (1 tx) | No refund needed — USDC in user's wallet |

### Two Approaches to the Same Goal

| | Proposal C | Proposal D |
|-|-----------|-----------|
| **How we sign user txs** | Store encrypted private keys in our DB | Use Privy's server SDK for delegated signing |
| **Key custody** | Us | Privy |
| **External server dependencies** | **Inflow only** (signing is local) | **Inflow + Privy** (both must be up) |
| **Privy dependency** | Auth only | Auth + signing |
| **Security risk** | DB breach = all keys exposed | Privy outage = delayed swaps (no fund loss) |
| **Regulatory position** | We are custodians | Privy is custodian |
| **Signing latency** | Local (fast) | Server → Privy → Base (extra roundtrip) |

---

## 2. Why the Scope Increased

### 2.1 New Engineering Work Required

The original Proposal B is a relatively straightforward backend: receive USDC, call Uniswap, send AQUARI. Proposals C and D add significant new layers:

**Proposal C adds:**
- Encrypted key store architecture (AES-256-GCM, per-user IVs, master key management)
- Key extraction from Privy embedded wallets during onboarding
- Transaction signing service (load key, decrypt, sign, submit)
- Security infrastructure (audit logging, rate limiting, key rotation)
- Gas sponsorship or pre-funding mechanism for user wallets
- USDC forwarding pipeline (merchant → user wallet)
- Terms & conditions consent flow in UI

**Proposal D adds:**
- Privy server SDK integration and testing
- Delegation policy configuration and management
- Privy-specific error handling (outages, rate limits, policy rejections)
- Gas sponsorship via Privy or separate paymaster
- USDC forwarding pipeline (merchant → user wallet)
- Terms & conditions consent flow in UI
- Coordination with Privy team (potentially weeks of back-and-forth)

### 2.2 Work That Didn't Exist in Proposal B

| New Work Item | Proposal C | Proposal D | Est. Time |
|---------------|-----------|-----------|-----------|
| USDC forwarding system (merchant → user wallets) | Yes | Yes | 2-3 days |
| Key extraction & encryption infrastructure | Yes | No | 3-5 days |
| Privy server SDK integration | No | Yes | 3-5 days |
| Policy/allowlist configuration | No | Yes | 1-2 days |
| Transaction signing service | Yes | Via Privy | 2-3 days |
| Gas sponsorship mechanism | Yes | Yes | 2-3 days |
| Terms & conditions UI flow | Yes | Yes | 1-2 days |
| Security audit of key store | Yes | No | 2-3 days |
| Privy team coordination | No | Yes | 1-2 weeks (variable) |
| Retry/reconciliation (same as B) | Yes | Yes | 2-3 days |
| Testing delegated signing end-to-end | No | Yes | 2-3 days |
| **Total additional work vs B** | | | **~2-3 weeks** |

### 2.3 Timeline Impact

| | Proposal B | Proposal C | Proposal D |
|-|-----------|-----------|-----------|
| Estimated dev time | 3-4 weeks | **4-5 weeks** | **4-5 weeks + Privy lead time** |
| Blocked by Inflow | Yes | Yes | Yes |
| Blocked by Privy team | No | No | **Yes** |
| Security review needed | Minimal | **Significant** | Minimal |
| New infrastructure | None | Key store | Privy enterprise |

---

## 3. Why This Approach Matters

### 3.1 Trust & Transparency

In Proposal B, when a user checks BaseScan, they see:
```
Admin wallet (0xABCD...) → bought AQUARI → sent to my wallet (0x1234...)
```

In Proposals C/D, they see:
```
My wallet (0x1234...) → bought AQUARI → received in my wallet (0x1234...)
```

This is a **fundamentally different trust story**. The user's wallet is the buyer. Their transaction history reads naturally. No mysterious admin wallet appears. For DeFi-native users, this matters.

### 3.2 Fund Safety

| Risk | B | C/D |
|------|---|-----|
| All user USDC in one wallet | **Yes** (co-mingled) | **No** (per-user wallets) |
| Single wallet = single point of failure | **Yes** | **No** |
| Admin wallet compromise = all funds lost | **Yes** | **No** |
| Cancellation refund needed | Yes (send USDC back) | No (already in user's wallet) |

In Proposal B, if the admin wallet is compromised, every user's unswapped USDC is at risk. In C/D, funds are distributed across individual wallets.

### 3.3 Regulatory & Legal

Holding user funds in a co-mingled wallet (Proposal B) creates implicit custody obligations. Proposal D, where Privy holds the keys and we only have delegated permission, is the cleanest regulatory position.

### 3.4 On-Chain Auditability

Every swap is traceable to the specific user's wallet. No need to trust that the platform's admin wallet "eventually" distributed correctly. The blockchain itself proves each user bought their own AQUARI.

---

## 4. Why C/D Over B

### 4.1 What C/D Adds Over Proposal B

| Benefit | Proposal B (baseline) | C/D |
|---------|----------------------|-----|
| On-chain transparency | Admin buys for user | **User buys directly** |
| Fund safety model | Co-mingled wallet | **Per-user wallets** |
| Cancellation UX | Refund tx needed | **Zero txs, instant** |
| User trust story | "Trust us" | **"Verify on-chain"** |
| BaseScan readability | Opaque (admin wallet) | **Clean (user wallet)** |
| Regulatory position | Implicit custody | **Delegated (D) or explicit custody (C)** |
| Single point of failure | Admin wallet | **Distributed** |

### 4.2 Server Dependency Tradeoff: C vs D

This is an important distinction for the client to understand:

```
Proposal C: Our Server ──→ Inflow (1 external dependency)
            Signing is local. If Privy is down, swaps still work.

Proposal D: Our Server ──→ Inflow + Privy (2 external dependencies)
            Every swap requires a roundtrip to Privy's servers.
            If either Inflow OR Privy is down, pipeline stalls.
```

| Factor | C (fewer dependencies) | D (more secure) |
|--------|----------------------|-----------------|
| External services needed | Inflow only | Inflow + Privy |
| Signing latency | Local (ms) | Server → Privy → Base (100-400ms extra) |
| Single point of failure | Our key store | Privy availability |
| What happens if Privy is down | Nothing — swaps continue | **Swaps halt until Privy recovers** |
| Security | We carry the risk | Privy carries the risk |

**C is simpler and more self-reliant. D is more secure but adds operational dependency.**

The right choice depends on what the client values more: independence (C) or security delegation (D).

### 4.3 Why Not Just Ship Proposal B?

Proposal B works. It is the fastest path. But it trades long-term trust and safety for short-term speed:

1. **Co-mingled wallet is a liability** — a single compromise affects all users
2. **On-chain optics matter** — DeFi users expect to see their own wallet in txs
3. **Refund flow is friction** — cancellations require a manual USDC transfer back
4. **Regulatory grey area** — holding all user USDC in one wallet without formal custody registration is risky
5. **Trust doesn't scale** — "trust our admin wallet" works at 100 users, not at 10,000

Proposals C and D solve these structurally, not with band-aids.

---

## 5. Recommendation

### If Privy supports server-side delegation for embedded wallets → **Go with Proposal D**

- No key storage on our side (eliminates the biggest risk)
- Privy handles key security professionally
- Policy constraints limit blast radius even if our backend is compromised
- Cleanest regulatory position
- Worth the Privy coordination time

### If Privy does NOT support this → **Go with Proposal C**

- Same UX benefits (swap from user's wallet, no co-mingled funds)
- But requires serious investment in key security infrastructure
- Higher risk profile — DB breach = all user wallets exposed
- Needs thorough security review before production

### In either case: the additional scope is justified

The client is getting:
- Better security model
- Better on-chain transparency
- Better regulatory positioning
- Better cancellation UX
- Eliminated single point of failure

This is not scope creep. This is a better product.

---

## 6. Document History

| Date | Change | Author |
|------|--------|--------|
| 2025-03-02 | Initial Proposals A and B published | DotMatrix |
| 2025-03-02 | Notes and recommendations published | DotMatrix |
| 2026-03-16 | Proposals C and D added; changelog created | DotMatrix |
