# Option B — Run the nightly refresh on GitHub Actions (step by step)

Goal: the estimator refreshes itself in the cloud every morning — pulls Asana,
recalibrates, republishes to Canvas — with no laptop involved. ~20 minutes once.

> Current state (2026-06-04): the local git repo already exists (`main`, one
> commit, `.github/workflows/refresh.yml` ready). You're starting at Step 1.

---

## Step 1 — Commit what's pending

```bash
cd "/Users/ljt/Library/CloudStorage/GoogleDrive-jliu234@nd.edu/My Drive/PM intern/odl_estimator"
git add -A
git commit -m "CI workflow + ND Learning button docs"
```

## Step 2 — Create `refresh_config.json` (CI reads it; it holds NO secrets)

```bash
cp refresh_config.example.json refresh_config.json
# one-off, to find the portfolio GID:
export ASANA_TOKEN="<your new asana token>"
python3 asana_pull.py discover
```

Edit `refresh_config.json`:
- `asana_portfolio_gid`: the GID from `discover`
- `canvas.course_id`: `148587`
- `canvas.page_url`: the slug from your embed page's address bar
  (`canvas.nd.edu/courses/148587/pages/THIS-PART`)
- `canvas.parent_folder_path`: the Files folder you uploaded into (`Uploaded Media`)

```bash
git add refresh_config.json && git commit -m "refresh config (gids/slugs only, no secrets)"
```

## Step 3 — Create the **private** repo (web browser, no CLI needed)

**Preferred: ND's GitHub** (data stays inside ND — the repo contains per-person
time entries, so it MUST be private either way):

1. Go to **github.nd.edu**, sign in with your netID.
2. **+ → New repository** → name `odl-estimator` → **Private** →
   do NOT add a README/.gitignore/license (the repo already has history) → Create.
3. Check Actions are enabled: repo **Settings → Actions → General** →
   "Allow all actions". (If your org has Actions disabled, ask OIT — or use
   github.com instead, with Michael/Annie's OK since staff data leaves ND infra.)

## Step 4 — Push

Copy the repo's HTTPS URL from the page GitHub shows you, then:

```bash
git remote add origin https://github.nd.edu/<your-netid>/odl-estimator.git
git push -u origin main
```

When asked for a password: GitHub doesn't accept account passwords on the
command line — create a **Personal Access Token** (GitHub → your avatar →
Settings → Developer settings → Personal access tokens → classic → scope
`repo`) and paste *that* as the password. macOS keychain remembers it after
the first push.

## Step 5 — Add the two secrets

Repo → **Settings → Secrets and variables → Actions → New repository secret**:

| Name | Value |
|---|---|
| `ASANA_TOKEN` | Asana → My Settings → Apps → Developer apps (your NEW token) |
| `CANVAS_TOKEN` | Canvas → Account → Settings → **+ New Access Token** |

Secrets are encrypted; nobody (including you) can read them back.

## Step 6 — First run, supervised

1. Repo → **Actions** tab → (enable workflows if prompted) →
   **nightly-refresh** → **Run workflow** → `main` → Run.
2. Watch it (~3–5 min). Green check = done. Click the run → the **Summary**
   shows the same delta report you get locally (headline numbers, review queue).
3. Verify Canvas: open your course page — the guide should load; in Files, the
   bundle's timestamp should be "a few minutes ago".
4. The run also commits the refreshed numbers back to `main`
   ("nightly refresh 2026-…") — that's the audit trail. `git pull` locally
   before your next manual edit.

From now on it fires **daily at 11:30 UTC (06:30/07:30 South Bend)**. A failed
run emails you automatically.

## Step 7 — Turn OFF the laptop schedule (avoid double publishing)

```bash
./install_schedule.sh uninstall
```

One home for the engine, per `HANDOFF.md`. Your Mac is now optional.

## Step 8 — Ongoing operations (all through GitHub, ~2 min when needed)

- **New project queued?** The run summary warns and shows
  `needs_review.csv`. Fix in the browser: repo → `project_registry.csv` →
  pencil icon → add the row (gid, archetype, is_course_dev) → Commit → Actions
  → Run workflow. Done.
- **Registry/docs edits**: same web editor, or locally + `git pull`/`push`.
- **Succession**: add the successor as repo **admin**; they replace both
  secrets with their own tokens (yours die when your ND account closes) —
  that's the whole handoff, plus reading `HANDOFF.md`.

## Caveats worth knowing

- **Google Drive + git**: your working copy lives in Drive, which can create
  "conflicted copy" noise inside `.git/`. Once GitHub is canonical, prefer
  editing via the GitHub web UI, or keep a clean clone outside Drive
  (`git clone <url> ~/odl-estimator`) for local work. (Today's `.gitignore`
  deletion was likely Drive sync at work.)
- **Token lifetimes**: Canvas tokens can have expiry dates — set "no
  expiration" or calendar a renewal. If a nightly run starts failing with 401,
  a token died; replace the secret.
- **Branch protection**: don't require PR reviews on `main`, or the bot's
  audit-trail commit in Step 6.4 will be rejected.
