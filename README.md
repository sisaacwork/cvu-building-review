# building-gap-dashboard

CVU's tall-building gap analysis pipeline + review dashboard, in one repo.

Finds buildings ≥100m that exist in real-world data (OpenStreetMap via Overture Maps, plus TUM's Global Building Atlas) but are missing from CVU's Skyscraper Center database, country by country, then surfaces them through a satellite-imagery dashboard for the Skyscraper Database team to triage.

> **First time setting this up? → Start at [`SETUP.md`](SETUP.md).**
> It's the canonical step-by-step. `CLOUDFLARE_SETUP.md`, `DATA_REPO_SETUP.md`, and `HOW_TO_READ_OUTPUTS.md` are reference material the setup guide links to as needed.

## Two-repo layout

This **private** repo holds the pipeline, the data outputs, and the master copy of `dashboard.html`. A small **public** companion repo (e.g. `building-gap-app`) holds a copy of `dashboard.html` as `index.html` so GitHub Pages can serve it on the free plan. Reviewers hit the public URL; the dashboard talks to a Cloudflare Worker which holds the secrets and proxies to this private repo.

Run `./publish_dashboard.sh` after any dashboard edit to push the change to the public repo.

## What's in this repo

```
.
├── data/                       # Per-country outputs (CSVs, parquets, polygons)
│   ├── manifest.json           #   index of available countries
│   ├── CHN/                    #   one folder per ISO-3 country code
│   │   ├── likely_new.csv      #     candidate gaps, ranked
│   │   ├── borderline.csv
│   │   ├── likely_already_in_cvu.csv
│   │   ├── cvu_CN.csv
│   │   ├── cvu_coord_quality.csv
│   │   ├── overture_candidates.parquet
│   │   ├── gba_candidates.parquet
│   │   ├── admin.geojson       #     adm0/1/2 + urban centres for click-to-filter
│   │   └── reviews.csv         #     sidecar of reviewer marks (written by dashboard)
│   └── CAN/, USA/, …
│
├── cities/                     # Optional metro-area bbox overlays (per country)
│   ├── CHN.yaml
│   ├── CAN.yaml
│   └── README.md
│
├── cvu-dash-proxy/             # Cloudflare Worker (shields the data repo from public)
│   ├── worker.js
│   └── wrangler.toml
│
├── .github/workflows/          # Nightly CVU resync
│   └── resync.yml
│
├── dashboard.html              # The review UI — static, hosted via Pages
│
├── scan_overture.py            # Pipeline — Overture/OSM scanner
├── scan_gba.py                 # Pipeline — GBA bulk-download scanner
├── pull_country_boundary.py    # Pipeline — fetches country polygon from VUI
├── pull_admin_polygons.py      # Pipeline — fetches adm0/1/2 + UCs from VUI
├── pull_cvu.py                 # Pipeline — pulls CVU baseline buildings
├── dedupe.py                   # Pipeline — cross-source reconcile + CVU match
│
├── run_country.sh              # Pipeline driver — one country end-to-end
├── run_all_countries.sh        # Pipeline driver — top-50 countries batch
├── pull_admin_polygons_all.sh  # Backfill polygons for already-scanned countries
├── commit_outputs.sh           # Manifest regen + git commit per country
├── resync.sh                   # Fast-path CVU + dedupe re-run (no rescans)
├── publish_dashboard.sh        # Copy dashboard.html → public companion repo
│
├── requirements.txt            # Python deps
├── README.md                   # ← you are here
├── HOW_TO_READ_OUTPUTS.md      # Reviewer-facing guide to the CSV columns
├── DATA_REPO_SETUP.md          # One-time setup walkthrough
└── CLOUDFLARE_SETUP.md         # Worker deploy walkthrough
```

## Quick start

```bash
# 1. Install deps
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. Put credentials in .env (gitignored)
cat > .env <<'EOF'
CVU_DB_HOST=mysql.ctbuh.org
CVU_DB_PORT=3306
CVU_DB_USER=...
CVU_DB_PASS=...
CVU_DB_NAME=buldingdb
VUI_DB_HOST=...
VUI_DB_PORT=5432
VUI_DB_USER=...
VUI_DB_PASS=...
VUI_DB_NAME=vui
EOF

# 3. Run a country
ISO=ARE ./run_country.sh && ISO=ARE ./commit_outputs.sh

# 4. Or run the top-50 batch (skips already-catalogued countries)
./run_all_countries.sh

# 5. Open dashboard.html in a browser to review results
```

## The two halves of the pipeline

| Half | Files | When it runs | Cost |
|---|---|---|---|
| **Candidate generation** (slow) | `scan_overture.py`, `scan_gba.py` | Once per country, then again when Overture/GBA publish a new release | Hours per country (GBA tile downloads dominate) |
| **CVU dedupe** (fast) | `pull_cvu.py`, `dedupe.py` | Nightly via GitHub Actions (`resync.sh`), so newly-onboarded CVU buildings drop off the gap list automatically | Seconds per country |

That separation is the whole point: heavy scans run rarely, light CVU comparison runs constantly.

## Reviewer dashboard

`dashboard.html` is a single static HTML file. Hosted publicly via GitHub Pages (or Cloudflare Pages) and gatekept by a Cloudflare Worker proxy (`cvu-dash-proxy/`) that holds a single bot PAT and accepts a shared dashboard password. Reviewers don't need GitHub accounts; the data repo stays private.

See `CLOUDFLARE_SETUP.md` to deploy the Worker, then `dashboard.html` is ready to use.

## Documentation index

- **`HOW_TO_READ_OUTPUTS.md`** — for non-technical reviewers. What each column means, how to triage `likely_new.csv`, common gotchas.
- **`DATA_REPO_SETUP.md`** — one-time setup of the data repo + reviewer access. (Some of this is now folded into the monorepo layout but useful for the "why" of certain design choices.)
- **`CLOUDFLARE_SETUP.md`** — step-by-step Worker deployment.
- **`cities/README.md`** — how to add a per-country city overlay.

## License

Internal CVU use only. GBA-derived heights flowing through this pipeline are CC BY-NC 4.0 (TUM); confirm with legal before any commercial redistribution.
