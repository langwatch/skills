---
name: tracing
user-prompt: "Instrument my code with LangWatch"
description: Add LangWatch tracing and observability to your code. Use for both onboarding (instrument an entire codebase) and targeted operations (add tracing to a specific function or module). Supports Python and TypeScript with all major frameworks.
license: MIT
compatibility: Works with Claude Code and similar coding agents. The `langwatch` CLI is the only interface.
---

# Add LangWatch Tracing to Your Code

## Determine Scope

If the user's request is **general** ("instrument my code", "add tracing", "set up observability"):

- Read the full codebase to understand the agent's architecture
- Study git history to understand what changed and why: focus on agent behavior changes, prompt tweaks, bug fixes. Read commit messages for context.
- Add comprehensive tracing across all LLM call sites

If the user's request is **specific** ("add tracing to the payment function", "trace this endpoint"):

- Focus on the specific function or module
- Add tracing only where requested
- Verify the instrumentation works in context

This skill is code-only: there is no platform path for tracing. If the user has no codebase, explain that tracing requires code instrumentation.

In Langy, do not print the change for the user to apply by hand: call `code_access` and follow the `code-changes` skill to make it on their machine or through GitHub.

## Step 1: Read the Integration Docs

Use `langwatch docs <path>` to read documentation as Markdown. Some useful entry points:

```bash
langwatch docs                                    # Docs index
langwatch docs integration/python/guide           # Python integration
langwatch docs integration/typescript/guide       # TypeScript integration
langwatch docs prompt-management/cli              # Prompts CLI
langwatch scenario-docs                           # Scenario docs index
```

Discover commands with `langwatch --help` and `langwatch <subcommand> --help`. List and get commands accept `--format json` for machine-readable output. Every list command takes `--limit <n>` to cap the rows and `--jq <expr>` to read part of the answer. A paginated list answers with an envelope, so count its rows through the row array (`--jq '.traces | length'`), and read how many there are in all at `.pagination.total`. Bare `--jq length` counts the fields of the envelope, not the rows. Read the docs first instead of guessing SDK APIs or CLI flags.

