# Known-issue search

A bug that is already fixed, already tracked, or documented as an accepted limitation is **not a
finding** (decision-tree node 2). Before confirming, search exhaustively and record what you
searched and what you found. This protects against wasting a maintainer's time and against
re-reporting a duplicate in a contest.

## Sources to search (record each, even if empty)

1. **Git history.** The highest-signal source.
   - `git log -p -- <buggy-file>` and `git blame -L <start>,<end> <file>` on the suspect lines.
   - `git log --all --oneline --grep="<keyword>"` for terms naming the bug class
     (`dedup`, `duplicate`, `transactional`, `atomic`, `frontrun`, `migration`, `rollback`).
   - Look specifically for a commit that *introduced* the regression (e.g. one that removed a
     `with_transaction` wrapper) and confirm no later commit fixes it. Note the introducing
     commit + date in the report.

2. **Issue / PR tracker.** `gh issue list`, `gh pr list`, and full-text search for the function
   name and the bug class. Check both open and closed/merged — a fix may be merged but the report
   you are writing might predate awareness of it. If a PR fixes it, cross-reference and stop
   (e.g. an issue closed with "Fixed in PR #…").

3. **In-code acknowledgment.** Search the suspect region for `TODO`/`FIXME`/`HACK`/`SAFETY`/
   `known issue` and any project-specific review annotation (`@qa-review`, `@audit`). Distinguish
   *your own* annotations (which reference this very bug id) from a pre-existing acknowledgment by
   the maintainers. A pre-existing "this is intentional" note can move the finding to node 3/2.

4. **SECURITY.md / changelog / release notes.** Accepted limitations and prior advisories.

5. **Existing tests.** Search the test suite for a test exercising this path. A subtly *wrong*
   existing test is strong evidence the bug was not noticed (e.g. a test whose assertion states the
   opposite of what the code actually does). Note it.

6. **External databases (when the component is public / on-chain).** CVE/NVD, GitHub Advisory
   Database, and for smart contracts the relevant contest archives (Code4rena / Sherlock /
   Immunefi). For a private/new module these are typically N/A — say so explicitly.

7. **Prior audit reports** for the project, if any.

## Output

In the report's **Known-Issue Search** block, list each source and the result. End with a clear
conclusion: **"Not a known/accepted issue. Confirmed new finding."** or **"Duplicate of <ref> —
rejected at node 2."** If a wrong existing test exists, quote its misleading assertion as evidence
the behavior was never intended.
