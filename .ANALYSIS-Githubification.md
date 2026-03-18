# Githubification Analysis — Pydantic AI

### How this repository could become a GitHub Action based mechanism

---

## What Is Githubification

[Githubification](https://github.com/japer-technology/githubification) is the act of converting a repository into **GitHub-as-infrastructure**. Instead of cloning the repo and running the software elsewhere, the repo becomes something that **runs on GitHub itself** via GitHub Actions. There is no separate local runtime to install — GitHub is the runtime.

Four GitHub primitives serve four roles:

| GitHub Primitive | Role |
|---|---|
| **GitHub Actions** | Compute — the runner that executes the agent |
| **Git** | Storage and memory — sessions, conversations, state are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread |
| **GitHub Secrets** | Credential store — LLM API keys and tokens |

---

## What Is GitHub Minimum Intelligence (GMI)

[GitHub Minimum Intelligence](https://github.com/japer-technology/github-minimum-intelligence) is a repository-local AI agent framework that plugs into a developer's existing workflow. It uses GitHub Issues for conversation, Git for persistent versioned memory, and GitHub Actions for execution. It is installed by adding one workflow file and running it once — the agent installs itself, opens conversations through Issues, and commits every prompt/response to the repository.

GMI demonstrates the **Native** Githubification strategy: the agent was designed for GitHub from the start. Its lifecycle is a closed loop — Issue opened → workflow triggers → agent processes → reply posted as comment → state committed to git.

---

## This Repository's Current State

This repository (`japer-technology/githubification-pydantic-ai`) is a fork of [pydantic/pydantic-ai](https://github.com/pydantic/pydantic-ai), the GenAI agent framework for Python. It is a **Type 1 — AI Agent Repo**: the repository already contains an AI agent framework. Githubification would convert the framework's functionality from something that must be installed and run locally into something that **runs natively inside GitHub as an Action**.

### What Already Exists

This repo is uniquely positioned because **AI agents are already operating inside the repository's development workflow**:

| Component | File | Current Purpose |
|---|---|---|
| `@claude` agent | `.github/workflows/at-claude.yml` | Responds to `@claude` mentions from maintainers and collaborators in issues and PRs |
| PR automation bots | `.github/workflows/bots.yml` | Auto-labels PR size, categorizes PRs (bug/feature/docs/chore/dependency), conducts AI-powered code reviews |
| CI pipeline | `.github/workflows/ci.yml` | Lint, typecheck, test (5 Python versions × 3 install variants), 100% coverage, docs build, release |
| Agent knowledge base | `AGENTS.md`, `agent_docs/`, directory-level `AGENTS.md` files | Hierarchical onboarding and coding guidelines for AI agents |
| Claude configuration | `.claude/settings.json`, `.claude/skills/` | Pre-approved tools and reusable agent skills |
| Gemini configuration | `.gemini/config.yaml` | Agent-specific behavior flags |

All four GitHub primitives are already in active use for AI:

| Primitive | Current Use |
|---|---|
| **GitHub Actions** | Runs Claude Code for PR review, issue categorization, `@claude` agent execution |
| **Git** | Stores the monorepo, test cassettes (VCR recordings), coverage data |
| **GitHub Issues** | Structured YAML templates for bug reports, feature requests, questions; `@claude` triggers |
| **GitHub Secrets** | `ANTHROPIC_API_KEY` for Claude Code, CI tokens, Cloudflare/Algolia/Twitter API keys |

### The Gap

The gap from current state to full Githubification is **not** introducing AI to the repository — AI agents are already here. The gap is **redirecting AI from serving developers to serving users**:

| Current | Githubified |
|---|---|
| `@claude` responds to maintainer mentions | An agent responds to **any authorized user** opening an issue |
| Claude reviews PRs against coding standards | An agent answers user questions about using Pydantic AI |
| No persistent session state across issues | Session state committed to git, conversations resume across issues |
| Agent serves the development workflow | Agent serves the user community |

---

## Githubification Strategy

Based on the [five strategies](https://github.com/japer-technology/githubification) and this repo's characteristics, the recommended approach is **Composition-Plus-Extension**: build the Githubification agent **with Pydantic AI itself**, extending the existing AI agent infrastructure already deployed in the repository.

### Why This Strategy

1. **The framework IS the runtime.** Pydantic AI's `Agent` class provides the agent loop, tool execution, output validation, and retry logic. The Githubification layer only needs to define configuration, tools, and dependencies — the framework does the rest.

2. **The agent demonstrates the product.** A Pydantic AI Githubification agent is a working example of the framework it serves. When a user asks "How do tools work?", the agent's own tools demonstrate the answer.

3. **Infrastructure already exists.** The `at-claude.yml` workflow is functionally a proto-Githubification pattern — it triggers on issue comments, authenticates the caller, sets up the development environment, executes an AI agent, and posts responses. The Githubification layer extends this, it does not replace it.

4. **Lightweight runtime.** Pydantic AI needs `uv` and Python — both available on every GitHub Actions runner. The entire development environment installs with `make install`. No Docker, no persistent servers, no GPU.

---

## How It Would Work

### The Lifecycle Pipeline

```
User opens an Issue (e.g., "How do I add a custom model provider?")
    → GitHub Actions workflow triggers on `issues.opened` or `issue_comment.created`
    → Workflow authenticates the user (OWNER / MEMBER / COLLABORATOR or configurable)
    → 🚀 reaction posted to indicate processing
    → `uv sync` installs Pydantic AI and dependencies
    → Python script constructs a Pydantic AI Agent with:
        - GitHub context as dependencies (issue number, repository, user info)
        - Repository docs as instructions (loaded selectively from agent_docs/)
        - Tools for reading files, searching code, running examples
        - Structured output type for validated responses
    → `agent.run()` handles the LLM interaction
    → Validated output posted as Issue comment
    → Session state committed to git
    → 👍 reaction posted on success
```

### The Agent Contract

```python
from pydantic_ai import Agent, RunContext
from pydantic import BaseModel

class GitHubContext(BaseModel):
    """Dependencies injected from the GitHub Actions environment."""
    issue_number: int
    repository: str
    commenter: str
    issue_title: str
    issue_body: str
    labels: list[str]

class AgentResponse(BaseModel):
    """Structured output — validated before posting to the issue."""
    reply: str
    references: list[str] = []

agent = Agent(
    'anthropic:claude-sonnet-4-6',  # Configurable via GitHub Secrets / env var
    deps_type=GitHubContext,
    output_type=AgentResponse,
    instructions='You are the Pydantic AI assistant...',
)

@agent.tool
async def search_docs(ctx: RunContext[GitHubContext], query: str) -> str:
    """Search the Pydantic AI documentation."""
    ...

@agent.tool
async def read_source(ctx: RunContext[GitHubContext], file_path: str) -> str:
    """Read a source file from the repository."""
    ...

@agent.tool
async def run_example(ctx: RunContext[GitHubContext], example_name: str) -> str:
    """Run a Pydantic AI example and return the output."""
    ...
```

### Required New Files

| File | Purpose |
|---|---|
| `.github/workflows/githubification-agent.yml` | The Githubification workflow — triggers on `issues.opened` and `issue_comment.created`, authenticates, runs the agent |
| `.githubification/agent.py` | The Pydantic AI agent — `Agent` construction, tools, dependencies, output types |
| `.githubification/state/issues/` | Issue-to-session mapping (e.g., `42.json` → session file) |
| `.githubification/state/sessions/` | Conversation transcripts (JSONL) |
| `.githubification/README.md` | Documentation for the Githubification layer |

### Workflow Skeleton

```yaml
name: Githubification Agent

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]

permissions:
  contents: write
  issues: write

jobs:
  agent:
    if: >
      contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'),
        github.event.comment.author_association ||
        github.event.issue.author_association)
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v6

      - uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Install dependencies
        run: uv sync --extra anthropic

      - name: Run agent
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          ISSUE_TITLE: ${{ github.event.issue.title }}
          ISSUE_BODY: ${{ github.event.issue.body || github.event.comment.body }}
          COMMENTER: ${{ github.event.sender.login }}
        run: uv run python .githubification/agent.py

      - name: Commit state
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .githubification/state/
          git diff --cached --quiet || git commit -m "Save session state for issue #${{ github.event.issue.number }}"
          git push
```

---

## Mapping to the Githubification Playbook

The [lesson consolidation](https://github.com/japer-technology/githubification) identifies universal patterns across all Githubified repositories. Here is how each pattern maps to this repo:

### 1. The Lifecycle Pipeline

| Step | Implementation |
|---|---|
| **Guard** | Workflow authorization — check `author_association` against allowed roles |
| **Indicate** | Post 🚀 reaction on the issue to show the agent is working |
| **Execute** | `uv run python .githubification/agent.py` — construct the Agent, call `agent.run()`, post the reply |
| **Commit** | `git add .githubification/state/ && git commit && git push` |

### 2. Fail-Closed Security

The `at-claude.yml` workflow already implements the security model:
- **Author association check** — only `OWNER`, `MEMBER`, `COLLABORATOR` can trigger the agent
- **Config file protection** — fork PRs that modify `AGENTS.md`, `CLAUDE.md`, or `.claude/` are rejected
- **Permission scoping** — workflow permissions limited to `contents: write` and `issues: write`
- **Concurrency controls** — one agent run per issue at a time

These patterns transfer directly to the Githubification workflow.

### 3. Issue-Driven Conversation

```
Issue #N → .githubification/state/issues/N.json → .githubification/state/sessions/<timestamp>.jsonl
```

When a user comments on issue #N weeks later, the agent loads the linked session file and resumes with full context. Pydantic AI's `message_history` parameter on `agent.run()` supports this natively — pass the prior conversation as message history and the agent continues where it left off.

### 4. Git as Memory

Session transcripts, issue mappings, and agent decisions are committed to `.githubification/state/`. Every interaction is versioned, auditable, and reversible via `git revert`.

### 5. The Four Primitives

| Primitive | Current | Githubified |
|---|---|---|
| **Actions** | CI/CD + Claude Code for dev | + Issue-driven agent for users |
| **Git** | Source code + test cassettes | + Session state and agent memory |
| **Issues** | Bug reports, features, questions | + Conversational AI interface |
| **Secrets** | `ANTHROPIC_API_KEY` + CI tokens | Same — already configured |

---

## Advantages of This Repo for Githubification

### 1. AI Infrastructure Pre-Exists

Every other Githubification case study starts from zero. This repo already has:
- AI agent workflows running in GitHub Actions (`at-claude.yml`, `bots.yml`)
- LLM API keys in GitHub Secrets (`ANTHROPIC_API_KEY`)
- Security patterns for AI agent triggering (association checks, config protection)
- A full development environment that installs with `make install`

### 2. The Framework Is the Agent Runtime

Pydantic AI's `Agent` class handles:
- The agent loop (LLM calls → tool execution → output validation → retry)
- Structured output validation via Pydantic models
- Dependency injection via `RunContext`
- Provider abstraction (`'anthropic:claude-sonnet-4-6'`, `'openai:gpt-4o'`, etc.)
- Message history for multi-turn conversations
- Streaming for real-time output

The Githubification layer defines *what* the agent does. The framework handles *how*.

### 3. Type-Safety as Reliability

A Pydantic AI Githubification agent's outputs are Pydantic-validated. If the LLM produces malformed output, the framework retries automatically. This means the agent will **never post a malformed response** to an Issue.

### 4. Provider Abstraction as Configuration

```python
# Any of these work identically — change via env var:
agent = Agent('anthropic:claude-sonnet-4-6')
agent = Agent('openai:gpt-4o')
agent = Agent('google-gla:gemini-2.0-flash')
agent = Agent('groq:llama-3.3-70b-versatile')
```

Switching providers requires changing one environment variable — not installing different packages or rewriting code.

### 5. Hierarchical Knowledge Enables Selective Context

The existing `AGENTS.md` hierarchy (root → `agent_docs/` → directory-level files) enables the Githubification agent to load context relevant to the user's specific question rather than a monolithic document:

| User Question | Context Loaded |
|---|---|
| "How do I add a model provider?" | `models/AGENTS.md`, `agent_docs/api-design.md` |
| "How do I write tests?" | `tests/AGENTS.md`, `agent_docs/index.md` |
| "How does dependency injection work?" | `docs/dependencies.md`, source code examples |
| "What's the graph library for?" | `pydantic_graph/` README, `docs/graph.md` |

### 6. Self-Referencing Agent

The Githubification agent is **built with Pydantic AI to explain Pydantic AI**. Its own source code is a working example of the framework — dependency injection, tools, structured output, provider abstraction — all in use.

---

## Comparison with GMI

[GitHub Minimum Intelligence](https://github.com/japer-technology/github-minimum-intelligence) provides the reference implementation for the Native Githubification strategy. Here is how a Pydantic AI Githubification compares:

| Dimension | GMI | Pydantic AI Githubification |
|---|---|---|
| **Runtime** | Bun + TypeScript (`pi-coding-agent`) | Python + `uv` (Pydantic AI `Agent`) |
| **Agent engine** | pi-mono | Pydantic AI framework |
| **Installation** | Copy one workflow, run once | Add workflow + agent script |
| **State format** | JSONL sessions + JSON issue mappings | Same pattern, Pydantic AI message history |
| **LLM providers** | OpenAI, Anthropic, Google, xAI, DeepSeek, Mistral, Groq, OpenRouter | All of the above + Cohere, Bedrock, Hugging Face, Ollama, and many more |
| **Output validation** | Unstructured text | Pydantic model validation with automatic retry |
| **Tool system** | pi-mono tools (read, grep, find, ls, edit) | Pydantic AI `@agent.tool` decorator with typed arguments |
| **Memory** | Git-committed JSONL | Git-committed state + Pydantic AI `message_history` |
| **Personality** | Hatch system (guided personality creation) | Configurable instructions |
| **Security** | Workflow authorization + DEFCON system | Workflow authorization + existing config protection |

---

## Ranking in the Githubification Winners

Based on the [Githubification winners ranking](https://github.com/japer-technology/githubification), this repo is ranked **#12** with status "AI-Agent-Operated". The lesson:

> *When AI agents already operate in the repo, the gap to Githubification is redirecting AI from developers to users.*

The path to climbing the ranking:

1. **Current state (AI-Agent-Operated):** AI agents review PRs, categorize issues, respond to `@claude` mentions — serving developers.
2. **Next step (Preparation):** Add the `.githubification/` folder with agent script and state directories.
3. **Target state (Fully Githubified):** The issue-driven agent is live — users interact with Pydantic AI through GitHub Issues, sessions persist in git, the agent is built with the framework it serves.

---

## Implementation Phases

### Phase 1 — Foundation
- [ ] Create `.githubification/` directory structure
- [ ] Write the `agent.py` script using Pydantic AI's `Agent` class
- [ ] Define `GitHubContext` dependencies and `AgentResponse` output type
- [ ] Add basic tools: `search_docs`, `read_source`, `read_examples`
- [ ] Create the `githubification-agent.yml` workflow

### Phase 2 — Session Lifecycle
- [ ] Implement issue-to-session mapping (`state/issues/N.json`)
- [ ] Implement session persistence (JSONL transcripts in `state/sessions/`)
- [ ] Wire Pydantic AI's `message_history` to load prior sessions on continued conversations
- [ ] Add git commit/push step to the workflow for state persistence

### Phase 3 — Intelligence
- [ ] Implement selective context loading from the `AGENTS.md` hierarchy
- [ ] Add tools for running code examples from `examples/`
- [ ] Add tools for searching the full documentation (`docs/`)
- [ ] Add a tool for checking test coverage and running targeted tests

### Phase 4 — Polish
- [ ] Add 🚀 / 👍 / 👎 reaction indicators on issues
- [ ] Add concurrency controls (one agent run per issue)
- [ ] Add configurable provider selection via environment variable
- [ ] Write `.githubification/README.md` with setup instructions
- [ ] Add tests for the Githubification agent

---

## Summary

This repository is **already halfway to Githubification**. The four GitHub primitives are active for AI, the security patterns are established, the development environment installs cleanly on GitHub Actions runners, and the framework itself provides the agent runtime. The remaining work is:

1. **Add an issue-driven workflow** that triggers on `issues.opened` and `issue_comment.created`
2. **Build a Pydantic AI agent** with GitHub dependencies, documentation tools, and structured output
3. **Implement session lifecycle** with git-committed state for persistent conversations
4. **Redirect the AI from developers to users** — extending the existing `@claude` pattern to serve the community

The result: a repository where users interact with Pydantic AI by opening GitHub Issues, get validated and contextual responses from an agent built with the framework itself, and every conversation is versioned in git. No local installation required. GitHub is the runtime.
