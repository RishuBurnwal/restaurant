# restaurant — rishu.dev → SiteGround staging auto-deploy

Live site: <https://anuranjanv17.sg-host.com/>
Repo: <https://github.com/anuranjanverma/restaurant>
Staging branch: `rishu.dev` (production branch stays `main`, untouched)

This README is a complete, repeatable runbook. Follow it top to bottom and anyone can
re-create the entire pipeline from scratch.

---

## 1. What this pipeline does

```
developer edit
      │
      ├─ git push origin rishu.dev
      │
      ▼
GitHub Actions  .github/workflows/deploy-rishu-dev.yml
      │  1. checkout
      │  2. guard: refuse to run if branch == main
      │  3. install deploy SSH key, pin host fingerprint
      │  4. SSH smoke test + confirm target directory exists
      │  5. rsync --dry-run      (always, output is uploaded as an artifact)
      │  6. rsync --itemize      (NO --delete, excludes .deployignore)
      │  7. ls -l on the server  (proves the file landed)
      │  8. curl the live URL    (proves the public web layer serves it)
      ▼
https://anuranjanv17.sg-host.com/rishu-test/deploy-test.html
```

Nothing on the server is ever deleted: `--delete` is deliberately absent, and deleting
is not exposed as a workflow input.

---

## 2. Safety model

| Risk | Control |
|---|---|
| Accidentally deploying to production | Workflow triggers on `rishu.dev` only; a hard guard fails the job if `GITHUB_REF_NAME == main` |
| Wiping live files | `rsync` runs without `--delete`; every run does a `--dry-run` first |
| Leaking secrets | `.env`, `*.pem`, `*.key`, `deploy-keys/`, `README-SSH.md` are git-ignored and rsync-excluded |
| Overwriting WordPress config | `wp-config.php`, `.user.ini`, `.htaccess` are rsync-excluded |
| Pushing 100 MB of media | `wp-content/uploads/`, caches, backups, `node_modules/`, `vendor/` are excluded |
| Man-in-the-middle on SSH | `ssh-keyscan` + optional pinned fingerprint secret; `StrictHostKeyChecking yes` |
| Deploy storms | `concurrency: deploy-rishu-dev` with `cancel-in-progress: false` |

Deployment target is a **subfolder** (`/rishu-test/`), not the web root, so the live
WordPress site cannot be affected while the pipeline is being proven.

---

## 3. Prerequisites

* Repository admin on `anuranjanverma/restaurant` (needed only once, for secrets).
* SiteGround plan with SSH access enabled (Site Tools → Devs → SSH Keys Manager).
* A SiteGround SSH user with write access to the target directory.
* Local: `git`, and optionally `gh` CLI.

> Note: the repository currently has a WordPress `.gitignore`. The theme/plugin code you
> actually want to ship lives under `wp-content/themes/...` and `wp-content/plugins/...`.
> Those paths are not ignored, so they will deploy normally.

---

## 4. One-time setup (do these once)

### 4.1 Generate a dedicated deploy key (local machine)

Never reuse a personal key. A dedicated key can be revoked without touching anything else.

```powershell
# Windows PowerShell
mkdir deploy-keys -Force
ssh-keygen -t ed25519 -C "gh-actions-deploy-rishu-dev" -f deploy-keys\sg_deploy -N '""'
```

```bash
# macOS / Linux
mkdir -p deploy-keys
ssh-keygen -t ed25519 -C "gh-actions-deploy-rishu-dev" -f deploy-keys/sg_deploy -N ""
```

This produces:

* `deploy-keys/sg_deploy` — **private** key. Goes into a GitHub secret. Never commit it.
* `deploy-keys/sg_deploy.pub` — **public** key. Goes into SiteGround. Safe to share.

`deploy-keys/` is already in `.gitignore`.

### 4.2 Authorize the public key on SiteGround

1. Open <https://tools.siteground.com/dashboard?siteId=SmczeFlYME1JUT09>
2. **Devs → SSH Keys Manager**
3. Paste the whole contents of `deploy-keys/sg_deploy.pub`
4. Save, then note:
   * **SSH host** (e.g. `ssh.anuranjanverma.com` or an IP shown on the page)
   * **SSH port** (SiteGround commonly uses `18765`)
   * **SSH username**

