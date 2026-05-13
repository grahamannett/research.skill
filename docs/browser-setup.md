# Browser setup — Chrome DevTools MCP attached to your real Chrome

Both `claude-research` and `chatgpt-research` drive their respective websites using a [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp) server **attached to your real, logged-in Chrome**. This page documents the one-time setup. It applies regardless of which skill you're running or which agent harness (Claude Code, OpenAI Codex CLI, opencode) you're using.

## Verified setup (macOS, Chrome 144+)

The one-shot known-working configuration for Claude Code:

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
```

Then in your normal Chrome, open `chrome://inspect/#remote-debugging` once and enable remote debugging. The MCP will attach to your real, logged-in Chrome on the next session — no separate Chrome process, no separate profile, no relaunch dance. This is the recommended path; the rest of this doc covers the `--browser-url` fallback for Chrome <144 or other edge cases, plus alternative MCP installs.

> **Codex / opencode:** the underlying MCP server is the same; only the registration command differs.
> - Codex CLI: register the MCP server via Codex's plugin / MCP config; pass `--autoConnect` as the server argument.
> - opencode: add an entry to `opencode.json` under `mcp` with `command: ["npx", "-y", "chrome-devtools-mcp@latest", "--autoConnect"]`.

## Why "attach mode" matters

By default, `chrome-devtools-mcp` **launches its own isolated Chrome** with a throwaway profile at `~/.cache/chrome-devtools-mcp/chrome-profile-stable` — same Chrome binary as your normal one, but no logins, no extensions, no history. That's not what you want for these skills: they need your real logged-in claude.ai / chatgpt.com session.

You need to register the MCP with one of these flags so it **attaches to your real Chrome** instead.

For chatgpt.com specifically, attach mode is doubly important: chatgpt.com is Cloudflare-protected, and a fresh-profile headless-ish browser is far more likely to be challenged than a real session with established cookies, history, and fingerprint.

## Option A — `--autoConnect` (Chrome 144+, recommended)

Check your Chrome version at `Chrome → About Google Chrome`. If it's 144 or newer:

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
```

Then in your normal Chrome, open `chrome://inspect/#remote-debugging` and enable remote debugging once. From the [official docs](https://github.com/ChromeDevTools/chrome-devtools-mcp): *"best for sharing state between manual and agent-driven testing."*

## Option B — `--browser-url` (any Chrome version)

You launch Chrome with `--remote-debugging-port=9222` yourself; the MCP connects to that port.

Chrome refuses `--remote-debugging-port` against your default profile dir (per its own security policy), so use a dedicated profile dir. Quit any Chrome using that profile first, then:

**macOS:**

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.config/chrome-research-profile"
```

**Linux:**

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

The dedicated profile persists across launches — log into claude.ai and/or chatgpt.com in that window once and you're done.

## Playwright MCP (alternative, not required)

Both skills auto-detect [Playwright MCP](https://github.com/microsoft/playwright-mcp) as a fallback if Chrome DevTools MCP isn't available. Configure it the same way — point it at the same `localhost:9222`:

```bash
claude mcp add playwright -- npx -y @playwright/mcp@latest --cdp-endpoint http://localhost:9222
```

Stay with one MCP per skill run — don't have both Chrome DevTools MCP and Playwright MCP fighting over the same browser context simultaneously.

## Troubleshooting

**A fresh Chrome window opened with no logins.**
Your MCP is registered without `--autoConnect` or `--browser-url`, so it's launching its own isolated Chrome with a throwaway profile. Re-register it with one of the flags above. Confirm with `claude mcp list` (or your harness's equivalent).

**`curl http://localhost:9222/json/version` returns nothing.**
Chrome isn't running with `--remote-debugging-port=9222`, or another Chrome is holding the profile lock on the dedicated user-data-dir. Quit Chrome instances using that dir and relaunch.

**`--autoConnect` doesn't attach.**
Confirm Chrome version is 144+, that you enabled remote debugging at `chrome://inspect/#remote-debugging`, and that no other client (e.g. a stale Playwright session) is holding the connection.

**The skill says "not logged in".**
Open the relevant site (`https://claude.ai` or `https://chatgpt.com`) in the Chrome window the MCP is attached to, and log in. The skills won't try to log in for you.

**The skill says "Cloudflare challenge".** (chatgpt-research only.)
Cloudflare is asking your browser to verify it's human. Complete the challenge manually in your Chrome window (the Turnstile checkbox, plus any image puzzle), then re-invoke the skill. If challenges keep recurring even after attach mode is confirmed, your session may have aged out — refresh chatgpt.com and re-authenticate if needed. Don't run the skill repeatedly into a challenge — that can escalate to a temporary IP block.

**The MCP isn't being detected.**
Run `/mcp` inside Claude Code (or the equivalent listing in your harness) to confirm the server is connected. If the status shows "failed", check stderr — usually a Node / `npx` version issue.
