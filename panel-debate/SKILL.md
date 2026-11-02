---
name: panel-debate
description: "Multi-model adversarial review + debate + revision-plan flow. A cheap prep seat reads the files once and builds a context pack; then several premium debaters review blind, ONE AT A TIME, from that pack; a head judge verifies findings against the real code, catches invented findings, and names every disagreement; contested points go through sequential rebuttal rounds; a cheap model writes the plain-English summary. Sequential and token-efficient by design, never a parallel fan-out. The round count is IN the trigger phrase: \"debate one\" / \"debate 1\" (1 rebuttal round), \"debate two\" / \"debate 2\", \"debate three\" / \"debate 3\", \"debate four\" / \"debate 4\" (max). Bare \"debate this\" / \"debate\" = 1 round. Also fires on \"panel debate\", \"tear this apart with the panel\", \"panel this\", \"multi model review\", \"cross-examine this\", \"review and debate\", \"full panel\", \"audit and debate\". Also usable as /panel-debate <target> [--rounds N]."
---

# Panel Debate

Runs a deterministic engine script (`panel_debate.sh`) for a multi-model adversarial
review + debate + revision plan. This skill builds the brief, calls the script, and
reads the output back. The script owns seat order, rounds, and logging. None of that
is model-improvised.

## Setup (one time)

