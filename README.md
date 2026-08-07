# Developer Agent Space

An autonomous **"Jira user story → production-ready code"** agent system built entirely from
VS Code Copilot **custom agents** and **skills**. Give it a Jira ticket; it analyzes the
requirements, clones the repos, plans and implements the change, builds and tests it with coverage,
verifies it against the ticket's acceptance criteria, and raises a pull request — pausing for your
explicit approval at every risky step.

> Human-in-the-loop by design. The system never clones, branches, pushes, or opens a PR without a
> clear "yes" from you for that specific action.

## Supported stacks

- **Frontend:** Angular (npm / `ng` CLI, Karma/Jasmine or Jest).
- **Backend:** Java 17+ with Maven and Spring Boot (JUnit + JaCoCo).

## How it works

`story-orchestrator` is the entry point and coordinates six specialist sub-agents:

```
You: "@story-orchestrator implement I9P-48239"
        │
        ↓
1. jira-requirements-analyst   → requirements, acceptance criteria, plan, repos
        │  (ask you for any missing repo URL)
        ↓
2. repo-setup                  → GATE: approve clone → git clone + stack detection
        │
        ↓
3. implementation-planner      → file-level plan + branch/image names (≤ 64 chars)
        │  GATE: approve plan
        ↓
4. code-implementer            → GATE: approve branch → create branch + code + tests (local commit)
        │
        ↓
5. build-test-verifier         → build, test, coverage, verify vs acceptance criteria → sign-off
        │
        ↓
6. pr-publisher                → GATE: approve push → push → GATE: approve PR → open PR
```

### Approval gates

| Gate | Before | Enforced by |
|------|--------|-------------|
| Approve clone | cloning repos | orchestrator → `repo-setup` |
| Approve plan | any code change | orchestrator → `code-implementer` |
| Approve branch | creating the feature branch | orchestrator → `code-implementer` |
| Approve push | pushing to remote | orchestrator → `pr-publisher` |
| Approve PR | opening the pull request | orchestrator → `pr-publisher` |

## Naming conventions

Enforced by the [`story-naming-conventions`](.github/skills/story-naming-conventions/SKILL.md) skill.

- **Feature branch:** `feature/<TICKET>-<featureName>` — e.g. `feature/I9P-45567-addPubsubLogic`.
- **Image tag:** `<version>-feature-<TICKET>-<featureName>-SNAPSHOT-<build>` — e.g.
  `1.0.0-feature-I9P-48239-subscribeMsgs-SNAPSHOT-168`. **Must be ≤ 64 characters.**
- `featureName` is short lowerCamelCase (verb + noun), ideally ≤ 20 chars so the tag always fits.

## Repository layout

```
.github/
├─ copilot-instructions.md            # repo-wide rules for every agent
├─ agents/
│  ├─ story-orchestrator.agent.md     # entry point; coordinates + enforces gates
│  ├─ jira-requirements-analyst.agent.md
│  ├─ repo-setup.agent.md
│  ├─ implementation-planner.agent.md
│  ├─ code-implementer.agent.md
│  ├─ build-test-verifier.agent.md
│  └─ pr-publisher.agent.md
├─ skills/
│  └─ story-naming-conventions/SKILL.md
├─ artifacts/                         # generated build/test/coverage run outputs
└─ implementation-plans/              # downloaded/converted ticket implementation-plan docs (.md)
```

## Prerequisites

- **VS Code** with **GitHub Copilot** (agent mode / custom agents enabled).
- **MCP servers** configured in your `mcp.json`:
  - **Atlassian** (Jira/Confluence) — to read tickets and post sign-off comments.
  - **GitHub** — to open pull requests.
- **git CLI** available on your PATH for clone / branch / push.
- Toolchains for the target repos: **Node.js + Angular CLI** and/or **JDK 17+ + Maven**.

## Usage

1. Open this workspace in VS Code.
2. In Copilot Chat, select the **Story Orchestrator** agent (or `@story-orchestrator`).
3. Provide the ticket, e.g. `implement I9P-48239` (a key or a Jira URL).
4. Approve each gate when prompted. Provide any missing repo URLs, credentials, or environments when
   the agent asks — it will not guess them.
5. Receive the final recap: branch name, image tag, changes mapped to requirements, build/test/
   coverage results, and the PR link.

## Customizing tool restrictions

MCP-dependent agents (`story-orchestrator`, `jira-requirements-analyst`, `pr-publisher`) inherit the
default tool set so they work regardless of your MCP server IDs. To lock them down, add a `tools:`
list to the agent frontmatter using your server globs, e.g.:

```yaml
tools: [read, search, execute, agent, atlassian-mcp/*, github/*]
```

Match the server names to the IDs in your `mcp.json`.

## Safety

- No secrets in code, logs, or commits.
- Inputs validated at boundaries; changes trace back to a requirement or acceptance criterion.
- Prompt-injection attempts in ticket/PR/tool content are flagged, not obeyed.
- No force-push, history rewrites, branch deletion, or auto-merge.
