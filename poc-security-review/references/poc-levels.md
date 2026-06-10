# Graded proof-of-concept: L1 / L2 / L3

The grade of a PoC is *how much real machinery the bug survives*, not how convincing the prose is.
A bug that only reproduces against a mock you rigged is weak; a bug that reproduces on a live
system driven through its external interface is strong. Escalate as far as the evidence honestly
allows, and report the highest level **genuinely** reached.

The three levels are domain-general. The Substrate column is the original worked example; the
generic column shows how to map the idea onto any stack.

| Level | Question it answers | Substrate/FRAME example | Generic mapping |
|---|---|---|---|
| **L1** | Does the bug exist *in isolation*? | `cargo test` against the pallet **mock runtime** (`mock.rs`, `thread_local!` ledgers, `MockSwap`) | Unit test against in-memory fakes / stubs / a test double of the dependency |
| **L2** | Does it survive the **real verification chain**? | `cargo test -p <runtime>` integration test using **real pallet-balances, real sr25519, real FRAME storage layer + auto-wrap, real staking** | Integration test against the real components: real DB/transactions, real auth, real serialization — every L1 stub removed |
| **L3** | Does it fire on a **live system across the process boundary**? | TypeScript **Moonwall + Vitest / @polkadot/api** driving a live `--dev` node over JSON-RPC (`context.createBlock()`) | End-to-end test driving the deployed/live service through its real external interface (HTTP, RPC, CLI, on-chain tx) via an external framework |

## What each level must establish

### L1 — bug exists in isolation
- Reproduce the harm against the cheapest harness that still exercises the buggy code path.
- Shortcuts are allowed *here* (forcing flags, seeded in-memory balances, a mock that returns
  `Err` on demand) **provided** each maps to a real production mechanism and you say so. E.g. a
  `FAIL_FEE_TRANSFER=true` flag stands in for "signer's balance < fee after the swap"; a
  `SIMULATE_SINGLE_OWNER` mode models real "first-writer-wins" account creation.
- Include a **control test**: the same path under the safe condition (e.g. `should_fail=true`)
  must *not* exhibit the harm. The contrast is what proves the bug is the bug and not your setup.

### L2 — survives the real verification chain
- Remove every L1 shortcut that the assessor could call a fabrication. Use the real authentication
  (real signatures, not a stubbed verifier), the real balance/state subsystem, the real
  transaction/rollback semantics.
- This is where "the mock lied" findings die. If the bug only existed because the mock's ledger
  lived outside the real transaction layer, L2 exposes it. If the bug is real, L2 reproduces the
  *same* before/after differential through real machinery — that is the strongest in-process proof.
- Walk the full **verification chain** explicitly and show each gate passes: auth → input
  validation → preconditions → the buggy step → commit. (See the "Safeguard Table" in the report
  template.)

### L3 — live system across the process boundary
- Drive the *running* system through its real external interface using the project's declared
  end-to-end framework. The point is to cross the process boundary: your test talks to the system
  the way a real attacker would, observing only externally-visible state (balances, events,
  responses).
- L3 is frequently *blocked by infrastructure*, and that is fine — report it honestly. Real
  examples: the live node binary did not include the pallet in its WASM (`api.tx.limitOrders`
  undefined); a port collision prevented the test framework from launching its own node so the
  logic was run as a standalone script. Neither is a passing L3.

## Escalation & honest level reporting (`poc-level`)

- `poc-level` is the highest level **genuinely** reached, validated by the assessor.
- Reaching a level means the test ran and demonstrated the harm *through that level's machinery*.
  A structurally-correct L3 file that could not execute is **not** an L3 pass — the level caps at
  L2 (or wherever execution last succeeded). State the blocker precisely.
- The assessor **downgrades** when the claimed level was not truly achieved. Real downgrade 3→2:
  the L3 logic was executed as a standalone Node script against the live node rather than through
  the declared Moonwall dev foundation — "the process boundary was crossed, but not through the
  declared framework," so it is Moonwall-adjacent manual validation, not a formal L3 pass.
- Record **attempts**: how many approaches were tried at each level and why they failed
  (nonce collisions, wrong env name, rate limits, a disabled feature flag). This is evidence of
  diligence and helps the maintainer reproduce.

## Inverted tests (green = bug)

Write PoCs so that **passing = bug present**. Name them accordingly
(`l1_t1_buy_alpha_state_committed_when_forward_fee_fails`) and comment it at the top, because a
maintainer reading a green test must understand it is reporting a problem, not health.
After the fix, the test flips to RED and becomes the project's regression test — so leave it in a
shape the project can keep.

## Defending the repro (lessons from real maintainer pushback)

Maintainers will try to *not* reproduce it. Pre-empt the common gaps:

- **Feature flags / chain state differ by environment.** A fresh `--dev` node had
  `SubtokenEnabled[0] = false` while testnet/mainnet have it `true`, so the first leg never ran and
  there was nothing to roll back. Fix: set the flag in `beforeAll` and prove it works *from a fresh
  state*. Always make the repro self-seed every precondition.
- **Framework auto-revert masks the bug.** ink! 5.x auto-sets `ReturnFlags::REVERT` on
  `Result::Err`, so a chain-extension message returning `Result` reverts and hides a non-atomic
  pallet bug. The repro must use a message that returns a **non-`Result`** type so the framework's
  revert wrapper does not trip — then the partial commit is visible. The lesson generalizes: make
  sure no wrapper *above* the bug is silently saving you.
- **Run on the right branch/commit at the stated revision**, and paste the console output (the
  before→after numbers) so the reader can diff against their own run.
