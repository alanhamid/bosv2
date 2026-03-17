# AiriPay V7 — Multi-Rail GCC Cross-Border Orchestration Layer

> **AiriPay is a bank-embedded orchestration + policy engine that routes each transfer to the best available rail** — AFAQ (intra-GCC RTGS), Buna (regional multi-currency), Arc/USDC (fast global settlement with sub-second onchain finality), or SWIFT (universal fallback) — while keeping **100% custody and execution inside the bank.**
>
> **The Core Problem**: Corporate customers defect to Wise/Revolut/Airwallex because of speed + cost transparency. Banks lose cross-border engagement, then deposit relationships, then credit lines. AiriPay stops the leakage by letting banks offer fintech-speed settlement (AFAQ-instant domestic + near-real-time Arc/USDC cross-border) while retaining custody and deepening banking relationships.
>
> **Banks maintain full control of funds**: RTGS accounts, Circle ARC business accounts, and Buna/SWIFT correspondents remain bank-owned. AiriPay is a **decision + evidence layer**, not a custodian or settlement institution.
>
> **Primary Deployment Model**: Bank-Embedded White-Label — where AiriPay is licensed to GCC banks, integrated into their digital banking platforms, and routes each payment to the optimal rail (domestic RTGS → AFAQ → Buna → Arc/USDC → SWIFT fallback) with just-in-time funding, zero pre-funding, and immutable audit trail.

---

## Repository Operating Model (bosV2)

This repo now includes a **baseline-driven workflow** so future architecture changes can build on the latest accepted state instead of relying on transient chat memory.

Start here before major changes:
- `docs/CURRENT_STATE.md`
- `docs/BASELINE.md`
- `docs/ARCHITECTURE.md`
- `docs/adr/`

AI workflow support added in this repo:
- `.github/copilot-instructions.md`
- `.github/instructions/`
- `.github/prompts/`
- `.claude/skills/`

