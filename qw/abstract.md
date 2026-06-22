# Abstract: Decentralized Skills Exchange and Mutual Credit Network

## Overview

A search-aware framework for discovering and appreciating developers across open source,
startup, and IT projects — without requiring money to change hands.
Participants find each other through verified skill reputation, commit to work via
signed contracts, and compensate contributions with personal credit tokens denominated
in time.

It solves three interlocked problems:
- **Verified skill rating** — contribution history is tamper-proof and resistant to fakes and bots
- **Fair compensation** — a time-based credit mechanism attracts qualified contributors and rewards real work
- **Decentralized trust** — reputation scores are computed locally by each participants rules, with no central authority

---

## The Core Idea: Quant as a Unit of Measure, Not a Currency

The fundamental design insight is that **Quant is not a token — it is a unit of measure**, analogous to a meter or an hour. 
It denominates the value of work on contracts but has no ledger of its own. The canonical definition is:

> **1 Quant = Quarter of hour i.e. 15 minutes of work at minimum qualification**

The formula for pricing any job is:

```
Quants = Hours × Rate × ko × km
```

Where:
- **Hours / Rate** — declared hours and skill level of the performer
- **ko** — objective coefficient (equipment use, working conditions, hazard)
- **km** — subjective motivation/quality coefficient derived from reputation history
ko & km might be omitted for simplifying negotiation.

The actual currency in the system is each person's **personal token** — a signed IOU denominated in Quants. 
You never hold "Quants"; you hold Alice's token, Bob's token, each comparable because they share the same unit.
This eliminates inflation by design: the total supply of token promises is bounded by productive human time, and no authority can "print" new units.

---

## Web of Trust: Reputation from Completed Work

The reputation system is not a separate computation engine — it **emerges from the job lifecycle itself**:

```
Job Request → Acceptance → Milestones → Completion → Multi-party Sign
```

Each signed, multi-party signed contract encodes a trust relationship: if Alice and Bob completed a job, Alice trusts Bob in domain X. 
If Carol trusts Alice, she can walk the chain `Carol → Alice → Bob` and evaluate Bob's competence in that domain. 
The theory of six degrees of separation makes this tractable across large networks.

Trust is:
- **Domain-specific** — trust in frontend development is independent of trust in accounting
- **Directional and local** — each agent computes their own subjective score with their own weights and tolerances
- **Non-canonical** — there is no global credit score; there are N subjective readings of the same public record

**Sybil resistance** is an elegant inverse of trust propagation: trust flows down a signed chain; a block propagates up it. 
A bot-farm root account, once identified, pulls down every account that signed through it. 
Detection is social, not algorithmic — any participant can announce a suspicious segment, and the network can react automatically.

The open governance question is: **who can initiate a block, and what downstream threshold triggers an automatic cascade?** 
Pure social consensus leaves this ambiguous. For deployments requiring hard finality, 
this specific decision — and only this decision — can be anchored to a conventional shared ledger 
(a standard blockchain or any common append-only log), while the trust graph itself remains fully P2P. 
The shared ledger acts as a neutral arbiter of block propagation without becoming a central authority over the economy.

---

## Balance Mechanics: Deficit is Permitted, but Publicly Priced

Tokens flow, not accumulate. A participant's **net position** is:

```
net_position = Σ(work delivered, in Quants) − Σ(own token issued, in Quants)
```

This quantity is a derivable fact from the public event graph — visible to anyone. 
The protocol does not enforce balance; **the market of counterparties prices it**. 
A reliable worker's token trades at par or above. A chronic over-issuer's token is discounted or refused, 
independently by every counterparty according to their own configuration.

The `wot_zome` therefore functions as a **pricing engine**, not a simple gatekeeper. 
It returns a per-counterparty assessment informing both *whether* to accept a token and at *what* effective exchange rate. 
A free-floating, reputation-priced personal currency — with no central authority setting the rate.

What the corporate paradigm frames as a **walk-away risk** is more accurately a **human-centric paradigm shift**: 
in conventional employment, legal and institutional mechanisms enforce payment whether or not the debtor is willing. 
This system replaces legal enforcement with social consequence — which is, in fact, how pre-institutional economies functioned. 
The community bears the responsibility that institutions formerly held. This is a design feature, not a flaw, but it does carry a **society-level risk**: 
if network density is insufficient or community norms are immature, organized defection could erode trust faster than the cascade block can contain it. 
The practical mitigations are the standard ones for unsecured credit: counterparties limit exposure per issuer, 
require shorter settlement cycles for unproven identities, and weight recent delivery history heavily. 
New identities should face conservative default limits on deficit accumulation until a track record is established.