1. Put this skill at `~/.claude/skills/panel-debate/` (or your project's `.claude/skills/`).
2. Put the engine scripts in `<skill-dir>/scripts/`: `panel_debate.sh` and `panel_lib.sh`
   (the shared watchdog with `with_timeout` / `write_timeout_status`).
3. Install and log in to the CLIs for the seats you want to use:
   - `claude` CLI (Anthropic login) for the Claude seats.
   - `codex` CLI (ChatGPT login) for the GPT seats.
   - Optional text-only seats call a chat-completions API directly. Export the key
     only for that script's process, never globally: `GLM_API_KEY`, `XAI_API_KEY`,
     `OPENROUTER_API_KEY`.
4. Set `PANEL_SCRIPT` to the engine path, or use the default
   `<skill-dir>/scripts/panel_debate.sh`.

Any seat can be dropped or swapped with the env overrides below. Model IDs in this
doc are defaults; use whatever your accounts have access to.

## Seat map (defaults)

| Step | Seat | Default model | Transport |
|---|---|---|---|
| R0 prep (grunt) | Context gatherer | Claude Sonnet, effort low (`PREP_ENGINE=luna` swaps to a cheap GPT model) | claude CLI / codex CLI |
| R1 debater A | Blind reviewer | Claude Opus, effort high | claude CLI |
| R1 debater B | Blind reviewer | GPT "Astra" tier, effort high | codex CLI |
| R1 debater C | Blind reviewer | GPT "Sol" tier, effort high | codex CLI |
| R1 debater D | Blind reviewer, TEXT-ONLY (optional) | GLM | direct API |
| R1 debater E | Blind reviewer, TEXT-ONLY (optional) | Grok | direct API |
| R1 debater F | Blind reviewer, TEXT-ONLY (opt-in only) | Kimi via OpenRouter | OpenRouter API (paid credits) |
| R2 + rebuttals | Head judge | Claude Opus, effort high | claude CLI |
| Final | Summary | Claude Sonnet, effort low | claude CLI |

Text-only seats have no file access. They work from the brief and the R0 context
pack and mark file-dependent claims UNVERIFIED. The judge's check against the real
code still applies to everything they say.

`PANEL_DEBATERS="opus astra sol"` gives a lean 3-seat panel.

### Why this is token-efficient

1. **Read once, cheaply.** The prep seat reads the artifact once and writes a capped
   context pack (verbatim excerpts with `path:line`, flow, facts checked, gaps).
   Every premium seat, including the judge and every rebuttal, starts from that pack
   and opens a file only to verify a specific claim.
2. **Sequential, not parallel.** Debaters run one after another. That does not save
   tokens per seat, but it avoids a machine-wide burst and plan-cap spikes.
3. **Early convergence.** If the judge finds nothing contested, rebuttals never run.
4. **Split logins.** Claude seats ride the Anthropic login; GPT seats ride the
   ChatGPT login. One capped plan cannot take out every debater.

### The self-judging guard

If the Opus model is both debater A and the judge, that is a real conflict. The judge
brief names Reviewer A as its own model, forbids deference to it, and requires a
COVERAGE line on whether A earned its seat. If a verdict looks like a model
rubber-stamping itself, the guard failed. Say so and re-run with a different judge
(`JUDGE_MODEL=<other model>`). The guard switches off automatically when the judge
is not the Opus seat. In that case the new judge may be judging its own review, and
no guard covers that, so say it in the report.

### Mandatory vs non-fatal

- **Judge failure = FATAL.** No arbiter, no panel.
- **Debater failure** = loud failure, nonzero exit, but the next debater still runs.
  All debaters failing = FATAL before the judge.
- **Prep failure** = loud, nonzero exit, but non-fatal: debaters read the files
  themselves (more premium tokens). A pack over `PREP_MAX_BYTES` (80000) is
  truncated AND the seats are told it was truncated.
- **Summary failure** = loud, non-fatal: read the judge's revision plan directly.

## When to fire

**The round count lives in the trigger phrase.** Never make the user type a flag:

| What the user says | Rounds |
|---|---|
| `debate one` / `debate 1` / `debate this` / `debate` | 1 (default) |
| `debate two` / `debate 2` | 2 |
| `debate three` / `debate 3` | 3 |
| `debate four` / `debate 4` | 4 (hard cap) |

`debate three the billing formula` → `--rounds 3`. No number → 1.

Also fires on: "panel debate", "tear this apart with the panel", "panel this",
"multi model review", "cross-examine this", "review and debate", "full panel",
"audit and debate", or `/panel-debate <target> [--rounds N]`.

**Sequential means longer wall clock.** R0 + every debater + judge + summary run back
to back. Each rebuttal round adds one pass per debater plus one judge pass. Round 1 is
roughly 10-20 minutes at effort high; round 4 can pass an hour. Tell the user the
expected time when they ask for 3 or 4, then run it. Rounds 3-4 also mean 15+ premium
calls, so check your plan usage first.

**Distinct from a review-only panel:** this is ONE flow (review → debate → summary),
not a single-pass multi-model review.

## How to build the brief

Use an adversarial brief template:

```
ROLE: Adversarial reviewer. Your job is to find what is wrong, not to approve.

ARTIFACT: <the code / copy / plan, inline or as file paths + line ranges>

PURPOSE: <what it is supposed to do, who uses it, when it runs>

BLAST RADIUS: <what it touches (money? production data? customer data? ad spend?
payments?) and what a silent failure would cost>

AUTHOR'S OWN WORRIES: <what you are already unsure about>

DELIVER: Findings ranked by severity. For each: the defect in one sentence, a
concrete failure scenario, and the minimal fix. Mark each finding CONFIRMED or
PLAUSIBLE.
```

**Name the files explicitly in ARTIFACT.** The prep seat only reads what the brief
points it at (plus direct callers/callees/config). A vague ARTIFACT line = a thin
pack = premium seats doing the grunt reading anyway.

Write the filled brief to a file (e.g. `.tmp/panel-debate-brief-<slug>.md`) and pass
an ABSOLUTE path.

## How to call the script

```bash
bash "${PANEL_SCRIPT:-$HOME/.claude/skills/panel-debate/scripts/panel_debate.sh}" \
  <brief-file> --rounds N --workdir <repo-dir>
```

- `--rounds N` = REBUTTAL rounds after the judge's first cross-exam. Default 1.
  **Hard cap 4.** Stops early when nothing is contested.
- `--workdir` = the directory the seats may READ. **Point it at a clean subtree for
  anything secret-adjacent.** The read-only sandbox can still read `.env` files, so
  stage copies of the relevant files under `.tmp/panel-<slug>/` instead of exposing
  the repo root.
- **Env overrides:**
  - `PANEL_DEBATERS`: space-separated debater list; order = run order. Drop a seat
    when its login is capped (e.g. `PANEL_DEBATERS="astra sol"`). Say so in the report.
  - `PREP_ENGINE`: `sonnet` (default) | `luna` | `off`. Use `luna` when the
    Anthropic plan is the tight one.
  - `PREP_MAX_BYTES`, `PREP_EFFORT`, `PREP_SONNET_MODEL`, `PREP_LUNA_MODEL`
  - `OPUS_MODEL/EFFORT`, `ASTRA_MODEL/EFFORT`, `SOL_MODEL/EFFORT`,
    `JUDGE_MODEL/EFFORT`, `SUMMARY_MODEL/EFFORT`, and each `*_TIMEOUT_S`.
- **Cost:** subscription-based seats (Claude / ChatGPT logins) have no per-token
  billing but do draw on plan limits. API-key seats (GLM, Grok, Kimi via OpenRouter)
  bill separately.

## Reading the output

The script prints per-step progress, a seat status summary, an OVERALL line, the
transcript path (under `<workdir>/.tmp/debate-*`), and a paste-ready HANDOFF PROMPT.

Report back to the user:
1. **The verdict.** The summary seat's plain-English summary + analogy.
2. **Whether it converged.** "converged after R2 / round X" or "rounds exhausted".
3. **Any seat failure, loudly, with its reason.** SESSION-LIMIT on a Claude seat =
   plan cap (an abstention, not an approval). A codex seat failing = check
   `codex login` / ChatGPT plan. Prep failed = say the debaters ran without a pack.
   The exit code is load-bearing.
4. **The handoff prompt**, verbatim, if the user wants to execute the plan.

## The rule that matters most

**A failed or abstained seat is reported loudly and never silently dropped.** If
the exit code is nonzero, say so first.

## Gotchas

- **A user Stop hook can REPLACE a claude seat's deliverable.** Headless `claude -p`
  seats run your global Stop hooks. A hook that pattern-matches too loosely can block
  the stop and change the seat's final stdout to something short and off-topic. If a
  claude seat returns a short, unrelated reply, read it: it may be a hook talking,
  not the model.
- **API-key seats on prepaid credits (e.g. OpenRouter) can run dry mid-panel.** The
  seat then fails every later call with HTTP 402. Top up the key or drop that seat
  from `PANEL_DEBATERS`. GLM and Grok use separate billing surfaces too.
- **The context pack is a CLAIM, not evidence.** It was written by a cheap model at
  low effort. Debaters and the judge are told to verify against the file before
  confirming anything. If a verdict's only evidence is the pack, treat it as PLAUSIBLE.
- **Every codex call carries a large fixed overhead** (system prompt + project
  instructions + hooks, on the order of 20k tokens). Each extra rebuttal round pays it
  per codex seat. Another reason not to inflate rounds.
- **`timeout` may not be on PATH** (macOS ships without GNU coreutils). The script
  uses `with_timeout` from `panel_lib.sh`. Never hand-roll a seat with `timeout`.
- **Claude seats can die with SESSION-LIMIT.** The claude CLI prints "You've hit your
  session limit" to STDOUT (the deliverable). Detection is gated on rc!=0 AND a tiny
  output. Report it as an abstention with the reset time.
- **Byte floors are a proxy.** R1 floor 500 bytes; rebuttals floor 120 plus a required
  `HOLD|CONCEDE` match. A false "seat died" call is as damaging as a real one.
- **A brief that asserts a conclusion gets that conclusion back.** Grep for the code
  that produces a number before claiming it's wrong; state unknowns as unknowns IN
  the brief.
- **Seats have NO network and NO bash.** Do live fetches in the parent session and
  paste results into the brief.
- **`--workdir <dir>` nests output under `<dir>/.tmp/debate-*`.**
- **Briefs are passed by ABSOLUTE path.**
- **Byte-check the `.md`, never the `.log`, and never mid-step.**
- **Never add parallelism back.** Sequential is deliberate. If you must hand-run
  seats, run them one at a time. When backgrounding via a harness, run the command in
  the FOREGROUND of that call; never add your own `&`/`nohup`.
- **`--rounds 5`+ is rejected before any model runs.**
- **Never use `--permission-mode plan` for a headless seat.** The deliverable goes to
  a plan file. Use `--allowedTools "Read Grep Glob"` and pass the prompt via stdin.
- **Never infer a timeout from shell log text.** Only the watchdog's marker file
  counts.
- **On a cap night the rebuttal round and the summary can die while the judge's
  output is complete.** Use the judge adjudication, write the summary yourself,
  DISCLOSE the lost round. Check `status.txt` first.
- **Write log timestamps from `date`,** never the model's sense of time.
- **An expired Anthropic OAuth login kills EVERY claude seat at once.** Prep, the
  Claude debater and the judge all return a short "Failed to authenticate" message.
  The judge is mandatory, so the run ends FATAL with no verdict even if the other
  debaters finished. Fix: re-login with `claude` in a terminal. If it happens
  mid-task, read the surviving `r1-*.md` reviews, judge them by hand against the real
  code, and DISCLOSE that the Claude and judge seats did not run. Check `status.txt`
  first, and do not trust "Terminated: 15" lines in logs: they are the watchdog being
  cleaned up, not the cause.
- **A text-only API seat can return empty content** if a reasoning model spends its
  whole `max_tokens` on thinking. Raise the token cap or drop the seat.

## Related

- `scripts/panel_debate.sh`: the engine (read it before changing any invocation).
- `scripts/panel_lib.sh`: shared watchdog (`with_timeout`, `write_timeout_status`).
