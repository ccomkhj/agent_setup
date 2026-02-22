---
name: data-pipeline-benchmarking
description: Benchmark data_pipeline outputs between main and a feature branch without changing application code by running each branch in isolated git worktrees, snapshotting canonical S3 outputs for a fixed client/env/ds, and comparing manifests or parquet outputs. Use when output paths are overwritten in S3 and you need reliable AS-IS vs new-branch validation for data_pipeline/executables/products/selectable.py.
---

# Data Pipeline Benchmarking

## Overview
Use this skill to benchmark `main` vs feature-branch outputs in S3 while preserving reproducibility and avoiding code changes in `data_pipeline/`.

Run both branches sequentially against the same canonical path, snapshot each run immediately to a benchmark namespace, then compare snapshots.

## Required Inputs
- `CLIENT`: target client name
- `ENV`: usually `dev`
- `DS`: fixed date partition in `YYYY-MM-DD`
- `FEATURE_BRANCH`: branch to compare against `main`
- `SCHEMA_VERSION`: usually `1`

## Workflow
1. Set a fixed benchmark context.

```bash
CLIENT="e-comerce-1"
ENV="dev"
DS="2026-02-11"
FEATURE_BRANCH="dse-670-my-change"
SCHEMA_VERSION="1"

BENCH_ID="$(date +%Y%m%d-%H%M%S)-${CLIENT}-${DS}"
BUCKET="bucket-name-${ENV}"
CANONICAL_BASE="s3://${BUCKET}/${CLIENT}"
BENCH_BASE="s3://${BUCKET}/benchmark-runs/${BENCH_ID}"

WT_DIR=".worktrees"
MAIN_WT="${WT_DIR}/benchmark-main"
FEATURE_WT="${WT_DIR}/benchmark-${FEATURE_BRANCH//\//-}"
```

2. Create isolated worktrees (pattern from `using-git-worktrees`).

```bash
# Ensure project-local worktree dir is ignored
if ! git check-ignore -q .worktrees; then
  echo "ERROR: .worktrees must be in .gitignore before use" >&2
  exit 1
fi

# Create worktrees
[ -d "${MAIN_WT}" ] || git worktree add "${MAIN_WT}" main
[ -d "${FEATURE_WT}" ] || git worktree add "${FEATURE_WT}" "${FEATURE_BRANCH}"
```

3. Define helper functions.

```bash
purge_ds_partition() {
  for layer in bronze silver gold platinum; do
    aws s3 rm "${CANONICAL_BASE}/${layer}/v${SCHEMA_VERSION}/" \
      --recursive \
      --exclude "*" \
      --include "*/ds=${DS}/*"
  done
}

run_products_pipeline() {
  local wt="$1"
  (
    cd "${wt}" || exit 1
    python data_pipeline/executables/products/selectable.py \
      --client_name "${CLIENT}" \
      --layers bronze silver \
      --env "${ENV}" \
      --ds "${DS}"

    python data_pipeline/executables/products/selectable.py \
      --client_name "${CLIENT}" \
      --layers gold platinum \
      --env "${ENV}" \
      --ds "${DS}"
  )
}

snapshot_ds_partition() {
  local label="$1"
  for layer in bronze silver gold platinum; do
    aws s3 sync "${CANONICAL_BASE}/${layer}/v${SCHEMA_VERSION}/" \
      "${BENCH_BASE}/${label}/${CLIENT}/${layer}/v${SCHEMA_VERSION}/" \
      --exclude "*" \
      --include "*/ds=${DS}/*"
  done
}
```

4. Snapshot current canonical output before benchmarking (for optional restore).

```bash
snapshot_ds_partition "pre-benchmark"
```

5. Run `main`, then snapshot.

```bash
purge_ds_partition
run_products_pipeline "${MAIN_WT}"
snapshot_ds_partition "main"
```

6. Run feature branch, then snapshot.

```bash
purge_ds_partition
run_products_pipeline "${FEATURE_WT}"
snapshot_ds_partition "feature"
```

7. Compare snapshot manifests.

```bash
TMP_DIR="/tmp/data-pipeline-benchmark/${BENCH_ID}"
mkdir -p "${TMP_DIR}/main" "${TMP_DIR}/feature"

for label in main feature; do
  aws s3 sync "${BENCH_BASE}/${label}/${CLIENT}/" "${TMP_DIR}/${label}/" \
    --exclude "*" \
    --include "*/ds=${DS}/manifest.json"
done

for label in main feature; do
  find "${TMP_DIR}/${label}" -name manifest.json -print0 \
    | xargs -0 -I{} jq -r '"\(.layer)|\(.entity)|rows=\(.record_count)|git=\(.model_versions.git)|pkg=\(.model_versions.package)"' "{}" \
    | sort > "${TMP_DIR}/${label}-manifest-summary.txt"
done

diff -u "${TMP_DIR}/main-manifest-summary.txt" "${TMP_DIR}/feature-manifest-summary.txt" || true
```

8. Optional restore to pre-benchmark state.

```bash
# Use only when you want to put canonical S3 paths back
purge_ds_partition
for layer in bronze silver gold platinum; do
  aws s3 sync "${BENCH_BASE}/pre-benchmark/${CLIENT}/${layer}/v${SCHEMA_VERSION}/" \
    "${CANONICAL_BASE}/${layer}/v${SCHEMA_VERSION}/" \
    --exclude "*" \
    --include "*/ds=${DS}/*"
done
```

## Guardrails
- Use a fixed `DS`; do not use floating "today" dates for benchmarks.
- Keep `ENV`, `CLIENT`, and source-account configuration identical between runs.
- Run sequentially; do not run both branches at the same time.
- Always purge the target `ds` partition before each branch run, otherwise "already exists" checks can skip compute.
- Default to `dev`. Use `prod` only with explicit approval.
- Do not modify `data_pipeline/` code for this workflow.

## Expected Result
- Canonical output path is reused only during execution.
- Snapshot paths preserve branch-specific results:
  - `s3://bucket-name-${ENV}/benchmark-runs/${BENCH_ID}/main/...`
  - `s3://bucket-name-${ENV}/benchmark-runs/${BENCH_ID}/feature/...`
- Manifest diff shows per-entity differences in row counts and model versions.
