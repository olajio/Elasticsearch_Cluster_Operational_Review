I'll perform a thorough operational review of the SSA cluster based on the data provided. Let me analyze the file systematically.The file content is in my context already. Let me analyze it directly.Let me work through the analysis based on the data in context. I'll compute the key metrics.Now I have all the analysis. Let me write the comprehensive operational review.

---

# SSA Cluster Operational Review

**Cluster:** `dac93f903e0340738c3aceb72c46bd51` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 43 total nodes — 3 master (`mr`), 36 hot/content/remote/transform (`hrst`, 29.6 GB heap, ~1.5 TB disk each), 1 frozen (`f`), 3 ingest (`ir`). 322 primaries / 1029 total shards / 0 unassigned.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here's what the per-primary math looks like for the indices that actually carry data:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_vuln_trending-6-0000001` | 12 | 145 GB | **12.1** | ✅ in range (read-heavy trending) |
| `cdm_swam_trending-6-0000001` | 6 | 136 GB | **22.7** | ✅ in range |
| `cdm_swam` | 6 | 45 GB | **7.5** | slightly small but acceptable |
| `cdm_vuln` | 12 | 34 GB | **2.8** | ⚠️ over-sharded |
| `cdm_csm_trending-6-0000001` | 6 | 31 GB | **5.2** | ⚠️ small |
| `cdm_csm` | 6 | 5 GB | **0.8** | ⚠️ over-sharded |
| `cdm_vuln_remediated` | 18 | 4 GB | **0.22** | 🔴 severely over-sharded |
| `cdm_device_scan_trending-6-0000001` | 1 | 1 GB | 1.0 | OK (single shard, harmless) |

**Good news first:** none of your major indices have *oversized* shards. The biggest data-bearing indices (`cdm_vuln_trending` at 12.1 GB/shard, `cdm_swam_trending` at 22.7 GB/shard) are sized correctly for read-heavy trending workloads. You don't need to *reduce* any shard counts on the big indices.

**The actual sizing problem is the opposite — over-sharding:**

- `cdm_vuln_remediated` with **18 primaries for 4 GB** is the worst offender. This should be **1 primary**, full stop. Each shard averages ~225 MB.
- `cdm_vuln` at **12 primaries for 34 GB** could comfortably be **1 primary** (or 2 if you anticipate growth toward the 80+ GB range). At 2.8 GB/shard you're paying coordination overhead for nothing.
- `cdm_csm` at **6 primaries for 5 GB** should be **1 primary**.

Resharding these requires either reindex into a new index with the corrected `number_of_shards`, or (cheaper for the trending family) wait for the next ILM rollover and update the index template so the *next* generation has the right primary count. Since these are non-data-stream regular indices (no `.ds-` prefix, no rollover number), reindex is the path. Use the Reindex API with `slices=auto` and a new target index, then alias-swap.

A note on `cdm_swam` (45 GB on 6 primaries = 7.5 GB/shard): it's borderline small but I'd leave it alone — it's clearly write-active (39M deleted docs vs 32M live, suggesting heavy update/upsert traffic), and consolidating to fewer primaries would concentrate write load.

## 2. Is the cluster oversharded?

**Quantitatively, no. Qualitatively, yes — in specific places.**

By the standard Elastic heuristic (~20 shards per GB of heap), each hot node with 29.6 GB heap can hold ~592 shards. You're sitting at an average of **~28 shards per hot node** — less than 5% of the safe ceiling. The cluster is nowhere near its shard-count limit.

But raw count isn't the only measure. The waste signal:

