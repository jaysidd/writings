# A weekend on the official Claude Agent SDK
## What an experienced engineer notices the first weekend out: the engine collapses to a function call, the product opinion is the work, and the model can do the routing.

![Command Center, a multi-agent dashboard built on the official Claude Agent SDK, running locally with four specialist agents.](https://raw.githubusercontent.com/jaysidd/claude-agent-lab/main/docs/screenshots/01-overview.png)
*The Command Center dashboard. Four specialist agents in the sidebar; chat in the main pane; tool-use trace inline. About 2,500 hand-written lines of code stand on top of the SDK to make this.*

Anthropic released the official Claude Agent SDK about a month ago. I read the docs the week it dropped, scanned the example projects, and put it back on the shelf with the intention of building something on it later. This past weekend was later.

Two and a half days of calendar, roughly a day and a half of actual hands-on work, and I had a working multi-agent command center running locally. Specialist agents in a sidebar. A router that delegates to them. A Haiku-classified task queue with auto-routing. Token-by-token streaming. Persistent SQLite memory. Folder scoping. A `@file` autocomplete popover. A `/command` autocomplete popover. Slash commands like `/think hard` and `/export md`. Conversation history with full session restore. Per-message and per-session token tracking. Voice input via a local Whisper gateway. A settings UI for integrations. Twenty-two offline Playwright tests plus two end-to-end against the real SDK, all green.

Roughly 2,500 hand-written lines of code, end to end. One person. No build system. The whole thing runs on `tsx` and `npm run serve`.

I want to share what I noticed about the experience as a practitioner, because the practitioner story underneath this is more interesting than the headline numbers.

## The mental model: SDK as engine, you as the product

The thing that used to be most of an agentic-app project, the engine, is now a parameter on a single function call.

I do not mean that as marketing language. I mean it as the literal experience of reading the source. The SDK exposes one primary surface, `query({ prompt, options })`, that returns an async iterable of message objects. You drive that iterable. The agent loop, the tool-use protocol, the streaming wire format, the JSON-RPC bridge to the underlying Claude Code subprocess, all of it lives behind the call.

> When Claude is the target model, the SDK collapses the engine layer to a function call.

The implication of that is not that agentic apps are now trivial. The implication is that the question for any builder shifts from *"how do I architect the engine?"* to *"what do I actually want to build?"* Once you internalize that, the rest of the experience changes shape.

## What the engine handed me, and what I had to write myself

I think it is worth getting concrete about this, because the compression is easier to argue with when you can see the line item.

Here is the rough split for Command Center, the lab I shipped this weekend.

**What the SDK gave me, each as one or two `query()` options:**

- The agent loop and the tool-use protocol. Built-in. I never see the loop iterations from my code; I just consume the message stream the SDK produces.
- Sub-agent delegation. Set `agents: { comms: {...}, content: {...}, ops: {...} }` and a router agent with the SDK's `Agent` tool in its allowlist can invoke any specialist as a tool. The whole "which specialist should this go to" decision is handled by the model, not by my code.
- Token-by-token streaming. Set `includePartialMessages: true` and the SDK emits `stream_event` messages whose `event.delta.text` carries the incremental text deltas straight from the API.
- Multi-turn sessions per agent. Capture the `session_id` from the first message of the stream, pass it back as `resume:` next turn, and the conversation continues with full context.
- Folder scoping. Set `cwd:` and the agent's filesystem tools are scoped to that directory.
- Per-agent model selection. Each agent picks its own model: Opus 4.7 for deeper work, Sonnet 4.6 for the workhorse turns, Haiku 4.5 for cheap classifier passes.
- Plan mode. Set `permissionMode: 'plan'` and tool calls are simulated instead of executed. Two characters of config; an entire safety net for destructive actions.
- Per-agent tool allowlists. Sub-agent delegation cannot escalate access; each specialist runs with only the tools it was given, even when invoked by a router that has the `Agent` tool.
- Authentication resolution. The SDK reads `ANTHROPIC_API_KEY`, falls back to enterprise transports (Bedrock, Vertex, Foundry), falls back to the local Claude Code CLI's OAuth session. I wrote zero auth code.
- Abort on disconnect. Pass an `AbortController` and listen on `res.on("close")`; if the browser tab closes mid-stream, the SDK iterator stops cleanly.

**What I had to write myself:**

- The four specialist agents and their system prompts. Main as a router with no tools. Comms with `WebFetch` for drafting messages. Content with `WebSearch` and `WebFetch` for long-form work, defaulting to Opus. Ops with `Read`, `Glob`, `Grep`, scoped read-only inside the active folder.
- The vanilla-JS frontend. Sidebar, chat window, header, modals, tool-use trace rendering, streaming bubble updates. About 620 lines, no framework.
- The Express server that wraps the SDK and exposes the `/api/*` routes the frontend talks to. About 300 lines.
- Persistent SQLite storage for memory, settings, custom agents, full session history, and per-message usage objects.
- A task board with a Haiku-powered one-shot classifier that decides which agent each task should go to.
- The `@file` autocomplete popover, the `/command` autocomplete popover, and the slash commands themselves.
- Integration with a local Whisper gateway for voice input, with WebM-to-WAV conversion in the browser to dodge an ffmpeg quirk on the receiving side.
- A settings UI backed by SQLite, with masked secret previews so tokens never round-trip to the browser in plaintext.
- Markdown rendering with sanitization. `marked` plus `DOMPurify` plus `highlight.js`, all via CDN, no build step.
- Twenty-two offline Playwright tests plus two end-to-end tests against the real SDK.
- A double-click `.command` launcher for macOS that auto-locates the project, kills any previous server on port 3333, runs `npm install` if needed, starts the server, and opens the browser.

![The custom-agent editor, with fields for name, emoji, accent color, default model, tool checkboxes, and a system-prompt textarea.](https://raw.githubusercontent.com/jaysidd/claude-agent-lab/main/docs/screenshots/12-new-agent-editor.png)
*The custom-agent editor. Spawning a new specialist is a first-class operation. The SDK has no opinion on this surface; the product opinion does.*

That second list is the real product work. It is what makes Command Center a thing somebody can actually use, instead of a thin wrapper around `query()`. None of it is hard. All of it requires opinion. The reason the build was fast is not that I skipped this work. It is that I was finally able to spend my full attention on it instead of on the agent loop underneath.

## Three patterns that surprised me

A practitioner read on what actually drove the velocity, in order of importance.

**1. The SDK has an opinion. I borrowed it.**

The SDK ships with a mental model: agents have prompts, agents have tool allowlists, agents have models, agents resume via session ids, routers delegate to specialists. I did not have to invent any of that vocabulary. I adopted it directly into my code, my UI labels, and my docs. When a framework's mental model is good, you skip the months of bad-abstraction back-and-forth that mediocre frameworks force on you. I think that, more than the code reuse, is the actual time savings.

**2. The model does the routing.**

![Task board with the create-task form open and the Haiku classifier ready to assign the task to a specialist agent.](https://raw.githubusercontent.com/jaysidd/claude-agent-lab/main/docs/screenshots/04-task-board-form.png)
*The task board. Free-form description in, classified agent assignment out. The classifier is twelve lines of TypeScript wrapping a one-shot Haiku call.*

The earlier agent frameworks I had looked at all included explicit routing logic. *If task description contains X, send it to agent Y.* My version does not have a single line of that.

The Main agent has the SDK's `Agent` tool in its allowlist and an `agents:` map populated with the specialist definitions. The model decides which specialist to invoke. There is zero routing logic in my server. The same logic applies to the task queue: a one-shot Haiku query, classifying free-form task descriptions into one of four agent ids, runs in about a second and costs a fraction of a cent. That is a classifier I would have spent a week tuning two years ago. Now it is twelve lines of TypeScript.

The pattern I keep coming back to: when the model can make the decision well, do not write the decision in code. Pass the decision to the model. The SDK is built around that assumption, and it shows in how thin the surrounding code can be when you take the assumption seriously.

**3. Small commits compounded.**

I did not try to design the whole system on a whiteboard before opening the editor. I shipped the four-agent sidebar first. Got it working. Committed. Then sub-agent delegation. Committed. Then streaming. Committed. Then the task board. Committed. By Sunday evening I had thirty small commits and a working app that I could grow without breaking. The compounding from this kind of cadence is real, and it is invisible from the outside until somebody looks at the git log.

## What I think this says about new projects

A few practitioner observations from the weekend that I think generalize, with the caveat that one weekend is not a sample size.

The shape of an agentic app is settling. Specialist agents with system prompts, tool allowlists, sessions, sub-agent delegation, streaming, plan mode. Frameworks across the ecosystem are converging on this vocabulary. Once a vocabulary settles, the layer above it gets cheaper, and the differentiation moves up to product opinion.

The thing that is no longer interesting is the agent loop. I do not think anyone should be writing one from scratch in 2026 unless they have a specific reason. Use the SDK that ships with the model you target.

The thing that is still interesting, and where I think the next year of practitioner work lives, is the product layer. Which agents you define. Which prompts they get. Which tools you trust them with. How conversations are persisted, exported, recovered. How the UI surfaces the model's reasoning. How the system handles failure. How it integrates with the rest of an organization. None of that is solved by the SDK, and none of it should be. That is product work, and product work is opinionated.

There is also the question of where this leaves multi-provider builds. The Claude Agent SDK is, by design, Claude only. If you need OpenAI, Gemini, Ollama, or local models behind the same UI, you are looking at a different problem with a different abstraction layer underneath. I think Claude-only and multi-provider end up being two different products with two different audiences, and the worst design choice is trying to be both at once. The lab I shipped this weekend stays cleanly Claude-only.

## What I keep thinking about

The thing that struck me most about the weekend was how unremarkable the experience felt while it was happening. There was no breakthrough moment. The SDK worked the way the docs said it would. The hard part was not the technology. It was deciding what I wanted to build, and being willing to actually build it on a Friday night instead of saving it for "a future quarter when I will have time."

I think the next interesting period for agentic-app practitioners is going to be quieter than the headlines suggest. Frameworks like the Claude Agent SDK make it possible for a single experienced engineer to ship a working agent system in a weekend. The visible tooling improvements are not the story; the story is the kind of project a small team or solo builder can now reasonably attempt. That class of project did not exist on the same timeline a year ago.

The lab is at [github.com/jaysidd/claude-agent-lab](https://github.com/jaysidd/claude-agent-lab). MIT licensed. Educational reference, not a product, with the authentication caveat documented in the README. If you fork it and ship something interesting, send me the link. The thing I am still chewing on is what the next compression will be, and what kind of project it will make possible that I am not currently imagining.

## Key takeaways

- The Claude Agent SDK collapses the agent loop, sub-agent routing, streaming, sessions, plan mode, and folder scoping into options on a single function call. The "engine layer" of an agentic app is no longer a project, it is a parameter.
- What is left to build is the product opinion: which agents, which prompts, which tools, which UI, which integrations. The SDK does not have a view on any of it.
- The best routing logic is no routing logic. When the model can make the decision well, pass the decision to the model.
- The shape of an agentic app is settling. The differentiation is moving up to product opinion, where it should be.

---

## Sources

1. [Claude Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview), Anthropic, 2026.
2. [Command Center, the multi-agent lab referenced in this article](https://github.com/jaysidd/claude-agent-lab), public GitHub repository, MIT licensed.

---

## About the Author

Junaid Siddiqi explores the intersection of enterprise AI adoption, system architecture, and practical engineering leadership. Connect on LinkedIn for more analysis of emerging AI development patterns.

[Connect with me on LinkedIn](https://www.linkedin.com/in/junaidsiddiqi/)
