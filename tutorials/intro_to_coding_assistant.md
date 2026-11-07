# Intro to AI Code Assistants

When implemented [Task P1](../tasks/p1_registry_api.md), you've probably **vibe coded** the API with a coding agent (Claude Code, Cursor, or Codex). You picked some strong model and hoped for the best. 

This tutorial is about learning how to work with a coding agent the way a good engineer works.

We'll keep working on the **AgroPilot** app, in your own copy of the repo (`ArgoPilot`), on the `registry-service` you built in Task P1. Run all the commands from `services/registry-service` unless told otherwise.

## Coding agents 

As discussed in class, a coding agent is a language model running in a **loop** with access to **tools** (read a file, edit a file, run a command, search the web). The LLM on its own can only produce text or ask you to run some tool on your local machine (a.k.a. **tool calling**). When you run a tool, the result is added to the conversation and you call the LLM again. This form an **agentic loop**.

```
        ┌──────────────┐
        │  Your prompt │
        └──────┬───────┘
               ▼
        ┌──────────────┐
  ┌────▶│    LLM       │  decides the next step
  │     └──────┬───────┘
  │            │
  │     tool call? ── no ──▶  Final answer  (loop ends)
  │            │
  │           yes
  │            ▼
  │     ┌──────────────┐
  │     │  Run tool    │  read / edit / bash / grep ...
  │     └──────┬───────┘
  │            │ result (file content, test output, error)
  └────────────┘  appended to the conversation
```

Notice that the model **doesn't run anything**. It only returns with a tool call (e.g. "please run this command"). Your coding agent runs it on your machine and sends the output back to the model, in a new request that contains everything again.


## Context management

The **context window** is everything the model sees in a single request: the system prompt, your rules, the tool definitions, the files it read, the terminal output, and the **whole conversation so far**. It has a hard size limit, and you pay for all of it **on every message**.

Here is roughly what your coding agent sends to the model. It's shortened (`...`) and simplified, but the structure is real.

