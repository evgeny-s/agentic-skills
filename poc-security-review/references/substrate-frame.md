# Substrate / FRAME / Subtensor playbook

First-class guidance for auditing Substrate pallets (a Subtensor-style runtime is the reference
target). Use this together with `poc-levels.md` (the level model) and
`fabrication-audit.md` (the discipline). The generic skill maps directly here:

| Generic level | Substrate concretization | Command |
|---|---|---|
| **L1** isolated | `cargo test` against the pallet **mock runtime** (`src/mock.rs`, `thread_local!` ledgers, `MockSwap`, forcing flags) | `cargo test -p <pallet-crate> --lib <test_prefix>` |
| **L2** integration | `runtime/tests/*.rs` against the **production runtime**: real `pallet-balances`, real sr25519 (`sp_keyring`), real FRAME storage layer + `#[pallet::call]` auto-wrap, real staking | `SKIP_WASM_BUILD=1 cargo test -p node-subtensor-runtime --test <file> <prefix>` |
| **L3** live | `ts-tests/suites/...` TypeScript **Moonwall + Vitest / @polkadot/api** against a live `node-subtensor --dev` over JSON-RPC (`context.createBlock()`) | `cd ts-tests && CI=true pnpm moonwall test dev <SUITE_ID>` |

`SKIP_WASM_BUILD=1` only speeds the build; it does not change *what* is tested, so it is fine at L2
(fabrication-audit item 19). Note: a live `--dev` binary may not contain a not-yet-shipped pallet
in its runtime WASM (`api.tx.<pallet>` is `undefined`) — that legitimately caps poc-level at 2.

## The `.qa/` knowledge base (recreate per target)

The pipeline keeps its scope and product knowledge under `.qa/`:
- `.qa/targets.md` — in-scope pallets/files, focus areas (e.g. "limit/advanced orders pallet").
- `.qa/knowledge/product/index.md`, `user-flows.md` — the promised invariants in plain language.
- `.qa/knowledge/architecture/index.md` — trust boundaries, entry points, storage model.
- `.qa/scripts/test.sh` — convenience runner: `.qa/scripts/test.sh <pallet> <test_prefix>` and
  `.qa/scripts/test.sh runtime <test_prefix>`.
- `@qa-review:` code annotations: `finding — issue-<id> <note>` (a candidate), `reviewed` (checked
  & cleared), `todo` (unresolved concern).

## FRAME-specific bug classes (high yield)

These are the highest-yield paths — check each for them:

1. **Missing `#[transactional]` / `with_storage_layer` on a multi-step body.** FRAME auto-wraps
   every `#[pallet::call]` in `with_storage_layer`, so a top-level extrinsic that returns `Err`
   rolls back. But:
   - An **inner** helper marked `#[transactional]` *commits to the parent layer* on `Ok`. If the
     outer function is *not* transactional and a later step fails but the extrinsic still returns
     `Ok(())` (best-effort / "skip on error" loops), the committed inner state survives. (E.g. a
     swap helper commits, a fee transfer fails, the status-recording write is skipped, the
     extrinsic returns `Ok` → the order is replayable.)
   - `on_initialize` / `on_runtime_upgrade` hooks run **without** an outer `with_storage_layer`,
     so a swallowed `Err` (`if let Ok(..)`, `let _ = ..`) leaves partial state with nothing to roll
     it back. (E.g. stranded funds, or a watermark bumped on a failed transfer.)
   - **Chain extensions** call `pub fn` helpers that are *not* `#[pallet::call]`, so the auto-wrap
     does not apply — atomicity must be explicit. A removed `with_transaction` here is a silent
     regression.
   → Always check: is the multi-step unit wrapped? Does an inner step commit independently? Does
   the surrounding mode convert `Err` into `Ok`?

2. **Swallowed errors.** Grep the suspect region for `let _ =`, `if let Ok(`, `.ok();`,
   `.unwrap_or(`, `= ...; // ignore`. Each is a candidate partial-commit or silent-failure site.

3. **No in-batch dedup / pre-snapshot validation.** Batched extrinsics that validate every entry
   against the *same* pre-batch storage snapshot and write the de-dup marker only in a later phase
   → the same `order_id`/nonce passes N times. Contrast the per-item path that writes between
   iterations (a batched extrinsic vs the per-order one).

4. **Front-run of a deterministic `PalletId` account.** `PalletId(*b"…").into_account_truncating()`
   is publicly derivable. If a one-shot migration registers it via `create_account_if_non_existent`
   (silent no-op when the key already exists) and then sets `HasMigrationRun = true`
   *unconditionally*, any unprivileged user can pre-claim the hotkey via `burned_register` /
   `try_associate_hotkey` and permanently brick the enable flow. Check `is_subnet_account_id`-style
   reserved-account guards — they often don't cover non-subnet PalletIds.

5. **Accounting divergence vs `check_total_issuance`.** Real `pallet-balances` moves while
   `SubnetTAO` / `TotalStake` / `SubnetExcessTao` / `record_protocol_inflow` do not (or vice-versa).
   The two *issuance* counters can stay in sync while a *derived* counter drifts, so the standard
   invariant check misses it.

6. **Over-reduction of a shared aggregate.** A decrement (e.g. a "force reduce lock") that subtracts
   more than this caller's contribution to a pool shared by others.

## L2 verification chain to walk explicitly (Safeguard Table)

For a limit-orders-style extrinsic, show each gate passes on the way to the bug:
`LimitOrdersEnabled` → `ensure_signed` → sr25519 verify → `chain_id` match → expiry → price
condition → `Orders` status → `hotkey_account_exists` → `>= DefaultMinStake` →
`can_remove_balance` → **the buggy step** → **the skipped write**. Mark the bug row
"YES — this is the bug" and the skipped row "SKIPPED (short-circuits via `?`)".

## L3 repro gotchas (from real maintainer pushback)

- **`SubtokenEnabled[netuid]`** is `false` on a fresh `--dev` but `true` on testnet/mainnet. Set it
  in `beforeAll` via a sudo call (`AdminUtils.sudo_set_subtoken_enabled`) so the first leg runs and
  there is something to roll back. Always self-seed from fresh state.
- **`setPalletStatus(true)`** — `on_runtime_upgrade` may *disable* a pallet; re-enable it in setup.
- **ink! 5.x auto-revert** — a chain-extension message returning `Result` auto-sets
  `ReturnFlags::REVERT` on `Err`, masking a non-atomic pallet bug. Use a message returning a
  **non-`Result`** type (e.g. `-> u64`, returning `0` on error) so the partial commit is visible.
- **Rate limits / nonce** — `RateLimitExceeded` from registering too many subnets; nonce
  collisions from concurrent sibling tests → prefer sequential `submitAndWait` with well-known dev
  keys (Alice/Bob) over random keypairs.
- Pin everything to the stated commit; paste the before→after console deltas in the issue.

## Severity inputs specific to this domain

- Open-relay (`relayer: None`) = any signed origin → `PR:N`. A *listed relayer* the signer chose =
  still in-threat-model (not an excluded actor; decision-tree node 3), but model as `PR:L`.
- Permanent brick of an enable flow with no recovery extrinsic → `VA:H`, High.
- Overspend beyond a signed cap (victim gets equivalent alpha, attacker nets nothing) → `VI:H`,
  bounded → High or Medium depending on precondition width. Not Critical: no insolvency, no theft,
  victim is made whole in the other asset.
