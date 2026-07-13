# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**Skrzynie Słówek** — a single-file HTML5 speech-therapy (logopedic) game for a small child,
in Polish. The player picks Polish phonemes (SZ, Ż, CZ, S, Z, C, R, L, K, G, P, B, T, D, F, W)
and where the phoneme should sit in the word (POCZĄTEK / ŚRODEK / KONIEC), then taps a colorful
treasure chest **3 times** to open it. Each chest rewards a word+emoji practising the chosen
phoneme; the phoneme is highlighted in yellow inside the word and the phone speaks the word
aloud (Web Speech API, `pl-PL`).

The entire app lives in `index.html` — inline CSS + one inline `<script>`. No build system,
no dependencies, no external requests (works offline once loaded).

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
   Claude Code web sessions). Dispatch raw `PointerEvent('pointerdown')` on `#chestBtn` —
   regular `page.click()` times out because the chest has an infinite idle-bob animation.
   Key flows to check: picker requires ≥1 phoneme, 3 taps open the chest, `#pWord` shows only
   words matching the selected phonemes+positions, rapid mashing (10 fast taps) opens exactly
   once and never freezes, tap after the 800 ms grace advances to the next chest.

## Architecture notes (all inside index.html)

- `PHONEMES` — word bank: phoneme → `[word, emoji]` pairs (~130 words). Every word must have a
  clearly recognizable emoji. `MATCHERS` maps a phoneme to the letter groups that count as it
  inside a word (e.g. `Ż` matches both `ż` and `rz`).
- `wordPos(word,key)` classifies the first phoneme occurrence as `start`/`middle`/`end`;
  `buildBag()` filters the bank by selected phonemes+positions (falls back to all positions if
  the combo has no words) and shuffles; prizes draw from the bag without repeats.
- Selections persist in `localStorage`: `sk_gloski` (phonemes), `sk_pozycje` (positions).
- Input model: taps count on `pointerdown` with **no cooldown** (a 3-year-old mashes the
  chest); only the open-prize screen has an 800 ms grace period so mashing can't skip the
  reward, plus a 350 ms `transitioning` guard during theme swap. Don't reintroduce per-tap
  cooldowns — that caused the game to "freeze" under fast tapping.
- Juice: squash/shake CSS animations, canvas confetti/sparks, rotating rays, screen flash,
  Web Audio sounds (no audio files), `navigator.vibrate`.
- All UI text is Polish and lives directly in the markup/JS strings.

## History

Originally prototyped inside the `lolekST1/Wedding-MC-Poland-game` repo (PRs #22/#23 there
added then removed it) — it is intentionally a separate product and must stay out of that repo.
