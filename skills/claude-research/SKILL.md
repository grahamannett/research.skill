---
name: claude-research
description: Run a research or deep-research query against claude.ai using the user's existing logged-in browser session via Chrome DevTools MCP. Navigates claude.ai, enables Research mode from the composer's "+" menu, submits the query, waits for completion, and returns the report text plus the conversation URL. Use when the user asks to run a claude.ai Research query, fetch a deep-research report, resume a prior Research conversation, or just preflight the MCP/Chrome/login setup.
---

# claude-research

Drives the **Research** feature on claude.ai using the user's already-authenticated Chrome session via Chrome DevTools MCP (Playwright MCP as fallback), then returns the produced report.

## When to use

Invoke when the user wants to:
- Run a Research / deep-research query on claude.ai and bring the result back into the current Claude Code session.
- Kick off a long-running claude.ai investigation while continuing other work locally.
- Re-open a previous claude.ai Research conversation to extract its output.
- Just verify the setup (MCP attach + login) without submitting a query — treat that as the **preflight-only** path (steps 1–5, then report).

Do NOT use for ordinary (non-Research) chats, scraping non-claude.ai sites, or one-off questions the local model can answer directly.

## Tool fallback order

1. **Primary:** Chrome DevTools MCP — tools named `mcp__chrome-devtools__*`.
2. **Fallback:** Playwright MCP — tools named `mcp__playwright__*`, configured to connect over CDP.

Pick one MCP at the start and stay with it. Don't mix tool families mid-flow.

## Preflight (always run these first)

### 1. Detect a browser MCP

Look at the registered tools in this session:
- `mcp__chrome-devtools__*` present → use Chrome DevTools MCP.
- Otherwise `mcp__playwright__*` present → use Playwright MCP.
- Neither → stop. Tell the user to install one (point at `README.md`).

State which MCP you picked before doing anything else.

### 2. Confirm the MCP is in *attach* mode, not *launch* mode

`chrome-devtools-mcp` has two modes:
- **Attach mode** — registered with `--autoConnect` (Chrome 144+) or `--browser-url=http://127.0.0.1:9222`. Drives the user's real, logged-in Chrome.
- **Launch mode** (default) — spins up its own Chrome with a throwaway profile at `~/.cache/chrome-devtools-mcp/chrome-profile-stable`. No logins. **Wrong for this skill.**

Sanity-check which mode is active before navigating to claude.ai. The simplest signal: navigate to `chrome://version` and read `Profile Path` from the accessibility snapshot. If it looks like the throwaway cache dir, stop and tell the user:
- The MCP is in launch mode.
- Point them at the README's "Verified setup" section (`--autoConnect`) or "Option B — `--browser-url`".
- Suggest `claude mcp list` to confirm current flags.

Do NOT log in inside the throwaway profile — that defeats the point.

### 3. Navigate to claude.ai

Open `https://claude.ai` in the attached browser.

### 4. Verify login

Read the page via the MCP's accessibility-tree / snapshot tool:
- **Logged in** signals: sidebar with chat history, the composer (placeholder `Type / for skills` or similar), a "+" button next to the composer, the user's avatar.
- **Not logged in** signals: a "Log in" / "Sign up" button, marketing homepage layout, an email/password form.

If signals are ambiguous (interstitial loading), wait a few seconds and re-snapshot once before deciding. If not logged in, stop and ask the user to log in in their Chrome window — do not attempt to log in yourself.

### 5. (Preflight-only path)

If the user only asked for a preflight, report and stop:
- Which MCP, which mode, which profile path you observed.
- Current URL.
- **Logged in** / **not logged in** verdict + the signal you matched.

Otherwise continue to step 6.

## Running a Research query

### 6. Open the composer's `+` menu

In the claude.ai composer there is a small `+` button below or beside the message input. Click it. A menu opens with entries that include:

```
Add files or photos
Take a screenshot
Add to project        >
Add from GitHub
---
Skills                >
Add connectors
---
Research
Web search            ✓   (checkmark when active)
Use style             >
```

These names are observed UI labels, not selectors. Find them in the accessibility tree by **role + accessible name**, not by CSS classes or DOM positions — claude.ai's markup changes.

If the `+` button is missing, the composer might be in a different state (a chat already open, mobile layout, etc.). Try clicking "New chat" first to get a fresh composer, then re-check.

### 7. Enable Research mode

Click the **Research** entry in the menu. After clicking:
- Re-snapshot the menu (or the composer if the menu closed) and confirm Research is now toggled on. The checkmark / highlighted state appears on the active item, mirroring how "Web search ✓" indicates an active toggle.
- If toggling Research disabled "Web search" (they may be mutually exclusive in claude.ai's UI), that's expected — note it in the final report but don't try to re-enable Web search.
- If Research doesn't appear in the menu at all, the user's account may not have Research access. Stop and report what entries you *did* see.

Close the menu (Escape or click outside) before composing.

### 8. Compose and submit

- Click into the message input.
- Type the user's query **verbatim**. Do not paraphrase, expand, or "improve" it.
- Submit (Enter, or the send button — whichever is exposed by the accessibility tree).

claude.ai's Research mode may show an intermediate "Plan your research" / clarification step before kicking off the actual search. If that happens:
- Read what claude.ai is asking.
- If the user gave a specific query without clarifying details, accept claude.ai's default plan (usually a "Start research" / confirm button).
- If the clarification needs the user's input, stop and surface the question to the user rather than guessing.

### 9. Wait for completion

Research queries take **5–30+ minutes**. While waiting:
- Poll every 30–60 seconds via accessibility snapshots. Suitable completion signals: streaming indicator disappears, a "Research complete" / "Finished" marker appears, the report renders fully, "Stop" button becomes "Regenerate".
- Do not navigate away, close the tab, or trigger other actions in that tab.
- If the user asked the skill to run in the background: return immediately with the conversation URL and tell the caller to re-invoke the skill with that URL once they want to pick up the result.

### 10. Extract the result

Once complete:
- Capture the rendered report from the DOM / accessibility tree as **text** (Markdown if claude.ai exposes it; otherwise plain text). Never substitute a screenshot — the user wants the report content, not a picture.
- Expand any collapsible sections (citations, plan steps, source lists) before reading, so the full body is captured.
- Capture the conversation's claude.ai URL from the address bar.

### 11. Return

To the caller:
- Full report text.
- claude.ai conversation URL.
- One-line summary of elapsed time and any non-obvious events (e.g. "Research mode was enabled; Web search was deactivated automatically. Took 12 minutes.").

## Resuming an existing Research conversation

If the user provides a claude.ai conversation URL instead of a fresh query:
- Navigate directly to that URL (skip steps 6–8).
- If the query is still running, jump to step 9 (wait).
- If already complete, jump to step 10 (extract).

## Failure modes — surface, don't paper over

- **MCP in launch mode** → stop at step 2; point at README's Verified setup section.
- **Not logged in** → stop at step 4; ask the user to log in.
- **`+` button or Research entry not found** → stop at step 6/7; report what menu entries you *did* see.
- **Account lacks Research access** → stop; surface claude.ai's exact message.
- **Chrome not reachable on 9222** (Option B users only) → stop; point at README's launch command.
- **Query rejected** (rate limit, content policy) → surface claude.ai's exact message. Don't retry silently.

## What this skill is NOT for

- Ordinary (non-Research) chats — use the API or a different skill.
- Bypassing claude.ai auth, rate limits, or content policies.
- Reading other users' conversations.
- Disabling or re-enabling other composer toggles (Web search, Style, etc.) unless the user explicitly asks.
