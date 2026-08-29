# dev-playbook

A Claude Code plugin that packages one opinionated method for taking a feature or a
new subsystem from a raw idea to reviewed, merged code — using a coordinating
session plus fresh subagents per task.

## What's in it

| Path | What |
|---|---|
| `skills/orchestrated-delivery/SKILL.md` | The whole method: classify -> brainstorm -> spec -> phase -> plan -> per-task implementer + reviewer loop -> whole-branch review -> single fix wave -> finish. Cross-cutting principles (rulings-not-stalls, autonomy boundaries, model economics, debugging discipline) and copy-paste dispatch templates. |

## Install

**As a plugin** (versioned, shared across projects) — add this repo/branch as a
plugin marketplace source, then enable `dev-playbook`.

**Or just the skill** — copy `skills/orchestrated-delivery/` into
`~/.claude/skills/` (personal) or a project's `.claude/skills/`.

## Use

The skill auto-surfaces when a request looks architectural. Or invoke it directly:
`/orchestrated-delivery`. It will announce itself and walk the stages, stopping at
the human gates (design approval, spec approval, integration choice).

For a small, well-scoped change to code that already exists, the skill tells you to
take the lighter "bounded" path instead — a short in-chat design, an explicit yes,
then implement. Don't run the full arc for a one-file fix.

## Relationship to Superpowers

This overlaps heavily with the `superpowers` skill chain
(`brainstorming` -> `writing-plans` -> `subagent-driven-development` ->
`finishing-a-development-branch`). It is a consolidated, single-document version of
that flow plus adaptations learned in practice:

- committed **carry-forward docs** so deferred findings survive a gitignored ledger,
- **GUI-driving for acceptance / repro** (screenshot + coordinate input + process
  sampling to tell idle from wedged),
- explicit **debugging discipline** as its own principle,
- tighter **ledger / ruling** bookkeeping rules.

If you already run Superpowers, treat this as a reference overlay rather than a
replacement.

## Roadmap

- `scripts/` — ledger init, task-brief extractor, branch-diff packager (currently
  described in the skill; not yet bundled as runnable scripts).
- `agents/implementer.md`, `agents/task-reviewer.md` — thin role definitions so a
  dispatch is one line instead of pasting a template.
- Split the single skill into stage skills (`brainstorming`, `writing-plans`,
  `orchestrating-execution`, `finishing`) if the one-document load gets heavy in use.
