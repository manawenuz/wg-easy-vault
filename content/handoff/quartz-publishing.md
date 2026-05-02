---
title: Publishing the vault with Quartz
status: shipped
type: handoff
---

# Publishing the vault with Quartz

The Obsidian vault in `docs/obsidian/` is published as a static site using [Quartz](https://quartz.jzhao.xyz/).

**Live site:** https://manawenuz.github.io/wg-easy-vault/

**Source repo:** https://github.com/manawenuz/wg-easy-vault

---

## Why a separate repo?

The main `wg-easy` repo already uses the `gh-pages` branch for MkDocs user documentation. GitHub Pages only supports one site per repo, so the Quartz site lives in its own repository.

---

## How it works

1. The Quartz repo (`wg-easy-vault`) contains a **copy** of the files from `docs/obsidian/` (in its `content/` folder).
2. A GitHub Actions workflow builds and deploys to GitHub Pages on every push to `main`.
3. `README.md` was renamed to `index.md` so Quartz treats it as the home page.
4. `.obsidian/` settings are ignored during the build.

---

## Updating the published site

After editing the vault in `docs/obsidian/`, run these commands from the project root:

```bash
cd ~/CascadeProjects/wg-easy-vault

# Sync latest vault content
rsync -av --delete --exclude='.obsidian' \
  ~/CascadeProjects/wg-easy-fork/docs/obsidian/ content/

# Commit and deploy
GIT_SSH_COMMAND="ssh -i ~/CascadeProjects/github -o IdentitiesOnly=yes" \
  git add -A && git commit -m "Sync vault" && git push
```

The workflow will build and deploy automatically. Check progress at:
https://github.com/manawenuz/wg-easy-vault/actions

---

## First-time setup (already done)

If you ever need to recreate this from scratch:

```bash
# 1. Create a new repo on GitHub
gh repo create wg-easy-vault --public --description "Obsidian vault for wg-easy"

# 2. Clone Quartz
git clone https://github.com/jackyzha0/quartz.git ~/CascadeProjects/wg-easy-vault
cd ~/CascadeProjects/wg-easy-vault
npm install

# 3. Initialize with the vault content
npx quartz create -X copy -s ~/CascadeProjects/wg-easy-fork/docs/obsidian -l shortest

# 4. Rename README -> index for the home page
mv content/README.md content/index.md

# 5. Edit quartz.config.ts:
#    pageTitle: "wg-easy fork — Vault"
#    baseUrl: "manawenuz.github.io/wg-easy-vault"
#    analytics.provider: "null"

# 6. Add the GitHub Actions workflow (see .github/workflows/deploy.yaml in the live repo)

# 7. Push
```

---

## Local preview

To preview changes before pushing:

```bash
cd ~/CascadeProjects/wg-easy-vault
npx quartz build --serve
```

Then open http://localhost:8080.
