---
name: gemini-research
description: >-
  This skill should be used when the user asks to "research with gemini",
  "gemini search", "gemini research", "ask gemini about", "look up with gemini",
  "search the web for", "find current info on", "what's the latest on",
  or invokes "/gemini-research". Delegates web-grounded research to Gemini CLI
  using the --search flag and synthesizes the findings into the current
  conversation context. Useful for questions requiring fresh, up-to-date
  information from the web.
---

# Gemini Research

Delegate research tasks to Gemini CLI, leveraging its web-grounded search
capabilities, then synthesize and adapt the results to the current project context.

## Why Delegate Research to Gemini

Gemini CLI with `--search` provides web-grounded answers with access to current
documentation, release notes, and community discussions. This complements
Claude's strengths by offloading research that benefits from fresh web data.

## Workflow

### Step 1: Identify the Research Topic

The user provides a research topic as an argument to the skill. Examples:

- `/gemini-research how does Swift Testing parameterized tests work?`
- `/gemini-research what changed in iOS 26 for HealthKit?`
- `/gemini-research compare SwiftData vs CoreData for offline-first apps`

Extract the research topic from the user's message. If no topic is provided,
ask the user what to research.

### Step 2: Build the Gemini Prompt

Construct a research prompt that:

1. States the research question clearly
2. Asks for specific, actionable information
3. Requests code examples where relevant
4. Asks for sources/references

Prompt structure:

```
Research the following topic thoroughly. Provide specific, actionable information
with code examples where relevant. Include sources and references.

Topic: [user's research topic]

Provide:
1. A clear explanation of the topic
2. Code examples if applicable
3. Key considerations, gotchas, or best practices
4. Links to official documentation or authoritative sources
```

### Step 3: Send to Gemini CLI

Run the prompt through Gemini CLI with the `--search` flag for web grounding:

```bash
gemini -p "<constructed prompt>" --search
```

**Important considerations:**
- Always use the `--search` flag to enable web-grounded results
- If `gemini` command fails (non-zero exit code), inform the user and suggest
  checking authentication or installation. Do not retry.

### Step 4: Synthesize and Present

Do NOT dump Gemini's raw output. Instead:

1. Read Gemini's response carefully
2. Synthesize the information into a clear, structured answer
3. **Adapt to the current project** — rewrite any code examples to match the
   project's conventions. Check CLAUDE.md for project standards if available
   and adapt accordingly.
4. Highlight the most relevant and actionable points
5. Include source links when Gemini provides them

Present the synthesized research in this format:

```
## Research Results: [topic]

[Synthesized explanation, adapted to project context]

### Key Takeaways
- [Most important point 1]
- [Most important point 2]
- [Most important point 3]

### Code Example
[If applicable — adapted to project conventions]

### Sources
- [Link 1]
- [Link 2]
```

## Error Handling

- **No topic provided**: Ask the user what to research.
- **Gemini CLI not found**: Tell user to install with `npm install -g @google/gemini-cli`
- **Gemini auth failure**: Tell user to run `gemini` once to re-authenticate
- **Gemini timeout or crash**: Report the error, move on. Do not retry.
- **Gemini returns low-quality results**: Note this to the user and offer to
  research the topic using Claude's own knowledge as a fallback.
