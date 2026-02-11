# Probe Quickstart

Half your team's time goes to finding things that already exist — in code, in tickets, in docs. Engineers context-switch between GitHub, Jira, Slack, and Confluence just to answer "where is this implemented?" or "what's the status of that feature?"

**Probe** fixes this. It's an AI agent that connects to your codebase, tickets, docs, and tools — then answers questions, explores code, makes changes, and automates workflows. You define everything in YAML. No custom code, no vendor lock-in, any LLM provider.

This quickstart gives you a working assistant in under a minute. Here's the entire config:

```yaml
version: "1.0"

# Import the assistant engine (intent classification, skill activation, tools)
imports:
  - https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/assistant.yaml

# Slack config (remove if CLI-only)
slack:
  version: "v1"
  mentions: all
  threads: required

checks:
  chat:
    type: workflow
    workflow: assistant
    assume: ["true"]
    args:
      question: "{{ conversation.current.text }}"

      system_prompt: |
        You are a Probe Labs assistant helping developers understand and build
        AI assistants with Visor. You can explore code, explain Visor concepts,
        and demonstrate how skills, intents, and tools work together.

      # Intents — broad request categories for routing
      intents:
        - id: chat
          description: General Q&A, follow-up questions, small talk
        - id: code_help
          description: Questions about code, implementation, or architecture
        - id: task
          description: Create, update, or execute something

      # Skills — capabilities that activate based on what the user asks
      skills:
        # Inline knowledge (no tools)
        - id: capabilities
          description: user asks what this assistant can do
          knowledge: |
            I can explain Visor, explore code across repos, and make changes via PRs.

        # Knowledge loaded from file via {% readfile %}
        - id: visor-guide
          description: questions about how Visor works, skills, intents, tools, or YAML config
          knowledge: |
            {% readfile "docs/visor-overview.md" %}

        # Workflow tool — code search across repos
        - id: code-explorer
          description: needs codebase exploration, code search, or implementation details
          tools:
            code-explorer:
              workflow: code-talk
              inputs:
                projects:
                  - name: quickstart
                    path: .
                  - name: visor
                    repo: probelabs/visor

        # Skill with dependency — auto-activates code-explorer
        - id: engineer
          description: user wants code changes, a PR, or a feature implemented
          requires: [code-explorer]
          tools:
            engineer:
              workflow: engineer
              inputs: {}

        # MCP tool (uncomment + set JIRA_* in .env)
        # - id: jira
        #   description: user mentions Jira or ticket IDs like PROJ-123
        #   tools:
        #     jira:
        #       command: uvx
        #       args: ["mcp-atlassian"]
        #       env:
        #         JIRA_URL: "${JIRA_URL}"
        #         JIRA_API_TOKEN: "${JIRA_API_TOKEN}"
        #       allowedMethods: [jira_get_issue, jira_search, jira_create_issue]
```

That's it. One file. Skills, tools, knowledge, and routing — all declared in YAML.

## Quick Start

```bash
git clone https://github.com/probelabs/visor-quickstart.git
cd visor-quickstart
cp .env.example .env
# Edit .env — uncomment and set ANTHROPIC_API_KEY (or another provider)
```

Launch the interactive TUI (recommended):
```bash
npx -y @probelabs/visor@latest run assistant.yaml --tui
```

The TUI gives you a full chat interface with real-time visibility into what the assistant is doing. Press **Shift+Tab** to cycle through three views:

- **Chat** — the conversation with your assistant
- **Logs** — runtime logs showing intent classification, skill activation, and tool calls
- **Trace** — detailed execution trace for debugging pipelines and workflows

Or send a single message from the command line with `--message`:
```bash
npx -y @probelabs/visor@latest run assistant.yaml --message "What can you do?"
```

## What Just Happened?

When you sent a message, Probe ran this pipeline:

1. **Intent classification** — determined the request type (`chat`, `code_help`, or `task`)
2. **Skill selection** — matched relevant skills based on their `description` fields
3. **Dependency expansion** — skills with `requires` pulled in other skills automatically
4. **Knowledge + tool injection** — activated skills' knowledge and tools were added to the AI's context
5. **Response** — the AI answered using the assembled context, calling tools if needed

Everything above is defined in `assistant.yaml`. Open it and read along.

## Try These

Each message activates a different skill. Try them in the TUI (`--tui`) or pass them with `--message`:

```bash
# capabilities — inline knowledge, no tools
npx -y @probelabs/visor@latest run assistant.yaml --message "What can you help me with?"

# visor-guide — knowledge loaded from docs/visor-overview.md
npx -y @probelabs/visor@latest run assistant.yaml --message "How do skills work in Visor?"

# code-explorer — searches code across repos
npx -y @probelabs/visor@latest run assistant.yaml --message "Show me what's in assistant.yaml"

# engineer — requires code-explorer, so both activate
npx -y @probelabs/visor@latest run assistant.yaml --message "Add a comment to the top of README.md"
```

