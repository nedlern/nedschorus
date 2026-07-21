---
status: draft
---

# fast-handoff — design and plan (for the boss + new-vp; not a choirmaster file)

The design of nedschorus's session-handoff system: what it is, how it works, every piece it needs, the decisions behind it, and the build and test order. Written to be readable without the design conversation that produced it. The skill file and script that ship into nedschorus are separate artifacts; this document is their specification.

**Review record:** d-reviewed 2026-07-20 (author soundness pass + one cold independent pass, verdict sound-with-named-risks); every HIGH and MED finding folded. Revised same day per the boss's content-model and counter-file rulings (§3, §4).

**Naming:** the skill's true name is `handoff`; that collides with the old system's skill until the project switch, so it is `fast-handoff` for now and renames to `handoff` when it enters nedschorus.

## 1. The problem and what success looks like

A session's context dies with the session. The old system's handoff — a summary written by the session's own agent — fails in a specific, observed way: the writer holds full context and writes for itself, so the summary compresses away exactly what a zero-context reader needed (observed 2026-07-20: a successor recovered a predecessor's key teaching only by reading the raw transcript, after the handoff had lost it). Meanwhile the boss must hand-type "read your handoff and continue" at every boot.

Throughout: **the quarry** = the old nedlern repo, kept as read-only reference; **choirmaster** = nedschorus's Claude-runtime agent.

**Success, observable per journey:**

| Journey | Success observable |
|---|---|
| Normal succession: session ends (boss present) → handoff written → next session boots | The successor completes the handoff's first next-step with zero context questions to the boss, and the boss types no boot instruction. |
| Confused successor: the handoff leaves a point unclear | The successor recovers the answer itself from the predecessor's transcript (the fallback path below), without the boss re-explaining. |
| Stale boot: a session starts when the latest handoff was already consumed | The staleness is detected mechanically and told to the boss at boot — never silently worked from. |

## 2. How it works, end to end

At session end, with the boss present:

1. **The transcript is extracted ONCE, then a naked subagent writes the handoff draft.** The session agent runs the transcript-extraction subcommand (§5, `transcript`) exactly once per session, saving the denoised dialog as `handoff/<NNNN>-transcript.md`; the naked writer is a fresh subagent whose only inputs are that file and the drafting instructions. No later step ever re-extracts: every subsequent reader uses the committed file. The naked writer is the audience it writes for — whatever it finds salient from zero context is what the successor will need at zero context. Its distillation of the dialog — including the boss's rulings in their conversational context — is the handoff's body; the raw transcript stays available behind it for full-context recovery on demand.
2. **The session agent corrects, adds, and prunes.** Corrects the draft's misunderstandings (they predict the successor's — the session agent is the only party still alive who can see them as misunderstandings; corrections become explicit "you will likely misread X as Y; actually Z" notes). Adds only what the transcript tail *misses* and no durable store already holds. Prunes what a durable store already holds, and routes suggestion-class material — proposed wiki/GHI/skill updates — to the boss-review-log instead of the handoff (the content rule of §3).
3. **The scrub, in the mode the boss picks at the ceremony ("handoff auto" / "handoff vet"; default auto — automation and autonomy are the default, boss ruling):** tasks scrubbed — close the dead, rewrite the stale, carry the live (transfer mechanism: §3, tasks row). Memories scrubbed — the memory store persists automatically and rots fast; this scrub is its recurring garbage collection. In **auto**, the session agent scrubs on its own judgment and the script transfers; in **vet**, the boss walks the scrub decisions before the transfer runs. Start very simple; juice it up later (boss ruling).
4. **The script finalizes** (§5, `write`): validates the finalized draft, injects the boilerplate If-already-read section itself, stamps `written by session <id> at <UTC>`, and writes it as the **next numbered file** in `handoff/` (the counter: `handoff/0001.md`, `handoff/0002.md`, …), then commits the handoff **and its `<NNNN>-transcript.md` together** (pathspec-scoped) and pushes. The committed transcript makes every session-provenance pointer resolve inside the repo (the JSONLs under `~/.claude` have no git backup), and the series doubles as the long-term-analysis dataset. A refused validation writes nothing — the prior handoff series is untouched. The numbered series is the archive, browsable without git archaeology — deliberately kept for long-term analysis of how handoffs evolve.

At the next session's start:

