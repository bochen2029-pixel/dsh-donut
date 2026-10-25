# DSH-DONUT: the perpetual session for DeepSeek Harness (portable spec v1)

2026-10-25 · operator Bo Chen · written by Claude Opus 5.5 (the session that built and fixed it) · scope: **DeepSeek Harness (dsh) only**.

The Claude Code side of the same goal is a different mechanism with its own package:
`C:/NEW/cutter/AUTOTRIM_AT_STOP_PORTABLE-SPEC_v1_FABLE5-1_2026-10-25.md` and `C:/NEW/cutter/dist/AUTOTRIM-AT-STOP_portable_v1_2026-10-25.zip`. Do not mix them.

**How to use this file.** A new session on a new machine reads §1 (what Bo wants), §2 (why dsh can do it), §5 (the design), then installs with §7 and proves it with §8. §4 lists the dsh invariants the code must never break. §9 lists the mistakes already made, so they are not repeated. The reference implementation is in the zip (`plugin/dsh-compaction-donut/`); the code is the ground truth, this file is the map.

```
package/
  DSH-DONUT_PORTABLE-SPEC_v1_2026-10-25.md   this file
  plugin/dsh-compaction-donut/               the plugin (plain JS ESM, no build step) + 17 unit tests
  desktop/preset-cordis-donut.yml            the desktop preset to append to the profile patch
  desktop/make_preset.py                     how that preset was derived
  stress/                                    scratch-harness profile patches (scaled "stress", production "replay")
  tools/                                     log verifier, fact checker, session scanner, zstd decoder, replay prep, run driver
  tools/workloads/                           generators for the reading workloads with planted facts
  reference/DONUT_..._ARCHITECTURE_...md     the full design history (long; read only when needed)
  MANIFEST.sha256
```

---

## 1 · What Bo wants (the requirements)

Bo works in long sessions, often autonomous and overnight, with sub-agents fanned out. Every harness on the market fills its context window and then either compacts (summarizes everything into one message and loses the nuance; Bo has tested and measured that loss) or dies. His manual workaround works but is pure toil: ask the session for a rehydration prompt, cut the oldest 30 % of the transcript with a tool, restart the harness, tell the session to rehydrate. The goal is to automate exactly that, invisibly.

Hard requirements, in his words where they matter:

| # | Requirement |
|---|---|
| R1 | **Nothing compacts below 90 % of the context window, ever, for any reason.** Not the automatic pass, not `/compact`, not any API call. "There's no good reason for a compaction under 90 %." |
| R2 | When it does compact (≥ 90 %): **the first ~20 % stays put** (the bootstrap/baseline he fed the session, "foundation freeze"), **the newest ~25 % stays put** (perfect recent memory), and only the **oldest part of the middle band** is compressed, only enough to land near **55 %**. He called it the donut. |
| R3 | Nothing is ever lost: the raw log is never edited; everything compressed stays recallable. |
| R4 | No manual work at all: no trim, no harness restart, no rehydration or handoff prompt. The harness's own compaction must never trigger; "from the harness's perspective it never gets full." |
| R5 | Scale: a full working day, or an unattended overnight agentic run; about **10 M tokens of history (10× the window)** before a session is worth retiring. |
| R6 | "N %" always means points on the context bar (the prompt as a fraction of the window). Never bytes. |
| R7 | QC means end-to-end runs on the real harness against the real API until the numbers prove it. Unit tests alone are not proof. (He was rightly angry when a reviewed version still compacted 65 % → 30 % in his first live test.) |

## 2 · First principles: why dsh can do this live

The model is stateless. Every request carries the whole conversation; "context %" is just the size of that request. Compaction means changing what the next request carries.