---

## Technical Architecture: Holochain Zomes

The system maps naturally onto three Holochain zomes:

| Zome | Responsibility |
|---|---|
| `skill_zome` | Qualification taxonomy, endorsements, rate tables in Quant/hr |
| `wot_zome` | Trust graph: propagation, domain scoping, local scoring, cascade block, net-position exposure |
| `exchange_zome` | Accept/reject others' tokens, cross-token swaps, dispute records |

`QuantUnit` is a **config constant**, not an entry type — it needs no mint, no ledger, no zome of its own. 
All transactions are final; disputes are appended records, not reversals. 
Tampering is provable because neighbors hold copies and any divergence from their stored version constitutes evidence.

The underlying data model follows **hREA** (Holochain Resource-Event-Agent), an implementation of the ValueFlows vocabulary. 
All economic activity — job contracts, deliveries, token transfers, disputes — is recorded as `EconomicEvent` entries. 
This makes `net_position` a directly queryable derivation from the public event graph rather than a separately maintained balance, 
and it means the `wot_zome` has no need for its own reputation store — it walks the existing `EconomicEvent` graph to derive trust transitively.

Critically, this architecture requires **no distributed consensus**. Unlike blockchains, there is no global agreement round, 
no mining, and no validator set. Each node maintains its own signed local chain; cross-verification happens bilaterally between transaction parties
and their neighbors. This makes the system viable on intermittent, low-bandwidth, or mesh networks where consensus protocols would be impractical.

The network layer requires no internet: nodes can operate as Wi-Fi hotspots with TURN servers, or over wired infrastructure with DNS. 
Recommended components: Yggdrasil for distributed routing, coturn for TURN/relay, Protobuf for the API protocol, Tox/Matrix for messaging.

---

## Skills Social Network: The Human Layer

The system doubles as a **decentralized professional skills network** — a "friends-of-friends" graph using public/private key pairs, without a central server. 

Key properties:

Unlike existing timebank platforms (hOurWorld, TimeRepublik, Community Weaver, Community Forge), 
which avoid mentioning tax obligations entirely in case of providing professional service,
this system acknowledges that mutual exchange of **professional services qualifies as barter** in most jurisdictions and is taxable accordingly. 
Timebanks sidestep this by limiting participation to non-commercial, non-professional help — but impose no such restriction explicitly,
leaving participants exposed.
This system resolves the tension through the open-source co-authorship framing described above: contributors act as project co-authors, 
not as service providers to each other, which removes the barter classification.

- **Reputation attaches to an account, not a person** — the cryptographic keypair is the identity. 
The same human can hold multiple accounts with independent reputations; an account can be pseudonymous or fully anonymous. 
This separation is architecturally intentional: the system tracks the reliability of a *signing key's history*, not the legal identity behind it.
- **Anonymization is a first-class option** — a participant can route transactions through a trusted mixer that re-signs under its own key, 
breaking the observable link between the originating account and the counterparty. 
The mixer's own reputation guarantees the anonymized transaction; the original account's reputation remains intact 
and continues to accumulate behind the pseudonym.
- **No mandatory identity** — real-person verification is not required; reputation is what matters
- **No common rules** — each participant configures their own privacy level and trust filters
- **Qualification is subjective-professional** — "who are these people to me, who gave this rating?" is the operative question, not a global score
- **Bot resistance as a side effect** — any quantity of bots traces back through a small number of real signers;
blocking the signers cascades to all their bots

The API lifecycle (in seven parts: Introduction, Profile, History, Revocation, Job Start, Job Delivery, Appeal) 
uses trust chains rather than central authentication. Loss of keys is recovered through multi-friend confirmation over independent channels -
more robust than email or SMS recovery.

---


## Economic Properties

| Property | Effect |
|---|---|
| Time-backed value | No inflation; bounded by productive human hours |
| No central issuer | No monetary policy failure; no quantitative easing |
| Deficit is social, not protocol | Market self-regulates; no enforcement overhead |
| Trust from contracts | No separate reputation computation; history IS reputation |
| Cascade block | Sybil resistance without proactive bot detection |
| Balance trends zero | No hoarding incentive; no "rich get richer" dynamic |
| Inheritance optional | Debts do not transfer; positive balances may transfer by choice |

