# Migration plan — `constructorfabric/website` on GitHub Pages

Target setup:

- **New repo:** `github.com/constructorfabric/website`
- **Hosting:** GitHub Pages (auto-deploy from `main`, no staging)
- **Custom domain:** `constructorfabric.org` (+ `www.constructorfabric.org`)
- **Layout:** Site flattened — `v3/*` becomes the repo root
- **Source:** Pure static HTML/CSS + JSX-via-Babel-Standalone (no build step)

---

## Phase 1 — Prepare a clean working copy

Work in a fresh local folder so the old clone stays intact as a safety net.

```sh
# Make a clean staging copy
cd ~/Documents/Projects
cp -R cyberfabric-website-preview website-migration
cd website-migration
```

### 1.1 Delete what you don't need

| Path | Why drop |
|---|---|
| `.git/` | Will create fresh history with new remote. |
| `.venv/` | Local Python venv, never belongs in the repo. |
| `canonical/` | Reference-only; not deployed (per old README). The new site is `v3/`. |
| `.github/workflows/deploy-staging.yml` | SSH/rsync deploy is replaced by Pages workflow. |
| `.github/workflows/promote-production.yml` | No staging→prod model anymore. |
| `README.md` | Will be rewritten. |
| `CONTRIBUTING.md` | Will be rewritten (or dropped if you don't need it). |
| `_migration.md` | This file — drop after migration is done. |

```sh
rm -rf .git .venv canonical
rm -f .github/workflows/deploy-staging.yml .github/workflows/promote-production.yml
rm -f README.md CONTRIBUTING.md _migration.md
```

### 1.2 Flatten `v3/` to repo root

```sh
# Move v3 contents (including hidden files) up one level, then remove v3
shopt -s dotglob 2>/dev/null || setopt dotglob
mv v3/* .
rmdir v3
```

Verify the root now contains: `index.html`, `elements.html`, `foundation.html`, `participate.html`, `learn.html`, `privacy-policy.html`, `404.html`, `partials.jsx`, `styles.css`, `sitemap.xml`, `robots.txt`, `assets/`.

### 1.3 No code changes needed for flattening

All asset paths in the HTML are already relative (`assets/img/...`, `styles.css`, `partials.jsx`) — they work identically at the root. Nav hrefs are relative filenames (`elements.html`, `learn.html`) — also unaffected.

---

## Phase 2 — Add GitHub Pages configuration

### 2.1 Custom domain file

```sh
echo 'constructorfabric.org' > CNAME
```

`CNAME` must live at the deployed root. The Pages workflow uploads the whole repo, so a root-level `CNAME` is correct.

### 2.2 Pages deploy workflow

Create `.github/workflows/pages.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .
      - id: deployment
        uses: actions/deploy-pages@v4
```

That's it — every push to `main` deploys automatically.

### 2.3 `.gitignore`

Create `.gitignore` at the root:

```
.venv/
.DS_Store
node_modules/
*.log
```

---

## Phase 3 — Push to the new repo

```sh
# Create repo first on github.com/constructorfabric (web UI)
# — name: website
# — visibility: public (required for free Pages)
# — no README/license/.gitignore (we already have files)

git init -b main
git add .
git commit -m "Initial site: Constructor Fabric (migrated from preview repo)"
git remote add origin git@github.com:constructorfabric/website.git
git push -u origin main
```

The push triggers the Pages workflow. Watch it under **Actions** tab. First run takes ~30–60 s.

---

## Phase 4 — Enable Pages and configure the domain

### 4.1 Repo settings

In GitHub → **Settings → Pages**:

1. **Source:** `GitHub Actions` (should already be selected if workflow ran).
2. **Custom domain:** `constructorfabric.org` — type it and click **Save**. GitHub verifies the `CNAME` file matches.
3. Leave **Enforce HTTPS** unchecked for now; enable it after the cert is issued (see 4.3).

### 4.2 DNS at your registrar

For the apex `constructorfabric.org`, create four **A** records:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

Plus AAAA records (IPv6, optional but recommended):

```
2606:50c0:8000::153
2606:50c0:8001::153
2606:50c0:8002::153
2606:50c0:8003::153
```

For `www.constructorfabric.org`, create a **CNAME** record pointing to `constructorfabric.github.io`.

> If GitHub asks you to verify domain ownership at the org level (Settings → Pages → Verified domains), do that — it prevents domain takeover.

### 4.3 TLS

Once DNS propagates (minutes to a few hours), GitHub auto-issues a Let's Encrypt cert. Then return to **Settings → Pages** and tick **Enforce HTTPS**.

---

## Phase 5 — Validate

- [ ] `https://constructor​fabric.github.io/website/` (raw URL, before DNS) — wait, this is `<org>.github.io/<repo>`: `https://constructorfabric.github.io/website/` → loads.
- [ ] `https://constructorfabric.org/` → loads with valid TLS cert.
- [ ] `https://www.constructorfabric.org/` → redirects to apex (Pages does this when both are configured).
- [ ] All nav links work: `/elements.html`, `/foundation.html`, `/participate.html`, `/learn.html`.
- [ ] Anchor links: `/elements.html#constructor-studio`, `#constructor-insight`, `#constructor-ware`.
- [ ] Footer "Learn" link works.
- [ ] DevTools → Network: zero 404s. Favicon, OG image, founder photos, dashboard screenshots, Constructor Insight screenshots all load.
- [ ] `/sitemap.xml` and `/robots.txt` reachable.
- [ ] Bad URL like `/nope` shows the custom `404.html`.

---

## Phase 6 — Decommission old setup

- **Old preview repo** (`vzhuman/cyberfabric-website-preview` / `aydinahmett/constructorfabric-website-preview`): archive via Settings → Danger Zone → Archive. Add a one-line README pointing to the new repo.
- **Old VPS** `23.109.29.133`: once the new site has been live for ~48 h with no issues, revoke the deploy SSH key from the server's `authorized_keys` and power off the box.

---

## What is intentionally NOT migrated

| Dropped | Reason |
|---|---|
| `canonical/` | Reference-only; superseded by `v3/`. |
| `canonical/cyber-ware-4blocks*.png` | Orphan images, not referenced. |
| `canonical/CLAUDE.md` | Internal notes from earlier phase. |
| Both old workflows | Replaced by a single Pages workflow. |
| `.venv/` | Local artifact. |
| Staging/promote model | Pages auto-deploys from `main` — single source of truth. |
| Server snapshot/rollback scripts | Pages keeps deployment history; revert by reverting the commit and pushing. |

---

## Local development (unchanged)

```sh
# From the repo root (after flatten)
python3 -m http.server 8000   # http://localhost:8000
```

The site uses Babel Standalone to load `partials.jsx` in the browser, so `file://` won't work — always use an HTTP server.
