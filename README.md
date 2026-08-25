# Engineering Case Studies

Real engineering practice pulled out of my own projects — not polished writeups after the fact, but the actual artifacts a real workflow produces.

## Contract-vs-Code Audit

**[`contract-audit-case-study.md`](./contract-audit-case-study.md)**

A design document, a README, an ADR — anything written before or early in implementation — drifts from the code the moment nobody is enforcing the link. Six months later the person who wrote it can't answer questions about their own product, and the documents that should help are quietly lying.

This is a full audit report from **[trade-copilot](https://github.com/wTHU1Ew/trade-copilot)**, a personal trading-assistant project (Go backend, React frontend, OKX exchange integration, local-first architecture) — one round of a repeatable practice I run against that codebase: read every contract document (PRD, ADRs, architecture docs, security review, error-code registry) against the actual implementation, and report every place they disagree.

What the methodology looks for, specifically:

- **Four-way classification, not two.** Every mismatch is sorted into: code contradicts the contract (a real bug or an undocumented decision), the contract lags the code (deliberate evolution nobody wrote back), the contract is silent (a whole module/endpoint/convention the docs never mention — usually the largest bucket), or the contract is dead (describes something that no longer exists — dangerous in a customer conversation).
- **Evidence or it doesn't count.** Every finding carries a file path, a line number, and the actual lines that demonstrate it — for both the code side and the contract side. A claim without a citation doesn't go in the report, full stop.
- **Six sweeps beyond line-by-line comparison** — config keys nobody consumes, tests that copy the implementation's own constant instead of asserting an independent expectation, "all requests are rate-limited"-style universal claims checked against every real exit path (not just the obvious ones), values owned by an external vendor (verified against upstream docs, not just internal consistency), what a derived value is *actually* computed from vs. what the contract claims it's computed from, and decisions superseded by a later one but never marked as such.
- **Confidence is graded, and downgraded on purpose** — findings that depend on dynamic dispatch, runtime config, cross-language boundaries, or concurrency timing get flagged as harder to trust from static reading alone, with the reasoning spelled out.
- **Reconciled against its own history.** Every report diffs against the previous one: what got fixed, what's still open, what wasn't rechecked this round — so the audit trail compounds instead of resetting every time.

This round: 7 parallel slices, each blind to the others' findings and to prior audit history (so a finding surviving independent re-derivation is worth more than one carried forward on trust), reconciled into one report — 30 findings across all four categories, several independently re-verified against upstream sources (exchange API docs, protocol specs) rather than left as "needs manual check."

Several of the report's own findings — including one in a document *this same audit process* had written the round before — are addressed in the commits that followed it, in the project's own history.
