---
name: gemini-review
description: >-
  This skill should be used when the user asks to "review my code with gemini",
  "gemini review", "cross-review my changes", "get a second opinion on my diff",
  "gemini code review", "have gemini check my code", "second model review",
  or invokes "/gemini-review". Delegates code review of the current git diff
  to Gemini CLI and synthesizes the findings into a severity-ranked report.
---

# Gemini Cross-Review

Perform a cross-model code review by sending the current git diff to Gemini CLI
and synthesizing the results back to the user.

## Why Cross-Review

Different AI models have different blind spots. Delegating review to a second
model catches issues that a single model might miss — especially concurrency
bugs, retain cycles, logic errors, and subtle security issues.

## Workflow

### Step 1: Gather Context

1. Run `git diff HEAD` to capture all staged and unstaged changes against the last commit
2. Run `git ls-files --others --exclude-standard` to detect untracked (new) files
3. For each untracked file, read its contents and format as a unified diff
   (prefix each line with `+`, use the header `--- /dev/null` / `+++ b/<file>`)
4. Combine the tracked diff and untracked file diffs into a single diff block
5. Check if a `CLAUDE.md` file exists in the project root
6. If `CLAUDE.md` exists, read its contents to use as coding standards context
7. Summarize the current conversation context — what the user has been working on,
   the intent behind the changes, and any relevant decisions made during the session.
   This summary helps Gemini review the diff with the right intent in mind.

If both the tracked diff and untracked file list are empty, inform the user there are
no changes to review and stop.

### Step 2: Build the Gemini Prompt

Construct a prompt for Gemini CLI with the following structure:

```
You are a senior code reviewer. Review the following git diff for:
- Bugs and logic errors
- Security vulnerabilities
- Concurrency and thread-safety issues
- Memory management problems (retain cycles, leaks)
- Code quality and readability issues
- Performance concerns

Context: The developer has been working on the following:
<context>
[conversation summary — what the user is building, the intent behind the changes,
key decisions made during the session]
</context>

[If CLAUDE.md exists, include:]
The project follows these coding standards — flag any violations:
<standards>
[CLAUDE.md contents]
</standards>

Here is the diff to review (includes tracked changes and any new untracked files):
<diff>
[combined diff output — tracked changes + untracked files]
</diff>

For each issue found, specify:
1. File and approximate location
2. Severity (critical / warning / suggestion)
3. Description of the issue
4. Suggested fix

If the code looks good, say so explicitly.
```

### Step 3: Send to Gemini CLI

Run the prompt through Gemini CLI using Bash:

```bash
gemini -p "<constructed prompt>"
```

**Important considerations:**
- The diff and CLAUDE.md may be large. If the combined prompt exceeds ~30,000 characters, truncate CLAUDE.md to its first 2,000 words and note this to the user.
- If `gemini` command fails (non-zero exit code), inform the user of the error and suggest checking `gemini` CLI authentication or installation. Do not retry.

### Step 4: Synthesize and Present

Do NOT simply dump Gemini's raw output. Instead:

1. Read Gemini's response
2. Filter out noise — remove generic advice, obvious false positives, or issues that contradict the project's CLAUDE.md standards
3. Group findings by severity (critical first, then warnings, then suggestions)
4. Present a synthesized review in this format:

```
## Cross-Review Results (via Gemini)

### Critical
- **file.swift:~42** — [description and fix]

### Warnings
- **file.swift:~15** — [description and fix]

### Suggestions
- **file.swift:~88** — [description and fix]

### Summary
[1-2 sentence overall assessment]
```

If Gemini found no issues, report that clearly:
```
## Cross-Review Results (via Gemini)
Gemini found no issues with the current changes.
```

## Error Handling

- **Empty diff and no untracked files**: Inform user, stop. Do not call Gemini.
- **Gemini CLI not found**: Tell user to install with `npm install -g @google/gemini-cli`
- **Gemini auth failure**: Tell user to run `gemini` once to re-authenticate
- **Gemini timeout or crash**: Report the error, move on. Do not retry.
