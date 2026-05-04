# SEC Cluster Operational Review

**Cluster:** `2a2f99e6310c4bba906abf3d70c46ffa` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 10 total nodes — 3 master (`mr`, 3.7 GB heap), 3 hot/remote/search/transform (`hrst`, **7.2 GB heap**, 405 GB disk each), 1 frozen (`f`, 3.7 GB heap, 600 GB disk), 3 ingest (`ir`, 7.2 GB heap). 279 primaries / 627 total shards / 0 unassigned.

**This deployment has substantial CDM data on undersized hot nodes and is the most operationally pressured cluster in the batch.** The cluster is GREEN and functioning, but multiple indicators show it's running well over recommended limits. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here is the per-primary picture:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_swam_trending-6-0000001` | 1 | 40 GB | **40.0** | ⚠️ at write-heavy ceiling |
| `cdm_swam_trending-6-000003` | 1 | 40 GB | **40.0** | ⚠️ at write-heavy ceiling |
| `cdm_swam_trending-6-000002` | 1 | 40 GB | **40.0** | ⚠️ at write-heavy ceiling |
| `cdm_vuln_trending-6-0000001` | 1 | 36 GB | **36.0** | ⚠️ approaching ceiling |
| `cdm_swam` | 1 | 14 GB | 14.0 | ✅ within target |
| `cdm_swam_trending-6-000004` | 1 | 5 GB | 5.0 | ✅ active rollover, growing |
| `cdm_device_scan_trending-6-0000001` | 1 | 2 GB | 2.0 | ✅ fine |
| `cdm_vuln`, `cdm_hwam_trending-6-0000001`, `cdm_cve_dictionary` | 1 | 1 GB | 1.0 | ✅ fine |
| All other indices | 1 | <1 GB | <1 | ✅ small or empty |

**This cluster has multiple primaries at or approaching the write-heavy ceiling.** Three generations of `cdm_swam_trending` are sitting at exactly 40 GB primary each — that's not a coincidence; it strongly suggests there's a `max_primary_shard_size: 40gb` rollover condition firing at exactly that bound. The `cdm_vuln_trending-6-0000001` at 36 GB is on the same trajectory.

**Recommendations:**

- **The `cdm_swam_trending` rollover policy works correctly** — it's catching at 40 GB and rolling over to a new generation. Generation `-000004` (current write index, 5 GB) confirms this. No action needed there beyond confirming the policy.
- **However, ILM is not deleting old generations.** All four generations (`-6-0000001`, `-000002`, `-000003`, `-000004`) are live concurrently, contributing 40+40+40+5 = 125 GB of primary store *just for `cdm_swam_trending`*. Each generation also has a replica = 250 GB total cluster-wide. Verify the ILM delete phase is configured and firing.
- **`cdm_vuln_trending-6-0000001` at 36 GB and growing** will hit the same threshold soon. If that's the intended behavior, fine. If a tighter primary size is preferred (closer to the 30 GB read-heavy boundary), tighten `max_primary_shard_size` for that policy.
- **No resharding of existing data** — the existing primaries are at the upper end of acceptable but not over it.

## 2. Is the cluster oversharded?

**Yes — significantly over capacity.** This is the most pressing finding for this deployment.

- 627 total shards across 3 hot nodes = **209 shards per hot node**. Against a ceiling of ~144 shards/node (20 shards per GB of 7.2 GB heap), the cluster is at **~145.1% of the recommended limit**.
- 279 primaries.

That's not "approaching" the threshold — the cluster is operating well above the recommended shard density per heap. This is consistent with the heap and RAM pressure observed (see §5):
- Hot node RAM: 92%, 90%, 75%
- Hot node heap: 27%, 63%, 74%

The shard count breakdown:

1. **Enterprise Search shards.** This deployment has the full `.ent-search-*` family — roughly 90 distinct indices, all replicated. That's roughly 200+ shards just for Enterprise Search. **Enterprise Search is removed in 9.0.** This is also a 9.0 upgrade blocker.
2. **A growing pile of `cdm_benchmark_vuln_trending_YYYY.MM.DD` daily indices** — **22 of them** in the snapshot (April 11 through May 2). Each is 2 shards (pri+rep), each holds 1 doc. That's 44 shards for 22 documents. Over a year this pattern would create ~365 indices = 730 shards.
3. **Multiple concurrent trending generations.** Four generations of `cdm_swam_trending` (8 shards), three of `cdm_vuln_trending` family (one populated, others empty currently), other CDM trending families.
4. **Enrich indices replicated to every hot node.** 6 enrich policies × 3 hot nodes = ~18 enrich shards.
5. **Long-tail SCuBA M365 / CyHy trending datastream backing indices** going back several months.
6. **All other CDM and Kibana indices.**

**Bottom line:** running at 145% of the shard ceiling on hot nodes that are also at 90%+ RAM is an operational risk situation, not just an inefficiency.

