# QW Design Review — FAQ

Consolidated from a review of `qw/abstract.md` (Decentralized Skills Exchange
and Mutual Credit Network).
Covers platform selection, mobile architecture, data model, privacy, trust
mechanics, and legal framing.

---

## 1. Platform & Architecture

### Q: Is a blockchain needed?

A chain provides global ordering, consensus, and token transfer. None are
required here:

- Ordering is unnecessary — bilateral signatures make omission provable by
  production (see §3).
- Consensus is unnecessary — the design already accepts detection over
  prevention.
- Token transfer is unnecessary — the unit is time, not a tradeable asset.

Costs a chain would impose: mandatory gas token, App Store crypto review
exposure, seed-phrase onboarding, and immutability that conflicts with deletion
rights.

### Q: If forced to pick a non-PoW chain, which?

Ranked, though all are dispreferred:

| Platform | Note |
|---|---|
| NEAR | Best of category — meta-transactions (no user gas), passkey onboarding, human-readable accounts |
| Solana | Cheapest writes, real mobile stack; aggressively token-centric |
| Cosmos SDK app-chain | Could encode contract lifecycle directly; requires running a validator set |
| Algorand / Hedera | Cheap and enterprise-legible; centralized governance, no gain over signed events |
| Ethereum L2 | Worst mobile UX, no compensating benefit |

### Q: What is the recommended stack?

| Layer | Choice |
|---|---|
| Identity | `did:key` + `did:web`, device keys under a controller DID |
| Attestation | W3C Verifiable Credentials, SD-JWT selective disclosure |
| Storage / transport | Nostr relays (ATProto as migration target) |
| Discovery | Relay lists + BitTorrent mainline DHT |
| Mailbox | Encrypted envelopes on own relay |
| Mobile | Tauri v2 — Android, iOS, web from one codebase |
| Signing | External signer app (Amber pattern) |
| Backup | Personal server replica + friend-hosted chunks |

### Q: Why Nostr over ATProto?

Nostr is cheapest to operate solo, has no central registry dependency, treats
the key as the identity, and gives "all records referencing A" natively via tag
filters. Its weaknesses — unenforced schema, advisory deletion — are manageable
when you and your users operate the relays carrying your custom event kinds.

ATProto is the migration target if enforced schemas or a real search index
become binding constraints. Because payloads are W3C VCs either way,
attestations port between them.

### Q: Why not Holochain?

Holochain remains the only option offering protocol-enforced fork detection.
That property was set aside once detection-over-prevention was accepted.

Costs: small ecosystem, near-zero hiring pool, iOS support pending for roughly
two years, paused official Launcher, and p2p Shipyard being source-available
with a pricing page (a paid dependency in the build pipeline).

**Keep on the bench** for the scenario where enforced double-spend prevention
becomes non-negotiable.

### Q: Do I need my own Holochain fork or my own Volla Cloud?

**No fork. Yes to own always-on node service.**

Forking core means inheriting maintenance of a distributed-systems codebase and
losing upstream fixes — not viable solo. Volla Cloud Services can't be ported;
it's an Android system service requiring VollaOS.

The equivalent function assembles from stock parts: a conductor or relay on a
personal server, thin phone client over WSS, own push relay.

---

## 2. Mobile Clients

### Q: Is mobile battery use a problem?

With relays providing store-and-forward, the phone is a thin signing client:
hold a key, sign payloads, sync on wake. That's the Signal model.

Holochain reached the same conclusion independently — mobile nodes run in
"zero-arc" configuration and hold no DHT portion, explicitly to save battery and
meet app store requirements.

### Q: What does thin-client mode unlock?

- iOS stops being blocked — no WASM execution, no Wasmer/JIT problem, no App
  Store 2.5.2 exposure
- Mobile web app becomes viable — browsers can sign and talk to a node over WSS
- No VollaOS dependency for v1
- Battery becomes a non-issue

### Q: Is a mobile web app possible?

**Yes, with keys outside the browser.**

Browser storage is the wrong place for an identity key — XSS-exposed, and Safari
evicts data after roughly a week of non-use, which would mean silent permanent
identity loss.

Pattern: web app composes and displays; phone or node signs via QR/deep link.
Web app handles discovery, job board, public profiles, drafting.

### Q: What about Holo hosting for browser access?

**Not recommended.** Chaperone derives keys in-browser from username and
password, so hosts can't forge signatures — but:

- Key derivation requires a salt from a Holo-hosted registration service. Down
  means nobody logs in; wrong salt means a different identity.
- Password-derived keys have no hardware backing and are offline-guessable.
- The HoloPort still sees all zome calls in plaintext and can censor or serve
  stale state.

