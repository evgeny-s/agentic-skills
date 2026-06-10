# Methodology: scope → find → prove → assess → report

This is the operating manual for the three role-phases. It is domain-general; the worked examples
come from a Substrate/FRAME blockchain audit (limit-orders / staking pallets), but the structure
applies to any codebase with a clear trust boundary — web backends, smart contracts, parsers,
auth systems, RPC services.

## Phase 0 — Establish the target and knowledge base

Before hunting, write down (or read, if it already exists) a small knowledge base. The original
pipeline kept this under `.qa/`:

- **`targets.md`** — the scope. Which packages/modules/files are in-scope; what the focus areas
  are; what is explicitly out-of-scope. Findings outside scope are discarded by the decision tree.
- **`knowledge/product/`** — what the system is *supposed* to do, in plain language: the user
  flows, the invariants the design promises ("a signed order debits at most `amount` once",
  "the two legs are atomic"). Most bugs are a *broken promise*; you cannot spot a broken promise
  without first recording the promise.
- **`knowledge/architecture/`** — how it is built: the trust boundaries, the entry points
  (extrinsics / routes / handlers / public ABI), the storage model, what runs without an outer
  transaction wrap.

Also pin the exact revision under review (commit hash / branch). Every line reference and every
issue's `Environment` field cites it.

## Phase 1 — Find (the finder mindset)

Sweep the in-scope code for candidate vulnerabilities. Productive lenses (each is a parallel
search angle — a finder blind to the others will miss things, so run several):

- **Atomicity / partial-commit.** A multi-step operation where step A mutates persistent state
  and step B can fail, with no transaction wrapping the pair. Look for: the outer function is
  *not* wrapped (`#[transactional]` / `with_storage_layer` / DB transaction / try-finally) while
  an inner step commits to the parent layer; `?`/early-return between a write and a compensating
  write; `let _ = ...` / `if let Ok(..)` that swallows an error after a prior write.
- **Missing dedup / replay.** The same authorized object (signed order, nonce, request id)
  accepted N times because validation reads a pre-batch snapshot and the de-dup write happens
  only later, or never.
- **Front-run / pre-claim of a deterministic resource.** A protocol-reserved, publicly-derivable
  identifier (deterministic account, well-known address, predictable id) that an unprivileged
  actor can claim *before* the system claims it, especially if the registration is a silent
  no-op-on-exists followed by an unconditional "migration done" flag.
- **Silent accounting divergence.** On-chain/real balance moves but the internal bookkeeping
  counter does not (or vice-versa), and no invariant check catches it because the two *issuance*
  counters stay in sync while a *third* derived counter drifts.
- **Over-/under-reduction of shared aggregates.** A decrement that subtracts more than this
  caller contributed to a pool shared by others.
- **Doc/code mismatch.** A doc-comment that promises a property the code no longer provides is a
  strong tell — and doubles as evidence the behavior is a bug, not a design choice (e.g. a comment
  still promising atomicity "without leaving residual stake if the second leg fails").
- **Regression via blame.** `git log -p`/`git blame` on the suspect lines: a commit that removed
  a `with_transaction` wrapper "while doing something else", with no mention in the commit message,
  is a classic silent regression.

For each candidate, mint a stable id: `issue-<unix-ms>-<kebab-slug>` (e.g.
`issue-1730000000000-fee-failure-skips-order-insert`). Drop an annotation at the
suspect line so later phases (and humans) can find it:
`// @qa-review: finding — issue-<id> <one-line>`. Use `@qa-review: reviewed` for lines you checked
and cleared, `@qa-review: todo` for concerns you could not yet resolve.

A candidate is just a hypothesis until Phase 2 proves it. Do not assign severity yet.

## Phase 2 — Prove (the PoC-writer mindset)

Write a graded proof-of-concept and escalate it as far as the evidence honestly allows. The full
level definitions, escalation rules, and self-validation are in `poc-levels.md`. In short:

- **L1** — bug exists in isolation (unit test against a mock/in-memory harness).
- **L2** — bug survives the real verification chain (integration test against the real runtime:
  real auth, real balances, real storage/transaction semantics — every L1 shortcut removed).
- **L3** — bug fires on a live system driven across the process boundary by an external
  framework (e.g. a TypeScript test driving a live node over RPC).

Tests are written **inverted**: GREEN means the bug is present. State this in the test name and a
comment, because a maintainer who reads it must understand a passing test is bad news, not health.
After the fix lands, the same test goes RED — which becomes the project's regression test.

Record, per finding: the test file paths, the highest level reached, the exact run command and its
output, and the number of attempts/approaches tried at each level (honesty about what did not
work matters for the assessor).

## Phase 3 — Assess (the adversarial assessor mindset)

The assessor's job is to *break* the finding. Do not trust the finder's or the PoC-writer's
conclusions — independently re-run the PoC and verify every claim against source. Then:

1. Walk the **verdict decision tree** (`verdict-decision-tree.md`). It gates on: in-scope?
   known/accepted already? requires an excluded actor? PoC executable? PoC fabricated?
   reachable end-to-end? above the impact threshold? Most rejected findings die here.
2. Assign **severity impact-first**, attach a **CVSS 4.0** vector, and a **maintainer-triage
   anchor**; assign **likelihood independently**; apply **downgrade modifiers**.
   (`severity-and-likelihood.md`)
3. Run the **fabrication audit** — the 22-item checklist that catches PoCs which "prove" the bug
   only because a mock was rigged to produce the result. (`fabrication-audit.md`)
4. Write the **reachability traces** (backward to the buggy line, forward past it to a closed
   end-to-end harm), the **action-state differential** (before / trigger / after / delta /
   invariant broken), and the **blast-radius** estimate.
5. **Validate poc-level** — confirm or downgrade the level actually achieved, with the reason.
6. Run the **known-issue search** (`known-issue-search.md`) — git log, issues/PRs, advisories,
   SECURITY.md, changelog, code comments. A bug already fixed or accepted is not a finding.

Only a finding that exits the tree as **confirmed** is written up.

## Orchestration (how the "6-hour batch" is shaped)

Sequential, single-agent runs are fine for one or two suspected bugs. For a real audit, fan out —
but **multi-agent orchestration is opt-in and expensive; confirm scale with the user first.**

A natural shape (pipeline by candidate, so each bug flows find → prove → assess without waiting on
the others):

- **Fan-out finders** over disjoint subsystems / lenses; dedup the candidate list.
- **One PoC-writer per candidate** (isolate in a git worktree if they mutate test files in
  parallel), escalating L1→L2→L3.
- **Independent assessors** that re-verify — ideally an assessor that did *not* write the PoC.
  For high-stakes findings, use a small panel (e.g. 3 assessors with distinct lenses:
  reachability, fabrication, severity) and require a majority to confirm.
- A final **completeness critic**: "what subsystem, entry point, or failure mode did no finder
  cover?" — that becomes the next round. Loop until a round surfaces nothing new.

If the user opts into a Workflow, the canonical pattern is `pipeline(candidates, prove, assess)`
with a barrier only where you genuinely need all findings at once (e.g. dedup before writing
issues). Log anything you cap (top-N, no-retry) so "covered everything" is never implied falsely.
