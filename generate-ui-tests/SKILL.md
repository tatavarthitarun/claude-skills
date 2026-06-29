---
name: generate-ui-tests
description: >-
  Autonomously explore a live Android app on a connected device (screenshot +
  accessibility-tree reason-act loop), grounded in its source code, then SYNTHESIZE
  durable Android UI tests (Compose test / Espresso) and JUnit unit tests, write them
  into the repo, and self-verify by compiling and running them. Plans coverage from the
  source navigation graph and explores loop-until-dry, handles auth-walled apps via
  credential injection / session seeding, and emits an honest coverage report (no silent
  gaps). Phase 1 (explore) + Phase 2 (synthesize). Use when the user wants an agent to
  write UI/unit tests for an Android app by driving it, not just to act as manual QA.
---

# generate-ui-tests

A **ReAct GUI agent** that authors durable Android tests. The LLM runs **once** at
authoring time; the **output is committed test code** that later runs in CI with no LLM.

> Two phases that must never be conflated:
> - **Author-time (this skill):** non-deterministic exploration + codegen on a dev machine.
> - **Run-time (CI, out of scope here):** deterministic `./gradlew` execution.

The agent has **two grounding sources** and must use BOTH:
1. **Source code** → stable selectors (string resources, `testTag`s, content descriptions),
   navigation graph, and logic worth unit-testing. Source says *how to target* and *what should exist*.
2. **Live device (screenshots + UI hierarchy)** → what *actually renders*, async states, dialogs.
   The device says *what really happens* and grounds assertions.

---

## Preconditions (verify before starting; stop and report if unmet)

1. **adb reachable.** First try `adb version`. If not found, locate the SDK and put its
   `platform-tools` (and `emulator`) on PATH for the session — paths differ per OS/shell:
   - **macOS/Linux (bash/zsh):** `export PATH="$ANDROID_HOME/platform-tools:$PATH"`, where
     `ANDROID_HOME` is typically `~/Library/Android/sdk` (macOS) or `~/Android/Sdk` (Linux).
   - **Windows (PowerShell):** `$env:Path += ";$env:LOCALAPPDATA\Android\Sdk\platform-tools"`
     (adb is `adb.exe`; SDK usually `%LOCALAPPDATA%\Android\Sdk`).
   - **Windows (Git Bash):** `export PATH="$LOCALAPPDATA/Android/Sdk/platform-tools:$PATH"`.
   - Honor an existing `ANDROID_HOME` / `ANDROID_SDK_ROOT` env var if set. Detect the OS first
     (`uname` / `$OS`) and use the matching form rather than assuming macOS.
2. **A device is authorized.** Run `adb devices -l`.
   - `device` → good.
   - `unauthorized` → STOP. Ask the user to unlock the phone and tap **"Allow USB debugging"**
     (check "Always allow"). Re-poll until authorized. Do not proceed.
   - `offline` / empty → ask the user to reconnect / re-enable USB debugging.
3. **Project builds.** Confirm the Gradle wrapper exists and an assemble succeeds. **Use the wrapper
   form for the OS:** `./gradlew <task>` on macOS/Linux/Git Bash, `.\gradlew.bat <task>` (or `gradlew.bat`)
   on Windows PowerShell/CMD. All `./gradlew` commands below follow this convention.
4. **Detect app facts** from source (do not hardcode):
   - `applicationId` / `namespace` and launcher activity from `app/build.gradle.kts` + `AndroidManifest.xml`.
   - **UI toolkit:** if `buildFeatures { compose = true }` and `activity-compose` are present →
     **Compose** (use Compose test APIs). Else → **Views** (use Espresso view matchers).
   - Existing `src/androidTest` / `src/test` conventions and the `testInstrumentationRunner`.
   - Whether `androidx.ui.test.junit4` (Compose) / `espresso.core` are in `androidTestImplementation`;
     if missing for the detected toolkit, add them.