### 4.3 Import the private key into a GitHub secret

Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value |
|---|---|
| `SSH_PRIVATE_KEY` | entire contents of `deploy-keys/sg_deploy` (including the BEGIN/END lines) |
| `SSH_HOST` | SSH host from Site Tools |
| `SSH_PORT` | SSH port from Site Tools |
| `SSH_USER` | SSH username from Site Tools |
| `DEPLOY_PATH` | absolute server path, e.g. `/home/customer/www/anuranjanv17.sg-host.com/public_html/rishu-test` |
| `SSH_KNOWN_HOSTS_FINGERPRINT` | *(optional)* pinned host key fingerprint, see 4.5 |

Via CLI (requires repo admin):

```powershell
gh secret set SSH_PRIVATE_KEY --repo anuranjanverma/restaurant < deploy-keys\sg_deploy
gh secret set SSH_HOST --repo anuranjanverma/restaurant --body "ssh.example.com"
gh secret set SSH_PORT --repo anuranjanverma/restaurant --body "18765"
gh secret set SSH_USER --repo anuranjanverma/restaurant --body "u1234-abc"
gh secret set DEPLOY_PATH --repo anuranjanverma/restaurant --body "/home/customer/www/anuranjanv17.sg-host.com/public_html/rishu-test"
```

### 4.4 Create the `staging` environment (optional but recommended)

Repo → **Settings → Environments → New environment → `staging`**.
Add required reviewers here if you want a human gate before every staging deploy.
If you skip this, GitHub creates the environment automatically on the first run.

### 4.5 Pin the server host key (recommended)

```powershell
ssh-keyscan -p 18765 ssh.example.com 2>$null | ssh-keygen -lf -
```

Copy the `SHA256:...` value into the `SSH_KNOWN_HOSTS_FINGERPRINT` secret. If the server
key ever changes, the workflow fails instead of silently trusting a new host.

### 4.6 Create the target directory on the server

The workflow **fails early** if `DEPLOY_PATH` does not exist, on purpose.

```bash
ssh -p 18765 u1234-abc@ssh.example.com
mkdir -p ~/www/anuranjanv17.sg-host.com/public_html/rishu-test
chmod 755 ~/www/anuranjanv17.sg-host.com/public_html/rishu-test
exit
```

---

## 5. Daily workflow

```bash
git checkout rishu.dev
git pull
# ... edit files ...
git add -A
git commit -m "describe the change"
git push
```

Then watch it:

```bash
gh run list  --repo anuranjanverma/restaurant --branch rishu.dev --limit 3
gh run watch --repo anuranjanverma/restaurant
```

Or in the browser: repo → **Actions** → *Deploy rishu.dev → SiteGround staging*.

Manual run without a commit: **Actions → the workflow → Run workflow**, optionally
ticking `dry_run_only` to preview the rsync plan with zero writes.

---

## 6. Verifying a deploy

Four independent proofs, cheapest first:

1. **Workflow green** — the run page shows all steps ticked.
2. **Dry-run artifact** — download `deploy-log` from the run; `dryrun.txt` lists exactly
   which bytes moved.
3. **Server-side** — the *Prove file landed on the server* step prints
   `ls -l --time-style=full-iso …/deploy-test.html`.
4. **Public web** — open with a cache-buster:
   <https://anuranjanv17.sg-host.com/rishu-test/deploy-test.html?cb=1>

Bump `?cb=` (or hard-reload with Ctrl+F5) because SiteGround caches aggressively.

---

## 7. Rollback

### 7.1 Revert a bad change (preferred)

```bash
git checkout rishu.dev
git revert <bad-commit-sha>
git push
```

The revert is itself a commit, so the pipeline redeploys the previous good state
automatically. This is the only rollback path that keeps git and the server in sync.

### 7.2 Emergency: stop the bleeding

Repo → **Settings → Secrets and variables → Actions** → delete `SSH_PRIVATE_KEY`.
Every future run fails at the key step; nothing reaches the server. Re-add it when fixed.

### 7.3 Restore from backup

Take a backup **before** the first production deploy:

