# ES|QL Datasources

Benchmarks the ES|QL `EXTERNAL` command across a matrix of (storage backend x file format x codec) source variants. Across the active variants every dimension cell — 3 formats, 3 codecs, 3 backends — is exercised at least once, without a Cartesian explosion.

## Layout

~76 EXTERNAL ops per challenge run, organised as:

- **Anchor** — 31 trimmed benchmark queries x the canonical source `H2` (Parquet / ZSTD / S3-anon) = **31 ops**. Tracks the full benchmark-suite signal on a stable layout.
- **Cross-source subset** — 8 representative queries x the 5 non-canonical sources = **40 ops**. Surfaces format/codec/backend differences per operator class.
- **External-basic gap-fillers** — 4 queries (DISSECT, GROK, MV_EXPAND, INLINE STATS) x `H2` = **4 ops**. Covers ES|QL features the benchmark suite doesn't exercise.
- **Cross-EXTERNAL LOOKUP JOIN** — 1 query x 1 NDJSON nyc_taxis source (`N1` GCS) = **1 op**. EXTERNAL left side x indexed `nyc_payment_types` lookup right side.

Plus one setup bulk op that loads the `nyc_payment_types` lookup index (5 docs).

## Source catalogue

| id | dataset     | format  | codec  | backend | auth       | URI | WITH clause |
| -- | ----------- | ------- | ------ | ------- | ---------- | --- | ----------- |
| H1 | hits        | Parquet | ZSTD   | HTTP    | anonymous  | `https://clickhouse-public-datasets.s3.amazonaws.com/hits_compatible/hits.parquet` | (none) |
| H2 | hits        | Parquet | ZSTD   | S3      | anonymous  | `s3://clickhouse-public-datasets/hits_compatible/hits.parquet` (canonical anchor) | `WITH { "auth": "none" }` |
| H3 | hits        | Parquet | ZSTD   | S3      | anonymous  | `s3://clickhouse-public-datasets/hits_compatible/athena_partitioned/hits_*.parquet` | `WITH { "auth": "none" }` |
| H4 | hits        | NDJSON  | gzip   | HTTP    | anonymous  | `https://datasets.clickhouse.com/hits_compatible/hits.json.gz` | (none) |
| H5 | hits        | CSV     | gzip   | HTTP    | anonymous  | `https://datasets.clickhouse.com/hits_compatible/hits.csv.gz` | (none) |
| N1 | nyc_taxis   | NDJSON  | bzip2  | GCS     | IAM        | `gs://es-perf-mirror-<region>/rally-tracks/nyc_taxis/documents.json.bz2` | dynamic — see below |
| N2 | nyc_taxis   | Parquet | Snappy | Azure   | anonymous  | `wasbs://nyctlc@azureopendatastorage.blob.core.windows.net/yellow/puYear=*/puMonth=*/part-*.parquet` | `WITH { "auth": "none" }` |

The WITH clause is required on H2/H3/N2 to opt out of any default credential chain on the cluster (`auth=none` for truly-anonymous public buckets). HTTP sources (H1, H4, H5) need no WITH at all. **N1** uses static credentials passed through track-params (`gcs_credentials` etc., see Track parameters below); when those track-params are not set, the WITH clause is omitted entirely so the plugin falls back to its native credential resolution path (Application Default Credentials on a GCE VM, etc.) — that's the right behaviour on the GCP-hosted production nightly where the cluster's GCE service account already has read on the mirror bucket.

The `s3://es-perf-mirror-<region>/...` mirror was previously available as a separate source variant relying on an AWS EC2 instance profile for credentials; it was dropped because the production nightly runs on GCP (no AWS instance profile available) and adding a long-lived static AWS access key purely for benchmark coverage wasn't worth the security overhead. The S3 backend is still exercised via H2/H3 (anonymous public bucket).

