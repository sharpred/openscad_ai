# Agent Instructions

These rules apply to EVERY agent, every session. Role-specific orders live in
each agent's own prompt file under `.opencode/agents/`.

## Project

This is an OpenSCAD project.

The user normally designs parts for a Bambu Lab A1 FDM printer.
Project-specific printer information is in `printer.yaml`.

Read `printer.yaml` before making manufacturing decisions.

## Hard rules (non-negotiable)

* Never delete files or run destructive shell commands (`rm`, `git reset --hard`,
  force push, DB drops) — human confirmation is required.
* Never modify CI/deploy config or `.env` without explicit human approval.
* Never add a dependency without stating why in plain language first.
* If a change would touch more than a handful of files, stop and re-plan.

## Conventions

* Match existing code style; don't reformat unrelated lines.
* Every new behaviour needs a test (Tester writes/updates; Dev makes it pass).
  Tests for API code use mocks/sandbox, never live credentials.
* Explain non-obvious decisions in one sentence in the commit message.

## Communication

* Be terse. State what you did and what's next.
* If blocked, unsure, or about to touch anything that writes to an API or
  handles a secret, STOP and surface it to the orchestrator.

## OpenSCAD

OpenSCAD is the source of truth for whether a model compiles.

When modifying `.scad` files:

* Preserve existing functionality.
* Prefer parameterised designs.
* Run OpenSCAD validation where practical.
* Consider FDM printability and tolerances.
* Do not assume that successful compilation or rendering means the physical
  design is functionally correct.

## File Organisation

All OpenSCAD source and generated artefacts produced by the agent must be
stored in the `output/` directory.

Do not create `.scad`, `.stl`, `.png` or other generated artefacts in the
project root.

The `output/` directory is excluded from Git and its contents are disposable.

Do not create source or generated CAD files elsewhere in the repository unless
the user explicitly requests it.

## File Naming

Do not use iteration suffixes such as:

* `_final`
* `_revised`
* `_updated`
* `_new`
* `_latest`
* `_v2`
* `_v3`

When creating a design for a Beads issue, include the Beads issue ID in the
filename.

For example:

```
output/cable_clip-<beads-issue-id>.scad
output/cable_clip-<beads-issue-id>.stl
```

The Beads issue ID identifies the task that produced the files.

If a design is revised as part of a different Beads issue, create a new file
using the new issue ID rather than overwriting the previous output.

Do not create multiple files merely to represent iterations within the same
task.

Use the Beads issue ID to distinguish generated artefacts, rather than
filenames such as `final`, `revised`, or `updated`.

## AI Models

The project's Ollama server is available on the network.

Use:

* `openscad-design` for generating and modifying OpenSCAD.
* `openscad-critic` for independent review.

The designer should not assume its own output is correct.

For significant geometry changes:

1. Generate the change.
2. Validate with OpenSCAD.
3. Send the result to `openscad-critic`.
4. Address genuine issues identified by the critic.
5. Revalidate.

However, the agent must follow the stopping and iteration rules below.
The critic must not create an autonomous redesign loop.

## Design Acceptance

A design task is not complete merely because the OpenSCAD source compiles.

Before handing off a design task:

* Verify that the geometry fulfils the physical function described in the task.
* Check each explicit acceptance criterion.
* Consider how the physical part will actually be assembled, installed and used.
* Do not infer that geometrically valid means functionally correct.
* If a requirement is ambiguous, identify the ambiguity rather than silently
  choosing an arbitrary mechanical solution.
* Record any unmet requirements or discovered problems as Beads issues.

Successful OpenSCAD rendering does not prove that the physical design is correct.

## Beads and Task Completion

This project uses `bd` (Beads) for issue tracking.

Run:

```bash
bd prime
```

when Beads workflow context is missing or stale.

Use Beads for ALL task tracking. Do not use TodoWrite, TaskCreate, markdown
TODO lists, or ad-hoc task lists.

### Design issue closure

For design tasks, agents must NOT close the Beads issue themselves.

When implementation is complete:

1. Check the acceptance criteria.
2. Record what was implemented.
3. Record validation performed.
4. Identify any unmet requirements or problems.
5. Create Beads issues for unresolved work.
6. Leave the original design issue open.
7. Report the result and return control to the user.