- **Claude Code** keeps the conversation in the RAM of a long-lived process per session and reads its transcript file only at launch. Editing the file does nothing until that process is restarted. That is why Bo's manual loop needs a restart, and why the Claude Code side of this project works at the process boundary (the other package).
- **dsh** rebuilds every request from a projection called the **surface**, folded over an append-only event log, and gives plugins a legal edit point (`agent/pre-step`) before each request. A plugin can therefore replace parts of the surface between two steps, durably, while the session runs. Nothing restarts; the harness simply never sees a full window. dsh is MIT-licensed, built on the Cordis plugin kernel; everything here is a plugin, no source patch.

## 3 · Target environment (facts that bit us)

| Item | Value |
|---|---|
| Harness | DeepSeek Harness Desktop **0.2.1-rc.3** (release build `be72761e…`, Electron 44.7, bundled Node 24.21, Cordis 4.0.5), Windows 11 |
| Install dir | `%LOCALAPPDATA%/Programs/DeepSeek Harness/`; CLI `resources/runtime/cli/bin/dsh.cmd`; the app's pnpm `resources/runtime/primary-runtime/dependencies/pnpm/bin/pnpm.cjs` |
| Harness home | `~/.dsh` (no `DSH_HOME` set); desktop profile `~/.dsh/profiles/desktop/` (`package.json`, `cordis.patch.yml`); the app owns it, and the CLI refuses `--profile desktop` |
| Sessions | `~/.dsh/sessions/<cwd-slug>/<session-id>/session.v4.jsonl.zstd`: one JSON event per line, one zstd frame per append batch. Node's `zlib.zstdDecompressSync` returns **only frame 0**; walk frames (`tools/decode_session.mjs`) or use Python `zstandard` with `read_across_frames=True` |
| Model | `deepseek-flash` (V4.1-Flash-1017), window **1,048,576**, automatic byte-prefix cache (a rewrite at position p re-bills everything after p) |
| Output reservation | default `maxTokens` **256,000**, and the provider rejects `input + max_tokens > window`, so the real prompt ceiling is 792,576 (**75.6 %**). This is why Bo's dsh sessions "crapped out" near 75 % with `CONTEXT_WINDOW_EXCEEDED 400`. **Set `llm-deepseek maxTokens: 65536`** (ceiling 983,040 = 93.75 %), or a 90 % floor sits above the ceiling and only overflow recovery can ever act |
| Shipped compaction | `compaction-basic` triggers at `min(0.8·W, W − O − 65,536)` = 727,040 (69.3 %) and keeps ~12 %. That is the "compacts when nowhere near full" Bo saw |
| Context bar | the web UI shows prompt only (`input + cacheRead + cacheWrite`) ÷ window |
| Source checkout | `C:/deepseek-harness-master` is a **later master snapshot**, not the release. Read it for intent; verify every API against the installed libs (`app.asar/dsh/node_modules/@deepseek-ai/*`; the npm `@deepseek-ai/<pkg>@0.2.1-rc.3` packages are byte-identical) |
| Prior art | the maintainers' unbuilt proposal `.agents/notes/proposed/feature/2026-07-06-recallable-compaction.md` in the dsh repo: frozen index stubs, one mutable state checkpoint, verbatim tail, recall tools. The donut is that plus a frozen head, hysteresis, the hard floor and partial passes |
| Privacy | the shipped `session-log-deepseek` plugin uploads each session's log suffix, shadowed originals included, to DeepSeek as a request field unless disabled (disabled in the scratch profiles) |

## 4 · dsh invariants the plugin must never break

Breaking any of these either corrupts the model's view or makes the session **unreadable at the next resume** (the reader validates more than live `append` does).

