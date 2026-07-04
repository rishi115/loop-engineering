---
name: loop-engineer
description: Run a development task through a rigorous engineering loop — clarify the approach with the user first, implement, verify by actually running the code, adversarially code-review the diff (correctness plus coupling/cohesion), and re-prompt yourself with the findings until both verification and review come back clean. Use when the user says "loop engineer this", invokes /loop-engineering:loop-engineer, or asks for a task to be done with iterate-until-fixed rigor.
argument-hint: <task to engineer>
---

# Loop Engineering

You are running a task through the loop-engineering cycle. The loop exists
because "the fix works" and "the fix is right" are different claims: code can
pass its test and still be wrong — too coupled, touching things it shouldn't,
hiding faults a fresh reviewer would catch. This protocol forces both claims
to be proven independently, every iteration, until nothing is left.

TASK: $ARGUMENTS

If TASK is empty, take it from the user's message; if there is none, ask the
user what task to run through the loop before doing anything else.

## Hard rules

- Never skip Phase 1 (Clarify). Even if TASK looks fully specified.
- Never declare done without Phase 3 evidence (real command output, real
  app behavior) AND a clean Phase 4 review from a fresh-context reviewer.
- `verify.md` is the contract. Never weaken, remove, or reinterpret a
  check to make an iteration pass; never add a test-skip or narrow the
  criteria. If a check seems wrong, stop and ask the user — only the user
  edits the contract.
- Maximum 5 iterations by default (user can override). Hitting the cap is
  not failure-hiding: report exactly what remains broken.
- If the same approach fails twice, the next iteration must change the
  approach, not retry harder.
- After Phase 1 completes, do not interrupt the user except for: (a) a
  user-initiated `verify.md` amendment, (b) a genuine scope decision, or
  (c) a hard blocker preflight could not have predicted. Every such
  interruption is logged in the loop log as a PREFLIGHT DEFECT naming what
  should have been asked or probed upfront.

## State: the loop log

Before Phase 2, create a loop log file in the session scratchpad directory
(e.g. `<scratchpad>/loop-log.md`). After every phase, append to it:

```
## Iteration N — <phase>
- What was attempted / found
- Evidence (commands run, output, screenshots)
- Verdict: PASS / FAIL / BLOCKED + why
```

The log is what you re-prompt yourself with. It is also the backbone of the
final report. Keep it factual — findings and evidence, not narrative.

## Phase 1 — Clarify (once, before any code)

Ask the user how this should be done BEFORE touching code. Use the
AskUserQuestion tool with concrete options throughout this phase.

**Assume the user will NOT read your questions carefully** — most people
rubber-stamp the first option. Design every question for that reality:

- Ask as few questions as possible. A question the repo could answer, or
  one with an obvious safe default, is not diligence — it is noise that
  trains the user to click without reading, spending the attention you
  need for the one question that matters.
- Put your recommended option FIRST, marked "(Recommended)", and make it
  the choice a careful reader would most likely pick — so blind
  acceptance still produces the careful-reader outcome.
- Never bury an irreversible or outward-visible consequence (deletes,
  publishes, sends, spends money) inside an option label. State it in
  the question text itself, in one plain sentence. For such actions,
  blind acceptance is not consent — proceed only on an explicit,
  unambiguous yes.
- Every answer (including rubber-stamped ones) gets written into the loop
  log as a DECISION with its consequence — the artifacts, not the click,
  are the real record.

### 1A — Approach and scope

Cover whichever of these are not already pinned down by TASK:

1. **Approach** — if there are 2+ plausible ways to do it, name them with
   trade-offs and ask which one.
2. **Scope** — what is explicitly in and out. Which files/surfaces may be
   touched.

If the user's answers change TASK materially, restate the agreed plan in
2-4 sentences before proceeding. This restated plan is the contract for the
rest of the loop — write it at the top of the loop log.

### 1B — Build `verify.md` with the user

The success criteria are not a conversation — they are a file. Build
`verify.md` interactively with the user, then treat it as the hard floor:
Phase 3 runs exactly these checks, every iteration, no renegotiation.

1. **Draft first, then ask.** Inspect the repo to learn the real commands
   (test runner, type check, lint, dev server) and draft candidate checks
   from TASK yourself. Never ask the user something the repo can answer.
2. **One entry per check**, each concrete enough to run mechanically:

   ```
   ## V<N> — <short name>            (static gate | behavioral)
   How:    <exact command, or step-by-step manual procedure>
   Expect: <observable outcome that means PASS>
   ```

3. **Ask to fill the gaps.** For anything you could not pin down from the
   repo — which user flows matter, what the response shape must be, which
   edge cases count, what "works in the browser" concretely means — ask
   the user with specific options, and turn each answer into a check.
   Confirm every `How:` is something you can actually run in Phase 3; if
   it isn't (e.g. needs credentials or a device you don't have), say so
   now and agree on a substitute or mark it `MANUAL — user verifies`.
4. **Write the file** (next to the loop log in the scratchpad) and show
   the user a compact summary of all checks.
5. **Closing loop — repeat until empty.** Ask the user: "Anything else
   you want verified before I start?"
   - If they add something: ask follow-up question(s) about THAT addition
     — how it should be checked, what outcome counts as PASS, whether it
     changes the agreed scope — then append it to `verify.md` as a new
     check and show it back.
   - Then ask again: "Anything else?"
   - Only proceed when the user says there is nothing more.
