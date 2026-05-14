# Browser setup — Chrome DevTools MCP, attach mode

Both skills need [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp) registered so it drives your real, logged-in Chrome — not a throwaway profile.

## Option A — `--autoConnect` (Chrome 144+, recommended)

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
```

Then open `chrome://inspect/#remote-debugging` in your normal Chrome once and enable remote debugging. The MCP attaches on the next session.

## Option B — `--browser-url` (any Chrome version)

Launch Chrome yourself with a dedicated profile dir (Chrome refuses `--remote-debugging-port` on the default profile):

```bash
# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.config/chrome-research-profile"

# Linux
google-chrome --remote-debugging-port=9222 --user-data-dir="$HOME/.config/chrome-research-profile"
```

Register the MCP against it:

```bash
claude mcp remove chrome-devtools 2>/dev/null
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --browser-url=http://127.0.0.1:9222
```

Log into claude.ai / chatgpt.com in that window once; the profile persists.

## Codex / opencode

Same MCP, different registration:

- **Codex CLI:** register `chrome-devtools-mcp@latest` via Codex's MCP config with `--autoConnect` (or `--browser-url=…`) as the server argument.
- **opencode:** add an entry to `opencode.json` under `mcp` with `command: ["npx", "-y", "chrome-devtools-mcp@latest", "--autoConnect"]`.

## Troubleshooting

- **A fresh Chrome window with no logins opened.** MCP is in launch mode — re-register with `--autoConnect` or `--browser-url`. Confirm via `claude mcp list`.
- **`curl http://localhost:9222/json/version` returns nothing.** Chrome isn't running with `--remote-debugging-port=9222`, or another Chrome holds the profile lock. Quit and relaunch.
- **Skill says "not logged in".** Open the site in the attached Chrome window and log in. The skills won't log in for you.
- **Skill says "Cloudflare challenge"** (`chatgpt:research`). Complete the Turnstile in your Chrome window, then re-invoke. Don't keep retrying into a challenge — that can escalate to a temporary IP block.
