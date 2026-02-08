---
name: commit
description: Creates professional conventional commit messages based on git changes. Use when user asks to commit, create a commit message, or wants to save changes to git.
---

# Commit Message Generator

Generate professional commit messages following the Conventional Commits specification based **only** on actual git changes.

## Critical Rules

1. **ONLY analyze git changes** - Never base commit messages on conversation history or AI discussions
2. **Analyze actual diffs** - Use `git diff --staged` and `git diff` to see real changes
3. **Professional tone** - Focus on intent and impact, not implementation minutiae
4. **Follow Conventional Commits** - Use proper type, scope, and description format

## Conventional Commit Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Types (in priority order)

- `feat`: New feature for the user
- `fix`: Bug fix for the user
- `refactor`: Code restructuring without changing behavior
- `perf`: Performance improvements
- `docs`: Documentation changes
- `style`: Formatting, missing semicolons, etc. (no code change)
- `test`: Adding or refactoring tests
- `build`: Build system or external dependencies
- `ci`: CI configuration changes
- `chore`: Other changes that don't modify src or test files

### Scope (optional but recommended)

- Feature module name: `auth`, `coffees`, `roasters`, `preferences`
- Component type: `api`, `ui`, `db`, `templates`
- Keep it short and lowercase

### Description

- Lowercase, no period at end
- Imperative mood ("add" not "added" or "adds")
- Max 72 characters for first line
- Focus on **why** and **what impact**, not **how**

## Workflow

1. **Analyze staged  changes**
   ```bash
   git diff --staged
   ```
2. **Determine commit type and scope**
   - What is the primary purpose of this change?
   - Which feature/module is affected?
   - Is this a breaking change?

3. **Draft message focusing on intent**
   - Why was this change made?
   - What problem does it solve?
   - What behavior changes for users?

4. **Keep it concise**
   - First line should tell the complete story
   - Add body only if context is needed
   - Use bullet points in body for multiple changes

5. Let user modify and accept the commit message before doing any action
## Examples

### Good Commit Messages

```
feat(auth): add OIDC logout flow

Implements proper logout with token revocation and session cleanup
```

```
fix(coffees): prevent duplicate entries on rapid form submission

Add debouncing to form handler and check for existing entries
```

```
refactor(templates): consolidate route helpers into trait

Reduces code duplication across template files
```

```
perf(db): add index on user_id for coffee queries

Improves list page load time from 500ms to 50ms
```

### Bad Commit Messages (Don't do this)

```
fix: fixed bug
# Too vague - what bug? where?
```

```
feat(auth): implemented the authentication middleware using tower and added session management with redis backend and also refactored the user model
# Too detailed, should be multiple commits
```

```
Updated files
# No type, no scope, no useful information
```

```
WIP
# Not a commit message, use git stash instead
```


## Breaking Changes

For breaking changes, add an exclamation mark after type/scope and explain in footer:

```
feat(api)!: change coffee endpoint response format

BREAKING CHANGE: Coffee API now returns `roaster_name` instead of
`roaster` field. Update all API clients to use new field name.
```

## Co-authoring

Do not append a footer

## Don't Include in Commits

- Generated files (unless intentional)
- `.env` files or secrets
- Large binary files
- `node_modules` or `target/` directories
- Personal config files (`.vscode/`, `.idea/`)

Warn the user if any of these appear in staged changes.
