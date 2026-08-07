# Peer-Verified Contribution Network

*Time contributed to shared projects, signed by the people you worked with.*

## Overview

A search-aware framework for discovering and recognizing developers across open
source, startup, and IT projects — without requiring money to change hands.
Participants find each other through verified contribution reputation, commit to
work through signed contracts, and record each contribution as a
counterparty-signed attestation denominated in time.

It solves three interlocked problems:

- **Verified skill rating** — contribution history is counterparty-signed,
  tamper-evident, and resistant to fakes and bots.
- **Fair recognition** — a time-based unit attracts qualified contributors and
  records real work, priced by the market that requested it.
- **Decentralized trust** — reputation is computed locally by each participant's
  own rules, with no central authority and no global score.

---

## The Core Idea: Quant as a Unit of Measure, Not a Currency

The fundamental design insight is that **Quant is not a token — it is a unit of
measure**, analogous to a meter or an hour. It denominates the value of work on
contracts but has no ledger of its own. The canonical definition is:

> **1 Quant = a quarter hour — 15 minutes of work at minimum qualification.**

The price of any job is:

```
Quants = Hours × Rate × ko × km
```

Where:

- **Hours / Rate** — declared hours and skill level of the performer
- **ko** — objective coefficient (equipment use, working conditions, hazard)
- **km** — subjective motivation/quality coefficient derived from reputation
  history

`ko` and `km` may be omitted to simplify negotiation.

What each person holds is not "Quants" but their counterparties' **signed
contribution records** — Alice's record, Bob's record — each comparable because
they share the same unit. These records are reciprocity signals, not debt
instruments: they carry no enforceable obligation, which is precisely what keeps
the system clear of money-transmission and lending questions.

One caveat the honest version of this document keeps in view: the *unit* is
fixed, but the system does **not** guarantee price stability. `Rate`, `ko`, and
`km` are subjective multipliers, so the Quant-denominated cost of comparable
work can drift over time. The claim is a stable unit of measure — not an
inflation-free economy.

---

## Web of Trust: Reputation from Completed Work

The reputation system is not a separate computation engine — it **emerges from
the job lifecycle itself**:

```
Job Request → Acceptance → Milestones → Completion → Multi-party Sign
```

Each multi-party-signed contract encodes a trust relationship: if Alice and Bob
completed a job, Alice trusts Bob in domain X. If Carol trusts Alice, she can
walk the chain `Carol → Alice → Bob` and evaluate Bob's competence in that
domain. Six degrees of separation makes this tractable across large networks.

Trust is:

- **Domain-specific** — trust in frontend development is independent of trust in
  accounting.
- **Directional and local** — each agent computes their own subjective score
  with their own weights and tolerances.
- **Non-canonical** — there is no global credit score; there are N subjective
  readings of the same public record.

**Sybil resistance** is an inverse of trust propagation: trust flows down a
signed chain, and a block propagates up it. Every account has to be signed into
the web of trust by someone, so behind any bot farm sits a limited number of
real signing accounts. Once identified, blocking those accounts — and publishing
the proof of contamination so others cascade the same block — takes the whole
farm down at its root. Detection is social, not algorithmic: any participant can
flag a suspicious segment, and the network reacts.

The open governance question is **who can initiate a block, and what downstream
threshold triggers an automatic cascade.** Pure social consensus leaves this
ambiguous. For deployments that require hard finality, this one decision — and
only this decision — can be anchored to a conventional shared ledger (a standard
blockchain or any common append-only log) while the trust graph itself remains
fully peer-to-peer. The shared ledger arbitrates block propagation without
becoming an authority over the economy.

---

## Balance Mechanics: Deficit is Permitted, but Publicly Priced

Contribution records flow; they do not accumulate into a balance anyone must
settle. A participant's **net position** is:

```
net_position = Σ(work delivered, in Quants) − Σ(own records issued, in Quants)
```

This quantity is derivable from the public record — visible to anyone. The
protocol does not enforce balance; **the market of counterparties prices it.** A
reliable worker's record is accepted at face value or better; a chronic
over-issuer's record is discounted or refused, independently by every
counterparty according to their own configuration.

Pricing is a pure market, not a hidden score. A worker asks a price and the
client approves it before work starts; if the worker asks more than the client
feels they received, the client's signature and rating reflect it, and the
worker's reputation falls. The trust layer therefore functions as a **pricing
engine**, not a gatekeeper: it returns a per-counterparty assessment of whether
to accept a record and at what effective rate — reputation-priced, with no
central authority setting the rate.

What a corporate paradigm frames as **walk-away risk** is more accurately a
**paradigm shift**. Conventional employment uses legal and institutional
machinery to enforce payment whether or not the debtor is willing; this system
replaces legal enforcement with social consequence — which is how
pre-institutional economies already worked. The community carries the
responsibility that institutions formerly held.

