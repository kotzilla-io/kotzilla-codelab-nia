![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*hVKWuT24riZnx4VzDpQVJQ.png)

Kotzilla Codelab: Fix Now in Android
==================

> **A self-paced codelab by [Kotzilla](https://kotzilla.io)**. Do it at home, at your own pace.

This app is a version of Google's **[Now in Android](https://github.com/android/nowinandroid)** where
Dagger Hilt is replaced by **Koin** (Koin Compiler Plugin), instrumented with the **Kotzilla SDK**.
It ships with **intentionally planted performance and stability bugs**. Your mission: use the
**Kotzilla MCP server** and your AI assistant to detect them from real runtime sessions, root-cause
them, fix them, and prove it with a before/after report.

You do not need to hunt for the bugs in the code. That is the point: the Kotzilla Platform finds
them from what the app actually does at runtime, and your AI assistant fixes them with that
evidence.

The Hilt to Koin migration deep-dive lives in [`docs/KOIN_COMPILER_PLUGIN.md`](docs/KOIN_COMPILER_PLUGIN.md).

---

## Prerequisites

1. **Kotlin** intermediate level, plus basic Android, ViewModel/Lifecycle, and DI familiarity.
2. **Android Studio** with the Android SDK and a working **emulator or device**.
3. **Clone this repo and run ONE successful Gradle sync + build** before starting
   (`./gradlew :app:assembleDemoDebug`). Now in Android is a 30-module app; the first build is slow.
4. **Create a free [Kotzilla account](https://console.kotzilla.io/signup)**. You need it before
   anything else: both the MCP server and the platform authenticate against this account.
5. **Connect an MCP-capable AI assistant** (Claude Code, Cursor, Windsurf, Android Studio) to the
   Kotzilla MCP server. For Claude Code:

```bash
claude mcp add kotzilla --transport http https://mcp.kotzilla.io/mcp
```

   For other clients, see the [MCP setup guide](https://doc.kotzilla.io/docs/getstartedCustom/mcpSetup).
   On first use, the server opens a browser auth flow with the account from step 4.

---

## Step 1: Register the app through MCP

No SDK key is bundled. Ask your assistant to set it up for you, in your own words, for example:

> "Register this app on Kotzilla and configure the SDK."

The MCP server guides the assistant through app registration and returns a `kotzilla.json`; the
assistant pastes it into `app/`. The SDK and Koin libraries are already wired into the project.

## Step 2: Capture a "before" session

Build and install the app (`./gradlew :app:installDemoDebug`), then run this navigation path:

1. Launch the app from the launcher (cold start). The first launch shows a notification permission
   dialog; either choice is fine. Notice how long the app takes to show the feed: that slowness is
   real, and it is being measured.
2. On the **For You** screen, follow a topic and scroll the feed.
3. Open **Search** (the magnifier, top left) and search for "compose".
4. Open the **Saved** tab.
5. Open the **Interests** tab. The app crashes. 💥 That crash is part of the codelab.
6. **Relaunch the app** and let it load: this uploads the crashed session. Then send the app to the
   background (home button) or close it.
7. Wait about a minute for the session to be processed.

## Step 3: The "before" report

Ask your assistant:

> "Generate a Kotzilla report for this app, version codelab-1.0."

You get a **FAIL** report listing around a dozen detected issues: a crash, a slow cold start, an
ANR, slow screens, and blocking components. The exact count varies with your device and how you
navigated, so do not worry if you see a few more or fewer. If the report shows no data yet, wait
another minute and ask again.

**Take a screenshot of this report.** You will attach it when you complete the codelab.

## Step 4: Diagnose and fix, issue by issue

Work top-down: the crash first, then everything blocking startup.
For each issue: read what is going on, optionally look at it in the
[Kotzilla Console](https://console.kotzilla.io/) (issues view, screen details, session timeline),
then hand it to your assistant. The assistant pulls the dependency trees, timings, and stack traces
from your real session through MCP, finds the root cause, and edits the code.

**Expect the version to change as you go.** The MCP fix flow verifies each fix against a fresh
session, so after every fix your assistant bumps `versionName` in `app/build.gradle.kts`, rebuilds,
re-runs the app, and re-checks the issue on that new version. Let it. Intermediate names like
`codelab-1.1`, `codelab-1.2` are fine, whatever it picks.

The only thing that matters for completion: once **all** the fixes are in, the final build must be
`codelab-2.0`. That is the version Step 5 measures and the one you submit.

### 4.1 The crash: Interests tab

Tapping **Interests** makes Koin instantiate `InterestsViewModel`. Its construction throws, Koin
wraps the failure in an `InstanceCreationException`, and the app dies. This is a classic
crash-on-resolution: the failure surfaces in the DI container, and the captured stack trace names
the exact component. Kotzilla recorded it even though the app died, because the session is uploaded
on the next launch.

> "My app crashes when I open the Interests screen. Find the root cause with Kotzilla and fix it."

**What the fix looks like:** the ViewModel's constructor enforces a precondition that can never be
met, so construction always throws. The fix removes that check (or makes the configuration truly
optional) so the ViewModel can be created.

### 4.2 Startup: everything that blocks the first frame

The report shows a cold start above 10 seconds, an **ANR** and a slow screen on MainActivity, a
main-thread issue on `MainActivityViewModel`, and a background-thread issue on the sync worker.
That reads like five problems. It is three blocking components, all inside the startup window, and
one prompt usually clears the lot:

> "My cold startup is over 10 seconds. Find everything that blocks it and fix it."

Expect your assistant to work through all three in one pass, rebuilding and re-measuring as it
goes. That is the tool working as intended, not it going off-script. Each one is a distinct
anti-pattern worth reading about as it gets fixed:

**`Application.onCreate`** runs synchronous "warm-up" work before any Activity can start. The most
expensive place to be slow: every user pays it, every launch, before seeing anything. The fix
removes it; anything genuinely needed at startup belongs on a background thread or deferred until
first use.

**`MainActivityViewModel`** takes about 5 seconds to resolve, on the main thread, while the screen
waits. Component resolution runs on the main thread by default, so a slow constructor freezes the
UI, and past 4.5 seconds of blocked rendering Kotzilla flags it as an **ANR** on top of the
slow-screen and main-thread issues. The tell is in the dependency tree: the whole subtree costs
single-digit milliseconds against a multi-second resolution, so the time is self time in the
constructor. A slow constructor, not a heavy graph. The fix removes the synchronous wait: a
ViewModel exposes state as a `StateFlow` and loads in `viewModelScope`, never in its constructor.

**The sync worker** (`androidx.work.ListenableWorker`) spends about 2 seconds blocked while being
created. It runs off the UI thread, so it is tempting to leave alone, but it is started from
`Application.onCreate` and resolves inside the cold-start window, so it counts. The fix removes the
check; a `CoroutineWorker` does its work in `doWork`, on the injected dispatcher, and its
construction stays instant.

If your assistant stops after the first component, point it at the rest:

> "MainActivityViewModel is still blocking the main thread. Diagnose it with Kotzilla and fix it."

> "Now analyze the background thread issue on the sync worker and fix it."

### 4.3 One more, and this one is not ours

Once the planted issues are gone, the report still shows a **slow transition** of roughly 700ms on
`InterestsRoute`, and similar numbers on Search and Saved. We did not plant that one. It is real,
it ships in Now in Android today, and it is a good example of what the tooling finds when nobody
put it there on purpose.

> "Now fix the slow transition on InterestsRoute."

The timeline is the whole story: the screen draws in about 11ms, then takes another ~700ms to reach
`RESUMED`. Nothing runs on the main thread in between. Three unrelated destinations landing on
nearly the same number is the clue: it is one shared cause, not three coincidences.

**What the fix looks like:** `NavHost` defaults to a 700ms crossfade, and a destination is not
interactive until that animation finishes. Overriding `enterTransition`/`exitTransition` to
200ms, inside Android's recommended 200-300ms band, gets the time back. Worth noticing that this
is a real UX decision, not just a metric: the delay *is* the animation.

This one is optional. Leaving it unfixed does not count against your completion.

## Step 5: Capture the "after" session

1. In `app/build.gradle.kts`, set `versionName` to `codelab-2.0` (you will be on whatever
   intermediate version Step 4 left you on, not on `codelab-1.0`), and bump `versionCode`.
2. Rebuild and install.
3. Run the Step 2 navigation path **twice** (this time Interests should not crash).
4. Background or close the app, then wait about a minute.

**If a run goes wrong, bump the version and redo it.** A version is a permanent measurement
bucket: reports aggregate every session ever recorded against it, and with only a handful of
sessions a single bad one sets the P95. An ANR is worse, since it is a counted event that no
amount of good runs can average away. So if a run gets interrupted or goes wrong, do not try to
fix it by running again on the same version. Set `versionName` to `codelab-2.1`, rebuild, and
measure clean. Submit whichever `codelab-2.x` you measured properly.

## Step 6: The before/after comparison

> "Generate a Kotzilla report for version codelab-2.0, compare it with the codelab-1.0 report, and
> summarize what changed."

(Use whichever `codelab-2.x` you actually measured, if a redo moved you off `codelab-2.0`.)

Your assistant pulls both version-scoped reports through MCP and builds the comparison.

**Expect the after report to still say FAIL, and finish anyway.** You are running on an emulator,
where cold start alone sits above the threshold no matter how good your code is. A green banner is
not the goal and chasing one will waste your time. What matters is that the specific issues you
were given are gone:

| Was in `codelab-1.0` | Should be absent in your final version |
| --- | --- |
| Crash on the Interests screen | no crashes |
| Slow cold start and warm start | startup down by several seconds |
| ANR and slow screen on `MainActivity` | neither |
| `MainActivityViewModel` blocking the main thread | absent |
| `ListenableWorker` blocking a background thread | absent |

That is the checklist we verify against, and it is the only one. Anything else the report mentions
is not something you failed to fix: emulator startup and first-composition times, the optional slow
transition from 4.3, or a slow `ForYouRoute` on a launch where the feed was still filling for the
first time. Report FAIL with that table satisfied means you finished. 🎉

Take a screenshot of that report, same as in Step 3.

The same story is visible in the [Kotzilla Console](https://console.kotzilla.io/): open the
**Dashboard**, switch the version filter between `codelab-1.0` and your final version, and watch the
issues, ANRs, startup times, and screen renderings improve between the two versions.

<!-- TODO(miguel): add a Console dashboard screenshot here, version filter on codelab-2.0 vs codelab-1.0 -->

## Completing the codelab

Email **both saved reports** (the codelab-1.0 "before" and your final codelab-2.x "after") plus
your **app name** as registered on Kotzilla to **codelab@kotzilla.io**. We verify completions
server-side. Every completer gets a shout-out, and completions enter a prize draw.

---

## What you practiced

- Registering an app and configuring an SDK entirely through MCP.
- Turning runtime symptoms (a crash, a slow start, an ANR, frozen screens, background stalls,
  slow transitions) into component-level evidence with the Kotzilla Platform.
- Letting an AI assistant fix issues from real dependency graphs and session timings instead of
  guessing from code.
- Proving a fix with version-scoped before/after reports, the same mechanism you would use as a
  release gate in CI.

---

# License

**Now in Android** is distributed under the terms of the Apache License (Version 2.0). See the
[license](LICENSE) for more information. Original project readme: [`README.original.md`](README.original.md).