```bash
ssh -p 18765 u1234-abc@ssh.example.com
cd ~/www/anuranjanv17.sg-host.com
tar -czf ~/backup-public_html-$(date +%Y%m%d-%H%M%S).tar.gz -C public_html .
# database, if you have wp-cli:
wp db export ~/backup-db-$(date +%Y%m%d-%H%M%S).sql --path=public_html
exit
```

Restore:

```bash
cd ~/www/anuranjanv17.sg-host.com
tar -xzf ~/backup-public_html-YYYYMMDD-HHMMSS.tar.gz -C public_html
```

Keep backups off the server too (`scp` them down). A backup that lives only on the box
you are about to break is not a backup.

### 7.4 Files are never deleted by this pipeline

Because `--delete` is not used, a rollback only needs to restore *changed* files.
Nothing that existed before a deploy can disappear because of it.

---

## 8. Promoting the same pipeline to `main` (production)

Do this only after `rishu.dev` has been trusted for a while.

1. **Copy the workflow** to a production-named file so both can coexist:

   ```bash
   cp .github/workflows/deploy-rishu-dev.yml .github/workflows/deploy-main.yml
   ```

2. **Edit the copy**:
   * `on.push.branches: ["main"]`
   * `concurrency.group: deploy-production`
   * `environment.name: production`
   * `environment.url:` the production URL
   * Replace the *Guard* step with the inverse check — refuse anything that is not `main`:

     ```yaml
     - name: Guard - production deploys only from main
       run: |
         if [ "${GITHUB_REF_NAME}" != "main" ]; then
           echo "::error::Production deploys must come from main."
           exit 1
         fi
     ```

   * Point `DEPLOY_PATH` at the **real web root** (`public_html`, not `rishu-test`).

3. **Use separate secrets for production.** Do not reuse staging secrets. Create a second
   key pair (`deploy-keys/sg_deploy_prod`) and a second SiteGround SSH key, then store it
   under a production environment rather than at repo level:

   ```powershell
   gh secret set SSH_PRIVATE_KEY_PROD --repo anuranjanverma/restaurant --env production < deploy-keys\sg_deploy_prod
   ```

   Reference `secrets.SSH_PRIVATE_KEY_PROD` in `deploy-main.yml`.

4. **Require a reviewer before production deploys.**
   Repo → **Settings → Environments → New environment → `production`** →
   ✅ **Required reviewers** → add yourself → Save.
   Optional hardening: restrict *Deployment branches* to `main` only, and set a
   **Wait timer** of 5 minutes as a last-chance abort window.

5. **Protect the branch.**
   Repo → **Settings → Branches → Add branch protection rule** for `main`:
   * Require a pull request before merging
   * Require approvals: 1
   * Do not allow bypassing the above settings

6. **First production run**: open a PR from `rishu.dev` → `main`, let CI run, merge.
   The workflow pauses at the environment gate, you approve, and only then does rsync run.

7. **Change the parity guard**: once `main` deploys to production, change the staging
   workflow's guard so `rishu.dev` can never write to the production path. Two different
   `DEPLOY_PATH` secrets (staging vs production environment) already enforce this.

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `SSH_PRIVATE_KEY` empty in logs | secret not set, or set on the wrong repo | Re-add under Settings → Secrets |
| `Permission denied (publickey)` | public key not saved in Site Tools, or wrong user | Re-paste `sg_deploy.pub`, verify `SSH_USER` |
| `DEPLOY_PATH missing` | target directory not created | Step 4.6 |
| `Host key verification failed` | `ssh-keyscan` blocked or fingerprint changed | Check `SSH_PORT`, re-run 4.5 |
| Workflow green but page unchanged | CDN/proxy cache | Append `?cb=<random>`; hard-reload |
| `rsync: command not found` | rsync absent on server | SiteGround includes rsync; otherwise switch the workflow to `sftp`/`scp` |
| Job skipped entirely | branch name typo (`rishu-dev` vs `rishu.dev`) | Branch must be exactly `rishu.dev` |

---

## 10. File map

| Path | Purpose |
|---|---|
| `.github/workflows/deploy-rishu-dev.yml` | the pipeline |
| `.deployignore` | rsync exclude list (secrets, uploads, caches) |
| `.gitignore` | same intent, for git |
| `.gitattributes` | forces LF so CI scripts do not break |
| `deploy-test.html` | visible end-to-end proof page |
| `README.md` | this runbook |