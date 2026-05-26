# Reproducing PFRED scores

Record of building the image, running the container, and verifying it reproduces the
`PFRED_SVM` / `PFRED_PLS` features (against TAUSO's `data_scored_pfred_final.csv`).
Done 2026-05-26 on an Ubuntu box (kernel 6.8).

## 1. Build the image (from the fork @ 11a9b39)

```bash
docker build -t tauso/pfred:v1 PFRED/
```
Multi-stage build (~185 MB image): compiles numpy 1.4.1, R 2.6.0, rpy 1.0.2 and the
R packages, then a slim SL6 runtime stage.

## 2. Run the container

```bash
docker run -d -t --name pfred tauso/pfred:v1
```
On first start, `entrypoint.sh` downloads the runtime deps **from this fork's own
release** (`RedPenguin100/PFRED-fork` `v1.0-alpha`): tomcat, the pfred/oligowalk/
ensemblapi scripts, and the bowtie indexes. The `pfred` scripts (incl.
`antisense_predictor.py` + the `AOBase_...csv` model) arrive early; the 3.79 GB bowtie
indexes are last and are **not needed** for SVM/PLS scoring.

## 3. Score sequences

`integration/pfred_runner.py` feeds cleaned sequences (≥16 nt, U→T, non-ACGT stripped)
to the predictor via `docker exec`:

```
python2 antisense_predictor.py AOBase <AOBase_...csv> c_a_thermo predict <input.csv>
```
with params `15 / 21 / 100 / 1000 / 12`, and reads back `SVMpred` → `PFRED_SVM`,
`PLSpred` → `PFRED_PLS`.

## 4. Verification

Re-scored 8 sequences sampled from the 15,151 unique cleaned sequences in
`data_scored_pfred_final.csv`. Result: **8/8 exact, `max|Δ| = 0` for both SVM and PLS.**

## Environment notes

- **vsyscall:** SL6's old glibc needs the legacy vsyscall page. This host is
  `CONFIG_LEGACY_VSYSCALL_XONLY` (not `NONE`), so the container runs without any GRUB
  change. On WSL2 kernels built with `NONE`, add `vsyscall=emulate` via `.wslconfig`
  (see `PFRED/`'s notebook notes), or the python2/R binaries segfault (exit 139).
- The bare image re-downloads the 3.85 GB deps on each fresh run (from the fork release).
  The fully self-contained artifact is the `docker commit` of the initialized container,
  archived on GHCR by digest:
  `ghcr.io/redpenguin100/pfred@sha256:738012805a446453fc1bd06bf3b63f4e0f9197a0f8d8f2919a2396545c0cfdd8`
  (`docker pull` it to run offline). Built from PFRED-fork @ 11a9b39.
