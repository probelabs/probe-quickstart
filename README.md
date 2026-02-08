# Visor Quickstart

A quickstart template for building AI assistants with [Visor](https://github.com/probelabs/visor).

This repository provides a generic, domain-agnostic example you can customize to build your own AI-powered assistant with:
- Intent classification and routing
- Dynamic MCP tool selection
- Knowledge embedding via docs
- Multi-repo code exploration
- Slack integration

## Quick Start

### 1. Clone this repository

```bash
git clone https://github.com/probelabs/visor-quickstart.git
cd visor-quickstart
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env with your API keys
```

### 3. Run the assistant

```bash
# Run as CLI chat
npx -y @probelabs/visor@latest run assistant.yaml --event manual

# Run as Slack bot
npx -y @probelabs/visor@latest run assistant.yaml --slack
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Message                             │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Intent Router                               │
│  Classifies: intent (chat, code_help, task) + tags              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Dynamic Tool Selection                         │
│  Based on tags → loads MCP servers, workflows-as-tools          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Knowledge Embedding                           │
│  Based on tags → injects relevant docs into prompt              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       AI Response                                │
│  Uses tools + knowledge to generate answer                       │
└─────────────────────────────────────────────────────────────────┘
```

## Key Concepts

### 1. Intent Router

The intent router classifies user messages into intents and tags:

```yaml
route-intent:
  type: workflow
  config: workflows/my-intent-router.yaml
  workflow_inputs:
    question: "{{ outputs['ask'].text }}"
```

**Intents** determine the high-level flow (chat, code exploration, task execution).

**Tags** modify behavior by enabling specific tools or injecting knowledge.

See `workflows/my-intent-router.yaml` for customization.

### 2. Dynamic MCP Tool Selection

Tools are loaded conditionally based on tags using `ai_mcp_servers_js`:

```javascript
ai_mcp_servers_js: |
  const servers = {};
  const tags = outputs['route-intent']?.tags || [];

  // Add Jira when 'jira' tag is present
  if (tags.includes('jira')) {
    servers.atlassian = {
      command: "uvx",
      args: ["mcp-atlassian"],
      env: {
        JIRA_URL: "${JIRA_URL}",
        JIRA_USERNAME: "${JIRA_USERNAME}",
        JIRA_API_TOKEN: "${JIRA_API_TOKEN}"
      }
    };
  }

  // Add code exploration when 'codebase' tag is present
  if (tags.includes('codebase')) {
    servers['code-helper'] = {
      workflow: 'code-helper',
      inputs: { question: outputs['ask']?.text }
    };
  }

  return servers;
```

**External MCP servers** have a `command` property (e.g., `uvx mcp-atlassian`).

**Workflow tools** have a `workflow` property (e.g., `workflow: 'code-helper'`).

### 3. Knowledge Embedding

Inject contextual knowledge based on tags using `{% readfile %}`:

```yaml
prompt: |
  You are an AI assistant.

  {% if outputs['route-intent'].tags contains 'capabilities' %}
  {% readfile "docs/capabilities.md" %}
  {% endif %}

  {% if outputs['route-intent'].tags contains 'codebase' %}
  {% readfile "docs/code-exploration.md" %}
  {% endif %}

  User: {{ outputs['ask'].text }}
```

This pattern keeps prompts focused and relevant to the current request.

### 4. Workflow-as-Tool Pattern

Expose workflows as MCP tools for the AI to call:

```javascript
servers['my-workflow'] = {
  workflow: 'my-workflow-id',
  inputs: {
    param1: 'value1',
    param2: outputs['ask']?.text
  }
};
```

The AI can then call this workflow like any other tool.

## Customization Guide

### Adding a New Intent

1. Edit `workflows/my-intent-router.yaml`:

```yaml
intents:
  - id: my_new_intent
    description: |
      When to use this intent:
      - Specific trigger conditions
      - Example phrases
```

2. Add routing in `assistant.yaml`:

```yaml
on_success:
  transitions:
    - when: "output.intent === 'my_new_intent'"
      to: my-new-handler
```

### Adding a New Tag

1. Edit `workflows/my-intent-router.yaml`:

```yaml
tags:
  - id: my_new_tag
    description: Apply when user needs this capability
```

2. Use the tag in `assistant.yaml`:

```yaml
# In ai_mcp_servers_js
if (tags.includes('my_new_tag')) {
  servers['my-tool'] = { ... };
}

# In prompt
{% if outputs['route-intent'].tags contains 'my_new_tag' %}
{% readfile "docs/my-new-capability.md" %}
{% endif %}
```

### Adding External MCP Tools

```javascript
if (tags.includes('my_tag')) {
  servers.my_mcp_server = {
    command: "npx",
    args: ["-y", "my-mcp-package"],
    env: {
      API_KEY: "${MY_API_KEY}"
    },
    // Optional: limit which methods are available
    allowedMethods: ["method1", "method2"]
  };
}
```

### Adding Knowledge Documents

1. Create a new file in `docs/`:

```markdown
# docs/my-feature.md

## My Feature Instructions

When handling requests about this feature:
- Guideline 1
- Guideline 2
```

2. Reference it in the prompt:

```yaml
{% if outputs['route-intent'].tags contains 'my_feature' %}
{% readfile "docs/my-feature.md" %}
{% endif %}
```

## Examples

| Example | Description |
|---------|-------------|
| `examples/minimal-chat.yaml` | Simplest possible chat setup |
| `examples/slack-integration.yaml` | Slack bot configuration |
| `examples/code-assistant.yaml` | Full code exploration setup |

## Advanced Features (visor-ee)

This quickstart uses simplified versions of workflows. For production use, consider upgrading to [visor-ee](https://github.com/probelabs/visor-ee) which includes:

| Feature | Quickstart | visor-ee |
|---------|------------|----------|
| Intent Router | Basic intents + tags | Explicit markers (#tag, %intent), confidence scoring, multi-intent |
| Code Explorer | Single project | Multi-repo with architecture-aware routing |
| Integrations | Manual setup | Pre-built Jira, Confluence, Zendesk, GitHub |

To use visor-ee workflows:

```yaml
imports:
  - https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/intent-router.yaml
  - https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/code-talk.yaml
```

## Directory Structure

```
visor-quickstart/
├── README.md                      # This file
├── assistant.yaml                 # Main workflow config
├── .env.example                   # Environment variables template
│
├── docs/                          # Knowledge documents
│   ├── capabilities.md            # What the assistant can do
│   ├── code-exploration.md        # Code tool instructions
│   └── custom-knowledge.md        # Example domain knowledge
│
├── workflows/                     # Reusable workflow components
│   ├── my-intent-router.yaml      # Intent router configuration
│   ├── code-helper.yaml           # Code exploration wrapper
│   └── engineer.yaml              # Code modification workflow
│
├── examples/                      # Example configurations
│   ├── minimal-chat.yaml          # Simplest chat setup
│   ├── slack-integration.yaml     # Slack bot example
│   └── code-assistant.yaml        # Full code exploration
│
└── tests/                         # Test cases
    └── cases/
        └── routing-tests.yaml     # Intent routing tests
```

## Debugging

### Enable debug mode

```bash
npx -y @probelabs/visor@latest run assistant.yaml --debug
```

### Use the logger check

```yaml
debug-flow:
  type: logger
  depends_on: [route-intent]
  message: |
    Intent: {{ outputs['route-intent'].intent }}
    Tags: {{ outputs['route-intent'].tags | json }}
```

### Check AI tool calls

Look for `ai_mcp_servers_js` output in debug logs to see which tools were loaded.

## Support

- [Visor Documentation](https://github.com/probelabs/visor)
- [visor-ee Workflows](https://github.com/probelabs/visor-ee)
- Questions? Open an issue or contact hello@probelabs.com

## License

MIT
