---
description: Generate a commit message based on staged changes
argument-hint: [custom-context]
allowed-tools: Bash, Read, Grep, Glob
---

Generate a well-structured commit message for the current staged changes.

## Analysis Steps

1. **Check git status and diff**
```bash
git status
git diff --staged
```

2. **Review recent commits for style consistency**
```bash
git log --oneline -10
```

3. **Analyze changes**
   - Identify the type of change (feat, fix, refactor, docs, test, chore, style)
   - Determine the scope (which component/module is affected)
   - Understand the purpose and impact of changes

## Commit Message Format

Generate a commit message following conventional commits format:

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type
- **feat**: New feature
- **fix**: Bug fix
- **refactor**: Code refactoring (no functional changes)
- **docs**: Documentation changes
- **test**: Adding or updating tests
- **chore**: Maintenance tasks (deps, config, build)
- **style**: Code style changes (formatting, whitespace)

### Guidelines
- Subject: Concise summary (50 chars max), imperative mood ("add" not "added")
- Body: Explain the "why" not the "what" (wrap at 72 chars)
- Footer: Reference issues, breaking changes if any

### Example Output
```
feat(auth): add OAuth2 integration for social login

Implement Google and GitHub OAuth2 providers to allow users
to sign in with their existing accounts. This improves UX
and reduces friction in the signup process.

Closes #123
```

## Additional Context

If arguments provided: $ARGUMENTS

Use this context to refine the commit message.

## Output

Provide:
1. **Recommended commit message** - Ready to use
2. **Analysis summary** - Brief explanation of changes
3. **Alternative options** - If multiple valid approaches exist