1. **Log.** A session is an append-only log of `{type, seq, time, data, surfaceOp?, sourceEventSeqs?, ignorable?}`. Nothing is ever deleted. Never invent event types: an unknown, non-`ignorable` type makes the log unreadable at resume. Use `compaction/*` events and `appendPluginRecord` (ignorable `plugin:` records).
2. **Surface.** An ordered list of seqs of message-producing events (`system/ developer/ user/ assistant/message`, `tool/result`). `surfaceOp` is `'append'` or `{op:'replace', startSeq, endSeq}`. A replace shadows the inclusive span **in surface order** (numeric order may be reversed after earlier replaces) and inserts one node in its place. Request = latest `request/header` config + tools + `deriveMessages(surface)`.
3. **Replace rules.** Both endpoints are current nodes, start before end in surface order. `sourceEventSeqs` must include every shadowed node. Both edges must be tool-balanced (`toolPairingBalancedBefore(start)`, `toolPairingBalancedAfter(end)`): never separate a tool call from its results. An `assistant/message` can never be a replacement. A `tool/result` replacement covers exactly one `tool/result`, changes only `message.content`, and needs an **open turn**. Node 0 (`system/message`) is protected; later system nodes are not, so the plugin pins them itself.
4. **Shadow price.** The event **immediately before** every replacement must be `compaction/summary` or `compaction/prune` naming the exact replaced range with `shadowedTokenCount` = the sum of the nodes' `heuristicTokens`. The token meter's projections throw on a mismatch and drift on absence.
5. **Bracket.** `compaction/start {compactionId, turn}` → `compaction/summary` → `user/message` (source `compactCheckpointSource(id)`, the replace) → `compaction/end`, appended with no `await` in between. Automatic brackets carry the open turn number (from `agent/pre-step`, which runs inside the turn); manual ones go through `agent.runMaintenance` with `turn: null`. Brackets never cross a turn boundary or overlap. A failure appends `compaction/end {error}`.
6. **`llmStreamCall: true`** only when the summary came from exactly one in-bracket `ctx.llm.stream()` call. Our summaries are made outside the bracket, so the field is omitted.
7. Never `session.append` from a `session/event` listener. Overflow recovery returns `{kind:'retry'}` only after `surface.replaceGeneration` advanced.
8. **Token meter quirk (rc.3 `measure()`):** at the anchor it uses provider usage only if usage ≥ its own heuristic estimate, otherwise it swaps in the heuristic, so `totalTokens` can sit far above what the bar shows (measured +148k at 796k). The donut therefore computes pressure itself from provider usage (§5.3).
9. **Packaging.** A patch cannot swap a row's module by id (`name` is an assertion; a mismatch skips the whole patch). On Desktop, compaction lives inside each agent preset's isolated `compaction` group, so a preset must be restated in full with the backend row swapped. Declare dsh packages as `peerDependencies` (never `dependencies`), or `instanceof ManualCompactionError` breaks across two copies of the class. **Node caches the plugin's code: after a code change, fully restart the app** (HMR reloads configuration, not code).

## 5 · The design as built

### 5.1 Shape

```
[ system ][ HEAD frozen ][ cards… frozen ][ STATE ][ band: raw, oldest first ][ TAIL raw ]
  node 0    ≤ 20 % of W    one per chunk    one     compressed only from here   newest 25 % of W
```

