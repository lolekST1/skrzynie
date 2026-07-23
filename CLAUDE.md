# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**Gierki** — a single-file HTML5 mini-game hub in Polish for a small child (~3 y.o.). The
opening screen (`#hub`) is a menu of nine cards leading to nine mini-games; a 🏠 button in
each game returns to the hub. All games share one juice toolkit (Web Audio sounds, canvas
confetti/sparks, screen flash, `navigator.vibrate`, and `pl-PL` speech via the Web Speech API).

Mini-games:
- **SKRZYNIE** (`#chestGame`) — the original speech-therapy game. The player picks Polish
  phonemes (SZ, Ż, CZ, S, Z, C, R, L, K, G, P, B, T, D, F, W) and where the phoneme should sit
  in the word (POCZĄTEK / ŚRODEK / KONIEC), then taps a colorful treasure chest **3 times** to
  open it. Each chest rewards a word+emoji practising the chosen phoneme; the phoneme is
  highlighted in yellow and the phone speaks the word aloud.
- **PUZZLE** (`#puzzleGame`) — a tap-to-swap jigsaw of one big emoji sliced into a grid.
  Selectable piece count via chips: **2×2 / 3×2 / 3×3** (cols×rows). Tap two cells to swap;
  when every piece is home it celebrates and speaks the picture's word. A target thumbnail
  shows the goal. Pieces are rendered by clipping an oversized emoji (`.pzInner` translated
  inside an `overflow:hidden` `.pzCell`).
- **ZDRAPKA** (`#scratchGame`) — a scratch-off card. A silver canvas foil covers a big
  emoji+word; dragging erases it (`destination-out`). When ~50% is uncovered it auto-reveals,
  celebrates and speaks the word.
- **PARY** (`#memoryGame`) — a memory / matching game. Cards flip on tap (CSS 3D `rotateY`);
  find two identical emoji. Selectable board via chips **2×2 / 3×2 / 4×3** (`sk_memory`).
  A matched pair locks face-up (sparks + speech); all pairs → celebrate. A brief `mem.lock`
  holds input only during the mismatch flip-back (not a per-tap cooldown).
- **CO SŁYSZYSZ?** (`#listenGame`) — listen-and-point. The phone speaks a word; the child taps
  the matching emoji among **2 / 3 / 4** choices (`sk_listen`). A 🔊 button repeats the word;
  wrong taps shake + gently re-speak with no penalty; correct → celebrate + advance.
- **BĄBELKI** (`#bubbleGame`) — endless bubble-pop. Emoji bubbles rise (rAF loop, DOM nodes in
  `#bubField`); tapping pops them (confetti + tone + counter). The loop self-stops when
  `current !== 'bubble'`, so it pauses on 🏠 and restarts on re-entry.
- **CIENIE** (`#shadowGame`) — shadow matching. A big black silhouette (an emoji rendered with
  `filter: brightness(0)`) sits above **2 / 3 / 4** colorful emoji tiles (`sk_shadow`); the
  child taps the one whose shape matches. Correct → the silhouette regains its colours,
  celebrate + speech + advance; wrong taps shake with no penalty. Mirrors CO SŁYSZYSZ?'s
  solve/advance flow (600 ms grace).
- **PIANINO** (`#pianoGame`) — free-play animal piano. Eight rainbow keys (C-major octave, an
  animal emoji on each) play a `tone()` note on press; a pressed finger sliding across keys
  glissandos (`pointerover` while `pianoDown`). No win state; keys are built once (`pianoBuilt`).
- **MALOWANIE** (`#paintGame`) — finger painting. Drag on `#paintCanvas` to draw thick round
  strokes; a palette of color swatches picks the brush, 🌈 gives a hue-cycling rainbow brush,
  🧽 clears. Canvas uses `devicePixelRatio` scaling and `touch-action:none`; `paintSize()`
  refits on resize while preserving the drawing (snapshot → redraw). Chosen color persists
  (`sk_paint`). No win state.

PUZZLE, ZDRAPKA, PARY, CO SŁYSZYSZ?, BĄBELKI and CIENIE draw pictures from the shared `PICS`
array (~178 simple, bold, recognizable emoji + Polish word); `randomPics(n)` returns n distinct.
After a win, tapping the board/card advances to the next round (600 ms grace so mashing can't
skip it).

The entire app lives in `index.html` — inline CSS + one inline `<script>`. No build system,
no dependencies, no external requests (works offline once loaded). Screens are `.screen`
elements toggled by a `.on` class via `show(name)`; `goHome()` cancels speech + timers.

## Deployment (Vercel)

