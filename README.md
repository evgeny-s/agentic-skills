# agentic-skills

A collection of [Claude](https://claude.com/claude-code) **skills** — reusable, self-contained
capabilities that teach an agent a methodology. Each skill is a directory with a `SKILL.md`
(frontmatter + when-to-use + the playbook) and optional `references/` files loaded on demand
(progressive disclosure).

## Skills

### `poc-security-review`

An adversarial, **proof-driven security review**. It hunts for exploitable bugs in a scoped
codebase, proves each one with a graded proof-of-concept (L1 isolated → L2 integration → L3 live),
then independently self-audits every finding (fabrication, severity/CVSS, likelihood,
reachability, known-issue search) before writing a defensible report and a filable GitHub issue.

Core discipline: **prove-or-discard** — no passing PoC, no finding. First-class support for
Substrate / FRAME pallets; the methodology is domain-general.

- Entry point: [`poc-security-review/SKILL.md`](poc-security-review/SKILL.md)
- References: pipeline, PoC levels, severity & CVSS, the 22-item fabrication audit, verdict
  decision tree, report templates, known-issue search, and a Substrate/FRAME playbook.

## Using a skill

Point Claude Code at this repo (or copy a skill directory into your skills location) and invoke it
by name. Claude reads `SKILL.md` first and pulls the `references/` files as the task needs them.
