# Example Knowledge Document

This is an example of a knowledge file that can be loaded into a skill.

## Usage

Reference this file in your skill definition:

```yaml
skills:
  - id: my-skill
    description: ...
    knowledge: |
      {% readfile "docs/example-knowledge.md" %}
```

## Your Content Here

Add your domain-specific knowledge, guidelines, or instructions here.
The content will be injected into the AI's context when this skill is activated.

### Tips

- Keep it focused and relevant
- Use markdown for structure
- Include specific instructions for the AI
- Add examples when helpful
