# Agent Instructions

These rules apply to EVERY agent, every session. Role-specific orders live in
each agent's own prompt file under `.opencode/agents/`.

## Project

This is an OpenSCAD project.

The user normally designs parts for a Bambu Lab A1 FDM printer.
Project-specific printer information is in printer.yaml.

Read printer.yaml before making manufacturing decisions.


## Hard rules (non-negotiable)
- Never delete files or run destructive shell commands (rm, git reset --hard,
  force push, DB drops) — human confirmation required.
- Never modify CI/deploy config or `.env` without explicit human approval.
- Never add a dependency without stating why in plain language first.
- If a change would touch more than a handful of files, stop and re-plan.

## Conventions
- Match existing code style; don't reformat unrelated lines.
- Every new behaviour needs a test (Tester writes/updates; Dev makes it pass).
  Tests for API code use mocks/sandbox, never live credentials.
- Explain non-obvious decisions in one sentence in the commit message.

## Communication
- Be terse. State what you did and what's next.
- If blocked, unsure, or about to touch anything that writes to the API or
  handles a secret, STOP and surface it to the orchestrator.

## OpenSCAD

OpenSCAD is the source of truth for whether a model compiles.

When modifying `.scad` files:
- Preserve existing functionality.
- Prefer parameterised designs.
- Run OpenSCAD validation where practical.
- Consider FDM printability and tolerances.

## AI models

The project's Ollama server is available on the network.

Use:
- `openscad-design` for generating and modifying OpenSCAD.
- `openscad-critic` for independent review.

The designer should not assume its own output is correct.

For significant geometry changes:
1. Generate the change.
2. Validate with OpenSCAD.
3. Send the result to `openscad-critic`.
4. Address genuine issues identified by the critic.
5. Revalidate.

## Design acceptance

A design task is not complete merely because the OpenSCAD source compiles.

Before closing a design issue:
- Verify that the geometry fulfils the physical function described in the task.
- Check each explicit acceptance criterion.
- Consider how the physical part will actually be assembled, installed and used.
- Do not infer that geometrically valid means functionally correct.
- If a requirement is ambiguous, identify the ambiguity rather than silently
  choosing an arbitrary mechanical solution.

## Beads and task completion

For design tasks, do NOT close the Beads issue yourself.

When implementation is complete:
- Check the acceptance criteria.
- Record what was implemented.
- Record any unmet requirements or problems as new Beads issues.
- Leave the original issue open.
- Return control to the user.

The user decides when a design task is complete and when its Beads issue
should be closed.

Do not interpret successful OpenSCAD rendering as proof that the physical
design is correct.

## Stopping and iteration

Do not repeatedly redesign or iterate on a task autonomously.

After one implementation pass:
- validate the OpenSCAD source;
- check the acceptance criteria;
- identify problems;
- create Beads issues for unresolved problems;
- stop.

Do not implement newly discovered improvements during the same task.

Only make additional changes when necessary to correct an immediate error
introduced by the current implementation, such as a syntax error or a model
that fails to render.

This project uses **bd** (beads) for issue tracking. Run `bd prime` for full workflow context.

> **Architecture in one line:** Issues live in a local Dolt database
> (`.beads/dolt/`); cross-machine sync uses `bd dolt push/pull` (a
> git-compatible protocol), stored under `refs/dolt/data` on your git
> remote — separate from `refs/heads/*` where your code lives.
> `.beads/issues.jsonl` is a passive export, not the wire protocol.
>
> See [SYNC_CONCEPTS.md](https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md)
> for the one-screen overview and anti-patterns (don't treat JSONL as the
> source of truth; don't `bd import` during normal operation; don't
> reach for third-party Dolt hosting before trying the default).

## Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work atomically
bd close <id>         # Complete work
bd dolt push          # Push beads data to remote
```

## Non-Interactive Shell Commands

**ALWAYS use non-interactive flags** with file operations to avoid hanging on confirmation prompts.

Shell commands like `cp`, `mv`, and `rm` may be aliased to include `-i` (interactive) mode on some systems, causing the agent to hang indefinitely waiting for y/n input.

**Use these forms instead:**
```bash
# Force overwrite without prompting
cp -f source dest           # NOT: cp source dest
mv -f source dest           # NOT: mv source dest
rm -f file                  # NOT: rm file

# For recursive operations
rm -rf directory            # NOT: rm -r directory
cp -rf source dest          # NOT: cp -r source dest
```

**Other commands that may prompt:**
- `scp` - use `-o BatchMode=yes` for non-interactive
- `ssh` - use `-o BatchMode=yes` to fail instead of prompting
- `apt-get` - use `-y` flag
- `brew` - use `HOMEBREW_NO_AUTO_UPDATE=1` env var

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:6cd5cc61 -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.

## Agent Context Profiles

The managed Beads block is task-tracking guidance, not permission to override repository, user, or orchestrator instructions.

- **Conservative (default)**: Use `bd` for task tracking. Do not run git commits, git pushes, or Dolt remote sync unless explicitly asked. At handoff, report changed files, validation, and suggested next commands.
- **Minimal**: Keep tool instruction files as pointers to `bd prime`; use the same conservative git policy unless active instructions say otherwise.
- **Team-maintainer**: Only when the repository explicitly opts in, agents may close beads, run quality gates, commit, and push as part of session close. A current "do not commit" or "do not push" instruction still wins.

## Session Completion

This protocol applies when ending a Beads implementation workflow. It is subordinate to explicit user, repository, and orchestrator instructions.

1. **File issues for remaining work** - Create beads for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **Handle git/sync by active profile**:
   ```bash
   # Conservative/minimal/default: report status and proposed commands; wait for approval.
   git status

   # Team-maintainer opt-in only, unless current instructions forbid it:
   git pull --rebase
   git push
   git status
   ```
5. **Hand off** - Summarize changes, validation, issue status, and any blocked sync/commit/push step

**Critical rules:**
- Explicit user or orchestrator instructions override this Beads block.
- Do not commit or push without clear authority from the active profile or the current user request.
- If a required sync or push is blocked, stop and report the exact command and error.
<!-- END BEADS INTEGRATION -->

<!-- BEGIN BEADS CODEX SETUP: generated by bd setup codex -->
## Beads Issue Tracker

Use Beads (`bd`) for durable task tracking in repositories that include it. Use the `beads` skill at `.agents/skills/beads/SKILL.md` (project install) or `~/.agents/skills/beads/SKILL.md` (global install) for Beads workflow guidance, then use the `bd` CLI for issue operations.

### Quick Reference

```bash
bd ready                # Find available work
bd show <id>            # View issue details
bd update <id> --claim  # Claim work
bd close <id>           # Complete work
bd prime                # Refresh Beads context
```

### Rules

- Use `bd` for all task tracking; do not create markdown TODO lists.
- Run `bd prime` when Beads context is missing or stale. Codex 0.129.0+ can load Beads context automatically through native hooks; use `/hooks` to inspect or toggle them.
- Keep persistent project memory in Beads via `bd remember`; do not create ad hoc memory files.

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.
<!-- END BEADS CODEX SETUP -->

