# replay-capture — 参考录屏/真机捕获方法(原 remotion-ref-replay skill 正文)

> brand-video 的分析+捕获模块:参考视频理解(storyboard 纪律)、真机捕获管线、舞台相机权重算法、speed cut、交付与排错。

**Goal:** given a reference screen-recording, rebuild it as a *programmatic*
video so it tracks the current UI. Manual re-recording is the thing we are
deleting — the subject changes with every UI iteration, and re-shooting by hand
is the pain. Instead: drive the **real** app with a scripted session, capture
frames, and render them with Remotion + a camera. UI iterates → re-run capture
→ re-render. No manual recording, ever.

This is the brand-studio production line, not a one-off. Two product shapes
ride the same pipeline:
- **Tutorials** — step-accurate walkthroughs; correctness of every on-screen
  step matters, usually rendered at 1x.
- **Pitch/marketing cuts** — the same capture rendered as a speed cut (see
  Step 3) with tighter camera work; energy matters, individual keystrokes don't.

What the pipeline guarantees: **stability and normalization**. The output is
deterministic given (frames.json, replay spec, speed) — no shaky hand, no missed
click, no "the recording person's env leaked in" (given a clean profile). An
asset regenerates with two commands when the product changes.

This skill is the **method** and the **hard-won pitfalls**. It is engine- and
subject-agnostic: the first implementation replays a terminal UI (kobe TUI, via
tmux `capture-pane`), but the same pipeline applies to any app you can script
and snapshot (a web app via headless-browser screenshots, an Electron app, etc.)
— only the "capture" primitive changes.

Domain Remotion knowledge (animations, compositions, assets, fonts, ffmpeg)
lives in the **remotion-best-practices** skill — load it too. This skill is the
*replay-a-reference* layer on top of it.

**Worked example:** `packages/branding/` composition `quicklook-replay`
(`scripts/capture-tui.ts`, `src/quicklook/`). Read it for concrete code; read
*this* for the reasoning so a new adaptation doesn't re-derive it.

## The pipeline (three steps)

```bash
# in the Remotion package (e.g. packages/branding)
bun scripts/capture-<subject>.ts            # 1. capture -> src/<name>/frames.json
bun run studio                              # 2. preview the composition, tune the camera
bun x remotion render src/index.ts <id> out/<id>.mp4   # 3. render
```

Studio hot-reloads the camera/composition; `frames.json` only changes on re-capture.

## The replay spec contract

Every replay composition has one AI-editable spec beside the composition:
`src/<name>/<name>.replay.json` or `.yaml`. This spec is the executable version
of the storyboard. It owns:

| field | owns |
|---|---|
| `viewport` | terminal/browser grid and output dimensions |
| `capture` | fps, duration, output path, isolated home/profile, socket/session names |
| `setup` | seed data shown before the recording starts |
| `waits` | stable readiness markers and timeouts |
| `text` | prompts, commands, captions typed on camera |
| `flows` | named product flows such as `createTask` with field-walk timing |
| `beats` | timed actions: type/key/wait/flow/sleep |
| `regions` | named screen rectangles plus an auto-seeded, AI-editable coordinate hash |
| `stages` | camera timeline: from/to/region, including dynamic `capture-end`/`end` |
| `camera` | transition, fit, zoom clamp, row-gap, quantiles, tail hold |
| `delivery` | speed cuts, poster frame, web delivery knobs |

**AI edits the spec, not the engine.** `capture-*.ts`, the replay renderer, and
the camera code should be generic interpreters of this spec. Change TS only when
adding a new primitive/action/schema field or fixing a real engine bug. If a UI
iteration changes timing, wording, regions, stages, or speed cuts, update the
spec and re-run capture/render.

Minimum implementation shape:
- `replay-spec.ts` exports types, validation, dynamic region/time resolution,
  and duration helpers.
- `capture-*.ts` reads the spec, validates it, executes `beats`, and writes
  `frames.json`.
- The Remotion component reads the same spec to resolve `regions`, `stages`,
  and `camera`.
- `Root.tsx` derives composition size, duration, and speed-cut variants from
  the spec.
- Tests must cover at least dynamic `full`/`end` resolution and bad references
  such as unknown `region`, `waitFor`, or `textRef`; also cover region hash
  behavior (auto-generate when absent, preserve when AI edits it, reject
  non-string hashes).

