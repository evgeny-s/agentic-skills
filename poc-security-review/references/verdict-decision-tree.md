# Verdict decision tree

Every candidate is run through this tree by the assessor. The tree is a sequence of gates; a
candidate that fails any gate is rejected at that node with the stated disposition. Only a
candidate that survives every gate reaches **node 9 → confirmed**. Record the node reached in the
report's Verdict (e.g. "Decision-tree node: node 9 — survives all checks").

```
1. In scope?
   - Buggy code AND observable effect within the declared scope (targets.md)?
   NO  → REJECT: out-of-scope. (Note it for the maintainer, but it is not a finding here.)
   YES → 2

2. Already known / accepted?
   - Fixed in git history, tracked in an open issue/PR, listed in SECURITY.md / changelog,
     or documented as an accepted limitation? (run known-issue-search.md)
   YES → REJECT: known/duplicate. (Cross-reference it.)
   NO  → 3

3. Requires an excluded actor?
   - Does the only path require a privileged/trusted role the threat model excludes
     (admin, governance, the protocol team itself), with no unprivileged variant?
   YES → REJECT or DOWNGRADE: trusted-actor-only. (A *listed relayer the signer chose to trust*
         is NOT excluded — betraying that trust is the threat the design must resist.)
   NO  → 4

4. PoC executable?
   - Does a PoC exist and run GREEN (bug present) on current code at >= L1?
   NO  → HOLD: hypothesis only. Not a finding until proven. (Send back to PoC-writer.)
   YES → 5

5. PoC fabricated?
   - Run fabrication-audit.md. Any triggered item unjustified?
   YES → REJECT (fabricated) or DOWNGRADE level if only the level was inflated.
   NO  → 6

6. Reachable end-to-end?
   - Backward trace: an in-scope, attacker-reachable entry point leads to the buggy line with all
     intervening gates passing.
   - Forward trace: past the buggy line, no later gate reverses the harm; the path closes on a
     concrete, measurable bad outcome.
   NO  → REJECT: unreachable / harm does not close.
   YES → 7

7. Above the impact threshold?
   - Is the proven harm at least Low-with-real-impact (a genuine integrity/availability/fund
     effect), not purely theoretical or informational?
   NO  → DOWNGRADE to informational / defense-in-depth note.
   YES → 8

8. Severity & likelihood assigned honestly?
   - Impact-first tier + CVSS + likelihood + downgrade modifiers applied (severity-and-likelihood.md),
     with not-higher / not-lower justification.
   - poc-level validated (confirm or downgrade with reason).
   → 9

9. CONFIRMED.
   - Write the full report (internal .md) and the external issue. Status: confirmed.
```

## Notes on the hard gates

- **Node 3 — the trusted-actor distinction is subtle.** "Any signed origin" (open relay) is not
  excluded. A "listed relayer" the signer explicitly named *is* a party the signer trusted, so a
  malicious listed relayer is squarely the threat the signed-order design exists to bound — also
  not excluded. Genuine exclusions are protocol-level privileged roles (root/sudo/governance)
  whose compromise is out of the threat model.
- **Node 5 — fabrication is fatal, level-inflation is not.** If auth was faked or source was
  patched, the finding dies. If the only sin is claiming L3 when only L2 truly ran, keep the
  finding but downgrade `poc-level` and re-check that severity still holds on the L2 evidence.
- **Node 6 — both directions must close.** A bug you can reach but whose effect a later gate
  silently undoes is not a finding. A real harm you cannot reach from an attacker-controlled entry
  point is not a finding. The report's Reachability section documents both traces.