### Q: What are the remaining mobile problems?

| Issue | Severity |
|---|---|
| Device-key model is load-bearing, not optional | **High** — same key on two devices forks a chain |
| Someone must still be a real DHT/relay peer | Medium — personal + friend servers |
| Node availability = your availability | Medium — register with 2–3 relays |
| Push routes through FCM/APNs | Low — payload encrypted, metadata visible |
| Key custody on device | Medium — hardware keystore + social recovery |

### Q: What is the App Store risk?

Without a bundled conductor, executable-code objections disappear. **The only
remaining trigger is Shadow Quant** (ERC-20) under guideline 3.1.5(ii).

---

## 3. Data Model & Integrity

### Q: Does the ledger need hash-chaining or ordering?

Every contract is bilateral and both parties sign. The record someone would want
to hide lives on the counterparty's side, which they don't control. Omission is
provable by production, not by gap analysis.

### Q: Then what does omission-detection require?

| Requirement | Detail |
|---|---|
| Dual indexing | Every contract published independently under both pubkeys |
| Union query | Evaluation asks "all records referencing A", never A's self-report |
| Publication window T | Hours to a day, measured **from the counterparty's signature timestamp**, not from first publication |
| Automatic verification | Client checks at offer time and shows a plain signal — manual verification doesn't happen |

### Q: What is the actual attack to design against?

**Not chain-forking** — it's expensive, provable, and permanent. Since multiple
accounts and pseudonymity are permitted, the cheap attack is **abandoning the
identity and starting fresh**.

This inverts the goal: don't build on punishing bad actors (they exit), build on
making good history valuable and slow to accumulate.

- New account = unknown-risk, not neutral
- Limits scale with history depth and counterparty diversity, not score
- Path-to-you through the WoT matters more than any number — paths can't be
  forged by a fresh key

### Q: Is this a ledger at all?

Better modeled as **Verifiable Credentials**: issuer is the counterparty,
subject is the worker, claim is time and skill. Gives DIDs, mature mobile
wallets, and SD-JWT/BBS+ selective disclosure natively.

---

## 4. Privacy & Disclosure

### Q: Which fields are public vs encrypted?

| Field | Visibility | Why |
|---|---|---|
| Party pubkeys | Public | Union indexing, graph paths |
| Signature timestamps | Public | Enforces window T |
| Status | Public | The reputation signal |
| Skill tags | Public | Referral routing |
| Quant amount | Public or ranged | `net_position` unverifiable otherwise |
| Rate multipliers | Public or ranged | Price discovery |
| Deliverable, terms, attachments | Encrypted | Confidential |
| Client identity, sensitive sectors | Ranged or encrypted | Leaks by association |

### Q: Can a relay buffer messages tagged with a hash of the recipient?

**Yes, but not a static hash.** Pubkeys are public, so `H(recipient_pubkey)` is
rainbow-tableable in seconds and is a persistent identifier leaking message
counts, timing, and burst patterns.

| Scheme | Trade-off |
|---|---|
| Per-pair rotating tags — `HMAC(shared_secret, epoch)` | Cheap; relay can still cluster polled tags |
| Prefix bucketing (truncate to k bits) | Tunable anonymity set vs bandwidth |
| **Trial decryption, no tag** | Perfect privacy; **practical here** because DHTs are scoped per community |

Envelope: seal to recipient's X25519 key (`crypto_box_seal`), pad to fixed size
buckets, sender identity inside the plaintext.

### Q: How to stop relay spam without proof-of-work?

**Blind-signature tokens (Privacy Pass).** Recipient issues blindly-signed
tokens to contacts; relay verifies validity without learning which recipient
issued or which contact spent. Plus per-tag quotas, hard TTL (~72h), size caps.

Prices naturally in Quants — relays earn for storage served, spam costs
work-backed credit.

### Q: Does encrypted-mailbox delivery cover query routing?

**No.** Relays holding pending referral queries must read skill tags in
plaintext to route. Two different problems; don't assume solving one covers the
other.

---

## 5. Trust, Sybil & Disputes

### Q: Does the anonymity mixer break Sybil resistance?

**It would — so there is no mixer. Resolved.**

The cascade block traverses signature chains, and a mixer would exist only to
sever exactly those chains. A bot farm behind a mixer is unblockable short of
blacklisting the mixer itself, which also harms its legitimate users. Rather
than carry that contradiction, the design drops the mixer and keeps the
reputation signature chain.

Without a mixer, Sybil resistance holds by construction. Every account has to be
signed into the web of trust by someone, so behind any bot farm sits a limited
number of real signing accounts. Blocking those accounts — and sharing the proof
of contamination so others cascade the same block — takes down the entire farm
at its root.

