# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Paletom is a French-language PWA for tracking scores and rankings ("classements") for Palet Vendéen, a regional game played 1v1, 1v1v1, 2v2, and 3v3. It's a single-page app with no build step, originally built with Claude (web) and iterated on via direct commits to `index.html`.

## Running and testing

No build system, package manager, or test suite — this is a static site.

Serve it locally with any static file server, e.g.:
```bash
python3 -m http.server 8787
```
Then open `http://localhost:8787/index.html`. There is no dev/prod config split — the same `index.html` runs locally and on GitHub Pages; Firebase is a live shared backend, so local testing writes to the same Firestore project as production (`paletom-f0fbf`) unless you swap the config.

Deployment is GitHub Pages serving directly from `main` (repo `applicatom/Paletom`, published at `applicatom.github.io/Paletom`). Pushing to `main` deploys — there is no CI, staging, or review step.

## Architecture

Everything lives in one file, [index.html](index.html) (~6800 lines): all CSS in a `<style>` block, all markup, and all JS in `<script>` blocks near the end. There's no bundler, no modules (aside from the Firebase `<script type="module">` import), no framework — plain DOM manipulation via `document.getElementById`/`innerHTML` and functions attached to `window`.

Some lines are extremely long (tens of thousands of chars) because icons/illustrations are embedded as base64 `data:` URIs directly in markup — don't try to read the whole file at once; grep or use offset/limit.

**Navigation** is a single-page-app pattern: all screens are `<div class="page">` elements toggled via `showPage(pageId)`, which also handles per-page setup (e.g. re-rendering rankings, showing/hiding bottom nav for "fullscreen" pages listed in `FULLSCREEN_PAGES`).

**Data model / backend**: Firestore (Firebase) under `ligues/{ligueId}/joueurs` and `ligues/{ligueId}/parties`, plus a top-level `ligues` collection and a `config/app` doc for version gating. `startLigueListeners(ligueId)` sets up `onSnapshot` listeners that keep `window._players` and `window._matches` in sync in real time and trigger re-renders (`renderPlayers`, `renderRanking`, `renderRecentMatches`, etc.) on every remote change — there's no local write-then-reconcile step, so UI updates flow from Firestore snapshots, not from optimistic local mutation.

A "ligue" (league) is the top-level tenant: players join a ligue via a share code, and all players/matches/rankings are scoped to the current ligue. The current ligue and a list of "my leagues" are kept in `localStorage` (`paletom_ligue_id`, `paletom_mes_ligues`, etc.), separate from the Firestore data itself.

**PEP rating system**: matches feed into a custom Elo-like rating ("PEP") computed per format (1v1, 1v1v1, 2v2, 3v3) via `calculerPEP_1v1`, `calculerPEP_1v1v1`, `calculerPEP_equipe`, using coefficients from `pepCoeffScore`/`pepCoeffFormat` and applied via `appliquerPEP`. The full spec for this system is in [Paletom_PEP_v1.pdf](Paletom_PEP_v1.pdf) — read it before touching any PEP calculation function, since the formulas encode rules (score-margin weighting, format weighting) that aren't self-evident from the code alone.

**Versioning / update flow**: `APP_VERSION` is a hardcoded constant; `verifierVersion()` compares it against a `minVersion` stored in Firestore (`config/app`) to force-block outdated clients, while `verifierNouvelleVersion()` separately polls the deployed `index.html`'s `Last-Modified` header to prompt users to reload when a newer build is live. Bump `APP_VERSION` when changing the Firestore data shape in a breaking way.

**Offline/installability**: [sw.js](sw.js) is a minimal service worker (cache name `paletom-vN` — bump `N` when changing cached assets so stale caches get evicted) caching `index.html`, `manifest.json`, and the logo. [manifest.json](manifest.json) declares the PWA metadata; note `start_url`/`scope` are hardcoded to `/Paletom/` (GitHub Pages project path), so local testing at a different path won't match the installed-app scope.