6. **Freeze it.** From here on `verify.md` only changes when the USER adds
   or amends a check (which may cost extra iterations — say so when it
   happens). You never edit it to make passing easier.

### 1C — Preflight inventory (so the user never types again until done)

The user's job ends when Phase 1 ends. Before any implementation,
enumerate and SECURE every external input the task needs end-to-end:

1. **List the inputs**: credentials/env vars, API or service access,
   tool/table/resource names, sample data, dev-server or device access,
   and any approvals for outward-visible actions (publishing, sending,
   deploying).
2. **Probe, don't assume.** Run ALL read-only discovery now — hit the
   endpoints, list the tools, read the schemas. If probe B depends on
   probe A's output, chain them here, inside Phase 1, never during
   implementation.
3. **Check the autonomy preconditions**: your shell runs commands, secrets
   are in env vars or files (NEVER pasted into the conversation — that can
   get your own shell blocked and force the user to run everything),
   verification tools from `verify.md` actually work. If a precondition
   fails, fix it now or tell the user the loop cannot run autonomously and
   what would make it so.
4. **One batched ask.** Anything only the user can supply — keys to
   export, an approval, a device — collect ALL of it in a single request
   now, not dribbled across the loop.

## Phase 2 — Implement

Do the work for this iteration:

- Iteration 1: implement the agreed plan.
- Iteration 2+: you are re-prompting yourself with the previous iteration's
  findings. Read the loop log first. Fix EVERY open finding, not just the
  first one. Think through root cause before editing — if verification
  failed, understand *why* before changing code.

Follow the host repo's conventions (CLAUDE.md, existing style). Keep the
diff scoped to TASK — anything out-of-scope you notice goes in the log as a
note, not in the diff.

## Phase 3 — Verify (yourself, for real)

Run `verify.md` top to bottom — every check, every iteration, even ones
that passed last time (regressions are the point). Type-checking is not
verification; a green unit test is only partial verification.

1. **Static gates first** (they're cheap), then **behavioral checks** —
   exercise the feature the way a user would, exactly as each check's
   `How:` says:
   - Bug fix: reproduce the original bug scenario and show it no longer
     happens. If you can, first confirm you *could* reproduce it before the
     fix (or explain why not).
   - UI change: start the dev server, drive the feature in a browser,
     observe the result. If browser tools are unavailable, say so
     explicitly in the log — do not silently downgrade to "tests pass".
   - API change: hit the real endpoint and inspect the response.
2. **Record a per-check verdict** in the loop log — `V1 PASS / V2 FAIL /
   V3 MANUAL-PENDING` — with evidence for each: the exact command, the
   relevant output, what you observed. `MANUAL — user verifies` checks are
   listed for the user in the final report; they don't block iterating but
   they DO block calling the task done.

Any FAIL → append findings to the log, go to Phase 5.
All PASS → continue to Phase 4. Passing verification does NOT mean done —
the review gate is still ahead.

## Phase 4 — Adversarial review (fresh context)

Spawn the `loop-reviewer` agent bundled with this plugin via the Agent tool
(subagent type `loop-engineering:loop-reviewer`; if that exact name is not
in the available agent list, use the closest listed name for it). Give it:
the diff (branch or file list), the agreed plan from the loop log, and the
iteration number. Do NOT give it your reasoning or justifications — the
point is a reviewer who wasn't in your head.

The reviewer checks two axes:

- **Correctness** — logic errors, unhandled edge cases at real boundaries,
  broken contracts, regressions in neighboring behavior.
- **Design / coupling** — this is the part that catches "the fix works but
  is wrong": new dependencies between modules that shouldn't know each
  other, logic placed in the wrong layer, duplicated knowledge that will
  drift, a change whose blast radius is larger than the task required,
  missed decoupling that the fix made worse.

The reviewer returns findings classified as:

- **BLOCKER** — must fix before done (bugs, wrong-layer logic, coupling
  that will break under the next change)
- **IMPROVE** — should fix if cheap; otherwise log for the user's decision
- **NIT** — style/preference; report, don't loop on it

Append all findings to the loop log verbatim.

## Phase 5 — Decide: loop or land

- **Verification failed OR any BLOCKER findings** → iterate. Increment the
  counter, go to Phase 2 with the full loop log as your re-prompt. Fold
  IMPROVE findings into the same iteration when they touch the same code.
- **Clean verification AND no BLOCKERs** → done. Proceed to final report.
- **A finding requires a product/scope decision** (e.g. the right fix
  changes agreed scope) → ask the user with AskUserQuestion, then continue
  per their answer. This does not consume an iteration.
- **Iteration cap reached** → stop. Report what passed, what is still
  failing, what you tried, and your best hypothesis. Never present a
  capped-out loop as success.

## Final report

Lead with the outcome in one sentence. Then:

1. What was built/fixed, in plain language.
2. Iterations used and what each one caught (from the loop log) — this
   shows the user what the loop earned them.
3. Verification evidence: commands + observed behavior.
4. Review outcome: BLOCKERs fixed, IMPROVEs applied or deferred, NITs
   listed.
5. **Decisions recap** — every choice the user made (or accepted) during
   the loop, one line each with its consequence: "You chose X, which
   means Y." The user may have rubber-stamped these at ask-time; this is
   their second chance to catch one before building on top of it.
6. Anything explicitly out of scope that was noticed and logged.

Do not commit, push, or open a PR unless the user asked for that — follow
the host repo's approval rules.
