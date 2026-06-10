---
name: poc-security-review
description: >
  Adversarial, proof-driven security review. Hunts for exploitable bugs in a scoped codebase,
  proves each one with a graded proof-of-concept test suite (L1 isolated → L2 integration →
  L3 live), then independently self-audits every finding against fabrication, severity (CVSS),
  likelihood, reachability, and known-issue checks before writing a defensible report and a
  filable GitHub issue. Use for security audits, vulnerability hunts, bug-bounty / contest
  submissions, "review this module/pallet/contract for exploits", or any time you need
  PoC-backed findings you can defend to skeptical maintainers. Prove-or-discard: no passing
  PoC, no finding.
metadata:
  author: reverse-engineered from a real Substrate audit pipeline
  version: "0.1.0"
---

# poc-security-review

A security review that **earns** every finding. Anyone can list "potential issues." This skill
only reports a bug after it has written a test that makes the bug fail loudly, re-run that test
to confirm it, and survived an adversarial self-audit designed to kill plausible-but-wrong
findings. The deliverable is not an opinion — it is a reproducible artifact a maintainer can run.

## When to use

- "Audit / security-review this codebase / pallet / contract / module / service."
- "Find exploitable bugs in X and prove them."
- Preparing bug-bounty, Code4rena/Sherlock/Immunefi, or private-disclosure submissions.
- You have a candidate vulnerability and need a defensible PoC + severity writeup.

When the user just wants a quick correctness/style pass over a diff, use `/code-review` instead.
This skill is heavier: it produces tests and a full investigation per finding.

## The pipeline (3 role-phases)

This is designed to run as a **batch of agents over hours**, not a single pass. The three phases
map to three independent mindsets — keep them separate even when one agent plays all three,
because the assessor's job is to *distrust* the finder and the PoC-writer.

1. **Scope & find** — establish the target and threat model, then sweep for candidate bugs.
   Each candidate gets a stable id `issue-<unix-ms>-<kebab-slug>` and an `@qa-review` annotation
   at the suspect line so it can be traced. → `references/methodology.md`
2. **Prove (PoC-writer)** — for each candidate, write a graded PoC and escalate as far as the
   evidence allows. L1 proves the bug in isolation; L2 proves it survives the real verification
   chain; L3 proves it on a live system across the process boundary. → `references/poc-levels.md`.
   For **Substrate / FRAME / Subtensor** targets, the concrete test layers, commands, FRAME bug
   classes, and live-repro gotchas are in `references/substrate-frame.md` — read it first when the
   target is a pallet.
3. **Assess (adversarial assessor)** — independently re-run the PoC, then walk the verdict
   decision tree and write the full Investigation: severity + CVSS, likelihood, reachability
   traces, action-state differential, fabrication audit, known-issue search, poc-level
   validation. The assessor may **downgrade or discard** the finding.
   → `references/verdict-decision-tree.md`, `references/severity-and-likelihood.md`,
   `references/fabrication-audit.md`, `references/known-issue-search.md`

Only findings that exit the decision tree as **confirmed** become reports.

## Hard rules

1. **Prove-or-discard.** No finding ships without a PoC that is GREEN on current code (green =
   bug present). If you cannot write a test that demonstrates the harm, you do not have a finding —
   you have a hypothesis. Say so and move on.
2. **No fabrication.** Every mock, seeded balance, forcing flag, or shortcut in a PoC must
   correspond to a real production mechanism, and that correspondence must be stated. A shortcut
   used at L1 must be discharged by a higher level that removes it. Run the full fabrication audit
   (`references/fabrication-audit.md`) before confirming.
3. **Impact-first severity.** Rate by realistic worst-case impact, then apply likelihood and
   documented downgrade modifiers — never inflate. Attach a CVSS 4.0 vector and a maintainer-triage
   anchor. → `references/severity-and-likelihood.md`
4. **Honest poc-level.** Report the highest level *genuinely* reached. If L3 was blocked
   (infra missing, framework not actually driven), say exactly why and validate the level down.
5. **Defend it.** Findings are filed as issues and then challenged by maintainers. Write the
   repro so a skeptic can run it from a fresh state; anticipate environment gaps (feature flags,
   chain state, framework auto-revert semantics). → `references/report-templates.md`

## Output

Per confirmed finding, two artifacts (templates in `references/report-templates.md`):

- **Internal report** `issue-<id>.md` — frontmatter (id, title, status, severity, likelihood,
  cvss, poc-level, file, poc[]) + Description / Preconditions / Code Path / PoC / Investigation.
- **External issue** — concise GitHub-issue form (Describe the bug / To Reproduce with a runnable
  test / Expected behavior / Environment @ commit / Additional context + recommended fix).

## How to run it

For a small, targeted review you can run all three phases yourself, sequentially, one finding at
a time. For a real audit ("be thorough", "find everything"), fan the work out: many finders over
different subsystems, one PoC-writer per candidate, independent assessors that re-verify. See
`references/methodology.md` for the orchestration shape (and a Workflow sketch). Multi-agent
orchestration is opt-in and token-heavy — confirm scale with the user first.