The `--message` flag is useful for scripting, CI pipelines, or quick one-off questions. For interactive exploration, use `--tui` instead.

## Customize It

**Change the identity** — edit `system_prompt` in `assistant.yaml`:
```yaml
system_prompt: |
  You are an assistant for Acme Corp's engineering team.
```

**Add your repos** — add entries to the `code-explorer` skill's `projects`:
```yaml
projects:
  - name: backend
    path: /path/to/backend
    description: Backend API server
  - name: frontend
    repo: myorg/frontend
    ref: main
    description: React frontend
```

**Add a knowledge skill** — load docs from a file:
```yaml
- id: onboarding
  description: questions about onboarding, setup, or getting started
  knowledge: |
    {% readfile "docs/onboarding-guide.md" %}
```

**Add an MCP tool** — uncomment the `jira` skill in `assistant.yaml` and set credentials in `.env`.

## Connect to Slack

Turn this assistant into a Slack bot your whole team can talk to.

**1. Create a Slack app** at [api.slack.com/apps](https://api.slack.com/apps):
- Enable **Socket Mode** (Settings → Socket Mode → Enable)
- Under **OAuth & Permissions**, add these Bot Token Scopes:
  `app_mentions:read`, `channels:history`, `groups:history`, `im:history`, `mpim:history`, `chat:write`, `reactions:read`, `reactions:write`, `im:read`, `im:write`
- Under **Event Subscriptions**, subscribe to bot events:
  `app_mention`, `message.channels`, `message.groups`, `message.im`, `message.mpim`
- Install the app to your workspace

**2. Grab two tokens** and add them to `.env`:
```bash
# Bot token (OAuth & Permissions → Bot User OAuth Token)
SLACK_BOT_TOKEN=xoxb-your-bot-token

# App token (Basic Information → App-Level Tokens → create one with connections:write scope)
SLACK_APP_TOKEN=xapp-your-app-token
```

**3. Run it:**
```bash
npx -y @probelabs/visor@latest run assistant.yaml --slack
```

Mention your bot in any Slack thread and it will respond. The `slack` section in `assistant.yaml` controls behavior — `threads: required` means it only responds in threads, not top-level channel messages.

For a full walkthrough, see the [Build a Slack Bot](https://probelabs.com/docs/guides/slack-bot) guide.

## Test & Lint

**Run tests** to verify that intent classification routes to the correct skills:

```bash
npx -y @probelabs/visor@latest test assistant.yaml
```

Each test case in the `tests` section of `assistant.yaml` sends a mock message and asserts which intents and skills activate. Use this to catch routing regressions when you add or edit skills.

**Lint your configuration** to catch YAML syntax errors, missing fields, and invalid references before running:

```bash
npx -y @probelabs/visor@latest lint assistant.yaml
```

Lint checks for common issues like misspelled skill IDs in `requires`, missing `description` fields, and invalid workflow references. Run it after every config change — it's instant and saves debugging time.

## Lint

Validate your configuration for schema errors and missing fields:

```bash
npx -y @probelabs/visor@latest validate assistant.yaml
```

## Examples

| File | What it shows |
|------|---------------|
| `assistant.yaml` | Full-featured assistant (start here) |
| `examples/minimal.yaml` | Simplest possible assistant (~25 lines) |
| `examples/with-jira.yaml` | External MCP tool integration |
| `examples/multi-repo.yaml` | Code exploration across multiple repos |

## Troubleshooting

### "No AI provider configured"
Your API key isn't set. Check `.env` has an uncommented key:
```bash
grep API_KEY .env
```

### "Unable to fetch workflow" or network errors
The imports URL must be accessible. To work offline:
```bash
curl -o workflows/assistant.yaml https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/assistant.yaml
```
Then change imports to: `imports: ["./workflows/assistant.yaml"]`

### "conversation.current.text is undefined"
You forgot `--message` in CLI mode:
```bash
npx -y @probelabs/visor@latest run assistant.yaml --message "Hello"
```

### Skill doesn't activate
Make the skill's `description` more specific about when it should trigger:
```yaml
# Too vague:
description: ticket stuff

# Better:
description: user mentions Jira, ticket IDs like PROJ-123, or needs ticket information
```

### Slack bot not responding
1. Check both tokens in `.env`: `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN`
2. Make sure you're mentioning the bot in a thread (not a channel message)
3. Run with: `npx -y @probelabs/visor@latest run assistant.yaml --slack`

## Next Steps

- [Probe Labs](https://probelabs.com) — the platform
- [Visor documentation](https://github.com/probelabs/visor) — workflow engine reference
- [visor-ee workflows](https://github.com/probelabs/visor-ee) — the assistant engine this quickstart imports
- Questions? Open an issue or contact hello@probelabs.com

## License

MIT
