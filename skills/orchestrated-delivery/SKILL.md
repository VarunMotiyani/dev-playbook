---
name: orchestrated-delivery
description: Use when taking a non-trivial feature, a new subsystem, or a from-scratch build from idea to merged code. Runs the full arc — classify the request, brainstorm to a committed spec, split into phases, write a bite-sized plan, execute it by dispatching one fresh implementer subagent and one reviewer per task, then a whole-branch review, a single consolidated fix wave, and finish. For architectural work, not one-file fixes (those are "bounded" — see the skill body).
---

# The Agentic Development System

A general method for taking a raw idea to shipped, reviewed code with an AI agent (or a team of them). Project-agnostic. Hand this to any agent on any codebase and it should know how work is supposed to flow.

**Announce at start:** "I'm using the orchestrated-delivery skill to take this from idea to merge."

The system has one core belief: **the expensive mistakes happen before any code is written** — in misunderstood intent, unbounded scope, and vague plans. So most of the discipline is front-loaded into understanding and planning, and execution becomes a tight, mechanical loop with review gates.

---

## Requires: the Superpowers plugin

This skill is an **overlay on `superpowers`**, not a replacement. Superpowers' skill chain already implements the mechanics of every stage; this skill orders them, wires in the gates, and adds a handful of practices Superpowers doesn't cover.

**Preflight — do this before Stage 1, every time:**