That is a design feature, but it carries a **society-level risk**: if network
density is insufficient or community norms are immature, organized defection
could erode trust faster than the cascade block contains it. The mitigations are
the standard ones for unsecured reciprocity, and they follow directly from the
real attack model — not chain-forgery (expensive, provable, permanent) but
**abandoning an identity and starting fresh**:

- A new account is **unknown-risk, not neutral**; limits scale with history
  depth and counterparty diversity, not with any score.
- The **path to you** through the web of trust matters more than any number —
  paths cannot be forged by a fresh key.
- Counterparties limit exposure per issuer, require shorter settlement cycles
  for unproven identities, and weight recent delivery heavily.

---

## Data Model & Architecture: Signed Records, No Consensus

The record is best modeled as a **Verifiable Credential**: the issuer is the
counterparty, the subject is the worker, and the claim is time and skill. That
gives DIDs, mature mobile wallets, and SD-JWT / BBS+ selective disclosure
natively — and it means there is no ledger to maintain and no global agreement
round to run.

Integrity does not come from hash-chaining or global ordering. Every contract is
bilateral and both parties sign, so the record someone would want to hide lives
on the counterparty's side, which they do not control. **Omission is provable by
production, not by gap analysis.** Detecting it requires only:

- **Dual indexing** — every contract is published independently under both
  parties' keys.
- **Union query** — evaluation asks "all records referencing A," never A's
  self-report.
- **Publication window T** — hours to a day, measured from the counterparty's
  signature timestamp; a client checks automatically at offer time and shows a
  plain signal.

Because records are counterparty-signed and dual-indexed, `net_position` is a
directly queryable derivation from the public graph rather than a separately
maintained balance, and the trust layer needs no reputation store of its own —
it walks the existing record graph to derive trust transitively.

Critically, this needs **no distributed consensus** — no mining, no validator
set, no global agreement round. Each participant holds their own signed records;
cross-verification happens bilaterally between transaction parties and their
neighbors. That makes the system viable on intermittent, low-bandwidth, or mesh
networks where consensus protocols would be impractical — and "no internet" here
means no hosted data store, not no discovery.

**Implementation note.** The current target stack is `did:key` / `did:web` for
identity, W3C Verifiable Credentials with SD-JWT selective disclosure for
attestations, **Nostr** relays for storage and transport (with ATProto as a
migration target if enforced schemas or a real search index become binding
constraints), and Tauri v2 for Android / iOS / web from one codebase, with
signing delegated to an external signer app. Discovery uses relay lists plus a
DHT, and referral queries route greedily toward tag-similar contacts rather than
flooding. **Holochain** is kept on the bench for the single scenario where
protocol-enforced double-spend prevention becomes non-negotiable; it was set
aside once detection-over-prevention was accepted.

---

## Skills Social Network: The Human Layer

The system doubles as a **peer-to-peer professional skills network** — a
friends-of-friends graph keyed on public/private key pairs, with no central
server. Key properties:

- **Reputation attaches to an account, not a person.** The cryptographic keypair
  is the identity. The same human can hold multiple accounts with independent
  reputations; an account can be pseudonymous or fully anonymous. The system
  tracks the reliability of a *signing key's history*, not the legal identity
  behind it — and multiple accounts do not weaken Sybil resistance, because an
  account with no real signers behind it carries no reputation, and any real
  signers can be traced and blocked.
- **No mandatory identity** — real-person verification is not required;
  reputation is what matters.
- **No common rules** — each participant configures their own privacy level and
  trust filters.
- **Qualification is subjective-professional** — the operative question is "who
  are these people to me, and who gave this rating?", not a global score.
- **Bot resistance as a side effect** — any quantity of bots traces back through
  a small number of real signers; blocking the signers cascades to all their
  bots.

There is deliberately **no unlinkable mixer.** Reputation requires a traceable,
counterparty-signed history, so a service that re-signs transactions under its
own key to break that link would sever the very chains the cascade block and
market pricing depend on. Pseudonymity — many independent accounts — is
first-class; unlinkable anonymization of a single account's economic position is
not.

The API lifecycle (Introduction, Profile, History, Revocation, Job Start, Job
Delivery, Appeal) uses trust chains rather than central authentication. Lost
keys are recovered through multi-friend confirmation over independent channels —
more robust than email or SMS recovery.

### Legal framing: contribution, not barter

Timebank platforms (hOurWorld, TimeRepublik, Community Weaver, Community Forge)
avoid the tax question by limiting participation to non-commercial,
non-professional help — but rarely say so explicitly, leaving participants
exposed. Mutual exchange of **professional services generally qualifies as
barter** and is taxable accordingly.

This system's defense is structural, and it holds under two conditions:

1. Work is on a **declared open-source project** — the output is public and
   non-appropriable.
2. **No project is controlled by the counterparty** in a way that privatizes the
   benefit.

