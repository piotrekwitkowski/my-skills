---
name: gh-push
description: Push code to GitHub without git push. Uses gh CLI and GitHub API to create/update files, commits, branches, and refs entirely via API calls. Use when git push is unavailable, blocked, or undesirable.
---

# gh-push — Push Code to GitHub Without `git push`

Push commits, files, and branches to GitHub using only `gh api` calls. No SSH keys, no git remotes, no `git push` required — just a valid `gh` auth token.

## Commit Author

All commits MUST include the author field:
```json
{"name": "Piotrek", "email": "noreply@piotrekwitkowski.com"}
```

## Prerequisites

```bash
# Must be authenticated
gh auth status

# Must know your repo (owner/repo format)
gh repo view --json nameWithOwner -q .nameWithOwner
```

### Required Token Scopes

| Scope | When needed |
|-------|-------------|
| `repo` | All API operations (default) |
| `workflow` | Creating/modifying files under `.github/workflows/` |

If you get `404 Not Found` when pushing workflow files, the token is missing the `workflow` scope:

```bash
gh auth refresh -s workflow
```

## Method 1: Single File Create/Update (Contents API)

Best for: updating 1-3 files. Simple but limited to one file per call.

### Create a new file

```bash
gh api repos/OWNER/REPO/contents/path/to/file.txt \
  -X PUT \
  -f message="Add file.txt" \
  -f content="$(base64 < file.txt)" \
  -f branch="main"
```

### Update an existing file

You MUST provide the current file's SHA (acts as optimistic concurrency lock):

```bash
# Get current SHA
SHA=$(gh api repos/OWNER/REPO/contents/path/to/file.txt -q .sha)

# Update
gh api repos/OWNER/REPO/contents/path/to/file.txt \
  -X PUT \
  -f message="Update file.txt" \
  -f content="$(base64 < file.txt)" \
  -f sha="$SHA" \
  -f branch="main"
```

### Delete a file

```bash
SHA=$(gh api repos/OWNER/REPO/contents/path/to/file.txt -q .sha)

gh api repos/OWNER/REPO/contents/path/to/file.txt \
  -X DELETE \
  -f message="Delete file.txt" \
  -f sha="$SHA" \
  -f branch="main"
```

## Method 2: Multi-File Commit (Git Data API)

Best for: pushing multiple files in a single atomic commit. This is the **primary method** for real work.

**IMPORTANT**: This method does NOT work on completely empty repos (GitHub returns 409). Bootstrap with Method 1 first, then use Method 2 for subsequent commits.

### Step-by-step workflow

```bash
OWNER="owner"
REPO="repo"
BRANCH="main"

# 1. Get the current commit SHA and tree SHA for the branch
COMMIT_SHA=$(gh api repos/$OWNER/$REPO/git/ref/heads/$BRANCH -q .object.sha)
TREE_SHA=$(gh api repos/$OWNER/$REPO/git/commits/$COMMIT_SHA -q .tree.sha)

# 2. Create blobs for each file
BLOB1=$(gh api repos/$OWNER/$REPO/git/blobs \
  -f content="$(base64 < src/app.ts)" \
  -f encoding="base64" \
  -q .sha)

BLOB2=$(gh api repos/$OWNER/$REPO/git/blobs \
  -f content="$(base64 < src/utils.ts)" \
  -f encoding="base64" \
  -q .sha)

# 3. Create a new tree with the file changes
#    base_tree preserves all existing files not listed here
#    NOTE: Use echo + pipe pattern — heredocs with --input - can be unreliable
NEW_TREE=$(echo '{"base_tree":"'"$TREE_SHA"'","tree":[{"path":"src/app.ts","mode":"100644","type":"blob","sha":"'"$BLOB1"'"},{"path":"src/utils.ts","mode":"100644","type":"blob","sha":"'"$BLOB2"'"}]}' | gh api repos/$OWNER/$REPO/git/trees --input - -q .sha)

# 4. Create the commit
NEW_COMMIT=$(echo '{"message":"feat: add app and utils","tree":"'"$NEW_TREE"'","parents":["'"$COMMIT_SHA"'"],"author":{"name":"Piotrek","email":"noreply@piotrekwitkowski.com"}}' | gh api repos/$OWNER/$REPO/git/commits --input - -q .sha)

# 5. Update the branch ref to point to the new commit
gh api repos/$OWNER/$REPO/git/refs/heads/$BRANCH \
  -X PATCH \
  -f sha="$NEW_COMMIT"
```

### File modes reference

| Mode     | Meaning          |
|----------|------------------|
| `100644` | Normal file      |
| `100755` | Executable file  |
| `040000` | Subdirectory     |
| `120000` | Symlink          |

### Deleting files in a multi-file commit

To delete a file, create a tree entry with `sha` set to `null`:

```bash
NEW_TREE=$(gh api repos/$OWNER/$REPO/git/trees \
  --input - -q .sha <<EOF
{
  "base_tree": "$TREE_SHA",
  "tree": [
    {
      "path": "src/deprecated.ts",
      "mode": "100644",
      "type": "blob",
      "sha": null
    }
  ]
}
EOF
)
```

## Method 3: Create and Push to a New Branch

```bash
OWNER="owner"
REPO="repo"
SOURCE_BRANCH="main"

# Get the SHA to branch from
SOURCE_SHA=$(gh api repos/$OWNER/$REPO/git/ref/heads/$SOURCE_BRANCH -q .object.sha)

# Create the new branch
gh api repos/$OWNER/$REPO/git/refs \
  -f ref="refs/heads/my-feature" \
  -f sha="$SOURCE_SHA"

# Now push files to the new branch using Method 1 or 2 with branch="my-feature"
```

### Force-update a branch (equivalent to `git push --force`)

