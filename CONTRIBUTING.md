# Publishing Your App to the Cribl Marketplace

Self-service guide for Cribl-Community app authors. Follow these steps to set up automated dispensary publishing in your repo.

## 1. package.json Requirements

Your `package.json` must include these fields:

```json
{
  "name": "cc-your-pack-name",
  "version": "1.0.0",
  "displayName": "Your Pack Display Name",
  "description": "What your pack does",
  "author": "John Smith - jsmith@cribl.io",
  "license": "Apache-2.0",
  "tags": {
    "product": ["stream", "search"]
  },
  "scripts": {
    "build": "...",
    "lint": "...",
    "package": "node scripts/package.mjs"
  }
}
```

**Requirements:**
- `name` must start with `cc-`
- `author` must be your name and email separated by a hyphen like the example above
- `tags.product` must be an array containing one or more of: `stream`, `search`, `lake`, `edge`
- `scripts.package` must point to `node scripts/package.mjs`

## 2. Add Build Scripts

Copy these three files into your repo's `scripts/` directory:

| File | Purpose |
|------|---------|
| `scripts/package.mjs` | CLI entry point for `npm run package` — builds the .tgz artifact |
| `scripts/pkgutil.mjs` | Shared utilities: creates the pack archive, bundles README, handles Git layout |
| `scripts/prepare-git-pack.mjs` | Materializes the pack layout at repo root for "Import from Git" installs |

You can get these files from:
- The `scripts.zip` in this repo's scaffold (`/Users/gmola/my-appNEWSCAFFOLD/scripts.zip`)
- Or copy them from any existing `cc-*` repo that's already set up (e.g., `cc-whoami`)

The build scripts expect your repo to have:
- A `dist/` directory produced by `npm run build` (your compiled app assets)
- Optionally `config/proxies.yml` and/or `config/policies.yml` (Cribl app config)

## 3. Add Release Workflow

Copy `templates/release.yml` from this repo into your repo at:

```
.github/workflows/release.yml
```

Or create it manually with this content:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - run: npm ci
      - run: npm run lint
      - run: npm run package -- --version "$(echo ${GITHUB_REF_NAME#v} | sed s/-staging$//)"

      - name: Publish pack layout for Git install
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          node scripts/prepare-git-pack.mjs --version "${GITHUB_REF_NAME#v}"
          git add -f static default package.json
          if ! git diff --staged --quiet; then
            git commit -m "chore(release): add Cribl pack layout for Git install"
            git tag -f "${GITHUB_REF_NAME}"
            git push origin "${GITHUB_REF_NAME}" --force
          fi
          git tag -f latest "${GITHUB_REF_NAME}"
          git push origin latest --force

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: build/*.tgz
          generate_release_notes: true
          fail_on_unmatched_files: true

      - name: Upload pack to Packs Dispensary (staging)
        if: contains(github.ref_name, '-staging')
        run: |
          TGZ_FILE=$(find build -name '*.tgz' | head -1)
          if [ -z "$TGZ_FILE" ]; then echo "ERROR: No .tgz found in build/"; exit 1; fi
          curl --fail --location '${{ vars.DISPENSARY_ENDPOINT_STAGING }}/${{ github.event.repository.name }}' \
            --header 'Authorization: Bearer ${{ secrets.PACKS_API_TOKEN_STAGING }}' \
            --form "file=@${TGZ_FILE}"

      - name: Upload pack to Packs Dispensary (prod)
        if: ${{ !contains(github.ref_name, '-staging') }}
        run: |
          TGZ_FILE=$(find build -name '*.tgz' | head -1)
          if [ -z "$TGZ_FILE" ]; then echo "ERROR: No .tgz found in build/"; exit 1; fi
          curl --fail --location '${{ vars.DISPENSARY_ENDPOINT }}/${{ github.event.repository.name }}' \
            --header 'Authorization: Bearer ${{ secrets.PACKS_API_TOKEN }}' \
            --form "file=@${TGZ_FILE}"
```

The org-level secrets and variables (`PACKS_API_TOKEN_*`, `DISPENSARY_ENDPOINT_*`) are already configured — your workflow will inherit them automatically.

## 4. Publishing a Release

Publishing is triggered by pushing a git tag:

```bash
# Bump version in package.json
npm version patch    # or minor/major

# Push the commit and tag
git push origin main --follow-tags
```

Or create a tag manually:

```bash
git tag v1.0.1
git push origin v1.0.1
```

This triggers the release workflow which:
1. Builds your app
2. Creates a `.tgz` pack archive
3. Publishes a GitHub Release
4. Uploads the pack to the dispensary

## 5. Testing on Staging First

Before publishing to prod, test on staging by appending `-staging` to your tag:

```bash
git tag v1.0.1-staging
git push origin v1.0.1-staging
```

This uploads to the staging dispensary only. Once verified, push the clean tag for prod:

```bash
git tag v1.0.1
git push origin v1.0.1
```

## 6. Verifying Your Setup Locally

Before pushing a tag, confirm everything builds locally:

```bash
npm ci
npm run lint
npm run package -- --version "1.0.0"
# Should produce build/cc-your-pack-name-1.0.0.tgz
ls build/*.tgz
```

## Quick Checklist

- [ ] `package.json` `name` starts with `cc-`
- [ ] `package.json` has `author` field (your name - email)
- [ ] `package.json` has `tags.product` array with at least one product
- [ ] `package.json` has `"package": "node scripts/package.mjs"` in scripts
- [ ] `scripts/package.mjs` exists
- [ ] `scripts/pkgutil.mjs` exists
- [ ] `scripts/prepare-git-pack.mjs` exists
- [ ] `.github/workflows/release.yml` exists with dispensary upload steps
- [ ] `npm run package -- --version "0.0.1"` succeeds locally
- [ ] (OPTIONAL)Staging test passed (`v*-staging` tag pushed and workflow succeeded)
