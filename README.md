# Visor Quickstart

Build AI assistants in minutes with a single YAML file.

## 2-Minute Quick Start

```bash
# 1. Clone and enter
git clone https://github.com/probelabs/visor-quickstart.git
cd visor-quickstart

# 2. Set up environment
cp .env.example .env

# 3. Add your API key (get one free at https://console.anthropic.com)
#    Edit .env and set: ANTHROPIC_API_KEY=sk-ant-your-key-here

# 4. Run it!
npx -y @probelabs/visor@latest run assistant.yaml --message "What can you help me with?"
```

That's it! You should see a response from your assistant.

---

## The Complete Example

Here's everything you need - this is what `assistant.yaml` contains:

```yaml
version: "1.0"

# ==============================================================================
# IMPORTS - Required: loads the assistant workflow engine from visor-ee
# ==============================================================================
# This single import provides:
#   - Automatic intent classification (routes user requests)
#   - Skill activation (picks which tools/knowledge to use)
#   - MCP server wiring (connects to external tools)
#   - Built-in guidelines (output formatting, error handling)
#
# For offline use: download the file and use imports: ["./assistant.yaml"]
imports:
  - https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/assistant.yaml

# Slack config (remove this section if using CLI only)
slack:
  version: "v1"
  mentions: all      # Respond to all @mentions
  threads: required  # Only respond in threads, not channels

checks:
  chat:
    type: workflow
    workflow: assistant
    assume: ["true"]
    args:
      # ==============================================================================
      # QUESTION - Where the user's message comes from
      # ==============================================================================
      # - CLI mode: from --message argument
      # - Slack mode: from the message that triggered the bot
      # This "just works" - no configuration needed
      question: "{{ conversation.current.text }}"

      # ==============================================================================
      # SYSTEM PROMPT - Your assistant's identity (keep it short!)
      # ==============================================================================
      # Operational guidelines are built-in, so you only need to define:
      # - Who is this assistant?
      # - What domain does it specialize in?
      system_prompt: |
        You are a helpful AI assistant for software development.

      # ==============================================================================
      # INTENTS - High-level routing categories
      # ==============================================================================
      # These help the AI understand what TYPE of request this is.
      # Keep them broad - skills handle the specifics.
      intents:
        - id: chat
          description: General Q&A and conversation
        - id: code_help
          description: Questions about code or implementation
        - id: task
          description: Create, update, or execute something

      # ==============================================================================
      # SKILLS - The core building blocks of your assistant
      # ==============================================================================
      # Each skill is a self-contained capability that activates when needed.
      #
      # Skill fields:
      #   id          - Unique identifier (required)
      #   description - When to activate this skill (required) - BE SPECIFIC!
      #   knowledge   - Context injected into the AI's prompt (optional)
      #   requires    - Other skills to auto-activate, e.g., [code-explorer] (optional)
      #   tools       - Tools to enable (optional) - see "Tool Types" below
      #
      skills:
        # ----------------------------------------------------------------------
        # Example 1: Knowledge-only skill (no tools)
        # ----------------------------------------------------------------------
        # Just injects context into the prompt when activated
        - id: capabilities
          description: user asks what this assistant can do
          knowledge: |
            I can help with code exploration, engineering, and Q&A.

        # ----------------------------------------------------------------------
        # Example 2: Code exploration skill (workflow tool)
        # ----------------------------------------------------------------------
        # Uses the code-talk workflow to enable code search
        - id: code-explorer
          description: needs codebase exploration or code search
          knowledge: |
            Use the code exploration tool to search and understand code.
            Always provide file paths and line numbers.
          tools:
            code-explorer:
              workflow: code-talk    # Built-in workflow for code exploration
              inputs:
                # Projects define which code the AI can search
                # Supports local paths OR GitHub repos
                projects:
                  - name: my-repo        # Name the AI uses to reference this
                    path: .              # Local path (. = current directory)
                    description: Current repository
                  # More examples:
                  # - name: backend
                  #   path: /absolute/path/to/backend
                  # - name: frontend
                  #   repo: myorg/frontend    # GitHub repo
                  #   ref: main               # Branch/tag

        # ----------------------------------------------------------------------
        # Example 3: Skill with external MCP tool
        # ----------------------------------------------------------------------
        # Connects to external services via MCP protocol
        - id: jira
          description: references Jira tickets
          knowledge: |
            Use jira_get_issue to fetch tickets, jira_search for JQL queries.
          tools:
            # Tool name (can be anything)
            jira:
              # MCP server via command (most common)
              command: uvx
              args: ["mcp-atlassian"]
              env:
                JIRA_URL: "${JIRA_URL}"
                JIRA_API_TOKEN: "${JIRA_API_TOKEN}"
              # Limit which methods the AI can call (security)
              allowedMethods: [jira_get_issue, jira_search]

        # ----------------------------------------------------------------------
        # Example 4: Skill with dependencies
        # ----------------------------------------------------------------------
        # When this skill activates, it also activates its dependencies
        - id: engineer
          description: user wants code changes or a PR created
          requires: [code-explorer]  # Auto-activates code-explorer too!
          tools:
            engineer:
              # Workflow tool - calls another workflow as a tool
              workflow: engineer
              inputs: {}
```

