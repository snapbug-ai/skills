# Snapbug agent skills

Let your AI coding agent add [Snapbug](https://snapbug.ai) to your app, so you never write the integration by hand. Snapbug is a debugging and bug-reporting SDK for Android, iOS, Flutter and React Native.

The `snapbug-install` skill detects the platform, adds the SDK with the release no-op twin (nothing ships to end users), writes the one-line start call, hooks your HTTP client and runs a build to verify.

## Install

Any agent (Claude Code, Cursor, Codex, Gemini CLI, Copilot and others):

```bash
npx skills add snapbug-ai/skills
```

Claude Code plugin marketplace:

```
/plugin marketplace add snapbug-ai/skills
/plugin install snapbug@snapbug
```

Manual: copy `skills/snapbug-install/` into your agent's skills folder, for example `.claude/skills/` in your project.

## Use

Ask your agent:

> Add Snapbug to this app.

No skill installed? Paste this prompt instead. It points the agent to the same instructions:

> Add the Snapbug SDK to this project. Follow https://snapbug.ai/llms-full.txt exactly, then build the debug and release variants to verify.

---

This folder is published from the Snapbug monorepo (`agent-skills/`). Versions in the reference files are kept in sync by `scripts/sync-versions.mjs`. Do not edit them by hand.
