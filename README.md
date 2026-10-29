# Assignment 3: Benchmarking ANN search (HNSW, IVF-Flat, IVF-PQ)

## Layout (project root = parent of `code/`)

```
report.pdf          
code/               
  run_all.sh        one command reproduction
  src/              common.py, part1_exact.py, part2_benchmark.py, part3_compression.py,
                    part4_drift.py (+ its data generator), make_assets.py, download_data.py
  tests/            unit tests for the recall function
  hypothesis.md     hypotheses for Parts 3 and 4 (committed before the experiments)
results/            results.csv, figures/, tables.md, system_info.txt  (full run)
ai_log/             ai_statement.md and the AI prompts/responses
data/               downloaded datasets (NOT submitted; created by run_all.sh)
```

## Setup

No GPU is needed (CPU only: `faiss-cpu`, `hnswlib`). Python 3.9+, ~8 GB RAM minimum, ~3 GB disk.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

## Run

```bash
bash run_all.sh            # full: SIFT 1M + GloVe-100 1.18M. Takes hours; leave the machine idle.
bash run_all.sh --quick    # first 100,000 vectors; writes to ../results_quick; target < 15 min
```

Each script downloads the data if missing, runs the unit tests, then Parts 1-4, then
`make_assets.py`, which writes figures and `tables.md`. `DATA_DIR` / `RESULTS_DIR` environment
variables override the default locations.

## Method summary

* **Distance**: Euclidean everywhere. GloVe vectors (base and queries) are L2-normalised.
* **Ground truth**: exact FAISS `IndexFlatL2`, cached in `data/`. Part 1 checks it against the
  neighbours stored in the HDF5 file (full data only).
* **Recall@10**: `common.recall_at_k` (own implementation). A returned id is correct if its true
  distance is <= the 10th true neighbour's distance (ties are not mistakes); `-1` and repeated ids
  are misses. Tests: `tests/test_recall.py`.
* **Split**: queries 0-4999 are used to *tune* (choose which build setting of each index is best);
  queries 5000-9999 give all reported numbers.
* **Speed**: one query at a time, one thread (`faiss.omp_set_num_threads(1)`, hnswlib
  `num_threads=1`), `time.perf_counter`, GC disabled, median of repeated passes (3 in the final
  stage; the tune stage uses 1 pass over fewer queries; exact search uses the first 300 report
  queries). Recall is computed with a separate batch search. Index building and batch searches
  use all cores, so build times depend on core count.
* **Sweeps** stop when recall >= 0.995 or when recall has gained < 0.002 over three grid points
  (plateau, e.g. the IVF-PQ quantisation ceiling).
* **Part 3** = Compression (SIFT). **Part 4** = drifting data on seeded synthetic Gaussian-mixture
  data (`gen_mixture`, seeds fixed in the file). The search parameter of each Part 4 variant is
  tuned once on the control (shift 0) to reach 0.95 recall, or 98% of the variant's own ceiling
  if it cannot, and is then held fixed.
* `results/results.csv` has one row per (experiment, stage, dataset, index, build, search
  parameter); `extra` holds JSON with extra measurements.

## Caveats to mention in the report

* Single-query timings include Python call overhead (the same for every index). Part 3
  re-ranking is done in NumPy, which adds a little extra overhead compared with C++.
* hnswlib multi-threaded insertion makes graphs slightly non-deterministic between runs.
* Index size is the serialised file size (`faiss.write_index` / `save_index`).
