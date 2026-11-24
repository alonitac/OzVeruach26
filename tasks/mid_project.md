# Mid-course project

In 3 weeks you should present a system you designed and built yourself. Pick something you actually care about: a real company, a product you use, or any tech system you're curious about.

The goal: rebuild a slice of a real system, using what we learned in the course (services, REST APIs, Git and PRs, CI/CD, tests, AWS), **and** go beyond it. A project that its architecture is a copy of AgroPilot will not impress anyone.

| Part | What | When |
|------|------|------|
| 1 | Spec document | 02-12-2026 |
| 2 | Implementation | Due 12-12-2026 |
| 3 | Internal review with the course staff | Due 15-12-2026 |

## Part 1: Research and spec

Research your company or product with an AI assistant, the same way we did for Netafim and AgroPilot: what the business does, who the users are, what pain the product solves, how the real system probably works. Then cut it down to a slice you can build in 2 weeks.

The deliverable is a **short spec document**, `docs/spec.md` in your repo.

> [!IMPORTANT]
> **Short. Max 4 pages.** If we can't read it in 10 minutes and understand your project, it doesn't pass.
>
> Your job is to cut what the AI wrote. No intros, no "Overview" sections that repeat the title, no buzzwords, no paragraphs that could fit any project. Prefer tables, bullet points and diagrams over long text.

The spec might cover:

1. **Business.** The company/product, who the users are, and the problem you solve. 3-5 lines.
2. **The solution.** What your system does, and what it deliberately does **not** do (scope). 3-5 lines.
3. **User flow.** The main flow as seen by the user and system.
4. **Service architecture.** A diagram of your services and how they talk. For each service, one line: its responsibility, and what it owns (data, external API).
5. **FR and NFR**.
7. **Data model.** Main entities and relations. A small diagram or table.
8. **Cloud architecture.** A diagram of what runs where in AWS
9. **Test plan.** How you test your system.

### Going beyond the course

The system should be deployed in AWS. You are welcome (and encouraged) to find services in AWS (beyond the ones we covered in the course) that are relevant to your project, for example:

- **SQS** - a queue between services, for slow or async work.
- **SNS** - notifications (email, SMS) or fan-out to several consumers.
- **Lambda** - a function triggered by an event (an S3 upload, a schedule, a queue message).
- **DynamoDB, RDS, CloudWatch, EventBridge**, ...

### AI inside your system

AI must be part of the product itself, not only a tool you used to write the code. Plan for it in your spec. In the next 2 weeks we'll cover:

- **Simple RAG** - answer questions grounded in your own data (docs, manuals, records).
- **AI agents with LangGraph** - an agent that calls your services as tools and runs a multi-step flow.
- **MCP** - expose your system to AI clients, or connect your agent to external systems.
- **Langfuse** - trace and monitor your LLM calls: prompts, outputs, latency and cost.
- **System design patterns** - use them when they fit your architecture.

We prefer to design the AI part around a real user need, not as a chatbot added on the side.

### Submit

A PR with `docs/spec.md` into `main`. We'll comment on the PR. Don't start Part 2 before the spec is approved.

## Part 2: Implementation

Build it. The minimum bar:

- At least **2 services** that talk over HTTP (Optional - an OpenAPI contract per service, committed to `contracts/openapi/<service>.json`).
- Structured errors, and predefinied behavior whenn not in the happy path. 
- Automated tests, with external dependencies (S3, external APIs) mocked. They run in CI.
- CI on every PR, CD to EC2 on merge to `main`.
- An AI feature in the product (RAG, a LangGraph agent, and/or MCP), as planned in the spec, traced with Langfuse.
- A basic UI to demo the main user flow.
- Work in branches and PRs, not directly on `main`. FROM DAY 1. 

Keep the spec up to date. If the implementation changed the design, update the spec in the same PR.

## Part 3: Internal review

Before demo day, you'll do a 20-minute review with the course staff. Come with:

- The system **deployed and working**. We will use it, not watch slides.
- The spec, up to date.
- Your repo open: PRs, CI runs, tests.

We'll give you feedback, and you'll have time to fix things before demo day.


# Good Luck!