5. **The kernel line fires.** Choirmaster's CLAUDE.md carries one conditional line: run the read command (§5, `read`); **only on a fresh successful read, follow the handoff**; if it reports stale or already-read-by-you, stop and tell the boss (or, for own-stamp re-entry, resume what you were doing). No SessionStart hook, no launch script, no boss typing. Because CLAUDE.md reloads at every session start and after a context clear, the pickup instruction can never be lost.
6. **The script reads** (§5, `read`): finds the highest-numbered handoff, prints it, appends `read by session <id> at <UTC>` to that file, commits the stamp (pathspec-scoped). If the latest handoff was already read by a *different* session, it instead prints a staleness warning plus the file's own If-already-read section (§4) and exits nonzero for the boss's decision.

## 3. The pieces, and what a handoff contains

**The content rule (boss ruling, 2026-07-20): a handoff's body is context-dependent, not a fixed template — and content routes by audience.** Three destinations:

- **The successor** gets the handoff: the naked writer's distillation of the session dialog, plus exactly the things that (a) reading the transcript tail would miss AND (b) no durable store already holds. The correction notes are its highest-value class.
- **Everyone** gets the durable stores: tasks in the task list, memories in the memory store, permanent learning in the wiki, tracked work in GHIs, behavior in skills and code. Nothing with a durable home is duplicated into the handoff.
- **The boss** gets the boss-review-log (name his to change): *suggested* updates to the non-code SSOTs — a wiki page that looks stale, a GHI worth filing or closing, a skill worth changing — are proposals for his judgment, not successor context, so they go to the log, NOT the handoff and NOT the walk. He reviews the log when he feels like it, never per-handoff.

Only the frame is fixed:

