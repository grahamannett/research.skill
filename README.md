# research.skill

Two skills that drive a Research / Deep Research run on claude.ai or chatgpt.com from inside your AI coding agent. Both attach to your real, logged-in Chrome via [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp).

- `claude-research` → claude.ai — verified end-to-end.
- `chatgpt-research` → chatgpt.com — written but not yet verified live.

## Setup

Chrome DevTools MCP must be registered in **attach mode** (not launch mode) so it drives your logged-in browser. See [`docs/browser-setup.md`](docs/browser-setup.md).

## Install

### Claude Code

```text
/plugin marketplace add https://github.com/grahamannett/research.skill
/plugin install claude-research@research-skill
/plugin install chatgpt-research@research-skill
```

Update: `/plugin marketplace update research-skill`. Uninstall: `/plugin uninstall <name>`.

### OpenAI Codex CLI

```bash
git clone https://github.com/grahamannett/research.skill ~/code/research.skill
ln -s ~/code/research.skill/claude-research/skills/claude-research ~/.agents/skills/claude-research
ln -s ~/code/research.skill/chatgpt-research/skills/chatgpt-research ~/.agents/skills/chatgpt-research
```

### opencode

Same as Codex, but symlink into `~/.config/opencode/skills/` (or `.claude/skills/`).

## Usage

Once installed and MCP is in attach mode, ask in natural language:

- *"Use claude-research to investigate the state of Rust async runtimes."*
- *"Run a ChatGPT Deep Research on WebGPU adoption."*
- *"Run the claude-research preflight to confirm my setup."* (verifies MCP + login, skips the query)

Each skill returns the report text plus the conversation URL. Runs take 5–30+ minutes.

### Clarifications

Both services ask 1–3 follow-up questions before starting. The skills **auto-answer by default**, drawing from your original query. Append "ask me" or "interactive" to your request to be looped in.

### Kickoff mode (chatgpt-research only)

After clarifications, ChatGPT shows a plan panel with Edit / Cancel / Start.

- **brief-wait** (default): waits ~12s, then starts.
- **immediate**: starts the moment the panel renders. Trigger: "no wait", "fully automated".
- **manual**: doesn't click Start — returns the URL for you to review. Trigger: "show me the plan first", "manual kickoff".

### chatgpt.com caveats

- Requires Deep Research access (Plus / Pro / Team / Enterprise / Edu). Free tier is gated.
- chatgpt.com is Cloudflare-protected. Attach mode is the mitigation. If a challenge appears, the skill stops and asks you to complete it manually — it will not click through automatically.

## License

MIT.
