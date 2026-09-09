# MCP Server-Client Demo — Explained Simply

This document explains the project in plain language: what it is, why
it works the way it does, and what each piece of code actually does.
No prior MCP knowledge assumed.

---

## 1. What problem does this project solve?

Imagine you're chatting with an AI (like Claude). The AI is great at
language, but it can't check your notes, look up live data, or run
code on its own — it can only work with what's inside the
conversation.

**Tools** fix this: they let the AI ask a program to do something
("search my notes for X") and get a real answer back before replying
to you.

The question is: *how does the AI know what tools exist, and how does
it actually call them?*

**MCP (Model Context Protocol)** is a standard, agreed-upon way to
answer that question. Instead of every company inventing its own
private way to hook up tools, MCP defines one common "language" that
any tool-provider (a *server*) and any AI-application (a *client*) can
both speak. Think of it like USB — one plug shape that works across
many devices, instead of a different cable for every gadget.

This project builds the two halves of that conversation from scratch,
so you can see exactly what's being sent back and forth.

---

## 2. The three characters in this story

| Character | Real-world analogy | In this project |
|---|---|---|
| **Server** | A vending machine — offers a fixed menu of things it can do | `server.py` — offers two tools: `search_notes`, `get_note_by_id` |
| **Client** | The person operating the vending machine on someone else's behalf | `client.py` — connects to the server, and manages the conversation |
| **LLM (Claude)** | The customer who decides *what* they want, but needs the operator to press the buttons | The Claude API — decides *when* to use a tool and *what* to ask for |

Important: **Claude never talks to the server directly.** Claude only
ever talks to the client. The client is the go-between — it shows
Claude the "menu" (available tools), passes along Claude's requests,
and reports back the results.

---

## 3. What actually happens, step by step

1. **You** type a question, e.g. *"find my notes about MCP"*.
2. **The client** starts up the server as a background program and
   asks it, "what tools do you offer?" The server replies with a list
   — in this case `search_notes` and `get_note_by_id` — along with a
   description of what each one needs as input.
3. **The client** hands that tool list to Claude, along with your
   question, and says "here's what you can use if you need it."
4. **Claude** reads your question and decides: *"I should use
   `search_notes` with the word 'MCP'."* It doesn't run anything
   itself — it just replies with a structured request, like a filled-out
   order form.
5. **The client** takes that order form and forwards it to the server:
   "please run `search_notes(query='MCP')`."
6. **The server** actually runs the search against its notes list and
   sends back the matching results as plain text.
7. **The client** hands those results back to Claude.
8. **Claude** reads the results and writes a normal, friendly answer
   for you — e.g. *"You have one note about MCP: it explains that MCP
   separates tool hosting from tool use."*
9. **You** see that final answer.

Everything from step 2 to step 7 is the part MCP standardizes. Steps
1 and 9 are just you talking to the app.

---

## 4. Why two separate files (`server.py` and `client.py`)?

This separation is the whole point of MCP. In the real world:

- A **server** might be built by one team (or company) and just
  focuses on exposing useful capabilities — like "search our
  database" or "send an email."
- A **client** might be built by a completely different team, and its
  job is just to connect various servers to an AI model and manage
  the conversation.

Because both sides agree to speak MCP, they can be swapped
independently. You could point this same `client.py` at a totally
different server (say, one that checks the weather) with almost no
changes — the client doesn't need to know in advance exactly what
tools it'll find; it discovers them at runtime.

---

## 5. Walking through `server.py` in plain terms

```python
mcp = FastMCP("notes-demo-server")

@mcp.tool()
def search_notes(query: str) -> str:
    ...
```

- `FastMCP("notes-demo-server")` creates "the vending machine" and
  gives it a name.
- `@mcp.tool()` is a label that says "this Python function is one of
  the machine's buttons — expose it to anyone who connects."
- The function's **docstring** (the text in `"""triple quotes"""`)
  isn't just a comment for humans — MCP actually reads it and sends it
  to the client as the tool's *description*. This is how Claude knows
  *when* to use the tool, without ever seeing the code.
- The function's **type hints** (`query: str`, `-> str`) tell MCP what
  shape of input the tool expects and what it returns. MCP turns this
  into a formal schema automatically.
- Inside the function is completely ordinary Python — in this demo, a
  simple search through a hardcoded list of notes. In a real system,
  this could hit a database, call an API, or run any other logic.
- `mcp.run(transport="stdio")` at the bottom means: "wait for
  instructions arriving over stdin/stdout" (a very simple, local way
  for two programs to talk — no network, no ports, no setup).

---

## 6. Walking through `client.py` in plain terms

```python
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
```

- This block **starts the server as a subprocess** and opens a
  two-way pipe to it (`read`/`write`). `session.initialize()` is the
  MCP equivalent of a handshake — "hello, are you there? what version
  of the protocol do you speak?"

```python
tools_result = await session.list_tools()
```

- This is the client asking the server, "what can you do?" The server
  answers with the tool names, descriptions, and input schemas — this
  is literally the docstrings and type hints from `server.py`,
  transmitted over the pipe as structured data.

```python
claude_tools = mcp_tools_to_anthropic_format(tools_result.tools)
```

- MCP's tool format and Claude's tool format are both JSON-based but
  shaped slightly differently. This function just relabels the fields
  so Claude can understand the menu it was just handed.

```python
response = anthropic.messages.create(..., tools=claude_tools, messages=messages)
```

- This sends your question to Claude, along with the (translated)
  tool menu. Claude replies either with a normal text answer, **or**
  with a request to use a tool — MCP/Claude call this a `tool_use`
  block.

```python
while response.stop_reason == "tool_use":
    ...
    result = await session.call_tool(block.name, block.input)
```

- This loop is the heart of the demo. As long as Claude keeps asking
  to use tools, the client:
  1. reads exactly which tool and which arguments Claude wants,
  2. sends that request to the server over the same pipe,
  3. gets the server's answer back,
  4. hands that answer back to Claude,
  5. asks Claude to respond again — now with the new information.
- Once Claude has everything it needs, it stops requesting tools and
  gives a final plain-English answer, which the client prints out.

---

## 7. Key terms in one line each

- **MCP (Model Context Protocol)** — a shared standard for how AI
  clients discover and call tools hosted by servers.
- **Server** — a program that *hosts* one or more tools and answers
  requests about them.
- **Client** — a program that *connects to* servers, shows their
  tools to an AI, and relays requests/results between them.
- **Tool** — a single named capability (a function) with a description
  and an input/output shape, exposed by a server.
- **stdio transport** — the simplest way two local programs can
  exchange MCP messages, using their standard input/output streams
  instead of a network connection.
- **Tool schema** — a structured description (name, description,
  expected inputs) that tells the AI what a tool does and how to call
  it correctly, without the AI ever seeing the underlying code.
- **tool_use / tool_result** — the two message types Claude's API uses
  to (a) request a tool call and (b) receive the tool's answer.

---

## 8. Why this matters beyond the demo

At work, tool integrations are often already wired up for you inside a
larger framework, so it's easy to use them without seeing what's
underneath. Building both the server and the client by hand — and
watching the raw request/response messages travel between them —
makes the protocol concrete: it's just structured text messages over a
pipe, following an agreed format. That understanding transfers
directly to debugging, extending, or evaluating any MCP-based system
later, production or otherwise.