Holding multiple accounts stays fine and does not reopen the hole: each account
maintains its reputation independently, and one with no real signers behind it
carries none. The protocol never requires a real name — only real,
counterparty-signed reputation.


### Q: How is pricing set, and does the mixer break it?

** pricing is a pure market, and there is no mixer to break it.**

Pricing isn't a hidden social score a mixer could obscure. Per abstract.md the
worker asks the price — declared hours × rate — and the recipient approves it
before work starts. That is a pure market bargain, recorded in the signed
contract.

Reputation is the market's verdict on that bargain: if a worker asks more than
the client feels they received for the price, the client's acceptance and rating
reflect it, and the worker's reputation falls. Every input — the asked price and
the counterparty's signature — is public and counterparty-signed, so the record
prices the worker directly. With the mixer cut (above), nothing hides any of it.

### Q: How are disputes handled?

Keep the record, attach annotations — never hide:

| Annotation | Signed by | Effect |
|---|---|---|
| Reply | Party being criticized | Visible alongside, no score effect |
| Audit request | Either party | Marks record "under review" |
| Audit opinion | Third-party auditor | Weight proportional to auditor's standing |

Design constraints:
- Auditors drawn from the intersection of both parties' WoT, or accepted by both
- Auditors stake reputation — the opinion attaches to their own record
- Auditors paid in Quants
- "Disputed, no audit" is a valid terminal state

This also closes the earlier hole where an unsigned completion left contracts in
limbo, and reduces the incentive to abandon an identity.

### Q: How does the contract lifecycle get signed?

| Step | Signed by | Atomic? |
|---|---|---|
| Offer | Client | No |
| Accept | Worker | No |
| Completion / acceptance | Each separately | No |
| Credit issuance | Both | Yes — countersign |

Only the last step needs atomicity. Everything prior tolerates one party being
offline, which is the mobile reality.

---

## 6. Discovery & Referrals

### Q: How does discovery work without a central directory?

Three-layer split:

1. **Bootstrap** — DNS, torrent DHT, public boards (Craigslist, HN, Telegram).
   Explicitly permitted; "no internet" means no hosted data store, not no
   discovery.
2. **Referral queries** — TTL-bounded propagation through contacts with
   per-contact relay policies.
3. **Public gateway** — indexable signed job postings with stable URLs.

### Q: Is TTL-bounded referral propagation better than graph traversal?

**Yes — significantly.** It collapses three components into one:

- Discovery becomes routing, not search — no index to build, host, or shard
- Skill sharing becomes each edge's cache of direct contacts' tags
- Trust is carried by the path itself

It also matches how people actually find work.

### Q: Does flooding scale?

**No — use greedy routing.** 50 contacts × 3 hops is up to 125,000 messages per
question (the Gnutella failure). Since each node already caches contacts' skill
tags, relays forward selectively toward tag-similar contacts. Small-world
routing reaches a match in ~logarithmic hops with fanout 2–3 instead of 50 —
same reach, ~1% of traffic.

**This is the single most important change to the referral design.**

### Q: What permission model?

Per-contact, not global:

| Setting | Range | Meaning |
|---|---|---|
| `relay_depth` | 0–3 | How far I pass queries onward |
| `accept_depth` | 0–3 | How far away a requester may be |
| `categories` | tag set | Only relay/answer in these areas |
| `rate_limit` | N/day | Ceiling per contact |
| `share_tags` | bool | May this contact cache my skill tags |

### Q: Who sees the query?

Middle path: reveal identity only to hop 1; each relay attaches its own vouch as
the query moves. Receiver sees "someone my contact Anna trusts, two hops out,
asking about Rust work." Responses return along the path. Dedup by pubkey but
keep path count — multiple independent paths is stronger signal.

### Q: Can relaying be incentivized?

Yes — charge a fractional Quant for relayed queries, or credit relays whose
match becomes a signed contract. Referral fees are normal in labor markets, and
it prices out spam.

### Q: Is a searchable directory feasible?

Search needs an index; an index needs a holder:

- Coordination server holds **Shard per community/guild** — most defensible, but yields federated search,
  not a global talent search, but regional
- Every node holds an index of the contracts he knows and able to forward request to another node or host

---

## 7. Economics & Positioning

### Q: Does "no inflation, bounded by human time" hold?

`Rate × ko × km` are unbounded subjective multipliers; rate creep is inflation
by another name. Recommend dropping the claim rather than defending it.

### Q: Is this "mutual credit"?

