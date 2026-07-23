harness.md

A single-file agent harness for frontend work in an existing codebase.

Drop this in your repo root. Reference it from CLAUDE.md with one line:

Follow the harness in harness.md for any feature work. Do not skip Phase 1.

Then start a feature with:

Run the harness in harness.md for: <one to four sentence feature request>
Why this exists

Given a bare prompt, an agent will produce something that looks right and is quietly wrong in ways review does not catch. It will invent an interface that already exists, ignore the shared component every other screen uses, and pick a default that is technically reasonable and semantically false. An empty grade cell becomes 0 instead of "not entered", a teacher saves a half-finished sheet, and thirty report cards go home saying something untrue.

You cannot prompt your way out of that. The fix is structural: split the work across agents that each do one thing, put a human at the point where a misunderstanding is still cheap, and separate the agent doing the work from the agent grading it.

Adapted from Prithvi Rajasekaran's Harness design for long-running application development.

Setup

Create .harness/ for working files and add it to .gitignore. Every agent communicates by writing files there. Nothing is passed agent to agent directly, because everything needs to be inspectable afterwards.

Fill this in before the first run. The evaluator runs these commands literally.

install:   [command]
dev:       [command]
lint:      [command]
typecheck: [command]
format:    [command]
test:      [command]
e2e:       [command]

House rules that are worth failing a review over. Replace with yours.

Design tokens only. No literal colour values, no magic spacing numbers.
Reuse before you build. If a component nearly fits, extend it.
Semantic HTML. A div with an onClick is a bug.
Every memoisation carries a stated reason.
Store subscriptions read the narrowest slice that works.
No any, no assertion used to silence a real problem.
Tests query by role and accessible name.
Phase 1: Plan

Two agents, run in parallel. The planner waits on the researcher's file.

Agent: Researcher

Tools: read-only. Writes: .harness/research.md. Does not plan.

Survey the codebase. You exist because a planner working alone designs features as if the repo were empty, and will happily spec a new table component next to the three that already exist.

Produce:

Existing surface. Components, hooks, and utilities that already do part of this. File path plus one line each. If something is close but not exact, say what would need to change.
Architecture and conventions. How this area is structured. Where state lives and the current rule for local vs global. File layout, naming, import style. Quote the pattern, do not describe it abstractly.
Design tokens and primitives available. Names, not descriptions.
Test conventions. How tests here are written and what they assert.
Landmines. Slow paths, unstable props, migrations in progress, places where the type says one thing and the runtime does another.
Open questions. Things the code alone cannot answer, specific enough that a human can reply in one line.

Fails if: you recommend an approach, name a file without its path, or stop at the first plausible match.

Agent: Planner

Tools: read plus write to .harness/. Writes: .harness/plan.md.

Read .harness/research.md first. If it is missing, stop.

Expand the request into a spec. Specify what must be true when this is done. Do not specify how to build it.

That line is not a style preference. A wrong technical guess here does not stay contained, it gets baked into the spec and everything downstream is faithfully wrong.

"Unsaved edits survive a page refresh" is a spec.
"Store the draft in localStorage under a class-scoped key" is not.
"The table renders once per batch of incoming updates" is a spec.
"Wrap the row in React.memo" is not.

One exception: anything the researcher found. If an existing component or store slice must be used, name it. Reuse is a deliverable.

Produce:

Problem. What the user cannot do today, in two or three sentences. If you cannot state it, the request is too vague. Say so instead of guessing.
Scope. What is in, and an explicit list of what is out. The out list is the more useful one.
Reuse. What this is built from, taken from the research file.
User stories. Behaviours, not screens.
Acceptance criteria. Numbered and testable. The evaluator grades against this list, so write each one so a disagreement about whether it passed is settled by looking, not by discussion.
States. Loading, empty, error, partial, success. What each shows and what triggers it. If one does not apply, say why. Be explicit about the difference between empty and zero, absent and false, not-yet and none.
Accessibility. The keyboard path as a sequence. What takes focus first, the tab order, what escape does, what gets announced.
Risks and open questions.

Be ambitious about scope and conservative about detail. A thin spec produces a thin feature and nothing downstream will expand it.

The human checkpoint

Stop here. Hand me .harness/plan.md. Do not write code until I approve it.

This is the part that pays for the whole harness. Reading a plan takes four minutes. Reading the diff from an agent that misunderstood the ticket takes an hour, and by then the tokens are already spent.

When you read it, check the things agents get wrong rather than the things they get right:

Does it solve the problem you actually have, or a neighbouring one?
Is every acceptance criterion testable, or are some of them opinions?
Does it reuse what the researcher found, or has it quietly reinvented it?
Are all five states specified, including the semantically awkward ones?
Is the keyboard path complete, or does it stop at "should be accessible"?
Has the planner leaked implementation detail it should not have?

Edit it directly. It is your file now.

Phase 2: Build

Only after explicit approval.

Contract first

Before any code, propose what you will build and how each acceptance criterion will be verified. Get that checked against the plan, then write .harness/contract.md. The plan is deliberately high level; this is the step that bridges user stories and testable implementation.

Agent: Generator

Implement against the contract and the plan. Not beyond it.

If you think of something worth adding that is out of scope, write it to .harness/out-of-scope.md and keep going. Scope creep from an agent is harder to catch in review than scope creep from a person, because it all arrives at once and it all looks deliberate.

Commit as you go, one logical change per commit.

Agent: Test writer, running alongside

Write tests from .harness/plan.md, not from the implementation. Read the code only for selectors, props, and module paths. Never read it to decide what the correct behaviour is. If the code and the plan disagree, the plan wins and you write the failing test.

