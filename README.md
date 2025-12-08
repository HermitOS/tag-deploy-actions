# 🚀 Git Tag Automation Actions

Make your nightly deployments *smart and effortless!*  
These two reusable GitHub Actions help you **detect new commits since your last deployment** and **update a moving tag** when your deploy succeeds — automatically, cleanly, and safely.

---

## 💡 What’s Inside

### 🕵️‍♂️ `tag-rollup`

Checks if there are any commits since your last deployment tag.

#### `tag-rollup` Inputs

| Name | Optional | Default | Description |
|------|-----------|----------|-------------|
| `tag` | ✅ | last-deploy | Tag name to check, e.g. `last-deploy` |
| `initial_as_changes` | ✅ | `true` | Treat the first run (when no tag exists) as “changes present” |

#### Outputs

| Name | Description |
|------|--------------|
| `has_changes` | `true` if HEAD is ahead of the tag |
| `base_tag` | The existing tag name if found |
| `ahead` | Number of commits since the tag |

---

### 🏷️ `tag-push`

Moves a tag to `HEAD` and pushes it back to your remote — great for marking a successful deploy.

#### `tag-push` Inputs

| Name | Optional  | Default | Description |
|------|-----------|----------|-------------|
| `tag` | ✅ | last-deploy | Tag name to move and push |
| `remote` | ✅ | `origin` | Remote name to push to |

---

## ✨ Example Usage

Here's how you can build a simple nightly workflow that checks for new commits, runs a deployment if needed, and updates your tag.

> **Important**: Always use `fetch-depth: 0` to get full history, and the actions will handle fetching tags properly.

### Option A - In one workflow action with two jobs

```yaml
name: Nightly Deploy
on:
  schedule:
    - cron: "5 0 * * *"
  workflow_dispatch: {}

permissions:
  contents: write

concurrency:
  group: nightly-rollup-${{ github.workflow }}-${{ github.ref_name }}
  cancel-in-progress: false

jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      has_changes: ${{ steps.rollup.outputs.has_changes }}
      base_tag:    ${{ steps.rollup.outputs.base_tag }}
      ahead:       ${{ steps.rollup.outputs.ahead }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: rollup
        uses: hermitos/tag-deploy-actions/tag-rollup@v2g-rollup@v2

  deploy:
    needs: check
    if: needs.check.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Build & Deploy
        run: |
          echo "Deploying changes (ahead=${{ needs.check.outputs.ahead }}) since ${{ needs.check.outputs.base_tag || 'initial' }}..."
          # dotnet build ...
          # dotnet test ...
          # deploy ...

      - name: Update last-deploy tag
        if: success()
        uses: hermitos/tag-deploy-actions/tag-push@v2
        with:
          tag: last-deploy
```

#### Option B: Reusable-workflow wrapper

Keep your deploy logic in a separate reusable workflow and only call it when changes exist.

.github/workflows/deploy-nightly.yml (caller)

```yaml
name: Nightly Deploy
on:
  schedule:
    - cron: "5 0 * * *"
  workflow_dispatch: {}

permissions:
  contents: write

concurrency:
  group: nightly-rollup-${{ github.workflow }}-${{ github.ref_name }}
  cancel-in-progress: false

jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      has_changes: ${{ steps.rollup.outputs.has_changes }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: rollup
        uses: hermitos/tag-deploy-actions/tag-rollup@v2

  call-deploy:
    needs: check
    if: needs.check.outputs.has_changes == 'true'
    uses: ./.github/workflows/deploy.yml
```

.github/workflows/deploy.yml (reusable)

```yaml
name: Deploy
on:
  workflow_call: {}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Build & Deploy
        run: |
          # your full deploy here, can be multiple steps to
      - name: Update last-deploy tag
        uses: hermitos/tag-deploy-actions/tag-push@v2

```

---

## 📌 Important Notes

### Fetching Updated Tags Locally

**Git pull does NOT automatically update force-pushed tags!** After a deployment updates your tag, fetch it locally with:

```bash
git fetch --tags --force
# or more explicitly:
git fetch origin '+refs/tags/*:refs/tags/*' --force
```

### Why Force Fetch?

When a tag is moved to a new commit (force-pushed), Git won't update it with a regular fetch because that would overwrite local data. The `--force` flag tells Git "yes, update my local tags to match the remote, even if they've moved."