```json
{
  "model": "claude-sonnet-...",
  "max_tokens": 32000,

  "system": "You are Claude Code, Anthropic's official CLI for Claude.\nYou are an interactive agent that helps users with software engineering tasks...\n- Read a file before you edit it. \nPrefer the dedicated file tools over terminal commands like cat or sed.\n... (several pages of behavior, formatting and safety rules)\n\n# Environment\n- Primary working directory: /home/you/ArgoPilot\n- Is a git repository: true\n- Platform: linux\n- Shell: bash\n...",

  "tools": [
    {
      "name": "Read",
      "description": "Reads a file from the local filesystem. You can optionally specify a line offset and limit...",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": { "type": "string", "description": "The absolute path of the file to read." },
          "offset":    { "type": "integer", "description": "The line number to start reading from." },
          "limit":     { "type": "integer", "description": "The number of lines to read." }
        },
        "required": ["file_path"]
      }
    },
    {
      "name": "Bash",
      "description": "Executes a given bash command...",
      "input_schema": {
        "type": "object",
        "properties": { "command": { "type": "string" } },
        "required": ["command"]
      }
    },
    {
      "name": "Edit",
      "description": "Performs exact string replacements in files...",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path":  { "type": "string" },
          "old_string": { "type": "string" },
          "new_string": { "type": "string" }
        },
        "required": ["file_path", "old_string", "new_string"]
      }
    },
    { "name": "Grep", "description": "...", "input_schema": { "...": "..." } },
    { "name": "WebSearch", "description": "...", "input_schema": { "...": "..." } },
    { "name": "Skill", "description": "Execute a skill within the main conversation...", "input_schema": { "...": "..." } },
    { "name": "mcp__github__create_pull_request", "description": "(from the GitHub MCP server) ...", "input_schema": { "...": "..." } }
  ],

  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "<system-reminder>\nAs you answer the user's questions, you can use the following context:\n# claudeMd\nContents of /home/you/.claude/AGENTS.md (user's private global instructions for all projects):\n\nI'm not a developer. Explain changes in plain language.\n\nContents of /home/you/ArgoPilot/AGENTS.md (project instructions, checked into the codebase):\n\n# AgroPilot\n- Work in services/registry-service. Run the tests with: uv run pytest\n- Re-create the database with: bash seed_registry.sh. Never edit registry.db by hand.\n- Every error is application/problem+json with a code: VALIDATION, NOT_FOUND, CONFLICT or UNAVAILABLE.\n- Dates are always YYYY-MM-DD.\n</system-reminder>" },
        { "type": "text", "text": "<system-reminder>\nThe following skills are available for use with the Skill tool:\n- tdd: Test-driven development. Use when the user wants to fix bugs test-first...\n- code-review: Review the changes since a fixed point...\n</system-reminder>" },
        { "type": "text", "text": "<ide_selection>The user selected lines 24 to 34 from services/registry-service/main.py:\n...\n</ide_selection>\n\nPOST /v1/plots with area_m2 = 0 returns 201. Why?" }
      ]
    },
    {
      "role": "assistant",
      "content": [
        { "type": "text", "text": "Let me look at how a plot is validated." },
        { "type": "tool_use", "id": "call_1", "name": "Read", "input": { "file_path": "/home/you/ArgoPilot/services/registry-service/main.py" } }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "call_1",
          "content": "     1→\"\"\"FastAPI app and routes for AgroPilot registry-service.\"\"\"\n ...\n    24→class PlotIn(BaseModel):\n    25→    farm_id: UUID\n    26→    name: str = Field(min_length=1)\n    27→    area_m2: float = Field(ge=0)\n    28→    crop_id: str\n ...\n    41→@app.post(\"/v1/plots\", status_code=201)\n    42→def create_plot(plot: PlotIn):\n ... (the whole file, ~150 lines)"
        }
      ]
    },
    {
      "role": "assistant",
      "content": [
        { "type": "tool_use", "id": "call_2", "name": "Bash", "input": { "command": "cd services/registry-service && uv run pytest tests/test_plots.py" } }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "call_2",
          "content": "============================= test session starts =============================\ncollected 9 items\ntests/test_plots.py ...F.....                                            [100%]\n================================== FAILURES ===================================\n>   assert response.status_code == 422\nE   assert 201 == 422\n... (full traceback)\n========================= 1 failed, 8 passed in 0.41s ========================="
        }
      ]
    },
    {
      "role": "assistant",
      "content": [
        { "type": "text", "text": "Found it. `PlotIn.area_m2` is declared with `Field(ge=0)`, which means greater than **or equal to** 0, so an area of 0 passes validation. The spec says `area_m2` must be greater than 0. Here's the fix: ..." },
        { "type": "tool_use", "id": "call_3", "name": "Edit", "input": { "file_path": "/home/you/ArgoPilot/services/registry-service/main.py", "old_string": "...", "new_string": "..." } }
      ]
    },
    {
      "role": "user",
      "content": [
        { "type": "tool_result", "tool_use_id": "call_3", "content": "The file /home/you/ArgoPilot/services/registry-service/main.py has been updated successfully." }
      ]
    },
    {
      "role": "user",
      "content": "Great, now run the tests again and commit."
    }
  ]
}
```

> [!TIP]
> When the window is almost full, Claude Code compacts it.
> You can run `/compact` yourself (optionally with a focus, like `/compact keep the decisions about the API`).


>[!TIP]
> In Claude code, type `/context` to see how full the window is, and what's taking the space: system prompt, tools, `AGENTS.md` files, skills, MCP tools, and conversation (`/context all` also lists the memory files that were loaded).


## AGENTS.md: steering the agent

`AGENTS.md` is a plain markdown file in the project root. Agents add it to the context at the start of **every** session in this project. Type `/init` and Claude writes a first version for you, after it explores the project. It's an open standard (<https://agents.md>) read by Codex, Cursor, Copilot, and many others.


### The Karpathy guidelines