The structure is: A contributes to a project led by B; separately, B contributes
to a project led by A. Neither is the other's service recipient — which is
materially different from reciprocal services, and is what millions of
open-source contributors already do without a tax event. The barter
classification turns on one party receiving a service; co-authorship of public,
non-appropriable work does not fit that shape.

The boundary is explicit: activity beyond these two conditions — direct
bilateral work-for-work, or contribution to a counterparty-controlled private
project — is the participants' own tax responsibility, not something the
protocol launders. This framing needs a written opinion from a tax attorney
before it is relied on.

---

## Economic Properties

| Property | Effect |
|---|---|
| Time-based unit | A stable unit of account; not a claim of price stability — rate multipliers are subjective |
| No central issuer | No monetary policy to fail; no quantitative easing |
| Deficit is social, not protocol | Market self-regulates; no enforcement overhead |
| Trust from contracts | No separate reputation computation; history *is* reputation |
| Cascade block | Sybil resistance without proactive bot detection |
| Attack model is exit, not forgery | New accounts start at unknown-risk; good history is slow to build |
| Balance trends to zero | No hoarding incentive; no "rich get richer" dynamic |
| Inheritance optional | A deficit does not transfer; a positive position may transfer by choice |

---

## Optional Coordination Server: Monetization for the Organizing Entity

The peer-to-peer architecture is fully functional without any central server —
participants exchange directly, trust chains propagate through signed contracts,
and local agents compute their own scores. But two operations are expensive
enough to create a natural service opportunity: **trust-chain traversal**
(walking the graph across many hops to score an unknown counterparty) and
**filtered history aggregation** (assembling a signed, scoped work history on
demand).

A voluntary coordination server offers these as paid services without becoming a
point of trust or control. It is a **convenience layer**, not an authority:
every result it returns is verifiable against the public record; it holds no
keys and cannot forge signatures.

### Services and Fee Model

| Service | What it does | Fee basis |
|---|---|---|
| **Chain calculation** | Traverses the trust graph on behalf of a client; returns a signed path and aggregated trust score for a requested counterparty | Per-query, in Quants |
| **Rating bureau** | Aggregates and re-signs a filtered work history (e.g. "frontend, last 2 years"), analogous to a credit-bureau report | Per-report subscription, in Quants |
| **Vault / neighbor storage** | Stores signed record copies for participants who lack always-on nodes, enabling tamper-evidence without self-hosting | Storage subscription, in Quants |
| **Community life-work insurance** | Pools a small fraction of each transaction into a mutual fund; compensates counterparties holding abandoned-identity deficit up to their covered exposure | Per-transaction micro-premium, in Quants |

All fees are denominated in Quants, paid in the server operator's own signed
record — the same instrument everyone else uses. The server's own net position
and reputation are public and priced by the same social mechanism as any
participant's, so a server that over-charges or returns inaccurate chains takes
measurable, verifiable reputation damage.

Deliberately **not offered**: any service that creates transferability. There is
no cross-token exchange marketplace, no unlinkable mixing, and no mirrored
on-ledger token — each would reintroduce the money-transmission, Sybil, and
tax-classification problems the design exists to avoid.

### Why This is Not Centralization

The server **computes but does not certify.** A client receiving a
chain-traversal result can spot-check any hop by querying that hop's neighbors
directly; the server's signature is just its reputation staked on the accuracy
of the computation, not a canonical truth. Multiple competing servers can
coexist, and clients route to whichever their own trust scores rank most
trustworthy or cheapest. This mirrors a rating agency in conventional credit
markets: useful for efficiency, discreditable if consistently wrong, and never a
single point of failure.

### Organizational Sustainability

Because the coordination server earns Quants — units redeemable for real work
from the network it serves — the operating organization can fund itself entirely
inside the system: infrastructure is offset by query fees, staff who maintain
the server earn spendable Quants, and the organization accrues a positive net
position as long as it delivers reliable service. It is the closest analog here
to a clearing house: a trusted intermediary earning a margin for reducing
friction, without holding deposits or issuing money.

---

## Summary

This is a **Peer-Verified Contribution Network**: time contributed to shared
projects, signed by the people you worked with. It records counterparty-signed,
time-denominated contributions, derives a Web of Trust from completed work, and
lets each participant price reliability locally from a public, socially
regulated record — with no money, no tokens-as-currency, and no central
authority. It does not compete with Bitcoin (a store of value and speculative
instrument); it serves a different purpose: letting qualified people collaborate
and recognize each other's contributions without fiat, while keeping a publicly
auditable reputation system that resists both Sybil attacks and centralized
capture.

A coordination server adds efficiency and gives the organizing entity a viable
funding path without compromising the trust model: it computes and aggregates,
but the network verifies — and the network prices the server's own reputation
like any other participant's.

---

## FAQ

See [the design FAQ](qw-design-faq.md) for open questions and the reasoning
behind each decision.

---

2026 Vladimir Krinitsyn