> [!NOTE]
> Against current `9.5.0-SNAPSHOT` builds, N2 (Azure-anon) fails with HTTP 401 from Azure even though the `nyctlc` container is anonymously readable via plain HTTPS — `esql-datasource-azure`'s anonymous code path builds a `BlobServiceClient` without an explicit anonymous credential, and the Azure SDK Java client doesn't truly skip auth in that mode. The track configuration matches the plugin's documented contract; tracking the plugin behaviour separately.

Set `mirror_region` (track param) to match the cluster's region (default: `europe-west1`).

## Queries

The benchmark queries cover real-world analytics shapes against the `hits` web-log dataset. Gap-fillers come from [`x-pack/plugin/esql/qa/testFixtures/src/main/resources/external-basic.csv-spec`](https://github.com/elastic/elasticsearch/blob/main/x-pack/plugin/esql/qa/testFixtures/src/main/resources/external-basic.csv-spec).

Operation names are `<source-id>__<query-id>` so dashboard filters can slice by source, query, or the `meta.{source-id,query-id,query-group,format,codec,backend,auth,dataset}` fields.

The 31 benchmark queries by operator class:

- **Plain count / point lookup** — `count_hits`, `count_advengineid_nonzero`, `filter_userid_435090932899640449`
- **Wide arithmetic aggregation** — `stats_resolution_width_sums` (90 `SUM`s in one `STATS`)
- **Distinct cardinality** — `stats_distinct_userid`, `stats_distinct_userid_by_region`, `stats_searchphrase_users`
- **Min/Max** — `stats_min_max_eventdate`
- **Group-by, low cardinality** — `stats_advengineid_by_advengineid`, `stats_region_aggregates`, `stats_searchphrase_count`
- **Group-by, high cardinality** — `stats_url_count`, `stats_userid_count`, `stats_watchid_clientip_all`, `stats_search_engine_clientip`
- **Top-N** — `sort_searchphrase_asc`, `sort_google_url_by_time`
- **Filter + group-by** — `stats_advengineid_resolution`, `stats_clientip_variations`, `stats_const_url_count`, `stats_url_length_by_counter`, `stats_referer_domain_analysis`, `stats_search_engine_phrase`, `stats_userid_searchphrase_count`, `stats_userid_minute_searchphrase`, `stats_mobile_phone_users`, `stats_google_url_count`
- **Time bucketing** — `stats_counter62_july14_15_minutes`, `stats_counter62_july_links`, `stats_counter62_july_titles`, `stats_counter62_july_traffic_sources`

8-query cross-source subset: `count_hits`, `filter_userid_435090932899640449`, `stats_distinct_userid`, `stats_region_aggregates`, `stats_url_count`, `sort_searchphrase_asc`, `stats_clientip_variations`, `stats_counter62_july14_15_minutes`.

4 gap-fillers (on H2 only): `dissect_url_host`, `grok_referer`, `mv_expand_url_parts`, `inline_stats_max_resolution`.

1 LOOKUP JOIN (on N1 only): `lookup_join_payment_types`.

## Cluster prerequisites

The cluster running this track needs:

1. The `esql-datasource-*` plugins installed: `http`, `s3`, `gcs`, `azure`, `parquet`, `ndjson`, `csv`, `gzip`, `bzip2`, `zstd`. The nightly benchmark entry in `elasticsearch-benchmarks/benchmarks/automated/nightlies/production/stateful/main.yml` sets `elasticsearch.plugins:` explicitly. For releases where any of these are bundled as modules, the redundant plugin installs are a no-op; prune as needed when they ship inside the distribution.
2. The ES|QL external-datasources feature flag enabled via the JVM system property **`es.esql_external_datasources_feature_flag_enabled=true`**. Set per-benchmark via `elasticsearch.jvm.options` in `main.yml`. Without it, every `EXTERNAL ...` query fails at parse/plan time.
3. Credentials for `N1` (GCS internal mirror) — supplied either via the cluster's native chain (Application Default Credentials on a GCE VM with read on the mirror bucket) or via the `gcs_credentials`/`gcs_project_id`/`gcs_endpoint` track-params (service-account JSON, e.g. resolved by esbench from Vault path `/secret/performance/employees/cloud/buckets/gcs/performance-testing-artifacts:service_account`). When neither is present the WITH clause is empty and the operation will fail at access time.
4. `N2` uses anonymous Azure Open Datasets — no credential setup needed.

## Track parameters

| param               | default          | purpose |
| ------------------- | ---------------- | ------- |
| `mirror_region`     | `europe-west1`   | Region suffix for the GCS mirror bucket (`es-perf-mirror-<region>`, N1). |
| `gcs_credentials`   | (empty)          | Service-account JSON for the GCS mirror bucket (N1). When set, populates the WITH clause; when empty, the WITH is omitted and the plugin's default credential resolution applies. |
| `gcs_project_id`    | (empty)          | Optional project ID for N1 (rarely needed when `gcs_credentials` already binds to a project). |
| `gcs_endpoint`      | (empty)          | Optional non-default GCS endpoint for N1. |
| `azure_account` / `azure_key` / `azure_sas_token` / `azure_connection_string` / `azure_endpoint` | (empty) | Azure-side credentials, only relevant if N2 is re-enabled with non-anonymous access. |
| `anchor_source_id`  | `H2`             | Source running the full 31-query benchmark suite. |
| `query_clients`     | `1`              | Rally clients per query op. |
| `query_iterations`  | `1`              | Rally iterations per query op. |
| `query_warmup`      | `1`              | Rally warmup iterations per query op. |
| `request_timeout`   | `1800`           | Per-op client-side timeout (s) for EXTERNAL queries. A single `COUNT(*)` over the 14 GB `hits.parquet` measured ~5 min locally; raise if heavier queries (`stats_resolution_width_sums`, full-scan group-bys) push past this. The Rally connection-level timeout (`--client-options=timeout:N` or `client.options.default.timeout` in `main.yml`) must be `>=` this value. |
| `cluster_health`    | `green`          | `wait_for_status` value in the cluster-health setup op. |

## Smoke-test checklist (before scheduling nightlies)

- [ ] **Feature flag present**: `curl $ES/_nodes/jvm | grep esql_external_datasources_feature_flag_enabled` returns true on every node.
- [ ] **Plugins installed**: `curl $ES/_cat/plugins?v` lists all `esql-datasource-*` plugins (or confirms they're bundled as modules via `_nodes/settings`).
- [ ] **`.bz2` over GCS / mirror**: verify the codec auto-detects from the `.bz2` extension on the GCS-streamed N1 source — if not, the N1 corpus needs either a `.gz`-rendered variant or a GCS-native streaming path.
- [ ] **N1 credentials**: `EXTERNAL "gs://es-perf-mirror-<region>/rally-tracks/nyc_taxis/documents.json.bz2" | LIMIT 1` returns a row (with credentials provided either via track-params or the cluster's default GCS chain).
- [ ] **N2 Azure anonymous**: `EXTERNAL "wasbs://nyctlc@azureopendatastorage.blob.core.windows.net/yellow/puYear=2019/puMonth=1/part-*.parquet" WITH { "auth": "none" } | LIMIT 1` returns a row without credentials.
- [ ] **Matrix op count**: `esrally info --track=esql-datasources --challenge=esql_datasources` reports 77 ops (76 EXTERNAL + 1 bulk setup).

## v2 follow-ups (separate PRs)

- Parquet layout variants (`1-file`, `50-files`, `1rg-per-file`) mirrored into `rally-tracks` so they're public — exposes the file-parallelism vs row-group-parallelism axis.
- ORC and Iceberg sources once `esql-datasource-iceberg` public test fixtures exist.
- Optional `FROM` baseline as a separate challenge if the EXTERNAL-vs-FROM overhead becomes interesting to track over time.
