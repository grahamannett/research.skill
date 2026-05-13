---
name: chatgpt-research
description: Run a Deep Research query against chatgpt.com using the user's existing logged-in browser session via Chrome DevTools MCP. Navigates chatgpt.com, enables Deep Research from the composer, submits the query, auto-answers any clarification prompts, waits for completion, and returns the report text plus the conversation URL. Use when the user asks to run a ChatGPT Deep Research query, fetch a deep-research report from ChatGPT, resume a prior Deep Research conversation, or preflight the MCP/Chrome/ChatGPT-login setup. Requires a ChatGPT account with Deep Research access (Plus/Pro/Team/Enterprise/Edu; Free gets a lightweight variant with very limited quota).
---

# chatgpt-research

Drives the **Deep Research** feature on chatgpt.com using the user's already-authenticated Chrome session via Chrome DevTools MCP (Playwright MCP as fallback), then returns the produced report.

**Status:** best-guess procedure — modeled on the sister `claude-research` skill but not yet verified end-to-end against chatgpt.com's live UI. Expect to discover and adjust some specifics on the first real run.

## When to use

Invoke when the user wants to:
- Run a Deep Research query on chatgpt.com and bring the result back into the current Claude Code / Codex CLI / opencode session.
- Kick off a long-running ChatGPT investigation while continuing other work locally.
- Re-open a previous ChatGPT Deep Research conversation to extract its output.
- Just verify the setup (MCP attach + login) without submitting a query — treat that as the **preflight-only** path (steps 1–5, then report).

Do NOT use for ordinary (non-Deep-Research) chats, scraping non-chatgpt.com sites, or one-off questions the local model can answer directly.

## Tool fallback order

1. **Primary:** Chrome DevTools MCP — tools named `mcp__chrome-devtools__*`.
2. **Fallback:** Playwright MCP — tools named `mcp__playwright__*`, configured to connect over CDP to the same `localhost:9222`.

Pick one MCP at the start and stay with it. Don't mix tool families mid-flow.

## Clarification handling — default is auto-answer

ChatGPT's Deep Research flow has a built-in clarification step: an intermediate model asks 1–N follow-up questions (scope, timeframe, source preferences, depth) before kicking off the actual research run. The whole point of this skill is to fire-and-forget a research run from your agent harness, so **the default is to auto-answer those clarifications** and let the run start unattended.

**Auto-answer rules:**
- Read what ChatGPT is asking.
- Reply with the most natural answer the **original user query** implies. Use the original query's words and intent — don't introduce scope, constraints, or preferences the user didn't ask for.
- When the original query is silent on a point, **defer to ChatGPT's judgment**. Short answers like "No preference — use your best judgment", "Whatever you think is most useful", or "Up to you, just proceed" are fine.
- If ChatGPT shows a "Start research" / "Skip clarifications" / "Use defaults" / "Looks good" button (or equivalent), prefer clicking that over composing a reply.
- Loop: if ChatGPT asks again after your reply, answer again. Hard cap at **3 clarification rounds** — if it's still asking after 3 replies, something is off; bail to the user with what you saw.

**Bail out of auto-answer and surface the question to the user** when:
- The user invoked the skill with explicit "ask me", "interactive", "wait for my input", or "check with me on the plan" framing.
- The clarification asks for a fact that's genuinely not derivable from the original query AND would materially change the research direction (e.g., "Which of these three companies should I focus on?" when the user didn't name any). Picking arbitrarily here would silently change what the user gets back.

When in doubt, lean toward auto-answering with "use your best judgment" — ChatGPT's defaults are usually fine, and the user can always re-run with a more specific query.

## Preflight (always run these first)

### 1. Detect a browser MCP

