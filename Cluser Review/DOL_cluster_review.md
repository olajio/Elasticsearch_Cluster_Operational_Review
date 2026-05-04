# DOL Cluster Operational Review

**Cluster:** `f700c2a9c08a4501a7dcb9a9e021e424` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 4 total nodes — 3 combined-role hot/ingest/master/remote/search/transform (`himrst`, **14.7 GB heap**, 810 GB disk each) + 1 frozen (`f`, 3.7 GB heap, 600 GB disk). 285 primaries / 639 total shards / 0 unassigned.

This is an active CDM deployment with real data and a high shard count for its size. Note that all three data nodes carry combined master+data role — there are no dedicated master nodes here. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here's the per-primary picture for the indices that carry data:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_vuln_trending-6-0000001` | 1 | 5 GB | 5.0 | ✅ comfortably under target |
| `cdm_swam_trending-6-0000001` | 1 | 3 GB | 3.0 | ✅ fine |
| `cdm_cve_dictionary` | 1 | 1 GB | 1.0 | ✅ small dictionary |
| `cdm_swam`, `cdm_vuln`, `cdm_vuln_remediated` | 1 | <1 GB | <1 | ✅ fine |
| `cdm_software_dictionary`, `cdm_cpe_dictionary` | 1 | <1 GB (≈250 MB) | <1 | ✅ fine |
| Active CyHy/SCuBA trending datastreams | 1 | 0 GB | 0 | small daily volumes |
| All other CDM indices | 1 | 0 GB | 0 | small or empty |

**There are no oversized shards.** The largest primary anywhere is `cdm_vuln_trending-6-0000001` at 5 GB, well under the 40 GB target. All single-primary configs are appropriate for current data volumes.

**No resharding is warranted.** If `cdm_vuln_trending` and `cdm_swam_trending` continue to grow at observed rates (collectively ~10M docs in the current generation), revisiting primary count for those specific datastreams might be worth doing once they cross the 20 GB primary mark.

## 2. Is the cluster oversharded?

**No.** Against the Elastic 20-shards-per-GB-heap heuristic:

- 639 total shards across 3 hot nodes = **213 shards per hot node**.
- Heap on each hot node: 14.7 GB → ceiling of ~294 shards/node.
- Current utilization: ~72.4% of the recommended limit.

The cluster is within Elastic's recommended shard density, though it is the highest utilization of any tier in the deployment.

For context, the shard count is dominated by:

1. **Enterprise Search shards.** This deployment has the full `.ent-search-*` family — roughly 90 distinct indices including both data indices and unique-constraint indices, all replicated. That's roughly 200+ shards just for Enterprise Search. **Enterprise Search is removed in 9.0**, so this is also an upgrade blocker, separate from the shard-density question.
2. **Enrich indices replicated to every hot node.** 6 enrich policies (`cdm_software_dictionary_eol`, `cdm_vuln_cve`, `cdm_vuln_aware_cve`, `cdm_csm_stig`, `cdm_csm_aware_stig`, `cdm_software_directive`, `cdm_cdo_temp`, `cdm_config_t0`) × 3 hot nodes = ~24 enrich shards. Notable: `.enrich-cdm_vuln_aware_cve` and `.enrich-cdm_csm_aware_stig` are newer "aware" variants, suggesting a recent enrich pipeline evolution.
3. **Daily `cdm_benchmark_vuln_trending_YYYY.MM.DD` indices** — 8 of them visible in the snapshot, 1 doc each, 2 shards (pri+rep). Wasteful pattern as a separate operational concern. Over a year, ~730 shards just for these.
4. **CDM application + trending datastreams + Kibana + ILM/SLM history** — the rest.

The trajectory matters here: with active ingestion creating new monthly trending generations and daily benchmark indices accumulating, shard count will keep climbing. ILM delete on older generations and the `.ent-search-*` cleanup proposed in §3 are the levers that keep this deployment inside guidance over time.

## 3. Empty indices & their sources

Most indices in this deployment are populated to some degree. Empty ones break down as:

