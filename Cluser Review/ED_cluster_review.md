# ED Cluster Operational Review

**Cluster:** `81ff697e0d2d418e903e8bca00ad065a` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 10 total nodes — 3 master (`mr`, 3.7 GB heap), 3 hot/remote/search/transform (`hrst`, **29.6 GB heap**, ~1.5 TB disk each), 1 frozen (`f`, 7.2 GB heap, 1 TB disk), 3 ingest (`ir`, 3.7 GB heap). 183 primaries / 374 total shards / 0 unassigned.

This deployment has active CDM data with several mid-sized shards. Walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here is the per-primary picture:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_csm_trending-6-0000001` | 1 | 15 GB | 15.0 | ✅ within target |
| `cdm_vuln_remediated` | 1 | 12 GB | 12.0 | ✅ within target |
| `cdm_vuln_trending-6-0000001` | 1 | 11 GB | 11.0 | ✅ within target |
| `cdm_swam_trending-6-0000001` | 1 | 7 GB | 7.0 | ✅ fine |
| `cdm_csm` | 1 | 3 GB | 3.0 | ✅ fine |
| `cdm_vuln`, `cdm_swam` | 1 | 2 GB | 2.0 | ✅ fine |
| `cdm_cve_dictionary` | 1 | 1 GB | 1.0 | ✅ small dictionary |
| All other indices | 1 | <1 GB | <1 | ✅ small or empty |

**There are no oversized shards.** The largest primary anywhere is `cdm_csm_trending-6-0000001` at 15 GB — comfortably within the 10-30 GB read-heavy band and well under the write-heavy ceiling. All single-primary configs are appropriate.

**No resharding is warranted.** The `-6-0000001` generation is collecting data well; if these grow toward 30 GB primary by next rollover, that would be the natural moment to reconsider primary count via the rollover policy (e.g., `max_primary_shard_size: 25gb`).

## 2. Is the cluster oversharded?

**No, comfortably under the limit.**

- 374 total shards across 3 hot nodes = **~125 shards per hot node**. Against a ceiling of ~592 shards/node (20 shards per GB of 29.6 GB heap), the cluster is at ~21.1% of the safe limit.
- 183 primaries.

The shard count breakdown:

1. **Enrich indices replicated to every hot node.** 6 enrich policies (`cdm_software_dictionary_eol`, `cdm_vuln_cve`, `cdm_vuln_aware_cve`, `cdm_csm_stig`, `cdm_csm_aware_stig`, `cdm_software_directive`, `cdm_cdo_temp`, `cdm_config_t0`) × 3 hot nodes = ~24 enrich shards. The "aware" enrich variants (`cdm_vuln_aware_cve`, `cdm_csm_aware_stig`) suggest a recent enrich pipeline evolution.
2. **A growing herd of `cdm_benchmark_vuln_trending_YYYY.MM.DD` daily indices** — 4 of them visible (April 29 through May 2), each at 1 doc with 2 shards.
3. **A long tail of monthly-rollover trending datastream backing indices** going back to 2025 — see §5.
4. **Active CDM application indices** carrying real data.

Plenty of headroom for growth. The recommendation here is operational hygiene rather than capacity reduction.

## 3. Empty indices & their sources

Most indices in this deployment are populated. Empty ones break down as:

**Kibana Alerting framework (15 indices)** — full `.internal.alerts-*-default-000001` family, all created on **2026-03-10**. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**SCuBA GWS family (empty)** — `cdm_scuba_gws_result`, `cdm_scuba_gws_result_latest`, `cdm_scuba_gws_common_control_alert_rule`, `cdm_scuba_gws_common_control_alert_rule_latest`, plus backing indices `.ds-cdm_scuba_gws_result_trending-2025.09.03-000001`, `.ds-cdm_scuba_gws_result_trending-2025.12.16-000002`, `.ds-cdm_scuba_gws_common_control_alert_rule_trending-2025.12.16-000001`. SCuBA Google Workspace ingestion appears never to have produced data on this cluster.

**Empty current-generation trending backing indices (4)** — `.ds-cdm_asm_nmi_trending-2026.04.16-000017`, `.ds-cdm_asm_nmi_trending-2026.02.14-000014`, `.ds-cdm_vdp_trending-2025.06.25-000001`, `.ds-cdm_cyhy_vuln_trending-2026.01.15-000016` (no — actually populated). The ASM NMI gap is notable: backing indices from February 2026 onward are empty. The June 2025 VDP backing index is suspicious because it's so old and isolated; might have been left behind from a prior naming convention.

**Empty CDM application indices** — `cdm_scan_trending-6-0000001`, `cdm_system_boundary_trending-6-0000001`, `cdm_system_boundary_fisma_1163_rollup`, `cdm_vdp`. Notable: vulnerability and CSM trending are populated heavily, but `cdm_scan_trending` and `cdm_system_boundary_trending` are not. Partial CDM ingestion.

**`tmp_cdm_vuln` (cleanup candidate)** — created 2026-03-30 with 11 documents. The `tmp_` prefix suggests this was a test or migration artifact that should have been deleted. Worth investigating before removing, but a clear cleanup candidate.

**Other:** `.kibana_locks-000001`. **No `.elastic-connectors*` or `.ent-search-*` indices** — good, this cluster skipped Enterprise Search and is in good shape for the eventual 9.0 upgrade.

**Frozen tier:** The single frozen node (`instance-0000000009`) shows **0 shards, 0 bytes of disk indices**. It reports `disk.used_percent: 91.13%` — that is an ESS reporting artifact. Nothing has aged into frozen yet.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

**Active multi-domain CDM ingestion.** Real data flowing into vulnerability, CSM, SWAM, HWAM, device scan, software/CPE/CVE/STIG dictionaries, CyHy domain/host scans/vuln/SSL trending, SCuBA M365, and errors indices. Several million documents in the larger trending indices. This looks like one of the healthier CDM deployments.

**Master node CPU is concerning.** `instance-0000000003` (master-eligible, not the elected master) shows `load_1m=7.23, load_5m=6.82, load_15m=6.61` — sustained load of ~7 on a 3.7 GB heap node. The elected master (`instance-0000000004`) is at ~1.5, and the third master (`instance-0000000005`) is quiet at ~0.6. One master inexplicably busy on an otherwise healthy cluster — recurring pattern. Worth checking transform health, enrich rebuild frequency, and any CCS/CCR activity targeting this cluster.

**Long monthly-rollover history on multiple datastreams.** `cdm_cyhy_domain_trending` has backing indices from `-000018` (2025-12-15) through `-000027` (2026-04-14). `cdm_scuba_m365_result_trending` going back to `-000011` (2025-09-03). `cdm_cyhy_host_scans_trending`, `cdm_cyhy_domain_ssl_trending`, `cdm_scuba_m365_license_trending`, `cdm_scuba_m365_conditional_access_policy_trending` all have similar 5-7 month tails. ILM retention is generous; consider tightening if the older months aren't routinely queried.

**`cdm_benchmark_vuln_trending_YYYY.MM.DD` daily index pattern.** 4 daily indices visible. Same wasteful pattern — recommend converting to a datastream.

**Two-phase deployment history.** Many indices created **2026-03-10** (Kibana, `cdm_hwam`, `.cdm_config*`, `cdm_scuba_m365_license`); CDM application indices and trending base indices created **2026-03-27** through **2026-04-28**. The CyHy and SCuBA datastreams have history predating both, going back to 2025-09 (and one to June 2025), suggesting datastreams persisted across re-provisionings.

**Heap utilization is uneven.** `instance-0000000007` (hot): 59% heap, 72% RAM. `instance-0000000006`: 40% / 71%. `instance-0000000008`: 33% / 71%. Disk usage 1.65-3.58% — minimal. The 59% heap on instance-0000000007 reflects which node holds the largest shards (`cdm_csm_trending`, `cdm_vuln_trending`, `cdm_vuln_remediated`).

**Ingest tier is doing real work.** Three `ir` (ingest+remote_cluster_client) nodes show heap 23-64% — `instance-0000000012` at 64% suggests it's the busiest of the three. RAM 59-61% across the three. Healthy ingest tier handling actual pipeline traffic.

**ILM activity is healthy.** `.ds-ilm-history-7-*` backing indices reaching `-000008`, indicating regular rollover activity. `.ds-.slm-history-7-*` reaching `-000008` indicates snapshots are running.

**Index naming is consistent.** All trending indices use `-6-0000001` (seven-digit, generation 1). No mixed-generation issues here.

**Errors indices are populated.** `cdm_errors_host_interface` (228K docs), `cdm_errors_system_boundary` (196K docs), `cdm_errors_device_scan` (83K), `cdm_errors_swam` (22K), `cdm_errors_vuln` (25K), `cdm_errors_hwam` (20K), `cdm_errors_csm` (15K), `cdm_errors_organization` (20K). Worth periodically reviewing top error patterns to understand what the pipelines are rejecting.

---

## Suggested actions

1. **Investigate `tmp_cdm_vuln`.** Created 2026-03-30 with 11 documents, suspicious `tmp_` prefix. Confirm it's an abandoned test artifact and delete it.
2. **Investigate the master node load** on `instance-0000000003`. Sustained 6-7 load average on a master-eligible node is unexpected and recurring.
3. **Investigate the SCuBA GWS ingestion gap.** Backing indices from September 2025 are still empty.
4. **Investigate the empty `cdm_scan_trending-6-0000001` and `cdm_system_boundary_trending-6-0000001`.** Other trending indices are working; these two specifically are empty.
5. **Investigate the empty ASM NMI backing indices.** Multiple recent (2026-02 onward) backing indices are empty, suggesting ASM NMI ingestion may have stopped.
6. **Convert `cdm_benchmark_vuln_trending_*` to a datastream.** One index per day for one document per day is wasteful — a datastream with monthly rollover would replace ~30 daily indices per month with 1 backing index.
7. **Tighten ILM retention on the long-tail trending datastreams** if the older months aren't routinely queried (SCuBA M365 going back 8 months, CyHy going back 5 months).
8. **No data-loss exposure** — every index has rep=1.
9. **No oversized primaries** that warrant resharding.
10. **Cluster is well-sized** — ample shard count headroom (21% of ceiling) and disk usage below 4%.