```bash
gh api repos/$OWNER/$REPO/git/refs/heads/$BRANCH \
  -X PATCH \
  -f sha="$NEW_COMMIT" \
  -F force=true
```

## Method 4: Create a PR with File Changes (Single Step)

If the goal is to open a PR with changes, you can combine branch creation + file push + PR creation:

```bash
OWNER="owner"
REPO="repo"
BASE="main"
HEAD="my-feature"

# 1. Create branch (from Method 3)
BASE_SHA=$(gh api repos/$OWNER/$REPO/git/ref/heads/$BASE -q .object.sha)
gh api repos/$OWNER/$REPO/git/refs -f ref="refs/heads/$HEAD" -f sha="$BASE_SHA"

# 2. Push files to branch (Method 1 or 2)
# ...

# 3. Create PR
gh pr create --repo $OWNER/$REPO --base $BASE --head $HEAD \
  --title "My feature" --body "Description here"
```

## Helper: Push All Staged Changes

A reusable pattern to push everything in `git diff --staged` via API:

```bash
#!/usr/bin/env bash
set -euo pipefail

OWNER="owner"
REPO="repo"
BRANCH="main"
MESSAGE="commit message here"

# Get current state
COMMIT_SHA=$(gh api repos/$OWNER/$REPO/git/ref/heads/$BRANCH -q .object.sha)
TREE_SHA=$(gh api repos/$OWNER/$REPO/git/commits/$COMMIT_SHA -q .tree.sha)

# Build tree entries from staged files
TREE_ENTRIES="["
FIRST=true
for FILE in $(git diff --cached --name-only); do
  if [ "$FIRST" = true ]; then FIRST=false; else TREE_ENTRIES+=","; fi

  if [ -f "$FILE" ]; then
    # File exists — create blob
    BLOB_SHA=$(gh api repos/$OWNER/$REPO/git/blobs \
      -f content="$(base64 < "$FILE")" \
      -f encoding="base64" -q .sha)
    TREE_ENTRIES+="{\"path\":\"$FILE\",\"mode\":\"100644\",\"type\":\"blob\",\"sha\":\"$BLOB_SHA\"}"
  else
    # File deleted
    TREE_ENTRIES+="{\"path\":\"$FILE\",\"mode\":\"100644\",\"type\":\"blob\",\"sha\":null}"
  fi
done
TREE_ENTRIES+="]"

# Create tree, commit, update ref
NEW_TREE=$(echo "{\"base_tree\":\"$TREE_SHA\",\"tree\":$TREE_ENTRIES}" | \
  gh api repos/$OWNER/$REPO/git/trees --input - -q .sha)

NEW_COMMIT=$(echo "{\"message\":\"$MESSAGE\",\"tree\":\"$NEW_TREE\",\"parents\":[\"$COMMIT_SHA\"],\"author\":{\"name\":\"Piotrek\",\"email\":\"noreply@piotrekwitkowski.com\"}}" | \
  gh api repos/$OWNER/$REPO/git/commits --input - -q .sha)

gh api repos/$OWNER/$REPO/git/refs/heads/$BRANCH -X PATCH -f sha="$NEW_COMMIT"

echo "Pushed $NEW_COMMIT to $BRANCH"
```

## Decision Guide

| Scenario | Method |
|----------|--------|
| Update 1-3 files | Method 1 (Contents API) |
| Atomic multi-file commit | Method 2 (Git Data API) |
| Push to new branch | Method 3 + Method 1 or 2 |
| Full PR workflow | Method 4 |
| Push staged changes | Helper script |
| Force push | Method 3 force-update variant |

## Known Gotchas

- **Empty repos reject Git Data API** — blobs, trees, and commits all return `409 Git Repository is empty`. You MUST bootstrap with the Contents API (Method 1) first to create the initial commit, then switch to Git Data API for subsequent pushes.
- **`workflow` scope required for `.github/workflows/`** — both REST and GraphQL APIs return 404/403 without it. This applies to Contents API, Git Data API, and `createCommitOnBranch` alike. Run `gh auth refresh -s workflow` to add it.
- **Contents API requires SHA for updates** — always fetch the current SHA first, or you'll get a 409 conflict.
- **base64 encoding** — all file content must be base64-encoded when using the blobs or contents API. Use `base64 < file` (macOS) or `base64 -w0 < file` (Linux).
- **Branch must exist** before pushing to it — create it first via the refs API.
- **Rate limits** — each blob creation is one API call. For repos with 100+ files changing, batch work or consider using the GraphQL `createCommitOnBranch` mutation instead.
- **Max file size** — the Contents API has a 100MB limit per file. The Blobs API has a 100MB limit too but is more reliable for large files.
- **Piping JSON to `gh api`** — use `echo '...' | gh api ... --input -` rather than heredocs with `--input -`, which can be unreliable with variable interpolation.

## GraphQL Alternative: `createCommitOnBranch`

For pushing multiple file changes in a single API call (no blob creation needed).

**Requires `workflow` token scope if touching `.github/workflows/` files.**

```bash
gh api graphql -f query='
mutation {
  createCommitOnBranch(input: {
    branch: {
      repositoryNameWithOwner: "OWNER/REPO"
      branchName: "main"
    }
    message: { headline: "Update files via API" }
    expectedHeadOid: "CURRENT_COMMIT_SHA"
    fileChanges: {
      additions: [
        { path: "src/app.ts", contents: "BASE64_CONTENT_HERE" }
        { path: "src/utils.ts", contents: "BASE64_CONTENT_HERE" }
      ]
      deletions: [
        { path: "src/old-file.ts" }
      ]
    }
  }) {
    commit { url oid }
  }
}'
```

This is the most efficient method for multi-file pushes — one API call, no intermediate blobs or trees. Does NOT work on empty repos.
