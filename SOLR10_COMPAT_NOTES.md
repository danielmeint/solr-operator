# Solr 10 Compatibility Notes

## What changed in Solr 10 that breaks the operator

### solr.xml (primary blocker)
Solr 10 removed several `<solrcloud>` configuration parameters. If present, Solr refuses to start with `SolrException: Unknown configuration parameter`.

Removed:
- `genericCoreNodeNames` — always true now
- `hostContext` — always "solr"
- `allowPaths` — removed entirely
- `metricsEnabled` — metrics always enabled

Still required but renamed sysprop:
- `host` — XML element still required, but the system property changed from `host` to `solr.host.advertise`. Setting `SOLR_HOST` still works (with deprecation warning), `SOLR_HOST_ADVERTISE` is the new env var.

### Module loading
`/opt/solr/contrib/<module>/lib` and `/opt/solr/dist` no longer exist. Modules must be loaded via the `SOLR_MODULES` env var (comma-separated list).

### zkcli.sh removed
`/opt/solr/server/scripts/cloud-scripts/zkcli.sh` is gone. Use `solr zk` subcommands instead. The operator already uses `solr zk` in most places; only `setUrlSchemeClusterPropCmd` (TLS setup) still used zkcli.sh.

### CLI changes
`solr api` syntax changed: `-get URL` / `-post URL` replaced with `--solr-url URL` (GET only). `solr create_collection` removed entirely — use the Collections API directly.

## What this changeset fixes

| Area | Approach |
|------|----------|
| solr.xml | Version-conditional template (`DefaultSolrXMLForSolr10`) selected via `IsSolr10OrLater()` |
| Host advertise | Added `SOLR_HOST_ADVERTISE` env var for Solr 10 pods |
| Modules | Added `SOLR_MODULES` env var; skip contrib sharedLib paths for Solr 10 |
| hostPort sysprop | Skip `-DhostPort` for Solr 10 (no longer needed) |
| zkcli.sh (TLS) | `setUrlSchemeClusterPropCmd` uses `solr zk cp` for Solr 10 |
| E2E tests | `callSolrApiInPod` uses `--solr-url` flag for Solr 10 |

## Version detection

`IsSolr10OrLater(imageTag string) bool` parses the major version from the Solr image tag. Returns false for unparseable tags (e.g. "latest", "nightly") — unknown versions are treated as pre-10 for backwards compatibility.

## Known remaining work

- **Prometheus Exporter**: `DefaultPrometheusExporterEntrypoint` references `/opt/solr/contrib/prometheus-exporter/bin/solr-exporter` which doesn't exist in Solr 10. The exporter was removed; metrics should be scraped directly from Solr's built-in metrics endpoint. See issue #820.
- **Deprecation warning**: `SOLR_HOST` env var triggers `"You are passing in deprecated system property host"` in Solr 10 logs. Harmless but noisy. Could be resolved by only setting `SOLR_HOST_ADVERTISE` for Solr 10.
- **More E2E coverage**: Only the basic test (start + create collection) has been verified. Scaling, TLS, backups, and security tests need validation.
- **`solr api` auth**: The Solr 10 CLI has native `--credentials` support. The current test workaround using `JAVA_TOOL_OPTIONS` may not work; tests with basic auth enabled will likely need updating.
- **Solr 10.1.0 targeting**: HoustonPutman suggested targeting 10.1.0 due to `-c` flag removal in `solr create`. This mainly affects the CLI, not the operator's API-based approach.

## E2E test results

| Solr Version | Basic Test | Notes |
|---|---|---|
| 9.8.0 | PASS | Baseline verification |
| 10.0.0 (before changes) | FAIL | `SolrException: Unknown configuration parameter genericCoreNodeNames` |
| 10.0.0 (after changes) | PASS | 1 Passed, 0 Failed, 62s |

## Upstream context

- PR #826 (tomaszpolachowski) — partial fix covering solr.xml + SOLR_HOST_ADVERTISE only
- Issue #820 — prometheus exporter removal
- Issue #821 — HoustonPutman confirmed "Solr 10.0 - No tests pass"
- v1.0.0 milestone (due 2026-05-01) — "Next major version supporting Solr 10.0"
