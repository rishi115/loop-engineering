# loop-engineering

A Claude Code plugin that runs a task through an iterate-until-truly-done
engineering loop:

1. **Clarify** — asks you how it should be done (approach, scope) before
   touching code, then builds a `verify.md` contract with you
   interactively: it drafts runnable checks from the repo, asks questions
   to complete each one, and keeps asking "anything else you want
   verified?" — with follow-up questions on each addition — until you say
   done. That file becomes the frozen hard floor for every iteration.
2. **Implement** — does the work.
3. **Verify** — actually runs it: type check + tests, then exercises the
   change for real (browser for UI, real endpoint for API, bug repro for
   fixes).
4. **Adversarial review** — a fresh-context `loop-reviewer` subagent reviews
   the diff for correctness *and* design faults (coupling, wrong-layer
   logic, blast radius) with none of the implementer's bias.
5. **Loop** — verification failures and BLOCKER findings are fed back as the
   next iteration's prompt. Repeats until both gates are clean, or the
   iteration cap (5) is hit — in which case it reports honestly what
   remains.

## Try it without installing

```bash
claude --plugin-dir ~/loop-engineering
```

## Install permanently

Inside Claude Code:

```
/plugin marketplace add ~/loop-engineering
/plugin install loop-engineering@rishikesh-plugins
```

## Use

```
/loop-engineering:loop-engineer <task>
```

or just say "loop engineer this: <task>" — the skill also triggers from
natural language.

After editing the plugin, run `/reload-plugins` in an open session to pick
up changes.
