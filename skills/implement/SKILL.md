---
name: implement
description: Execute one ticket from a shaped bundle until its done-when passes — TDD at the spec's agreed seam, then verify, reconcile, and open the PR — or run a fix round against an open PR's review findings. Resolves bundle, ticket, PR, and mode from the current branch with no arguments; `<bundle id> [ticket NN]` or `fix <PR> [F<id> ...]` name a target explicitly.
argument-hint: "[bundle id] [ticket NN] | fix <PR> [F<id> ...]"
disable-model-invocation: true
---

# Implement

**Executor role**: turn one settled ticket into a verified, reconciled change set on its own
branch, ending at an open PR — or, once review has posted findings on that PR, turn its
verdict into a fix round. Everything decidable was decided upstream — the spec passed the
human's Plan gate, and in fix mode the reviewer already ruled each finding's tier and class —
so the ticket, or the verdict, is the full input. Stage definitions (verify, reconcile, where
review sits) live in [docs/agentic-workflow.md](../../docs/agentic-workflow.md); the
review-fix loop's shared contracts — markers, the verdict and fix-round schemas, tiers,
classes, loop state, the round cap — live in
[skills/review/references/protocol.md](../review/references/protocol.md). Neither is restated
here.

Two rules apply across every step of ticket mode:

**Test-first at the agreed seam.** The spec's `Testing Decisions` section names the seam — the
observable boundary the human confirmed during shaping (in a single-file bundle, the `Seam:`
line under `Done when`). Every test you write attaches there. The seam is a shaping decision:
you have no authority to move it, add one, or test below it at internals.

**Surface drift, don't absorb it.** When the bundle and the codebase disagree, the kind of
disagreement decides the move:

- _Factual drift_ — the spec or ticket describes code that has since moved, been renamed, or
  never existed: fix the spec or ticket first, then write the code against the corrected text,
  and say so in the PR. The spec is living documentation; code written against its wrong claim
  entrenches the error.
- _Decision drift_ — the seam won't attach, an AC is unreachable as written, a requirement has
  two viable readings, or the change would contradict a `docs/decisions/` record: stop and put
  the question to the human. Resolving it yourself silently re-runs the Plan gate without them.

## Process

### 1. Resolve mode and target

**Zero arguments resolve from the current branch first** (ID-7): a branch matching
`<bundle-id>/NN-<slug>` (single-file bundle: `<bundle-id>`) names the bundle and ticket, and
its open PR, if any, is the PR. On the default branch, resolve the bundle first — exactly one
in the in-progress tree, else exactly one shaped, else ask, don't guess — then the PR: exactly
one open PR on the bundle's branches drives mode inference; several → ask, listing each with
its state; none → ticket mode on the next unblocked ticket.

**Mode follows the PR's loop state** (BR-5, protocol reference): `fixes-pending` → fix mode
against that PR; any other state, or no PR → ticket mode. Explicit arguments override
inference: `/implement fix <PR> [F<id> ...]` forces fix mode against the named PR — trailing
finding IDs direct the round at named concerns or nits beyond the default blocker set (ID-6);
`/implement <bundle id> [ticket NN]` forces ticket mode, resolving the bundle with
`ls work/*/<bundle id>*` (a single `.md` file is the whole bundle, spec and ticket merged; a
directory bundle: read `spec.md` in full, then the ticket).

**Echo the resolved mode and target before acting.** If the resolved mode's state doesn't
call for action — fix mode on a PR with nothing pending, ticket mode with no unblocked ticket
— report the state and the next command and stop without side effects (BR-11).

**A verdict marker present on the PR with a block that doesn't parse, or missing a required
field, fails loud**: report what didn't parse and stop (BR-12). No marker at all is not an
error — it's `needs-review`, which routes to ticket mode.

Ticket mode continues at step 2. Fix mode is its own section, below step 7.

**Pick the ticket** (ticket mode): the one named in the invocation, otherwise the
lowest-numbered ticket with `status: todo` whose `depends_on` entries are all `done`. If the
named ticket has an unmet dependency, stop and report it — ticket order is part of the
approved decomposition, not yours to reshuffle.

### 2. Activate

**Move a fresh bundle to active**: if the bundle still sits under `work/shaped/`, `git mv` it
to `work/active/` — starting its first ticket is what "in progress" means.

**Read `docs/agents/git.md` and take the branch strategy it declares.** A missing file or
absent declaration line means `trunk`. A single-file bundle takes the `trunk` path in
either mode — its one PR already lands whole.

**Under `bundle-branch`, sync the bundle's integration branch before branching**: the
branch is `<bundle-id>/integration`, created from the default branch's head when the
bundle's first ticket starts; whenever it has fallen behind the default branch, merge the
default branch into it, so drift is paid per ticket rather than all at once at ship. A
conflict in that merge is decision drift: stop and put it to the human.