---

## How Long Until I Can Chat?

| Time | What You Get |
|------|--------------|
| **2 min** | Basic assistant running (CLI) |
| **5 min** | Customized system_prompt for your domain |
| **15 min** | Add code explorer + your repositories |
| **30 min** | Add Slack integration |
| **1 hour** | Add Jira/Zendesk/external integrations |

---

## Tool Types Explained

The `tools` field in skills supports three types:

```yaml
tools:
  # TYPE 1: Command (external MCP server)
  # Runs a command that speaks MCP protocol
  jira:
    command: uvx                    # Command to run
    args: ["mcp-atlassian"]         # Arguments
    env:                            # Environment variables
      JIRA_URL: "${JIRA_URL}"
    allowedMethods: [jira_search]   # Optional: limit available methods

  # TYPE 2: Workflow (internal workflow as tool)
  # Calls another workflow defined in imports
  engineer:
    workflow: engineer              # Workflow ID from imports
    inputs: {}                      # Inputs to pass

  # TYPE 3: Built-in tool
  # Uses a tool built into Visor
  scheduler:
    tool: schedule                  # Built-in tool name
```

---

## Loading External Config Files

For larger configurations, split into multiple files:

```yaml
# assistant.yaml
skills:
  expression: "loadConfig('config/skills.yaml')"
```

```yaml
# config/skills.yaml - can use {% readfile %} for knowledge
- id: jira
  description: Jira ticket access
  knowledge: |
    {% readfile "docs/jira-guide.md" %}
  tools:
    jira:
      command: uvx
      args: ["mcp-atlassian"]
```

---

## Troubleshooting

### "No AI provider configured"
Your API key isn't set. Check:
```bash
cat .env | grep API_KEY
```
Make sure you have `ANTHROPIC_API_KEY=sk-ant-...` (not commented out with #)

### "Unable to fetch workflow" or network errors
The imports URL must be accessible. Check your internet connection, or download the workflow locally:
```bash
curl -o workflows/assistant.yaml https://raw.githubusercontent.com/probelabs/visor-ee/master/workflows/assistant.yaml
```
Then change imports to: `imports: ["./workflows/assistant.yaml"]`

### "conversation.current.text is undefined"
You forgot `--message` in CLI mode:
```bash
# Wrong:
visor run assistant.yaml

# Right:
visor run assistant.yaml --message "Hello"
```

### Skill doesn't activate for my question
The skill's `description` field must match what the user is asking. Make it more specific:
```yaml
# Too vague - might not activate:
- id: jira
  description: ticket stuff

# Better - clear trigger:
- id: jira
  description: user mentions Jira, ticket IDs like PROJ-123, or needs ticket information
```

### Slack bot not responding
1. Check both tokens are set in `.env`: `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN`
2. Make sure you're mentioning the bot in a thread (not a channel message)
3. Run with `--slack` flag: `visor run assistant.yaml --slack`

---

## Examples

| File | Description | Use When |
|------|-------------|----------|
| `assistant.yaml` | Full-featured assistant | Starting point for real projects |
| `examples/minimal.yaml` | ~25 lines, just chat | Learning the basics |
| `examples/with-jira.yaml` | Jira integration | Adding external tools |
| `examples/multi-repo.yaml` | Multiple code repos | Exploring large codebases |

---

## Configuration Reference

### Skill Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique identifier |
| `description` | Yes | When to activate (be specific!) |
| `knowledge` | No | Context injected into prompt |
| `requires` | No | Array of skill IDs to auto-activate |
| `tools` | No | Tools to enable when skill activates |

### Tool Fields

| Field | Description |
|-------|-------------|
| `command` | Command to run (for external MCP servers) |
| `args` | Command arguments |
| `env` | Environment variables (use `${VAR}` syntax) |
| `allowedMethods` | Limit which methods AI can call |
| `workflow` | Workflow ID (for workflow-as-tool) |
| `inputs` | Inputs to pass to workflow |
| `tool` | Built-in tool name |

### Code Exploration (code-talk workflow)

For code exploration, use a workflow tool with `workflow: code-talk`:

| Input Field | Description |
|-------------|-------------|
| `projects` | Array of repositories to search |
| `projects[].name` | Identifier for the AI |
| `projects[].path` | Local filesystem path |
| `projects[].repo` | GitHub repo (e.g., `owner/repo`) |
| `projects[].ref` | Branch or tag (default: main) |
| `projects[].description` | What this repo contains |
| `architecture` | Architecture documentation (optional) |
| `docs_repo` | Documentation repository (optional) |

---

## Built-in Features

The assistant workflow includes:

- **Automatic routing** - Classifies intent and activates relevant skills
- **Tool merging** - Same-named tools merge their `allowedMethods`
- **Dependency expansion** - `requires` auto-activates other skills
- **Built-in guidelines** - Output formatting, URL handling, error messages
- **Thread context** - Slack history automatically considered

---

## Support

- [Visor Documentation](https://github.com/probelabs/visor)
- [visor-ee Workflows](https://github.com/probelabs/visor-ee)
- Questions? Open an issue or contact hello@probelabs.com

## License

MIT