**The recommended remediation has multiple parts:**
- **Scale up the hot tier** — doubling heap from 7.2 GB to 14.7 GB raises the ceiling from ~144 to ~294 shards/node, putting the cluster back inside guidance even before any cleanup.
- **Delete the `.ent-search-*` family** if Enterprise Search isn't actively used (estimated ~200 shards = 33% drop).
- **Convert `cdm_benchmark_vuln_trending_*` to a datastream** (estimated ~30-40 shards now, more over time).

Either #1 alone, or the combination of #2 and #3, would put the cluster back inside guidance.

## 3. Empty indices & their sources

Most CDM application indices in this deployment are populated. Empty ones break down as:

**Kibana Alerting framework (15 indices)** — full `.internal.alerts-*-default-000001` family, all created on **2026-03-09**. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless a SIEM migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**`.elastic-connectors-v1`, `.elastic-connectors-sync-jobs-v1`** — Elastic Connectors framework metadata indices. Empty if no connectors are configured.

**`.ent-search-*` (~90 indices, mostly empty)** — same as on DOL. The entire Enterprise Search backing index set, mostly empty (crawler, search relevance, oauth, app_search, workplace_search, etc.). Some have ≤2 docs (auth tokens, users, accounts). Audit logs (`.ds-logs-enterprise_search.audit-default-2026.03.09-000001` at 9 docs, `.ds-logs-enterprise_search.audit-default-2026.04.08-000002` at 0 docs, `.ds-logs-enterprise_search.api-default-2026.03.16-000002` at 0 docs) confirm Enterprise Search is *running* but barely *used*. Strong candidate for removal.

**SCuBA GWS family (empty)** — `cdm_scuba_gws_result`, `cdm_scuba_gws_result_latest`, `cdm_scuba_gws_common_control_alert_rule`, `cdm_scuba_gws_common_control_alert_rule_latest`, plus all backing indices. SCuBA GWS appears never to have produced data on this cluster.

**Empty CDM application indices** — `cdm_vuln_remediated` (created 2026-03-16, never received data), `cdm_csm_trending-6-0000001`, `cdm_scan_trending-6-0000001`, `cdm_system_boundary_trending-6-0000001`, `cdm_system_boundary_fisma_1163_rollup`, `cdm_vdp`. Also note: there's no `cdm_csm` index visible in the data at all (only the empty `cdm_csm_trending`), suggesting CSM ingestion isn't configured here.

**Other:** `.kibana_locks-000001`.

**Frozen tier:** The single frozen node (`instance-0000000009`) shows **0 shards, 0 bytes of disk indices**. It reports `disk.used_percent: 90.04%` — that is an ESS reporting artifact, not a real signal. Nothing has aged into frozen yet.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts, consistent with the frozen node carrying no shards.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

**RAM exhaustion on hot nodes is the most urgent signal.**
- `instance-0000000006` (hrst): 27% heap (1.9 GB), **92% RAM**, 33.99% disk used (137 GB / 405 GB), 208 shards.
- `instance-0000000007` (hrst): 63% heap (4.6 GB), **90% RAM**, 23.13% disk, 211 shards.
- `instance-0000000008` (hrst): 74% heap (5.4 GB), **75% RAM**, 36.48% disk, 208 shards.

92% RAM on a hot node leaves essentially no margin for memory spikes. While Elasticsearch JVMs allocate fixed heap and the rest is OS file cache (which is healthy to be high), 92% means the OS is running close to its eviction policy. Combined with 74% heap on a peer node and shard count 145% over ceiling, this is consistent with the deployment being structurally undersized for its workload.

**Heap utilization is uneven** because data isn't perfectly balanced across hot nodes (instance-7 holds the largest shards — `cdm_swam_trending-6-000002`, `cdm_vuln_trending-6-0000001` replicas, etc.).

**Master node CPU is concerning.** `instance-0000000004` (master-eligible, not the elected master) shows `load_1m=3.88, load_5m=3.76, load_15m=3.76` — sustained load of ~4 on a 3.7 GB heap node. The elected master (`instance-0000000005`) is quiet at ~0.3, and `instance-0000000013` (third master-eligible) is at ~1.0-1.7. One master inexplicably busy on an otherwise active cluster — worth diagnosing.

**Reporting anomaly: instance-0000000013.** This node appears in the node list as `mr` (master+remote_cluster_client) with IP `10.160.49.220`, but `instance-0000000007` (`hrst`) also has IP `10.160.49.220` in the same listing. Same with `instance-0000000006` (`hrst`) and `instance-0000000005` (`mr`, elected) — both at `10.160.53.192`. This may be a snapshot-timing artifact (a node restarted and got the same IP), or it may indicate the `_cat/nodes` output captured a transient state. Worth re-running `_cat/nodes` to confirm the actual topology and master count.

