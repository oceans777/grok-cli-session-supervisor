---
name: grok-cli-session-supervisor
description: >-
  Use when supervising other Grok CLI (Grok Build) sessions for stall,
  idle-after-done, session stop, goal drift, or loops—on a schedule or when
  asked to watch CLI health. Report issues only; do not kill or re-prompt other
  sessions by default.
---
# Grok CLI session supervisor

## Goal
Watch **other** Grok CLI sessions on this computer for **session stop**, stall / idle-after-done, goal drift, and loops. **Default action is report only**—never kill a process, never inject a new prompt into another session, unless the current caller explicitly authorized that stronger action.

This skill is the checklist. Cadence, work dirs, and who gets the alert belong in the routine or chat ask that invokes it—not here.

## What counts as a supervised session
- Live processes whose command line is clearly `grok` / Grok Build (exclude this supervisor run itself and unrelated tools).
- Matching session dirs under `~/.grok/sessions/` (or the current machine’s Grok session root), keyed by cwd / session id from the process.

## Checks (each run)
For every supervised session:

1. **Session stopped / turn ended (report promptly)**
   - Detect when the session’s last turn has **ended**: `turn_ended` / completed in session artifacts, **or** the expected result file named in the launch prompt already exists with a fresh mtime, **and** there is no newer tool/assistant activity after that.
   - Flag `session_stopped` on the **first** detection of that transition (process may still be alive as a zombie—that is fine).
   - Message must say clearly: this round’s task finished (or stopped), not “maybe stalled”. Include cwd, short session id, PID(s), result path if any, one-line summary from the last assistant turn or result file head, and ask the host whether to kill the idle process / send the next prompt.
   - Do **not** wait for a long quiet window before this flag. Quiet windows apply only to `stalled_or_done_but_alive` when `turn_ended` is missing.

2. **Alive vs advancing**
   - Note PID, cwd, resume id, model, elapsed wall time.
   - Read the session’s newest file mtimes (chat/events/updates/terminal logs).
   - **Stall / zombie**: process still up, but session artifacts have not changed for a long quiet window (default **≥ 10 minutes**), **or** the last turn already ended / result file already written and nothing new is expected → flag `stalled_or_done_but_alive` (use this when `session_stopped` was already alerted earlier for the same finish, or when finish markers are ambiguous).
   - **Still advancing**: recent artifact mtime inside the quiet window → healthy; do not alert for “still running”.

3. **Goal drift**
   - Compare the session’s original / latest user prompt intent (prompt-file or first user turn) with the **latest assistant actions** (last tool calls / last assistant text).
   - Flag `drift` only when work clearly left the stated goal (wrong repo, unrelated feature, rewriting an out-of-scope product) with a one-line evidence quote. Uncertainty → no drift flag.

4. **Loop / thrash**
   - Flag `loop` when the same failing command, same error, or same short assistant plan repeats many times with no new progress (default: **≥ 3** near-identical cycles in recent history), or the agent oscillates between two actions without advancing the goal.

5. **Self-exclude**
   - Never treat the current supervisor invocation as a patient.
   - Do not alert on the routine that only peeks progress for humans unless it is itself wedged.

## Quiet vs alert
- **No supervised sessions** → silent (message no one).
- **All healthy / advancing** → silent.
- **Any flag** → one short alert in the language the user uses with this assistant: session identity (cwd + short session id), flag type, one-line evidence, suggested next step for the **human or host assistant** (e.g. kill zombie, send next prompt, leave alone). Do not claim you already fixed it if you only reported.

Keep a small state file (path chosen by the routine) so the **same** flag fingerprint is not re-alerted every tick. Re-alert only when the fingerprint changes, clears then returns, or (for stall only) the quiet window doubles with the process still stuck. A new `session_stopped` for a newly finished turn must change the fingerprint and alert once.

## Forbidden by default
- `kill` / `kill -9` on other grok PIDs
- Writing into another session’s prompt or chatting into it
- Merging PRs, pushing code, or continuing the patient’s engineering work “to help”
- Spamming identical alerts

## After the run
Deliver only per the caller’s destination line (usually the user in the Grok Bot chat that owns the routine). If nothing to report: message no one.