**Not in the legal sense.** If credits create no obligation, they're a
reciprocity signal, not credit. That's a strength — no debt instrument, no money
transmission, no securities question — but the title oversells. Consider
"reciprocity ledger" or "skill attestation network."

Cost: an unenforceable IOU has weak pull. Works in communities with repeated
interaction — another argument for launching inside one project ecosystem.

### Q: Why avoid "Web3" and on-chain framing?

2021–22 was peak on-chain reputation — soulbound tokens, decentralized identity,
on-chain badges. Nearly all stalled: credentials weren't demanded by employers,
tokens attracted speculators, compliance overhead crushed teams.

The damage is reputational. An investor hearing "portable work history,
on-chain, with a token" pattern-matches to a failed category instead of
evaluating the referral mechanic.

### Q: How is this different from that cohort?

| Dimension | 2021 on-chain reputation | QW |
|---|---|---|
| Who attests | Platform, DAO, or self | The counterparty |
| Attester's stake | None — minting is free | Their own record |
| When created | Separate act of issuance | Byproduct of job lifecycle |
| Unit | Badge, score, balance | 15 minutes at minimum qualification |
| Score shape | One global number | N subjective readings |
| What's evaluated | The number | The path — who vouched, how far |
| Token required | Yes | No |
| Privacy | Public forever | Selective disclosure |
| Deletion | Impossible | Possible on relay/PDS |

### Q: Which 2021 failure modes are avoided?

| Failure mode | Avoided? |
|---|---|
| Costless attestation | **Yes** — requires a real counterparty |
| Token overwhelmed product | **Yes**, if Shadow Quant is cut |
| Wallet/gas/seed onboarding | **Yes** — external signer |
| Public forever | **Yes** — field-level encryption |
| Fake global objectivity | **Yes** — no global score exists |
| No verifier demand | **Partly** — consumed inside the network |
| Cold start | **No — worse.** Badges could be backfilled; attestations can't |

### Q: What are the drawbacks of this approach?

| Choice | Cost |
|---|---|
| No global score | Nothing for a résumé; harder to market |
| Time as unit | Rate multipliers reintroduce subjectivity |
| Selective disclosure | Verifiers discount what they can't see |
| Detection, not enforcement | Walk-away risk (acknowledged in abstract) |
| Keypair = identity, multi-account | Abandonment is free |
| Mixer as first-class | Contradicts cascade block |

### Q: Is counterparty participation an adoption tax?

**Largely no — earlier objection withdrawn.** Both parties are already in
negotiation, often face-to-face or synchronously online. Each contract onboards
one person through an existing relationship — the Venmo/WhatsApp pattern.

What remains is **signature at completion**: the counterparty's benefit is
diffuse while the worker's is immediate. Threaded comments with review scores
mitigate this by giving the counterparty a record of their own conduct and a
place to raise quality concerns rather than silently withholding.

---

## 8. Legal & Compliance

### Q: Does the co-authorship framing avoid barter classification?

**Holds under two conditions — Risk, needs professional review.**

Works when:
1. Work is on a **declared open-source project** — output is public and
   non-appropriable
2. **No project is controlled by the counterparty** in a way that privatizes the
   benefit

The structure is: A contributes to a project led by B; separately, B contributes
to a project led by A. Neither is a service recipient. This is materially
different from reciprocal services, and is what millions of open-source
contributors already do without tax events.

Fails for direct bilateral work-for-work with a contemporaneous record.

### Q: Where does the abstract weaken its own position?

| Feature | Effect |
|---|---|
| Shadow Quant (ERC-20 mirror, fiat off-ramp) | **Creates a market price. Ends the argument.** |
| "Private with no profit" projects | Weakest clause — "no profit yet" isn't "no value" |
| "Removes the barter classification" | Stated as settled; invites testing |

### Q: Recommended edits?

- Replace "removes the barter classification" with the reasoning itself
- Cut "private with no profit"; scope to declared open source only
- State the boundary explicitly — activity beyond these conditions is the
  participants' own tax responsibility
- Obtain a written opinion from a tax attorney before any data room

### Q: What about deletion rights?

Records are work history about identifiable people; EU and several US state laws
grant deletion rights. Immutable public chains cannot comply. Nostr (advisory)
and ATProto (real, at the PDS) can — document the reasoning that custom event
kinds aren't mirrored by general-purpose relays.

### Q: What should be cut?

**Shadow Quant.** One feature simultaneously fixes:
- App Store review exposure (3.1.5(ii))
- 2021 pattern-matching in investor conversations
- Tax position (creates ascertainable fair market value)

Surviving monetization: chain calculation, rating bureau, vault storage,
community insurance. None create transferability.