| Artifact | What it is |
|---|---|
| `handoff/<NNNN>.md` | One numbered file per handoff, counter ascending, zero-padded for sort order. Fixed frame: the written-by stamp at top (doubling as the pointer to the predecessor's transcript), the read-by stamp appended on consumption, the script-injected If-already-read section at bottom. Body: per the content rule above — free-form, whatever this succession actually needs. |
| `scripts/handoff.py` | The one code piece. Subcommands `write`, `read`, `transcript` (§5). Chosen over prompt discipline by ruling: when it is a choice between python and prompt, go with python — code is testable and cannot forget. |
| The kernel lines | A few tokens in choirmaster's CLAUDE.md: run `read` at session start; follow the handoff only on a fresh successful read; otherwise stop and tell the boss. Plus the more-context pointer (boss ruling 2026-07-21): `handoff/<NNNN>-transcript.md` holds each past session's full dialog — Read its tail when the handoff leaves you needing more context. This replaces the old system's read-the-JSONL doctrine line: agents are trained not to know about JSONLs, and the committed transcript is in-repo, plain markdown, already denoised. Other meta-knowledge stays the wiki's job, not the kernel's. |
| `.claude/skills/fast-handoff/SKILL.md` | The choirmaster-side ceremony instructions (steps 1–4 above) plus the naked writer's drafting instructions. Renames to `handoff` at nedschorus entry. |
| Task carry (CONFIRMED, script-owned) | "Tasks live in the task list" — but the harness task store is keyed per session, so the successor's list starts empty. Per python-over-prompt: `write` exports the surviving (post-scrub) tasks to a small file the script owns; `read` imports them into the successor's store — the import runs inside the successor session, which knows its own session id, so no id needs passing. No hand-copying, no handoff-prose duplication. The scrub that precedes the export is vetted or not per the auto/vet mode (§2 step 3). |
| `boss-review-log.md` (name boss-ratified 2026-07-20) | Append-only file of suggested non-code SSOT updates, for the boss's at-leisure review. Entry format pinned: a dated checkbox line — `- [ ] <UTC date> <session-id>: <suggestion> → <target: wiki page / GHI / skill>` — so unreviewed items are visibly unchecked and Typora/Obsidian render them clickable. Fed at minimum by the handoff ceremony (step 2 routes suggestion-class material here); any mid-session moment may append too — it is just a file. **Non-blocking by rule: no work ever waits on a log entry** — anything that genuinely blocks goes to the boss in chat, where he is. Plain markdown at the manual rung; a script subcommand earns its way in only if format drift shows. |
| The quarry prototype | `handoff.py` and its tests live and run in the quarry first (§8), entering nedschorus through the entry checkpoint at populate. |

## 4. Handoff states, and what happens in each

The unit is the **latest** numbered file — earlier files are archive, never read at boot.

| State | Meaning | Exit, and who moves it |
|---|---|---|
| NO-SERIES | `handoff/` is empty or absent — a brand-new agent, before any succession exists. | Out of scope here (boss ruling): the future new-agent-start skill owns first boot. `read` still refuses cleanly ("no handoff series — ask the boss") rather than inventing behavior. |
| WRITTEN | Latest file has a written-by stamp, no read-by stamp. The normal baton. | The next session's `read` consumes it → READ. |
| READ | Latest file's read-by stamp matches the current session. | Normal work proceeds. Re-running `read` in the same session (e.g. after compaction re-fires the kernel line) prints exactly `already read by this session — resume` and nothing else. The state ends when this session's `write` creates the next numbered file → WRITTEN. |
| ALREADY-READ | Latest file's read-by stamp names a *different* session. Covers accidental double-boot, the ended-without-writing case (a session that booted, read, and died without handing off leaves exactly this state), and a mid-session context clear (a cleared terminal gets a fresh session id, and its state genuinely is behind — the warning is correct, not a false alarm). | `read` prints the warning + the file's If-already-read section, appends nothing, exits nonzero. The boss decides: continue anyway, or recover first. |

A file with no written-by stamp is unreachable by construction (`write`'s atomic finalize is the only producer); if `read` ever encounters one at the head of the series, that is corrupt state — it refuses and says so, never prints it as a handoff.

**The If-already-read section** — script-injected boilerplate, never hand-authored (a zero-context drafter would not know to include it, and hand-copying is the "agent remembers" failure the python-over-prompt ruling forbids): *"This handoff was already consumed by session `<reader-id>` — your picture is likely behind reality. Tell the boss. To recover what that session did: `python3 scripts/handoff.py transcript <reader-id>` and read the output. Do not start work before the boss says continue."* The script fills `<reader-id>` from the stamp when printing. A template change reaches only future handoffs; old files retain their embedded text (accepted — the archive is immutable).

Every non-terminal state has a reachable exit with a named actor; the ALREADY-READ exit is the boss, who is present at every boot by the launch model (he starts the sessions).

## 5. The script — `scripts/handoff.py`

| Subcommand | Does | Refuses / surfaces |
|---|---|---|
| `write` | Takes the finalized draft, validates it (§ below), injects the If-already-read boilerplate, stamps `written by session <id> at <UTC>` (session id from the harness env), writes it atomically as `handoff/<max+1>.md`, commits pathspec-scoped, pushes. | Validation failure → refuses, names the defect, writes nothing. Missing/empty session-id env → refuses loudly, distinct exit code, never stamps an empty id. Commit/push failure (no remote, unconfigured identity, conflict) → surfaced verbatim, the file already durable locally, never swallowed. |
| `read` | Finds the highest-numbered file; prints it; appends + commits (pathspec-scoped) the read-by stamp on first read; same-session re-read prints `already read by this session — resume` only; foreign read-by stamp → staleness warning + If-already-read section, appends nothing, exits nonzero. | Empty/absent `handoff/` → "no handoff series — ask the boss," distinct exit code. Missing written-by stamp on the head file → refuses as corrupt state. Missing/empty session-id env → refuses loudly. Stamp write failure → surfaced. |
| `transcript` | Extracts the readable two-voice dialog from a session's JSONL (args: session id, line count, default 1000): keeps boss and agent text; drops tool dumps, thinking fragments, scheduled-prompt turns, and subagent turns. Derives the transcript directory slug from the current repo path at runtime — never hardcoded from the prototype's home. Runs in exactly two situations: once at ceremony start (producing the committed `<NNNN>-transcript.md`), and on demand against a session that died without its ceremony (the ALREADY-READ recovery — the only case whose dialog is not already in the repo). | Missing JSONL → refuses with the exact path tried. Present-but-empty or unparseable JSONL → refuses and says which, never yields a silent empty dialog. |

**What validation is, honestly:** the fixed frame only — stamp block well-formed, If-already-read injected, body non-empty, counter uncollided. The body is free-form by the §3 content rule, so there is no section checklist to enforce; whether the residual is the *right* residual is the session agent's and the boss's judgment at the walk. The script checks frame, not meaning.

Every failure is structured and named — what happened, why refused, what to do next. No silent exceptions, no default-and-continue. (A v1 cut, recorded: a `--history` flag merging the boss's typed prompts from `~/.claude/history.jsonl` was considered and dropped — the session JSONL already carries the boss's voice, and the global history file is unversioned and format-unstable. Re-add only if a concrete gap shows.)

## 6. Decisions and rejected alternatives

| Decision (all boss-ruled 2026-07-20) | Rejected alternative and why |
|---|---|
| A naked subagent drafts; the session agent corrects, adds tail-missed facts, prunes store-duplicated ones. | Session-agent-writes (the old system): the writer holds full context and writes for itself — the observed compression failure this design exists to fix. |
| The handoff carries the distillation; the successor reads only the latest file. | Pointer-only handoff ("go read the transcript yourself"): slower and riskier at every boot, and it leaves no analyzable record — the committed distillation series is deliberately kept for long-term analysis. The transcript read survives as the on-demand fallback (§4). |
| Context-dependent body under the §3 content rule; no fixed section template. | The fixed State/Decisions/Next/Traps template (this design's own earlier draft): forces duplication of what durable stores already hold and pads every handoff with sections this succession may not need. |
| Numbered files in `handoff/`, counter ascending; the series is the archive. | Overwrite-one-file-with-git-as-archive (this design's own earlier draft): analysis requires git archaeology; the numbered series is browsable directly and costs one small file per session. Growth: retained indefinitely, revisited if size ever matters. |
| No two-phase vetted handoff (draft → d-review-style second pass). | The second phase re-reads with the same zero context as the first writer and mostly re-finds the same confusions; the annotation pass is the vet. The boss's bet, tested once at dogfood (§7 T7) — kept out permanently if the trial catches nothing material. |
| Pickup by CLAUDE.md line. | SessionStart hook (machinery to rig and maintain — the old system needed it only because five roles have five handoff paths); launch-script argument (the boss doesn't want a script in the launch path); boss typing (the current annoyance being deleted). The kernel line also self-restores after a context clear. |
| Stamps + validation + recovery enforced by `handoff.py`, not by agent discipline. | Prompt-side discipline ("remember to stamp"): agents at degraded moments demonstrably forget; python over prompt is the ruled general principle. Applied consistently: the If-already-read boilerplate is script-injected for the same reason. |
| Tasks and memories scrubbed at handoff, never copied verbatim; tasks live in the task list, carried by the proposed §3 script mechanism. | Verbatim carryover: observed 2026-07-20 to propagate dead and deferred items into the successor's working set. Handoff-prose task lists: duplicates a durable store, violating the content rule. |
| Suggested non-code SSOT updates go to the boss-review-log, reviewed at the boss's leisure — never walked at handoff. | Walking suggestions at every handoff: the boss declined the per-handoff review burden. Putting them in the handoff: wrong audience — they are for the boss's judgment, not the successor's context. Silently dropping them: the observed old-system failure (observations narrated once, then lost). |
| The ceremony runs with the boss present, and he picks the scrub mode per handoff: "handoff auto" (default — agent scrubs autonomously, script transfers) or "handoff vet" (he walks the scrub first). Default to automation and autonomy. | Mandatory walk every handoff: the boss declined the per-handoff review burden. Boss-absent unattended handoffs: still out of scope — the launch model (he starts every session) makes them rare; reopens if that changes. |
| The denoised transcript is committed beside each handoff; JSONL extraction happens once per session (boss refinement 2026-07-21). | Extract-on-every-read (the earlier shape): re-derives the same content each time, and leaves every session pointer dangling on a `~/.claude` file with no git backup. |
| Read stamps commit locally and do not push. | Pushing every read stamp: a boot-time push failure would block every boot for bookkeeping. The deliberate consequence: staleness detection holds within one clone only — fine while one machine and one writer exist; the Codex-companion admission must revisit (§9). |
| First-ever boot (no handoff series) is out of scope — owned by a future new-agent-start skill. | Designing first-boot behavior here: a different problem (there is no predecessor to distill) bolted onto this one. |

## 7. Assumptions and test plan

**Assumptions, labeled:**

- MEASURED (2026-07-20, this role's session): JSONL tail extraction of a *closed predecessor session* yields readable dialog; noise strip is mechanical; the harness exposes the session id in the environment; a fresh agent reading a predecessor's tail recovered teaching the written handoff had lost; a mid-terminal context clear mints a NEW session id (observed live — which is what makes the ALREADY-READ semantics on a cleared session correct rather than a false alarm).
- MEASURED (2026-07-21, three context-visibility probes, zero tool calls each, verified by verbatim-quote checks): a **fresh subagent** sees the project CLAUDE.md, the memory index, the git status block (whose recent-commit subjects can leak project terms), and the spawning prompt — and NO parent conversation. A **fork** (`subagent_type: fork`) inherits the full parent conversation (and costs it: ~371K tokens for a three-question probe). A **headless session in an empty directory** sees no project CLAUDE.md but DOES see the user-global `~/.claude/CLAUDE.md`, which follows every session on the machine. Two consequences: the fresh subagent's reader-state is exactly choirmaster's boot state (kernel + prompt + ambient surfaces), so the naked writer and every boot-test run against the true reader; and in the T6a quarry dogfood the naked writer carries the QUARRY's large CLAUDE.md rather than nedschorus's small kernel — a stated fidelity caveat, acceptable because T6a tests draft quality, not kernel fit.
- INFERRED (harness-documented behavior): CLAUDE.md reloads at session start and survives context clears — the kernel line's reliability rests on this; a spawned subagent starts with no parent conversation context — the writer's nakedness rests on this; the session id is stable across *compaction* (the process and JSONL continue; only a clear mints a new id) — a false ALREADY-READ after routine compaction would cry wolf, so T6a verifies this live before anything ships.
- UNMEASURED, each with its gate: the naked draft's quality from tail-only input (gate: T6a); `transcript` against the session's *own still-open* JSONL — the ceremony reads a live file, which is not the closed-file case measured above (gate: probed during build step 1); whether the statusline's context-fill feed can be read reliably at PostToolUse time for the 50%-trigger idea (gate: probed only if that automation rung is attempted — §9).

**Test cases** (companion tests ride with the script in the quarry; red-first where marked):

| Id | Case | Red condition proven by |
|---|---|---|
| T1 | `write` refuses a draft failing frame validation (malformed stamp block, empty body, counter collision), naming the defect — and the existing series is byte-identical before and after the refusal | Fixtures per defect; diff the directory |
| T2 | `read` stamps exactly once; same-session re-read prints only the resume line, adds nothing | Run twice under one session id |
| T3 | Foreign read-by stamp on the latest file → warning + If-already-read section printed, nonzero exit, no new stamp | Fixture with a foreign session id. Strip-the-gate: neuter the stamp comparison → T3 must go red |
| T4 | Empty/absent `handoff/` → distinct "no handoff series" refusal; stampless head file → corrupt-state refusal, not printed as a handoff | Run against an empty directory, then an unstamped fixture |
| T5 | `transcript` output contains both voices and zero tool-dump / scheduled-prompt / subagent lines; runs against a closed session AND the live current session; empty or unparseable JSONL → named refusal | Real quarry JSONLs; seed a scheduled-prompt line → must be dropped |
| T8 | Missing or empty session-id env → `write` and `read` refuse loudly, distinct exit code, nothing stamped | Unset the env var in the test harness |
| T9 | Commit/push failure (no remote; unconfigured git identity) → surfaced verbatim; after a refused or failed `write` the series head is unchanged | Fixture repo with no remote / no identity |
| T10 | Counter mechanics: `write` creates max+1 zero-padded; `read` targets the highest number, never an earlier file | Seed a multi-file series; verify targets |
| T6a | **Dogfood, quarry half (system layer):** the ceremony — naked draft from the live session, corrections counted, boss walk, `write` — runs as this role's own next real handoff, targeting the old system's handoff path and using the quarry's sanctioned push path (the raw-push guard would otherwise intercept `write`). Verifies session-id stability across any compaction that occurs. | Live run, boss present. Gate: the successor completes the handoff's first next-step with zero context questions to the boss. Correction count recorded as evidence for the naked-writer bet, not as a threshold |
| T6b | **Dogfood, nedschorus half:** kernel-line pickup — first real nedschorus boot runs `read` unprompted and continues. Not testable in the quarry: its SessionStart hook and existing onboarding already load context at boot, so "boss types nothing" discriminates nothing there. | First choirmaster boot. Gate for declaring the system live, not for entry (entry gate = T1–T5, T8–T10, T6a) |
| T7 | **Second-pass trial (one-time):** a d-review-style second pass on the T6a handoff | If it catches nothing material, two-phase is out permanently, with this trial as the recorded reason |

## 8. Build and test order

1. **`handoff.py` prototyped in the quarry** — `transcript` first (already half-built 2026-07-20: the denoise extraction ran live), including the live-own-session probe; then `write`/`read` with the counter; tests T1–T5, T8–T10 green. Cheap probe before anything depends on it.
2. **This plan d-reviewed** — done 2026-07-20 (author + cold pass); findings folded. Content-model and counter revisions (this version) ride the next review pass only if the boss wants one — they change the artifact's shape, not the mechanism's risks.
3. **The skill file drafted** as writing-for-a-zero-context-reader — the ceremony steps plus the naked writer's drafting instructions — then a d-review clarity pass (the near-zero-context reader is exactly who clarity mode simulates).
4. **Cross-runtime review of the near-final plan** — the design + the founding boot-up plan go to the Codex side (CVP-grade consult: `gpt-5.6-sol` at xhigh reasoning, repo access, analysis-only, adversarial prompt per inter-llm doctrine) — the d-review so far is one runtime's eyes; the second runtime demonstrably catches what the first misses. Findings folded before dogfood.
5. **Dogfood T6a + T7** on this role's next real handoff, boss present.
6. **Enter nedschorus at populate** (boot-up plan step 6), through the entry checkpoint: script + tests, skill (renamed `handoff`), kernel line, and the founding handoff created via `write` by the new-agent-start path. Git identity is part of the populate substrate, so the script's commits work from the first boot. **T6b at first boot** declares the system live.

## 9. Known holes — stated, not discovered later

- **Naked-draft quality is unmeasured** until T6a; the whole design bets on it. Owner: new-vp. Closing trigger: T6a's correction count and the successor's boot experience.
- **The correction pass and the walk are still degraded-moment work.** Drafting moved to a fresh subagent, but corrections, the task scrub, and the memory scrub are done by the ending agent at low context — and memory is the one store that grows without bound if the scrub is skipped (the quarry's memory index needed repeated manual prunes). Mitigation, stated plainly: the boss is present at the walk and is the backstop; the correction count in T6a doubles as evidence on correction quality. Owner: new-vp. Reopens as a mechanism candidate if dogfood shows scrubs being skipped.
- **The 50%-context auto-trigger is an automation-ladder candidate, not v1.** The boss's ideal: the handoff fires automatically at ~50% context fill (his guess at the right moment), plausibly via a PostToolUse hook. Deliberately not built yet, on the ladder's own terms: v1 is boss-triggered (he watches the context meter and calls the ceremony — the manual rung); a script-you-run reminder or hook earns its way in later. The data source exists: the harness delivers context-fill % to the statusline regularly, and that feed's consuming code is visible (boss observation 2026-07-20) — so the trigger needs wiring, not invention; what remains unmeasured is only whether a PostToolUse-time read of that feed is reliable. And 50% is a guess — T6a's dogfood shows at what fill the ceremony still produces good corrections. Owner: the boss (trigger policy), new-vp (mechanism if attempted).
- **Tail-window luck:** material from early in a very long session can be beyond the tail and absent from every durable store. Accepted residual — the boss is present at the walk and the norm is committing as you go. Bounding detection: a confused successor's fallback read failing to find the answer.
- **Staleness detection is single-clone.** Read stamps are local commits (deliberate, §6); a second clone or second writer would not see them. Reopens at Codex-companion admission, which must pick a mechanism (push the stamps, or move detection server-side) as part of mini-comms design.
- **The boss-review-log accumulates at the boss's pace.** Its reader reviews at leisure by design, so entries can sit — acceptable for suggestions (nothing blocks on them, by rule), but an ever-growing unchecked backlog would quietly become the old system's stale-queue failure. Bounding: the unchecked-checkbox count is visible at a glance; if it grows past what the boss actually drains, that is the signal to prune or change the mechanism. Owner: the boss (drain), new-vp (flag when it looks unhealthy).
- **The old system's handoff machinery is not touched by this design.** This role keeps using the old skill until T6a replaces it for this role's own handoffs; nothing else in the old fleet changes.

## 10. Open items for the boss

- The checkbox entry format for `boss-review-log.md` (name ratified; format still mine to confirm with you).
- ~~The system's name~~ — ruled: `fast-handoff` now, renames to `handoff` at nedschorus entry.
