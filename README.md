# PFREDIntegration

Reproducible record of how we run **PFRED** (an antisense-oligo design/scoring tool)
to produce the `PFRED_SVM` / `PFRED_PLS` features used in TAUSO.

PFRED runs as a legacy (Scientific Linux 6 / Python 2.6 / R 2.6) service inside Docker,
so this repo separates the two concerns cleanly:

```
PFREDIntegration/
├── PFRED/                     # submodule → RedPenguin100/PFRED-fork (the Docker harness)
└── integration/
    ├── pfred_runner.py        # tauso-FREE runner: sequences → PFRED_SVM/PFRED_PLS
    ├── sample_seqs.csv        # tiny smoke-test input
    └── results/               # small, diffable results (big ones → Zenodo)
```

## Layout rationale

- **`PFRED/`** is the [fork](https://github.com/RedPenguin100/PFRED-fork) of upstream
  `pfred/pfred-docker`, touched for **fixes only** (see its `CHANGES.md`). It is
  self-contained: its `entrypoint.sh` pulls all 3.85 GB of runtime deps (bowtie indexes,
  tomcat, scripts) from the **fork's own** `v1.0-alpha` release, not upstream.
- **`integration/pfred_runner.py`** is pure Docker orchestration with **no `tauso`
  dependency** — it takes sequences and returns scores. The TAUSO-specific wiring
  (loading the oligo dataset, `index_oligo`, writing to the feature store) lives on the
  **TAUSO side** (`notebooks/competitors/pfred_glue.py`), which imports this runner.

## Reproduce end-to-end

```bash
git clone --recurse-submodules https://github.com/RedPenguin100/PFREDIntegration.git
cd PFREDIntegration

# 1. Build the image (or pull a pinned digest — see Reproducibility note)
docker build -t tauso/pfred:v1 PFRED/

# 2. Start a container named "pfred" (first run downloads deps via entrypoint, ~10 min)
docker run -d -t --name pfred tauso/pfred:v1

# 3. Smoke test the runner
python integration/pfred_runner.py
```

## Reproducibility note

A from-scratch `docker build` still pulls from external mirrors (the SL6 obsolete vault,
CRAN archive, SourceForge), so it can rot. The robust artifact is the **already-built,
initialized image** — push it to a registry by **digest** and pull that instead of
rebuilding. *(TODO: publish `ghcr.io/redpenguin100/pfred@sha256:…` and pin it here.)*

Large outputs (the full scored TAUSO table) go to **Zenodo**; only small results are
committed under `integration/results/`.

## Submodule pin

`PFRED/` is currently pinned to `11a9b39` (the `mk/self-host-deps` tip of PFRED-fork,
which carries the self-hosted-deps fix). Re-pin to the `master` SHA once PFRED-fork
[PR #3](https://github.com/RedPenguin100/PFRED-fork/pull/3) is merged.