**Kibana Alerting framework (15 indices)** — full `.internal.alerts-*-default-000001` family, all created on **2026-02-02** (which predates many of the CDM indices — see §5). Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (3 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`, `.kibana-siem-rule-migrations-rules-health`, `.kibana-siem-rule-migrations-migrations-health`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**`.elastic-connectors-v1`, `.elastic-connectors-sync-jobs-v1`** — Elastic Connectors framework metadata indices. Empty if no connectors are configured.

**`.ent-search-*` (~90 indices, mostly empty)** — the entire Enterprise Search backing index set. Some have ≤2 docs (auth tokens, users, accounts, organizations); many are completely empty (crawler, search relevance, oauth_access_tokens, app_search engines, workplace_search content sources, etc.). **None of this looks actively used.** Since Enterprise Search is removed in 9.0, the entire `.ent-search-*` family can be deleted if no team actually uses Enterprise Search/App Search/Workplace Search in this deployment. That alone would drop ~200 shards from the cluster.

**SCuBA GWS trending (empty)** — `cdm_scuba_gws_result`, `cdm_scuba_gws_result_latest`, `cdm_scuba_gws_common_control_alert_rule`, `cdm_scuba_gws_common_control_alert_rule_latest`, plus `.ds-cdm_scuba_gws_result_trending-2025.09.03-000001`, `.ds-cdm_scuba_gws_result_trending-2025.12.16-000002`, `.ds-cdm_scuba_gws_common_control_alert_rule_trending-2025.12.16-000001` are all empty. Backing indices going back to September 2025 with no docs strongly suggests SCuBA Google Workspace ingestion has never produced data on this cluster.

**Other empty CDM indices** — `cdm_scan_trending-6-0000001`, `cdm_csm_trending-6-0000001`, `cdm_system_boundary_fisma_1163_rollup`, `cdm_system_boundary` (1 doc with 655 deleted, effectively empty). Notable: `cdm_csm_trending` is empty even though `cdm_swam_trending` and `cdm_vuln_trending` are populated. A partial CDM ingestion gap.

**Frozen tier:** The single frozen node (`instance-0000000008`) shows **0 shards, 0 bytes of disk indices**. It reports `disk.used_percent: 90.05%` — that is an ESS reporting artifact, not real data. Nothing has aged into frozen yet.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts, consistent with the frozen node carrying no shards.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This deployment has been running long enough to show clear ingestion patterns and historical depth. A few things worth flagging:

**Two-phase deployment history visible in timestamps.** The Kibana alerting framework (`.internal.alerts-*-default-000001`) was created **2026-02-02**, while the CDM application indices were created **2026-04-22** through **2026-05-03**. That's ~10 weeks between Kibana setup and CDM provisioning. Either this cluster sat idle for two months before CDM workloads were added, or there was a CDM template re-deployment that recreated the indices with newer creation dates.

**CyHy datastreams have a long, healthy history.** `cdm_cyhy_domain_trending` has backing indices from `-000003` (2025-12-14) through `-000029` (2026-04-25) — 26 monthly rollovers visible, with consistent ~5-6K docs per backing index. Same pattern for `cdm_cyhy_vuln_trending`, `cdm_cyhy_host_scans_trending`, `cdm_cyhy_domain_ssl_trending`. CyHy ingestion is solid here.

**Fleet/Elastic Agent traffic visible.** `.ds-metrics-fleet_server.agent_versions-default-*` and `.ds-metrics-fleet_server.agent_status-default-*` going back to 2026-02-02 with steady ~43K docs per backing index. Fleet is actively managing agents on this cluster. Matches the `.fleet-agents-7`, `.fleet-policies-7`, `.fleet-enrollment-api-keys-7` system indices being populated.

**`cdm_benchmark_vuln_trending_YYYY.MM.DD` is creating one index per day.** 8 daily indices visible (2026-04-25 through 2026-05-02), each with exactly 1 document and 2 shards. Over a year this creates ~365 indices, ~730 shards, for ~365 documents — wasteful for what amounts to a low-volume time series.

**Heap utilization is variable across nodes.** `instance-0000000004`: 31% heap, 83% RAM. `instance-0000000006`: 25% heap, 68% RAM. `instance-0000000007`: 17% heap, 65% RAM. The 83% RAM on `instance-0000000004` is worth tracking — it's not at the level seen on small-heap deployments, but it's the highest of the three. Disk usage on all three is ~6-13 GB out of 810 GB — plenty of headroom.

**No master CPU concerns.** All three combined-role nodes show `load_5m` between 0.22 and 0.35 — quiet. This is unusual for a `himrst` deployment and is a good sign.

**Index naming.** All trending indices use the `-6-0000001` (seven-digit, generation 1) convention consistently. No mixed-generation naming on this cluster.

**Errors indices are populated.** `cdm_errors_host_interface` (1.4M docs), `cdm_errors_system_boundary` (725K docs), `cdm_errors_device_scan` (50K docs), `cdm_errors_hwam` (118 docs), `cdm_errors_swam` (65 docs), `cdm_errors_organization` (3 docs). The volume gradient from `host_interface` → `organization` is informative: the largest error categories are coming from inventory ingestion edge cases. Worth periodically reviewing what's in these to understand what's being rejected by the pipelines.

**Enterprise Search audit logs are flowing but minimal.** `.ds-logs-enterprise_search.audit-default-2026.02.02-000001` (24 docs), `.ds-logs-enterprise_search.audit-default-2026.03.04-000002` (0 docs), and `.ds-logs-enterprise_search.api-default-2026.03.05-000004` (0 docs). Confirms Enterprise Search is *running* but not really being *used* — supporting the case for removing it.

---

## Suggested actions

1. **Audit Enterprise Search usage.** This is the highest-value cleanup — the `.ent-search-*` family appears unused (most indices empty or near-empty, audit logs minimal), and Enterprise Search is removed in 9.0. Confirming with stakeholders that no one depends on App Search / Workplace Search would let the entire `.ent-search-*` family be deleted, dropping ~200 shards from the cluster (about 30%) and unblocking the eventual 9.0 upgrade.
2. **Convert `cdm_benchmark_vuln_trending_*` to a datastream.** One index per day for one doc per day is wasteful. A single datastream with monthly rollover would replace 30+ daily indices with 1 backing index.
3. **Investigate the empty `cdm_csm_trending-6-0000001`.** SWAM and VULN trending are populated; CSM trending is not. Check whether CSM ingestion is hitting a different codepath or simply isn't enabled.
4. **Investigate the SCuBA GWS ingestion gap.** Backing indices from September 2025 are still empty — SCuBA Google Workspace ingestion does not appear to have ever produced data on this cluster.
5. **No data-loss exposure** — every index has rep=1.
6. **No oversized primaries** that warrant resharding.
7. **Watch shard count growth.** At 72% of the per-node ceiling on a `himrst` topology with no dedicated masters, there's less margin than the percentage suggests. The Enterprise Search cleanup in #1 would buy meaningful headroom.
