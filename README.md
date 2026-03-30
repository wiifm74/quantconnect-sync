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

## Configuration (GitHub Secrets / Variables)

### GitHub Secrets (required)
Set: **Settings → Secrets and variables → Actions → Secrets**

- `QC_USER_ID` — QuantConnect User ID (keep private)
- `QC_API_TOKEN` — QuantConnect API token (keep private)

### GitHub Variables (required)
Set: **Settings → Secrets and variables → Actions → Variables**

- `QC_PROJECT_ID` — target QuantConnect project id

## Repo structure (recommended
Typical layout (adjust to match your repo):
- `QuantConnect/` (default) — the QuantConnect project files tracked in Git
- `scripts/` — custom scripts that call the QuantConnect API
- `.github/workflows/` — workflows you run from the Actions tab

## Using GitHub Actions (click-to-run)

3. Go to **Actions** and run the workflows:
   - `Compare QuantConnect and Github` (Compare manually)
   - `Pull from QuantConnect` (QuantConnect → GitHub)
   - `Push to QuantConnect` (GitHub → QuantConnect)
   - `QC Auto Compare` (Compare automatically)
   - 'QuantConnect Sync (manual sync)

> Workflow names and exact behavior are defined in `.github/workflows/`

## Safety notes
- Never commit tokens/credentials. Use GitHub Secrets.
- Consider protecting `main` and using PRs for changes.

## Troubleshooting
- **Auth fails:** verify `QC_API_TOKEN` and confirm your QC account has API access enabled.
- **Wrong project updated:** confirm `QC_PROJECT_ID`.
- **Workflows not visible:** ensure workflow files exist in `.github/workflows/` on the main branch.

## License
Choose a license appropriate for reuse (MIT is common for templates).