Use the scratchpad for all transient artifacts (screenshots, dumps, the trace, the ledger):
`<scratchpad>/generate-ui-tests/` (screenshots `screen_NN.png`, dumps `ui_NN.xml`, `trace.json`,
`coverage.json`).

---

## Inputs & configuration (gather up front; ask only if blocked)

Read these from the skill args / a `generate-ui-tests.json` in the repo root / env, else use defaults.
Surface what you're using so the run is reproducible.

- **Auth** (`auth`): how to get past a login wall, if one exists. One of:
  - `credentials`: `{ usernameField, passwordField, username, password, submit }` — selectors + test creds.
    Prefer creds from env (`IWT_TEST_USER` / `IWT_TEST_PASS`), never hard-code real secrets into tests.
  - `seed`: a way to pre-authenticate WITHOUT driving the UI — strongly preferred for speed & stability:
    - DataStore/SharedPreferences/Room write (`adb shell run-as <app> ...` on debuggable builds), or
    - a deep link / `am start ... -e token <…>`, or a debug-only login backdoor.
  - `none`: no auth needed (default).
  - If a login wall is detected and no `auth` is configured → STOP and ask for test credentials or a seed
    recipe. Do not attempt to bypass real authentication.
- **Coverage target** (`coverage`): `{ maxSteps: 120, dryRounds: 3, revisitCap: 2 }` (defaults).
  These bound exploration; every item dropped because of a bound is logged, never silently skipped.
- **Destructive opt-in** (`allowDestructive`): `false` by default. When `true`, list which actions
  (e.g. delete, sign-out) are in scope; otherwise they are recorded as deliberately-uncovered.

---

## Phase 0 — Build, install, prepare state

```bash
./gradlew :app:assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell pm clear <applicationId>            # reset to clean first-run state
# Pre-grant runtime permissions to avoid system dialogs derailing exploration:
adb shell pm grant <applicationId> android.permission.POST_NOTIFICATIONS 2>/dev/null || true
```

Launch: `adb shell am start -n <applicationId>/<launcherActivity>` (e.g. `com.tatav.iwt/.MainActivity`).

**Authenticate (if needed).** If `auth.seed` is configured, apply it now (write prefs/DB/token, or
fire the deep link) and relaunch so exploration starts already logged-in. If `auth.credentials`, the
login is exercised as the first explored flow (and becomes a real `LoginTest`). Re-verify the app
landed past the wall before continuing.

**Build the expected-screen map (symbolic planning input).** Before driving, read the navigation graph
from source so you know the *universe* of screens to aim for — not just what you stumble into:
- Compose: `composable("<route>")` / `navigate("<route>")` entries in the `NavHost`.
- Views/Fragments: nav-graph XML `<fragment>`/`<action>`, or `startActivity`/intent-filter targets.
- Record these as the **expected screens** in `coverage.json`. Exploration's job is to reach each one;
  any never reached is a logged gap with a reason.

---

## Phase 1 — Explore (planning-driven, loop-until-dry)

Systematic coverage, not a fixed walk. Maintain three structures in `coverage.json`, updated every step:
- **expected**: screens from the source nav graph (Phase 0) — the denominator for coverage.
- **visited**: `{ screenSignature → { name, interactiveElements[], exercised[] } }`. A screen signature is
  a stable hash of its resource-ids / content-descs (not volatile text/values).
- **frontier**: a worklist of `(screenSignature, unexercised interactive element)` pairs still to try.

**The loop (plan → act → observe → update frontier), until the frontier is dry:**

1. **Plan.** Pop the highest-value item from the frontier. Prioritise: (a) an action that should reach an
   *expected screen not yet visited*, then (b) breadth — an unexercised control on the current or a nearby
   screen, then (c) depth. Navigate to that item's screen if not already there (replay a known path, or
   relaunch + renavigate). This beats greedy random-walk: it targets the known-but-unreached.

Each step then:

