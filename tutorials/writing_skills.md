# Writing Skills

So far you've used skills that other people wrote. In this tutorial you'll write your own.

Remember what a skill is: a folder with a `SKILL.md` file that teaches the agent how to do one kind of task. The agent sees only the skill's **description** all the time. It loads the rest of the file only when the description matches what you asked.


## Where skills live

```text
.claude/skills/              # project skills: committed, the whole team gets them
  registry-api/
    SKILL.md                 # the instructions (required)
~/.claude/skills/            # user skills: only on your computer, in all your projects
```


`SKILL.md` might look like this (the folder name and the `name` in the file must be the same, `registry-api`):

```markdown
---
name: registry-api
description: How we build endpoints in the AgroPilot registry-service (FastAPI, SQLite). Use whenever you add, change, or test an endpoint, a request or response model, or an error response in registry-service.
---

# Registry API conventions

## Adding an endpoint
1. ...
```

The part between the `---` lines is the **frontmatter**:

| Field | What it's for |
| --- | --- |
| `name` | Lowercase, with dashes. Same as the folder name. |
| `description` | **The most important line.** The agent decides whether to load the skill based on it alone. |
| `disable-model-invocation` | Optional. `true` means the skill runs only when you type `/name`, never automatically. |
| `allowed-tools` | Optional. Tools Claude can use without asking for permission while the skill is active, like `Bash(uv run pytest *)`. |

## How to write a good skill

**1. The description says *what* and *when*.** "API helper" won't load when it should. "How we build endpoints in the AgroPilot registry-service. Use whenever you add, change, or test an endpoint, a request or response model, or an error response." will. Mention the words people actually use when asking. Be a bit pushy: agents tend to skip skills, not overuse them.

**2. Short.** The agent is already smart. It knows FastAPI and REST. Write only what it **doesn't** know: your team's rules, your names, your order of steps. Keep `SKILL.md` well under 100-200 lines. Move long reference material to `references/` and point to it ("for the full list of error codes and when to use each, read `references/errors.md`").

**3. Explain why.** "Return only the fields in the spec" works. "Return only the fields in the spec: other services read these responses, and every extra field becomes something we can't remove later" works better. When the agent understands the reason, it handles situations you didn't think of.


## Tools for writing skills

You don't have to write skills from scratch. Let an agent interview you and write the first draft.

**`skill-creator`** (by Anthropic) asks what the skill should do, asks about edge cases, and creates the folder and `SKILL.md` in the right place. Then it **tests the skill with evals** (see below), and can even tune the description so the skill loads when it should. Install it:

```bash
npx skills add anthropics/skills --skill skill-creator -a claude-code
```

Restart Claude Code, then type `/skill-creator` to use it.

Whatever tool you use, **read the result**. It's your team's process, not the agent's.

## Skill evals

How do you know your skill works? You test it, like code. An **eval** is a test for a skill:

1. **A prompt**, like a real user would write: *"Add a `GET /v1/farms/{id}/area` endpoint"*.
2. **What should happen**, as checks you can verify: "an unknown farm id returns `404` with `application/problem+json` and `code: NOT_FOUND`", "a test for the `404` case was added".
3. **Run it with the skill and without it**, and compare. If the result is the same, the skill doesn't add anything.

Evals check two things: that the skill **loads** when it should (and doesn't when it shouldn't), and that once it's loaded, the agent **does the right thing**.

### How evals are graded

You ran the prompt. Now who decides if it passed? There are three ways, and good evals mix them:

- **Code checks.** Anything you can check with a command, check with a command. Does an unknown farm return the right error? Run `curl -si localhost:8000/v1/farms/00000000-0000-0000-0000-000000000000/area` and check the status, the `Content-Type` header, and the `code`. Do the tests pass? Run `uv run pytest`. It's fast, free, and always gives the same answer. Use it whenever you can.

