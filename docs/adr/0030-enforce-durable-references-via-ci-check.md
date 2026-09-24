---
title: "Enforce the Durable-References Rule with an Automated, Diff-Scoped Check"
status: accepted
date: 2026-09-24
decision_makers: ["Claude Code (agent-assisted change)"]
category: tooling
nist_controls: ["CM-3", "SI-7"]
impact_level: low
ato_relevance: yes-process
risk_treatment: mitigate
supersedes: []
---

# ADR-0030: Enforce the Durable-References Rule with an Automated, Diff-Scoped Check

## Context and Problem Statement

AGENTS.md has two related sections — "Durable References: Cross-Link Code,
Docs, and ADRs — Not Trackers" and "Fully Qualify Issue/PR References in
Anything Durable" — that say code comments, docs, and ADR body prose must not
rely on a GitHub issue/PR reference (bare `#NNN`, shorthand `quickstart#NNN`,
or a full URL) to carry meaning. Both sections have existed as prose only:
nothing checked whether a new line of code or docs actually followed them.

Auditing the current tree shows the gap is not theoretical: `acq.backends/*.sh`,
several `docs/adr/*.md` bodies (outside their own "Links" sections),
`docs/KNOWN_FAILURE_MODES.md`, `docs/explorations/*.md`, and `CONTRIBUTING.md`
all carry inline issue-number citations that predate the rule being written
down, or were added after without anything catching it. Prose that nobody
verifies drifts from what the repository actually does — the general problem
this ADR addresses for one concrete, mechanically-checkable rule.

## Decision

Add `scripts/check-durable-references`, a POSIX-ish bash script (bash 3.2
safe — no `mapfile`, guarded empty-array expansion, consistent with `acq`'s
own constraints) that scans **added lines in a diff** for tracker-reference
patterns and fails if it finds one outside an allowed location.

### Diff-scoped, not whole-file

The script only ever looks at lines added relative to a diff — either
`git diff --cached` (local pre-commit hook) or `git diff <merge-base>...HEAD`
against a PR's base branch (CI). It never scans a whole file's existing
content. This is deliberate: the pre-existing backlog described above is real
and non-trivial, and requiring it all to be cleaned up before this check could
ship would block the check indefinitely. Diff-scoping makes this a ratchet —
new violations are caught at the point they're introduced; the existing
backlog is grandfathered, not silently excused (see "Consequences" below).

### Two call sites, one script

- **Local**: a new `check-durable-references` hook in `.pre-commit-config.yaml`
  runs `scripts/check-durable-references --cached` against files pre-commit
  selects. Under a normal `git commit`, that's exactly the staged diff. Under
  `pre-commit run --all-files` (the existing `pre-commit.yml` CI job, a fresh
  checkout with nothing staged), `git diff --cached` is empty for every file,
  so the hook is a **deliberate no-op** there — it does not turn the existing
  "lint everything" job into a mass-failure on the legacy backlog.
- **CI**: a new `.github/workflows/durable-references.yml` workflow runs
  `scripts/check-durable-references --against <PR base SHA>` on `pull_request`,
  which is the actual enforcement — it fails a PR that adds a new tracker
  reference to durable content, regardless of whether the contributor has
  pre-commit installed locally.

### Allowed locations and escape hatch

Consistent with AGENTS.md's own exceptions: an ADR's own `## Links` section,
`CHANGELOG.md` (release-please-generated), and `AGENTS.md` itself (which
cites `#NNN`-style patterns as examples while explaining the rule, not as
durable pointers) are exempt. A line carrying the literal marker
`durable-ref-ok` is also skipped, for a reviewed, deliberate exception the
rule's author didn't anticipate.

**Known limitation**: the `## Links`-section exemption is detected by
scanning the diff's own added lines in order for a `## Links` heading — it
only works when that heading line is itself part of the diff (e.g. a new ADR
added wholesale). Adding a line to an *existing* ADR's Links section without
touching the heading needs the `durable-ref-ok` escape hatch instead.
Precise detection would require correlating diff hunk line numbers against
the current file's heading positions; not worth the complexity for a section
that changes rarely.

## Consequences

- New PRs and commits that add a code comment or doc line depending on a bare
  or qualified issue/PR reference now fail fast, with a message pointing at
  the AGENTS.md rule and the escape hatch — this is the concrete instance of
  "pair the rule with a test that fails when it's violated" instead of prose
  alone.
- The existing backlog (`acq.backends/*.sh`, several ADR bodies,
  `docs/KNOWN_FAILURE_MODES.md`, `docs/explorations/*.md`,
  `docs/howto/*.md`, `CONTRIBUTING.md`, `scripts/verify-issue-320`) is
  **not** touched by this change and will not fail CI unless a future edit
  adds a *new* violating line near existing ones. It is tracked as deferred
  work per AGENTS.md's "Track Deferred Work" rule — see
  `docs/KNOWN_FAILURE_MODES.md` for the entry — rather than left as an
  unrecorded TODO.
- The pattern match is a heuristic (see the script's header comment for its
  known false-positive edge case with bare numeric hex-like strings); it is
  not a claim of perfect precision, only that new violations are now caught
  automatically instead of never.
- This does not touch the *external* `agentic-coding-playbook` repo's own
  AGENTS.md (the one actually injected into a running agent's sandbox via
  the `agentic-coding-playbook` kit) — it only affects work on this
  quickstart repo.

## Links

- Related: AGENTS.md, "Durable References" and "Fully Qualify Issue/PR
  References" sections
- Implementation: `scripts/check-durable-references`,
  `.pre-commit-config.yaml`, `.github/workflows/durable-references.yml`