That constraint is the whole value. A test written by reading the implementation asserts that the code does what the code does.

One test per acceptance criterion, named so the number is visible.
Query by role and accessible name. If you cannot find an element that way, that is an accessibility failure and you report it rather than reaching for a test id.
Every state gets a test, not just success.
The async edges: unmount mid-request, out-of-order responses, rapid repeated triggers.
The keyboard path as a real test, tabbing in order.
A criterion you cannot test gets written failing, with a note.

Fails if: snapshots stand in for behaviour, assertions reach into internals, or mocking is heavy enough that the test would pass with the feature deleted.

Phase 3: Grade
Criteria

The generator and the evaluator get this same list, up front. That matters. An agent that knows what it will be graded on produces better work before any feedback exists.

Score 1 to 5. Any criterion below its threshold fails the round, regardless of the others. No averaging a failure away behind four good scores.

Criterion	Weight	Threshold
System fit	3x	4
Render behaviour	3x	3
Accessibility	2x	4
Correctness and states	2x	4
Craft	1x	3

The weights go where the model is careless, not where you personally care most. Claude arrives competent at craft and careless about render cost and keyboard access, so those are weighted accordingly. Re-tune when you change models.

System fit. Does this look like someone who works on this codebase wrote it? Reuse over rebuild. Tokens, not hex values. Follows the layout, naming, and state conventions already in the directory it lives in.

This is the inverse of the originality criterion in the original post, and the inversion is the point. Generating a standalone design, template defaults are the failure mode. Inside a product that already exists, inventing a novel pattern is the failure mode. A 5 here looks completely unremarkable.

Render behaviour. How many times does this render, and why? State the count for the primary interaction path, or say you could not measure it. Every memo carries a stated reason; memo on a cheap component that re-renders anyway is a fail, not a win. Subscriptions read the narrowest slice. No effect that should be derived state, no effect firing every render on an unstable dependency. Keys are stable and not array indices. For anything streaming, describe behaviour at ten updates a second, not just at rest.

Accessibility. Test it with the keyboard only and report what breaks. Semantic elements over clickable divs. Complete keyboard path, sensible tab order, focus visible at every stop, escape closes what it should. Focus managed on mount, unmount, and route change; modals trap and return it. Accessible name on everything interactive. Contrast at AA. Live regions for content that appears without user action.

Correctness and states. Loading, empty, error, partial, success, all reachable. Slow networks, failed requests, unexpected payload shapes, a null where the type promised a value. Async races: out-of-order responses, unmount mid-request, rapid repeated triggers. Cleanup for subscriptions, timers, listeners, streams. Types describe reality.

Craft. Spacing and typography on the system scale. No dead code, no commented-out blocks, no leftover console statements. Names say what the thing is. Baseline competence; failing means broken fundamentals.

Agent: Evaluator

Tools: read plus bash. Never fixes what it finds.

The moment you start repairing, you are the generator and this harness has one agent instead of two, which defeats the point. File findings precisely enough that someone else fixes them in one pass, and move on.

Your default failure mode is finding a real problem, deciding it is not a big deal, and approving anyway. You will also test the obvious path and call it thorough. Both are the reason you exist. When you catch yourself writing "minor" or "not blocking" or "acceptable for now", go back to the threshold table. Being generous here is not kindness, it is how the broken version shipped.

Verify everything yourself. Do not take the generator's word for anything.

git diff against the base
the lint, typecheck, format, and test commands above
walk the running feature, including the keyboard-only path

Write .harness/review-<n>.md:

## Verdict: PASS | FAIL

## Scores
| Criterion | Score | Threshold | Pass |
Weighted total: N

## Failures
- CRITERION: what was required
- FINDING: what actually happens, with file:line
- EVIDENCE: the command run or the interaction performed
- FIX: one sentence, specific enough to act on without investigation

## Acceptance criteria
Numbered, each met or not met, each with evidence.

## Not blocking
Genuinely outside the criteria. Keep it short. If this section is longer
than the failures section, you are being generous.

Rules: no finding without a path and line or a reproduction sequence. No score without evidence; "looks fine" is not evidence. Probe edge cases before signing off. A criterion you could not verify is scored unverified and fails the round.

Loop

On FAIL, fix the listed failures and grade again. Three rounds maximum. On the third failure, stop and hand me the review instead of trying again, because three failures usually means the plan was wrong, not the code.

Phase 4: Automation

The agent runs the pipeline itself and works through what it reports. Lint, types, formatting, tests. Warnings get resolved, not handed back.

Do not report the work as done because the tests pass. Report the verdict.

Calibration

This is the part that decides whether the harness works, and it is the part only you can write.

Out of the box the evaluator grades generously. Fix that with examples, not adjectives. Keep a section here with three or four real diffs from your own repo, the score you would give each, and why.

### Example 1
Diff: [paste or link]
Scores: system fit 2, render 4, a11y 3, correctness 4, craft 4
Verdict: FAIL
Reasoning: [why you scored it this way, in your words]

Every time the evaluator's judgement diverges from yours, add that case here. Expect several rounds before it grades the way you do. This is most of the work.

Maintenance

Every piece of a harness encodes an assumption about something the model cannot do on its own, and those assumptions go stale as models improve. The original post built context resets for one model and deleted them for the next.

When a new model lands, remove one piece at a time and see what actually degrades. Test these first, in this order:

The contract negotiation step
The researcher
The evaluator on smaller features

Keep whatever still earns its cost. Delete the rest.
