# QuantConnect Repo Sync Template (GitHub Actions + Custom API Scripts)

This repository is a **template** for syncing a QuantConnect project with GitHub using the **Actions** tab (click-to-run workflows).

It supports:
- **Pull**: QuantConnect → GitHub repo
- **Diff/Compare**: show what changed between QC and this repo
- **Push**: GitHub repo → QuantConnect

## Requirements / prerequisites
- **QuantConnect account with API access enabled** (your account must be allowed to use the QuantConnect API)
- A QuantConnect project (existing or newly created)
- GitHub repo admin access (to set Secrets/Variables and run workflows)

## How to use this template
1. Create a new repository from this template.
2. Configure the required GitHub **Secrets/Variables** (below).
3. Go to **Actions** and run the workflows:
   - `QC Pull` (QuantConnect → GitHub)
   - `QC Diff` (compare)
   - `QC Push` (GitHub → QuantConnect)

> Workflow names and exact behavior are defined in `.github/workflows/`.

## Configuration (GitHub Secrets / Variables)

### GitHub Secrets (required)
Set: **Settings → Secrets and variables → Actions → Secrets**

- `QC_API_TOKEN` — QuantConnect API token (keep private)

### GitHub Variables (recommended)
Set: **Settings → Secrets and variables → Actions → Variables**

- `QC_PROJECT_ID` — target QuantConnect project id
- `QC_ORG_ID` — optional (if you use an organization)
- `QC_API_BASE_URL` — optional (only if your scripts support overriding the API URL)

## Repo structure
Typical layout (adjust to match your repo):
- `Algorithm/` (or similar) — the QuantConnect project files tracked in Git
- `scripts/` — custom scripts that call the QuantConnect API
- `.github/workflows/` — workflows you run from the Actions tab

## Using GitHub Actions (click-to-run)

### Pull (QuantConnect → GitHub)
Runs the pull workflow to download the latest QC project files and commit them to the repo (or upload as artifacts, depending on your workflow).

**Actions → `QC Pull` → Run workflow**

Underlying command (example):
```bash
# TODO: replace with your real command
python scripts/qc_pull.py
```

### Diff / Compare
Runs a workflow that compares the current repo contents with what is in QuantConnect and reports differences.

**Actions → `QC Diff` → Run workflow**

Underlying command (example):
```bash
# TODO: replace with your real command
python scripts/qc_diff.py
```

### Push (GitHub → QuantConnect)
Runs a workflow that uploads repo files to the configured QuantConnect project.

**Actions → `QC Push` → Run workflow**

Underlying command (example):
```bash
# TODO: replace with your real command
python scripts/qc_push.py
```

## Safety notes
- Prefer **Pull → Diff → Push** to avoid overwriting changes made in the QuantConnect web editor.
- Never commit tokens/credentials. Use GitHub Secrets.
- Consider protecting `main` and using PRs for changes.

## Troubleshooting
- **Auth fails:** verify `QC_API_TOKEN` and confirm your QC account has API access enabled.
- **Wrong project updated:** confirm `QC_PROJECT_ID`.
- **Workflows not visible:** ensure workflow files exist in `.github/workflows/` on the default branch.

## License
Choose a license appropriate for reuse (MIT is common for templates).
