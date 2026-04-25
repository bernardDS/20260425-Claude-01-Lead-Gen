# /github-push

Deploy this project to GitHub Pages. Execute all steps below in sequence. Stop immediately and report if any step fails.

---

## Step 1 — Security Scan (ABORT if any secrets found)

Refresh PATH first so git is available:
```powershell
$env:PATH = [System.Environment]::GetEnvironmentVariable("PATH","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("PATH","User")
```

Run all four checks. If ANY match is found, print a clear warning listing the file and matched line, then STOP — do not proceed to Step 2.

**Check A — .env must never be staged:**
```powershell
git ls-files --cached --error-unmatch .env 2>$null; if ($LASTEXITCODE -eq 0) { Write-Error "BLOCKED: .env is tracked by git. Run: git rm --cached .env" }
```

**Check B — Scan staged content for secret patterns:**
```powershell
$patterns = @(
  'apify_api_[A-Za-z0-9]{20,}',
  'ghp_[A-Za-z0-9]{36,}',
  'github_pat_[A-Za-z0-9_]{80,}',
  'AKIA[0-9A-Z]{16}',
  'sk-[A-Za-z0-9]{32,}',
  'password\s*=\s*\S+',
  'secret\s*=\s*[A-Za-z0-9+/]{8,}',
  'private_key\s*=\s*\S+'
)
$staged = git diff --cached
foreach ($p in $patterns) {
  $hits = $staged | Select-String -Pattern $p
  if ($hits) { Write-Error "BLOCKED: Secret pattern '$p' found in staged changes:`n$hits" }
}
```

**Check C — Scan all committed files in the repo for secrets:**
```powershell
$patterns = @('apify_api_[A-Za-z0-9]{20,}','ghp_[A-Za-z0-9]{36,}','AKIA[0-9A-Z]{16}','sk-[A-Za-z0-9]{32,}')
foreach ($p in $patterns) {
  $hits = git grep -I --line-number $p -- 2>$null
  if ($hits) { Write-Error "BLOCKED: Secret pattern '$p' found in tracked files:`n$hits" }
}
```

**Check D — Ensure .env and .mcp.json are in .gitignore:**
```powershell
$gi = Get-Content .gitignore -ErrorAction SilentlyContinue
if ($gi -notmatch '\.env') { Write-Error "BLOCKED: .env missing from .gitignore" }
if ($gi -notmatch '\.mcp\.json') { Write-Error "BLOCKED: .mcp.json missing from .gitignore" }
```

If all four checks pass, print: `✅ Security scan passed — no secrets detected.`

---

## Step 2 — Generate or Update README.md

Read the current `index.html` to understand the app. Then write a professional `README.md` that includes:

1. **Project title and one-line description**
2. **Live demo link** — `https://{owner}.github.io/{repo}/` (derive from `git remote get-url origin`)
3. **Features** — bullet list based on what the app actually does
4. **Tech Stack** — list the actual technologies used
5. **Setup & Usage** — how to open locally and how to configure the Apify API key
6. **Screenshots** — placeholder section with instruction: `> Add screenshots to a /screenshots folder and reference them here`
7. **Environment Variables** — table listing `APIFY_API_TOKEN` and where to get it
8. **Deployment** — note that GitHub Actions auto-deploys on push to `main`

---

## Step 3 — Ensure GitHub Actions workflow is correct

Check if `.github/workflows/deploy.yml` exists. If it does not exist, create it with this exact content:

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
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deploy.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .
      - id: deploy
        uses: actions/deploy-pages@v4
```

If it already exists and is correct, leave it unchanged.

---

## Step 4 — Stage, commit, and push

```powershell
git add .
git status
```

Review `git status` output. If `.env` or `.mcp.json` appears in staged files, run `git restore --staged .env` / `git restore --staged .mcp.json` before committing.

Build a descriptive commit message that summarises what changed (e.g. `"update README, add deploy workflow"`). Then:

```powershell
git commit -m "<descriptive message>"
git push
```

---

## Step 5 — Enable GitHub Pages via gh CLI

Get the owner and repo name from the remote URL:
```powershell
$remote = git remote get-url origin
# parse owner/repo from https://github.com/owner/repo.git
```

Enable GitHub Pages with GitHub Actions as the source:
```powershell
gh api repos/{owner}/{repo}/pages --method POST -f build_type=workflow -f source='{"branch":"main","path":"/"}' 2>$null
# If already enabled the POST will fail — that is fine, ignore the error
gh api repos/{owner}/{repo}/pages --method PUT -f build_type=workflow 2>$null
```

---

## Step 6 — Update repo description and topics via gh CLI

Derive a short description from the README title/features. Then:

```powershell
gh repo edit --description "Singapore B2B lead generation tool powered by Apify Google Maps scraper"
gh repo edit --add-topic "lead-generation" --add-topic "apify" --add-topic "singapore" --add-topic "google-maps" --add-topic "html" --add-topic "github-pages"
```

---

## Step 7 — Print the live URL

Derive and print:
```
🚀 Deployed! Live at: https://{owner}.github.io/{repo}/
📋 Actions log:       https://github.com/{owner}/{repo}/actions
```
