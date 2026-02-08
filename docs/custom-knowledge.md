## Custom Knowledge Template

This is an example knowledge document. Create your own documents in this folder
and reference them in your assistant's prompt using tags.

### How to Use This Pattern

1. Create a markdown file in `docs/` with your knowledge
2. Add a tag to `workflows/my-intent-router.yaml`
3. Reference it in `assistant.yaml` prompt section

### Example: Domain-Specific Knowledge

```markdown
## My Feature Instructions

When handling requests about this feature:
- Important guideline 1
- Important guideline 2

### Tools Available
- tool_name: what it does

### Common Scenarios
1. Scenario A: how to handle
2. Scenario B: how to handle
```

### Example: Integration Instructions

```markdown
## Jira Integration

You have access to Jira via the atlassian MCP server.

### Available Methods
- `jira_get_issue(key)` - Get issue details
- `jira_search(jql)` - Search issues
- `jira_create_issue(...)` - Create new issue

### Usage Notes
- Always include ticket IDs in PR titles
- Link PRs to tickets when creating
```

### Injecting Knowledge

In your `assistant.yaml` prompt:

```yaml
{% if outputs['route-intent'].tags contains 'my_feature' %}
{% readfile "docs/my-feature.md" %}
{% endif %}
```

This keeps prompts focused by only including relevant knowledge.
