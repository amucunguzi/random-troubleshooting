# Sentry MCP Subagent Integration UX Regression

Main reference: [https://cra.mr/subagents-with-mcp](https://cra.mr/subagents-with-mcp)

## Tldr
MCP currently tightly couples "what the server can do" with "which tools are available." When you reduce the number of the tools without ensuring the model can still easily see and is fully aware of all the capabilities provided by the server, you lose organicness; *the model no longer has enough clarity about what exactly it can ask and as a consequence chooses not to call tools unless explicitly prompted*.

IMO, this limitation should probably be addressed at the protocol layer. Capabilities discovery for an MCP server should be supported as a first-class primitive, not coupled with `tools/list`.

## Summary by Deepseek V4 Flash Max 
NOTE: I read the analysis, and it makes sense.

--

**ME**: 

Read https://cra.mr/subagents-with-mcp

Can an argument be made that the coupling of "what can this server do" and "which tools are available", where reducing tools count affects visibility of "what can this server do", is the root cause of the regression seen, in particular UX, model not using the mcp organically? Feel free to read more relevant blog posts from https://cra.mr/ for context.

Be thorough. 

---

**AGENT**:

After reading the full series [see references], the argument is compelling — and I'd go a step further: it's not just the root cause of the regression, it's a **structural blind spot in MCP's architecture**.

Here's the full analysis:

### The coupling, traced through the experiments

The Sentry team ran through the entire design space, and every attempt hits the same wall:

| Approach | Context saved | UX impact | Root cause |
|---|---|---|---|
| **Agent mode** (`use_sentry`) | 95% ↓ | 110% slower steering, less organic, URL handling broke | Hiding tools hid *awareness* of capabilities. Model stopped knowing to invoke Sentry unprompted. |
| **Overloaded tools** (`get_sentry_resource`) | Moderate ↓ | "Only solved a fraction of our problems" — broke on relations, listing, search | Squashing N tools into 1 loses the semantic granularity the model uses to reason about *which* capability to invoke. |
| **search_tools + execute_tool** (current GA) | Large ↓ | "Agents are naturally making the right tool calls" — but this is just pushing the problem down a layer | The model still needs to know to call `search_tools` first, and the search itself is a brittle string-matching layer. |

Every regression traces back to one thing: **the protocol provides no mechanism to communicate "what this server can do" separately from "which atomic tools implement it."**


### The structural problem in MCP

MCP's primitive model conflates three distinct concepts into one (`tools/list`):

1. **Intent / domain** — "this server helps debug Sentry errors"
2. **Availability** — "tools exist that can do this" 
3. **Mechanics** — "here is the exact JSON schema to invoke them"

There is no way to signal #1 or #2 without paying the full cost of #3. The `initialize` handshake only says "I have tools" like a boolean flag. The `instructions` field is a single opaque string. There is no structured capability ontology, no skill/category hierarchy, no way to say "this server supports Issue Triage, Project Management, and Performance Analysis" as first-class signals.

Cramer makes this explicit in *Context Management and MCP*:

> **Progressive disclosure in any form hides context from the agent, which means things like description-based steering disappear. It means you've dumbed down tool calls to being no different than a shitty wrapper on your API. MCP using progressive disclosure is worse than a CLI.**

He's saying: the only reason tools are valuable is the rich steering metadata embedded in them. The moment you reduce tool count (for context economy), you lose that steering. But if you keep all tools, you drown in context. The architecture gives you no escape from this dilemma.

### Why the regression is specifically about *organic* usage

The phrase "model not using MCP organically" is key. In the subagent experiment, the model still *could* use Sentry — you could tell it to and it would work. But it stopped doing so unprompted. Why?

The model's awareness of a server's capabilities is entirely mediated by the tools in its visible tool list. That list functions as a kind of *working memory* of what's available. When you collapse N tools into 1 (`use_sentry`), the model loses:

- **Semantic surface area**: 14 distinct tool descriptions → 1 description. Less surface to match against user intent.
- **Contextual triggering**: A user says "why is this error happening" → model sees `get_issue_details`, `search_events`, `get_stacktrace` → pattern matches naturally. With `use_sentry`, it has to *remember* to delegate, which is a much weaker cognitive signal.
- **Confidence in cost/benefit**: The model can assess "this specific tool handles exactly this case." With a subagent, it has to trust a black box — higher perceived risk, less likely to invoke.

This is fundamentally because **tools are the protocol's only mechanism for capability advertising**, and advertising is what drives organic discovery.

### The deeper architectural issue

MCP's design assumes tools are the right unit of both *capability* and *discovery*. But they're not — they're the unit of *execution*. The missing primitive is a **"skill" or "capability" layer** that sits above tools and communicates server intent without requiring full tool schema disclosure.

Cramer keeps circling this:

From *Rethinking Tools*:
> "We are trying to utilize this protocol as a way to give customers access to a set of tools to interact with our services. We are *not* trying to give them a bunch of new API endpoints."

From *Context Management and MCP*:
> "I want Skills, but with fewer consequences... An entire, functional, contained system prompt that would let us ship a subagent to your local interpreter."

From *MCP, Skills, and Agents*:
> "IMO sub-agents are the ignored opportunity... imagine you could take something like SKILL.md and craft it accordingly."

The Server Card SEP (still in draft) is the closest thing to addressing this, but it's focused on pre-connection discovery, not on signaling capabilities within an active session. Even then, it's just metadata — not a structural decoupling.

### Conclusion

**Yes, the coupling is the root cause.** The protocol makes `tools/list` the single source of truth for both "does this server have capabilities relevant to my task?" and "how do I invoke them?" — which forces a zero-sum tradeoff between context economy and model awareness. Every workaround (subagents, overloaded tools, search+execute) just moves the coupling around without resolving it.

The fix would be a protocol-level separation between:
- **Capability advertisement**: structured, lightweight signals about what domains/intents the server covers (server-level metadata, not per-tool)
- **Tool execution**: the full schema-driven mechanism for invoking specific operations

Until that exists, server authors are stuck choosing between context bloat and invisible capabilities — and the model's organic usage will suffer accordingly.

---

### References:
#### From the blog:
[https://cra.mr/subagents-with-mcp](https://cra.mr/subagents-with-mcp)
[https://cra.mr/](https://cra.mr/)
[https://cra.mr/a-bigger-toolbox-for-mcp](https://cra.mr/a-bigger-toolbox-for-mcp)
[https://cra.mr/context-management-and-mcp](https://cra.mr/context-management-and-mcp)
[https://cra.mr/mcp-skills-and-agents](https://cra.mr/mcp-skills-and-agents)
[https://cra.mr/rethinking-the-definition-of-tools-in-mcp](https://cra.mr/rethinking-the-definition-of-tools-in-mcp)

#### From the MCP docs:
[https://modelcontextprotocol.io/docs/learn/architecture](https://modelcontextprotocol.io/docs/learn/architecture)
[https://modelcontextprotocol.io/specification/latest/basic/lifecycle](https://modelcontextprotocol.io/specification/latest/basic/lifecycle)
[https://modelcontextprotocol.io/llms.txt](https://modelcontextprotocol.io/llms.txt)
[https://modelcontextprotocol.io/community/server-card/charter.md](https://modelcontextprotocol.io/community/server-card/charter.md)
[https://modelcontextprotocol.io/docs/learn/server-concepts.md](https://modelcontextprotocol.io/docs/learn/server-concepts.md)