---

## Optional Coordination Server: Monetization for the Organizing Entity

The P2P architecture is fully functional without any central server — participants exchange directly, trust chains propagate through signed contracts,
and local agents compute their own scores. However, two computationally expensive operations create a natural service opportunity:
**trust-chain traversal** (walking the WoT graph across many hops to score an unknown counterparty)
and **cross-token exchange coordination** (finding a willing swap counterparty when Alice holds Bob's token but wants Carol's).

A voluntary coordination server can offer these as paid services without becoming a point of trust or control.
The architecture remains unchanged: the server is a **convenience layer**, not an authority.
All results it returns are verifiable against the public event graph; it holds no keys and cannot forge signatures.

### Services and Fee Model

| Service | What it does | Fee basis |
|---|---|---|
| **Chain calculation** | Traverses the WoT graph on behalf of a client, returns a signed path and aggregated trust score for a requested counterparty | Per-query, in Quants |
| **Exchange marketplace** | Matches holders of token A who want token B, coordinates atomic swaps | Percentage of swap volume, in Quants |
| **Rating bureau** | Aggregates and re-signs a filtered work history for a participant (e.g. "frontend skills, last 2 years"), analogous to a credit bureau report | Per-report subscription, in Quants |
| **Anonymity mixing** | Issues bureau-signed tokens from its own account in exchange for participant tokens, breaking the direct link between payer and payee | Spread or flat fee, in Quants |
| **Vault / neighbor storage** | Stores signed transaction copies on behalf of participants who lack always-on nodes, enabling tamper-evidence without self-hosting | Storage subscription, in Quants |
| **Shadow Quant credit** | Issues a tokenized mirror of Quant credits on a conventional public ledger (e.g. an ERC-20 token), pegged to verified P2P net positions — enabling interoperability with the broader crypto ecosystem and fiat off-ramps | Minting/burning fee + spread, in Quants or fiat |
| **Community life-work insurance** | Pools a small fraction of each transaction into a mutual fund; compensates counterparties holding abandoned-identity debt up to their covered exposure — turning the society-level walk-away risk into a managed, premium-funded instrument | Per-transaction micro-premium, in Quants |

All fees are denominated in Quants, paid in the server operator's own personal token — the same instrument everyone else uses.
The server's own net position and reputation are public and subject to the same social pricing as any participant.
This creates a self-regulating incentive: a server that over-charges, returns inaccurate chains, or mishandles anonymity
takes measurable reputation damage that its clients can verify.

### Why This is Not Centralization

The critical distinction is that the server **computes but does not certify**. A client receiving a chain-traversal
result can spot-check any hop by querying that hop's neighbors directly. 
The server's signature on a result is just its reputation staked on the accuracy of the computation — not a canonical truth.
Multiple competing servers can exist; clients route to whichever their own WoT scores most trustworthy or cheapest.

This mirrors how a rating agency works in conventional credit markets: useful for efficiency, but not a single point of failure,
and discreditable if consistently wrong.

### Organizational Sustainability

Because the coordination server earns Quants — units redeemable for real work from the network it serves — 
the operating organization can fund itself entirely within the system:

- Server infrastructure costs are offset by query fees
- Staff who maintain the server earn Quants for that work, spendable inside the network
- The organization accrues positive net position as long as it delivers reliable service

This is the closest analog in the system to a bank or clearing house: a trusted intermediary that earns a margin for reducing friction,
without holding deposits or issuing money. The organization can also offer the **anonymity mixing** service (as described in eco.md), 
acting as a Quant bank that issues its own tokens in exchange for others, charges a spread, and shoulders the counterparty-identification burden -
a service participants pay for precisely because it protects their privacy while the bank's own reputation provides the guarantee.

---

## Summary

The system described across these documents is a **mutual credit network with work-time as its unit of account**,
a **Web of Trust derived from job completion history**, and a **decentralized skills exchange** built on peer-to-peer cryptographic identities.
It does not replace Bitcoin (which serves as a store of value and speculative instrument);
it serves a different purpose — enabling qualified people to collaborate and compensate each other without fiat money, while maintaining a publicly auditable,
socially regulated reputation system that resists both Sybil attacks and centralized capture.

A coordination server adds efficiency and creates a viable monetization path for the organizing entity, without compromising the decentralized trust model:
it computes, matches, and mixes — but the network verifies, and the network prices the server's own reputation just like any other participant's.

---

2021'6 Vladimir Krinitsyn
