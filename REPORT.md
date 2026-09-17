# Report — AmazonHelp Support Agent

Hiver SDE Intern take-home. Runnable pipeline and code: see [`README.md`](README.md). System design and mechanics: see [`ARCHITECTURE.md`](ARCHITECTURE.md). This document covers the six required report sections only.

## 1. Problem framing

**What "good" means for AmazonHelp specifically.** AmazonHelp replies happen on a public, high-visibility channel, at very high volume, and — looking at the actual historical replies in the corpus — almost never fully resolve an issue in-thread; they triage and redirect ("sorry to hear that, please DM us" / a support link). Given that, "good" for this system means, in priority order:

1. **Never send something unsafe or wrong on a public channel** — a bad auto-reply is worse than no reply, so precision matters more than coverage.
2. **Correctly identify the small set of high-stakes cases** (account security, money in dispute) that must reach a human, even at the cost of over-escalating some safe ones.
3. **Match the brand's actual first-touch style** — a short, honest triage/redirect, not an attempt at full resolution.
4. **Be honest about uncertainty** — an unmapped intent or an escalation is a visible "I don't know," which is better than a confident wrong guess.

**What I chose not to build**, and why:

- **Multi-turn handling.** Only the customer's first turn is ever classified or acted on. A live system has to decide before AmazonHelp has replied, so later turns are information that doesn't exist yet at the real decision point — building for them would overstate what's achievable live.
- **A model of resolution success.** There's no outcome/resolution label anywhere in the data, and no reliable proxy for one (thread length and final-reply sentiment are both too noisy to trust). I used AmazonHelp's own first reply to a similar past message as a stated, named approximation instead of inventing a shakier signal.
- **A fine-tuned or custom model.** Prompted `gpt-4o-mini` with the taxonomy and examples, rather than fine-tuning — sufficient for a 12-way classification task, and far cheaper to iterate on within the assignment's timeframe.
- **A live serving API.** The pipeline is batch-oriented notebooks, not a deployed service — the task was proving the approach works, not shipping it.
- **Full-corpus processing.** Classification is scoped to 5,000 of 44,654 conversations, per the assignment's own encouragement to subsample.
- **Non-English support.** Handled upstream, not by choice here — `dataprep.ipynb` already filters to English-only conversations.

## 2. Results vs. two baselines

Measured on the 150-row hand-labeled golden set (`golden_set.csv`), recomputed directly in `eval_harness.ipynb`:

| method | accuracy | how |
|---|---|---|
| **Trivial** | 12.0% | always predict the single most common intent in the golden set |
| **Simple** | 44.7% | keyword/regex rules, no model, first match wins across 12 patterns |
| **This system** | **89.3%** | `gpt-4o-mini`, prompted with the taxonomy + one example per intent |

The LLM beats the simple baseline by 45 points and the trivial one by 77 — but see Section 4: the trivial baseline's exact value is an artifact of how the golden set was sampled, not a stable number.

