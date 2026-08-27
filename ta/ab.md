# There Is No Best Model. There Is Only Your Label Quality.

**What a five-way A/B on REST API anomaly detection taught me — and why I applied the same standard to our own product page.**

Every conversation about AI for API security starts in the same place: *which model is best?* Isolation Forest or an autoencoder? GBM or a fine-tuned transformer?

We ran that comparison properly — six candidate architectures, scored on the same data slice, time-based split, shadow mode, no cherry-picked windows. The answer was not a winner. The answer was that the question is malformed.

Three findings, and I think all three generalize well beyond our stack.

---

## 1. A model leaderboard is a verdict on your labels, not on the models

Everyone asks "how long until the model is good?" as if it were a calendar question. It isn't. A supervised model does not become good after N weeks. It becomes good after it has ingested enough *clean, verified* labels. "Time to learn" is a data-quality curve wearing a calendar's clothes.

That single reframe reorders the leaderboard:

- Unsupervised detectors (Isolation Forest, Half-Space Trees) are usable on tens of thousands of normal samples with zero attack labels. They are the honest day-one baseline, and they plateau early and low.
- GBM needs a few thousand verified attack samples per class, and it learns your label noise directly and enthusiastically.
- A fine-tuned transformer on tokenized HTTP has the highest ceiling on payload work and the slowest bootstrap. Mislabeled injections teach it the wrong semantics, permanently.

So: with dirty or scarce labels, unsupervised wins. With large, clean labeled sets, GBM wins, and then the transformer overtakes it. Same models, same code, different data maturity, different champion.

The operational consequence is that the A/B is not a one-time bake-off you run before procurement. It is a recurring measurement you re-run every time label quality moves. We re-ranked our own candidates purely by cleaning labels. The winners changed.

There is a dataset caveat that deserves more airtime than it gets. The public IDS corpora most teams reach for — CIC-IDS2017, CSE-CIC-IDS2018, UNSW-NB15 — are predominantly flow-level. They are fine for volumetric and protocol anomalies and weak for payload work. Payload and injection detection needs CSIC-2010 plus synthetic OWASP-style generation, and in every case you need your own replayed traffic. Label quality, not model choice, is the primary limiter on real-world performance. Anyone who tells you otherwise is selling a leaderboard.

## 2. "REST API anomaly detection" is five different problems

This is the finding that changed our architecture. There is no single anomaly class. There are at least five, and each carries a structurally different signal:

- **Volumetric / behavioral** (credential stuffing, scraping) — signal is rate, sequence, session shape.
- **Payload / injection** (SQLi, XSS, path traversal, SSRF) — signal is request body and URI token structure.
- **Protocol / schema** (malformed methods, parameter pollution) — signal is structural deviation.
- **Contextual / authz** (BOLA, IDOR) — signal is per-identity baseline drift.
- **Low-and-slow recon** — signal only exists across long time windows.

A model that is excellent at one of these is mediocre-to-useless at another, because the signal lives in a different representation. Detection quality collapses off-domain. So we stopped seeking one model and paired a specialist to each class: GBM on rate and session features for volumetric; a fine-tuned transformer for payload; schema validation plus Isolation Forest for protocol; a GNN over the identity → endpoint → resource graph for BOLA/IDOR, because that anomaly is relational and invisible per-request; LSTM/TCN over long session windows for low-and-slow.

Which means the "A/B test" is really five parallel A/B tests, and no cross-class leaderboard is valid. High-severity enforcement is then gated on ensemble consensus rather than any single specialist firing.

One methodology note, since this is where most published results quietly fall apart: under a sub-1% attack base rate, accuracy is a meaningless metric. Score on PR-AUC and cost-weighted detection. Split by time, never at random — random splits inflate every model equally and are therefore useless for selection. Promote a challenger only when it beats the incumbent on *held-out novel attack families*, not on the families it was trained against.

## 3. The real gap is not detection. It's the binding.

Here is the market observation I keep coming back to. Mature products exist for detection. Mature products exist for enforcement. What does not exist off the shelf is the binding between them: dynamic model switching tied to graduated enforcement levels — a paranoia dial where the detection posture and the policy response move together, per tenant, per threat class, per time of day.

Today, every serious team builds that binding by hand. That is a genuine gap, and it is where I think the next durable control point sits.

---

## The sub-line: I ran the same standard against our own site

We build Atria — a zero-trust policy gateway for cross-organizational AI agents. Someone else's AI agent wants access to your API; Atria is how you say yes. We just put the site and the security page through an outside evaluation, and I want to share what the reviewer flagged, because the discipline is identical to the one above.

What held up: leading with the buyer's problem instead of abstract zero-trust language. Showing the actual request flow — identity resolution, revocation check, policy match, forward, audit write. Publishing the honest performance decomposition, separating the microsecond decision cost from the extra-hop and TLS costs. And, most importantly, a security page that names its own limitations out loud: fail-open revocation, an unauthenticated admin endpoint, no hash-chaining yet, no gRPC, no certifications.

What we're fixing: the homepage says revocations sync from Atria Cloud, but the stale-list behavior when the cloud is unreachable belongs on the homepage, not buried in the security page. It's an architectural trade-off, not a footnote. "Append-only log" reads as tamper-proof to a security reviewer; it's becoming "append-only local audit log," with hash-chaining explicitly marked Phase 2. The compliance prose becomes a scannable table: control concern, mechanism, evidence produced, current limitation. And we're stating in one sentence why an existing API gateway, service mesh, or cloud IAM layer is insufficient for third-party autonomous agents, because that objection arrives in the first ninety seconds of every call.

The through-line between the model work and the product work is the same principle: **publish the conditions under which you lose.** A leaderboard that doesn't disclose its label quality is marketing. A security page that doesn't disclose its failure modes is marketing. Neither survives contact with a serious evaluator, and the serious evaluators are the only ones worth winning.

We're taking on three closed-alpha design partners for external agent access control. If you're running APIs that third-party agents are already calling — or will be by Q1 — I'd like to hear how you're handling it today.

*Charts, model comparison tables, and the full threat-class-to-model mapping are in the report excerpts above.*
