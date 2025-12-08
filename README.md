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
| `base_tag` | The existing tag name if found (use this for safety check) |
| `ahead` | Number of commits since the tag |
| `current_branch` | The branch the check ran on |

---

### 🏷️ `tag-push`

Moves a tag to `HEAD` and pushes it back to your remote — great for marking a successful deploy.

#### `tag-push` Inputs

| Name | Optional  | Default | Description |
|------|-----------|----------|-------------|
| `tag` | ✅ | last-deploy | Tag name to move and push |
| `remote` | ✅ | `origin` | Remote name to push to |
| `expected_base_tag` | ✅ | "" | Expected tag from rollup (safety check) |

---

## ✨ Example Usage

Here's how you can build a simple nightly workflow that checks for new commits, runs a deployment if needed, and updates your tag.

> **Important**: Always use `fetch-depth: 0` to get full history, and the actions will handle fetching tags properly.

> **Real Examples**: See working implementations in the [HermitOS/TestTag](https://github.com/HermitOS/TestTag) repository:
>
> - [Prod.Nightly.Deploy.yml](https://github.com/HermitOS/TestTag/blob/dev/.github/workflows/Prod.Nightly.Deploy.yml) - Nightly deployment with rollup check
> - [Prod.Publish.yml](https://github.com/HermitOS/TestTag/blob/dev/.github/workflows/Prod.Publish.yml) - Manual deployment workflow

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
        uses: hermitos/tag-deploy-actions/tag-rollup@v2
        with:
          tag: last-deploy-prod

  deploy:
    needs: check
    if: needs.check.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      - name: Build & Deploy
        run: |
          echo "Deploying changes (ahead=${{ needs.check.outputs.ahead }}) since ${{ needs.check.outputs.base_tag || 'initial' }}..."
          # dotnet build ...
          # dotnet test ...
          # deploy ...

      - name: Update deployment tag
        if: success()
        uses: hermitos/tag-deploy-actions/tag-push@v2
        with:
          tag: last-deploy-prod
          expected_base_tag: ${{ needs.check.outputs.base_tag }}  # Safety check!
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
      base_tag: ${{ steps.rollup.outputs.base_tag }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0
      - id: rollup
        uses: hermitos/tag-deploy-actions/tag-rollup@v2
        with:
          tag: last-deploy-prod

  call-deploy:
    needs: check
    if: needs.check.outputs.has_changes == 'true'
    uses: ./.github/workflows/deploy.yml
    with:
      base_tag: ${{ needs.check.outputs.base_tag }}
```

.github/workflows/deploy.yml (reusable)

```yaml
name: Deploy
on:
  workflow_call:
    inputs:
      base_tag:
        description: 'Base tag from rollup check'
        required: false
        type: string
        default: ''

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0
      - name: Build & Deploy
        run: |
          echo "Deploying..."
          # your deploy steps
      - name: Update deployment tag
        uses: hermitos/tag-deploy-actions/tag-push@v2
        with:
          tag: last-deploy-prod
          expected_base_tag: ${{ inputs.base_tag }}  # Safety check!

```

---

## 📌 Important Notes

### Safety Check: Tag Mismatch Detection 🔒

Pass `base_tag` from `tag-rollup` to `tag-push` as `expected_base_tag`. If the tags don't match (e.g., you searched for `last-deploy-prod` but are trying to push `latest-deploy-prod`), the action will **fail loudly** and prevent the mistake.

```yaml
- id: rollup
  uses: hermitos/tag-deploy-actions/tag-rollup@v2
  with:
    tag: last-deploy-prod

- uses: hermitos/tag-deploy-actions/tag-push@v2
  with:
    tag: last-deploy-prod
    expected_base_tag: ${{ steps.rollup.outputs.base_tag }}  # ← Catches typos!
```

### Fetching Updated Tags Locally

**Git pull does NOT automatically update force-pushed tags!** After a deployment updates your tag, fetch it locally with:

```bash
git fetch --tags --force
# or more explicitly:
git fetch origin '+refs/tags/*:refs/tags/*' --force
```

### Why Force Fetch?

When a tag is moved to a new commit (force-pushed), Git won't update it with a regular fetch because that would overwrite local data. The `--force` flag tells Git "yes, update my local tags to match the remote, even if they've moved."


---

## 🔧 Troubleshooting

### "No tag found" but the tag exists

**Check for typos!** The action will suggest similar tags if it can't find an exact match:

```text
❌ ERROR: Tag 'latest-deploy-prod' NOT found!

🔍 Looking for similar tags...
   Did you mean one of these?
     - last-deploy-prod
     - last-deploy-test

   ⚠️  TYPO DETECTED: You searched for 'latest-deploy-prod' but 'last-deploy-prod' exists!
```

**Common typo**: `latest-deploy` vs `last-deploy` (note the extra 't')

### Tag mismatch error during push

```text
❌ ERROR: Tag mismatch detected!
   Rollup checked tag: 'last-deploy-prod'
   Trying to push tag: 'latest-deploy-prod'
```

**Fix:** Ensure both `tag-rollup` and `tag-push` use the exact same tag name. Use `expected_base_tag` parameter to catch this automatically.

### Tag not updating after deployment

**Check:** Does your workflow have `contents: write` permission?

```yaml
permissions:
  contents: write  # ← Required for pushing tags
```

---

## 📚 Real-World Examples

See these actions in use in the [HermitOS/TestTag](https://github.com/HermitOS/TestTag) repository:

- **[Prod.Nightly.Deploy.yml](https://github.com/HermitOS/TestTag/blob/dev/.github/workflows/Prod.Nightly.Deploy.yml)** - Complete nightly deployment workflow with rollup check
- **[Prod.Publish.yml](https://github.com/HermitOS/TestTag/blob/dev/.github/workflows/Prod.Publish.yml)** - Manual deployment trigger workflow

---

## 📝 License

MIT