2. **Observe — capture both:**
   ```bash
   adb exec-out screencap -p > <scratchpad>/.../screen_NN.png
   adb exec-out uiautomator dump /dev/tty > <scratchpad>/.../ui_NN.xml   # if /dev/tty fails:
   # adb shell uiautomator dump /sdcard/ui.xml && adb pull /sdcard/ui.xml ui_NN.xml
   ```
   **Windows PowerShell note:** `>` corrupts binary output, so the `exec-out > file.png` form mangles the
   PNG. Use the on-device-then-pull form on Windows (works everywhere, so prefer it if unsure):
   `adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png screen_NN.png` (same for the UI dump).
   `Read` the screenshot (vision) AND parse the XML. The XML gives every node's
   `text`, `content-desc`, `resource-id`, `clickable`, and `bounds="[x1,y1][x2,y2]"`.
   **Compute tap coordinates from `bounds` (center point) — never guess pixels.**

3. **Identify the screen** by its signature. If new, add to `visited`, enumerate ALL its interactive
   elements (clickable / editable / checkable / scrollable nodes), and push each as a frontier item.
   Match it against an `expected` screen and mark that expected entry reached.

4. **Act** on the planned item (skip destructive/irreversible actions unless `allowDestructive`):
   ```bash
   adb shell input tap <cx> <cy>
   adb shell input text "..."         # for fields; escape spaces as %s
   adb shell input keyevent KEYCODE_BACK   # to back out
   adb shell input swipe x1 y1 x2 y2 300   # scroll
   ```
   Wait briefly for UI to settle (poll the dump rather than fixed long sleeps).

5. **Update frontier & ledger.** Mark the acted element `exercised`. Diff the resulting screen: if it's
   new, enqueue its elements; if an action reached an `expected` screen, mark it reached. Append to
   `trace.json`:
   ```json
   {
     "step": 12, "screen": "Settings", "action": "tap",
     "target": {"text": "Dark theme", "contentDesc": null, "resourceId": null,
                "bounds": [44,910,1036,1040], "tapped": [540,975]},
     "result_screen": "Settings", "observation": "Toggle switched on; background darkened",
     "screenshot": "screen_12.png"
   }
   ```

**Loop-until-dry stopping rule** (replaces a fixed step count): keep going until the **frontier is empty**,
OR `dryRounds` consecutive steps discover no new screen AND no new interactive element, OR the `maxSteps`
hard cap is hit. A simple "covered the top-level" heuristic misses the tail — drive to dry.

When stopped, summarize the discovered **flows** (ordered step sequences that accomplish a user goal)
and finalize `coverage.json`.

