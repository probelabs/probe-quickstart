# Visor Quickstart

**Visor** is a YAML-first framework for building AI assistants. Define skills, tools, and knowledge in a single file — Visor handles intent classification, skill activation, and response generation.

This quickstart gives you a working assistant you can run in under a minute.

## Quick Start

```bash
# Clone and enter
git clone https://github.com/probelabs/visor-quickstart.git
cd visor-quickstart

# Set up environment
cp .env.example .env
# Edit .env — uncomment and set ANTHROPIC_API_KEY (or another provider)

# Run it
npx -y @probelabs/visor@latest run assistant.yaml --message "What can you do?"
```

## What Just Happened?

When you sent that message, Visor ran this pipeline:

1. **Intent classification** — determined this is a `chat` request
2. **Skill selection** — matched the `capabilities` skill based on "what can you do"
3. **Knowledge injection** — added the skill's knowledge block to the AI's context
4. **Response generation** — the AI answered using the injected context

All of this is defined in `assistant.yaml` — open it and read along.

## Try These

Each message activates a different skill:

```bash
# capabilities skill — inline knowledge
npx -y @probelabs/visor@latest run assistant.yaml --message "What can you help me with?"

# visor-guide skill — knowledge loaded from docs/visor-overview.md via {% readfile %}
npx -y @probelabs/visor@latest run assistant.yaml --message "How do skills work in Visor?"

# code-explorer skill — workflow tool (code-talk)
npx -y @probelabs/visor@latest run assistant.yaml --message "Show me what's in assistant.yaml"

# engineer skill — requires code-explorer, so both activate
npx -y @probelabs/visor@latest run assistant.yaml --message "Add a comment to the top of README.md"
```

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

## Run Tests

```bash
npx -y @probelabs/visor@latest test assistant.yaml
```

Tests verify that intent classification routes to the correct skills. See the `tests` section at the bottom of `assistant.yaml`.

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

- [Visor documentation](https://github.com/probelabs/visor) — full reference
- [visor-ee workflows](https://github.com/probelabs/visor-ee) — the assistant engine this quickstart imports
- Questions? Open an issue or contact hello@probelabs.com

## License

MIT
