# Claude Code skills

> Agentic [Claude Code](https://claude.com/claude-code) skills I build and use.

## Skills

| Skill | What it does |
|-------|--------------|
| [**`generate-ui-tests`**](generate-ui-tests/) | A ReAct GUI agent that drives a live Android app (screenshot + accessibility-tree reason-act loop), grounded in source, then **writes & self-verifies** Compose/Espresso + JUnit tests. Write once, run in CI with no LLM in the loop. |
| [**`ship-project`**](ship-project/) | Ship a personal project end-to-end: `~/Projects` code + LifeOS bridge docs & symlink, public GitHub repo, product-page README, interactive HTML explainer, GitHub Pages. |
| [**`explainer-html`**](explainer-html/) | Build a self-contained interactive HTML explainer — analogy → technical → example, step-through visualizations, quiz tab — with every control verified working. |

## Install
Clone into your Claude Code user skills directory:

```bash
# macOS / Linux
git clone https://github.com/tatavarthitarun/claude-skills "$HOME/.claude/skills"
```
```powershell
# Windows
git clone https://github.com/tatavarthitarun/claude-skills "$env:USERPROFILE\.claude\skills"
```
Restart Claude Code; each skill is invocable as `/<skill-name>`.

> Already have skills in that folder? Clone elsewhere and copy the skill directory you want in.

---

<sub>Built by [Tarun Tatavarthi](https://github.com/tatavarthitarun) — Android engineer exploring agentic AI.</sub>

<sub>**Built with heart. Rise with purpose.** ❤️🎈</sub>
