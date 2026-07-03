# Reflection

**Which fault types were hardest to catch, and why?**

The subtle-tier instances across every pillar, by construction — but two in
particular stood out once I actually measured clean-vs-faulty separation on
the practice and public streams rather than eyeballing the baseline numbers:

- `feature_skew` (subtle): the published baseline
  (`feature_mean_shift_sigma_max = 0.41`) is calibrated tight enough that
  ordinary clean-stream noise pokes just above it sometimes (observed clean
  max ~0.47 across both streams), which cost me two false positives on the
  public run. Every genuine feature_skew instance, including the subtle
  ones, measured well above 1.8, so widening the internal threshold to ~0.9
  removed both false positives with a large margin still separating it from
  real faults.
- `embedding_drift` (subtle): one instance in the public stream had a
  centroid shift of 0.04, only marginally above the observed clean ceiling
  of ~0.039 and *below* the published baseline max (0.0435) entirely. A
  single-metric threshold structurally cannot catch this one without also
  flagging clean noise — the centroid_shift and avg_doc_age_days
  distributions for clean vs. this particular fault overlap almost
  completely on their own. I added a composite z-style score across both
  embedding metrics together (mirroring what I did for `data_batch`'s
  `distribution_shift`, where row_count/mean_amount/null_rate/staleness are
  each individually unremarkable but jointly elevated) which recovers most
  but not quite all of this category — the assignment's framing that some
  instances need "real statistical judgment, not just a threshold check"
  turned out to be literally true for this one, not just a warning.

`missing_upstream`/`orphan_output` (lineage) were conceptually the hardest
up front, since `ctx.baseline` has no expected-shape field for a graph —
solved by learning the majority upstream-set/downstream-count per job
online in `ctx.state` rather than hardcoding an expected graph, but that
approach is inherently weakest in the first few events for a given job,
before a confident majority has accumulated.

**What would you change about your cost/coverage tradeoff, if you had
another pass?**

Every event still pays for exactly one metered call, and I added a soft
budget guard (`budget_remaining()`/`spend_so_far()` are free) that only
starts refusing calls once spend would exceed *double* the budget — since
`cost_overage` in the score formula saturates at 100% overage, spending
past that buys nothing, but stopping earlier (which I tried first) turned
out to cost far more TPR than the mild overage penalty it avoided on a
public stream ~33% longer than the practice one. If I had another pass, I'd
spend the freed-up budget more deliberately rather than uniformly: a second
`lineage_graph_slice` call at `depth=2` when the depth-1 shape looks
borderline, specifically, since lineage is where a single cheap call is
weakest early in a job's history and a confirmation call would directly
address that rather than just buying generic extra margin.