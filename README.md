# dev-playbook

A Claude Code plugin that packages one opinionated method for taking a feature or a
new subsystem from a raw idea to reviewed, merged code — using a coordinating
session plus fresh subagents per task.

## Requires

**The [`superpowers`](https://github.com/obra/superpowers) plugin.** `dev-playbook`
is an *overlay*: it orders Superpowers' skills into one arc, adds the human gates,
and layers on a few practices Superpowers doesn't cover. It is not a standalone
reimplementation. The skill's preflight step stops and tells the user to install
Superpowers if it's missing.

Install it through the `/plugin` menu — it's published in the official marketplace
as `superpowers`. Built and tested against superpowers **6.3.0**.

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

## How it uses Superpowers

Each stage delegates to a Superpowers skill; `dev-playbook` supplies the ordering,
the gates, and the additions:

| Stage | Delegates to |
|---|---|
| Shared understanding | `superpowers:brainstorming` |
| Spec, phasing, plan | `superpowers:writing-plans` |
| Execution, task loop, whole-branch review | `superpowers:subagent-driven-development` |
| Finishing | `superpowers:finishing-a-development-branch` |
| Bug hunts | `superpowers:systematic-debugging` |

What the overlay adds on top:

- committed **carry-forward docs** so deferred findings survive a gitignored ledger,
- **GUI-driving for acceptance / repro** (screenshot + coordinate input + process
  sampling to tell idle from wedged),
- **debugging discipline** as an explicit gate,
- explicit **autonomy boundaries** + the four halting conditions,
- tighter **ledger / ruling** bookkeeping (the `Ruling:` format; surface every
  ruling to the human at finish),
- **phasing** a large spec into sequential shippable sub-projects as a first-class step.

Where this document and a live Superpowers skill disagree, follow the Superpowers skill.

## Roadmap

- `scripts/` — ledger init, task-brief extractor, branch-diff packager (currently
  described in the skill; not yet bundled as runnable scripts).
- `agents/implementer.md`, `agents/task-reviewer.md` — thin role definitions so a
  dispatch is one line instead of pasting a template.
- Split the single skill into stage skills (`brainstorming`, `writing-plans`,
  `orchestrating-execution`, `finishing`) if the one-document load gets heavy in use.
