# FDIC Cluster Operational Review

**Cluster:** `9da25159a01a407b944caa5e7a8bd9ef` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 4 total nodes — 3 combined-role hot/ingest/master/remote/search/transform (`himrst`, **7.2 GB heap**, 405 GB disk each) + 1 frozen (`f`, 3.7 GB heap, 600 GB disk). 110 primaries / 225 total shards / 0 unassigned.

This deployment looks essentially idle from a data perspective. Three combined-role data nodes with smaller heap (7.2 GB) and no dedicated masters. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

**There are no oversized shards in this cluster — there is essentially no data.**

Every single index has either zero documents or a primary store that rounds to 0 GB. The only indices with meaningful document count are `cdm_cpe_dictionary` (1.49M docs) and `data_dictionary` (3,858 docs); both are small. Everything else — every CDM application index, every trending datastream, every Kibana alerting index — is empty.

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_cpe_dictionary` | 1 | <1 GB (≈250 MB) | <1 | OK |
| Every other `cdm_*` and `.ds-cdm_*` | 1 | 0 GB | 0 | empty |
| `.kibana_*`, `.security*`, `.tasks`, `.transform-*` | 1 | 0 GB | 0 | system-tiny |

**No resharding is warranted on the data side**, because there is no data to reshard. Every CDM index is at `pri=1, rep=1`, which is the right default for 0 GB content.

The smaller hot nodes (7.2 GB heap) here imply tighter sizing math when data does arrive: the heap-based shard ceiling is ~144 shards/node. Primary count discipline matters more here than on the bigger 14.7 GB or 29.6 GB heap deployments.

## 2. Is the cluster oversharded?

**No.** Against the Elastic 20-shards-per-GB-heap heuristic:

- 225 total shards across 3 hot nodes = **75 shards per hot node**.
- Heap on each hot node: 7.2 GB → ceiling of ~144 shards/node.
- Current utilization: ~52.1% of the recommended limit.

The cluster is within Elastic's recommended shard density.

For context, the shard count breaks down as:

1. **Enrich indices replicated to every hot node.** 5 enrich policies (`cdm_config_t0`, `cdm_software_directive`, `cdm_csm_stig`, `cdm_software_dictionary_eol`, `cdm_vuln_cve`) × 3 hot nodes = ~15 enrich shards.
2. **Empty CDM and Kibana alerting indices** that are pre-provisioned but never written to.

The relevant trajectory observation: if/when data starts flowing in and ILM begins creating new monthly trending backing indices, shard count will climb. With 7.2 GB heap, the per-node ceiling is ~144 shards — there is room for growth, but less absolute room than a 14.7 GB or 29.6 GB heap topology would provide.

## 3. Empty indices & their sources

This deployment is almost entirely empty — only `cdm_cpe_dictionary`, `data_dictionary`, `.cdm_config_history`, and a handful of small Kibana/SLM/system indices have any content. Every CDM application index, every CDM trending datastream, and every Kibana alerting index is at 0 docs.

Categorized:

**Kibana Alerting framework (15 indices)** — full `.internal.alerts-*-default-000001` family covering security, ML anomaly detection (×2), observability (metrics/logs/uptime/threshold/slo/apm), streams, stack, transform health, dataset quality, attack discovery, default. All created on **2026-03-30**. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**CDM trending datastream backers (10 indices)** — all `.ds-cdm_*_trending-2026.03.31-000001`. Every single one is the *first* generation backing index, all created on **2026-03-31** within a tight window. None has rolled over, none has data.

**CDM application indices (24 indices)** — `cdm_vuln`, `cdm_vuln_remediated`, `cdm_vuln_trending-6-000001`, `cdm_csm_trending-6-000001`, `cdm_swam_trending-6-000001`, `cdm_hwam_trending-6-000001`, `cdm_device_scan_trending-6-000001`, `cdm_scan_trending-6-000001`, `cdm_system_boundary_trending-6-000001`, `cdm_cyhy_*`, `cdm_scuba_m365_*`, `cdm_scuba_gws_*`, `cdm_asm_nmi`, `cdm_vdp`, `cdm_threat`, `cdm_hwam`, `cdm_software_dictionary`, `cdm_kev_dictionary`, `cdm_stig_dictionary`, `cdm_cve_dictionary`, `cdm_directive_dictionary`, `cdm_benchmark_vuln`, `cdm_system_boundary_fisma_1163_rollup`. **All zero docs.** Even the dictionary indices that should be populated by their respective seed/sync jobs (CVE, software, KEV, directive, STIG) are all empty.

**Trending naming convention.** All trending base indices use `-6-000001` (six-digit, generation 1) — `cdm_vuln_trending-6-000001`, `cdm_swam_trending-6-000001`, etc. Consistent within this deployment. Worth noting if shared tooling targets these by exact-name pattern.

**Other:** `.kibana_locks-000001`. **No `.elastic-connectors*` or `.ent-search-*` indices** — good, this cluster skipped Enterprise Search and is in good shape for the eventual 9.0 upgrade.

**Frozen tier:** The single frozen node (`instance-0000000005`) shows **0 shards, 0 bytes of disk indices**. It reports `disk.used_percent: 90.04%` — that is an ESS reporting artifact (overcommit math), not a real signal. Nothing has aged into frozen yet.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts, consistent with the frozen node carrying no shards.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This cluster looks like a **freshly provisioned environment that has not yet started receiving data**. The evidence is consistent and clean:

**The 2026-03-31 creation timestamp pattern.** Every empty CDM index has a `creation.date.string` of **2026-03-31**, all within a roughly 50-minute window (`15:21:08` to `16:07:49`). The Kibana alerting framework was created the day before (**2026-03-30**). This is a single bulk provisioning event from about five weeks ago. A healthy ingesting cluster would show staggered creation timestamps reflecting rollover events; here the templates were installed and the cluster has been idle since.

**Enrich policies were rebuilt very recently.** Most enrich indices are timestamped from `1774970...` (which is **2026-03-31**, same as provisioning), but `.enrich-cdm_config_t0-1774971043844` is also **2026-03-31**. The `.enrich-cdm_vuln_cve-1775145552864` and `.enrich-cdm_software_directive-1775145553303` are stamped **2026-04-02**. So enrich rebuilds did happen briefly after provisioning, but nothing recent. This is consistent with no data flow.

**No ILM activity beyond housekeeping.** `.ds-ilm-history-7-*` reaches only `-000003` after five weeks. The `.ds-.slm-history-7-*` series shows steady weekly snapshots happening on schedule, which is the only meaningful activity on this cluster.

**Heap utilization is variable but moderate.** `instance-0000000001` (elected master + himrst): 61% heap, 59% RAM. `instance-0000000003`: 19% heap, 60% RAM. `instance-0000000004`: 38% heap, 59% RAM. The 61% heap on the elected master is interesting given there's effectively no workload — possibly the cost of cluster-state churn for housekeeping, or fanout from being the single master coordinator.

**No master CPU concerns.** All three nodes show `load_5m` between 0.96 and 2.05 — quiet. `instance-0000000004` at 2.05 load_5m is the highest but still benign for a 7.2 GB heap node.

**Disk usage is minimal.** All hot nodes ~0.1-0.2% disk used out of 405 GB. Plenty of room.

**Frozen tier is unused.** A single 600 GB frozen node sitting empty.

---

## Suggested actions

1. **Verify expected state.** Is this cluster supposed to be ingesting? If yes, this is the headline finding — every CDM ingestion path is silent. Five weeks of empty trending datastreams (still on backing index `-000001`) suggests no documents have ever arrived.
2. **If this cluster should be receiving data:** check the upstream Logstash / pipeline configuration pointing at this cluster. Given the past CDM pipeline silent-failure pattern (`Connection reset by peer`), this looks like the same failure mode at scale.
3. **Investigate the dictionary indices.** `cdm_cve_dictionary`, `cdm_kev_dictionary`, `cdm_software_dictionary`, `cdm_stig_dictionary`, `cdm_directive_dictionary`, `cdm_threat` should all be populated by their respective seed/sync jobs. All are empty. These are usually pulled by a separate scheduled job, not by Logstash — worth checking whether those refresh jobs even target this cluster.
4. **Note the smaller heap topology.** With `himrst` combined-role nodes at 7.2 GB heap and the cluster already at 52% of the per-heap shard ceiling while empty, this deployment has relatively little room to grow before shard count discipline becomes a real concern. Cleanup of empty system indices that aren't strictly needed (e.g., SIEM rule migrations if no migration is planned) could help.
5. **Frozen tier is overprovisioned for current state.** A 600 GB frozen node for 0 frozen shards. If frozen retention is part of the long-term plan, leave it. If not, downsizing would save cost.
6. **No data-loss exposure** — every index has rep=1.
7. **No master CPU or load anomalies** to investigate on this cluster.
