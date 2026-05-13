# claude-research

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that runs a **Research** query on [claude.ai](https://claude.ai) using a logged-in browser session via [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp), then returns the report back into your Claude Code conversation.

Distributed as a Claude Code plugin. This repo is a single-plugin marketplace, so installation is two `/plugin` commands.

## Verified setup (macOS, Chrome 144+)

The one-shot known-working configuration:

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
```

Then in your normal Chrome, open `chrome://inspect/#remote-debugging` once and enable remote debugging. The MCP will attach to your real, logged-in Chrome on the next session — no separate Chrome process, no separate profile, no relaunch dance. This is the recommended path; the longer setup section below covers the `--browser-url` fallback for Chrome <144 or other edge cases.

---

## How it works

1. You set up Chrome DevTools MCP to **attach to a running Chrome** (not launch its own).
2. You log into claude.ai in that Chrome once. The session persists in that profile.
3. From inside Claude Code, you trigger the skill. It attaches via the MCP, navigates to claude.ai, and verifies you're logged in.

No API keys, no separate auth — the skill uses your real claude.ai session.

---

## Prerequisites

### 1. Chrome DevTools MCP, configured to *attach*

By default, `chrome-devtools-mcp` launches its own isolated Chrome with a throwaway profile at `~/.cache/chrome-devtools-mcp/chrome-profile-stable` — same Chrome binary as your normal one, but no logins. That's not what you want. You need to register it with one of these flags so it attaches to your real Chrome instead.

Pick **one** path:

#### Option A — `--autoConnect` (Chrome 144+, recommended)

Check your Chrome version at `Chrome → About Google Chrome`. If it's 144 or newer:

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
```

Then in your normal Chrome, open `chrome://inspect/#remote-debugging` and enable remote debugging once. From the [official docs](https://github.com/ChromeDevTools/chrome-devtools-mcp): *"best for sharing state between manual and agent-driven testing."*

#### Option B — `--browser-url` (any Chrome version)

You launch Chrome with `--remote-debugging-port=9222` yourself; the MCP connects to that port.

Chrome refuses `--remote-debugging-port` against your default profile dir (per its own security policy), so use a dedicated profile dir. Quit any Chrome using that profile first, then:

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.config/chrome-research-profile"
```

Linux:

```bash
google-chrome --remote-debugging-port=9222 --user-data-dir="$HOME/.config/chrome-research-profile"
```

Verify it's reachable:

```bash
curl -s http://localhost:9222/json/version | head -1
```

Then register the MCP to attach to it:

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --browser-url=http://127.0.0.1:9222
```

The dedicated profile persists across launches — you log into claude.ai in that window once and you're done.

### 2. Logged into claude.ai

In whichever Chrome window the MCP will attach to (your normal one for Option A, the dedicated `--remote-debugging-port` window for Option B), go to [https://claude.ai](https://claude.ai) and log in if you aren't already.

### Playwright MCP (alternative, not required)

The skill auto-detects [Playwright MCP](https://github.com/microsoft/playwright-mcp) as a fallback if Chrome DevTools MCP isn't available. Configure it the same way — point it at the same `localhost:9222`:

```bash
claude mcp add playwright -- npx -y @playwright/mcp@latest --cdp-endpoint http://localhost:9222
```

---

## Install the plugin

From inside Claude Code:

```text
/plugin marketplace add https://github.com/grahamannett/research.skill
/plugin install claude-research@claude-research
```

The first command registers this repo as a marketplace. The second installs the plugin from it.

To update:

```text
/plugin marketplace update claude-research
/plugin install claude-research@claude-research
```

To uninstall:

```text
/plugin uninstall claude-research
```

---

## Usage

Once installed and the MCP is in attach mode, ask Claude Code in natural language:

- *"Use claude-research to investigate the state of Rust async runtimes."*
- *"Kick off a deep research query on claude.ai about WebGPU adoption."*
- *"Run the claude-research preflight to confirm my setup."* (skips the query — just verifies MCP attach + login)

The skill will navigate claude.ai, enable Research mode via the composer's `+ → Research` menu, submit the query, wait for completion (5–30+ min), and return the report text plus the claude.ai conversation URL.

### Clarification questions

claude.ai's Research mode usually asks 1–3 follow-up questions (scope, time period, depth, etc.) before kicking off the real run. By **default the skill auto-answers** those — drawing the reply from your original query, deferring to claude.ai's defaults when your query is silent — so the run starts unattended.

The skill returns a short audit of those auto-replies so you can see what it answered on your behalf.

If you want to be in the loop on the plan, say so in the request:

- *"Research X on claude.ai, but **ask me before answering** any clarifying questions."*
- *"Kick off a research query on Y, **interactive** mode."*

---

## Repo layout

```
research.skill/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace manifest (so this repo is installable as-is)
├── skills/
│   └── claude-research/
│       └── SKILL.md         # The skill's prompt + procedure
├── README.md
└── .gitignore
```

---

## Troubleshooting

**A fresh Chrome window opened with no logins.**
Your MCP is registered without `--autoConnect` or `--browser-url`, so it's launching its own isolated Chrome with a throwaway profile. Re-register it with one of the flags above. Confirm with `claude mcp list`.

**`curl http://localhost:9222/json/version` returns nothing.**
Chrome isn't running with `--remote-debugging-port=9222`, or another Chrome is holding the profile lock on the dedicated user-data-dir. Quit Chrome instances using that dir and relaunch.

**`--autoConnect` doesn't attach.**
Confirm Chrome version is 144+, that you enabled remote debugging at `chrome://inspect/#remote-debugging`, and that no other client (e.g. a stale Playwright session) is holding the connection.

**The skill says "not logged in".**
Open `https://claude.ai` in the Chrome window the MCP is attached to, and log in. The skill won't try to log in for you.

**The MCP isn't being detected.**
Run `/mcp` inside Claude Code to confirm the server is connected. If the status shows "failed", check stderr — usually a Node / `npx` version issue.

---

## License

MIT.
