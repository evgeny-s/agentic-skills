# Fabrication audit (22-item checklist)

The most dangerous failure mode of a PoC is that it "proves" a bug only because the harness was
rigged to produce the result. A test that asserts `2 == 2` after a mock was told to return `2` is
not evidence. The fabrication audit is the assessor's defense against this: walk **all 22 items**,
mark each `clean` / `triggered`, and for every triggered item either **justify** it (the shortcut
maps to a real production mechanism *and* is discharged by a higher PoC level) or **condemn** it
(the finding is fabricated → discard or downgrade).

Core principle: **every deviation from production must (a) correspond to a real mechanism and
(b) be named.** A shortcut at L1 is acceptable only if L2/L3 removes it and reproduces the same
differential. State the correspondence explicitly — "mock TAO map = pallet-balances free balance;
mock alpha map = `Stake` storage."

The audit runs against the strongest PoC level the finding reached. Record results in the
Investigation section: which items triggered, the justification or condemnation for each, and the
overall verdict (`honest` / `fabricated`).

## A. State seeding & setup (items 1–5)

1. **Invariant checkpoint uses the real mechanism.** The storage/value you read to prove the
   harm is the *actual* state under test (e.g. real `Orders::get(id)` inside the transaction
   layer), not a side-variable you control. — *Triggered when the checkpoint is a mock field.*
2. **Seeded balances/ledgers map to real storage.** Pre-seeded accounts, balances, stakes
   correspond to what a real user would hold, at realistic magnitudes. An L1 `thread_local!`
   ledger living *outside* the real transaction layer is the classic shortcut — justified only if
   L2 uses the real balance subsystem and reproduces the differential. — *Triggered when seeding
   sets up the conclusion.*
3. **Seeded state is reachable in production.** The starting state could actually be produced by
   normal operation (a chain that added the pallet post-genesis; an account funded normally), not
   an impossible configuration. — *Triggered when the precondition cannot occur in prod.*
4. **Magnitudes are realistic.** Amounts clear real minimums (min-stake, existential deposit,
   gas) rather than being tuned to dodge a guard. — *Triggered when values are gamed.*
5. **No precondition silently disables a guard.** Setup does not flip a feature flag or config
   that removes the very check whose absence you are claiming. (Inverse also matters: if prod has
   a flag *off* that you turned *on*, the bug may not exist in prod — see the live-repro gotchas.)

## B. Identity & authorization (items 6–9)

6. **Authorization uses real credentials.** Signatures are produced and verified by the real
   primitive (real sr25519 / real JWT / real session), not a stubbed `verify() -> true`. Origins
   are the real origin type. — *Triggered when auth is mocked; justified only if a control proves
   the real auth path is exercised.*
7. **The attacker's privilege is honestly modeled.** If you claim "any unprivileged user," the
   PoC's caller is unprivileged — not secretly root/admin/sudo. Test-only `sudo`/`root` used for
   *setup* (registering a subnet, funding) is fine and must be flagged as setup, not as the attack.
8. **No impersonation shortcut.** The PoC does not sign as the victim or reuse the victim's key to
   manufacture an authorization the attacker could not obtain.
9. **The trust boundary is the real one.** You are attacking across the boundary that exists in
   deployment (external network participant → contract), not an internal API that no real caller
   reaches.

## C. Dependency & mock behavior (items 10–13)

10. **Forcing flags / error injection correspond to a real failure mode.** A `FAIL_FEE_TRANSFER`
    flag or a mock returning `Err` must stand in for a condition that genuinely occurs in prod
    (balance < fee → `FundsUnavailable`). Justified only when a higher level reproduces the
    failure *natively* through real arithmetic. — *Triggered always when a forcing flag is used;
    the question is whether it is justified.*
11. **Mocked dependency returns production-faithful values.** Prices, swap outputs, oracle reads
    are within realistic ranges and do not themselves cause the harm (e.g. a price feed irrelevant
    to the path because the limit is `u64::MAX`).
12. **No mock hides a guard that prod has.** The mock does not omit a validation/limit that the
    real dependency enforces and that would block the attack.
13. **The mock cannot roll back what prod would.** Be explicit when an external mock ledger is not
    inside the real transaction layer, so it cannot model rollback — this is exactly why such a
    finding must be re-proven at L2.

## D. Assertions & measurement (items 14–17)

14. **Assertions are non-vacuous.** They would *fail* on correct code. The PoC includes (or the
    report references) the fixed-code RED behavior, or a control test showing the safe path.
15. **The measured differential is the claimed harm.** "Signer debited 2× cap" is measured as an
    actual before→after balance delta, not inferred from an event or a log line.
16. **No off-by-one or unit confusion** inflating the result (RAO vs TAO, wei vs ether, ms vs s).
17. **Events/logs are corroborating, not load-bearing.** The proof rests on real state deltas;
    emitted events are supporting evidence, not the sole basis.

## E. Source & build integrity (items 18–20)

18. **The code under test is unmodified.** The PoC does not patch, comment out, or weaken the
    source to make the bug appear. (If you changed anything in `src/`, the finding is void.)
19. **Built at the stated revision** with normal build settings; no `SKIP_WASM_BUILD`-style
    shortcut that changes *what* is tested (it may change build speed only).
20. **The buggy line is actually executed.** Confirm via the trace / a log / coverage that control
    reached the line you blame — not a sibling branch.

## F. Environment & reproducibility (items 21–22)

21. **Reproducible from a fresh state.** A clean checkout + documented setup reproduces it; the
    repro self-seeds every precondition (no reliance on leftover local state — the fresh-state lesson).
22. **No framework wrapper above the bug is silently saving or faking it.** Confirm no
    auto-revert / auto-retry / outer transaction above the tested unit is either hiding the bug
    (so you must bypass it to see it — ink! `Result` auto-revert) or *creating* the appearance of a
    bug that prod would not have.

## Verdict

- **Honest** — all triggered items justified with stated production correspondences, and the L2/L3
  level discharges any L1 shortcut. Proceed.
- **Fabricated** — any triggered item is unjustified (auth faked, source patched, vacuous
  assertion, guard disabled, magnitude gamed). Discard the finding, or if only the *level* is
  inflated, downgrade poc-level and re-assess severity on the honest evidence.
