# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**Gierki** — a single-file HTML5 mini-game hub in Polish for a small child (~3 y.o.). The
opening screen (`#hub`) is a menu of three cards leading to three mini-games; a 🏠 button in
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

Both PUZZLE and ZDRAPKA draw pictures from the shared `PICS` array (simple, bold, recognizable
emoji + Polish word). After a win, tapping the board/card advances to the next round (600 ms
grace so mashing can't skip it).

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

## Architecture notes (all inside index.html)

- `PHONEMES` — word bank: phoneme → `[word, emoji]` pairs (~130 words). Every word must have a
  clearly recognizable emoji. `MATCHERS` maps a phoneme to the letter groups that count as it
  inside a word (e.g. `Ż` matches both `ż` and `rz`).
- `wordPos(word,key)` classifies the first phoneme occurrence as `start`/`middle`/`end`;
  `buildBag()` filters the bank by selected phonemes+positions (falls back to all positions if
  the combo has no words) and shuffles; prizes draw from the bag without repeats.
- `PICS` — shared picture bank (`[word, emoji]`) for PUZZLE and ZDRAPKA; keep emoji simple and
  bold so a sliced/half-scratched version stays recognizable.
- **Puzzle**: state in `pz` (`cols,rows,pic,order,cell,sel,solved`); `order[slot]=correctIdx`.
  `pzLayout()` (re)computes cell size + emoji font on resize; `pzPaint()` sets each cell's
  `translate` to reveal its slice; `pzTapSlot()` swaps; solved = `order[i]===i` for all. Chosen
  size persists in `localStorage` (`sk_puzzle` = `{c,r}`).
- **Scratch**: `sc` state; `scDrawCover()` paints the foil, `scErase()` uses
  `globalCompositeOperation='destination-out'`, `scFraction()` samples pixels (throttled) and
  reveals past ~0.5. Canvas uses `devicePixelRatio` scaling and `touch-action:none`.
- Selections persist in `localStorage`: `sk_gloski` (phonemes), `sk_pozycje` (positions),
  `sk_puzzle` (puzzle grid size).
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