- **322 primaries**, of which a huge chunk are 0–1 GB tiny shards (`cdm_vuln_remediated` alone contributes 18 primaries × 2 = 36 shards for 9 GB total — half a node's worth of shard slots for a tablespoon of data).
- **74 of the ~322 primaries** have store size of 0 GB (effectively empty — see §3).
- The **`.enrich-*`** indices each replicate to all 36 hot nodes. That's by design (enrich indices need a copy on every node that runs the ingest pipeline), but you have **6 distinct enrich indices** × 36 copies = **216 shards** purely for enrich. They're tiny (≤2 MB each for most) so it's cheap, but it explains why your shard total looks inflated relative to your data volume.

**Net assessment:** The cluster isn't over-sharded in a way that risks stability. It *is* over-sharded in a way that wastes cluster-state memory and master overhead. Fixing the over-sharded indices in §1 plus deleting/cleaning up the empty indices in §3 would meaningfully simplify cluster state without changing capacity.

## 3. Empty indices & their sources

There are **36 indices with zero documents**. They break down cleanly by origin:

**Kibana Alerting framework (15 indices)** — the entire `.internal.alerts-*-default-000001` family: security, ML anomaly detection, observability (metrics/logs/uptime/threshold/slo/apm), streams, stack, transform health, dataset quality, attack discovery, and the default rule. These are auto-created by Kibana the first time each rule type is initialized; they stay empty if you have no detection rules of that type firing. **Action:** leave them alone — Kibana recreates them if deleted, and they cost nothing.

**Enterprise Search / connectors (4 indices)** — `.elastic-connectors-v1`, `.elastic-connectors-sync-jobs-v1`, plus the `.ds-logs-enterprise_search.{audit,api}-*` datastream backers. Auto-provisioned by Enterprise Search even when the feature is unused. Since Enterprise Search is **deprecated in 8.x and removed in 9.0**, and you're on 8.19.14, this is a candidate for removal: if no one uses Enterprise Search/App Search/Workplace Search in this deployment, you can delete the entire `.ent-search-*` family (I count ~80 of these in the shard listing) and the connectors indices. That alone would shed a significant chunk of cluster state.

**CDM trending datastream backers (7 indices)** — `.ds-cdm_cyhy_*_trending-*`, `.ds-cdm_asm_nmi_trending-*`, `.ds-cdm_scuba_gws_*_trending-*`, `.ds-cdm_vdp_trending-*`. These are normal datastream rollover artifacts: the active write index for a datastream that hasn't received documents in the current window. Empty after rollover is expected; ILM should age them out. Worth checking whether the upstream ingestion for `cdm_asm_nmi`, `cdm_vdp`, and `cdm_scuba_gws_*` is *supposed* to be running — those datastreams have no live data anywhere (the latest, the result, and the trending all show 0 docs), which suggests broken pipelines, not idle ones.

**CDM application indices that should have data but don't (9 indices)** — this is the concerning set:
- `cdm_scuba_gws_result`, `cdm_scuba_gws_result_latest`, `cdm_scuba_gws_common_control_alert_rule`, `cdm_scuba_gws_common_control_alert_rule_latest` — entire SCuBA Google Workspace family is empty.
- `cdm_asm_nmi`, `cdm_vdp` — empty.
- `cdm_scan_trending-6-0000001`, `cdm_system_boundary_trending-6-0000001`, `cdm_system_boundary_fisma_1163_rollup` — empty.

These look like targets of pipelines that aren't producing. Given your CDM pipeline silent-failure history (the `Connection reset by peer` zero-document case), I'd treat these as candidates for an ingestion audit.

**Frozen tier (1)** — `partial-.ds-cdm_asm_nmi_trending-2025.03.12-000001` — empty searchable snapshot, harmless.

## 4. Indices with zero replicas in the hot tier

**There are no hot-tier indices with `rep=0`.** All 9 zero-replica indices are `partial-*` searchable-snapshot mounts living exclusively on `instance-0000000042` (the single frozen-tier node, role `f`).

This is correct by design — frozen-tier searchable snapshots store the canonical copy in the snapshot repository (typically S3 in Elastic Cloud), and the frozen node holds only a small disk cache. The `rep=0` is expected; durability comes from the snapshot repo, not the node. **No data-loss risk here.**

That said, the frozen node itself is at **95.57% disk used** (99.7 GB of 99.7 GB usable on a 2.1 TB volume — note the math: ESS frozen tier uses overcommit). This is worth watching. If the cache fills and starts evicting aggressively, frozen-tier query latency will degrade. If frozen usage is expected to grow, you may want to scale the frozen tier up.

## 5. Other operational insights

A few things worth flagging that you didn't specifically ask about:

**Disk skew across hot nodes is dramatic.** Looking at `disk.indices` per node, the spread runs from 404 MB (`instance-0000000038`) to 51.1 GB (`instance-0000000016`). That's a **126× spread** on nodes that should be roughly balanced. Five nodes are sitting under 5 GB of indices while six are over 35 GB. The shard *count* is balanced (27–30 per node), but shard *sizes* are not — the few large primaries (`cdm_vuln_trending`, `cdm_swam_trending`) dominate, and the disk shard allocator doesn't currently use `write_load.forecast` aggressively because most of your `write_load.forecast` values are effectively zero. With absolute disk usage so low (max 3.18% on any hot node), this isn't urgent — but if you grow into it, you'll want to look at the desired balancer settings (`cluster.routing.allocation.balance.*`) or wait for 8.19's improved write-load-aware allocation to kick in.

**Heap utilization is healthy but uneven.** Heap percent ranges from 6% (`instance-0000000040`) to 60% (`instance-0000000019`). RAM percent is 57–78%. Nothing alarming, but the variance correlates with the disk skew above — same root cause.

**Master node CPU spike.** `instance-0000000005` (a master-eligible node) shows `load_1m=2.80, load_5m=2.78, load_15m=2.59` — sustained load on a 3.7 GB heap node. The other masters look fine. Worth checking what's running on it; could be a long master task or a misallocated workload.

**Ingest node load is high.** All three ingest nodes (`instance-0000000043/44/46`) show load averages of 3.3–5.2 on what are small (3.7 GB heap) nodes. This is consistent with active pipeline work but suggests they're a candidate for sizing review if pipeline volume grows.

**Active deletes signal write/update pressure.** `cdm_swam` has 39 M deleted docs against 32 M live (54% delete ratio). `cdm_vuln` has 10 M deleted against 15 M live (41%). Heavy `update`/`index`-by-id traffic generates these tombstones; merge throttling won't reclaim them until segment merges run. If query latency on these indices feels off, a `_forcemerge?max_num_segments=1&only_expunge_deletes=true` during a quiet window will help — but be aware force-merge is expensive and non-throttled, so plan it.

**Enrich index proliferation.** Six distinct enrich policies (`cdm_software_dictionary_eol`, `cdm_vuln_cve`, `cdm_vuln_aware_cve`, `cdm_csm_aware_stig`, `cdm_cdo_temp`, `cdm_software_directive`, `cdm_config_t0`, `cdm_csm_stig`) each create one shard per hot node = ~7 × 36 ≈ 252 enrich shards. They're tiny so the disk cost is nil, but each rebuild via `_enrich/policy/<name>/_execute` rewrites all 36 copies. If you're rebuilding these on a schedule, check the cadence — daily rebuilds across all of them is doable but adds master and indexing load.

**Reporting datastream is healthy.** `.ds-.kibana-reporting-2026.04.08-000001` has 11 docs — small but active, suggesting Kibana reporting is being used.

**No unassigned shards, no relocating shards, no in-flight fetches, no pending tasks.** From a stability standpoint, this is a clean cluster. The improvements above are quality-of-life and efficiency, not firefighting.

---

**Suggested priority of remediation:**

1. **High-value, low-risk:** Reindex `cdm_vuln_remediated` (18→1 primaries) and `cdm_csm` (6→1). Frees ~25 cluster-state slots, immediate win.
2. **Medium effort:** Investigate the 9 empty `cdm_*` application indices (SCuBA GWS, ASM NMI, VDP, scan/system_boundary trending). Are pipelines broken or intentionally idle?
3. **Optional cleanup:** If Enterprise Search isn't used, decommission the entire `.ent-search-*` and `.elastic-connectors*` set before 9.0 forces the issue.
4. **Watch:** Frozen node disk at 95.57% — confirm cache eviction is keeping up, scale frozen tier if growth is expected.
5. **Investigate:** Master node `instance-0000000005` sustained load.

Happy to dig into any of these — I can sketch the reindex command for `cdm_vuln_remediated`, or help you build a watcher for the frozen disk threshold if either would be useful next.
