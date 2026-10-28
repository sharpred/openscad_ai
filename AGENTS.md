# Agent Instructions

These rules apply to EVERY agent, every session. Role-specific orders live in
each agent's own prompt file under `.claude/agents/` (`orchestrator`,
`cad-designer`, `cad-critic`).

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

## Job Intake (`source/`)

New work arrives as a job folder under `source/<job-slug>/`, containing one
or more reference photos of the real object and a markdown file describing
the job and referencing those photos by filename. `source/` is excluded
from Git — like `output/`, it's disposable/local, not project history.

The `orchestrator` agent scans `source/` for job folders that don't yet have
a `.beads-issue` marker file (written once a design issue is created for
that job) and kicks off the design -> critic -> human cycle described under
"AI Models" below. A job with no photos, an empty spec, or photos the spec
doesn't actually reference is incomplete — it should not be handed to
`cad-designer`; report the gap instead of guessing at missing dimensions or
shape.

### Measurement Sanity Check

Before `orchestrator` delegates a job to `cad-designer`, it must check the
stated measurements against each other and report any problem to the human.
This costs nothing and happens before any agent pass is spawned.

Check:

* **Containment.** Every internal feature must fit inside the outer envelope
  (`bore < hub < outside diameter`; every feature radius < outer radius). A
  figure that cannot physically fit is the strongest signal available that a
  radius has been written where a diameter was meant.
* **Sum closure.** Do the stated thicknesses add up to the stated overall
  dimension? If two measurements sum to a known overall, the datum they share
  is confirmed. This is the cheapest way to settle a datum question and is
  usually more reliable than reading a photograph.
* **Datum declared.** Every depth or height must say what it is measured
  from. "Depth 4.48" is not a measurement until the datum is stated.
* **Spec text against photos.** Check countable and categorical claims —
  how many spokes, whether a feature protrudes or is recessed. Where the
  written spec and the photographs disagree, the photographs usually win,
  but the human decides.
* **Tolerance realism.** Flag any dimension landing near the FDM minimum
  wall (0.8 mm, two perimeters at a 0.4 mm nozzle) as needing a human
  decision, not a silent trade.

Report impossibilities and ambiguities to the human and wait. Do not pick a
resolution, and do not delegate a job whose measurements contradict each
other — a design pass built on a bad datum is wasted work.

A sanity check detects inconsistency, not error. A set of measurements can
be perfectly self-consistent and still wrong. Passing this check is not
evidence that the numbers are right.

One optional addition, if you want the photo lesson in the file rather than only in bd remember — it fits naturally just above, in the existing Job Intake prose:

Reference photos should be **cropped tightly to the part, not downscaled**.
Images are resized server-side before tokens are counted, so a tightly
cropped photo and a wide one cost the same — but the cropped one gives
several times the usable detail. For measurement photos, frame the caliper
display and the jaw contact point together, and state the figure in text as
well.

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

The `.scad` and `.stl` are both real deliverables, not intermediate
artefacts — once a job is accepted, the human edits either file directly
for out-of-phase changes. A `.step` file is intended to join them
eventually, but is not currently produced: the project's installed OpenSCAD
(2021.01) has no STEP export support, and adding a newer build for that has
been deliberately deferred (decided 2026-10-28). Do not attempt
`--export-format=step`; it will fail against this build.

If a design is revised as part of a different Beads issue, create a new file
using the new issue ID rather than overwriting the previous output.

Do not create multiple files merely to represent iterations within the same
task.

Use the Beads issue ID to distinguish generated artefacts, rather than
filenames such as `final`, `revised`, or `updated`.

## AI Models

Design and review run locally via the Claude CLI, using three dedicated
Claude Code agents (`.claude/agents/`):

* `orchestrator` — scans `source/` for new jobs (see "Job Intake" above),
  drives the Beads workflow below, and raises the human decision bead. Never
  models, renders, or critiques anything itself.
* `cad-designer` — generates and modifies OpenSCAD from source photos and
  measurements. CAD/3D-printing expert; owns manufacturability, tolerances,
  and FDM printability for the Bambu Lab A1.
* `cad-critic` — adversarial industrial-design reviewer. Deliberately a
  different model and a different kind of expert (form/proportion, not
  manufacturing). Compares a rendered image of the model against the
  original photos and spec, and does not touch printability.

The designer should not assume its own output is correct.

For a design task, the normal flow (driven by `orchestrator`) is:

1. `orchestrator` creates a design issue for the job and delegates to
   `cad-designer`, which generates the model from the photos/measurements
   and renders a PNG (`openscad -o render.png --render ...`), then validates
   with OpenSCAD.
2. `orchestrator` creates a review issue for `cad-critic`, depending on the
   design issue (`bd dep add <critic-id> <design-id>`), carrying the render,
   the source photos, and the spec.
3. `cad-critic` reports visual/proportional discrepancies (or confirms the
   render matches) — one pass, no auto-fixes, no closing the design issue.
   `orchestrator` closes the review issue once the critique is recorded.
4. `orchestrator` raises a human decision bead (`bd create ... -l human`
   immediately followed by `bd defer`, per the convention in
   `.claude/agents/orchestrator.md`) summarising the design and the critic's
   findings. The human decides whether the design is accepted; comments on
   that bead describing a change start a new design issue and a fresh pass
   through this same cycle — not a reopening of the old one.

Every agent in this cycle must follow the stopping and iteration rules
below. None of them may create an autonomous redesign loop or close the
design issue itself; see "Design Acceptance" below.

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