**Active multi-domain CDM ingestion.** `cdm_swam` and `cdm_vuln` ingestion is heavy (millions of docs in their respective trending datastreams). `cdm_hwam_trending` (1.5M docs in current generation), `cdm_device_scan_trending` (5.1M docs), CyHy domain trending (small but flowing), errors indices (1.7M for host_interface, 1.5M for system_boundary). This is one of the higher-throughput CDM deployments.

**Long historical context.** `.cdm_config_history` created **2026-03-09** (almost two months of operating history). Multiple `cdm_*` indices created **2026-03-09 to 2026-04-10** with rollover history visible. CyHy datastreams go back to **2025-12-16** (some), and SCuBA M365 backing indices go back to **2025-09-03**. The `.ds-cdm_vdp_trending-2025.06.25-000001` from June 2025 is the oldest backing index in the cluster — interestingly, it's empty.

**Two-phase deployment.** Kibana alerting framework created **2026-03-09**. CDM application indices created **2026-04-10**. About a month between Kibana setup and CDM workload deployment. Some legacy backing indices (CyHy, SCuBA M365 from 2025) predate both.

**Daily `cdm_benchmark_vuln_trending_YYYY.MM.DD` index pattern is heavier here than elsewhere.** **22 daily indices visible** — more than NASA (11), DOL (8), or ED (4). Going back to April 11. If left running for a year, this single pattern accounts for ~730 shards.

**Disk usage is meaningful but not alarming.** Hot nodes 23-36% disk used (94-148 GB on 405 GB nodes). The lower limit (instance-7 at 93 GB) and the spread suggests `disk-based shard balancing` is doing real work but the cluster has runway on disk.

**Errors indices show heavy ingestion.** `cdm_errors_host_interface` (1.7M docs), `cdm_errors_system_boundary` (1.5M docs), `cdm_errors_device_scan` (454K docs), `cdm_errors_swam` (53K), `cdm_errors_organization` (1.4K), `cdm_errors_vuln` (8.7K), `cdm_errors_hwam` (668). Worth periodic review to understand what the pipelines are rejecting.

**ILM and SLM are active.** `.ds-ilm-history-7-*` reaching `-000008`, `.ds-.slm-history-7-*` reaching `-000008` — both rolling regularly.

---

## Suggested actions (priority ordered)

1. **Scale up the hot tier — this is the headline recommendation.** Three 7.2 GB heap nodes hosting 627 shards is operating at ~145% of safe shard density. RAM at 90-92% on two of three hot nodes leaves no spike margin. Doubling heap to 14.7 GB or moving to a larger tier would put the cluster comfortably inside Elastic's recommended limits. This is operational risk, not just inefficiency.

2. **Audit and remove Enterprise Search if unused.** The `.ent-search-*` family (~200 shards, ~33% of cluster total) appears unused (most indices empty or near-empty, audit logs minimal). Enterprise Search is removed in 9.0. Confirming with stakeholders that no one depends on App Search / Workplace Search would let the entire `.ent-search-*` family be deleted, immediately cutting shard count by ~33% and unblocking the eventual 9.0 upgrade.

3. **Verify ILM delete phase is configured and firing on `cdm_swam_trending`.** Four concurrent generations of 40 GB primary shards (250 GB cluster-wide just for SWAM trending) suggests retention is unbounded or the delete phase hasn't fired. Tightening retention would free meaningful disk and shard count.

4. **Convert `cdm_benchmark_vuln_trending_*` to a datastream.** 22 daily indices for 22 docs is the worst case of this pattern seen in the batch. A datastream with monthly rollover would replace ~30 daily indices per month with 1 backing index.

5. **Re-verify the actual master topology.** The duplicate IPs in the `_cat/nodes` output (`instance-0000000013` and `instance-0000000007` both at `10.160.49.220`; `instance-0000000006` and `instance-0000000005` both at `10.160.53.192`) suggest a reporting anomaly or recent topology change. Worth re-running `_cat/nodes` and `_cluster/state/nodes` to confirm the cluster has the expected 3 master-eligible nodes.

6. **Investigate the master node load** on `instance-0000000004` (load_5m ~3.76 on a 3.7 GB heap master node). Consistent with the recurring pattern of one master being mysteriously busy.

7. **Investigate the silent CDM domains.** `cdm_vuln_remediated` (empty since 2026-03-16), `cdm_csm_trending`, `cdm_scan_trending`, `cdm_system_boundary_trending`, the entire SCuBA GWS family — all empty while VULN and SWAM trending are heavily populated. Worth diagnosing per pipeline.

8. **No data-loss exposure** — every index has rep=1.

9. **Consider tightening rollover policy on `cdm_swam_trending`.** 40 GB primary is at the write-heavy ceiling; 30 GB would be safer and would also force more frequent rollovers, which combined with ILM delete in #3 keeps cluster state cleaner.
