![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*hVKWuT24riZnx4VzDpQVJQ.png)

Kotzilla Codelab: Fix Now in Android
==================

> **A self-paced codelab by [Kotzilla](https://kotzilla.io)**. Do it at home, at your own pace.

This app is a version of Google's **[Now in Android](https://github.com/android/nowinandroid)** where
Dagger Hilt is replaced by **Koin** (Koin Compiler Plugin), instrumented with the **Kotzilla SDK**.
It ships with **performance and stability bugs we introduced on purpose**: a crash, a slow cold
start, an ANR, and main thread and background thread blocks. Your mission: use the
**Kotzilla MCP Server** and your AI assistant to detect them from real runtime sessions, fix them,
and prove it with a before/after comparison.

You do not need to hunt for the bugs in the code. The Kotzilla Platform finds them from what the
app actually does at runtime, and your AI assistant fixes them with that evidence.

## 📖 Follow the codelab on doc.kotzilla.io

The full walkthrough, with explanations and Console screenshots, lives in the Kotzilla docs:

1. **[Overview and prerequisites](https://doc.kotzilla.io/docs/getstartedNIA/overview)**
2. **[Set up and capture a session](https://doc.kotzilla.io/docs/getstartedNIA/setup)**
3. **[Find and fix the issues](https://doc.kotzilla.io/docs/getstartedNIA/validationIssues)**
4. **[Verify your fixes](https://doc.kotzilla.io/docs/getstartedNIA/verify)**

The Hilt to Koin migration deep-dive lives in [`docs/KOIN_COMPILER_PLUGIN.md`](docs/KOIN_COMPILER_PLUGIN.md).

---

## Quick reference

### Prerequisites

- Android Studio with a working **emulator or device**.
- A free **[Kotzilla account](https://console.kotzilla.io/signup)**.
- An MCP-capable AI assistant connected to the Kotzilla MCP Server. For Claude Code:

```bash
claude mcp add kotzilla --transport http https://mcp.kotzilla.io/mcp
```

  For other clients, see the [MCP setup guide](https://doc.kotzilla.io/docs/getstartedCustom/mcpSetup).
  With the [Koin IDE Plugin](https://plugins.jetbrains.com/plugin/26131-koin-dependency-injection-official-),
  the **Connect AI agents** popup does it in one click.

- One successful build before you start (`./gradlew :app:assembleDemoDebug`). Now in Android is a
  30-module app, so the first build is slow.

### The steps and their prompts

| Step | What you do | Prompt |
| --- | --- | --- |
| 1. Register | Let your assistant register the app and write `app/kotzilla.json` | *Register this app on Kotzilla and configure the SDK.* |
| 2. Before session | `./gradlew :app:installDemoDebug`, cold start, follow a topic, scroll, search "compose", open Saved, open Interests (it crashes), relaunch | |
| 3. Before report | Read the FAIL report, about a dozen issues | *Generate a Kotzilla report for this app, version codelab-1.0.* |
| 4. Fix | The crash first, then everything that blocks startup | *My app crashes when I open the Interests screen. Find the root cause with Kotzilla and fix it.*<br/>*My cold startup is over 10 seconds. Find everything that blocks it and fix it.* |
| 5. After session | Bump `versionName` and `versionCode`, rebuild, run the Step 2 path **twice** | |
| 6. Compare | Check each issue against `codelab-1.0` | *On Kotzilla, compare codelab-1.0 with my latest version, for the crash on Interests and the issues on MainActivity, MainActivityViewModel and ListenableWorker. For each one: gone, or how far did it drop?* |

Expect the version to change as you go: the MCP fix flow verifies each fix on a fresh session, so
your assistant bumps `versionName` after each fix. Only your last build needs all the fixes.

---

## What you practiced

- Registering an app and configuring an SDK entirely through MCP.
- Turning runtime symptoms (a crash, a slow start, an ANR, frozen screens, background stalls,
  slow transitions) into component-level evidence with the Kotzilla Platform.
- Letting an AI assistant fix issues from real dependency graphs and session timings instead of
  guessing from code.
- Proving a fix by comparing versions on the platform.

---

# License

**Now in Android** is distributed under the terms of the Apache License (Version 2.0). See the
[license](LICENSE) for more information. Original project readme: [`README.original.md`](README.original.md).