- **Live URL: https://skrzynie.vercel.app**
- Vercel project: `skrzynie` (team `lolekst1s-projects`, project id `prj_yCsPAmdIF3O5fChlEoSIjoC5IikY`), framework: none (static).
- If this GitHub repo is connected to the Vercel project (Vercel dashboard → project →
  Settings → Git), **pushing to `main` auto-deploys production** — that is the preferred flow.
- Fallback without Git integration: deploy `index.html` with the Vercel MCP tool
  (`deploy_to_vercel`, name `skrzynie`, target `production`) — same project, same URL.

## Running & testing locally

Open `index.html` via any static server (or `file://` — no CDN deps):

```bash
python3 -m http.server 8080   # http://localhost:8080
```

After every edit:

1. **Syntax check** the inline script:
   `node -e "const fs=require('fs'),vm=require('vm');new vm.Script(fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/)[1]);console.log('OK')"`
2. **Playwright smoke test** (Chromium is preinstalled at `/opt/pw-browsers/chromium` in
   Claude Code web sessions). Dispatch raw `PointerEvent('pointerdown')` — regular
   `page.click()` times out because animated elements (chest idle-bob) never settle.
   Key flows to check:
   - **Hub**: `#hub` is shown on load; each card opens its game; each 🏠 (`.homeBtn`) returns
     to the hub (there are multiple `.homeBtn`s — scope to the game screen).
   - **Skrzynie**: picker requires ≥1 phoneme, 3 taps open the chest, `#pWord` shows only words
     matching the selected phonemes+positions, rapid mashing (10 fast taps) opens exactly once
     and never freezes, tap after the 800 ms grace advances to the next chest.
   - **Puzzle**: size chips give 4 / 6 / 9 `.pzCell`s; tapping two cells swaps them; a solved
     board gets `#pzBoard.solved` and shows `#pzBravo`. (A test can read each cell's
     `translate(...)` to recover the current order and solve blind.)
   - **Zdrapka**: heavily dragging `pointermove` across `#scCanvas` reveals the picture
     (`#scBravo` shown, canvas faded out).
   - **Pary**: `.mzchip` gives 4 / 6 / 12 `.memCard`s; a test can read each `.memFace`'s emoji
     to pair slots blind; matched pairs get `.matched`, all matched → `#memBravo`.
   - **Co słyszysz?**: `.lzchip` gives 2 / 3 / 4 `.lisTile`s; the correct tile carries the same
     `data-w` the game will speak; wrong tile gets `.wrong` (no solve), correct → `#lisBravo`.
   - **Bąbelki**: after ~1.5 s `#bubField .bubble` nodes exist; a `pointerdown` on one pops it
     and bumps `#bubCount`; leaving via 🏠 stops the rAF loop (node count stays constant).
   - **Cienie**: `.hzchip` gives 2 / 3 / 4 `.shTile`s; the correct tile's emoji equals
     `#shTarget`'s (a test finds it by matching `textContent`); correct tap gets `.right` and
     shows `#shBravo`, wrong gets `.wrong` (no solve).
   - **Pianino**: `#pianoKeys .pKey` has 8 keys; a `pointerdown` on one adds `.down` and plays a
     tone. No win state.
   - **Malowanie**: `#paintTools .swatch` has 11 buttons (9 colors + 🌈 + 🧽); dragging
     `pointermove` across `#paintCanvas` paints (non-transparent pixels appear); the 🧽 tool
     clears the canvas back to empty.

## Architecture notes (all inside index.html)

- `PHONEMES` — word bank: phoneme → `[word, emoji]` pairs (~130 words). Every word must have a
  clearly recognizable emoji. `MATCHERS` maps a phoneme to the letter groups that count as it
  inside a word (e.g. `Ż` matches both `ż` and `rz`).
- `wordPos(word,key)` classifies the first phoneme occurrence as `start`/`middle`/`end`;
  `buildBag()` filters the bank by selected phonemes+positions (falls back to all positions if
  the combo has no words) and shuffles; prizes draw from the bag without repeats.
- `PICS` — shared picture bank (`[word, emoji]`, ~178 entries) for PUZZLE, ZDRAPKA, PARY,
  CO SŁYSZYSZ?, BĄBELKI and CIENIE; keep emoji simple and bold so a
  sliced/half-scratched/bubbled/silhouetted version stays recognizable. `randomPics(n)` returns
  n distinct `{w,e}`.
- **Puzzle**: state in `pz` (`cols,rows,pic,order,cell,sel,solved`); `order[slot]=correctIdx`.
  `pzLayout()` (re)computes cell size + emoji font on resize; `pzPaint()` sets each cell's
  `translate` to reveal its slice; `pzTapSlot()` swaps; solved = `order[i]===i` for all. Chosen
  size persists in `localStorage` (`sk_puzzle` = `{c,r}`).
