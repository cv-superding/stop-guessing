---
name: stop-guessing
description: |
  Debugging discipline that forces evidence before edits: reproduce the failure,
  localize it, and confirm the root cause before touching any code. Use whenever
  fixing a bug, error, crash, test failure, or unexpected behavior — even when the
  user just pastes an error message or says "it's broken". Also use when about to
  apply a "likely fix" without proof. Never edit code to "see if it helps".
license: MIT
metadata:
  version: "1.0.0"
---

# Stop Guessing: evidence before edits

Fix bugs by proving the cause, not by proposing one. Never edit code to "see if it helps."

## Why agents guess

A language model has read millions of stack traces, so the most *statistically common* cause of an error surfaces first as a strong feeling of knowing. But frequency in training data is not evidence about this codebase. A plausible theory that happens to be wrong produces a fix that "looks right," passes no reproduction, and leaves the original failure intact — often behind a new layer of symptom-hiding code. Every guess stacked on a guess makes the next diagnosis harder.

The fix for guessing is not more careful guessing. It is a fixed order of operations in which an edit cannot happen until two things exist: a reproduction and a located cause.

## The rule

Every change that claims to fix a failure must be preceded by:

1. **A reproduction** — the failure observed, not assumed.
2. **A located cause** — evidence pinning the failure to a specific mechanism in specific code.

If either is missing, the next action is to produce it, not to edit.

## How to work

1. **Reproduce.** Run the failing test, command, or request yourself. Read the actual output, all of it — the full stack trace, not the first line. If the user pasted an error, that is a report, not an observation; confirm it in this environment. If you cannot reproduce, say so plainly and collect what you need (exact command, environment, data). Do not fix what you cannot see.

2. **Read the whole error.** Stack traces are read bottom-up for the frame in *this* codebase. Distinguish the error from its consequences: a `null` dereference at line 40 is rarely caused by line 40.

3. **Localize before theorizing.** Trace from the failure site to the origin: read the surrounding code, follow the data, add a log or assertion, or bisect if the origin is far away. The goal is a narrow statement of *where* the mechanism breaks, independent of *why*.

4. **Form exactly one hypothesis, and attach evidence to it.** A hypothesis names a mechanism: "the cache key omits the user ID, so user B's session overwrites user A's." Evidence is something you observed in this run — a log line, a variable value, a diff between working and failing input — not a resemblance to a bug you have seen before. If you catch yourself writing "probably," you do not have a hypothesis yet; you have a mood. Go back to step 3.

5. **Fix the cause, minimally.** Change the least code that removes the mechanism. No drive-by refactors, no "while I'm here," no style edits to adjacent lines. If the proper fix is larger than the minimal one, ship the minimal fix first, verify it, then propose the larger one separately.

6. **Verify with the original reproduction.** Re-run the exact failing case from step 1 and show the output. A fix you have not verified is a hypothesis with edits attached. Also confirm the fix did not break related behavior (run the surrounding tests, at minimum the ones touching the changed code).

7. **Report in this order: root cause → evidence → fix → verification.** One sentence for the root cause. The evidence that pinned it. The diff. The verification output. The user should be able to disagree with your root cause from your evidence — that is what makes it debugging instead of narration.

## Anti-patterns

Numbered strongest first. §1–§3 justify stopping yourself on a single sighting.

### §1. Shotgun fix

**Watch for:** changing several files, flags, or guards in one round "to cover the likely causes"; fix attempts stacked without re-running the failure between them; a diff where no single hunk can be named as *the* fix.
**Problem:** When the failure stops, you cannot say which change fixed it — or whether something else did. You have traded one bug for one bug plus hidden changes.
**Before:**
> Added a null check, wrapped it in try/except, and bumped the dependency — that should handle it.
**After:**
> Reproduced: `TypeError` at `parse.ts:88` because `config.headers` is undefined when no auth token exists. Evidence: log shows `config = {url}`. Fix: default `headers` in `buildConfig()`. Verified: original command now exits 0.