Region hash rule: first generate `regions[*].hash` from `region name + capture
size + c0/c1/r0/r1`; then leave it in the spec. AI may edit the hash as a
self-eval marker ("I verified this still maps to composer after the UI change").
The renderer should not depend on the hash for framing; it is a cheap stale
coordinate signal and review affordance.

## Step 0 — content understanding (the single highest-leverage step)

Both the storyboard and the camera derive from what the reference actually
shows. The dominant failure mode is a *superficial* read that ships a video
doing the wrong thing — we paid for this more than once. In the worked example
most of round-one's wall-clock went into re-deriving stages and steps that a
proper first pass should have produced — budget the time HERE, not in rework.

```bash
# 1 frame/sec is the right granularity — 1/2s misses beats; denser wastes context
ffmpeg -y -i <ref>.mp4 -vf fps=1 <dir>/s%02d.png
```

`Read` the stills in order and write down the **storyboard table** — it is the
analysis source that becomes the replay spec. Capture beats, camera `stages`,
regions, and speed cuts must be projections of the same table. A complete row has:

| beat | ~when | user action (exact keys/inputs) | screen shows | subject pane/region | camera intent (wide/zoom) |
|---|---|---|---|---|---|

Grade your own pass: if a row's "user action" isn't concrete enough to become a
`beat`, or "subject region" isn't concrete enough to become a `region` rect, the
understanding pass isn't done. Keep the table/spec in the composition's
directory and update them when the reference or beats change — capture, camera,
and cut must never drift from separate hand edits.

Watch specifically for beats a shallow read collapses:
- A real **modal/page/dialog** vs. a shortcut that produces the same end-state.
  (In quicklook the "new task" beat is the real full-window NewTaskDialog, not
  an API call — the API call skips the on-screen page entirely.)
- **Pre-existing state** the video opens with (populated lists, prior items) —
  seed it, don't start from empty.
- **Input and waiting are animated** — typing appears char-by-char, spinners and
  loads actually run. That motion is the demo; don't paste instantly or jump-cut.

## Step 1 — the capture script (beat model)

Run the **real** app in an **isolated** context with **throwaway state** so it
never touches the user's real data. For a TUI: an isolated tmux server
(`-L <socket>`) + throwaway `HOME`/app-home dir, polling `capture-pane -ep`
(the `-e` keeps ANSI). Store a keyframe whenever the screen changes, stamped
with **wall-clock elapsed seconds** — NOT nominal frame indices; nominal time
drifts and typing/spinners replay at the wrong speed.

Interaction is a list of timed **beats** in the replay spec: `at + action`.
The capture script interprets generic primitives (`typeText`, `key`, `waitFor`,
named `flow`, `sleep`). Fire them fire-and-forget so a slow beat never stalls
the polling loop.

Reusable gotchas (all cost us time at least once):
- **`cd` into the Remotion package before running** — script + `frames.json`
  paths are relative; a backgrounded run from the wrong cwd fails silently with
  "Module not found".
- **Bun Shell promises are lazy** — a beat that isn't awaited never runs. Force
  it (`.catch(() => {})`) so it executes without blocking the loop.
- **Drive real dialogs/pages, not stand-ins.** Reproduce the actual keystrokes
  (open, walk fields, confirm) and give the surface ~1.5–2s on camera before
  acting. A hand-rolled stand-in is both wrong and re-broken every UI change.
- **Type char-by-char** (`send-keys -l <char>` + delay): ~45ms/char for prompts
  (readable), slower (~160ms) for short deliberate commands.
- **Chain submit onto the typing, never a separate timed beat** — per-char
  send-keys overhead makes real typing finish later than nominal, and a timed
  Enter fires mid-prompt and submits a truncated message (it did):
  `typeText(...).then(() => sleep(500)).then(() => key("Enter"))`.