- **Scratch**: `sc` state; `scDrawCover()` paints the foil, `scErase()` uses
  `globalCompositeOperation='destination-out'`, `scFraction()` samples pixels (throttled) and
  reveals past ~0.5. Canvas uses `devicePixelRatio` scaling and `touch-action:none`.
- **Pary**: `mem` state; `deck[slot]={id,p}`, two of each `id`. `mem.lock` guards only the
  compare window; matched slots get `.matched`, solved when all pairs matched. `memLayout()`
  recomputes on resize. Grid size persists (`sk_memory` = `{c,r}`).
- **Co słyszysz?**: `lis` state; `newListen()` renders `lis.count` tiles, one is `lis.target`.
  One `#lisArea` handler routes 🔊-repeat / advance-after-solve / tile-tap. Count persists
  (`sk_listen`). If TTS doesn't work (`speechWorks===false`), `lisApplyFallback()` reveals the
  target word (`#lisWord`) with a "Rodzicu, przeczytaj" hint so a parent can voice it.
- **Bąbelki**: `bubbles[]` of DOM nodes moved by a single `bubStep` rAF loop that returns
  (stops) when `current !== 'bubble'`; `openBubbleGame()` clears + restarts it. No win state.
- **Cienie**: `sh` state; `newShadow()` renders `sh.count` tiles + sets `#shTarget` to one
  pic's emoji shown as a black silhouette (`filter: brightness(0)` in CSS). Correct tap clears
  the filter to reveal the colours; mirrors CO SŁYSZYSZ?'s single-handler solve/advance. Count
  persists (`sk_shadow`).
- **Pianino**: `buildPiano()` (once) makes 8 `.pKey`s from `PIANO_NOTES`/`PIANO_COLORS`/
  `PIANO_EMOJI`; `pianoNote()` plays a `tone()` + sparks. `pointerover` while `pianoDown`
  glissandos across keys. No state persisted, no win.
- **Malowanie**: `paint` state; canvas drawn with round `lineCap`/`lineJoin` strokes. `paintSize()`
  fits the backing store to the CSS box at `devicePixelRatio`, snapshotting + redrawing so a
  resize doesn't wipe the picture. 🌈 sets `paint.rainbow` (hue cycles per stroke segment); 🧽
  (`paintWipe()`) clears. Selected color index persists (`sk_paint`).
- **Speech on iOS/iPad**: iOS ignores `speechSynthesis.speak()` unless first primed *inside* a
  user gesture — every prize speaks from a timer, so a document-level `pointerdown` capture
  listener calls `primeSpeech()` once. `say()` uses **default** rate/pitch and **no explicit
  `u.voice`** (both silence some iOS builds; `lang='pl-PL'` still selects Zosia); utterances are
  kept referenced (`retain()`, anti-GC) and `speakUnblocked()` **closes the `AudioContext`** for
  the word's duration (a running WebAudio context otherwise starves speech — the utterance never
  even fires `onstart`). `ctx()` recreates the context afterwards. Some iPads still never start
  speech (Safari limitation) — `speechWorks` (tri-state, persisted as `sk_tts`) detects this via
  `onstart`/timeout and drives the visible word fallback. The 🔔/🗣️ **sound self-test** in the
  hub footer lets a parent tell mute-vs-no-voice-vs-engine-dead apart. (Also: Control-Center
  **mute** silences audio; the Polish "Zosia" voice may need downloading in iOS Settings.)
- Selections persist in `localStorage`: `sk_gloski` (phonemes), `sk_pozycje` (positions),
  `sk_puzzle` (puzzle grid size), `sk_memory` (memory grid size), `sk_listen` (choice count),
  `sk_shadow` (Cienie choice count), `sk_paint` (Malowanie brush color index),
  `sk_tts` (`1`/`0` — whether speech synthesis works on this device).
- Input model: taps count on `pointerdown` with **no cooldown** (a 3-year-old mashes); only
  post-win screens have a grace period (chest 800 ms; puzzle/scratch 600 ms) so mashing can't
  skip the reward, plus a 350 ms `transitioning` guard during chest theme swap. Don't
  reintroduce per-tap cooldowns — that caused the game to "freeze" under fast tapping.
- Juice: squash/shake CSS animations, canvas confetti/sparks, rotating rays, screen flash,
  Web Audio sounds (no audio files), `navigator.vibrate`. Shared helpers (`say`, `tone`,
  `celebrate`, `spawnConfetti`, `flash`, `buzz`) live once and are used by all games.
- All UI text is Polish and lives directly in the markup/JS strings.

## History

Originally prototyped inside the `lolekST1/Wedding-MC-Poland-game` repo (PRs #22/#23 there
added then removed it) — it is intentionally a separate product and must stay out of that repo.
