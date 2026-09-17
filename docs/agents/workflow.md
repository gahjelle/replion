# Workflow

How agents make changes in this repo.

## Rules

- **Never commit to `main`.** All changes land on `main` through pull requests.
- **One worktree per agent.** Do your work in your own git worktree under `.claude/worktrees/`, never in the main checkout, so that concurrent agents don't step on each other.
- **One branch per piece of work.** Create a fresh branch from the latest `main` in that worktree. Name it after the work, e.g. `issue-12-session-model` or `research/<topic>`.
- **Push and open a PR targeting `main`.** Use `gh pr create --base main`. Reference the issue the work resolves.
- **Clean up.** Remove your worktree once its branch is pushed.

## Exceptions

- Throwaway `research/<topic>` branches from wayfinder research tickets are pushed but not opened as PRs; the ticket links to the findings on the branch.
