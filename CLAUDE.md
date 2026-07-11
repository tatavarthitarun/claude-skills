# claude-skills — repo conventions (read before adding/editing a skill)

This is a **collection** of Claude Code skills, published at
https://github.com/tatavarthitarun/claude-skills (public). It syncs across Tarun's Mac and
Windows office laptop via git.

## Structure (keep this shape)
```
claude-skills/
├── README.md                 ← SLIM root index ONLY: intro + skills table + install + signature.
│                               Add one table row per skill; link it to the skill's folder.
│                               Do NOT put skill-specific content (banners, how-it-works) here.
├── <skill-name>/
│   ├── SKILL.md              ← the skill itself (YAML frontmatter: name, description + body).
│   │                           This is what Claude Code executes. Required.
│   ├── README.md             ← this skill's human-facing doc (banner, how-it-works, proof). Optional but preferred.
│   └── assets/               ← this skill's images (banner.png, screenshots). Per-skill, not shared.
```

## Public/private split
This working copy also holds **local-only personal skills** that must never be published.
They are listed in `.git/info/exclude` (untracked by design). Before committing a new skill,
decide which side it belongs to: portfolio-grade/generic → commit + README row + push;
personal-workflow → add its folder to `.git/info/exclude` and do NOT list it in the README.

## Adding a new skill
1. `mkdir <skill-name>/` and write `<skill-name>/SKILL.md` (frontmatter `name:` must match the folder).
2. Optionally add `<skill-name>/README.md` + `<skill-name>/assets/` for a rich doc.
3. Add ONE row to the root `README.md` skills table linking to `<skill-name>/`.
4. Commit and push (see below).

## Signature
End the root README with: **Built with heart. Rise with purpose.** ❤️🎈 (Tarun's North Star).
Keep public READMEs professional — NO private LifeOS details (finances, exit plan, etc.).

## Git / push notes
- Use `git mv` when relocating files so history is preserved.
- If `git push` fails with "Password authentication is not supported", run `gh auth setup-git` once,
  then push (gh is authenticated as `tatavarthitarun`).
- Pushing is outward-facing/public — fine to push skill content Tarun asked for; confirm if unsure.

## Generating README images
Render a self-contained HTML to PNG headlessly, then view it to verify before committing:
```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --hide-scrollbars --force-device-scale-factor=2 --window-size=1200,630 \
  --screenshot=out.png "file://$PWD/banner.html"
```
(Gotcha: a `background-image:` declaration overrides the `background:` shorthand's gradient — set
`background-color` + layer gradients in one `background-image` to keep a dark background.)
