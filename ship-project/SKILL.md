---
name: ship-project
description: Ship a personal project Tarun-style — code in ~/Projects/<name>, LifeOS notes + symlink, public GitHub repo with a beautiful README (heart-balloon signature, "Built with Claude Code"), interactive HTML explainer, and GitHub Pages when there's something visual. Use when starting a new personal project OR when an existing local project is ready to publish ("push this to github", "make it public", "add readme").
---

# ship-project

The standard shipping ritual for Tarun's personal projects. Every step below was requested
individually across past sessions — do them all by default; ask only when a step genuinely
doesn't apply.

## 1. Placement — the LifeOS bridge pattern (non-negotiable layout)
This matches the existing Kiro `/buildproject` convention; keep both tools compatible:
- Actual code lives in `~/Projects/<PascalName>/` — NEVER inside `~/LifeOS`.
- LifeOS side `~/LifeOS/04_Projects/<slug>/` (lowercase-hyphen slug) gets:
  `README.md` (project brief — template at `~/LifeOS/09_Systems/Templates/Project.md`) and
  `project_status.md` (session log — template `ProjectStatus.md`).
- Symlink **from repo to LifeOS**: `ln -s ~/LifeOS/04_Projects/<slug> ~/Projects/<PascalName>/.lifeos`
- Repo gets a `CLAUDE.md` (Claude Code equivalent of the Kiro steering file): "read
  `.lifeos/README.md` + `.lifeos/project_status.md` for context before working; on finishing
  a session, append a 2–3 line session entry to `.lifeos/project_status.md` and check off
  completed Next_actions in `.lifeos/README.md`."
- If the project already exists in one place, bridge it — don't create a parallel copy.
- Android package names: `com.tatav.<name>` (ask if unclear).

## 2. GitHub (public by default)
- `gh repo create tatavarthitarun/<name> --public`, push `main`.
- If the project produces an APK or binary: attach it via a GitHub Release and link it in
  the README (don't commit large binaries to the tree).

## 3. README (this is the product page — make it beautiful)
Must include, in roughly this order:
- Hero: name, one-line pitch, badges if meaningful.
- Screenshots / demo GIF (take real ones — emulator or `adb exec-out screencap` for Android).
- What it does + how it works (short, diagrams welcome — mermaid ok).
- Install / build / run instructions someone else could follow.
- Link to the live HTML explainer (see step 4) — prefer the **rendered GitHub Pages URL**,
  not the raw file.
- Footer signature, always:
  `Built with heart. Rise with purpose.` ❤️🎈 and `Built with Claude Code on Mac` (Claude logo ok).

## 4. Interactive HTML explainer
Self-contained single HTML (inline CSS/JS, no CDNs): what was built, the steps taken,
architecture, screenshots, key learnings. Same visual quality bar as the README. Include the
heart-balloon signature. Put it in the repo (e.g. `docs/explainer.html`).

## 5. GitHub Pages
If there's anything visual (explainer, web app, demo), enable Pages
(`gh api -X POST repos/tatavarthitarun/<name>/pages -f 'source[branch]=main' -f 'source[path]=/docs'`
or via `docs/`), wait for the build, then **verify the live URL actually loads** before
reporting it. Put the live link at the top of the README.

## 6. Close the loop
- Add one row for the project in any LifeOS project index if present.
- Report: repo URL, Pages URL, what was skipped and why.

Keep public content professional — never leak private LifeOS details (finances, plans,
personal logs) into the repo, README, or explainer.
