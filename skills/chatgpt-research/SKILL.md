---
name: chatgpt-research
description: Run, preflight, or resume a ChatGPT Deep Research task using the user's existing logged-in browser session. Use when the user asks Codex to use ChatGPT Deep Research, start a research report on chatgpt.com, check whether the browser/login/deep research setup works, resume an existing ChatGPT research URL, or collect the finished report/export result.
---

# chatgpt-research

Drives **Deep Research** in ChatGPT using the user's already-authenticated browser session. This is for using ChatGPT's product UI with the user's own account and plan, not for replacing the OpenAI API or bypassing usage limits.

## When to use

Invoke when the user wants to:
- Start a ChatGPT Deep Research task from Codex.
- Resume a previous ChatGPT Deep Research conversation or report URL.
- Preflight the browser/login/Deep Research setup without submitting a task.
- Retrieve the final ChatGPT URL and visible export/download result for a completed research task.

Do not use for normal ChatGPT chats, API-backed research, bulk research queues, scraping unrelated ChatGPT conversations, login automation, quota workarounds, or anything intended to bypass ChatGPT product limits.

## Tool fallback order

1. Chrome DevTools MCP tools, usually named `mcp__chrome-devtools__*`.
2. Playwright MCP tools, usually named `mcp__playwright__*`, configured to connect over CDP.
3. Computer Use or Browser plugin only if the available tools are explicitly connected to the user's logged-in browser profile.

Pick one browser-control tool family at the start and stay with it. Do not mix tool families mid-flow unless the first one fails before any research task is submitted.

## Safety and product boundaries

- Use the user's existing authenticated ChatGPT session. Do not log in for them or handle credentials.
- Respect ChatGPT plan limits, usage counters, rate limits, country/workspace availability, and policy messages.
- Do not use hidden endpoints, local storage scraping, or internal network calls.
- Prefer ChatGPT's visible report view and built-in export/download controls for completed reports. Return the conversation/report URL and the export result. If a full text capture is explicitly requested, use only visible page content or visible copy/export affordances and say how it was obtained.
- Do not run batches or silently start multiple research tasks.

## Clarification and plan handling

ChatGPT Deep Research may ask clarifying questions and/or show a proposed research plan before the task begins.

Default behavior:
- Read the question or plan.
- If ChatGPT offers a **Start research**, **Begin**, **Continue**, **Use this plan**, or similar button and the plan matches the user's original request, proceed.
- If it asks a clarification whose answer is obvious from the user's original request, answer using the user's words and intent.
- If the user request is silent, prefer short deference such as "Use your best judgment" only for low-impact details.

Stop and ask the user when:
- They asked for interactive mode, plan review, or "ask me first".
- The plan narrows or changes the task in a way the user did not ask for.
- ChatGPT asks for a choice that would materially change the research direction.

Cap auto-answered clarification rounds at 3. If ChatGPT is still asking after that, stop and summarize what happened.

## Preflight

Always run these checks before submitting a task.

### 1. Detect browser control

Look for available browser tools in the tool list:
- Chrome DevTools MCP present -> use it.
- Else Playwright MCP present -> use it.
- Else stop and tell the user the skill needs a browser-control MCP attached to a logged-in Chrome session.

State which tool family you picked.

### 2. Confirm attach mode and profile

The browser controller must drive the user's real logged-in Chrome profile or a persistent dedicated profile. It must not drive a disposable automation profile with no login.

Navigate to `chrome://version` when the tool allows it and read the profile path:
- Good signals: the user's normal Chrome profile or a named persistent research profile.
- Bad signals: throwaway/cache profiles such as `~/.cache/chrome-devtools-mcp/chrome-profile-stable`, temporary Playwright profiles, or newly launched empty profiles.

If the profile is wrong, stop. Tell the user to configure Chrome DevTools MCP or Playwright MCP to attach to their real/persistent Chrome profile before using this skill.

### 3. Navigate to ChatGPT

Open `https://chatgpt.com`.

### 4. Verify login

Use the accessibility tree or visible page state.

Logged-in signals:
- ChatGPT sidebar or conversation history.
- A prompt composer.
- User/account controls.
- Model/tool controls or a new chat screen.

Not-logged-in signals:
- "Log in" / "Sign up" buttons.
- Auth forms.
- Marketing landing page without the ChatGPT app shell.

If ambiguous, wait briefly and re-check once. If still not logged in, stop and ask the user to log in in the attached browser.

### 5. Verify Deep Research availability

Check for at least one official Deep Research entry point:
- Typing `/Deepresearch` in the composer.
- A **Deep research** item in the `+` menu beside the composer placeholder (`Ask anything`). In the current ChatGPT UI this menu commonly includes items such as **Add photos & files**, **Recent files**, **Create image**, **Deep research**, **Web search**, **More**, and **Projects**.
- A **Deep research** option from the sidebar.

If Deep Research is unavailable, stop and report exactly what controls or product messages were visible. Availability depends on plan, workspace configuration, and location.

### 6. Preflight-only path

If the user only asked for preflight, report:
- Browser tool family used.
- Profile path or profile verdict.
- Current ChatGPT URL.
- Login verdict.
- Deep Research availability verdict.

Then stop without submitting anything.

## Starting a Deep Research task

1. Start from a fresh ChatGPT chat unless the user asked to use an existing one.
2. Enable Deep Research using the visible `+` menu's **Deep research** item when available. Fall back to `/Deepresearch` or the sidebar entry if the menu shape differs.
3. Configure sources only when the user requested them or ChatGPT asks:
   - Default web/file behavior is acceptable when the user gave no source constraints.
   - Use specific sites only when the user named sites/domains or asked for that restriction.
   - Use connected apps only when the user explicitly requests them and they are already enabled.
4. Enter the user's research prompt verbatim. Do not silently expand, narrow, or editorialize it.
5. Handle clarifications and proposed plans using the rules above.
6. Once research starts, capture the conversation/report URL.

## Waiting and resuming

Deep Research can take several minutes or longer. While waiting:
- Poll every 30-60 seconds using visible page state.
- Do not navigate away or close the research tab.
- If the user asked for background mode, return the URL immediately and tell them to re-invoke `chatgpt-research` with that URL later.

Completion signals include:
- A completed report view.
- A table of contents, sources section, or activity history.
- Export/download controls for Markdown, Word, or PDF.
- No active progress indicator.

If the user provides an existing ChatGPT URL:
- Navigate directly to it.
- If still running, wait or return the status depending on the user's request.
- If complete, collect the URL and export/result information.

## Returning the result

For a completed task, return:
- The ChatGPT conversation or report URL.
- Whether Deep Research completed or is still running.
- Which export/download options are visible or which export was produced.
- A concise note about elapsed time and any notable events.
- Clarification audit: number of rounds and a short summary of questions/answers or plan approval.

If the user explicitly requested report text, include it only when it can be captured from visible report content or ChatGPT's visible copy/export controls. Otherwise return the URL/export path and explain that the report should be opened or exported from ChatGPT.

## Failure modes

Surface failures plainly:
- Browser MCP missing or attached to the wrong profile.
- Not logged in.
- Deep Research unavailable for the account/workspace/location.
- Usage limit or rate-limit message.
- Query rejected by ChatGPT.
- Export/download unavailable.
- UI changed and expected controls cannot be found.

Do not retry silently, start a second research task, or switch accounts.
