# Commit Message Guidelines - LIONGATE Engineering

This is the standard commit message format for all LIONGATE repositories. It
follows [Conventional Commits](https://www.conventionalcommits.org/), adapted with a ticket-reference footer to match
our GitHub Projects workflow (see wiki: *Ticket & Branch Workflow*).

---

## 1. Format

```
<type>(<scope>): <short summary>

<optional body>

Refs: <TICKET-ID>
```

- **type** - what kind of change (see table below)
- **scope** - optional, the area of the codebase affected (e.g. `footer`, `ci`, `auth`, `i18n`)
- **short summary** - imperative mood, lowercase, no period, ≤ 50 characters (`add`, not `added`/`adds`)
- **body** - optional, explains *why*, wrapped at ~72 characters per line
- **footer** - always include `Refs: TICKET-ID` so commits are traceable back to the board

## 2. Commit Types

| Type       | Use for                                                         |
|------------|-----------------------------------------------------------------|
| `feat`     | A new feature                                                   |
| `fix`      | A bug fix                                                       |
| `docs`     | Documentation only changes                                      |
| `style`    | Formatting, whitespace, missing semicolons - no logic change    |
| `refact`   | Code change that neither fixes a bug nor adds a feature         |
| `perf`     | Performance improvement                                         |
| `test`     | Adding or correcting tests                                      |
| `devops`   | Changes to CI configuration/scripts                             |
| `chore`    | Tooling, dependencies, build config, anything not covered above |
| `revert`   | Reverts a previous commit                                       |

## 3. Rules

- One logical change per commit - don't bundle a fix and a new feature together
- Imperative mood: `fix: correct footer email domain`, not `fixed` or `fixes`
- Reference the ticket ID in the footer on every commit, not just the PR
- Breaking changes get a `BREAKING CHANGE:` line in the footer, explaining the impact
- Keep the subject line short; put detail in the body if needed
- Squash noisy WIP commits before merging if your workflow allows it - the `develop` history should read cleanly

## 4. Worked Example - LG-DEVOPS-001 (Add CI tests on PR to develop)

This is the kind of commit sequence expected for a real ticket implementation - several small, well-scoped commits
rather than one giant one.

```
devops(workflows): add GitHub Actions workflow for PR tests against develop

Adds .github/workflows/ci.yml, which runs lint, test, and build
on every pull request opened against develop. Required so broken
code is caught before human review instead of after merge.

Refs: LG-DEVOPS-001
```

```
fix(lint): resolve existing lint warnings blocking new CI check

First CI run surfaced pre-existing lint warnings across the
codebase that were previously silent. Fixed so the new required
check can pass on a clean baseline.

Refs: LG-DEVOPS-001
```

```
test(ci): add missing npm test script for CI compatibility

package.json had no "test" script entry, so `npm test` failed
in the workflow. Added a minimal script wired to the existing
test runner.

Refs: LG-DEVOPS-001
```

```
docs(contributing): document required CI checks for contributors

Explains what the CI Tests check runs (lint, test, build) and
that it must pass before a PR can be merged into develop.

Refs: LG-DEVOPS-001
```

```
devops(workflows): cache npm dependencies to speed up CI run

Adds npm cache to actions/setup-node to bring workflow runtime
under the 5-minute target defined in the ticket's acceptance
criteria.

Refs: LG-DEVOPS-001
```

> Note: enabling the branch protection rule itself (Settings → Branches) is a repo configuration change, not a commit -
> it's tracked as a Definition of Done checkbox on the ticket, not in git history.