Look at the registered tools in this session:
- `mcp__chrome-devtools__*` present → use Chrome DevTools MCP.
- Otherwise `mcp__playwright__*` present → use Playwright MCP.
- Neither → stop. Tell the user to install one (point at the repo's `docs/browser-setup.md`).

State which MCP you picked before doing anything else.

### 2. Confirm the MCP is in *attach* mode, not *launch* mode

`chrome-devtools-mcp` has two modes:
- **Attach mode** — registered with `--autoConnect` (Chrome 144+) or `--browser-url=http://127.0.0.1:9222`. Drives the user's real, logged-in Chrome.
- **Launch mode** (default) — spins up its own Chrome with a throwaway profile at `~/.cache/chrome-devtools-mcp/chrome-profile-stable`. No logins. **Wrong for this skill, and also more likely to be blocked by Cloudflare on chatgpt.com.**

Sanity-check which mode is active before navigating to chatgpt.com. The simplest signal: navigate to `chrome://version` and read `Profile Path` from the accessibility snapshot. If it looks like the throwaway cache dir, stop and tell the user:
- The MCP is in launch mode.
- Point them at `docs/browser-setup.md` (Verified setup / `--autoConnect`, or Option B / `--browser-url`).
- Suggest `claude mcp list` (or the equivalent for their harness) to confirm current flags.

Do NOT log in inside the throwaway profile — that defeats the point AND increases bot-detection risk.

### 3. Navigate to chatgpt.com

Open `https://chatgpt.com` in the attached browser. Be redirect-aware: ChatGPT may land you at `https://chatgpt.com/?model=...` or redirect from `chat.openai.com` to `chatgpt.com`. That's fine — proceed once a stable URL is reached.

### 4. Verify login (and detect Cloudflare)

Read the page via the MCP's accessibility-tree / snapshot tool. Decide based on what's visible:

- **Logged in** signals: a sidebar with chat history, the composer (message input with placeholder like `Ask anything` or `Message ChatGPT…`), an account avatar / initials in the corner, a model picker near the composer.
- **Not logged in** signals: a "Log in" / "Sign up" button on the landing page, an OpenAI marketing layout, an email/password form.
- **Cloudflare challenge** signals: a page titled "Just a moment…", a Turnstile checkbox ("Verify you are human"), a Cloudflare-branded interstitial. **If you see this, STOP immediately** — see the Cloudflare failure mode below.

If signals are ambiguous (interstitial loading, navigation pending), wait a few seconds and re-snapshot once before deciding. If not logged in, stop and ask the user to log in in their Chrome window — do not attempt to log in yourself.

### 5. (Preflight-only path)

If the user only asked for a preflight, report and stop:
- Which MCP, which mode, which profile path you observed.
- Current URL.
- **Logged in** / **not logged in** / **Cloudflare-challenged** verdict + the signal you matched.

Otherwise continue to step 6.

## Running a Deep Research query

### 6. Open the composer's `+` menu

In the chatgpt.com composer there is a small `+` button at the left of the message input (placeholder reads "Ask anything"). Click it. A popover menu opens.

**Observed menu (as of 2026-05) — these are accessible-name labels, not selectors. Match by role + accessible name; do NOT hard-code DOM positions:**

```
Add photos & files
Recent files        >
---
Create image
Deep research       ← the entry we want
Web search
··· More            >
---
Projects            >
```

Find the menu item whose accessible name **contains "deep research"** (case-insensitive — the UI label is "Deep research" with a lowercase 'r', but match permissively in case OpenAI renames or capitalizes differently).

If the `+` button is missing, the composer might be in a state this skill doesn't know about (a tool-output panel obscuring the input, mobile layout, an in-progress run). Try clicking "New chat" first to get a fresh composer, then re-check.

If you can't find "Deep research" in the menu after it opens, look under "··· More" — it's possible OpenAI moves modes between the top level and the More submenu. Expand More once and re-scan.

If Deep research is still missing, stop and report what menu entries you *did* see. The user's account tier may not include Deep Research, or OpenAI may have moved it elsewhere entirely.

### 7. Enable Deep Research and verify it stuck

Click the "Deep research" entry. The menu typically closes on selection.

ChatGPT's active-mode indicator is **different from claude.ai's in-menu checkmark**. After the menu closes, look for any of these signals on or near the composer:
- A "Deep research" pill / badge / chip rendered inside or next to the message input.
- The model picker text changes (e.g. to a Deep-Research-specific model label).
- An icon or color treatment on the input border indicating a non-default mode.
- Re-opening the `+` menu shows "Deep research" with a checkmark, highlight, or "active" annotation.

Re-snapshot the composer area and confirm at least one of those signals is present. If none are, the click may not have landed — try once more, then bail if it still doesn't take.

If clicking surfaces a tier-gate dialog or inline message instead (e.g. "Deep Research is available on Plus, Pro, Team, Enterprise, and Edu", "You've reached your monthly Deep Research limit", "Upgrade to unlock Deep Research"), **surface that message verbatim and stop**. Don't fall back to a regular chat or to the lightweight Free variant unless the user explicitly asked for it.

If toggling Deep Research alters other composer state (model picker locks, "Web search" / other tools disable), note it in the final report but don't fight it.

### 8. Compose and submit

- Click into the message input.
- Type the user's query **verbatim**. Do not paraphrase, expand, or "improve" it.
- Submit (Enter, or the send button — whichever is exposed by the accessibility tree).

ChatGPT's Deep Research flow will almost always show one or more clarification prompts before kicking off the actual run. Handle them per the **Clarification handling** section above:
- Default: auto-answer (deriving the reply from the original user query, deferring to ChatGPT's defaults when the original is silent).
- Override (caller said "ask me", "interactive", etc.) or genuinely-ambiguous direction-changing question: stop and surface to the user.
- Loop until ChatGPT stops asking clarifications and the actual research starts streaming. Cap at 3 rounds.

Note in the final report (step 11) how many clarification rounds happened and a one-line summary of what you answered, so the user can audit your auto-replies.

### 9. Wait for completion

Deep Research runs take **5–30+ minutes** (sometimes longer). While waiting:
- Poll every 30–60 seconds via accessibility snapshots. Suitable completion signals: streaming indicator disappears, a "Research complete" / "Finished" marker appears, the report renders fully with citations, an "Export to PDF" / "Export to DOCX" / "Copy markdown" affordance becomes available.
- Do not navigate away, close the tab, or trigger other actions in that tab.
- If the user asked the skill to run in the background: return immediately with the conversation URL and tell the caller to re-invoke the skill with that URL once they want to pick up the result.
- If a Cloudflare challenge appears mid-run (rare but possible on long sessions), see the Cloudflare failure mode below.

### 10. Extract the result

Once complete:
- If ChatGPT exposes a "Copy as Markdown" or similar export affordance, prefer that — the formatted Markdown is the cleanest payload.
- Otherwise, capture the rendered report from the DOM / accessibility tree as **text**. Never substitute a screenshot — the user wants the report content, not a picture.
- Expand any collapsible sections (citations, sources, plan steps) before reading, so the full body is captured. ChatGPT's Deep Research output typically includes inline citations and a sources list — both should be in the captured text.
- Capture the conversation's chatgpt.com URL from the address bar.

### 11. Return

To the caller:
- Full report text (Markdown preferred).
- chatgpt.com conversation URL.
- One-line summary of elapsed time and any non-obvious events (e.g. "Deep Research enabled; model locked to gpt-5-deep-research. Took 18 minutes.").
- **Clarification audit** — N rounds, plus the question(s) ChatGPT asked and the reply you gave. Keep this brief (one line per round) so the user can spot if you auto-answered something they'd have answered differently.

## Resuming an existing Deep Research conversation

If the user provides a chatgpt.com conversation URL instead of a fresh query:
- Navigate directly to that URL (skip steps 6–8).
- If the query is still running, jump to step 9 (wait).
- If already complete, jump to step 10 (extract).

## Failure modes — surface, don't paper over

- **MCP in launch mode** → stop at step 2; point at `docs/browser-setup.md` (Verified setup).
- **Cloudflare / Turnstile challenge** → stop immediately. Tell the user: chatgpt.com is showing a Cloudflare bot-detection challenge; they need to complete it manually in their Chrome window (click the checkbox, satisfy any image puzzle), then re-invoke the skill. **Do not attempt to click through the challenge automatically** — that's exactly the behavior Cloudflare detects, and repeated attempts can escalate to a hard block on the IP. Attaching to the user's real logged-in Chrome (per `docs/browser-setup.md`) is the mitigation; if challenges keep recurring, the MCP may not be in attach mode after all, or the session may have aged out.
- **Not logged in** → stop at step 4; ask the user to log in in their Chrome window.
- **Tier gate** (account lacks Deep Research / quota exhausted) → stop; surface ChatGPT's exact message verbatim.
- **`+` / Tools menu not found** → stop at step 6; report what controls you *did* see. The composer may be in a state the skill doesn't know about, or the user's account tier may be hiding the menu.
- **Deep Research entry not in the menu** → stop at step 6; report what entries you saw. Likely a tier-gate or UI variation.
- **Query rejected** (rate limit, content policy, "I can't help with that") → surface ChatGPT's exact message verbatim. Don't retry silently.

## Cross-harness notes

This skill's `SKILL.md` is portable across Claude Code, OpenAI Codex CLI, and opencode — they all read the same frontmatter + body. The only difference is how the user installs the skill on each (see the repo's top-level `README.md`). The procedure above is harness-agnostic: it references MCP tool names by prefix (`mcp__chrome-devtools__*`, `mcp__playwright__*`) which each harness exposes identically.

## What this skill is NOT for

- Ordinary (non-Deep-Research) ChatGPT chats — use the OpenAI API or a different skill.
- Bypassing chatgpt.com auth, rate limits, content policies, or Cloudflare challenges.
- Reading other users' conversations.
- Using ChatGPT Agents / Operator / Tasks — those are separate features with different UIs.
