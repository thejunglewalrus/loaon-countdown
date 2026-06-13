# 2026 LOA ON SUMMER — English Countdown Page + `!loaon` Bot Command

**Date:** 2026-06-13
**Status:** Approved design

## Goal

Two deliverables:

1. An **English-language clone** of the official 2026 LOA ON SUMMER announcement
   page (countdown + ships background), hosted on GitHub Pages under
   `thejunglewalrus`.
2. A **`!loaon` chat command** on the `junglewalrus-twitch-bot` that replies with
   a live countdown to the showcase and a link to the page.

## Source

Official page: `https://lostark.game.onstove.com/Event/Promotion/LOAON/260603`
- Full-screen **looping video** background (`LOAON_MP4.mp4`, ~19.7 MB): ocean,
  two sailing galleons, clouds, "LOA ON" sky lettering.
- Overlaid HTML/CSS: title art "2026 / LOA ON / SUMMER"; red **LIVE** badge +
  `2026.6.20(토) 16:00`; 4-segment countdown (DAYS : HOURS : MINUTES : SECONDS);
  "방송 알람 신청하기" heading + Official YouTube (red) and KakaoTalk (yellow)
  buttons.
- Korean strings to translate: the date weekday `(토)` → `(Sat)`, the alert
  heading, and the two button labels.

## Event target time (single source of truth)

**2026-06-20 16:00 KST = `2026-06-20T07:00:00Z` (UTC) = 3:00 AM ET (EDT).**
Matches the `loaon-bingo` lock time. Both the page and the bot compute their
countdown from this UTC instant.

## Deliverable 1 — English countdown page

**Repo:** new public `thejunglewalrus/loaon-countdown`.
**URL:** `https://thejunglewalrus.github.io/loaon-countdown/`.
**Deploy:** GitHub Pages from the repo root on `main` (static; no build step).

**Layout (faithful to the original):**
- `index.html` — single static page.
- Background: self-hosted `assets/loaon.mp4` (downloaded from the official CDN),
  `<video autoplay loop muted playsinline>`, full-viewport `object-fit: cover`,
  with a dark fallback gradient behind it.
- Centered overlay column:
  - Title block: "2026" (small) · **LOA ON** (large) · **SUMMER** (spaced).
    Rendered as styled text (web-safe serif stack approximating the original);
    no copyrighted title image required.
  - Red **LIVE** pill + date line: `2026.6.20 (Sat) 16:00 KST`.
  - **Countdown**: four segments `DAYS : HOURS : MINUTES : SECONDS`, each a
    two-digit zero-padded number with a label underneath. Updates every second
    via `setInterval`. When the target passes, show a "🔴 LIVE NOW" state with
    the YouTube link emphasized.
  - Heading "Sign up for broadcast alerts" + two buttons:
    - **Official YouTube** (red) → `https://www.youtube.com/@LostArkGame`
      (the channel linked on the official page).
    - **Official LOA ON page** (gold) → the onstove promo URL (replaces the
      KR-only KakaoTalk button).
- Small footer note: "Unofficial English countdown · event by Smilegate /
  Lost Ark" to make clear this is a fan page, not official.

**Countdown logic (vanilla JS, no deps):**
```
const TARGET = Date.UTC(2026, 5, 20, 7, 0, 0); // months 0-indexed → June
function tick() {
  const diff = TARGET - Date.now();
  if (diff <= 0) { showLiveNow(); return; }
  const d = Math.floor(diff / 86400000);
  const h = Math.floor(diff / 3600000) % 24;
  const m = Math.floor(diff / 60000) % 60;
  const s = Math.floor(diff / 1000) % 60;
  // render zero-padded
}
```

**Responsive:** countdown wraps gracefully on narrow viewports; video stays
cover-cropped. No mobile/desktop split needed for a single hero page.

## Deliverable 2 — `!loaon` bot command

**File:** `src/index.js` in `twitch-chat-bot` (= repo `junglewalrus-twitch-bot`).
**Pattern:** a code handler mirroring the existing `!song` block
(`/^!loaon(\s|$)/i`), placed **inline alongside the other command handlers**
(next to `!song`/`!queue`), not appended as a stray block at the bottom of the
file. Per-user cooldown consistent with the others (e.g. `LOAON_COOLDOWN_MS`,
default ~10 s) to avoid spam.

**Countdown lives in the response, computed live per invocation.** Each time
`!loaon` runs, the handler computes the remaining time from
`LOAON_TARGET - Date.now()` and embeds the resulting `Xd Yh Zm` directly into
the chat reply string — there is no separate/standalone countdown routine. The
number is therefore always current at the moment the command is used.

**Shared target constant:** define `const LOAON_TARGET = Date.UTC(2026,5,20,7,0,0)`
near the top so it is the bot's single source of truth.

**Reply format (pre-event):**
```
🚢 2026 LOA ON SUMMER airs in 6d 13h 47m — Sat Jun 20, 3:00 AM ET / 16:00 KST.
Live countdown + ships 🌊 → https://thejunglewalrus.github.io/loaon-countdown/
```
- Compute `d`/`h`/`m` from `LOAON_TARGET - Date.now()`; omit `0d`/`0h` segments
  cleanly (e.g. "13h 47m", "47m").
- **Post-event state:** if the target has passed within the last few hours, reply
  "🔴 2026 LOA ON SUMMER is LIVE now! → <youtube link>"; once well past, reply
  that the showcase has aired. Keep this simple — a single time comparison.
- Reply via `sendChatMessage(..., { replyTo: tags.id })` like the other commands.

**Discoverability:** add `!loaon` to the `!commands` reference page if that page
is a static list the bot links to (line ~914). If editing it is non-trivial,
skip — out of scope.

## Out of scope (YAGNI)

- No timezone auto-detection on the page beyond showing KST + ET labels.
- No OBS overlay variant.
- No CMS/config; the target time is hard-coded in both places (one-off event).
- No reuse of the official title/button image assets (text/CSS recreation only,
  except the background video which is reused per decision).

## Deployment / verification

- Page: `git init` → push to new `thejunglewalrus/loaon-countdown` → enable
  Pages → load the URL, confirm the video plays and the countdown decrements.
- Bot: edit locally, restart locally to smoke-test `!loaon` output formatting
  (or unit-style check the formatter), then deploy to the OCI VM per the repo's
  documented `git pull` + `systemctl restart` flow. Use the `thejunglewalrus`
  git identity for all commits.

## Risks / notes

- Reusing Smilegate's video asset: acceptable for a small fan countdown with a
  clear attribution footer; can swap to a CSS recreation later if desired.
- The official YouTube channel handle should be verified at build time
  (`@LostArkGame`) against the link on the official page.
- Bot deploy touches a live production bot — test the handler in isolation
  before restarting the service.
