# OxCaml main vs OCaml 5.4.1

Macro benchmark results comparing OxCaml main (0c3b79a) with OCaml 5.4.1, with
and without flambda. 42 macro benchmarks (default and large inputs), 5
invocations each, on monolith (AMD Ryzen 9 9950X).

- `default-gc/`: every runtime with its default GC parameters (2026-10-29).
- `oxcaml-gc/`: 5.4.1 and 5.4.1 flambda with OxCaml's GC parameters
  (`s=1M,o=80,m=400,n=10`); OxCaml unchanged (2026-10-30).

Each run folder holds the per-benchmark `*.json` sidecars, the native
`contract/`, `runbms.yml` and `runbms_args.yml`. Logs are not included. Home
directories are redacted.

## coqc cells

The `coqc_tree_default` and `coqc_tree_large` cells come from reruns on
2026-11-04, after fixing the benchmark (macro-benches#70, running-ng#60), and
replace the original ones in both runs. Before the fix, every runtime ran the
same stock-built `rocqworker`, with Rocq's own GC settings turned off. The new
inputs are larger (tree depth 20 and 21 instead of 18 and 19). The rerun's
`runbms.yml` and `runbms_args.yml` are in `coq-rerun/`.

Rocq sets its own GC parameters on every runtime (32M-word minor heap,
`space_overhead=200`). So in `oxcaml-gc/`, the coqc cells for 5.4.1 and 5.4.1
flambda use `s=32M,o=200,m=400,n=10` (modifier `oxgc_rocq`), which is what
OxCaml runs with under Rocq.

## Dashboards

Images are `ghcr.io/udesou/ocaml-bench-dashboard:<tag>` (amd64):

- `macro-oxcaml-vs-5.4.1-monolith` (`default-gc/`)
- `macro-oxcaml-vs-5.4.1-oxgc-monolith` (`oxcaml-gc/`)

```sh
docker run --rm -p 8080:80 ghcr.io/udesou/ocaml-bench-dashboard:macro-oxcaml-vs-5.4.1-monolith
```

Then open http://localhost:8080.
