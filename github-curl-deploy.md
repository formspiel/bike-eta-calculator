# GitHub FTPS Deployment with curl — Session Instructions

You are helping set up or maintain automated FTP deployment via GitHub Actions using `curl`.
Follow these instructions exactly. They encode hard-won lessons from real failures.

`curl` is pre-installed on every GitHub Actions runner. No third-party action is needed.
This approach is best for projects that deploy a small, known set of files.
For projects with many files or nested directories, consider the FTP Deploy Action instead — see the note at the end.

---

## Step 1 — Ask about the trigger branch

Before writing any workflow file, ask the user:

> "The workflow will trigger on push to `main` by default.
> Does that fit this project, or would a different branch work better?
> Common alternatives:
> - `main` — deploy every merge to main (standard for simple sites)
> - `ready` — a dedicated deploy branch; push here only when you want to ship
> - `workflow_dispatch` only — manual trigger from the Actions tab, no automatic deploys
> Which fits this project best?"

Wait for the user's answer before proceeding.

---

## Step 2 — Ask which files to deploy

`curl` uploads files explicitly, one at a time. Look at the repository and ask:

> "Which files should be deployed to the server? I can see:
> [list the web-facing files you find — e.g. index.html, sw.js, style.css]
>
> Files I'd suggest excluding:
> - Everything in `.git/`, `.github/`, `.claude/`
> - `*.md` — documentation
> - Any config, test, or build files not needed at runtime"

Wait for confirmation. Build the final file list from their answer.

---

## Step 3 — Create the workflow file

Create `.github/workflows/deploy-ftp.yml` with the following content,
substituting `BRANCH` with the branch chosen in Step 1 and the file list from Step 2:

```yaml
name: Deploy via FTP

on:
  push:
    branches:
      - BRANCH

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deploy via FTPS
        env:
          FTP_SERVER: ${{ secrets.FTP_SERVER }}
          FTP_USERNAME: ${{ secrets.FTP_USERNAME }}
          FTP_PASSWORD: ${{ secrets.FTP_PASSWORD }}
          FTP_PORT: ${{ secrets.FTP_PORT }}
        run: |
          curl --ssl-reqd --disable-epsv --silent --show-error \
               --user "$FTP_USERNAME:$FTP_PASSWORD" \
               --upload-file index.html \
               "ftp://$FTP_SERVER:$FTP_PORT/index.html"
          curl --ssl-reqd --disable-epsv --silent --show-error \
               --user "$FTP_USERNAME:$FTP_PASSWORD" \
               --upload-file sw.js \
               "ftp://$FTP_SERVER:$FTP_PORT/sw.js"
```

Add one `curl` line per file. Repeat the pattern for each file in the list from Step 2.

### Adding a version stamp (optional)

If the project uses a `__VERSION__` placeholder in a file, add these steps before Deploy:

```yaml
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0        # required for git rev-list --count to work

      - name: Stamp version
        run: |
          VERSION="v1.$(git rev-list --count HEAD) ($(git rev-parse --short HEAD))"
          sed -i "s/__VERSION__/$VERSION/" index.html
```

**For large repositories** where fetching the full history is too slow, use the GitHub run
number instead — no `fetch-depth: 0` needed:

```yaml
      - name: Stamp version
        run: |
          VERSION="v1.${{ github.run_number }} ($(git rev-parse --short HEAD))"
          sed -i "s/__VERSION__/$VERSION/" index.html
```

### Adding HTML minification (optional)

If the project is a static HTML site, add this step after Stamp version and before Deploy:

```yaml
      - name: Minify
        run: npx html-minifier-terser --collapse-whitespace --remove-comments --minify-css true --minify-js true index.html -o index.html
```

---

## Step 4 — Confirm GitHub Secrets are configured

Tell the user:

> "Before the workflow can run, four secrets must be added to this repository under
> **Settings → Secrets and variables → Actions**:"