> If exploration stalls (modal you can't dismiss, unconfigured login wall, a control that does nothing),
> record it in the ledger with a reason, back out, and move to the next frontier item — never burn the
> budget retrying the same stuck action.

---

## Phase 2 — Synthesize tests

### 2a. UI tests (toolkit-specific)

**Compose app (IWt's case):** generate `androidTest` classes using:
```kotlin
@get:Rule val composeTestRule = createAndroidComposeRule<MainActivity>()
// finders, in order of preference:
composeTestRule.onNodeWithTag("home_start_button")        // best: stable, add to source if absent
composeTestRule.onNodeWithText("Start")                   // ok: from string resources / observed text
composeTestRule.onNodeWithContentDescription("Settings")  // ok: a11y labels
// actions + assertions:
  .performClick();  .performTextInput("...")
  .assertIsDisplayed();  .assertIsOn()
composeTestRule.waitUntil(5_000) { composeTestRule.onAllNodesWithText("...").fetchSemanticsNodes().isNotEmpty() }
```
- **Resolve every selector against source.** Prefer string-resource values over literals; if a
  target has no stable identifier and its text is dynamic, **add `Modifier.testTag("...")`** to
  the composable in `src/main` and use that tag (note each source edit explicitly to the user).
- Use Compose **idling** (`waitUntil`, `waitForIdle`) — never `Thread.sleep`.
- One test class per flow/screen; one `@Test` per coherent user flow. Assertions must verify the
  **correct resulting state** observed on device, not merely that a tap didn't crash.

**Views app:** classic Espresso (`onView(withId(R.id.x))...perform(click())...check(matches(isDisplayed()))`),
reusing the real `R.id`s read from source, plus `IdlingResource` for async.

### 2b. Unit tests
From the source read during setup, generate `src/test` JUnit tests for pure logic — repositories,
view-model state transitions, formatters/util (e.g. IWt's `WorkoutHistoryRepository`, phase/streak
logic). Mock Android deps; keep them JVM-only (fast, no device).

### 2c. Auth in tests
If the app needs auth, the generated UI tests must reach a logged-in state deterministically:
- **Preferred — seed in `@Before`:** a `setUp()` that applies the same `auth.seed` (write
  prefs/DB/token, or launch a deep link) so every test starts authenticated without UI login. Put this
  in a shared base class / test rule so all flow tests reuse it.
- **Otherwise — a `loginViaUi()` helper** using `auth.credentials`, pulling creds from
  `InstrumentationRegistry` args / `BuildConfig` (injected from env/CI secrets), never literals.
- Also generate one explicit **`LoginTest`** that asserts the wall → authenticated transition.

### 2d. Placement & conventions
Write into `app/src/androidTest/java/<pkg>/ui/...` and `app/src/test/java/<pkg>/...`, matching the
project's existing package layout and style. Don't overwrite hand-written tests — add alongside.

---

## Phase 2.5 — Coverage report (honest, no silent gaps)

From `coverage.json`, write `app/build/reports/generate-ui-tests/coverage.md` and summarise to the user:
- **Screens:** `visited / expected` with a checklist; every expected-but-unreached screen listed with a
  reason (`blocked: login`, `unreachable from launcher`, `destructive: opted-out`, `dynamic/server data`).
- **Interactive elements:** `exercised / discovered` per screen.
- **Flows → tests:** which generated test covers which flow; flows discovered but NOT turned into a test.
- **Dropped by bounds:** anything skipped due to `maxSteps` / `revisitCap` / a stuck modal — named, not hidden.

The guiding rule: **coverage is a measured, reported number, never implied.** "I explored everything" is
only ever true relative to this ledger; say what you did and did not reach.

---

## Phase 3 — Self-verify (NON-NEGOTIABLE gate)

Generated code is untrusted until it compiles and runs green. Loop up to 3×:

```bash
./gradlew :app:assembleDebugAndroidTest        # compile UI tests — catches bad selectors/APIs
./gradlew :app:testDebugUnitTest               # run unit tests
./gradlew :app:connectedDebugAndroidTest       # run UI tests on the device
```
Read failures, fix the generated test (or selector/testTag), re-run. A test that can't be made
to pass within the loop must be **quarantined** (marked `@Ignore` with a `// TODO` reason) and
reported, not left red.

Reports are emitted by Gradle automatically:
- Unit: `app/build/reports/tests/testDebugUnitTest/index.html`
- UI: `app/build/reports/androidTests/connected/index.html`

---

## Final output to the user

- List of test files created (UI + unit), with the flow each covers.
- Any **source edits** made (e.g. added `testTag`s) — call these out explicitly.
- Pass/fail summary from Phase 3 + paths to the HTML reports.
- **Coverage summary** (from Phase 2.5): screens reached / expected, elements exercised, and the explicit
  list of what was NOT covered and why. Link `coverage.md`.
- **Auth**: how the wall was handled (seed vs UI login) and where creds come from.
- Reminder that these now run in CI via `./gradlew connectedAndroidCheck` with no LLM (out of scope here).

## Guardrails
- Never take irreversible in-app actions during exploration unless `allowDestructive` is set.
- Never hard-code real credentials or secrets into tests or the repo; inject via env/CI args.
- Never attempt to bypass a real authentication wall — require a seed recipe or test credentials.
- All screenshots/dumps stay in scratchpad; never commit them.
- Bound the loop (`maxSteps`, `revisitCap`, `dryRounds`) — but **log every item dropped by a bound**;
  silent truncation reads as "covered everything" when it isn't.
- Don't put the LLM in any runtime/CI path; this skill is author-time only.