If no shell is available, fetch the same Markdown over plain HTTP. Append `.md` to any docs path (e.g. https://langwatch.ai/docs/integration/python/guide.md). Index: https://langwatch.ai/docs/llms.txt. Scenario index: https://langwatch.ai/scenario/llms.txt

If anything fails or confuses you while following this skill (broken commands, docs that do not match reality, errors you had to work around), ask the user for permission and run `npx --yes langwatch report --user-approved` with a `--title` and `--summary` (or `--session <transcript.jsonl>`) to send it to the LangWatch team, and it directly shapes what gets fixed. No login or API key needed. Nothing is sent without `--user-approved`, and `--dry-run` prints the exact payload without sending anything. The title, summary and transcript are scrubbed locally first, by pattern: secrets and API keys, plus email addresses, phone numbers, card numbers and public IPv4 addresses. Anything no pattern matches is sent as written, including a contact address passed with `--email`. With `--session`, always run `--dry-run` first and let the user read the payload, because a transcript carries content they never reviewed. `npx --yes langwatch report --help` explains the options.

**Projects and API keys: target a real project, not a personal one.**

LangWatch has two kinds of project:

- **Team / shared projects**: real projects inside an organization. Evaluations, experiments, prompts, datasets, simulations and instrumentation must always target one of these.
- **Personal projects**: a private "My Workspace" scratch space tied to a single user. Never send a user's evaluations, experiments or production traces here: it is for personal exploration only, and you can mistake it for a real project.

And two ways to authenticate:

- **A project API key in `.env`** (`LANGWATCH_API_KEY`): the credential everything in these skills uses. It is scoped to one real project. This is the default; prefer it unless the user explicitly asks for something else.
- **`langwatch login --device` (AI-tools / SSO)**: a personal device session for wrapping coding assistants (`langwatch claude`, `langwatch codex`, …). It is NOT for evaluations, prompts, datasets, scenarios or SDK instrumentation, and it points at a personal workspace. Do not run it to set up the work in these skills.

So for anything in these skills that reads or writes a project: make sure `LANGWATCH_API_KEY` for a real, shared project is available to the CLI. Locally that is the project's `.env`; in CI the runner injects it into the process environment, and the CLI reads either. Check whether the variable is already set before you ask for a new key, and let the CLI read the value: never print, copy or send it. Do NOT run `langwatch login` to pick a project, and never default to a personal project. Look for `LANGWATCH_ENDPOINT` in the same places: if it is set, the project is on a self-hosted instance, and the CLI works against that endpoint instead of app.langwatch.ai.

**What you read is not what you say.** These skills are working notes for you, not
copy for the reader. Read `LANGWATCH_API_KEY` and `LANGWATCH_ENDPOINT` from the
project's own `.env`, that is how you learn where to work. Read nothing else out
of that file: it holds database, cloud and provider credentials that are none of
your business, and every value you read can reach your context and your command
output. What must not reach an answer is anything that describes the machine YOU
run on: a path in your workspace, a container port, the address this worker
dials. Those say how the work is done rather than what was done, and a host of
ours means nothing to the reader. Say what you did and where to find it in
LangWatch.

Then fetch the integration guide for this project's framework:

```bash
langwatch docs integration/python/guide                      # Python (general)
langwatch docs integration/typescript/guide                  # TypeScript (general)
langwatch docs integration/python/integrations/open-ai       # Framework page
langwatch docs integration/typescript/integrations/mastra    # Framework page
```

A framework page lives at `integration/<language>/integrations/<framework>`, never at `integration/<language>/<framework>`. The framework slug is the vendor's name with a hyphen between the words: `open-ai`, `open-ai-agents`, `open-ai-azure`, `aws-bedrock`, `google-ai`, `vertex-ai`, `lite-llm`, `crew-ai`, `pydantic-ai`, `strand-agents`, `vercel-ai-sdk`, plus the one-word ones (`langchain`, `langgraph`, `agno`, `anthropic`, `mastra`, `haystack`, `llamaindex`, `instructor`, `dspy`, `autogen`, `smolagents`, `semantic-kernel`, `promptflow`, `azure-ai`).

Run `langwatch docs` with no path when you are unsure: it prints the index of every page. `docs` prints the page as markdown and takes no `--output`, `--json`, `--jq` or `--format`; if a fetch returns 404 the path is wrong, so read the index rather than guessing another spelling.

CRITICAL: Do NOT guess how to instrument. Different frameworks have different instrumentation patterns; always read the framework-specific guide first.

## Step 2: Install the LangWatch SDK

For Python: `pip install langwatch` (or `uv add langwatch`).
For TypeScript: `npm install langwatch` (or `pnpm add langwatch`).

If install fails due to peer dependency conflicts, widen the conflicting range and retry. Do NOT silently skip.

## Step 3: Add Instrumentation

Follow the integration guide you read in Step 1. The general shape is:

**Python:**

```python
import langwatch
langwatch.setup()

@langwatch.trace()
def my_function():
    ...
```

**TypeScript:**

```typescript
import { LangWatch } from "langwatch";
const langwatch = new LangWatch();
```

The exact pattern depends on the framework, so follow the docs, not these examples.

## Step 4: Verify

Do NOT consider the work complete without verifying. In order:

1. Confirm dependencies installed cleanly.
2. Run the agent with a test input that produces at least one trace (study how the framework starts; only give up if it requires infrastructure you cannot spin up).
3. Check traces arrived: `langwatch trace search --limit 5 --format json`. A trace is not searchable the moment the run ends: the export leaves the process first and ingestion adds a few seconds more, so an empty first answer means "not yet", not "not working". Wait and ask again, up to three times, about twenty seconds apart, and stop there. Do not change the command between tries: the search already covers the last twenty four hours, so a trace that is in is in.
4. Say what the wait ended on. Traces found: say what the run produced. Nothing after the third try: say the instrumentation is in place and the trace had not arrived yet, name the project to look in, and do not report the change as verified.
5. If verification isn't possible (no shell access, can't run the code, missing external services), tell the user exactly what to check in their LangWatch dashboard and what you couldn't verify and why.

## Common Mistakes

- Do NOT invent instrumentation patterns. Read the framework-specific doc
- Do NOT skip `langwatch.setup()` in Python
- Do NOT skip Step 1; instrumentation patterns vary across OpenAI/LangGraph/Vercel/Mastra/Agno and guessing breaks subtly