Escalation and reply quality have no equivalent "baseline" in the same sense (there's nothing simpler than a rule to compare a rule against), so those are reported directly: **70.7% escalation agreement** with human judgment, and an LLM-judge reply-quality score whose relationship to a human rating is covered in Section 4.

## 3. Failure analysis — top 5 failure modes

| # | failure mode | real example | hypothesis |
|---|---|---|---|
| 1 | **`order_status` / `delivery_problem` boundary confusion** | *"Gotta love going from 1 day shipping to 'yeah, hopefully by the weekend'"* → predicted `order_status`, gold `delivery_problem` | The model leans toward `order_status` whenever a shipping noun appears, under-weighting sarcasm/implicit-complaint cues that signal "problem" rather than "question" |
| 2 | **Keyword-triggered high-stakes false positives** | *"thank you for the quick resolution and response to my account being compromised"* (gratitude about a *resolved* issue) → predicted `account_access`, auto-escalated as if active | The classifier over-indexes on the word "account" appearing anywhere, regardless of whether the message describes an active threat, a resolved one, or a benign question |
| 3 | **`other` swallows classifiable messages** | *"who do I contact about a problem with a seller? ordered 100ml sent 50ml"* (gold: `refund_or_return`) → predicted `other` | When a message doesn't state its category in an obvious lexical way, the model defaults to the safe catch-all instead of reasoning through the 12 real options |
| 4 | **Escalation over-fires almost entirely from the confidence gate** | 39 of 42 over-escalations in the golden set are "confidence <75" on messages a human reads easily, e.g. *"Some days you have a great shopping experience... Others you consider cancelling Prime. #FirstWorldProblems"* | The model's self-reported confidence is a poor correctness proxy — it hedges on sarcasm, typos, or indirect phrasing even when the true category is unambiguous |
| 5 | **LLM judge doesn't track human-perceived quality** | Draft *"I'm really sorry for the unexpected charge! Let's get this sorted out. Please reach out to us via DM"* — human rated 5 (safe, correct, no over-promising), judge rated 3 | The judge seems to penalize brevity/genericness on its own, without recognizing that terse, non-committal deflection is often the *correct* AmazonHelp-style behavior |

## 4. What's misleading about my headline number

Three ways the numbers above overstate themselves if quoted alone:

- **"89.3% intent accuracy" blends two very different populations.** It's a weighted mix of 94.3% accuracy on the confidently-classified 59% of messages and 82.3% on the 41% the system itself flagged as uncertain. In production, that second group gets escalated regardless of whether it was actually right — so 89.3% isn't "how often the system is right," it's a number that only makes sense paired with the confidence split it's hiding.
- **The trivial baseline (12.0%) is an artifact of the sampling method, not a stable fact about the corpus.** The golden set was deliberately stratified to ~12 examples per intent so every category would be represented — which also makes "always guess the most common one" score far worse than it would on the real traffic distribution, where `delivery_problem` alone is 24.1% of conversations. The LLM's 77-point margin over trivial looks more dramatic here than it would against a realistically-skewed baseline.
- **"97.5% of judge scores land within one point of the human rating" sounds like agreement — the actual Spearman correlation is 0.26 and not statistically significant (p=0.107).** Both the judge and the human rater cluster most scores in a narrow 3–5 band, so a high within-1-point rate mostly reflects that neither uses the full scale, not that the judge is tracking real quality differences. Reporting only the 97.5% figure would be technically true and substantively misleading.

## 5. What I'd do next, with one more week

| day(s) | action |
|---|---|
| 1 | Sweep the 75% confidence threshold against the golden set to find where accuracy actually drops off, instead of keeping 75 as an untested guess |
| 1 | Add a precision guard on high-stakes intents so a bare keyword ("account") can't trigger escalation without corroborating content |
| 1–2 | Get an independent human pass over the 150-row golden set and the 40-row quality ratings — right now both were labeled by the assistant that also built the system being graded |
| 1 | Extend intent classification from 5,000 to the full 44,654 conversations — cheap in API cost, mostly wall-clock/rate-limit handling |
| 1 | Add a "customer already followed up / waited too long" escalation signal, which the golden set shows is a real, currently-unmodeled trigger |
| 1 | Re-run the full eval harness against all of the above and update every number in this report |

## 6. Decision log

Non-obvious calls made while building this, and why — a different reasonable engineer could have gone the other way on any of these.

| decision | why |
|---|---|
| Union-find over reply edges, not author-grouping | grouping by author silently splits a thread the moment a third party replies |
| Discard filters run cheapest-first | keeps the 4 discard files mutually exclusive; slow `langdetect` only runs on survivors |
| 24h gap cap, not a looser 7-day window | drops ~3% vs. ~0.6%, but guarantees every kept conversation is same-day |
| Discard multi-customer conversations entirely | once a third party joins, "the customer's problem" isn't well-defined |
| Classify only the first customer turn | later turns are info that doesn't exist yet at the real decision point |
| 75% confidence gate instead of always guessing | traded coverage for precision — unmapped is a visible "don't know" |
| Scoped to first 5,000 (oldest) conversations | pragmatic subsample; explicitly not representative of the corpus |
| Brand's first reply as the "resolution" proxy | no outcome label exists in the data — a named proxy, not a hidden assumption |
| Retrieval restricted to same intent bucket only | stops grounding a reply in a resolution for a different type of problem |
| Escalation is a fixed rule, not an LLM call | auditable, deterministic, always comes with a stated reason |
| `account_access`/`billing_or_payment` always escalate | money/security topics go to a human even at high confidence |
| Escalated tickets still get a draft reply | a human reviewing it still benefits from a starting point |
| Gold escalation policy applied to *gold* intent, not predicted | tests whether the decisions are right, not whether the code agrees with itself |
| Reported the judge's weak correlation, not just the flattering agreement % | both numbers are true; only one alone would have been honest |
| Excluded large regenerable files from the GitHub repo | rebuildable from `dataprep.ipynb`; keeping them would only bloat the repo |
