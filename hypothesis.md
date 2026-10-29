

## Part 3: Compression (SIFT, 1M vectors)

Question: what happens to recall, speed and memory when the number of PQ
sub-vectors `m` is varied, and does exact re-ranking of the top-100 PQ
candidates repair the damage?

Hypotheses:

1. **Plain IVF-PQ has a recall ceiling that rises with m but stays well below
   0.95 for small m.** Increasing `nprobe` only fixes the *coarse* error (the
   right list is visited). It cannot fix the *quantisation* error: with 8-bit
   codes and few sub-vectors, distances are approximated so coarsely that the
   true top-10 are shuffled among many near-equal candidates. I expect the
   ceiling to be low for m=8 (well under 0.6) and clearly higher, but still
   under 0.95, for m=64.
2. **Re-ranking the top 100 with exact distances lifts the ceiling for every m**
   to roughly what IVF-Flat achieves at the same nprobe, because the true
   neighbours are very likely inside the PQ top-100 even when they are not
   in the PQ top-10. The benefit should be largest for small m.
3. **Re-ranking costs speed and gives back most of the memory saving.** It needs
   the original float32 vectors (~512 MB for SIFT) in RAM, which dwarfs the PQ
   codes (e.g. 16 bytes/vector for m=16). I expect QPS to drop noticeably
   (extra gather of 100 vectors per query) and total memory to be dominated by
   the raw vectors, so IVF-PQ+rerank at m=8 uses far more memory than plain
   IVF-PQ at m=64 but beats it on recall.

The hypothesis is *not* supported if: re-ranking leaves the ceiling clearly
below IVF-Flat's at the same nprobe, or plain IVF-PQ reaches 0.95 for some m.

## Part 4: Drifting data (own synthetic data, d=64)

Question: build an index on distribution A, then add vectors from a shifted
distribution B (same mixture of Gaussian clusters, translated by `s * r`, where
r is the typical point-to-centre distance), and query from B. What degrades,
and does retraining recover it?

Varied property (one at a time): shift `s` in {0 (control), 1, 3, 10}.
Everything else is fixed (sizes, cluster structure, seeds, search parameters
tuned once at s=0 to reach ~0.95 recall).

Hypotheses:

1. **Stale IVF-Flat keeps most of its recall but gets much slower.** New points
   are assigned to the nearest *old* centroids, so a few centroids at the
   "edge" of A absorb almost all of B. Queries from B probe those same fat
   lists, so the true neighbours are still found (recall roughly unchanged),
   but each list holds many more vectors, so the number of distances computed
   per query (and so latency) explodes. Retraining on A+B restores balanced
   lists and the original speed.
2. **Stale IVF-PQ loses both recall and speed.** On top of unbalanced lists,
   the PQ codebooks were learned on residuals from A and fit B badly, so
   quantisation error is larger for B vectors. Retraining should recover most
   of both.
3. **HNSW is the most robust**: insertion is local and needs no training, so
   incremental insertion should stay close to a full rebuild at small shifts.
   At the largest shift (B far from A, entry point lives in A) I expect a
   modest recall loss at fixed efSearch, because the greedy search has to
   cross a long path to reach the region where queries land.
4. The **control (s=0)** should show no meaningful difference between stale
   and retrained variants; if it does, something other than drift is
   responsible and the experiment is flawed.

The hypothesis is *not* supported if: stale IVF-Flat loses recall but not
speed, or HNSW degrades more than IVF.
