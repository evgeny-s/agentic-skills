# Report templates

Two deliverables per confirmed finding: the **internal report** (`issue-<id>.md`, the full audit
artifact) and the **external issue** (the concise, filable GitHub issue). Both are reverse-engineered
from a real Substrate audit pipeline.

Replace every `<...>`. Delete sections that genuinely do not apply, but prefer to keep a section
and write "N/A — <why>" (keep even empty modifier lists, because the *absence* is evidence of
diligence).

---

## A. Internal report — `issue-<id>.md`

```markdown
---
id: issue-<unix-ms>-<kebab-slug>
title: "<one-sentence what-breaks-and-how>"
status: confirmed            # confirmed | rejected | hypothesis
severity: <critical|high|medium|low|informational>
likelihood: <unlikely|possible|likely|almost-certain>
cvss: "CVSS:4.0/AV:.../..."
poc-level: <1|2|3>           # highest level genuinely reached (assessor-validated)
file: "<primary buggy file>"
docs:
  - "<knowledge/product or architecture doc consulted>"
poc:
  - "<path to L1 test>"
  - "<path to L2 test>"
  - "<path to L3 test>"
reasoning: >
  <2-6 sentence executive summary: what is confirmed, at what levels, why this severity,
  why not higher/lower, which CWE applies, maintainer-triage anchor.>
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
---

## Description
<Plain-language mechanism. Name the functions and line refs. Quote the smoking-gun code.
Explain how it surfaces in each surrounding mode. State the broken invariant in the design's
own words, and cite any doc-comment that promises the property the code violates.>

## Preconditions
<Bulleted list. Every condition the attack needs; note which are ordinary vs engineered.>

## Code Path
<Numbered steps from entry point to harm, with code snippets and `file:line`. Show where the
`?`/early-return/swallowed-error short-circuits before the compensating write. Include a concrete
walk-through with real numbers.>

## Scope Impact
<Which in-scope files; which entry point / call index; what related paths are NOT affected and why.>

## Suggested Fix
<One or more concrete directions, marked informational/not-part-of-the-issue.>

## POC
**Test files:**
- Level 1: `<path>` -- GREEN (n/n) | FAILED: <reason>
- Level 2: `<path>` -- GREEN (n/n) | ...
- Level 3: `<path>` -- GREEN | FAILED: <precise infra blocker>

**Result:** reproduced
**Highest level reached:** <n>   (must match `poc-level`)
**Infrastructure used at L3:** <node/endpoint/framework>
**Damage site proven:** <the exact site; note if escalated beyond the reported site>
**Attempts:** <n> (<breakdown per level and what failed>)

### Summary
<Per level: each test name, what it sets up, the asserted differential, GREEN/RED, run command,
and pasted output.>

### Escalation
<L1→L2→L3: what each level removed/added; which approaches failed and why.>

### Safeguard Table
| Check | Passes? | Why |
|---|---|---|
| <auth gate> | Yes | ... |
| <the buggy step> | YES — this is the bug | ... |
| <the skipped write> | SKIPPED | short-circuits via `?` |

### Attacker Profile
- Who / Privileges / Cost / Preconditions / Repeatability

### Known Issue Search
<what was searched, result>

## Investigation
### Verdict
confirmed
Decision-tree node: node 9 — <one-line why it survived every gate>

### Severity (impact-first)
- Tier: <...>
- CVSS: <vector>
- Basis: <ties tier to the proven differential; why not higher; why not lower>
- Maintainer-triage anchor: <"security advisory" | ...>

### Likelihood (independent)
- Level: <...>
- Named factors: attack vector / access-ladder / preconditions / capital / race-window /
  reversibility / public-exploit signal → combined band.

### Reachability
- Backward trace to the buggy line: <numbered>
- Forward trace past it (harm closes): <numbered>

### PoC Execution Evidence
- Commands run: <...>
- Outcome: <L1 n/n, L2 n/n, L3 ...>
- Action-state differential: Before / Trigger / After / Delta / **Invariant broken**.
- Blast-radius: <trigger shape; population estimate; evidence source>.
- Verdict: reproduced and invariant broken.

### Fabrication Audit
<All 22 items examined. List triggered items with justification or condemnation. Overall: honest.>

### Threat-model Audit
- Named attacker / concrete gain / concrete cost / vulnerable-deployment population /
  extrapolation tells (none, ideally).

### Downgrade Modifiers Applied
<Each modifier: applies / does not apply, with one-line why. Often "None.">

### poc-level Validation
- Claimed: <n> / Validated: <n> / Reason if downgraded.

### Scope Analysis
<In scope on both code and effect dimensions.>

### Reasoning
<The full argument: why confirmed, why this exact severity, the atomicity/root-cause analysis,
the fix requirement.>
```

---

## B. External issue (GitHub) — concise and filable

Keep it tight. Maintainers read this, not the 400-line internal artifact. Link line refs to the
**exact commit** (`/blob/<commit>/path#Lx-Ly`). Include a runnable test in *their* idiom.

```markdown
### Describe the bug

<2–4 sentences: the mechanism, the missing guard, the consequence. Link the affected lines at the
pinned commit. Quote the few load-bearing lines. If a doc-comment promises the violated property,
quote it. If it's a regression, name the commit that introduced it and note it was unmentioned in
the commit message.>

### To Reproduce

<Numbered steps, then a runnable test. Prefer the project's own test idiom (a Rust unit test, or a
TypeScript/Moonwall e2e test). Write it inverted (green = bug) and say so. Make it self-seed every
precondition — feature flags, balances, accounts — so a reviewer reproduces from a fresh state.>

```rust
#[test]
fn <inverted_test_green_means_bug>() { /* ... assert the harmful differential ... */ }
```

Observed: <the concrete before→after numbers>.

### Expected behavior

<The invariant that must hold after the function returns. The two acceptable fixes, briefly.>

### Screenshots
_No response_

### Environment

<repo> `<branch>` @ `<commit>`

### Additional context

- <impact bullet> ...
- **Recommended fix**: <one or two concrete options>.
```

### Defending it in the thread

Maintainers will try not to reproduce it. The winning moves are:
- When they "can't reproduce," ask exactly which **branch/commit** and which **environment** they
  ran (a fresh `--dev` had a feature flag off that testnet has on → the first leg never ran).
- Re-state that the test is **inverted** (green = bug) if they misread a pass as health.
- If a framework wrapper auto-reverts (ink! `Result` → `ReturnFlags::REVERT`), supply a variant
  that returns a **non-`Result`** type so the wrapper does not mask the partial commit.
- Paste your console showing the before→after deltas, and link a commit with the working repro
  that reproduces **from fresh state**.
- Be gracious and precise; the goal is a merged fix, not winning an argument.
```
