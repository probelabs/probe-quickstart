## Engineer Tool

The `engineer` tool makes actual code changes, creates PRs, and implements features using Claude Code.

### When to Use

Use the engineer tool **ONLY** when the user explicitly asks to:
- Create a PR or pull request
- Implement a feature or fix a bug
- Make code changes or modifications
- Add new endpoints, functions, or components
- Debug and fix issues (not just investigate)

**Do NOT use for:**
- Investigating or understanding code (use code-helper only)
- Answering questions about how something works
- Finding where something is implemented
- Research or exploration tasks

**Ask yourself:** Did the user explicitly ask me to CREATE, IMPLEMENT, FIX, or CHANGE code? If not, don't use engineer.

### Workflow: Explore First, Then Engineer

**Always explore first with code-helper**, then call engineer:

#### Step 1: Explore with `code-helper`

First, call `code-helper` to understand what needs to be changed:
- Which files need modification
- Existing patterns to follow
- Related code to consider

Note the `checkout_projects` paths from the result.

#### Step 2: Implement with `engineer`

Then call `engineer` with:
- `task`: Your specific implementation task (be detailed!)
- `context`: Include the relevant findings from code-helper
- `projects`: List of checkout paths from code-helper

### Example Flow

User: "Create a PR to add rate limit headers to API responses"

1. Call code-helper:
   ```
   code-helper("Where are API response headers set? What's the pattern for adding new headers?")
   ```

2. Get response with:
   - answer.text explaining the header middleware
   - references to relevant files
   - checkout_projects: ["/tmp/visor/my-project"]

3. Call engineer with:
   ```
   task: "Add X-RateLimit-Limit and X-RateLimit-Remaining headers to API responses"
   context: "Headers are added in src/middleware/response.go using the AddHeader method..."
   projects: ["/tmp/visor/my-project"]
   ```

### Important Notes

- **Pass context**: Don't make engineer re-discover what code-helper found
- **Be specific**: Detailed tasks get better results
- **Multi-repo**: If changes span multiple repos, engineer will handle all of them
- **PRs**: Engineer will create PRs automatically when done
- **Verification**: Engineer will verify the build passes before creating PRs

### Handling Results

**NEVER call the engineer tool more than once for the same task!**

When engineer returns success:
1. Summarize the result for the user (PR link, what was changed, status)
2. Complete your response - do NOT call engineer again
3. If the user wants additional changes, wait for their next message
