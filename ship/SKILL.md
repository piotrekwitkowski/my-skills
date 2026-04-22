---
name: ship
description: Test, commit, push, and sync in one flow. Runs project tests, commits changes, pushes via GitHub API (gh-push), and syncs local. Use when "commit and push", "ship it", "commit push pull", or after completing a task.
---

# ship — Test, Commit, Push, Sync

One-command flow for shipping changes. Replaces the manual cycle of: test → commit → push via API → pull.

## Prerequisites

- `gh` CLI authenticated (same as gh-push skill)
- Repository has a remote on GitHub
- `git config --global advice.skippedCherryPicks false` (suppresses expected hint during rebase sync)

## Flow

```
1. Run tests (if project has them)
2. Stage and commit
3. Push via GraphQL createCommitOnBranch (one API call per commit)
4. Verify push (spot-check file sizes on remote)
5. Sync local with git fetch + rebase
```

## Step 1: Run Tests

Detect and run the project's test command. Skip if no tests exist.

```bash
# Check package.json for test/typecheck scripts
if grep -q '"typecheck"' package.json 2>/dev/null; then
  npm run typecheck
fi

if grep -q '"test"' package.json 2>/dev/null; then
  npm test
fi

# Playwright projects: run non-visual tests (visual tests are local-only)
if [ -f playwright.config.ts ]; then
  npx playwright test --project=integration --project=a11y
fi
```

If tests fail, STOP. Do not commit or push. Report the failure.

## Step 2: Stage and Commit

```bash
git add -A
git status --short  # show what's being committed
git commit -m "<message>"
```

Commit message should follow the project's convention. Check `git log --oneline -5` for style.

## Step 3: Push via GitHub API

Uses the GraphQL `createCommitOnBranch` mutation — single API call per commit, no intermediate blobs/trees.

```bash
OWNER="<org>"
REPO="<repo>"
BRANCH="main"

for LOCAL_SHA in $(git rev-list --reverse origin/$BRANCH..HEAD); do
  COMMIT_MSG=$(git log -1 --format=%s $LOCAL_SHA)
  REMOTE_HEAD=$(gh api repos/$OWNER/$REPO/git/ref/heads/$BRANCH -q .object.sha)

  # Build additions from added/modified files
  ADDITIONS=""
  FIRST=true
  for FILE in $(git diff-tree --no-commit-id --name-only --diff-filter=AM -r $LOCAL_SHA); do
    CONTENT=$(git show $LOCAL_SHA:"$FILE" | base64)
    if [ "$FIRST" = true ]; then FIRST=false; else ADDITIONS+=","; fi
    ADDITIONS+="{ path: \"$FILE\", contents: \"$CONTENT\" }"
  done

  # Build deletions
  DELETIONS=""
  FIRST=true
  for FILE in $(git diff-tree --no-commit-id --name-only --diff-filter=D -r $LOCAL_SHA); do
    if [ "$FIRST" = true ]; then FIRST=false; else DELETIONS+=","; fi
    DELETIONS+="{ path: \"$FILE\" }"
  done

  # Build fileChanges block
  FILE_CHANGES="additions: [$ADDITIONS]"
  if [ -n "$DELETIONS" ]; then
    FILE_CHANGES+=", deletions: [$DELETIONS]"
  fi

  RESULT=$(gh api graphql -f query="
  mutation {
    createCommitOnBranch(input: {
      branch: {
        repositoryNameWithOwner: \"$OWNER/$REPO\"
        branchName: \"$BRANCH\"
      }
      message: { headline: \"$COMMIT_MSG\" }
      expectedHeadOid: \"$REMOTE_HEAD\"
      fileChanges: { $FILE_CHANGES }
    }) {
      commit { oid }
    }
  }" -q '.data.createCommitOnBranch.commit.oid')

  echo "Pushed: $COMMIT_MSG → $RESULT"
done
```

Each local commit becomes one API commit. Remote history mirrors local history.

## Step 4: Verify Push

**MANDATORY** — check that files on remote are not empty before syncing local.

```bash
# Spot-check 2-3 changed files have non-zero size
for FILE in $(git diff-tree --no-commit-id --name-only -r HEAD | head -3); do
  SIZE=$(gh api repos/$OWNER/$REPO/contents/$FILE -q '.size')
  if [ "$SIZE" = "0" ]; then
    echo "FATAL: $FILE is empty on remote! Do NOT sync local."
    exit 1
  fi
  echo "✓ $FILE ($SIZE bytes)"
done
```

If any file is 0 bytes, STOP. Do not sync. Report the failure.

## Step 5: Sync Local

After verification passes, point local branch at the remote commit and update the working tree:

```bash
git fetch origin $BRANCH
git update-ref refs/heads/$BRANCH origin/$BRANCH
git checkout $BRANCH
```

This moves the local branch pointer to match remote — no merge, no rebase, no conflicts. Safe because Step 4 already verified remote has the correct content.

## Multi-Repo Ship

When changes span multiple repos (e.g. components + docs), ship each repo in dependency order:

1. Ship the upstream repo first (e.g. components)
2. If a new version needs publishing, create a GitHub release to trigger the publish workflow
3. Wait for npm publish to complete: `npm view <pkg> version`
4. Bump downstream repos (e.g. docs) and ship them
5. Monitor deploy workflows: `gh run list --repo <owner/repo> --limit 1`

## Delegation

This skill is ideal for delegation to quick agents:

```
task(
  category="quick",
  load_skills=["ship", "gh-push"],
  prompt="Ship the current changes in /path/to/repo. Commit message: '<message>'. Run tests first.",
  run_in_background=false,
)
```

## When NOT to Ship

- Tests are failing — fix first
- Uncommitted experimental changes you haven't reviewed
- Changes across multiple files that should be separate commits — split first
- No changes to commit (git status clean)