1. Check whether the `superpowers` skills are available (e.g. `superpowers:brainstorming`, `superpowers:subagent-driven-development` appear in your skill list, or `~/.claude/plugins/cache/*/superpowers/` exists).
2. **If they are not installed, STOP.** Tell the user:
   > This skill requires the **superpowers** plugin. Install it through the `/plugin`
   > menu (it's published in the official marketplace as `superpowers`), then
   > re-invoke me. Built and tested against superpowers 6.3.0.
   Do not proceed with a hand-rolled version of the method — the point of this skill is to run Superpowers correctly, not to reimplement it.
3. Once present, `superpowers:using-superpowers` governs skill discipline; invoke the stage skills below as you reach each stage.

**Stage → Superpowers skill:**

| Stage(s) | Invoke |
|---|---|
| 1 — Shared understanding | `superpowers:brainstorming` |
| 2–4 — Spec, phasing, plan | `superpowers:writing-plans` (brainstorming produces the spec first) |
| 5–9 — Execution setup, task loop, whole-branch review | `superpowers:subagent-driven-development` |
| 10 — Finishing | `superpowers:finishing-a-development-branch` |
| any bug hunt | `superpowers:systematic-debugging` |

**What this overlay adds on top of Superpowers** (the rest of this document is the detail):

- **Committed carry-forward docs** (§13) — deferred findings survive the gitignored SDD ledger.
- **Debugging discipline as a gate** (§15) — reproduce → inspect real failure state → one hypothesis → test it; plus driving a GUI for acceptance/repro.
- **Explicit autonomy boundaries** (§14) and the four halting conditions (§12), stated up front.
- **Tighter ledger / ruling bookkeeping** — the `Ruling:` format and the "surface every ruling to the human at finish" rule.
- The **Stage 3 phasing** step — splitting a large spec into sequential, independently shippable sub-projects — as a first-class stage rather than a note.

The stage sections below carry the full method so it stays legible in one read, and so it degrades gracefully if a Superpowers skill is mid-refactor. Where a stage section and the live Superpowers skill disagree, **follow the Superpowers skill** and treat this as commentary.

---

## 0. The shape of the whole thing

```
raw idea
  │
  ▼  STAGE 1 — Shared understanding  (conversation, not code)
  │   classify: spike / bounded / architectural
  │   one question at a time → 2–3 approaches → sectioned design → EXPLICIT approval
  │
  ▼  STAGE 2 — Spec               a written design doc, committed
  │
  ▼  STAGE 3 — Phasing            split a big spec into sequential sub-projects,
  │                               each independently shippable
  │
  ▼  STAGE 4 — Plan (per phase)   file-structure first, then bite-sized tasks
  │                               with REAL code in every step, no placeholders
  │
  ▼  STAGE 5 — Execution setup    isolated branch/worktree · ledger file ·
  │                               pre-flight conflict scan
  │
  ▼  STAGE 6 — The task loop  ┌────────────────────────────────────────────┐
  │   repeat per task:        │ record BASE → brief → dispatch implementer  │
  │                           │ → implement (TDD) → commit → self-review    │
  │                           │ → dispatch reviewer → 2 verdicts            │
  │                           │ → fix loop (≤5 rounds) → scoped re-review   │
  │                           │ → ledger: "Task N complete"                 │
  │                           └────────────────────────────────────────────┘
  │
  ▼  STAGE 7 — Whole-branch review   one broad review · one consolidated fix wave ·
  │                                  one scoped re-review · adjudicate residuals
  │
  ▼  STAGE 8 — Finish            full suite green → present integration options →
  │                              merge / PR (human decides)
  │
  ▼  STAGE 9 — Acceptance & carry-forward
      human validates the running result · deferred items recorded for the next phase
```

Each arrow is a gate. You do not cross it until its condition is met.

---

## 1. Why it works this way

- **Context is built collaboratively, then frozen.** The agent starts knowing nothing about the intent. The conversation in Stage 1 — questions, options, a design presented in pieces — is how shared understanding is built. Once the human approves it, that understanding is written down (the spec) and everything downstream argues from it.
- **The spec is the authority; the plan is an argument for it.** When they conflict, the spec wins. Executors read both.
- **Subagents isolate context.** A coordinating session ("the controller") holds the plan, the ledger, and the cross-task history. Each unit of work goes to a **fresh subagent** given exactly — and only — what it needs. This keeps the controller's context clean for coordination, and gives every task fresh eyes.
- **Review is a separate seat from implementation.** The thing that built the code never certifies it. A different agent, with the requirements and the diff but not the implementer's rationalizations, does.
- **Decisions are made, not deferred.** A running plan does not stop to ask a human about every ambiguity. It decides, records the decision and its risk, and continues. Only a few classes of action actually halt it (§10).

---

## 2. Roles

| Role | Context it holds | Job | Never does |
|---|---|---|---|
| **Human** | the intent; final say on scope and integration | approves the design, approves the spec, picks how work integrates, validates the running result | — |
| **Controller** | spec, plan, ledger, all cross-task history | decomposes, dispatches, coordinates reviews, rules on conflicts, keeps the ledger | write task code itself; fix findings itself |
| **Implementer** | one task's brief + the interfaces it touches + global constraints | implement exactly that task, test-first, commit, self-review, write a report | dispatch anyone; touch other tasks' code |
| **Task reviewer** | one task's brief + report + diff + constraints | two verdicts — does it match the spec, is it well-built | dispatch anyone; mutate the tree |
| **Re-reviewer** | a findings list + the fix diff | verdict each finding addressed/not; flag new breakage in the fix diff only | re-review untouched code |
| **Final reviewer** | the whole-branch diff + the ledger's deferred/parked items | one broad merge review | — |

**One implementer at a time.** Parallel implementers collide.

---

## 3. Stage 1 — Idea to shared understanding

> **Run `superpowers:brainstorming`.** This section is what to hold in mind while you do — the classification call, the delegated-decisions rule, and the approval gate.

This stage is a conversation. No code, no scaffolding, no file creation until the human approves what you intend.

### 3.1 Classify the request first, and say the classification out loud

| Class | What it is | Output | Process |
|---|---|---|---|
| **Spike** | a feasibility question ("can we…", "is X possible", "quick and dirty is fine") | an answer / recommendation, not kept code | say what you'll try in 2–3 sentences, get a nod, investigate as cheaply as correctness allows, report findings; anything built is labeled throwaway |
| **Bounded** | a small, well-scoped change to code that already exists and can be read | working code | ask the few questions that matter, present a short design **in chat**, get an explicit "yes", implement directly (test-first); no spec doc, no plan doc |
| **Architectural** | a new project, a new subsystem, or a change that restructures how parts fit or alters shared interfaces | working code behind a full paper trail | the whole system below |

When unsure between two classes, take the heavier one. The ratchet is one-way: if hidden complexity shows up mid-task, stop, say so, and step up a class. Nothing steps down mid-task.

**"Too simple to need a design" is a trap.** Simple means a *short* design — two sentences — not *no* design and *no* approval. Unexamined assumptions cause the most wasted work on tasks that looked trivial.

### 3.2 Build understanding (architectural path)

1. **Explore the existing context** — files, docs, recent history — enough to ask good questions.
2. **Scope check.** If the "idea" is really several independent subsystems, say so now and help decompose into sub-projects, each with its own spec → plan → build cycle. Don't spend questions refining the details of something that needs splitting first.
3. **Ask clarifying questions one at a time.** One question per message. Prefer multiple-choice; open-ended is fine when it fits. Aim at: purpose, constraints, success criteria, what's explicitly out of scope.
4. **Propose 2–3 approaches with trade-offs.** Lead with your recommendation and why. Cut every feature that isn't needed.
5. **Present the design in sections**, each sized to its complexity (a sentence to a few short paragraphs). After each section, ask whether it's right. Be ready to go back.
6. **Design for isolation.** Every unit should answer, without reading its internals: *what does it do, how do you use it, what does it depend on.* If you can't change a unit's internals without breaking its callers, the boundary is wrong.

### 3.3 Delegated decisions

The human may hand back open design questions — "you decide the best ones, you know what I want." That is real delegation, not a blank cheque: decide each one the way you'd rule on a conflict (§12), and write the decision and its rationale into the spec's **Resolved decisions** section so it's visible and reversible. Delegation covers design detail inside the approved direction; it does not move the approval gate.

### 3.4 The approval gate

The stage ends with the human explicitly approving what you intend to build. Presenting the design and starting to build it in the same breath **skips the gate**. Wait for the "yes".

---

## 4. Stage 2 — The spec

> Still inside `superpowers:brainstorming` — it writes the design doc to `docs/superpowers/specs/` and commits it. This section is the section checklist and the self-review to apply before the human gate.

Write the approved design to a dated design doc and **commit it before planning starts**. Suggested location: `docs/specs/YYYY-MM-DD-<topic>.md`. The spec and every plan are committed to version control before their execution run begins — they are the durable record the run argues from.

**Typical sections:**

- **Goal** — one sentence.
- **Scope** — in, and explicitly out.
- **Architecture** — the components and how data flows.
- **Each major operation** — inputs, output shape, validation/guardrails, failure/fallback behavior, cost if relevant.
- **Data model** — the durable shapes; what's fixed vs. extensible.
- **UI / interface surface** if any.
- **Testing strategy** — what's unit-tested exhaustively, what's integration-tested, what's not tested and why.
- **Resolved decisions** — numbered, each with its rationale. This is where "we considered X and chose Y because Z" lives.
- **Module / file impact** — what gets created or touched.
- **Sub-project breakdown** — if this needs phasing (§5), the phases, each as a paragraph.

### 4.1 Spec self-review (do it yourself, no subagent)

- **Placeholders** — any "TBD" or vague requirement? Resolve it.
- **Internal consistency** — do any two sections contradict? Does the architecture match the feature list?
- **Scope** — one plan's worth, or does it still need splitting?
- **Ambiguity** — any requirement readable two ways? Pick one and state it.

### 4.2 Human review gate

> "Spec written and committed to `<path>`. Please review it and tell me if you want changes before we write the implementation plan."

Wait. On changes, make them and re-run 4.1.

---

## 5. Stage 3 — Phasing a large spec

A large spec is built as **sequential sub-projects**. Each one:

- leaves the product **working and usable** on its own,
- is **independently reviewable**,
- is divided **by responsibility / subsystem**, never by technical layer ("all the models", then "all the views" is the wrong cut).

**Split when:**

- the resulting plan would exceed roughly ten tasks, or
- it spans two subsystems that share no test surface, or
- one part could ship and deliver value before the other is even designed.

Each sub-project gets its own spec section (or its own spec), its own plan, its own execution run, its own branch, its own integration. A later phase may itself be split again once its plan is drafted and turns out too big.

Order phases so each depends only on what's already built. A common shape: **foundation (pure logic, no I/O) → persistence + core flow (offline, no external services) → external integrations behind the seams the previous phase left → secondary surfaces (history, admin, settings).**

---

## 6. Stage 4 — The plan (one per phase)

> **Run `superpowers:writing-plans`** (once per phase). This section is its intent restated — file-structure-first, task right-sizing, real code in every step, the `Interfaces` block, the placeholder ban, the self-review.

Write for a competent engineer who knows nothing about this codebase or domain and has questionable taste. Document everything they need.

### 6.1 Header

```
# [Phase] Implementation Plan

**Goal:** [one sentence]
**Architecture:** [2–3 sentences on the approach]
**Tech stack:** [key technologies]
**Spec:** [path — the plan argues from it; executors read both]

## Global Constraints
[the spec's project-wide rules, one line each, exact values verbatim:
 version floors, isolation/threading rules, what's in vs out of scope for
 this phase, "no network in tests", commit conventions, the exact
 commands that must pass at the end of every task and at phase end]
```

### 6.2 File Structure — before any task

List every file created or modified and its **single responsibility**. This locks the decomposition. Files that change together live together. Prefer small, focused files over large ones doing several jobs.

### 6.3 Task sizing

A task is **the smallest unit that carries its own test cycle and is worth a fresh reviewer's gate.** Fold setup, config, scaffolding, and docs into the task whose deliverable needs them. Split only where a reviewer could reasonably reject one task while approving its neighbor. Every task ends with something independently testable.

### 6.4 Task shape

```
### Task N: [Component]

**Files:**
- Create: exact/path
- Modify: exact/path:line-range
- Test:   exact/test/path

**Interfaces:**
- Consumes: [what earlier tasks produce that this uses — exact signatures]
- Produces: [what later tasks will rely on — exact names, parameter & return types]

- [ ] Step 1: Write the failing test        <real test code>
- [ ] Step 2: Run it, watch it fail         Run: <cmd>   Expect: FAIL "<reason>"
- [ ] Step 3: Minimal implementation        <real implementation code>
- [ ] Step 4: Run it, watch it pass         Run: <cmd>   Expect: PASS
- [ ] Step 5: Commit                        <exact commit command + message>
```

The **Interfaces** block is how an implementer who only sees its own task learns the names and types its neighbors use. Keep them identical across tasks — the same function called `flush()` in one task and `flushAll()` in another is a planning bug.

### 6.5 No placeholders — each of these is a plan failure

- "TBD", "implement later", "add error handling", "handle edge cases"
- "write tests for the above" without the test code
- "similar to Task N" — repeat the code; tasks get read out of order
- a step that says *what* without *how* (code steps need code)
- a reference to a type or function that no task defines

### 6.6 Plan self-review

1. **Spec coverage** — every spec requirement maps to a task. List gaps, add tasks.
2. **Placeholder scan** — kill every pattern in 6.5.
3. **Type consistency** — signatures and names in later tasks match earlier definitions.

### 6.7 Hand off to execution

Offer the human the execution mode: subagent-driven (fresh subagent per task, review between — the default here) or inline (batch execution in one session with checkpoints).

---

## 7. Stage 5 — Execution setup

> **Run `superpowers:subagent-driven-development`** — it owns Stages 5 through 9: workspace, ledger, pre-flight scan, the per-task implementer + reviewer loop, the fix loop, and the whole-branch final review. Its bundled `scripts/` (`sdd-workspace`, `task-brief`, `review-package`) are the tools referenced throughout. The sections below are the operating detail and the overlay's additions (carry-forward docs, ruling bookkeeping).

### 7.1 Isolated workspace

Work on a branch or worktree off the mainline — never directly on the mainline without explicit consent. Name it after the plan.

### 7.2 The ledger

A single file the controller owns, outside the committed tree (scratch). It is the **recovery map**: conversation memory does not survive compaction, but the ledger plus version-control history does. Controllers that lost their place and had no ledger have re-dispatched entire finished task sequences.

- First line names the plan it belongs to.
- On resume: tasks with a `Task N: complete` line are done — do not re-run them. Start at the first task without one. A task whose last line is a fix round is mid-loop.
- It records: the pre-flight scan, each task's dispatch (agent identity, model, base commit), each fix round, each `Ruling:`, each completion line.
- Anything that must outlive the run goes into a **committed** doc, because the ledger is scratch and does not travel.

### 7.3 Pre-flight conflict scan

Before dispatching Task 1, read the plan once and write a **table** to the ledger — not a verdict:

- one row per **pair of tasks that share a file or an interface**: what one produces vs. what the other consumes, and what you found;
- one row per **task's internal consistency**: do its own tests match its own code; do the files it creates match the files it later modifies.

Rule on every conflict the scan surfaces (spec is the authority), record each ruling beside its row, then dispatch Task 1. "The scan is clean" without the rows is not a scan.

---

## 8. Stage 6 — The task loop

### 8.1 Dispatch an implementer

1. **Record the base commit** (`HEAD` right now). The review package and every fix-round diff need it. Never use "the previous commit" — a multi-commit task would lose all but the last.
2. **Extract the task brief** — the task's full text to its own file. This file is the single source of requirements.
3. **Compose the dispatch**, containing only:
   1. one line on where this task fits in the project;
   2. the brief path — "read this first; it is your requirements, use its exact values verbatim";
   3. interfaces and decisions from earlier tasks that the brief cannot know;
   4. your resolution of any ambiguity you spotted in the brief;
   5. the report-file path and the report contract.
   - Exact values (numbers, strings, signatures, test cases) live **only in the brief**. Never make a subagent read the whole plan.
   - **Never paste session history or "state after tasks 1–3" into a dispatch.** A fresh subagent needs its task, its interfaces, and the global constraints. Nothing else.
   - Hand over context as **files**, not pasted text — anything pasted into a dispatch stays in the controller's context for the rest of the session.
4. **Record the implementer's identity** — fix rounds 1–3 resume it.

### 8.2 Choosing the model

Always specify it explicitly; an omitted model silently inherits the most expensive one available.

| Work | Model tier |
|---|---|
| The plan already contains the code to write (transcription + testing) | cheapest |
| Single-file mechanical change | cheapest |
| Small, complete spec, mechanical | cheap |
| Multi-file, integration concerns, working from prose | mid — this is the floor for prose work |
| Architecture, design judgment, broad understanding | most capable |
| The whole-branch final review | most capable, explicitly |
| Reviewers | scaled to the diff's size and risk |
| Fix-loop rounds 4–5 | one tier above the implementer that got stuck |

**Turn count beats token price.** The cheapest models often take 2–3× the turns on multi-step work and cost more overall. Cost and wall-clock scale with turns.

### 8.3 Batch same-shape work

Several tasks that are each the same one-line edit across files → **one dispatch** listing every file and its change, reviewed as one diff. Keep one-dispatch-per-task for work that needs its own judgment, tests, or review surface.

### 8.4 The implementer's four possible reports

- **Done** → package the diff, dispatch the reviewer.
- **Done with concerns** → read them first. Correctness or scope concern → resolve before review. An observation → note it, proceed.
- **Needs context** → provide it, re-dispatch.
- **Blocked** → diagnose: missing context → add it, same model; needs more reasoning → stronger model; too big → split; the plan is wrong → rule on the fix, record it, re-dispatch with the ruling. Never ignore an escalation; never make the same model retry unchanged.

If the implementer asks questions, answer completely before it proceeds.

### 8.5 The task review

Package the task's diff (commit list + stat + full diff with generous context) into one file. Dispatch a reviewer with **four inputs**: the brief, the implementer's report, the diff package, and the **global constraints copied verbatim from the plan/spec** — that block is the reviewer's attention lens.

The reviewer returns **two verdicts, both mandatory**:

- **Spec compliance** — met / not met (what's missing, extra, or built the wrong way, with line references) / "can't verify from this diff".
- **Code quality** — separation of concerns, error handling, DRY without premature abstraction, edge cases, whether tests check real behavior, whether this change made a file too big.

Controller hygiene writing the reviewer prompt:

- No open-ended directives ("check everything", "run stress tests if useful") without a concrete, task-specific reason.
- Don't ask it to re-run tests the implementer already ran on the same code.
- **Never pre-judge.** No "don't flag X", "at most minor", "the plan chose this". If you're writing those, you're trying to skip a review loop. Let the reviewer raise it; adjudicate it in the loop.

"Can't verify" items don't block the review, but the controller must resolve each one before completing the task (you hold the cross-task context). A confirmed gap becomes a failed spec review.

### 8.6 The fix loop — at most 5 rounds per task

Triggered by: spec failure, any critical or important finding, or a "can't verify" you confirmed as real.

Two things leave the loop immediately:

- **Minor findings** never enter it. Record each in the ledger and point the final review at that list.
- A finding that **conflicts with what the plan mandates** is the controller's to rule on (spec is the authority) and record *before* acting. Don't dismiss it because the plan says so; don't dispatch a contradicting fix without a recorded ruling.

A round is **one fix dispatch + one scoped re-review**.

- **Rounds 1–3:** resume the original implementer with the findings verbatim — its context is intact. If the harness can't message a live subagent, dispatch fresh with the brief path, report-file path, and findings (the report file is the shared memory).
- **Rounds 4–5:** fresh implementer, **one model tier up**, told "a prior implementer tried this N times; you own it now; read the report for what was tried." Three failed resumes usually means the implementer can't see its own problem.
- **Every round:** implementer fixes, re-runs the tests covering the changed code, appends a fix report (tests named, command, output), returns a short status. The controller confirms those three are present before dispatching the re-review.
- **The re-review is scoped:** diff from the head the previous review saw to now. The re-reviewer verdicts each finding *addressed / not addressed* and flags new breakage **in the fix diff only**. New critical/important breakage joins the open list. Anything outside the fix diff goes to the ledger, not the loop.
- **Record each round** in the ledger: how many addressed, how many open, the commit range.
- **Never fix findings in the controller session** — it pollutes coordination context and skips review.

**The breaker.** If round 5's re-review still leaves findings open, stop dispatching and adjudicate each one:

- reviewer is wrong or the point is contestable → **park it** with a ruling stating why the code stands;
- real but nothing depends on it → park it with a ruling stating it's real and deferred;
- **real and load-bearing** (a later task builds on it, or it exposes a plan defect) → rule on the smallest change that unblocks the dependent work, record the ruling, carry it into the next task's dispatch. Stop entirely only when every path forward is a guess.

Adjudicate **only at the cap.** Doing it earlier to end a loop is pre-judging under another name. Every adjudication is a ledger entry; a silent discard is forbidden.

### 8.7 Complete the task

Review clean, or every open finding parked-with-ruling at the cap → write the completion line (commit range, "review clean" or "K parked"). Mark it done. Never start the next task with unresolved critical/important findings that are neither fixed nor parked-with-ruling.

### 8.8 Execute continuously

Do **not** check in with the human between tasks. Run the whole plan. "Should I continue?" and progress recaps waste their time. Between tool calls, at most one short line of narration — the ledger and the tool results are the record.

---

## 9. Stage 7 — The whole-branch review

After the last task:

1. Package the **entire branch diff** (from where the branch left the mainline).
2. Dispatch **one** reviewer on the **most capable model**, pointed at the ledger's deferred-minor and parked lines so it can triage what must be fixed before merge. It classifies findings critical / important / minor and returns a verdict.
3. **One consolidated fix dispatch** — not one fixer per finding. Per-finding fixers each rebuild context and re-run suites; this is the single most expensive avoidable pattern in the whole system.
4. The fix brief is structured: a **critical** section, an **important** section, a **cheap extras** section ("while you're in these files"), and a **carry-forward** section (items explicitly deferred to the next phase — record only). It ends with a report contract.
5. **One** scoped re-review of the fix-wave diff.
6. Adjudicate residuals as in the breaker. **There is no second fix wave** — remaining load-bearing findings surface to the human at finish time.

### 9.1 Hand the human the rulings

Before deleting anything, collect **every ledger line containing a ruling** — pre-flight, fix loop, breaker — into one list for the human, in order, each with what it costs if wrong. This is the only place decisions you made on their behalf reach them. A ruling that dies with the scratch workspace was a secret decision.

Then delete the run's scratch workspace — version history is the record now.

---

## 10. Stage 8 — Finishing

> **Run `superpowers:finishing-a-development-branch`.** This section is its shape — verify on the real tree, confirm the base, present the options and wait, discard only on the explicit word.

1. **Run the full test suite on the exact tree you're about to integrate.** A green run earlier in the session only proves the tree it ran on. Failing → report and stop; the options come after green.
2. **Confirm the base** the work forks from. Integrating into the wrong base is expensive to undo.
3. **Present the integration options and wait** — typically: merge locally, or push and open a change request, or keep the branch as-is. The choice is the human's. Discarding the work happens **only** if they explicitly ask for it, in unambiguous words.
4. Execute the choice. If merging, re-run the suite on the merged result before deleting anything.
5. Clean up only workspaces this system created; leave everything else.

**Change-request description shape:** *why* (the root cause or motivation, with evidence — a stack trace, a failing assertion, the spec section) · *what changed* (a per-commit narrative, including any wrong turns and what superseded them) · *testing* (a before/after table) · *what's deliberately out of scope*.

---

## 11. Stage 9 — Acceptance and the next phase

- The human validates the running result against an acceptance checklist (usually carried in the final task's report).
- Bugs found here are triaged like any finding: reproduce, diagnose from evidence, fix on a branch, review, merge. **Don't ship a guess** — reproduce, read the actual failure, form a hypothesis, test *that*.
- **Deferred items are written to a committed carry-forward doc**, grouped by which future phase picks them up, each with the precise defect, the conditions it bites under, and a fix recipe. The next phase's plan reads that doc and folds the relevant items into its own tasks, then ticks them off.

---

## 12. Cross-cutting principle: rulings, not stalls

A running plan does not wait on a human for conflicts, ambiguities, plan defects, or a limit you'd otherwise have asked to exceed. **Decide, record, continue.**

Record every decision as:

```
Ruling: <what you decided> — <why> — <what it costs if wrong>
```

A wrong ruling costs rework the human can see and undo. A session parked on a question costs their whole day and buys nothing.

**Only these four things actually stop you:**

1. an irreversible or destructive operation,
2. a security-sensitive action,
3. a side effect outside the workspace that convention says you ask about first (integrating, publishing, pushing to a shared branch),
4. a plan so broken that every path forward is a guess.

Everything else is a ruling.

---

## 13. Cross-cutting principle: autonomy boundaries

When a human grants autonomy ("you drive, I'll check later"), it does **not** extend to the four halting classes above. In particular: no integrating, no publishing, no pushing to shared branches, no destructive operations — even under broad autonomy — unless that specific action was named. Approval in one context does not carry to the next.

---

## 14. Cross-cutting principle: model economics and waiting

- **Match the model to the work** (§8.2). The final review is the one place to always reach for the most capable model.
- **If a capable model hits a rate limit** mid-run, fall back to a mid-tier model for implementation and keep the capable one for the final review.
- **Never poll a wait interface with short timeouts.** While a subagent runs, do local work — ledger updates, packaging the next review, reading reports. Results arrive on their own.
- **When genuinely idle, wait in bounded stretches** (a few minutes), and between them post one status line and reconcile live children: list them, chase any that finished without reporting. A bounded stretch keeps almost all of a long wait's efficiency while guaranteeing a stuck or silently-dead child is caught in minutes, not hours.
- **A subagent can die silently.** If one goes quiet well past its expected duration, re-dispatch a fresh one rather than waiting indefinitely.

---

## 15. Cross-cutting principle: debugging discipline

When something is broken — a failing test you don't understand, a hang, a wrong result — **do not ship a guess.** A guessed fix that looks plausible costs a full review cycle and often a second bug.

1. **Reproduce it** deterministically first. If you can't reproduce it, you can't verify a fix.
2. **Inspect the actual failure state**, don't infer it. Read the real stack trace, the real assertion diff, the real logged values. For a hang or a freeze, sample the stuck process and read where it actually is — a process idle in its event loop looks completely different from one spinning or deadlocked, and the sampled call tree names the culprit.
3. **Form one hypothesis** that explains the evidence.
4. **Test that hypothesis** — the smallest change or probe that confirms or kills it — before writing the real fix.
5. **Expect the first hypothesis to be wrong sometimes.** When a fix doesn't take, go back to the evidence, not to a second guess. Two wrong fixes in a row means you're pattern-matching, not diagnosing.

**Driving a GUI for reproduction or acceptance:** an agent can usually operate a running app itself — capture the screen to an image, send coordinate taps / keystrokes through the OS accessibility or automation layer, and sample the process to tell "waiting for input" from "wedged." Map screen coordinates from the window's reported position and size. Know the harness's limits (synthetic input may not trigger every UI framework's gestures) and hand those cases to the human.

---

## 16. Cross-cutting principle: what gets written down where

| Artifact | Lifetime | Committed? |
|---|---|---|
| Spec | permanent | yes |
| Plan (per phase) | permanent | yes |
| Carry-forward doc (deferred items for later phases) | permanent | yes |
| Living status / handoff doc | permanent, updated every phase | yes |
| Ledger | one execution run | no — scratch |
| Task briefs, task reports, review packages, fix-wave brief/report | one execution run | no — scratch |

Rule of thumb: **if it must survive the branch or move between machines, it is committed.** The ledger and everything beside it are scratch, so anything durable (deferred findings, decisions with lasting consequences) is copied into a committed doc before the workspace is deleted.

Commit messages: imperative, plain, no automated co-author trailers, and no push or integration without the human asking.

---

## 17. Templates

### 17.1 Implementer dispatch

```
Subagent:
  model: <explicit tier>
  prompt: |
    You are implementing Task N: <name>, for the "<plan title>" plan.

    ## Task
    Read your brief first — it is your requirements, use its exact values
    verbatim: <path to task-N-brief>
    Work from: <repo root>. Branch: <branch> (checked out).

    ## Context
    <1–3 sentences: where this fits; what earlier tasks produced that you consume>

    ## Before you begin
    If the brief is unclear or you hit an error you can't resolve from it,
    ask before guessing.

    ## Your job
    1. Follow the brief's steps exactly (test-first where it says to).
    2. Gates: <exact test commands + expected results>.
    3. Commit with the message in the brief's final step.
    4. Self-review your diff (completeness / quality / discipline / tests).
    5. Write your full report to: <path to task-N-report>.
    Constraints: plain commit message, no co-author trailer, no push,
    no dispatching subagents, don't commit scratch files.

    ## Report
    Full detail to the report file. Then reply with only:
    - Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
    - Commit(s): short hash + subject
    - One-line test summary
    - Concerns, if any
    - The report file path
```

### 17.2 Task reviewer dispatch

```
Subagent:
  model: <tier scaled to diff size/risk>
  prompt: |
    Review one task's implementation: first whether it matches its
    requirements, then whether it is well-built. Task-scoped gate, not a
    merge review.

    ## What was requested
    Brief: <path>
    Global constraints that bind this task (verbatim from the spec/plan):
    <paste>

    ## What the implementer claims
    Report: <path>

    ## Diff under review
    Base: <hash>   Head: <hash>   Diff file: <path>
    Read the diff file once. Don't re-run git. Don't crawl the codebase;
    inspect outside the diff only to check a concrete risk you can name.
    Read-only — don't mutate the tree.

    Don't trust the report — verify claims against the diff. A stated
    rationale never downgrades a finding.
    Don't re-run tests the implementer already ran on this code.

    ## Output
    ### Spec compliance — met / not met (missing / extra / misunderstood,
        with line refs) / can't-verify items
    ### Strengths
    ### Issues — Critical / Important / Minor, each with file:line, what's
        wrong, why it matters, how to fix
    ### Assessment — Approved | Needs fixes, plus 1–2 sentences
```

### 17.3 Scoped re-review dispatch

```
Subagent:
  model: <cheap-to-mid for small fix diffs>
  prompt: |
    Re-review one fix round. Verdict each finding; inspect the fix diff for
    new breakage. Nothing else.

    Brief: <path>
    Findings under verification: <verbatim, one per bullet>
    Report (fix reports appended at end): <path>
    Fix base: <hash>   Head: <hash>   Diff file: <path>
    Read-only.

    ## Output
    ### Finding verdicts — each: ADDRESSED | NOT ADDRESSED, with file:line.
        "Attempted" is not addressed.
    ### New breakage in the fix diff — severity + file:line, or "none"
    ### Out-of-scope observations — non-blocking, for the final review, or "none"
    ### Verdict — all addressed & no new critical/important breakage, or list what's open
```

### 17.4 Consolidated fix-wave brief

```
# <phase> — final fix wave

The whole-branch review (see <final-review file> for exact file:line + fix
recipes) returned "<verdict>". This is the single consolidated fix dispatch.
Branch <branch>, head <hash>. Scope: <...>.
Global constraints still bind: <restate>. Small commits (per item or tight
cluster). At the end run: <exact gate commands>.

## REQUIRED — Critical
### F1 — <finding id>: <title>
<file — location>. <what's wrong + the failure scenario>.
Fix: <numbered recipe with real code>.
Test: <what to assert>.

## REQUIRED — Important
### F<n> — ...

## REQUIRED — cheap extras (you're in these files anyway)
- <one-liners>

## CARRY FORWARD to the next phase (record only — do NOT do now)
- <item> — <why> — <where it bites>

## Report
Full report to <fix-wave-report file>. Return to the controller only:
status, commit range first..last, the suite results on one line, anything
not completed.
```

---

## 18. Anti-patterns

| Rationalization | Reality |
|---|---|
| "This is simple, skip the design" | Simple → a two-sentence design and an explicit yes. That's where unexamined assumptions bite hardest. |
| "They obviously want it merged" | Integration is the human's call. Present options, wait. |
| "I'll fix this finding myself, dispatching is overhead" | Controller fixes pollute coordination context and skip review. Resume the implementer. |
| "One more round will converge" | Past the cap, rounds don't converge — the failure is structural. Adjudicate and route. |
| "This finding is clearly wrong, I'll drop it" | You adjudicate only at the cap, and every ruling is recorded. Silent discards are forbidden. |
| "Small fix, skip the re-review" | Unreviewed fixes are how regressions land. Every round ends with a scoped re-review. |
| "I'll paste the last three tasks' state so the subagent has context" | It needs its task, its interfaces, the constraints. Pasted history bloats context and misleads. |
| "The ledger is bookkeeping overhead" | It's what survives compaction. Without it, controllers re-run finished work. |
| "The whole suite passed earlier this session" | Run it on the exact tree you're integrating. A green run only proves the tree it ran on. |
| "Broad autonomy means I can merge/publish" | It never extends to irreversible, security-sensitive, or outside-the-workspace actions unless named. |
| "The final review will catch it" | The final review is one pass, not a safety net for skipped task reviews. |
| "Ship the fix, the diagnosis is probably right" | Reproduce, read the real failure, test the hypothesis. A guessed fix wastes a whole review cycle. |
