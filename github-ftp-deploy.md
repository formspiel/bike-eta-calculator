# GitHub FTP Deployment — Session Instructions

You are helping set up or maintain automated FTP deployment via GitHub Actions.
Follow these instructions exactly. They encode hard-won lessons from real failures.

---

## Step 1 — Ask about the trigger branch

Before writing any workflow file, ask the user:

> "The workflow will trigger on push to `main` by default.
> Does that fit this project, or would a different branch work better?
> Common alternatives:
> - `main` — deploy every merge to main (standard for simple sites)
> - `ready` — a dedicated deploy branch; push here only when you want to ship
> - `release/**` — deploy on any release branch
> - `workflow_dispatch` only — manual trigger from the Actions tab, no automatic deploys
> Which fits this project best?"

Wait for the user's answer before proceeding. Use their chosen branch in the workflow below.

---

## Step 2 — Ask about excluded files

Before writing the workflow, look at the repository contents and ask the user which
files and folders should NOT be deployed to the server.

First, scan the repo for common patterns and tell the user what you found.
Then present the proposed exclude list and ask for confirmation.

Say:

> "These files and folders will be excluded from deployment. The first group is
> always excluded — the rest I'm suggesting based on what I see in this repo.
> Please confirm or adjust:"
>
> **Always excluded (never deploy these):**
> - `**/.git*` and `**/.git*/**` — git metadata
> - `.github/**` — workflow files
> - `.claude/**` and `CLAUDE.md` — Claude session files (deploying these caused a real ECONNRESET failure when the action tried to create the `.claude/` directory on the server)
> - `**/*.md` — documentation files
> - `lighthouserc.json` — CI config, not web content
>
> **Suggested for this project** *(remove any that should actually be deployed)*:
> - `**/node_modules/**` — if `package.json` is present
> - `**/vendor/**` — if `composer.json` is present
> - `**/.env*` — environment files (never deploy these)
> - `**/tests/**` or `**/test/**` — test suites
> - `**/dist/**` — build output, if the server should build itself
> - `**/*.md` — documentation files (README, CHANGELOG, etc.)
> - `.claude/**` and `CLAUDE.md` — Claude session files
>
> "Are there other files or folders in this repo that should stay off the server?"

Wait for the user's answer. Build the final exclude list from their response.

Common project types for reference:

| Project type | Typical extra excludes |
|---|---|
| Static HTML site | `**/*.md` |
| Node.js app | `**/node_modules/**`, `**/.env*` |
| PHP / WordPress | `**/vendor/**`, `**/.env*`, `**/tests/**` |
| Built frontend (e.g. Vite) | everything except `dist/**` — use `local-dir: ./dist/` instead |
| WordPress theme/plugin | `**/node_modules/**`, `**/vendor/**`, `**/.env*`, `**/tests/**` |

---

## Step 3 — Create the workflow file

Create `.github/workflows/deploy-ftp.yml` with the following content,
substituting `BRANCH` with the branch chosen in Step 1 and the `exclude` block
with the list agreed in Step 2:

```yaml
name: Deploy via FTP

on:
  push:
    branches:
      - BRANCH

jobs:
  deploy:
    name: FTP Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Disable IPv6
        run: |
          sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
          sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1

      - name: Deploy to FTP server
        uses: SamKirkland/FTP-Deploy-Action@v4.4.0
        with:
          server: ${{ secrets.FTP_SERVER }}
          username: ${{ secrets.FTP_USERNAME }}
          password: ${{ secrets.FTP_PASSWORD }}
          port: ${{ secrets.FTP_PORT }}
          protocol: ftps
          server-dir: /
          local-dir: ./
          exclude: |
            **/.git*
            **/.git*/**
            **/node_modules/**
            .github/**
            .claude/**
            CLAUDE.md
            lighthouserc.json
            **/*.md
          log-level: verbose
          dry-run: false
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

If they need to add them, wait for confirmation before proceeding.

---

## Step 5 — Commit and push

Commit the workflow file and push to the repository.
If the trigger branch doesn't exist yet, create it.

---

## Step 6 — Verify the run

After pushing, monitor the Actions tab. Common errors and their fixes:

| Error | Fix |
|---|---|
| `unable to find version` | The tag format is wrong — it must be exactly `@v4.4.0` (with `v`) |
| `ECONNRESET` on data socket (control) | Change `protocol: ftp` to `protocol: ftps` — never use plain `ftp` |
| `ECONNRESET` on data socket (upload) | The runner uses EPSV (IPv6) for the data channel even over FTPS. Add the **Disable IPv6** step before the deploy step (see template above). Modifying `/etc/gai.conf` does NOT fix this. |
| `Login incorrect` | A secret value is wrong — double-check FTP_USERNAME and FTP_PASSWORD |
| `550 Permission denied` | FTP user lacks write access to `server-dir` — adjust the path |
| `No files uploaded` | Check `local-dir` and `exclude` patterns |

---

## Hard rules — never break these

1. **Always use `protocol: ftps`**, never `protocol: ftp`.
   Plain FTP causes `ECONNRESET` on the data socket when the runner connects via IPv6.

2. **Always include the Disable IPv6 step** before the deploy step.
   Even with `protocol: ftps`, the underlying `basic-ftp` library sets passive mode based on the control socket's address family. Node.js connects to IPv4 addresses via IPv4-mapped IPv6 sockets (`::ffff:x.x.x.x`), so `socket.remoteFamily` returns `'IPv6'` and the library picks EPSV. Kasserver drops IPv6 data connections with ECONNRESET. Disabling IPv6 at the kernel level forces pure IPv4 sockets, so `basic-ftp` uses PASV instead.
   **Note:** modifying `/etc/gai.conf` does NOT fix this — it only affects DNS resolution, not the socket address family.
   ```yaml
   - name: Disable IPv6
     run: |
       sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
       sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
   ```

3. **Use exactly `@v4.4.0`** — this is the verified working version. Do not look up the
   latest release or substitute a different tag. Tags without the `v` prefix do not exist.
   Only update the version if the user explicitly asks to upgrade.

4. **Never hardcode credentials**. Always use `${{ secrets.* }}`.
