# NASA Cluster Operational Review

**Cluster:** `72480ae364fd4199bdf01ee6b8e95891` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 10 total nodes — 3 master (`mr`, 1.7 GB heap), 6 hot/ingest/remote/search/transform (`hirst`, 29.6 GB heap, ~1.5 TB disk each), 1 frozen (`f`, 3.7 GB heap, 600 GB disk). 165 primaries / 354 total shards / 0 unassigned.

This deployment has actual production data flowing for some CDM domains. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here is the per-primary picture for the indices that carry data:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_cve_dictionary` | 1 | 1 GB | 1.0 | ✅ tiny dictionary, fine |
| `cdm_software_dictionary` | 1 | <1 GB | <1 | ✅ fine |
| `cdm_cpe_dictionary` | 1 | <1 GB (≈250 MB) | <1 | ✅ fine |
| `cdm_threat`, `cdm_stig_dictionary` | 1 | 0 GB | 0 | small reference data |
| All `.ds-cdm_cyhy_domain_trending-*` | 1 | 0 GB | 0 | active but small |
| All other CDM indices | 1 | 0 GB | 0 | empty |

**There are no oversized shards in this cluster.** The largest primary anywhere is `cdm_cve_dictionary` at 1 GB on a single primary — well under the 40 GB target. Most data-bearing indices are dictionaries (CVE, software, CPE, STIG) or trending datastreams with relatively small daily volumes.

**No resharding is warranted.** Single-primary configs are appropriate for the current data volumes. If `cdm_cyhy_domain_trending` continues to grow at its observed rate (the 2026-04-15 backing index is at 27K docs and the 2026-03-16 one is at 79K), the primary count for that datastream might eventually need revisiting — but the threshold is far off.

## 2. Is the cluster oversharded?

**Yes, mildly.** The numbers:

- 354 total shards across 6 hot nodes = **~59 shards per hot node**. Against a ceiling of ~592 shards/node (20 shards per GB of 29.6 GB heap), the cluster is at ~10% of the safe limit.
- 165 primaries holding effectively a few GB of real data.

The shard count is dominated by:

1. **Enrich indices replicated to every hot node.** There are 5 enrich policies (`cdm_software_dictionary_eol`, `cdm_software_directive`, `cdm_cdo_temp`, `cdm_config_t0`, `cdm_csm_stig`, `cdm_vuln_cve`) × 6 nodes = ~36 shards just for enrich. About 10% of all shards.
2. **A long tail of trending datastream backing indices** that haven't been deleted by ILM yet — see §5 for specifics.
3. **A growing herd of daily `cdm_benchmark_vuln_trending_YYYY.MM.DD` indices** — these are not datastreams; they are individually-named daily indices, one per day, each at 1 doc with rep=1 = 2 shards each. There are 11 of them in the snapshot (April 22 through May 2), and the pattern suggests a new one is created every day with no consolidation. Over a year that is ~730 shards just for these.

The cluster is nowhere near its shard-count limit, but the trending pattern in particular is wasteful. See §5 for the recommendation.

## 3. Empty indices & their sources

There are roughly **40+ empty indices** in the cluster. They break down by origin:

**Kibana Alerting framework (15 indices)** — the full `.internal.alerts-*-default-000001` family, all created on 2026-03-24. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**CDM trending datastream backers — empty current generation (5+ indices)** — `.ds-cdm_cyhy_vuln_trending-2026.05.01-000017`, `.ds-cdm_asm_nmi_trending-2026.04.16-000022`, `.ds-cdm_scuba_m365_*_trending-2026.04.28-000014` family, `.ds-cdm_scuba_gws_result_trending-2025.09.03-000001` and `2025.12.16-000002`, `.ds-cdm_scuba_gws_common_control_alert_rule_trending-2025.12.16-000001`. These are recent rollover backing indices that simply haven't received documents yet. Mostly normal.

The SCuBA GWS trending case is concerning though: `.ds-cdm_scuba_gws_result_trending-2025.09.03-000001` and `-2025.12.16-000002` are both empty. With backing indices going back to September 2025 and still empty, this suggests SCuBA Google Workspace ingestion has never produced data on this cluster — same pattern observed in other deployments.

**CDM application indices that should have data but don't (15 indices)** — this is the more concerning set:
- `cdm_vuln`, `cdm_vuln_remediated`, `cdm_vuln_trending-6-0000001` — all empty.
- `cdm_swam_trending-6-0000001`, `cdm_csm_trending-6-0000001`, `cdm_hwam_trending-6-0000001`, `cdm_device_scan_trending-6-0000001`, `cdm_scan_trending-6-0000001`, `cdm_system_boundary_trending-6-0000001` — entire trending family for SWAM, CSM, HWAM, device scan, system boundary all empty.
- `cdm_hwam`, `cdm_scuba_gws_result`, `cdm_scuba_gws_result_latest`, `cdm_scuba_gws_common_control_alert_rule`, `cdm_scuba_gws_common_control_alert_rule_latest`, `cdm_system_boundary_fisma_1163_rollup` — all empty.

Of note: `cdm_vuln` and `cdm_vuln_remediated` were created 2026-03-25 and have never received data. The trending indices were created 2026-04-21 (about two weeks before the snapshot) and have no docs. So multiple major CDM data domains (vulnerability, software/hardware/config inventory, system boundary, SCuBA GWS) appear to be silent on ingestion to this cluster, while CyHy and SCuBA M365 are working. This is a partial ingestion problem, not a total one.

**Other (1):** `.kibana_locks-000001`. No `.elastic-connectors*` or `.ent-search-*` indices — good, this cluster skipped Enterprise Search.

**Frozen tier:** The single frozen node (`instance-0000000012`) shows **0 shards, 0 bytes of disk indices**. It reports `disk.used_percent: 90.04%` — that is an ESS reporting artifact. Nothing has aged into frozen yet.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts either, consistent with the frozen node carrying no shards.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This cluster has working ingestion for some data domains and silent ingestion for others. A few things stand out:

**Two-stage provisioning visible in timestamps.** Looking at index creation dates, there are two cohorts:
- Some indices created **2026-03-24/25** (Kibana alerting, `cdm_vuln`, `cdm_vuln_remediated`, `cdm_hwam`, `cdm_cpe_dictionary`, `data_dictionary`, `cdm_scuba_m365_license`, `.cdm_config*`).
- Most CDM application indices (`cdm_cyhy_*`, `cdm_scuba_*`, `cdm_asm_nmi`, `cdm_vdp`, `cdm_threat`, `cdm_kev_dictionary`, `cdm_software_dictionary`, `cdm_cve_dictionary`, `cdm_stig_dictionary`, `cdm_directive_dictionary`, `cdm_aim`, `cdm_benchmark_vuln`) created **2026-04-21/22** or later — about a month after the initial deploy.
- Trending base indices (`cdm_*_trending-6-0000001`) created **2026-04-21**.

This suggests an initial provisioning followed by a re-deployment / template refresh ~4 weeks later. The CyHy datastreams (`cdm_cyhy_domain_trending`, `cdm_cyhy_vuln_trending`, `cdm_cyhy_host_scans_trending`, `cdm_cyhy_domain_ssl_trending`, `cdm_vdp_trending`) have history going back to 2025-09 to 2025-12 — those datastreams predate the rest of the deployment, suggesting they were retained across re-provisionings (or the dataststream itself is older than the current cluster generation).

**`cdm_benchmark_vuln_trending_YYYY.MM.DD` is creating one index per day.** There are 11 daily indices visible (2026-04-22 through 2026-05-02), each with exactly 1 document and 2 shards (pri+rep). This pattern is wasteful — over a year it creates ~365 indices, ~730 shards, for ~365 documents. Strong recommendation: convert this to a datastream, or at minimum increase the shard size by aggregating to weekly/monthly granularity. Each daily index has its own mapping, settings, and shard slot in cluster state.

**Active CyHy trending data with reasonable history.** The CyHy datastreams show meaningful document counts:
- `cdm_cyhy_domain_trending`: 79K → 67K → 66K → 55K (March → February → January → December backing indices) plus a current April backing index at 27K
- `cdm_cyhy_host_scans_trending`: 15K → 14K → 13K → 11K → 10K
- `cdm_cyhy_domain_ssl_trending`: 13K → 9.8K → 9.5K → 8.4K → 4.4K (current)
- `cdm_vdp_trending`: ~750 docs per monthly backing index

These look like healthy monthly rollover patterns with steady ingestion. Good.

**SCuBA M365 has a long backing-index tail.** `cdm_scuba_m365_result_trending` has backing indices from 2025-09 through 2026-04, all sized 100-1200 docs. This is a small steady stream — but the tail of monthly indices going back 8 months suggests ILM retention is long or absent. If September 2025 data isn't routinely queried, the ILM delete phase should be tightened.

**Master node CPU is significant.** `instance-0000000004` (the elected master) shows `load_1m=8.56, load_5m=7.51, load_15m=8.06` — sustained load of 7-9 on a 1.7 GB heap node. `instance-0000000005` (master-eligible, not elected) is similar at ~7.4. `instance-0000000003` is normal at ~1. With the cluster mostly idle from a data perspective, this elevated master load on two of three masters is unexpected. Could be cluster-state churn from the daily `cdm_benchmark_vuln_trending` index creation, frequent enrich rebuilds, or a misbehaving transform.

**Heap utilization is uneven across hot nodes.** Heap percent ranges from 7% (`instance-0000000008`) to 40% (`instance-0000000009`). This isn't problematic — it reflects which nodes hold the larger shards (`cdm_cve_dictionary`, `cdm_software_dictionary`, `cdm_cpe_dictionary`) — but worth knowing for per-node monitoring graphs.

**ILM history is healthy.** Unlike the empty deployments, this cluster has multiple `.ds-ilm-history-7-*` backing indices reaching `-000004` and `-000005`, indicating ILM is actively rolling things over. Good signal.

**No `partial-*` shards.** Frozen tier remains unused. Either ILM hasn't aged anything into frozen yet (likely — the data is too recent) or the frozen-tier policy isn't yet wired up.

**Index naming.** `cdm_vuln_trending-6-0000001` uses seven-digit zero-padding for the rollover number, matching the SSA convention.

---

## Suggested actions

1. **Investigate the silent CDM ingestion paths.** Ingestion clearly works for CyHy, SCuBA M365, and dictionaries — but `cdm_vuln`, `cdm_swam_trending`, `cdm_csm_trending`, `cdm_hwam_trending`, `cdm_device_scan_trending`, `cdm_system_boundary_trending`, `cdm_scan_trending`, the SCuBA GWS family, and `cdm_hwam` are all empty. This is a partial ingestion failure that's worth chasing in the Logstash configuration. Check whether these specific pipelines exist, are enabled, and have working credentials.
2. **Convert `cdm_benchmark_vuln_trending_*` to a datastream.** One index per day with one doc per day is the clearest cluster-state waste in this deployment. A single datastream with monthly rollover would replace 30 indices with 1 backing index.
3. **Investigate the master node load** on `instance-0000000004` and `instance-0000000005`. Sustained 7-9 load on otherwise idle masters needs a reason. Check transform health, enrich rebuild frequency, and ILM activity.
4. **Tighten ILM retention on SCuBA M365 trending** if 8 months of small monthly indices isn't operationally needed. The data volume is trivial but cluster-state cost is paid per backing index.
5. **No data-loss exposure** — every index has rep=1.
6. **No oversized or under-sharded primaries** that warrant resharding right now.
