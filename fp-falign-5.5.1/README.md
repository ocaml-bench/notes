# OCaml 5.5.1: frame pointers and function alignment

Macro benchmark results from the running-ng experiment `macro_5.5.1_falign.yml`.
Six configs: stock, fp, falign, fp+falign, flambda+falign, fp+flambda+falign.
42 macro benchmarks, 5 invocations each.

Each `<machine>/<run>/` holds the per-benchmark `*.json` sidecars, the native
`contract/` (manifest plus olly and perf measurements), `runbms.yml` and
`runbms_args.yml`. Logs are not included. Home directories are redacted.

## Dashboards

Images are `ghcr.io/udesou/ocaml-bench-dashboard:<tag>`:

- monolith (AMD Ryzen 9 9950X, 2026-10-22): `macro-5.5.1-falign-monolith`
- obelisk (Intel Xeon E5-2667 v2, 2026-10-20): `macro-5.5.1-falign-obelisk`
- arm (Arm Neoverse-N1, 2026-10-15): `macro-5.5.1-falign-arm`

```sh
docker run --rm -p 8080:80 ghcr.io/udesou/ocaml-bench-dashboard:macro-5.5.1-falign-obelisk
```

Then open http://localhost:8080. The monolith image is amd64 only; the others
are amd64 and arm64.