When architecture changes materially, update the state docs and add or revise an ADR.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Principle](#core-principle)
- [System Architecture Diagram](#system-architecture-diagram)
- [Tech Stack & Third-Party Vendors](#tech-stack--third-party-vendors)
- [End-to-End Payment Flow](#end-to-end-payment-flow)
  - [Phase 0 — Enterprise Onboarding](#phase-0--enterprise-onboarding-one-time)
  - [Phase 1 — Payment Initiation](#phase-1--user-initiates-a-payment)
  - [Phase 2 — Agentic Ranking](#phase-2--agentic-ranking--presentation)
  - [Phase 3 — Execution via Circle ARC](#phase-3--user-selects-circle-arc--airipay-pings-circle)
  - [Phase 4 — Recipient Settlement](#phase-4--recipient-receives-funds)
- [Circle ARC Internals](#what-happens-inside-circle-arc)
- [Fireblocks Integration](#where-fireblocks-fits)
- [Data Model](#airipays-data-model)
- [Subscription Tiers](#subscription-tiers)
- [AiriPay Scope — Does vs. Does Not](#what-airipay-does-vs-what-it-does-not)
- [Deployment Model B — AiriPay for Banks (White-Label)](#deployment-model-b--airipay-for-banks-white-label)
  - [Why Bank-Embedded Is Superior](#why-bank-embedded-is-superior)
  - [Just-In-Time Funding (No Pre-Funding)](#just-in-time-funding-no-pre-funding)
  - [Bank-Embedded Architecture](#bank-embedded-architecture)
  - [Bank Customer Experience](#bank-customer-experience-what-the-end-user-sees)
  - [Bank-Embedded Payment Flow](#bank-embedded-end-to-end-payment-flow)
  - [Bank Licensing Revenue Model](#bank-licensing-revenue-model)
  - [Corridor Feasibility Matrix](#corridor-feasibility-matrix-gcc-bank-reality)
  - [Settlement Timing](#settlement-timing-onchain-vs-end-to-end-critical-nuance)
  - [Regulatory Positioning](#regulatory-positioning-who-owns-what)
  - [Additional Tech Stack for Bank-Embedded](#additional-tech-stack-for-bank-embedded-model)
  - [Strategic Advantages](#strategic-advantages)
  - [Risks & Mitigations](#risks--mitigations)
  - [Suggested Pilot Approach](#suggested-pilot-approach)
- [Getting Started (Pilot)](#getting-started-pilot)
- [Intellectual Property Protection](#intellectual-property-protection)
- [License](#license)

---

## Core Principle

> AiriPay is a **decision + evidence system**, not a settlement institution.
>
> - It **analyzes jurisdiction regulations**, applies enterprise policies, queries multiple payment rails for real-time quotes, ranks the top option by cost/speed/compliance, explains *why*, and initiates execution via the bank's own accounts (RTGS, Circle ARC, Buna, SWIFT).
> - It **never touches funds**, never holds keys, never has custody, and never bears settlement risk.
> - It **generates immutable audit trails** — ISO 20022 message mirrors + decision rationale + compliance evidence — for regulatory reporting.
>
> Think of it as a **flight search engine that doesn't own the planes, doesn't touch the tickets, and doesn't carry passengers** — but gives the airline real-time data on which routes to operate and why.

---

## System Architecture Diagram (GCC Multi-Rail Orchestration)

```
┌────────────────────────────────────────────────────────────────────┐
│                     GCC BANKING LAYER                              │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ QA-RTGS  │  │  SAMA    │  │ CBUAE    │  │ BNB-RTGS │  ...     │
│  │ (Qatar)  │  │ (KSA)    │  │ (UAE)    │  │(Bahrain) │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │             │              │             │                │
│       └─────────────┴──────────────┴─────────────┘                │
│                         │                                          │
│                         │ (Domestic settlement)                    │
└─────────────────────────┼──────────────────────────────────────────┘
                          │
                          │
       ┌──────────────────▼────────────────────────────────┐
       │      AIRIPAY V7 ORCHESTRATOR                      │
       │      (Rail-Agnostic Decision Engine)              │
       │   (Zero Custody / Zero Keys / Zero Settlement)    │
       │                                                   │
       │  ┌─────────────────────────────────────────┐     │
       │  │   Jurisdiction + Policy Analyzer        │     │
       │  │  • Where is recipient? → Select rails   │     │
       │  │  • Domestic? → Bank RTGS only           │     │
       │  │  • Intra-GCC? → Prefer AFAQ             │     │
       │  │  • Regional? → Prefer Buna              │     │
       │  │  • Global? → Arc/USDC + SWIFT fallback  │     │
       │  │  • Policy: AML/approval/corridors       │     │
       │  └─────────────────────────────────────────┘     │
       │                                                   │
       │  ┌─────────────────────────────────────────┐     │
       │  │     Rail Ranker (Cost/Speed/Compliance) │     │
       │  │  • Query AFAQ / Buna / Arc / SWIFT rate │     │
       │  │  • Score by corridor, policy, risk      │     │
       │  │  • Present top 3 + rationale            │     │
       │  └─────────────────────────────────────────┘     │
       │                                                   │
       │  ┌─────────────────────────────────────────┐     │
       │  │   Audit Trail (ISO 20022)               │     │
       │  │  • Rail selection decision               │     │
       │  │  • Compliance checks                     │     │
       │  │  • Immutable execution record            │     │
       │  └─────────────────────────────────────────┘     │
       └─────────┬──────┬──────┬──────┬─────────────────────┘
                 │      │      │      │
    ┌────────────┴─┐  ┌─┴──────┴──┐  ┌──────────┐  ┌───────────┐
    │   BANK RTGS  │  │ AFAQ/Buna │  │Arc/USDC  │  │  SWIFT/   │
    │  (Domestic)  │  │(Regional) │  │  (Fast)  │  │Corres.    │
    └──────────────┘  └───────────┘  └──────────┘  └───────────┘

KEY DIFFERENCE: AiriPay routes intelligently based on corridor.
All settlement accounts remain bank-owned. No funds touch AiriPay.
```

---

## Tech Stack & Third-Party Vendors

### Payment Settlement (What Moves the Money)

| Layer | Vendor | Purpose | AiriPay's Role | When to Use |
|---|---|---|---|---|
| **Domestic Settlement** | **Bank's RTGS** (QA-RTGS, SAMA, CBUAE, BNB-RTGS, CBO-RTGS, CBK-RTGS) | Real-time gross settlement for intra-country payments | Routes domestic payments to bank's RTGS; **does not touch funds** | Same-country payments (fastest, free) |
| **Intra-GCC Regional** | **[AFAQ](https://afaqltd.com/)** | GCC Interbank RTGS settlement model | Orchestrates via bank's AFAQ membership; **does not touch funds** | Qatar ↔ UAE ↔ KSA (preferred for regulator posture) |
| **Regional Multi-Currency** | **[Buna](https://www.buna.ae/)** | GCC multi-currency + PvP settlement | Queries Buna for rates; routes via bank's Buna account | GCC corridors where multi-currency improves reach |
| **Fast Global** | **[Arc/USDC Rails](https://docs.arc.network)** | USDC-based settlement with ~$0.01 target fee (sub-second onchain finality) | Routes via bank's Circle ARC business account; **does not touch funds** | Global corridors with payout programs supporting near-real-time delivery |
| **Universal Fallback** | **Bank's SWIFT + Correspondent Network** | Traditional correspondentbanking | Bank executes; AiriPay logs audit only | Unsupported corridors or when Arc payout unavailable |

### Key & Wallet Custody (Bank Maintains Control — Not AiriPay)

| Layer | Recommended Vendor | Alternative | Purpose |
|---|---|---|---|
| **Circle ARC Account** | **Circle** | — | Bank maintains their own Circle ARC business account for cross-border settlement |
| **Optional: MPC Wallet** | **[Fireblocks](https://www.fireblocks.com/)** | Anchorage Digital, Copper.co | For banks that want institutional-grade MPC custody of USDC wallets (AiriPay has **read-only** access) |

### Identity, KYC & Compliance (Handled by Payment Providers, Referenced by AiriPay)

| Layer | Recommended Vendor | Alternative Vendors | Purpose |
|---|---|---|---|
| **KYC / KYB** | **[Sumsub](https://sumsub.com/)** | Onfido, Jumio, Persona | Enterprise and beneficiary identity verification. Called by Circle/Fireblocks during their onboarding — AiriPay stores only pass/fail status for audit. |
| **Sanctions & AML Screening** | **[ComplyAdvantage](https://complyadvantage.com/)** | Chainalysis (on-chain), Dow Jones Risk & Compliance, Refinitiv (LSEG) | AiriPay queries sanctions/PEP lists to inform rail ranking. Actual enforcement is by the payment provider. |
| **Transaction Monitoring** | **[Chainalysis KYT](https://www.chainalysis.com/kyt/)** | Elliptic, TRM Labs | On-chain transaction monitoring for USDC flows (provider-side). AiriPay references risk scores in audit trail. |

### Application Infrastructure

| Layer | Recommended Vendor | Alternative Vendors | Purpose |
|---|---|---|---|
| **Frontend** | **React + Vite** (existing) | Next.js | Enterprise dashboard — payment initiation, option ranking UI, transaction history, audit reports. |
| **Backend / API** | **Node.js + Fastify** | Express.js, NestJS | Orchestration server — jurisdiction engine, policy engine, rail queries, audit logging, webhook handling. |
| **Database** | **[Supabase](https://supabase.com/)** (PostgreSQL) | AWS RDS, PlanetScale, Neon | Enterprise profiles, transaction mirrors, audit trail, policy configurations. |
| **Immutable Audit Log** | **[Amazon QLDB](https://aws.amazon.com/qldb/)** | Hyperledger Fabric (self-hosted), Immudb | Tamper-proof, cryptographically verifiable audit trail — AiriPay's core compliance deliverable. |
| **Enterprise Auth & SSO** | **[Auth0](https://auth0.com/)** | Supabase Auth, Clerk, WorkOS | Enterprise SSO (SAML/OIDC), RBAC, MFA, approval workflows per enterprise. |
| **Subscription Billing** | **[Stripe Billing](https://stripe.com/billing)** | Chargebee, Paddle | Tier-based subscription management for AiriPay's own revenue. |
| **Email Notifications** | **[SendGrid](https://sendgrid.com/)** | AWS SES, Postmark, Resend | Transaction status notifications, compliance alerts, approval requests. |
| **Real-Time Updates** | **[Pusher](https://pusher.com/)** or **Supabase Realtime** | Ably, Socket.io | Live dashboard updates as transaction status changes. |
| **Hosting / Infra** | **[Vercel](https://vercel.com/)** (frontend) + **[Railway](https://railway.app/)** or **AWS** (backend) | Render, Fly.io, GCP Cloud Run | Application deployment and scaling. |
| **Secrets & Key Vault** | **[AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)** | HashiCorp Vault, Doppler | Encrypt and store enterprise API keys for Circle, Fireblocks, Wise, etc. |
| **Observability & Logging** | **[Datadog](https://www.datadoghq.com/)** | Grafana Cloud, New Relic, Sentry | API monitoring, error tracking, performance dashboards. |

### AI / Agentic Layer

| Layer | Recommended Vendor | Alternative Vendors | Purpose |
|---|---|---|---|
| **LLM for Decision Rationale** | **[OpenAI GPT-4](https://platform.openai.com/)** or **[Azure OpenAI](https://azure.microsoft.com/en-us/products/ai-services/openai-service)** | Anthropic Claude, Google Gemini | Generate human-readable explanations for why a specific rail was ranked #1 (audit trail enrichment). |
| **Agent Framework** | **[LangChain](https://www.langchain.com/)** or **[Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/)** | CrewAI, AutoGen | Orchestrate multi-step agentic workflows: analyze jurisdiction → check policies → query rails → rank → present. |
| **Regulatory Knowledge Base** | **[Pinecone](https://www.pinecone.io/)** (vector DB) + RAG | Weaviate, Chroma, Qdrant | Embed and retrieve jurisdiction-specific regulations (CBUAE, SAMA, MAS, etc.) for real-time compliance checking. |

---

## GCC Regional RTGS Integration Strategy

### Positioning: Cross-Border Overlay, Not Competitor

AiriPay **complements** GCC banks' existing domestic settlement infrastructure rather than competing with it:

| Payment Type | Routing | Settlement System | AiriPay's Role |
|---|---|---|---|
| **Domestic (e.g., QNB → Doha Bank)** | Bank's RTGS | QA-RTGS (seconds) | Routes to RTGS; logs audit trail |
| **Cross-Border (e.g., Qatar → UAE)** | AiriPay + Circle ARC | USDC settlement (1-2 minutes) | Orchestrates Circle ARC |
| **Cross-Border (unsupported corridor)** | Bank's correspondent | SWIFT/bilateral (1-3 days) | **Informational only** — bank handles |

### GCC RTGS Systems Supported

| Country | RTGS System | Status | Integration Method |
|---|---|---|---|
| 🇶🇦 Qatar | **QA-RTGS** | ISO 20022 (2024) | Via member bank's API |
| 🇸🇦 Saudi Arabia | **SAMA RTGS (SARIE)** | ISO 20022 | Via member bank's API |
| 🇦🇪 UAE | **CBUAE RTGS (UAEFTS)** | ISO 20022 | Via member bank's API |
| 🇧🇭 Bahrain | **BNB-RTGS** | ISO 20022 | Via member bank's API |
| 🇴🇲 Oman | **CBO-RTGS** | ISO 20022 | Via member bank's API |
| 🇰🇼 Kuwait | **CBK-RTGS** | ISO 20022 | Via member bank's API |

### Use Case Examples (Multi-Rail Optimization)

#### **Use Case 1: Domestic Payment** (Same Country)
```
✅ Customer: QNB customer sends 10,000 QAR to Doha Bank
✅ AiriPay decision: Same country
✅ Routing: QA-RTGS only (instant, free)
✅ Settlement: 5-30 seconds
✅ AiriPay role: Routes to RTGS, logs audit trail; does NOT touch funds
```

#### **Use Case 2: Intra-GCC (Preferred: AFAQ)** (Qatar → UAE)
```
✅ Customer: QNB customer sends 50,000 QAR to Emirates NBD, Dubai
✅ AiriPay decision: Intra-GCC
✅ Option 1: AFAQ (preferred for regulatory posture) — ~10 min, 0.15% fee ← TOP
✅ Option 2: Arc/USDC rail (near-real-time, ~$0.01 flat) — ~15 min crediting ← FAST
✅ Option 3: SWIFT gpi (fallback) — 2-4 hours, 0.5% fee
✅ Customer selects AFAQ (bank's preferred corridor)
✅ Settlement: AFAQ RTGS settlement via GCC interbank network
✅ AiriPay role: Routes via AFAQ, logs decision rationale + audit
```

#### **Use Case 3: GCC → Global** (Qatar → USA)
```
✅ Customer: QNB customer sends 50,000 QAR → 13,340 USD to New York
✅ AiriPay decision: Global corridor
✅ Option 1: Arc/USDC rail — receives USD in ~15 min (sub-second onchain finality + bank delivery) ← TOP
✅ Option 2: SWIFT gpi — receives USD in 1-2 days, 0.5% fee
✅ Customer selects Arc/USDC
✅ Settlement: Sub-second onchain finality; USD payout to recipient bank via Circle's US partners
✅ AiriPay role: Queries Arc for quote, ranks, orchestrates via bank's Circle ARC account
✅ Key point: Receipient crediting is ~15 min (onchain finality is sub-second, but bank delivery adds ~15 min)
```

#### **Use Case 4: Regional Multi-Currency** (GCC ↔ GCC, multi-currency)
```
✅ Customer: ENBD (UAE) customer sends 100,000 AED to a buyer in KSA wanting payment in USD
✅ AiriPay decision: Regional, multi-currency benefit
✅ Option 1: Buna (regional PvP settlement) — ~2 hours, reduces settlement risk ← PREFERRED
✅ Option 2: Arc/USDC — ~15 min, if payout availability ✅
✅ Option 3: SWIFT gpi — 1-3 days
✅ Customer selects Buna
✅ Settlement: AED → USD via Buna's multi-currency PvP model
✅ AiriPay role: Queries Buna, ranks, orchestrates
```

#### **Use Case 5: Unsupported Corridor** (GCC → Pakistan, when Arc unavailable)
```
✅ Customer: QNB customer sends 5,000 QAR → PKR to Karachi
✅ AiriPay decision: Global, but Arc payout unavailable in Pakistan
✅ Option 1: SWIFT (bank's correspondent) — 2-5 days, 0.5% fee ← ONLY OPTION
✅ AiriPay displays: "This corridor is available through your bank's network (standard timing)"
✅ AiriPay role: Informational; bank executes via SWIFT; AiriPay logs audit only
```

### Future: GCC Interbank Consortium

**Vision**: AiriPay as orchestrator layer above all GCC RTGS systems

```
┌──────────────────────────────────────────────────────────────┐
│               GCC INTERBANK CONSORTIUM                       │
│          (Federated RTGS Settlement Network)                 │
│                                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │ QA-RTGS  │  │  SAMA    │  │ CBUAE    │  │ BNB-RTGS │   │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│        │             │              │             │          │
│        └─────────────┴──────────────┴─────────────┘          │
│                          │                                   │
│                          │                                   │
│              ┌───────────▼───────────┐                       │
│              │   AIRIPAY ORCHESTRATOR │                       │
│              │   (Cross-Border Layer) │                       │
│              └───────────┬───────────┘                       │
│                          │                                   │
│                          │                                   │
│                   ┌──────▼──────┐                            │
│                   │  Circle ARC │                            │
│                   │  (Global)   │                            │
│                   └─────────────┘                            │
└──────────────────────────────────────────────────────────────┘

Benefits:
• Domestic settlement: Real-time via local RTGS
• Intra-GCC: Real-time via federated ISO 20022 messaging
• GCC ↔ Global: Instant via Circle ARC stablecoin rails
• Fallback: Each bank's correspondent network (informational)
```

### Technical Integration Requirements

For AiriPay to integrate with a GCC bank:

| Requirement | Details |
|---|---|
| **Bank's RTGS connectivity** | Bank must be direct participant in their national RTGS |
| **Circle ARC business account** | Bank maintains their own Circle ARC account for cross-border |
| **API exposure** | Bank exposes read-only API to RTGS account status (for audit) |
| **ISO 20022 messaging** | AiriPay logs pacs.008, pacs.002, pacs.004 messages |
| **Policy engine integration** | Bank configures AML tiers, approval workflows, dual-authorization rules |
| **Optional: Fireblocks** | For banks wanting MPC custody of USDC wallets |

---

## AiriPay's Value Proposition for GCC Banks

### Problem Banks Face Today

| Challenge | Impact |
|---|---|
| **Slow cross-border settlement** | 1-3 days via SWIFT; customers frustrated |
| **High correspondent fees** | 2-3% + intermediary charges; uncompetitive for corporates |
| **Limited corridor coverage** | Emerging markets require multiple correspondent relationships |
| **Opaque pricing** | FX spreads hidden; customers don't know true cost until after settlement |
| **No instant cross-border RTGS** | Domestic RTGS is real-time, but ends at the border |

### How AiriPay Solves This

| Solution | Benefit for Bank | Benefit for Customer |
|---|---|---|
| **Circle ARC integration** | Instant cross-border settlement (1-2 minutes) | Same-day receipt of funds |
| **Transparent pricing** | Clear 0.5-1% fee vs. 2-3% SWIFT | Lower total cost |
| **USDC rails** | Leverage stablecoin infrastructure without crypto operations | Recipients receive local currency (don't see crypto) |
| **Audit trail** | ISO 20022 message logging for central bank compliance | Clear transaction history |
| **Policy engine** | Configurable approval workflows per corporate | Meets treasury policy requirements |
| **Zero pre-funding** | Just-in-time settlement from customer's bank account | No idle capital |

### Competitive Positioning

| | **Traditional SWIFT** | **Wise/Revolut (Fintech)** | **AiriPay (Multi-Rail)** |
|---|---|---|---|
| **Settlement Speed** | 1-3 days | 2-24 hours | Domestic: instant (RTGS) / Intra-GCC: ~10 min (AFAQ) / Global: ~15 min (Arc+delivery) |
| **Cost** | 2-3% + intermediary | 0.5-1.5% | Domestic: free / Intra-GCC: 0.15-0.5% / Global: 0.05-0.5% |
| **Finality** | SWIFT: 1-3 days | Fintech: real-time (proprietary network) | **Domestic RTGS**: instant / **Arc onchain**: sub-second / **Delivery**: near-real-time |
| **Transparency** | Opaque (hidden spreads) | Transparent rates | **Transparent** (multi-option ranking) |
| **Integration** | Bank's existing system | Separate platform (customer leaves) | **White-label inside bank's app** |
| **Custody** | Bank holds funds | Fintech holds funds (trust issue) | **Bank holds funds (never leaves)** |
| **Regulatory** | Bank's license | Fintech's license (varies) | **Bank's license** (zero AiriPay regulatory burden) |
| **Rail Flexibility** | SWIFT only | Fintech proprietary | **Multi-rail** (AFAQ/Buna/Arc/SWIFT) |
| **Regulator Preference** | ✅ Known | ⚠️ Varies by jurisdiction | ✅ **AFAQ/Buna preferred for GCC** |

---

### Cost Savings Examples (Customer View)

#### **Example 1: Qatar → UAE (50,000 QAR) — Intra-GCC**

| Method | Settlement Time | Total Cost | Comment |
|---|---|---|---|
| **🚀 AiriPay (AFAQ)** | **~10 min** | **75 QAR (0.15%)** | **Bank's preferred corridor** |
| **🚀 AiriPay (Arc/USDC)** | **~15 min (Bank delivery)** | **25-75 QAR (~0.05% + $0.01 flat)** | **Sub-second finality, near-real-time** |
| 🏦 Traditional Wire/SWIFT | 2-4 hours | 1,250 QAR (2.5%) | ← Customer's old alternative |
| 💳 Wise Business | 1-2 hours | 750 QAR (1.5%) | ← Why customers leave |
| 💳 Revolut Business | 1-2 hours | 850 QAR (1.7%) | ← Why customers leave |

**Customer Impact**: Saves **1,000 QAR per transaction** vs. traditional banking  
**Annual Impact** (100 tx/mo): **1.2M QAR saved/year (~$330K USD)**

#### **Example 2: Qatar → India (20,000 QAR) — Global Corridor**

| Method | Settlement Time | Total Cost | Comment |
|---|---|---|---|
| **🚀 AiriPay (Arc/USDC)** | **~15 min** | **100 QAR (~0.5%)** | **Sub-second finality + INR payout** |
| **🚀 AiriPay (SWIFT gpi fallback)** | **2-5 days** | **500 QAR (2.5%)** | **If Arc unavailable** |
| 💳 Wise Business | 1-2 days | 300 QAR (1.5%) | ← Customer's current fintech |
| 🏦 Bank Wire/SWIFT | 2-5 days | 500 QAR (2.5%) + intermediary | ← Why customer switched |

---

### Bank Business Case: Why License AiriPay

#### **The Fintech Threat**

**Market Reality**: Banks are losing 25-40% of cross-border payment volume to Wise/Revolut

```
Typical Customer Journey (Without AiriPay):
1. Corporate needs frequent international transfers
2. Bank quotes 2.5% + 2-day SWIFT settlement
3. Finance manager opens Wise account (1.5% + same day)
4. ALL cross-border moves to Wise
5. Eventually moves full banking relationship
6. Bank loses customer
```

**GCC Banks' Cross-Border Revenue Leakage**:
- Tier 1 Bank: Losing $50M-$150M/year to fintechs
- Tier 2 Bank: Losing $15M-$50M/year to fintechs
- Tier 3 Bank: Losing $5M-$20M/year to fintechs

---

#### **AiriPay's Solution: Offer "Instant Cross-Border" Free to Customers**

**Business Model**:

| Revenue Stream | Who Pays | Amount | Customer Sees |
|---|---|---|---|
| **License Fee** | Bank pays AiriPay | $100K-$500K/year | Nothing |
| **Volume Fee** | Bank pays AiriPay | 0.02% of volume | Nothing |
| **Customer Fee** | Customer pays bank | **$0 or 0.5%** (bank decides) | "Free" or "Instant Transfer" |

**Example ROI** (QNB with $15B cross-border volume):

```
QNB's Cost:
- AiriPay license: $500K/year
- AiriPay volume fee (0.02%): $3M/year
- Total AiriPay cost: $3.5M/year

QNB's Benefits:
- Prevents revenue leakage: $56M/year retained
- Customer retention: 1,000-5,000 corporates stay
- New customer acquisition: 500-2,000 corporates switch to QNB
- Brand differentiation: "First GCC bank with instant cross-border"

Net Benefit: $52.5M/year
ROI: 1,400%
Payback: 2-4 weeks
```

---

#### **Bank Can Offer Service FREE and Still Profit**

**Traditional Wire Economics** (What Banks Lose Today):
```
Transaction: 50,000 QAR → UAE
Customer pays: 1,250 QAR (2.5%)
Bank costs: 750 QAR (correspondent + SWIFT + processing)
Bank profit: 500 QAR

Problem: Customer leaves to Wise → Bank earns $0
```

**AiriPay Economics** (What Banks Keep):
```
Transaction: 50,000 QAR → UAE
Customer pays: 0 QAR ("FREE instant transfer promotion")
Bank costs: 230 QAR (Circle + AiriPay + processing)
Bank profit: -230 QAR (small loss per transaction)

BUT: Customer stays with bank → Bank retains:
- $500K deposit balance (earns $10K/year interest margin)
- $2M credit facility ($20K/year fees)
- 50 cross-border tx/year (relationship depth)
- Customer lifetime value: $275K over 5 years

Result: -$230 loss per tx << $275K lifetime value
```

**Break-Even**: Bank needs to retain just **13-64 customers** to pay for AiriPay license  
**Typical Bank**: Has **1,000-10,000 corporate customers** at risk of leaving to fintechs

---

#### **Three Customer Offer Strategies**

| Strategy | Customer Pays | Bank Positioning | Result |
|---|---|---|---|
| **"Free Instant Cross-Border"** | 0% (bank subsidizes) | Premium service, no fees | Massive adoption, Wise migration stops |
| **"Instant for 0.5%"** | 0.5% (vs. 2.5% standard) | Speed tier option | 70-80% choose instant, bank earns 0.5% |
| **"Premium Banking Perk"** | 0% for $250K+ balance | VIP relationship benefit | Deepens relationships, prevents defection |

**Recommended Year 1**: Strategy #1 (market share grab), then transition to #3 (monetize relationships)

---

#### **Competitive Advantage Matrix**

|  | **Bank w/o AiriPay** | **Wise/Revolut** | **Bank w/ AiriPay** |
|---|---|---|---|
| Speed | ❌ 1-3 days | ✅ 2-4 hours | ✅✅ **1-2 minutes (FASTEST)** |
| Cost | ❌ 2-3% | ✅ 1.5% | ✅✅ **0.5% (CHEAPEST)** |
| Trust | ✅ High | ⚠️ Medium | ✅✅ **High** |
| Funding | ✅ Bank account | ❌ Separate balance | ✅✅ **Bank account** |
| Customer stays | ⚠️ Unless they find Wise | ❌ **Leaves bank** | ✅✅ **Stays with bank** |

**Result**: AiriPay-powered banks **beat fintechs on speed, cost, AND convenience**

---

##End-to-End Payment Flow (GCC Bank-Embedded Model)

### Phase 0 — Bank Onboarding (One-Time)

| Step | Actor | Action | AiriPay's Role |
|---|---|---|---|
| **0.1** | Bank | Signs AiriPay white-label licensing agreement | Provides bank with white-label SDK for integration into digital banking platform |
| **0.2** | Bank | Connects **Circle ARC** business account (API key) | AiriPay uses bank's Circle API key to orchestrate cross-border payments on bank's behalf - funds flow directly between Circle and bank |
| **0.3** | Bank (optional) | Connects **Fireblocks vault** (API key) for MPC custody | AiriPay gets **read-only** access to see USDC wallet balances/status |
| **0.4** | Bank | Configures **RTGS integration** | Bank provides read-only API access to domestic RTGS account for audit trail logging |
| **0.5** | Bank | Sets default payment policies | E.g., "Max settlement 4 hours," "Require dual approval for >$100K," "Blocked countries list" |
| **0.6** | Bank | Deploys AiriPay module to production | Embedded in bank's mobile app and web banking under "International Payments" |

> **Critical**: Bank's Circle account, bank's RTGS account, bank's customer funds — all remain with the bank. AiriPay is an **orchestration and audit layer only.**

---

### Phase 1 — Customer Initiates a Payment (Bank's Digital Platform)

| Step | Actor | Action | Detail |
|---|---|---|---|
| **1.1** | Bank Customer | Opens bank's app → "International Payments" | (AiriPay-powered, but white-labeled as bank's feature) |
| **1.2** | Customer | Enters payment details | Recipient name, recipient bank/IBAN, country, amount in recipient currency, purpose code |
| **1.3** | AiriPay Orchestrator | **Routing Decision** | Detects if domestic (same country) or cross-border |
| **1.4a** | **If Domestic** | Route to bank's RTGS | Payment settles via QA-RTGS/SAMA/CBUAE; AiriPay logs audit trail only |
| **1.4b** | **If Cross-Border** | Compliance screening + Circle ARC quote | Checks sanctions/PEP lists; queries Circle ARC for real-time quote |

---

### Phase 2 —Payment Options Presentation (Cross-Border Only)

| Step | Actor | Action | Detail |
|---|---|---|---|
| **2.1** | AiriPay Decision Engine | **Checks Circle ARC availability** | Determines if corridor is supported by Circle ARC |
| **2.2a** | **If Circle ARC Available** | Present Circle ARC option | Shows: FX rate, fees, settlement time (~1-2 minutes) |
| **2.2b** | **If Circle ARC NOT Available** | Display informational message | "This corridor is available through your bank's existing correspondent network (1-3 days, traditional fees)" |
| **2.3** | AiriPay Audit Trail | **Logs the decision** | Records: routing decision, Circle ARC availability, compliance results, timestamp, customer ID |

**Example: Payment Options UI (Embedded in Bank's App)**

```
┌─────────────────────────────────────────────────────────────────────┐
│              QNB International Payments  (Powered by AiriPay)       │
│  Sending: 50,000 QAR to Emirates NBD, Dubai, UAE                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ★ INSTANT TRANSFER (Recommended)                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Circle ARC Settlement                                       │    │
│  │    Recipient receives: 48,750 AED                          │    │
│  │    Fee: 250 QAR (0.5%)                                     │    │
│  │    Exchange rate: 1 QAR = 0.975 AED                       │    │
│  │    Settlement: ~2 minutes                                   │    │
│  │    ✅ Real-time settlement via USDC rails                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ℹ️  Traditional Wire Transfer Available                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Standard International Transfer                             │    │
│  │    Settlement: 1-3 business days                           │    │
│  │    Fees: 2-3% + intermediary charges                       │    │
│  │    (Processed through QNB's correspondent network)         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Phase 3 — Customer Selects Option → Bank Executes via Circle ARC

| Step | Actor | Action | Detail |
|---|---|---|---|
| **3.1** | Customer | Clicks "Send via Instant Transfer" | Authorizes payment via bank's auth (biometric, 2FA, OTP) |
| **3.2** | Bank's System | **Debits customer account** | 50,250 QAR debited from customer's checking account |
| **3.3** | AiriPay Backend | **Pings Circle ARC API** using **bank's Circle API credentials** | Instructs Circle to execute transfer; passes: amount, destination, recipient bankdetails |
| **3.4** | Circle ARC | **Executes settlement** | USDC settlement → FX conversion (QAR→AED) → fiat payout to recipient bank in Dubai |
| **3.5** | AiriPay Backend | **Polls Circle for status** | Updates: `pending` → `in_progress` → `complete`; real-time updates via bank's app |
| **3.6** | AiriPay Audit Trail | **Logs ISO 20022 pacs.008 + Circle confirmation** | Immutable audit record with Circle transaction ID, timestamps, final rates |

---

### Phase 4 — Recipient Receives Funds

| Step | Actor | Action | Detail |
|---|---|---|---|
| **4.1** | Circle + UAE Banking Partner | Wire 48,750 AED to Emirates NBD | Via UAE's UAEFTS (local payment rails) |
| **4.2** | Recipient | Sees deposit in Emirates NBD account | Receives standard bank deposit; **no indication of crypto/USDC** |
| **4.3** | Bank's App | Notifies customer (QNB) | "International transfer completed in 2 minutes. 48,750 AED delivered to [Recipient]." |
| **4.4** | AiriPay | Updates audit trail | Final status, actual settlement time, Circle confirmation, compliance attestation |

**Key Customer Experience Points**:
- ✅ Customer **never leaves bank's app**
- ✅ Customer **never sees "crypto" or "USDC"** — completely abstracted
- ✅ Funds debit from normal checking account in real-time
- ✅ Settlement happens in minutes instead of days
- ✅ Transparent pricing (no hidden FX spreads)

---

## What Happens Inside Circle ARC

> AiriPay does not manage any of this — Circle handles it end to end.

```
Enterprise's Circle Account (pre-funded with USD or USDC)
    │
    ▼
Circle debits enterprise's account
    │
    ▼
Circle mints/moves USDC on-chain (internal)
    │
    ▼
Circle redeems USDC → USD → SAR (FX)
    │
    ▼
Circle's banking partner wires SAR to recipient bank
    │
    ▼
Recipient receives SAR ✓
```

**AiriPay's visibility**: Read-only status updates via API. AiriPay never signs transactions, never holds USDC, never touches fiat.

---

## Where Fireblocks Fits

If the enterprise uses **Fireblocks** for wallet/key management instead of Circle's built-in custody:

```
Enterprise's Fireblocks Vault
  └── Holds USDC in enterprise's wallet
  └── Fireblocks manages private keys (MPC custody)
  └── Fireblocks policy engine: spending limits, whitelist

AiriPay: "Execute this payment via Circle ARC"
    │
    ▼
Fireblocks: Signs the USDC transfer to Circle's settlement address
    │
    ▼
Circle ARC: Receives USDC → converts → wires fiat to recipient
```

**AiriPay's role**: Tells Fireblocks *what* to do (via API, using enterprise's Fireblocks API key). Fireblocks decides *if* it's allowed (its own policy engine) and *signs* the transaction. **AiriPay never sees the private key.**

---

## AiriPay's Data Model

| Data | Purpose | Sensitivity |
|---|---|---|
| Enterprise profiles | Subscription tier, policies, connected APIs | Medium |
| User roles & permissions | RBAC / approval workflows | Medium |
| **Payment decisions (audit trail)** | Every option ranked, rationale, scores, policies applied | **Core IP** |
| Transaction status (mirror) | Cached status from Circle/Wise/etc. | Low (read-only copy) |
| Compliance screening results | Sanctions/PEP check pass/fail | High |
| **Never stored**: Private keys, wallet seeds, user funds, bank credentials | — | — |

---

## Subscription Tiers (Deployment Model A — Direct-to-Enterprise)

AiriPay charges **zero per-transaction fees.** Revenue is purely subscription-based.

| Tier | Monthly Fee | Included Transactions | Features |
|---|---|---|---|
| **Starter** | $1,298/mo | Up to 1,000 tx/mo | 3-option ranking, basic audit trail, email support |
| **Growth** | $2,798/mo | Up to 10,000 tx/mo | + custom policies, approval workflows, API access, priority support |
| **Enterprise** | $10,498/mo | Unlimited | + dedicated account manager, custom compliance rules, SLA guarantees, SSO/SAML |

> For bank licensing fees, see [Deployment Model B — Bank Licensing Revenue Model](#bank-licensing-revenue-model).

---

## What AiriPay Does vs. What It Does Not

| ✅ AiriPay DOES | ❌ AiriPay DOES NOT |
|---|---|
| Analyze jurisdiction regulations | Hold or custody any funds |
| Apply enterprise payment policies | Manage private keys or wallets |
| Query multiple payment rails for quotes | Process payments directly |
| Rank top 3 options by cost/speed/compliance | Perform KYC (payment provider does this) |
| Present options to enterprise user | Touch USDC, fiat, or any money |
| Ping selected provider's API to initiate | Sign blockchain transactions |
| Monitor transaction status (read-only) | Bear settlement risk |
| Generate immutable audit trail | Charge per-transaction fees |
| Provide compliance evidence & reports | Act as a money transmitter |

---

## Deployment Model B — AiriPay for Banks (White-Label)

> **The big idea**: Instead of selling to enterprises one by one, **license AiriPay to banks** as a white-label module embedded in their digital banking platform. One bank deal = thousands of corporate customers onboarded instantly. Funds stay in the customer's bank account until the exact moment of payment — **zero pre-funding required.**

---

### Why Bank-Embedded Is Superior

| Dimension | Model A: Direct-to-Enterprise | Model B: Bank-Embedded |
|---|---|---|
| **Sales cycle** | Long — sell to each enterprise individually | Shorter — one bank deal = thousands of enterprises onboarded instantly |
| **Trust barrier** | High — enterprises must trust a startup with API keys to their Circle/Fireblocks accounts | Low — enterprises already trust their bank. AiriPay is invisible. |
| **Funds flow** | Enterprise must pre-fund Circle wallet | Funds stay in bank account until payment. Just-in-time settlement. |
| **Customer acquisition cost** | High (enterprise sales team) | Low (bank distributes for you) |
| **Revenue per deal** | $1K–$10K/mo per enterprise | $50K–$500K+/mo per bank license |
| **Compliance burden on AiriPay** | Must navigate regulations directly | Bank is already regulated. AiriPay operates under the bank's license. |
| **KYC/AML** | AiriPay must orchestrate or rely on providers | Bank already has full KYC on every customer. Zero additional work. |
| **Scalability** | Linear — one enterprise at a time | Exponential — one bank = 10K+ corporate customers |

---

### Just-In-Time Funding (No Pre-Funding)

This solves the biggest friction in the direct-to-enterprise model: **enterprises don't want idle funds sitting in Circle wallets.**

```
MODEL A (Direct-to-Enterprise) — THE PROBLEM:
  Enterprise pre-funds Circle wallet with $500K USDC
  → Money sits idle until payments are made
  → Opportunity cost, treasury management burden
  → CFOs hate this

MODEL B (Bank-Embedded) — THE SOLUTION:
  Customer's funds sit in their normal bank account (earning interest, fully liquid)
  → Customer initiates payment via bank's app (AiriPay-powered)
  → Customer selects option (e.g., Circle ARC)
  → Customer authorizes via bank's existing auth (biometric, 2FA)
  → Bank debits customer's account at that moment
  → Bank's treasury/Circle integration moves exact amount to Circle
  → Circle settles → Recipient gets paid
  → Zero idle funds in crypto wallets ever
```

**Payment timeline:**

```
T+0s     Customer clicks "Send 50,000 SAR" in bank app
T+1s     AiriPay engine returns 3 ranked options
T+5s     Customer selects Circle ARC, authorizes via Face ID
T+6s     Bank debits customer account (AED 48,500)
T+8s     Bank's system sends USD/USDC to Circle via API
T+10s    Circle initiates cross-border settlement
T+15min  Recipient receives 50,000 SAR in their bank
         ───────────────────────────────────────────
         Customer's funds were "in transit" for ~15 minutes.
         They never saw the word "crypto" or "USDC".
```

---

### Bank-Embedded Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    BANK'S DIGITAL PLATFORM                   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            AIRIPAY ENGINE (White-Labeled)             │   │
│  │                                                       │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │   │
│  │  │ Jurisdiction │  │ Bank Policy  │  │ Rail       │  │   │
│  │  │ Analyzer     │  │ Engine       │  │ Ranker     │  │   │
│  │  │              │  │ (per-bank    │  │            │  │   │
│  │  │              │  │  configured) │  │            │  │   │
│  │  └─────────────┘  └──────────────┘  └────────────┘  │   │
│  │                                                       │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │              Audit Trail Engine                  │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                              │                               │
│  ┌──────────────────────────┴───────────────────────────┐   │
│  │              BANK'S CORE BANKING SYSTEM               │   │
│  │  • Customer account debit (just-in-time)              │   │
│  │  • Bank's own Circle/Fireblocks treasury account      │   │
│  │  • Bank's existing KYC (no duplication)               │   │
│  │  • Bank's auth (biometric, 2FA, token)                │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                              │                               │
└──────────────────────────────┼───────────────────────────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │    PAYMENT RAILS           │
        ┌────────┼────────┬──────────┬───────┤
        ▼        ▼        ▼          ▼       ▼
   ┌────────┐ ┌──────┐ ┌───────┐ ┌──────┐ ┌──────┐
   │Circle  │ │ Wise │ │Thunes │ │SWIFT │ │Bank's│
   │ARC     │ │      │ │       │ │gpi   │ │own   │
   │        │ │      │ │       │ │      │ │rails │
   └────────┘ └──────┘ └───────┘ └──────┘ └──────┘

   KEY DIFFERENCE: The BANK owns the Circle/Fireblocks account.
   The BANK holds the custody. AiriPay still touches nothing.
```

**Key architectural differences from Model A:**

1. **AiriPay deploys as a module inside the bank's infrastructure** (on-premise or private cloud) — not as a standalone SaaS the customer logs into separately.
2. **Bank's core banking system handles the debit** — AiriPay tells it "debit $X from account Y and send to Circle endpoint Z."
3. **Bank's own Circle/Fireblocks treasury account** is used — not the customer's personal crypto wallet. The bank aggregates.
4. **Bank's existing auth** is used — customer is already logged into their banking app.
5. **The bank can add its own FX markup** on top of the rates AiriPay surfaces — extra revenue source for the bank.
6. **AiriPay's audit trail feeds into the bank's regulatory reporting** system.

---

### Bank Customer Experience (What the End User Sees)

The customer **never sees "AiriPay," "Circle," "USDC," or "crypto"**. From their perspective, it's a native bank feature:

```
┌─────────────────────────────────────────────┐
│  🏦  EMIRATES NBD  —  International Transfer │
│                                              │
│  From: Current Account ****4521              │
│  Balance: AED 2,340,000.00                   │
│                                              │
│  To: Saudi Trading Co.                       │
│  Bank: Al Rajhi Bank                         │
│  IBAN: SA44 2000 0001 2345 6789 0123         │
│  Amount: SAR 50,000.00                       │
│                                              │
│  ─────────────────────────────────────────── │
│                                              │
│  ★ Best Option                               │
│  ┌─────────────────────────────────────────┐ │
│  │ Instant Transfer                        │ │
│  │ Rate: 1 AED = 1.0197 SAR               │ │
│  │ You pay: AED 49,034.00                  │ │
│  │ Arrives: ~15 minutes                    │ │
│  │ Fee: AED 45                             │ │
│  │                            [Select ✓]   │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │ Standard Transfer                       │ │
│  │ Rate: 1 AED = 1.0185 SAR               │ │
│  │ You pay: AED 49,092.00                  │ │
│  │ Arrives: ~2–4 hours                     │ │
│  │ Fee: AED 25                             │ │
│  │                            [Select]     │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │ Regular Wire                            │ │
│  │ Rate: 1 AED = 1.0150 SAR               │ │
│  │ You pay: AED 49,261.00                  │ │
│  │ Arrives: 1–2 business days              │ │
│  │ Fee: AED 75                             │ │
│  │                            [Select]     │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  [Authorize with Face ID →]                  │
└─────────────────────────────────────────────┘
```

The customer sees "Instant Transfer" — not "Circle ARC USDC stablecoin settlement." **Crypto is the invisible infrastructure**, like TCP/IP is invisible when you browse the web.

---

### Bank-Embedded End-to-End Payment Flow

#### Phase B0 — Bank Onboarding (One-Time)

| Step | Actor | Action | Detail |
|---|---|---|---|
| **B0.1** | Bank | Signs AiriPay license agreement | Annual license fee based on bank tier. AiriPay deploys engine into bank's infrastructure. |
| **B0.2** | Bank + AiriPay | Configure payment rails | Activate the rails the bank wants to offer: Circle ARC, Wise, SWIFT gpi, and/or the bank's own domestic rails. |
| **B0.3** | Bank | Connects their **Circle ARC treasury account** | Bank's own Circle account. Bank funds it from their treasury. Customers never interact with it directly. |
| **B0.4** | Bank | Connects their **Fireblocks vault** (if used) | Bank's institutional Fireblocks account for USDC custody. |
| **B0.5** | Bank + AiriPay | Configure bank-specific policies | Corridors offered, FX markup rules, customer eligibility tiers, approval thresholds, regulatory constraints. |
| **B0.6** | Bank | Integrates AiriPay API into their digital banking platform | AiriPay exposes a REST/gRPC API; bank's frontend calls it. White-labeled — no AiriPay branding visible. |

#### Phase B1 — Customer Initiates Payment

| Step | Actor | Action | Detail |
|---|---|---|---|
| **B1.1** | Bank Customer | Logs into bank's app/portal (already authenticated) | Standard bank auth — biometric, 2FA, hardware token. No separate AiriPay login. |
| **B1.2** | Bank Customer | Opens "International Transfer" | Enters: recipient, bank, IBAN, country, amount, purpose. |
| **B1.3** | Bank's Backend | Calls AiriPay API: `POST /api/v1/quotes` | Passes: source country, destination country, amount, currency, customer tier, bank policy ID. |
| **B1.4** | AiriPay Engine | **Jurisdiction Analysis + Multi-Rail Queries** | Detects: Domestic? → RTGS only. Intra-GCC? → Prefer AFAQ, offer Buna + Arc. Global? → Prefer Arc, fallback SWIFT. Queries: AFAQ (if eligible), Buna, Circle Arc, Wise, bank's SWIFT. |
| **B1.5** | AiriPay Engine | **Ranks 3 options** | Scores by: cost, speed, compliance, regulator preference. Returns ranked list with FX rates, fees, settlement times, decision rationale. |
| **B1.6** | AiriPay Engine | Returns 3 ranked options to bank's backend | Bank's backend may add its FX markup before displaying to customer. |

#### Phase B2 — Customer Selects & Authorizes

| Step | Actor | Action | Detail |
|---|---|---|---|
| **B2.1** | Bank Customer | Reviews 3 options, selects one (e.g., "Instant Transfer" = Circle ARC) | Customer sees bank-branded labels, not provider names. |
| **B2.2** | Bank Customer | Authorizes with Face ID / 2FA / PIN | Bank's standard authorization flow. |
| **B2.3** | Bank's Backend | Calls AiriPay API: `POST /api/v1/execute` | Passes: selected option ID, customer authorization token. |

#### Phase B3 — Just-In-Time Execution (Multi-Rail Example: Arc/USDC → India)

| Step | Actor | Action | Detail |
|---|---|---|---|
| **B3.1** | AiriPay Engine | Returns execution instruction to bank | "Debit QAR 20,100 from customer, send $5,370 USDC to Circle settlement address." |
| **B3.2** | Bank's Core Banking | **Debits customer's account** (QAR 20,100) | Standard debit. Funds leave customer only now — not before. |
| **B3.3** | Bank's Treasury System (Circle Arc) | **Sends USDC to Circle** from bank's ARC business account | Bank's account initiates USD settlement instruction to Circle. |
| **B3.4** | Circle ARC | **Executes Arc settlement** | USDC settlement on Arc network (sub-second finality), then Circle's India partner wires INR to recipient bank via local rails. |
| **B3.5** | AiriPay Engine | **Monitors status** | Reports to bank: pending → on-chain confirmed → bank delivery → complete. |
| **B3.6** | AiriPay Audit Trail | **Logs everything** | Which rail selected (Arc), why (cost/speed ranking), onchain finality time, bank delivery time (~15 min total), compliance attestation. |
| **B3.7** | Bank | Notifies customer | "Transfer complete in ~15 minutes. Recipient received 1,60,000 INR." |

#### Phase B4 — Settlement Complete

| Step | Actor | Action | Detail |
|---|---|---|---|
| **B4.1** | Recipient | Receives 50,000 SAR in their Al Rajhi Bank account | Standard bank deposit. No crypto awareness required. |
| **B4.2** | Bank | Notifies customer | "Your international transfer of SAR 50,000 has been completed." — standard bank notification. |
| **B4.3** | AiriPay | Provides audit data to bank | Immutable log entry with: timestamps, FX rates, fees, rail used, compliance checks, settlement confirmation. |

---

### Bank Licensing Revenue Model

| License Tier | Annual Fee | What the Bank Gets |
|---|---|---|
| **Tier 1 — Regional / Digital Bank** | $600K–$1.2M/year | AiriPay engine configured for up to 3 corridors, up to 3 payment rails, standard support, quarterly updates |
| **Tier 2 — National Bank** | $1.2M–$3M/year | Full corridor coverage (AFAQ/Buna/Arc/SWIFT), unlimited rails, custom policy engine, dedicated integration team, monthly updates |
| **Tier 3 — Multinational / Tier-1 Bank** | $3M–$10M+/year | Multi-country deployment, custom compliance modules, on-premise option, SLA with penalties, 24/7 support, co-development roadmap, AFAQ/Buna/Arc/SWIFT optimization |

**Why banks would pay this:**

- Cross-border payments are a **$150B+ annual revenue pool** for banks globally.
- They're losing market share to fintechs (Wise, Revolut, Airwallex) because of speed and cost.
- AiriPay lets them offer **fintech-speed settlement at bank-grade trust** — without building it themselves.
- It's a **customer retention and acquisition tool**, not a cost center.

**Why banks would offer it free to their customers:**

- **Attracts large corporate accounts** — cross-border capability is a key decision factor for corporate treasury. A bank offering 15-minute settlement wins over one offering 2-day SWIFT.
- **Retains existing corporates** who are evaluating Wise/Airwallex for cross-border.
- **Cross-border is a loss-leader** — the bank earns from the deposit relationship (corporate keeping $50M+ in operating accounts is worth far more than per-tx fees).
- **Banks still earn on the FX spread** — they can mark up the rate AiriPay surfaces by 0.1–0.3%, generating revenue per transaction without charging an explicit fee.
- **Incentivizes ALL customers** (not just enterprise) to use the bank for international payments instead of third-party apps.

---

## Corridor Feasibility Matrix (GCC Bank Reality)

**Legend**: ✅ Best fit, 🟨 Sometimes, ❌ Not typical. Banks configure which rails are **actively offered** in their geography.

| Corridor | AFAQ | Buna | **Arc/USDC** | SWIFT gpi |
|---|:---:|:---:|:---:|:---:|
| **Domestic (QA, KSA, AE, BH, OM, KW)** | N/A | N/A | N/A | N/A (RTGS only) |
| **UAE ↔ KSA** | ✅ Best | 🟨 Option | 🟨 Option | Fallback |
| **Qatar ↔ UAE** | ✅ Best | 🟨 Option | 🟨 Option | Fallback |
| **Qatar ↔ KSA** | ✅ Best | 🟨 Option | 🟨 Option | Fallback |
| **GCC ↔ US/EU/UK** | ❌ | 🟨 | ✅ **Sub-sec finality, fast delivery** | ✅ Universal |
| **GCC ↔ India** | ❌ | 🟨 | ✅ **Fast delivery via Arc** | ✅ Standard |
| **GCC ↔ Philippines** | ❌ | 🟨 | ✅ **Available** | ✅ Standard |
| **GCC ↔ Pakistan** | ❌ | 🟨 | 🟨 **Payout availability varies** | ✅ Standard |
| **GCC ↔ Africa** | ❌ | 🟨 | 🟨 | ✅ Standard |

**How AiriPay Uses This Matrix**:

1. **Intra-GCC transfers**: Present **AFAQ as the top option** (regulator preferred), with Buna and Arc as fast alternatives.
2. **Global transfers**: Present **Arc/USDC as top option** (sub-second onchain finality + fast delivery where supported), with SWIFT gpi as universal fallback.
3. **Unsupported corridors**: Display informational message: "This corridor is available through your bank's standard network."

---

## Settlement Timing: Onchain vs. End-to-End (Critical Nuance)

**What regulators care about**: Money actually credited to recipient's bank account.

### Arc/USDC Rails Explained

**What Arc claims** (publicly, via [docs.arc.network](https://docs.arc.network)):
- **Sub-second deterministic finality** at the chain layer (USDC settlement is instant onchain)

**What this actually means** (for AiriPay):
- ✅ USDC settlement happens sub-second on the blockchain
- ❌ **End-to-end delivery** (money in recipient's bank account) is **minutes** (typically 10-20 min), depending on:
  - Circle's banking partner's speed
  - Recipient country's local payment rail speed (e.g., SARIE for UAE, RBI instant for India, ACH for US)
  - Recipient bank's processing

**How to position this**:
- **Public**: "Sub-second onchain finality and near-real-time settlement spine"
- **Customer-facing**: "Recipient receives funds in approximately 15 minutes" (for key corridors)
- **Regulator-facing**: "Deterministic onchain settlement (sub-second) + bank-dependent delivery (10-20 minutes for major corridors)"

### Corridor-Specific Delivery Times (Arc/USDC)

| Corridor | Onchain Finality | Bank Delivery | Total E2E | Notes |
|---|---|---|---|---|
| **GCC ↔ US** | Sub-second | 2-5 min (ACH) | ~5-10 min | Circle has direct FedNow partnerships |
| **GCC ↔ EU** | Sub-second | 5-10 min (SEPA) | ~10-15 min | Depends on EU partner availability |
| **GCC ↔ India** | Sub-second | 5-10 min (RBI instant) | ~10-15 min | Circle has RBI-approved partners |
| **GCC ↔ UAE** | Sub-second | 2-5 min (UAEFTS) | ~5-10 min | Domestic-speed local rails |
| **GCC ↔ Pakistan** | Sub-second | Varies (Standard ACE) | 30 min - 2 hours | Pakistan instant rails may not be payout-enabled |

---

## Regulatory Positioning: Who Owns What

**Critical for SAMA, CBUAE, MAS, and CBQ conversations**:

AiriPay is **not** a regulated entity. The **bank is the regulated entity**.

| Responsibility | Owner | AiriPay's Role |
|---|---|---|
| **Customer funds** | Bank holds 100% | Never touches |
| **Execution decision** | Bank (with AiriPay recommendation) | Provides data/evidence |
| **KYC/AML** | Bank (already done) | Adds compliance gates, doesn't replace |
| **Settlement accounts** | Bank owns (RTGS, Circle ARC, Buna, SWIFT) | Orchestrates, doesn't own |
| **Regulatory reporting** | Bank to regulator | Provides audit trail data |
| **Liability** | Bank (for customer and regulator) | AiriPay: indemnified under contract |

**Regulator Questions & Answers**:

**Q1: "What is the regulated entity?"**  
**A**: The bank. AiriPay is a decision support tool. The bank remains the regulated entity for all customer interactions.

**Q2: "Where does screening happen?"**  
**A**: Bank's core screening remains primary (bank's KYC, AML, sanctions lists). AiriPay adds a **policy gating layer** (e.g., "block high-risk corridors") and **compliance evidence** (audit trail of why a rail was selected).

**Q3: "What if there's fraud on an Arc transaction?"**  
**A**: Bank bears the risk and liability (as with all settlements). AiriPay provides decision evidence (audit trail) for investigation.

**Q4: "What's the bank's exposure to stablecoin risk?"**  
**A**: Bank chooses. AiriPay is **rail-agnostic**:
- Bank doesn't want USDC? AiriPay routes to AFAQ/Buna/SWIFT instead for that corridor.
- Bank wants limited Arc exposure? AiriPay can gate Arc to specific tier-1 customers only.
- Regulator restricts Arc? AiriPay pivots to AFAQ/Buna/SWIFT without re-engineering.

**Q5: "Does AiriPay hold customer data or keys?"**  
**A**: No. AiriPay holds: transaction mirrors (status), decision audit trail, compliance check results. Never holds keys, customer wallets, or any fiat account credentials.

---

### Additional Tech Stack for Bank-Embedded Model

| Layer | Recommended Vendor | Alternative Vendors | Purpose |
|---|---|---|---|
| **Bank Integration API** | **gRPC + REST** (AiriPay provides both) | GraphQL | High-performance API for bank's core banking to call AiriPay engine. gRPC for low-latency internal calls; REST for simpler integrations. |
| **On-Premise Deployment** | **Kubernetes (Helm charts)** | Docker Compose, OpenShift | Banks that require on-premise deployment get AiriPay as containerized microservices deployed into their own Kubernetes cluster. |
| **Private Cloud Option** | **AWS Private Link** or **Azure Private Endpoint** | GCP Private Service Connect | For banks on cloud but requiring network isolation. AiriPay deploys in an isolated VPC peered with the bank's network. |
| **Core Banking Middleware** | **[Temenos Transact API](https://www.temenos.com/)** or **[Finastra FusionFabric](https://www.finastra.com/)** | Mambu, Thought Machine Vault | Integration adapters for the bank's core banking system. AiriPay provides pre-built connectors for major platforms. |
| **Bank Regulatory Reporting** | **AiriPay Audit Export API** (CSV, JSON, XBRL) | Custom data pipeline | AiriPay exports audit trail data in formats compatible with CBUAE, SAMA, MAS, and other regulators' reporting requirements. |
| **Multi-Tenancy** | **PostgreSQL Row-Level Security** + **Kubernetes Namespaces** | Dedicated database per bank | Each bank's data is strictly isolated. AiriPay never has cross-bank visibility. |
| **HSM / Key Management** | Bank's own **HSM** (Thales Luna, Utimaco) | AWS CloudHSM, Azure Dedicated HSM | For banks that require hardware security modules. AiriPay never manages HSMs — the bank's infra team does. |
| **Message Queue** | **[Apache Kafka](https://kafka.apache.org/)** | RabbitMQ, AWS SQS | Async event processing between AiriPay engine and bank's core banking system. Ensures no message loss. |
| **API Gateway** | **[Kong](https://konghq.com/)** or bank's existing gateway | Apigee, AWS API Gateway | Rate limiting, authentication, request routing between bank's systems and AiriPay engine. |

---

### Strategic Advantages

1. **One bank deal = thousands of customers.** Selling to Emirates NBD means AiriPay instantly powers cross-border for all their corporate clients — without selling to each enterprise individually.

2. **Regulatory moat.** Once AiriPay is embedded in a regulated bank, competitors need the same bank relationships to compete. Each bank integration is a defensive moat.

3. **Network effects.** Bank A in UAE using AiriPay + Bank B in KSA using AiriPay = an optimized corridor with pre-negotiated rates and instant settlement. More banks on AiriPay → better rates → more banks want to join.

4. **Land and expand.** Start with one corridor (AED ↔ SAR), prove it works, then the bank asks AiriPay to add more corridors (AED ↔ INR, AED ↔ PKR, AED ↔ PHP, etc.).

5. **The bank's incentive is real.** Corporate treasury management is a $500B+ market. Losing a $50M corporate account because they moved to Wise costs the bank far more than AiriPay's license fee.

6. **Zero pre-funding eliminates the biggest enterprise objection.** CFOs don't want idle capital in crypto wallets. With the bank-embedded model, money moves just-in-time.

7. **Crypto is invisible.** The customer never sees USDC, blockchain, or any crypto terminology. This removes the psychological and regulatory friction that scares traditional banking customers.

---

### Risks & Mitigations

| Risk | Severity | Mitigation |
|---|---|---|
| **Bank integration cycles are long** (6–18 months) | High | Target digital-first banks with modern APIs first (Wio Bank, Mashreq Neo, SNB Digital). They move in weeks, not months. |
| **Banks may want to build it themselves** | Medium | AiriPay's value is the multi-rail integration + agentic engine + regulatory knowledge base. Maintaining Circle, Wise, Thunes integrations + a compliance DB across 50+ jurisdictions is not something a bank's IT team should own. |
| **Regulatory uncertainty around stablecoin rails** | Medium | AiriPay is rail-agnostic. If a regulator blocks USDC, AiriPay simply ranks Wise or SWIFT higher for that corridor. The engine adapts; the bank is never locked into one rail. |
| **Circle ARC availability in target corridors** | Medium | AiriPay always has 2–3 fallback rails. Circle is the primary for speed, but not the only option. |
| **Data sovereignty / on-premise requirements** | Low–Medium | AiriPay is deployable on-premise via Kubernetes. Bank's data never leaves their infrastructure. |

---

### Suggested Pilot Approach

1. **Pick one digital-first GCC bank** — e.g., Wio Bank (UAE, backed by ADQ), Mashreq Neo, or SNB Digital (KSA).
2. **Deploy AiriPay as a single-corridor pilot**: AED ↔ SAR.
3. **Two payment rails for the pilot**: Circle ARC (testnet) + the bank's own SWIFT gpi.
4. **Run with 10–20 selected corporate clients** for 3 months.
5. **Measure**: settlement speed, cost savings vs. current SWIFT, customer satisfaction, bank's operational cost reduction.
6. **Use pilot results** to sell to the next bank and expand corridors.

**Pilot success criteria:**

| Metric | Target |
|---|---|
| Average settlement time (Circle ARC rail) | < 30 minutes |
| Cost savings vs. SWIFT (per transaction) | > 40% |
| Customer satisfaction (NPS) | > 70 |
| System uptime | > 99.9% |
| Audit trail completeness | 100% of transactions logged |

---

## Getting Started (Pilot)

The controlled pilot uses **Circle ARC testnet**. No real funds are involved.

### Prerequisites

1. Circle ARC **sandbox/testnet** API key → [Circle Developer Console](https://console.circle.com/)
2. Fireblocks **sandbox** API key (if using Fireblocks) → [Fireblocks Sandbox](https://www.fireblocks.com/)
3. Node.js 18+ and npm/yarn
4. PostgreSQL database (Supabase free tier recommended for pilot)

### Environment Variables

```env
# Circle ARC (Testnet)
CIRCLE_API_KEY=TEST_xxxxxxxxxxxx
CIRCLE_API_BASE_URL=https://api-sandbox.circle.com

# Fireblocks (Sandbox) — optional
FIREBLOCKS_API_KEY=xxxxxxxx
FIREBLOCKS_API_SECRET_PATH=./fireblocks-secret.key
FIREBLOCKS_VAULT_ACCOUNT_ID=0

# Database
DATABASE_URL=postgresql://user:pass@host:5432/airipay

# Auth
AUTH0_DOMAIN=airipay.auth0.com
AUTH0_CLIENT_ID=xxxxxxxx
AUTH0_CLIENT_SECRET=xxxxxxxx

# Notifications
SENDGRID_API_KEY=SG.xxxxxxxx

# Subscription Billing
STRIPE_SECRET_KEY=sk_test_xxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxx
```

### Project Structure (Target)

```
airipayV7/
├── README.md                           # This file
├── packages/
│   ├── frontend/                       # React + Vite enterprise dashboard
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   ├── PaymentForm.jsx     # New payment initiation
│   │   │   │   ├── RailOptions.jsx     # 3-option ranking display
│   │   │   │   ├── TransactionList.jsx # Transaction history
│   │   │   │   └── AuditTrail.jsx      # Compliance audit viewer
│   │   │   ├── services/
│   │   │   │   └── api.js              # API client
│   │   │   └── App.jsx
│   │   └── package.json
│   │
│   └── backend/                        # Node.js orchestration server
│       ├── src/
│       │   ├── routes/                 # API endpoints
│       │   ├── services/
│       │   │   ├── circle-arc.js       # Circle ARC integration
│       │   │   ├── wise.js             # Wise Business API integration
│       │   │   ├── thunes.js           # Thunes integration
│       │   │   ├── fireblocks.js       # Fireblocks read-only integration
│       │   │   └── compliance.js       # Sanctions/PEP screening
│       │   ├── engine/
│       │   │   ├── jurisdiction.js     # Jurisdiction analyzer
│       │   │   ├── policy.js           # Enterprise policy engine
│       │   │   ├── ranker.js           # Cost/speed/compliance ranker
│       │   │   └── agent.js            # Agentic orchestrator
│       │   ├── audit/
│       │   │   └── trail.js            # Immutable audit logger
│       │   └── server.js
│       └── package.json
│
├── infra/                              # Infrastructure as Code
│   └── docker-compose.yml
└── docs/                               # Additional documentation
    ├── api-spec.yaml                   # OpenAPI spec
    └── compliance-matrix.md            # Jurisdiction compliance matrix
```

---

## Quick Start (Local MVP)

```bash
# 1. Clone and install
cd airipayV7
npm install

# 2. Start both servers
npm run dev
# Backend: http://localhost:3001
# Frontend: http://localhost:5173

# 3. Login with demo credentials
#    Email:    treasury@gulftradingcorp.ae
#    Password: demo1234
```

All payment rails are **mocked** — no real funds are processed, no external accounts needed.

---

## Docker

```bash
docker compose up --build
# Frontend: http://localhost:5173
# Backend:  http://localhost:3001
```

---

## Third-Party Vendor Setup Guide

### What You Need Right Now (MVP with Mocks): **Nothing**

The MVP runs entirely self-contained with hardcoded mock responses. No external accounts, APIs, or services required. Just `npm install && npm run dev`.

### When You're Ready for Real Integrations

| # | Vendor | Purpose | Account URL | Free Tier? | When to Set Up |
|---|--------|---------|-------------|------------|----------------|
| 1 | **Circle** | Circle ARC (USDC rails, cross-border settlement) | [circle.com/developer](https://developers.circle.com) | Yes (sandbox) | Phase 1 — primary rail |
| 2 | **Wise** | Wise Business API (mid-market FX transfers) | [wise.com/gb/business/api](https://wise.com/gb/business/api) | Yes (sandbox) | Phase 1 — second rail |
| 3 | **Fireblocks** | Wallets & stablecoin custody (enterprise manages own keys) | [fireblocks.com](https://www.fireblocks.com/developer-sandbox-sign-up/) | Sandbox available | Phase 1 — if using Circle via Fireblocks vault |
| 4 | **ComplyAdvantage** | Sanctions screening, PEP checks, adverse media | [complyadvantage.com](https://complyadvantage.com) | Trial available | Phase 2 — compliance |
| 5 | **Supabase** | PostgreSQL + Row Level Security (replaces SQLite) | [supabase.com](https://supabase.com) | Yes (free tier) | Phase 2 — production DB |
| 6 | **Auth0** or **Clerk** | Production authentication + SSO for enterprises | [auth0.com](https://auth0.com) / [clerk.com](https://clerk.com) | Yes (free tier) | Phase 2 — production auth |
| 7 | **SendGrid** | Email notifications (transfer confirmations, alerts) | [sendgrid.com](https://sendgrid.com) | 100 emails/day free | Phase 2 — notifications |
| 8 | **Railway** or **Render** | Backend hosting (Node.js server) | [railway.app](https://railway.app) / [render.com](https://render.com) | Yes (limited) | Deployment |
| 9 | **Vercel** | Frontend hosting (React SPA) | [vercel.com](https://vercel.com) | Yes (free tier) | Deployment |

### NOT Needed

| Vendor | Why Not |
|--------|---------|
| IBM Watson Orchestrate | AiriPay builds its own agentic orchestrator (jurisdiction → compliance → quote → policy → rank) |
| Stripe | AiriPay orchestrates cross-border payment rails, not card payments |
| AWS/Azure/GCP | Railway or Render is simpler and cheaper for MVP. Cloud only if bank-embedded |

### Phase 1 Setup Steps (Circle + Wise Sandbox)

1. **Circle Developer Account**
   - Go to [developers.circle.com](https://developers.circle.com)
   - Create account → Sandbox environment
   - Get API Key from Dashboard → Developers → API Keys
   - Add to `.env`: `CIRCLE_API_KEY=SAND_API_KEY_xxx`
   - Enable: Transfers API, Accounts API
   - Documentation: [developers.circle.com/circle-mint](https://developers.circle.com/circle-mint)

2. **Wise Business API**
   - Go to [wise.com/gb/business/api](https://wise.com/gb/business/api)
   - Create Business account → Apply for API access
   - Sandbox: [api.sandbox.transferwise.tech](https://api.sandbox.transferwise.tech)
   - Get API Token from Settings → API tokens
   - Add to `.env`: `WISE_API_KEY=sandbox_token_xxx`

3. **Replace Mock Services**
   - Each file in `packages/backend/src/services/` has a comment indicating where to add real API calls
   - The mock functions match the real API response shapes, so swapping is straightforward

---

## Intellectual Property Protection

AiriPay's payment orchestration technology, algorithms, and business model are protected by comprehensive intellectual property measures:

### Patent Protection

**Patent Pending**: US Provisional Application (Filed 2026)
- **Key Innovations**: Non-custodial cross-border payment orchestration, real-time rail ranking algorithm, jurisdiction-aware compliance routing, RTGS overlay architecture
- **Coverage**: US provisional filing completed, PCT international filing in progress (2026-2027)
- **GCC Filing**: GCC Unified Patent Application planned for Q2 2026

See [IP-PROTECTION-STRATEGY.md](docs/IP-PROTECTION-STRATEGY.md) for complete patent filing roadmap.

### Trademark Protection

**AiriPay™** is a protected trademark of AiriPay Technologies:
- Trademark applications filed in US, GCC, UK, and EU
- Covers: Payment processing services, financial software, cross-border payment platforms
- Registration pending (2026)

### Trade Secrets

The following AiriPay components are protected as confidential trade secrets:
- **Rail Ranking Algorithm** ([ranker.js](packages/backend/src/engine/ranker.js)) - Proprietary scoring weights and optimization formulas
- **Policy Evaluation Engine** ([policy.js](packages/backend/src/engine/policy.js)) - Enterprise policy scoring methodology
- **Orchestration Logic** ([agent.js](packages/backend/src/engine/agent.js)) - Payment routing decision tree
- **Compliance Integration** ([compliance.js](packages/backend/src/services/compliance.js)) - Risk assessment algorithms

⚠️ **All source code files contain copyright notices and confidentiality warnings. Unauthorized use, disclosure, or distribution violates copyright, patent, and trade secret law.**

### Licensing Model

AiriPay technology is licensed exclusively to financial institutions under white-label agreements:
- **No source code distribution** - Banks receive compiled SDKs only
- **Server-side execution** - Core algorithms never deployed to client devices
- **License validation** - SDKs include phone-home validation and usage tracking
- **Non-exclusive licensing** - Banks can license in their geographic region
- **IP ownership** - AiriPay Technologies retains all IP rights

### Before Bank Discussions

**CRITICAL**: If you are considering discussions with banks or financial institutions:

1. ✅ **File Provisional Patent First** - Complete before ANY bank meetings (Cost: $3K-$5K, Timeline: 2 weeks)
2. ✅ **Execute NDA** - Require signed NDA before sharing any technical details
3. ✅ **Progressive Disclosure** - Share high-level concepts only, no source code or algorithms
4. ❌ **Never Share Source Code** - Only provide compiled SDKs after licensing agreement
5. ❌ **Never Accept Joint IP Terms** - AiriPay Technologies retains 100% IP ownership

📋 **See [IP-CHECKLIST.md](docs/IP-CHECKLIST.md) for week-by-week action plan before market engagement.**

📄 **See [IP-PROTECTION-STRATEGY.md](docs/IP-PROTECTION-STRATEGY.md) for comprehensive IP protection framework ($77K-$139K budget, 12-month roadmap).**

### Legal Jurisdiction

- **Enforcement Venue**: DIFC Courts (Dubai International Financial Centre) for GCC matters, US Federal Courts for US/international matters
- **Governing Law**: Delaware law for US operations, DIFC law for GCC operations
- **Patents**: Coordinated filing via GCC Patent Office (Saudi Arabia) + PCT for international protection

---

## License

**Proprietary and Confidential**

Copyright © 2024-2026 AiriPay Technologies. All rights reserved.

**Patent Pending**: US Provisional Application (Filed 2026), International PCT Filing (In Progress)

This software and associated documentation files (the "Software") contain proprietary and confidential information that is protected by copyright, patent, and trade secret law. The Software is licensed, not sold.

### Restrictions

- **No Distribution**: You may not distribute, sublicense, or transfer the Software or any portion thereof
- **No Reverse Engineering**: You may not decompile, disassemble, or reverse engineer the Software
- **No Source Code Access**: Source code is never provided to licensees; compiled SDKs only
- **Confidential Information**: All algorithms, methodologies, and technical implementations are trade secrets
- **License Required**: Commercial use requires signed white-label licensing agreement

### Authorized Use

- Development and testing for evaluation purposes only (with explicit written permission)
- Licensed deployment by authorized financial institution partners (under signed agreements)
- Internal use by AiriPay Technologies employees and contractors (under NDA)

### Violations

Unauthorized use, reproduction, disclosure, or distribution of this Software or any portion thereof may result in severe civil and criminal penalties, and will be prosecuted to the maximum extent possible under the law.

### Contact

For licensing inquiries: [Insert licensing contact email]  
For legal matters: [Insert legal contact email]

---

**AiriPay™** is a trademark of AiriPay Technologies (trademark registration pending).