- **LLM as a judge.** Some things a command can't check: "do the tests cover the failure cases, not only the happy path?", "is the error `detail` useful to the client?". So you give **another model** the output, plus the checks, and ask it to grade: pass or fail for each check, and the **evidence** (a quote from the output). It's flexible, but the judge is an LLM too, so it can be wrong.

- **Human review.** You read some of the results yourself. Especially at the start, to make sure the judge grades like you would.

Agents are not deterministic, so one good run proves little. Run each eval a few times, and look at the **pass rate** (for example, 5 of 6 runs passed), with and without the skill.

`skill-creator` automates all this. It keeps the evals in `evals/evals.json` inside the skill folder. For each prompt, it starts two subagents, one with the skill and one without, then a **grader** agent (an LLM judge) checks each assertion, and writes a report with pass rates, time, and tokens:

```json
{
  "skill_name": "registry-api",
  "evals": [
    {
      "id": 1,
      "prompt": "Add a GET /v1/farms/{id}/area endpoint that returns the total area of the farm's plots.",
      "expected_output": "A /v1 route that returns only the listed fields, an RFC 9457 error for an unknown farm, and tests for both cases.",
      "assertions": [
        "An unknown farm id returns 404 with Content-Type application/problem+json and code NOT_FOUND",
        "A test for the 404 case was added",
        "uv run pytest passes"
      ]
    }
  ]
}
```

Every time you change the skill, run the evals again, to make sure you didn't break something that worked.


# Exercises

### :pencil2: A registry API skill

In Task P1, you probably told the agent the same things again and again: "errors must be `application/problem+json`", "use the `VALIDATION` code", "don't return extra fields", "add a test for that". Let's write these down once, so the agent follows them every time it touches the API.

From `services/registry-service`, start a new Claude Code session:


> /skill-creator A project skill called `registry-api` that makes the agent follow our API conventions every time it adds or changes an endpoint in registry-service:
> - Every route is under `/v1`. Lists are JSON arrays, and no rows means `200` and `[]`.
> - Return only the fields the spec lists. Never add fields on your own.
> - `PATCH` changes only the fields in the body.
> - Every error is `application/problem+json` (RFC 9457) with a stable `code`: `VALIDATION` (422), `NOT_FOUND` (404), `CONFLICT` (409), `UNAVAILABLE` (503, with a `Retry-After` header). Never return FastAPI's default error body.
> - Every endpoint gets tests: the success case, and every failure case it can return.
> - Run `uv run pytest` before saying the work is done.
> - If the spec doesn't say what to do in some case, ask. Don't invent behavior.


Read the `SKILL.md` it created, and check it against the guidelines above. Did it put an example error body in the skill? Did it explain *why* for the rules?

Commit the skill. It's a project skill (in `.claude/skills/`), so after you merge it, everyone on the team gets it.

Let's try it. Start a new session (`/clear`):

> Add a `GET /v1/farms/{id}/area` endpoint that returns the total area of the farm's plots.

Don't mention errors, tests, or conventions at all. Ask for an unknown farm id: is the answer `404`, with `application/problem+json` and `code: NOT_FOUND`? Did the agent write tests for the failure case? Did it run them?

### :pencil2: Skills evals

Does your `registry-api` skill really work? Let's ask `skill-creator` to test it.

New session (`/clear`):

> /skill-creator Write evals for the `registry-api` skill. If possible, use code checks (curl against the running service, and `uv run pytest`). Otherwise, use an LLM as a judge. Run the evals, with and without the skill.

Look at what it created (usually an `evals/evals.json` file inside the skill folder):

- **The prompts.** Are they realistic? Is there at least one prompt where the skill should **not** load (for example, *"Explain what `available_water_mm_per_m` means"*)?
- **The checks.** Which ones are code checks (a command like `curl -si ...` or `uv run pytest`), and which ones need a judge? Could any of the judge checks be a code check?

Missing something? Ask it to add it.

If an eval failed, improve the skill (usually the description), and run **all** the evals again. A change that fixes one eval can break another.
