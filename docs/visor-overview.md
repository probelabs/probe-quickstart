# Visor Overview

Visor is a YAML-first framework for building AI assistants. You define your assistant's identity, skills, and tools in a single configuration file, and Visor handles intent classification, skill activation, tool orchestration, and response generation.

## Key Concepts

### Assistant Configuration
Every assistant starts with a YAML file that declares:
- **System prompt** — the assistant's identity and domain
- **Intents** — broad categories of user requests (e.g., chat, code_help, task)
- **Skills** — capabilities that activate based on what the user asks

### Skills
A skill is a self-contained capability. Each skill has:
- **`id`** — unique identifier
- **`description`** — when to activate (the AI uses this to match user requests)
- **`knowledge`** (optional) — context injected into the AI's prompt when activated
- **`requires`** (optional) — other skill IDs to co-activate
- **`tools`** (optional) — external tools the AI can call

Skills activate automatically — Visor classifies the user's intent and selects relevant skills based on their descriptions.

### Intents
Intents are broad routing categories like `chat`, `code_help`, or `task`. They help Visor understand the *type* of request before selecting specific skills. Keep intents general; skills handle specifics.

### Tools
Skills can include tools of three types:

1. **Workflow tools** — call another Visor workflow (e.g., `code-talk` for code exploration, `engineer` for code changes)
   ```yaml
   tools:
     code-explorer:
       workflow: code-talk
       inputs:
         projects:
           - name: my-repo
             path: .
   ```

2. **Command tools (MCP)** — run an external MCP server process
   ```yaml
   tools:
     jira:
       command: uvx
       args: ["mcp-atlassian"]
       env:
         JIRA_URL: "${JIRA_URL}"
       allowedMethods: [jira_get_issue, jira_search]
   ```

3. **Built-in tools** — tools provided by the Visor runtime
   ```yaml
   tools:
     scheduler:
       tool: schedule
   ```

### Knowledge and `{% readfile %}`
Knowledge is markdown text injected into the AI's context when a skill activates. You can inline it or load from a file:

```yaml
# Inline knowledge
knowledge: |
  Here is context the AI will see.

# Knowledge from file
knowledge: |
  {% readfile "docs/my-guide.md" %}
```

The `{% readfile %}` directive loads file contents at runtime, keeping your YAML clean and your knowledge docs easy to maintain separately.

## Imports
Assistants import workflow definitions — typically the `assistant.yaml` workflow from `visor-ee`, which provides the intent classification and skill activation engine:

```yaml
imports:
  - https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/assistant.yaml
```

## Deployment Modes

- **CLI** — run locally for testing and development:
  ```
  npx -y @probelabs/visor@latest run assistant.yaml --message "Hello"
  ```
- **Slack** — connect to a Slack workspace with socket mode:
  ```
  npx -y @probelabs/visor@latest run assistant.yaml --slack
  ```

## Runtime Flow

When a user sends a message:
1. **Intent classification** — Visor determines the request type
2. **Skill selection** — relevant skills activate based on their descriptions
3. **Dependency expansion** — `requires` fields pull in additional skills
4. **Tool + knowledge injection** — activated skills' tools and knowledge are added to the AI's context
5. **Response generation** — the AI responds using the assembled context and tools

## Testing
Define test cases directly in your YAML to verify routing and skill activation:
```
npx -y @probelabs/visor@latest test assistant.yaml
```
