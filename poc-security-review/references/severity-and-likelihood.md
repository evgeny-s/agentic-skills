# Severity, CVSS, likelihood, and downgrade modifiers

Severity is assigned **impact-first**: start from the realistic worst-case harm a confirmed
attack causes, then temper with likelihood and the documented downgrade modifiers. Never start
from "this feels scary." Never inflate to make a finding look better.

## Step 1 — Impact tier (impact-first)

Pick the tier from the worst-case *proven* harm (the PoC must actually demonstrate it):

- **Critical** — direct protocol/system insolvency, arbitrary theft/loss of others' funds or
  data, permanent freeze of existing user assets, full auth bypass / RCE.
- **High** — assets drained *beyond a user's authorized amount* via a valid path; permanent DoS of
  a core function with no on-chain/in-system recovery; privilege escalation. (Examples:
  duplicate-order-id draining N× the signed cap; front-run permanently bricking the pallet enable
  flow.)
- **Medium** — real, measurable integrity/fund harm that requires a non-trivial precondition or a
  specific state window; bounded loss; recoverable. (Example: fee-failure replay — real overpay
  beyond the signed cap, but only when the signer's balance sits in a narrow window.)
- **Low** — limited impact, easily recovered, cosmetic-adjacent, or requires implausible setup.
- **Informational** — doc/code mismatch, defense-in-depth, no demonstrable harm.

Write a one-paragraph **basis** tying the tier to the *proven* differential, and note explicitly
why it is **not** the tier above and **not** the tier below (do this for every finding — it is
the most scrutinized part of the writeup).

## Step 2 — CVSS 4.0 vector

Attach a full CVSS:4.0 vector. The base metrics that matter most here:

- `AV` (Attack Vector): `N` for anything reachable over the network / public RPC / public ABI.
- `AC` (Attack Complexity): `L` if no special conditions; `H` if it needs a specific state window
  or timing.
- `AT` (Attack Requirements): `P` (present) when the attack needs timing relative to an event
  (e.g. acting before a runtime upgrade lands); `N` otherwise.
- `PR` (Privileges Required): `N` for fully unprivileged; `L` for any authenticated/signed origin
  with no special role; `H` for admin/root.
- `VC/VI/VA` (Confidentiality/Integrity/Availability impact on the vulnerable system): set the one
  that is actually harmed to `H`, the rest `N`. A fund-overspend is `VI:H`. A permanent brick is
  `VA:H`. A stranded-funds accounting break is `VI:H` (integrity of bookkeeping).
- `SC/SI/SA` (subsequent-system impact): usually `N` unless harm crosses a trust boundary.

Example vectors:
- Fee-failure replay (medium): `CVSS:4.0/AV:N/AC:H/AT:N/PR:N/UI:N/VC:N/VI:H/VA:N/SC:L/SI:L/SA:N`
- Front-run brick (high): `CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:N/VC:N/VI:H/VA:H/SC:N/SI:N/SA:N`
- Duplicate order-id (high): `CVSS:4.0/AV:N/AC:L/AT:N/PR:L/UI:N/VC:N/VI:H/VA:N/SC:N/SI:N/SA:N`

## Step 3 — Likelihood (assessed independently of severity)

Levels: **unlikely / possible / likely / almost-certain**. Decide it from *named factors*, not vibe:

- **Attack vector** — network vs local vs physical.
- **Access-ladder position** — anonymous external? any authenticated user? listed/trusted relayer?
  privileged admin? The lower the rung required, the higher the likelihood.
- **Preconditions** — none, vs a specific state/window. A window of width `fee_tao` that a target
  must drift into is a real damper (→ *possible*); a days-to-weeks timing window before an upgrade
  is *wide* (→ *likely*).
- **Capital** — zero (reuse an existing signed object) raises likelihood.
- **Race/timing window** — single-block race (narrow) vs days (wide).
- **Reversibility / detection** — irreversible, no recovery extrinsic → both higher severity and,
  for the attacker, lower deterrence.
- **Public-exploit signal** — is there a circulating PoC / is the component live yet?

State the combined estimate with a rough probability band (e.g. "40–60% → possible").

## Step 4 — Downgrade modifiers

These *reduce* the rating when the attack leans on an unrealistic crutch. Evaluate each and say
"applies / does not apply" (list them all explicitly, even when none apply):

- **Requires a privileged role** (admin/root/governance) the threat model trusts → downgrade.
- **Requires a leaked/compromised key** of an otherwise-trusted party → downgrade.
- **Depends on another protocol's failure** / external oracle going rogue → downgrade.
- **Flash-loan dependency** where the attacker must source capital they don't have → downgrade.
- **Private-mempool / block-builder front-running** in the narrow single-block-ordering sense
  (does NOT apply to a days-wide pre-upgrade window — that is just a normal public action).
- **Single-occurrence, self-healing DoS** (clears itself next block) → downgrade; a *permanent*
  DoS does not.

A finding where **no** modifier applies (unprivileged, zero capital, no flash loan, no private
mempool, permanent effect) holds its impact-first tier.

## Step 5 — Maintainer-triage anchor

State how the project's maintainers would classify it, in their language. Use
**"security advisory"** for anything that would be privately disclosed and patched before
deployment (a signed-amount invariant broken by a missing transaction wrapper; a zero-cost
permanent brick). This anchors the severity to real-world triage rather than an abstract rubric,
and signals to the maintainer how to handle the report.

## Putting it in the report

The Investigation section carries: **Tier**, **CVSS**, **Basis** (with not-higher / not-lower
justification), **Maintainer-triage anchor**, then a separate **Likelihood** block with the named
factors, and a **Downgrade Modifiers Applied** block listing each modifier's verdict. See
`report-templates.md`.