**Create the ticket's branch** — `<bundle-id>/NN-<slug>` (single-file bundle:
`<bundle-id>`) — off the default branch's head under `trunk`, off
`<bundle-id>/integration` under `bundle-branch`. One ticket, one branch, one session —
parallel tickets belong in separate sessions on separate branches.

**Set the ticket's `status: doing`.** Done when: you are on a fresh branch and the ticket
frontmatter says `doing`.

### 3. Read the code

**Read the files the ticket's Scope names, plus their colocated READMEs.** Where a README and
the spec disagree about the current system, the README wins — it describes what is; the spec
owns only what this change makes true. A path the ticket cites that doesn't resolve is
factual drift — handle it under the drift rule before writing anything.

### 4. Red-green loop

Work one acceptance criterion at a time, in the ticket's `Done when` order:

1. **Red** — write one failing test for the criterion, at the seam, phrased in the spec's
   Given/When/Then language. Run it and watch it fail; a test that passes before the change
   exists tests nothing.
2. **Green** — write the least code that makes it pass. Run that test file and the typechecker
   before moving on, not the full suite — the suite comes once, at verify.
3. Next criterion.

**Locked acceptance tests are read-only.** Where the bundle lists pre-authored test files,
run them as-is; a locked test that can't pass unmodified is drift, never an edit.

**Write unit tests below the seam freely** — but they are part of the diff under review's
judgment, not verification, so they earn nothing at the verify step.

**Refactor only inside the ticket's scope, only on green.** Wider cleanup you're tempted into
is a backlog line, not a commit — wide refactors get shaped into their own expand → migrate →
contract tickets.

Done when: every acceptance criterion in the ticket's `Done when` has a test at the seam that
failed first and passes now.

### 5. Verify

**Run every `Done when` line and the repo's own checks** (full test suite, lint, typecheck —
whatever the consuming repo's CI runs). Every line passes, exactly as written — a line that
won't pass goes back through the drift rule, not into a weakened test or a reworded ticket.

### 6. Reconcile

**Repair what the diff made stale, in the same branch**: colocated READMEs now describing
old behavior, glossary entries the change renamed or redefined, `spec.md` where implementation
corrected it, and remaining tickets this ticket invalidated — amend them, don't leave them to
mislead the next session. Done when: no document in the repo describes the pre-change system.

### 7. Close out

**Set `status: done`** — legitimate only while every `Done when` line holds.

**Commit, push the branch, open the PR** — commit messages and the PR title both follow
the conventions in `docs/agents/git.md`, and the PR targets the declared mode's branch:
the default branch under `trunk`, `<bundle-id>/integration` under `bundle-branch`
(single-file bundle: the default branch either way). Reference the bundle by permalink, never bare path —
the bundle dies at ship; the permalink survives it. The PR body carries the verify results and
names what reconcile touched, so review can judge the reconcile half's honesty.

**Print the next command** (BR-11): `/review <PR>`.

**Stop there.** Review is the next stage and a human gate: no merging, no self-review, no
starting the next ticket — the next ticket gets a fresh session.

## Fix mode

Runs fresh on the PR's branch, entered on `fixes-pending` state or explicitly via
`/implement fix <PR> [F<id> ...]` (ID-6). Markers, the verdict and fix-round schemas, tiers,
classes, and the round cap are defined once in
[the protocol reference](../review/references/protocol.md); this section states only the
procedure.

1. **Cap check** (BR-9). Count fix-round reports already posted on this PR. At 3, refuse a
   fourth: report the unresolved findings and what each round tried, and stop — the human
   takes over. The cap never triggers a merge and is never silently continued past.
2. **Checkout.** Fetch and check out the PR's branch. The branch only moves forward from here
   — no force-push, no rewrite of pushed commits (BR-14); this round adds commits only.
3. **Select findings** (BR-6). Every open `blocker` from the latest verdict, plus any concern
   or nit named as a trailing argument (ID-6).
4. **Work each selected finding** (BR-3, BR-7). `mechanical` — fix it. `decision` — do not
   attempt a fix; escalate. A fix round may escalate a mechanical finding to `decision`, never
   the reverse, and never resolves a `decision` finding itself. An escalated finding is only
   usable by a later round once the human's ruling is recorded in the bundle (a spec
   amendment) or a decision record — never only in chat; until then it stays escalated. Nits
   touched by the same code may be batch-fixed while already there; concerns and nits not
   selected are reported `deferred`.
5. **Re-verify.** Re-run the ticket's affected done-when lines plus the repo's checks. A fix
   that cannot reach green is dropped and its finding escalated, not pushed.
6. **Push and report.** Push only once the branch re-verifies green. Copy the verdict's
   `backlog` list into `work/backlog.md`. Post one fix-round report (ID-3) giving every open
   finding ID an explicit status — a finding left out is a protocol violation the next review
   flags.
7. **Close out.** Print the next command (BR-11): `/review <PR>`. Stop there — re-review is
   the next stage and a human-triggered gate, same as after a ticket's PR.