In early 2026 Andrej Karpathy (a founding member of OpenAI and former head of AI at Tesla) [posted a list](https://x.com/karpathy/status/2015883857489522876) of the ways LLM coding agents misbehave: they make wrong assumptions silently, overcomplicate code, and change things they shouldn't. Someone turned his observations into a single instructions file, and [the repo](https://github.com/forrestchang/andrej-karpathy-skills) went viral. It has four principles:

1. **Think Before Coding:** Don't assume. Don't hide confusion. Surface tradeoffs.
2. **Simplicity First:** Minimum code that solves the problem. Nothing speculative.
3. **Surgical Changes:** Touch only what you must. Clean up only your own mess.
4. **Goal-Driven Execution:** Define success criteria. Loop until verified.

Keep in mind what this file is and isn't. It **doesn't make the model smarter**, it's a **behavioral constraint**: it makes the agent stay inside the lines you drew, and ask when the lines are unclear.

## Rules 

`AGENTS.md` can grow big, and everything in it is loaded in every session. **Rules** let you split your instructions into small, focused markdown files, and apply them only to the files you need.

**Try it:**  For example, a rule for the registry service could look like this:

```markdown
---
paths:
  - "services/registry-service/**/*.py"
---

# Registry service rules

- Every request body is a Pydantic model. Never use a `dict` or loose parameters.
- Every numeric field has a constraint (`gt=0` or `ge=0`). No unbounded numbers.
- Every endpoint needs a test in `tests/`, including at least one invalid-input case.
```

Save it as `.claude/rules/registry-service.md`, then start a new session and send this prompt:

> Add `POST /v1/irrigation/volume`. Given `area_m2` and `depth_mm`, it returns the water volume in liters.

Rules files lives in the following locations:

- **Project rules** live in `.claude/rules/`. They apply only in this project, and since they're in the repo, the whole team gets them.
- **User rules** live in `~/.claude/rules/`. They apply in all your projects. Use them for your personal preferences.


## Skills

A **skill** is one or more Markdown files that teach the agent how to do one specific kind of task. 

Each skills has it's own folder with a `SKILL.md` file of instructions, and it can also hold scripts for the agent to run, templates, and reference documents.

Skills are an open standard ([agentskills.io](https://agentskills.io)), so the same skill works in Cursor, Claude Code, Codex, and others.

Two kinds of skills (but many skills mix both kinds): 

- **Procedural skills** teach a *process*: which steps to take, in what order, and when to stop and ask you.
- **Knowledge skills** teach *what the model doesn't know*: your company's conventions, an internal API, or how drip irrigation works. 

Here's a knowledge skill for our app:

```markdown
---
name: drip-irrigation
description: How drip irrigation is described in AgroPilot data (soils, emitters, spacing, plot area) and how to calculate with it. Use when working with plots, soils, emitters, or irrigation calculations.
---

# Drip irrigation in AgroPilot

[useful information the agent should know about how your organization describes and calculates irrigation: units, formulas, rounding rules, worked examples with numbers]
```

The header (a.k.a. **Frontmatter**) is used to identify the skill and provide a description.

The agent decides if the description fits what you asked, and loads the full `SKILL.md`, usually **automatically**. You don't have to ask for it. But if you want to, you can type `/` in the chat and pick it from the list. 

**Try it:** you don't need to write a skill to see one work. Make any small change in your project (for example, the `POST /v1/irrigation/volume` endpoint from the Rules section), then type:

```
/code-review
```

The agent loads the built-in `code-review` skill, looks at your uncommitted changes, and replies with a list of issues (bugs, missing tests, unclear code). You typed one word, and got a whole review process that you didn't write.


> [!TIP]
> Claude Code comes with bundled skills, for example `/code-review` (reviews your changes), `/debug` (helps when something is wrong in your session), `/verify` (builds and runs your app to confirm a change does what it should), `/loop` (runs a prompt again and again), and `/doctor` (checks your setup, including your `AGENTS.md`). They show up in the `/` list next to the skills you add.




# Exercises


### :pencil2: Your first AI agent, on your own machine with Ollama

In this exercise we will meet [Ollama](https://ollama.com): an open-source tool that downloads **open-source LLMs** and runs them **on your own computer**. It also starts a small web server on `localhost:11434` that exposes the model through an API.

A model that runs on a laptop CPU is much weaker than the models behind Claude Code. That's fine: our goal is to understand how an agent works, and a small model is enough for that.

**Our goal:** build an agent that can answer *"What is the date today?"*. A language model on its own can **never** answer this question correctly. It only knows the text it was trained on, and it has no clock. We will see it fail, and then give it a tool that fixes it.

1. Download it from <https://ollama.com/download>. On Linux, and in the WSL terminal for Windows users, run:

  ```bash
  curl -fsSL https://ollama.com/install.sh | sh
  ```

  Check that the server is running:

  ```bash
  curl http://localhost:11434
  ```

  You should get `Ollama is running`. If not, start it in a separate terminal with `ollama serve`.

2. Ollama's [library](https://ollama.com/search?c=tools&s=8) lists the available models (we filtered by the **tools** capability and up to 8GB of size). Choose some model. A good start is `ministral-3` or `qwen3:1.7b`.

3. Download the model and chat with it in the terminal. Assuming you chose `qwen3:1.7b`, run:

    ```bash
    ollama run qwen3:1.7b
    ```

  Ask it *"What is the date today?"*.


4. Talk to Ollama over HTTP

  Ollama has an API, so any program can talk to it. Here is the same conversation with `curl`:

  ```bash
  curl http://localhost:11434/api/chat -d '{
    "model": "qwen3:1.7b",
    "stream": false,
    "messages": [
      { "role": "user", "content": "What is the date today?" }
    ]
  }'
  ```

  The answer is in `message.content`. It is a wrong date again. Notice that the request carries the **whole conversation** in `messages`. The server doesn't remember anything between requests.

  Now let the model know about a **tool**. A tool is just a description (name, purpose, parameters) that we send along with the messages:

  ```bash
  curl http://localhost:11434/api/chat -d '{
    "model": "qwen3:1.7b",
    "stream": false,
    "messages": [
      { "role": "user", "content": "What is the date today?" }
    ],
    "tools": [{
      "type": "function",
      "function": {
        "name": "current_date",
        "description": "Get the current date and day of the week",
        "parameters": {
          "type": "object",
          "properties": {}
        }
      }
    }]
  }'
  ```

  Look at the answer. Instead of text, `message.tool_calls` should hold the name of the tool (`current_date`) and empty arguments (`{}`), because this tool takes none. The model **did not run anything**. It only asked *us* to run the tool.

#### Step V: build the simplest AI agent

An agent is just a loop: ask the model, if it asks for a tool, run the tool, send the result back, and ask again, until the model answers with text. Let's have your coding agent build it.

Open your AgroPilot repo, start a new Claude Code session, and send this prompt (replace the model name if you chose another one). The agent creates the experiment in its own folder, away from `registry-service`:

```text
Build a minimal terminal chat agent in `experiments/ollama-agent/agent.py` (under 50 lines).
Use uv, the `ollama` Python package, and no framework. Read
https://docs.ollama.com/capabilities/tool-calling first.

- Use qwen3:1.7b (thinking off).
- One tool, `current_date()`, that returns today's date and weekday, e.g. "2026-11-07 (Saturday)".
- Loop: read a user line, call the model with the tool, run any tool calls, send the results
  back, and repeat until the model answers with text. Keep one messages list. `exit` quits.
- System message: "You don't know today's date. Always use the tool, never guess."
```

Read the code. Do you understand every line? Find the loop, the place where the tool runs, and the `messages` list.

Run it and talk to it.



### :pencil2: Move the registry to PostgreSQL, with Superpowers skills set

Today the registry service uses one SQLite file, `registry.db`, created by `seed_registry.sh`. That works on your laptop, but SQLite is not a good choice for a real production environment.

Your mission: let the registry run on a real database. According to the requirements below: 

1. The existing endpoints must keep behaving **exactly** the same: same status codes, same `application/problem+json` errors and codes, same `Retry-After` on `503`.
2. When running the app locally or during tests, it should still use the SQLite database file `registry.db`. When running in production, it should work with an existing PostgreSQL database. To switch between the two, the app should read the database URL from the environment variable `DATABASE_URL` from the `.env` file.


Quite a large and risky change. 
It touches data access, models, error handling, tests, and setup, and the existing endpoints must keep behaving **exactly** the same. 

That's where vibe coding will get you into trouble! 

We'll use [Superpowers](https://github.com/obra/superpowers), it is a complete **development process** done by senior engineers as set of skills that call each other.

#### Setp I

First let's install the plugin, from the official Claude Code marketplace. Start a Claude Code session, open the command menu with `/` and select `Plugins`. In the search bar, search for `superpowers` and install the plugin.


#### Setp II

To work with Superpowers, you usually start with a **brainstorming process** with the agent. 
Open a new conversation and choose some strong model for brainstorming (Sonnet or above). 

Type `/brainstorming` to load the skill, and then send the short requirements above

The agent will ask you questions. At the end, the skill writes a **design document** and asks you to approve it. Read it. Is everything you decided there? Is anything there that you *didn't* decide? Fix it before you approve.

#### Setp III

Next, Superpowers writes an implementation plan. A plan is a list of **small**, **simple** tasks with a clear "done" check.

Once you approve the plan, Superpowers is ready to execute the tasks. Two options ahead: 

- **Subagent-Driven (recommended)**: using the `/subagent-driver-development` skill - a fresh subagent handles each task, followed by a two-stage review (subagents will be discussed later on in the course)
- **Inline Execution**: using `/executing-plans` skill - tasks run in the current coversation session . 


> [!TIP]
> Depending on the amount of work, but you might want to **execute the plan** with relatievly a cheap model, since the plan is just a list of simple tasks.