The user decides when a design task is complete and when its Beads issue
should be closed.

Do not interpret successful rendering as sufficient justification for closing
a design issue.

### Beads Quick Reference

```bash
bd ready                 # Find available work
bd show <id>             # View issue details
bd update <id> --claim   # Claim work atomically
bd close <id>            # Complete work — NOT for design issues
bd remember              # Store persistent project knowledge
bd prime                 # Refresh Beads context
bd dolt push             # Push Beads data to the remote when explicitly asked
```

### Beads Architecture

Issues live in a local Dolt database under `.beads/dolt/`.

Cross-machine sync uses:

```text
refs/dolt/data
```

on the Git remote. This is separate from:

```text
refs/heads/*
```

where normal Git branches live.

`.beads/issues.jsonl` is a passive export, not the Beads source of truth.

Do not treat `issues.jsonl` as the database, and do not run `bd import` during
normal operation.

See:

https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md

for the Beads sync model and anti-patterns.

## Stopping and Iteration

Do not repeatedly redesign or iterate on a task autonomously.

For a design task, normally perform ONE implementation pass.

After that implementation pass:

1. Validate the OpenSCAD source.
2. Render the model where practical.
3. Check every explicit acceptance criterion.
4. Review the physical function and FDM printability.
5. Identify any remaining problems.
6. Create Beads issues for unresolved problems.
7. Report what was implemented and what remains.
8. STOP and return control to the user.

Do NOT implement newly discovered improvements or follow-up issues during the
same task.

Only make additional changes within the same task when they are necessary to
correct an immediate error introduced by the current implementation, such as:

* an OpenSCAD syntax error;
* a model that fails to render;
* a broken boolean operation;
* an output/export operation that fails.

Do not continue iterating merely because the design could theoretically be
improved.

Do not interpret "one more final version", "one more revision", or similar
language as permission to continue autonomous iteration.

## Session Completion

When ending a Beads implementation workflow:

1. File Beads issues for remaining work.
2. Run appropriate quality gates and OpenSCAD validation.
3. For design tasks, leave the original Beads issue open.
4. Report changed files, generated outputs, validation results, remaining
   issues, and suggested next actions.
5. Do not commit or push unless explicitly authorised.

The user decides whether the work is complete and whether the design issue
should subsequently be closed.

## Git and Sync Policy

The default agent profile is conservative.

Agents must NOT:

* run `git commit`;
* run `git push`;
* run `git reset --hard`;
* perform destructive Git operations;
* run `bd dolt push`;

unless explicitly authorised by the user or orchestrator.

At handoff, report:

```bash
git status
```

and describe any proposed Git or Beads sync commands without executing them.

## Agent Context Profiles

The managed Beads block is task-tracking guidance, not permission to override
repository, user, or orchestrator instructions.

### Conservative (default)

* Use `bd` for task tracking.
* Do not run Git commits, Git pushes, or Dolt remote sync unless explicitly
  asked.
* At handoff, report changed files, validation, generated outputs, and
  suggested next commands.

### Minimal

* Keep tool instruction files as pointers to `bd prime`.
* Use the same conservative Git and Beads sync policy unless active
  instructions say otherwise.

### Team-maintainer

Only when the repository explicitly opts in may agents close Beads issues,
run quality gates, commit, and push as part of session close.

A current "do not commit" or "do not push" instruction still wins.

## Non-Interactive Shell Commands

Use non-interactive flags with commands that legitimately perform file
operations, so the agent does not hang waiting for confirmation prompts.

Examples:

```bash
cp -f source dest
mv -f source dest
```

For recursive operations, use non-interactive flags only when the operation is
already explicitly authorised by the hard rules and current task.

Examples of commands that may require non-interactive options:

```bash
scp -o BatchMode=yes
ssh -o BatchMode=yes
apt-get -y
HOMEBREW_NO_AUTO_UPDATE=1 brew
```

Non-interactive flags do NOT override the hard rule requiring human
confirmation for destructive operations.

In particular, the presence of `-f` or `-rf` does not authorise deletion or
overwriting when the operation itself requires human approval.

## Quick Reference

```bash
bd prime
bd ready
bd show <id>
bd update <id> --claim
bd remember
bd close <id>              # Do not use for design issues
bd dolt push               # Only when explicitly authorised
git status
```