| Secret name | Description |
|---|---|
| `FTP_SERVER` | FTP hostname (e.g. `w01*****.kasserver.com`) |
| `FTP_USERNAME` | FTP account username |
| `FTP_PASSWORD` | FTP account password |
| `FTP_PORT` | FTP port (usually `21`) |

Ask: "Are the secrets already configured, or do you need to add them now?"

---

## Step 5 — Commit and push

Commit the workflow file and push to the repository.
If the trigger branch doesn't exist yet, create it.

---

## Step 6 — Verify the run

After pushing, monitor the Actions tab. Common errors and their fixes:

| Error | Fix |
|---|---|
| `curl: (67) Login denied` | A secret value is wrong — check FTP_USERNAME and FTP_PASSWORD |
| `curl: (9) Access denied` | The remote path is wrong — check that the path in the URL matches the server structure |
| `curl: (35) SSL connect error` | Server doesn't support FTPS — check with your host; try removing `--ssl-reqd` temporarily to diagnose |
| `ECONNRESET` / connection drops during upload | Add `--disable-epsv` if not already present (forces PASV mode, avoids IPv6 data connection issues) |
| File not updated on server | Check that the upload URL path is correct; some hosts use a subdirectory like `/httpdocs/` |

---

## Hard rules — never break these

1. **Always use `--ssl-reqd`**, never omit it.
   This requires FTPS and refuses to fall back to plain FTP. Plain FTP sends credentials
   and file content unencrypted.

2. **Always use `--disable-epsv`**.
   Without it, curl may negotiate EPSV (Extended Passive mode) for the data connection.
   Some servers (e.g. kasserver) drop IPv6 data connections, causing uploads to fail silently
   or with a connection reset. `--disable-epsv` forces PASV (standard passive mode) which
   works reliably over IPv4. It is harmless on servers that support EPSV.

3. **Always use `--silent --show-error` together**.
   `--silent` suppresses the progress bar (noisy in CI logs).
   `--show-error` restores error output even when `--silent` is set.
   Without this combination you either get a cluttered log or silent failures.

4. **Never hardcode credentials**. Always use `${{ secrets.* }}` and pass them
   as environment variables. Never interpolate secrets directly into the `run:` script.

5. **List files explicitly**.
   curl uploads one file at a time. There is no automatic directory mirroring.
   If a new file needs to be deployed, add a curl line for it.
   This is a feature, not a limitation — you always know exactly what is on the server.

6. **Use `--ftp-create-dirs` for subdirectory deployments**.
   When deploying to a path that may not exist yet (e.g. `/v2/index.html` for a staging
   environment), add `--ftp-create-dirs` to let curl create the directory automatically.
   Without it, the upload fails if the directory is missing.
   ```bash
   curl --ssl-reqd --disable-epsv --silent --show-error --ftp-create-dirs \
        --user "$FTP_USERNAME:$FTP_PASSWORD" \
        --upload-file index.html \
        "ftp://$FTP_SERVER:$FTP_PORT/v2/index.html"
   ```
   This is the basis for a simple staging/preview setup: a second workflow on a feature
   branch deploys to `/v2/` (or any subdirectory), leaving the production root untouched.

---

## When to use the FTP Deploy Action instead

`curl` is the right choice when:
- You deploy a small, fixed set of files (2–10 files)
- You want no third-party action dependencies
- You want a transparent, easy-to-read workflow

The **FTP Deploy Action** (`SamKirkland/FTP-Deploy-Action`) is worth considering when:
- The project has many files (themes, plugins, PHP apps with dozens of files)
- You need **smart sync** — the action tracks file hashes and only uploads changed files,
  which saves significant time on large deploys
- You need **directory mirroring** — the action recursively copies a local folder to the
  server, preserving structure, without you having to list every file
- You want a **dry-run** mode to preview what would be uploaded before committing

The trade-off: the action is a pinned third-party dependency. Pin it to an exact version
tag (e.g. `@v4.4.0`) and review the changelog before upgrading.