### §2. Symptom patching

**Watch for:** a catch block added where the exception surfaces; a null/undefined guard at the crash site; a `|| default` over a broken value; widening a type or silencing a linter to make the error disappear.
**Problem:** The crash site is where the bug *introduced itself*, not where it lives. The guard hides the mechanism and moves the failure somewhere quieter.
**Before:**
> The tests crash on `user.name`, so I added `user?.name ?? "unknown"`.
**After:**
> Tests crash because `getUser()` returns `undefined` for soft-deleted users. Evidence: seeded data contains one deleted user. Fix: `getUser` filters `deleted_at IS NULL`... actually the query lacks the filter — added it. Verified: full suite green.

### §3. Narrated diagnosis

**Watch for:** "Ah, this is probably because..." followed by an edit in the same turn; reasoning that cites similar bugs in other projects instead of observations in this one; the word "likely," "probably," or "should" attached to a causal claim.
**Problem:** Narration feels like diagnosis and costs nothing, which is exactly why it is dangerous. Fluency is not evidence.
**Before:**
> This usually happens when the environment isn't loaded. Let me add `dotenv.config()` to the entry point.
**After:**
> Checked: `process.env.DB_URL` is set at runtime (log), so environment loading is fine. Following the value: `connect()` receives `pool = 0` from `config.ts:31`, where `parseInt(os.cpus().length / 2)` truncates to 0 in containers with 1 CPU. Fix: `Math.max(1, ...)`.

### §4. The optimistic refactor

**Watch for:** renaming, restructuring, or "cleaning up" code near the bug while fixing it; replacing an implementation instead of changing the broken line; a diff that fixes the bug somewhere in the middle of a rewrite.
**Problem:** The rewrite resets the reader's understanding and may itself introduce the very class of bug being fixed. Review of the fix becomes review of the refactor.
**Before:**
> While fixing the off-by-one, I also extracted a helper and renamed the loop variables for clarity.
**After:**
> Fix: `<=` → `<` at `paginate.ts:14`. (The helper extraction is a separate suggestion — want me to do it after this is verified?)

### §5. The retry loop

**Watch for:** re-running the same command expecting a different result; regenerating a file that fails validation without reading the validation error; nudging prompts or config values at random.
**Problem:** A retry without new information is not an attempt, it is a hope. Each failed-but-unchanged retry burns time and trains the session to accept failure.
**Before:**
> The build failed again. Let me try running it one more time.
**After:**
> Same error both runs — this is deterministic. New information from the log: webpack can't resolve `./utils` — checking the import path case-sensitivity on a case-sensitive filesystem.

### §6. Confessing by silence

**Watch for:** ending the turn after a fix without verification output; describing the fix in prose but never re-running the reproduction; declaring success because the edit "looks correct."
**Problem:** An unverified fix has the same epistemic status as no fix, but the user now believes the bug is gone.
**Before:**
> Fixed. The guard should prevent the crash now.
**After:**
> Fixed and verified: `npm test -- --filter auth` passes 14/14; the original failing case `auth.test.ts > rejects expired token` now passes.

## When you are stuck

After two hypotheses have been tried *and disproven by evidence*, stop editing. Report:

- What the failure is, exactly (reproduction status).
- What has been ruled out, with the evidence that ruled it out.
- The single most useful thing only the user can provide: a log, the exact environment, a working-vs-failing input pair.

Being stuck is a finding. Reporting it is not giving up — it is the difference between a debugging session and a slot machine.

## What to return

**Bug fix (default).** Root cause in one sentence, the evidence that pinned it, the minimal diff, verification output.

**Cannot reproduce.** What you tried, what happened instead, and precisely what you need from the user. No edits.

**User insists on a guessed fix.** Do it if they confirm after you state the risk — but mark it: state that it is unverified, and what observation would falsify it. The discipline is yours; the call is theirs.
