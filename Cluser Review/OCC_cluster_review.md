# OCC Cluster Operational Review

**Cluster:** `c9cd36f5cf224badafbe9344ecb2b974` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 10 total nodes — 3 master (`mr`, 1.7 GB heap), 3 hot/remote/search/transform (`hrst`, **14.7 GB heap**, 810 GB disk each), 1 frozen (`f`, 7.2 GB heap, 1 TB disk), 3 ingest (`ir`, 3.7 GB heap). 114 primaries / 233 total shards / 0 unassigned.

This deployment uses smaller hot nodes (14.7 GB heap), only 3 of them, and a dedicated ingest tier. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

**There are no oversized shards in this cluster — there is essentially no data.**

Every single index has either zero documents or a primary store that rounds to 0 GB. The only index with meaningful document count is `cdm_cpe_dictionary` (1.49M docs, ~250 MB on disk). Everything else is empty.

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_cpe_dictionary` | 1 | <1 GB (≈250 MB) | <1 | OK |
| Every other `cdm_*` and `.ds-cdm_*` | 1 | 0 GB | 0 | empty |
| `.kibana_*`, `.security*`, `.tasks`, `.transform-*` | 1 | 0 GB | 0 | system-tiny |

**No resharding is warranted on the data side**, because there is no data to reshard. Every CDM index is at `pri=1, rep=1`, which is the right default for 0 GB content. Note that the smaller hot nodes (14.7 GB heap) imply a slightly tighter target shard size when data does arrive — the 40 GB/shard recommendation still holds, but the lower heap means less headroom for shard count, so primary count discipline matters more here.

## 2. Is the cluster oversharded?

**No.** Against the Elastic 20-shards-per-GB-heap heuristic:

- 233 total shards across 3 hot nodes = **~78 shards per hot node**.
- Heap on each hot node: 14.7 GB → ceiling of ~294 shards/node.
- Current utilization: ~26.4% of the recommended limit.

The cluster is comfortably within Elastic's recommended shard density.

For context, the shard count is dominated by:

1. **Enrich indices** — 5 enrich policies (`cdm_config_t0`, `cdm_software_directive`, `cdm_csm_stig`, `cdm_software_dictionary_eol`, `cdm_vuln_cve`) × 3 hot nodes = ~15 enrich shards.
2. **Empty CDM and Kibana alerting indices** that are pre-provisioned but never written to.

If data starts flowing in and ingestion creates additional rolled-over backing indices each month, shard count will climb. With 14.7 GB heap on three hot nodes, the per-node ceiling is ~294 shards — there is room for growth before any shard-density concern arises.

## 3. Empty indices & their sources

This deployment is almost entirely empty — only `cdm_cpe_dictionary` (1.49M docs), `data_dictionary` (3,858 docs), and a handful of small Kibana/SLM/system indices have any content. Every CDM application index, every CDM trending datastream, and every Kibana alerting index is at 0 docs.

Categorized:

**Kibana Alerting framework (15 indices)** — the full `.internal.alerts-*-default-000001` family covering security, ML anomaly detection (×2), observability (metrics/logs/uptime/threshold/slo/apm), streams, stack, transform health, dataset quality, attack discovery, default. All created on **2026-03-10**. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**CDM trending datastream backers (10 indices)** — all `.ds-cdm_*_trending-2026.03.12-000001` (CyHy domain/vuln/host_scans/domain_ssl, scuba M365 result/license/conditional_access_policy, scuba GWS result/common_control, asm_nmi, vdp). Plus `.ds-cdm_asm_nmi_trending-2026.03.17-000002` (a single rollover that happened on 2026-03-17 with no docs). All empty. None has rolled over since March 12.

**CDM application indices (24+ indices)** — `cdm_vuln`, `cdm_vuln_remediated`, `cdm_vuln_trending-6-000002`, `cdm_csm_trending-6-000002`, `cdm_swam_trending-6-000002`, `cdm_hwam_trending-6-000002`, `cdm_device_scan_trending-6-000002`, `cdm_scan_trending-6-0000001`, `cdm_system_boundary_trending-6-0000001`, `cdm_cyhy_*`, `cdm_scuba_m365_*`, `cdm_scuba_gws_*`, `cdm_asm_nmi`, `cdm_vdp`, `cdm_threat`, `cdm_hwam`, `cdm_software_dictionary`, `cdm_kev_dictionary`, `cdm_cve_dictionary`, `cdm_directive_dictionary`, `cdm_benchmark_vuln`, `cdm_system_boundary_fisma_1163_rollup`. **All zero docs.** Even the dictionary indices that should be populated by their respective seed/sync jobs (CVE, software, KEV, directive) are all empty.

**Naming inconsistency worth flagging:** Some trending indices use the suffix `-6-000002` (six-digit padding, generation 2) — `cdm_vuln_trending`, `cdm_csm_trending`, `cdm_swam_trending`, `cdm_hwam_trending`, `cdm_device_scan_trending` — while others use `-6-0000001` (seven-digit padding, generation 1) — `cdm_scan_trending`, `cdm_system_boundary_trending`. The fact that some are at generation 2 implies a previous generation existed and was rolled over, but those base indices were created on 2026-03-17 and are still empty. This is unusual and worth investigating: why did they roll over once but never receive data?

**Other:** `.kibana_locks-000001`. No `.elastic-connectors*` or `.ent-search-*` indices — good, this cluster skipped Enterprise Search.

**Frozen tier:** The single frozen node (`instance-0000000009`) shows **0 shards, 0 bytes of disk indices**. It reports `disk.used_percent: 91.13%` — that is an ESS reporting artifact (overcommit math), not a real signal. Nothing has aged into frozen yet.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts either, consistent with the frozen node carrying no shards.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This cluster shows the same "provisioned but never ingested" pattern, but with one architectural twist: there's a dedicated ingest tier (3 `ir` nodes), implying this deployment was *expected* to receive significant pipeline traffic that never materialized.

**Multiple provisioning waves visible.** Looking at index creation timestamps, there are at least three cohorts:
- **2026-03-10**: Kibana alerting framework (`.internal.alerts-*` family).
- **2026-03-12**: Most CDM application indices and trending datastreams (`cdm_cyhy_*`, `cdm_scuba_*`, `cdm_vuln`, etc.).
- **2026-03-17**: A second wave including the trending base indices (`cdm_*_trending-6-0000001`/`-6-000002`), dictionary indices (`cdm_cve_dictionary`, `cdm_kev_dictionary`, `cdm_software_dictionary`, `cdm_threat`, `cdm_directive_dictionary`), and `data_dictionary`.

The two trending naming generations (`-6-000002` vs `-6-0000001`) suggest an iteration cycle where the original templates were rolled forward at least once, possibly because of a template change. Worth confirming this is intentional.

**Dedicated ingest tier with no work.** Three `ir` (ingest+remote_cluster_client) nodes are present. Ingest nodes exist to run ingest pipelines for indexed documents and to act as remote_cluster_client coordinators for cross-cluster search/replication. With zero documents being indexed, the ingest aspect is idle. The `remote_cluster_client` aspect could still be active if this cluster is consuming data from another cluster via CCS or CCR — that would be worth confirming. The ingest nodes show modest load (CPU 0-1%, load_1m 0.09-0.81), so they're not idle-idle but they're not under stress either.

**Master node CPU is very high on one master.** `instance-0000000005` (master-eligible, not elected) shows `load_1m=6.56, load_5m=7.10, load_15m=6.90` — sustained load of ~7 on a 1.7 GB heap node. The elected master (`instance-0000000004`) is quiet at ~0.6, and the third master (`instance-0000000003`) is at ~1.7. One master inexplicably busy on an otherwise idle cluster is worth diagnosing: stuck transforms, enrich rebuild loops, cluster-state churn from empty index housekeeping, or pending tasks from CCR.

**Heap utilization is bimodal.** Hot node heap ranges from 8% (`instance-0000000006`) to 54% (`instance-0000000008`). With nearly identical shard counts (77-78 per node) and effectively no data, this variance reflects which node holds the few populated shards (`cdm_cpe_dictionary`, `data_dictionary`, `.tasks`). On an active cluster this should be more uniform; here it's just noise.

**RAM utilization is uniformly elevated.** All hot nodes show ~55% RAM. Master nodes are 63-70%. Nothing alarming, but the floor is high enough that significant additional load would push warning territory quickly.

**Frozen tier is unused.** A single frozen node sized at 1 TB sitting idle. If frozen retention is part of the plan for when ingestion eventually starts, leave it. If not, this is a candidate for downsizing.

**Index naming generations.** The `-6-000002` pattern (six digits, gen 2) for some trending indices and `-6-0000001` (seven digits, gen 1) for others is genuinely inconsistent. One convention should be picked via the index template; mixed padding can confuse glob-based tooling.

---

## Suggested actions

1. **Verify expected state.** Is this cluster supposed to be ingesting? If yes, this is the headline finding — every CDM ingestion path is silent. Six weeks since the most recent provisioning wave (2026-03-17) with no documents arriving suggests broken pipelines.
2. **If this cluster should be receiving data:** check the upstream Logstash / pipeline configuration pointing at this cluster. Given the past CDM pipeline silent-failure pattern (`Connection reset by peer`), this is the same failure mode at scale. Three dedicated ingest nodes were provisioned to handle this traffic, suggesting the volume was expected to be substantial.
3. **Investigate why trending indices have a generation 2 (`-6-000002`).** Something rolled them over. Was it intentional template change? An accidental rollover via the API? If unintended, the answer affects how the empty state should be interpreted.
4. **Investigate the master node load** on `instance-0000000005`. Sustained 7 load with no real workload is unexpected.
5. **Investigate the dictionary indices.** `cdm_cve_dictionary`, `cdm_kev_dictionary`, `cdm_software_dictionary`, `cdm_directive_dictionary`, `cdm_threat` should all be populated by their respective seed/sync jobs. All five are empty. These are usually pulled by a separate scheduled job, not by Logstash — worth checking whether those refresh jobs even target this cluster.
6. **Standardize the trending index naming pattern.** Either `-6-0000001` (seven digits) or `-6-000001` (six digits) should be used consistently across all CDM trending indices. Mixed conventions invite tooling bugs.
7. **Consider right-sizing the frozen and ingest tiers** if the deployment is going to remain idle. With no data flowing in, three ingest nodes and a 1 TB frozen node are unused capacity.