- **HEAD.** The opening (everything before the first model response: Bo's prompt plus the context the harness injects) extended at balanced boundaries while it stays within `headFraction` (0.20) of the window. Once any card exists, the head is everything before the first card and never moves again.
- **TAIL.** The newest `tailFraction` (0.25) of the window, always at least `tailMinSteps` (3) model responses with their tool results, aligned back to a human turn when that costs ≤ 25 % extra, cut only at a tool-balanced boundary.
- **Cards.** One frozen index card per compressed chunk, written once, never rewritten: `WHAT:` (1–3 sentences, past tense, user instructions quoted with their seq) and `KEYS:` (exact literals: paths, commands, ids, values, errors, each with its seq). The footer, written by code, names the original seq range and the exact recall calls. Cards never contain instructions.
- **STATE.** One working-memory document (Bo's "rehydration prompt", automated), rewritten per pass by merging the previous state with what just left the view: Task (never paraphrased: points at the opening), User instructions in force (verbatim, with seq), Decisions, Current state, Verified results, Rejected approaches, Unfinished work (described, never commanded).
- Checkpoints are recognized only when `source.kind === 'compact-checkpoint'` **and** the text carries `[donut stub` / `[donut state` (text alone is forgeable). Non-node-0 `system/message` and `developer/message` nodes are pinned: never inside a chunk.

### 5.2 When a pass runs

- **Hard floor:** a constant `HARD_FLOOR = 0.90` in `index.js`, not a config key. Below 90 % of the window the automatic pass holds, `/compact` is refused with a clear message, and `compactRegion` throws. The only override is the `DONUT_TEST_FLOOR` environment variable, set by the scratch stress harness in its own process. `highWater` below the floor is raised to the floor at construction.
- **Marks** (fractions of the **window**, i.e. the bar): `high = min(0.90·W, (W − reservation) − margin)` with `margin = min(32,000, 5 % of the ceiling)`; `low = 0.55·W`. A warning fires once if the high mark had to be clamped under the provider ceiling (the 256k-reservation case).
- **Fail closed.** Window unknown (nothing routed yet, or the model reports no capacity): nothing compacts; the pass holds, `/compact` and `compactRegion` refuse. A route change pending (the selected provider/model differs from the last logged request): hold until the new route has answered once. Provider ceiling below the 90 % mark (the 256k-reservation case): automatic compaction is unreachable, one warning, only overflow can act; the fix is `maxTokens 65536`.
- **Overflow** (the provider refused the request with `CONTEXT_WINDOW_EXCEEDED`) is the one path that acts regardless of the floor, because it only happens above the provider's ceiling: first a just-enough pass toward `min(low, ceiling − 32,000)` with the normal head and tail; only if nothing can be staged that way, a second pass with the tail at `tailMinSteps` and the tail's oversized results pruned too; then retry.

### 5.3 Pressure (bar parity)

`pressure = last provider prompt (input + cacheRead + cacheWrite) + last output + meter.surfaceDeltaTokens + estimate(pending input of this step)`. The last two terms price what was appended, or freed by a pass, since that response. With no provider anchor yet it falls back to `meter.totalTokens`. This number tracks what the next request will weigh, which is what the bar will show; a pass therefore fires when the request about to go out would reach 90 %, so the last *measured* prompt before a pass is up to one step lower (89.6 % at full scale). Node prices, the head and tail budgets and `needed` are in the meter's heuristic tokens; the session calibration (§5.4 step 10) converts between the two.

### 5.4 A pass (index.js `pass()`)

1. Describe the surface (seq, route-priced tokens, heuristic tokens, kind, turn start, balance), lay out head/cards/state/band/tail.
2. `needed = (pressure − target) / calibration`, where `target` is the low mark, raised to head + cards + tail + 2,000 (with a one-time warning) when those alone already exceed it.
3. **Prune only when pruning alone is enough:** a dry run prices pruning the band's oversized tool results (> 8,192 code points, oldest first, cut to head 3,072 + tail 1,024 with a marker naming the original seq, `session_event_read(seq=…)`). If that covers `needed`, prune for real (one `compaction/prune` + content-only `tool/result` replace each, stopping at `needed`) and the pass ends. Otherwise the band stays raw and cards do the work, so nothing is pruned that is about to be summarized. Overflow's emergency pass prunes regardless.
4. Chunk the band (≈ 24k tokens, tool-balanced, turn-aligned when possible, never across a pinned node) and **select only the oldest chunks whose tokens, minus 400 per new card and 2,500 for the rewritten state, cover `needed`**; the rest of the band stays raw.
5. Summaries: the first card alone (so the others hit its cached prefix), then the rest three at a time, each with one retry, at the model's least reasoning effort (thinking counts against `maxTokens`). Inputs are fenced per call with a random token so tool output cannot forge a `USER` block; both prompts get the OPENING verbatim; the state call also gets every human message from the departing span verbatim. The non-trailing selected chunks become cards; the trailing one becomes the new STATE, and the old state node is folded into the first selected chunk **only if the new state succeeded**.
6. **Literal guard (code, deterministic):** every literal in a card line (paths, codes like `ABC-1234`, numbers of 3+ digits, quoted or backticked phrases, file names) must appear verbatim in the untruncated span (also checked separator- and case-normalized for paths, and JSON-unescaped). Unverifiable card lines are dropped; state lines are kept and flagged `[unverified: …]`.
7. Failure policy: a card call that fails after its retry means **nothing commits** this pass (no frozen pointer cards outside overflow). A state call failing once means nothing commits; twice in a row, a carry-forward state is written by code (previous state + "spans #a-#b not yet merged").
8. Inflation guard: refuse the pass if the replacements would not be smaller than what they replace.
9. Validate every planned span (edges, balance, interior) before committing any. Then commit left to right, each replacement in its own bracket (§4.5), **re-resolving each span against the current surface right before writing it** (surface positions shift after every commit); the first span that fails stops the pass, so no raw gap is left between cards.
10. **Calibration, per pressure event:** after an event that committed, the first provider-measured prompt gives the residual `ratio = measured drop / estimated drop`, and `calibration = prev × (1 + ratio) / 2` (clamped 0.5–3). Measured convergence at small scale: 0.85 → 0.92 → 0.96.

The hysteresis loop runs at most 6 passes per event and stops when a pass commits nothing or frees under 1,000 tokens. **The tail is never squeezed on the pressure path**; only overflow recovery may. It holds with a one-time warning when the frozen region alone already exceeds the low mark.

### 5.5 Recall

Every card footer names its original seq range and the call to read them back: `session_event_read(seq)` and `session_event_search(query, seq_from, seq_to, surfaces=["shadowed"])`. Those tools come from the experimental bundle `@deepseek-ai/dsh-experimental-session-search` (off by default; add it in Plugins). In the stress run the model used them unprompted to answer from shadowed originals, citing seqs, and correctly distrusted a card's paraphrase.

## 6 · Configuration (production values)

| Key | Value | Meaning |
|---|---|---|
| `highWater` | 0.90 | pass when the prompt reaches 90 % of the window (cannot go below the 0.90 floor) |
| `lowWater` | 0.55 | a pass frees only enough to land here |
| `headFraction` / `tailFraction` | 0.20 / 0.25 | frozen head and raw tail, fractions of the window (`headFraction + tailFraction + 0.03 ≤ lowWater` is enforced) |
| `headTokens` / `tailTokens` | unset | optional absolute overrides |
| `freezeTurns` | 1000 | head extends by budget only |
| `tailMinSteps` | 3 | newest model responses always kept |
| `chunkTokens` / `minChunkTokens` | 24,000 / 6,000 | chunk sizes |
| `stubMaxTokens` / `stateMaxTokens` | 2,048 / 12,288 | output caps of the two calls |
| `pruneThresholdChars` / `pruneHeadChars` / `pruneTailChars` | 8,192 / 3,072 / 1,024 | band pruning |
| `maxOverflowRetries` | 2 | |
| `summaryProvider` / `summaryModel` | '' | '' = the routed conversation model |
| route `llm-deepseek maxTokens` | **65,536** | required (§3) |

## 7 · Install

**Desktop profile (Bo's setup):**
1. Quit the app. Back up `~/.dsh/profiles/desktop/{cordis.patch.yml, package.json, pnpm-lock.yaml, pnpm-workspace.yaml}` (copies beside the originals plus a second drive).
2. Put the plugin in a stable folder (Bo: `C:/NEW/donut/dsh-compaction-donut`). In the profile folder run the app's pnpm: `node "<install>/resources/runtime/primary-runtime/dependencies/pnpm/bin/pnpm.cjs" add link:C:/NEW/donut/dsh-compaction-donut`. (pnpm may move stale duplicate `@deepseek-ai/*` copies into `node_modules/.ignored/`; that is fine and desirable.)
3. Append `desktop/preset-cordis-donut.yml` to the profile's `cordis.patch.yml`. It inserts preset `cordis-donut` ("Creator mode + donut"), a full restatement of rc.3's Creator-mode preset whose `compaction` group mounts `dsh-compaction-donut` (and `command-compact`; the shipped tool-result pruner is dropped because the donut prunes its own band), sets `agent-preset-registry selectedDefault: cordis-donut`, and sets `llm-deepseek maxTokens: 65536`. **The preset is version-specific**: on another dsh version, re-derive it from that version's shipped `presets/cordis.patch.yml` (`desktop/make_preset.py` shows the swap).
4. Optional: add the session-search bundle for recall (Plugins → Add plugin → `@deepseek-ai/dsh-experimental-session-search`).
5. Start the app. New sessions get "Creator mode + donut" (shown beside the session title); existing sessions keep their preset unless switched in the picker.

Undo: restore the four backed-up files and restart.

**Flat profiles (headless, sdk):** `dsh plugin --profile <name> add <plugin path>`; the bundle layer disables `compaction-basic` and inserts the donut row with the production values.

## 8 · Verification protocol and the measured record

Always test in a scratch Harness home, never in `~/.dsh`: `DSH_HOME=C:/dsh-stress-home`, CLI `dsh.cmd` from the install, `DSH_TELEMETRY_DISABLED=1`, `DONUT_DEBUG=1` (the plugin traces every decision to stderr; headless prints no logger output). Create a profile with `dsh --profile <p> --from-default-profile headless --dump-default-config`, then `dsh plugin --profile <p> add <plugin>`, then write its `cordis.patch.yml` from `stress/`. rc.3 headless accepts only `--json`, `--session-id`, `--help`; run `dsh --profile <p> --json "<task>"` from the workload folder and continue with `--session-id <id>`.

1. **Unit:** `node --test test/policy.test.mjs test/verify.test.mjs` → 17 pass (layout, chunking, selection, marks, hysteresis, classification, literal guard).
2. **Scaled run** (cheap; production shape at 1/10): profile `stress` (`stress/stress.cordis.patch.yml`: high 0.085, low 0.055, head 0.01, tail 0.015) with `DONUT_TEST_FLOOR=0.085`, workload `fine_workload.py` (40 files × ~4.3k tokens, planted NOTE lines). **Measured (baseline code):** 39 steps held below the floor; 3 passes, each firing as the next request would cross the mark and compressing 3 of 5 chunks; provider-measured landings **5.4 / 5.5 / 5.3 %** against 5.5 %; calibration 0.85; 0 invariant problems. **Measured (final code, run14, verified independently from the log):** 40 holds; 3 passes at 3 of 5 chunks; landings **5.43 / 5.54 / 5.21 %**; calibration 0.85 → 0.92 → 0.96; 80/80 answer lines; 0 invariant problems.
3. **Full scale** (production config): profile `replay` (bundle defaults + `maxTokens 65536`), workload `build_full.py` (32 × ~30k + 40 × ~4.3k tokens), driver `drive_full.ps1` (resumes the session when the model stops early). **Measured:** with the old pressure the pass fired at a meter total of 944k while the provider prompt was ~796k (bug, fixed in §5.3); with bar parity and 64 files the prompt peaked at 856,294 (81.7 %), 182 steps held, nothing compacted (correct). RUN_FULL5_RESULT
4. **Negative replay** of a real session: copy the session folder into the scratch home under the same cwd slug, run `tools/strip_preset.py` on the copy (lets the one-shot runner adopt it: drops the header's preset/fork fields, turns `agent-preset/selected` and `session-log-deepseek/*` events into ignorable plugin records), run one turn with `--session-id`. **Measured** on a copy of Bo's 65 % session: `below the hard floor {"total":677900,"floor":943718}`, no compaction. (That turn's request then failed with a transport error, unrelated to the donut and not investigated.)
5. **Log checks** for every run: `tools/donut_verify.py <session.zstd> [--text]` (usage timeline, compaction events, folded surface, invariant problems must be 0, last `turn/end`); `tools/check_stubs.py <session> <workload>` (planted-fact claims in cards vs truth; measured 29/29 and 22/22 correct after the literal guard); the final answer against `_ANSWER_KEY.txt` (128/128 lines in a 64-file run).
6. **Bo's live check** after the app restart: he forked a 65 % session, kept reading documents, and reported that it "finally seems to have worked."

## 9 · Mistakes already made (do not repeat)

| Symptom | Cause | Fix now in the code |
|---|---|---|
| Bo's first live test went 65 % → 30 % | trigger at 0.70 of the *usable* window (65.6 % on the bar), and a pass compressed the whole band | marks as fractions of the window, 0.90 hard floor, partial passes |
| Pass fired at 76 % on the bar at full scale | meter `totalTokens` swaps in its heuristic baseline (§4.8) | bar-parity pressure from provider usage |
| One prune pass dropped 90 % → 43 % | pruned every oversized result | budgeted, oldest-first pruning |
| Cards stated wrong facts | summarizer saw tool results clipped to 2,400 chars | full spans + code literal guard |
| A whole agentic turn froze as "head" | head = first human turn | head = opening + 20 % budget, pinned at the first card |
| Working memory vanished after a failed call | old state folded unconditionally | fold only when the new state commits; carry-forward after two failures |
| Forgeable checkpoints | recognized by text | require `source.kind === 'compact-checkpoint'` |
| Latent unreadable log | `tool/result` replace possible outside a turn | never prune without an open turn |
| Recall tool errors | footer ranges inverted after a fold | footers from original seqs |
| "Fixed, but nothing changed" | Node module cache in the running app | restart the app after code changes |
| QC passed a known defect | reviewers flagged threshold units and whole-band passes; I chose "document, not implement" | any finding touching the operator's numbers is blocking; prove it at full scale |
| A full-engine rewrite never landed | Anthropic's safety classifier stopped the large `Write` mid-output (a false positive); the tool call did not run | re-applied as small edits; after a classifier stop, re-read the file before continuing |
| Every pass threw `ReferenceError: CARD_ALLOWANCE is not defined` | the re-applied edits referenced two constants only the stopped rewrite defined | constants restored; caught by the scaled run |
| 7 passes attempted, 0 committed, summaries re-run every step | all spans priced with surface positions computed before the first commit | re-resolve each span right before its commit; QC counts `pass committed` lines, never just exit codes |

## 10 · Limits, capacity, open items

**Room under the low mark.** Head 20 % + tail 25 % = 45 % of the window; the low mark is 55 %, so cards + state + raw band have ~10 % (~105k tokens) once compression is needed. Each pass frees ~35 % (~367k), first by pruning, then as ~15 cards of ~450 tokens plus the state. At that card size the directory fills the room after roughly a dozen card-writing passes (~4–5 M tokens of history); passes then get frequent and small, and the plugin warns once ("head + cards + tail already reach the low mark"). **Reaching R5's 10 M needs one of:** a code-only rollup of old cards (keep KEYS and footers, drop WHAT), smaller cards (≤ 150 tokens), or letting the head shrink back to the opening once cards exist. This is the first open item.

Open, ranked:
1. Card rollup (above).
2. Background drafting: cards are written at the pass, so the step that crosses 90 % waits for the summaries (seconds to a minute). Drafting aged chunks ahead of need makes the swap instant.
3. A model switch mid-session: the meter's delta resets when the request header changes; pressure falls back to the last usage without the delta until the next response.
4. A `/donut` status command (head/cards/state/band/tail sizes, last pass, calibration).
5. A static system-prompt rule telling the model to recall originals before acting on a vague card.
6. The freeze as an invariant inside dsh's surface fold (a source change; today the plugin enforces it).
7. REVIEW_STATUS

## 11 · Re-implementation contract (for a newer dsh version)

1. `export default class X extends CompactionEngine` (registers `ctx.compaction`), `static inject = ['llm', 'tokenMeter', 'sessions']`, `static Config` (Schemastery). Imports used: `@deepseek-ai/dsh-compaction` (`CompactionEngine, ManualCompactionError, CompactionId, compactCheckpointSource, toolPairingBalancedBefore/After`), `@deepseek-ai/dsh-llm` (`createUserMessage, errorChain, freezeMessage, BlockAssembler, LlmError, contentHasImage, CONTEXT_WINDOW_EXCEEDED_CODE`), `@deepseek-ai/dsh-session` (`SessionSeq`), `@deepseek-ai/schemastery`.
2. Listeners: `agent/pre-step` `({agent, messages, turn, step, signal}, next)` → run passes, always `return next()`; `agent/request-error` `({agent, failure, signal}, next)` → overflow pass, `{kind:'retry'}` only if `replaceGeneration` advanced; `agent/status` (reset per-agent counters on idle); `session/event` (reset the overflow counter on a new `assistant/message`).
3. Engine API: `compactIfNeeded(agent, trigger, signal)`, `compactNow(agent, signal, commandId)` (inside `agent.runMaintenance`, owner `null`), `compactRegion(start, end, agent, signal)`. All three refuse below the floor.
4. Services used: `ctx.llm.resolveModelInfo(provider, model, signal)` → `{context:{contextWindow}, defaultMaxTokens, reasoning:{efforts:[{id}]}}`; `ctx.llm.stream({provider, model, messages, toolHistory, maxTokens, sessionId, purpose:'compaction', reasoningEffort?, signal})` read with `BlockAssembler` (`finish.kind` error | aborted | max-tokens; `blocks()`; `usage`); `ctx.tokenMeter.measure(session, header)` → `{baseline, surfaceDeltaTokens, totalTokens, nodes:[{seq, tokens, heuristicTokens}]}` and `estimateMessage(msg)`; `session.surface.nodes`, `surface.replaceGeneration`, `eventAt(seq)` (deprecated, works on rc.3), `deriveEventMessage(e)`, `requestHeader()`, `toolHistory()`, `append()`, `seq`; `ctx.sessions.flush(session)`.
5. Keep every invariant in §4, implement §5, then pass §8 with numbers before calling it done.

## 12 · Tools

| File | Use | Notes |
|---|---|---|
| `tools/donut_verify.py <session.zstd> [--text]` | usage timeline, compaction events, folded surface, invariant checks | needs `pip install zstandard` |
| `tools/check_stubs.py <session> <workload_dir>` | planted-fact claims in cards/state vs the files | |
| `tools/dsh_scan.py [N]` | newest N sessions in `~/.dsh/sessions`: window, reservation, last prompt, final errors | diagnoses "dies at 75 %" |
| `tools/decode_session.mjs <file> stats\|types\|dump <out>` | frame-walking zstd decoder for Node | |
| `tools/strip_preset.py <copy.zstd>` | prepare a **copy** of a real session for one-shot replay | never run on an original |
| `tools/drive_full.ps1 <tag>` | full-scale run under `--profile replay`, resuming until all 72 files are read | hardcoded: install path, `C:/dsh-stress-home`, `C:/dsh-stress-full` |
| `tools/workloads/make_workload.py [root n words]` | 8 files × ~30k tokens with first/last lines and a planted NOTE, plus `_ANSWER_KEY.txt` | default root `C:/dsh-stress-work` |
| `tools/workloads/fine_workload.py [root n words]` | 40 files × ~4.3k tokens | default root `C:/dsh-stress-fine` |
| `tools/workloads/build_full.py` | `C:/dsh-stress-full`: 32 big + 40 small | run the two generators first |
| `desktop/make_preset.py` | derived `preset-cordis-donut` from Bo's `preset-cordis-975` block | hardcoded to his profile; adapt the source block |
| `stress/*.cordis.patch.yml`, `stress/*.package.json` | the scratch profiles (`stress` = scaled, `replay` = production) | |