- **Wait for readiness before typing into a just-booted surface** — boot time
  jitters run to run; typing at a fixed offset raced the boot and dropped
  leading chars. Poll `capture-pane` for a readiness marker first — and pick a
  **stable glyph** (e.g. the composer's prompt char), never placeholder wording:
  rotating placeholder examples burned two runs.
- **Warm up off camera** — boot the app once (not captured) so pre-seeded
  state finishes its expensive init (worktree, installs); the video must open
  on a settled screen, not an install spinner.
- **Teardown fully** — kill every socket/process the run spawned (outer + inner
  tmux, the daemon, engine sessions) or they leak between runs.
- **Side effects persist** — created tasks/branches/files in the target repo are
  real sandbox artifacts. Per repo rules, do NOT delete them without an explicit
  instruction; surface them instead.

## Step 2 — the stage camera (algorithm + weights)

Where the taste lives. These rules are the *result* of wrong turns — the commit
history of the worked example shows each fix.

**Model: storyboard stages, not per-frame tracking.** The demo is scripted, so
the camera is too. The spec's `stages` table of `{name, from, to, region?}`
gives **one fixed shot per stage**, eased between stages (`camera.transition`
≈ 1.2s, smoothstep).
Per-frame "follow the motion" tracking was tried and **REJECTED — it twitches**:
every keyframe spawns a new target and the camera chases between clusters. Do
not reintroduce it.

**Framing a stage — `frameStage(from, to, region)`:**
1. Accumulate a **binary** changed-cell mask over the stage: a cell that changes
   at least once counts **once**. Weighting by change *count* lets an
   ever-repainting spinner / status bar / composer outweigh the real subject —
   a bug we hit. Binary fixes it.
2. Only look **inside the stage's `region`** (a grid rect for the subject pane).
   Chrome and unrelated panes are excluded so their noise can't win the frame.
   No region = a forced **wide** shot (use for full repaints / boot / the final
   pull-back).
3. Cluster changed rows into **bands** (row-gap > 3 splits a band), frame the
   **heaviest band**, then take the **5–95% column quantiles** within it so a
   stray edge glyph can't stretch the box.
4. Scale = fit the band into **~80%** of the viewport, **clamped to [1, 1.6]** —
   past ~1.6 mono glyphs alias; below 1 is a pull-back (express it as a wide
   stage instead).

**Aiming — translate, not transform-origin.** Center the target point (px) via
`translate(...) scale(...)` with the translate **edge-clamped so the viewport
never leaves the content**. `transform-origin` percentages were tried and
**REJECTED — they crop edge-adjacent targets** (a top-row target lost its top).
A target near an edge sticks to that edge instead of cropping.

**Tuning knobs (priority order when a stage looks wrong):**
1. Wrong subject → fix the stage's `region` in the spec (most common fix).
2. Too tight / loose → `camera.fit` and `camera.minScale/maxScale`.
3. Subject cut in half, or noise merged in → `camera.rowGap`.
4. Edge glyph stretching the box → `camera.colQuantiles`.
5. Move feels abrupt → `camera.transitionSeconds` / the easing.

## Step 3 — speed cuts and delivery

A marketing/landing cut is usually the same capture at 2–4x. **Never speed up
the rendered file with ffmpeg `setpts`** — that compresses the camera easing
too (a 1.2s zoom becomes 0.3s and every move turns into a lurch). Instead the
composition takes a `speed` prop and decouples the two clocks:

- **Content clock**: `t_capture = (frame / fps) * speed` — frames, stage
  boundaries, everything scripted runs sped up.
- **Camera clock**: easing progress runs in OUTPUT seconds:
  `into = (t_capture - stage.from) / speed`, eased over the same ~1.2s a 1x
  render uses — zooms feel identical at any speed.
- **Clamp each transition to ~half the stage's on-screen duration**
  (`min(TRANSITION, stageOutputLen * 0.5)`) so short sped-up stages still
  *settle* on their subject instead of drifting through the whole stage.

Register cuts from `delivery.speedCuts` (`<id>`, `<id>-4x`, etc.); duration is
capture length / speed so studio previews every configured cut.

Delivery checklist (a raw Remotion render is not a web asset):
- Re-encode: `ffmpeg -i in.mp4 -c:v libx264 -crf 27 -preset slow -movflags
  +faststart -an out.mp4` — took the worked example 9.1MB → 2.4MB.
- Regenerate the poster frame from the NEW render (`-ss <t> -frames:v 1`) —
  a stale poster shows the old UI for exactly the flash that matters.
- Verify after deploy: `curl -sI <url>` content-length matches the local file.

## Effect boundaries (what NOT to do)

- **No generative video (Seedance/i2v/etc.) for the UI surface** — it
  hallucinates text and layout; a product demo must be pixel-true. Generative
  passes are fine only for ambient/brand shots or transitions, never the UI
  frames. (Explicitly evaluated and set aside.)
- **Don't hand-paint a fake UI.** A static mock still needs hand-editing every
  iteration — which defeats the whole point. Capture the real app.
- **Camera stays inside the content rect** — never reveal black beyond the frame.
  The translate clamp enforces this; keep it.
- **One shot per stage.** If you want to move within a stage, that's two stages —
  split the table; don't animate the target mid-stage.
- **Respect engine/product-owned identity** (repo rule) — the app already renders
  the real vendor name/model; never overlay hard-coded vendor chrome.

## Troubleshooting (symptom → cause → fix)

| Symptom | Cause | Fix |
|---|---|---|
| Camera jitters/twitches | per-frame motion tracking | fixed shot per stage (Step 2 model) |
| Zoom lands on a spinner / status bar | change-count weighting | binary change mask (count each cell once) |
| Wrong pane framed | region too wide / unset | narrow the stage's `region` to the subject pane |
| Top/edge of subject cropped | transform-origin % framing | translate + edge-clamp (never leaves content) |
| Text mushy at zoom | scale > ~1.6 | clamp scale to [1, 1.6] |
| Subject cut in half | band split too eager | raise the row-gap threshold (`> 3`) |
| Box stretched by a stray glyph | no column trimming | 5–95% column quantiles within the band |
| "Module not found" running capture | wrong cwd (relative paths) | `cd` into the Remotion package first |
| A scripted beat never happens | Bun Shell promise not forced | `.catch(() => {})` to execute it |
| Typing/spinners replay too fast/slow | nominal frame timestamps | stamp keyframes with wall-clock elapsed |
| Engine sessions leak after a run | partial teardown | kill both tmux sockets + stop the daemon |
| Demo shows a fake/oversimplified dialog | stand-in instead of real surface | drive the real dialog's keystrokes |
| Frames show env noise (nags, banners) | non-pristine capture profile | clean app-home / suppress interstitials first |
| Video opens on an install/loading screen | pre-seeded state initializes on camera | warm-up boot off camera before capturing |
| Submitted prompt is truncated | Enter scheduled as its own timed beat | chain Enter after typeText completes |
| Leading chars of a typed prompt missing | typing raced a jittery engine boot | waitFor a readiness marker before typing |
| waitFor never matches / fires late | matched rotating placeholder wording | match a stable glyph (composer prompt char) |
| Zoom lands on the pane's header/banner | one-off banner repaint inside the region | start the region below the banner block |
| Pane borders drift out of column | fallback glyphs (icons/braille) have a different advance | pin each span to its grid column (absolute left = col × cellW) |
| Dialog walk lands on the wrong tab/field | blind timed keys vs. a UI whose bindings changed | read the dialog's key-binding source; prefer position-independent chords (ctrl+e) + waitFor the dialog |
| Typing lands in a trust/confirm prompt | interstitial shares the composer's marker glyph | after waitFor, re-read the screen and dismiss the prompt first |
| Zooms lurch in a sped-up render | ffmpeg setpts sped up the camera too | `speed` prop: content clock × speed, camera easing in output time |
| Camera never settles in short stages | transition longer than the sped-up stage | clamp transition to ~0.5 × stage output duration |
| Web asset heavy / starts slowly | raw Remotion render shipped as-is | re-encode crf 27 + faststart; regenerate the poster |
| Black bars / frame not filled | fallback font's advance ≠ cell width | bundle the app's font (e.g. @remotion/google-fonts) |
| Camera frames chrome above the input while typing | input row outside the stage region | dedicated region for the composer rows (it may drift — widen) |
| UI change forces editing capture/render TS | beats, regions, stages, or camera knobs live in engine code | move them into `<name>.replay.json`; engine reads spec |
| AI "fixes the video" by hand-editing frames or TS constants | no declared editable surface | expose/validate replay spec and make it the only default edit target |

## Production checklist (before a capture replaces a shipped asset)

- Bundle the app's actual font in the composition — headless Chrome's fallback
  mono has a different cell width and can clip trailing chars.
- Use a **pristine** app-home / suppress environment noise (update nags,
  auth warnings, promo banners, third-party interstitials).
- Re-run analysis if the reference changed; keep the storyboard table in sync
  with the capture beats and the camera stages (all three share it).
- Validate the replay spec before capture/render; bad refs (`region`, `waitFor`,
  `textRef`) should fail before a long recording run starts.
