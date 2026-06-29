# Claude Code skills

Personal [Claude Code](https://claude.com/claude-code) skills. Clone into your user skills directory:

- **macOS/Linux:** `~/.claude/skills/`
- **Windows:** `%USERPROFILE%\.claude\skills\`

```bash
git clone https://github.com/tatavarthitarun/claude-skills "$HOME/.claude/skills"
```

## Skills

### `generate-ui-tests`
A **ReAct GUI agent** that authors durable Android UI tests. It explores a live app on a
connected device (screenshot + accessibility-tree reason-act loop), grounded in the app's
source code, then **synthesizes** Compose/Espresso UI tests + JUnit unit tests, writes them
into the repo, and **self-verifies** by compiling and running them on the device.

The LLM runs **once** at authoring time; the output is committed test code that runs in CI
with no LLM in the loop.

Highlights:
- **Two grounding sources:** source code (stable selectors, nav graph, unit-testable logic) +
  the live device (what actually renders, async states).
- **Planning + loop-until-dry** exploration driven by the source navigation graph.
- **Auth handling** for login-walled apps (credential injection or session seeding).
- **Honest coverage report** — screens/elements reached vs. expected, with reasons for any gaps.
- **OS-agnostic** (macOS / Linux / Windows).
