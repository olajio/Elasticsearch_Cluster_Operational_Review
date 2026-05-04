# NRC Cluster Operational Review

**Cluster:** `b1a9029e8e984cac99106eb202f71e1f` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 3 total nodes — all combined-role hot/ingest/master/remote/search/transform (`himrst`, **7.2 GB heap**, 1.1 TB disk each). **No frozen tier. No dedicated masters. No dedicated ingest tier.** 146 primaries / 299 total shards / 0 unassigned.

This is a tightly-packed three-node deployment with active CDM data, smaller heap, and no tiering. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here is the per-primary picture:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_vuln_trending-6-000002` | 1 | 19 GB | 19.0 | ✅ within target |
| `cdm_vuln_trending-6-000003` | 1 | 17 GB | 17.0 | ✅ within target |
| `cdm_swam_trending-6-000002` | 1 | 6 GB | 6.0 | ✅ fine |
| `cdm_swam_trending-6-000003` | 1 | 4 GB | 4.0 | ✅ fine |
| `cdm_vuln_remediated` | 1 | 1 GB | 1.0 | ✅ fine |
| `cdm_cve_dictionary`, `cdm_vuln`, `cdm_cpe_dictionary` | 1 | <1 GB | <1 | ✅ fine |
| All other indices | 1 | <1 GB | <1 | ✅ small or empty |

**There are no oversized shards.** The largest primary anywhere is `cdm_vuln_trending-6-000002` at 19 GB, which sits in the 10-30 GB read-heavy band. All single-primary configs are appropriate.

**One observation:** `cdm_vuln_trending` has both generation `-6-000002` (19 GB pri) and `-6-000003` (17 GB pri) live concurrently. That's two consecutive trending generations both holding meaningful data. Same for `cdm_swam_trending` (-000002 at 6 GB pri, -000003 at 4 GB pri). This is healthy active rollover, but it does mean shard counts will keep climbing as new generations are created without older ones being aged out — see §2 and §5.

**No resharding is warranted right now.** If `cdm_vuln_trending-6-000003` continues to grow toward 30 GB primary, that would be the natural moment to consider a `max_primary_shard_size: 25gb` rollover condition.

## 2. Is the cluster oversharded?

**No.** Against the Elastic 20-shards-per-GB-heap heuristic:

- 299 total shards across 3 hot nodes = **~100 shards per hot node**.
- Heap on each hot node: 7.2 GB → ceiling of ~144 shards/node.
- Current utilization: ~69.2% of the recommended limit.

The cluster is within Elastic's recommended shard density.

For context, the shard count breakdown:

1. **Enrich indices replicated to every hot node.** 7 enrich policies (`cdm_software_dictionary_eol`, `cdm_vuln_cve`, `cdm_vuln_aware_cve`, `cdm_csm_stig`, `cdm_csm_aware_stig`, `cdm_software_directive`, `cdm_config_t0`) × 3 hot nodes = ~21 enrich shards. The `.enrich-cdm_vuln_aware_cve-1771622709930` and `.enrich-cdm_csm_aware_stig-1771622709128` "aware" variants exist but with timestamps from late February 2026 — older than other enrich indices, suggesting these specific aware enrich policies haven't been rebuilt recently.
2. **Multiple concurrent trending generations.** `cdm_vuln_trending-6-000002` and `-000003`, `cdm_swam_trending-6-000002` and `-000003`, `cdm_hwam_trending-6-000002` and `-000003`, `cdm_device_scan_trending-6-000002` and `-000003`, `cdm_system_boundary_trending-6-000002`, `-000003`, AND `-6-0000001` (mixed naming). Each generation is 2 shards (pri+rep). This is a substantial fraction of the cluster's shard budget.
3. **All other CDM and Kibana indices.**

The relevant trajectory observation: this cluster has **no frozen tier**, so nothing can age out of the hot nodes via tiering. ILM delete is the only retention mechanism. With multiple concurrent trending generations sitting in hot and active monthly rollovers adding new generations, shard count will keep rising unless ILM delete starts firing on older generations. The cluster is currently within guidance, but its trajectory should be monitored.

## 3. Empty indices & their sources

Most CDM indices in this deployment are populated. Empty ones break down as:

**Kibana Alerting framework (15 indices)** — full `.internal.alerts-*-default-000001` family, all created on **2026-02-20**. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**SCuBA M365 + GWS (mostly empty)** — `cdm_scuba_m365_result`, `cdm_scuba_m365_result_latest`, `cdm_scuba_m365_conditional_access_policy`, `cdm_scuba_m365_conditional_access_policy_latest`, `cdm_scuba_m365_license`, `cdm_scuba_m365_license_latest`, `cdm_scuba_gws_result`, `cdm_scuba_gws_result_latest`, `cdm_scuba_gws_common_control_alert_rule`, `cdm_scuba_gws_common_control_alert_rule_latest`, plus all their trending datastream backing indices (`.ds-cdm_scuba_m365_*-2026.02.20-000001`, `.ds-cdm_scuba_gws_*-2026.02.20-000001`). All empty. Both SCuBA M365 *and* GWS ingestion appear silent on this cluster.

**Other empty CDM indices** — `cdm_csm_trending-6-000002` (only the gen-2 — there is no gen-3 visible, suggesting CSM trending stopped), `cdm_scan_trending-6-0000001`, `cdm_asm_nmi`, `cdm_vdp`, `cdm_cyhy_*` family (all CyHy indices empty: domain, vuln, host_scans, domain_ssl, plus their trending), `cdm_benchmark_vuln`, `cdm_system_boundary_fisma_1163_rollup`. **CyHy ingestion is silent on this cluster.** This is unusual — CyHy is typically among the more reliably-flowing data domains.

**Mixed trending naming convention:** `cdm_system_boundary_trending` has THREE generations: `-6-000002`, `-6-000003`, AND `-6-0000001` (seven-digit padding from an older convention). The `-6-0000001` generation has 76 docs; both newer six-digit-padded generations also have 76 docs; this looks like the index template was changed at some point and the older `-6-0000001` was not cleaned up. Worth investigating.

**Other:** `.kibana_locks-000001`. **No `.elastic-connectors*` or `.ent-search-*` indices** — good, this cluster skipped Enterprise Search.

**Frozen tier: none.** No frozen node, no `partial-*` shards. All data lives on the three `himrst` nodes.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts (which is correct — there's no frozen tier here).

**No data-loss risk from missing replicas.**

## 5. Other operational insights

**No frozen tier means hot tier carries everything.** This deployment lacks a frozen tier. Disk usage on hot nodes is healthy at 23-50 GB out of 1.1 TB (~2-4%), so there's plenty of physical room. But the absence of frozen means data retention is entirely a function of ILM delete phases, and shard count growth has no relief valve via searchable-snapshot mounts. With shard count at 69% of the per-heap ceiling and active monthly rollovers adding new trending generations, this is the structural concern to watch.

**Vulnerability and SWAM are the active pipelines; CSM, CyHy, and SCuBA are silent.** Per-domain breakdown:
- ✅ **Active**: `cdm_vuln`, `cdm_vuln_remediated`, `cdm_vuln_trending` (gens 2 and 3 both populated), `cdm_swam`, `cdm_swam_trending` (gens 2 and 3), `cdm_hwam_trending` (gens 2 and 3), `cdm_device_scan_trending` (gens 2 and 3), `cdm_hwam`, `cdm_device_scan`, `cdm_host_interface`, `cdm_organization`, `cdm_system_boundary`, dictionaries (CVE, CPE, software, STIG, KEV, directive, threat).
- ❌ **Silent**: `cdm_csm`, `cdm_csm_trending` (only gen 2 exists, empty), entire SCuBA M365 family, entire SCuBA GWS family, entire CyHy family, `cdm_asm_nmi`, `cdm_vdp`, `cdm_scan_trending`.

That's a partial ingestion gap covering several major data domains. CyHy being silent here is particularly worth investigating, given that CyHy is typically among the most reliable data domains in CDM deployments.

**Errors indices have small populations.** `cdm_errors_swam` (5K docs, 167K deleted), `cdm_errors_system_boundary` (47K), `cdm_errors_vuln` (3K, with 149K deleted), `cdm_errors_hwam` (296 docs, 4K deleted), `cdm_errors_device_scan` (5.6K), `cdm_errors_organization` (661 docs, 9K deleted), `cdm_errors_host_interface` (3.7K). The high deleted-doc counts relative to live counts suggest active ILM trimming on these — healthy.

**Heap utilization is uneven and one node is RAM-pressured.** 
- `instance-0000000003`: 35% heap (2.6 GB), 74% RAM, load_5m 2.64.
- `instance-0000000005` (elected master): 21% heap (1.5 GB), 64% RAM, load_5m 0.42.
- `instance-0000000006`: 61% heap (4.4 GB), 58% RAM, load_5m 2.93.

The 74% RAM on `instance-0000000003` is the highest figure here and worth tracking. The 61% heap on `instance-0000000006` reflects which node holds the larger shards (`cdm_vuln_trending-6-000002` at 19 GB pri, `cdm_swam_trending-6-000002` at 6 GB pri, etc.).

**Sustained CPU on master/data nodes.** All three `himrst` nodes show CPU 2-5% with load_5m 0.42-2.93. The combined-role topology means master work shares the JVM with data work; with active ingestion plus master responsibilities on these 7.2 GB heap nodes, this is expected and not currently alarming. But there's less margin than a tiered topology would offer.

**Long historical context for `.cdm_config_history`.** Created **2026-02-20**, suggesting this cluster has been operating since at least mid-February 2026. The mix of gen-2 and gen-3 trending indices (created 2026-03-16 and 2026-04-15 respectively) shows monthly rollover happening on schedule.

**Mixed `cdm_system_boundary_trending` generations.** Three concurrent generations exist: `-6-0000001` (seven-digit, 2026-02-27), `-6-000002` (six-digit, 2026-03-29), `-6-000003` (six-digit, 2026-04-28). All have 76 docs. The `-6-0000001` should probably have been deleted when `-6-000002` was created; instead, it's still around with the same content. Index template inconsistency.

**SLM is producing snapshots regularly.** `.ds-.slm-history-7-*` reaches `-000011` (2026-05-01), with weekly rollover backing indices going back to `-000001` (2026-02-20). Snapshots are running.

---

## Suggested actions

1. **Investigate the silent CDM ingestion paths.** `cdm_csm`, the entire CyHy family, both SCuBA M365 and SCuBA GWS, ASM NMI, and VDP are all empty while VULN and SWAM are flowing. This is a substantial partial-ingestion gap. Start with the Logstash pipelines for these specific domains.
2. **Standardize the trending index naming convention.** `cdm_system_boundary_trending` has both `-6-0000001` (seven-digit, gen 1) and `-6-000002`/`-000003` (six-digit, gens 2 and 3) live concurrently. The seven-digit-padded gen 1 should likely be deleted once it ages out of the retention window. Confirm the index template uses a consistent pattern.
3. **Watch shard count growth.** At 69% of the per-heap ceiling on a 7.2 GB heap topology with no frozen tier, there's limited runway. Each new monthly rollover adds shards without relief. ILM delete on older trending generations is the only mechanism that prevents climbing past the ceiling — verify ILM delete phases are configured and firing.
4. **Consider adding a frozen tier** if long-term retention is a requirement. Without one, all retained data lives on the hot tier and counts against the per-heap shard ceiling.
5. **Track RAM on `instance-0000000003`** (74%). Combined with the structural shard count concern in #3, this deployment has the least margin for unexpected load spikes.
6. **No data-loss exposure** — every index has rep=1.
7. **No oversized primaries** that warrant resharding.
