---
name: review-pr
description: >
  Fetch and address GitHub PR review comments for the current branch. Trigger when: user asks to "address PR comments", "fix review feedback", "handle PR reviews", "review-pr", "address review", "fix PR comments", or any request to act on pull request review feedback. Also trigger when the user pastes a PR URL and asks to address its comments.
---

# Review PR: Address GitHub Review Comments

Fetch unresolved review comments on the current branch's PR, analyze what each reviewer is asking for, and make the code changes.

## Prerequisites

`gh` CLI must be installed and authenticated:

```bash
which gh
gh auth status
```

## Workflow

### Step 1: Get the PR for the current branch

```bash
gh pr view --json number,url,title,headRefName
```

If user provided a PR number or URL, use that instead:

```bash
gh pr view <number-or-url> --json number,url,title,headRefName
```

### Step 2: Extract owner and repo

```bash
gh repo view --json owner,name --jq '"\(.owner.login) \(.name)"'
```

### Step 3: Fetch unresolved review threads

Use GraphQL to get review threads with resolution status, file locations, and all comments:

```bash
gh api graphql -F owner='<owner>' -F repo='<repo>' -F pr='<number>' -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          isResolved
          isOutdated
          path
          line
          startLine
          comments(first: 50) {
            nodes {
              body
              author { login }
              createdAt
              diffHunk
            }
          }
        }
      }
    }
  }
}
'
```

Filter to threads where `isResolved: false` and `isOutdated: false`. These are the active, unresolved comments that need attention.

### Step 4: Process each thread

For each unresolved thread:

1. **Read the file** at `path` around the indicated `line`/`startLine`
2. **Read the full comment thread** — later replies may clarify, narrow, or withdraw the original request
3. **Categorize** the comment:
   - **Actionable code change** — reviewer requests a specific fix, refactor, rename, or improvement → make the change
   - **Nit / style** — minor style suggestion (naming, formatting) → make the change, these are quick
   - **Question / discussion** — reviewer is asking for context or debating an approach → skip, flag for the user to respond
   - **Already addressed** — the requested change is already present in the current code (comment may be stale even if thread isn't marked resolved) → skip, note it
4. **Make the fix** if actionable — edit the file at the relevant location
5. **Track** what was done and what was skipped

### Step 5: Summary

After processing all threads, report:

- **Addressed**: list of changes made, with file paths and what was changed
- **Skipped**: threads that need human response (questions, design discussions), with the reviewer's comment quoted
- **Already addressed**: threads where the code already reflects the requested change

## Handling Edge Cases

- **Thread with multiple comments**: read the entire thread — the last comment often supersedes or refines earlier ones
- **Conflicting reviews**: if two reviewers disagree, skip and flag for the user
- **Comments on deleted lines**: if `isOutdated` is false but the file/line no longer exists, skip and note it
- **Large refactor requests**: if a comment asks for a significant architectural change, describe what would be needed and ask the user before proceeding

## Keywords
review, PR, pull request, comments, feedback, address, fix, review comments, code review, unresolved, github
