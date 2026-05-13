# research.skill

Research-driving skills for AI coding agents. Both skills attach to your real, logged-in Chrome via [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp) and drive a research feature on a Claude or OpenAI website without you having to context-switch out of your terminal.

| Skill              | Target site   | Default harness         | Status                                                |
|--------------------|---------------|-------------------------|-------------------------------------------------------|
| `claude-research`  | claude.ai     | Claude Code             | Stable — verified end-to-end                          |
| `chatgpt-research` | chatgpt.com   | OpenAI Codex CLI, opencode (also Claude Code) | Best-guess — written but not yet verified live |

Distributed as:

- **Claude Code:** a multi-plugin marketplace (`research-skill`) exposing each skill as a separately-installable plugin.
- **Codex CLI / opencode:** the `SKILL.md` files are portable — symlink the skill dir of your choice into the harness's skills path.

## Shared setup

Both skills require Chrome DevTools MCP configured to **attach to your real, logged-in Chrome** (not launch its own throwaway profile). One-time setup:

📄 See **[`docs/browser-setup.md`](docs/browser-setup.md)** — covers Verified setup (`--autoConnect`, Chrome 144+), the `--browser-url` fallback, Playwright MCP alternative, and troubleshooting.

---

## Install

### Claude Code

```text
/plugin marketplace add https://github.com/grahamannett/research.skill
/plugin install claude-research@research-skill
```

or

```text
/plugin install chatgpt-research@research-skill
```

Install only the skill(s) you actually want — they're independent.

> **Breaking-change note:** the marketplace was renamed from `claude-research` to `research-skill` in version 0.2.0 to reflect that it now hosts multiple plugins. If you previously ran `/plugin marketplace add` against an older version:
>
> ```text
> /plugin marketplace remove claude-research
> /plugin marketplace add https://github.com/grahamannett/research.skill
> /plugin install claude-research@research-skill
> ```

To update:

```text
/plugin marketplace update research-skill
```

To uninstall:

```text
/plugin uninstall claude-research
/plugin uninstall chatgpt-research
```

### OpenAI Codex CLI

Clone the repo somewhere stable, then symlink the skill dir(s) you want into `~/.agents/skills/`:

```bash
git clone https://github.com/grahamannett/research.skill ~/code/research.skill
ln -s ~/code/research.skill/chatgpt-research/skills/chatgpt-research ~/.agents/skills/chatgpt-research
# (or claude-research/skills/claude-research, or both)
```

Copying instead of symlinking is fine if you'd rather not chase updates.

### opencode

Same idea, different path:

```bash
git clone https://github.com/grahamannett/research.skill ~/code/research.skill
ln -s ~/code/research.skill/chatgpt-research/skills/chatgpt-research ~/.config/opencode/skills/chatgpt-research
```

opencode also reads `.claude/skills/` directories, so a Claude-Code-style install would work too.

---

## Usage

### claude-research

Once installed and your MCP is in attach mode, ask in natural language:

- *"Use claude-research to investigate the state of Rust async runtimes."*
- *"Kick off a deep research query on claude.ai about WebGPU adoption."*
- *"Run the claude-research preflight to confirm my setup."* (skips the query — just verifies MCP attach + claude.ai login)

The skill navigates claude.ai, enables Research via the composer's `+ → Research` menu, submits the query, auto-answers clarifications (see below), waits for completion (5–30+ min), and returns the report text plus the claude.ai conversation URL.

### chatgpt-research

Same shape, different target:

- *"Use chatgpt-research to investigate WebGPU adoption."*
- *"Kick off a ChatGPT Deep Research query on the state of Rust async runtimes."*
- *"Run the chatgpt-research preflight to confirm my setup."*

The skill navigates chatgpt.com, enables Deep Research from the composer, submits, auto-answers, waits 5–30+ min, returns the report (Markdown when available) plus the conversation URL.

**ChatGPT-specific caveats:**

- Requires a ChatGPT account with Deep Research access (Plus / Pro / Team / Enterprise / Edu — Free gets a very limited lightweight variant). The skill surfaces tier-gate messages verbatim and stops, rather than silently downgrading.
- chatgpt.com is Cloudflare-protected. Attach mode (real-session cookies + fingerprint) is the mitigation, and is usually enough. If a Cloudflare challenge appears, the skill stops and asks you to complete it manually in your Chrome window — it will NOT click through the challenge automatically (doing so is the exact pattern Cloudflare detects, and repeated attempts can escalate to a hard block).

### Clarification questions (both skills)

Both target services' research flows ask 1–3 follow-up questions (scope, time period, depth, etc.) before kicking off the real run. **By default, the skills auto-answer** those — drawing the reply from your original query, deferring to the service's defaults when your query is silent — so the run starts unattended.

Each skill returns a short audit of those auto-replies so you can see what it answered on your behalf.

If you want to be in the loop on the plan, say so in the request:

- *"Research X on claude.ai, but **ask me before answering** any clarifying questions."*
- *"Kick off a ChatGPT Deep Research on Y, **interactive** mode."*

### Kickoff mode (chatgpt-research only)

After clarifications, ChatGPT shows a plan confirmation panel with **Edit / Cancel / Start** buttons. The Start button has its own ~18-second auto-start countdown. The skill picks when to click Start based on the **kickoff mode**:

- **brief-wait** (default): waits ~12 seconds — long enough for you to glance at the plan and intervene with Edit/Cancel if needed, faster than ChatGPT's own countdown.
- **immediate**: kicks off as soon as the panel appears. Use for loops or fully unattended runs.
  - *"Kick off a Deep Research on X **immediately** / **no wait** / **fully automated**."*
- **manual**: doesn't click Start — returns the conversation URL and leaves the plan panel waiting for you. Use when you want to review and start it yourself.
  - *"Show me the plan first" / "wait for me to start" / "manual kickoff"*

If you also asked for interactive clarifications, manual kickoff is the natural default ("supervise everything" combo).

---

## Repo layout

```
research.skill/
├── .claude-plugin/
│   └── marketplace.json              # Multi-plugin marketplace manifest
├── claude-research/                  # Plugin: claude-research
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── claude-research/
│           └── SKILL.md              # The skill itself (portable across harnesses)
├── chatgpt-research/                 # Plugin: chatgpt-research
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── chatgpt-research/
│           └── SKILL.md
├── docs/
│   └── browser-setup.md              # Shared Chrome DevTools MCP setup
├── README.md
└── .gitignore
```

The double-nested `<plugin-name>/skills/<skill-name>/` path is Claude Code's plugin convention. Codex / opencode users symlink directly to the inner `skills/<skill-name>/` directory and ignore the outer plugin wrapper.

---

## License

MIT.
